---
title: "GPU-krasjet 0x060C201E i Claude-skrivebordsappen: feilsøking helt frem til minidumpen"
navTitle: "GPU-krasj 0x060C201E"
description: "Claude-skrivebordsappen avsluttes reproduserbart med «GPU process gone». Først ser alt ut som en AMD-driverfeil, så motbeviser egne eksperimenter teorien, og til slutt gir en oppfanget minidump den faktiske årsaken: Chromiums innebygde selvavbrudd «GPU process isn't usable. Goodbye.»"
date: "2026-08-25"
kategorie: "Claude"
timeToRead: "12 min å lese"
themen:
  - claude
slug: "gpu-krasjet-0x060c201e-i-claude-skrivebordsappen-feilsoking-helt-frem-til-minidumpen"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
url: https://rafaelpfister.ch/no/blog/gpu-krasjet-0x060c201e-i-claude-skrivebordsappen-feilsoking-helt-frem-til-minidumpen
translationSourceHash: 6bd2b58fe661a5639010e16b417412ca9e85f687bae94531890c8fefaef4050d
translationModel: gpt-5.6-terra
translatedAt: 2026-08-25T04:07:49.521Z
translationReview: required
---

Siden slutten av juli har Claude-skrivebordsappen min under Windows avsluttet seg flere ganger daglig. Ingen dialog, ingen feilmelding, appen er bare borte, sammen med alle pågående Claude Code-økter. Over 25 ganger nå. På tide å slutte å starte på nytt og heller undersøke hvor feilen faktisk oppstår. Så mye på forhånd: Hovedmistenkte i første omgang viser seg å være uskyldig, og den faktiske årsaken står til slutt svart på hvitt i en minidump som appen egentlig ikke ville utlevere.

## Sporet i loggen

Appen lagrer loggene sine under `%LOCALAPPDATA%\Claude\Logs`, eldre generasjoner og konfigurasjonen ligger i den virtualiserte Store-banen `%LOCALAPPDATA%\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude`. I `main.log` står nøyaktig det samme før hvert krasj:

```text
16:01:38 [info] GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
16:03:34 [info] Starting app { appVersion: '1.34493.1', ... }
```

101457950 er heksadesimalt `0x060C201E`. Husk dette tallet, det er feilens signatur. Vindusloggen gir også utløseren: Umiddelbart før hvert krasj ber en side i appens innebygde nettleser om en WebGPU-adapter.

```text
16:01:38 [warn] The powerPreference option is currently ignored
                when calling requestAdapter() on Windows.
16:01:38 [warn] A valid external Instance reference no longer exists.
14:59:15 [warn] WebGL: CONTEXT_LOST_WEBGL: loseContext: context lost
```

Altså: `navigator.gpu.requestAdapter()` går i Chromiums GPU-prosess inn i Dawns adapteropplisting, GPU-prosessen krasjer, og i stedet for at appen starter den på nytt, avsluttes hele programmet.

## Mistenkt nr. 1: grafikkdriveren

Maskinen har et Radeon RX 7900 XT med Adrenalin 32.0.31035.1003, og appen pakker Electron 42.9.2 med Chromium 148. Den enkle forklaringen ligger på bordet: Gammel Dawn-kode møter RDNA3-driver, driveren krasjer, saken er oppklart. Praktisk, plausibelt og, som det skal vise seg: feil. Men først ting først, for man kan bare motbevise med eksperimenter.

To forhold falt på forhånd bort som falske spor. Den deaktiverte iGPU-en i Enhetsbehandling (status «Error») er ganske enkelt kode 22, bevisst deaktivert. Og appen hadde maskinvareakselerasjon avslått for lengst (`isHardwareAccelerationDisabled: true` i config.json), uten at det påvirket krasjene. Hvorfor denne innstillingen til og med forverrer problemet, viser seg først helt til slutt.

## Eksperiment 1: kontrolltest i gjeldende Chromium

Samme belastning, samme maskin, gjeldende nettleser: webgpureport.org i Chromium 151 initialiserer WebGPU fullstendig, adapter `amd / rdna-3`, inkludert opprettelse av enhet, uten noen avvik. Den gjeldende driveren med gjeldende Dawn fungerer altså rent.

## Eksperiment 2: standard Electron 42.9.2, maskinvarebane

Hvis Electron 42 ikke fungerer med denne driveren, må det kunne gjenskapes med 20 linjer. Altså: nøyaktig samme Electron-versjon som i appen som et rent testprosjekt, ett vindu, én side, én `requestAdapter()`:

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

Resultat med maskinvareakselerasjon: `adapter ok (amd/rdna-3), device ok`. Ingen krasj. Electron 42s D3D12-bane med denne driveren fungerer feilfritt. Hypotesen «gammel Dawn-kode tåler ikke RDNA3-driveren» er dermed motbevist.

## Eksperiment 3: standard Electron 42.9.2, programvarebane som i appen

Appen kjører imidlertid uten maskinvareakselerasjon. Derfor samme eksperiment med `app.disableHardwareAcceleration()`, i tillegg en aktiv WebGL-kontekst (som i programvaremodus kjører via SwiftShader) og `powerPreference: 'high-performance'` ved adapterforespørselen, for å gjenskape forløpet i apploggene nøyaktig:

```text
[renderer] webgl context: WebKit WebGL
[renderer] The powerPreference option is currently ignored
           when calling requestAdapter() on Windows.
[renderer] No available adapters.
[renderer] RESULT: adapter=null
TIMEOUT: no crash after 25s
```

Samme powerPreference-advarsel som i apploggen, samme kodebane inn i adapteropplistingen, og deretter korrekt svar: ingen adapter tilgjengelig, rent avvist, prosessen lever. Standard Electron 42.9.2 krasjer rett og slett ikke på denne maskinen, uansett hvilken bane.

## Eksperiment 4: annen maskinvare, samme signatur

Før man gjetter videre, er det verdt å se i issue-trackeren, og der blir det tydelig: Det identiske krasjet med identisk avslutningskode 0x060C201E er rapportert flere ganger, blant annet på en NVIDIA RTX 5080 Laptop-GPU. I systemets hendelseslogg: ingen TDR-hendelser, ingen driverresett. Driveren, uansett produsent, er ikke årsaken. Krasjårsaken ligger i selve appens GPU-prosess, eller rettere sagt, som straks skal vise seg, i appens reaksjon på krasjet.

## Eksperiment 5: få tak i minidumpen som appen sletter

Frem til dette manglet det avgjørende beviset: en krasjdump. Appens Crashpad-mappe var tom etter hvert krasj, noe som først så ut som deaktivert krasjrapportering. Prosesslisten sier noe annet: En `crashpad-handler`-prosess kjører, kommandolinjen peker på databasen i Roaming-profilen og på en plassholderopplastings-URL. Dette er det vanlige mønsteret for Sentry-integrasjon i Electron-apper: Crashpad skriver dumpen lokalt, Sentry-biblioteket leser den ved neste appstart, sender den til produsentens telemetri og sletter den lokalt. Dumpene finnes altså, bare ikke for brukeren.

Løsningen er lite spektakulær: en overvåker uavhengig av appens prosesstre (startet via WMI, slik at appkrasjet ikke tar den med seg), som gjennomsøker Crashpad-databasen hvert 200. millisekund etter `*.dmp` og straks kopierer funnene bort. Deretter utløses krasjet målrettet: åpne webgpureport.org i appens innebygde nettleser. Sekunder senere ligger en 35 MB-minidump i sikkerhetskopimappen, som Sentry forsøker å slette forgjeves ved neste appstart.

## Minidumpen: ingen driver i sikte

Analysen med Python-pakken `minidump` gir tre funn som snur hele bildet:

```text
Exception: EXCEPTION_BREAKPOINT (0x80000003)
Adresse:   Claude.exe+0x5e8a6c9
Prozess:   PID 27660
```

For det første: Den dumpede prosessen er ikke GPU-prosessen, men appens **hovedprosess** (PID-en dukker opp i apploggene som `electron_main`). For det andre: Unntaket er et breakpoint, altså en bevisst utført `int3`. Slik avslutter Chromium seg selv når en `CHECK()` eller `LOG(FATAL)` inntreffer; en driverfeil ville sett ut som Access Violation. For det tredje: I prosessens modulliste er ikke én eneste grafikkdriver-DLL lastet.

Og i dumpens minne står den fatale loggmeldingen i klartekst:

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

## Oppklaringen: Chromiums innebygde selvavbrudd

Denne linjen er ikke en feilfunksjon, den er tilsiktet. I Chromium-kildekoden til den nøyaktig inkluderte versjonen (148.0.7778.280) står det i `gpu_data_manager_impl_private.cc`:

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

Den kalles fra `FallBackToNextGpuMode()`: Krasjer GPU-prosessen, går Chromium ett trinn tilbake (maskinvare-GL → programvare-GL → ren display compositor). Er listen over reserve-moduser tom, avslutter Chromium nettleserprosessen med vilje, for uten en fungerende GPU-prosess kan den ikke engang koordinere programvaregjengivelse lenger.

Det forklarer også hvorfor appen rammes langt hardere enn en vanlig nettleser: Den starter med deaktivert maskinvareakselerasjon, altså allerede nederst i reservekjeden. Når en side i den innebygde nettleseren da ber om WebGPU og programvare-GPU-prosessen krasjer, finnes det ikke flere trinn Chromium kan falle tilbake til. Neste stopp er «Goodbye». I en normal Chrome med aktiv maskinvareakselerasjon koster det samme krasjet ett reservetrinn, og nettleseren fortsetter å kjøre.

Særlig uheldig: Appkonfigurasjonen har et felt `isHardwareAccelerationAutoDisabled`, så appen ser ut til selv å slå av maskinvareakselerasjon etter problemer. Ment som et tiltak mot krasj forkorter nettopp dette reservekjeden og gjør det fatale selvavbruddet mer sannsynlig i stedet for mindre. En beskyttelsesmekanisme og en nødutkobling som skjerper hverandre.

## Hva som gjenstår av avslutningskoden

Da gjenstår GPU-barneprosessen selv, som hver gang setter forløpet i gang. Den etterlater ingen egen krasjrapport, selv om Crashpad-handleren demonstrerbart fungerer (den dumpet hovedprosessen sekunder senere). Det tyder på at GPU-prosessen ikke utløser et normalt unntak, men avsluttes hardt, i stil med `TerminateProcess`, og at den udokumenterte avslutningskoden 0x060C201E stammer nettopp derfra. Den siste delen ligger dermed hos Anthropic: Deres Sentry-telemetri mottar dumpene som slettes lokalt, inkludert spørsmålet om krasjrapporteringen i det hele tatt dekker GPU-prosessen.

## Løsning og status for rapportene

Siden utløseren er WebGPU-forespørslene fra sidene i den innebygde nettleseren, hjelper det frem til en rettelse å deaktivere WebGPU med et Chromium-flagg. Ved en Store-installasjon endres installasjonsbanen ved hver oppdatering, derfor løser en liten launcher den på nytt ved hver oppstart:

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

Siden omleggingen: ikke ett eneste krasj til. Den fullstendige analysen er rapportert: laboratorieeksperimentene og duplikatlenkene i første issue, minidump-analysen med årsakskjeden i det andre. De tre fornuftige rettelsene følger direkte av funnet: avklare krasjårsaken i programvare-GPU-prosessen (dumpene for dette ligger i produsenttelemetrien), ved gjentatte GPU-krasj deaktivere WebGPU målrettet i stedet for å la reservekjeden gå tom, og revurdere automatisk deaktivering av maskinvareakselerasjon, fordi den forkorter kjeden.

## Tillegg: løsningen strekker ikke til, den egentlige løsningen ligger dypere

Allerede samme kveld kom neste krasj, med identisk signatur. Grunnen er enkel: Launcheren med `--disable-features=WebGPU` virker bare når appen også startes gjennom den. Ved vanlig oppstart fra Start-menyen kjører appen uten flagget, og med en Store-app finnes det ingen ryddig måte å gi en vanlig oppstart varige kommandolinjeflagg på.

Den varige løsningen står imidlertid allerede i denne artikkelens årsakskjede: Det fatale selvavbruddet forutsetter at reservekjeden er tom, og den er bare tom umiddelbart fordi appen starter med deaktivert maskinvareakselerasjon. Maskinvareakselerasjon må derfor slås på igjen, i appens `config.json`:

```json
"isHardwareAccelerationDisabled": false
```

Dette virker fra neste appstart og løser begge sider av problemet samtidig. For det første kjører `requestAdapter()` da via maskinvarebanen, som på denne maskinen beviselig er stabil (eksperiment 2: adapter og enhet uten feil). For det andre har Chromium igjen reservetrinn tilgjengelig: Skulle GPU-prosessen krasje én gang til, går nettleseren tilbake til programvaregjengivelse og fortsetter å kjøre, i stedet for å avslutte seg. Den opprinnelige deaktiveringen av maskinvareakselerasjon, trolig en gang satt som stabilitetstiltak, var i virkeligheten forutsetningen for krasjet.

Konklusjonen på feilsøkingen: Den mest nærliggende forklaringen («det var driveren») ville ført til en resultatløs driverodysse. To timer i laboratoriet med den faktiske engine-versjonen motbeviste den, og årsaken ble først funnet i minidumpen som appen rutinemessig rydder bort. Når en GPU-prosess krasjer, bør derfor fire kontroller komme først, før man legger skylden på en produsent: kontrolltesten i den gjeldende nettleseren, kontrolltesten i den rene engine-versjonen, kontroll av om annen maskinvare viser samme signatur, og dumpen av prosessen som faktisk beslutter avbruddet.

## Kilder

1.  [Root cause: Chromiums 'GPU process isn't usable. Goodbye.' (GitHub-Issue #89250)](https://github.com/anthropics/claude-code/issues/89250): Minidump-analysen i denne artikkelen som feilrapport, inkludert opptaksmetode og forslag til rettelser.

2.  [Egen første rapport med laboratorieresultater (GitHub-Issue #89226)](https://github.com/anthropics/claude-code/issues/89226): Eksperiment 1 til 3 og korrigeringen av AMD-hypotesen, med henvisninger til duplikatene.

3.  [GPU process crash kills entire app (GitHub-Issue #81698)](https://github.com/anthropics/claude-code/issues/81698): Samme krasj med identisk avslutningskode på NVIDIA RTX 5080, uten TDR-hendelser; frikjenner grafikkdriverne.

4.  [Chromium-kildekode: gpu_data_manager_impl_private.cc (tagg 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): Funksjonen IntentionallyCrashBrowserForUnusableGpuProcess og reservelogikken.

5.  [Electron-dokumentasjon: child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): Hendelsen som en Electron-app kan bruke til å overvåke GPU-prosesskrasj og iverksette egne tiltak.

6.  [Python-pakken minidump](https://pypi.org/project/minidump/): Verktøyet for dumpanalyse (exception-record, modulliste, minnestrenger), helt uten WinDbg.

7.  [webgpureport.org](https://webgpureport.org/): WebGPU-diagnoseside; fungerte som minimal utløser og kontrolltest i gjeldende Chromium.
