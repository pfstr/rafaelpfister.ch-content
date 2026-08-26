---
title: "Claude Desktop krasjer stadig: «GPU process gone» med avslutningskode 101457950, årsak og løsning"
navTitle: "Claude Desktop-krasj"
description: "Claude Desktop-appen i Windows avsluttes fullstendig med «GPU process gone: exitCode 101457950» (0x060C201E), ofte etterfulgt av reparasjonsdialogen for Store-appen. Den komplette årsakskjeden: Code Integrity blokkerer vk_swiftshader.dll, Chromiums reservekjede går tom, og den innebygde selvavslutningen lukker appen. Med umiddelbar løsning, selvdiagnose via hendelsesloggen og analyse helt ned til minidumpen."
date: "2026-08-25"
kategorie: "Claude"
timeToRead: "9 min å lese"
themen:
  - claude
slug: "gpu-krasjet-0x060c201e-i-claude-skrivebordsappen-feilsoking-helt-frem-til-minidumpen"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
translationSourceHash: 2cf7e9455d4d9b5c148e7b55fd0433206810dc26e53bacb85e1d2dc82a0444c6
translationModel: gpt-5.6-terra
translatedAt: 2026-08-26T04:08:59.772Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/gpu-krasjet-0x060c201e-i-claude-skrivebordsappen-feilsoking-helt-frem-til-minidumpen
---

Claude Desktop-appen i Windows avsluttes uten feilmelding, alle pågående Claude Code-økter forsvinner, og noen ganger starter appen først igjen etter «Reparer» via Windows-innstillingene. I appens logg står da denne linjen:

```text
GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
```

101457950 er heksadesimalt `0x060C201E`. Hvis du finner denne signaturen i loggen din, er du på rett sted: Denne artikkelen dokumenterer hele årsakskjeden bak dette krasjet, tiltakene som umiddelbart gjør appen stabil igjen, og selvdiagnosen som lar deg bekrefte funnet på ditt eget system på to minutter. Installasjoner fra Microsoft Store (MSIX) på alle GPU-produsenter er berørt, fra Intel-integrerte GPU-er via NVIDIA til AMD; maskinvaren er, for å si det med en gang, ikke årsaken.

## Løsningen kort fortalt

Den egentlige feilen ligger i appens installasjonspakke og kan bare rettes av Anthropic (fortsatt åpen per 25.08.2026, sak [#81341](https://github.com/anthropics/claude-code/issues/81341)). Inntil da gjør tre tiltak appen stabil, med det mest effektive først:

**1. Aktiver maskinvareakselerasjon.** Kontroller disse to verdiene i filen `%LOCALAPPDATA%\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude\config.json` og sett dem til `false` ved behov (avslutt appen først, og start den deretter på nytt):

```json
"isHardwareAccelerationDisabled": false,
"isHardwareAccelerationAutoDisabled": false
```

Det høres paradoksalt ut, siden deaktivert maskinvareakselerasjon vanligvis er det mer stabile valget. For denne feilen er det motsatt, og årsakskjeden nedenfor forklarer hvorfor: Innstillingen avgjør om et krasj i GPU-prosessen bare koster ett reservetrinn eller hele appen.

**2. Bruk den innebygde nettleseren sparsomt.** Sider i appens nettleser-/forhåndsvisningsområde utløser krasjet. De som lukker området etter bruk i stedet for å la faner stå åpne, reduserer krasjfrekvensen drastisk; denne sammenhengen er dokumentert flere ganger med tall i fellesskapstråden.

**3. Valgfritt: Slå av WebGPU.** Oppstart med `--disable-features=WebGPU` forhindrer den vanligste utløseren fullstendig. For en Store-app endres installasjonsstien ved hver oppdatering, derfor brukes en launcher som løser den på nytt ved hver oppstart:

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

Ulempen: Dette virker bare når appen også startes via denne launcheren. Tiltak 1 virker ved hver oppstart.

«Reparer» eller nyinstallering av appen løser for øvrig ikke problemet; det behandler bare følgesymptomet (mer om det nedenfor). Oppdateringer av grafikkdrivere er også bortkastet arbeid.

## Selvdiagnose: bekreft funnet på ditt eget system

To kontroller er nok. Først krasjsignaturen i apploggen:

```powershell
Select-String -Path "$env:LOCALAPPDATA\Claude\Logs\main.log" `
  -Pattern 'GPU process gone'
```

For det andre, og dette er det egentlige beviset, Windows' CodeIntegrity-logg:

```powershell
Get-WinEvent -FilterHashtable @{
  LogName='Microsoft-Windows-CodeIntegrity/Operational'; Id=3033
} -MaxEvents 30 | Where-Object { $_.Message -match 'claude' } |
  Select-Object TimeCreated, Message
```

På berørte systemer finner du Event 3033-oppføringer der med tidsstempler som stemmer på sekundet med krasjtidspunktene, med denne meldingen:

```text
Code Integrity determined that a process
(...\WindowsApps\Claude_..._x64__pzs8sxrjxfjjc\app\claude.exe)
attempted to load ...\app\vk_swiftshader.dll that did not meet
the Microsoft signing level requirements.
```

På systemet som ble undersøkt her, samsvarte sju av sju krasj i løpet av tre uker på sekundet med en slik hendelse, inkludert et målrettet utløst kontrollkrasj.

## Den komplette årsakskjeden

Krasjet er siste ledd i en kjede med fire ledd, som fremkommer av to analyser sammen: Code Integrity-sporet fra fellesskapssaken [#81698](https://github.com/anthropics/claude-code/issues/81698) og en egen minidump-analyse ([#89250](https://github.com/anthropics/claude-code/issues/89250)).

**Ledd 1: En side i den innebygde nettleseren trenger programvaregjengivelse.** Den typiske utløseren er et WebGPU-kall (`navigator.gpu.requestAdapter()`), som kan gjenkjennes i vindusloggen på denne advarselen rett før krasjet:

```text
[warn] The powerPreference option is currently ignored
       when calling requestAdapter() on Windows.
```

Når appen kjører uten maskinvareakselerasjon, går veien nødvendigvis via programvare-Vulkan-implementasjonen SwiftShader: GPU-prosessen forsøker å laste den medfølgende `vk_swiftshader.dll`.

**Ledd 2: Windows Code Integrity blokkerer appens egen DLL.** GPU-prosessen kjører med hardeningspolicyen «MicrosoftSignedOnly» (kan kontrolleres med `Get-ProcessMitigation`). For at en Store-app skal kunne laste sine egne, produsentsignerte DLL-er, må MSIX-pakken inneholde en signaturkatalog `AppxMetadata\CodeIntegrity.cat`. Nettopp denne filen mangler i den leverte pakken, slik medlemmer av fellesskapet har dokumentert ved å inspisere MSIX-filen. Følgen: Signaturkontrollen mislykkes, Windows logger Event 3033 og avslutter GPU-prosessen hardt. Avslutningskoden `0x060C201E` er en AppX-integritetsfeil fra Windows-lasteren, ikke en Chromium-kode; derfor finnes den ikke i noen Chromium-kilde, og derfor etterlater heller ikke GPU-prosessen noen krasjdump – det finnes ingen exception å dumpe.

**Ledd 3: Chromiums reservekjede går tom.** Når GPU-prosessen krasjer, går Chromium ett trinn tilbake i gjengivelsen: maskinvare-GL, deretter programvare-GL, deretter ren display-kompositor. Først når ingen trinn gjenstår, aktiveres den innebygde selvavslutningen. I kildekoden til den medfølgende versjonen (Chromium 148.0.7778.280 i Electron 42.9.2) står det ordrett slik:

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

**Ledd 4: Hovedprosessen avslutter seg med vilje.** Dette `LOG(FATAL)` er øyeblikket da «appen krasjer». Det er dokumentert av en minidump fra hovedprosessen: `EXCEPTION_BREAKPOINT` (en bevisst `int3`, ikke en driverfeil), ikke én eneste grafikkdriver-DLL i prosessen, og i minnet i klartekst:

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

At denne dumpen i det hele tatt eksisterer, var den vanskeligste delen av analysen: Appens Sentry-integrasjon henter Crashpad-dumper ved neste appoppstart, sender dem til produsentens telemetri og sletter dem lokalt. Crashpad-mappen er derfor alltid tom for brukeren. En observatør som er uavhengig av appens prosesstre (startet via WMI, slik at appkrasjet ikke også avslutter den) løser dette. Den gjennomsøker Crashpad-databasen hvert 200. millisekund etter `*.dmp` og kopierer funn umiddelbart før de slettes. Python-pakken `minidump` står for analysen, helt uten WinDbg.

## Hvorfor «deaktivere maskinvareakselerasjon» gjør alt verre

Kjeden forklarer også det mest kontraintuitive funnet. Deaktivert maskinvareakselerasjon har to fatale effekter samtidig her. For det første tvinger den frem SwiftShader-stien, altså nettopp DLL-lasteforsøket som Code Integrity blokkerer; med aktiv maskinvareakselerasjon trengs `vk_swiftshader.dll` derimot nesten aldri. For det andre starter GPU-prosessen allerede nederst i reservekjeden: Ett eneste krasj er nok, og ledd 4 slår inn. Dette forklarer også observasjonen fra fellesskapstråden om at en Code Integrity-blokkering noen ganger ikke får noen konsekvenser og andre ganger avslutter appen: Det avhenger av hvor mange reservetrinn nettleserprosessen fortsatt har igjen.

Særlig uheldig: Appen har en automatisk deaktivering av maskinvareakselerasjon etter problemer (`isHardwareAccelerationAutoDisabled`). Den er ment som et stabilitetstiltak, men flytter berørte systemer rett inn i konfigurasjonen der neste krasj koster hele appen.

## Følgesymptomet: reparasjonssløyfen

Code Integrity-feilen har en bieffekt som mange berørte oppfatter som et eget problem: Windows klassifiserer av og til apppakken som «Modified, NeedsRemediation» etter hendelsen. Appen starter da ikke i det hele tatt før den tilbakestilles via Innstillinger → Apper → Claude → Avanserte alternativer → «Reparer». De som altså «stadig må reparere appen», ser det samme grunnproblemet, bare ett ledd videre: Reparasjonen retter pakkestatusen, ikke årsaken; neste krasj følger ved neste blokkerte DLL-lasteforsøk.

## Status for rapportene

Pakkeringsårsaken er rapportert som [#81341](https://github.com/anthropics/claude-code/issues/81341), samlingstråden med dokumentasjon fra fellesskapet er [#81698](https://github.com/anthropics/claude-code/issues/81698), og minidump-analysen med forklaringen av reservekjeden er [#89250](https://github.com/anthropics/claude-code/issues/89250). Den egentlige løsningen, en komplett signaturkatalog i MSIX-pakken, ligger hos Anthropic. Inntil da gjelder følgende: maskinvareakselerasjon på, lukk nettleserområdet disiplinert, og slå ved behov av WebGPU med flagget. På systemet som ble undersøkt her, har appen vært krasjfri siden tiltak 1 ble gjennomført.

## Kilder

1.  [GitHub-sak #81698: GPU process crash kills entire app](https://github.com/anthropics/claude-code/issues/81698): Samlingstråden med dokumentasjon fra fellesskapet om Code Integrity-kjeden, datapunkter på tvers av produsenter og korrelasjonen med nettleserpanelet.

2.  [GitHub-sak #81341: CIG + vk_swiftshader.dll kills GPU process](https://github.com/anthropics/claude-code/issues/81341): Pakkeringsårsaken; manglende CodeIntegrity-katalog i MSIX.

3.  [GitHub-sak #89250: Minidump-analyse av appavslutningen](https://github.com/anthropics/claude-code/issues/89250): Den andre halvdelen av kjeden som beskrives her, med metode for dumpopptak og forslag til løsninger.

4.  [Chromium-kildekode: gpu_data_manager_impl_private.cc (tagg 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): Funksjonen IntentionallyCrashBrowserForUnusableGpuProcess og reservelogikken.

5.  [Electron-dokumentasjon: child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): Hendelsen som lar en Electron-app observere krasj i GPU-prosesser og iverksette egne mottiltak.

6.  [Python-pakken minidump](https://pypi.org/project/minidump/): Verktøy for dumpanalyse (exception-record, modulliste, minnestrenger).

7.  [webgpureport.org](https://webgpureport.org/): WebGPU-diagnoseside; brukt som minimal utløser for kontrollkrasjet og for sammenligningstesten i gjeldende Chromium.
