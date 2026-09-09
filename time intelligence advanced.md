# Time intelligence w DAX poza podręcznikowym SAMEPERIODLASTYEAR

> **Uwaga metodologiczna:** jak w pozostałych dokumentach o DAX w tym repo — nie mam tu środowiska Power BI/DAX Studio do wykonania i zweryfikowania tych wzorców na żywo. Mechanika opisana niżej jest zgodna z udokumentowanym zachowaniem funkcji time intelligence w silniku DAX, ale zanim wdrożysz coś z tego w produkcji, przetestuj na swoim modelu — zwłaszcza fragmenty dotyczące roku fiskalnego, gdzie drobne różnice w konstrukcji tabeli dat realnie zmieniają wynik.

## Fundament, o którym wbudowane funkcje milcząco zakładają

Wszystkie standardowe funkcje time intelligence (`SAMEPERIODLASTYEAR`, `DATEADD`, `TOTALYTD`, `PARALLELPERIOD`...) działają poprawnie WYŁĄCZNIE, gdy spełnione są dwa warunki:
1. Istnieje osobna **tabela dat** — ciągła (bez luk, jeden wiersz na każdy dzień kalendarzowy w zakresie), oznaczona w modelu jako **Date Table** (z unikalną kolumną typu data).
2. Tabela dat jest połączona relacją z tabelą faktów po kolumnie daty.

Jeśli któryś z tych warunków nie jest spełniony (tabela dat zbudowana z samych dat WYSTĘPUJĄCYCH w transakcjach, więc z lukami w weekendy/święta), wbudowane funkcje mogą dawać wyniki, które wyglądają na policzone poprawnie, ale są ciche błędne — bo \"przesunięcie o rok\" czy \"suma od początku roku\" liczy się względem tabeli dat, nie względem rzeczywistego kalendarza.

---

## Szybki przegląd: cztery wbudowane funkcje i ich różnice

```dax
-- SAMEPERIODLASTYEAR - skrót, zawsze dokładnie "ten sam okres rok wstecz"
Sales LY = CALCULATE([Total Sales], SAMEPERIODLASTYEAR('Date'[Date]))

-- DATEADD - elastyczny interwał (dni/miesiące/kwartały/lata), nie tylko rok
Sales 3M Ago = CALCULATE([Total Sales], DATEADD('Date'[Date], -3, MONTH))

-- PARALLELPERIOD - jak DATEADD, ale przesuwa CAŁE okresy kalendarzowe
-- (np. cały poprzedni miesiąc, nawet jeśli bieżący filtr to tylko kilka dni tego miesiąca)
Sales Prev Month (full) = CALCULATE([Total Sales], PARALLELPERIOD('Date'[Date], -1, MONTH))

-- TOTALYTD - suma narastająca od początku roku (odpowiednik running total z notatek o pandas/T-SQL)
Sales YTD = TOTALYTD([Total Sales], 'Date'[Date])
```

**`SAMEPERIODLASTYEAR`** to w istocie skrót dla `DATEADD(..., -1, YEAR)` — te dwie formy są równoważne dla przesunięcia rocznego. **`PARALLELPERIOD`** różni się od `DATEADD` tym, że zawsze zwraca PEŁNY przesunięty okres, niezależnie od tego, jak dokładnie wygląda bieżący filtr dat — istotna różnica przy częściowych okresach (patrz niżej).

---

## Rok fiskalny inny niż kalendarzowy

`TOTALYTD` przyjmuje opcjonalny trzeci argument — datę końca roku fiskalnego:

```dax
-- Rok fiskalny konczacy sie 31 marca (typowy np. w niektorych branzach/krajach)
Sales FYTD = TOTALYTD([Total Sales], 'Date'[Date], "3/31")
```

To rozwiązuje prosty przypadek — rok fiskalny przesunięty o stałą liczbę miesięcy względem kalendarzowego, ale wciąż trwający 12 miesięcy w standardowym układzie. **Nie rozwiązuje** bardziej złożonych kalendarzy fiskalnych (np. układ 4-4-5 tygodni, popularny w handlu detalicznym, gdzie miesiące fiskalne nie pokrywają się z kalendarzowymi wcale). Dla takich przypadków standardowe rozwiązanie to:

1. Dodanie do tabeli dat WŁASNYCH kolumn: `FiscalYear`, `FiscalQuarter`, `FiscalMonth`, `FiscalMonthNumberInYear` — obliczonych raz, w Power Query albo jako kolumny obliczeniowe.
2. Budowanie miar YTD/QTD ręcznie, przez `CALCULATE` z filtrem na tych kolumnach zamiast wbudowanych funkcji:

```dax
Sales Fiscal YTD (recznie) =
VAR CurrentFiscalYear = MAX('Date'[FiscalYear])
VAR CurrentFiscalMonthNum = MAX('Date'[FiscalMonthNumberInYear])
RETURN
CALCULATE(
    [Total Sales],
    'Date'[FiscalYear] = CurrentFiscalYear,
    'Date'[FiscalMonthNumberInYear] <= CurrentFiscalMonthNum
)
```

---

## Tabela dat z lukami — cichy błąd, nie awaria

Jeśli tabela dat powstała np. przez `VALUES('Sales'[TransactionDate])` zamiast `CALENDAR`/`CALENDARAUTO`, brakuje w niej dni bez żadnej transakcji (typowo weekendy, święta, dni bez sprzedaży). Konsekwencja: `SAMEPERIODLASTYEAR` licząc \"dzień odpowiadający rok wcześniej\" może nie znaleźć dokładnie odpowiadającej daty, a `TOTALYTD` może zaniżać sumę, jeśli w środku okresu brakuje dni w samej tabeli dat (nie w tabeli faktów — to dwie różne rzeczy, patrz Pułapka 2).

**Zasada:** tabela dat ZAWSZE budowana jako ciągła (`CALENDAR(MIN_DATA, MAX_DATA)` albo `CALENDARAUTO()`), niezależnie od tego, czy w każdym dniu faktycznie wystąpiła transakcja. Brak transakcji w danym dniu to informacja (\"sprzedaż = 0 lub pusta\"), nie powód, żeby dzień w ogóle nie istniał w kalendarzu.

---

## Częściowe okresy: pułapka porównań rok-do-roku w trakcie miesiąca/roku

Jeśli raport jest generowany np. 15. dnia miesiąca, `TOTALYTD` dla bieżącego roku naturalnie obejmuje mniej dni niż analogiczna miara dla roku ubiegłego (który ma \"pełne\" dane za cały analogiczny okres, jeśli patrzysz na dane historyczne bez świadomego obcięcia). Prosta różnica `Sales YTD - Sales YTD LY` w takiej sytuacji **zaniża** rzeczywistą dynamikę — porównujesz 15 dni bieżącego roku z pełnym miesiącem/rokiem poprzedniego, jeśli nie obetniesz świadomie danych historycznych do tego samego \"punktu w czasie\".

```dax
-- Punkt odniesienia: ostatni dzień z FAKTYCZNYMI danymi w bieżącym okresie
VAR LastDateWithData = CALCULATE(MAX('Sales'[TransactionDate]), ALL('Date'))
-- Miara LY obcięta do TEGO SAMEGO dnia w roku ubiegłym, nie do końca całego okresu
Sales YTD LY (comparable) =
VAR LastDateWithData = CALCULATE(MAX('Sales'[TransactionDate]), ALL('Date'))
VAR LastDateLY = SAMEPERIODLASTYEAR(LastDateWithData)
RETURN
CALCULATE(
    [Total Sales],
    DATESYTD('Date'[Date]),
    'Date'[Date] <= LastDateLY
)
```

---

## Pułapki

### Pułapka 1 — tabela dat nieoznaczona jako Date Table w modelu

Samo posiadanie kolumny z datami nie wystarczy — bez jawnego oznaczenia tabeli jako **Date Table** w modelu (z jednoznacznie wskazaną kolumną daty), silnik może nie stosować pełnej, poprawnej logiki niektórych funkcji time intelligence, albo zgłaszać ostrzeżenia przy walidacji miar.

### Pułapka 2 — luki w tabeli FAKTÓW mylone z lukami w tabeli DAT

To dwie zupełnie różne rzeczy. Brak transakcji w danym dniu (tabela faktów) jest NORMALNY i oczekiwany — miara powinna wtedy zwrócić `0`/`BLANK()`. Brak dnia W TABELI DAT to błąd konstrukcji modelu — dokładnie ten opisany wyżej. Objawy bywają podobne (\"miara wygląda dziwnie w okolicy tej daty\"), ale przyczyna i naprawa są różne.

### Pułapka 3 — rok przestępny i inne osobliwości kalendarzowe

Porównanie \"29 lutego\" rok do roku w roku nieprzestępnym nie ma jednoznacznej, uniwersalnej odpowiedzi (czy odpowiednikiem jest 28 lutego, czy 1 marca?) — `SAMEPERIODLASTYEAR` ma tu określone, udokumentowane zachowanie, ale warto świadomie sprawdzić je na własnym modelu, jeśli raportujesz na poziomie dziennym w okolicach przełomu lutego/marca.

---

## Podsumowanie

| Sytuacja | Rozwiązanie |
|---|---|
| Standardowy rok kalendarzowy, ciągła tabela dat | Wbudowane funkcje (`SAMEPERIODLASTYEAR`, `TOTALYTD`) wystarczą |
| Rok fiskalny przesunięty o stałą liczbę miesięcy | `TOTALYTD(..., "MM/DD")` z argumentem końca roku |
| Kalendarz fiskalny 4-4-5 lub inny niestandardowy | Własne kolumny `FiscalYear`/`FiscalMonth` w tabeli dat + ręczne `CALCULATE` |
| Porównanie rok-do-roku w trakcie niedokończonego okresu | Jawne obcięcie obu okresów do tego samego \"punktu w czasie\" |
| Tabela dat budowana z dat transakcji | Zawsze zamień na `CALENDAR`/`CALENDARAUTO` — ciągłość jest wymogiem, nie opcją |

**Wniosek:** wbudowane funkcje time intelligence są niezawodne dokładnie w takim stopniu, w jakim niezawodna jest leżąca pod nimi tabela dat. Większość \"dziwnych\" wyników time intelligence w praktyce to nie błąd funkcji, tylko luka lub nieciągłość w tabeli dat, na której te funkcje operują.
