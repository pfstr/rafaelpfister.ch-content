---
title: "Claude Desktop stürzt ständig ab: „GPU process gone“ mit Exit-Code 101457950, Ursache und Lösung"
navTitle: "Claude-Desktop-Absturz"
description: "Die Claude-Desktop-App unter Windows beendet sich komplett mit „GPU process gone: exitCode 101457950“ (0x060C201E), oft gefolgt vom Reparatur-Dialog der Store-App. Die vollständige Ursachenkette: Code Integrity blockiert vk_swiftshader.dll, Chromiums Fallback-Kette läuft leer, der eingebaute Selbstabbruch beendet die App. Mit Sofortlösung, Selbstdiagnose per Event-Log und der Analyse bis zum Minidump."
date: "2026-08-25"
kategorie: "Claude"
timeToRead: "11 min to read"
themen:
  - "claude"
slug: "claude-desktop-webgpu-absturz"
url: "https://rafaelpfister.ch/blog/claude-desktop-webgpu-absturz"
translationId: "article-0932cd50b8160b45"
---

Die Claude-Desktop-App unter Windows beendet sich ohne Fehlermeldung, alle laufenden Claude-Code-Sessions sind weg, und manchmal startet die App danach erst nach einem „Reparieren" über die Windows-Einstellungen wieder. Im Log der App steht dann diese Zeile:

```text
GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
```

101457950 ist hexadezimal `0x060C201E`. Wenn Sie diese Signatur in Ihrem Log finden, sind Sie hier richtig: Dieser Artikel dokumentiert die vollständige Ursachenkette dieses Absturzes, die Sofortmassnahmen, mit denen die App wieder stabil läuft, und die Selbstdiagnose, mit der Sie den Befund auf Ihrem eigenen System in zwei Minuten bestätigen. Betroffen sind Installationen aus dem Microsoft Store (MSIX) auf allen GPU-Herstellern, von Intel-iGPUs über NVIDIA bis AMD; die Hardware ist, so viel vorweg, nicht die Ursache.

## Die Lösung in Kürze

Der eigentliche Fehler liegt im Installationspaket der App und kann nur von Anthropic behoben werden (Stand 25.08.2026 offen, Issue [#81341](https://github.com/anthropics/claude-code/issues/81341)). Bis dahin machen drei Massnahmen die App stabil, wirksamste zuerst:

**1. Hardware-Beschleunigung aktivieren.** Prüfen Sie in der Datei `%LOCALAPPDATA%\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude\config.json` diese beiden Werte und setzen Sie sie bei Bedarf auf `false` (App vorher beenden, danach neu starten):

```json
"isHardwareAccelerationDisabled": false,
"isHardwareAccelerationAutoDisabled": false
```

Das klingt paradox, weil deaktivierte Hardware-Beschleunigung üblicherweise die stabilere Wahl ist. Bei diesem Bug ist es umgekehrt, und warum, erklärt die Ursachenkette weiter unten: Die Einstellung entscheidet darüber, ob ein GPU-Prozess-Absturz nur eine Fallback-Stufe kostet oder die ganze App.

**2. Den eingebetteten Browser sparsam einsetzen.** Auslöser des Absturzes sind Seiten im Browser-/Vorschau-Bereich der App. Wer den Bereich nach Gebrauch schliesst, statt Tabs geparkt zu lassen, reduziert die Absturzfrequenz drastisch; dieser Zusammenhang ist im Community-Thread mehrfach mit Zahlen belegt.

**3. Optional: WebGPU abschalten.** Ein Start mit `--disable-features=WebGPU` unterbindet den häufigsten Auslöser vollständig. Bei einer Store-App wechselt der Installationspfad mit jedem Update, deshalb ein Launcher, der ihn bei jedem Start neu auflöst:

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

Der Haken: Das wirkt nur, wenn die App auch über diesen Launcher gestartet wird. Massnahme 1 wirkt bei jedem Start.

Ein „Reparieren" oder Neuinstallieren der App behebt das Problem übrigens nicht, es kuriert nur das Folgesymptom (dazu unten mehr). Auch Grafiktreiber-Updates sind vergebliche Mühe.

## Selbstdiagnose: den Befund auf dem eigenen System bestätigen

Zwei Prüfungen genügen. Erstens die Absturz-Signatur im App-Log:

```powershell
Select-String -Path "$env:LOCALAPPDATA\Claude\Logs\main.log" `
  -Pattern 'GPU process gone'
```

Zweitens, und das ist der eigentliche Beweis, das CodeIntegrity-Log von Windows:

```powershell
Get-WinEvent -FilterHashtable @{
  LogName='Microsoft-Windows-CodeIntegrity/Operational'; Id=3033
} -MaxEvents 30 | Where-Object { $_.Message -match 'claude' } |
  Select-Object TimeCreated, Message
```

Auf betroffenen Systemen finden Sie dort Event-3033-Einträge, deren Zeitstempel sekundengenau mit den Absturzzeiten übereinstimmen, mit dieser Meldung:

```text
Code Integrity determined that a process
(...\WindowsApps\Claude_..._x64__pzs8sxrjxfjjc\app\claude.exe)
attempted to load ...\app\vk_swiftshader.dll that did not meet
the Microsoft signing level requirements.
```

Auf dem hier untersuchten System stimmten sieben von sieben Abstürzen über drei Wochen sekundengenau mit einem solchen Event überein, inklusive eines gezielt ausgelösten Kontroll-Absturzes.

## Die vollständige Ursachenkette

Der Absturz ist das Endglied einer Kette aus vier Gliedern, die zwei Analysen gemeinsam ergeben haben: die Code-Integrity-Spur aus dem Community-Issue [#81698](https://github.com/anthropics/claude-code/issues/81698) und eine eigene Minidump-Analyse ([#89250](https://github.com/anthropics/claude-code/issues/89250)).

**Glied 1: Eine Seite im eingebetteten Browser braucht Software-Rendering.** Typischer Auslöser ist ein WebGPU-Aufruf (`navigator.gpu.requestAdapter()`), erkennbar im Fenster-Log an dieser Warnung unmittelbar vor dem Absturz:

```text
[warn] The powerPreference option is currently ignored
       when calling requestAdapter() on Windows.
```

Läuft die App ohne Hardware-Beschleunigung, führt der Weg zwingend über die Software-Vulkan-Implementierung SwiftShader: Der GPU-Prozess versucht, die mitgelieferte `vk_swiftshader.dll` zu laden.

**Glied 2: Windows Code Integrity blockiert die eigene DLL der App.** Der GPU-Prozess läuft mit der Härtungs-Richtlinie „MicrosoftSignedOnly" (per `Get-ProcessMitigation` überprüfbar). Damit eine Store-App ihre eigenen, herstellersignierten DLLs laden darf, muss das MSIX-Paket einen Signatur-Katalog `AppxMetadata\CodeIntegrity.cat` mitbringen. Genau diese Datei fehlt im ausgelieferten Paket, wie Community-Mitglieder durch Inspektion der MSIX-Datei belegt haben. Die Folge: Die Signaturprüfung schlägt fehl, Windows protokolliert Event 3033 und beendet den GPU-Prozess hart. Der Exit-Code `0x060C201E` ist ein AppX-Integritätsfehler aus dem Windows-Loader, kein Chromium-Code; deshalb findet man ihn in keiner Chromium-Quelle, und deshalb hinterlässt der GPU-Prozess auch keinen Crash-Dump, es gibt keine Exception, die man dumpen könnte.

**Glied 3: Chromiums Fallback-Kette läuft leer.** Stürzt der GPU-Prozess ab, schaltet Chromium eine Rendering-Stufe zurück: Hardware-GL, dann Software-GL, dann reiner Display-Compositor. Erst wenn keine Stufe mehr übrig ist, greift der eingebaute Selbstabbruch. Im Quellcode der gebündelten Version (Chromium 148.0.7778.280 in Electron 42.9.2) steht er wörtlich so:

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

**Glied 4: Der Main-Prozess beendet sich absichtlich.** Dieses `LOG(FATAL)` ist der Moment, in dem „die App abstürzt". Belegt ist er durch einen Minidump des Main-Prozesses: `EXCEPTION_BREAKPOINT` (ein absichtliches `int3`, kein Treiberfehler), keine einzige Grafiktreiber-DLL im Prozess, und im Speicher im Klartext:

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

Dass dieser Dump überhaupt existiert, war der schwierigste Teil der Analyse: Die Sentry-Integration der App konsumiert Crashpad-Dumps beim nächsten App-Start, schickt sie an die Hersteller-Telemetrie und löscht sie lokal. Der Crashpad-Ordner ist für den Nutzer deshalb immer leer. Abhilfe schafft ein vom Prozessbaum der App unabhängiger Beobachter (per WMI gestartet, damit ihn der App-Absturz nicht mit beendet), der die Crashpad-Datenbank alle 200 Millisekunden nach `*.dmp` absucht und Funde sofort wegkopiert, bevor sie gelöscht werden. Die Auswertung übernimmt das Python-Paket `minidump`, ganz ohne WinDbg.

## Warum „Hardware-Beschleunigung deaktivieren" alles verschlimmert

Die Kette erklärt auch den kontraintuitivsten Befund. Deaktivierte Hardware-Beschleunigung hat hier zwei fatale Effekte gleichzeitig. Erstens erzwingt sie den SwiftShader-Pfad, also genau den DLL-Ladeversuch, den Code Integrity blockiert; mit aktiver Hardware-Beschleunigung wird `vk_swiftshader.dll` dagegen kaum je gebraucht. Zweitens startet der GPU-Prozess dann bereits am unteren Ende der Fallback-Kette: Ein einziger Absturz genügt, und Glied 4 schlägt zu. Das erklärt auch die Beobachtung aus dem Community-Thread, dass ein Code-Integrity-Block mal folgenlos bleibt und mal die App beendet: Es hängt davon ab, wie viele Fallback-Stufen der Browser-Prozess noch übrig hat.

Besonders unglücklich: Die App kennt ein automatisches Abschalten der Hardware-Beschleunigung nach Problemen (`isHardwareAccelerationAutoDisabled`). Als Stabilitätsmassnahme gedacht, befördert es betroffene Systeme genau in die Konfiguration, in der der nächste Absturz die ganze App kostet.

## Das Folgesymptom: die Reparatur-Schleife

Der Code-Integrity-Fehlschlag hat eine Nebenwirkung, die viele Betroffene für ein eigenes Problem halten: Windows stuft das App-Paket nach dem Vorfall teils als „Modified, NeedsRemediation" ein. Die App startet dann gar nicht mehr, bis man sie über Einstellungen → Apps → Claude → Erweiterte Optionen → „Reparieren" zurücksetzt. Wer die App also „ständig reparieren muss", sieht dasselbe Grundproblem, nur ein Glied weiter: Die Reparatur behebt den Paketstatus, nicht die Ursache; der nächste Absturz folgt beim nächsten blockierten DLL-Ladeversuch.

## Wie die Fehlersuche verlief, und was sich daraus mitnehmen lässt

Der Weg zur Kette war lehrreicher als die Kette selbst, deshalb in Kürze. Verdächtiger Nr. 1 war der Grafiktreiber (AMD RX 7900 XT, RDNA3), und die These fiel durch vier Gegenproben: Aktuelles Chromium 151 rendert WebGPU auf demselben Treiber fehlerfrei. Stock Electron 42.9.2, dieselbe Version wie in der App, als 20-Zeilen-Testprojekt: Hardware-Pfad fehlerfrei, Software-Pfad lehnt sauber mit „No available adapters" ab, kein Absturz, egal wie exakt man den Ablauf der App nachbaut. Und im Issue-Tracker fand sich dieselbe Absturz-Signatur auf NVIDIA- und Intel-Systemen ohne jedes Treiber-Event im Windows-Log, sogar auf einem Rechner mit frisch getauschter Grafikkarte. Ein Fehler, der in der reinen Engine nicht reproduzierbar ist und jede Hardware gleich trifft, wohnt in der Verpackung, und genau dort wurde er gefunden: Der entscheidende Unterschied zwischen dem Testprojekt und der echten App ist die MSIX-Paketierung mit ihrer Code-Integrity-Härtung.

Daraus ergibt sich eine brauchbare Checkliste für GPU-Prozess-Abstürze in Electron-Apps: die Gegenprobe im aktuellen Browser, die Gegenprobe in der reinen Engine-Version, der Blick, ob andere Hardware dieselbe Signatur zeigt, das CodeIntegrity-Log bei Store-Apps, und der Dump des Prozesses, der den Abbruch tatsächlich beschliesst.

## Stand der Meldungen

Die Paketierungs-Ursache ist als [#81341](https://github.com/anthropics/claude-code/issues/81341) gemeldet, der Sammel-Thread mit den Community-Belegen ist [#81698](https://github.com/anthropics/claude-code/issues/81698), die Minidump-Analyse mit der Fallback-Ketten-Erklärung ist [#89250](https://github.com/anthropics/claude-code/issues/89250). Der eigentliche Fix, ein vollständiger Signatur-Katalog im MSIX-Paket, liegt bei Anthropic. Bis dahin gilt: Hardware-Beschleunigung an, Browser-Bereich diszipliniert schliessen, und bei Bedarf WebGPU per Flag aus. Auf dem hier untersuchten System ist die App seit Umsetzung von Massnahme 1 absturzfrei.

## Quellen

1.  [GitHub-Issue #81698: GPU process crash kills entire app](https://github.com/anthropics/claude-code/issues/81698): Der Sammel-Thread mit den Community-Belegen zur Code-Integrity-Kette, den Hersteller-übergreifenden Datenpunkten und der Browser-Pane-Korrelation.

2.  [GitHub-Issue #81341: CIG + vk_swiftshader.dll kills GPU process](https://github.com/anthropics/claude-code/issues/81341): Die Paketierungs-Ursache; fehlender CodeIntegrity-Katalog im MSIX.

3.  [GitHub-Issue #89250: Minidump-Analyse des App-Abbruchs](https://github.com/anthropics/claude-code/issues/89250): Die hier beschriebene zweite Ketten-Hälfte mit Dump-Capture-Methode und Fix-Vorschlägen.

4.  [Chromium-Quellcode: gpu_data_manager_impl_private.cc (Tag 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): Die Funktion IntentionallyCrashBrowserForUnusableGpuProcess und die Fallback-Logik.

5.  [Electron-Dokumentation: child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): Das Ereignis, mit dem eine Electron-App GPU-Prozess-Abstürze beobachten und eigene Gegenmassnahmen ergreifen kann.

6.  [Python-Paket minidump](https://pypi.org/project/minidump/): Werkzeug der Dump-Analyse (Exception-Record, Modulliste, Speicher-Strings).

7.  [webgpureport.org](https://webgpureport.org/): WebGPU-Diagnoseseite; diente als minimaler Auslöser für den Kontroll-Absturz und als Gegenprobe im aktuellen Chromium.
