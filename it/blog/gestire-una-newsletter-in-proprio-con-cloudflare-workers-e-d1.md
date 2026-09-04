---
title: "Gestire una newsletter propria con Cloudflare Workers e D1"
navTitle: "Newsletter su Workers"
description: "Il template open source mette a disposizione iscrizione, cancellazione, coda e database nel proprio account Cloudflare. Un pulsante di deploy configura Worker, D1 e CI senza server locale."
date: "2026-07-22"
kategorie: "Cloudflare Workers"
timeToRead: "8 min di lettura"
themen:
  - cloudflare-workers
slug: "gestire-una-newsletter-in-proprio-con-cloudflare-workers-e-d1"
translationOf: "serverloser-newsletter-cloudflare-workers-d1"
translationId: article-4e7139acdb90923b
translationReview: automatic
translationSourceHash: ad5b78d6330d06a17259e464c0fb8bb9713b3fdf5cd6c77ac1d300d9fea2a48e
translatedAt: 2026-09-04T08:38:44.886Z
url: https://rafaelpfister.ch/it/blog/gestire-una-newsletter-in-proprio-con-cloudflare-workers-e-d1
translationModel: gpt-5.6-terra
---

# Gestire una newsletter propria con Cloudflare Workers e D1

Con un servizio di newsletter ospitato, la lista dei destinatari resta presso il fornitore e i costi spesso aumentano con il numero di iscritti. Un server proprio offre maggiore controllo, ma comporta lavoro continuo: aggiornamenti, monitoraggio, backup e gestione di un sistema che magari invia solo una volta alla settimana.

Per questo caso d'uso snello bastano endpoint HTTP, un piccolo database e un processo di invio pianificato. Cloudflare Workers e D1 forniscono esattamente questi componenti. Il mio template open source li configura nel proprio account tramite un **pulsante Deploy to Cloudflare**. Non sono necessari una riga di comando locale né un server da mantenere continuamente. Il codice sorgente con licenza MIT è disponibile su [GitHub](https://github.com/pfstr/newsletter-template).

[![Deploy to Cloudflare](../images/serverloser-newsletter-cloudflare-workers-d1/deploy-to-cloudflare.svg)](https://deploy.workers.cloudflare.com/?url=https://github.com/pfstr/newsletter-template)

![Il modulo di iscrizione ospitato del template](../images/serverloser-newsletter-cloudflare-workers-d1/newsletter-template-signup.png)

## Cosa può fare il template

- **Iscrizione**: una pagina di iscrizione ospitata, un modulo incorporabile per il proprio sito web e un endpoint JSON
- **Cancellazione con un clic**: conforme a RFC 8058, con token individuale per ogni iscritto
- **Informazioni obbligatorie integrate**: ogni e-mail riceve automaticamente un footer con link di cancellazione e indirizzo postale; vengono memorizzati i momenti del consenso e della cancellazione
- **Invio**: su una pagina protetta si possono inserire oggetto e HTML, inviare un'e-mail di prova e accodare la campagna; un processo in background invia in batch e ritenta i tentativi non riusciti
- **Dati propri**: gli iscritti sono conservati in un database D1 nel proprio account e possono essere esportati in qualsiasi momento
- **Opzionali, disattivati per impostazione predefinita**: double opt-in, protezione dai bot tramite Turnstile e invio automatico di nuovi articoli del blog dal feed RSS

## Architettura: un Worker, un database

L'intero sistema consiste in un singolo Cloudflare Worker con due handler: `fetch` per HTTP (instradato con Hono) e `scheduled` per il trigger Cron, oltre a un database D1. Non esiste un secondo servizio, nessun broker di coda separato, nessun backend amministrativo proprio; persino la coda di invio è solo una tabella D1.

| Route | Funzione |
| --- | --- |
| `GET /` | Pagina di iscrizione ospitata |
| `GET /embed` | Modulo trasparente da incorporare tramite iframe |
| `POST /api/subscribe` | Iscrizione (CORS aperto per il proprio sito web) |
| `GET /confirm` | Link di conferma per il double opt-in |
| `GET/POST /unsubscribe` | Cancellazione: pagina di conferma tramite GET, esecuzione tramite POST (one-click secondo RFC 8058) |
| `GET /admin` | Pagina di invio (modulo) |
| `POST /api/send` | Accoda la campagna, protetto da token amministrativo |

Il modello dati comprende quattro tabelle: `subscribers` (e-mail come chiave primaria, nome, stato, token di cancellazione e conferma, una colonna JSON per campi aggiuntivi definiti dall'utente nonché timestamp per conferma e cancellazione), `campaigns` con oggetto, contenuto e contatori per ogni invio, `outbox` come coda di invio (una riga per destinatario) e `sent_posts` per la deduplicazione dell'invio RSS.

## Deploy senza riga di comando

Più interessante del codice è il percorso verso un sistema funzionante. Il pulsante Deploy to Cloudflare legge la configurazione Wrangler del repository e completa l'intera configurazione: clona il repository nel proprio account GitHub, effettua il provisioning del database D1, esegue le migrazioni dello schema e configura la CI, in modo che ogni push venga distribuito automaticamente. Da luglio 2025, il flusso di deploy richiede inoltre direttamente nel modulo variabili d'ambiente e segreti: nel caso di questo template la password amministrativa (`ADMIN_TOKEN`), il nome e l'indirizzo del mittente, l'interruttore del double opt-in e la dimensione del batch di invio (`SEND_BATCH`).

Il risultato dopo un clic e un modulo: la pagina di iscrizione è disponibile su `https://<worker-name>.workers.dev` e raccoglie iscritti. Non viene mai aperto un terminale.

## Raccogliere iscritti

Per l'integrazione nel proprio sito web sono disponibili tre modalità, in ordine crescente di profondità d'integrazione. La più semplice: condividere il link alla pagina di iscrizione ospitata. La più pratica per i site builder (WordPress, Webflow, Squarespace, Framer): una singola riga iframe in qualsiasi blocco HTML incorporato.

```html
<iframe
  src="https://<worker-name>.workers.dev/embed"
  style="width:100%;max-width:420px;height:90px;border:0"
></iframe>
```

Chi desidera il modulo con il proprio design può inviare direttamente all'endpoint:

```html
<form
  onsubmit="event.preventDefault();
  fetch('https://<worker-name>.workers.dev/api/subscribe', {
    method:'POST', headers:{'Content-Type':'application/json'},
    body: JSON.stringify({ email: this.email.value })
  }).then(()=>this.reset());"
>
  <input name="email" type="email" placeholder="you@example.com" required />
  <button>Abonnieren</button>
</form>
```

Il modulo raccoglie per impostazione predefinita l'e-mail e, facoltativamente, il nome. Altri campi (azienda, paese, …) vengono definiti in un unico file (`src/fields.ts`); appaiono automaticamente su entrambi i moduli e vengono salvati come JSON nel database.

## Invio: provider proprio anziché vendor integrato

Per l'invio delle e-mail, il template compie una scelta consapevole: è **agnostico rispetto al provider**. Il file `src/email.ts` contiene un singolo adattatore `sendEmail()` con un esempio commentato per un'API HTTP generica. La scelta del servizio di invio da collegare spetta a voi. Nessun fornitore è codificato rigidamente, non è richiesta alcuna registrazione presso un servizio specifico. La raccolta degli iscritti funziona già completamente senza configurazione di invio; l'invio viene abilitato non appena l'adattatore è implementato e il segreto del provider è impostato. Se il provider offre anche un endpoint batch (una chiamata API, molte e-mail), nello stesso file è possibile aggiungere un adattatore opzionale `sendEmailBatch()`; anche per questo è disponibile un esempio commentato.

L'invio viene gestito dalla pagina `/admin`: inserire l'oggetto e l'HTML dell'e-mail, inviare un test al proprio indirizzo, quindi accodare la campagna per tutti gli iscritti. Nelle e-mail sono disponibili i merge tag `{{unsubscribe_url}}`, `{{email}}` e `{{name}}`.

L'invio effettivo avviene in background, secondo il modello Transactional Outbox: `POST /api/send` scrive la campagna e una riga per destinatario nel database e risponde subito. Un processo Cron ogni minuto consegna quindi `SEND_BATCH` e-mail per esecuzione, 40 per impostazione predefinita: valore scelto affinché ogni esecuzione resti entro i limiti di subrequest del piano Workers Free. Le righe vengono rivendicate atomicamente, quindi esecuzioni sovrapposte non possono mai inviare due volte; le consegne fallite vengono ritentate fino a tre volte, le esecuzioni interrotte riprese dopo dieci minuti. E chi si cancella mentre la propria e-mail è ancora nella coda non la riceve più: l'opt-out annulla anche i messaggi già accodati.

## Cancellazione e prove fanno parte del nucleo

Chi invia una newsletter è soggetto alla normativa antispam e sulla protezione dei dati: al CAN-SPAM Act statunitense, al GDPR e all'ePrivacy nell'UE, all'UWG in Svizzera. Una parte essenziale di ciò per cui si paga un servizio di newsletter è proprio l'adempimento di questi obblighi. Il template si occupa della parte meccanica:

- **Footer obbligatorio**: ogni e-mail della campagna riceve automaticamente un footer con un link di cancellazione funzionante e l'indirizzo postale del mittente (`SENDER_ADDRESS`); il CAN-SPAM richiede un indirizzo fisico nelle e-mail commerciali. La pagina di invio avverte finché manca l'indirizzo.
- **Header List-Unsubscribe secondo RFC 8058** a ogni invio: il pulsante nativo di cancellazione in Gmail e Outlook, richiesto da Gmail e Yahoo ai mittenti di massa dal 2024. L'app compone completamente gli header; l'adattatore del provider deve solo inoltrarli.
- **Cancellazione sicura contro gli scanner**: il link di cancellazione porta a una pagina di conferma con un unico pulsante. Gli scanner di e-mail aziendali, che recuperano in anticipo ogni link di un'e-mail, non possono così cancellare accidentalmente nessuno; i client e-mail usano direttamente il POST one-click.
- **Minimizzazione dei dati e prova**: un opt-out ha effetto immediato, cancella nome e campi aggiuntivi e viene registrato con timestamp, così come l'iscrizione e la conferma del double opt-in. Il consenso può quindi essere dimostrato in seguito (obbligo di responsabilizzazione del GDPR).
- **Link alla privacy**: impostando `PRIVACY_URL` appare un link alla propria informativa sulla privacy sotto il modulo di iscrizione.

Restano a carico dell'operatore righe del mittente e dell'oggetto veritiere, l'invio solo a indirizzi effettivamente iscritti e l'autenticazione del dominio (SPF/DKIM/DMARC) presso il servizio di invio. Tutto questo non costituisce consulenza legale.

## Opzioni: double opt-in, Turnstile, automazione RSS

Tre funzioni sono integrate, ma disattivate per impostazione predefinita affinché il sistema resti utilizzabile senza configurazione:

- **Double opt-in** (`DOUBLE_OPT_IN = "true"`): i nuovi iscritti vengono salvati come `pending` e diventano attivi solo dopo aver fatto clic su un link di conferma. Per la Svizzera (DSG) e l'UE, questa procedura è la scelta più corretta.
- **Protezione dai bot** con Cloudflare Turnstile: impostare site key e secret key come variabili; il widget appare automaticamente su entrambi i moduli e il Worker verifica ogni iscrizione lato server. Senza un token valido, l'iscrizione viene rifiutata.
- **Invio automatico RSS**: un processo Cron controlla ogni 15 minuti il feed del proprio blog (RSS 2.0 o Atom) e accoda automaticamente i nuovi articoli nella coda di invio. Sono integrate due protezioni: alla primissima esecuzione il feed esistente viene contrassegnato solo come base di riferimento (l'archivio non viene quindi inviato come una valanga di e-mail) e ogni ID articolo viene registrato in `sent_posts`, in modo che nessun contributo venga inviato due volte.

## Limiti

Il template è volutamente minimale. L'invio tramite coda nel piano Free consegna per impostazione predefinita circa 40 e-mail al minuto; una campagna a 1'000 destinatari richiede quindi circa 25 minuti, cosa irrilevante per una newsletter. Nel piano Workers a pagamento (10'000 subrequest per chiamata invece di 50), `SEND_BATCH` può essere aumentato a centinaia; con un adattatore batch (una chiamata API, fino a circa 1'000 e-mail) anche il piano Free invia grandi liste in pochi minuti. La recapitalibilità dipende, come in ogni sistema, dal proprio dominio mittente: SPF, DKIM e DMARC devono essere verificati presso il servizio di invio scelto, altrimenti la newsletter finirà nello spam. E il single opt-in predefinito è l'avvio più semplice, ma non la variante di conformità più conservativa; per questo esiste l'interruttore.

Quanto ai costi: Workers e D1 dispongono di generose quote Free Tier (tra cui 100'000 richieste al giorno), che un modulo di iscrizione e invii settimanali a una lista piccola o media non esauriscono. Se viene raggiunto un limite, Cloudflare limita la velocità nel piano Free invece di emettere una fattura.

## Provarlo

Il codice sorgente, incluso il pulsante di deploy, è disponibile su [GitHub](https://github.com/pfstr/newsletter-template); lì si trova anche la documentazione completa delle variabili di configurazione.

[![GitHub: pfstr/newsletter-template](../images/serverloser-newsletter-cloudflare-workers-d1/github-newsletter-template.svg)](https://github.com/pfstr/newsletter-template)

## Fonti

1.  [pfstr/newsletter-template](https://github.com/pfstr/newsletter-template): codice sorgente del template (MIT) con pulsante di deploy e documentazione.

2.  [Deploy to Cloudflare buttons](https://developers.cloudflare.com/workers/platform/deploy-buttons/): provisioning automatico delle risorse, clonazione del repository e CI durante il deploy.

3.  [Deploy buttons: environment variables and secrets](https://developers.cloudflare.com/changelog/post/2025-07-01-workers-deploy-button-supports-environment-variables-and-secrets/): segreti e variabili vengono richiesti nel modulo di deploy da luglio 2025.

4.  [Cloudflare D1](https://developers.cloudflare.com/d1/): SQLite serverless, qui per iscritti, registro degli invii e deduplicazione RSS.

5.  [Cloudflare Turnstile](https://developers.cloudflare.com/turnstile/): protezione dai bot senza rompicapi CAPTCHA, attivabile opzionalmente nel template.

6.  [RFC 8058](https://datatracker.ietf.org/doc/html/rfc8058): Signaling One-Click Functionality for List Email Headers; base del pulsante nativo di cancellazione in Gmail e Outlook.

7.  [Workers limits](https://developers.cloudflare.com/workers/platform/limits/): limiti di subrequest per chiamata (50 nel piano Free, 10'000 nel piano a pagamento); da qui deriva la dimensione del batch dell'invio tramite coda.

8.  [FTC: CAN-SPAM Act Compliance Guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business): obblighi per le e-mail commerciali, tra cui indirizzo postale e opt-out funzionante.
