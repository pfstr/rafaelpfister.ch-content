---
title: "S/MIME hos banker: rättsläget i Schweiz och valet mellan Silver och Gold"
navTitle: "S/MIME Silver/Gold"
description: "Vilka rättsliga krav gäller när en schweizisk bank signerar och krypterar e-post till kunder: banksekretess, FINMA-cirkulär 2023/1, DSG, ZertES, bevisrätt och arkivering. Dessutom vilken certifikatnivå från SwissSign som följer av detta."
date: "2026-11-16"
kategorie: "E-postkryptering"
timeToRead: "12 min läsning"
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
translationSourceHash: f4b52127b6883bb31850668cc18a27ebba43f75200bfdc605d199f427d602ddf
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T03:27:49.043Z
translationReview: required
url: https://rafaelpfister.ch/sv/blog/s-mime-med-banker-silver-eller-gold-certifikat-fran-swisssign
---

# S/MIME hos banker: rättsläget i Schweiz och valet mellan Silver och Gold

Kort slutsats: Ingen schweizisk författning kräver att en bank använder S/MIME eller en viss certifikatnivå. Banksekretessen, FINMA-cirkulär 2023/1 och dataskyddslagen kräver dock att kunduppgifter skyddas på ett lämpligt sätt vid överföring, och banken måste kunna motivera och dokumentera sin lösning. En S/MIME-signatur ersätter inte en underskrift, inte ens med ett Gold-certifikat. Som bevismedel och som intyg på ursprung är Gold klart överlägset Silver-certifikatet, eftersom endast Gold innehåller bankens verifierade namn.

Frågan om vilket SwissSign-certifikat en bank behöver för e-postutskick kommer regelbundet upp i mina projekt med finansinstitut. Bakom den tekniska frågan finns oftast en rättslig fråga: Vad måste banken göra för att uppfylla sina skyldigheter vid överföring av kunduppgifter via e-post, och vilken verkan har ett signerat e-postmeddelande?

**Fokus på Schweiz:** Denna artikel behandlar rättsläget i Schweiz. Där EU-reglerna avviker anges detta särskilt. Artikeln ersätter inte juridisk rådgivning i enskilda fall.

## Rättslig grund i korthet

| Regelverk | Innehåll | Betydelse för e-postutskick |
|---|---|---|
| BankG art. 47 | Banksekretess, straffsanktionerad även vid oaktsamhet | Kunduppgifter får inte röjas obehörigen |
| FINMA-RS 2023/1, randnummer 73 till 82 | Skydd av kritiska data under hela livscykeln | Skydd även vid överföring, teknikneutralt |
| DSG art. 8 och 24, DSV art. 1 till 6 | Datasäkerhet, anmälan av överträdelser | Lämpliga åtgärder, anmälan till EDÖB vid hög risk |
| ZertES och OR art. 14 stycke 2bis | Typer av elektroniska signaturer, formgiltighet | S/MIME-signaturer ersätter inte en underskrift |
| ZPO art. 157 och 177 | Bevismedel och fri bevisvärdering | Signerade e-postmeddelanden är urkunder med bevisvärde som bedöms fritt |
| OR art. 958f, GeBüV | Arkivering av affärshandlingar | Krypterade e-postmeddelanden måste förbli läsbara |

Ingen av dessa författningar nämner S/MIME eller en certifikatnivå. De fastställer skyddsmål och rättsverkningar; hur banken uppnår dem avgör den själv.

## Banksekretess

Art. 47 BankG straffar den som i egenskap av organ, anställd eller uppdragstagare för en bank röjer en hemlighet som anförtrotts personen i denna egenskap. Även oaktsamt handlande är straffbart. Bestämmelsen riktar sig till fysiska personer, men den bygger på avtalsförhållandet mellan kunden som hemlighetens innehavare och banken.

För e-postutskick innebär detta: Ett okrypterat e-postmeddelande med kunduppgifter som skickas till fel adress eller läses under överföringen kan utgöra ett oaktsamt röjande. Banken måste därför organisatoriskt och tekniskt säkerställa att medarbetare endast skickar kunduppgifter skyddat.

Kunden kan som hemlighetens innehavare samtycka till röjandet. Därför låter många banker bekräfta i allmänna villkor eller i ett separat avtal att kunden känner till och önskar riskerna med okrypterad kommunikation via e-post. Ett sådant samtycke omfattar endast kommunikationen med denna kund och endast i den avtalade omfattningen. För handlingar som berör tredje man eller för kommunikation med affärspartners ersätter det inte kryptering.

## FINMA-cirkulär 2023/1: kritiska data

FINMA-cirkulär 2023/1 «Operationella risker och resiliens: banker» har varit i kraft sedan den 1 januari 2024 och ersatte cirkulär 2008/21. FINMA har utvidgat det tidigare fokuset på sekretessen för kundidentifieringsuppgifter (CID) till kritiska data generellt. Varje bank fastställer själv vad som är kritiskt utifrån risk; kunduppgifter hör i regel dit vad gäller sekretess.

För e-postutskick är särskilt följande randnummer relevanta:

| Rn | Innehåll | Koppling till e-postutskick |
|---|---|---|
| 73 | Identifiera kritiska data och kategorisera dem efter kritikalitet | Fastställa vilka data som endast får skickas skyddat |
| 74, 75 | Säkerställa sekretess, integritet och tillgänglighet under hela livscykeln | Överföringen är en del av livscykeln |
| 76 | Skydda kritiska data mot åtkomst från obehöriga | Skydd av e-post under överföring och hos mottagaren |
| 79 | Begränsa ökade risker vid lagring i eller åtkomst från utlandet | Berör gateways och e-postplattformar med placering utomlands |
| 81 | Omedelbart anmäla väsentliga incidenter till FINMA | Exempelvis massutskick av kunduppgifter till fel mottagare |
| 82 | Noggrant välja ut och övervaka tjänsteleverantörer som behandlar kritiska data | Berör leverantörer av krypteringsgateways och certifieringsinstanser med Managed PKI |

Cirkuläret är teknikneutralt och nämner varken S/MIME eller certifikatnivåer. I sina kommentarer till remissen nämner FINMA datakryptering som ett exempel på särskilda åtgärder för att säkra kritiska data. Banken måste fastställa sin lösning i interna riktlinjer och där även ange vilken certifikatnivå den använder och varför.

## Dataskydd: DSG och DSV

Art. 8 DSG ålägger personuppgiftsansvariga att säkerställa datasäkerhet som är lämplig med hänsyn till risken. Dataskyddsförordningen konkretiserar detta med skyddsmål som sekretess, integritet och spårbarhet, men föreskriver inget särskilt förfarande. S/MIME, en gateway med leverans via webbportal eller en utbytesplattform uppfyller kravet i lika grad om de används på ett sätt som är lämpligt med hänsyn till risken.

Om en överträdelse av datasäkerheten sannolikt leder till hög risk för de berörda personernas personlighet eller grundläggande rättigheter måste banken enligt art. 24 DSG anmäla den till EDÖB så snart som möjligt. Denna anmälningsskyldighet gäller utöver skyldigheten att anmäla till FINMA. Om e-postmeddelandet var krypterat är risken vid felutskick normalt låg, och någon anmälan behövs inte.

## Vad en S/MIME-signatur innebär rättsligt

### Ingen underskrift enligt OR

ZertES skiljer mellan elektronisk, avancerad, reglerad och kvalificerad elektronisk signatur samt, för företag, det reglerade elektroniska sigillet. Enligt art. 14 stycke 2bis OR är endast den kvalificerade elektroniska signaturen med kvalificerad tidsstämpel likställd med en egenhändig underskrift.

SwissSigns S/MIME-certifikat utfärdas enligt reglerna från CA/Browser Forum och ETSI-policyerna LCP (Silver) respektive NCP (Gold). De är inte kvalificerade certifikat enligt ZertES. Förklaringar som enligt lag eller avtal kräver skriftlig form kan en bank därför inte lämna formgiltigt via S/MIME-signerad e-post, inte ens med Gold.

### Personlig signatur och gateway-signatur

En avancerad signatur förutsätter bland annat att signaturnyckeln står under den undertecknande personens exklusiva kontroll. Om en gateway centralt signerar alla utgående e-postmeddelanden med nycklar som finns på gatewayen är detta villkor inte uppfyllt. Signaturen visar då att e-postmeddelandet kommer från bankens system och inte har ändrats, men inte att en viss person har godkänt det. Innehållsmässigt motsvarar den därför snarare organisationens sigill, utan att uppfylla kraven på ett reglerat sigill enligt ZertES.

### Bevisvärde

Ett e-postmeddelande är en urkund enligt art. 177 ZPO, och domstolen bedömer dess bevisvärde fritt enligt art. 157 ZPO. En giltig S/MIME-signatur visar att e-postmeddelandet inte har ändrats sedan signeringen och att det är signerat med nyckeln till ett visst certifikat. Hur mycket detta säger om avsändaren beror på vad certifieringsinstansen kontrollerade före utfärdandet. Här ligger den rättsligt relevanta skillnaden mellan Silver och Gold.

## Silver eller Gold ur rättslig synvinkel

Sedan den 1 september 2023 gäller `S/MIME Baseline Requirements` från CA/Browser Forum för offentligt betrodda S/MIME-certifikat. SwissSign placerar sina produkter i certifikatprofilerna (CPR S/MIME) inom dessa valideringstyper:

| Produkt | Typ enligt CA/B Forum | Vad SwissSign kontrollerar | Vad som står i certifikatet | Vad signaturen säger |
|---|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Kontroll över e-postlådan | Endast e-postadressen | Någon med åtkomst till denna e-postlåda |
| E-Mail ID Gold | `Sponsor-validated` | Organisation och person | Namn, organisation, organisationsidentifierare, land, e-postadress | En verifierad person hos denna bank |
| Organization Validated | `Organization-validated` | Organisation | Organisationens namn, ingen person | En e-postlåda hos denna bank |

Kryptografiskt är nivåerna likvärdiga. För krypteringen och därmed för skyddsskyldigheterna enligt banksekretessen, FINMA-cirkuläret och DSG spelar nivån därför ingen roll.

För signaturen är skillnaden betydande. Ett Silver-certifikat får alla som kan ta emot e-post till en adress. Den som registrerar en snarlik domän, till exempel `musterbank-ch.com` i stället för `musterbank.ch`, kan därmed skicka e-postmeddelanden som är lika giltigt signerade som bankens äkta e-postmeddelanden. Vid en tvist om huruvida ett e-postmeddelande kommer från banken bevisar en Silver-signatur därför lite. Ett Gold- eller Organization-Validated-certifikat innehåller däremot bankens namn, verifierat av SwissSign. Med Gold tillkommer kopplingen till en person vars identitet banken bekräftar som registreringsinstans inom ramen för Managed PKI.

## Arkivering: OR och GeBüV

Enligt art. 958f OR ska affärsböcker och bokföringsunderlag bevaras i tio år, och förordningen om affärsböcker (GeBüV) kräver att elektroniskt arkiverade handlingar kan göras läsbara under denna tid. För banker tillkommer tillsynsrättsliga och avtalsenliga arkiveringsskyldigheter. Detta medför två krav för S/MIME-driften:

- Krypterade e-postmeddelanden måste kunna dekrypteras under hela arkiveringstiden. Banken måste därför lagra de privata nycklarna för utgångna certifikat säkert eller dekryptera e-postmeddelandena före arkivering.
- Om en signatur även efter flera år ska tjäna som bevis måste det gå att fastställa att certifikatet var giltigt vid signeringstillfället. Därför ska kontrollresultaten eller spärrinformationen från signeringstillfället också arkiveras.

**EU-anmärkning:** Även inom EU är enligt art. 25.2 eIDAS endast den kvalificerade elektroniska signaturen likställd med en handskriven underskrift. För finansiella företag i EU gäller sedan den 17 januari 2025 dessutom DORA-förordningen med egna krav på IKT-riskhantering.

## Exempel: En gateway för alla medarbetare

Många banker använder en central krypteringsgateway, till exempel PrivaSphere, SEPPmail eller Totemomail, så att alla medarbetare kan kryptera och signera e-post utan att själva hantera certifikat. Nycklarna finns på gatewayen, och gatewayen signerar och krypterar på uppdrag av de enskilda e-postlådorna. För denna uppbyggnad följer följande val av föregående avsnitt:

| E-postlåda | Nivå | Skäl |
|---|---|---|
| Personliga e-postlådor | E-Mail ID Gold | Visar banken som avsändare med verifierat namn och organisationsidentifierare |
| Funktionsbrevlådor (kundtjänst, betalningshantering) | Organization Validated | Visar banken utan att ange en enskild person |
| Ingen | E-Mail ID Silver | Styrker endast åtkomst till en e-postlåda; e-post från snarlika domäner är lika giltigt signerad |

Eftersom gatewayen signerar och inte den enskilda personen står nycklarna inte under medarbetarnas exklusiva kontroll. Signaturen fungerar rättsligt som bankens sigill och inte som en personlig signatur. Det avgörande är därför att organisationen anges som verifierad i certifikatet, och det gör endast Gold och Organization Validated. För själva krypteringen saknar nivån betydelse.

Före införandet ska följande punkter klargöras med gatewayleverantören och certifieringsinstansen:

- **Utfärdande och förnyelse:** Om gatewayen automatiskt hämtar och förnyar Gold-certifikat via certifieringsinstansens Managed PKI. För Gold bekräftar banken som registreringsinstans varje persons identitet; detta kräver en process för anställningar och avgångar, namnändringar och spärrningar.
- **Granskning av tjänsteleverantörer:** Gatewayleverantörer och Managed PKI behandlar kritiska data enligt randnummer 82 i FINMA-cirkulär 2023/1. Var nycklar och data finns ska dokumenteras; vid lagring utomlands eller åtkomst därifrån gäller dessutom randnummer 79.
- **Arkivering:** De privata nycklarna för utgångna certifikat och kontrolldata från signeringstidpunkten måste finnas tillgängliga under hela arkiveringstiden.
- **Riktlinje:** De valda nivåerna och godkända förfarandena hör hemma i den interna riktlinjen, som fungerar som underlag gentemot FINMA och revisionen.

## Rekommendation

- **Fastställ riktlinjer:** Banken fastställer i en intern riktlinje vilka data som endast får skickas krypterat, vilka förfaranden som är tillåtna för detta och vilka certifikatnivåer den använder. Detta är grunden för att visa att kraven från FINMA-cirkuläret och DSG uppfylls.
- **Gold för personliga e-postlådor, Organization Validated för funktionsbrevlådor:** Endast dessa nivåer visar banken som avsändare i certifikatet och har därmed ett bevisvärde som går utöver åtkomst till en e-postlåda. Silver är inte lämpligt för kommunikation med kunder.
- **Klargör samtycken:** Den som kommunicerar okrypterat med kunder via e-post behöver deras dokumenterade samtycke. Det ersätter inte kryptering gentemot tredje man och affärspartners.
- **Skicka inte formbundna förklaringar via e-post:** För förklaringar som kräver skriftlig form behövs en kvalificerad elektronisk signatur eller en egenhändig underskrift.
- **Granska tjänsteleverantörer:** Gatewayleverantörer och certifieringsinstansens Managed PKI omfattas av randnummer 82 i FINMA-cirkuläret och, vid placering utomlands, även av randnummer 79.
- **Arkivera nycklar och kontrolldata:** Privata nycklar för dekryptering och data för signaturkontroll måste finnas tillgängliga under arkiveringstiden.
- **Koppla samman anmälningsprocesser:** Ett felutskick av kunduppgifter kan samtidigt utlösa en anmälan till FINMA (randnummer 81) och till EDÖB (art. 24 DSG). Båda skyldigheterna kan hanteras i en gemensam incidentprocess.

## Källor

1.  [FINMA-cirkulär 2023/1 «Operationella risker och resiliens: banker»](https://www.finma.ch/de/~/media/finma/dokumente/dokumentencenter/myfinma/rundschreiben/finma-rs-2023-01-20221207.pdf): randnummer 73 till 82 om kritiska data, i kraft sedan den 1 januari 2024.

2.  [datenrecht.ch: FINMA-RS 2023/1 med kommentarer](https://datenrecht.ch/wp-content/uploads/FINMA-RS-OpRisk-%E2%80%93-Text-revidiert-1.1.2024.pdf): reviderad text med FINMA:s uttalanden från remissen, bland annat om datakryptering som skyddsåtgärd.

3.  [Banklagen (BankG)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): art. 47 om banksekretess.

4.  [Swissbanking: Banksekretess och molnet, rättsutlåtande 2019](https://www.swissbanking.ch/_Resources/Persistent/b/1/3/3/b13339fe5dcd344e715dbe0a899afe3c8c56ed0b/SBVg_Bankgeheimnis_und_Cloud_Gutachten_LLAG_2019_DE.pdf): rättsutlåtande om tolkningen av art. 47 BankG, bland annat om kundens samtycke.

5.  [Federala lagen om dataskydd (DSG)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 8 om datasäkerhet, art. 24 om anmälan av överträdelser av datasäkerheten.

6.  [Dataskyddsförordningen (DSV)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): art. 1 till 6 med principer och skyddsmål för datasäkerhet.

7.  [Federala lagen om elektroniska signaturer (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): typer av elektroniska signaturer och sigill, krav på kvalificerade certifikat.

8.  [Obligationsrätten (OR)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): art. 14 stycke 2bis om likställandet av kvalificerad elektronisk signatur med egenhändig underskrift, art. 958f om arkivering.

9.  [Civilprocesslagen (ZPO)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): art. 157 om fri bevisvärdering, art. 177 om urkundsbegreppet.

10.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): SwissSigns certifikatprofiler (per 18.09.2025), klassificering av Silver, Gold och Organization Validated efter valideringstyp och Subject-fält.

11.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): bindande regler för offentligt betrodda S/MIME-certifikat, definition av valideringstyperna.
