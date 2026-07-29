---
title: "Understanding mail routing between totemomail and Exchange Online"
navTitle: "Apache James ↔ M365"
description: "How totemomail stores and processes messages, how the underlying Apache James switches between processors, and what matters for a secure mail loop with Exchange Online."
date: "2026-06-17"
kategorie: "Totemomail"
timeToRead: "10 min read"
themen:
  - totemomail
slug: "apache-james-exchange-online-mail-routing"
translationOf: "totemomail-m365"
url: "https://rafaelpfister.ch/en/blog/apache-james-exchange-online-mail-routing"
translationId: article-60a86616507315fa
translatedAt: 2026-07-28T11:10:30.444Z
translationReview: automatic
translationSourceHash: 8dabf54e50de750dbd1e13baf487ccb1fa9db0d7bd98afcd1933e87bdb57f0af
---

# Understanding mail routing between totemomail and Exchange Online

In a mail loop between Exchange Online and totemomail, each system has a clearly defined role. Exchange Online provides the mailboxes. Totemomail, now known as the Kiteworks Email Protection Gateway, handles encryption, signatures, policies and special routing rules.

To create a reliable mail flow, it is not enough to configure two SMTP connectors. For troubleshooting, it must also be clear what happens inside the gateway after a message has been accepted: Where is it stored? Which rule is executed next? And why can a message wait in a queue even though the SMTP dialogue has already completed successfully?

This article therefore explains the processing model of [Apache James](https://james.apache.org/), on which totemomail is built. The specific routing configuration depends on the respective environment; however, the processors, matchers, mailets and repositories described form the technical foundation of every installation.

One important security rule applies regardless of the details: If totemomail is the upstream gateway, Exchange Online must accept internet mail only from this gateway. This requires a restrictive partner connector. An MX record alone does not block the direct delivery path. The article [An MX record is not a firewall](/blog/ghost-sender-exchange-online-nebeneingang) shows how this side entrance arises and how to close it.

## From SMTP ingress to processing

The processing logic of Apache James consists of four components:

- **Matchers** check conditions and determine which recipients a rule applies to.
- **Mailets** perform the actual action, such as modifying headers, encrypting, delivering or ending further processing.
- **Processors** combine matchers and mailets into ordered processing steps.
- **Mail repositories** store messages during processing or after an error.

This separation is crucial for analysis: The repository answers the question of **where** a message is located. The processor determines **how** it is processed further.

![Using James as an SMTP relay](../images/4CixEi383SY5WdvwMSGZ67odMU.png)

The SMTP server accepts the connection and reads the message through to the end of the `DATA` section. James then creates a `MailImpl` object. It contains the MIME content as `MimeMessage` as well as the information required for processing: sender, recipients, status and further attributes.

With a file-based repository, James stores this information separately:

- `FileStreamStore` contains the complete RFC 822/MIME message as a byte stream.
- `FileObjectStore` contains the serialised `MailImpl` object with status and metadata.

A message may therefore already have been fully accepted and stored even though its business processing is still pending.

## Repositories and queues under `/var/mail`

The individual repositories appear as directories in the file system. Under normal operation, a message remains there only very briefly. If a queue backs up, this usually indicates a faulty rule, an unreachable destination or a failed backend service.

In addition to the standard queues, the following example also contains optional directories for a HIN connection. HIN provides the secure communication environment for the Swiss healthcare sector.

> If you need support connecting the HIN mail gateway or migrating to the new HIN Stargate solution, you can find the relevant experts at [adeptio](https://adeptio.ch/).  
>   
> **adeptio** is an official partner of [Health Info Net AG](https://www.hin.ch/de/index.cfm) and, as such, also has direct contacts at the manufacturer.  
> [➜ Book an appointment today.](https://outlook.office.com/bookwithme/user/b4d64d6bdbca4b489074d459cd30b50c@adeptio.ch/meetingtype/3Wgk7rXJfk261852Hyovkg2?anonymous&ismsaljsauthenabled&ep=mlink)

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

## How a message is stored in the file system

Each stored message consists of two files.

### `FileStreamStore`: Message content

The `*.FileStreamStore` file contains the complete RFC 822/MIME message. With `cat`, the headers and body are readable:

```text
From:
To:
Subject:
...
Body
```

The underlying message format is described in [RFC 822](https://datatracker.ietf.org/doc/html/rfc822).

### `FileObjectStore`: Status and metadata

The `*.FileObjectStore` file is a serialised Java object of type `org.apache.james.core.MailImpl`. Its fields include:

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

## Status selects the next processor

The directory structure shows only the repository. The actual processing state is held in the `state` field of `FileObjectStore`. Its value refers to the `name` attribute of a processor.

After each mailet, the SpoolManager checks this status:

1. If the status remains unchanged, the next matcher-mailet pair in the same processor follows.
2. If a mailet changes the status, James ends the current processor and jumps to the processor with the same name.
3. The special status `ghost` ends processing completely.

The mandatory processors `root` and `error` have fixed tasks. New messages start in `root`; internal errors and appropriately configured mailets redirect to `error`. By contrast, the order of the `<processor>` elements in the XML file does **not** determine the execution order.

## Processor structure in `totemomail_config.xml`

Before making any changes, export and back up the current `totemomail_config.xml`:

![Configuration / Open Current / Export to File](../images/kWKIN3vramf0IAuPjzioEGV4Znw.png)

The various processors and the mailets they contain can be found in totemomail\_config.xml. Here is another practical example:

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

Although `root` appears at the end of this excerpt, every new message starts there. The name is decisive, not its position in the document.

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

The XML file configures the classes but does not implement them. `SimpleLogger`, for example, is a class provided by totemomail or Kiteworks whose source code is not available in the appliance. However, the help in the admin GUI explains its parameters:

- `log-message` defines the log text and is mandatory.
- `showSenderEmailAddress` optionally adds the sender address.
- `showRecipientsEmailAddress` adds the recipient addresses.
- `showSubject` adds the subject.

The order **within** a processor is binding. A matcher may select no recipients, all recipients or only some recipients. For a subset, James splits the message: the matching recipients pass through the mailet, while the remaining recipients are processed separately. If a mailet then changes the status, processing immediately jumps to the specified processor; the remaining rules of the current processor are skipped.

This results in a reliable troubleshooting procedure:

1. Identify the repository and the associated `FileStreamStore`/`FileObjectStore` files.
2. Determine the current `state` in `FileObjectStore`.
3. Find the processor with the same name in `totemomail_config.xml`.
4. Check matchers and mailets in their actual order.
5. If the status changes, continue in the target processor.

This makes it possible to trace a mail flow step by step without mistakenly reading the XML file from top to bottom as a linear programme.

## Sources

1.  [Apache James – project page](https://james.apache.org/): Open-source MTA on which totemomail or Kiteworks EPG is technically based.
    
2.  [Apache James – “Spool Manager”](https://james.apache.org/server/head/spoolmanager.html): Processing of incoming mail, spool and queues.
    
3.  [Apache James – “Spool Manager Configuration”](https://james.apache.org/server/head/spoolmanager_configuration.html): Processor configuration and rule order.
    
4.  [Apache James – “Mailet API”](https://james.apache.org/server/head/mailet_api.html): The mailet and matcher concept behind the rules.
    
5.  [Apache James – “MailImpl” (API documentation)](https://james.apache.org/server/3/apidocs/org/apache/james/core/MailImpl.html): Mail object model behind FileStreamStore and FileObjectStore.
    
6.  [IETF – RFC 822](https://datatracker.ietf.org/doc/html/rfc822): Format of internet text messages (headers and body).
    
7.  [Microsoft Learn – “Connectors for secure mail flow with a partner”](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-for-secure-mail-flow-with-a-partner): Connector configuration for secure mail flow between Exchange Online and the gateway.
