---
title: "SEPPmail 15.0.7, 15.1.0 und 15.1.0.1: CVE-Korrekturen und neuer Feature-Zweig"
navTitle: "SEPPmail 15.1.0.1"
description: "SEPPmail 15.0.7 schliesst eine Command Injection (CVE-2026-84830) und eine MFA-Umgehung (CVE-2026-84831) und ist das letzte Release im 15.0-Zweig. 15.1.0 und der Hotfix 15.1.0.1 bringen neue Admin- und Überwachungsfunktionen sowie zwei weitere Sicherheitskorrekturen, darunter eine mit CVSS 9.9."
date: "2026-10-26"
kategorie: "SEPPmail"
timeToRead: "6 Min. Lesezeit"
themen:
  - "seppmail"
produkte:
  - "seppmail"
  - "seppmail-releases"
protokolle:
  - "releases"
  - "haertung"
slug: "seppmail-releases-15-1-0-1"
translationId: article-33610202fa8d9241
url: https://rafaelpfister.ch/blog/seppmail-releases-15-1-0-1
---

# SEPPmail 15.0.7, 15.1.0 und 15.1.0.1: CVE-Korrekturen und neuer Feature-Zweig

SEPPmail hat am 2. September 2026 das Patch-Release 15.0.7 veröffentlicht, am 28. September 2026 das Feature-Release 15.1.0 und am 29. September 2026 den Hotfix 15.1.0.1. Das Patch-Release schliesst mehrere Schwachstellen mit CVSS-Werten zwischen 7.7 und 8.6, aktualisiert OpenSSL und ist laut Hersteller das letzte Release im 15.0-Zweig, das zugleich das Update auf 15.1 ermöglicht. 15.1.0 bringt neue Funktionen für Administration, Überwachung, REST-API und Webmail sowie zwei weitere Sicherheitskorrekturen, eine davon mit CVSS 9.9. Die Releases richten sich an alle Betreiber von SEPPmail-Appliances, auch an Appliances, die als HIN Mailgateway laufen, da diese auf der SEPPmail-Firmware basieren. Der vorherige Stand ist im Artikel zu [SEPPmail 15.0.6 und 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1) beschrieben.

## Die Versionen im Überblick

| Version | Datum | Art | Kernpunkte |
|---|---|---|---|
| 15.0.7 | 02.09.2026 | Patch-Release | CVE-2026-84830 (CVSS 8.6), CVE-2026-84831 (CVSS 7.7), OpenSSL 3.0.22, letztes Release im 15.0-Zweig |
| 15.1.0 | 28.09.2026 | Feature-Release | Root-CA-Zugriff per REST-API (CVSS 9.9), Stored XSS im Log-Viewer (CVSS 7.1), REST-API-Auditing, neue Benachrichtigungen, OpenSSL 3.5.8 |
| 15.1.0.1 | 29.09.2026 | Hotfix | interner Serverfehler beim Bearbeiten einer GINA-Domain |

## Sicherheitskorrekturen in 15.0.7

Die Revision History nennt für 15.0.7 drei Sicherheitseinträge. Zwei davon sind seit dem 3. September 2026 als CVE veröffentlicht; CNA ist das NCSC, gefunden hat sie Oualid Lkhaouni (Emposo GmbH).

- **CVE-2026-84830, CVSS 8.6 (v4.0):** Command Injection (CWE-78) in der privilegierten Konfigurationsverarbeitung. Authentifizierte Administratoren konnten Befehle mit erhöhten Rechten ausführen. Betroffen sind alle Versionen vor 15.0.7.
- **CVE-2026-84831, CVSS 7.7 (v4.0):** Umgehung der Pflicht-MFA vor der Registrierung (CWE-287, CWE-306). Die Appliance erstellte eine voll privilegierte Sitzung, bevor die vorgeschriebene MFA-Registrierung abgeschlossen war. Wer das Passwort eines MFA-pflichtigen, aber noch nicht registrierten Kontos kennt, konnte geschützte Funktionen ohne zweiten Faktor nutzen. Betroffen sind Versionen vor 15.0.7.
- **CVE-2026-84832, CVSS 8.6 (v4.0):** unsichere Deserialisierung in einem privilegierten REST-Importablauf (CWE-502). Ein Angreifer mit einem privilegierten API-Token konnte beliebige Befehle mit den Rechten des Benutzers „nobody“ ausführen. Der Eintrag steht im 15.0.7-Abschnitt, als betroffen nennen Release Notes und CVE-Eintrag jedoch Versionen vor 15.0.6. Systeme auf 15.0.6 oder 15.0.6.1 sind nach diesen Angaben bereits geschützt.

Die CISA-ADP-Bewertung in den CVE-Einträgen lautet bei allen drei Lücken „Exploitation: none“ (Stand 8. Oktober 2026). Alle drei setzen eine Authentifizierung voraus: ein Admin-Konto, ein bekanntes Passwort eines noch nicht MFA-registrierten Kontos oder ein privilegiertes API-Token. Prüfen Sie deshalb neben dem Update, welche Konten MFA-pflichtig, aber noch nicht registriert sind, und welche API-Token mit welchen Rechten ausgestellt sind.

## Weitere Änderungen in 15.0.7

OpenSSL wurde auf 3.0.22 aktualisiert. Die Release Notes nennen dazu CVE-2026-63072, CVE-2026-63076, CVE-2026-54874, CVE-2026-63074 und CVE-2026-75803 sowie eine korrigierte Prüfung von Authentifizierungs-Tags bei leeren Chiffretexten im CCM-Modus.

Die Fehlerbehebungen betreffen mehrere Bereiche:

- **Admin:** Lokale Benutzer liessen sich nach einer Zertifikatssperrung nicht löschen, und Zeitzoneneinstellungen wurden nicht korrekt gespeichert (auch im Cfgserver).
- **Clustering:** `connector.pl` beendete initiale Injection-Verbindungen nicht. Bei der Anzeige des Cluster-Status erscheinen keine unnötigen SSH-Warnungen mehr.
- **Hintergrundaufgaben:** Noch laufende Prozesse werden so behandelt, dass sich Prozesse nicht mehr überschneiden.
- **RuleEngine:** Die Verwaltung des temporären Verzeichnisses wurde korrigiert, LFT-Nachrichten fehlten teilweise verschlüsselte Anhänge, und übersprungene Sperrprüfungen werden besser protokolliert.
- **Webmail (GINA):** Zwischengespeicherte Daten wurden gegen einen falschen Wert validiert, die Sitzungsbehandlung wurde korrigiert.

Die Zeitzonenkorrektur schliesst an 15.0.6 an, das die Appliance auf gültige Zeitzonen (Standard Europe/Zurich) umgestellt hatte.

## Sicherheitskorrekturen in 15.1.0

15.1.0 enthält zwei Sicherheitskorrekturen, beide laut Release Notes aus einem Audit von Emposo. Eine CVE-ID war in der Revision History zum Stand 26. Oktober 2026 für beide noch ausstehend („Pending“).

- **Root-CA-Verwaltung per REST-API, CVSS 9.9:** Unzureichende Berechtigungsprüfung in der Krypto-Verwaltungs-API. Ein API-Token ohne MSP-Rechte konnte RootCA-Zertifikate hinzufügen, ändern oder löschen. Der Zugriff auf den Endpunkt `/crypto/rootca` ist nun auf MSP-Token beschränkt. Die Revision History nennt als betroffen Versionen unter 15.0.6.1, führt die Korrektur aber unter 15.1.0. Ob 15.0.6.1 und 15.0.7 bereits geschützt sind, lässt sich aus den Quellen nicht eindeutig ableiten (Stand 26. Oktober 2026).
- **Stored XSS im Log-Viewer, CVSS 7.1:** Über eine manipulierte Message-ID liess sich JavaScript in die Admin-GUI einschleusen. Externe Daten werden nun escaped. Betroffen sind Versionen unter 15.1.0.

Zusätzlich wurden alle `qx(...)`-Aufrufe durch `run_shell_safe` ersetzt, damit Argumente beim Shell-Aufruf escaped werden; der Hersteller führt das als Wartungsarbeit. Da die Root-CA-Lücke über ein API-Token ausgenutzt wird, sollten Sie die vergebenen REST-API-Token und deren Rechte überprüfen, bis 15.1.0.1 installiert ist.

## Neue Funktionen in 15.1.0

- **Administration:** Das Wachstum der LDAP-Datenbank wird überwacht. Delegierte Administratoren sehen nur noch Konten ihres Mandanten. Für externe Keyserver gibt es einen Verbindungstest.
- **Benachrichtigungen:** Die Appliance meldet fehlgeschlagene E-Mail-Sicherungen, benachrichtigt Kommunikationspartner über ablaufende Zertifikate und warnt, wenn der LFM-Speicher seine Kapazitätsgrenze erreicht.
- **REST-API:** Anfragen an die REST-API lassen sich auditieren.
- **RuleEngine:** Nachrichten erhalten Metadaten zu den ausgeführten Aktionen, und benutzerdefinierte Befehle können PostgreSQL anbinden.
- **Webmail (GINA):** Die Dateitypen, die externe Partner über GINA hochladen dürfen, lassen sich einschränken.

Geändert wurden ausserdem die LDAP-Synchronisation für mittlere und grosse Installationen, die Prüfung auf Zertifikatssperrungen und die Darstellung generierter E-Mail-Benachrichtigungen. OpenSSL steht nun auf 3.5.8, OpenSSH (sshd) auf 10.4. Eine unvollständige API-Definition für Zertifikatsinformationen wurde korrigiert.

Die Upload-Einschränkung in GINA und das REST-API-Auditing lohnt es sich nach dem Update zu konfigurieren, besonders wenn externe Partner Dateien über GINA einreichen oder mehrere Systeme per API auf die Appliance zugreifen.

## Hotfix 15.1.0.1

Der Hotfix vom 29. September 2026 behebt einen internen Serverfehler auf der Seite zum Bearbeiten einer GINA-Domain. Weitere Punkte nennen die Release Notes nicht.

## Empfohlenes Vorgehen

Zielversion ist 15.1.0.1. Laut Revision History ist 15.0.7 das Release, das das Update auf 15.1 ermöglicht; Systeme auf 15.0.6.1 oder älter aktualisieren Sie deshalb zuerst auf 15.0.7 und anschliessend auf 15.1.0.1. Wer vorerst im 15.0-Zweig bleibt, sollte mindestens 15.0.7 installieren, da dort CVE-2026-84830 und CVE-2026-84831 behoben sind; die Root-CA-Korrektur und die XSS-Korrektur aus 15.1.0 sind in den Release Notes nur für 15.1 aufgeführt. Erstellen Sie vor jedem Update ein Backup und bei virtuellen Appliances zusätzlich einen Snapshot, wie es die Herstellerdokumentation empfiehlt. Angaben zu Known Issues, Supportende des 15.0-Zweigs oder zur Update-Reihenfolge im Cluster enthalten die Release Notes für diese Versionen nicht (Stand 26. Oktober 2026).

## Quellen

1.  [SEPPmail-Dokumentation: Revision History](https://docs.seppmail.com/ch/20_revision-history.html): Release Notes zu 15.0.7, 15.1.0 und 15.1.0.1 mit Datum, Sicherheitskorrekturen und Hinweis zum Update-Pfad.

2.  [SEPPmail: Extended Release Notes 15.0](https://downloads.seppmail.com/extrelnotes/150/ERN15.0.html): Ausführliche Release Notes zu 15.0.7 mit den internen Nummern SEC-99 und SEC-100.

3.  [SEPPmail: Extended Release Notes 15.1](https://downloads.seppmail.com/extrelnotes/151/ERN15.1.html): Ausführliche Release Notes zu 15.1.0 und 15.1.0.1, inklusive Details zur Root-CA- und XSS-Korrektur.

4.  [CVE.org: CVE-2026-84830](https://www.cve.org/CVERecord?id=CVE-2026-84830): Command Injection, CVSS 4.0 8.6, betroffen vor 15.0.7.

5.  [CVE.org: CVE-2026-84831](https://www.cve.org/CVERecord?id=CVE-2026-84831): MFA-Umgehung vor der Registrierung, CVSS 4.0 7.7, betroffen vor 15.0.7.

6.  [CVE.org: CVE-2026-84832](https://www.cve.org/CVERecord?id=CVE-2026-84832): Unsichere Deserialisierung in der REST-Schnittstelle, CVSS 4.0 8.6, betroffen vor 15.0.6.

7.  [SEPPmail 15.0.6 und 15.0.6.1: Sicherheitskorrekturen und neue Admin-Funktionen](/blog/seppmail-releases-15-0-6-und-15-0-6-1): Vorheriger Release-Stand mit den Korrekturen aus 15.0.6.
