---
title: "Proton Drive unter Linux: Stand der Dinge im Oktober 2026"
navTitle: "Proton Drive & Linux"
description: "Der offizielle Linux-Client ist angekündigt, aber noch nicht verfügbar. Für Skripte und Server gibt es seit Juni 2026 die offizielle Proton Drive CLI; einhängen lässt sich Proton Drive weiterhin nur mit Rclone. Was fehlt, ist ein auf einzelne Ordner oder Aufgaben beschränkter Maschinenzugang."
date: "2026-10-01"
kategorie: "Proton Drive"
timeToRead: "8 Min. Lesezeit"
themen:
  - "proton-drive"
  - "rclone"
produkte:
  - "proton-drive"
  - "rclone"
protokolle:
  - "storage"
  - "releases"
related:
  - "proton-drive-cli"
  - "paperless-dokumente-clouddienst-auslagern"
  - "rclone-mount-in-docker-container"
slug: "proton-drive-linux-status"
translationId: "article-ca282447e0b9acff"
url: "https://rafaelpfister.ch/blog/proton-drive-linux-status"
---

Für Windows und macOS bietet Proton Drive seit 2023 eigene Sync-Clients. Unter Linux gibt es bislang die Weboberfläche, Community-Werkzeuge und seit Juni 2026 eine offizielle Kommandozeilen-Anwendung, aber noch keinen Sync-Client. Auf einem Server ist die Lage nochmals schwieriger, weil dort weder ein Desktop-Sync noch eine interaktive Anmeldung gut passt.

Dieser Überblick beschreibt den Stand am 1. Oktober 2026. Grundlage sind die veröffentlichten Roadmaps, der Quellcode der Proton Drive CLI und ein Praxistest des Rclone-Backends [als Dokumentablage für Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern).

**Aktualisierung vom 1. Oktober 2026:** Die erste Fassung vom 26. Juli beschrieb die Kommandozeilen-Anwendung nur als Werkzeug im SDK-Repository. Proton hatte sie jedoch bereits am 9. Juni 2026 offiziell als **Proton Drive CLI** veröffentlicht, mit fertigen Builds für Windows, macOS und Linux. Der Abschnitt dazu und die Empfehlungstabelle sind entsprechend überarbeitet; die Details stehen im eigenen Artikel zur [Proton Drive CLI](/blog/proton-drive-cli).

## Der Linux-Client ist angekündigt, aber noch ohne Termin

Im Juni 2026 bestätigte Proton erstmals ausdrücklich, dass ein Linux-Client entwickelt wird. Er entsteht auf dem neuen, vereinheitlichten SDK und soll dieselbe technische Basis wie die Anwendungen für Windows und macOS verwenden. Anfang Oktober 2026 gibt es weiterhin weder einen Termin noch eine öffentliche Beta.

Wichtig für die Einordnung: Das wird ein **Desktop-Sync-Client**. Für den Schreibtisch löst er das Problem. Für Server-Anwendungen ist ein Sync-Client hingegen das falsche Werkzeug, denn ein Dienst soll Dateien direkt aus Proton Drive lesen und dorthin schreiben. Ein Sync-Client hält eine lokale Vollkopie, genau das, was man bei knappem Speicher vermeiden will.

## Rclone bleibt für Mounts und Spiegelungen nötig

Unter Linux ist Rclone mit seinem `protondrive`-Backend derzeit das vielseitigste Werkzeug. Es kann Dateien kopieren und synchronisieren und als einzige verfügbare Lösung Proton Drive per **FUSE-Mount** wie ein lokales Verzeichnis bereitstellen. Zwei Einschränkungen sind dabei wichtig:

**Es ist Beta auf einer nachgebauten API.** Proton dokumentiert seine Drive-API nicht öffentlich; das Backend basiert auf Reverse Engineering. Im Test funktionierte es zuverlässig, drosselte aber bei schnellen Aufruffolgen mit inkonsistenten Verzeichnislisten.

**Für den unbeaufsichtigten Betrieb fragt Rclone nach dem TOTP-Schlüssel.** Der Konfigurationsassistent bezeichnet das Feld als `otp_secret_key`. Gemeint ist der dauerhafte Schlüssel aus der 2FA-Einrichtung, nicht der sechsstellige Code, den eine Authenticator-App gerade anzeigt. Rclone speichert diesen Wert verschleiert und erzeugt daraus bei jeder Anmeldung selbst einen gültigen TOTP-Code.

Wer versehentlich einen aktuellen Einmalcode einträgt, kann die erste Anmeldung abschliessen. Die nächste erneute Authentifizierung scheitert jedoch mit Fehler 8002, weil Rclone denselben Code nicht noch einmal verwenden kann.

Damit bleibt das Konto gegen ein isoliert gestohlenes Passwort geschützt. Ein kompromittierter Server gibt jedoch Passwort und TOTP-Schlüssel preis. Für automatisierte Zugriffe empfiehlt sich deshalb ein **dediziertes Proton-Konto**.

Wie sich so ein Mount in Docker-Umgebungen verhält, inklusive zweier undokumentierter Probleme, steht im [eigenen Artikel zu Rclone in Containern](/blog/rclone-mount-in-docker-container).

## Die offizielle CLI deckt Skripte und Sicherungen ab

Am 9. Juni 2026 hat Proton die **Proton Drive CLI** veröffentlicht, eine einzelne ausführbare Datei `proton-drive` für Windows, macOS und Linux. Sie basiert auf demselben SDK wie die offiziellen Apps; der Quellcode liegt im öffentlichen SDK-Repository. Aktuell ist Version 0.8.0 vom 13. August 2026, mit Builds für x86-64 (auch ohne AVX2), ARM64 und musl-Distributionen wie Alpine.

Das Anmeldemodell ist sauberer als das des Rclone-Backends:

- `auth login` gibt eine Anmelde-URL aus, die sich auch **auf einem anderen Gerät** öffnen lässt; die Anmeldung läuft regulär **inklusive Zwei-Faktor-Authentifizierung**, also auch per SSH auf einem Server ohne Desktop
- die Session landet im **Schlüsselspeicher des Betriebssystems** (Keychain, Credential Manager, libsecret) oder, seit Version 0.6.0, im Passwortmanager `pass`, der auf Servern ohne Desktop-Sitzung praktikabler ist
- danach: Dateien hoch- und herunterladen, verschieben, in den Papierkorb legen, Freigaben, Einladungen und öffentliche Links verwalten, Proton Photos bedienen; jeweils mit `--json` für maschinenlesbare Ausgabe

Passwort und TOTP-Schlüssel müssen so nicht auf dem Server liegen. Für Sicherungen und Build-Artefakte ist die CLI deshalb heute die bessere Wahl als Rclone. Zwei Grenzen bleiben: Die CLI kann **kein Dateisystem einhängen** und **keinen Spiegel mit Löschungen** herstellen; sie lädt hoch und herunter, gleicht aber nicht ab. Ein Befehl `takeout` für einen vollständigen lokalen Export ist im Repository bereits enthalten, aber noch nicht veröffentlicht.

Das SDK selbst stuft Proton weiterhin nicht als produktionsreif für Drittanwendungen ein; die Freigabe ist für Ende 2026 bis Anfang 2027 vorgesehen. Die CLI ist davon nicht betroffen, weil Proton sie selbst herausgibt.

## Die eigentliche Lücke: Maschinenzugänge

Der Kern des Problems liegt eine Ebene tiefer als Client oder SDK: **Proton kennt keine Maschinenzugänge.** Kein App-Passwort, kein Service-Account, kein Token mit begrenztem Umfang. Jede Automatisierung, ob Backup-Skript, Server-Mount oder CI-Job, muss mit den vollwertigen Zugangsdaten des Kontos arbeiten.

Zum Vergleich: Bei S3-kompatiblen Speichern sind Zugriffsschlüssel-Paare der Normalfall, widerrufbar und auf Buckets oder Präfixe einschränkbar. Google und Microsoft kennen App-Passwörter und Service-Accounts. Bei Proton gilt hingegen Alles oder nichts: Wer einem Server Zugriff auf einen Ordner geben will, gibt ihm das ganze Konto.

Bei einem Ende-zu-Ende-verschlüsselten Dienst ist das schwieriger als bei S3, weil ein begrenzter Zugang auch begrenztes Schlüsselmaterial bedeuten müsste. Die Sessions der CLI zeigen aber, dass Proton solche Konstrukte beherrscht. Eine Session ist bereits ein abgeleiteter, widerrufbarer Zugang, nur eben mit dem vollen Umfang des Kontos. Ein offizieller „Maschinen-Token für genau diesen Ordner, nur lesend" wäre der grösste einzelne Fortschritt für den Server-Einsatz, weit vor jedem Client.

## Empfehlung nach Anwendungsfall

| Anwendungsfall | Stand Oktober 2026 |
|---|---|
| Desktop-Sync unter Linux | Warten auf den angekündigten Client; bis dahin Rclone-Sync oder Web-Oberfläche |
| Server-Backup (Dateien hochladen) | [Proton Drive CLI](/blog/proton-drive-cli) mit `filesystem upload` und Konfliktstrategie `create-new-revision`; offiziell unterstützt, ohne gespeichertes Passwort |
| Spiegel mit Löschungen | Rclone mit `sync`; Beta-Status einkalkulieren |
| Dateisystem-Mount für Dienste | Rclone mit `mount`, hinterlegtem TOTP-Schlüssel und dediziertem Konto; der einzige [praxiserprobte Weg](/blog/paperless-dokumente-clouddienst-auslagern) |
| Skript-Automatisierung, Freigaben verwalten | Proton Drive CLI mit `--json`; Versionsstand 0.x, Befehle können sich noch ändern |

Auf dem Linux-Desktop kann man auf den angekündigten Client warten oder vorerst Rclone verwenden. Auf Servern übernimmt die offizielle CLI inzwischen Sicherungen und Automatisierung; für einen Mount bleibt Rclone die einzige praktikable Lösung. Aus einem funktionierenden Behelf wird jedoch erst dann eine belastbare Plattform, wenn Proton begrenzte Maschinenzugänge und einen offiziell unterstützten Mount anbietet.

## Quellen

1.  [OMG Ubuntu: Proton Drive client is (finally) coming to Linux](https://www.omgubuntu.co.uk/2026/06/proton-drive-linux-client): die Bestätigung vom Juni 2026, dass der Linux-Client in Entwicklung ist, ohne Termin.

2.  [Proton: Product roadmaps for spring and summer 2026](https://proton.me/blog/2026-spring-summer-roadmaps): die Roadmap mit dem Linux-Client ohne Zeitfenster und dem SDK als Fundament der eigenen Apps.

3.  [ProtonDriveApps/sdk auf GitHub](https://github.com/ProtonDriveApps/sdk): das öffentliche SDK-Repository samt Quellcode und Changelog der CLI.

4.  [Proton: Introducing Proton Drive CLI](https://proton.me/blog/proton-drive-cli): die offizielle Veröffentlichung der CLI am 9. Juni 2026.

5.  [Proton Drive CLI: Downloads](https://proton.me/download/drive/cli/index.html): aktuelle Version 0.8.0 vom 13. August 2026 mit allen Plattform-Builds.

6.  [Proton Drive SDK preview](https://proton.me/blog/proton-drive-sdk-preview): Protons eigene Einordnung: noch nicht produktionsreif für Drittanwendungen.

7.  [Rclone: Proton Drive](https://rclone.org/protondrive/): das Backend samt Beta-Hinweis und der Option `otp_secret_key` für die unbeaufsichtigte Anmeldung.
