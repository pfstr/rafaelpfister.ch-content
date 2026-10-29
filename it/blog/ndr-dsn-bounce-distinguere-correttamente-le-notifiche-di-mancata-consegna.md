---
title: "NDR, DSN, Bounce: distinguere correttamente le notifiche di mancata consegna"
navTitle: "NDR e Bounce"
description: "NDR, DSN, Bounce, Reject, Backscatter: i termini relativi alle consegne non riuscite sono spesso usati come sinonimi, ma indicano cose diverse. Cosa definiscono gli RFC, chi genera quale messaggio, com'è strutturata una DSN e perché la differenza tra Reject e Bounce determina il Backscatter."
date: "2026-08-28"
kategorie: "SMTP e flusso di posta"
timeToRead: "10 min di lettura"
themen:
  - smtp-mailflow
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "ndr-dsn-bounce-distinguere-correttamente-le-notifiche-di-mancata-consegna"
translationId: "article-5c5164049a129fa4"
translationOf: ndr-dsn-bounce-unterschiede
translationSourceHash: 1500e27283dc4c53456810f133d12d81e1f60d7e874b6f3450b0a50fb47d4cf2
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:20:10.257Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/ndr-dsn-bounce-distinguere-correttamente-le-notifiche-di-mancata-consegna
---

# NDR, DSN, Bounce: distinguere correttamente le notifiche di mancata consegna

Un'e-mail non arriva e nel ticket si parla alternativamente di «Bounce», «NDR», «Mailer-Daemon» o «messaggio di errore del server». Nella quotidianità amministrativa questi termini vengono usati come sinonimi, benché indichino cose diverse: un Reject durante la sessione SMTP non è un'e-mail di ritorno, una notifica di ritardo non è un errore di consegna e una conferma di lettura non ha nulla a che vedere con la mancata consegna. Chi distingue chiaramente i termini individua prima la causa, perché ogni tipo di messaggio dice qualcosa di diverso sul punto del percorso di trasporto in cui si trova il problema e su chi può risolverlo.

## DSN: il termine generale degli RFC

Il termine formale generale è Delivery Status Notification (DSN), definito negli RFC da 3461 a 3464. Una DSN è un'e-mail generata automaticamente che informa il mittente sullo stato di consegna del suo messaggio. Punto decisivo: una DSN non segnala solo gli insuccessi. Il campo `Action` nella parte leggibile dalla macchina prevede cinque valori:

| Action | Significato |
|---|---|
| `failed` | Consegna definitivamente non riuscita; l'e-mail non verrà ritentata |
| `delayed` | Consegna ritardata; il server continua a tentare |
| `delivered` | Consegnata con successo (conferma di consegna, solo su richiesta esplicita) |
| `relayed` | Inoltrata a un server che non genera DSN |
| `expanded` | Consegnata a una lista di distribuzione ed espansa |

La notifica di mancata consegna è quindi solo un caso particolare: una DSN con `Action: failed`. Microsoft chiama proprio questo caso particolare Non-Delivery Report (NDR). Il termine NDR proviene dal mondo Exchange, ma nel frattempo è diventato di uso comune indipendentemente dal produttore. Volendo essere precisi: ogni NDR è una DSN, ma non ogni DSN è un NDR.

La notifica di ritardo (`Action: delayed`) merita particolare attenzione, poiché nel supporto viene regolarmente scambiata per un errore di consegna. Un oggetto tipico è «Delivery delayed» o «Consegna ritardata». L'e-mail si trova ancora nella coda del server mittente, che continua a tentare, solitamente per uno o due giorni. Solo alla scadenza della durata della coda segue l'NDR definitivo. Un utente che invia nuovamente l'e-mail in seguito a una notifica di ritardo genera duplicati non appena il sistema di destinazione torna raggiungibile.

## Reject o Bounce: la distinzione più importante

Prima di proseguire con gli altri termini, va spiegato lo snodo tecnico centrale, poiché è qui che si decide quale server genera una notifica.

**Reject (rifiuto nella sessione):** Il server ricevente rifiuta l'e-mail già durante la sessione SMTP, con un codice di risposta 5xx a `RCPT TO` o dopo `DATA`. Non accetta mai l'e-mail e non genera alcuna e-mail di risposta. L'obbligo di informare il mittente spetta al server che ha ricevuto il messaggio per l'invio: l'MTA mittente rileva la risposta 5xx e genera quindi l'NDR per il proprio utente locale. L'NDR letto dall'utente proviene in questo caso dal proprio server, ma cita il messaggio di errore della controparte.

**Bounce (accettazione con successivo messaggio di ritorno):** Il server ricevente accetta l'e-mail con `250 OK` e solo in seguito rileva di non poterla consegnare, ad esempio perché la casella di posta non esiste, la quota è piena o un server a valle rifiuta. A questo punto è responsabile del messaggio e deve inviare lui stesso una DSN al mittente. Questa e-mail di ritorno successiva è il Bounce in senso stretto.

Per la ricerca guasti, la differenza è immediatamente utile: se nell'NDR il sistema generante è il proprio server, l'e-mail è stata rifiutata durante la sessione oppure non è mai uscita. Se un server esterno figura come mittente della notifica, la controparte ha inizialmente accettato l'e-mail e il problema si trova oltre il suo punto di accettazione, invisibile al mittente.

Dal contesto del marketing provengono altri due termini relativi ai Bounce, assenti da qualsiasi RFC: Hard Bounce per gli errori definitivi (5xx, `Action: failed`) e Soft Bounce per quelli temporanei (4xx, `Action: delayed`). Per le piattaforme di mailing la distinzione è fondamentale, poiché gli Hard Bounce dovrebbero portare alla pulizia immediata delle liste. Tecnicamente, si tratta degli stessi meccanismi descritti sopra.

## I termini in sintesi

| Termine | Cos'è | Chi genera la notifica | Standard |
|---|---|---|---|
| DSN | Termine generale: messaggio di stato sulla consegna (failed, delayed, delivered, relayed, expanded) | L'MTA responsabile dell'e-mail | RFC da 3461 a 3464 |
| NDR | DSN con `Action: failed`; termine Microsoft per la notifica di mancata consegna | MTA mittente (dopo Reject) o MTA ricevente (dopo accettazione) | RFC 3464, documentazione Microsoft |
| Reject | Rifiuto 5xx nella sessione SMTP in corso; nessuna e-mail separata | Nessuno; l'MTA mittente ne ricava un NDR | RFC 5321 |
| Bounce | E-mail di ritorno dopo un'accettazione già avvenuta | MTA ricevente | RFC 5321, RFC 3464 |
| Hard/Soft Bounce | Classificazione di marketing: definitivo (5xx) o temporaneo (4xx) | come Bounce | nessun RFC |
| Notifica di ritardo | DSN con `Action: delayed`; l'e-mail è ancora nella coda | MTA mittente o relay | RFC 3464 |
| Backscatter | NDR inviati a indirizzi mittente falsificati, solitamente innescati da spam | MTA riceventi configurati in modo errato | nessun RFC, termine anti-abuso |
| MDN / conferma di lettura | Notifica dell'apertura o eliminazione da parte del destinatario | Client di posta del destinatario | RFC 8098 |
| Risposta automatica di assenza | Risposta automatica di una casella di posta raggiunta | Server di casella di posta o groupware | RFC 3834 |

## Struttura di una DSN

Le DSN conformi allo standard usano il tipo MIME `multipart/report; report-type=delivery-status` con tre parti: una spiegazione leggibile dall'utente, una parte leggibile dalla macchina di tipo `message/delivery-status` e, facoltativamente, il messaggio originale o le sue intestazioni. La parte leggibile dalla macchina è la più preziosa per la diagnosi, poiché i suoi campi sono standardizzati:

```text
Reporting-MTA: dns; mail01.example.net
Received-From-MTA: dns; client.example.org

Final-Recipient: rfc822; max.muster@example.com
Action: failed
Status: 5.1.1
Remote-MTA: dns; mx.example.com
Diagnostic-Code: smtp; 550 5.1.1 <max.muster@example.com>:
    Recipient address rejected: User unknown
```

| Campo | Significato |
|---|---|
| `Reporting-MTA` | Il server che ha generato questa DSN; primo indizio sulla responsabilità |
| `Final-Recipient` | L'indirizzo del destinatario a cui si riferisce lo stato (un blocco per destinatario) |
| `Action` | Uno dei cinque valori di stato (failed, delayed, delivered, relayed, expanded) |
| `Status` | Enhanced Status Code secondo RFC 3463, ad es. `5.1.1` |
| `Remote-MTA` | La controparte con cui l'MTA di reporting ha comunicato |
| `Diagnostic-Code` | La risposta SMTP letterale della controparte; spesso la riga più significativa |

Una DSN viene sempre inviata con un mittente envelope vuoto (`MAIL FROM:<>`). Non è una negligenza, bensì una prescrizione dell'RFC 5321: il mittente vuoto impedisce che a una DSN non consegnabile segua un'altra DSN e che due server si inviino messaggi di errore all'infinito. Ne deriva una regola di configurazione: un sistema di posta non deve rifiutare indiscriminatamente le e-mail con mittente envelope vuoto, altrimenti le notifiche di mancata consegna legittime non raggiungeranno mai i propri utenti.

Exchange e Exchange Online rispettano lo standard per il formato, ma presentano il contenuto in una forma propria: l'utente vede una pagina elaborata con una spiegazione in chiaro e, sotto, «Generating server» (corrisponde a `Reporting-MTA`) e i dati raw. Per la diagnosi vale sempre la pena guardare questa parte tecnica inferiore.

## Leggere gli Enhanced Status Codes

Nel campo `Status` e solitamente anche in `Diagnostic-Code` compare un codice in tre parti secondo RFC 3463: classe.soggetto.dettaglio. La classe indica la definitività, soggetto e dettaglio la causa:

| Intervallo di codici | Significato |
|---|---|
| `2.x.x` | Successo (solo nelle conferme di consegna) |
| `4.x.x` | Errore temporaneo; il server ritenta |
| `5.x.x` | Errore definitivo; nessun ulteriore tentativo |
| `x.1.x` | Problema di indirizzamento, ad es. `5.1.1` destinatario sconosciuto, `5.1.10` dominio senza MX |
| `x.2.x` | Problema della casella di posta, ad es. `5.2.2` casella piena, `5.2.3` messaggio troppo grande per la casella |
| `x.3.x` | Problema del sistema di destinazione, ad es. `4.3.2` il sistema non accetta nulla al momento |
| `x.4.x` | Rete e routing, ad es. `4.4.1` nessuna risposta, `4.4.7` durata della coda scaduta |
| `x.5.x` | Errore di protocollo nel dialogo SMTP |
| `x.7.x` | Criteri e sicurezza, ad es. `5.7.1` relay rifiutato o rifiuto per criteri, `5.7.26` autenticazione mancante (SPF/DKIM/DMARC) |

Il classico codice di risposta SMTP a tre cifre (ad esempio `550`) e l'Enhanced Status Code compaiono spesso insieme in una riga: `550 5.7.1 ...`. Il codice a tre cifre determina il comportamento del protocollo del server mittente, mentre il codice esteso fornisce l'indicazione diagnostica. In caso di contraddizioni tra codice e testo libero, il testo libero della controparte è spesso la fonte più precisa, poiché molti sistemi impostano codici generici e indicano la vera causa nel commento, comprese le ID di riferimento per il supporto della controparte.

Da notare: i rifiuti `5.7.x` da parte di filtri di reputazione e contenuto spesso forniscono intenzionalmente poche informazioni. Chi qui guarda solo il codice cerca nel posto sbagliato; la blocklist o il produttore del filtro indicato nel testo libero porta più rapidamente alla soluzione.

## Backscatter: il tipo dannoso di Bounce

Il Backscatter si verifica quando un server accetta inizialmente spam con mittente falsificato e invia poi un NDR all'indirizzo contraffatto. L'NDR raggiunge quindi una persona estranea, il cui indirizzo è stato abusato dallo spammer. Durante grandi ondate di spam, le persone coinvolte ricevono migliaia di NDR per e-mail che non hanno mai inviato, e i server che generano tali NDR in massa finiscono essi stessi sulle blocklist, ad esempio nella lista Backscatterer di UCEPROTECT.

Il rimedio deriva direttamente dalla distinzione tra Reject e Bounce: tutto ciò che può essere rifiutato va rifiutato durante la sessione SMTP, non rinviato dopo l'accettazione. In concreto, ciò significa validare i destinatari nel punto di accettazione più esterno (l'Edge Gateway conosce gli indirizzi validi, tramite confronto con la directory o Recipient Callout, anziché accettare tutto e lasciare che il fallimento avvenga internamente), rifiutare spam e malware durante la sessione invece di inviare NDR per la quarantena, e rinunciare agli NDR per i messaggi classificati come spam. Un Reject non genera Backscatter, poiché con un mittente falsificato la risposta 5xx arriva al server dello spammer, che non ne genera un NDR per la vittima.

## Cosa non è una notifica di mancata consegna

Tre tipi di messaggi finiscono regolarmente insieme nei ticket, ma non ne fanno parte:

**MDN (Message Disposition Notification, RFC 8098):** La conferma di lettura. Non viene generata dal sistema di trasporto, bensì dal client di posta del destinatario, e segnala l'apertura o l'eliminazione del messaggio, non la sua consegna. Il tipo MIME è quindi `multipart/report; report-type=disposition-notification`. Una conferma di lettura assente non dice nulla sulla consegna; la maggior parte dei client chiede all'utente oppure sopprime completamente gli MDN.

**Risposte automatiche di assenza e autoresponder (RFC 3834):** Una risposta automatica di assenza dimostra il contrario di un errore di consegna, perché presuppone che l'e-mail abbia raggiunto la casella di posta. Nelle descrizioni dei ticket («ricevo una risposta automatica, la mia e-mail arriva?») vale la pena chiedere quale messaggio sia effettivamente presente.

**Notifiche di quarantena:** Messaggi come il digest di quarantena di Microsoft 365 o di un gateway informano il destinatario sulle e-mail trattenute. Vengono inviati al destinatario, non al mittente, e non seguono alcuno standard DSN. In questo scenario il mittente spesso non riceve nulla, il che spiega i casi in cui un'e-mail «scompare senza messaggio di errore».

## Checklist per la diagnosi

Se è presente una notifica, chiarire nell'ordine seguente:

1. Di che tipo si tratta: NDR (`Action: failed`), ritardo (`Action: delayed`), MDN, autoresponder o avviso di quarantena? In caso di notifica di ritardo: attendere, non inviare nuovamente.
2. Chi ha generato la notifica (`Reporting-MTA` o «Generating server»)? Il proprio server indica Reject o errore interno, un server esterno indica accettazione con successivo fallimento dalla controparte.
3. Cosa indicano Status e Diagnostic-Code? La classe 4 rispetto alla classe 5 separa il temporaneo dal definitivo, il soggetto (`x.1` indirizzo, `x.2` casella di posta, `x.4` rete, `x.7` criterio) delimita la causa, mentre il testo libero della controparte fornisce i dettagli.
4. Non è presente alcuna notifica, pur non arrivando l'e-mail: verificare il Message Tracking sul proprio sistema e considerare la quarantena o il filtraggio silenzioso dalla controparte.

I contributi sul [Message Tracking e sulla diagnostica SMTP nel Generatore di comandi](https://rafaelpfister.ch/tools/command-builder) e sul [Mail-Header-Analyzer](https://rafaelpfister.ch/tools/header-analyzer) mostrano come riprodurre miratamente i singoli percorsi di consegna e valutare il percorso di trasporto di un'e-mail arrivata.

## Cronologia delle modifiche

- **25 ottobre 2026:** Corretto: impostato il link del Mail-Header-Analyzer all'indirizzo corretto. Aggiornato il link alle Fonti al nuovo indirizzo di Microsoft Learn.

## Fonti

1.  [RFC 3461: SMTP Service Extension for Delivery Status Notifications](https://www.rfc-editor.org/rfc/rfc3461): estensione SMTP con cui i mittenti possono richiedere e controllare le DSN.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): definizione dei codici di stato in tre parti (classe.soggetto.dettaglio).

3.  [RFC 3464: An Extensible Message Format for Delivery Status Notifications](https://www.rfc-editor.org/rfc/rfc3464): struttura della DSN come multipart/report, campi quali Action, Status e Diagnostic-Code.

4.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): regole di base su codici di risposta, trasferimento della responsabilità all'accettazione e mittente envelope vuoto per i messaggi di errore.

5.  [RFC 8098: Message Disposition Notification](https://www.rfc-editor.org/rfc/rfc8098): standard per le conferme di lettura, per distinguerle dalle DSN.

6.  [RFC 3834: Recommendations for Automatic Responses to Electronic Mail](https://www.rfc-editor.org/rfc/rfc3834): regole per gli autoresponder come le risposte automatiche di assenza.

7.  [Microsoft Learn: Email non-delivery reports and SMTP errors in Exchange Online](https://learn.microsoft.com/en-us/troubleshoot/exchange/email-delivery/ndr/non-delivery-reports-in-exchange-online): struttura NDR ed elenco dei codici dal punto di vista di Exchange Online.

8.  [UCEPROTECT Backscatterer](https://www.backscatterer.org/): blocklist per sistemi che generano Backscatter; spiega i criteri di inserimento nella lista.
