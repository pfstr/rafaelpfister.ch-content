---
title: "Der GPU-Crash 0x060C201E in der Claude-Desktop-App: eine Fehlersuche bis zum Minidump"
navTitle: "GPU-Crash 0x060C201E"
description: "Die Claude-Desktop-App beendet sich reproduzierbar mit 'GPU process gone'. Erst sieht alles nach einem AMD-Treiberbug aus, dann widerlegen eigene Experimente die These, und am Ende liefert ein abgefangener Minidump die tatsächliche Ursache: Chromiums eingebauter Selbstabbruch 'GPU process isn't usable. Goodbye.'"
date: "2026-08-24"
kategorie: "Claude"
timeToRead: "12 min to read"
themen:
  - "claude"
slug: "claude-desktop-webgpu-absturz"
url: "https://rafaelpfister.ch/blog/claude-desktop-webgpu-absturz"
translationId: "article-0932cd50b8160b45"
---

Seit Ende Juli beendet sich meine Claude-Desktop-App unter Windows mehrmals täglich. Kein Dialog, kein Fehlerfenster, die App ist einfach weg, und alle laufenden Claude-Code-Sessions mit ihr. Über 25 Mal inzwischen. Zeit, nicht mehr neu zu starten, sondern nachzusehen, wo der Fehler tatsächlich auftritt. So viel vorab: Der Hauptverdächtige der ersten Stunde stellt sich als unbeteiligt heraus, und die tatsächliche Ursache steht am Ende schwarz auf weiss in einem Minidump, den die App eigentlich gar nicht herausgeben wollte.

## Die Spur im Log

Die App legt ihre Logs unter `%LOCALAPPDATA%\Claude\Logs` ab, ältere Generationen und die Konfiguration liegen im virtualisierten Store-Pfad `%LOCALAPPDATA%\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude`. In `main.log` steht vor jedem Absturz exakt dasselbe:

```text
16:01:38 [info] GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
16:03:34 [info] Starting app { appVersion: '1.34493.1', ... }
```

101457950 ist hexadezimal `0x060C201E`. Merken Sie sich diese Zahl, sie ist die Signatur des Bugs. Das Fenster-Log liefert den Auslöser dazu: Unmittelbar vor jedem Crash fordert eine Seite im eingebetteten Browser der App einen WebGPU-Adapter an.

```text
16:01:38 [warn] The powerPreference option is currently ignored
                when calling requestAdapter() on Windows.
16:01:38 [warn] A valid external Instance reference no longer exists.
14:59:15 [warn] WebGL: CONTEXT_LOST_WEBGL: loseContext: context lost
```

Also: `navigator.gpu.requestAdapter()` läuft im GPU-Prozess von Chromium in die Adapter-Enumeration von Dawn, der GPU-Prozess stürzt ab, und statt dass die App ihn neu startet, beendet sich die komplette Anwendung.

## Verdächtiger Nr. 1: der Grafiktreiber

Die Maschine hat eine Radeon RX 7900 XT mit Adrenalin 32.0.31035.1003, die App bündelt Electron 42.9.2 mit Chromium 148. Die bequeme Erklärung liegt auf dem Tisch: alter Dawn-Code trifft RDNA3-Treiber, Treiber stürzt ab, Fall geschlossen. Bequem, plausibel, und wie sich zeigen wird: falsch. Aber der Reihe nach, denn widerlegen kann man nur mit Experimenten.

Zwei Dinge fielen vorab als falsche Fährten weg. Die deaktivierte iGPU im Gerätemanager (Status "Error") ist schlicht Code 22, bewusst deaktiviert. Und die App hatte Hardware-Beschleunigung längst abgeschaltet (`isHardwareAccelerationDisabled: true` in der config.json), was die Abstürze nicht beeindruckt hat. Warum diese Einstellung das Problem sogar verschärft, zeigt sich erst ganz am Ende.

## Experiment 1: Gegenprobe im aktuellen Chromium

Dieselbe Last, dieselbe Maschine, aktueller Browser: webgpureport.org in Chromium 151 initialisiert WebGPU vollständig, Adapter `amd / rdna-3`, Device-Erstellung inklusive, ohne jede Auffälligkeit. Der aktuelle Treiber mit aktuellem Dawn ist also sauber.

## Experiment 2: stock Electron 42.9.2, Hardware-Pfad

Wenn Electron 42 mit diesem Treiber nicht zurechtkommt, muss sich das mit 20 Zeilen nachstellen lassen. Also: exakt dieselbe Electron-Version wie in der App als reines Testprojekt, ein Fenster, eine Seite, ein `requestAdapter()`:

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

Ergebnis mit Hardware-Beschleunigung: `adapter ok (amd/rdna-3), device ok`. Kein Absturz. Der D3D12-Pfad von Electron 42 auf diesem Treiber funktioniert einwandfrei. Die These "alter Dawn-Code verträgt den RDNA3-Treiber nicht" ist damit widerlegt.

## Experiment 3: stock Electron 42.9.2, Software-Pfad wie in der App

Die App läuft aber ohne Hardware-Beschleunigung. Also dasselbe Experiment mit `app.disableHardwareAcceleration()`, zusätzlich ein aktiver WebGL-Kontext (der im Software-Modus über SwiftShader läuft) und `powerPreference: 'high-performance'` beim Adapter-Request, um den Ablauf der App-Logs exakt nachzubauen:

```text
[renderer] webgl context: WebKit WebGL
[renderer] The powerPreference option is currently ignored
           when calling requestAdapter() on Windows.
[renderer] No available adapters.
[renderer] RESULT: adapter=null
TIMEOUT: no crash after 25s
```

Dieselbe powerPreference-Warnung wie im App-Log, derselbe Codepfad bis in die Adapter-Enumeration, und dann die korrekte Antwort: kein Adapter verfügbar, sauber abgelehnt, Prozess lebt. Stock Electron 42.9.2 stürzt auf dieser Maschine schlicht nicht ab, egal welcher Pfad.

## Experiment 4: andere Hardware, gleiche Signatur

Bevor man weiterrät, lohnt der Blick in den Issue-Tracker, und dort wird es deutlich: Der identische Absturz mit identischem Exit-Code 0x060C201E ist mehrfach gemeldet, unter anderem auf einer NVIDIA RTX 5080 Laptop-GPU. In deren System-Eventlog: keine TDR-Events, keine Treiber-Resets. Der Treiber, egal von welchem Hersteller, ist nicht die Ursache. Die Absturzursache liegt im GPU-Prozess der App selbst, beziehungsweise, wie sich gleich zeigt, in der Reaktion der App auf dessen Absturz.

## Experiment 5: an den Minidump kommen, den die App löscht

Bis hierhin fehlte das entscheidende Beweisstück: ein Crash-Dump. Der Crashpad-Ordner der App war nach jedem Absturz leer, was zunächst nach abgeschaltetem Crash-Reporting aussah. Die Prozessliste sagt etwas anderes: Ein `crashpad-handler`-Prozess läuft, seine Kommandozeile zeigt auf die Datenbank im Roaming-Profil und auf eine Platzhalter-Upload-URL. Das ist das übliche Muster der Sentry-Integration in Electron-Apps: Crashpad schreibt den Dump lokal, die Sentry-Bibliothek konsumiert ihn beim nächsten App-Start, schickt ihn an die Telemetrie des Herstellers und löscht ihn lokal. Die Dumps existieren also, nur nicht für den Nutzer.

Die Lösung ist unspektakulär: ein vom Prozessbaum der App unabhängiger Beobachter (gestartet über WMI, damit ihn der App-Absturz nicht mitnimmt), der die Crashpad-Datenbank alle 200 Millisekunden nach `*.dmp` absucht und Funde sofort wegkopiert. Dann den Absturz gezielt auslösen: webgpureport.org im eingebetteten Browser der App öffnen. Sekunden später liegt ein 35-MB-Minidump im Sicherungsordner, den Sentry beim nächsten App-Start ins Leere zu löschen versucht.

## Der Minidump: kein Treiber weit und breit

Die Analyse mit dem Python-Paket `minidump` liefert drei Befunde, die das Bild komplett drehen:

```text
Exception: EXCEPTION_BREAKPOINT (0x80000003)
Adresse:   Claude.exe+0x5e8a6c9
Prozess:   PID 27660
```

Erstens: Der gedumpte Prozess ist nicht der GPU-Prozess, sondern der **Main-Prozess** der App (die PID taucht in den App-Logs als `electron_main` auf). Zweitens: Die Exception ist ein Breakpoint, also ein absichtlich ausgeführtes `int3`. So beendet sich Chromium selbst, wenn ein `CHECK()` oder `LOG(FATAL)` zuschlägt; ein Treiberfehler sähe nach Access Violation aus. Drittens: In der Modulliste des Prozesses ist keine einzige Grafiktreiber-DLL geladen.

Und im Speicher des Dumps steht die fatale Log-Meldung im Klartext:

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

## Die Auflösung: Chromiums eingebauter Selbstabbruch

Diese Zeile ist keine Fehlfunktion, sie ist Absicht. Im Chromium-Quellcode der exakt gebündelten Version (148.0.7778.280) steht in `gpu_data_manager_impl_private.cc`:

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

Aufgerufen wird sie von `FallBackToNextGpuMode()`: Stürzt der GPU-Prozess ab, schaltet Chromium eine Stufe zurück (Hardware-GL → Software-GL → reiner Display-Compositor). Ist die Liste der Fallback-Modi leer, beendet Chromium den Browser-Prozess absichtlich, denn ohne funktionierenden GPU-Prozess kann es nicht einmal mehr Software-Rendering koordinieren.

Damit erklärt sich auch, warum die App so viel härter getroffen wird als ein normaler Browser: Sie startet mit deaktivierter Hardware-Beschleunigung, also bereits am unteren Ende der Fallback-Kette. Wenn dann eine Seite im eingebetteten Browser WebGPU anfordert und der Software-GPU-Prozess dabei abstürzt, gibt es keine Stufe mehr, auf die Chromium ausweichen könnte. Der nächste Halt ist "Goodbye". In einem normalen Chrome mit aktiver Hardware-Beschleunigung kostet derselbe Absturz eine Fallback-Stufe, und der Browser läuft weiter.

Besonders unglücklich: Die App-Konfiguration kennt ein Feld `isHardwareAccelerationAutoDisabled`, die App schaltet die Hardware-Beschleunigung nach Problemen also offenbar selbst ab. Als Absturz-Gegenmassnahme gedacht, verkürzt genau das die Fallback-Kette und macht den fatalen Selbstabbruch wahrscheinlicher statt seltener. Ein Schutzmechanismus und ein Notausschalter, die sich gegenseitig scharfstellen.

## Was vom Exit-Code übrig bleibt

Bleibt der GPU-Kindprozess selbst, der den Ablauf jeweils anstösst. Er hinterlässt keinen eigenen Crash-Report, obwohl der Crashpad-Handler nachweislich funktioniert (er hat Sekunden später den Main-Prozess gedumpt). Das spricht dafür, dass der GPU-Prozess keine normale Exception auslöst, sondern hart beendet wird, im Stil von `TerminateProcess`, und der undokumentierte Exit-Code 0x060C201E genau daher stammt. Seine letzte Meile liegt damit bei Anthropic: Deren Sentry-Telemetrie empfängt die Dumps, die lokal gelöscht werden, inklusive der Frage, ob das Crash-Reporting den GPU-Prozess überhaupt abdeckt.

## Workaround und Stand der Meldungen

Da der Auslöser die WebGPU-Anfragen der Seiten im eingebetteten Browser sind, hilft bis zum Fix das Abschalten von WebGPU per Chromium-Flag. Bei einer Store-Installation wechselt der Installationspfad mit jedem Update, deshalb löst ein kleiner Launcher ihn bei jedem Start neu auf:

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

Seit der Umstellung: kein einziger Absturz mehr. Die vollständige Analyse ist gemeldet: die Labor-Experimente und die Duplikat-Verweise im ersten Issue, die Minidump-Auswertung mit der Ursachenkette im zweiten. Die drei sinnvollen Fixes ergeben sich direkt aus dem Befund: die Absturzursache im Software-GPU-Prozess klären (die Dumps dafür liegen in der Hersteller-Telemetrie), bei wiederholten GPU-Abstürzen WebGPU gezielt abschalten statt die Fallback-Kette auslaufen zu lassen, und das automatische Deaktivieren der Hardware-Beschleunigung überdenken, weil es die Kette verkürzt.

## Nachtrag: der Workaround greift zu kurz, die Lösung liegt tiefer

Noch am selben Abend der nächste Absturz, identische Signatur. Der Grund ist einfach: Der Launcher mit `--disable-features=WebGPU` wirkt nur, wenn die App auch über ihn gestartet wird. Beim gewohnten Start über das Startmenü läuft die App ohne den Flag, und bei einer Store-App gibt es keinen sauberen Weg, einem normalen Start dauerhaft Kommandozeilen-Flags mitzugeben.

Die dauerhafte Lösung steht aber längst in der Ursachenkette dieses Artikels: Der fatale Selbstabbruch setzt voraus, dass die Fallback-Kette leer ist, und sie ist nur deshalb sofort leer, weil die App mit deaktivierter Hardware-Beschleunigung startet. Also gehört die Hardware-Beschleunigung wieder eingeschaltet, in der `config.json` der App:

```json
"isHardwareAccelerationDisabled": false
```

Das wirkt ab dem nächsten App-Start und behebt beide Seiten des Problems auf einmal. Erstens läuft `requestAdapter()` dann über den Hardware-Pfad, der auf dieser Maschine nachweislich stabil ist (Experiment 2: Adapter und Device ohne Fehler). Zweitens hat Chromium wieder Fallback-Stufen in Reserve: Sollte der GPU-Prozess doch einmal abstürzen, schaltet der Browser auf Software-Rendering zurück und läuft weiter, statt sich zu beenden. Das ursprüngliche Deaktivieren der Hardware-Beschleunigung, vermutlich irgendwann als Stabilitätsmassnahme gesetzt, war in Wahrheit die Vorbedingung des Absturzes.

Das Fazit der Fehlersuche: Die naheliegendste Erklärung ("der Treiber war es") hätte in eine ergebnislose Treiber-Odyssee geführt. Widerlegt haben sie zwei Stunden Labor mit der echten Engine-Version, und die Ursache fand sich erst im Minidump, den die App routinemässig wegräumt. Wenn ein GPU-Prozess abstürzt, gehören deshalb vier Prüfungen an den Anfang, bevor man einem Hersteller die Schuld gibt: die Gegenprobe im aktuellen Browser, die Gegenprobe in der reinen Engine-Version, der Blick, ob andere Hardware dieselbe Signatur zeigt, und der Dump des Prozesses, der tatsächlich den Abbruch beschliesst.

## Quellen

1.  [Root cause: Chromiums 'GPU process isn't usable. Goodbye.' (GitHub-Issue #89250)](https://github.com/anthropics/claude-code/issues/89250): Die Minidump-Analyse dieses Artikels als Bug-Report, inklusive Capture-Methode und Fix-Vorschlägen.

2.  [Eigener Erst-Report mit Labor-Ergebnissen (GitHub-Issue #89226)](https://github.com/anthropics/claude-code/issues/89226): Die Experimente 1 bis 3 und die Korrektur der AMD-These, mit Verweisen auf die Duplikate.

3.  [GPU process crash kills entire app (GitHub-Issue #81698)](https://github.com/anthropics/claude-code/issues/81698): Derselbe Absturz mit identischem Exit-Code auf NVIDIA RTX 5080, ohne TDR-Events; entlastet die Grafiktreiber.

4.  [Chromium-Quellcode: gpu_data_manager_impl_private.cc (Tag 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): Die Funktion IntentionallyCrashBrowserForUnusableGpuProcess und die Fallback-Logik.

5.  [Electron-Dokumentation: child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): Das Ereignis, mit dem eine Electron-App GPU-Prozess-Abstürze beobachten und eigene Gegenmassnahmen ergreifen kann.

6.  [Python-Paket minidump](https://pypi.org/project/minidump/): Werkzeug der Dump-Analyse (Exception-Record, Modulliste, Speicher-Strings), ganz ohne WinDbg.

7.  [webgpureport.org](https://webgpureport.org/): WebGPU-Diagnoseseite; diente als minimaler Auslöser und als Gegenprobe im aktuellen Chromium.
