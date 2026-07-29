---
title: "SEPPmail 15.0.6 und 15.0.6.1: Sicherheitskorrekturen und neue Admin-Funktionen"
navTitle: "SEPPmail 15.0.6"
description: "SEPPmail hat im Juli 2026 das Patch-Release 15.0.6 und den Hotfix 15.0.6.1 veröffentlicht. Neben behobenen Schwachstellen in PDF-Generierung und PGP-Verarbeitung bringen die Releases ein separates MFA-Feld, LDAP-Authentifizierung für die Admin-GUI und Korrekturen an RuleEngine, Webmail und REST-API."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "5 Min. Lesezeit"
themen:
  - "seppmail"
slug: "seppmail-releases-15-0-6-und-15-0-6-1"
url: "https://rafaelpfister.ch/blog/seppmail-releases-15-0-6-und-15-0-6-1"
draft: false
---
# SEPPmail 15.0.6 und 15.0.6.1: Sicherheitskorrekturen und neue Admin-Funktionen

SEPPmail hat am 21. Juli 2026 das Patch-Release 15.0.6 und einen Tag später den Hotfix 15.0.6.1 veröffentlicht. Das Patch-Release schliesst mehrere Schwachstellen, aktualisiert OpenSSH und OpenSSL und bringt spürbare Verbesserungen für die Administration. Der Hotfix korrigiert zwei Fehler in der RuleEngine, die mit 15.0.6 eingeführt oder sichtbar wurden. Die Änderungen betreffen auch Appliances, die als HIN Mailgateway betrieben werden, da diese auf derselben SEPPmail-Firmware basieren.

## Hotfix 15.0.6.1 vom 22. Juli 2026

Der Hotfix behebt zwei Punkte in der RuleEngine. Erstens verhinderte ein undefinierter Wert im Message-Objekt, dass Logeinträge ins Mail-Log geschrieben wurden — betroffene Nachrichten liefen damit ohne Protokollierung durch das System. Zweitens erkennt die RuleEngine nun die Richtung archivierter E-Mails, damit deren Zustellung korrekt behandelt wird.

Wer 15.0.6 bereits installiert hat oder das Update plant, sollte direkt auf 15.0.6.1 gehen.

## Sicherheitskorrekturen in 15.0.6

Der wichtigste Teil des Patch-Releases sind drei Korrekturen an der Sicherheitsarchitektur:

- Eine mögliche Path-Traversal-Schwachstelle in der PDF-Generierung wurde geschlossen. Gefunden wurde sie von InfoGuard.
- Sämtlicher per PGP entschlüsselter Inhalt wird nun Base64-codiert, um MIME-Structure-Injection zu verhindern.
- Die Funktion hashencrypt wurde auf AES-256-CBC mit PBKDF2 umgestellt.

Dazu kommen aktualisierte Bibliotheken: OpenSSH 10.4 und OpenSSL 3.0.21 beheben zusammen über zwanzig CVEs. Allein wegen dieser Punkte ist das Update für produktive Systeme empfehlenswert.

## Neue Funktionen für die Administration

Drei Änderungen in der Admin-GUI fallen im Alltag auf:

- **Separates MFA-Eingabefeld:** Der zweite Faktor muss nicht mehr an das Passwort angehängt werden, sondern hat ein eigenes Feld. Das beseitigt eine langjährige Stolperfalle beim Login.
- **LDAP-Authentifizierung für die Admin-GUI:** Administratoren können sich nun gegen einen externen LDAP-Server authentifizieren, statt lokale Konten auf der Appliance zu pflegen.
- **AutoRenew-Button für MPKI:** In den MPKI-Connector-Einstellungen lässt sich die automatische Zertifikatserneuerung per «Trigger AutoRenew...» manuell anstossen.

Daneben verwendet die Appliance jetzt konsequent gültige Zeitzonen (Standard: Europe/Zurich), und die System Object ID unter System >> Advanced View wird als gültige OID validiert.

## Mailverarbeitung und Webmail

In der RuleEngine wurden vier Punkte korrigiert. Die Betreffbehandlung funktioniert nun auch bei unbekanntem Encoding. Nachrichten werden gebounct, wenn eine Signatur explizit angefordert, aber nicht erstellt werden kann — bisher konnten solche Nachrichten unsigniert weiterlaufen. Archivkopien laufen neu durch die Zustellfunktion und erhalten damit ARC-Header. Und bei PGP-Nachrichten ohne MDC-Daten werden MDC-Fehler ignoriert, statt die Verarbeitung zu stören.

Im Webmail (GINA) wurden vier Fehler behoben: Die automatische Löschung nicht registrierter Konten nach Ablauf der Karenzfrist funktioniert wieder, die Funktion hashdecrypt lieferte in bestimmten Fällen ein falsch-positives Entschlüsselungsergebnis, das Hinzufügen eines Anhangs leerte die Felder An und CC, und die Zeitausgabe in den SMS-Logs war fehlerhaft.

## REST-API, Cluster und Backup

Die REST-API erhält Korrekturen an mehreren Endpunkten: /system/ifaliasconfig (Umgang mit null-Werten), /system/applySysconfig (Zugriffskonfiguration), /crypto/domain/{domainName} (Upload von Domänenzertifikaten) sowie GET und POST /ssl/csr. Der Timeout für REST-Aufrufe wurde von 300 auf 900 Sekunden erhöht, was langlaufende Anfragen wie grössere Konfigurationsänderungen zuverlässiger macht.

Im Cluster-Betrieb blockierte bisher eine bestehende CARP-IP die IP-Einstellungen eines neu aufgenommenen Mitglieds; das ist behoben. Ausserdem wird das Passwort-Rehashing unterdrückt, wenn Cluster-Mitglieder unterschiedliche Firmware-Versionen fahren — dazu gleich mehr. Vor der täglichen Snapshot-Erstellung prüft das Backup neu zusätzlich auf eine korrupte Datenbank, bevor der Snapshot geschrieben wird.

## Bezug zum Login-Ausfall unter 15.0.5

Beim Update eines Clusters auf 15.0.5 konnte die Anmeldung auf beiden Knoten ausfallen — das Fehlerbild und die Wiederherstellung sind im Artikel zum [Login-Ausfall nach dem 15.0.5-Update](/blog/hin-update-issue-version-15.0.5) beschrieben. Der Hersteller kannte das Problem damals bereits und stellte eine Korrektur für eine folgende Version in Aussicht.

In den Release Notes zu 15.0.6 findet sich nun genau ein Eintrag, der zu diesem Fehlerbild passt: «prevent password rehashing when cluster members use different firmware versions». Während eines Cluster-Updates laufen die Knoten zwangsläufig vorübergehend mit unterschiedlichen Firmware-Ständen. Berechnet ein Knoten in dieser Phase Passwort-Hashes neu und repliziert sie in den Cluster, passen die Hashes auf dem anderen Stand nicht mehr — die Anmeldung scheitert auf beiden Knoten, genau wie beim damals beobachteten Ausfall. Die Release Notes nennen den Login-Ausfall nicht ausdrücklich, der Eintrag deckt aber exakt die Konstellation ab, die ihn ausgelöst hatte. Die Ursache ist damit in 15.0.6 adressiert; die in 15.0.5 nötige Notfallprozedur mit Cluster-Auflösung sollte sich bei künftigen Updates erübrigen.

## Kleinere Korrekturen

Im Mail-Log wurde die Datumssortierung korrigiert, die bisher alphabetisch statt chronologisch sortierte, und die angezeigte Grösse von LFT-Nachrichten stimmt wieder. Zugriffe auf nicht vorhandene X-Header werden nicht mehr protokolliert. Der CertCentral-Connector der MPKI geht robuster mit Eingabe- und REST-Fehlern um.

## Einordnung

15.0.6 ist ein Patch-Release mit ungewöhnlich viel Substanz: drei Schwachstellen-Fixes, zwei aktualisierte Kryptobibliotheken und mit dem separaten MFA-Feld sowie der LDAP-Authentifizierung zwei Funktionen, die den Administrationsalltag direkt verbessern. Die beiden RuleEngine-Fehler aus dem Hotfix sprechen dafür, 15.0.6 zu überspringen und direkt 15.0.6.1 einzusetzen. Für Cluster gilt wie immer: Update-Reihenfolge gemäss Herstellerdokumentation einhalten und vorher Snapshots beider Knoten erstellen — die Erfahrungen mit dem Login-Ausfall unter 15.0.5 haben gezeigt, weshalb das kein Formalismus ist.

## Quellen

1.  [SEPPmail-Dokumentation – «Revision History»](https://docs.seppmail.com/ch/20_revision-history.html) — offizielle Release Notes zu 15.0.6 und 15.0.6.1 mit allen Einzelpunkten.

2.  [HIN Mailgateway 15.0.5: Login-Ausfall nach dem Cluster-Update beheben](/blog/hin-update-issue-version-15.0.5) — weshalb Snapshots und die korrekte Update-Reihenfolge im Cluster entscheidend sind.
