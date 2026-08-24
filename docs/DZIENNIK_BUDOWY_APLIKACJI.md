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

Stan na 2026-08-24:

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

---

# 7. Weryfikacja

Stan na 2026-08-24:

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
| Reset pojedynczej starej ramki UI | wymaga końcowego testu wizualnego |
| Badge numeru przy aktywnym MP | problem zdiagnozowany, finalna weryfikacja otwarta |
| Ulepszona obsługa tablic dwurzędowych | analiza zakończona, implementacja otwarta |
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
| Parser raportu desktopowego | raport przyjęty |
| Bramka jakości bez ground truth | poprawnie odrzucona |

Aktualny artefakt debug:

```text
app/build/outputs/apk/debug/app-debug.apk
```

Po zakończeniu zmian dotyczących badge'ów i kolejności znaków należy ponownie
uruchomić pełny zestaw regresyjny przed zamrożeniem wersji badawczej.

---

# 8. Otwarte zadania i ryzyka

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

- zakończyć poprawkę badge'a tekstu tablicy;
- wykonać wizualny test resetu sceny obejmujący UI;
- przenieść `reading_order.py` do Androida;
- poprawić postprocessing układów wielorzędowych;
- dodać testy regresji tablic jedno- i dwurzędowych;
- wykonać kontrolne przebiegi R1 i R2;
- wykonać pełne A/B/C strategii ROI:
  - R0 — MT pełna klatka;
  - R1 — MP→maks. 1 ROI→MT;
  - R2 — MP→maks. 2 ROI→MT;
- wykonać kilka powtórzeń każdego wariantu na identycznym materiale;
- porównać runtime'y NCNN, ONNX i TFLite;
- zamrozić protokół camera-in-the-loop;
- wykonać pełny benchmark z ground truth;
- porównać Android z referencyjną inferencją Python.

## Priorytet średni

- dodać `scene_id` do raportu;
- raportować liczbę `scene_reset`;
- powiązać `track_id` ze sceną;
- rozważyć monotoniczne `track_id` w obrębie sesji;
- rozważyć strukturę rzędów w `PlateObservation`;
- rozważyć głosowanie temporalne po `(row, col)`;
- ograniczyć logi diagnostyczne;
- dopracować importer `.alprsession` po stronie Python;
- dodać jawną bramkę jakości dla INT8.

## Kierunki opcjonalne

- bezpośredni benchmark statycznych obrazów w aplikacji;
- NCNN/Vulkan po osobnym benchmarku;
- NNAPI/NPU, jeżeli runtime i urządzenie zapewnią stabilne wsparcie;
- tryb testowania wyłącznie MZ;
- porównanie kilku kompletnych kompozycji bez ponownego importu;
- uczony model temporalny jako oddzielny eksperyment.

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

- `docs/model_package_v1.md` — format pojedynczego i kompletnego pakietu;
- `docs/alpr-model-v1.schema.json` — schemat pojedynczego modelu;
- `docs/alpr-package-v1.schema.json` — schemat pakietu MT+MZ albo MP+MT+MZ;
- `docs/mobile_architecture.md` — potok wykonawczy i raportowanie;
- `docs/mobile_research_export.md` — kontrakt pełnego eksportu i TeX;
- `docs/alpr-mobile-research-bundle-v1.schema.json` — schemat eksportu
  badawczego;
- `docs/model_package_test_strategy.md` — strategia fixture, parytetu,
  runtime'ów i bramek akceptacyjnych;
- `docs/siatka_eksperymentow_mobilnych_alpr.md` — metodyka eksperymentów i
  rankingu;
- `docs/specyfikacja_agenta_aplikacji_mobilnej_alpr.md` — wymagania klienta
  mobilnego;
- `docs/alpr_python_exporter_handoff.md` — kontrakt Python → Android;
- `docs/eksport_mobilny_kwantyzacja.md` — uwagi dotyczące wariantów
  eksportowych i kwantyzacji;
- `docs/podbudowa_literaturowa_metodyki_testow_alpr.md` — podbudowa
  literaturowa metodologii pomiarów.
