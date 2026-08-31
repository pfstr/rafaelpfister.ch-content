---
title: "I controlli più importanti per gli amministratori di Totemomail: arrestare il server, verificare le code e ripulirle in modo controllato"
navTitle: "Controlli Totemomail"
description: "I controlli più importanti per la gestione di un gateway totemomail: arrestare il servizio tramite systemd e lo script di controllo Tanuki, contare le code per repository, ispezionare singoli messaggi, ripulire in modo controllato e riavviare il servizio."
date: "2026-08-28"
kategorie: "Totemomail"
timeToRead: "9 min di lettura"
themen:
  - totemomail
produkte:
  - "totemomail"
  - "apache-james"
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "i-controlli-piu-importanti-per-gli-amministratori-di-totemomail-arrestare-il-server-verificare"
translationId: "article-3a0a526ab6e38a06"
translationOf: totemomail-server-stoppen-queues-bereinigen
url: https://rafaelpfister.ch/it/blog/i-controlli-piu-importanti-per-gli-amministratori-di-totemomail-arrestare-il-server-verificare
translationSourceHash: bc887dcd4aa82db7e020247f75b86528f0fa331e1643c28a215a1638587197a6
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:40:30.423Z
translationReview: automatic
---

# I controlli più importanti per gli amministratori di Totemomail: arrestare il server, verificare le code e ripulirle in modo controllato

Per gestire un gateway totemomail (oggi Kiteworks Email Protection Gateway), quattro operazioni fanno parte degli strumenti di base: arrestare correttamente il servizio, rilevare il contenuto delle code, ispezionare singoli messaggi e ripulire le code in modo controllato prima di riavviare il servizio.

Questi passaggi sono necessari sia per la manutenzione pianificata sia in caso di problemi, ad esempio quando una regola errata, una destinazione irraggiungibile o un test di carico ha riempito le code. Questo articolo illustra ogni passaggio con i comandi concreti, compresa la questione di come arrestare correttamente il servizio. Il modello di elaborazione sottostante (processor, repository, formati di file) è descritto nell'articolo [Comprendere il routing della posta tra totemomail ed Exchange Online](/blog/totemomail-m365).

Tutti i percorsi si riferiscono a un'installazione in `/opt/totemomail` con l'utente di servizio `totemo`. Adattate i percorsi al vostro ambiente.

## Come viene avviato e arrestato totemomail

Prima di arrestare un servizio, è opportuno sapere come viene eseguito. In totemomail sono coinvolti tre livelli:

- Un'**unità systemd** `totemomail.service` come livello di controllo più esterno.
- Lo **script di controllo** `/opt/totemomail/bin/totemomail`, che richiama l'unità all'avvio e all'arresto.
- Il **Tanuki Java Service Wrapper**: un processo nativo `wrapper` che avvia e monitora il processo Java vero e proprio e può riavviarlo in caso di arresto anomalo.

Potete verificare questa struttura sul vostro sistema senza dover leggere il file dell'unità. `systemctl show` interroga direttamente le proprietà presso systemd e funziona anche se il file in `/etc/systemd/system/` è leggibile solo da root:

```bash
systemctl show totemomail.service -p Type -p User -p ExecStart -p ExecStop \
  -p KillMode -p TimeoutStopUSec --no-pager
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `show totemomail.service` | Mostra le proprietà a runtime dell'unità, come caricate da systemd |
| `-p <Property>` | Limita l'output alla proprietà indicata; può essere specificata più volte |
| `--no-pager` | Stampa direttamente sulla console invece di aprire un pager come `less` |

</details>

Un output tipico è il seguente:

```text
Type=oneshot
TimeoutStopUSec=1min 30s
ExecStart={ path=/opt/totemomail/bin/totemomail ; argv[]=/opt/totemomail/bin/totemomail start ; ... }
ExecStop={ path=/opt/totemomail/bin/totemomail ; argv[]=/opt/totemomail/bin/totemomail stop ; ... }
User=totemo
KillMode=control-group
```

Da qui si possono leggere le proprietà importanti: `systemctl stop totemomail` richiama lo script di controllo con l'argomento `stop`, attende fino a 90 secondi una terminazione corretta e poi termina con `KillMode=control-group` tutti i processi ancora presenti nell'unità. L'arresto tramite systemd equivale quindi alla chiamata diretta dello script, ma esegue anche la pulizia se lo script rimane bloccato.

Lo stato `active (exited)` in `systemctl status totemomail` è normale in questa configurazione e non è un errore: l'unità è `Type=oneshot`, lo script di avvio termina dopo l'avvio e il wrapper continua a essere eseguito come demone autonomo gestito solo indirettamente da systemd. Per questo, lo stato dell'unità non indica se il servizio è realmente in esecuzione: occorre invece verificare l'elenco dei processi:

```bash
ps -ef | grep -E 'wrapper|TotemoBootStrapper' | grep -v grep
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-e` | Mostra tutti i processi, non soltanto quelli della propria sessione |
| `-f` | Formato di output completo con riga di comando integrale |
| `grep -E 'wrapper\|TotemoBootStrapper'` | Filtra il processo wrapper e la classe principale Java |
| `grep -v grep` | Rimuove dall'elenco dei risultati gli stessi processi grep |

</details>

Nel normale funzionamento compaiono due processi: il processo nativo `wrapper` (avviato con `../conf/wrapper.conf` e il file PID `totemomail.pid`) e il processo Java con la classe principale `ch.totemo.bootstrapper.TotemoBootStrapper`. Se ne manca uno, il servizio non è stato avviato completamente.

## Passaggio 1: arrestare il servizio

Per qualsiasi intervento sulle code, arrestate prima il servizio. Finché totemomail è in esecuzione, accetta messaggi, elabora le code ed effettua le consegne; solo l'arresto blocca il contenuto per l'analisi.

```bash
sudo systemctl stop totemomail
```

Verificate quindi che i processi wrapper e Java siano terminati:

```bash
ps -ef | grep -E 'wrapper|TotemoBootStrapper' | grep -v grep
```

L'output deve essere vuoto. Inoltre scompare il file PID `/opt/totemomail/bin/totemomail.pid`. Se un processo resta attivo dopo la scadenza del timeout di arresto, systemd lo termina tramite il control group; in questo caso verificate `journalctl -u totemomail` prima di procedere.

Non dimenticate il livello a monte: durante l'arresto, i nuovi messaggi in arrivo si accumulano nel sistema di consegna, ad esempio nella coda di Exchange o nel relay a monte. È voluto. I mittenti affidabili ritentano automaticamente la consegna dopo il riavvio.

## Passaggio 2: rilevare il contenuto delle code

Le code di totemomail sono repository di posta basati su file dell'Apache James sottostante. Si trovano nella directory dell'applicazione James, qui `/opt/totemomail/mailer/apps/james/var/mail/`. Ogni sottodirectory è un repository e ogni messaggio è composto da due file: `*.FileStreamStore` contiene il messaggio MIME completo, `*.FileObjectStore` l'oggetto di stato serializzato con i metadati.

Una panoramica del contenuto si ottiene contando i file `FileObjectStore` per directory:

```bash
for d in /opt/totemomail/mailer/apps/james/var/mail/*/; do \
  printf '%-22s %s\n' "$(basename "$d")" \
  "$(find "$d" -maxdepth 1 -name '*.FileObjectStore' | wc -l)"; \
done
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `for d in .../*/` | Itera su tutte le directory dei repository (il `/` finale limita alle directory) |
| `printf '%-22s %s\n'` | Formatta l'output in due colonne; `%-22s` riempie il nome a sinistra fino a 22 caratteri |
| `basename "$d"` | Riduce il percorso completo al nome della directory |
| `find "$d" -maxdepth 1` | Cerca soltanto direttamente nella directory, senza sottodirectory |
| `-name '*.FileObjectStore'` | Conta un file per messaggio; il corrispondente stream raddoppierebbe il numero |
| `wc -l` | Conta i file trovati |

</details>

Il risultato è una riga per coda con il numero di messaggi, ad esempio:

```text
DBUnavailable          0
error                  12
incoming               121
outgoing               0
spool                  0
```

I repository standard hanno il seguente significato: `spool` contiene i messaggi accettati ma non ancora elaborati, `incoming` quelli da consegnare internamente, `outgoing` quelli in uscita, `error` quelli non riusciti e `DBUnavailable` i messaggi parcheggiati a causa di un backend irraggiungibile. A seconda della configurazione, esistono altri repository per percorsi specifici; seguono lo stesso schema di file.

Se `find` viene eseguito da una directory a cui l'utente di servizio non ha accesso, ad esempio la home di un altro utente dopo `sudo -u totemo`, per ogni chiamata compare l'avviso `Failed to restore initial working directory`. È innocuo e scompare dopo un `cd ~`.

## Passaggio 3: esaminare i messaggi

I soli numeri non sono sufficienti per prendere una decisione. Prima di eliminare qualcosa, dovreste sapere cosa si trova nelle code: messaggi indesiderati causati da un problema oppure email legittime da consegnare dopo il riavvio?

I file `FileStreamStore` sono messaggi RFC 822 non modificati. I principali header possono quindi essere letti direttamente:

```bash
for f in /opt/totemomail/mailer/apps/james/var/mail/incoming/*.FileStreamStore; do \
  awk 'BEGIN{IGNORECASE=1} /^(From|To|Subject|Date):/{print} /^\r?$/{exit}' "$f"; \
  echo ---; \
done | less
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `BEGIN{IGNORECASE=1}` | Confronta i nomi degli header senza distinguere maiuscole e minuscole (GNU awk) |
| `/^(From\|To\|Subject\|Date):/{print}` | Emette soltanto le quattro righe di header rilevanti |
| `/^\r?$/{exit}` | Si interrompe alla riga vuota tra header e corpo; il contenuto del messaggio non viene letto |
| `echo ---` | Riga separatrice tra i messaggi |
| `less` | Consente di sfogliare anziché scorrere con molti messaggi |

</details>

Con grandi volumi, la distribuzione è più significativa della visualizzazione dei singoli messaggi. I mittenti più frequenti vengono mostrati da:

```bash
grep -him1 '^From:' /opt/totemomail/mailer/apps/james/var/mail/incoming/*.FileStreamStore \
  | sort | uniq -c | sort -rn | head
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-h` | Sopprime i nomi dei file nell'output, in modo che mittenti identici vengano raggruppati |
| `-i` | Ignora maiuscole e minuscole |
| `-m1` | Solo la prima corrispondenza per file (l'header, non le righe `From:` citate nel corpo) |
| `sort \| uniq -c` | Raggruppa le righe di mittente identiche e le conta |
| `sort -rn \| head` | Ordina in ordine decrescente di frequenza e mostra le dieci più frequenti |

</details>

Se un singolo mittente o oggetto domina con centinaia di copie, ciò indica un loop o un invio di massa inoltrato erroneamente; questi messaggi sono candidati per la pulizia. Un'occhiata ai timestamp dei file (`ls -lt`) delimita ulteriormente il periodo e mostra se vi sono messaggi legittimi più vecchi nel mezzo.

## Passaggio 4: ripulire in modo controllato

Solo ora si procede all'eliminazione, e anche ora con un passaggio intermedio: spostate prima il contenuto in una directory di backup invece di eliminarlo direttamente. Il risultato per il funzionamento della posta è lo stesso (la coda è vuota), ma il passaggio è reversibile e in seguito è possibile ripristinare singoli messaggi legittimi dal backup o riutilizzarli come `.eml`.

```bash
mkdir -p /opt/totemomail/queue-backup-$(date +%F)
mv /opt/totemomail/mailer/apps/james/var/mail/incoming/* \
   /opt/totemomail/queue-backup-$(date +%F)/
```

Importante: le directory dei repository restano in posizione, viene spostato solo il loro contenuto. Inoltre, i file stream e object di un messaggio appartengono insieme; rimuovendone solo uno si lasciano file orfani che generano errori nel log al successivo avvio.

Se il backup è stato verificato o il contenuto è sicuramente privo di valore, ad esempio messaggi di puro test di carico, eliminate l'intero contenuto delle code in tutti i repository:

```bash
find /opt/totemomail/mailer/apps/james/var/mail/ -mindepth 2 -maxdepth 2 -type f \
  \( -name '*.FileStreamStore' -o -name '*.FileObjectStore' \) -delete
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-mindepth 2 -maxdepth 2` | Interessa soltanto i file direttamente nelle directory dei repository, non `var/mail` stesso né livelli più profondi |
| `-type f` | Solo file regolari; le directory rimangono inalterate |
| `\( -name ... -o -name ... \)` | Entrambi i tipi di file di un messaggio, stream e oggetto di stato |
| `-delete` | Elimina direttamente le corrispondenze; eseguite prima senza questa opzione per verificare l'elenco delle corrispondenze |

</details>

Eseguite poi lo stesso conteggio del passaggio 2: tutti i repository devono mostrare 0.

## Passaggio 5: riavviare il servizio

```bash
sudo systemctl start totemomail
```

L'avvio richiama lo script di controllo con `start`, che rende il wrapper un demone; il wrapper avvia quindi il processo Java. Verificate entrambi nell'elenco dei processi della prima sezione e date un'occhiata ai file di log in `/opt/totemomail/bin/`: `wrapper.log` registra l'avvio del wrapper e della JVM, mentre `console.log` e `console.err` registrano gli output dell'applicazione stessa.

Come conclusione, eseguite un test funzionale con un singolo messaggio di prova attraverso il gateway prima di riabilitare il normale flusso di posta. E se una regola o un loop di posta aveva riempito le code, correggete prima la causa e solo poi consentite nuovamente il traffico. Altrimenti il rilevamento del contenuto delle code ricomincia da capo.

## Riepilogo

| Passaggio | Comando | Verifica |
|---|---|---|
| Arrestare | `sudo systemctl stop totemomail` | Filtro `ps` vuoto, file PID scomparso |
| Contare il contenuto | Ciclo `find` su `var/mail/*/` | Numero per repository |
| Ispezionare | Estrazione degli header con `awk`, statistica dei mittenti con `grep` | Separare messaggi indesiderati da quelli legittimi |
| Ripulire | `mv` nel backup, quindi `find ... -delete` | Il conteggio mostra 0 ovunque |
| Avviare | `sudo systemctl start totemomail` | Processi, `wrapper.log`, messaggio di prova |

## Fonti

1.  [Apache James Server 2: Provided Mailets](https://james.apache.org/server/2/provided_mailets.html): Documentazione dei mailet e dei repository su cui si basa la struttura delle code di totemomail.

2.  [Tanuki Software: Java Service Wrapper](https://wrapper.tanukisoftware.com/doc/english/introduction.html): Funzionamento del wrapper che avvia e monitora il processo Java di totemomail, incluso il file PID e `wrapper.conf`.

3.  [systemd.service(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html): Significato di `Type=oneshot`, `ExecStop` e `TimeoutStopSec` per le unità che richiamano uno script di controllo esterno.

4.  [systemd.kill(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.kill.html): `KillMode=control-group` come protezione che termina i processi dell'unità rimasti dopo lo script di arresto.

5.  [RFC 5322: Internet Message Format](https://datatracker.ietf.org/doc/html/rfc5322): Struttura degli header dei messaggi letti durante l'ispezione dei file `FileStreamStore`.
