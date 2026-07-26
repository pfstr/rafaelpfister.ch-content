---
title: "Proton Drive unter Linux: Status quo von offiziellem Client, Rclone-Backend und SDK"
navTitle: "Proton Drive & Linux"
description: "Windows und macOS haben ihre Sync-Clients, Linux wartet — und auf dem Server sieht es nochmal anders aus. Was heute mit Rclone geht, was das offizielle SDK samt CLI schon kann, warum Maschinenzugänge die eigentliche Lücke sind und was Proton angekündigt hat."
date: "2026-07-25"
kategorie: "Proton Drive"
timeToRead: "8 min to read"
themen:
  - "proton-drive"
  - "rclone"
related:
  - "paperless-dokumente-clouddienst-auslagern"
  - "rclone-mount-in-docker-container"
slug: "proton-drive-linux-status"
url: "https://rafaelpfister.ch/blog/proton-drive-linux-status"
---

Proton Drive hat seit 2023 Sync-Clients für Windows und macOS — Linux ging leer aus. Wer Proton Drive auf einem Linux-Desktop oder gar auf einem Server nutzen will, bewegt sich in einem Flickenteppich aus Community-Werkzeugen, einem offiziellen SDK im Vorschaustadium und Ankündigungen. Dieser Artikel sortiert den Stand per Juli 2026 — aus der Praxis heraus, denn ich habe das Rclone-Backend gerade ausgiebig [als Dokumentenablage für Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern) getestet.

## Der offizielle Client: angekündigt, ohne Termin

Im Juni 2026 hat Proton erstmals bestätigt, dass ein Linux-Client tatsächlich in Entwicklung ist — „from the ground up" auf dem neuen, vereinheitlichten SDK, mit derselben Architektur wie die Windows- und macOS-Anwendungen. Einen Termin oder eine Beta gibt es nicht; die Roadmap nennt den „highly anticipated Linux app" ohne Zeitfenster.

Wichtig für die Einordnung: Das wird ein **Desktop-Sync-Client**. Für den Schreibtisch löst er das Problem, für Server-Anwendungsfälle — ein Dienst soll Dateien direkt aus Proton Drive lesen und dorthin schreiben — ist ein Sync-Client das falsche Werkzeug: Er hält eine lokale Vollkopie, genau das, was man bei knappem Speicher vermeiden will.

## Was heute funktioniert: das Rclone-Backend

Der Arbeitsweg für Linux ist seit Jahren Rclone mit seinem `protondrive`-Backend: Kopieren, Synchronisieren und — als einziges Werkzeug — ein **FUSE-Mount**, mit dem Anwendungen die Cloud wie ein Verzeichnis sehen. Zwei Einschränkungen gehören zur ehrlichen Beschreibung:

**Es ist Beta auf einer nachgebauten API.** Proton dokumentiert seine Drive-API nicht öffentlich; das Backend basiert auf Reverse Engineering. Im Test funktionierte es zuverlässig, drosselte aber bei schnellen Aufruffolgen mit inkonsistenten Verzeichnislisten.

**Die unbeaufsichtigte Anmeldung braucht einen Kniff.** Wer die Zwei-Faktor-Authentifizierung bei der Einrichtung mit einem Einmalcode füttert, erlebt nach etwa 35 Minuten das Ende der Session:

```text
422 POST https://drive-api.proton.me/auth/v4/2fa:
Ungültige Anmeldedaten (Code=8002)
```

Rclone versucht, sich mit dem längst verbrauchten Code neu anzumelden. Die Lösung ist, den dauerhaften TOTP-Schlüssel (den Base32-Wert aus der 2FA-Einrichtung) als `otp_secret_key` in der Rclone-Konfiguration zu hinterlegen — obscured über `rclone obscure`. Damit erzeugt Rclone die Codes selbst und läuft dauerhaft. Das ist weniger heikel, als es klingt: Gegen geleakte Passwörter schützt der zweite Faktor unverändert, nur gegen die Kompromittierung des Servers nicht — und diesen Fall hat er nie verteidigt, denn dort liegt auch das Passwort. Ein **dediziertes Konto** nur für den jeweiligen Dienst bleibt trotzdem Pflicht.

Wie sich so ein Mount in Docker-Umgebungen verhält — inklusive zweier undokumentierter Fallen — steht im [eigenen Artikel zu Rclone in Containern](/blog/rclone-mount-in-docker-container).

## Das SDK und seine CLI: offiziell, aber noch nicht für alle

Parallel baut Proton seine Anwendungen auf ein **offizielles SDK** um (JavaScript und C#, mit Bindings für Swift und Kotlin). Das Repository ist öffentlich, und seit Kurzem liegt dort auch ein **Kommandozeilenwerkzeug** — und dessen Anmeldemodell zeigt, wohin die Reise geht:

- `auth login` öffnet den Browser; die Anmeldung läuft regulär **inklusive Zwei-Faktor-Authentifizierung**
- die Session landet im **Schlüsselspeicher des Betriebssystems** (Keychain, Credential Manager, libsecret), das SDK erneuert sie selbst
- danach: Dateien auflisten, hochladen, Freigaben prüfen — mit maschinenlesbarer JSON-Ausgabe

Kein Passwort auf der Kommandozeile, kein TOTP-Schlüssel in einer Konfigurationsdatei. Genau das Modell, das dem Rclone-Backend fehlt. Aber: Die CLI kann **keine Dateisysteme einhängen**, der Browser-Login passt schlecht zu headless Servern, und Proton erklärt das SDK ausdrücklich für noch nicht produktionsreif für Drittanwendungen — die Freigabe ist für Ende 2026 bis Anfang 2027 angepeilt.

## Die eigentliche Lücke: Maschinenzugänge

Der Kern des Problems liegt eine Ebene tiefer als Client oder SDK: **Proton kennt keine Maschinenzugänge.** Kein App-Passwort, kein Service-Account, kein Token mit begrenztem Umfang. Jede Automatisierung — Backup-Skript, Server-Mount, CI-Job — muss mit den vollwertigen Zugangsdaten des Kontos arbeiten.

Zum Vergleich: Bei S3-kompatiblen Speichern sind Zugriffsschlüssel-Paare der Normalfall, widerrufbar und auf Buckets oder Präfixe einschränkbar. Google und Microsoft kennen App-Passwörter und Service-Accounts. Bei Proton gibt es nur Alles-oder-nichts — wer einem Server Zugriff auf einen Ordner geben will, gibt ihm das ganze Konto.

Fairerweise: Bei einem Ende-zu-Ende-verschlüsselten Dienst ist das schwieriger als bei S3, weil ein begrenzter Zugang auch begrenztes Schlüsselmaterial bedeuten müsste. Die SDK-Sessions zeigen aber, dass Proton solche Konstrukte beherrscht — eine Session ist bereits ein abgeleiteter, widerrufbarer Zugang. Ein offizieller „Maschinen-Token für genau diesen Ordner, nur lesend" wäre der grösste einzelne Fortschritt für den Server-Einsatz, weit vor jedem Client.

## Empfehlung nach Anwendungsfall

| Anwendungsfall | Stand Juli 2026 |
|---|---|
| Desktop-Sync unter Linux | Warten auf den angekündigten Client; bis dahin Rclone-Sync oder Web-Oberfläche |
| Server-Backup (Dateien hochladen) | Rclone `copy`/`sync` — funktioniert, Beta-Status einkalkulieren |
| Dateisystem-Mount für Dienste | Rclone `mount` mit hinterlegtem TOTP-Schlüssel und dediziertem Konto — der einzige Weg, [praxiserprobt](/blog/paperless-dokumente-clouddienst-auslagern) |
| Skript-Automatisierung mit sauberer Anmeldung | SDK-CLI im Auge behalten; für Produktion noch zu früh |

Der Status quo in einem Satz: Linux-Nutzer tragen Proton Drive heute mit Community-Werkzeugen, die erstaunlich weit tragen — aber die Bausteine, die aus „funktioniert" ein „dafür gebaut" machen würden, heissen offizieller Mount-Support und Maschinenzugänge, und beide stehen noch aus.

## Quellen

1.  [OMG Ubuntu: Proton Drive client is (finally) coming to Linux](https://www.omgubuntu.co.uk/2026/06/proton-drive-linux-client) — die Bestätigung vom Juni 2026, dass der Linux-Client in Entwicklung ist, ohne Termin.

2.  [Proton: Product roadmaps for spring and summer 2026](https://proton.me/blog/2026-spring-summer-roadmaps) — die Roadmap mit dem Linux-Client ohne Zeitfenster und dem SDK als Fundament der eigenen Apps.

3.  [ProtonDriveApps/sdk auf GitHub](https://github.com/ProtonDriveApps/sdk) — das öffentliche SDK-Repository samt CLI mit Browser-Login und Session im Schlüsselspeicher.

4.  [Proton Drive SDK preview](https://proton.me/blog/proton-drive-sdk-preview) — Protons eigene Einordnung: noch nicht produktionsreif für Drittanwendungen.

5.  [Rclone: Proton Drive](https://rclone.org/protondrive/) — das Backend samt Beta-Hinweis und der Option `otp_secret_key` für die unbeaufsichtigte Anmeldung.
