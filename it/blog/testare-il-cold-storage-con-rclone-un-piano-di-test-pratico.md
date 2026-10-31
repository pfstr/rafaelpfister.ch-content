---
title: "Testare il Cold Storage con Rclone: un piano di test pratico"
navTitle: "Testare Rclone"
description: "Prima che un servizio legga i propri file dal cloud tramite un mount Rclone, è opportuno verificare più del semplice accesso alle directory. Questo piano di test copre Cold Read, Warm Read, operazioni di scrittura, comportamento della cache, integrità dei file e guasti."
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
translatedAt: 2026-10-31T03:26:41.130Z
translationReview: required
translationSourceHash: 820965b014837c0d0b5b5bf629a02842f2ebbbb26f62d343d8f1630278cd3bec
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/testare-il-cold-storage-con-rclone-un-piano-di-test-pratico
---

Un mount Rclone si configura rapidamente. Il remote appare come una directory, `ls` mostra i file e il primo test funzionale è superato. Tuttavia, questo dice ancora poco sul funzionamento in produzione.

Non appena un servizio accede al mount, sorgono ulteriori domande: quanto dura il primo accesso a un file? Quali accessi vengono serviti dalla cache locale? Cosa succede a un file non ancora caricato se Rclone si arresta in modo anomalo? Un container in esecuzione vede di nuovo il mount ricreato? E come reagisce il servizio se il cloud non è temporaneamente raggiungibile?

Questo articolo fornisce un piano di test generale a tale scopo. Può essere utilizzato per un archivio di documenti, un media server, una gestione fotografica o qualsiasi altro servizio che recupera file usati raramente tramite Rclone da un Cold Storage.

## Le opzioni più importanti di rclone

Per orientarsi, ecco le opzioni Rclone presenti in questo piano di test, tradotte liberamente dalla documentazione:

<details class="options-details">
<summary>Panoramica delle opzioni</summary>

| Opzione | Significato |
|---|---|
| `--seed n` | Valore iniziale del generatore casuale in `rclone test makefiles`; lo stesso seed produce lo stesso albero di file |
| `--files n` | Numero di file di test da generare |
| `--files-per-directory n` | Numero medio di file per directory |
| `--min-file-size grösse` | Dimensione minima dei file generati (sono ammessi suffissi come K, M, G) |
| `--max-file-size grösse` | Dimensione massima dei file generati |
| `--progress` | Indicatore di avanzamento continuo durante il trasferimento |
| `--stats dauer` | Intervallo di emissione delle statistiche di trasferimento |
| `--log-file datei` | Scrive il log nel file specificato |
| `--log-level stufe` | Livello di dettaglio del log: DEBUG, INFO, NOTICE o ERROR |
| `--one-way` | Con `rclone check` verifica solo che tutti i file sorgente siano presenti e identici nella destinazione; i file aggiuntivi nella destinazione non sono considerati errori |
| `--download` | Durante il confronto scarica effettivamente i dati, anziché confrontare soltanto gli hash |
| `--vfs-cache-mode modus` | Strategia di cache del layer VFS; `full` memorizza localmente nella cache gli accessi in lettura e scrittura |
| `--cache-dir verzeichnis` | Directory per la cache locale |
| `--vfs-cache-max-size grösse` | Limite morbido per la dimensione totale della cache VFS |
| `--vfs-cache-poll-interval dauer` | Intervallo con cui Rclone controlla e pulisce la cache |
| `--vfs-write-back dauer` | Tempo di attesa dopo la chiusura di un file prima che inizi il caricamento nel remote |
| `--vfs-read-ahead grösse` | Quantità aggiuntiva di dati letta in anticipo oltre la posizione richiesta con `full` |
| `--poll-interval dauer` | Intervallo con cui Rclone interroga il remote per rilevare modifiche (solo per backend con supporto polling) |
| `--dir-cache-time dauer` | Durata di validità degli elenchi di directory memorizzati nella cache |
| `--allow-other` | Consente l'accesso al mount FUSE a utenti diversi da quello che lo ha montato |

</details>

Gli elenchi completi sono disponibili nella documentazione di Rclone, in particolare in [rclone mount](https://rclone.org/commands/rclone_mount/) e nella panoramica delle [flag globali](https://rclone.org/flags/).

## Definire anzitutto l'obiettivo

Cold Storage non significa automaticamente la stessa cosa per ogni applicazione. Un media server legge in genere file grandi in modo sequenziale. Una gestione fotografica carica molti piccoli dati di anteprima e salta tra posizioni diverse. Un archivio di documenti apre file relativamente piccoli, ma spesso una sola volta.

Prima del test, annotate le caratteristiche più importanti del vostro archivio reale:

- dimensione tipica dei file e file più grande presente
- numero di file per directory
- lettura completa oppure accessi casuali a singole aree
- rapporto tra accessi in lettura e scrittura
- numero di utenti o processi simultanei
- modifiche effettuate direttamente nel remote al di fuori del mount
- tempo di attesa accettabile per un Cold Read
- spazio massimo disponibile per la cache locale

Solo da questi elementi derivano criteri di successo sensati. Aprire un singolo file in 1,2 secondi può essere del tutto accettabile per un archivio e inutilizzabile per un'applicazione interattiva.

## Generare un set di test riproducibile

Rclone dispone già di uno strumento adatto. `rclone test makefiles` genera ogni volta lo stesso albero di file con un seed fisso:

```bash
rclone test makefiles ./testdata \
  --seed 42 \
  --files 250 \
  --files-per-directory 25 \
  --min-file-size 16K \
  --max-file-size 32M
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `./testdata` | Directory di destinazione in cui viene creato l'albero di test |
| `--seed 42` | Valore iniziale fisso del generatore casuale; ogni esecuzione genera lo stesso set |
| `--files 250` | 250 file di test complessivi |
| `--files-per-directory 25` | In media 25 file per directory |
| `--min-file-size 16K` | File più piccolo: 16 KiB |
| `--max-file-size 32M` | File più grande: 32 MiB |

</details>

Adattate numero e dimensioni al vostro set reale. Non testate solo file medi. Alcuni file molto piccoli mostrano quanto sono costosi gli accessi ai metadati; alcuni file grandi rendono visibili throughput, read-ahead e comportamento della cache.

Aggiungete inoltre nomi di file che nella pratica possono causare problemi:

```bash
mkdir -p "testdata/Sonderfälle/Unterordner"
printf 'Leerzeichen\n' > "testdata/Sonderfälle/Datei mit Leerzeichen.txt"
printf 'Umlaute\n' > "testdata/Sonderfälle/Grösse und Änderung.txt"
printf 'Grossschreibung\n' > "testdata/Sonderfälle/Test.txt"
printf 'Kleinschreibung\n' > "testdata/Sonderfälle/test.txt"
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `mkdir -p` | Crea anche le directory superiori mancanti e non segnala errori se la directory esiste |
| `printf '…\n' > datei` | Scrive il testo specificato come contenuto del file; la redirezione crea il file con il nome problematico |

</details>

L'ultimo test è particolarmente importante se il file system locale e il backend cloud trattano maiuscole e minuscole in modo diverso.

Se il vostro servizio accetta solo formati specifici, file binari qualsiasi non sono sufficienti. Generate allora anche file sintetici esattamente in tali formati. Per Paperless-ngx erano PDF con un vero livello di testo, affinché il test non misurasse involontariamente le prestazioni OCR anziché il percorso di archiviazione. Per una gestione fotografica, il set dovrebbe includere immagini di dimensioni e formati diversi; per un media server, file brevi con codec diversi.

## Una misurazione di riferimento senza mount

Prima che entrino in gioco FUSE e la cache VFS, misurate direttamente il backend. Copiate il set nel remote di test con Rclone e salvate un log dettagliato:

```bash
rclone copy ./testdata remote:cold-storage-test \
  --progress \
  --stats 5s \
  --log-file rclone-copy.log \
  --log-level INFO
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `./testdata` | Sorgente della copia: il set di test generato localmente |
| `remote:cold-storage-test` | Destinazione: percorso nel remote configurato |
| `--progress` | Indicatore di avanzamento continuo nel terminale |
| `--stats 5s` | Statistiche di trasferimento ogni 5 secondi anziché all'intervallo predefinito |
| `--log-file rclone-copy.log` | Log completo in un file per l'analisi successiva |
| `--log-level INFO` | Registra trasferimenti, retry ed errori, senza il livello DEBUG |

</details>

Verificate quindi che sorgente e destinazione corrispondano:

```bash
rclone check ./testdata remote:cold-storage-test \
  --one-way \
  --download
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `./testdata` | Riferimento: il set originale locale |
| `remote:cold-storage-test` | Elemento da verificare: il set appena caricato nel remote |
| `--one-way` | Verifica solo che tutti i file sorgente siano presenti e identici nella destinazione; i file aggiuntivi nella destinazione non vengono segnalati |
| `--download` | Scarica i dati e confronta i contenuti, anziché affidarsi agli hash |

</details>

`--download` è decisivo in questo caso, poiché alcuni backend non forniscono hash adeguati. Il confronto richiede più tempo, ma offre una base utile per il successivo test di integrità.

Annotate tempo di upload, velocità di trasferimento, numero di retry ed errori API. Se già l'accesso diretto è instabile, il mount non può risolverlo.

## Separare il mount di test dalla cache di produzione

Per la misurazione utilizzate un punto di mount e una directory cache separati:

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
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `remote:cold-storage-test` | Remote con il percorso da montare |
| `/mnt/rclone-test` | Punto di mount sul sistema di test |
| `--vfs-cache-mode full` | Memorizza completamente nella cache locale gli accessi in lettura e scrittura |
| `--cache-dir /var/cache/rclone-test` | Directory cache dedicata, separata dalla cache di produzione |
| `--vfs-cache-max-size 10G` | Limite morbido di 10 GiB per la cache VFS |
| `--vfs-cache-poll-interval 1m` | Controllo e pulizia della cache ogni minuto |
| `--allow-other` | Possono accedere anche utenti diversi da quello che ha montato; necessario per servizi e container |
| `--log-file /var/log/rclone-test.log` | Log in un file per tracciare gli accessi durante i test |
| `--log-level INFO` | Livello di dettaglio medio del log |

</details>

I valori sono un esempio e non una raccomandazione generale. La separazione è decisiva: una cache di test vuota rende i Cold Read riproducibili senza dover eliminare file da una cache di produzione in uso.

`--vfs-cache-mode full` è solitamente la modalità di test più istruttiva per le applicazioni. Rclone memorizza localmente nella cache gli accessi in lettura e scrittura e può rappresentare meglio gli accessi ai file che non sarebbero possibili con un puro object storage. Questa compatibilità aggiuntiva richiede spazio locale.

## Verificare sempre dal punto di vista del servizio reale

Un mount può funzionare per il vostro utente ed essere comunque inutilizzabile per il servizio. Cause frequenti sono un ID utente diverso, l'assenza di `--allow-other`, limiti dei container oppure una propagazione del mount errata.

Eseguite quindi almeno un accesso in lettura completo con la stessa identità con cui verrà eseguita in seguito l'applicazione:

```bash
sudo -u <service-user> sha256sum /mnt/rclone-test/pfad/zur/datei
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-u <service-user>` | Esegue il comando come l'utente indicato, non come root |
| `/mnt/rclone-test/pfad/zur/datei` | File da leggere; `sha256sum` forza una lettura completa |

</details>

Se il servizio viene eseguito in Docker, il test deve avvenire nel container:

```bash
docker exec --user <uid>:<gid> <app-container> \
  sha256sum /pfad/im/container/datei
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `--user <uid>:<gid>` | Esegue il comando nel container con questo ID utente e gruppo, indipendentemente dall'utente predefinito dell'immagine |
| `<app-container>` | Nome o ID del container dell'applicazione in esecuzione |
| `sha256sum /pfad/im/container/datei` | Comando da eseguire; il percorso è il mount dal punto di vista del container |

</details>

Meglio ancora è un test reale dell'applicazione. Aprite il file tramite l'interfaccia web o l'API del servizio. Solo così noterete se, per esempio, l'applicazione avvia più letture parallele, salta alla fine del file o richiede metadati aggiuntivi.

## Misurare separatamente Cold Read e Warm Read

Con `--vfs-cache-mode full` ci sono tre livelli tra l'applicazione e il cloud:

| Livello | Cosa contiene |
|---|---|
| Remote | Il file completo nel servizio cloud |
| Cache VFS | Parti memorizzate localmente di file già letti |
| Linux page cache | Dati utilizzati di recente nella RAM |

Per un Cold Read scegliete un file il cui contenuto non sia mai stato letto tramite il mount di test. Nel Warm Read eseguito subito dopo, il file si trova nella cache VFS e nella maggior parte dei casi anche nella RAM.

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
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `date +%s%3N` | Timestamp in millisecondi: secondi Unix seguiti direttamente dalle prime tre cifre della parte in nanosecondi (GNU date) |
| `cat "$file" > /dev/null` | Legge completamente il file e scarta l'output; viene misurato solo il tempo di lettura |
| `"$1"`, `"$2"` | Argomenti della funzione shell: percorso del file ed etichetta della riga di misurazione |

</details>

Non misurate un solo file. Utilizzate almeno dieci file di dimensioni diverse mai letti in precedenza e annotate mediana, valore più lento e dimensione del file. Un singolo valore migliore non è una base decisionale.

Un Warm Read non è un puro test del disco, poiché il kernel può mantenere parti del file nella RAM. Per la maggior parte degli scenari Cold Storage questo non è un problema. Conta ciò che un utente sperimenta alla prima apertura e a quelle successive. Se volete valutare separatamente RAM e disco locale, dovete controllare e svuotare in modo dimostrabile anche la page cache.

## Non testare solo letture complete

`cat` legge un file dall'inizio alla fine. Molte applicazioni si comportano diversamente:

- Un videoplayer legge inizialmente header e indice, in seguito salta a un'altra posizione e continua poi sequenzialmente.
- Una gestione immagini legge i metadati e genera quindi una miniatura.
- Un programma di archiviazione può leggere prima la fine del file.
- Più worker possono accedere contemporaneamente a file diversi.

Testate questi flussi con l'applicazione effettiva. Osservate in parallelo il log Rclone e la cache. Con file grandi è interessante quanto Rclone memorizzi realmente in locale e se `--vfs-read-ahead` si adatti al modello di accesso.

Inoltre, un mount Rclone non è un luogo sensato per archiviare database o altri file che richiedono locking affidabile e modifiche frequenti all'interno dello stesso file. Il layer VFS compensa le differenze tra file system e object storage, ma non trasforma il backend in un file system locale.

## Verificare separatamente il percorso di scrittura

Se il vostro servizio legge soltanto, montate il remote in sola lettura quando possibile. Se deve scrivere, testate separatamente creazione, sovrascrittura, rinomina ed eliminazione.

Un file scritto non appare necessariamente subito nel remote. Con la cache VFS attiva, l'upload inizia soltanto dopo la chiusura del file e la scadenza di `--vfs-write-back`. Verificate pertanto entrambi gli stati:

1. L'applicazione ha chiuso correttamente il file.
2. Il file è successivamente leggibile nel remote tramite un accesso Rclone diretto.

```bash
printf 'writeback-test\n' > /mnt/rclone-test/writeback-test.txt

# Dopo la scadenza di --vfs-write-back:
rclone cat remote:cold-storage-test/writeback-test.txt
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `/mnt/rclone-test/writeback-test.txt` | File di destinazione nel mount; la redirezione scrive tramite la cache VFS |
| `remote:cold-storage-test/writeback-test.txt` | Accesso diretto senza passare dal mount: `rclone cat` legge il file dal remote e lo emette su stdout |

</details>

Ripetete il test con un file grande e terminate Rclone mentre l'upload è ancora in corso. Riavviate quindi usando la stessa directory cache e controllate se l'upload riprende. Proprio questa finestra temporale determina quanti dati sono a rischio in caso di guasto del server.

Testate anche rinomina ed eliminazione. Molti backend cloud mappano queste operazioni diversamente da un file system locale. Non è rilevante solo se il comando termina con successo, ma anche quando la modifica diventa visibile tramite un accesso diretto al remote e per altri client.

## Testare le modifiche esterne al mount

I file possono essere modificati tramite l'interfaccia web del provider, un secondo processo Rclone o un altro server. Il mount non sempre rileva immediatamente tali modifiche, perché le informazioni delle directory vengono memorizzate nella cache.

Create quindi un file direttamente nel remote con una seconda chiamata Rclone:

```bash
printf 'external-change\n' > external-change.txt
rclone copyto external-change.txt \
  remote:cold-storage-test/external-change.txt
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `external-change.txt` | Sorgente: il file generato localmente |
| `remote:cold-storage-test/external-change.txt` | Destinazione con nome file esatto; `copyto` copia un singolo file con esattamente questo nome, anziché copiarlo in una directory come `copy` |

</details>

Misurate quando il file appare nel mount. Ripetete il test per modifica ed eliminazione. Il risultato dipende dal backend, dal suo supporto al polling e da `--poll-interval` e `--dir-cache-time`. Se l'applicazione deve vedere subito le modifiche correnti, questo comportamento deve essere esplicitamente incluso nei criteri di accettazione.

Con l'interfaccia Remote Control attivata, potete invalidare in modo mirato la cache delle directory:

```bash
rclone rc vfs/forget
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `vfs/forget` | Comando Remote Control da eseguire: invalida l'albero di directory VFS memorizzato nella cache, così che il successivo accesso consulti nuovamente il remote |

</details>

Ciò è utile per un test manuale, ma non sostituisce una strategia operativa adeguata.

## Mettere la cache sotto pressione

Una cache quasi vuota è il caso più semplice. In un secondo ciclo di test, impostate intenzionalmente `--vfs-cache-max-size` a un valore ridotto e leggete più dati di quanti ve ne possano entrare.

```bash
du -sh /var/cache/rclone-test/vfs
du -sh --apparent-size /var/cache/rclone-test/vfs
find /var/cache/rclone-test/vfs -type f | wc -l
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `du -s` | Riassume l'utilizzo dello spazio in una riga totale, invece di elencare ogni sottodirectory |
| `du -h` | Output in unità leggibili (K, M, G) |
| `du --apparent-size` | Mostra la dimensione logica dei file anziché lo spazio di archiviazione effettivamente occupato |
| `find … -type f` | Considera soltanto file regolari, non directory |
| `wc -l` | Conta le righe dell'output, quindi in questo caso il numero di file in cache |

</details>

Le due dimensioni possono differire notevolmente. In modalità Full, Rclone usa file sparse: un file mostra la sua intera dimensione logica, anche se soltanto le aree lette occupano spazio locale.

Inoltre, il limite della cache è morbido. Rclone lo controlla con la frequenza di `--vfs-cache-poll-interval`, e i file aperti non possono essere rimossi. La cache può quindi superare temporaneamente il limite. Dopo la chiusura dei file e il successivo ciclo di pulizia dovrebbe tuttavia ridursi nuovamente.

Registrate il valore di picco, il valore dopo la pulizia e il tempo necessario. In questo modo è possibile dimensionare ragionevolmente lo spazio locale richiesto.

## Simulare due guasti diversi

Un cloud irraggiungibile e un processo Rclone arrestato in modo anomalo sono due errori diversi:

| Guasto | Cosa verificate |
|---|---|
| Backend o rete non raggiungibili, Rclone continua a funzionare | Comportamento con retry, timeout e file già in cache |
| Processo Rclone terminato | Comportamento del mount FUSE e ripristino del punto di mount |

Simulate entrambi solo nell'ambiente di test. Per il secondo caso, potete terminare forzatamente un container Rclone:

```bash
docker kill --signal KILL <rclone-container>
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `--signal KILL` | Invia SIGKILL (con `docker kill` è comunque il segnale predefinito, qui specificato per chiarezza); il processo non ha possibilità di effettuare la pulizia |
| `<rclone-container>` | Nome o ID del container Rclone |

</details>

Durante il guasto, verificate l'applicazione e non solo il punto di mount:

- Quali funzioni rimangono disponibili?
- Quanto attende un accesso prima che appaia un errore?
- I file già completamente in cache restano raggiungibili?
- L'applicazione interrompe le nuove operazioni di scrittura?
- Compare un messaggio di errore comprensibile o soltanto un processo bloccato?
- Il monitoraggio si attiva?

Un servizio di scrittura non deve scrivere inosservato nella directory locale sottostante quando il mount non è disponibile. Dopo il ritorno del mount, questi file sarebbero nascosti. Una semplice protezione prima di ogni job di scrittura è:

```bash
mountpoint -q /mnt/rclone-test || exit 1
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-q` | Nessun output; il risultato è disponibile solo nel codice di uscita |
| `/mnt/rclone-test` | Percorso da verificare; codice di uscita 0 solo se vi è effettivamente attivo un mount |
| `\|\| exit 1` | Interrompe lo script se il percorso non è un punto di mount |

</details>

Dopo il riavvio di Rclone, controllate il mount sull'host e da ogni container che lo utilizza. Un mount ricreato raggiunge un container già in esecuzione solo con un'adeguata propagazione del mount. Per Docker, sul lato utilizzatore è generalmente necessario `rslave`. I dettagli sono disponibili nell'articolo [Gestire in modo affidabile i mount Rclone in Docker](/blog/rclone-mount-in-docker-container).

## Un esempio concreto da Paperless-ngx

Per il mio test con Paperless ho generato 40 PDF per un totale di 13,9 MB. Un documento mai aperto prima richiedeva circa 1,8 secondi, mentre un accesso ripetuto immediatamente richiedeva da 19 a 24 millisecondi. Una cache VFS limitata a 4 MB è salita temporaneamente a 12,7 MiB ed è stata nuovamente pulita al ciclo successivo.

Mentre il remote non era raggiungibile, l'elenco dei documenti, la ricerca full-text e le anteprime continuavano a funzionare perché tali dati erano locali. Non era possibile aprire soltanto l'originale. Dopo la ricreazione del mount, il container Paperless in esecuzione poteva di nuovo accedere ai file senza dover essere riavviato.

Questi numeri non sono un benchmark per Rclone o Proton Drive. È interessante il comportamento: l'Hot Storage è rimasto disponibile localmente, i Cold Read erano più lenti ma prevedibili e il servizio si è ripreso dopo il guasto.

## Cosa includere nel protocollo di test

Un risultato che possa essere verificato in seguito contiene almeno:

- versione Rclone e backend utilizzato
- sistema operativo, variante FUSE e file system della directory cache
- comando di mount completo senza credenziali
- numero, distribuzione delle dimensioni e struttura dei file di test
- valori di Cold Read e Warm Read per più file
- durata della scrittura fino alla visibilità nel remote
- valore di picco della cache e durata della pulizia
- risultato di `rclone check --download`
- comportamento in caso di guasto del backend e terminazione del processo Rclone
- tempo di ripristino dal punto di vista dell'applicazione
- retry, timeout, limitazioni e errori di autenticazione dal log

Definite in anticipo un valore limite per ogni punto. Il test si conclude così con una decisione e non solo con una raccolta di numeri interessanti.

## Quando la configurazione è pronta

Un mount Cold Storage è pronto all'uso se potete rispondere sì a queste domande:

- I Cold Read sono abbastanza veloci per il servizio previsto?
- La cache accelera gli accessi ripetuti come previsto?
- Il fabbisogno di spazio locale rimane controllabile anche sotto carico?
- Tutti i file corrispondono dopo un download completo?
- Tutte le operazioni sui file necessarie funzionano con il backend scelto?
- L'applicazione si comporta in modo controllato in caso di guasto del cloud?
- Le operazioni di scrittura vengono interrotte in sicurezza quando il mount manca?
- Un mount ricreato raggiunge tutti i consumer in esecuzione?
- Il monitoraggio segnala il guasto prima che lo faccia un utente?

Se manca una risposta, sapete almeno esattamente su cosa dovete continuare a lavorare. È molto più utile di un mount che sembrava valido al primo `ls` e mostra i propri limiti solo durante l'esercizio.

## Cronologia delle modifiche

- **25 ottobre 2026:** Corretto: con docker kill SIGKILL è già il segnale predefinito; l'opzione --signal KILL serve soltanto a chiarirlo.

## Fonti

1.  [Rclone test makefiles](https://rclone.org/commands/rclone_test_makefiles/): file di test e strutture di directory riproducibili con dimensioni configurabili.

2.  [Rclone mount](https://rclone.org/commands/rclone_mount/): modalità della cache VFS, writeback, file sparse, limiti della cache e cache delle directory.

3.  [Rclone check](https://rclone.org/commands/rclone_check/): confronto tra sorgente e destinazione, inclusa la verifica completa con `--download`.

4.  [Rclone Remote Control](https://rclone.org/rc/): invalidazione mirata della cache delle directory VFS con `vfs/forget`.

5.  [Rclone Global Flags](https://rclone.org/flags/): riferimento completo delle opzioni globali, incluse registrazione, statistiche e parametri VFS.
