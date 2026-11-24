---
title: "S/MIME med organisationsuppgift: organizationIdentifier och UID i certifikatet"
navTitle: "S/MIME med organisation"
description: "Vad ett S/MIME-certifikat med organisationsuppgift innehåller, hur UID hamnar i certifikatet som organizationIdentifier och vad certifikatutfärdaren kontrollerar för detta. Dessutom en jämförelse av produkterna från SwissSign och DigiCert, kommandon för att kontrollera certifikattypen och avgränsningen mot ZertES."
date: "2026-12-15"
kategorie: "E-postkryptering"
timeToRead: "11 min lästid"
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
url: https://rafaelpfister.ch/sv/blog/s-mime-med-organisationsuppgift-organizationidentifier-och-uid-i-certifikatet
translationSourceHash: f628a8f86635449d2a9e43f78f1a960ce5c8bc20af519eac110b77b2dad385a9
translationModel: gpt-5.6-terra
translatedAt: 2026-11-24T13:55:04.471Z
translationReview: automatic
---

# S/MIME med organisationsuppgift: organizationIdentifier och UID i certifikatet

Certifikatutfärdare (CA) skiljer mellan S/MIME-certifikat med och utan organisationsuppgift. Ett certifikat med organisationsuppgift visar, utöver e-postadressen, även företaget som postlådan tillhör: med dess kontrollerade namn och med ett registreringsnummer. För schweiziska företag är detta registreringsnummer UID, alltså CHE-numret från handelsregistret.

Kort slutsats: En organisationsuppgift finns endast för certifikattyperna `Organization-validated` och `Sponsor-validated` i S/MIME Baseline Requirements. Den består av företagsnamnet i fältet `O` och fältet `organizationIdentifier` (OID 2.5.4.97), exempelvis `NTRCH-CHE-123.456.789` för ett schweiziskt företag. CA:n kontrollerar företaget i ett officiellt register innan den skriver in dessa uppgifter i certifikatet. Ett mailbox-validerat certifikat styrker däremot endast åtkomsten till en postlåda. Hos SwissSign motsvarar detta valet mellan Silver och Gold, hos DigiCert valet mellan «Secure Email for Individual Mailbox» och produkterna «for Employee» och «for Organization». En kvalificerad signatur enligt ZertES uppstår inte genom organisationsuppgiften.

**Schweiziskt fokus:** Den rättsliga bedömningen följer schweizisk rätt (ZertES, OR). Där EU-reglerna skiljer sig åt finns en markerad hänvisning.

## De fyra certifikattyperna

Sedan den 1 september 2023 gäller `S/MIME Baseline Requirements` (BR) från CA/Browser Forum för offentligt betrodda S/MIME-certifikat. De skiljer mellan fyra certifikattyper beroende på vem som anges som innehavare (`Subject`) i certifikatet:

| Typ | Innehavare i certifikatet | Organisationsuppgift | Vad CA:n kontrollerar |
|---|---|---|---|
| `Mailbox-validated` (MV) | endast e-postadressen | inte tillåten | Kontroll över postlådan eller e-postdomänen |
| `Organization-validated` (OV) | en organisation, ingen person | obligatorisk | Organisationen samt postlådan eller e-postdomänen |
| `Sponsor-validated` (SV) | en person som tillhör en organisation | obligatorisk | som OV, dessutom personen; denna kontrollerar CA:n eller organisationen själv |
| `Individual-validated` (IV) | en person utan organisation | inte tillåten | Personens identitet och postlådan |

Med «S/MIME med organisationsuppgift» avses typerna OV och SV. För båda föreskriver BR obligatoriskt fälten `organizationName` och `organizationIdentifier`, i samtliga tre generationer `Legacy`, `Multipurpose` och `Strict`. Generationerna skiljer sig åt beträffande tillåtna Subject-fält, tillägg och användningsområden. BR avvecklade generationen `Legacy` den 15 juli 2025; SwissSign och DigiCert utfärdar den inte längre.

OV-certifikat lämpar sig för funktionspostlådor som `info@` eller `buchhaltung@`, som inte är knutna till någon person. SV-certifikat är avsedda för medarbetares personliga postlådor och innehåller dessutom personens namn eller en pseudonym.

## organizationIdentifier:s uppbyggnad

Fältet `organizationIdentifier` (OID 2.5.4.97) innehåller ett registreringsnummer för organisationen. Samma fält används även av certifikat enligt eIDAS och ZertES. För S/MIME-certifikat föreskriver BR följande uppbyggnad:

| Del | Exempel | Betydelse |
|---|---|---|
| Registreringsschema, 3 tecken | `NTR` | `National Trade Register`, handelsregistret |
| Landskod enligt ISO 3166-1 | `CH` | Landet där registret förs |
| Bindestreck | `-` | Avskiljare |
| Registreringsnummer | `CHE-123.456.789` | Organisationens nummer i registret |

Sammansatt blir detta `NTRCH-CHE-123.456.789`. Eftersom UID själv innehåller ett bindestreck fastställer BR att schema och landskod aldrig innehåller ett bindestreck. Det första bindestrecket från vänster skiljer alltså schemat från registreringsnumret, alla ytterligare hör till numret. Om ett register tilldelar nummer endast på delstats- eller provinsnivå följer en `+` efter landskoden med regionens kod, exempelvis `NTRUS+CA-12345678` för Kalifornien.

BR känner till dessa registreringsscheman:

| Schema | Numrets ursprung | Exempel från BR |
|---|---|---|
| `NTR` | Handels- eller företagsregister | `NTRGB-12345678` |
| `VAT` | Skattemyndighet (mervärdesskattenummer) | `VATEL-123456789` |
| `PSD` | Tillståndsnummer för en betaltjänstleverantör enligt EU-direktiv 2015/2366 | `PSDBE-NBB-1234.567.890` |
| `LEI` | `Legal Entity Identifier` enligt ISO 17442, landskod alltid `XG` | `LEIXG-` följt av LEI |
| `GOV` | Myndigheter utan nummer från något av de övriga schemana | `GOVUS` |
| `INT` | Internationella organisationer | `INTXG` |

För schweiziska företag är `NTR` med UID det normala fallet. UID tilldelas av Bundesamt für Statistik. Handelsregistren använder den som företagsnummer, den används även som mervärdesskattenummer och förblir densamma när ett företag flyttar sitt säte till en annan kanton. Ett mervärdesskattenummer kan däremot upphöra när skattskyldigheten upphör, och alla organisationer har inte ett sådant. SwissSign övertar UID med punkterna, precis som den står i UID-registret, och anger `NTRCH-CHE-109.357.012` som exempel för SwissSign AG.

**EU-anmärkning:** I Tyskland tilldelar Amtsgerichte handelsregisternummer, och samma nummer förekommer vid flera domstolar. Sedan version 1.0.9 från maj 2025 tillåter BR därför `NTR`-schemat `European Unique Identifier` (EUID) och rekommenderar det för tyska företag, exempelvis `NTRDE-DER3306.HRB12345` för ett företag i Handelsregister vid Amtsgericht Köln. Enligt BR behöver registreringsnumret endast vara entydigt där register och rättsordning tilldelar entydiga nummer.

## Vad CA:n kontrollerar

Innan en CA utfärdar ett OV- eller SV-certifikat måste den enligt BR kontrollera följande uppgifter:

- **Organisationens namn och existens:** via en myndighet i organisationens land, via LEI-data, ett besök på plats eller en bekräftelse med registerutdrag. För ett schweiziskt företag är det handelsregistret och UID-registret. Om företaget där anges som avregistrerat eller inaktivt får CA:n inte utfärda certifikatet.
- **Registreringsnummer:** CA:n bekräftar att numret i `organizationIdentifier` tillhör organisationen i fältet `O`.
- **Fullmakt:** Via en tillförlitlig kommunikationskanal kontrollerar CA:n att den sökande personen får ansöka om certifikat för organisationen eller agera som `Enterprise RA`.
- **Postlåda eller e-postdomän:** CA:n kontrollerar kontrollen över e-postadresserna i certifikatet. Om organisationen agerar som `Enterprise RA` måste den styrka kontrollen över hela e-postdomänen.
- **Person (endast SV):** BR tillåter att organisationen som `Enterprise RA` själv bekräftar sina medarbetare. CA:n kontrollerar då organisationen och inte varje enskild person.

En CA får återanvända resultaten av dessa kontroller: kontrollen av organisationen och fullmakten under 825 dagar, kontrollen av domänkontrollen under 398 dagar. Ett certifikat av generationerna `Multipurpose` och `Strict` är giltigt i högst 825 dagar.

## Jämförelse mellan SwissSign och DigiCert

Båda CA:erna erbjuder typerna MV, SV och OV under egna produktnamn. Ingen av dem erbjuder ett IV-certifikat. BR fastställer uppbyggnaden av `organizationIdentifier`, och den är densamma för båda leverantörerna.

| | SwissSign | DigiCert |
|---|---|---|
| MV | Personal S/MIME E-Mail ID Silver | Secure Email for Individual Mailbox |
| SV | Pro S/MIME E-Mail ID Gold | Secure Email for Employee (tidigare Secure Email for Business) |
| OV | Organization Validated E-Mail Certificate | Secure Email for Organization |
| IV | erbjuds inte | erbjuds inte |
| Generation för SV och OV | `Multipurpose` enligt certifikatprofilerna | `Strict` som standard, `Multipurpose` valbar |
| Person vid SV | För- och efternamn eller pseudonym med prefixet `pseudo:` i `CN` | För- och efternamn eller pseudonym; i `CN` valfritt namnet eller e-postadressen |
| Ytterligare användningar vid SV | i profilen med autentisering: klientautentisering, smartkortsinloggning, EFS | vid `Strict` oavvislighet, vid `Multipurpose` dessutom datakryptering och klientautentisering |

DigiCert beskriver uttryckligen SV-produkten enligt modellen `Enterprise RA`: Organisationen bekräftar att personen i certifikatet tillhör dess medarbetare eller representanter och bevarar underlagen för detta. DigiCert kontrollerar själv endast organisationen. Vid OV och SV kräver DigiCert alltid bevis på domänkontroll. Organisationskontrollen gäller i 825 dagar, domänkontrollen i 398 dagar. Som giltighetstid kan man välja ett eller två år eller ett eget utgångsdatum på högst 824 dagar. DigiCert avvecklade de tidigare produkterna Premium, Email Security Plus, Digital Signature Plus och Class 1 S/MIME den 10 juli 2025 och ersatte dem med dessa produkter.

Hos SwissSign innehåller OV-certifikatet i `CN` organisationens namn som i fältet `O`. Artikeln [S/MIME hos banker: rättsläge i Schweiz och valet mellan Silver och Gold](/blog/smime-silver-gold-schweiz) beskriver vilken nivå en bank bör välja för vilken postlåda och vad det innebär juridiskt.

## Kontrollera certifikattypen

I Outlook och andra e-postprogram går det att öppna certifikatuppgifterna för ett signerat e-postmeddelande via signaturikonen. I Windows finns fälten på fliken «Details» under posten «Subject». För OID 2.5.4.97 känner Windows inte till något visningsnamn och visar den som `OID.2.5.4.97`.

### Visa innehavare med PowerShell

För ett exporterat certifikat (till exempel från certifikatuppgifterna för ett signerat e-postmeddelande):

```powershell
$zert = Get-PfxCertificate -FilePath .\zertifikat.cer
$zert.SubjectName.Format($true)
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `Get-PfxCertificate` | läser ett certifikat från en fil, även från en `.cer`-fil utan privat nyckel |
| `-FilePath .\zertifikat.cer` | sökväg till certifikatfilen |
| `SubjectName.Format($true)` | skriver ut innehavarens fält rad för rad |

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

### Fastställ typ och generation via Policy-OID

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
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `$zert.Extensions` | tillägg för det tidigare inlästa certifikatet |
| `'2.5.29.32'` | OID för tillägget `Certificate Policies` |
| `Format($true)` | skriver ut tilläggets innehåll på flera rader |

</details>

Utdata (förkortad):

```text
[1]Certificate Policy:
     Policy Identifier=2.23.140.1.5.3.2
```

`2.23.140.1.5.3.2` står för ett SV-certifikat av generationen `Multipurpose`. Ytterligare poster med OID:er från CA:n själv är normala.

### Lista S/MIME-certifikat i det egna certifikatarkivet

```powershell
Get-ChildItem Cert:\CurrentUser\My |
  Where-Object { $_.EnhancedKeyUsageList.ObjectId -contains '1.3.6.1.5.5.7.3.4' } |
  Select-Object Subject, NotAfter
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `Cert:\CurrentUser\My` | den inloggade användarens personliga certifikatarkiv |
| `EnhancedKeyUsageList.ObjectId` | certifikatets utökade nyckelanvändningar |
| `'1.3.6.1.5.5.7.3.4'` | OID för e-postskydd (`id-kp-emailProtection`) |
| `Select-Object Subject, NotAfter` | visar innehavare och utgångsdatum |

</details>

### Med OpenSSL

```bash
openssl x509 -in zertifikat.pem -noout -subject -nameopt multiline
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `x509` | underkommando för certifikat |
| `-in zertifikat.pem` | indatafil i PEM-format; för en DER-fil dessutom `-inform der` |
| `-noout` | undertrycker utmatning av det kodade certifikatet |
| `-subject` | skriver ut innehavaren |
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

### Kontrollera registreringsnumret

Som mottagare av ett signerat e-postmeddelande kan du kontrollera organisationsuppgiften i fyra steg:

1. E-postprogrammet anger signaturen som giltig, och certifikatet kommer från en CA som e-postprogrammet litar på.
2. Policy-OID börjar med `2.23.140.1.5.2` eller `2.23.140.1.5.3`, och certifikatet innehåller fälten `O` och `organizationIdentifier`.
3. UID från `organizationIdentifier` tillhör den förväntade firman i UID-registret, och firman är aktiv där.
4. Avsändaradressens domän tillhör detta företag.

## Avgränsning mot ZertES

Organisationsuppgiften gör inte ett S/MIME-certifikat till ett reglerat certifikat enligt ZertES. Reglerade certifikat utfärdas endast av leverantörer som är erkända enligt ZertES, och certifikatet måste innehålla upplysningen att det är ett reglerat certifikat (art. 2 bokst. g och art. 7 stycke 2 bokst. b ZertES).

Innehållsmässigt finns paralleller. Ett reglerat certifikat för en UID-enhet måste innehålla dess UID (art. 7 stycke 2 bokst. e ZertES), precis som `organizationIdentifier` innehåller den för OV och SV. Ett certifikat för en fysisk person kan dessutom innehålla upplysningen att personen har rätt att företräda en viss UID-enhet (art. 7 stycke 3 bokst. b ZertES); detta motsvarar till sin uppbyggnad ett SV-certifikat.

| | S/MIME med organisationsuppgift | reglerat certifikat enligt ZertES |
|---|---|---|
| Grund | S/MIME Baseline Requirements från CA/Browser Forum | ZertES och tillämpningsbestämmelser |
| Utfärdare | offentligt betrodd CA | leverantör erkänd enligt ZertES |
| Organisationsidentifierare | `organizationIdentifier`, för schweiziska företag med UID | UID, obligatoriskt för UID-enheter |
| Kontroll av personen | för SV av CA:n eller av organisationen som `Enterprise RA` | i princip personligt inställelse med identitetshandling; för UID-enheter en företrädare med bevis på behörighet att företräda (art. 9 ZertES) |
| Nyckel | enligt CA:ns krav, även på en gateway | krav enligt art. 6 ZertES |

Endast den kvalificerade elektroniska signaturen med kvalificerad tidsstämpel är enligt art. 14 stycke 2bis OR likställd med egenhändig underskrift, och ett kvalificerat certifikat får endast utfärdas till en fysisk person (art. 8 stycke 1 ZertES). För en organisations signatur föreskriver ZertES det reglerade elektroniska sigillet: en avancerad elektronisk signatur som skapas med en säker enhet för sigillframställning och grundar sig på ett reglerat certifikat för en UID-enhet (art. 2 bokst. d ZertES). Ett S/MIME-signat e-postmeddelande är bevisning vars värde domstolen bedömer fritt (art. 157 ZPO); en kontrollerad organisationsuppgift stöder då kopplingen till det avsändande företaget.

**EU-anmärkning:** I EU reglerar eIDAS-förordningen elektroniska signaturer och sigill. Enligt art. 25 stycke 2 eIDAS är även där endast den kvalificerade elektroniska signaturen likställd med en handskriven underskrift.

## Rekommendation

- **Funktionspostlådor:** OV-certifikat (SwissSign Organization Validated, DigiCert Secure Email for Organization). Det styrker företaget utan att nämna en person.
- **Personliga postlådor med signatur gentemot kunder och partner:** SV-certifikat (SwissSign Gold, DigiCert Secure Email for Employee). Det styrker personen och företaget.
- **Endast kryptering:** Kryptografiskt är alla typer likvärdiga. För enbart kryptering räcker ett MV-certifikat.
- **Planera för kontroller:** Organisationskontrollen gäller i 825 dagar, domänkontrollen i 398 dagar. Efter en ändring av företagsnamn eller säte överensstämmer uppgifterna i certifikatet inte längre med registret; planera då för nya certifikat.
- **Kontrollera vid mottagande:** Policy-OID och UID visar om en signatur faktiskt kan kopplas till ett kontrollerat företag.

## Källor

1.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): Version 1.0.17 av den 20 november 2026; certifikattyper och Policy-OID:er (avsnitt 1.1), kontroll av organisationen (3.2.3), kontroll av fullmakt (3.2.6), återanvändning av kontrollresultat (4.2.1), giltighetstider (6.3.2), Subject-fält (7.1.4.2) och registreringsscheman (bilaga A).

2.  [cabforum/smime på GitHub: SBR.md](https://github.com/cabforum/smime/blob/main/SBR.md): Markdown-version av S/MIME Baseline Requirements med versionshistorik, bland annat EUID från version 1.0.9.

3.  [SwissSign: EUID, How Your Digital Certificates Become Even More Trustworthy](https://www.swisssign.com/en/blog/european-unique-identifier-euid): Produktnamnen Silver och Gold, exempel `NTRCH-CHE-109.357.012`, UID som registreringsnummer och EUID för tyska företag (28.04.2025).

4.  [SwissSign CPR S/MIME](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): SwissSigns certifikatprofiler för MV, SV och OV med Subject-fält, användningsområden och Policy-OID:er, version 14 av den 18.09.2025.

5.  [DigiCert: Secure Email products](https://docs.digicert.com/en/certcentral/order-and-manage-certificates/secure-email-certificates.html): Produktöversikt med koppling till MV, OV och SV, slutet för generationen `Legacy` och de tidigare S/MIME-produkterna den 10 juli 2025.

6.  [DigiCert: Order your Secure Email for Employee certificate](https://docs.digicert.com/en/certcentral/order-and-manage-certificates/secure-email-certificates/order-your-secure-email-for-employee-certificate.html): Organisations- och domänkontroll, organisationens bekräftelse av medarbetare, profilerna `Strict` och `Multipurpose`, giltighetstider.

7.  [Bundesgesetz über die elektronische Signatur (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): Art. 2 (begrepp, reglerat sigill), art. 6 (nycklar), art. 7 (innehåll i reglerade certifikat, UID), art. 8 (kvalificerade certifikat) och art. 9 (identitetskontroll).

8.  [Obligationenrecht (OR)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): Art. 14 stycke 2bis om likställandet av den kvalificerade elektroniska signaturen med egenhändig underskrift.

9.  [UID-register](https://www.uid.admin.ch): Sökning av en UID med namn, säte och status, förs av Bundesamt für Statistik.

10.  [Zefix](https://www.zefix.ch): Centralt företagsindex för de kantonala handelsregistren.

11.  [Zivilprozessordnung (ZPO)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): Art. 157 om fri bevisvärdering.

12.  [Förordning (EU) nr 910/2014 (eIDAS)](https://eur-lex.europa.eu/legal-content/DE/TXT/?uri=CELEX:32014R0910): Art. 25 stycke 2 om likställandet av den kvalificerade elektroniska signaturen med en handskriven underskrift.
