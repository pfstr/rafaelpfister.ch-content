---
title: "Exchange-säkerhetsuppdateringar från september 2026: nio sårbarheter, wrapper-problem åtgärdat, v2 släppt i efterhand"
navTitle: "Exchange SU 09/2026"
description: "September-SU åtgärdar nio sårbarheter i Exchange SE och 2019 (åtta i Exchange 2016), inklusive en spoofing-sårbarhet med CVSS 9.3, och löser wrapper-problemet i hybridmiljöer. Den 2 oktober följde v2 med ytterligare en CVE; dessutom finns tre kända problem med lösningar och en SettingOverride som nu bör tas bort."
date: "2026-10-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "11 min. läsning"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
  - "powershell"
slug: "exchange-sakerhetsuppdateringar-fran-september-2026-nio-sarbarheter-wrapper-problem-atgardat-v2"
translationId: article-53db0c02bc33f9bd
translationOf: exchange-security-updates-september-2026
translationSourceHash: 4b2c59794f87e999c5b89865e0005faa7d5cbb36478f6b8475d5e3f121186a7d
translationModel: gpt-5.6-terra
translatedAt: 2026-10-10T10:01:45.342Z
translationReview: automatic
url: https://rafaelpfister.ch/sv/blog/exchange-sakerhetsuppdateringar-fran-september-2026-nio-sarbarheter-wrapper-problem-atgardat-v2
---

# Exchange-säkerhetsuppdateringar från september 2026: nio sårbarheter, wrapper-problem åtgärdat, v2 släppt i efterhand

Microsoft släppte säkerhetsuppdateringar (SU) för Exchange Server den 8 september 2026. De åtgärdar nio sårbarheter i Exchange SE och Exchange 2019, och åtta i Exchange 2016. Ingen av dem var offentligt känd i förväg, ingen utnyttjas aktivt enligt Security Update Guide, och Microsoft klassar samtliga som *Important* med «Exploitation Less Likely». Det högsta CVSS-värdet, 9.3, ligger dock betydligt över föregående månads. Den här månaden är särskild av tre skäl: SU:t åtgärdar det sedan juni kända problemet med *wrapper-meddelanden* i delade postlådor, det medför tre nya eller kvarstående Known Issues, och den 2 oktober släppte Microsoft en **version 2** som åtgärdar ytterligare en sårbarhet.

## Vilka Exchange-versioner uppdateringen är tillgänglig för

SU:erna från den 8 september 2026 finns för följande versioner:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121608, version 15.2.2562.49; offentligt tillgänglig.
- **Exchange Server 2019 CU15**: KB5121609, version 15.2.1748.51; endast via **Period-2-ESU-programmet**.
- **Exchange Server 2019 CU14**: KB5121610, version 15.2.1544.46; endast via Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5121611, version 15.1.2507.73; endast via Period 2 ESU.

Exchange 2016 och 2019 saknar support. Enligt Microsoft får endast organisationer som är registrerade i Period-2-ESU-programmet SU:erna från maj till oktober 2026. Enligt KB-artiklarna gäller denna behörighet till och med oktober 2026. Därtill kommer trycket från Exchange Online: sedan andra veckan i september stryper och blockerar Exchange Online hybridmailflödet från servrar som ligger före oktober 2025-nivån, med detaljer i [artikeln om transport enforcement](/blog/exchange-online-transport-enforcement-hybrid-server). Exchange Online självt är redan skyddat enligt tillkännagivandet; i hybridmiljöer behöver ändå varje Exchange-server SU:t, liksom datorer med Exchange Management Tools.

Du kan jämföra din egen nivå med översikten över [Exchange-buildnummer](/tools/exchange-builds).

## Översikt över sårbarheterna

| CVE | Typ | CVSS |
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

CVE-2026-55007 berör inte Exchange 2016; Security Update Guide listar endast Exchange SE och 2019 CU14/CU15 för den här CVE:n. En detalj i dokumentationen: CVE-2026-69380 saknas i CVE-listan i KB-artiklarna för de fyra SU:erna från september. Security Update Guide anger dock exakt de fyra septemberbyggena som åtgärd för denna CVE (status 7 oktober 2026).

**[CVE-2026-69356](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69356)** har med CVSS 9.3 det högsta värdet. Enligt Microsoft kan en oautentiserad angripare skicka en manipulerad kalenderinbjudan med en skadlig möteslänk; om mottagaren öppnar mötet och väljer länken för att delta utlöses Cross-Site Scripting. Användarinteraktion krävs alltså, men inget konto i organisationen.

**[CVE-2026-69380](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69380)** (Elevation of Privilege, CVSS 8.1) kräver endast ett konto med låga behörigheter och en tilldelad postlåda. Enligt FAQ:n i Security Update Guide kan en angripare, genom svagheter i kontrollen av förfrågningar och identity tokens, utge sig för att vara en annan användare och ta över alla Exchange-användares postlådor: läsa och skicka e-post samt hämta bilagor. Ett enda komprometterat användarkonto räcker som utgångspunkt.

**[CVE-2026-69641](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69641)** (Elevation of Privilege, CVSS 9.1) leder till samma resultat, övertagande av alla postlådor, men kräver medlemskap i en grupp med höga behörigheter.

Utgångsläget för de båda Remote Code Execution-sårbarheterna skiljer sig åt: [CVE-2026-69355](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69355) (CVSS 8.8) kräver ett autentiserat konto med låga behörigheter, medan [CVE-2026-55007](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-55007) (CVSS 8.1) kan utlösas utan inloggning via en manipulerad Visio-bilaga, men enligt Microsoft förutsätter varaktigt ont om arbetsminne på målsystemet. De övriga fyra sårbarheterna: CVE-2026-69378 (DoS genom okontrollerad rekursion, utan inloggning), CVE-2026-69361 (SSRF, där servern skickar HTTP-förfrågningar till interna system eller loopback-system), CVE-2026-69375 (en autentiserad angripare kan ersätta filinnehåll) och CVE-2026-69382 (exponering av autentiseringsuppgifter genom en svag krypteringsalgoritm, kräver en redan stulen autentiseringscookie).

## Version 2 från 2 oktober: CVE-2026-96940 tillagd

Den 2 oktober 2026 släppte Microsoft «Version 2» av september-SU:erna. Enligt tillkännagivandet är den enda skillnaden från den första versionen den ytterligare åtgärden för **[CVE-2026-96940](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940). Denna Elevation-of-Privilege-sårbarhet har CVSS 8.8, är varken offentligt känd eller utnyttjad, men är den enda av de tio CVE:erna som Microsoft bedömer som **«Exploitation More Likely»**. En autentiserad angripare kan därmed komma åt andra postlådor i samma organisation och läsa e-post inklusive bilagor. Exchange Online är redan korrigerat på serversidan.

| Version | KB | Build v2 |
| --- | --- | --- |
| Exchange SE RTM | KB5129955 | 15.2.2562.53 |
| Exchange 2019 CU15 | KB5129956 | 15.2.1748.53 |
| Exchange 2019 CU14 | KB5129957 | 15.2.1544.48 |
| Exchange 2016 CU23 | KB5129958 | 15.1.2507.75 |

I praktiken innebär det: korrigeringen för CVE-2026-96940 finns endast i v2-byggena. Servrar där SU:t från 8 september redan körs behöver dessutom v2. Den som patchar först nu kan installera v2 direkt, eftersom SU:er är kumulativa. KB-artiklarna säger inte uttryckligen om servrar med det första september-SU:t måste installera v2; eftersom den nya CVE:n endast åtgärdas med v2 är det den rimliga tolkningen.

## Wrapper-problem åtgärdat: ta nu bort SettingOverride

Det sedan juni-SU:t kända problemet där *wrapper-meddelanden* visas i inkorgen för delade postlådor i hybridmiljöer är åtgärdat med september-SU:t i samtliga fyra versioner. I [augustiartikeln](/blog/exchange-security-updates-august-2026) stod det fortfarande att SettingOverride som dokumenterats som en lösning kan ligga kvar. Efter installationen av september-SU:t gäller motsatsen: Microsoft rekommenderar i den tillhörande supportartikeln att kontrollera och ta bort override-inställningen.

```powershell
Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Kommando | Effekt |
|---|---|
| `Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Kontrollerar om workaround-override-inställningen är angiven i organisationen. |
| `Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Tar bort override-inställningen när september-SU:t är installerat. |

</details>

Om det första kommandot rapporterar att objektet `DisableBlockSharedAndUserMailboxHeaders` inte hittades behöver inget mer göras enligt Microsoft.

För Exchange SE åtgärdar SU:t dessutom ett fel i hybrid-fri/upptagen-förfrågningar via Microsoft Graph: lokala användare såg upptagettiderna för Exchange Online-postlådor förskjutna med sin egen UTC-förskjutning, utan felmeddelande.

## Kända problem

**Publicerade kalendrar (.ics) ger HTTP 500 till kalenderappar.** Problemet finns sedan augusti-SU:t (Exchange SE från build 15.2.2562.46 samt Exchange 2019 och 2016) och är inte åtgärdat vare sig i september-SU:t eller v2. Prenumerationer på anonymt publicerade kalendrar uppdateras inte längre; samma URL fungerar i webbläsaren. Enligt Microsoft är orsaken att Exchange identifierar klienter via User-Agent, och kalenderappar utan webbläsaridentifiering hamnar i en kodväg som augusti-SU:t har inaktiverat. Som lösning beskriver Microsoft en URL-rewrite-regel på webbplatsen «Exchange Back End» i IIS, som för .ics-förfrågningar under `/owa/calendar/` lägger till parametern `layout=premium`; IIS-modulen URL Rewrite måste vara installerad. De exakta stegen (via IIS Manager eller direkt i `applicationHost.config`) finns i [supportartikeln KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672). Microsoft anger inget datum för åtgärden (status 7 oktober 2026).

**Fritt/upptaget för delegerade postlådor i hybridmiljöer (endast Exchange SE).** Om tillgänglighetsförfrågan uteslutande är konfigurerad via Graph API misslyckas förfrågningar för Exchange Online-postlådor via delegerad lokal åtkomst. Outlook rapporterar «Your server location could not be determined», OWA visar «No information», och i EWS-loggarna visas `(403) Forbidden`. Den dokumenterade lösningen skickar förfrågningarna via EWS igen i stället för Graph:

```powershell
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=False"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-Identity EnableRouteThroughMSGraphFeature` | Override-inställningen som styr vidarebefordran av tillgänglighetsförfrågningar via Microsoft Graph. |
| `-Parameters "Enabled=False"` | Stänger av Graph-sökvägen så att förfrågningarna åter går via EWS. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Riktar diagnostikanropet till topologitjänsten. |
| `-Component VariantConfiguration -Argument Refresh` | Läser in Variant Configuration på nytt så att override-inställningen tillämpas utan väntan. |

</details>

I tillkännagivandet om v2 listar Microsoft problemet bland de åtgärdade problemen. Den som har satt lösningen bör efter installation av v2 kontrollera i [supportartikeln KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) om den bör återställas; någon instruktion för detta fanns där ännu inte den 7 oktober 2026.

**ContentEngine-deadlock på grund av saknade koreanska WordBreaker-regelfiler (endast Exchange SE).** Med september-SU:t (build 15.2.2562.49 och v2-build 15.2.2562.53) installeras inte regelfilerna för den uppdaterade koreanska WordBreaker. Följderna är saknade sökresultat, fördröjd e-postleverans, Outlook- eller MAPI-klienter som hänger sig eller tappar anslutningen. [KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098) anger miljöer som behandlar e-post med koreanskspråkigt innehåll som berörda. Det beror inte på språket hos de egna användarna: meddelanden med koreansk text når även organisationer utan koreansk korrespondens, exempelvis som skräppost eller nyhetsbrev. Schweiziska miljöer är därför inte undantagna på förhand. Microsoft undersöker fortfarande problemet; kontroll och lösning finns i nästa avsnitt.

## Lösning för WordBreaker-deadlock

Lösningen från KB5130098 lägger till två regelfiler från paketet SQL Server 2025 Express RTM. Microsoft kopplar den till villkor: den ska inte tillämpas enbart på grund av symptomen, utan endast på servrar som uppfyller samtliga kontrollkriterier och där **båda** filerna saknas. [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) kontrollerar inte dessa filer; version v26.10.02 från 2 oktober 2026 innehåller ingen sådan kontroll (status 9 oktober 2026). Kontroll sker därför manuellt.

### 1. Kontrollera servern

Följande skript ändrar ingenting i systemet. Det läser installationssökvägen från miljövariabeln `ExchangeInstallPath` och skriver ut de värden som KB5130098 anger som förutsättning.

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
<summary>Förklaring av alternativ</summary>

| Rad | Effekt |
|---|---|
| `Join-Path $env:ExchangeInstallPath 'Bin\Search\Ceres\Native'` | Skapar sökvägen till katalogen där regelfilerna ska placeras. |
| `Get-Item ...\ExSetup.exe` | Läser filen vars version anger det installerade Exchange-bygget. |
| `VersionInfo.FileVersion` | Filversion från resurserna i EXE-filen respektive DLL-filen. |
| `Get-FileHash -LiteralPath ... -Algorithm SHA256` | Beräknar SHA256-hashen för den installerade `korwbrkr.dll`. |
| `Test-Path ...` | Returnerar `True` om respektive regelfil redan finns. |

</details>

Lösningen är endast tillåten om alla värden stämmer:

| Värde | Förväntat |
| --- | --- |
| `ExSetup` | 15.02.2562.049 eller 15.02.2562.053 |
| `KorwbrkrVersion` | 16.0.5194.1000 |
| `KorwbrkrBytes` | 326544 |
| `KorwbrkrSha256` | 1C6BD8E144BA677EBCC83323AE59DB3881918170F9B3A5189B44611558B92C61 |
| `KoTokenRule`, `KoComplexRule` | båda `False` |

Om build, version eller DLL-hash avviker hänvisar Microsoft till supporten. Om någon av de två regelfilerna redan finns ska du avbryta och inte skriva över något.

### 2. Hämta regelfilerna

Filerna kommer från `SQLEXPR_x64_ENU.exe`, det engelska x64-paketet för SQL Server 2025 Express RTM (version 17.0.1000.7, 748772024 byte). Microsoft rekommenderar att detta steg utförs på en management-workstation, inte på Exchange-servern. SQL Server installeras inte: paketet packas bara upp och Full-Text-MSI:n extraheras endast administrativt. Kontrollera först nedladdningens signatur och hash:

```powershell
Get-AuthenticodeSignature .\SQLEXPR_x64_ENU.exe |
    Select-Object Status, SignerCertificate
Get-FileHash .\SQLEXPR_x64_ENU.exe -Algorithm SHA256
```

Förväntat är statusen `Valid` med ett Microsoft-certifikat och SHA256-värdet `74AA90C11202A5524E769B9BC22531BAEF22D91E9B2D2E8C3CB99E89A65C5297`. Packa sedan upp och extrahera:

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
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `/q` | Packar upp utan dialogrutor. |
| `/x:<Pfad>` | Extract-läge för SQL-paketet: packar upp installationsmediet i mappen utan att starta SQL Setup. |
| `-Wait -PassThru` | Väntar tills processen är klar och returnerar processobjektet så att exitkoden kan läsas. |
| `msiexec /a <MSI>` | Administrativ installation: lägger ut MSI:ns innehåll som filer utan att installera eller registrera något. |
| `TARGETDIR=<Pfad>` | Målkatalog för de extraherade filerna. |
| `/qn` | Utan användargränssnitt. |
| `/L*V <Datei>` | Utförlig logg som visar orsaken vid ett fel. |

</details>

Båda exitkoderna måste vara `0`. Om extraheringen misslyckas ska du spara `extract.log` och avbryta. Kontrollera därefter de båda filernas storlek och hash:

```powershell
Get-ChildItem "$work\Files" -Recurse -Include 'ko.token.rule.bin', 'ko.complex.rule.bin' |
    Select-Object FullName, Length,
        @{ n = 'SHA256'; e = { (Get-FileHash -LiteralPath $_.FullName -Algorithm SHA256).Hash } }
```

| Fil | Byte | SHA256 |
| --- | --- | --- |
| `ko.token.rule.bin` | 56132 | 8F2BD853593913EB8F73DCD4FCAC4216F216A0FF76A4569DF071BE3C36773010 |
| `ko.complex.rule.bin` | 717792 | 0390D1E9A76EF33283025CF8F164430E311584B9535949C4EA1A74B6BB107B87 |

Filerna finns i det extraherade trädet under `...\MSSQL\Binn\ftcomponents\wordbreakers`. Överför endast dessa två filer till en staging-mapp på Exchange-servern, inga SQL-medier och uttryckligen inte `korwbrkr.dll` från SQL-paketet.

### 3. Kopiera filer

```powershell
$native  = Join-Path $env:ExchangeInstallPath 'Bin\Search\Ceres\Native'
$staging = 'C:\Temp\KoStaging'
foreach ($f in 'ko.token.rule.bin', 'ko.complex.rule.bin') {
    Copy-Item -LiteralPath (Join-Path $staging $f) -Destination $native
    Get-FileHash -LiteralPath (Join-Path $native $f) -Algorithm SHA256
}
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `Copy-Item -LiteralPath <Quelle>` | Kopierar filen från staging-mappen; sökvägen tolkas utan jokertecken. |
| `-Destination $native` | Målkatalog bredvid den installerade `korwbrkr.dll`. |
| `Get-FileHash ... -Algorithm SHA256` | Kontrollerar åter hashen på målplatsen mot tabellen ovan. |

</details>

Använd `Copy-Item`, inte `Move-Item`. En kopierad fil får målkatalogens ärvda behörigheter, medan en flyttad fil behåller behörigheterna från sin ursprungliga mapp. Filerna behöver de vanliga läsrättigheterna för `Native`. Ersätt inga andra filer i denna katalog.

### 4. Starta om Search Host Controller

```powershell
Restart-Service -Name HostControllerService
```

Omstarten avbryter tillfälligt sökning och Content Processing på servern; Microsoft avsätter ett underhållsfönster för detta. Själva servern behöver inte startas om. Avsluta inte Exchange-processer med tvång: om tjänsten inte startar korrekt är nästa steg enligt Microsoft att kontakta supporten.

### 5. Kontroll

Host Controller startar själv processen `NodeRunner.exe` för ContentEngine; starta den inte manuellt. Denna fråga visar om den körs:

```powershell
Get-CimInstance Win32_Process -Filter "Name = 'NodeRunner.exe'" |
    Where-Object CommandLine -like '*ContentEngineNode1*' |
    Select-Object ProcessId, CreationDate
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `Get-CimInstance Win32_Process -Filter "Name = 'NodeRunner.exe'"` | Läser alla NodeRunner-processer med kommandorad. |
| `Where-Object CommandLine -like '*ContentEngineNode1*'` | Filtrerar på ContentEngine-processen. |
| `Select-Object ProcessId, CreationDate` | Visar process-ID och starttid; starttiden måste ligga efter tjänstens omstart. |

</details>

Om ingen process visas eller om den avslutas upprepade gånger ska du inte bearbeta fler servrar. Annars kontrollerar du i Outlook på webben med en postlåda vars aktiva databaskopia finns på den här servern om ny e-post levereras och hittas, även e-post med koreansk text. Kontrollera dessutom tjänsten som ursprungligen stördes (leverans, Outlook-anslutning). Övervaka separat eftersläpningen i sökindexet: att nya e-postmeddelanden hittas bevisar inte att äldre objekt redan har bearbetats. Bearbeta ytterligare servrar en i taget och först efter framgångsrik kontroll av den föregående.

## Installation och efterarbete

Microsoft rekommenderar det välkända förfarandet: inventera med [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), fastställ vägen med [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) om nivån är föråldrad, installera SU:t, starta om servern och kontrollera att alla Exchange-tjänster körs. Health Checker visar därefter även om SU:t installerats korrekt. Security Update Guide anger att uppdateringarna kräver omstart.

Efter installationen finns tre efterarbeten:

1. Ta bort Wrapper-SettingOverride `DisableBlockSharedAndUserMailboxHeaders` om den är angiven (se ovan).

2. Kontrollera på Exchange SE med kontrollskriptet från [WordBreaker-avsnittet](#workaround-für-den-wordbreaker-deadlock) om de två regelfilerna saknas, och kontrollera om fritt/upptaget-problemet uppstår. Tillämpa lösningarna endast om respektive villkor är uppfyllda.

3. För publicerade kalendrar med externa prenumeranter ska URL-rewrite-regeln från KB5126672 konfigureras, om detta inte redan gjorts sedan augusti-SU:t.

Från juli kvarstår dessutom kontrollen av om CVE-2026-42897-mitigeringen (M2.1.0) fortfarande är aktiv; hur den tas bort finns i [artikeln om juli-SU:t](/blog/exchange-security-updates-juli-2026).

## Rekommenderat tillvägagångssätt

Installera v2-byggena från 2 oktober direkt på alla Exchange-servrar och datorer med Management Tools; servrar med SU:t från 8 september behöver dessutom v2 för CVE-2026-96940. Spoofing-sårbarheten med CVSS 9.3 och postlådeövertagandet via ett användarkonto med låga behörigheter (CVE-2026-69380) är skäl nog att inte vänta till nästa patchdag. Ta därefter bort wrapper-override-inställningen, kontrollera de tre Known Issues och kör Health Checker. Health Checker rapporterar inte de saknade WordBreaker-filerna; för det behövs kontrollskriptet från WordBreaker-avsnittet. ESU-programmet för Exchange 2016 och 2019 upphör i oktober 2026; migreringen till Exchange SE kan inte skjutas upp längre.

## Källor

1.  [Released: September 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-september-2026-exchange-server-security-updates/4554411): Officiellt release-tillkännagivande med versioner som stöds, ESU-information, Known Issues, åtgärdade problem och installationsförfarande (granskat via RSS-flödet för Exchange Team Blogs).

2.  [Released: September 2026 V2 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-september-2026-v2-exchange-server-security-updates/ba-p/4561718): Tillkännagivande om v2 från den 2 oktober 2026; enda skillnaden är CVE-2026-96940.

3.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM: September 8, 2026 (KB5121608) – Microsoft Support](https://support.microsoft.com/help/5121608): CVE-lista, åtgärdade problem och de tre Known Issues för Exchange SE.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU15: September 8, 2026 (KB5121609) – Microsoft Support](https://support.microsoft.com/help/5121609): KB-artikel för Exchange 2019 CU15.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU14: September 8, 2026 (KB5121610) – Microsoft Support](https://support.microsoft.com/help/5121610): KB-artikel för Exchange 2019 CU14.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: September 8, 2026 (KB5121611) – Microsoft Support](https://support.microsoft.com/help/5121611): KB-artikel för Exchange 2016 CU23, utan CVE-2026-55007.

7.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Typ, CVSS, allvarlighetsgrad, bedömning av utnyttjande och FAQ för samtliga nio CVE:er från september samt CVE-2026-96940, inklusive berörda builds per CVE.

8.  [Description of version 2 of the security update for Microsoft Exchange Server Subscription Edition RTM October 2, 2026 (KB5129955) – Microsoft Support](https://support.microsoft.com/help/5129955): KB-artikel om v2 för Exchange SE; v2-artiklarna för 2019 och 2016 är KB5129956 till KB5129958.

9.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Buildnummer för september-SU:erna och v2 från 2 oktober 2026.

10. [Wrapper messages appear in shared mailbox in hybrid environments after installing the June 2026 Security Update – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Åtgärd i september-SU:t och instruktioner för att ta bort SettingOverride.

11. [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Orsak och URL-rewrite-lösning.

12. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Symptom och SettingOverride-lösning för Exchange SE.

13. [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Berörda SE-builds, kontrollkriterier, förväntade filstorlekar och hashar, hämtning från SQL Server 2025 Express, omstart av tjänsten och kontroll.

14. [Hybrid free/busy through Microsoft Graph incorrectly shifts busy times by requester timezone – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5125804): Tidszonsfelet i Exchange SE som åtgärdades med september-SU:t.

15. [Neue Sicherheitsupdates für Exchange Server (September 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-september-2026/): Tyskspråkig genomgång av de nio CVE:erna med CVSS-värden och builds.

16. [Exchange Server: Sicherheitsupdates 8. September 2026 – Borns Tech and Windows World](https://borncity.com/blog/?p=329371): Tyskspråkig sammanfattning med information om den senare ersättningsuppdateringen.

17. [microsoft/CSS-Exchange – GitHub](https://github.com/microsoft/CSS-Exchange): Källkod för Exchange Health Checker; release v26.10.02 från 2 oktober 2026 utan kontroll av WordBreaker-filerna (status 9 oktober 2026).
