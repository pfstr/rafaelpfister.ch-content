---
title: "Testare il Cold Storage con Rclone: un piano di test pratico"
navTitle: "Testare Rclone"
description: "Prima che un servizio legga i propri file dal cloud tramite un mount Rclone, occorre verificare più del semplice accesso alle directory. Questo piano di test copre Cold Read, Warm Read, operazioni di scrittura, comportamento della cache, integrità dei file e guasti."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "11 min di lettura"
themen:
  - rclone
  - testing
related:
  - rclone-mount-in-docker-container
  - paperless-dokumente-clouddienst-auslagern
slug: "testare-il-cold-storage-con-rclone-un-piano-di-test-pratico"
translationOf: "cloud-mount-testen-dummy-pdfs"
translationId: article-8592f808b2e93cd4
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:20:23.034Z
translationReview: required
translationSourceHash: 27bc45a50d8e84fc785eaf04ec6814054e327f516587d0f9f9a101c989ce2aa1
url: https://rafaelpfister.ch/it/blog/testare-il-cold-storage-con-rclone-un-piano-di-test-pratico
---

Un mount Rclone si configura rapidamente. Il remote appare come una directory, `ls` mostra i file e il primo test funzionale è superato. Tuttavia, questo dice ancora poco sull'uso in produzione.

Non appena un servizio accede al mount, sorgono ulteriori domande: quanto dura il primo accesso a un file? Quali accessi vengono serviti dalla cache locale? Cosa accade a un file non ancora caricato se Rclone si arresta? Un container in esecuzione vede nuovamente il mount ricreato? E come reagisce il servizio se il cloud è temporaneamente irraggiungibile?

Questo articolo fornisce un piano di test generale. Può essere utilizzato per un archivio di documenti, un media server, una gestione di foto o qualsiasi altro servizio che recuperi file usati raramente tramite Rclone da un Cold Storage.

## Le opzioni principali di rclone

Per orientarsi, ecco le opzioni Rclone presenti in questo piano di test, tradotte liberamente dalla documentazione:

<details class="options-details">
<summary>Panoramica delle opzioni</summary>

| Opzione | Significato |
|---|---|
| `--seed n` | Valore iniziale del generatore casuale in `rclone test makefiles`; lo stesso seed genera lo stesso albero di file |
| `--files n` | Numero di file di test da generare |
| `--files-per-directory n` | Numero medio di file per directory |
| `--min-file-size grösse` | Dimensione minima del file generato (sono consentiti suffissi come K, M, G) |
| `--max-file-size grösse` | Dimensione massima del file generato |
| `--progress` | Indicatore di avanzamento durante il trasferimento |
| `--stats dauer` | Intervallo con cui vengono visualizzate le statistiche di trasferimento |
| `--log-file datei` | Scrive il log nel file specificato |
| `--log-level stufe` | Livello di dettaglio del log: DEBUG, INFO, NOTICE o ERROR |
| `--one-way` | Con `rclone check` verifica solo che tutti i file sorgente siano presenti e identici nella destinazione; i file aggiuntivi nella destinazione non sono considerati errori |
| `--download` | Durante il confronto scarica effettivamente i dati invece di confrontare solo gli hash |
| `--vfs-cache-mode modus` | Strategia di cache del layer VFS; `full` memorizza localmente nella cache gli accessi in lettura e scrittura |
| `--cache-dir verzeichnis` | Directory per la cache locale |
| `--vfs-cache-max-size grösse` | Limite soft per la dimensione complessiva della cache VFS |
| `--vfs-cache-poll-interval dauer` | Intervallo con cui Rclone controlla e pulisce la cache |
| `--vfs-write-back dauer` | Tempo di attesa dopo la chiusura di un file prima che inizi il caricamento nel remote |
| `--vfs-read-ahead grösse` | Quantità aggiuntiva di dati letta in anticipo oltre la posizione richiesta con `full` |
| `--poll-interval dauer` | Intervallo con cui Rclone interroga il remote per rilevare modifiche, solo per backend che supportano il polling |
| `--dir-cache-time dauer` | Durata di validità degli elenchi di directory memorizzati nella cache |
| `--allow-other` | Consente l'accesso al mount FUSE anche a utenti diversi da quello che ha effettuato il mount |

</details>

Gli elenchi completi sono disponibili nella documentazione di Rclone, in particolare in [rclone mount](https://rclone.org/commands/rclone_mount/) e nella panoramica dei [flag globali](https://rclone.org/flags/).

## Definire innanzitutto ciò che si vuole ottenere

Cold Storage non significa automaticamente la stessa cosa per ogni applicazione. Un media server legge generalmente file grandi in modo sequenziale. Una gestione di foto carica molti piccoli dati di anteprima e salta a posizioni diverse. Un archivio di documenti apre file relativamente piccoli, ma spesso una sola volta.

Prima del test, annotate le caratteristiche più importanti del vostro archivio reale:

- dimensione tipica dei file e file più grande presente
- numero di file per directory
- lettura completa o accessi casuali a singole aree
- rapporto tra accessi in lettura e scrittura
- numero di utenti o processi simultanei
- modifiche effettuate direttamente nel remote al di fuori del mount
- tempo di attesa accettabile per un Cold Read
- spazio massimo disponibile per la cache locale

Solo da questi elementi derivano criteri di successo sensati. Aprire un singolo file in 1,2 secondi può essere perfettamente accettabile per un archivio e inutilizzabile per un'applicazione interattiva.

## Generare un set di dati di test riproducibile

Rclone include già uno strumento adatto. `rclone test makefiles` genera ogni volta lo stesso albero di file con un seed fisso:

```bash
rclone test makefiles ./testdata \
  --seed 42 \
  --files 250 \
  --files-per-directory 25 \
  --min-file-size 16K \
  --max-file-size 32M
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `./testdata` | Directory di destinazione in cui viene creato l'albero di test |
| `--seed 42` | Valore iniziale fisso del generatore casuale; ogni esecuzione genera lo stesso insieme di dati |
| `--files 250` | 250 file di test in totale |
| `--files-per-directory 25` | In media 25 file per directory |
| `--min-file-size 16K` | File più piccolo: 16 KiB |
| `--max-file-size 32M` | File più grande: 32 MiB |

</details>

Adattate numero e dimensioni al vostro insieme di dati reale. Non testate solo file di dimensione media. Alcuni file molto piccoli mostrano quanto siano costosi gli accessi ai metadati; alcuni file grandi rendono visibili throughput, read-ahead e comportamento della cache.

Aggiungete inoltre nomi di file che potrebbero causare problemi nella pratica:

```bash
mkdir -p "testdata/Sonderfälle/Unterordner"
printf 'Leerzeichen\n' > "testdata/Sonderfälle/Datei mit Leerzeichen.txt"
printf 'Umlaute\n' > "testdata/Sonderfälle/Grösse und Änderung.txt"
printf 'Grossschreibung\n' > "testdata/Sonderfälle/Test.txt"
printf 'Kleinschreibung\n' > "testdata/Sonderfälle/test.txt"
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `mkdir -p` | Crea anche le directory principali mancanti e non segnala un errore se la directory esiste |
| `printf '…\n' > datei` | Scrive il testo indicato come contenuto del file; la redirezione crea il file con il nome problematico |

</details>

L'ultimo test è particolarmente importante se il file system locale e il backend cloud trattano in modo diverso maiuscole e minuscole.

Se il vostro servizio accetta solo determinati formati, file binari arbitrari non sono sufficienti. In tal caso, generate anche file sintetici esattamente in questi formati. Con Paperless-ngx si trattava di PDF con un vero livello di testo, affinché il test non misurasse per errore le prestazioni OCR invece del percorso di archiviazione. Per una gestione di foto, l'insieme deve includere diverse dimensioni e formati di immagine; per un media server, file brevi con codec diversi.

## Una misurazione di riferimento senza mount

Prima che entrino in gioco FUSE e la cache VFS, dovreste misurare direttamente il backend. Copiate l'insieme di dati nel remote di test con Rclone e salvate un log dettagliato:

```bash
rclone copy ./testdata remote:cold-storage-test \
  --progress \
  --stats 5s \
  --log-file rclone-copy.log \
  --log-level INFO
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `./testdata` | Origine della copia: l'insieme di dati di test generato localmente |
| `remote:cold-storage-test` | Destinazione: percorso nel remote configurato |
| `--progress` | Indicatore di avanzamento nel terminale |
| `--stats 5s` | Statistiche di trasferimento ogni 5 secondi invece dell'intervallo predefinito |
| `--log-file rclone-copy.log` | Log completo in un file per analisi successive |
| `--log-level INFO` | Registra trasferimenti, retry ed errori senza il dettaglio di DEBUG |

</details>

Verificate poi che origine e destinazione corrispondano:

```bash
rclone check ./testdata remote:cold-storage-test \
  --one-way \
  --download
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `./testdata` | Riferimento: l'insieme di dati originale locale |
| `remote:cold-storage-test` | Oggetto della verifica: l'insieme di dati appena caricato nel remote |
| `--one-way` | Verifica solo che tutti i file sorgente siano presenti e identici nella destinazione; i file aggiuntivi nella destinazione non vengono segnalati |
| `--download` | Scarica i dati e confronta i contenuti invece di affidarsi agli hash |

</details>

`--download` è decisivo in questo caso, perché alcuni backend non forniscono hash adeguati. Il confronto richiede più tempo, ma offre una base utile per il successivo test di integrità.

Annotate tempo di upload, velocità di trasferimento, numero di retry ed errori API. Se già l'accesso diretto è instabile, il mount non può risolvere il problema.

## Separare il mount di test dalla cache di produzione

Per le misurazioni, utilizzate un punto di mount e una directory di cache dedicati:

```bash
rclone mount remote:cold-storage-test /mnt/rclone-test \
  --vfs-cache-mode full \
  --cache-dir /var/cache/rclone-test \
  --vfs-cache-max-size 10G \
  --vfs-cache-poll-interval 1m \
  --allow-other \
  --log-file /var/log/rclone-test.log \
  --log-level INFO
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `remote:cold-storage-test` | Remote da montare con relativo percorso |
| `/mnt/rclone-test` | Punto di mount sul sistema di test |
| `--vfs-cache-mode full` | Memorizza completamente nella cache locale gli accessi in lettura e scrittura |
| `--cache-dir /var/cache/rclone-test` | Directory di cache dedicata, separata dalla cache di produzione |
| `--vfs-cache-max-size 10G` | Limite soft di 10 GiB per la cache VFS |
| `--vfs-cache-poll-interval 1m` | Controllo e pulizia della cache ogni minuto |
| `--allow-other` | Consente l'accesso anche a utenti diversi da quello che ha effettuato il mount; necessario per servizi e container |
| `--log-file /var/log/rclone-test.log` | Log in un file per tracciare gli accessi durante i test |
| `--log-level INFO` | Livello di dettaglio medio del log |

</details>

I valori sono un esempio e non una raccomandazione generale. Ciò che conta è la separazione: una cache di test vuota rende riproducibili i Cold Read senza dover eliminare file da una cache di produzione in uso.

`--vfs-cache-mode full` è generalmente la modalità di test più istruttiva per le applicazioni. Rclone memorizza localmente nella cache gli accessi in lettura e scrittura e può rappresentare meglio accessi ai file che non sarebbero possibili con un puro object storage. La compatibilità aggiuntiva richiede spazio locale.

## Verificare sempre dal punto di vista del servizio reale

Un mount può funzionare per il vostro utente ma risultare comunque inutilizzabile per il servizio. Cause frequenti sono un ID utente diverso, l'assenza di `--allow-other`, limiti dei container o una propagazione del mount errata.

Eseguite quindi almeno un accesso in lettura completo con la stessa identità con cui verrà eseguita in seguito l'applicazione:

```bash
sudo -u <service-user> sha256sum /mnt/rclone-test/pfad/zur/datei
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-u <service-user>` | Esegue il comando come l'utente indicato, non come root |
| `/mnt/rclone-test/pfad/zur/datei` | File da leggere; `sha256sum` forza una lettura completa |

</details>

Se il servizio viene eseguito in Docker, il test deve essere effettuato nel container:

```bash
docker exec --user <uid>:<gid> <app-container> \
  sha256sum /pfad/im/container/datei
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `--user <uid>:<gid>` | Esegue il comando nel container con questo ID utente e gruppo, indipendentemente dall'utente predefinito dell'immagine |
| `<app-container>` | Nome o ID del container applicativo in esecuzione |
| `sha256sum /pfad/im/container/datei` | Comando da eseguire; il percorso è il mount dal punto di vista del container |

</details>

Ancora meglio è un test dell'applicazione reale. Aprite il file tramite l'interfaccia web o l'API del servizio. Solo così noterete se l'applicazione, ad esempio, avvia più letture parallele, salta alla fine del file o richiede metadati aggiuntivi.

## Misurare separatamente Cold Read e Warm Read

Con `--vfs-cache-mode full` esistono tre livelli tra l'applicazione e il cloud:

| Livello | Cosa contiene |
|---|---|
| Remote | il file completo nel servizio cloud |
| Cache VFS | aree memorizzate localmente di file già letti |
| Linux Page Cache | dati utilizzati di recente nella RAM |

Per un Cold Read, scegliete un file il cui contenuto non sia mai stato letto tramite il mount di test. Nel Warm Read effettuato immediatamente dopo, il file si trova nella cache VFS e di norma anche nella RAM.

```bash
measure_read() {
  file="$1"
  label="$2"
  start=$(date +%s%3N)
  cat "$file" > /dev/null
  end=$(date +%s%3N)
  printf '%s: %s ms\n' "$label" "$((end - start))"
}

measure_read "/mnt/rclone-test/grosse-datei.bin" "Cold Read"
measure_read "/mnt/rclone-test/grosse-datei.bin" "Warm Read"
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `date +%s%3N` | Timestamp in millisecondi: secondi Unix seguiti direttamente dalle prime tre cifre della parte in nanosecondi (GNU date) |
| `cat "$file" > /dev/null` | Legge completamente il file e scarta l'output; viene misurato solo il tempo di lettura |
| `"$1"`, `"$2"` | Argomenti della funzione shell: percorso del file ed etichetta della riga di misurazione |

</details>

Non misurate un solo file. Utilizzate almeno dieci file di dimensioni diverse mai letti in precedenza e annotate mediana, valore più lento e dimensione del file. Un singolo valore migliore non è una base per decidere.

Un Warm Read non è un puro test del disco, perché il kernel può mantenere in RAM parti del file. Per la maggior parte degli scenari Cold Storage non è un problema. Ciò che conta è ciò che un utente sperimenta alla prima apertura e alle aperture successive. Se volete valutare separatamente RAM e disco locale, dovete controllare e svuotare in modo dimostrabile anche la page cache.

## Non testare solo letture complete

`cat` legge un file dall'inizio alla fine. Molte applicazioni si comportano diversamente:

- Un lettore video legge dapprima header e indice, salta in seguito a un'altra posizione e poi continua a caricare in modo sequenziale.
- Una gestione di immagini legge i metadati e genera poi un'anteprima.
- Un programma di archiviazione può leggere prima la fine del file.
- Più worker possono accedere contemporaneamente a file diversi.

Testate questi flussi con l'applicazione effettiva. Osservate parallelamente il log di Rclone e la cache. Per i file grandi, è interessante vedere quanto Rclone memorizzi davvero localmente e se `--vfs-read-ahead` si adatti al modello di accesso.

Inoltre, un mount Rclone non è una posizione di archiviazione sensata per database o altri file che richiedono locking affidabile e modifiche frequenti all'interno dello stesso file. Il layer VFS compensa le differenze tra file system e object storage, ma non trasforma il backend in un file system locale.

## Collaudare separatamente il percorso di scrittura

Se il servizio legge soltanto, montate il remote in sola lettura quando possibile. Se deve scrivere, testate separatamente creazione, sovrascrittura, rinomina ed eliminazione.

Un file scritto non compare necessariamente subito nel remote. Con la cache VFS attiva, l'upload inizia solo dopo che il file è stato chiuso e `--vfs-write-back` è trascorso. Verificate quindi entrambi gli stati:

1. L'applicazione ha chiuso il file correttamente.
2. Il file è poi leggibile nel remote tramite un accesso diretto con Rclone.

```bash
printf 'writeback-test\n' > /mnt/rclone-test/writeback-test.txt

# Dopo la scadenza di --vfs-write-back:
rclone cat remote:cold-storage-test/writeback-test.txt
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `/mnt/rclone-test/writeback-test.txt` | File di destinazione nel mount; la redirezione scrive tramite la cache VFS |
| `remote:cold-storage-test/writeback-test.txt` | Accesso diretto senza passare dal mount: `rclone cat` legge il file dal remote e lo invia su stdout |

</details>

Ripetete il test con un file grande e terminate Rclone mentre l'upload è ancora in corso. Riavviate poi usando la stessa directory di cache e controllate se l'upload riprende. Proprio questa finestra temporale determina quanti dati siano a rischio in caso di guasto del server.

Testate anche rinomina ed eliminazione. Molti backend cloud rappresentano queste operazioni in modo diverso da un file system locale. Non è rilevante solo che il comando termini con successo, ma anche quando la modifica diventa visibile tramite un accesso diretto al remote e per altri client.

## Testare le modifiche al di fuori del mount

I file possono essere modificati tramite l'interfaccia web del provider, un secondo processo Rclone o un altro server. Il mount non vede sempre subito tali modifiche, poiché le informazioni delle directory sono memorizzate nella cache.

Create quindi un file direttamente nel remote con una seconda chiamata Rclone:

```bash
printf 'external-change\n' > external-change.txt
rclone copyto external-change.txt \
  remote:cold-storage-test/external-change.txt
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `external-change.txt` | Origine: il file generato localmente |
| `remote:cold-storage-test/external-change.txt` | Destinazione con nome file esatto; `copyto` copia un singolo file con esattamente questo nome, anziché copiarlo in una directory come `copy` |

</details>

Misurate quando il file appare nel mount. Ripetete il test per modifica ed eliminazione. Il risultato dipende dal backend, dal suo supporto per il polling, nonché da `--poll-interval` e `--dir-cache-time`. Se l'applicazione deve vedere subito le modifiche attuali, questo comportamento deve rientrare esplicitamente nei criteri di accettazione.

Con l'interfaccia Remote Control attivata, potete svuotare in modo mirato la cache delle directory:

```bash
rclone rc vfs/forget
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `vfs/forget` | Comando Remote Control da eseguire: scarta l'albero delle directory VFS memorizzato nella cache, così che l'accesso successivo interroghi nuovamente il remote |

</details>

Questo è utile per un test manuale, ma non sostituisce una strategia operativa adeguata.

## Mettere la cache sotto pressione

Una cache quasi vuota è il caso più semplice. In un secondo ciclo di test, impostate intenzionalmente `--vfs-cache-max-size` su un valore basso e leggete più dati di quanti ve ne possano entrare.

```bash
du -sh /var/cache/rclone-test/vfs
du -sh --apparent-size /var/cache/rclone-test/vfs
find /var/cache/rclone-test/vfs -type f | wc -l
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `du -s` | Riassume l'utilizzo dello spazio in una riga di totale anziché elencare ogni sottodirectory |
| `du -h` | Output in unità leggibili dall'uomo (K, M, G) |
| `du --apparent-size` | Mostra la dimensione logica del file invece dello spazio effettivamente occupato sul disco |
| `find … -type f` | Considera solo file regolari, non directory |
| `wc -l` | Conta le righe dell'output, in questo caso il numero di file nella cache |

</details>

Le due dimensioni possono differire notevolmente. In modalità Full, Rclone usa sparse file: un file mostra la sua intera dimensione logica, anche se solo le aree lette occupano spazio locale.

Inoltre, il limite della cache è soft. Rclone lo controlla con la frequenza di `--vfs-cache-poll-interval`, e i file aperti non possono essere rimossi. La cache può quindi superare temporaneamente il limite. Tuttavia, dopo la chiusura dei file e il successivo ciclo di pulizia dovrebbe tornare a ridursi.

Registrate il valore di picco, il valore dopo la pulizia e il tempo necessario. In questo modo è possibile dimensionare in modo ragionevole lo spazio locale necessario.

## Simulare due guasti diversi

Un cloud irraggiungibile e un processo Rclone arrestato sono due errori differenti:

| Guasto | Cosa viene verificato |
|---|---|
| Backend o rete irraggiungibili, Rclone continua a funzionare | Comportamento con retry, timeout e file già memorizzati nella cache |
| Processo Rclone terminato | Comportamento del mount FUSE e ripristino del punto di mount |

Simulate entrambi solo nell'ambiente di test. Per il secondo caso, potete terminare forzatamente un container Rclone:

```bash
docker kill --signal KILL <rclone-container>
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `--signal KILL` | Invia SIGKILL anziché il segnale predefinito SIGTERM; il processo non ha occasione di eseguire la pulizia |
| `<rclone-container>` | Nome o ID del container Rclone |

</details>

Durante il guasto, verificate l'applicazione e non solo il punto di mount:

- Quali funzioni rimangono disponibili?
- Quanto attende un accesso prima che compaia un errore?
- I file già interamente memorizzati nella cache sono ancora accessibili?
- L'applicazione interrompe le nuove operazioni di scrittura?
- Compare un messaggio di errore comprensibile oppure solo un processo bloccato?
- Il monitoraggio si attiva?

Un servizio di scrittura non deve scrivere inosservato nella directory locale sottostante quando il mount è assente. Dopo il ritorno del mount, questi file verrebbero nascosti. Una semplice protezione prima di ogni job di scrittura è:

```bash
mountpoint -q /mnt/rclone-test || exit 1
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-q` | Nessun output; il risultato è disponibile solo nel codice di uscita |
| `/mnt/rclone-test` | Percorso da verificare; codice di uscita 0 solo se vi è realmente attivo un mount |
| `\|\| exit 1` | Interrompe lo script se il percorso non è un punto di mount |

</details>

Dopo il riavvio di Rclone, controllate il mount sull'host e da ogni container che lo utilizza. Un mount ricreato raggiunge un container già in esecuzione solo con la corretta propagazione del mount. Per Docker, sul lato che lo utilizza è generalmente necessario `rslave`. I dettagli sono disponibili nell'articolo [Gestire in modo affidabile i mount Rclone in Docker](/blog/rclone-mount-in-docker-container).

## Un esempio concreto con Paperless-ngx

Per il mio test con Paperless ho generato 40 PDF per un totale di 13,9 MB. Un documento mai aperto prima ha richiesto circa 1,8 secondi; un accesso ripetuto immediatamente ha richiesto da 19 a 24 millisecondi. Una cache VFS limitata a 4 MB è salita temporaneamente a 12,7 MiB ed è stata ripulita al ciclo successivo.

Mentre il remote non era raggiungibile, l'elenco dei documenti, la ricerca full-text e le anteprime hanno continuato a funzionare, perché questi dati erano locali. Solo l'originale non poteva essere aperto. Dopo il ripristino del mount, il container Paperless in esecuzione ha potuto nuovamente accedere ai file senza dover essere riavviato.

Questi numeri non sono un benchmark per Rclone o Proton Drive. È interessante il comportamento: l'Hot Storage è rimasto disponibile localmente, i Cold Read erano più lenti ma prevedibili e il servizio si è ripreso dopo il guasto.

## Cosa includere nel protocollo di test

Un risultato tracciabile anche in seguito contiene almeno:

- versione di Rclone e backend utilizzato
- sistema operativo, variante FUSE e file system della directory di cache
- comando di mount completo senza credenziali
- numero, distribuzione delle dimensioni e struttura dei file di test
- valori di Cold Read e Warm Read per più file
- durata della scrittura fino alla visibilità nel remote
- valore di picco della cache e durata della pulizia
- risultato di `rclone check --download`
- comportamento in caso di guasto del backend e processo Rclone terminato
- tempo di ripristino dal punto di vista dell'applicazione
- retry, timeout, limitazioni e errori di autenticazione nel log

Definite in anticipo un valore limite per ogni punto. Così il test termina con una decisione e non solo con una raccolta di numeri interessanti.

## Quando la configurazione è pronta

Un mount Cold Storage è pronto all'uso se potete rispondere sì a queste domande:

- I Cold Read sono abbastanza rapidi per il servizio previsto?
- La cache accelera gli accessi ripetuti come previsto?
- Il fabbisogno di spazio locale resta controllabile anche sotto carico?
- Tutti i file corrispondono dopo un download completo?
- Tutte le operazioni sui file necessarie funzionano con il backend scelto?
- L'applicazione si comporta in modo controllato in caso di guasto del cloud?
- Le operazioni di scrittura vengono interrotte in sicurezza quando il mount è assente?
- Un mount ricreato raggiunge tutti i consumer in esecuzione?
- Il monitoraggio segnala il guasto prima che lo segnali un utente?

Se manca una risposta, sapete almeno esattamente su cosa dovete continuare a lavorare. È molto più utile di un mount che al primo `ls` sembrava valido e mostra i suoi limiti solo durante l'esercizio.

## Fonti

1.  [Rclone test makefiles](https://rclone.org/commands/rclone_test_makefiles/): file di test e strutture di directory riproducibili con dimensioni configurabili.

2.  [Rclone mount](https://rclone.org/commands/rclone_mount/): modalità della cache VFS, writeback, sparse file, limiti della cache e cache delle directory.

3.  [Rclone check](https://rclone.org/commands/rclone_check/): confronto tra origine e destinazione, inclusa la verifica completa con `--download`.

4.  [Rclone Remote Control](https://rclone.org/rc/): svuotamento mirato della cache delle directory VFS con `vfs/forget`.

5.  [Rclone Global Flags](https://rclone.org/flags/): riferimento completo delle opzioni globali, incluse registrazione, statistiche e parametri VFS.
