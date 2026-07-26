---
title: "Understanding Mail Routing Between totemomail and Exchange Online"
navTitle: "Apache James ↔ M365"
description: "How totemomail stores and processes messages, how the underlying Apache James switches between processors, and what matters for a secure mail loop with Exchange Online."
date: "2026-06-17"
kategorie: "Totemomail"
timeToRead: "10 min to read"
themen:
  - "totemomail"
slug: "apache-james-exchange-online-mail-routing"
translationOf: "totemomail-m365"
url: "https://rafaelpfister.ch/en/blog/apache-james-exchange-online-mail-routing"
---

# Understanding Mail Routing Between totemomail and Exchange Online

In a mail loop between Exchange Online and totemomail, each system has a clearly defined role. Exchange Online provides the mailboxes. Totemomail, or today's Kiteworks Email Protection Gateway, handles encryption, signatures, policies, and special routing rules.

Turning that into a reliable mail flow takes more than setting up two SMTP connectors. For troubleshooting, you also need to understand what happens inside the gateway once a message has been accepted: Where does it end up? Which rule runs next? And why can a message sit waiting in a queue even though the SMTP dialog already completed successfully?

This post therefore explains the processing model of [Apache James](https://james.apache.org/), on which totemomail is built. The actual routing configuration depends on the specific environment, but the processors, matchers, mailets, and repositories described here form the technical foundation of every installation.

One important security rule applies regardless of the details: if totemomail sits in front as the gateway, Exchange Online must accept internet mail only from that gateway. That requires a restrictive partner connector. An MX record alone does not block the direct delivery path. The post [An MX Record Is Not a Firewall](/en/blog/ghost-sender-exchange-online-side-entrance) shows how this side entrance appears and how to close it.

## From SMTP Intake to Processing

Apache James's processing logic consists of four building blocks:

- **Matchers** check conditions and determine which recipients a rule applies to.
- **Mailets** perform the actual action, such as changing headers, encrypting, delivering, or ending further processing.
- **Processors** combine matchers and mailets into ordered processing steps.
- **Mail repositories** store messages during processing or after an error.

This separation is essential for analysis: the repository answers the question of **where** a message resides. The processor determines **how** it gets processed further.

![Using James as an SMTP relay](../images/4CixEi383SY5WdvwMSGZ67odMU.png)

The SMTP server accepts the connection and reads the message through to the end of the `DATA` section. James then creates a `MailImpl` object. It holds the MIME content as a `MimeMessage`, along with the information needed for processing: sender, recipients, status, and further attributes.

With a file-based repository, James stores this information separately:

- `FileStreamStore` contains the complete RFC 822/MIME message as a byte stream.
- `FileObjectStore` contains the serialized `MailImpl` object with status and metadata.

A message can therefore already be fully accepted and stored even though its business-logic processing is still pending.

## Repositories and Queues Under `/var/mail`

The individual repositories appear in the file system as directories. Under normal operation, a message stays there only very briefly. If a queue backs up, that usually points to a faulty rule, an unreachable destination, or a failed backend service.

In addition to the standard queues, the following example also includes optional directories for a HIN connection. HIN provides the secure communication space for the Swiss healthcare system.

> If you need assistance connecting to the HIN Mailgateway or migrating to the new HIN Stargate solution, you'll find the relevant experts at [adeptio](https://adeptio.ch/).  
>   
> **adeptio** is an official partner of the [Health Info Net AG](https://www.hin.ch/de/index.cfm) and, as such, also has direct points of contact at the manufacturer.  
> [➜ Book an appointment today.](https://outlook.office.com/book/Erstgesprchadeptio@adeptio.ch/s/Akxr6wxKAEGw3d5sEmi-AQ2?ismsaljsauthenabled)

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
│   → Fehlgeschlagene Mails (z. B. Policy, Encryption, Routing)
│
├── DBUnavailable/
│   → Mails, die wegen Backend-/DB-Problemen nicht verarbeitet werden konnten
```

## How a Message Is Stored in the File System

Each stored message consists of two files.

### `FileStreamStore`: Message Content

The `*.FileStreamStore` file contains the complete RFC 822/MIME message. Using `cat`, the header and body are readable:

```text
From:
To:
Subject:
...
Body
```

The underlying message format is described in [RFC 822](https://datatracker.ietf.org/doc/html/rfc822).

### `FileObjectStore`: Status and Metadata

The `*.FileObjectStore` file is a serialized Java object of type `org.apache.james.core.MailImpl`. Its fields include:

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

The [API documentation for `MailImpl`](https://james.apache.org/server/3/apidocs/org/apache/james/core/MailImpl.html) describes the object model in detail.

## The Status Selects the Next Processor

The directory structure shows only the repository. The actual processing state lives in the `state` field of the `FileObjectStore`. Its value refers to the `name` attribute of a processor.

After each mailet, the SpoolManager checks this status:

1. If the status stays unchanged, the next matcher-mailet pair in the same processor follows.
2. If a mailet changes the status, James ends the current processor and jumps to the processor of the same name.
3. The special status `ghost` ends processing entirely.

The mandatory `root` and `error` processors have fixed roles. New messages start in `root`; internal errors and mailets configured accordingly redirect to `error`. The order of the `<processor>` elements in the XML file, by contrast, does **not** determine the execution order.

## Processor Structure in `totemomail_config.xml`

Before making any change, export and back up the current `totemomail_config.xml`:

![Configuration / Open Current / Export to File](../images/kWKIN3vramf0IAuPjzioEGV4Znw.png)

The various processors and the mailets they contain are visible in totemomail_config.xml. Here's another real-world example:

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

Although `root` appears at the end of this excerpt, every new message starts there. What matters is the name, not its position in the document.

The `root` processor itself contains an ordered list of matcher-mailet pairs:

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

The XML file configures the classes but doesn't implement them. `SimpleLogger`, for instance, is a class provided by totemomail or Kiteworks whose source code isn't exposed on the appliance. The help text in the admin GUI, however, explains its parameters:

- `log-message` sets the log text and is required.
- `showSenderEmailAddress` optionally adds the sender address.
- `showRecipientsEmailAddress` adds the recipient addresses.
- `showSubject` adds the subject.

The order **within** a processor is binding. A matcher can select none, all, or only some of the recipients. For a subset, James splits the message: the matching recipients pass through the mailet, while the rest continue separately. If a mailet then changes the status, processing jumps immediately to the named processor; the remaining rules in the current processor are skipped.

This gives you a reliable troubleshooting workflow:

1. Identify the repository and the associated `FileStreamStore`/`FileObjectStore` files.
2. Determine the current `state` in the `FileObjectStore`.
3. Look up the processor of the same name in `totemomail_config.xml`.
4. Check the matchers and mailets in their actual order.
5. On a status change, continue in the target processor.

This lets you trace a mail flow step by step, without mistakenly reading the XML file top to bottom as a linear program.

## Sources

1.  [Apache James – Project Page](https://james.apache.org/): An open-source MTA on which totemomail and Kiteworks EPG are technically based.
    
2.  [Apache James – "Spool Manager"](https://james.apache.org/server/head/spoolmanager.html): Processing incoming emails, spool files, and queues.
    
3.  [Apache James – «Spool Manager Configuration»](https://james.apache.org/server/head/spoolmanager_configuration.html): Processor configuration and order of the rules.
    
4.  [Apache James – "Mailet API"](https://james.apache.org/server/head/mailet_api.html): The mailet and matcher concepts behind the rules.
    
5.  [Apache James – "MailImpl" (API Documentation)](https://james.apache.org/server/3/apidocs/org/apache/james/core/MailImpl.html): Mail object model behind FileStreamStore and FileObjectStore.
    
6.  [IETF – RFC 822](https://datatracker.ietf.org/doc/html/rfc822): Format of Internet text messages (header and body).
    
7.  [Microsoft Learn – «Connectors for secure mail flow with a partner»](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-for-secure-mail-flow-with-a-partner): Connector configuration for secure email flow between Exchange Online and the gateway.
