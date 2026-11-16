---
title: "S/MIME med banker: Silver- eller Gold-certifikat från SwissSign?"
navTitle: "S/MIME Silver/Gold"
description: "För krypterade e-postmeddelanden med en bank räcker ett Silver-certifikat från SwissSign tekniskt sett. Gold bekräftar dessutom avsändarens namn och organisation. Vilken nivå som kontrollerar vad, vad som står i certifikatet och vilka rättsliga krav som gäller i Schweiz."
date: "2026-11-16"
kategorie: "E-postkryptering"
timeToRead: "8 min. lästid"
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
url: https://rafaelpfister.ch/sv/blog/s-mime-med-banker-silver-eller-gold-certifikat-fran-swisssign
translationSourceHash: cbcb890de323f2b9db40a5682192296524250e7bc8c23a67c6f643bcba605a14
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T14:05:09.046Z
translationReview: automatic
---

# S/MIME med banker: Silver- eller Gold-certifikat från SwissSign?

Kort slutsats: Varken schweizisk lag eller FINMA föreskriver en viss certifikatnivå för S/MIME. För kryptering räcker ett Silver-certifikat fullt ut rent tekniskt. Gold bekräftar dessutom avsändarens namn och organisation och är rätt val om banken inte har några särskilda krav.

Frågan om ett Silver- eller Gold-certifikat från SwissSign behövs för e-postkommunikation med en bank återkommer regelbundet i mina projekt. Oftast finns inget informationsblad från banken, och kundrådgivaren kan inte svara på frågan. Svaret beror på vad certifikatet ska användas till: för kryptering eller som bevis på vem som har skickat ett e-postmeddelande.

**Fokus på Schweiz:** Den här artikeln behandlar rättsläget i Schweiz. Där EU-reglerna skiljer sig åt finns en markerad anmärkning.

## Vad SwissSign kontrollerar för Silver och Gold

Sedan den 1 september 2023 gäller `S/MIME Baseline Requirements` från CA/Browser Forum för offentligt betrodda S/MIME-certifikat. De skiljer mellan fyra valideringstyper, beroende på vad certifikatutfärdaren kontrollerar före utfärdandet. SwissSign tilldelar sina produkter dessa typer i de publicerade certifikatprofilerna (CPR S/MIME):

| Produkt | Typ enligt CA/B Forum | Vad som kontrolleras | Vad som står i certifikatet |
|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Kontroll över e-postlådan | endast e-postadressen |
| E-Mail ID Gold | `Sponsor-validated` | Organisation och person | För- och efternamn (eller pseudonym), organisation, organisationsidentifierare, land, e-postadress |
| Organization Validated | `Organization-validated` | Organisation | Organisationsnamn, ingen person |

För Silver skickar SwissSign en bekräftelsekod till den ansökta adressen. Den som kan ta emot e-postmeddelandet får certifikatet. Personens identitet kontrolleras inte, och adressen anges i certifikatet även som `Common Name`.

Gold anges hos SwissSign uteslutande som `Sponsor-validated`. Organisationen kontrolleras och personen kopplas till den: Identiteten kontrolleras antingen av SwissSign eller av organisationen själv som registrerande instans, exempelvis inom ramen för en Managed PKI. En profil för privatpersoner utan organisation (`Individual-validated`) anges inte i certifikatprofilerna.

För funktionsbrevlådor som `support@` eller `buchhaltung@` är profilen `Organization-validated` avsedd. Gold-ansökningar för sådana brevlådor kan misslyckas eftersom SwissSign avvisar generiska begrepp som «Support» eller «Helpdesk» i fältet `GivenName`.

Kryptografiskt skiljer sig nivåerna inte åt. Alla profiler använder RSA-nycklar från 2048 bitar, nyckelanvändningarna `digitalSignature` och `keyEncipherment` samt den utökade användningen `id-kp-emailProtection`. Ett e-postmeddelande som krypteras med Silver är lika väl skyddat som ett med Gold.

## Vad skillnaden innebär i praktiken

| | Silver | Gold |
|---|---|---|
| Kryptera och dekryptera e-postmeddelanden | ja | ja |
| Visning av signaturen i Outlook | E-postadress | Namn och organisation |
| Vad signaturen intygar | Avsändaren har åtkomst till e-postlådan | Avsändaren är en kontrollerad person i denna organisation |
| Arbetsinsats vid ansökan | Bekräftelse via e-post | Bevis på organisation och identitetskontroll |

Skillnaden blir tydlig vid ett kontokapande. Den som kontrollerar en e-postlåda kan ansöka om ett Silver-certifikat för denna adress och därmed skicka giltigt signerade e-postmeddelanden. Ett Gold-certifikat förutsätter däremot att organisationen och personen har kontrollerats. För en bank som exempelvis tar emot betalningsuppdrag eller avtalskorrespondens via e-post har en Gold-signatur därför högre bevisvärde.

För kryptering spelar detta ingen roll. Banken behöver endast mottagarens offentliga nyckel, och den är likvärdig på båda nivåerna.

## Rättsliga krav i Schweiz

Ingen schweizisk lag föreskriver S/MIME, och ingen skiljer mellan certifikatnivåer. Det finns allmänna skyldigheter att skydda data, medan den tekniska implementeringen är öppen.

| Regelverk | Innehåll | Koppling till S/MIME |
|---|---|---|
| DSG art. 8 och DSV art. 1–6 | Datasäkerhet som är anpassad till risken | Kryptering är ett vanligt medel för känsliga personuppgifter, men någon viss metod föreskrivs inte |
| BankG art. 47 | Banksekretess | förpliktar banken, inte dess kunder |
| FINMA-RS 2023/1 | Skydd av kritiska data hos banker | teknikneutral, inga krav på certifikat |
| ZertES och OR art. 14 punkt 2bis | endast den kvalificerade elektroniska signaturen är likställd med egenhändig underskrift | Silver och Gold är inte kvalificerade signaturer |
| VeÜ-ZSSV | elektroniska inlagor till domstolar och indrivningsmyndigheter | kvalificerad signatur och erkänd delgivningsplattform krävs, S/MIME räcker inte |

### Dataskydd: DSG och DSV

Art. 8 DSG förpliktar personuppgiftsansvariga och personuppgiftsbiträden att säkerställa en datasäkerhet som är anpassad till risken. Dataskyddsförordningen konkretiserar detta med skyddsmål som konfidentialitet och integritet, men nämner inga metoder. Om man använder S/MIME, en gateway med webbmailleverans eller en utbytesplattform är en bedömning i det enskilda fallet.

### Banksekretess och FINMA

Banksekretessen enligt art. 47 BankG och FINMA:s cirkulär 2023/1 «Operationella risker och resiliens: Banker» riktar sig till banken. Cirkuläret kräver att bankerna skyddar sina kritiska data under hela livscykeln, även vid överföring. Hur de genomför detta fastställer bankerna i sina egna riktlinjer. Om Silver accepteras är därför en fråga om den aktuella bankens interna policy.

### Elektronisk signatur: ZertES och OR

ZertES skiljer mellan elektronisk, avancerad, reglerad och kvalificerad elektronisk signatur. Enligt art. 14 punkt 2bis OR är endast den kvalificerade elektroniska signaturen med kvalificerad tidsstämpel likställd med egenhändig underskrift. SwissSigns S/MIME-certifikat utfärdas enligt reglerna från CA/Browser Forum och ETSI-policyerna LCP (Silver) respektive NCP (Gold) och är inte kvalificerade certifikat enligt ZertES.

Ett avtal som kräver skriftlig form kan därför inte ingås formellt giltigt med ett S/MIME-signerat e-postmeddelande, inte ens med Gold. Vid tvist kan en Gold-signatur ändå fungera som bevis; domstolen bedömer dess vikt fritt.

### Inlagor till domstolar och myndigheter

För elektroniska inlagor i civilrättsliga, straffrättsliga samt utsöknings- och konkursförfaranden kräver VeÜ-ZSSV en kvalificerad elektronisk signatur och överföring via en erkänd delgivningsplattform. Ett S/MIME-e-postmeddelande uppfyller inte dessa krav.

**EU-anmärkning:** I EU är enligt art. 25 punkt 2 eIDAS också endast den kvalificerade elektroniska signaturen likställd med handskriven underskrift.

## När banken inte har några krav

Om det saknas ett informationsblad kan situationen klarläggas med tre frågor till banken:

1.  Kräver banken ett certifikat med kontrollerad identitet för signerade e-postmeddelanden, eller räcker en ren e-postvalidering?

2.  Arbetar banken med en krypteringsgateway (t.ex. SEPPmail eller Kiteworks/Totemomail) och ett domäncertifikat, eller med personliga certifikat för sina medarbetare?

3.  Hur tillhandahåller banken sina egna offentliga nycklar?

En kort förfrågan till kundrådgivningen som kan vidarebefordras till IT-avdelningen:

> För den krypterade e-postkommunikationen med er kommer jag att skaffa ett S/MIME-certifikat. Vänligen meddela om ni kräver ett certifikat med kontrollerad identitet (t.ex. SwissSign E-Mail ID Gold med namn och organisation) eller om ett certifikat med enbart e-postvalidering (Silver) är tillräckligt. Jag ber även om ert offentliga certifikat eller information om huruvida ni använder en gateway med domäncertifikat.

Många banker skickar ändå konfidentiella e-postmeddelanden via en gateway med webbmailleverans. Mottagaren loggar då in i portalen och behöver inget eget certifikat. Om båda parter arbetar med en gateway räcker ofta domäncertifikat eller tvingad TLS mellan e-postservrarna, och personliga certifikat behövs inte.

Om frågan förblir öppen är Gold rätt val för företagskorrespondens. Gold täcker kryptering och signatur, och ett senare byte från Silver till Gold innebär en ny ansökan och en ny distribution av den offentliga nyckeln till alla motparter. Silver räcker om banken bekräftar att den endast krypterar och inte utvärderar signaturer. För funktionsbrevlådor är ett `Organization-validated`-certifikat avsett.

## Källor

1.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): SwissSigns certifikatprofiler (status 18.09.2025), tilldelning av Silver, Gold och Organization Validated till valideringstyperna, Subject-fält och policy-OID:er.

2.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): bindande regler för offentligt betrodda S/MIME-certifikat, definition av de fyra valideringstyperna.

3.  [Federala dataskyddslagen (DSG)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 8 om datasäkerhet.

4.  [Dataskyddsförordningen (DSV)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): art. 1–6 med principer och skyddsmål för datasäkerhet.

5.  [Banklagen (BankG)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): art. 47 om banksekretess.

6.  [FINMA: Cirkulär](https://www.finma.ch/de/dokumentation/rundschreiben/): översikt med cirkulär 2023/1 «Operationella risker och resiliens: Banker», i kraft sedan den 1 januari 2024.

7.  [Federala lagen om elektronisk signatur (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): typer av elektronisk signatur och krav på kvalificerade certifikat.

8.  [Obligationsrätt (OR)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): art. 14 punkt 2bis om likställandet av den kvalificerade elektroniska signaturen med egenhändig underskrift.
