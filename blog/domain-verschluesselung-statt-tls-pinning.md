---
title: "S/MIME-Domainverschlüsselung statt TLS-Pinning: Sicherheit und Rechtslage in der Schweiz"
navTitle: "Domain statt Pinning"
description: "Warum S/MIME-Domainverschlüsselung den Mailverkehr zwischen zwei Organisationen besser schützt als TLS-Pinning, was ein Angreifer mit einer abgefangenen S/MIME-Nachricht anfangen kann, wie DSG, DSV und Berufsgeheimnisse die Wahl beeinflussen und warum MTA-STS und DANE den Transportschutz übernehmen."
date: "2026-11-16"
kategorie: "E-Mail-Verschlüsselung"
timeToRead: "10 Min. Lesezeit"
themen:
  - "e-mail-verschluesselung"
produkte: []
protokolle:
  - "verschluesselung"
  - "tls"
  - "dns"
hauptthema: "e-mail-verschluesselung"
startseiteAb: "2099-12-31"
slug: "domain-verschluesselung-statt-tls-pinning"
translationId: "article-92db7a4587660dec"
url: "https://rafaelpfister.ch/blog/domain-verschluesselung-statt-tls-pinning"
---
# S/MIME-Domainverschlüsselung statt TLS-Pinning: Sicherheit und Rechtslage in der Schweiz

Kurzfazit: TLS-Pinning schützt die Verbindung zwischen zwei Mailservern, S/MIME-Domainverschlüsselung schützt den Inhalt der Nachricht auf dem ganzen Weg zwischen zwei Organisationen. Muss man sich zwischen beiden entscheiden, ist die Domainverschlüsselung aus technischer und rechtlicher Sicht die bessere Wahl. Den Transportschutz, den das Pinning bisher geleistet hat, übernehmen MTA-STS und DANE.

In einem aktuellen Projekt soll eine bestehende Mailverbindung zwischen zwei Organisationen, die bisher mit erzwungenem TLS und Zertifikats-Pinning abgesichert ist, auf S/MIME-Domainverschlüsselung über einen Secure-Mail-Dienst umgestellt werden. Der Dienst kann für dieselbe Verbindung nicht beides gleichzeitig: Entweder gilt das Pinning oder die Domainverschlüsselung. Die Frage, ob damit Sicherheit verloren geht, ist berechtigt. Die Antwort fällt eindeutig aus, sobald man betrachtet, was die beiden Verfahren jeweils schützen.

**Schweiz-Fokus:** Die rechtliche Einordnung folgt dem Schweizer Recht, insbesondere DSG, DSV und StGB. Wo die EU-Regeln abweichen, steht ein gekennzeichneter Hinweis. Der Artikel ersetzt keine Rechtsberatung im Einzelfall.

## Was TLS-Pinning schützt

Ohne Zusatzmassnahmen ist TLS zwischen Mailservern opportunistisch. Daraus ergeben sich drei Angriffsmöglichkeiten:

- **Downgrade:** Ein Angreifer im Übertragungsweg entfernt das STARTTLS-Angebot aus der SMTP-Sitzung. Der sendende Server stellt dann im Klartext zu.
- **Beliebiges Zertifikat:** Sendende Mailserver prüfen das Zertifikat der Gegenstelle standardmässig nicht. Ein Server mit einem selbst ausgestellten Zertifikat wird akzeptiert.
- **Gefälschter MX:** Wer die DNS-Antwort manipuliert, leitet die Mail auf einen eigenen Server um, der sauberes TLS anbietet.

Pinning schliesst alle drei Lücken für einen bestimmten Partner. Das sendende Gateway erzwingt TLS und akzeptiert nur ein festgelegtes Zertifikat oder einen festgelegten Schlüssel. Der Schutz endet jedoch mit der Verbindung. Auf dem empfangenden Server, in Warteschlangen, Spam- und Virenfiltern, Journaling- und Archivsystemen liegt die Nachricht im Klartext vor. Durchläuft sie mehrere Server, gilt das Pinning nur für den Abschnitt, für den es konfiguriert wurde.

Dazu kommt der Betrieb. Der Pin ist eine manuelle Regel beim Absender. Erneuert der Empfänger sein Zertifikat oder ändert er sein Routing, etwa durch einen neuen Filterdienst oder einen Backup-MX, stoppt die Zustellung, bis die Regel nachgeführt ist. Im Störungsfall wird die Regel deshalb oft vorübergehend gelockert, und nicht immer wieder aktiviert.

## Was S/MIME-Domainverschlüsselung schützt

Bei der Domainverschlüsselung verschlüsselt das Gateway des Absenders die Nachricht mit dem öffentlichen Schlüssel aus dem Domainzertifikat der Empfängerdomain. Erst das Gateway des Empfängers entschlüsselt sie wieder. Alle Systeme dazwischen sehen nur den verschlüsselten Inhalt, unabhängig davon, wie viele Server die Nachricht durchläuft und wo sie zwischengespeichert wird. Zusätzlich signiert das Gateway des Absenders die Nachricht mit seinem eigenen Domainzertifikat.

| Schutz | TLS-Pinning | S/MIME-Domainverschlüsselung |
|---|---|---|
| Mailtext und Anhänge | nur während der Übertragung auf dem gepinnten Abschnitt | auf dem ganzen Weg, auch auf Zwischenservern und in Warteschlangen |
| Betreff, Absender, Empfänger, Datum | während der Übertragung verschlüsselt | lesbar, sofern kein Header-Schutz aktiv ist |
| Envelope, Grösse, Zeitpunkt | während der Übertragung verschlüsselt | lesbar |
| Echtheit des Empfangsservers | ja, ein falscher Server wird abgelehnt | nein, die Mail wird dem Server übergeben, den das MX-Routing liefert |
| Forward Secrecy | ja, bei ECDHE-Schlüsselaustausch | nein |
| Integrität und Herkunft nach der Zustellung | nein | ja, per Signatur |
| Abhängigkeit von Routing und Infrastruktur | hoch, Pin gilt für einen Server bzw. Schlüssel | gering, Schutz hängt am Domainzertifikat |

Pinning hat damit drei Eigenschaften, die die Domainverschlüsselung nicht bietet: Es schützt die Metadaten, es verhindert, dass ein falscher Server die Mail annimmt, und es bietet Forward Secrecy. Für den Inhalt ist die Domainverschlüsselung klar überlegen.

## Was ein Angreifer mit einer abgefangenen S/MIME-Nachricht anfangen kann

Angenommen, ein Angreifer bringt einen eigenen Mailserver in den Übertragungsweg, über gefälschte MX-Einträge, BGP-Hijacking oder ein kompromittiertes Relay.

Folgendes kann er nicht:

- **Inhalt lesen:** Text und Anhänge sind mit dem öffentlichen Schlüssel der Empfängerdomain verschlüsselt. Ohne den privaten Schlüssel des Empfänger-Gateways bleiben sie unlesbar.
- **Inhalt unbemerkt verändern:** Ist die Nachricht signiert und verlangt das Empfänger-Gateway eine gültige Signatur, fällt jede Änderung auf.
- **Nachrichten im Namen des Absenders erzeugen:** Dafür fehlt ihm der private Signaturschlüssel der Absenderdomain.

Folgendes kann er:

- **Metadaten auswerten:** Absender, Empfänger, Zeitpunkt, Grösse und in der Regel der Betreff sind lesbar. Ein Betreff wie «Austrittsbericht Max Muster» ist bereits eine Bekanntgabe von Personendaten.
- **Nachrichten speichern und später entschlüsseln:** S/MIME bietet keine Forward Secrecy. Wird der private Domainschlüssel später kompromittiert, sind alle gespeicherten Nachrichten lesbar. Dasselbe gilt langfristig, falls RSA durch Quantencomputer angreifbar wird.
- **Nachrichten unterdrücken oder verzögern:** Der Absender bemerkt das nicht zwingend. Betroffen ist die Verfügbarkeit, was bei Fristen relevant ist.
- **Ungeschützte Header ändern:** Er kann etwa Betreff oder `Reply-To` umschreiben, um Antworten umzulenken. Der verschlüsselte Inhalt bleibt unverändert.
- **Geheimtext gezielt manipulieren:** Bei Nachrichten, die nur verschlüsselt und nicht signiert sind und AES-CBC verwenden, lässt sich der Geheimtext so verändern, dass Teile des Klartexts nach der Entschlüsselung abfliessen können. Das ist das Prinzip der EFAIL-Angriffe von 2018. Eine verpflichtende Signaturprüfung oder authentisierte Verschlüsselung mit AES-GCM nach S/MIME 4.0 verhindert das.

Ein abgefangener Klartext bei reiner Transportverschlüsselung ist ein vollständiger Datenabfluss. Eine abgefangene S/MIME-Nachricht betrifft im Kern Metadaten und Verfügbarkeit.

## Rechtliche Einordnung

Kein Schweizer Erlass schreibt S/MIME, TLS-Pinning oder ein anderes bestimmtes Verfahren vor. Die Vorgaben sind technologieneutral und legen Schutzziele fest. Die Wahl des Verfahrens muss sich jedoch an diesen Schutzzielen messen lassen und dokumentiert sein.

| Regelung | Inhalt | Bedeutung für die Wahl des Verfahrens |
|---|---|---|
| DSG Art. 8, DSV Art. 1 bis 3 | angemessene Datensicherheit, Schutzziele, Transportkontrolle | Schutz bei der Bekanntgabe, gemessen am Risiko und am Stand der Technik |
| DSG Art. 61 lit. c | Busse bei Verletzung der Mindestanforderungen an die Datensicherheit | persönliche Strafbarkeit der verantwortlichen Person |
| DSG Art. 9 | Bearbeitung durch Auftragsbearbeiter | jedes System mit Zugriff auf den Klartext ist zu berücksichtigen |
| DSG Art. 16 und 17 | Bekanntgabe ins Ausland | Verarbeitung von Klartext auf ausländischen Systemen |
| DSG Art. 24 | Meldung von Verletzungen der Datensicherheit | Risiko für Betroffene hängt davon ab, ob abgeflossene Daten lesbar sind |
| StGB Art. 321, DSG Art. 62, ATSG Art. 33 | Berufsgeheimnis, berufliche Schweigepflicht, Schweigepflicht in den Sozialversicherungen | Inhalt ist vor jeder unbefugten Kenntnisnahme zu schützen |
| ZPO Art. 177 | elektronische Dateien als Urkunden | Signatur stärkt den Nachweis von Herkunft und Unversehrtheit |

### Datensicherheit nach DSG und DSV

Art. 8 DSG verlangt eine dem Risiko angemessene Datensicherheit. Die DSV konkretisiert die Schutzziele Vertraulichkeit, Verfügbarkeit, Integrität und Nachvollziehbarkeit und nennt in Art. 3 als Massnahme unter anderem die Transportkontrolle: Personendaten dürfen bei der Bekanntgabe nicht unbefugt gelesen, kopiert, verändert oder gelöscht werden. Für besonders schützenswerte Personendaten nach Art. 5 lit. c DSG, etwa Gesundheitsdaten oder Daten über Massnahmen der sozialen Hilfe, gelten entsprechend höhere Anforderungen.

TLS-Pinning erfüllt die Transportkontrolle für einen Abschnitt der Übermittlung. Die Nachricht wird jedoch auf mehreren Systemen gespeichert, bevor sie ihr Ziel erreicht, und dort ist sie im Klartext lesbar. Mit der Domainverschlüsselung ist der Inhalt auf der gesamten Strecke zwischen den beiden Organisationen geschützt. Die Signatur deckt zusätzlich das Schutzziel Integrität ab, das TLS nach der Zustellung nicht mehr belegen kann.

### Persönliche Strafbarkeit

Nach Art. 61 lit. c DSG wird auf Antrag mit Busse bis CHF 250'000 bestraft, wer vorsätzlich die Mindestanforderungen an die Datensicherheit nach Art. 8 Abs. 3 DSG nicht einhält. Die Busse richtet sich an die verantwortliche natürliche Person, nicht an das Unternehmen. Eine Inhaltsverschlüsselung nach aktuellem Stand der Technik, die in einer Risikobeurteilung dokumentiert ist, ist für diese Personen der stärkere Nachweis als eine Transportregel für einzelne Server.

**EU-Hinweis:** Die DSGVO verlangt in Art. 32 ebenfalls angemessene technische und organisatorische Massnahmen und nennt die Verschlüsselung ausdrücklich. Bussen nach Art. 83 DSGVO richten sich jedoch gegen das Unternehmen und betragen bei Verstössen gegen Art. 32 bis zu 10 Mio. Euro oder 2 % des weltweiten Jahresumsatzes.

### Auftragsbearbeiter und Bekanntgabe ins Ausland

Jedes System auf dem Weg, das den Klartext verarbeitet, ist rechtlich zu betrachten: Filterdienste, Relays, Cloud-Gateways, Archivierungsdienste. Betreiben Dritte diese Systeme, handelt es sich in der Regel um Auftragsbearbeitung nach Art. 9 DSG, mit den entsprechenden Verträgen und Kontrollpflichten. Stehen die Systeme im Ausland, kommen die Regeln zur Bekanntgabe ins Ausland nach Art. 16 und 17 DSG hinzu.

Mit der Domainverschlüsselung verarbeiten diese Systeme nur Geheimtext. Der Kreis der Stellen, die auf Inhalte zugreifen können, beschränkt sich auf die beiden Gateways. Die Metadaten bleiben jedoch Personendaten, und für sie gelten die genannten Pflichten weiterhin.

### Meldepflicht bei Datensicherheitsverletzungen

Eine Verletzung der Datensicherheit, die voraussichtlich zu einem hohen Risiko für die betroffenen Personen führt, muss nach Art. 24 DSG dem EDÖB gemeldet werden. Fliessen Nachrichten über einen kompromittierten Zwischenserver ab, hängt die Einschätzung davon ab, ob die Inhalte lesbar sind. Sind sie stark verschlüsselt und die Schlüssel nicht betroffen, ist das Risiko für die Betroffenen in der Regel gering, sofern keine sensiblen Angaben im Betreff stehen. Bei reiner Transportverschlüsselung liegen die Inhalte dem Angreifer im Klartext vor.

### Berufsgeheimnisse und Schweigepflichten

Wo Daten unter einem Berufsgeheimnis nach Art. 321 StGB stehen, etwa bei Ärztinnen, Anwälten oder deren Hilfspersonen, oder unter einer gesetzlichen Schweigepflicht wie Art. 33 ATSG im Sozialversicherungsbereich, muss der Inhalt vor jeder unbefugten Kenntnisnahme geschützt sein. Dazu gehören auch die Betreiber von Zwischensystemen. Art. 62 DSG stellt zudem die Verletzung der beruflichen Schweigepflicht für Personen unter Strafe, die nicht unter Art. 321 StGB fallen. Diese Anforderung erfüllt nur eine Verschlüsselung des Inhalts.

### Beweiswert der Signatur

Nach Art. 177 ZPO sind elektronische Dateien Urkunden, deren Beweiskraft das Gericht frei würdigt. Eine gültige S/MIME-Signatur der Absenderdomain belegt, dass die Nachricht von dieser Domain stammt und nach dem Signieren nicht verändert wurde. Das stärkt den Nachweis bei Fristen, Zusagen oder Weisungen. Eine Unterschrift ersetzt sie nicht; die Formvorschriften des OR erfüllt nur eine qualifizierte elektronische Signatur nach ZertES. Mit TLS-Pinning lässt sich nach der Zustellung weder Herkunft noch Unversehrtheit nachweisen.

## MTA-STS und DANE als Ersatz für TLS-Pinning

Fällt das Pinning weg, fehlen die Eigenschaften, die nur der Transportschutz bietet: erzwungenes TLS, ein authentifizierter Empfangsserver und damit der Schutz der Metadaten gegen Mitlesen und Umleiten. Genau diese Eigenschaften liefern MTA-STS und DANE, und zwar als Standard, den der Empfänger einmal für alle Absender veröffentlicht.

**MTA-STS (RFC 8461):** Der Empfänger veröffentlicht unter `https://mta-sts.<domain>/.well-known/mta-sts.txt` eine Policy mit den zulässigen MX-Hostnamen und dem Modus `enforce`. Ein Absender, der MTA-STS unterstützt, ruft die Policy ab, speichert sie und stellt nur noch über TLS mit gültigem Zertifikat an einen der genannten MX zu. Ein gefälschter MX oder ein Server ohne gültiges Zertifikat für den MX-Namen wird abgelehnt.

**DANE (RFC 7672):** Der Empfänger veröffentlicht in seiner DNSSEC-signierten Zone einen TLSA-Eintrag mit dem Hash seines Zertifikats oder Schlüssels. Der Absender prüft das Zertifikat des MX gegen diesen Eintrag. Technisch ist DANE ein Pinning, das der Empfänger selbst im DNS veröffentlicht und mit DNSSEC absichert.

| | TLS-Pinning | MTA-STS | DANE |
|---|---|---|---|
| Gepflegt von | Absender, pro Partner manuell | Empfänger, einmal für alle Absender | Empfänger, einmal für alle Absender |
| Wirkt für | einen konfigurierten Partner | alle Absender mit MTA-STS-Unterstützung | alle Absender mit DANE-Unterstützung |
| Zertifikatserneuerung | Pin muss beim Absender nachgeführt werden | ohne Anpassung, solange das neue Zertifikat gültig ist | ohne Anpassung bei gleichem Schlüssel, sonst neuen TLSA-Eintrag vorab veröffentlichen |
| Vertrauensbasis | exakt dieses Zertifikat bzw. dieser Schlüssel | öffentliche Zertifizierungsstellen | DNSSEC |
| Schwachstelle | Betriebsaufwand, Lockerung bei Störungen | erster Abruf und abgelaufener Cache (Trust on First Use), Fehlausstellung durch eine CA | setzt DNSSEC voraus |
| Fehlerberichte | keine | TLS-RPT (RFC 8460) | TLS-RPT (RFC 8460) |

Für die Dokumentation nach Art. 8 DSG haben MTA-STS und DANE einen weiteren Vorteil: Über TLS-RPT erhält der Empfänger täglich Berichte, ob Zustellungen verschlüsselt erfolgten oder fehlschlugen. Beim Pinning gibt es diese Rückmeldung nicht. Die Einrichtung beider Verfahren beschreibt der Artikel [DNS-Records für E-Mail](https://rafaelpfister.ch/blog/dns-records-e-mail-stolpersteine) im Abschnitt zu MTA-STS, TLS-RPT und DANE.

## Wer MTA-STS und DANE prüfen muss

Prüfen muss immer das System, das die Nachricht direkt an den MX des Empfängers übergibt. Veröffentlichen muss die Einträge der Betreiber des MX.

Bei einem Secure-Mail-Dienst sieht der ausgehende Weg typischerweise so aus:

```text
Eigener Mailserver -> Secure-Mail-Dienst (verschlüsselt, signiert) -> MX des Partners
```

Den letzten Abschnitt hat bisher das Pinning geschützt. Hier muss der Dienst MTA-STS und DANE des Partners prüfen; der eigene Mailserver hat auf diesen Abschnitt keinen Einfluss. Den Abschnitt vom eigenen Mailserver zum Dienst sichern Sie selbst, mit erzwungenem TLS und Zertifikatsprüfung auf dem Connector.

Auf der Empfangsseite hängt es vom MX ab. Zeigt der MX auf den Dienst, muss der Dienst passende Zertifikate bereitstellen, und Sie veröffentlichen MTA-STS- und TLSA-Einträge, die zu seinen Servern passen. Bei DANE setzt das voraus, dass der Dienst Schlüsselwechsel rechtzeitig ankündigt. Zeigt der MX auf das eigene Gateway, etwa Exchange Online, richten Sie MTA-STS und DANE dort ein. Nutzen beide Organisationen denselben Dienst, stellt dieser die Nachricht möglicherweise innerhalb der eigenen Plattform zu; dann gibt es zwischen den beiden Organisationen keinen öffentlichen SMTP-Abschnitt.

Unterstützt der Dienst beim Versand weder MTA-STS noch DANE, bleibt der Inhalt durch S/MIME geschützt. Offen sind dann der Schutz der Metadaten und die Authentifizierung des Empfangsservers auf dem letzten Abschnitt. Dieses Restrisiko gehört in die Risikobeurteilung.

## Restrisiken und Massnahmen

| Restrisiko | Massnahme |
|---|---|
| Personendaten im Betreff sind lesbar | Regel: keine Personendaten im Betreff, schriftlich vereinbart mit dem Partner |
| fehlende Forward Secrecy | privaten Domainschlüssel nicht exportierbar speichern, idealerweise in einem HSM; Zugriff protokollieren |
| Manipulation unsignierter Nachrichten | Signaturpflicht auf beiden Seiten, AES-GCM, sofern beide Gateways es unterstützen |
| Zustellung im Klartext bei Fehlern | Nachrichten bei fehlgeschlagener Verschlüsselung zurückhalten statt im Klartext zustellen |
| Metadaten auf dem Transportweg | MTA-STS im Modus `enforce` und, wo DNSSEC vorhanden ist, DANE; TLS-RPT auswerten |
| abgelaufene Zertifikate | gegenseitige Mitteilung mindestens 30 Tage vor Ablauf, Fingerprints über einen zweiten Kanal abgleichen |

Die Entscheidung für die Domainverschlüsselung und die verbleibenden Risiken gehören in das Datensicherheitskonzept bzw. in die Dokumentation der Bearbeitungstätigkeit.

## Fragen an den Gateway-Dienstleister

1.  Prüft der Dienst beim Versand MTA-STS-Policies im Modus `enforce` und DANE-Einträge der Empfängerdomain?

2.  Von welchen Servern wird zugestellt, und lassen sich TLS-RPT-Berichte oder Zustellprotokolle zur Transportverschlüsselung einsehen?

3.  Unterstützt der Dienst als MX MTA-STS und DANE für eingehende Nachrichten, und wie werden Schlüsselwechsel angekündigt?

4.  Kann der Connector vom eigenen Mailserver zum Dienst mit erzwungenem TLS und Zertifikatsprüfung betrieben werden?

5.  Wird zwischen zwei Kunden des Dienstes über öffentliches SMTP zugestellt oder innerhalb der Plattform?

6.  Welche Algorithmen werden für Verschlüsselung und Signatur verwendet, und wird AES-GCM unterstützt?

## Quellen

1.  [Bundesgesetz über den Datenschutz (DSG)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): Art. 8 zur Datensicherheit, Art. 9 zur Auftragsbearbeitung, Art. 16 und 17 zur Bekanntgabe ins Ausland, Art. 24 zur Meldepflicht, Art. 61 und 62 zu den Strafbestimmungen.

2.  [Datenschutzverordnung (DSV)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): Art. 1 bis 3 mit Grundsätzen, Schutzzielen und technischen und organisatorischen Massnahmen, unter anderem der Transportkontrolle.

3.  [Schweizerisches Strafgesetzbuch (StGB)](https://www.fedlex.admin.ch/eli/cc/54/757_781_799/de): Art. 321 zur Verletzung des Berufsgeheimnisses.

4.  [Bundesgesetz über den Allgemeinen Teil des Sozialversicherungsrechts (ATSG)](https://www.fedlex.admin.ch/eli/cc/2002/510/de): Art. 33 zur Schweigepflicht der Durchführungsorgane.

5.  [Zivilprozessordnung (ZPO)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): Art. 177 zum Urkundenbegriff, einschliesslich elektronischer Dateien.

6.  [RFC 8461: SMTP MTA Strict Transport Security (MTA-STS)](https://www.rfc-editor.org/rfc/rfc8461): Spezifikation der Policy, der Abrufregeln und des Modus `enforce`.

7.  [RFC 7672: SMTP Security via Opportunistic DANE TLS](https://www.rfc-editor.org/rfc/rfc7672): DANE für SMTP, TLSA-Einträge und Prüfregeln für sendende Server.

8.  [RFC 8460: SMTP TLS Reporting](https://www.rfc-editor.org/rfc/rfc8460): Format und Zustellung der TLS-RPT-Berichte.

9.  [RFC 8551: S/MIME Version 4.0 Message Specification](https://www.rfc-editor.org/rfc/rfc8551): aktuelle S/MIME-Spezifikation mit AuthEnvelopedData und AES-GCM.

10.  [EFAIL](https://efail.de/): Beschreibung der 2018 veröffentlichten Angriffe auf unsignierte S/MIME- und OpenPGP-Nachrichten im CBC-Modus.

11.  [Datenschutz-Grundverordnung (DSGVO)](https://eur-lex.europa.eu/legal-content/DE/TXT/?uri=CELEX:32016R0679): Art. 32 zur Sicherheit der Verarbeitung, Art. 83 zu den Geldbussen (EU-Hinweis).
