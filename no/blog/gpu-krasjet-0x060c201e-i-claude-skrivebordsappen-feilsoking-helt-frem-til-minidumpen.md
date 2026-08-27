---
title: "Claude Desktop krasjer stadig: «GPU process gone» med avslutningskode 101457950, årsak og løsning"
navTitle: "Claude Desktop-krasj"
description: "Claude Desktop-appen på Windows avsluttes fullstendig med «GPU process gone: exitCode 101457950» (0x060C201E), ofte etterfulgt av reparasjonsdialogen i Store-appen. Hele årsakskjeden: Code Integrity blokkerer vk_swiftshader.dll, Chromiums fallback-kjede tømmes, og den innebygde selvavslutningen lukker appen. Med umiddelbar løsning, selvdiagnose via hendelseslogg og analyse helt ned til minidumpen."
date: "2026-08-25"
kategorie: "Claude"
timeToRead: "9 min å lese"
themen:
  - claude
slug: "gpu-krasjet-0x060c201e-i-claude-skrivebordsappen-feilsoking-helt-frem-til-minidumpen"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
translationSourceHash: 61bcad89e160ee37f5abd04905ed9e425236f770f9cfcc4448716acbd3569939
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:36:23.091Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/gpu-krasjet-0x060c201e-i-claude-skrivebordsappen-feilsoking-helt-frem-til-minidumpen
---

Claude Desktop-appen på Windows avsluttes uten feilmelding, alle aktive Claude Code-økter forsvinner, og noen ganger starter appen først igjen etter «Reparer» via Windows-innstillingene. I appens logg står da denne linjen:

```text
GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
```

101457950 er heksadesimalt `0x060C201E`. Hvis du finner denne signaturen i loggen din, har du kommet til rett sted: Denne artikkelen dokumenterer hele årsakskjeden bak krasjet, strakstiltakene som får appen til å kjøre stabilt igjen, og selvdiagnosen som lar deg bekrefte funnet på ditt eget system på to minutter. Installasjoner fra Microsoft Store (MSIX) på alle GPU-produsenter er berørt, fra Intel-integrerte GPU-er via NVIDIA til AMD; maskinvaren er, for å si det med én gang, ikke årsaken.

## Løsningen kort fortalt

Den egentlige feilen ligger i appens installasjonspakke og kan bare rettes av Anthropic (fortsatt åpen per 25.08.2026, issue [#81341](https://github.com/anthropics/claude-code/issues/81341)). Inntil da gjør tre tiltak appen stabil, med det mest effektive først:

**1. Aktiver maskinvareakselerasjon.** Kontroller disse to verdiene i filen `%LOCALAPPDATA%\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude\config.json` og sett dem om nødvendig til `false` (avslutt appen først, og start den deretter på nytt):

```json
"isHardwareAccelerationDisabled": false,
"isHardwareAccelerationAutoDisabled": false
```

Det høres paradoksalt ut, siden deaktivert maskinvareakselerasjon vanligvis er det mer stabile valget. For denne feilen er det motsatt, og årsaken forklares lenger ned i årsakskjeden: Innstillingen avgjør om en GPU-prosesskrasj bare koster ett fallback-trinn eller hele appen.

**2. Bruk den innebygde nettleseren sparsomt.** Utløsere for krasjet er sider i appens nettleser-/forhåndsvisningsområde. De som lukker området etter bruk i stedet for å la faner stå åpne, reduserer krasjfrekvensen drastisk; denne sammenhengen er dokumentert flere ganger med tall i fellesskapstråden.

**3. Valgfritt: Slå av WebGPU.** Oppstart med `--disable-features=WebGPU` hindrer den vanligste utløseren fullstendig. For en Store-app endres installasjonsstien ved hver oppdatering, derfor brukes en launcher som finner den på nytt ved hver oppstart:

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

Ulempen: Dette virker bare når appen også startes via denne launcheren. Tiltak 1 virker ved hver oppstart.

«Reparer» eller reinstallasjon av appen løser for øvrig ikke problemet, det kurerer bare følgesymptomet (mer om det nedenfor). Oppdateringer av grafikkdrivere er også bortkastet arbeid.

## Selvdiagnose: bekreft funnet på ditt eget system

To kontroller er nok. Først krasjsignaturen i appens logg:

```powershell
Select-String -Path "$env:LOCALAPPDATA\Claude\Logs\main.log" `
  -Pattern 'GPU process gone'
```

Deretter, og dette er det egentlige beviset, Windows' Code Integrity-logg:

```powershell
Get-WinEvent -FilterHashtable @{
  LogName='Microsoft-Windows-CodeIntegrity/Operational'; Id=3033
} -MaxEvents 30 | Where-Object { $_.Message -match 'claude' } |
  Select-Object TimeCreated, Message
```

På berørte systemer finner du Event 3033-oppføringer der tidsstempelet stemmer på sekundet med krasjtiden, med denne meldingen:

```text
Code Integrity determined that a process
(...\WindowsApps\Claude_..._x64__pzs8sxrjxfjjc\app\claude.exe)
attempted to load ...\app\vk_swiftshader.dll that did not meet
the Microsoft signing level requirements.
```

På systemet som ble undersøkt her, stemte sju av sju krasj over tre uker på sekundet med en slik hendelse, inkludert et målrettet utløst kontrollkrasj.

## Den fullstendige årsakskjeden

Krasjet er siste ledd i en kjede med fire ledd, som to analyser til sammen har avdekket: Code Integrity-sporet fra fellesskaps-issue [#81698](https://github.com/anthropics/claude-code/issues/81698) og en egen minidump-analyse ([#89250](https://github.com/anthropics/claude-code/issues/89250)).

**Ledd 1: En side i den innebygde nettleseren trenger programvaregjengivelse.** En typisk utløser er et WebGPU-kall (`navigator.gpu.requestAdapter()`), synlig i vindusloggen som denne advarselen rett før krasjet:

```text
[warn] The powerPreference option is currently ignored
       when calling requestAdapter() on Windows.
```

Hvis appen kjører uten maskinvareakselerasjon, går veien nødvendigvis via programvare-Vulkan-implementasjonen SwiftShader: GPU-prosessen forsøker å laste den medfølgende `vk_swiftshader.dll`.

**Ledd 2: Windows Code Integrity blokkerer appens egen DLL.** GPU-prosessen kjører med herdepolicyen «MicrosoftSignedOnly» (kan kontrolleres med `Get-ProcessMitigation`). For at en Store-app skal kunne laste sine egne, produsentsignerte DLL-er, må MSIX-pakken inneholde en signaturkatalog `AppxMetadata\CodeIntegrity.cat`. Nettopp denne filen mangler i den leverte pakken, slik fellesskapsmedlemmer har dokumentert ved å inspisere MSIX-filen. Konsekvensen: Signaturkontrollen mislykkes, Windows logger Event 3033 og avslutter GPU-prosessen hardt. Avslutningskoden `0x060C201E` er en AppX-integritetsfeil fra Windows-loaderen, ikke en Chromium-kode; derfor finnes den ikke i noen Chromium-kilde, og derfor etterlater GPU-prosessen heller ingen krasjdump – det finnes ingen exception å dumpe.

**Ledd 3: Chromiums fallback-kjede tømmes.** Når GPU-prosessen krasjer, går Chromium tilbake ett gjengivelsestrinn: maskinvare-GL, deretter programvare-GL, deretter ren display-compositor. Først når ingen trinn gjenstår, aktiveres den innebygde selvavslutningen. I kildekoden til den medfølgende versjonen (Chromium 148.0.7778.280 i Electron 42.9.2) står det ordrett slik:

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

**Ledd 4: Hovedprosessen avslutter seg med hensikt.** Denne `LOG(FATAL)` er øyeblikket da «appen krasjer». Dette er dokumentert av en minidump fra hovedprosessen: `EXCEPTION_BREAKPOINT` (en tilsiktet `int3`, ikke en driverfeil), ikke én eneste grafikkdriver-DLL i prosessen, og i minnet i klartekst:

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

At denne dumpen i det hele tatt eksisterer, var den vanskeligste delen av analysen: Appens Sentry-integrasjon forbruker Crashpad-dumper ved neste appstart, sender dem til produsentens telemetri og sletter dem lokalt. Crashpad-mappen er derfor alltid tom for brukeren. Løsningen er en overvåker uavhengig av appens prosesstre (startet via WMI, slik at appkrasjet ikke også avslutter den), som gjennomsøker Crashpad-databasen hvert 200. millisekund etter `*.dmp` og kopierer funn umiddelbart før de slettes. Python-pakken `minidump` utfører analysen, helt uten WinDbg.

## Hvorfor «deaktivere maskinvareakselerasjon» forverrer alt

Kjeden forklarer også det mest kontraintuitive funnet. Deaktivert maskinvareakselerasjon har her to fatale effekter samtidig. For det første tvinger den fram SwiftShader-banen, altså nettopp DLL-lasteforsøket som Code Integrity blokkerer; med aktiv maskinvareakselerasjon trengs `vk_swiftshader.dll` derimot nesten aldri. For det andre starter GPU-prosessen allerede nederst i fallback-kjeden: Ett enkelt krasj er nok, og ledd 4 aktiveres. Dette forklarer også observasjonen fra fellesskapstråden om at en Code Integrity-blokkering noen ganger er uten konsekvenser og andre ganger avslutter appen: Det avhenger av hvor mange fallback-trinn nettleserprosessen fortsatt har igjen.

Særlig uheldig er det at appen har automatisk deaktivering av maskinvareakselerasjon etter problemer (`isHardwareAccelerationAutoDisabled`). Ment som et stabilitetstiltak fører det berørte systemer rett inn i konfigurasjonen der neste krasj koster hele appen.

## Følgesymptomet: reparasjonssløyfen

Code Integrity-feilen har en bivirkning som mange berørte anser som et eget problem: Windows klassifiserer etter hendelsen av og til apppakken som «Modified, NeedsRemediation». Appen starter da ikke i det hele tatt før du tilbakestiller den via Innstillinger → Apper → Claude → Avanserte alternativer → «Reparer». De som altså «stadig må reparere appen», ser det samme grunnproblemet, bare ett ledd lenger ute i kjeden: Reparasjonen retter pakkestatusen, ikke årsaken; neste krasj kommer ved neste blokkerte DLL-lasteforsøk.

## Status for rapportene

Pakkeringsårsaken er rapportert som [#81341](https://github.com/anthropics/claude-code/issues/81341), samletråden med fellesskapsbevisene er [#81698](https://github.com/anthropics/claude-code/issues/81698), og minidump-analysen med forklaringen av fallback-kjeden er [#89250](https://github.com/anthropics/claude-code/issues/89250). Den egentlige løsningen, en fullstendig signaturkatalog i MSIX-pakken, ligger hos Anthropic. Inntil da gjelder følgende: ha maskinvareakselerasjon på, lukk nettleserområdet disiplinert, og slå ved behov av WebGPU med flagget. På systemet som ble undersøkt her, har appen vært krasjfri siden tiltak 1 ble gjennomført.

## Kilder

1.  [GitHub-issue #81698: GPU process crash kills entire app](https://github.com/anthropics/claude-code/issues/81698): Samletråden med fellesskapsbevisene for Code Integrity-kjeden, datapunktene på tvers av produsenter og korrelasjonen med nettleserpanelet.

2.  [GitHub-issue #81341: CIG + vk_swiftshader.dll kills GPU process](https://github.com/anthropics/claude-code/issues/81341): Pakkeringsårsaken; manglende CodeIntegrity-katalog i MSIX.

3.  [GitHub-issue #89250: Minidump-analyse av appavslutningen](https://github.com/anthropics/claude-code/issues/89250): Den andre halvdelen av kjeden som beskrives her, med dump-capture-metode og løsningsforslag.

4.  [Chromium-kildekode: gpu_data_manager_impl_private.cc (tagg 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): Funksjonen IntentionallyCrashBrowserForUnusableGpuProcess og fallback-logikken.

5.  [Electron-dokumentasjon: child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): Hendelsen som en Electron-app kan bruke til å overvåke GPU-prosesskrasj og iverksette egne mottiltak.

6.  [Python-pakken minidump](https://pypi.org/project/minidump/): Verktøy for dump-analysen (exception-record, modulliste, minnestrenger).

7.  [webgpureport.org](https://webgpureport.org/): WebGPU-diagnoseside; ble brukt som minimal utløser for kontrollkrasjet og for sammenligningstesten i gjeldende Chromium.
