---
title: "Invio di e-mail tramite un relay: verificare TLS e autenticazione"
navTitle: "Relay: verificare TLS"
description: "Una guida sintetica per gli Application Manager le cui applicazioni inviano e-mail tramite un relay: quali tre impostazioni dell’applicazione contano (porta, modalità TLS, autenticazione), come si chiamano le opzioni negli ambienti comuni e come una sola e-mail di test dimostra, tramite l’header Received, che la connessione è effettivamente crittografata e autenticata."
date: "2026-08-28"
kategorie: "SMTP e flusso della posta"
timeToRead: "5 min di lettura"
themen:
  - smtp-mailflow
protokolle:
  - "smtp"
  - "tls"
  - "troubleshooting"
slug: "invio-di-e-mail-tramite-un-relay-verificare-tls-e-autenticazione"
translationId: "article-734e79c4a87105e3"
translationOf: mail-relay-tls-authentisierung-pruefen
translationSourceHash: a1f56d0ba84377020bb56c927ee5d8789198061a0f941b463e3fd0e929262082
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T10:04:25.965Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/invio-di-e-mail-tramite-un-relay-verificare-tls-e-autenticazione
---

# Invio di e-mail tramite un relay: verificare TLS e autenticazione

Molte applicazioni non inviano direttamente le e-mail su Internet, ma le consegnano a un relay interno: l’ERP invia le conferme d’ordine, il monitoraggio gli allarmi, il sistema di ticketing le notifiche. Il relay è gestito dal team e-mail; dal lato dell’applicazione è responsabile l’Application Manager. In caso di audit o analisi del fabbisogno di protezione, la domanda arriva quindi a lui: l’applicazione si connette al relay in modo crittografato e si autentica correttamente?

La risposta si trova in due punti, per i quali non sono necessari né uno strumento di posta né l’accesso al relay: nella configurazione SMTP dell’applicazione e nell’header di una sola e-mail di test. Ciò che il relay stesso offre e il modo in cui crittografa ulteriormente le e-mail fino al destinatario sono responsabilità del team e-mail; dal lato dell’applicazione è sufficiente documentare il proprio tratto.

## Dove si trovano le impostazioni

A seconda dell’applicazione, la configurazione SMTP si trova in uno di tre luoghi: nell’interfaccia di amministrazione (solitamente in «E-mail», «Notifiche», «SMTP» o «Server in uscita»), in un file di configurazione o nelle variabili d’ambiente del deployment (tipicamente `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER` e varianti). Si cercano sempre le stesse informazioni: nome del server, porta, un’opzione di crittografia e le credenziali.

## Le tre impostazioni che contano

**Primo: porta e modalità TLS.** Entrambe devono corrispondere, poiché dietro ai valori selezionabili si celano due procedure diverse: con STARTTLS la connessione inizia in chiaro e passa poi a TLS, mentre con TLS implicito (solitamente denominato «SSL/TLS» o «SSL» nelle schermate) è crittografata fin dall’inizio.

| Porta | Impostazione TLS nell’applicazione | Valutazione |
|---|---|---|
| 587 | STARTTLS | Stato desiderato per l’invio da parte delle applicazioni |
| 465 | SSL/TLS (implicito) | anch’esso corretto |
| 25 | nessuna o STARTTLS | comune per relay con autorizzazione IP; attivare comunque l’impostazione TLS se il relay offre STARTTLS |
| qualsiasi | «Nessuna» / «None» | Riscontro: l’invio avviene in chiaro |
| qualsiasi | «TLS, se disponibile» / opportunistico | Riscontro: in caso di problema torna silenziosamente al testo in chiaro; passare a TLS obbligatorio |

Una combinazione errata (ad esempio «SSL/TLS» sulla porta 587) causa interruzioni della connessione, non comunicazioni in chiaro inosservate. Le impostazioni rischiose sono le ultime due righe della tabella, perché in questi casi l’applicazione invia senza crittografia e senza messaggio di errore.

**Secondo: la verifica del certificato.** Molte applicazioni offrono un’opzione come «Non verificare il certificato», «Allow insecure» o `verify=false`, che nei progetti di introduzione viene spesso impostata perché il relay utilizza un certificato interno. La connessione resta crittografata, ma l’applicazione accetta qualsiasi controparte. Se l’opzione è impostata, va riportata come riscontro nel rapporto; la soluzione corretta è fidarsi della CA interna invece di disattivare la verifica.

**Terzo: l’autenticazione.** I relay conoscono due modelli: SMTP AUTH con nome utente e password oppure autorizzazione IP senza account. La variante applicabile è indicata nell’abilitazione del relay fornita dal team e-mail. Con SMTP AUTH, tre punti devono comparire nella checklist: l’autenticazione avviene tramite un account di servizio dedicato dell’applicazione (non tramite un account personale che verrà disattivato alla prossima uscita), la password è memorizzata come secret anziché in chiaro in un file di configurazione e l’opzione TLS è attiva, poiché le procedure comuni PLAIN e LOGIN altrimenti trasmettono le credenziali in chiaro.

## Come si chiamano le impostazioni negli ambienti comuni

| Ambiente | Crittografia | Autenticazione |
|---|---|---|
| Interfacce di amministrazione (ERP, monitoraggio, appliance) | Menu a discesa «Crittografia»: None / STARTTLS / SSL-TLS | Campi nome utente/password; vuoti = nessuna autenticazione |
| Java (Jakarta Mail, Spring) | `mail.smtp.starttls.enable=true` più `mail.smtp.starttls.required=true`; per la porta 465 `mail.smtp.ssl.enable=true` | `mail.smtp.auth=true` |
| .NET | `SmtpClient.EnableSsl=true` (attiva STARTTLS); MailKit: `SecureSocketOptions.StartTls` | `Credentials` o `Authenticate()` |
| PHP (PHPMailer) | `SMTPSecure='tls'` per 587, `'ssl'` per 465 | `SMTPAuth=true` |
| Python (smtplib) | `starttls()` dopo la connessione o `SMTP_SSL` per 465 | `login()` |
| Node.js (Nodemailer) | Porta 465: `secure:true`; porta 587: `secure:false` più `requireTLS:true` | `auth: {user, pass}` |

Due punti di questa tabella sono, per esperienza, i riscontri più frequenti: in Java, `starttls.enable` da solo attiva solo TLS opportunistico; solo `starttls.required` impedisce il ritorno al testo in chiaro. In Nodemailer, `secure:false` non significa «non crittografato», bensì «nessun TLS implicito»; senza `requireTLS:true`, tuttavia, anche STARTTLS resta opportunistico.

## Controprova: una e-mail di test e il relativo header Received

La configurazione indica lo stato desiderato, ma non dimostra ciò che avviene sul collegamento. La prova si trova nell’header Received, che il relay inserisce alla ricezione di ogni e-mail. È sufficiente inviare dall’applicazione una e-mail di test alla propria casella di posta; visualizzare quindi l’header del messaggio (Outlook: File, Proprietà, Intestazioni Internet; Gmail: Mostra originale) e leggere la riga Received più in basso, poiché gli header crescono dal basso verso l’alto:

```text
Received: from app01.example.com (app01.example.com [10.1.2.3])
        by relay.example.com (Postfix) with ESMTPSA id 4XyZk12Fzq
        (version=TLSv1.3 cipher=TLS_AES_256_GCM_SHA384);
        Thu, 28 Aug 2026 09:15:04 +0200
```

La parola chiave dopo `with` è la sintesi del risultato della verifica. Le sigle sono standardizzate (registro IANA «Mail Transmission Types»):

| Sigla | Significato | Valutazione |
|---|---|---|
| `SMTP` / `ESMTP` | non crittografato, senza autenticazione | Intervento necessario se TLS è richiesto |
| `ESMTPS` | TLS, senza autenticazione | corretto con autorizzazione IP |
| `ESMTPA` | autenticato, ma senza TLS | critico: le credenziali sono state trasmesse in chiaro |
| `ESMTPSA` | TLS e autenticato | Stato desiderato con SMTP AUTH |

Postfix e Exchange aggiungono tra parentesi la versione TLS e il cipher, consentendo di riconoscere anche versioni obsolete del protocollo. Per analizzare header più lunghi con più stazioni, [Mail-Header-Analyzer](https://rafaelpfister.ch/tools/header-analyzer) su questo sito evita il lavoro manuale; funziona interamente in locale nel browser e l’header non lascia il computer.

Se l’header resta poco chiaro o un load balancer a monte modifica l’indicazione della connessione, è il momento di rivolgersi al team e-mail: il log del relay registra per ogni consegna se TLS è stato negoziato e con quale account l’applicazione si è autenticata.

## Checklist breve per il rapporto di verifica

1. Configurazione SMTP dell’applicazione individuata (interfaccia, file di configurazione o variabili d’ambiente) e documentata.
2. Porta e modalità TLS corrispondono (587/STARTTLS o 465/SSL-TLS); nessuna impostazione «Nessuna» o «TLS, se disponibile».
3. Verifica del certificato attiva; l’eventuale impostazione «Non verificare il certificato» è registrata come riscontro.
4. Modello di autenticazione chiarito: SMTP AUTH con account di servizio e memorizzazione come secret, oppure autorizzazione IP secondo l’abilitazione del relay.
5. L’header Received dell’e-mail di test mostra `ESMTPSA` (con account) oppure `ESMTPS` (con autorizzazione IP); `ESMTPA` e `ESMTP` sono riscontri.
6. Se è richiesta la crittografia fino al destinatario: indirizzare il requisito al team e-mail, poiché il tratto a partire dal relay è esterno all’applicazione.

## Cronologia delle modifiche

- **25 ottobre 2026:** aggiornati i link alle fonti con i nuovi indirizzi IANA; la fonte Jakarta Mail ora rimanda alla documentazione di Eclipse Angus Mail.

## Fonti

1.  [RFC 3207: SMTP Service Extension for Secure SMTP over Transport Layer Security](https://www.rfc-editor.org/rfc/rfc3207): definisce STARTTLS e il passaggio della connessione in chiaro a TLS.

2.  [RFC 4954: SMTP Service Extension for Authentication](https://www.rfc-editor.org/rfc/rfc4954): definisce SMTP AUTH e i meccanismi quali PLAIN e LOGIN.

3.  [RFC 8314: Cleartext Considered Obsolete](https://www.rfc-editor.org/rfc/rfc8314): raccomanda TLS implicito sulla porta 465 per l’invio da parte dei client.

4.  [IANA: Mail Transmission Types](https://www.iana.org/assignments/smtp/smtp.xhtml#smtp-mail-transmission-types): registro delle sigle `with` nell’header Received (ESMTPS, ESMTPA, ESMTPSA).

5.  [Eclipse Angus Mail: Package org.eclipse.angus.mail.smtp](https://eclipse-ee4j.github.io/angus-mail/docs/api/org.eclipse.angus.mail/org/eclipse/angus/mail/smtp/package-summary.html): provider SMTP dell’implementazione Jakarta Mail; documenta le proprietà mail.smtp.starttls.enable, starttls.required e ssl.enable.
