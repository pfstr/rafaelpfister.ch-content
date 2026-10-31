---
title: "Paperless-ngx 3.4.0: OIDC-Login ohne doppelte 2FA, Thumbnails ohne ImageMagick"
navTitle: "Paperless-ngx 3.4.0"
description: "Paperless-ngx 3.4.0 überspringt die lokale 2FA, wenn der OIDC-Provider einen vertrauenswürdigen zweiten Faktor meldet, erzeugt PDF-Vorschaubilder mit Poppler statt ImageMagick und aktualisiert mehrere Abhängigkeiten aus Sicherheitsgründen. Prüfen Sie vor dem Update die neue AMR-Einstellung und auf Bare-Metal-Systemen, ob pdftoppm installiert ist."
date: "2026-10-30"
kategorie: "Paperless-ngx"
timeToRead: "5 Min. Lesezeit"
themen:
  - "paperless-ngx"
produkte:
  - "paperless-ngx"
protokolle:
  - "releases"
slug: "paperless-ngx-3-4-0"
translationId: article-57582656ad9c441b
url: https://rafaelpfister.ch/blog/paperless-ngx-3-4-0
---

# Paperless-ngx 3.4.0: OIDC-Login ohne doppelte 2FA, Thumbnails ohne ImageMagick

Das Paperless-ngx-Projekt hat am 29. Oktober 2026 die Version 3.4.0 veröffentlicht, eine Woche nach 3.3.4. Das Feature-Release bringt eine Einstellung, mit der die lokale Zwei-Faktor-Abfrage bei OIDC-Logins entfällt, wenn der Identity Provider bereits einen zweiten Faktor bestätigt hat. Die Vorschaubilder für PDF-Dokumente erzeugt Paperless-ngx neu mit den Poppler-Tools, ImageMagick ist keine Abhängigkeit mehr. Dazu kommen Erweiterungen bei Workflows und Custom Fields, Änderungen an Upload und Benachrichtigungen in der Weboberfläche, acht Fehlerbehebungen sowie Updates von Frontend- und Python-Abhängigkeiten, die das Projekt als Sicherheitsupdates führt. Eine eigene Sicherheitsmeldung mit CVE oder GHSA nennt das Release nicht. Es richtet sich an alle Betreiber von Paperless-ngx 3.x, besonders an Installationen mit OIDC-Anmeldung und an Bare-Metal-Installationen, deren Systempakete Sie selbst verwalten.

## Die Versionen im Überblick

| Version | Datum | Art | Kernpunkte |
|---|---|---|---|
| 3.4.0 | 29.10.2026 | Feature-Release | lokale 2FA bei vertrauenswürdigem OIDC-AMR überspringen, PDF-Thumbnails über Poppler, ImageMagick entfernt, Custom Fields in Workflow-Titeln, Sicherheitsupdates von Abhängigkeiten |

## OIDC-Anmeldung: lokale 2FA bei vertrauenswürdigem AMR

Neu wertet Paperless-ngx beim Login über einen OIDC-Provider den `amr`-Claim des ID-Tokens aus (PR #14403). Enthält er einen der Werte aus `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`, überspringt Paperless-ngx die eigene Zwei-Faktor-Abfrage für diesen Login. Der Standardwert der Variable ist laut Konfigurationsdokumentation `mfa`. Die Dokumentation nennt als Beispiele Authelia, das `mfa` meldet, und Pocket ID, das bei Passkeys `phr` meldet. Pocket ID sendet für Einmalcodes per E-Mail den Wert `otp`; diesen stuft die Dokumentation als nicht vertrauenswürdig ein.

```shell
PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR=mfa
```

Da der Standardwert bereits gesetzt ist, ändert sich das Verhalten nach dem Update für Benutzer mit lokal eingerichteter 2FA, sobald ihr Identity Provider `mfa` im `amr`-Claim liefert. Prüfen Sie deshalb, welche AMR-Werte Ihr Provider tatsächlich sendet und ob diese Werte nur bei einem echten zweiten Faktor gesetzt werden. Die Dokumentation verlangt ausdrücklich, nur Werte einzutragen, die bedeuten, dass beim Login tatsächlich ein zweiter Faktor verwendet wurde. Parallel aktualisiert 3.4.0 django-allauth auf 65.20 und oauthlib auf 4.0 (PR #14570).

## PDF-Vorschaubilder mit Poppler, ImageMagick entfernt

Bisher rief Paperless-ngx für das Vorschaubild eines PDF ImageMagick (`convert -density 300`) auf, das die erste Seite über Ghostscript mit 300 DPI renderte. Ab 3.4.0 rendert `pdftoppm` aus den Poppler-Tools die erste Seite, Pillow skaliert das Bild und speichert es als WebP (PR #14420). Die Auflösung berechnet Paperless-ngx aus der Seitengrösse, die es über `pdfinfo` ermittelt, mit höchstens 300 DPI. Der Ghostscript-Fallback für Thumbnails entfällt, die Reparatur über qpdf bleibt. Der Autor des PR hat deutlich kürzere Laufzeiten und tieferen Speicherbedarf gemessen, bei einem leeren A0-Dokument etwa 0,21 statt 12,9 Sekunden; die Werte sind nicht unabhängig geprüft.

Mit PR #14440 entfällt ImageMagick vollständig. Die verbleibenden Aufgaben, im Kern das Entfernen eines Alpha-Kanals aus Bildern, übernimmt ein Helfer auf Basis von Pillow; WebP-Dateien speichert er dabei verlustfrei. Die ImageMagick-Richtlinie, die bisher für die PDF-Verarbeitung angepasst werden musste, ist damit nicht mehr nötig. Ghostscript bleibt installiert, weil OCRmyPDF es weiterhin braucht.

Für Admins hat das zwei Folgen:

- **Bare Metal:** `pdftoppm` muss vorhanden sein. Fehlt es, gibt Paperless-ngx beim Start eine Warnung aus. Gleichen Sie die installierten Systempakete mit der aktualisierten Abhängigkeitsliste in der Installationsdokumentation ab.
- **Konfiguration:** `PAPERLESS_CONVERT_MEMORY_LIMIT` und `PAPERLESS_CONVERT_TMPDIR` sind laut PR #14420 veraltet und ohne Wirkung, `PAPERLESS_GS_BINARY` wurde ebenfalls als veraltet markiert. Diese Einträge können Sie aus `paperless.conf` bzw. der `.env`-Datei entfernen.

## Workflows, Custom Fields und Weboberfläche

- **Workflows:** Titelvorlagen in Workflows können Custom Fields verwenden (PR #14564). In Tag-Feldern von Workflows lassen sich über- und untergeordnete Tags unabhängig voneinander auswählen (PR #14400).
- **Custom Fields:** Abfragen auf Datumsfelder sind relativ möglich (PR #14549).
- **Upload und Benachrichtigungen:** Die Navigationsleiste hat einen Upload-Button (PR #14442). Der Button zeigt die Zahl laufender Uploads und verweist auf die Übersicht, die Zahl der Benachrichtigungen sinkt (PR #14501, #14500).
- **Oberfläche:** Die Dokumentdetails wurden überarbeitet (PR #14485), das Aufgaben-Badge lädt schneller (PR #14552), und der KI-Chat stellt Markdown dar (PR #14409, #14415).

## Fehlerbehebungen

- Der `document_exporter` brach beim Export in einen Ordner mit `--delete` ab, wenn eine veraltete `manifest.json.tmp` vorhanden war (PR #14526). Wer den Exporter mit `--delete` für Backups nutzt, sollte nach dem Update prüfen, ob die Exporte wieder vollständig durchlaufen.
- Beim Bearbeiten eines Dokuments wird der Suchindex erst nach den Workflow-Läufen aktualisiert (PR #14558).
- Ergibt ein Barcode keine Dokumenttrennung, werden Tags nicht mehr verzögert zugewiesen (PR #14497).
- Abgebrochene oder abgelaufene Aufgaben werden verständlicher angezeigt (PR #14566).
- Die Schema-Ansicht der API löst keine Berechtigungsabfragen auf Dokumente mehr aus (PR #14503), und der Listentyp der Filter für Dokumentobjekte im API-Schema ist korrigiert (PR #14524).
- Lange Versionsbezeichnungen werden im Dropdown gekürzt, die Grösse der Dropdowns ist vereinheitlicht (PR #14578, #14510).

## Abhängigkeiten und Sicherheit

Die Release Notes führen Updates von Frontend-Abhängigkeiten (PR #14581) sowie von brotli, djangorestframework, filelock, msgpack und setuptools (PR #14575) als Sicherheitsupdates. Welche Schwachstellen in diesen Paketen geschlossen werden, nennen die Release Notes nicht. Die Angular-Komponenten stehen auf 22.2.2. Ein eigenes Security Advisory des Projekts zu 3.4.0 gibt es nicht (Stand 30. Oktober 2026).

Das jüngste Advisory GHSA-c8f3-pqr2-rr24 vom 16. Oktober 2026 betrifft die Versionen 3.0 bis vor 3.3.2 und ist in 3.3.2 behoben. Bei aktivierten KI-Funktionen konnten KI-Klassifizierungs-Workflows für Dokumente ohne Eigentümer Auszüge und Metadaten privater Dokumente anderer Benutzer an den konfigurierten KI-Anbieter übermitteln (CVSS 3.1: 4.3, keine CVE). Systeme, die noch auf einer älteren 3.x-Version laufen, schliessen diese Lücke mit dem Update auf 3.4.0 ebenfalls.

## Empfohlenes Vorgehen

Zielversion ist 3.4.0. Stoppen Sie vor dem Update laufende Verarbeitungen wie den Dokumenteneinzug und erstellen Sie ein Backup, wie es die Administrationsdokumentation verlangt. Bei Docker-Installationen mit Image von Docker Hub stoppen Sie den Stack mit `docker compose down`, laden das neue Image mit `docker compose pull` und starten mit `docker compose up`, wobei Datenbankmigrationen automatisch laufen; läuft alles, starten Sie den Stack mit `-d` im Hintergrund. Bei Bare-Metal-Installationen installieren Sie zuerst die neuen Systemabhängigkeiten (insbesondere Poppler mit `pdftoppm`), aktualisieren die Python-Pakete mit `pip install -r requirements.txt` und führen im Verzeichnis `src` die Migrationen mit `python3 manage.py migrate` aus. Legen Sie vor dem ersten OIDC-Login nach dem Update fest, welche Werte in `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR` stehen sollen. Eine Docker-Installation mit ausgelagertem Dokumentenspeicher beschreibt der Artikel [Paperless-ngx mit wenig Speicherplatz betreiben](/blog/paperless-dokumente-clouddienst-auslagern). Known Issues oder Breaking Changes nennen die Release Notes zu 3.4.0 nicht (Stand 30. Oktober 2026).

## Quellen

1.  [GitHub: Paperless-ngx Release v3.4.0](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.4.0): Release Notes mit Datum, neuen Funktionen, Fehlerbehebungen und Abhängigkeiten.

2.  [Paperless-ngx: Changelog](https://github.com/paperless-ngx/paperless-ngx/blob/main/docs/changelog.md): Changelog in der Projektdokumentation mit den Einträgen zu 3.3.x und 3.4.0.

3.  [GitHub: PR #14403, skip local 2FA for trusted OIDC AMR](https://github.com/paperless-ngx/paperless-ngx/pull/14403): Neue Auswertung des AMR-Claims beim OIDC-Login.

4.  [Paperless-ngx: Konfigurationsdokumentation](https://github.com/paperless-ngx/paperless-ngx/blob/main/docs/configuration.md): Beschreibung und Standardwert von `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`.

5.  [GitHub: PR #14420, PDF-Thumbnails über Poppler](https://github.com/paperless-ngx/paperless-ngx/pull/14420): Umstellung auf `pdftoppm`, veraltete Einstellungen und Messwerte des Autors.

6.  [GitHub: PR #14440, ImageMagick als Abhängigkeit entfernt](https://github.com/paperless-ngx/paperless-ngx/pull/14440): Ersatz der restlichen ImageMagick-Aufrufe durch Pillow.

7.  [GitHub: Security Advisory GHSA-c8f3-pqr2-rr24](https://github.com/paperless-ngx/paperless-ngx/security/advisories/GHSA-c8f3-pqr2-rr24): Informationsoffenlegung in KI-Klassifizierungs-Workflows, betroffen 3.0 bis vor 3.3.2, CVSS 4.3.

8.  [Paperless-ngx: Administrationsdokumentation](https://github.com/paperless-ngx/paperless-ngx/blob/main/docs/administration.md): Update-Vorgehen für Docker und Bare Metal, Exporter-Option `--delete`.

9.  [GitHub API: Paperless-ngx Releases](https://api.github.com/repos/paperless-ngx/paperless-ngx/releases): Veröffentlichungsdaten von 3.3.4 und 3.4.0.
