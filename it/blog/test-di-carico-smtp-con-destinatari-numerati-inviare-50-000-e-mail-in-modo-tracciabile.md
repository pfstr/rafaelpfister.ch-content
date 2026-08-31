---
title: "Test di carico SMTP con destinatari numerati: inviare ogni email in modo tracciabile"
navTitle: "Test di carico numerati"
description: "Un test di carico è valido solo quanto la sua valutazione. Con l'opzione -N, smtp-source numera ogni email tramite l'indirizzo del destinatario senza sacrificare il throughput. Come strutturare l'esecuzione, quante sessioni hanno senso e come individuare automaticamente i numeri mancanti."
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
translationSourceHash: 7145f2b49fb0b141d9c74d009d7c480ce4d119b4c97236e2ed7d92a39f65a1c5
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:47:29.025Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/test-di-carico-smtp-con-destinatari-numerati-inviare-50-000-e-mail-in-modo-tracciabile
---

# Test di carico SMTP con destinatari numerati: inviare ogni email in modo tracciabile

Chi esegue un test di carico vuole poter rispondere a due domande al termine: tutte le email sono arrivate e, in caso contrario, quali mancano? Con email di test identiche si può solo contare, e un contatore che indica 13 messaggi mancanti non dice quando e dove siano andati persi. Se invece ogni email porta un numero progressivo, il conteggio diventa un confronto: ogni numero può essere individuato singolarmente nei log del sistema di destinazione, le lacune mostrano il momento della perdita e si può verificare l'ordine di consegna.

La reazione istintiva più diffusa è uno script che incrementa l'oggetto. Funziona, ma riduce il throughput, perché il generatore di carico `smtp-source` del pacchetto Postfix imposta l'oggetto in modo fisso per ogni chiamata e un ciclo con una chiamata per email impone una nuova connessione per ogni messaggio. L'identificativo migliore del messaggio è già integrato: l'opzione `-N` numera l'indirizzo del destinatario per ogni messaggio, all'interno di una singola chiamata con sessioni parallele. Per la valutazione, l'indirizzo del destinatario è utile quanto l'oggetto, poiché è presente in ogni log di tracciamento.

Questa configurazione di test, a differenza di un puro test funzionale di loopback, invia tramite la rete a un altro sistema. Se sul sistema sorgente non è installato Postfix, l'articolo [smtp-source senza installazione di Postfix](https://rafaelpfister.ch/blog/smtp-source-ohne-postfix-installation) mostra come estrarre gli strumenti dall'RPM.

## Le principali opzioni di smtp-source

Per orientarsi, ecco le opzioni trattate in questo articolo, tradotte liberamente dalla manpage:

<details class="options-details">
<summary>Panoramica delle opzioni</summary>

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
| `-r n` | Numero di destinatari per messaggio (predefinito: 1), generazione dell'indirizzo come con `-N` |
| `-d` | Non disconnettersi dopo un messaggio, inviare il successivo sulla stessa connessione |
| `-c` | Mostra un contatore progressivo che aumenta a ogni `DATA` completato |
| `-w n` | Attesa fissa di n secondi tra i messaggi (per sessione) |
| `-v` | Output dettagliato per la ricerca degli errori |
| `host:port` | Destinazione della consegna tramite TCP; senza porta viene usata la porta smtp predefinita |

</details>

L'elenco completo, comprese le opzioni TLS, LMTP e di timing, è disponibile nella manpage di `smtp-source(1)`; la controparte per il lato ricevente è `smtp-sink(1)` e viene utilizzata più avanti nella valutazione.

## Come -N numera i destinatari

`-N` attiva un contatore per processo integrato nell'indirizzo del destinatario. Tre caratteristiche determinano la configurazione del test e tutte e tre sono documentate nel codice sorgente di `smtp-source.c`:

In primo luogo, la forma esatta dell'indirizzo dipende dalla versione di Postfix. Postfix 3.5, distribuito da RHEL 8, antepone il numero all'intero indirizzo (`RCPT TO:<%d%s>`): da `-t test@example.com` diventano `1test@example.com`, `2test@example.com` e così via, e il contatore inizia da 1. Le versioni attuali di Postfix aggiungono invece il numero alla fine della parte locale e iniziano da 0 (`test0@` fino a `test49999@`); per questa variante la manpage raccomanda l'indirizzamento con più (`-t 'test+@example.com'` diventa `test+0@` e successivi), affinché un sistema di destinazione con sottoindirizzamento assegni tutto alla stessa casella di posta. Prima dell'esecuzione estesa, verificate il formato con una manciata di email verso un `smtp-sink` oppure nel log della destinazione; da questo dipendono l'insieme atteso e il modello di ricerca per la valutazione.

In secondo luogo, il contatore è condiviso a livello di processo da tutte le sessioni parallele. Con `-s 8`, le otto sessioni assegnano insieme i numeri e ogni numero compare esattamente una volta. L'ordine tra le sessioni non è deterministico, ma la completezza dell'insieme dei numeri è garantita.

In terzo luogo, il valore iniziale non è configurabile: 1 con Postfix 3.5, 0 con le versioni attuali. Le email hanno quindi i numeri da 1 al numero totale definito da `-m`, oppure da 0 al totale meno 1, e l'insieme atteso per il confronto deve corrispondere.

## L'esecuzione del test in una chiamata

Il numero di email incluse nell'esecuzione non cambia la procedura; `-m` determina il totale e gli esempi di questo articolo usano 50'000 come segnaposto arbitrario.

```bash
smtp-source -c -d -N -s 8 -m 50000 -l 5120 \
  -f lasttest@example.com \
  -t test@example.com \
  gateway.example.com:25
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-c` | Contatore progressivo delle consegne completate come indicatore di avanzamento su una riga |
| `-d` | Le connessioni restano aperte per tutti i messaggi; senza `-d` viene stabilita una nuova connessione per ogni messaggio |
| `-N` | Numerazione dei destinatari: aggiunge il contatore per processo alla parte locale |
| `-s 8` | Otto sessioni SMTP parallele |
| `-m 50000` | Numero totale di messaggi, distribuiti tra le sessioni |
| `-l 5120` | Dimensione del messaggio in byte (senza header), qui 5 KB |
| `-f` | Indirizzo del mittente |
| `-t` | Indirizzo base del destinatario; `-N` lo trasforma in `1test@`, `2test@` e così via (Postfix 3.5), oppure `test0@`, `test1@` e così via (versioni attuali) |
| `gateway.example.com:25` | Host e porta di destinazione |

</details>

`-d` è determinante per il profilo di carico: senza questa opzione, `smtp-source` chiude la connessione dopo ogni messaggio e ne apre una nuova per il successivo; con `-d`, le otto connessioni restano aperte e consegnano in successione tutti i messaggi, come farebbe un mittente di massa.

Manca deliberatamente l'opzione `-v`, nota dai test funzionali: registra ogni singolo dialogo SMTP da `HELO` a `QUIT` e, in una grande esecuzione, genera centinaia di migliaia di righe di log senza valore aggiunto per la valutazione. `-c` fornisce invece il riepilogo, dal quale è possibile seguire in tempo reale l'avanzamento dell'esecuzione. Un `time` anteposto fornisce la durata complessiva per calcolare la velocità.

Prerequisito per l'intero approccio: il sistema di destinazione accetta gli indirizzi generati. Sono adatti un `smtp-sink`, un dominio catch-all, un dominio di scarto del provider oppure un gateway che risolva i destinatari solo dopo l'accettazione. Se invece la destinazione verifica ogni destinatario rispetto a una directory, rifiuta gli indirizzi numerati e resta soltanto la variante con l'oggetto.

## Impostare header personalizzati

Alcuni test di carico richiedono un header personalizzato, ad esempio come marcatore con cui il gateway riconosce le email di test o per applicare una regola. `smtp-source` non dispone di un'opzione per questo, ma `-F` legge da un file un messaggio completamente preformattato, nel quale può comparire qualsiasi header desiderato. Il file è composto dalle righe di header, una riga vuota e il body, con tutte le righe terminate da `\r\n`:

```bash
{ printf 'X-Lasttest: aktiv\r\n'
  printf 'Subject: Lasttest\r\n'
  printf '\r\n'
  head -c 5120 /dev/zero | tr '\0' 'x'
  printf '\r\n'; } > lasttest.eml
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `head -c 5120` | Emette i primi 5120 byte dell'input, qui da `/dev/zero` |
| `tr '\0' 'x'` | Sostituisce ogni byte nullo con il carattere `x`, generando così il testo di riempimento da 5 KB |
| `> lasttest.eml` | Scrive il messaggio composto nel file per `-F` |

</details>

```bash
smtp-source -c -d -N -s 8 -m 50000 -F lasttest.eml \
  -f lasttest@example.com \
  -t test@example.com \
  gateway.example.com:25
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-F datei` | Invia header e body invariati dal file; sostituisce il contenuto del messaggio generato |

</details>

Due conseguenze: `-F` sostituisce `-l` e `-S`, perché dimensione e oggetto provengono ora dal file (perciò entrambi devono esservi inclusi). `-N` resta invece attivo e i destinatari continuano a essere numerati; l'header è identico in tutti i messaggi, poiché proviene dal file fisso.

## Quante sessioni?

Il modo più affidabile per determinare il numero di sessioni adatto è misurare, usando esattamente le stesse opzioni previste per l'esecuzione principale: stessa sorgente del messaggio (lo stesso file `-F` oppure lo stesso `-l`), stesso mittente, stessa destinazione. Solo la quantità viene ridotta a 2'000 per livello e si varia `-s`. Una breve esecuzione di calibrazione con un numero crescente di sessioni mostra da quando sessioni aggiuntive non apportano più benefici:

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

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `date +%s%N` | Restituisce i secondi Unix seguiti direttamente dalla componente in nanosecondi come un unico numero |
| `-d` | Le connessioni restano aperte per tutti i messaggi del livello |
| `-N` | Numerazione dei destinatari tramite il contatore per processo |
| `-s "$s"` | Numero di sessioni parallele, da 1 a 32 per iterazione del ciclo |
| `-m 2000` | 2'000 messaggi per livello di misurazione |
| `-F lasttest.eml` | Lo stesso file di messaggio previsto per l'esecuzione principale |
| `-f` | Indirizzo del mittente |
| `-t '@blackhole.example.com'` | Indirizzo base del destinatario con parte locale vuota su un dominio di scarto |
| `gateway.example.com:25` | Host e porta di destinazione |

</details>

Due dettagli della chiamata: si rinuncia deliberatamente a `-c`, affinché tra le righe di misurazione non compaiano contatori in corso; il ciclo restituisce esattamente una riga di risultato per livello. Inoltre, la parte locale vuota in `-t` funziona bene con la numerazione su un dominio di scarto: con il contatore anteposto di Postfix 3.5 si ottengono indirizzi destinatario puramente numerici (`1@blackhole.example.com`, `2@…`), che mantengono chiara la valutazione nei log.

Nel dettaglio avviene quanto segue: il ciclo esterno percorre i numeri di sessione da 1 a 32, raddoppiandoli a ogni passaggio. Prima e dopo ogni esecuzione, `date +%s%N` registra l'ora corrente come un numero grande, ovvero i secondi Unix seguiti direttamente dalla componente in nanosecondi. Nel mezzo, `smtp-source` invia 2'000 messaggi (contenuto, header e dimensione provengono dal file `-F`) attraverso il rispettivo numero di connessioni parallele che, grazie a `-d`, restano aperte; il ciclo attende finché la chiamata non è completamente terminata. La riga `echo` converte la differenza temporale in una velocità: 2'000 email divisi per la durata in secondi, mentre la durata è espressa in nanosecondi. Da 2'000 per 10⁹ risulta quindi la costante `2000000000000`. L'aritmetica Bash `$(( ))` usa numeri interi e tronca le cifre decimali, cosa sufficientemente precisa per questa misurazione.

Tre indicazioni pratiche: `%N` fornisce i nanosecondi solo con GNU date (come su RHEL e sulla maggior parte dei sistemi Linux; BusyBox e macOS non lo supportano). L'esecuzione completa invia 6 × 2'000 = 12'000 email, che necessitano anch'esse di un indirizzo destinatario controllato, e la numerazione `-N` ricomincia dal valore iniziale a ogni chiamata. Se una chiamata di `smtp-source` si interrompe con un messaggio d'errore, la velocità di quella riga non ha significato; correggete prima la causa, poi misurate di nuovo.

L'output previsto è una riga per livello. Con valori di esempio inventati ma tipici, appare così:

```text
1 Sessions: 11 Mails/s
2 Sessions: 21 Mails/s
4 Sessions: 40 Mails/s
8 Sessions: 71 Mails/s
16 Sessions: 79 Mails/s
32 Sessions: 80 Mails/s
```

L'interpretazione: finché la velocità raddoppia approssimativamente con il numero di sessioni, le sessioni parallele coprono il tempo di attesa delle risposte della destinazione; il collo di bottiglia è quindi la latenza del percorso, non la capacità. Dal punto in cui la curva si appiattisce (nell'esempio tra 8 e 16 sessioni), il sistema di destinazione è saturo oppure la sorgente ha raggiunto il proprio limite. Scegliete il valore più piccolo per cui la velocità non aumenta più in modo significativo, nell'esempio quindi da 8 a 16; ulteriori sessioni aumentano solo il carico dovuto al parallelismo, non il throughput. Per l'esecuzione principale, dalla velocità misurata si può anche stimare subito la durata prevista: il totale definito da `-m` diviso per la velocità.

## Valutazione sul lato ricevente

Se sul sistema di destinazione è disponibile un destinatario di test dedicato, `smtp-sink` si occupa anche della registrazione:

```bash
smtp-sink -c -d "mails/%Y%m%d-%H%M%S." 0.0.0.0:2525 200
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-c` | Contatori progressivi invece del dialogo SMTP completo |
| `-d "mails/…"` | Nel sink: dump, non mantenimento della connessione. Scrive ogni messaggio accettato in un file separato (modello di nome tramite strftime), incluso un header `X-Rcpt-Args` con l'indirizzo del destinatario |
| `0.0.0.0:2525` | Rimane in ascolto su tutte le interfacce alla porta 2525 |
| `200` | Backlog: lunghezza massima della coda di connessioni in attesa secondo listen(2) |

</details>

Dopo l'esecuzione, estraete i numeri ricevuti e confrontateli con l'insieme atteso. Poiché i numeri non hanno zeri iniziali, entrambe le liste vengono portate a una lunghezza fissa prima del confronto, affinché l'ordinamento alfabetico di `comm` corrisponda a quello numerico. Il modello di ricerca corrisponde al formato di indirizzo di Postfix 3.5 (numero prima dell'indirizzo); per le versioni attuali usare invece `test[0-9]+@` e `seq` a partire da 0:

```bash
grep -rhoE '[0-9]+test@example\.com' mails/ | \
  grep -oE '^[0-9]+' | sort -u | \
  awk '{printf "%08d\n", $1}' | sort > empfangen.txt
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `grep -r` | Cerca ricorsivamente nella directory `mails/` |
| `grep -h` | Sopprime i nomi dei file prima delle corrispondenze |
| `grep -o` | Emette solo la parte dell'indirizzo corrispondente, non l'intera riga |
| `grep -E` | Espressioni regolari estese, qui per `[0-9]+` |
| `sort -u` | Ordina e rimuove i duplicati (ogni numero una volta) |
| `awk '{printf "%08d\n", $1}'` | Completa ogni numero con zeri iniziali fino a otto cifre |
| `sort` | Ordina i numeri completati per il confronto con `comm` |

</details>

```bash
seq 1 50000 | awk '{printf "%08d\n", $1}' | \
  comm -23 - empfangen.txt
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `seq 1 50000` | Genera l'insieme atteso dei numeri; il valore finale corrisponde al totale inviato definito da `-m` |
| `comm -23` | Sopprime la colonna 2 (solo nel file 2) e la colonna 3 (in entrambi); restano le righe presenti solo nell'insieme atteso |
| `-` | Legge la prima lista di confronto dalla pipe anziché da un file |
| `empfangen.txt` | Seconda lista di confronto: i numeri effettivamente ricevuti |

</details>

`comm -23` restituisce esattamente i numeri presenti nell'insieme atteso ma non nella lista di ricezione: le email mancanti. Un output vuoto significa consegna completa. Se alcuni numeri compaiono due volte (riconoscibile dalla differenza tra `sort` e `sort -u`), un sistema lungo il percorso ha duplicato il messaggio, e anche questo è un risultato rilevante.

Se la destinazione è un sistema vicino alla produzione invece di un smtp-sink, il suo logging assume il ruolo dei file di dump. Su un server Exchange, ad esempio, `Get-MessageTrackingLog -Recipients` oppure un filtro sull'indirizzo del destinatario fornisce i numeri arrivati; su un sistema Postfix, un `grep` su `to=` e sull'indirizzo base nel maillog. Questo è precisamente il vantaggio del numero nell'indirizzo: il destinatario compare in ogni tracciamento dei messaggi, mentre l'oggetto può mancare a seconda del sistema oppure deve prima essere attivato.

## Quando il numero deve essere nell'oggetto

Alcune valutazioni dipendono dall'oggetto, ad esempio quando il sistema di destinazione riscrive gli indirizzi dei destinatari o i log mostrano il destinatario soltanto mascherato. In tal caso resta la variante a ciclo: una chiamata `smtp-source` per email con `-m 1` e un oggetto incrementato dalla shell, distribuita su più worker paralleli con intervalli di numeri contigui.

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

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-s 1` | Una sessione per chiamata; il parallelismo è fornito dai quattro worker |
| `-m 1` | Esattamente un messaggio per chiamata, affinché l'oggetto possa essere impostato per ogni email |
| `-l 5120` | Dimensione del messaggio in byte (senza header), qui 5 KB |
| `-S "$(printf 'Lasttest %05d' "$i")"` | Oggetto con il numero progressivo completato a cinque cifre |
| `-f` / `-t` | Indirizzo del mittente e del destinatario |
| `gateway.example.com:25` | Host e porta di destinazione |

</details>

Il prezzo è una connessione completa per email: handshake TCP, banner, `HELO`, invio, `QUIT`. Questa esecuzione non misura quindi il throughput massimo del sistema di destinazione, ma un caso deliberatamente ad alta intensità di connessioni. Il numero di worker viene determinato in modo analogo all'esecuzione di calibrazione precedente, solo con il ciclo dei worker al posto di `-s`. Gli zeri iniziali nell'oggetto evitano la riformattazione necessaria per il confronto nella variante `-N`.

## Regole per test contro altri sistemi

Non appena il test lascia il proprio sistema, si applicano tre condizioni. Primo: il gestore del sistema di destinazione ne è informato e ha approvato la finestra temporale; per qualsiasi monitoraggio, un test di carico appare come un attacco o un'ondata di spam. Secondo: l'indirizzo del destinatario termina in modo controllato, in una casella di posta di test dedicata, in una regola di scarto sulla destinazione o in un dominio di scarto predisposto dal provider; gli indirizzi produttivi non devono essere usati in un test di carico. Terzo: prima dell'avvio viene definito un criterio di interruzione, ad esempio una coda in crescita sulla destinazione o un tasso di errore oltre una soglia, e qualcuno osserva questi valori durante l'esecuzione.

Con questi tre punti e la numerazione, al termine l'esecuzione non fornisce solo un valore di throughput, ma un'affermazione dimostrabile: quali email sono arrivate, quali mancano e dove sono state viste per l'ultima volta lungo il percorso.

## Fonti

1.  [Postfix: smtp-source(1)](https://www.postfix.org/smtp-source.1.html): Manpage del generatore di carico; descrive il comportamento di `-N` nella versione attuale (contatore nella parte locale, indirizzamento con più).

2.  [Codice sorgente Postfix 3.5.8: smtp-source.c](https://github.com/vdukhovni/postfix/blob/v3.5.8/postfix/src/smtpstone/smtp-source.c): documenta, per la versione RHEL 8, l'anteposizione del numero (`RCPT TO:<%d%s>`) con valore iniziale 1; nella [versione attuale](https://github.com/vdukhovni/postfix/blob/master/postfix/src/smtpstone/smtp-source.c) il numero viene invece aggiunto alla parte locale, a partire da 0.

3.  [Postfix: smtp-sink(1)](https://www.postfix.org/smtp-sink.1.html): Manpage del destinatario di test con le opzioni di dump e gli header X registrati.

4.  [GNU Coreutils: comm](https://www.gnu.org/software/coreutils/manual/html_node/comm-invocation.html): confronto di insiemi tra due liste ordinate, qui per confrontare i numeri attesi e ricevuti.
