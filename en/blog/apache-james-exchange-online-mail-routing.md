---
title: "Set Up Email Routing Between Apache James (Totemomail / Kiteworks EPG) and Exchange Online"
navTitle: "Apache James ↔ M365"
description: "SMTP Routing, Apache James Internals, and What It Takes to Ensure Emails Flow Smoothly Between Exchange Online and Apache James"
date: "2026-06-17"
kategorie: "Totemomail"
timeToRead: "3 min to read"
themen:
  - "totemomail"
slug: "apache-james-exchange-online-mail-routing"
translationOf: "totemomail-m365"
url: "https://rafaelpfister.ch/en/blog/apache-james-exchange-online-mail-routing"
---

# Set Up Email Routing Between Apache James (Totemomail / Kiteworks EPG) and Exchange Online

In modern email architectures, it is often necessary to connect different email platforms: whether as part of migrations, hybrid environments, or to integrate MTAs that handle specialized tasks. This will be illustrated using the example of a mail loop between a gateway system such as TotemoMail (based on Apache James) and a cloud service such as Exchange Online.

While Exchange Online primarily serves as the destination or source system for user mailboxes, TotemoMail acts as a mail gateway for encryption, signing, policy enforcement, or specialized routing logic. To ensure these components work together seamlessly, incoming and outgoing messages must be routed between the systems in a controlled manner, without loops, delivery errors, or unexpected side effects.

Once the mail loop is in place, one point matters most: Exchange Online must accept mail exclusively from TotemoMail, not directly from the internet. For how that gets locked down with a restrictive partner connector, and what happens when that step is missing, see [Ghost Sender or Ghost Admin?](/en/blog/ghost-sender-exchange-online-side-entrance)

However, setting up such an email loop is anything but trivial. In addition to traditional SMTP routing issues, Apache James's internal mechanisms also play a crucial role.

It is precisely this internal processing flow (hidden behind XML configurations) that is crucial for an email to function correctly within the system.

This post covers:

-   how to set up an email loop between TotemoMail and Exchange Online
    
-   How Apache James processes and forwards emails internally
    
-   and what you need to keep in mind to ensure clean, stable email flows
    

The focus here is not only on configuration, but also on understanding the underlying mechanics. Only those who truly understand the flow of email can analyze and prevent errors in a targeted manner.

### Spool Manager (Processing Incoming Emails)

The XML describes the configuration of the Totemomail mail-processing pipeline (the underlying principle is [Apache James](https://james.apache.org/)). It describes how emails are processed, encrypted, decrypted, routed, and delivered.

Totemomail is built on Apache James and uses its Mailet containers. The Mailet container's processing model is based on four key components:

-   **Mailets** perform specific actions on emails, such as modifying content or headers, triggering actions, or terminating processing.
    
-   **Matchers** define conditions and determine whether an email should be processed by a specific Mailet.
    
-   **Processors** organize these matcher and mailet combinations as sequential processing steps in a pipeline.
    
-   **Mail-Repositories** serve as storage locations for emails during or after processing.
    

This modular structure allows administrators to flexibly assemble complex processing logic. In Totemomail (as well as in Apache James in general), the processing logic looks like this:

![Using James as an SMTP relay](../images/4CixEi383SY5WdvwMSGZ67odMU.png)

Before an email can be processed in Apache James, it must first be delivered to the internal mail repository. This is handled by the SMTP server, which accepts incoming connections and receives messages via the SMTP protocol. In modern versions, this SMTP server is based on Netty, an asynchronous network engine that processes incoming TCP connections. As soon as an email has been fully transmitted via SMTP (specifically after the DATA command), it is converted into an internal object model within James.

Specifically, the message is converted into a so-called MailImpl object. This object not only encapsulates the actual content of the email (as a MimeMessage), but also supplements it with additional metadata such as the sender, recipient, processing status, and other attributes required for internal routing and processing. From this point on, the actual mail processing takes place via the configured processors, matchers, and mailets.

During persistence, this object is split into two parts in the FileMailRepository:

-   The MIME content is stored as a raw byte stream in the FileStreamStore
    
-   The Java object (MailImpl) is stored in the FileObjectStore using Java serialization
    

In doing so, James deliberately separates the message content from the processing state. Let's now take a look at how this is implemented in the system.

##### Structure of the /var/mail Repository (the various queues)

Queuing takes place through various folders (known as repositories). This means that emails are temporarily stored to be processed later. In reality, of course, this happens in fractions of a second. However, if there is a configuration error, the emails may end up in the *spool*\-Wait in the queue until it's processed. Simply put:

-   Repository = where is the email stored?
    
-   Processor = How is the email processed?
    

Here is an example setup with various repositories (including those used specifically for a HIN encryption gateway connection). HIN is a secure trust environment for the Swiss healthcare system, distributed by Health Infonet AG.

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
│   → Fehlgeschlagene Mails (z. B. Policy, Encryption, Routing)
│
├── DBUnavailable/
│   → Mails, die wegen Backend-/DB-Problemen nicht verarbeitet werden konnten
```

##### Structure of Emails Within the Totemomail File System

As previously described, an email within the repository consists of two separate files:

###### File Stream Store

\*.FileStreamStore = contains the MIME content of the emails as complete RFC822/MIME emails

A \`cat\` dump displays the following content:

```text
From:
To:
Subject:
...
Body
```

See [RFC 822 - STANDARD FOR THE FORMAT OF ARPA INTERNET TEXT MESSAGES](https://datatracker.ietf.org/doc/html/rfc822)

###### FileObjectStore

\*.FileObjectStore = serialized Java object (org.apache.james.core.MailImpl) containing the current processing state and metadata

A \`cat\` dump displays the following content:

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

See [MailImpl (Apache James Server 3.0-beta5-SNAPSHOT API)](https://james.apache.org/server/3/apidocs/org/apache/james/core/MailImpl.html)

### Processor (Which rules are applied, and in what order?)

We learned above that the processing logic in Apache James is completely decoupled from the memory model. The directory structure reflects only the queue and not the current processing*condition* the email. The processing status is stored in the metadata (\*.FileObjectStore), more specifically in the *state*\-parameter. According to the specification, any string can generally be defined here. This corresponds to the **name**\-attribute of the processor. The official James documentation states the following:

> [**James Server - James 2.3 - Configuring the SpoolManager**](https://james.apache.org/server/2.3.1/spoolmanager_configuration.html)
> 
> Each processor has a required attribute, **name**. The value of this attribute must be unique for each processor tag. The name of a processor is significant. Certain processors are required (specifically root and error). The name "ghost" is forbidden as a processor name, as it is used to denote a message that should not undergo any further processing.
> 
> The James SpoolManager creates a correspondance between processor names and the "state" of a mail as defined in the Mailet API. Specifically, after each mailet processes a mail, the state of the message is examined. If the state has been changed, the message does not continue in the current processor. If the new state is "ghost" then processing of that message terminates completely. If the new state is anything else, the message is re-routed to the processor with the name matching the new state.
> 
> The root processor is a required processor. All new messages that the SpoolManager finds on the spool are directed to this processor.
> 
> The error processor is another required processor. Under certain circumstances James itself will redirect messages to the error processor. It is also the standard processor to which mailets redirect messages when an error condition is encountered.

The state is therefore the key to invoking the so-called "processor" within the Totemomail ruleset. The next state is then set within the processor. From this, an important architectural principle for the order in which processors are processed can be defined. Totemomail does not process data linearly in the sense of Processor1 → Processor2 → Processor3, but rather according to the following principle:

1.  state → processor
    
2.  processor → sets a new state
    
3.  → next processor
    

#### Processor Structure in the Totemomail Ruleset (totemomail\_config.xml)

Before making any changes to the system, you should create a backup of the totemomail\_config.xml file. The current Totemomail ruleset can be downloaded as follows:

![Configuration / Open Current / Export to File](../images/kWKIN3vramf0IAuPjzioEGV4Znw.png)

The various processors and the mailets they contain are listed in totemomail\_config.xml. Here is another real-world example:

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

Here, it is clear that the ruleset is not processed linearly. The *root*\-Processor can also be placed at the very end and will still be processed first.

Now, if we look at the *root*\-If you take a closer look at the processor, it becomes clear that very little of the logic is visible in the XML:

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

So this isn't the implementation of the logic, but merely a configuration file. The actual logic is located here, for example, in the class that is called *SimpleLogger*. Apparently, the class appears to be a custom development by Totemomail (now Kiteworks), so the code cannot be viewed directly and is available only as compiled bytecode.

In the GUI, however, you can display a help text by right-clicking on the corresponding mail item:

> This action gives you the possibility to insert own log messages in the log file. The name and the location of the log file depend on the property totemo.mailer.logFile
> 
> **Parameters:**  
> **log-message (required)**: The text to log.  
> **showSenderEmailAddress (optional)**: Set it to "true" if the log message should contain the senders email address  
> **showRecipientsEmailAddress (optional)**: Set it to "true" if the log message should contain the recipients email addresses  
> **showSubject (optional)**: Set it to "true" if the log message should contain the subject

The following information regarding the logic behind the execution of individual matchers and mailets can be found in James' documentation:

> [James Server - James 2.3 - The SpoolManager, Matchers, and Mailets](https://james.apache.org/server/head/spoolmanager.html)
> 
> Matchers and mailets are used in pairs. At each stage in processing a message is checked against a matcher. The matcher will attempt to match the mail message. The match is not simply a yes or no issue. Instead, the match method returns a collection of matched recipients. If the this collection of matched recipients is empty, the mailet is not invoked. If the collection of matched recipients is the entire set of original recipients, the mail is then processed by the associated mailet. Finally, if the matcher only matches a proper subset of the original recipients, the original mail is duplicated. The recipients for one message are set to the matched recipients, and that message is processed by the mailet. The recipients for the other mail are set to the non-matching recipients, and that message is not processed by the mailet.
> 
> More on matchers and mailets can be found [here](https://james.apache.org/server/head/mailet_api.html).
> 
> One level up from the matchers and mailets are the processors. Each processor is a list of matcher/mailet pairs. **During mail processing, mail messages will be processed by each pair, in order.** In most cases, the message will be processed by all the pairs in the processor. **However, it is possible for a mailet to change the state of the mail message so it is immediately directed to another processor, and no additional processing occurs in the current processor.** Typically this occurs when the mailet wants to prevent future processing of this message (i.e. the mail message has been delivered locally, and hence requires no further processing) or when the mail message has been identified as a candidate for special processing (i.e. the message is spam and thus should be routed to the spam processor). **Because of this redirection, the processors in the SpoolManager form a tree. The root processor, which must be present, is the root of this tree.**
> 
> The SpoolManager continually checks for mail in the spool repository. When mail is first found in the repository, it is delivered to the root processor. Mail can be placed on this spool from a number of sources (SMTP, FetchPOP, a custom component). This spool repository is also used for storage of mail that is being redirected from one processor to another. Mail messages are driven through the processor tree until they reach the end of a processor or are marked completed by a mailet.
> 
> More on configuration of the SpoolManager can be found [here](https://james.apache.org/server/head/spoolmanager_configuration.html).

## Sources

1.  [Apache James – Project Page](https://james.apache.org/) — An open-source MTA on which totemomail and Kiteworks EPG are technically based.
    
2.  [Apache James – “Spool Manager”](https://james.apache.org/server/head/spoolmanager.html) — Processing incoming emails, spool files, and queues.
    
3.  [Apache James – «Spool Manager Configuration»](https://james.apache.org/server/head/spoolmanager_configuration.html) — Processor configuration and order of the rules.
    
4.  [Apache James – “Mail API”](https://james.apache.org/server/head/mailet_api.html) — The Mailet and Matcher concepts behind the Rules.
    
5.  [Apache James – “MailImpl” (API Documentation)](https://james.apache.org/server/3/apidocs/org/apache/james/core/MailImpl.html) — Mail object model behind FileStreamStore and FileObjectStore.
    
6.  [IETF – RFC 822](https://datatracker.ietf.org/doc/html/rfc822) — Format of Internet text messages (header and body).
    
7.  [Microsoft Learn – «Connectors for secure mail flow with a partner»](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-for-secure-mail-flow-with-a-partner) — Connector configuration for secure email flow between Exchange Online and the gateway.
