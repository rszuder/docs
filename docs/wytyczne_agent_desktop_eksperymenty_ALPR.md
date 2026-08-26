# Importer, model danych, analiza i wizualizacja kampanii eksperymentalnej mobilnego ALPR

Stan źródeł: 26.08.2026 • Dokument roboczy do przekazania agentowi implementacyjnemu

| **Stan repozytorium:** W zdalnym rszuder/auto_annotation_tool importer/przeglądarka mobilna znajduje się na gałęzi checkpoint/stabilizacja-20260714, nie na obecnym main. Przed zmianami agent ma potwierdzić bazę pracy i świadomie przenieść/scalać tę implementację, zamiast rozwijać stary main. |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

## 1. Cel agenta

- **Zbudować warstwę analityczną, nie tylko większy podgląd JSON.** System ma łączyć wiele raportów w serię eksperymentalną i tworzyć porównania nadające się do pracy inżynierskiej.

- **Nie tracić surowych danych.** Importer ma zachować oryginalne archiwum, walidację i pełną semantykę braków.

- **Oddzielić ingest od UI.** Parser/normalizacja/metryki pochodne powinny działać niezależnie od Tkintera i być testowalne.

- **Utrzymać zgodność wsteczną.** Stare report.json / ZIP / .alprsession oraz nowe rozszerzenia Androida muszą współistnieć.

## 2. Stan aktualny repozytorium Python

| **Element**                   | **Stan znaleziony**                                                                                               | **Wniosek**                                                                                                         |
|-------------------------------|-------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| z4_mobile_report_browser.py   | Importuje .alprsession/ZIP, pokazuje listę raportów, p95, jakość, score i zakładki szczegółów.                    | Rozwijać, ale nie doklejać wszystkich analiz bezpośrednio do GUI.                                                   |
| mobile_package_experiments.py | Ma bezpieczny ReportBundleReader, MobileReportBundle, MobileBenchmarkReport, store, walidację archiwum i scoring. | To właściwa baza kontraktu danych.                                                                                  |
| Źródło metryk                 | UI deklaruje report.json dla summary i traces.csv dla wykresów.                                                   | Zmienić: pełny report_payload + nowe artefakty są równorzędnymi źródłami; JSON traces może zawierać więcej niż CSV. |
| Wykres klatkowy               | Tylko pipeline total, MT, MZ, MP z traces.csv.                                                                    | Rozszerzyć modularnie: termika, flow, jakość obrazu, ROI, consensus, autozoom.                                      |
| Limit preview                 | MOBILE_REPORT_TRACE_PREVIEW_ROWS = 5000.                                                                          | Preview nie może być jednocześnie limitem danych używanych do analizy całej sesji.                                  |
| Ranking                       | Score 0.50 jakość / 0.25 latency / 0.15 memory / 0.10 reliability.                                                | Zostawić jako pomoc w wyborze pakietu; nie zastępować nim analizy eksperymentalnej.                                 |

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

## 3. P0 – warstwa normalizacji danych

Najważniejsza zmiana architektoniczna: MobileReportBrowser nie powinien bezpośrednio interpretować dziesiątek kluczy JSON/CSV. Utworzyć neutralny model danych, do którego adaptery mapują raporty stare i nowe.

| **Proponowany typ**     | **Ziarnistość**     | **Minimalne pola**                                                                                                                                                                      |
|-------------------------|---------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| ExperimentSessionRecord | 1 / raport          | report_id, experiment_session_id, series_id, scenario_id, variant, replicate_index, device, app_git_sha, model fingerprints, runtime, resolution, recognition profile, autozoom config. |
| FrameTraceRecord        | 1 / processed frame | frame_id, timestamp/elapsed, status, text, stage_ms.\*, confidence.\*, counters.\*, memory.\*, geometry/scene/lock fields.                                                              |
| FrameFlowBucket         | ~1 / s              | elapsed_ms, received, processed, skipped_gate, skipped_transform, estimated_upstream_gaps.                                                                                              |
| ThermalSample           | ~1 / s              | elapsed_ms, temp_c, thermal_status, headroom, battery%, charging, optional available_memory.                                                                                            |
| RecognitionEvent        | event               | event_seq, elapsed_ms, frame_id, track_id, event_type, prediction, confidence, attempts/observations, stable/support flags, zoom/lock context.                                          |
| TrackCropRecord         | crop / quality unit | session_id, track_id, capture_id, prediction, GT, exact, edit distance/CER contribution, confidence, sharpness, plate geometry, zoom, consensus.                                        |

- Każdy typ ma zachować raw_extra / source_payload albo możliwość dojścia do surowego źródła – nowe pola Androida nie mogą znikać tylko dlatego, że Desktop jeszcze ich nie zna.

- Adapter v1 ma tolerować brak nowych plików i tworzyć rekordy z None/unavailable, nie z 0.

- Parser ma preferować strukturalny report_payload/traces, gdy zawiera pola niewystępujące w traces.csv; CSV pozostaje formatem interoperacyjnym i fallbackiem.

## 4. P0 – poprawna obsługa długich raportów

| **Ryzyko:** Obecny MobileReportBundle przechowuje trace_rows jako preview ograniczony do 5000. To dobre dla UI, ale nie może być podstawą obliczeń dla 10–20 minutowej sesji, jeśli źródło zawiera więcej danych. |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

- **Rozdziel preview i analysis source.** trace_rows_preview może być ograniczony; analiza musi czytać pełny traces.csv/report traces strumieniowo lub z lokalnego cache.

- **Nie ładować wszystkiego bez potrzeby.** Dla dużych plików użyć iteracyjnego csv.DictReader / JSONL; dla wykresu można downsamplować dopiero po obliczeniu metryk.

- **Zachować liczniki kompletności.** trace_total_source, trace_rows_loaded, trace_rows_retained_by_android, evicted_count – jeżeli dostępne.

- **Ostrzegać użytkownika.** Raport z uciętym początkiem serii nie może po cichu zasilać wykresu thermal trend.

## 5. P0 – ingest i walidacja nowych artefaktów

| **Artefakt**      | **Zachowanie Desktopu**                                                                                            |
|-------------------|--------------------------------------------------------------------------------------------------------------------|
| report.json       | Czytać summary i pełne konfiguracje. Zachować raw payload.                                                         |
| traces.csv        | Czytać pełny strumień do warstwy FRAME; nie ograniczać obliczeń do preview.                                        |
| thermal.csv       | Walidować monotoniczny elapsed_ms; normalizować brak headroomu do None.                                            |
| frame_flow.csv    | Sprawdzić spójność sum: received/processed/skips; nie traktować estimated_upstream_gaps jako pewnego CameraX drop. |
| events.jsonl/csv  | Walidować event_seq, session ID, kolejność czasu; tolerować eventy nieznane nowszej wersji Desktopu.               |
| samples/index.csv | Łączyć z annotations.jsonl i report quality po capture_id/session_id/track_id.                                     |
| manifest.json     | Weryfikować SHA-256 nowych plików zgodnie z istniejącą logiką. Nie wyodrębniać archiwum w niebezpieczny sposób.    |

## 6. P0 – indeks raportów i deduplikacja

| **Pole indeksu**                                           | **Po co**                                                                         |
|------------------------------------------------------------|-----------------------------------------------------------------------------------|
| source_archive_sha256                                      | Deduplikacja identycznych importów niezależnie od nazwy pliku.                    |
| report_id / experiment_session_id                          | Tożsamość konkretnego przebiegu.                                                  |
| series_id / scenario_id / variant / replicate_index        | Automatyczne grupowanie porównań.                                                 |
| app_git_sha                                                | Guard przed porównaniem różnych wersji kodu.                                      |
| device identity                                            | Guard urządzenia.                                                                 |
| MP/MT/MZ fingerprint + runtime/precision                   | Guard modeli i środowisk wykonawczych.                                            |
| capture actual resolution / recognition profile / autozoom | Guard konfiguracji pipeline’u.                                                    |
| validation completeness flags                              | Czy raport nadaje się do latency, thermal, quality, consensus, autozoom analysis. |

- MobilePackageExperimentStore może pozostać centralnym store, ale rekord importu powinien przechowywać wersję kontraktu i hash źródła.

- Nigdy nie nadpisywać źródłowego raportu po dodaniu opisów/etykiet w Desktopie. Lokalne adnotacje mają osobny plik/store.

## 7. P0 – Comparison Eligibility Guard

Przed wygenerowaniem wykresu porównawczego Desktop powinien sprawdzić, czy raporty różnią się tylko badaną zmienną albo czy użytkownik jawnie zaakceptował dodatkowe różnice.

| **Guard**               | **Domyślne zachowanie**                                                                        |
|-------------------------|------------------------------------------------------------------------------------------------|
| app_git_sha             | Błąd/ostrzeżenie wysokiego poziomu przy różnicy.                                               |
| device + Android        | Błąd dla eksperymentu mającego porównywać runtime/ROI na jednym urządzeniu.                    |
| model fingerprints      | Muszą być stałe przy porównaniu runtime/ROI; mogą być zmienne tylko w eksperymencie modelowym. |
| resolution/profile      | Muszą być stałe poza eksperymentem badającym rozdzielczość/profil.                             |
| scenario_id / scene set | Muszą odpowiadać temu samemu materiałowi lub porównanie oznaczyć jako niesparowane.            |
| thermal start condition | Sprawdzić, czy starty są porównywalne; różne temperatury startowe flagować.                    |
| GT coverage             | Pokazywać N i coverage; nie porównywać exact/CER bez jawnej liczebności.                       |
| trace completeness      | Raport ucięty/niepełny nie może być użyty do trendu czasowego bez ostrzeżenia.                 |

## 8. P0 – metryki pochodne do obliczania na Desktopie

| **Metryka**                  | **Definicja / warunek**                                                                                       |
|------------------------------|---------------------------------------------------------------------------------------------------------------|
| received_fps                 | frames_received / czas bucketu.                                                                               |
| processed_fps                | frames_processed / czas bucketu.                                                                              |
| app_skip_rate                | (skipped_gate + skipped_transform) / frames_received.                                                         |
| estimated_upstream_gap_rate  | estimated_upstream_gaps / (received + gaps); zawsze oznaczać jako estymację.                                  |
| stage share                  | inference_sum / total; auxiliary_sum / total; pipeline_overhead / total.                                      |
| MZ runs per confirmed track  | suma mz_attempt / liczba tracków potwierdzonych; z eventów, nie z liczby cropów.                              |
| time_to_first_recognition    | pierwszy mz_result z niepustym tekstem – track/session; rozróżnić od globalnego report time_to_first_result.  |
| time_to_confirmed            | consensus_confirmed elapsed – track_created/first detection.                                                  |
| correction_rate              | udział tracków, gdzie pierwszy pełny odczyt był błędny względem GT, a późniejszy konsensus stał się poprawny. |
| false_confirmation_rate      | udział tracków z track_confirmed=true, których potwierdzony tekst ≠ GT.                                       |
| plate exact match by size    | Binning po plate_height_px lub area_ratio; zawsze pokazać N w binie.                                          |
| CER by difficulty            | CER/normalized edit distance względem sharpness, luminance, plate_fit/perspective.                            |
| target_lock_success_rate     | autozoom cycles, które utrzymały target do końca / kompletne autozoom cycles; definicję oprzeć na eventach.   |
| cost per correct recognition | łączny compute/pipeline time / liczba poprawnych unikalnych tracków – pomocnicza metryka systemowa.           |

## 9. Katalog wykresów do pracy inżynierskiej

| **ID** | **Wykres**                                         | **Dane / filtr**                                                                           |
|--------|----------------------------------------------------|--------------------------------------------------------------------------------------------|
| W1     | Latency w czasie + temperatura / headroom          | FRAME total/MT/MZ z THERMAL po elapsed_ms; jedna sesja długa.                              |
| W2     | Received FPS vs processed FPS + skip rate          | FRAME_FLOW; pokazuje przepustowość i adaptacyjny gate.                                     |
| W3     | Udział czasu: inference / auxiliary / overhead     | Stacked bar per wariant R0–R4/runtime.                                                     |
| W4     | Plate height/area vs exact match lub edit distance | TRACK/CROP z GT; scatter + bin summary.                                                    |
| W5     | Sharpness / plate_fit vs błąd                      | TRACK/CROP z GT; scatter / binning.                                                        |
| W6     | R0–R4: p50/p95 pipeline + exact/CER                | SESSION grupowane po series/scenario; powtórzenia jako N/error bars.                       |
| W7     | Liczba prób MZ do potwierdzenia                    | EVENT; histogram/CDF + median/p95.                                                         |
| W8     | Pierwszy odczyt vs konsensus                       | EVENT + GT: correction rate i false confirmation.                                          |
| W9     | Autozoom OFF vs ON wg rozmiaru tablicy             | Porównanie sparowane/warstwowe; exact/CER + time_to_confirmed.                             |
| W10    | Autozoom lock states / failures                    | EVENT: acquired/uncertain/lost, misses i wrong-target jeśli da się oznaczyć z GT/scenario. |
| W11    | Lejek awarii ALPR                                  | GT → MT detected → valid geometry → MZ result → consensus → exact match.                   |
| W12    | Runtime/model Pareto: jakość vs p95 latency        | SESSION; kolor/facet runtime, rozmiar package/RAM jako dodatkowe informacje.               |

## 10. UI przeglądarki – proponowana organizacja

- **Widok „Raport”.** Zachować obecne summary, konfigurację, model/runtime, raw JSON, integralność i pojedynczy trace.

- **Widok „Seria eksperymentalna”.** Nowy: filtr series/scenario/variant/replicate, guard porównywalności, tabela powtórzeń i wykresy porównawcze.

- **Widok „Czas i termika”.** W1/W2 + status termiczny, start/end temperature, throttling transitions.

- **Widok „Jakość i geometria”.** W4/W5/W11, filtrowanie po GT, rozmiarze, sharpness, layout.

- **Widok „Konsensus / Autozoom”.** W7–W10 + event timeline dla wybranego tracku.

- **Widok „Eksport do pracy”.** Wybór wykresów/tabel → zapis obrazu + CSV + LaTeX + analysis_manifest.json.

| **Wykresy:** Obecny ręcznie rysowany Canvas jest wystarczający do prostego trendu latency, ale nie powinien stać się oddzielną implementacją dla każdego scattera, histogramu i wykresu wieloserii. Wydzielić warstwę plot-spec/data i użyć jednego spójnego backendu wykresów dostępnego w projekcie; jeżeli trzeba dodać matplotlib, zrobić to jako świadomą zależność badawczą. |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

## 11. P0 – analiza wielu powtórzeń

- Jednostką porównania jest REPORT/SESSION, nie pojedyncza klatka. Nie wolno traktować 2000 klatek z jednej sesji jak 2000 niezależnych powtórzeń eksperymentu.

- Dla R0–R4 wyświetlać wynik każdej replikacji, medianę/średnią między replikacjami oraz N sesji.

- Dla danych trackowych pokazywać N unikalnych tracków/tablic i coverage GT.

- Jeżeli sceny są sparowane między wariantami, zachować pair/scenario instance ID; jeśli nie – oznaczać analizę jako niesparowaną.

- Wysokie percentyle p90/p95/p99 liczyć w obrębie każdej sesji; agregację między sesjami dokumentować osobno.

## 12. P1 – eksport wyników do pracy

| **Artefakt**           | **Wymaganie**                                                                                            |
|------------------------|----------------------------------------------------------------------------------------------------------|
| figure.png             | Raster do szybkiego wklejenia; min. 1800–2400 px szerokości lub 300 dpi przy docelowym rozmiarze.        |
| figure.svg lub PDF     | Preferowany format wektorowy do finalnej pracy, jeśli backend pozwala.                                   |
| figure_data.csv        | Dokładne dane użyte do wykresu po filtrach.                                                              |
| table.tex              | Tabela LaTeX zgodna z polskim separatorem/opisem jednostek; bez ręcznego kopiowania.                     |
| analysis_manifest.json | Source report IDs/SHA, filters, grouping, metric definitions/version, generated_at, desktop app git SHA. |
| analysis_notes.md/txt  | Opcjonalne automatyczne streszczenie warunków; nie zastępuje manifestu.                                  |

- Każdy wykres powinien być odtwarzalny z analysis_manifest + source report hashes.

- Tytuł/legenda wykresu nie może sugerować przyczynowości, jeśli eksperyment nie kontrolował innych zmiennych.

- Eksport ma zachować jawne N, jednostki i informację o filtrach (np. tylko GT, tylko plate_height \< 35 px).

## 13. Testy i kompatybilność

| **Fixture / test**       | **Kryterium**                                                                                           |
|--------------------------|---------------------------------------------------------------------------------------------------------|
| Stary report.json        | Otwiera się; nowe pola są unavailable bez exception.                                                    |
| Stary legacy ZIP         | Integralność i dotychczasowe wykresy działają.                                                          |
| Nowy .alprsession pełny  | Czyta report + traces + thermal + flow + events + samples; wszystkie klucze łączą się.                  |
| Nowy raport bez headroom | Termika działa z temp/status; brak headroom nie staje się 0.                                            |
| Sesja \>5000 frame rows  | UI preview ograniczony, ale metryki i wykres pełnej sesji korzystają z całości.                         |
| Android trace eviction   | Importer wykrywa retained/evicted i blokuje/ostrzega przy analizie trendu.                              |
| Mixed app SHA            | Comparison guard ostrzega/blokuje domyślnie.                                                            |
| Mixed model fingerprints | Guard zależny od typu eksperymentu.                                                                     |
| Missing GT               | Quality chart/score nie fałszuje 0%; pokazuje brak danych i N=0.                                        |
| Consensus event replay   | Z eventów da się odtworzyć pierwszy odczyt, wszystkie próby i moment confirmation dla testowego tracku. |
| Round-trip analysis      | Wykres → figure_data.csv + manifest → ponowne wygenerowanie daje te same wartości.                      |

## 14. Kolejność implementacji Desktop

1.  **P0. Ustalić właściwą gałąź/bazę i zachować obecny importer.** Nie implementować na starym main bez przeniesienia mobilnych modułów.

2.  **P0. Wydzielić parser + normalized model.** UI ma konsumować gotowe rekordy.

3.  **P0. Pełne dane \> preview.** Oddzielić limit 5000 podglądu od źródła analizy.

4.  **P0. Obsłużyć identity/series/scenario/replicate i comparison guard.** Bez tego nie zaczynać dużej kampanii.

5.  **P0. Obsłużyć thermal/frame_flow/events/extended samples.** Z fallbackiem dla starych raportów.

6.  **P0. Dodać derived metrics.** Formuły wersjonowane i testowane.

7.  **P1. Widoki/wykresy serii.** Najpierw W1–W8; pozostałe po pierwszych danych.

8.  **P1. Eksport do pracy + analysis_manifest.** Automatyzacja danych/tabel/figures.

## 15. Handoff do agenta Android

- Agent Desktop ma zwrócić agentowi Android listę rzeczywiście używanych pól oraz fixture parsera dla nowego kontraktu.

- Jeśli Desktop potrzebuje pola, którego Android nie może tanio zmierzyć, uzgodnić alternatywę (surowe dane do wyliczenia po stronie Desktopu zamiast gotowej metryki).

- Zmiana formatu artefaktu wymaga równoległego fixture i testu kompatybilności; nie uzgadniać schematu wyłącznie ustnie.

- Po pierwszym pełnym end-to-end teście zamrozić wersję kontraktu dla właściwej kampanii eksperymentalnej.
