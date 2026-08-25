---
title: "Massenversand-Gateway evaluieren: Lastprofil aus dem Message Tracking, Burst-Test mit JMeter, Live-Blick in die Exchange-Queues"
navTitle: "Burst-Test JMeter"
description: "Bevor ein neues Massenmail-Gateway beschafft wird, muss klar sein, welche Last es tragen soll. Wie Sie das echte Lastprofil aus Message-Tracking-Logs herauslesen, daraus einen realistischen Burst-Test mit JMeter konstruieren und live prüfen, ob Exchange auf dem Weg drosselt."
date: "2026-08-25"
kategorie: "SMTP und Mailflow"
timeToRead: "12 Min. Lesezeit"
themen:
  - "smtp-mailflow"
  - "exchange-onprem-hybrid"
  - "testing"
produkte:
  - "exchange-on-premises"
  - "uebergreifend"
protokolle:
  - "smtp"
  - "testing"
  - "powershell"
  - "troubleshooting"
slug: "massenversand-lastprofil-burst-test-jmeter"
translationId: "article-0c62303db425ce2e"
url: "https://rafaelpfister.ch/blog/massenversand-lastprofil-burst-test-jmeter"
aiPrompt: |
  Du bist mein Mailflow-Assistent. Ich habe einen Message-Tracking-Export (CSV) eines Mail-Gateways. Hilf mir Schritt für Schritt: 1. Die Zustellungen zu deduplizieren (mehrere Logzeilen pro Mail erkennen). 2. Das Lastprofil pro Minute, Stunde und Tag zu berechnen und Bursts zu identifizieren. 3. Daraus ein JMeter-Testdesign mit Precise Throughput Timer abzuleiten, das gegen einen Mail-Sink testet und niemals echte Empfänger anschreibt.
---
# Massenversand-Gateway evaluieren: Lastprofil aus dem Message Tracking, Burst-Test mit JMeter, Live-Blick in die Exchange-Queues

Ein Kundenportal versendet Benachrichtigungen: neue Rechnungen, neue Dokumente, neue Nachrichten. Das bestehende Mail-Gateway soll abgelöst werden, und bei jeder Gateway-Evaluation stellt sich früh die Frage: Welche Last muss das neue System eigentlich tragen? Die Antwort steht nicht im Datenblatt des Herstellers, sondern in den eigenen Logs. Der Weg dorthin, hier an einem anonymisierten realen Fall: das Lastprofil aus dem Message-Tracking-Export lesen (inklusive der zwei Fallen, die in solchen Exporten stecken), daraus einen Burst-Test mit JMeter konstruieren, und während des Tests oder jederzeit im Betrieb live in die Exchange-Queues schauen.

## Schritt 1: Das Lastprofil aus dem Message Tracking lesen

Ausgangsmaterial war ein CSV-Export aus dem Message Tracking des bestehenden Gateways: Zeitstempel, Message-ID, Host, Absender, Empfänger, Betreff, letzter Zustellstatus. Bevor Sie daraus Zahlen ableiten, lohnt sich ein kritischer Blick auf den Export selbst. In diesem Fall steckten gleich zwei Fallen darin.

**Falle 1: Der Export ist gekappt.** Die Datei hatte exakt 50'000 Zeilen, ein klassisches Export-Limit. Der Zeitraum umfasste dadurch nur rund 21 Stunden, und der Schnitt lag ausgerechnet mitten im grössten Versand-Burst. Wer daraus naiv ein Monatsvolumen hochrechnet, liegt beliebig falsch. Für belastbare Monatszahlen braucht es einen Export ohne Limit über mindestens einen vollen Batch-Zyklus.

**Falle 2: Eine Mail ist nicht eine Logzeile.** Jede Nachricht erzeugte zwei Einträge: einen bei der Annahme durch ein zwischengeschaltetes Gateway und einen bei der Zustellung an den Ziel-MTA. Die Mails liefen also eine Schlaufe über eine zweite Instanz, und wer die Logzeilen zählt statt der Nachrichten, überschätzt das Volumen um Faktor zwei. Erst nach Deduplizierung (nur Zeilen mit der finalen Remote-Antwort zählen) stimmen die Raten.

Die Auswertung selbst ist mit wenigen Zeilen Python erledigt. Das Muster ist immer gleich: einlesen, deduplizieren, nach Zeitfenstern aggregieren.

```python
import csv, collections, datetime

per_min = collections.Counter()
per_hour = collections.Counter()
with open("tracking-export.csv", encoding="utf-8") as f:
    reader = csv.reader(f)
    next(reader)
    for row in reader:
        # nur finale Zustellungen zaehlen, nicht die Annahme-Zeile
        if "remote SMTP response '2" not in row[6]:
            continue
        d = datetime.datetime.strptime(row[0][:16], "%Y-%m-%d %H:%M")
        per_min[d.strftime("%Y-%m-%d %H:%M")] += 1
        per_hour[d.strftime("%Y-%m-%d %H")] += 1

print("Top-Minuten:", per_min.most_common(10))
print("Top-Stunden:", per_hour.most_common(10))
```

Das Ergebnis in diesem Fall, dedupliziert auf rund 25'000 echte Nachrichten in 21 Stunden:

| Kennzahl | Wert |
|---|---|
| Durchschnitt über das Fenster | ~20 Mails/min |
| Burst-Peak | ~420 Mails/min, über eine Stunde gehalten |
| Zweiter Batch am Abend | ~180 Mails/min |
| Grundlast tagsüber | 100 bis 500 Mails/h, nachts nahe null |
| Eindeutige Empfänger | ~16'000 in einem einzigen Batch-Lauf, fast exakt eine Mail pro Person |
| Empfänger-Domains | ~42 % Gmail, ~20 % Hotmail/Outlook, ~10 % grösster Schweizer Provider |

Drei Erkenntnisse daraus prägen die ganze Evaluation. Erstens: Nicht das Monatsvolumen dimensioniert das Gateway, sondern der Burst. Sieben Mails pro Sekunde über eine Stunde gehalten ist eine andere Anforderung als dieselbe Menge über den Tag verteilt. Zweitens: Über 60 Prozent des Verkehrs geht an Gmail und Microsoft. Bei solchen Raten entscheiden IP-Reputation und die Rate-Limits der grossen Provider über den Durchsatz, nicht die eigene Hardware. Drittens: Der Batch trifft 16'000 verschiedene Empfänger. Pro-Empfänger-Lookups (Routing, Policy, Verschlüsselungsregeln) sind bei Gateways häufig der eigentliche Engpass, nicht der SMTP-Durchsatz.

## Schritt 2: Den Burst-Test mit JMeter konstruieren

Aus dem Lastprofil folgt das Testdesign direkt. Der Test bildet die drei beobachteten Phasen nach, mit einem 90-Minuten-Gesamtlauf:

| Phase | Zeitfenster | Rate | Bildet ab |
|---|---|---|---|
| Grundlast | 0 bis 90 min durchgehend | 20/min | Transaktionale Einzelmails |
| Burst | Minute 15 bis 75 | 420/min | Rechnungslauf |
| Auslauf | Minute 75 bis 85 | 180/min | Abend-Batch |

Die Grundmechanik eines JMeter-SMTP-Testplans (Sampler, CSV-Testdaten, HTML-Report) ist im Artikel [SMTP-Lasttest mit Apache JMeter in der Praxis](/blog/jmeter-smtp-lasttest-html-report) beschrieben, ein Tool-Vergleich für solche Tests im Artikel [Mail-Lasttests planen](/blog/mail-lasttest-tools-linux-windows-vergleich). Hier geht es um die Konstruktion des Lastprofils. In JMeter entspricht das obige Profil drei Thread Groups mit Scheduler (Startverzögerung und Dauer) und je einem **Precise Throughput Timer**, der die Zielrate unabhängig von der Thread-Zahl exakt hält. Für 420 Mails/min reichen 20 Threads komfortabel; der Timer taktet die Samples, die Threads sind nur Arbeitskapazität.

Die wichtigsten Designentscheidungen liegen aber nicht im Timing, sondern in den Testdaten und den Leitplanken:

**Synthetische Empfänger, niemals echte.** Ein Lasttest darf unter keinen Umständen Kunden anschreiben oder auch nur echte Provider-Domains berühren. Die Lösung: Ein Generator erzeugt 16'000 eindeutige Adressen unter einer Test-Domain und bildet dabei die echte Domain-Verteilung als Subdomains nach (`lt-004711@gmail.sink.example.test`, `lt-000815@hotmail.sink.example.test`). Das Gateway sieht so realistisch viele verschiedene Ziel-Routen, aber jede Mail endet am eigenen Sink. Die eindeutigen Adressen sind kein Selbstzweck: Sie zwingen das Gateway zu 16'000 einzelnen Empfänger-Lookups, genau wie im echten Batch.

**Ein Mail-Sink mit Catch-All als einziges Ziel.** Ein smtp-sink oder Mailpit nimmt alles entgegen und macht die Ankunftszeiten auswertbar. Wenn jede Testmail einen Header wie `X-Loadtest-Sent` mit dem Sende-Zeitstempel trägt, liefert der Sink die End-to-End-Latenz ohne zusätzliches Messverfahren.

**Technische Leitplanken statt Konvention.** Damit ein Konfigurationsfehler nicht doch eine Mail nach draussen bringt: ausgehend Port 25, 465 und 587 auf der Firewall der Testumgebung sperren (einzige Ausnahme: der Sink), auf dem Gateway eine Transport-Regel, die sämtlichen Verkehr zum Sink zwingt, und ein dedizierter Test-Absender statt der produktiven Adresse.

**Gemessen wird auf beiden Seiten.** Auf der JMeter-Seite die Annahme-Latenz (p95 muss über die volle Burst-Stunde stabil bleiben, steigende Latenz heisst: das Gateway staut) und die Fehlerquote (jede 4xx- oder 5xx-Antwort während des Bursts ist ein Finding). Am Sink die Drain-Rate: Kommen hinten auch 420/min an, oder nimmt das Gateway schnell an und staut intern?

Als Abnahmekriterium hat sich das Doppelte des beobachteten Peaks bewährt: 840 Mails/min fehlerfrei, als Reserve für Wachstum und für Batch-Läufe, die im Analysefenster nicht sichtbar waren.

## Die Grenze des SMTP-Samplers: keine stehenden Sessions

Der eingebaute SMTP-Sampler von JMeter hat eine Eigenheit, die man kennen muss: Er baut über JavaMail **pro Mail eine neue Verbindung** samt TLS-Handshake auf und schliesst sie wieder. Eine Keep-Alive-Option gibt es nicht. Ein echter MTA hält Verbindungen dagegen offen und schickt mehrere Mails pro Session.

Für das Gateway-Sizing ist das nicht unbedingt ein Nachteil: Der Test ist härter als die Realität, weil er zusätzlich sieben TLS-Handshakes pro Sekunde erzwingt und damit auch Connection-Rate-Limits des Gateways prüft. Wer zusätzlich das realistische Verhalten messen will, ersetzt den Sampler durch einen JSR223-Sampler, der pro Thread ein Jakarta-Mail-Transport-Objekt offen hält:

```groovy
import jakarta.mail.*
import jakarta.mail.internet.*

def transport = vars.getObject("smtpTransport")
if (transport == null || !transport.isConnected()) {
    def props = new Properties()
    props.put("mail.smtp.host", "gateway-test.example.test")
    props.put("mail.smtp.port", "587")
    props.put("mail.smtp.starttls.enable", "true")
    def session = Session.getInstance(props)
    transport = session.getTransport("smtp")
    transport.connect()
    vars.putObject("smtpTransport", transport)
    vars.putObject("smtpSession", session)
}

def session = vars.getObject("smtpSession")
def msg = new MimeMessage(session)
msg.setFrom(new InternetAddress("loadtest@example.test"))
msg.setRecipient(Message.RecipientType.TO,
    new InternetAddress(vars.get("rcpt")))
msg.setSubject("[LOADTEST] Burst-Phase")
msg.setHeader("X-Loadtest-Sent", String.valueOf(System.currentTimeMillis()))
msg.setContent(vars.get("htmlBody"), "text/html; charset=utf-8")
transport.sendMessage(msg, msg.getAllRecipients())
```

Empfehlung: beide Varianten fahren. Die Differenz der beiden Läufe zeigt, wie teuer der Verbindungsaufbau für das Gateway ist, ohne dass Sie dafür ein separates Messverfahren brauchen.

## Wenn Exchange in der Kette steht: der Puffer und die neue Frage

Im Zielbild dieses Falls liefert die Applikation künftig über Microsoft Exchange ein, und Exchange reicht an das Massenmail-Gateway weiter. Das verändert die Risikolage grundlegend. Ein Gateway ohne eigene Queue ist plötzlich ein kleineres Problem, denn Exchange bringt vollwertige Queues mit: Ein 4xx vom Gateway führt zu Queue-Aufbau und Retry in Exchange, nicht zu Mailverlust in der Applikation. Auch das Session-Thema erledigt sich, Exchange hält Verbindungen offen und bündelt Mails pro Session (SmtpMaxMessagesPerConnection, Standardwert 20).

Die kritische Frage lautet dann nicht mehr "nimmt das Gateway 420/min an?", sondern: **Wie schnell drainiert Exchange Richtung Gateway, und wie lange dauert der Batch-Lauf Ende-zu-Ende?** Exchange glättet den Burst. Wenn das Gateway drosselt, kommt trotzdem jede Mail an, nur später. Ob der Lauf eine Stunde oder drei dauert, ist dann eine Frage des Zustell-SLA, keine Verlustfrage. Das Abnahmekriterium formuliert man entsprechend zeitbasiert: Ein Lauf mit 20'000 Mails ist in maximal 90 Minuten vollständig am Sink, und die Exchange-Queue wächst am Ende des Bursts nicht mehr.

Daraus ergibt sich ein zweistufiges Testdesign: Stufe 1 testet das Gateway isoliert (JMeter direkt, rohe Leistungsgrenze), Stufe 2 den ganzen Pfad über Exchange. Ohne Stufe 1 können Sie bei Problemen in Stufe 2 nicht sagen, ob Exchange oder das Gateway bremst.

## Live in Exchange schauen: drosselt gerade etwas?

Ob Exchange gedrosselt wird, ob es staut und wie schnell die Queues drainieren, zeigt Exchange im laufenden Betrieb, ganz ohne Testaufbau. Der erste Blick gilt den Queues aller Transport-Server:

```powershell
Get-TransportService |
    ForEach-Object { Get-Queue -Server $_.Name } |
    Sort-Object MessageCount -Descending |
    Select-Object Identity, Status, MessageCount, IncomingRate,
        OutgoingRate, Velocity, NextHopDomain, LastError |
    Format-Table -AutoSize
```

Die Lesart der wichtigsten Spalten:

| Signal | Bedeutung |
|---|---|
| `Status Retry` plus 421/451/452 in `LastError` | Die Gegenstelle drosselt Exchange |
| `MessageCount` hoch, `Velocity` nahe null | Es staut, die Queue drainiert nicht |
| `Submission` mit hoher Rate bei Tiefe 0 | Exchange verarbeitet die Last, ohne anzustauen |
| `Shadow`-Queues mit Bestand | Kein Rückstau, das sind Redundanzkopien (Shadow Redundancy) für den Partner-Server |

Im konkreten Fall zeigten die beiden Transport-Server im Normalbetrieb rund 190 Mails/min über die Submission-Queues bei durchgehend leeren Ausgangs-Queues, ohne Retry und ohne Fehler. Das ist die dokumentierte Baseline für den späteren Test, und nebenbei die Antwort auf eine Frage, die der Tracking-Export offen liess: Richtung Smarthost floss weniger als eine Mail pro Minute, der Massenversand lief also am Exchange-Cluster vorbei direkt zum Gateway. Der künftige Pfad über Exchange ist für diese Server Neuland, Faktor 400 auf der Smarthost-Route, und genau deshalb Teil des Testplans.

Ob es in einem vergangenen Batch-Lauf gestaut hat, lässt sich rückwirkend aus dem Message Tracking ablesen. DEFER-Events in nennenswerter Zahl bedeuten Drosselung mit Retry-Zyklen:

```powershell
$s = Get-Date "2026-08-24 15:00"
$e = Get-Date "2026-08-24 16:30"
Get-TransportService |
    ForEach-Object {
        Get-MessageTrackingLog -Server $_.Name -ResultSize Unlimited `
            -Start $s -End $e -Sender "portal@example.test"
    } |
    Group-Object EventId |
    Select-Object Name, Count
```

Die Verweildauer in der Queue pro Mail liefert der Abstand zwischen RECEIVE- und SEND-Event derselben Message-ID. Durchschnittswerte unter etwa 30 Sekunden bedeuten: Exchange hat die Last direkt verarbeitet. Minutenwerte bedeuten: Es hat sich gestaut.

Für die Dauerbeobachtung während eines Testlaufs genügt eine Schleife, die alle fünf Sekunden Queue-Tiefe, Raten und `LastError` als CSV wegschreibt. Zwei Ergänzungen vervollständigen das Bild: Erstens die Back-Pressure-Events 15004 bis 15007 der Quelle MSExchangeTransport im Application-Log, sie zeigen an, wenn Exchange sich wegen eigener Ressourcenknappheit selbst drosselt. Zweitens das Protokoll-Logging am Send Connector (`Set-SendConnector -ProtocolLoggingLevel Verbose`), denn im SmtpSend-Log stehen die konkreten 4xx-Antworten der Gegenstelle im Wortlaut, inklusive Tarpitting-Verzögerungen, die in keiner Queue-Statistik auftauchen.

## Fazit

Eine Gateway-Evaluation ohne eigenes Lastprofil stützt sich allein auf Herstellerangaben. Der Weg über die eigenen Logs kostet einen halben Tag und liefert dafür die drei Zahlen, auf die es ankommt: den Burst-Peak (hier 420 Mails/min über eine Stunde), die Empfänger-Vielfalt pro Lauf (hier 16'000 eindeutige Adressen) und den Domain-Mix (hier über 60 Prozent Richtung Gmail und Microsoft). Daraus wird ein reproduzierbarer JMeter-Test mit klarem Abnahmekriterium, und mit ein paar Zeilen PowerShell sehen Sie jederzeit, ob Exchange auf dem Weg zum Gateway staut oder gedrosselt wird. Achten Sie beim Auswerten der Rohdaten auf die beiden klassischen Fallen: Export-Limits, die mitten im Burst schneiden, und Mehrfach-Logzeilen pro Mail, die das Volumen verdoppeln.

## Quellen

1.  [Apache JMeter: SMTP Sampler](https://jmeter.apache.org/usermanual/component_reference.html#SMTP_Sampler): Referenz des eingebauten SMTP-Samplers; eine Option für stehende Verbindungen existiert nicht.

2.  [Apache JMeter: Precise Throughput Timer](https://jmeter.apache.org/usermanual/component_reference.html#Precise_Throughput_Timer): Timer für exakte Zielraten unabhängig von der Thread-Zahl.

3.  [Microsoft Learn: Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue): Referenz der Queue-Abfrage inklusive der Felder IncomingRate, OutgoingRate und Velocity.

4.  [Microsoft Learn: Back Pressure](https://learn.microsoft.com/en-us/exchange/mail-flow/back-pressure): Ressourcen-Selbstschutz des Transport-Diensts, Event-IDs 15004 bis 15007.

5.  [Microsoft Learn: Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): Referenz des Message Trackings inklusive der Event-Typen RECEIVE, SEND, DELIVER und DEFER.

6.  [Mailpit](https://mailpit.axllent.org/): leichtgewichtiger Mail-Sink mit Catch-All, Web-UI und API für die Auswertung von Ankunftszeiten.
