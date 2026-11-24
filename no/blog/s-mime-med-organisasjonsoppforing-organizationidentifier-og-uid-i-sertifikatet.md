---
title: "S/MIME med organisasjonsoppføring: organizationIdentifier og UID i sertifikatet"
navTitle: "S/MIME med organisasjon"
description: "Hva et S/MIME-sertifikat med organisasjonsoppføring inneholder, hvordan UID-en kommer inn i sertifikatet som organizationIdentifier, og hva sertifiseringsinstansen kontrollerer. I tillegg en sammenligning av produktene fra SwissSign og DigiCert, kommandoer for å kontrollere sertifikattypen og avgrensningen mot ZertES."
date: "2026-12-15"
kategorie: "E-postkryptering"
timeToRead: "11 min lesetid"
themen:
  - e-mail-verschluesselung
produkte: []
protokolle:
  - "verschluesselung"
  - "powershell"
hauptthema: "e-mail-verschluesselung"
startseiteAb: "2026-12-15"
slug: "s-mime-med-organisasjonsoppforing-organizationidentifier-og-uid-i-sertifikatet"
translationId: "article-e0f9bcc57f360835"
translationOf: smime-organisation-uid
url: https://rafaelpfister.ch/no/blog/s-mime-med-organisasjonsoppforing-organizationidentifier-og-uid-i-sertifikatet
translationSourceHash: f628a8f86635449d2a9e43f78f1a960ce5c8bc20af519eac110b77b2dad385a9
translationModel: gpt-5.6-terra
translatedAt: 2026-11-24T13:56:10.109Z
translationReview: automatic
---

# S/MIME med organisasjonsoppføring: organizationIdentifier og UID i sertifikatet

Sertifiseringsinstanser (CA) skiller mellom S/MIME-sertifikater med og uten organisasjonsoppføring. Et sertifikat med organisasjonsoppføring viser, i tillegg til e-postadressen, selskapet som postkassen tilhører: med det verifiserte navnet og et registreringsnummer. For sveitsiske selskaper er dette registreringsnummeret UID-en, altså CHE-nummeret fra handelsregisteret.

Kort oppsummert: En organisasjonsoppføring finnes bare for sertifikattypene `Organization-validated` og `Sponsor-validated` i S/MIME Baseline Requirements. Den består av selskapsnavnet i feltet `O` og feltet `organizationIdentifier` (OID 2.5.4.97), for et sveitsisk selskap for eksempel `NTRCH-CHE-123.456.789`. CA-en kontrollerer selskapet i et offentlig register før den skriver disse opplysningene i sertifikatet. Et postkassevalidert sertifikat dokumenterer derimot bare tilgang til en postkasse. Hos SwissSign tilsvarer dette valget mellom Silver og Gold, hos DigiCert valget mellom «Secure Email for Individual Mailbox» og produktene «for Employee» og «for Organization». En kvalifisert signatur etter ZertES oppstår ikke gjennom organisasjonsoppføringen.

**Sveits-fokus:** Den rettslige vurderingen følger sveitsisk rett (ZertES, OR). Der EU-reglene avviker, finnes en merket merknad.

## De fire sertifikattypene

Siden 1. september 2023 gjelder `S/MIME Baseline Requirements` (BR) fra CA/Browser Forum for offentlig tillitsverdige S/MIME-sertifikater. De skiller mellom fire sertifikattyper ut fra hvem som står som innehaver (`Subject`) i sertifikatet:

| Type | Innehaver i sertifikatet | Organisasjonsoppføring | Hva CA-en kontrollerer |
|---|---|---|---|
| `Mailbox-validated` (MV) | bare e-postadressen | ikke tillatt | Kontroll over postkassen eller e-postdomenet |
| `Organization-validated` (OV) | en organisasjon, ingen person | obligatorisk | Organisasjon samt postkasse eller e-postdomene |
| `Sponsor-validated` (SV) | en person som tilhører en organisasjon | obligatorisk | som OV, i tillegg personen; denne kontrolleres av CA-en eller av organisasjonen selv |
| `Individual-validated` (IV) | en person uten organisasjon | ikke tillatt | Personens identitet og postkasse |

Med «S/MIME med organisasjonsoppføring» menes typene OV og SV. For begge krever BR feltene `organizationName` og `organizationIdentifier`, i alle de tre generasjonene `Legacy`, `Multipurpose` og `Strict`. Generasjonene skiller seg ved tillatte Subject-felt, utvidelser og bruksformål. BR avviklet generasjonen `Legacy` 15. juli 2025; SwissSign og DigiCert har ikke utstedt den siden.

OV-sertifikater egner seg for funksjonelle postkasser som `info@` eller `buchhaltung@`, som ikke er tilordnet en person. SV-sertifikater er beregnet på ansattes personlige postkasser og inneholder i tillegg personens navn eller et pseudonym.

## Oppbygning av organizationIdentifier

Feltet `organizationIdentifier` (OID 2.5.4.97) inneholder et registreringsnummer for organisasjonen. Det samme feltet brukes også av sertifikater etter eIDAS og ZertES. For S/MIME-sertifikater krever BR følgende oppbygning:

| Del | Eksempel | Betydning |
|---|---|---|
| Registreringsskjema, 3 tegn | `NTR` | `National Trade Register`, handelsregisteret |
| Landskode etter ISO 3166-1 | `CH` | Landet der registeret føres |
| Bindestrek | `-` | Skilletegn |
| Registreringsnummer | `CHE-123.456.789` | Organisasjonens nummer i registeret |

Sammensatt blir dette `NTRCH-CHE-123.456.789`. Fordi UID-en selv inneholder en bindestrek, fastsetter BR at skjema og landskode aldri inneholder bindestrek. Den første bindestreken fra venstre skiller dermed skjemaet fra registreringsnummeret, mens alle øvrige hører til nummeret. Dersom et register bare tildeler numre på delstats- eller provinsnivå, følger et `+` med regionkoden etter landskoden, som `NTRUS+CA-12345678` for California.

BR kjenner disse registreringsskjemaene:

| Skjema | Nummerets opprinnelse | Eksempel fra BR |
|---|---|---|
| `NTR` | Handels- eller foretaksregister | `NTRGB-12345678` |
| `VAT` | Skattemyndighet (merverdiavgiftsnummer) | `VATEL-123456789` |
| `PSD` | Tillatelsesnummer for betalingsforetak etter EU-direktiv 2015/2366 | `PSDBE-NBB-1234.567.890` |
| `LEI` | `Legal Entity Identifier` etter ISO 17442, landskoden alltid `XG` | `LEIXG-` etterfulgt av LEI |
| `GOV` | Offentlige myndigheter uten nummer fra et av de øvrige skjemaene | `GOVUS` |
| `INT` | Internasjonale organisasjoner | `INTXG` |

For sveitsiske selskaper er `NTR` med UID-en hovedregelen. UID-en tildeles av Bundesamt für Statistik. Handelsregistrene bruker den som selskapsnummer, den fungerer også som merverdiavgiftsnummer, og den forblir den samme når et selskap flytter sitt sete til en annen kanton. Et merverdiavgiftsnummer kan derimot falle bort når skatteplikten opphører, og ikke alle organisasjoner har et. SwissSign overtar UID-en med punktene, slik den står i UID-registeret, og oppgir `NTRCH-CHE-109.357.012` som eksempel for SwissSign AG.

**EU-merknad:** I Tyskland tildeler Amtsgerichte handelsregisternumrene, og samme nummer forekommer ved flere domstoler. Siden versjon 1.0.9 fra mai 2025 tillater BR derfor `European Unique Identifier` (EUID) i skjemaet `NTR` og anbefaler det for tyske selskaper, for eksempel `NTRDE-DER3306.HRB12345` for et selskap i handelsregisteret hos Amtsgericht Köln. Etter BR må registreringsnummeret bare være entydig der register og rettsorden tildeler entydige numre.

## Hva CA-en kontrollerer

Før en CA utsteder et OV- eller SV-sertifikat, må den etter BR kontrollere følgende opplysninger:

- **Navn og eksistens til organisasjonen:** gjennom en myndighet i organisasjonens land, gjennom LEI-data, et besøk på stedet eller en bekreftelse med registerutskrift. For et sveitsisk selskap er dette handelsregisteret og UID-registeret. Dersom selskapet der er oppført som slettet eller inaktivt, kan CA-en ikke utstede sertifikatet.
- **Registreringsnummer:** CA-en bekrefter at nummeret i `organizationIdentifier` tilhører organisasjonen i feltet `O`.
- **Fullmakt:** Gjennom en pålitelig kommunikasjonskanal kontrollerer CA-en at personen som søker, kan søke sertifikater for organisasjonen eller opptre som `Enterprise RA`.
- **Postkasse eller e-postdomene:** CA-en kontrollerer kontrollen over e-postadressene i sertifikatet. Dersom organisasjonen opptrer som `Enterprise RA`, må den dokumentere kontroll over hele e-postdomenet.
- **Person (kun SV):** BR tillater at organisasjonen som `Enterprise RA` selv bekrefter sine ansatte. CA-en kontrollerer da organisasjonen og ikke hver enkelt person.

En CA kan gjenbruke resultatene av disse kontrollene: kontrollen av organisasjonen og fullmakten i 825 dager, og kontrollen av domenekontrollen i 398 dager. Et sertifikat i generasjonene `Multipurpose` og `Strict` er gyldig i høyst 825 dager.

## Sammenligning av SwissSign og DigiCert

Begge CA-ene tilbyr typene MV, SV og OV under egne produktnavn. Ingen av de to tilbyr et IV-sertifikat. BR fastsetter oppbygningen av `organizationIdentifier`, og den er den samme hos begge leverandørene.

| | SwissSign | DigiCert |
|---|---|---|
| MV | Personal S/MIME E-Mail ID Silver | Secure Email for Individual Mailbox |
| SV | Pro S/MIME E-Mail ID Gold | Secure Email for Employee (tidligere Secure Email for Business) |
| OV | Organization Validated E-Mail Certificate | Secure Email for Organization |
| IV | ikke i tilbudet | ikke i tilbudet |
| Generasjon for SV og OV | `Multipurpose` ifølge sertifikatprofilene | `Strict` som standard, `Multipurpose` kan velges |
| Person ved SV | Fornavn og etternavn eller pseudonym med prefikset `pseudo:` i `CN` | Fornavn og etternavn eller pseudonym; i `CN` kan enten navnet eller e-postadressen velges |
| Ytterligere bruksområder ved SV | i profilen med autentisering: klientautentisering, smartkortpålogging, EFS | for `Strict` ikke-benekting, for `Multipurpose` i tillegg datakryptering og klientautentisering |

DigiCert beskriver SV-produktet uttrykkelig etter modellen `Enterprise RA`: Organisasjonen bekrefter at personen i sertifikatet hører til dens ansatte eller representanter, og oppbevarer dokumentasjonen for dette. DigiCert selv kontrollerer bare organisasjonen. For OV og SV krever DigiCert alltid dokumentasjon på domenekontroll. Organisasjonskontrollen gjelder i 825 dager, domenekontrollen i 398 dager. Du kan velge en gyldighet på ett eller to år, eller en egen utløpsdato på høyst 824 dager. DigiCert avviklet de tidligere produktene Premium, Email Security Plus, Digital Signature Plus og Class 1 S/MIME 10. juli 2025 og erstattet dem med disse produktene.

Hos SwissSign inneholder OV-sertifikatet i `CN` organisasjonens navn som i feltet `O`. Artikkelen [S/MIME hos banker: rettslig situasjon i Sveits og valget mellom Silver og Gold](/blog/smime-silver-gold-schweiz) beskriver hvilket nivå en bank bør velge for hvilken postkasse, og hva dette betyr juridisk.

## Kontrollere sertifikattypen

I Outlook og andre e-postprogrammer kan sertifikatdetaljene for en signert e-post åpnes via signaturikonet. I Windows finnes feltene i fanen «Detaljer» under oppføringen «Emne». For OID 2.5.4.97 kjenner Windows ikke noe visningsnavn og viser den som `OID.2.5.4.97`.

### Vise innehaver med PowerShell

For et eksportert sertifikat (for eksempel fra sertifikatdetaljene i en signert e-post):

```powershell
$zert = Get-PfxCertificate -FilePath .\zertifikat.cer
$zert.SubjectName.Format($true)
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Get-PfxCertificate` | leser et sertifikat fra en fil, også fra en `.cer`-fil uten privat nøkkel |
| `-FilePath .\zertifikat.cer` | bane til sertifikatfilen |
| `SubjectName.Format($true)` | skriver ut innehaverens felt, én per linje |

</details>

Utdata for et SV-sertifikat (eksempel):

```text
CN=Max Muster
G=Max
SN=Muster
O=Muster AG
OID.2.5.4.97=NTRCH-CHE-123.456.789
S=ZH
C=CH
```

For et MV-sertifikat inneholder utdataene bare `CN=` med e-postadressen.

### Bestemme type og generasjon via Policy-OID

Hvert sertifikat etter BR inneholder i utvidelsen `Certificate Policies` en OID med mønsteret `2.23.140.1.5.<Typ>.<Generation>`:

| Posisjon | Verdi | Betydning |
|---|---|---|
| Type | `1` | `Mailbox-validated` |
| Type | `2` | `Organization-validated` |
| Type | `3` | `Sponsor-validated` |
| Type | `4` | `Individual-validated` |
| Generasjon | `1` | `Legacy` |
| Generasjon | `2` | `Multipurpose` |
| Generasjon | `3` | `Strict` |

```powershell
$zert.Extensions |
  Where-Object { $_.Oid.Value -eq '2.5.29.32' } |
  ForEach-Object { $_.Format($true) }
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `$zert.Extensions` | utvidelser for sertifikatet som ble lest inn tidligere |
| `'2.5.29.32'` | OID for utvidelsen `Certificate Policies` |
| `Format($true)` | skriver ut innholdet i utvidelsen over flere linjer |

</details>

Utdata (forkortet):

```text
[1]Certificate Policy:
     Policy Identifier=2.23.140.1.5.3.2
```

`2.23.140.1.5.3.2` står for et SV-sertifikat i generasjonen `Multipurpose`. Ytterligere oppføringer med OID-er for selve CA-en er normale.

### Liste over S/MIME-sertifikater i eget sertifikatlager

```powershell
Get-ChildItem Cert:\CurrentUser\My |
  Where-Object { $_.EnhancedKeyUsageList.ObjectId -contains '1.3.6.1.5.5.7.3.4' } |
  Select-Object Subject, NotAfter
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Cert:\CurrentUser\My` | personlig sertifikatlager for den påloggede brukeren |
| `EnhancedKeyUsageList.ObjectId` | sertifikatets utvidede nøkkelbruk |
| `'1.3.6.1.5.5.7.3.4'` | OID for e-postbeskyttelse (`id-kp-emailProtection`) |
| `Select-Object Subject, NotAfter` | viser innehaver og utløpsdato |

</details>

### Med OpenSSL

```bash
openssl x509 -in zertifikat.pem -noout -subject -nameopt multiline
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `x509` | underkommando for sertifikater |
| `-in zertifikat.pem` | inndatafil i PEM-format; for en DER-fil i tillegg `-inform der` |
| `-noout` | undertrykker utdata av det kodede sertifikatet |
| `-subject` | skriver ut innehaveren |
| `-nameopt multiline` | ett felt per linje med utskrevne feltnavn |

</details>

OpenSSL 3 kjenner feltnavnet og viser det fullt utskrevet:

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

### Kontrollere registreringsnummeret

Som mottaker av en signert e-post kan du kontrollere organisasjonsoppføringen i fire trinn:

1. E-postprogrammet oppgir at signaturen er gyldig, og sertifikatet kommer fra en CA som e-postprogrammet stoler på.
2. Policy-OID-en begynner med `2.23.140.1.5.2` eller `2.23.140.1.5.3`, og sertifikatet inneholder feltene `O` og `organizationIdentifier`.
3. UID-en fra `organizationIdentifier` tilhører det forventede selskapet i UID-registeret, og selskapet er aktivt der.
4. Domenet til avsenderadressen tilhører dette selskapet.

## Avgrensning mot ZertES

Organisasjonsoppføringen gjør ikke et S/MIME-sertifikat til et regulert sertifikat etter ZertES. Regulerte sertifikater utstedes bare av tilbydere som er anerkjent etter ZertES, og sertifikatet må inneholde opplysningen om at det er et regulert sertifikat (art. 2 bokstav g og art. 7 avsnitt 2 bokstav b ZertES).

Innholdsmessig finnes det paralleller. Et regulert sertifikat for en UID-enhet må inneholde dens UID (art. 7 avsnitt 2 bokstav e ZertES), på samme måte som `organizationIdentifier` inneholder den for OV og SV. Et sertifikat for en fysisk person kan dessuten inneholde opplysningen om at vedkommende har rett til å representere en bestemt UID-enhet (art. 7 avsnitt 3 bokstav b ZertES); oppbygningen tilsvarer et SV-sertifikat.

| | S/MIME med organisasjonsoppføring | regulert sertifikat etter ZertES |
|---|---|---|
| Grunnlag | S/MIME Baseline Requirements fra CA/Browser Forum | ZertES og gjennomføringsbestemmelser |
| Utsteder | offentlig tillitsverdig CA | tilbyder anerkjent etter ZertES |
| Organisasjonsidentifikator | `organizationIdentifier`, for sveitsiske selskaper med UID-en | UID, obligatorisk for UID-enheter |
| Kontroll av personen | ved SV av CA-en eller av organisasjonen som `Enterprise RA` | som hovedregel personlig oppmøte med identitetsbevis; for UID-enheter en representant med dokumentasjon på representasjonsfullmakt (art. 9 ZertES) |
| Nøkkel | etter CA-ens krav, også på en gateway | krav etter art. 6 ZertES |

Bare den kvalifiserte elektroniske signaturen med kvalifisert tidsstempel er likestilt med håndskrevet underskrift etter art. 14 avsnitt 2bis OR, og et kvalifisert sertifikat kan bare utstedes til en fysisk person (art. 8 avsnitt 1 ZertES). For en organisasjons signatur har ZertES det regulerte elektroniske seglet: en avansert elektronisk signatur som opprettes med en sikker seglopprettingsenhet og bygger på et regulert sertifikat for en UID-enhet (art. 2 bokstav d ZertES). En S/MIME-signert e-post er et bevismiddel som retten vurderer fritt (art. 157 ZPO); en kontrollert organisasjonsoppføring underbygger da tilordningen til det avsendende selskapet.

**EU-merknad:** I EU regulerer eIDAS-forordningen elektroniske signaturer og segl. Etter art. 25 avsnitt 2 eIDAS er også bare den kvalifiserte elektroniske signaturen likestilt med håndskrevet underskrift.

## Anbefaling

- **Funksjonelle postkasser:** OV-sertifikat (SwissSign Organization Validated, DigiCert Secure Email for Organization). Det identifiserer selskapet uten å nevne en person.
- **Personlige postkasser med signatur overfor kunder og partnere:** SV-sertifikat (SwissSign Gold, DigiCert Secure Email for Employee). Det identifiserer personen og selskapet.
- **Bare kryptering:** Kryptografisk er alle typene likeverdige. For kryptering alene er et MV-sertifikat tilstrekkelig.
- **Planlegg kontroller:** Organisasjonskontrollen gjelder i 825 dager, domenekontrollen i 398 dager. Etter en endring av selskapsnavn eller sete samsvarer ikke lenger opplysningene i sertifikatet med registeret; planlegg da nye sertifikater.
- **Kontroller ved mottak:** Policy-OID og UID viser om en signatur faktisk er tilordnet et kontrollert selskap.

## Kilder

1.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): Versjon 1.0.17 av 20. november 2026; sertifikattyper og Policy-OID-er (avsnitt 1.1), kontroll av organisasjonen (3.2.3), kontroll av fullmakt (3.2.6), gjenbruk av kontrollresultater (4.2.1), gyldighetsperioder (6.3.2), Subject-felt (7.1.4.2) og registreringsskjemaer (vedlegg A).

2.  [cabforum/smime på GitHub: SBR.md](https://github.com/cabforum/smime/blob/main/SBR.md): Markdown-versjon av S/MIME Baseline Requirements med versjonshistorikk, blant annet EUID fra versjon 1.0.9.

3.  [SwissSign: EUID, How Your Digital Certificates Become Even More Trustworthy](https://www.swisssign.com/en/blog/european-unique-identifier-euid): Produktbetegnelsene Silver og Gold, eksempel `NTRCH-CHE-109.357.012`, UID som registreringsnummer og EUID for tyske selskaper (28.04.2025).

4.  [SwissSign CPR S/MIME](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): Sertifikatprofiler fra SwissSign for MV, SV og OV med Subject-felt, bruksformål og Policy-OID-er, versjon 14 av 18.09.2025.

5.  [DigiCert: Secure Email products](https://docs.digicert.com/en/certcentral/order-and-manage-certificates/secure-email-certificates.html): Produktoversikt med tilordning til MV, OV og SV, slutten på generasjonen `Legacy` og de tidligere S/MIME-produktene 10. juli 2025.

6.  [DigiCert: Order your Secure Email for Employee certificate](https://docs.digicert.com/en/certcentral/order-and-manage-certificates/secure-email-certificates/order-your-secure-email-for-employee-certificate.html): Organisasjons- og domenekontroll, bekreftelse av ansatte fra organisasjonen, profilene `Strict` og `Multipurpose`, gyldighetsperioder.

7.  [Bundesgesetz über die elektronische Signatur (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): Art. 2 (begreper, regulert segl), art. 6 (nøkler), art. 7 (innhold i regulerte sertifikater, UID), art. 8 (kvalifiserte sertifikater) og art. 9 (identitetskontroll).

8.  [Obligationenrecht (OR)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): Art. 14 avsnitt 2bis om likestilling av kvalifisert elektronisk signatur med håndskrevet underskrift.

9.  [UID-register](https://www.uid.admin.ch): Oppslag av en UID med navn, sete og status, ført av Bundesamt für Statistik.

10.  [Zefix](https://www.zefix.ch): Sentralt selskapsregister for de kantonale handelsregistrene.

11.  [Zivilprozessordnung (ZPO)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): Art. 157 om fri bevisvurdering.

12.  [Forordning (EU) nr. 910/2014 (eIDAS)](https://eur-lex.europa.eu/legal-content/DE/TXT/?uri=CELEX:32014R0910): Art. 25 avsnitt 2 om likestilling av kvalifisert elektronisk signatur med håndskrevet underskrift.
