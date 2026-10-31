---
title: "Hvem leverer egentlig til tenanten din? Aggreger innleverende IP-adresser"
navTitle: "Innleverende IP-er"
description: "Én enkelt analyse viser hvilke systemer som faktisk leverer e-post til tenanten din: glemte connectorer, programmer som sender direkte og tjenesteleverandører ingen har dokumentert, inkludert de typiske analysefeilene knyttet til sidelogikk og tolkning."
date: "2026-08-11"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "12 min. lesetid"
themen:
  - microsoft-365-exchange
  - smtp-mailflow
  - exchange-onprem-hybrid
hauptthema: "microsoft-365-exchange"
produkte:
  - "exchange-online"
  - "exchange-hybrid"
protokolle:
  - "smtp"
  - "powershell"
  - "haertung"
related:
  - exchange-message-tracking-und-receive-connectoren-analysieren
  - ghost-sender-exchange-online-nebeneingang
  - mailflow-fehlersuche-kontrollierte-experimente
slug: "hvem-leverer-egentlig-e-post-til-tenant-en-din-aggreger-innkommende-ip-adresser"
translationId: "article-5879cc0eb17ed951"
draft: false
translationOf: einliefernde-ip-adressen-aggregieren
translationSourceHash: 291ed92e756d5fa011212a5cf9d3ce9061df96d73857f508c838183965ba25b4
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:22:29.227Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/hvem-leverer-egentlig-e-post-til-tenant-en-din-aggreger-innkommende-ip-adresser
---

# Hvem leverer egentlig til tenanten din? Aggreger innleverende IP-adresser

Nesten ingen e-postmiljøer vet lenger fullt ut hvem som leverer e-post til dem. Gjennom årene samler det seg connectorer fra migreringer, programmer som sender direkte, tjenesteleverandører med kontrakter som for lengst har utløpt, og testoppsett som aldri ble ryddet bort. Så lenge e-posten flyter, legger ingen merke til det.

Én enkelt analyse skaper klarhet: grupperingen av alle innkommende meldinger etter kilde-IP-adressen. Den er gjort på to minutter, og resultatlisten er jevnlig overraskende. Denne artikkelen viser spørringen, forklarer hvordan du får den **fullstendig**, og fremfor alt hvordan du leser tallene riktig. For tolkningen er den vanskeligste delen.

## Hvorfor det lønner seg

Listen besvarer fire spørsmål som ellers er tidkrevende å avklare enkeltvis. Hvilke systemer sender egentlig til tenanten din? Går alt via veiene du har dokumentert, eller finnes det en annen inngang? Brukes en connector du vil avvikle fortsatt? Og: Sender et program direkte til tjenesten forbi gatewayen din, altså utenom filtreringen?

Særlig det siste spørsmålet er sikkerhetsrelevant. Den som leverer direkte, omgår ikke bare filtreringen, men ofte også loggingen du ønsker å kunne stole på i en alvorlig situasjon.

## Spørringen

I tenanten grupperer du meldingssporingen etter `FromIP`:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-2) `
    -EndDate (Get-Date) `
    -ResultSize 5000 |
  Group-Object FromIP |
  Sort-Object Count -Descending |
  Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-StartDate (Get-Date).AddHours(-2)` | Starten på spørringsvinduet, her for to timer siden |
| `-EndDate (Get-Date)` | Slutten på spørringsvinduet, det nåværende tidspunktet |
| `-ResultSize 5000` | maksimalt antall rader per kall; 5000 er også maksimumsverdien |
| `Group-Object FromIP` | grupperer meldingene etter den innleverende IP-adressen |
| `Sort-Object Count -Descending` | sorterer gruppene synkende etter antall meldinger |
| `Format-Table Count, Name -AutoSize` | utdata i to kolonner (antall, IP-adresse) med automatisk kolonnebredde |

</details>

Et typisk utdata:

```text
Count Name
----- ----
 1771 255.255.255.255
 1649 10.0.20.23
  260 10.0.20.21
   49 2603:10a6:150:1f3::17
   46 165.225.94.87
   36 136.226.192.164
   35 147.161.246.105
   12 198.51.100.77
    3 203.0.113.9
```

Før du trekker konklusjoner av dette, må to ting stemme: Listen må være fullstendig, og du må vite hva oppføringene betyr.

## Feilkilde 1: Listen er nesten alltid ufullstendig

`Get-MessageTraceV2` leverer sidevis, maksimalt 5000 rader per kall. Ved høy trafikk dekker én side bare en brøkdel av tidsvinduet ditt. Da grupperer du over et utsnitt og anser resultatet som helheten.

Dette ser du på denne advarselen:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

**Hvis denne advarselen vises, er analysen din verdiløs.** Særlig må en manglende oppføring da ikke tolkes som fravær. En adresse med tre meldinger per dag vil uansett ikke vises i et utsnitt.

Det finnes to utveier. Den enkle: Reduser tidsvinduet til advarselen ikke lenger vises. Ved 5000 meldinger i timen er det altså 55 minutter og ikke sju dager. For spørsmålet «hvilke systemer sender i det hele tatt» er et fullstendig kort vindu som regel helt tilstrekkelig.

Den grundige metoden blar gjennom alle sidene og samler inn:

```powershell
$start = (Get-Date).AddHours(-24)
$ende  = Get-Date
$alle  = @()
$naechster = $null

do {
    $seite = if ($naechster) {
        Get-MessageTraceV2 -StartDate $start -EndDate $ende `
            -StartingRecipientAddress $naechster -ResultSize 5000
    } else {
        Get-MessageTraceV2 -StartDate $start -EndDate $ende -ResultSize 5000
    }

    $alle += $seite
    if ($seite.Count -eq 5000) {
        $naechster = $seite[-1].RecipientAddress
        $ende      = $seite[-1].Received
    } else {
        $naechster = $null
    }
    Write-Host "Gesammelt: $($alle.Count)"
} while ($naechster)

$alle | Group-Object FromIP | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-StartDate` / `-EndDate` | Spørringsvindu, her de siste 24 timene |
| `-StartingRecipientAddress` | Fortsettelsespunkt for sidelogikken: mottakeradressen til siste oppføring på forrige side; sammen med `-EndDate` = mottakstidspunktet (`Received`) begynner neste side der |
| `-ResultSize 5000` | Sidestørrelse; en full side signaliserer at flere resultater følger |
| `Group-Object FromIP` | grupperer hele datamengden etter den innleverende IP-adressen |
| `Sort-Object Count -Descending` | sorterer gruppene synkende etter antall meldinger |
| `Format-Table Count, Name -AutoSize` | utdata av antall per adresse med automatisk kolonnebredde |

</details>

Løkken henter flere sider så lenge en side returneres med nøyaktig 5000 rader, og fortsetter hver gang med mottakeradressen og mottakstidspunktet for siste oppføring på forrige side, slik dokumentasjonen for `Get-MessageTraceV2` angir; først den fullstendige datamengden grupperes.

Regn med noen minutters kjøretid for 24 timer i et mellomstort miljø. For en engangskartlegging er det vel anvendt tid.

## Feilkilde 2: Tallene betyr ikke det de ser ut til å bety

Resultatlisten inneholder fire helt forskjellige typer oppføringer, og den som blander dem sammen, trekker feil konklusjoner.

**`255.255.255.255` står ikke for et system.** Denne verdien vises når det ikke fantes noen innkommende SMTP-forbindelse utenfra for meldingen. Det gjelder meldinger opprettet i selve tjenesten: journalrapporter, leveringsfeilmeldinger, fraværsnotater, meldinger mellom postbokser i samme tenant. I nesten alle miljøer er dette den største posten, og den er helt uproblematisk.

**Private adresser fra RFC 1918** stammer fra ditt eget nettverk. I hybridmiljøer ser du de lokale transportserverne her, fordi den interne adressen deres beholdes ved overlevering til tjenesten. Dette er de store tallene i listen og som regel den forventede hovedveien.

**Adresser fra sikkerhets- og filtreringstjenester** kjenner du igjen på operatøren, ikke på tallverdien. Cloud-proxyer, foranstilte e-postgatewayer og web-sikkerhetstjenester vises med mange nærliggende adresser og middels tall. De hører som regel hjemme der, men bør stå i driftsdokumentasjonen.

**Enkeltstående offentlige adresser med små tall** er de interessante. Det er nettopp der glemte programmer, gamle tjenesteleverandører og systemer ingen lenger kjenner til, skjuler seg.

## Oppløsningen: Fra adresser til navn

For alt du ikke umiddelbart kan plassere, hjelper omvendt oppslag. Det er ikke alltid satt og ikke alltid pålitelig, men i de fleste tilfeller gir det det avgjørende hintet:

```powershell
$unbekannt = '198.51.100.77','203.0.113.9'

$unbekannt | ForEach-Object {
    $name = try { (Resolve-DnsName $_ -Type PTR -ErrorAction Stop).NameHost } catch { '(kein PTR)' }
    [pscustomobject]@{ IP = $_; Name = $name }
} | Format-Table -AutoSize
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Resolve-DnsName $_ -Type PTR` | spør etter den omvendte oppføringen (PTR) for hver IP-adresse |
| `-ErrorAction Stop` | gjør en manglende oppføring til en fangbar feil for `try`/`catch`-blokken |
| `[pscustomobject]@{ … }` | bygger et objekt per adresse med IP og oppløst navn for tabellutdata |
| `Format-Table -AutoSize` | utdata med automatisk kolonnebredde |

</details>

```text
IP            Name
--            ----
198.51.100.77 mail-out-03.newsletter-provider.example
203.0.113.9   (kein PTR)
```

En manglende PTR er ikke i seg selv et tegn på et problem, men det er en god grunn til å se nærmere på det. Se på de tilhørende meldingene for slike adresser:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date) -ResultSize 5000 |
  Where-Object { $_.FromIP -eq '203.0.113.9' } |
  Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-StartDate` / `-EndDate` / `-ResultSize` | Spørringsvindu og sidestørrelse som i hovedspørringen |
| `Where-Object { $_.FromIP -eq '203.0.113.9' }` | filtrerer på klientsiden etter den aktuelle kildeadressen |
| `Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize` | viser mottakstidspunkt, avsender, mottaker, emne og leveringsstatus per melding |

</details>

Avsender og emne forteller deg som regel umiddelbart hvilket program som står bak.

## Sammenligningen: Hvilken adresse hører til hvilken connector?

Sammenlign resultatlisten med de konfigurerte connectorene:

```powershell
Get-InboundConnector |
    Format-List Name, Enabled, ConnectorType, ConnectorSource,
        @{n='SenderIPAddresses'; e={$_.SenderIPAddresses -join ', '}},
        @{n='SenderDomains';     e={$_.SenderDomains -join ', '}},
        RequireTls, TlsSenderCertificateName,
        RestrictDomainsToCertificate, CloudServicesMailEnabled, TreatMessagesAsInternal
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Get-InboundConnector` | lister alle innkommende connectorer i tenanten; her bevisst uten begrensende parametere |
| `Format-List <Eigenschaften>` | utdata som en liste over de angitte egenskapene, én per linje |
| `@{n='…'; e={…}}` | beregnet egenskap med navn (`n`) og uttrykk (`e`) |
| `-join ', '` | gjør arrayet av adresser eller domener om til en lesbar, kommaseparert linje |

</details>

Tre konstellasjoner er opplysende.

**En adresse leverer, men er ikke angitt i noen connector.** Da kommer e-posten inn som vanlig Internett-e-post. Det er tillatt, men betyr at dette programmet ikke får noen særbehandling, og at meldingene er underlagt full filtrering. Hvis noen hevder at en connector er konfigurert for dette systemet, stemmer det åpenbart ikke lenger.

**En connector angir adresser som det ikke kommer noe fra.** En kandidat for avvikling. Kontroller før du sletter om det dreier seg om sesongbaserte eller sjeldne systemer, og deaktiver først i stedet for å fjerne med én gang.

**En connector setter `TreatMessagesAsInternal` eller `CloudServicesMailEnabled` til sann.** Her bør du se nøye etter. Begge innstillingene gjør at meldinger via denne veien behandles som interne i organisasjonen. Hvis e-post fra Internett kommer inn denne veien, omgår den dermed kontroller som er ment for eksterne meldinger, blant annet beskyttelsen mot forfalskede avsendere fra eget domene. For en ren hybrid-connector er dette riktig; for en connector som vilkårlige systemer leverer via, er det et funn.

## Hva du vanligvis finner

Fra praksis, uten å gjøre krav på fullstendighet: En testconnector fra en migrering som har vært aktiv i årevis. Et fagprogram som sender direkte til tjenesten, selv om alle tror det går via gatewayen. En nyhetsbrevleverandør med utløpt kontrakt som fortsatt får levere. Og jevnlig en connector med vidåpne betingelser som noen en gang opprettet for å løse et akutt problem.

Ingen av disse funnene er dramatiske i seg selv. Sammen gir de bildet av et miljø ingen lenger har full oversikt over, og nettopp det er den egentlige risikoen.

## Metodens begrensninger

Du bør kjenne til tre begrensninger.

`Get-MessageTraceV2` dekker de siste 90 dagene, men leverer høyst ti dager per spørring. For lengre tidsrom spør du derfor i flere tidsvinduer eller bruker det historiske søket, som kjører asynkront og også dekker opptil 90 dager. Ellers går du glipp av sjeldne systemer som sender månedlig.

`FromIP` betyr ikke det samme overalt. For e-post fra Internett er det adressen til den innleverende serveren. For hybrid-e-post er det adressen til den lokale transportserveren din, ikke den opprinnelige avsenderen. Analysen viser deg altså **siste stasjon før tjenesten**, ikke opprinnelsen.

Og tilordningen til en connector er ikke direkte synlig i tenanten. Du utleder den fra adresse, sertifikat og avsenderdomene. For en pålitelig uttalelse om bruken av en enkelt connector er connector-rapporten i Exchange Admin Center under Rapporter og e-postflyt den bedre kilden, fordi den aggregerer på serversiden over lengre tidsrom.

## Som tilbakevendende kontroll

Denne analysen egner seg godt som kvartalsrutine. Lagre resultatet og sammenlign ved neste gjennomgang. Nye adresser i listen er enten dokumenterte endringer eller noe du bør vite om.

Hvis du uansett kontrollerer e-postkonfigurasjonen for domenene dine: [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check) viser SPF, DKIM, DMARC og de øvrige e-poststandardene for vilkårlige domener direkte i nettleseren, også for under- og markedsføringsdomener som erfaringsmessig blir glemt i slike kartlegginger. Og for selve spørringene gir [Befehls-Generator](https://rafaelpfister.ch/tools/command-builder) ferdige byggeklosser for PowerShell og Unix-skall.

Hvordan du følger opp enkeltstående mistenkelige meldinger, står i [Analyser Exchange-e-postflyt: Message Tracking, SMTP-protokoller og Receive Connectors](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren).

## Endringslogg

- **25. oktober 2026:** Korrigert: Sidelogikken for Get-MessageTraceV2 fortsetter med mottakeradresse og mottakstidspunkt (EndDate) for siste oppføring. Korrigert: Cmdleten dekker de siste 90 dagene, men leverer høyst ti dager per spørring.

## Kilder

1.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): Feltliste inkludert FromIP og ToIP, spørringsperiode (90 dager, høyst ti dager per spørring) samt fortsettelsen med StartingRecipientAddress og EndDate.

2.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch): asynkron meldingssporing over opptil 90 dager for eldre tidsrom.

3.  [Get-InboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchange/get-inboundconnector): parametere for innkommende connectorer, blant annet SenderIPAddresses og TreatMessagesAsInternal.

4.  [Configure mail flow using connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow): samspillet mellom connectortypene og når hvilken brukes.

5.  [RFC 1918: Address Allocation for Private Internets](https://www.rfc-editor.org/rfc/rfc1918): definerer de private adresseområdene du må skille fra offentlige adresser i analysen.
