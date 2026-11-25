---
title: "SEPPmail 15.1.2: elf CVE-Korrekturen, vor allem in GINA, und Post-Quantum-TLS"
navTitle: "SEPPmail 15.1.2"
description: "SEPPmail 15.1.2 schliesst elf Schwachstellen mit CVSS-Werten bis 9.3, die meisten davon in GINA und Large File Transfer. Dazu kommen X25519MLKEM768 für den Webserver, ein CSV-Export im Log-View und strengere REST-API-Logins. Das Update sollte zeitnah installiert werden."
date: "2026-11-25"
kategorie: "SEPPmail"
timeToRead: "6 Min. Lesezeit"
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
  - "tls"
slug: "seppmail-releases-15-1-2"
translationId: article-0c960bb6538915dc
url: https://rafaelpfister.ch/blog/seppmail-releases-15-1-2
---

# SEPPmail 15.1.2: elf CVE-Korrekturen, vor allem in GINA, und Post-Quantum-TLS

SEPPmail hat am 24. November 2026 das Patch-Release 15.1.2 veröffentlicht. Es schliesst elf Schwachstellen mit CVSS-Werten zwischen 6.3 und 9.3, die alle Versionen vor 15.1.2 betreffen. Acht davon liegen in GINA (Webmail), darunter eine Template-Injection auf der Login-Seite und zwei Wege zur Übernahme registrierter Konten; eine weitere betrifft einen fest codierten Ersatzschlüssel bei Large File Transfer (LFT). Neu sind ausserdem Post-Quantum-Schlüsselaustausch (X25519MLKEM768) in der TLS-Konfiguration des Webservers, ein CSV-Export im Log-View, eine grössere `/etc`-Partition und eine REST-API-Anmeldung, die die MFA-Richtlinie für Administratoren durchsetzt. Das Release richtet sich an alle Betreiber von SEPPmail-Appliances, auch an Appliances, die als HIN Mailgateway laufen, da diese auf der SEPPmail-Firmware basieren. Der vorherige Stand ist im Artikel zu [SEPPmail 15.1.1](/blog/seppmail-releases-15-1-1) beschrieben.

## Die Versionen im Überblick

| Version | Datum | Art | Kernpunkte |
|---|---|---|---|
| 15.1.2 | 24.11.2026 | Patch-Release | Elf CVEs (CVSS 6.3–9.3), höchster Wert CVE-2026-130637 (CVSS 9.3, GINA-Login), X25519MLKEM768 für den Webserver, CSV-Export im Log-View, MFA-Durchsetzung beim REST-Login, LFT ohne fest codierten Ersatzschlüssel |

## Sicherheitskorrekturen in 15.1.2

Die Revision History und die Extended Release Notes führen elf Einträge mit CVE-ID. Als betroffen nennen sie jeweils Versionen vor 15.1.2; eine CVSS-Version oder einen Vektor geben sie nicht an.

**GINA-Konten und Login:**

- **CVE-2026-130637, CVSS 9.3:** Werte aus Anfragen an die GINA-Login-Seite wurden von der Template-Engine ausgewertet. Laut Extended Release Notes konnte ein nicht angemeldeter Angreifer so Attribute beliebiger GINA-Konten auslesen und Konten übernehmen. Anfragewerte werden nicht mehr als Template-Platzhalter ausgewertet.
- **CVE-2026-124840, CVSS 9.2:** Eine Sitzung aus der Wiederherstellung per Sicherheitsfrage wurde auch für Kontoregistrierung und Einstellungen akzeptiert. Wer die Antwort auf die Sicherheitsfrage kannte, konnte das Passwort ändern. Die Wiederherstellung nutzt nun einen eigenen Sitzungszweck, der nur in diesem Ablauf gilt.
- **CVE-2026-124842, CVSS 8.3:** Eine nicht authentifizierte Anfrage konnte das Passwort eines registrierten GINA-Kontos zurücksetzen, ohne dass die Sicherheitsfrage beantwortet wurde. Das Zurücksetzen ohne Authentifizierung gilt nur noch für nicht registrierte Konten.
- **CVE-2026-124841, CVSS 6.9:** Platzhalterzeichen in Kontoadressen wurden in LDAP-Abfragen von GINA nicht escaped, eine Abfrage konnte dadurch ein anderes Konto treffen. Alle Werte in diesen LDAP-Filtern werden nun escaped.
- **CVE-2026-124845, CVSS 6.3:** Ein vom Client gesetzter HTTP-Header konnte die Auflösung der GINA-Domain scheitern lassen; die Anfrage lief dann mit den Einstellungen der `[default]`-Domain. Anfragen für nicht auflösbare GINA-Domains werden nun abgelehnt.
- **CVE-2026-124844, CVSS 8.7:** Dateien aus nicht authentifizierten GINA-Uploads wurden nicht gelöscht, wodurch der Speicherplatz volllaufen konnte. Uploads sind nun auf die maximale Nachrichtengrösse der GINA-Domain begrenzt, nicht benötigte Dateien werden entfernt.

**Large File Transfer:**

- **CVE-2026-124843, CVSS 9.2:** Fehlte das Schlüsselmaterial der Appliance, verwendete die LFT-Verschlüsselung ein fest codiertes Ersatzgeheimnis. Anhänge liessen sich dadurch ausserhalb der Appliance entschlüsseln. Der Ersatz entfällt; ohne den benötigten Schlüssel verweigert die Appliance die LFT-Verschlüsselung.

**Rechteausweitung auf der Appliance:**

- **CVE-2026-124838, CVSS 8.5:** Der lokale Socket des Konfigurationsdienstes war für alle lokalen Benutzer erreichbar. Ein Angreifer mit lokaler Codeausführung konnte darüber Befehle mit erhöhten Rechten ausführen. Der Zugriff ist nun auf berechtigte Systemkonten beschränkt, übergebene Befehlsdaten werden escaped.
- **CVE-2026-124839, CVSS 7.7:** Eine präparierte GINA-Kontoadresse konnte beim Systemstart über ein Wartungsskript Befehle mit erhöhten Rechten auslösen. Die Kontoadresse wird nicht mehr an einen Shell-Befehl übergeben.

**Mailverarbeitung:**

- **CVE-2026-124836, CVSS 6.3:** Die Regelbefehle `ldap_compare` und `ldap_read` setzten Werte aus der Nachricht ohne Escaping in LDAP-Filter ein. Ein Absender konnte so das Ergebnis einer LDAP-Regel beeinflussen. Die Werte werden nun escaped.
- **CVE-2026-124837, CVSS 6.3:** Eine präparierte DKIM-Signatur konnte gefälschte Einträge in den Header `Authentication-Results` einschleusen, die anschliessend vom ARC-Siegel des Gateways gedeckt wurden. Werte aus der DKIM-Signatur werden nun vor der Übernahme validiert.

Ohne CVE-ID nennen die Release Notes zusätzlich eine Härtung interner Hilfsfunktionen zur Befehlsausführung.

Zum Stand 25. November 2026 liefert die NVD für CVE-2026-130637 und CVE-2026-124840 keinen Eintrag, und CVE.org führt CVE-2026-124840 nicht als veröffentlicht. Vektoren, CWE-Zuordnung und eine Einschätzung zur Ausnutzung liegen deshalb noch nicht vor; die CVSS-Werte stammen aus den Release Notes des Herstellers. Mehrere Lücken in GINA lassen sich nach Herstellerbeschreibung ohne Anmeldung auslösen, CVE-2026-124836 und CVE-2026-124837 über eingehende Nachrichten. Prüfen Sie bis zum Update, ob GINA aus dem Internet erreichbar ist, und ob Regeln mit `ldap_compare` oder `ldap_read` Werte aus der Nachricht verwenden.

## Neue Funktionen und Änderungen

- **Post-Quantum-TLS:** In den TLS-Einstellungen der GINA-Domains heisst die Einstellung „Curves“ nun „Groups“. Dort lassen sich Gruppen mit Post-Quantum-Unterstützung wie X25519MLKEM768 auswählen; die Einstellung wirkt auf die Konfiguration des Webservers (httpd). Standardwerte nennen die Release Notes nicht.
- **Log-View:** Eine neue Schaltfläche „Export CSV...“ exportiert alle angezeigten bzw. gefilterten Einträge, sowohl beim datei- als auch beim datenbankbasierten Logging.
- **Betriebssystem:** Die `/etc`-Partition wurde vergrössert, da sie in manchen Installationen zu klein war und Konfigurationsdateien gelegentlich nicht geschrieben werden konnten.
- **REST-API:** Der Endpunkt `/auth` berücksichtigt die MFA-Einstellungen für Administratoren, fehlgeschlagene Logins erhalten einheitlich HTTP 403, und erfolgreiche Logins werden protokolliert. REST-API-Tokens von Benutzern werden nach dem für die Admin-GUI konfigurierten Timeout ungültig. In mehreren REST-Endpunkten werden Werte des Clients in LDAP-Suchfiltern escaped.
- **LFT:** Die Verschlüsselung verwendet keinen fest codierten Ersatzschlüssel mehr (siehe CVE-2026-124843).

Zwei Änderungen können bestehende Abläufe betreffen. Skripte und Integrationen, die sich mit einem Administratorkonto an der REST-API anmelden, müssen nach dem Update die MFA-Richtlinie erfüllen, und lange laufende Automatisierungen mit Benutzer-Tokens laufen in den Admin-GUI-Timeout; testen Sie solche Integrationen nach dem Update. Da LFT ohne vorhandenes Schlüsselmaterial nicht mehr verschlüsselt, prüfen Sie nach dem Update, ob LFT-Versände weiterhin funktionieren.

## Fehlerbehebungen

- **Background Tasks:** Ein GINA-Satellit konnte keine Benutzer und Schlüssel senden.
- **CfgServer:** Die S/MIME-Signaturerzeugung für die automatische Vertrauensstellung von Domain-Zertifikaten funktioniert wieder.
- **Datenbank:** Eine LDAP-Änderung wiederholte frühere Änderungen, wenn derselbe Eintrag erneut bearbeitet wurde.
- **Large Files:** Verwaiste Dateien werden bei der Bereinigung abgelaufener LFT-Daten korrekt behandelt.
- **Logging:** Die Fortschrittsanzeige beim Neuaufbau des Datenbank-Log-Caches ist korrigiert; laut Extended Release Notes fehlten zudem CC-Empfänger im Log-View.
- **Webmail (GINA):** Ein Doppelklick auf „Senden“ verschickte eine Antwort zweimal.

## Bekannte Probleme

- **CVE-IDs für die Sicherheitskorrekturen aus 15.1.0:** weiterhin offen. Die Revision History führt für beide Einträge (CVSS 9.9 und 7.1) unverändert „CVE-ID: Pending“ (Stand 25. November 2026).
- **NVD-Einträge zu den CVEs aus 15.1.1:** inzwischen teilweise veröffentlicht. CVE-2026-114424 steht seit dem 4. November 2026 in der NVD, mit dem NCSC als Quelle und CVSS 4.0 mit 9.2; die NVD beschreibt den bestätigten Ausführungskontext als root.

Einen Abschnitt zu Known Issues enthalten die Release Notes für 15.1.2 nicht.

## Empfohlenes Vorgehen

Zielversion ist 15.1.2, da alle elf CVEs Versionen vor 15.1.2 betreffen und mehrere davon ohne Anmeldung an GINA oder über eingehende Nachrichten auslösbar sind. Systeme im 15.0-Zweig aktualisieren Sie wie im Artikel zu 15.1.1 beschrieben zuerst auf 15.0.7, das laut Revision History das Update auf 15.1 ermöglicht. Erstellen Sie vor dem Update ein Backup und bei virtuellen Appliances zusätzlich einen Snapshot, wie es die Herstellerdokumentation empfiehlt. Testen Sie danach REST-API-Integrationen mit Administratorkonten und den LFT-Versand. Angaben zur Update-Reihenfolge im Cluster oder zu einer Mindestversion für 15.1.2 enthalten die Release Notes nicht (Stand 25. November 2026).

## Quellen

1.  [SEPPmail-Dokumentation: Revision History](https://docs.seppmail.com/ch/20_revision-history.html): Release Notes zu 15.1.2 mit Datum, elf CVEs mit CVSS-Werten, neuen Funktionen und Fehlerbehebungen sowie Status der CVE-IDs aus 15.1.0.

2.  [SEPPmail: Extended Release Notes 15.1](https://downloads.seppmail.com/extrelnotes/151/ERN15.1.html): Ausführliche Beschreibung von 15.1.2, unter anderem zur Umbenennung von „Curves“ in „Groups“, zum REST-API-Token-Timeout und zur LFT-Verschlüsselung.

3.  [NVD: CVE-2026-130637](https://nvd.nist.gov/vuln/detail/CVE-2026-130637): Zum Stand 25. November 2026 noch kein Eintrag, ebenso für CVE-2026-124840.

4.  [CVE.org: CVE-2026-124840](https://www.cve.org/CVERecord?id=CVE-2026-124840): Zum Stand 25. November 2026 nicht veröffentlicht.

5.  [NVD: CVE-2026-114424](https://nvd.nist.gov/vuln/detail/CVE-2026-114424): Eintrag zu einer der Lücken aus 15.1.1, veröffentlicht am 4. November 2026, CVSS 4.0 mit 9.2.

6.  [SEPPmail 15.1.1: fünf CVE-Korrekturen und Fix für die DNS-Regression aus 15.1.0.1](/blog/seppmail-releases-15-1-1): Vorheriger Release-Stand mit den Korrekturen aus 15.1.1 und dem Update-Pfad über 15.0.7.
