# DAX Studio — kompletny przewodnik: analiza modelu, wydajność, testowanie miar

Przewodnik zakłada, że pracujesz na modelu gwiazdy (dim_Klienci, dim_Placowki, dim_Kalendarz, dim_Produkty, fact_Sprzedaz) i masz już zestaw miar DAX do analizy sprzedaży. Skupiam się na tym, co realnie zmienia pracę: analizę struktury modelu (VertiPaq), diagnostykę wydajności runtime, warsztat pracy z zapytaniami i metodykę testowania wariantów.

---

## 1. Połączenie z modelem

- **Connect → Power BI / Analysis Services** — DAX Studio wykrywa otwarte instancje Power BI Desktop (lokalny port SSAS) i pliki `.pbix` z otwartym modelem. Możesz też połączyć się z Power BI Service (Premium/Fabric) albo lokalnym SSAS Tabular.
- Po połączeniu masz dostęp do **Metadata pane** (lewa strona) — drzewo tabel, kolumn, miar, hierarchii i perspektyw.
- **Ważne dla Twojego workflow**: DAX Studio łączy się *na żywo* z silnikiem — zmiany w modelu (np. w Tabular Editor) widoczne są po odświeżeniu metadanych (`Refresh` w metadata pane), bez potrzeby zamykania sesji.

---

## 2. Architektura silnika — jak Power BI faktycznie przechowuje i przetwarza dane

Zanim zaczniesz mierzyć wydajność, warto rozumieć, co dzieje się "pod spodem" — to tłumaczy, dlaczego pewne wzorce DAX są tanie, a inne kosztowne.

### 2.1 VertiPaq (xVelocity) — silnik pamięciowy dla trybu Import

To domyślny silnik przy imporcie danych do Power BI (Twój przypadek dla `fact_Sprzedaz` itd.). Kluczowe cechy:

- **Column store, nie row store** — każda kolumna jest przechowywana i kompresowana osobno, niezależnie od pozostałych. To dlatego szerokie tabele faktów z wieloma rzadko używanymi kolumnami "kosztują" tylko za te kolumny, które faktycznie skanujesz w zapytaniu — silnik nie musi czytać całego wiersza.
- **Segmentacja** — dane w tabeli dzielone są na segmenty (domyślnie **ok. 8 mln wierszy** na segment w silniku Analysis Services/Power BI Premium; w Power BI Desktop/Pro segmentacja też występuje, ale rzadziej ma znaczenie przy mniejszych wolumenach). Segmenty są kompresowane niezależnie i mogą być skanowane równolegle przez Storage Engine — to jeden z powodów, dla których SE jest wielowątkowy, a FE nie.
- **Dictionary encoding** — dla kolumn tekstowych i kolumn o wysokiej kardynalności silnik buduje słownik unikalnych wartości i przechowuje wiersze jako wektor indeksów do słownika (liczby całkowite zamiast tekstu). Im mniej unikalnych wartości, tym mniejszy słownik i lepsza kompresja.
- **Value encoding** — dla kolumn liczbowych o niskiej-średniej kardynalności silnik może pominąć słownik i kodować wartość bezpośrednio (czasem z przesunięciem/skalowaniem, np. przechowując różnicę względem minimum). To najtańszy typ kompresji.
- **RLE (Run-Length Encoding)** — gdy dane w tabeli faktów są fizycznie posortowane/powtarzalne w obrębie segmentu (np. kolumna daty w `fact_Sprzedaz` ułożona chronologicznie), silnik dodatkowo kompresuje powtarzające się sekwencje: zamiast zapisywać każdą wartość osobno, zapisuje parę (wartość, liczba powtórzeń z rzędu). Im dłuższe nieprzerwane serie tej samej wartości, tym mniej par do zapisania i lepsza kompresja — na danych nieposortowanych RLE praktycznie nie ma czego skompresować. **Zastrzeżenie:** to VertiPaq sam automatycznie dobiera przy przetwarzaniu, która kolumna segmentu najlepiej nadaje się jako klucz sortowania — nie masz na to wpływu przez kolejność kolumn w definicji tabeli. Jedyne, na co realnie masz wpływ, to fizyczna kolejność wierszy w danych źródłowych, czyli `ORDER BY` w widoku SQL zasilającym `fact_Sprzedaz` — a ponieważ w obrębie segmentu wszystkie kolumny dzielą tę samą kolejność wierszy, sortowanie po jednej kolumnie (np. `Data`) poprawia RLE też dla kolumn z nią skorelowanych.

### 2.2 Formula Engine vs Storage Engine — pełniejszy obraz

- **Storage Engine (SE)** — wielowątkowy, operuje na skompresowanych danych VertiPaq, wykonuje skany, filtrowanie na poziomie kolumn, proste agregacje (SUM, COUNT, MIN, MAX, DISTINCTCOUNT). Generuje zapytania w wewnętrznym języku **xmSQL** (widoczne w Server Timings) — to nie jest prawdziwy SQL, ale czytelna reprezentacja tego, co SE faktycznie robi.
- **Formula Engine (FE)** — jednowątkowy, interpretuje logikę DAX, którą SE nie potrafi wykonać natywnie (iteracje z warunkami, `RANKX`, zagnieżdżone `CALCULATE` z modyfikacją kontekstu, funkcje czasowe, część funkcji tekstowych). FE "zamawia" dane z SE, po czym łączy/przetwarza wyniki.
- **Konsekwencja praktyczna**: SE skaluje się z liczbą rdzeni, FE nie. Miara, która generuje dużo pracy FE, nie przyspieszy nawet na potężnym sprzęcie — dlatego wysoki % FE w Server Timings to sygnał do przepisania logiki, a nie do dodania mocy obliczeniowej.

### 2.3 Tryby przechowywania danych — Import, DirectQuery, Dual, Composite

- **Import** — dane fizycznie w VertiPaq, jak opisano wyżej. Najszybszy tryb dla analityki, ale wymaga odświeżania.
- **DirectQuery** — brak kopii danych w VertiPaq; każde zapytanie DAX jest tłumaczone na SQL i wysyłane do źródła (np. SQL Server) w czasie rzeczywistym. Storage Engine w tym trybie to *silnik źródła*, nie VertiPaq — DAX Studio pokaże wygenerowany SQL zamiast xmSQL. Wydajność zależy od optymalizacji bazy źródłowej (indeksy, statystyki), nie od VertiPaq.
- **Dual** — tabela (zwykle wymiar) dostępna jednocześnie jako Import i DirectQuery; silnik wybiera tryb zależnie od kontekstu zapytania (np. przy łączeniu z tabelą faktów w DirectQuery, wymiar w Dual też będzie odpytany przez DirectQuery, żeby uniknąć niespójności).
- **Composite model** — mieszanka trybów w jednym modelu (część tabel Import, część DirectQuery). To tu najczęściej pojawiają się problemy z **RI Violation** (patrz sekcja 6) i wymuszonymi cross-source joinami, które bywają bardzo kosztowne.

Dla Twojego stacku (Power BI + SQL Server): jeśli w ogóle rozważasz DirectQuery/Composite dla części modelu (np. świeże dane transakcyjne łączone z historycznym Import), DAX Studio jest jedynym narzędziem, które pokaże Ci realnie wygenerowany SQL i pozwoli ocenić, czy zapytanie faktycznie foldowało się dobrze do źródła.

---

## 3. Analiza struktury modelu — VertiPaq Analyzer

**Advanced → View Metrics** (`Model Metrics`) generuje raport statyczny — pokazuje *co jest w modelu*, nie jak szybko się liczy (to rola sekcji 5-8).

### 3.1 Co znaczą poszczególne kolumny raportu

| Kolumna | Co dokładnie pokazuje | Jak interpretować |
|---|---|---|
| **Cardinality** | Liczba unikalnych wartości w kolumnie | Podstawowy czynnik kosztu Dictionary encoding — im wyżej, tym większy słownik. Sama w sobie nie mówi jeszcze, czy to problem (patrz Encoding niżej). |
| **Data** (Data Size) | Rozmiar w bajtach samych **zakodowanych wartości** (wektor indeksów przy Hash, wektor wartości przy Value) — bez słownika | To jest część, na którą wpływa RLE i typ encoding. Rośnie liniowo z liczbą wierszy, ale wolniej, jeśli dane dobrze się kompresują (posortowane, niska kardynalność). |
| **Dictionary** (Dictionary Size) | Rozmiar w bajtach **słownika unikalnych wartości** — istnieje tylko przy Hash/Dictionary encoding | Niezależny od liczby wierszy, zależny od kardynalności i długości/typu przechowywanych wartości (dłuższe teksty = większy słownik). Przy Value encoding to pole jest bliskie zeru, bo słownika nie ma. |
| **Total Size** | `Data + Dictionary + HierSize` — pełny koszt pamięciowy kolumny | To jest liczba, którą realnie porównujesz między kolumnami przy szukaniu "gdzie model jest ciężki". |
| **HierSize** (Hierarchy Size) | Rozmiar dodatkowej struktury **attribute hierarchy** — wewnętrznego indeksu budowanego automatycznie dla każdej kolumny (nawet ukrytej), używanego m.in. przy filtrowaniu/sortowaniu i przez silnik MDX | Często pomijane, a bywa zaskakująco duże dla kolumn technicznych/kluczy o wysokiej kardynalności. Da się to wyłączyć właściwością `IsAvailableInMDX = false` (patrz przewodnik Tabular Editor, sekcja 13, przykład 7) dla kolumn ukrytych, niesłużących jako oś w wizualach — bezpośrednio redukuje `HierSize` do ~0. |
| **Encoding** | `VALUE` albo `HASH` — wybrany przez silnik sposób kodowania kolumny | Rozwinięte w sekcji 3.2 — nie zawsze oczywiste "na logikę", warto rozumieć czynniki decydujące. |
| **% Table / % Database** | Udział rozmiaru kolumny/tabeli w rozmiarze odpowiednio całej tabeli / całego modelu | Szybki sposób znalezienia "gdzie szukać" bez porównywania bezwzględnych liczb między tabelami różnej wielkości. |
| **Partitions** | Liczba partycji fizycznych danej tabeli | W Power BI Pro/Desktop zwykle `1` (patrz przewodnik Tabular Editor, sekcja 7 — partycjonowanie wymaga Premium/Fabric/Azure AS). Więcej niż 1 partycja sugeruje Incremental Refresh już skonfigurowany na serwerze albo model na Azure AS. |
| **Segments** | Liczba segmentów fizycznych (podział wewnątrz partycji, patrz sekcja 2.1 — ok. 8 mln wierszy/segment) | Rośnie wraz z liczbą wierszy tabeli. Duża liczba segmentów sama w sobie nie jest problemem (SE skanuje je równolegle), ale warto wiedzieć, że to inny podział niż Partitions — Partitions to Twój świadomy podział (np. wg miesiąca), Segments to wewnętrzny podział silnika w obrębie jednej partycji. |

### 3.2 Encoding — dlaczego kolumna czasem "wbrew logice" dostaje Hash zamiast Value

To jedno z częstszych źródeł zaskoczenia przy pierwszym kontakcie z VertiPaq Analyzer — kolumna liczbowa o niskiej kardynalności "powinna" dostać Value, a dostaje Hash. Silnik wybiera encoding na podstawie **szacowanego kosztu w bitach**, ale kilka czynników wymusza Hash niezależnie od tego rachunku:

- **Kolumna uczestniczy w relacji** (jako `From` lub `To`) — VertiPaq zawsze koduje klucze relacji jako Hash/Dictionary, niezależnie od tego, czy Value byłoby tańsze, bo mechanizm joina działa na indeksach słownika (RID-ach), nie na surowych wartościach. To jest świadoma decyzja architektoniczna, nie "pominięcie optymalizacji".
- **Typ danych Decimal/Double (zmiennoprzecinkowy)** — Value encoding wymaga reprezentacji całkowitoliczbowej (ew. przeskalowanego Fixed Decimal); zwykły `Double` prawie zawsze idzie do Hash niezależnie od zakresu wartości.
- **Kolumna obliczana (calculated column)** — domyślnie kodowana jako Hash, bo faza optymalizacji encoding dotyczy kolumn natywnie importowanych, przetwarzanych podczas ładowania danych — kolumny obliczane są wyliczane przez Formula Engine już po tej fazie.
- **Remis kosztowy przy bardzo niskiej kardynalności** — gdy koszt bitowy Value i Hash wychodzi praktycznie identyczny (mały zakres, mała kardynalność), algorytm VertiPaq (niepubliczny, oparty o próbkowanie) może rozstrzygnąć na Hash bez wyraźnej korzyści z Value — udokumentowanej reguły tie-breakingu Microsoft nie publikuje.

**Jak jawnie wymusić Value**, jeśli masz pewność, że powinno się opłacać: właściwość kolumny **`Encoding Hint`** (`Default` / `Value` / `Hash`) w Tabular Editor → Properties, potem `Process Full` tabeli i weryfikacja w View Metrics. To "podpowiedź", nie twardy nakaz — silnik może ją zignorować w skrajnych przypadkach.

### 3.3 Współczynnik kompresji — co to właściwie znaczy i jak go policzyć

Wcześniej we wcześniejszej wersji tego przewodnika pojawiło się sformułowanie "współczynnik kompresji poniżej 3:1-4:1" bez wyjaśnienia, co to znaczy — doprecyzowanie:

**Współczynnik kompresji = rozmiar, jaki kolumna zajęłaby bez żadnej kompresji VertiPaq, podzielony przez jej faktyczny `Total Size` z View Metrics.**

Rozmiar "surowy" (bez kompresji) szacujesz jako: `liczba wierszy × rozmiar typu danych w bajtach` (np. 8 bajtów dla Int64/Double, zmienna długość dla tekstu — przyjmij średnią długość string × 2 przy Unicode jako przybliżenie).

**Przykład na Twoim modelu:** kolumna `fact_Sprzedaz[ID_Klienta]` (Int64, 8 bajtów), 2 450 000 wierszy, kardynalność 15 000 (liczba unikalnych klientów):
- Rozmiar surowy: `2 450 000 × 8 bajtów ≈ 18,7 MB`.
- Jeśli `Total Size` w View Metrics wynosi np. `1,2 MB` → współczynnik kompresji ≈ `18,7 / 1,2 ≈ 15:1` — dobry wynik, typowy dla klucza obcego o umiarkowanej kardynalności.
- Jeśli `Total Size` wynosi np. `6 MB` → współczynnik ≈ `18,7 / 6 ≈ 3:1` — słaby wynik jak na kolumnę liczbową o tej kardynalności; sygnał, żeby sprawdzić Encoding (sekcja 3.2) i czy dane są posortowane (RLE, sekcja 2.1).

**Dlaczego akurat próg 3:1-4:1 jest sygnałem ostrzegawczym:** to nie jest oficjalna, udokumentowana przez Microsoft granica — to praktyczna heurystyka. Typowe, dobrze skompresowane kolumny w modelach gwiazdy osiągają 10:1 i więcej; wynik bliski 3:1-4:1 oznacza, że kompresja ledwo działa (kolumna zajmuje niewiele mniej niż "na surowo"), co przy kolumnach, które "wyglądają" na powtarzalne (kategorie, statusy, klucze o umiarkowanej kardynalności), jest sygnałem błędu w modelowaniu (zły typ danych, brak sortowania, niepotrzebna precyzja liczbowa), a nie naturalną granicą fizyczną.

### 3.4 Konkretne wartości, które powinny Cię niepokoić — zbiorczo

- **Kolumna stanowiąca >20-25% rozmiaru całej tabeli faktów** — zwykle sygnał zbyt wysokiej kardynalności (np. ID transakcji, timestamp z sekundami).
- **Kardynalność kolumny >1 mln unikalnych wartości** przy jednoczesnym dużym rozmiarze — kandydat do redukcji granularności.
- **Współczynnik kompresji poniżej ok. 3:1-4:1** (patrz 3.3) dla kolumny, która "wygląda" na powtarzalną.
- **Kolumna numeryczna zakodowana jako Hash zamiast Value** bez oczywistej przyczyny z sekcji 3.2 (brak relacji, typ całkowity, nie kolumna obliczana) — warto zbadać przez `Encoding Hint`, choć przy małych kolumnach efekt bywa pomijalnie mały w praktyce.
- **`HierSize` nieproporcjonalnie duży** dla ukrytej kolumny technicznej — kandydat do `IsAvailableInMDX = false`.

Eksport wyników (`Export Model Metrics`) pozwala śledzić zmiany rozmiaru modelu między iteracjami.

### 3.5 Przydatne gotowe zapytania DMV — surowe dane pod raportem

View Metrics buduje swój raport na podstawie zapytań DMV (Dynamic Management Views) — możesz je odpytać bezpośrednio w oknie zapytania DAX Studio, kiedy potrzebujesz czegoś, czego UI View Metrics nie pokazuje wprost, albo chcesz wyeksportować surowe dane do dalszej analizy w Pythonie.

```sql
-- Rozmiar i liczba wierszy każdej tabeli w modelu
SELECT * FROM $SYSTEM.DISCOVER_STORAGE_TABLES

-- Szczegóły kolumn: encoding, cardinality, rozmiar słownika — źródło danych dla zakładki Columns w View Metrics
SELECT * FROM $SYSTEM.DISCOVER_STORAGE_TABLE_COLUMNS
WHERE [TABLE_ID] LIKE '%fact_Sprzedaz%'

-- Szczegóły na poziomie segmentów fizycznych — najbliżej informacji o RLE (patrz też sekcja 2.1)
SELECT * FROM $SYSTEM.DISCOVER_STORAGE_TABLE_COLUMN_SEGMENTS
WHERE [TABLE_ID] LIKE '%fact_Sprzedaz%'

-- Partycje fizyczne tabeli (przydatne przy Premium/Fabric/Azure AS, patrz przewodnik Tabular Editor sekcja 7)
SELECT * FROM $SYSTEM.DISCOVER_STORAGE_TABLE_PARTITIONS

-- Zużycie pamięci przez poszczególne obiekty modelu (tabele, kolumny, hierarchie, relacje) w jednym miejscu
SELECT * FROM $SYSTEM.DISCOVER_OBJECT_MEMORY_USAGE
ORDER BY [SHRINKABLE_MEMORY] DESC

-- Zależności obliczeniowe między miarami/kolumnami — odpowiednik "Show Dependencies" z Tabular Editor, ale z poziomu silnika
SELECT * FROM $SYSTEM.DISCOVER_CALC_DEPENDENCY
WHERE [OBJECT_TYPE] = 'MEASURE'
```

Praktyczne zastosowanie: `DISCOVER_OBJECT_MEMORY_USAGE` posortowane malejąco to często szybszy sposób znalezienia "co zajmuje najwięcej pamięci w całym modelu" niż przeklikiwanie się przez zakładki View Metrics tabela po tabeli — jedno zapytanie, jedna posortowana lista.

---

## 4. Testowanie i pomiar wydajności miar — Server Timings

**Home → Server Timings** (włącz przed uruchomieniem zapytania) rejestruje podział czasu na FE i SE.

### 4.1 Kluczowe metryki i orientacyjne progi

| Metryka | Sygnał ostrzegawczy | Interpretacja |
|---|---|---|
| **% Formula Engine** | >40-50% czasu całkowitego | Logika DAX jest "droga" — kandydat do przepisania wzorca, nie do dodania mocy obliczeniowej |
| **Liczba SE Queries** | >10-15 zapytań dla jednej, pojedynczej miary bez złożonych zależności | Możliwa nadmiarowa iteracja/wielokrotne skanowanie tych samych tabel |
| **SE Cache hit** | 0% przy powtórnym uruchomieniu identycznego zapytania | Sprawdź, czy nie testujesz przypadkiem z włączonym `Clear Cache` przy każdym uruchomieniu (to normalne w testach, ale mylące jeśli porównujesz z realnym użyciem raportu) |
| **Total duration** dla pojedynczego wizuala | >1-2s w typowym raporcie interaktywnym | Microsoft rekomenduje orientacyjnie <1s dla płynnego UX, do ok. 5s jako granica akceptowalności |

**Zawsze `Clear Cache` przed testem porównawczym** — inaczej drugie uruchomienie tej samej miary będzie sztucznie szybsze przez trafienie w SE cache.

### 4.2 Dodatkowe kolumny w liście zapytań Storage Engine — Subclass i Par.

Po kliknięciu na zapytanie w dolnej liście Server Timings widzisz szczegółową tabelę pojedynczych operacji SE. Dwie kolumny, które nie są od razu oczywiste:

- **Subclass** — pokazuje **typ/operator** danej operacji Storage Engine, np. `VertiPaqScan` (zwykły skan danych), `Batch`/`BatchVertiPaqScan` (grupa operacji wykonanych wsadowo — od DAX Studio 2.17 widoczna domyślnie, zawiera zagregowany koszt kilku operacji Scan wykonanych razem), `VertiPaqScanInternal` (operacja pomocnicza wewnątrz batcha, niekopiowana bezpośrednio do Formula Engine), `VertiPaqCacheExactMatch` (dokładne trafienie w cache — zapytanie nie dotknęło surowych danych). To Twój pierwszy krop wskazujący, **jakiego rodzaju** pracę silnik wykonał na danej linii — przydatne przy szukaniu, czy dana operacja to "prawdziwy" skan, czy tylko odczyt z cache.
- **Par.** (Parallelism) — stosunek `CPU / Duration` dla danej operacji. Wartość **>1 oznacza, że operacja wykonywała się równolegle** na wielu wątkach (np. `Par. = 4` sugeruje z grubsza 4 wątki pracujące jednocześnie nad tą operacją) — to bezpośredni, liczbowy dowód na to, czy Storage Engine faktycznie wykorzystał wielowątkowość przy tej konkretnej operacji, czy wykonał ją jednowątkowo. Niska wartość `Par.` przy operacji skanującej dużą tabelę (dużo `Records`/`KB`) to sygnał, że coś ogranicza równoległość — np. mała liczba segmentów (za mało danych, żeby sensownie rozdzielić na wątki) albo operacja z natury sekwencyjna.

### 4.3 Test A/B alternatywnych wzorców

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

`RANKX` w Wariancie A wymusza obliczenie rangi dla *każdego* klienta w kontekście `ALL`, co przy dużej tabeli wymiaru generuje znacznie więcej pracy FE niż `TOPN`, które od razu operuje na już posortowanym podzbiorze. Różnica typowo widoczna w liczbie SE queries i w % FE.

Jeśli masz kilka wariantów w jednym pliku, **Home → All Queries** wykona je sekwencyjnie i pokaże zbiorczy raport czasów.

---

## 5. CallbackDataID — najczęściej pomijany, a bardzo kosztowny problem

**Co to jest:** gdy Storage Engine skanuje dane, ale natrafia na fragment logiki DAX, którego nie potrafi wykonać natywnie (np. nietypowa funkcja tekstowa, warunek wymagający wywołania Formula Engine dla każdego wiersza/grupy), musi "oddzwonić" do FE w trakcie skanu. To wywołanie zwrotne widoczne jest w xmSQL (Server Timings → szczegóły zapytania SE) jako funkcja **`CallbackDataID`**.

**Dlaczego to boli:** CallbackDataID oznacza, że SE traci swoją główną przewagę — działanie wsadowe na skompresowanych danych — i zamiast tego wykonuje operację *wiersz po wierszu*, wywołując wolniejszy, jednowątkowy FE dla każdego rekordu. Na małej tabeli niewidoczne, na tabeli faktów z milionami wierszy potrafi zamienić zapytanie z ułamka sekundy w kilkanaście sekund.

**Typowe przyczyny w praktyce:**
- Użycie funkcji, które nie foldują się do SE wewnątrz `FILTER`/`CALCULATE` operującego na tabeli faktów (np. `SEARCH`, `FORMAT`, niektóre funkcje daty stosowane wiersz po wierszu).
- Warunki logiczne mieszające kolumny z różnych tabel w sposób wymuszający obliczenie skalarnego wyniku per wiersz.
- Iteratory (`SUMX`, `FILTER`) z wyrażeniem, które samo w sobie wymaga FE.

**Jak wykryć:** w Server Timings kliknij na zapytanie SE i sprawdź treść xmSQL — obecność `CallbackDataID` to jednoznaczny sygnał. Warto też zwrócić uwagę na czas trwania pojedynczego SE query nieproporcjonalny do liczby przetworzonych wierszy.

**Jak sobie radzić:**
1. Przenieś logikę wymagającą FE poza pętlę skanowania — oblicz warunek raz jako `VAR` na poziomie kontekstu filtru.
2. Zamień funkcje niefoldowalne na ich odpowiedniki natywne dla SE.
3. Rozważ przeniesienie logiki per wiersz do warstwy SQL/Power Query jako kolumnę obliczoną przy ładowaniu, zamiast liczyć ją w locie w DAX.

---

## 6. RI Violation (Referential Integrity Violation)

**Co to jest:** VertiPaq domyślnie **nie zakłada**, że każda wartość klucza obcego w tabeli faktów ma odpowiadający wiersz w tabeli wymiaru. Jeśli silnik nie może zagwarantować pełnej integralności referencyjnej, musi traktować join jako potencjalny outer join, zamiast czystego inner joina — dodatkowy narzut na każdym zapytaniu przechodzącym przez tę relację.

**Gdzie to widać:** w **Query Plan** (Physical Plan) jako adnotacja przy operacji join/scan wskazująca, że relacja może naruszać RI — silnik dokłada sprawdzenie obecności "blank row" (niewidocznego wiersza reprezentującego niedopasowane klucze).

**Kiedy to występuje:**
- **Import mode**: zawsze technicznie możliwe, narzut zwykle niewielki przy dobrze zbudowanym modelu gwiazdy z czystymi kluczami.
- **DirectQuery / Composite model**: znacznie ważniejsze — opcja **"Assume Referential Integrity"** przy definicji relacji generuje INNER JOIN zamiast OUTER JOIN w SQL wysyłanym do źródła.

**Jak sobie radzić:**
1. W Import mode: zadbaj, żeby klucze obce w `fact_Sprzedaz` nie zawierały wartości sierocych — najlepiej wymuszone na poziomie SQL, zanim dane trafią do Power Query.
2. W DirectQuery/Composite: zaznacz **"Assume Referential Integrity"**, jeśli masz pewność co do czystości kluczy (np. FK constraint w SQL Server).
3. Zweryfikuj w DAX Studio: uruchom `EVALUATE`, sprawdź Physical Query Plan pod kątem adnotacji o RI, porównaj czas przed i po włączeniu tej opcji.

---

## 7. Query Plan — dogłębna analiza fizycznego i logicznego planu

**Home → Query Plan** pokazuje dwa widoki po uruchomieniu zapytania.

### 7.1 Logical Query Plan

Wysokopoziomowa reprezentacja *co* silnik ma zrobić: `Scan_Vertipaq`, `Filter`, `AggregationSpool`/`Sum_Vertipaq`, `GroupBy_Vertipaq`. Dobra do szybkiego zrozumienia intencji, nie pokazuje realnego kosztu.

### 7.2 Physical Query Plan — tu jest realna diagnostyka

Pokazuje faktycznie wykonane operacje z liczbą przetworzonych wierszy (**`Records`**) na każdym etapie.

**Na co patrzeć konkretnie:**
- **`Records` nieproporcjonalnie duże względem finalnego wyniku** — filtrowanie nie zostało zepchnięte wystarczająco wcześnie w planie.
- **Powtarzające się operacje `Spool`** — częsty efekt zagnieżdżonych `CALCULATE`/`FILTER`, gdzie ten sam podzbiór danych jest budowany wielokrotnie zamiast raz i buforowany w `VAR`.
- **`Cache` w planie** — dobry znak, wynik pośredni wykorzystany bez ponownego przeliczania.
- Adnotacje sugerujące **RI Violation** (sekcja 6).
- Wywołania **`CallbackDataID`** widoczne pośrednio przez drobnoziarnisty wzorzec skanowania (sekcja 5).

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

Różnica: w wersji A silnik materializuje ranking dla całej populacji klientów (180k `Spool`), mimo że interesuje Cię tylko 10. W wersji B `TOPN` pozwala silnikowi ograniczyć się do właściwego podzbioru dużo wcześniej w planie.

**Praktyczna zasada:** nie analizuj Query Plan dla każdej miary — sięgaj po niego dopiero gdy Server Timings wskaże konkretny problem (wysoki FE%, dużo SE queries, podejrzenie CallbackDataID/RI Violation).

---

## 8. Warsztat pracy z zapytaniami

### 8.1 Formatowanie DAX

**Format Query** (Ctrl+Shift+F) — integracja z DAX Formatter (Sql BI). Ten sam silnik formatujący co w Tabular Editor (przewodnik TE, sekcja 11), więc styl kodu zostaje spójny między narzędziami.

### 8.2 Testowanie miar bez wdrażania do modelu — `DEFINE MEASURE`

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

### 8.3 Eksport wyników i integracja z Excel

- **Output → Excel / CSV / Linked Excel Table** — eksport wyniku zapytania. `Linked Excel Table` tworzy połączenie odświeżalne z poziomu Excela.
- **Copy as JSON / Copy as INSERT** — przydatne do dokumentacji.

---

## 9. Testowanie na realnym obciążeniu z Power BI — Trace / All Queries

**Home → All Queries** loguje zapytania DAX faktycznie wysyłane przez Power BI Desktop przy interakcji z wizualami — realny kontekst filtru wygenerowany przez wizual, zwykle bardziej złożony niż ręcznie napisany `EVALUATE`. Włącz trace, wejdź w interakcję z raportem, zatrzymaj i przeanalizuj najwolniejsze zapytania z listy.

---

## 10. Szukanie alternatyw DAX — praktyczny proces

1. Zbuduj 2-3 warianty logiki jako `DEFINE MEASURE` w jednym pliku, oddzielone komentarzami.
2. `Clear Cache` przed każdym uruchomieniem, porównaj FE/SE/Total (sekcja 4).
3. Sprawdź liczbę SE queries i obecność `CallbackDataID` w xmSQL (sekcja 5).
4. Jeśli masz Composite/DirectQuery — sprawdź RI Violation i wpływ "Assume Referential Integrity" (sekcja 6).
5. Dopiero gdy wariant jest wybrany — sprawdź Physical Query Plan (sekcja 7).
6. Zwaliduj wynik liczbowo zanim wdrożysz do modelu.

---

## 11. Zbiorcze porównanie wielu wariantów miary w jednym miejscu

Server Timings świetnie sprawdza się przy analizie *pojedynczego* uruchomienia, ale nie agreguje wyników wielu wariantów obok siebie — do tego służy **Query History**, w połączeniu z metodyką wielokrotnych przebiegów.

### 11.1 Query History — wbudowany log wszystkich uruchomień

**View → Query History** loguje *każde* wykonane zapytanie: znacznik czasu, treść zapytania, **Duration**, **CPU Time**, **Rows Returned**.

Workflow:
1. Otwórz Query History (zostaje widoczny przez cały czas pracy).
2. Dla każdego wariantu miary stwórz osobną kartę zapytania (`Ctrl+T`) z `DEFINE MEASURE` + `EVALUATE { [Wariant] }`.
3. Przed każdym uruchomieniem: `Clear Cache`.
4. Uruchom wariant — wpis pojawia się automatycznie w Query History.
5. Powtórz dla wszystkich wariantów — historia buduje się w jednej tabeli, chronologicznie.
6. **Eksport**: prawym przyciskiem na Query History → CSV — dalszą analizę zrób w Pythonie/Excelu.

### 11.2 Dlaczego pojedynczy przebieg nie wystarcza

1. Dla każdego wariantu wykonaj **min. 5 przebiegów**, każdorazowo z `Clear Cache` przed uruchomieniem.
2. Odrzuć pierwszy przebieg, jeśli podejrzewasz "cold start".
3. Z Query History wyeksportuj wyniki i policz **medianę** `Duration` dla każdego wariantu (bardziej odporna na outliery niż średnia).

### 11.3 Zestawienie wyników w jednej tabeli

| Wariant | Przebieg | FE (ms)* | SE (ms)* | Total (ms) | SE Queries* | CallbackDataID?* |
|---|---|---|---|---|---|---|
| A: RANKX+FILTER | 1-5 | z Server Timings | z Server Timings | z Query History | z Server Timings | TAK/NIE |
| B: TOPN | 1-5 | z Server Timings | z Server Timings | z Query History | z Server Timings | TAK/NIE |

*Kolumny FE/SE/SE Queries/CallbackDataID nie są eksportowane automatycznie przez Query History — odczytujesz je ręcznie z Server Timings i dopisujesz. Query History daje automatycznie tylko `Total Duration`, `CPU Time` i `Rows Returned`, co zwykle wystarcza do pierwszego, szybkiego rankingu wariantów.

### 11.4 Alternatywa: jeden plik zapytania, wiele wariantów

Trzymaj warianty w jednym pliku `.dax`, oddzielone komentarzami, i uruchamiaj każdy z osobna przez zaznaczenie tekstu + F5. Query History zaloguje to jako osobne uruchomienie — efekt identyczny jak z osobnymi kartami, wygodniejsze przy szybkiej iteracji nad 3-4 wariantami naraz.

### 11.5 Ograniczenie, o którym warto wiedzieć

DAX Studio **nie ma** wbudowanego dashboardu side-by-side pokazującego FE/SE/SE Queries dla wielu wariantów jednocześnie — to trzeba zbudować samemu (sekcja 11.3). Jeśli ten proces powtarzasz regularnie, warto rozważyć krótki skrypt Python parsujący eksport Query History + notatki z Server Timings do jednego podsumowania.
