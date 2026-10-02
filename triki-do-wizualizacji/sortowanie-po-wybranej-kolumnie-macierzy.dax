Użytkownik wybiera miesiąc na slicerze, a macierz sortuje wiersze według wartości miary `[Sprzedaż]` dla wybranego miesiąca.

### 1. Utwórz odłączoną tabelę kalendarza do slicera
```DAX
dim_kalendarz_sort =
VALUES ( 'dim kalendarz'[rok/mc] )

### 2. Dodajemy kolumnę z nowo utworzonej tabeli do slicera - najlepiej w formie listy rozwijalnej, włączamy wybór jednokrotny

### 3. Dodajemy do naszej wizualizzacji macierzy miarę
```DAX
Sprzedaż (z sortowaniem) =
VAR _mc_sortowanie =
    VALUES ( dim_kalendarz_sort[rok/mc] )

RETURN
    IF (
        HASONEVALUE ( 'dim kalendarz'[rok/mc] ),
        [Sprzedaż],
        CALCULATE (
            [Sprzedaż],
            TREATAS (
                _mc_sortowanie,
                'dim kalendarz'[rok/mc]
            )
        )
    )

### 4. Ustawiamy na macierzy sortowanie po kolumnie sumy - ukrywamy kolumnę z sumą bo pokazuje ona teraz tylko i wyłacznie wartość dla wybranego miesiąca co nie jest prawdą.
### 5. Możemy wykorzystać formatowanie warunkowe, aby pokolorować kolumnę, po której aktualnie sortujemy.
```DAX
Sprzedaż (z sortowaniem) format =
VAR _mc_w_kolumnie =
    SELECTEDVALUE ( dim_Kalendarz[rok/mc] )
VAR _mc_sortowanie =
    SELECTEDVALUE ( dim_Kalendarz_sort[rok/mc] )
RETURN
    IF (
        NOT ISBLANK ( _mc_w_kolumnie ) && _mc_w_kolumnie = _mc_sortowanie,
        1,
        BLANK () // w argumencie można podać od razu kolor zamiast "1"
    )
