---
title: "RustDesk 1.5.3: Verbindung mit Freigabe, geprüfte Relay-Ziele und gehärtete Windows-IPC"
navTitle: "RustDesk 1.5.3"
description: "RustDesk 1.5.3 bringt Verbindungen ohne Passwort mit Freigabe durch die Gegenseite, akzeptiert Relay-Anfragen nur noch für angebotene Adressen und authentisiert die IPC-Pipes unter Windows. Server-Strategien dürfen sensible Einstellungen nicht mehr ändern. Ein Update aller Clients ist empfohlen."
date: "2026-11-11"
kategorie: "Fernwartung und Support"
timeToRead: "5 Min. Lesezeit"
themen:
  - "fernwartung"
produkte:
  - "rustdesk"
protokolle:
  - "releases"
  - "haertung"
slug: "rustdesk-1-5-3"
translationId: article-557eece28ec889f1
url: https://rafaelpfister.ch/blog/rustdesk-1-5-3
---

# RustDesk 1.5.3: Verbindung mit Freigabe, geprüfte Relay-Ziele und gehärtete Windows-IPC

RustDesk hat am 9. November 2026 die Version 1.5.3 des Clients veröffentlicht, acht Tage nach 1.5.1. Eine Version 1.5.2 ist auf der Release-Seite nicht aufgeführt. Das Release bringt eine neue Funktion (Verbindungsaufbau mit Freigabe durch die Gegenseite), eine Änderung an der Relay-Logik und 22 Fehlerbehebungen. Mehrere davon betreffen sicherheitsrelevante Abläufe: Prüfung von Relay-Adressen und Relay-UUIDs, Authentisierung der Named Pipes unter Windows, gesperrte Konfigurationsschlüssel in Server-Strategien und erneut die Installer-Übergabe unter Windows. Eine CVE oder ein Security Advisory nennt der Hersteller dazu nicht (Stand 11. November 2026). Das Release richtet sich an alle, die RustDesk-Clients betreiben, insbesondere unter Windows und mit eigenem Server. Der vorherige Stand mit signierten Updates und Trusted Devices V2 ist im Artikel [RustDesk 1.5.1: signierte Updates, Trusted Devices V2 und Härtung der Dateiübertragung](/blog/rustdesk-1-5-1) beschrieben.

## Die Versionen im Überblick

| Version | Datum | Art | Kernpunkte |
|---|---|---|---|
| 1.5.3 | 09.11.2026 | Client-Release | Request approval, Relay nur zu angebotenen Adressen, IPC-Authentisierung unter Windows, gesperrte Schlüssel in Strategien, Installer-Fixes (F-Secure, Junction), Fixes für macOS, DRM, Wayland, Android und iOS |

Der Open-Source-Server (`rustdesk-server`) hat kein neues Release erhalten, die letzte Version ist weiterhin 1.1.16 vom 20. Juli 2026.

## Neue Funktion: Request approval

Mit PR #16784 lässt sich eine Verbindung ohne RustDesk-Passwort aufbauen. Die Gegenseite muss sie in einem Dialog annehmen. Laut PR sendet der Client in diesem Modus kein Passwort aus irgendeiner Quelle, auch nicht aus dem lokalen Speicher oder dem Adressbuch. Ein ausdrücklich übergebenes Passwort wird ignoriert. Die Auswahl erscheint als Untermenü „Request approval" im ID-Dropdown und in den Menüs der Peer-Karten, jeweils mit den Verbindungsarten, die Plattform und Gegenstelle unterstützen (Verbindung, Dateiübertragung und weitere).

Für Skripte und Verknüpfungen nennt der PR den Kommandozeilenparameter `--request-approval` und den URI-Parameter `request-approval=true`. Der Modus bleibt beim Wiederverbinden, beim Verschieben einer Sitzung in ein anderes Fenster und bei weiteren Verbindungen aus der Remote-Toolbar erhalten. Normale Verbindungen behalten ihr bisheriges Passwortverhalten.

Laut PR gelten folgende Grenzen:

- Die verfügbaren Authentisierungsmethoden werden nicht mit der Gegenstelle ausgehandelt. Ist dort nur Passwort-Anmeldung erlaubt oder gelten Einschränkungen am Anmeldebildschirm, greifen diese wie bisher.
- Besteht für denselben Peer und dieselbe Verbindungsart bereits ein Fenster, aktiviert RustDesk dieses, ohne die Anmeldeart zu ändern. Für eine neue Verbindung mit Freigabe schliessen Sie die bestehende Sitzung zuerst.
- Bei Tunnel-Multiplexing kann ein bereits freigegebener Tunnel ohne erneute Rückfrage wiederverwendet werden.

Angaben zur Kompatibilität mit älteren Versionen der Gegenstelle enthält der PR nicht.

## Relay nur zu angebotenen Adressen

PR #16858 schliesst eine Lücke in der Relay-Aushandlung. Bisher übernahm die kontrollierte Seite die Relay-Adresse und die UUID aus einer weitergeleiteten `RequestRelay`-Nachricht ungeprüft und verband sich mit der angegebenen Adresse. Neu merkt sich die kontrollierte Seite die Relay-Adressen, die ihr angeboten wurden, sowie das Relay, das sie selbst wählen würde (höchstens 32 Einträge, Gültigkeit zehn Minuten ab der letzten Meldung). Anfragen mit einem anderen Relay verwirft sie vor der Namensauflösung. Beide Seiten lehnen zudem UUIDs ab, die leer oder länger als 128 Bytes sind oder andere Zeichen als ASCII-Buchstaben, Ziffern, `-` und `_` enthalten. Laut PR entspricht das der Prüfung, die hbbs bereits vornimmt.

Eine abgelehnte Anfrage erhält keine Antwort, der Relay-Versuch des Controllers läuft deshalb in ein Timeout statt in eine Fehlermeldung. Ein Relay, das Sie aus der Konfiguration entfernen, wird spätestens zehn Minuten nach seiner letzten Meldung nicht mehr akzeptiert.

Für Betreiber eines eigenen Servers ist eine Einschränkung relevant: Läuft hbbs mit der als veraltet markierten Option `--mask`, wird der Rückfall auf ein Relay nach einem gescheiterten WebRTC-Versuch abgelehnt, wenn hbbs die Relay-Adresse für das LAN umgeschrieben hat. Dieser Punkt war beim Merge offen. Prüfen Sie, ob Ihre hbbs-Instanz `--mask` verwendet, und testen Sie Relay-Verbindungen nach dem Update.

## Weitere sicherheitsrelevante Korrekturen

- **Windows-IPC (PR #16755):** Laut PR konnten andere Benutzer bisher weitere Serverinstanzen der Named Pipes für Haupt- und Dienst-IPC anlegen und so Verbindungen an RustDesk abfangen, IPC-Daten mitlesen oder Konfigurationsantworten fälschen. Clients öffnen die Pipes nun ohne dieses Recht und authentisieren den Server vor dem Datenaustausch. Dienst-Pipes müssen SYSTEM gehören. Laut PR bleibt ein Denial of Service möglich, wenn ein Angreifer den Pipe-Namen zuerst belegt, und ältere Versionen mit offeneren Zugriffsregeln können bis zum Update keine Verbindung herstellen.
- **Strategien (PR #16696):** Vom Server per Heartbeat verteilte Strategien dürfen sensible Einstellungen nicht mehr ändern. Gesperrt sind unter anderem `api-server`, `custom-rendezvous-server`, `relay-server`, `key`, `2fa`, `bot`, `enable-trusted-devices`, `trusted-device-expiration-days` und `websocket-port`. Weiterhin setzbar sind etwa `approve-mode`, `verification-method`, `access-mode`, `whitelist`, `id-whitelist` und `allow-auto-update`. Abgelehnte Schlüssel werden ohne ihren Wert protokolliert. Der Autor bezeichnet die Änderung ausdrücklich als begrenzte Härtung: Strategie-Antworten sind weiterhin nicht authentisiert.
- **Windows-Installer (PR #16889, #16908, #16871):** Die mit 1.5.1 eingeführte Übergabe über eine Named Pipe ist wieder entfernt, da sie laut PR auf Systemen mit F-Secure DeepGuard Installation und „Stop service" verhinderte (Fehler `0x800700E8`). Der Schutz der vorbereiteten Dateien beruht nun auf den restriktiven DACLs aus 1.5.1 und auf eigenen Staging-Verzeichnissen, deren Handles während der Übergabe offen bleiben. Ist `C:\Windows\Temp` über eine Junction umgeleitet, verwendet der Installer stattdessen den ProgramData-Ordner des Systems, statt mit einem Fehler abzubrechen.
- **macOS (PR #16729, #16879):** Der Privacy Mode wird am Sitzungsende vor dem Sperren abgeschaltet, und der Dienst-Daemon startet direkt statt über `sh`.
- **IP-Whitelist (PR #16692):** Leerzeichen um Einträge der IP-Whitelist werden ignoriert.

Bricht die Installation unter Windows mit F-Secure ab, testen Sie 1.5.3 vor dem breiten Rollout auf einem betroffenen Gerät. Laut PR #16889 stand die Bestätigung des F-Secure-Fixes durch den Melder beim Merge noch aus. Verteilen Sie Konfigurationswerte wie API-, ID- oder Relay-Server bisher über Strategien, müssen Sie diese künftig auf anderem Weg setzen.

## Fehlerbehebungen nach Plattform

- **Windows:** Deinstallation läuft nach Fehlern beim Konfigurationsabgleich weiter, Dienst-IPC wird erneut versucht, bevor die GUI beendet wird, Clipboard-Owner-Marker wird zuerst geschrieben.
- **Linux:** Login-Bildschirm unter greetd mit Wayland wird über DRM bereitgestellt, unveränderte Bildschirminhalte werden unter DRM nicht erneut kodiert, nicht zugeordnete Wayland-Capture-Fehler erscheinen mit ihrer eigentlichen Meldung statt „x11 expected".
- **Android:** kein Absturz mehr bei Zwischenablage-Inhalten über 1 MB, Frames werden vor der Freigabe des Images kopiert, Hebräisch und Indonesisch werden aus alten Locale-Codes korrekt zugeordnet.
- **iOS:** Startbildschirm folgt dem Dark Mode des Systems.
- **Weitere:** Port-Forwarding hält das KCP-Handle offen und sendet einen Schliessgrund, Seitenleisten des Annahmefensters bleiben unter Windows auf dem aktuellen Monitor, Audio-Eingabegeräte werden nur unter Eingabegeräten gesucht, Mindestgrösse des Sciter-Hauptfensters wird nicht mehr doppelt skaliert.

## Bekannte Probleme

- **Rollback-Schutz bei signierten Updates (aus 1.5.1):** offen. Die Release Notes zu 1.5.3 nennen dazu keine Änderung.
- **Portable Variante unter Windows (aus 1.5.1):** Die Härtung war laut PR #15914 auf einen separaten Branch verschoben. Die Release Notes zu 1.5.3 nennen dazu keine Änderung.
- **Trusted Devices V2 (aus 1.5.1):** Die bekannten Grenzen (asynchroner Widerruf, Ablauf nach Systemuhr des Hosts) sind in den Release Notes zu 1.5.3 nicht behandelt.
- **Relay-Fallback mit `--mask` (neu):** offen laut PR #16858.
- **Pipe-Name vorab belegt (neu):** Denial of Service bleibt laut PR #16755 möglich.
- **Abgebrochene UAC-Abfrage (neu):** Die vorbereiteten Dateien bleiben laut PR #16889 im geschützten Verzeichnis liegen und lassen sich durch einen Administrator entfernen.
- **Strategie-Antworten (neu):** weiterhin nicht authentisiert. Bei Deployments mit HTTP statt HTTPS sind die weiterhin erlaubten Optionen laut PR #16696 manipulierbar.

## Empfohlenes Vorgehen

Zielversion für alle Clients ist 1.5.3. Da ältere Versionen laut PR #16755 bis zum Update keine Verbindung zu den gehärteten IPC-Pipes herstellen können, aktualisieren Sie unter Windows alle RustDesk-Komponenten eines Geräts gemeinsam. Prüfen Sie vor dem Rollout, ob Sie Server-Adressen oder Schlüssel über Strategien verteilen, ob hbbs mit `--mask` läuft und ob auf den Zielsystemen F-Secure im Einsatz ist. Der API-Server sollte per HTTPS erreichbar sein, wie bereits für 1.5.1 empfohlen. Der Open-Source-Server muss für dieses Release nicht aktualisiert werden. Angaben zu Supportfristen enthalten die Release Notes nicht (Stand 11. November 2026).

## Quellen

1.  [GitHub: RustDesk Release 1.5.3](https://github.com/rustdesk/rustdesk/releases/tag/1.5.3): Release Notes mit Datum, neuer Funktion, Änderung und Fehlerbehebungen.

2.  [GitHub: RustDesk Releases](https://github.com/rustdesk/rustdesk/releases): Versionsliste ohne Release 1.5.2.

3.  [GitHub: PR #16784, request approval](https://github.com/rustdesk/rustdesk/pull/16784): Ablauf, Parameter und Grenzen der Verbindung mit Freigabe.

4.  [GitHub: PR #16858, Relay nur zu angebotenen Adressen](https://github.com/rustdesk/rustdesk/pull/16858): Allowlist für Relay-Adressen, UUID-Prüfung und Einschränkung mit `--mask`.

5.  [GitHub: PR #16755, Windows IPC server auth](https://github.com/rustdesk/rustdesk/pull/16755): Absicherung der Named Pipes, Kompatibilität und verbleibendes Risiko.

6.  [GitHub: PR #16696, gesperrte Schlüssel in Strategien](https://github.com/rustdesk/rustdesk/pull/16696): gesperrte und weiterhin erlaubte Optionen, fehlende Authentisierung der Antworten.

7.  [GitHub: PR #16889, Entfernung der Pipe-Übergabe im Installer](https://github.com/rustdesk/rustdesk/pull/16889): Bezug zu F-Secure DeepGuard und Verhalten bei abgebrochener UAC-Abfrage.

8.  [GitHub: PR #16908, Schutz des Installer-Verzeichnisses](https://github.com/rustdesk/rustdesk/pull/16908): eigene Staging-Verzeichnisse mit offenen Handles.

9.  [GitHub: PR #16871, Installer-Staging bei Junction](https://github.com/rustdesk/rustdesk/pull/16871): Ausweichen auf ProgramData bei umgeleitetem Windows Temp.

10.  [GitHub: RustDesk Security Advisories](https://github.com/rustdesk/rustdesk/security/advisories): keine veröffentlichten Advisories (Stand 11. November 2026).

11.  [GitHub: rustdesk-server Releases](https://github.com/rustdesk/rustdesk-server/releases): letzte Server-Version 1.1.16 vom 20. Juli 2026.

12.  [RustDesk 1.5.1: signierte Updates, Trusted Devices V2 und Härtung der Dateiübertragung](/blog/rustdesk-1-5-1): vorheriger Release-Stand.
