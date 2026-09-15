# RANK vs RANKX oraz funkcje okienkowe w DAX — pełny przewodnik

Dotyczy funkcji wprowadzonych w grudniu 2022 (`INDEX`, `OFFSET`, `WINDOW`) i w 2023 (`RANK`, `ROWNUMBER`) — nowej rodziny funkcji tabelarycznych do nawigacji po posortowanej, podzielonej na partycje tabeli. Przykłady na Twoim modelu: `fact_Sprzedaz`, `dim_Klienci`, `dim_Produkty`, `dim_Kalendarz`.

---

## 1. Wspólne budulce — zanim przejdziemy do poszczególnych funkcji

Wszystkie funkcje okienkowe (`RANK`, `ROWNUMBER`, `INDEX`, `OFFSET`, `WINDOW`) przyjmują ten sam zestaw parametrów pomocniczych — raz zrozumiane, działają identycznie wszędzie.

### `ORDERBY( <kolumna>, <ASC|DESC> [, <kolumna>, <ASC|DESC> ...] )`

Definiuje sortowanie, względem którego liczona jest pozycja. Przyjmuje **wiele kolumn naraz** — to jest największa przewaga tej rodziny funkcji nad `RANKX` (który sortuje tylko po jednym wyrażeniu).

```dax
ORDERBY ( [Sprzedaz Total], DESC, dim_Klienci[ID_Klienta], ASC )
```

### `PARTITIONBY( <kolumna> [, <kolumna> ...] )`

Dzieli tabelę na niezależne grupy — pozycja/ranking liczony jest osobno w obrębie każdej partycji. Jeśli pominięty, cała tabela to jedna partycja.

```dax
PARTITIONBY ( dim_Klienci[Segment] )   -- ranking liczony osobno w każdym segmencie
```

### `MATCHBY( <kolumna> [, <kolumna> ...] )`

Kolumny jednoznacznie identyfikujące "bieżący wiersz" w kontekście, z którego funkcja jest wywoływana — potrzebne, gdy `ORDERBY`/`PARTITIONBY` same w sobie nie wystarczają do ustalenia, który wiersz jest "bieżący" (np. tabela ma duplikaty wartości sortowania). Rzadko potrzebne w prostych scenariuszach — DAX próbuje domyślnie dobrać potrzebne kolumny automatycznie i zwraca błąd, jeśli się nie da.

### `Blanks` (parametr sortowania pustych wartości)

Określa, gdzie w sortowaniu lądują wartości `BLANK` — `LAST` (zawsze na końcu, niezależnie od ASC/DESC) jest częstym wyborem jawnym, domyślne zachowanie różni się w zależności od funkcji (patrz sekcje poszczególnych funkcji).

### `Reset` — tylko w kalkulacjach wizualnych (visual calculations)

Określa, kiedy licznik/ranking "resetuje się" w hierarchii wizuala (`NONE`, `LOWESTPARENT`, `HIGHESTPARENT`, konkretny poziom). Dotyczy tylko nowszej funkcji kalkulacji wizualnych w Power BI, nie zwykłych miar modelu — nie zagłębiam się w to tutaj, bo prawdopodobnie nie używasz jeszcze tej funkcji w swoim workflow.

---

## 2. `RANKX` — klasyka, pełny opis parametrów

```dax
RANKX ( <Table>, <Expression> [, <Value>] [, <Order>] [, <Ties>] )
```

| Parametr | Wymagany | Opis |
|---|---|---|
| `<Table>` | Tak | Tabela, po której iterujemy, żeby zbudować listę wartości do rankingu |
| `<Expression>` | Tak | Wyrażenie liczone dla każdego wiersza `<Table>` — buduje "tabelę odniesienia" do rankingu |
| `<Value>` | Nie | Wartość, której ranking szukamy. Domyślnie: wartość `<Expression>` w bieżącym kontekście. Rzadko ustawiana ręcznie — głównie przy porównywaniu rankingu do zewnętrznej, stałej wartości spoza `<Table>` |
| `<Order>` | Nie | `0`/`FALSE`/`DESC` (domyślnie) = malejąco, ranga 1 dla najwyższej wartości. `1`/`TRUE`/`ASC` = rosnąco |
| `<Ties>` | Nie | `SKIP` (domyślnie) — po remisie 5 wartości na randze 11, następna dostaje 16 (11+5, "przeskakuje" zajęte pozycje). `DENSE` — następna dostaje 12 (bez przerwy w numeracji) |

**Przykład — ranking klientów wg sprzedaży:**

```dax
Ranking Klientow =
RANKX ( ALL ( dim_Klienci[ID_Klienta] ), [Sprzedaz Total] )
```

**Przykład z `DENSE` — ranking bez "dziur" po remisach:**

```dax
Ranking Klientow Dense =
RANKX ( ALL ( dim_Klienci[ID_Klienta] ), [Sprzedaz Total], , DESC, DENSE )
```

**Parametr `<Value>` - przykładowe scenariusze użycia**

**a) scenariusz "co gdyby" (What-If parameter)**
Załóżmy, że masz w Power BI parametr What-If pozwalający użytkownikowi wpisać hipotetyczną kwotę sprzedaży (np. suwak od 0 do 500 000 zł) i chcesz pokazać, na którym miejscu w rankingu klientów znalazłaby się taka wartość — niezależnie od tego, który klient jest aktualnie w kontekście wiersza:
```dax
Ranking Hipotetycznej Sprzedazy =
VAR HipotetycznaSprzedaz = SELECTEDVALUE ( dim_ParametrWhatIf[Wartosc] )
RETURN
    RANKX (
        ALL ( dim_Klienci[ID_Klienta] ),
        [Sprzedaz Total],
        HipotetycznaSprzedaz
    )
```
**b) Przykład 2 — ranking klienta z zeszłego roku względem tegorocznej listy**
Praktyczniejszy przypadek: chcesz sprawdzić, na którym miejscu w tegorocznym rankingu znalazłby się klient, gdyby porównać go z jego zeszłoroczną sprzedażą (klasyczne pytanie: "czy klient awansował, czy spadł w rankingu, licząc względem tej samej, aktualnej listy konkurentów"):
```dax
Ranking PY Wzgledem Biezacej Listy =
VAR SprzedazPYKlienta =
    CALCULATE ( [Sprzedaz Total], SAMEPERIODLASTYEAR ( dim_Kalendarz[Data] ) )
RETURN
    RANKX (
        ALL ( dim_Klienci[ID_Klienta] ),
        [Sprzedaz Total],          -- tabela odniesienia: BIEŻĄCA sprzedaż wszystkich klientów
        SprzedazPYKlienta          -- wartość do zrankowania: zeszłoroczna sprzedaż TEGO klienta
    )
```

**Ograniczenie, które rozwiązuje `RANK` (sekcja 3):** `RANKX` sortuje tylko po jednym `<Expression>` — remisy w tej jednej wartości nie mają wbudowanego, prostego sposobu rozstrzygnięcia drugą kolumną (np. alfabetycznie). Da się to obejść, ale wymaga sztucznego "podbicia" wartości drugą kolumną (np. dodanie znikomego ułamka zależnego od ID), co SQLBI opisuje jako poprawne, ale nieintuicyjne i czasochłonne do napisania poprawnie.

---

## 3. `RANK` — nowocześniejszy, wielokolumnowy ranking

```dax
RANK ( [<Ties>] [, <Relation> lub <Axis>] [, <OrderBy>] [, <Blanks>] [, <PartitionBy>] [, <MatchBy>] [, <Reset>] )
```

| Parametr | Wymagany | Opis |
|---|---|---|
| `<Ties>` | Nie | `SKIP` (domyślnie) lub `DENSE` — identyczna semantyka jak w `RANKX` |
| `<Relation>` | Nie* | Tabela źródłowa. Jeśli pominięta: `<OrderBy>` musi być podane jawnie, a wszystkie kolumny `<OrderBy>`/`<PartitionBy>` muszą pochodzić z jednej, w pełni kwalifikowanej tabeli — silnik domyślnie użyje `ALLSELECTED()` tych kolumn |
| `<OrderBy>` | Warunkowo | Klauzula `ORDERBY(...)` — **może zawierać wiele kolumn**, to jest kluczowa przewaga nad `RANKX` |
| `<Blanks>` | Nie | Sposób sortowania pustych wartości |
| `<PartitionBy>` | Nie | Klauzula `PARTITIONBY(...)` — ranking osobno w obrębie każdej partycji |
| `<MatchBy>` | Nie | Kolumny identyfikujące bieżący wiersz, gdy `OrderBy`/`PartitionBy` nie wystarczają |
| `<Reset>` | Nie | Tylko w kalkulacjach wizualnych |

**Ważne zdanie wprost z dokumentacji Microsoftu, warte zapamiętania:** *"RANK nie ma się tak do RANKX, jak SUM do SUMX"* — `RANK` to nie jest po prostu "nowszy RANKX". Ma inną semantykę ("current row" ustalane przez `PartitionBy`/`MatchBy`, nie przez klasyczny Row Context iteratora) i **zwraca `BLANK` dla wierszy podsumowań/totali** — trzeba to świadomie testować, nie zakładać automatycznie poprawnego zachowania na każdym poziomie hierarchii wizuala.

**Przykład — ranking marek z rozstrzyganiem remisów alfabetycznie (dokładnie ten przypadek, w którym `RANKX` jest kłopotliwy):**

```dax
Ranking Produktow =
RANK (
    DENSE,
    ALLSELECTED ( dim_Produkty[Nazwa] ),
    ORDERBY ( [Sprzedaz Total], DESC, dim_Produkty[Nazwa], ASC )
)
```

Dwaj producenci z identyczną sprzedażą dostają różne rangi rozstrzygnięte alfabetycznie po nazwie — jedna linia kodu, bez sztuczek.

**Przykład — ranking klienta w obrębie jego segmentu (partycjonowanie):**

```dax
Ranking Klienta W Segmencie =
RANK (
    SKIP,
    ALLSELECTED ( dim_Klienci[ID_Klienta] ),
    ORDERBY ( [Sprzedaz Total], DESC ),
    ,
    PARTITIONBY ( dim_Klienci[Segment] )
)
```

**Kiedy wybrać `RANK` zamiast `RANKX`:** gdy potrzebujesz tie-breakingu wieloma kolumnami, albo rankingu w obrębie partycji bez ręcznego filtrowania tabeli przed `RANKX`. W prostym, jednokolumnowym rankingu bez remisów — `RANKX` jest równie dobry, prostszy i lepiej znany (więcej materiałów, więcej osób potrafi go czytać).

---

## 4. `ROWNUMBER` — unikalna numeracja wierszy

```dax
ROWNUMBER ( [<Relation> lub <Axis>] [, <OrderBy>] [, <Blanks>] [, <PartitionBy>] [, <MatchBy>] [, <Reset>] )
```

Parametry identyczne jak w `RANK` (bez `<Ties>`). **Kluczowa różnica względem `RANK`/`RANKX`: `ROWNUMBER` zawsze zwraca unikalny numer dla każdego wiersza — nie ma pojęcia remisu.** Przy identycznych wartościach sortowania, kolejność między "remisującymi" wierszami jest deterministyczna dzięki dodatkowym kolumnom domyślnie dołączanym do sortowania (albo jawnie przez `MatchBy`), ale nie ma dwóch wierszy z tym samym numerem.

**Przykład — numeracja transakcji klienta chronologicznie (przydatne np. do identyfikacji "która to zakupowo transakcja klienta"):**

```dax
Numer Transakcji Klienta =
ROWNUMBER (
    ORDERBY ( fact_Sprzedaz[Data], ASC ),
    ,
    PARTITIONBY ( fact_Sprzedaz[ID_Klienta] )
)
```

**Kiedy `ROWNUMBER` zamiast `RANK`:** gdy potrzebujesz gwarantowanej unikalności (np. do numerowania wierszy w eksporcie, paginacji, albo identyfikacji "n-tej transakcji"), a nie klasycznego rankingu biznesowego, gdzie remisy powinny dostawać tę samą pozycję.

---

## 5. `INDEX` — n-ty wiersz w kolejności

```dax
INDEX ( <Position> [, <Relation> lub <Axis>] [, <OrderBy>] [, <Blanks>] [, <PartitionBy>] [, <MatchBy>] [, <Reset>] )
```

| Parametr | Wymagany | Opis |
|---|---|---|
| `<Position>` | Tak | Pozycja wiersza do zwrócenia (1-based — `1` to pierwszy wiersz w sortowaniu) |
| pozostałe | jak wyżej | Identyczna logika co w `RANK`/`ROWNUMBER` |

Zwraca **tabelę** (cały wiersz, nie skalar) — zwykle używane wewnątrz `SELECTCOLUMNS`/jako argument tabelaryczny, żeby wyciągnąć konkretną kolumnę.

**Przykład — najlepiej sprzedający się produkt (odpowiednik "TOP 1" bez pisania `TOPN`):**

```dax
Bestseller =
VAR ProduktySprzedaz =
    ADDCOLUMNS ( VALUES ( dim_Produkty[Nazwa] ), "@Sprzedaz", [Sprzedaz Total] )
VAR NajlepszyWiersz = INDEX ( 1, ProduktySprzedaz, ORDERBY ( [@Sprzedaz], DESC ) )
RETURN
    SELECTCOLUMNS ( NajlepszyWiersz, "Nazwa", dim_Produkty[Nazwa] )
```

**Uwaga o automatycznym dobieraniu kolumn:** jeśli `OrderBy`/`PartitionBy` nie identyfikują jednoznacznie każdego wiersza (np. dwa produkty o identycznej sprzedaży), `INDEX` **samodzielnie szuka minimalnego zestawu dodatkowych kolumn**, żeby jednoznacznie posortować tabelę, i dopisuje je do `OrderBy`. Jeśli się nie da (kolumny nie istnieją albo nadal niejednoznaczne) — zwraca błąd, zamiast zgadywać. To zachowanie różni się od `RANK`/`ROWNUMBER`, gdzie wprost odpowiadasz za jednoznaczność przez `MatchBy`.

---

## 6. `OFFSET` — wiersz przed/po bieżącym

```dax
OFFSET ( <Delta> [, <Relation> lub <Axis>] [, <OrderBy>] [, <Blanks>] [, <PartitionBy>] [, <MatchBy>] [, <Reset>] )
```

| Parametr | Wymagany | Opis |
|---|---|---|
| `<Delta>` | Tak | Liczba wierszy przed (wartość ujemna) lub po (wartość dodatnia) bieżącym wierszu. Dowolne wyrażenie DAX zwracające skalar |
| `<Relation>` | Warunkowo | Jeśli podana: wszystkie kolumny `PartitionBy` muszą z niej pochodzić (lub z tabeli powiązanej). Jeśli pominięta: `OrderBy` musi być podane jawnie, wszystkie kolumny w pełni kwalifikowane, z jednej tabeli |
| pozostałe | jak wyżej | |

Zwraca **pojedynczy wiersz** (albo wiele, jeśli "bieżący wiersz" nie da się jednoznacznie zredukować do jednego — patrz sekcja 8 o apply semantics).

**Przykład — sprzedaż względem poprzedniego miesiąca (alternatywa dla `PREVIOUSMONTH`/`DATEADD`):**

```dax
Sprzedaz vs Poprzedni Miesiac =
[Sprzedaz Total] - CALCULATE (
    [Sprzedaz Total],
    OFFSET ( -1, ORDERBY ( dim_Kalendarz[RokMiesiac] ) )
)
```

**Przykład — poprzednia transakcja tego samego klienta (czego żadna funkcja Time Intelligence z sekcji o kalendarzu nie da wprost, bo to nie jest przesunięcie kalendarzowe, tylko "poprzedni wiersz w kolejności"):**

```dax
Kwota Poprzedniej Transakcji Klienta =
VAR WierszPoprzedni =
    OFFSET (
        -1,
        fact_Sprzedaz,
        ORDERBY ( fact_Sprzedaz[Data], ASC ),
        ,
        PARTITIONBY ( fact_Sprzedaz[ID_Klienta] )
    )
RETURN
    SUMX ( WierszPoprzedni, fact_Sprzedaz[Kwota] )
```

**Wymóg kluczy przy braku unikalności:** jeśli `fact_Sprzedaz` nie ma kolumny jednoznacznie identyfikującej wiersz (np. samego `Numer_Zamowienia`), a sortowanie po samej dacie daje remisy (dwie transakcje tego samego dnia), `OFFSET` może zażądać `MatchBy` z dodatkową kolumną-kluczem, inaczej zwróci błąd albo wiele wierszy naraz.

---

## 7. `WINDOW` — zakres wierszy naraz

```dax
WINDOW (
    <From> [, <FromType>], <To> [, <ToType>]
    [, <Relation> lub <Axis>] [, <OrderBy>] [, <Blanks>] [, <PartitionBy>] [, <MatchBy>] [, <Reset>]
)
```

| Parametr | Wymagany | Opis |
|---|---|---|
| `<From>` | Tak | Początek okna. Znaczenie zależy od `<FromType>` |
| `<FromType>` | Nie | `REL` (domyślnie) — liczba wierszy wstecz (ujemna)/wprzód (dodatnia) od bieżącego wiersza. `ABS` — pozycja bezwzględna w partycji: dodatnia liczy od początku (1 = pierwszy wiersz), ujemna od końca (-1 = ostatni wiersz) |
| `<To>` | Tak | Koniec okna (wiersz włącznie). Ta sama logika co `<From>` |
| `<ToType>` | Nie | Jak `<FromType>` |
| pozostałe | jak wyżej | |

Zwraca **tabelę wierszy** (nie pojedynczy wiersz) — do agregowania przez `SUMX`/`AVERAGEX`/`COUNTROWS` itd.

**Przykład — średnia krocząca 3-miesięczna (dokładnie ten wzorzec, który wcześniej budowałeś przez `DATESINPERIOD` w przewodniku Time Intelligence):**

```dax
Srednia Kroczaca 3M (WINDOW) =
AVERAGEX (
    WINDOW ( -2, REL, 0, REL, ORDERBY ( dim_Kalendarz[RokMiesiac] ) ),
    [Sprzedaz Total]
)
```

`-2 REL` do `0 REL` = "od 2 wierszy wcześniej do bieżącego wiersza" = 3 miesiące łącznie.

**Przykład — pierwsze 3 miesiące każdego roku, pozycją bezwzględną (`ABS`), z partycją po roku:**

```dax
Sprzedaz Pierwszy Kwartal (WINDOW ABS) =
SUMX (
    WINDOW ( 1, ABS, 3, ABS, dim_Kalendarz, ORDERBY ( dim_Kalendarz[Miesiac] ), , PARTITIONBY ( dim_Kalendarz[Rok] ) ),
    [Sprzedaz Total]
)
```

**Porównanie z `DATESINPERIOD` z przewodnika Time Intelligence:** oba podejścia dają ten sam wynik dla średniej kroczącej — `WINDOW` jest czytelniejszy przy pracy na dowolnym sortowaniu (nie tylko datach), `DATESINPERIOD` jest bardziej wyspecjalizowany i lepiej "rozumiany" przez klasyczne materiały edukacyjne DAX. Dla miar czysto kalendarzowych (TTM, YTD-podobne) trzymałbym się `DATESINPERIOD` — dla okien opartych na dowolnym innym sortowaniu (np. kolejność transakcji, ranking) `WINDOW` jest naturalniejszy.

---

## 8. Apply semantics — dlaczego "bieżący wiersz" nie zawsze jest oczywisty

To jest koncepcja unikalna dla tej rodziny funkcji, konieczna do zrozumienia, zanim zaufasz wynikom w złożonym wizualu.

**Problem:** `OFFSET`/`WINDOW`/`INDEX` muszą jakoś ustalić, czym jest "bieżący wiersz", żeby policzyć względem niego przesunięcie. Gdy iterujesz wprost po tabeli, na której operuje funkcja (np. `SUMX(fact_Sprzedaz, ...OFFSET...)`), to oczywiste — bieżący wiersz to wiersz iteracji. **Problem pojawia się, gdy funkcja jest wywołana jako miara w wizualu o innej granularności niż tabela źródłowa funkcji** — np. wizual pokazuje kontynenty i lata, a `OFFSET` operuje na tabeli z osobnym wierszem na każdy kraj i rok. Wtedy "bieżący wiersz" to w rzeczywistości **wiele wierszy** (wszystkie kraje danego kontynentu), i apply semantics określa, jak `OFFSET` radzi sobie z tą wieloznacznością — w praktyce: może zwrócić wynik zagregowany po wszystkich pasujących "bieżących wierszach" naraz, co bywa poprawne, ale wymaga zrozumienia, że to się dzieje.

**Praktyczna konsekwencja dla Ciebie:** SQLBI wprost zaznacza, że pisząc miarę z funkcją okienkową, **nie możesz zakładać, że będzie wywołana w konkretnym, wąskim kontekście filtru** — użytkownik raportu może umieścić ją w dowolnym wizualu, na dowolnym poziomie hierarchii. Dlatego te funkcje są **naturalniejsze w zapytaniach DAX** (`EVALUATE` w DAX Studio, eksploracja, przygotowanie danych) niż jako stałe miary modelu w rozbudowanym, wielopoziomowym raporcie — tam, gdzie kontrolujesz dokładnie, w jakim kontekście funkcja się wykona.

---

## 9. Wydajność — co realnie wiadomo

**Ogólna zasada wprost od SQLBI: funkcje okienkowe "czasem, ale nie zawsze" poprawiają wydajność.** Ich głównym celem jest **czytelność i mniej kodu do napisania i utrzymania**, nie gwarantowana optymalizacja. Konkretne obserwacje:

- **`RANKX` ma wyjątkowo dobrze zoptymalizowany, wewnętrzny operator silnika** — patrz nasza wcześniejsza rozmowa o `RANKX` vs `COUNTROWS`+`FILTER` (ponad 1000× różnicy). To sprawia, że `RANKX` bywa **szybszy** niż intuicyjnie prostszy, ale gorzej zoptymalizowany kod ręczny.
- **`RANK`/`ROWNUMBER`/`INDEX`/`OFFSET`/`WINDOW` nie mają jeszcze tak długiej historii optymalizacji silnika** jak `RANKX` (który istnieje od początku DAX) — bywają szybsze niż odpowiadający im ręczny kod, ale nie jest to reguła bez wyjątków, zwłaszcza w złożonych scenariuszach z apply semantics (sekcja 8), gdzie silnik może wykonać więcej pracy, żeby poprawnie obsłużyć wieloznaczność "bieżącego wiersza".
- **Nie zakładaj z góry kierunku różnicy** — zmierz obie wersje (klasyczną i okienkową) w DAX Studio (Server Timings), dokładnie tak samo jak przy każdym innym porównaniu w tych przewodnikach. To jest young, wciąż rozwijana część silnika — jego charakterystyka wydajnościowa może się zmieniać między wersjami szybciej niż dawno ugruntowanych funkcji jak `RANKX` czy `CALCULATE`.

---

## 10. Podsumowanie — która funkcja do którego zadania

| Potrzebujesz | Funkcja |
|---|---|
| Prosty ranking, jedna kolumna, bez tie-breakingu | `RANKX` |
| Ranking z tie-breakingiem wieloma kolumnami, albo w obrębie partycji | `RANK` |
| Unikalna numeracja wierszy (bez remisów) — paginacja, identyfikacja n-tej pozycji | `ROWNUMBER` |
| Konkretny n-ty wiersz posortowanej tabeli (np. bestseller = pozycja 1) | `INDEX` |
| Wartość z poprzedniego/następnego wiersza w dowolnym sortowaniu (nie tylko kalendarzowym) | `OFFSET` |
| Zakres wierszy do agregacji (średnia/suma krocząca) na dowolnym sortowaniu | `WINDOW` |
| Krocząca suma/średnia konkretnie na osi czasu | `DATESINPERIOD` (przewodnik Time Intelligence) — bardziej wyspecjalizowane i lepiej ugruntowane niż `WINDOW` do tego zastosowania |
| Kod do zapytania/eksploracji w DAX Studio, przygotowania danych | Funkcje okienkowe — naturalne środowisko dla apply semantics |
| Stała miara modelu w złożonym, wielopoziomowym raporcie | Rozważ ostrożnie — przetestuj na każdym poziomie hierarchii wizuala przed wdrożeniem |
