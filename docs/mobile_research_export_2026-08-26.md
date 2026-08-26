# Eksport badawczy klienta mobilnego ALPR

Stan dokumentu: 2026-08-26.

## 1. Artefakty

Aplikacja udostępnia trzy formaty eksportu:

- `*.alprsession` — pełny `alpr.mobile_research_bundle.v1`;
- `alpr_thesis_*.zip` — `alpr.mobile_thesis_bundle.v1`;
- klasyczny ZIP raportu `alpr.mobile_benchmark_report.v1` dla zgodności wstecznej.

Pełne opcje pozostają w Diagnostyce. Po zakończeniu eksperymentu przycisk na ekranie
głównym eksportuje bezpośrednio pełną sesję `.alprsession`.

Eksport jest strumieniowy. Wagi modeli i JPEG-i nie są agregowane w jednym dużym `byte[]`.
Na czas zapisu snapshot cropów jest chroniony przed wyparciem.

## 2. Zamrożenie sesji eksperymentalnej

Przed uruchomieniem wątku eksportującego `MainActivity` pobiera
`ExperimentSession.Snapshot`.

Dzięki temu zmiana ustawień R0/R1/R2 po zakończeniu przebiegu, ale przed eksportem,
nie może zmienić opisu już wykonanej sesji.

Snapshot zawiera:

- identyfikator sesji;
- stan;
- typ eksperymentu;
- wariant;
- czas startu i końca;
- czas trwania;
- powód zakończenia;
- informację, czy timer był aktywny;
- skonfigurowany czas timera.

## 3. Konfiguracja i rozdzielczość

`report.json/capture` rozdziela:

- `selection_mode` — `auto` lub `explicit`;
- `selected_resolution` — ustawienie użytkownika;
- `requested_width` / `requested_height`;
- `actual_width` / `actual_height`;
- `actual_resolution`;
- `requested_resolution_matched`;
- `extended_high_resolution_mode_requested`;
- `pixel_format = YUV_420_888`;
- dostępność żyroskopu.

`extended_high_resolution_mode_requested` nie oznacza po prostu dużej liczby pikseli.
Informuje, że CameraX został uruchomiony w trybie dopuszczającym specjalną pulę
high-resolution.

## 4. Czas i ślady pipeline'u

Czas mierzony jest zegarem monotonicznym.

Najważniejsze grupy:

- `PIPE` / `total` — pełny czas iteracji;
- `INF` / `inference_sum` — suma inferencji MP, MT i MZ;
- `AUX` / `auxiliary_sum` — jawnie zmierzone operacje poza inferencją;
- `OVH` / `pipeline_overhead` — różnica `PIPE - (INF + AUX)`.

Trace zawiera także:

- `camera_conversion`;
- `camera_to_bitmap`;
- `camera_rotation`;
- PRE/INF/POST per model;
- `rectification`;
- liczbę uruchomień MT na ROI i pełnej klatce;
- full-frame fallbacki;
- liczniki MZ;
- zmiany sceny;
- liczniki i confidence blokady auto-zoomu, jeśli była aktywna.

Brak etapu w trace oznacza brak wykonania. Nie wolno zastępować brakującej inferencji MZ
wartością `0 ms`, ponieważ zera zafałszowałyby percentyle.

## 5. Ground truth i znaczenie `confirmed`

Każdy crop ma niezależny stan ręcznej walidacji:

- `not_reviewed`;
- `accepted`;
- `rejected`;
- `corrected`.

CER i exact match są liczone tylko dla rekordów, dla których istnieje poprawny ground truth.

Pole `confirmed` w bieżącym modelu danych nie oznacza ręcznej poprawności cropa.
Jest kopią stanu `stable` temporalnego konsensusu MZ dla całego tracku.

Dlatego możliwy jest crop:

```text
confirmed = true
fresh MZ = brak
text = wynik z pamięci tracku
```

Takiego rekordu nie wolno interpretować jako „ten crop samodzielnie potwierdził numer”.

Docelowo kontrakt powinien rozdzielić:

```text
trackConfirmed
freshMzSuccessful
cropSupportsConsensus
manualVerificationStatus
```

## 6. Metadane cropów

`CapturedPlateItem`, `MetricsCollector` i `CropMiniReport` zapisują m.in.:

- `capture_id`;
- `session_id`;
- `track_id`;
- prediction;
- confidence MT;
- confidence MZ;
- `confirmed`;
- sharpness;
- znaki;
- czasy;
- `camera_zoom_ratio`;
- `capture_source`;
- `human_verification`.

`capture_source` rozróżnia co najmniej:

- `normal`;
- `auto_zoom_retry`.

`camera_zoom_ratio` pozwala zestawić próbki wykonane przy normalnym widoku
z próbkami powstałymi po zbliżeniu.

## 7. Auto-zoom w trace i raporcie

Aktywna blokada celu może raportować m.in.:

- `auto_zoom_target_roi`;
- `auto_zoom_target_roi_area_ratio`;
- `auto_zoom_lock_candidates`;
- `auto_zoom_lock_misses`;
- `auto_zoom_lock_score`;
- `auto_zoom_lock_second_score`;
- `auto_zoom_lock_confidence`.

Są to dane diagnostyczne konkretnej iteracji.

Bieżący raport nie posiada jeszcze jednego, zagregowanego obiektu sesji auto-zoom
z polami typu liczba prób, przyczyna zoom-in, przyczyna powrotu, confidence przed/po
i wynik końcowy. Taka agregacja pozostaje zadaniem do wykonania przed osobnym
eksperymentem `auto-zoom OFF vs ON`.

## 8. Warunek termiczny

`ThermalMonitor` odczytuje:

- BAT — temperaturę baterii w °C;
- TH — Android Thermal Status;
- HEAD — Thermal Headroom, jeśli urządzenie go udostępnia.

HEAD jest bezwymiarowy i próbkowany nie częściej niż co 10 s.

Warunek termiczny może blokować START eksperymentu przed uruchomieniem kamery.
Timer rozpoczyna odliczanie dopiero po faktycznym rozpoczęciu sesji.

Ważne ograniczenie bieżącego kontraktu:

- BAT/TH/HEAD są dostępne w UI i logu;
- `DeviceProfile.capture(...)` daje snapshot urządzenia przy eksporcie;
- raport nie przechowuje jeszcze pełnego, strukturalnego zestawu
  `thermal_start` i `thermal_end` należącego do zamrożonej `ExperimentSession`.

Dlatego dokumentacja wyników nie powinna twierdzić, że strukturalny termiczny
snapshot START/STOP jest już częścią `.alprsession`.

## 9. Modele

`report.json/execution` wiąże osobno MP, MT i MZ z:

- `model_id`, nazwą, wersją i fingerprintem;
- wariantem, runtime'em i precyzją;
- delegatem i liczbą wątków;
- efektywnym wejściem;
- dekoderem i progami;
- metadanymi artefaktu.

Kompletny pakiet może zawierać różne runtime'y dla poszczególnych etapów,
np. MP/NCNN/FP32 oraz MT/MZ w TFLite.

## 10. TeX

`summary.tex` korzysta ze względnych `\input{}` dla tabel.
Generator powinien escapować znaki specjalne TeX.

Paczka pracy zawiera konfigurację modeli, metryki jakości dostępne z ground truth,
percentyle czasu, wybrane cropy oraz dane śladów do własnych wykresów.

## 11. Metodyka

`protocol.json` opisuje założenia benchmarku inspirowanego MLPerf Mobile.
Pole `mlperf_compliant` pozostaje `false`, ponieważ aplikacja nie korzysta
z oficjalnego LoadGena.

Należy rozdzielać:

- kontrolowany replay — porównanie runtime'ów i modeli na stałym materiale;
- sesję live / camera-in-the-loop — pełny system z kamerą, trackingiem,
  schedulerem, ROI, termiką i opcjonalnym auto-zoomem.

Wyników tych dwóch scenariuszy nie należy łączyć w jedną średnią.

## 12. Otwarte rozszerzenia kontraktu

Do wdrożenia pozostają:

1. `thermal_start` / `thermal_end` z BAT, TH i HEAD;
2. zagregowany raport całego cyklu auto-zoom;
3. rozdzielenie stabilności tracku od wsparcia konkretnego cropa;
4. importer `.alprsession` po stronie programu macierzystego;
5. kontrolowany replay do końcowego porównania runtime'ów i modeli.
