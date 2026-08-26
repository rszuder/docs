# Aktualizacja `model_package_test_strategy.md` — 2026-08-26

Poniższe zmiany należy nanieść na obecną strategię bez usuwania jej szczegółowych
bramek T0–T10 i G0–G9.

## 1. Nagłówek

Zastąpić:

```text
Stan dokumentu: 2026-08-20
```

przez:

```text
Stan dokumentu: 2026-08-26
```

## 2. T5 — macierz runtime

Wiersz NCNN jest nieaktualny.

Było:

```text
NCNN | import zawsze, wykonanie po dodaniu JNI | jawny status unsupported bez crasha
```

Powinno być:

```text
NCNN FP32 | CPU 1/2/4 wątki na obsługiwanym ABI | inferencja przez JNI, brak NaN/crasha i zgodność semantyczna
```

Dopisać:

- backend NCNN `20260526` jest wykonywalny na `arm64-v8a` i `armeabi-v7a`;
- Vulkan nie należy do bieżącej obowiązkowej macierzy;
- autotuning NCNN porównuje profile CPU tego samego wariantu modelu.

## 3. T8 — kontrolowany benchmark mobilny

Ogólna zasada co najmniej trzech serii pozostaje poprawna dla końcowego benchmarku
modeli/runtime'ów.

Nie należy jednak przenosić jej mechanicznie na każdy eksperyment diagnostyczny.
Dla eksperymentów inżynierskich o bardzo dużym efekcie, np. R0/R1/R2, można wcześniej
wykonać pilot i serię potwierdzającą, pod warunkiem jawnego opisania protokołu,
surowych trace'ów i kryterium wykonania dodatkowej próby.

## 4. T9 — sesje live

Dodać scenariusze:

- R0/R1/R2 przy identycznym warunku termicznym START;
- auto-zoom OFF vs ON jako osobny eksperyment po zamrożeniu bazowego pipeline'u;
- pojedyncza tablica oraz kilka konkurencyjnych tablic;
- mała tablica uruchamiająca zoom;
- brak tekstu przed zoomem i pierwszy poprawny MZ dopiero po zoom-in;
- rzeczywista zmiana sceny podczas zbliżenia;
- lekkie drżenie telefonu i kontrola dryfu trackera;
- powrót do `1x` i zachowanie tożsamości celu.

## 5. T10 — integralność raportu

Dodać kontrolę:

- `camera_zoom_ratio` i `capture_source` w cropach;
- spójność `auto_zoom_target_roi` i liczników blokady celu w trace;
- brak mylenia `confirmed` ze stanem ręcznej walidacji;
- w przyszłości: strukturalne `thermal_start/thermal_end`.

## 6. Sekcja „Stan pokrycia”

Zaktualizować co najmniej:

| Obszar | Stan 2026-08-26 |
| --- | --- |
| NCNN/JNI | wdrożone i potwierdzone na SM-A125F, CPU |
| realny pakiet MP+MT+MZ | zaimportowany i uruchomiony |
| rzeczywisty MP/NCNN | inferencja i autotuning potwierdzone |
| R0/R1/R2 | wykonane sesje live, w tym scena z jednym i dwoma pojazdami |
| bilans PIPE/INF/AUX/OVH | wdrożony i zweryfikowany na urządzeniu |
| rotacja CameraX | zoptymalizowana przez `setOutputImageRotationEnabled(true)` |
| warunek termiczny START | wdrożony: BAT + TH + stabilizacja |
| Thermal Headroom | odczyt obserwacyjny, bez progu decyzyjnego |
| auto-zoom | logika i testy JVM wdrożone; pełna walidacja urządzeniowa jeszcze otwarta |
| kontrolowany replay 1024/60 s | nadal brak |
| importer `.alprsession` w Pythonie | nadal brak |

## 7. Ważne rozdzielenie eksperymentów

Nie wolno łączyć w jednej serii zmian:

- polityki ROI;
- rozdzielczości źródła;
- runtime'u/precyzji modelu;
- auto-zoomu;
- progów trackera/schedulera.

Najpierw zamrozić bazowy wariant, potem zmieniać jedną warstwę naraz.
