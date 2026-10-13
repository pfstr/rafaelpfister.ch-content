---
title: "Exchange-sikkerhetsoppdateringer for september 2026: ni sårbarheter, Wrapper-problem løst, v2 lagt til"
navTitle: "Exchange SU 09/2026"
description: "September-SU-en lukker ni sårbarheter i Exchange SE og 2019 (åtte i Exchange 2016), blant annet en spoofing-sårbarhet med CVSS 9.3, og løser Wrapper-problemet i hybridmiljøer. 2. oktober fulgte en v2 med en ekstra CVE; i tillegg kommer tre kjente problemer med løsninger og en SettingOverride som nå bør fjernes."
date: "2026-10-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "11 min lesetid"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
  - "powershell"
slug: "exchange-sikkerhetsoppdateringer-fra-september-2026-ni-sarbarheter-wrapper-problem-lost-v2"
translationId: article-53db0c02bc33f9bd
translationOf: exchange-security-updates-september-2026
translationSourceHash: 4b2c59794f87e999c5b89865e0005faa7d5cbb36478f6b8475d5e3f121186a7d
translationModel: gpt-5.6-terra
translatedAt: 2026-10-10T10:02:35.909Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/exchange-sikkerhetsoppdateringer-fra-september-2026-ni-sarbarheter-wrapper-problem-lost-v2
---

# Exchange-sikkerhetsoppdateringer for september 2026: ni sårbarheter, Wrapper-problem løst, v2 lagt til

Microsoft publiserte 8. september 2026 sikkerhetsoppdateringer (SU-er) for Exchange Server. De lukker ni sårbarheter i Exchange SE og Exchange 2019, og åtte i Exchange 2016. Ingen av dem var offentlig kjent på forhånd, ingen utnyttes aktivt ifølge Security Update Guide, og Microsoft klassifiserer alle som *Important* med «Exploitation Less Likely». Den høyeste CVSS-verdien er imidlertid 9.3, betydelig høyere enn måneden før. Denne måneden skiller seg ut av tre grunner: SU-en løser problemet med *Wrapper-meldinger* i delte postbokser, som har vært åpent siden juni, den medfører tre nye eller fortsatt eksisterende kjente problemer, og 2. oktober publiserte Microsoft en **versjon 2** som lukker en ekstra sårbarhet.

## Hvilke Exchange-versjoner oppdateringen er tilgjengelig for

SU-ene fra 8. september 2026 er tilgjengelige for følgende versjoner:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121608, build 15.2.2562.49; offentlig tilgjengelig.
- **Exchange Server 2019 CU15**: KB5121609, build 15.2.1748.51; kun via **Period-2-ESU-programmet**.
- **Exchange Server 2019 CU14**: KB5121610, build 15.2.1544.46; kun via Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5121611, build 15.1.2507.73; kun via Period 2 ESU.

Exchange 2016 og 2019 er ute av støtte. Ifølge Microsoft er SU-ene fra mai til oktober 2026 kun tilgjengelige for organisasjoner som er registrert i Period-2-ESU-programmet. Ifølge KB-artiklene gjelder denne retten til oktober 2026. I tillegg kommer presset fra Exchange Online: Siden den andre uken i september begrenser og blokkerer Exchange Online hybrid e-postflyt fra servere under oktober 2025-nivået, med detaljer i [artikkelen om Transport Enforcement](/blog/exchange-online-transport-enforcement-hybrid-server). Exchange Online er ifølge kunngjøringen allerede beskyttet; i hybridmiljøer trenger likevel hver Exchange-server SU-en, inkludert maskiner med Exchange Management Tools.

Du kan sammenligne din egen versjon med oversikten over [Exchange-buildnumre](/tools/exchange-builds).

## Oversikt over sårbarhetene

| CVE | Type | CVSS |
| --- | --- | --- |
| CVE-2026-69356 | Spoofing (Cross-Site Scripting) | 9.3 |
| CVE-2026-69641 | Elevation of Privilege | 9.1 |
| CVE-2026-69355 | Remote Code Execution | 8.8 |
| CVE-2026-55007 | Remote Code Execution | 8.1 |
| CVE-2026-69380 | Elevation of Privilege | 8.1 |
| CVE-2026-69378 | Denial of Service | 7.5 |
| CVE-2026-69361 | Spoofing (Server-Side Request Forgery) | 6.5 |
| CVE-2026-69375 | Tampering | 6.5 |
| CVE-2026-69382 | Information Disclosure | 5.9 |

CVE-2026-55007 gjelder ikke Exchange 2016; Security Update Guide angir kun Exchange SE og 2019 CU14/CU15 for denne CVE-en. En detalj ved dokumentasjonen: CVE-2026-69380 mangler i CVE-listen i KB-artiklene for de fire september-SU-ene. Security Update Guide angir imidlertid nøyaktig de fire september-buildene som løsning for denne CVE-en (per 7. oktober 2026).

**[CVE-2026-69356](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69356) har med CVSS 9.3 den høyeste verdien. Ifølge Microsoft kan en ikke-autentisert angriper sende en manipulert kalenderinvitasjon med en ondsinnet møtelenke; når mottakeren åpner møtet og velger lenken for å delta, utløses Cross-Site Scripting. Det kreves altså brukerinteraksjon, men ingen konto i organisasjonen.

**[CVE-2026-69380](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69380) (Elevation of Privilege, CVSS 8.1) krever kun en konto med lave rettigheter og en tildelt postboks. Ifølge FAQ-en i Security Update Guide kan en angriper, gjennom svakheter i kontrollen av forespørsler og identitetstokener, utgi seg for å være en annen bruker og overta postboksene til alle Exchange-brukere: lese og sende e-post samt laste ned vedlegg. Én kompromittert brukerkonto er tilstrekkelig som utgangspunkt.

**[CVE-2026-69641](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69641) (Elevation of Privilege, CVSS 9.1) fører til samme resultat, overtakelse av alle postbokser, men krever medlemskap i en rollegruppe med høye rettigheter.

For de to Remote Code Execution-sårbarhetene er utgangspunktet ulikt: [CVE-2026-69355](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69355) (CVSS 8.8) krever en autentisert konto med lave rettigheter, mens [CVE-2026-55007](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-55007) (CVSS 8.1) kan utløses uten pålogging via et manipulert Visio-vedlegg, men ifølge Microsoft forutsetter vedvarende lite arbeidsminne på målsystemet. De øvrige fire sårbarhetene er: CVE-2026-69378 (DoS gjennom ukontrollert rekursjon, uten pålogging), CVE-2026-69361 (SSRF, der serveren sender HTTP-forespørsler til interne systemer eller loopback-systemer), CVE-2026-69375 (en autentisert angriper kan erstatte filinnhold) og CVE-2026-69382 (avsløring av påloggingsdata via en svak kryptografialgoritme, krever en allerede stjålet autentiseringsinformasjonskapsel).

## Versjon 2 fra 2. oktober: CVE-2026-96940 lagt til

2. oktober 2026 publiserte Microsoft «versjon 2» av september-SU-ene. Ifølge kunngjøringen er den eneste forskjellen fra første versjon en ekstra løsning for **[CVE-2026-96940](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940). Denne Elevation-of-Privilege-sårbarheten har CVSS 8.8, er verken offentlig kjent eller utnyttet, men er den eneste av de ti CVE-ene som Microsoft vurderer som **«Exploitation More Likely»**. En autentisert angriper kan dermed få tilgang til andre postbokser i samme organisasjon og lese e-post med vedlegg. Exchange Online er allerede korrigert på serversiden.

| Versjon | KB | Build v2 |
| --- | --- | --- |
| Exchange SE RTM | KB5129955 | 15.2.2562.53 |
| Exchange 2019 CU15 | KB5129956 | 15.2.1748.53 |
| Exchange 2019 CU14 | KB5129957 | 15.2.1544.48 |
| Exchange 2016 CU23 | KB5129958 | 15.1.2507.75 |

I praksis betyr dette: Løsningen for CVE-2026-96940 finnes bare i v2-buildene. Servere som allerede kjører SU-en fra 8. september, trenger i tillegg v2. De som først oppdaterer nå, kan installere v2 direkte, siden SU-er er kumulative. KB-artiklene sier ikke uttrykkelig om servere med første september-SU må installere v2, men siden den nye CVE-en kun lukkes med v2, er det den nærliggende tolkningen.

## Wrapper-problem løst: Fjern SettingOverride nå

Problemet kjent siden juni-SU-en, der *Wrapper-meldinger* vises i innboksen til delte postbokser i hybridmiljøer, er løst med september-SU-en for alle fire versjoner. I [augustartikkelen](/blog/exchange-security-updates-august-2026) stod det fortsatt at SettingOverride-en dokumentert som løsning kan beholdes. Etter installasjon av september-SU-en gjelder det motsatte: Microsoft anbefaler i den tilhørende støtteartikkelen å kontrollere og fjerne overstyringen.

```powershell
Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Kommando | Virkning |
|---|---|
| `Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Kontrollerer om løsningsoverstyringen er angitt i organisasjonen. |
| `Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Fjerner overstyringen når september-SU-en er installert. |

</details>

Hvis den første kommandoen oppgir at objektet `DisableBlockSharedAndUserMailboxHeaders` ikke ble funnet, er det ifølge Microsoft ikke nødvendig å gjøre noe mer.

For Exchange SE løser SU-en i tillegg en feil ved Hybrid-ledig/opptatt-forespørsler via Microsoft Graph: Lokale brukere så opptatt-tidene til Exchange Online-postbokser forskjøvet med sin egen UTC-forskyvning, uten feilmelding.

## Kjente problemer

**Publiserte kalendere (.ics) gir HTTP 500 til kalenderapper.** Problemet har eksistert siden august-SU-en (Exchange SE fra build 15.2.2562.46 samt Exchange 2019 og 2016), og er heller ikke løst i september-SU-en eller v2. Abonnementer på anonymt publiserte kalendere oppdateres ikke lenger; samme URL fungerer i nettleseren. Ifølge Microsoft er årsaken at Exchange gjenkjenner klienter via User-Agent, og kalenderapper uten nettleseridentifikasjon havner i en kodebane som august-SU-en deaktiverte. Som løsning beskriver Microsoft en URL Rewrite-regel på nettstedet «Exchange Back End» i IIS, som legger parameteren `layout=premium` til .ics-forespørsler under `/owa/calendar/`; IIS-modulen URL Rewrite må være installert. De nøyaktige trinnene (via IIS Manager eller direkte i `applicationHost.config`) finnes i [støtteartikkelen KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672). Microsoft oppgir ikke noen dato for løsningen (per 7. oktober 2026).

**Ledig/opptatt for delegerte postbokser i hybridmiljøer (kun Exchange SE).** Hvis tilgjengelighetsoppslag utelukkende er konfigurert via Graph API, feiler oppslag for Exchange Online-postbokser gjennom delegert lokal tilgang. Outlook rapporterer «Your server location could not be determined», OWA viser «No information», og EWS-loggene inneholder `(403) Forbidden`. Den dokumenterte løsningen sender forespørslene tilbake via EWS i stedet for Graph:

```powershell
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=False"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `-Identity EnableRouteThroughMSGraphFeature` | Overstyringen som styrer videresendingen av tilgjengelighetsforespørsler via Microsoft Graph. |
| `-Parameters "Enabled=False"` | Slår av Graph-banen, slik at forespørslene igjen går via EWS. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Retter diagnosekallet mot topologitjenesten. |
| `-Component VariantConfiguration -Argument Refresh` | Laster Variant Configuration på nytt, slik at overstyringen trer i kraft uten venting. |

</details>

I kunngjøringen for v2 oppgir Microsoft dette problemet som løst. De som har angitt løsningen, bør etter installasjon av v2 kontrollere i [støtteartikkelen KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) om den skal tilbakeføres; per 7. oktober 2026 var det ennå ingen veiledning om dette der.

**ContentEngine-deadlås på grunn av manglende koreanske WordBreaker-regelfiler (kun Exchange SE).** Med september-SU-en (build 15.2.2562.49 og v2-build 15.2.2562.53) installeres ikke regelfilene for den oppdaterte koreanske WordBreaker-en. Følgene er manglende søkeresultater, forsinket e-postlevering, og Outlook- eller MAPI-klienter som henger eller mister forbindelsen. [KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098) angir miljøer som behandler e-post med koreanskspråklig innhold som berørt. Dette avhenger ikke av språket til egne brukere: Meldinger med koreansk tekst når også organisasjoner uten koreansk korrespondanse, for eksempel som spam eller nyhetsbrev. Sveitsiske miljøer er derfor ikke unntatt på forhånd. Microsoft undersøker fortsatt problemet; kontroll og løsning beskrives i neste avsnitt.

## Løsning for WordBreaker-deadlåsen

Løsningen fra KB5130098 legger til to regelfiler fra pakken SQL Server 2025 Express RTM. Microsoft knytter den til betingelser: Den skal ikke brukes bare på grunn av symptomene, men kun på servere som oppfyller alle kontrollkriteriene og mangler **begge** filene. [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) kontrollerer ikke disse filene; versjon v26.10.02 fra 2. oktober 2026 inneholder ingen tilsvarende kontroll (per 9. oktober 2026). Kontroll må derfor utføres manuelt.

### 1. Kontroller serveren

Følgende skript endrer ingenting i systemet. Det leser installasjonsbanen fra miljøvariabelen `ExchangeInstallPath` og viser verdiene som KB5130098 angir som forutsetning.

```powershell
$native  = Join-Path $env:ExchangeInstallPath 'Bin\Search\Ceres\Native'
$exsetup = Get-Item (Join-Path $env:ExchangeInstallPath 'Bin\ExSetup.exe')
$dll     = Get-Item (Join-Path $native 'korwbrkr.dll')
[pscustomobject]@{
    ExSetup         = $exsetup.VersionInfo.FileVersion
    KorwbrkrVersion = $dll.VersionInfo.FileVersion
    KorwbrkrBytes   = $dll.Length
    KorwbrkrSha256  = (Get-FileHash -LiteralPath $dll.FullName -Algorithm SHA256).Hash
    KoTokenRule     = Test-Path (Join-Path $native 'ko.token.rule.bin')
    KoComplexRule   = Test-Path (Join-Path $native 'ko.complex.rule.bin')
}
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Linje | Virkning |
|---|---|
| `Join-Path $env:ExchangeInstallPath 'Bin\Search\Ceres\Native'` | Bygger banen til katalogen der regelfilene skal ligge. |
| `Get-Item ...\ExSetup.exe` | Leser filen der versjonen angir installert Exchange-build. |
| `VersionInfo.FileVersion` | Filversjon fra ressursene i EXE-filen eller DLL-en. |
| `Get-FileHash -LiteralPath ... -Algorithm SHA256` | Beregner SHA256-hashen for den installerte `korwbrkr.dll`. |
| `Test-Path ...` | Returnerer `True` hvis den aktuelle regelfilen allerede finnes. |

</details>

Løsningen er kun tillatt dersom alle verdier stemmer:

| Verdi | Forventet |
| --- | --- |
| `ExSetup` | 15.02.2562.049 eller 15.02.2562.053 |
| `KorwbrkrVersion` | 16.0.5194.1000 |
| `KorwbrkrBytes` | 326544 |
| `KorwbrkrSha256` | 1C6BD8E144BA677EBCC83323AE59DB3881918170F9B3A5189B44611558B92C61 |
| `KoTokenRule`, `KoComplexRule` | begge `False` |

Hvis build, versjon eller hash for DLL-en avviker, henviser Microsoft til støtteapparatet. Hvis én av de to regelfilene allerede finnes, må du avbryte og ikke overskrive noe.

### 2. Skaff regelfilene

Filene kommer fra `SQLEXPR_x64_ENU.exe`, den engelske x64-pakken for SQL Server 2025 Express RTM (versjon 17.0.1000.7, 748772024 byte). Microsoft anbefaler å utføre dette trinnet på en administrasjonsarbeidsstasjon, ikke på Exchange-serveren. SQL Server installeres ikke: Pakken pakkes bare ut, og Full-Text-MSI-en ekstraheres kun administrativt. Kontroller først signaturen og hashen for nedlastingen:

```powershell
Get-AuthenticodeSignature .\SQLEXPR_x64_ENU.exe |
    Select-Object Status, SignerCertificate
Get-FileHash .\SQLEXPR_x64_ENU.exe -Algorithm SHA256
```

Forventet er statusen `Valid` med et Microsoft-sertifikat og SHA256-verdien `74AA90C11202A5524E769B9BC22531BAEF22D91E9B2D2E8C3CB99E89A65C5297`. Pakk deretter ut og ekstraher:

```powershell
$work = 'C:\Temp\KoreanRules'
$p = Start-Process .\SQLEXPR_x64_ENU.exe -ArgumentList '/q', "/x:$work\Media" -Wait -PassThru
$p.ExitCode
$msi = "$work\Media\x64\Setup\SQL_FULLTEXT.MSI"
$msiArgs = '/a', $msi, "TARGETDIR=$work\Files", '/qn', '/L*V', "$work\extract.log"
$p = Start-Process msiexec.exe -ArgumentList $msiArgs -Wait -PassThru
$p.ExitCode
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `/q` | Pakker ut uten dialogbokser. |
| `/x:<Pfad>` | Ekstraheringsmodus for SQL-pakken: pakker ut installasjonsmediene i mappen uten å starte SQL Setup. |
| `-Wait -PassThru` | Venter på at prosessen avsluttes og returnerer prosessobjektet, slik at avslutningskoden kan leses. |
| `msiexec /a <MSI>` | Administrativ installasjon: legger innholdet i MSI-en som filer uten å installere eller registrere noe. |
| `TARGETDIR=<Pfad>` | Målkatalog for de ekstraherte filene. |
| `/qn` | Uten brukergrensesnitt. |
| `/L*V <Datei>` | Detaljert logg som viser årsaken ved feil. |

</details>

Begge avslutningskodene må være `0`. Hvis ekstraheringen feiler, tar du vare på `extract.log` og avbryter. Kontroller deretter størrelse og hash for de to filene:

```powershell
Get-ChildItem "$work\Files" -Recurse -Include 'ko.token.rule.bin', 'ko.complex.rule.bin' |
    Select-Object FullName, Length,
        @{ n = 'SHA256'; e = { (Get-FileHash -LiteralPath $_.FullName -Algorithm SHA256).Hash } }
```

| Fil | Byte | SHA256 |
| --- | --- | --- |
| `ko.token.rule.bin` | 56132 | 8F2BD853593913EB8F73DCD4FCAC4216F216A0FF76A4569DF071BE3C36773010 |
| `ko.complex.rule.bin` | 717792 | 0390D1E9A76EF33283025CF8F164430E311584B9535949C4EA1A74B6BB107B87 |

Filene ligger i det ekstraherte treet under `...\MSSQL\Binn\ftcomponents\wordbreakers`. Overfør bare disse to filene til en staging-mappe på Exchange-serveren, ingen SQL-medier og uttrykkelig ikke `korwbrkr.dll` fra SQL-pakken.

### 3. Kopier filene

```powershell
$native  = Join-Path $env:ExchangeInstallPath 'Bin\Search\Ceres\Native'
$staging = 'C:\Temp\KoStaging'
foreach ($f in 'ko.token.rule.bin', 'ko.complex.rule.bin') {
    Copy-Item -LiteralPath (Join-Path $staging $f) -Destination $native
    Get-FileHash -LiteralPath (Join-Path $native $f) -Algorithm SHA256
}
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `Copy-Item -LiteralPath <Quelle>` | Kopierer filen fra staging-mappen; banen evalueres uten jokertegn. |
| `-Destination $native` | Målkatalog ved siden av den installerte `korwbrkr.dll`. |
| `Get-FileHash ... -Algorithm SHA256` | Kontrollerer hashen på målplasseringen på nytt mot tabellen over. |

</details>

Bruk `Copy-Item`, ikke `Move-Item`. En kopiert fil får de arvede tillatelsene til målkatalogen, mens en flyttet fil beholder tillatelsene fra opprinnelsesmappen. Filene trenger de vanlige leserettighetene til `Native`. Ikke erstatt andre filer i denne katalogen.

### 4. Start Search Host Controller på nytt

```powershell
Restart-Service -Name HostControllerService
```

Omstarten avbryter søk og innholdsbehandling på serveren midlertidig; Microsoft anbefaler et vedlikeholdsvindu for dette. Selve serveren trenger ikke omstart. Ikke avslutt Exchange-prosesser med tvang: Hvis tjenesten ikke starter rent, er neste steg ifølge Microsoft å kontakte støtteapparatet.

### 5. Kontroll

Host Controller starter selv prosessen `NodeRunner.exe` for ContentEngine; ikke start den manuelt. Denne forespørselen viser om den kjører:

```powershell
Get-CimInstance Win32_Process -Filter "Name = 'NodeRunner.exe'" |
    Where-Object CommandLine -like '*ContentEngineNode1*' |
    Select-Object ProcessId, CreationDate
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `Get-CimInstance Win32_Process -Filter "Name = 'NodeRunner.exe'"` | Leser alle NodeRunner-prosesser med kommandolinje. |
| `Where-Object CommandLine -like '*ContentEngineNode1*'` | Filtrerer etter prosessen til ContentEngine. |
| `Select-Object ProcessId, CreationDate` | Viser prosess-ID og starttidspunkt; starttidspunktet må ligge etter tjenesteomstarten. |

</details>

Hvis ingen prosess vises eller den avsluttes gjentatte ganger, må du ikke behandle flere servere. Ellers kontrollerer du i Outlook på nettet med en postboks der den aktive databasekopien ligger på denne serveren, om ny e-post leveres og kan finnes, også e-post med koreansk tekst. Kontroller også tjenesten som opprinnelig var forstyrret (levering, Outlook-tilkobling). Overvåk etterslepet i søkeindeksen separat: At ny e-post kan finnes, beviser ikke at eldre elementer allerede er behandlet. Behandle ytterligere servere enkeltvis og først etter vellykket kontroll av den forrige.

## Installasjon og oppfølging

Microsoft anbefaler den kjente fremgangsmåten: inventariser med [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), finn oppdateringsbanen med [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) ved utdatert versjon, installer SU-en, start serveren på nytt og kontroller at alle Exchange-tjenestene kjører. Health Checker viser deretter også om SU-en er installert riktig. Security Update Guide angir at oppdateringene krever omstart.

Etter installasjonen er det tre oppfølgingsoppgaver:

1. Fjern Wrapper-SettingOverride-en `DisableBlockSharedAndUserMailboxHeaders` hvis den er angitt (se over).

2. På Exchange SE kontrollerer du med kontrollskriptet fra [WordBreaker-avsnittet](#workaround-für-den-wordbreaker-deadlock) om de to regelfilene mangler, og undersøker om ledig/opptatt-problemet oppstår. Bruk bare løsningene når de respektive vilkårene er oppfylt.

3. For publiserte kalendere med eksterne abonnenter konfigurerer du URL Rewrite-regelen fra KB5126672, hvis dette ikke allerede er gjort siden august-SU-en.

Fra juli gjenstår dessuten kontrollen av om CVE-2026-42897-avbøtningen (M2.1.0) fortsatt er aktiv; hvordan den fjernes står i [artikkelen om juli-SU-en](/blog/exchange-security-updates-juli-2026).

## Anbefalt fremgangsmåte

Installer v2-buildene fra 2. oktober direkte på alle Exchange-servere og maskiner med Management Tools; servere med SU-en fra 8. september trenger i tillegg v2 for CVE-2026-96940. Spoofing-sårbarheten med CVSS 9.3 og overtakelse av postbokser gjennom en brukerkonto med lave rettigheter (CVE-2026-69380) er god nok grunn til ikke å vente til neste patchdag. Fjern deretter Wrapper-overstyringen, kontroller de tre kjente problemene og kjør Health Checker. Health Checker varsler ikke om manglende WordBreaker-filer; for det trengs kontrollskriptet fra WordBreaker-avsnittet. ESU-programmet for Exchange 2016 og 2019 utløper i oktober 2026; migreringen til Exchange SE kan ikke utsettes videre.

## Kilder

1.  [Released: September 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-september-2026-exchange-server-security-updates/4554411): Offisiell kunngjøring med støttede versjoner, ESU-merknad, kjente problemer, løste problemer og installasjonsprosess (åpnet via RSS-feeden til Exchange Team Blog).

2.  [Released: September 2026 V2 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-september-2026-v2-exchange-server-security-updates/ba-p/4561718): Kunngjøring av v2 fra 2. oktober 2026; eneste forskjell er CVE-2026-96940.

3.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM: September 8, 2026 (KB5121608) – Microsoft Support](https://support.microsoft.com/help/5121608): CVE-liste, løste problemer og de tre kjente problemene for Exchange SE.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU15: September 8, 2026 (KB5121609) – Microsoft Support](https://support.microsoft.com/help/5121609): KB-artikkel for Exchange 2019 CU15.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU14: September 8, 2026 (KB5121610) – Microsoft Support](https://support.microsoft.com/help/5121610): KB-artikkel for Exchange 2019 CU14.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: September 8, 2026 (KB5121611) – Microsoft Support](https://support.microsoft.com/help/5121611): KB-artikkel for Exchange 2016 CU23, uten CVE-2026-55007.

7.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Type, CVSS, alvorlighetsgrad, vurdering av utnyttelse og FAQ for alle ni september-CVE-er og CVE-2026-96940, inkludert berørte build per CVE.

8.  [Description of version 2 of the security update for Microsoft Exchange Server Subscription Edition RTM October 2, 2026 (KB5129955) – Microsoft Support](https://support.microsoft.com/help/5129955): KB-artikkel om v2 for Exchange SE; v2-artiklene for 2019 og 2016 er KB5129956 til KB5129958.

9.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Buildnumre for september-SU-ene og v2 fra 2. oktober 2026.

10. [Wrapper messages appear in shared mailbox in hybrid environments after installing the June 2026 Security Update – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Løsning i september-SU-en og veiledning for å fjerne SettingOverride-en.

11. [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Årsak og URL Rewrite-løsning.

12. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Symptomer og SettingOverride-løsning for Exchange SE.

13. [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Berørte SE-build, kontrollkriterier, forventede filstørrelser og hasher, anskaffelse fra SQL Server 2025 Express, tjenesteomstart og kontroll.

14. [Hybrid free/busy through Microsoft Graph incorrectly shifts busy times by requester timezone – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5125804): Tidssonefeilen i Exchange SE som ble løst med september-SU-en.

15. [Neue Sicherheitsupdates für Exchange Server (September 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-september-2026/): Tyskspråklig oversikt over de ni CVE-ene med CVSS-verdier og build.

16. [Exchange Server: Sicherheitsupdates 8. September 2026 – Borns Tech and Windows World](https://borncity.com/blog/?p=329371): Tyskspråklig sammendrag med informasjon om den senere erstatningsoppdateringen.

17. [microsoft/CSS-Exchange – GitHub](https://github.com/microsoft/CSS-Exchange): Kildekoden til Exchange Health Checker; utgivelse v26.10.02 fra 2. oktober 2026 uten kontroll av WordBreaker-filene (per 9. oktober 2026).
