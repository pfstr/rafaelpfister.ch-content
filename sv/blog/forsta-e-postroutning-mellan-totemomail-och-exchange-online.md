---
title: "Förstå e-postrouting mellan totemomail och Exchange Online"
navTitle: "Apache James ↔ M365"
description: "Hur totemomail lagrar och bearbetar meddelanden, hur den underliggande Apache James växlar mellan processorer och vad som krävs för en säker e-postslinga med Exchange Online."
date: "2026-06-17"
kategorie: "Totemomail"
timeToRead: "10 min. lästid"
themen:
  - totemomail
slug: "forsta-e-postroutning-mellan-totemomail-och-exchange-online"
translationOf: "totemomail-m365"
translationId: article-60a86616507315fa
translationReview: automatic
translationSourceHash: d92dbf8dac2f4fe0cd6e28f70e9e5575c76b56acb4afa5c5444f9e10895b4abf
translatedAt: 2026-10-14T08:58:33.901Z
url: https://rafaelpfister.ch/sv/blog/forsta-e-postroutning-mellan-totemomail-och-exchange-online
translationModel: gpt-5.6-terra
---

# Förstå e-postrouting mellan totemomail och Exchange Online

I en e-postslinga mellan Exchange Online och totemomail har varje system en tydligt avgränsad uppgift. Exchange Online tillhandahåller brevlådorna. Totemomail, eller dagens Kiteworks Email Protection Gateway, hanterar kryptering, signaturer, policyer och särskilda routningsregler.

För att detta ska bli ett tillförlitligt e-postflöde räcker det inte att konfigurera två SMTP-connectors. Vid felsökning måste det också vara tydligt vad som sker i gatewayen efter att ett meddelande har tagits emot: Var finns det? Vilken regel körs härnäst? Och varför kan ett meddelande vänta i en kö trots att SMTP-dialogen redan har slutförts?

Därför förklarar den här artikeln bearbetningsmodellen i [Apache James](https://james.apache.org/), som totemomail bygger på. Den konkreta routningskonfigurationen beror på den aktuella miljön, men de beskrivna processorerna, matchers, mailets och repositories utgör den tekniska grunden för varje installation.

En viktig säkerhetsregel gäller oberoende av detaljerna: Om totemomail är den främre gatewayen får Exchange Online endast ta emot e-post från internet från denna gateway. Detta kräver en restriktiv partner-connector. Enbart en MX-post blockerar inte den direkta leveransvägen. Artikeln [En MX-post är ingen brandvägg](/blog/ghost-sender-exchange-online-nebeneingang) visar hur denna sidoingång uppstår och hur den stängs.

## Från SMTP-ingång till bearbetning

Bearbetningslogiken i Apache James består av fyra byggstenar:

- **Matchers** kontrollerar villkor och avgör för vilka mottagare en regel gäller.
- **Mailets** utför själva åtgärden, till exempel ändrar headers, krypterar, levererar eller avslutar den fortsatta bearbetningen.
- **Processors** sammanför matchers och mailets till ordnade bearbetningssteg.
- **Mail-repositories** lagrar meddelanden under bearbetningen eller efter ett fel.

Denna uppdelning är avgörande för analysen: Repositoryt besvarar frågan **var** ett meddelande finns. Processorn avgör **hur** det bearbetas vidare.

![Användning av James som SMTP-relä](../images/4CixEi383SY5WdvwMSGZ67odMU.png)

SMTP-servern accepterar anslutningen och läser meddelandet till slutet av avsnittet `DATA`. Därefter skapar James ett `MailImpl`-objekt. Det innehåller MIME-innehållet som `MimeMessage` samt de uppgifter som behövs för bearbetningen: avsändare, mottagare, status och ytterligare attribut.

I ett filbaserat repository lagrar James denna information separat:

- `FileStreamStore` innehåller hela RFC-822-/MIME-meddelandet som en byte-ström.
- `FileObjectStore` innehåller det serialiserade `MailImpl`-objektet med status och metadata.

Ett meddelande kan därför redan vara helt mottaget och lagrat, trots att dess innehållsmässiga bearbetning fortfarande återstår.

## Repositories och köer under `/var/mail`

De enskilda repositories visas i filsystemet som kataloger. Under normal drift stannar ett meddelande där endast mycket kort. Om en kö byggs upp tyder det oftast på en felaktig regel, ett oåtkomligt mål eller en backend-tjänst som har slutat fungera.

Följande exempel innehåller, utöver standardköerna, även valfria kataloger för en HIN-anslutning. HIN tillhandahåller den säkra kommunikationsmiljön för det schweiziska hälso- och sjukvårdsväsendet.

> Om du behöver hjälp med anslutningen av HIN-Mailgateway eller med migreringen till den nya HIN-Stargate-lösningen hittar du rätt experter hos [adeptio](https://adeptio.ch/en).  
>   
> **adeptio** är officiell partner till [Health Info Net AG](https://www.hin.ch/de/index.cfm) och har som sådan även direkta kontaktpersoner hos tillverkaren.  
> [➜ Boka ett möte redan i dag.](/kontakt)

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

## Så lagras ett meddelande i filsystemet

Varje lagrat meddelande består av två filer.

### `FileStreamStore`: Meddelandets innehåll

Filen `*.FileStreamStore` innehåller hela RFC-822-/MIME-meddelandet. Med `cat` kan headers och brödtext läsas:

```text
From:
To:
Subject:
...
Body
```

Det underliggande meddelandeformatet beskrivs i [RFC 822](https://datatracker.ietf.org/doc/html/rfc822).

### `FileObjectStore`: Status och metadata

Filen `*.FileObjectStore` är ett serialiserat Java-objekt av typen `org.apache.james.core.MailImpl`. Dess fält omfattar:

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

[API-dokumentationen för `MailImpl`](https://james.apache.org/server/3/apidocs/org/apache/james/core/MailImpl.html) beskriver objektmodellen i detalj.

## Statusen väljer nästa processor

Katalogstrukturen visar endast repositoryt. Det faktiska bearbetningsläget finns i fältet `state` i `FileObjectStore`. Dess värde hänvisar till attributet `name` i en processor.

Efter varje mailet kontrollerar SpoolManager denna status:

1. Om statusen förblir oförändrad följer nästa matcher-mailet-par i samma processor.
2. Om ett mailet ändrar statusen avslutar James den aktuella processorn och hoppar till processorn med samma namn.
3. Den särskilda statusen `ghost` avslutar bearbetningen helt.

De obligatoriska processorerna `root` och `error` har fasta uppgifter. Nya meddelanden startar i `root`; interna fel och mailets som konfigurerats för detta omdirigerar till `error`. Ordningen på elementen `<processor>` i XML-filen bestämmer däremot **inte** körordningen.

## Processorstruktur i `totemomail_config.xml`

Före varje ändring bör den aktuella `totemomail_config.xml` exporteras och säkerhetskopieras:

![Configuration / Open Current / Export to File](../images/kWKIN3vramf0IAuPjzioEGV4Znw.png)

De olika processorerna och mailets som ingår i dem visas i totemomail\_config.xml. Här är återigen ett exempel från praktiken:

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

Även om `root` står sist i detta utdrag börjar varje nytt meddelande där. Det är namnet, inte placeringen i dokumentet, som är avgörande.

Processorn `root` innehåller själv en ordnad lista med matcher-mailet-par:

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

XML-filen konfigurerar klasserna men implementerar dem inte. `SimpleLogger` är till exempel en klass som tillhandahålls av totemomail respektive Kiteworks, vars källkod inte är öppen i appliance-miljön. Hjälpen i administratörsgränssnittet förklarar dock parametrarna:

- `log-message` definierar loggtexten och är obligatorisk.
- `showSenderEmailAddress` lägger vid behov till avsändaradressen.
- `showRecipientsEmailAddress` lägger till mottagaradresserna.
- `showSubject` lägger till ämnesraden.

Ordningen **inom** en processor är bindande. En matcher kan välja inga, alla eller endast en del av mottagarna. Vid en delmängd delar James upp meddelandet: De matchande mottagarna går igenom mailet, medan de övriga bearbetas vidare separat. Om ett mailet därefter ändrar statusen hoppar bearbetningen omedelbart till den angivna processorn; återstående regler i den aktuella processorn hoppas över.

För felsökning ger detta ett tillförlitligt arbetssätt:

1. Fastställ repositoryt och tillhörande filer `FileStreamStore`-/`FileObjectStore`.
2. Identifiera den aktuella `state` i `FileObjectStore`.
3. Sök efter processorn med samma namn i `totemomail_config.xml`.
4. Kontrollera matchers och mailets i deras faktiska ordning.
5. Fortsätt i målprocessorn vid en statusändring.

På så sätt kan ett e-postflöde följas steg för steg utan att felaktigt läsa XML-filen uppifrån och ned som ett linjärt program.

## Källor

1.  [Apache James – projektsida](https://james.apache.org/): MTA med öppen källkod som totemomail respektive Kiteworks EPG tekniskt bygger på.
    
2.  [Apache James – «Spool Manager»](https://james.apache.org/server/head/spoolmanager.html): Bearbetning av inkommande e-post, spool och köer.
    
3.  [Apache James – «Spool Manager Configuration»](https://james.apache.org/server/head/spoolmanager_configuration.html): Processorkonfiguration och ordning för reglerna.
    
4.  [Apache James – «Mailet API»](https://james.apache.org/server/head/mailet_api.html): Mailet- och matcher-konceptet bakom reglerna.
    
5.  [Apache James – «MailImpl» (API-dokumentation)](https://james.apache.org/server/3/apidocs/org/apache/james/core/MailImpl.html): Objektmodell för e-post bakom FileStreamStore och FileObjectStore.
    
6.  [IETF – RFC 822](https://datatracker.ietf.org/doc/html/rfc822): Format för textmeddelanden på internet (headers och brödtext).
    
7.  [Microsoft Learn – «Connectors for secure mail flow with a partner»](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-for-secure-mail-flow-with-a-partner): Connector-konfiguration för säkert e-postflöde mellan Exchange Online och gatewayen.
