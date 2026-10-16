---
title: "Forstå e-postruting mellom totemomail og Exchange Online"
navTitle: "Apache James ↔ M365"
description: "Hvordan totemomail lagrer og behandler meldinger, hvordan underliggende Apache James veksler mellom processorer, og hva som er viktig for en sikker e-postsløyfe med Exchange Online."
date: "2026-06-17"
kategorie: "Totemomail"
timeToRead: "10 min lesetid"
themen:
  - totemomail
slug: "forsta-e-postruting-mellom-totemomail-og-exchange-online"
translationOf: "totemomail-m365"
translationId: article-60a86616507315fa
translationReview: automatic
translationSourceHash: d92dbf8dac2f4fe0cd6e28f70e9e5575c76b56acb4afa5c5444f9e10895b4abf
translatedAt: 2026-10-14T08:58:57.483Z
url: https://rafaelpfister.ch/no/blog/forsta-e-postruting-mellom-totemomail-og-exchange-online
translationModel: gpt-5.6-terra
---

# Forstå e-postruting mellom totemomail og Exchange Online

I en e-postsløyfe mellom Exchange Online og totemomail har hvert system en tydelig avgrenset oppgave. Exchange Online tilbyr postboksene. Totemomail, eller dagens Kiteworks Email Protection Gateway, håndterer kryptering, signaturer, retningslinjer og spesielle rutingsregler.

For at dette skal bli en pålitelig e-postflyt, holder det ikke å sette opp to SMTP-koblinger. Ved feilsøking må det også være klart hva som skjer etter at en melding er mottatt i gatewayen: Hvor ligger den? Hvilken regel kjøres deretter? Og hvorfor kan en melding vente i en kø selv om SMTP-dialogen allerede er fullført?

Denne artikkelen forklarer derfor behandlingsmodellen i [Apache James](https://james.apache.org/), som totemomail bygger på. Den konkrete rutingskonfigurasjonen avhenger av det enkelte miljøet, men de beskrevne processorene, matcherene, mailetene og repositoriene utgjør det tekniske grunnlaget for enhver installasjon.

En viktig sikkerhetsregel gjelder uavhengig av detaljene: Hvis totemomail er gatewayen foran, må Exchange Online bare motta internett-e-post fra denne gatewayen. Dette krever en restriktiv partnerkobling. En MX-oppføring alene blokkerer ikke den direkte leveringsveien. Artikkelen [En MX-record er ikke en brannmur](/blog/ghost-sender-exchange-online-nebeneingang) viser hvordan denne sideinngangen oppstår og hvordan den lukkes.

## Fra SMTP-mottak til behandling

Behandlingslogikken i Apache James består av fire byggesteiner:

- **Matchere** kontrollerer betingelser og bestemmer hvilke mottakere en regel gjelder for.
- **Maileter** utfører selve handlingen, for eksempel å endre headere, kryptere, levere eller avslutte videre behandling.
- **Processorer** samler matchere og maileter i ordnede behandlingstrinn.
- **E-postrepositorier** lagrer meldinger under behandling eller etter en feil.

Dette skillet er avgjørende for analysen: Repositoriet svarer på spørsmålet om **hvor** en melding befinner seg. Processoren bestemmer **hvordan** den behandles videre.

![Bruk av James som SMTP-relé](../images/4CixEi383SY5WdvwMSGZ67odMU.png)

SMTP-serveren godtar forbindelsen og leser meldingen til slutten av `DATA`-delen. Deretter oppretter James et `MailImpl`-objekt. Det inneholder MIME-innholdet som `MimeMessage`, samt informasjonen som kreves for behandlingen: avsender, mottaker, status og andre attributter.

I et filbasert repositorium lagrer James denne informasjonen separat:

- `FileStreamStore` inneholder hele RFC-822-/MIME-meldingen som en byte-strøm.
- `FileObjectStore` inneholder det serialiserte `MailImpl`-objektet med status og metadata.

En melding kan derfor allerede være fullstendig mottatt og lagret, selv om den faglige behandlingen fortsatt gjenstår.

## Repositorier og køer under `/var/mail`

De enkelte repositoriene vises som kataloger i filsystemet. Ved normal drift blir en melding bare der i svært kort tid. Hvis en kø hoper seg opp, tyder det vanligvis på en feil regel, et mål som ikke er tilgjengelig, eller en backend-tjeneste som er nede.

Eksempelet nedenfor inneholder, i tillegg til standardkøene, også valgfrie kataloger for en HIN-tilkobling. HIN tilbyr et sikkert kommunikasjonsområde for det sveitsiske helsevesenet.

> Hvis du trenger hjelp med tilkobling av HIN-Mailgateway eller migrering til den nye HIN-Stargate-løsningen, finner du de riktige ekspertene hos [adeptio](https://adeptio.ch/).  
>   
> **adeptio** er offisiell partner for [Health Info Net AG](https://www.hin.ch/de/index.cfm) og har dermed også direkte kontaktpersoner hos produsenten.  
> [➜ Bestill en time i dag.](/kontakt)

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

## Slik ligger en melding i filsystemet

Hver lagrede melding består av to filer.

### `FileStreamStore`: Innholdet i meldingen

Filen `*.FileStreamStore` inneholder hele RFC-822-/MIME-meldingen. Med `cat` kan headere og brødtekst leses:

```text
From:
To:
Subject:
...
Body
```

Det underliggende meldingsformatet er beskrevet i [RFC 822](https://datatracker.ietf.org/doc/html/rfc822).

### `FileObjectStore`: Status og metadata

Filen `*.FileObjectStore` er et serialisert Java-objekt av typen `org.apache.james.core.MailImpl`. Blant feltene er:

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

[API-dokumentasjonen for `MailImpl`](https://james.apache.org/server/3/apidocs/org/apache/james/core/MailImpl.html) beskriver objektmodellen i detalj.

## Statusen velger neste processor

Katalogstrukturen viser bare repositoriet. Den faktiske behandlingsstatusen står i feltet `state` i `FileObjectStore`. Verdien peker på `name`-attributtet til en processor.

Etter hver mailet kontrollerer SpoolManager denne statusen:

1. Hvis statusen forblir uendret, følger neste matcher-mailet-par i samme processor.
2. Hvis en mailet endrer statusen, avslutter James gjeldende processor og hopper til processoren med samme navn.
3. Den spesielle statusen `ghost` avslutter behandlingen fullstendig.

De obligatoriske processorene `root` og `error` har faste oppgaver. Nye meldinger starter i `root`; interne feil og tilsvarende konfigurerte maileter videresender til `error`. Rekkefølgen på `<processor>`-elementene i XML-filen bestemmer derimot **ikke** utførelsesrekkefølgen.

## Processor-struktur i `totemomail_config.xml`

Før enhver endring bør den gjeldende `totemomail_config.xml` eksporteres og sikres:

![Configuration / Open Current / Export to File](../images/kWKIN3vramf0IAuPjzioEGV4Znw.png)

De ulike processorene og mailetene de inneholder, vises i totemomail\_config.xml. Her er igjen et eksempel fra praksis:

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

Selv om `root` står sist i dette utdraget, starter hver nye melding der. Navnet er avgjørende, ikke plasseringen i dokumentet.

Selve `root`-processoren inneholder en ordnet liste med matcher-mailet-par:

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

XML-filen konfigurerer klassene, men implementerer dem ikke. `SimpleLogger` er for eksempel en klasse levert av totemomail eller Kiteworks, der kildekoden ikke er åpen i appliance-løsningen. Hjelpen i administratorgrensesnittet forklarer imidlertid parameterne:

- `log-message` angir loggteksten og er obligatorisk.
- `showSenderEmailAddress` legger eventuelt til avsenderadressen.
- `showRecipientsEmailAddress` legger til mottakeradressene.
- `showSubject` legger til emnet.

Rekkefølgen **innenfor** en processor er bindende. En matcher kan velge ingen, alle eller bare en del av mottakerne. Ved en delmengde deler James meldingen: De aktuelle mottakerne går gjennom maileten, mens de øvrige behandles videre separat. Hvis en mailet deretter endrer statusen, hopper behandlingen umiddelbart til den angitte processoren; de resterende reglene i gjeldende processor blir hoppet over.

Dette gir en pålitelig fremgangsmåte for feilsøking:

1. Bestem repositoriet og tilhørende `FileStreamStore`-/`FileObjectStore`-filer.
2. Finn gjeldende `state` i `FileObjectStore`.
3. Finn processoren med samme navn i `totemomail_config.xml`.
4. Kontroller matchere og maileter i faktisk rekkefølge.
5. Fortsett i mål-processoren ved et statusskifte.

Slik kan en e-postflyt følges trinn for trinn, uten feilaktig å lese XML-filen ovenfra og ned som et lineært program.

## Kilder

1.  [Apache James – prosjektside](https://james.apache.org/): Open-source-MTA som totemomail eller Kiteworks EPG bygger teknisk på.
    
2.  [Apache James – «Spool Manager»](https://james.apache.org/server/head/spoolmanager.html): Behandling av innkommende e-post, spool og køer.
    
3.  [Apache James – «Spool Manager Configuration»](https://james.apache.org/server/head/spoolmanager_configuration.html): Processor-konfigurasjon og rekkefølge på reglene.
    
4.  [Apache James – «Mailet API»](https://james.apache.org/server/head/mailet_api.html): Mailet- og matcher-konseptet bak reglene.
    
5.  [Apache James – «MailImpl» (API-dokumentasjon)](https://james.apache.org/server/3/apidocs/org/apache/james/core/MailImpl.html): E-postobjektmodellen bak FileStreamStore og FileObjectStore.
    
6.  [IETF – RFC 822](https://datatracker.ietf.org/doc/html/rfc822): Formatet for tekstmeldinger på internett (headere og brødtekst).
    
7.  [Microsoft Learn – «Connectors for secure mail flow with a partner»](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-for-secure-mail-flow-with-a-partner): Koblingskonfigurasjon for sikker e-postflyt mellom Exchange Online og gatewayen.
