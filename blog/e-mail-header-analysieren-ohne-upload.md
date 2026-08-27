---
title: "E-Mail-Header analysieren, ohne die Mail hochzuladen: lokal im Browser statt im Web-Tool"
navTitle: "Header lokal analysieren"
description: "E-Mail-Header enthalten interne Hostnamen, IP-Adressen und Personendaten. Wer sie in ein Online-Tool einfügt, übermittelt diese Informationen an einen fremden Server. Warum die Analyse keinen Server braucht und was ein lokal im Browser laufendes Tool leisten kann."
date: "2026-08-26"
kategorie: "SMTP & Mailflow"
timeToRead: "7 Min. Lesezeit"
themen:
  - "smtp-mailflow"
hauptthema: "smtp-mailflow"
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "mail-auth"
  - "troubleshooting"
related:
  - "microsoft-365-compauth-reason-codes"
  - "exchange-hybrid-header-intern-extern"
  - "typische-ursachen-fuer-mail-loops-und-deren-behebung"
slug: "e-mail-header-analysieren-ohne-upload"
translationId: "article-cad792e705cee24e"
url: "https://rafaelpfister.ch/blog/e-mail-header-analysieren-ohne-upload"
---

# E-Mail-Header analysieren, ohne die Mail hochzuladen: lokal im Browser statt im Web-Tool

Der übliche Weg, einen E-Mail-Header zu analysieren, sieht so aus: Header aus dem Mailclient kopieren, in ein Online-Tool einfügen, auswerten lassen. Das ist praktisch, aber dabei geht der komplette Header an den Server des Tool-Betreibers. Was genau man damit übermittelt, ist den wenigsten bewusst.

## Was in einem Header wirklich steht

Ein vollständiger Header einer Mail aus einer Unternehmensumgebung enthält typischerweise:

- **Interne Hostnamen und IP-Adressen:** Jede `Received`-Zeile dokumentiert einen Server auf dem Zustellweg, inklusive der internen Exchange-Server, Gateways und Loadbalancer mit FQDN und oft privater IP-Adresse. Zusammen ergibt das eine Skizze der Mail-Infrastruktur.
- **Personendaten:** Absender- und Empfängeradressen, Anzeigenamen, der Betreff, Message-IDs und je nach Client die IP-Adresse des ursprünglichen Absenders.
- **Software und Versionen:** Received-Zeilen und produktspezifische Header nennen die eingesetzten Produkte, teils mit Versionsständen.
- **Organisationsinterne Bewertung:** Bei Microsoft 365 etwa die komplette Spam- und Authentifizierungs-Bewertung, Tenant-Kennungen und die interne Einstufung der Nachricht.

Für einen Angreifer ist das brauchbares Material zur Vorbereitung, für den Datenschutz sind es Personendaten: Absender, Empfänger und Betreff einer konkreten Nachricht. Nach dem revidierten Datenschutzgesetz bleibt die Bearbeitung durch ein ausländisches Online-Tool eine Bekanntgabe an einen Dritten, im Zweifel ins Ausland. Bei einem Header aus einem Support-Fall eines Kunden verschärft sich die Frage: Dessen Daten in ein fremdes Web-Tool einzufügen, ist ohne Rechtsgrundlage oder Einwilligung kaum zu begründen.

## Die Analyse braucht keinen Server

Der entscheidende Punkt: Ein Header ist reiner Text, und seine Auswertung ist reines Parsen. Received-Kette chronologisch ordnen, Zeitstempel differenzieren, `Authentication-Results` dekodieren, Domains vergleichen: Nichts davon erfordert eine Server-Komponente. Alles läuft in JavaScript im Browser, ohne dass der Header das Gerät verlässt.

Ein Tool, das so gebaut ist, unterscheidet sich für den Datenschutz grundsätzlich von einem klassischen Online-Analyzer: Es gibt keine Übermittlung, keine Speicherung beim Betreiber, keine Logfiles mit fremden Headern. Die Analyse eines Kunden-Headers bleibt damit auf demselben Stand wie das Öffnen der Datei in einem lokalen Editor, nur lesbarer.

## Was ein lokales Tool leisten kann

Der [Mail-Header-Analyzer](/tools/header-analyzer) auf dieser Website ist nach diesem Prinzip gebaut. Der eingefügte Header wird ausschliesslich lokal im Browser ausgewertet. Der Funktionsumfang zeigt, dass dabei nichts verloren geht:

- **Zustellweg mit Laufzeiten:** Die `Received`-Kette wird chronologisch geordnet, die Verweildauer pro Station berechnet und der längste Abschnitt markiert. So ist sichtbar, wo eine langsame Zustellung tatsächlich hing. Uhrenversatz zwischen Servern wird erkannt und ausgewiesen.
- **Transportverschlüsselung pro Hop:** TLS-Version und Cipher werden aus den Received-Zeilen gelesen, wo der empfangende Server sie protokolliert; Microsoft, Postfix und Exim schreiben unterschiedliche Formate.
- **Authentifizierung:** SPF-, DKIM- und DMARC-Ergebnisse aus `Authentication-Results` (RFC 8601) samt Details wie `header.d`, `smtp.mailfrom` und Microsofts `compauth` mit Reason-Code.
- **DMARC-Alignment:** From-Domain, Envelope-From und DKIM-Domain stehen nebeneinander, bewertet nach strict und relaxed Alignment.
- **ARC und DKIM-Integrität:** Eigene Spuren in der Flussgrafik zeigen, von wo bis wo der DKIM-Hash intakt war und ab welcher Station die ARC-Kette die Prüfergebnisse konserviert.
- **Microsoft-Umgebungen:** Die Spam-Filter-Felder (`X-Forefront-Antispam-Report`, SCL, CAT) werden dekodiert, Tenant-Übergänge und die Hybrid-Einstufung im Zustellweg markiert.

Eine Einschränkung gilt für jedes Header-Tool, lokal oder nicht: Es zeigt die dokumentierte Bewertung des Empfangsservers, keine eigene Nachprüfung. Ob ein SPF-Record heute noch so aussieht wie zum Empfangszeitpunkt, beantwortet der Header nicht.

## Einordnung der übrigen Tools

Auch einige andere Anbieter werten inzwischen client-seitig aus; ein Blick in die Datenschutzerklärung und die Netzwerk-Konsole des Browsers schafft Klarheit, ob beim Einfügen tatsächlich keine Anfrage mit dem Header-Inhalt abgeht. Bei klassischen server-seitigen Analyzern gilt die einfache Regel: keine Header aus produktiven Umgebungen oder von Dritten einfügen, sondern höchstens anonymisierte Beispiele.

Für regelmässige Analysen von Incident- oder Support-Headern ist ein lokal laufendes Tool deshalb die naheliegende Wahl: Die Frage, wo die Daten gelandet sind, stellt sich nicht.

## Quellen

1.  [RFC 8601: Message Header Field for Indicating Message Authentication Status](https://datatracker.ietf.org/doc/html/rfc8601): Standard für die Authentication-Results-Kopfzeile, die Grundlage der Authentifizierungs-Auswertung.

2.  [RFC 5321: Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321): Definition der Received-Zeilen (Trace Information), aus denen sich Zustellweg und Laufzeiten rekonstruieren lassen.

3.  [Microsoft Learn: Anti-spam message headers](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo): Referenz der Microsoft-365-spezifischen Header-Felder, die ein Analyzer dekodiert.
