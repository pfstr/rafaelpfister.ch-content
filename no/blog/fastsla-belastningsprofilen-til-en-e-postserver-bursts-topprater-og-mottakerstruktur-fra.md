---
title: "Fastslå belastningsprofilen til en e-postserver: Bursts, topprater og mottakerstruktur fra Message Tracking"
navTitle: "Fastslå belastningsprofil"
description: "Hvor mange e-poster per minutt behandler e-postserveren din egentlig, og hvor høye er toppene? Slik bruker du PowerShell til å hente den reelle belastningsprofilen fra Exchange Message Tracking: rater per minutt og time, burst-varighet, mottakerstruktur og meldingsstørrelser. Med de typiske analysefellene."
date: "2026-08-25"
kategorie: "SMTP og e-postflyt"
timeToRead: "9 min lesetid"
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
slug: "fastsla-belastningsprofilen-til-en-e-postserver-bursts-topprater-og-mottakerstruktur-fra"
translationId: "article-1ff17a188d73e289"
aiPrompt: |
  Du bist mein Mailflow-Assistent. Hilf mir Schritt für Schritt, das Lastprofil meines Mailservers zu ermitteln: 1. Die richtige Datenquelle wählen (Message Tracking, Gateway-Logs) und das passende Event pro Nachricht bestimmen. 2. Raten pro Minute, Stunde und Tag berechnen und Bursts mit Dauer und Peak charakterisieren. 3. Empfängerstruktur, Domain-Verteilung und Nachrichtengrössen auswerten. Weise mich auf Doppelzählungen, Export-Limits und Zeitzonen-Fallen hin.
translationOf: mailserver-lastprofil-ermitteln
url: https://rafaelpfister.ch/no/blog/fastsla-belastningsprofilen-til-en-e-postserver-bursts-topprater-og-mottakerstruktur-fra
translationSourceHash: 16095cf53ce6f67abe31387ce2f02958eacc3898d3a42b61ad8c7b885ab7ce5d
translationModel: gpt-5.6-terra
translatedAt: 2026-08-26T04:11:14.999Z
translationReview: automatic
---

# Fastslå belastningsprofilen til en e-postserver: Bursts, topprater og mottakerstruktur fra Message Tracking

Enten en gateway skal erstattes, en server dimensjoneres eller et vedlikeholdsvindu planlegges: Før eller siden trenger enhver e-postadministrator svar på hvor mye systemet faktisk behandler. Magefølelsen tar regelmessig feil, for e-posttrafikk er sjelden jevn. Et system som i gjennomsnitt per dag ser 20 e-poster per minutt, kan måtte behandle 400 per minutt i en time under en fakturakjøring. Den som bare kjenner gjennomsnittet, dimensjonerer forbi det egentlige problemet.

En brukbar belastningsprofil består av fire nøkkeltall: gjennomsnittsrate (per minutt, time, dag), burst-er (hvor høy er toppen, hvor lenge varer den, når oppstår den), mottakerstruktur (hvor mange forskjellige mottakere, hvilke måldomener) og meldingsstørrelser. Alle fire finnes i Message Tracking, og i Exchange kan de beregnes med noen få linjer PowerShell.

## Datakilden: Message Tracking

Exchange logger hver melding i Message Tracking Log. Før du analyserer, må du kontrollere hvor langt tilbake dataene går; standarden er 30 dager, men en lav størrelsesgrense kan forkorte den faktiske oppbevaringen betydelig:

```powershell
Get-TransportService |
    Select-Object Name, MessageTrackingLogMaxAge,
        MessageTrackingLogMaxDirectorySize, MessageTrackingLogPath
```

For en belastningsprofil bør tidsrommet dekke minst én full batchsyklus i virksomheten: månedlige fakturakjøringer, lønnsutbetalinger, nyhetsbrev. Én uke er minimum, én måned er bedre.

## Samle inn rådata: én hendelse per melding

Det viktigste valget på forhånd: Hvilken hendelse teller som «én e-post»? Message Tracking skriver flere oppføringer per melding (RECEIVE ved mottak, SEND ved videresending til neste hop, DELIVER ved levering til postboks, i tillegg AGENTINFO, HAREDIRECT og flere). Den som bare teller alle linjene, overvurderer volumet mange ganger. For innleveringsbelastning teller du RECEIVE, for utgående belastning mot smarthost eller Internett SEND.

```powershell
$start = (Get-Date).AddDays(-7)
$events = Get-TransportService | ForEach-Object {
    Get-MessageTrackingLog -Server $_.Name -ResultSize Unlimited `
        -Start $start -EventId RECEIVE
}
"{0} Nachrichten seit {1:yyyy-MM-dd}" -f $events.Count, $start
```

Spørringen kjøres bevisst mot alle transportservere, for hver server logger bare sin egen andel. Den som bare spør én server, ser bare en brøkdel av belastningen i en klynge.

## Rater per minutt og time: her blir burst-ene synlige

Aggregeringen er et Group-Object på det avrundede tidsstempelet. Toppminuttene er direkte dine burst-kandidater:

```powershell
$proMinute = $events |
    Group-Object { $_.Timestamp.ToString("yyyy-MM-dd HH:mm") } |
    Sort-Object Count -Descending

$proMinute | Select-Object -First 10 Name, Count
```

Det samme per time og som døgnprofil (hvilket klokkeslett er typisk hvor tungt belastet):

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

En burst er først karakterisert når du, i tillegg til toppen, også kjenner varigheten. En topp på 400/min som varer i to minutter, stiller andre krav enn samme topp over en time. Tell minuttene over en terskelverdi:

```powershell
$schwelle = 100
$burstMinuten = $proMinute | Where-Object Count -ge $schwelle
"{0} Minuten mit >= {1}/min, Peak: {2}/min" -f $burstMinuten.Count,
    $schwelle, ($proMinute | Select-Object -First 1).Count
```

Ligger burst-minuttene sammenhengende (direkte synlig i utdataene fra `$burstMinuten | Sort-Object Name`), er det en batchkjøring. Noter starttid, varighet og gjentakelsesmønster, for infrastrukturen må tåle nettopp dette vinduet.

## Mottakerstruktur: hvor mange mål, hvilke domener

For gatewayer er mottakervariasjonen ofte viktigere enn selve raten, fordi det gjøres oppslag per mottaker (ruting, policyer, krypteringsregler). En e-post til en distribusjonsliste med 5'000 medlemmer belaster annerledes enn 5'000 enkeltmails. Feltet `RecipientCount` og mottakerlisten gir begge perspektivene:

```powershell
"Nachrichten: {0}, Empfänger-Zustellungen: {1}" -f $events.Count,
    ($events | Measure-Object RecipientCount -Sum).Sum

$alleEmpfaenger = $events | ForEach-Object { $_.Recipients } |
    ForEach-Object { $_.ToLower() }
"Eindeutige Empfänger: {0}" -f ($alleEmpfaenger | Sort-Object -Unique).Count
```

Domenefordelingen viser hvor trafikken går. Hvis Gmail og Microsoft dominerer, avgjør deres rategrenser og din egen IP-reputasjon den oppnåelige gjennomstrømmingen, ikke din egen maskinvare:

```powershell
$alleEmpfaenger |
    ForEach-Object { ($_ -split "@")[1] } |
    Group-Object |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

Og motsatt vei: Hvilke avsendere (applikasjoner, funksjonspostbokser) skaper egentlig belastningen? Dette besvarer samtidig spørsmålet om hvilke systemer som må tas hensyn til ved en migrering:

```powershell
$events |
    Group-Object Sender |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

## Meldingsstørrelser: byte per sekund i stedet for e-poster per sekund

Gjennomstrømmingsangivelser for gatewayer refererer ofte til datavolum, ikke antall meldinger. To systemer med identisk e-postrate kan skille seg med en faktor på 100 dersom det ene sender varsler på 50 KB og det andre faktura-PDF-er på 5 MB. Feltet `TotalBytes` gir fordelingen:

```powershell
$events | Measure-Object TotalBytes -Average -Maximum -Sum |
    Select-Object @{n = "MittelKB"; e = { [math]::Round($_.Average / 1KB) } },
        @{n = "MaxMB"; e = { [math]::Round($_.Maximum / 1MB, 1) } },
        @{n = "TotalGB"; e = { [math]::Round($_.Sum / 1GB, 1) } }
```

Multipliser burst-raten med gjennomsnittsstørrelsen i burst-vinduet, så har du båndbreddekravet som en ny gateway eller en WAN-forbindelse må tåle.

## Direkterater uten sporing: se på køene

For et øyeblikksbilde (behandler serveren mye akkurat nå, hoper noe seg opp?) trenger du ingen sporing; køene viser det direkte. `IncomingRate` og `OutgoingRate` er e-poster per minutt, utjevnet over de siste minuttene:

```powershell
Get-TransportService |
    ForEach-Object { Get-Queue -Server $_.Name } |
    Sort-Object MessageCount -Descending |
    Select-Object Identity, Status, MessageCount, IncomingRate,
        OutgoingRate, NextHopDomain, LastError |
    Format-Table -AutoSize
```

Tolkningen: En `Submission`-kø med høy rate og dybde 0 betyr at serveren behandler belastningen uten opphopning. `MessageCount` høy med `OutgoingRate` nær null betyr opphopning. `Status Retry` med en 4xx-melding i `LastError` betyr at motparten begrenser trafikken. `Shadow`-køer med innhold er derimot normalt; dette er redundanskopier for partner-serveren, ikke opphopning.

For en kontinuerlig kurve i et belastningsvindu egner ytelsestelleren for transportkøene seg, her hvert femte sekund i ett minutt:

```powershell
Get-Counter "\MSExchangeTransport Queues(_total)\Messages Completed Delivery Per Second" `
    -SampleInterval 5 -MaxSamples 12
```

## Andre systemer: samme prinsipp med CSV

Gatewayer og appliances leverer vanligvis en CSV-eksport av sporingen i stedet for PowerShell-objekter. Fremgangsmåten er den samme (velg én hendelse per e-post, grupper etter tidsvinduer), bare verktøyet byttes, for eksempel til Python:

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

## De fem klassiske analysefellene

**Flere hendelser per e-post.** Den vanligste feilkilden: å telle linjer i stedet for meldinger. Kontroller med `$events | Group-Object EventId` hva som faktisk finnes i datamengden, og filtrer på nøyaktig én hendelse per melding.

**Avkuttede eksporter.** Mange eksportfunksjoner leverer maksimalt 10'000 eller 50'000 linjer og kutter deretter stille av, gjerne midt i den største burst-en. Et mistenkelig rundt antall linjer er et alarmsignal. Kontroller alltid om dataperioden tilsvarer den forespurte perioden.

**Gateway-sløyfer.** Hvis e-postflyten går via en mellomstasjon (krypteringsgateway, hygiene-appliance) og tilbake igjen, dukker samme e-post opp flere ganger i sporingen. Fjern duplikater basert på Message-ID, eller filtrer på ett entydig punkt i kjeden.

**Tidssoner.** `Get-MessageTrackingLog` leverer tidsstempler i lokal servertid, mens CSV-eksporter fra appliances ofte er i UTC. En burst som tilsynelatende kjører klokken 13, kan i realiteten være batchen klokken 15. Avklar tidsgrunnlaget før tolkning.

**For korte vinduer.** En belastningsprofil fra to rolige dager er verdiløs hvis den månedlige fakturakjøringen mangler. Analysevinduet må inneholde de kjente batchsyklusene; spør applikasjonsansvarlige om utsendelsesplanene deres før du fastsetter vinduet.

## Hva du kan bruke profilen til

Til slutt står fire tall på én side: gjennomsnittsrate, burst (topp, varighet, tidspunkt, gjentakelsesmønster), mottakerstruktur (entydige mottakere per kjøring, toppdomener) og størrelsesfordeling. Med dette kan gatewayer dimensjoneres, vedlikeholdsvinduer legges til nattestimer med reell nullbelastning og akseptansekriterier formuleres, for eksempel: Det nye systemet må kunne behandle det dobbelte av den målte toppen uten feil. Artikkelen [SMTP-belastningstest med Apache JMeter i praksis](/blog/jmeter-smtp-lasttest-html-report) viser hvordan en slik profil blir til en reproduserbar belastningstest.

## Kilder

1.  [Microsoft Learn: Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): Referanse for sporingsspørringen, inkludert alle felt som EventId, RecipientCount og TotalBytes.

2.  [Microsoft Learn: Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): Oppbygging av sporingsloggene, hendelsestyper og konfigurasjon av oppbevaring og katalogstørrelse.

3.  [Microsoft Learn: Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue): Referanse for køspørringen, inkludert feltene IncomingRate, OutgoingRate og Velocity.

4.  [Microsoft Learn: Queues and messages in queues](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues): Køtyper, Shadow Redundancy og betydningen av statusverdiene.
