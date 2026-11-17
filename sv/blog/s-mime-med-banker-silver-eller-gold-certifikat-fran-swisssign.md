---
title: "S/MIME för banker: rättsläget i Schweiz och valet mellan Silver och Gold"
navTitle: "S/MIME Silver/Gold"
description: "Vilka rättsliga krav gäller när en schweizisk bank signerar och krypterar e-post till kunder: banksekretess, FINMA-cirkulär 2023/1, DSG, ZertES, bevisrätt och arkivering. Dessutom vilken certifikatnivå från SwissSign som följer av detta."
date: "2026-11-16"
kategorie: "E-postkryptering"
timeToRead: "11 min läsning"
themen:
  - e-mail-verschluesselung
produkte: []
protokolle:
  - "verschluesselung"
  - "tls"
hauptthema: "e-mail-verschluesselung"
startseiteAb: "2099-12-31"
slug: "s-mime-med-banker-silver-eller-gold-certifikat-fran-swisssign"
translationId: "article-5c2fd8d42f088dcb"
translationOf: smime-silver-gold-schweiz
translationSourceHash: 60de430124ebe20a3001bd48fc55affcb921141039e7c85303da9c7216b01e89
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T15:22:26.251Z
translationReview: required
url: https://rafaelpfister.ch/sv/blog/s-mime-med-banker-silver-eller-gold-certifikat-fran-swisssign
---

# S/MIME för banker: rättsläget i Schweiz och valet mellan Silver och Gold

Kort slutsats: Ingen schweizisk författning kräver att en bank använder S/MIME eller en viss certifikatnivå. Banksekretessen, FINMA-cirkulär 2023/1 och dataskyddslagen kräver dock att kunduppgifter skyddas på ett lämpligt sätt vid överföring, och banken måste kunna motivera och dokumentera sin lösning. En S/MIME-signatur ersätter inte en underskrift, inte ens med ett Gold-certifikat. Som bevismedel och som belägg för ursprung är Gold klart överlägset Silver-certifikatet, eftersom endast Gold innehåller bankens verifierade namn.

Frågan om vilket SwissSign-certifikat en bank behöver för e-postutskick uppkommer regelbundet i mina projekt med finansinstitut. Bakom den tekniska frågan ligger oftast en rättslig fråga: Vad måste banken göra för att uppfylla sina skyldigheter vid överföring av kunduppgifter per e-post, och vilken verkan har ett signerat e-postmeddelande?

**Fokus på Schweiz:** Den här artikeln behandlar rättsläget i Schweiz. Där EU-reglerna skiljer sig åt finns en markerad anmärkning. Artikeln ersätter inte juridisk rådgivning i enskilda fall.

## Översikt över rättsgrunderna

| Regelverk | Innehåll | Betydelse för e-postöverföring |
|---|---|---|
| BankG art. 47 | Banksekretess, straffsanktionerad även vid oaktsamhet | Kunduppgifter får inte röjas obehörigen |
| FINMA-RS 2023/1, randnummer 73–82 | Skydd av kritiska data under hela livscykeln | Skydd även vid överföring, teknikneutralt |
| DSG art. 8 och 24, DSV art. 1–6 | Datasäkerhet, anmälan av överträdelser | Lämpliga åtgärder, anmälan till EDÖB vid hög risk |
| ZertES och OR art. 14.2bis | Typer av elektroniska signaturer, formgiltighet | S/MIME-signaturer ersätter inte en underskrift |
| ZPO art. 157 och 177 | Bevismedel och fri bevisvärdering | Signerade e-postmeddelanden är handlingar med fritt värderat bevisvärde |
| OR art. 958f, GeBüV | Arkivering av affärshandlingar | Krypterade e-postmeddelanden måste förbli läsbara |

Ingen av dessa författningar nämner S/MIME eller en certifikatnivå. De fastställer skyddsmål och rättsföljder; hur banken uppnår dem avgör den själv.

## Banksekretess

Art. 47 BankG straffar den som i egenskap av organ, anställd eller uppdragstagare åt en bank röjer en hemlighet som anförtrotts personen i denna egenskap. Även oaktsamt handlande är straffbart. Bestämmelsen riktar sig till fysiska personer, men bygger på avtalsförhållandet mellan kunden som hemlighetens innehavare och banken.

För e-postöverföring innebär detta: Ett okrypterat e-postmeddelande med kunduppgifter som skickas till fel adress eller läses av under transporten kan utgöra ett oaktsamt röjande. Banken måste därför genom organisatoriska och tekniska åtgärder säkerställa att medarbetare endast skickar kunduppgifter på ett skyddat sätt.

Kunden kan som hemlighetens innehavare samtycka till röjandet. Många banker låter därför kunden i de allmänna villkoren eller i ett separat avtal bekräfta att kunden känner till och önskar riskerna med okrypterad kommunikation per e-post. Ett sådant samtycke omfattar endast kommunikationen med denna kund och endast i den avtalade omfattningen. För handlingar som rör tredje man eller för kommunikation med affärspartner ersätter det inte kryptering.

## FINMA-cirkulär 2023/1: kritiska data

FINMA-cirkulär 2023/1 «Operationella risker och resiliens: banker» har varit i kraft sedan den 1 januari 2024 och ersatte cirkulär 2008/21. FINMA har utvidgat det tidigare fokuset på konfidentialiteten hos kundidentifieringsdata (CID) till kritiska data i allmänhet. Varje bank fastställer själv, baserat på risk, vad som är kritiskt; kunduppgifter hör i regel dit med avseende på konfidentialitet.

För e-postöverföring är framför allt följande randnummer relevanta:

| Rn | Innehåll | Koppling till e-postöverföring |
|---|---|---|
| 73 | Identifiera kritiska data och kategorisera dem efter kritikalitet | Fastställa vilka data som endast får skickas skyddat |
| 74, 75 | Säkerställa konfidentialitet, integritet och tillgänglighet under hela livscykeln | Överföringen är en del av livscykeln |
| 76 | Skydda kritiska data mot åtkomst från obehöriga | Skydd av e-postmeddelandet under transport och hos mottagaren |
| 79 | Begränsa ökade risker vid lagring i eller åtkomst från utlandet | Berör gateways och e-postplattformar med plats utomlands |
| 81 | Omedelbart anmäla väsentliga incidenter till FINMA | Exempelvis massutskick av kunduppgifter till fel mottagare |
| 82 | Omsorgsfullt välja ut och övervaka tjänsteleverantörer som behandlar kritiska data | Berör leverantörer av krypteringsgateways och certifieringsorgan med hanterad PKI |

Cirkuläret är teknikneutralt och nämner varken S/MIME eller certifikatnivåer. I sina förklaringar till samrådet nämner FINMA datakryptering som ett exempel på de särskilda åtgärder som skyddar kritiska data. Banken måste fastställa sin lösning i egna riktlinjer och där även ange vilken certifikatnivå den använder och varför.

## Dataskydd: DSG och DSV

Art. 8 DSG förpliktar personuppgiftsansvariga att säkerställa datasäkerhet som är lämplig i förhållande till risken. Dataskyddsförordningen konkretiserar detta med skyddsmål som konfidentialitet, integritet och spårbarhet, men föreskriver inget visst förfarande. S/MIME, en gateway med leverans via en webbportal eller en utbytesplattform uppfyller kravet i lika hög grad om de används på ett sätt som är lämpligt i förhållande till risken.

Om en överträdelse av datasäkerheten sannolikt medför en hög risk för de berörda personernas personlighet eller grundläggande rättigheter, måste banken enligt art. 24 DSG så snart som möjligt anmäla den till EDÖB. Denna anmälningsskyldighet gäller utöver skyldigheten att anmäla till FINMA. Om e-postmeddelandet var krypterat är risken vid felaktig sändning i regel låg och någon anmälan behövs inte.

## Vad en S/MIME-signatur har för rättsverkan

### Ingen underskrift i OR:s mening

ZertES skiljer mellan elektronisk, avancerad, reglerad och kvalificerad elektronisk signatur samt, för företag, reglerat elektroniskt sigill. Enligt art. 14.2bis OR är endast den kvalificerade elektroniska signaturen med kvalificerad tidsstämpel likställd med en egenhändig underskrift.

SwissSigns S/MIME-certifikat utfärdas enligt reglerna från CA/Browser Forum och ETSI-policyerna LCP (Silver) respektive NCP (Gold). De är inte kvalificerade certifikat enligt ZertES. Banken kan därför inte avge förklaringar som enligt lag eller avtal kräver skriftlig form med formgiltig verkan genom S/MIME-signerad e-post, inte ens med Gold.

### Personlig signatur och gateway-signatur

En avancerad signatur förutsätter bland annat att signeringsnyckeln står under den undertecknande personens exklusiva kontroll. Om en gateway centralt signerar alla utgående e-postmeddelanden med nycklar som finns på gatewayen, är detta krav inte uppfyllt. Signaturen visar då att e-postmeddelandet kommer från bankens system och inte har ändrats, men inte att en viss person har godkänt det. Innehållsmässigt motsvarar den därför snarare organisationens sigill, utan att uppfylla kraven för ett reglerat sigill enligt ZertES.

### Bevisvärde

Ett e-postmeddelande är enligt art. 177 ZPO en handling, och domstolen värderar dess beviskraft fritt enligt art. 157 ZPO. En giltig S/MIME-signatur visar att e-postmeddelandet inte har ändrats sedan signeringen och att det har signerats med nyckeln till ett visst certifikat. Hur mycket detta säger om avsändaren beror på vad certifieringsorganet har kontrollerat före utfärdandet. Här ligger den rättsligt relevanta skillnaden mellan Silver och Gold.

## Silver eller Gold ur rättslig synvinkel

Sedan den 1 september 2023 gäller `S/MIME Baseline Requirements` från CA/Browser Forum för offentligt betrodda S/MIME-certifikat. SwissSign tilldelar sina produkter dessa valideringstyper i certifikatprofilerna (CPR S/MIME):

| Produkt | Typ enligt CA/B Forum | Vad SwissSign kontrollerar | Vad som står i certifikatet | Vad signaturen intygar |
|---|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Kontroll över e-postlådan | Endast e-postadressen | Någon med åtkomst till denna e-postlåda |
| E-Mail ID Gold | `Sponsor-validated` | Organisation och person | Namn, organisation, organisationsidentifierare, land, e-postadress | En verifierad person vid denna bank |
| Organization Validated | `Organization-validated` | Organisation | Organisationens namn, ingen person | En e-postlåda hos denna bank |

Kryptografiskt är nivåerna likvärdiga. För krypteringen och därmed för skyddsskyldigheterna enligt banksekretessen, FINMA-cirkuläret och DSG spelar nivån därför ingen roll.

För signaturen är skillnaden betydande. Ett Silver-certifikat kan fås av vem som helst som kan ta emot e-post till en adress. Den som registrerar en snarlikt klingande domän, till exempel `musterbank-ch.com` i stället för `musterbank.ch`, kan därmed skicka e-postmeddelanden som är lika giltigt signerade som bankens äkta e-postmeddelanden. Vid en tvist om huruvida ett e-postmeddelande kommer från banken bevisar en Silver-signatur därför föga. Ett Gold- eller Organization Validated-certifikat innehåller däremot bankens namn som kontrollerats av SwissSign. Med Gold tillkommer kopplingen till en person vars identitet banken bekräftar som registrerande instans inom ramen för den hanterade PKI:n.

## Arkivering: OR och GeBüV

Enligt art. 958f OR ska affärsböcker och verifikationer bevaras i tio år, och förordningen om affärsböcker (GeBüV) kräver att elektroniskt arkiverade handlingar kan göras läsbara under denna period. För banker tillkommer tillsynsrättsliga och avtalsmässiga bevarandeskyldigheter. Detta leder till två krav för S/MIME-driften:

- Krypterade e-postmeddelanden måste kunna dekrypteras under hela bevarandetiden. Banken måste därför förvara privata nycklar till utgångna certifikat säkert eller dekryptera e-postmeddelandena före arkivering.
- Om en signatur ska tjäna som bevis även efter flera år måste det vara möjligt att följa att certifikatet var giltigt vid tidpunkten för signeringen. Därför måste kontrollresultaten eller spärrinformationen från signeringstidpunkten också arkiveras.

**EU-anmärkning:** I EU är enligt art. 25.2 eIDAS endast den kvalificerade elektroniska signaturen likställd med en handskriven underskrift. För finansföretag i EU gäller dessutom sedan den 17 januari 2025 DORA-förordningen med egna krav på IKT-riskhantering.

## Rekommendation

- **Fastställ riktlinjer:** Banken fastställer i en intern riktlinje vilka data som endast får skickas krypterade, vilka metoder som är tillåtna för detta och vilka certifikatnivåer den använder. Detta är grunden för att kunna styrka kraven i FINMA-cirkuläret och DSG.
- **Gold för personliga e-postlådor, Organization Validated för funktionsbrevlådor:** Endast dessa nivåer anger banken som avsändare i certifikatet och har därmed ett bevisvärde som går utöver åtkomst till en e-postlåda. Silver lämpar sig inte för kommunikation med kunder.
- **Klargör samtycken:** Den som kommunicerar okrypterat med kunder via e-post behöver deras dokumenterade samtycke. Det ersätter inte kryptering gentemot tredje man och affärspartner.
- **Inte formbundna förklaringar via e-post:** För förklaringar som kräver skriftlig form behövs en kvalificerad elektronisk signatur eller en egenhändig underskrift.
- **Kontrollera tjänsteleverantörer:** Gateway-leverantörer och certifieringsorganets hanterade PKI omfattas av randnummer 82 i FINMA-cirkuläret, och vid platser utomlands dessutom av randnummer 79.
- **Arkivera nycklar och verifieringsdata:** Privata nycklar för dekryptering och data för signaturverifiering måste förbli tillgängliga under bevarandetiden.
- **Koppla samman anmälningsprocesser:** En felaktig sändning av kunduppgifter kan samtidigt utlösa en anmälan till FINMA (randnummer 81) och till EDÖB (art. 24 DSG). Båda skyldigheterna kan hanteras i en gemensam incidentprocess.

## Källor

1.  [FINMA-cirkulär 2023/1 «Operationella risker och resiliens: banker»](https://www.finma.ch/de/~/media/finma/dokumente/dokumentencenter/myfinma/rundschreiben/finma-rs-2023-01-20221207.pdf): randnummer 73–82 om kritiska data, i kraft sedan den 1 januari 2024.

2.  [datenrecht.ch: FINMA-RS 2023/1 med förklaringar](https://datenrecht.ch/wp-content/uploads/FINMA-RS-OpRisk-%E2%80%93-Text-revidiert-1.1.2024.pdf): reviderad text med FINMA:s uttalanden från samrådet, bland annat om datakryptering som skyddsåtgärd.

3.  [Banklagen (BankG)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): art. 47 om banksekretess.

4.  [Swissbanking: Banksekretess och molnet, utlåtande 2019](https://www.swissbanking.ch/_Resources/Persistent/b/1/3/3/b13339fe5dcd344e715dbe0a899afe3c8c56ed0b/SBVg_Bankgeheimnis_und_Cloud_Gutachten_LLAG_2019_DE.pdf): juridiskt utlåtande om tolkningen av art. 47 BankG, bland annat om kundens samtycke.

5.  [Federala lagen om dataskydd (DSG)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 8 om datasäkerhet, art. 24 om anmälan av överträdelser av datasäkerheten.

6.  [Dataskyddsförordningen (DSV)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): art. 1–6 med principer och skyddsmål för datasäkerhet.

7.  [Federala lagen om elektronisk signatur (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): typer av elektroniska signaturer och sigill, krav på kvalificerade certifikat.

8.  [Obligationsrätten (OR)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): art. 14.2bis om likställande av den kvalificerade elektroniska signaturen med den egenhändiga underskriften, art. 958f om arkivering.

9.  [Civilprocesslagen (ZPO)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): art. 157 om fri bevisvärdering, art. 177 om handlingsbegreppet.

10.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): certifikatprofiler från SwissSign (version 18.09.2025), tilldelning av Silver, Gold och Organization Validated till valideringstyperna och Subject-fälten.

11.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): bindande regler för offentligt betrodda S/MIME-certifikat, definition av valideringstyperna.
