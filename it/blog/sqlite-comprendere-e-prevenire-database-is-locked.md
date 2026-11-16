---
title: "SQLite: comprendere e prevenire «database is locked»"
navTitle: "Blocchi SQLite"
description: "Perché SQLite risponde con SQLITE_BUSY e «database is locked», come la modalità WAL, busy_timeout, transazioni brevi e retry evitano i blocchi e perché supporti lenti e carico I/O su un NAS possono innescare il problema. Con un caso pratico di Grafana."
date: "2026-11-12"
kategorie: "Database"
timeToRead: "11 min di lettura"
themen:
  - datenbanken
produkte:
  - "sqlite"
protokolle:
  - "troubleshooting"
  - "container"
slug: "sqlite-comprendere-e-prevenire-database-is-locked"
translationId: "article-2a328dabf9164dae"
translationOf: sqlite-database-locked-wal-io-last
url: https://rafaelpfister.ch/it/blog/sqlite-comprendere-e-prevenire-database-is-locked
translationSourceHash: 121b1c0f6dc58cb2249e9e0276937397b52725621cd1553b455a37069f5958de
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T13:51:26.266Z
translationReview: required
---

SQLite è integrato in molti servizi autogestiti: Grafana, Home Assistant, Vaultwarden, Uptime Kuma, Jellyfin e numerose altre applicazioni memorizzano i propri dati per impostazione predefinita in un singolo file. Funziona finché nel log non compare il messaggio `database is locked`. Indica che un accesso non ha ottenuto un blocco entro il tempo di attesa consentito; il file del database stesso rimane intatto. Che ciò accada raramente o continuamente dipende da tre fattori: la modalità journal, il comportamento dell'applicazione in caso di conflitti e la latenza del supporto di memorizzazione.

## Caso pratico: Grafana su un NAS

Grafana era in esecuzione come container Docker su un NAS Synology, mentre il file SQLite si trovava in un Named Volume. Sullo stesso NAS erano in esecuzione Prometheus, Loki, Paperless-ngx e diversi backup restic. I supporti di memorizzazione erano dischi rigidi con Shingled Magnetic Recording (SMR).

All'inizio, l'endpoint health di Grafana segnalava `"database": "failing"`. Nel log comparivano a brevi intervalli voci di questo tipo:

```text
level=error msg="Failed to save history batch"
  error="database is locked (5) (SQLITE_BUSY)"
```

Il messaggio proviene dalla cronologia degli stati degli alert: per impostazione predefinita, Grafana scrive i cambiamenti di stato delle regole di alert come annotazioni nel proprio database. Con molte regole, ciò genera continuamente piccole transazioni di scrittura. Dopo il riavvio del container, Grafana non si avviava più e veniva riavviato in un ciclo:

```text
Failed to stop all modules ... kv get:
  [sqlstore.max-retries-reached] retry 1: database is locked
```

Il caso è stato risolto con due impostazioni in `docker-compose.yml`: attivare la modalità WAL e fare in modo che Grafana ritenti più volte in caso di blocchi. Il motivo per cui proprio questi due valori funzionano risulta dal funzionamento dei blocchi di SQLite.

## Come SQLite gestisce i blocchi

SQLite non dispone di un processo server che coordini gli accessi. Ogni connessione apre direttamente il file e regola l'accesso condiviso tramite blocchi di file del sistema operativo. Ne deriva la caratteristica più importante: **in ogni momento può scrivere una sola connessione.** Questo vale in tutte le modalità journal.

Nella classica modalità rollback journal (predefinita, `journal_mode=DELETE`), una transazione di scrittura attraversa più livelli di blocco:

| Blocco | Significato |
|---|---|
| `SHARED` | La connessione legge. È consentito un numero qualsiasi di lettori contemporaneamente. |
| `RESERVED` | La connessione vuole scrivere e prepara le modifiche. I lettori possono continuare a leggere, ma non può esserci un secondo scrittore. |
| `PENDING` | La connessione attende la fine delle letture in corso. I nuovi lettori vengono respinti. |
| `EXCLUSIVE` | La connessione scrive nel file del database. Nessun altro può leggere o scrivere. |

Per il commit, lo scrittore necessita quindi del blocco esclusivo, che ottiene solo quando non vi sono più lettori attivi. Una lettura di lunga durata blocca così ogni scrittore e, durante il commit, tutti i lettori sono bloccati.

Se una connessione non ottiene il blocco necessario, SQLite restituisce il codice di errore `SQLITE_BUSY` (5) con il testo `database is locked`. Da distinguere è `SQLITE_LOCKED` (6, `database table is locked`): questo codice riguarda conflitti all'interno della stessa connessione o in modalità cache condivisa e ha altre cause.

È decisivo quanto a lungo SQLite attenda prima di questo messaggio. Senza configurazione SQLite non attende affatto: il valore predefinito di `busy_timeout` è 0 e il messaggio compare subito quando un'altra connessione detiene il blocco. Molte librerie impostano un proprio valore; ad esempio il modulo Python `sqlite3` attende 5 secondi.

## Modalità WAL: separare lettori e scrittori

Nella modalità Write-Ahead Log (`journal_mode=WAL`), SQLite aggiunge le modifiche a un file separato `<name>-wal` invece di scriverle direttamente nel file del database. I lettori vedono lo stato all'inizio della loro transazione di lettura e leggono a tale scopo dal file del database e dal WAL. Ne derivano due vantaggi:

- I lettori non bloccano lo scrittore e lo scrittore non blocca i lettori.
- Un commit consiste nell'aggiungere dati a un file, anziché in più operazioni di scrittura sparse con journal. Per ogni transazione si genera meno I/O e il blocco di scrittura viene mantenuto più brevemente.

La modalità WAL non cambia il fatto che possa essere attivo un solo scrittore alla volta. Riduce però il tempo per cui il blocco viene mantenuto ed elimina i conflitti tra lettura e scrittura. Per un'applicazione come Grafana, che distribuisce dashboard e memorizza stati di alert simultaneamente, è la misura più efficace.

La modalità viene impostata una sola volta e rimane memorizzata nel file:

```sql
PRAGMA journal_mode = WAL;
PRAGMA busy_timeout = 5000;
PRAGMA synchronous = NORMAL;
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `journal_mode = WAL` | Passa a Write-Ahead Log. L'impostazione è persistente; le nuove connessioni aprono automaticamente il file in modalità WAL. |
| `busy_timeout = 5000` | Tempo di attesa in millisecondi durante il quale SQLite tenta ripetutamente di ottenere un blocco occupato. Vale solo per la connessione corrente e deve essere impostato nuovamente a ogni apertura. |
| `synchronous = NORMAL` | In modalità WAL nessun `fsync` per commit, ma solo ai checkpoint. Il database rimane coerente in caso di interruzione dell'alimentazione; le ultime transazioni confermate possono andare perse. Vale per connessione. |

</details>

La modalità WAL ha dei requisiti. Tutti i processi che accedono al file devono essere eseguiti sullo stesso computer, perché si coordinano tramite un file di memoria condivisa `<name>-shm`. La directory necessita di permessi di scrittura per i due file aggiuntivi, anche per i processi che effettuano solo letture. Inoltre, durante il backup i file devono essere considerati insieme (maggiori dettagli più avanti).

## Checkpoint: quando il WAL cresce

Il WAL cresce finché un checkpoint non trasferisce le modifiche nel vero e proprio file del database. SQLite lo attiva automaticamente non appena il WAL contiene circa 1000 pagine (`wal_autocheckpoint`, con pagine di 4 KiB circa 4 MB). Il checkpoint scrive le pagine nelle loro posizioni nel file del database e quindi richiama `fsync`. Su un supporto lento, proprio questo passaggio richiede più tempo.

Un checkpoint può trasferire le pagine solo finché nessun lettore necessita di uno stato precedente. Se è sempre in esecuzione almeno una transazione di lettura, il checkpoint rimane incompleto e il WAL continua a crescere. Un WAL di diverse centinaia di megabyte indica transazioni di lettura di lunga durata o checkpoint che non terminano a causa del carico I/O. È possibile verificarlo così:

```sql
PRAGMA wal_checkpoint(TRUNCATE);
```

L'output è composto da tre numeri: se il checkpoint è stato bloccato (`1` = sì), quante pagine sono presenti nel WAL e quante di esse sono state trasferite. Con `TRUNCATE` il WAL viene poi ridotto a zero byte, purché il checkpoint sia stato completo.

## Busy timeout e retry

Con un busy timeout, SQLite attende in caso di conflitto anziché interrompersi subito. L'errore compare soltanto se il blocco viene mantenuto più a lungo del tempo impostato. Per applicazioni con poche e brevi operazioni di scrittura, sono sufficienti da 5 a 10 secondi.

Esiste un caso in cui il busy timeout non si applica. Se una transazione inizia leggendo (`BEGIN` senza aggiunta, quindi `DEFERRED`) e vuole poi scrivere mentre nel frattempo un altro scrittore ha completato un commit, SQLite restituisce subito `SQLITE_BUSY`. La transazione lavora su uno stato obsoleto e non può diventare una transazione di scrittura; attendere non cambierebbe nulla. Nella modalità rollback esiste una situazione analoga quando due connessioni vogliono passare contemporaneamente da `SHARED` a un blocco di scrittura.

La soluzione consiste nel contrassegnare le transazioni di scrittura come tali fin dall'inizio:

```sql
BEGIN IMMEDIATE;
UPDATE alert_state SET state = 'firing' WHERE id = 42;
COMMIT;
```

`BEGIN IMMEDIATE` richiede il blocco di scrittura all'avvio. Se è occupato, il busy timeout si applica normalmente. In Python è così:

```python
import sqlite3

con = sqlite3.connect("app.db", timeout=10.0,
                      isolation_level=None)
con.execute("PRAGMA journal_mode = WAL")
con.execute("BEGIN IMMEDIATE")
con.execute("UPDATE jobs SET status = ? WHERE id = ?",
            ("done", 42))
con.execute("COMMIT")
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `"app.db"` | Percorso del file del database. |
| `timeout=10.0` | Busy timeout in secondi; corrisponde a `PRAGMA busy_timeout = 10000`. |
| `isolation_level=None` | Disattiva la gestione automatica delle transazioni del modulo, affinché `BEGIN IMMEDIATE` e `COMMIT` possano essere impostati esplicitamente. |

</details>

I retry a livello dell'applicazione integrano il busy timeout. Intercettano `SQLITE_BUSY`, attendono brevemente e rieseguono l'intera transazione. Coprono anche il caso dello stato di lettura obsoleto, poiché la ripetizione inizia con uno stato nuovo. Grafana offre impostazioni proprie a questo scopo (vedere la sezione sulla soluzione).

## Transazioni brevi e un solo scrittore

La durata per cui viene mantenuto un blocco di scrittura è determinata dall'applicazione. Tre regole lo mantengono breve:

- **Nessun lavoro nella transazione che non riguardi il database.** Chiamate HTTP, accessi ai file o calcoli devono avvenire prima di `BEGIN` o dopo `COMMIT`.
- **Raggruppare molte piccole operazioni di scrittura.** 500 commit singoli comportano 500 cicli di blocco e, in modalità rollback, 500 volte più chiamate a `fsync`. Una transazione con 500 righe mantiene il blocco una sola volta e un po' più a lungo.
- **Chiudere i cursori di lettura.** Un risultato di query non recuperato completamente mantiene aperta la transazione di lettura e blocca il checkpoint in modalità WAL.

Dove scrivono più thread o processi, un unico scrittore con coda è la soluzione più affidabile: un thread o processo possiede l'unica connessione in scrittura, tutti gli altri gli inviano i lavori. In questo modo i conflitti di blocco tra scrittori vengono completamente eliminati, mentre i lettori continuano a lavorare in parallelo in modalità WAL.

Più container che aprono lo stesso file SQLite tramite un volume condiviso sono invece una causa frequente di blocchi. Ogni container è uno scrittore separato e nessuno è a conoscenza dell'altro. Se un'applicazione viene scalata orizzontalmente, un server di database come PostgreSQL o MySQL è la scelta appropriata.

## Nessun file system di rete

SQLite fa affidamento sul fatto che i blocchi di file funzionino correttamente e in modo atomico e che un `fsync` porti effettivamente i dati sul supporto di memorizzazione. Con NFS, SMB/CIFS e molti file system FUSE, entrambe le cose non sono garantite. Gli sviluppatori di SQLite sconsigliano espressamente di utilizzare un database da più computer tramite un file system di rete. Le conseguenze vanno da messaggi di blocco permanenti a un database danneggiato, e la modalità WAL non funziona fondamentalmente in questi casi, perché la sincronizzazione della memoria condivisa è possibile solo localmente.

Per Docker ciò significa: un Named Volume o un bind mount su un supporto locale è adatto. Un bind mount su una condivisione SMB o NFS montata non lo è, anche se in condizioni normali sembra funzionare.

## Carico I/O: perché il blocco improvvisamente dura troppo a lungo

Il blocco è solo un sintomo. Diventa un problema quando viene mantenuto più a lungo di quanto consenta il busy timeout, e ciò dipende quasi sempre dalla latenza di scrittura del supporto di memorizzazione.

Ogni commit termina con un `fsync`. Solo quando il supporto di memorizzazione conferma che i dati sono stati scritti in modo permanente, SQLite rilascia il blocco. Su un SSD ciò richiede meno di un millisecondo, su un disco rigido inattivo alcuni millisecondi. Se il supporto deve elaborare contemporaneamente altri lavori, il `fsync` attende nella coda e il blocco rimane attivo per tutto quel tempo. Tutti gli altri scrittori attendono fino alla scadenza del loro busy timeout.

Diversi fattori aumentano questa latenza su un NAS:

- **Dischi rigidi SMR.** Nello Shingled Magnetic Recording le tracce si sovrappongono. Le operazioni di scrittura casuali finiscono prima in un'area cache e in seguito vengono riorganizzate nelle zone sovrapposte. Se l'area cache è piena sotto carico continuo, la latenza di scrittura aumenta da millisecondi a diversi secondi. I database con molte piccole operazioni di scrittura sparse rappresentano il profilo di carico più sfavorevole.
- **Backup paralleli.** restic, Hyper Backup o rsync leggono grandi quantità di dati e competono con ogni `fsync` per gli stessi piatti.
- **Scrubbing e controlli di coerenza RAID.** Uno scrub del supporto legge l'intero patrimonio di dati e su molti sistemi NAS viene eseguito secondo una pianificazione, spesso per ore.
- **Altri database sullo stesso volume.** Prometheus e Loki scrivono continuamente blocchi e li comprimono periodicamente. Queste compaction generano picchi di carico.
- **Checkpoint in modalità WAL.** Il checkpoint scrive molte pagine in una volta sola e termina con `fsync`. Se avviene in una fase di carico elevato, si prolunga di conseguenza.

Nel caso pratico si sono sommati diversi fattori: dischi rigidi SMR, backup e un ambiente di monitoraggio con più servizi che scrivono. Grafana era in esecuzione in modalità rollback senza retry. Ogni commit della cronologia degli stati degli alert richiedeva più chiamate a `fsync`, e ciascuna attendeva dietro il carico restante. All'avvio, Grafana legge tra l'altro valori dal proprio archivio chiave-valore nel database. Senza ripetizione, l'avvio si interrompeva al primo conflitto di blocco e il container veniva riavviato.

### Misurare la latenza

Se i supporti di memorizzazione sono il problema può essere misurato direttamente. L'utilizzo per supporto viene mostrato da `iostat` del pacchetto sysstat:

```bash
iostat -x 5
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-x` | Statistiche estese per dispositivo, tra cui `r_await`/`w_await` (tempo medio di attesa in ms) e `%util` (utilizzo). |
| `5` | Intervallo in secondi; il primo output mostra valori medi dall'avvio del sistema e non è significativo per la situazione corrente. |

</details>

Valori di `w_await` nell'ordine delle centinaia di millisecondi e `%util` vicino a 100 indicano che il supporto deve accodare i lavori. La latenza delle singole operazioni di scrittura sincrone può essere verificata con `dd` nella directory del database:

```bash
dd if=/dev/zero of=./fsync-test bs=4k count=500 \
   oflag=dsync
rm ./fsync-test
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `if=/dev/zero` | Origine: byte nulli, viene misurata solo la prestazione di scrittura. |
| `of=./fsync-test` | File di destinazione nella directory corrente; questa deve trovarsi sul volume del database. |
| `bs=4k` | Dimensione del blocco di 4 KiB, corrispondente alla comune dimensione di pagina SQLite. |
| `count=500` | Numero di blocchi, 2 MB in totale. |
| `oflag=dsync` | Ogni blocco viene scritto in modo sincrono, paragonabile a un `fsync` per operazione di scrittura. |

</details>

`dd` riporta alla fine durata e throughput. 500 blocchi in 2-5 secondi corrispondono a 4-10 ms per operazione di scrittura sincrona e sono normali per i dischi rigidi. Se lo stesso test richiede 60 secondi o più durante un backup, la causa dei blocchi è il supporto di memorizzazione.

## Soluzione nel caso pratico

Grafana supporta entrambe le misure tramite parametri di configurazione nella sezione `[database]`, che possono essere impostati come variabili d'ambiente:

```yaml
services:
  grafana:
    image: grafana/grafana-oss:13.0.2
    environment:
      GF_DATABASE_WAL: "true"
      GF_DATABASE_QUERY_RETRIES: "5"
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  grafana-data:
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `GF_DATABASE_WAL` | Corrisponde a `wal = true`: Grafana apre il file SQLite in modalità WAL. Il valore predefinito è `false`. |
| `GF_DATABASE_QUERY_RETRIES` | Corrisponde a `query_retries`: numero di ripetizioni di una query quando SQLite segnala `database is locked`. Il valore predefinito è `0`. |
| `grafana-data:/var/lib/grafana` | Named Volume sul supporto locale del NAS; qui si trovano `grafana.db` e, dopo la conversione, `grafana.db-wal` e `grafana.db-shm`. |

</details>

Per le transazioni esiste inoltre `transaction_retries` (predefinito `5`), che nel caso pratico non ha dovuto essere modificato. Dopo il riavvio con entrambe le impostazioni, Grafana si è avviato senza errori, l'endpoint health segnalava nuovamente `"database": "ok"` e i messaggi SQLITE_BUSY della cronologia degli stati degli alert non sono più comparsi. I nuovi file nel volume mostrano se la conversione ha avuto effetto:

```bash
docker exec grafana ls -l /var/lib/grafana
```

Se compaiono `grafana.db-wal` e `grafana.db-shm`, il database lavora in modalità WAL.

Su sistemi con carico elevato è possibile valutare altre due misure. La cronologia degli stati degli alert può essere scritta in Loki invece che nel database (`[unified_alerting.state_history]` con `backend = loki`), se Loki è già in esecuzione. Inoltre, per installazioni con molti utenti o più istanze Grafana, Grafana raccomanda un database esterno (MySQL o PostgreSQL) anziché SQLite.

## Backup senza conflitti di blocco

Un backup del file `grafana.db` durante il funzionamento acquisisce uno stato intermedio qualsiasi. In modalità WAL, senza il file `-wal` mancano inoltre tutte le modifiche non ancora trasferite. SQLite stesso crea una copia coerente:

```bash
sqlite3 /var/lib/grafana/grafana.db \
  ".backup /backup/grafana-$(date +%F).db"
```

L'API di backup online copia il database pagina per pagina e tiene conto delle operazioni di scrittura in corso. In alternativa, `VACUUM INTO '/backup/grafana.db'` genera una copia compatta. Entrambe le varianti possono essere eseguite prima del backup effettivo dei file, che eseguirà quindi il backup della copia invece del file live. Pianificate backup e scrub in modo che non si sovrappongano e non coincidano con picchi di carico noti di altri servizi.

## Checklist

- Attivare la modalità WAL, purché tutti gli accessi provengano dallo stesso computer.
- Impostare il busy timeout su ogni connessione, con 5-10 secondi come valore iniziale.
- Avviare le transazioni di scrittura con `BEGIN IMMEDIATE` e mantenerle brevi.
- Prevedere ripetizioni per `SQLITE_BUSY` a livello dell'applicazione, in Grafana tramite `query_retries`.
- Utilizzare un solo processo scrittore per file di database.
- Archiviare i file di database solo su file system locali, mai su condivisioni SMB o NFS.
- Misurare la latenza di scrittura se i blocchi si verificano frequentemente; evitare dischi rigidi SMR per i database o spostare i database su un SSD.
- Creare backup tramite `.backup` o `VACUUM INTO` e separarli temporalmente dagli scrub.

## Fonti

1.  [SQLite: Result and Error Codes](https://www.sqlite.org/rescode.html): significato di `SQLITE_BUSY` (5), `SQLITE_LOCKED` (6) e dei codici estesi come `SQLITE_BUSY_SNAPSHOT`.

2.  [SQLite: File Locking And Concurrency In SQLite Version 3](https://www.sqlite.org/lockingv3.html): i livelli di blocco SHARED, RESERVED, PENDING ed EXCLUSIVE nella modalità rollback.

3.  [SQLite: Write-Ahead Logging](https://www.sqlite.org/wal.html): funzionamento, checkpoint, `wal_autocheckpoint` e la limitazione ai file system locali.

4.  [SQLite: PRAGMA Statements](https://www.sqlite.org/pragma.html): `journal_mode`, `busy_timeout`, `synchronous` e `wal_checkpoint`.

5.  [SQLite: BEGIN TRANSACTION](https://www.sqlite.org/lang_transaction.html): differenza tra DEFERRED, IMMEDIATE ed EXCLUSIVE.

6.  [SQLite: SQLite Over a Network, Caveats and Considerations](https://www.sqlite.org/useovernet.html): perché i file system di rete non sono adatti a SQLite.

7.  [SQLite: How To Corrupt An SQLite Database File](https://www.sqlite.org/howtocorrupt.html): blocchi di file difettosi e `fsync` come cause di database danneggiati.

8.  [SQLite: SQLite Backup API](https://www.sqlite.org/backup.html): copie coerenti di un database durante il funzionamento.

9.  [Python: sqlite3 DB-API 2.0 interface](https://docs.python.org/3/library/sqlite3.html): parametri `timeout` e `isolation_level` di `sqlite3.connect()`.

10.  [Grafana: Configure Grafana, sezione database](https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/#database): parametri `wal`, `query_retries` e `transaction_retries`, nonché la mappatura alle variabili d'ambiente `GF_`.

11.  [Wikipedia: Shingled magnetic recording](https://en.wikipedia.org/wiki/Shingled_magnetic_recording): struttura dei dischi rigidi SMR e loro comportamento con le operazioni di scrittura casuali.
