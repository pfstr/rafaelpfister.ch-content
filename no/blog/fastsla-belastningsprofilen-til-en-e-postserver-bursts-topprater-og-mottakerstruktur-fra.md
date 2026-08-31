---
title: "Fastslå lastprofilen til en e-postserver: Bursts, topprater og mottakerstruktur fra Message Tracking"
navTitle: "Fastslå lastprofil"
description: "Hvor mange e-poster per minutt behandler e-postserveren din egentlig, og hvor høye er toppene? Slik fastslår du den reelle lastprofilen med PowerShell fra Exchange Message Tracking: rater per minutt og time, burst-varighet, mottakerstruktur, meldingsstørrelser og typiske analysefeil."
date: "2026-08-25"
kategorie: "SMTP og e-postflyt"
timeToRead: "9 min. lesetid"
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
translationSourceHash: 298fabdf5f8f248539ea8a119681be130cd76f5c8ebc35db5d0c61e1126251b0
translationModel: gpt-5.6-terra
translatedAt: 2026-08-30T09:21:51.840Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/fastsla-belastningsprofilen-til-en-e-postserver-bursts-topprater-og-mottakerstruktur-fra
---

# Fastslå lastprofilen til en e-postserver: Bursts, topprater og mottakerstruktur fra Message Tracking

Enten en gateway skal erstattes, en server skal dimensjoneres eller et vedlikeholdsvindu skal planlegges: Før eller siden trenger alle e-postadministratorer svaret på hvor mye systemet deres egentlig behandler. Magefølelsen tar regelmessig feil, for e-posttrafikk er sjelden jevn. Et system som i daglig gjennomsnitt ser 20 e-poster per minutt, kan måtte behandle 400 per minutt i en time under en fakturakjøring. Den som bare kjenner gjennomsnittet, dimensjonerer forbi det egentlige problemet.

En brukbar lastprofil består av fire nøkkeltall: gjennomsnittsraten (per minutt, time, dag), burstene (hvor høy er toppen, hvor lenge varer den, når oppstår den), mottakerstrukturen (hvor mange ulike mottakere, hvilke måldomener) og meldingsstørrelsene. Alle fire finnes i Message Tracking, og i Exchange kan de beregnes med noen få linjer PowerShell.

## Datakilden: Message Tracking

Exchange logger hver melding i Message Tracking Log. Før du analyserer, må du kontrollere hvor langt tilbake dataene går; standarden er 30 dager, men en knapp størrelsesgrense kan forkorte den faktiske oppbevaringstiden betydelig:

```powershell
Get-TransportService |
    Select-Object Name, MessageTrackingLogMaxAge,
        MessageTrackingLogMaxDirectorySize, MessageTrackingLogPath
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `Get-TransportService` | Viser alle organisasjonens transportservere; uten parameter vises alle servere |
| `Select-Object Name, MessageTrackingLog…` | Begrenser utdataene til de angitte egenskapene: oppbevaringstid, størrelsesgrense for loggkatalogen og loggsti |

</details>

For en lastprofil bør perioden dekke minst én full batchsyklus i virksomheten: månedlige fakturakjøringer, lønnskjøringer, nyhetsbrev. Én uke er minimum, én måned er bedre.

## Samle inn rådata: én hendelse per melding

Den viktigste innledende beslutningen: Hvilken hendelse teller som «én e-post»? Message Tracking skriver flere oppføringer per melding (RECEIVE ved mottak, SEND ved videresending til neste hop, DELIVER ved levering til postboks, i tillegg AGENTINFO, HAREDIRECT og flere). Den som bare teller alle linjene, overvurderer volumet mange ganger. For innleveringslast teller du RECEIVE, for utgående last mot smarthost eller Internett SEND.

```powershell
$start = (Get-Date).AddDays(-7)
$events = Get-TransportService | ForEach-Object {
    Get-MessageTrackingLog -Server $_.Name -ResultSize Unlimited `
        -Start $start -EventId RECEIVE
}
"{0} Nachrichten seit {1:yyyy-MM-dd}" -f $events.Count, $start
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `-Server $_.Name` | Spør etter sporingsloggen til den aktuelle transportserveren fra pipelinen |
| `-ResultSize Unlimited` | Opphever standardgrensen på 1'000 returnerte oppføringer |
| `-Start $start` | Nedre tidsgrense for spørringen; her de siste sju dagene |
| `-EventId RECEIVE` | Filtrerer til nøyaktig én hendelse per melding, her mottak av transporttjenesten |
| `-f` | Formatoperator: setter verdiene til høyre inn i plassholderne `{0}` og `{1}` i strengen |

</details>

Spørringen kjøres bevisst mot alle transportservere, ettersom hver server bare logger sin egen andel. Den som bare spør én server, ser bare en brøkdel av lasten i en klynge.

## Rater per minutt og time: her blir burstene synlige

Aggregeringen er et Group-Object på det avrundede tidsstempelet. Toppminuttene er direkte kandidatene dine for burst:

```powershell
$proMinute = $events |
    Group-Object { $_.Timestamp.ToString("yyyy-MM-dd HH:mm") } |
    Sort-Object Count -Descending

$proMinute | Select-Object -First 10 Name, Count
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `Group-Object { … }` | Grupperer etter returverdien fra skriptblokken, her tidsstempelet avkortet til minuttet |
| `Sort-Object Count -Descending` | Sorterer gruppene synkende etter antall; de mest belastede minuttene står øverst |
| `Select-Object -First 10 Name, Count` | Viser bare de ti største gruppene, begrenset til minutt og antall |

</details>

Det samme per time og som døgnmønster (hvilket klokkeslett er vanligvis hvor hardt belastet):

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
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `Group-Object { … ToString("yyyy-MM-dd HH") }` | Grupperer på hele timer i en konkret dag |
| `Group-Object { … ToString("HH") }` | Grupperer bare etter klokkeslettet og aggregerer dermed over alle dager: døgnmønsteret |
| `Sort-Object Count -Descending` | Mest belastede timer øverst |
| `Sort-Object Name` | Sorterer døgnmønsteret kronologisk etter klokkeslett i stedet for etter antall |
| `Format-Table Name, Count` | Tabellvisning av de to kolonnene |

</details>

En burst er først karakterisert når du kjenner varigheten i tillegg til toppen. En topp på 400/min som varer i to minutter, stiller andre krav enn samme topp over én time. Tell minuttene over en terskelverdi:

```powershell
$schwelle = 100
$burstMinuten = $proMinute | Where-Object Count -ge $schwelle
"{0} Minuten mit >= {1}/min, Peak: {2}/min" -f $burstMinuten.Count,
    $schwelle, ($proMinute | Select-Object -First 1).Count
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `Where-Object Count -ge $schwelle` | Filtrerer til minutter med minst så mange meldinger som terskelverdien (forenklet syntaks uten skriptblokk) |
| `Select-Object -First 1` | Første gruppe i den synkende sorterte listen, altså det mest belastede minuttet |
| `-f` | Formatoperator: setter antall, terskelverdi og topp inn i plassholderne `{0}` til `{2}` |

</details>

Hvis burst-minuttene er sammenhengende (direkte synlig i utdataene fra `$burstMinuten | Sort-Object Name`), dreier det seg om en batchkjøring. Noter starttid, varighet og gjentakelsesmønster, for det er nettopp dette vinduet infrastrukturen må tåle.

## Mottakerstruktur: hvor mange mål, hvilke domener

For gatewayer er mottakermangfoldet ofte viktigere enn selve raten, fordi det oppstår oppslag per mottaker (ruting, policyer, krypteringsregler). En e-post til en distribusjonsliste med 5'000 medlemmer belaster annerledes enn 5'000 enkeltepister. Feltet `RecipientCount` og mottakerlisten gir begge perspektivene:

```powershell
"Nachrichten: {0}, Empfänger-Zustellungen: {1}" -f $events.Count,
    ($events | Measure-Object RecipientCount -Sum).Sum

$alleEmpfaenger = $events | ForEach-Object { $_.Recipients } |
    ForEach-Object { $_.ToLower() }
"Eindeutige Empfänger: {0}" -f ($alleEmpfaenger | Sort-Object -Unique).Count
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `Measure-Object RecipientCount -Sum` | Summerer feltet `RecipientCount` over alle hendelser: antall mottakerleveringer |
| `ForEach-Object { $_.Recipients }` | Bretter ut mottakerlisten for hver hendelse til enkeltdresser |
| `ForEach-Object { $_.ToLower() }` | Normaliserer adressene til små bokstaver slik at duplikater gjenkjennes som sådanne |
| `Sort-Object -Unique` | Sorterer og fjerner duplikater; `Count` gir deretter de unike adressene |

</details>

Domenefordelingen viser hvor trafikken flyter. Dersom Gmail og Microsoft dominerer, er det deres rategrenser og din egen IP-reputasjon som avgjør oppnåelig gjennomstrømming, ikke din egen maskinvare:

```powershell
$alleEmpfaenger |
    ForEach-Object { ($_ -split "@")[1] } |
    Group-Object |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `($_ -split "@")[1]` | Deler adressen ved `@` og beholder domene-delen |
| `Group-Object` | Grupperer uten argument etter selve verdien, her domenet |
| `Sort-Object Count -Descending` | Hyppigste domener øverst |
| `Select-Object -First 10 Name, Count` | Begrenser utdataene til topp 10 |

</details>

Og motsatt retning: Hvilke avsendere (applikasjoner, funksjonspostbokser) skaper egentlig lasten? Dette besvarer også spørsmålet om hvilke systemer som må tas med i vurderingen ved en migrering:

```powershell
$events |
    Group-Object Sender |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `Group-Object Sender` | Grupperer etter feltet `Sender` (posisjonsparameter `-Property`) |
| `Sort-Object Count -Descending` | Avsendere med flest meldinger øverst |
| `Select-Object -First 10 Name, Count` | Begrenser utdataene til topp 10 |

</details>

## Meldingsstørrelser: byte per sekund i stedet for e-poster per sekund

Gjennomstrømmingsangivelser for gatewayer gjelder ofte datavolum, ikke antall meldinger. To systemer med identisk e-postrate skiller seg med en faktor på 100 dersom det ene sender varslinger på 50 KB og det andre faktura-PDF-er på 5 MB. Feltet `TotalBytes` gir fordelingen:

```powershell
$events | Measure-Object TotalBytes -Average -Maximum -Sum |
    Select-Object @{n = "MittelKB"; e = { [math]::Round($_.Average / 1KB) } },
        @{n = "MaxMB"; e = { [math]::Round($_.Maximum / 1MB, 1) } },
        @{n = "TotalGB"; e = { [math]::Round($_.Sum / 1GB, 1) } }
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `Measure-Object TotalBytes -Average -Maximum -Sum` | Beregner gjennomsnitt, maksimum og sum for feltet `TotalBytes` i én operasjon |
| `@{n = "…"; e = { … }}` | Beregnet egenskap: `n` navngir kolonnen, `e` leverer verdien per skriptblokk, her omregningen til KB, MB og GB |

</details>

Multipliser burstraten med gjennomsnittsstørrelsen i burst-vinduet, så har du båndbreddekravet som en ny gateway eller en WAN-forbindelse må tåle.

## Direkterater uten sporing: et blikk på køene

For et øyeblikksbilde (behandler serveren mye akkurat nå, hoper noe seg opp?) trenger du ingen sporing; køene viser det direkte. `IncomingRate` og `OutgoingRate` er e-poster per minutt, glattet over de siste minuttene:

```powershell
Get-TransportService |
    ForEach-Object { Get-Queue -Server $_.Name } |
    Sort-Object MessageCount -Descending |
    Select-Object Identity, Status, MessageCount, IncomingRate,
        OutgoingRate, NextHopDomain, LastError |
    Format-Table -AutoSize
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `Get-Queue -Server $_.Name` | Viser transportkøene til den aktuelle serveren fra pipelinen |
| `Sort-Object MessageCount -Descending` | Fulleste køer øverst |
| `Select-Object Identity, Status, …` | Begrenser utdataene til feltene som er relevante for lastvurderingen |
| `Format-Table -AutoSize` | Tilpasser kolonnebreddene til innholdet i stedet for å kutte kolonner |

</details>

Slik leses det: En `Submission`-kø med høy rate og dybde 0 betyr at serveren behandler lasten uten at det bygger seg opp. `MessageCount` høy mens `OutgoingRate` er nær null betyr opphopning. `Status Retry` med en 4xx-melding i `LastError` betyr at motparten struper. `Shadow`-køer med innhold er derimot normalt; det er redundanskopier for partnerserveren, ikke opphopning.

For en kontinuerlig kurve under et lastvindu egner ytelsestelleren for transportkøene seg, her hvert femte sekund i ett minutt:

```powershell
Get-Counter "\MSExchangeTransport Queues(_total)\Messages Completed Delivery Per Second" `
    -SampleInterval 5 -MaxSamples 12
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `"\MSExchangeTransport Queues(_total)\…"` | Sti til ytelsestelleren (posisjonsparameter `-Counter`); instansen `_total` summerer over alle køer |
| `-SampleInterval 5` | Avstand mellom to målinger i sekunder |
| `-MaxSamples 12` | Antall målinger; 12 målinger hvert 5. sekund gir ett minutt |

</details>

## Andre systemer: samme prinsipp med CSV

Gatewayer og apparater leverer vanligvis en CSV-eksport av sporingen i stedet for PowerShell-objekter. Fremgangsmåten er den samme (velg én hendelse per e-post, grupper etter tidsvinduer), bare verktøyet endres, for eksempel til Python:

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

## De fem typiske analysefeilene

**Flere hendelser per e-post.** Den vanligste feilkilden: å telle linjer i stedet for meldinger. Kontroller med `$events | Group-Object EventId` hva som faktisk finnes i datamengden, og filtrer til nøyaktig én hendelse per melding.

**Avkuttede eksporter.** Mange eksportfunksjoner leverer maksimalt 10'000 eller 50'000 linjer og kutter deretter uten kommentar, gjerne midt i den største burst-en. Et mistenkelig rundt linjetall er et alarmsignal. Kontroller alltid om dataperioden tilsvarer den forespurte perioden.

**Gateway-sløyfer.** Hvis e-postflyten går via en mellomstasjon (krypteringsgateway, hygieneapparat) og tilbake igjen, vises samme e-post flere ganger i sporingen. Fjern duplikater basert på Message-ID, eller filtrer til et entydig punkt i kjeden.

**Tidssoner.** `Get-MessageTrackingLog` leverer tidsstempler i lokal servertid, mens CSV-eksporter fra apparater ofte er i UTC. En burst som tilsynelatende skjer klokken 13, kan i virkeligheten være 15-batchen. Avklar tidsgrunnlaget før tolkning.

**For korte vinduer.** En lastprofil basert på to rolige dager er verdiløs hvis den månedlige fakturakjøringen mangler. Analysevinduet må inneholde de kjente batchsyklusene; spør applikasjonsansvarlige om sendeplanene deres før du fastsetter vinduet.

## Hva du kan bruke profilen til

Til slutt står fire tall på én side: gjennomsnittsraten, burst (topp, varighet, tidspunkt, gjentakelsesmønster), mottakerstruktur (unike mottakere per kjøring, toppdomener) og størrelsesfordeling. Med dette kan gatewayer dimensjoneres, vedlikeholdsvinduer legges til nattetimene med reell nullbelastning og akseptansekriterier formuleres, for eksempel: Det nye systemet må kunne behandle det dobbelte av den målte toppen uten feil. Artikkelen [SMTP-lasttest med Apache JMeter i praksis](/blog/jmeter-smtp-lasttest-html-report) viser hvordan en reproduserbar lasttest kan lages av en slik profil.

## Kilder

1.  [Microsoft Learn: Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): Referanse for sporingsspørringen, inkludert alle felt som EventId, RecipientCount og TotalBytes.

2.  [Microsoft Learn: Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): Oppbygging av sporingsloggene, hendelsestyper og konfigurasjon av oppbevaring og katalogstørrelse.

3.  [Microsoft Learn: Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue): Referanse for køspørringen, inkludert feltene IncomingRate, OutgoingRate og Velocity.

4.  [Microsoft Learn: Queues and messages in queues](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues): Køtyper, Shadow Redundancy og betydningen av statusverdiene.
