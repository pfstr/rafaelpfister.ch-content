---
title: "Proton Drive unter Linux: Was heute auf Desktop und Server funktioniert"
navTitle: "Proton Drive & Linux"
description: "Ein Linux-Client ist angekündigt, aber noch nicht verfügbar. Bis dahin bleiben rclone und das neue SDK. Entscheidend für Server fehlt weiterhin: ein begrenzter Maschinenzugang."
date: "2026-07-26"
kategorie: "Proton Drive"
timeToRead: "8 Min. Lesezeit"
themen:
  - "proton-drive"
  - "rclone"
related:
  - "paperless-dokumente-proton-drive-auslagern"
  - "rclone-mount-in-docker-container"
slug: "proton-drive-linux-status"
url: "https://rafaelpfister.ch/blog/proton-drive-linux-status"
---

Für Windows und macOS bietet Proton Drive seit 2023 eigene Sync-Clients. Unter Linux gibt es bislang nur die Weboberfläche, Community-Werkzeuge und ein offizielles SDK im Vorschaustadium. Auf einem Server ist die Lage nochmals schwieriger, weil dort weder ein Desktop-Sync noch eine interaktive Anmeldung gut passt.

Dieser Überblick beschreibt den Stand im Juli 2026. Grundlage ist neben den veröffentlichten Roadmaps ein Praxistest des rclone-Backends [als Dokumentablage für Paperless-ngx](/blog/paperless-dokumente-proton-drive-auslagern).

## Der Linux-Client ist angekündigt, aber noch ohne Termin

Im Juni 2026 bestätigte Proton erstmals ausdrücklich, dass ein Linux-Client entwickelt wird. Er entsteht auf dem neuen, vereinheitlichten SDK und soll dieselbe technische Basis wie die Anwendungen für Windows und macOS verwenden. Einen Termin oder eine öffentliche Beta gibt es noch nicht.

Wichtig für die Einordnung: Das wird ein **Desktop-Sync-Client**. Für den Schreibtisch löst er das Problem. Für Server-Anwendungen ist ein Sync-Client hingegen das falsche Werkzeug, denn ein Dienst soll Dateien direkt aus Proton Drive lesen und dorthin schreiben. Ein Sync-Client hält eine lokale Vollkopie, genau das, was man bei knappem Speicher vermeiden will.

## Heute übernimmt rclone die praktische Arbeit

Unter Linux ist rclone mit seinem `protondrive`-Backend derzeit das vielseitigste Werkzeug. Es kann Dateien kopieren und synchronisieren und als einzige verfügbare Lösung Proton Drive per **FUSE-Mount** wie ein lokales Verzeichnis bereitstellen. Zwei Einschränkungen sind dabei wichtig:

**Es ist Beta auf einer nachgebauten API.** Proton dokumentiert seine Drive-API nicht öffentlich; das Backend basiert auf Reverse Engineering. Im Test funktionierte es zuverlässig, drosselte aber bei schnellen Aufruffolgen mit inkonsistenten Verzeichnislisten.

**Die unbeaufsichtigte Anmeldung braucht einen Kniff.** Wer die Zwei-Faktor-Authentifizierung bei der Einrichtung mit einem Einmalcode füttert, erlebt nach etwa 35 Minuten das Ende der Session:

```text
422 POST https://drive-api.proton.me/auth/v4/2fa:
Ungültige Anmeldedaten (Code=8002)
```

rclone versucht dann, sich mit dem bereits verbrauchten Einmalcode neu anzumelden. Für den unbeaufsichtigten Betrieb muss der dauerhafte TOTP-Schlüssel aus der 2FA-Einrichtung als `otp_secret_key` in der rclone-Konfiguration hinterlegt werden, verschleiert mit `rclone obscure`. rclone erzeugt daraus bei jeder Anmeldung einen aktuellen Code.

Der zweite Faktor schützt damit weiterhin gegen ein isoliert gestohlenes Passwort. Wird jedoch der Server kompromittiert, liegen dort sowohl Passwort als auch TOTP-Schlüssel. Für jeden Dienst sollte deshalb ein **dediziertes Proton-Konto** verwendet werden.

Wie sich so ein Mount in Docker-Umgebungen verhält, inklusive zweier undokumentierter Fallen, steht im [eigenen Artikel zu rclone in Containern](/blog/rclone-mount-in-docker-container).

## Das offizielle SDK zeigt den künftigen Weg

Parallel stellt Proton seine Anwendungen auf ein **offizielles SDK** für JavaScript und C# mit Bindings für Swift und Kotlin um. Das öffentliche Repository enthält auch ein Kommandozeilenwerkzeug. Dessen Anmeldemodell ist sauberer als das des rclone-Backends:

- `auth login` öffnet den Browser; die Anmeldung läuft regulär **inklusive Zwei-Faktor-Authentifizierung**
- die Session landet im **Schlüsselspeicher des Betriebssystems** (Keychain, Credential Manager, libsecret), das SDK erneuert sie selbst
- danach: Dateien mit maschinenlesbarer JSON-Ausgabe auflisten, hochladen und Freigaben prüfen

Passwort und TOTP-Schlüssel müssen so nicht in einer Konfigurationsdatei stehen. Für den Serverbetrieb bleiben dennoch drei Grenzen: Die CLI kann **kein Dateisystem einhängen**, die Anmeldung öffnet einen Browser, und Proton stuft das SDK noch nicht als produktionsreif für Drittanwendungen ein. Die Freigabe ist für Ende 2026 bis Anfang 2027 vorgesehen.

## Die eigentliche Lücke: Maschinenzugänge

Der Kern des Problems liegt eine Ebene tiefer als Client oder SDK: **Proton kennt keine Maschinenzugänge.** Kein App-Passwort, kein Service-Account, kein Token mit begrenztem Umfang. Jede Automatisierung, ob Backup-Skript, Server-Mount oder CI-Job, muss mit den vollwertigen Zugangsdaten des Kontos arbeiten.

Zum Vergleich: Bei S3-kompatiblen Speichern sind Zugriffsschlüssel-Paare der Normalfall, widerrufbar und auf Buckets oder Präfixe einschränkbar. Google und Microsoft kennen App-Passwörter und Service-Accounts. Bei Proton gilt hingegen Alles oder nichts: Wer einem Server Zugriff auf einen Ordner geben will, gibt ihm das ganze Konto.

Fairerweise ist das bei einem Ende-zu-Ende-verschlüsselten Dienst schwieriger als bei S3, weil ein begrenzter Zugang auch begrenztes Schlüsselmaterial bedeuten müsste. Die SDK-Sessions zeigen aber, dass Proton solche Konstrukte beherrscht. Eine Session ist bereits ein abgeleiteter, widerrufbarer Zugang. Ein offizieller „Maschinen-Token für genau diesen Ordner, nur lesend" wäre der grösste einzelne Fortschritt für den Server-Einsatz, weit vor jedem Client.

## Empfehlung nach Anwendungsfall

| Anwendungsfall | Stand Juli 2026 |
|---|---|
| Desktop-Sync unter Linux | Warten auf den angekündigten Client; bis dahin rclone-Sync oder Web-Oberfläche |
| Server-Backup (Dateien hochladen) | rclone `copy`/`sync`; funktioniert, Beta-Status einkalkulieren |
| Dateisystem-Mount für Dienste | rclone `mount` mit hinterlegtem TOTP-Schlüssel und dediziertem Konto; der einzige [praxiserprobte Weg](/blog/paperless-dokumente-proton-drive-auslagern) |
| Skript-Automatisierung mit sauberer Anmeldung | SDK-CLI im Auge behalten; für Produktion noch zu früh |

Auf dem Linux-Desktop kann man auf den angekündigten Client warten oder vorerst rclone verwenden. Auf Servern bleibt rclone die einzige praktikable Mount-Lösung. Aus einem funktionierenden Behelf wird jedoch erst dann eine belastbare Plattform, wenn Proton begrenzte Maschinenzugänge und einen offiziell unterstützten Mount anbietet.

## Quellen

1.  [OMG Ubuntu: Proton Drive client is (finally) coming to Linux](https://www.omgubuntu.co.uk/2026/06/proton-drive-linux-client): die Bestätigung vom Juni 2026, dass der Linux-Client in Entwicklung ist, ohne Termin.

2.  [Proton: Product roadmaps for spring and summer 2026](https://proton.me/blog/2026-spring-summer-roadmaps): die Roadmap mit dem Linux-Client ohne Zeitfenster und dem SDK als Fundament der eigenen Apps.

3.  [ProtonDriveApps/sdk auf GitHub](https://github.com/ProtonDriveApps/sdk): das öffentliche SDK-Repository samt CLI mit Browser-Login und Session im Schlüsselspeicher.

4.  [Proton Drive SDK preview](https://proton.me/blog/proton-drive-sdk-preview): Protons eigene Einordnung: noch nicht produktionsreif für Drittanwendungen.

5.  [rclone: Proton Drive](https://rclone.org/protondrive/): das Backend samt Beta-Hinweis und der Option `otp_secret_key` für die unbeaufsichtigte Anmeldung.
