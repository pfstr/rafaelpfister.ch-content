---
title: "S/MIME för banker: Silver- eller Gold-certifikat från SwissSign?"
navTitle: "S/MIME Silver/Gold"
description: "En bank som signerar och krypterar kundmejl med S/MIME behöver Gold-certifikat för personliga postlådor och Organization-Validated-certifikat för funktionspostlådor. Silver krypterar lika starkt, men visar inte banken som avsändare. Vad SwissSign kontrollerar, vad som står i certifikatet och vilka regler som gäller i Schweiz."
date: "2026-11-16"
kategorie: "E-postkryptering"
timeToRead: "9 min lästid"
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
translationSourceHash: f95529dd7c72d4d783714affbcf18d96e37adf66a52379b94212034dda610b74
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T14:49:30.507Z
translationReview: automatic
url: https://rafaelpfister.ch/sv/blog/s-mime-med-banker-silver-eller-gold-certifikat-fran-swisssign
---

# S/MIME för banker: Silver- eller Gold-certifikat från SwissSign?

Kort slutsats: En bank som signerar och krypterar mejl till kunder och affärspartner med S/MIME bör utrusta personliga postlådor med Gold-certifikat och funktionspostlådor med Organization-Validated-certifikat från SwissSign. Silver krypterar lika starkt, men certifikatet innehåller endast e-postadressen och inte bankens namn. Varken lagen eller FINMA ställer krav på certifikatnivå.

Frågan om vilken certifikatnivå som krävs för e-postutskick uppkommer regelbundet i mina projekt med finansinstitut. Oftast aktualiseras den i början av ett gateway-projekt eller när företagskunder kräver att konfidentiella handlingar endast skickas krypterat. För att besvara frågan måste två funktioner hos S/MIME hållas isär: krypteringen, som skyddar innehållet, och signaturen, som visar mottagaren vem som har skickat mejlet.

**Fokus Schweiz:** Den här artikeln behandlar rättsläget i Schweiz. Där EU-reglerna skiljer sig åt finns en markerad anmärkning.

## Vad SwissSign kontrollerar för Silver och Gold

Sedan den 1 september 2023 gäller `S/MIME Baseline Requirements` från CA/Browser Forum för offentligt betrodda S/MIME-certifikat. De skiljer mellan fyra valideringstyper beroende på vad certifikatutfärdaren kontrollerar före utfärdandet. SwissSign tilldelar sina produkter dessa typer i de publicerade certifikatprofilerna (CPR S/MIME):

| Produkt | Typ enligt CA/B Forum | Vad som kontrolleras | Vad som står i certifikatet |
|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Kontroll över postlådan | endast e-postadressen |
| E-Mail ID Gold | `Sponsor-validated` | Organisation och person | För- och efternamn (eller pseudonym), organisation, organisationsidentifierare, land, e-postadress |
| Organization Validated | `Organization-validated` | Organisation | Organisationsnamn, ingen person |

För Silver skickar SwissSign en bekräftelsekod till den ansökta adressen. Den som kan ta emot mejlet får certifikatet. Personens identitet och organisationen kontrolleras inte, och adressen står även i certifikatet som `Common Name`.

Gold är hos SwissSign uteslutande klassificerat som `Sponsor-validated`. SwissSign kontrollerar organisationen, och personen är knuten till den. Personens identitet kontrolleras antingen av SwissSign eller av organisationen själv som registreringsinstans, vanligtvis inom ramen för en Managed PKI. För funktionspostlådor som `kundendienst@` eller `zahlungsverkehr@` är profilen `Organization-validated` avsedd; där står banken som organisation i certifikatet, men ingen person.

Kryptografiskt skiljer sig nivåerna inte åt. Alla profiler använder RSA-nycklar från 2048 bitar, nyckelanvändningarna `digitalSignature` och `keyEncipherment` samt den utökade användningen `id-kp-emailProtection`.

## Vad skillnaden innebär för en bank som avsändare

| | Silver | Gold | Organization Validated |
|---|---|---|---|
| Kryptera och dekryptera mejl | ja | ja | ja |
| Signaturen visar i e-postprogrammet | E-postadress | Namn och bank | Bank |
| Vad signaturen intygar | Avsändaren har åtkomst till postlådan | Kontrollerad person vid denna bank | Denna banks postlåda |
| Användning hos en bank | olämpligt för kundkommunikation | personliga postlådor | funktionspostlådor |

Skillnaden gäller signaturen. En bank signerar utgående mejl så att mottagarna kan kontrollera att meddelandet verkligen kommer från banken och inte har ändrats på vägen. Med Silver är detta bara möjligt i begränsad utsträckning:

- Den som registrerar en snarlik domän, till exempel `musterbank-ch.com` i stället för `musterbank.ch`, kan få ett Silver-certifikat för adresser på den domänen utan ytterligare kontroll. Personens nätfiskemejl är då lika giltigt signerat som bankens riktiga mejl, och certifikatet innehåller i båda fallen endast en e-postadress.
- Den som tar över en av bankens postlådor kan också ansöka om ett Silver-certifikat för den adressen.

Ett Gold- eller Organization-Validated-certifikat innehåller bankens kontrollerade namn i fältet `Organization`. Endast den som klarar SwissSigns organisationskontroll får denna uppgift. För mottagare som granskar signaturen är detta det relevanta beviset.

För krypteringen spelar nivån ingen roll. Ett mejl som krypterats med Silver är lika väl skyddat som ett som krypterats med Gold.

## Kryptering förutsätter mottagarnas certifikat

För att kryptera ett mejl med S/MIME behöver banken mottagarens publika nyckel. Bankens eget certifikat används inte för detta; det används för signaturen och för att ta emot krypterade svar.

De flesta privatkunder har inget S/MIME-certifikat. Banker levererar därför vanligtvis konfidentiella mejl till dem via en krypteringsgateway, till exempel SEPPmail (GINA) eller Kiteworks/Totemomail, som tillhandahåller meddelandet i en webbportal eller som en krypterad bilaga. S/MIME-kryptering används främst för företagskunder och partner som själva använder certifikat eller en gateway. Om båda parter använder en gateway räcker ofta domäncertifikat eller tvingad TLS mellan e-postservrarna.

## Certifikat i drift: Managed PKI och gateway

För fler än en handfull postlådor anskaffar en bank sina certifikat via SwissSigns Managed PKI. Banken fungerar då som registreringsinstans och bekräftar själv de anställdas identitet; detta motsvarar typen `Sponsor-validated`. Gateways som SEPPmail och Kiteworks/Totemomail kan via ett gränssnitt till Managed PKI automatiskt ansöka om certifikat, förnya dem och tilldela dem till postlådor.

Tre punkter från projekt är kända för att orsaka problem i driften:

- Gold-ansökningar för funktionspostlådor misslyckas eftersom gatewayen hämtar för- och efternamn från katalogen och SwissSign avvisar generiska beteckningar som «Support» eller «Helpdesk» i fältet `GivenName`. För sådana postlådor kan profilen `Organization-validated` konfigureras i gatewayen.
- Automatiska förnyelser kan misslyckas i veckor utan att det märks, tills de första certifikaten löper ut och kunder inte längre kan skicka krypterade mejl. Utgångsdatumen för utfärdade certifikat och felmeddelandena från förnyelsekörningarna bör därför övervakas.
- Gatewayen signerar sina ansökningar till Managed PKI med ett eget certifikat för registreringsinstansen. Om detta löper ut upphör hela automatiken att fungera.

## Rättsliga krav i Schweiz

Ingen schweizisk lag föreskriver S/MIME, och ingen skiljer mellan certifikatnivåer. En banks skyldigheter gäller skyddet av kunduppgifter; hur banken genomför detta avgör den själv.

| Regelverk | Innehåll | Betydelse för bankens e-postutskick |
|---|---|---|
| BankG art. 47 | Banksekretess | Kunduppgifter får inte röjas obehörigen; okrypterad överföring över internet är en risk som banken måste bedöma |
| FINMA-RS 2023/1 | Skydd av kritiska data | Skydd även vid överföring, teknikneutralt, inga krav på certifikat |
| DSG art. 8 och DSV art. 1 till 6 | Datasäkerhet som är lämplig i förhållande till risken | Kryptering är ett vanligt medel, men en viss metod är inte föreskriven |
| ZertES och OR art. 14 stycke 2bis | endast den kvalificerade elektroniska signaturen är likställd med egenhändig underskrift | S/MIME-signaturer ersätter inte en underskrift |

### Banksekretess och FINMA

Banksekretessen enligt art. 47 BankG förpliktar banken, dess organ och anställda att inte obehörigen röja kunduppgifter. FINMA-cirkulär 2023/1 «Operativa risker och resiliens: Banker» kräver att banker skyddar sina kritiska data under hela livscykeln, även vid överföring. Hur detta görs fastställer banken i sina egna riktlinjer. Valet av certifikatnivå är därmed en del av den interna policyn och bör dokumenteras där.

### Dataskydd: DSG och DSV

Art. 8 DSG förpliktar personuppgiftsansvariga att säkerställa datasäkerhet som är lämplig i förhållande till risken. Dataskyddsförordningen konkretiserar detta med skyddsmål som konfidentialitet och integritet, men anger inga metoder. S/MIME, en gateway med webbmejlleverans eller en utbytesplattform uppfyller detta krav på samma sätt, förutsatt att de används på ett sätt som är lämpligt i förhållande till risken.

### Elektronisk signatur: ZertES och OR

ZertES skiljer mellan elektronisk, avancerad, reglerad och kvalificerad elektronisk signatur. Enligt art. 14 stycke 2bis OR är endast den kvalificerade elektroniska signaturen med kvalificerad tidsstämpel likställd med egenhändig underskrift. SwissSigns S/MIME-certifikat utfärdas enligt reglerna från CA/Browser Forum och ETSI-policyerna LCP (Silver) respektive NCP (Gold) och är inte kvalificerade certifikat enligt ZertES. Avtal eller förklaringar som kräver skriftlig form kan inte lämnas in formellt giltigt med ett S/MIME-signerat mejl, inte heller med Gold.

**EU-anmärkning:** Även inom EU är enligt art. 25 stycke 2 eIDAS endast den kvalificerade elektroniska signaturen likställd med en handskriven underskrift.

## Rekommendation

- **Personliga postlådor** för kundrådgivning och andra medarbetare med kundkontakt: Gold (`Sponsor-validated`) via Managed PKI.
- **Funktionspostlådor:** `Organization-validated`, så att banken anges som avsändare i certifikatet.
- **Silver** bör inte användas för kundkommunikation eftersom signaturen inte visar banken som avsändare.
- **Kryptering till privatkunder** via gatewayens push- eller portalleverans; S/MIME för mottagare med eget certifikat eller gateway.
- **Signatur och avsändarverifiering** kompletterar varandra: S/MIME-signaturen ersätter inte SPF, DKIM och DMARC, men visar mottagaren den kontrollerade organisationen i e-postprogrammet.
- **Informationsblad för kunder:** Beskriv vilka certifikat banken använder, hur kunder kontrollerar en signatur och vilka certifikatnivåer banken accepterar från kunder som själva vill skicka krypterat. Om ett sådant informationsblad saknas hamnar frågan hos kundrådgivningen, som vanligtvis inte kan besvara den.

## Källor

1.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): Certifikatprofiler från SwissSign (version 18.09.2025), tilldelning av Silver, Gold och Organization Validated till valideringstyperna, Subject-fält och policy-OID:er.

2.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): bindande regler för offentligt betrodda S/MIME-certifikat, definition av de fyra valideringstyperna.

3.  [SEPPmail: Managed PKI SwissSign](https://docs.seppmail.com/en/07_mi_10_mpki_11_swisssign.html): anslutning av SEPPmail-gatewayen till SwissSigns Managed PKI.

4.  [Bankengesetz (BankG)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): art. 47 om banksekretess.

5.  [FINMA: Rundschreiben](https://www.finma.ch/de/dokumentation/rundschreiben/): översikt med cirkulär 2023/1 «Operativa risker och resiliens: Banker», i kraft sedan 1 januari 2024.

6.  [Bundesgesetz über den Datenschutz (DSG)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 8 om datasäkerhet.

7.  [Datenschutzverordnung (DSV)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): art. 1 till 6 med principer och skyddsmål för datasäkerhet.

8.  [Bundesgesetz über die elektronische Signatur (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): typer av elektronisk signatur och krav för kvalificerade certifikat.

9.  [Obligationenrecht (OR)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): art. 14 stycke 2bis om likställandet av den kvalificerade elektroniska signaturen med egenhändig underskrift.
