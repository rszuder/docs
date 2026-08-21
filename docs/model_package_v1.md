# Pakiet modelu ALPR v1

Klient Android nie importuje surowych checkpointów `.pt`. Program Python eksportuje jeden lub kilka wariantów tego samego checkpointu do archiwum ZIP z rozszerzeniem `.alprmodel`.

## Struktura

```text
model.alprmodel
├── manifest.json
└── variants/
    ├── tflite/model.tflite
    ├── onnx/model.onnx
    └── ncnn/model.param + model.bin
```

`manifest.json` musi znajdować się w katalogu głównym archiwum. Każdy plik wariantu ma obowiązkową sumę SHA-256. Importer odrzuca ścieżki wychodzące poza katalog pakietu, powtórzone wpisy, brakujące pliki, błędne sumy oraz archiwa większe niż 512 MB po rozpakowaniu.

## Manifest

```json
{
  "schema": "alpr.model.v1",
  "model_id": "plate-yolo-pose-001",
  "name": "Detektor tablic 001",
  "version": "1",
  "role": "plate",
  "task": "pose",
  "input": {
    "width": 640,
    "height": 640,
    "channels": 3,
    "layout": "NHWC",
    "color": "RGB",
    "data_type": "FLOAT32",
    "scale": 0.0039215686,
    "offset": 0.0
  },
  "output": {
    "decoder": "ultralytics_pose_raw_v1",
    "class_count": 1,
    "keypoint_count": 4,
    "has_objectness": false,
    "tensor_layout": "channels_first",
    "normalized_coordinates": false,
    "nms_in_graph": false,
    "confidence_threshold": 0.25,
    "iou_threshold": 0.45
  },
  "labels": ["plate"],
  "variants": [
    {
      "id": "tflite-fp32",
      "runtime": "tflite",
      "precision": "fp32",
      "file": "variants/tflite/model.tflite",
      "sha256": {
        "variants/tflite/model.tflite": "SUMA_SHA256_UZUPELNIANA_PRZEZ_EKSPORTER"
      }
    },
    {
      "id": "onnx-fp32",
      "runtime": "onnx",
      "precision": "fp32",
      "file": "variants/onnx/model.onnx",
      "input": {
        "width": 640,
        "height": 640,
        "channels": 3,
        "layout": "NCHW",
        "color": "RGB",
        "data_type": "FLOAT32",
        "scale": 0.0039215686,
        "offset": 0.0
      },
      "sha256": {
        "variants/onnx/model.onnx": "SUMA_SHA256_UZUPELNIANA_PRZEZ_EKSPORTER"
      }
    }
  ]
}
```

Dozwolone role to `vehicle`, `plate` i `character`. Obsługiwane formaty pakietu to `tflite`, `onnx` i `ncnn`. Klient uruchamia warianty TFLite oraz ONNX i porównuje je podczas autotuningu. Poprawne warianty NCNN są importowane i przechowywane, ale ich wykonanie wymaga opcjonalnego adaptera JNI.

Model znaków używa `role: character`, `task: detect` oraz pełnej tablicy `labels` w kolejności identyfikatorów klas datasetu. Model tablic typu `pose` musi zwracać przynajmniej cztery keypointy.

Obsługiwane są zarówno surowe dekodery Ultralytics `*_raw_v1`, jak i wyjścia
`ultralytics_pose_end2end_v1`, `ultralytics_detect_end2end_v1` oraz
`ultralytics_yolo_end2end_v1`. Format end-to-end zawiera rekordy `xyxy`, confidence,
class ID i opcjonalne keypointy, dlatego nie jest dla niego wykonywany dodatkowy NMS.
W standardowym surowym układzie `[1, N, C]` manifest powinien deklarować
`"tensor_layout": "anchors_first"`, a dla wyjścia end-to-end może użyć równoważnej
wartości `"detections_first"`. Pole `keypoint_dimensions` określa, czy każdy punkt
zawiera `x, y`, czy `x, y, confidence`.

Pola `input` i `output` z poziomu pakietu są wartościami domyślnymi. Wariant może je nadpisać, co jest potrzebne np. wtedy, gdy TFLite przyjmuje tensor `NHWC`, a ONNX tensor `NCHW`.

## Zasady porównania formatów

Warianty TFLite, ONNX i NCNN znajdujące się w jednym pakiecie muszą pochodzić z tego samego checkpointu. Tylko wtedy wyniki autotuningu pozwalają porównać wpływ środowiska wykonawczego, bez mieszania go z różnicami wag modelu.

## Kompletny pakiet MP+MT+MZ

Importer rozpoznaje również schemat `alpr.package.v1`. Wymagane są modele tablic
(MT) i znaków (MZ), a model pojazdów (MP) jest opcjonalny. Dzięki temu istniejące
paczki MT+MZ pozostają zgodne, natomiast eksporter desktopowy może utworzyć pełną
kaskadę MP → obszar zainteresowania → MT → MZ:

```text
manifest.json
models/vehicle/model.alprmodel       # opcjonalny MP
models/vehicle/manifest.json         # opcjonalny MP
models/plate/model.alprmodel
models/plate/manifest.json
models/character/model.alprmodel
models/character/manifest.json
```

Manifest nadrzędny musi mieć `kind: "complete_alpr_pipeline"`, wpisy `models.plate`
i `models.character`. Opcjonalny wpis `models.vehicle` musi wskazywać pakiet
`alpr.model.v1` z `role: "vehicle"` i `task: "detect"`.

Minimalny przykład pełnego manifestu (skróty SHA-256 należy zastąpić rzeczywistymi
64-znakowymi wartościami):

```json
{
  "schema": "alpr.package.v1",
  "kind": "complete_alpr_pipeline",
  "package_id": "mp-mt-mz-001",
  "name": "Kaskada MP+MT+MZ 001",
  "version": "1",
  "created_at": "2026-08-20T12:00:00Z",
  "models": {
    "vehicle": {
      "role": "vehicle",
      "task": "detect",
      "model_id": "vehicle-yolo-001",
      "schema": "alpr.model.v1",
      "package_file": "models/vehicle/model.alprmodel",
      "manifest_file": "models/vehicle/manifest.json",
      "sha256": {
        "models/vehicle/model.alprmodel": "<sha256>",
        "models/vehicle/manifest.json": "<sha256>"
      }
    },
    "plate": {
      "role": "plate",
      "task": "pose",
      "model_id": "plate-yolo-pose-001",
      "schema": "alpr.model.v1",
      "package_file": "models/plate/model.alprmodel",
      "manifest_file": "models/plate/manifest.json",
      "sha256": {
        "models/plate/model.alprmodel": "<sha256>",
        "models/plate/manifest.json": "<sha256>"
      }
    },
    "character": {
      "role": "character",
      "task": "detect",
      "model_id": "character-yolo-001",
      "schema": "alpr.model.v1",
      "package_file": "models/character/model.alprmodel",
      "manifest_file": "models/character/manifest.json",
      "sha256": {
        "models/character/model.alprmodel": "<sha256>",
        "models/character/manifest.json": "<sha256>"
      }
    }
  },
  "pipeline": [
    {"stage":"vehicle_detection","model":"vehicle","role":"vehicle","task":"detect"},
    {"stage":"plate_detection","model":"plate","role":"plate","task":"pose"},
    {"stage":"plate_rectification","implementation":"android_alpr_rectifier"},
    {"stage":"character_detection","model":"character","role":"character","task":"detect"},
    {"stage":"sequence_assembly","implementation":"android_alpr_sequence_decoder"}
  ]
}
```

Paczka bez `models.vehicle` zachowuje dotychczasowy, dokładnie czteroetapowy
pipeline rozpoczynający się od `plate_detection`. Paczka z `models.vehicle` musi
mieć dokładnie pięć etapów, a `vehicle_detection` musi być pierwszy. Importer
sprawdza SHA-256 wszystkich pakietów potomnych i manifestów bocznych, a następnie
ponownie waliduje każdy pakiet pojedynczego modelu.

Kompletny pakiet ma limit 640 wpisów i 1 GiB po rozpakowaniu. Po poprawnej
walidacji tworzy jeden rekord kompletu oraz dwa albo trzy rekordy modeli. Obecne
w komplecie MP, MT i MZ są aktywowane jedną transakcją; ich warianty runtime są
wybierane i autotuningowane osobno. Import nie włącza automatycznie przełącznika
„Kaskada pojazdów” — użytkownik nadal decyduje, czy MP ogranicza obszar analizy.
Import pojedynczego `alpr.model.v1` pozostaje obsługiwany jako tryb częściowy.

Rozmiar wejścia, progi, dekoder, etykiety i wariant wykonawczy nie są wspólne
dla kompletu. Klient rozwiązuje je osobno z manifestu i wybranego wariantu MT
oraz osobno dla MZ. Dopuszczalny jest więc np. komplet `MT 640×640` i
`MZ 416×416`, także z różnymi runtime'ami.

Formalny kontrakt manifestu nadrzędnego znajduje się w
`docs/alpr-package-v1.schema.json`. Eksporter desktopowy powinien najpierw
zbudować i zwalidować każdy pakiet potomny, zapisać identyczny manifest potomny
jako plik boczny, policzyć SHA-256 po finalnym zapisie plików, a dopiero potem
utworzyć nadrzędny `manifest.json` i atomowo zamknąć archiwum `.alprmodel`.

Autotuning mierzy warianty kwantyzowane, ale nie wybiera INT8 automatycznie,
jeżeli dany model ma wykonywalny wariant FP32. Promocja INT8 wymaga osobnej
bramki jakościowej na danych z ground truth.

Pełną strategię fixture, parytetu Python–Android, macierzy runtime, jakości,
wydajności i bramek akceptacyjnych opisuje
`docs/model_package_test_strategy.md`.
