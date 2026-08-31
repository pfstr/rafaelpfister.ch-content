---
title: "NDR, DSN, Bounce: distinguere correttamente le notifiche di mancata consegna"
navTitle: "NDR e Bounce"
description: "NDR, DSN, Bounce, Reject, Backscatter: i termini relativi alle consegne non riuscite sono spesso usati come sinonimi, ma indicano cose diverse. Cosa definiscono le RFC, chi genera quale messaggio, come è strutturata una DSN e perché la differenza tra Reject e Bounce determina il Backscatter."
date: "2026-08-28"
kategorie: "SMTP e flusso di posta"
timeToRead: "10 min di lettura"
themen:
  - smtp-mailflow
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "ndr-dsn-bounce-distinguere-correttamente-le-notifiche-di-mancata-consegna"
translationId: "article-5c5164049a129fa4"
aiPrompt: |
  Du bist mein Mailflow-Assistent. Ich füge dir gleich eine Unzustellbarkeitsmeldung (NDR/DSN) ein. Analysiere sie Schritt für Schritt: 1. Welcher Server hat die Meldung erzeugt (Reporting-MTA bzw. Generating server)? 2. Wurde die Mail in der SMTP-Session abgewiesen oder nach Annahme zurückgeschickt? 3. Was bedeuten SMTP-Antwortcode und Enhanced Status Code (RFC 3463) konkret? 4. Liegt die Ursache beim Absender, beim Empfänger oder auf dem Transportweg? 5. Welche nächsten Diagnose-Schritte empfiehlst du?
translationOf: ndr-dsn-bounce-unterschiede
url: https://rafaelpfister.ch/it/blog/ndr-dsn-bounce-distinguere-correttamente-le-notifiche-di-mancata-consegna
translationSourceHash: e526de6f4a454b4f4975eac3e8a406ab5b30314c624bf12c69f87bec99fdd0e7
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:32:56.683Z
translationReview: automatic
---

# NDR, DSN, Bounce: distinguere correttamente le notifiche di mancata consegna

Un’email non arriva e nel ticket si legge alternativamente «Bounce», «NDR», «Mailer-Daemon» o «messaggio di errore del server». Nella quotidianità amministrativa questi termini vengono usati come sinonimi, sebbene indichino cose diverse: un Reject durante la sessione SMTP non è un’email di ritorno, una notifica di ritardo non è un errore di consegna e una conferma di lettura non ha nulla a che fare con la mancata consegna. Chi distingue chiaramente i termini individua più rapidamente la causa, perché ogni tipo di messaggio indica qualcosa di diverso su dove si trova il problema nel percorso di trasporto e chi può risolverlo.

## DSN: il termine generale definito dalle RFC

Il termine formale generale è Delivery Status Notification (DSN), definito nelle RFC da 3461 a 3464. Una DSN è un’email generata automaticamente che informa il mittente sullo stato di consegna del suo messaggio. Un punto decisivo: una DSN non segnala solo gli insuccessi. Il campo `Action` nella parte leggibile dalla macchina prevede cinque valori:

| Action | Significato |
|---|---|
| `failed` | Consegna definitivamente non riuscita; l’email non verrà ritentata |
| `delayed` | Consegna ritardata; il server continua a tentare |
| `delivered` | Consegnata correttamente (conferma di consegna, solo su richiesta esplicita) |
| `relayed` | Inoltrata a un server che non genera DSN |
| `expanded` | Consegnata a una lista di distribuzione ed espansa |

La notifica di mancata consegna è quindi solo un caso particolare: una DSN con `Action: failed`. Microsoft chiama proprio questo caso particolare Non-Delivery Report (NDR). Il termine NDR proviene dal mondo Exchange, ma ormai è comunemente usato indipendentemente dal produttore. Volendo essere precisi: ogni NDR è una DSN, ma non ogni DSN è un NDR.

La notifica di ritardo (`Action: delayed`) merita particolare attenzione, perché nel supporto viene regolarmente fraintesa come errore di consegna. Un oggetto tipico è «Delivery delayed» o «Consegna ritardata». L’email si trova ancora nella coda del server mittente, che continua a tentare, solitamente per uno o due giorni. Solo alla scadenza della durata di vita della coda segue l’NDR definitivo. Un utente che invia nuovamente l’email in seguito a una notifica di ritardo genera duplicati non appena il sistema di destinazione torna raggiungibile.

## Reject o Bounce: la distinzione più importante

Prima di introdurre gli altri termini, occorre spiegare il punto tecnico centrale, perché da esso dipende quale server genera un messaggio.

**Reject (rifiuto durante la sessione):** Il server ricevente rifiuta l’email già durante la sessione SMTP, con un codice di risposta 5xx a `RCPT TO` oppure dopo `DATA`. Non accetta mai l’email e non genera autonomamente alcuna email di notifica. L’obbligo di informare il mittente spetta al server che ha immesso il messaggio: l’MTA mittente rileva la risposta 5xx e genera quindi l’NDR per il suo utente locale. In questo caso, l’NDR letto dall’utente proviene dal proprio server, ma cita il messaggio di errore della controparte.

**Bounce (accettazione con messaggio di ritorno successivo):** Il server ricevente accetta l’email con `250 OK` e solo successivamente rileva di non poterla consegnare, ad esempio perché la casella non esiste, la quota è piena o un server a valle la rifiuta. A questo punto è responsabile del messaggio e deve inviare esso stesso una DSN al mittente. Questa email di ritorno successiva è il Bounce in senso stretto.

Per la ricerca guasti, la differenza è immediatamente utile: se nell’NDR il proprio server risulta come sistema generante, l’email è stata rifiutata nella sessione oppure non è mai uscita. Se un server esterno è indicato come mittente della notifica, la controparte ha inizialmente accettato l’email e il problema si trova dopo il suo punto di accettazione, invisibile al mittente.

Dal contesto del marketing provengono altri due termini relativi ai Bounce, non presenti in alcuna RFC: Hard Bounce per gli errori definitivi (5xx, `Action: failed`) e Soft Bounce per quelli temporanei (4xx, `Action: delayed`). Per le piattaforme di mailing la distinzione è essenziale, perché gli Hard Bounce dovrebbero portare alla pulizia immediata della lista. Dal punto di vista tecnico, si tratta degli stessi meccanismi descritti sopra.

## Panoramica dei termini

| Termine | Che cos’è | Chi genera la notifica | Standard |
|---|---|---|---|
| DSN | Termine generale: notifica di stato della consegna (failed, delayed, delivered, relayed, expanded) | L’MTA responsabile dell’email | RFC da 3461 a 3464 |
| NDR | DSN con `Action: failed`; termine Microsoft per la notifica di mancata consegna | MTA mittente (dopo Reject) o MTA ricevente (dopo accettazione) | RFC 3464, documentazione Microsoft |
| Reject | Rifiuto 5xx nella sessione SMTP in corso; nessuna email autonoma | Nessuno; l’MTA mittente ne ricava un NDR | RFC 5321 |
| Bounce | Email di ritorno dopo un’accettazione già avvenuta | MTA ricevente | RFC 5321, RFC 3464 |
| Hard/Soft Bounce | Classificazione di marketing: definitivo (5xx) vs. temporaneo (4xx) | come Bounce | nessuna RFC |
| Notifica di ritardo | DSN con `Action: delayed`; l’email è ancora nella coda | MTA mittente o relay | RFC 3464 |
| Backscatter | NDR inviati a indirizzi mittente falsificati, solitamente causati da spam | MTA riceventi configurati in modo errato | nessuna RFC, termine anti-abuso |
| MDN / conferma di lettura | Notifica della visualizzazione o eliminazione da parte del destinatario | Client di posta del destinatario | RFC 8098 |
| Notifica di assenza | Risposta automatica da una casella raggiunta | Server della casella o groupware | RFC 3834 |

## Struttura di una DSN

Le DSN conformi allo standard usano il tipo MIME `multipart/report; report-type=delivery-status` con tre parti: una spiegazione leggibile dall’uomo, una parte leggibile dalla macchina di tipo `message/delivery-status` e, facoltativamente, il messaggio originale o le sue intestazioni. La parte leggibile dalla macchina è la più preziosa per la diagnosi, perché i suoi campi sono standardizzati:

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
| `Final-Recipient` | L’indirizzo del destinatario a cui si riferisce lo stato (un blocco per destinatario) |
| `Action` | Uno dei cinque valori di stato (failed, delayed, delivered, relayed, expanded) |
| `Status` | Enhanced Status Code secondo RFC 3463, ad esempio `5.1.1` |
| `Remote-MTA` | La controparte con cui l’MTA di reporting ha comunicato |
| `Diagnostic-Code` | La risposta SMTP letterale della controparte; spesso la riga più significativa |

Una DSN viene sempre inviata con mittente Envelope vuoto (`MAIL FROM:<>`). Non è una negligenza, ma un requisito della RFC 5321: il mittente vuoto impedisce che una DSN non consegnabile generi un’altra DSN e che due server si inviino messaggi di errore all’infinito. Ne consegue una regola di configurazione: un sistema di posta non deve rifiutare indiscriminatamente le email con mittente Envelope vuoto, altrimenti le notifiche legittime di mancata consegna non raggiungeranno mai i propri utenti.

Exchange e Exchange Online rispettano lo standard per il formato, ma presentano il contenuto in una rappresentazione propria: l’utente vede una pagina elaborata con una spiegazione in chiaro; sotto sono riportati «Generating server» (corrisponde a `Reporting-MTA`) e i dati raw. Per la diagnosi vale sempre la pena consultare questa sezione tecnica inferiore.

## Leggere gli Enhanced Status Codes

Nel campo `Status` e solitamente anche in `Diagnostic-Code` è presente un codice in tre parti secondo RFC 3463: classe.soggetto.dettaglio. La classe indica la natura vincolante, soggetto e dettaglio la causa:

| Intervallo di codici | Significato |
|---|---|
| `2.x.x` | Successo (solo nelle conferme di consegna) |
| `4.x.x` | Errore temporaneo; il server ritenta |
| `5.x.x` | Errore definitivo; nessun ulteriore tentativo |
| `x.1.x` | Problema di indirizzamento, ad esempio destinatario sconosciuto `5.1.1`, dominio senza MX `5.1.10` |
| `x.2.x` | Problema della casella, ad esempio casella piena `5.2.2`, messaggio troppo grande per la casella `5.2.3` |
| `x.3.x` | Problema del sistema di destinazione, ad esempio il sistema non accetta nulla al momento `4.3.2` |
| `x.4.x` | Rete e routing, ad esempio nessuna risposta `4.4.1`, durata di vita della coda scaduta `4.4.7` |
| `x.5.x` | Errore di protocollo nel dialogo SMTP |
| `x.7.x` | Criteri e sicurezza, ad esempio relay negato o rifiuto per criteri `5.7.1`, autenticazione mancante (SPF/DKIM/DMARC) `5.7.26` |

Il classico codice di risposta SMTP a tre cifre (ad esempio `550`) e l’Enhanced Status Code compaiono spesso insieme su una riga: `550 5.7.1 ...`. Il codice a tre cifre controlla il comportamento di protocollo del server mittente, mentre il codice esteso fornisce l’informazione diagnostica. In caso di contraddizioni tra codice e testo libero, il testo libero della controparte è spesso la fonte più precisa, poiché molti sistemi impostano codici generici e riportano la causa effettiva nel commento, comprese le ID di riferimento per il supporto della controparte.

Da notare: i rifiuti `5.7.x` da parte dei filtri di reputazione e contenuto spesso forniscono volutamente poche informazioni. Chi guarda solo il codice cerca nel posto sbagliato; la blocklist o il produttore del filtro indicati nel testo libero conducono più rapidamente alla soluzione.

## Backscatter: il tipo dannoso di Bounce

Il Backscatter nasce quando un server accetta prima spam con mittente falsificato e poi invia un NDR all’indirizzo falsificato. L’NDR raggiunge quindi una persona non coinvolta, il cui indirizzo è stato abusato dallo spammer. Durante grandi ondate di spam, le persone coinvolte ricevono migliaia di NDR per email che non hanno mai inviato e i server che generano questi NDR in massa finiscono essi stessi nelle blocklist (ad esempio nella lista Backscatterer di UCEPROTECT).

Il rimedio deriva direttamente dalla distinzione Reject-Bounce: tutto ciò che può essere rifiutato deve essere rifiutato durante la sessione SMTP, non rispedito dopo l’accettazione. In concreto, ciò significa validazione dei destinatari nel punto di accettazione più esterno (l’Edge Gateway conosce gli indirizzi validi, tramite confronto con la directory o Recipient Callout, anziché accettare tutto e fallire internamente), rifiuto di spam e malware durante la sessione invece di NDR di quarantena, e rinuncia agli NDR per messaggi classificati come spam. Un Reject non genera Backscatter, perché con un mittente falsificato la risposta 5xx arriva al server dello spammer, che non ne genera un NDR verso la vittima.

## Cosa non è una notifica di mancata consegna

Tre tipi di messaggi finiscono regolarmente nello stesso calderone nei ticket, ma non ne fanno parte:

**MDN (Message Disposition Notification, RFC 8098):** La conferma di lettura. Non viene generata dal sistema di trasporto, ma dal client di posta del destinatario, e segnala la visualizzazione o l’eliminazione del messaggio, non la sua consegna. Il tipo MIME si chiama di conseguenza `multipart/report; report-type=disposition-notification`. Una conferma di lettura assente non dice nulla sulla consegna; la maggior parte dei client chiede all’utente o sopprime del tutto gli MDN.

**Notifiche di assenza e autoresponder (RFC 3834):** Una notifica di assenza dimostra il contrario di un errore di consegna, perché presuppone che l’email abbia raggiunto la casella. Nelle descrizioni dei ticket («ricevo una risposta automatica, la mia email arriva?») è utile chiedere quale messaggio sia effettivamente presente.

**Notifiche di quarantena:** Messaggi come il digest di quarantena di Microsoft 365 o di un gateway informano il destinatario sulle email trattenute. Sono inviati al destinatario, non al mittente, e non seguono alcuno standard DSN. In questo scenario, il mittente spesso non riceve nulla, il che spiega i casi in cui un’email «scompare senza messaggio di errore».

## Checklist per la diagnosi

Se è presente una notifica, chiarite nell’ordine seguente:

1. Di quale tipo si tratta: NDR (`Action: failed`), ritardo (`Action: delayed`), MDN, autoresponder o avviso di quarantena? In caso di notifica di ritardo: attendere, non inviare di nuovo.
2. Chi ha generato la notifica (`Reporting-MTA` o «Generating server»)? Il proprio server indica Reject o errore interno, un server esterno indica accettazione con successivo insuccesso presso la controparte.
3. Cosa indicano status e Diagnostic-Code? La classe 4 rispetto alla classe 5 distingue il temporaneo dal definitivo, il soggetto (`x.1` indirizzo, `x.2` casella, `x.4` rete, `x.7` criterio) delimita la causa e il testo libero della controparte fornisce i dettagli.
4. Se manca qualsiasi notifica nonostante l’email non arrivi: verificare il Message Tracking sul proprio sistema e considerare la quarantena o il filtraggio silenzioso presso la controparte.

I contributi sul [Message Tracking e sulla diagnosi SMTP nel generatore di comandi](https://rafaelpfister.ch/tools/command-builder) e il [Mail Header Analyzer](https://rafaelpfister.ch/tools/mail-header-analyzer) mostrano come riprodurre in modo mirato i singoli percorsi di consegna e analizzare il percorso di trasporto di un’email arrivata.

## Fonti

1.  [RFC 3461: SMTP Service Extension for Delivery Status Notifications](https://www.rfc-editor.org/rfc/rfc3461): estensione SMTP con cui i mittenti possono richiedere e controllare le DSN.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): definizione dei codici di stato in tre parti (classe.soggetto.dettaglio).

3.  [RFC 3464: An Extensible Message Format for Delivery Status Notifications](https://www.rfc-editor.org/rfc/rfc3464): struttura della DSN come multipart/report, campi quali Action, Status e Diagnostic-Code.

4.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): regole fondamentali sui codici di risposta, trasferimento della responsabilità all’accettazione e mittente Envelope vuoto per i messaggi di errore.

5.  [RFC 8098: Message Disposition Notification](https://www.rfc-editor.org/rfc/rfc8098): standard per le conferme di lettura, per distinguerle dalle DSN.

6.  [RFC 3834: Recommendations for Automatic Responses to Electronic Mail](https://www.rfc-editor.org/rfc/rfc3834): regole per gli autoresponder quali le notifiche di assenza.

7.  [Microsoft Learn: Email non-delivery reports and SMTP errors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/non-delivery-reports-in-exchange-online/non-delivery-reports-in-exchange-online): struttura degli NDR ed elenco dei codici dal punto di vista di Exchange Online.

8.  [UCEPROTECT Backscatterer](https://www.backscatterer.org/): blocklist per sistemi che generano Backscatter; spiega i criteri di inserimento in lista.
