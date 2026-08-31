---
title: "Fastställa en e-postservers belastningsprofil: burstar, topphastigheter och mottagarstruktur från Message Tracking"
navTitle: "Fastställa belastningsprofil"
description: "Hur många e-postmeddelanden per minut behandlar din e-postserver egentligen, och hur höga är topparna? Så fastställer du den verkliga belastningsprofilen från Exchange Message Tracking med PowerShell: hastigheter per minut och timme, burstlängd, mottagarstruktur, meddelandestorlekar och de vanligaste analysfelen."
date: "2026-08-25"
kategorie: "SMTP och e-postflöde"
timeToRead: "9 min lästid"
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
translationSourceHash: 298fabdf5f8f248539ea8a119681be130cd76f5c8ebc35db5d0c61e1126251b0
translationModel: gpt-5.6-terra
translatedAt: 2026-08-30T09:21:16.001Z
translationReview: required
url: https://rafaelpfister.ch/sv/blog/faststalla-en-e-postservers-belastningsprofil-burstar-toppnivaer-och-mottagarstruktur-fran
---

# Fastställa en e-postservers belastningsprofil: burstar, topphastigheter och mottagarstruktur från Message Tracking

Oavsett om en gateway ska bytas ut, en server dimensioneras eller ett underhållsfönster planeras: förr eller senare behöver varje e-postadministratör svaret på frågan om hur mycket systemet egentligen behandlar. Magkänslan har regelbundet fel, eftersom e-posttrafik sällan är jämn. Ett system som i dagsgenomsnitt tar emot 20 e-postmeddelanden per minut kan behöva behandla 400 per minut under en faktureringskörning i en timme. Den som bara känner till genomsnittet dimensionerar förbi det egentliga problemet.

En användbar belastningsprofil består av fyra nyckeltal: genomsnittshastigheten (per minut, timme, dag), burstarna (hur hög är toppen, hur länge varar den, när inträffar den), mottagarstrukturen (hur många olika mottagare, vilka måldomäner) och meddelandestorlekarna. Alla fyra finns i Message Tracking, och i Exchange kan de räknas fram med några få rader PowerShell.

## Datakällan: Message Tracking

Exchange loggar varje meddelande i Message Tracking Log. Innan du analyserar bör du kontrollera hur långt tillbaka data finns; standardvärdet är 30 dagar, men en snäv storleksgräns kan avsevärt förkorta den faktiska lagringstiden:

```powershell
Get-TransportService |
    Select-Object Name, MessageTrackingLogMaxAge,
        MessageTrackingLogMaxDirectorySize, MessageTrackingLogPath
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `Get-TransportService` | Listar organisationens alla transportservrar; utan parametrar alla servrar |
| `Select-Object Name, MessageTrackingLog…` | Begränsar utdata till de angivna egenskaperna: lagringstid, storleksgräns för loggkatalogen och loggsökväg |

</details>

För en belastningsprofil bör perioden omfatta minst en fullständig batchcykel i företaget: månatliga faktureringskörningar, löneutbetalningar, nyhetsbrev. En vecka är minimum, en månad är bättre.

## Samla in rådata: ett event per meddelande

Det viktigaste förhandsbeslutet är: Vilket event räknas som ”ett e-postmeddelande”? Message Tracking skriver flera poster per meddelande (RECEIVE vid mottagande, SEND vid vidarebefordran till nästa hopp, DELIVER vid leverans till postlådan, samt AGENTINFO, HAREDIRECT och fler). Den som bara räknar alla rader överskattar volymen flera gånger om. För inlämningsbelastningen räknar du RECEIVE, för utgående belastning mot Smarthost eller internet SEND.

```powershell
$start = (Get-Date).AddDays(-7)
$events = Get-TransportService | ForEach-Object {
    Get-MessageTrackingLog -Server $_.Name -ResultSize Unlimited `
        -Start $start -EventId RECEIVE
}
"{0} Nachrichten seit {1:yyyy-MM-dd}" -f $events.Count, $start
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-Server $_.Name` | Frågar ut trackingloggen för respektive transportserver från pipelinen |
| `-ResultSize Unlimited` | Tar bort standardgränsen på 1 000 returnerade poster |
| `-Start $start` | Nedre tidsgräns för frågan; här de senaste sju dagarna |
| `-EventId RECEIVE` | Filtrerar till exakt ett event per meddelande, här mottagandet av transporttjänsten |
| `-f` | Formatoperator: placerar värdena till höger i platshållarna `{0}` och `{1}` i strängen |

</details>

Frågan körs medvetet mot alla transportservrar, eftersom varje server bara loggar sin egen andel. Den som bara frågar en server ser i ett kluster bara en bråkdel av belastningen.

## Hastigheter per minut och timme: här syns burstarna

Aggregeringen sker med Group-Object på den avrundade tidsstämpeln. Toppminuterna är direkt dina burstkandidater:

```powershell
$proMinute = $events |
    Group-Object { $_.Timestamp.ToString("yyyy-MM-dd HH:mm") } |
    Sort-Object Count -Descending

$proMinute | Select-Object -First 10 Name, Count
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `Group-Object { … }` | Grupperar efter returvärdet från skriptblocket, här tidsstämpeln avkortad till minut |
| `Sort-Object Count -Descending` | Sorterar grupperna fallande efter antal; de mest belastade minuterna visas överst |
| `Select-Object -First 10 Name, Count` | Visar bara de tio största grupperna, begränsat till minut och antal |

</details>

Samma sak per timme och som dygnsprofil (vilken tid på dygnet är vanligtvis hur hårt belastad):

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

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `Group-Object { … ToString("yyyy-MM-dd HH") }` | Grupperar på hela timmar under en specifik dag |
| `Group-Object { … ToString("HH") }` | Grupperar endast efter klockslag och aggregerar därmed över alla dagar: dygnsprofilen |
| `Sort-Object Count -Descending` | De mest belastade timmarna överst |
| `Sort-Object Name` | Sorterar dygnsprofilen kronologiskt efter klockslag i stället för efter antal |
| `Format-Table Name, Count` | Tabellutskrift av de båda kolumnerna |

</details>

En burst är först karakteriserad när du utöver toppen även känner till dess varaktighet. En topp på 400/min som varar i två minuter ställer andra krav än samma topp under en timme. Räkna minuterna över ett tröskelvärde:

```powershell
$schwelle = 100
$burstMinuten = $proMinute | Where-Object Count -ge $schwelle
"{0} Minuten mit >= {1}/min, Peak: {2}/min" -f $burstMinuten.Count,
    $schwelle, ($proMinute | Select-Object -First 1).Count
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `Where-Object Count -ge $schwelle` | Filtrerar till minuter med minst så många meddelanden som tröskelvärdet (förenklad syntax utan skriptblock) |
| `Select-Object -First 1` | Första gruppen i den fallande sorterade listan, alltså den mest belastade minuten |
| `-f` | Formatoperator: placerar antal, tröskelvärde och topp i platshållarna `{0}` till `{2}` |

</details>

Om burstminuterna ligger sammanhängande (direkt synligt i utdata från `$burstMinuten | Sort-Object Name`) handlar det om en batchkörning. Anteckna starttid, varaktighet och upprepningsmönster, eftersom det är just detta fönster som infrastrukturen måste klara.

## Mottagarstruktur: hur många mål, vilka domäner

För gateways är variationen av mottagare ofta viktigare än själva hastigheten, eftersom uppslag görs per mottagare (routning, policyer, krypteringsregler). Ett e-postmeddelande till en distributionslista med 5 000 medlemmar belastar annorlunda än 5 000 enskilda e-postmeddelanden. Fältet `RecipientCount` och mottagarlistan ger båda perspektiven:

```powershell
"Nachrichten: {0}, Empfänger-Zustellungen: {1}" -f $events.Count,
    ($events | Measure-Object RecipientCount -Sum).Sum

$alleEmpfaenger = $events | ForEach-Object { $_.Recipients } |
    ForEach-Object { $_.ToLower() }
"Eindeutige Empfänger: {0}" -f ($alleEmpfaenger | Sort-Object -Unique).Count
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `Measure-Object RecipientCount -Sum` | Summerar fältet `RecipientCount` över alla event: antalet mottagarleveranser |
| `ForEach-Object { $_.Recipients }` | Vecklar ut varje events mottagarlista till enskilda adresser |
| `ForEach-Object { $_.ToLower() }` | Normaliserar adresserna till gemener så att dubbletter identifieras som sådana |
| `Sort-Object -Unique` | Sorterar och tar bort dubbletter; `Count` ger sedan de unika adresserna |

</details>

Domänfördelningen visar vart trafiken går. Om Gmail och Microsoft dominerar avgör deras rate limits och det egna IP-ryktet den möjliga genomströmningen, inte den egna hårdvaran:

```powershell
$alleEmpfaenger |
    ForEach-Object { ($_ -split "@")[1] } |
    Group-Object |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `($_ -split "@")[1]` | Delar adressen vid `@` och behåller domändelen |
| `Group-Object` | Grupperar utan argument efter värdet självt, här domänen |
| `Sort-Object Count -Descending` | De vanligaste domänerna överst |
| `Select-Object -First 10 Name, Count` | Begränsar utdata till topp 10 |

</details>

Och i motsatt riktning: Vilka avsändare (applikationer, funktionspostlådor) genererar egentligen belastningen? Det besvarar samtidigt frågan om vilka system som måste tas med i beräkningen vid en migrering:

```powershell
$events |
    Group-Object Sender |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `Group-Object Sender` | Grupperar efter fältet `Sender` (positionsparametern `-Property`) |
| `Sort-Object Count -Descending` | Avsändare med flest meddelanden överst |
| `Select-Object -First 10 Name, Count` | Begränsar utdata till topp 10 |

</details>

## Meddelandestorlekar: byte per sekund i stället för e-postmeddelanden per sekund

Gatewayers genomströmningsuppgifter avser ofta datavolym, inte antal meddelanden. Två system med identisk e-posthastighet skiljer sig med en faktor 100 om det ena skickar notifieringar på 50 KB och det andra faktura-PDF:er på 5 MB. Fältet `TotalBytes` ger fördelningen:

```powershell
$events | Measure-Object TotalBytes -Average -Maximum -Sum |
    Select-Object @{n = "MittelKB"; e = { [math]::Round($_.Average / 1KB) } },
        @{n = "MaxMB"; e = { [math]::Round($_.Maximum / 1MB, 1) } },
        @{n = "TotalGB"; e = { [math]::Round($_.Sum / 1GB, 1) } }
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `Measure-Object TotalBytes -Average -Maximum -Sum` | Beräknar medelvärde, maximum och summa för fältet `TotalBytes` i en körning |
| `@{n = "…"; e = { … }}` | Beräknad egenskap: `n` namnger kolumnen, `e` ger värdet via skriptblock, här omräkningen till KB, MB och GB |

</details>

Multiplicera bursthastigheten med genomsnittsstorleken i burstfönstret, så har du bandbreddskravet som en ny gateway eller en WAN-länk måste klara.

## Livehastigheter utan tracking: en titt i köerna

För en ögonblicksbild (behandlar servern just nu mycket, byggs något upp?) behöver du ingen tracking; köerna visar det direkt. `IncomingRate` och `OutgoingRate` är e-postmeddelanden per minut, utjämnade över de senaste minuterna:

```powershell
Get-TransportService |
    ForEach-Object { Get-Queue -Server $_.Name } |
    Sort-Object MessageCount -Descending |
    Select-Object Identity, Status, MessageCount, IncomingRate,
        OutgoingRate, NextHopDomain, LastError |
    Format-Table -AutoSize
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `Get-Queue -Server $_.Name` | Listar transportköerna för respektive server från pipelinen |
| `Sort-Object MessageCount -Descending` | De mest fyllda köerna överst |
| `Select-Object Identity, Status, …` | Begränsar utdata till de fält som är relevanta för belastningsbedömningen |
| `Format-Table -AutoSize` | Anpassar kolumnbredderna efter innehållet i stället för att kapa kolumner |

</details>

Tolkningen: En `Submission`-kö med hög hastighet och djup 0 betyder att servern behandlar belastningen utan att bygga upp en kö. `MessageCount` högt vid `OutgoingRate` nära noll betyder uppdämd kö. `Status Retry` med ett 4xx-meddelande i `LastError` betyder att motparten stryper. `Shadow`-köer med innehåll är däremot normalt; det är redundanskopior för partnerservern, inte en uppdämd kö.

För en kontinuerlig kurva under ett belastningsfönster passar prestandaräknaren för transportköerna, här var femte sekund under en minut:

```powershell
Get-Counter "\MSExchangeTransport Queues(_total)\Messages Completed Delivery Per Second" `
    -SampleInterval 5 -MaxSamples 12
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `"\MSExchangeTransport Queues(_total)\…"` | Sökväg till prestandaräknaren (positionsparametern `-Counter`); instansen `_total` summerar över alla köer |
| `-SampleInterval 5` | Intervall mellan två mätningar i sekunder |
| `-MaxSamples 12` | Antal mätningar; 12 mätningar var femte sekund blir en minut |

</details>

## Andra system: samma princip med CSV

Gateways och apparater tillhandahåller vanligtvis en CSV-export av trackingen i stället för PowerShell-objekt. Tillvägagångssättet är identiskt (välj ett event per e-postmeddelande, gruppera efter tidsfönster), endast verktyget ändras, till exempel till Python:

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

## De fem vanligaste analysfelen

**Flera event per e-postmeddelande.** Den vanligaste felkällan: att räkna rader i stället för meddelanden. Kontrollera med `$events | Group-Object EventId` vad som faktiskt finns i din datamängd och filtrera till exakt ett event per meddelande.

**Avkortade exporter.** Många exportfunktioner levererar högst 10 000 eller 50 000 rader och kapar sedan tyst, gärna mitt i den största bursten. Ett misstänkt jämnt radantal är en varningssignal. Kontrollera alltid om dataperioden motsvarar den begärda perioden.

**Gateway-loopar.** Om e-postflödet går via en mellanliggande station (krypteringsgateway, hygienapparat) och tillbaka, visas samma e-postmeddelande flera gånger i trackingen. Deduplicera med Message-ID eller filtrera till en entydig punkt i kedjan.

**Tidszoner.** `Get-MessageTrackingLog` ger tidsstämplar i lokal servertid, medan CSV-exporter från apparater ofta är i UTC. En burst som till synes inträffar klockan 13 kan i själva verket vara batchen klockan 15. Klargör tidsbasen innan du tolkar resultaten.

**För korta fönster.** En belastningsprofil från två lugna dagar är värdelös om den månatliga faktureringskörningen saknas. Analysfönstret måste innehålla de kända batchcyklerna; fråga applikationsansvariga om deras sändningsplaner innan du fastställer fönstret.

## Vad du gör med profilen

I slutet finns fyra siffror på en sida: genomsnittshastighet, burst (topp, varaktighet, tidpunkt, upprepningsmönster), mottagarstruktur (unika mottagare per körning, toppdomäner) och storleksfördelning. Med detta kan gateways dimensioneras, underhållsfönster läggas till nattimmarna med verklig nollbelastning och acceptanskriterier formuleras, till exempel: Det nya systemet måste kunna behandla dubbelt så mycket som den uppmätta toppen utan fel. Artikeln [SMTP-belastningstest med Apache JMeter i praktiken](/blog/jmeter-smtp-lasttest-html-report) visar hur en reproducerbar belastningstest skapas från en sådan profil.

## Källor

1.  [Microsoft Learn: Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): Referens för trackingfrågan inklusive alla fält som EventId, RecipientCount och TotalBytes.

2.  [Microsoft Learn: Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): Struktur för trackingloggarna, eventtyper och konfiguration av lagring och katalogstorlek.

3.  [Microsoft Learn: Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue): Referens för köfrågan inklusive fälten IncomingRate, OutgoingRate och Velocity.

4.  [Microsoft Learn: Queues and messages in queues](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues): Kötyper, Shadow Redundancy och innebörden av statusvärden.
