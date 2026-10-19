---
title: "Der angegriffene Pensionskassen-Dienstleister hat gar keinen DMARC-Record"
navTitle: "DMARC Pensionskassen"
description: "Der Ende September 2026 angegriffene Pensionskassen-Dienstleister PK Softech hat bis heute keinen DMARC-Record. Eine passive Prüfung von 66 Vorsorgeeinrichtungen und 30 Dienstleistern zeigt, dass das kein Einzelfall ist: besonders die Hersteller der Verwaltungssoftware sind ungeschützt."
date: "2026-10-18"
kategorie: "SMTP und Mailflow"
timeToRead: "9 min to read"
themen:
  - "smtp-mailflow"
produkte: []
protokolle:
  - "mail-auth"
  - "dns"
slug: "dmarc-pensionskassen-schweiz"
url: "https://rafaelpfister.ch/blog/dmarc-pensionskassen-schweiz"
translationId: "article-a34017b39fecf271"
zielgruppe: "entscheider"
aiPrompt: |
  Du bist mein Berater für E-Mail-Sicherheit. Prüfe für meine Domain, ob SPF, DKIM, DMARC und MTA-STS korrekt gesetzt sind, erkläre mir den aktuellen Stand in einfachen Worten und zeige mir Schritt für Schritt, wie ich DMARC gefahrlos von p=none über p=quarantine auf p=reject anhebe.
---

Ende September 2026 wurde die PK Softech AG angegriffen, ein Softwarehersteller, dessen Vorsorgelösung mehrere Schweizer Pensionskassen einsetzen. Über diesen Weg flossen Daten ab. Betroffen sind nach der Berichterstattung unter anderem die Bundespensionskasse Publica, die Pensionskasse der Migros und die Pensionskasse der Stadt Zürich sowie weitere Einrichtungen. Je nach Kasse geht es um Name, AHV-Nummer, Zivilstand, Adresse und IBAN sowie Lohn- und Vorsorgedaten.

Dieser Abfluss ist für sich genommen gravierend. Er betrifft Identifikations- und Finanzdaten eines grossen Teils der arbeitenden Bevölkerung und, über Publica, grosser Teile der Bundesverwaltung. Daten in dieser Menge und Qualität sind nicht nur ein Problem für die einzelne Person. Sie ermöglichen Identitätsmissbrauch, gezielte Täuschung und Erpressung im grossen Stil und sind damit eine Gefahr für die öffentliche Ordnung und die nationale Sicherheit. Die Vorsorgeguthaben selbst blieben unberührt, die Informationsplattform Cybercrimepolice.ch warnte aber bereits vor Betrugsanrufen mit den erbeuteten Daten.

Dazu kommt ein Befund, der für einen Dienstleister dieser Kategorie schwer zu erklären ist: **Die PK Softech AG hat bis heute keinen DMARC-Record.** Ihre Domain trägt einen strikten SPF-Eintrag, aber keine DMARC-Richtlinie (geprüft am 18.10.2026). Damit lässt sich die Absenderadresse des angegriffenen Dienstleisters fälschen, ohne dass eine empfangende Mailplattform dies zuverlässig ablehnt. Nach einem Datenabfluss ist das die denkbar ungünstigste Ausgangslage. Wer Name, Adresse und Bankverbindung einer versicherten Person kennt und zusätzlich eine Mail im Namen des Dienstleisters oder der Kasse fälschen kann, schreibt eine Nachricht, die sich von einer echten kaum unterscheiden lässt, etwa die Aufforderung, die IBAN für die nächste Rentenzahlung zu aktualisieren. Genau gegen diese Fälschung schützt DMARC.

PK Softech ist dabei kein Einzelfall, sondern das deutlichste Beispiel eines Musters. Ich habe geprüft, wie gut Schweizer Vorsorgeeinrichtungen und ihre typischen Dienstleister bei der E-Mail-Authentifizierung aufgestellt sind.

## Wie geprüft wurde

Die Auswertung umfasst 66 Vorsorgeeinrichtungen der zweiten Säule, von den grossen öffentlich-rechtlichen Kassen über die Sammelstiftungen der Versicherer bis zu kleineren Gemeinschaftseinrichtungen. Geprüft wurden ausschliesslich öffentliche DNS-Einträge: SPF, DKIM, DMARC, MTA-STS, TLS-RPT und die MX-Einträge. Das sind dieselben Angaben, die jedes empfangende Mailsystem bei jeder eingehenden Nachricht abfragt.

Die Prüfung war rein passiv. Es wurde nur öffentliches DNS abgefragt, keine Verbindung zu den Systemen der Einrichtungen aufgebaut, keine Anmeldung versucht und keine personenbezogenen Daten erhoben. Die Auswertung bildet nur ab, was aus öffentlichen Quellen ohnehin für jeden ablesbar ist. Konzern-Sammelstiftungen sind auf die Hauptdomain des Trägers zusammengefasst; einzelne angeschlossene Stiftungen können abweichen.

## Das Ergebnis

| Merkmal | Anteil der 66 Einrichtungen |
|---|---|
| SPF vorhanden | 88 % |
| SPF strikt (`-all`) | 59 % |
| DMARC vorhanden | 59 % |
| DMARC `p=reject` (voller Schutz) | 20 % |
| DMARC `p=quarantine` | 26 % |
| DMARC `p=none` (nur Beobachtung) | 14 % |
| kein DMARC | 41 % |
| DKIM erkennbar | 55 % |
| MTA-STS | 2 % |
| TLS-RPT | 5 % |

Drei Punkte stechen heraus.

**Nur 20 Prozent sind mit DMARC `p=reject` wirksam geschützt.** Erst diese Richtlinie weist eine empfangende Mailplattform an, gefälschte Nachrichten tatsächlich abzulehnen. Bei allen anderen landet eine Fälschung im Zweifel im Postfach der versicherten Person.

**41 Prozent haben kein DMARC.** Ihre Absenderdomain lässt sich ohne weiteres für betrügerische Mails verwenden, und die Einrichtung erfährt es nicht einmal, weil ohne DMARC auch die Reporting-Funktion fehlt. Auffällig ist, dass darunter auch Kassen sind, die ihre Mail bei Microsoft 365 betreiben, wo sich DMARC mit geringem Aufwand einrichten lässt.

**DMARC `p=none` ist kein Schutz.** Die 14 Prozent in dieser Gruppe haben eine DMARC-Richtlinie gesetzt, aber auf der Beobachtungsstufe belassen. Das liefert Reports, lehnt aber keine einzige gefälschte Nachricht ab. In der Praxis ist das ein Zwischenschritt, der oft über Monate oder Jahre stehen bleibt.

Dazu kommt ein durchgehender Befund: MTA-STS ist mit 2 Prozent praktisch inexistent. Ohne MTA-STS wird die Transportverschlüsselung eingehender Mail nicht erzwungen, was einen Angreifer mit Zugriff auf den Transportweg begünstigt.

## Alle Datensätze zum Durchklicken

Die folgende Infografik zeigt den geprüften Stand aller 66 Vorsorgeeinrichtungen und 30 Dienstleister. Über die Filter lässt sich nach DMARC-Status, Kategorie und Mail-Provider eingrenzen, und jeder Eintrag lässt sich für die vollständigen DNS-Befunde aufklappen.

<div data-dmarc-infographic></div>

## Zum Vergleich: die grössten Schweizer Unternehmen

Wie die Vorsorgeeinrichtungen im Vergleich zur übrigen Wirtschaft dastehen, zeigt der [Mail-Sicherheitsindex Schweiz](/open-data/mail-security-index). Er prüft mit demselben passiven Verfahren täglich über 500 der grössten Schweizer Unternehmen, mit interaktiver Karte nach Kantonen, Vergleichen nach Branche und Mail-Anbieter und einer Detailseite je Unternehmen. Darunter sind auch die Banken, Versicherer und IT-Dienstleister, mit denen Vorsorgeeinrichtungen zusammenarbeiten.

Der Abstand ist deutlich: Von den 518 Unternehmen im Index nutzen am 18.10.2026 38 Prozent DMARC mit `p=reject`, bei den Vorsorgeeinrichtungen sind es 20 Prozent. Ganz ohne DMARC sind 5 Prozent der Unternehmen, bei den Vorsorgeeinrichtungen 41 Prozent.

## Die Zulieferer sind das grössere Thema

Der Vorfall von Ende September traf nicht eine Kasse, sondern einen gemeinsamen Dienstleister. Deshalb habe ich mit demselben passiven Verfahren 30 typische Zulieferer geprüft: Hersteller von Verwaltungssoftware, Firmen für die ausgelagerte Geschäftsführung, Aktuariat und Consulting, Depotbanken und IT-Dienstleister.

Im Mittel stehen die Dienstleister etwas besser da als die Kassen: 40 Prozent nutzen DMARC mit `p=reject`, gegenüber 20 Prozent bei den Einrichtungen. Entscheidend ist aber die Verteilung über die Kategorien.

| Kategorie | mit DMARC `p=reject` |
|---|---|
| Aktuariat und Consulting | gut |
| Depotbanken | gut |
| IT-Dienstleister | gemischt |
| Ausgelagerte Geschäftsführung | schwach |
| Verwaltungssoftware | am schwächsten |

Die Hersteller der Verwaltungssoftware sind die schwächste Gruppe. Keiner der geprüften Software-Anbieter nutzt DMARC mit voller Wirkung, und der angegriffene Anbieter PK Softech hat gar keinen DMARC-Record. Das ist ausgerechnet die Kategorie, die Vorsorgedaten verarbeitet und über Schnittstellen tief mit den Kassen verbunden ist, und es ist die Kategorie des aktuellen Vorfalls.

Daraus folgt der eigentliche Punkt für die Sorgfalt: Eine Kasse mit `p=reject` ist gut geschützt, solange der Angreifer ihre Domain fälschen will. Kann er stattdessen die Domain eines schlecht geschützten Dienstleisters fälschen, über den ohnehin Kommunikation mit den Versicherten läuft, ist die Mail genauso glaubwürdig. Die Absicherung der eigenen Domain ist deshalb nur der erste Schritt. Der zweite ist, die wichtigsten Dienstleister auf denselben Stand zu verpflichten, am besten vertraglich.

## Was das für den Stiftungsrat bedeutet

Die Verantwortung für die Informationssicherheit liegt beim obersten Organ der Vorsorgeeinrichtung. Ein Datenabfluss löst zudem konkrete Pflichten nach dem revidierten Datenschutzgesetz aus: Eine Verletzung der Datensicherheit ist dem EDÖB zu melden, wenn sie voraussichtlich zu einem hohen Risiko für die betroffenen Personen führt, und die Betroffenen sind zu informieren, soweit es zu ihrem Schutz nötig ist.

E-Mail-Authentifizierung ist in dieser Lage die wirksamste Einzelmassnahme mit dem kleinsten Aufwand. SPF, DKIM und DMARC kosten keine Lizenz und keine neue Software. Sie bestehen aus DNS-Einträgen und einer sauberen Konfiguration der bestehenden Mailplattform. Der Schutz wirkt sofort für jede versicherte Person, ohne dass diese selbst etwas tun muss.

## Was zu tun ist

Für Einrichtungen ohne wirksamen Schutz führt ein klarer Pfad zum Ziel:

1. **SPF prüfen und strikt setzen.** Der SPF-Eintrag listet die berechtigten Versandsysteme und schliesst mit `-all`, damit alle übrigen Absender als nicht autorisiert gelten.
2. **DKIM aktivieren.** Jede ausgehende Nachricht wird kryptografisch signiert. Bei Microsoft 365 und bei den gängigen Mailgateways ist das eine Einstellung, kein Projekt.
3. **DMARC einführen und schrittweise scharf schalten.** Start mit `p=none` und aktivem Reporting, um den legitimen Mailverkehr zu erfassen. Danach `p=quarantine`, zuletzt `p=reject`. Der gesamte Weg dauert je nach Umgebung einige Wochen, nicht Monate.
4. **Reporting auswerten.** Die DMARC-Reports zeigen, wer im Namen der Domain Mail versendet, und decken sowohl vergessene legitime Systeme als auch Missbrauch auf.
5. **MTA-STS und TLS-RPT ergänzen.** Beide erzwingen und überwachen die Transportverschlüsselung der eingehenden Mail.
6. **Die Dienstleister in die Pflicht nehmen.** Software-Anbieter, ausgelagerte Geschäftsführung und IT-Partner sollten denselben Stand bei SPF, DKIM und DMARC nachweisen. Das gehört in die Verträge und in die regelmässige Lieferantenprüfung.

## Fazit

Die Angriffe der jüngsten Zeit zielten auf einen Dienstleister, nicht auf die Kassen selbst. Die Folgeschäden entstehen aber dort, wo eine Absenderdomain ungeschützt ist, bei der Kasse ebenso wie bei ihren Zulieferern. Vier von fünf geprüften Vorsorgeeinrichtungen sind gegen gefälschte Absender nicht vollständig geschützt, zwei von fünf haben gar kein DMARC, und gerade die Hersteller der Verwaltungssoftware schneiden am schlechtesten ab. Die Lücke ist bekannt, günstig zu schliessen und wirkt sofort. Nach einem Datenabfluss, der Name, Adresse und Bankverbindung betrifft, ist sie keine technische Nebensache, sondern Teil der Sorgfaltspflicht des obersten Organs, und zwar entlang der ganzen Lieferkette.

## Quellen

1.  [SRF: Pensionskasse des Bundes, wie konnten heikle Daten bei externer Softwarefirma landen](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen): Hintergrund zum Vorfall, betroffene Datenarten und Rolle des Softwarelieferanten.

2.  [cash.ch (AWP): Angriff auf Publica-Lieferanten, Lohndaten konnten betroffen sein](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247): Publica benennt gegenüber AWP die PK Softech AG als angegriffene Firma.

3.  [Netzwoche: Bundespensionskasse Publica meldet Datenabfluss](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss): Meldung zum Angriff auf den Softwarelieferanten und zum Datenabfluss.

4.  [Mimikama: Schweizer Polizei warnt nach Datenabfluss vor möglichen Anlageanrufen](https://www.mimikama.org/?p=436628): Einordnung der Warnung vor Betrugsanrufen mit den abgeflossenen Daten.

5.  [EDÖB: Meldung von Verletzungen der Datensicherheit](https://www.edoeb.admin.ch/de/meldung-von-verletzungen-der-datensicherheit): Meldepflicht nach revDSG und Vorgehen bei Datensicherheitsverletzungen.
