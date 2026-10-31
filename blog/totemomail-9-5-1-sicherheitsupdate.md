---
title: "totemomail 9.5.1: Account-Übernahme im Web Portal und SSRF über eingehende Mails geschlossen"
navTitle: "totemomail 9.5.1"
description: "totemomail 9.5.1 schliesst acht Sicherheitslücken. Zwei davon lassen sich ohne Anmeldung ausnutzen: ein Fehler im Passwort-Reset des Web Portals, der die Übernahme von Administratorkonten erlaubt, und serverseitige Abrufe von URLs aus eingehenden Nachrichten. Dazu kommen Path-Traversal-Fixes in Admin API und Mail API sowie aktualisierte Tomcat- und Batik-Bibliotheken."
date: "2026-10-31"
kategorie: "Totemomail"
timeToRead: "5 Min. Lesezeit"
themen:
  - "totemomail"
  - "sicherheitsluecken"
produkte:
  - "totemomail"
  - "sicherheitsluecken"
protokolle:
  - "releases"
  - "haertung"
slug: "totemomail-9-5-1-sicherheitsupdate"
translationId: "article-2bd15f63632ab728"
url: "https://rafaelpfister.ch/blog/totemomail-9-5-1-sicherheitsupdate"
draft: false
---
# totemomail 9.5.1: Account-Übernahme im Web Portal und SSRF über eingehende Mails geschlossen

totemo hat totemomail 9.5.1 (Build 101) veröffentlicht. Der Patch enthält acht Sicherheitskorrekturen, eine Verbesserung und drei Fehlerbehebungen. Zwei der Lücken lassen sich ohne Anmeldung ausnutzen: ein Fehler im Passwort-Reset des Web Portals, über den sich auch Administratorkonten übernehmen lassen, und serverseitige Abrufe von URLs, die in eingehenden Nachrichten stehen. Beide betreffen Komponenten, die bei den meisten Installationen aus dem Internet erreichbar sind. Planen Sie das Update deshalb zeitnah ein.

## Von 9.2.1 direkt zu 9.5.1

Die Release Notes führen 9.5.1 unmittelbar nach 9.2.1; Versionen 9.3 und 9.4 kommen in der totemomail-Historie nicht vor. Ein Eintrag in 9.5.1 verweist trotzdem auf eine «security hardening introduced in 9.3.1». Wahrscheinlich stammt diese Nummer aus der Versionierung des Kiteworks Email Protection Gateway (EPG), das auf derselben Codebasis aufbaut und eigene Releases erhält. Für totemomail-Installationen mit 9.2.1 ist 9.5.1 das nächste Update; die Release Notes führen dazwischen keinen weiteren Patch.

## Lücken ohne Anmeldung

**Server-Side Request Forgery über den Nachrichteninhalt.** totemomail rief URLs aus verarbeiteten Nachrichten serverseitig ab, ohne das Ziel ausreichend einzuschränken. Ein Absender ohne Anmeldung konnte eine Nachricht so gestalten, dass das Gateway Anfragen an interne Dienste oder an Metadaten-Endpunkte von Cloud-Instanzen stellte und die Antworten zurückgab. Ausreichend ist dafür eine Mail, die das Gateway verarbeitet; bei einem Gateway im eingehenden Mailfluss kann das jeder externe Absender. Besonders heikel ist das bei Installationen in Azure, AWS oder anderen Clouds, wo der Metadaten-Endpunkt `169.254.169.254` Konfigurationsdaten und je nach Plattform Zugangstokens der Instanz liefert. Die Release Notes nennen für diese Lücke keine CVE-Nummer.

**Passwort-Reset im Web Portal (CVE-2026-85066).** Ein Fehler im Ablauf zum Zurücksetzen des Passworts erlaubte es, die Prüfschritte zu überspringen und für ein bestehendes Konto ein neues Passwort zu setzen. Die Release Notes schliessen Administratorkonten ausdrücklich ein. Das Web Portal ist für externe Empfänger von verschlüsselten Nachrichten gedacht und deshalb in der Regel öffentlich erreichbar.

## Lücken mit Administratorrechten

Drei weitere Korrekturen setzen eine Anmeldung mit administrativen Rechten voraus. In Kombination mit der Account-Übernahme im Web Portal verlieren diese Voraussetzungen allerdings an Gewicht.

- **Admin API, Queue-Verwaltung (CVE-2026-34843):** Über unzureichend geschützte Datei-Uploads liessen sich Dateien an beliebige Stellen im Dateisystem schreiben. Laut Release Notes war damit Remote Code Execution möglich.
- **Rule Designer:** Beim Import einer Regelkonfiguration konnte die Verarbeitungspipeline Code ausserhalb der freigegebenen Mail-Processing-Komponenten laden und ausführen. Voraussetzung sind Rechte zur Konfiguration von Mailregeln.
- **Mail API:** Beim Entpacken hochgeladener ZIP-Dateien bestand eine Path-Traversal-Lücke. Dateien werden jetzt nur noch in das temporäre Verzeichnis entpackt.

Die vierte Korrektur betrifft die Property `security.login.requestingKeysAvailable`. Bisher blendete sie im Web Portal nur den Link zum Anfordern öffentlicher Schlüssel aus; die Funktion selbst blieb über eine direkte Anfrage erreichbar. Ab 9.5.1 deaktiviert die Property die Funktion. Wenn Sie die Property gesetzt haben, um Schlüsselanfragen zu unterbinden, wirkt sie erst mit diesem Update wie erwartet.

## Aktualisierte Bibliotheken

Die mitgelieferten Bibliotheken von Apache Tomcat und Apache XML Graphics Batik wurden aktualisiert. Die Release Notes nennen dazu CVE-2026-50229, CVE-2026-53404, CVE-2026-53434, CVE-2026-55276, CVE-2026-55955, CVE-2026-55956 und CVE-2026-59083, ohne sie den beiden Bibliotheken einzeln zuzuordnen. Zusätzlich wurde woodstox-core-asl wegen einer Schwachstelle aktualisiert; dafür ist keine CVE-Nummer angegeben.

## Verbesserung und Fehlerbehebungen

- **EPG-Migration:** Bei der Migration aus einer Oracle-Datenbank unterstützt totemomail jetzt Oracle Wallet. Ist kein Datenbankpasswort konfiguriert, verbindet sich die Migration über das Wallet.
- **RMS:** Nachrichten konnten in der Verarbeitung hängen bleiben, weil RMS-Aktionen unbegrenzt auf die Authentifizierung warteten. Die Wartezeit ist jetzt auf eine Minute begrenzt. Das betrifft Installationen mit Microsoft Purview Information Protection (RMS).
- **TRE:** Die Funktion «Open Online» für Registered Envelopes funktioniert wieder. Sie war seit der erwähnten Härtung aus 9.3.1 ausgefallen.
- **Lizenzbericht:** Bei Cloud-Lizenzen zählt der Bericht interne Benutzer mit vordefinierten Feature-Typen wieder korrekt. Wie sich verwaiste Konten auf die Lizenzzählung auswirken, beschreibt der Artikel zum [Lizenzlimit und der Bereinigung per LDAP](/blog/totemomail-licensed-user-limit-ldap-cleanup).

## Installation

Der Patch lässt sich auf Systemen ab totemomail 7 b985 unter Windows, Linux, Solaris und FreeBSD installieren. Melden Sie sich an der Administrationskonsole an, laden Sie unter `[Instanzname]` > «Update» die mitgelieferte TPF-Datei hoch und starten Sie das Update. Bei mehreren Instanzen ist das für jede Instanz einzeln nötig. totemomail startet danach nach Möglichkeit selbständig neu; planen Sie dafür ein Wartungsfenster ein und erstellen Sie vorher ein Backup oder einen Snapshot.

Bis das Update installiert ist, reduzieren zwei Massnahmen das Risiko: Admin-Konsole und Admin API nur aus dem Verwaltungsnetz erreichbar machen und ausgehende Verbindungen des Gateways auf die tatsächlich benötigten Ziele beschränken. Bei Cloud-Installationen gehört der Metadaten-Endpunkt `169.254.169.254` zu den Zielen, die das Gateway nicht erreichen muss. Beide Massnahmen ersetzen den Patch nicht; die Lücke im Passwort-Reset des Web Portals schliesst nur das Update.

## Quellen

1.  totemomail Release Notes 9.5.1 (Build 101), totemo ag: Installationsanleitung und Release Notes als PDF, für Kunden über den Hersteller-Support erhältlich; Grundlage aller Angaben in diesem Artikel.

2.  [Mailrouting zwischen totemomail und Exchange Online verstehen](/blog/totemomail-m365): Wie totemomail eingehende Nachrichten verarbeitet; relevant für die Frage, welche Mails die SSRF-Lücke auslösen können.

3.  [Totemomail-Lizenzlimit erreicht: verwaiste Benutzer per LDAP bereinigen](/blog/totemomail-licensed-user-limit-ldap-cleanup): Hintergrund zur Lizenzzählung, die der Lizenzbericht-Fix betrifft.
