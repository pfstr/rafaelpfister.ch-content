---
title: "Claude Desktop kraschar ständigt: ”GPU process gone” med avslutskod 101457950 – orsak och lösning"
navTitle: "Claude Desktop-krasch"
description: "Claude Desktop-appen på Windows avslutas helt med ”GPU process gone: exitCode 101457950” (0x060C201E), ofta följt av Store-appens reparationsdialog. Den kompletta orsakskedjan: Code Integrity blockerar vk_swiftshader.dll, Chromiums reservkedja töms, den inbyggda självavslutningen avslutar appen. Med permanent lösning (byte till den klassiska installationen utan MSIX), självdiagnos via händelseloggen och analys ända ned till minidumpen."
date: "2026-08-25"
kategorie: "Claude"
timeToRead: "10 min läsning"
themen:
  - claude
slug: "gpu-kraschen-0x060c201e-i-claude-skrivbordsappen-felsokning-anda-till-minidumpen"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
translationSourceHash: 9bd177698f756b588c97f11ea4c73ea48efcfb96c8dbe1435f5ff9fc11dffec0
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:31:23.687Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/gpu-kraschen-0x060c201e-i-claude-skrivbordsappen-felsokning-anda-till-minidumpen
---

Claude Desktop-appen på Windows avslutas utan felmeddelande, alla pågående Claude Code-sessioner försvinner, och ibland startar appen därefter först igen efter en ”Reparera” via Windows-inställningarna. I appens logg står då denna rad:

```text
GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
```

101457950 är i hexadecimal form `0x060C201E`. Om du hittar denna signatur i din logg har du kommit rätt: Den här artikeln dokumenterar hela orsakskedjan bakom kraschen, de omedelbara åtgärder som får appen att fungera stabilt igen samt självdiagnosen som bekräftar resultatet på ditt eget system på två minuter. MSIX-installationer (från Microsoft Store eller via MSIX-installationsprogram) hos alla GPU-tillverkare berörs, från Intel-integrerade GPU:er via NVIDIA till AMD; hårdvaran är, för att säga det direkt, inte orsaken. Den klassiska installationen utan MSIX påverkas inte, och det är just lösningen.

## Lösningen i korthet: byt till den klassiska installationen

Det egentliga felet ligger i MSIX-installationspaketet och kan bara åtgärdas av Anthropic (Issue [#81341](https://github.com/anthropics/claude-code/issues/81341) är fortfarande öppet per den 25.10.2026; rapporter i ärendet bekräftade också felet i version 1.40609.1 i början av september 2026). Samma app finns dock också som klassisk installation utan MSIX, och den omfattas inte av AppX-signaturkontrollen som avslutar GPU-processen. Bytet är därmed den enda åtgärden som helt eliminerar kraschen; det bekräftas både i Issue [#81341](https://github.com/anthropics/claude-code/issues/81341) och på systemet som undersökts här. Funktionsuppsättningen är identisk och uppdateringsflödet levererar samma versioner för båda varianterna.

**Steg 1: Hämta och kör det klassiska installationsprogrammet.** Nedladdningen på [claude.com/download](https://claude.com/download) ger ett Squirrel-installationsprogram som installerar appen under `%LOCALAPPDATA%\AnthropicClaude` (administratörsbehörighet krävs inte). Via kommandoraden:

```powershell
curl.exe -L -o "$env:USERPROFILE\Downloads\Claude-Setup-x64.exe" `
  "https://storage.googleapis.com/osprey-downloads-c02f6a0d-347c-492b-a752-3e0651722e97/nest-win-x64/Claude-Setup-x64.exe"
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-L` | följer HTTP-omdirigeringar till den faktiska filen |
| `-o <pfad>` | målfil; här nedladdningsmappen |
| `<url>` | officiell installationskälla; identisk med målet för nedladdningsomdirigeringen från claude.ai |

</details>

Kontrollera signaturen efter nedladdningen (`Get-AuthenticodeSignature`, förväntat: `Valid`, utfärdare ”Anthropic, PBC”) och starta filen. Installationsprogrammet lägger först in en äldre basversion; uppdateringsmekanismen uppdaterar den till aktuell version, antingen automatiskt vid första start eller direkt med:

```powershell
& "$env:LOCALAPPDATA\AnthropicClaude\Update.exe" `
  --update https://downloads.claude.ai/releases/win32/x64
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `--update <url>` | hämtar den senaste versionen från det angivna versionsflödet och installerar den som en ny `app-<version>`-katalog |

</details>

**Steg 2: Överta konfigurationen.** MSIX-versionen sparar inloggning, MCP-serverkonfiguration och inställningar i sin virtualiserade container; den klassiska appen läser `%APPDATA%\Claude`. Kopiera en gång (avsluta först MSIX-appen; de båda varianterna kan ändå inte köras samtidigt på grund av ett gemensamt single-instance-lås):

```powershell
robocopy "$env:LOCALAPPDATA\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude" `
  "$env:APPDATA\Claude" /E /XD Cache "Code Cache" GPUCache claude-code Crashpad logs sentry
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `<quelle>` | konfigurationskatalog i MSIX-paketets virtualiserade AppData |
| `<ziel>` | konfigurationskatalog för den klassiska installationen |
| `/E` | kopierar alla underkataloger, även tomma |
| `/XD <namen>` | hoppar över angivna kataloger; här cacheminnen och körningsdata som den nya appen själv skapar på nytt |

</details>

Chatthistoriken går inte förlorad: den finns i claude.ai-kontot respektive (för Claude Code-sessioner) under `%USERPROFILE%\.claude` och är inte knuten till appinstallationen.

**Steg 3: Ta bort MSIX-paketet.** Annars startar den kraschande varianten fortfarande via gamla genvägar:

```powershell
Get-AppxPackage Claude | Remove-AppxPackage
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `Claude` | positionsargumentet namn för `Get-AppxPackage`: filtrerar installerade AppX-/MSIX-paket på paketnamn (jokertecken tillåts) |
| `Remove-AppxPackage` | tar bort paketet som skickats via pipelinen för det aktuella användarkontot |

</details>

Startmenyposten ”Anthropic → Claude” hör därefter till den klassiska installationen; en eventuell fästning i aktivitetsfältet måste göras om.

## Om du måste stanna kvar med MSIX-paketet

Utan byte återstår endast åtgärder som minskar kraschfrekvensen utan att eliminera orsaken:

**Använd den inbäddade webbläsaren sparsamt.** Utlösare för kraschen är sidor i appens webbläsar-/förhandsvisningsområde. Den som stänger området efter användning, i stället för att lämna flikar parkerade, minskar kraschfrekvensen avsevärt; detta samband är belagt med siffror flera gånger i community-tråden.

**Stäng av WebGPU.** Start med `--disable-features=WebGPU` förhindrar den vanligaste utlösaren. För ett MSIX-paket ändras installationssökvägen vid varje uppdatering, därför behövs en startare som löser upp den på nytt vid varje start:

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `for /f "delims="` | bearbetar kommandoutdata rad för rad; tomt `delims=` överför hela raden (inklusive blanksteg i sökvägen) till `%%i` |
| `-NoProfile` | startar PowerShell utan profilskript, för snabb och reproducerbar start |
| `-Command` | kör det angivna uttrycket; `(Get-AppxPackage Claude).InstallLocation` returnerar paketets aktuella installationssökväg |
| `start ""` | startar programmet frikopplat från batchfönstret; de tomma citationstecknen är den (här tomma) fönstertiteln |
| `--disable-features=WebGPU` | Chromium-växel: inaktiverar den angivna funktionen, här WebGPU-API:t |

</details>

Det fungerar endast om appen också startas via denna startare.

I den första versionen av denna artikel stod rekommendationen att aktivera hårdvaruacceleration via `isHardwareAccelerationDisabled: false` i `config.json` först. Den rekommendationen är inaktuell: I aktuella versioner (1.37937.x) finns flaggan inte längre, appen startar med aktiv hårdvaruacceleration som standard och kraschar ändå (detaljer i tillägget nedan).

Att ”Reparera” eller installera om MSIX-paketet löser för övrigt inte problemet, utan botar bara följdsymptomet (mer om det nedan). Även uppdateringar av grafikdrivrutinen är bortkastad möda.

## Självdiagnos: bekräfta resultatet på ditt eget system

Två kontroller räcker. Först kraschsignaturen i appens logg:

```powershell
Select-String -Path "$env:LOCALAPPDATA\Claude\Logs\main.log" `
  -Pattern 'GPU process gone'
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

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
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-FilterHashtable` | filtrerar redan vid hämtningen: `LogName` anger händelseloggen, `Id` händelse-ID 3033 (Code Integrity-blockering) |
| `-MaxEvents 30` | begränsar frågan till de 30 senaste träffarna |
| `Where-Object { … -match 'claude' }` | behåller endast händelser vars meddelandetext innehåller appsökvägen |
| `Select-Object TimeCreated, Message` | begränsar utdata till tidsstämpel och meddelande för jämförelse med kraschtidpunkterna |

</details>

På berörda system hittar du där Event 3033-poster vars tidsstämplar stämmer överens med kraschtidpunkterna på sekunden, med detta meddelande:

```text
Code Integrity determined that a process
(...\WindowsApps\Claude_..._x64__pzs8sxrjxfjjc\app\claude.exe)
attempted to load ...\app\vk_swiftshader.dll that did not meet
the Microsoft signing level requirements.
```

På systemet som undersökts här stämde sju av sju krascher under tre veckor överens på sekunden med en sådan händelse, inklusive en avsiktligt utlöst kontrollkrasch.

## Den kompletta orsakskedjan

Kraschen är slutledet i en kedja med fyra länkar, som framgår av två analyser tillsammans: Code Integrity-spåret från community-ärendet [#81698](https://github.com/anthropics/claude-code/issues/81698) och en egen minidump-analys ([#89250](https://github.com/anthropics/claude-code/issues/89250)).

**Länk 1: En sida i den inbäddade webbläsaren behöver programvarurendering.** En typisk utlösare är ett WebGPU-anrop (`navigator.gpu.requestAdapter()`), synligt i fönsterloggen genom denna varning precis före kraschen:

```text
[warn] The powerPreference option is currently ignored
       when calling requestAdapter() on Windows.
```

Om appen körs utan hårdvaruacceleration måste vägen gå via programvaru-Vulkan-implementeringen SwiftShader: GPU-processen försöker läsa in den medföljande `vk_swiftshader.dll`.

**Länk 2: Windows Code Integrity blockerar appens egen DLL.** GPU-processen körs med härdningsprincipen ”MicrosoftSignedOnly” (kan kontrolleras med `Get-ProcessMitigation`). För att en Store-app ska få läsa in sina egna tillverkarsignerade DLL:er måste MSIX-paketet innehålla en signaturkatalog `AppxMetadata\CodeIntegrity.cat`. Just den filen saknas i det levererade paketet, vilket community-medlemmar har visat genom inspektion av MSIX-filen. Följden: signaturkontrollen misslyckas, Windows loggar Event 3033 och avslutar GPU-processen hårt. Avslutskoden `0x060C201E` är ett AppX-integritetsfel från Windows-inläsaren, inte en Chromium-kod; därför finns den inte i någon Chromium-källkod, och därför lämnar GPU-processen inte heller någon kraschdump – det finns inget undantag som kan dumpas.

**Länk 3: Chromiums reservkedja töms.** När GPU-processen kraschar växlar Chromium ned en renderingsnivå: hårdvaru-GL, sedan programvaru-GL, sedan en ren display-compositor. Först när ingen nivå återstår aktiveras den inbyggda självavslutningen. I källkoden för den medföljande versionen (Chromium 148.0.7778.280 i Electron 42.9.2) står det ordagrant:

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

**Länk 4: Huvudprocessen avslutas avsiktligt.** Detta `LOG(FATAL)` är ögonblicket då ”appen kraschar”. Det beläggs av en minidump från huvudprocessen: `EXCEPTION_BREAKPOINT` (en avsiktlig `int3`, inget drivrutinsfel), inte en enda grafikdrivrutins-DLL i processen och, i minnet i klartext:

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

Att denna dump över huvud taget finns var den svåraste delen av analysen: Appens Sentry-integration konsumerar Crashpad-dumpar vid nästa appstart, skickar dem till tillverkarens telemetri och raderar dem lokalt. Crashpad-katalogen är därför alltid tom för användaren. En observatör som är oberoende av appens processträd hjälper (startas via WMI så att appkraschen inte även avslutar den), och söker igenom Crashpad-databasen var 200:e millisekund efter `*.dmp` samt kopierar omedelbart bort fynd innan de raderas. Utvärderingen görs av Python-paketet `minidump`, helt utan WinDbg.

## Varför ”inaktivera hårdvaruacceleration” förvärrar allt

Kedjan förklarar också det mest kontraintuitiva resultatet. Inaktiverad hårdvaruacceleration har här två ödesdigra effekter samtidigt. För det första tvingar den fram SwiftShader-sökvägen, alltså just det DLL-inläsningsförsök som Code Integrity blockerar; med aktiv hårdvaruacceleration behövs `vk_swiftshader.dll` däremot nästan aldrig. För det andra startar GPU-processen då redan längst ned i reservkedjan: en enda krasch räcker och länk 4 slår till. Det förklarar även observationen från community-tråden att en Code Integrity-blockering ibland är utan följder och ibland avslutar appen: det beror på hur många reservnivåer webbläsarprocessen har kvar.

Särskilt olyckligt: Appen kände till en automatisk inaktivering av hårdvaruacceleration efter problem (`isHardwareAccelerationAutoDisabled`). Avsedd som stabilitetsåtgärd ledde den berörda system rakt in i den konfiguration där nästa krasch kostar hela appen.

## Tillägg 27.08.2026: Enbart hårdvaruacceleration räcker inte

Den första versionen av denna artikel rekommenderade aktiv hårdvaruacceleration som den effektivaste omedelbara åtgärden, och i två dagar förblev appen faktiskt kraschfri med den. Sedan följde den automatiska uppdateringen till 1.37937.3, och med den tre krascher under en eftermiddag, var och en med det välkända Event 3033 för `vk_swiftshader.dll`. Två slutsatser:

För det första saknas även den saknade signaturkatalogen i det aktuella MSIX-paketet; grundproblemet kvarstår oförändrat i 1.37937.3.

För det andra skyddar aktiv hårdvaruacceleration bara statistiskt: den förlänger reservkedjan, men hindrar inte att Chromium vid belastning eller efter ett fel i hårdvarans GPU-process ändå går igenom den till SwiftShader-nivån. När det sker blockerar Code Integrity DLL:en och kedjan kan trots allt tömmas. Därtill har konfigurationsflaggorna `isHardwareAccelerationDisabled`/`isHardwareAccelerationAutoDisabled` försvunnit från `config.json` i 1.37937.x; inställningen går inte längre att låsa där.

Därmed återstod endast bytet till den klassiska installationen som beskrivs ovan som en tillförlitlig lösning. Sedan bytet på systemet som undersökts här: samma appversion, identisk användning inklusive webbläsarområdet, inte ett enda Event 3033 och ingen krasch.

## Följdsymptomet: reparationsslingan

Code Integrity-felet har en bieffekt som många berörda uppfattar som ett separat problem: Windows klassar ibland appaketet efter händelsen som ”Modified, NeedsRemediation”. Appen startar då inte alls förrän den återställs via Inställningar → Appar → Claude → Avancerade alternativ → ”Reparera”. Den som alltså ”ständigt måste reparera appen” ser samma grundproblem, bara en länk längre fram: Reparationen åtgärdar paketstatusen, inte orsaken; nästa krasch följer vid nästa blockerade DLL-inläsningsförsök.

## Status för rapporterna

Paketeringsorsaken är rapporterad som [#81341](https://github.com/anthropics/claude-code/issues/81341), samlingstråden med community-bevisen är [#81698](https://github.com/anthropics/claude-code/issues/81698), minidump-analysen med förklaringen av reservkedjan är [#89250](https://github.com/anthropics/claude-code/issues/89250), ytterligare en utförlig rapport inklusive reparationsslingan är [#80444](https://github.com/anthropics/claude-code/issues/80444). Den egentliga fixen, en fullständig signaturkatalog i MSIX-paketet, ligger hos Anthropic. Per den 25.10.2026 är #81341 fortfarande öppet och rapporter från början av september 2026 bekräftar också felet i version 1.40609.1. #80444 stängdes den 15 september 2026 utan uppgift om en åtgärdad version, #89250 den 20 oktober 2026 automatiskt på grund av inaktivitet; inget av de två ärendena nämner någon fix. Fram till dess gäller: byt till den klassiska installationen; den som måste stanna kvar med MSIX-paketet stänger webbläsarområdet disciplinerat och stänger vid behov av WebGPU via flagga. På systemet som undersökts här har appen varit kraschfri sedan bytet till den klassiska installationen, utan ett enda ytterligare Event 3033.

## Ändringshistorik

- **25 oktober 2026:** Tillagt: Issue #81341 är fortfarande öppet per den 25.10.2026, felet är även bekräftat i version 1.40609.1. Issues #80444 och #89250 stängdes utan att någon fix nämndes.

## Källor

1.  [GitHub-issue #81698: GPU process crash kills entire app](https://github.com/anthropics/claude-code/issues/81698): Samlingstråden med community-bevis för Code Integrity-kedjan, data från olika tillverkare och korrelationen med webbläsarpanelen.

2.  [GitHub-issue #81341: CIG + vk_swiftshader.dll kills GPU process](https://github.com/anthropics/claude-code/issues/81341): Paketeringsorsaken; saknad CodeIntegrity-katalog i MSIX.

3.  [GitHub-issue #89250: Minidump-analys av appavslutningen](https://github.com/anthropics/claude-code/issues/89250): Den andra halvan av kedjan som beskrivs här, med dump-insamlingsmetod och förslag på åtgärder.

4.  [GitHub-issue #80444: GPU-krasch med forensik och reparationsslinga](https://github.com/anthropics/claude-code/issues/80444): Utförlig enskild rapport med tidslinjer, utvärdering av händelseloggen och slutsatsen att varje krasch försätter paketet i tillståndet ”Modified”.

5.  [Claude Desktop: officiell nedladdningssida](https://claude.com/download): Källa för det klassiska Windows-installationsprogrammet (x64 och ARM64).

6.  [Chromium-källkod: gpu_data_manager_impl_private.cc (tagg 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): Funktionen IntentionallyCrashBrowserForUnusableGpuProcess och reservlogiken.

7.  [Electron-dokumentation: child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): Händelsen som en Electron-app kan använda för att observera GPU-processkrascher och vidta egna motåtgärder.

8.  [Python-paketet minidump](https://pypi.org/project/minidump/): Verktyg för dumpanalysen (exception-record, modullista, minnessträngar).

9.  [webgpureport.org](https://webgpureport.org/): WebGPU-diagnossida; användes som minimal utlösare för kontrollkraschen och för jämförelsetestet i aktuella Chromium.
