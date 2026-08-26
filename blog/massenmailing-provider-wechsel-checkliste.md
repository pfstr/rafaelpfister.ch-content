---
title: "Der Provider wechselt die Massenmailing-Plattform: Fragenkatalog und Mailvorlage für Entscheider"
navTitle: "Massenmailing-Wechsel"
description: "Ihr Provider stellt die Massenmailing-Plattform um und Ihr Unternehmen muss mitziehen. Der Artikel erklärt die neun Klärungsblöcke von Rate-Limits bis Auftragsbearbeitung nach revDSG und liefert eine versandfertige Mail, mit der Sie verbindliche Antworten und Vertragszusagen einfordern."
date: "2026-08-03"
kategorie: "SMTP und Mailflow"
timeToRead: "10 Min. Lesezeit"
themen:
  - "smtp-mailflow"
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "dns"
  - "tls"
  - "mail-auth"
hauptthema: "smtp-mailflow"
related:
  - "e-mail-security-gateway-evaluieren"
  - "dmarc-einfuehrung-business-case"
slug: "massenmailing-provider-wechsel-checkliste"
translationId: "article-937453390abec5c8"
url: "https://rafaelpfister.ch/blog/massenmailing-provider-wechsel-checkliste"
zielgruppe: "entscheider"
draft: true
aiPrompt: |
  Du bist mein Assistent für IT-Beschaffung und Vertragsfragen. Unser Provider stellt die Massenmailing-Plattform um und wir müssen auf einen neuen Versandserver wechseln. Erstelle auf Basis der Anfrage-Mail aus diesem Artikel eine auf uns zugeschnittene Version. Frage mich zuerst: 1. Versandvolumen pro Tag und Spitzenlast, 2. eingesetzte Systeme (ERP, Newsletter-Tool, CRM), 3. Branche und regulatorische Anforderungen, 4. ob Schweizer Recht (revDSG) oder zusätzlich die DSGVO anwendbar ist. Streiche danach nicht zutreffende Fragen, ergänze branchenspezifische Punkte und behalte die Struktur mit neun Blöcken sowie die Forderung bei, dass die Antworten als Leistungsbeschreibung, SLA und Vertrag zur Auftragsbearbeitung Vertragsbestandteil werden.
---

Ihr Provider kündigt an, dass Massenmailings künftig über eine neue Versandplattform laufen: anderer Server, andere Limits, gleicher Vertrag? Solche Umstellungen werden gerne als rein technische Migration verkauft, die die IT nebenbei erledigt. Das ist die falsche Betrachtungsebene. Ob Rechnungen, Mahnungen und Newsletter Ihre Kundschaft weiterhin zuverlässig erreichen, entscheidet sich an Rate-Limits, Absenderreputation und Vertragsklauseln, nicht am Serverwechsel selbst.

Dieser Artikel liefert Ihnen zwei Dinge: die neun Klärungsblöcke, die Sie vor der Freigabe der Migration beantwortet haben müssen, und eine versandfertige Mail an den Provider, die Sie kopieren, anpassen und verschicken können.

**Schweiz-Fokus:** Die rechtlichen Aussagen in diesem Artikel beziehen sich auf Schweizer Recht, insbesondere das revidierte Datenschutzgesetz (revDSG) und die Praxis des EDÖB. Wo die Rechtslage in der EU abweicht, ist der Absatz ausdrücklich als EU-Hinweis gekennzeichnet.

## Weshalb die Antworten in den Vertrag gehören

Eine freundliche Antwortmail des Providers ist keine Zusage. Massgebend ist, was in der Leistungsbeschreibung, im SLA und im Vertrag zur Auftragsbearbeitung steht, und in welcher Rangfolge diese Dokumente gelten. Wer nur Antworten sammelt, aber die Vertragsanhänge nicht anpasst, steht beim ersten Vorfall mit den allgemeinen Geschäftsbedingungen da, und die sind selten zu Gunsten des Kunden formuliert.

Deshalb verlangt die Vorlage unten nicht nur Auskunft, sondern ausdrücklich die verbindliche Aufnahme der zugesicherten Eigenschaften in den Vertrag. Bis dahin gilt: keine abschliessende Freigabe der neuen Lösung.

## Die neun Klärungsblöcke

### 1. Versandkapazität und Rate-Limits

Massenmailing-Plattformen begrenzen den Durchsatz pro Minute, Stunde und Tag, oft zusätzlich pro Absenderdomain, IP-Adresse oder Empfängerdomain. Entscheidend ist das Verhalten beim Erreichen des Limits: Stellt die Plattform in eine Warteschlange zu, verzögert sie, oder lehnt sie mit einem SMTP-Fehler ab? Für Ihre Systeme ist das der Unterschied zwischen einer verspäteten Rechnung und einer verlorenen. Lassen Sie sich die Limits, die Statuscodes und die Wiederholungslogik schriftlich geben und prüfen Sie, ob Ihre Spitzenlast (Monatsabschluss, Jahresendversand) hineinpasst.

### 2. Authentisierung und Zugriffsschutz

Die Einlieferung muss mit aktuellen Verfahren abgesichert sein: SMTP AUTH mit starken Passwörtern ist das Minimum, besser sind API-Keys, OAuth 2.0 oder Client-Zertifikate, ergänzt um IP-Allowlisting. Genauso wichtig ist das Administrationsportal: Multi-Faktor-Authentisierung, rollenbasierte Berechtigungen, getrennte technische Konten und eine revisionssichere Protokollierung von Zugriffen und Konfigurationsänderungen. Fragen Sie auch nach dem Prozess für die Sperrung von Zugangsdaten im Vorfallfall.

### 3. Verschlüsselung und technische Sicherheit

Für die Einlieferung und die Zustellung gehört TLS 1.2 oder höher als Mindeststandard vertraglich fixiert. Dazu kommt die Speicherseite: Nachrichten, Empfängerlisten, Warteschlangen, Backups und Protokolle sollen verschlüsselt abgelegt sein. Unabhängige Nachweise wie ISO 27001 oder SOC 2 ersetzen keine eigene Prüfung, sind aber ein belastbares Indiz, dass der Provider seine technischen und organisatorischen Massnahmen im Griff hat.

### 4. Absenderauthentizität und Zustellbarkeit

Hier entscheidet sich der Geschäftsnutzen der ganzen Plattform. SPF, DKIM und DMARC müssen vollständig unterstützt sein, inklusive eigener DKIM-Schlüssel oder Selektoren und angekündigter Schlüsselwechsel. Klären Sie, ob Sie über dedizierte oder geteilte IP-Adressen versenden: Bei geteilten Adressen erbt Ihre Absenderreputation das Verhalten der anderen Kunden, und der Provider muss erklären, wie er Sie davor schützt. Bounce-Auswertung, Beschwerde-Feedback und eine zentrale Suppression List gehören ebenfalls zur Grundausstattung. Ob Ihre Domain heute sauber aufgestellt ist, prüfen Sie in Sekunden mit dem [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check).

### 5. Datenbearbeitung und Datenschutz

Beim Massenmailing bearbeitet der Provider Personendaten in Ihrem Auftrag: Empfängeradressen, Nachrichteninhalte, Versandprotokolle. Nach Art. 9 revDSG bleibt Ihr Unternehmen dafür verantwortlich und muss die Auftragsbearbeitung vertraglich regeln: Zwecke, Bearbeitungsorte, Aufbewahrungsfristen, Löschung, Unterauftragnehmer und die technischen und organisatorischen Massnahmen. Der EDÖB hält in seinen Hinweisen zum Outsourcing fest, dass Weisungsbindung, Datensicherheit und Mitwirkungspflichten geprüft und abgesichert sein müssen. Verlangen Sie die Liste der Unterauftragnehmer und ein Widerspruchsrecht bei Wechseln.

**EU-Hinweis:** Bearbeitet der Provider Daten von Personen in der EU oder hat Ihr Unternehmen dort Niederlassungen, kommt zusätzlich die DSGVO ins Spiel. Dann braucht es einen Auftragsverarbeitungsvertrag nach Art. 28 DSGVO, und Übermittlungen in Drittländer müssen über geeignete Garantien wie Standardvertragsklauseln abgesichert sein. Lassen Sie die Vereinbarung in diesem Fall juristisch auf beide Rechtsordnungen abstimmen.

### 6. Sicherheits- und Datenschutzvorfälle

Vereinbaren Sie eine verbindliche Frist, innert der der Provider Sie über Vorfälle informiert, samt Mindestinhalt der Erstmeldung und einer rund um die Uhr erreichbaren Kontaktstelle. Der Hintergrund: Nach Art. 24 revDSG müssen Sie als Verantwortlicher Verletzungen der Datensicherheit mit hohem Risiko so rasch als möglich dem EDÖB melden. Das können Sie nur, wenn der Auftragsbearbeiter Sie unverzüglich und vollständig informiert.

**EU-Hinweis:** Unter der DSGVO gilt für die Meldung an die Aufsichtsbehörde die konkrete Frist von 72 Stunden nach Kenntnisnahme (Art. 33 DSGVO). Wenn die DSGVO anwendbar ist, muss die Meldekette mit dem Provider entsprechend enger getaktet sein.

### 7. Verfügbarkeit, Support und Änderungen

Verfügbarkeit, Reaktions- und Wiederherstellungszeiten je Störungskategorie gehören ins SLA, ebenso Servicegutschriften bei Verletzung. Mindestens so wichtig ist die Änderungsklausel: Mit welcher Vorlaufzeit darf der Provider Schnittstellen, Limits oder Authentisierungsverfahren ändern? Eine Plattform, die Ihnen mit zwei Wochen Frist das Authentisierungsverfahren wechselt, produziert genau die Migration, die Sie gerade durchleben, einfach regelmässig.

### 8. Migration und Abnahme

Verlangen Sie eine Testumgebung, einen Parallelbetrieb mit der bisherigen Lösung und definierte Abnahme- und Lasttests. Klären Sie, wer die DNS-Anpassungen (SPF, DKIM, DMARC, Return-Path) vornimmt, wie der Rollback-Prozess aussieht und bis wann die alte Lösung verfügbar bleibt. Und: Welche Migrationsleistungen sind im Preis enthalten, welche werden zusätzlich verrechnet?

### 9. Vertragliche und kommerzielle Regelungen

Der letzte Block bündelt, was verbindlich in den Vertrag oder seine Anhänge gehört: Leistungsbeschreibung, SLA, Sicherheitsmassnahmen, Auftragsbearbeitung, Meldefristen, Ankündigungsfristen, Preise, Haftung, Exit- und Datenrückgaberegelungen sowie die Rangfolge der Vertragsdokumente. Die Rangfolge ist der unscheinbarste und zugleich wichtigste Punkt: Sie verhindert, dass individuell verhandelte Zusicherungen durch allgemeine Geschäftsbedingungen wieder relativiert werden.

## Die versandfertige Mail

Ersetzen Sie die Platzhalter in eckigen Klammern, streichen Sie nicht zutreffende Fragen und versenden Sie die Anfrage an Ihre Ansprechperson beim Provider. Die Vorlage ist bewusst vollständig gehalten; kürzen ist einfacher als nachfordern.

```text
Betreff: Wechsel der Massenmailing-Infrastruktur: Klärungs- und Vertragsbedarf

Guten Tag [Name]

Sie haben uns darüber informiert, dass unsere Massenmailings künftig über eine
neue Server- bzw. Versandplattform abgewickelt werden müssen.

Damit wir die technische Migration, die Betriebssicherheit und unsere
Datenschutzpflichten beurteilen können, bitten wir um verbindliche
Beantwortung der folgenden Punkte:

1. Versandkapazität und Rate-Limits
   - Welche Limits gelten pro Minute, Stunde und Tag?
   - Gelten zusätzliche Limits pro Absender, Domain, IP-Adresse,
     Empfängerdomain oder Verbindung?
   - Welche Burst-Limits und maximalen parallelen Verbindungen sind zulässig?
   - Können die Limits bei Bedarf temporär oder dauerhaft erhöht werden?
   - Wie reagiert die Plattform beim Erreichen eines Limits: Warteschlange,
     verzögerter Versand oder Ablehnung?
   - Welche SMTP-Statuscodes werden zurückgegeben, und wie lange werden nicht
     zustellbare Nachrichten erneut versucht?
   - Welche Versandmengen und Zustellzeiten sichern Sie vertraglich zu?

2. Authentisierung und Zugriffsschutz
   - Welche Authentisierungsmethoden werden unterstützt (SMTP AUTH,
     OAuth 2.0, API-Key, Client-Zertifikate)?
   - Werden veraltete Verfahren wie Basic Authentication ohne zusätzliche
     Absicherung ausgeschlossen?
   - Können Zugriffe zusätzlich mittels IP-Allowlisting, gegenseitiger
     TLS-Authentisierung, VPN oder privater Netzwerkverbindung abgesichert
     werden?
   - Unterstützt das Administrationsportal MFA, rollenbasierte Berechtigungen
     und getrennte technische Konten?
   - Können Berechtigungen auf bestimmte Absenderdomains, IP-Adressen oder
     Versandvolumen eingeschränkt werden?
   - Werden Zugriffe und Konfigurationsänderungen revisionssicher
     protokolliert?
   - Wie werden Zugangsdaten ausgestellt, gespeichert, erneuert und bei einem
     Sicherheitsvorfall gesperrt?

3. Verschlüsselung und technische Sicherheit
   - Welche TLS-Versionen und Cipher Suites werden für die Einlieferung
     unterstützt beziehungsweise erzwungen?
   - Wird TLS auch bei der Zustellung an Empfängerserver verwendet?
   - Können Mindestanforderungen wie TLS 1.2 oder höher verbindlich vereinbart
     werden?
   - Werden Nachrichten, Empfängeradressen, Warteschlangen, Backups und
     Protokolle verschlüsselt gespeichert?
   - Welche technischen und organisatorischen Sicherheitsmassnahmen gelten für
     die neue Plattform?
   - Liegen aktuelle unabhängige Prüfberichte oder Zertifizierungen vor
     (z. B. ISO 27001, SOC 2)?
   - Wie werden Schwachstellen, Sicherheitsupdates, Schlüsselrotationen und
     Penetrationstests gehandhabt?

4. Absenderauthentizität und Zustellbarkeit
   - Unterstützt die Plattform SPF, DKIM und DMARC vollständig?
   - Wer richtet SPF, DKIM, Return-Path und gegebenenfalls eine
     Tracking-Domain ein?
   - Können wir eigene DKIM-Schlüssel beziehungsweise eigene Selektoren
     verwenden?
   - Wie erfolgen DKIM-Schlüsselwechsel, und wie lange im Voraus werden diese
     angekündigt?
   - Erfolgt der Versand über dedizierte oder gemeinsam genutzte IP-Adressen?
   - Falls IP-Adressen geteilt werden: Wie schützen Sie unsere
     Absenderreputation vor dem Verhalten anderer Kunden?
   - Wer übernimmt IP-Warm-up, Reputationsüberwachung und die Behandlung von
     Blocklist-Einträgen?
   - Welche Auswertungen stehen für Zustellungen, Bounces, Beschwerden und
     Ablehnungsgründe zur Verfügung?
   - Unterstützen Sie Bounce- und Complaint-Feedback sowie eine zentrale
     Suppression List?
   - Welche messbaren Zustellbarkeits- oder Reputationsleistungen werden
     zugesichert?

5. Datenbearbeitung und Datenschutz (Schweizer revDSG)
   - Welche Personen- und Kommunikationsdaten werden bearbeitet oder
     gespeichert, und zu welchen Zwecken?
   - In welchen Ländern und Rechenzentren erfolgen Bearbeitung, Speicherung,
     Support und Backup?
   - Werden Daten für eigene Zwecke, Analysen oder Produktverbesserungen
     verwendet?
   - Welche Aufbewahrungsfristen gelten für Nachrichteninhalte,
     Empfängeradressen, Versandprotokolle, Backups und Statistiken?
   - Wie können Daten berichtigt, exportiert und vollständig gelöscht werden?
   - Welche Unterauftragnehmer werden eingesetzt, und wie werden wir über
     Änderungen informiert?
   - Können wir neuen Unterauftragnehmern aus sachlichen Datenschutz- oder
     Sicherheitsgründen widersprechen?
   - Stellen Sie einen Vertrag zur Auftragsbearbeitung nach Art. 9 revDSG
     inklusive dokumentierter technischer und organisatorischer Massnahmen
     bereit?
   - Wie unterstützen Sie uns bei Auskunfts-, Lösch- und weiteren
     Betroffenenbegehren?
   - Wie werden allfällige Datenübermittlungen ins Ausland rechtlich und
     technisch abgesichert?
   - [Nur bei EU-Bezug:] Stellen Sie einen Auftragsverarbeitungsvertrag nach
     Art. 28 DSGVO inklusive geeigneter Garantien für Drittlandtransfers
     bereit?

6. Sicherheits- und Datenschutzvorfälle
   - Innerhalb welcher verbindlichen Frist werden wir über einen tatsächlichen
     oder vermuteten Vorfall informiert?
   - Welche Mindestinformationen enthält eine Erstmeldung?
   - Wer ist die rund um die Uhr erreichbare Kontaktstelle?
   - Wie unterstützen Sie die Untersuchung, Eindämmung, Dokumentation und die
     Erfüllung unserer gesetzlichen Meldepflichten (Art. 24 revDSG)?
   - Werden Ursachenanalyse und Abschlussbericht innerhalb definierter Fristen
     bereitgestellt?
   - Wer trägt die Kosten eines von Ihnen oder Ihren Unterauftragnehmern
     verursachten Vorfalls?

7. Verfügbarkeit, Support und Änderungen
   - Welche Verfügbarkeit und welche maximalen Wiederherstellungszeiten werden
     vertraglich zugesichert?
   - Welche Reaktions- und Lösungszeiten gelten je Störungskategorie?
   - Gibt es einen 24/7-Support für kritische Versand- oder
     Sicherheitsstörungen?
   - Wie werden Wartungsfenster und nicht dringende Änderungen angekündigt?
   - Mit welcher Vorlaufzeit dürfen Schnittstellen, Limits,
     Authentisierungsverfahren oder Sicherheitsanforderungen geändert werden?
   - Welche Servicegutschriften oder anderen Rechtsfolgen gelten bei
     SLA-Verletzungen?
   - Wie sind Backup, Wiederanlauf, Redundanz und Disaster Recovery
     ausgestaltet?

8. Migration und Abnahme
   - Welche technische Dokumentation, Endpunkte, Ports und
     Konfigurationsänderungen sind erforderlich?
   - Gibt es eine Testumgebung oder Testzugänge?
   - Ist ein Parallelbetrieb mit der bisherigen Lösung möglich?
   - Welche Abnahme- und Lasttests sind vorgesehen?
   - Unterstützen Sie die Anpassung von DNS-, SPF-, DKIM- und
     DMARC-Einträgen?
   - Wie sieht der Rollback-Prozess aus, wenn bei der Umstellung Probleme
     auftreten?
   - Bis wann bleibt die bisherige Lösung verfügbar?
   - Welche Migrationsleistungen sind im Preis enthalten, und welche werden
     zusätzlich verrechnet?

9. Vertragliche und kommerzielle Regelungen

Wir bitten darum, mindestens folgende Punkte verbindlich in den Vertrag
beziehungsweise dessen Anhänge aufzunehmen:

   - vollständige Leistungsbeschreibung und Kapazitätsgrenzen;
   - SLA, Support-, Reaktions- und Wiederherstellungszeiten;
   - Sicherheitsanforderungen und vereinbarte technische und organisatorische
     Massnahmen;
   - Auftragsbearbeitung, Bearbeitungsorte und Unterauftragnehmer;
   - verbindliche Meldefristen bei Sicherheitsvorfällen;
   - Ankündigungsfristen für technische und vertragliche Änderungen;
   - Preise, Mengentarife, Zusatzkosten und Regeln für Preisanpassungen;
   - Haftung bei Ausfällen, Datenverlusten, Sicherheitsvorfällen und
     Reputationsschäden;
   - Kündigungs-, Exit- und Datenrückgaberegelungen;
   - Pflicht zur sicheren Löschung nach Vertragsende;
   - Unterstützung bei einer späteren Migration zu einem anderen Anbieter;
   - Rangfolge der Vertragsdokumente, sodass die vereinbarten Zusicherungen
     nicht durch allgemeine Geschäftsbedingungen relativiert werden.

Bitte stellen Sie uns neben den Antworten auch die aktuelle technische
Dokumentation, das SLA, den Vertrag zur Auftragsbearbeitung, die Liste der
Unterauftragnehmer, die Sicherheitsdokumentation und das vorgesehene
Migrationskonzept zur Verfügung.

Wir bitten zudem um Bestätigung, welche der genannten Eigenschaften bereits
Standardbestandteil der Leistung sind und welche zusätzlich vereinbart oder
beauftragt werden müssen.

Bis zur Klärung und vertraglichen Fixierung dieser Punkte können wir die neue
Lösung noch nicht abschliessend freigeben.

Freundliche Grüsse
[Name]
[Unternehmen]
```

## So setzen Sie die Vorlage ein

Priorisieren Sie nach Ihrer Situation: Wer nur Transaktionsmails über den Provider verschickt, kann die Newsletter-spezifischen Punkte kürzen; wer Marketing-Volumen fährt, sollte Block 4 vollständig stellen. Setzen Sie dem Provider eine Antwortfrist, die vor dem geplanten Migrationstermin liegt, und lassen Sie den Datenschutzteil von Ihrer Rechtsabteilung oder externen Beratung gegenlesen. Die Antworten fliessen anschliessend in die Vertragsanhänge, nicht in einen Mailordner.

Weitere Vorlagen und Werkzeuge für Entscheider, etwa den Kriterienkatalog für die [Evaluation eines E-Mail-Security-Gateways](https://rafaelpfister.ch/blog/e-mail-security-gateway-evaluieren), finden Sie gesammelt auf der [Entscheider-Seite](https://rafaelpfister.ch/cio).

## Quellen

1.  [EDÖB: Outsourcing und Auftragsdatenbearbeitung](https://www.edoeb.admin.ch/de/outsourcing-auftragsdatenbearbeitung): Hinweise des Eidgenössischen Datenschutz- und Öffentlichkeitsbeauftragten zur Verantwortung des Auftraggebers bei ausgelagerter Datenbearbeitung

2.  [EDÖB: Leitfaden zu den technischen und organisatorischen Massnahmen](https://www.edoeb.admin.ch/dam/de/sd-web/eVhrh8wY3QcR/leitfaden_tom.pdf): Konkretisiert die Anforderungen an TOM, auf die der Vertrag zur Auftragsbearbeitung verweisen sollte

3.  [Bundesgesetz über den Datenschutz (DSG) vom 25. September 2020](https://www.fedlex.admin.ch/eli/cc/2022/491/de): Massgebliche Artikel für diesen Anwendungsfall sind Art. 9 (Auftragsbearbeitung) und Art. 24 (Meldung von Verletzungen der Datensicherheit)

4.  [Verordnung (EU) 2016/679 (DSGVO)](https://eur-lex.europa.eu/eli/reg/2016/679/oj): Für den gekennzeichneten EU-Hinweis relevant sind Art. 28 (Auftragsverarbeiter) und Art. 33 (Meldefrist von 72 Stunden)
