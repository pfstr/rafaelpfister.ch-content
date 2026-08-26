---
title: "Fastställa en e-postservers belastningsprofil: burstar, toppnivåer och mottagarstruktur från Message Tracking"
navTitle: "Fastställa belastningsprofil"
description: "Hur många e-postmeddelanden per minut hanterar din e-postserver egentligen, och hur höga är topparna? Så fastställer du den verkliga belastningsprofilen från Exchange Message Tracking med PowerShell: frekvenser per minut och timme, burst-varaktighet, mottagarstruktur och meddelandestorlekar. Med de vanligaste analysfällorna."
date: "2026-08-25"
kategorie: "SMTP och e-postflöde"
timeToRead: "9 min. lästid"
themen:
  - smtp-mailflow
  - exchange-onprem-hybrid
produkte:
  - "exchange-on-premises"
  - "uebergreifend"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
slug: "faststalla-en-e-postservers-belastningsprofil-burstar-toppnivaer-och-mottagarstruktur-fran"
translationId: "article-1ff17a188d73e289"
aiPrompt: |
  Du bist mein Mailflow-Assistent. Hilf mir Schritt für Schritt, das Lastprofil meines Mailservers zu ermitteln: 1. Die richtige Datenquelle wählen (Message Tracking, Gateway-Logs) und das passende Event pro Nachricht bestimmen. 2. Raten pro Minute, Stunde und Tag berechnen und Bursts mit Dauer und Peak charakterisieren. 3. Empfängerstruktur, Domain-Verteilung und Nachrichtengrössen auswerten. Weise mich auf Doppelzählungen, Export-Limits und Zeitzonen-Fallen hin.
translationOf: mailserver-lastprofil-ermitteln
url: https://rafaelpfister.ch/sv/blog/faststalla-en-e-postservers-belastningsprofil-burstar-toppnivaer-och-mottagarstruktur-fran
translationSourceHash: 16095cf53ce6f67abe31387ce2f02958eacc3898d3a42b61ad8c7b885ab7ce5d
translationModel: gpt-5.6-terra
translatedAt: 2026-08-26T04:10:48.718Z
translationReview: automatic
---

# Fastställa en e-postservers belastningsprofil: burstar, toppnivåer och mottagarstruktur från Message Tracking

Oavsett om en gateway ska bytas ut, en server dimensioneras eller ett underhållsfönster planeras behöver varje e-postadministratör förr eller senare svaret på frågan om hur mycket systemet faktiskt hanterar. Magkänslan har regelbundet fel, eftersom e-posttrafik sällan är jämn. Ett system som i genomsnitt hanterar 20 e-postmeddelanden per minut under dagen kan behöva behandla 400 per minut under en timme vid en fakturakörning. Den som bara känner till genomsnittet dimensionerar förbi det egentliga problemet.

En användbar belastningsprofil består av fyra nyckeltal: genomsnittlig takt (per minut, timme, dag), burstarna (hur hög är toppen, hur länge varar den, när inträffar den), mottagarstrukturen (hur många olika mottagare, vilka måldomäner) och meddelandestorlekarna. Alla fyra finns i Message Tracking, och i Exchange kan de beräknas med några få rader PowerShell.

## Datakällan: Message Tracking

Exchange loggar varje meddelande i Message Tracking Log. Innan du analyserar bör du kontrollera hur långt tillbaka data sträcker sig; standardvärdet är 30 dagar, men en snäv storleksgräns kan förkorta den faktiska lagringstiden avsevärt:

```powershell
Get-TransportService |
    Select-Object Name, MessageTrackingLogMaxAge,
        MessageTrackingLogMaxDirectorySize, MessageTrackingLogPath
```

För en belastningsprofil bör perioden omfatta minst en fullständig batchcykel i företaget: månatliga fakturakörningar, löneutbetalningar, nyhetsbrev. En vecka är minimum, en månad är bättre.

## Samla in rådata: en händelse per meddelande

Det viktigaste inledande valet: Vilken händelse räknas som "ett e-postmeddelande"? Message Tracking skriver flera poster per meddelande (RECEIVE vid mottagning, SEND vid vidarebefordran till nästa hopp, DELIVER vid leverans till postlådan samt AGENTINFO, HAREDIRECT och fler). Den som bara räknar alla rader överskattar volymen mångfalt. För inleveransbelastningen räknar du RECEIVE, för utgående belastning mot Smarthost eller internet SEND.

```powershell
$start = (Get-Date).AddDays(-7)
$events = Get-TransportService | ForEach-Object {
    Get-MessageTrackingLog -Server $_.Name -ResultSize Unlimited `
        -Start $start -EventId RECEIVE
}
"{0} Nachrichten seit {1:yyyy-MM-dd}" -f $events.Count, $start
```

Frågan körs medvetet mot alla transportservrar, eftersom varje server bara loggar sin egen andel. Den som bara frågar en server ser bara en bråkdel av belastningen i ett kluster.

## Takt per minut och timme: här syns burstarna

Aggregeringen är ett Group-Object på den avrundade tidsstämpeln. Toppminuterna är direkt dina burst-kandidater:

```powershell
$proMinute = $events |
    Group-Object { $_.Timestamp.ToString("yyyy-MM-dd HH:mm") } |
    Sort-Object Count -Descending

$proMinute | Select-Object -First 10 Name, Count
```

Samma sak per timme och som dygnsprofil (vilken tid på dygnet är normalt hur hårt belastad):

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

En burst är först karakteriserad när du, utöver toppen, även känner till dess varaktighet. En topp på 400/minut som varar i två minuter ställer andra krav än samma topp under en timme. Räkna minuterna över ett tröskelvärde:

```powershell
$schwelle = 100
$burstMinuten = $proMinute | Where-Object Count -ge $schwelle
"{0} Minuten mit >= {1}/min, Peak: {2}/min" -f $burstMinuten.Count,
    $schwelle, ($proMinute | Select-Object -First 1).Count
```

Om burst-minuterna ligger sammanhängande (syns direkt i utdata från `$burstMinuten | Sort-Object Name`) rör det sig om en batchkörning. Notera starttid, varaktighet och upprepningsmönster, eftersom det är just detta fönster som infrastrukturen måste klara.

## Mottagarstruktur: hur många mål, vilka domäner

För gateways är mottagarmångfalden ofta viktigare än den rena takten, eftersom uppslag sker per mottagare (routning, policyer, krypteringsregler). Ett e-postmeddelande till en distributionslista med 5'000 medlemmar belastar annorlunda än 5'000 enskilda e-postmeddelanden. Fältet `RecipientCount` och mottagarlistan ger båda perspektiven:

```powershell
"Nachrichten: {0}, Empfänger-Zustellungen: {1}" -f $events.Count,
    ($events | Measure-Object RecipientCount -Sum).Sum

$alleEmpfaenger = $events | ForEach-Object { $_.Recipients } |
    ForEach-Object { $_.ToLower() }
"Eindeutige Empfänger: {0}" -f ($alleEmpfaenger | Sort-Object -Unique).Count
```

Domänfördelningen visar vart trafiken går. Om Gmail och Microsoft dominerar avgör deras rate limits och det egna IP-ryktet den möjliga genomströmningen, inte den egna hårdvaran:

```powershell
$alleEmpfaenger |
    ForEach-Object { ($_ -split "@")[1] } |
    Group-Object |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

Och i motsatt riktning: Vilka avsändare (applikationer, funktionspostlådor) skapar egentligen belastningen? Detta besvarar samtidigt frågan om vilka system som måste tas med i beräkningen vid en migrering:

```powershell
$events |
    Group-Object Sender |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

## Meddelandestorlekar: byte per sekund i stället för e-postmeddelanden per sekund

Gatewayers uppgifter om genomströmning avser ofta datavolym, inte antal meddelanden. Två system med identisk e-posttakt skiljer sig med en faktor 100 om det ena skickar aviseringar på 50 KB och det andra faktura-PDF:er på 5 MB. Fältet `TotalBytes` ger fördelningen:

```powershell
$events | Measure-Object TotalBytes -Average -Maximum -Sum |
    Select-Object @{n = "MittelKB"; e = { [math]::Round($_.Average / 1KB) } },
        @{n = "MaxMB"; e = { [math]::Round($_.Maximum / 1MB, 1) } },
        @{n = "TotalGB"; e = { [math]::Round($_.Sum / 1GB, 1) } }
```

Multiplicera burst-takten med den genomsnittliga storleken i burst-fönstret, så får du bandbreddskravet som en ny gateway eller en WAN-länk måste klara.

## Realtidstakter utan tracking: en titt i köerna

För en ögonblicksbild (hanterar servern just nu mycket, byggs något upp?) behöver du ingen tracking; köerna visar det direkt. `IncomingRate` och `OutgoingRate` är e-postmeddelanden per minut, utjämnade över de senaste minuterna:

```powershell
Get-TransportService |
    ForEach-Object { Get-Queue -Server $_.Name } |
    Sort-Object MessageCount -Descending |
    Select-Object Identity, Status, MessageCount, IncomingRate,
        OutgoingRate, NextHopDomain, LastError |
    Format-Table -AutoSize
```

Tolkningen: En `Submission`-kö med hög takt vid djup 0 betyder att servern hanterar belastningen utan att bygga upp en kö. `MessageCount` hög medan `OutgoingRate` ligger nära noll betyder eftersläpning. `Status Retry` med ett 4xx-meddelande i `LastError` betyder att motparten begränsar takten. `Shadow`-köer med innehåll är däremot normala; de är redundanskopior för partnerservern, inte eftersläpning.

För en kontinuerlig kurva under ett belastningsfönster passar transportköernas Performance Counter, här under en minut var femte sekund:

```powershell
Get-Counter "\MSExchangeTransport Queues(_total)\Messages Completed Delivery Per Second" `
    -SampleInterval 5 -MaxSamples 12
```

## Andra system: samma princip med CSV

Gateways och apparater levererar vanligtvis en CSV-export av trackingen i stället för PowerShell-objekt. Tillvägagångssättet är identiskt (välj en händelse per e-postmeddelande, gruppera efter tidsfönster), bara verktyget byts ut, till exempel mot Python:

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

## De fem klassiska analysfällorna

**Flera händelser per e-postmeddelande.** Den vanligaste felkällan: att räkna rader i stället för meddelanden. Kontrollera med `$events | Group-Object EventId` vad som faktiskt finns i din datamängd och filtrera på exakt en händelse per meddelande.

**Avkapade exporter.** Många exportfunktioner levererar högst 10'000 eller 50'000 rader och kapar sedan utan kommentar, gärna mitt i den största bursten. Ett misstänkt runt radantal är en varningssignal. Kontrollera alltid om dataperioden motsvarar den begärda perioden.

**Gateway-loopar.** Om e-postflödet går via en mellanliggande station (krypteringsgateway, hygienapparat) och tillbaka, visas samma e-postmeddelande flera gånger i trackingen. Deduplikera med Message-ID eller filtrera på en entydig punkt i kedjan.

**Tidszoner.** `Get-MessageTrackingLog` levererar tidsstämplar i serverns lokala tid, medan CSV-exporter från apparater ofta är i UTC. En burst som till synes körs klockan 13 kan i själva verket vara batchen klockan 15. Klargör tidsbasen innan du tolkar resultaten.

**För korta fönster.** En belastningsprofil från två lugna dagar är värdelös om den månatliga fakturakörningen saknas. Analysfönstret måste innehålla de kända batchcyklerna; fråga applikationsansvariga om deras sändningsplaner innan du fastställer fönstret.

## Vad du gör med profilen

I slutändan finns fyra siffror på en sida: genomsnittlig takt, burst (topp, varaktighet, tidpunkt, upprepningsmönster), mottagarstruktur (unika mottagare per körning, toppdomäner) och storleksfördelning. Med detta går det att dimensionera gateways, lägga underhållsfönster till nattimmarna med verklig nollbelastning och formulera acceptanskriterier, till exempel: Det nya systemet måste kunna hantera dubbelt så hög belastning som den uppmätta toppen utan fel. Artikeln [SMTP-belastningstest med Apache JMeter i praktiken](/blog/jmeter-smtp-lasttest-html-report) visar hur ett sådant profil blir ett reproducerbart belastningstest.

## Källor

1.  [Microsoft Learn: Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): Referens för trackingfrågan, inklusive alla fält som EventId, RecipientCount och TotalBytes.

2.  [Microsoft Learn: Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): Struktur för trackingloggarna, händelsetyper och konfiguration av lagringstid och katalogstorlek.

3.  [Microsoft Learn: Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue): Referens för köfrågan, inklusive fälten IncomingRate, OutgoingRate och Velocity.

4.  [Microsoft Learn: Queues and messages in queues](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues): Kötyper, Shadow Redundancy och betydelsen av statusvärden.
