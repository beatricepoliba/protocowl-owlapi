# Verifica e benchmark (fase 2)

Il branch `fix/benchmark-compliance` parte dalla PR del collega (`review/pr-3`, commit
`06d4163`). Le correzioni sono state organizzate in cinque commit locali. La guida
[MODIFICHE_BENCHMARK.md](MODIFICHE_BENCHMARK.md) spiega ogni gruppo e la storia Git.
Prima della pubblicazione rivedere i commit e il problema residuo nei dati.

## Ambiente e dataset

Usare JDK 21 o successivo. Con il wrapper Gradle 9.0.0 incluso è stato verificato
JDK 24; un JDK più recente potrebbe richiedere un aggiornamento di Gradle.
Su questa macchina:

```bash
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk-24.jdk/Contents/Home
export PATH="$JAVA_HOME/bin:$PATH"
```

Il dataset deve essere in `../dataset` o `../dataset_onto` oppure si può specificare un percorso
assoluto con `DATASET_DIR` (o `-PdatasetDir=...` per Gradle). Sono riconosciute le
cartelle `protocowl/standard` e `protocowl/std`, i file standard `.oprt` e i nomi
`*_protocowl.owl` della consegna locale.

L'elenco di riferimento è `DATASET_DIR/metadata.csv`, se presente, oppure
`benchmark/metadata.csv`. `METADATA_FILE` (o `-PmetadataFile=...`) lo sostituisce.
Il file incluso contiene le 100 righe delle ontologie consegnate, estratte dal
`../metadata.csv` originale che contiene invece 16.555 righe ORE. Non è una
selezione basata sull'esito dei test: tutte le 100 ontologie devono essere presenti
e devono passare. Un file rimosso o una variante mancante causa un errore.

## Esecuzione nell'ordine richiesto

Dalla cartella `protocowl-owlapi`:

```bash
bash ./gradlew :lib:test
bash ./gradlew :lib:coverageTest
bash ./gradlew :lib:datasetTest
```

- `test`: test rapidi, incluse regressioni su annotazioni, costrutti prima omessi,
  catene di proprietà, tutti i bit Reset, riferimenti invalidi e decimali.
- `coverageTest`: 300 operazioni; report `lib/coverage_report.csv` e riepilogo degli
  errori raggruppati per costrutto. Un errore fa fallire il task.
- `datasetTest`: 200 casi distinti; ognuno verifica a, b e c contro il Functional
  originale. Report HTML in `lib/build/reports/tests/datasetTest/index.html` e
  dimensioni in `roundtrip_sizes.csv` (200 righe, una per variante).

Il confronto include ontology/version IRI, imports, prefissi, annotazioni di
ontologia e insieme completo degli assiomi annotati. Gli identificativi anonimi
sono mantenuti durante il caricamento. Nella serializzazione Functional si usa
il renderer OWLAPI con aggiunta automatica delle dichiarazioni disabilitata e
mappa dei prefissi esplicita: nessun assioma viene filtrato dal confronto.
La compatibilità già presente con i file del dataset in codifica legacy è
mantenuta; l'output è nella codifica corrente standard, senza Reset.

## Correzioni del codec

- Annotazioni degli assiomi, incluse declaration, annotation assertion e
  annotazioni annidate; bit della catena di proprietà separato dal bit annotazioni.
- Scrittura di DisjointUnion, SubPropertyChainOf, assiomi sulle data property,
  DatatypeDefinition, HasKey, asserzioni negative e assiomi sulle annotation property.
- Eccezione esplicita per assiomi non supportati (ad esempio regole SWRL).
- Reset selettivo degli indici senza cancellare i prefissi dell'ontologia; errori
  `OWLParserException` per riferimenti non dichiarati.
- Decimali compatti: ricostruzione della frazione memorizzata al contrario.
- Rimozione delle stampe e scritture di debug dal parser, che alteravano le misure.
- Conservazione delle forme lessicali non canoniche in scrittura numerica.

## Benchmark

Servono Bash e GNU time. Su Linux/WSL verificare `/usr/bin/time -v true`.
Su macOS installare GNU time (`brew install gnu-time`) e usare `TIME_CMD=gtime`.
Il BSD time di macOS non fornisce l'interfaccia GNU richiesta.

```bash
export BENCH_JAVA_OPTS="-Xms2g -Xmx8g"  # adattare alla memoria disponibile
bash ./run_benchmark.sh
```

Lo script esegue prima test, copertura e round-trip; ogni misura usa poi una JVM
separata. `benchmark_results.csv` viene sovrascritto. L'ambiente, le opzioni JVM,
la revisione e il manifest dei file vengono registrati in `benchmark_environment.txt`.
La validazione finale richiede 500 righe, tutte le combinazioni attese, nessun
duplicato, nessun render MIS_128, metriche numeriche corrette e dimensioni del
render ProtocOWL uguali a quelle misurate nel round-trip standard della fase 2.

I CSV del benchmark ereditati dalla PR non rappresentano una nuova misura del
codec corretto: rigenerarli con GNU time prima della consegna.

## Come integrare le correzioni nella PR

Non fondere la PR difettosa in `main` per poterla correggere. Il branch di correzione
contiene già i commit del collega e le modifiche successive. Dopo revisione e test:

1. Rivedere i cinque commit delle correzioni sul branch `fix/benchmark-compliance`.
2. Se si vuole conservare la PR del collega, trasferire i commit sul suo branch
   sorgente con il suo accordo e i permessi necessari: il push su quel branch
   aggiorna automaticamente la PR esistente.
3. In alternativa, pubblicare questo branch e aprire una PR verso il branch del
   collega (sole correzioni), oppure verso `main` come PR sostitutiva (comprende
   anche il lavoro del collega). Indicare chiaramente la relazione con la PR #3.

Non serve riscrivere la storia né eseguire force-push.

## Stato della verifica e blocco sui dati forniti

Vedere [known_dataset_issues.md](benchmark/known_dataset_issues.md): cinque
ontologie hanno decimali nei binari che non corrispondono ai Functional.
Non sono state escluse. Il benchmark resta bloccato dalla fase 2 finché vengono
forniti i file corretti; i report rossi sono intenzionalmente conservati.
