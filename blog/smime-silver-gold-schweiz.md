---
title: "S/MIME für Banken: Silver- oder Gold-Zertifikat von SwissSign?"
navTitle: "S/MIME Silver/Gold"
description: "Eine Bank, die Kundenmails mit S/MIME signiert und verschlüsselt, braucht für persönliche Postfächer Gold-Zertifikate und für Funktionspostfächer Organization-Validated-Zertifikate. Silver verschlüsselt gleich stark, weist die Bank aber nicht als Absender aus. Was SwissSign prüft, was im Zertifikat steht und welche Vorgaben in der Schweiz gelten."
date: "2026-11-16"
kategorie: "E-Mail-Verschlüsselung"
timeToRead: "9 Min. Lesezeit"
themen:
  - "e-mail-verschluesselung"
produkte: []
protokolle:
  - "verschluesselung"
  - "tls"
hauptthema: "e-mail-verschluesselung"
startseiteAb: "2099-12-31"
slug: "smime-silver-gold-schweiz"
translationId: "article-5c2fd8d42f088dcb"
url: "https://rafaelpfister.ch/blog/smime-silver-gold-schweiz"
---
# S/MIME für Banken: Silver- oder Gold-Zertifikat von SwissSign?

Kurzfazit: Eine Bank, die Mails an Kundinnen, Kunden und Geschäftspartner mit S/MIME signiert und verschlüsselt, sollte persönliche Postfächer mit Gold-Zertifikaten und Funktionspostfächer mit Organization-Validated-Zertifikaten von SwissSign ausstatten. Silver verschlüsselt gleich stark, im Zertifikat steht aber nur die E-Mail-Adresse und nicht der Name der Bank. Eine Vorgabe zur Zertifikatsstufe machen weder das Gesetz noch die FINMA.

Die Frage, welche Zertifikatsstufe für den Mailversand nötig ist, kommt in meinen Projekten mit Finanzinstituten regelmässig auf. Meist steht sie am Anfang eines Gateway-Projekts oder wenn Geschäftskunden verlangen, dass vertrauliche Unterlagen nur noch verschlüsselt verschickt werden. Für die Antwort sind zwei Aufgaben von S/MIME zu trennen: die Verschlüsselung, die den Inhalt schützt, und die Signatur, die gegenüber dem Empfänger belegt, wer die Mail verschickt hat.

**Schweiz-Fokus:** Dieser Artikel behandelt die Rechtslage in der Schweiz. Wo die EU-Regeln abweichen, steht ein gekennzeichneter Hinweis.

## Was SwissSign bei Silver und Gold prüft

Seit dem 1. September 2023 gelten für öffentlich vertrauenswürdige S/MIME-Zertifikate die `S/MIME Baseline Requirements` des CA/Browser Forum. Sie unterscheiden vier Validierungstypen, je nachdem, was die Zertifizierungsstelle vor der Ausstellung prüft. SwissSign ordnet seine Produkte in den veröffentlichten Zertifikatsprofilen (CPR S/MIME) diesen Typen zu:

| Produkt | Typ nach CA/B Forum | Was geprüft wird | Was im Zertifikat steht |
|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Kontrolle über das Postfach | nur die E-Mail-Adresse |
| E-Mail ID Gold | `Sponsor-validated` | Organisation und Person | Vor- und Nachname (oder Pseudonym), Organisation, Organisationskennung, Land, E-Mail-Adresse |
| Organization Validated | `Organization-validated` | Organisation | Organisationsname, keine Person |

Bei Silver schickt SwissSign einen Bestätigungscode an die beantragte Adresse. Wer die Mail empfangen kann, erhält das Zertifikat. Die Identität der Person und die Organisation werden nicht geprüft, und im Zertifikat steht die Adresse auch als `Common Name`.

Gold ist bei SwissSign ausschliesslich als `Sponsor-validated` geführt. SwissSign prüft die Organisation, und die Person ist ihr zugeordnet. Ihre Identität prüft entweder SwissSign oder die Organisation selbst als registrierende Stelle, in der Regel im Rahmen einer Managed PKI. Für Funktionspostfächer wie `kundendienst@` oder `zahlungsverkehr@` ist das Profil `Organization-validated` vorgesehen; dort steht die Bank als Organisation im Zertifikat, aber keine Person.

Kryptografisch unterscheiden sich die Stufen nicht. Alle Profile verwenden RSA-Schlüssel ab 2048 Bit, die Schlüsselverwendung `digitalSignature` und `keyEncipherment` sowie die erweiterte Verwendung `id-kp-emailProtection`.

## Was der Unterschied für eine Bank als Absender bedeutet

| | Silver | Gold | Organization Validated |
|---|---|---|---|
| Mails verschlüsseln und entschlüsseln | ja | ja | ja |
| Signatur zeigt im Mailprogramm | E-Mail-Adresse | Name und Bank | Bank |
| Aussage der Signatur | Absender hat Zugriff auf das Postfach | geprüfte Person dieser Bank | Postfach dieser Bank |
| Einsatz bei einer Bank | ungeeignet für Kundenkommunikation | persönliche Postfächer | Funktionspostfächer |

Der Unterschied betrifft die Signatur. Eine Bank signiert ausgehende Mails, damit Empfängerinnen und Empfänger prüfen können, dass die Nachricht tatsächlich von ihr stammt und unterwegs nicht verändert wurde. Mit Silver ist das nur eingeschränkt möglich:

- Wer eine ähnlich lautende Domain registriert, etwa `musterbank-ch.com` statt `musterbank.ch`, erhält für Adressen dieser Domain ohne weitere Prüfung ein Silver-Zertifikat. Seine Phishing-Mails sind dann genauso gültig signiert wie die echten Mails der Bank, und im Zertifikat steht in beiden Fällen nur eine E-Mail-Adresse.
- Wer ein Postfach der Bank übernimmt, kann für diese Adresse ebenfalls ein Silver-Zertifikat beantragen.

Ein Gold- oder Organization-Validated-Zertifikat enthält den geprüften Namen der Bank im Feld `Organization`. Diesen Eintrag erhält nur, wer die Organisationsprüfung durch SwissSign besteht. Für Empfänger, die die Signatur ansehen, ist das der relevante Nachweis.

Für die Verschlüsselung spielt die Stufe keine Rolle. Eine mit Silver verschlüsselte Mail ist genauso gut geschützt wie eine mit Gold verschlüsselte.

## Verschlüsseln setzt Zertifikate der Empfänger voraus

Um eine Mail mit S/MIME zu verschlüsseln, braucht die Bank den öffentlichen Schlüssel des Empfängers. Das eigene Zertifikat der Bank wird dafür nicht verwendet; es dient der Signatur und dem Empfang verschlüsselter Antworten.

Die meisten Privatkundinnen und Privatkunden haben kein S/MIME-Zertifikat. Banken stellen vertrauliche Mails an sie deshalb meist über ein Verschlüsselungs-Gateway zu, etwa SEPPmail (GINA) oder Kiteworks/Totemomail, das die Nachricht in einem Webportal oder als verschlüsselten Anhang bereitstellt. S/MIME-Verschlüsselung kommt vor allem bei Geschäftskunden und Partnern zum Einsatz, die selbst Zertifikate oder ein Gateway betreiben. Arbeiten beide Seiten mit einem Gateway, genügen oft Domain-Zertifikate oder erzwungenes TLS zwischen den Mailservern.

## Zertifikate im Betrieb: Managed PKI und Gateway

Bei mehr als einer Handvoll Postfächer bezieht eine Bank ihre Zertifikate über die Managed PKI von SwissSign. Die Bank tritt dabei als registrierende Stelle auf und bestätigt die Identität der Mitarbeitenden selbst; das entspricht dem Typ `Sponsor-validated`. Gateways wie SEPPmail und Kiteworks/Totemomail können über eine Schnittstelle zur Managed PKI Zertifikate automatisch beantragen, verlängern und den Postfächern zuordnen.

Aus Projekten sind drei Punkte bekannt, die im Betrieb Probleme verursachen:

- Gold-Anträge für Funktionspostfächer scheitern, weil das Gateway Vor- und Nachname aus dem Verzeichnis übernimmt und SwissSign Gattungsbegriffe wie «Support» oder «Helpdesk» im Feld `GivenName` ablehnt. Für solche Postfächer lässt sich im Gateway das Profil `Organization-validated` hinterlegen.
- Automatische Verlängerungen können über Wochen fehlschlagen, ohne dass es auffällt, bis die ersten Zertifikate ablaufen und Kunden verschlüsselte Mails nicht mehr senden können. Die Ablaufdaten der ausgestellten Zertifikate und die Fehlermeldungen der Verlängerungsläufe sollten deshalb überwacht werden.
- Das Gateway signiert seine Anträge an die Managed PKI mit einem eigenen Zertifikat der registrierenden Stelle. Läuft dieses ab, stoppt die gesamte Automatik.

## Rechtliche Vorgaben in der Schweiz

Kein Schweizer Gesetz schreibt S/MIME vor, und keines unterscheidet zwischen Zertifikatsstufen. Die Pflichten einer Bank betreffen den Schutz der Kundendaten; wie sie diesen umsetzt, entscheidet sie selbst.

| Regelung | Inhalt | Bedeutung für den Mailversand der Bank |
|---|---|---|
| BankG Art. 47 | Bankkundengeheimnis | Kundendaten dürfen nicht unbefugt offenbart werden; unverschlüsselter Versand über das Internet ist ein Risiko, das die Bank bewerten muss |
| FINMA-RS 2023/1 | Schutz kritischer Daten | Schutz auch bei der Übertragung, technologieneutral, keine Vorgabe zu Zertifikaten |
| DSG Art. 8 und DSV Art. 1 bis 6 | Datensicherheit, die dem Risiko angemessen ist | Verschlüsselung ist ein übliches Mittel, ein bestimmtes Verfahren ist nicht vorgeschrieben |
| ZertES und OR Art. 14 Abs. 2bis | nur die qualifizierte elektronische Signatur ist der eigenhändigen Unterschrift gleichgestellt | S/MIME-Signaturen ersetzen keine Unterschrift |

### Bankkundengeheimnis und FINMA

Das Bankkundengeheimnis nach Art. 47 BankG verpflichtet die Bank, ihre Organe und Mitarbeitenden, Kundendaten nicht unbefugt zu offenbaren. Das FINMA-Rundschreiben 2023/1 «Operationelle Risiken und Resilienz: Banken» verlangt, dass Banken ihre kritischen Daten über den gesamten Lebenszyklus schützen, auch bei der Übertragung. Wie das geschieht, legt die Bank in ihren eigenen Weisungen fest. Die Wahl der Zertifikatsstufe ist damit Teil der internen Policy und sollte dort dokumentiert sein.

### Datenschutz: DSG und DSV

Art. 8 DSG verpflichtet Verantwortliche, eine dem Risiko angemessene Datensicherheit zu gewährleisten. Die Datenschutzverordnung konkretisiert das mit Schutzzielen wie Vertraulichkeit und Integrität, nennt aber keine Verfahren. S/MIME, ein Gateway mit Webmail-Zustellung oder eine Austauschplattform erfüllen diese Anforderung gleichermassen, sofern sie dem Risiko angemessen eingesetzt werden.

### Elektronische Signatur: ZertES und OR

Das ZertES unterscheidet die elektronische, die fortgeschrittene, die geregelte und die qualifizierte elektronische Signatur. Nach Art. 14 Abs. 2bis OR ist nur die qualifizierte elektronische Signatur mit qualifiziertem Zeitstempel der eigenhändigen Unterschrift gleichgestellt. Die S/MIME-Zertifikate von SwissSign werden nach den Regeln des CA/Browser Forum und den ETSI-Policies LCP (Silver) bzw. NCP (Gold) ausgestellt und sind keine qualifizierten Zertifikate nach ZertES. Verträge oder Erklärungen, für die die Schriftform vorgeschrieben ist, lassen sich mit einer S/MIME-signierten Mail nicht formgültig abgeben, auch nicht mit Gold.

**EU-Hinweis:** In der EU ist nach Art. 25 Abs. 2 eIDAS ebenfalls nur die qualifizierte elektronische Signatur der handschriftlichen Unterschrift gleichgestellt.

## Empfehlung

- **Persönliche Postfächer** der Kundenberatung und anderer Mitarbeitender mit Kundenkontakt: Gold (`Sponsor-validated`) über die Managed PKI.
- **Funktionspostfächer:** `Organization-validated`, damit die Bank als Absender im Zertifikat steht.
- **Silver** nicht für die Kundenkommunikation einsetzen, weil die Signatur die Bank nicht als Absender ausweist.
- **Verschlüsselung an Privatkunden** über die Push- oder Portal-Zustellung des Gateways; S/MIME für Empfänger mit eigenem Zertifikat oder Gateway.
- **Signatur und Absenderprüfung** ergänzen sich: Die S/MIME-Signatur ersetzt SPF, DKIM und DMARC nicht, sie zeigt dem Empfänger aber im Mailprogramm die geprüfte Organisation an.
- **Merkblatt für Kunden:** Beschreiben Sie, welche Zertifikate die Bank verwendet, wie Kunden eine Signatur prüfen und welche Zertifikatsstufen die Bank von Kunden akzeptiert, die selbst verschlüsselt senden wollen. Fehlt ein solches Merkblatt, geht die Frage an die Kundenberatung, die sie meist nicht beantworten kann.

## Quellen

1.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): Zertifikatsprofile von SwissSign (Stand 18.09.2025), Zuordnung von Silver, Gold und Organization Validated zu den Validierungstypen, Subject-Felder und Policy-OIDs.

2.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): verbindliche Regeln für öffentlich vertrauenswürdige S/MIME-Zertifikate, Definition der vier Validierungstypen.

3.  [SEPPmail: Managed PKI SwissSign](https://docs.seppmail.com/en/07_mi_10_mpki_11_swisssign.html): Anbindung des SEPPmail-Gateways an die Managed PKI von SwissSign.

4.  [Bankengesetz (BankG)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): Art. 47 zum Bankkundengeheimnis.

5.  [FINMA: Rundschreiben](https://www.finma.ch/de/dokumentation/rundschreiben/): Übersicht mit dem Rundschreiben 2023/1 «Operationelle Risiken und Resilienz: Banken», in Kraft seit 1. Januar 2024.

6.  [Bundesgesetz über den Datenschutz (DSG)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): Art. 8 zur Datensicherheit.

7.  [Datenschutzverordnung (DSV)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): Art. 1 bis 6 mit Grundsätzen und Schutzzielen der Datensicherheit.

8.  [Bundesgesetz über die elektronische Signatur (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): Arten der elektronischen Signatur und Anforderungen an qualifizierte Zertifikate.

9.  [Obligationenrecht (OR)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): Art. 14 Abs. 2bis zur Gleichstellung der qualifizierten elektronischen Signatur mit der eigenhändigen Unterschrift.
