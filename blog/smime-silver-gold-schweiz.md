---
title: "S/MIME bei Banken: Rechtslage in der Schweiz und die Wahl zwischen Silver und Gold"
navTitle: "S/MIME Silver/Gold"
description: "Welche rechtlichen Vorgaben gelten, wenn eine Schweizer Bank Mails an Kunden signiert und verschlüsselt: Bankkundengeheimnis, FINMA-Rundschreiben 2023/1, DSG, ZertES, Beweisrecht und Aufbewahrung. Dazu, welche SwissSign-Zertifikatsstufe sich daraus ableitet."
date: "2026-11-16"
kategorie: "E-Mail-Verschlüsselung"
timeToRead: "12 Min. Lesezeit"
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
# S/MIME bei Banken: Rechtslage in der Schweiz und die Wahl zwischen Silver und Gold

Kurzfazit: Kein Schweizer Erlass schreibt einer Bank S/MIME oder eine bestimmte Zertifikatsstufe vor. Das Bankkundengeheimnis, das FINMA-Rundschreiben 2023/1 und das Datenschutzgesetz verlangen aber, dass Kundendaten beim Versand angemessen geschützt sind, und die Bank muss ihre Lösung begründen und dokumentieren können. Eine S/MIME-Signatur ersetzt keine Unterschrift, auch nicht mit einem Gold-Zertifikat. Als Beweismittel und als Nachweis der Herkunft ist Gold dem Silver-Zertifikat klar überlegen, weil nur Gold den geprüften Namen der Bank enthält.

Die Frage, welches SwissSign-Zertifikat eine Bank für den Mailversand braucht, kommt in meinen Projekten mit Finanzinstituten regelmässig auf. Hinter der technischen Frage steht meist eine rechtliche: Was muss die Bank tun, um ihre Pflichten beim Versand von Kundendaten per Mail zu erfüllen, und welche Wirkung hat eine signierte Mail?

**Schweiz-Fokus:** Dieser Artikel behandelt die Rechtslage in der Schweiz. Wo die EU-Regeln abweichen, steht ein gekennzeichneter Hinweis. Er ersetzt keine Rechtsberatung im Einzelfall.

## Die Rechtsgrundlagen im Überblick

| Regelung | Inhalt | Bedeutung für den Mailversand |
|---|---|---|
| BankG Art. 47 | Bankkundengeheimnis, strafbewehrt auch bei Fahrlässigkeit | Kundendaten dürfen nicht unbefugt offenbart werden |
| FINMA-RS 2023/1, Rz 73 bis 82 | Schutz kritischer Daten über den ganzen Lebenszyklus | Schutz auch beim Versand, technologieneutral |
| DSG Art. 8 und 24, DSV Art. 1 bis 6 | Datensicherheit, Meldung von Verletzungen | angemessene Massnahmen, Meldung an den EDÖB bei hohem Risiko |
| ZertES und OR Art. 14 Abs. 2bis | Arten elektronischer Signaturen, Formgültigkeit | S/MIME-Signaturen ersetzen keine Unterschrift |
| ZPO Art. 157 und 177 | Beweismittel und freie Beweiswürdigung | signierte Mails sind Urkunden mit frei zu würdigender Beweiskraft |
| OR Art. 958f, GeBüV | Aufbewahrung von Geschäftsunterlagen | verschlüsselte Mails müssen lesbar bleiben |

Keiner dieser Erlasse nennt S/MIME oder eine Zertifikatsstufe. Sie legen Schutzziele und Rechtsfolgen fest; wie die Bank sie erreicht, entscheidet sie selbst.

## Bankkundengeheimnis

Art. 47 BankG bestraft, wer als Organ, Angestellter oder Beauftragter einer Bank ein Geheimnis offenbart, das ihm in dieser Eigenschaft anvertraut wurde. Strafbar ist auch die fahrlässige Begehung. Die Norm richtet sich an natürliche Personen; zugrunde liegt aber das vertragliche Verhältnis zwischen der Kundin oder dem Kunden als Geheimnisherr und der Bank.

Für den Mailversand bedeutet das: Eine unverschlüsselte Mail mit Kundendaten, die an eine falsche Adresse geht oder unterwegs mitgelesen wird, kann eine fahrlässige Offenbarung sein. Die Bank muss deshalb organisatorisch und technisch dafür sorgen, dass Mitarbeitende Kundendaten nur geschützt verschicken.

Der Kunde kann als Geheimnisherr in die Offenbarung einwilligen. Viele Banken lassen sich deshalb in den AGB oder in einer separaten Vereinbarung bestätigen, dass der Kunde die Risiken der unverschlüsselten Kommunikation per Mail kennt und sie wünscht. Eine solche Einwilligung deckt nur die Kommunikation mit diesem Kunden und nur im vereinbarten Umfang. Für Unterlagen, die Dritte betreffen, oder für die Kommunikation mit Geschäftspartnern ersetzt sie die Verschlüsselung nicht.

## FINMA-Rundschreiben 2023/1: kritische Daten

Das FINMA-Rundschreiben 2023/1 «Operationelle Risiken und Resilienz: Banken» ist seit dem 1. Januar 2024 in Kraft und hat das Rundschreiben 2008/21 abgelöst. Den früheren Fokus auf die Vertraulichkeit von Kundenidentifikationsdaten (CID) hat die FINMA auf kritische Daten allgemein erweitert. Was kritisch ist, legt jede Bank risikobasiert selbst fest; Kundendaten zählen in Bezug auf die Vertraulichkeit in aller Regel dazu.

Für den Mailversand sind vor allem diese Randziffern relevant:

| Rz | Inhalt | Bezug zum Mailversand |
|---|---|---|
| 73 | kritische Daten identifizieren und nach Kritikalität kategorisieren | festlegen, welche Daten nur geschützt verschickt werden dürfen |
| 74, 75 | Vertraulichkeit, Integrität und Verfügbarkeit über den gesamten Lebenszyklus gewährleisten | der Versand ist Teil des Lebenszyklus |
| 76 | kritische Daten vor Zugriff durch Unberechtigte schützen | Schutz der Mail auf dem Transportweg und beim Empfänger |
| 79 | erhöhte Risiken bei Speicherung im oder Zugriff aus dem Ausland begrenzen | betrifft Gateways und Mailplattformen mit Standort im Ausland |
| 81 | wesentliche Vorfälle unverzüglich der FINMA melden | etwa ein Massenversand von Kundendaten an falsche Empfänger |
| 82 | Dienstleister, die kritische Daten bearbeiten, sorgfältig auswählen und überwachen | betrifft Anbieter von Verschlüsselungs-Gateways und Zertifizierungsstellen mit Managed PKI |

Das Rundschreiben ist technologieneutral und nennt weder S/MIME noch Zertifikatsstufen. In ihren Erläuterungen zur Anhörung führt die FINMA Datenverschlüsselung als Beispiel für die besonderen Massnahmen an, mit denen kritische Daten gesichert werden. Die Bank muss ihre Lösung in eigenen Weisungen festlegen und dort auch angeben, welche Zertifikatsstufe sie einsetzt und warum.

## Datenschutz: DSG und DSV

Art. 8 DSG verpflichtet Verantwortliche, eine dem Risiko angemessene Datensicherheit zu gewährleisten. Die Datenschutzverordnung konkretisiert das mit Schutzzielen wie Vertraulichkeit, Integrität und Nachvollziehbarkeit, schreibt aber kein Verfahren vor. S/MIME, ein Gateway mit Zustellung über ein Webportal oder eine Austauschplattform erfüllen die Anforderung gleichermassen, wenn sie dem Risiko angemessen eingesetzt werden.

Führt eine Verletzung der Datensicherheit voraussichtlich zu einem hohen Risiko für die Persönlichkeit oder die Grundrechte der betroffenen Personen, muss die Bank sie nach Art. 24 DSG so rasch wie möglich dem EDÖB melden. Diese Meldepflicht besteht neben der Meldepflicht gegenüber der FINMA. War die Mail verschlüsselt, ist das Risiko bei einem Fehlversand in der Regel gering, und eine Meldung entfällt.

## Was eine S/MIME-Signatur rechtlich bewirkt

### Keine Unterschrift im Sinne des OR

Das ZertES unterscheidet die elektronische, die fortgeschrittene, die geregelte und die qualifizierte elektronische Signatur sowie für Unternehmen das geregelte elektronische Siegel. Nach Art. 14 Abs. 2bis OR ist nur die qualifizierte elektronische Signatur mit qualifiziertem Zeitstempel der eigenhändigen Unterschrift gleichgestellt.

Die S/MIME-Zertifikate von SwissSign werden nach den Regeln des CA/Browser Forum und den ETSI-Policies LCP (Silver) bzw. NCP (Gold) ausgestellt. Sie sind keine qualifizierten Zertifikate nach ZertES. Erklärungen, für die das Gesetz oder ein Vertrag die Schriftform verlangt, kann eine Bank deshalb nicht formgültig per S/MIME-signierter Mail abgeben, auch nicht mit Gold.

### Persönliche Signatur und Gateway-Signatur

Eine fortgeschrittene Signatur setzt unter anderem voraus, dass der Signaturschlüssel unter der alleinigen Kontrolle der unterzeichnenden Person steht. Signiert ein Gateway alle ausgehenden Mails zentral mit Schlüsseln, die auf dem Gateway liegen, ist diese Voraussetzung nicht erfüllt. Die Signatur belegt dann, dass die Mail aus dem System der Bank stammt und nicht verändert wurde, aber nicht, dass eine bestimmte Person sie freigegeben hat. Sie entspricht damit inhaltlich eher einem Siegel der Organisation, ohne die Anforderungen an ein geregeltes Siegel nach ZertES zu erfüllen.

### Beweiswert

Eine Mail ist nach Art. 177 ZPO eine Urkunde, und das Gericht würdigt ihre Beweiskraft nach Art. 157 ZPO frei. Eine gültige S/MIME-Signatur zeigt, dass die Mail seit dem Signieren nicht verändert wurde und mit dem Schlüssel eines bestimmten Zertifikats signiert ist. Wie viel das über den Absender aussagt, hängt davon ab, was die Zertifizierungsstelle vor der Ausstellung geprüft hat. Hier liegt der rechtlich relevante Unterschied zwischen Silver und Gold.

## Silver oder Gold aus rechtlicher Sicht

Seit dem 1. September 2023 gelten für öffentlich vertrauenswürdige S/MIME-Zertifikate die `S/MIME Baseline Requirements` des CA/Browser Forum. SwissSign ordnet seine Produkte in den Zertifikatsprofilen (CPR S/MIME) diesen Validierungstypen zu:

| Produkt | Typ nach CA/B Forum | Was SwissSign prüft | Was im Zertifikat steht | Aussage der Signatur |
|---|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Kontrolle über das Postfach | nur die E-Mail-Adresse | jemand mit Zugriff auf dieses Postfach |
| E-Mail ID Gold | `Sponsor-validated` | Organisation und Person | Name, Organisation, Organisationskennung, Land, E-Mail-Adresse | eine geprüfte Person dieser Bank |
| Organization Validated | `Organization-validated` | Organisation | Name der Organisation, keine Person | ein Postfach dieser Bank |

Kryptografisch sind die Stufen gleichwertig. Für die Verschlüsselung und damit für die Schutzpflichten aus Bankkundengeheimnis, FINMA-Rundschreiben und DSG spielt die Stufe deshalb keine Rolle.

Für die Signatur ist der Unterschied erheblich. Ein Silver-Zertifikat erhält jeder, der Mails an eine Adresse empfangen kann. Wer eine ähnlich lautende Domain registriert, etwa `musterbank-ch.com` statt `musterbank.ch`, kann damit Mails verschicken, die genauso gültig signiert sind wie die echten Mails der Bank. In einem Streit darüber, ob eine Mail von der Bank stammt, belegt eine Silver-Signatur deshalb wenig. Ein Gold- oder Organization-Validated-Zertifikat enthält dagegen den von SwissSign geprüften Namen der Bank. Bei Gold kommt die Zuordnung zu einer Person hinzu, deren Identität die Bank im Rahmen der Managed PKI als registrierende Stelle bestätigt.

## Aufbewahrung: OR und GeBüV

Nach Art. 958f OR sind Geschäftsbücher und Buchungsbelege zehn Jahre aufzubewahren, und die Geschäftsbücherverordnung (GeBüV) verlangt, dass elektronisch aufbewahrte Unterlagen während dieser Zeit lesbar gemacht werden können. Für Banken kommen aufsichtsrechtliche und vertragliche Aufbewahrungspflichten hinzu. Daraus ergeben sich zwei Anforderungen an den S/MIME-Betrieb:

- Verschlüsselte Mails müssen über die gesamte Aufbewahrungsfrist entschlüsselt werden können. Die Bank muss die privaten Schlüssel abgelaufener Zertifikate deshalb sicher aufbewahren oder die Mails vor der Archivierung entschlüsseln.
- Soll eine Signatur noch nach Jahren als Beweis dienen, muss nachvollziehbar bleiben, dass das Zertifikat zum Zeitpunkt des Signierens gültig war. Dafür sind die Prüfergebnisse oder die Sperrinformationen zum Signaturzeitpunkt mit zu archivieren.

**EU-Hinweis:** In der EU ist nach Art. 25 Abs. 2 eIDAS ebenfalls nur die qualifizierte elektronische Signatur der handschriftlichen Unterschrift gleichgestellt. Für Finanzunternehmen in der EU gilt seit dem 17. Januar 2025 zusätzlich die DORA-Verordnung mit eigenen Anforderungen an das IKT-Risikomanagement.

## Beispiel: Ein Gateway für alle Mitarbeitenden

Viele Banken setzen ein zentrales Verschlüsselungs-Gateway ein, etwa PrivaSphere, SEPPmail oder Totemomail, damit alle Mitarbeitenden Mails verschlüsseln und signieren können, ohne selbst Zertifikate zu verwalten. Die Schlüssel liegen auf dem Gateway, und das Gateway signiert und verschlüsselt stellvertretend für die einzelnen Postfächer. Für diesen Aufbau ergibt sich aus den vorangehenden Abschnitten folgende Wahl:

| Postfach | Stufe | Grund |
|---|---|---|
| persönliche Postfächer | E-Mail ID Gold | weist die Bank mit geprüftem Namen und Organisationskennung als Absender aus |
| Funktionspostfächer (Kundendienst, Zahlungsverkehr) | Organization Validated | weist die Bank aus, ohne eine einzelne Person zu nennen |
| keine | E-Mail ID Silver | belegt nur den Zugriff auf ein Postfach; Mails von ähnlich lautenden Domains sind genauso gültig signiert |

Weil das Gateway signiert und nicht die einzelne Person, stehen die Schlüssel nicht unter der alleinigen Kontrolle der Mitarbeitenden. Die Signatur wirkt rechtlich wie ein Siegel der Bank und nicht wie eine persönliche Signatur. Entscheidend ist darum, dass die Organisation im Zertifikat geprüft ausgewiesen ist, und das leisten nur Gold und Organization Validated. Für die Verschlüsselung selbst ist die Stufe ohne Bedeutung.

Vor der Einführung sind mit dem Gateway-Anbieter und der Zertifizierungsstelle folgende Punkte zu klären:

- **Ausstellung und Erneuerung:** Ob das Gateway Gold-Zertifikate automatisiert über die Managed PKI der Zertifizierungsstelle bezieht und erneuert. Bei Gold bestätigt die Bank als registrierende Stelle die Identität jeder Person; dafür braucht es einen Prozess für Ein- und Austritte, Namensänderungen und Sperrungen.
- **Dienstleisterprüfung:** Gateway-Anbieter und Managed PKI bearbeiten kritische Daten im Sinne von Rz 82 des FINMA-Rundschreibens 2023/1. Der Standort von Schlüsseln und Daten ist zu dokumentieren; bei Speicherung im Ausland oder Zugriff von dort gilt zusätzlich Rz 79.
- **Archivierung:** Die privaten Schlüssel abgelaufener Zertifikate und die Prüfdaten zum Signaturzeitpunkt müssen über die gesamte Aufbewahrungsfrist verfügbar bleiben.
- **Weisung:** Die gewählten Stufen und die zugelassenen Verfahren gehören in die interne Weisung, die gegenüber FINMA und Revision als Nachweis dient.

## Empfehlung

- **Weisung festlegen:** Die Bank legt in einer internen Weisung fest, welche Daten nur verschlüsselt verschickt werden dürfen, welche Verfahren dafür zugelassen sind und welche Zertifikatsstufen sie einsetzt. Das ist die Grundlage, um die Anforderungen aus dem FINMA-Rundschreiben und dem DSG nachzuweisen.
- **Gold für persönliche Postfächer, Organization Validated für Funktionspostfächer:** Nur diese Stufen weisen die Bank im Zertifikat als Absender aus und haben damit einen Beweiswert, der über den Zugriff auf ein Postfach hinausgeht. Silver eignet sich nicht für die Kommunikation mit Kunden.
- **Einwilligungen klären:** Wer mit Kunden unverschlüsselt per Mail kommuniziert, braucht deren dokumentierte Einwilligung. Sie ersetzt die Verschlüsselung gegenüber Dritten und Geschäftspartnern nicht.
- **Formgebundene Erklärungen nicht per Mail:** Für Erklärungen, die Schriftform verlangen, braucht es eine qualifizierte elektronische Signatur oder eine eigenhändige Unterschrift.
- **Dienstleister prüfen:** Gateway-Anbieter und die Managed PKI der Zertifizierungsstelle fallen unter Rz 82 des FINMA-Rundschreibens, bei Standorten im Ausland zusätzlich unter Rz 79.
- **Schlüssel und Prüfdaten archivieren:** Private Schlüssel für die Entschlüsselung und die Daten zur Signaturprüfung müssen über die Aufbewahrungsfrist verfügbar bleiben.
- **Meldeprozesse verbinden:** Ein Fehlversand von Kundendaten kann gleichzeitig eine Meldung an die FINMA (Rz 81) und an den EDÖB (Art. 24 DSG) auslösen. Beide Pflichten lassen sich in einem gemeinsamen Vorfallprozess abbilden.

## Quellen

1.  [FINMA-Rundschreiben 2023/1 «Operationelle Risiken und Resilienz: Banken»](https://www.finma.ch/de/~/media/finma/dokumente/dokumentencenter/myfinma/rundschreiben/finma-rs-2023-01-20221207.pdf): Rz 73 bis 82 zu kritischen Daten, in Kraft seit 1. Januar 2024.

2.  [datenrecht.ch: FINMA-RS 2023/1 mit Erläuterungen](https://datenrecht.ch/wp-content/uploads/FINMA-RS-OpRisk-%E2%80%93-Text-revidiert-1.1.2024.pdf): revidierter Text mit den Ausführungen der FINMA aus der Anhörung, unter anderem zur Datenverschlüsselung als Schutzmassnahme.

3.  [Bankengesetz (BankG)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): Art. 47 zum Bankkundengeheimnis.

4.  [Swissbanking: Bankgeheimnis und Cloud, Gutachten 2019](https://www.swissbanking.ch/_Resources/Persistent/b/1/3/3/b13339fe5dcd344e715dbe0a899afe3c8c56ed0b/SBVg_Bankgeheimnis_und_Cloud_Gutachten_LLAG_2019_DE.pdf): Rechtsgutachten zur Auslegung von Art. 47 BankG, unter anderem zur Einwilligung des Kunden.

5.  [Bundesgesetz über den Datenschutz (DSG)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): Art. 8 zur Datensicherheit, Art. 24 zur Meldung von Verletzungen der Datensicherheit.

6.  [Datenschutzverordnung (DSV)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): Art. 1 bis 6 mit Grundsätzen und Schutzzielen der Datensicherheit.

7.  [Bundesgesetz über die elektronische Signatur (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): Arten elektronischer Signaturen und Siegel, Anforderungen an qualifizierte Zertifikate.

8.  [Obligationenrecht (OR)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): Art. 14 Abs. 2bis zur Gleichstellung der qualifizierten elektronischen Signatur mit der eigenhändigen Unterschrift, Art. 958f zur Aufbewahrung.

9.  [Zivilprozessordnung (ZPO)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): Art. 157 zur freien Beweiswürdigung, Art. 177 zum Urkundenbegriff.

10.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): Zertifikatsprofile von SwissSign (Stand 18.09.2025), Zuordnung von Silver, Gold und Organization Validated zu den Validierungstypen und Subject-Felder.

11.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): verbindliche Regeln für öffentlich vertrauenswürdige S/MIME-Zertifikate, Definition der Validierungstypen.
