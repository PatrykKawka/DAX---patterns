# Tabular Editor 2.x (darmowy) — kompletny przewodnik funkcjonalny

Przewodnik komplementarny do przewodnika po DAX Studio — tam masz diagnostykę wydajności runtime, tu masz **edycję i architekturę metadanych modelu**. Zakładam Twój model gwiazdy (dim_Klienci, dim_Placowki, dim_Kalendarz, dim_Produkty, fact_Sprzedaz) jako punkt odniesienia w przykładach.

Ważne zastrzeżenie na start: Tabular Editor 2.x (open source, darmowy) różni się od komercyjnego Tabular Editor 3 Business/Enterprise — **C# scripting i Best Practice Analyzer są darmowe w 2.x** (to jego główna przewaga nad konkurencją), ale np. wbudowana integracja z Git, wizualny DAX debugger krok-po-kroku czy "Calculation Impact Analysis" to funkcje Pro (TE3). W przewodniku zaznaczam, gdzie dotyczy to ograniczenia.

---

## 1. Połączenie z modelem

- **File → Open → From File** — otwiera plik `.bim` (offline model project) lub plik Model.bim wyeksportowany z Analysis Services.
- **File → Open → From DB** — połączenie z lokalną/serwerową instancją Analysis Services (Azure AS, SSAS Tabular, Power BI Premium/Fabric XMLA endpoint).
- **File → Open → From Power BI Desktop** — DOKŁADNIE jak w DAX Studio: TE2 wykrywa otwarte instancje Power BI Desktop przez lokalny port SSAS i pozwala edytować model **na żywo**, z bezpośrednim zapisem zmian z powrotem do pliku PBIX po `Save`.
- **Kolejność pracy, którą warto zapamiętać**: zmiany w TE2 na modelu podłączonym do Power BI Desktop stosują się natychmiast po `Ctrl+S` (Save) — ale to nadpisuje metadane w otwartym pliku PBIX. Jeśli masz otwarty ten sam model równolegle w DAX Studio, **odśwież metadane w DAX Studio** po zapisie w TE2, inaczej będziesz patrzeć na nieaktualną strukturę.

---

## 2. Interfejs — z czego realnie korzystasz na co dzień

- **Model Explorer** (drzewo po lewej) — Tables → Columns/Measures/Hierarchies/Partitions, Relationships, Perspectives, Roles, Translations, Cultures, Expressions (dla parametrów Power Query dzielonych między zapytaniami).
- **Properties pane** (prawy dół) — pełna lista właściwości zaznaczonego obiektu (Format String, Display Folder, Data Type, Is Hidden, Description itd.) — dużo szybszy dostęp niż w panelu Power BI Desktop, bo widzisz wszystkie właściwości naraz, nie tylko te wyeksponowane w UI Power BI.
- **DAX Editor** (środek) — edycja wyrażenia miary/kolumny obliczanej w dużym oknie tekstowym z podświetlaniem składni — bez limitu wysokości, jaki ma pole edycji miary w Power BI Desktop. Przy dłuższych miarach (np. Twoje wielowarunkowe miary TOP10%) to realna różnica w komforcie pracy.
- **C# Script** (zakładka na dole) — panel do Advanced Scripting (sekcja 5).
- **Best Practice Analyzer** (zakładka na dole) — lista reguł i wyników analizy (sekcja 4).

---

## 3. Edycja metadanych na skalę — multi-select editing

To jest funkcja, która najbardziej odróżnia pracę w TE2 od Power BI Desktop: **zaznaczasz wiele obiektów naraz (Ctrl+klik lub Shift+klik) i zmieniasz wspólną właściwość jednym ruchem** w panelu Properties.

Praktyczne zastosowania w Twoim kontekście:
- Zaznacz wszystkie miary walutowe (`Sprzedaz Total`, `Sprzedaz TOP10`, `Sprzedaz TOP10%` itd.) → ustaw jednolity `Format String` (np. `#,##0.00 "zł"`) jednym kliknięciem, zamiast wchodzić w każdą miarę osobno.
- Zaznacz grupę miar → ustaw wspólny `Display Folder` (np. `Sprzedaz\TOP Klienci`), co porządkuje drzewo pól w Power BI bez ręcznego przeciągania w interfejsie raportu.
- Zaznacz kolumny techniczne (klucze, ID) → `Is Hidden = True` masowo, zamiast pojedynczo ukrywać każdą.
- Dodawanie/masowa zmiana `Description` dla dokumentacji modelu — szczególnie wartościowe, jeśli chcesz, żeby model miał sensowne opisy widoczne w Power BI przy najechaniu na pole.

---

## 4. Best Practice Analyzer (BPA) — darmowy, statyczny audyt modelu

BPA to zestaw reguł sprawdzających model pod kątem dobrych praktyk modelowania (nie wydajności runtime — to rola DAX Studio). W TE2 jest **w pełni darmowy**.

**Konfiguracja:**
- Domyślnie TE2 nie ma wbudowanego zestawu reguł — musisz pobrać plik `BPARules.json`. Najpopularniejsze, aktywnie utrzymywane zestawy to reguły **oficjalne Tabular Editor (community rules na GitHubie: `TabularEditor/BestPracticeRules`)** oraz reguły **SQLBI**. Plik wrzucasz do katalogu `%LocalAppData%\TabularEditor\BestPracticeRules\` (globalnie dla wszystkich modeli) albo obok pliku modelu (lokalnie dla projektu).
- **Tools → Best Practice Analyzer** (albo zakładka na dole) uruchamia analizę i pokazuje listę naruszeń pogrupowanych wg kategorii (Performance, Maintenance, Naming Conventions, Formatting, Error Prevention).

**Typowe reguły, które realnie łapią błędy w modelach sprzedażowych jak Twój:**
- Kolumny/miary bez `Format String` ustawionego jawnie.
- Kolumny numeryczne, które powinny być ukryte, bo mają odpowiadającą miarę (np. surowa kolumna `Kwota` w `fact_Sprzedaz` widoczna zamiast tylko miary `Sprzedaz Total`).
- Relacje nieaktywne bez odpowiadającej miary z `USERELATIONSHIP` (martwa relacja — założona, ale niewykorzystana w żadnej mierze).
- Kolumny kalendarzowe/daty przechowywane jako tekst zamiast typu Date.
- Brak `Description` na kluczowych miarach biznesowych.
- Relacje typu many-to-many bez świadomego uzasadnienia (częsty błąd modelowania, a nie zawsze intencjonalny wzorzec).
- Miary odwołujące się bezpośrednio do kolumn zamiast przez inne miary (naruszenie zasady "jedna kolumna liczona = jedna miara bazowa", utrudniające późniejszy refactoring).

**Naprawa masowa:** po zidentyfikowaniu naruszenia BPA często można je naprawić dla wszystkich obiektów naraz przez zaznaczenie multi-select (sekcja 3) albo skryptem C# (sekcja 5) — to połączenie BPA + scripting jest najmocniejszą stroną darmowej wersji narzędzia.

---

## 5. C# Advanced Scripting — najmocniejsza darmowa funkcja TE2

Panel **C# Script** pozwala pisać krótkie skrypty operujące na całym obiekcie modelu (`Model`) — pętle po tabelach/miarach/kolumnach, masowe modyfikacje, generowanie nowych obiektów, eksport dokumentacji. To jest odpowiednik pracy z API modelu, bez potrzeby znajomości TOM (Tabular Object Model) w C# poza tym, co widzisz w przykładach.

**Przykład 1 — masowe ustawienie formatu dla wszystkich miar zawierających "Sprzedaz" w nazwie:**

```csharp
foreach (var m in Model.AllMeasures.Where(x => x.Name.Contains("Sprzedaz")))
{
    m.FormatString = "#,##0.00 \"zł\"";
}
```

**Przykład 2 — generowanie standardowego zestawu miar time intelligence dla każdej miary bazowej w folderze "Podstawowe":**

```csharp
foreach (var baseMeasure in Model.Tables["fact_Sprzedaz"].Measures
             .Where(m => m.DisplayFolder == "Podstawowe"))
{
    var ytdName = baseMeasure.Name + " YTD";
    if (Model.Tables["fact_Sprzedaz"].Measures.Contains(ytdName)) continue;

    var newMeasure = Model.Tables["fact_Sprzedaz"].AddMeasure(
        ytdName,
        $"CALCULATE ( [{baseMeasure.Name}], DATESYTD ( dim_Kalendarz[Data] ) )",
        "Czas"
    );
    newMeasure.FormatString = baseMeasure.FormatString;
}
```

To dokładnie odpowiada temu, co pewnie robiłbyś ręcznie kilkanaście razy dla różnych miar sprzedażowych (YTD, MTD, PY) — skrypt robi to raz, dla całej grupy, z zachowaniem formatu.

**Przykład 3 — eksport dokumentacji modelu do CSV (nazwa, tabela, wyrażenie, opis) do dalszej analizy w Pythonie:**

```csharp
var lines = new List<string> { "Table;MeasureName;Expression;Description" };
foreach (var m in Model.AllMeasures)
{
    lines.Add($"{m.Table.Name};{m.Name};{m.Expression.Replace(";", ",").Replace("\n", " ")};{m.Description}");
}
SaveFile("C:\\temp\\dokumentacja_miar.csv", string.Join("\n", lines));
```

Wynikowy plik CSV możesz dalej przetwarzać w `pandas`/`polars` — np. budować automatyczną dokumentację modelu w repo albo szukać duplikatów logiki między miarami.

**Inne typowe zastosowania skryptów w praktyce:**
- Walidacja konwencji nazewnictwa (np. wymuszenie, że wszystkie miary zaczynają się wielką literą, brak spacji podwójnych).
- Masowe przypisanie `Display Folder` na podstawie prefiksu nazwy.
- Generowanie kolumn obliczeniowych do testów (np. flaga walidacyjna) i ich usunięcie po teście — powtarzalny, skryptowalny proces zamiast ręcznego klikania.

---

## 6. Relacje, klucze i integralność referencyjna

To bezpośrednio uzupełnia sekcję o RI Violation z przewodnika DAX Studio — tu je **definiujesz i konfigurujesz**.

**W zakładce Relationships (albo bezpośrednio w Model Explorer → Relationships):**
- **From/To Column** — kolumna klucza obcego (strona "wiele", zwykle `fact_Sprzedaz[ID_Klienta]`) i klucza głównego (strona "jeden", `dim_Klienci[ID_Klienta]`). Klucz po stronie "jeden" musi mieć unikalne wartości — TE2 nie waliduje tego automatycznie (silnik zrobi to dopiero przy przetwarzaniu/refreshu), więc warto mieć pewność co do jakości danych już na poziomie SQL.
- **Cardinality** — One-to-Many (typowe dla gwiazdy), One-to-One (rzadkie, np. tabela rozszerzająca atrybuty), Many-to-Many (świadomie, zwykle przez tabelę pomostową/bridge, nie bezpośrednio między dwiema tabelami faktów).
- **Cross Filter Direction** — Single (standard w gwieździe — filtr płynie tylko od wymiaru do faktu) vs Both (filtr płynie w obie strony — używane ostrożnie, bo może tworzyć niejednoznaczne ścieżki filtrowania przy wielu relacjach naraz; częsty powód nieoczekiwanych wyników w miarach).
- **Is Active** — tylko jedna relacja między parą tabel może być aktywna. Nieaktywne relacje (np. druga relacja data zamówienia vs data dostawy w `fact_Sprzedaz` do `dim_Kalendarz`) wymagają jawnego użycia przez `USERELATIONSHIP` w konkretnej mierze — TE2 pozwala to od razu widzieć i zarządzać wieloma relacjami do tej samej tabeli kalendarza (klasyczny **role-playing dimension**).
- **Assume Referential Integrity** — opcja istotna głównie w DirectQuery/Composite (opisana szerzej w przewodniku DAX Studio, sekcja 6) — w TE2 ustawiasz ją jako właściwość relacji w Properties pane.

**Klucze złożone / tabele pomostowe (bridge tables):** VertiPaq nie wspiera natywnie relacji po wielu kolumnach jednocześnie (composite keys) tak jak SQL Server. Standardowe podejście: scal klucz złożony w jedną kolumnę tekstową (w SQL/Power Query, nie w DAX w locie) i buduj relację po tej jednej scalonej kolumnie — TE2 nie robi tego za Ciebie, ale ułatwia dokumentację takiej kolumny (`Description` wyjaśniający, że to sztuczny klucz złożony) i jej ukrycie (`Is Hidden`), żeby nie myliła użytkowników raportu.

---

## 7. Partycjonowanie tabel

**Zastrzeżenie na start — dotyczy Twojego środowiska:** ręczne partycjonowanie tabel (wiele partycji na tabelę, definiowanych osobno) jest funkcją silnika Analysis Services/Fabric, dostępną gdy model działa na **Premium/Fabric capacity, Azure Analysis Services albo lokalnym SSAS Tabular**. W zwykłym **Power BI Pro (współdzielona pojemność) tabela ma zawsze jedną partycję** — TE2 pokaże Ci ją w drzewie, ale nie dodasz kolejnych z sensem, bo silnik współdzielony i tak przetworzy tabelę jako całość przy odświeżeniu.

**Gdzie partycjonowanie ma realny sens (jeśli masz/będziesz mieć dostęp do Premium/Fabric/Azure AS):**
- **`fact_Sprzedaz`** dzielona np. po roku/miesiącu — pozwala odświeżać tylko najnowszą partycję (np. bieżący miesiąc) zamiast przetwarzać całą historię przy każdym refreshu, co drastycznie skraca czas odświeżania dużych tabel faktów.
- To jest dokładnie mechanizm, na którym oparte jest **Incremental Refresh** w Power BI — pod spodem Power BI Service automatycznie tworzy partycje wg zdefiniowanej polityki (RangeStart/RangeEnd), a TE2 pozwala Ci **zobaczyć te partycje jawnie** po opublikowaniu (w modelu na serwerze, nie w pliku PBIX lokalnie) i np. ręcznie przetworzyć (`Process`) pojedynczą partycję zamiast całej tabeli.

**Praktyczne operacje w TE2 (Model Explorer → Table → Partitions):**
- Podgląd zapytania źródłowego (M lub natywne SQL) dla każdej partycji osobno.
- Ręczne dodanie partycji z innym zapytaniem źródłowym niż reszta tabeli (rzadziej potrzebne przy Power BI, częstsze przy Azure AS).
- **Process Partition** (prawy klik → Process) — przetworzenie/odświeżenie pojedynczej partycji bez ruszania reszty tabeli — przydatne przy debugowaniu, czy dana partycja w ogóle poprawnie się ładuje, zanim zlecisz pełny refresh całego modelu.

**Dla Twojego stacku (SQL Server jako źródło):** jeśli docelowo pracujesz/będziesz pracować na Premium/Fabric, warto rozważyć **Incremental Refresh z partycjonowaniem po dacie** dla `fact_Sprzedaz` już teraz, projektując kolumnę daty w SQL pod kątem filtrów `RangeStart`/`RangeEnd` — to jest jedna z tych decyzji architektonicznych, które są tanie na starcie projektu, a drogie do wprowadzenia post factum na dużym, działającym modelu.

---

## 8. Perspektywy i tłumaczenia

- **Perspektywy** (Perspectives) — podzbiory modelu (wybrane tabele/kolumny/miary) widoczne dla użytkownika przy łączeniu z modelem przez Excel/inne narzędzia klienckie. Nie są mechanizmem bezpieczeństwa (nie ukrywają danych, tylko upraszczają widok) — dobre np. dla oddzielenia perspektywy "Sprzedaż" od "Finanse", jeśli model rośnie i obsługuje wiele działów.
- **Translations** — tłumaczenia nazw obiektów (tabel, kolumn, miar) na inne języki/kultury, przełączane automatycznie w zależności od ustawień klienta raportu. Rzadziej potrzebne w kontekście czysto polskojęzycznej organizacji, ale warto wiedzieć, że to jedyne sensowne narzędzie do zarządzania tym poza ręczną edycją XML modelu.

---

## 9. Row-Level Security (RLS) i Object-Level Security (OLS)

- **Roles** (Model Explorer → Roles) — definiujesz rolę, przypisujesz do niej filtr DAX na poziomie tabeli (np. na `dim_Placowki`: `[ID_Placowki] = LOOKUPVALUE(...)` w oparciu o `USERPRINCIPALNAME()`), a następnie przypisujesz użytkowników/grupy AD po publikacji w Power BI Service.
- **Object-Level Security (OLS)** — ukrywanie/blokowanie dostępu do konkretnych **kolumn lub tabel** dla danej roli (nie tylko wierszy) — ustawiasz to w Properties danej kolumny w kontekście wybranej roli (`Object Level Security` w panelu właściwości po zaznaczeniu roli).
- **Ograniczenie darmowej wersji**: TE2 **nie ma** wbudowanego testera "Analyze as Role" pokazującego wynik z perspektywy konkretnego użytkownika bezpośrednio w interfejsie tak wygodnie jak niektóre płatne narzędzia — testowanie RLS praktycznie robisz albo w Power BI Desktop (`View As Roles`), albo w **DAX Studio** przez ustawienie `EffectiveUserName` w connection string / `EVALUATE` z rolą — to kolejny punkt, gdzie DAX Studio i TE2 się uzupełniają: definiujesz RLS w TE2, testujesz efekt w DAX Studio.

---

## 10. Find & Replace i Dependency View — refaktoryzacja na skalę modelu

- **Ctrl+Shift+F (Find & Replace)** — szuka tekstu **we wszystkich wyrażeniach DAX modelu jednocześnie** (miary, kolumny obliczane, role RLS), z opcją zamiany. To jest kluczowe narzędzie przy refaktoryzacji: zmiana nazwy tabeli lub kolumny w modelu wymaga aktualizacji każdej miary, która się do niej odwołuje — ręcznie w Power BI Desktop musiałbyś przeklikać każdą miarę osobno, tu robisz to jednym poleceniem dla całego modelu.
- **Show Dependencies** (prawy klik na mierze/kolumnie → Show Dependencies) — pokazuje graf: od czego dana miara zależy (inne miary, kolumny) i co zależy od niej. Bardzo przydatne przed usunięciem lub zmianą logiki miary bazowej (np. `Sprzedaz Total`), żeby wiedzieć, ile innych miar (TOP10, TOP10%, YTD itd.) się na niej opiera i mogłoby się "zepsuć" po zmianie.

---

## 11. Formatowanie DAX

**Ctrl+Shift+F** w oknie edycji wyrażenia (nie mylić z Find & Replace, które działa w widoku drzewa) formatuje pojedyncze wyrażenie DAX przez integrację z DAX Formatter — identyczny silnik formatujący jak w DAX Studio, więc styl kodu zostaje spójny między obydwoma narzędziami.

---

## 12. Deployment i automatyzacja (CLI)

- **Save (Ctrl+S)** przy modelu podłączonym do Power BI Desktop — zapisuje zmiany bezpośrednio do otwartego pliku PBIX.
- **Model → Deploy** przy modelu podłączonym do serwera (Azure AS / SSAS / Fabric XMLA) — generuje i wykonuje skrypt TMSL (Tabular Model Scripting Language) aktualizujący model na serwerze, z opcją podglądu różnic przed wdrożeniem.
- **Command-line automation (`TabularEditor.exe -S skrypt.cs -B plik.bim`)** — to jest funkcja, która wyróżnia darmową wersję na tle konkurencji: możesz uruchamiać skrypty C# (te same co w panelu, sekcja 5) z linii poleceń, co pozwala budować **pipeline CI/CD** dla modelu tabularnego — np. automatyczne uruchomienie BPA i przerwanie deploymentu, jeśli są naruszenia reguł, albo automatyczne wygenerowanie dokumentacji przy każdym pushu do repo. Jeśli rozwijasz Airflow w swoim stacku, to naturalne miejsce integracji: task w DAG-u wywołujący `TabularEditor.exe` w trybie CLI jako krok walidacji/deploymentu modelu.

---

## Uzupełnienie: podział pracy DAX Studio vs Tabular Editor — pełny obraz

| Zadanie | Narzędzie |
|---|---|
| Pomiar wydajności miary (FE/SE, Query Plan, CallbackDataID) | **DAX Studio** |
| Analiza rozmiaru/kompresji modelu (VertiPaq Analyzer) | **DAX Studio** (View Metrics dostępny też z poziomu TE2 przez integrację, ale pełna analiza wygodniejsza w DAX Studio) |
| Edycja definicji miar/kolumn, masowe zmiany właściwości | **Tabular Editor** |
| Definiowanie relacji, RI, partycji, RLS/OLS | **Tabular Editor** |
| Statyczny audyt dobrych praktyk modelowania (BPA) | **Tabular Editor** |
| Testowanie RLS z perspektywy użytkownika | DAX Studio (`EffectiveUserName`) lub Power BI Desktop |
| Refaktoryzacja nazw w całym modelu (Find & Replace) | **Tabular Editor** |
| Automatyzacja/CI-CD deploymentu modelu | **Tabular Editor** (CLI) |

W praktyce: projektujesz i porządkujesz model w Tabular Editor, mierzysz i optymalizujesz wydajność konkretnych miar w DAX Studio, a decyzje o architekturze (partycjonowanie, RI, relacje) zapadają na podstawie tego, co pokaże Ci runtime diagnostyka z DAX Studio — te dwa narzędzia najlepiej działają w parze, nie zamiennie.

---

## 13. Więcej przydatnych skryptów C# (Advanced Scripting)

Kontynuacja sekcji 5 — kolejne wzorce skryptów, które realnie przyspieszają pracę przy modelu takim jak Twój.

**Przykład 4 — audyt: lista miar bez opisu i bez ustawionego formatu (szybki raport jakości modelu przed BPA):**

```csharp
var brakiOpisu = Model.AllMeasures
    .Where(m => string.IsNullOrWhiteSpace(m.Description))
    .Select(m => $"{m.Table.Name}[{m.Name}]");

var brakiFormatu = Model.AllMeasures
    .Where(m => string.IsNullOrWhiteSpace(m.FormatString))
    .Select(m => $"{m.Table.Name}[{m.Name}]");

Info($"Brak opisu ({brakiOpisu.Count()}):\n" + string.Join("\n", brakiOpisu));
Info($"Brak formatu ({brakiFormatu.Count()}):\n" + string.Join("\n", brakiFormatu));
```

**Przykład 5 — masowe dodanie miar "poprzedni rok" (PY) dla wszystkich miar z folderu "Podstawowe", analogicznie do YTD z sekcji 5:**

```csharp
foreach (var baseMeasure in Model.Tables["fact_Sprzedaz"].Measures
             .Where(m => m.DisplayFolder == "Podstawowe"))
{
    var pyName = baseMeasure.Name + " PY";
    if (Model.Tables["fact_Sprzedaz"].Measures.Contains(pyName)) continue;

    var newMeasure = Model.Tables["fact_Sprzedaz"].AddMeasure(
        pyName,
        $"CALCULATE ( [{baseMeasure.Name}], SAMEPERIODLASTYEAR ( dim_Kalendarz[Data] ) )",
        "Czas"
    );
    newMeasure.FormatString = baseMeasure.FormatString;
}
```

**Przykład 6 — wyszukanie kolumn, które nie są używane w żadnej relacji ani żadnej mierze/kolumnie obliczanej (kandydaci do ukrycia lub usunięcia):**

```csharp
foreach (var t in Model.Tables)
{
    foreach (var c in t.Columns.OfType<DataColumn>())
    {
        bool wRelacji = Model.Relationships.Any(r =>
            r.FromColumn == c || r.ToColumn == c);
        bool referowanaPrzezInne = Model.AllMeasures
            .Any(m => m.DependsOn.Columns.Contains(c));

        if (!wRelacji && !referowanaPrzezInne && !c.IsHidden)
        {
            Output($"Nieużywana, widoczna kolumna: {t.Name}[{c.Name}]");
        }
    }
}
```

(Uwaga: `DependsOn` wymaga, aby model miał zbudowane/odświeżone zależności — w razie błędu użyj `Model.SaveDB()` albo odśwież zależności z menu przed uruchomieniem).

**Przykład 7 — optymalizacja pamięci: wyłączenie `IsAvailableInMdx` (attribute hierarchies) dla kolumn o wysokiej kardynalności, które nie są używane w sliderach/wizualach jako pojedyncze wartości:**

```csharp
foreach (var c in Model.Tables["fact_Sprzedaz"].Columns.OfType<DataColumn>())
{
    if (c.Name.Contains("ID") && c.IsHidden)
    {
        c.IsAvailableInMDX = false;   // usuwa domyślną hierarchię atrybutu, oszczędza pamięć
    }
}
```

To realna optymalizacja rozmiaru modelu — silnik domyślnie buduje dodatkową strukturę (attribute hierarchy) dla każdej kolumny, nawet jeśli jest ukryta i używana wyłącznie jako klucz relacji. Wyłączenie tego dla kolumn technicznych zmniejsza rozmiar modelu bez wpływu na funkcjonalność raportów.

**Przykład 8 — eksport wszystkich relacji modelu do CSV (dokumentacja architektury):**

```csharp
var lines = new List<string> { "FromTable;FromColumn;ToTable;ToColumn;Cardinality;CrossFilter;IsActive" };
foreach (var r in Model.Relationships.OfType<SingleColumnRelationship>())
{
    lines.Add($"{r.FromColumn.Table.Name};{r.FromColumn.Name};{r.ToColumn.Table.Name};{r.ToColumn.Name};{r.FromCardinality}-{r.ToCardinality};{r.CrossFilteringBehavior};{r.IsActive}");
}
SaveFile("C:\\temp\\relacje_modelu.csv", string.Join("\n", lines));
```

**Przykład 9 — walidacja konwencji nazewnictwa (np. brak podwójnych spacji, brak nazw kończących się spacją) z automatyczną korektą:**

```csharp
foreach (var m in Model.AllMeasures)
{
    var poprawiona = System.Text.RegularExpressions.Regex.Replace(m.Name.Trim(), @"\s+", " ");
    if (poprawiona != m.Name)
    {
        Output($"Poprawiam nazwę: '{m.Name}' -> '{poprawiona}'");
        m.Name = poprawiona;
    }
}
```

Wszystkie powyższe warto trzymać jako osobne pliki `.cs` w repo (np. `scripts/te2/`) i uruchamiać zarówno ręcznie z poziomu panelu C# Script, jak i z linii poleceń (sekcja 12) — to ten sam kod w obu przypadkach.

---

## 14. Tabular Editor 3.x — co dodaje płatna wersja

Tabular Editor 3 (TE3) to osobny, komercyjny produkt (nie aktualizacja TE2 "w miejscu") — przepisany od podstaw jako pełne IDE do pracy z modelami tabularycznymi. TE2 pozostaje darmowy i rozwijany równolegle — nie jest to "wersja okrojona TE3", tylko odrębne narzędzie o innej filozofii (lekkie wsparcie vs kompletne środowisko).

**Edycje TE3** (licencjonowanie warstwowe):
- **Desktop** — najniższy płatny poziom.
- **Business** — pośredni poziom (m.in. perspektywy i wiele partycji dla modeli Power BI, przy trybie zgodności ustawionym na PowerBI).
- **Enterprise** — pełny zakres, wymagany m.in. dla modeli na Azure AS/SSAS korzystających z perspektyw, wielu partycji czy DirectQuery na niższych warstwach licencyjnych SQL Server/Azure AS, oraz dla modeli Power BI Premium-Per-User korzystających z Direct Lake. Enterprise obejmuje też dostęp do DAX Optimizer.
- Dostępny jest 30-dniowy pełny trial odpowiadający edycji Enterprise (bez DAX Optimizera).

### 14.1 Nowoczesny edytor DAX — realna zmiana jakości pracy

- **Peek i go-to-definition** — podgląd wyrażenia miary bez opuszczania kontekstu edycji + szybka nawigacja (`Alt+Left`/`Alt+Right`) między zależnymi/referencyjnymi miarami. W praktyce: klikasz na `[Sprzedaz Total]` użyte wewnątrz `[Sprzedaz TOP10]` i od razu widzisz jego definicję, bez przełączania obiektów w drzewie.
- **IntelliSense i podpowiedzi kontekstowe** — pełne autouzupełnianie funkcji/kolumn/miar z opisem parametrów w locie, czego TE2 (i formularz miary w Power BI Desktop) nie oferuje w takim zakresie.
- **Code Actions** — automatyczne sugestie poprawy kodu DAX (czytelność i wydajność), np. wykrywanie zbędnych `CALCULATE` czy wzorców, które można zapisać wydajniej — TE3 podpowiada poprawkę do zaakceptowania jednym kliknięciem, analogicznie do "quick fixes" w nowoczesnych IDE.
- **Find/Replace z wyrażeniami regularnymi po całym modelu** — rozszerzenie mechanizmu z sekcji 10 o pełne wsparcie regex.
- **Refaktoryzacja zmiennych** (`Ctrl+R`) i zaznaczanie wszystkich wystąpień słowa — typowe funkcje refaktoryzacyjne znane z nowoczesnych edytorów kodu, przeniesione do kontekstu DAX.
- **DAX scripts** — edycja wielu obiektów DAX naraz jako jeden dokument tekstowy (koncepcyjny poprzednik formatu TMDL) — możesz np. sformatować cały model jednym przyciskiem.

### 14.2 Zintegrowana optymalizacja — bez przełączania się do DAX Studio

- **VertiPaq Analyzer wbudowany bezpośrednio w interfejs** — nie tylko rozmiar/kardynalność/encoding (jak w DAX Studio), ale dodatkowo pomoc w namierzaniu przyczyn pustych wartości (`(Blank)`) wynikających z naruszeń integralności referencyjnej relacji (bezpośrednie powiązanie z sekcją 6 tego przewodnika), walidację danych per partycja przy incremental refresh, oraz tzw. "temperaturę" kolumn — czyli które kolumny są faktycznie najczęściej odpytywane przez zapytania, co pomaga priorytetyzować optymalizację.
- **DAX Optimizer** (tylko Enterprise) — automatyczne skanowanie modelu pod kątem wąskich gardeł wydajnościowych w zapytaniach DAX, z konkretnymi rekomendacjami.
- **Okno zapytań DAX z debuggerem krok-po-kroku** — możesz wykonywać zapytanie częściowo (fragment po fragmencie), zapisywać/eksportować wyniki, oraz debugować złożone zapytania (np. wklejone z Performance Analyzer w Power BI Desktop) linia po linii. Ważne rozróżnienie: to narzędzie do znajdowania *poprawnych* wyników (debugowanie logiki), nie do mierzenia *szybkości* — do tego drugiego nadal najlepszym narzędziem pozostaje DAX Studio (Server Timings/Query Plan z Twojego pierwszego przewodnika).

### 14.3 Rozbudowane wersje funkcji znanych z TE2

- **Edytory perspektyw i tłumaczeń** — dedykowane widoki graficzne zamiast surowej listy checkboxów jak w TE2, ułatwiające masowe zarządzanie członkostwem obiektów w wielu perspektywach naraz.
- **Śledzenie zależności powiązane z eksploratorem modelu** — panel Dependency automatycznie aktualizuje się wraz z zaznaczeniem w drzewie obiektów, bez konieczności każdorazowego "Show Dependencies" z menu kontekstowego.
- **Bulk rename** (batch rename children) — obecne też w TE2 (prawy klik na tabeli → "Batch rename children"), ale w TE3 zintegrowane z pełnym silnikiem find/replace regex.

### 14.4 Udogodnienia praktyczne (jakość życia, nie tylko nowe możliwości)

- **Table groups** — organizacja tabel w foldery (np. `Fakty`, `Wymiary`, `Parametry` — dokładnie jak Twoje dwie tabele parametryczne `dim_Liczba_miesiecy`/`dim_%_sprzedazy` mogłyby siedzieć osobno od właściwych wymiarów). Uwaga: to widoczne tylko w TE3, nie przenosi się do struktury folderów w samym Power BI Desktop.
- **Konfigurowalne, zapisywalne układy okien** (multi-monitor, popout) — osobne układy pod różne tryby pracy: modelowanie, audyt, optymalizacja, edycja perspektyw/tłumaczeń.
- **Motywy i tryb ciemny.**
- **AI Assistant** (funkcja wprowadzona w 2026) — wspomaga pisanie DAX, generowanie skryptów C#, uruchamianie kontroli BPA i odpytywanie statystyk modelu; działa w modelu "bring-your-own" dostawcy AI z kontrolą zgody na to, co jest udostępniane.
- **Tabular Editor CLI** — command-line interface (rozwinięcie automatyzacji znanej z TE2, opisanej w sekcji 12), z myślą o pipeline'ach CI/CD w środowiskach Power BI/Fabric/Analysis Services na wielu platformach.

### 14.5 Czy warto — perspektywa Twojego stacku

Skoro pracujesz już z Tabular Editor (darmowym) obok DAX Studio i planujesz rozwój w stronę bardziej zaawansowanych/zespołowych scenariuszy (Fabric, większe modele, częstsze iteracje nad miarami), największą realną korzyść z TE3 dałyby: **zintegrowany VertiPaq Analyzer z wykrywaniem RI Violation i "temperatury" kolumn** (mniej przełączania się między narzędziami) oraz **Code Actions/IntelliSense** przy pisaniu bardziej złożonych miar. Dla jednoosobowej, niezespołowej pracy nad pojedynczym modelem sprzedażowym TE2 + DAX Studio w połączeniu, jak opisane w obu przewodnikach, pokrywa większość realnych potrzeb bez kosztu licencji — TE3 zaczyna się bardziej opłacać przy pracy zespołowej, większej skali modeli lub częstej pracy z Azure AS/Fabric, gdzie dochodzą ograniczenia licencyjne niedostępne w darmowej wersji (np. wymagane funkcje Enterprise dla niektórych konfiguracji SSAS/Azure AS).
