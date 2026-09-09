# Iteratory a wydajność w DAX — SUMX/AVERAGEX/FILTER, VAR

> **Uwaga metodologiczna:** ten dokument nie został wykonany na żywym modelu Power BI/DAX Studio w tej sesji (brak takiego środowiska tutaj). Mechanika opisana niżej (storage engine vs formula engine, context transition, działanie `VAR`) to stabilna, dobrze udokumentowana architektura silnika VertiPaq/DAX — ale zanim potraktujesz konkretną rekomendację jako pewnik na swoich danych, zweryfikuj ją w DAX Studio (zakładka Server Timings) na swoim modelu. Różnice wydajności zależą od rozmiaru tabel, kardynalności kolumn i istniejących relacji — nie da się ich uczciwie zmierzyć w oderwaniu od realnego modelu.

## Dwa silniki pod maską: storage engine vs formula engine

Każde zapytanie DAX jest wykonywane przez dwa współpracujące silniki:
- **Storage engine (VertiPaq)** — silnik kolumnowy, bardzo szybki w prostych agregacjach (`SUM`, `COUNT`, `MIN`, `MAX`) i filtrach na pojedynczych kolumnach. Działa na skompresowanych danych w pamięci.
- **Formula engine** — obsługuje wszystko, czego storage engine nie potrafi zrobić natywnie: iteratory z złożoną logiką, funkcje czasu, zagnieżdżone `CALCULATE`. Wolniejszy, bo działa wiersz po wierszu (albo na wirtualnych tabelach materializowanych w pamięci).

**Konsekwencja praktyczna:** im więcej pracy uda się "zepchnąć" do storage engine, tym szybsza miara. Cała reszta tego dokumentu to w gruncie rzeczy różne sposoby na to zepchnięcie — albo ostrzeżenia, gdzie formula engine włącza się niepotrzebnie.

---

## `SUMX` vs `SUM`: ten sam wynik, inny mechanizm

```dax
-- Agregator natywny - storage engine, szybkie
Total Sales (SUM) = SUM(Sales[Amount])

-- Iterator - formula engine, wolniejsze, choć wynik IDENTYCZNY
Total Sales (SUMX) = SUMX(Sales, Sales[Amount])
```

Dla prostej sumy JEDNEJ kolumny oba wyrażenia dają ten sam wynik — ale `SUMX` tworzy **kontekst wiersza** i iteruje po tabeli w formula engine, podczas gdy `SUM` to natywna operacja agregująca w storage engine. Na dużej tabeli faktów różnica bywa odczuwalna. **Zasada:** `SUMX` tylko wtedy, gdy naprawdę potrzebujesz obliczenia PER WIERSZ przed zsumowaniem (np. `Ilość × Cena`, gdzie żadna z tych kolumn osobno nie daje właściwego wyniku):

```dax
-- Tu SUMX jest KONIECZNE - Amount nie istnieje jako gotowa kolumna
Total Revenue = SUMX(Sales, Sales[Quantity] * Sales[UnitPrice])
```

---

## `FILTER` vs filtr bezpośrednio w `CALCULATE`

```dax
-- Wolniejszy wariant: FILTER materializuje CAŁĄ tabelę w formula engine,
-- wiersz po wierszu sprawdzając warunek
Sales North (FILTER) =
CALCULATE(
    SUM(Sales[Amount]),
    FILTER(Sales, Sales[Region] = "North")
)

-- Szybszy wariant: warunek boolowski wprost w CALCULATE - silnik może
-- zepchnąć filtrowanie do storage engine
Sales North (direct) =
CALCULATE(
    SUM(Sales[Amount]),
    Sales[Region] = "North"
)
```

Obie formy dają identyczny wynik. Różnica: `FILTER` w formula engine sprawdza warunek dla każdego wiersza po kolei; filtr boolowski bezpośrednio w `CALCULATE` może zostać zoptymalizowany i wykonany w storage engine (jako natywny filtr kolumnowy VertiPaq). **Zasada:** jeśli warunek filtra dotyczy JEDNEJ kolumny i prostego porównania, prawie zawsze da się go zapisać bezpośrednio w `CALCULATE` — zostaw `FILTER` na sytuacje, gdzie warunek jest naprawdę złożony (patrz niżej).

### Kiedy `FILTER` jest naprawdę potrzebny

- Warunek łączący WIELE kolumn w logice, której nie da się wyrazić jako prosty filtr na jednej kolumnie:
  ```dax
  FILTER(Sales, Sales[Amount] > Sales[Budget] * 1.1)
  ```
- Warunek na poziomie AGREGACJI per grupa (np. "zostaw tylko klientów, których suma zakupów przekracza próg") — to wymaga materializacji tabeli z policzoną agregacją per wiersz grupy, więc `FILTER` jest tu naturalnym narzędziem:
  ```dax
  FILTER(
      VALUES(Customer[CustomerID]),
      CALCULATE(SUM(Sales[Amount])) > 10000
  )
  ```

---

## `VAR`: materializacja wyniku RAZ, nie za każdym odwołaniem

Bez `VAR`, powtórzone w tym samym wyrażeniu podwyrażenie jest **przeliczane od nowa przy każdym użyciu**:

```dax
-- Bez VAR: [Total Sales] i [Total Cost] mogą zostać policzone wielokrotnie,
-- jeśli wyrażenie jest bardziej złożone niż w tym uproszczonym przykładzie
Margin % (bez VAR) =
DIVIDE(
    SUM(Sales[Amount]) - SUM(Sales[Cost]),
    SUM(Sales[Amount])
)

-- Z VAR: SUM(Sales[Amount]) policzone RAZ, wynik zapamiętany i użyty dwukrotnie
Margin % (z VAR) =
VAR TotalSales = SUM(Sales[Amount])
VAR TotalCost = SUM(Sales[Cost])
RETURN
    DIVIDE(TotalSales - TotalCost, TotalSales)
```

Przy prostych agregatorach jak w tym przykładzie silnik często i tak optymalizuje powtórzenie — ale przy bardziej złożonych wyrażeniach (zagnieżdżone `CALCULATE`, iteratory, funkcje czasu) `VAR` bywa różnicą między jednym a kilkoma pełnymi przejściami przez silnik dla tego samego podwyrażenia. **Dodatkowa korzyść, niezależna od wydajności:** `VAR` z opisową nazwą (`TotalSales` zamiast powtórzonego `SUM(Sales[Amount])`) czyni miarę czytelniejszą i łatwiejszą do debugowania krok po kroku.

### Ważna, częsta pomyłka: `VAR` NIE zamraża kontekstu filtra

`VAR` przechowuje WYNIK obliczenia, ale samo obliczenie odbywa się w kontekście filtra obowiązującym W MIEJSCU DEFINICJI zmiennej — nie \"zamraża\" niejawnie żadnego wcześniejszego stanu poza tym, co i tak wynika z normalnych zasad kontekstu. Jeśli chcesz świadomie zapamiętać wartość SPRZED zmiany kontekstu (np. przed `CALCULATE` z nowym filtrem), musisz to zrobić jawnie — sama zmienna tego automatycznie nie gwarantuje inaczej niż zwykłe wyrażenie DAX.

---

## Receptura — sprawdzenie kosztu miary w DAX Studio

1. Otwórz zakładkę **Server Timings** w DAX Studio.
2. Uruchom miarę (np. przez wklejenie zapytania `EVALUATE`).
3. Sprawdź proporcję czasu **Storage Engine (SE)** do **Formula Engine (FE)** oraz liczbę zapytań SE.
4. Duża liczba osobnych zapytań SE (zamiast jednego zbiorczego) często sygnalizuje niepotrzebny iterator formula engine tam, gdzie wystarczyłby natywny agregator.
5. Wysoki % czasu FE względem SE — sygnał do przejrzenia miary pod kątem zbędnych `FILTER`/zagnieżdżonych iteratorów opisanych w tym dokumencie.

---

## Pułapki

### Pułapka 1 — wywołanie miary wewnątrz iteratora wymusza context transition dla KAŻDEGO wiersza

```dax
-- Kosztowne: [Total Sales] jest miarą - jej wywołanie wewnątrz SUMX
-- wymusza context transition (wiersz -> kontekst filtra) dla KAŻDEGO wiersza tabeli
Avg Customer Value (kosztowne) =
AVERAGEX(
    VALUES(Customer[CustomerID]),
    [Total Sales]
)
```

To bywa konieczne i poprawne (to naturalny sposób liczenia \"średniej wartości na klienta\"), ale warto mieć świadomość, że przy dużej liczbie klientów oznacza to tyle context transitions, ile wierszy w `VALUES(Customer[CustomerID])` — każde relatywnie kosztowne. Jeśli tabela klientów jest bardzo duża, warto rozważyć, czy nie da się tego przeformułować przez bezpośrednią agregację bez wywołania miary per wiersz.

### Pułapka 2 — `FILTER` na dużej tabeli faktów zamiast na małej tabeli wymiaru

```dax
-- Wolniej: FILTER materializuje warunek na potencjalnie milionach wierszy faktów
CALCULATE(SUM(Sales[Amount]), FILTER(Sales, RELATED(Product[Category]) = "Electronics"))

-- Szybciej: filtrowanie na małej tabeli wymiaru, propagacja przez relację
CALCULATE(SUM(Sales[Amount]), FILTER(Product, Product[Category] = "Electronics"))
```

Filtrowanie po stronie tabeli wymiaru (zwykle rzędy wielkości mniejszej niż tabela faktów) i poleganie na propagacji filtra przez relację jest niemal zawsze tańsze niż iterowanie po samej tabeli faktów.

### Pułapka 3 — zagnieżdżone iteratory mnożą złożoność

`SUMX` wewnątrz `SUMX`, albo `FILTER` wewnątrz `FILTER` na dużych tabelach, oznacza pracę rzędu iloczynu liczby wierszy obu poziomów, nie sumy. Warto przejrzeć, czy zagnieżdżenie da się rozbić na osobne `VAR`, policzone raz i połączone na końcu, zamiast liczyć wewnętrzny iterator od nowa dla każdego wiersza zewnętrznego.

---

## Podsumowanie

| Sytuacja | Zalecane podejście |
|---|---|
| Prosta suma jednej kolumny | `SUM`, nie `SUMX` |
| Obliczenie per wiersz przed sumowaniem (`Ilość×Cena`) | `SUMX` — tu jest konieczne |
| Filtr na jednej kolumnie, prosty warunek | Boolowski filtr wprost w `CALCULATE`, nie `FILTER` |
| Filtr złożony (wiele kolumn / agregacja per grupa) | `FILTER` — tu jest konieczny |
| Powtórzone podwyrażenie w tej samej mierze | `VAR` |
| Filtrowanie dużej tabeli faktów po atrybucie wymiaru | Filtruj tabelę WYMIARU, nie faktów |
| Diagnoza kosztu miary | DAX Studio → Server Timings, proporcja SE/FE |

**Wniosek:** najbardziej uniwersalna zasada to \"zepchnij jak najwięcej do storage engine\" — proste agregatory i bezpośrednie filtry boolowskie w `CALCULATE` są tanie; iteratory, `FILTER`, i wywołania miar wewnątrz iteratorów są potrzebne, ale kosztowne i warte świadomego, a nie automatycznego użycia.
