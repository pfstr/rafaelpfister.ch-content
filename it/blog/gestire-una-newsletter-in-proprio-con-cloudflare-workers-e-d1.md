---
title: "Gestire una newsletter in proprio con Cloudflare Workers e D1"
navTitle: "Newsletter su Workers"
description: "Il template open source mette a disposizione iscrizione, disiscrizione, coda e database nel proprio account Cloudflare. Un pulsante di deploy configura Worker, D1 e CI senza un server locale."
date: "2026-07-22"
kategorie: "Cloudflare Workers"
timeToRead: "8 min di lettura"
themen:
  - "cloudflare-workers"
slug: "gestire-una-newsletter-in-proprio-con-cloudflare-workers-e-d1"
translationOf: "serverloser-newsletter-cloudflare-workers-d1"
url: "https://rafaelpfister.ch/it/blog/gestire-una-newsletter-in-proprio-con-cloudflare-workers-e-d1"
---

# Gestire una newsletter in proprio con Cloudflare Workers e D1

Con un servizio di newsletter ospitato, la lista dei destinatari rimane presso il fornitore e i costi spesso aumentano con il numero di iscritti. Un server proprio offre maggiore controllo, ma comporta lavoro continuo: aggiornamenti, monitoraggio, backup e gestione di un sistema che magari invia solo una volta alla settimana.

Per questo caso d'uso snello bastano endpoint HTTP, un piccolo database e un processo di invio pianificato. Cloudflare Workers e D1 forniscono proprio questi componenti. Il mio template open source li configura nel proprio account tramite un **pulsante Deploy to Cloudflare**. Non è necessaria una riga di comando locale né un server da mantenere in modo permanente. Il codice sorgente con licenza MIT è disponibile su [GitHub](https://github.com/pfstr/newsletter-template).

[![Deploy to Cloudflare](../images/serverloser-newsletter-cloudflare-workers-d1/deploy-to-cloudflare.svg)](https://deploy.workers.cloudflare.com/?url=https://github.com/pfstr/newsletter-template)

![Il modulo di iscrizione ospitato del template](../images/serverloser-newsletter-cloudflare-workers-d1/newsletter-template-signup.png)

## Cosa può fare il template

- **Iscrizione**: una pagina di iscrizione ospitata, un modulo incorporabile per il proprio sito web e un endpoint JSON
- **Disiscrizione con un clic**: conforme a RFC 8058, con token individuale per ogni iscritto
- **Informazioni obbligatorie integrate**: ogni e-mail riceve automaticamente un footer con link di disiscrizione e indirizzo postale; vengono memorizzati i momenti del consenso e della disiscrizione
- **Invio**: su una pagina protetta è possibile inserire oggetto e HTML, inviare un'e-mail di prova e mettere in coda la campagna; un processo in background invia in batch e ripete i tentativi non riusciti
- **Dati propri**: gli iscritti risiedono in un database D1 nel proprio account e possono essere esportati in qualsiasi momento
- **Opzionale, disattivato per impostazione predefinita**: double opt-in, protezione bot tramite Turnstile e invio automatico di nuovi articoli del blog dal feed RSS

## Architettura: un Worker, un database

L'intero sistema è un singolo Cloudflare Worker con due handler: `fetch` per HTTP (instradato con Hono) e `scheduled` per il trigger Cron, oltre a un database D1. Non esiste un secondo servizio, nessun broker di code separato, nessun backend di amministrazione dedicato; persino la coda di invio è solo una tabella D1.

| Route | Funzione |
| --- | --- |
| `GET /` | Pagina di iscrizione ospitata |
| `GET /embed` | Modulo trasparente da incorporare tramite iframe |
| `POST /api/subscribe` | Iscrizione (CORS aperto per il proprio sito web) |
| `GET /confirm` | Link di conferma per il double opt-in |
| `GET/POST /unsubscribe` | Disiscrizione: pagina di conferma tramite GET, esecuzione tramite POST (one-click secondo RFC 8058) |
| `GET /admin` | Pagina di invio (modulo) |
| `POST /api/send` | Mettere in coda la campagna, protetto tramite token admin |

Il modello dati comprende quattro tabelle: `subscribers` (e-mail come chiave primaria, nome, stato, token di disiscrizione e conferma, una colonna JSON per campi aggiuntivi definiti dall'utente e timestamp per conferma e disiscrizione), `campaigns` con oggetto, contenuto e contatori per ogni invio, `outbox` come coda di invio (una riga per destinatario) e `sent_posts` per la deduplicazione dell'invio RSS.

## Deployment senza riga di comando

La parte più interessante non è il codice, ma il percorso verso un sistema funzionante. Il pulsante Deploy to Cloudflare legge la configurazione Wrangler del repository ed esegue l'intera configurazione: clona il repository nel proprio account GitHub, effettua il provisioning del database D1, esegue le migrazioni dello schema e configura la CI, così ogni push viene distribuito automaticamente. Da luglio 2025, il flusso di deploy richiede inoltre variabili d'ambiente e segreti direttamente nel modulo: nel caso di questo template, la password admin (`ADMIN_TOKEN`), il nome e l'indirizzo del mittente, l'interruttore double opt-in e la dimensione del batch di invio (`SEND_BATCH`).

Il risultato dopo un clic e un modulo: la pagina di iscrizione è online su `https://<worker-name>.workers.dev` e raccoglie iscritti. Non viene aperto alcun terminale.

## Raccogliere iscritti

Per l'integrazione nel proprio sito web ci sono tre strade, in ordine crescente di profondità di integrazione. La più semplice: condividere il link alla pagina di iscrizione ospitata. La più pratica per i site builder (WordPress, Webflow, Squarespace, Framer): una riga iframe in un qualsiasi blocco di incorporamento HTML.

```html
<iframe
  src="https://<worker-name>.workers.dev/embed"
  style="width:100%;max-width:420px;height:90px;border:0"
></iframe>
```

Chi desidera il modulo con il proprio design può pubblicare direttamente sull'endpoint:

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

Il modulo raccoglie per impostazione predefinita l'e-mail e, facoltativamente, il nome. Altri campi (azienda, paese, …) si definiscono in un unico file (`src/fields.ts`); appaiono automaticamente su entrambi i moduli e vengono salvati come JSON nel database.

## Invio: provider proprio invece di un vendor integrato

Per l'invio delle e-mail, il template compie una scelta consapevole: è **agnostico rispetto al provider**. Il file `src/email.ts` contiene un singolo adattatore `sendEmail()` con un esempio commentato per una API HTTP generica. La scelta del servizio di invio da collegare resta a voi. Nessun fornitore è cablato in modo fisso, non è richiesta alcuna registrazione a un servizio specifico. La raccolta degli iscritti funziona già completamente senza configurazione di invio; l'invio viene abilitato non appena l'adattatore è implementato e il segreto del provider è impostato. Se il provider offre anche un endpoint batch (una chiamata API, molte e-mail), nello stesso file si può aggiungere un adattatore facoltativo `sendEmailBatch()`; anche per questo è disponibile un esempio commentato.

L'invio si gestisce dalla pagina `/admin`: inserire oggetto e HTML dell'e-mail, inviare un test al proprio indirizzo, poi mettere in coda la campagna per tutti gli iscritti. Nelle e-mail sono disponibili i merge tag `{{unsubscribe_url}}`, `{{email}}` e `{{name}}`.

L'invio vero e proprio avviene in background, secondo il pattern Transactional Outbox: `POST /api/send` scrive la campagna e una riga per destinatario nel database e risponde immediatamente. Un processo Cron ogni minuto consegna poi `SEND_BATCH` e-mail per esecuzione, 40 per impostazione predefinita: una scelta che mantiene ogni esecuzione entro i limiti di subrequest del piano Workers Free. Le righe vengono rivendicate atomicamente, quindi esecuzioni sovrapposte non possono mai inviare due volte; le consegne non riuscite vengono ritentate fino a tre volte, mentre le esecuzioni interrotte vengono riprese dopo dieci minuti. E chi si disiscrive mentre la propria e-mail è ancora nella coda non la riceve più: l'opt-out annulla anche i messaggi già messi in coda.

## Disiscrizione e prove sono parte del nucleo

Chi invia una newsletter è soggetto alle norme antispam e sulla protezione dei dati: il CAN-SPAM Act statunitense, il GDPR e la direttiva ePrivacy nell'UE, la UWG in Svizzera. Una parte essenziale di ciò per cui si pagano i servizi di newsletter è proprio l'adempimento di questi obblighi. Il template ne gestisce la parte meccanica:

- **Footer obbligatorio**: ogni e-mail di campagna riceve automaticamente un footer con un link di disiscrizione funzionante e l'indirizzo postale del mittente (`SENDER_ADDRESS`); il CAN-SPAM richiede un indirizzo fisico nelle e-mail commerciali. La pagina di invio avvisa finché l'indirizzo manca.
- **Header List-Unsubscribe secondo RFC 8058** su ogni invio: il pulsante nativo di disiscrizione in Gmail e Outlook, richiesto da Gmail e Yahoo ai mittenti di massa dal 2024. L'app compone già gli header; l'adattatore del proprio provider deve solo inoltrarli.
- **Disiscrizione sicura dagli scanner**: il link di disiscrizione porta a una pagina di conferma con un unico pulsante. Gli scanner delle e-mail aziendali, che recuperano preventivamente ogni link di un'e-mail, non possono così disiscrivere accidentalmente nessuno; i client di posta usano direttamente il POST one-click.
- **Minimizzazione dei dati e prova**: un opt-out ha effetto immediato, cancella nome e campi aggiuntivi e viene registrato con timestamp, così come l'iscrizione e la conferma double opt-in. Il consenso può quindi essere dimostrato in seguito (principio di responsabilizzazione GDPR).
- **Link alla privacy**: con `PRIVACY_URL` impostato, sotto il modulo di iscrizione compare un link alla propria informativa sulla privacy.

Al gestore restano l'uso di righe del mittente e dell'oggetto veritiere, l'invio solo a indirizzi effettivamente iscritti e l'autenticazione del dominio (SPF/DKIM/DMARC) presso il servizio di invio. Nulla di tutto questo costituisce consulenza legale.

## Opzioni: double opt-in, Turnstile, automazione RSS

Tre funzioni sono integrate, ma disattivate per impostazione predefinita affinché il sistema resti utilizzabile senza configurazione:

- **Double opt-in** (`DOUBLE_OPT_IN = "true"`): i nuovi iscritti vengono salvati come `pending` e diventano attivi solo dopo aver fatto clic su un link di conferma. Per la Svizzera (LPD) e l'UE, questa procedura è la scelta più corretta.
- **Protezione bot** con Cloudflare Turnstile: basta impostare site key e secret key come variabili; il widget compare automaticamente su entrambi i moduli e il Worker verifica ogni iscrizione lato server. Senza un token valido, l'iscrizione viene rifiutata.
- **Invio automatico RSS**: un processo Cron controlla ogni 15 minuti il feed del proprio blog (RSS 2.0 o Atom) e mette automaticamente in coda i nuovi articoli. Sono integrate due protezioni: alla primissima esecuzione, il feed esistente viene solo contrassegnato come baseline (l'archivio non viene quindi inviato come un'ondata di e-mail), e ogni ID dell'articolo viene registrato in `sent_posts`, così nessun articolo viene inviato due volte.

## Limiti

Il template è volutamente minimale. L'invio dalla coda nel piano Free consegna per impostazione predefinita circa 40 e-mail al minuto; una campagna a 1'000 destinatari dura quindi circa 25 minuti, cosa irrilevante per una newsletter. Nel piano Workers a pagamento (10'000 subrequest per chiamata invece di 50), `SEND_BATCH` può essere aumentato alle centinaia; con un adattatore batch (una chiamata API, fino a circa 1'000 e-mail), anche il piano Free invia grandi liste in pochi minuti. La recapibilità dipende, come per ogni sistema, dal proprio dominio mittente: SPF, DKIM e DMARC devono essere verificati presso il servizio di invio scelto, altrimenti la newsletter finirà nello spam. E il valore predefinito single opt-in è il modo più semplice per iniziare, ma non la variante di conformità più prudente; per questo c'è l'interruttore.

Quanto ai costi: Workers e D1 dispongono di generose quote free tier (tra cui 100'000 richieste al giorno), che un modulo di iscrizione e invii settimanali a una lista piccola o media non esauriscono. Se viene raggiunto un limite, Cloudflare limita il servizio nel piano Free invece di emettere una fattura.

## Provare

Il codice sorgente, incluso il pulsante di deploy, è disponibile su [GitHub](https://github.com/pfstr/newsletter-template); lì si trova anche la documentazione completa delle variabili di configurazione.

[![GitHub: pfstr/newsletter-template](../images/serverloser-newsletter-cloudflare-workers-d1/github-newsletter-template.svg)](https://github.com/pfstr/newsletter-template)

## Fonti

1.  [pfstr/newsletter-template](https://github.com/pfstr/newsletter-template): codice sorgente del template (MIT) con pulsante di deploy e documentazione.

2.  [Deploy to Cloudflare buttons](https://developers.cloudflare.com/workers/platform/deploy-buttons/): provisioning automatico delle risorse, clonazione del repository e CI durante il deploy.

3.  [Deploy buttons: environment variables and secrets](https://developers.cloudflare.com/changelog/post/2025-07-01-workers-deploy-button-supports-environment-variables-and-secrets/): segreti e variabili vengono richiesti nel modulo di deploy da luglio 2025.

4.  [Cloudflare D1](https://developers.cloudflare.com/d1/): SQLite serverless, usato qui per iscritti, registro degli invii e deduplicazione RSS.

5.  [Cloudflare Turnstile](https://developers.cloudflare.com/turnstile/): protezione bot senza rompicapi CAPTCHA, attivabile facoltativamente nel template.

6.  [RFC 8058](https://datatracker.ietf.org/doc/html/rfc8058): Signaling One-Click Functionality for List Email Headers; base del pulsante nativo di disiscrizione in Gmail e Outlook.

7.  [Workers limits](https://developers.cloudflare.com/workers/platform/limits/): limiti di subrequest per chiamata (50 nel piano Free, 10'000 nel piano a pagamento); da qui deriva la dimensione del batch dell'invio dalla coda.

8.  [FTC: CAN-SPAM Act Compliance Guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business): obblighi per le e-mail commerciali, tra cui indirizzo postale e opt-out funzionante.
