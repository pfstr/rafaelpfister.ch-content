---
title: "GPU-kraschen 0x060C201E i Claude-skrivbordsappen: felsökning ända till minidumpen"
navTitle: "GPU-krasch 0x060C201E"
description: "Claude-skrivbordsappen avslutas reproducerbart med ”GPU process gone”. Först ser allt ut som en AMD-drivrutinsbugg, sedan motbevisar egna experiment teorin, och till slut avslöjar en fångad minidump den verkliga orsaken: Chromiums inbyggda självavslutning ”GPU process isn't usable. Goodbye.”"
date: "2026-08-24"
kategorie: "Claude"
timeToRead: "12 min läsning"
themen:
  - claude
slug: "gpu-kraschen-0x060c201e-i-claude-skrivbordsappen-felsokning-anda-till-minidumpen"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
url: https://rafaelpfister.ch/sv/blog/gpu-kraschen-0x060c201e-i-claude-skrivbordsappen-felsokning-anda-till-minidumpen
translationSourceHash: 6bd2b58fe661a5639010e16b417412ca9e85f687bae94531890c8fefaef4050d
translationModel: gpt-5.6-terra
translatedAt: 2026-08-25T04:06:36.693Z
translationReview: automatic
---

Sedan slutet av juli avslutas min Claude-skrivbordsapp i Windows flera gånger om dagen. Ingen dialogruta, inget felmeddelande – appen är bara borta, tillsammans med alla pågående Claude Code-sessioner. Över 25 gånger vid det här laget. Dags att sluta starta om och i stället ta reda på var felet faktiskt uppstår. Så mycket kan avslöjas redan nu: huvudmisstänkten från första stund visar sig vara oskyldig, och den verkliga orsaken står till slut svart på vitt i en minidump som appen egentligen inte ville lämna ut.

## Spåret i loggen

Appen sparar sina loggar under `%LOCALAPPDATA%\Claude\Logs`, medan äldre generationer och konfigurationen finns i den virtualiserade Store-sökvägen `%LOCALAPPDATA%\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude`. I `main.log` står exakt samma sak före varje krasch:

```text
16:01:38 [info] GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
16:03:34 [info] Starting app { appVersion: '1.34493.1', ... }
```

101457950 är `0x060C201E` i hexadecimal form. Kom ihåg det talet – det är buggens signatur. Fönsterloggen ger också utlösaren: omedelbart före varje krasch begär en sida i appens inbäddade webbläsare en WebGPU-adapter.

```text
16:01:38 [warn] The powerPreference option is currently ignored
                when calling requestAdapter() on Windows.
16:01:38 [warn] A valid external Instance reference no longer exists.
14:59:15 [warn] WebGL: CONTEXT_LOST_WEBGL: loseContext: context lost
```

Alltså: `navigator.gpu.requestAdapter()` går i Chromiums GPU-process in i Dawns adapteruppräkning, GPU-processen kraschar och i stället för att appen startar om den avslutas hela applikationen.

## Misstänkt nr 1: grafikdrivrutinen

Maskinen har ett Radeon RX 7900 XT med Adrenalin 32.0.31035.1003, och appen paketerar Electron 42.9.2 med Chromium 148. Den bekväma förklaringen ligger på bordet: gammal Dawn-kod möter RDNA3-drivrutin, drivrutinen kraschar, fallet löst. Bekvämt, plausibelt och, som det ska visa sig: fel. Men först i turordning, för det går bara att motbevisa med experiment.

Två saker föll bort som villospår redan på förhand. Den inaktiverade iGPU:n i Enhetshanteraren (status ”Error”) är helt enkelt kod 22, avsiktligt inaktiverad. Och appen hade redan stängt av hårdvaruacceleration (`isHardwareAccelerationDisabled: true` i config.json), vilket krascherna inte brydde sig om. Varför den inställningen till och med förvärrar problemet framgår först allra sist.

## Experiment 1: kontroll i aktuellt Chromium

Samma belastning, samma maskin, aktuell webbläsare: webgpureport.org i Chromium 151 initierar WebGPU fullt ut, adapter `amd / rdna-3`, inklusive skapande av enhet, utan minsta avvikelse. Den aktuella drivrutinen med aktuell Dawn fungerar alltså korrekt.

## Experiment 2: standard-Electron 42.9.2, hårdvarusökväg

Om Electron 42 inte klarar denna drivrutin måste det gå att återskapa med 20 rader. Alltså: exakt samma Electron-version som i appen som rent testprojekt, ett fönster, en sida, ett `requestAdapter()`:

```js
const { app, BrowserWindow, crashReporter } = require('electron');
crashReporter.start({ submitURL: '', uploadToServer: false });
app.on('child-process-gone', (e, d) =>
  console.log('GONE: ' + JSON.stringify(d)));
app.whenReady().then(() => {
  const win = new BrowserWindow({ show: false });
  win.loadFile('index.html'); // ruft requestAdapter() auf
});
```

Resultat med hårdvaruacceleration: `adapter ok (amd/rdna-3), device ok`. Ingen krasch. Electron 42:s D3D12-sökväg med denna drivrutin fungerar felfritt. Hypotesen ”gammal Dawn-kod är inte kompatibel med RDNA3-drivrutinen” är därmed motbevisad.

## Experiment 3: standard-Electron 42.9.2, programvarusökväg som i appen

Appen kör dock utan hårdvaruacceleration. Alltså samma experiment med `app.disableHardwareAcceleration()`, dessutom en aktiv WebGL-kontext (som körs via SwiftShader i programvaruläge) och `powerPreference: 'high-performance'` vid adapterbegäran, för att återskapa förloppet i apploggarna exakt:

```text
[renderer] webgl context: WebKit WebGL
[renderer] The powerPreference option is currently ignored
           when calling requestAdapter() on Windows.
[renderer] No available adapters.
[renderer] RESULT: adapter=null
TIMEOUT: no crash after 25s
```

Samma powerPreference-varning som i apploggen, samma kodsökväg in i adapteruppräkningen, och sedan rätt svar: ingen adapter tillgänglig, avvisas korrekt, processen lever vidare. Standard-Electron 42.9.2 kraschar helt enkelt inte på denna maskin, oavsett sökväg.

## Experiment 4: annan hårdvara, samma signatur

Innan man fortsätter gissa är det värt att titta i issue-trackern, och där blir det tydligt: den identiska kraschen med identisk exit-kod 0x060C201E har rapporterats flera gånger, bland annat på en NVIDIA RTX 5080 Laptop-GPU. I systemets händelselogg: inga TDR-händelser, inga drivrutinsåterställningar. Drivrutinen, oavsett tillverkare, är inte orsaken. Kraschoraken ligger i appens GPU-process själv, eller snarare, som snart framgår, i appens reaktion på dess krasch.

## Experiment 5: få tag i minidumpen som appen raderar

Hittills saknades det avgörande beviset: en kraschdump. Appens Crashpad-mapp var tom efter varje krasch, vilket först såg ut som avstängd kraschrapportering. Processlistan säger något annat: en `crashpad-handler`-process körs, och dess kommandorad pekar på databasen i Roaming-profilen och på en platshållar-URL för uppladdning. Det är det vanliga mönstret för Sentry-integrering i Electron-appar: Crashpad skriver dumpen lokalt, Sentry-biblioteket konsumerar den vid nästa appstart, skickar den till tillverkarens telemetri och raderar den lokalt. Dumparna finns alltså, bara inte för användaren.

Lösningen är odramatisk: en observatör som är oberoende av appens processträd (startad via WMI så att appkraschen inte tar med sig den), som genomsöker Crashpad-databasen efter `*.dmp` var 200:e millisekund och kopierar undan fynd direkt. Utlös sedan kraschen avsiktligt: öppna webgpureport.org i appens inbäddade webbläsare. Sekunder senare finns en 35 MB stor minidump i säkerhetskopieringsmappen, som Sentry försöker radera ut i tomma intet vid nästa appstart.

## Minidumpen: ingen drivrutin så långt ögat når

Analysen med Python-paketet `minidump` ger tre fynd som vänder bilden helt:

```text
Exception: EXCEPTION_BREAKPOINT (0x80000003)
Adresse:   Claude.exe+0x5e8a6c9
Prozess:   PID 27660
```

För det första: den dumpade processen är inte GPU-processen, utan appens **huvudprocess** (PID:t förekommer i apploggarna som `electron_main`). För det andra: undantaget är en breakpoint, alltså ett avsiktligt utfört `int3`. Så avslutar Chromium sig självt när ett `CHECK()` eller `LOG(FATAL)` inträffar; ett drivrutinsfel skulle se ut som en Access Violation. För det tredje: i processens modullista finns inte en enda grafikdrivrutins-DLL laddad.

Och i dumpens minne står det ödesdigra loggmeddelandet i klartext:

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

## Upplösningen: Chromiums inbyggda självavslutning

Den här raden är inte ett fel, den är avsiktlig. I Chromium-källkoden för den exakt paketerade versionen (148.0.7778.280) står följande i `gpu_data_manager_impl_private.cc`:

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

Den anropas av `FallBackToNextGpuMode()`: kraschar GPU-processen växlar Chromium ned ett steg (hårdvaru-GL → programvaru-GL → ren displaykompositor). Om listan över fallback-lägen är tom avslutar Chromium avsiktligt webbläsarprocessen, eftersom det utan en fungerande GPU-process inte ens längre kan samordna programvarurendering.

Det förklarar också varför appen drabbas så mycket hårdare än en vanlig webbläsare: den startar med avstängd hårdvaruacceleration, alltså redan längst ned i fallback-kedjan. Om en sida i den inbäddade webbläsaren då begär WebGPU och programvaru-GPU-processen kraschar, finns inget steg kvar som Chromium kan falla tillbaka till. Nästa stopp är ”Goodbye”. I en vanlig Chrome med aktiv hårdvaruacceleration kostar samma krasch ett fallback-steg, och webbläsaren fortsätter köra.

Särskilt olyckligt: appkonfigurationen känner till ett fält `isHardwareAccelerationAutoDisabled`, så appen tycks själv stänga av hårdvaruacceleration efter problem. Avsett som en åtgärd mot krascher förkortar det i stället fallback-kedjan och gör den fatala självavslutningen mer sannolik, inte mindre. En skyddsmekanism och en nödbrytare som skärper varandra.

## Vad som återstår av exit-koden

Kvar är GPU-barnprocessen själv, som sätter igång förloppet varje gång. Den lämnar ingen egen kraschrapport, trots att Crashpad-hanteraren bevisligen fungerar (den dumpade huvudprocessen sekunder senare). Det talar för att GPU-processen inte utlöser ett normalt undantag utan avslutas hårt, i stil med `TerminateProcess`, och att den odokumenterade exit-koden 0x060C201E härrör exakt från detta. Dess sista sträcka ligger därmed hos Anthropic: deras Sentry-telemetri tar emot dumparna som raderas lokalt, inklusive frågan om kraschrapporteringen över huvud taget omfattar GPU-processen.

## Workaround och status för rapporterna

Eftersom utlösaren är sidornas WebGPU-begäranden i den inbäddade webbläsaren hjälper det tills en fix finns att stänga av WebGPU med en Chromium-flagga. Vid en Store-installation ändras installationssökvägen vid varje uppdatering, därför löser en liten launcher upp den på nytt vid varje start:

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

Sedan ändringen: inte en enda krasch till. Den fullständiga analysen är rapporterad: laboratorieexperimenten och dubblettreferenserna i det första ärendet, minidump-utvärderingen med orsakskedjan i det andra. De tre meningsfulla korrigeringarna följer direkt av fyndet: klargöra kraschoraken i programvaru-GPU-processen (dumparna för detta finns i tillverkarens telemetri), stänga av WebGPU riktat vid upprepade GPU-krascher i stället för att låta fallback-kedjan ta slut, och ompröva den automatiska avstängningen av hårdvaruacceleration eftersom den förkortar kedjan.

## Tillägg: workarounden räcker inte, lösningen ligger djupare

Redan samma kväll kom nästa krasch, med identisk signatur. Orsaken är enkel: launchern med `--disable-features=WebGPU` fungerar bara när appen också startas via den. Vid den vanliga starten via Start-menyn kör appen utan flaggan, och för en Store-app finns inget rent sätt att permanent ge en normal start kommandoradsflaggor.

Den permanenta lösningen finns dock sedan länge i den här artikelns orsakskedja: den fatala självavslutningen förutsätter att fallback-kedjan är tom, och den är bara tom omedelbart eftersom appen startar med avstängd hårdvaruacceleration. Hårdvaruacceleration ska alltså slås på igen i appens `config.json`:

```json
"isHardwareAccelerationDisabled": false
```

Detta träder i kraft vid nästa appstart och löser båda sidor av problemet på en gång. För det första körs `requestAdapter()` då via hårdvarusökvägen, som på denna maskin bevisligen är stabil (experiment 2: adapter och enhet utan fel). För det andra har Chromium åter fallback-steg i reserv: om GPU-processen ändå skulle krascha någon gång växlar webbläsaren tillbaka till programvarurendering och fortsätter köra, i stället för att avslutas. Den ursprungliga avstängningen av hårdvaruacceleration, sannolikt inställd någon gång som en stabilitetsåtgärd, var i själva verket förutsättningen för kraschen.

Slutsatsen av felsökningen: den mest närliggande förklaringen (”det var drivrutinen”) skulle ha lett till en resultatlös drivrutinsodyssé. Den motbevisades av två timmars laborationer med den verkliga motorversionen, och orsaken hittades först i minidumpen som appen rutinmässigt städar undan. När en GPU-process kraschar bör därför fyra kontroller göras först, innan man skyller på en tillverkare: kontrollen i den aktuella webbläsaren, kontrollen i den rena motorversionen, kontrollen av om annan hårdvara visar samma signatur, och dumpen av processen som faktiskt beslutar om avslutet.

## Källor

1.  [Grundorsak: Chromiums ”GPU process isn't usable. Goodbye.” (GitHub-issue #89250)](https://github.com/anthropics/claude-code/issues/89250): Minidump-analysen i denna artikel som felrapport, inklusive insamlingsmetod och förslag på korrigeringar.

2.  [Egen första rapport med laboratorieresultat (GitHub-issue #89226)](https://github.com/anthropics/claude-code/issues/89226): Experiment 1 till 3 och korrigeringen av AMD-hypotesen, med hänvisningar till dubbletterna.

3.  [GPU process crash kills entire app (GitHub-issue #81698)](https://github.com/anthropics/claude-code/issues/81698): Samma krasch med identisk exit-kod på NVIDIA RTX 5080, utan TDR-händelser; frikänner grafikdrivrutinerna.

4.  [Chromium-källkod: gpu_data_manager_impl_private.cc (tagg 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): Funktionen IntentionallyCrashBrowserForUnusableGpuProcess och fallback-logiken.

5.  [Electron-dokumentation: child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): Händelsen som en Electron-app kan använda för att övervaka GPU-processkrascher och vidta egna åtgärder.

6.  [Python-paketet minidump](https://pypi.org/project/minidump/): Verktyg för dumpanalys (exception-record, modullista, minnessträngar), helt utan WinDbg.

7.  [webgpureport.org](https://webgpureport.org/): WebGPU-diagnossida; användes som minimal utlösare och som kontroll i aktuellt Chromium.
