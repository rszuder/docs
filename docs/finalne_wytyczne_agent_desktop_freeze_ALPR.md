# FINALNE WYTYCZNE DLA AGENTA DESKTOPOWEGO
## Domknięcie systemu ALPR przed freeze i rozpoczęciem badań

**Repozytorium:** `rszuder/auto_annotation_tool`  
**Gałąź:** `main`  
**Punkt odniesienia:** `184ffa6fbe4795692ab228f32a2f31f4424cb8d1`  
**Cel:** doprowadzić desktop do stanu „skończone stanowisko badawcze”, następnie zamrozić kod i rozpocząć właściwą kampanię eksperymentalną.

---

# 1. Zasada nadrzędna

Agent ma wykonać wyłącznie prace potrzebne do:

1. kompletnej diagnostyki i kontrolowanego uzupełniania zbioru MZ,
2. zachowania odtwarzalności wariantu treningowego,
3. bezpiecznego importu i pełnej analizy danych Android → Desktop,
4. kontroli porównywalności eksperymentów,
5. testów regresyjnych,
6. smoke testu,
7. freeze.

**Nie rozwijamy już funkcjonalności poza tym zakresem.**

Po wykonaniu niniejszego dokumentu aplikacja ma być traktowana jako **ukończone stanowisko badawcze**, nie jako projekt do dalszej rozbudowy.

---

# 2. Stan obecny — czego nie przepisywać

Aktualny `main` posiada już:

## MZ / balans klas

- alfabet:
  `0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ`,
- analiza `train / val / test / total`,
- `train_count`,
- `unique_train_plate_count`,
- `count_status`,
- `diversity_status`,
- `LOW`,
- `CRITICAL`,
- `LOW_DIVERSITY`,
- uwzględnianie `augmentation_manifest.json`,
- walidację `data.yaml`,
- obsługę `names` jako listy i słownika,
- target:
  `target_count = ceil(target_ratio * median_nonzero_class_count)`,
- domyślny `target_ratio = 0.50`,
- `deficit_count`,
- GUI do analizy,
- CSV/JSON,
- `plan_character_train_augmentation(...)`,
- `build_character_training_variant_manifest(...)`.

## Raporty Android

- import `.alprsession`, ZIP i JSON,
- wiele raportów z jednego JSON-a,
- deduplikację po hash źródła,
- repliki eksperymentów,
- `ExperimentSessionRecord`,
- `series_id`,
- `scenario_id`,
- `experiment_variant`,
- `replicate_index`,
- SHA aplikacji,
- urządzenie,
- runtime/delegate,
- rozdzielczość,
- profile,
- fingerprinty modeli w indeksie,
- preview artefaktów,
- pełne iteratory:
  - `iter_full_trace_rows`,
  - `iter_full_thermal_rows`,
  - `iter_full_frame_flow_rows`,
  - `iter_full_event_rows`,
  - `iter_full_sample_rows`,
- guard porównywalności,
- quality z GT,
- exact match / CER,
- test danych > preview,
- checklistę smoke testu.

**Nie przepisywać tych elementów od nowa.**

---

# 3. Priorytety końcowe

Kolejność obowiązkowa:

```text
A. domknięcie MZ planner/executor
B. dodatkowe realne źródła
C. before/after + manifest + hash
D. guard: Android version + model fingerprints
E. testy
F. smoke test
G. freeze
H. badania
```

---

# 4. P0 — deduplikacja planner candidates po realnym źródle

## Problem

`plan_character_train_augmentation(...)` normalizuje źródło przez `source_key`, ale może zwrócić kilka kandydatów reprezentujących to samo realne źródło, jeżeli dataset zawiera już:

```text
plate_001
plate_001_aug1
plate_001_aug2
```

Wszystkie powinny reprezentować jedno źródło bazowe.

## Wymaganie

Planner musi grupować kandydatów po:

```text
source_key
```

i zwracać **maksymalnie jeden kandydat na jedno realne źródło**.

### Reguła wyboru reprezentanta

Jeżeli istnieje oryginalny plik:

```text
plate_001
```

preferować oryginał nad augmentowaną kopią.

Jeżeli oryginału nie ma:

- wybrać deterministycznie jedną reprezentację,
- np. leksykograficznie pierwszą,
- zachować warning w planie.

## Kryterium

Dla:

```text
plate_001
plate_001_aug1
plate_001_aug2
```

plan zawiera:

```text
1 candidate
source_key = plate_001
```

---

# 5. P0 — realny limit augmentacji na jedno źródło

Obecnie plan posiada:

```text
max_augmented_variants_per_source
```

Ten parametr musi być respektowany również przez wykonanie augmentacji.

Nie wystarczy przechowywać go w planie.

## Wymaganie

Executor augmentacji musi gwarantować:

```text
generated_variants(source_key) <= max_augmented_variants_per_source
```

nawet jeżeli:

- to samo źródło zawiera kilka deficytowych znaków,
- planner oblicza wysoki priorytet,
- źródło pojawia się wielokrotnie w starym datasetcie.

---

# 6. P0 — podłączyć planner do istniejącej augmentacji train-only

## Stan

Planner istnieje, ale nie stanowi jeszcze pełnego workflow użytkownika.

## Wymagany proces

```text
Analiza rozkładu
        ↓
Plan uzupełnienia
        ↓
Podgląd planu
        ↓
Zatwierdzenie użytkownika
        ↓
Utworzenie NOWEGO wariantu datasetu
        ↓
Targeted train-only augmentation
        ↓
Analiza after
        ↓
Manifest wariantu
```

## Bardzo ważne

Nie modyfikować aktywnego/base datasetu in-place.

Tworzyć nowy wariant.

---

# 7. P0 — UI do planu augmentacji

Nie budować dużego dashboardu.

W modalnym workflow analizy MZ dodać minimalne akcje:

```text
[Zapisz before]
[Przygotuj plan uzupełnienia]
```

Po przygotowaniu planu pokazać:

| Źródło | Znaki deficytowe | Priorytet | Maks. wariantów |
|---|---|---:|---:|

oraz podsumowanie:

```text
target_count
total_deficit_count
liczba źródeł-kandydatów
max_variants_per_source
szacowana maksymalna liczba nowych próbek
```

Użytkownik zatwierdza wykonanie.

---

# 8. P0 — wyszukiwanie dodatkowych realnych źródeł

## Zasada

Przed augmentacją należy spróbować znaleźć **niewykorzystane realne tablice**.

## Cel

Dla klasy:

```text
Q
```

system ma móc powiedzieć np.:

```text
train:
  12 unikalnych tablic

poza train / w dostępnej puli:
  47 dodatkowych realnych tablic zawierających Q
```

## Źródła informacji

Wykorzystać istniejące metadane projektu, np.:

```text
source_expected_text
expected_text
plate_text
ground_truth
source_expected_texts
```

Nie budować nowej bazy.

## Wynik

Dla deficytowych klas zwrócić listę:

```text
symbol
current_train_count
current_unique_train
available_unused_real_sources
```

---

# 9. P0 — kolejność dołączania realnych źródeł

Dodatkowe realne źródła mają pierwszeństwo przed augmentacją.

Workflow:

```text
deficyt
→ real source search
→ użytkownik wybiera/zatwierdza
→ nowy wariant train
→ analiza ponownie
→ dopiero potem targeted augmentation
```

Nie dołączać automatycznie bez zatwierdzenia.

---

# 10. P0 — nie ruszać val/test

Przed i po wykonaniu korekty:

- policzyć liczbę plików `val`,
- policzyć liczbę plików `test`,
- opcjonalnie hash listy plików / rozmiarów.

Po operacji:

```text
val_before == val_after
test_before == test_after
```

Jeżeli nie:

```text
ERROR
```

i wariant nie może zostać oznaczony jako gotowy.

---

# 11. P0 — artefakt BEFORE

Przed jakąkolwiek korektą zapisać automatycznie:

```text
character_class_distribution_before.csv
character_class_distribution_before.json
```

Nie wymagać od użytkownika ręcznego wpisywania nazwy.

Dopuszczalny folder np.:

```text
<variant>/analysis/
```

lub istniejący katalog eksperymentu.

---

# 12. P0 — artefakt AFTER

Po:

- dołączeniu realnych źródeł,
- targeted augmentation,

automatycznie wykonać ponowną analizę i zapisać:

```text
character_class_distribution_after.csv
character_class_distribution_after.json
```

---

# 13. P0 — hash artefaktów before/after

Manifest wariantu treningowego ma przechowywać nie tylko ścieżkę, ale również SHA-256.

Preferowany kontrakt:

```json
"before_distribution": {
  "path": "character_class_distribution_before.json",
  "sha256": "..."
},
"after_distribution": {
  "path": "character_class_distribution_after.json",
  "sha256": "..."
}
```

Nie używać samego katalogu datasetu jako referencji do before/after.

---

# 14. P0 — finalny manifest wariantu MZ

Manifest wariantu ma zawierać minimum:

```json
{
  "schema": "alpr.mz_training_variant.v1",
  "created_at": "...",
  "base_dataset": "...",
  "base_dataset_sha_or_fingerprint": "...",
  "alphabet": "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ",
  "target_ratio": 0.5,
  "target_count": 0,
  "selection_policy": "deficit_weighted",
  "max_augmented_variants_per_source": 0,
  "sources": {
    "real": 0,
    "added_real": 0,
    "augmented_real": 0,
    "synthetic": 0,
    "augmented_synthetic": 0
  },
  "before_distribution": {
    "path": "...",
    "sha256": "..."
  },
  "after_distribution": {
    "path": "...",
    "sha256": "..."
  },
  "balance_plan": {},
  "val_test_unchanged": true
}
```

---

# 15. P0 — pochodzenie próbek

Jeżeli istniejący system manifestów pozwala, zachować logiczne pochodzenie:

```text
real
added_real
augmented_real
synthetic
augmented_synthetic
```

Nie zmieniać formatu YOLO tylko po to, żeby dopisać pochodzenie.

Przechować je w manifeście wariantu / augmentacji.

---

# 16. P1 tylko jeśli konieczne — syntetyczne tablice

Nie implementować teraz automatycznie.

Uruchamiać wyłącznie gdy po:

```text
real source search
+
targeted real augmentation
```

pozostaje:

```text
LOW/CRITICAL + LOW_DIVERSITY
```

Jeżeli nie występuje taki przypadek:

```text
synthetic = OUT OF SCOPE
```

---

# 17. P0 — poprawić guard porównywalności: Android version

Aktualny indeks eksperymentu przechowuje informacje o urządzeniu/systemie.

Guard ma jawnie pokazywać:

```text
Android version
```

jako osobny wiersz.

Wymóg:

Jeśli w jednej porównywanej serii występują różne wersje Androida:

```text
Status = Różne
```

---

# 18. P0 — poprawić guard porównywalności: model fingerprints

Guard ma jawnie porównywać:

```text
MP fingerprint
MT fingerprint
MZ fingerprint
```

lub stabilny hash całego słownika fingerprintów.

Preferowane:

osobne role, jeśli dane istnieją.

Przykład:

| Kryterium | Wybrany | Seria | Status |
|---|---|---|---|
| MT fingerprint | abc123 | abc123 | Spójne |
| MZ fingerprint | def456 | 2 wartości | Różne |

To jest krytyczne dla poprawnych porównań modeli.

---

# 19. P0 — scenario_id i series_id

Guard ma:

- grupować po `series_id`,
- dodatkowo uwzględniać `scenario_id`,
- sygnalizować brak `scenario_id`,
- nie sugerować pełnej porównywalności przy różnych scenariuszach.

---

# 20. P0 — pełne dane niezależne od preview

Ta część jest już zaimplementowana.

Nie przebudowywać jej.

Należy tylko:

- upewnić się, że publiczne API jest stabilne,
- dodać test regresyjny, jeśli czegoś brakuje,
- używać pełnych iteratorów w przyszłych analizach badawczych.

Nie wykonywać obliczeń naukowych na:

```text
bundle.trace_rows
```

jeżeli raport jest większy niż preview.

Do analizy używać:

```python
iter_full_trace_rows(...)
iter_full_thermal_rows(...)
iter_full_frame_flow_rows(...)
iter_full_event_rows(...)
iter_full_sample_rows(...)
```

---

# 21. P0 — jakość i ground truth

Zasada bez zmian:

```text
brak GT != accuracy 0
```

Ma być:

```text
quality unavailable
```

Exact match / CER:

- tylko dla próbek z ground truth,
- jednostka jakości = unikalna tablica/track,
- confidence nie jest miarą poprawności.

---

# 22. P0 — test deduplikacji planner source_key

Dodać test:

```text
plate_001.txt
plate_001_aug1.txt
plate_001_aug2.txt
```

z manifestem augmentacji.

Oczekiwanie:

```text
len(plan.candidates_for_source_001) == 1
```

---

# 23. P0 — test max variants per source

Przy:

```text
max_augmented_variants_per_source = 3
```

executor nie może utworzyć więcej niż 3 nowych próbek dla jednego realnego źródła.

---

# 24. P0 — test val/test untouched

Utworzyć fixture:

```text
train
val
test
```

Uruchomić targeted balancing.

Sprawdzić:

```text
val file set before == val file set after
test file set before == test file set after
```

---

# 25. P0 — test before/after manifest

Sprawdzić:

```text
before.path
before.sha256
after.path
after.sha256
val_test_unchanged
selection_policy
target_count
max_augmented_variants_per_source
```

---

# 26. P0 — test real source search

Fixture:

```text
train: 2 tablice z Q
pool: 5 dodatkowych realnych tablic z Q
```

Oczekiwanie:

```text
available_unused_real_sources(Q) == 5
```

Źródła już obecne w train nie mogą zostać policzone jako unused.

---

# 27. P0 — test guarda: Android version

Dwa raporty:

```text
Android 12
Android 13
```

ten sam `series_id` i `scenario_id`.

Oczekiwanie:

```text
Android version -> Różne
```

---

# 28. P0 — test guarda: model fingerprints

Dwa raporty:

```text
MT=A, MZ=B
MT=A, MZ=C
```

Oczekiwanie:

```text
MT -> Spójne
MZ -> Różne
```

---

# 29. P0 — test importu replik

Dwa raporty:

```text
same series_id
same scenario_id
same experiment_variant
replicate_index = 1
replicate_index = 2
```

Oba muszą pozostać widoczne.

---

# 30. P0 — test deduplikacji identycznego pliku

Dwukrotny import dokładnie tego samego archiwum:

```text
reports count += 0
```

Nie dublować.

---

# 31. P0 — test full data > preview

Zachować istniejący test 12000 rekordów.

Sprawdzić:

```text
trace_total = 12000
preview = 5000
full iterator = 12000
```

Analogicznie:

```text
thermal
frame_flow
events
samples
```

---

# 32. P0 — test mapy klas

Zachować i uruchomić:

- poprawna lista,
- poprawny dict,
- swap A/B,
- missing class,
- extra class,
- missing YAML flat = warning,
- missing YAML split = error.

---

# 33. P0 — uruchomienie całego test suite

Agent ma uruchomić:

```bash
python -m unittest
```

lub właściwy command repozytorium, jeśli suite korzysta z innego runnera.

Wynik:

```text
0 failures
0 errors
```

Jeżeli są istniejące niezwiązane testy czerwone:

- nie ignorować ich po cichu,
- opisać je w raporcie,
- rozdzielić regresję od starego problemu.

---

# 34. P0 — compile check

Uruchomić dla zmienionych modułów:

```bash
python -m compileall auto_annotation_tool
```

albo równoważny test składni.

Oczekiwanie:

```text
0 syntax errors
```

---

# 35. P0 — git diff check

Przed freeze:

```bash
git diff --check
```

Oczekiwanie:

```text
brak błędów whitespace
```

---

# 36. P0 — smoke test ręczny

Wykorzystać:

```text
docs/freeze_smoke_test_checklist.md
```

Przejść **każdy punkt**.

Nie oznaczać freeze tylko dlatego, że testy jednostkowe są zielone.

---

# 37. P0 — smoke test MZ

Minimalnie:

1. wybrać prawdziwy dataset MZ,
2. uruchomić analizę,
3. sprawdzić mapę klas,
4. zapisać before,
5. uruchomić real-source search,
6. zatwierdzić minimalne uzupełnienie,
7. uruchomić targeted augmentation,
8. zapisać after,
9. sprawdzić manifest,
10. sprawdzić, że val/test są identyczne.

---

# 38. P0 — smoke test Android → Desktop

Minimalnie:

1. import jednego `.alprsession`,
2. import kilku plików,
3. import JSON z wieloma raportami,
4. duplikat identycznego pliku,
5. dwie repliki,
6. `series_id`,
7. `scenario_id`,
8. `replicate_index`,
9. guard,
10. pełne dane > preview.

---

# 39. P0 — zachować wynik smoke testu

Po przejściu checklisty zapisać krótki artefakt:

```text
docs/freeze_smoke_test_result.md
```

Minimalnie:

```text
date
desktop SHA
python version
OS
test suite result
smoke test result
known limitations
```

---

# 40. P0 — known limitations

Przed freeze zapisać jawnie rzeczy, których NIE poprawiamy.

Przykładowo:

- brak automatycznego generatora syntetycznych tablic,
- część wykresów badawczych będzie generowana poza GUI,
- preview GUI jest ograniczony,
- pełna analiza korzysta z iteratorów backendowych.

To nie są błędy, jeśli są świadome i udokumentowane.

---

# 41. P0 — commit freeze

Po wszystkim:

```bash
git status
git diff
git diff --check
```

następnie commit np.:

```text
Freeze desktop research system for thesis experiments
```

---

# 42. P0 — tag freeze

Utworzyć tag:

```text
thesis-experiment-freeze
```

Jeżeli tag istnieje:

```text
thesis-experiment-freeze-v2
```

Nie nadpisywać starego tagu bez potrzeby.

---

# 43. P0 — zanotować SHA

SHA freeze musi znaleźć się:

- w dzienniku architektury,
- w smoke test result,
- później w pracy / notatkach eksperymentalnych.

---

# 44. Zasady po freeze

Od momentu freeze:

Nie zmieniać w ramach tej samej kampanii:

- pipeline,
- parserów,
- schedulerów,
- thresholdów,
- datasetu,
- model fingerprintów,
- logiki raportowania,
- kontraktu Android/Desktop,
- algorytmów balansu.

---

# 45. Krytyczny bug po freeze

Jeżeli pojawi się krytyczny bug:

```text
fix
→ nowy commit
→ testy
→ nowy smoke test
→ nowy freeze SHA
→ nowa seria eksperymentów
```

Nie mieszać wyników sprzed i po poprawce.

---

# 46. Poza zakresem

Agent nie może teraz dodawać:

- nowych architektur ML,
- nowego trackera,
- nowych backendów inferencji,
- nowego dashboardu,
- nowej bazy danych,
- automatycznego syntetycznego generatora bez realnej potrzeby,
- kosmetycznych przebudów GUI,
- refactorów „dla porządku”,
- zmian niezwiązanych z freeze.

---

# 47. Finalna definicja DONE

System desktopowy można uznać za skończony tylko wtedy, gdy:

## MZ

- [ ] mapa klas jest walidowana,
- [ ] balans jest analizowany,
- [ ] różnorodność źródeł jest analizowana,
- [ ] planner jest deduplikowany po `source_key`,
- [ ] real-source search działa,
- [ ] planner jest podłączony do augmentacji,
- [ ] limit wariantów per source jest respektowany,
- [ ] base dataset nie jest modyfikowany,
- [ ] val/test są nietknięte,
- [ ] before zapisany,
- [ ] after zapisany,
- [ ] manifest ma SHA artefaktów,
- [ ] pochodzenie danych jest udokumentowane.

## Android → Desktop

- [ ] import pojedynczego raportu działa,
- [ ] import wielu raportów działa,
- [ ] repliki są zachowywane,
- [ ] duplikaty są deduplikowane,
- [ ] pełne dane są dostępne niezależnie od preview,
- [ ] series/scenario/replicate są indeksowane,
- [ ] guard sprawdza urządzenie,
- [ ] guard sprawdza Android version,
- [ ] guard sprawdza app SHA,
- [ ] guard sprawdza runtime,
- [ ] guard sprawdza delegate,
- [ ] guard sprawdza resolution,
- [ ] guard sprawdza recognition profile,
- [ ] guard sprawdza MP/MT/MZ fingerprints,
- [ ] brak GT nie jest traktowany jako accuracy=0.

## Jakość kodu

- [ ] test suite green,
- [ ] compileall green,
- [ ] git diff --check green,
- [ ] smoke test green,
- [ ] smoke result zapisany,
- [ ] freeze commit wykonany,
- [ ] freeze tag wykonany,
- [ ] SHA zapisane.

---

# 48. Raport końcowy agenta

Po wykonaniu agent ma zwrócić użytkownikowi raport w formacie:

```text
1. Zmienione pliki
2. Co zostało domknięte
3. Testy uruchomione
4. Wyniki testów
5. Smoke test
6. Znane ograniczenia
7. Freeze SHA
8. Freeze tag
9. Czy system jest gotowy do badań: TAK/NIE
```

Jeżeli odpowiedź brzmi `NIE`, agent ma wskazać dokładnie **co blokuje rozpoczęcie badań**.

---

# 49. Ostatnia zasada

> Nie kończ zadania na „kod został napisany”.  
> Zadanie kończy się dopiero wtedy, gdy kod został uruchomiony, przetestowany, smoke-testowany i oznaczony SHA/tagiem freeze.
