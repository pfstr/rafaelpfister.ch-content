---
title: "Test di carico SMTP con destinatari numerati: inviare 50'000 e-mail in modo tracciabile"
navTitle: "Test di carico numerati"
description: "Un test di carico è valido solo quanto la sua valutazione. Con l'opzione -N, smtp-source numera ogni e-mail tramite l'indirizzo del destinatario senza sacrificare il throughput. Come impostare un'esecuzione con 50'000 e-mail, quante sessioni sono sensate e come individuare automaticamente i numeri mancanti."
date: "2026-08-27"
kategorie: "SMTP e flusso di posta"
timeToRead: "8 min di lettura"
themen:
  - smtp-lasttests
  - smtp-mailflow
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "testing"
slug: "test-di-carico-smtp-con-destinatari-numerati-inviare-50-000-e-mail-in-modo-tracciabile"
translationId: "article-57f09c758baf6e1e"
translationOf: smtp-lasttest-nummerierte-empfaenger
url: https://rafaelpfister.ch/it/blog/test-di-carico-smtp-con-destinatari-numerati-inviare-50-000-e-mail-in-modo-tracciabile
translationSourceHash: a2ec75884c06a6d736ea9b5895211ddc4cbba252c7ddf491752e1bec5ab1a24d
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:20:50.743Z
translationReview: automatic
---

# Test di carico SMTP con destinatari numerati: inviare 50'000 e-mail in modo tracciabile

Chi esegue un test di carico con 50'000 e-mail vuole poter rispondere in seguito a due domande: sono arrivate tutte e, in caso contrario, quali mancano? Con e-mail di test identiche si può solo contare, e una differenza tra 49'987 e 50'000 non dice quando e dove siano andati persi i 13 messaggi mancanti. Se invece ogni e-mail reca un numero progressivo, il conteggio diventa un confronto: ogni numero può essere individuato singolarmente nei log del sistema di destinazione, le lacune mostrano il momento della perdita e si può verificare l'ordine di consegna.

La reazione istintiva più diffusa è uno script che incrementa l'oggetto. Funziona, ma costa throughput, perché il generatore di carico `smtp-source` del pacchetto Postfix imposta l'oggetto in modo fisso per chiamata e un ciclo con una chiamata per e-mail impone una nuova connessione per ogni messaggio. L'identificatore migliore del messaggio è già integrato: l'opzione `-N` numera l'indirizzo del destinatario per ogni messaggio, all'interno di un'unica chiamata con sessioni parallele. Per la valutazione, l'indirizzo del destinatario è utile quanto l'oggetto, poiché compare in ogni log di tracciamento.

Questa configurazione di test, a differenza di un puro test funzionale in loopback, invia a un altro sistema attraverso la rete. Se sul sistema sorgente non è installato Postfix, l'articolo [smtp-source senza installazione di Postfix](https://rafaelpfister.ch/blog/smtp-source-ohne-postfix-installation) mostra come estrarre gli strumenti dall'RPM.

## Le opzioni più importanti di smtp-source

Per orientarsi, ecco le opzioni trattate in questo articolo, tradotte per significato dalla manpage:

| Opzione | Significato |
|---|---|
| `-s n` | Numero di sessioni SMTP parallele (predefinito: 1) |
| `-m n` | Numero totale di messaggi da inviare (predefinito: 1) |
| `-l n` | Dimensione del testo del messaggio in byte, senza header |
| `-f adresse` | Indirizzo del mittente |
| `-t adresse` | Indirizzo del destinatario (predefinito: `foo@hostname`) |
| `-S text` | Riga dell'oggetto, fissa per tutti i messaggi della chiamata |
| `-F datei` | Invia header e body invariati da un file; sostituisce `-l` e `-S` |
| `-N` | Numera l'indirizzo del destinatario per messaggio (contatore per processo; posizione e valore iniziale dipendono dalla versione, vedi sotto) |
| `-r n` | Numero di destinatari per messaggio (predefinito: 1), formazione dell'indirizzo come con `-N` |
| `-d` | Non disconnettersi dopo un messaggio, inviare il successivo sulla stessa connessione |
| `-c` | Mostra un contatore progressivo che aumenta a ogni `DATA` completato |
| `-w n` | Tempo di attesa fisso di n secondi tra i messaggi (per sessione) |
| `-v` | Output dettagliato per la risoluzione dei problemi |
| `host:port` | Destinazione della consegna via TCP; senza porta viene usata la porta SMTP standard |

L'elenco completo, comprese le opzioni TLS, LMTP e di timing, è disponibile nella manpage di `smtp-source(1)`; la controparte per il lato ricevente è `smtp-sink(1)` e viene usata più avanti nella valutazione.

## Come -N numera i destinatari

`-N` attiva un contatore per processo incorporato nell'indirizzo del destinatario. Tre proprietà determinano la configurazione del test; tutte e tre sono consultabili nel codice sorgente di `smtp-source.c`:

Innanzitutto, il formato esatto dell'indirizzo dipende dalla versione di Postfix. Postfix 3.5, come fornito da RHEL 8, antepone il numero all'intero indirizzo (`RCPT TO:<%d%s>`): da `-t test@example.com` diventano `1test@example.com`, `2test@example.com` e così via, e il contatore inizia da 1. Le versioni attuali di Postfix aggiungono invece il numero alla fine della parte locale e iniziano da 0 (`test0@` fino a `test49999@`); per questa variante la manpage raccomanda l'indirizzamento con plus (`-t 'test+@example.com'` produce `test+0@` e i successivi), affinché un sistema di destinazione con sottoindirizzamento associ tutto alla stessa casella di posta. Prima dell'esecuzione principale, verificate il formato con una manciata di e-mail tramite un `smtp-sink` o nel log della destinazione; da questo dipendono il set atteso e il modello di ricerca per la valutazione.

In secondo luogo, il contatore è globale al processo ed è condiviso da tutte le sessioni parallele. Con `-s 8`, le otto sessioni assegnano insieme i numeri e ogni numero compare esattamente una volta. L'ordine tra le sessioni non è deterministico, ma la completezza dell'insieme di numeri è garantita.

In terzo luogo, il valore iniziale non è configurabile: 1 in Postfix 3.5, 0 nelle versioni attuali. Le 50'000 e-mail portano quindi i numeri da 1 a 50'000 oppure da 0 a 49'999, e il set atteso per il confronto deve corrispondere.

## L'esecuzione del test in un'unica chiamata

```bash
smtp-source -c -d -N -s 8 -m 50000 -l 5120 \
  -f lasttest@example.com \
  -t test@example.com \
  gateway.example.com:25
```

| Opzione | Effetto |
|---|---|
| `-c` | Contatore progressivo delle consegne completate come indicatore di avanzamento su una riga |
| `-d` | Le connessioni restano aperte per tutti i messaggi; senza `-d` viene aperta una nuova connessione per messaggio |
| `-N` | Numerazione dei destinatari: aggiunge il contatore per processo alla parte locale |
| `-s 8` | Otto sessioni SMTP parallele |
| `-m 50000` | Numero totale di messaggi, distribuiti tra le sessioni |
| `-l 5120` | Dimensione del messaggio in byte (senza header), qui 5 KB |
| `-f` | Indirizzo del mittente |
| `-t` | Indirizzo base del destinatario; `-N` lo trasforma in `1test@` fino a `50000test@` (Postfix 3.5) oppure da `test0@` a `test49999@` (versioni attuali) |
| `gateway.example.com:25` | Host e porta di destinazione |

`-d` è decisivo per il profilo di carico: senza questa opzione, `smtp-source` chiude la connessione dopo ogni messaggio e ne apre una nuova per il successivo; con `-d`, le otto connessioni restano aperte e consegnano tutti i messaggi uno dopo l'altro, come farebbe un mittente di massa.

Manca deliberatamente il noto `-v` dei test funzionali: registra ogni singolo dialogo SMTP da `HELO` a `QUIT` e genera centinaia di migliaia di righe di log per 50'000 e-mail senza valore aggiunto per la valutazione. `-c` fornisce invece il riepilogo dal quale è possibile leggere in tempo reale l'avanzamento dell'esecuzione. La durata totale per calcolare la velocità è fornita da un `time` posto prima del comando.

Prerequisito dell'intero approccio: il sistema di destinazione accetta gli indirizzi generati. Un `smtp-sink`, un dominio catch-all, un dominio di scarto del provider o un gateway che risolve i destinatari soltanto dopo l'accettazione soddisfano questo requisito. Se invece la destinazione verifica ogni destinatario rispetto a una directory, rifiuta gli indirizzi numerati e resta solo la variante con l'oggetto.

## Impostare header personalizzati

Alcuni test di carico richiedono un header personalizzato, ad esempio come marcatore con cui il gateway riconosce le e-mail di test o attiva una regola. `smtp-source` non dispone di un'opzione a questo scopo, ma `-F` legge da un file un messaggio completamente preformattato, nel quale può figurare qualsiasi header desiderato. Il file è composto dalle righe di header, una riga vuota e il body; tutte le righe terminano con `\r\n`:

```bash
{ printf 'X-Lasttest: aktiv\r\n'
  printf 'Subject: Lasttest\r\n'
  printf '\r\n'
  head -c 5120 /dev/zero | tr '\0' 'x'
  printf '\r\n'; } > lasttest.eml
```

```bash
smtp-source -c -d -N -s 8 -m 50000 -F lasttest.eml \
  -f lasttest@example.com \
  -t test@example.com \
  gateway.example.com:25
```

| Opzione | Effetto |
|---|---|
| `-F datei` | Invia header e body invariati dal file; sostituisce il contenuto del messaggio generato |

Due conseguenze: `-F` sostituisce `-l` e `-S`, poiché dimensione e oggetto provengono ora dal file (entrambi devono quindi essere inclusi). `-N` rimane invece attivo e i destinatari continuano a essere numerati; l'header è identico in tutti i messaggi, poiché proviene dal file fisso.

## Quante sessioni?

Il modo più affidabile per determinare il numero di sessioni adatto è misurare, utilizzando esattamente le stesse opzioni dell'esecuzione principale prevista: stessa sorgente dei messaggi (lo stesso file `-F` oppure lo stesso `-l`), stesso mittente, stessa destinazione. Solo la quantità viene ridotta a 2'000 per livello e `-s` varia. Una breve esecuzione di calibrazione con un numero crescente di sessioni mostra da quando sessioni aggiuntive non apportano più vantaggi:

```bash
for s in 1 2 4 8 16 32; do
  t0=$(date +%s%N)
  smtp-source -d -N -s "$s" -m 2000 -F lasttest.eml \
    -f lasttest@example.com -t '@blackhole.example.com' \
    gateway.example.com:25
  t1=$(date +%s%N)
  echo "$s Sessions: $(( 2000000000000 / (t1 - t0) )) Mails/s"
done
```

Due dettagli sulla chiamata: qui si rinuncia deliberatamente a `-c`, in modo che tra le righe di misurazione non compaiano output di contatori progressivi; il ciclo produce esattamente una riga di risultato per livello. Inoltre, la parte locale vuota in `-t` funziona bene insieme alla numerazione per un dominio di scarto: con il contatore anteposto di Postfix 3.5 ne risultano indirizzi di destinatari puramente numerici (`1@blackhole.example.com`, `2@…`), rendendo più chiara la valutazione nei log.

Nel dettaglio avviene quanto segue: il ciclo esterno attraversa i numeri di sessioni da 1 a 32 raddoppiandoli a ogni passaggio. Prima e dopo ogni esecuzione, `date +%s%N` registra l'ora corrente come un numero grande, ovvero i secondi Unix seguiti direttamente dalla parte in nanosecondi. Nel mezzo, `smtp-source` invia 2'000 messaggi (contenuto, header e dimensione provengono dal file `-F`) attraverso il rispettivo numero di connessioni parallele che, grazie a `-d`, restano aperte; il ciclo attende il completamento della chiamata. La riga `echo` converte la differenza temporale in una velocità: 2'000 e-mail divise per la durata in secondi, mentre la durata è espressa in nanosecondi. Da 2'000 per 10⁹ risulta quindi la costante `2000000000000`. L'aritmetica Bash `$(( ))` calcola con numeri interi e tronca i decimali, cosa sufficientemente precisa per questa misurazione.

Tre indicazioni pratiche: `%N` fornisce nanosecondi solo con GNU date (come avviene su RHEL e sulla maggior parte dei sistemi Linux; BusyBox e macOS non lo supportano). L'esecuzione completa invia 6 × 2'000 = 12'000 e-mail, che richiedono anch'esse un indirizzo del destinatario controllato, e la numerazione `-N` ricomincia dal valore iniziale a ogni chiamata. Se una chiamata di `smtp-source` termina con un messaggio di errore, la velocità di quella riga è priva di significato; risolvete prima la causa, poi misurate nuovamente.

L'output previsto è una riga per livello. Con valori di esempio inventati ma tipici, appare così:

```text
1 Sessions: 11 Mails/s
2 Sessions: 21 Mails/s
4 Sessions: 40 Mails/s
8 Sessions: 71 Mails/s
16 Sessions: 79 Mails/s
32 Sessions: 80 Mails/s
```

L'interpretazione: finché la velocità circa raddoppia con il numero di sessioni, le sessioni parallele coprono il tempo di attesa delle risposte della destinazione; il collo di bottiglia è quindi la latenza del percorso, non la capacità. Dal punto in cui la curva si appiattisce (nell'esempio tra 8 e 16 sessioni), il sistema di destinazione è saturo oppure la sorgente è al limite. Scegliete il valore più piccolo per cui la velocità non aumenta più in modo significativo, nell'esempio quindi da 8 a 16; ulteriori sessioni aumentano solo il carico dovuto al parallelismo, non il throughput. Per l'esecuzione principale con 50'000 e-mail, la velocità misurata consente anche di stimare la durata prevista: a 71 e-mail/s, circa 12 minuti.

## Valutazione sul lato ricevente

Se sul sistema di destinazione è disponibile un destinatario di test dedicato, `smtp-sink` gestisce direttamente anche la registrazione:

```bash
smtp-sink -c -d "mails/%Y%m%d-%H%M%S." 0.0.0.0:2525 200
```

| Opzione | Effetto |
|---|---|
| `-c` | Contatori progressivi invece del dialogo SMTP completo |
| `-d "mails/…"` | Per il sink: dump, non mantenimento della connessione. Scrive ogni messaggio accettato in un file separato (modello del nome tramite strftime), incluso un header `X-Rcpt-Args` con l'indirizzo del destinatario |
| `0.0.0.0:2525` | Rimane in ascolto su tutte le interfacce sulla porta 2525 |
| `200` | Backlog: lunghezza massima della coda di connessioni in attesa secondo listen(2) |

Dopo l'esecuzione, estraete i numeri ricevuti e confrontateli con il set atteso. Poiché i numeri non hanno zeri iniziali, entrambe le liste vengono portate a una lunghezza fissa prima del confronto, affinché l'ordinamento alfabetico di `comm` corrisponda a quello numerico. Il modello di ricerca è adatto al formato di indirizzo di Postfix 3.5 (numero prima dell'indirizzo); per le versioni attuali, usate invece `test[0-9]+@` e `seq 0 49999`:

```bash
grep -rhoE '[0-9]+test@example\.com' mails/ | \
  grep -oE '^[0-9]+' | sort -u | \
  awk '{printf "%08d\n", $1}' | sort > empfangen.txt
```

```bash
seq 1 50000 | awk '{printf "%08d\n", $1}' | \
  comm -23 - empfangen.txt
```

`comm -23` restituisce esattamente i numeri presenti nel set atteso ma non nella lista di ricezione: le e-mail mancanti. Un output vuoto significa consegna completa. Se alcuni numeri compaiono due volte (riconoscibile dalla differenza tra `sort` e `sort -u`), un sistema lungo il percorso ha duplicato il messaggio, il che costituisce anch'esso un riscontro.

Se la destinazione è un sistema simile alla produzione anziché un smtp-sink, il suo logging assume il ruolo dei file dump. Su un server Exchange, ad esempio, `Get-MessageTrackingLog -Recipients` oppure un filtro sull'indirizzo del destinatario fornisce i numeri arrivati; su un sistema Postfix, un `grep` su `to=` e l'indirizzo base tramite il maillog. Questo è precisamente il vantaggio del numero nell'indirizzo: il destinatario è presente in ogni tracciamento dei messaggi, mentre l'oggetto può mancare a seconda del sistema o richiedere l'attivazione.

## Quando il numero deve essere nell'oggetto

Alcune valutazioni dipendono dall'oggetto, ad esempio se il sistema di destinazione riscrive gli indirizzi dei destinatari o se i log mostrano il destinatario solo mascherato. In tal caso resta la variante con il ciclo: una chiamata di `smtp-source` per e-mail con `-m 1` e un oggetto incrementato dalla shell, distribuita su più worker paralleli con intervalli di numeri contigui.

```bash
worker() {
  local i
  for ((i = $1; i <= $2; i++)); do
    smtp-source -s 1 -m 1 -l 5120 \
      -S "$(printf 'Lasttest %05d' "$i")" \
      -f lasttest@example.com -t test@example.com \
      gateway.example.com:25 || echo "$i" >> fehlend.log
  done
}
for w in 0 1 2 3; do
  worker $(( w * 12500 + 1 )) $(( (w + 1) * 12500 )) &
done
wait
```

Il prezzo è una connessione completa per e-mail: handshake TCP, banner, `HELO`, invio, `QUIT`. Questa esecuzione non misura quindi il throughput massimo del sistema di destinazione, bensì un caso deliberatamente intensivo in termini di connessioni. Il numero di worker viene determinato analogamente all'esecuzione di calibrazione sopra, ma con il ciclo dei worker invece di `-s`. Gli zeri iniziali nell'oggetto evitano, durante il confronto, la riformattazione richiesta dalla variante `-N`.

## Regole per i test verso altri sistemi

Non appena il test esce dal proprio sistema, si applicano tre condizioni. Primo: il gestore del sistema di destinazione deve essere informato e avere approvato la finestra temporale; 50'000 e-mail sembrano a qualsiasi monitoraggio un attacco o un'ondata di spam. Secondo: l'indirizzo del destinatario deve terminare in modo controllato, in una casella di posta di test dedicata, in una regola di scarto sulla destinazione o in un dominio di scarto predisposto dal provider; gli indirizzi di produzione non appartengono a un test di carico. Terzo: prima dell'avvio deve essere definito un criterio di interruzione, ad esempio una coda in crescita sulla destinazione o un tasso di errore superiore a una soglia, e qualcuno deve osservare questi valori durante l'esecuzione.

Con questi tre punti e la numerazione, alla fine l'esecuzione non fornisce solo un valore di throughput, ma un'affermazione dimostrabile: quali delle 50'000 e-mail sono arrivate, quali mancano e dove sono state viste per l'ultima volta lungo il percorso.

## Fonti

1.  [Postfix: smtp-source(1)](https://www.postfix.org/smtp-source.1.html): manpage del generatore di carico; descrive il comportamento di `-N` nella versione attuale (contatore nella parte locale, indirizzamento con plus).

2.  [Codice sorgente Postfix 3.5.8: smtp-source.c](https://github.com/vdukhovni/postfix/blob/v3.5.8/postfix/src/smtpstone/smtp-source.c): documenta per la versione RHEL 8 l'anteposizione del numero (`RCPT TO:<%d%s>`) con valore iniziale 1; nella [versione attuale](https://github.com/vdukhovni/postfix/blob/master/postfix/src/smtpstone/smtp-source.c) il numero viene invece aggiunto alla parte locale, a partire da 0.

3.  [Postfix: smtp-sink(1)](https://www.postfix.org/smtp-sink.1.html): manpage del destinatario di test con le opzioni dump e gli header X registrati.

4.  [GNU Coreutils: comm](https://www.gnu.org/software/coreutils/manual/html_node/comm-invocation.html): confronto tra insiemi di due liste ordinate, qui per confrontare i numeri attesi e quelli ricevuti.
