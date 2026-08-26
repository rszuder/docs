# Wytyczne dla agenta desktopowego — analiza balansu znaków MZ

## 1. Cel

Przed zamrożeniem aplikacji desktopowej należy dodać **minimalny moduł diagnostyczny rozkładu klas znaków dla datasetu MZ**.

Celem nie jest budowa automatycznego systemu równoważenia zbioru, lecz:

- policzenie rzeczywistej liczby wystąpień klas `0–9` i `A–Z`,
- pokazanie rozkładu osobno dla `train`, `val`, `test`,
- wykrycie klas zerowych i silnie niedoreprezentowanych,
- umożliwienie szybkiej decyzji, czy zbiór treningowy wymaga uzupełnienia,
- wygenerowanie tabeli i wykresu możliwych do wykorzystania w pracy inżynierskiej.

Moduł ma być mały, bezpieczny i możliwy do wdrożenia w kilka godzin.

## 2. Uzasadnienie badawcze

Rozkład klas znaków ma bezpośredni wpływ na trening modelu MZ i może tłumaczyć część charakterystycznych pomyłek znaków o podobnym kształcie, np. `B ↔ 8`, `S ↔ 5`, `Z ↔ 7`.

W analizowanej pracy referencyjnej zauważono problem dominacji niektórych znaków wynikający z lokalnego charakteru danych oraz uzupełniano rzadkie klasy materiałem syntetycznym. W naszym systemie należy zrobić to bardziej kontrolowanie:

1. najpierw zmierzyć rozkład klas,
2. dopiero potem zdecydować, czy konieczna jest korekta,
3. korektę wykonywać przede wszystkim dla `train`,
4. `val` i `test` pozostawić możliwie naturalne i reprezentatywne.

Nie należy dążyć automatycznie do idealnie równej liczby próbek dla wszystkich 36 klas.

## 3. Stan obecny i miejsce integracji

W Z4 istnieją już elementy związane z datasetem znaków:

- obsługa datasetu YOLO `images/labels`,
- rozpoznawanie splitów `train/val/test`,
- liczenie par obraz–etykieta,
- liczenie łącznej liczby etykiet znaków,
- prezentowanie bilansu typu „liczba tablic / liczba znaków”,
- istniejący przepływ augmentacji i budowy datasetu znaków.

Brakuje natomiast **rozkładu per klasa** oraz informacji, z ilu **unikalnych tablic źródłowych** pochodzi dana klasa.

Ważne rozróżnienie dotyczące obecnego kodu:

- `PlateGenerator` wycina, prostuje i normalizuje **istniejące tablice**; nie generuje nowych syntetycznych numerów,
- obecny moduł augmentacji tworzy dodatkowe próbki **wyłącznie ze splitu `train`** i transformuje obraz razem z etykietami YOLO,
- dlatego przed decyzją o uzupełnianiu klas trzeba odróżnić:
  1. **niedobór liczby wystąpień**, który można częściowo skorygować celowaną augmentacją,
  2. **niedobór różnorodności materiału bazowego**, którego sama augmentacja nie naprawi.

Nowy moduł powinien wykorzystać istniejące źródło datasetu znaków, bez budowania osobnego systemu importu.

## 4. Zakres P0 — funkcjonalność obowiązkowa

### 4.1. Analiza klas

Dla wybranego datasetu YOLO należy przeskanować wszystkie pliki etykiet `.txt`.

Pierwsza wartość każdego poprawnego wiersza YOLO jest `class_id`.

Należy policzyć wystąpienia dla 36 klas w ustalonej kolejności:

```text
0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ
```

Wyniki liczyć osobno dla:

- `train`,
- `val`,
- `test`,
- `total`.

Dla każdego znaku przechowywać co najmniej:

```text
class_id
symbol
train_count
val_count
test_count
total_count
train_share
unique_train_plate_count
unique_val_plate_count
unique_test_plate_count
unique_total_plate_count
```

`train_share` oznacza udział danej klasy w całkowitej liczbie etykiet znaków zbioru treningowego.

`unique_*_plate_count` oznacza liczbę **różnych obrazów/tablic**, w których występuje dana klasa. Parametr ten jest obowiązkowy, ponieważ pozwala odróżnić sytuację, w której klasa ma mało przykładów, ale dużą różnorodność źródeł, od sytuacji, w której setki próbek pochodzą z kilku powielanych tablic.

Przykład:

```text
Q: 180 wystąpień w 37 różnych tablicach
X: 420 wystąpień w 290 różnych tablicach
```

W obu przypadkach histogram jest niski, ale drugi przypadek znacznie lepiej nadaje się do celowanej augmentacji.

### 4.2. Obsługiwane układy datasetu

Analiza powinna działać dla układów już obsługiwanych przez aplikację, w szczególności:

```text
images/train
labels/train
images/val
labels/val
images/test
labels/test
```

oraz:

```text
train/images
train/labels
val/images
val/labels
test/images
test/labels
```

Jeśli dostępny jest wyłącznie układ płaski:

```text
images/
labels/
```

należy pokazać wynik jako `unsplit/total`, bez udawania, że istnieją `train/val/test`.

## 5. Diagnostyka balansu

Moduł powinien obliczać następujące wartości zbiorcze dla `train`:

```text
total_labels
minimum_class_count
maximum_class_count
median_class_count
max_min_ratio
zero_classes
low_count_classes
minimum_unique_plate_count
median_unique_plate_count
low_diversity_classes
```

### 5.1. Klasy zerowe

Każda klasa z liczbą `0` w `train` powinna być oznaczona jako:

```text
CRITICAL
```

Brak klasy w `val` lub `test` należy pokazać jako informację/warning, ale nie wolno automatycznie „naprawiać” splitu.

### 5.2. Klasy słabo reprezentowane

Nie należy kodować jednego arbitralnego progu jako naukowej definicji niezbalansowanego datasetu.

Do diagnostyki UI można zastosować prostą regułę pomocniczą, np.:

```text
count < 0.25 * median_nonzero_count
```

Klasa spełniająca warunek jest oznaczana jako:

```text
LOW
```

Próg powinien być traktowany wyłącznie jako **wskaźnik pomocniczy**, a nie automatyczny warunek przebudowy zbioru.

W UI należy jasno opisać, że naturalny rozkład znaków nie musi być jednostajny.

### 5.3. Różnorodność materiału bazowego

Dla każdej klasy należy niezależnie raportować liczbę unikalnych tablic źródłowych.

Do prostej diagnostyki można zastosować pomocniczą regułę:

```text
unique_train_plate_count < 0.25 * median_nonzero_unique_plate_count
```

i status:

```text
LOW_DIVERSITY
```

Reguła ma charakter wyłącznie diagnostyczny.

Nie wolno utożsamiać dużej liczby przykładów po augmentacji z dużą różnorodnością danych bazowych.

## 6. Interfejs użytkownika — wersja minimalna

Nie budować nowej dużej zakładki.

Preferowane miejsce: istniejący obszar Z4 związany z datasetem/treningiem MZ.

Dodać przycisk:

```text
Analizuj rozkład klas
```

Po analizie pokazać:

### 6.1. Tabela

| Znak | Train | Unikalne tablice train | Val | Test | Razem | Udział train | Status |
|---|---:|---:|---:|---:|---:|---:|---|
| 0 | ... | ... | ... | ... | ... | ... | OK |
| 1 | ... | ... | ... | ... | ... | ... | LOW |
| ... | ... | ... | ... | ... | ... | ... | ... |
| Z | ... | ... | ... | ... | ... | ... | OK |

Statusy:

```text
OK
LOW
CRITICAL
```

### 6.2. Podsumowanie

Pokazać minimum:

```text
Liczba etykiet train:
Mediana na klasę:
Minimum:
Maksimum:
Max/min:
Mediana liczby unikalnych tablic na klasę:
Minimum unikalnych tablic:
Klasy bez przykładów:
Klasy słabo reprezentowane:
Klasy o małej różnorodności źródeł:
```

### 6.3. Wykres

Wykres słupkowy:

```text
X: 0–9 A–Z
Y: liczba wystąpień w train
```

Wykres ma być prosty i czytelny.

Nie implementować obecnie rozbudowanej interaktywności, filtrowania, zoomowania ani wielu wariantów wizualizacji.

## 7. Eksport wyników analizy

Analiza musi być możliwa do wykorzystania poza GUI.

Dodać możliwość zapisania co najmniej:

### 7.1. CSV

Przykład:

```text
character_class_distribution.csv
```

Kolumny:

```text
class_id
symbol
train_count
val_count
test_count
total_count
train_share
unique_train_plate_count
unique_val_plate_count
unique_test_plate_count
unique_total_plate_count
count_status
diversity_status
status
```

### 7.2. JSON

Przykład:

```text
character_class_distribution.json
```

Powinien zawierać co najmniej:

```json
{
  "schema": "alpr.character_class_distribution.v1",
  "dataset_path": "...",
  "generated_at": "...",
  "alphabet": "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ",
  "summary": {},
  "classes": []
}
```

### 7.3. Obraz wykresu

Preferowany format:

```text
character_class_distribution_train.png
```

Jeśli obecny system eksportu wykresów nie obsługuje tego bez większych zmian, PNG można odłożyć do P1. CSV i JSON są P0.

## 8. Zasady dotyczące korekty datasetu i uzupełniania materiału bazowego

### 8.1. Materiał bazowy i wariant treningowy to nie to samo

Nie należy modyfikować materiału bazowego tylko po to, aby histogram wyglądał równo.

Należy rozdzielić:

```text
MATERIAŁ BAZOWY
= realne, zebrane i zweryfikowane tablice

WARIANT TRENINGOWY
= materiał bazowy
+ celowana augmentacja realnych tablic
+ ewentualnie syntetyczne tablice, jeśli realnych źródeł jest zbyt mało
```

Dzięki temu można udokumentować:

- naturalny rozkład danych,
- klasy wymagające uzupełnienia,
- sposób uzupełnienia,
- końcowy skład zbioru treningowego.

### 8.2. Nie wyrównywać wszystkich klas do maksimum

Nie należy stosować:

```text
target_count = maximum_class_count
```

Preferowana pragmatyczna reguła robocza:

```text
target_count = 0.50 * median_nonzero_class_count
```

albo inna jawnie zapisana wartość ustalona po analizie rzeczywistego histogramu.

Deficyt klasy:

```text
deficit(symbol) = max(0, target_count - train_count(symbol))
```

Próg jest operacyjny, nie stanowi uniwersalnej naukowej definicji „dobrego balansu”.

### 8.3. Poziom 1 — dodatkowe realne tablice z istniejącej puli

To jest **preferowane źródło uzupełnienia**.

Dla klas `LOW`, `CRITICAL` lub `LOW_DIVERSITY` należy w pierwszej kolejności wyszukać dodatkowe realne tablice w istniejącej puli materiału źródłowego.

Jeśli metadane to umożliwiają, wykorzystać m.in.:

```text
source_expected_text
expected_text
plate_text
```

Przepływ:

```text
analiza histogramu
→ wskazanie słabych klas
→ przeszukanie istniejącej puli realnych tablic
→ dołączenie wcześniej niewykorzystanych poprawnych przykładów
→ ponowna analiza
```

Nie kopiować tej samej tablicy wielokrotnie jako „nowych źródeł”.

### 8.4. Poziom 2 — celowana augmentacja realnych tablic

Jeżeli klasa ma niską liczebność, ale występuje w wystarczająco wielu różnych tablicach, można wykorzystać istniejący mechanizm `train-only augmentation`.

Tablic do augmentacji nie należy wybierać całkowicie losowo.

Można nadać tablicy priorytet:

```text
plate_priority =
sum(deficit(symbol) for symbol in unique_symbols_on_plate)
```

Przykład:

```text
WX12345
```

może mieć wysoki priorytet nie dlatego, że zawiera częste `W`, lecz dlatego, że zawiera deficytowe `X`.

Dopuszczalne jest uproszczenie algorytmu, jeśli lepiej pasuje do aktualnej architektury Z4, ale zasada pozostaje:

> im większy deficyt klas obecnych na tablicy, tym częściej dana realna tablica może być wybierana do augmentacji.

Augmentujemy **całą tablicę wraz z etykietami**, nie pojedynczy znak.

### 8.5. Kiedy augmentacja nie wystarcza

Przykład:

```text
Q:
180 wystąpień
12 unikalnych tablic
```

Wygenerowanie 1300 wariantów z 12 obrazów poprawi histogram, ale nie zapewni odpowiedniej różnorodności.

Jeżeli występuje jednocześnie:

```text
LOW / CRITICAL
+
LOW_DIVERSITY
```

należy najpierw szukać nowych realnych źródeł.

### 8.6. Poziom 3 — syntetyczne tablice bazowe

Dopiero jeśli:

1. klasa jest wyraźnie niedoreprezentowana,
2. ma mało unikalnych źródeł,
3. nie udało się znaleźć wystarczającego materiału realnego,

można rozważyć syntetyczne tablice.

Obecny `PlateGenerator` nie jest generatorem nowych numerów — wycina i normalizuje istniejące tablice. Nie należy przebudowywać go w pośpiechu, jeśli analiza nie wykaże takiej potrzeby.

Jeżeli generator syntetycznych tablic okaże się konieczny, powinien tworzyć **całe tablice zawierające deficytowe znaki**, a nie izolowane glify.

Przykładowo dla deficytowego `X`:

```text
WX48217
DX7245
WX9A321
```

Syntetyczne tablice powinny mieć automatycznie poprawne bounding boxy znaków, a następnie mogą być poddawane istniejącej augmentacji `train-only`.

### 8.7. Hierarchia uzupełniania

Obowiązuje kolejność:

```text
MATERIAŁ BAZOWY
      ↓
analiza 36 klas + unique_plate_count_per_class
      ↓
[1] dodatkowe realne źródła
      ↓
analiza ponownie
      ↓
[2] celowana augmentacja realnych tablic
      ↓
analiza ponownie
      ↓
[3] nadal LOW/CRITICAL + LOW_DIVERSITY?
      ↓
tak → rozważyć syntetyczne tablice bazowe
      ↓
augmentacja syntetyków
```

### 8.8. Train vs val/test

- `train` może być świadomie uzupełniany,
- `val` i `test` nie powinny być sztucznie równoważone tylko po to, aby poprawić histogram.

Walidacja i test mają zachować możliwie realistyczny rozkład danych.

### 8.9. Pochodzenie próbek

Jeśli obecna architektura na to pozwala, należy zachować logiczne pochodzenie próbek:

```text
real
augmented_real
synthetic
augmented_synthetic
```

Nie trzeba zmieniać formatu etykiet YOLO. Informację można przechować w manifeście wariantu datasetu albo pomocniczym JSON/CSV.

### 8.10. Czego nie robić automatycznie w P0

Nie implementować systemu, który bez decyzji użytkownika:

- generuje syntetyczne tablice,
- oversampluje wszystkie klasy do tej samej liczby,
- przebudowuje `val` i `test`,
- usuwa przykłady klas częstych,
- modyfikuje materiał bazowy,
- uruchamia augmentację tylko dlatego, że klasa ma status `LOW`.

P0 ma przede wszystkim **zmierzyć problem oraz przygotować kontrolowany wybór realnych źródeł do augmentacji**.

## 9. Zastosowanie w pracy inżynierskiej

Po wdrożeniu modułu należy zachować wyniki **przed ewentualną korektą** i **po korekcie**.

Potencjalne zestawienie do pracy:

```text
Rozkład klas znaków w zbiorze treningowym MZ przed i po uzupełnieniu danych
```

Można wykorzystać:

- tabelę 36 klas,
- wykres słupkowy,
- listę klas o najmniejszej liczbie przykładów,
- liczbę unikalnych tablic źródłowych dla klas,
- opis sposobu uzupełnienia danych,
- zestawienie źródeł wariantu treningowego.

Przydatna tabela do pracy:

| Typ danych MZ | Liczba tablic | Liczba znaków |
|---|---:|---:|
| realne | ... | ... |
| augmentowane realne | ... | ... |
| syntetyczne | ... | ... |
| augmentowane syntetyczne | ... | ... |
| razem | ... | ... |

Następnie przy analizie błędów MZ można zestawić:

```text
częstość klasy w train
vs
częstość pomyłek tej klasy
```

Nie należy jednak zakładać z góry związku przyczynowego. Jeśli dane nie pokażą zależności, należy to uczciwie opisać.

## 10. Plan wdrożenia dla agenta desktopowego

### Krok 1 — lokalizacja źródła danych

Wykorzystać istniejącą logikę rozpoznawania datasetu znaków w Z4.

Nie dodawać nowego pickera datasetu, jeśli aktualny kontekst MZ już wskazuje poprawny katalog.

### Krok 2 — funkcja analizy

Dodać funkcję niezależną od UI, np.:

```python
analyze_character_class_distribution(dataset_root: Path) -> CharacterClassDistribution
```

Funkcja nie może zależeć od widgetów Tkinter.

Powinna zwracać kompletny model danych analizy.

### Krok 3 — model danych

Preferowane proste struktury:

```python
@dataclass
class CharacterClassCount:
    class_id: int
    symbol: str
    train_count: int
    val_count: int
    test_count: int
    total_count: int
    train_share: float
    unique_train_plate_count: int
    unique_val_plate_count: int
    unique_test_plate_count: int
    unique_total_plate_count: int
    count_status: str
    diversity_status: str
    status: str
```

oraz:

```python
@dataclass
class CharacterClassDistribution:
    dataset_path: str
    classes: list[CharacterClassCount]
    summary: dict
```

Jeśli repo posiada już ustalony styl modeli danych dla Z4, należy zachować ten styl zamiast tworzyć równoległą architekturę.

### Krok 4 — parser YOLO

Parser musi:

- ignorować puste linie,
- ignorować komentarze, jeśli występują,
- walidować `class_id`,
- odrzucać `class_id < 0`,
- odrzucać `class_id >= 36`,
- zliczać błędne rekordy diagnostycznie,
- nie przerywać całej analizy przez pojedynczy uszkodzony plik.

W podsumowaniu warto zachować:

```text
invalid_label_lines
invalid_class_ids
files_read
```

### Krok 5 — cache

Dataset może zawierać tysiące plików.

Można wykorzystać podobny mechanizm cache jak w istniejącym liczeniu par obraz–etykieta.

Sygnatura cache powinna zależeć co najmniej od:

- ścieżki datasetu,
- mtime katalogów etykiet,
- opcjonalnie liczby/mtime plików.

Nie należy budować skomplikowanej bazy indeksowej.

### Krok 6 — UI

Dodać:

```text
Analizuj rozkład klas
```

i prosty panel wyników.

Analiza większego datasetu nie może blokować UI na długo; jeśli obecny wzorzec Z4 wykorzystuje worker/thread dla cięższych operacji, zastosować ten sam wzorzec.

### Krok 7 — eksport CSV/JSON

Eksport ma wykorzystywać dokładnie te same dane, które pokazuje UI.

Nie liczyć statystyk drugi raz podczas eksportu.

### Krok 8 — testy

Minimum:

1. dataset z równym rozkładem,
2. dataset z jedną brakującą klasą,
3. dataset z klasą silnie niedoreprezentowaną,
4. dataset bez `test`,
5. dataset płaski `images/labels`,
6. uszkodzona linia etykiety,
7. `class_id = 36` lub większy,
8. pusty dataset,
9. wiele wystąpień klasy w jednej tablicy — `unique_plate_count` ma wzrosnąć tylko o 1,
10. ta sama klasa na wielu tablicach — poprawne zliczenie różnorodności,
11. test priorytetu tablicy zawierającej rzadki znak.

### Krok 9 — weryfikacja na realnym zbiorze

Po wdrożeniu uruchomić analizę na aktualnym datasetcie MZ i zapisać:

```text
character_class_distribution_before.csv
character_class_distribution_before.json
```

Wynik musi zawierać również `unique_*_plate_count`.

### Krok 10 — minimalne wsparcie uzupełniania

Jeżeli realny histogram pokaże problem:

1. policzyć deficyty klas względem jawnie ustalonego `target_count`,
2. wyszukać dodatkowe realne tablice zawierające klasy deficytowe,
3. jeśli to nie wystarczy, wyznaczyć priorytet realnych tablic do istniejącej augmentacji `train-only`,
4. zachować manifest sposobu uzupełnienia.

Generator syntetycznych tablic wdrażać tylko wtedy, gdy analiza pokaże jednocześnie `LOW/CRITICAL` i `LOW_DIVERSITY`.

### Krok 11 — analiza po korekcie

Po ewentualnym uzupełnieniu zapisać:

```text
character_class_distribution_after.csv
character_class_distribution_after.json
```

Zachować oba zestawy `before/after`.

### Krok 12 — freeze

Po potwierdzeniu poprawności:

- nie rozwijać dalej modułu,
- nie dodawać kolejnych strategii balansowania,
- oznaczyć wersję aplikacji i wariantu datasetu używanego do treningu,
- przejść do właściwej kampanii eksperymentalnej.

## 11. Kryteria odbioru P0

Funkcjonalność uznajemy za ukończoną, jeśli:

- [ ] aplikacja liczy wszystkie 36 klas,
- [ ] dla każdej klasy liczy liczbę unikalnych tablic źródłowych,
- [ ] wynik jest dostępny osobno dla `train/val/test`,
- [ ] poprawnie obsługiwany jest brak jednego ze splitów,
- [ ] klasy zerowe są widoczne natychmiast,
- [ ] klasy o małej różnorodności źródeł są widoczne niezależnie od liczby wystąpień,
- [ ] dostępna jest tabela w UI,
- [ ] dostępny jest prosty wykres dla `train`,
- [ ] można zapisać CSV,
- [ ] można zapisać JSON,
- [ ] wynik jest zgodny z ręcznym przeliczeniem małego datasetu testowego,
- [ ] pojedyncza błędna linia YOLO nie przerywa całej analizy,
- [ ] sama analiza nie modyfikuje datasetu,
- [ ] po analizie aktualnego zbioru zachowano pliki `before`,
- [ ] jeśli wykonano uzupełnienie, zachowano również `after` i manifest sposobu korekty,
- [ ] celowana augmentacja działa tylko na `train`.

## 12. Poza zakresem przed oddaniem pracy

Nie implementować teraz:

- pełnego automatycznego rebalansowania bez decyzji użytkownika,
- rozbudowanego generatora syntetycznych tablic, chyba że analiza wykaże `LOW/CRITICAL + LOW_DIVERSITY`,
- zaawansowanych rekomendacji augmentacji,
- statystycznego testowania hipotez w GUI,
- macierzy pomyłek zintegrowanej z tym modułem,
- rozbudowanego dashboardu,
- dodatkowego systemu bazodanowego,
- przebudowy istniejącego procesu Z4.

Jeśli te funkcje okażą się interesujące, można wskazać je jako kierunek dalszego rozwoju.

## 13. Priorytet przy 3 dniach do oddania

Kolejność prac:

```text
P0  domknięcie Android → eksport → Desktop/import
P0  analiza rozkładu 36 klas MZ + unique_plate_count_per_class
P0  analiza realnego datasetu i zapis `before`
P0  najpierw dołączenie dostępnych realnych przykładów klas deficytowych
P0  następnie celowana augmentacja realnych tablic tylko w `train`
P0  syntetyki tylko jeśli pozostaje jednocześnie duży deficyt i mała różnorodność bazowa
P0  zapis `after` + manifest wariantu treningowego
P0  freeze
P0  właściwe badania i generowanie wyników do rozdziału 5
```

Najważniejsza zasada:

> Nie dodajemy funkcji dlatego, że mogą być przydatne. Dodajemy wyłącznie to, czego potrzebujemy do wiarygodnego przeprowadzenia badań i zamknięcia pracy.
