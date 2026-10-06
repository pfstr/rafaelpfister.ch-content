---
title: "Accesso a HIN Mail: Webmail, token per Outlook, password dimenticata e HIN Mail Global"
navTitle: "Accesso a HIN Mail"
description: "Dove e come accedere a HIN Mail: Webmail su webmail.hin.ch, token di posta per Outlook e smartphone, accesso senza HIN Client tramite codice SMS o app Authenticator, reimpostazione della password e apertura di HIN Mail Global come destinatario senza account HIN."
date: "2026-10-05"
kategorie: "Gateway HIN"
timeToRead: "7 min di lettura"
themen:
  - hin-gateway
produkte:
  - "hin"
  - "outlook"
protokolle:
  - "troubleshooting"
  - "verschluesselung"
related:
  - hin-plattformerneuerung-2026
  - hin-mailgateway-backup-disaster-recovery
slug: "accesso-a-hin-mail-webmail-token-per-outlook-password-dimenticata-e-hin-mail-global"
translationId: "article-6f466c817c7c7c66"
translationOf: hin-mail-login
url: https://rafaelpfister.ch/it/blog/accesso-a-hin-mail-webmail-token-per-outlook-password-dimenticata-e-hin-mail-global
translationSourceHash: 8f863faa139fd22182bf59e849447e241e339e4fcc4e6a073e6c1a89344f8a06
translationModel: gpt-5.6-terra
translatedAt: 2026-10-06T10:39:38.624Z
translationReview: automatic
---

# Accesso a HIN Mail: Webmail, token per Outlook, password dimenticata e HIN Mail Global

Dietro la ricerca «HIN Mail Login» si celano di solito tre situazioni diverse: siete membri HIN e desiderate leggere le e-mail nel browser, volete configurare HIN Mail in Outlook o sullo smartphone, oppure avete ricevuto, in qualità di paziente o ente esterno, un'e-mail HIN crittografata e non disponete di alcun account HIN. L'accesso funziona in modo diverso in tutti e tre i casi.

| Situazione | Indirizzo | Nome utente | Password |
|---|---|---|---|
| Webmail nel browser | `webmail.hin.ch` | HIN ID | Password HIN, protetta da HIN Client, codice SMS o app Authenticator |
| Outlook, Thunderbird, Apple Mail, smartphone | `imap.mail.hin.ch`, `smtp.mail.hin.ch` | HIN ID | Token di posta da `apps.hin.ch` |
| Gestione, token, password di inizializzazione | `apps.hin.ch` | HIN ID | Password HIN più secondo fattore |
| Destinatario senza account HIN (HIN Mail Global) | Allegato `secure-email.html` o link nell'e-mail | Numero di cellulare | Codice SMS |

## HIN Webmail: accesso nel browser

Il Webmail è disponibile all'indirizzo `https://webmail.hin.ch`. Accedete con il vostro HIN ID (il nome di accesso HIN, ad es. `hmuster`) e la password HIN. Il secondo fattore è fornito da uno dei seguenti metodi:

- **HIN Client:** se il client è in esecuzione sulla postazione di lavoro ed è connesso, il Webmail si apre senza ulteriori richieste. Con HIN Client 4.0, l'accesso avviene direttamente nel browser: inserite l'indirizzo protetto da HIN e venite reindirizzati alla pagina di accesso.
- **Codice SMS (mTAN):** dopo la password ricevete un codice sul numero di cellulare registrato. Questo metodo è adatto ai dispositivi senza HIN Client, ad esempio un laptop privato.
- **App HIN Authenticator:** l'app (App Store e Google Play, «HIN Authenticator») conferma l'accesso sullo smartphone.

Il codice SMS e l'app Authenticator devono essere attivati preventivamente. Se entrambi mancano e HIN Client non è disponibile, non è possibile accedere al Webmail. Configurate quindi almeno un'alternativa finché il client funziona ancora.

Nel Webmail potete scrivere e gestire e-mail, curare contatti e rubriche, cercare nella directory dei partecipanti HIN e creare più firme. Sotto il nome utente, una barra mostra lo spazio libero della casella di posta; se è rossa, dovete eliminare e-mail o ampliare lo spazio di archiviazione.

## HIN Mail in Outlook e sullo smartphone: accesso con token di posta

I programmi di posta non accedono con la password HIN, bensì con un **token di posta**. Questa è la causa più frequente delle ripetute richieste di password in Outlook: la password HIN viene rifiutata durante l'accesso IMAP.

Ecco come generare il token:

1. Aprite `https://apps.hin.ch` e accedete con l'HIN ID (HIN Client, codice SMS o app Authenticator).
2. Selezionate la sezione «HIN Mail».
3. Fate clic su «Aggiungi dispositivo» e selezionate «Mail Clients» come tipo di dispositivo.
4. Copiate il token visualizzato e inseritelo come password nel programma di posta.

Le impostazioni del server:

| Impostazione | Posta in arrivo (IMAP) | Posta in uscita (SMTP) |
|---|---|---|
| Server | `imap.mail.hin.ch` | `smtp.mail.hin.ch` |
| Porta | `993` | `587` |
| Crittografia | SSL/TLS | STARTTLS |
| Nome utente | HIN ID | HIN ID |
| Password | Token di posta | Token di posta |
| Autenticazione | normale | attivare «Il server della posta in uscita richiede l'autenticazione» |

Come nome utente dovete usare l'HIN ID, non l'indirizzo e-mail. Outlook propone l'indirizzo e-mail durante la configurazione automatica; occorre quindi effettuare la configurazione manualmente tramite «Opzioni avanzate» o «Configurazione manuale».

Il token è associato alla vostra identità HIN e può essere revocato e generato nuovamente in qualsiasi momento in `apps.hin.ch`. È consigliabile utilizzare un token distinto per ogni dispositivo: se uno smartphone viene smarrito, bloccate solo quel dispositivo e Outlook sul PC dello studio continua a funzionare. Una volta configurato il token, Outlook non necessita più di HIN Client per scaricare la posta.

## Primo accesso senza HIN Client: attivare l'identità

Le nuove identità HIN possono essere attivate senza HIN Client. Sono necessari il nome utente e la password di inizializzazione riportati nella documentazione HIN.

1. Aprite il link di attivazione nel Servicecenter (`servicecenter.hin.ch`).
2. Inserite il nome utente e la password di inizializzazione.
3. Impostate una password personale: almeno 10 caratteri con cifre, lettere maiuscole e minuscole e caratteri speciali.
4. Configurate immediatamente un secondo fattore (codice SMS o app Authenticator).
5. Successivamente, accedete a `apps.hin.ch`.

Per l'attivazione sono disponibili 10 minuti. Successivamente, la password di inizializzazione viene reimpostata per motivi di sicurezza e dovete richiederne una nuova.

## Password HIN dimenticata

La reimpostazione dipende dalla presenza di un metodo di accesso alternativo configurato.

**Con codice SMS o app Authenticator:**

1. Accedete a `apps.hin.ch` con il metodo alternativo.
2. Selezionate la scheda «HIN Client» e generate una nuova password di inizializzazione.
3. In HIN Client selezionate «Registrazione», inserite il nome di accesso e la nuova password di inizializzazione.
4. Impostate una nuova password e confermate «Registra nuova identità HIN».

**Senza metodo alternativo e senza dispositivo connesso**, resta solo il supporto telefonico HIN (0848 830 740, dal lunedì al venerdì dalle 08:00 alle 18:00). Annotate la nuova password di inizializzazione prima di chiudere la finestra.

## HIN Mail Global: aprire un'e-mail crittografata senza account HIN

Studi medici, ospedali e laboratori inviano messaggi a pazienti o enti senza affiliazione HIN tramite HIN Mail Global. L'e-mail arriva come una normale e-mail, ma il contenuto è crittografato nell'allegato `secure-email.html` o dietro il link «Leggi messaggio crittografato». Non esiste un accesso HIN in senso stretto; l'autenticazione avviene tramite il vostro numero di cellulare.

**La prima volta:**

1. Scaricate e aprite l'allegato `secure-email.html` (oppure fate clic sul link nell'e-mail). Con servizi di Webmail quali Gmail, Bluewin o GMX, salvatelo prima e poi apritelo.
2. Selezionate la lingua e inserite il vostro numero di cellulare.
3. Inserite il codice ricevuto via SMS. Il messaggio viene visualizzato.

**Per i messaggi successivi** è sufficiente aprire l'allegato; il codice SMS viene inviato automaticamente al numero registrato.

Da notare:

- Il messaggio è disponibile tramite il link per **30 giorni**, dopodiché l'accesso scade. Salvate o stampate i contenuti importanti entro questo termine.
- I messaggi inoltrati possono essere aperti solo dal destinatario originale.
- Potete rispondere in modo crittografato.
- Potete contrassegnare un dispositivo come attendibile; la richiesta SMS viene quindi omessa per un anno.
- Se non ricevete alcun codice entro 15 minuti, potete richiederne uno nuovo.
- Sono supportate le due versioni più recenti di Chrome, Safari, Edge e Firefox.

In caso di problemi con HIN Mail Global, il supporto HIN Mail è disponibile al numero +41 58 670 48 69. Alle domande sul contenuto del messaggio può rispondere solo il mittente.

## Errori frequenti nell'accesso a HIN Mail

| Sintomo | Causa | Soluzione |
|---|---|---|
| Outlook richiede continuamente la password | Inserita la password HIN anziché il token di posta | Generate il token in `apps.hin.ch` e inseritelo come password |
| L'accesso a Outlook fallisce nonostante il token | Indirizzo e-mail come nome utente | Utilizzate l'HIN ID come nome utente |
| L'invio non riesce, la ricezione funziona | Autenticazione SMTP non attivata, porta o crittografia errata | Usate la porta `587` con STARTTLS e attivate l'autenticazione |
| Il Webmail richiede un codice che non arriva mai | Numero di cellulare obsoleto o codice SMS non attivato | Usate l'app Authenticator oppure accedete tramite HIN Client e aggiornate il numero |
| La password di inizializzazione viene rifiutata | Superato il termine di 10 minuti | Richiedete una nuova password di inizializzazione |
| Il token funzionava, poi improvvisamente non più | Token revocato in `apps.hin.ch` o dispositivo rimosso | Generate un nuovo token per questo dispositivo |
| HIN Mail Global: il link non si apre più | Termine di 30 giorni scaduto | Chiedete al mittente di inviarlo nuovamente |

## HIN Client 4.0: cosa cambia nell'accesso

HIN distribuisce automaticamente HIN Client 4.0 entro il 26 ottobre 2026; chi non riceve un aggiornamento automatico può scaricarlo da `download.hin.ch`. La password esistente rimane valida e viene confermata una sola volta al primo accesso con la nuova versione. L'accesso ai servizi protetti da HIN, come il Webmail, avviene poi direttamente nel browser.

La versione 4.0 richiede Windows 11 con TPM oppure macOS 14 con Security Chip. I dispositivi che non soddisfano questi requisiti non possono più utilizzare il client. L'accesso al Webmail rimane possibile tramite codice SMS o app Authenticator, mentre il recupero della posta in Outlook avviene tramite il token di posta. I dettagli sulle scadenze del rinnovamento della piattaforma sono disponibili nell'articolo [Rinnovamento della piattaforma HIN 2026](/blog/hin-plattformerneuerung-2026).

## Studi medici e ospedali con server di posta proprio

Le organizzazioni con un proprio HIN Mailgateway non accedono a HIN Mail. Le caselle di posta si trovano nel proprio Exchange o in Exchange Online, e il gateway (un'appliance SEPPmail) crittografa le e-mail nel percorso da e verso la community HIN. I problemi di accesso riguardano il sistema di posta interno, non la piattaforma HIN. Questo gateway viene sostituito dal nuovo HIN Gateway («Stargate»).

<aside class="offer-box">
  <span class="offer-box__tag">Verifica gratuita</span>
  <p><strong>Gestite un HIN Mailgateway proprio?</strong> Verifico il vostro ambiente e vi indico cosa occorre fare prima della migrazione a «Stargate».</p>
  <a class="offer-box__cta" href="/stargate">Registratevi ora</a>
</aside>

## Fonti

1.  [Supporto HIN: HIN Mail e Mobile](https://support.hin.ch/de/thema/hin-mail-mobile/): panoramica con indirizzo Webmail e istruzioni per Outlook, Apple Mail, Thunderbird, iOS e Android.

2.  [Supporto HIN: come funziona il Webmail?](https://support.hin.ch/de/service/hin-mail-und-mobile/wie-funktioniert-das-webmail.cfm): funzioni del Webmail, indicatore dello spazio di archiviazione.

3.  [Supporto HIN: guida al servizio Mail Token per Mail Clients](https://support.hin.ch/de/hin-mail-mobile/mail-token-service-mail-clients/): generazione del token in apps.hin.ch, server IMAP e SMTP, porte e crittografia.

4.  [Supporto HIN: cos'è un token?](https://support.hin.ch/de/service/hin-mail-und-mobile/was-ist-ein-token.cfm): token di posta e token di accesso, associazione all'identità HIN.

5.  [Supporto HIN: attivare l'identità HIN (senza HIN Client)](https://support.hin.ch/de/hin-2fa/hin-identitaet-aktivieren/): attivazione nel Servicecenter, termine di 10 minuti, regole per le password, secondo fattore.

6.  [Supporto HIN: password dimenticata](https://support.hin.ch/de/thema/hin-client/passwort-vergessen.cfm): reimpostazione tramite password di inizializzazione, supporto telefonico senza metodo alternativo.

7.  [Supporto HIN: HIN Mail ai non membri](https://support.hin.ch/en/service/hin-mail-to-non-members.cfm): disponibilità per 30 giorni, registrazione via SMS, dispositivi attendibili, browser supportati, numero di supporto.

8.  [HIN: HIN Mail Global](https://www.hin.ch/services/hin-mail/hin-mail-global/): invio di e-mail crittografate a destinatari senza affiliazione HIN.

9.  [Blog HIN: il nuovo HIN Client è arrivato](https://www.hin.ch/de/blog/2026/neuer-hin-client.cfm): accesso tramite browser, requisiti di sistema con TPM o Security Chip, distribuzione entro il 26 ottobre 2026.

10.  [HIN Authenticator nell'App Store](https://apps.apple.com/ch/app/hin-authenticator/id1535944002): app per il secondo fattore.
