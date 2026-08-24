---
title: "Testare il Cold Storage con Rclone: un piano di test pratico"
navTitle: "Testare Rclone"
description: "Prima che un servizio legga i propri file dal cloud tramite un mount Rclone, dovresti verificare più del semplice accesso alle directory. Questo piano di test copre cold read, warm read, operazioni di scrittura, comportamento della cache, integrità dei file e guasti."
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
url: "https://rafaelpfister.ch/it/blog/testare-il-cold-storage-con-rclone-un-piano-di-test-pratico"
translationId: article-8592f808b2e93cd4
translationModel: gpt-5.6-terra
translatedAt: 2026-07-28T13:31:55.763Z
translationReview: automatic
translationSourceHash: 4dd3058563b8e3853528cbd3cb5b216cc840923ceee9250055c3000c296232b9
---

Un mount Rclone si configura rapidamente. Il remote appare come una directory, `ls` mostra i file e il primo test funzionale è superato. Tuttavia, questo dice ancora poco sul funzionamento in produzione.

Non appena un servizio accede al mount, sorgono altre domande: quanto dura il primo accesso a un file? Quali accessi vengono gestiti dalla cache locale? Cosa accade a un file non ancora caricato se Rclone va in crash? Un container in esecuzione vede di nuovo il mount ricostruito? E come reagisce il servizio se il cloud non è temporaneamente raggiungibile?

Questo articolo fornisce un piano di test generale. Puoi usarlo per un archivio di documenti, un media server, una gestione di foto o qualsiasi altro servizio che recupera file usati raramente tramite Rclone da un Cold Storage.

## Definire prima cosa vuoi ottenere

Cold Storage non significa automaticamente la stessa cosa per ogni applicazione. Un media server legge di solito file di grandi dimensioni in modo sequenziale. Una gestione di foto carica molte piccole anteprime e salta a diverse posizioni. Un archivio di documenti apre file relativamente piccoli, ma spesso una sola volta.

Prima del test, annota le caratteristiche principali del tuo archivio reale:

- dimensione tipica dei file e file più grande presente
- numero di file per directory
- lettura completa o accessi casuali a singole aree
- rapporto tra accessi in lettura e in scrittura
- numero di utenti o processi simultanei
- modifiche eseguite direttamente nel remote al di fuori del mount
- tempo di attesa accettabile per un cold read
- spazio massimo disponibile per la cache locale

Solo da questi elementi derivano criteri di successo sensati. Aprire un singolo file in 1,2 secondi può essere del tutto accettabile per un archivio e inutilizzabile per un'applicazione interattiva.

## Creare un set di dati di test riproducibile

Rclone include già uno strumento adatto. `rclone test makefiles` genera ogni volta lo stesso albero di file con un seed fisso:

```bash
rclone test makefiles ./testdata \
  --seed 42 \
  --files 250 \
  --files-per-directory 25 \
  --min-file-size 16K \
  --max-file-size 32M
```

Adatta quantità e dimensioni al tuo archivio reale. Non testare solo file di dimensione media. Alcuni file molto piccoli mostrano quanto siano costosi gli accessi ai metadati; alcuni file grandi rendono visibili throughput, read-ahead e comportamento della cache.

Aggiungi inoltre nomi di file che potrebbero causare problemi nella pratica:

```bash
mkdir -p "testdata/Casi speciali/Sottocartella"
printf 'Spazi\n' > "testdata/Casi speciali/File con spazi.txt"
printf 'Accenti\n' > "testdata/Casi speciali/Dimensione e modifica.txt"
printf 'Maiuscole\n' > "testdata/Casi speciali/Test.txt"
printf 'Minuscole\n' > "testdata/Casi speciali/test.txt"
```

L'ultimo test è particolarmente importante quando il file system locale e il backend cloud gestiscono in modo diverso maiuscole e minuscole.

Se il tuo servizio accetta solo determinati formati, file binari qualsiasi non sono sufficienti. Crea quindi anche file sintetici esattamente in questi formati. Con Paperless-ngx si trattava di PDF con un vero livello di testo, affinché il test non misurasse involontariamente le prestazioni OCR anziché il percorso di archiviazione. Per una gestione di foto, il set dovrebbe includere immagini di diverse dimensioni e formati; per un media server, file brevi con codec diversi.

## Una misurazione di riferimento senza mount

Prima che entrino in gioco FUSE e la cache VFS, dovresti misurare direttamente il backend. Copia il set nel remote di test con Rclone e salva un log dettagliato:

```bash
rclone copy ./testdata remote:cold-storage-test \
  --progress \
  --stats 5s \
  --log-file rclone-copy.log \
  --log-level INFO
```

Verifica poi che origine e destinazione coincidano:

```bash
rclone check ./testdata remote:cold-storage-test \
  --one-way \
  --download
```

Con `--download` Rclone legge effettivamente i dati e li confronta, anche se il backend non fornisce hash adeguati. Richiede più tempo, ma offre una base utile per il successivo test di integrità.

Registra tempo di upload, velocità di trasferimento, numero di retry ed errori API. Se già l'accesso diretto è instabile, il mount non può risolvere il problema.

## Separare il mount di test dalla cache di produzione

Per la misurazione utilizza un punto di mount e una directory cache dedicati:

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

I valori sono un esempio e non una raccomandazione generale. La separazione è ciò che conta: una cache di test vuota rende i cold read riproducibili, senza dover eliminare file da una cache di produzione in uso.

`--vfs-cache-mode full` è solitamente la modalità di test più significativa per le applicazioni. Rclone memorizza localmente gli accessi in lettura e scrittura e può rappresentare meglio accessi ai file che non sarebbero possibili con un puro object storage. La compatibilità aggiuntiva richiede spazio locale.

## Verificare sempre dal punto di vista del servizio reale

Un mount può funzionare per il tuo utente e risultare comunque inutilizzabile per il servizio. Cause frequenti sono un ID utente diverso, l'assenza di `--allow-other`, limiti dei container o una propagazione del mount errata.

Esegui quindi almeno un accesso completo in lettura con la stessa identità con cui verrà eseguita l'applicazione:

```bash
sudo -u <utente-servizio> sha256sum /mnt/rclone-test/percorso/al/file
```

Se il servizio viene eseguito in Docker, il test va effettuato nel container:

```bash
docker exec --user <uid>:<gid> <app-container> \
  sha256sum /percorso/nel/container/file
```

Ancora meglio è un test con l'applicazione reale. Apri il file tramite l'interfaccia web o l'API del servizio. Solo così noterai, ad esempio, se l'applicazione avvia più letture parallele, salta alla fine del file o si aspetta metadati aggiuntivi.

## Misurare separatamente cold read e warm read

Con `--vfs-cache-mode full` tra applicazione e cloud ci sono tre livelli:

| Livello | Contenuto |
|---|---|
| Remote | il file completo nel servizio cloud |
| Cache VFS | aree memorizzate localmente di file già letti |
| Linux Page Cache | dati usati di recente nella RAM |

Per un cold read, scegli un file il cui contenuto non sia mai stato letto tramite il mount di test. Nel warm read immediatamente successivo, il file si trova nella cache VFS e generalmente anche nella RAM.

```bash
measure_read() {
  file="$1"
  label="$2"
  start=$(date +%s%3N)
  cat "$file" > /dev/null
  end=$(date +%s%3N)
  printf '%s: %s ms\n' "$label" "$((end - start))"
}

measure_read "/mnt/rclone-test/file-grande.bin" "Lettura a freddo"
measure_read "/mnt/rclone-test/file-grande.bin" "Lettura a caldo"
```

Non misurare un solo file. Utilizza almeno dieci file di dimensioni diverse mai letti prima e annota mediana, valore più lento e dimensione del file. Un singolo risultato migliore non è una base decisionale.

Un warm read non è un puro test del disco, perché il kernel può mantenere parti del file nella RAM. Per la maggior parte degli scenari Cold Storage non è un problema. Ciò che conta è l'esperienza dell'utente alla prima apertura e alle aperture successive. Se vuoi valutare separatamente RAM e disco locale, devi controllare e svuotare in modo verificabile anche la Page Cache.

## Non testare solo letture complete

`cat` legge un file dall'inizio alla fine. Molte applicazioni si comportano diversamente:

- Un lettore video legge inizialmente header e indice, in seguito salta a un'altra posizione e poi continua a caricare in modo sequenziale.
- Una gestione di immagini legge i metadati e crea poi una miniatura.
- Un programma di archiviazione può leggere prima la fine del file.
- Più worker possono accedere contemporaneamente a file diversi.

Testa questi flussi con l'applicazione reale. Osserva parallelamente il log di Rclone e la cache. Con file grandi è interessante quanto Rclone memorizzi davvero in locale e se `--vfs-read-ahead` si adatti al modello di accesso.

Inoltre, un mount Rclone non è un luogo di archiviazione sensato per database o altri file che richiedono locking affidabile e modifiche frequenti all'interno dello stesso file. Il livello VFS compensa le differenze tra file system e object storage, ma non trasforma il backend in un file system locale.

## Testare separatamente il percorso di scrittura

Se il tuo servizio effettua solo letture, monta il remote in sola lettura se possibile. Se deve scrivere, testa separatamente creazione, sovrascrittura, rinomina ed eliminazione.

Un file scritto non appare necessariamente subito nel remote. Con la cache VFS attiva, l'upload inizia solo dopo la chiusura del file e la scadenza di `--vfs-write-back`. Verifica quindi entrambi gli stati:

1. L'applicazione ha chiuso correttamente il file.
2. Il file è poi leggibile nel remote tramite un accesso diretto con Rclone.

```bash
printf 'writeback-test\n' > /mnt/rclone-test/writeback-test.txt

# Dopo la scadenza di --vfs-write-back:
rclone cat remote:cold-storage-test/writeback-test.txt
```

Ripeti il test con un file grande e termina Rclone durante l'upload ancora in corso. Riavvia quindi con la stessa directory cache e controlla se l'upload riprende. Proprio questa finestra temporale determina quanti dati sono a rischio in caso di guasto del server.

Testa anche rinomina ed eliminazione. Molti backend cloud rappresentano queste operazioni in modo diverso rispetto a un file system locale. Non conta solo che il comando termini con successo, ma anche quando la modifica diventa visibile tramite un accesso diretto al remote e per altri client.

## Testare le modifiche al di fuori del mount

I file possono essere modificati tramite l'interfaccia web del provider, un secondo processo Rclone o un altro server. Il mount non vede sempre subito queste modifiche, perché le informazioni delle directory vengono memorizzate nella cache.

Crea quindi un file direttamente nel remote con una seconda chiamata a Rclone:

```bash
printf 'external-change\n' > external-change.txt
rclone copyto external-change.txt \
  remote:cold-storage-test/external-change.txt
```

Misura quando il file appare nel mount. Ripeti il test per modifica ed eliminazione. Il risultato dipende dal backend, dal suo supporto del polling e da `--poll-interval` e `--dir-cache-time`. Se l'applicazione deve vedere immediatamente le modifiche correnti, questo comportamento deve rientrare esplicitamente nei criteri di accettazione.

Con l'interfaccia Remote Control attivata, puoi invalidare selettivamente la cache delle directory:

```bash
rclone rc vfs/forget
```

È utile per un test manuale, ma non sostituisce una strategia operativa adeguata.

## Mettere la cache sotto pressione

Una cache quasi vuota è il caso più semplice. In un secondo ciclo di test, imposta `--vfs-cache-max-size` intenzionalmente basso e leggi più dati di quanti ne possano entrare.

```bash
du -sh /var/cache/rclone-test/vfs
du -sh --apparent-size /var/cache/rclone-test/vfs
find /var/cache/rclone-test/vfs -type f | wc -l
```

Le due dimensioni possono differire notevolmente. In modalità Full, Rclone utilizza Sparse Files: un file mostra la sua dimensione logica completa, anche se solo le aree lette occupano spazio locale.

Inoltre, il limite della cache è flessibile. Rclone lo controlla al ritmo di `--vfs-cache-poll-interval`, e i file aperti non possono essere rimossi. La cache può quindi superare brevemente il limite. Dopo la chiusura dei file e il successivo ciclo di pulizia, dovrebbe però diminuire di nuovo.

Registra il valore massimo, il valore dopo la pulizia e il tempo necessario. In questo modo puoi dimensionare ragionevolmente lo spazio locale richiesto.

## Simulare due guasti diversi

Un cloud non raggiungibile e un processo Rclone arrestato sono due errori diversi:

| Guasto | Cosa verifichi |
|---|---|
| Backend o rete non raggiungibili, Rclone continua a funzionare | Comportamento con retry, timeout e file già memorizzati nella cache |
| Processo Rclone terminato | Comportamento del mount FUSE e ripristino del punto di mount |

Simula entrambi solo nell'ambiente di test. Per il secondo caso puoi terminare forzatamente un container Rclone:

```bash
docker kill --signal KILL <rclone-container>
```

Durante il guasto, verifica l'applicazione e non solo il punto di mount:

- Quali funzioni restano disponibili?
- Quanto attende un accesso prima che appaia un errore?
- I file già completamente memorizzati nella cache sono ancora raggiungibili?
- L'applicazione interrompe le nuove operazioni di scrittura?
- Compare un messaggio di errore comprensibile o solo un processo bloccato?
- Il monitoraggio si attiva?

Un servizio di scrittura non deve scrivere inosservato nella directory locale sottostante quando il mount è assente. Dopo il ritorno del mount, questi file verrebbero nascosti. Una semplice protezione prima di ogni job di scrittura è:

```bash
mountpoint -q /mnt/rclone-test || exit 1
```

Dopo il riavvio di Rclone, verifica il mount sull'host e da ogni container che lo utilizza. Un mount ricostruito raggiunge un container già in esecuzione solo con una propagazione del mount adeguata. Per Docker, di solito è necessario `rslave` sul lato che utilizza il mount. I dettagli sono disponibili nell'articolo [Gestire in modo affidabile i mount Rclone in Docker](/blog/rclone-mount-in-docker-container).

## Un esempio concreto con Paperless-ngx

Per il mio test con Paperless ho creato 40 PDF per un totale di 13,9 MB. Un documento mai aperto prima richiedeva circa 1,8 secondi, mentre un accesso ripetuto immediatamente richiedeva da 19 a 24 millisecondi. Una cache VFS limitata a 4 MB è salita temporaneamente a 12,7 MiB ed è stata ripulita al ciclo successivo.

Mentre il remote non era raggiungibile, l'elenco dei documenti, la ricerca full-text e le anteprime continuavano a funzionare perché questi dati erano locali. Solo l'originale non poteva essere aperto. Dopo la ricostruzione del mount, il container Paperless in esecuzione ha potuto accedere di nuovo ai file senza essere riavviato.

Questi valori non sono un benchmark per Rclone o Proton Drive. È interessante il comportamento: l'Hot Storage restava disponibile in locale, i cold read erano più lenti ma prevedibili e il servizio si riprendeva dopo il guasto.

## Cosa includere nel protocollo di test

Un risultato verificabile in seguito contiene almeno:

- versione di Rclone e backend utilizzato
- sistema operativo, variante FUSE e file system della directory cache
- comando di mount completo senza credenziali
- numero, distribuzione delle dimensioni e struttura dei file di test
- valori di cold read e warm read per più file
- durata della scrittura fino alla visibilità nel remote
- valore massimo della cache e durata della pulizia
- risultato di `rclone check --download`
- comportamento in caso di guasto del backend e processo Rclone terminato
- tempo di ripristino dal punto di vista dell'applicazione
- retry, timeout, limitazioni e errori di autenticazione dal log

Definisci in anticipo un valore limite per ogni punto. Così il test termina con una decisione e non solo con una raccolta di numeri interessanti.

## Quando la configurazione è pronta

Un mount Cold Storage è pronto all'uso se puoi rispondere sì a queste domande:

- I cold read sono abbastanza rapidi per il servizio previsto?
- La cache accelera gli accessi ripetuti come previsto?
- Il fabbisogno di spazio locale rimane controllabile anche sotto carico?
- Tutti i file coincidono dopo un download completo?
- Tutte le operazioni sui file richieste funzionano con il backend scelto?
- L'applicazione si comporta in modo controllato durante un guasto del cloud?
- Le operazioni di scrittura vengono interrotte in sicurezza quando il mount è assente?
- Un mount ricostruito raggiunge tutti i consumer in esecuzione?
- Il monitoraggio segnala il guasto prima che lo faccia un utente?

Se manca una risposta, sai almeno esattamente su cosa devi continuare a lavorare. È molto più utile di un mount che sembrava funzionare al primo `ls` e mostra i suoi limiti solo in esercizio.

## Fonti

1.  [Rclone test makefiles](https://rclone.org/commands/rclone_test_makefiles/): file di test e strutture di directory riproducibili con dimensioni configurabili.

2.  [Rclone mount](https://rclone.org/commands/rclone_mount/): modalità della cache VFS, writeback, Sparse Files, limiti della cache e cache delle directory.

3.  [Rclone check](https://rclone.org/commands/rclone_check/): confronto tra origine e destinazione, inclusa la verifica completa con `--download`.

4.  [Rclone Remote Control](https://rclone.org/rc/): invalidazione mirata della cache delle directory VFS con `vfs/forget`.
