---
title: "Das Lastprofil eines Mailservers ermitteln: Bursts, Spitzenraten und Empfängerstruktur aus dem Message Tracking"
navTitle: "Lastprofil ermitteln"
description: "Wie viele Mails pro Minute verarbeitet Ihr Mailserver wirklich, und wie hoch sind die Spitzen? Wie Sie mit PowerShell aus dem Exchange Message Tracking das echte Lastprofil ermitteln: Raten pro Minute und Stunde, Burst-Dauer, Empfängerstruktur, Nachrichtengrössen und die typischen Auswertungsfehler."
date: "2026-08-25"
kategorie: "SMTP und Mailflow"
timeToRead: "9 Min. Lesezeit"
themen:
  - "smtp-mailflow"
  - "exchange-onprem-hybrid"
produkte:
  - "exchange-on-premises"
  - "uebergreifend"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
slug: "mailserver-lastprofil-ermitteln"
translationId: "article-1ff17a188d73e289"
url: "https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln"
aiPrompt: |
  Du bist mein Mailflow-Assistent. Hilf mir Schritt für Schritt, das Lastprofil meines Mailservers zu ermitteln: 1. Die richtige Datenquelle wählen (Message Tracking, Gateway-Logs) und das passende Event pro Nachricht bestimmen. 2. Raten pro Minute, Stunde und Tag berechnen und Bursts mit Dauer und Peak charakterisieren. 3. Empfängerstruktur, Domain-Verteilung und Nachrichtengrössen auswerten. Weise mich auf Doppelzählungen, Export-Limits und Zeitzonen-Fehler hin.
---
# Das Lastprofil eines Mailservers ermitteln: Bursts, Spitzenraten und Empfängerstruktur aus dem Message Tracking

Ob ein Gateway abgelöst, ein Server dimensioniert oder ein Wartungsfenster geplant werden soll: Früher oder später braucht jeder Mail-Admin die Antwort auf die Frage, wie viel sein System eigentlich verarbeitet. Das Bauchgefühl liegt dabei regelmässig daneben, denn Mailverkehr ist selten gleichmässig. Ein System, das im Tagesmittel 20 Mails pro Minute sieht, kann während eines Rechnungslaufs eine Stunde lang 400 pro Minute verarbeiten müssen. Wer nur den Durchschnitt kennt, dimensioniert am eigentlichen Problem vorbei.

Ein brauchbares Lastprofil besteht aus vier Kennzahlen: der Durchschnittsrate (pro Minute, Stunde, Tag), den Bursts (wie hoch ist der Peak, wie lange hält er an, wann tritt er auf), der Empfängerstruktur (wie viele verschiedene Empfänger, welche Ziel-Domains) und den Nachrichtengrössen. Alle vier stehen im Message Tracking, und auf Exchange lassen sie sich mit wenigen Zeilen PowerShell herausrechnen.

## Die Datenquelle: Message Tracking

Exchange protokolliert jede Nachricht im Message Tracking Log. Bevor Sie auswerten, prüfen Sie, wie weit die Daten zurückreichen; der Standard sind 30 Tage, aber ein knappes Grössenlimit kann die reale Aufbewahrung deutlich verkürzen:

```powershell
Get-TransportService |
    Select-Object Name, MessageTrackingLogMaxAge,
        MessageTrackingLogMaxDirectorySize, MessageTrackingLogPath
```

Für ein Lastprofil sollte der Zeitraum mindestens einen vollen Batch-Zyklus des Unternehmens abdecken: Monatsrechnungsläufe, Lohnabrechnungen, Newsletter. Eine Woche ist das Minimum, ein Monat ist besser.

## Rohdaten einsammeln: ein Event pro Nachricht

Die wichtigste Vorentscheidung: Welches Event zählt als "eine Mail"? Das Message Tracking schreibt pro Nachricht mehrere Einträge (RECEIVE bei der Annahme, SEND bei der Weitergabe an den nächsten Hop, DELIVER bei Postfach-Zustellung, dazu AGENTINFO, HAREDIRECT und weitere). Wer einfach alle Zeilen zählt, überschätzt das Volumen um ein Mehrfaches. Für die Einlieferungslast zählen Sie RECEIVE, für die Ausgangslast Richtung Smarthost oder Internet SEND.

```powershell
$start = (Get-Date).AddDays(-7)
$events = Get-TransportService | ForEach-Object {
    Get-MessageTrackingLog -Server $_.Name -ResultSize Unlimited `
        -Start $start -EventId RECEIVE
}
"{0} Nachrichten seit {1:yyyy-MM-dd}" -f $events.Count, $start
```

Die Abfrage läuft bewusst über alle Transport-Server, denn jeder Server protokolliert nur seinen eigenen Anteil. Wer nur einen Server abfragt, sieht bei einem Cluster nur einen Bruchteil der Last.

## Raten pro Minute und Stunde: hier zeigen sich die Bursts

Die Aggregation ist ein Group-Object auf den gerundeten Zeitstempel. Die Top-Minuten sind direkt Ihre Burst-Kandidaten:

```powershell
$proMinute = $events |
    Group-Object { $_.Timestamp.ToString("yyyy-MM-dd HH:mm") } |
    Sort-Object Count -Descending

$proMinute | Select-Object -First 10 Name, Count
```

Dasselbe pro Stunde und als Tagesgang (welche Uhrzeit ist typischerweise wie stark belastet):

```powershell
$events |
    Group-Object { $_.Timestamp.ToString("yyyy-MM-dd HH") } |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count

$events |
    Group-Object { $_.Timestamp.ToString("HH") } |
    Sort-Object Name |
    Format-Table Name, Count
```

Ein Burst ist erst charakterisiert, wenn Sie neben dem Peak auch seine Dauer kennen. Ein Peak von 400/min, der zwei Minuten anhält, ist eine andere Anforderung als derselbe Peak über eine Stunde. Zählen Sie die Minuten über einem Schwellwert:

```powershell
$schwelle = 100
$burstMinuten = $proMinute | Where-Object Count -ge $schwelle
"{0} Minuten mit >= {1}/min, Peak: {2}/min" -f $burstMinuten.Count,
    $schwelle, ($proMinute | Select-Object -First 1).Count
```

Liegen die Burst-Minuten zusammenhängend (im Output von `$burstMinuten | Sort-Object Name` direkt sichtbar), handelt es sich um einen Batch-Lauf. Notieren Sie Startzeit, Dauer und Wiederholmuster, denn genau dieses Fenster muss die Infrastruktur tragen.

## Empfängerstruktur: wie viele Ziele, welche Domains

Für Gateways ist die Empfänger-Vielfalt oft wichtiger als die reine Rate, denn pro Empfänger fallen Lookups an (Routing, Policies, Verschlüsselungsregeln). Eine Mail an einen Verteiler mit 5'000 Mitgliedern belastet anders als 5'000 Einzelmails. Das Feld `RecipientCount` und die Empfängerliste liefern beide Sichten:

```powershell
"Nachrichten: {0}, Empfänger-Zustellungen: {1}" -f $events.Count,
    ($events | Measure-Object RecipientCount -Sum).Sum

$alleEmpfaenger = $events | ForEach-Object { $_.Recipients } |
    ForEach-Object { $_.ToLower() }
"Eindeutige Empfänger: {0}" -f ($alleEmpfaenger | Sort-Object -Unique).Count
```

Die Domain-Verteilung zeigt, wohin der Verkehr fliesst. Dominieren Gmail und Microsoft, entscheiden deren Rate-Limits und die eigene IP-Reputation über den erreichbaren Durchsatz, nicht die eigene Hardware:

```powershell
$alleEmpfaenger |
    ForEach-Object { ($_ -split "@")[1] } |
    Group-Object |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

Und die Gegenrichtung: Welche Absender (Applikationen, Funktionspostfächer) erzeugen die Last überhaupt? Das beantwortet nebenbei die Frage, welche Systeme bei einer Migration mitgedacht werden müssen:

```powershell
$events |
    Group-Object Sender |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

## Nachrichtengrössen: Bytes pro Sekunde statt Mails pro Sekunde

Durchsatzangaben von Gateways beziehen sich oft auf Datenvolumen, nicht auf Nachrichtenzahl. Zwei Systeme mit identischer Mail-Rate unterscheiden sich um Faktor 100, wenn das eine Benachrichtigungen mit 50 KB und das andere Rechnungs-PDFs mit 5 MB versendet. Das Feld `TotalBytes` liefert die Verteilung:

```powershell
$events | Measure-Object TotalBytes -Average -Maximum -Sum |
    Select-Object @{n = "MittelKB"; e = { [math]::Round($_.Average / 1KB) } },
        @{n = "MaxMB"; e = { [math]::Round($_.Maximum / 1MB, 1) } },
        @{n = "TotalGB"; e = { [math]::Round($_.Sum / 1GB, 1) } }
```

Multiplizieren Sie die Burst-Rate mit der Durchschnittsgrösse im Burst-Fenster, und Sie haben die Bandbreiten-Anforderung, die ein neues Gateway oder eine WAN-Strecke tragen muss.

## Live-Raten ohne Tracking: der Blick in die Queues

Für den Moment-Blick (verarbeitet der Server gerade viel, staut etwas?) brauchen Sie kein Tracking, die Queues zeigen es direkt. `IncomingRate` und `OutgoingRate` sind Mails pro Minute, geglättet über die letzten Minuten:

```powershell
Get-TransportService |
    ForEach-Object { Get-Queue -Server $_.Name } |
    Sort-Object MessageCount -Descending |
    Select-Object Identity, Status, MessageCount, IncomingRate,
        OutgoingRate, NextHopDomain, LastError |
    Format-Table -AutoSize
```

Die Lesart: Eine `Submission`-Queue mit hoher Rate bei Tiefe 0 heisst, der Server verarbeitet die Last ohne anzustauen. `MessageCount` hoch bei `OutgoingRate` nahe null heisst Rückstau. `Status Retry` mit einer 4xx-Meldung in `LastError` heisst, die Gegenstelle drosselt. `Shadow`-Queues mit Bestand sind dagegen normal, das sind Redundanzkopien für den Partner-Server, kein Rückstau.

Für eine kontinuierliche Kurve während eines Lastfensters eignet sich der Performance Counter der Transport-Queues, hier eine Minute lang alle fünf Sekunden:

```powershell
Get-Counter "\MSExchangeTransport Queues(_total)\Messages Completed Delivery Per Second" `
    -SampleInterval 5 -MaxSamples 12
```

## Andere Systeme: dasselbe Prinzip mit CSV

Gateways und Appliances liefern statt PowerShell-Objekten meist einen CSV-Export des Trackings. Das Vorgehen bleibt identisch (ein Event pro Mail wählen, nach Zeitfenstern gruppieren), nur das Werkzeug wechselt, zum Beispiel zu Python:

```python
import csv, collections, datetime

per_min = collections.Counter()
with open("tracking-export.csv", encoding="utf-8") as f:
    reader = csv.reader(f)
    next(reader)
    for row in reader:
        if "response '2" not in row[6]:   # nur finale Zustellungen
            continue
        d = datetime.datetime.strptime(row[0][:16], "%Y-%m-%d %H:%M")
        per_min[d.strftime("%Y-%m-%d %H:%M")] += 1

print(per_min.most_common(10))
```

## Die fünf typischen Auswertungsfehler

**Mehrfach-Events pro Mail.** Die häufigste Fehlerquelle: Zeilen zählen statt Nachrichten. Prüfen Sie mit `$events | Group-Object EventId`, was wirklich in Ihrer Datenmenge steckt, und filtern Sie auf genau ein Event pro Nachricht.

**Gekappte Exporte.** Viele Export-Funktionen liefern maximal 10'000 oder 50'000 Zeilen und schneiden dann kommentarlos ab, gern mitten im grössten Burst. Eine verdächtig runde Zeilenzahl ist ein Alarmsignal. Prüfen Sie immer, ob der Zeitraum der Daten dem angeforderten Zeitraum entspricht.

**Gateway-Schlaufen.** Läuft der Mailfluss über eine Zwischenstation (Verschlüsselungs-Gateway, Hygiene-Appliance) und wieder zurück, taucht dieselbe Mail mehrfach im Tracking auf. Deduplizieren Sie über die Message-ID oder filtern Sie auf einen eindeutigen Punkt in der Kette.

**Zeitzonen.** `Get-MessageTrackingLog` liefert Zeitstempel in lokaler Serverzeit, CSV-Exporte von Appliances dagegen oft in UTC. Ein Burst, der scheinbar um 13 Uhr läuft, kann in Wirklichkeit der 15-Uhr-Batch sein. Vor dem Interpretieren die Zeitbasis klären.

**Zu kurze Fenster.** Ein Lastprofil aus zwei ruhigen Tagen ist wertlos, wenn der Monatsrechnungslauf fehlt. Das Auswertefenster muss die bekannten Batch-Zyklen enthalten; fragen Sie die Applikationsverantwortlichen nach ihren Versandplänen, bevor Sie das Fenster festlegen.

## Was Sie mit dem Profil anfangen

Am Ende stehen vier Zahlen auf einer Seite: Durchschnittsrate, Burst (Peak, Dauer, Zeitpunkt, Wiederholmuster), Empfängerstruktur (eindeutige Empfänger pro Lauf, Top-Domains) und Grössenverteilung. Damit lassen sich Gateways dimensionieren, Wartungsfenster in die Nachtstunden mit realer Nulllast legen und Abnahmekriterien formulieren, etwa: Das neue System muss das Doppelte des gemessenen Peaks fehlerfrei verarbeiten. Wie aus einem solchen Profil ein reproduzierbarer Lasttest wird, zeigt der Artikel [SMTP-Lasttest mit Apache JMeter in der Praxis](/blog/jmeter-smtp-lasttest-html-report).

## Quellen

1.  [Microsoft Learn: Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): Referenz der Tracking-Abfrage inklusive aller Felder wie EventId, RecipientCount und TotalBytes.

2.  [Microsoft Learn: Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): Aufbau der Tracking-Logs, Event-Typen und Konfiguration von Aufbewahrung und Verzeichnisgrösse.

3.  [Microsoft Learn: Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue): Referenz der Queue-Abfrage inklusive der Felder IncomingRate, OutgoingRate und Velocity.

4.  [Microsoft Learn: Queues and messages in queues](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues): Queue-Typen, Shadow Redundancy und die Bedeutung der Statuswerte.
