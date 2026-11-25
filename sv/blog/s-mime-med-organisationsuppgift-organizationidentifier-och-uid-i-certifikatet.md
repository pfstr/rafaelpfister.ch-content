---
title: "S/MIME med organisationsuppgift: organizationIdentifier och UID i certifikatet"
navTitle: "S/MIME med organisation"
description: "Vad ett S/MIME-certifikat med organisationsuppgift innehåller, hur UID införs som organizationIdentifier i certifikatet och vad certifikatutfärdaren kontrollerar för företag och person. Dessutom en jämförelse mellan SwissSign och DigiCert, kommandon för att kontrollera certifikattypen och avgränsningen mot ZertES."
date: "2026-12-15"
kategorie: "E-postkryptering"
timeToRead: "13 min lästid"
themen:
  - e-mail-verschluesselung
produkte: []
protokolle:
  - "verschluesselung"
  - "powershell"
hauptthema: "e-mail-verschluesselung"
startseiteAb: "2026-12-15"
slug: "s-mime-med-organisationsuppgift-organizationidentifier-och-uid-i-certifikatet"
translationId: "article-e0f9bcc57f360835"
translationOf: smime-organisation-uid
translationSourceHash: 849361658c86c3e7bae12ee2d7e018cf43807a611de4911b9ad1bdbe870ea0fe
translationModel: gpt-5.6-terra
translatedAt: 2026-11-24T14:58:27.333Z
translationReview: automatic
url: https://rafaelpfister.ch/sv/blog/s-mime-med-organisationsuppgift-organizationidentifier-och-uid-i-certifikatet
---

# S/MIME med organisationsuppgift: organizationIdentifier och UID i certifikatet

Certifikatutfärdare (CA) skiljer mellan S/MIME-certifikat med och utan organisationsuppgift. Ett certifikat med organisationsuppgift visar, utöver e-postadressen, även företaget som postlådan tillhör: med dess verifierade namn och ett registernummer. För schweiziska företag är detta registernummer UID, alltså CHE-numret från handelsregistret.

Kort slutsats: En organisationsuppgift finns endast för certifikattyperna `Organization-validated` och `Sponsor-validated` i S/MIME Baseline Requirements. Den består av företagsnamnet i fältet `O` och fältet `organizationIdentifier` (OID 2.5.4.97), för ett schweiziskt företag till exempel `NTRCH-CHE-123.456.789`. CA:n kontrollerar företaget i ett officiellt register innan den skriver in dessa uppgifter i certifikatet. Ett mailbox-validerat certifikat styrker däremot endast åtkomst till en postlåda. Hos SwissSign motsvarar det valet mellan Silver och Gold, hos DigiCert valet mellan «Secure Email for Individual Mailbox» och produkterna «for Employee» och «for Organization». En kvalificerad signatur enligt ZertES uppstår inte genom organisationsuppgiften.

**Schweiz-fokus:** Den rättsliga bedömningen följer schweizisk rätt (ZertES, OR). Där EU-reglerna avviker finns en markerad anmärkning.

## De fyra certifikattyperna

Sedan den 1 september 2023 gäller `S/MIME Baseline Requirements` (BR) från CA/Browser Forum för offentligt betrodda S/MIME-certifikat. De skiljer mellan fyra certifikattyper utifrån vem som står som innehavare (`Subject`) i certifikatet:

| Typ | Innehavare i certifikatet | Organisationsuppgift | Vad CA:n kontrollerar |
|---|---|---|---|
| `Mailbox-validated` (MV) | endast e-postadressen | inte tillåten | Kontroll över postlådan eller e-postdomänen |
| `Organization-validated` (OV) | en organisation, ingen person | obligatorisk | Organisation samt postlåda eller e-postdomän |
| `Sponsor-validated` (SV) | en person som tillhör en organisation | obligatorisk | som OV, plus personen; denna kontrollerar CA:n eller organisationen själv |
| `Individual-validated` (IV) | en person utan organisation | inte tillåten | Personens identitet och postlådan |

Med «S/MIME med organisationsuppgift» avses typerna OV och SV. För båda föreskriver BR fälten `organizationName` och `organizationIdentifier` obligatoriskt, i samtliga tre generationer `Legacy`, `Multipurpose` och `Strict`. Generationerna skiljer sig åt vad gäller tillåtna Subject-fält, tillägg och användningsändamål. BR avvecklade generationen `Legacy` den 15 juli 2025; SwissSign och DigiCert utfärdar den inte längre.

OV-certifikat lämpar sig för funktionspostlådor som `info@` eller `buchhaltung@`, som inte är kopplade till någon person. SV-certifikat är avsedda för medarbetares personliga postlådor och innehåller dessutom personens namn eller en pseudonym.

## organizationIdentifier:s uppbyggnad

Fältet `organizationIdentifier` (OID 2.5.4.97) innehåller ett registernummer för organisationen. Samma fält används även av certifikat enligt eIDAS och ZertES. För S/MIME-certifikat föreskriver BR följande uppbyggnad:

| Del | Exempel | Betydelse |
|---|---|---|
| Registreringsschema, 3 tecken | `NTR` | `National Trade Register`, handelsregistret |
| Landskod enligt ISO 3166-1 | `CH` | Landet där registret förs |
| Bindestreck | `-` | Avgränsare |
| Registernummer | `CHE-123.456.789` | Organisationens nummer i registret |

Sammansatt blir det `NTRCH-CHE-123.456.789`. Eftersom UID självt innehåller bindestreck fastställer BR att schema och landskod aldrig innehåller bindestreck. Det första bindestrecket från vänster skiljer alltså schemat från registernumret, medan alla följande tillhör numret. Om ett register endast tilldelar nummer på delstats- eller provinsnivå följer ett `+` med regionkoden efter landskoden, exempelvis `NTRUS+CA-12345678` för Kalifornien.

BR känner till följande registreringsscheman:

| Schema | Numrets ursprung | Exempel från BR |
|---|---|---|
| `NTR` | Handels- eller företagsregister | `NTRGB-12345678` |
| `VAT` | Skattemyndighet (momsregistreringsnummer) | `VATEL-123456789` |
| `PSD` | Auktorisationsnummer för betaltjänstleverantörer enligt EU-direktiv 2015/2366 | `PSDBE-NBB-1234.567.890` |
| `LEI` | `Legal Entity Identifier` enligt ISO 17442, landskoden alltid `XG` | `LEIXG-` följt av LEI |
| `GOV` | Myndigheter utan nummer från något av de övriga schemana | `GOVUS` |
| `INT` | internationella organisationer | `INTXG` |

För schweiziska företag är `NTR` med UID normalfallet. UID tilldelas av Federal Statistical Office. Handelsregistren använder den som företagsnummer, den fungerar även som momsregistreringsnummer och förblir densamma om ett företag flyttar sitt säte till en annan kanton. Ett momsregistreringsnummer kan däremot upphöra när skattskyldigheten upphör, och inte varje organisation har ett sådant. SwissSign övertar UID med punkterna, precis som den står i UID-registret, och anger `NTRCH-CHE-109.357.012` som exempel för SwissSign AG.

**EU-anmärkning:** I Tyskland tilldelar lokala domstolar handelsregisternummer, och samma nummer förekommer hos flera domstolar. Sedan version 1.0.9 från maj 2025 tillåter BR därför `NTR`-schemat att använda `European Unique Identifier` (EUID) och rekommenderar det för tyska företag, till exempel `NTRDE-DER3306.HRB12345` för ett företag i handelsregistret vid Amtsgericht Köln. Enligt BR måste registernumret endast vara unikt där register och rättsordning tilldelar unika nummer.

## Vad CA:n kontrollerar

Innan en CA utfärdar ett OV- eller SV-certifikat måste den enligt BR kontrollera följande uppgifter:

- **Organisationens namn och existens:** via en myndighet i organisationens land, LEI-data, ett platsbesök eller en bekräftelse med registerutdrag. För ett schweiziskt företag är det handelsregistret och UID-registret. Om företaget där anges som avregistrerat eller inaktivt får CA:n inte utfärda certifikatet.
- **Registernummer:** CA:n bekräftar att numret i `organizationIdentifier` tillhör organisationen i fältet `O`.
- **Fullmakt:** Via en tillförlitlig kommunikationskanal kontrollerar CA:n att den ansökande personen får ansöka om certifikat för organisationen eller agera som `Enterprise RA`.
- **Postlåda eller e-postdomän:** CA:n kontrollerar kontrollen över e-postadresserna i certifikatet. Om organisationen agerar som `Enterprise RA` måste den styrka kontrollen över hela e-postdomänen.
- **Person (endast SV):** Personens namn eller pseudonym. Nästa avsnitt beskriver vilka bevis som är tillåtna för detta.

En CA får återanvända resultaten av dessa kontroller: kontrollen av organisationen och fullmakten under 825 dagar, och kontrollen av domänkontrollen under 398 dagar. Ett certifikat i generationerna `Multipurpose` och `Strict` är giltigt i högst 825 dagar.

## Hur personen kontrolleras vid SV

BR tillåter följande bevis för personens identitet:

| Bevis | Förlopp |
|---|---|
| officiell identitetshandling | Original på plats eller i realtid framför en kamera; den kontrollerande personen jämför fotot på handlingen med personen och dokumenterar utfärdare, giltighet och handlingsnummer |
| chip i pass eller identitetskort | Kontroll av utfärdarens signatur på chipdata samt fotojämförelse |
| eID | endast anmälda eID-system enligt eIDAS med tillitsnivån «väsentlig» eller «hög» |
| mobilt körkort (mDL) | Förfrågan i realtid enligt ISO/IEC 18013-5 och 18013-7 med kontroll av utfärdarens signaturkedja |
| digital signatur | Ansökan signeras med ett befintligt personligt certifikat från ett erkänt ramverk; listan över dessa ramverk är fortfarande tom i BR |
| bekräftelse från notarie eller advokatbyrå | skriftlig bekräftelse från den ansökande personens land |
| handlingar från `Enterprise RA` | Företaget kontrollerar sina medarbetare självt och får stödja sig på befintliga interna uppgifter, exempelvis från personaladministrationen |

Vilka av dessa bevis en CA accepterar måste den offentliggöra i sin CPS. Kontrollen av personen får vid utfärdandet vara högst 825 dagar gammal. Om företaget självt kontrollerar som `Enterprise RA` måste CA:n avtalsmässigt förplikta det och kontrollera det minst en gång om året, såvida företaget inte självt revideras årligen. Kopplingen till personen är i detta fall lika tillförlitlig som företagets interna process.

## Jämförelse mellan SwissSign och DigiCert

Båda CA:erna erbjuder typerna MV, SV och OV under egna produktnamn. Ingen av dem erbjuder ett IV-certifikat. BR fastställer uppbyggnaden av `organizationIdentifier`, och den är densamma för båda leverantörerna.

| | SwissSign | DigiCert |
|---|---|---|
| MV | Personal S/MIME E-Mail ID Silver | Secure Email for Individual Mailbox |
| SV | Pro S/MIME E-Mail ID Gold | Secure Email for Employee (tidigare Secure Email for Business) |
| OV | Organization Validated E-Mail Certificate | Secure Email for Organization |
| IV | erbjuds inte | erbjuds inte |
| Anskaffning | Silver i webbshoppen eller via Managed PKI, Gold och OV endast via företagets Managed PKI | alla tre via företagets CertCentral-konto |
| Kontroll av personen vid SV | officiell identitetshandling i original, på plats eller genom ett likvärdigt förfarande; andra bevis endast med godkännande från SwissSign | bekräftelse från företaget; DigiCert kontrollerar inte personen |
| Kontrollernas giltighet | uppgifter och bevis högst 398 dagar | organisation 825 dagar, domän 398 dagar |
| Generation för SV och OV | `Multipurpose`; en mellanliggande CA för SV i generationen `Strict` beskrivs redan i CP/CPS | `Strict` som standard, `Multipurpose` valbar |
| Person vid SV | för- och efternamn eller pseudonym med prefixet `pseudo:` i `CN` | för- och efternamn eller pseudonym; i `CN` valfritt namnet eller e-postadressen |
| Ytterligare användningar vid SV | inga i standardprofilen; i profilen med autentisering klientautentisering, smartkortsinloggning och EFS | vid `Strict` oavvislighet, vid `Multipurpose` dessutom datakryptering och klientautentisering |

DigiCert beskriver uttryckligen SV-produkten enligt modellen `Enterprise RA`: Organisationen bekräftar att personen i certifikatet tillhör dess medarbetare eller företrädare och bevarar underlagen för detta. DigiCert självt kontrollerar endast organisationen. Den som får beställa i företagets CertCentral-konto kan därmed ange vilket namn som helst för en adress i de egna domänerna. För OV och SV kräver DigiCert alltid bevis på domänkontroll. Organisationskontrollen gäller i 825 dagar, domänkontrollen i 398 dagar. Giltighetstid på ett eller två år eller ett eget utgångsdatum på högst 824 dagar kan väljas. DigiCert avvecklade de tidigare produkterna Premium, Email Security Plus, Digital Signature Plus och Class 1 S/MIME den 10 juli 2025 och ersatte dem med dessa produkter.

SwissSign utfärdar Gold- och OV-certifikat endast via företagets Managed PKI. Personen måste ändå identifiera sig med en officiell identitetshandling i original, personligen eller genom ett likvärdigt förfarande enligt ETSI EN 319 411-1. Identifieringen utförs av den registrerande instans som handlägger ansökan eller av en utbildad identifieringspartner med avtal: kopiera, skanna eller fotografera identitetshandlingen, jämföra fotot med personen och bekräfta processen för SwissSign. Användningen av företagsnamnet måste godkännas av den företagsledning eller styrelse som är registrerad i handelsregistret, och de personer som ansöker om certifikat i Managed PKI identifieras också med identitetshandling. Ett annat förfarande, exempelvis baserat på personaluppgifter, får ett företag endast använda om SwissSign har granskat och godkänt det som likvärdigt. Uppgifter och bevis gäller hos SwissSign i högst 398 dagar, därefter kontrollerar SwissSign på nytt.

Hos SwissSign innehåller OV-certifikatet i `CN` organisationens namn såsom i fältet `O`. Vilken nivå en bank bör välja för vilken postlåda och vad detta innebär juridiskt beskrivs i artikeln [S/MIME hos banker: rättsläget i Schweiz och valet mellan Silver och Gold](/blog/smime-silver-gold-schweiz).

## Kontrollera certifikattypen

I Outlook och andra e-postprogram kan certifikatinformationen för ett signerat e-postmeddelande öppnas via signaturikonen. I Windows finns fälten på fliken «Detaljer» under posten «Ämne». För OID 2.5.4.97 känner Windows inte till något visningsnamn och visar den som `OID.2.5.4.97`.

### Visa innehavaren med PowerShell

För ett exporterat certifikat (till exempel från certifikatinformationen i ett signerat e-postmeddelande):

```powershell
$zert = Get-PfxCertificate -FilePath .\zertifikat.cer
$zert.SubjectName.Format($true)
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Funktion |
|---|---|
| `Get-PfxCertificate` | läser ett certifikat från en fil, även från en `.cer`-fil utan privat nyckel |
| `-FilePath .\zertifikat.cer` | sökväg till certifikatfilen |
| `SubjectName.Format($true)` | visar innehavarens fält rad för rad |

</details>

Utdata för ett SV-certifikat (exempel):

```text
CN=Max Muster
G=Max
SN=Muster
O=Muster AG
OID.2.5.4.97=NTRCH-CHE-123.456.789
S=ZH
C=CH
```

För ett MV-certifikat innehåller utdata endast `CN=` med e-postadressen.

### Bestäm typ och generation via policy-OID

Varje certifikat enligt BR innehåller i tillägget `Certificate Policies` en OID enligt mönstret `2.23.140.1.5.<Typ>.<Generation>`:

| Position | Värde | Betydelse |
|---|---|---|
| Typ | `1` | `Mailbox-validated` |
| Typ | `2` | `Organization-validated` |
| Typ | `3` | `Sponsor-validated` |
| Typ | `4` | `Individual-validated` |
| Generation | `1` | `Legacy` |
| Generation | `2` | `Multipurpose` |
| Generation | `3` | `Strict` |

```powershell
$zert.Extensions |
  Where-Object { $_.Oid.Value -eq '2.5.29.32' } |
  ForEach-Object { $_.Format($true) }
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Funktion |
|---|---|
| `$zert.Extensions` | tillägg till det tidigare inlästa certifikatet |
| `'2.5.29.32'` | OID för tillägget `Certificate Policies` |
| `Format($true)` | visar tilläggets innehåll på flera rader |

</details>

Utdata (förkortad):

```text
[1]Certificate Policy:
     Policy Identifier=2.23.140.1.5.3.2
```

`2.23.140.1.5.3.2` står för ett SV-certifikat av generationen `Multipurpose`. Ytterligare poster med CA:ns egna OID:er är normala.

### Lista S/MIME-certifikat i den egna certifikatlagringen

```powershell
Get-ChildItem Cert:\CurrentUser\My |
  Where-Object { $_.EnhancedKeyUsageList.ObjectId -contains '1.3.6.1.5.5.7.3.4' } |
  Select-Object Subject, NotAfter
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Funktion |
|---|---|
| `Cert:\CurrentUser\My` | personlig certifikatlagring för den inloggade användaren |
| `EnhancedKeyUsageList.ObjectId` | certifikatets utökade nyckelanvändningar |
| `'1.3.6.1.5.5.7.3.4'` | OID för e-postskydd (`id-kp-emailProtection`) |
| `Select-Object Subject, NotAfter` | visar innehavare och utgångsdatum |

</details>

### Med OpenSSL

```bash
openssl x509 -in zertifikat.pem -noout -subject -nameopt multiline
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Funktion |
|---|---|
| `x509` | underkommando för certifikat |
| `-in zertifikat.pem` | indatafil i PEM-format; för en DER-fil dessutom `-inform der` |
| `-noout` | undertrycker utmatningen av det kodade certifikatet |
| `-subject` | visar innehavaren |
| `-nameopt multiline` | ett fält per rad med utskrivna fältnamn |

</details>

OpenSSL 3 känner till fältnamnet och visar det utskrivet:

```text
subject=
    commonName                = Max Muster
    givenName                 = Max
    surname                   = Muster
    organizationName          = Muster AG
    organizationIdentifier    = NTRCH-CHE-123.456.789
    stateOrProvinceName       = ZH
    countryName               = CH
```

### Kontrollera registernumret

Som mottagare av ett signerat e-postmeddelande kan du kontrollera organisationsuppgiften i fyra steg:

1. E-postprogrammet visar signaturen som giltig och certifikatet kommer från en CA som e-postprogrammet litar på.
2. Policy-OID:n börjar med `2.23.140.1.5.2` eller `2.23.140.1.5.3`, och certifikatet innehåller fälten `O` och `organizationIdentifier`.
3. UID:n från `organizationIdentifier` tillhör det förväntade företaget i UID-registret och företaget är aktivt där.
4. Avsändaradressens domän tillhör detta företag.

## Avgränsning mot ZertES

Organisationsuppgiften gör inte ett S/MIME-certifikat till ett reglerat certifikat enligt ZertES. Reglerade certifikat utfärdas endast av leverantörer som är erkända enligt ZertES, och certifikatet måste innehålla uppgiften att det är ett reglerat certifikat (art. 2 lit. g och art. 7 stycke 2 lit. b ZertES).

Innehållsmässigt finns paralleller. Ett reglerat certifikat för en UID-enhet måste innehålla dess UID (art. 7 stycke 2 lit. e ZertES), på samma sätt som `organizationIdentifier` innehåller den för OV och SV. Ett certifikat för en fysisk person kan dessutom innehålla uppgiften att personen har rätt att företräda en viss UID-enhet (art. 7 stycke 3 lit. b ZertES); detta motsvarar i uppbyggnad ett SV-certifikat.

| | S/MIME med organisationsuppgift | reglerat certifikat enligt ZertES |
|---|---|---|
| Grund | S/MIME Baseline Requirements från CA/Browser Forum | ZertES och tillämpningsbestämmelser |
| Utfärdare | offentligt betrodd CA | leverantör erkänd enligt ZertES |
| Organisationsidentifiering | `organizationIdentifier`, för schweiziska företag med UID | UID, obligatoriskt för UID-enheter |
| Kontroll av personen | vid SV, beroende på CA, med identitetshandling eller genom organisationens bekräftelse som `Enterprise RA` | i princip personligt inställelse med identitetsbevis; för UID-enheter en företrädare med bevis på representationsrätt (art. 9 ZertES) |
| Nyckel | enligt CA:ns krav, även på en gateway | krav enligt art. 6 ZertES |

Endast den kvalificerade elektroniska signaturen med kvalificerad tidsstämpel är enligt art. 14 stycke 2bis OR likställd med egenhändig underskrift, och ett kvalificerat certifikat får endast utfärdas till en fysisk person (art. 8 stycke 1 ZertES). För en organisations signatur föreskriver ZertES det reglerade elektroniska sigillet: en avancerad elektronisk signatur som skapas med en säker enhet för sigillframställning och grundar sig på ett reglerat certifikat för en UID-enhet (art. 2 lit. d ZertES). Ett S/MIME-signerat e-postmeddelande är bevisning vars värde domstolen fritt bedömer (art. 157 ZPO); en verifierad organisationsuppgift stöder därvid kopplingen till det avsändande företaget.

**EU-anmärkning:** Inom EU reglerar eIDAS-förordningen elektroniska signaturer och sigill. Enligt art. 25 stycke 2 eIDAS är även där endast den kvalificerade elektroniska signaturen likställd med handskriven underskrift.

## Rekommendation

- **Funktionspostlådor:** OV-certifikat (SwissSign Organization Validated, DigiCert Secure Email for Organization). Det visar företaget utan att ange en person.
- **Personliga postlådor med signatur gentemot kunder och partner:** SV-certifikat (SwissSign Gold, DigiCert Secure Email for Employee). Det visar person och företag.
- **Endast kryptering:** Kryptografiskt är alla typer likvärdiga. För enbart kryptering räcker ett MV-certifikat.
- **Planera för kontroller:** Organisationskontrollen gäller i 825 dagar, domänkontrollen i 398 dagar. Efter en ändring av företagsnamn eller säte överensstämmer uppgifterna i certifikatet inte längre med registret; planera då för nya certifikat.
- **Kontrollera vid mottagande:** Policy-OID och UID visar om en signatur är kopplad till ett verifierat företag. Hur väl personen kontrollerats beror vid SV på CA:ns förfarande: hos SwissSign ligger en kontroll av identitetshandling bakom, hos DigiCert företagets bekräftelse.

## Källor

1.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): Version 1.0.17 från 20 november 2026; certifikattyper och policy-OID:er (avsnitt 1.1), `Enterprise RA` (1.3.2), kontroll av organisationen (3.2.3), kontroll av personen (3.2.4), kontroll av fullmakten (3.2.6), återanvändning av kontrollresultat (4.2.1), giltighetstider (6.3.2), Subject-fält (7.1.4.2) och registreringsscheman (bilaga A).

2.  [cabforum/smime på GitHub: SBR.md](https://github.com/cabforum/smime/blob/main/SBR.md): Markdown-version av S/MIME Baseline Requirements med versionshistorik, bland annat EUID från version 1.0.9.

3.  [SwissSign: EUID, How Your Digital Certificates Become Even More Trustworthy](https://www.swisssign.com/en/blog/european-unique-identifier-euid): Produktnamnen Silver och Gold, exempel `NTRCH-CHE-109.357.012`, UID som registernummer och EUID för tyska företag (28.04.2025).

4.  [SwissSign CP/CPS S/MIME](https://repository.swisssign.com/SwissSign_CPS_SMIME.pdf): SwissSigns kombinerade regelverk, version 1.0 från 31.08.2026, ersätter bland annat CPR S/MIME; identifiering av personen (3.2.3), fullmakt (3.2.5), bevisens giltighet (3.3.1), anskaffningsvägar (4.1.1) och certifikatprofiler för MV, SV och OV (7.1).

5.  [DigiCert: Secure Email products](https://docs.digicert.com/en/certcentral/order-and-manage-certificates/secure-email-certificates.html): Produktöversikt med koppling till MV, OV och SV, slutet för generationen `Legacy` och de tidigare S/MIME-produkterna den 10 juli 2025.

6.  [DigiCert: Order your Secure Email for Employee certificate](https://docs.digicert.com/en/certcentral/order-and-manage-certificates/secure-email-certificates/order-your-secure-email-for-employee-certificate.html): Organisations- och domänkontroll, bekräftelse av medarbetare genom organisationen utan DigiCerts egen kontroll av personen, profilerna `Strict` och `Multipurpose`, giltighetstider.

7.  [Federal Act on Electronic Signatures (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): art. 2 (begrepp, reglerat sigill), art. 6 (nycklar), art. 7 (innehåll i reglerade certifikat, UID), art. 8 (kvalificerade certifikat) och art. 9 (identitetskontroll).

8.  [Obligationsrätt (OR)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): art. 14 stycke 2bis om likställandet av den kvalificerade elektroniska signaturen med egenhändig underskrift.

9.  [UID-register](https://www.uid.admin.ch): Sökning av en UID med namn, säte och status, förs av Federal Statistical Office.

10.  [Zefix](https://www.zefix.ch): centralt företagsindex för de kantonala handelsregistren.

11.  [Civilprocesslagen (ZPO)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): art. 157 om fri bevisvärdering.

12.  [Förordning (EU) nr 910/2014 (eIDAS)](https://eur-lex.europa.eu/legal-content/DE/TXT/?uri=CELEX:32014R0910): art. 25 stycke 2 om likställandet av den kvalificerade elektroniska signaturen med handskriven underskrift.
