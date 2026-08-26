# Audyt repozytorium i dokumentacji — 2026-08-26

Zakres porównania: stan po commicie z warunkiem termicznym 2026-08-25
względem bieżącego `main` z auto-zoomem.

## Skala zmian

Bieżący commit auto-zoomu zmienia kilkadziesiąt plików. Największe obszary:

- `MainActivity` — integracja auto-zoomu, pamięć overlayu, transformacje kamery,
  UI, warunek termiczny, HUD i obsługa cyklu życia;
- `DiagnosticsActivity` — duża przebudowa tabel modeli i logu eksperymentów;
- `CameraController` — płynny zoom, AF/AE/AWB i raportowanie postępu;
- `AlprPipeline` / `MobileAlprEngine` — blokada celu i ROI auto-zoomu;
- nowe klasy `AutoZoomController`, `AutoZoomRecognitionMemory`,
  `AutoZoomTargetLock`, `PlateAppearanceDescriptor`,
  `PreviewTrackerDriftGuard`;
- `CapturedPlateItem` / `CropSamplingPolicy` — metadane zoomu i zapis lepszego cropa;
- `ThermalMonitor` — HEAD;
- nowe testy jednostkowe auto-zoomu i dryfu trackera.

## Najważniejsze zmiany architektoniczne

1. Zoom nie jest już traktowany jako zwykła zmiana sceny.
2. Geometria celu, tożsamość celu i pamięć tekstu są rozdzielone.
3. Po zoom-in wymuszana jest świeża próba MZ na zablokowanym celu.
4. Aktywna blokada celu ogranicza MT do własnego ROI i wyłącza okresowy
   full-frame fallback.
5. Tracker Preview dostał ochronę przed dryfem.
6. Crop zapisuje `camera_zoom_ratio` i `capture_source`.
7. HEAD jest dostępny obserwacyjnie.
8. Diagnostyka pokazuje tabelę runtime'ów i grupuje log eksperymentów.

## Dokumenty nieaktualne przed audytem

### `docs/mobile_architecture.md`

Nieaktualne były m.in.:

- statyczne profile rozdzielczości;
- ręczna rotacja bitmapy jako zwykła ścieżka;
- NCNN opisany jako niewykonywalny;
- twierdzenie, że stabilny wynik kończy MZ;
- stara semantyka galerii/Start-Stop;
- brak EXP R0/R1/R2;
- brak warunku termicznego;
- brak auto-zoomu i blokady celu.

### `docs/mobile_research_export.md`

Brakowało:

- bezpośredniego eksportu sesji z głównego ekranu;
- `camera_zoom_ratio`;
- `capture_source`;
- diagnostyki auto-zoomu;
- jawnego ograniczenia: brak strukturalnego thermal START/STOP;
- doprecyzowania semantyki `confirmed`.

### `docs/model_package_test_strategy.md`

Nieaktualne były:

- NCNN jako „przyszłe JNI”;
- stan pokrycia datowany na 2026-08-20;
- brak scenariusza auto-zoom;
- brak aktualnego stanu R0/R1/R2 i bramki termicznej.

### `DZIENNIK_BUDOWY_APLIKACJI.md`

Repozytoryjny dziennik ma już wpis 2026-08-26 o auto-zoomie, ale jego starsze
sekcje syntetyczne nadal zawierają historyczne stany. Przygotowana wersja audytowa:

- aktualizuje stan technologiczny na 2026-08-26;
- aktualizuje weryfikację;
- uzupełnia HEAD;
- aktualizuje backlog;
- dodaje pełny wpis 2026-08-26;
- utrzymuje słownik skrótów na końcu i dodaje AWB, AUTO-ZOOM i LOCK.

## Pliki bez konieczności zmiany kontraktu

Na podstawie bieżącej zmiany nie ma potrzeby zmieniać:

- `alpr-model-v1.schema.json`;
- `alpr-package-v1.schema.json`;
- `model_package_v1.md`.

Auto-zoom jest funkcją wykonawczą klienta, a nie nowym formatem modelu.

## Otwarte problemy dokumentacyjne / kontraktowe

- raport `.alprsession` nie ma jeszcze strukturalnego `thermal_start/thermal_end`;
- nie ma zagregowanego obiektu całej sesji auto-zoom;
- `confirmed` nadal opisuje stabilność tracku, nie wsparcie bieżącego cropa;
- pełny cykl auto-zoomu po najnowszej korekcie pamięci tekstu wymaga jeszcze
  walidacji na fizycznym urządzeniu.
