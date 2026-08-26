---
title: "E-Mail-Security-Gateway evaluieren: Kriterienkatalog und Pflichtenheft-Vorlage"
navTitle: "Gateway-Evaluation"
description: "Ob Ablösung, Konsolidierung oder erzwungene Migration: Wer ein E-Mail-Security-Gateway beschafft, braucht messbare Kriterien statt Produktbroschüren. Der Artikel liefert den Kriterienkatalog in sechs Blöcken und ein kopierfertiges Pflichtenheft mit Bewertungsmatrix, ausgerichtet auf Schweizer Anforderungen."
date: "2026-08-03"
kategorie: "E-Mail-Verschlüsselung"
timeToRead: "9 Min. Lesezeit"
themen:
  - "e-mail-verschluesselung"
  - "seppmail"
  - "totemomail"
produkte:
  - "uebergreifend"
protokolle:
  - "verschluesselung"
hauptthema: "e-mail-verschluesselung"
related:
  - "massenmailing-provider-wechsel-checkliste"
  - "dmarc-einfuehrung-business-case"
slug: "e-mail-security-gateway-evaluieren"
translationId: "article-f1bce23ac6985202"
url: "https://rafaelpfister.ch/blog/e-mail-security-gateway-evaluieren"
zielgruppe: "entscheider"
draft: true
aiPrompt: |
  Du bist mein Assistent für die Evaluation eines E-Mail-Security-Gateways. Erstelle auf Basis der Pflichtenheft-Vorlage aus diesem Artikel eine auf unser Unternehmen zugeschnittene Version. Frage mich zuerst: 1. Branche und regulatorische Anforderungen (Schweiz: revDSG, Berufsgeheimnis, Bankkundengeheimnis; EU-Bezug ja/nein), 2. Anzahl Benutzer und erwartetes Mailvolumen, 3. Mailumgebung (Exchange OnPrem, Hybrid, Exchange Online, andere), 4. benötigte Verschlüsselungsarten (S/MIME, PGP, Domainverschlüsselung, Web-Portal für Ad-hoc-Empfänger), 5. Betriebsmodell (eigene Appliance, Hosting in der Schweiz, SaaS). Passe danach die MUSS- und SOLL-Anforderungen an, schlage eine Gewichtung für die Bewertungsmatrix vor und formuliere die Vorlage so, dass sie direkt als Beilage einer Offertanfrage taugt.
---

Der Markt für E-Mail-Verschlüsselungsgateways ist in Bewegung: Produkte wechseln den Besitzer, Plattformen werden abgelöst, Hersteller richten ihre Roadmaps neu aus. Früher oder später fällt deshalb bei vielen Unternehmen dieselbe Aufgabe an: das bestehende Gateway ersetzen oder ein neues beschaffen, oft unter Zeitdruck und mit laufendem Betrieb im Rücken.

Die typische Abkürzung, die Auswahl der Produktbroschüre des lautesten Anbieters zu überlassen, rächt sich in diesem Segment besonders. Ein E-Mail-Security-Gateway sitzt mitten im Mailfluss, berührt Schlüsselmaterial, Verzeichnisdienste und regulierte Daten. Wer hier ohne messbare Kriterien entscheidet, geht eine Wette ein. Dieser Artikel liefert den Kriterienkatalog in sechs Blöcken und ein Pflichtenheft, das Sie kopieren, anpassen und Ihrer Offertanfrage beilegen können.

**Schweiz-Fokus:** Die rechtlichen Aussagen in diesem Artikel beziehen sich auf Schweizer Recht, insbesondere das revidierte Datenschutzgesetz (revDSG) und die branchenspezifischen Geheimnispflichten. Wo die Rechtslage in der EU abweicht, ist der Absatz ausdrücklich als EU-Hinweis gekennzeichnet.

## Die sechs Kriterienblöcke

### 1. Verschlüsselungsfunktionen

Der Kern der Beschaffung: Welche Verfahren braucht Ihr Unternehmen wirklich? S/MIME und PGP für den strukturierten Austausch mit Partnern, erzwungenes TLS oder Domainverschlüsselung für feste Beziehungen, und ein Web-Portal- oder Passwort-Verfahren für Empfänger ohne eigene Verschlüsselung. Prüfen Sie die Verfahren gegen Ihre realen Kommunikationsbeziehungen statt gegen die Feature-Liste: Ein Gateway, das die fünf wichtigsten Gegenstellen Ihrer Branche nicht sauber bedient, fällt durch, egal wie lang seine Broschüre ist.

### 2. Architektur und Mailfluss-Integration

Das Gateway muss zu Ihrer Mailarchitektur passen, nicht umgekehrt: Exchange OnPrem, Hybrid oder Exchange Online, vorgeschaltete Hygiene-Lösungen, Load Balancer, mehrere Standorte. Klären Sie Clusterfähigkeit und Ausfallverhalten, das Verhalten bei Überlast und die Frage, wie sich das Gateway in bestehende Mailrouten einfügt. Verlangen Sie ein Architekturdiagramm für Ihr konkretes Szenario als Teil der Offerte.

### 3. Verzeichnis- und Schlüsselverwaltung

Im Betrieb entscheidet die Anbindung an Active Directory, LDAP oder Entra ID über den Pflegeaufwand: automatisches Anlegen und Deaktivieren von Benutzern, Gruppenregeln, Lizenzverbrauch. Bei der Schlüsselverwaltung zählen Zertifikatsbezug und Erneuerung, die Trennung von Signatur- und Verschlüsselungsschlüsseln, Export und Backup des Schlüsselmaterials sowie die Frage, ob ein HSM unterstützt wird. Ein Gateway, aus dem Sie Ihr Schlüsselmaterial nicht sauber exportieren können, ist ein Lock-in mit Ablaufdatum.

### 4. Betrieb, Monitoring und Wartung

Updates ohne Unterbruch des Mailflusses, nachvollziehbare Release-Notes, Backup und Restore der Konfiguration, Monitoring-Schnittstellen und aussagekräftige Protokolle: Das sind die Kriterien, die nach dem Kauf über die Betriebskosten entscheiden. Fragen Sie explizit nach dem Ablauf eines Restores auf leerer Hardware und nach der durchschnittlichen Dauer eines Versionssprungs im Cluster.

### 5. Compliance und Datenstandort

Beim Gateway laufen Personendaten und oft besonders schützenswerte Daten durch: Nach revDSG braucht es bei externem Betrieb einen Vertrag zur Auftragsbearbeitung mit dokumentierten technischen und organisatorischen Massnahmen. In regulierten Branchen kommen Geheimnispflichten dazu, etwa das Berufsgeheimnis nach Art. 321 StGB im Gesundheitswesen oder das Bankkundengeheimnis; dort gehören Datenstandort Schweiz, Zugriffskonzepte des Herstellers und Supportzugriffe vertraglich geregelt. Klären Sie für jedes Betriebsmodell (Appliance, Hosting, SaaS), wo Nachrichten, Schlüssel und Protokolle liegen und wer darauf zugreifen kann.

**EU-Hinweis:** Für Unternehmen mit EU-Bezug gelten zusätzlich die DSGVO-Anforderungen an Auftragsverarbeitung (Art. 28) und Drittlandtransfers. Ein Datenstandort in der Schweiz ist aus EU-Sicht ein Drittland mit Angemessenheitsbeschluss; in der Praxis unkritisch, aber im Auftragsverarbeitungsvertrag sauber abzubilden.

### 6. Hersteller und Zukunftssicherheit

Gerade weil sich der Markt konsolidiert, gehört der Hersteller selbst auf den Prüfstand: Roadmap und Investitionen in das Produkt, Support-Organisation und SLA, Referenzen in Ihrer Branche und Grössenklasse, Preismodell inklusive Wachstum, und die Exit-Frage: Wie kommen Sie mit Schlüsseln, Konfiguration und Archivdaten wieder heraus? Lassen Sie sich Zusagen zur Produktpflege schriftlich geben; eine mündliche Roadmap-Aussage am Messestand ist keine Investitionsgrundlage.

## Das kopierfertige Pflichtenheft

Die Vorlage nutzt MUSS für Ausschlusskriterien und SOLL für gewichtete Kriterien. Streichen Sie, was nicht zutrifft, ergänzen Sie Ihre Mengengerüste und legen Sie die Vorlage der Offertanfrage bei. Die Bewertungsmatrix am Ende macht die Offerten vergleichbar und die Entscheidung dokumentierbar; das zahlt sich spätestens bei internen Audits oder Rückfragen des Verwaltungsrats aus.

```text
Pflichtenheft E-Mail-Security-Gateway
Version: [1.0], Datum: [TT.MM.JJJJ], Kontakt: [Name, Funktion]

0. Ausgangslage und Mengengerüst
   - Benutzer: [Anzahl], davon verschlüsselungspflichtig: [Anzahl]
   - Mailvolumen: [ø/Tag und Spitze/Stunde]
   - Mailumgebung: [Exchange OnPrem/Hybrid/Online, weitere Systeme]
   - Wichtigste Gegenstellen: [Partner, Behörden, Branchennetze]
   - Zielbetriebsmodell: [Appliance/Hosting CH/SaaS]

A. Verschlüsselung
   MUSS  S/MIME-Verschlüsselung und -Signatur (Senden und Empfangen)
   MUSS  Erzwungenes TLS pro Domain konfigurierbar
   MUSS  Verfahren für Empfänger ohne Verschlüsselung
         (Web-Portal oder passwortgeschützte Zustellung)
   SOLL  OpenPGP (Senden und Empfangen)
   SOLL  Domainverschlüsselung mit festen Partnern
   SOLL  Automatische Zertifikats- und Schlüsselbeschaffung
   SOLL  Regelwerk: Verschlüsselungspflicht pro Absender, Empfänger,
         Gruppe oder Inhalt (Betreff-Kennwort, Header)

B. Architektur und Integration
   MUSS  Betrieb im bestehenden Mailfluss: [kurz beschreiben]
   MUSS  Clusterfähigkeit ohne Single Point of Failure
   MUSS  Unterstützung von [Exchange OnPrem/Hybrid/Exchange Online]
   SOLL  Lastverteilung über bestehende Load Balancer möglich
   SOLL  Mandantenfähigkeit für [Tochtergesellschaften/Marken]
   SOLL  Referenzarchitektur für unser Szenario als Teil der Offerte

C. Verzeichnis- und Schlüsselverwaltung
   MUSS  Anbindung an [Active Directory/LDAP/Entra ID] mit
         automatischer Benutzerprovisionierung und -deaktivierung
   MUSS  Export des gesamten Schlüsselmaterials in offenen Formaten
   MUSS  Rollenbasierte Administration mit MFA
   SOLL  HSM-Unterstützung
   SOLL  Automatische Bereinigung verwaister Konten (Lizenzhygiene)
   SOLL  Revisionssichere Protokollierung aller Admin-Aktionen

D. Betrieb und Wartung
   MUSS  Updates ohne Unterbruch des Mailflusses im Cluster
   MUSS  Konfigurations-Backup und dokumentierter Restore
   MUSS  Monitoring-Schnittstelle [SNMP/REST/Syslog]
   SOLL  Test-/Staging-Instanz im Lizenzumfang
   SOLL  Durchschnittliche Update-Dauer und -Frequenz offenlegen

E. Compliance (Schweiz)
   MUSS  Vertrag zur Auftragsbearbeitung nach Art. 9 revDSG mit
         dokumentierten TOM (bei Hosting/SaaS)
   MUSS  Angabe aller Bearbeitungsorte und Unterauftragnehmer
   MUSS  Verbindliche Meldefristen bei Sicherheitsvorfällen
         (Unterstützung der Meldepflicht nach Art. 24 revDSG)
   SOLL  Datenstandort Schweiz für Nachrichten, Schlüssel, Protokolle
   SOLL  Nachweis Berufsgeheimnis-Eignung (Art. 321 StGB) bzw.
         branchenspezifischer Anforderungen: [Branche eintragen]
   SOLL  [Nur bei EU-Bezug:] Auftragsverarbeitungsvertrag nach
         Art. 28 DSGVO inkl. Regelung der Drittlandtransfers
   SOLL  Unabhängige Nachweise (ISO 27001, SOC 2) aktuell vorhanden

F. Hersteller und Kommerzielles
   MUSS  Support mit definierten Reaktionszeiten, Eskalationsweg
         und deutschsprachiger Anlaufstelle
   MUSS  Preismodell über 5 Jahre inkl. Wachstum [+X % Benutzer]
   MUSS  Exit-Unterstützung: Export von Schlüsseln, Konfiguration
         und Protokollen bei Vertragsende
   SOLL  Schriftliche Roadmap-Zusagen für [Produktlinie]
   SOLL  Referenzen aus Branche und Grössenklasse

Bewertungsmatrix (Vorschlag)
   Block A Verschlüsselung        Gewicht 25 %   Punkte 1-5
   Block B Architektur            Gewicht 15 %   Punkte 1-5
   Block C Verzeichnis/Schlüssel  Gewicht 15 %   Punkte 1-5
   Block D Betrieb                Gewicht 15 %   Punkte 1-5
   Block E Compliance             Gewicht 15 %   Punkte 1-5
   Block F Hersteller/Preis       Gewicht 15 %   Punkte 1-5
   Nicht erfüllte MUSS-Kriterien führen zum Ausschluss.
   Bewertung je Block: gewichteter Durchschnitt der SOLL-Kriterien.
```

## Vom Pflichtenheft zur Entscheidung

Drei Hinweise aus der Praxis: Erstens, testen Sie die zwei bestplatzierten Produkte mit einem Proof of Concept gegen Ihre realen Gegenstellen, bevor Sie unterschreiben; kein Papier ersetzt den Versuch mit echten Zertifikatsketten und echten Empfängern. Zweitens, beziehen Sie den Betrieb früh ein: Die Personen, die das Gateway nachher warten, erkennen Betriebsrisiken schneller als jede Projektgruppe. Drittens, verhandeln Sie die Exit-Klauseln vor der Unterschrift; nach der Migration ist Ihre Verhandlungsposition weg.

Wie Sie bei einer erzwungenen Migration die Fragen an den bisherigen oder neuen Betreiber stellen, zeigt die [Checkliste zum Provider-Wechsel beim Massenmailing](https://rafaelpfister.ch/blog/massenmailing-provider-wechsel-checkliste). Weitere Vorlagen für Entscheider finden Sie auf der [Entscheider-Seite](https://rafaelpfister.ch/cio).

## Quellen

1.  [EDÖB: Outsourcing und Auftragsdatenbearbeitung](https://www.edoeb.admin.ch/de/outsourcing-auftragsdatenbearbeitung): Verantwortung des Auftraggebers und Anforderungen an die vertragliche Absicherung bei ausgelagerter Datenbearbeitung

2.  [EDÖB: Leitfaden zu den technischen und organisatorischen Massnahmen](https://www.edoeb.admin.ch/dam/de/sd-web/eVhrh8wY3QcR/leitfaden_tom.pdf): Referenz für die im Pflichtenheft geforderten TOM-Nachweise

3.  [Bundesgesetz über den Datenschutz (DSG) vom 25. September 2020](https://www.fedlex.admin.ch/eli/cc/2022/491/de): Grundlage für die Compliance-Anforderungen in Block E, insbesondere Art. 9 (Auftragsbearbeitung) und Art. 24 (Meldepflicht)

4.  [Schweizerisches Strafgesetzbuch, Art. 321](https://www.fedlex.admin.ch/eli/cc/54/757_781_799/de): Berufsgeheimnis als massgebliche Anforderung an Gateways im Gesundheitswesen und bei weiteren Geheimnisträgern

5.  [Verordnung (EU) 2016/679 (DSGVO)](https://eur-lex.europa.eu/eli/reg/2016/679/oj): Für den gekennzeichneten EU-Hinweis relevant sind Art. 28 (Auftragsverarbeitung) und Kapitel V (Drittlandtransfers)
