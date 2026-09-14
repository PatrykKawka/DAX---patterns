# DAX Studio — kompletny przewodnik: analiza modelu, wydajność, testowanie miar

---

## 1. Połączenie z modelem

- **Connect → Power BI / Analysis Services** — DAX Studio wykrywa otwarte instancje Power BI Desktop (lokalny port SSAS) i pliki `.pbix` z otwartym modelem. Możesz też połączyć się z Power BI Service (Premium/Fabric) albo lokalnym SSAS Tabular.
- Po połączeniu masz dostęp do **Metadata pane** (lewa strona) — drzewo tabel, kolumn, miar, hierarchii i perspektyw.
- **Ważne dla Twojego workflow**: DAX Studio łączy się *na żywo* z silnikiem — zmiany w modelu (np. w Tabular Editor) widoczne są po odświeżeniu metadanych (`Refresh` w metadata panel), bez potrzeby zamykania sesji.

---

## 2. Architektura silnika — jak Power BI faktycznie przechowuje i przetwarza dane

Zanim zaczniesz mierzyć wydajność, warto rozumieć, co dzieje się "pod spodem" — to tłumaczy, dlaczego pewne wzorce DAX są tanie, a inne kosztowne.

### 2.1 VertiPaq (xVelocity) — silnik pamięciowy dla trybu Import

To domyślny silnik przy imporcie danych do Power BI (Twój przypadek dla `fact_Sprzedaz` itd.). Kluczowe cechy:

- **Column store, nie row store** — każda kolumna jest przechowywana i kompresowana osobno, niezależnie od pozostałych. To dlatego szerokie tabele faktów z wieloma rzadko używanymi kolumnami "kosztują" tylko za te kolumny, które faktycznie skanujesz w zapytaniu — silnik nie musi czytać całego wiersza.
- **Segmentacja** — dane w tabeli dzielone są na segmenty (domyślnie **ok. 8 mln wierszy** na segment w silniku Analysis Services/Power BI Premium; w Power BI Desktop/Pro segmentacja też występuje, ale rzadziej ma znaczenie przy mniejszych wolumenach). Segmenty są kompresowane niezależnie i mogą być skanowane równolegle przez Storage Engine — to jeden z powodów, dla których SE jest wielowątkowy, a FE nie.
- **Dictionary encoding** — dla kolumn tekstowych i kolumn o wysokiej kardynalności silnik buduje słownik unikalnych wartości i przechowuje wiersze jako wektor indeksów do słownika (liczby całkowite zamiast tekstu). Im mniej unikalnych wartości, tym mniejszy słownik i lepsza kompresja.
- **Value encoding** — dla kolumn liczbowych o niskiej-średniej kardynalności silnik może pominąć słownik i kodować wartość bezpośrednio (czasem z przesunięciem/skalowaniem, np. przechowując różnicę względem minimum). To najtańszy typ kompresji.
- **RLE (Run-Length Encoding)** — gdy dane są posortowane/powtarzalne (np. kolumna daty w tabeli faktów posortowanej chronologicznie), silnik dodatkowo kompresuje powtarzające się sekwencje. To jeden z powodów, dla których **kolejność kolumn przy imporcie i sortowanie danych źródłowych** (np. `ORDER BY` w widoku SQL zasilającym `fact_Sprzedaz`) może realnie zmniejszyć rozmiar modelu.

### 2.2 Formula Engine vs Storage Engine — pełniejszy obraz

- **Storage Engine (SE)** — wielowątkowy, operuje na skompresowanych danych VertiPaq, wykonuje skany, filtrowanie na poziomie kolumn, proste agregacje (SUM, COUNT, MIN, MAX, DISTINCTCOUNT). Generuje zapytania w wewnętrznym języku **xmSQL** (widoczne w Server Timings) — to nie jest prawdziwy SQL, ale czytelna reprezentacja tego, co SE faktycznie robi.
- **Formula Engine (FE)** — jednowątkowy, interpretuje logikę DAX, którą SE nie potrafi wykonać natywnie (iteracje z warunkami, `RANKX`, zagnieżdżone `CALCULATE` z modyfikacją kontekstu, funkcje czasowe, część funkcji tekstowych). FE "zamawia" dane z SE, po czym łączy/przetwarza wyniki.
- **Konsekwencja praktyczna**: SE skaluje się z liczbą rdzeni, FE nie. Miara, która generuje dużo pracy FE, nie przyspieszy nawet na potężnym sprzęcie — dlatego wysoki % FE w Server Timings to sygnał do przepisania logiki, a nie do dodania mocy obliczeniowej.

### 2.3 Tryby przechowywania danych — Import, DirectQuery, Dual, Composite

- **Import** — dane fizycznie w VertiPaq, jak opisano wyżej. Najszybszy tryb dla analityki, ale wymaga odświeżania.
- **DirectQuery** — brak kopii danych w VertiPaq; każde zapytanie DAX jest tłumaczone na SQL i wysyłane do źródła (np. SQL Server) w czasie rzeczywistym. Storage Engine w tym trybie to *silnik źródła*, nie VertiPaq — DAX Studio pokaże wygenerowany SQL zamiast xmSQL. Wydajność zależy od optymalizacji bazy źródłowej (indeksy, statystyki), nie od VertiPaq.
- **Dual** — tabela (zwykle wymiar) dostępna jednocześnie jako Import i DirectQuery; silnik wybiera tryb zależnie od kontekstu zapytania (np. przy łączeniu z tabelą faktów w DirectQuery, wymiar w Dual też będzie odpytany przez DirectQuery, żeby uniknąć niespójności).
- **Composite model** — mieszanka trybów w jednym modelu (część tabel Import, część DirectQuery). To tu najczęściej pojawiają się problemy z **RI Violation** (patrz sekcja 5) i wymuszonymi cross-source joinami, które bywają bardzo kosztowne.

Dla Twojego stacku (Power BI + SQL Server): jeśli w ogóle rozważasz DirectQuery/Composite dla części modelu (np. świeże dane transakcyjne łączone z historycznym Import), DAX Studio jest jedynym narzędziem, które pokaże Ci realnie wygenerowany SQL i pozwoli ocenić, czy zapytanie faktycznie foldowało się dobrze do źródła.

---

## 3. Analiza struktury modelu — VertiPaq Analyzer

**Advanced → View Metrics** (`Model Metrics`) generuje raport z:

| Sekcja | Co pokazuje | Na co patrzeć |
|---|---|---|
| **Tables** | Rozmiar tabeli (in-memory), liczba wierszy, % całego modelu | Które tabele dominują rozmiarem |
| **Columns** | Rozmiar kolumny, kardynalność, encoding, % skompresowania | Kolumny o wysokiej kardynalności i dużym rozmiarze |
| **Relationships** | Rozmiar tabeli po stronie "wiele", kardynalność kluczy | Relacje o bardzo wysokiej kardynalności |
| **Hierarchies** | Rozmiar struktur hierarchii | Rzadziej krytyczne |

**Konkretne wartości, które powinny Cię niepokoić:**

- **Kolumna stanowiąca >20-25% rozmiaru całej tabeli faktów** — zwykle sygnał, że to kolumna o zbyt wysokiej kardynalności (np. ID transakcji, timestamp z sekundami) i albo nie jest potrzebna w modelu, albo powinna być rozbita (np. data + godzina osobno zamiast pełnego datetime co niszczy RLE).
- **Kardynalność kolumny >1 mln unikalnych wartości** przy jednoczesnym dużym rozmiarze — realny kandydat do redukcji granularności lub przeniesienia poza model (np. zostawienie w źródle, dociąganie przez drillthrough zamiast trzymania w VertiPaq).
- **Współczynnik kompresji poniżej ok. 3:1-4:1** dla kolumny, która "wygląda" na powtarzalną (np. kategoria, status) — to znak, że encoding poszedł w złą stronę (często przez zbyt dużą precyzję liczb zmiennoprzecinkowych albo brak sortowania danych źródłowych).
- **Kolumna numeryczna zakodowana jako Hash zamiast Value** — sprawdź typ danych (czy to przypadkiem nie decimal/float z nadmiarową precyzją, którą można zaokrąglić bez utraty sensu biznesowego).

Eksport wyników (`Export Model Metrics`) pozwala śledzić zmiany rozmiaru modelu między iteracjami.

---

## 4. Testowanie i pomiar wydajności miar — Server Timings

**Home → Server Timings** (włącz przed uruchomieniem zapytania) rejestruje podział czasu na FE i SE.

**Kluczowe metryki i orientacyjne progi:**

| Metryka | Sygnał ostrzegawczy | Interpretacja |
|---|---|---|
| **% Formula Engine** | >40-50% czasu całkowitego | Logika DAX jest "droga" — kandydat do przepisania wzorca, nie do dodania mocy obliczeniowej |
| **Liczba SE Queries** | >10-15 zapytań dla jednej, pojedynczej miary bez złożonych zależności | Możliwa nadmiarowa iteracja/wielokrotne skanowanie tych samych tabel |
| **SE Cache hit** | 0% przy powtórnym uruchomieniu identycznego zapytania | Sprawdź, czy nie testujesz przypadkiem z włączonym `Clear Cache` przy każdym uruchomieniu (to normalne w testach, ale mylące jeśli porównujesz z realnym użyciem raportu, gdzie cache pomaga) |
| **Total duration** dla pojedynczego wizuala | >1-2s w typowym raporcie interaktywnym | Microsoft rekomenduje orientacyjnie <1s dla płynnego UX, do ok. 5s jako granica akceptowalności — powyżej tego użytkownicy realnie to odczuwają |

**Zawsze `Clear Cache` przed testem porównawczym** — inaczej drugie uruchomienie tej samej miary będzie sztucznie szybsze przez trafienie w SE cache, co zafałszuje porównanie wariantów.

**Test A/B alternatywnych wzorców:**

```dax
-- Wariant A: RANKX + FILTER
DEFINE
    MEASURE fact_Sprzedaz[TOP10_A] =
        CALCULATE (
            [Sprzedaz Total],
            FILTER (
                VALUES ( dim_Klienci[ID_Klienta] ),
                RANKX ( ALL ( dim_Klienci[ID_Klienta] ), [Sprzedaz Total] ) <= 10
            )
        )
EVALUATE { [TOP10_A] }

-- Wariant B: TOPN (zwykle tańszy, bo unika RANKX per wiersz)
DEFINE
    MEASURE fact_Sprzedaz[TOP10_B] =
        VAR TopKlienci =
            TOPN ( 10, VALUES ( dim_Klienci[ID_Klienta] ), [Sprzedaz Total], DESC )
        RETURN
            CALCULATE ( [Sprzedaz Total], TopKlienci )
EVALUATE { [TOP10_B] }
```

`RANKX` w Wariancie A wymusza obliczenie rangi dla *każdego* klienta w kontekście `ALL`, co przy dużej tabeli wymiaru generuje znacznie więcej pracy FE niż `TOPN`, które od razu operuje na już posortowanym podzbiorze. Różnica typowo widoczna w liczbie SE queries i w % FE — na Twoim modelu warto to zmierzyć dosłownie tym patternem.

Jeśli masz kilka wariantów w jednym pliku, **Home → All Queries** wykona je sekwencyjnie i pokaże zbiorczy raport czasów.

---

## 5. CallbackDataID — najczęściej pomijany, a bardzo kosztowny problem

**Co to jest:** gdy Storage Engine skanuje dane, ale natrafia na fragment logiki DAX, którego nie potrafi wykonać natywnie (np. nietypowa funkcja tekstowa, warunek wymagający wywołania Formula Engine dla każdego wiersza/grupy), musi "oddzwonić" do FE w trakcie skanu. To wywołanie zwrotne widoczne jest w xmSQL (zakładka Server Timings → szczegóły zapytania SE) jako funkcja **`CallbackDataID`**.

**Dlaczego to boli:** CallbackDataID oznacza, że SE traci swoją główną przewagę — działanie wsadowe na skompresowanych danych — i zamiast tego wykonuje operację *wiersz po wierszu*, wywołując wolniejszy, jednowątkowy FE dla każdego rekordu (albo każdej grupy). Na małej tabeli niewidoczne, na tabeli faktów z milionami wierszy potrafi zamienić zapytanie z ułamka sekundy w kilkanaście sekund.

**Typowe przyczyny w praktyce:**
- Użycie funkcji, które nie foldują się do SE wewnątrz `FILTER`/`CALCULATE` operującego na tabeli faktów (np. `SEARCH`, `FORMAT`, niektóre funkcje daty stosowane wiersz po wierszu zamiast na poziomie kontekstu filtru).
- Warunki logiczne mieszające kolumny z różnych tabel w sposób wymuszający obliczenie skalarnego wyniku per wiersz zamiast operacji na poziomie kolumny.
- Iteratory (`SUMX`, `FILTER`) z wyrażeniem, które samo w sobie wymaga FE (np. wywołanie innej miary CALCULATE-owanej wewnątrz iteratora na dużej tabeli).

**Jak wykryć:** w Server Timings kliknij na zapytanie SE (dolna lista) i sprawdź treść xmSQL — obecność `CallbackDataID` w tekście to jednoznaczny sygnał. Warto też zwrócić uwagę na **czas trwania pojedynczego SE query nieproporcjonalny do liczby przetworzonych wierszy** — jeśli SE query skanujące relatywnie mało danych trwa podejrzanie długo, to częsty odcisk callbacku.

**Jak sobie radzić:**
1. Przenieś logikę wymagającą FE poza pętlę skanowania — np. oblicz warunek raz jako zmienną (`VAR`) na poziomie kontekstu filtru zamiast oceniać go per wiersz.
2. Zamień funkcje niefoldowalne na ich odpowiedniki natywne dla SE, jeśli istnieją (np. zamiast złożonych warunków tekstowych w `FILTER`, rozważ przygotowanie flagi/kolumny obliczonej w Power Query/SQL zamiast liczyć ją w DAX per wiersz).
3. Jeśli logika biznesowo wymaga oceny per wiersz (np. złożone reguły klasyfikacji klienta), rozważ **przeniesienie tej kolumny do warstwy SQL/Power Query jako kolumnę obliczoną przy ładowaniu**, zamiast liczyć ją w locie w DAX — to jest dokładnie przypadek, gdzie "zrób to w SQL, nie w DAX" ma twarde uzasadnienie wydajnościowe, nie tylko stylistyczne.

---

## 6. RI Violation (Referential Integrity Violation)

**Co to jest:** VertiPaq domyślnie **nie zakłada**, że każda wartość klucza obcego w tabeli faktów ma odpowiadający wiersz w tabeli wymiaru (czyli że relacja jest "czysta"). Jeśli silnik nie może zagwarantować pełnej integralności referencyjnej, musi traktować join jako potencjalny outer join (uwzględniając możliwe niedopasowane/puste wartości), zamiast czystego inner joina — to dodatkowy narzut na każdym zapytaniu przechodzącym przez tę relację.

**Gdzie to widać:** w **Query Plan** (Physical Plan) jako adnotacja przy operacji join/scan wskazująca, że relacja może naruszać RI (referential integrity) — silnik dokłada dodatkowe sprawdzenie obecności "blank row" (niewidocznego wiersza reprezentującego niedopasowane klucze) w tabeli wymiaru.

**Kiedy to występuje:**
- **Import mode**: zawsze technicznie możliwe, dopóki nie masz absolutnej pewności co do jakości danych — silnik i tak wykonuje sprawdzenie, ale narzut jest zwykle niewielki przy dobrze zbudowanym modelu gwiazdy z czystymi kluczami.
- **DirectQuery / Composite model**: znacznie ważniejsze — tu masz opcję **"Assume Referential Integrity"** przy definicji relacji. Gdy włączona, silnik generuje INNER JOIN zamiast OUTER JOIN w zapytaniu SQL wysyłanym do źródła, co bywa istotnie szybsze po stronie bazy danych, zwłaszcza przy dużych tabelach faktów.

**Jak sobie radzić:**
1. W Import mode: zadbaj, żeby `fact_Sprzedaz[ID_Klienta]`, `[ID_Placowki]`, `[ID_Produktu]` nie zawierały wartości sierocych (bez odpowiednika w wymiarze) — najlepiej wymuszone na poziomie SQL (FK constraint lub walidacja w procesie ETL) zanim dane trafią do Power Query.
2. W DirectQuery/Composite: jeśli masz pewność co do czystości kluczy (np. FK constraint w SQL Server), **zaznacz "Assume Referential Integrity"** na relacji — to bezpośrednio wpływa na wygenerowany SQL i widoczne jest w DAX Studio jako zmiana z LEFT JOIN na INNER JOIN w podglądzie zapytania źródłowego.
3. Zweryfikuj w DAX Studio konkretnie: uruchom zapytanie z `EVALUATE`, sprawdź Physical Query Plan pod kątem adnotacji o RI, porównaj czas przed i po włączeniu tej opcji (jeśli masz Composite/DirectQuery).

---

## 7. Query Plan — dogłębna analiza fizycznego i logicznego planu

**Home → Query Plan** pokazuje dwa widoki po uruchomieniu zapytania (`Query Plan` musi być włączony przed `Run`, podobnie jak Server Timings).

### 7.1 Logical Query Plan

Wysokopoziomowa reprezentacja *co* silnik ma zrobić, w kolejności logicznej — bliżej odzwierciedla strukturę wyrażenia DAX niż faktyczne wykonanie. Operacje typowo widoczne:

- **Scan_Vertipaq** — odczyt danych z tabeli/kolumny.
- **Filter** — zawężenie wierszy wg warunku.
- **AggregationSpool / Sum_Vertipaq** — agregacja.
- **GroupBy_Vertipaq** — grupowanie (np. bazowe dla `SUMMARIZE`/`SUMMARIZECOLUMNS`).

To warstwa bardziej "deklaratywna" — dobra do szybkiego zrozumienia intencji, ale nie pokazuje realnego kosztu.

### 7.2 Physical Query Plan — tu jest realna diagnostyka

Pokazuje faktycznie wykonane operacje z liczbą przetworzonych wierszy (**`Records`**) na każdym etapie. To jest odpowiednik execution plan w SQL Server — i czytasz go analogicznie.

**Na co patrzeć konkretnie:**

- **`Records` nieproporcjonalnie duże względem finalnego wyniku** — np. plan skanuje 5 mln wierszy, żeby zwrócić 10 wierszy wyniku. To sygnał, że filtrowanie nie zostało "zepchnięte" (pushed down) wystarczająco wcześnie w planie — silnik materializuje dużo danych pośrednich zanim je odrzuci.
- **Powtarzające się operacje `Spool`** (materializacja tabeli pośredniej w pamięci) — częsty efekt zagnieżdżonych `CALCULATE`/`FILTER`, gdzie ten sam podzbiór danych jest budowany wielokrotnie zamiast raz i buforowany w zmiennej (`VAR`).
- **`Cache` w planie** — dobry znak, oznacza że silnik wykorzystał wynik pośredni bez ponownego przeliczania.
- Adnotacje sugerujące **RI Violation** (opisane w sekcji 6) — dodatkowy narzut na relacjach.
- Wywołania **`CallbackDataID`** widoczne pośrednio przez nietypowo drobnoziarnisty wzorzec skanowania (wiele małych operacji zamiast jednego wsadowego skanu) — potwierdzenie tego, co zobaczysz najpierw w xmSQL (sekcja 5).

**Przykład interpretacji (uproszczony, koncepcyjnie):**

```
Physical Query Plan dla źle napisanej miary TOP10% (z FILTER + RANKX per wiersz):
  VertiPaq Scan: fact_Sprzedaz     Records: 2 450 000
  Spool (Lookup): dim_Klienci      Records: 180 000   <- policzone RANKX dla WSZYSTKICH klientów
  Spool (Lookup): dim_Klienci      Records: 180 000   <- powtórzone przy każdej zmianie filtru z wizuala
  AggregationSpool: SUM            Records: 10

Physical Query Plan dla wersji z TOPN:
  VertiPaq Scan: fact_Sprzedaz     Records: 2 450 000
  Cache: TopN result               Records: 10        <- silnik od razu operuje na 10 wierszach
  AggregationSpool: SUM            Records: 10
```

Różnica: w wersji A silnik materializuje ranking dla całej populacji klientów (180k `Spool`), mimo że interesuje Cię tylko 10 — to bezpośrednio przekłada się na czas FE. W wersji B `TOPN` pozwala silnikowi ograniczyć się do właściwego podzbioru dużo wcześniej w planie.

**Praktyczna zasada:** nie analizuj Query Plan dla każdej miary — sięgaj po niego dopiero gdy Server Timings wskaże konkretny problem (wysoki FE%, dużo SE queries, albo podejrzenie CallbackDataID/RI Violation) i chcesz precyzyjnie zlokalizować, na którym etapie planu następuje przeciążenie.

---

## 8. Formatowanie i porządkowanie kodu DAX

- **Format Query** (Ctrl+Shift+F) — integracja z DAX Formatter (Sql BI).
- **Edit → Comment/Uncomment** (Ctrl+K/Ctrl+U) — przydatne przy iteracyjnym testowaniu wariantów miary w jednym oknie zapytania.

---

## 9. Testowanie miar bez wdrażania do modelu — `DEFINE MEASURE`

```dax
DEFINE
    MEASURE fact_Sprzedaz[Sprzedaz TOP10 Test] =
        VAR TopKlienci =
            TOPN ( 10, VALUES ( dim_Klienci[ID_Klienta] ), [Sprzedaz Total], DESC )
        RETURN
            CALCULATE ( [Sprzedaz Total], TopKlienci )

EVALUATE
SUMMARIZECOLUMNS(
    dim_Kalendarz[Rok],
    "Sprzedaz TOP10", [Sprzedaz TOP10 Test]
)
ORDER BY dim_Kalendarz[Rok]
```

To pozwala testować logikę i wydajność (przez Server Timings i Query Plan) zanim zapiszesz zmianę w modelu.

---

## 10. Eksport wyników i integracja z Excel

- **Output → Excel / CSV / Linked Excel Table** — eksport wyniku zapytania.
- **Copy as JSON / Copy as INSERT** — przydatne do dokumentacji.

---

## 11. Trace / All Queries — testowanie na realnym obciążeniu z Power BI

**Home → All Queries** loguje zapytania DAX faktycznie wysyłane przez Power BI Desktop przy interakcji z wizualami — realny kontekst filtru wygenerowany przez wizual, zwykle bardziej złożony niż ręcznie napisany `EVALUATE`. Włącz trace, wejdź w interakcję z raportem, zatrzymaj i przeanalizuj najwolniejsze zapytania z listy.

---

## 12. Szukanie alternatyw DAX — praktyczny proces

1. Zbuduj 2-3 warianty logiki jako `DEFINE MEASURE` w jednym pliku, oddzielone komentarzami.
2. `Clear Cache` przed każdym uruchomieniem, porównaj FE/SE/Total (sekcja 4).
3. Sprawdź liczbę SE queries i obecność `CallbackDataID` w xmSQL (sekcja 5).
4. Jeśli masz Composite/DirectQuery — sprawdź RI Violation i wpływ "Assume Referential Integrity" (sekcja 6).
5. Dopiero gdy wariant jest wybrany — sprawdź Physical Query Plan (sekcja 7), żeby upewnić się, że nie ma nieoczekiwanej materializacji.
6. Zwaliduj wynik liczbowo zanim wdrożysz do modelu.

---

## Uzupełnienie: DAX Studio vs Tabular Editor — podział odpowiedzialności

- **DAX Studio** = pomiar wydajności zapytań/miar w czasie rzeczywistym, analiza VertiPaq, xmSQL, Query Plan, eksploracja danych przez `EVALUATE`.
- **Tabular Editor** = edycja metadanych modelu, Best Practice Analyzer (statyczna analiza reguł modelowania — inny poziom niż runtime performance).

BPA w Tabular Editor łapie problemy strukturalne (opisy, typy danych, nieużywane kolumny). DAX Studio odpowiada na pytanie "dlaczego TA konkretna miara jest wolna, TERAZ, na TYCH danych, z TYM kontekstem filtru" — dwa różne poziomy diagnostyki, warto ich używać łącznie.

---

## 13. Zbiorcze porównanie wielu wariantów miary w jednym miejscu

Server Timings świetnie sprawdza się przy analizie *pojedynczego* uruchomienia, ale nie agreguje wyników wielu wariantów obok siebie — do tego służy **Query History**, w połączeniu z metodyką wielokrotnych przebiegów (żeby wyeliminować szum systemowy, tak jak przy benchmarkingu kodu w Pythonie).

### 13.1 Query History — wbudowany log wszystkich uruchomień

**View → Query History** otwiera panel logujący *każde* wykonane zapytanie w bieżącej sesji: znacznik czasu, treść zapytania, **Duration**, **CPU Time**, **Rows Returned**. To jest Twoje centralne miejsce porównania — nie musisz ręcznie przepisywać wyników z Server Timings po każdym uruchomieniu.

Workflow:
1. Otwórz Query History (zostaje widoczny przez cały czas pracy).
2. Dla każdego wariantu miary stwórz osobną kartę zapytania (`Ctrl+T`) z `DEFINE MEASURE` + `EVALUATE { [Wariant] }`.
3. Przed każdym uruchomieniem: `Clear Cache`.
4. Uruchom wariant — wpis pojawia się automatycznie w Query History z dokładnym `Duration`/`CPU Time`.
5. Powtórz dla wszystkich wariantów — historia buduje się w jednej tabeli, posortowanej chronologicznie, więc widzisz je jedno pod drugim bez przełączania się między zakładkami Server Timings.
6. **Eksport**: prawym przyciskiem na Query History → eksport do pliku (CSV) — dalszą analizę (agregacja, wykres) możesz zrobić w Pythonie/Excelu zamiast ręcznie w interfejsie DAX Studio.

### 13.2 Dlaczego pojedynczy przebieg nie wystarcza — uruchamiaj każdy wariant wielokrotnie

Pojedynczy pomiar czasu wykonania jest podatny na szum (inne procesy w tle, stan pamięci, drobne wahania szeregowania wątków SE). Standardowa praktyka benchmarkingu (analogicznie jak przy `timeit` w Pythonie) to:

1. Dla każdego wariantu wykonaj **min. 5 przebiegów**, każdorazowo z `Clear Cache` przed uruchomieniem.
2. Odrzuć pierwszy przebieg jeśli podejrzewasz "cold start" po stronie systemu operacyjnego (rzadkie, ale zdarza się przy dużych modelach ładowanych z dysku).
3. Z Query History wyeksportuj wyniki i policz **medianę** (bardziej odporna na outliery niż średnia) `Duration` dla każdego wariantu — dokładnie tak, jak podszedłbyś do porównania czasu wykonania dwóch implementacji funkcji w Pythonie.

### 13.3 Zestawienie wyników w jednej tabeli — przykładowa struktura

Po eksporcie z Query History zestaw dane w formie, którą łatwo dalej analizować (Excel, albo `pandas`/`polars`, skoro i tak tam pracujesz):

| Wariant | Przebieg | FE (ms)* | SE (ms)* | Total (ms) | SE Queries* | CallbackDataID?* |
|---|---|---|---|---|---|---|
| A: RANKX+FILTER | 1-5 | z Server Timings | z Server Timings | z Query History | z Server Timings | TAK/NIE |
| B: TOPN | 1-5 | z Server Timings | z Server Timings | z Query History | z Server Timings | TAK/NIE |
| C: SUMMARIZE+RANK | 1-5 | z Server Timings | z Server Timings | z Query History | z Server Timings | TAK/NIE |

*Kolumny FE/SE/SE Queries/CallbackDataID nie są eksportowane automatycznie przez Query History — te musisz odczytać ręcznie z panelu Server Timings przy każdym przebiegu i dopisać. Query History daje Ci automatycznie tylko `Total Duration`, `CPU Time` i `Rows Returned`, ale to zwykle wystarcza do pierwszego, szybkiego rankingu wariantów — szczegółowy rozkład FE/SE sprawdzasz już tylko dla 1-2 najlepiej rokujących kandydatów, zamiast robić to dla każdego przebiegu każdego wariantu.

### 13.4 Alternatywa: jeden plik zapytania, wiele wariantów, ręczne przełączanie zaznaczenia

Jeśli nie chcesz mnożyć kart zapytań, możesz trzymać wszystkie warianty w jednym pliku `.dax`, oddzielone komentarzami, i uruchamiać każdy z osobna przez **zaznaczenie tekstu danego wariantu + F5** (DAX Studio wykonuje wtedy tylko zaznaczony fragment, nie cały plik). Query History i tak zaloguje to jako osobne uruchomienie — efekt identyczny jak z osobnymi kartami, ale wygodniejsze przy szybkiej iteracji nad 3-4 wariantami naraz.

### 13.5 Ograniczenie, o którym warto wiedzieć

DAX Studio **nie ma** wbudowanego dashboardu "side-by-side" pokazującego FE/SE/SE Queries dla wielu wariantów jednocześnie w jednej tabeli — to świadomie trzeba zbudować samemu (sekcja 13.3). Jeśli ten proces powtarzasz regularnie (np. przy każdej większej optymalizacji modelu sprzedażowego), warto rozważyć krótki skrypt Python parsujący eksport Query History + notatki z Server Timings do jednego podsumowania — naturalne rozszerzenie Twojej serii notebooków referencyjnych, gdyby chciał(a)ś to sformalizować jako powtarzalne narzędzie.
