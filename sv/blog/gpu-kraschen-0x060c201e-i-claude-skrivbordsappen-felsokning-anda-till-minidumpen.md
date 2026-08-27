---
title: "Claude Desktop kraschar ständigt: ”GPU process gone” med exitkod 101457950, orsak och lösning"
navTitle: "Claude Desktop-krasch"
description: "Claude Desktop-appen i Windows avslutas helt med ”GPU process gone: exitCode 101457950” (0x060C201E), ofta följt av reparationsdialogen för Store-appen. Den fullständiga orsakskedjan: Code Integrity blockerar vk_swiftshader.dll, Chromiums fallback-kedja tar slut och den inbyggda självavslutningen stänger appen. Med omedelbar lösning, självdiagnos via händelseloggen och analys ända ned till minidumpen."
date: "2026-08-25"
kategorie: "Claude"
timeToRead: "9 min läsning"
themen:
  - claude
slug: "gpu-kraschen-0x060c201e-i-claude-skrivbordsappen-felsokning-anda-till-minidumpen"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
translationSourceHash: 61bcad89e160ee37f5abd04905ed9e425236f770f9cfcc4448716acbd3569939
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:35:50.386Z
translationReview: automatic
url: https://rafaelpfister.ch/sv/blog/gpu-kraschen-0x060c201e-i-claude-skrivbordsappen-felsokning-anda-till-minidumpen
---

Claude Desktop-appen i Windows avslutas utan felmeddelande, alla pågående Claude Code-sessioner försvinner, och ibland startar appen därefter först igen efter en ”Reparera” via Windows-inställningarna. I appens logg står då följande rad:

```text
GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
```

101457950 är hexadecimal `0x060C201E`. Om du hittar denna signatur i din logg har du kommit rätt: Den här artikeln dokumenterar den fullständiga orsakskedjan bakom kraschen, de omedelbara åtgärder som gör appen stabil igen och självdiagnosen som låter dig bekräfta fyndet på ditt eget system på två minuter. Installationer från Microsoft Store (MSIX) påverkas på alla GPU-tillverkare, från Intel-integrerade GPU:er via NVIDIA till AMD; hårdvaran är, för att avslöja det redan nu, inte orsaken.

## Lösningen i korthet

Det egentliga felet finns i appens installationspaket och kan endast åtgärdas av Anthropic (fortfarande öppet den 25.08.2026, ärende [#81341](https://github.com/anthropics/claude-code/issues/81341)). Fram till dess gör tre åtgärder appen stabil, i fallande effektivitetsordning:

**1. Aktivera hårdvaruacceleration.** Kontrollera dessa två värden i filen `%LOCALAPPDATA%\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude\config.json` och sätt dem vid behov till `false` (avsluta appen först och starta sedan om den):

```json
"isHardwareAccelerationDisabled": false,
"isHardwareAccelerationAutoDisabled": false
```

Det låter paradoxalt, eftersom avaktiverad hårdvaruacceleration normalt är det stabilare valet. För den här buggen är det tvärtom, och varför förklaras längre ned i orsakskedjan: Inställningen avgör om en GPU-processkrasch bara kostar ett fallback-steg eller hela appen.

**2. Använd den inbäddade webbläsaren sparsamt.** Sidor i appens webbläsar-/förhandsvisningsområde utlöser kraschen. Den som stänger området efter användning i stället för att låta flikar ligga kvar minskar kraschfrekvensen drastiskt; detta samband är dokumenterat flera gånger med siffror i community-tråden.

**3. Valfritt: stäng av WebGPU.** En start med `--disable-features=WebGPU` förhindrar den vanligaste utlösaren helt. För en Store-app ändras installationssökvägen vid varje uppdatering, därför behövs en launcher som löser den på nytt vid varje start:

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

Nackdelen: Det fungerar bara om appen också startas via denna launcher. Åtgärd 1 fungerar vid varje start.

”Reparera” eller ominstallation av appen löser för övrigt inte problemet, utan behandlar bara följdsymptomet (mer om detta nedan). Även uppdateringar av grafikdrivrutiner är bortkastad möda.

## Självdiagnos: bekräfta fyndet på det egna systemet

Två kontroller räcker. Först kraschsiganaturen i appens logg:

```powershell
Select-String -Path "$env:LOCALAPPDATA\Claude\Logs\main.log" `
  -Pattern 'GPU process gone'
```

För det andra, och detta är det egentliga beviset, Windows CodeIntegrity-logg:

```powershell
Get-WinEvent -FilterHashtable @{
  LogName='Microsoft-Windows-CodeIntegrity/Operational'; Id=3033
} -MaxEvents 30 | Where-Object { $_.Message -match 'claude' } |
  Select-Object TimeCreated, Message
```

På berörda system hittar du där Event 3033-poster vars tidsstämplar stämmer på sekunden med kraschtiderna, med följande meddelande:

```text
Code Integrity determined that a process
(...\WindowsApps\Claude_..._x64__pzs8sxrjxfjjc\app\claude.exe)
attempted to load ...\app\vk_swiftshader.dll that did not meet
the Microsoft signing level requirements.
```

På systemet som undersöktes här stämde sju av sju krascher under tre veckor på sekunden överens med en sådan händelse, inklusive en avsiktligt utlösta kontrollkrasch.

## Den fullständiga orsakskedjan

Kraschen är slutleden i en kedja med fyra länkar, som framgår av två analyser tillsammans: Code Integrity-spåret från community-ärendet [#81698](https://github.com/anthropics/claude-code/issues/81698) och en egen minidump-analys ([#89250](https://github.com/anthropics/claude-code/issues/89250)).

**Länk 1: En sida i den inbäddade webbläsaren behöver mjukvarurendering.** En typisk utlösare är ett WebGPU-anrop (`navigator.gpu.requestAdapter()`), vilket syns i fönsterloggen som denna varning precis före kraschen:

```text
[warn] The powerPreference option is currently ignored
       when calling requestAdapter() on Windows.
```

Om appen körs utan hårdvaruacceleration går vägen oundvikligen via SwiftShader, programvaruimplementeringen av Vulkan: GPU-processen försöker ladda den medföljande `vk_swiftshader.dll`.

**Länk 2: Windows Code Integrity blockerar appens egen DLL.** GPU-processen körs med härdningsprincipen ”MicrosoftSignedOnly” (kan kontrolleras med `Get-ProcessMitigation`). För att en Store-app ska få ladda sina egna tillverkarsignerade DLL:er måste MSIX-paketet innehålla en signaturkatalog `AppxMetadata\CodeIntegrity.cat`. Just denna fil saknas i det levererade paketet, vilket community-medlemmar har visat genom att inspektera MSIX-filen. Följden: signaturkontrollen misslyckas, Windows loggar Event 3033 och avslutar GPU-processen hårt. Exitkoden `0x060C201E` är ett AppX-integritetsfel från Windows-laddaren, inte en Chromium-kod; därför finns den inte i någon Chromium-källkod och därför lämnar GPU-processen inte heller någon kraschdump efter sig – det finns inget undantag att dumpa.

**Länk 3: Chromiums fallback-kedja tar slut.** När GPU-processen kraschar växlar Chromium ned ett renderingssteg: hårdvaru-GL, sedan mjukvaru-GL, därefter en ren displaykompositor. Först när inga steg återstår träder den inbyggda självavslutningen i kraft. I källkoden för den paketerade versionen (Chromium 148.0.7778.280 i Electron 42.9.2) står det ordagrant så här:

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

**Länk 4: Main-processen avslutar sig avsiktligt.** Detta `LOG(FATAL)` är ögonblicket då ”appen kraschar”. Det styrks av en minidump av main-processen: `EXCEPTION_BREAKPOINT` (ett avsiktligt `int3`, inget drivrutinsfel), inte en enda grafikdrivrutins-DLL i processen, och i minnet i klartext:

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

Att denna dump över huvud taget finns var den svåraste delen av analysen: Appens Sentry-integrering konsumerar Crashpad-dumpar vid nästa appstart, skickar dem till tillverkarens telemetri och raderar dem lokalt. Crashpad-mappen är därför alltid tom för användaren. Lösningen är en observatör som är oberoende av appens processträd (startad via WMI, så att appkraschen inte avslutar den) och som genomsöker Crashpad-databasen var 200:e millisekund efter `*.dmp` samt omedelbart kopierar bort träffar innan de raderas. Python-paketet `minidump` hanterar utvärderingen, helt utan WinDbg.

## Varför ”avaktivera hårdvaruacceleration” förvärrar allt

Kedjan förklarar även det mest kontraintuitiva fyndet. Avaktiverad hårdvaruacceleration har här två ödesdigra effekter samtidigt. För det första tvingar den fram SwiftShader-sökvägen, alltså just det DLL-laddningsförsök som Code Integrity blockerar; med aktiv hårdvaruacceleration behövs `vk_swiftshader.dll` däremot knappast någonsin. För det andra startar GPU-processen då redan i den nedre änden av fallback-kedjan: En enda krasch räcker, och länk 4 träder i kraft. Detta förklarar även observationen från community-tråden att ett Code Integrity-block ibland inte får några följder och ibland avslutar appen: Det beror på hur många fallback-steg webbläsarprocessen har kvar.

Särskilt olyckligt är att appen automatiskt kan stänga av hårdvaruacceleration efter problem (`isHardwareAccelerationAutoDisabled`). Avsett som en stabilitetsåtgärd förflyttar detta berörda system till just den konfiguration där nästa krasch kostar hela appen.

## Följdsymptomet: reparationsloopen

Code Integrity-felet har en bieffekt som många drabbade uppfattar som ett separat problem: Windows klassar efter händelsen ibland apppaketet som ”Modified, NeedsRemediation”. Appen startar då inte alls längre förrän den återställs via Inställningar → Appar → Claude → Avancerade alternativ → ”Reparera”. Den som alltså ”ständigt måste reparera” appen ser samma grundproblem, bara en länk längre fram: reparationen åtgärdar paketstatusen, inte orsaken; nästa krasch följer vid nästa blockerade DLL-laddningsförsök.

## Rapporteringsstatus

Paketeringsorsaken har rapporterats som [#81341](https://github.com/anthropics/claude-code/issues/81341), samlingstråden med community-bevisen är [#81698](https://github.com/anthropics/claude-code/issues/81698), och minidump-analysen med förklaringen av fallback-kedjan är [#89250](https://github.com/anthropics/claude-code/issues/89250). Den egentliga lösningen, en komplett signaturkatalog i MSIX-paketet, ligger hos Anthropic. Fram till dess gäller: hårdvaruacceleration på, stäng webbläsarområdet disciplinerat och stäng vid behov av WebGPU med en flagga. På systemet som undersöktes här har appen varit kraschfri sedan åtgärd 1 infördes.

## Källor

1.  [GitHub-ärende #81698: GPU process crash kills entire app](https://github.com/anthropics/claude-code/issues/81698): Samlingstråden med community-bevisen för Code Integrity-kedjan, datapunkter över olika tillverkare och korrelationen med webbläsarpanelen.

2.  [GitHub-ärende #81341: CIG + vk_swiftshader.dll kills GPU process](https://github.com/anthropics/claude-code/issues/81341): Paketeringsorsaken; saknad CodeIntegrity-katalog i MSIX.

3.  [GitHub-ärende #89250: Minidump-analys av appavslutningen](https://github.com/anthropics/claude-code/issues/89250): Den andra halvan av kedjan som beskrivs här, med dumpinsamlingsmetod och förslag på lösningar.

4.  [Chromium-källkod: gpu_data_manager_impl_private.cc (tagg 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): Funktionen IntentionallyCrashBrowserForUnusableGpuProcess och fallback-logiken.

5.  [Electron-dokumentation: child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): Händelsen med vilken en Electron-app kan övervaka GPU-processkrascher och vidta egna motåtgärder.

6.  [Python-paketet minidump](https://pypi.org/project/minidump/): Verktyg för dumpanalysen (exception record, modullista, minnessträngar).

7.  [webgpureport.org](https://webgpureport.org/): WebGPU-diagnossida; användes som minimal utlösare för kontrollkraschen och för jämförelsetestet i aktuell Chromium.
