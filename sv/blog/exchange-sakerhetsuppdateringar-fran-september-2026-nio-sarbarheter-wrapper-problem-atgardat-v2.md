---
title: "Exchange-säkerhetsuppdateringar från september 2026: nio sårbarheter, wrapper-problem åtgärdat, v2 tillagd"
navTitle: "Exchange SU 09/2026"
description: "September-SU åtgärdar nio sårbarheter i Exchange SE och 2019 (åtta i Exchange 2016), däribland en spoofing-sårbarhet med CVSS 9.3, och löser wrapper-problemet i hybridmiljöer. Den 2 oktober följde en v2 med ytterligare en CVE; dessutom finns tre Known Issues med lösningar och en SettingOverride som nu ska tas bort."
date: "2026-10-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "11 min lästid"
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
translationSourceHash: 1f9a8027af593738b429df5a1b20adab97301c20761ef0cfe02e2691a12e880a
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T15:47:32.157Z
translationReview: automatic
url: https://rafaelpfister.ch/sv/blog/exchange-sakerhetsuppdateringar-fran-september-2026-nio-sarbarheter-wrapper-problem-atgardat-v2
---

# Exchange-säkerhetsuppdateringar från september 2026: nio sårbarheter, wrapper-problem åtgärdat, v2 tillagd

Microsoft publicerade säkerhetsuppdateringar (SU) för Exchange den 8 september 2026. De åtgärdar nio sårbarheter i Exchange SE och Exchange 2019, och åtta i Exchange 2016. Ingen av dem var offentligt känd i förväg, ingen utnyttjas aktivt enligt Security Update Guide, och Microsoft klassificerar samtliga som *Important* med «Exploitation Less Likely». Det högsta CVSS-värdet på 9.3 ligger dock tydligt över föregående månads. Den här månaden är särskild av tre skäl: SU åtgärdar det sedan juni öppna problemet med *wrapper-meddelanden* i delade postlådor, det medför tre nya eller kvarstående Known Issues, och den 2 oktober släppte Microsoft en **Version 2** som åtgärdar ytterligare en sårbarhet.

## Vilka Exchange-versioner uppdateringen är tillgänglig för

SU:erna från den 8 september 2026 finns för följande versioner:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121608, build 15.2.2562.49; offentligt tillgänglig.
- **Exchange Server 2019 CU15**: KB5121609, build 15.2.1748.51; endast via **Period-2-ESU-programmet**.
- **Exchange Server 2019 CU14**: KB5121610, build 15.2.1544.46; endast via Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5121611, build 15.1.2507.73; endast via Period 2 ESU.

Exchange 2016 och 2019 stöds inte längre. Enligt Microsoft får endast organisationer som är registrerade i Period-2-ESU-programmet SU:erna från maj till oktober 2026. Enligt KB-artiklarna gäller denna behörighet fram till oktober 2026. Det finns också press från Exchange Online: sedan den andra veckan i september begränsar och blockerar Exchange Online hybrid-e-postflödet från servrar under nivån från oktober 2025, mer information finns i [artikeln om transportenforcement](/blog/exchange-online-transport-enforcement-hybrid-server). Exchange Online är enligt tillkännagivandet redan skyddat, men i hybridmiljöer behöver ändå varje Exchange-server SU:t, liksom maskiner med Exchange Management Tools.

Du kan jämföra din aktuella nivå med översikten över [Exchange-buildnummer](/tools/exchange-builds).

## Översikt över sårbarheterna

| CVE | Typ | CVSS |
| --- | --- | --- |
| CVE-2026-69356 | Spoofing (Cross-Site-Scripting) | 9.3 |
| CVE-2026-69641 | Elevation of Privilege | 9.1 |
| CVE-2026-69355 | Remote Code Execution | 8.8 |
| CVE-2026-55007 | Remote Code Execution | 8.1 |
| CVE-2026-69380 | Elevation of Privilege | 8.1 |
| CVE-2026-69378 | Denial of Service | 7.5 |
| CVE-2026-69361 | Spoofing (Server-Side Request Forgery) | 6.5 |
| CVE-2026-69375 | Tampering | 6.5 |
| CVE-2026-69382 | Information Disclosure | 5.9 |

CVE-2026-55007 berör inte Exchange 2016; Security Update Guide listar endast Exchange SE och 2019 CU14/CU15 för den. En dokumentationsdetalj: CVE-2026-69380 saknas i CVE-listan i KB-artiklarna för de fyra SU:erna från september. Security Update Guide anger dock exakt de fyra septemberbuildarna som åtgärd för denna CVE (status 7 oktober 2026).

**[CVE-2026-69356](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69356)** har det högsta värdet med CVSS 9.3. Enligt Microsoft kan en oautentiserad angripare skicka en förberedd kalenderinbjudan med en skadlig möteslänk; om mottagaren öppnar mötet och väljer länken för att delta utlöses cross-site-scripting. Det krävs alltså användarinteraktion, men inget konto i organisationen.

**[CVE-2026-69380](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69380)** (Elevation of Privilege, CVSS 8.1) kräver endast ett konto med begränsade rättigheter och en tilldelad postlåda. Enligt FAQ:n i Security Update Guide kan en angripare, genom svagheter i kontrollen av begäranden och identitetstoken, utge sig för att vara en annan användare och ta över alla Exchange-användares postlådor: läsa och skicka e-post samt hämta bilagor. Ett enda komprometterat användarkonto räcker som utgångspunkt.

**[CVE-2026-69641](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69641)** (Elevation of Privilege, CVSS 9.1) leder till samma resultat, övertagande av alla postlådor, men kräver medlemskap i en rollgrupp med höga behörigheter.

För de två Remote Code Execution-sårbarheterna är utgångsläget olika: [CVE-2026-69355](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69355) (CVSS 8.8) kräver ett autentiserat konto med begränsade rättigheter, [CVE-2026-55007](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-55007) (CVSS 8.1) kan utlösas utan inloggning via en förberedd Visio-bilaga, men kräver enligt Microsoft ett varaktigt ont om arbetsminne på målsystemet. De övriga fyra sårbarheterna: CVE-2026-69378 (DoS genom okontrollerad rekursion, utan inloggning), CVE-2026-69361 (SSRF, där servern skickar HTTP-begäranden till interna system eller loopback-system), CVE-2026-69375 (en autentiserad angripare kan ersätta filinnehåll) och CVE-2026-69382 (exponering av inloggningsuppgifter genom en svag kryptoalgoritm, kräver en redan stulen autentiseringscookie).

## Version 2 från 2 oktober: CVE-2026-96940 tillagd

Den 2 oktober 2026 publicerade Microsoft «Version 2» av SU:erna från september. Enligt tillkännagivandet är den enda skillnaden från den första versionen den extra åtgärden för **[CVE-2026-96940](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940). Denna Elevation-of-Privilege-sårbarhet har CVSS 8.8, är varken offentligt känd eller utnyttjad, men bedöms av Microsoft som den enda av de tio CVE:erna med **«Exploitation More Likely»**. En autentiserad angripare kan därmed få åtkomst till andra postlådor i samma organisation och läsa e-post med bilagor. Exchange Online är redan korrigerat på serversidan.

| Version | KB | Build v2 |
| --- | --- | --- |
| Exchange SE RTM | KB5129955 | 15.2.2562.53 |
| Exchange 2019 CU15 | KB5129956 | 15.2.1748.53 |
| Exchange 2019 CU14 | KB5129957 | 15.2.1544.48 |
| Exchange 2016 CU23 | KB5129958 | 15.1.2507.75 |

I praktiken innebär detta: åtgärden för CVE-2026-96940 finns endast i v2-buildarna. Servrar där SU:t från 8 september redan körs behöver dessutom v2. Den som patchar först nu kan installera v2 direkt, eftersom SU:er är kumulativa. KB-artiklarna säger inte uttryckligen om servrar med det första september-SU:t måste installera v2; eftersom den nya CVE:n endast åtgärdas med v2 är det den rimliga tolkningen.

## Wrapper-problem åtgärdat: ta nu bort SettingOverride

Det sedan juni-SU:t kända problemet att *wrapper-meddelanden* visas i inkorgen för delade postlådor i hybridmiljöer är åtgärdat med september-SU:t på alla fyra versioner. I [augustiartikeln](/blog/exchange-security-updates-august-2026) stod det fortfarande att SettingOverride som dokumenterats som workaround kan finnas kvar. Efter installation av september-SU:t gäller motsatsen: Microsoft rekommenderar i den tillhörande supportartikeln att kontrollera och ta bort överstyrningen.

```powershell
Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Kommando | Effekt |
|---|---|
| `Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Kontrollerar om workaround-överstyrningen är inställd i organisationen. |
| `Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Tar bort överstyrningen när september-SU:t är installerat. |

</details>

Om det första kommandot rapporterar att objektet `DisableBlockSharedAndUserMailboxHeaders` inte hittades behöver inget mer göras enligt Microsoft.

För Exchange SE åtgärdar SU:t dessutom ett fel vid hybridfrågor om ledig/upptagen via Microsoft Graph: lokala användare såg upptagna tider i Exchange Online-postlådor förskjutna med sin egen UTC-förskjutning, utan felmeddelande.

## Kända problem

**Publicerade kalendrar (.ics) ger HTTP 500 till kalenderappar.** Problemet har funnits sedan augusti-SU:t (Exchange SE från build 15.2.2562.46 samt Exchange 2019 och 2016) och är inte åtgärdat i vare sig september-SU:t eller v2. Prenumerationer på anonymt publicerade kalendrar uppdateras inte längre; samma URL fungerar i webbläsaren. Orsaken enligt Microsoft: Exchange identifierar klienter via User-Agent, och kalenderappar utan webbläsaridentifiering hamnar i en kodväg som augusti-SU:t har inaktiverat. Som workaround beskriver Microsoft en URL Rewrite-regel på webbplatsen «Exchange Back End» i IIS, som vid .ics-begäranden under `/owa/calendar/` lägger till parametern `layout=premium`; IIS-modulen URL Rewrite måste vara installerad. De exakta stegen (via IIS Manager eller direkt i `applicationHost.config`) finns i [supportartikeln KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672). Microsoft anger inget datum för en korrigering (status 7 oktober 2026).

**Ledig/upptagen för delegerade postlådor i hybridmiljöer (endast Exchange SE).** Om tillgänglighetsfrågan är konfigurerad uteslutande via Graph API misslyckas frågor för Exchange Online-postlådor via delegerad lokal åtkomst. Outlook rapporterar «Your server location could not be determined», OWA visar «No information», och i EWS-loggarna visas `(403) Forbidden`. Den dokumenterade workarounden leder frågorna åter via EWS i stället för via Graph:

```powershell
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=False"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-Identity EnableRouteThroughMSGraphFeature` | Överstyrningen som styr vidarebefordran av tillgänglighetsfrågor via Microsoft Graph. |
| `-Parameters "Enabled=False"` | Stänger av Graph-vägen, så att frågorna åter går via EWS. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Riktar diagnostikanropet till topologitjänsten. |
| `-Component VariantConfiguration -Argument Refresh` | Läser in Variant Configuration på nytt så att överstyrningen träder i kraft utan väntan. |

</details>

I tillkännagivandet om v2 listar Microsoft detta problem bland de åtgärdade problemen. Den som har ställt in workarounden bör efter installation av v2 kontrollera i [supportartikeln KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) om den ska återställas; den 7 oktober 2026 fanns där ännu ingen anvisning för detta.

**ContentEngine-deadlock på grund av saknade koreanska WordBreaker-regelfiler (endast Exchange SE).** Med september-SU:t (build 15.2.2562.49 och v2-build 15.2.2562.53) installeras inte regelfilerna för den uppdaterade koreanska WordBreaker. Följderna är saknade sökresultat, fördröjd e-postleverans samt Outlook- eller MAPI-klienter som hänger sig eller förlorar anslutningen. [KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098) anger miljöer som behandlar e-post med koreanskspråkigt innehåll som berörda. Detta beror inte på användarnas eget språk: meddelanden med koreansk text når även organisationer utan koreansk korrespondens, exempelvis som skräppost eller nyhetsbrev. Schweiziska miljöer är därför inte på förhand undantagna. Microsoft utreder fortfarande problemet; kontroll och workaround beskrivs i nästa avsnitt.

## Workaround för WordBreaker-deadlock

Workarounden från KB5130098 lägger till två regelfiler från paketet SQL Server 2025 Express RTM. Microsoft kopplar den till villkor: den ska inte användas enbart på grund av symptomen, utan endast på servrar som uppfyller alla kontrollkriterier och där **båda** filerna saknas. [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) kontrollerar inte dessa filer; version v26.10.02 från 2 oktober 2026 innehåller ingen sådan kontroll (status 9 oktober 2026). Kontrollen görs därför manuellt.

### 1. Kontrollera servern

Följande skript ändrar ingenting i systemet. Det läser installationssökvägen från miljövariabeln `ExchangeInstallPath` och visar de värden som KB5130098 anger som förutsättning.

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
| `Join-Path $env:ExchangeInstallPath 'Bin\Search\Ceres\Native'` | Skapar sökvägen till katalogen där regelfilerna hör hemma. |
| `Get-Item ...\ExSetup.exe` | Läser filen vars version anger den installerade Exchange-builden. |
| `VersionInfo.FileVersion` | Filversion från resurserna i EXE-filen respektive DLL-filen. |
| `Get-FileHash -LiteralPath ... -Algorithm SHA256` | Beräknar SHA256-hashen för den installerade `korwbrkr.dll`. |
| `Test-Path ...` | Returnerar `True` om respektive regelfil redan finns. |

</details>

Workarounden är endast tillåten om alla värden stämmer:

| Värde | Förväntat |
| --- | --- |
| `ExSetup` | 15.02.2562.049 eller 15.02.2562.053 |
| `KorwbrkrVersion` | 16.0.5194.1000 |
| `KorwbrkrBytes` | 326544 |
| `KorwbrkrSha256` | 1C6BD8E144BA677EBCC83323AE59DB3881918170F9B3A5189B44611558B92C61 |
| `KoTokenRule`, `KoComplexRule` | båda `False` |

Om build, version eller DLL-hash avviker hänvisar Microsoft till supporten. Om en av de två regelfilerna redan finns ska du avbryta och inte skriva över något.

### 2. Hämta regelfilerna

Filerna kommer från `SQLEXPR_x64_ENU.exe`, det engelska x64-paketet för SQL Server 2025 Express RTM (version 17.0.1000.7, 748772024 byte). Microsoft rekommenderar att utföra detta steg på en hanteringsarbetsstation, inte på Exchange-servern. SQL Server installeras inte: paketet packas endast upp och Full-Text-MSI:n extraheras endast administrativt. Kontrollera först nedladdningens signatur och hash:

```powershell
Get-AuthenticodeSignature .\SQLEXPR_x64_ENU.exe |
    Select-Object Status, SignerCertificate
Get-FileHash .\SQLEXPR_x64_ENU.exe -Algorithm SHA256
```

Statusen `Valid` med ett Microsoft-certifikat och SHA256-värdet `74AA90C11202A5524E769B9BC22531BAEF22D91E9B2D2E8C3CB99E89A65C5297` förväntas. Packa sedan upp och extrahera:

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
| `/x:<Pfad>` | Extraheringsläge för SQL-paketet: packar upp installationsmedierna till mappen utan att starta SQL Setup. |
| `-Wait -PassThru` | Väntar tills processen har avslutats och returnerar processobjektet så att exitkoden kan läsas. |
| `msiexec /a <MSI>` | Administrativ installation: lägger ut MSI-innehållet som filer utan att installera eller registrera något. |
| `TARGETDIR=<Pfad>` | Målmapp för de extraherade filerna. |
| `/qn` | Utan användargränssnitt. |
| `/L*V <Datei>` | Utförlig logg som visar orsaken vid fel. |

</details>

Båda exitkoderna måste vara `0`. Om extraheringen misslyckas, spara `extract.log` och avbryt. Kontrollera sedan storlek och hash för de två filerna:

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

### 3. Kopiera filerna

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
| `Get-FileHash ... -Algorithm SHA256` | Kontrollerar hashen på målplatsen på nytt mot tabellen ovan. |

</details>

Använd `Copy-Item`, inte `Move-Item`. En kopierad fil får målatalogens ärvda behörigheter, medan en flyttad fil behåller behörigheterna från ursprungskatalogen. Filerna behöver de vanliga läsrättigheterna för `Native`. Ersätt inga andra filer i denna katalog.

### 4. Starta om Search Host Controller

```powershell
Restart-Service -Name HostControllerService
```

Omstarten avbryter tillfälligt sökning och innehållsbearbetning på servern; Microsoft avsätter ett underhållsfönster för detta. Du behöver inte starta om själva servern. Avsluta inte några Exchange-processer med tvång: om tjänsten inte startar korrekt är supporten nästa steg enligt Microsoft.

### 5. Kontroll

Host Controller startar själv processen `NodeRunner.exe` för ContentEngine; starta den inte manuellt. Den här frågan visar om den körs:

```powershell
Get-CimInstance Win32_Process -Filter "Name = 'NodeRunner.exe'" |
    Where-Object CommandLine -like '*ContentEngineNode1*' |
    Select-Object ProcessId, CreationDate
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `Get-CimInstance Win32_Process -Filter "Name = 'NodeRunner.exe'"` | Läser alla NodeRunner-processer inklusive kommandorad. |
| `Where-Object CommandLine -like '*ContentEngineNode1*'` | Filtrerar till ContentEngine-processen. |
| `Select-Object ProcessId, CreationDate` | Visar process-ID och starttid; starttiden måste ligga efter tjänstomstarten. |

</details>

Om ingen process visas eller om den avslutas upprepade gånger ska du inte bearbeta fler servrar. Annars ska du i Outlook på webben, med en postlåda vars aktiva databaskopia ligger på denna server, kontrollera om nya e-postmeddelanden levereras och hittas, även e-post med koreansk text. Kontrollera även den tjänst som ursprungligen stördes (leverans, Outlook-anslutning). Övervaka eftersläpningen i sökindexet separat: att nya e-postmeddelanden hittas bevisar inte att äldre objekt redan har bearbetats. Bearbeta ytterligare servrar en i taget och först efter lyckad kontroll av föregående.

## Installation och efterarbete

Microsoft rekommenderar det välkända förfarandet: inventera med [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), fastställ vid en föråldrad nivå vägen med [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard), installera SU:t, starta om servern och kontrollera att alla Exchange-tjänster körs. Health Checker visar därefter även om SU:t har installerats korrekt. Security Update Guide anger att omstart krävs för uppdateringarna.

Efter installationen finns tre efterarbeten:

1. Ta bort wrapper-SettingOverride `DisableBlockSharedAndUserMailboxHeaders` om den är inställd (se ovan).

2. Kontrollera på Exchange SE med kontrollskriptet från [WordBreaker-avsnittet](#workaround-für-den-wordbreaker-deadlock) om de båda regelfilerna saknas, och kontrollera om problemet med ledig/upptagen uppstår. Använd workarounderna endast om respektive villkor är uppfyllda.

3. För publicerade kalendrar med externa prenumeranter, konfigurera URL Rewrite-regeln från KB5126672 om detta inte redan har gjorts sedan augusti-SU:t.

Från juli kvarstår dessutom kontrollen av om CVE-2026-42897-mitigeringen (M2.1.0) fortfarande är aktiv; hur den tas bort beskrivs i [artikeln om juli-SU:t](/blog/exchange-security-updates-juli-2026).

## Rekommenderat tillvägagångssätt

Installera v2-buildarna från 2 oktober direkt på alla Exchange-servrar och maskiner med Management Tools; servrar med SU:t från 8 september behöver dessutom v2 för CVE-2026-96940. Spoofing-sårbarheten med CVSS 9.3 och postlådeövertagandet via ett användarkonto med begränsade rättigheter (CVE-2026-69380) är tillräckliga skäl att inte vänta till nästa patchdag. Ta sedan bort wrapper-överstyrningen, kontrollera de tre Known Issues och kör Health Checker. Health Checker rapporterar inte de saknade WordBreaker-filerna; för detta krävs kontrollskriptet från WordBreaker-avsnittet. ESU-programmet för Exchange 2016 och 2019 upphör i oktober 2026; migreringen till Exchange SE kan inte skjutas upp längre.

## Källor

1.  [Released: September 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-september-2026-exchange-server-security-updates/4554411): Officiellt release-meddelande med versioner som stöds, ESU-information, Known Issues, åtgärdade problem och installationsförfarande (läst via RSS-flödet från Exchange Team Blogs).

2.  [Released: September 2026 V2 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-september-2026-v2-exchange-server-security-updates/ba-p/4561718): Tillkännagivande av v2 från 2 oktober 2026; den enda skillnaden är CVE-2026-96940.

3.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM: September 8, 2026 (KB5121608) – Microsoft Support](https://support.microsoft.com/help/5121608): CVE-lista, åtgärdade problem och de tre Known Issues för Exchange SE.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU15: September 8, 2026 (KB5121609) – Microsoft Support](https://support.microsoft.com/help/5121609): KB-artikel för Exchange 2019 CU15.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU14: September 8, 2026 (KB5121610) – Microsoft Support](https://support.microsoft.com/help/5121610): KB-artikel för Exchange 2019 CU14.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: September 8, 2026 (KB5121611) – Microsoft Support](https://support.microsoft.com/help/5121611): KB-artikel för Exchange 2016 CU23, utan CVE-2026-55007.

7.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Typ, CVSS, allvarlighetsgrad, bedömning av exploatering och FAQ för alla nio CVE:er från september samt CVE-2026-96940, inklusive berörda buildar per CVE.

8.  [Description of version 2 of the security update for Microsoft Exchange Server Subscription Edition RTM October 2, 2026 (KB5129955) – Microsoft Support](https://support.microsoft.com/help/5129955): KB-artikel om v2 för Exchange SE; v2-artiklarna för 2019 och 2016 är KB5129956 till KB5129958.

9.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Buildnummer för september-SU:erna och v2 från 2 oktober 2026.

10. [Wrapper messages appear in shared mailbox in hybrid environments after installing the June 2026 Security Update – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Åtgärd i september-SU:t och instruktioner för att ta bort SettingOverride.

11. [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Orsak och URL Rewrite-workaround.

12. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Symptom och SettingOverride-workaround för Exchange SE.

13. [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Berörda SE-buildar, kontrollkriterier, förväntade filstorlekar och hashar, hämtning från SQL Server 2025 Express, tjänsteomstart och kontroll.

14. [Hybrid free/busy through Microsoft Graph incorrectly shifts busy times by requester timezone – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5125804): Tidszonsfelet i Exchange SE som åtgärdats med september-SU:t.

15. [Neue Sicherheitsupdates für Exchange Server (September 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-september-2026/): Tyskspråkig genomgång av de nio CVE:erna med CVSS-värden och buildar.

16. [Exchange Server: Sicherheitsupdates 8. September 2026 – Borns Tech and Windows World](https://borncity.com/blog/?p=329371): Tyskspråkig sammanfattning med information om den senare ersättningsuppdateringen.

17. [microsoft/CSS-Exchange – GitHub](https://github.com/microsoft/CSS-Exchange): Källkod för Exchange Health Checker; release v26.10.02 från 2 oktober 2026 utan kontroll av WordBreaker-filerna (status 9 oktober 2026).
