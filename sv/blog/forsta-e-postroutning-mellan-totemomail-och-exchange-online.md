---
title: "Förstå e-postroutning mellan totemomail och Exchange Online"
navTitle: "Apache James ↔ M365"
description: "Hur totemomail lagrar och behandlar meddelanden, hur den underliggande Apache James växlar mellan processorer och vad som krävs för en säker e-postslinga med Exchange Online."
date: "2026-06-17"
kategorie: "Totemomail"
timeToRead: "10 min. lästid"
themen:
  - totemomail
slug: "forsta-e-postroutning-mellan-totemomail-och-exchange-online"
translationOf: "totemomail-m365"
url: "https://rafaelpfister.ch/sv/blog/forsta-e-postroutning-mellan-totemomail-och-exchange-online"
translationId: article-60a86616507315fa
translationReview: automatic
translationSourceHash: 8dabf54e50de750dbd1e13baf487ccb1fa9db0d7bd98afcd1933e87bdb57f0af
translatedAt: 2026-07-29T12:29:38.959Z
---

# Förstå e-postroutning mellan totemomail och Exchange Online

I en e-postslinga mellan Exchange Online och totemomail har varje system en tydligt avgränsad uppgift. Exchange Online tillhandahåller postlådorna. Totemomail, eller dagens Kiteworks Email Protection Gateway, hanterar kryptering, signaturer, principer och särskilda routningsregler.

För att få ett tillförlitligt e-postflöde räcker det inte att konfigurera två SMTP-anslutningar. Vid felsökning måste det även vara tydligt vad som händer med ett meddelande i gatewayen efter att det har tagits emot: Var finns det? Vilken regel körs härnäst? Och varför kan ett meddelande vänta i en kö trots att SMTP-dialogen redan har slutförts?

Därför förklarar denna artikel bearbetningsmodellen i [Apache James](https://james.apache.org/), som totemomail bygger på. Den konkreta routningskonfigurationen beror på den aktuella miljön, men de beskrivna processorerna, matcharna, maileten och repositorierna utgör den tekniska grunden för varje installation.

En viktig säkerhetsregel gäller oavsett detaljerna: Om totemomail är den framförliggande gatewayen får Exchange Online endast ta emot internet-e-post från denna gateway. Det kräver en restriktiv partneranslutning. En MX-post blockerar inte den direkta leveransvägen på egen hand. Artikeln [En MX-post är ingen brandvägg](/blog/ghost-sender-exchange-online-nebeneingang) visar hur denna sidoingång uppstår och hur den stängs.

## Från SMTP-ingång till bearbetning

Bearbetningslogiken i Apache James består av fyra byggstenar:

- **Matchers** kontrollerar villkor och avgör för vilka mottagare en regel gäller.
- **Mailets** utför själva åtgärden, till exempel ändrar rubriker, krypterar, levererar eller avslutar den fortsatta bearbetningen.
- **Processors** sammanfattar matchers och mailets i ordnade bearbetningssteg.
- **Mail-repositories** lagrar meddelanden under bearbetningen eller efter ett fel.

Denna uppdelning är avgörande för analysen: Repositoriet besvarar frågan **var** ett meddelande finns. Processorn avgör **hur** det bearbetas vidare.

![Användning av James som SMTP-relä](../images/4CixEi383SY5WdvwMSGZ67odMU.png)

SMTP-servern accepterar anslutningen och läser meddelandet till slutet av avsnittet `DATA`. Därefter skapar James ett `MailImpl`-objekt. Det innehåller MIME-innehållet som `MimeMessage` samt de uppgifter som behövs för bearbetningen: avsändare, mottagare, status och ytterligare attribut.

I ett filbaserat repository lagrar James denna information separat:

- `FileStreamStore` innehåller det fullständiga RFC-822-/MIME-meddelandet som en byteström.
- `FileObjectStore` innehåller det serialiserade `MailImpl`-objektet med status och metadata.

Ett meddelande kan därför redan ha tagits emot och sparats fullständigt, även om dess verksamhetsmässiga bearbetning fortfarande väntar.

## Repositories och köer under `/var/mail`

De enskilda repositorierna visas i filsystemet som kataloger. Vid normal drift stannar ett meddelande där bara mycket kort. Om en kö byggs upp tyder det vanligtvis på en felaktig regel, ett mål som inte går att nå eller en backend-tjänst som har slutat fungera.

Följande exempel innehåller, utöver standardköerna, även valfria kataloger för en HIN-anslutning. HIN tillhandahåller den säkra kommunikationsmiljön för det schweiziska hälso- och sjukvårdsväsendet.

> Om du behöver hjälp med anslutningen till HIN Mailgateway eller med migreringen till den nya HIN Stargate-lösningen hittar du rätt experter hos [adeptio](https://adeptio.ch/).  
>   
> **adeptio** är officiell partner till [Health Info Net AG](https://www.hin.ch/de/index.cfm) och har som sådan även direkta kontaktpersoner hos tillverkaren.  
> [➜ Boka en tid redan i dag.](https://outlook.office.com/bookwithme/user/b4d64d6bdbca4b489074d459cd30b50c@adeptio.ch/meetingtype/3Wgk7rXJfk261852Hyovkg2?anonymous&ismsaljsauthenabled&ep=mlink)

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

Filen `*.FileStreamStore` innehåller det fullständiga RFC-822-/MIME-meddelandet. Med `cat` går rubriker och brödtext att läsa:

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

Katalogstrukturen visar endast repositoriet. Det egentliga bearbetningsläget finns i fältet `state` i `FileObjectStore`. Dess värde hänvisar till attributet `name` för en processor.

Efter varje mailet kontrollerar SpoolManager denna status:

1. Om statusen förblir oförändrad följer nästa matcher-mailet-par i samma processor.
2. Om ett mailet ändrar statusen avslutar James den aktuella processorn och hoppar till processorn med samma namn.
3. Den särskilda statusen `ghost` avslutar bearbetningen helt.

De obligatoriska processorerna `root` och `error` har fasta uppgifter. Nya meddelanden startar i `root`; interna fel och mailets som konfigurerats för detta vidarebefordrar till `error`. Ordningen på elementen `<processor>` i XML-filen avgör däremot **inte** körordningen.

## Processorstruktur i `totemomail_config.xml`

Före varje ändring bör den aktuella `totemomail_config.xml` exporteras och säkerhetskopieras:

![Configuration / Open Current / Export to File](../images/kWKIN3vramf0IAuPjzioEGV4Znw.png)

De olika processorerna och maileten som de innehåller visas i totemomail\_config.xml. Här följer åter ett exempel från praktiken:

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

Även om `root` står sist i detta utdrag börjar varje nytt meddelande där. Namnet är avgörande, inte positionen i dokumentet.

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

XML-filen konfigurerar klasserna men implementerar dem inte. `SimpleLogger` är exempelvis en klass som tillhandahålls av totemomail eller Kiteworks och vars källkod inte är öppet tillgänglig i appliance-lösningen. Hjälpen i administratörsgränssnittet förklarar dock dess parametrar:

- `log-message` anger protokolltexten och är obligatorisk.
- `showSenderEmailAddress` lägger vid behov till avsändaradressen.
- `showRecipientsEmailAddress` lägger till mottagaradresserna.
- `showSubject` lägger till ämnet.

Ordningen **inom** en processor är bindande. En matcher kan välja inga, alla eller endast en del av mottagarna. Vid en delmängd delar James upp meddelandet: De matchande mottagarna passerar maileten, medan övriga bearbetas vidare separat. Om ett mailet därefter ändrar statusen hoppar bearbetningen direkt till den angivna processorn; återstående regler i den aktuella processorn hoppas över.

För felsökning ger detta ett tillförlitligt arbetsflöde:

1. Fastställ repository och tillhörande filer `FileStreamStore`-/`FileObjectStore`.
2. Fastställ aktuell `state` i `FileObjectStore`.
3. Sök efter processorn med samma namn i `totemomail_config.xml`.
4. Kontrollera matchers och mailets i deras faktiska ordning.
5. Fortsätt i målprocessorn vid en statusändring.

På så sätt kan ett e-postflöde följas steg för steg, utan att felaktigt läsa XML-filen uppifrån och ned som ett linjärt program.

## Källor

1.  [Apache James – projektsida](https://james.apache.org/): MTA med öppen källkod som totemomail respektive Kiteworks EPG bygger på tekniskt.
    
2.  [Apache James – «Spool Manager»](https://james.apache.org/server/head/spoolmanager.html): Bearbetning av inkommande e-post, spool och köer.
    
3.  [Apache James – «Spool Manager Configuration»](https://james.apache.org/server/head/spoolmanager_configuration.html): Processorkonfiguration och ordning på reglerna.
    
4.  [Apache James – «Mailet API»](https://james.apache.org/server/head/mailet_api.html): Mailet- och matcher-konceptet bakom reglerna.
    
5.  [Apache James – «MailImpl» (API-dokumentation)](https://james.apache.org/server/3/apidocs/org/apache/james/core/MailImpl.html): E-postobjektmodellen bakom FileStreamStore och FileObjectStore.
    
6.  [IETF – RFC 822](https://datatracker.ietf.org/doc/html/rfc822): Format för internettextmeddelanden (rubriker och brödtext).
    
7.  [Microsoft Learn – «Connectors for secure mail flow with a partner»](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-for-secure-mail-flow-with-a-partner): Konfiguration av anslutningar för säkert e-postflöde mellan Exchange Online och gatewayen.
