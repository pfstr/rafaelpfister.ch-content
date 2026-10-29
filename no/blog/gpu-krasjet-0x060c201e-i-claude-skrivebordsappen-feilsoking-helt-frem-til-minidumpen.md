---
title: "Claude Desktop krasjer stadig: «GPU process gone» med avslutningskode 101457950, årsak og løsning"
navTitle: "Claude Desktop-krasj"
description: "Claude Desktop-appen på Windows avsluttes fullstendig med «GPU process gone: exitCode 101457950» (0x060C201E), ofte etterfulgt av reparasjonsdialogen i Store-appen. Hele årsakskjeden: Code Integrity blokkerer vk_swiftshader.dll, Chromiums reservekjede går tom, og den innebygde selvavslutningen lukker appen. Med en permanent løsning (bytte til den klassiske installasjonen uten MSIX), selvdiagnose via hendelseslogg og analyse helt ned til minidumpen."
date: "2026-08-25"
kategorie: "Claude"
timeToRead: "10 min lesetid"
themen:
  - claude
slug: "gpu-krasjet-0x060c201e-i-claude-skrivebordsappen-feilsoking-helt-frem-til-minidumpen"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
translationSourceHash: 9bd177698f756b588c97f11ea4c73ea48efcfb96c8dbe1435f5ff9fc11dffec0
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:32:13.101Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/gpu-krasjet-0x060c201e-i-claude-skrivebordsappen-feilsoking-helt-frem-til-minidumpen
---

Claude Desktop-appen på Windows avsluttes uten feilmelding, alle pågående Claude Code-økter forsvinner, og av og til starter appen først igjen etter «Reparer» via Windows-innstillingene. I appens logg står da denne linjen:

```text
GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
```

101457950 er heksadesimalt `0x060C201E`. Hvis du finner denne signaturen i loggen din, har du kommet rett: Denne artikkelen dokumenterer hele årsakskjeden bak krasjet, de umiddelbare tiltakene som får appen til å kjøre stabilt igjen, og selvdiagnosen som lar deg bekrefte funnet på ditt eget system på to minutter. MSIX-installasjoner (fra Microsoft Store eller via MSIX-oppsett) på alle GPU-produsenter er berørt, fra Intel-integrerte GPU-er via NVIDIA til AMD; maskinvaren er, for å røpe det på forhånd, ikke årsaken. Den klassiske installasjonen uten MSIX er ikke berørt, og nettopp det er løsningen.

## Løsningen kort fortalt: Bytt til den klassiske installasjonen

Selve feilen ligger i MSIX-installasjonspakken og kan bare rettes av Anthropic (issue [#81341](https://github.com/anthropics/claude-code/issues/81341) er fortsatt åpent per 25.10.2026; tidlig i september 2026 bekreftet rapporter i issuet feilen også i versjon 1.40609.1). Den samme appen finnes imidlertid også som klassisk installasjon uten MSIX, og den er ikke underlagt AppX-signaturkontrollen som avslutter GPU-prosessen. Byttet er dermed det eneste tiltaket som fjerner krasjet fullstendig; det er bekreftet både i issue [#81341](https://github.com/anthropics/claude-code/issues/81341) og på systemet som er undersøkt her. Funksjonsutvalget er identisk, og oppdateringsfeeden leverer de samme versjonene for begge variantene.

**Trinn 1: Last ned og kjør det klassiske installasjonsprogrammet.** Nedlastingen på [claude.com/download](https://claude.com/download) gir et Squirrel-installasjonsprogram som installerer appen i `%LOCALAPPDATA%\AnthropicClaude` (ingen administratorrettigheter kreves). Fra kommandolinjen:

```powershell
curl.exe -L -o "$env:USERPROFILE\Downloads\Claude-Setup-x64.exe" `
  "https://storage.googleapis.com/osprey-downloads-c02f6a0d-347c-492b-a752-3e0651722e97/nest-win-x64/Claude-Setup-x64.exe"
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-L` | følger HTTP-viderekoblinger frem til den faktiske filen |
| `-o <pfad>` | målfil; her nedlastingsmappen |
| `<url>` | offisiell installasjonskilde; identisk med målet for nedlastingsviderekoblingen fra claude.ai |

</details>

Kontroller signaturen etter nedlastingen (`Get-AuthenticodeSignature`, forventet: `Valid`, utsteder «Anthropic, PBC») og start filen. Installasjonsprogrammet legger først inn en eldre basisversjon; oppdateringsmekanismen bringer den til gjeldende versjon, enten automatisk ved første oppstart eller straks med:

```powershell
& "$env:LOCALAPPDATA\AnthropicClaude\Update.exe" `
  --update https://downloads.claude.ai/releases/win32/x64
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `--update <url>` | laster ned den nyeste versjonen fra den angitte utgivelsesfeeden og installerer den som en ny `app-<version>`-mappe |

</details>

**Trinn 2: Overfør konfigurasjonen.** MSIX-versjonen lagrer innlogging, MCP-serverkonfigurasjon og innstillinger i den virtualiserte beholderen sin; den klassiske appen leser `%APPDATA%\Claude`. Kopier én gang (avslutt MSIX-appen først; de to variantene kjører uansett ikke samtidig på grunn av en felles enkeltinstanslås):

```powershell
robocopy "$env:LOCALAPPDATA\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude" `
  "$env:APPDATA\Claude" /E /XD Cache "Code Cache" GPUCache claude-code Crashpad logs sentry
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `<quelle>` | konfigurasjonsmappe i det virtualiserte AppData-området til MSIX-pakken |
| `<ziel>` | konfigurasjonsmappe for den klassiske installasjonen |
| `/E` | kopierer alle underkataloger, også tomme |
| `/XD <namen>` | hopper over de angitte mappene; her cacher og kjøretidsdata som den nye appen oppretter på nytt selv |

</details>

Chatthistorikk går ikke tapt: Den ligger i claude.ai-kontoen eller (for Claude Code-økter) under `%USERPROFILE%\.claude` og er ikke knyttet til appinstallasjonen.

**Trinn 3: Fjern MSIX-pakken.** Ellers starter den krasjende varianten fortsatt via gamle snarveier:

```powershell
Get-AppxPackage Claude | Remove-AppxPackage
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Claude` | posisjonsargumentet Name for `Get-AppxPackage`: filtrerer installerte AppX-/MSIX-pakker etter pakkenavn (jokertegn tillatt) |
| `Remove-AppxPackage` | fjerner pakken som overføres via pipelinen, for gjeldende brukerkonto |

</details>

Startmenyoppføringen «Anthropic → Claude» tilhører deretter den klassiske installasjonen; en eventuell festing til oppgavelinjen må settes på nytt.

## Hvis du må beholde MSIX-pakken

Uten bytte gjenstår bare tiltak som reduserer krasjfrekvensen uten å fjerne årsaken:

**Bruk den innebygde nettleseren sparsommelig.** Utløsere for krasjet er sider i appens nettleser-/forhåndsvisningsområde. De som lukker området etter bruk i stedet for å la faner stå parkert, reduserer krasjfrekvensen betydelig; denne sammenhengen er dokumentert flere ganger med tall i fellesskapstråden.

**Slå av WebGPU.** Oppstart med `--disable-features=WebGPU` hindrer den vanligste utløseren. For en MSIX-pakke endres installasjonsstien ved hver oppdatering, derfor brukes en starter som løser den på nytt ved hver oppstart:

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `for /f "delims="` | behandler kommandoutdata linje for linje; tomme `delims=` overtar hele linjen (inkludert mellomrom i stien) til `%%i` |
| `-NoProfile` | starter PowerShell uten profilskript, for rask og reproduserbar oppstart |
| `-Command` | kjører det angitte uttrykket; `(Get-AppxPackage Claude).InstallLocation` gir pakkens gjeldende installasjonssti |
| `start ""` | starter programmet frakoblet fra batchvinduet; de tomme anførselstegnene er vindustittelen (her tom) |
| `--disable-features=WebGPU` | Chromium-bryter: deaktiverer den angitte funksjonen, her WebGPU-API-et |

</details>

Dette virker bare dersom appen også startes via denne starteren.

I førsteutgaven av denne artikkelen sto anbefalingen om å aktivere maskinvareakselerasjon via `isHardwareAccelerationDisabled: false` i `config.json` først. Denne anbefalingen er foreldet: I gjeldende versjoner (1.37937.x) finnes flagget ikke lenger, appen starter som standard med aktiv maskinvareakselerasjon, og den krasjer likevel (detaljer i tillegget nedenfor).

«Reparer» eller nyinstallering av MSIX-pakken løser for øvrig ikke problemet; det kurerer bare følgesymptomet (mer om det nedenfor). Oppdateringer av grafikkdrivere er også forgjeves.

## Selvdiagnose: Bekreft funnet på eget system

To kontroller er nok. Først krasjsignaturen i apploggen:

```powershell
Select-String -Path "$env:LOCALAPPDATA\Claude\Logs\main.log" `
  -Pattern 'GPU process gone'
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-Path` | filen som skal gjennomsøkes, her appens hovedlogg |
| `-Pattern` | søkemønster (regulært uttrykk); viser alle linjer med krasjsignaturen |

</details>

For det andre, og dette er det egentlige beviset, Windows' CodeIntegrity-logg:

```powershell
Get-WinEvent -FilterHashtable @{
  LogName='Microsoft-Windows-CodeIntegrity/Operational'; Id=3033
} -MaxEvents 30 | Where-Object { $_.Message -match 'claude' } |
  Select-Object TimeCreated, Message
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-FilterHashtable` | filtrerer allerede ved henting: `LogName` angir hendelsesloggen, `Id` hendelses-ID 3033 (Code Integrity-blokkering) |
| `-MaxEvents 30` | begrenser spørringen til de 30 nyeste treffene |
| `Where-Object { … -match 'claude' }` | beholder bare hendelser der meldingsteksten inneholder appstien |
| `Select-Object TimeCreated, Message` | reduserer utdataene til tidsstempel og melding for sammenligning med krasjtidspunktene |

</details>

På berørte systemer finner du Event 3033-oppføringer der tidsstempelet stemmer på sekundet med krasjtidspunktene, med denne meldingen:

```text
Code Integrity determined that a process
(...\WindowsApps\Claude_..._x64__pzs8sxrjxfjjc\app\claude.exe)
attempted to load ...\app\vk_swiftshader.dll that did not meet
the Microsoft signing level requirements.
```

På systemet som er undersøkt her, samsvarte sju av sju krasj over tre uker på sekundet med en slik hendelse, inkludert et målrettet utløst kontrollkrasj.

## Hele årsakskjeden

Krasjet er siste ledd i en kjede med fire ledd, som to analyser til sammen avdekker: Code Integrity-sporet fra fellesskapsissue [#81698](https://github.com/anthropics/claude-code/issues/81698) og en egen minidump-analyse ([#89250](https://github.com/anthropics/claude-code/issues/89250)).

**Ledd 1: En side i den innebygde nettleseren trenger programvaregjengivelse.** En typisk utløser er et WebGPU-kall (`navigator.gpu.requestAdapter()`), synlig i vindusloggen som denne advarselen rett før krasjet:

```text
[warn] The powerPreference option is currently ignored
       when calling requestAdapter() on Windows.
```

Når appen kjører uten maskinvareakselerasjon, går veien nødvendigvis via programvare-Vulkan-implementasjonen SwiftShader: GPU-prosessen forsøker å laste den medfølgende `vk_swiftshader.dll`.

**Ledd 2: Windows Code Integrity blokkerer appens egen DLL.** GPU-prosessen kjører med herdepolicyen «MicrosoftSignedOnly» (kan kontrolleres via `Get-ProcessMitigation`). For at en Store-app skal kunne laste sine egne produsentsignerte DLL-er, må MSIX-pakken inneholde en signaturkatalog `AppxMetadata\CodeIntegrity.cat`. Nettopp denne filen mangler i den leverte pakken, slik fellesskapsmedlemmer har dokumentert ved å inspisere MSIX-filen. Følgen er at signaturkontrollen feiler, Windows logger Event 3033 og avslutter GPU-prosessen hardt. Avslutningskoden `0x060C201E` er en AppX-integritetsfeil fra Windows-lasteren, ikke en Chromium-kode; derfor finnes den ikke i noen Chromium-kilde, og derfor etterlater GPU-prosessen heller ingen krasjdump – det finnes ingen exception å dumpe.

**Ledd 3: Chromiums reservekjede går tom.** Når GPU-prosessen krasjer, går Chromium ett trinn ned i gjengivelsen: maskinvare-GL, deretter programvare-GL, deretter ren skjermkompositor. Først når ingen trinn er igjen, utløses den innebygde selvavslutningen. I kildekoden til den medfølgende versjonen (Chromium 148.0.7778.280 i Electron 42.9.2) står den ordrett slik:

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

**Ledd 4: Hovedprosessen avslutter seg med vilje.** Dette `LOG(FATAL)` er øyeblikket der «appen krasjer». Det dokumenteres av en minidump av hovedprosessen: `EXCEPTION_BREAKPOINT` (en bevisst `int3`, ikke en driverfeil), ikke én eneste grafikkdriver-DLL i prosessen, og i minnet som klartekst:

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

At denne dumpen i det hele tatt finnes, var den vanskeligste delen av analysen: Appens Sentry-integrasjon forbruker Crashpad-dumper ved neste appstart, sender dem til produsentens telemetri og sletter dem lokalt. Crashpad-mappen er derfor alltid tom for brukeren. Løsningen er en overvåker uavhengig av appens prosesstre (startet via WMI, slik at appkrasjet ikke også avslutter den), som gjennomsøker Crashpad-databasen hvert 200. millisekund etter `*.dmp` og kopierer funn umiddelbart bort før de slettes. Python-pakken `minidump` står for analysen, helt uten WinDbg.

## Hvorfor «deaktiver maskinvareakselerasjon» gjør alt verre

Kjeden forklarer også det mest kontraintuitive funnet. Deaktivert maskinvareakselerasjon har her to fatale virkninger samtidig. For det første tvinger den SwiftShader-stien, altså nettopp DLL-lasteforsøket som Code Integrity blokkerer; med aktiv maskinvareakselerasjon blir `vk_swiftshader.dll` derimot nesten aldri nødvendig. For det andre starter GPU-prosessen da allerede nederst i reservekjeden: Ett eneste krasj er nok, og ledd 4 slår inn. Dette forklarer også observasjonen fra fellesskapstråden om at en Code Integrity-blokkering noen ganger ikke får følger og andre ganger avslutter appen: Det avhenger av hvor mange reservetrinn nettleserprosessen har igjen.

Særlig uheldig: Appen kjente til automatisk deaktivering av maskinvareakselerasjon etter problemer (`isHardwareAccelerationAutoDisabled`). Dette var ment som et stabilitetstiltak, men førte berørte systemer rett inn i konfigurasjonen der neste krasj koster hele appen.

## Tillegg 27.08.2026: Maskinvareakselerasjon alene er ikke nok

Førsteutgaven av denne artikkelen anbefalte aktiv maskinvareakselerasjon som det mest effektive umiddelbare tiltaket, og i to dager forble appen faktisk krasjfri med dette. Deretter kom den automatiske oppdateringen til 1.37937.3, og med den tre krasj på én ettermiddag, alle med den kjente Event 3033 for `vk_swiftshader.dll`. To funn fra dette:

For det første mangler den manglende signaturkatalogen også i den gjeldende MSIX-pakken; grunnproblemet er uendret i 1.37937.3.

For det andre beskytter aktiv maskinvareakselerasjon bare statistisk: Den forlenger reservekjeden, men hindrer ikke at Chromium under belastning eller etter en feil i maskinvare-GPU-prosessen likevel går gjennom den helt til SwiftShader-trinnet. Så snart dette skjer, blokkerer Code Integrity DLL-en, og kjeden kan likevel gå tom. I tillegg har konfigurasjonsflaggene `isHardwareAccelerationDisabled`/`isHardwareAccelerationAutoDisabled` forsvunnet fra `config.json` i 1.37937.x; innstillingen kan ikke lenger låses der.

Dermed gjensto bare byttet til den klassiske installasjonen beskrevet ovenfor som en pålitelig løsning. Siden byttet på systemet som er undersøkt her: samme appversjon, identisk bruk inkludert nettleserområdet, ikke én eneste Event 3033 og intet krasj.

## Følgesymptomet: Reparasjonssløyfen

Code Integrity-feilen har en bivirkning som mange berørte anser som et eget problem: Windows klassifiserer tidvis apppakken etter hendelsen som «Modified, NeedsRemediation». Appen starter da ikke i det hele tatt før du tilbakestiller den via Innstillinger → Apper → Claude → Avanserte alternativer → «Reparer». Den som altså «stadig må reparere appen», ser det samme grunnproblemet, bare ett ledd videre: Reparasjonen fikser pakkestatusen, ikke årsaken; neste krasj følger ved neste blokkerte DLL-lasteforsøk.

## Status for rapportene

Pakketeringsårsaken er rapportert som [#81341](https://github.com/anthropics/claude-code/issues/81341), samletråden med belegg fra fellesskapet er [#81698](https://github.com/anthropics/claude-code/issues/81698), minidump-analysen med forklaringen av reservekjeden er [#89250](https://github.com/anthropics/claude-code/issues/89250), og en annen utførlig rapport, inkludert reparasjonssløyfen, er [#80444](https://github.com/anthropics/claude-code/issues/80444). Selve rettelsen, en komplett signaturkatalog i MSIX-pakken, ligger hos Anthropic. Per 25.10.2026 er #81341 fortsatt åpent, og rapporter fra tidlig i september 2026 bekrefter feilen også i versjon 1.40609.1. #80444 ble lukket 15. september 2026 uten angivelse av en rettet versjon, #89250 20. oktober 2026 automatisk på grunn av inaktivitet; ingen av de to issuene nevner en rettelse. Inntil da gjelder følgende: Bytt til den klassiske installasjonen; de som må beholde MSIX-pakken, lukker nettleserområdet disiplinert og slår ved behov av WebGPU via flagg. På systemet som er undersøkt her, har appen vært krasjfri siden byttet til den klassiske installasjonen, uten én eneste ytterligere Event 3033.

## Endringslogg

- **25. oktober 2026:** Lagt til: Issue #81341 er fortsatt åpent per 25.10.2026, og feilen er også bekreftet i versjon 1.40609.1. Issue #80444 og #89250 ble lukket uten å nevne en rettelse.

## Kilder

1.  [GitHub-issue #81698: GPU process crash kills entire app](https://github.com/anthropics/claude-code/issues/81698): Samletråden med fellesskapets belegg for Code Integrity-kjeden, datapunkter på tvers av produsenter og korrelasjonen med nettleserpanelet.

2.  [GitHub-issue #81341: CIG + vk_swiftshader.dll kills GPU process](https://github.com/anthropics/claude-code/issues/81341): Pakketeringsårsaken; manglende CodeIntegrity-katalog i MSIX.

3.  [GitHub-issue #89250: Minidump-analyse av appavslutningen](https://github.com/anthropics/claude-code/issues/89250): Andre halvdel av kjeden som beskrives her, med dumpopptaksmetode og forslag til rettelser.

4.  [GitHub-issue #80444: GPU-krasj med forensikk og reparasjonssløyfe](https://github.com/anthropics/claude-code/issues/80444): Utførlig enkeltreport med tidslinjer, hendelseslogganalyse og funnet om at hvert krasj setter pakken i tilstanden «Modified».

5.  [Claude Desktop: offisiell nedlastingsside](https://claude.com/download): Kilde for det klassiske Windows-installasjonsprogrammet (x64 og ARM64).

6.  [Chromium-kildekode: gpu_data_manager_impl_private.cc (tagg 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): Funksjonen IntentionallyCrashBrowserForUnusableGpuProcess og reservelogikken.

7.  [Electron-dokumentasjon: child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): Hendelsen som lar en Electron-app overvåke krasj i GPU-prosessen og iverksette egne mottiltak.

8.  [Python-pakken minidump](https://pypi.org/project/minidump/): Verktøy for dumpanalysen (exception-record, modulliste, minnestrenger).

9.  [webgpureport.org](https://webgpureport.org/): WebGPU-diagnoseside; ble brukt som minimal utløser for kontrollkrasjet og for sammenligningstesten i gjeldende Chromium.
