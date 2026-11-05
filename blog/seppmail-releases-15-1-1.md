---
title: "SEPPmail 15.1.1: fünf CVE-Korrekturen und Fix für die DNS-Regression aus 15.1.0.1"
navTitle: "SEPPmail 15.1.1"
description: "SEPPmail 15.1.1 schliesst fünf Schwachstellen mit CVSS-Werten bis 9.2, darunter Codeausführung über Adressersetzungsregeln und Dateizugriff über Large File Transfer. Das Patch-Release behebt ausserdem eine DNS-Regression aus 15.1.0.1 und sollte zeitnah installiert werden."
date: "2026-11-03"
kategorie: "SEPPmail"
timeToRead: "5 Min. Lesezeit"
themen:
  - "seppmail"
  - "hin-gateway"
produkte:
  - "seppmail"
  - "seppmail-releases"
  - "hin"
protokolle:
  - "releases"
  - "haertung"
slug: "seppmail-releases-15-1-1"
translationId: article-e39f532b921dfc69
url: https://rafaelpfister.ch/blog/seppmail-releases-15-1-1
---

# SEPPmail 15.1.1: fünf CVE-Korrekturen und Fix für die DNS-Regression aus 15.1.0.1

SEPPmail hat am 27. Oktober 2026 das Patch-Release 15.1.1 veröffentlicht. Es schliesst fünf Schwachstellen mit CVSS-Werten zwischen 6.9 und 9.2, darunter eine mögliche Codeausführung über Adressersetzungsregeln und das Lesen oder Löschen beliebiger Dateien über manipulierte Large-File-Transfer-Anhänge. Dazu kommen eine Härtung der PGP-Verarbeitung, eine Korrektur der DNS-Auflösung, die mit 15.1.0.1 fehlerhaft wurde, sowie Änderungen an MPKI, S/MIME, Backup und Virtualisierung. Das Release richtet sich an alle Betreiber von SEPPmail-Appliances, auch an Appliances, die als HIN Mailgateway laufen, da diese auf der SEPPmail-Firmware basieren. Der vorherige Stand ist im Artikel zu [SEPPmail 15.0.7, 15.1.0 und 15.1.0.1](/blog/seppmail-releases-15-1-0-1) beschrieben.

## Die Versionen im Überblick

| Version | Datum | Art | Kernpunkte |
|---|---|---|---|
| 15.1.1 | 27.10.2026 | Patch-Release | CVE-2026-114424 und CVE-2026-114425 (je CVSS 9.2), CVE-2026-114426 (CVSS 8.7), CVE-2026-114420 (CVSS 8.3), CVE-2026-114421 (CVSS 6.9), PGP-Härtung, DNS-Regression aus 15.1.0.1 behoben, QEMU Guest Agent |

## Sicherheitskorrekturen in 15.1.1

Die Revision History führt fünf Einträge mit CVE-ID. Als betroffen nennt sie jeweils Versionen vor 15.1.1; eine CVSS-Version gibt sie nicht an.

- **CVE-2026-114424, CVSS 9.2:** Ersetzungswerte in Adressersetzungsregeln wurden fehlerhaft verarbeitet. Verwendet ein Regelsatz Werte aus der Nachricht als Ersatzwert, konnte ein entfernter Absender Befehle auf der Appliance ausführen. Die Auswertung wurde gehärtet.
- **CVE-2026-114425, CVSS 9.2:** Speziell gestaltete Anhangs-Metadaten in eingehenden Nachrichten konnten die Large-File-Transfer-Verarbeitung (LFT) dazu bringen, beliebige Dateien zu lesen oder zu löschen. Die Metadaten werden nun vor jedem Dateizugriff validiert.
- **CVE-2026-114426, CVSS 8.7:** Die Upload-Schnittstelle verarbeitete Anfragedaten, bevor die Authentifizierung geprüft war. Nicht authentifizierte Angreifer konnten so Speicherplatz und Worker-Prozesse erschöpfen. Die Authentifizierung erfolgt nun zuerst, zusätzlich ist die Upload-Grösse begrenzt.
- **CVE-2026-114420, CVSS 8.3, und CVE-2026-114421, CVSS 6.9:** Unzureichendes Escaping bei Kontosuchen und eine fehlende Prüfung im Ablauf zur Passwort-Wiederherstellung ermöglichten die Übernahme von GINA-Konten. LDAP-Filterwerte werden nun escaped, und die Wiederherstellung fragt nur noch registrierte Benutzer mit konfigurierten Sicherheitsfragen ab. Die Revision History beschreibt beide CVEs mit identischem Text und identischer Korrektur, nennt aber unterschiedliche CVSS-Werte.

Zwei weitere Punkte betreffen PGP, tragen aber keine CVE-ID: Schlüssel mit User-IDs, die nicht selbstsigniert sind, werden nicht mehr akzeptiert, und PGP-Nachrichten ohne tatsächliche Verschlüsselung (NULL-Verschlüsselung) werden erkannt und nicht mehr als verschlüsselt markiert.

Zum Stand 3. November 2026 sind die fünf CVE-Einträge weder bei CVE.org noch in der NVD veröffentlicht. Vektoren, CWE-Zuordnung und eine Einschätzung zur Ausnutzung liegen deshalb noch nicht vor; die CVSS-Werte stammen aus der Revision History. Zwei der Lücken (CVE-2026-114424 und CVE-2026-114425) lassen sich nach Herstellerbeschreibung über eingehende Nachrichten auslösen, CVE-2026-114426 ohne Authentifizierung. Prüfen Sie bis zum Update, welche Adressersetzungsregeln Werte aus der Nachricht übernehmen, und ob LFT und die GINA-Passwort-Wiederherstellung aktiv sind.

Für die beiden Sicherheitskorrekturen aus 15.1.0 (Root-CA-Verwaltung per REST-API mit CVSS 9.9 und Stored XSS im Log-Viewer mit CVSS 7.1) steht in der Revision History weiterhin „CVE-ID: Pending“ (Stand 3. November 2026).

## Neue Funktionen und Änderungen

- **Virtualisierung:** Die Appliance unterstützt den QEMU Guest Agent (qemu-ga).
- **Admin:** Eine neue Systemoption deaktiviert das Sammeln der Customer-Summary-Daten, das auf grossen Installationen laut Hersteller lange dauern kann.
- **Backup im Cluster:** Alle Cluster-Mitglieder erstellen ein Datenbank-Backup, versendet wird es nur vom Maintenance-Node. Ein fehlender Backup-Operator erscheint im Daily Report nicht mehr als Fehler, da grosse Datenbanken meist per SCP gesichert werden.
- **Core Dumps:** Die Standardgrösse ist begrenzt, damit Core Dumps die `/tmp`-Partition nicht füllen.
- **RuleEngine:** Fehlt beim Signieren oder Verschlüsseln das Schlüsselmaterial, erzeugt die Appliance es automatisch (bei Bedarf mit lokalem Benutzer), sofern die entsprechenden Optionen aktiviert sind. Andernfalls wird die Nachricht nicht mehr abgewiesen.
- **S/MIME:** Sind die automatische Löschung und die tägliche Sperrprüfung aktiviert, werden widerrufene Zertifikate ebenfalls automatisch gelöscht.
- **MPKI:** Der D-Trust-Connector entspricht der Connector-Spezifikation 2.28, die GlobalSign-Root-CA-Kette wurde aktualisiert, und nicht rechtzeitig ausgestellte SwissSignREST-Bestellungen bleiben als ausstehend erhalten und werden später abgeschlossen, damit keine Duplikate entstehen.
- **Disclaimer:** Die Content-ID-Header von Inline-Bildern und Anhängen entsprechen RFC 2045.
- **Outlook Add-In:** Die Unterstützung für HIN ACS im COM Outlook Add-In wurde entfernt, da HIN laut Release Notes eine eigene Software bereitstellt.

Betreiber eines HIN Mailgateway, deren Benutzer HIN ACS über das COM Outlook Add-In nutzen, sollten vor dem Update klären, welche HIN-Software diese Funktion übernimmt. Ist die automatische Zertifikatslöschung zusammen mit der täglichen Sperrprüfung aktiviert, entfernt die Appliance nach dem Update auch widerrufene S/MIME-Zertifikate.

## Fehlerbehebungen

- **Betriebssystem:** Nach dem Einrichten einer lokalen DNS-Zone mit dem integrierten Resolver liessen sich seit 15.1.0.1 externe DNS-Namen nicht mehr auflösen.
- **Admin:** Die HTML-Vorschau für Disclaimer funktionierte seit 15.0.6.1 nicht. Disclaimer- und Template-Dateien werden nun in UTF-8 geschrieben, Sonderzeichen werden dadurch nicht mehr beschädigt. Die Seite mit den Managed-Domain-Einstellungen lädt nach dem Anlegen einer Managed Domain korrekt.
- **Logging:** Die Datenbank-Logging-Seite lädt auch bei Einträgen mit leeren Daten, rotierte Mail-Logs sind wieder einsehbar, und der Neuaufbau des Mail-Log-Archivindex läuft ohne Fehler.
- **REST-API:** `PUT /v1/ssl` liefert die eigentliche Fehlermeldung statt eines internen Serverfehlers, und `PUT /webmail/domain/{domainName}` funktioniert mit aktivierter Option „Initial password via SMS“.
- **S/MIME:** Das Ergebnis der CRL-Prüfung überschreibt nun das OCSP-Ergebnis, sodass widerrufene Zertifikate als widerrufen markiert werden.
- **RuleEngine:** `make_lft()` funktioniert, wenn ein LFT mehrere Empfänger hat und nur ein Teil davon berechtigt ist.
- **Webmail (GINA):** Lesebestätigungen gehen bei Nachrichten an mehrere Empfänger an alle Empfänger statt nur an den ersten.
- **MPKI:** Die Einstellungen des SwissSignREST-Connectors werden in der Admin-GUI geladen.

## Bekannte Probleme

- **DNS-Auflösung mit lokaler DNS-Zone (15.1.0.1):** behoben in 15.1.1.
- **HTML-Vorschau für Disclaimer (seit 15.0.6.1):** behoben in 15.1.1.
- **CVE-IDs für die Sicherheitskorrekturen aus 15.1.0:** weiterhin offen („Pending“, Stand 3. November 2026).

Einen Abschnitt zu Known Issues enthalten die Release Notes für 15.1.1 nicht.

## Empfohlenes Vorgehen

Zielversion ist 15.1.1, da alle fünf neuen CVEs Versionen vor 15.1.1 betreffen und zwei davon über eingehende Nachrichten ausnutzbar sind. Laut Revision History ermöglicht 15.0.7 das Update auf 15.1; Systeme im 15.0-Zweig aktualisieren Sie deshalb zuerst auf 15.0.7 und anschliessend auf 15.1.1. Erstellen Sie vor dem Update ein Backup und bei virtuellen Appliances zusätzlich einen Snapshot, wie es die Herstellerdokumentation empfiehlt. Angaben zur Update-Reihenfolge im Cluster oder zu einer Mindestversion für 15.1.1 enthalten die Release Notes nicht (Stand 3. November 2026).

## Quellen

1.  [SEPPmail-Dokumentation: Revision History](https://docs.seppmail.com/ch/20_revision-history.html): Release Notes zu 15.1.1 mit Datum, Sicherheitskorrekturen, CVSS-Werten, Änderungen und Fehlerbehebungen sowie Status der CVE-IDs aus 15.1.0.

2.  [SEPPmail: Extended Release Notes 15.1](https://downloads.seppmail.com/extrelnotes/151/ERN15.1.html): Ausführliche Release Notes zu 15.1.0 und 15.1.0.1; ein Eintrag zu 15.1.1 fehlt dort zum Stand 3. November 2026.

3.  [NVD: CVE-2026-114424](https://nvd.nist.gov/vuln/detail/CVE-2026-114424): Zum Stand 3. November 2026 noch kein Eintrag, gilt auch für CVE-2026-114420, -114421, -114425 und -114426.

4.  [CVE.org: CVE-2026-114424](https://www.cve.org/CVERecord?id=CVE-2026-114424): Zum Stand 3. November 2026 noch nicht veröffentlicht.

5.  [SEPPmail 15.0.7, 15.1.0 und 15.1.0.1: CVE-Korrekturen und neuer Feature-Zweig](/blog/seppmail-releases-15-1-0-1): Vorheriger Release-Stand mit den Korrekturen aus 15.1.0 und dem Update-Pfad über 15.0.7.
