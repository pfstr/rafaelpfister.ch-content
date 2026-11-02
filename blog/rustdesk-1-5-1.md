---
title: "RustDesk 1.5.1: signierte Updates, Trusted Devices V2 und Härtung der Dateiübertragung"
navTitle: "RustDesk 1.5.1"
description: "RustDesk 1.5.1 prüft Desktop-Updates anhand signierter Metadaten, führt Trusted Devices V2 ein und härtet Dateiübertragung, Windows-Installer und Auto-Login. Dazu kommen rund 35 Fehlerbehebungen, vor allem für DRM- und Wayland-Capture. Ein Update der Clients ist empfohlen, der Server bleibt unverändert."
date: "2026-11-02"
kategorie: "Fernwartung und Support"
timeToRead: "5 Min. Lesezeit"
themen:
  - "fernwartung"
produkte:
  - "rustdesk"
protokolle:
  - "releases"
slug: "rustdesk-1-5-1"
translationId: article-c95ef39dfcc09875
url: https://rafaelpfister.ch/blog/rustdesk-1-5-1
---

# RustDesk 1.5.1: signierte Updates, Trusted Devices V2 und Härtung der Dateiübertragung

RustDesk hat am 1. November 2026 die Version 1.5.1 des Clients veröffentlicht, gut einen Monat nach 1.5.0 (30. September 2026). Das Release bringt fünf neue Funktionen, darunter die Prüfung signierter Update-Metadaten und eine neue Version der Trusted Devices für die Zwei-Faktor-Anmeldung, drei Änderungen und rund 35 Fehlerbehebungen. Mehrere davon härten sicherheitsrelevante Abläufe: Validierung von Download-Pfaden, geschützte Installer-Übergabe unter Windows, Warnung bei unverschlüsselten API-Servern und ein enger gefasstes Auto-Login. Eine CVE oder ein Security Advisory nennt der Hersteller dazu nicht (Stand 2. November 2026). Das Release richtet sich an alle, die RustDesk-Clients auf Windows, macOS, Linux, Android oder iOS betreiben, insbesondere mit eigenem Server. Einrichtung und Betrieb eines eigenen RustDesk-Servers sind im Artikel [RustDesk: die quelloffene TeamViewer-Alternative einrichten](/blog/rustdesk-teamviewer-alternative) beschrieben.

## Die Versionen im Überblick

| Version | Datum | Art | Kernpunkte |
|---|---|---|---|
| 1.5.1 | 01.11.2026 | Client-Release | signierte Update-Metadaten, Trusted Devices V2, Tastenkürzel, Download-Pfad-Validierung, Installer-Härtung unter Windows, Warnung bei HTTP-API-Server, Fixes für DRM, Wayland, X11 und QSV |

Der Open-Source-Server (`rustdesk-server`) ist von diesem Release nicht betroffen. Dessen letzte Version ist 1.1.16 vom 20. Juli 2026.

## Signierte Update-Metadaten

Mit PR #15914 bezieht der Desktop-Updater unter Windows und macOS seine Informationen aus signierten Release-Metadaten. Jedes Release enthält dazu die Assets `rustdesk-update.json` (Version, URL, Dateiname, Grösse und SHA-256 je Artefakt) und `rustdesk-update.json.sig` (Ed25519-Signatur). Der Client prüft die Signatur, bevor er die Metadaten auswertet, und akzeptiert nur Download-URLs unter dem Release von `rustdesk/rustdesk`. Schlägt die Prüfung fehl, bricht das Update ab; einen Rückfall auf unsignierte Daten gibt es nicht. Heruntergeladene Installer werden bis zum Start in einem geschützten Verzeichnis gehalten und vor der Ausführung mit erhöhten Rechten erneut gehasht.

Laut PR belegt die Signatur Echtheit und Integrität, aber nicht die Aktualität: Ein Schutz gegen das Wiedereinspielen älterer, korrekt signierter Releases (Rollback) ist nicht enthalten. Die portable Variante unter Windows entpackt und startet weiterhin aus einem für den Benutzer beschreibbaren Verzeichnis; die Härtung dafür ist laut PR auf einen separaten Branch verschoben.

## Trusted Devices V2

PR #16548 ersetzt die bisherige Vertrauensstellung über die Hardware-ID durch Tokens, die der Host ausstellt. Für eine neue Registrierung braucht es weiterhin das permanente Passwort und TOTP; ein Trusted-Device-Token erspart danach nur die TOTP-Eingabe. Der Host speichert lediglich den SHA-256-Hash des Tokens in verschlüsselter Form, der Controller bindet sein Token an den geprüften Public Key des Hosts. Bei direkten IP-Verbindungen wird keine Vertrauensstellung angeboten, da dort die Host-Identität nicht geprüft wird.

Für den Betrieb sind folgende Punkte aus dem PR relevant:

- **Gültigkeit:** einstellbar auf 1, 7, 30 oder 90 Tage oder unbegrenzt, Standard 90 Tage. Vertrauensstellungen lassen sich in den Einstellungen einzeln erneuern, entfernen oder gesammelt löschen.
- **Widerruf:** Eine Änderung von Passwort, 2FA oder der Trusted-Device-Einstellung widerruft die betroffenen Tokens. Der Host hält höchstens 128 Einträge.
- **Gemischte Versionen:** Bestehende, nicht abgelaufene Vertrauensstellungen bleiben nach dem Update des Hosts gültig. Neue Registrierungen setzen 1.5.1 auf beiden Seiten voraus; in gemischten Sitzungen fragt RustDesk TOTP ab und bietet „Trust this device“ nicht an.
- **Downgrade:** Laut PR nicht sicher, da ältere Versionen die importierten Vertrauensstellungen nicht lesen können.
- **Bekannte Grenzen:** Ein Widerruf wird zuerst lokal gespeichert und asynchron in die Root-Konfiguration übernommen; ein Neustart vor dem Abgleich kann eine ältere Vertrauensstellung zurückbringen. Das Ablaufdatum richtet sich nach der Systemuhr des Hosts.

Planen Sie das Update deshalb so, dass Hosts und Controller zeitnah auf 1.5.1 wechseln, und vermeiden Sie ein Zurücksetzen auf 1.5.0.

## Weitere sicherheitsrelevante Korrekturen

- **Download-Pfade (PR #16601):** Downloads lehnen unsichere Dateinamen und Pfade ab, die aus dem gewählten Zielverzeichnis herausführen oder über symbolische Links laufen. Entfernte Dateilisten werden vor dem Download geprüft, das Überschreiben bestehender Dateien und das Fortsetzen abgebrochener Downloads erfordern eine Bestätigung.
- **Windows-Installer (PR #16554, #16562):** Die Skripte für die Installer-Übergabe mit erhöhten Rechten liegen nicht mehr in `System32`, sondern in einem Verzeichnis unter Windows Temp mit eingeschränkter DACL (Administrators und SYSTEM). Der Dienst wird über die native Service Control gesteuert. Anlass war unter anderem, dass Bitdefender das temporäre Batch-Skript blockierte (Issue #16557).
- **Unverschlüsselter API-Server (PR #16616):** Die Server-Einstellungen zeigen eine Warnung, wenn der API-Server leer ist oder eine entfernte HTTP-Adresse nutzt. HTTPS sowie lokale Adressen lösen keine Warnung aus. Die Verbindung wird nicht blockiert.
- **Auto-Login (PR #16520):** Das automatische Login gibt das Betriebssystem-Passwort nur noch auf dem Anmelde- oder Sperrbildschirm ein.
- **Privacy Mode (PR #16507):** Eine abgelehnte Privacy-Mode-Anfrage schaltet den Privacy Mode einer anderen Verbindung nicht mehr ab.

Betreiben Sie einen eigenen Server mit einer `http://`-Adresse als API-Server, erscheint die neue Warnung bei Ihren Anwendern. Stellen Sie den API-Server auf HTTPS um, bevor Sie 1.5.1 verteilen, um Rückfragen zu vermeiden.

## Neue Funktionen und Änderungen

Neben den bereits genannten Punkten bringt 1.5.1 Tastenkürzel (PR #14933), im Adressbuch eine Schaltfläche zum Öffnen der Web-Konsole und unter Android die direkte Dateiübertragung in SAF-Ordner. Geändert wurden die Kodierung statischer Bildinhalte nach dem Scrollen (schärfere Darstellung) und die Refresh-Budgets für MediaCodec. Die Verarbeitung von `ConfigUpdate` wurde entfernt.

## Fehlerbehebungen nach Plattform

- **Linux, DRM-Capture:** korrekte Grösse bei rotierten Displays, kein Aufwecken schlafender Displays beim Start, Verwerfen doppelter Frames und an den Encoder angepasste Bildrate.
- **Linux, Wayland:** PipeWire-Capture ohne Wartezeiten, korrekte Zuordnung des Portal-Streams zum Ausgang, Tastaturlayout bei der Eingabe, und eine offene Portal-Anfrage blockiert den Server nicht mehr.
- **Linux, allgemein:** korrekter aktiver Benutzer unter Flatpak, Absturz beim Schliessen des GTK-Fensters, sudo-Abfrage ohne falsches „Sorry, try again.“ und mit Unterstützung für sudo-rs, xrandr-Ausgabenamen, RandR-Monitorabfrage unter X11.
- **Windows:** Lesen von `host=` aus Programmpfaden mit Nicht-ASCII-Zeichen, explizit benannte Executable für das Privacy-Fenster, QSV-Encoder-Hänger.
- **macOS:** Link zu den Einstellungen für Screen Recording, Loslassen der Maustaste.
- **Mobil:** kein verspätetes Mouse-Down mehr nach dem Loslassen des schwebenden Buttons.
- **Netzwerk:** HTTP-API-Fehler 407 über HTTPS-Proxys, IPv6-ID-Server ohne Klammern bei der Ableitung des API-Servers, eine Fehlermeldung statt zwei, wenn beide WebRTC-Verbindungsversuche scheitern.
- **Weitere:** gelöschte Drucker bleiben erhalten, Replay-Buffer im Terminal, Tastaturnavigation in der Dateiliste, Laden der Peers ohne gesetztes App-Verzeichnis, Klicks auf Peer-Karten im Web-Client.

## Empfohlenes Vorgehen

Zielversion für alle Clients ist 1.5.1. Aktualisieren Sie Hosts und Controller möglichst im selben Zeitraum, damit neue Trusted-Device-Registrierungen funktionieren, ein Downgrade auf 1.5.0 ist laut PR #16548 nicht sicher. Prüfen Sie vor dem Rollout, ob Ihr eigener API-Server per HTTPS erreichbar ist. Der Open-Source-Server muss für dieses Release nicht aktualisiert werden. Angaben zu Known Issues oder Supportfristen enthalten die Release Notes nicht (Stand 2. November 2026).

## Quellen

1.  [GitHub: RustDesk Release 1.5.1](https://github.com/rustdesk/rustdesk/releases/tag/1.5.1): Release Notes mit Datum, neuen Funktionen, Änderungen und Fehlerbehebungen.

2.  [GitHub: RustDesk Release 1.5.0](https://github.com/rustdesk/rustdesk/releases/tag/1.5.0): Vorgängerversion vom 30. September 2026.

3.  [GitHub: PR #15914, verify signed update metadata](https://github.com/rustdesk/rustdesk/pull/15914): Ablauf der Signaturprüfung, fehlender Rollback-Schutz, portable Variante.

4.  [GitHub: PR #16548, Trusted Devices V2](https://github.com/rustdesk/rustdesk/pull/16548): Token-Modell, Kompatibilität mit älteren Versionen, Downgrade und bekannte Grenzen.

5.  [GitHub: PR #16601, Download path validation](https://github.com/rustdesk/rustdesk/pull/16601): Prüfung von Dateinamen, Pfaden und symbolischen Links bei Downloads.

6.  [GitHub: PR #16554, Installer-Skripte in geschützten Temp-Verzeichnissen](https://github.com/rustdesk/rustdesk/pull/16554): Verschiebung aus System32, DACL und Bezug zu Bitdefender.

7.  [GitHub: PR #16616, Warnung bei unsicherem API-Server](https://github.com/rustdesk/rustdesk/pull/16616): Bedingungen und Text der neuen Warnung.

8.  [GitHub: RustDesk Security Advisories](https://github.com/rustdesk/rustdesk/security/advisories): keine veröffentlichten Advisories (Stand 2. November 2026).

9.  [GitHub: rustdesk-server Releases](https://github.com/rustdesk/rustdesk-server/releases): letzte Server-Version 1.1.16 vom 20. Juli 2026.

10.  [RustDesk: die quelloffene TeamViewer-Alternative einrichten](/blog/rustdesk-teamviewer-alternative): Einrichtung von Client und eigenem Server.
