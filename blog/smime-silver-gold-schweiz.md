---
title: "S/MIME mit Banken: Silver- oder Gold-Zertifikat von SwissSign?"
navTitle: "S/MIME Silver/Gold"
description: "Für verschlüsselte Mails mit einer Bank genügt technisch ein Silver-Zertifikat von SwissSign. Gold bestätigt zusätzlich Name und Organisation des Absenders. Welche Stufe was prüft, was im Zertifikat steht und welche rechtlichen Vorgaben in der Schweiz gelten."
date: "2026-11-16"
kategorie: "E-Mail-Verschlüsselung"
timeToRead: "8 Min. Lesezeit"
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
# S/MIME mit Banken: Silver- oder Gold-Zertifikat von SwissSign?

Kurzfazit: Weder ein Schweizer Gesetz noch die FINMA schreibt eine bestimmte Zertifikatsstufe für S/MIME vor. Zum Verschlüsseln genügt ein Silver-Zertifikat technisch vollständig. Gold bestätigt zusätzlich Name und Organisation des Absenders und ist die passende Wahl, wenn die Bank keine Vorgabe macht.

Die Frage, ob für den Mailverkehr mit einer Bank ein Silver- oder ein Gold-Zertifikat von SwissSign nötig ist, kommt in meinen Projekten regelmässig auf. Meist gibt es dazu kein Merkblatt der Bank, und die Kundenberaterin oder der Kundenberater kann die Frage nicht beantworten. Die Antwort hängt davon ab, wofür das Zertifikat eingesetzt wird: für die Verschlüsselung oder als Nachweis, wer eine Mail geschickt hat.

**Schweiz-Fokus:** Dieser Artikel behandelt die Rechtslage in der Schweiz. Wo die EU-Regeln abweichen, steht ein gekennzeichneter Hinweis.

## Was SwissSign bei Silver und Gold prüft

Seit dem 1. September 2023 gelten für öffentlich vertrauenswürdige S/MIME-Zertifikate die `S/MIME Baseline Requirements` des CA/Browser Forum. Sie unterscheiden vier Validierungstypen, je nachdem, was die Zertifizierungsstelle vor der Ausstellung prüft. SwissSign ordnet seine Produkte in den veröffentlichten Zertifikatsprofilen (CPR S/MIME) diesen Typen zu:

| Produkt | Typ nach CA/B Forum | Was geprüft wird | Was im Zertifikat steht |
|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Kontrolle über das Postfach | nur die E-Mail-Adresse |
| E-Mail ID Gold | `Sponsor-validated` | Organisation und Person | Vor- und Nachname (oder Pseudonym), Organisation, Organisationskennung, Land, E-Mail-Adresse |
| Organization Validated | `Organization-validated` | Organisation | Organisationsname, keine Person |

Bei Silver schickt SwissSign einen Bestätigungscode an die beantragte Adresse. Wer die Mail empfangen kann, erhält das Zertifikat. Die Identität der Person wird nicht geprüft, und im Zertifikat steht die Adresse auch als `Common Name`.

Gold ist bei SwissSign ausschliesslich als `Sponsor-validated` geführt. Die Organisation wird geprüft, und die Person ist ihr zugeordnet: Ihre Identität prüft entweder SwissSign oder die Organisation selbst als registrierende Stelle, etwa im Rahmen einer Managed PKI. Ein Profil für Privatpersonen ohne Organisation (`Individual-validated`) ist in den Zertifikatsprofilen nicht aufgeführt.

Für Funktionspostfächer wie `support@` oder `buchhaltung@` ist das Profil `Organization-validated` vorgesehen. Gold-Anträge für solche Postfächer können scheitern, weil SwissSign Gattungsbegriffe wie «Support» oder «Helpdesk» im Feld `GivenName` ablehnt.

Kryptografisch unterscheiden sich die Stufen nicht. Alle Profile verwenden RSA-Schlüssel ab 2048 Bit, die Schlüsselverwendung `digitalSignature` und `keyEncipherment` sowie die erweiterte Verwendung `id-kp-emailProtection`. Eine mit Silver verschlüsselte Mail ist genauso gut geschützt wie eine mit Gold verschlüsselte.

## Was der Unterschied in der Praxis bedeutet

| | Silver | Gold |
|---|---|---|
| Mails verschlüsseln und entschlüsseln | ja | ja |
| Anzeige der Signatur in Outlook | E-Mail-Adresse | Name und Organisation |
| Aussage der Signatur | Absender hat Zugriff auf das Postfach | Absender ist eine geprüfte Person dieser Organisation |
| Aufwand bei der Beantragung | Bestätigung per Mail | Organisationsnachweis und Identitätsprüfung |

Der Unterschied zeigt sich bei einer Kontoübernahme. Wer ein Postfach kontrolliert, kann für diese Adresse ein Silver-Zertifikat beantragen und damit gültig signierte Mails verschicken. Ein Gold-Zertifikat setzt dagegen voraus, dass die Organisation und die Person geprüft sind. Für eine Bank, die etwa Zahlungsaufträge oder Vertragskorrespondenz per Mail entgegennimmt, hat eine Gold-Signatur deshalb mehr Aussagekraft.

Für die Verschlüsselung spielt das keine Rolle. Die Bank braucht nur den öffentlichen Schlüssel des Empfängers, und der ist bei beiden Stufen gleichwertig.

## Rechtliche Vorgaben in der Schweiz

Kein Schweizer Gesetz schreibt S/MIME vor, und keines unterscheidet zwischen Zertifikatsstufen. Es gibt allgemeine Pflichten zum Schutz von Daten, deren technische Umsetzung offen bleibt.

| Regelung | Inhalt | Bezug zu S/MIME |
|---|---|---|
| DSG Art. 8 und DSV Art. 1 bis 6 | Datensicherheit, die dem Risiko angemessen ist | Verschlüsselung ist ein übliches Mittel für heikle Personendaten, ein bestimmtes Verfahren ist nicht vorgeschrieben |
| BankG Art. 47 | Bankkundengeheimnis | verpflichtet die Bank, nicht ihre Kundschaft |
| FINMA-RS 2023/1 | Schutz kritischer Daten bei Banken | technologieneutral, keine Vorgabe zu Zertifikaten |
| ZertES und OR Art. 14 Abs. 2bis | nur die qualifizierte elektronische Signatur ist der eigenhändigen Unterschrift gleichgestellt | Silver und Gold sind keine qualifizierten Signaturen |
| VeÜ-ZSSV | elektronische Eingaben an Gerichte und Betreibungsämter | qualifizierte Signatur und anerkannte Zustellplattform nötig, S/MIME genügt nicht |

### Datenschutz: DSG und DSV

Art. 8 DSG verpflichtet Verantwortliche und Auftragsbearbeiter, eine dem Risiko angemessene Datensicherheit zu gewährleisten. Die Datenschutzverordnung konkretisiert das mit Schutzzielen wie Vertraulichkeit und Integrität, nennt aber keine Verfahren. Ob jemand S/MIME, ein Gateway mit Webmail-Zustellung oder eine Austauschplattform einsetzt, ist eine Abwägung im Einzelfall.

### Bankkundengeheimnis und FINMA

Das Bankkundengeheimnis nach Art. 47 BankG und das FINMA-Rundschreiben 2023/1 «Operationelle Risiken und Resilienz: Banken» richten sich an die Bank. Das Rundschreiben verlangt, dass Banken ihre kritischen Daten über den gesamten Lebenszyklus schützen, auch bei der Übertragung. Wie sie das umsetzen, legen die Banken in ihren eigenen Weisungen fest. Ob Silver akzeptiert wird, ist deshalb eine Frage der internen Policy der jeweiligen Bank.

### Elektronische Signatur: ZertES und OR

Das ZertES unterscheidet die elektronische, die fortgeschrittene, die geregelte und die qualifizierte elektronische Signatur. Nach Art. 14 Abs. 2bis OR ist nur die qualifizierte elektronische Signatur mit qualifiziertem Zeitstempel der eigenhändigen Unterschrift gleichgestellt. Die S/MIME-Zertifikate von SwissSign werden nach den Regeln des CA/Browser Forum und den ETSI-Policies LCP (Silver) bzw. NCP (Gold) ausgestellt und sind keine qualifizierten Zertifikate nach ZertES.

Ein Vertrag, für den die Schriftform vorgeschrieben ist, lässt sich mit einer S/MIME-signierten Mail daher nicht formgültig abschliessen, auch nicht mit Gold. Im Streitfall kann eine Gold-Signatur dennoch als Beweismittel dienen; ihr Gewicht beurteilt das Gericht frei.

### Eingaben an Gerichte und Behörden

Für elektronische Eingaben in Zivil-, Straf- sowie Schuldbetreibungs- und Konkursverfahren verlangt die VeÜ-ZSSV eine qualifizierte elektronische Signatur und die Übermittlung über eine anerkannte Zustellplattform. Eine S/MIME-Mail erfüllt diese Anforderungen nicht.

**EU-Hinweis:** In der EU ist nach Art. 25 Abs. 2 eIDAS ebenfalls nur die qualifizierte elektronische Signatur der handschriftlichen Unterschrift gleichgestellt.

## Wenn die Bank keine Vorgabe macht

Fehlt ein Merkblatt, lässt sich die Lage mit drei Fragen an die Bank klären:

1.  Verlangt die Bank für signierte Mails ein Zertifikat mit geprüfter Identität, oder genügt eine reine E-Mail-Validierung?

2.  Arbeitet die Bank mit einem Verschlüsselungs-Gateway (z. B. SEPPmail oder Kiteworks/Totemomail) und einem Domain-Zertifikat, oder mit persönlichen Zertifikaten ihrer Mitarbeitenden?

3.  Wie stellt die Bank ihre eigenen öffentlichen Schlüssel zur Verfügung?

Eine kurze Anfrage an die Kundenberatung, die an die IT weitergeleitet werden kann:

> Für den verschlüsselten Mailverkehr mit Ihnen beschaffe ich ein S/MIME-Zertifikat. Bitte teilen Sie mir mit, ob Sie ein Zertifikat mit geprüfter Identität (z. B. SwissSign E-Mail ID Gold mit Name und Organisation) voraussetzen oder ob ein Zertifikat mit reiner E-Mail-Validierung (Silver) genügt. Zudem bitte ich um Ihr öffentliches Zertifikat bzw. um die Angabe, ob Sie über ein Gateway mit Domain-Zertifikat arbeiten.

Viele Banken verschicken vertrauliche Mails ohnehin über ein Gateway mit Webmail-Zustellung. Die Empfängerin oder der Empfänger meldet sich dann im Portal an und braucht kein eigenes Zertifikat. Arbeiten beide Seiten mit einem Gateway, genügen oft Domain-Zertifikate oder erzwungenes TLS zwischen den Mailservern, und persönliche Zertifikate sind nicht nötig.

Bleibt die Frage offen, ist Gold für Firmenkorrespondenz die passende Wahl. Gold deckt Verschlüsselung und Signatur ab, und ein späterer Wechsel von Silver auf Gold bedeutet eine neue Beantragung und eine erneute Verteilung des öffentlichen Schlüssels an alle Gegenstellen. Silver reicht, wenn die Bank bestätigt, dass sie nur verschlüsselt und Signaturen nicht auswertet. Für Funktionspostfächer ist ein `Organization-validated`-Zertifikat vorgesehen.

## Quellen

1.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): Zertifikatsprofile von SwissSign (Stand 18.09.2025), Zuordnung von Silver, Gold und Organization Validated zu den Validierungstypen, Subject-Felder und Policy-OIDs.

2.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): verbindliche Regeln für öffentlich vertrauenswürdige S/MIME-Zertifikate, Definition der vier Validierungstypen.

3.  [Bundesgesetz über den Datenschutz (DSG)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): Art. 8 zur Datensicherheit.

4.  [Datenschutzverordnung (DSV)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): Art. 1 bis 6 mit Grundsätzen und Schutzzielen der Datensicherheit.

5.  [Bankengesetz (BankG)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): Art. 47 zum Bankkundengeheimnis.

6.  [FINMA: Rundschreiben](https://www.finma.ch/de/dokumentation/rundschreiben/): Übersicht mit dem Rundschreiben 2023/1 «Operationelle Risiken und Resilienz: Banken», in Kraft seit 1. Januar 2024.

7.  [Bundesgesetz über die elektronische Signatur (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): Arten der elektronischen Signatur und Anforderungen an qualifizierte Zertifikate.

8.  [Obligationenrecht (OR)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): Art. 14 Abs. 2bis zur Gleichstellung der qualifizierten elektronischen Signatur mit der eigenhändigen Unterschrift.
