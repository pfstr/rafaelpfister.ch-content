---
title: "Claude Desktop krasjer stadig: «GPU process gone» med avslutningskode 101457950, årsak og løsning"
navTitle: "Claude Desktop-krasj"
description: "Claude Desktop-appen på Windows avsluttes helt med «GPU process gone: exitCode 101457950» (0x060C201E), ofte etterfulgt av reparasjonsdialogen i Store-appen. Hele årsakskjeden: Code Integrity blokkerer vk_swiftshader.dll, Chromiums reservekjede tømmes, og den innebygde selvavslutningen lukker appen. Med permanent løsning (bytte til den klassiske installasjonen uten MSIX), selvdiagnose via hendelsesloggen og analyse helt ned til minidumpen."
date: "2026-08-25"
kategorie: "Claude"
timeToRead: "10 min lesetid"
themen:
  - claude
slug: "gpu-krasjet-0x060c201e-i-claude-skrivebordsappen-feilsoking-helt-frem-til-minidumpen"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
translationSourceHash: 769984b49b04b65b0b8f8a91ce3b6dd65e2eef1a4212bed32b83422f431a8559
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:27:15.855Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/gpu-krasjet-0x060c201e-i-claude-skrivebordsappen-feilsoking-helt-frem-til-minidumpen
---

Claude Desktop-appen på Windows avsluttes uten feilmelding, alle aktive Claude Code-økter forsvinner, og noen ganger starter appen deretter først igjen etter en «Reparer» via Windows-innstillingene. I apploggen står da denne linjen:

```text
GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
```

101457950 er heksadesimalt `0x060C201E`. Hvis du finner denne signaturen i loggen din, har du kommet til rett sted: Denne artikkelen dokumenterer hele årsakskjeden bak krasjet, umiddelbare tiltak som får appen til å kjøre stabilt igjen, og selvdiagnosen som lar deg bekrefte funnet på ditt eget system på to minutter. MSIX-installasjoner (fra Microsoft Store eller via MSIX-oppsett) på alle GPU-produsenter er berørt, fra Intel-integrert GPU via NVIDIA til AMD; maskinvaren er, for å avsløre så mye, ikke årsaken. Den klassiske installasjonen uten MSIX er ikke berørt, og det er nettopp løsningen.

## Løsningen kort fortalt: Bytt til den klassiske installasjonen

Den egentlige feilen ligger i MSIX-installasjonspakken og kan bare rettes av Anthropic (fortsatt åpen per 27.08.2026, sak [#81341](https://github.com/anthropics/claude-code/issues/81341); også den nåværende versjonen 1.37937.3 er berørt). Den samme appen finnes imidlertid også som en klassisk installasjon uten MSIX, og den er ikke underlagt AppX-signaturkontrollen som avslutter GPU-prosessen. Byttet er dermed det eneste tiltaket som eliminerer krasjet fullstendig; det er bekreftet både i sak [#81341](https://github.com/anthropics/claude-code/issues/81341) og på systemet som er undersøkt her. Funksjonssettet er identisk, og oppdateringsfeeden leverer de samme versjonene for begge variantene.

**Trinn 1: Last ned og kjør den klassiske installatøren.** Nedlastingen på [claude.com/download](https://claude.com/download) gir en Squirrel-installasjon som installerer appen under `%LOCALAPPDATA%\AnthropicClaude` (ingen administratorrettigheter kreves). På kommandolinjen:

```powershell
curl.exe -L -o "$env:USERPROFILE\Downloads\Claude-Setup-x64.exe" `
  "https://storage.googleapis.com/osprey-downloads-c02f6a0d-347c-492b-a752-3e0651722e97/nest-win-x64/Claude-Setup-x64.exe"
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-L` | følger HTTP-videresendinger frem til den faktiske filen |
| `-o <pfad>` | målfil; her nedlastingsmappen |
| `<url>` | offisiell installatørkilde; identisk med målet for nedlastingsvideresendingen fra claude.ai |

</details>

Kontroller signaturen etter nedlastingen (`Get-AuthenticodeSignature`, forventet: `Valid`, utsteder «Anthropic, PBC») og start filen. Installasjonsprogrammet legger først inn en eldre grunnversjon; oppdateringsmekanismen bringer den til gjeldende versjon, enten automatisk ved første oppstart eller umiddelbart med:

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

**Trinn 2: Overta konfigurasjonen.** MSIX-versjonen lagrer innlogging, MCP-serverkonfigurasjon og innstillinger i den virtualiserte beholderen sin; den klassiske appen leser `%APPDATA%\Claude`. Kopier én gang (avslutt MSIX-appen først; variantene kjører uansett ikke samtidig på grunn av en felles enkeltinstanslås):

```powershell
robocopy "$env:LOCALAPPDATA\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude" `
  "$env:APPDATA\Claude" /E /XD Cache "Code Cache" GPUCache claude-code Crashpad logs sentry
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `<quelle>` | konfigurasjonsmappe i den virtualiserte AppData-en til MSIX-pakken |
| `<ziel>` | konfigurasjonsmappe for den klassiske installasjonen |
| `/E` | kopierer alle undermapper, også tomme |
| `/XD <namen>` | hopper over de angitte mappene; her cacher og kjøretidsdata som den nye appen oppretter på nytt selv |

</details>

Chatthistorikk går ikke tapt: Den ligger i claude.ai-kontoen, eller (for Claude Code-økter) under `%USERPROFILE%\.claude`, og er ikke knyttet til appinstallasjonen.

**Trinn 3: Fjern MSIX-pakken.** Ellers vil den krasjende varianten fortsatt starte via gamle snarveier:

```powershell
Get-AppxPackage Claude | Remove-AppxPackage
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Claude` | posisjonsargumentet Name for `Get-AppxPackage`: filtrerer de installerte AppX-/MSIX-pakkene på pakkenavn (jokertegn tillatt) |
| `Remove-AppxPackage` | fjerner pakken som overføres via pipelinen, for gjeldende brukerkonto |

</details>

Startmenyoppføringen «Anthropic → Claude» tilhører deretter den klassiske installasjonen; en eventuell festing på oppgavelinjen må opprettes på nytt.

## Hvis du må beholde MSIX-pakken

Uten bytte gjenstår bare tiltak som reduserer krasjfrekvensen uten å fjerne årsaken:

**Bruk den innebygde nettleseren sparsomt.** Sider i appens nettleser-/forhåndsvisningsområde utløser krasjet. Den som lukker området etter bruk i stedet for å la faner stå åpne, reduserer krasjfrekvensen betydelig; denne sammenhengen er dokumentert med tall flere ganger i fellesskapstråden.

**Slå av WebGPU.** Oppstart med `--disable-features=WebGPU` forhindrer den hyppigste utløseren. Med en MSIX-pakke endres installasjonsbanen ved hver oppdatering, derfor brukes en launcher som finner den på nytt ved hver oppstart:

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
| `for /f "delims="` | behandler kommandoutdata linje for linje; tom `delims=` overtar hele linjen (inkludert mellomrom i banen) i `%%i` |
| `-NoProfile` | starter PowerShell uten profilskript, for rask og reproduserbar oppstart |
| `-Command` | utfører det angitte uttrykket; `(Get-AppxPackage Claude).InstallLocation` returnerer pakkens gjeldende installasjonsbane |
| `start ""` | starter programmet frikoblet fra batch-vinduet; de tomme anførselstegnene er vindustittelen (som her er tom) |
| `--disable-features=WebGPU` | Chromium-bryter: deaktiverer den angitte funksjonen, her WebGPU-API-et |

</details>

Dette virker bare dersom appen også startes via denne launcheren.

I første versjon av denne artikkelen var den første anbefalingen å aktivere maskinvareakselerasjon via `isHardwareAccelerationDisabled: false` i `config.json`. Denne anbefalingen er utdatert: I nåværende versjoner (1.37937.x) finnes flagget ikke lenger, appen starter som standard med aktiv maskinvareakselerasjon, og den krasjer likevel (detaljer i tillegget nedenfor).

«Reparer» eller reinstallering av MSIX-pakken løser for øvrig ikke problemet, det behandler bare følgesymptomet (mer om dette nedenfor). Oppdateringer av grafikkdrivere er også bortkastet arbeid.

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
| `-Path` | filen som skal søkes i, her appens hovedlogg |
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
| `Where-Object { … -match 'claude' }` | beholder bare hendelser der meldingsteksten inneholder appbanen |
| `Select-Object TimeCreated, Message` | reduserer utdataene til tidsstempel og melding for sammenligning med krasjtidspunktene |

</details>

På berørte systemer finner du Event 3033-oppføringer der tidsstempelet samsvarer på sekundet med krasjtidspunktene, med denne meldingen:

```text
Code Integrity determined that a process
(...\WindowsApps\Claude_..._x64__pzs8sxrjxfjjc\app\claude.exe)
attempted to load ...\app\vk_swiftshader.dll that did not meet
the Microsoft signing level requirements.
```

På systemet som er undersøkt her, samsvarte alle syv av syv krasj over tre uker på sekundet med en slik hendelse, inkludert et målrettet utløst kontrollkrasj.

## Den komplette årsakskjeden

Krasjet er siste ledd i en kjede med fire ledd, som to analyser samlet har avdekket: Code Integrity-sporet fra fellesskapssak [#81698](https://github.com/anthropics/claude-code/issues/81698) og en egen minidump-analyse ([#89250](https://github.com/anthropics/claude-code/issues/89250)).

**Ledd 1: En side i den innebygde nettleseren trenger programvarerendering.** En typisk utløser er et WebGPU-kall (`navigator.gpu.requestAdapter()`), synlig i vindusloggen som denne advarselen rett før krasjet:

```text
[warn] The powerPreference option is currently ignored
       when calling requestAdapter() on Windows.
```

Hvis appen kjører uten maskinvareakselerasjon, må den gå via programvare-Vulkan-implementasjonen SwiftShader: GPU-prosessen forsøker å laste den medfølgende `vk_swiftshader.dll`.

**Ledd 2: Windows Code Integrity blokkerer appens egen DLL.** GPU-prosessen kjører med hardeningspolicyen «MicrosoftSignedOnly» (kan kontrolleres med `Get-ProcessMitigation`). For at en Store-app skal kunne laste sine egne, produsentsignerte DLL-er, må MSIX-pakken inneholde en signaturkatalog `AppxMetadata\CodeIntegrity.cat`. Nettopp denne filen mangler i den leverte pakken, slik fellesskapsmedlemmer har dokumentert ved å inspisere MSIX-filen. Følgen er at signaturkontrollen feiler, Windows logger Event 3033 og avslutter GPU-prosessen hardt. Avslutningskoden `0x060C201E` er en AppX-integritetsfeil fra Windows-lasteren, ikke en Chromium-kode; derfor finnes den ikke i noen Chromium-kilde, og derfor etterlater GPU-prosessen heller ingen krasjdump – det finnes ingen exception å dumpe.

**Ledd 3: Chromiums reservekjede tømmes.** Når GPU-prosessen krasjer, går Chromium ett renderingsnivå tilbake: maskinvare-GL, deretter programvare-GL, deretter ren skjermkompositor. Først når ingen nivåer gjenstår, utløses den innebygde selvavslutningen. I kildekoden til den medfølgende versjonen (Chromium 148.0.7778.280 i Electron 42.9.2) står det bokstavelig:

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

**Ledd 4: Hovedprosessen avslutter seg med hensikt.** Dette `LOG(FATAL)` er øyeblikket da «appen krasjer». Dette dokumenteres av en minidump fra hovedprosessen: `EXCEPTION_BREAKPOINT` (en bevisst `int3`, ikke en driverfeil), ikke én eneste grafikkdriver-DLL i prosessen, og i klartekst i minnet:

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

At denne dumpen i det hele tatt eksisterer, var den vanskeligste delen av analysen: Appens Sentry-integrasjon konsumerer Crashpad-dumper ved neste appoppstart, sender dem til produsentens telemetri og sletter dem lokalt. Crashpad-mappen er derfor alltid tom for brukeren. Løsningen er en overvåker som er uavhengig av appens prosesstre (startet via WMI, slik at appkrasjet ikke også avslutter den), og som skanner Crashpad-databasen hvert 200. millisekund etter `*.dmp` og kopierer funn umiddelbart før de slettes. Python-pakken `minidump` håndterer analysen, helt uten WinDbg.

## Hvorfor «deaktiver maskinvareakselerasjon» gjør alt verre

Kjeden forklarer også det mest kontraintuitive funnet. Deaktivert maskinvareakselerasjon har her to fatale effekter samtidig. For det første tvinger den frem SwiftShader-banen, altså nettopp DLL-lastingsforsøket som Code Integrity blokkerer; med aktiv maskinvareakselerasjon blir `vk_swiftshader.dll` derimot nesten aldri nødvendig. For det andre starter GPU-prosessen allerede nederst i reservekjeden: Ett enkelt krasj er nok, og ledd 4 slår til. Dette forklarer også observasjonen fra fellesskapstråden om at en Code Integrity-blokkering noen ganger er uten konsekvenser og andre ganger avslutter appen: Det avhenger av hvor mange reserveledd nettleserprosessen fortsatt har igjen.

Særlig uheldig: Appen hadde en automatisk deaktivering av maskinvareakselerasjon etter problemer (`isHardwareAccelerationAutoDisabled`). Ment som et stabilitetstiltak, førte det berørte systemer rett inn i konfigurasjonen der neste krasj koster hele appen.

## Tillegg 27.08.2026: Maskinvareakselerasjon alene er ikke nok

Den første versjonen av denne artikkelen anbefalte aktiv maskinvareakselerasjon som det mest effektive umiddelbare tiltaket, og i to dager holdt appen seg faktisk krasjfri. Så kom den automatiske oppdateringen til 1.37937.3, og med den tre krasj på én ettermiddag, hvert med den kjente Event 3033 for `vk_swiftshader.dll`. To funn følger av dette:

For det første mangler den manglende signaturkatalogen også i den nåværende MSIX-pakken; grunnproblemet er uendret i 1.37937.3.

For det andre beskytter aktiv maskinvareakselerasjon bare statistisk: Den forlenger reservekjeden, men forhindrer ikke at Chromium under belastning eller etter en maskinvare-GPU-prosessfeil likevel går helt ned til SwiftShader-nivået. Så snart det skjer, blokkerer Code Integrity DLL-en, og kjeden kan likevel tømmes. I tillegg har konfigurasjonsflaggene `isHardwareAccelerationDisabled`/`isHardwareAccelerationAutoDisabled` forsvunnet fra `config.json` i 1.37937.x; innstillingen kan ikke lenger låses der.

Dermed gjensto bare byttet til den klassiske installasjonen beskrevet ovenfor som pålitelig løsning. Siden byttet på systemet som er undersøkt her: samme appversjon, identisk bruk inkludert nettleserområdet, ikke én eneste Event 3033 og ingen krasj.

## Følgesymptomet: reparasjonssløyfen

Code Integrity-feilen har en bivirkning som mange berørte anser som et eget problem: Windows klassifiserer av og til apppakken etter hendelsen som «Modified, NeedsRemediation». Appen starter da ikke i det hele tatt før den tilbakestilles via Innstillinger → Apper → Claude → Avanserte alternativer → «Reparer». Den som altså «stadig må reparere appen», ser det samme grunnproblemet, bare ett ledd lenger: Reparasjonen retter pakkestatusen, ikke årsaken; neste krasj følger ved neste blokkerte DLL-lastingsforsøk.

## Status for rapportene

Pakketeringsårsaken er rapportert som [#81341](https://github.com/anthropics/claude-code/issues/81341), samletråden med fellesskapsdokumentasjonen er [#81698](https://github.com/anthropics/claude-code/issues/81698), minidump-analysen med forklaringen av reservekjeden er [#89250](https://github.com/anthropics/claude-code/issues/89250), og en annen detaljert rapport, inkludert reparasjonssløyfen, er [#80444](https://github.com/anthropics/claude-code/issues/80444). Den egentlige løsningen, en fullstendig signaturkatalog i MSIX-pakken, ligger hos Anthropic og mangler fortsatt i 1.37937.3. Inntil da gjelder følgende: Bytt til den klassiske installasjonen; den som må beholde MSIX-pakken, lukker nettleserområdet konsekvent og deaktiverer WebGPU med flagg ved behov. På systemet som er undersøkt her, har appen vært krasjfri siden byttet til den klassiske installasjonen, uten én eneste ytterligere Event 3033.

## Kilder

1.  [GitHub-sak #81698: GPU process crash kills entire app](https://github.com/anthropics/claude-code/issues/81698): Samletråden med fellesskapsdokumentasjon for Code Integrity-kjeden, datapunkter på tvers av produsenter og korrelasjonen med nettleserpanelet.

2.  [GitHub-sak #81341: CIG + vk_swiftshader.dll kills GPU process](https://github.com/anthropics/claude-code/issues/81341): Pakketeringsårsaken; manglende CodeIntegrity-katalog i MSIX.

3.  [GitHub-sak #89250: Minidump-analyse av appavslutningen](https://github.com/anthropics/claude-code/issues/89250): Den andre halvdelen av kjeden beskrevet her, med dumpfangstmetode og løsningsforslag.

4.  [GitHub-sak #80444: GPU-krasj med forensikk og reparasjonssløyfe](https://github.com/anthropics/claude-code/issues/80444): Detaljert enkeltrapport med tidslinjer, analyse av hendelseslogg og funnet om at hvert krasj setter pakken i tilstanden «Modified».

5.  [Claude Desktop: offisiell nedlastingsside](https://claude.com/download): Kilde for den klassiske Windows-installatøren (x64 og ARM64).

6.  [Chromium-kildekode: gpu_data_manager_impl_private.cc (tagg 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): Funksjonen IntentionallyCrashBrowserForUnusableGpuProcess og reservelogikken.

7.  [Electron-dokumentasjon: child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): Hendelsen som en Electron-app kan bruke til å observere GPU-prosesskrasj og iverksette egne mottiltak.

8.  [Python-pakken minidump](https://pypi.org/project/minidump/): Verktøy for dumpanalysen (exception-record, modulliste, minnestrenger).

9.  [webgpureport.org](https://webgpureport.org/): WebGPU-diagnoseside; brukt som minimal utløser for kontrollkrasjet og til sammenligningstesten i nåværende Chromium.
