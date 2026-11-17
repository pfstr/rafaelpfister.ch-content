---
title: "Paperless-ngx 3.4.1, 3.4.2 und 3.5.0: strengere ALLOWED_HOSTS, korrigierte OIDC-2FA, asynchrone Bulk-Operationen"
navTitle: "Paperless-ngx 3.5.0"
description: "Paperless-ngx 3.5.0 erlaubt bei gesetzter PAPERLESS_URL nur noch deren Hostname und localhost, korrigiert den wirkungslosen AMR-Standardwert aus 3.4.x und führt Bulk-Operationen asynchron aus. Prüfen Sie vor dem Update, über welche Hostnamen Paperless-ngx erreicht wird, und ergänzen Sie diese in PAPERLESS_ALLOWED_HOSTS."
date: "2026-11-17"
kategorie: "Paperless-ngx"
timeToRead: "7 Min. Lesezeit"
themen:
  - "paperless-ngx"
produkte:
  - "paperless-ngx"
protokolle:
  - "releases"
slug: "paperless-ngx-3-5-0"
translationId: article-4c816dd3cd08ff6e
url: https://rafaelpfister.ch/blog/paperless-ngx-3-5-0
---

# Paperless-ngx 3.4.1, 3.4.2 und 3.5.0: strengere ALLOWED_HOSTS, korrigierte OIDC-2FA, asynchrone Bulk-Operationen

Seit 3.4.0 hat das Paperless-ngx-Projekt drei Versionen veröffentlicht: die Bugfix-Releases 3.4.1 am 31. Oktober 2026 und 3.4.2 am 3. November 2026 sowie das Feature-Release 3.5.0 am 17. November 2026. Die beiden Patch-Versionen korrigieren vor allem Darstellungsfehler der überarbeiteten Weboberfläche und aktualisieren Django und weitere Abhängigkeiten. 3.5.0 ändert den Standardwert für die erlaubten Hostnamen, behebt einen Fehler, durch den die in 3.4.0 eingeführte Einstellung `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR` ohne Wirkung blieb, führt alle Bulk-Operationen asynchron aus, räumt abgelaufene Sessions automatisch auf und bringt manuell auslösbare Workflows. Eine Sicherheitsmeldung mit CVE oder GHSA nennt keines der drei Releases. Der vorherige Stand ist im Artikel [Paperless-ngx 3.4.0: OIDC-Login ohne doppelte 2FA, Thumbnails ohne ImageMagick](/blog/paperless-ngx-3-4-0) beschrieben. Die Updates richten sich an alle Betreiber von Paperless-ngx 3.x; besonders betroffen sind Installationen, die über interne Hostnamen angesprochen werden, und Installationen mit OIDC-Anmeldung.

## Die Versionen im Überblick

| Version | Datum | Art | Kernpunkte |
|---|---|---|---|
| 3.4.1 | 31.10.2026 | Bugfix-Release | Korrekturen am API-Schema und an der Weboberfläche, geringerer Speicherbedarf im Leerlauf |
| 3.4.2 | 03.11.2026 | Bugfix-Release | Bulk-Edit-Tasks in Celery-Workern, Tesseract-Sprachinstallation vor Indexaufbau, filelock 4.2.6, Updates von Django und Frontend-Abhängigkeiten |
| 3.5.0 | 17.11.2026 | Feature-Release | ALLOWED_HOSTS ohne Wildcard bei gesetzter PAPERLESS_URL, AMR-Standardwert korrigiert, asynchrone Bulk-Operationen, automatisches Aufräumen von Sessions, manueller Workflow-Trigger, Tag-Hierarchie ohne django-treenode |

## 3.4.1 und 3.4.2: Korrekturen nach dem Feature-Release

3.4.1 korrigiert die Platzierung einer Schema-Korrektur für `id`-Felder (PR #14604), entfernt doppelte Berechtigungsprüfungen beim Öffnen-Button der Dokumentkarten (PR #14601), blendet die neuen PDF-Steuerelemente weitgehend aus (PR #14595), verschiebt den Vorschau-Button in der Dokumentliste (PR #14594) und behebt die Anzeige der Prüfsumme im hellen Modus sowie die Position des Custom-Field-Dropdowns auf Mobilgeräten (PR #14590). Zusätzlich lädt Paperless-ngx mehr Module erst bei Bedarf, was den Speicherbedarf im Leerlauf senkt (PR #14599).

3.4.2 enthält elf Fehlerbehebungen. Für den Betrieb relevant sind:

- Importaufgaben aus `bulk_edit` laufen sicher in den Celery-Workern (PR #14653).
- Der Aufbau des Suchindex und des LLM-Index wartet, bis die Tesseract-Sprachpakete installiert sind (PR #14670).
- filelock steht auf 4.2.6; die Release Notes begründen das Update mit Korrekturen zu ACLs (PR #14656).
- Die Dokumenthistorie zeigt Tag-Namen statt IDs (PR #14625), Zoom-Buttons lösen kein Absenden von Formularen mehr aus (PR #14624), und der Name einer gespeicherten Ansicht bleibt beim Speichern und Schliessen erhalten (PR #14667).

Bei den Abhängigkeiten aktualisiert 3.4.2 Django, django-guardian, django-filter und drf-spectacular-sidecar (PR #14661), turbohtml auf 1.15.3 (PR #14662) und pdfjs-dist auf 6.4.299 (PR #14644). Die nicht mehr gepflegte Bibliothek ngx-bootstrap-icons ersetzt das Projekt durch eine eigene Lösung (PR #14665). In den Beispiel-Compose-Dateien steigen Gotenberg auf 8.38 (PR #14634) und nginx auf 1.31.7 (PR #14633). Als Sicherheitsupdates kennzeichnen die Release Notes von 3.4.1 und 3.4.2 keinen dieser Einträge.

## 3.5.0: ALLOWED_HOSTS ohne Wildcard

Die Release Notes von 3.5.0 beginnen mit dem Hinweis, dass manche Benutzer ihre Einstellung `PAPERLESS_ALLOWED_HOSTS` anpassen müssen (PR #14735). Bis 3.4.2 dokumentierte das Projekt für `PAPERLESS_ALLOWED_HOSTS` den Standardwert `*`, also alle Hosts. Ab 3.5.0 gilt laut Konfigurationsdokumentation:

- Ist weder `PAPERLESS_URL` noch `PAPERLESS_ALLOWED_HOSTS` gesetzt, akzeptiert Paperless-ngx weiterhin jeden Host.
- Ist nur `PAPERLESS_URL` gesetzt, akzeptiert Paperless-ngx nur den Hostnamen aus dieser URL und `localhost`.
- `PAPERLESS_ALLOWED_HOSTS` ergänzt weitere Hosts zu `localhost` und dem Hostnamen aus `PAPERLESS_URL`. `localhost` bleibt immer erlaubt, weil der Docker-Healthcheck ihn braucht.
- `PAPERLESS_ALLOWED_HOSTS=*` erlaubt jeden Host, auch wenn `PAPERLESS_URL` gesetzt ist.

Betroffen sind Installationen mit gesetzter `PAPERLESS_URL`, die zusätzlich über andere Namen erreicht werden, etwa über einen internen Hostnamen, eine IP-Adresse oder den Dienstnamen in einem Container-Netz, über den andere Dienste die API aufrufen. Solche Anfragen lehnt Django nach dem Update ab. Tragen Sie alle zusätzlichen Namen vor dem Update in `PAPERLESS_ALLOWED_HOSTS` ein, zum Beispiel:

```shell
PAPERLESS_URL=https://paperless.example.com
PAPERLESS_ALLOWED_HOSTS=paperless.intern.example.com
```

Den Wildcard-Wert `*` sollten Sie nur setzen, wenn Paperless-ngx ausschliesslich in einem abgeschotteten Netz erreichbar ist; die Dokumentation rät für Installationen im offenen Internet ausdrücklich dazu, die erlaubten Hosts festzulegen.

## 3.5.0: OIDC-Anmeldung und Sessions

Mit 3.4.0 kam die Einstellung `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`, mit der Paperless-ngx die lokale Zwei-Faktor-Abfrage überspringt, wenn der OIDC-Provider im `amr`-Claim einen vertrauenswürdigen zweiten Faktor meldet. Laut Issue #14692 blieb die Liste der vertrauenswürdigen Werte in 3.4.x immer leer: Der Code übergab `mfa` als Trennzeichen statt als Standardwert. Ohne gesetzte Variable und auch mit `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR=mfa` fragte Paperless-ngx deshalb weiterhin nach dem lokalen zweiten Faktor. PR #14693 behebt das in 3.5.0. Der dokumentierte Standardwert ist `mfa`; Logins mit Passwort verlangen weiterhin den Code aus Paperless-ngx.

Damit tritt die im Artikel zu 3.4.0 beschriebene Verhaltensänderung erst mit 3.5.0 tatsächlich ein: Benutzer mit lokal eingerichteter 2FA, deren Identity Provider `mfa` im `amr`-Claim liefert, sehen nach dem Update keine zweite Abfrage mehr. Prüfen Sie vor dem Update, welche AMR-Werte Ihr Provider sendet. Die Dokumentation verlangt, nur Werte einzutragen, die bedeuten, dass beim Login tatsächlich ein zweiter Faktor verwendet wurde. Gleichzeitig steigt django-allauth auf 65.21.1 (PR #14672).

Paperless-ngx hat abgelaufene Sessions bisher nie aus der Datenbank entfernt. Auslöser für die Änderung war Issue #14721: Mit `PAPERLESS_AUTO_LOGIN_USERNAME` entstand alle 30 Sekunden eine neue Session, die nie gelöscht wurde. Ab 3.5.0 entfernt eine geplante Aufgabe abgelaufene Sessions (PR #14722). Den Zeitpunkt steuert `PAPERLESS_SESSION_CLEANUP_TASK_CRON`, Standard ist `0 3 * * *`, also täglich um 03:00 Uhr.

## 3.5.0: Bulk-Operationen, Workflows und Tags

- **Asynchrone Bulk-Operationen:** Alle Bulk-Operationen auf Dokumente und die Bulk-Bearbeitung von Metadaten laufen über die Aufgabenwarteschlange (PR #14773, #14711) und erscheinen in der Aufgabenliste. Laut API-Dokumentation antworten eingereihte Bulk-Operationen mit `{"result":"OK","task_id":"…"}`; den Ausgang fragen Sie über `/api/tasks/?task_id=<uuid>` ab. Ist die Bulk-Aufgabe abgeschlossen, sind Folgeaufgaben wie das Neuverarbeiten von Dateien erst eingeplant, nicht erledigt. Skripte, die direkt nach dem Aufruf mit dem Ergebnis weiterarbeiten, sollten Sie entsprechend anpassen.
- **Manueller Workflow-Trigger:** Workflows lassen sich mit dem neuen Trigger-Typ manuell auf ausgewählte Dokumente anwenden, in der Bulk-Bearbeitung und in den Dokumentdetails (PR #14704).
- **Tag-Hierarchie:** Paperless-ngx speichert die Tag-Hierarchie neu als Adjazenzliste und verzichtet auf django-treenode (PR #14612). Das Update enthält dafür eine Datenbankmigration. Der Autor misst in einem Prototyp mit 6000 Tags auf SQLite deutlich kürzere Zeiten, etwa 1,3 statt 4409 Millisekunden für das Anlegen eines Tags; die Werte sind nicht unabhängig geprüft. Beim Löschen eines Tags prüft Paperless-ngx zusätzlich die Berechtigungen auf untergeordnete Tags (PR #14687).
- **Weitere Neuerungen:** Notizen zu Dokumenten werden als Markdown dargestellt (PR #14626). Ein lokal zwischengespeichertes Hugging-Face-Embedding-Modell lädt ohne Verbindung zum Hub (PR #14728), und der KI-Chat filtert auch über mehr als 32 700 Dokumente (PR #14754).

## 3.5.0: weitere Fehlerbehebungen

- **OCR-Argumente:** Die Argumente, die Paperless-ngx selbst an OCRmyPDF übergibt (`input_file_or_options`, `output_file` und `sidecar`), weist die API zurück; in `PAPERLESS_OCR_USER_ARGS` werden sie ignoriert (PR #14734). Prüfen Sie, ob Ihre Konfiguration diese Argumente enthält.
- **Workflows:** Der Dateinamenfilter geplanter Workflows nutzt einen mit PostgreSQL kompatiblen regulären Ausdruck (PR #14783).
- **Mail:** Gespeicherte E-Mails werden mit CRLF-Zeilenenden geschrieben (PR #14745), und die PDF/A-Konvertierung von Mail-Archiven läuft nur einmal (PR #14698).
- **Vorschaubilder:** Für Bilder ohne Archivversion erzeugt Paperless-ngx wieder Vorschaubilder (PR #14741).
- **Berechtigungen und Konten:** Die Berechtigungen der Konfigurationsansicht sind vereinheitlicht (PR #14709), bei der Registrierung des ersten Benutzers sperrt Paperless-ngx die betroffenen Datenbankzeilen (PR #14688), und Passwörter werden mit einer konstanten Maske angezeigt (PR #14784). Der Autor von PR #14709 stuft die Änderung ausdrücklich nicht als Schwachstelle ein.

## Abhängigkeiten und Sicherheit

Ein Security Advisory des Projekts zu 3.4.1, 3.4.2 oder 3.5.0 gibt es nicht (Stand 17. November 2026). Das jüngste Advisory GHSA-c8f3-pqr2-rr24 vom 16. Oktober 2026 betrifft KI-Klassifizierungs-Workflows auf Dokumenten ohne Eigentümer und ist seit 3.3.2 behoben; Details stehen im Artikel zu 3.4.0. Systeme auf einer älteren 3.x-Version schliessen diese Lücke mit dem Update auf 3.5.0 ebenfalls.

## Bekannte Probleme

- **Lokale 2FA bei OIDC trotz AMR `mfa` (3.4.0 bis 3.4.2):** behoben in 3.5.0 (Issue #14692, PR #14693).
- **Known Issues zu 3.5.0:** Die Release Notes nennen keine (Stand 17. November 2026). Der Hinweis zu `PAPERLESS_ALLOWED_HOSTS` ist eine beabsichtigte Verhaltensänderung.

## Empfohlenes Vorgehen

Zielversion ist 3.5.0. Legen Sie vor dem Update fest, über welche Hostnamen Paperless-ngx erreicht wird, und ergänzen Sie diese in `PAPERLESS_ALLOWED_HOSTS`; prüfen Sie ausserdem die Werte in `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`. Stoppen Sie laufende Verarbeitungen wie den Dokumenteneinzug und erstellen Sie ein Backup, wie es die Administrationsdokumentation verlangt. Bei Docker-Installationen mit Image von Docker Hub stoppen Sie den Stack mit `docker compose down`, laden das neue Image mit `docker compose pull` und starten mit `docker compose up`; dabei laufen die Datenbankmigrationen, darunter die neue Migration der Tag-Hierarchie, automatisch. Bei Bare-Metal-Installationen aktualisieren Sie die Python-Pakete mit `pip install -r requirements.txt` und führen im Verzeichnis `src` die Migrationen mit `python3 manage.py migrate` aus; anschliessend aktualisieren `python3 manage.py document_index reindex --if-needed` und `python3 manage.py document_llmindex migrate` bei Bedarf den Suchindex und den LLM-Index. Bei einem Update direkt von 3.3.x gelten zusätzlich die Hinweise aus dem Artikel zu 3.4.0, insbesondere zu `pdftoppm`. Eine Docker-Installation mit ausgelagertem Dokumentenspeicher beschreibt der Artikel [Paperless-ngx mit wenig Speicherplatz betreiben](/blog/paperless-dokumente-clouddienst-auslagern).

## Quellen

1.  [GitHub: Paperless-ngx Release v3.5.0](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.5.0): Release Notes mit Hinweis zu `PAPERLESS_ALLOWED_HOSTS`, neuen Funktionen und Fehlerbehebungen.

2.  [GitHub: Paperless-ngx Release v3.4.2](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.4.2): Fehlerbehebungen und aktualisierte Abhängigkeiten.

3.  [GitHub: Paperless-ngx Release v3.4.1](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.4.1): Fehlerbehebungen und geringerer Speicherbedarf im Leerlauf.

4.  [GitHub API: Paperless-ngx Releases](https://api.github.com/repos/paperless-ngx/paperless-ngx/releases): Veröffentlichungsdaten von 3.4.1, 3.4.2 und 3.5.0.

5.  [Paperless-ngx: Konfigurationsdokumentation 3.5.0](https://github.com/paperless-ngx/paperless-ngx/blob/v3.5.0/docs/configuration.md): neues Verhalten von `PAPERLESS_ALLOWED_HOSTS`, Standardwert von `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`, `PAPERLESS_SESSION_CLEANUP_TASK_CRON` und ignorierte OCR-Argumente.

6.  [Paperless-ngx: Konfigurationsdokumentation 3.4.2](https://github.com/paperless-ngx/paperless-ngx/blob/v3.4.2/docs/configuration.md): bisheriger Standardwert `*` für `PAPERLESS_ALLOWED_HOSTS`.

7.  [GitHub: Issue #14692, MFA trusted AMR immer leer](https://github.com/paperless-ngx/paperless-ngx/issues/14692): Ursache und Symptome des wirkungslosen AMR-Standardwerts in 3.4.x.

8.  [GitHub: PR #14722, automatisches Aufräumen von Sessions](https://github.com/paperless-ngx/paperless-ngx/pull/14722): Hintergrund zu nie gelöschten Sessions bei Auto-Login.

9.  [GitHub: PR #14612, Tag-Hierarchie als Adjazenzliste](https://github.com/paperless-ngx/paperless-ngx/pull/14612): Ablösung von django-treenode, Migration und Messwerte des Autors.

10.  [Paperless-ngx: API-Dokumentation 3.5.0](https://github.com/paperless-ngx/paperless-ngx/blob/v3.5.0/docs/api.md): asynchrone Bulk-Operationen und Abfrage über `task_id`.

11.  [Paperless-ngx: Administrationsdokumentation 3.5.0](https://github.com/paperless-ngx/paperless-ngx/blob/v3.5.0/docs/administration.md): Update-Vorgehen für Docker und Bare Metal.

12.  [GitHub: Paperless-ngx Security Advisories](https://github.com/paperless-ngx/paperless-ngx/security/advisories): Übersicht der Advisories, keines zu 3.4.1 bis 3.5.0.
