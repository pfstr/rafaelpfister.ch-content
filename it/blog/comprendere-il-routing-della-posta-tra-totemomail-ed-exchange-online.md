---
title: "Comprendere il routing della posta tra totemomail ed Exchange Online"
navTitle: "Apache James ↔ M365"
description: "Come totemomail archivia ed elabora i messaggi, come l'Apache James sottostante passa da un processor all'altro e cosa conta per un ciclo di posta sicuro con Exchange Online."
date: "2026-06-17"
kategorie: "Totemomail"
timeToRead: "10 min di lettura"
themen:
  - totemomail
slug: "comprendere-il-routing-della-posta-tra-totemomail-ed-exchange-online"
translationOf: "totemomail-m365"
translationId: article-60a86616507315fa
translationReview: automatic
translationSourceHash: d92dbf8dac2f4fe0cd6e28f70e9e5575c76b56acb4afa5c5444f9e10895b4abf
translatedAt: 2026-10-14T08:57:51.783Z
url: https://rafaelpfister.ch/it/blog/comprendere-il-routing-della-posta-tra-totemomail-ed-exchange-online
translationModel: gpt-5.6-terra
---

# Comprendere il routing della posta tra totemomail ed Exchange Online

In un ciclo di posta tra Exchange Online e totemomail, ogni sistema svolge un compito chiaramente delimitato. Exchange Online fornisce le cassette postali. Totemomail, ovvero l'attuale Kiteworks Email Protection Gateway, si occupa di crittografia, firme, criteri e regole di routing speciali.

Affinché ne risulti un flusso di posta affidabile, non basta configurare due connettori SMTP. Per la risoluzione dei problemi deve anche essere chiaro cosa accade all'interno del gateway dopo l'accettazione di un messaggio: dove si trova? Quale regola viene eseguita successivamente? E perché un messaggio può restare in attesa in una coda, anche se il dialogo SMTP si è già concluso correttamente?

Questo articolo spiega quindi il modello di elaborazione di [Apache James](https://james.apache.org/), su cui si basa totemomail. La configurazione concreta del routing dipende dal rispettivo ambiente; i processor, matcher, mailet e repository qui descritti costituiscono tuttavia il fondamento tecnico di ogni installazione.

Un'importante regola di sicurezza vale indipendentemente dai dettagli: se totemomail è il gateway a monte, Exchange Online deve accettare la posta Internet solo da questo gateway. A tal fine è necessario un connettore partner restrittivo. Un record MX da solo non blocca il percorso di consegna diretto. L'articolo [Un record MX non è un firewall](/blog/ghost-sender-exchange-online-nebeneingang) mostra come nasce questo accesso secondario e come chiuderlo.

## Dall'ingresso SMTP all'elaborazione

La logica di elaborazione di Apache James è composta da quattro elementi:

- **Matcher** verificano le condizioni e determinano per quali destinatari si applica una regola.
- **Mailet** eseguono l'azione vera e propria, ad esempio modificare gli header, crittografare, consegnare o terminare l'ulteriore elaborazione.
- **Processor** raggruppano matcher e mailet in fasi di elaborazione ordinate.
- **Repository di posta** memorizzano i messaggi durante l'elaborazione o dopo un errore.

Questa separazione è fondamentale per l'analisi: il repository risponde alla domanda **dove** si trova un messaggio. Il processor determina **come** verrà ulteriormente elaborato.

![Utilizzo di James come relay SMTP](../images/4CixEi383SY5WdvwMSGZ67odMU.png)

Il server SMTP accetta la connessione e legge il messaggio fino alla fine della sezione `DATA`. Successivamente James crea un oggetto `MailImpl`. Esso contiene il contenuto MIME come `MimeMessage`, nonché le informazioni necessarie per l'elaborazione: mittente, destinatari, stato e altri attributi.

Con un repository basato su file, James memorizza queste informazioni separatamente:

- `FileStreamStore` contiene il messaggio RFC 822/MIME completo come flusso di byte.
- `FileObjectStore` contiene l'oggetto `MailImpl` serializzato con stato e metadati.

Un messaggio può quindi essere già stato accettato e memorizzato completamente, anche se la sua elaborazione funzionale è ancora in sospeso.

## Repository e code in `/var/mail`

I singoli repository appaiono nel file system come directory. In condizioni normali, un messaggio vi rimane solo per pochissimo tempo. Se una coda si accumula, ciò indica generalmente una regola errata, una destinazione non raggiungibile o un servizio backend non disponibile.

L'esempio seguente contiene, oltre alle code standard, anche directory opzionali per una connessione HIN. HIN fornisce lo spazio di comunicazione sicuro per il settore sanitario svizzero.

> Se avete bisogno di supporto per la connessione al gateway di posta HIN o per la migrazione alla nuova soluzione HIN-Stargate, troverete gli esperti giusti presso [adeptio](https://adeptio.ch/).  
>   
> **adeptio** è partner ufficiale di [Health Info Net AG](https://www.hin.ch/de/index.cfm) e, in quanto tale, dispone anche di interlocutori diretti presso il produttore.  
> [➜ Prenotate oggi stesso un appuntamento.](/kontakt)

```text
Root-Folder:
~/mailer/apps/james/var/mail

├── spool/
│   → Eingehende Mails (initiale Queue, noch nicht verarbeitet)
│
├── incoming/
│   → Mails, die als intern zuzustellen erkannt wurden (Standardfolder)
│
├── incomingHIN/
│   → Eingehende Mails für HIN-Netzwerk (Optional)
│
├── outgoing/
│   → Normale ausgehende Mails (Standardfolder)
│
├── outgoingHIN/
│   → Ausgehende Mails über HIN-Netzwerk (Optional)
│
├── outgoingNotifications/
│   → System- oder Zertifikatsbenachrichtigungen
│
├── error/
│   → Fehlgeschlagene Mails (z. B. Policy, Encryption, Routing)
│
├── DBUnavailable/
│   → Mails, die wegen Backend-/DB-Problemen nicht verarbeitet werden konnten
```

## Come si presenta un messaggio nel file system

Ogni messaggio memorizzato è composto da due file.

### `FileStreamStore`: contenuto del messaggio

Il file `*.FileStreamStore` contiene il messaggio RFC 822/MIME completo. Con `cat` sono leggibili header e body:

```text
From:
To:
Subject:
...
Body
```

Il formato di messaggio sottostante è descritto in [RFC 822](https://datatracker.ietf.org/doc/html/rfc822).

### `FileObjectStore`: stato e metadati

Il file `*.FileObjectStore` è un oggetto Java serializzato di tipo `org.apache.james.core.MailImpl`. I suoi campi includono:

```text
attributes: HashMap
errorMessage: String
lastUpdated: Date
message: MimeMessage
name: String
state: String
recipients: Collection
remoteAddr
remoteHost
sender
```

La [documentazione API di `MailImpl`](https://james.apache.org/server/3/apidocs/org/apache/james/core/MailImpl.html) descrive in dettaglio il modello a oggetti.

## Lo stato seleziona il processor successivo

La struttura delle directory mostra solo il repository. Lo stato di elaborazione effettivo si trova nel campo `state` di `FileObjectStore`. Il suo valore fa riferimento all'attributo `name` di un processor.

Dopo ogni mailet, SpoolManager verifica questo stato:

1. Se lo stato rimane invariato, segue la coppia matcher-mailet successiva nello stesso processor.
2. Se un mailet modifica lo stato, James termina il processor corrente e passa al processor con lo stesso nome.
3. Lo stato speciale `ghost` termina completamente l'elaborazione.

I processor obbligatori `root` e `error` hanno compiti fissi. I nuovi messaggi iniziano in `root`; gli errori interni e i mailet configurati di conseguenza reindirizzano a `error`. L'ordine degli elementi `<processor>` nel file XML non determina invece **non** l'ordine di esecuzione.

## Struttura dei processor in `totemomail_config.xml`

Prima di ogni modifica, è opportuno esportare e salvare la configurazione `totemomail_config.xml` attuale:

![Configuration / Open Current / Export to File](../images/kWKIN3vramf0IAuPjzioEGV4Znw.png)

I vari processor e i mailet in essi contenuti sono visibili in totemomail\_config.xml. Ecco un altro esempio pratico:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<spoolmanager>
    <multiparamformat>XML</multiparamformat>

    <processor name="addExtSender">
    <processor name="decrypt">
    <processor name="error">
    <processor name="externalDelivery">
    <processor name="externalDeliveryToHIN">
    <processor name="incoming">
    <processor name="internalDelivery">
    <processor name="internalDeliveryToHIN">
    <processor name="outgoing">
    <processor name="outgoingCheckRecipientCertificate">
    <processor name="outgoingProcessAutoGeneratedMessages">
    <processor name="outgoingProcessEncryptionTriggers">
    <processor name="outgoingProcessEncryptionTriggersRemoval">
    <processor name="outgoingProcessExceptionTriggers">
    <processor name="processIncoming">
    <processor name="processOutgoing">
    <processor name="processOutgoingCertificateExchange">
    <processor name="processOutgoingDomainEncryptionPGP">
    <processor name="processOutgoingDomainEncryptionSMIME">
    <processor name="processOutgoingNotifications">
    <processor name="root">

</spoolmanager>
```

Sebbene `root` si trovi alla fine di questo estratto, ogni nuovo messaggio inizia lì. Conta il nome, non la posizione nel documento.

Il processor `root` stesso contiene un elenco ordinato di coppie matcher-mailet:

```xml
   <processor name="root">
      <mailet class="SimpleLogger" match="All">
         <log-message>totemomail: New Mail</log-message>
         <showSenderEmailAddress>true</showSenderEmailAddress>
         <showRecipientsEmailAddress>true</showRecipientsEmailAddress>
         <showSubject>false</showSubject>
      </mailet>
      <mailet class="ToRepository" match="RelayLimit?Limit=20">
         <repositoryPath>file://var/mail/error</repositoryPath>
         <passThrough>false</passThrough>
         <notifySender>false</notifySender>
         <takeSenderInfoFrom>SMTP</takeSenderInfoFrom>
      </mailet>
      <mailet class="ToProcessor" match="HostIsLocal?includeSubdomains=no">
         <processor>incoming</processor>
      </mailet>
      <mailet class="ToProcessor" match="All">
         <processor>outgoing</processor>
      </mailet>
   </processor>
```

Il file XML configura le classi, ma non le implementa. `SimpleLogger` è, ad esempio, una classe fornita da totemomail o Kiteworks, il cui codice sorgente non è disponibile nell'appliance. La guida nella GUI di amministrazione ne spiega tuttavia i parametri:

- `log-message` definisce il testo del log ed è obbligatorio.
- `showSenderEmailAddress` aggiunge, se desiderato, l'indirizzo del mittente.
- `showRecipientsEmailAddress` aggiunge gli indirizzi dei destinatari.
- `showSubject` aggiunge l'oggetto.

L'ordine **all'interno** di un processor è vincolante. Un matcher può selezionare nessuno, tutti o solo una parte dei destinatari. Nel caso di un sottoinsieme, James divide il messaggio: i destinatari corrispondenti passano attraverso il mailet, mentre gli altri vengono elaborati separatamente. Se un mailet modifica successivamente lo stato, l'elaborazione passa immediatamente al processor indicato; le restanti regole del processor corrente vengono ignorate.

Per la risoluzione dei problemi ne risulta una procedura affidabile:

1. Determinare il repository e i file `FileStreamStore` / `FileObjectStore` associati.
2. Determinare lo `state` corrente in `FileObjectStore`.
3. Cercare il processor con lo stesso nome in `totemomail_config.xml`.
4. Verificare matcher e mailet nel loro ordine effettivo.
5. In caso di cambio di stato, proseguire nel processor di destinazione.

In questo modo è possibile seguire un flusso di posta passo dopo passo, senza leggere erroneamente il file XML dall'alto verso il basso come un programma lineare.

## Fonti

1.  [Apache James – Pagina del progetto](https://james.apache.org/): MTA open source su cui si basano tecnicamente totemomail e Kiteworks EPG.
    
2.  [Apache James – «Spool Manager»](https://james.apache.org/server/head/spoolmanager.html): elaborazione delle e-mail in entrata, spool e code.
    
3.  [Apache James – «Spool Manager Configuration»](https://james.apache.org/server/head/spoolmanager_configuration.html): configurazione dei processor e ordine delle regole.
    
4.  [Apache James – «Mailet API»](https://james.apache.org/server/head/mailet_api.html): concetto di mailet e matcher alla base delle regole.
    
5.  [Apache James – «MailImpl» (documentazione API)](https://james.apache.org/server/3/apidocs/org/apache/james/core/MailImpl.html): modello a oggetti Mail alla base di FileStreamStore e FileObjectStore.
    
6.  [IETF – RFC 822](https://datatracker.ietf.org/doc/html/rfc822): formato dei messaggi di testo Internet (header e body).
    
7.  [Microsoft Learn – «Connectors for secure mail flow with a partner»](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-for-secure-mail-flow-with-a-partner): configurazione del connettore per il flusso di posta sicuro tra Exchange Online e il gateway.
