---
title: "NDR, DSN, Bounce: Unzustellbarkeitsmeldungen richtig unterscheiden"
navTitle: "NDR & Bounces"
description: "NDR, DSN, Bounce, Reject, Backscatter: Die Begriffe rund um fehlgeschlagene Zustellung werden oft synonym verwendet, bezeichnen aber unterschiedliche Dinge. Was die RFCs definieren, wer welche Meldung erzeugt, wie eine DSN aufgebaut ist und warum der Unterschied zwischen Reject und Bounce über Backscatter entscheidet."
date: "2026-08-28"
kategorie: "SMTP und Mailflow"
timeToRead: "10 Min. Lesezeit"
themen:
  - "smtp-mailflow"
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "ndr-dsn-bounce-unterschiede"
translationId: "article-5c5164049a129fa4"
url: "https://rafaelpfister.ch/blog/ndr-dsn-bounce-unterschiede"
aiPrompt: |
  Du bist mein Mailflow-Assistent. Ich füge dir gleich eine Unzustellbarkeitsmeldung (NDR/DSN) ein. Analysiere sie Schritt für Schritt: 1. Welcher Server hat die Meldung erzeugt (Reporting-MTA bzw. Generating server)? 2. Wurde die Mail in der SMTP-Session abgewiesen oder nach Annahme zurückgeschickt? 3. Was bedeuten SMTP-Antwortcode und Enhanced Status Code (RFC 3463) konkret? 4. Liegt die Ursache beim Absender, beim Empfänger oder auf dem Transportweg? 5. Welche nächsten Diagnose-Schritte empfiehlst du?
---
# NDR, DSN, Bounce: Unzustellbarkeitsmeldungen richtig unterscheiden

Eine Mail kommt nicht an, und im Ticket steht wahlweise «Bounce», «NDR», «Mailer-Daemon» oder «Fehlermeldung vom Server». Im Admin-Alltag werden diese Begriffe synonym verwendet, obwohl sie unterschiedliche Dinge bezeichnen: Ein Reject in der SMTP-Session ist keine Rückläufer-Mail, eine Verzögerungsmeldung ist kein Zustellfehler, und eine Lesebestätigung hat mit Unzustellbarkeit gar nichts zu tun. Wer die Begriffe sauber trennt, findet die Ursache schneller, denn jede Meldungsart sagt etwas anderes darüber aus, wo im Transportweg das Problem liegt und wer es beheben kann.

## DSN: der Oberbegriff aus den RFCs

Der formale Oberbegriff heisst Delivery Status Notification (DSN), definiert in den RFCs 3461 bis 3464. Eine DSN ist eine maschinell erzeugte Mail, die den Absender über den Zustellstatus seiner Nachricht informiert. Entscheidend: Eine DSN meldet nicht nur Fehlschläge. Das Feld `Action` im maschinenlesbaren Teil kennt fünf Werte:

| Action | Bedeutung |
|---|---|
| `failed` | Zustellung endgültig fehlgeschlagen; die Mail wird nicht erneut versucht |
| `delayed` | Zustellung verzögert; der Server versucht es weiter |
| `delivered` | Erfolgreich zugestellt (Zustellbestätigung, nur auf explizite Anforderung) |
| `relayed` | An einen Server weitergegeben, der selbst keine DSNs erzeugt |
| `expanded` | An eine Verteilerliste übergeben und aufgefächert |

Die Unzustellbarkeitsmeldung ist also nur ein Spezialfall: eine DSN mit `Action: failed`. Genau diesen Spezialfall nennt Microsoft Non-Delivery Report (NDR). Der Begriff NDR stammt aus der Exchange-Welt, ist aber inzwischen herstellerübergreifend gebräuchlich. Wer präzise sein will: Jeder NDR ist eine DSN, aber nicht jede DSN ist ein NDR.

Die Verzögerungsmeldung (`Action: delayed`) verdient besondere Aufmerksamkeit, weil sie im Support regelmässig als Zustellfehler missverstanden wird. Ein typischer Betreff lautet «Delivery delayed» oder «Zustellung verzögert». Die Mail liegt dann noch in der Queue des sendenden Servers, der es weiter versucht, üblicherweise über ein bis zwei Tage. Erst wenn die Queue-Lebensdauer abläuft, folgt der endgültige NDR. Ein Anwender, der auf eine Verzögerungsmeldung hin die Mail erneut sendet, erzeugt Duplikate, sobald das Zielsystem wieder erreichbar ist.

## Reject oder Bounce: die wichtigste Unterscheidung

Bevor die weiteren Begriffe folgen, gehört die zentrale technische Weiche erklärt, denn an ihr entscheidet sich, welcher Server eine Meldung erzeugt.

**Reject (Abweisung in der Session):** Der empfangende Server lehnt die Mail bereits während der SMTP-Session ab, mit einem 5xx-Antwortcode auf `RCPT TO` oder nach `DATA`. Er nimmt die Mail nie an und erzeugt selbst keine Rückmeldungs-Mail. Die Pflicht, den Absender zu informieren, liegt beim einliefernden Server: Der sendende MTA sieht die 5xx-Antwort und erzeugt daraufhin den NDR für seinen lokalen Benutzer. Der NDR, den der Anwender liest, stammt in diesem Fall vom eigenen Server, zitiert aber die Fehlermeldung der Gegenstelle.

**Bounce (Annahme mit späterem Rückläufer):** Der empfangende Server nimmt die Mail mit `250 OK` an und stellt erst danach fest, dass er sie nicht zustellen kann, etwa weil das Postfach nicht existiert, das Kontingent voll ist oder ein nachgelagerter Server ablehnt. Jetzt trägt er die Verantwortung für die Nachricht und muss selbst eine DSN an den Absender schicken. Diese nachträgliche Rückläufer-Mail ist der Bounce im engeren Sinn.

Für die Fehlersuche ist der Unterschied unmittelbar nutzbar: Steht im NDR der eigene Server als erzeugendes System, wurde die Mail in der Session abgewiesen oder kam gar nie hinaus. Steht ein fremder Server als Absender der Meldung, hat die Gegenseite die Mail zunächst angenommen, und das Problem liegt hinter deren Annahmepunkt, für den Absender unsichtbar.

Aus dem Marketing-Umfeld stammen zwei weitere Bounce-Begriffe, die in keinem RFC stehen: Hard Bounce für endgültige Fehler (5xx, `Action: failed`) und Soft Bounce für temporäre (4xx, `Action: delayed`). Für Mailing-Plattformen ist die Unterscheidung zentral, weil Hard Bounces zur sofortigen Listenbereinigung führen sollten. Technisch sind es dieselben Mechanismen wie oben.

## Die Begriffe im Überblick

| Begriff | Was es ist | Wer erzeugt die Meldung | Standard |
|---|---|---|---|
| DSN | Oberbegriff: Statusmeldung zur Zustellung (failed, delayed, delivered, relayed, expanded) | Der MTA, der die Verantwortung für die Mail trägt | RFC 3461 bis 3464 |
| NDR | DSN mit `Action: failed`; Microsoft-Begriff für die Unzustellbarkeitsmeldung | Sendender MTA (nach Reject) oder empfangender MTA (nach Annahme) | RFC 3464, Microsoft-Doku |
| Reject | 5xx-Abweisung in der laufenden SMTP-Session; keine eigene Mail | Niemand; der sendende MTA formt daraus einen NDR | RFC 5321 |
| Bounce | Rückläufer-Mail nach bereits erfolgter Annahme | Empfangender MTA | RFC 5321, RFC 3464 |
| Hard/Soft Bounce | Marketing-Einteilung: endgültig (5xx) vs. temporär (4xx) | wie Bounce | kein RFC |
| Verzögerungsmeldung | DSN mit `Action: delayed`; Mail liegt noch in der Queue | Sendender oder relayender MTA | RFC 3464 |
| Backscatter | NDRs an gefälschte Absenderadressen, meist ausgelöst durch Spam | Fehlkonfigurierte empfangende MTAs | kein RFC, Anti-Abuse-Begriff |
| MDN / Lesebestätigung | Meldung über die Anzeige oder Löschung durch den Empfänger | Mailclient des Empfängers | RFC 8098 |
| Abwesenheitsnotiz | Automatische Antwort eines erreichten Postfachs | Postfach- bzw. Groupware-Server | RFC 3834 |

## So ist eine DSN aufgebaut

Standardkonforme DSNs verwenden den MIME-Typ `multipart/report; report-type=delivery-status` mit drei Teilen: einer menschenlesbaren Erklärung, einem maschinenlesbaren Teil vom Typ `message/delivery-status` und optional der Originalnachricht oder deren Headern. Der maschinenlesbare Teil ist für die Diagnose der wertvollste, weil seine Felder normiert sind:

```text
Reporting-MTA: dns; mail01.example.net
Received-From-MTA: dns; client.example.org

Final-Recipient: rfc822; max.muster@example.com
Action: failed
Status: 5.1.1
Remote-MTA: dns; mx.example.com
Diagnostic-Code: smtp; 550 5.1.1 <max.muster@example.com>:
    Recipient address rejected: User unknown
```

| Feld | Bedeutung |
|---|---|
| `Reporting-MTA` | Der Server, der diese DSN erzeugt hat; erster Anhaltspunkt für die Zuständigkeit |
| `Final-Recipient` | Die Empfängeradresse, auf die sich der Status bezieht (pro Empfänger ein Block) |
| `Action` | Einer der fünf Statuswerte (failed, delayed, delivered, relayed, expanded) |
| `Status` | Enhanced Status Code nach RFC 3463, z. B. `5.1.1` |
| `Remote-MTA` | Die Gegenstelle, mit der der Reporting-MTA gesprochen hat |
| `Diagnostic-Code` | Die wörtliche SMTP-Antwort der Gegenstelle; oft die aussagekräftigste Zeile |

Eine DSN wird immer mit leerem Envelope-Absender verschickt (`MAIL FROM:<>`). Das ist keine Nachlässigkeit, sondern Vorschrift aus RFC 5321: Der leere Absender verhindert, dass auf eine unzustellbare DSN eine weitere DSN folgt und sich zwei Server endlos Fehlermeldungen zuschicken. Daraus folgt eine Konfigurationsregel: Ein Mailsystem darf Mails mit leerem Envelope-Absender nicht pauschal ablehnen, sonst erreichen legitime Unzustellbarkeitsmeldungen die eigenen Benutzer nie.

Exchange und Exchange Online halten sich beim Format an den Standard, verpacken den Inhalt aber in eine eigene Darstellung: Der Anwender sieht eine aufbereitete Seite mit Klartext-Erklärung, darunter stehen «Generating server» (entspricht `Reporting-MTA`) und die Raw-Angaben. Für die Diagnose lohnt sich immer der Blick in diesen unteren, technischen Teil.

## Enhanced Status Codes lesen

Im Feld `Status` und meist auch im `Diagnostic-Code` steht ein dreiteiliger Code nach RFC 3463: Klasse.Subjekt.Detail. Die Klasse nennt die Verbindlichkeit, Subjekt und Detail die Ursache:

| Code-Bereich | Bedeutung |
|---|---|
| `2.x.x` | Erfolg (nur in Zustellbestätigungen) |
| `4.x.x` | Temporärer Fehler; der Server versucht es erneut |
| `5.x.x` | Endgültiger Fehler; keine weiteren Versuche |
| `x.1.x` | Adressierungsproblem, z. B. `5.1.1` unbekannter Empfänger, `5.1.10` Domain ohne MX |
| `x.2.x` | Postfachproblem, z. B. `5.2.2` Postfach voll, `5.2.3` Nachricht zu gross fürs Postfach |
| `x.3.x` | Problem des Zielsystems, z. B. `4.3.2` System nimmt gerade nichts an |
| `x.4.x` | Netzwerk und Routing, z. B. `4.4.1` keine Antwort, `4.4.7` Queue-Lebensdauer abgelaufen |
| `x.5.x` | Protokollfehler im SMTP-Dialog |
| `x.7.x` | Richtlinie und Sicherheit, z. B. `5.7.1` Relay verweigert oder Richtlinien-Ablehnung, `5.7.26` fehlende Authentifizierung (SPF/DKIM/DMARC) |

Der klassische dreistellige SMTP-Antwortcode (etwa `550`) und der Enhanced Status Code stehen oft gemeinsam in einer Zeile: `550 5.7.1 ...`. Der dreistellige Code steuert das Protokollverhalten des sendenden Servers, der erweiterte Code liefert die Diagnose-Aussage. Bei Widersprüchen zwischen Code und Freitext ist der Freitext der Gegenstelle häufig die genauere Quelle, denn viele Systeme setzen generische Codes und schreiben die eigentliche Ursache in den Kommentar, inklusive Referenz-IDs für den Support der Gegenseite.

Zu beachten: `5.7.x`-Ablehnungen durch Reputations- und Inhaltsfilter sagen oft bewusst wenig. Wer hier nur auf den Code schaut, sucht am falschen Ort; die Blockliste oder der Filterhersteller aus dem Freitext führt schneller zum Ziel.

## Backscatter: die schädliche Sorte Bounce

Backscatter entsteht, wenn ein Server Spam mit gefälschtem Absender zuerst annimmt und anschliessend einen NDR an die gefälschte Adresse schickt. Der NDR trifft damit einen Unbeteiligten, dessen Adresse der Spammer missbraucht hat. Bei grossen Spam-Wellen erhalten die Betroffenen tausende NDRs für Mails, die sie nie versandt haben, und Server, die solche NDRs in Masse erzeugen, landen selbst auf Blocklisten (etwa der Backscatterer-Liste von UCEPROTECT).

Die Abhilfe folgt direkt aus der Reject-Bounce-Unterscheidung: Alles, was ablehnbar ist, gehört in der SMTP-Session abgelehnt, nicht nach Annahme zurückgeschickt. Konkret bedeutet das Empfängervalidierung am äussersten Annahmepunkt (das Edge-Gateway kennt die gültigen Adressen, per Verzeichnisabgleich oder Recipient Callout, statt alles anzunehmen und intern scheitern zu lassen), Spam- und Malware-Ablehnung während der Session statt Quarantäne-NDRs, und der Verzicht auf NDRs für als Spam klassifizierte Nachrichten. Ein Reject erzeugt keinen Backscatter, denn bei gefälschtem Absender läuft die 5xx-Antwort beim Spammer-Server auf, der daraus keinen NDR an das Opfer baut.

## Was keine Unzustellbarkeitsmeldung ist

Drei Meldungsarten landen in Tickets regelmässig im selben Topf, gehören aber nicht dazu:

**MDN (Message Disposition Notification, RFC 8098):** Die Lesebestätigung. Sie wird nicht vom Transportsystem erzeugt, sondern vom Mailclient des Empfängers, und meldet die Anzeige oder Löschung der Nachricht, nicht deren Zustellung. Der MIME-Typ heisst entsprechend `multipart/report; report-type=disposition-notification`. Eine ausbleibende Lesebestätigung sagt nichts über die Zustellung aus; die meisten Clients fragen den Benutzer oder unterdrücken MDNs ganz.

**Abwesenheitsnotizen und Autoresponder (RFC 3834):** Eine Abwesenheitsnotiz beweist das Gegenteil eines Zustellfehlers, denn sie setzt voraus, dass die Mail das Postfach erreicht hat. In Ticket-Schilderungen («ich bekomme eine automatische Antwort, kommt meine Mail an?») lohnt die Rückfrage, welche Meldung genau vorliegt.

**Quarantäne-Benachrichtigungen:** Meldungen wie der Quarantäne-Digest von Microsoft 365 oder eines Gateways informieren den Empfänger über zurückgehaltene Mails. Sie gehen an den Empfänger, nicht an den Absender, und folgen keinem DSN-Standard. Der Absender erhält in diesem Szenario oft gar nichts, was die Fälle erklärt, in denen eine Mail «ohne Fehlermeldung verschwindet».

## Checkliste für die Diagnose

Liegt eine Meldung vor, klären Sie in dieser Reihenfolge:

1. Um welche Art handelt es sich: NDR (`Action: failed`), Verzögerung (`Action: delayed`), MDN, Autoresponder oder Quarantäne-Hinweis? Bei einer Verzögerungsmeldung: warten, nicht erneut senden.
2. Wer hat die Meldung erzeugt (`Reporting-MTA` bzw. «Generating server»)? Der eigene Server bedeutet Reject oder internen Fehler, ein fremder Server bedeutet Annahme mit späterem Fehlschlag auf der Gegenseite.
3. Was sagen Status und Diagnostic-Code? Klasse 4 gegen Klasse 5 trennt temporär von endgültig, das Subjekt (`x.1` Adresse, `x.2` Postfach, `x.4` Netz, `x.7` Richtlinie) grenzt die Ursache ein, der Freitext der Gegenstelle liefert die Details.
4. Fehlt jede Meldung, obwohl die Mail nicht ankommt: Message Tracking auf dem eigenen System prüfen und an Quarantäne oder stille Filterung auf der Gegenseite denken.

Wie sich einzelne Zustellwege danach gezielt nachstellen lassen, zeigen die Beiträge zum [Message Tracking und zur SMTP-Diagnose im Befehls-Generator](https://rafaelpfister.ch/tools/command-builder) sowie der [Mail-Header-Analyzer](https://rafaelpfister.ch/tools/mail-header-analyzer) für die Auswertung des Transportwegs einer angekommenen Mail.

## Quellen

1.  [RFC 3461: SMTP Service Extension for Delivery Status Notifications](https://www.rfc-editor.org/rfc/rfc3461): SMTP-Erweiterung, mit der Absender DSNs anfordern und steuern.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): Definition der dreiteiligen Statuscodes (Klasse.Subjekt.Detail).

3.  [RFC 3464: An Extensible Message Format for Delivery Status Notifications](https://www.rfc-editor.org/rfc/rfc3464): Aufbau der DSN als multipart/report, Felder wie Action, Status und Diagnostic-Code.

4.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Grundregeln zu Antwortcodes, Verantwortungsübergang bei Annahme und leerem Envelope-Absender für Fehlermeldungen.

5.  [RFC 8098: Message Disposition Notification](https://www.rfc-editor.org/rfc/rfc8098): Standard für Lesebestätigungen, zur Abgrenzung von DSNs.

6.  [RFC 3834: Recommendations for Automatic Responses to Electronic Mail](https://www.rfc-editor.org/rfc/rfc3834): Regeln für Autoresponder wie Abwesenheitsnotizen.

7.  [Microsoft Learn: Email non-delivery reports and SMTP errors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/non-delivery-reports-in-exchange-online/non-delivery-reports-in-exchange-online): NDR-Aufbau und Codeliste aus Sicht von Exchange Online.

8.  [UCEPROTECT Backscatterer](https://www.backscatterer.org/): Blockliste für Systeme, die Backscatter erzeugen; erklärt die Listungskriterien.
