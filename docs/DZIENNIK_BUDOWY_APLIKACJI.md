# Dziennik budowy mobilnej aplikacji ALPR

## 1. Cel dokumentu

Dokument rejestruje rozwój aplikacji Android `ALPR_v1`: wykonane prace,
decyzje architektoniczne, kontrakty wymiany danych, wyniki weryfikacji oraz
zadania pozostające do wykonania. Ma służyć jako:

- kronika techniczna projektu;
- źródło uzasadnień decyzji do pracy inżynierskiej;
- punkt startowy dla kolejnych zmian;
- zapis rozdzielający funkcje potwierdzone od planowanych.

Daty odnoszą się do historii Git i faktycznych sesji rozwojowych. Jeżeli dana
funkcja nie została sprawdzona na fizycznym urządzeniu, jest to zaznaczone
wprost.

---

## 2. Założenia aplikacji

Aplikacja jest demonstratorem i stanowiskiem pomiarowym mobilnego systemu
automatycznego rozpoznawania tablic rejestracyjnych. Nie trenuje modeli i nie
importuje checkpointów PyTorch `best.pt`. Otrzymuje gotowe pakiety
`.alprmodel` wyeksportowane przez macierzystą aplikację Python.

Docelowy przepływ:

```text
obraz CameraX
  -> opcjonalna detekcja pojazdu MP
  -> wybór ROI pojazdu
  -> MT: YOLO Pose — detekcja tablicy i czterech narożników
  -> rektyfikacja perspektywy
  -> MZ: YOLO Character Detection — detekcja i klasyfikacja znaków
  -> ustalenie kolejności znaków
  -> stabilizacja wyniku między klatkami
  -> prezentacja i raport pomiarowy
```

Terminologia stosowana w projekcie:

- **MP** — model pojazdów; lekki detektor obiektów wykorzystywany opcjonalnie
  do wyznaczania ROI dla MT;
- **MT** — model YOLO Pose wykrywający tablicę jako obiekt oraz jej cztery
  narożniki;
- **MZ** — model detekcyjny YOLO, w którym klasami detekcji są poszczególne
  znaki; wynik tekstowy powstaje dopiero po posortowaniu ramek znaków w
  kolejności odczytu;
- **OCR** — odrębna klasa metod rozpoznawania tekstu, np. model sekwencyjny
  CRNN/CTC albo usługa OCR. Obecny pipeline Androida nie zawiera takiego
  modelu;
- **ROI — Region of Interest** — fragment obrazu wybrany do dalszej analizy.

Określenia „recognizer” i „OCR” mogą pojawiać się w omówieniu literatury,
jeżeli tak nazywają swoje komponenty autorzy publikacji. Nie są używane jako
zamienniki nazwy MZ ani jako opis jego architektury.

Najważniejszą granicą systemu jest kontrakt `.alprmodel`. Aplikacja desktopowa
odpowiada za trening, eksport i kalibrację. Android odpowiada za ponowną
walidację pakietu, wybór runtime'u, inferencję, tracking, prezentację i pomiary
na urządzeniu.

---

## 3. Stos technologiczny

Stan na 2026-08-26:

| Obszar | Rozwiązanie |
| --- | --- |
| Język | Java 11 |
| Minimalny Android | API 29 |
| Docelowy Android | API 36 |
| Kamera | CameraX |
| Główny runtime | LiteRT/TFLite 1.4.1 |
| Akceleracja | LiteRT GPU 1.4.2 |
| Runtime kontrolny | ONNX Runtime Android 1.26.0 |
| Runtime natywny | NCNN `20260526`, CPU przez JNI |
| ABI | `arm64-v8a`, `armeabi-v7a` |
| Format wymiany | ZIP z rozszerzeniem `.alprmodel` |
| Schemat modelu | `alpr.model.v1` |
| Schemat kompletu | `alpr.package.v1` |
| Schemat raportu | `alpr.mobile_benchmark_report.v1` |
| Schemat pełnego eksportu badawczego | `alpr.mobile_research_bundle.v1` |

Backend NCNN jest wykonywalny na urządzeniu przez bibliotekę JNI
`libalpr_ncnn.so`. Pierwsza wersja runtime'u NCNN korzysta wyłącznie z CPU;
Vulkan pozostaje wyłączony do czasu osobnego benchmarku stabilności,
wydajności, temperatury i zużycia energii.

Aplikacja potrafi aktywować kompletny pakiet `MP+MT+MZ`, w którym poszczególne
etapy korzystają z różnych runtime'ów i precyzji, np.:

```text
MP / NCNN / FP32
MT / TFLite / INT8
MZ / TFLite / INT8
```

Każdy etap zachowuje własny kontrakt wejścia, wyjścia, layoutu tensora,
progów, wariantu wykonawczego i autotuningu.

Warstwa kamery zawiera również opcjonalny auto-zoom oparty na `CameraControl`.
Funkcja jest sterowana przez `AutoZoomController`, blokuje pojedynczy cel przez
`AutoZoomTargetLock`, utrzymuje tekst w `AutoZoomRecognitionMemory` i korzysta z
osobnej generacji transformacji kamery, aby kontrolowanego zoomu nie traktować
jak zmiany logicznej sceny.

---

## 4. Podział kodu

| Pakiet | Odpowiedzialność |
| --- | --- |
| `camera` | uruchomienie CameraX i dostarczanie klatek |
| `model` | manifesty, walidacja ZIP, rejestr i aktywacja modeli |
| `inference` | wspólne API backendów TFLite, ONNX i NCNN |
| `vision` | preprocessing, dekodery YOLO, NMS, geometria, kolejność znaków i rektyfikacja |
| `pipeline` | wykonanie MP → MT → rektyfikacja → MZ, tracking i stabilizacja wyniku |
| `tracking` | lekki tracker ruchu ramek |
| `autotune` | porównanie wariantów i profili wykonania na urządzeniu |
| `metrics` | ślady inferencji, statystyki i raport benchmarkowy |
| `ui` | overlay detekcji, galeria cropów, prezentacja aktywnego kompletu |

`MainActivity` pełni rolę koordynatora UI. Logika manifestów, inferencji,
autotuningu, trackingu, agregacji temporalnej i raportowania pozostaje w
wyspecjalizowanych klasach.

---

# 5. Chronologia budowy

## 2026-08-18 — pierwsza zarejestrowana wersja kompletnego klienta

Commit bazowy:

```text
2696fbf Implement mobile ALPR pipeline and model runtimes
```

Zakres:

- skonfigurowano projekt Android i zależności ML;
- dodano podgląd i analizę klatek przez CameraX;
- utworzono wspólne API backendów inferencji;
- wdrożono LiteRT/TFLite na CPU i GPU;
- wdrożono ONNX Runtime na CPU;
- dodano kontrakt pojedynczego pakietu `alpr.model.v1`;
- dodano bezpieczne rozpakowywanie ZIP do prywatnego katalogu aplikacji;
- dodano walidację manifestu, ról, zadań, plików i SHA-256;
- utworzono lokalny rejestr modeli dla ról `vehicle`, `plate` i `character`;
- zbudowano preprocessing letterbox na podstawie manifestu;
- dodano obsługę wejść `NHWC` i `NCHW` oraz typów zmiennoprzecinkowych i
  kwantyzowanych;
- wdrożono dekoder surowego wyjścia YOLO bez NMS w grafie;
- dodano NMS po stronie Androida;
- dla MT wdrożono odczyt czterech keypointów i rektyfikację perspektywy;
- dla MZ wdrożono sortowanie znaków w kolejności odczytu;
- dodano opcjonalny pierwszy etap detekcji pojazdu;
- dodano stabilizację wyniku między kolejnymi klatkami;
- dodano adaptacyjne ograniczanie liczby analizowanych klatek przy presji
  pamięci i throttlingu termicznym;
- wdrożono autotuning wariantów runtime i liczby wątków;
- dodano pomiary czasów etapów, pamięci i eksport raportu ZIP;
- przygotowano testy jednostkowe geometrii, dekodera, kolejności znaków,
  statystyk, stabilizacji i archiwum raportu.

Decyzja architektoniczna:

Manifest jest źródłem prawdy. Aplikacja nie odgaduje rozmiaru wejścia, layoutu,
normalizacji, liczby klas, liczby keypointów ani progów z nazwy pliku.

---

## 2026-08-19 — kompletny pakiet ALPR MT+MZ

Problem:

Pojedynczy `alpr.model.v1` opisuje jeden model, natomiast gotowy system ALPR
wymaga co najmniej pary MT+MZ. Oddzielny import obu plików zwiększał ryzyko
pomylenia ról lub aktywowania niespójnej pary.

Rozwiązanie:

- dodano nadrzędny schemat `alpr.package.v1`;
- dodano struktury `AlprPackageManifest`, `AlprPackageModelEntry` i
  `PipelineStage`;
- dodano rozpoznawanie pojedynczego modelu i kompletnego pakietu;
- każdy zagnieżdżony model przechodzi pełną walidację `alpr.model.v1`;
- sprawdzane są role, zadania, `model_id`, manifesty boczne i sumy SHA-256;
- aktywacja MT i MZ odbywa się wspólnie;
- zachowano możliwość importu pojedynczego modelu jako trybu częściowego;
- pipeline wykonawczy wykorzystuje sekwencję MT → rektyfikacja → MZ.

Decyzja:

Najpierw walidowane są wszystkie wymagane modele. Dopiero potem komplet może
zostać oznaczony jako gotowy i aktywowany.

---

## 2026-08-19 — rozdzielenie parametrów MT i MZ

Kompletny pakiet nie oznacza wspólnej konfiguracji inferencji.

Dla MT i MZ niezależnie rozwiązywane są:

- rozmiar wejścia;
- layout i typ danych;
- skala i offset;
- `confidence_threshold` i `iou_threshold`;
- dekoder;
- liczba klas i keypointów;
- runtime, precyzja, delegat i liczba wątków;
- etykiety klas.

Poprawny jest więc np. komplet:

```text
MT 640×640 / TFLite GPU
MZ 416×416 / ONNX CPU
```

UI zostało zmienione tak, aby prezentować rzeczywiście wybrany wariant i jego
parametry, a nie wyłącznie wartości domyślne manifestu.

---

## 2026-08-19 — polityka FP32 i INT8

Problem:

Pierwszy autotuner wybierał wariant głównie według opóźnienia. Mogło to
automatycznie promować INT8 bez znajomości wpływu kwantyzacji na jakość.

Rozwiązanie:

- autotuner może mierzyć INT8;
- jeżeli istnieje wykonywalny FP32, automat preferuje FP32;
- fallback preferuje TFLite FP32, potem ONNX FP32;
- INT8 pozostaje kandydatem wymagającym bramki jakościowej.

Decyzja badawcza:

Szybszy wariant nie jest automatycznie lepszy. Zysk czasu i pamięci należy
zestawić z CER, exact match i metrykami detekcji.

---

## 2026-08-19 — ujednolicenie raportu z aplikacją desktopową

Raport Androida otrzymał schemat:

```text
alpr.mobile_benchmark_report.v1
```

Raport obejmuje m.in.:

- `package_id` i identyfikator kombinacji wariantów;
- osobne rekordy wykonania MT i MZ;
- identyfikatory modeli i fingerprinty;
- runtime, precyzję, delegata i liczbę wątków;
- efektywne wejścia i progi;
- p50, p90 i p95 etapów;
- pamięć procesu i rozmiar pakietu;
- dane urządzenia i Androida;
- profile autotuningu;
- statusy i błędy runtime;
- ślady klatek;
- CSV z jednym wierszem na przetworzoną klatkę.

Metryki jakościowe nie są wyliczane bez ground truth.

Decyzja:

Confidence modelu nie jest dokładnością. Raport bez ground truth może
wiarygodnie opisywać czas i zasoby, ale nie końcową skuteczność ALPR.

---

## 2026-08-20 — decyzja badawcza: adaptacyjna kaskada wideo MT–MZ

Problem:

Pełna sekwencja MT → rektyfikacja → MZ wykonywana dla każdej klatki powoduje
wiele kosztownych wywołań MZ dla niemal identycznych cropów.

Rozważono:

- pełne MT+MZ dla każdej klatki;
- oczekiwanie na sztywne `N` klatek;
- wybór dokładnie jednej klatki;
- tracker + adaptacyjne `WhenToRead`;
- uczony model temporalny.

Przyjęto jako kierunek docelowy:

```text
klatka
  -> MT
  -> deduplikacja
  -> tracking
  -> ocena jakości cropu
  -> MZ tylko dla pierwszego dobrego lub wyraźnie lepszego cropu
  -> ważony konsensus znaków
  -> early exit po stabilizacji
```

Wniosek:

Tracker ma pośredniczyć pomiędzy detektorem tablic a MZ. MZ nie powinien
pracować na każdej klatce bezwarunkowo.

Literatura wykorzystana do uzasadnienia:

- C. Zhang, Q. Wang, X. Li, „V-LPDR: Towards a unified framework for license
  plate detection, tracking, and recognition in real-world traffic videos”,
  *Neurocomputing*, 449, 189–206, 2021, DOI `10.1016/j.neucom.2021.03.103`;
- G. R. Gonçalves, D. Menotti, W. R. Schwartz, „License Plate Recognition
  Based on Temporal Redundancy”, ITSC 2016, DOI `10.1109/ITSC.2016.7795970`;
- Q. H. Che, T. D. Thanh, C. T. Van, „Character Time-series Matching for
  Robust License Plate Recognition”, MAPR 2022,
  DOI `10.1109/MAPR56351.2022.9924897`;
- Q. Wang i in., „LSV-LP: Large-Scale Video-Based License Plate Detection and
  Recognition”, *IEEE TPAMI*, 2023, DOI `10.1109/TPAMI.2022.3153691`;
- N. Aharon, R. Orfaig, B.-Z. Bobrovsky, „BoT-SORT: Robust Associations
  Multi-Pedestrian Tracking”, arXiv `2206.14651`.

---

## 2026-08-20 — wdrożenie adaptacyjnego pipeline'u MT–MZ

Rozwiązanie:

- dodano `DetectionDeduplicator`;
- dodano `PlateQualityScorer`;
- odrzucane są zdegenerowane czworokąty i słaba geometria;
- dodano `CharacterSequencePostProcessor`;
- słabsi kandydaci znaków mogą zostać odzyskani z tego samego tensora MZ;
- dodano `PlateTrackCoordinator`;
- dodano `TemporalCharacterAggregator`;
- stabilny track przestaje wykonywać MZ;
- niestabilny track może później wykonywać rzadsze próby okresowe;
- zmiana profilu resetuje stan czasowy.

Profile początkowe:

| Profil | Budżet MZ | Minimalny fit MT | Poprawa jakości | Próba okresowa |
| --- | ---: | ---: | ---: | ---: |
| Szybki | 2 | 0,45 | 0,08 | 10 klatek |
| Zrównoważony | 4 | 0,35 | 0,04 | 6 klatek |
| Dokładny | 6 | 0,25 | 0,02 | 4 klatki |

Weryfikacja:

- `gradlew testDebugUnitTest` — sukces;
- dodano testy geometrii, deduplikacji, tablic dwurzędowych, konsensusu
  temporalnego i zatrzymania MZ po stabilizacji;
- `gradlew lintDebug` — sukces;
- `gradlew assembleDebug` — sukces.

---

## 2026-08-20 — menu komunikacji użytkownika i uproszczenie ekranu kamery

Rozwiązanie:

- dodano `MaterialToolbar`;
- konfigurację i diagnostykę przeniesiono z głównego kadru do menu;
- dodano ręczne czyszczenie bieżącego śledzenia;
- dodano pomoc i opis ról MT/MZ;
- zmiana profilu zeruje tracker i konsensus.

Decyzja:

Główny ekran ma pozostać skoncentrowany na obrazie kamery i wynikach, a
konfiguracja ma być odsunięta do osobnych kontrolek.

---

## 2026-08-20 — decyzja projektowa: rozdzielczość źródła i kaskada ROI

Problem:

Mała tablica może tracić szczegóły przy skalowaniu całej sceny do wejścia MT.
Samo zwiększanie rozdzielczości kamery podnosi jednak koszt pamięci i
preprocessingu.

Przyjęty kierunek:

```text
kamera
  -> opcjonalny lekki MP
  -> wybór maksymalnie 1–2 ROI pojazdów
  -> MT na ROI
  -> rektyfikacja
  -> MZ
```

Założenia początkowe:

- MP co kilka analizowanych klatek;
- ROI poszerzane o margines;
- okresowy fallback MT na pełnej klatce;
- wysoka rozdzielczość ma sens głównie wtedy, gdy później analizowane jest
  ciaśniejsze ROI;
- profile rozdzielczości powinny być jawne i powtarzalne w benchmarkach.

Pamięć samego bufora RGBA:

| Rozdzielczość | Bufor |
| --- | ---: |
| 640×480 | ok. 1,23 MB |
| 1280×720 | ok. 3,69 MB |
| 1920×1080 | ok. 8,29 MB |

Literatura dla decyzji o ROI:

- R. Laroca i in., „A Robust Real-Time Automatic License Plate Recognition
  Based on the YOLO Detector”, IJCNN 2018,
  DOI `10.1109/IJCNN.2018.8489629`;
- K. Khan i in., „Performance Enhancement Method for Multiple License Plate
  Recognition in Challenging Environments”, EURASIP JIVP 2021,
  DOI `10.1186/s13640-021-00572-4`;
- S. M. Silva, C. R. Jung, „License Plate Detection and Recognition in
  Unconstrained Scenarios”, ECCV 2018,
  DOI `10.1007/978-3-030-01258-8_36`;
- S. Ren i in., „Faster R-CNN: Towards Real-Time Object Detection with Region
  Proposal Networks”, IEEE TPAMI 2017, DOI `10.1109/TPAMI.2016.2577031`;
- J. Redmon i in., „You Only Look Once: Unified, Real-Time Object Detection”,
  CVPR 2016, DOI `10.1109/CVPR.2016.91`.

Wniosek:

ROI może ograniczyć tło i zwiększyć względną skalę tablicy, ale dodatkowy MP
nie jest automatycznie optymalizacją. Jego koszt musi być zmierzony w pełnym
pipeline.

---

## 2026-08-20 — pierwsze wdrożenie szybkiego wyniku i kaskady ROI

Rozwiązanie:

- pierwszy kompletny wynik MZ jest prezentowany jako wstępny;
- konsensus temporalny nadaje mu później stan potwierdzony;
- dodano ocenę ostrości opartą na wariancji Laplasjanu;
- MP wybiera maksymalnie dwa ROI;
- startowy margines ROI: 18%;
- MP odświeżany co trzy analizowane klatki;
- brak tablicy w ROI powoduje fallback MT na pełnej klatce;
- dodatkowy pełnoklatkowy fallback co 15 analizowanych klatek;
- raport zapisuje czas pierwszego wyniku wstępnego i potwierdzonego.

Decyzja:

Wynik wstępny nie może być ukrywany przez dodatkową stabilizację UI, ale nie
może być również przedstawiany jako wynik potwierdzony.

---

## 2026-08-20 — profile kamery, YUV i reakcja na szybki ruch

Rozwiązanie:

- profile rozdzielczości:
  - Auto;
  - Szybkość 640×480;
  - Daleki plan 1920×1080;
- raport zapisuje rozdzielczość żądaną i rzeczywistą;
- dodano monitor żyroskopu;
- przy szybkim ruchu MP może być odświeżany częściej;
- margines ROI rośnie dynamicznie;
- CameraX przełączono na `YUV_420_888`;
- bitmapę tworzy natywna ścieżka CameraX.

Decyzja:

Rozdzielczość jest parametrem sesji, a nie klatki. Adaptacja między klatkami
powinna odbywać się głównie przez tracking, ROI i harmonogram modeli.

---

## 2026-08-20 — trwała galeria cropów tablic

Rozwiązanie:

- wynik tekstowy zastąpiono galerią cropów;
- `MobileAlprEngine` przekazuje zrektyfikowany crop tablicy;
- `PlateObservation` przenosi track, tekst, confidence, stan stabilizacji i
  pozycje znaków;
- `PlateCropView` rysuje znaki i ich ramki;
- historia wyników nie znika po pojedynczej pustej klatce;
- wprowadzono jawne zarządzanie własnością bitmap.

Decyzja:

Do galerii przechowywane są małe zrektyfikowane obrazy tablic, a nie pełne
klatki kamery.

---

## 2026-08-20 — sterowana sesja cropów i trwały zapis

Rozwiązanie:

- `Start/Stop` steruje tylko kolektorem cropów, nie inferencją;
- sesja ma własne `session_id`;
- dodano limity `Auto/10/25/50/100`;
- kolektor zapisuje m.in.:
  - pierwszy crop tracku;
  - zmianę tekstu;
  - pierwsze potwierdzenie;
  - istotną poprawę ostrości;
  - próbkę okresową;
- `CropInferenceTiming` przechowuje czasy per crop;
- zapis trwały używa JPEG + JSON;
- wybrany katalog korzysta ze scoped storage;
- ekran pozostaje aktywny podczas sesji.

Weryfikacja:

- testy limitów i polityki próbkowania — sukces;
- `gradlew testDebugUnitTest`, `lintDebug`, `assembleDebug` — sukces;
- test na fizycznym Samsungu potwierdził zbieranie cropów.

---

## 2026-08-20 — osobne ekrany Opcje i Diagnostyka

Rozwiązanie:

- dodano `SettingsActivity`;
- dodano `DiagnosticsActivity`;
- import modeli, profile i kompozycja znajdują się w Opcjach;
- Diagnostyka pokazuje urządzenie, pamięć, modele, runtime i log;
- eksport raportu jest dostępny z Diagnostyki;
- dodano wektorowe ikony i ciemny motyw;
- konfiguracja ma licznik rewizji.

Weryfikacja:

- `gradlew testDebugUnitTest lintDebug assembleDebug` — sukces;
- brak błędów lint;
- aplikacja zainstalowana na fizycznym urządzeniu.

---

## 2026-08-20 — lekki tracker i bezkolizyjne badge’e

Rozwiązanie:

- uproszczono wizualnie ramki;
- rozdzielono kolor nazwy detekcji i confidence;
- badge jest rozmieszczany poza ramkami, gdy istnieje wolne miejsce;
- predykcja bez świeżej obserwacji jest rysowana linią przerywaną;
- `OverlayItem` otrzymał `trackId` i informację o predykcji;
- kosztowne obliczenia geometrii przeniesiono poza `onDraw()`.

Weryfikacja:

- testy jednostkowe — sukces;
- lint bez błędów i bez `DrawAllocation`.

---

## 2026-08-20 — zaznaczanie zbiorcze i płynniejszy tracker

Rozwiązanie:

- checkbox cropu oznacza wyłącznie wybór;
- dodano `Zaznacz wszystkie` i `Zapisz (N)`;
- zapis partii odbywa się sekwencyjnie;
- tracker ma krótsze okno predykcji;
- zmniejszono nadmierne reagowanie na drobny jitter;
- wprowadzono adaptacyjną korekcję pozycji.

Weryfikacja:

- dodano test `smoothsSmallFrameToFrameJitter`;
- testy, lint i build — sukces.

---

## 2026-08-20 — wdrożenie eksportu badawczego i ground truth

Rozwiązanie:

- cztery stany `human_verification`:
  - `not_reviewed`;
  - `accepted`;
  - `rejected`;
  - `corrected`;
- ocena użytkownika nie zmienia confidence ani wyniku inferencji;
- `MetricsCollector` liczy exact match i CER wyłącznie dla rekordów z ground
  truth;
- jednostką jakościową jest unikalny track, nie pojedyncza klatka;
- importer zachowuje źródłowy `source.alprmodel`;
- dodano strumieniowy `ResearchArchive`;
- `.alprsession` zawiera raport, trace'y, log, manifesty, modele i cropy;
- `thesis_bundle.zip` zawiera:
  - `summary.tex`;
  - tabele TeX;
  - `references.bib`;
  - metadane;
  - wybrane cropy.

Weryfikacja:

- testy Levenshteina i escapowania TeX — sukces;
- testy JVM i instrumentalne — sukces;
- test urządzeniowy potwierdził strukturę ZIP;
- test jakości potwierdził obliczenie exact match i CER.

---

## 2026-08-20 — strategia testowania paczki modeli

Utworzono `docs/model_package_test_strategy.md`.

Najważniejsze zasady:

- jednostką wdrożeniową jest kompletny pakiet;
- MT i MZ testuje się również osobno w celu lokalizacji błędu;
- warianty runtime danego modelu muszą pochodzić z tego samego checkpointu;
- testy jakości, wydajności i throttlingu są osobnymi eksperymentami;
- confidence nie zastępuje ground truth;
- `final_test` nie służy do strojenia;
- bramki twarde poprzedzają ranking;
- parytet Python–Android należy sprawdzać warstwowo;
- INT8 wymaga osobnej bramki jakości.

Zdefiniowano poziomy testów T0–T10 i bramki G0–G9.

---

## 2026-08-20 — model pojazdów: konwersja desktopowa, import mobilny

Decyzja:

Konwersja `.pt` nie powinna odbywać się na telefonie. Android ma otrzymywać
gotowy `alpr.model.v1` z:

```text
role: vehicle
task: detect
```

Telefon ma walidować, przechowywać i uruchamiać gotowe artefakty.

Założenia startowe kandydata MP:

- rodzina YOLO `n`;
- lekkie wejście;
- FP32 jako wariant referencyjny;
- INT8 dopiero po bramce jakości;
- maksymalnie dwa ROI;
- obowiązkowy fallback MT na pełnej klatce.

---

## 2026-08-20 — obsługa kompletnej paczki MP+MT+MZ

Rozszerzono `alpr.package.v1` o opcjonalny model `vehicle`.

Warianty:

```text
historyczny:
MT + MZ

pełny:
MP + MT + MZ
```

Importer:

- waliduje MP tak samo jak MT/MZ;
- sprawdza jego manifest i SHA-256;
- zapisuje `vehicle_storage_id`;
- aktywuje wszystkie zawarte modele jedną transakcją;
- zachowuje dokładny źródłowy `source.alprmodel`.

Weryfikacja:

- testy JVM — sukces;
- testy instrumentalne na Samsungu — sukces;
- potwierdzono kompatybilność zarówno pakietu MT+MZ, jak i MP+MT+MZ.

---

## 2026-08-21 — trwała, zwijana galeria przy obrocie ekranu

Rozwiązanie:

- galeria ma jawny stan `Pokaż/Ukryj`;
- pusta galeria pokazuje komunikat;
- `CaptureGalleryViewModel` zachowuje stan podczas zmiany konfiguracji;
- cropy nie są zwalniane przy zwykłym obrocie;
- stan rozwinięcia jest zachowywany;
- layout pozostaje użyteczny także w orientacji poziomej.

Weryfikacja:

- testy jednostkowe, instrumentalne, lint i build — sukces;
- wizualny test na urządzeniu — sukces.

---

## 2026-08-21 — dwuslotowy pasek i maksymalizacja galerii

Rozwiązanie:

- najnowszy crop jest pierwszy;
- standardowy pasek pokazuje dwa sloty;
- użytkownik może przewijać historię bez automatycznego „wyrywania” pozycji;
- tryb maksymalny rozwija galerię;
- stan maksymalizacji jest zachowywany przy obrocie.

Weryfikacja:

- test adaptera potwierdził kolejność `najnowszy → starsze`;
- testy na SM-A125F — sukces.

---

## 2026-08-21 — kompozycja uruchomieniowa modeli

Problem:

Kompletny pakiet nie wystarczał do eksperymentów, w których trzeba podmieniać
pojedynczy model lub runtime.

Rozwiązanie:

- wprowadzono trwałą kompozycję uruchomieniową;
- pakiet bazowy pozostaje niemutowalny;
- MP, MT i MZ mogą wskazywać zainstalowane modele;
- dla każdej roli przewidziano wybór wariantu `auto` albo `pinned`;
- zmiana kompozycji odtwarza silnik i czyści tracking;
- raport rozróżnia pakiet bazowy i lokalną kompozycję.

Decyzja:

Eksperymentalna konfiguracja urządzenia nie może modyfikować źródłowego
artefaktu `.alprmodel`.

---

## 2026-08-21 — degradacja pakietu z MP tylko w NCNN

Problem:

Pierwszy rzeczywisty pakiet zawierał:

```text
MP: NCNN FP32
MT: TFLite INT8
MZ: TFLite INT8
```

Przed wdrożeniem backendu NCNN Android nie mógł uruchomić MP.

Rozwiązanie:

- brak wykonywalnego opcjonalnego MP nie blokuje MT+MZ;
- pakiet pozostaje bazą;
- MP może być chwilowo nieaktywny;
- późniejsza podmiana tylko MP nie wymaga przebudowy całego kompletu.

Weryfikacja:

- import rzeczywistego pakietu — sukces;
- MT i MZ zostały aktywowane.

---

## 2026-08-21 — graf kompozycji i porządkowanie Opcji

Rozwiązanie:

- dodano klikalny graf:

```text
MP → MT → MZ
```

- węzły pokazują runtime i precyzję;
- z poziomu węzła można zmienić model lub rozpocząć import;
- import z węzła sprawdza zgodność roli;
- nazwy profili rozdzielczości zostały doprecyzowane;
- uproszczono obsługę limitu galerii.

Weryfikacja:

- kontrola wizualna na SM-A125F — sukces;
- testy instrumentalne — sukces.

---

## 2026-08-21 — backend NCNN na Androidzie

Problem:

Rzeczywisty MP był dostępny wyłącznie jako `ncnn-fp32`.

Rozwiązanie:

- dodano oficjalny NCNN `20260526`;
- dodano `NcnnBackend`;
- dodano JNI `libalpr_ncnn.so`;
- wspierane ABI:
  - `arm64-v8a`;
  - `armeabi-v7a`;
- pierwszy backend NCNN jest CPU-only;
- runtime korzysta z jednego tensora wejściowego i jednego głównego wyjścia;
- NCNN został zintegrowany z:
  - fabryką backendów;
  - aktywacją modeli;
  - kompozycją;
  - autotuningiem.

Weryfikacja:

- test JNI `Input → Reshape` na SM-A125F — sukces;
- rzeczywisty pakiet 19,41 MB aktywowany;
- graf:

```text
MP / NCNN / FP32
MT / TFLITE / INT8
MZ / TFLITE / INT8
```

- realny MP wykonał autotuning;
- wybrano 4 wątki CPU;
- pełny zestaw testów instrumentalnych — sukces;
- build obu ABI — sukces.

Pomiary MP/NCNN na urządzeniu wykazały silną zależność czasu od liczby wątków;
4 wątki były najszybszym z badanych profili.

---

## 2026-08-21 — sterowanie kaskadą bezpośrednio w grafie modeli

Rozwiązanie:

- pod węzłami MP/MT/MZ dodano badge stanu;
- MP może być `WŁ./WYŁ.` bez usuwania modelu;
- MT i MZ są obowiązkowe i pokazują `WŁ.` albo `BRAK`;
- usunięto zdublowany przełącznik kaskady;
- całkowite usunięcie MP nadal odbywa się przez wybór `Brak`.

Weryfikacja:

- test ręczny `WYŁ. → WŁ. → WYŁ.` — sukces;
- testy jednostkowe, lint, build i testy instrumentalne — sukces.

---

## 2026-08-22 — diagnostyka rzeczywistej kaskady MP→MT→MZ, niezależne sceny testowe i analiza kolejności odczytu

### Problem

- rzeczywisty pakiet `MP+MT+MZ` uruchamiał model pojazdów MP przez NCNN, ale
  początkowo na podglądzie nie pojawiała się poprawna ramka pojazdu ani ROI
  przekazywany do MT;
- logi MP wskazywały confidence znacznie wykraczające poza zakres `0..1`,
  błędne współrzędne i `regions=0`;
- brakowało wizualizacji pokazującej, czy MT pracuje na pełnej klatce, czy ROI;
- przy prezentowaniu kolejnych obrazów na monitorze tracker i konsensus MZ
  mogły zachowywać stan poprzedniej sceny;
- poprzedni numer mógł zostać przypisany do nowej tablicy o podobnej pozycji;
- po dodaniu ramek `VEHICLE` i `VEHICLE_ROI` ujawnił się problem
  rozmieszczania etykiety tekstowej tablicy;
- testy tablic dwurzędowych wskazały potrzebę porównania mobilnej kolejności
  znaków z aplikacją Python.

### Rozwiązanie — wizualizacja faktycznie wykonywanej kaskady

`OverlayItem` rozróżnia:

```text
PLATE
VEHICLE
VEHICLE_ROI
```

`VehicleRoiSelector.Region` zachowuje źródłową detekcję pojazdu. Dzięki temu
`MobileAlprEngine` może pokazać jednocześnie:

- ramkę obiektu MP;
- poszerzony ROI MP→MT;
- ramkę tablicy MT.

`CameraMotionOverlayTracker` śledzi wyłącznie `PLATE`. Ramki diagnostyczne
`VEHICLE` i `VEHICLE_ROI` są przekazywane bez trackera.

Przyjęto zasadę:

> Overlay ma prezentować operacje rzeczywiście wykonane w bieżącej klatce,
> a nie jedynie modele obecne w konfiguracji.

### Rozwiązanie — diagnoza błędnego dekodowania MP/NCNN

Rzeczywisty model pojazdów zwrócił:

```text
OUTPUT shape=[1, 84, 8400]
decoder=ultralytics_detect_raw_v1
classCount=80
hasObjectness=false
```

Przed poprawką Android rozwiązywał:

```text
channelsFirst=false
```

mimo rzeczywistego układu:

```text
[1, C, N] = [1, 84, 8400]
```

Skutki:

- confidence np. `632`, `144`, `54`;
- błędne boxy;
- zapadanie ramek po mapowaniu letterbox;
- `regions=0`.

Porównanie z desktopowym `mobile_model_exporter.py` wykazało, że eksporter
prawidłowo rozpoznaje `[1,84,8400]` jako `channels_first`.

Błąd znajdował się w:

```text
ModelOutputSpec.asNcnnRawOutput()
```

gdzie wymuszano `channelsFirst=false`.

Poprawka zachowuje rzeczywistą wartość `channelsFirst`.

Po poprawce:

```text
OUTPUT shape=[1, 84, 8400]
decoder=ultralytics_detect_raw_v1
channelsFirst=true
classCount=80
hasObjectness=false
normalized=false
```

Confidence wróciło do zakresu `0..1`, a selektor ROI zaczął zwracać regiony:

```text
MP detections=3, regions=2
MP detections=6, regions=2
```

Wizualnie potwierdzono ramkę pojazdu i ROI MP→MT.

Wniosek:

Warstwa runtime nie może nadpisywać layoutu tensora na podstawie samego typu
runtime'u. Kontrakt wynikający z artefaktu musi zostać zachowany.

### Rozwiązanie — kontrolowany test kamera–monitor

Przyjęto pomocniczy scenariusz:

```text
nieruchomy telefon
        ↓
kamera telefonu
        ↓
monitor w stałym położeniu
        ↓
kolejne obrazy pojazdów z tej samej galerii
```

To jest test `camera-in-the-loop`, nie deterministyczny replay tensora.

W torze pozostają:

- autofocus;
- autoekspozycja;
- parametry kamery;
- charakterystyka monitora;
- odświeżanie ekranu;
- geometria stanowiska.

Dla porównywalności należy utrzymywać:

- tę samą galerię;
- tę samą kolejność obrazów;
- stałą pozycję telefonu i monitora;
- stałą jasność monitora;
- ten sam rozmiar obrazu;
- tę samą konfigurację aplikacji;
- kilka powtórzeń.

Wniosek metodologiczny:

Krótki, sztywny timer zmiany slajdu może faworyzować szybkie konfiguracje.
Wolniejszy pipeline może zobaczyć zbyt mało użytecznych klatek. Należy użyć
odpowiednio długiego czasu sceny albo jawnego warunku zakończenia obserwacji.

### Rozwiązanie — automatyczne rozdzielanie scen

Zmiana obrazu na monitorze nie oznacza automatycznej zmiany tracku. Nowa
tablica o podobnej pozycji może zostać dopasowana do starego tracku i
odziedziczyć stan `TemporalCharacterAggregator`.

Nie osłabiono globalnie trackera.

Dodano:

```text
vision/SceneChangeDetector.java
```

Detektor:

- pobiera siatkę `20×20`, czyli 400 próbek luminancji;
- porównuje kolejne przetworzone klatki;
- kompensuje globalną zmianę średniej jasności;
- wylicza:
  - `score`;
  - `changedFraction`;
  - `brightnessDelta`.

Progi startowe:

```text
SCORE_THRESHOLD = 0.14
CHANGED_FRACTION_THRESHOLD = 0.38
```

Pierwsza wersja z cooldownem klatkowym została odrzucona. Przy wolnym
pipeline'ie kolejne wywołania mogą być oddalone o wiele sekund.

Wprowadzono stan `armed`:

```text
start / nowa scena
        ↓
detektor rozbrojony
        ↓
pierwsza stabilna klatka
        ↓
armed=true
        ↓
duża zmiana obrazu
        ↓
SCENE RESET
        ↓
armed=false
        ↓
oczekiwanie na stabilizację
```

Mechanizm ogranicza również fałszywe resetowanie podczas początkowej zmiany
ekspozycji kamery.

### Kalibracja SceneChangeDetector

Dla stabilnej sceny uzyskano m.in.:

```text
score=0.075 fraction=0.160
score=0.033 fraction=0.065
score=0.025 fraction=0.050
score=0.011 fraction=0.013
```

Dla rzeczywistej zmiany:

```text
frame=16
changed=true
candidate=true
armed=false
score=0.200
fraction=0.550
```

oraz:

```text
RESET frame=16 -> wyczyszczono tracki MT/MZ i cache MP
```

W kolejnym przebiegu:

```text
frame=22
changed=true
candidate=true
armed=false
score=0.213
fraction=0.463
```

Następna klatka nadal była mocno zmieniona:

```text
frame=24
candidate=true
armed=false
score=0.239
fraction=0.512
```

ale nie powodowała ponownego resetu.

Stabilizacja nowej sceny:

```text
frame=26
candidate=false
armed=true
score=0.123
fraction=0.243
```

Później:

```text
frame=32 score=0.011 fraction=0.000
frame=34 score=0.022 fraction=0.045
frame=36 score=0.050 fraction=0.105
```

Wniosek:

Dla badanego stanowiska progi `0.14 / 0.38` dobrze rozdzielają stabilny obraz
od wyraźnej zmiany sceny. Pozostają jednak progami inżynierskimi, nie
wynikiem formalnej kalibracji na dużym zbiorze.

### Zakres resetu sceny

Reset sceny czyści:

- `PlateTrackCoordinator`;
- temporalny konsensus MZ;
- `cachedVehicleRegions`;
- `lastVehicleDetectionFrame`.

Rozdzielono:

```text
resetSceneDependentState()
```

od:

```text
resetTracking()
```

Pełny reset czyści również `SceneChangeDetector`. Automatyczny reset sceny
nie może zerować detektora, ponieważ bieżąca klatka jest już początkiem nowej
sceny.

Informacja:

```text
PipelineResult.sceneReset
```

jest przekazywana do `MainActivity`, aby UI mogło wyczyścić również:

- `CameraMotionOverlayTracker`;
- `lastCaptureByTrack`.

Historia cropów nie jest kasowana.

### Problem etykiety numeru nad tablicą

Analiza `MobileAlprEngine` potwierdziła, że wynik MZ trafia do
`OverlayItem.label`.

Problem znaleziono w `DetectionOverlayView`.

Algorytm rozmieszczania badge'ów traktował jako przeszkody wszystkie ramki:

```text
PLATE
VEHICLE
VEHICLE_ROI
```

Dla struktury:

```text
VEHICLE
└── VEHICLE_ROI
    └── PLATE
```

większość sensownych pozycji napisu tablicy znajduje się wewnątrz większego
ROI albo ramki pojazdu. `findFreeBadge()` może więc zwrócić `null`.

Przyjęty kierunek:

- etykieta `PLATE` powinna unikać innych ramek `PLATE`;
- powinna unikać innych badge'ów;
- `VEHICLE` i `VEHICLE_ROI` nie powinny blokować napisu tablicy.

Pełne potwierdzenie wizualne tej poprawki pozostaje do wykonania.

### Analiza rzędowości tablic i kolejności znaków

Porównano:

```text
desktop:
auto_annotation_tool/character_recognition/reading_order.py

Android:
vision/ReadingOrderResolver.java
vision/CharacterSequencePostProcessor.java
```

Desktopowy algorytm:

- liczy `cx`, `cy` i wysokość każdego znaku;
- wyznacza medianę wysokości;
- używa tolerancji:

```text
max(6 px, median_height * 0.68)
```

- jeżeli cały zakres `centerY` mieści się w tolerancji, jawnie wybiera jeden
  rząd;
- w przeciwnym razie tworzy rzędy;
- przypisuje znak do najbliższego rzędu;
- aktualizuje `center_y` rzędu medianą;
- aktualizuje medianę wysokości per rząd;
- sortuje rzędy góra→dół;
- sortuje znaki w rzędzie lewo→prawo;
- potrafi oznaczyć:
  - `reading_row`;
  - `reading_col`;
  - `reading_index`;
- posiada klasyfikację:
  - `single_row`;
  - `two_row`;
  - `two_row_candidate`;
  - `unknown`.

Mobilny `ReadingOrderResolver`:

- ma zbliżoną podstawę;
- używa jednej globalnej tolerancji;
- środek rzędu liczy średnią;
- nie utrzymuje mediany wysokości per rząd;
- nie wykonuje jawnego testu całego zakresu Y przed grupowaniem.

Większy problem znaleziono w `CharacterSequencePostProcessor`.

Po poprawnym wcześniejszym podziale na rzędy kod może później wykonać:

```java
if (expectedCount > 0 && filtered.size() > expectedCount) {
    filtered = bestCountCandidates(filtered, expectedCount);
}
```

`bestCountCandidates()` ponownie otrzymuje znaki ze wszystkich rzędów i
wyznacza jedną globalną `medianCenterY`.

Dla tablicy dwurzędowej może to karać poprawne znaki obu rzędów za oddalenie
od jednej wspólnej linii.

Kierunek zmian:

1. możliwie wiernie przenieść `reading_order.py` do `ReadingOrderResolver`;
2. nie stosować globalnego `bestCountCandidates()` dla układu wielorzędowego;
3. dodać testy regresji tablic jedno- i dwurzędowych;
4. rozważyć rozszerzenie agregacji temporalnej o strukturę:

```text
layout=TWO_ROW
rowCounts=[3,5]
```

zamiast wyłącznie:

```text
expectedCharacterCount=8
```

Wniosek:

Sama liczba znaków nie opisuje geometrii tablicy dwurzędowej.

### Decyzje z sesji 2026-08-22

- runtime nie może samowolnie zmieniać layoutu tensora;
- overlay pokazuje rzeczywiste wykonanie kaskady;
- kolejne sceny testowe nie mogą dzielić stanu trackera i konsensusu MZ;
- zmiana sceny jest wykrywana na podstawie obrazu;
- mechanizm `armed` jest właściwszy od cooldownu klatkowego;
- historia cropów jest oddzielona od stanu trackera;
- test kamera–monitor jest eksperymentem całego systemu, nie replay samych
  modeli;
- tablice dwurzędowe wymagają zachowania struktury rzędów w późniejszym
  postprocessingu.

### Weryfikacja

- poprawkę `channelsFirst` sprawdzono z rzeczywistym MP/NCNN na SM-A125F;
- rzeczywisty tensor MP: `[1,84,8400]`;
- po poprawce: `channelsFirst=true`;
- confidence MP wróciło do poprawnego zakresu;
- `VehicleRoiSelector` zwraca 1–2 ROI;
- ramka pojazdu i ROI MP→MT są widoczne;
- `ALPR_SCENE` potwierdził stabilną pracę progu;
- ręczna zmiana obrazu wywołuje pojedynczy reset;
- kolejna niestabilna klatka nie wywołuje ponownego resetu;
- detektor uzbraja się po stabilizacji;
- `assembleDebug` wykonywano po zmianach z powodzeniem;
- analiza rzędowości Python↔Android została wykonana;
- nowy port algorytmu rzędowości nie został jeszcze wdrożony.

### Pozostało

- potwierdzić wizualnie pełne czyszczenie starej ramki i tekstu po
  `sceneReset`;
- potwierdzić poprawne rozmieszczenie badge'a tablicy przy aktywnym MP;
- ograniczyć tymczasowe logowanie `ALPR_MP` i `ALPR_SCENE`;
- przenieść ulepszony algorytm rzędowości;
- poprawić globalne `bestCountCandidates()` dla układów wielorzędowych;
- dodać testy regresji tablic jedno- i dwurzędowych;
- ocenić rozszerzenie `TemporalCharacterAggregator` o strukturę rzędów;
- ograniczyć MP do właściwych klas pojazdów;
- wykorzystać kontrakt `vehicle_detection.include_class_indices/include_labels`
  i bezpieczny fallback klas COCO;
- wykonać A/B NCNN, ONNX i TFLite;
- zamrozić protokół eksperymentu kamera–monitor przed wykorzystaniem go w
  pracy inżynierskiej.

---

## 2026-08-23 — filtr klas MP, warianty R0/R1/R2, nadrzędny tryb eksperymentalny i kontrolowana sesja pomiarowa

### Problem

Dalsze wykorzystanie modelu pojazdów MP jako pierwszego etapu kaskady wymagało
rozwiązania kilku niezależnych problemów badawczych i wykonawczych:

- MP zwracał detekcje wszystkich klas dostępnych w modelu COCO, podczas gdy
  tylko część z nich ma sens jako źródło ROI dla tablicy rejestracyjnej;
- stały limit dwóch ROI był zaszyty w pipeline i nie pozwalał wykonać czystego
  eksperymentu porównującego pełną klatkę z jednym i dwoma ROI;
- pierwsza wersja wyboru R0/R1/R2 modyfikowała normalną konfigurację MP, przez
  co po zakończeniu eksperymentu nie było jednoznaczne, czy stan grafu wynika
  z ustawienia użytkownika, czy z wariantu eksperymentalnego;
- kamera uruchamiała się automatycznie wraz z `MainActivity`, więc część
  inferencji i metryk powstawała jeszcze przed przygotowaniem stanowiska;
- czas raportu był związany z życiem `MetricsCollector`, a nie z faktycznym
  przedziałem pomiarowym;
- potrzebny był punkt startowy do późniejszego rozszerzenia eksperymentów o
  opcjonalne ograniczenie czasowe.

### Rozwiązanie — filtr klas modelu pojazdów

W `MobileAlprEngine` dodano filtrowanie wyników MP przed przekazaniem ich do
`VehicleRoiSelector`.

Dla modelu COCO dopuszczane są klasy odpowiadające pojazdom:

```text
car
motorcycle
bus
truck
```

W testowanym manifeście odpowiadały one indeksom:

```text
[2, 3, 5, 7]
```

Jeżeli model ma niestandardową listę klas i nie uda się rozpoznać żadnej klasy
pojazdu, zachowywany jest bezpieczny fallback dopuszczający wszystkie etykiety,
aby nie zablokować modelu wyspecjalizowanego wyłącznie w pojazdach.

Do `InferenceTrace` dodano liczniki:

```text
vehicle_detections_raw
vehicle_detections_used
vehicle_detections_rejected_class
vehicle_regions_selected
```

W kontrolowanym przebiegu diagnostycznym MP zwrócił łącznie:

```text
raw detections       = 110
used after filter    = 37
rejected by class    = 73
```

Oznacza to odrzucenie około 66,4% detekcji przed selekcją ROI.

Po dalszej deduplikacji i ograniczeniu liczby regionów w tym samym przebiegu
wybrano łącznie 25 ROI.

Wniosek:

Filtr klas nie skraca samej inferencji MP, ponieważ działa po otrzymaniu
tensora wynikowego. Może jednak ograniczyć koszt dalszej części kaskady,
ponieważ zmniejsza liczbę kandydatów konkurujących o miejsca w budżecie ROI
oraz ryzyko wykonywania MT dla obiektów niebędących pojazdami.

### Rozwiązanie — jawna polityka budżetu ROI

Wprowadzono `RoiBudgetPolicy` i trzy pierwsze warianty badawcze:

```text
R0 / FULL_FRAME
  MP pomijany
  MT analizuje pełną klatkę

R1 / ONE_ROI
  MP aktywny
  maksymalnie 1 ROI przekazywany do MT

R2 / TWO_ROI
  MP aktywny
  maksymalnie 2 ROI przekazywane do MT
```

Budżet ROI przestał być wyłącznie stałą wewnątrz `MobileAlprEngine`.

Wariant R0 pełni rolę punktu odniesienia dla eksperymentu. R1 i R2 pozwalają
ocenić, czy koszt dodatkowego MP i kolejnych uruchomień MT dla ROI zostaje
zrekompensowany przez ograniczenie analizowanego obszaru i zwiększenie
względnej skali tablicy.

### Rozwiązanie — oddzielenie konfiguracji normalnej od eksperymentalnej

Pierwsza implementacja R0/R1/R2 zapisywała jednocześnie:

```text
vehicle_cascade_enabled
```

co powodowało, że wybór wariantu eksperymentalnego modyfikował normalną
konfigurację aplikacji.

Zmieniono architekturę na dwie niezależne warstwy:

```text
NormalConfig
    +
Experiment override
    ↓
EffectiveConfig
```

Normalna konfiguracja użytkownika przechowuje rzeczywisty stan kaskady MP.
Tryb eksperymentalny jest warstwą nadrzędną i nie zmienia tej wartości.

Przykład:

```text
normalnie:
MP = WŁ.

EXP + R0:
MP efektywnie pomijany

EXP OFF:
MP ponownie WŁ.
```

Analogicznie normalne `MP = WYŁ.` może zostać chwilowo nadpisane przez R1 lub
R2, a po wyłączeniu eksperymentu aplikacja wraca do wcześniejszego stanu.

W Opcjach dodano nadrzędny przełącznik trybu eksperymentalnego. W czasie jego
działania graf MP→MT→MZ pokazuje stan efektywnie wykonywany przez pipeline,
ale normalna konfiguracja MP nie jest edytowana.

Decyzja:

Eksperyment nie jest edytorem normalnej konfiguracji aplikacji. Jest
tymczasową warstwą nadpisującą wyłącznie parametry potrzebne do konkretnego
badania.

### Rozwiązanie — raportowanie konfiguracji normalnej, eksperymentalnej i efektywnej

Raport `.alprsession` rozróżnia obecnie:

```text
normal_configuration
experiment
roi_budget_policy
```

Przykładowa konfiguracja testu R0:

```json
{
  "normal_configuration": {
    "vehicle_cascade_enabled": true
  },
  "experiment": {
    "enabled": true,
    "type": "roi_budget",
    "roi_budget_policy": "r0_full_frame",
    "effective_roi_budget_policy": "r0_full_frame"
  }
}
```

Pozwala to odtworzyć zarówno normalne ustawienia aplikacji, jak i warstwę
użytą tylko podczas konkretnego eksperymentu.

### Weryfikacja R0 na pełnym eksporcie `.alprsession`

Wykonano kontrolny przebieg R0 na fizycznym urządzeniu.

Raport potwierdził:

```text
normal_configuration.vehicle_cascade_enabled = true
experiment.enabled                           = true
experiment.roi_budget_policy                 = r0_full_frame
experiment.effective_roi_budget_policy       = r0_full_frame
```

Dane wykonawcze:

```text
processed_frames         = 51
plate_roi_runs           = 0
plate_full_frame_runs    = 51
full_frame_fallbacks     = 0
dropped_frames           = 52
```

W `traces.csv` nie było żadnego czasu:

```text
vehicle_preprocess
vehicle_inference
vehicle_postprocess
```

co potwierdza, że MP nie został jedynie ukryty w UI, lecz rzeczywiście nie był
wykonywany.

Dla tego przebiegu uzyskano pomocniczy baseline wydajności:

```text
MT p50       ≈ 2978 ms
MT p90       ≈ 2997 ms
MT p95       ≈ 3006 ms

pipeline p50 ≈ 3530 ms
pipeline p90 ≈ 3602 ms
pipeline p95 ≈ 4278 ms
```

Wyniki te są punktem kontrolnym, a nie jeszcze wynikiem formalnego porównania.
Właściwy eksperyment wymaga identycznego materiału, kilku powtórzeń i
porównania R0, R1 oraz R2.

Podczas analizy raportu zauważono również, że statystyki MZ zawierają wpisy
zerowe dla klatek, w których MZ nie został rzeczywiście uruchomiony. Przed
końcową analizą wydajności MZ należy rozdzielić brak wykonania od czasu
inferencji równego zero.

### Rozwiązanie — ręczny Start/Stop analizy

Automatyczne uruchamianie CameraX przy starcie `MainActivity` zostało usunięte.

Ekran główny ma obecnie jawny stan:

```text
analiza zatrzymana
        ↓
[Uruchom analizę]
        ↓
kamera + pipeline
        ↓
[Zatrzymaj analizę]
        ↓
analiza zatrzymana
```

Przycisk `Start/Stop` kolektora cropów zachował osobną odpowiedzialność i nie
steruje inferencją.

Po zatrzymaniu analizy:

- CameraX jest odpinany przez `CameraController.stop()`;
- `cameraStarted=false`;
- zatrzymywany jest monitor ruchu;
- resetowany jest tracking pipeline;
- resetowany jest `CameraMotionOverlayTracker`;
- czyszczony jest bieżący overlay;
- `PreviewView` jest ukrywany, aby nie pozostawiać ostatniej klatki jako
  pozornie aktywnego obrazu;
- historia już zebranych cropów pozostaje zachowana.

Przy kolejnym uruchomieniu analiza startuje bez stanu trackingu z poprzedniego
przebiegu.

Decyzja:

Użytkownik musi jednoznacznie określać moment, w którym aplikacja zaczyna
obciążać urządzenie i przetwarzać klatki. Jest to istotne zarówno dla zwykłego
użycia, jak i dla powtarzalności eksperymentów.

### Rozwiązanie — sesja pomiarowa niezależna od czasu życia aplikacji

`MetricsCollector` wcześniej rozpoczynał pomiar w chwili utworzenia obiektu.
W konsekwencji raport obejmował również czas przygotowania ustawień i
stanowiska przed faktycznym uruchomieniem kamery.

Dodano jawne metody:

```text
startMeasurementSession()
finishMeasurementSession()
```

Rozpoczęcie analizy:

- czyści wcześniejsze trace'y pomiarowe;
- zeruje dropped frames;
- zeruje czasy pierwszego wyniku;
- zapisuje nowe `sessionStartedMillis` i `sessionStartedNanos`;
- aktywuje przyjmowanie trace'ów.

Zatrzymanie analizy:

- zapisuje `sessionFinishedMillis`;
- blokuje przyjmowanie kolejnych trace'ów;
- zachowuje dane zakończonego przebiegu do eksportu.

`add()`, `frameDropped()` i `recordRecognitionState()` respektują stan aktywnej
sesji pomiarowej.

Raport otrzymał:

```text
session_started_ms
session_finished_ms
session_duration_ms
measurement_session_active
```

Weryfikacja na fizycznym urządzeniu:

```text
session_duration_ms = 41718
measurement_session_active = false
```

Log aplikacji wskazywał uruchomienie i zatrzymanie analizy w tym samym
przedziale. Różnica pomiędzy czasem raportu a wpisem końcowym logu wynika z
tego, że sesja pomiarowa jest kończona przed sprzątaniem UI i odpinaniem
CameraX.

Wniosek:

Czas benchmarku jest obecnie związany z rzeczywistym przebiegiem analizy, a nie
z czasem, przez jaki użytkownik miał otwartą aplikację.

### Kierunek architektury eksperymentów

Ustalono rozdzielenie czterech pojęć:

```text
NormalConfig
ExperimentConfig
ExperimentSession
MetricsCollector
```

Znaczenie:

- `NormalConfig` — normalne ustawienia użytkownika;
- `ExperimentConfig` — określa co jest badane, np. wariant R0/R1/R2;
- `ExperimentSession` — ma reprezentować jeden konkretny przebieg od startu
  do zakończenia;
- `MetricsCollector` — zbiera dane pomiarowe z aktywnego przebiegu.

Planowany minimalny `ExperimentSession` ma przechowywać:

```text
session_id
state: IDLE / RUNNING / FINISHED
start
stop
duration
completion_reason
```

Rozważane powody zakończenia:

```text
MANUAL
TIMER
ERROR
```

`ExperimentSession` nie został jeszcze wdrożony.

### Opcjonalny TimerConfig

Timer został uznany za opcjonalny element konfiguracji eksperymentu, a nie za
cechę obowiązkową każdego eksperymentu.

Docelowo:

```text
ExperimentConfig
  ├── wariant badawczy
  ├── TimerConfig?       opcjonalny
  └── przyszłe warunki
```

Bez timera sesja trwa do ręcznego zatrzymania.

Z timerem:

```text
start
  ↓
odliczanie
  ↓
osiągnięcie limitu
  ↓
ExperimentSession.finish(TIMER)
```

Rzeczywisty czas sesji ma być mierzony zawsze, niezależnie od tego, czy timer
ograniczający przebieg został włączony.

Timer i `ExperimentSession` są na tym etapie projektem architektonicznym, a nie
funkcją ukończoną.

### Znaczenie dla pracy inżynierskiej

Zmiany z 2026-08-23 bezpośrednio wspierają dwa fragmenty pracy.

**Rozdział implementacyjny klienta mobilnego:**

- jawne sterowanie CameraX i momentem rozpoczęcia analizy;
- oddzielenie normalnej konfiguracji od eksperymentalnych override'ów;
- jawna polityka budżetu ROI;
- raportowanie efektywnej konfiguracji;
- powiązanie pomiaru z rzeczywistą sesją analizy.

**Rozdział dotyczący analizy wyników:**

- wariant referencyjny R0;
- wariant R1 z jednym ROI;
- wariant R2 z maksymalnie dwoma ROI;
- liczniki liczby detekcji MP, wybranych ROI, fallbacków i uruchomień MT;
- p50, p90, p95 i p99 dla etapów i całego pipeline'u;
- możliwość wykonania kilku powtarzalnych przebiegów o jednoznacznie
  określonym początku i końcu.

Najważniejsze pytanie badawcze:

> Czy koszt dodatkowego modelu pojazdów MP i kolejnych wywołań MT dla ROI
> zostaje zrekompensowany przez analizę ograniczonego obszaru obrazu, a jeżeli
> tak, jaki budżet ROI daje najlepszy kompromis pomiędzy wydajnością i
> skutecznością końcowego ALPR?

### Decyzje z sesji 2026-08-23

- MP filtruje klasy niebędące pojazdami przed wyborem ROI;
- filtr klas nie jest traktowany jako skrócenie czasu inferencji samego MP;
- liczba ROI jest jawnym parametrem eksperymentalnym;
- R0/R1/R2 nie mogą modyfikować normalnej konfiguracji użytkownika;
- tryb eksperymentalny jest warstwą override nad konfiguracją normalną;
- graf UI ma prezentować konfigurację efektywnie wykonywaną;
- raport musi zachować jednocześnie normalny stan i konfigurację
  eksperymentalną;
- kamera nie uruchamia się automatycznie;
- użytkownik jawnie wyznacza moment rozpoczęcia i zakończenia analizy;
- pomiar nie obejmuje czasu przygotowania stanowiska;
- historia cropów jest niezależna od bieżącej sesji analizy;
- timer jest opcjonalnym warunkiem zakończenia przyszłej
  `ExperimentSession`, a nie obowiązkową częścią eksperymentu.

### Weryfikacja

- filtr klas MP sprawdzono na rzeczywistych logach;
- potwierdzono dozwolone klasy COCO `[2,3,5,7]`;
- w analizowanym przebiegu odrzucono 73 z 110 surowych detekcji MP;
- tryb eksperymentalny sprawdzono ręcznie dla powrotu do normalnego stanu MP;
- graf poprawnie pokazuje efektywny stan MP w R0;
- R0 zweryfikowano przez pełny `.alprsession`;
- w R0 potwierdzono 51 uruchomień MT full-frame i 0 uruchomień MT na ROI;
- w R0 nie zarejestrowano żadnej inferencji MP;
- ręczny Start/Stop analizy sprawdzono na urządzeniu;
- po STOP ostatni obraz kamery jest usuwany z podglądu;
- nowy pomiar jest rozpoczynany dopiero przy `Uruchom analizę`;
- kontrolny raport potwierdził sesję pomiarową około 41,7 s;
- kod po zmianach został zbudowany, uruchomiony na fizycznym urządzeniu oraz
  zapisany w repozytorium Git.

### Pozostało

- wdrożyć minimalny `ExperimentSession`;
- dodać opcjonalny `TimerConfig`;
- określić UI timera bez uzależniania wszystkich eksperymentów od czasu;
- zablokować lub oznaczać zmianę kluczowej konfiguracji eksperymentu w aktywnej `ExperimentSession`;
- wykonać kontrolne przebiegi R1 i R2;
- zweryfikować, że `vehicle_regions_selected` nigdy nie przekracza odpowiednio
  1 dla R1 i 2 dla R2;
- wykonać właściwą serię porównawczą R0/R1/R2 na identycznym materiale;
- wykonać kilka powtórzeń każdego wariantu;
- rozdzielić rzeczywiste wywołania MZ od zerowych wpisów czasowych dla klatek,
  w których MZ został pominięty;
- zamrozić końcowy protokół camera-in-the-loop;
- zdefiniować sposób zakończenia pojedynczej sceny w testach skuteczności;
- po zakończeniu warstwy eksperymentalnej ponownie uruchomić pełny zestaw
  regresyjny.

---

## 2026-08-26 — ciągły auto-zoom celu, blokada tablicy i pamięć odczytu MZ

### Problem

Pierwsza wersja auto-zoomu traktowała zbliżenie podobnie do zmiany sceny.
Prowadziło to do niespójnego przebiegu widocznego dla użytkownika:

- ramki potrafiły zniknąć tuż przed rozpoczęciem zoomu albo po powrocie do
  `1x`;
- celownik nie zawsze pozostawał na tablicy, która wywołała zbliżenie;
- świeży wynik MT na zbliżeniu zastępował etykietę numeru tekstem
  `tablica · MT`, mimo że numer został rozpoznany wcześniej;
- MZ mógł analizować inne tablice znajdujące się w widocznym kadrze zamiast
  celu wybranego przed zoomem;
- przy drżeniu telefonu tracker oparty na ruchu potrafił akumulować błąd i
  przesuwać ramkę poza rzeczywisty obszar tablicy;
- kontrolowany zoom-in i zoom-out bywał błędnie interpretowany przez detektor
  sceny jako przejście do nowej sceny i uruchamiał ponowną stabilizację;
- pusty lub słabszy odczyt po zmianie skali usuwał wcześniej widoczny numer.

Wymagany przebieg został określony następująco:

```text
MT wykrywa tablicę i UI pokazuje ramkę
  -> ramka oraz dostępny numer pozostają widoczne
  -> płynny zoom-in z celownikiem na tej samej tablicy
  -> świeże MT aktualizuje geometrię celu
  -> MZ ponawia odczyt wyłącznie dla zablokowanego celu
  -> płynny zoom-out z zachowaniem ramki i najlepszego numeru
  -> powrót do tej samej sceny bez zbędnej ponownej stabilizacji
```

### Rozwiązanie — rozdzielenie geometrii, tożsamości celu i tekstu

Stan prezentacji auto-zoomu rozdzielono na trzy niezależne warstwy:

```text
geometria ramki       <- świeże MT i lekki tracker Preview
tożsamość celu        <- AutoZoomTargetLock
najlepszy numer       <- AutoZoomRecognitionMemory / konsensus MZ
```

Świeże MT może zatem poprawić położenie ramki bez kasowania tekstu. Brak
znaków w bieżącej próbie MZ również nie oznacza już automatycznie utraty
ostatniego sensownego odczytu.

Przed rozpoczęciem zbliżenia `MainActivity`:

- zapamiętuje numer i confidence wybranego tracku;
- zamraża aktualny overlay jako pamięć sceny;
- pokazuje ramkę przed pierwszą zmianą zoomu;
- przechowuje współrzędne celu w przestrzeni sceny sprzed zbliżenia.

Podczas zmiany skali ramki pamięci są transformowane razem z obrazem. Po
otrzymaniu świeżego MT geometria zostaje zastąpiona dokładniejszą ramką, ale
etykieta numeru pochodzi nadal z pamięci MZ, dopóki nowy wynik nie przejdzie
bramki akceptacji.

### Rozwiązanie — blokada pojedynczego celu

Dodano `AutoZoomTargetLock` ze stanami:

```text
DISABLED -> ACQUIRING -> LOCKED -> UNCERTAIN -> LOST
```

Kandydat celu jest oceniany na podstawie:

- przewidywanego położenia i ruchu środka;
- IoU z przewidywaną ramką;
- zgodności skali i proporcji;
- podobieństwa prostego deskryptora wyglądu;
- confidence MT i poprawności geometrii keypointów.

Blokada używa dynamicznego obszaru poszukiwania. Przy stanie niepewnym może go
rozszerzyć, ale odrzuca kandydatów zbyt odległych, o niezgodnej skali albo
niejednoznacznych względem drugiego najlepszego dopasowania. Brak pewnego
dopasowania nie powoduje natychmiastowego przełączenia na inną tablicę.

Aktywna blokada ogranicza obszar przekazywany do MT/MZ. Inne tablice widoczne
w całym kadrze nie powinny uczestniczyć w ponownej próbie MZ celu auto-zoomu.

Celownik jest aktualizowany ze świeżej geometrii MT albo kontrolowanej
predykcji trackera. Współrzędne są przeliczane pomiędzy przestrzenią sceny,
aktualnym zoomem i `PreviewView`, dzięki czemu celownik ma pozostać związany z
tą samą tablicą podczas zoom-in, oczekiwania na MZ i zoom-out.

### Rozwiązanie — ograniczenie dryfu trackera Preview

Dodano `PreviewTrackerDriftGuard`, który porównuje:

- mały ruch przyrostowy pomiędzy sąsiednimi klatkami;
- niezależny ruch bezwzględny względem ostatniej ramki zakotwiczonej przez MT.

Korekta jest przyjmowana tylko przy wystarczającym wsparciu punktów. Zbyt duży
jednorazowy skok, nadmierny offset skumulowany albo niezgodność estymacji
przyrostowej z kotwicą MT unieważnia aktualizację zamiast przesuwać ramkę poza
tablicę. Tracker Preview podtrzymuje overlay pomiędzy ciężkimi inferencjami,
ale nie zastępuje ponownej detekcji MT.

### Rozwiązanie — zoom jako transformacja tej samej sceny

Kontrolowana zmiana zoomu otrzymała własną generację transformacji kamery.
Wyniki rozpoczęte dla poprzedniej skali są odrzucane bez czyszczenia całej
logicznej sceny.

Przed zoomem i po ustabilizowaniu zbliżenia przechowywane są osobne kotwice
obrazu. Podczas samej transformacji detektor zmiany sceny jest maskowany. Po
zoom-out obraz jest porównywany z kotwicą sprzed zoomu:

- niewielka różnica oznacza powrót do tej samej sceny i zachowanie ramek oraz
  wyniku;
- duża różnica oznacza rzeczywistą zmianę kadru i uruchamia normalny reset;
- dynamiczny ruch kamery może być chwilowo kompensowany, jeżeli tracker nadal
  jednoznacznie utrzymuje wybrany cel.

### Rozwiązanie — trwała pamięć numeru podczas całego cyklu

Dodano `AutoZoomRecognitionMemory`. Numer rozpoznany przed zbliżeniem jest
inicjalizacją pamięci i nie jest już zerowany w chwili rozpoczęcia zoomu.

Pusty wynik MT/MZ nie kasuje numeru. Nowy tekst może zastąpić pamięć, jeżeli:

- po normalizacji ma od 4 do 10 znaków alfanumerycznych;
- jest potwierdzony temporalnie albo jego confidence wynosi co najmniej
  `0,45`;
- dla tekstu różnego od pamięci jest potwierdzony i nie jest istotnie słabszy
  albo ma confidence co najmniej `0,72` i przewagę co najmniej `0,10`.

Ten sam tekst może podnieść zapamiętane confidence, ale słabsza obserwacja nie
obniża wcześniej osiągniętej wartości. Dopasowanie wyniku odbywa się do ramki
celu, a nie do dowolnej tablicy w kadrze.

Pierwszy sensowny wynik MZ po powrocie do `1x` może jeszcze zaktualizować
numer. Do tego czasu UI pokazuje ostatni zaakceptowany tekst na aktualizowanej
ramce celu.

Szczególny przypadek pozostaje obsłużony:

```text
MT wykrywa tablicę, ale nie ma żadnego tekstu
  -> zoom-in nadal jest dozwolony
  -> świeży crop po zbliżeniu dostaje wymuszoną próbę MZ
  -> pierwszy sensowny odczyt MZ inicjalizuje pamięć numeru
```

### Doprecyzowanie — co oznacza `confirmed` przy zapisanym cropie

Pole `confirmed` nie opisuje jakości samego pliku cropa i nie oznacza ręcznej
weryfikacji numeru. Jest kopią stanu `stable` temporalnego konsensusu MZ dla
całego tracku tablicy.

`TemporalCharacterAggregator` rozdziela obserwacje według struktury tablicy,
czyli liczby wierszy i liczby znaków w każdym wierszu. Wynik staje się
`stable=true`, gdy:

- dominująca struktura ma co najmniej dwie obserwacje MZ;
- dla każdej pozycji wybrany znak otrzymał co najmniej dwa głosy;
- przy równej liczbie głosów wygrywa wariant z większą sumą confidence.

Przy dwóch pierwszych obserwacjach zwykle odpowiada to sytuacji:

```text
MZ #1: WA1234 -> wynik wstępny
MZ #2: WA1234 -> wynik potwierdzony temporalnie
```

Jeżeli jedna z pozycji jest sporna, np. `WA1234` i `WA1284`, wynik nie jest
jeszcze stabilny. Po większej liczbie obserwacji nie jest jednak wymagane, aby
całe sekwencje były identyczne klatka po klatce; stabilność jest wyliczana z
głosów dla poszczególnych pozycji w dominującej strukturze.

Confidence wyniku jest minimum ze średnich confidence zwycięskich znaków na
poszczególnych pozycjach. Sam warunek `stable` nie ma dodatkowego minimalnego
progu confidence ponad progi wcześniejszego postprocessingu MZ.

Istotne ograniczenie obecnego modelu danych:

- `PlateObservation.confirmed` pochodzi ze stanu tracku;
- `MainActivity` kopiuje tę flagę do `CapturedPlateItem.confirmed`;
- gdy bieżąca próba MZ nie wykryje znaków, agregator zwraca wcześniejszy wynik
  tracku;
- w rezultacie bieżący crop może otrzymać `confirmed=true` oraz tekst z
  pamięci, mimo że właśnie ten crop nie dostarczył znaków wspierających
  odczyt.

Etykieta galerii `Potwierdzona` oznacza zatem obecnie **stabilny wynik tracku**,
a nie „ten crop samodzielnie potwierdza numer”. Jest też niezależna od ręcznej
walidacji użytkownika:

```text
confirmed / stable        -> automatyczny konsensus temporalny MZ
VerificationStatus.ACCEPTED -> użytkownik oznaczył odczyt jako poprawny
VerificationStatus.CORRECTED -> użytkownik podał poprawiony ground truth
```

Docelowe doprecyzowanie kontraktu powinno rozdzielić co najmniej:

```text
trackConfirmed            -> stabilność numeru w historii tracku
freshMzSuccessful         -> MZ wykrył znaki na bieżącym cropie
cropSupportsConsensus     -> bieżący odczyt wspiera numer tracku
manualVerificationStatus  -> niezależna ocena użytkownika
```

Zmiana nazw i pól nie została jeszcze wdrożona. Powyższy opis dokumentuje
rzeczywistą semantykę aktualnego kodu i zapobiega traktowaniu `confirmed` jako
ground truth.

### Decyzje

- zoom jest transformacją kamery w obrębie tej samej sceny, dopóki analiza
  obrazu nie wykaże rzeczywistej zmiany kadru;
- geometria MT może być aktualizowana niezależnie od tekstu MZ;
- brak świeżego tekstu nie jest dowodem unieważniającym wcześniejszy odczyt;
- nowy numer musi przejść konserwatywną bramkę jakości i być przypisany do
  zablokowanego celu;
- blokada celu ma stan niepewny i może odmówić wyboru, zamiast automatycznie
  przeskoczyć na najbliższą tablicę;
- tracker Preview służy do ciągłości wizualnej, natomiast MT pozostaje źródłem
  dokładnej geometrii;
- `confirmed` nie jest metryką dokładności ani ground truth.

### Weryfikacja

- dodano testy jednostkowe `AutoZoomControllerTest`;
- dodano testy transformacji overlayu `AutoZoomOverlayTransformTest`;
- dodano testy blokady celu `AutoZoomTargetLockTest`;
- dodano testy ochrony przed dryfem `PreviewTrackerDriftGuardTest`;
- dodano testy polityki pamięci numeru `AutoZoomRecognitionMemoryTest`, w tym
  pusty MZ, słabszy odmienny tekst, wyraźnie lepszy tekst i odrzucenie zbyt
  krótkiego odczytu;
- pełne `testDebugUnitTest` zakończyło się powodzeniem;
- `lintDebug` zakończył się powodzeniem.

Najnowsza korekta pamięci numeru nie została jeszcze potwierdzona wizualnie na
fizycznym urządzeniu. Zielone testy JVM i lint potwierdzają logikę oraz
integrację kompilacyjną, ale nie zastępują próby kamera–monitor.

### Pozostało

- wykonać na fizycznym urządzeniu pełną sekwencję: detekcja z daleka,
  zoom-in, świeże MT, świeże MZ, zoom-out i ponowny wynik przy `1x`;
- potwierdzić, że celownik pozostaje na jednej tablicy przy drżeniu oraz
  umiarkowanym przesunięciu całego kadru;
- sprawdzić scenę z kilkoma tablicami i potwierdzić brak przełączenia MZ na
  obiekt konkurencyjny;
- sprawdzić przypadek, w którym przed zoomem nie ma znaków, a numer pojawia
  się dopiero po zbliżeniu;
- zweryfikować progi `AutoZoomRecognitionMemory` na rzeczywistych błędach MZ;
- rozdzielić w modelu danych stabilność tracku od wsparcia bieżącego cropa i
  ręcznej walidacji;
- po rozdzieleniu flag zaktualizować eksport badawczy, miniraport cropa i
  nazwy stanów w galerii.

---


## 2026-08-24 — domknięcie warstwy pomiarowej, dynamiczna rozdzielczość kamery, uproszczenie UI i walidacja wydajności

### Problem

Po wdrożeniu wariantów R0/R1/R2 i ręcznie sterowanej sesji analizy pozostały
cztery obszary, które mogły zniekształcić późniejsze eksperymenty albo utrudnić
ich obsługę:

- brakowało trwałego obiektu reprezentującego pojedynczy przebieg eksperymentu;
- timer był projektowany jako element ustawień, mimo że faktycznie jest
  parametrem pojedynczego uruchomienia eksperymentu;
- ustawienia kamery posługiwały się presetami, które nie gwarantowały zgodności
  deklarowanej i rzeczywistej rozdzielczości `ImageAnalysis`;
- raport MZ uwzględniał wartości `0 ms` również wtedy, gdy MZ nie wykonano,
  przez co percentyle czasu inferencji były zaniżane;
- główny ekran stopniowo urósł do panelu technicznego z osobnymi przyciskami
  Start/Stop, rozwijaniem i maksymalizacją galerii oraz ręcznie sterowanymi
  wysokościami kontenerów.

### Rozwiązanie — `ExperimentSession` i timer pojedynczego przebiegu

Wdrożono `ExperimentSession` jako niezależny od CameraX obiekt cyklu życia
jednego eksperymentu. Sesja przechowuje:

- unikalne `session_id`;
- stan `IDLE`, `RUNNING` albo `FINISHED`;
- typ eksperymentu i wariant, np. `roi_budget / r0_full_frame`;
- czas początku i końca;
- czas trwania liczony zegarem monotonicznym;
- powód zakończenia `manual`, `timer` albo `error`;
- zamrożoną konfigurację opcjonalnego timera.

`ExperimentSession` rozpoczyna się razem z pomiarem po naciśnięciu
`Uruchom analizę`, natomiast techniczny restart CameraX, np. po zmianie
rozdzielczości, nie tworzy nowej sesji i nie zeruje timera. Eksport pobiera
niemutowalny snapshot zakończonej sesji, dzięki czemu późniejsza zmiana ustawień
UI nie zmienia opisu już wykonanego przebiegu.

Timer przeniesiono z ekranu Opcje na ekran główny. Jest ustawieniem pojedynczego
przebiegu: użytkownik może uzbroić timer przed START, po czym wybrana wartość
zostaje zamrożona w `ExperimentSession`. Po zakończeniu sesji timer jest
rozbrajany; kolejne uruchomienie jest bez limitu czasu, dopóki użytkownik nie
uzbroi timera ponownie.

Weryfikacja:

- dwa kolejne START/STOP tworzą różne `session_id`;
- zmiana rozdzielczości podczas aktywnej analizy restartuje CameraX bez
  utworzenia nowej `ExperimentSession`;
- eksport po restarcie zawiera trace'y sprzed i po restarcie w tej samej sesji;
- odmowa uprawnienia kamery nie tworzy pustej sesji eksperymentalnej;
- timer 15 s automatycznie zakończył sesję z `completion_reason = timer`;
- timer 60 s pozostał liczony od pierwotnego START mimo restartu CameraX po
  około 16 s; log potwierdził zakończenie po około `60005 ms`, a nie 60 s po
  restarcie;
- snapshot `ExperimentSession` trafia do `report.json` razem z typem, wariantem,
  czasem trwania, powodem zakończenia i konfiguracją timera.

### Rozwiązanie — rzeczywiste rozdzielczości aparatu zamiast sztywnych presetów

Zrezygnowano z traktowania `FAST/AUTO/DISTANT` jako zamkniętej listy rozmiarów.
Dodano katalog rozdzielczości tylnej kamery odczytywany z Camera2 dla formatu
`YUV_420_888`. Ekran Opcje prezentuje użytkownikowi rzeczywiste rozdzielczości
zgłaszane przez urządzenie, wraz z proporcjami obrazu i liczbą megapikseli.

Obsługiwane są:

- tryb `Auto`, w którym aplikacja wybiera rozsądny format standardowy;
- jawny wybór konkretnego `width × height`;
- standardowa pula formatów Camera2;
- rozszerzona pula high-resolution, jeżeli urządzenie ją udostępnia.

Raport rozróżnia obecnie:

- `selection_mode` — `auto` albo `explicit`;
- `selected_resolution` — wybór użytkownika;
- `requested_width/height` — rozmiar zażądany przez aplikację;
- `actual_width/height` i `actual_resolution` — rozmiar rzeczywiście otrzymany
  z CameraX;
- `requested_resolution_matched` — jawna zgodność żądania z klatką;
- `extended_high_resolution_mode_requested` — informację, czy użyto specjalnej
  puli high-resolution Camera2.

Starsze pola `profile` i `high_resolution_mode_requested` pozostawiono jako
aliasy kompatybilności raportu v1.

Weryfikacja na Samsungu SM-A125F:

- katalog kamery wykrył formaty aż do `4000×3000`, czyli 12 MP;
- wybranie `4000×3000` dało `requested = actual = 4000×3000` i
  `requested_resolution_matched = true`;
- wszystkie trace'y tej sesji potwierdziły źródło `4000×3000`;
- `extended_high_resolution_mode_requested = false`, co oznacza, że na tym
  urządzeniu 12 MP należy do zwykłej puli YUV, mimo że jest wysoką
  rozdzielczością w potocznym znaczeniu;
- osobny test `640×480` również potwierdził pełną zgodność
  `requested = actual = 640×480`.

Wniosek metodologiczny:

Rozdzielczość źródła może być od tej chwili traktowana jako kontrolowana
zmienna eksperymentalna. Do porównań należy wykorzystywać rzeczywiste
`actual_width/height`, a nie samą etykietę ustawienia.

### Rozwiązanie — uproszczenie ekranu głównego i galerii

Ekran główny uproszczono tak, aby kamera i sterowanie przebiegiem były
najważniejszymi elementami interfejsu:

- osobne przyciski START i STOP zastąpiono jednym CTA zmieniającym funkcję;
- główny panel nie zawiera już osadzonej galerii ani mechanizmu
  `ukryta → rozwinięta → zmaksymalizowana`;
- usunięto ręczne wyliczanie wysokości panelu i kontenera RecyclerView;
- na ekranie głównym pozostaje prosty stan kolektora cropów i przycisk
  `Cropy · N`;
- galeria została przeniesiona do przeciąganego `BottomSheetDialog`;
- zaznaczanie wszystkich cropów, zapis wybranych oraz manualna walidacja
  pozostają w galerii, a nie na ekranie kamery;
- stan cropów, bitmapy, ground truth i eksport nie zostały zmienione — była to
  refaktoryzacja prezentacji, nie logiki danych.

Weryfikacja:

- kompilacja po migracji layoutów zakończyła się powodzeniem;
- nowy `bottom_sheet_gallery.xml` zawiera oddzielny RecyclerView, licznik,
  status kolekcji, selekcję zbiorczą i zapis;
- ręczny przegląd na urządzeniu potwierdził poprawne otwieranie i zamykanie
  Bottom Sheeta, zachowanie cropów oraz jeden przycisk Start/Stop analizy.

### Rozwiązanie — poprawne statystyki MZ

Wcześniej `MobileAlprEngine` bezwarunkowo wpisywał do `InferenceTrace`:

```text
character_preprocess = 0
character_inference  = 0
character_postprocess = 0
```

również dla klatek, w których MZ nie został uruchomiony. `MetricsCollector`
traktował takie zera jako prawdziwe obserwacje i uwzględniał je w p50/p90/p95.

Po poprawce czasy `rectification`, `character_preprocess`,
`character_inference` i `character_postprocess` są zapisywane tylko wtedy,
gdy w danej klatce wykonano co najmniej jedno uruchomienie MZ. Liczniki
`mz_runs`, `mz_skipped` i `invalid_plate_geometry` są raportowane niezależnie.
Brak etapu oznacza teraz „etap nie wystąpił”, a nie „wykonał się w 0 ms”.

Weryfikacja kontrolna przy źródle `640×480`:

- czas sesji: `74483 ms`;
- przetworzone klatki: 15;
- dropped frames: 16;
- statusy: 12 × `no_plate`, 1 × `preliminary`, 2 × `recognized`;
- `mz_runs = 2`;
- `character_preprocess.count = 2`;
- `character_inference.count = 2`;
- `character_postprocess.count = 2`;
- klatki bez MZ mają brak czasu MZ, a nie `0 ms`;
- potwierdzono również klatkę ze statusem `recognized` i `mz_runs = 0`, ponieważ
  stan rozpoznania może pochodzić z wcześniej zbudowanego konsensusu czasowego.

### Pomiar kontrolny wydajności i identyfikacja wąskiego gardła

Po usunięciu sztucznych zer wykonano kontrolny pomiar pełnego pipeline'u na
Samsungu SM-A125F przy źródle `640×480`.

| Etap | p50 [ms] | p95 [ms] | liczba próbek |
| --- | ---: | ---: | ---: |
| konwersja obrazu CameraX | 23,83 | 29,08 | 15 |
| MP preprocessing | 557,27 | 561,91 | 14 |
| MP inference | 375,86 | 416,02 | 14 |
| MP postprocessing | 187,04 | 188,39 | 14 |
| MT preprocessing | 517,89 | 531,20 | 15 |
| MT inference | 2973,36 | 2999,22 | 15 |
| MT postprocessing | 2,34 | 2,59 | 15 |
| rektyfikacja | 1,67 | 1,81 | 2 |
| MZ preprocessing | 256,24 | 256,29 | 2 |
| MZ inference | 472,33 | 475,09 | 2 |
| MZ postprocessing | 8,35 | 8,63 | 2 |
| cały pipeline | 4633,93 | 4949,70 | 15 |

Konfiguracja wykonawcza pomiaru:

- MP: YOLO26n Detect, NCNN FP32, CPU, 4 wątki;
- MT: YOLO26s Pose, około 10,58 mln parametrów, TFLite INT8, CPU, 2 wątki;
- MZ: YOLO26n Detect, około 2,52 mln parametrów, TFLite INT8, CPU, 1 wątek;
- wejście MT pozostaje stałe `640×640` niezależnie od tego, czy źródłem jest
  pełna klatka, czy ROI.

Wnioski:

- głównym wąskim gardłem bieżącej konfiguracji jest MT; sama inferencja
  detektora tablic zajmuje około 2,97 s dla mediany i dominuje czas całej
  klatki;
- preprocessing MT dodaje około 0,52 s;
- konwersja źródła `640×480` kosztuje około 24 ms, dlatego w tym pomiarze
  rozdzielczość kamery nie jest głównym ograniczeniem;
- pojedyncza inferencja MZ kosztuje około 0,47 s i jest wyraźnie lżejsza od MT;
- mediana całego pipeline'u `4,63 s` odpowiada teoretycznej przepustowości
  około `0,22` przetworzonej klatki/s, zatem bieżącej konfiguracji na
  SM-A125F nie należy określać jako systemu czasu rzeczywistego;
- ROI nie zmniejsza kosztu pojedynczej inferencji MT, ponieważ każdy wycinek
  jest ostatecznie skalowany do tego samego wejścia `640×640`;
- potencjalna korzyść MP→ROI→MT może wynikać z lepszej reprezentacji tablicy na
  wejściu modelu albo z harmonogramu wykonywania, ale każda dodatkowa analiza
  ROI oznacza kolejne pełne uruchomienie MT;
- wariant R2, a szczególnie R2 z pełnoklatkowym fallbackiem, może więc zwiększyć
  opóźnienie. Jest to hipoteza wymagająca bezpośredniego porównania R0/R1/R2,
  a nie gotowy wynik końcowy.

Decyzje badawcze:

- bieżący zestaw modeli pozostaje konfiguracją bazową do eksperymentu R0/R1/R2;
- przed tą serią nie należy zmieniać architektury MT, ponieważ wymieszałoby to
  wpływ polityki ROI z wpływem modelu;
- końcowe wartości wydajności wymagają co najmniej trzech powtórzeń każdego
  wariantu w tych samych warunkach;
- w eksperymencie ROI należy raportować liczbę faktycznych uruchomień MT,
  `plate_roi_runs`, `plate_full_frame_runs`, `full_frame_fallbacks` oraz
  `vehicle_runs`;
- po zakończeniu serii bazowej można wykonać osobny eksperyment optymalizacyjny
  porównujący lżejszy MT, mniejszy rozmiar wejścia, inne runtime'y i liczbę
  wątków.

### Interpretacja `dropped frames` i czasowego próbkowania sceny

W trakcie analizy wyników doprecyzowano znaczenie licznika `dropped_frames`.
Nie jest on bezpośrednią miarą stabilności obrazu ani czasu, przez jaki tablica
pozostawała dobrze widoczna. Jest natomiast wskaźnikiem niedopasowania
wydajności pipeline'u do szybkości strumienia wideo.

W konfiguracji CameraX wykorzystującej strategię `KEEP_ONLY_LATEST` kolejne
klatki mogą być generowane przez kamerę szybciej, niż pipeline jest w stanie je
przetwarzać. W takim przypadku starsze klatki są pomijane, a po zakończeniu
analizy bieżącej klatki aplikacja pobiera najnowszą dostępną obserwację. Wysoka
wartość `dropped_frames` oznacza więc, że system rzadko próbkuje scenę, mimo że
kamera fizycznie zarejestrowała w tym czasie więcej obrazów.

Zależność interpretacyjna jest następująca:

```text
większa liczba dropped frames
        ↓
rzadsze efektywne próbkowanie sceny
        ↓
większy odstęp czasowy między analizowanymi klatkami
        ↓
większe ryzyko pominięcia krótkiego okresu dobrej widoczności tablicy
```

Z tego powodu `dropped_frames` należy interpretować łącznie z czasem
przetwarzania pipeline'u oraz efektywną częstotliwością przetwarzania klatek.
Jeżeli mediana czasu pipeline'u wynosi około `4,63 s`, to teoretyczna
częstotliwość przetwarzania wynosi około `0,22 kl./s`. Oznacza to, że między
dwiema analizowanymi klatkami może minąć kilka sekund, mimo że kamera w tym
czasie wygenerowała wiele obrazów.

W praktyce jest to istotne dla ALPR w ruchu. Jeżeli tablica znajduje się w
korzystnym położeniu — jest ostra, czytelna i geometrycznie stabilna — tylko
przez krótki przedział czasu krótszy niż typowy odstęp między analizowanymi
klatkami, pipeline może nie otrzymać żadnej próbki reprezentującej ten moment.
Nie oznacza to błędu kamery ani detektora, lecz ograniczenie czasowej zdolności
systemu do próbkowania strumienia.

Wniosek badawczy:

- `dropped_frames` nie powinno być nazywane miarą stabilności obrazu;
- jest to pośrednia miara ograniczonej zdolności pipeline'u do czasowego
  próbkowania strumienia wideo;
- do analizy wyników live warto raportować również efektywną częstotliwość
  przetwarzania (`processed_fps`) albo typowy odstęp czasu między kolejnymi
  przetworzonymi klatkami (`processed_frame_interval_ms`);
- miary te pozwalają lepiej ocenić, czy system ma realną szansę uchwycić
  krótkotrwały okres stabilnego obrazu tablicy.

#### Gotowiec do pracy — dropped frames i stabilne okno obserwacji

> Liczba pominiętych klatek (`dropped frames`) nie stanowi bezpośredniej miary
> stabilności obrazu. Jest natomiast wskaźnikiem niedopasowania wydajności
> pipeline’u do szybkości strumienia wideo. Wzrost liczby pominiętych klatek
> oznacza rzadsze próbkowanie sceny przez system, a tym samym zwiększa
> prawdopodobieństwo pominięcia krótkiego przedziału czasu, w którym tablica
> rejestracyjna jest dobrze widoczna, ostra i geometrycznie stabilna. Z tego
> względu metrykę `dropped frames` należy interpretować łącznie z efektywną
> częstotliwością przetwarzania klatek lub średnim odstępem czasowym między
> kolejnymi analizowanymi klatkami.

> Jeżeli czas stabilnej obserwacji tablicy jest krótszy niż typowy odstęp
> między kolejnymi przetwarzanymi klatkami, system może nie zarejestrować
> żadnej próbki reprezentującej ten korzystny moment, nawet jeśli kamera
> fizycznie wygenerowała w tym czasie wiele klatek.

### Rozwiązanie — temporalny konsensus struktury wierszy znaków

Po rozszerzeniu `ReadingOrderResolver` o bardziej odporną rekonstrukcję
wierszy rozszerzono również `TemporalCharacterAggregator`, ponieważ sama
całkowita liczba znaków nie wystarcza do jednoznacznego opisu obserwacji.
Przykładowo ciąg o ośmiu znakach może pochodzić zarówno z tablicy
jednorzędowej `[8]`, jak i dwurzędowej `[3,5]` albo `[4,4]`.

W nowej wersji stan konsensusu jest rozdzielany według struktury przestrzennej
sekwencji. Dla każdej obserwacji MZ:

- znaki są najpierw grupowane w wiersze przez `ReadingOrderResolver`;
- dla każdego wiersza zapisywana jest liczba znaków;
- powstaje sygnatura struktury, np. `single_row [7]` albo `two_row [3,5]`;
- obserwacje o tej samej całkowitej długości, ale innej strukturze wierszy,
  trafiają do osobnych stanów konsensusu;
- dominujący stan jest wybierany na podstawie liczby zgodnych obserwacji, a
  przy remisie na podstawie sumarycznego confidence sekwencji;
- po co najmniej dwóch zgodnych obserwacjach struktura może zostać uznana za
  oczekiwaną dla danego tracku.

Przykład:

```text
klatka 1: [3,5]
klatka 2: [3,5]

=> expectedLayout    = two_row
=> expectedRowCounts = [3,5]
=> expectedCount     = 8
```

Dzięki temu dwa wyniki o tej samej liczbie znaków nie są już traktowane jako
równoważne wyłącznie na podstawie długości:

```text
stary mechanizm:
[8]   == [3,5]        # oba warianty mają 8 znaków

nowy mechanizm:
[8]   != [3,5]        # różna struktura przestrzenna
```

Istotnym założeniem jest to, że struktura czasowa nie stanowi ground truth.
Program nie „wie” z góry, ile wierszy ma tablica. Wnioskuje o dominującym
układzie z kolejnych obserwacji tego samego tracku. Z tego powodu
`expectedRowCounts` powinno być wykorzystywane jako kryterium preferencji lub
rankingu kandydatów, a nie jako bezwzględny filtr odrzucający każdy wynik o
innej strukturze. Kilka kolejnych błędnych obserwacji mogłoby bowiem utrwalić
błędny układ.

Weryfikacja:

- testy `ReadingOrderResolver` potwierdzają zachowanie pojedynczego wiersza przy
  jitterze współrzędnych Y oraz poprawną kolejność dwóch wierszy o różnych
  wysokościach znaków;
- testy `CharacterSequencePostProcessor` potwierdzają, że płaski
  `expectedCount` nadal ogranicza kandydatów dla tablic jednorzędowych, ale nie
  spłaszcza poprawnie rozpoznanej struktury dwurzędowej;
- testy `TemporalCharacterAggregator` potwierdzają, że `[4]` i `[2,2]` są
  przechowywane jako osobne stany mimo tej samej całkowitej długości oraz że po
  dwóch zgodnych obserwacjach udostępniane są `expectedLayout` i
  `expectedRowCounts`;
- `gradlew testDebugUnitTest` zakończył się powodzeniem po wprowadzeniu zmian.

Stan integracji:

- struktura wierszy jest już rozpoznawana i przechowywana w konsensusie
  temporalnym;
- kolejnym krokiem jest przekazanie `expectedLayout` i `expectedRowCounts`
  przez `PlateTrackCoordinator.Decision` do wyboru kandydatów MZ;
- do czasu wykonania tego kroku struktura jest dostępna w stanie tracku, ale
  nie wpływa jeszcze na ranking normalnego wyniku i ścieżki recall.

#### Gotowiec do pracy — konsensus czasowy struktury tablicy

> W celu zwiększenia stabilności rozpoznawania zastosowano konsensus czasowy
> dla kolejnych obserwacji tej samej tablicy. Po każdej inferencji MZ znaki są
> grupowane w wiersze na podstawie ich położenia geometrycznego, co pozwala
> opisać strukturę odczytu nie tylko przez liczbę znaków, lecz także przez
> liczbę wierszy i liczbę znaków w każdym z nich, np. `[7]` dla tablicy
> jednorzędowej lub `[3,5]` dla tablicy dwurzędowej. Dla każdego tracku
> przechowywane są osobne stany konsensusu dla różnych struktur. Po uzyskaniu
> co najmniej dwóch zgodnych obserwacji dominująca struktura może być
> wykorzystana jako informacja pomocnicza przy ocenie kolejnych wyników MZ.

> Informacja o oczekiwanej strukturze nie jest traktowana jako reguła
> bezwzględna ani jako ground truth. Służy jedynie do preferowania wyników
> zgodnych z dominującą historią danego tracku. Dzięki temu chwilowa detekcja,
> w której znaki dwóch wierszy zostaną błędnie potraktowane jako jeden ciąg,
> może otrzymać niższy priorytet niż alternatywny wynik zachowujący wcześniej
> obserwowany układ wierszy.

> Ograniczeniem mechanizmu jest możliwość utrwalenia błędnej struktury, jeżeli
> kilka kolejnych obserwacji zostanie błędnie zinterpretowanych w ten sam
> sposób. Z tego względu informacja temporalna powinna pełnić rolę kryterium
> rankingowego, a nie twardego filtra odrzucającego wyniki o odmiennej liczbie
> wierszy.

### Gotowce do pracy inżynierskiej

#### Opis implementacyjny — stały rozmiar wejścia MT i znaczenie ROI

> W zastosowanym rozwiązaniu detekcja tablic wykonywana jest przez model o
> stałym rozmiarze wejścia 640×640 pikseli. Oznacza to, że zarówno pełna klatka,
> jak i wycinek ROI przed inferencją są skalowane do identycznego rozmiaru
> tensora. Ograniczenie analizowanego obszaru do ROI nie zmniejsza zatem kosztu
> pojedynczego wykonania sieci MT. Potencjalna korzyść kaskady MP→ROI→MT wynika
> przede wszystkim ze zmiany zawartości obrazu wejściowego oraz możliwości
> sterowania częstotliwością uruchamiania kolejnych etapów, a nie ze zmniejszenia
> liczby operacji wykonywanych przez pojedynczą inferencję MT. Z tego powodu
> liczba ROI została potraktowana jako zmienna eksperymentalna.

#### Wynik kontrolny — identyfikacja głównego kosztu pipeline'u

> Pomiar kontrolny wykonany na urządzeniu Samsung SM-A125F wykazał, że głównym
> składnikiem opóźnienia badanego pipeline'u jest detektor tablic MT. Przy obrazie
> źródłowym 640×480 mediana czasu samej inferencji MT wyniosła około 2,97 s,
> natomiast mediana całego przetwarzania klatki około 4,63 s. Dla porównania
> konwersja obrazu z CameraX zajmowała około 24 ms, a inferencja modelu znaków
> MZ około 472 ms. Wynik wskazuje, że w bieżącej konfiguracji
> sprzętowo-modelowej podstawowym ograniczeniem wydajności nie jest akwizycja
> obrazu, lecz koszt obliczeniowy MT. Pomiar ma charakter kontrolny; końcowe
> wartości pracy powinny pochodzić z wielokrotnie powtórzonego eksperymentu.

#### Hipoteza do weryfikacji w eksperymencie R0/R1/R2

> Ponieważ każda analiza ROI wymaga pełnej inferencji modelu MT o niezmiennym
> rozmiarze wejścia, zwiększenie budżetu ROI może prowadzić do wzrostu, a nie
> spadku opóźnienia. Kaskada z detektorem pojazdów powinna być zatem oceniana nie
> tylko pod względem skuteczności lokalizacji tablicy, lecz także liczby
> faktycznych uruchomień MT oraz częstości pełnoklatkowego fallbacku.

Pozostało:

- wykonać kontrolowane R1 i R2 oraz pełną serię R0/R1/R2;
- wykonać co najmniej trzy powtórzenia każdego wariantu na tym samym materiale;
- domknąć obsługę kolejności znaków dla tablic dwurzędowych;
- zablokować lub jawnie raportować zmianę kluczowej konfiguracji eksperymentu
  podczas aktywnej `ExperimentSession`;
- po zamrożeniu wersji badawczej wykonać osobny eksperyment rozdzielczości oraz
  ewentualny eksperyment optymalizacyjny MT.

---


## 2026-08-25 — responsywny overlay między inferencjami, śledzenie wielu tablic, semantyka świeżości wyniku i nowe kierunki badawcze

### Problem — ciężki pipeline i nieciągła prezentacja wyniku

Przy czasie pojedynczego przebiegu liczonym w sekundach wynik MP/MT/MZ dociera do
warstwy UI znacznie rzadziej niż kolejne klatki `PreviewView`. Powodowało to kilka
niezależnych problemów prezentacyjnych:

- ramka tablicy mogła pozostawać nieruchoma do chwili nadejścia kolejnego MT;
- przy zmianie sceny stary overlay i HUD mogły przez pewien czas opisywać poprzedni
  obraz;
- `VEHICLE` i `VEHICLE_ROI` były jedynie migawką ostatniego MP i po upływie
  sztucznego TTL znikały;
- komunikat `MZ w tej klatce: 0` mógł występować jednocześnie z widocznym numerem,
  ponieważ numer pochodził z pamięci temporalnej tracku, a nie z nowej inferencji MZ;
- przy więcej niż jednej tablicy pierwszy prototyp lekkiego trackera nie zapewniał
  niezależnego śledzenia wszystkich obiektów.

Problem nie dotyczył wyłącznie estetyki. Warstwa prezentacji musi jednoznacznie
odróżniać świeżą obserwację modelu od informacji przeniesionej pomiędzy drogimi
inferencjami.

### Rozwiązanie — lekki `PreviewPlateTracker` niezależny od ciężkiego analizatora

Dodano tracker pracujący na bitmapach pobieranych bezpośrednio z `PreviewView`,
niezależnie od głównego analizatora CameraX. Tracker działa na zmniejszonym obrazie
w skali szarości i wykonuje lokalne dopasowanie fragmentów obrazu wokół narożników
quada tablicy.

Wersja wieloobiektowa:

- utrzymuje osobny `TrackState` dla każdej tablicy;
- przy świeżym wyniku MT kotwiczy wszystkie prawidłowe elementy `PLATE`;
- aktualizuje położenie na lekkich klatkach Preview pomiędzy kolejnymi wynikami MT;
- toleruje krótką serię nieudanych dopasowań, aby pojedynczy miss nie powodował
  migotania;
- po utracie wszystkich aktywnych tracków zwraca jawnie pustą listę, co powoduje
  unieważnienie starego overlayu;
- po świeżym MT jest ponownie kotwiczony dokładną geometrią modelu.

Bieżąca wersja trackera estymuje przede wszystkim translację quada. Cztery narożniki
są przechowywane osobno, ale ruch pomiędzy ciężkimi inferencjami jest jeszcze
sprowadzany do wspólnego przesunięcia. Niezależne śledzenie deformacji/perspektywy
każdego narożnika pozostaje możliwym rozszerzeniem.

Weryfikacja ręczna potwierdziła, że ramka `PLATE` pozostaje widoczna i podąża za
obiektem pomiędzy inferencjami. Potwierdzono również działanie wersji wielotablicowej.

### Rozwiązanie — `VEHICLE` i `VEHICLE_ROI` podążające za tablicą

Sztuczny czas życia pomarańczowych ramek został uznany za mylący. Ostatni prawidłowy
wynik MP ma znaczenie diagnostyczne do chwili kolejnego wyniku albo zmiany sceny.

Przyjęto lekki kompromis zamiast uruchamiania drugiego trackera pojazdów:

```text
ostatni wynik MP/MT
  -> VEHICLE
  -> VEHICLE_ROI
  -> PLATE bazowa

PreviewPlateTracker
  -> aktualna PLATE
  -> dx, dy względem PLATE bazowej
  -> to samo przesunięcie stosowane do VEHICLE i VEHICLE_ROI
```

Dla wielu pojazdów element diagnostyczny jest kojarzony z tablicą znajdującą się
wewnątrz jego obszaru, a następnie z odpowiadającym jej `trackId`. Nie jest to
pełnoprawny tracker pojazdów — ruch ramki MP jest wyprowadzany z ruchu tablicy — ale
pozwala zachować spójność wizualną bez kolejnego kosztownego algorytmu.

Weryfikacja ręczna na urządzeniu: rozwiązanie działa.

### Rozwiązanie — płynna korekta overlayu i geometria quada

`DetectionOverlayView` wykorzystuje krótką animację pomiędzy kolejnymi znanymi
położeniami. Elementy `PLATE` są dopasowywane po `trackId`, natomiast `VEHICLE` i
`VEHICLE_ROI` geometrycznie. Interpolowane mogą być nie tylko współrzędne bbox,
lecz również listy keypointów.

Dla tablicy właściwą reprezentacją wizualną pozostaje czworokąt wyznaczony przez
cztery narożniki MT. Mechanizm interpolacji keypointów jest gotowy do płynnego
morfowania quada pomiędzy kolejnymi obserwacjami MT. Końcowa weryfikacja wyglądu
animowanego quada po ostatniej zmianie renderera pozostaje do wykonania.

### Rozwiązanie — świeżość sceny i unieważnianie spóźnionych wyników

Dodano lekką warstwę obserwującą rzeczywisty obraz `PreviewView` niezależnie od
ciężkiego pipeline'u:

- `uiSceneGeneration` identyfikuje generację aktualnej sceny;
- wynik ciężkiej inferencji rozpoczętej przed zmianą generacji jest odrzucany;
- reset z UI jest przekazywany do pipeline'u jako nieblokujące
  `requestTrackingReset()` i wykonywany przed następnym `engine.run()`;
- `SceneChangeDetector` działa również na lekkich klatkach Preview;
- `SceneAnchorGuard` porównuje obraz z referencją ustawioną po świeżej obserwacji MT;
- po unieważnieniu sceny czyszczone są trackery, overlay diagnostyczny i stan HUD;
- HUD pokazuje kreski do chwili nadejścia pierwszego świeżego wyniku nowej sceny.

Mechanizm poprawia responsywność, ale `SceneAnchorGuard` nadal wymaga końcowej
kalibracji dla ruchu kamery: kontrolowana zmiana położenia nie powinna być mylona z
całkowicie nową sceną.

### Rozwiązanie — oddzielenie „wyniku MZ” od „MZ wykonano teraz”

Wynik tekstowy może pozostać dostępny z temporalnego stanu tracku, mimo że w
bieżącej klatce MZ nie zostało uruchomione. Zmieniono semantykę komunikatów i
etykiet:

```text
świeża próba MZ:
  <numer>

brak nowej próby MZ, wynik historyczny:
  <numer> · pamięć
```

Analogicznie komunikat statusu nie interpretuje już `MZ = 0` jako braku
rozpoznania, lecz jako „bez nowej próby; wynik z pamięci”.

Decyzja:

> Położenie tablicy, wynik tekstowy i fakt wykonania MZ są trzema różnymi
> informacjami. UI nie może sugerować, że model znaków pracował w każdej klatce
> tylko dlatego, że tekst pozostaje widoczny.

### Nowa obserwacja badawcza — różnica między `PIPE` a sumą inferencji

Podczas obserwacji HUD zauważono, że czas `PIPE` może być znacznie większy od sumy
czasów pokazywanych jako `MP + MT + MZ`, w niektórych bieżących przebiegach co
najmniej około dwukrotnie.

Nie oznacza to automatycznie ukrytego opóźnienia modelu. HUD pokazuje dla MP, MT i
MZ wyłącznie czasy samego `backend.run()`, natomiast `PIPE` obejmuje cały przebieg
od rozpoczęcia przetwarzania klatki do zakończenia `engine.run()`. Poza czystymi
inferencjami występują m.in.:

- konwersja `ImageProxy -> Bitmap` i ewentualny obrót bitmapy;
- preprocessing MP, MT i MZ;
- postprocessing i dekodowanie wyników;
- wielokrotne uruchomienia MT dla kilku ROI lub fallbacku pełnoklatkowego;
- deduplikacja, scoring jakości i tracking;
- rektyfikacja perspektywy;
- kolejność znaków i konsensus temporalny;
- tworzenie obiektów wynikowych i kopii bitmap cropów;
- pozostała logika Java pomiędzy mierzonymi etapami.

Dotychczasowy pomiar kontrolny `640×480` wykazał już, że preprocessing MP i MT jest
istotny, ale nowa obserwacja uzasadnia osobny **audyt budżetu czasu klatki**.

Planowany eksperyment:

```text
PIPE
  - camera_conversion
  - MP preprocess / inference / postprocess
  - MT preprocess / inference / postprocess
  - rectification
  - MZ preprocess / inference / postprocess
  = niewyjaśniony narzut (OVH)
```

Do HUD i raportu warto dodać `SUM` oraz `OVH`, a następnie — jeżeli `OVH` nadal
pozostanie duży — instrumentować kolejne fragmenty `MobileAlprEngine` aż do
wyjaśnienia różnicy. Przed wdrożeniem autozoomu audyt czasu ma wyższy priorytet,
ponieważ dodatkowa próba MT/MZ zwiększy koszt przetwarzania.

### Plan — rozszerzenie polityki cropów o poprawę confidence

Obecna `CropSamplingPolicy` zapisuje pierwszy crop tracku, zmianę tekstu, pierwsze
potwierdzenie, istotną poprawę ostrości oraz próbkę okresową. Nie uwzględnia jednak
wyraźnej poprawy `recognitionConfidence`, jeżeli tekst pozostał taki sam.

Planowana zmiana:

- `Previous` otrzyma `recognitionConfidence`;
- wzrost confidence MZ o ustalony próg, początkowo np. `0.10`, stanie się osobnym
  powodem zapisu;
- wcześniejsze słabsze lub błędne cropy nie będą usuwane, ponieważ stanowią
  wartościowy materiał do analizy przebiegu rozpoznawania;
- raport będzie mógł zachować sekwencję obserwacji od słabego wyniku do wyniku
  potwierdzonego.

Zmiana jest zaplanowana, ale na moment aktualizacji dziennika nie została jeszcze
wdrożona.

### Plan badawczy — adaptacyjny zoom i ponowna próba rozpoznania

Powstała koncepcja wykorzystania sterowania kamerą po uzyskaniu stabilnej ramki
tablicy. Docelowy przepływ nie powinien wykonywać MZ na starej geometrii po zmianie
zoomu, lecz ponownie przejść przez lokalizację tablicy:

```text
MT
  -> tablica zbyt mała / MZ mało pewne / brak stabilizacji
  -> AUTO ZOOM + AF/AE na tablicę
  -> krótka stabilizacja kamery
  -> ponowne MT
  -> nowy quad i rektyfikacja
  -> ponowne MZ
  -> porównanie wyniku przed i po zoomie
```

W kodzie funkcja powinna być traktowana jako `AutoZoomController`, a nie
`OpticalZoomController`, ponieważ CameraX może realizować żądany zoom przez zmianę
obiektywu fizycznego, crop cyfrowy albo kombinację obu metod zależnie od urządzenia.

Wymagania projektowe przed implementacją:

- najwyżej jedna kontrolowana próba zoom dla danego tracku;
- jawny stan np. `NORMAL -> ZOOM_REQUESTED -> ZOOM_SETTLING -> ZOOMED_RETRY -> DONE`;
- wybór jednego tracku priorytetowego przy wielu tablicach;
- blokada reakcji `SceneChangeDetector/SceneAnchorGuard` na zmianę obrazu wywołaną
  świadomie przez aplikację;
- zapis `zoom_ratio` i źródła próby w metadanych cropu/raportu;
- osobne porównanie confidence i skuteczności przed/po zoomie.

Autozoom pozostaje hipotezą i planem eksperymentalnym; nie jest jeszcze funkcją
ukończoną.

### Decyzje z sesji 2026-08-25

- warstwa UI może śledzić obiekt częściej niż wykonywane są modele, ale musi
  zachowywać informację o pochodzeniu wyniku;
- `PLATE` jest aktywnie śledzona na lekkich klatkach Preview pomiędzy MT;
- `VEHICLE` i `VEHICLE_ROI` mogą być wizualnie przenoszone wraz z przypisaną
  tablicą bez uruchamiania osobnego trackera pojazdów;
- wynik tekstowy z konsensusu temporalnego nie oznacza wykonania MZ w bieżącej
  klatce;
- spóźniony wynik ciężkiego pipeline'u nie może ponownie narysować starej sceny;
- wcześniejsze słabe cropy są materiałem badawczym i nie powinny być automatycznie
  zastępowane lepszymi;
- przed dokładaniem autozoomu należy wyjaśnić pełny budżet czasu `PIPE` i narzut
  poza samymi inferencjami;
- autozoom będzie oddzielnym eksperymentem, a nie ukrytą optymalizacją bazowego
  porównania R0/R1/R2.

### Weryfikacja

- wielotablicowy `PreviewPlateTracker` — potwierdzony ręcznie;
- stała widoczność ramki tablicy pomiędzy inferencjami — potwierdzona ręcznie;
- wspólne przesuwanie `VEHICLE/VEHICLE_ROI` wraz z przypisaną tablicą — potwierdzone
  ręcznie;
- semantyka `wynik z pamięci` dla klatki bez nowej próby MZ — obecna w kodzie;
- animacja/interpolacja geometrii overlayu — obecna w kodzie, końcowy test
  animowanego quada pozostaje otwarty;
- końcowa kalibracja unieważniania sceny przy ruchu kamery — otwarta;
- rozszerzona polityka cropów o `recognitionConfidence` — plan, nie wdrożono;
- adaptacyjny autozoom — plan, nie wdrożono;
- audyt `PIPE vs SUM vs OVH` — plan eksperymentalny, nie wykonano.

### Pozostało

- wykonać audyt czasu klatki i dodać `SUM/OVH` do diagnostyki;
- rozszerzyć `CropSamplingPolicy` o istotny wzrost `recognitionConfidence`;
- po audycie czasu zaprojektować i wdrożyć `AutoZoomController`;
- dodać metadane zoomu do cropów i raportu przed eksperymentem zoomu;
- przetestować końcowy animowany quad na urządzeniu;
- skalibrować `SceneAnchorGuard` dla ruchu kamery i przyszłej kontrolowanej zmiany
  zoomu;
- po domknięciu bieżących zmian uruchomić pełny zestaw regresyjny;
- kontynuować serię R0/R1/R2 na niezmienionej konfiguracji bazowej.


### Domknięcie audytu czasu pipeline'u — rozdzielenie `INF/AUX/OVH` i koszt rotacji 12 MP

#### Problem

Po wcześniejszej obserwacji, że `PIPE` bywa znacznie większy od sumy czasów
inferencji `MP + MT + MZ`, dodano pełniejszy bilans pojedynczej przetwarzanej
klatki. Celem było rozstrzygnięcie, czy brakujący czas wynika z nieopomiarowanej
logiki Java, czy z jawnych operacji pomocniczych otaczających modele.

Wprowadzono trzy rozłączne pojęcia:

```text
INF = vehicle_inference + plate_inference + character_inference
AUX = suma jawnie zmierzonych etapów pomocniczych
      (pre/postprocessing, konwersja kamery, rektyfikacja, setup/finalizacja)
OVH = PIPE - (INF + AUX)
```

`OVH` nie oznacza zatem czasu poza inferencją. Jest wyłącznie resztą, której nie
udało się przypisać do żadnego jawnie zmierzonego etapu i pełni rolę kontroli
kompletności pomiaru.

#### Instrumentacja

Do `InferenceTrace`/diagnostyki dodano m.in.:

- `engine_setup`;
- `engine_total`;
- `pipeline_finalize`;
- `inference_sum`;
- `auxiliary_sum`;
- `measured_stage_sum`;
- `pipeline_overhead`;
- `engine_measured_sum` i `engine_overhead`;
- rozbicie konwersji kamery na `camera_to_bitmap` oraz `camera_rotation`;
- diagnostyczny log `ALPR_TIMING_AUDIT` zawierający również liczbę uruchomień
  MT na ROI/full-frame, liczbę prób MZ i rzeczywistą rozdzielczość źródła.

Pomiar potwierdził, że `OVH` wynosił typowo około `63–69 ms` przy czasie
pipeline'u rzędu `10–12 s`, czyli mniej niż 1% całego przebiegu. Oznacza to, że
niemal cały czas został poprawnie przypisany do konkretnych etapów i nie
występował wielosekundowy „ukryty” koszt pomiędzy licznikami.

#### Identyfikacja kosztu rotacji pełnej bitmapy

Dla źródła `4000×3000` i orientacji wymagającej obrotu o `90°` rozbito
`camera_conversion` na dwie części. W stanie ustalonym otrzymano m.in.:

```text
frame=4
CAM        = 3096,621 ms
TO_BITMAP  =   52,714 ms
ROTATE     = 3026,201 ms
ROT        = 90°

frame=6
CAM        = 3524,660 ms
TO_BITMAP  =   61,393 ms
ROTATE     = 3445,532 ms
ROT        = 90°
```

Sama konwersja `ImageProxy -> Bitmap` była więc relatywnie tania. Dominującym
kosztem okazało się ręczne tworzenie obróconej bitmapy `4000×3000` przez
`Bitmap.createBitmap(..., Matrix, ...)`.

Zmiana parametru filtrowania z `true` na `false` zmniejszyła koszt, ale nie
usunęła problemu: obrót nadal trwał około `3 s` dla pojedynczej 12-megapikselowej
klatki.

#### Optymalizacja — rotacja po stronie CameraX

W `ImageAnalysis.Builder` włączono:

```java
.setOutputImageRotationEnabled(true)
```

Po zmianie CameraX dostarcza analizatorowi obraz już w docelowej orientacji.
Rzeczywisty rozmiar raportowany przez analizator zmienił się z `4000×3000` na
`3000×4000`, `rotationDegrees` spadło do `0`, a ręczna rotacja przestała być
wykonywana:

```text
frame=4
CAM        = 51,986 ms
TO_BITMAP  = 51,875 ms
ROTATE     = 0,000 ms
ROT        = 0°

frame=6
CAM        = 54,624 ms
TO_BITMAP  = 54,495 ms
ROTATE     = 0,000 ms
ROT        = 0°
```

Dla porównywalnych klatek w stanie ustalonym cały pipeline skrócił się:

```text
MZ wykonane:
10,653 s -> 7,710 s   (-27,6%)

bez nowej próby MZ:
11,638 s -> 8,108 s   (-30,3%)
```

Koszt etapu `CAM` spadł odpowiednio o około `98,3–98,5%` względem wariantu z
ręcznym obrotem pełnej bitmapy.

Pierwsza klatka po uruchomieniu nie jest używana do tego porównania jako wynik
steady-state, ponieważ zawiera dodatkowy koszt inicjalizacji silnika i backendów.

#### Dodatkowy wniosek — dwa ROI oznaczają dwa pełne uruchomienia MT

Log `ALPR_TIMING_AUDIT` potwierdził dla wariantu `R2`:

```text
MT_ROI = 2
MT_FULL = 0
MT_INF ≈ 5,8–5,9 s
```

Wartość `plate_inference` jest sumą uruchomień MT w danej iteracji. Oznacza to
około `2,9–3,0 s` na pojedyncze wykonanie MT, co jest zgodne z wcześniejszym
baseline'em `p50 ≈ 2,97 s`. Wzrost czasu R2 nie oznacza więc spowolnienia jednej
inferencji MT; wynika z wykonania dwóch pełnych inferencji dla dwóch ROI.

Po optymalizacji rotacji, dla przykładowych klatek steady-state, same inferencje
stanowiły około `78–83%` całego czasu pipeline'u. Głównym kosztem ponownie stało
się więc wykonywanie modeli, przede wszystkim wielokrotne MT, a nie obsługa
pełnej bitmapy kamery.

#### Kontrola granicy pomiaru CameraX

Koszt `setOutputImageRotationEnabled(true)` może być technicznie wykonywany przed
wejściem do metody `process()`, więc sam spadek `PIPE` nie wystarczałby do
stwierdzenia poprawy przepustowości. Pomocniczo porównano odstępy czasowe między
kolejnymi logami zakończonych iteracji z odpowiadającymi im czasami `PIPE`.
Różnica wynosiła około `0,2–0,25 s`, a nie kilka sekund. Nie ma więc przesłanki,
że dawny koszt `3–4 s` został jedynie przeniesiony poza licznik `PIPE`.

Jest to wniosek diagnostyczny dla badanego urządzenia, a nie uniwersalna gwarancja
wydajności CameraX na wszystkich telefonach. W eksperymencie końcowym należy
raportować także rzeczywistą przepustowość/odstęp między przetworzonymi klatkami.

#### Gotowiec do pracy — pełny budżet czasu zamiast samej inferencji

> Czas wykonania modelu nie jest równoważny czasowi przetwarzania pojedynczej
> klatki przez kompletny system ALPR. W aplikacji mobilnej rozdzielono czas
> inferencji sieci neuronowych (`INF`) od kosztu operacji pomocniczych (`AUX`),
> obejmujących m.in. konwersję obrazu, preprocessing, postprocessing i
> rektyfikację. Dodatkowo wyznaczono resztę `OVH`, rozumianą jako różnicę między
> pełnym czasem pipeline'u a sumą wszystkich jawnie zmierzonych etapów. W
> przeprowadzonym audycie `OVH` stanowił mniej niż 1% czasu przetwarzania, co
> potwierdziło, że obserwowana wcześniej różnica między sumą inferencji a czasem
> całego pipeline'u wynikała przede wszystkim z mierzalnych etapów pomocniczych,
> a nie z nieznanego narzutu wykonawczego.

#### Gotowiec do pracy — wpływ rotacji obrazu wysokiej rozdzielczości

> W przypadku źródła `4000×3000` stwierdzono, że ręczny obrót pełnej bitmapy o
> 90° po konwersji z `ImageProxy` stanowił istotne wąskie gardło. Sama operacja
> `ImageProxy -> Bitmap` trwała około `53–61 ms`, podczas gdy utworzenie
> obróconej bitmapy wymagało około `3,0–3,45 s`. Po przeniesieniu odpowiedzialności
> za rotację do `ImageAnalysis` CameraX analizator otrzymywał obraz w docelowej
> orientacji (`rotationDegrees = 0`), a zmierzony wewnątrz pipeline'u koszt
> konwersji spadł do około `52–55 ms`. Dla porównywalnych klatek steady-state
> czas pełnego pipeline'u zmniejszył się o około `28–30%`. Wynik pokazuje, że
> przekształcenia pełnorozdzielczych bitmap mogą stanowić koszt porównywalny z
> obliczeniami sieci neuronowych i powinny być uwzględniane w analizie
> wydajności mobilnego systemu ALPR.

#### Gotowiec do pracy — znaczenie budżetu ROI

> W wariancie z dwoma ROI (`R2`) łączny czas inferencji MT wynosił około
> `5,8–5,9 s`, przy czym licznik wykonania potwierdzał dwa osobne uruchomienia
> modelu. Czas pojedynczej inferencji pozostawał zbliżony do wcześniejszego
> baseline'u około `2,97 s`. Oznacza to, że ograniczenie przestrzenne obrazu do
> ROI nie obniża kosztu pojedynczego wywołania modelu o stałym wejściu `640×640`;
> zwiększenie liczby ROI może natomiast niemal liniowo zwiększać łączny koszt MT.
> Z tego względu warianty R0, R1 i R2 powinny być porównywane jako odmienne
> strategie harmonogramu i jakości lokalizacji, a nie jako prosta optymalizacja
> czasu inferencji pojedynczego modelu.

#### Decyzje

- `setOutputImageRotationEnabled(true)` pozostaje aktywne w bazowej konfiguracji
  CameraX;
- awaryjna obsługa niezerowego `rotationDegrees` pozostaje w
  `CameraImageConverter`, ale w normalnym przebiegu nie wykonuje pracy;
- `OVH` pozostaje metryką kontrolną w trace/raporcie, lecz do codziennej
  interpretacji HUD bardziej użyteczne są `INF` i `AUX`;
- audyt budżetu czasu uznaje się za domknięty na poziomie potrzebnym przed
  eksperymentem R0/R1/R2;
- bazowego modelu MT nie należy teraz zmieniać, aby nie mieszać wpływu strategii
  ROI z wpływem optymalizacji modelu;
- eksperyment autozoomu pozostaje osobnym etapem po bazowej serii R0/R1/R2.

#### Weryfikacja

- `ALPR_TIMING_AUDIT` działa na fizycznym urządzeniu;
- `INF/AUX/OVH` sumują się do pełnego budżetu klatki z resztą poniżej 1%;
- rozbicie `camera_conversion` na `camera_to_bitmap` i `camera_rotation`
  potwierdziło dominujący koszt ręcznej rotacji 12 MP;
- po włączeniu rotacji CameraX: `SRC=3000x4000`, `ROT=0`, `CAM_ROT=0`;
- porównywalne klatki steady-state wykazały skrócenie `PIPE` o około `27,6%` i
  `30,3%`;
- `MT_ROI=2` oraz zagregowany czas `MT_INF` potwierdzają dwa pełne uruchomienia MT
  w wariancie R2.

#### Pozostało

- uprościć HUD do czytelnego rozróżnienia `PIPE / INF / AUX`; `OVH` pozostawić
  przede wszystkim w trace i raporcie;
- wykonać kontrolowane przebiegi R0, R1 i R2 po tej samej wersji kodu;
- rozszerzyć `CropSamplingPolicy` o istotny wzrost `recognitionConfidence`;
- po bazowej serii R0/R1/R2 wrócić do eksperymentu autozoomu;
- wykonać pełny zestaw regresyjny przed zamrożeniem wersji badawczej.


### Plan dalszych badań — drzewo eksperymentów i kolejność prac

Po domknięciu audytu czasu pipeline'u przyjęto kolejność badań, która ogranicza
ryzyko mieszania wpływu kilku zmiennych jednocześnie. Nie planuje się wykonywania
pełnego iloczynu wszystkich możliwych konfiguracji. Najpierw badany jest wpływ
pojedynczych czynników przy pozostałych parametrach zamrożonych, następnie z
wariantów przechodzących bramki jakościowe budowane są nieliczne konfiguracje
końcowe.

Diagram planu badań:

![Plan badań aplikacji mobilnej ALPR](plan_badan_aplikacji_mobilnej_alpr.png)

Diagram ma charakter planistyczny. Lista wariantów eksportu oznacza kandydatów do
porównania, o ile dany wariant jest rzeczywiście generowany przez program
macierzysty i przechodzi walidację kontraktu `.alprmodel`. Brakującego artefaktu
nie zastępuje się sztucznie innym runtime'em ani innym checkpointem.

#### Etap 0 — zamrożenie wersji pomiarowej

Przed właściwą serią eksperymentów należy:

- uprościć HUD do wartości roboczych `PIPE / INF / AUX`;
- pozostawić `OVH` w trace i raporcie jako kontrolę kompletności pomiaru;
- zachować aktywne `setOutputImageRotationEnabled(true)`;
- utrwalić w raporcie liczbę faktycznych wywołań `MP/MT/MZ`, liczbę ROI,
  fallbacki, rozdzielczość źródła oraz czasy `camera_to_bitmap` i
  `camera_rotation`;
- uruchomić `testDebugUnitTest` i `assembleDebug`, a przed zamrożeniem wersji
  badawczej również pełny zestaw regresyjny.

Ta wersja ma być bazą porównawczą dla kolejnych badań. W trakcie pojedynczej
serii nie należy zmieniać architektury modeli, wejść ani zasad schedulera, chyba
że właśnie ta zmiana jest badaną zmienną.

#### Etap 1 — eksperyment R0/R1/R2: strategia ROI

Pierwszy eksperyment ma ocenić wpływ liczby ROI na czas i skuteczność całego
systemu:

```text
R0 — MT na pełnej klatce, MP pomijany
R1 — MP -> maksymalnie 1 ROI -> MT
R2 — MP -> maksymalnie 2 ROI -> MT
```

Warunki kontrolowane:

- ten sam telefon;
- ten sam komplet modeli i wariantów runtime;
- ta sama rozdzielczość źródła;
- ten sam materiał testowy i kolejność scen;
- ten sam profil rozpoznawania;
- co najmniej trzy powtórzenia każdego wariantu.

Metryki podstawowe:

- `PIPE`, `INF`, `AUX` i kontrolnie `OVH`;
- `plate_roi_runs`, `plate_full_frame_runs`, `full_frame_fallbacks`;
- liczba uruchomień MP, MT i MZ;
- czas do pierwszego wyniku i do potwierdzenia;
- `dropped_frames` i efektywny odstęp między przetworzonymi klatkami;
- po dodaniu ground truth: exact match i CER per unikalny track.

Pytanie badawcze:

> Czy poprawa reprezentacji tablicy uzyskana przez zawężenie obrazu do ROI
> rekompensuje koszt dodatkowego MP i kolejnych pełnych uruchomień MT, a jeśli
> tak, jaki budżet ROI daje najlepszy kompromis jakości i czasu?

#### Etap 2 — eksperyment rozdzielczości źródła

Po wyborze jednej strategii ROI należy zamrozić ją i porównać kilka rzeczywistych
rozdzielczości `ImageAnalysis`, np.:

```text
640×480
1280×720
1920×1080
3000×4000
```

Rozdzielczości należy dobierać z katalogu faktycznie wspieranego przez urządzenie.
Raportowane są wartości `actual_width/height`, nie tylko etykieta wyboru.

Oceniane będą przede wszystkim:

- koszt `CAM` i preprocessingów;
- `PIPE`, `INF`, `AUX`;
- liczba dropów / efektywne próbkowanie sceny;
- skuteczność dla małych i dalekich tablic;
- wpływ rozdzielczości na jakość MT, rektyfikacji i końcowego odczytu.

Pytanie badawcze:

> Czy dodatkowa informacja przestrzenna obrazu wysokiej rozdzielczości poprawia
> skuteczność ALPR na tyle, aby uzasadnić dodatkowy koszt akwizycji i
> preprocessingu?

#### Etap 3 — warianty eksportu z programu macierzystego

Eksperymenty dotyczące eksportu mają rozdzielać wpływ formatu/runtime'u od wpływu
samego checkpointu. Porównywane warianty muszą pochodzić z tego samego modelu
źródłowego i zachowywać zgodny kontrakt wejścia/wyjścia.

Planowani kandydaci, zależnie od faktycznie dostępnych artefaktów eksportera:

```text
ONNX FP32
TFLite FP32
TFLite INT8
NCNN FP32
NCNN INT8 — tylko jeżeli program macierzysty rzeczywiście generuje taki wariant
```

Nie należy od razu wykonywać pełnego iloczynu `MP × MT × MZ`. Najpierw dla
każdej roli osobno zmieniany jest jeden model/runtime przy dwóch pozostałych
rolach stałych. Dopiero warianty, które przejdą bramki jakości i stabilności,
mogą zostać użyte do zbudowania 2–3 najlepszych kompletnych konfiguracji.

Mierzone są:

- czasy preprocessingu, inferencji i postprocessingu;
- `PIPE/INF/AUX`;
- pamięć i termika;
- dokładność / exact match / CER z ground truth;
- stabilność i zgodność Python ↔ Android.

INT8 nie może zostać uznany za lepszy wyłącznie dlatego, że jest szybszy.
Kwantyzacja wymaga osobnej bramki jakościowej.

#### Etap 4 — runtime, liczba wątków i akceleracja

Po wybraniu sensownych artefaktów eksportowych należy osobno zbadać parametry
wykonania dostępne dla danego runtime'u:

- liczba wątków CPU;
- TFLite / ONNX / NCNN, jeżeli dla danego modelu dostępne są porównywalne
  artefakty;
- delegaty i akceleracja tylko wtedy, gdy działają stabilnie na urządzeniu;
- Vulkan / NNAPI / NPU jako osobne eksperymenty, a nie automatyczne rozszerzenie
  baseline'u.

Największy potencjalny zysk dotyczy obecnie MT, ponieważ jest głównym kosztem
obliczeniowym pojedynczej iteracji.

#### Etap 5 — polityka cropów

Po zamrożeniu baseline'u wydajności należy rozszerzyć `CropSamplingPolicy` o
informację `recognitionConfidence`. Nowy crop powinien zostać zachowany również,
gdy confidence odczytu poprawi się istotnie, np. o co najmniej `0,10`, nawet przy
niezmienionym tekście.

Docelowa polityka zachowuje crop, gdy występuje co najmniej jeden warunek:

- pierwszy crop tracku;
- zmiana rozpoznanego tekstu;
- pierwsze potwierdzenie;
- istotna poprawa confidence MZ;
- istotna poprawa ostrości;
- próbka okresowa.

Wcześniejsze słabsze cropy nie są automatycznie usuwane, ponieważ są wartościowe
w analizie błędów i przebiegu dochodzenia systemu do wyniku.

#### Etap 6 — autozoom jako osobny eksperyment

Autozoom nie wchodzi do baseline'u R0/R1/R2. Po ustaleniu zwykłego pipeline'u
należy zbadać osobny mechanizm:

```text
MT wykrywa tablicę
  -> tablica zbyt mała / MZ mało pewne / brak stabilizacji
  -> kontrolowany zoom kamery
  -> AF/AE na obszar tablicy
  -> krótka stabilizacja
  -> ponowne MT
  -> nowy quad i rektyfikacja
  -> MZ retry
  -> porównanie wyniku przed i po zoomie
```

Należy ograniczyć liczbę prób autozoomu na track, zabezpieczyć detekcję zmiany
sceny przed resetem wywołanym własnym zoomem aplikacji oraz raportować co najmniej:

- `zoom_ratio`;
- źródło próby `normal / auto_zoom_retry`;
- confidence przed i po zoomie;
- exact match / CER;
- dodatkowy czas do wyniku;
- informację o fizycznym obiektywie, jeżeli urządzenie i API pozwalają ją
  wiarygodnie ustalić.

Pierwsza wersja powinna być opisywana jako `adaptive camera zoom`, a nie
bezwzględnie „zoom optyczny”, ponieważ CameraX może realizować żądany zoom przez
kombinację zmiany fizycznego obiektywu i cropu cyfrowego.

#### Etap 7 — badanie schedulera MZ, trackingu i konsensusu temporalnego

Po ustabilizowaniu pipeline'u należy wykonać eksperyment A/B pokazujący wartość
przetwarzania temporalnego. Porównywane mogą być m.in.:

```text
A — niezależne odczyty / minimalna pamięć temporalna
B — scheduler MZ + tracking + temporalny konsensus tekstu i struktury wierszy
```

Metryki:

- liczba prób MZ na track;
- czas do pierwszego wyniku;
- czas do potwierdzenia;
- liczba zmian tekstu w czasie;
- udział wyników potwierdzonych;
- koszt MZ i `PIPE`;
- exact match / CER na poziomie tracku.

Tracker i konsensus poprawiają ciągłość i mogą ograniczać koszt obliczeniowy, ale
nie są traktowane jako ground truth.

#### Etap 8 — końcowy benchmark jakości

Końcowa ocena ma obejmować dwa rozdzielone scenariusze:

**Replay kontrolowany** — stałe obrazy/cropy i znane ground truth, używane do
porównań modeli, runtime'ów, kwantyzacji i zgodności Python ↔ Android.

**Camera-in-the-loop / live** — pełny tor kamery z MP, trackingiem, schedulerem,
rektyfikacją i konsensusem temporalnym, używany do oceny zachowania kompletnego
systemu.

Wyników obu scenariuszy nie należy łączyć w jedną średnią.

#### Etap 9 — wybór konfiguracji końcowej

Końcowy ranking nie powinien wybierać wyłącznie najszybszej konfiguracji.
Oceniany będzie kompromis pomiędzy:

- skutecznością end-to-end;
- exact match i CER;
- czasem `PIPE` i czasem do potwierdzenia;
- liczbą wywołań MT/MZ;
- stabilnością śledzenia i odczytu;
- pamięcią, termiką i przepustowością;
- odpornością na różne typy tablic i warunki sceny.

Z najlepszych wariantów należy wybrać niewielką liczbę finalistów, wykonać dla
nich powtarzalne serie końcowe i dopiero wtedy wskazać konfigurację rekomendowaną
do pracy inżynierskiej.

#### Kolejność wykonawcza

Przyjęta kolejność dalszych prac:

```text
HUD / raport i regresja
        ↓
R0 / R1 / R2
        ↓
rozdzielczość źródła
        ↓
warianty eksportu i runtime
        ↓
polityka cropów
        ↓
autozoom
        ↓
A/B mechanizmów temporalnych
        ↓
benchmark końcowy z ground truth
        ↓
ranking i konfiguracja rekomendowana
```

Decyzja metodologiczna:

> Nowa funkcja nie może zostać włączona do baseline'u przed zakończeniem
> eksperymentu, którego wynik mogłaby zaburzyć. W szczególności autozoom, nowe
> warianty MT i zmiana schedulera MZ nie powinny być wprowadzane do bazowej serii
> R0/R1/R2.

---


## 2026-08-25 — ergonomia prowadzenia eksperymentów i uproszczenie głównego ekranu

### Problem

Po rozpoczęciu właściwych serii R0/R1/R2 okazało się, że kilka czynności
wykonywanych po każdym 60-sekundowym przebiegu niepotrzebnie wydłużało badanie:

- eksport wymagał wejścia do ekranu Diagnostyki;
- po zakończeniu eksperymentu timer wracał do stanu nieaktywnego i trzeba było
  ustawiać go ponownie;
- systemowe paski Androida, wysoka belka tytułowa i tekst powtarzający treść
  przycisku `Uruchom analizę` ograniczały powierzchnię `PreviewView`;
- na głównym ekranie eksport był początkowo dostępny także bez wykonania
  eksperymentu, mimo że jego głównym zastosowaniem w tym miejscu jest zapis
  zakończonej sesji badawczej.

### Rozwiązanie

- dodano przycisk bezpośredniego eksportu zakończonej sesji
  `ResearchArchive.Kind.RESEARCH_SESSION` z głównego ekranu;
- przycisk jest widoczny wyłącznie w trybie `EXP`, gdy `ExperimentSession`
  znajduje się w stanie `FINISHED`; podczas trwania pomiaru oraz przed pierwszą
  sesją jest ukryty;
- pełny wybór trzech formatów eksportu nadal pozostaje w Diagnostyce;
- `TimerConfig` nie jest już zerowany po zakończeniu przebiegu; ustawione `60 s`
  pozostaje przygotowane dla kolejnego eksperymentu;
- `MainActivity` korzysta z trybu immersyjnego z chwilowym przywracaniem pasków
  systemowych gestem;
- toolbar zmniejszono do `48 dp`;
- komunikat instrukcyjny nad przyciskiem startu jest ukrywany, gdy analiza nie
  działa, a wraca jako bieżący status rozpoznania podczas pracy kamery;
- uporządkowano ikony: analiza, kolekcja cropów, galeria i eksport otrzymały
  różne znaczenia wizualne.

### Decyzja

Główny ekran ma wspierać krótką pętlę badawczą:

```text
ustaw wariant / timer
        ↓
uruchom eksperyment
        ↓
automatyczny STOP
        ↓
eksport sesji
        ↓
następny wariant
```

Eksport ogólny pozostaje funkcją diagnostyczną, natomiast główny przycisk
eksportu ma znaczenie wyłącznie dla zakończonego eksperymentu.

### Weryfikacja

- timer `60 s` pozostaje ustawiony po automatycznym zakończeniu eksperymentu;
- po zakończeniu sesji pojawia się bezpośredni przycisk eksportu
  `.alprsession`;
- trzy późniejsze serie R0/R1/R2 zostały wyeksportowane z głównego ekranu;
- tryb immersyjny i zmniejszony toolbar zwalniają dodatkową przestrzeń dla
  podglądu kamery.

---

## 2026-08-25 — S1: kontrola R0/R1/R2 na scenie z jednym pojazdem

### Cel

Pierwsza seria miała sprawdzić zachowanie trzech polityk ROI przy pojedynczym,
dobrze widocznym pojeździe i tej samej tablicy. Utrzymano:

- ten sam Samsung SM-A125F;
- te same modele i runtime'y;
- rozdzielczość wybraną `4000×3000`, faktyczny obraz po rotacji CameraX
  `3000×4000`;
- timer `60 s`;
- ten sam profil rozpoznawania i nieruchomą scenę.

W tej fazie nie istniała jeszcze automatyczna bramka termiczna, dlatego seria ma
również charakter pilotażowy dla metodologii temperatury.

### R0 — pełna klatka

Wykonano trzy sesje po 60 s. Każda przetworzyła 14 iteracji i wykonywała jedno
pełnoklatkowe MT na iterację.

Po połączeniu 42 iteracji:

| Metryka | Wartość |
| --- | ---: |
| `PIPE p50` | `3,561 s` |
| `PIPE p90` | `4,256 s` |
| `PIPE p95` | `4,260 s` |
| `MT p50` | `2,970 s` |
| `MT p90` | `3,142 s` |
| `CAM p50` | `56,9 ms` |
| czas do pierwszego wyniku — średnio z 3 sesji | `5,032 s` |
| czas do potwierdzenia — średnio z 3 sesji | `9,481 s` |

W każdej sesji scheduler MZ wykonał 4 próby i pominął 10 możliwych wywołań.
Obserwowany tekst był stabilny, ale brak ground truth oznacza, że nie jest to
dowód poprawności.

### R1 — maksymalnie jedno ROI

Trzy sesje przetworzyły łącznie 33 iteracje. Każda normalna iteracja używała
jednego ROI. W każdej sesji wystąpił jeden okresowy fallback pełnoklatkowy.

Wynik zbiorczy:

| Metryka | Wartość |
| --- | ---: |
| `PIPE p50` | `4,742 s` |
| `MT p50` | `2,979 s` |
| liczba `plate_roi_runs` | `33` |
| liczba dodatkowych full-frame | `3` |
| średni czas do pierwszego wyniku | `6,532 s` |
| średni czas do potwierdzenia | `11,293 s` |

Samo pojedyncze MT nie przyspieszyło względem R0. Dodatkowy koszt wynikał z MP,
preprocessingu i okazjonalnego fallbacku.

W trzeciej sesji pojawił się `THERMAL_STATUS_LIGHT`; równocześnie wzrosły czasy
MT, MZ i pełnego pipeline'u. Była to pierwsza mocna przesłanka, że kolejne serie
nie powinny startować w przypadkowym stanie termicznym.

### R2 — maksymalnie dwa ROI

Na scenie z jednym pojazdem R2 nie wykorzystało drugiego budżetu ROI. We
wszystkich 35 iteracjach trzech sesji licznik `plate_roi_runs` wynosił dokładnie
1 na iterację. Z tego powodu wynik był praktycznie równoważny R1:

| Metryka | Wartość |
| --- | ---: |
| `PIPE p50` | `4,743 s` |
| `MT p50` | `2,983 s` |
| `CAM p50` | `57,7 ms` |
| ROI na normalną iterację | `1` |

W drugiej i trzeciej sesji wystąpił pojedynczy okresowy fallback full-frame.
Trzecia sesja ponownie rozpoczęła raportowanie przy statusie termicznym 1 i była
wolniejsza od pierwszej.

### Wnioski z S1

1. Ograniczenie źródła do ROI nie obniża kosztu pojedynczego MT, ponieważ
   niezależnie od wielkości źródłowego wycinka model otrzymuje tensor
   `640×640`.
2. Przy jednym pojeździe R2 zachowuje się jak R1, jeżeli MP zwraca tylko jeden
   użyteczny kandydat.
3. R1/R2 dokładają koszt MP i jego preprocessingu, dlatego na tej scenie były
   wolniejsze od R0.
4. Okresowy fallback zwiększa ogon rozkładu opóźnienia, lecz nie wyjaśnia całej
   różnicy mediany R0 względem R1/R2.
5. Rosnący stan termiczny telefonu jest czynnikiem zakłócającym i wymaga
   kontrolowanego warunku startu.
6. Stabilność tekstu nie zastępuje ground truth; seria S1 jest przede wszystkim
   eksperymentem wydajności i zachowania schedulera.

---

## 2026-08-25 — S2: dwa pojazdy ujawniają rzeczywistą różnicę R1 i R2

### Cel

Druga scena została przygotowana tak, aby MP miał dwa sensowne pojazdy do
wyboru. Jest to konieczne, ponieważ dopiero wtedy różnica między limitem jednego
i dwóch ROI staje się rzeczywistą zmianą wykonawczą.

### Pierwszy blok pilotażowy

W pierwszej serii z dwoma pojazdami otrzymano:

| Metryka | R0 | R1 | R2 |
| --- | ---: | ---: | ---: |
| przetworzone iteracje | 14 | 11 | 6 |
| `PIPE p50` | `3,553 s` | `4,751 s` | `8,509 s` |
| `MT p50` | `2,963 s` | `2,984 s` | `6,004 s` |
| `MT PRE p50` | `0,500 s` | `0,495 s` | `0,992 s` |
| `MP p50` | — | `0,461 s` | `0,456 s` |
| `MZ p50` | `0,928 s` | `0,468 s` | `0,927 s` |
| ROI na iterację | 0 | 1 | 2 |
| full-frame fallback | 0 | 1 | 1 |
| pierwszy wynik | `5,554 s` | `11,155 s` | `19,419 s` |
| pierwsze potwierdzenie | `31,070 s` | brak | brak |

R2 w każdej normalnej iteracji wykonało dwa MT na dwóch ROI. Łączny czas MT
wzrósł więc do około `6 s`. R1 wykonywało tylko jedno ROI, a R0 jedno MT na
pełnej klatce.

W obserwowanych wynikach R0 i R2 zwracały dwa ciągi tablic, natomiast R1
konsekwentnie obejmowało tylko jeden pojazd. Bez ground truth można mówić o
pokryciu dwóch kandydatów, a nie o poprawności obu odczytów.

### Wniosek z pilotażu

R2 nie jest „szybszą wersją ROI”. Jest strategią zwiększającą pokrycie sceny
kosztem niemal liniowego wzrostu czasu MT. Dodatkowo mniejsza liczba
przetworzonych iteracji na minutę ogranicza liczbę obserwacji dostępnych dla
konsensusu temporalnego.

---

## 2026-08-25 — warunek termiczny przed eksperymentem

### Problem

Kolejne 60-sekundowe przebiegi wykonywane bez przerwy powodowały wzrost stanu
termicznego telefonu. Samo przeplatanie kolejności R0/R1/R2 ograniczałoby
systematyczny błąd, ale nie gwarantowałoby porównywalnego stanu urządzenia przed
każdym START-em.

Dodatkowym problemem jest rzadkie odświeżanie temperatury baterii przez Androida.
Aplikacja może odpytywać stan co sekundę, ale wartość `BatteryManager`
potrafi pozostać niezmieniona przez kilka minut.

### Rozwiązanie

Dodano klasy:

- `ThermalMonitor` — odczytuje temperaturę baterii oraz
  `PowerManager.getCurrentThermalStatus()`;
- `ThermalConfig` — opisuje opcjonalny warunek rozpoczęcia eksperymentu:
  maksymalną temperaturę baterii, maksymalny status termiczny i czas
  stabilizacji.

Monitor działa również wtedy, gdy kamera jest zatrzymana. W trybie `EXP`
bieżący stan jest dostępny na głównym ekranie przed uruchomieniem analizy.

Jeżeli warunek termiczny jest aktywny, żądanie START nie powinno uruchamiać
właściwego pomiaru, dopóki warunek nie pozostaje spełniony przez zadany czas.
W okresie oczekiwania kamera i timer eksperymentu nie powinny generować
dodatkowego obciążenia. Po spełnieniu warunku rozpoczyna się normalna
`ExperimentSession`, pomiar `MetricsCollector` i timer.

Dla finalnej serii S2 przyjęto roboczo:

```text
BAT <= 33,0°C
TH = 0 / THERMAL_STATUS_NONE
stabilizacja = 5 s
```

Początkowy próg `32,0°C` okazał się niepraktyczny z powodu długiego czasu
chłodzenia i wolnej aktualizacji czujnika baterii. Zmiana progu jest parametrem
metodyki, nie wynikiem modelu.

### Znaczenie TH

`TH` jest dyskretnym statusem termicznym Androida:

```text
0 = NONE
1 = LIGHT
2 = MODERATE
3 = SEVERE
4 = CRITICAL
5 = EMERGENCY
6 = SHUTDOWN
```

Dla bieżącej serii dopuszczany jest wyłącznie `TH0`, czyli
`THERMAL_STATUS_NONE`.

### Thermal Headroom — plan pomiarowy, jeszcze nie baseline

Rozważono również `PowerManager.getThermalHeadroom()`. Wskaźnik ten jest
bezwymiarowy — nie ma jednostki °C. Wartość `1,0` odpowiada progowi
`THERMAL_STATUS_SEVERE`, a wartości mniejsze oznaczają większy zapas termiczny;
wartość może również przekroczyć `1,0`.

Przyjęto decyzję, aby najpierw tylko obserwować `HEAD` i zebrać jego zakres dla
konkretnego SM-A125F. Dopiero później można zdecydować, czy powinien wejść do
automatycznej bramki startowej. W bieżącym stanie repozytorium `ThermalMonitor`
raportuje BAT i TH; odczyt HEAD pozostaje rozszerzeniem do wdrożenia.

### Ograniczenie raportu

`DeviceProfile` tworzony podczas eksportu opisuje stan urządzenia w chwili
generowania raportu, a nie gwarantowany snapshot dokładnie z chwili START.
Zdarzenie:

```text
Warunek termiczny spełniony: <BAT> TH<status>
```

jest obecnie zachowywane w `application.log`.

Do docelowego schematu raportu należy dopisać osobno:

- konfigurację bramki termicznej;
- `start_battery_temperature_c`;
- `start_thermal_status`;
- `end_battery_temperature_c`;
- `end_thermal_status`;
- po wdrożeniu: `start/end_thermal_headroom`.

---

## 2026-08-25 — S2 kontrolowane termicznie, powtórzenie 1

### Warunki startu

Po wdrożeniu bramki termicznej powtórzono serię R0 → R1 → R2. Z logów
`application.log` odczytano rzeczywisty stan przy uruchomieniu:

```text
R0: BAT 32,4°C · TH0
R1: BAT 32,5°C · TH0
R2: BAT 32,5°C · TH0
```

Wszystkie trzy warianty startowały więc z praktycznie identycznego pułapu
termicznego.

### Wyniki

| Metryka | R0 | R1 | R2 |
| --- | ---: | ---: | ---: |
| przetworzone iteracje | 14 | 12 | 6 |
| `DROP` | 14 | 13 | 7 |
| `PIPE p50` | `3,564 s` | `4,523 s` | `8,265 s` |
| `PIPE p90` | `4,998 s` | `5,441 s` | `9,208 s` |
| `MT p50` | `2,974 s` | `2,979 s` | `5,979 s` |
| `MT PRE p50` | `0,508 s` | `0,503 s` | `1,007 s` |
| `MP p50` | — | `0,465 s` | `0,433 s` |
| `MZ p50` | `0,928 s` | `0,472 s` | `0,932 s` |
| ROI łącznie | 0 | 12 | 12 |
| ROI / iterację | 0 | 1 | 2 |
| MT full-frame | 14 | 0 | 0 |
| fallback full-frame | 0 | 0 | 0 |
| czas do pierwszego wyniku | `5,875 s` | `7,191 s` | `10,792 s` |
| czas do pierwszego potwierdzenia | `31,520 s` | brak | brak |

### Interpretacja

- R0 wykonuje jedno pełnoklatkowe MT i na tej konkretnej scenie obejmowało oba
  pojazdy przy najniższym czasie pipeline'u.
- R1 wykonuje jedno MT na jednym ROI. Jest wolniejsze od R0 przez dodatkowy MP,
  a limit jednego ROI oznacza pominięcie drugiego kandydata.
- R2 wykonuje dwa MT na dwóch ROI. `MT p50` i `MT PRE p50` są praktycznie
  dwukrotnością R1, co potwierdza liniowy koszt drugiego ROI dla modelu o stałym
  wejściu.
- R2 przetworzyło tylko 6 iteracji w 60 s, podczas gdy R0 14, a R1 12. Wysoki
  koszt pojedynczej iteracji zmniejsza więc także liczbę obserwacji temporalnych
  dostępnych w tym samym czasie.
- `MZ p50` dla R0 i R2 jest około dwa razy większe niż dla R1, ponieważ w
  wybranych próbach obsługiwane były dwie tablice.
- w 60-sekundowym oknie tylko R0 osiągnęło stan potwierdzony; nie oznacza to
  jeszcze lepszej jakości odczytu, ponieważ brak ground truth.

### Powtarzalność względem pilotażu

Porównanie median `PIPE`:

| Wariant | pilot S2 | S2 z bramką termiczną |
| --- | ---: | ---: |
| R0 | `3,553 s` | `3,564 s` |
| R1 | `4,751 s` | `4,523 s` |
| R2 | `8,509 s` | `8,265 s` |

Podstawowy wzorzec pozostał taki sam po wyrównaniu warunków termicznych, co
wzmacnia wniosek o charakterze kosztu R0/R1/R2.

### Stan termiczny w chwili eksportu

`DeviceProfile` jest tworzony podczas generowania raportu, dlatego poniższe
wartości należy traktować jako snapshot z chwili eksportu, a nie dokładny
pomiar końca timera:

| Wariant | BAT przy starcie z `application.log` | BAT przy eksporcie | TH przy eksporcie |
| --- | ---: | ---: | ---: |
| R0 | `32,4°C` | `32,5°C` | `0 / NONE` |
| R1 | `32,5°C` | `32,5°C` | `0 / NONE` |
| R2 | `32,5°C` | `33,7°C` | `1 / LIGHT` |

R2, jako najcięższy wariant obliczeniowy, doprowadziło w tej sesji do
przejścia systemowego statusu termicznego z `TH0` przy starcie do `TH1` w
chwili eksportu. Nie należy utożsamiać temperatury baterii z temperaturą CPU;
status `TH` jest niezależną oceną systemu Android.

### Artefakty pomiarowe z 2026-08-25

Zachowano źródłowe paczki `.alprsession` użyte do obliczeń w tym wpisie:

**S1 — jeden pojazd**

- R0: `alpr_session_r0_20260825_141442.alprsession.zip`,
  `alpr_session_r0_20260825_141655.alprsession.zip`,
  `alpr_session_r0_20260825_141856.alprsession.zip`;
- R1: `alpr_session_r1_20260825_143427.alprsession.zip`,
  `alpr_session_r1_20260825_143649.alprsession.zip`,
  `alpr_session_r1_20260825_143854.alprsession.zip`;
- R2: `alpr_session_r2_20260825_154740.alprsession.zip`,
  `alpr_session_r2_20260825_154946.alprsession.zip`,
  `alpr_session_r2_20260825_155124.alprsession.zip`.

**S2 — dwa pojazdy, pilot**

- R0: `alpr_session_20260825_165135.alprsession.zip`;
- R1: `alpr_session_20260825_165318.alprsession.zip`;
- R2: `alpr_session_20260825_165454.alprsession.zip`.

**S2 — dwa pojazdy, powtórzenie 1 z kontrolą termiczną**

- R0: `alpr_session_20260825_191723.alprsession.zip`;
- R1: `alpr_session_20260825_191902.alprsession.zip`;
- R2: `alpr_session_20260825_192108.alprsession.zip`.

Nazwy plików są częścią śladu reprodukowalności. Same paczki pomiarowe nie
powinny być automatycznie commitowane do publicznego repozytorium, jeżeli
zawierają obrazy tablic lub inne dane, których publikacja nie została
świadomie zatwierdzona.

### Decyzja o liczbie powtórzeń

Po wprowadzeniu automatycznej bramki termicznej permutowanie kolejności wariantów
nie jest już głównym mechanizmem kontroli nagrzewania. Każdy przebieg ma
rozpoczynać się dopiero po osiągnięciu tego samego kryterium startowego.

Plan:

```text
S2 — powtórzenie 1: R0 -> R1 -> R2   [wykonane]
S2 — powtórzenie 2: R0 -> R1 -> R2   [do wykonania]
S2 — powtórzenie 3                    [tylko przy dużej zmienności / wyniku odstającym]
```

Trzecie powtórzenie nie jest wykonywane mechanicznie, jeżeli dwa kontrolowane
powtórzenia oraz wcześniejszy pilot dają zgodny wzorzec. Jeżeli jednak pojawi się
nietypowy fallback, zmiana liczby ROI lub wyraźnie odstający czas, trzecia seria
staje się próbą rozstrzygającą.

---

## 2026-08-25 — audyt dokumentacji w repozytorium

Sprawdzono katalog `docs/` oraz główny dziennik w gałęzi `main`.

Aktualne dokumenty repozytorium:

- `docs/alpr-model-v1.schema.json`;
- `docs/alpr-package-v1.schema.json`;
- `docs/alpr-mobile-research-bundle-v1.schema.json`;
- `docs/model_package_v1.md`;
- `docs/model_package_test_strategy.md`;
- `docs/mobile_architecture.md`;
- `docs/mobile_research_export.md`.

Główny `DZIENNIK_BUDOWY_APLIKACJI.md` w repozytorium kończył chronologię na
2026-08-21, dlatego nie zawierał dużej części zmian z 22–25 sierpnia.

Dwa dokumenty opisowe wymagają późniejszej synchronizacji z kodem:

1. `docs/mobile_architecture.md` nadal zawiera elementy wcześniejszego stanu,
   m.in. poziomy `RecyclerView` galerii, stare trzy profile rozdzielczości,
   informację o przyszłym adapterze NCNN oraz eksport wyłącznie z Diagnostyki.
   Aktualny kod ma Bottom Sheet galerii, dynamiczny katalog rozdzielczości,
   działający NCNN/JNI, tryb `EXP`, HUD, lekki tracker Preview i bezpośredni
   eksport zakończonej sesji.
2. `docs/mobile_research_export.md` poprawnie opisuje strukturę paczki
   `.alprsession`, ale nie opisuje jeszcze bezpośredniego eksportu zakończonego
   eksperymentu z głównego ekranu ani planowanych pól termicznych START/STOP.

Decyzja:

> Bieżący kod, raporty `.alprsession` oraz ten dziennik są źródłem prawdy dla
> stanu eksperymentalnego z 2026-08-25. Dokumenty `mobile_architecture.md` i
> `mobile_research_export.md` należy zsynchronizować przed zamrożeniem
> dokumentacji do pracy.

---

# 6. Najważniejsze decyzje projektowe

## Pakiet zamiast surowych wag

Android nie interpretuje `best.pt`. Format `.alprmodel` oddziela środowisko
treningowe od wdrożeniowego i przenosi kontrakt inferencji.

## Walidacja po obu stronach

Eksporter Python waliduje artefakt przed zapisem, ale Android powtarza
walidację.

## Osobny wariant dla każdego etapu

MP, MT i MZ mogą mieć inne runtime'y, wejścia, precyzje i liczbę wątków.

## NMS poza grafem

Modele mogą być eksportowane bez NMS w grafie, a Android wykonuje wspólny
kontrolowany postprocessing.

## Brak udawanej jakości

Confidence, stabilność tekstu i częstość pojawienia się wyniku nie są
dokładnością. CER i exact match wymagają ground truth.

## Prywatne i trwałe przechowywanie

Modele są rozpakowywane do prywatnego katalogu aplikacji. Aktywne rekordy są
odtwarzane na podstawie rejestru i fingerprintów.

## Niemutowalny pakiet bazowy

Eksperymentalna kompozycja urządzenia nie modyfikuje źródłowego
`source.alprmodel`.

## Tracking nie jest ground truth

Tracker poprawia ciągłość i ogranicza koszt MZ, ale nie może być używany do
udowadniania poprawności odczytu.

## Scene reset nie kasuje historii badawczej

Zmiana sceny czyści stan czasowy inferencji i overlayu, ale nie usuwa cropów
zebranych wcześniej.


## Eksperyment jest warstwą nad konfiguracją normalną

Konfiguracja eksperymentalna nie może trwale modyfikować normalnych ustawień
użytkownika. `ExperimentConfig` tworzy efektywną konfigurację tylko na czas
badania, a po jego wyłączeniu system wraca do stanu normalnego.

## Sesja pomiarowa jest krótsza niż życie aplikacji

Czas raportowany jako czas przebiegu nie rozpoczyna się przy utworzeniu
`MainActivity` ani `MetricsCollector`. Sesja pomiarowa rozpoczyna się jawnie
wraz z uruchomieniem analizy i kończy przed sprzątaniem CameraX oraz UI.

## Timer eksperymentalny jest opcjonalny

Ograniczenie czasowe ma być dodatkowym warunkiem zakończenia eksperymentu, a
nie częścią obowiązkową wszystkich przebiegów. Eksperyment bez timera kończy
się ręcznie.


## Ciągłość UI jest oddzielna od częstotliwości inferencji

Ciężki pipeline może analizować klatki znacznie rzadziej niż odświeżany jest
`PreviewView`. Dlatego ciągłość overlayu jest realizowana lekkim trackerem obrazu,
a nie przez udawanie, że MP/MT/MZ wykonały się w każdej klatce podglądu.

## Świeży wynik modelu i wynik z pamięci są rozróżniane

Widoczny numer rejestracyjny może pochodzić z konsensusu temporalnego mimo braku
nowej próby MZ. UI ma jawnie sygnalizować pochodzenie wyniku, a metryki inferencji
mają opisywać wyłącznie rzeczywiście wykonane etapy.

## Pomiar pełnego pipeline'u wymaga bilansu czasu

`total` nie może być interpretowany jako suma samych czasów `backend.run()`.
Analiza wydajności ma obejmować preprocessing, postprocessing, konwersję kamery,
rektyfikację i pozostały narzut. Planowany bilans `PIPE = SUM + OVH` ma wskazać,
czy po uwzględnieniu znanych etapów pozostaje istotny niezmierzony koszt.

---


## Warunek termiczny jest częścią protokołu eksperymentu

Timer opisuje warunek STOP, natomiast `ThermalConfig` opisuje warunek START.
Pomiar nie powinien rozpoczynać się w trakcie chłodzenia urządzenia. Dla
kontrolowanej serii S2 przyjęto `BAT <= 33°C`, `TH0` oraz 5 s stabilizacji.

## ROI nie zmniejsza kosztu stałego wejścia MT

Dla MT o wejściu `640×640` wycięcie mniejszego obszaru źródłowego poprawia
reprezentację przestrzenną obiektu, ale nie zmniejsza liczby operacji pojedynczej
inferencji. Każde dodatkowe ROI oznacza kolejne pełne preprocessing + inference
+ postprocessing MT.

## Kolejność R0/R1/R2 nie zastępuje kontroli termicznej

Po wdrożeniu automatycznej bramki termicznej kolejność wariantów nie jest już
podstawową metodą równoważenia nagrzewania. Wciąż należy rejestrować stan
termiczny i powtarzać eksperyment, ale każdy wariant powinien startować z tego
samego jawnego kryterium.

## HEAD jest wskaźnikiem bezwymiarowym

`Thermal Headroom` nie jest temperaturą i nie ma jednostki fizycznej. Zanim
zostanie użyty jako warunek START, należy zebrać jego zakres na konkretnym
telefonie i powiązać go z obserwowanymi statusami TH oraz spadkiem wydajności.


# 7. Weryfikacja

Stan na 2026-08-26:

| Kontrola | Wynik |
| --- | --- |
| `gradlew testDebugUnitTest` | ostatni pełny przebieg: sukces, 51/51 testów JVM |
| `gradlew assembleDebug` | sukces; wykonywany również po zmianach 2026-08-22 |
| `gradlew connectedDebugAndroidTest` | ostatni pełny przebieg: sukces, 14/14 testów na SM-A125F |
| `gradlew lintDebug` | ostatni pełny przebieg: sukces, 0 błędów |
| Rzeczywisty pakiet `MP+MT+MZ` | zaimportowany i aktywowany |
| Rzeczywisty MP/NCNN | inferencja i autotuning potwierdzone |
| Kaskada MP→ROI→MT | działa na rzeczywistym obrazie |
| `VEHICLE` + `VEHICLE_ROI` | potwierdzone wizualnie |
| `SceneChangeDetector` | potwierdzony w kilku przebiegach |
| Reset tracków po zmianie sceny | potwierdzony w logach |
| Reset pojedynczej starej ramki UI | mechanizm generacji sceny i lekkiego resetu wdrożony; końcowa kalibracja ruchu kamery pozostaje otwarta |
| Semantyka wyniku MZ bez nowej próby | wynik z pamięci jest jawnie odróżniony od świeżej inferencji MZ |
| Rzędowość i konsensus temporalny | implementacja i testy jednostkowe struktury wierszy wykonane; dalsza walidacja na materiale dwurzędowym trwa |
| Filtr klas MP | potwierdzony na rzeczywistych logach; COCO `[2,3,5,7]` |
| Polityka R0/R1/R2 | implementacja gotowa; R0 potwierdzone w `.alprsession` |
| Rozdzielenie normalnej konfiguracji i EXP | potwierdzone ręcznie |
| R0: MP pominięty | potwierdzone: 0 inferencji MP, 51 MT full-frame |
| Ręczny Start/Stop analizy | potwierdzony na urządzeniu |
| Czyszczenie podglądu po STOP | potwierdzone |
| Pomiar tylko w aktywnej sesji analizy | potwierdzony; przebieg kontrolny 41,718 s |
| `ExperimentSession` | wdrożony; ręczny STOP, ERROR, restart CameraX i eksport snapshotu potwierdzone |
| Opcjonalny `TimerConfig` | wdrożony; 15 s i 60 s potwierdzone, timer nie zeruje się przy restarcie CameraX |
| Dynamiczny katalog rozdzielczości kamery | potwierdzony; m.in. `640×480` i `4000×3000`, requested=actual |
| Raport rozdzielczości `selected/requested/actual` | potwierdzony w `.alprsession` |
| Galeria jako Bottom Sheet i pojedynczy Start/Stop | potwierdzone ręcznym przeglądem UI |
| Statystyki MZ bez sztucznych `0 ms` | potwierdzone; `mz_runs=2`, `character_inference.count=2` |
| Pomiar kontrolny wąskiego gardła | MT p50 ≈ 2973 ms, pipeline p50 ≈ 4634 ms na SM-A125F |
| `PreviewPlateTracker` | wersja wielotablicowa potwierdzona ręcznie; ramka PLATE pozostaje widoczna między inferencjami |
| `VEHICLE/VEHICLE_ROI` śledzone przez ruch tablicy | potwierdzone ręcznie |
| Animacja quada PLATE | interpolacja keypointów dostępna; końcowy test renderera po ostatniej zmianie otwarty |
| Audyt `PIPE/INF/AUX/OVH` | wykonany na urządzeniu; `OVH` <1%, główny koszt pomocniczy 12 MP stanowiła ręczna rotacja bitmapy |
| Parser raportu desktopowego | raport przyjęty |
| Bramka jakości bez ground truth | poprawnie odrzucona |


### Aktualizacja weryfikacji po serii ROI i bramce termicznej

- S1, jeden pojazd: wykonano po trzy sesje R0, R1 i R2;
- S2, dwa pojazdy: wykonano pilot R0/R1/R2 oraz pierwsze kontrolowane
  powtórzenie z wyrównanym stanem termicznym;
- w kontrolowanym S2 starty wynosiły `32,4–32,5°C`, `TH0`;
- R1 wykonywało dokładnie jedno ROI na iterację, R2 dokładnie dwa;
- w kontrolowanym S2 nie wystąpił żaden full-frame fallback w R1 ani R2;
- koszt MT i jego preprocessingu w R2 był praktycznie dwukrotnością R1;
- bezpośredni eksport `.alprsession` po zakończonym eksperymencie działa;
- timer pozostaje ustawiony po zakończeniu sesji;
- `ThermalMonitor` i `ThermalConfig` są obecne w repozytorium;
- `ThermalMonitor` odczytuje również `Thermal Headroom`; HEAD jest próbkowany
  nie częściej niż co 10 s i pozostaje wartością obserwacyjną, a nie kryterium
  startu eksperymentu;
- po wdrożeniu auto-zoomu dodano testy `AutoZoomControllerTest`,
  `AutoZoomOverlayTransformTest`, `AutoZoomTargetLockTest`,
  `PreviewTrackerDriftGuardTest` i `AutoZoomRecognitionMemoryTest`;
- pełne `testDebugUnitTest` oraz `lintDebug` po integracji auto-zoomu zakończyły
  się powodzeniem; najnowsza wersja auto-zoomu wymaga jeszcze końcowej
  weryfikacji kamera–monitor na fizycznym urządzeniu.


Aktualny artefakt debug:

```text
app/build/outputs/apk/debug/app-debug.apk
```

Po zmianie polityki cropów, bazowej serii R0/R1/R2 i końcowej walidacji warstwy
tracking/scene należy ponownie uruchomić pełny zestaw regresyjny przed zamrożeniem
wersji badawczej. Audyt czasu pipeline'u i optymalizację rotacji 12 MP uznaje się
za wykonane.

---


# 8. Otwarte zadania i ryzyka

## Najbliższe kroki po integracji auto-zoomu

1. wykonać końcową próbę urządzeniową pełnego cyklu auto-zoom:
   detekcja → zoom-in → świeże MT → wymuszona próba MZ → zoom-out → zachowanie
   geometrii i najlepszego tekstu przy `1x`;
2. sprawdzić blokadę celu na scenie z kilkoma tablicami oraz przy lekkim
   drżeniu telefonu;
3. rozdzielić w modelu danych `trackConfirmed`, `freshMzSuccessful`,
   `cropSupportsConsensus` i ręczną walidację, aby `confirmed` nie było
   interpretowane jako jakość konkretnego cropa;
4. rozszerzyć raport eksperymentalny o strukturalny snapshot termiczny
   START/STOP; obecnie BAT/TH/HEAD są dostępne w UI/logu, ale nie jako pełny
   kontrakt sesji;
5. zdecydować, czy druga kontrolowana seria S2 jest potrzebna po analizie
   zmienności pierwszego powtórzenia; trzecie powtórzenie wykonywać tylko przy
   wyniku odstającym lub niejednoznacznym;
6. po zamrożeniu auto-zoomu przejść do osobnych eksperymentów rozdzielczości
   źródła i wariantów runtime;
7. przed finalnym zamrożeniem wersji badawczej wykonać pełny zestaw regresyjny
   na JVM i urządzeniu.


## Backlog raportu badawczego

Raport ma wiązać trzy warstwy:

1. dokładną konfigurację eksportową modeli;
2. wykonanie na konkretnym telefonie;
3. skuteczność na materiale z ground truth.

Nie wolno mieszać wyniku jakościowego z samym confidence.

## Dwa scenariusze pomiarowe

### Replay kontrolowany

Stały, wersjonowany zbiór obrazów/cropów w niezmiennej kolejności.

Cel:

- parytet Python–Android;
- runtime;
- kwantyzacja;
- progi;
- pomiar czystej inferencji.

### Sesja live / camera-in-the-loop

Rzeczywista kamera, tracking, harmonogram, MP i temporalny konsensus.

Cel:

- czas do pierwszego wyniku;
- czas do potwierdzenia;
- stabilność;
- koszt całego pipeline'u;
- dropped frames;
- termika.

Wyników replay i live nie należy łączyć w jedną średnią.

## Protokół wydajności

- cold start raportować osobno;
- początek i koniec właściwego pomiaru wiązać z jawną sesją analizy;
- przygotowanie stanowiska przed naciśnięciem `Uruchom analizę` nie należy do czasu benchmarku;
- steady state mierzyć zegarem monotonicznym;
- raportować co najmniej p50, p90, p95 i p99;
- wykonywać kilka powtórzeń;
- przechowywać surowe trace'y;
- kontrolować temperaturę i stan baterii;
- nie nazywać wyniku „MLPerf”, jeżeli nie spełniono oficjalnych reguł MLPerf.

## Metryki jakości

### MT

- precision;
- recall;
- F1;
- mAP@0.5;
- mAP@0.5:0.95;
- błąd narożników;
- poprawność rektyfikacji.

### MZ

- exact match;
- CER;
- accuracy per znak;
- macierz pomyłek;
- udział cropów zakończonych odczytem.

### End-to-end

- exact match per unikalny track/tablicę;
- czas do pierwszego wyniku;
- czas do potwierdzenia;
- liczba prób MZ;
- udział tracków bez odczytu.

## Priorytet wysoki

- utrzymać diagnostyczny bilans `PIPE/INF/AUX/OVH` w trace/raporcie i uprościć
  HUD do wartości przydatnych podczas pracy (`PIPE`, `INF`, `AUX`);
- wykonać bazową serię R0/R1/R2 po optymalizacji rotacji CameraX, bez zmiany
  modeli i ich wejść;
- utrzymywać aktualną politykę cropów: nowy zapis przy zmianie tekstu,
  przejściu do stanu stabilnego, poprawie ostrości, poprawie
  `recognitionConfidence` o co najmniej `0,10` lub po interwale okresowym;
- wykonać końcowy test animowanego quada i kalibrację unieważniania sceny przy
  ruchu kamery;
- zweryfikować urządzeniowo wdrożony `AutoZoomController`, blokadę celu,
  pamięć numeru i powrót do `1x`; eksperyment auto-zoomu nadal traktować
  oddzielnie od bazowego R0/R1/R2;
- wykonać kontrolne przebiegi R1 i R2;
- wykonać pełne A/B/C strategii ROI:
  - R0 — MT pełna klatka;
  - R1 — MP→maks. 1 ROI→MT;
  - R2 — MP→maks. 2 ROI→MT;
- wykonać kilka powtórzeń każdego wariantu na identycznym materiale;
- porównać runtime'y NCNN, ONNX i TFLite;
- zamrozić protokół camera-in-the-loop;
- wykonać pełny benchmark z ground truth;
- porównać Android z referencyjną inferencją Python;
- po domknięciu bieżących zmian uruchomić pełny zestaw regresyjny.

## Priorytet średni

- dodać `scene_id` do raportu;
- raportować liczbę `scene_reset`;
- powiązać `track_id` ze sceną;
- rozważyć monotoniczne `track_id` w obrębie sesji;
- rozszerzyć `PlateObservation` i raport o strukturę wierszy, jeżeli będzie potrzebna
  do analizy wyników;
- rozważyć głosowanie temporalne po `(row, col)`;
- rozbudować metadane auto-zoomu w raporcie sesji; `camera_zoom_ratio` i
  `capture_source` są już zapisywane dla cropów, natomiast stan kontrolera,
  wynik blokady celu i pełny cykl zoomu wymagają agregacji na poziomie sesji;
- ograniczyć logi diagnostyczne;
- dopracować importer `.alprsession` po stronie Python;
- dodać jawną bramkę jakości dla INT8.

## Kierunki opcjonalne

- bezpośredni benchmark statycznych obrazów w aplikacji;
- NCNN/Vulkan po osobnym benchmarku;
- NNAPI/NPU, jeżeli runtime i urządzenie zapewnią stabilne wsparcie;
- tryb testowania wyłącznie MZ;
- porównanie kilku kompletnych kompozycji bez ponownego importu;
- uczony model temporalny jako oddzielny eksperyment;
- niezależne śledzenie czterech narożników quada pomiędzy inferencjami MT;
- wymuszenie konkretnego fizycznego teleobiektywu dopiero jako osobny, zależny od
  urządzenia eksperyment po wersji opartej na ogólnym `CameraControl` zoom.

---


## 2026-08-26 — auto-zoom celu, blokada tablicy, pamięć numeru i ograniczenie dryfu trackera

Problem:

- pierwsza koncepcja auto-zoomu nie rozdzielała dostatecznie zmiany skali kamery
  od rzeczywistej zmiany sceny;
- podczas zoom-in/zoom-out ramka i tekst mogły znikać albo zostać zastąpione
  świeżym, ale uboższym wynikiem MT;
- przy kilku tablicach konieczne było utrzymanie tożsamości dokładnie tej
  tablicy, która wywołała zbliżenie;
- tracker Preview mógł przy drżeniu telefonu akumulować błąd translacji;
- po zbliżeniu potrzebna była wymuszona świeża próba MZ na nowym cropie bez
  kasowania wcześniejszego konsensusu.

Rozwiązanie:

- dodano `AutoZoomController`, który obsługuje stany `DISABLED`, `READY`,
  `ZOOM_SETTLING`, `ZOOMED_RETRY` i `RETURNING`;
- domyślny żądany zoom wynosi `1,8x`, okres stabilizacji po zmianie skali
  `450 ms`, a maksymalny czas oczekiwania na poprawę wyniku przy zoomie
  `9 s`;
- kandydat do zoomu musi mieć co najmniej dwie obserwacje, poprawny quad i być
  mały albo mieć niski confidence; jedna tablica dostaje co najwyżej jedną
  próbę auto-zoomu w danej sesji kontrolera;
- `CameraController` wykonuje płynną zmianę `zoomRatio`, a po zoom-in ustawia
  punkt AF/AE/AWB w okolicy celu; powrót do `1x` ma osobną, wolniejszą animację;
- dodano osobną generację transformacji kamery. Wyniki rozpoczęte dla poprzedniej
  skali są odrzucane bez zerowania logicznej sceny;
- w `AlprPipeline` i `MobileAlprEngine` kontrolowany zoom jest maskowany przed
  `SceneChangeDetector`, a po zakończeniu transformacji geometria tracków jest
  przeliczana przez względny współczynnik zoomu;
- po zoom-in `PlateTrackCoordinator` przyznaje aktywnemu trackowi jedną świeżą
  próbę MZ niezależnie od normalnego harmonogramu, nadal wymagając poprawnej
  geometrii rektyfikacji;
- dodano `AutoZoomTargetLock` ze stanami `DISABLED`, `ACQUIRING`, `LOCKED`,
  `UNCERTAIN`, `LOST`. Blokada ocenia ruch, IoU, skalę, proporcje, confidence
  MT i prosty deskryptor wyglądu;
- aktywna blokada wyznacza własne ROI `ROI AUTO ZOOM`; w tym stanie okresowy
  full-frame fallback jest wyłączony, aby inne tablice nie przejęły próby MZ;
- dodano `PlateAppearanceDescriptor` i pamięć deskryptora wyglądu per track;
- dodano `AutoZoomRecognitionMemory`. Pusty albo słabszy wynik po zmianie skali
  nie usuwa ostatniego wiarygodnego numeru, a nowy tekst zastępuje pamięć tylko
  po przejściu konserwatywnej bramki jakości;
- `PreviewTrackerDriftGuard` zestawia ruch przyrostowy z ruchem względem ostatniej
  kotwicy MT i odrzuca aktualizacje o zbyt małym wsparciu, zbyt dużym skoku albo
  nadmiernym dryfie;
- `CapturedPlateItem` i `CropMiniReport` zapisują `camera_zoom_ratio` oraz
  `capture_source`; próba powstała podczas ponownego odczytu po zoomie może być
  oznaczona jako `auto_zoom_retry`;
- `CropSamplingPolicy` uwzględnia poprawę `recognitionConfidence` o co najmniej
  `0,10`, dzięki czemu lepszy crop po zoomie może zostać zachowany bez usuwania
  wcześniejszego materiału;
- UI otrzymał kontrolkę auto-zoomu, pulsujący stan aktywny oraz celownik
  nanoszony na zablokowaną tablicę;
- `ThermalMonitor` odczytuje dodatkowo `Thermal Headroom`; HEAD jest próbkowany
  najwyżej raz na `10 s` i pozostaje wskaźnikiem obserwacyjnym.

Semantyka `confirmed`:

- `CapturedPlateItem.confirmed` jest nadal kopią `stable` z temporalnego
  konsensusu całego tracku;
- nie oznacza, że konkretny crop samodzielnie potwierdził tekst i nie jest
  odpowiednikiem ground truth;
- bieżący crop może mieć `confirmed=true` nawet wtedy, gdy tekst pochodzi z
  wcześniejszego stanu tracku;
- docelowo należy rozdzielić `trackConfirmed`, `freshMzSuccessful`,
  `cropSupportsConsensus` i `manualVerificationStatus`.

Diagnostyka i UI:

- `DiagnosticsActivity` otrzymała tabelę MP/MT/MZ z runtime'em, precyzją,
  profilem wykonania, liczbą wątków i parametrami wejścia;
- trwały log jest prezentowany jako osobne wydarzenia z grupowaniem kolejnych
  eksperymentów R0/R1/R2;
- ekran główny zachowuje tryb immersyjny, skrócony HUD, timer, monitor termiczny
  i eksport sesji po zakończeniu eksperymentu;
- usunięto część nieużywanych zasobów starych przycisków galerii i dodano
  dedykowane zasoby auto-zoomu oraz tabel diagnostycznych.

Weryfikacja:

- dodano `AutoZoomControllerTest`;
- dodano `AutoZoomOverlayTransformTest`;
- dodano `AutoZoomTargetLockTest`;
- dodano `PreviewTrackerDriftGuardTest`;
- dodano `AutoZoomRecognitionMemoryTest`;
- rozszerzono testy `MotionIntensityFilter`, `CropSamplingPolicy`,
  `PlateTrackCoordinator`, `VehicleRoiSelector` i `MotionBoxTracker`;
- `testDebugUnitTest` po integracji auto-zoomu zakończył się powodzeniem;
- `lintDebug` po integracji zakończył się powodzeniem;
- `app/lint.xml` jawnie wycisza wyłącznie ostrzeżenia o świadomie przypiętych
  wersjach target API, Android Gradle Plugin i zależności;
- brak statusu CI na GitHubie nie oznacza błędu — repozytorium nie ma
  opublikowanego checka dla tego commita;
- najnowsza korekta pamięci numeru i pełny cykl kamera–zoom–MZ–powrót nie są
  jeszcze uznane za potwierdzone wizualnie na urządzeniu.

Pozostało:

- wykonać pełny test urządzeniowy auto-zoomu na jednej i kilku tablicach;
- sprawdzić zachowanie przy drżeniu oraz rzeczywistej zmianie kadru podczas
  zbliżenia;
- zweryfikować progi pamięci numeru na rzeczywistych pomyłkach MZ;
- rozszerzyć raport sesji o agregaty auto-zoomu i strukturalny snapshot
  termiczny START/STOP;
- rozdzielić semantykę stabilności tracku od wsparcia konkretnego cropa;
- dopiero po tej walidacji prowadzić osobny eksperyment „auto-zoom OFF vs ON”.

---

# 9. Zasady aktualizowania dziennika

Po większej zmianie należy dopisać wpis zawierający:

```text
### RRRR-MM-DD — nazwa zmiany

Problem:
- co wymagało rozwiązania;

Rozwiązanie:
- co zmieniono w kodzie i kontrakcie;

Decyzje:
- dlaczego wybrano takie podejście;

Weryfikacja:
- jakie testy/buildy wykonano i z jakim wynikiem;

Pozostało:
- czego wpis nie potwierdza lub co wymaga dalszej pracy.
```

Nie należy przedstawiać planu jako funkcji ukończonej. Wyniki testów na
lokalnym JVM, emulatorze, fizycznym telefonie i w aplikacji desktopowej należy
rozróżniać.

Wpisy historyczne mogą zawierać stan, który został później zmieniony. Nie
należy ich usuwać tylko dlatego, że następny wpis dokumentuje nowsze
rozwiązanie. Sekcje „Stos technologiczny”, „Weryfikacja” i „Otwarte zadania”
powinny natomiast reprezentować aktualny stan projektu.

---

# 10. Dokumenty powiązane

## Dokumenty znajdujące się w repozytorium

- `docs/model_package_v1.md` — opis formatu pojedynczego modelu i kompletnego
  pakietu;
- `docs/alpr-model-v1.schema.json` — schemat pojedynczego modelu;
- `docs/alpr-package-v1.schema.json` — schemat kompletnej paczki MT+MZ albo
  MP+MT+MZ;
- `docs/mobile_architecture.md` — opis potoku wykonawczego, trackingu, trybu
  eksperymentalnego i auto-zoomu; zsynchronizowany z kodem po audycie
  2026-08-26;
- `docs/mobile_research_export.md` — kontrakt eksportu `.alprsession`, paczki
  TeX i bieżącej semantyki metadanych cropów; zsynchronizowany z kodem po
  audycie 2026-08-26;
- `docs/alpr-mobile-research-bundle-v1.schema.json` — schemat manifestu pełnego
  eksportu badawczego;
- `docs/model_package_test_strategy.md` — strategia fixtures, parytetu,
  runtime'ów i bramek akceptacyjnych paczki modeli.

## Materiały robocze poza aktualnym katalogiem `docs/`

W historii projektu występowały również lokalne dokumenty metodologiczne i
handoff programu Python. Jeżeli mają wejść do repozytorium jako źródła
wersjonowane, należy dodać je jawnie zamiast odwoływać się wyłącznie do ścieżek
lokalnego komputera.

---

# 11. Słownik skrótów, oznaczeń i identyfikatorów

Słownik obejmuje skróty, oznaczenia techniczne oraz nazwy metryk używane w
dzienniku, interfejsie aplikacji i generowanych raportach badawczych.

| Skrót / oznaczenie | Znaczenie |
| --- | --- |
| **A/B** | porównanie dwóch kontrolowanych wariantów tego samego eksperymentu |
| **ABI** | *Application Binary Interface* — interfejs binarny platformy, np. `arm64-v8a` |
| **AE** | *Auto Exposure* — automatyczna ekspozycja kamery |
| **AF** | *Auto Focus* — automatyczne ustawianie ostrości |
| **AWB** | *Auto White Balance* — automatyczny balans bieli kamery |
| **ALPR** | *Automatic License Plate Recognition* — automatyczne rozpoznawanie tablic rejestracyjnych |
| **ANPR** | *Automatic Number Plate Recognition* — alternatywna nazwa systemów ALPR |
| **API** | *Application Programming Interface* — interfejs programistyczny |
| **APK** | *Android Package Kit* — instalacyjny pakiet aplikacji Android |
| **AUX** | *auxiliary time* — suma jawnie zmierzonych operacji pomocniczych poza samą inferencją modeli |
| **AUTO-ZOOM** | automatyczna, kontrolowana zmiana `zoomRatio` kamery wykonywana dla wybranego tracku tablicy w celu uzyskania lepszego cropu i ponownej próby MZ |
| **BAT** | temperatura baterii raportowana przez Androida, podawana w `°C` |
| **CAM** | łączny zmierzony czas przygotowania obrazu kamery w pipeline |
| **CAM_BITMAP / TO_BITMAP** | czas konwersji `ImageProxy` do `Bitmap` |
| **CAM_ROT / ROTATE** | czas ręcznej rotacji bitmapy obrazu |
| **CER** | *Character Error Rate* — współczynnik błędów znakowych oparty na odległości edycyjnej |
| **CNN** | *Convolutional Neural Network* — konwolucyjna sieć neuronowa |
| **CVPR** | *Conference on Computer Vision and Pattern Recognition* — konferencja IEEE z obszaru widzenia komputerowego |
| **COCO** | *Common Objects in Context* — popularny zbiór i zestaw klas detekcji obiektów |
| **CPU** | *Central Processing Unit* — główny procesor urządzenia |
| **CRNN** | *Convolutional Recurrent Neural Network* — konwolucyjno-rekurencyjna sieć do rozpoznawania sekwencji |
| **CTC** | *Connectionist Temporal Classification* — funkcja/strategia uczenia i dekodowania sekwencji |
| **CSV** | *Comma-Separated Values* — tekstowy format danych tabelarycznych |
| **CTA** | *Call To Action* — główny przycisk akcji w interfejsie |
| **DOI** | *Digital Object Identifier* — trwały identyfikator publikacji naukowej |
| **ECCV** | *European Conference on Computer Vision* — europejska konferencja z obszaru widzenia komputerowego |
| **EURASIP** | *European Association for Signal Processing* — organizacja/wydawca występujący w nazwie cytowanego czasopisma |
| **DROP** | liczba klatek odrzuconych/pominiętych przez analizator podczas aktywnej sesji |
| **EXP** | tryb eksperymentalny aplikacji |
| **F1** | średnia harmoniczna precision i recall |
| **G0–G9** | kolejne bramki akceptacyjne w strategii testowania pakietów modeli |
| **FP32** | 32-bitowa reprezentacja zmiennoprzecinkowa |
| **GPU** | *Graphics Processing Unit* — procesor graficzny wykorzystywany m.in. jako delegat inferencji |
| **HEAD** | *Thermal Headroom* — bezwymiarowy wskaźnik zapasu termicznego Androida; nie ma jednostki fizycznej, `1,0` odpowiada progowi `THERMAL_STATUS_SEVERE`, a wartości mogą przekraczać `1,0` |
| **HUD** | *Head-Up Display* — bieżący panel telemetryczny nad podglądem kamery |
| **INF** | suma czasów właściwej inferencji MP, MT i MZ w danej iteracji |
| **INT8** | 8-bitowa reprezentacja całkowitoliczbowa używana m.in. w modelach skwantyzowanych |
| **IEEE** | *Institute of Electrical and Electronics Engineers* — organizacja naukowo-techniczna i wydawca wielu cytowanych publikacji |
| **IJCNN** | *International Joint Conference on Neural Networks* — konferencja dotycząca sieci neuronowych |
| **ITSC** | *IEEE Intelligent Transportation Systems Conference* — konferencja dotycząca inteligentnych systemów transportowych |
| **IoU** | *Intersection over Union* — miara nakładania się dwóch obszarów |
| **JNI** | *Java Native Interface* — interfejs pomiędzy Javą a kodem natywnym C/C++ |
| **JPEG** | stratny format zapisu obrazów |
| **JIVP** | *Journal on Image and Video Processing* — czasopismo z obszaru przetwarzania obrazu i wideo |
| **JSON** | *JavaScript Object Notation* — tekstowy format danych strukturalnych |
| **JVM** | *Java Virtual Machine* — maszyna wirtualna uruchamiająca m.in. testy jednostkowe Java |
| **LiteRT / TFLite** | mobilny runtime modeli wywodzący się z TensorFlow Lite |
| **LOCK** | blokada tożsamości celu auto-zoomu; stan logiczny utrzymujący tę samą tablicę mimo zmiany skali i ruchu |
| **mAP** | *mean Average Precision* — średnia precyzja detekcji dla zadanego zakresu progów IoU |
| **mAP50-95** | mAP liczony dla progów IoU od 0,50 do 0,95 |
| **MB** | megabajt; w dzienniku używany przy orientacyjnych rozmiarach danych i plików |
| **ML** | *Machine Learning* — uczenie maszynowe |
| **MLPerf** | rodzina standardowych benchmarków uczenia maszynowego MLCommons |
| **MP** | model pojazdów — opcjonalny etap wykrywający pojazdy i wyznaczający ROI |
| **MT** | model tablic — YOLO Pose wykrywający tablicę i jej narożniki |
| **MT_FULL** | liczba uruchomień MT na pełnej klatce |
| **MT_INF** | łączny czas inferencji MT w danej iteracji; przy kilku ROI jest sumą kilku uruchomień |
| **MT_PRE** | łączny czas preprocessingu wejść MT |
| **MT_ROI** | liczba uruchomień MT na wycinkach ROI |
| **MZ** | model znaków — detektor YOLO klasyfikujący poszczególne znaki tablicy |
| **MZ_RUNS** | liczba faktycznych uruchomień MZ |
| **NCHW** | układ tensora: batch, kanały, wysokość, szerokość |
| **NCNN** | mobilny runtime inferencji Tencent; w aplikacji wykonywany przez JNI na CPU |
| **NHWC** | układ tensora: batch, wysokość, szerokość, kanały |
| **NMS** | *Non-Maximum Suppression* — usuwanie nakładających się detekcji |
| **NNAPI** | *Android Neural Networks API* — systemowe API Androida do akcelerowanej inferencji |
| **NPU** | *Neural Processing Unit* — wyspecjalizowany akcelerator sieci neuronowych |
| **OCR** | *Optical Character Recognition* — ogólna klasa metod rozpoznawania tekstu; bieżący MZ nie jest klasycznym OCR |
| **ONNX** | *Open Neural Network Exchange* — format wymiany modeli sieci neuronowych |
| **OVH** | *overhead* — reszta `PIPE - (INF + AUX)`, czyli czas nieprzypisany do jawnie mierzonych etapów |
| **p50 / p90 / p95 / p99** | percentyle rozkładu czasu; np. p90 oznacza wartość, poniżej której znajduje się 90% obserwacji |
| **PIPE** | całkowity czas wykonania jednej mierzonej iteracji pipeline'u |
| **PLATE** | identyfikator elementu overlayu reprezentującego tablicę |
| **POST** | postprocessing — przetwarzanie surowego wyjścia modelu |
| **PRE** | preprocessing — przygotowanie obrazu/tensora przed inferencją |
| **PSS** | *Proportional Set Size* — miara pamięci procesu z proporcjonalnym udziałem pamięci współdzielonej |
| **R-CNN** | *Region-based Convolutional Neural Network* — rodzina metod detekcji obiektów opartych na regionach i CNN |
| **R0** | wariant eksperymentalny: MT analizuje pełną klatkę, MP jest pomijany |
| **R1** | wariant eksperymentalny: MP → maksymalnie 1 ROI → MT |
| **R2** | wariant eksperymentalny: MP → maksymalnie 2 ROI → MT |
| **RAM** | *Random Access Memory* — pamięć operacyjna |
| **RECT** | rektyfikacja perspektywy tablicy przed MZ |
| **RGB / RGBA** | reprezentacja obrazu kanałami czerwonym, zielonym, niebieskim i opcjonalnie alfa |
| **ROI** | *Region of Interest* — wybrany fragment obrazu przekazywany do dalszej analizy |
| **ROT** | kąt rotacji obrazu raportowany w diagnostyce |
| **SRC** | *source* — oznaczenie rzeczywistych wymiarów źródłowej klatki w logu diagnostycznym, np. `SRC=3000x4000` |
| **S1** | scena eksperymentalna z jednym pojazdem użyta do pilotażu R0/R1/R2 |
| **S2** | scena eksperymentalna z dwoma pojazdami użyta do rzeczywistego porównania limitu 1 i 2 ROI |
| **SHA-256** | kryptograficzna funkcja skrótu używana do identyfikacji i walidacji artefaktów |
| **SORT** | *Simple Online and Realtime Tracking* — rodzina prostych metod śledzenia obiektów |
| **SUM** | diagnostyczna suma jawnie mierzonych etapów; w bieżącym bilansie odpowiada `INF + AUX` |
| **T0–T10** | poziomy strategii testów paczek od preflight do testów eksportu/live |
| **TeX** | system składu dokumentów używany do generowania materiałów do pracy |
| **TH** | *Thermal Status* — systemowa, dyskretna ocena stanu termicznego Androida od 0 do 6 |
| **TH0** | `THERMAL_STATUS_NONE` — brak zgłoszonego ograniczenia termicznego |
| **TH1** | `THERMAL_STATUS_LIGHT` — lekki poziom obciążenia termicznego raportowany przez Androida |
| **TFLITE** | zapis etykiety runtime'u spotykany w logach/UI; odpowiada TFLite/LiteRT |
| **TPAMI** | *IEEE Transactions on Pattern Analysis and Machine Intelligence* — czasopismo IEEE cytowane w przeglądzie literatury |
| **TFLite** | *TensorFlow Lite*, obecnie rozwijany jako LiteRT; mobilny runtime inferencji |
| **TTL** | *Time To Live* — sztuczny czas życia obiektu/stanu; w aktualnej prezentacji VEHICLE/ROI nie jest używany do arbitralnego wygaszania |
| **UI** | *User Interface* — interfejs użytkownika |
| **UTC** | *Coordinated Universal Time* — uniwersalny czas koordynowany |
| **UUID** | *Universally Unique Identifier* — identyfikator o bardzo małym prawdopodobieństwie kolizji |
| **VEHICLE** | identyfikator overlayu ramki pojazdu z MP |
| **VEHICLE_ROI** | identyfikator overlayu obszaru ROI wyznaczonego dla pojazdu |
| **V-LPDR** | nazwa metody z cytowanej publikacji dotyczącej wspólnej detekcji, śledzenia i rozpoznawania tablic w wideo |
| **LSV-LP** | nazwa zbioru/metody z pracy *Large-Scale Video-Based License Plate Detection and Recognition* |
| **YUV / YUV_420_888** | format obrazu kamery rozdzielający luminancję i składowe chrominancji; `YUV_420_888` jest formatem CameraX/ImageAnalysis |
| **YOLO** | *You Only Look Once* — rodzina modeli detekcyjnych używana przez MP, MT i MZ |
| **ZIP** | format archiwum używany jako baza `.alprmodel` i eksportów badawczych |

### Skala statusu TH

| TH | Nazwa Androida | Interpretacja robocza |
| ---: | --- | --- |
| 0 | `NONE` | brak zgłoszonego obciążenia termicznego |
| 1 | `LIGHT` | lekkie obciążenie termiczne |
| 2 | `MODERATE` | umiarkowane |
| 3 | `SEVERE` | poważne ograniczenie termiczne |
| 4 | `CRITICAL` | stan krytyczny |
| 5 | `EMERGENCY` | stan awaryjny |
| 6 | `SHUTDOWN` | próg wymagający wyłączenia urządzenia |

