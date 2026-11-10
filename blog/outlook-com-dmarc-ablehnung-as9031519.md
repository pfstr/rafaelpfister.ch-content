---
title: "Outlook.com lehnt Mails ohne DMARC-Record ab: Fehler AS9031519 und die Folgen in der Schweiz"
navTitle: "Outlook.com: AS9031519"
description: "Seit dem 5. November 2026 weisen Outlook.com, Hotmail und Live Nachrichten von Domains ohne DMARC-Record mit 550 5.7.1 und dem Zusatz AS9031519 ab, auch bei kleinem Versandvolumen. Was Microsoft dokumentiert, was Betreiber gemessen haben, wie Sie die eigene Domain prüfen und was der Mail-Sicherheitsindex bei Schweizer Gemeinden zeigt."
date: "2026-11-10"
kategorie: "SMTP und Mailflow"
timeToRead: "8 Min. Lesezeit"
themen:
  - "smtp-mailflow"
  - "microsoft-365-exchange"
hauptthema: "smtp-mailflow"
produkte:
  - "exchange-online"
protokolle:
  - "mail-auth"
  - "dns"
  - "smtp"
  - "troubleshooting"
related:
  - "dns-records-e-mail-stolpersteine"
  - "microsoft-365-compauth-reason-codes"
slug: "outlook-com-dmarc-ablehnung-as9031519"
translationId: "article-84656f0bf3da2006"
url: "https://rafaelpfister.ch/blog/outlook-com-dmarc-ablehnung-as9031519"
---

# Outlook.com lehnt Mails ohne DMARC-Record ab: Fehler AS9031519 und die Folgen in der Schweiz

Seit dem 5. November 2026 kommen Nachrichten an Postfächer bei Outlook.com, Hotmail, Live und MSN mit einem neuen Fehler zurück, wenn die Absenderdomain keinen DMARC-Record publiziert hat:

```text
550 5.7.1 Message rejected: Message lacks SPF/DKIM/DMARC Auth Records
(AS9031519)
```

Frank Carius hat den Fall am 9. November auf msxfaq.de unter dem Titel «Hotmail und DMARC Zwang» beschrieben. Kurzfazit nach Durchsicht der Primärquellen: Die Ablehnungen sind real und betreffen auch Domains mit wenigen Mails pro Woche. Eine angekündigte Regeländerung von Microsoft gibt es dazu jedoch bisher keine. Die einzige Stellungnahme mit Microsoft-Bezug spricht von einer Dienststörung, deren Ursache untersucht wird. Ein DMARC-Record behebt das Problem in fast allen dokumentierten Fällen, und er ist unabhängig vom Ausgang dieser Störung fällig.

Stand der Angaben: 10. November 2026.

## Was Microsoft dokumentiert

Die Postmaster-Richtlinien von Outlook.com enthalten weiterhin nur die Regel vom 5. Mai 2025: Domains, die mehr als 5000 Nachrichten pro Tag an Outlook.com-Konten senden, müssen SPF und DKIM bestehen und einen DMARC-Record mit mindestens `p=none` publizieren, der über SPF oder DKIM ausgerichtet ist. Nachrichten, die das nicht erfüllen, landen zuerst im Junk-Ordner und können später mit folgendem Fehler abgelehnt werden:

```text
550; 5.7.515 Access denied, sending domain [SendingDomain] does not meet
the required authentication level.
```

Für alle anderen Absender formuliert dieselbe Seite nur eine Empfehlung. Der Zusatz `AS9031519` und der Fehlertext `Message lacks SPF/DKIM/DMARC Auth Records` kommen weder in den Postmaster-Richtlinien noch im Support-Artikel zu 5.7.515 vor.

Zur aktuellen Lage hat sich Microsoft nur in den Microsoft-Q&A-Foren geäussert. Ein als «Microsoft External Staff» gekennzeichneter Moderator schrieb am 9. November, es bestehe eine laufende Störung im Exchange-Dienst bei der Zustellung an Outlook.com, Hotmail.com und Live.com. Sie sei als Service Degradation eingestuft, und Microsoft untersuche die Ursache. Eine Incident-Nummer, eine Ursache oder einen Termin für die Behebung nennt der Beitrag nicht. Ein Blogbeitrag, ein Eintrag in den Postmaster-Richtlinien oder eine öffentliche Service-Health-Meldung liegen nicht vor.

## Was Betreiber gemessen haben

Die belastbarsten Daten stammen von Mailprovidern, die ihre Logs ausgewertet und auf der Mailingliste mailop sowie in eigenen Blogbeiträgen veröffentlicht haben (MailRoute, Mythic Beasts, MXroute, Fastmail, one.com, EasyDMARC). Die Beobachtungen decken sich:

- **Auslöser:** Die Absenderdomain hat keinen DMARC-Record. SPF und DKIM waren in den dokumentierten Fällen gültig und ausgerichtet. Der Fehlertext nennt zwar alle drei Verfahren, abgelehnt wird jedoch wegen des fehlenden DMARC-Records.
- **Volumen:** Betroffen sind auch Domains mit wenigen Nachrichten pro Woche, weit unter der Schwelle von 5000 pro Tag.
- **Empfänger:** Nur die Consumer-Postfächer (outlook.com, hotmail.com, live.com, msn.com und Ländervarianten wie hotmail.de oder outlook.be). Exchange-Online-Tenants als Empfänger lehnen nach den vorliegenden Berichten nicht mit diesem Fehler ab.
- **Unregelmässig:** Nur ein Teil der Nachrichten wird abgewiesen. Ein erneuter Versand derselben Nachricht kann durchgehen.
- **Behebung:** Ein Record mit `v=DMARC1; p=none` genügte in den dokumentierten Fällen. Nach der Publikation dauerte es zwischen wenigen Minuten und rund anderthalb Stunden, bis Outlook.com wieder annahm.

Zwei Auswertungen grenzen den Auslöser weiter ein: Bei one.com wurden nur Domains abgewiesen, deren Abfrage von `_dmarc.<domain>` mit `NXDOMAIN` endet. Domains, bei denen der Name existiert und nur kein TXT-Record vorhanden ist (`NOERROR` ohne Antwort), waren dort nicht betroffen. MXroute fand in einem Log 367 abgewiesene Absenderdomains, 286 davon mit `NXDOMAIN` auf `_dmarc`. Diese Eingrenzung beruht auf Daten einzelner Betreiber und ist nicht abschliessend geklärt.

Seit dem Mittag des 9. November melden einzelne Betroffene, dass die Ablehnungen aufgehört haben. Bestätigt hat Microsoft das nicht.

## Sonderfall Microsoft 365 als Absender

Tenants in Microsoft 365 sind als Absender überdurchschnittlich betroffen. Für eine eigene Domain legt Microsoft 365 keinen DMARC-Record an, und solange DKIM für die Domain nicht aktiviert ist, signiert Exchange Online mit der `onmicrosoft.com`-Domain des Tenants. Der Unzustellbarkeitsbericht verpackt die Ablehnung zusätzlich in einen zweiten Statuscode, der auf SPF verweist:

```text
550 5.7.23 The message was rejected because of Sender Policy Framework
violation -> 550 5.7.1 Message rejected: Message lacks SPF/DKIM/DMARC
Auth Records (AS9031519)
```

Der SPF-Record ist in diesen Fällen in der Regel korrekt. Massgebend ist der innere Fehler nach dem Pfeil.

## Die eigene Domain prüfen

Abgefragt wird der TXT-Record unter `_dmarc.<domain>`:

```bash
dig +short TXT _dmarc.example.com
```

```powershell
Resolve-DnsName -Type TXT -Name _dmarc.example.com
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `+short` | `dig` gibt nur den Inhalt des Records aus |
| `TXT` bzw. `-Type TXT` | Record-Typ der Abfrage |
| `_dmarc.example.com` bzw. `-Name` | abgefragter Name: das Label `_dmarc` vor der Absenderdomain |

</details>

Eine leere Ausgabe bei `dig +short` oder der Fehler `DNS name does not exist` bei `Resolve-DnsName` bedeutet, dass kein DMARC-Record existiert. Für Subdomains gilt: Sendet `news.example.com` und hat selbst keinen Record, zählt nach RFC 7489 der Record der Organisationsdomain `example.com`.

Der [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check) auf dieser Seite prüft DMARC zusammen mit SPF, DKIM und der Ausrichtung in einem Durchgang.

## Sofortmassnahme und der Weg danach

Fehlt der Record, stellt dieser Eintrag die Zustellung an Outlook.com wieder her:

```text
_dmarc.example.com.  3600  IN  TXT  "v=DMARC1; p=none; rua=mailto:dmarc-reports@example.com"
```

`p=none` ändert nichts an der Behandlung Ihrer Mails bei anderen Empfängern und kann deshalb ohne Vorarbeiten publiziert werden. Das `rua`-Tag ist für die Annahme bei Outlook.com nicht erforderlich. Ohne `rua` erhalten Sie jedoch keine Aggregatberichte, und ohne diese Berichte fehlt die Grundlage für jeden weiteren Schritt. Die Adresse muss ein Postfach oder einen Auswertungsdienst erreichen, der die XML-Berichte verarbeitet.

`p=none` schützt die Domain nicht vor Missbrauch. Der Record erfüllt lediglich die Annahmebedingung. Für einen wirksamen Schutz folgen diese Schritte:

1. Alle Systeme erfassen, die mit der Domain senden (Mailserver, Newsletter-Dienst, ERP, Webshop, Scanner).
2. SPF für diese Systeme vollständig und gültig halten, mit höchstens zehn DNS-Lookups.
3. DKIM mit der eigenen Domain aktivieren, in Microsoft 365 pro Domain im Defender-Portal.
4. Die Aggregatberichte einige Wochen auswerten und nicht ausgerichtete Quellen korrigieren.
5. Die Policy auf `p=quarantine` und anschliessend auf `p=reject` anheben.

Die typischen Fehler beim Publizieren dieser Records (doppelte SPF-Records, zu lange DKIM-Schlüssel, automatisch angehängter Zonenname) beschreibt der [Guide für DNS-Admins](https://rafaelpfister.ch/blog/dns-records-e-mail-stolpersteine).

## Die Wirkung in der Schweiz: 152 Gemeinden in fünf Tagen

Der [Mail-Sicherheitsindex Schweiz](https://rafaelpfister.ch/open-data/mail-security-index) prüft jede Nacht die DNS-Records von 3364 Schweizer Organisationen, darunter 2078 Gemeinden. Das [Änderungsprotokoll](https://rafaelpfister.ch/open-data/mail-security-index/aenderungen) zeigt seit dem 6. November eine Welle neu publizierter DMARC-Records. Das Datum ist jeweils das des nächtlichen Abgleichs.

| Abgleich | Organisationen neu mit DMARC-Record | davon Gemeinden |
|---|---|---|
| 19. Oktober bis 5. November | höchstens 5 pro Tag | höchstens 4 pro Tag |
| 6. November | 4 | 4 |
| 7. November | 22 | 21 |
| 8. November | 1 | 1 |
| 9. November | 2 | 2 |
| 10. November | 140 | 124 |

In fünf Tagen haben 169 Organisationen erstmals einen DMARC-Record publiziert, 152 davon sind Gemeinden. Die Zahl der Organisationen ohne DMARC im Index sank von 817 auf 650. Der Zeitpunkt deckt sich mit den Ablehnungen bei Outlook.com. Bei den Gemeinden selbst habe ich die Gründe nicht erfragt.

Die Records haben folgende Merkmale:

- **Policy:** 139 der 152 Gemeinden haben `p=none` gesetzt, fünf `p=quarantine`, drei `p=reject`. Bei fünf besteht der Record nur aus `v=DMARC1` ohne `p=`-Tag und ist damit ungültig.
- **Berichte:** 114 Records enthalten ausschliesslich `v=DMARC1; p=none`. Insgesamt haben 124 der 152 Gemeinden keine `rua`-Adresse eingetragen und erhalten somit keine Berichte.
- **Zentrale Umstellungen:** Die Änderungen treten gebündelt auf. Am 10. November kamen 32 Genfer Gemeinden hinzu, deren Mailserver alle unter `acg-geneve.ch` laufen, dazu 26 Zürcher und 19 Freiburger Gemeinden. Am 7. November waren es acht Nidwaldner und sieben Tessiner Gemeinden. Die Bündelung nach Mailserver und Kanton spricht dafür, dass Informatikdienstleister die Records für alle betreuten Gemeinden in einem Durchgang gesetzt haben.

Für die Zustellung an Outlook.com reichen diese Records. Als Schutz der Gemeindedomains vor gefälschten Absendern wirken sie noch nicht: Ohne `rua` fehlen die Daten, um die Policy später anzuheben. 537 der 2078 Gemeinden im Index haben weiterhin keinen DMARC-Record.

## Einordnung

Ob Microsoft die Ablehnung zurücknimmt oder als Regel bestätigt, ist offen. Für die Planung spielt das eine geringe Rolle: Google und Yahoo verlangen DMARC seit Februar 2024 von Massenversendern, Microsoft seit Mai 2025, und die Postmaster-Richtlinien von Outlook.com empfehlen es allen Absendern. Eine Domain ohne DMARC-Record war in den letzten Tagen bei einem der grössten Mailanbieter nicht mehr zuverlässig zustellbar. Der Record mit `p=none` und `rua` ist in wenigen Minuten publiziert und hat keine Nebenwirkungen auf die Zustellung.

## Quellen

1.  [msxfaq.de: Hotmail und DMARC Zwang](https://www.msxfaq.de/spam/hotmail_dmarc_zwang.htm): Praxisbeitrag von Frank Carius vom 9. November 2026 mit Unzustellbarkeitsbericht und empfohlenem Vorgehen; ohne Verweis auf eine Microsoft-Quelle.

2.  [Outlook.com Postmaster: Policies, Practices, and Guidelines](https://substrate.office.com/ip-domain-management-snds/postmaster/Policies): Offizielle Anforderungen an Absender; nennt die Schwelle von 5000 Nachrichten pro Tag, den Fehler 5.7.515 und die Empfehlung an alle Absender.

3.  [Microsoft Support: Fix NDR error 550 5.7.515 in Outlook.com](https://support.microsoft.com/en-us/outlook/fix-ndr-error-550-5-7-515-in-outlook-com): Support-Artikel zum dokumentierten Fehler für Absender mit hohem Volumen.

4.  [Microsoft Q&A: SPF Validation Error](https://learn.microsoft.com/en-us/answers/questions/6063210/spf-validation-error): Forenthread mit der Moderatoren-Antwort vom 9. November 2026 zur laufenden Störung; Forenbeitrag, keine offizielle Service-Health-Meldung.

5.  [MailRoute: Outlook.com Is Quietly Rejecting Email From Domains Without DMARC](https://mailroute.net/blog/outlook-rejecting-email-without-dmarc): Auswertung der ausgehenden Logs eines Mailproviders ab dem 5. November 2026.

6.  [Mythic Beasts: Microsoft are rejecting your mail](https://www.mythic-beasts.com/blog/2026/11/09/microsoft-are-rejecting-your-mail/): Hoster-Bericht mit über 500 Ablehnungen am ersten Tag und der Behebung über `p=none`.

7.  [mailop: Microsoft requiring DMARC now or just us?](https://www.mail-archive.com/mailop@mailop.org/msg27569.html): Thread der Betreiber-Mailingliste mit den Messungen von MXroute, Fastmail und one.com, darunter die Eingrenzung auf `NXDOMAIN`.

8.  [EasyDMARC: Microsoft Rejecting Emails Without DMARC](https://easydmarc.com/blog/microsoft-rejecting-emails-without-dmarc/): Reproduzierter Test mit gültigem und ausgerichtetem SPF und DKIM, ohne DMARC-Record.

9.  [heise online: Outlook.com: Die Mail ist echt und sie kommt trotzdem nicht an](https://www.heise.de/news/Outlook-com-Die-Mail-ist-echt-und-sie-kommt-trotzdem-nicht-an-11517776.html): Deutschsprachige Zusammenfassung der Berichte vom 9. November 2026.

10.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489): Spezifikation, unter anderem zur Ermittlung des Records über die Organisationsdomain.

11.  [Mail-Sicherheitsindex Schweiz: Änderungsprotokoll](https://rafaelpfister.ch/open-data/mail-security-index/aenderungen): Tägliche Änderungen der DNS-Records von 3364 Schweizer Organisationen; Grundlage der Zahlen zu den Gemeinden.
