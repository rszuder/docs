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

## 2. Założenia aplikacji

Aplikacja jest demonstratorem i stanowiskiem pomiarowym mobilnego systemu
automatycznego rozpoznawania tablic rejestracyjnych. Nie trenuje modeli i nie
importuje checkpointów PyTorch `best.pt`. Otrzymuje gotowe pakiety
`.alprmodel` wyeksportowane przez macierzystą aplikację Python.

Docelowy przepływ:

```text
obraz CameraX
  -> opcjonalna detekcja pojazdu
  -> MT: YOLO Pose — detekcja tablicy i czterech narożników
  -> rektyfikacja perspektywy
  -> MZ: YOLO Character Detection — detekcja i klasyfikacja znaków
  -> ustalenie kolejności znaków
  -> stabilizacja wyniku między klatkami
  -> prezentacja i raport pomiarowy
```

Terminologia stosowana w projekcie:

- **MT** oznacza model YOLO Pose wykrywający tablicę jako obiekt oraz jej
  cztery narożniki;
- **MZ** oznacza model detekcyjny YOLO, w którym klasami detekcji są
  poszczególne znaki; wynik tekstowy powstaje dopiero po posortowaniu ramek
  znaków w kolejności odczytu;
- **OCR** oznacza odrębną klasę metod rozpoznawania tekstu, np. model
  sekwencyjny CRNN/CTC albo usługę OCR. Obecny pipeline Androida nie zawiera
  takiego modelu.

Określenia „recognizer” i „OCR” mogą pojawiać się w omówieniu literatury,
jeżeli tak nazywają swoje komponenty autorzy publikacji. Nie są one używane
jako zamienniki nazwy MZ ani jako opis jego architektury.

Najważniejszą granicą systemu jest kontrakt `.alprmodel`. Aplikacja desktopowa
odpowiada za trening, eksport i kalibrację. Android odpowiada za ponowną
walidację pakietu, wybór runtime'u, inferencję oraz pomiary na urządzeniu.

## 3. Stos technologiczny

Stan na 2026-08-20:

| Obszar | Rozwiązanie |
| --- | --- |
| Język | Java 11 |
| Minimalny Android | API 29 |
| Docelowy Android | API 36 |
| Kamera | CameraX |
| Główny runtime | LiteRT/TFLite 1.4.1 |
| Akceleracja | LiteRT GPU 1.4.2 |
| Runtime kontrolny | ONNX Runtime Android 1.26.0 |
| ABI | `arm64-v8a`, `armeabi-v7a` |
| Format wymiany | ZIP z rozszerzeniem `.alprmodel` |
| Schemat modelu | `alpr.model.v1` |
| Schemat kompletu | `alpr.package.v1` |
| Schemat raportu | `alpr.mobile_benchmark_report.v1` |

NCNN może znajdować się w pakiecie, ale jego wykonanie wymaga przyszłego
adaptera JNI. Pakiety zawierające NCNN są walidowane i przechowywane.

## 4. Podział kodu

| Pakiet | Odpowiedzialność |
| --- | --- |
| `camera` | uruchomienie CameraX i dostarczanie klatek |
| `model` | manifesty, walidacja ZIP, rejestr i aktywacja modeli |
| `inference` | wspólne API backendów, LiteRT i ONNX Runtime |
| `vision` | preprocessing, dekoder YOLO, NMS, geometria i rektyfikacja |
| `pipeline` | wykonanie MT → rektyfikacja → MZ i stabilizacja wyniku |
| `autotune` | porównanie wariantów i profili wykonania na urządzeniu |
| `metrics` | ślady inferencji, statystyki i raport benchmarkowy |
| `ui` | overlay detekcji i prezentacja aktywnego kompletu |

`MainActivity` pełni rolę koordynatora UI. Logika manifestów, inferencji,
autotuningu i raportowania pozostaje w wyspecjalizowanych klasach.

## 5. Chronologia budowy

### 2026-08-18 — pierwsza zarejestrowana wersja kompletnego klienta

Commit bazowy:

```text
2696fbf Implement mobile ALPR pipeline and model runtimes
```

Zakres:

- skonfigurowano projekt Android i zależności ML;
- dodano podgląd oraz analizę klatek przez CameraX;
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
- dla MT wdrożono odczyt czterech keypointów oraz rektyfikację perspektywy;
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

### 2026-08-19 — kompletny pakiet ALPR MT+MZ

Problem:

Pojedynczy `alpr.model.v1` opisuje jeden model, natomiast gotowy system ALPR
wymaga co najmniej pary MT+MZ. Oddzielny import obu plików zwiększał ryzyko
pomylenia ról lub aktywowania niespójnej pary.

Rozwiązanie:

- dodano rozpoznawanie nadrzędnego schematu `alpr.package.v1`;
- dodano struktury `AlprPackageManifest`, `AlprPackageModelEntry` i
  `PipelineStage`;
- dodano dispatcher rozróżniający pakiet pojedynczego modelu i komplet MT+MZ;
- kompletny pakiet otrzymał limity 640 wpisów i 1 GiB po rozpakowaniu;
- zaostrzono walidację ścieżek ZIP: tylko względne ścieżki POSIX, bez `..`,
  ścieżek absolutnych, separatorów Windows i duplikatów;
- sprawdzane są SHA-256 zagnieżdżonych `.alprmodel` oraz obu manifestów
  bocznych;
- każdy zagnieżdżony model przechodzi ponownie pełną walidację
  `alpr.model.v1`;
- rola, zadanie i `model_id` muszą być spójne na wszystkich poziomach;
- manifest boczny musi odpowiadać manifestowi wewnątrz zagnieżdżonego ZIP;
- utworzono jeden trwały rekord kompletu i dwa rekordy modeli logicznych;
- aktywacja MT i MZ odbywa się wspólnie jednym zapisem preferencji;
- zachowano kompatybilność z importem pojedynczego modelu jako trybem
  częściowym;
- pipeline wykonawczy wykorzystuje istniejącą sekwencję MT → rektyfikacja → MZ.

Decyzja architektoniczna:

Najpierw walidowane są oba modele. Dopiero potem powstaje rekord kompletnego
pakietu i możliwa jest jego aktywacja. Uszkodzony lub niekompletny zestaw nie
jest oznaczany jako gotowy pipeline.

### 2026-08-19 — rozdzielenie parametrów MT i MZ

Doprecyzowano, że komplet wdrożeniowy nie oznacza wspólnej konfiguracji
inferencji. Aplikacja niezależnie dla MT i MZ rozwiązuje:

- rozmiar wejścia `width × height`;
- layout, typ danych, skalę i offset;
- `confidence_threshold` i `iou_threshold`;
- dekoder, liczbę klas i keypointy;
- wariant, runtime, precyzję, delegata i liczbę wątków;
- etykiety klas.

Przykładowo poprawny jest komplet `MT 640×640` oraz `MZ 416×416`, a także
połączenie TFLite/GPU dla MT z ONNX/CPU dla MZ.

UI zostało zmienione tak, aby prezentować efektywnie wybrany wariant i jego
parametry, a nie wyłącznie wartości domyślne manifestu. Wyświetlane są także
dostępne warianty, rodzina YOLO, liczba parametrów, metryka `mAP50-95` i data
eksportu, jeżeli eksporter je podał.

### 2026-08-19 — polityka FP32 i INT8

Problem:

Pierwsza wersja autotuningu wybierała wariant wyłącznie według opóźnienia.
Mogła więc automatycznie wybrać INT8 bez potwierdzenia wpływu kwantyzacji na
jakość rozpoznawania.

Rozwiązanie:

- autotuner nadal mierzy warianty kwantyzowane;
- jeżeli istnieje wykonywalny FP32, tylko warianty FP32 mogą zostać wybrane
  automatycznie;
- fallback jawnie preferuje TFLite FP32, następnie ONNX FP32, a dopiero potem
  inne wykonywalne warianty;
- profil zapisuje nazwę polityki wyboru oraz liczbę warmupów i pomiarów;
- promocja INT8 pozostaje zadaniem przyszłej bramki jakościowej wykorzystującej
  dane z ground truth.

Decyzja badawcza:

Szybszy wariant nie jest automatycznie lepszym wariantem. Zysk czasu lub
pamięci po kwantyzacji musi zostać zestawiony z CER, exact match i metrykami
detekcji.

### 2026-08-19 — ujednolicenie raportu z aplikacją desktopową

Raport Androida zmieniono na schemat:

```text
alpr.mobile_benchmark_report.v1
```

Raport zawiera:

- `package_id` i identyfikator kombinacji wariantów;
- osobne rekordy `execution.plate` i `execution.character`;
- identyfikatory modeli i fingerprinty;
- wybrane warianty, runtime'y, precyzje, delegaty i liczbę wątków;
- efektywne wejścia oraz progi obu modeli;
- p50, p90 i p95 dla MT, MZ i całego pipeline'u;
- peak PSS procesu i rozmiar źródłowego pakietu;
- dane urządzenia, Androida i wersji aplikacji;
- profile autotuningu;
- statusy, błędy runtime i pełne ślady klatek;
- CSV z jednym wierszem na przetworzoną klatkę.

Metryki `CER`, exact match i `success_rate` nie są wyliczane z samej sesji
kamery. Raport jawnie oznacza brak ground truth. Dzięki temu desktop może
zaimportować pomiary czasu i pamięci, ale poprawnie odrzucić raport jako dowód
jakości do czasu wykonania kontrolowanego benchmarku referencyjnego.

Test integracyjny z istniejącym parserem desktopowym potwierdził:

- poprawny odczyt `package_id` i kombinacji wariantów;
- poprawny odczyt p95 pipeline'u;
- poprawny odczyt RAM i rozmiaru pakietu;
- świadome odrzucenie rankingu bez metryk end-to-end.

### 2026-08-20 — decyzja badawcza: adaptacyjna kaskada wideo MT–MZ

Problem:

Pierwotny pipeline wykonuje dla każdej przyjętej klatki pełną sekwencję
MT → rektyfikacja → MZ. Zapewnia to małe opóźnienie pierwszej próby odczytu,
ale wielokrotnie uruchamia kosztowny MZ dla niemal identycznych wycinków.
Rozważono dwa podstawowe warianty alternatywne:

1. zgromadzenie konfigurowalnej liczby `N` klatek, a następnie jednorazowe
   wykonanie MZ dla najlepszego wycinka;
2. synchroniczne wykonanie MT i MZ niezależnie dla każdej klatki.

Przegląd literatury:

- V-LPDR łączy detekcję, śledzenie oraz jakościowo sterowaną rekomendację
  klatek do rozpoznania. Autorzy wskazują, że tracker powinien pośredniczyć
  między detektorem a ich modułem rozpoznającym, a do rozpoznania powinny
  trafiać wybrane klatki wysokiej jakości, nie wszystkie obrazy strumienia
  [L1]. W aplikacji rolę konsumenta wybranych wycinków pełni MZ oparty na
  detekcji YOLO, a nie model OCR. Jest to najbliższy problemowi aplikacji
  model referencyjny na poziomie organizacji potoku.
- Zastosowanie redundancji czasowej w ALPR zwiększyło współczynnik poprawnego
  rozpoznania o 15,3 punktu procentowego względem wariantu bazowego, przy
  deklarowanej szybkości 34 FPS [L2]. Wynik potwierdza, że decyzji nie należy
  opierać na pojedynczej, przypadkowej klatce.
- Character Time-series Matching wraz z adaptacyjną korekcją obrotu osiągnął
  96,7% dokładności na UFPR-ALPR i 47,56 FPS w środowisku z GPU [L3]. Autorzy
  agregują pewności znaków w obrębie tracku, co uzasadnia głosowanie
  per znak zamiast wyboru całego napisu wyłącznie według średniego confidence.
  Wynik wydajnościowy nie może być bezpośrednio przeniesiony na telefon,
  ponieważ uzyskano go na układzie NVIDIA RTX A5000.
- MFLPR-Net wykorzystuje informacje z sąsiednich klatek i został oceniony na
  zbiorze LSV-LP obejmującym 1 402 filmy, 401 347 klatek i 364 607 adnotowanych
  tablic [L4]. Potwierdza to zasadność wykorzystania kontekstu czasowego, ale
  oznacza osobny model temporalny, dodatkowy trening i większy koszt wdrożenia.
- W systemie edge o strukturze detektor–tracker–OCR moduł `WhenToRead`
  uruchamiał zewnętrzny model OCR tylko po istotnej poprawie metryki jakości.
  Zwiększył średnią szybkość z 10,40 do 24,69 FPS i poprawił trafność odczytu
  znalezionych znaczników z 48,3% do 64,6% [L5]. Przedmiotem badania były
  znaczniki identyfikacyjne bydła, a nie tablice rejestracyjne, dlatego wynik
  stanowi dowód dla wzorca wykonawczego edge, a nie dla zastosowanego w
  aplikacji modelu MZ i nie jest bezpośrednim benchmarkiem ALPR.
- Selekcja dokładnie jednej klatki na pojazd może być około trzykrotnie
  szybsza od analizy klatka po klatce [L6], lecz badany wariant zakłada kamerę
  statyczną, ruch jednokierunkowy i przekraczanie zdefiniowanej linii. Założenia
  te nie odpowiadają kamerze telefonu poruszanej przez użytkownika.
- Trackery oparte wyłącznie na IoU są wrażliwe na duże przesunięcie obrazu
  wywołane ruchem kamery. BoT-SORT kompensuje globalny ruch kamery przed
  kojarzeniem detekcji i pokazuje, że dla zastosowań o ograniczonych zasobach
  samo IoU pozostaje dobrym wyborem po poprawieniu predykcji położenia [L7].

Porównanie rozważanych modeli wykonania:

| Model wykonania | Opóźnienie pierwszego wyniku | Koszt MZ | Odporność na słabą klatkę | Ocena dla aplikacji |
| --- | --- | --- | --- | --- |
| MT → MZ dla każdej klatki | najmniejsze | najwyższy | mała | wariant bazowy |
| sztywne oczekiwanie na `N` klatek | zależne od `N` | niski | średnia lub duża | zbędne opóźnienie |
| dokładnie jedna wybrana klatka | średnie | najniższy | zależna od selektora | zbyt kruche przy ruchu telefonu |
| tracker + adaptacyjne `WhenToRead` | małe dzięki early exit | ograniczony | duża | wariant docelowy |
| uczony model temporalny | zależne od modelu | duży | potencjalnie największa | poza zakresem pierwszego wdrożenia |

Wniosek i decyzja architektoniczna:

Jako docelowy model wykonania przyjęto adaptacyjną kaskadę typu
`detect–track–when-to-read`, a nie jeden z dwóch wariantów skrajnych.
Planowany przepływ ma następującą postać:

```text
klatka -> MT -> deduplikacja -> kompensacja ruchu kamery -> track tablicy
                                                               |
                                                    ocena jakości wycinka
                                                               |
                                    MZ dla pierwszego dobrego lub wyraźnie
                                             lepszego wycinka danego tracku
                                                               |
                                      ważony konsensus znaków + early exit
                                                               |
                                                  stabilny wynik w interfejsie
```

Dla każdego tracku należy przechowywać wyłącznie ograniczony zbiór małych,
zrektyfikowanych wycinków, nie pełne klatki. Wstępnie przyjęto maksimum 2–3
najlepszych cropów. Ocena jakości powinna uwzględniać co najmniej confidence
MT, minimalną pewność narożników, powierzchnię tablicy, ostrość, położenie
względem krawędzi i deformację perspektywiczną. MZ powinien zostać uruchomiony:

- natychmiast dla pierwszego wycinka przekraczającego próg jakości;
- ponownie po istotnej poprawie jakości względem najlepszego wycinka tracku;
- po niskiej pewności wcześniejszych detekcji znaków MZ;
- przed wygaśnięciem tracku lub jego budżetu czasu.

Liczba klatek ustawiana przez użytkownika będzie górnym limitem materiału
dowodowego, a nie obowiązkową liczbą klatek do odczekania. W podstawowym UI
preferowane są profile `Szybki`, `Zrównoważony` i `Dokładny`; dokładne limity,
progi i budżety czasowe muszą zostać wyznaczone eksperymentalnie na urządzeniu
docelowym. MT i MZ mogą pozostać wykonywane szeregowo przez jeden executor,
aby uniknąć konkurencji o delegata GPU, mimo rozdzielenia ich harmonogramów.

Planowany podział odpowiedzialności:

- `PlateDetector` — MT i końcowa deduplikacja;
- `PlateTrackManager` — cykl życia tracków i kompensacja ruchu kamery;
- `CropQualityEvaluator` — porównywalna ocena wycinków;
- `CharacterDetectionScheduler` — decyzja o uruchomieniu i budżet MZ;
- `TemporalCharacterAggregator` — konsensus detekcji znaków i early exit;
- `MainActivity` — wyłącznie prezentacja stanu i konfiguracja profilu.

Status realizacji:

Decyzja została wdrożona w pierwszej wersji opisanej w kolejnym wpisie.
Zaimplementowano tracking sterujący MZ, ocenę geometrii, adaptacyjny budżet
prób oraz konsensus detekcji per znak. Pełna kompensacja globalnego ruchu
kamery i bufor kilku cropów ocenianych również pod kątem ostrości pozostają
zadaniami dalszymi.

Literatura:

- [L1] C. Zhang, Q. Wang, X. Li, „V-LPDR: Towards a unified framework for
  license plate detection, tracking, and recognition in real-world traffic
  videos”, *Neurocomputing*, vol. 449, s. 189–206, 2021.
  DOI: `10.1016/j.neucom.2021.03.103`; ISSN: `0925-2312`.
- [L2] G. R. Gonçalves, D. Menotti, W. R. Schwartz, „License Plate
  Recognition Based on Temporal Redundancy”, w: *2016 IEEE 19th International
  Conference on Intelligent Transportation Systems (ITSC)*, s. 2577–2582,
  2016. DOI: `10.1109/ITSC.2016.7795970`; ISBN elektroniczny:
  `978-1-5090-1889-5`; ISBN drukowany: `978-1-5090-1890-1`.
- [L3] Q. H. Che, T. D. Thanh, C. T. Van, „Character Time-series Matching
  for Robust License Plate Recognition”, w: *2022 International Conference on
  Multimedia Analysis and Pattern Recognition (MAPR)*, s. 185–190, 2022.
  DOI: `10.1109/MAPR56351.2022.9924897`; ISBN elektroniczny:
  `978-1-6654-7410-8`; ISBN drukowany: `978-1-6654-7411-5`.
- [L4] Q. Wang, X. Lu, C. Zhang, Y. Yuan, X. Li, „LSV-LP: Large-Scale
  Video-Based License Plate Detection and Recognition”, *IEEE Transactions on
  Pattern Analysis and Machine Intelligence*, vol. 45, nr 1, s. 752–767,
  2023. DOI: `10.1109/TPAMI.2022.3153691`; ISSN: `0162-8828`.
- [L5] M. Smink, H. Liu, D. Döpfer, Y. J. Lee, „Computer Vision on the Edge:
  Individual Cattle Identification in Real-Time With ReadMyCow System”, w:
  *2024 IEEE/CVF Winter Conference on Applications of Computer Vision (WACV)*,
  s. 7041–7050, 2024. DOI: `10.1109/WACV57701.2024.00690`;
  ISBN elektroniczny: `979-8-3503-1892-0`; ISBN drukowany:
  `979-8-3503-1893-7`.
- [L6] V. N. Ribeiro, N. S. T. Hirata, „Efficient License Plate Recognition
  in Videos Using Visual Rhythm and Accumulative Line Analysis”, preprint,
  2025. DOI: `10.48550/arXiv.2501.04750`; ISBN: nie dotyczy.
- [L7] N. Aharon, R. Orfaig, B.-Z. Bobrovsky, „BoT-SORT: Robust Associations
  Multi-Pedestrian Tracking”, preprint, 2022.
  DOI: `10.48550/arXiv.2206.14651`; ISBN: nie dotyczy.

### 2026-08-20 — wdrożenie adaptacyjnego pipeline'u MT–MZ

Problem:

Tracker działający wcześniej w `MainActivity` stabilizował wyłącznie rysowane
ramki. Otrzymywał wynik dopiero po zakończeniu pełnej inferencji, dlatego nie
mógł ograniczyć liczby wywołań MZ. Dodatkowo Android stosował dla tablic tylko
IoU-NMS, a dla znaków brakowało dodatkowej deduplikacji niezależnej od klasy
i filtra spójności sekwencji znanego z aplikacji macierzystej.

Rozwiązanie:

- dodano `DetectionDeduplicator`, który łączy IoU z pokryciem mniejszej ramki;
- deduplikacja tablic odbywa się przed rektyfikacją i MZ, dzięki czemu kopia
  tej samej tablicy nie powoduje następnej inferencji znaków;
- dodano `PlateQualityScorer`, będący lekkim portem geometrycznego `fit_score`
  aplikacji macierzystej;
- wynik jakości MT obejmuje confidence keypointów, poprawność czworokąta,
  proporcje i spójność boków, zgodność polygon–bbox oraz względny rozmiar;
- narożniki poza obrazem, czworokąty zdegenerowane i geometria poniżej progu
  profilu nie są kierowane do MZ;
- dodano `CharacterSequencePostProcessor`: class-agnostic deduplikację znaków,
  grupowanie wierszy, medianowy model geometrii, odrzucanie outlierów oraz
  rozstrzyganie konfliktów sąsiednich boxów;
- słabsi kandydaci znaków są dekodowani z tego samego tensora MZ z progiem
  podłogowym maksymalnie `0,10`; ich użycie nie wymaga drugiej inferencji;
- dodano `PlateTrackCoordinator`, który przypisuje detekcje MT do tracków i
  podejmuje decyzję, czy w bieżącej klatce uruchomić MZ;
- dodano `TemporalCharacterAggregator`, który głosuje osobno dla każdej
  pozycji znaku i uznaje tekst za stabilny dopiero po co najmniej dwóch
  zgodnych obserwacjach wszystkich pozycji;
- stabilny track przestaje uruchamiać MZ; niestabilny track po wykorzystaniu
  początkowego budżetu przechodzi na rzadsze próby okresowe, aby nie zostać
  trwale zablokowanym po serii słabych klatek;
- zmiana profilu resetuje tracki i stabilizatory, żeby nie mieszać dowodów
  zebranych według różnych polityk;
- `MainActivity` nadal odpowiada tylko za wybór profilu i prezentację; decyzje
  o inferencji zostały przeniesione do pipeline'u.

Profile wykonania:

| Profil | Początkowy budżet MZ | Minimalny fit MT | Wymagana poprawa jakości | Odstęp próby okresowej |
| --- | ---: | ---: | ---: | ---: |
| Szybki | 2 | 0,45 | 0,08 | 10 klatek |
| Zrównoważony | 4 | 0,35 | 0,04 | 6 klatek |
| Dokładny | 6 | 0,25 | 0,02 | 4 klatki |

Wartości są początkową polityką inżynierską, a nie wynikiem benchmarku na
telefonie. Profil jest zapisywany w preferencjach aplikacji. W raporcie sesji
zapisywane są `recognition_profile` oraz liczniki `mz_runs`, `mz_skipped` i
`invalid_plate_geometry`. CSV zawiera również `plate_fit`, co umożliwi
wyznaczenie rzeczywistych progów na podstawie materiału z ground truth.

Rozróżnienie modeli:

Implementacja nie dodaje OCR. MT pozostaje modelem YOLO Pose tablic i
narożników, natomiast MZ pozostaje modelem YOLO Character Detection. Konsensus
temporalny agreguje klasy i confidence detekcji MZ.

Weryfikacja:

- `gradlew testDebugUnitTest` — sukces, 34 testy;
- dodano testy jakości czworokąta, deduplikacji zagnieżdżonych tablic,
  duplikatów różnych klas w jednym slocie znaku, outlierów geometrii,
  tablic dwurzędowych, konsensusu temporalnego i zatrzymania MZ po stabilizacji;
- `gradlew lintDebug` — sukces;
- `gradlew assembleDebug` — sukces.

Pozostało:

- zweryfikować progi trzech profili na fizycznym urządzeniu i materiale z
  ground truth;
- zmierzyć rzeczywistą redukcję `mz_runs` oraz zmianę p50/p95 całego pipeline'u;
- dodać ocenę ostrości cropu, jeżeli sama geometria MT okaże się
  niewystarczającym predyktorem jakości MZ;
- rozważyć globalną kompensację ruchu kamery; aktualny tracker stosuje model
  prędkości ramki, ale nie estymuje homografii całej sceny;
- rozważyć bufor 2–3 najlepszych cropów, jeżeli odczyt najlepszego wycinka
  dopiero przy wygasaniu tracku poprawi exact match bez istotnego wzrostu RAM.

### 2026-08-20 — menu komunikacji użytkownika i uproszczenie ekranu kamery

Problem:

- ekran roboczy eksponował jednocześnie wynik, stan modeli, dziennik, wybór
  profilu oraz akcje techniczne, przez co komunikacja z aplikacją nie miała
  jednego, przewidywalnego punktu wejścia;
- przycisk profilu zajmował stałe miejsce, chociaż ustawienie jest zmieniane
  sporadycznie;
- brakowało dostępnych z poziomu interfejsu poleceń wyczyszczenia bieżącego
  śledzenia, pomocy oraz opisu ról MT i MZ.

Rozwiązanie:

- dodano górny `MaterialToolbar` z menu obejmującym import pakietu, eksport
  raportu, wybór profilu szybki/zrównoważony/dokładny, diagnostykę,
  wyczyszczenie wyniku, pomoc i informacje o aplikacji;
- aktywny profil oraz widoczność diagnostyki są zapamiętywane w
  `SharedPreferences`, a aktualny wybór profilu jest oznaczony w menu;
- panel stanu modeli i trwałego dziennika jest domyślnie ukryty i otwierany
  opcją Diagnostyka;
- usunięto stały przycisk przełączania profilu z ekranu głównego;
- polecenie czyszczenia resetuje stabilizację tekstu, śledzenie ramek w UI i
  stan `PlateTrackCoordinator` w pipeline, dzięki czemu poprzednia tablica nie
  wpływa na kolejną sesję obserwacji;
- dialog pomocy prowadzi użytkownika przez import kompletu MT+MZ, wybór profilu
  i stabilizację obrazu, a dialog „O aplikacji” wyjaśnia, że MT jest detektorem
  tablic/narożników, a MZ detektorem znaków YOLO, nie klasycznym OCR.

Decyzje:

- główny ekran pozostaje skoncentrowany na kadrze kamery, stanie bieżącym i
  ustabilizowanym wyniku; funkcje konfiguracyjne i diagnostyczne zebrano w menu;
- przyciski importu i eksportu zachowano również jako szybkie akcje w panelu
  dolnym, natomiast menu zapewnia ich stałą dostępność i odkrywalność;
- zmiana profilu zeruje stan czasowy, ponieważ próbki zebrane przy odmiennej
  polityce MZ nie powinny być łączone w jeden konsensus.

Weryfikacja:

- `gradlew testDebugUnitTest` — sukces;
- `gradlew assembleDebug` — sukces;
- `gradlew lintDebug` — sukces, 0 błędów;
- kompilacja zasobów menu, layoutu i kodu Java — sukces.

Pozostało:

- zweryfikować ergonomię menu, czytelność na małym ekranie i działanie
  TalkBack na fizycznym telefonie;
- wykonać test instrumentalny resetu śledzenia w trakcie aktywnej kamery.

### 2026-08-20 — decyzja projektowa: rozdzielczość źródła i kaskada ROI

Problem:

- telefon pracuje w ciągłym ruchu, dlatego czas uzyskania pierwszego wyniku,
  ostrość klatki i odporność śledzenia na przesunięcie kamery są równie ważne
  jak sama dokładność modeli;
- podniesienie rozdzielczości całej klatki zwiększa liczbę dostępnych szczegółów,
  ale jednocześnie zwiększa koszt konwersji obrazu, transferu pamięci i
  przygotowania tensora;
- mała tablica zostaje przeskalowana do wejścia MT razem z całą sceną. Samo
  zwiększenie rozdzielczości kamery nie gwarantuje więc poprawy, jeżeli MT nadal
  analizuje pełną klatkę;
- opcjonalny model pojazdów jest już obsługiwany przez klienta, ale obecnie
  działa tylko jako bramka obecności pojazdu. Po jego wykonaniu MT nadal
  otrzymuje pełny obraz;
- aktualny wybór rozdzielczości odbywa się raz przy uruchomieniu: `640×480` na
  urządzeniach z małą ilością pamięci i `1280×720` na pozostałych. Polityka nie
  uwzględnia wielkości tablicy, ostrości obrazu ani ruchu kamery.

#### Rozwinięcie skrótów i terminów

- **ALPR — Automatic License Plate Recognition**: automatyczne wykrywanie i
  odczytywanie tablic rejestracyjnych;
- **ROI — Region of Interest**: obszar zainteresowania, czyli fragment obrazu
  wybrany do dalszej analizy; w proponowanej kaskadzie jest nim poszerzone
  otoczenie wykrytego pojazdu;
- **MT — Model Tablic**: model YOLO Pose wykrywający tablicę oraz jej cztery
  narożniki potrzebne do rektyfikacji;
- **MZ — Model Znaków**: model YOLO Character Detection wykrywający osobne
  litery i cyfry na wyprostowanym obrazie tablicy; nie jest to klasyczny OCR;
- **YOLO — You Only Look Once**: rodzina jednostopniowych modeli detekcji
  obiektów; w projekcie wykorzystywana zarówno do lokalizacji tablic, jak i
  znaków;
- **OCR — Optical Character Recognition**: optyczne rozpoznawanie znaków,
  zwykle rozumiane jako bezpośrednia zamiana obrazu na tekst; nie opisuje
  zastosowanego tutaj detektora znaków YOLO;
- **RGBA — Red, Green, Blue, Alpha**: czterokanałowa reprezentacja obrazu
  używana obecnie przez `ImageAnalysis`; każdy piksel zajmuje cztery bajty;
- **YUV**: reprezentacja obrazu rozdzielająca jasność od składowych barwnych,
  typowa dla kamer; może ograniczyć kopiowanie danych, jeżeli preprocessing
  modeli będzie wykonywany bez pośredniej konwersji całej klatki do RGBA;
- **RAM — Random Access Memory**: pamięć operacyjna urządzenia;
- **CPU — Central Processing Unit**: główny procesor urządzenia;
- **GPU — Graphics Processing Unit**: procesor graficzny, który może wykonywać
  część operacji modeli przez delegata akcelerującego;
- **TFLite/LiteRT — TensorFlow Lite / Lite Runtime**: mobilny format i środowisko
  wykonywania modeli;
- **ONNX — Open Neural Network Exchange**: przenośny format wymiany modeli,
  uruchamiany w aplikacji przez ONNX Runtime;
- **ground truth**: referencyjna, ręcznie zweryfikowana odpowiedź używana do
  pomiaru rzeczywistej jakości rozpoznawania.

Rozwiązanie docelowe:

```text
kamera 1280×720 lub 1920×1080
  -> ocena ruchu, ostrości i obciążenia urządzenia
  -> pomniejszony obraz wejściowy opcjonalnego detektora pojazdów
  -> mapowanie wykrytych pojazdów na pełną rozdzielczość źródła
  -> wybór i poszerzenie maksymalnie 1–2 ROI pojazdów
  -> MT uruchamiany na wybranych ROI
  -> rektyfikacja najlepszej tablicy
  -> MZ uruchamiany na wyprostowanym obrazie tablicy
  -> natychmiastowy wynik wstępny
  -> konsensus kolejnych obserwacji i wynik potwierdzony
```

Model pojazdów powinien otrzymywać pomniejszony tensor, na przykład o boku
`320` lub `416` pikseli. Jego ramki należy następnie przeliczyć na współrzędne
obrazu źródłowego. Dopiero wtedy wycina się ROI z klatki `1280×720` albo
`1920×1080`. Pozwala to zachować szczegóły małej tablicy bez uruchamiania MT na
całej klatce o wysokiej rozdzielczości.

Detektora pojazdów nie należy uruchamiać obowiązkowo dla każdej klatki. Zalecana
polityka początkowa to wykonanie go co 3–5 analizowanych klatek i przewidywanie
położenia ROI przez tracker pomiędzy inferencjami. ROI powinno zostać poszerzone
o około 10–20%, aby ruch kamery i niedokładność ramki pojazdu nie obcięły
tablicy. Należy również okresowo uruchamiać MT na pełnej klatce, na przykład po
utracie tracku albo co 10–15 klatek, aby nie utrwalić błędu pierwszego detektora
i nie pomijać motocykli lub pojazdów spoza jego klas.

#### Koszt pamięci klatki

Przy bieżącej reprezentacji RGBA teoretyczny rozmiar samego bufora wynosi
`szerokość × wysokość × 4 bajty`, bez dodatkowych kopii i obiektów runtime'u:

| Rozdzielczość | Liczba pikseli | Bufor RGBA | Zastosowanie |
| --- | ---: | ---: | --- |
| `640×480` | 307 200 | 1,23 MB | tryb awaryjny i słabsze urządzenia |
| `1280×720` | 921 600 | 3,69 MB | zalecany punkt bazowy |
| `1920×1080` | 2 073 600 | 8,29 MB | odległe tablice, najlepiej razem z ROI |

Pełnoklatkowe `1920×1080` nie powinno być domyślnym wejściem całej kaskady.
Jeżeli obraz jest od razu skalowany do stałego wejścia MT, dodatkowy koszt może
przewyższyć zysk jakości. Wysoka rozdzielczość źródłowa staje się najbardziej
użyteczna wtedy, gdy z oryginalnej klatki wycinany jest mały ROI, który dopiero
później zostaje przeskalowany do wejścia MT.

#### Profile rozdzielczości w interfejsie

Użytkownik nie powinien wybierać surowych parametrów każdego tensora. Menu
„Pipeline i modele” powinno udostępniać trzy zrozumiałe tryby:

| Profil | Rozdzielczość źródłowa | Przeznaczenie |
| --- | --- | --- |
| Automatyczny | `1280×720` lub `1920×1080` | dobór na podstawie urządzenia, ruchu, temperatury i wielkości tablicy |
| Szybki | `640×480` | bliskie pojazdy, ograniczony RAM lub wysoka temperatura |
| Daleki odczyt | `1920×1080` | małe i odległe tablice; wymaga aktywnej kaskady ROI |

Zmiana rozdzielczości `ImageAnalysis` wymaga ponownego związania strumienia
CameraX, dlatego nie należy przełączać jej co klatkę. Regulator może podjąć
decyzję przy rozpoczęciu sesji albo po utrzymującym się przez kilka sekund
przekroczeniu progów obciążenia. W obrębie klatek szybką adaptację powinny
zapewniać wybór ROI, pomijanie słabych klatek i harmonogram modeli.

#### Stabilizacja przy ruchu telefonu

Większa rozdzielczość nie kompensuje rozmycia ruchowego. Ostra klatka `1280×720`
może zawierać więcej użytecznej informacji niż rozmyta klatka `1920×1080`.
Scheduler powinien zatem uwzględniać:

- miarę ostrości i rozmycia wycinka;
- prędkość oraz zmianę skali śledzonego ROI;
- wysokość tablicy w pikselach obrazu źródłowego;
- poprawność czterech narożników i wynik dopasowania geometrycznego;
- czas konwersji obrazu, inferencji modelu pojazdów, MT i MZ;
- liczbę pomijanych klatek, stan pamięci i temperaturę urządzenia.

Jako początkową heurystykę badawczą można przyjąć, że tablica niższa niż około
30–40 pikseli wymaga ciaśniejszego ROI albo wyższej rozdzielczości źródła.
Nie jest to próg końcowy: musi zostać wyznaczony eksperymentalnie dla używanego
aparatu, odległości, modeli i zbioru z ground truth.

#### Podparcie w literaturze

Literatura wspiera zastosowanie ROI i kaskady pojazd–tablica, ale nie uzasadnia
bezwarunkowego włączenia dodatkowego modelu na każdym urządzeniu i w każdej
klatce. Wnioski źródłowe są następujące:

- Laroca i in. opisują typowy ALPR jako potok wieloetapowy i wskazują, że
  wcześniejsze prace lokalizowały najpierw pojazd, a następnie jego tablicę w
  celu skrócenia przetwarzania i ograniczenia wyników fałszywie dodatnich [R1].
  Ich własny system używa osobnych sieci CNN do detekcji pojazdu i tablicy
  wewnątrz wykrytego pojazdu. Badanie obejmuje również zbiór UFPR-ALPR, w którym
  poruszają się zarówno pojazdy, jak i kamery. Jest to bezpośrednie wsparcie dla
  rozważanej kaskady, chociaż czasy uzyskane na GPU nie mogą zostać przeniesione
  wprost na telefon;
- Khan i in. zauważają, że tablica zajmuje małą część obrazu, obrazy HD
  zwiększają koszt obliczeniowy, a lokalizacja małego obiektu w całej klatce
  jest trudna [R2]. Zaproponowany przez nich układ najpierw wykrywa pojazdy, a
  następnie lokalizuje tablice. Autorzy raportują ograniczenie tła i zakłóceń,
  wzrost dokładności oraz skrócenie czasu przetwarzania. Wynik wspiera użycie
  ROI pojazdu jako sposobu zwiększenia względnej skali tablicy;
- Silva i Jung analizują ALPR w scenach niekontrolowanych, wprost wymieniając
  kamerę mobilną lub smartfon oraz wynikające z tego widoki ukośne [R3]. Ich
  model estymuje geometrię tablicy i umożliwia jej rektyfikację przed odczytem.
  Wspiera to decyzję, aby po wyborze ROI nadal zachować detekcję czterech
  narożników i ocenę geometrii, zamiast ograniczać potok do prostokątnego cropu;
- Ren i in. formalizują detekcję opartą na propozycjach regionów oraz
  reprezentacji cech wybranych obszarów [R4]. Publikacja uzasadnia ogólną ideę
  kierowania kosztowniejszej analizy do regionów kandydujących. Należy jednak
  rozróżnić `RoI pooling` z Faster R-CNN, wykonywany na mapach cech, od
  planowanego w aplikacji wycięcia pikselowego ROI z klatki kamery;
- Redmon i in. przedstawiają YOLO jako jednostopniową detekcję całego obrazu,
  projektowaną z myślą o pracy w czasie rzeczywistym [R5]. Uzasadnia to wybór
  lekkiego YOLO jako pierwszego stopnia kaskady, ale historyczne wyniki FPS z
  GPU nie stanowią prognozy wydajności Androida.

Z powyższych prac wynika, że ROI może ograniczyć tło, zwiększyć względną skalę
małej tablicy i zmniejszyć koszt następnego etapu. Jest to jednak wniosek o
architekturze, nie dowód, że `pojazd → MT → MZ` zawsze przewyższy `MT → MZ`.
Dodatkowy model ma własny koszt, a pominięcie pojazdu uniemożliwia wykonanie
następnych etapów. Z tego powodu aplikacja powinna zachować okresowy fallback
pełnoklatkowy, mierzyć każdy etap osobno i automatycznie wyłączać kaskadę
pojazdu, gdy nie daje zysku na danym urządzeniu lub w scenie z wieloma
pojazdami.

Wartości `3–5` klatek między detekcjami pojazdu, margines ROI `10–20%` oraz
próg wysokości tablicy `30–40` pikseli pozostają hipotezami inżynierskimi.
Nie pochodzą bezpośrednio z wymienionych publikacji i muszą zostać zweryfikowane
eksperymentalnie na urządzeniu docelowym.

Literatura dla decyzji o ROI:

- [R1] R. Laroca, E. Severo, L. A. Zanlorensi, L. S. Oliveira,
  G. R. Gonçalves, W. R. Schwartz, D. Menotti, „A Robust Real-Time Automatic
  License Plate Recognition Based on the YOLO Detector”, w: *2018
  International Joint Conference on Neural Networks (IJCNN)*, s. 1–10, 2018.
  DOI: `10.1109/IJCNN.2018.8489629`; ISBN online: `978-1-5090-6014-6`;
  ISBN Print-on-Demand: `978-1-5090-6015-3`; ISSN: `2161-4393`.
- [R2] K. Khan, A. Imran, H. Z. Ur Rehman, A. Fazil, M. Zakwan,
  Z. Mahmood, „Performance Enhancement Method for Multiple License Plate
  Recognition in Challenging Environments”, *EURASIP Journal on Image and
  Video Processing*, vol. 2021, art. 30, 2021.
  DOI: `10.1186/s13640-021-00572-4`; eISSN: `1687-5281`.
- [R3] S. M. Silva, C. R. Jung, „License Plate Detection and Recognition in
  Unconstrained Scenarios”, w: *Computer Vision – ECCV 2018*, LNCS, vol. 11216,
  Springer, s. 593–609, 2018. DOI: `10.1007/978-3-030-01258-8_36`;
  ISBN drukowany: `978-3-030-01257-1`; ISBN online: `978-3-030-01258-8`.
- [R4] S. Ren, K. He, R. Girshick, J. Sun, „Faster R-CNN: Towards Real-Time
  Object Detection with Region Proposal Networks”, *IEEE Transactions on
  Pattern Analysis and Machine Intelligence*, vol. 39, nr 6, s. 1137–1149,
  2017. DOI: `10.1109/TPAMI.2016.2577031`; ISSN: `0162-8828`.
- [R5] J. Redmon, S. Divvala, R. Girshick, A. Farhadi, „You Only Look Once:
  Unified, Real-Time Object Detection”, w: *2016 IEEE Conference on Computer
  Vision and Pattern Recognition (CVPR)*, s. 779–788, 2016.
  DOI: `10.1109/CVPR.2016.91`.

Decyzje:

- pozostawić `1280×720` jako bazowy kompromis do czasu wykonania benchmarku;
- nie zwiększać bezwarunkowo całego pipeline'u do `1920×1080`;
- najpierw wdrożyć rzeczywiste przekazywanie ROI z modelu pojazdów do MT;
- pokazywać pierwszy poprawny odczyt MZ jako wynik wstępny, a potwierdzenie
  temporalne wykonywać bez ukrywania go przed użytkownikiem;
- oddzielić stabilizację prezentacji od stabilizacji wejścia inferencji:
  tracker nakładki wygładza ramkę na ekranie, natomiast pipeline musi osobno
  oceniać ruch, jakość cropu i ciągłość ROI;
- pobieranie modeli z sieci realizować przez kontrolowany katalog podpisanych
  pakietów `.alprmodel`, a nie przez wykonywanie dowolnych plików YOLO `.pt`;
- katalog powinien podawać rolę modelu, klasy pojazdów, format wykonawczy,
  rozmiar wejścia, rozmiar pliku, sumę SHA-256 i zgodność z urządzeniem.

Plan weryfikacji:

- porównać co najmniej `640×480`, `1280×720` i `1920×1080`;
- dla każdej rozdzielczości zmierzyć wariant pełnoklatkowy oraz wariant z ROI;
- raportować czas pierwszego wyniku wstępnego, czas wyniku potwierdzonego,
  opóźnienia poszczególnych modeli, liczbę pominiętych klatek, zużycie pamięci
  i stan termiczny;
- mierzyć wysokość tablicy w pikselach oraz udział ROI w powierzchni klatki;
- jakość oceniać przez exact match i CER na materiale z ground truth, a nie
  przez samo confidence modeli;
- testować osobno sceny z jednym pojazdem, wieloma pojazdami, motocyklem,
  szybkim przesunięciem telefonu i słabym oświetleniem.

Status:

- zaimplementowano wynik wstępny, ocenę ostrości, opcjonalne przekazywanie ROI
  pojazdu do MT i pełnoklatkowy fallback;
- zaimplementowano profile rozdzielczości, wejście YUV i reakcję harmonogramu
  ROI na gwałtowny ruch z żyroskopu;
- globalna kompensacja ruchu i katalog modeli sieciowych nie są jeszcze
  zaimplementowane.

### 2026-08-20 — pierwsze wdrożenie szybkiego wyniku i kaskady ROI

Problem:

- UI ponownie stabilizował tekst, który wcześniej przeszedł konsensus w
  pipeline, przez co użytkownik nie widział pierwszego kompletnego odczytu MZ;
- opcjonalny model pojazdów zatrzymywał pipeline przy braku detekcji, ale nie
  przekazywał ROI do MT;
- decyzja o ponowieniu MZ uwzględniała geometrię, lecz nie ostrość obrazu;
- raport nie rozróżniał czasu wyniku wstępnego i potwierdzonego ani użycia ROI.

Rozwiązanie:

- `TemporalCharacterAggregator` pozostał jedynym właścicielem konsensusu;
  pierwszy kompletny odczyt jest prezentowany jako wstępny, a kolejne zgodne
  obserwacje nadają mu stan potwierdzony;
- usunięto drugą bramkę `MultiRecognitionStabilizer` z `MainActivity`;
- dodano bezwzorcową ocenę ostrości opartą na wariancji dyskretnego Laplasjanu;
  ostrość zwiększa priorytet lepszej kolejnej klatki, ale nie obniża jakości
  pierwszej próby poniżej wyniku geometrii MT;
- dodano opcję menu „Kaskada pojazd → tablica”, domyślnie wyłączoną i
  zapamiętywaną w preferencjach;
- po włączeniu MP wybierane są maksymalnie dwa dominujące ROI, poszerzane o
  18%; MP jest odświeżany co trzy analizowane klatki;
- MT otrzymuje wycinki źródłowego obrazu, a jego detekcje i narożniki są
  mapowane z powrotem do współrzędnych pełnej klatki;
- brak tablicy w ROI powoduje natychmiastowe wykonanie MT na pełnej klatce;
  dodatkowy fallback kontrolny występuje co 15 analizowanych klatek;
- brak MP nie blokuje rozpoznawania — aktywowany zostaje wariant pełnoklatkowy;
- raport zapisuje `time_to_first_preliminary_ms`,
  `time_to_first_confirmed_ms`, `plate_sharpness`, `vehicle_roi_area_ratio`,
  liczbę wykonań i pominięć MP, uruchomienia MT na ROI oraz fallbacki.

Decyzje:

- kaskada pozostaje opcjonalna, ponieważ jej koszt może przewyższać zysk przy
  wielu pojazdach lub bardzo szybkim MT;
- wartości 18%, 3 i 15 są parametrami startowymi do benchmarku, nie progami
  potwierdzonymi eksperymentalnie;
- wynik wstępny nie jest przedstawiany jako potwierdzony, dzięki czemu poprawa
  czasu reakcji UI nie ukrywa niepewności pojedynczej obserwacji;
- zachowano `STRATEGY_KEEP_ONLY_LATEST` CameraX, aby wolniejsza kaskada nie
  tworzyła kolejki nieaktualnych klatek.

Weryfikacja:

- dodano testy oceny ostrości i wyboru ograniczonej liczby poszerzonych ROI;
- `gradlew testDebugUnitTest` — sukces, 40 testów;
- `gradlew lintDebug` — sukces, 0 błędów; pozostało 11 ostrzeżeń o wersjach
  SDK i zależności;
- `gradlew assembleDebug` — sukces.

Pozostało:

- wykonać benchmark na fizycznym telefonie z rzeczywistym MP+MT+MZ;
- sprawdzić wpływ ponownego użycia ROI przez trzy klatki podczas szybkiego
  przesuwania kamery;
- wdrożyć globalną kompensację ruchu albo częstsze odświeżanie MP, jeżeli
  margines ROI okaże się niewystarczający;
- dobrać parametry kaskady na podstawie `exact match`, CER, czasu pierwszego
  wyniku oraz stanu termicznego.

### 2026-08-20 — profile kamery, YUV i reakcja na szybki ruch

Problem:

- użytkownik nie mógł wybrać kompromisu między szybkością a szczegółowością
  obrazu źródłowego;
- raport nie zapisywał faktycznej rozdzielczości dostarczonej przez CameraX;
- ROI pojazdu było ponownie używane przez kilka klatek również podczas
  gwałtownego obrotu telefonu;
- CameraX zwracał RGBA, a aplikacja ręcznie kopiowała każdy piksel do bitmapy
  w kodzie Java.

Rozwiązanie:

- dodano profile `Automatyczna`, `Szybka 640×480` i `Daleki odczyt
  1920×1080`; wybór jest zapamiętywany, oznaczany w menu i powoduje bezpieczne
  ponowne związanie `ImageAnalysis`;
- raport zapisuje profil, rozdzielczość żądaną, faktyczny rozmiar klatki oraz
  format pikseli;
- dodano monitor żyroskopu z wygładzoną prędkością kątową; po przekroczeniu
  początkowego progu 1 rad/s MP jest odświeżany w każdej przetwarzanej klatce,
  a margines ROI rośnie z 18% do 28%;
- próbki żyroskopu starsze niż 300 ms nie wpływają na pipeline;
- CameraX przełączono na `YUV_420_888`, a ręczny konwerter RGBA zastąpiono
  obsługiwaną przez CameraX 1.4 natywną metodą `ImageProxy.toBitmap()`;
- dodano okno „Pipeline i modele”, pozwalające wyłączyć MP, wybrać jeden z
  zainstalowanych modeli pojazdów albo przejść do importu pliku.

Decyzje:

- profile rozdzielczości działają na poziomie sesji, nie pojedynczej klatki;
- tryb `1920×1080` nie włącza automatycznie MP, ponieważ użytkownik może chcieć
  porównać wariant pełnoklatkowy w benchmarku;
- żyroskop steruje częstotliwością odświeżania ROI, ale nie jest przedstawiany
  jako pełna stabilizacja obrazu ani kompensacja homografii;
- zachowano bitmapę jako wspólne wejście runtime'ów. Bezpośredni preprocessing
  YUV → tensor wymaga testu zgodności RGB/BGR i kwantyzacji dla wszystkich
  wariantów modeli;
- nie dodano pobierania dowolnych modeli z URL. Bezpieczny katalog wymaga
  wskazanego endpointu HTTPS, sum SHA-256 i klucza do weryfikacji podpisu
  wydawcy; tych danych projekt jeszcze nie posiada.

Weryfikacja:

- dodano testy profili rozdzielczości i filtra intensywności ruchu;
- `gradlew testDebugUnitTest` — sukces, 42 testy;
- `gradlew lintDebug` — sukces, 0 błędów; pozostało 11 ostrzeżeń o wersjach
  SDK i zależności;
- `gradlew assembleDebug` — sukces;
- kompilacja ścieżki CameraX YUV — sukces.

Pozostało:

- zmierzyć koszt natywnej konwersji YUV na fizycznym urządzeniu;
- dobrać próg żyroskopu, margines dynamiczny i częstotliwość MP z raportów;
- porównać 720p i 1080p z ROI oraz bez ROI;
- skonfigurować zaufany endpoint katalogu i klucz podpisu modeli.

### 2026-08-20 — trwała dwukolumnowa galeria tablic

Problem:

- import i eksport były zdublowane w menu oraz jako dwa duże CTA w dolnym
  panelu, przez co interfejs wyglądał jak ekran techniczny zamiast narzędzia
  pracującego z kamerą;
- pojedynczy napis wyniku znikał po pustej klatce, mimo że tablica została już
  wcześniej wykryta;
- użytkownik nie widział cropu tablicy, położenia znaków ani rozdzielonych
  confidence MT i MZ.

Rozwiązanie:

- usunięto dolne przyciski importu i eksportu; obie akcje pozostały w menu;
- pojedynczy panel tekstowy zastąpiono przewijaną galerią maksymalnie sześciu
  kart w dwóch kolumnach;
- `MobileAlprEngine` tworzy małą bitmapę zrektyfikowanej tablicy wyłącznie przy
  rzeczywistym wykonaniu MZ i mapuje ramki znaków z tensora na crop;
- `PlateObservation` przenosi track, crop, tekst, stan potwierdzenia, confidence
  tablicy i odczytu oraz znormalizowane pozycje znaków;
- karta rysuje znak bezpośrednio na cropie oraz pokazuje tekstową listę pozycji
  `(x, y)` i confidence każdego znaku;
- wynik wstępny jest przechowywany przez 6 sekund, potwierdzony przez 15 sekund;
  pojedyncza pusta klatka nie usuwa karty;
- liczba kart jest ograniczona do sześciu, a najstarsze są usuwane jako
  pierwsze; zgodne teksty z ponownie utworzonych tracków są scalane;
- dodano jawne zarządzanie własnością bitmap: UI kopiuje podgląd, a
  `PipelineResult.close()` zwalnia bitmapy pipeline'u także dla wyników
  pominiętych przez limiter odświeżania ekranu.

Decyzje:

- podglądy mają standardowy rozmiar rektyfikacji 256×64 lub 256×128, dzięki
  czemu trwałość wyników nie wymaga przechowywania pełnych klatek kamery;
- położenia znaków są prezentowane w procentach cropu, aby nie zależały od
  rozmiaru widoku i orientacji telefonu;
- stan wstępny i potwierdzony otrzymały inne kolory, ale confidence nie jest
  nazywane dokładnością;
- galeria jest czyszczona po zmianie profilu, rozdzielczości, kaskady, modelu
  albo po ręcznym poleceniu wyczyszczenia wyniku.

Weryfikacja:

- dodano test ograniczania współrzędnych znaków do obszaru podglądu;
- kompilacja kontraktu `PlateObservation`, widoku cropu i layoutów — sukces;
- `gradlew testDebugUnitTest` — sukces, 43 testy;
- `gradlew lintDebug` — sukces, 0 błędów; pozostało 11 ostrzeżeń o wersjach
  SDK i zależności;
- `gradlew assembleDebug` — sukces;
- zainstalowano debug APK na fizycznym telefonie Samsung `720×1600`;
- potwierdzono brak dolnych CTA, poprawne menu, dwie kolumny kart, cropy,
  nakładki znaków, listę pozycji oraz odmienne kolory stanu wstępnego i
  potwierdzonego.

Pozostało:

- wykonać wizualny test kart z rzeczywistym kompletem modeli i tablicą przed
  kamerą;
- dobrać wysokość galerii oraz TTL na podstawie testu użytkowego;
- rozważyć ekran historii sesji, jeśli sześć bieżących kart okaże się
  niewystarczające.

### 2026-08-20 — sterowana sesja cropów i trwały zapis

Problem:

- galeria działała jako automatyczny, czasowo wygaszany podgląd, a użytkownik
  nie mógł zdecydować, kiedy rozpoczyna się zbieranie materiału;
- pionowa siatka nie skalowała się do kilkudziesięciu cropów;
- brakowało limitu zależnego od pamięci, dat rejestracji, czasów per crop i
  kontrolowanego zapisu wybranych obserwacji na nośniku użytkownika;
- ekran mógł wygasnąć podczas długiej sesji.

Rozwiązanie:

- rozdzielono inferencję na żywo od kolektora: overlay i status działają stale,
  a przycisk `Start/Stop` steruje tylko dodawaniem do sesji;
- pierwsze uruchomienie Start tworzy `session_id`, Stop wstrzymuje, a kolejny
  Start wznawia tę samą sesję; wyczyszczenie tworzy warunki dla nowej sesji;
- galerię zastąpiono poziomym `RecyclerView` z kartami o stałej szerokości;
- kolektor zapisuje pierwszy crop tracku, zmianę tekstu, pierwsze
  potwierdzenie, poprawę ostrości o co najmniej 0,08 albo próbkę po 1,5 s;
- dodano ustawienie limitu `Auto/10/25/50/100`; tryb Auto wykorzystuje około
  3% maksymalnego heapu, szacuje 160 KiB na element i ogranicza urządzenia
  `lowRamDevice` do 25 elementów;
- po przekroczeniu limitu usuwany jest najstarszy element niebędący w trakcie
  zapisu; zapisane elementy mogą opuścić pamięć, ponieważ istnieją już na
  nośniku;
- `CropInferenceTiming` wiąże z cropem `frame_id`, konwersję kamery, wspólne
  etapy MP i MT, rektyfikację, preprocessing, inferencję i postprocessing MZ
  oraz czas pipeline'u do utworzenia obserwacji;
- karta pokazuje lokalną datę z milisekundami, czas pipeline'u i MZ;
- zaznaczenie „Zapisz trwale” otwiera systemowy wybór katalogu, zachowuje
  trwałe uprawnienie URI i zapisuje parę JPEG + JSON;
- miniraport `alpr.mobile.crop_report.v1` zawiera identyfikatory, datę UTC,
  strefę czasową, tekst, confidence, sharpness, pozycje znaków, czasy, profile,
  fingerprinty modeli i urządzenie;
- eksport całej sesji zawiera sekcję `crop_session` z rekordami wszystkich
  zebranych cropów oraz informacją o trwałym zapisie;
- błędna próba zapisu usuwa pliki utworzone tylko przez tę próbę;
- napływ nowych cropów nie przewija galerii automatycznie; aktualizacja
  różnicowa adaptera zachowuje ręcznie wybrany fragment także przy usuwaniu
  najstarszego elementu po osiągnięciu limitu;
- `FLAG_KEEP_SCREEN_ON` zapobiega wygaszeniu ekranu, gdy aktywność jest
  widoczna, bez utrzymywania pełnego wakelocka w tle.

Decyzje:

- do galerii trafiają cropy już tworzone dla MZ; kolektor nie wymusza
  dodatkowej rektyfikacji każdej detekcji MT;
- zapis używa JPEG z jakością 94 i osobnego, czytelnego JSON zamiast zamkniętego
  formatu binarnego;
- użytkownik wybiera katalog przez mechanizm scoped storage, dlatego aplikacja
  nie żąda szerokiego uprawnienia do całej pamięci;
- czasy MP i MT są kosztami wspólnymi klatki, natomiast rektyfikacja i etapy MZ
  są mierzone osobno dla konkretnego cropu;
- ręczny limit może przekroczyć limit automatyczny urządzenia, ponieważ jest
  świadomą decyzją użytkownika, ale pozostaje ograniczony do 100.

Weryfikacja:

- dodano testy polityki limitu, selekcji cropów i czasów per crop;
- `gradlew testDebugUnitTest` — sukces, 46 testów;
- `gradlew lintDebug` i `gradlew assembleDebug` — sukces;
- zainstalowano APK na fizycznym telefonie Samsung `720×1600`;
- potwierdzono stan bezczynny, Start, gromadzenie 9 cropów, Stop, licznik
  `9/49`, poziome przewijanie, daty, czasy i kontrolkę trwałego zapisu;
- w `dumpsys power` potwierdzono aktywny stan `mStayOn=true`.

Pozostało:

- przejść pełny zapis JPEG+JSON dla katalogu wskazanego ręcznie przez
  użytkownika i sprawdzić zachowanie z fizyczną kartą SD;
- wykonać długą sesję do osiągnięcia limitu automatycznego oraz ręcznego 100;
- porównać strategię selekcji z zapisem każdej próby MZ w benchmarku pracy.

### 2026-08-20 — osobne ekrany Opcje i Diagnostyka

Problem:

- wielopoziomowe menu główne mieszało konfigurację, import, diagnostykę i
  obsługę sesji;
- panel diagnostyczny zajmował miejsce pod galerią i nie zapewniał czytelnej
  hierarchii informacji;
- aplikacja nie miała spójnego zestawu skalowalnych ikon ani semantycznej
  palety kolorów.

Rozwiązanie:

- dodano `SettingsActivity` z kartami konfiguracji profilu, jakości obrazu,
  limitu cropów, kaskady MP, modeli YOLO i katalogu zapisu;
- import `.alprmodel`, wybór modelu pojazdu i systemowy selektor katalogu
  działają bezpośrednio w ekranie Opcje;
- dodano `DiagnosticsActivity` z dwukolumnowym gridem urządzenia, pamięci,
  pipeline'u i sesji oraz osobnymi kartami modeli/runtime'u i logu;
- eksport raportu z Diagnostyki wraca kontraktem wyniku do `MainActivity`,
  która zachowuje własność metryk bieżącej sesji;
- toolbar obu aktywności ma strzałkę `finish()`, a deklaracje `parentActivity`
  i systemowy Back zapewniają powrót do istniejącego okna głównego;
- konfiguracja ma licznik rewizji w `SharedPreferences`; po powrocie kamera
  stosuje tylko zmienione wartości i przeładowuje modele;
- dodano natywne `VectorDrawable` ze ścieżkami SVG oraz ciemną paletę z
  akcentami niebieskim, fioletowym, zielonym i różowym; zestaw obejmuje 18
  skalowalnych ikon dla nawigacji, ustawień, modeli, pamięci, sesji i eksportu;
- menu `MainActivity` uproszczono do dwóch widocznych akcji ikonowych — Opcje
  i Diagnostyka — oraz pomocniczego menu przepełnienia z czyszczeniem sesji,
  pomocą i informacją o aplikacji;
- `MainActivity` przekazuje Diagnostyce stan sesji (`session_id`, aktywność,
  licznik i limit cropów), natomiast dane urządzenia, rejestru modeli i logów
  są odczytywane ponownie podczas odświeżania ekranu;
- zachowano niezależność nawigacji galerii: przechodzenie między aktywnościami
  ani napływ nowych cropów nie wywołują automatycznego przewijania listy;
- usunięto osadzony panel diagnostyczny i stare wielopoziomowe pozycje menu.

Weryfikacja:

- `gradlew testDebugUnitTest lintDebug assembleDebug` — sukces, 46/46 testów;
- Android Lint — 0 błędów i 11 wcześniejszych ostrzeżeń: 9 dotyczących
  dostępnych wersji zależności, 1 nowszej wersji narzędzia i 1 docelowego SDK;
- `git diff --check` — brak błędów spójności zmian; komunikaty dotyczą jedynie
  normalizacji zakończeń linii LF/CRLF przez Git;
- finalny `app-debug.apk` (75 847 420 bajtów) zbudowano i zainstalowano na
  fizycznym telefonie Samsung;
- urządzenie podczas odbioru wizualnego pozostawało na bezpiecznym ekranie
  blokady, dlatego finalne zrzuty obu aktywności wymagają odblokowania przez
  użytkownika.

Pozostało:

- wykonać końcowy odbiór wizualny na odblokowanym ekranie 720×1600 i sprawdzić
  pełny import modelu z nowej aktywności Opcje.

### 2026-08-20 — lekki tracker i bezkolizyjne badge’e

Problem:

- tekst detekcji i confidence miały ten sam kolor, przez co nie tworzyły
  czytelnej hierarchii;
- badge nad ramką był w sytuacji braku miejsca przenoszony do jej wnętrza;
- w galerii etykiety znaków były rysowane bezpośrednio na ramkach;
- pełne, mocne ramki i duże punkty sprawiały, że tracker dominował nad obrazem.

Rozwiązanie:

- ciąg/nazwa detekcji otrzymały kolor niebieski, confidence zielony, a pozycje
  i pozostałe metadane kolor przygaszony;
- overlay układa badge dopiero po wyliczeniu wszystkich ramek i odrzuca każdą
  pozycję przecinającą ramkę lub wcześniej umieszczoną etykietę; przy braku
  bezpiecznego miejsca etykieta nie jest rysowana;
- `PlateCropView` otrzymał osobny pas legendy pod obrazem; ramki znaków nie
  zawierają już badge’ów, a znak i jego confidence mają oddzielne kolory;
- ramkę trackera zwężono do 1,35 dp i obniżono jej alfa, punkty narożne
  zmniejszono do 2,2 dp;
- krótkie podtrzymanie tracku bez nowej detekcji jest rysowane cienką,
  półprzezroczystą linią przerywaną bez punktów i etykiety;
- rozszerzono `OverlayItem` o `trackId` oraz informację o predykcji przeniesionej
  bez świeżej obserwacji;
- obliczenia geometrii i kolizji przeniesiono poza `onDraw()`, aby odświeżanie
  overlayu nie tworzyło nowych kolekcji ani `RectF` podczas rysowania.

Weryfikacja:

- dodano dwa testy rozdzielania tekstu detekcji od końcowego confidence;
- `gradlew testDebugUnitTest lintDebug assembleDebug` — sukces, 48/48 testów;
- Android Lint — 0 błędów, brak `DrawAllocation` i brak nowych ostrzeżeń;
- pozostało 11 wcześniejszych ostrzeżeń wersji SDK i zależności;
- `git diff --check` — brak błędów spójności zmian.

Pozostało:

- wykonać wizualny test kolizji kilku sąsiadujących ramek na odblokowanym
  telefonie i ocenić czy alfa trackera jest czytelna w pełnym słońcu.

### 2026-08-20 — zaznaczanie zbiorcze i płynniejszy tracker

Problem:

- przy większej liczbie znaków szczegóły karty spychały checkbox zapisu poza
  widoczny obszar poziomej galerii;
- checkbox jednocześnie oznaczał wybór i natychmiast uruchamiał zapis, co
  uniemożliwiało przygotowanie partii cropów;
- tracker zbyt szybko przyjmował małe różnice między kolejnymi detekcjami i
  zbyt długo ekstrapolował ruch.

Rozwiązanie:

- checkbox przeniesiono do nagłówka obok odczytanego numeru i pozostawiono mu
  wyłącznie funkcję zaznaczenia `selectedForSave`;
- szczegóły znaków są prezentowane po dwa na wiersz, maksymalnie w czterech
  wierszach dla typowej tablicy ośmioznakowej;
- w wierszu CTA dodano `Zaznacz wszystkie` i `Zapisz (N)` obok `Start/Stop`;
- zapis wybranej partii wykorzystuje wspólny katalog, wykonuje operacje
  sekwencyjnie, blokuje ponowne CTA w toku i kończy się jednym podsumowaniem;
- zapisane elementy są odznaczane, elementy błędne pozostają zaznaczone do
  ponowienia, a elementy zapisywane nadal są chronione przed wyparciem;
- skrócono predykcję trackera z 220 do 140 ms, obniżono filtr prędkości do
  0,30 i zmiany rozmiaru do 0,22 oraz ograniczono składowe prędkości do ±1,25;
- korekcja położenia używa adaptacyjnego alfa 0,38/0,58/0,78 zależnie od skali
  przesunięcia, dzięki czemu drobny jitter jest tłumiony mocniej niż duży ruch.

Weryfikacja:

- dodano test `smoothsSmallFrameToFrameJitter` potwierdzający, że małe
  przesunięcie nie jest kopiowane bezpośrednio do ramki;
- `gradlew testDebugUnitTest lintDebug assembleDebug` — sukces, 49/49 testów;
- po usunięciu nieużywanych komunikatów zapisowych Android Lint nie zgłasza
  nowych uwag UI; pozostaje bazowe 11 ostrzeżeń wersji SDK i zależności.

Pozostało:

- sprawdzić ergonomię trzech kontrolek CTA oraz pełny zapis wieloelementowej
  partii na odblokowanym telefonie 720×1600.

### 2026-08-20 — wdrożenie eksportu badawczego i ground truth

Problem:

- klasyczny raport ZIP nie zawierał cropów, dokładnych artefaktów modeli ani
  materiałów gotowych do włączenia do pracy inżynierskiej;
- confidence i konsensus były dostępne, ale brakowało ground truth, więc raport
  celowo nie wyliczał exact match ani CER;
- importer usuwał źródłowe archiwum po instalacji, co utrudniało pełne
  odtworzenie konfiguracji MT+MZ w programie macierzystym;
- eksport był składany w `byte[]`, co nie skaluje się do wag modeli i wielu
  cropów na telefonie.

Rozwiązanie:

- dodano cztery stany `human_verification`: `not_reviewed`, `accepted`,
  `rejected`, `corrected`; karta udostępnia zaakceptowanie, odrzucenie i dialog
  wpisania poprawnego numeru;
- ocena człowieka przechowuje oryginalną predykcję, ground truth, czas, rewizję
  i nie zmienia confidence, trackera ani wyniku inferencji;
- `MetricsCollector` wylicza exact match, globalny CER i średnią znormalizowaną
  odległość edycyjną wyłącznie dla `accepted/corrected`;
- jednostką jakościową jest unikalne `session_id + track_id`; spośród wielu
  cropów tracku wybierana jest oceniona obserwacja, dzięki czemu klatki nie
  zawyżają liczebności próby;
- miniraport cropu zawiera walidację i jest aktualizowany również wtedy, gdy
  użytkownik oceni crop już zapisany na nośniku;
- importer zachowuje `source.alprmodel` dla kompletnego pipeline'u i modeli
  pojedynczych; istniejące starsze instalacje są obsługiwane przez fallback
  pełnych katalogów modeli;
- dodano strumieniowy `ResearchArchive`, który nie buforuje wag i JPEG-ów w
  jednym `byte[]`;
- pełny `.alprsession` zawiera raport, trace'y, log, protokół, środowisko,
  manifesty i artefakty modeli, `index.csv`, `annotations.jsonl`, cropy oraz
  SHA-256 wszystkich wpisów;
- `thesis_bundle.zip` zawiera samodzielny `summary.tex`, trzy tabele TeX,
  `references.bib`, `metadata.json`, dane śladów i cropy;
- generator TeX escapuje dane użytkownika i modelu, używa względnych ścieżek
  oraz umieszcza do 12 cropów w głównym dokumencie;
- Diagnostyka oferuje wybór pełnej paczki, skrótu TeX lub klasycznego ZIP;
- na czas eksportu zbieranie jest pauzowane, snapshot cropów chroniony przed
  wyparciem, a po zakończeniu ponownie egzekwowany jest limit galerii;
- dodano schemat `docs/alpr-mobile-research-bundle-v1.schema.json` i opis
  `docs/mobile_research_export.md`.

Weryfikacja:

- testy JVM obejmują odległość Levenshteina oraz bezpieczne escapowanie TeX;
- `gradlew testDebugUnitTest lintDebug assembleDebug assembleDebugAndroidTest`
  — sukces, 51/51 testów JVM i 0 błędów lint;
- `gradlew connectedDebugAndroidTest` — sukces, 3/3 testy na Samsungu SM-A125F
  z Androidem 12;
- test urządzeniowy potwierdził strukturę ZIP, obecność TeX/BibTeX/tabel oraz
  SHA-256 wpisów;
- test urządzeniowy jakości potwierdził deduplikację po tracku, exact match
  0,5 i CER `1/14` dla kontrolowanej pary przykładów.

Pozostało:

- wdrożyć kontrolowany replay co najmniej 1024 próbek/60 s oraz osobne
  oznaczenie cold start;
- dodać importer `alpr.mobile_research_bundle.v1` w programie macierzystym;
- wykonać pełny eksport z rzeczywistym kompletem MT+MZ i cropami na wskazany
  katalog użytkownika oraz skompilować wygenerowany TeX na komputerze;
- rozbudować ground truth MT o referencyjne boxy/narożniki, aby liczyć mAP i
  błąd geometrii, nie tylko jakość sekwencji MZ/end-to-end.

### 2026-08-20 — strategia testowania paczki modeli

Problem:

- istniejące testy jednostkowe dobrze sprawdzają dekodery, geometrię,
  kolejność znaków i raporty, ale nie dowodzą jeszcze, że rzeczywisty eksport
  Python daje na Androidzie ten sam wynik;
- samo poprawne otwarcie `.alprmodel` nie wykrywa błędnego preprocessing,
  layoutu tensora, kolejności klas, NMS, rektyfikacji ani strat kwantyzacji;
- brakowało jednej kolejności testów i twardych warunków odrzucenia pakietu;
- ranking szybkości mógłby błędnie promować wariant niespójny numerycznie albo
  istotnie gorszy jakościowo.

Wnioski:

- jednostką wdrożeniową i końcową jednostką rankingu jest kompletny pakiet
  `MT+MZ`, natomiast MT i MZ testuje się osobno w celu lokalizacji błędu;
- wszystkie warianty runtime porównywane w ramach modelu muszą pochodzić z
  tego samego checkpointu i mieć zgodny SHA-256 źródła;
- testy jakości, wydajności chłodnej i throttlingu są osobnymi eksperymentami;
- confidence nie zastępuje ground truth, a wiele klatek jednego tracku nie
  może zwiększać liczebności exact match;
- `final_test` musi zostać zamrożony przed wyborem pakietu i nie może służyć do
  strojenia progów;
- pakiet najpierw przechodzi bramki twarde; dopiero poprawni kandydaci trafiają
  do rankingu Pareto jakość–czas–rozmiar;
- parytet musi być sprawdzany warstwowo: tensor wejściowy, surowe wyjście,
  dekoder, NMS, rogi, rektyfikacja, znaki i sekwencja;
- INT8 wymaga osobnej bramki jakościowej względem FP32 tego samego checkpointu,
  a mała próba ma dawać wynik `inconclusive`, nie automatyczny sukces.

Rozwiązanie:

- utworzono `docs/model_package_test_strategy.md`;
- zdefiniowano splity `ranking`, `calibration_int8`, `mobile_replay` i
  `final_test` oraz obowiązek wersjonowania ich indeksów i SHA-256;
- zaprojektowano zestaw ważnych i celowo uszkodzonych fixtures pojedynczego
  modelu i kompletnego pipeline'u;
- opisano poziomy T0–T10: preflight, bezpieczeństwo ZIP, kontrakt tensorów,
  parytet, import, runtime, jakość, INT8, benchmark, live/stres i eksport;
- zdefiniowano bramki G0–G9 od integralności do kompletności raportu;
- przyjęto początkowe tolerancje testu parytetu, m.in. IoU boxów co najmniej
  0,98, błąd confidence do 0,02 i błąd narożników do 0,005 przekątnej;
- próbki położone do 0,02 od progu oznaczono jako `threshold_sensitive`, aby
  niestabilność decyzji progowej nie była mylona z błędem kontraktu runtime;
- dla INT8 zapisano początkową, konfigurowalną bramkę: spadek exact match do
  1 p.p., wzrost CER do 0,005 oraz wymagany zysk p90 lub rozmiaru;
- zdefiniowano fazy P0–P4 od testów na commit do zamkniętego final testu i
  archiwizacji artefaktów pracy.

Stan pokrycia:

- istnieją testy ścieżek ZIP, dekoderów raw/end-to-end, geometrii, NMS,
  trackera, stabilizacji, CER/exact match i eksportu badawczego;
- brakuje pełnych fixtures `.alprmodel`, testu importu przez `ContentResolver`,
  golden tensorów/detekcji Python–Android, realnej macierzy runtime i replay
  1024 próbek/60 s;
- brakuje również automatycznej kompilacji TeX oraz importera `.alprsession` w
  programie macierzystym.

Decyzje:

- wartości tolerancji i bramki INT8 są progami startowymi do zatwierdzenia na
  rzeczywistych modelach, a nie wynikami już wykonanego eksperymentu;
- pakiet z niezaliczoną bramką ma status `rejected` albo `inconclusive` i nie
  może odzyskać akceptacji przez wysoki score wydajności;
- wynik nazwany MLPerf pozostaje niedozwolony bez oficjalnego LoadGena i reguł
  submission; aplikacja używa określenia „MLPerf Mobile inspired”.

Weryfikacja:

- porównano strategię z aktualnym kontraktem `alpr.model.v1`, kompletnym
  `alpr.package.v1`, raportem badawczym i faktycznym zestawem testów Android;
- strategia nie zmienia kodu wykonawczego ani wyników wcześniejszych 51 testów
  JVM i 3 testów instrumentalnych;
- dokument wyraźnie oznacza testy istniejące, częściowe i brakujące.

Pozostało:

- utworzyć katalog fixtures i zaimplementować T1/T4 dla pełnych paczek;
- wyeksportować pierwszy rzeczywisty komplet MT+MZ oraz przygotować golden
  tensorów i detekcji po stronie Python;
- wdrożyć tryb `mobile_replay` i plik konfiguracyjny zatwierdzonych progów
  jakości/wydajności dla urządzenia docelowego.

### 2026-08-20 — model pojazdów: konwersja desktopowa, import mobilny

Problem:

- rozważono możliwość importowania dowolnego checkpointu YOLO pojazdów i jego
  konwersji bezpośrednio na telefonie;
- należało rozstrzygnąć, czy moduł `.pt → ONNX/TFLite/INT8` jest zasadny dla
  urządzenia docelowego oraz czy dodatkowy MP faktycznie przyspieszy pipeline.

Pomiary urządzenia:

- fizyczny telefon: Samsung SM-A125F, ABI `arm64-v8a` z fallbackiem
  `armeabi-v7a`;
- całkowity RAM odczytany z `/proc/meminfo`: około 3,84 GB;
- dostępny RAM w chwili pomiaru: około 1,49 GB;
- wolne miejsce w `/data`: około 14 GB z 51 GB;
- wartości pamięci są snapshotem systemu i mogą zmieniać się między sesjami.

Wniosek:

- telefon ma wystarczającą przestrzeń na gotowy model mobilny, ale nie jest
  odpowiednim środowiskiem do konwersji checkpointu PyTorch;
- konwersja na Androidzie wymagałaby ciężkiego stosu PyTorch, Ultralytics,
  ONNX i często TensorFlow, zwiększyłaby APK oraz ryzyko OOM, throttlingu,
  długiego czasu operacji i niezgodności operatorów;
- odpowiedzialność za konwersję `best.pt` pozostaje w programie macierzystym;
- Android ma wyłącznie walidować, importować, zachowywać i autotuningować
  gotowy `alpr.model.v1` z `role: vehicle`, `task: detect`, wariantami runtime i
  SHA-256;
- pobieranie z sieci, jeżeli zostanie dodane, powinno dotyczyć gotowych,
  wersjonowanych paczek `.alprmodel`, a nie uruchamiania konwertera na telefonie.

Stan architektury:

- rejestr i importer Androida obsługują już rolę `vehicle`;
- ekran Opcje pozwala wybrać model pojazdu i włączyć kaskadę MP → ROI → MT → MZ;
- brakuje wygodnej, jednoznacznej ścieżki eksportu MP w programie Python oraz
  rzeczywistego pakietu do testów macierzy runtime;
- import gotowego MP jest małym rozszerzeniem istniejącego przepływu, natomiast
  konwerter `.pt` na Androidzie byłby dużym, odrębnym projektem.

Parametry startowe kandydata MP:

- rodzina YOLO `n`;
- wejście 320×320 albo 416×416;
- TFLite FP32 jako wariant referencyjny;
- INT8 wyłącznie po przejściu bramki jakości względem FP32;
- MP wykonywany najwyżej co trzy analizowane klatki;
- maksymalnie dwa ROI i obowiązkowy pełnoklatkowy fallback MT.

Decyzja pomiarowa:

- dodatkowy MP nie jest automatycznie optymalizacją: jeżeli jego koszt jest
  większy niż oszczędność MT na ROI, może pogorszyć opóźnienie i dropped frames;
- wymagany jest test A/B tego samego kompletu MT+MZ: pełna klatka MT kontra
  MP → ROI → MT;
- porównanie obejmuje p90 całego pipeline'u, dropped frames, termikę, recall MT,
  exact match i CER end-to-end;
- MP może zostać aktywowany jako rekomendowany tylko wtedy, gdy poprawia koszt
  całego pipeline'u bez przekroczenia zamrożonej bramki jakościowej.

Pozostało:

- dodać/zweryfikować eksport `role: vehicle` w programie macierzystym;
- przygotować paczkę MP z TFLite FP32, kandydatem INT8 i ONNX FP32;
- wykonać T0–T9 zgodnie z `docs/model_package_test_strategy.md`, w tym test A/B
  na SM-A125F;
- dobrać wejście, częstotliwość MP i limity ROI na podstawie pomiaru, a nie
  przyjąć wartości startowe jako wynik końcowy.

### 2026-08-20 — obsługa kompletnej paczki MP+MT+MZ na Androidzie

Problem:

- samo włączenie kaskady pojazdów nie mogło uruchomić MP, jeżeli użytkownik nie
  zaimportował osobnego pakietu `role: vehicle`;
- dotychczasowy `alpr.package.v1` przenosił wyłącznie MT i MZ, więc eksporter
  desktopowy nie miał kontraktu pozwalającego dostarczyć całej kaskady w jednym
  artefakcie;
- należało zachować zgodność z już istniejącymi paczkami MT+MZ.

Rozwiązanie mobilne:

- `AlprPackageManifest` obsługuje opcjonalny wpis `models.vehicle` z rolą
  `vehicle` i zadaniem `detect`;
- paczka historyczna bez MP nadal ma dokładnie cztery etapy, a paczka z MP ma
  dokładnie pięć etapów i obowiązkowo rozpoczyna się od `vehicle_detection`;
- importer waliduje pakiet MP, manifest boczny i obie sumy SHA-256 tak samo jak
  dla MT i MZ;
- instalacja kompletu rejestruje `vehicle_storage_id`, a aktywacja kompletu
  ustawia wszystkie zawarte modele jedną transakcją;
- rejestr odtwarza komplet MP+MT+MZ po restarcie i odrzuca rekord, w którym
  manifest oraz `installation.json` nie zgadzają się co do obecności MP;
- dokładny źródłowy `source.alprmodel` kompletnej paczki jest zachowywany do
  eksportu badawczego, bez powielania zagnieżdżonych archiwów MP/MT/MZ;
- import i aktywacja MP nie włącza automatycznie przełącznika kaskady — decyzja
  o użyciu MP pozostaje jawna i odwracalna w ekranie Opcje.

Kontrakt dla eksportera desktopowego:

- dodano `docs/alpr-package-v1.schema.json` z warunkową walidacją wariantu
  cztero- i pięcioetapowego;
- `docs/model_package_v1.md` zawiera strukturę katalogów, pełny przykład wpisów
  MP/MT/MZ, kolejność pipeline'u i kolejność liczenia SHA-256;
- eksporter powinien najpierw zakończyć i zwalidować trzy potomne
  `alpr.model.v1`, następnie zapisać zgodne manifesty boczne, obliczyć hashe
  finalnych plików i dopiero na końcu utworzyć nadrzędny manifest oraz archiwum;
- kod programu desktopowego nie znajduje się w repozytorium Android, dlatego
  ten etap ustanawia gotowy kontrakt odbiorczy, ale nie jest implementacją
  samego eksportera Python.

Weryfikacja:

- kompilacja `testDebugUnitTest assembleDebug`: sukces, 51/51 testów JVM;
- kompilacja `assembleDebugAndroidTest`: sukces;
- testy na fizycznym Samsungu SM-A125F z Androidem 12: 7/7 sukces;
- nowe testy instrumentalne potwierdziły przyjęcie historycznej paczki
  czteroetapowej, przyjęcie kaskady pięcioetapowej oraz odrzucenie obu
  niespójności: MP bez etapu i etap bez MP.

Pozostało:

- zaimplementować generowanie opisanego wariantu w repozytorium aplikacji
  desktopowej;
- wyeksportować rzeczywisty MP+MT+MZ, wykonać pełny import przez SAF i
  potwierdzić inferencję wszystkich trzech rzeczywistych modeli;
- porównać A/B pełną klatkę MT z MP → ROI → MT według zamrożonych bramek
  jakościowych i wydajnościowych.

### 2026-08-21 — trwała, zwijana galeria przy obrocie ekranu

Problem:

- widoczność listy była bezpośrednio zależna od liczby cropów, bez osobnego
  sterowania panelem galerii;
- `MainActivity.onDestroy()` usuwało i zwalniało bitmapy także podczas zwykłej
  zmiany konfiguracji, dlatego obrót ekranu zerował galerię;
- zatrzymanie kolekcji i pusty stan nie dawały użytkownikowi jednoznacznego,
  stale dostępnego punktu wejścia do galerii;
- rozwinięty panel o stałej wysokości mógł przekraczać wysokość ekranu w
  orientacji poziomej.

Rozwiązanie:

- dodano stale widoczny przycisk `Pokaż/Ukryj` z ikoną kierunku oraz licznikiem
  cropów niezależnym od stanu Start/Stop;
- pusta, rozwinięta galeria pokazuje jawny komunikat zamiast znikać;
- `CaptureGalleryViewModel` zachowuje bitmapy, stan sesji, numer sekwencji,
  informacje próbkowania i `MetricsCollector` podczas odtwarzania aktywności;
- bitmapy są zwalniane dopiero po rzeczywistym zamknięciu właściciela stanu,
  nie przy obrocie;
- stan zwinięcia galerii również przechodzi przez zmianę konfiguracji;
- panel sterowania jest ograniczony do dostępnej wysokości i przewijany
  pionowo, dzięki czemu pozostaje obsługiwalny w orientacji poziomej.
- po korekcie `NestedScrollView` zwinięty panel używa ponownie wyłącznie
  `wrap_content`; limit wysokości jest nakładany dynamicznie tylko na
  rozwiniętą galerię w orientacji poziomej, więc podgląd kamery odzyskuje całą
  wolną przestrzeń w pionie.

Weryfikacja:

- `testDebugUnitTest`, `assembleDebug`, `assembleDebugAndroidTest` i
  `lintDebug`: sukces;
- pełny zestaw testów instrumentalnych na Samsungu SM-A125F: 8/8 sukces;
- test stanu potwierdził zachowanie cropu, aktywnej sesji, numeru sekwencji i
  zwinięcia po odtworzeniu właściciela oraz zwolnienie bitmapy po jego
  ostatecznym zamknięciu;
- wykonano kontrolę wizualną na urządzeniu: pusta galeria rozwinięta, galeria
  zwinięta oraz obrót do orientacji poziomej z zachowaniem przycisku i stanu;
- dodatkowy zrzut pionowego widoku potwierdził brak rezerwowanego pustego
  obszaru po zwinięciu galerii;
- finalny APK został zainstalowany na telefonie.

### 2026-08-21 — dwuslotowy pasek i maksymalizacja galerii

Problem:

- strzałka zwijania nie komunikowała jednoznacznie, że steruje galerią obrazów;
- chronologiczna kolejność adaptera umieszczała najnowszy odczyt na końcu;
- pasek nie miał określonej liczby widocznych slotów ani większego trybu
  przeglądania.

Rozwiązanie:

- dodano wektorowe piktogramy obrazu dla `Pokaż/Ukryj` oraz osobne piktogramy
  maksymalizacji i przywrócenia paska;
- adapter odwraca kolejność wyłącznie w warstwie prezentacji: najnowszy crop
  zajmuje slot 1, natomiast chronologia źródłowa raportów pozostaje bez zmian;
- standardowy pasek wylicza szerokość kafelka jako połowę dostępnego viewportu,
  tworząc dwuslotowe okno przesuwane poziomym gestem;
- dopływ nowego wyniku pokazuje go na początku tylko wtedy, gdy użytkownik jest
  przy początku listy; podczas przeglądania historii zachowywany jest bieżący
  element i offset, więc kolekcja nie przejmuje przewijania;
- maksymalizacja rozwija panel do dostępnej wysokości i przełącza RecyclerView
  na pionową, dwukolumnową siatkę, pokazując więcej kafelków;
- stan maksymalizacji jest zachowywany razem ze stanem galerii przy obrocie;
- checkbox kafelka pozostał samym polem wyboru, bez tekstu zajmującego miejsce;
  opis dostępności nadal informuje o stanie wyboru/zapisu.

Weryfikacja:

- `testDebugUnitTest`, `assembleDebug`, `assembleDebugAndroidTest` i
  `lintDebug`: sukces;
- pełny zestaw testów na Samsungu SM-A125F: 9/9 sukces;
- nowy test adaptera potwierdził kolejność `najnowszy → starsze`;
- kontrola wizualna potwierdziła piktogram obrazu, przycisk maksymalizacji i
  brak ponownego pojawienia się pustego obszaru po zwinięciu;
- finalny APK został zainstalowany na telefonie.

## 6. Najważniejsze decyzje projektowe

### Pakiet zamiast surowych wag

Android nie interpretuje `best.pt`. Format `.alprmodel` oddziela środowisko
treningowe od wdrożeniowego i przenosi pełny kontrakt inferencji.

### Walidacja po obu stronach

Eksporter Python waliduje artefakt przed zapisem, ale Android powtarza
walidację. Chroni to przed uszkodzeniem podczas przenoszenia i przed ręcznie
zmodyfikowanym pakietem.

### Osobny wariant dla każdego etapu

MT i MZ mogą mieć inne wymagania sprzętowe. Autotuning i fallback są wykonywane
per model, nie globalnie dla całego kompletu.

### NMS poza grafem

Modele są eksportowane z `nms=false`. Android wykonuje wspólny, kontrolowany
postprocessing i stosuje progi zapisane w manifestach.

### Brak udawanej jakości

Confidence modelu i częstość uzyskania tekstu nie są dokładnością. CER i exact
match wymagają znanej odpowiedzi referencyjnej.

### Prywatne i trwałe przechowywanie

Modele są rozpakowywane do prywatnego katalogu aplikacji. Aktywne rekordy są
odtwarzane po restarcie na podstawie rejestru i fingerprintów.

## 7. Weryfikacja

Stan na 2026-08-21:

| Kontrola | Wynik |
| --- | --- |
| `gradlew testDebugUnitTest` | sukces, 51/51 testów JVM |
| `gradlew assembleDebug` | sukces |
| `gradlew connectedDebugAndroidTest` | sukces, 9/9 testów na SM-A125F |
| `gradlew lintDebug` | sukces, 0 błędów |
| Android Lint | 11 ostrzeżeń o dostępnych nowszych SDK/zależnościach |
| Parser raportu desktopowego | raport przyjęty, latency i memory odczytane |
| Bramka jakości bez ground truth | poprawnie odrzucona |
| Test rzeczywistego MT+MZ na telefonie | jeszcze niewykonany |

Ostrzeżenia Lint nie dotyczą nowego importera ani raportu. Informują o
dostępnych nowszych wersjach SDK i bibliotek. Aktualizacja zależności powinna
być osobnym zadaniem z testami regresji runtime'ów ML.

Aktualny artefakt debug:

```text
app/build/outputs/apk/debug/app-debug.apk
```

## 8. Otwarte zadania i ryzyka

### Backlog raportu badawczego i eksportu do pracy inżynierskiej

Status: rdzeń eksportu, manualna walidacja i metryki sekwencji są wdrożone.
Kontrolowany replay oraz importer w programie macierzystym pozostają otwarte.

#### Cel

Raport ma umożliwić powiązanie trzech warstw, których nie wolno mieszać:

1. konfiguracji eksportowej dokładnego kompletu MT+MZ;
2. rzeczywistego wykonania tego kompletu na konkretnym telefonie;
3. skuteczności na materiale z jawnym ground truth.

Confidence modelu, częstość pojawienia się wyniku i konsensus czasowy nie są
miarą poprawności. Exact match i Character Error Rate (CER) mogą zostać
wyliczone dopiero po przypisaniu referencyjnego ciągu do cropu albo tracku.

#### Dwa scenariusze pomiarowe

- **benchmark kontrolowany/replay** — stały, wersjonowany zbiór cropów lub
  klatek w niezmiennej kolejności, identyczny dla wszystkich wariantów MT+MZ;
  służy porównaniu runtime'u, precyzji, rozdzielczości i progów;
- **sesja terenowa/live** — rzeczywista kamera w ruchu, tracker, scheduler,
  opcjonalny MP i konsensus; służy ocenie czasu do pierwszego wyniku, czasu do
  potwierdzenia, pokrycia odczytem, stabilności i zachowania termicznego.

Wyników obu scenariuszy nie należy łączyć w jedną średnią. Replay izoluje
konfigurację modelu, natomiast live ocenia kompletny system i warunki akwizycji.

#### Protokół wydajności

- osobno raportować cold start: ładowanie modelu, inicjalizację delegata i
  pierwszą inferencję;
- steady state mierzyć zegarem monotonicznym per etap: konwersja kamery, MP,
  MT preprocessing/inference/postprocessing, rektyfikacja oraz MZ
  preprocessing/inference/postprocessing;
- dla replay przyjąć metodykę inspirowaną MLPerf Mobile single-stream: co
  najmniej 1024 próbki i co najmniej 60 s, z p90 jako wynikiem głównym; p50,
  p95 i p99 zachować jako diagnostykę rozkładu;
- jeżeli zbiór nie pozwala spełnić 1024 prób, zapisać rzeczywiste `n` i czas
  trwania oraz nie opisywać wyniku jako zgodnego z MLPerf;
- wykonać co najmniej trzy powtórzenia każdej konfiguracji; publikować wyniki
  wszystkich powtórzeń, medianę między seriami i surowe ślady;
- benchmark chłodny wykonywać przy temperaturze otoczenia 20–25°C, z telefonem
  wentylowanym i co najmniej 10 min przerwy między pełnymi seriami; osobny test
  długotrwały ma celowo pokazać throttling zamiast ukrywać go w wyniku chłodnym;
- zapisywać stan termiczny Androida, poziom i stan ładowania baterii, pamięć,
  liczbę wątków, delegata, ABI, wersję Androida, wersje runtime'ów oraz liczbę
  pominiętych klatek.

Metodyka jest inspirowana MLPerf Mobile, ale aplikacja nie powinna używać nazwy
„wynik MLPerf”, dopóki nie spełnia pełnych reguł i nie korzysta z LoadGena.

#### Metryki jakości

- **MT / detekcja tablicy**: precision, recall, F1, mAP@0.5 i mAP@0.5:0.95,
  jeżeli dostępne są referencyjne boxy; dla YOLO Pose dodatkowo błąd narożników
  znormalizowany przekątną tablicy oraz udział poprawnych rektyfikacji;
- **MZ / detekcja znaków**: exact match pełnego ciągu, CER oparty na odległości
  Levenshteina z równym kosztem wstawienia, usunięcia i zamiany, accuracy per
  znak, macierz pomyłek i udział cropów zakończonych odczytem;
- **end-to-end**: exact match liczony per unikalna tablica/track, czas do
  pierwszego wyniku i potwierdzenia, liczba prób MZ do wyniku, udział tracków
  bez odczytu oraz skuteczność w przedziałach confidence;
- wyniki dzielić co najmniej według wysokości tablicy w pikselach, ostrości,
  ruchu kamery i trybu rozdzielczości; opcjonalnie według oświetlenia, kąta i
  liczby wierszy tablicy;
- jedna tablica obserwowana w wielu klatkach jest jedną jednostką jakościową;
  klatki nie mogą sztucznie zwiększać liczebności próby exact match.

#### Manualna walidacja — backlog

Karta galerii powinna docelowo otrzymać niezależny od zaznaczenia do zapisu
stan `human_verification`:

- `not_reviewed` — brak oceny, nigdy nieinterpretowany jako błąd;
- `accepted` — ciąg modelu jest dokładnie poprawny;
- `rejected` — odczyt jest błędny, ale brak transkrypcji referencyjnej;
- `corrected` — użytkownik podał `ground_truth_text`.

Rekord powinien zawierać `verified_at`, oryginalny niezmieniony prediction,
tekst poprawiony, wersję rewizji i opcjonalną uwagę. Ocena człowieka nie zmienia
confidence, trackera ani wyniku inferencji. Do CER i exact match kwalifikują się
rekordy `accepted` i `corrected`; samo `rejected` pozwala liczyć acceptance
rate, ale nie pozwala wyznaczyć liczby błędnych znaków.

#### Artefakt 1 — samodzielna paczka dla programu macierzystego

Planowane rozszerzenie: `.alprsession` będące archiwum ZIP ze schematem
`alpr.mobile_research_bundle.v1`:

```text
session.alprsession
├── manifest.json                 # schemat, wersje i SHA-256 wszystkich plików
├── README.md                     # protokół i warunki odtworzenia
├── pipeline/
│   ├── pipeline.alprmodel        # dokładny kompletny pakiet MT+MZ, opcjonalnie MP
│   ├── package_manifest.json
│   ├── mt_manifest.json
│   └── mz_manifest.json
├── environment/
│   ├── device.json
│   └── software.json
├── runs/
│   └── <run_id>/
│       ├── report.json
│       ├── traces.csv
│       └── events.log
└── samples/
    ├── index.csv
    ├── annotations.jsonl
    └── crops/<capture_id>.jpg
```

Wariant samodzielny zawiera dokładne wagi użyte na telefonie, dzięki czemu
program Python nie musi odgadywać konfiguracji na podstawie nazw. Dla małego
eksportu można później dodać profil `manifest_only`, ale nie powinien on być
używany jako jedyny artefakt reprodukowalnego eksperymentu.

`manifest.json` musi jednoznacznie zestawiać dla MT i MZ: `model_id`, wersję,
SHA-256 checkpointu i artefaktu, rodzinę i rozmiar YOLO, liczbę parametrów,
rozmiar wejścia, układ i typ tensora, runtime, precyzję/kwantyzację, wariant,
opset lub wersję TFLite, dekoder, progi confidence/IoU, NMS, labels, rozmiar
pliku, delegata, liczbę wątków oraz identyfikator commita eksportera.

#### Artefakt 2 — skrót TeX z paczką cropów

Eksport `thesis_bundle.zip` powinien być mały i bez zależności od programu
macierzystego:

```text
thesis_bundle.zip
├── summary.tex
├── references.bib
├── metadata.json
├── tables/
│   ├── configuration_mt_mz.tex
│   ├── mobile_quality.tex
│   └── mobile_latency.tex
└── figures/
    ├── selected_crops/*.jpg
    └── latency_distribution.pdf
```

`summary.tex` ma używać `\input{}` dla tabel i względnych ścieżek do figur,
zawierać podpisy z `capture_id`, prediction, ground truth, confidence MT/MZ i
czasem inferencji oraz nie wymagać ręcznego przepisywania liczb. Generator musi
escapować znaki specjalne TeX i oferować pseudonimizację tablic/cropów przed
umieszczeniem materiału w publicznej wersji pracy.

#### Literatura dla backlogu raportowego

- [B1] V. J. Reddi i in., „MLPerf Mobile Inference Benchmark: An
  Industry-Standard Open-Source Machine Learning Benchmark for On-Device AI”,
  *Proceedings of Machine Learning and Systems*, vol. 4, s. 352–369, 2022,
  arXiv: `2012.02328`. Publikacja nie ma przypisanego DOI ani ISBN; oficjalny
  artefakt i reguły publikuje MLCommons.
- [B2] R. Laroca i in., „A Robust Real-Time Automatic License Plate
  Recognition Based on the YOLO Detector”, *IJCNN 2018*, s. 1–10. DOI:
  `10.1109/IJCNN.2018.8489629`; ISBN online: `978-1-5090-6014-6`.
- [B3] A. Shahab, F. Shafait, A. Dengel, „ICDAR 2011 Robust Reading
  Competition Challenge 2: Reading Text in Scene Images”, *ICDAR 2011*,
  s. 1491–1496. DOI: `10.1109/ICDAR.2011.296`; ISBN drukowany:
  `978-1-4577-1350-7`; ISBN elektroniczny: `978-0-7695-4520-2`.
- [B4] M. Mitchell i in., „Model Cards for Model Reporting”, `FAT* '19`,
  s. 220–229. DOI: `10.1145/3287560.3287596`; ISBN:
  `978-1-4503-6125-5`.
- [B5] S. M. Silva, C. R. Jung, „License Plate Detection and Recognition in
  Unconstrained Scenarios”, *ECCV 2018*, LNCS 11216, s. 593–609. DOI:
  `10.1007/978-3-030-01258-8_36`; eISBN tomu: `978-3-030-01258-8`.

Wniosek z [B1] i [B4] uzasadnia pełne raportowanie konfiguracji systemu i
artefaktów zamiast samej nazwy modelu. [B2] i [B5] uzasadniają oddzielne
raportowanie skuteczności i szybkości całego potoku ALPR w warunkach
niekontrolowanych. [B3] uzasadnia exact match oraz znormalizowaną odległość
edycyjną dla sekwencji znaków.

### Priorytet wysoki

- wyeksportować rzeczywisty komplet `alpr.package.v1` w aplikacji Python;
- zaimportować go na fizycznym telefonie;
- potwierdzić uruchomienie MT, rektyfikacji i MZ na realnym obrazie;
- sprawdzić TFLite CPU, TFLite GPU i ONNX CPU;
- porównać wartości wyjściowe Androida z inferencją referencyjną Python;
- przygotować benchmark z obrazami i poprawnymi numerami rejestracyjnymi;
- uzupełnić ocenę MT o mAP i błąd narożników z geometrycznego ground truth;
- wdrożyć kontrolowany replay oraz importer `.alprsession` po stronie Python.
- wyeksportować gotowy model pojazdów jako `role: vehicle` i wykonać test A/B
  pełna klatka MT kontra MP → ROI → MT na SM-A125F.

### Priorytet średni

- dodać jawną bramkę zatwierdzającą INT8 po porównaniu jakości z FP32;
- dodać ekran historii zaimportowanych kompletów i ręcznego wyboru aktywnego
  pipeline'u;
- dodać testy instrumentalne importu przez Android `ContentResolver`;
- dodać fixture poprawnego i uszkodzonego `.alprmodel` do testów regresji;
- mierzyć stabilność dłuższych sesji i zachowanie termiczne urządzenia;
- doprecyzować obsługę błędów runtime w rankingu desktopowym.

### Kierunki opcjonalne

- opcjonalny profil NCNN/Vulkan po benchmarku stabilności i energii;
- benchmark NNAPI/NPU, jeżeli wybrany runtime i urządzenie zapewnią stabilne
  wsparcie;
- tryb testowania samego MZ na gotowych cropach tablic;
- porównanie kilku kompletów MT+MZ bez ponownego importowania plików.

### 2026-08-21 — kompozycja uruchomieniowa modeli

Problem:
- kompletny pakiet `MP+MT+MZ` albo `MT+MZ` jest obecnie aktywowany jako stały
  zestaw, a podmiana pojedynczego modelu usuwa powiązanie z pakietem bazowym;
- pakiet może zawierać kilka wariantów wykonawczych jednego modelu, lecz klient
  nie udostępnia trwałego wyboru wariantu osobno dla MP, MT i MZ;
- raport nie rozróżnia pakietu bazowego, podmienionych modeli i ręcznie
  przypiętych wariantów.

Rozwiązanie:
- wprowadzić trwałą kompozycję uruchomieniową opartą na opcjonalnym pakiecie
  bazowym oraz aktywnych modelach MP, MT i MZ;
- umożliwić atomową podmianę jednego lub kilku modeli bez modyfikowania
  źródłowego `.alprmodel`;
- dodać dla każdej roli tryb wariantu `auto` albo `pinned` z identyfikatorem
  wariantu; nieobsługiwane runtime'y mają pozostać widoczne, ale niedostępne;
- zachować osobne wejście, wyjście, runtime, precyzję, delegata i autotuning dla
  każdego etapu;
- po zmianie kompozycji odtworzyć silnik pipeline'u i wyczyścić stan śledzenia,
  bez usuwania zebranej galerii;
- raportować pakiet bazowy, identyfikator kompozycji, pochodzenie modeli,
  wybrane warianty i informację, czy kompozycja została zmodyfikowana;
- dodać testy trwałości konfiguracji, walidacji ról, ręcznego wariantu,
  przywracania pakietu bazowego oraz zachowania po restarcie.

Decyzje:
- źródłowe paczki pozostają niemutowalne; kompozycja jest lokalnym rekordem
  konfiguracji i wskazuje już zainstalowane modele;
- MT i MZ są obowiązkowe, MP pozostaje opcjonalny;
- wariant ręczny ma pierwszeństwo przed autotuningiem, a `auto` zachowuje
  bezpieczny wybór FP32; INT8 wymaga jawnego wyboru do czasu wdrożenia bramki
  jakości, a NCNN nie może zostać aktywowany bez runtime'u JNI.

Weryfikacja:
- `compileDebugJavaWithJavac` i `compileDebugAndroidTestJavaWithJavac` zakończone
  powodzeniem;
- dwa izolowane testy instrumentalne przeszły na fizycznym SM-A125F: trwałość
  pakietu bazowego i podmiany modelu oraz ręczny wybór ONNX/powrót do Auto z
  blokadą NCNN bez backendu;
- `testDebugUnitTest`, `lintDebug` i `assembleDebug` zakończone powodzeniem.

Pozostało:
- ręcznie przejść nowy modal Opcji na docelowych paczkach `MT+MZ` i
  `MP+MT+MZ`, uruchomić inferencję dla każdej wybranej kombinacji i sprawdzić
  wynikowy `.alprsession`;
- oddzielnie potwierdzić prawdziwe artefakty eksportera `start4` na urządzeniu.

### 2026-08-21 — degradacja pakietu z MP tylko w NCNN

Problem:
- rzeczywisty pakiet `MP+MT+MZ` ze `start4` został poprawnie rozpakowany i
  zwalidowany, lecz aktywacja całego kompletu była odrzucana, ponieważ MP miał
  wyłącznie wariant `ncnn-fp32`, a klient nie zawiera jeszcze backendu NCNN;
- wykonywalne warianty `tflite-int8` modeli MT i MZ nie były przez to
  aktywowane.

Rozwiązanie:
- brak wykonywalnego wariantu opcjonalnego MP nie blokuje już aktywacji MT+MZ;
- kompletny pakiet pozostaje pakietem bazowym, MP jest zainstalowany, lecz
  nieaktywny, a kompozycja otrzymuje stan zmodyfikowany;
- użytkownik może później podmienić tylko MP na pakiet TFLite/ONNX;
- niewykonywalne modele pozostają widoczne w selektorze, ale nie można ich
  aktywować; komunikat importu wyjaśnia częściową aktywację.

Weryfikacja:
- manifest rzeczywistego eksportu potwierdził `vehicle/ncnn-fp32`,
  `plate/tflite-int8` i `character/tflite-int8`;
- trzy testy kompozycji, w tym degradacja MP-NCNN do MT+MZ, przeszły na
  fizycznym SM-A125F;
- ponowny import rzeczywistego archiwum 19,4 MB na SM-A125F zakończył się
  powodzeniem; ekran Opcji pokazał bazę `MP+MT+MZ`, nieaktywne MP oraz aktywne
  `MT/tflite-int8` i `MZ/tflite-int8`;
- `testDebugUnitTest`, `lintDebug`, `assembleDebug` i `installDebug` zakończone
  powodzeniem.

Pozostało:
- eksporter `start4` powinien ostrzegać lub blokować kompletny pakiet docelowy
  dla Androida, jeżeli którykolwiek wymagany aktywny etap nie ma TFLite/ONNX;
- backend NCNN/JNI pozostaje osobnym zadaniem.

### 2026-08-21 — graf kompozycji i porządkowanie Opcji

Problem:
- karta modeli miała kilka konkurujących przycisków importu/konfiguracji i
  przedstawiała kompozycję głównie jako długi tekst;
- nazwy profili `Szybka` i `Daleka` sugerowały typ kamery zamiast rozdzielczości
  klatek analizy;
- skrajne etykiety `Auto` i `100` w pasku limitu cropów były ucinane.

Rozwiązanie:
- pozostawiono jeden główny przycisk `Importuj .alprmodel`;
- dodano klikalny graf `MP → MT → MZ`; węzeł pokazuje runtime i precyzję, a jego
  menu pozwala zmienić zainstalowany model/format albo uruchomić import
  przypisany do konkretnej roli;
- import z węzła sprawdza zgodność roli przed aktywacją modelu;
- profile opisano jako `Rozdzielczość analizy`: Auto, Szybkość 640×480 i Daleki
  plan 1920×1080;
- usunięto minimalną szerokość i wewnętrzne poziome odstępy przycisków limitu,
  a skrajnym opcjom przydzielono nieco większy udział szerokości.

Weryfikacja:
- kontrola wizualna na SM-A125F (720×1600) potwierdziła pełne etykiety profili,
  wartości `Auto`/`100`, poprawny graf i menu węzła MT;
- pełny zestaw 13 testów instrumentalnych przeszedł na SM-A125F.

Pozostało:
- sprawdzić import pojedynczych modeli przez każdy z trzech węzłów na docelowych
  artefaktach MP, MT i MZ.

### 2026-08-21 — backend NCNN na Androidzie

Problem:
- pakiet `MP+MT+MZ` wyeksportowany ze `start4` zawierał dla MP wyłącznie wariant
  `ncnn-fp32`; Android potrafił go zwalidować i zainstalować, ale wcześniej nie
  mógł uruchomić MP ani aktywować kompletnej, niezmodyfikowanej kompozycji;
- manifest wspólny dla wariantów opisywał wyjście end-to-end, podczas gdy eksport
  PNNX/NCNN modelu YOLO wystawia surowy tensor detekcji przed NMS.

Rozwiązanie:
- dodano oficjalny, CPU-only runtime NCNN `20260526` dla `arm64-v8a` i
  `armeabi-v7a`, budowany z aplikacją przez CMake jako bibliotekę JNI
  `libalpr_ncnn.so`;
- dodano `NcnnBackend`, który ładuje parę `.param`/`.bin`, przyjmuje bezpośredni
  tensor `NCHW/FLOAT32`, steruje liczbą wątków CPU i zwraca kształt oraz dane
  wyjściowe do istniejącego potoku Java;
- NCNN jest uwzględniany przez fabrykę backendów, rejestr wykonywalnych modeli,
  wybór wariantu, autotuning i aktywację kompletnego pakietu;
- gdy wariant NCNN nie ma własnego opisu wyjścia, wspólny kontrakt end-to-end jest
  lokalnie zamieniany na `ultralytics_detect_raw_v1`, układ anchors-first, `xywh`
  i NMS po stronie aplikacji; źródłowy manifest i pozostałe warianty nie są
  modyfikowane;
- w repozytorium zapisano licencję, pochodzenie paczki i sumę SHA-256 oficjalnego
  archiwum: `85b18b875488585c2d21360430e0e54abb6c04aa88094b471c20208ab55ff796`.

Decyzje:
- pierwszy etap NCNN korzysta wyłącznie z CPU; Vulkan pozostaje wyłączony do czasu
  osobnego benchmarku stabilności, zużycia energii i kompatybilności urządzeń;
- most JNI v1 świadomie akceptuje jeden tensor wejściowy i jeden dwuwymiarowy
  tensor wyjściowy bez packingu, zgodnie z aktualnym eksportem MP; inne topologie
  są odrzucane czytelnym błędem zamiast niejawnej interpretacji danych;
- komplet z wykonywalnym MP/NCNN jest ponownie traktowany jako dokładny aktywny
  pakiet, a wcześniejsza degradacja do MT+MZ pozostaje mechanizmem awaryjnym tylko
  dla urządzeń lub ABI bez dostępnego backendu.

Weryfikacja:
- deterministyczny test JNI `Input → Reshape` potwierdził na fizycznym SM-A125F
  ładowanie `.param/.bin`, przekazanie bezpośredniego bufora NCHW, wykonanie sieci,
  kształt `[1,3,4]` i zgodność wartości;
- ponowny import rzeczywistego archiwum 19,41 MB zakończył się aktywacją pełnego
  pakietu; graf pokazuje `MP/NCNN/FP32`, `MT/TFLITE/INT8` i `MZ/TFLITE/INT8`;
- realny model `vehicle-yolo26n` wykonał inferencję oraz autotuning na SM-A125F:
  mediana `972,04 ms` dla 1 wątku, `540,65 ms` dla 2 wątków i `438,24 ms` dla
  4 wątków; po czystej reinstalacji powtórny pomiar wyniósł odpowiednio
  `969,62 ms`, `536,26 ms` i `372,72 ms`; w obu przebiegach wybrano profil
  NCNN/CPU z 4 wątkami;
- kaskada `MP → MT` pracowała na obrazie z kamery bez błędu kształtu, dekodera ani
  wyjątku runtime;
- pełny zestaw 14 testów instrumentalnych przeszedł na SM-A125F; dodatkowo
  `testDebugUnitTest`, `lintDebug` i `assembleDebug` zakończyły się powodzeniem,
  łącznie z natywnym buildem obu ABI.

Pozostało:
- porównać wyniki detekcji MP z referencyjną inferencją Python na kontrolowanym
  zbiorze obrazów, a następnie zmierzyć pełne A/B: MT na całej klatce kontra
  `MP → ROI → MT`;
- rozważyć Vulkan dopiero po pomiarach termicznych i energetycznych oraz rozszerzyć
  JNI, jeżeli kolejne eksporty będą miały wiele wejść/wyjść albo inny wymiar
  tensora.

### 2026-08-21 — sterowanie kaskadą bezpośrednio w grafie modeli

Problem:
- osobny przełącznik `Kaskada pojazd → tablica` dublował znaczenie aktywnego
  węzła MP i po dodaniu grafu kompozycji nie wskazywał jednoznacznie, którego
  etapu dotyczy;
- stan udziału modelu w potoku był oddzielony wizualnie od jego runtime'u i
  formatu.

Rozwiązanie:
- pod każdym węzłem `MP`, `MT` i `MZ` dodano kompaktowy badge stanu;
- badge MP `WŁ./WYŁ.` bezpośrednio steruje preferencją
  `vehicle_cascade_enabled`, nie usuwa modelu i po powrocie odtwarza silnik
  pipeline'u przez istniejący mechanizm rewizji ustawień;
- obowiązkowe etapy MT i MZ pokazują `WŁ.` albo `BRAK`, ale nie można ich
  wyłączyć, ponieważ aplikacja nie ma wykonywalnego pipeline'u bez detektora
  tablic i modelu znaków;
- usunięto osobny przełącznik oraz powieloną akcję `Wyłącz model MP`; całkowite
  usunięcie MP z kompozycji nadal jest dostępne przez wybór modelu `Brak`.

Weryfikacja:
- kontrola wizualna na SM-A125F potwierdziła czytelny układ trzech węzłów,
  badge'y bez kolizji z opisem pakietu i brak starego przełącznika;
- ręczny test MP `WYŁ. → WŁ. → WYŁ.` potwierdził zmianę preferencji i wzrost
  `settings_revision` przy każdym przełączeniu;
- `assembleDebug`, `lintDebug`, `testDebugUnitTest` oraz pełne 14 testów
  instrumentalnych na SM-A125F zakończyły się powodzeniem;
- po regresji ponownie zainstalowano APK, przywrócono pakiet MP+MT+MZ i wykonano
  autotuning wszystkich trzech modeli.

Pozostało:
- ewentualne wyłączanie MT lub MZ wymagałoby zaprojektowania nowych, jawnych
  trybów pracy (`MP only` albo `MT bez OCR`); nie jest to obecnie ukryte pod
  nieaktywnymi badge'ami.

## 9. Zasady aktualizowania dziennika

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
emulatorze, lokalnym JVM, parserze desktopowym i fizycznym telefonie powinny
być rozróżniane.

## 10. Dokumenty powiązane

- `docs/model_package_v1.md` — format pojedynczego i kompletnego pakietu;
- `docs/alpr-model-v1.schema.json` — schemat pojedynczego modelu;
- `docs/alpr-package-v1.schema.json` — schemat kompletnej paczki MT+MZ albo
  MP+MT+MZ;
- `docs/mobile_architecture.md` — potok wykonawczy i raportowanie;
- `docs/mobile_research_export.md` — kontrakt pełnego eksportu i TeX;
- `docs/alpr-mobile-research-bundle-v1.schema.json` — schemat manifestu
  eksportu badawczego;
- `docs/model_package_test_strategy.md` — strategia fixture, parytetu,
  runtime'ów i bramek akceptacyjnych paczki modeli;
- `C:\Users\48572\Desktop\dyplom\start4\alpr_python_exporter_handoff.md` —
  kontrakt Python → Android;
- `C:\Users\48572\Desktop\dyplom\start4\docs\specyfikacja_agenta_aplikacji_mobilnej_alpr.md`
  — wymagania klienta mobilnego;
- `C:\Users\48572\Desktop\dyplom\start4\docs\siatka_eksperymentow_mobilnych_alpr.md`
  — metodyka eksperymentów i rankingu.
