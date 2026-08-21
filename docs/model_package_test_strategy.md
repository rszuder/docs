# Strategia testowania paczki modeli ALPR

Stan dokumentu: 2026-08-20. Dokument definiuje plan testów pojedynczych
`alpr.model.v1`, kompletnych `alpr.package.v1` oraz ich wykonania w Androidzie.
Nie oznacza, że wszystkie opisane testy są już zaimplementowane.

## 1. Cel i jednostka oceny

Testowanym produktem wdrożeniowym jest kompletny pakiet `MT+MZ` albo opcjonalnie
`MP+MT+MZ`, a nie sam checkpoint. Pakiet obejmuje:

- wagi MT i MZ oraz, jeśli używana jest kaskada pojazdów, wagi MP;
- manifesty i sumy SHA-256;
- preprocessing i kontrakty tensorów;
- dekodery i progi postprocessingu;
- runtime, precyzję, delegata oraz parametry wykonania;
- rektyfikację i składanie sekwencji po stronie Androida.

Pojedyncze MT i MZ testuje się osobno, aby zlokalizować przyczynę błędu.
Decyzję o wdrożeniu podejmuje się jednak na podstawie wyniku kompletnego
pipeline'u na wspólnym, zamrożonym torze testowym.

## 2. Zasady nadrzędne

1. Wszystkie warianty runtime jednego modelu muszą pochodzić z tego samego
   checkpointu i mieć ten sam `source.checkpoint_sha256`.
2. Ranking kandydatów używa identycznego splitu i kolejności próbek.
3. `final_test` jest zamrażany przed wyborem zwycięzcy i nie służy do strojenia
   progów ani wyboru wariantu.
4. Confidence nie jest miarą skuteczności. Jakość wymaga ground truth.
5. Pomiary jakości, wydajności chłodnej i throttlingu są osobnymi wynikami.
6. Najpierw obowiązują bramki twarde. Punktację Pareto/ranking liczy się tylko
   dla pakietów poprawnych, zgodnych numerycznie i stabilnych.
7. Każdy wynik wiąże się z `package_id`, SHA-256 artefaktu, wersją aplikacji,
   urządzeniem, runtime'em i pełnym identyfikatorem runu.

## 3. Zamrożone dane i artefakty testowe

### 3.1. Splity

| Split | Zastosowanie | Zakaz |
| --- | --- | --- |
| `train` | uczenie | nie raportować jako wynik końcowy |
| `val` | wybór epoki | nie używać jako final test |
| `ranking` | porównanie kandydatów desktopowych | nie zmieniać między kandydatami |
| `calibration_int8` | kalibracja INT8 | nie używać jako test jakości |
| `mobile_replay` | parytet i wydajność telefonu | nie stroić per runtime |
| `final_test` | wynik końcowy pracy | nie używać do wyboru modelu |

Każdy split ma plik indeksu, SHA-256 indeksu i SHA-256 każdego obrazu.
`dataset_id` powinien zawierać nazwę, iterację i hash indeksu.

### 3.2. Golden fixtures

Minimalny zestaw kontrolny:

```text
test-fixtures/model-packages/
├── valid/
│   ├── mt_tflite_fp32.alprmodel
│   ├── mz_tflite_fp32.alprmodel
│   ├── mt_multi_runtime.alprmodel
│   ├── complete_mt_mz_legacy.alprmodel
│   └── complete_mp_mt_mz.alprmodel
├── invalid/
│   ├── bad_manifest_schema.alprmodel
│   ├── bad_sha256.alprmodel
│   ├── missing_variant.alprmodel
│   ├── duplicate_entry.alprmodel
│   ├── zip_slip.alprmodel
│   ├── labels_class_count_mismatch.alprmodel
│   ├── plate_without_four_keypoints.alprmodel
│   ├── unsupported_role_task.alprmodel
│   ├── dynamic_or_batch_gt_1.alprmodel
│   ├── broken_complete_pipeline.alprmodel
│   ├── vehicle_entry_without_stage.alprmodel
│   └── vehicle_stage_without_entry.alprmodel
└── golden/
    ├── frames/
    ├── rectified_crops/
    ├── ground_truth.jsonl
    ├── python_tensors/
    └── python_detections.jsonl
```

Fixture ważnych modeli przechowuje małe wagi testowe lub syntetyczne grafy,
nie pełne modele produkcyjne. Produkcyjne paczki są przechowywane jako osobne,
wersjonowane artefakty badawcze.

## 4. Poziomy testów

### T0 — preflight eksportera Python

Sprawdzić przed konwersją:

- istnienie i SHA-256 checkpointu;
- rolę `plate/character/vehicle` oraz zadanie `pose/detect`;
- liczbę klas i kolejność labels;
- dla MT co najmniej cztery keypointy oraz ich wymiar 2 lub 3;
- statyczny batch 1 i statyczne `imgsz`;
- dostępność zależności tylko dla wybranych formatów;
- obecność reprezentatywnego `data.yaml` dla INT8;
- jednoznaczne `model_id`, `package_id`, wersję i pochodzenie datasetu.

Wynik: eksport nie rozpoczyna się przy błędzie preflight. Nie wolno pozostawić
częściowego pliku docelowego.

### T1 — integralność i bezpieczeństwo archiwum

Testy parsera ZIP i manifestu obejmują:

- zgodność z JSON Schema;
- brak ścieżek absolutnych, `..`, separatorów Windows i pustych segmentów;
- brak powtórzonych wpisów;
- limit liczby wpisów i sumarycznego rozmiaru po rozpakowaniu;
- obecność każdego pliku zadeklarowanego przez wariant;
- poprawne SHA-256 plików, manifestów bocznych i pakietów potomnych;
- odrzucenie uszkodzonego ZIP, nieznanego schematu i niepełnego pipeline'u;
- atomową instalację: błąd nie zmienia aktywnego pakietu ani rejestru.

### T2 — statyczny kontrakt tensorów

Dla każdego wariantu otworzyć model przez rzeczywisty runtime i porównać z
manifestem:

- wejście `[1,H,W,3]` albo `[1,3,H,W]`;
- batch równy 1, kanały równe 3 i statyczne H/W;
- `FLOAT32`, `INT8` lub `UINT8` zgodnie z deklaracją;
- scale/zero point kwantyzacji oraz RGB/BGR;
- kształt wyjścia, układ tensorów, objectness, class count;
- dla MT liczbę i wymiar keypointów;
- zgodność `nms_in_graph` z dekoderem raw/end-to-end.

Nie wolno uruchamiać inferencji, jeżeli introspekcja runtime przeczy manifestowi.

### T3 — parytet referencyjny Python–Android

Porównanie jest warstwowe dla tych samych obrazów golden:

1. wynik resize/letterbox, padding i tensor wejściowy;
2. surowy tensor wyjściowy;
3. zdekodowane boxy, klasy, confidence i keypointy przed NMS;
4. wynik NMS i deduplikacji;
5. homografia i obraz zrektyfikowany;
6. boxy znaków, kolejność i końcowa sekwencja.

Początkowe bramki parytetu FP32 dla próbek oddalonych od progu decyzyjnego:

| Element | Bramka początkowa |
| --- | --- |
| tensor wejścia FLOAT32 | `max_abs_error <= 1e-6` |
| confidence zdekodowane | `abs_error <= 0.02` |
| dopasowane boxy | `IoU >= 0.98` |
| narożniki MT | średni błąd `<= 0.005` przekątnej tablicy |
| rektyfikacja | identyczny rozmiar i jawnie raportowany MAE pikseli |
| kolejność znaków | dokładnie zgodna z referencją Python |
| pełna sekwencja | exact match 100% względem referencji na golden set |

Próbki z confidence w odległości mniejszej niż 0,02 od progu trafiają do osobnej
kategorii `threshold_sensitive`; nie powinny same blokować kontraktu runtime.
Podane tolerancje są wartościami startowymi do potwierdzenia na rzeczywistych
modelach, a nie wynikami eksperymentu.

### T4 — import i cykl życia Androida

Test instrumentalny przez `ContentResolver` powinien potwierdzić:

- import pojedynczych MT/MZ i kompletnego `alpr.package.v1`;
- zgodność paczki historycznej MT+MZ z czterema etapami;
- import paczki MP+MT+MZ z pięcioma etapami i `vehicle_detection` na początku;
- zachowanie `source.alprmodel` i jego SHA-256;
- aktywację wszystkich modeli obecnych w komplecie jako jednej transakcji;
- pozostawienie włączenia kaskady MP jako jawnej decyzji użytkownika;
- odtworzenie rejestru po restarcie procesu i aktywności;
- ponowny import tej samej paczki bez duplikacji;
- zmianę wersji/fingerprintu i unieważnienie profilu autotuningu;
- brak zmiany aktywnego pipeline'u po błędzie importu;
- działanie z SAF i odebraniem/nadaniem uprawnień URI;
- brak plików staging/cache po sukcesie i po błędzie.

### T5 — smoke test macierzy runtime

| Runtime | Tryby obowiązkowe | Wynik |
| --- | --- | --- |
| LiteRT FP32 | CPU 1/2/4 wątki | inferencja MT i MZ bez NaN/crasha |
| LiteRT FP32 | GPU, jeżeli dostępny | poprawny delegate i parytet semantyczny |
| LiteRT INT8/UINT8 | CPU, opcjonalnie GPU | poprawna kwantyzacja i jakość |
| ONNX FP32 | CPU 1/2/4 wątki | inferencja i fallback |
| NCNN | import zawsze, wykonanie po dodaniu JNI | jawny status unsupported bez crasha |

Autotuning musi porównywać warianty tych samych wag. Każda awaria kandydata ma
być zapisana, a fallback nie może zmieniać modelu logicznego bez raportowania.

### T6 — jakość modeli i pipeline'u

#### MT

- precision, recall, F1;
- mAP@0.5 i mAP@0.5:0.95;
- błąd narożników znormalizowany przekątną tablicy;
- udział poprawnych rektyfikacji;
- FP/FN według wysokości tablicy, kąta, blur i oświetlenia.

#### MZ na referencyjnych cropach

- precision, recall, F1 oraz mAP znaków;
- accuracy per znak i macierz pomyłek;
- braki, nadmiarowe znaki i duplikaty po NMS;
- CER, znormalizowana odległość edycyjna i exact match tablicy.

#### End-to-end

- exact match oraz CER per unikalny `session_id + track_id`;
- udział tracków bez odczytu;
- czas i liczba prób do pierwszego/potwierdzonego wyniku;
- kategorie błędu: brak MT, złe rogi, rektyfikacja, brak/duplikat/zła klasa MZ,
  kolejność, niestabilny konsensus.

### T7 — INT8 kontra FP32

INT8 nie może zostać promowany wyłącznie dlatego, że jest mniejszy lub szybszy.
Początkowa bramka kandydacka, konfigurowalna w protokole:

- spadek exact match end-to-end nie większy niż 1 punkt procentowy;
- wzrost CER nie większy niż 0,005 bezwzględnie;
- spadek recall MT nie większy niż 1 punkt procentowy;
- oraz co najmniej 10% poprawy p90 albo 30% redukcji rozmiaru artefaktu.

Jeżeli próba jest zbyt mała, raport ma oznaczyć bramkę jako `inconclusive`, a nie
zaliczoną. Progi należy zatwierdzić przed otwarciem `final_test`.

### T8 — kontrolowany benchmark mobilny

Tryb replay wykorzystuje zamrożony `mobile_replay` i raportuje:

- cold load, inicjalizację delegata i pierwszą inferencję osobno;
- steady-state per etap i pipeline;
- p50/p90/p95/p99, średnią, odchylenie, min/max i liczbę próbek;
- minimum 1024 inferencje i 60 s dla serii inspirowanej MLPerf single-stream;
- minimum trzy serie na konfigurację;
- stan termiczny, baterię, ładowanie, RAM/PSS, dropped frames;
- temperaturę otoczenia 20–25°C i minimum 10 min cooldown między pełnymi seriami.

Wyniku nie wolno nazywać „MLPerf”, ponieważ aplikacja nie używa LoadGena i nie
przechodzi oficjalnego procesu submission.

### T9 — sesje live i odporność

Scenariusze ręczne lub z kontrolowanym nagraniem:

- kamera statyczna i telefon w ruchu;
- bliski/daleki odczyt oraz mała tablica w pikselach;
- dzień, cień, noc, refleks, rozmycie i zabrudzenie;
- obrót, perspektywa, tablica jedno- i dwuwierszowa;
- jedna i wiele tablic, pojazd osobowy, motocykl i pojazd ciężki;
- MP wyłączony/włączony, utrata ROI i pełnoklatkowy fallback;
- 30-minutowa sesja termiczna, przejście tło/pierwszy plan i obrót aktywności;
- osiągnięcie limitu cropów, zbiorczy zapis i eksport `.alprsession`.

Bramka stabilności: brak crasha/ANR, NaN i uszkodzonych raportów; brak
nieograniczonego wzrostu pamięci po zakończeniu rozgrzewki.

### T10 — integralność raportu badawczego

Sprawdzić dla `.alprsession` oraz `thesis_bundle.zip`:

- zgodność manifestu ze schematem;
- SHA-256 wszystkich wpisów;
- zgodność `package_id/variant_id` z faktycznie aktywnymi modelami;
- obecność trace'ów, środowiska, protokołu, cropów i adnotacji;
- exact match/CER odtworzone niezależnie w Pythonie;
- kompilację `summary.tex` z czystego katalogu;
- względne ścieżki figur i bezpieczne escapowanie TeX;
- import `.alprsession` w programie macierzystym po wdrożeniu importera.

## 5. Bramki akceptacyjne pakietu

| ID | Bramka twarda | Warunek zaliczenia |
| --- | --- | --- |
| G0 | integralność | schema, struktura i wszystkie SHA-256 poprawne |
| G1 | bezpieczeństwo | wszystkie fixtures niebezpieczne są odrzucane |
| G2 | kontrakt runtime | introspekcja tensorów zgodna z manifestem |
| G3 | parytet | golden set mieści się w zatwierdzonych tolerancjach |
| G4 | import | instalacja atomowa, aktywacja i restart poprawne |
| G5 | jakość | osiągnięte wcześniej zamrożone minimum MT/MZ/E2E |
| G6 | INT8 | przechodzi osobną bramkę jakości, jeśli ma być aktywny |
| G7 | wydajność | mieści się w budżecie urządzenia docelowego |
| G8 | stabilność | brak crash/ANR/NaN i nieograniczonego wzrostu RAM |
| G9 | raport | pełny, walidowalny eksport z reprodukowalną konfiguracją |

Pakiet z niezaliczoną bramką twardą ma status `rejected` albo `inconclusive`.
Nie trafia do rankingu ważonego.

## 6. Ranking po przejściu bramek

Ranking jest analizą Pareto, a nie zamianą jakości na dowolnie dużą szybkość.
Minimalne wykresy:

- exact match end-to-end kontra p90 pipeline;
- exact match kontra rozmiar pakietu;
- CER kontra p90 MZ;
- mAP50-95 MT kontra p90 MT;
- FP32 kontra INT8 tego samego checkpointu;
- p50/p90/p95/p99 per runtime i urządzenie;
- skuteczność według wielkości tablicy, blur i ruchu.

Jeżeli potrzebny jest pojedynczy score roboczy, jego wagi i normalizacja muszą
zostać zamrożone przed `final_test`, a w pracy należy pokazać także surowe
metryki i wykres Pareto.

## 7. Identyfikacja runu i wyniki

Identyfikator:

```text
<package_id>__<variant_mt>__<variant_mz>__<device_id>__<scenario>__r<repeat>
```

Każdy run zapisuje:

- pełny `.alprsession` lub odwołanie do jego SHA-256;
- hash aplikacji/APK i wersję kodu;
- hash dataset index;
- konfigurację MT/MZ/MP i runtime;
- surowe predykcje oraz ground truth;
- trace'y czasowe i dane termiczne;
- wynik każdej bramki z przyczyną.

Nie nadpisywać runów. Powtórzenie ma nowy `run_id`, nawet jeżeli konfiguracja
jest identyczna.

## 8. Kolejność realizacji

### P0 — testy szybkie na każdy commit

- parser manifestu i ścieżek ZIP;
- dekodery raw/end-to-end;
- geometria, NMS, reading order i stabilizator;
- metryki CER/exact match i generatory raportów;
- build, unit tests i lint.

### P1 — na każdy nowy eksport

- T0–T3 na desktopie;
- walidacja schematu i SHA-256;
- golden parity wszystkich wariantów.

### P2 — na każdy kandydacki pakiet

- T4–T6 na telefonie;
- pełna macierz dostępnych runtime'ów;
- jakość MT, MZ i end-to-end na ranking/mobile_replay.

### P3 — dla finalistów

- T7–T9: INT8, powtarzany benchmark, termika, live i stres;
- analiza Pareto.

### P4 — raz dla zwycięzcy

- zamknięty `final_test`;
- T10 i eksport materiałów do pracy;
- archiwizacja APK, `.alprmodel`, `.alprsession`, TeX i wszystkich hashy.

## 9. Stan pokrycia w bieżącym repozytorium Android

| Obszar | Stan 2026-08-20 |
| --- | --- |
| ścieżki ZIP i podstawowe bezpieczeństwo | częściowo pokryte JVM |
| dekodery YOLO raw/end-to-end | pokryte testami syntetycznymi |
| geometria, kolejność, NMS/deduplikacja | pokryte testami JVM |
| tracker, scheduler i stabilizacja | pokryte testami JVM |
| CER, exact match per track | JVM + test instrumentalny |
| struktura thesis bundle i SHA-256 | test instrumentalny na Samsungu |
| pełny fixture `.alprmodel` | brak |
| import przez rzeczywisty `ContentResolver` | brak regresji instrumentalnej |
| zachowanie `source.alprmodel` | kod istnieje, brak fixture testu |
| manifesty kompletów 4/5-etapowych | test instrumentalny parsera Android |
| parytet Python–Android | brak golden tensor/detection fixtures |
| realne MT+MZ na telefonie | brak zakończonego protokołu |
| kontrolowany replay 1024/60 s | brak trybu wykonawczego |
| kompilacja wygenerowanego TeX | brak testu CI/desktop |
| importer `.alprsession` w Pythonie | brak |

## 10. Powiązane źródła i dokumenty

- `docs/model_package_v1.md` — kontrakt pakietu;
- `docs/alpr-model-v1.schema.json` — schema pojedynczego modelu;
- `docs/alpr-package-v1.schema.json` — schema kompletnej paczki MT+MZ lub
  MP+MT+MZ;
- `docs/mobile_research_export.md` — raport badawczy;
- `docs/alpr-mobile-research-bundle-v1.schema.json` — schema eksportu sesji;
- `DZIENNIK_BUDOWY_APLIKACJI.md` — decyzje i stan realizacji;
- desktopowy `docs/siatka_eksperymentow_mobilnych_alpr.md` — szersza siatka
  eksperymentów treningowych i rankingu.

Podbudowa metodyki: MLPerf Mobile, PMLSys 4 (2022), arXiv `2012.02328`;
Laroca i in., DOI `10.1109/IJCNN.2018.8489629`; Shahab i in., DOI
`10.1109/ICDAR.2011.296`; Silva i Jung, DOI
`10.1007/978-3-030-01258-8_36`.
