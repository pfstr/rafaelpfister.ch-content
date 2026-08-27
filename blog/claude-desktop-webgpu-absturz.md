---
title: "Claude Desktop stürzt ständig ab: „GPU process gone“ mit Exit-Code 101457950, Ursache und Lösung"
navTitle: "Claude-Desktop-Absturz"
description: "Die Claude-Desktop-App unter Windows beendet sich komplett mit „GPU process gone: exitCode 101457950“ (0x060C201E), oft gefolgt vom Reparatur-Dialog der Store-App. Die vollständige Ursachenkette: Code Integrity blockiert vk_swiftshader.dll, Chromiums Fallback-Kette läuft leer, der eingebaute Selbstabbruch beendet die App. Mit dauerhafter Lösung (Wechsel auf die klassische Installation ohne MSIX), Selbstdiagnose per Event-Log und der Analyse bis zum Minidump."
date: "2026-08-25"
kategorie: "Claude"
timeToRead: "10 min to read"
themen:
  - "claude"
produkte:
  - "claude"
protokolle:
  - "troubleshooting"
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

101457950 ist hexadezimal `0x060C201E`. Wenn Sie diese Signatur in Ihrem Log finden, sind Sie hier richtig: Dieser Artikel dokumentiert die vollständige Ursachenkette dieses Absturzes, die Sofortmassnahmen, mit denen die App wieder stabil läuft, und die Selbstdiagnose, mit der Sie den Befund auf Ihrem eigenen System in zwei Minuten bestätigen. Betroffen sind MSIX-Installationen (aus dem Microsoft Store oder per MSIX-Setup) auf allen GPU-Herstellern, von Intel-iGPUs über NVIDIA bis AMD; die Hardware ist, so viel vorweg, nicht die Ursache. Die klassische Installation ohne MSIX ist nicht betroffen, und genau das ist die Lösung.

## Die Lösung in Kürze: Wechsel auf die klassische Installation

Der eigentliche Fehler liegt im MSIX-Installationspaket und kann nur von Anthropic behoben werden (Stand 27.08.2026 offen, Issue [#81341](https://github.com/anthropics/claude-code/issues/81341); betroffen ist auch die aktuelle Version 1.37937.3). Dieselbe App gibt es aber zusätzlich als klassische Installation ohne MSIX, und die unterliegt der AppX-Signaturprüfung nicht, die den GPU-Prozess beendet. Der Wechsel ist damit die einzige Massnahme, die den Absturz vollständig beseitigt; er ist sowohl im Issue [#81341](https://github.com/anthropics/claude-code/issues/81341) als auch auf dem hier untersuchten System bestätigt. Die Feature-Ausstattung ist identisch, der Update-Feed liefert für beide Varianten dieselben Versionen.

**Schritt 1: Klassischen Installer laden und ausführen.** Der Download auf [claude.com/download](https://claude.com/download) liefert einen Squirrel-Installer, der die App nach `%LOCALAPPDATA%\AnthropicClaude` installiert (kein Administratorrecht nötig). Per Kommandozeile:

```powershell
curl.exe -L -o "$env:USERPROFILE\Downloads\Claude-Setup-x64.exe" `
  "https://storage.googleapis.com/osprey-downloads-c02f6a0d-347c-492b-a752-3e0651722e97/nest-win-x64/Claude-Setup-x64.exe"
```

| Option | Wirkung |
|---|---|
| `-L` | folgt HTTP-Weiterleitungen bis zur eigentlichen Datei |
| `-o <pfad>` | Zieldatei; hier der Downloads-Ordner |
| `<url>` | offizielle Installer-Quelle; identisch mit dem Ziel des Download-Redirects von claude.ai |

Prüfen Sie nach dem Download die Signatur (`Get-AuthenticodeSignature`, erwartet: `Valid`, Aussteller „Anthropic, PBC") und starten Sie die Datei. Der Installer legt zunächst eine ältere Basisversion ab; auf den aktuellen Stand bringt sie der Update-Mechanismus, entweder automatisch beim ersten Start oder sofort per:

```powershell
& "$env:LOCALAPPDATA\AnthropicClaude\Update.exe" `
  --update https://downloads.claude.ai/releases/win32/x64
```

| Option | Wirkung |
|---|---|
| `--update <url>` | lädt die neueste Version aus dem angegebenen Release-Feed und installiert sie als neues `app-<version>`-Verzeichnis |

**Schritt 2: Konfiguration übernehmen.** Die MSIX-Version hält Login, MCP-Server-Konfiguration und Einstellungen in ihrem virtualisierten Container; die klassische App liest `%APPDATA%\Claude`. Einmalig kopieren (die MSIX-App vorher beenden, beide Varianten laufen wegen eines gemeinsamen Single-Instance-Locks ohnehin nicht gleichzeitig):

```powershell
robocopy "$env:LOCALAPPDATA\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude" `
  "$env:APPDATA\Claude" /E /XD Cache "Code Cache" GPUCache claude-code Crashpad logs sentry
```

| Option | Wirkung |
|---|---|
| `<quelle>` | Konfigurationsordner im virtualisierten AppData des MSIX-Pakets |
| `<ziel>` | Konfigurationsordner der klassischen Installation |
| `/E` | kopiert alle Unterverzeichnisse, auch leere |
| `/XD <namen>` | überspringt die genannten Verzeichnisse; hier Caches und Laufzeitdaten, welche die neue App selbst neu anlegt |

Chat-Verläufe gehen dabei nicht verloren: Sie liegen im claude.ai-Konto beziehungsweise (für Claude-Code-Sessions) unter `%USERPROFILE%\.claude` und hängen nicht an der App-Installation.

**Schritt 3: MSIX-Paket entfernen.** Sonst startet über alte Verknüpfungen weiterhin die abstürzende Variante:

```powershell
Get-AppxPackage Claude | Remove-AppxPackage
```

Der Startmenü-Eintrag „Anthropic → Claude" gehört danach zur klassischen Installation; ein allfälliger Taskleisten-Pin muss neu gesetzt werden.

## Falls Sie beim MSIX-Paket bleiben müssen

Ohne Wechsel bleiben nur Massnahmen, die die Absturzfrequenz senken, ohne die Ursache zu beseitigen:

**Den eingebetteten Browser sparsam einsetzen.** Auslöser des Absturzes sind Seiten im Browser-/Vorschau-Bereich der App. Wer den Bereich nach Gebrauch schliesst, statt Tabs geparkt zu lassen, reduziert die Absturzfrequenz deutlich; dieser Zusammenhang ist im Community-Thread mehrfach mit Zahlen belegt.

**WebGPU abschalten.** Ein Start mit `--disable-features=WebGPU` unterbindet den häufigsten Auslöser. Bei einem MSIX-Paket wechselt der Installationspfad mit jedem Update, deshalb ein Launcher, der ihn bei jedem Start neu auflöst:

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

Das wirkt nur, wenn die App auch über diesen Launcher gestartet wird.

In der Erstfassung dieses Artikels stand an erster Stelle die Empfehlung, die Hardware-Beschleunigung über `isHardwareAccelerationDisabled: false` in der `config.json` zu aktivieren. Diese Empfehlung ist überholt: In aktuellen Versionen (1.37937.x) existiert das Flag nicht mehr, die App startet standardmässig mit aktiver Hardware-Beschleunigung, und sie stürzt trotzdem ab (Details im Nachtrag unten).

Ein „Reparieren" oder Neuinstallieren des MSIX-Pakets behebt das Problem übrigens nicht, es kuriert nur das Folgesymptom (dazu unten mehr). Auch Grafiktreiber-Updates sind vergebliche Mühe.

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

Besonders unglücklich: Die App kannte ein automatisches Abschalten der Hardware-Beschleunigung nach Problemen (`isHardwareAccelerationAutoDisabled`). Als Stabilitätsmassnahme gedacht, beförderte es betroffene Systeme genau in die Konfiguration, in der der nächste Absturz die ganze App kostet.

## Nachtrag 27.08.2026: Hardware-Beschleunigung allein reicht nicht

Die Erstfassung dieses Artikels empfahl aktive Hardware-Beschleunigung als wirksamste Sofortmassnahme, und zwei Tage lang blieb die App damit tatsächlich absturzfrei. Dann folgte das Auto-Update auf 1.37937.3, und mit ihm drei Abstürze an einem Nachmittag, jeder mit dem bekannten Event 3033 zu `vk_swiftshader.dll`. Zwei Befunde daraus:

Erstens fehlt der fehlende Signatur-Katalog auch im aktuellen MSIX-Paket; das Grundproblem besteht in 1.37937.3 unverändert.

Zweitens schützt aktive Hardware-Beschleunigung nur statistisch: Sie verlängert die Fallback-Kette, verhindert aber nicht, dass Chromium sie unter Last oder nach einem Hardware-GPU-Prozessfehler doch bis zur SwiftShader-Stufe durchläuft. Sobald das passiert, blockiert Code Integrity die DLL, und die Kette kann trotzdem leerlaufen. Dazu kommt, dass die Konfigurationsflags `isHardwareAccelerationDisabled`/`isHardwareAccelerationAutoDisabled` in 1.37937.x aus der `config.json` verschwunden sind; die Einstellung lässt sich dort gar nicht mehr festschreiben.

Damit blieb als zuverlässige Lösung nur der oben beschriebene Wechsel auf die klassische Installation. Seit dem Wechsel auf dem hier untersuchten System: dieselbe App-Version, identische Nutzung inklusive Browser-Bereich, kein einziges Event 3033 und kein Absturz.

## Das Folgesymptom: die Reparatur-Schleife

Der Code-Integrity-Fehlschlag hat eine Nebenwirkung, die viele Betroffene für ein eigenes Problem halten: Windows stuft das App-Paket nach dem Vorfall teils als „Modified, NeedsRemediation" ein. Die App startet dann gar nicht mehr, bis man sie über Einstellungen → Apps → Claude → Erweiterte Optionen → „Reparieren" zurücksetzt. Wer die App also „ständig reparieren muss", sieht dasselbe Grundproblem, nur ein Glied weiter: Die Reparatur behebt den Paketstatus, nicht die Ursache; der nächste Absturz folgt beim nächsten blockierten DLL-Ladeversuch.

## Stand der Meldungen

Die Paketierungs-Ursache ist als [#81341](https://github.com/anthropics/claude-code/issues/81341) gemeldet, der Sammel-Thread mit den Community-Belegen ist [#81698](https://github.com/anthropics/claude-code/issues/81698), die Minidump-Analyse mit der Fallback-Ketten-Erklärung ist [#89250](https://github.com/anthropics/claude-code/issues/89250), ein weiterer ausführlicher Report inklusive der Reparatur-Schleife ist [#80444](https://github.com/anthropics/claude-code/issues/80444). Der eigentliche Fix, ein vollständiger Signatur-Katalog im MSIX-Paket, liegt bei Anthropic und steht auch in 1.37937.3 noch aus. Bis dahin gilt: auf die klassische Installation wechseln; wer beim MSIX-Paket bleiben muss, schliesst den Browser-Bereich diszipliniert und schaltet bei Bedarf WebGPU per Flag ab. Auf dem hier untersuchten System ist die App seit dem Wechsel auf die klassische Installation absturzfrei, ohne ein einziges weiteres Event 3033.

## Quellen

1.  [GitHub-Issue #81698: GPU process crash kills entire app](https://github.com/anthropics/claude-code/issues/81698): Der Sammel-Thread mit den Community-Belegen zur Code-Integrity-Kette, den Hersteller-übergreifenden Datenpunkten und der Browser-Pane-Korrelation.

2.  [GitHub-Issue #81341: CIG + vk_swiftshader.dll kills GPU process](https://github.com/anthropics/claude-code/issues/81341): Die Paketierungs-Ursache; fehlender CodeIntegrity-Katalog im MSIX.

3.  [GitHub-Issue #89250: Minidump-Analyse des App-Abbruchs](https://github.com/anthropics/claude-code/issues/89250): Die hier beschriebene zweite Ketten-Hälfte mit Dump-Capture-Methode und Fix-Vorschlägen.

4.  [GitHub-Issue #80444: GPU-Crash mit Forensik und Reparatur-Schleife](https://github.com/anthropics/claude-code/issues/80444): Ausführlicher Einzelreport mit Zeitachsen, Event-Log-Auswertung und dem Befund, dass jeder Absturz das Paket in den Zustand „Modified" versetzt.

5.  [Claude Desktop: offizielle Download-Seite](https://claude.com/download): Bezugsquelle des klassischen Windows-Installers (x64 und ARM64).

6.  [Chromium-Quellcode: gpu_data_manager_impl_private.cc (Tag 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): Die Funktion IntentionallyCrashBrowserForUnusableGpuProcess und die Fallback-Logik.

7.  [Electron-Dokumentation: child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): Das Ereignis, mit dem eine Electron-App GPU-Prozess-Abstürze beobachten und eigene Gegenmassnahmen ergreifen kann.

8.  [Python-Paket minidump](https://pypi.org/project/minidump/): Werkzeug der Dump-Analyse (Exception-Record, Modulliste, Speicher-Strings).

9.  [webgpureport.org](https://webgpureport.org/): WebGPU-Diagnoseseite; diente als minimaler Auslöser für den Kontroll-Absturz und für den Vergleichstest im aktuellen Chromium.
