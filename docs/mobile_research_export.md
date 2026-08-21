# Eksport badawczy klienta mobilnego ALPR

## Artefakty

Ekran Diagnostyka zwraca sterowanie do `MainActivity`, która udostępnia trzy
formaty:

- `*.alprsession` — pełny `alpr.mobile_research_bundle.v1`;
- `alpr_thesis_*.zip` — `alpr.mobile_thesis_bundle.v1`;
- klasyczny ZIP `alpr.mobile_benchmark_report.v1` dla zgodności wstecznej.

Eksport jest strumieniowy. Pliki modeli i JPEG nie są agregowane w pamięci RAM.
Na czas zapisu kolektor zostaje wstrzymany, a snapshot cropów jest chroniony
przed wyparciem. `manifest.json` jest ostatnim wpisem archiwum i zawiera
SHA-256 wszystkich wcześniejszych wpisów.

## Ground truth

Każdy crop ma niezależny stan:

- `not_reviewed`;
- `accepted` — ground truth jest równy oryginalnej predykcji;
- `rejected` — wiadomo, że odczyt jest błędny, ale nie znamy transkrypcji;
- `corrected` — `ground_truth_text` pochodzi od użytkownika.

Raport zawsze przechowuje `original_prediction`, status, czas i rewizję. CER i
exact match są liczone tylko dla `accepted/corrected`. Jednostką jakościową
jest unikalna para `session_id + track_id`; kilka klatek tego samego tracku nie
zwiększa sztucznie liczebności próby.

Normalizacja sekwencji jest jawna: wielkie litery i usunięcie białych znaków.
CER jest ilorazem sumy odległości Levenshteina i sumy długości ground truth.
Raport zawiera też średnią znormalizowaną odległość edycyjną per próbka.

## Modele

Importer zachowuje od tej wersji dokładny plik `source.alprmodel` obok
zainstalowanego modelu lub kompletnego pipeline'u. Pełny eksport preferuje
oryginalny kompletny pakiet. Dla starszej instalacji, która nie ma źródłowego
archiwum, dołącza manifesty i pełne katalogi plików aktywnych modeli.

`report.json/execution` wiąże osobno MP, MT i MZ z:

- `model_id`, nazwą, wersją i fingerprintem;
- wariantem, runtime'em, precyzją, delegatem i liczbą wątków;
- efektywnym wejściem, dekoderem i progami;
- listą plików, SHA-256 oraz rozmiarem artefaktu;
- rodziną YOLO, liczbą parametrów i datą eksportu, jeżeli manifest je zawiera.

## TeX

`summary.tex` jest samodzielnym dokumentem i używa względnych `\input{}` dla
tabel. Generator escapuje znaki specjalne TeX. Paczka zawiera:

- konfigurację MT/MZ/MP;
- exact match, CER i liczebność ground truth;
- p50/p90/p95/p99 MT, MZ i pipeline'u;
- do 12 cropów w treści dokumentu oraz wszystkie dostępne cropy w katalogu;
- `references.bib`, `metadata.json` i CSV śladów do własnych wykresów.

## Metodyka

`protocol.json` opisuje założenia benchmarku inspirowanego MLPerf Mobile:
single-stream, cel 1024 próbek i 60 sekund, p90 jako percentyl główny. Pole
`mlperf_compliant` ma wartość `false`, ponieważ aplikacja nie korzysta z
oficjalnego LoadGena. Bieżący eksport opisuje sesję live; kontrolowany replay
cropów pozostaje osobnym trybem wykonawczym do wdrożenia.
