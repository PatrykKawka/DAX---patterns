# Time Intelligence w DAX — kompletny przewodnik na przykładach

Pełne, systematyczne omówienie wszystkich 38 funkcji z oficjalnej kategorii Time Intelligence (dax.guide), pogrupowanych funkcjonalnie, na przykładach opartych na Twoim modelu: `fact_Sprzedaz`, `dim_Kalendarz[Data]`, miara bazowa `[Sprzedaz Total] = SUM(fact_Sprzedaz[Kwota])`. Rozszerzony o materiał z SQLBI (optymalizacja LASTNONBLANK, mechanizm DATEADD, obsługa BLANK w datach) i DAX Patterns (porównywanie niestandardowych okresów).

---

## 0. Wymóg wstępny — zanim jakakolwiek funkcja Time Intelligence zadziała poprawnie

1. **Ciągłość** — `dim_Kalendarz` musi zawierać *wszystkie* dni od 1 stycznia do 31 grudnia dla każdego reprezentowanego roku (nie tylko dni, w których była sprzedaż) — inaczej `DATESYTD`, `SAMEPERIODLASTYEAR` itp. "przeskoczą" brakujące dni i dadzą niepełny zakres.
2. **Jedna kolumna typu Date/DateTime z unikalnymi wartościami** — zwykle `dim_Kalendarz[Data]`, bez części godzinowej.
3. **`Mark as Date Table`** (Power BI: tabela → Table tools → "Mark as date table", wskaż `Data`) — formalnie wymagane tylko, gdy relacja z `fact_Sprzedaz` nie opiera się na tej kolumnie, ale w praktyce zawsze warto to ustawić.

---

## 1. Przesunięcie o okres — porównania rok do roku, miesiąc do miesiąca

### 1.1 `SAMEPERIODLASTYEAR` — najczęściej używana, najbardziej czytelna

```dax
Sprzedaz PY =
CALCULATE (
    [Sprzedaz Total],
    SAMEPERIODLASTYEAR ( dim_Kalendarz[Data] )
)
```

Zwraca zbiór dat z bieżącego kontekstu przesunięty dokładnie o rok wstecz (zachowuje długość okresu).

### 1.2 `DATEADD` — uniwersalne przesunięcie o dowolny interwał

```dax
Sprzedaz Miesiac Wczesniej =
CALCULATE (
    [Sprzedaz Total],
    DATEADD ( dim_Kalendarz[Data], -1, MONTH )
)

Sprzedaz PY (przez DATEADD) =
CALCULATE (
    [Sprzedaz Total],
    DATEADD ( dim_Kalendarz[Data], -1, YEAR )
)
```

Interwał to `DAY`, `MONTH`, `QUARTER` lub `YEAR`, liczba ujemna = wstecz. `DATEADD(..., -1, YEAR)` daje wynik identyczny co `SAMEPERIODLASTYEAR`.

### 1.3 `PARALLELPERIOD` — i dokładny mechanizm różnicy względem `DATEADD`

```dax
Sprzedaz Poprzedni Miesiac Caly =
CALCULATE (
    [Sprzedaz Total],
    PARALLELPERIOD ( dim_Kalendarz[Data], -1, MONTH )
)
```

**Różnica podstawowa:** `PARALLELPERIOD` zawsze zwraca **pełny okres kalendarzowy** (cały miesiąc/kwartał/rok), niezależnie od tego, ile dat faktycznie było w bieżącym kontekście. `DATEADD` przesuwa dokładnie ten zestaw dat, jaki miałeś w kontekście — nie "zaokrągla" do pełnego miesiąca.

**Dokładniejszy mechanizm (na podstawie analizy SQLBI):** dla interwału `DAY` obie funkcje działają identycznie — każda data w `Date[Date]` jest przesunięta o dokładnie N dni. Różnica ujawnia się dopiero przy interwałach `MONTH`/`QUARTER`/`YEAR`: `DATEADD` analizuje, jakie **miesiące kalendarzowe** są reprezentowane przez daty w bieżącym kontekście, i przesuwa cały ten zestaw o odpowiednią liczbę miesięcy, starając się zachować relatywną pozycję dat w miesiącu — natomiast `PARALLELPERIOD` po prostu bierze pełny zakres docelowego miesiąca/kwartału/roku, ignorując, która część oryginalnego miesiąca była faktycznie widoczna w kontekście.

**Kiedy to ma znaczenie w praktyce:** jeśli w wizualu masz filtr na pojedynczy dzień albo tylko część miesiąca (np. slicer ograniczający do pierwszych 10 dni marca), `DATEADD(..., -1, MONTH)` zwróci analogiczne 10 dni lutego, a `PARALLELPERIOD(..., -1, MONTH)` zwróci **cały** luty. Dla typowych miar "cały miesiąc/kwartał/rok" w standardowym wizualu (bez filtrowania na pojedyncze dni) oba dają ten sam wynik.

**Zastrzeżenie dot. przyszłości:** Microsoft wprowadził w Power BI (wersja preview, wrzesień 2025) tzw. **calendar-based time intelligence** — nową generację funkcji z dodatkowymi parametrami `DATEADD` pozwalającymi jawnie kontrolować zachowanie przy nierównej długości miesięcy (np. przesunięcie 31 stycznia o -1 miesiąc, gdy luty ma tylko 28 dni). To wciąż funkcja w wersji preview w chwili pisania tego przewodnika — nie zmienia działania klasycznych funkcji opisanych tutaj (opartych na "Mark as Date Table"), ale warto o niej wiedzieć, jeśli Twój model kiedyś przejdzie na nowszy mechanizm kalendarza w Power BI.

### 1.4 `PREVIOUSDAY` / `PREVIOUSMONTH` / `PREVIOUSQUARTER` / `PREVIOUSYEAR`

```dax
Sprzedaz Poprzedni Dzien =
CALCULATE ( [Sprzedaz Total], PREVIOUSDAY ( dim_Kalendarz[Data] ) )

Sprzedaz Poprzedni Kwartal =
CALCULATE ( [Sprzedaz Total], PREVIOUSQUARTER ( dim_Kalendarz[Data] ) )
```

Zachowują się jak `PARALLELPERIOD` ograniczony do konkretnej jednostki — zawsze zwracają pełny poprzedni dzień/miesiąc/kwartał/rok względem *ostatniej* daty w bieżącym kontekście. Czytelniejsze niż `PARALLELPERIOD(..., -1, ...)`, gdy nie potrzebujesz parametryzacji.

### 1.5 `NEXTDAY` / `NEXTMONTH` / `NEXTQUARTER` / `NEXTYEAR`

Lustrzane odbicie funkcji `PREVIOUS*` — przesunięcie wprzód. Przydatne np. przy analizie planu/budżetu:

```dax
Budzet Nastepny Kwartal =
CALCULATE ( [Budzet Total], NEXTQUARTER ( dim_Kalendarz[Data] ) )
```

---

## 2. Kumulacja od początku okresu — YTD / QTD / MTD

### 2.1 `DATESYTD` / `DATESQTD` / `DATESMTD`

```dax
Sprzedaz YTD =
CALCULATE ( [Sprzedaz Total], DATESYTD ( dim_Kalendarz[Data] ) )

Sprzedaz YTD (rok fiskalny konczacy sie 30 czerwca) =
CALCULATE ( [Sprzedaz Total], DATESYTD ( dim_Kalendarz[Data], "06-30" ) )
```

Drugi (opcjonalny) parametr to data końca roku fiskalnego w formacie `"MM-DD"`.

### 2.2 `TOTALYTD` / `TOTALQTD` / `TOTALMTD`

```dax
Sprzedaz YTD (skrot) =
TOTALYTD ( [Sprzedaz Total], dim_Kalendarz[Data] )

Sprzedaz YTD Tylko Premium =
TOTALYTD (
    [Sprzedaz Total],
    dim_Kalendarz[Data],
    dim_Klienci[Segment] = "Premium"
)
```

`TOTALYTD` przyjmuje tylko jeden dodatkowy argument filtru — `CALCULATE + DATESYTD` dowolnie wiele, co jest lepszym wyborem przy potrzebie łączenia YTD z kilkoma modyfikacjami filtru naraz.

---

## 3. Niestandardowe zakresy dat — podstawa wartości kroczących

### 3.1 `DATESBETWEEN` — zakres między dwiema konkretnymi datami

```dax
Sprzedaz H1 2025 =
CALCULATE (
    [Sprzedaz Total],
    DATESBETWEEN ( dim_Kalendarz[Data], DATE(2025,1,1), DATE(2025,6,30) )
)
```

**Specjalne znaczenie `BLANK()` jako argumentu:** `DATESBETWEEN` traktuje `BLANK()` jako "brak granicy" — użyte jako `StartDate` oznacza "od najwcześniejszej daty w tabeli", użyte jako `EndDate` oznacza "do najpóźniejszej daty":

```dax
Sprzedaz Do Konca Marca 2025 =
CALCULATE (
    [Sprzedaz Total],
    DATESBETWEEN ( dim_Kalendarz[Data], BLANK(), DATE(2025,3,31) )
)
```

To wygodny sposób na "wszystko od początku danych do X" bez potrzeby jawnego wpisywania pierwszej daty w modelu.

### 3.2 `DATESINPERIOD` — główne narzędzie do wartości kroczących

```dax
DATESINPERIOD ( kolumna_daty, data_startowa, liczba_okresow, interwal )
```

Punkt startowy jest dynamiczny (zwykle `MAX`/`LASTDATE` bieżącego kontekstu), zakres liczony jest względem niego.

**Uwaga o `BLANK()` w `DATESINPERIOD`:** w odróżnieniu od `DATESBETWEEN`, tu `BLANK()` jako `StartDate` **nie ma specjalnego znaczenia** — jest po prostu traktowany jako data sprzed roku 1900, co w praktyce daje ten sam efekt co "od początku tabeli dat" (bo żadna data w `dim_Kalendarz` nie będzie wcześniejsza), ale technicznie to inny mechanizm niż w `DATESBETWEEN` — warto o tym wiedzieć, żeby nie zakładać identycznego zachowania obu funkcji.

**Sprzedaż z ostatnich 30 dni (kroczące okno, nie kalendarzowy miesiąc):**

```dax
Sprzedaz Kroczace 30 Dni =
CALCULATE (
    [Sprzedaz Total],
    DATESINPERIOD ( dim_Kalendarz[Data], MAX ( dim_Kalendarz[Data] ), -30, DAY )
)
```

**Średnia krocząca 3-miesięczna:**

```dax
Srednia Kroczaca 3M (po miesiacach) =
AVERAGEX (
    VALUES ( dim_Kalendarz[RokMiesiac] ),
    CALCULATE (
        [Sprzedaz Total],
        DATESINPERIOD ( dim_Kalendarz[Data], MAX ( dim_Kalendarz[Data] ), -3, MONTH )
    )
)
```

**Suma krocząca 12-miesięczna (Trailing Twelve Months, TTM):**

```dax
Sprzedaz TTM =
CALCULATE (
    [Sprzedaz Total],
    DATESINPERIOD ( dim_Kalendarz[Data], MAX ( dim_Kalendarz[Data] ), -12, MONTH )
)
```

TTM eliminuje sezonowość widoczną w YTD (które "resetuje się" 1 stycznia).

---

## 4. Granice okresu

### 4.1 `STARTOFMONTH` / `STARTOFQUARTER` / `STARTOFYEAR`

```dax
Sprzedaz od Poczatku Kwartalu do Dzisiaj =
CALCULATE (
    [Sprzedaz Total],
    DATESBETWEEN ( dim_Kalendarz[Data], STARTOFQUARTER ( dim_Kalendarz[Data] ), MAX ( dim_Kalendarz[Data] ) )
)
```

### 4.2 `ENDOFMONTH` / `ENDOFQUARTER` / `ENDOFYEAR`

```dax
Sprzedaz na Koniec Miesiaca =
CALCULATE ( [Sprzedaz Total], ENDOFMONTH ( dim_Kalendarz[Data] ) )
```

---

## 5. Salda otwarcia/zamknięcia — miary "stanu", nie "przepływu"

```dax
Liczba Klientow Aktywnych (na koniec miesiaca) =
CLOSINGBALANCEMONTH (
    CALCULATE ( DISTINCTCOUNT ( fact_Sprzedaz[ID_Klienta] ) ),
    dim_Kalendarz[Data]
)

Liczba Klientow Aktywnych (na poczatek miesiaca) =
OPENINGBALANCEMONTH (
    CALCULATE ( DISTINCTCOUNT ( fact_Sprzedaz[ID_Klienta] ) ),
    dim_Kalendarz[Data]
)
```

`CLOSINGBALANCEMONTH` oblicza wyrażenie tak, jakby kontekstem był tylko ostatni dzień bieżącego miesiąca. Analogiczne pary: `CLOSINGBALANCEQUARTER`/`OPENINGBALANCEQUARTER`, `CLOSINGBALANCEYEAR`/`OPENINGBALANCEYEAR`.

**Test poprawności:** jeśli suma miary przez 12 miesięcy ma sens biznesowy (jak `[Sprzedaz Total]`) — nie potrzebujesz `CLOSINGBALANCE*`. Jeśli nie (stan magazynowy, liczba aktywnych klientów, saldo) — to sygnał, że jej potrzebujesz.

---

## 6. `FIRSTDATE`/`LASTDATE` vs `FIRSTNONBLANK`/`LASTNONBLANK` vs `FIRSTNONBLANKVALUE`/`LASTNONBLANKVALUE`

### 6.1 `FIRSTDATE` / `LASTDATE` — pierwsza/ostatnia data w kontekście, bez sprawdzania danych

```dax
Pierwszy Dzien Okresu = FIRSTDATE ( dim_Kalendarz[Data] )
Ostatni Dzien Okresu = LASTDATE ( dim_Kalendarz[Data] )
```

Zwracają pierwszą/ostatnią datę widoczną w bieżącym kontekście filtru **niezależnie od tego, czy w te dni była jakakolwiek sprzedaż**. `MAX(dim_Kalendarz[Data])` i `LASTDATE(dim_Kalendarz[Data])` dają w praktyce ten sam wynik przy poprawnej tabeli dat — SQLBI rekomenduje `MAX` jako czytelniejsze w nowoczesnym kodzie DAX.

### 6.2 `FIRSTNONBLANK` / `LASTNONBLANK` — pierwsza/ostatnia wartość kolumny, dla której wyrażenie NIE jest puste

```dax
Data Pierwszego Zakupu Klienta =
CALCULATE (
    FIRSTNONBLANK ( dim_Kalendarz[Data], CALCULATE ( COUNTROWS ( fact_Sprzedaz ) ) ),
    ALL ( dim_Kalendarz )
)
```

Iteruje po unikalnych wartościach kolumny i zwraca pierwszą, dla której wyrażenie (obliczone w kontekście tej wartości — Context Transition) daje wynik niepusty.

**Ważna uwaga: `FIRSTNONBLANK`/`LASTNONBLANK` nie wymagają kolumny dat** — pierwszy argument może być dowolnego typu, w tym tekstowy. Przydatne np. do znajdowania poprzedniego numeru zamówienia klienta, nie tylko poprzedniej daty (patrz sekcja 7.4).

### 6.3 `FIRSTNONBLANKVALUE` / `LASTNONBLANKVALUE` — zwraca WARTOŚĆ wyrażenia, nie datę

```dax
Liczba Klientow Aktywnych (ostatni znany stan) =
LASTNONBLANKVALUE (
    dim_Kalendarz[Data],
    CALCULATE ( DISTINCTCOUNT ( fact_Sprzedaz[ID_Klienta] ) )
)
```

Zwraca **wynik wyrażenia** obliczony dla ostatniej niepustej daty — wzorzec semi-additive measure, elastyczniejszy niż `CLOSINGBALANCE*`, bo nie ogranicza się do sztywnych granic miesiąca/kwartału/roku.

| | `CLOSINGBALANCEMONTH` | `LASTNONBLANKVALUE` |
|---|---|---|
| Punkt odniesienia | Zawsze ostatni kalendarzowy dzień miesiąca | Ostatni dzień z faktycznymi danymi |
| Ryzyko | Dla bieżącego, niezakończonego miesiąca może zwrócić `BLANK()` | Poprawnie "cofnie się" do ostatniej daty z danymi |
| Kiedy używać | Analiza zamkniętych, historycznych okresów | Dashboardy operacyjne, "stan na dziś" w trakcie trwającego okresu |

---

## 7. Optymalizacja `FIRSTNONBLANK`/`LASTNONBLANK`/`LASTNONBLANKVALUE` — kiedy są drogie i jak je zastąpić

To rozszerzenie sekcji 6, oparte na dedykowanej analizie wydajności SQLBI — istotne, bo wzorce z sekcji 6.2-6.3 są wygodne, ale **nie zawsze najszybsze możliwe**.

### 7.1 Dlaczego te funkcje bywają kosztowne

`FIRSTNONBLANK`/`LASTNONBLANK`/`*VALUE` to **iteratory** — dla każdej wartości w pierwszej kolumnie tworzą Row Context i ewaluują wyrażenie z drugiego argumentu, co przy odwołaniu do miary oznacza Context Transition (patrz przewodnik o Row/Filter Context). Gdy pierwszy argument to sama kolumna (nie tabela), DAX **niejawnie przepisuje** wywołanie, owijając kolumnę w `CALCULATETABLE(DISTINCT(...))` — czyli w praktyce budujesz i skanujesz potencjalnie dużą tabelę dat (np. wszystkie dni sprzed bieżącej daty — przy 10-letniej tabeli dat to tysiące wierszy) **dla każdej komórki raportu**.

### 7.2 Pierwsza, tania optymalizacja — nie licz tam, gdzie nie musisz

Jeśli miara jest widoczna tylko wtedy, gdy dla bieżącego kontekstu istnieją transakcje, opakuj całość w `IF(NOT ISEMPTY(fact_Sprzedaz), ...)` — dzięki temu kosztowne wyszukiwanie uruchamia się tylko dla komórek z realnymi danymi, nie dla każdej daty w całej tabeli kalendarza:

```dax
Data Poprzedniego Zakupu Klienta v1 =
IF (
    NOT ISEMPTY ( fact_Sprzedaz ),
    VAR PierwszaWidocznaData = MIN ( dim_Kalendarz[Data] )
    VAR PoprzedniaData =
        CALCULATE (
            LASTNONBLANK ( dim_Kalendarz[Data], [Sprzedaz Total] ),
            dim_Kalendarz[Data] < PierwszaWidocznaData
        )
    RETURN
        PoprzedniaData
)
```

### 7.3 Druga optymalizacja — zamiana `LASTNONBLANK`/`LASTNONBLANKVALUE` na `MAX` + `TREATAS`

Jeśli możesz przyjąć rozsądne założenie modelowe — **obecność wiersza w `fact_Sprzedaz` wystarczy, żeby uznać dany dzień za "niepusty"**, bez potrzeby faktycznego liczenia `[Sprzedaz Total]` dla każdej daty — zastąp iterator prostą agregacją `MAX` na kolumnie faktów, co eliminuje Context Transition w pętli:

```dax
-- Wersja zoptymalizowana (v3 w terminologii SQLBI) — dużo szybsza niż LASTNONBLANK
Data Poprzedniego Zakupu Klienta v3 =
IF (
    NOT ISEMPTY ( fact_Sprzedaz ),
    VAR PierwszaWidocznaData = MIN ( dim_Kalendarz[Data] )
    VAR PoprzedniaData =
        CALCULATE (
            MAX ( fact_Sprzedaz[Data] ),
            dim_Kalendarz[Data] < PierwszaWidocznaData
        )
    RETURN
        PoprzedniaData
)

Sprzedaz Poprzedniego Zakupu v3 =
VAR PierwszaWidocznaData = MIN ( dim_Kalendarz[Data] )
VAR PoprzedniaData =
    CALCULATE ( MAX ( fact_Sprzedaz[Data] ), dim_Kalendarz[Data] < PierwszaWidocznaData )
VAR FiltrPoprzedniaData =
    TREATAS ( { PoprzedniaData }, dim_Kalendarz[Data] )
VAR Wynik =
    CALCULATE ( [Sprzedaz Total], FiltrPoprzedniaData )
RETURN
    Wynik
```

`TREATAS` przenosi wartość `PoprzedniaData` (zwykły skalar, bez "lineage" do `dim_Kalendarz`) i traktuje ją jako filtr na kolumnie `dim_Kalendarz[Data]` — dzięki temu `CALCULATE` filtruje poprawnie, mimo że `PoprzedniaData` formalnie pochodzi z `fact_Sprzedaz[Data]`, nie z tabeli kalendarza.

**Wynik benchmarku SQLBI** (na modelu testowym, granularność dnia): wersja z `MAX`+`TREATAS` jest o **~20% szybsza niż `LASTNONBLANK`** i **~35% szybsza niż `LASTNONBLANKVALUE`** — co jest wynikiem kontrintuicyjnym, bo `LASTNONBLANKVALUE` "wygląda" na bardziej zoptymalizowaną (krótszy zapis, jedno wywołanie zamiast dwóch), ale w praktyce **nie unika kosztu iteracji**, tylko oszczędza Ci pisania kodu — silnik nadal iteruje po tej samej liczbie wierszy.

### 7.4 To samo bez dat — na przykładzie numeru zamówienia

`LASTNONBLANK`/`MAX`-based wzorzec działa identycznie na kolumnach nietekstowych związanych z porządkiem (np. numer zamówienia), nie tylko na datach — przydatne przy analizie "poprzednie zamówienie tego klienta", niezależnie od tego, którego dnia padło:

```dax
Poprzedni Numer Zamowienia (zoptymalizowany) =
IF (
    NOT ISEMPTY ( fact_Sprzedaz ),
    VAR PierwszyWidocznyNumer = MIN ( fact_Sprzedaz[Numer_Zamowienia] )
    VAR PoprzedniNumer =
        CALCULATE (
            MAX ( fact_Sprzedaz[Numer_Zamowienia] ),
            REMOVEFILTERS ( dim_Kalendarz ),
            fact_Sprzedaz[Numer_Zamowienia] < PierwszyWidocznyNumer
        )
    RETURN
        PoprzedniNumer
)
```

`REMOVEFILTERS(dim_Kalendarz)` jest tu potrzebny, żeby szukać poprzedniego zamówienia niezależnie od filtra daty widocznego akurat w tym wierszu raportu (zamówienia klienta mogą być rozrzucone na różne dni).

**Kiedy warto sięgnąć po tę optymalizację, a kiedy zostać przy `LASTNONBLANK`/`LASTNONBLANKVALUE`:** różnica wydajności jest zauważalna dopiero przy dużej granularności iteracji (dużo unikalnych dat/numerów zamówień w kontekście, np. widok bez filtra na pojedynczego klienta) — przy analizie pojedynczego klienta z niewielką liczbą transakcji różnica bywa nieodczuwalna (rząd milisekund). Zacznij od czytelnego `LASTNONBLANK`/`LASTNONBLANKVALUE`, zmierz w DAX Studio (Server Timings), i sięgnij po wersję z `MAX`+`TREATAS` tylko jeśli pomiar faktycznie pokaże problem.

---

## 8. `BLANK` w kolumnie dat — wpływ na funkcje Time Intelligence

To dotyczy scenariusza, w którym `fact_Sprzedaz` (albo inna tabela faktów) ma kolumnę daty, która **czasem jest pusta** — np. `Data_Dostawy` niewypełniona dla zamówień jeszcze niezrealizowanych, w odróżnieniu od zawsze wypełnionej `Data_Zamowienia`.

### 8.1 Dlaczego BLANK to poprawny wybór, nie problem do "naprawienia"

Jeśli rozważasz zastąpienie brakującej daty wartością zastępczą (np. datą "daleko w przyszłości", `9999-12-31`) — to zwykle gorsze rozwiązanie: albo musiałbyś sztucznie rozszerzyć `dim_Kalendarz` o dziesiątki tysięcy zbędnych dni (żeby uniknąć pustego wiersza po stronie "jeden" relacji), albo złamałbyś wymóg ciągłości kalendarza z sekcji 0. BLANK w kolumnie dat jest poprawną, zamierzoną reprezentacją braku informacji.

### 8.2 Efekt uboczny: pusty wiersz w tabeli dat przez nieprawidłową relację

Gdy `fact_Sprzedaz[Data_Dostawy]` zawiera puste wartości, a masz (nieaktywną, przez `USERELATIONSHIP`) relację do `dim_Kalendarz`, powstaje **dodatkowy pusty wiersz po stronie "jeden"** — to jest ten sam mechanizm "blank row" znany z relacji naruszających RI (patrz przewodnik DAX Studio, sekcja 6), tylko wywołany brakiem wartości zamiast błędem jakości danych. Ten pusty wiersz pojawi się w slicerach na `dim_Kalendarz`, nawet jeśli w ogóle nie używasz relacji po `Data_Dostawy` w bieżącym wizualu.

**Jak to ukryć w raporcie** (bez zmiany modelu): filtruj `dim_Kalendarz[Data] <> BLANK()` na poziomie strony/slicera — to propaguje się na całą tabelę kalendarza, nie tylko na kolumnę, w której zauważyłeś problem.

**Czego NIE robić:** zmiana relacji na many-to-many z pojedynczym kierunkiem filtrowania "żeby ukryć pusty wiersz" — SQLBI wprost odradza to podejście: total przestaje się zgadzać z sumą po latach (bo relacja ograniczona nie propaguje filtra "nie pusty" na Total), pogarsza wydajność i maskuje realne problemy jakości danych zamiast je ujawniać.

### 8.3 Pułapka: porównania `<` i `<=` z kolumną dat zawierającą BLANK

To jest najważniejsza praktyczna konsekwencja — **BLANK w kolumnie dat jest traktowany jako wartość mniejsza niż każda inna data** w porównaniach `<`/`<=`. Miara licząca "narastająco do bieżącej daty" przez filtr `dim_Kalendarz[Data] <= LastDateVisible` **niechcący włączy** wiersze z pustą datą dostawy do każdego okresu, zawyżając wynik od samego początku:

```dax
-- BŁĘDNE: BLANK w Data_Dostawy zawsze przechodzi przez "< LastDateVisible"
Sprzedaz Dostarczona Narastajaco (błędna) =
VAR LastDateVisible = MAX ( dim_Kalendarz[Data] )
RETURN
    CALCULATE (
        [Sprzedaz Dostarczona],
        USERELATIONSHIP ( fact_Sprzedaz[Data_Dostawy], dim_Kalendarz[Data] ),
        dim_Kalendarz[Data] <= LastDateVisible
    )

-- POPRAWNE: jawne wykluczenie BLANK
Sprzedaz Dostarczona Narastajaco (poprawna) =
VAR LastDateVisible = MAX ( dim_Kalendarz[Data] )
RETURN
    CALCULATE (
        [Sprzedaz Dostarczona],
        USERELATIONSHIP ( fact_Sprzedaz[Data_Dostawy], dim_Kalendarz[Data] ),
        NOT ISBLANK ( dim_Kalendarz[Data] ) && dim_Kalendarz[Data] <= LastDateVisible
    )
```

**Zasada ogólna:** zawsze dodawaj `NOT ISBLANK(<kolumna_daty>)` przy porównaniach z operatorami `<`/`<=`/`>`/`>=` na kolumnie, która może zawierać BLANK — dla `<`/`<=` jest to ściśle konieczne, dla `>`/`>=` teoretycznie mniej krytyczne, ale SQLBI rekomenduje stosować to konsekwentnie wszędzie ("better safe than sorry"), żeby nie musieć za każdym razem analizować, czy dany przypadek jest bezpieczny.

### 8.4 Pułapka w iteratorach po tabeli dat

`SUMX('Date', ...)` (odwołanie wprost do nazwy tabeli) **pomija** dodatkowy pusty wiersz powstały z nieprawidłowej relacji — jeśli chcesz go uwzględnić, iteruj po `VALUES('Date')` zamiast po samej tabeli:

```dax
-- pomija pusty wiersz Date (i powiązane z nim niedostarczone zamówienia)
Prowizja Dostawy =
SUMX ( dim_Kalendarz, [Sprzedaz Dostarczona] * 0.02 )

-- uwzględnia pusty wiersz
Prowizja Dostawy (z BLANK) =
SUMX ( VALUES ( dim_Kalendarz ), [Sprzedaz Dostarczona] * 0.02 )
```

To rozróżnienie bywa trudne do zauważenia w praktyce — różnica bez wyraźnej tabeli z widocznym pustym wierszem jest "cicha": wynik jest inny, ale bez ewidentnej przyczyny w wizualu.

### 8.5 BLANK jako argument `DATESBETWEEN`/`DATESINPERIOD` — patrz sekcja 3

Sekcje 3.1 i 3.2 opisują już specjalne (i różne między tymi dwiema funkcjami!) znaczenie `BLANK()` jako argumentu granicznego — warto przeczytać je łącznie z tą sekcją, bo to dwa różne konteksty użycia BLANK: BLANK jako brakująca wartość w danych (tu) vs BLANK jako świadomy argument funkcji (sekcja 3).

---

## 9. Porównywanie niestandardowych okresów o różnej długości

To osobny wzorzec (DAX Patterns, Marco Russo/Alberto Ferrari) — inny niż standardowe "rok do roku" z sekcji 1, bo pozwala użytkownikowi raportu **samodzielnie wybrać dwa dowolne okresy** do porównania (np. "sierpień 2025" vs "cały rok 2024"), niekoniecznie tej samej długości.

### 9.1 Problem, który to rozwiązuje

Standardowe funkcje z sekcji 1 (`SAMEPERIODLASTYEAR`, `DATEADD`) zakładają **stały, przewidywalny** offset (rok, miesiąc). Gdy chcesz dać użytkownikowi dwa niezależne slicery ("okres bieżący" i "okres porównawczy") o dowolnie różnej długości, prosta różnica sum wprowadza w błąd — porównanie "sierpień 2025 (31 dni)" vs "cały 2024 (365 dni)" bez normalizacji sugeruje spadek, mimo że to po prostu efekt różnej liczby dni.

### 9.2 Wymagana struktura modelu — druga tabela dat

Wzorzec wymaga **dwóch niezależnych tabel dat** w modelu: `dim_Kalendarz` (do wyboru okresu bieżącego) oraz druga, np. `dim_Kalendarz_Porownawczy` (do wyboru okresu porównawczego), obie połączone z `fact_Sprzedaz` — jedna aktywną relacją, druga nieaktywną (używaną przez `USERELATIONSHIP`).

### 9.3 Miara z normalizacją po liczbie dni

```dax
Sprzedaz Porownawcza (znormalizowana) =
VAR OkresBiezacy = VALUES ( dim_Kalendarz[Data] )
VAR OkresPorownawczy =
    CALCULATETABLE (
        VALUES ( dim_Kalendarz[Data] ),
        REMOVEFILTERS ( dim_Kalendarz ),
        USERELATIONSHIP ( dim_Kalendarz[Data], dim_Kalendarz_Porownawczy[Data] )
    )
VAR SprzedazPorownawcza = CALCULATE ( [Sprzedaz Total], OkresPorownawczy )
VAR DniBiezacy = COUNTROWS ( OkresBiezacy )
VAR DniPorownawczy = COUNTROWS ( OkresPorownawczy )
VAR SredniaDziennaPorownawcza = DIVIDE ( SprzedazPorownawcza, DniPorownawczy )
VAR WynikZnormalizowany = SredniaDziennaPorownawcza * DniBiezacy
RETURN
    WynikZnormalizowany
```

Logika: licz sprzedaż okresu porównawczego, podziel przez liczbę dni w tym okresie (średnia dzienna), pomnóż przez liczbę dni okresu bieżącego — otrzymujesz wartość okresu porównawczego **przeskalowaną do długości okresu bieżącego**, więc porównanie "sierpień 2025 vs znormalizowany 2024" jest uczciwe (mówi "gdyby 2024 trwał tyle samo dni co sierpień 2025, wyniósłby X").

**Kiedy sięgać po ten wzorzec, a kiedy wystarczą funkcje z sekcji 1:** jeśli porównanie zawsze dotyczy tego samego typu okresu (miesiąc do miesiąca, rok do roku) — sekcja 1 w zupełności wystarcza i jest prostsza. Ten wzorzec ma sens dopiero, gdy raport ma dawać użytkownikowi **swobodę wyboru dwóch niezależnych, potencjalnie nierównych okresów** przez slicery — typowy przypadek w raportach ad-hoc do analizy kampanii/promocji o różnym czasie trwania.

---

## 10. Podsumowanie — która funkcja do którego zadania

| Potrzebujesz | Funkcja |
|---|---|
| Porównanie rok do roku, ten sam okres | `SAMEPERIODLASTYEAR` |
| Przesunięcie o dowolny interwał, zachowując liczbę dat | `DATEADD` |
| Przesunięcie o pełny okres, niezależnie od zakresu w kontekście | `PARALLELPERIOD`, `PREVIOUSMONTH`/`PREVIOUSQUARTER`/`PREVIOUSYEAR` |
| Suma narastająco od początku roku/kwartału/miesiąca | `TOTALYTD`/`TOTALQTD`/`TOTALMTD` (lub `CALCULATE`+`DATESYTD`/`DATESQTD`/`DATESMTD`) |
| Dowolny, przesuwający się zakres N dni/miesięcy wstecz | `DATESINPERIOD` |
| Suma/średnia krocząca (TTM, średnia 3-miesięczna) | `DATESINPERIOD` wewnątrz `CALCULATE`/`AVERAGEX` |
| Data początku/końca miesiąca/kwartału/roku jako budulec | `STARTOFMONTH`/`ENDOFMONTH` itd. |
| Stan "na koniec/początek okresu" dla miar nieaddytywnych | `CLOSINGBALANCE*`/`OPENINGBALANCE*` |
| Pierwsza/ostatnia data w kontekście, bez sprawdzania danych | `FIRSTDATE`/`LASTDATE` (lub `MIN`/`MAX`) |
| Pierwsza/ostatnia data, w której faktycznie są dane | `FIRSTNONBLANK`/`LASTNONBLANK` |
| To samo, ale przy dużej granularności i wysokich wymaganiach wydajności | `MAX` + `TREATAS` zamiast `LASTNONBLANK`/`LASTNONBLANKVALUE` (sekcja 7) |
| Wartość dla ostatniego dnia z danymi — "ostatni znany stan" | `FIRSTNONBLANKVALUE`/`LASTNONBLANKVALUE` |
| Kolumna dat z brakującymi wartościami | Sekcja 8 — `NOT ISBLANK()` przy porównaniach, `VALUES()` w iteratorach |
| Porównanie dwóch dowolnych, niestandardowych okresów różnej długości | Wzorzec z sekcji 9 — druga tabela dat + normalizacja po liczbie dni |

---


