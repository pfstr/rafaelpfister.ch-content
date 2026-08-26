---
title: "SEPPmail Admin-GUI an Active Directory anbinden: LDAP-Authentifizierung ab 15.0.6 einrichten"
navTitle: "Admin-LDAP-Login"
description: "Seit Firmware 15.0.6 können sich Administratoren der SEPPmail-Appliance gegen einen externen LDAP-Server wie Active Directory authentifizieren, inklusive Gruppen-Mapping auf die lokale admin-Gruppe. Die Einrichtung unter User > Advanced Settings, Schritt für Schritt."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "7 Min. Lesezeit"
themen:
  - "seppmail"
produkte:
  - "seppmail"
protokolle:
  - "ldap"
  - "tls"
  - "haertung"
slug: "seppmail-admin-gui-ldap-authentifizierung"
translationId: "article-21092a3dad6b84cb"
url: "https://rafaelpfister.ch/blog/seppmail-admin-gui-ldap-authentifizierung"
draft: false
---
# SEPPmail Admin-GUI an Active Directory anbinden: LDAP-Authentifizierung ab 15.0.6 einrichten

Bis Firmware 15.0.5 kannte die Administrationsoberfläche des SEPPmail Secure E-Mail Gateway nur lokale Konten. Wer sauber arbeiten wollte, legte für jeden Administrator einen eigenen lokalen Benutzer an und nahm ihn in die Gruppe admin auf. Das funktioniert, hat aber die üblichen Nachteile lokaler Konten: eigene Passwörter pro Appliance, kein zentrales Offboarding und keine Durchsetzung der Passwort-Richtlinien aus dem Verzeichnisdienst. Mit dem Patch-Release 15.0.6 ändert sich das. Die Admin-GUI authentifiziert Administratoren auf Wunsch gegen einen externen LDAP-Server wie Active Directory und bildet AD-Gruppen auf lokale Gruppen der Appliance ab.

Die weiteren Änderungen des Releases sind im Artikel zu [SEPPmail 15.0.6 und 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1) zusammengefasst. Hier geht es nur um die neue externe Authentifizierung.

## Was die Funktion leistet

Laut den Extended Release Notes fügt 15.0.6 unter **User > Advanced Settings** einen neuen Abschnitt **External Authentication** hinzu. Damit authentifiziert sich die Admin-GUI gegen einen externen LDAP-Server, und externe Gruppen (etwa AD-Sicherheitsgruppen) werden auf lokale Gruppen der Appliance gemappt.

Extern authentifizierte Benutzer erscheinen lokal auf der Appliance und verhalten sich wie lokale Benutzer, mit einem Unterschied: Ihr Passwort lässt sich auf der Appliance nicht ändern, denn es liegt im externen LDAP-Server. Die Passwort-Hoheit wandert also vollständig ins Verzeichnis.

Wichtig zur Abgrenzung: Die Appliance kannte schon vorher eine externe Authentifizierung, allerdings nur für die GINA-Weboberfläche, konfiguriert pro Managed Domain (Abschnitt External authentication in der Domänenkonfiguration). Neu in 15.0.6 ist, dass auch der Zugang zur Administrationsoberfläche selbst über LDAP läuft.

Ob das HIN Mailgateway die LDAP-Anmeldung ebenfalls erhalten hat, teste ich noch und ergänze den Artikel anschliessend. Da die HIN-Appliances auf derselben SEPPmail-Firmware basieren, gehe ich davon aus.

## Voraussetzungen

Vor der Einrichtung sollten drei Dinge bereitstehen:

- **Firmware 15.0.6.1:** Das Feature kommt mit 15.0.6; wegen der beiden RuleEngine-Fehler des Releases ist direkt der Hotfix 15.0.6.1 die richtige Wahl.
- **Ein LDAP-fähiges Verzeichnis:** Ein Active Directory, OpenLDAP oder Vergleichbares. Liegen die Benutzer nur in Entra ID, das selbst kein LDAP spricht, schlägt [Microsoft Entra Domain Services](/blog/microsoft-entra-domain-services-ldap-kerberos) die Brücke.
- **Ein Bind-Konto im Verzeichnis:** Ein dediziertes, unprivilegiertes Servicekonto mit Lesezugriff, mit dem die Appliance die LDAP-Suche ausführt. Kein Domänen-Administrator.
- **Eine AD-Gruppe für die Gateway-Administratoren:** Zum Beispiel eine Sicherheitsgruppe SEPPmail-Admins, die später auf die lokale Gruppe admin gemappt wird. Die Mitgliedschaft in dieser Gruppe entscheidet dann über den administrativen Vollzugriff.

TLS ist in den Verbindungseinstellungen standardmässig aktiviert und sollte es bleiben; die Anmeldedaten der Administratoren gehören nicht unverschlüsselt ins Netz. Die Appliance muss den LDAP-Server auf dem konfigurierten Port erreichen (LDAPS üblicherweise 636).

## Einrichtung unter User > Advanced Settings

Die Konfiguration findet sich in der Admin-GUI unter **User > Advanced Settings** im Abschnitt **External Authentication** und besteht aus vier Blöcken.

**1. Connection Settings:** Die Checkbox *Authenticate users to external LDAP server (e.g. Active Directory)* aktiviert die Funktion. Danach folgen Server-Adresse, Port, die Option *TLS required* sowie Bind DN und Bind Password des Servicekontos.

**2. User Attributes:** Hier wird definiert, wie die Appliance Benutzerobjekte findet: die LDAP Object Class (bei Active Directory typischerweise person), die Search Base (die OU oder der Container, unter dem die Administratorkonten liegen) und das E-Mail-Attribut (Standard: mail).

**3. Group Attributes:** Analog dazu die Angaben für Gruppenobjekte, damit die Appliance die Gruppenmitgliedschaften auflösen kann.

**4. Mapping Settings:** Der entscheidende Teil. Unter *Remote Group* wird die Gruppe aus dem LDAP-Server ausgewählt, unter *Local Group* eine oder mehrere lokale Gruppen, auf die sie abgebildet wird. Für administrativen Vollzugriff ist das die Gruppe admin; deren Mitglieder sind dem Standardbenutzer admin gleichgestellt. Wer differenzieren will, mappt stattdessen auf eingeschränkte Gruppen wie readonly admin oder auf funktionsbezogene Gruppen der Appliance.

Vor dem Speichern lohnt sich der eingebaute **Login Test**: Mit Benutzername und Passwort eines Testkontos lässt sich prüfen, ob Verbindung, Suche und Authentifizierung funktionieren, bevor die Konfiguration aktiv wird.

## Beispielkonfigurationen

Die folgenden Werte sind an die eigene Umgebung anzupassen (Beispieldomäne example.com). Die Feldnamen entsprechen dem External-Authentication-Abschnitt der Appliance.

### Active Directory

| Feld | Wert |
|---|---|
| Server | dc01.example.com |
| Port | 636 |
| TLS required | aktiviert |
| Bind DN | CN=svc-seppmail,OU=ServiceAccounts,DC=example,DC=com |
| Bind Password | Passwort des Servicekontos |
| User: LDAP Object Class | person |
| User: Search Base | OU=IT,DC=example,DC=com |
| User: E-Mail Attribute | mail |
| Group: LDAP Object Class | group |
| Group: Search Base | OU=Groups,DC=example,DC=com |
| Mapping: Remote Group | SEPPmail-Admins |
| Mapping: Local Group | admin |

Hinweise zu Active Directory: Als Server eignet sich jeder erreichbare Domain Controller; in Umgebungen mit mehreren Standorten empfiehlt sich ein DC am gleichen Standort oder ein Alias, der auf mehrere DCs zeigt. Port 636 ist LDAPS; dafür muss das Zertifikat des DC von der Appliance validierbar sein. Die Search Base sollte so eng gefasst sein, dass sie die Administratorkonten enthält, aber nicht das ganze Verzeichnis. Das Attribut mail muss an den AD-Konten gepflegt sein.

### OpenLDAP

| Feld | Wert |
|---|---|
| Server | ldap01.example.com |
| Port | 636 |
| TLS required | aktiviert |
| Bind DN | cn=seppmail,ou=services,dc=example,dc=com |
| Bind Password | Passwort des Servicekontos |
| User: LDAP Object Class | inetOrgPerson |
| User: Search Base | ou=people,dc=example,dc=com |
| User: E-Mail Attribute | mail |
| Group: LDAP Object Class | groupOfNames |
| Group: Search Base | ou=groups,dc=example,dc=com |
| Mapping: Remote Group | seppmail-admins |
| Mapping: Local Group | admin |

Hinweise zu OpenLDAP: Benutzer liegen in typischen Setups als inetOrgPerson unter ou=people. Für die Gruppen ist groupOfNames die zuverlässige Wahl, da die Mitgliedschaft dort über das member-Attribut mit vollem DN abgebildet wird. posixGroup-Gruppen führen ihre Mitglieder dagegen nur als memberUid (Benutzername statt DN); ob die Appliance das auflöst, ist nicht dokumentiert und sollte vor dem Umstellen mit dem Login Test geprüft werden. Läuft der Server nur mit STARTTLS auf Port 389, gehört der entsprechende Port ins Server-Feld; unverschlüsselt sollte die Verbindung in keinem Fall laufen.

## Betriebshinweise

Drei Punkte verdienen Beachtung, bevor die LDAP-Anmeldung zum einzigen Weg in die Appliance wird:

- **Lokalen Notfallzugang behalten.** Die Passwörter externer Benutzer liegen im LDAP-Server. Ist das Verzeichnis nicht erreichbar (Netzwerkproblem, AD-Wartung, oder das Gateway soll gerade ein Problem mit ebendiesem Netzwerk beheben), braucht es weiterhin ein lokales Administratorkonto mit sicher hinterlegtem Passwort. Der Standardbenutzer admin sollte also nicht abgeschafft, sondern als dokumentierter Notfallzugang gepflegt werden.
- **MFA bleibt relevant.** 15.0.6 hat auch die MFA-Anmeldung überarbeitet: Der zweite Faktor wird nicht mehr ans Passwort angehängt, sondern in einem eigenen Feld abgefragt. Externe Authentifizierung ersetzt den zweiten Faktor nicht.
- **Offboarding über das Verzeichnis.** Der eigentliche Gewinn der Anbindung: Verlässt ein Administrator das Unternehmen, genügt das Deaktivieren des AD-Kontos beziehungsweise das Entfernen aus der gemappten Gruppe. Das bisher nötige Nachpflegen lokaler Konten auf jeder Appliance entfällt. Die lokal sichtbaren, extern authentifizierten Benutzerobjekte sollten dennoch periodisch mit dem Verzeichnis abgeglichen werden.

## Fazit

Die LDAP-Authentifizierung für die Admin-GUI schliesst eine Lücke, die bei der Appliance lange bestand: Administratorzugänge lassen sich nun zentral im Verzeichnis steuern statt pro Gerät. Zusammen mit dem separaten MFA-Feld verbessert 15.0.6 die Anmeldung an der Administrationsoberfläche damit in einem einzigen Release deutlich. Wer die Funktion einführt, sollte das Gruppen-Mapping bewusst restriktiv halten und den lokalen Notfallzugang beibehalten.

## Quellen

1.  [SEPPmail Extended Release Notes 15.0](https://downloads.seppmail.com/extrelnotes/150/ERN15.0.html): Eintrag zur Admin-GUI-Authentifizierung mit Funktionsbeschreibung, Konfigurationsort und dem Verhalten extern authentifizierter Benutzer.

2.  [SEPPmail-Dokumentation – «User > Advanced Settings»](https://docs.seppmail.com/ch/07_mi_15_usr_04_advanced-settings.html): Referenz der Felder des External-Authentication-Abschnitts (Connection, User Attributes, Group Attributes, Mapping, Login Test).

3.  [SEPPmail-Dokumentation – «Groups»](https://docs.seppmail.com/ch/07_mi_16_groups.html): Vordefinierte Gruppen der Appliance; Mitglieder der Gruppe admin haben uneingeschränkten administrativen Zugang.

4.  [SEPPmail-Dokumentation – «Revision History»](https://docs.seppmail.com/ch/20_revision-history.html): Offizielle Release Notes zu 15.0.6 mit dem Eintrag zur Admin-GUI-Authentifizierung gegen externe LDAP-Server.

5.  [SEPPmail 15.0.6 und 15.0.6.1: Sicherheitskorrekturen und neue Admin-Funktionen](/blog/seppmail-releases-15-0-6-und-15-0-6-1): Überblick über alle Änderungen der beiden Releases.
