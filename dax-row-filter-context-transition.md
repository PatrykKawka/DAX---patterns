# Row Context, Filter Context i Context Transition w DAX — przewodnik na przykładach

Przewodnik komplementarny do przewodników o DAX Studio i Tabular Editor — tam mierzysz/edytujesz, tu rozumiesz *dlaczego* konkretny wzorzec DAX daje taki, a nie inny wynik. To jest najczęstsze źródło błędów logicznych (nie wydajnościowych) w modelach takich jak Twój. Przykłady oparte na modelu: `dim_Klienci`, `dim_Placowki`, `dim_Kalendarz`, `dim_Produkty`, `fact_Sprzedaz` (standardowa gwiazda) + `dim_Liczba_miesiecy`, `dim_%_sprzedazy` (tabele parametryczne bez relacji).

---

## 1. Dwa rodzaje kontekstu — definicje, zanim zobaczysz kod

- **Filter Context (kontekst filtru)** — zbiór filtrów aktywnych w danym momencie obliczenia. Powstaje z: wierszy/kolumn wizuala, slicerów, `CALCULATE`, relacji między tabelami. To jest kontekst "z zewnątrz" — mówi silnikowi *które wiersze tabel bazowych są w ogóle widoczne* przy obliczeniu miary.
- **Row Context (kontekst wiersza)** — istnieje tylko wtedy, gdy silnik "stoi" na konkretnym, pojedynczym wierszu tabeli. Powstaje w: kolumnach obliczanych (zawsze) i funkcjach iterujących (`SUMX`, `FILTER`, `ADDCOLUMNS`, `RANKX` i inne z końcówką `X` lub iterujące po naturze). Row Context **nie filtruje** niczego samoczynnie — to tylko wskaźnik "jestem teraz na tym wierszu".
- **Context Transition (transformacja kontekstu)** — mechanizm, w którym `CALCULATE` (jawne lub niejawne, np. wewnątrz miary wywołanej z iteratora) **zamienia aktualny Row Context na Filter Context** równoważny "ten jeden wiersz i tylko ten wiersz". To jest most między dwoma światami i najczęstsze źródło nieporozumień w DAX.

Zasada, którą warto zapamiętać na starcie: **kolumna obliczana widzi tylko Row Context (i istniejący na starcie Filter Context, jeśli jakiś jest), a miara zawsze operuje przez Filter Context — jeśli miara jest wywołana wewnątrz Row Context, musi dojść do Context Transition, żeby w ogóle dała wynik.**

---

## 2. Filter Context — podstawa, na której stoi każda miara

```dax
Sprzedaz Total = SUM ( fact_Sprzedaz[Kwota] )
```

To najprostsza miara — nie ma tu żadnego Row Context, tylko czysty Filter Context. Kiedy umieścisz tę miarę w wizualu z osią `dim_Kalendarz[Rok]` i `dim_Klienci[Segment]`, silnik:

1. Buduje Filter Context z bieżącej kombinacji Rok + Segment (np. Rok=2025, Segment="Premium").
2. Filtr ten propaguje się przez relacje z `dim_Kalendarz` i `dim_Klienci` do `fact_Sprzedaz` (kierunek Single, standardowy w Twoim modelu — patrz przewodnik Tabular Editor, sekcja 6).
3. `SUM` sumuje kolumnę `Kwota` **tylko dla wierszy `fact_Sprzedaz`, które przeszły przez ten filtr**.

**Kluczowy fakt:** Filter Context nie musi pochodzić z wizuala. `CALCULATE` pozwala go modyfikować jawnie:

```dax
Sprzedaz PY =
CALCULATE (
    [Sprzedaz Total],
    SAMEPERIODLASTYEAR ( dim_Kalendarz[Data] )
)
```

Tu `CALCULATE` **nadpisuje** filtr na `dim_Kalendarz[Data]` istniejący dotąd w kontekście, zastępując go przesuniętym o rok. Reszta filtrów (np. Segment="Premium") pozostaje bez zmian — `CALCULATE` modyfikuje tylko te wymiary filtru, które jawnie wskażesz.

---

## 3. Row Context — powstaje w iteratorach i kolumnach obliczanych

**Kolumna obliczana** (przykład koncepcyjny — nie zalecam tego akurat jako kolumny w produkcyjnym modelu, ale dobrze ilustruje Row Context):

```dax
fact_Sprzedaz[Marza_procentowa] =
DIVIDE ( fact_Sprzedaz[Kwota] - fact_Sprzedaz[Koszt], fact_Sprzedaz[Kwota] )
```

Silnik oblicza tę formułę **osobno dla każdego wiersza** `fact_Sprzedaz` — `fact_Sprzedaz[Kwota]` odnosi się zawsze do wartości w *aktualnie przetwarzanym wierszu*, nie do sumy całej tabeli. To jest Row Context: "jesteś na wierszu X, `[Kwota]` = wartość Kwota w wierszu X".

**Iterator** — dokładnie ten sam mechanizm, ale tymczasowy, w obrębie jednej miary:

```dax
Sprzedaz Netto SUMX =
SUMX (
    fact_Sprzedaz,
    fact_Sprzedaz[Kwota] - fact_Sprzedaz[Koszt]
)
```

`SUMX` przechodzi wiersz po wierszu przez `fact_Sprzedaz` (w granicach bieżącego Filter Context — to ważne, iterator i tak "widzi" tylko wiersze przepuszczone przez filtr z zewnątrz), dla każdego wiersza tworzy Row Context, oblicza wyrażenie `Kwota - Koszt` w tym kontekście, a na końcu sumuje wszystkie wyniki. Efekt identyczny jak `SUM(Kwota) - SUM(Koszt)` w tym konkretnym przypadku, ale `SUMX` jest niezbędny, gdy wyrażenie nie jest liniowo rozdzielne (np. `Kwota * Ilość`, `Kwota / (1 + VAT)` per wiersz).

**Pułapka podstawowa:** `SUMX(fact_Sprzedaz, [Sprzedaz Total])` (odwołanie do miary wewnątrz iteratora) **nie** zwróci sumy `Kwota` per wiersz — bo `[Sprzedaz Total]` to miara, a miary zawsze przechodzą przez Context Transition (sekcja 4) i w tym przypadku zwrócą, dla każdego wiersza, sumę Kwota dla *tego jednego wiersza* (bo Context Transition zamienia Row Context na filtr "dokładnie ten wiersz"), czyli efektywnie to samo co `[Kwota]` per wiersz — ale kosztem dużo droższego wykonania (osobne wywołanie `CALCULATE` na każdy wiersz). To pierwszy sygnał, dlaczego rozróżnienie ma znaczenie nie tylko poznawcze, ale wydajnościowe (patrz też przewodnik DAX Studio, sekcja 5 — CallbackDataID i koszt operacji per wiersz).

---

## 4. Context Transition — most między Row Context a Filter Context

To jest najważniejsza koncepcja w tym przewodniku. Zasada: **`CALCULATE` (jawne lub ukryte wewnątrz wywołania miary) bierze aktualny Row Context i zamienia go w Filter Context, który filtruje model do "dokładnie tych wartości kolumn, jakie ma bieżący wiersz".**

### 4.1 Przykład od podstaw — dlaczego to w ogóle jest potrzebne

```dax
Liczba Klientow z Duza Sprzedaza =
SUMX (
    VALUES ( dim_Klienci[ID_Klienta] ),
    IF ( [Sprzedaz Total] > 10000, 1, 0 )
)
```

Co się dzieje krok po kroku:

1. `VALUES(dim_Klienci[ID_Klienta])` zwraca listę unikalnych klientów widocznych w bieżącym Filter Context (np. z wizuala).
2. `SUMX` iteruje po tej liście — dla każdego klienta tworzy Row Context, w którym `dim_Klienci[ID_Klienta]` = konkretny klient.
3. Wewnątrz `IF` wywołujesz **miarę** `[Sprzedaz Total]`. Miara zawsze potrzebuje Filter Context, a mamy tylko Row Context (jesteśmy "na wierszu" klienta w tabeli wirtualnej z `VALUES`) — więc silnik **automatycznie opakowuje to wywołanie w `CALCULATE`**, co wyzwala Context Transition: bieżąca wartość `dim_Klienci[ID_Klienta]` z Row Context staje się filtrem "ID_Klienta = ten konkretny klient" w nowym Filter Context.
4. `[Sprzedaz Total]` liczy się więc dla **tego jednego klienta**, mimo że formalnie miara nie wie nic o "wierszach" — dostaje gotowy, zawężony Filter Context.

**To jest dokładnie mechanizm, na którym opierają się Twoje miary TOP10/TOP10%.**

### 4.2 Ten sam mechanizm, ale jawny `CALCULATE`

Powyższy przykład bez ukrytej transformacji, zapisany explicite — dokładnie to samo dzieje się "pod maską":

```dax
Liczba Klientow z Duza Sprzedaza v2 =
SUMX (
    VALUES ( dim_Klienci[ID_Klienta] ),
    IF (
        CALCULATE ( [Sprzedaz Total] ) > 10000,   -- jawny CALCULATE, identyczny efekt jak niejawny w 4.1
        1,
        0
    )
)
```

Sam `CALCULATE()` bez żadnych dodatkowych filtrów, wywołany wewnątrz iteratora, **też wykonuje Context Transition** — sam fakt użycia `CALCULATE` wewnątrz Row Context wystarczy, żeby zamienić Row Context na Filter Context oparty o wartości bieżącego wiersza. To częsty błąd w interpretacji: ludzie myślą, że `CALCULATE` "nic nie robi", jeśli nie ma dodatkowych argumentów filtru — w rzeczywistości w Row Context zawsze robi Context Transition.

### 4.3 Context Transition na Twojej mierze TOP10 klientów — pełna analiza

```dax
Sprzedaz TOP10 Klientow =
VAR TopKlienci =
    TOPN ( 10, VALUES ( dim_Klienci[ID_Klienta] ), [Sprzedaz Total], DESC )
RETURN
    CALCULATE ( [Sprzedaz Total], TopKlienci )
```

Rozbijmy to na konteksty:

1. **`VALUES(dim_Klienci[ID_Klienta])`** — lista klientów w bieżącym Filter Context (np. wszystkich klientów, albo tylko z wybranej `dim_Placowki`, zależnie od wizuala).
2. **`TOPN(10, ..., [Sprzedaz Total], DESC)`** — to też iterator. Dla każdego klienta z listy tworzy Row Context, wywołuje `[Sprzedaz Total]` (Context Transition — dokładnie jak w 4.1), sortuje malejąco, zwraca 10 najlepszych jako tabelę.
3. **`TopKlienci`** — to jest **tabela**, nie skalar — mimo że powstała przez iterację z Row Context, wynik `TOPN` sam w sobie to zbiór wierszy (Filter Context "gotowy do użycia" jako argument filtru).
4. **Zewnętrzny `CALCULATE([Sprzedaz Total], TopKlienci)`** — to jest kolejna, **osobna** operacja: nadpisuje Filter Context na `dim_Klienci`, zawężając go do tylko tych 10 klientów z `TopKlienci`, po czym liczy `[Sprzedaz Total]` w tym nowym, zawężonym kontekście.

**Dlaczego to jest tańsze niż wariant z `RANKX` + `FILTER`** (patrz przewodnik DAX Studio, sekcja 4): `TOPN` w kroku 2 owszem wykonuje Context Transition dla każdego klienta (to nieuniknione, potrzebujesz `[Sprzedaz Total]` per klient, żeby wiedzieć kto jest w TOP10), ale robi to **raz**, żeby wybrać 10 najlepszych, a następnie krok 4 to już tylko jedna, prosta agregacja na przefiltrowanej tabeli faktów. Wariant z `RANKX(ALL(...), ...)` wymusza policzenie rankingu dla *wszystkich* klientów (każdy przechodzi przez Context Transition), a dopiero potem filtruje do TOP10 — więcej Context Transitions = więcej pracy Formula Engine.

### 4.4 Miara "liczba klientów z >10 zamówieniami" — kolejny klasyczny wzorzec

```dax
Liczba Klientow Ponad 10 Zamowien =
COUNTROWS (
    FILTER (
        VALUES ( dim_Klienci[ID_Klienta] ),
        CALCULATE ( COUNTROWS ( fact_Sprzedaz ) ) > 10
    )
)
```

Analogicznie: `FILTER` iteruje po klientach (Row Context), `CALCULATE(COUNTROWS(fact_Sprzedaz))` wykonuje Context Transition per klient (filtruje `fact_Sprzedaz` do transakcji tego jednego klienta przez relację), `COUNTROWS` zewnętrzny liczy, ilu klientów przeszło przez warunek `> 10`.

**Wariant bez jawnego `CALCULATE` wewnątrz `FILTER` — czy zadziała?**

```dax
Liczba Klientow Ponad 10 Zamowien v2 =
COUNTROWS (
    FILTER (
        VALUES ( dim_Klienci[ID_Klienta] ),
        COUNTROWS ( fact_Sprzedaz ) > 10   -- BEZ CALCULATE!
    )
)
```

To **nie zadziała poprawnie** — `COUNTROWS(fact_Sprzedaz)` bez `CALCULATE` **nie jest miarą**, tylko wyrażeniem operującym na tabeli fizycznej `fact_Sprzedaz` — nie ma tu żadnej niejawnej Context Transition, bo `COUNTROWS` samo w sobie nie jest miarą wywołaną w Row Context, tylko funkcją tabelaryczną operującą na całej tabeli w bieżącym Filter Context (który na tym etapie jeszcze nie uwzględnia bieżącego klienta z Row Context `FILTER`). Wynik: dla każdego klienta dostaniesz tę samą liczbę — łączną liczbę wierszy `fact_Sprzedaz` w całym, niezawężonym kontekście. **To jest jeden z najczęstszych błędów logicznych w DAX** — brak `CALCULATE` tam, gdzie Row Context powinien "wejść" do filtra tabeli faktów.

---

## 5. Miara "klienci aktywni w każdym miesiącu" — Context Transition na dwóch poziomach naraz

```dax
Liczba Klientow Aktywnych =
CALCULATE (
    DISTINCTCOUNT ( fact_Sprzedaz[ID_Klienta] ),
    dim_Kalendarz[Miesiac] = SELECTEDVALUE ( dim_Liczba_miesiecy[Wartosc] )
)
```

Tu nie ma iteratora — to prosta miara z `CALCULATE` modyfikującym filtr na podstawie tabeli parametrycznej `dim_Liczba_miesiecy` (bez relacji do modelu, stąd `SELECTEDVALUE` zamiast polegania na propagacji filtru przez relację). Warto to zestawić z poprzednimi przykładami, bo pokazuje, że **nie każdy `CALCULATE` robi Context Transition** — tylko wtedy, gdy jest wywołany *wewnątrz Row Context*. Tutaj `CALCULATE` jest na najwyższym poziomie miary (brak otaczającego iteratora), więc po prostu modyfikuje istniejący Filter Context — żadnej transformacji z Row Context nie ma, bo Row Context w ogóle nie istnieje w tym miejscu.

**Rozszerzenie — aktywni klienci per miesiąc, jako iteracja po `dim_Kalendarz` (żeby np. narysować trend):**

```dax
Sredni Miesieczny Klienci Aktywni =
AVERAGEX (
    VALUES ( dim_Kalendarz[Miesiac] ),
    CALCULATE ( DISTINCTCOUNT ( fact_Sprzedaz[ID_Klienta] ) )
)
```

Tu `AVERAGEX` tworzy Row Context po miesiącach, `CALCULATE` wewnątrz wyzwala Context Transition (filtr do konkretnego miesiąca), `DISTINCTCOUNT` liczy unikalnych klientów w tym zawężonym kontekście, a `AVERAGEX` na końcu uśrednia wyniki po wszystkich miesiącach z bieżącego Filter Context.

---

## 6. Podwójny Context Transition — iterator wewnątrz iteratora

To już poziom zaawansowany, ale bardzo praktyczny przy analizach klient × produkt.

```dax
Liczba Klientow z Koncentracja Kategorii =
SUMX (
    VALUES ( dim_Klienci[ID_Klienta] ),                          -- Row Context #1: klient
    VAR SprzedazKlienta = CALCULATE ( [Sprzedaz Total] )         -- Context Transition #1 -> Filter Context: ten klient
    VAR NajlepszaKategoria =
        MAXX (
            VALUES ( dim_Produkty[Kategoria] ),                  -- Row Context #2: kategoria, ZAGNIEŻDŻONY w Filter Context "ten klient"
            CALCULATE ( [Sprzedaz Total] )                       -- Context Transition #2 -> Filter Context: ten klient + ta kategoria
        )
    RETURN
        IF ( DIVIDE ( NajlepszaKategoria, SprzedazKlienta ) > 0.7, 1, 0 )
)
```

Krok po kroku:
1. Zewnętrzny `SUMX` iteruje po klientach — Row Context #1.
2. `VAR SprzedazKlienta` — Context Transition #1: filtr zawężony do jednego klienta, licząc jego całkowitą sprzedaż.
3. Wewnętrzny `MAXX` iteruje po kategoriach produktów — Row Context #2, **ale wewnątrz już zawężonego Filter Context "ten klient"** (bo `VALUES(dim_Produkty[Kategoria])` samo w sobie nie zeruje filtra na klienta ustawionego wyżej — filtry z różnych tabel kumulują się, nie nadpisują się nawzajem, chyba że jawnie to zrobisz).
4. `CALCULATE([Sprzedaz Total])` wewnątrz `MAXX` — Context Transition #2: teraz filtr to "ten klient" **ORAZ** "ta kategoria" jednocześnie.
5. `MAXX` zwraca największą sprzedaż w pojedynczej kategorii dla tego klienta; porównanie z sumą całkowitą tego klienta daje wskaźnik koncentracji zakupów.

**Dlaczego to jest kosztowne (link do przewodnika DAX Studio):** dwa zagnieżdżone Context Transitions oznaczają, że dla *każdej* kombinacji klient × kategoria silnik wykonuje osobne, kosztowne przeliczenie Formula Engine. Przy dużej liczbie klientów i kategorii to jest dokładnie wzorzec, który w Server Timings pokaże wysoki % FE i dużą liczbę SE queries — dobry kandydat do zmierzenia i przetestowania alternatywy (np. przez `SUMMARIZECOLUMNS` budujący tabelę klient×kategoria jednym przebiegiem zamiast zagnieżdżonej iteracji) w DAX Studio, zanim wdrożysz do modelu.

---

## 7. `KEEPFILTERS` — kiedy Context Transition/CALCULATE "za bardzo" nadpisuje filtr

Domyślnie `CALCULATE` z warunkiem na kolumnie **nadpisuje** istniejący filtr na tej kolumnie (nie dodaje do niego). To bywa nieoczekiwane przy Context Transition połączonym z dodatkowym filtrem:

```dax
Sprzedaz Bez Segmentu Premium =
CALCULATE (
    [Sprzedaz Total],
    dim_Klienci[Segment] <> "Premium"
)
```

Jeśli w wizualu masz już wybrany konkretny `Segment = "VIP"` (filtr z zewnątrz), powyższa miara **nadpisze** ten filtr swoim własnym warunkiem `<> "Premium"` — czyli policzy sprzedaż dla wszystkich segmentów poza Premium, ignorując wybór "VIP" z wizuala. Żeby warunki się **łączyły** (AND) zamiast nadpisywać:

```dax
Sprzedaz Bez Segmentu Premium v2 =
CALCULATE (
    [Sprzedaz Total],
    KEEPFILTERS ( dim_Klienci[Segment] <> "Premium" )
)
```

Teraz silnik zachowuje istniejący filtr z wizuala **i** dokłada nowy warunek — wynik to przecięcie obu. To nie jest bezpośrednio Context Transition, ale jest ściśle powiązane, bo `KEEPFILTERS` najczęściej staje się potrzebny właśnie tam, gdzie masz `CALCULATE` wywoływany w kontekście, który już przeszedł przez wcześniejszą transformację (np. wewnątrz iteratora, gdzie Context Transition już ustawiła filtr na konkretnym kliencie, a Ty chcesz dodać kolejny warunek, nie zastępując go).

---

## 8. Podsumowanie — checklist mentalny przy pisaniu miary

Przy każdej nowej mierze z iteratorem warto zadać sobie kolejno:

1. **Czy jestem w Row Context czy Filter Context w tym miejscu kodu?** (Row Context = jestem wewnątrz `SUMX`/`FILTER`/`ADDCOLUMNS`/kolumny obliczanej; Filter Context = na zewnątrz, albo po `CALCULATE`).
2. **Czy odwołuję się do miary czy do kolumny/funkcji tabelarycznej wewnątrz Row Context?** Miara → automatyczna Context Transition (koszt + zmiana semantyki na "ten jeden wiersz"). Kolumna/funkcja tabelaryczna bez `CALCULATE` → brak transformacji, operuje na całej widocznej tabeli, nie na bieżącym wierszu (klasyczny błąd z sekcji 4.4).
3. **Czy mój `CALCULATE` ma nadpisać, czy dołożyć do istniejącego filtru?** Jeśli dołożyć — `KEEPFILTERS` (sekcja 7).
4. **Czy mam zagnieżdżone iteratory?** Jeśli tak — policz, ile Context Transitions faktycznie się wykona (per wiersz zewnętrznej iteracji × per wiersz wewnętrznej) i zweryfikuj w DAX Studio (Server Timings), zanim uznasz wzorzec za gotowy do wdrożenia.
