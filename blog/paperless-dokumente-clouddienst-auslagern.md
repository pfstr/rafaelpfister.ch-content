---
title: "Paperless-ngx mit wenig Speicherplatz betreiben: Dokumente zu Clouddienst auslagern"
navTitle: "Paperless mit Clouddienst"
description: "Paperless-ngx braucht lokal nur Datenbank, Suchindex und Vorschaubilder — die Dokumente selbst können in einem Clouddienst liegen. Was der Praxistest ergeben hat und wie die Einrichtung mit der fertigen Vorlage in drei Befehlen gelingt."
date: "2026-07-26"
kategorie: "Paperless-ngx"
timeToRead: "6 min to read"
themen:
  - "paperless-ngx"
related:
  - "rclone-mount-in-docker-container"
  - "proton-drive-linux-status"
  - "cloud-mount-testen-dummy-pdfs"
slug: "paperless-dokumente-clouddienst-auslagern"
url: "https://rafaelpfister.ch/blog/paperless-dokumente-clouddienst-auslagern"
---

Paperless-ngx speichert seine Dokumente in einem lokalen Verzeichnis, und dieses Verzeichnis wächst mit jedem Scan. Dabei braucht Paperless die Dateien im Alltag kaum: Die Suche läuft gegen die Datenbank, die Liste zeigt Vorschaubilder, und die eigentliche Datei wird erst beim Öffnen gelesen. Also habe ich getestet, ob sich die Ablage in einen Clouddienst verlagern lässt — mit Rclone, so wie es Plex-Nutzer seit Jahren mit Mediensammlungen machen.

Das Ergebnis: **Es funktioniert in beide Richtungen**, und die Einrichtung ist inzwischen auf drei Befehle geschrumpft. Dieser Artikel fasst zusammen, was der Test ergeben hat und wie Sie das Setup selbst aufsetzen. Die technischen Tiefen — Docker-Mount-Propagation, AppArmor-Fallen, Zwei-Faktor-Authentifizierung und die Messmethodik — stehen in eigenen Artikeln, die am Ende verlinkt sind.

## Das Prinzip: heiss bleibt lokal, kalt geht in die Cloud

| Bestandteil | Ort | Warum |
|---|---|---|
| Datenbank (enthält den OCR-Text) | lokal | braucht echtes Locking |
| Suchindex, Vorschaubilder | lokal | ständige Zugriffe |
| **Dokumentdateien** | **Cloud** | werden selten gelesen |
| Cache (zuletzt geöffnete Dokumente) | lokal, begrenzt | wiederholte Zugriffe bleiben schnell |

Eine Stolperfalle vorweg: In Paperless ist das Verzeichnis `archive/` **nicht** das kalte Archiv, sondern enthält die PDF/A-Version, die bei jeder Ansicht ausgeliefert wird — trotz des Namens die heisseste Datei im System. Kalt ist `originals/`. Wer maximal sparen will, schaltet die Archiv-Kopie mit `PAPERLESS_ARCHIVE_FILE_GENERATION=never` ganz ab; die Volltextsuche bleibt davon unberührt, weil der Text in der Datenbank liegt.

Paperless-ngx bringt übrigens keine eigene Cloud-Anbindung mit — kein S3, kein `django-storages`. Ein Dateisystem-Mount über Rclone ist derzeit der einzige Weg, und der funktioniert mit jedem der über 70 von Rclone unterstützten Dienste. Proton Drive war meine Testwahl wegen der Ende-zu-Ende-Verschlüsselung; ein S3-kompatibler Speicher ist die robustere Alternative.

## Was der Test ergeben hat

Getestet mit einer isolierten Paperless-Instanz, 40 generierten Test-PDFs (13.9 MB) und einem dedizierten Proton-Konto:

| Vorgang | Ergebnis |
|---|---|
| Dokument zum ersten Mal öffnen (aus der Cloud) | ~1.8 s |
| Dasselbe Dokument erneut öffnen (aus dem Cache) | ~20 ms |
| Neues Dokument aufnehmen, bis es in der Cloud liegt | ~20 s |
| Dokumentliste, Volltextsuche | 39 ms / 272 ms — funktioniert auch **ohne** Cloud-Verbindung |
| Integritätsprüfung (Prüfsumme jeder Datei) | bestanden, keine Abweichung |
| Ausfall des Mounts | Selbstheilung ohne Paperless-Neustart, verifiziert |

Der lokale Speicherbedarf ist damit von der Grösse des Archivs entkoppelt: Die Sammlung darf wachsen, die Platte nicht.

## So richten Sie es ein

Die komplette Konfiguration liegt als Vorlage auf GitHub: [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage). Auf dem Server:

```bash
git clone https://github.com/pfstr/paperless-cloud-storage.git
cd paperless-cloud-storage
sudo ./setup.sh      # einmalig, bereitet den Host vor (einziger Root-Schritt)
./wizard.sh          # geführt: Anbieter wählen, Zugangsdaten, Rundlauf-Test
```

Der Wizard fragt den Clouddienst ab (Proton, S3, Backblaze B2, WebDAV, SFTP — oder „Not in the list" für jeden anderen Rclone-Dienst), prüft die Verbindung mit einem echten Hoch- und Runterlade-Test und startet den Storage-Container. Danach:

- **Neuinstallation:** `docker compose -f paperless.yml up -d` — fertig.
- **Bestehende Paperless-Instanz:** Datenbank und Einstellungen bleiben unangetastet; die Anleitung [RETROFIT.md](https://github.com/pfstr/paperless-cloud-storage/blob/main/docs/RETROFIT.md) beschreibt Upload der Bestandsdokumente und die eine nötige Änderung an Ihrer Compose-Datei.

Ein bewusst gesetzter Verzicht: Es gibt **kein Webinterface**. Wir hatten rclones Web-GUI im Einsatz — SSH-Tunnel, CORS und flüchtige Mounts machten sie schlimmer als die Kommandozeile, die sie ersetzen sollte. Drei Fragen im Terminal sind schneller.

## Die vier Regeln für den stabilen Betrieb

Die Vorlage setzt sie alle um; wer selbst baut, sollte sie kennen:

1. **`propagation: rslave`** am Media-Bind-Mount des Paperless-Containers — sonst überlebt der Container keinen Neustart des Mounts. Details und die AppArmor-Falle dahinter: [Rclone-Mount im Docker-Container](/blog/Rclone-mount-in-docker-container).
2. **Paperless anhalten, wenn der Mount fehlt** — sonst schreibt es Dokumente in ein leeres lokales Verzeichnis, die der zurückkehrende Mount unsichtbar überdeckt. Ein Watchdog-Skript liegt in der Vorlage bei.
3. **Ein Konto, das sich unbeaufsichtigt anmelden kann** — bei Proton heisst das: den TOTP-Schlüssel in der Rclone-Konfiguration hinterlegen. Warum das die Zwei-Faktor-Authentifizierung nicht entwertet und wo Proton unter Linux insgesamt steht: [Proton Drive unter Linux](/blog/proton-drive-linux-status).
4. **Geplante Volllese-Aufgaben abschalten** (`PAPERLESS_SANITY_TASK_CRON=disable`) — die Integritätsprüfung liest sonst regelmässig den kompletten Bestand aus der Cloud.

## Grenzen, die bleiben

Ein frisch aufgenommenes Dokument liegt einige Sekunden nur im lokalen Cache, bis der Upload durch ist — stirbt die Maschine genau dann, fehlt die Datei. Das Cache-Limit ist weich und kann bei Zugriffsschüben kurzzeitig deutlich überschritten werden. Und rclones Proton-Backend ist offiziell Beta; unter schnellen API-Aufrufen zeigte es Drosselungssymptome. Langzeitdaten aus dem Dauerbetrieb fehlen noch — die Vorlage ist als experimentell gekennzeichnet.

Wie die Messwerte zustande kamen, welche Ausfälle simuliert wurden und wie sich so ein Aufbau überhaupt seriös testen lässt, steht im Methodik-Artikel: [Cloud-Mounts testen mit generierten PDFs](/blog/cloud-mount-testen-dummy-pdfs).

## Fazit

Paperless-ngx auf einer kleinen Platte mit Cloud-Ablage ist machbar und alltagstauglich: knapp zwei Sekunden beim ersten Öffnen, danach Cache-Geschwindigkeit, Suche und Oberfläche bleiben cloud-unabhängig, und der Aufbau heilt sich nach Ausfällen selbst. Wer dagegen auf einem normal dimensionierten Server nur ein paar Gigabyte sparen will, sollte nachrechnen — in meinem Fall belegte die gesamte Ablage 71 MB, das Betriebssystem mehrere Gigabyte. Der Gewinn liegt nicht im sofort gesparten Platz, sondern darin, dass der Bestand wachsen darf, ohne dass die Platte mitwachsen muss.

## Quellen

1.  [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage) — die Vorlage aus diesem Artikel: setup.sh, wizard.sh, Compose-Dateien, Watchdog und Retrofit-Anleitung.

2.  [Rclone: Overview of cloud storage systems](https://rclone.org/overview/) — die über 70 unterstützten Dienste und ihre Fähigkeiten im Vergleich.

3.  [Paperless-ngx: Configuration](https://docs.paperless-ngx.com/configuration/) — `PAPERLESS_ARCHIVE_FILE_GENERATION`, `PAPERLESS_SANITY_TASK_CRON` und die übrigen genutzten Einstellungen.

4.  [Paperless-ngx: Administration](https://docs.paperless-ngx.com/administration/) — Sanity-Checker, Export und Import sowie die geplanten Hintergrundaufgaben.
