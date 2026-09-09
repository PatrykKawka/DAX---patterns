# Zaawansowane wzorce DAX — TREATAS, dynamiczna segmentacja, miary semi-addytywne

> **Uwaga metodologiczna:** jak w pozostałych dokumentach o DAX w tym repo — brak środowiska Power BI/DAX Studio w tej sesji, nic z poniższego nie zostało przeliczone na żywo. Wzorce opisane niżej są stabilnymi, szeroko udokumentowanymi technikami (część z nich pochodzi z powszechnie znanych rozwiązań SQLBI), ale zweryfikuj je na swoim modelu przed wdrożeniem produkcyjnym.

## `TREATAS` — relacje wirtualne bez fizycznej relacji w modelu

### Problem, który rozwiązuje

Model gwiazdy zwykle wymaga fizycznej relacji między tabelą faktów a wymiarem, żeby filtr się propagował. Czasem fizyczna relacja jest niemożliwa albo niepożądana:
- Tabela budżetu ma inną granularność niż tabela sprzedaży (budżet per kategoria/miesiąc, sprzedaż per produkt/dzień) — nie da się połączyć relacją 1:N w prosty sposób.
- Dwie tabele faktów muszą filtrować się nawzajem przez wspólny wymiar, ale model już ma jedną aktywną relację do tego wymiaru (Power BI pozwala tylko na JEDNĄ aktywną relację między dwiema tabelami).
- Dane pochodzą z różnych źródeł, mają wspólną kolumnę (np. kod produktu), ale nie warto tworzyć fizycznej relacji dla jednorazowego use case'u.

### Mechanizm

`TREATAS` bierze wynik tabelaryczny (np. z `VALUES`) i \"udaje\", że to wartości pochodzące z INNEJ kolumny — efektywnie tworząc filtr, tak jakby istniała relacja, bez faktycznego jej definiowania w modelu.

```dax
Budget Amount =
CALCULATE(
    SUM(Budget[Amount]),
    TREATAS(VALUES(Sales[ProductCategory]), Budget[ProductCategory])
)
```

To wyrażenie mówi: \"weź kategorie produktów widoczne w bieżącym kontekście filtra (z tabeli `Sales`), i zastosuj je jako filtr na kolumnie `Budget[ProductCategory]`\" — mimo że `Sales` i `Budget` nie są połączone fizyczną relacją.

### Kiedy używać zamiast fizycznej relacji

- Model budżetowy/planistyczny o innej granularności niż fakty rzeczywiste.
- Obejście limitu jednej aktywnej relacji (np. tabela dat połączona już z `OrderDate`, a potrzebujesz też filtrować po `ShipDate` bez przełączania aktywności relacji przez `USERELATIONSHIP` — `TREATAS` bywa czytelniejszą alternatywą w niektórych przypadkach).
- Dane z różnych źródeł, gdzie tworzenie formalnej relacji fizycznej byłoby nadmiarowe dla jednej, konkretnej miary.

---

## Dynamiczna segmentacja: kategorie liczone w locie, nie na stałe w modelu

### Problem, który rozwiązuje

Standardowe podejście — kolumna obliczeniowa `CustomerSegment` w tabeli klientów, na stałe przypisująca każdego klienta do przedziału wartości. Działa, ale użytkownik NIE MOŻE zmienić progów segmentacji bez przeładowania modelu. Dynamiczna segmentacja przenosi tę logikę do miary, liczonej na bieżąco w kontekście filtra.

### Wzorzec: tabela pomocnicza + `SWITCH`

```dax
-- Tabela pomocnicza (nie relacyjna, tylko do obsługi wyboru w slicerze)
SegmentThresholds =
DATATABLE(
    "SegmentName", STRING,
    "MinValue", INTEGER,
    "MaxValue", INTEGER,
    {
        {"Niski", 0, 999},
        {"Średni", 1000, 4999},
        {"Wysoki", 5000, 999999999}
    }
)

-- Miara przypisująca klienta do segmentu W LOCIE, na podstawie bieżącej sprzedaży
Customer Segment (dynamic) =
VAR CustomerValue = [Total Sales]
RETURN
    SWITCH(
        TRUE(),
        CustomerValue < 1000, "Niski",
        CustomerValue < 5000, "Średni",
        "Wysoki"
    )
```

Zaleta względem kolumny obliczeniowej: progi (`1000`, `5000`) można wystawić jako parametry `What-If` w Power BI, dając użytkownikowi realną kontrolę nad definicją segmentów bez ingerencji w model. Koszt: to iterator wykonywany dla każdego klienta w bieżącym kontekście (patrz dokument o iteratorach) — przy bardzo dużej liczbie klientów warto sprawdzić wydajność w DAX Studio.

---

## Miary semi-addytywne: salda i stany, które NIE sumują się w czasie

### Problem

Sprzedaż jest w pełni addytywna — suma sprzedaży ze stycznia i lutego to poprawna suma za styczeń-luty. **Stan magazynu**, **saldo konta**, **liczba aktywnych pracowników na koniec miesiąca** — NIE są addytywne w czasie. Zsumowanie stanu magazynu z każdego dnia miesiąca daje liczbę bez sensu (to tak, jakby liczyć \"sumę sald konta bankowego z każdego dnia miesiąca\" zamiast salda na koniec miesiąca).

### Wzorzec: ostatnia znana wartość w okresie, nie suma

```dax
-- Błędnie: to zsumuje stan magazynu ze WSZYSTKICH dni w bieżącym kontekście
Stock Level (WRONG) = SUM(Inventory[StockLevel])

-- Poprawnie: wartość z OSTATNIEGO dnia bieżącego kontekstu czasowego
Stock Level (correct) =
CALCULATE(
    SUM(Inventory[StockLevel]),
    LASTDATE('Date'[Date])
)
```

`LASTDATE` zwraca ostatnią datę w bieżącym kontekście filtra (np. ostatni dzień wybranego miesiąca) — miara liczy stan TYLKO na ten jeden dzień, ignorując resztę okresu. To poprawnie oddaje semi-addytywny charakter miary: agreguje się normalnie po wymiarach nieczasowych (np. po magazynie/produkcie), ale po osi czasu bierze tylko ostatnią wartość, nie sumę.

### Wariant: ostatnia NIEPUSTA wartość

Jeśli w niektóre dni brak jest pomiaru (np. inwentaryzacja nie odbywa się codziennie), `LASTDATE` może trafić na dzień bez danych. `LASTNONBLANK` szuka ostatniego dnia, dla którego wyrażenie faktycznie zwraca wartość:

```dax
Stock Level (last non-blank) =
CALCULATE(
    SUM(Inventory[StockLevel]),
    LASTNONBLANK('Date'[Date], SUM(Inventory[StockLevel]))
)
```

---

## Pułapki

### Pułapka 1 — `TREATAS` z niezgodnymi typami danych daje CICHY brak wyników, nie błąd

Jeśli kolumna źródłowa i docelowa w `TREATAS` mają różne typy danych (np. tekst vs liczba, albo różne formatowanie tego samego tekstu — spacje, wielkość liter), filtr po prostu nie dopasuje żadnych wierszy — miara zwróci pustkę, bez żadnego komunikatu o błędzie. Warto to świadomie sprawdzić przy pierwszym wdrożeniu `TREATAS` na nowej parze kolumn.

### Pułapka 2 — dynamiczna segmentacja jako iterator na dużej tabeli klientów bywa kosztowna

`SWITCH(TRUE(), ...)` wykonywany dla każdego klienta w kontekście (patrz dokument o iteratorach) — przy setkach tysięcy klientów i złożonej logice segmentacji warto sprawdzić w DAX Studio, czy koszt jest akceptowalny, zanim wdroży się to na produkcyjnym dashboardzie z dużym ruchem.

### Pułapka 3 — semi-addytywność pomylona ze zwykłą sumą to jeden z częstszych błędów raportowania

Miara stanu magazynowego/salda zbudowana jako zwykły `SUM` (bez `LASTDATE`/`LASTNONBLANK`) da liczbowo \"poprawnie wyglądający\", ale merytorycznie bezsensowny wynik przy zsumowaniu po miesiącu/kwartale — klasyczny przykład liczby, która \"działa\" na poziomie dnia (gdzie suma i ostatnia wartość to to samo), ale psuje się dopiero przy agregacji do wyższego poziomu czasu, co bywa przeoczone podczas testowania miary tylko na granularności dziennej.

---

## Podsumowanie

| Zadanie | Wzorzec |
|---|---|
| Filtrowanie tabeli bez fizycznej relacji do wymiaru | `TREATAS(VALUES(...), InnaTabela[Kolumna])` |
| Model budżetowy o innej granularności niż fakty | `TREATAS` między wspólną kolumną kategoryzującą |
| Segmentacja z progami zmienianymi przez użytkownika | Tabela pomocnicza (`DATATABLE`/parametr What-If) + `SWITCH(TRUE(), ...)` |
| Stan magazynu / saldo konta / inny wskaźnik semi-addytywny | `CALCULATE(SUM(...), LASTDATE(...))` lub `LASTNONBLANK` |
| Braki w codziennym pomiarze przy mierze semi-addytywnej | `LASTNONBLANK` zamiast `LASTDATE` |

**Wniosek:** wszystkie trzy wzorce w tym dokumencie łączy jedno — to techniki na sytuacje, gdzie \"oczywiste\" podejście (fizyczna relacja, kolumna obliczeniowa, zwykła suma) nie wystarcza z powodu ograniczeń modelu albo natury samych danych. Warto sięgać po nie świadomie, dopiero gdy prostsze podejście faktycznie zawodzi — nie jako domyślny, bardziej \"zaawansowany\" wybór.
