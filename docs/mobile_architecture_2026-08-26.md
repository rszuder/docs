# Architektura klienta mobilnego ALPR

Stan dokumentu: 2026-08-26.

## 1. Zakres

Aplikacja Android jest jednocześnie demonstratorem mobilnego ALPR i stanowiskiem pomiarowym.
Nie trenuje modeli. Otrzymuje gotowe pakiety `.alprmodel`, waliduje je, wybiera wykonywalny
wariant runtime i uruchamia kaskadę na urządzeniu.

Główne role modeli:

- **MP** — opcjonalny model pojazdów, używany do wyznaczania ROI;
- **MT** — YOLO Pose wykrywający tablicę i jej cztery narożniki;
- **MZ** — YOLO Character Detection wykrywający poszczególne znaki. Nie jest klasycznym OCR-em.

## 2. Potok wykonawczy

```text
CameraX / ImageAnalysis
  -> YUV_420_888
  -> rotacja wyjścia wykonywana przez CameraX
  -> ImageProxy.toBitmap()
  -> adaptacyjna redukcja klatek
  -> wybór polityki R0 / R1 / R2
      R0: MT na pełnej klatce
      R1: MP -> maks. 1 ROI -> MT
      R2: MP -> maks. 2 ROI -> MT
  -> YOLO Pose MT
  -> NMS i deduplikacja detekcji
  -> walidacja czterech narożników
  -> przypisanie do tracku
  -> decyzja schedulera MZ
  -> rektyfikacja perspektywy
  -> MZ
  -> porządkowanie znaków i wierszy
  -> temporalny konsensus znaków
  -> wynik wstępny / stabilny
  -> overlay, galeria i raport
```

`ImageAnalysis` używa `STRATEGY_KEEP_ONLY_LATEST`. Rotacja obrazu jest włączona przez
`setOutputImageRotationEnabled(true)`, dzięki czemu typowy przebieg na urządzeniu nie
wykonuje kosztownej ręcznej rotacji pełnej bitmapy. `CameraImageConverter` pozostawia
ścieżkę awaryjną dla urządzenia, które mimo to dostarczy niezerowy obrót.

## 3. Polityki ROI i tryb eksperymentalny

Warstwa eksperymentalna jest oddzielona od normalnej konfiguracji użytkownika.

- **R0 / `r0_full_frame`** — MP jest pomijany, MT analizuje pełną klatkę;
- **R1 / `r1_one_roi`** — MP może przekazać maksymalnie jeden ROI do MT;
- **R2 / `r2_two_roi`** — MP może przekazać maksymalnie dwa ROI do MT.

ROI nie zmniejsza rozmiaru wejścia modelu MT. Każdy wycinek jest nadal przygotowywany
do stałego wejścia MT, dlatego dwa ROI oznaczają dwa pełne przebiegi preprocessingu
i inferencji MT.

Okresowy full-frame fallback działa dla zwykłej kaskady ROI, ale jest wyłączony
podczas aktywnej blokady celu auto-zoomu.

MP jest odświeżany okresowo, a przy szybkim ruchu kamery częściej. Margines ROI rośnie
podczas szybkiego ruchu. `VehicleRoiSelector` odpowiada za wybór regionów i ograniczenie
ich liczby do budżetu R1/R2.

## 4. Tracking i świeżość sceny

Aplikacja ma dwa poziomy trackingu:

1. `PlateTrackCoordinator` i `MotionBoxTracker` wewnątrz pipeline'u — utrzymują tracki
   tablic, stan MZ i temporalny konsensus;
2. `PreviewPlateTracker` w UI — lekki tracker klatek `PreviewView`, którego celem jest
   płynne przesuwanie overlayu pomiędzy kosztownymi inferencjami MT.

`SceneChangeDetector` i `SceneAnchorGuard` odrzucają stare wyniki po rzeczywistej zmianie
kadru. UI używa generacji sceny, dzięki czemu wynik rozpoczęty dla poprzedniej sceny
nie może ponownie pojawić się nad nowym obrazem.

`PreviewTrackerDriftGuard` ogranicza dryf lekkiego trackera. Zestawia ruch klatka-do-klatki
z niezależnym pomiarem względem ostatniej kotwicy MT i odrzuca aktualizację, jeśli wsparcie
punktów jest zbyt małe, skok jest zbyt duży albo narasta nadmierny offset.

Ramka tablicy jest rysowana z czterech keypointów MT, a nie tylko z prostokątnego boxa.
Interpolacja keypointów wygładza zmianę geometrii między świeżymi wynikami MT.

## 5. Scheduler MZ i konsensus temporalny

Pierwsza poprawna geometria tracku otrzymuje próbę MZ. Następne próby zależą od profilu
rozpoznawania, poprawy jakości i odstępu pomiędzy próbami.

Stabilny wynik nie kończy już bezwzględnie działania MZ. Po wykorzystaniu początkowego
burstu scheduler okresowo ponawia rozpoznanie, aby dwa zgodne, ale błędne odczyty
nie zamroziły tracku na cały czas jego życia.

`TemporalCharacterAggregator` rozdziela stan według struktury wierszy, np. `[7]` i `[3,5]`
są osobnymi hipotezami. Dla każdej pozycji znak wybierany jest na podstawie liczby głosów,
a przy remisie sumy confidence.

Wynik jest stabilny, gdy dominująca struktura ma co najmniej dwie obserwacje i zwycięski
znak na każdej pozycji ma co najmniej dwa głosy.

Pole `confirmed` używane obecnie w `PlateObservation` i `CapturedPlateItem` oznacza
stabilność wyniku tracku, a nie ręczną poprawność konkretnego cropa. Ręczna walidacja
`ACCEPTED/CORRECTED/REJECTED` pozostaje osobnym mechanizmem ground truth.

## 6. Auto-zoom

Auto-zoom jest opcjonalną warstwą działającą ponad zwykłym pipeline'em.
Nie jest częścią bazowego eksperymentu R0/R1/R2.

`AutoZoomController` ma stany:

```text
DISABLED -> READY -> ZOOM_SETTLING -> ZOOMED_RETRY -> RETURNING
```

Kandydat do zbliżenia musi mieć co najmniej dwie obserwacje, poprawny quad i być mały
albo mieć niski confidence. Domyślny żądany zoom wynosi `1.8x`.

Po zoom-in kontroler czeka na świeżą próbę MZ. Powrót może nastąpić po potwierdzonej
poprawie, silnej poprawie confidence, dwóch spójnych umiarkowanych poprawach albo timeout.

`CameraController` animuje `zoomRatio` i po zbliżeniu ustawia region AF/AE/AWB w pobliżu
celu. Kamera może nie obsługiwać pełnego zestawu regionów pomiarowych; brak tej funkcji
nie blokuje samego zoomu.

### 6.1. Tożsamość celu

`AutoZoomTargetLock` utrzymuje dokładnie tę tablicę, która wywołała zoom.

```text
DISABLED -> ACQUIRING -> LOCKED -> UNCERTAIN -> LOST
```

Dopasowanie uwzględnia:

- przewidywane położenie;
- IoU;
- zgodność skali i proporcji;
- confidence MT;
- poprawność geometrii;
- prosty deskryptor wyglądu z `PlateAppearanceDescriptor`.

Aktywna blokada wyznacza własny obszar `ROI AUTO ZOOM`. MT analizuje ten obszar,
a `AutoZoomTargetLock` wybiera tylko kandydata zgodnego z zablokowanym celem.
Inne tablice nie powinny przejmować ponownej próby MZ.

### 6.2. Zoom jako transformacja tej samej sceny

Kontrolowany zoom ma osobną generację transformacji kamery. Wynik rozpoczęty dla
poprzedniej skali może zostać odrzucony bez zerowania całej logicznej sceny.

Podczas transformacji zwykłe wykrywanie zmiany sceny jest maskowane. Po zakończeniu
zoomu geometria tracków jest przeliczana przez względny współczynnik skali,
a referencja detektora sceny jest ustawiana ponownie.

Po powrocie do `1x` obraz może zostać porównany z kotwicą sprzed zoomu, aby rozróżnić
powrót do tej samej sceny od rzeczywistej zmiany kadru.

### 6.3. Pamięć tekstu

`AutoZoomRecognitionMemory` rozdziela geometrię celu od najlepszego tekstu.
Pusty albo słabszy świeży wynik nie usuwa poprzedniego wiarygodnego numeru.
Nowy tekst może zastąpić pamięć tylko po przejściu konserwatywnej bramki jakości.

Dzięki temu świeże MT może poprawić geometrię ramki bez kasowania numeru, a pierwszy
sensowny MZ po zoom-in może zainicjalizować tekst także wtedy, gdy przed zbliżeniem
MT widziało tablicę bez odczytu znaków.

## 7. Kamera, rozdzielczość i ruch

Lista rozdzielczości nie jest już ograniczona do trzech statycznych profili.
`CameraResolutionCatalog` odczytuje formaty obsługiwane przez urządzenie,
a `CameraResolutionSelection` zapisuje tryb `auto` albo konkretną rozdzielczość.

Raport rozdziela rozdzielczość wybraną, żądaną i faktycznie otrzymaną.

Dla formatów high-resolution CameraX może preferować większą rozdzielczość kosztem
częstotliwości. Pole raportu `extended_high_resolution_mode_requested` oznacza użycie
specjalnej puli high-resolution, a nie po prostu dużą liczbę pikseli.

`CameraMotionMonitor` i `MotionIntensityFilter` informują pipeline o szybkim ruchu
urządzenia. Ruch może wymusić odświeżenie MP, rozszerzyć ROI lub przerwać
niebezpieczny etap auto-zoomu.

## 8. Runtime'y modeli

Obsługiwane wykonanie:

- LiteRT/TFLite — CPU oraz delegat GPU, zależnie od modelu i urządzenia;
- ONNX Runtime Android — CPU;
- NCNN — natywny runtime CPU przez JNI `libalpr_ncnn.so` dla ARM.

NCNN nie jest już tylko formatem przechowywanym. Backend jest wykonywalny i bierze udział
w aktywacji pakietu oraz autotuningu. Vulkan pozostaje wyłączony do czasu osobnego
eksperymentu.

Kompletny pakiet może zawierać różne runtime'y i precyzje dla MP, MT i MZ.
Ręcznie przypięty wariant ma pierwszeństwo przed autotuningiem.
Automatyczna promocja INT8 nadal wymaga osobnej bramki jakościowej.

## 9. Główny ekran

`MainActivity` koordynuje kamerę, overlay, eksperyment, galerię i auto-zoom.

Najważniejsze elementy:

- jeden przycisk `Uruchom analizę / Zatrzymaj analizę`;
- Bottom Sheet galerii cropów;
- skrócony HUD telemetryczny;
- timer eksperymentu;
- monitor i warunek termiczny;
- kontrolka auto-zoomu widoczna podczas analizy;
- bezpośredni eksport `.alprsession` po zakończeniu eksperymentu.

Tryb immersyjny ukrywa systemowe paski na głównym ekranie.

## 10. Warunek termiczny eksperymentu

`ThermalConfig` opisuje warunek START, a `TimerConfig` warunek STOP.

Po naciśnięciu `Uruchom analizę` kamera nie musi startować natychmiast. Jeżeli warunek
termiczny jest aktywny, aplikacja czeka bez uruchamiania kamery, aż temperatura baterii
i Android Thermal Status spełnią ustawiony próg przez wymagany czas stabilizacji.

`ThermalMonitor` raportuje:

- `BAT` — temperaturę baterii w °C;
- `TH` — dyskretny Android Thermal Status;
- `HEAD` — bezwymiarowy Thermal Headroom, jeśli urządzenie go udostępnia.

HEAD jest próbkowany nie częściej niż co 10 s i obecnie jest wskaźnikiem obserwacyjnym.
Nie jest używany jako automatyczny próg startu.

Timer zaczyna odliczanie dopiero po faktycznym rozpoczęciu `ExperimentSession`
i `MetricsCollector`. Konfiguracja timera pozostaje ustawiona po zakończeniu przebiegu.

## 11. HUD i bilans czasu

Najważniejsze metryki:

- `PIPE` — całkowity czas mierzonej iteracji;
- `INF` — suma inferencji MP+MT+MZ;
- `AUX` — jawnie zmierzone operacje poza inferencją;
- `OVH` — `PIPE - (INF + AUX)`;
- `DROP` — liczba odrzuconych klatek.

Trace zawiera również m.in. `CAM`, `camera_to_bitmap`, `camera_rotation`, PRE/INF/POST
dla modeli, rektyfikację, liczbę ROI, fallbacki oraz liczniki i confidence blokady auto-zoomu.

## 12. Cropy i galeria

Galeria działa jako Bottom Sheet. `CropSamplingPolicy` nie zapisuje każdej klatki.
Nowy crop może zostać przyjęty m.in. przy:

- pierwszej obserwacji tracku;
- zmianie tekstu;
- przejściu do stabilnego wyniku;
- poprawie recognition confidence o co najmniej `0.10`;
- poprawie ostrości o co najmniej `0.08`;
- upływie interwału okresowego około `1.5 s`.

`CapturedPlateItem` przechowuje m.in. tekst, confidence MT/MZ, `confirmed`, sharpness,
czasy, `camera_zoom_ratio` i `capture_source`. Crop z ponownej próby po zbliżeniu
może mieć `capture_source=auto_zoom_retry`.

Ręczna walidacja jest niezależna od zaznaczenia do zapisu i od stabilności tracku.

## 13. Diagnostyka i ustawienia

`SettingsActivity` obsługuje profil rozpoznawania, dynamiczny wybór rozdzielczości,
limit cropów, eksperyment R0/R1/R2, kompozycję modeli, warianty runtime i katalog zapisu.

`DiagnosticsActivity` pokazuje:

- stan urządzenia i pamięci;
- stan pipeline'u i sesji;
- tabelę MP/MT/MZ z nazwą modelu, runtime'em, precyzją, profilem wykonania,
  liczbą wątków, wejściem i progami;
- trwały log podzielony na zdarzenia;
- grupowanie kolejnych eksperymentów R0/R1/R2;
- dostęp do pełnych opcji eksportu badawczego.

## 14. Raportowanie

`MetricsCollector` zapisuje ślady tylko w aktywnej sesji pomiarowej.
`ExperimentSession` zamraża typ eksperymentu, wariant, czas startu/końca,
powód zakończenia i konfigurację timera.

`report.json` zawiera m.in.:

- identyfikację pakietu i wariantów modeli;
- runtime, precyzję, delegata i liczbę wątków per MP/MT/MZ;
- konfigurację normalną i eksperymentalną;
- wybraną, żądaną i faktyczną rozdzielczość;
- dropped frames;
- percentyle etapów;
- surowe trace'y i liczniki;
- czas do pierwszego wyniku wstępnego i stabilnego;
- rekordy cropów oraz ręczną walidację.

Metadane cropów zawierają `camera_zoom_ratio` i `capture_source`.
Trace może zawierać m.in. `auto_zoom_target_roi`, `auto_zoom_lock_candidates`,
`auto_zoom_lock_misses`, `auto_zoom_lock_score` i `auto_zoom_lock_confidence`.

Bieżący raport nie ma jeszcze pełnego, strukturalnego snapshotu termicznego START/STOP
ani agregatu całego cyklu auto-zoomu. Tych elementów nie należy opisywać jako wdrożonych.

## 15. Weryfikacja i ograniczenia

Po integracji auto-zoomu dodano testy jednostkowe kontrolera zoomu, transformacji
overlayu, blokady celu, pamięci odczytu i ograniczenia dryfu trackera.
`testDebugUnitTest` i `lintDebug` zakończyły się powodzeniem.

Najnowsza korekta pamięci numeru i pełny cykl kamera -> zoom-in -> świeże MT/MZ
-> zoom-out -> `1x` wymagają jeszcze końcowej weryfikacji wizualnej na urządzeniu.
Zielone testy JVM i lint potwierdzają logikę i integrację kompilacyjną,
ale nie zastępują próby kamera–monitor.
