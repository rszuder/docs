# Telemetria, kontrakt danych i eksport kampanii eksperymentalnej mobilnego ALPR

Stan źródeł: 26.08.2026 • Dokument roboczy do przekazania agentowi implementacyjnemu

| **Zakres:** Dokument dotyczy repozytorium rszuder/alpr_mobile_client (main) oraz sposobu rozszerzenia istniejącego raportu bez niszczenia zgodności z bieżącym importerem desktopowym. |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

## 1. Cel agenta

- **Nie przebudowywać pipeline’u dla samego raportowania.** W pierwszej kolejności wykorzystać telemetrię, którą kod już tworzy.

- **Dostarczyć kompletne dane surowe.** Android jest źródłem pomiaru; Desktop ma wykonywać większość analiz, agregacji i wykresów.

- **Minimalizować narzut.** Nowe pomiary nie mogą istotnie zmieniać czasu pipeline’u, który mają mierzyć.

- **Zachować kompatybilność.** Istniejące report.json, traces.csv, .alprsession i alpr.mobile_benchmark_report.v1 pozostają obsługiwane.

## 2. Stan aktualny – czego NIE implementować drugi raz

Aktualny klient już raportuje rozbudowaną telemetrię. Należy ją zachować, ujednolicić i udostępnić Desktopowi, zamiast tworzyć równoległe liczniki.

| **Obszar**          | **Już istnieje w Androidzie**                                                                                   | **Uwagi**                                                               |
|---------------------|-----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------|
| Timing per frame    | total, camera_conversion, MP/MT/MZ preprocess/inference/postprocess, rectification                              | InferenceTrace + MetricsCollector; summary p50/p90/p95/p99.             |
| Timing audit        | engine_setup, engine_total, pipeline_finalize, inference_sum, auxiliary_sum, pipeline_overhead, engine_overhead | Część pól jest w report.json/traces, ale nie wszystkie są w traces.csv. |
| Jakość wejścia      | plate_fit, plate_sharpness, characters_min/mean, confidence MP/MT/MZ                                            | Zachować surowe wartości.                                               |
| ROI / scheduler     | vehicle_roi_area_ratio, mz_runs/skipped, vehicle_runs/skipped, plate_roi/full-frame runs, fallbacki             | Podstawa eksperymentów R0–R4.                                           |
| Scena               | scene_change_score/fraction/brightness_delta, scene_reset, camera_transform_in_progress                         | Już zbierane w trace; obecny CSV nie pokazuje wszystkiego.              |
| Autozoom / lock     | target ROI, lock candidates/misses/score/second score/confidence; zoom ratio i capture_source dla cropa         | Dane istnieją, ale brakuje spójnego event streamu.                      |
| Pamięć              | PSS i native heap co 30. trace + ram_peak_mb                                                                    | Nie zwiększać częstotliwości bez benchmarku narzutu.                    |
| GT / jakość końcowa | accepted/corrected, exact match, CER, edit distance per unique session+track                                    | Confidence nie może zastępować GT.                                      |
| Termika             | ThermalMonitor: temperatura baterii, thermalStatus, thermalHeadroom; DeviceProfile: pojedynczy snapshot         | Brakuje szeregu czasowego powiązanego z eksperymentem.                  |

## Wspólny kontrakt Android ↔ Desktop

**Cel kontraktu.** Zapewnić powtarzalny łańcuch: konfiguracja eksperymentu → pomiar na telefonie → eksport surowych danych → import bez utraty semantyki → analiza porównawcza → wykresy/tabele do pracy inżynierskiej.

| **Zasada**               | **Wymaganie wspólne**                                                                                                                                  |
|--------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| Surowe dane są nadrzędne | Android może generować percentyle i summary, ale Desktop musi mieć możliwość ponownego obliczenia metryk z surowych rekordów.                          |
| Brak ≠ zero              | Etap niewykonany ma być null/pusty/brak klucza; 0 oznacza rzeczywiście zmierzony zerowy licznik/czas. Importer nie może wypełniać braków zerami.       |
| Jednostki w nazwach      | Stosować jawne sufiksy: \_ms, \_ns, \_px, \_ratio, \_c, \_bytes, \_kb. Nie używać wartości procentowych bez określenia, czy 0–1 czy 0–100.             |
| Czas monotoniczny        | Do synchronizacji serii stosować elapsed_ms / elapsed_ns. Czas UTC służy do identyfikacji i prezentacji, nie do mierzenia latency.                     |
| Identyfikacja            | Każdy przebieg musi mieć stabilne report_id, experiment.session.id, series_id, scenario_id, variant i replicate_index.                                 |
| Wersjonowanie            | Raport musi identyfikować aplikację, modele i runtime. Porównania nie mogą mieszać niejawnie różnych buildów/modeli.                                   |
| Kompatybilność           | Rozszerzać alpr.mobile_benchmark_report.v1 addytywnie. Stare pola i stare archiwa pozostają czytelne; nowe pola są opcjonalne dla starych raportów.    |
| Ground truth             | Exact match i CER liczyć wyłącznie tam, gdzie istnieje jawny ground truth. Confidence nie jest metryką poprawności.                                    |
| confirmed                | Nie traktować pola confirmed jako ground truth. Docelowo rozdzielić stabilność tracku, świeży sukces MZ, wsparcie cropa dla konsensusu i ocenę ręczną. |
| Nie modyfikować źródeł   | Desktop przechowuje raporty źródłowe jako immutable; wszystkie metryki pochodne, adnotacje analityczne i wykresy mają osobny cache/manifest analizy.   |

### Wspólny model logiczny danych

| **Warstwa**  | **Klucz / ziarnistość**                               | **Przeznaczenie**                                                                            |
|--------------|-------------------------------------------------------|----------------------------------------------------------------------------------------------|
| SESSION      | 1 rekord na raport / przebieg                         | Konfiguracja, wersje, urządzenie, model/runtime, eksperyment, wyniki zbiorcze.               |
| FRAME        | 1 rekord na przetworzoną klatkę                       | Czasy etapów, confidence, ROI, geometria, ruch, status, pamięć.                              |
| FRAME_FLOW   | 1 rekord / sekunda lub lekki rekord wejściowej klatki | Received/processed/skipped, przepustowość, estymacja upstream gaps i frame age.              |
| TRACK / CROP | 1 rekord na zapisany crop / jednostkę jakościową      | GT, exact/CER, rozmiar/geometria tablicy, jakość obrazu, zoom, wynik.                        |
| THERMAL      | 1 rekord ~1 Hz                                        | Temperatura, thermal status/headroom, bateria; korelacja z wydajnością.                      |
| EVENT        | rekord tylko przy zdarzeniu                           | MZ/consensus, autozoom, lock, scene reset, track lifecycle; rekonstrukcja przebiegu systemu. |

> Klucze łączenia:
> SESSION.report_id / experiment.session.id
> FRAME.frame_id
> TRACK/CROP: session_id + track_id (+ capture_id)
> EVENT: experiment_session_id + elapsed_ms + event_seq; opcjonalnie frame_id/track_id
> THERMAL / FRAME_FLOW: experiment_session_id + elapsed_ms
> Łączenie time-series: po elapsed_ms (nearest/backward z jawnie określoną tolerancją).

## 3. P0 – identyfikacja eksperymentu i odtwarzalność

| **Priorytet P0:** Bez stabilnej identyfikacji serii nie wolno rozpoczynać kampanii pomiarowej. Późniejsze ręczne sortowanie plików jest niedopuszczalne. |
|----------------------------------------------------------------------------------------------------------------------------------------------------------|

| **Pole**                   | **Typ / przykład**                                       | **Wymaganie**                                                                      |
|----------------------------|----------------------------------------------------------|------------------------------------------------------------------------------------|
| experiment.series_id       | string, np. ROI-2026-08-A                                | Wspólny identyfikator całej serii porównawczej.                                    |
| experiment.scenario_id     | string, np. single_static / distant_plate / multi_static | Kontrolowana etykieta materiału/warunków.                                          |
| experiment.variant         | R0, R1, R2… / autozoom_on / tflite_fp32                  | Jedyna główna zmienna niezależna w danym porównaniu.                               |
| experiment.replicate_index | int \>=1                                                 | Numer powtórzenia tego samego wariantu/scenariusza.                                |
| experiment.session.id      | istniejący exp-…                                         | Nie zmieniać po zakończeniu przebiegu.                                             |
| app_build.git_commit       | 40-znakowy SHA                                           | Wymagane do porównywalności buildów. app_version=1.0 jest niewystarczające.        |
| app_build.build_type       | debug/release/research                                   | Wymagane, bo profil kompilacji może zmieniać czasy.                                |
| app_build.built_at_utc     | ISO-8601                                                 | Pomocniczo; SHA jest kluczem nadrzędnym.                                           |
| experiment.notes           | opcjonalny string                                        | Notatka operatora, nie może być używana jako jedyny nośnik krytycznych parametrów. |

- Wartości serii/scenariusza/wariantu/replikacji muszą zostać zamrożone w ExperimentSession w chwili startu, tak jak obecnie wariant ROI jest chroniony przed zmianą UI po zakończeniu pomiaru.

- Jeżeli użytkownik zmieni ustawienia po zakończeniu eksperymentu, raport ma nadal opisywać konfigurację z momentu startu.

- Do porównania muszą trafić fingerprinty MP/MT/MZ, runtime, precyzja, liczba wątków, delegate, rozdzielczość oraz recognition profile – większość już istnieje.

## 4. P0 – termika jako szereg czasowy

| **Pole próbki**        | **Źródło**                               | **Częstotliwość / semantyka**                                                                  |
|------------------------|------------------------------------------|------------------------------------------------------------------------------------------------|
| elapsed_ms             | SystemClock.elapsedRealtime()            | Od startu sesji; główna oś czasu.                                                              |
| battery_temperature_c  | ThermalMonitor                           | 1 Hz; NaN/brak jeśli niedostępne.                                                              |
| thermal_status         | PowerManager                             | 1 Hz; wartość liczbowa + opcjonalna etykieta.                                                  |
| thermal_headroom       | ThermalMonitor cache                     | Zapisywać każdą próbkę wraz z flagą available; realny odczyt może być rzadszy (obecnie ~10 s). |
| battery_percent        | BatteryManager                           | 1 Hz lub 5 s; potrzebne przy długich seriach.                                                  |
| charging               | BatteryManager                           | bool; warunki porównania muszą być jawne.                                                      |
| available_memory_bytes | ActivityManager.MemoryInfo – opcjonalnie | 1–5 s; nie dublować kosztownego PSS per frame.                                                 |

> Rekomendowany artefakt: thermal.csv
> experiment_session_id,elapsed_ms,battery_temperature_c,thermal_status,thermal_headroom,headroom_available,battery_percent,charging,available_memory_bytes

- Pojedynczy snapshot DeviceProfile zostaje – jest metadanym startu/eksportu, ale nie zastępuje thermal.csv.

- Próbkowanie termiki ma działać niezależnie od tego, czy dana klatka została przetworzona; inaczej throttling i dropy zniekształcą serię.

- Warunek termiczny startu (np. \<= 32°C) nadal może istnieć, ale raport musi zachować jego konfigurację i faktyczny stan przy starcie.

## 5. P0 – przepływ klatek i poprawna semantyka „drop”

| **Istotne:** Obecne dropped_frames rośnie przy odrzuceniu przez AdaptiveFrameGate albo podczas transformacji kamery. Nie jest to liczba wszystkich klatek utraconych przez CameraX STRATEGY_KEEP_ONLY_LATEST. |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

| **Metryka**                     | **Znaczenie**                                        | **Jak zbierać**                                                                                                                             |
|---------------------------------|------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| frames_received                 | Liczba wejść do analizatora / otrzymanych ImageProxy | Inkrement przed gate i przed camera-transform skip.                                                                                         |
| frames_processed                | Liczba klatek z pełnym InferenceTrace                | Powinna odpowiadać measured_runs/processed_frames.                                                                                          |
| frames_skipped_gate             | Świadome pominięcia AdaptiveFrameGate                | Osobny licznik.                                                                                                                             |
| frames_skipped_camera_transform | Pominięcia podczas kontrolowanej transformacji/zoomu | Osobny licznik.                                                                                                                             |
| estimated_upstream_gaps         | Szacowane braki przed analyzerem                     | Wyłącznie jako ESTYMACJA z ImageInfo.timestamp i oczekiwanej kadencji; nigdy nie nazywać CameraX dropped_frames bez bezpośredniego pomiaru. |
| received_fps / processed_fps    | Przepustowość                                        | Liczyć w Desktopie z szeregu/bucketów; Android może podać summary.                                                                          |
| frame_age_ms                    | Świeżość wyniku względem znacznika akwizycji         | Jeżeli timestamp kamery da się spójnie odnieść do monotonicznego zegara; jeśli nie – raportować „unavailable”, nie zgadywać.                |

> Preferowany lekki artefakt: frame_flow.csv (bucket 1 s)
> experiment_session_id,elapsed_ms,frames_received,frames_processed,frames_skipped_gate,frames_skipped_camera_transform,estimated_upstream_gaps
> Alternatywa: event dla każdej wejściowej klatki tylko w trybie badawczym, jeśli narzut i rozmiar pozostają akceptowalne.

## 6. P0 – rozmiar i geometria tablicy

Celem jest możliwość odpowiedzi: „przy jakim względnym/pikselowym rozmiarze i jakiej perspektywie system zaczyna tracić exact match?”.

| **Pole**                               | **Ziarnistość**                  | **Uwagi**                                                                         |
|----------------------------------------|----------------------------------|-----------------------------------------------------------------------------------|
| plate_bbox_width_px / height_px        | każda wybrana detekcja MT / crop | Wymagane; rozmiar obiektu w źródle.                                               |
| plate_bbox_area_ratio                  | detekcja/crop                    | bbox_area / frame_area.                                                           |
| plate_quad_area_ratio                  | detekcja/crop                    | Pole czworokąta / frame_area; lepsze dla perspektywy.                             |
| plate_corners_norm                     | 4 × (x,y) w 0–1                  | Preferowane surowe dane. Desktop może wyliczać nachylenie/perspective metrics.    |
| plate_fit                              | już istnieje                     | Zachować jako wynik algorytmu Androida, ale nie zastępować nim surowej geometrii. |
| plate_sharpness                        | już istnieje                     | Zachować.                                                                         |
| mean_luminance                         | rektyfikowany crop lub plate ROI | P1; tani wskaźnik jasności.                                                       |
| luminance_stddev                       | rektyfikowany crop               | P1; prosty kontrast.                                                              |
| underexposed_ratio / overexposed_ratio | rektyfikowany crop               | P1; udział pikseli poniżej/powyżej jawnych progów.                                |

- Nie tworzyć ręcznej kategorii „blisko/średnio/daleko” jako jedynego parametru. Zachować wartości ciągłe; Desktop utworzy biny do wykresów.

- Jeżeli dodatkowe wskaźniki obrazu zwiększają czas pipeline’u, liczyć je tylko przy cropie/MZ albo w trybie eksperymentalnym i zmierzyć ich narzut.

## 7. P0 – konsensus czasowy i historia MZ

| **Luka w aktualnym modelu:** TemporalCharacterAggregator i PlateTrackCoordinator znają observations, attempts, layout i rowCounts, lecz CapturedPlateItem nie zachowuje pełnej historii. Z samych wybranych cropów nie da się wiarygodnie odtworzyć correction rate ani false confirmation rate. |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

| **Pole / zdarzenie**                 | **Wymaganie**                                                                |
|--------------------------------------|------------------------------------------------------------------------------|
| track_confirmed                      | Stan stabilności konsensusu tracku; następca obecnego confirmed.             |
| fresh_mz_successful                  | Czy bieżąca próba MZ wykryła i zwróciła użyteczne znaki.                     |
| crop_supports_consensus              | Czy bieżący wynik wspiera tekst wybrany przez konsensus.                     |
| consensus_observations               | Liczba obserwacji dominującej struktury przy danym zdarzeniu/cropie.         |
| mz_attempt_index                     | Kolejny numer próby MZ dla tracku.                                           |
| layout / row_counts                  | single_row/two_row/multi_row + np. \[3,5\].                                  |
| prediction_before / prediction_after | P1; ułatwia audyt zmian konsensusu.                                          |
| manual_verification_status           | Niezależne accepted/rejected/corrected; nigdy nie mieszać z track_confirmed. |

> Rekomendowany events.jsonl (lub events.csv):
> experiment_session_id,event_seq,elapsed_ms,frame_id,track_id,event_type,prediction,confidence,consensus_observations,mz_attempt_index,track_confirmed,fresh_mz_successful,crop_supports_consensus,layout,row_counts,zoom_ratio
> Minimalne event_type:
> track_created, track_lost, mz_attempt, mz_result, consensus_updated, consensus_confirmed

## 8. P1 – autozoom i lock jako eksperyment

| **Element**        | **Wymagane dane**                                                                                                        |
|--------------------|--------------------------------------------------------------------------------------------------------------------------|
| Konfiguracja sesji | auto_zoom_enabled, profil/progi, max zoom; stan zamrożony przy starcie.                                                  |
| Per frame / event  | zoom_ratio, target_track_id, lock_state (acquiring/locked/uncertain/lost), lock_score, second_score, confidence, misses. |
| Zdarzenia          | auto_zoom_started, lock_acquired, lock_uncertain, lock_lost, mz_retry_after_zoom, zoom_finished.                         |
| Wynik              | czy odczyt przed zoomem istniał; czy po zoomie został poprawiony; czas cyklu; GT z tracku/cropa.                         |

- Istniejące pola lock/ROI w InferenceTrace mają pozostać; event stream ma jedynie uporządkować przejścia stanu.

- Target lock success rate powinien być liczony w Desktopie, nie jako nieprzejrzysta liczba w Androidzie.

## 9. Rozszerzenie artefaktu .alprsession

| **Plik**                  | **Status**            | **Rola**                                                                                                                          |
|---------------------------|-----------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| report.json               | zachować              | Summary, konfiguracja, execution/model fingerprints, jakość, liczniki.                                                            |
| traces.csv                | zachować i rozszerzyć | Przenośna tabela per processed frame; dodać obecnie istniejące timing-audit/scene/autozoom fields lub zapewnić równoważne źródło. |
| thermal.csv               | NOWY P0               | Szereg termiczny niezależny od FPS.                                                                                               |
| frame_flow.csv            | NOWY P0               | Received/processed/skipped w bucketach czasu.                                                                                     |
| events.jsonl              | NOWY P0/P1            | Track/MZ/consensus/autozoom/scene events.                                                                                         |
| samples/index.csv         | rozszerzyć            | Plate size/geometry, consensus fields, zoom/source, timing, GT.                                                                   |
| samples/annotations.jsonl | rozszerzyć            | Pełniejsze dane strukturalne znaków/geometrii bez ograniczeń CSV.                                                                 |
| manifest.json             | zachować              | SHA-256 wszystkich nowych artefaktów; manifest ostatnim wpisem.                                                                   |

## 10. Limit danych i długie eksperymenty

| **Ryzyko P0:** MetricsCollector ma obecnie MAX_TRACES = 5000 i usuwa najstarsze trace’y. Dla długiego eksperymentu termicznego może zniknąć początek serii, czyli dokładnie część potrzebna do analizy nagrzewania. |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

- **Nie zwiększać limitu w ciemno.** Najpierw określić oczekiwane długości eksperymentów i zużycie pamięci.

- **Preferowane rozwiązanie.** W trybie badawczym strumieniować trace’y/telemetrię do pliku tymczasowego lub eksportować lekkie bucketowane szeregi, zamiast trzymać wszystko w RAM.

- **Jeżeli zostaje ring buffer.** Raport musi jawnie zawierać trace_total_seen, trace_records_retained, trace_records_evicted i zakres czasowy retained.

- **Zawsze chronić wynik.** Eksport nie może blokować inferencji w trakcie właściwej części eksperymentu; snapshot/stop powinien być jednoznaczny.

## 11. Reguły jakości danych

- Wartości NaN/Infinity nie mogą być serializowane jako liczby JSON; użyć null/braku pola + \*\_available.

- Nie sumować per-frame liczników, które są już kumulacyjne. Każdy licznik musi mieć dokumentację: delta_per_frame albo cumulative_session.

- Czasy etapów nie mogą być sztucznie wpisywane jako 0, gdy etap się nie wykonał (obecna praktyka z brakiem etapu jest poprawna).

- Wprowadzenie nowego pomiaru wymaga mikropomiaru jego narzutu – szczególnie luminancja/kontrast, geometria i zapis na dysk.

- Każda sesja musi zawierać status kompletności: complete / stopped_manual / timer / error / thermal_abort oraz powód.

- Nie zmieniać semantyki istniejących pól bez aliasu/migracji i aktualizacji dokumentacji kontraktu.

## 12. Kolejność implementacji

1.  **P0. Tożsamość eksperymentu i app_build SHA.** series/scenario/replicate + zamrożenie konfiguracji w ExperimentSession.

2.  **P0. Thermal time-series.** thermal.csv + summary start/end/min/max/status transitions.

3.  **P0. Frame flow.** rozbić dropped_frames na jawne przyczyny i dodać frames_received/processed.

4.  **P0. Plate geometry.** wymiary/area ratio/corners w rekordzie cropa lub eventu.

5.  **P0. Recognition events / consensus semantics.** attempts/observations + split confirmed.

6.  **P1. Autozoom state/events.** uporządkować istniejące lock metrics.

7.  **P1. Image difficulty metrics.** luminancja/kontrast/ekspozycja po pomiarze narzutu.

8.  **P0. Long-session storage.** zapobiec niewidocznemu ucinaniu początku eksperymentu.

## 13. Kryteria odbioru Android

| **Test**               | **Kryterium zaliczenia**                                                                                                                                     |
|------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Backward compatibility | Stary importer otwiera nowy raport na tyle, na ile pozwala mu dotychczasowy schemat; stare raporty nadal są czytelne przez nowy Desktop.                     |
| Identity               | Dwa powtórzenia R1 mają ten sam series_id/scenario_id/variant i różne replicate_index/session.id; po zmianie UI po stop dane w raporcie się nie zmieniają.   |
| Thermal                | 10-min test daje ciąg próbek od początku do końca bez zależności od processed FPS.                                                                           |
| Frame flow             | frames_received = processed + app-skips (z uwzględnieniem jasno opisanej granicy pomiaru); upstream gap pozostaje estymacją.                                 |
| Geometry               | Dla każdego jakościowego cropa Desktop może odtworzyć plate height/area i 4 corners lub jasno wykryć brak.                                                   |
| Consensus              | Z eventów można odtworzyć kolejność MZ attempts, pierwszy tekst, zmiany tekstu i moment potwierdzenia.                                                       |
| Long run               | Brak cichego utracenia początku serii; jeśli bufor uciął dane, raport jawnie to deklaruje i kampania zostaje oznaczona jako niepełna do analizy time-series. |
| Overhead               | Narzut nowej telemetrii zmierzony i opisany; brak istotnego pogorszenia pipeline’u lub telemetria możliwa do wyłączenia poza trybem badawczym.               |

## 14. Handoff do agenta Desktop

- Po zmianie kontraktu dostarczyć przykładowy .alprsession zawierający wszystkie nowe pliki i pola oraz osobny przykład z częścią pól niedostępnych.

- Dostarczyć krótką tabelę: pole → typ → jednostka → nullable → źródło → częstotliwość → semantyka.

- Nie wymagać od agenta Desktop znajomości kodu Androida do interpretacji pola.

- Każda zmiana nazwy istniejącego pola musi mieć alias kompatybilności lub jawny numer wersji podschematu.
