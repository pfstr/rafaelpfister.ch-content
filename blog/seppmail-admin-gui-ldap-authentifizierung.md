---
title: "SEPPmail Admin-GUI an Active Directory anbinden: LDAP-Authentifizierung ab 15.0.6 einrichten"
navTitle: "Admin-LDAP-Login"
description: "Seit Firmware 15.0.6 können sich Administratoren der SEPPmail-Appliance gegen einen externen LDAP-Server wie Active Directory authentifizieren, inklusive Gruppen-Mapping auf die lokale admin-Gruppe. Die Einrichtung unter User > Advanced Settings, Schritt für Schritt."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "6 Min. Lesezeit"
themen:
  - "seppmail"
slug: "seppmail-admin-gui-ldap-authentifizierung"
url: "https://rafaelpfister.ch/blog/seppmail-admin-gui-ldap-authentifizierung"
draft: false
---
# SEPPmail Admin-GUI an Active Directory anbinden: LDAP-Authentifizierung ab 15.0.6 einrichten

Bis Firmware 15.0.5 kannte die Administrationsoberfläche des SEPPmail Secure E-Mail Gateway nur lokale Konten. Wer sauber arbeiten wollte, legte für jeden Administrator einen eigenen lokalen Benutzer an und nahm ihn in die Gruppe admin auf. Das funktioniert, hat aber die üblichen Nachteile lokaler Konten: eigene Passwörter pro Appliance, kein zentrales Offboarding und keine Durchsetzung der Passwort-Richtlinien aus dem Verzeichnisdienst. Mit dem Patch-Release 15.0.6 ändert sich das. Die Admin-GUI authentifiziert Administratoren auf Wunsch gegen einen externen LDAP-Server wie Active Directory und bildet AD-Gruppen auf lokale Gruppen der Appliance ab.

Die weiteren Änderungen des Releases sind im Artikel zu [SEPPmail 15.0.6 und 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1) zusammengefasst. Hier geht es nur um die neue externe Authentifizierung.

## Was die Funktion leistet

Laut den Extended Release Notes (Ticket 58159) fügt 15.0.6 unter **User > Advanced Settings** einen neuen Abschnitt **External Authentication** hinzu. Damit authentifiziert sich die Admin-GUI gegen einen externen LDAP-Server, und externe Gruppen (etwa AD-Sicherheitsgruppen) werden auf lokale Gruppen der Appliance gemappt.

Extern authentifizierte Benutzer erscheinen lokal auf der Appliance und verhalten sich wie lokale Benutzer, mit einem Unterschied: Ihr Passwort lässt sich auf der Appliance nicht ändern, denn es liegt im externen LDAP-Server. Die Passwort-Hoheit wandert also vollständig ins Verzeichnis.

Wichtig zur Abgrenzung: Die Appliance kannte schon vorher eine externe Authentifizierung, allerdings nur für die GINA-Weboberfläche, konfiguriert pro Managed Domain (Abschnitt External authentication in der Domänenkonfiguration). Neu in 15.0.6 ist, dass auch der Zugang zur Administrationsoberfläche selbst über LDAP läuft.

## Voraussetzungen

Vor der Einrichtung sollten drei Dinge bereitstehen:

- **Firmware 15.0.6.1:** Das Feature kommt mit 15.0.6; wegen der beiden RuleEngine-Fehler des Releases ist direkt der Hotfix 15.0.6.1 die richtige Wahl.
- **Ein Bind-Konto im Verzeichnis:** Ein dediziertes, unprivilegiertes Servicekonto mit Lesezugriff, mit dem die Appliance die LDAP-Suche ausführt. Kein Domänen-Administrator.
- **Eine AD-Gruppe für die Gateway-Administratoren:** Zum Beispiel eine Sicherheitsgruppe SEPPmail-Admins, die später auf die lokale Gruppe admin gemappt wird. Die Mitgliedschaft in dieser Gruppe entscheidet dann über den administrativen Vollzugriff.

TLS ist in den Verbindungseinstellungen standardmässig aktiviert und sollte es bleiben; die Anmeldedaten der Administratoren gehören nicht unverschlüsselt ins Netz. Die Appliance muss den LDAP-Server auf dem konfigurierten Port erreichen (LDAPS üblicherweise 636).

## Einrichtung unter User > Advanced Settings

Die Konfiguration findet sich in der Admin-GUI unter **User > Advanced Settings** im Abschnitt **External Authentication** und besteht aus vier Blöcken.

**1. Connection Settings:** Die Checkbox *Authenticate users to external LDAP server (e.g. Active Directory)* aktiviert die Funktion. Danach folgen Server-Adresse, Port, die Option *TLS required* sowie Bind DN und Bind Password des Servicekontos.

**2. User Attributes:** Hier wird definiert, wie die Appliance Benutzerobjekte findet: die LDAP Object Class (bei Active Directory typischerweise person), die Search Base (die OU oder der Container, unter dem die Administratorkonten liegen) und das E-Mail-Attribut (Standard: mail).

**3. Group Attributes:** Analog dazu die Angaben für Gruppenobjekte, damit die Appliance die Gruppenmitgliedschaften auflösen kann.

**4. Mapping Settings:** Der entscheidende Teil. Unter *Remote Group* wird die Gruppe aus dem LDAP-Server ausgewählt, unter *Local Group* eine oder mehrere lokale Gruppen, auf die sie abgebildet wird. Für administrativen Vollzugriff ist das die Gruppe admin; deren Mitglieder sind dem Standardbenutzer admin gleichgestellt. Wer differenzieren will, mappt stattdessen auf eingeschränkte Gruppen wie readonly admin oder auf funktionsbezogene Gruppen der Appliance.

Vor dem Speichern lohnt sich der eingebaute **Login Test**: Mit Benutzername und Passwort eines Testkontos lässt sich prüfen, ob Verbindung, Suche und Authentifizierung funktionieren, bevor die Konfiguration aktiv wird. Anschliessend sichert *Save* die Einstellungen.

## Betriebshinweise

Drei Punkte verdienen Beachtung, bevor die LDAP-Anmeldung zum einzigen Weg in die Appliance wird:

- **Lokalen Notfallzugang behalten.** Die Passwörter externer Benutzer liegen im LDAP-Server. Ist das Verzeichnis nicht erreichbar (Netzwerkproblem, AD-Wartung, oder das Gateway soll gerade ein Problem mit ebendiesem Netzwerk beheben), braucht es weiterhin ein lokales Administratorkonto mit sicher hinterlegtem Passwort. Der Standardbenutzer admin sollte also nicht abgeschafft, sondern als dokumentierter Notfallzugang gepflegt werden.
- **MFA bleibt relevant.** 15.0.6 hat auch die MFA-Anmeldung überarbeitet: Der zweite Faktor wird nicht mehr ans Passwort angehängt, sondern in einem eigenen Feld abgefragt. Externe Authentifizierung ersetzt den zweiten Faktor nicht.
- **Offboarding über das Verzeichnis.** Der eigentliche Gewinn der Anbindung: Verlässt ein Administrator das Unternehmen, genügt das Deaktivieren des AD-Kontos beziehungsweise das Entfernen aus der gemappten Gruppe. Das bisher nötige Nachpflegen lokaler Konten auf jeder Appliance entfällt. Die lokal sichtbaren, extern authentifizierten Benutzerobjekte sollten dennoch periodisch mit dem Verzeichnis abgeglichen werden.

Die Handbuchseite zu den Advanced Settings dokumentiert derzeit weder das erwartete Login-Format (E-Mail-Adresse, UPN oder sAMAccountName) noch ein explizites Fallback-Verhalten für lokale Konten. Beides lässt sich mit dem Login Test und einem zweiten Browserfenster gefahrlos verifizieren: die neue Konfiguration testen, während die bestehende Admin-Session offen bleibt.

## Fazit

Die LDAP-Authentifizierung für die Admin-GUI schliesst eine Lücke, die bei der Appliance lange bestand: Administratorzugänge lassen sich nun zentral im Verzeichnis steuern statt pro Gerät. Zusammen mit dem separaten MFA-Feld macht 15.0.6 die Anmeldung an der Administrationsoberfläche damit in einem einzigen Release deutlich erwachsener. Wer die Funktion einführt, sollte das Gruppen-Mapping bewusst restriktiv halten und den lokalen Notfallzugang nicht opfern.

## Quellen

1.  [SEPPmail Extended Release Notes 15.0](https://downloads.seppmail.com/extrelnotes/150/ERN15.0.html): Eintrag zu Ticket 58159 mit Funktionsbeschreibung, Konfigurationsort und dem Verhalten extern authentifizierter Benutzer.

2.  [SEPPmail-Dokumentation – «User > Advanced Settings»](https://docs.seppmail.com/ch/07_mi_15_usr_04_advanced-settings.html): Referenz der Felder des External-Authentication-Abschnitts (Connection, User Attributes, Group Attributes, Mapping, Login Test).

3.  [SEPPmail-Dokumentation – «Groups»](https://docs.seppmail.com/ch/07_mi_16_groups.html): Vordefinierte Gruppen der Appliance; Mitglieder der Gruppe admin haben uneingeschränkten administrativen Zugang.

4.  [SEPPmail-Dokumentation – «Revision History»](https://docs.seppmail.com/ch/20_revision-history.html): Offizielle Release Notes zu 15.0.6 mit dem Eintrag zur Admin-GUI-Authentifizierung gegen externe LDAP-Server.

5.  [SEPPmail 15.0.6 und 15.0.6.1: Sicherheitskorrekturen und neue Admin-Funktionen](/blog/seppmail-releases-15-0-6-und-15-0-6-1): Überblick über alle Änderungen der beiden Releases.
