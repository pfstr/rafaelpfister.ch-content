---
title: "Claude Desktop kraschar ständigt: ”GPU process gone” med exitkod 101457950, orsak och lösning"
navTitle: "Krasch i Claude Desktop"
description: "Claude Desktop-appen i Windows avslutas helt med ”GPU process gone: exitCode 101457950” (0x060C201E), ofta följt av reparationsdialogrutan för Store-appen. Den fullständiga orsakskedjan: Code Integrity blockerar vk_swiftshader.dll, Chromiums fallback-kedja töms, det inbyggda självavbrottet avslutar appen. Med en permanent lösning (byte till den klassiska installationen utan MSIX), självdiagnostik via händelseloggen och analys ända ned till minidumpen."
date: "2026-08-25"
kategorie: "Claude"
timeToRead: "10 min lästid"
themen:
  - claude
slug: "gpu-kraschen-0x060c201e-i-claude-skrivbordsappen-felsokning-anda-till-minidumpen"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
translationSourceHash: 769984b49b04b65b0b8f8a91ce3b6dd65e2eef1a4212bed32b83422f431a8559
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:26:33.340Z
translationReview: automatic
url: https://rafaelpfister.ch/sv/blog/gpu-kraschen-0x060c201e-i-claude-skrivbordsappen-felsokning-anda-till-minidumpen
---

Claude Desktop-appen i Windows avslutas utan felmeddelande, alla pågående Claude Code-sessioner försvinner och ibland startar appen därefter först igen efter en ”Reparera” via Windows-inställningarna. I appens logg står då denna rad:

```text
GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
```

101457950 är hexadecimalt `0x060C201E`. Om du hittar denna signatur i din logg har du kommit rätt: Den här artikeln dokumenterar hela orsakskedjan bakom kraschen, de omedelbara åtgärder som gör appen stabil igen och självdiagnostiken som låter dig bekräfta fyndet på ditt eget system på två minuter. MSIX-installationer (från Microsoft Store eller via MSIX-installationsprogram) påverkas på alla GPU-tillverkare, från Intel-integrerade GPU:er via NVIDIA till AMD; hårdvaran är, för att avslöja det redan nu, inte orsaken. Den klassiska installationen utan MSIX påverkas inte, och det är just lösningen.

## Lösningen i korthet: byt till den klassiska installationen

Det egentliga felet ligger i MSIX-installationspaketet och kan bara åtgärdas av Anthropic (fortfarande öppet 27.08.2026, ärende [#81341](https://github.com/anthropics/claude-code/issues/81341); även den aktuella versionen 1.37937.3 påverkas). Samma app finns även som klassisk installation utan MSIX, och den omfattas inte av AppX-signaturkontrollen som avslutar GPU-processen. Bytet är därmed den enda åtgärden som helt eliminerar kraschen; det har bekräftats både i ärende [#81341](https://github.com/anthropics/claude-code/issues/81341) och på systemet som undersökts här. Funktionsuppsättningen är identisk och uppdateringsflödet levererar samma versioner för båda varianterna.

**Steg 1: Hämta och kör det klassiska installationsprogrammet.** Nedladdningen på [claude.com/download](https://claude.com/download) ger ett Squirrel-installationsprogram som installerar appen i `%LOCALAPPDATA%\AnthropicClaude` (administratörsbehörighet krävs inte). Via kommandoraden:

```powershell
curl.exe -L -o "$env:USERPROFILE\Downloads\Claude-Setup-x64.exe" `
  "https://storage.googleapis.com/osprey-downloads-c02f6a0d-347c-492b-a752-3e0651722e97/nest-win-x64/Claude-Setup-x64.exe"
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-L` | följer HTTP-omdirigeringar fram till den faktiska filen |
| `-o <pfad>` | målfil; här mappen Hämtade filer |
| `<url>` | officiell källa för installationsprogrammet; identisk med målet för nedladdningsomdirigeringen från claude.ai |

</details>

Kontrollera signaturen efter nedladdningen (`Get-AuthenticodeSignature`, förväntat: `Valid`, utfärdare ”Anthropic, PBC”) och starta filen. Installationsprogrammet lägger först in en äldre basversion; den uppdateras till aktuell version via uppdateringsmekanismen, antingen automatiskt vid första starten eller direkt med:

```powershell
& "$env:LOCALAPPDATA\AnthropicClaude\Update.exe" `
  --update https://downloads.claude.ai/releases/win32/x64
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `--update <url>` | hämtar den senaste versionen från det angivna releaseflödet och installerar den som en ny `app-<version>`-katalog |

</details>

**Steg 2: Överför konfigurationen.** MSIX-versionen lagrar inloggning, MCP-serverkonfiguration och inställningar i sin virtualiserade container; den klassiska appen läser `%APPDATA%\Claude`. Kopiera en gång (avsluta först MSIX-appen; båda varianterna kan ändå inte köras samtidigt på grund av ett gemensamt single-instance-lås):

```powershell
robocopy "$env:LOCALAPPDATA\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude" `
  "$env:APPDATA\Claude" /E /XD Cache "Code Cache" GPUCache claude-code Crashpad logs sentry
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `<quelle>` | konfigurationskatalog i MSIX-paketets virtualiserade AppData |
| `<ziel>` | konfigurationskatalog för den klassiska installationen |
| `/E` | kopierar alla underkataloger, även tomma |
| `/XD <namen>` | hoppar över de angivna katalogerna; här cache och körningsdata som den nya appen själv skapar på nytt |

</details>

Chatthistorik går inte förlorad: den finns i claude.ai-kontot respektive (för Claude Code-sessioner) under `%USERPROFILE%\.claude` och är inte kopplad till appinstallationen.

**Steg 3: Ta bort MSIX-paketet.** Annars startar den kraschande varianten fortfarande via gamla genvägar:

```powershell
Get-AppxPackage Claude | Remove-AppxPackage
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `Claude` | positionsargumentet Name för `Get-AppxPackage`: filtrerar installerade AppX-/MSIX-paket på paketnamnet (jokertecken tillåts) |
| `Remove-AppxPackage` | tar bort paketet som skickas via pipeline för det aktuella användarkontot |

</details>

Startmenyposten ”Anthropic → Claude” tillhör därefter den klassiska installationen; en eventuell fästning i aktivitetsfältet måste göras om.

## Om du måste behålla MSIX-paketet

Utan byte återstår bara åtgärder som minskar kraschfrekvensen utan att eliminera orsaken:

**Använd den inbäddade webbläsaren sparsamt.** Kraschen utlöses av sidor i appens webbläsar-/förhandsgranskningsområde. Den som stänger området efter användning i stället för att lämna flikar öppna minskar kraschfrekvensen avsevärt; detta samband har dokumenterats flera gånger med siffror i communitytråden.

**Stäng av WebGPU.** Start med `--disable-features=WebGPU` förhindrar den vanligaste utlösaren. För ett MSIX-paket ändras installationssökvägen vid varje uppdatering, därför behövs en startare som löser upp den på nytt vid varje start:

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `for /f "delims="` | bearbetar kommandoutdata rad för rad; tom `delims=` kopierar hela raden (inklusive blanksteg i sökvägen) till `%%i` |
| `-NoProfile` | startar PowerShell utan profilskript, för snabb och reproducerbar start |
| `-Command` | kör det angivna uttrycket; `(Get-AppxPackage Claude).InstallLocation` returnerar paketets aktuella installationssökväg |
| `start ""` | startar programmet frikopplat från batchfönstret; de tomma citationstecknen är fönstrets (här tomma) titel |
| `--disable-features=WebGPU` | Chromium-växel: inaktiverar den angivna funktionen, här WebGPU-API:et |

</details>

Det fungerar bara om appen också startas via denna startare.

I den första versionen av denna artikel stod rekommendationen att aktivera maskinvaruacceleration via `isHardwareAccelerationDisabled: false` i `config.json` först. Den rekommendationen är föråldrad: I aktuella versioner (1.37937.x) finns flaggan inte längre, appen startar med maskinvaruacceleration aktiverad som standard och kraschar ändå (se detaljer i tillägget nedan).

”Reparera” eller ominstallation av MSIX-paketet löser för övrigt inte problemet; det åtgärdar bara följdsymptomet (mer om detta nedan). Uppdateringar av grafikdrivrutiner är också bortkastad möda.

## Självdiagnostik: bekräfta fyndet på ditt eget system

Två kontroller räcker. Först kraschsignaturen i appens logg:

```powershell
Select-String -Path "$env:LOCALAPPDATA\Claude\Logs\main.log" `
  -Pattern 'GPU process gone'
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-Path` | fil att söka igenom, här appens huvudlogg |
| `-Pattern` | sökmönster (reguljärt uttryck); visar alla rader med kraschsignaturen |

</details>

För det andra, och detta är det egentliga beviset, Windows CodeIntegrity-logg:

```powershell
Get-WinEvent -FilterHashtable @{
  LogName='Microsoft-Windows-CodeIntegrity/Operational'; Id=3033
} -MaxEvents 30 | Where-Object { $_.Message -match 'claude' } |
  Select-Object TimeCreated, Message
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-FilterHashtable` | filtrerar redan vid hämtningen: `LogName` anger händelseloggen, `Id` händelse-ID 3033 (Code Integrity-blockering) |
| `-MaxEvents 30` | begränsar frågan till de 30 senaste träffarna |
| `Where-Object { … -match 'claude' }` | behåller endast händelser vars meddelandetext innehåller appsökvägen |
| `Select-Object TimeCreated, Message` | begränsar utdata till tidsstämpel och meddelande för jämförelse med kraschtiderna |

</details>

På berörda system hittar du där Event 3033-poster vars tidsstämplar stämmer överens med kraschtiderna på sekunden, med detta meddelande:

```text
Code Integrity determined that a process
(...\WindowsApps\Claude_..._x64__pzs8sxrjxfjjc\app\claude.exe)
attempted to load ...\app\vk_swiftshader.dll that did not meet
the Microsoft signing level requirements.
```

På systemet som undersökts här stämde sju av sju krascher under tre veckor överens på sekunden med en sådan händelse, inklusive en avsiktligt utlöst kontrollkrasch.

## Den fullständiga orsakskedjan

Kraschen är slutledet i en kedja med fyra länkar, som två analyser tillsammans visar: Code Integrity-spåret från communityärendet [#81698](https://github.com/anthropics/claude-code/issues/81698) och en egen minidump-analys ([#89250](https://github.com/anthropics/claude-code/issues/89250)).

**Länk 1: En sida i den inbäddade webbläsaren behöver mjukvarurendering.** En typisk utlösare är ett WebGPU-anrop (`navigator.gpu.requestAdapter()`), vilket i fönsterloggen kan kännas igen på denna varning omedelbart före kraschen:

```text
[warn] The powerPreference option is currently ignored
       when calling requestAdapter() on Windows.
```

Körs appen utan maskinvaruacceleration går vägen oundvikligen via programvaru-Vulkan-implementeringen SwiftShader: GPU-processen försöker läsa in den medföljande `vk_swiftshader.dll`.

**Länk 2: Windows Code Integrity blockerar appens egen DLL.** GPU-processen körs med härdningsprincipen ”MicrosoftSignedOnly” (kan kontrolleras via `Get-ProcessMitigation`). För att en Store-app ska få läsa in sina egna, tillverkarsignerade DLL:er måste MSIX-paketet innehålla en signaturkatalog `AppxMetadata\CodeIntegrity.cat`. Just denna fil saknas i det levererade paketet, vilket communitymedlemmar har visat genom inspektion av MSIX-filen. Följden: signaturkontrollen misslyckas, Windows loggar Event 3033 och avslutar GPU-processen hårt. Exitkoden `0x060C201E` är ett AppX-integritetsfel från Windows-laddaren, inte en Chromium-kod; därför återfinns den inte i någon Chromium-källa och därför lämnar GPU-processen ingen kraschdump efter sig – det finns inget undantag att dumpa.

**Länk 3: Chromiums fallback-kedja töms.** När GPU-processen kraschar växlar Chromium ned ett renderingssteg: maskinvaru-GL, sedan mjukvaru-GL och därefter en ren display-kompositor. Först när inget steg återstår aktiveras det inbyggda självavbrottet. I källkoden för den medföljande versionen (Chromium 148.0.7778.280 i Electron 42.9.2) står det ordagrant så här:

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

**Länk 4: Huvudprocessen avslutar sig avsiktligt.** Detta `LOG(FATAL)` är ögonblicket då ”appen kraschar”. Det bevisas av en minidump från huvudprocessen: `EXCEPTION_BREAKPOINT` (en avsiktlig `int3`, inget drivrutinsfel), inte en enda grafikdrivrutins-DLL i processen, och i minnet i klartext:

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

Att denna dump alls finns var den svåraste delen av analysen: Appens Sentry-integration förbrukar Crashpad-dumpar vid nästa appstart, skickar dem till tillverkarens telemetri och raderar dem lokalt. Crashpad-katalogen är därför alltid tom för användaren. Lösningen är en övervakare som är oberoende av appens processträd (startad via WMI så att appkraschen inte också avslutar den), som genomsöker Crashpad-databasen efter `*.dmp` var 200:e millisekund och omedelbart kopierar bort träffar innan de raderas. Analysen görs av Python-paketet `minidump`, helt utan WinDbg.

## Varför ”inaktivera maskinvaruacceleration” förvärrar allt

Kedjan förklarar också det mest kontraintuitiva fyndet. Inaktiverad maskinvaruacceleration har här två ödesdigra effekter samtidigt. För det första tvingar den fram SwiftShader-sökvägen, alltså precis det DLL-inläsningsförsök som Code Integrity blockerar; med aktiv maskinvaruacceleration behövs `vk_swiftshader.dll` däremot nästan aldrig. För det andra startar GPU-processen redan i fallback-kedjans nedre ände: en enda krasch räcker och länk 4 slår till. Det förklarar även observationen i communitytråden att en Code Integrity-blockering ibland inte får några följder och ibland avslutar appen: det beror på hur många fallback-steg webbläsarprocessen ännu har kvar.

Särskilt olyckligt: Appen hade en automatisk inaktivering av maskinvaruacceleration efter problem (`isHardwareAccelerationAutoDisabled`). Avsedd som stabilitetsåtgärd förde den berörda system direkt in i den konfiguration där nästa krasch kostar hela appen.

## Tillägg 27.08.2026: Maskinvaruacceleration räcker inte i sig

Den första versionen av denna artikel rekommenderade aktiv maskinvaruacceleration som den mest effektiva omedelbara åtgärden, och under två dagar förblev appen faktiskt kraschfri med den. Sedan kom den automatiska uppdateringen till 1.37937.3, och med den tre krascher under en eftermiddag, var och en med det välkända Event 3033 för `vk_swiftshader.dll`. Två fynd följer av detta:

För det första saknas även den saknade signaturkatalogen i det aktuella MSIX-paketet; grundproblemet är oförändrat i 1.37937.3.

För det andra skyddar aktiv maskinvaruacceleration endast statistiskt: den förlänger fallback-kedjan men förhindrar inte att Chromium under belastning eller efter ett fel i maskinvarans GPU-process ändå går hela vägen till SwiftShader-steget. Så snart det händer blockerar Code Integrity DLL:en och kedjan kan ändå tömmas. Därtill har konfigurationsflaggorna `isHardwareAccelerationDisabled`/`isHardwareAccelerationAutoDisabled` försvunnit från `config.json` i 1.37937.x; inställningen går inte längre att låsa där.

Därmed återstod som tillförlitlig lösning endast bytet till den klassiska installationen som beskrivs ovan. Sedan bytet på systemet som undersökts här: samma appversion, identisk användning inklusive webbläsarområdet, inte ett enda Event 3033 och ingen krasch.

## Följdsymptomet: reparationsloopen

Code Integrity-felet har en bieffekt som många berörda betraktar som ett eget problem: Windows klassar ibland appaketet efter incidenten som ”Modified, NeedsRemediation”. Appen startar då inte alls förrän den återställs via Inställningar → Appar → Claude → Avancerade alternativ → ”Reparera”. Den som alltså ”ständigt måste reparera appen” ser samma grundproblem, bara en länk längre fram: reparationen åtgärdar paketstatusen, inte orsaken; nästa krasch följer vid nästa blockerade DLL-inläsningsförsök.

## Status för rapporterna

Paketeringsorsaken har rapporterats som [#81341](https://github.com/anthropics/claude-code/issues/81341), samlingstråden med communitybevisen är [#81698](https://github.com/anthropics/claude-code/issues/81698), minidump-analysen med förklaringen av fallback-kedjan är [#89250](https://github.com/anthropics/claude-code/issues/89250), och en annan utförlig rapport inklusive reparationsloopen är [#80444](https://github.com/anthropics/claude-code/issues/80444). Den egentliga korrigeringen, en komplett signaturkatalog i MSIX-paketet, ligger hos Anthropic och saknas fortfarande även i 1.37937.3. Tills dess gäller: byt till den klassiska installationen; den som måste behålla MSIX-paketet stänger webbläsarområdet disciplinerat och inaktiverar vid behov WebGPU med en flagga. På systemet som undersökts här är appen kraschfri sedan bytet till den klassiska installationen, utan ett enda ytterligare Event 3033.

## Källor

1.  [GitHub-ärende #81698: GPU process crash kills entire app](https://github.com/anthropics/claude-code/issues/81698): Samlingstråden med communitybevis för Code Integrity-kedjan, data från olika tillverkare och korrelationen med webbläsarpanelen.

2.  [GitHub-ärende #81341: CIG + vk_swiftshader.dll kills GPU process](https://github.com/anthropics/claude-code/issues/81341): Paketeringsorsaken; saknad CodeIntegrity-katalog i MSIX.

3.  [GitHub-ärende #89250: Minidump-analys av appavbrottet](https://github.com/anthropics/claude-code/issues/89250): Den andra halvan av kedjan som beskrivs här, med dumpinsamlingsmetod och förslag till korrigeringar.

4.  [GitHub-ärende #80444: GPU-krasch med forensik och reparationsloop](https://github.com/anthropics/claude-code/issues/80444): Utförlig enskild rapport med tidslinjer, analys av händelseloggen och fyndet att varje krasch försätter paketet i tillståndet ”Modified”.

5.  [Claude Desktop: officiell nedladdningssida](https://claude.com/download): Källa för det klassiska Windows-installationsprogrammet (x64 och ARM64).

6.  [Chromium-källkod: gpu_data_manager_impl_private.cc (tagg 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): Funktionen IntentionallyCrashBrowserForUnusableGpuProcess och fallback-logiken.

7.  [Electron-dokumentation: child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): Händelsen som en Electron-app kan använda för att observera GPU-processkrascher och vidta egna motåtgärder.

8.  [Python-paketet minidump](https://pypi.org/project/minidump/): Verktyg för dumpanalys (exception record, modullista, minnessträngar).

9.  [webgpureport.org](https://webgpureport.org/): WebGPU-diagnossida; användes som minimal utlösare för kontrollkraschen och för jämförelsetestet i aktuellt Chromium.
