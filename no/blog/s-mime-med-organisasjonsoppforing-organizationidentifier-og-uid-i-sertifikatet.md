---
title: "S/MIME med organisasjonsoppføring: organizationIdentifier og UID i sertifikatet"
navTitle: "S/MIME med organisasjon"
description: "Hva et S/MIME-sertifikat med organisasjonsoppføring inneholder, hvordan UID-en kommer inn i sertifikatet som organizationIdentifier, og hva sertifiseringsinstansen kontrollerer for selskap og person. I tillegg en sammenligning av SwissSign og DigiCert, kommandoer for å kontrollere sertifikattypen og avgrensningen mot ZertES."
date: "2026-12-15"
kategorie: "E-postkryptering"
timeToRead: "13 min lesetid"
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
translationSourceHash: 849361658c86c3e7bae12ee2d7e018cf43807a611de4911b9ad1bdbe870ea0fe
translationModel: gpt-5.6-terra
translatedAt: 2026-11-24T14:59:40.922Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/s-mime-med-organisasjonsoppforing-organizationidentifier-og-uid-i-sertifikatet
---

# S/MIME med organisasjonsoppføring: organizationIdentifier og UID i sertifikatet

Sertifiseringsinstanser (CA-er) skiller mellom S/MIME-sertifikater med og uten organisasjonsoppføring. Et sertifikat med organisasjonsoppføring angir, i tillegg til e-postadressen, selskapet postkassen tilhører: med dets kontrollerte navn og et registreringsnummer. For sveitsiske selskaper er dette registreringsnummeret UID-en, altså CHE-nummeret fra handelsregisteret.

Kort oppsummert: En organisasjonsoppføring finnes bare for sertifikattypene `Organization-validated` og `Sponsor-validated` i S/MIME Baseline Requirements. Den består av selskapsnavnet i feltet `O` og feltet `organizationIdentifier` (OID 2.5.4.97), for et sveitsisk selskap for eksempel `NTRCH-CHE-123.456.789`. CA-en kontrollerer selskapet i et offentlig register før den skriver disse opplysningene inn i sertifikatet. Et mailbox-validert sertifikat dokumenterer derimot bare tilgang til en postkasse. Hos SwissSign tilsvarer dette valget mellom Silver og Gold, hos DigiCert valget mellom «Secure Email for Individual Mailbox» og produktene «for Employee» og «for Organization». En kvalifisert signatur etter ZertES oppstår ikke gjennom organisasjonsoppføringen.

**Sveits-fokus:** Den juridiske vurderingen følger sveitsisk rett (ZertES, OR). Der EU-reglene avviker, finnes en merket merknad.

## De fire sertifikattypene

Siden 1. september 2023 gjelder `S/MIME Baseline Requirements` (BR) fra CA/Browser Forum for offentlig betrodde S/MIME-sertifikater. De skiller mellom fire sertifikattyper etter hvem som står som innehaver (`Subject`) i sertifikatet:

| Type | Innehaver i sertifikatet | Organisasjonsoppføring | Hva CA-en kontrollerer |
|---|---|---|---|
| `Mailbox-validated` (MV) | bare e-postadressen | ikke tillatt | Kontroll over postkassen eller e-postdomenet |
| `Organization-validated` (OV) | en organisasjon, ingen person | obligatorisk | Organisasjonen samt postkasse eller e-postdomene |
| `Sponsor-validated` (SV) | en person som tilhører en organisasjon | obligatorisk | som OV, i tillegg personen; denne kontrolleres av CA-en eller av organisasjonen selv |
| `Individual-validated` (IV) | en person uten organisasjon | ikke tillatt | Personens identitet og postkasse |

Med «S/MIME med organisasjonsoppføring» menes typene OV og SV. For begge krever BR feltene `organizationName` og `organizationIdentifier`, i alle de tre generasjonene `Legacy`, `Multipurpose` og `Strict`. Generasjonene skiller seg ved tillatte Subject-felt, utvidelser og bruksformål. BR har avviklet generasjonen `Legacy` per 15. juli 2025; SwissSign og DigiCert har ikke utstedt den siden.

OV-sertifikater passer for funksjonelle postkasser som `info@` eller `buchhaltung@`, som ikke er knyttet til en person. SV-sertifikater er beregnet på personlige postkasser for ansatte og inneholder i tillegg personens navn eller et pseudonym.

## Oppbygging av organizationIdentifier

Feltet `organizationIdentifier` (OID 2.5.4.97) inneholder et registreringsnummer for organisasjonen. Det samme feltet brukes også av sertifikater etter eIDAS og ZertES. For S/MIME-sertifikater krever BR følgende oppbygging:

| Del | Eksempel | Betydning |
|---|---|---|
| Registreringsordning, 3 tegn | `NTR` | `National Trade Register`, handelsregisteret |
| Landskode etter ISO 3166-1 | `CH` | Landet der registeret føres |
| Bindestrek | `-` | Skilletegn |
| Registreringsnummer | `CHE-123.456.789` | Organisasjonens nummer i registeret |

Satt sammen blir dette `NTRCH-CHE-123.456.789`. Fordi UID-en selv inneholder en bindestrek, fastsetter BR at ordningen og landskoden aldri inneholder bindestrek. Den første bindestreken fra venstre skiller derfor ordningen fra registreringsnummeret, mens alle de øvrige hører til nummeret. Dersom et register bare tildeler numre på delstats- eller provinsnivå, følger en `+` med regionskoden etter landskoden, for eksempel `NTRUS+CA-12345678` for California.

BR kjenner disse registreringsordningene:

| Ordning | Nummerets opprinnelse | Eksempel fra BR |
|---|---|---|
| `NTR` | Handels- eller virksomhetsregister | `NTRGB-12345678` |
| `VAT` | Skattemyndighet (merverdiavgiftsnummer) | `VATEL-123456789` |
| `PSD` | Tillatelsesnummer for en betalingstjenesteleverandør etter EU-direktiv 2015/2366 | `PSDBE-NBB-1234.567.890` |
| `LEI` | `Legal Entity Identifier` etter ISO 17442, landskoden alltid `XG` | `LEIXG-` etterfulgt av LEI |
| `GOV` | Offentlige myndigheter uten nummer fra en av de øvrige ordningene | `GOVUS` |
| `INT` | Internasjonale organisasjoner | `INTXG` |

For sveitsiske selskaper er `NTR` med UID-en standardtilfellet. UID-en tildeles av Bundesamt für Statistik. Handelsregistrene bruker den som selskapsnummer, den brukes også som merverdiavgiftsnummer, og den forblir den samme når et selskap flytter sitt sete til en annen kanton. Et merverdiavgiftsnummer kan derimot bortfalle når avgiftsplikten opphører, og ikke alle organisasjoner har et. SwissSign overtar UID-en med punktene, slik den står i UID-registeret, og oppgir `NTRCH-CHE-109.357.012` som eksempel for SwissSign AG.

**EU-merknad:** I Tyskland tildeler Amtsgerichte handelsregisternumrene, og samme nummer forekommer ved flere domstoler. Siden versjon 1.0.9 fra mai 2025 tillater BR derfor `NTR`-ordningen `European Unique Identifier` (EUID) og anbefaler den for tyske selskaper, for eksempel `NTRDE-DER3306.HRB12345` for et selskap i handelsregisteret til Amtsgericht Köln. Etter BR må registreringsnummeret bare være entydig der registeret og rettsordenen tildeler entydige numre.

## Hva CA-en kontrollerer

Før en CA utsteder et OV- eller SV-sertifikat, må den kontrollere følgende opplysninger i henhold til BR:

- **Organisasjonens navn og eksistens:** via en myndighet i organisasjonens land, via LEI-data, et besøk på stedet eller en bekreftelse med registerutskrift. For et sveitsisk selskap er dette handelsregisteret og UID-registeret. Dersom selskapet er oppført der som slettet eller inaktivt, kan CA-en ikke utstede sertifikatet.
- **Registreringsnummer:** CA-en bekrefter at nummeret i `organizationIdentifier` tilhører organisasjonen i feltet `O`.
- **Fullmakt:** Gjennom en pålitelig kommunikasjonskanal kontrollerer CA-en at personen som søker, kan søke om sertifikater for organisasjonen eller handle som `Enterprise RA`.
- **Postkasse eller e-postdomene:** CA-en kontrollerer kontrollen over e-postadressene i sertifikatet. Dersom organisasjonen handler som `Enterprise RA`, må den dokumentere kontroll over hele e-postdomenet.
- **Person (kun SV):** Personens navn eller pseudonym. Hvilken dokumentasjon som er tillatt for dette, beskrives i neste avsnitt.

En CA kan gjenbruke resultatene av disse kontrollene: kontrollen av organisasjonen og fullmakten i 825 dager, kontrollen av domenekontrollen i 398 dager. Et sertifikat av generasjonene `Multipurpose` og `Strict` er gyldig i maksimalt 825 dager.

## Hvordan personen kontrolleres ved SV

BR tillater følgende dokumentasjon for personens identitet:

| Dokumentasjon | Forløp |
|---|---|
| Offentlig identitetsdokument | Original på stedet eller i sanntid foran et kamera; personen som kontrollerer sammenligner bildet i dokumentet med personen og registrerer utsteder, gyldighet og dokumentnummer |
| Chip i pass eller identitetskort | Kontroll av utstedersignaturen på chipdataene, i tillegg til fotosammenligningen |
| eID | bare notifiserte eID-systemer etter eIDAS med tillitsnivået «substansielt» eller «høyt» |
| Mobilt førerkort (mDL) | Spørring i sanntid etter ISO/IEC 18013-5 og 18013-7 med kontroll av utsteders signaturkjede |
| Digital signatur | Signering av søknaden med et eksisterende personlig sertifikat fra et anerkjent rammeverk; listen over disse rammeverkene er fortsatt tom i BR |
| Bekreftelse fra notarius eller advokatfirma | Skriftlig bekreftelse fra landet til personen som søker |
| Dokumentasjon fra `Enterprise RA` | Selskapet kontrollerer sine ansatte selv og kan støtte seg på eksisterende interne data, for eksempel fra personalavdelingen |

Hvilken av disse dokumentasjonene en CA aksepterer, må den offentliggjøre i sin CPS. Kontrollen av personen kan ved utstedelsen maksimalt være 825 dager gammel. Dersom selskapet selv kontrollerer som `Enterprise RA`, må CA-en kontraktsmessig forplikte det og kontrollere det minst én gang årlig, med mindre selskapet selv blir revidert årlig. Tilknytningen til personen er i dette tilfellet like pålitelig som selskapets interne prosess.

## Sammenligning av SwissSign og DigiCert

Begge CA-ene tilbyr typene MV, SV og OV under egne produktnavn. Ingen av dem tilbyr et IV-sertifikat. Oppbyggingen av `organizationIdentifier` fastsettes av BR og er den samme for begge leverandørene.

| | SwissSign | DigiCert |
|---|---|---|
| MV | Personal S/MIME E-Mail ID Silver | Secure Email for Individual Mailbox |
| SV | Pro S/MIME E-Mail ID Gold | Secure Email for Employee (tidligere Secure Email for Business) |
| OV | Organization Validated E-Mail Certificate | Secure Email for Organization |
| IV | ikke tilgjengelig | ikke tilgjengelig |
| Anskaffelse | Silver i nettbutikken eller via Managed PKI, Gold og OV bare via selskapets Managed PKI | alle tre via selskapets CertCentral-konto |
| Kontroll av personen ved SV | offentlig identitetsdokument i original, på stedet eller i en tilsvarende prosedyre; annen dokumentasjon bare med godkjenning fra SwissSign | bekreftelse fra selskapet; DigiCert kontrollerer ikke personen |
| Gyldighet for kontrollene | opplysninger og dokumentasjon maksimalt 398 dager | organisasjon 825 dager, domene 398 dager |
| Generasjon ved SV og OV | `Multipurpose`; en mellomliggende CA for SV i generasjonen `Strict` er allerede beskrevet i CP/CPS | `Strict` som standard, `Multipurpose` kan velges |
| Person ved SV | for- og etternavn eller pseudonym med prefikset `pseudo:` i `CN` | for- og etternavn eller pseudonym; i `CN` kan enten navnet eller e-postadressen oppgis |
| Ytterligere bruksområder ved SV | ingen i standardprofilen; i profilen med autentisering: klientautentisering, smartkortpålogging og EFS | ved `Strict` ikke-benekting, ved `Multipurpose` i tillegg datakryptering og klientautentisering |

DigiCert beskriver SV-produktet uttrykkelig etter modellen `Enterprise RA`: Organisasjonen bekrefter at personen i sertifikatet hører til blant de ansatte eller representantene, og oppbevarer dokumentasjonen for dette. DigiCert selv kontrollerer bare organisasjonen. Den som kan bestille i selskapets CertCentral-konto, kan dermed oppgi hvilket som helst navn på en adresse i egne domener. Ved OV og SV krever DigiCert alltid dokumentasjon på domenekontroll. Organisasjonskontrollen gjelder i 825 dager, domenekontrollen i 398 dager. Man kan velge en løpetid på ett eller to år, eller en egen utløpsdato på inntil 824 dager. DigiCert avviklet de tidligere produktene Premium, Email Security Plus, Digital Signature Plus og Class 1 S/MIME 10. juli 2025 og erstattet dem med disse produktene.

SwissSign utsteder Gold- og OV-sertifikater bare via selskapets Managed PKI. Personen må likevel identifisere seg med et offentlig identitetsdokument i original, personlig eller i en tilsvarende prosedyre etter ETSI EN 319 411-1. Identifikasjonen utføres av den registrerende instansen som behandler søknaden, eller en opplært identifikasjonspartner på kontrakt: kopiere, skanne eller fotografere dokumentet, sammenligne bildet med personen og bekrefte prosessen overfor SwissSign. Bruken av selskapsnavnet må godkjennes av den daglige ledelsen eller styret som er registrert i handelsregisteret, og personene som søker om sertifikater i Managed PKI, identifiseres også med identitetsdokument. En annen prosedyre, for eksempel basert på personaldata, kan et selskap bare bruke dersom SwissSign har kontrollert og godkjent den som likeverdig. Opplysninger og dokumentasjon gjelder hos SwissSign i maksimalt 398 dager, deretter kontrollerer SwissSign på nytt.

Hos SwissSign inneholder OV-sertifikatet i `CN` organisasjonens navn slik det står i feltet `O`. Hvilket nivå en bank bør velge for hvilken postkasse, og hva dette betyr juridisk, beskriver artikkelen [S/MIME hos banker: rettstilstanden i Sveits og valget mellom Silver og Gold](/blog/smime-silver-gold-schweiz).

## Kontrollere sertifikattypen

I Outlook og andre e-postprogrammer kan sertifikatdetaljene for en signert e-post åpnes via signaturikonet. I Windows finnes feltene under fanen «Detaljer» ved oppføringen «Emne». For OID 2.5.4.97 kjenner Windows ikke noe visningsnavn og viser den som `OID.2.5.4.97`.

### Vise innehaver med PowerShell

For et eksportert sertifikat (for eksempel fra sertifikatdetaljene i en signert e-post):

```powershell
$zert = Get-PfxCertificate -FilePath .\zertifikat.cer
$zert.SubjectName.Format($true)
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `Get-PfxCertificate` | leser et sertifikat fra en fil, også fra en `.cer`-fil uten privat nøkkel |
| `-FilePath .\zertifikat.cer` | sti til sertifikatfilen |
| `SubjectName.Format($true)` | viser innehaverens felt linje for linje |

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

### Bestemme type og generasjon via policy-OID

Hvert sertifikat etter BR inneholder en OID i utvidelsen `Certificate Policies` etter mønsteret `2.23.140.1.5.<Typ>.<Generation>`:

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
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `$zert.Extensions` | utvidelser i sertifikatet som ble lest inn tidligere |
| `'2.5.29.32'` | OID for utvidelsen `Certificate Policies` |
| `Format($true)` | viser innholdet i utvidelsen over flere linjer |

</details>

Utdata (forkortet):

```text
[1]Certificate Policy:
     Policy Identifier=2.23.140.1.5.3.2
```

`2.23.140.1.5.3.2` står for et SV-sertifikat i generasjonen `Multipurpose`. Flere oppføringer med OID-er fra selve CA-en er normalt.

### Liste over S/MIME-sertifikater i egen sertifikatlager

```powershell
Get-ChildItem Cert:\CurrentUser\My |
  Where-Object { $_.EnhancedKeyUsageList.ObjectId -contains '1.3.6.1.5.5.7.3.4' } |
  Select-Object Subject, NotAfter
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `Cert:\CurrentUser\My` | personlig sertifikatlager for den påloggede brukeren |
| `EnhancedKeyUsageList.ObjectId` | utvidede nøkkelbruksområder for sertifikatet |
| `'1.3.6.1.5.5.7.3.4'` | OID for e-postbeskyttelse (`id-kp-emailProtection`) |
| `Select-Object Subject, NotAfter` | viser innehaver og utløpsdato |

</details>

### Med OpenSSL

```bash
openssl x509 -in zertifikat.pem -noout -subject -nameopt multiline
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `x509` | underkommando for sertifikater |
| `-in zertifikat.pem` | inndatafil i PEM-format; for en DER-fil i tillegg `-inform der` |
| `-noout` | undertrykker utdata av det kodede sertifikatet |
| `-subject` | viser innehaveren |
| `-nameopt multiline` | ett felt per linje med fullstendig utskrevne feltnavn |

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

1. E-postprogrammet melder at signaturen er gyldig, og sertifikatet kommer fra en CA som e-postprogrammet stoler på.
2. Policy-OID-en begynner med `2.23.140.1.5.2` eller `2.23.140.1.5.3`, og sertifikatet inneholder feltene `O` og `organizationIdentifier`.
3. UID-en fra `organizationIdentifier` tilhører det forventede selskapet i UID-registeret, og selskapet er aktivt der.
4. Domenet i avsenderadressen tilhører dette selskapet.

## Avgrensning mot ZertES

Organisasjonsoppføringen gjør ikke et S/MIME-sertifikat til et regulert sertifikat etter ZertES. Regulerte sertifikater utstedes bare av leverandører som er anerkjent etter ZertES, og sertifikatet må inneholde en henvisning til at det er et regulert sertifikat (art. 2 bokstav g og art. 7 avsnitt 2 bokstav b ZertES).

Innholdsmessig finnes det paralleller. Et regulert sertifikat for en UID-enhet må inneholde dens UID (art. 7 avsnitt 2 bokstav e ZertES), på samme måte som `organizationIdentifier` inneholder den ved OV og SV. Et sertifikat for en fysisk person kan dessuten inneholde en henvisning til at vedkommende er berettiget til å representere en bestemt UID-enhet (art. 7 avsnitt 3 bokstav b ZertES); dette tilsvarer oppbyggingen av et SV-sertifikat.

| | S/MIME med organisasjonsoppføring | regulert sertifikat etter ZertES |
|---|---|---|
| Grunnlag | S/MIME Baseline Requirements fra CA/Browser Forum | ZertES og gjennomføringsbestemmelser |
| Utsteder | offentlig betrodd CA | leverandør anerkjent etter ZertES |
| Organisasjonsidentifikator | `organizationIdentifier`, for sveitsiske selskaper med UID-en | UID, obligatorisk for UID-enheter |
| Kontroll av personen | ved SV avhengig av CA med identitetsdokument eller gjennom bekreftelse fra organisasjonen som `Enterprise RA` | som hovedregel personlig fremmøte med identitetsbevis; for UID-enheter en representant med dokumentasjon på representasjonsfullmakt (art. 9 ZertES) |
| Nøkkel | etter CA-ens krav, også på en gateway | krav etter art. 6 ZertES |

Bare den kvalifiserte elektroniske signaturen med kvalifisert tidsstempel er likestilt med egenhendig underskrift etter art. 14 avsnitt 2bis OR, og et kvalifisert sertifikat kan bare utstedes til en fysisk person (art. 8 avsnitt 1 ZertES). For signatur fra en organisasjon har ZertES det regulerte elektroniske seglet: en avansert elektronisk signatur som opprettes med en sikker seglopprettingsenhet og bygger på et regulert sertifikat for en UID-enhet (art. 2 bokstav d ZertES). En S/MIME-signert e-post er et bevismiddel som retten vurderer fritt (art. 157 ZPO); en kontrollert organisasjonsoppføring støtter dermed tilordningen til det avsendende selskapet.

**EU-merknad:** I EU regulerer eIDAS-forordningen elektroniske signaturer og segl. Etter art. 25 avsnitt 2 eIDAS er også bare den kvalifiserte elektroniske signaturen likestilt med den håndskrevne underskriften.

## Anbefaling

- **Funksjonelle postkasser:** OV-sertifikat (SwissSign Organization Validated, DigiCert Secure Email for Organization). Det angir selskapet uten å nevne en person.
- **Personlige postkasser med signatur overfor kunder og partnere:** SV-sertifikat (SwissSign Gold, DigiCert Secure Email for Employee). Det angir personen og selskapet.
- **Bare kryptering:** Kryptografisk er alle typene likeverdige. For bare kryptering er et MV-sertifikat tilstrekkelig.
- **Planlegg kontroller:** Organisasjonskontrollen gjelder i 825 dager, domenekontrollen i 398 dager. Etter en endring av selskapsnavn eller sete samsvarer opplysningene i sertifikatet ikke lenger med registeret; planlegg da nye sertifikater.
- **Kontroller ved mottak:** Policy-OID og UID viser om en signatur er knyttet til et kontrollert selskap. Hvor godt personen er kontrollert, avhenger ved SV av CA-ens prosedyre: Hos SwissSign ligger det en kontroll av identitetsdokument bak, hos DigiCert selskapets bekreftelse.

## Kilder

1.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): Versjon 1.0.17 av 20. november 2026; sertifikattyper og policy-OID-er (avsnitt 1.1), `Enterprise RA` (1.3.2), kontroll av organisasjonen (3.2.3), kontroll av personen (3.2.4), kontroll av fullmakt (3.2.6), gjenbruk av kontrollresultater (4.2.1), gyldighetsperioder (6.3.2), Subject-felt (7.1.4.2) og registreringsordninger (vedlegg A).

2.  [cabforum/smime på GitHub: SBR.md](https://github.com/cabforum/smime/blob/main/SBR.md): Markdown-versjon av S/MIME Baseline Requirements med versjonshistorikk, blant annet EUID fra versjon 1.0.9.

3.  [SwissSign: EUID, How Your Digital Certificates Become Even More Trustworthy](https://www.swisssign.com/en/blog/european-unique-identifier-euid): Produktbetegnelsene Silver og Gold, eksempel `NTRCH-CHE-109.357.012`, UID som registreringsnummer og EUID for tyske selskaper (28.04.2025).

4.  [SwissSign CP/CPS S/MIME](https://repository.swisssign.com/SwissSign_CPS_SMIME.pdf): Kombinert regelverk fra SwissSign, versjon 1.0 av 31.08.2026, erstatter blant annet CPR S/MIME; identifikasjon av personen (3.2.3), fullmakt (3.2.5), gyldighet av dokumentasjon (3.3.1), anskaffelsesveier (4.1.1) og sertifikatprofiler for MV, SV og OV (7.1).

5.  [DigiCert: Secure Email products](https://docs.digicert.com/en/certcentral/order-and-manage-certificates/secure-email-certificates.html): Produktoversikt med tilordning til MV, OV og SV, slutten på generasjonen `Legacy` og de tidligere S/MIME-produktene 10. juli 2025.

6.  [DigiCert: Order your Secure Email for Employee certificate](https://docs.digicert.com/en/certcentral/order-and-manage-certificates/secure-email-certificates/order-your-secure-email-for-employee-certificate.html): Organisasjons- og domenekontroll, bekreftelse av ansatte gjennom organisasjonen uten DigiCerts egen kontroll av personen, profilene `Strict` og `Multipurpose`, gyldighetsperioder.

7.  [Bundesgesetz über die elektronische Signatur (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): Art. 2 (begreper, regulert segl), art. 6 (nøkler), art. 7 (innhold i regulerte sertifikater, UID), art. 8 (kvalifiserte sertifikater) og art. 9 (identitetskontroll).

8.  [Obligationenrecht (OR)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): Art. 14 avsnitt 2bis om likestilling av den kvalifiserte elektroniske signaturen med egenhendig underskrift.

9.  [UID-register](https://www.uid.admin.ch): Oppslag av en UID med navn, sete og status, ført av Bundesamt für Statistik.

10.  [Zefix](https://www.zefix.ch): Sentralt selskapsregister for de kantonale handelsregistrene.

11.  [Zivilprozessordnung (ZPO)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): Art. 157 om fri bevisvurdering.

12.  [Forordning (EU) nr. 910/2014 (eIDAS)](https://eur-lex.europa.eu/legal-content/DE/TXT/?uri=CELEX:32014R0910): Art. 25 avsnitt 2 om likestilling av den kvalifiserte elektroniske signaturen med den håndskrevne underskriften.
