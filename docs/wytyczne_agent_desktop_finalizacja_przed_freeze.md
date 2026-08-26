# Finalizacja systemu desktopowego przed freeze
## Wytyczne dla agenta — ALPR desktop / MZ / import badań mobilnych

**Repozytorium:** `rszuder/auto_annotation_tool`  
**Punkt odniesienia:** commit `33b7b07558a1d34edfa5a558d2a6b1b2873b553f`  
**Cel:** domknąć system desktopowy przed właściwą kampanią badawczą i następnie zamrozić implementację.

---

# 1. Zasada nadrzędna

Nie rozwijamy już aplikacji „w szerz”.

Agent ma:

1. usunąć zidentyfikowane ryzyka metodologiczne,
2. domknąć analizę balansu klas MZ,
3. domknąć kontrolowane uzupełnianie `train`,
4. upewnić się, że import Android → Desktop nie ogranicza analizy badawczej do preview,
5. dodać testy regresyjne dla krytycznych kontraktów,
6. przygotować wersję do freeze.

Nie dodawać funkcji niezwiązanych bezpośrednio z badaniami.

---

# 2. Stan obecny — czego NIE przebudowywać

W aktualnym systemie są już poprawnie zaimplementowane:

- stały alfabet MZ:
  `0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ`,
- analiza rozkładu klas `train / val / test / total`,
- liczba wystąpień każdej klasy,
- `unique_*_plate_count`,
- rozróżnienie `LOW`, `CRITICAL`, `LOW_DIVERSITY`,
- uwzględnianie `augmentation_manifest.json`, aby augmentowane kopie nie zwiększały sztucznie liczby unikalnych źródeł,
- obsługa datasetów:
  - `images/train + labels/train`,
  - `train/images + train/labels`,
  - płaski `labels/`,
- eksport CSV/JSON,
- UI: tabela + histogram + podsumowanie,
- analiza w osobnym wątku,
- cache analizy,
- parser błędnych linii i błędnych `class_id`,
- importer raportów mobilnych,
- `ExperimentSessionRecord`,
- `series_id`,
- `scenario_id`,
- `experiment_variant`,
- `replicate_index`,
- SHA aplikacji,
- urządzenie,
- runtime/delegate,
- fingerprinty modeli,
- rozdzielczość,
- profil rozpoznawania,
- autozoom config,
- `thermal.csv`,
- `frame_flow.csv`,
- `events.csv/jsonl`,
- `samples/index.csv`,
- guard porównywalności,
- deduplikacja przez hash źródła,
- import wielu raportów z jednego JSON-a.

Nie przepisywać tych elementów od nowa.

---

# 3. P0 — walidacja mapy klas `data.yaml`

## Problem

Analizator interpretuje:

```text
0  -> 0
1  -> 1
...
9  -> 9
10 -> A
...
35 -> Z
```

To jest poprawne tylko wtedy, gdy aktywny dataset posiada dokładnie taką samą mapę klas.

Jeżeli użytkownik wybierze dataset z inną kolejnością `names`, histogram będzie liczony poprawnie po `class_id`, ale błędnie opisany symbolami.

## Wymaganie

Przed analizą należy odczytać `data.yaml`.

Obsłużyć typowe warianty:

```yaml
names:
  0: "0"
  1: "1"
```

oraz:

```yaml
names:
  - "0"
  - "1"
```

Oczekiwana mapa:

```text
0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ
```

## Zachowanie

### Zgodność pełna

Analiza uruchamia się normalnie.

### Brak `data.yaml`

Dopuszczalne tylko dla jawnie niesplitowanego surowego źródła, ale UI ma pokazać:

```text
WARNING: brak mapy klas — interpretacja oparta na standardowym alfabecie MZ.
```

### `data.yaml` istnieje, ale mapa jest inna

Analiza nie może udawać poprawnej interpretacji.

Preferowane zachowanie:

```text
ERROR: mapa klas datasetu nie jest zgodna ze standardem MZ.
```

Pokazać:

- oczekiwaną mapę,
- wykrytą mapę,
- pierwszy niezgodny indeks.

Nie należy automatycznie przemianowywać klas.

## Testy

Dodać co najmniej:

1. lista `names`,
2. słownik `names`,
3. poprawne 36 klas,
4. zamienione `A` i `B`,
5. brak jednej klasy,
6. dodatkowa klasa 36,
7. brak `data.yaml`.

---

# 4. P0 — rozdzielić balans liczebności od różnorodności

Dla każdej klasy muszą być równolegle widoczne:

```text
train_count
unique_train_plate_count
count_status
diversity_status
```

UI nie może prezentować jednej liczby jako pełnej informacji o balansie.

Przykład:

```text
Q:
train_count = 180
unique_train_plate_count = 12
```

jest inną sytuacją niż:

```text
X:
train_count = 180
unique_train_plate_count = 155
```

Pierwsza wymaga nowych źródeł.
Druga może zostać rozsądnie uzupełniona augmentacją.

---

# 5. P0 — kontrolowany mechanizm uzupełniania materiału MZ

## 5.1. Nie modyfikować materiału bazowego

Rozdzielić pojęcia:

```text
BASE DATASET
```

i:

```text
TRAINING VARIANT
```

Materiał bazowy pozostaje niezmieniony.

Wariant treningowy może zawierać:

```text
real
augmented_real
synthetic
augmented_synthetic
```

---

# 6. P0 — kolejność uzupełniania

Obowiązuje hierarchia:

```text
analiza bazowego train
        ↓
[1] dodatkowe realne źródła
        ↓
analiza ponownie
        ↓
[2] celowana augmentacja realnych tablic
        ↓
analiza ponownie
        ↓
[3] LOW/CRITICAL + LOW_DIVERSITY nadal występuje
        ↓
opcjonalnie syntetyczne tablice
```

Agent nie może od razu przechodzić do syntetyków.

---

# 7. P0 — wyznaczenie deficytu klas

Nie wyrównywać wszystkich klas do maksimum.

Domyślny operacyjny target:

```text
target_count = 0.50 * median_nonzero_class_count
```

Parametr powinien być widoczny i możliwy do nadpisania przez użytkownika.

Dla klasy:

```text
deficit(symbol) = max(0, target_count - train_count(symbol))
```

Wynik ma charakter pomocniczy.

Nie przedstawiać tego progu jako uniwersalnej definicji „dobrego balansu”.

---

# 8. P0 — wyszukiwanie dodatkowych realnych źródeł

Jeżeli klasa ma status:

```text
LOW
CRITICAL
LOW_DIVERSITY
LOW+LOW_DIVERSITY
```

system powinien spróbować znaleźć inne dostępne realne tablice.

Preferowane pola wyszukiwania:

```text
source_expected_text
expected_text
plate_text
ground_truth
```

oraz inne istniejące metadane opisujące tekst tablicy.

## Wynik

Dla każdej deficytowej klasy pokazać:

```text
symbol
current_count
unique_sources
available_unused_real_sources
```

Nie dołączać nic automatycznie.

Użytkownik zatwierdza wykorzystanie dodatkowych realnych źródeł.

---

# 9. P0 — celowana augmentacja realnych tablic

Jeżeli klasa:

- ma niedobór liczebności,
- ale ma sensowną liczbę różnych realnych źródeł,

należy wykorzystać istniejący mechanizm `train-only augmentation`.

Nie wybierać próbek całkowicie losowo.

## Priorytet tablicy

Minimalna implementacja:

```text
plate_priority =
sum(deficit(symbol) for symbol in unique_symbols_on_plate)
```

Można zastosować równoważny prosty algorytm.

Zasada:

> tablice zawierające bardziej deficytowe znaki są częściej wybierane do augmentacji.

## Bardzo ważne

Augmentujemy:

```text
całą tablicę + wszystkie bounding boxy znaków
```

a nie pojedynczy znak.

---

# 10. P0 — limit augmentacji jednego źródła

Żeby nie uzyskać:

```text
12 tablic Q
→ 1500 kopii Q
```

należy dodać limit generacji z jednego realnego źródła.

Przykładowy parametr:

```text
max_augmented_variants_per_source
```

Wartość ma być jawna w manifeście.

Nie ustalać jej na sztywno w logice bez możliwości konfiguracji.

---

# 11. P0 — manifest wariantu treningowego

Każdy utworzony wariant treningowy ma mieć manifest.

Minimalne pola:

```json
{
  "schema": "alpr.mz_training_variant.v1",
  "created_at": "...",
  "base_dataset": "...",
  "alphabet": "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ",
  "target_count": 0,
  "selection_policy": "deficit_weighted",
  "sources": {
    "real": 0,
    "augmented_real": 0,
    "synthetic": 0,
    "augmented_synthetic": 0
  },
  "before_distribution": "...",
  "after_distribution": "...",
  "augmentation": {
    "max_variants_per_source": 0
  }
}
```

Jeżeli istniejące manifesty augmentacji można rozszerzyć bez łamania kompatybilności, preferować rozszerzenie istniejącego kontraktu zamiast tworzenia zbędnej równoległej struktury.

---

# 12. P0 — artefakty `before` i `after`

Przed uzupełnieniem:

```text
character_class_distribution_before.csv
character_class_distribution_before.json
```

Po uzupełnieniu:

```text
character_class_distribution_after.csv
character_class_distribution_after.json
```

Nie nadpisywać `before`.

Te pliki będą materiałem do pracy inżynierskiej.

---

# 13. P1 tylko jeśli realnie potrzebne — syntetyczne tablice

Nie budować generatora syntetycznego automatycznie.

Uruchomić ten etap tylko wtedy, gdy po:

1. dodatkowych realnych źródłach,
2. celowanej augmentacji,

pozostaje:

```text
LOW / CRITICAL
+
LOW_DIVERSITY
```

Jeżeli syntetyki są potrzebne:

- generować całe tablice,
- stosować poprawny alfabet i format,
- zapisywać automatyczne bounding boxy znaków,
- rozdzielać `synthetic` i `augmented_synthetic`,
- używać tylko w `train`.

`val` i `test` pozostają nietknięte.

---

# 14. P0 — pełne dane analityczne raportu mobilnego nie mogą być ograniczone przez preview

## Stan obecny

UI posiada limit preview, np. 5000 rekordów.

To jest poprawne dla responsywności GUI.

Nie może jednak oznaczać, że właściwa analiza badawcza korzysta tylko z pierwszych 5000 rekordów.

## Wymaganie

Rozdzielić:

```text
preview source
```

od:

```text
analysis source
```

### Preview

Może pozostać ograniczony.

### Analiza / eksport danych do pracy

Musi korzystać z pełnego źródła.

Dopuszczalne rozwiązania:

- ponowne strumieniowe czytanie CSV z archiwum na żądanie,
- lazy iterator,
- funkcja `read_full_*`,
- osobny backend analizujący cały wpis archiwum.

Nie ma potrzeby ładowania wszystkiego naraz do RAM.

## Dotyczy co najmniej

```text
traces.csv
thermal.csv
frame_flow.csv
events.csv/jsonl
samples/index.csv
```

---

# 15. P0 — API do pełnych danych

Dodać backendowe funkcje niezależne od UI, np.:

```python
iter_full_trace_rows(bundle_or_path)
iter_full_thermal_rows(bundle_or_path)
iter_full_frame_flow_rows(bundle_or_path)
iter_full_event_rows(bundle_or_path)
iter_full_sample_rows(bundle_or_path)
```

lub równoważny interfejs.

Funkcje mają:

- czytać bezpiecznie,
- nie rozpakowywać archiwum do losowych katalogów,
- respektować kontrolę ścieżek ZIP,
- umożliwiać obliczenia na wszystkich rekordach.

---

# 16. P0 — test pełne źródło vs preview

Test syntetyczny:

```text
trace_total = 12000
preview = 5000
```

Oczekiwanie:

```text
bundle.trace_total == 12000
len(bundle.trace_rows) == 5000
full_analysis_count == 12000
```

Analogiczne testy dla:

```text
thermal
frame_flow
events
samples
```

---

# 17. P0 — spójność eksperymentów Android → Desktop

Guard porównywalności ma pozostać nieblokujący, ale przed eksportem/porównaniem serii należy wyraźnie sygnalizować różnice w:

```text
device
android version
app git SHA
model fingerprints
runtime
delegate
resolution
recognition profile
scenario_id
series_id
```

Jeśli użytkownik porównuje raporty z różnym `scenario_id`, UI powinien pokazać ostrzeżenie.

---

# 18. P0 — ground truth

Brak GT nie może zostać potraktowany jako:

```text
accuracy = 0
```

Ma być:

```text
quality unavailable
```

Exact match i CER liczyć tylko dla próbek posiadających ground truth.

Confidence nie jest miarą poprawności.

Nie zmieniać tej zasady.

---

# 19. P0 — testy wymagane przed freeze

Uruchomić istniejące testy oraz dodać brakujące.

Minimalna lista:

## Balans MZ

- 36 klas,
- poprawne zliczenie `train/val/test`,
- płaski dataset,
- jedna klasa zerowa,
- klasa `LOW`,
- klasa `LOW_DIVERSITY`,
- wiele wystąpień w jednej tablicy = jedno unikalne źródło,
- augmentowana kopia = to samo źródło,
- błędna linia YOLO,
- `class_id = 36`,
- zgodny `data.yaml`,
- niezgodny `data.yaml`.

## Uzupełnianie

- wyliczenie `target_count`,
- wyliczenie deficytu,
- priorytet tablicy z rzadkim znakiem,
- limit augmentacji jednego źródła,
- `val` i `test` nie są modyfikowane,
- manifest `before/after`.

## Mobile import

- pojedynczy `.alprsession`,
- JSON z jednym raportem,
- JSON z wieloma raportami,
- deduplikacja identycznego pliku,
- dwie repliki tej samej konfiguracji pozostają osobnymi raportami,
- `thermal.csv`,
- `frame_flow.csv`,
- `events.jsonl`,
- pełne dane > limit preview,
- guard porównywalności.

---

# 20. Smoke test użytkownika przed freeze

Agent ma przygotować checklistę ręcznego testu:

1. uruchomić aplikację,
2. wejść w tor MZ,
3. wybrać realny dataset,
4. uruchomić analizę klas,
5. sprawdzić mapę `data.yaml`,
6. zapisać `before`,
7. wykonać minimalne uzupełnienie,
8. zapisać `after`,
9. upewnić się, że `val/test` się nie zmieniły,
10. zaimportować jeden raport Android,
11. zaimportować serię kilku raportów,
12. sprawdzić `series/scenario/replicate`,
13. sprawdzić artefakty,
14. sprawdzić guard,
15. odczytać pełną liczbę rekordów z raportu większego niż preview.

---

# 21. Kryteria freeze

Desktop jest gotowy do zamrożenia, gdy:

- [ ] mapa klas datasetu jest walidowana,
- [ ] 36 klas jest analizowanych poprawnie,
- [ ] unikalne źródła są liczone poprawnie,
- [ ] augmentacje nie zwiększają sztucznie różnorodności,
- [ ] istnieje kontrolowane uzupełnianie realnych danych `train`,
- [ ] uzupełnianie jest sterowane deficytem klas,
- [ ] istnieje limit augmentacji pojedynczego źródła,
- [ ] `val/test` pozostają niezmienione,
- [ ] są artefakty `before/after`,
- [ ] jest manifest wariantu treningowego,
- [ ] import Android obsługuje serię i repliki,
- [ ] pełne dane raportu są dostępne niezależnie od preview,
- [ ] guard porównywalności działa,
- [ ] wszystkie testy przechodzą,
- [ ] smoke test GUI przeszedł.

---

# 22. Po freeze

Po spełnieniu kryteriów:

1. wykonać commit,
2. oznaczyć SHA,
3. opcjonalnie utworzyć tag, np.:

```text
thesis-experiment-freeze
```

4. nie zmieniać pipeline'u, kontraktów raportu, datasetu ani algorytmów w trakcie jednej kampanii eksperymentalnej.

Jeżeli później pojawi się krytyczny bug:

```text
fix
→ nowy commit
→ nowy freeze SHA
→ jawnie nowa seria eksperymentów
```

---

# 23. Poza zakresem

Nie implementować teraz:

- nowego dużego dashboardu,
- nowych systemów bazodanowych,
- pełnego automatycznego AutoML,
- nowych architektur modeli,
- automatycznego generatora syntetyków, jeśli realne dane są wystarczające,
- nowych strategii trackingu po stronie desktopu,
- zmian kosmetycznych niezwiązanych z badaniami,
- przepisywania działającego importera.

---

# 24. Priorytet wykonania

Kolejność:

```text
1. walidacja data.yaml
2. domknięcie targeted balancing MZ
3. before/after + manifest
4. full analysis source niezależny od preview
5. testy
6. smoke test
7. freeze
8. badania
```

Najważniejsza zasada:

> Po wykonaniu tej listy system ma być traktowany jako skończone stanowisko badawcze, a nie projekt do dalszego rozwoju.
