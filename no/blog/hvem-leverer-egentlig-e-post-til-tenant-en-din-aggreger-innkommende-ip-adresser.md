---
title: "Hvem leverer egentlig e-post til tenant-en din? Aggreger innkommende IP-adresser"
navTitle: "Innkommende IP-er"
description: "En enkelt analyse viser hvilke systemer som faktisk leverer e-post til tenant-en din: glemte koblinger, programmer som sender direkte, og tjenesteleverandører som ingen har dokumentert. Inkludert fallgruvene ved sidelogiikk og tolkning."
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
url: https://rafaelpfister.ch/no/blog/hvem-leverer-egentlig-e-post-til-tenant-en-din-aggreger-innkommende-ip-adresser
translationSourceHash: 9dc48329a06945f705380eb3db428efb548f0c36a1fe3c4f2fb7de1185fee879
translationModel: gpt-5.6-terra
translatedAt: 2026-08-12T05:14:01.177Z
translationReview: automatic
---

# Hvem leverer egentlig e-post til tenant-en din? Aggreger innkommende IP-adresser

Knapt noe e-postmiljø vet lenger fullt ut hvem som leverer e-post til det. Gjennom årene samler det seg opp koblinger fra migreringer, programmer som sender direkte, tjenesteleverandører med kontrakter som for lengst har utløpt, og testoppsett som aldri ble fjernet. Så lenge e-posten flyter, legger ingen merke til det.

En enkelt analyse skaper klarhet: grupperingen av alle innkommende meldinger etter kilde-IP-adresse. Den er ferdig på to minutter, og resultatlisten er jevnlig overraskende. Denne artikkelen viser spørringen, forklarer hvordan du får den **fullstendig**, og fremfor alt hvordan du leser tallene riktig. For tolkningen er den vanskeligste delen.

## Hvorfor det lønner seg

Listen besvarer fire spørsmål som ellers er krevende å avklare enkeltvis. Hvilke systemer sender i det hele tatt til tenant-en din? Går alt via veiene du har dokumentert, eller finnes det en annen inngang? Er en kobling du vil avvikle, fortsatt i bruk? Og: Sender et program direkte til tjenesten utenom gatewayen din, altså utenom filtreringen?

Særlig det siste spørsmålet er sikkerhetsrelevant. Den som leverer direkte, omgår ikke bare filtreringen, men ofte også loggføringen du ønsker å kunne stole på i en alvorlig situasjon.

## Spørringen

I tenant-en grupperer du meldingssporingen etter `FromIP`:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-2) `
    -EndDate (Get-Date) `
    -ResultSize 5000 |
  Group-Object FromIP |
  Sort-Object Count -Descending |
  Format-Table Count, Name -AutoSize
```

En typisk utdata:

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

Før du trekker konklusjoner, må to ting være på plass: Listen må være fullstendig, og du må vite hva oppføringene betyr.

## Fallgruve 1: Listen er nesten alltid ufullstendig

`Get-MessageTraceV2` leverer sidevis, maksimalt 5000 linjer per kall. Ved høyt volum dekker én side bare en brøkdel av tidsvinduet ditt. Da grupperer du over et utsnitt og holder resultatet for helheten.

Dette kjenner du igjen på denne advarselen:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

**Vises denne advarselen, er analysen din verdiløs.** Særlig må en manglende oppføring da ikke tolkes som fravær. En adresse med tre meldinger per dag dukker uansett ikke opp i et utsnitt.

Det finnes to utveier. Den enkle: Reduser tidsvinduet til advarselen ikke lenger vises. Ved 5000 meldinger per time er det 55 minutter og ikke sju dager. For spørsmålet «hvilke systemer sender i det hele tatt» er et fullstendig, kort tidsvindu som regel helt tilstrekkelig.

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
    $naechster = if ($seite.Count -eq 5000) { $seite[-1].RecipientAddress } else { $null }
    Write-Host "Gesammelt: $($alle.Count)"
} while ($naechster)

$alle | Group-Object FromIP | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

Regn med noen minutters kjøretid for 24 timer i et mellomstort miljø. For en engangs kartlegging er det vel anvendt tid.

## Fallgruve 2: Tallene betyr ikke det de ser ut til å bety

Resultatlisten inneholder fire helt ulike typer oppføringer, og den som slår dem sammen, trekker feil konklusjoner.

**`255.255.255.255` står ikke for et system.** Denne verdien vises når det ikke fantes en innkommende SMTP-forbindelse utenfra for meldingen. Det gjelder meldinger som genereres i selve tjenesten: journalrapporter, leveringsfeilmeldinger, fraværsmeldinger og meldinger mellom postbokser i samme tenant. I nesten alle miljøer er dette den største posten, og den er helt uproblematisk. Ikke bli skremt.

**Private adresser fra RFC 1918** kommer fra ditt eget nettverk. I hybridmiljøer ser du de lokale transportserverne her, fordi den interne adressen deres beholdes ved overlevering til tjenesten. Dette er de store tallene i listen og som regel den forventede hovedveien.

**Adresser fra sikkerhets- og filtreringstjenester** gjenkjenner du på operatøren, ikke tallverdien. Skyproxyer, e-postgatewayer foran tjenesten og nettsikkerhetstjenester vises med mange nærliggende adresser og middels tall. De hører som regel hjemme der, men bør stå i driftsdokumentasjonen.

**Enkeltstående offentlige adresser med lave tall** er de interessante. Det er nettopp der de glemte programmene, gamle tjenesteleverandørene og systemene ingen lenger kjenner til, skjuler seg.

## Oppløsningen: fra adresser til navn

For alt du ikke umiddelbart kan tilordne, hjelper omvendt oppslag. Det er ikke alltid satt opp og ikke alltid ærlig, men gir i de fleste tilfeller det avgjørende hintet:

```powershell
$unbekannt = '198.51.100.77','203.0.113.9'

$unbekannt | ForEach-Object {
    $name = try { (Resolve-DnsName $_ -Type PTR -ErrorAction Stop).NameHost } catch { '(kein PTR)' }
    [pscustomobject]@{ IP = $_; Name = $name }
} | Format-Table -AutoSize
```

```text
IP            Name
--            ----
198.51.100.77 mail-out-03.newsletter-provider.example
203.0.113.9   (kein PTR)
```

Manglende PTR er ikke bevis på noe ondsinnet, men det er en god grunn til å se nærmere. Se på de tilhørende meldingene for slike adresser:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date) -ResultSize 5000 |
  Where-Object { $_.FromIP -eq '203.0.113.9' } |
  Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize
```

Avsender og emne forteller deg som regel umiddelbart hvilket program som står bak.

## Sammenligningen: Hvilken adresse tilhører hvilken kobling?

Nå kommer den egentlige innsikten. Sammenlign resultatlisten med de konfigurerte koblingene:

```powershell
Get-InboundConnector |
    Format-List Name, Enabled, ConnectorType, ConnectorSource,
        @{n='SenderIPAddresses'; e={$_.SenderIPAddresses -join ', '}},
        @{n='SenderDomains';     e={$_.SenderDomains -join ', '}},
        RequireTls, TlsSenderCertificateName,
        RestrictDomainsToCertificate, CloudServicesMailEnabled, TreatMessagesAsInternal
```

Tre konstellasjoner er opplysende.

**En adresse leverer, men er ikke nevnt i noen kobling.** Da kommer e-posten inn som vanlig internett-e-post. Det er tillatt, men betyr at dette programmet ikke får noen særbehandling, og at meldingene er underlagt full filtrering. Hvis noen hevder at det er konfigurert en kobling for dette systemet, stemmer det åpenbart ikke lenger.

**En kobling nevner adresser som det ikke kommer noe fra.** Kandidaten for avvikling. Kontroller før du sletter om det dreier seg om sesongbaserte eller sjeldne systemer, og deaktiver først i stedet for å fjerne med en gang.

**En kobling setter `TreatMessagesAsInternal` eller `CloudServicesMailEnabled` til sann.** Her er det verdt å se nøye etter. Begge innstillingene gjør at meldinger via denne veien behandles som interne i organisasjonen. Hvis e-post fra internett kommer inn denne veien, omgår den dermed kontroller som er ment for eksterne meldinger, blant annet beskyttelsen mot forfalskede avsendere fra eget domene. For en ren hybridkobling er dette riktig; for en kobling som vilkårlige systemer leverer via, er det et funn.

## Hva du typisk finner

Fra praksis, uten å gjøre krav på fullstendighet: en testkobling fra en migrering som har vært aktiv i årevis. Et fagsystem som sender direkte til tjenesten, selv om alle tror det går via gatewayen. En nyhetsbrevleverandør med utløpt kontrakt som fortsatt får levere. Og jevnlig en kobling med svært åpne betingelser, som noen en gang opprettet for å løse et akutt problem.

Ingen av disse funnene er dramatiske i seg selv. Sammen gir de bildet av et miljø som ingen lenger har full oversikt over, og nettopp det er den egentlige risikoen.

## Begrensninger ved metoden

Du bør kjenne til tre begrensninger.

Meldingssporingen går bare rundt ti dager tilbake via cmdleten. For lengre tidsrom trenger du det historiske søket, som kjører asynkront og dekker opptil 90 dager. Ellers går sjeldne systemer som sender månedlig, under radaren.

`FromIP` betyr ikke det samme overalt. For e-post fra internett er det adressen til serveren som leverer. For hybrid-e-post er det adressen til den lokale transportserveren din, ikke den opprinnelige avsenderen. Analysen viser altså **siste stasjon før tjenesten**, ikke opprinnelsen.

Og tilordningen til en kobling er ikke direkte synlig i tenant-en. Du slutter deg til den via adresse, sertifikat og avsenderdomene. For en robust uttalelse om bruken av én enkelt kobling er koblingsrapporten i Exchange Admin Center under Rapporter og e-postflyt en bedre kilde, fordi den aggregeres på serversiden over lengre tidsrom.

## Som tilbakevendende kontroll

Denne analysen egner seg godt som kvartalsrutine. Lagre resultatet og sammenlign ved neste gjennomgang. Nye adresser i listen er enten dokumenterte endringer eller noe du ønsker å vite om.

Når du likevel kontrollerer e-postkonfigurasjonen for domenene dine: [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check) viser SPF, DKIM, DMARC og de øvrige e-poststandardene for vilkårlige domener direkte i nettleseren, også for underdomener og markedsføringsdomener som erfaringsmessig glemmes i slike kartlegginger. Og for selve spørringene leverer [Befehls-Generator](https://rafaelpfister.ch/tools/command-builder) ferdige byggeklosser for PowerShell og Unix-skall.

Hvordan du følger opp enkeltstående mistenkelige meldinger, står i [Analyser Exchange-e-postflyt: Message Tracking, SMTP-protokoller og Receive Connectors](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren).

## Kilder

1.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): Feltliste inkludert FromIP og ToIP samt sideloggikken med StartingRecipientAddress.

2.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch): Asynkron meldingssporing over opptil 90 dager for eldre tidsrom.

3.  [Get-InboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchange/get-inboundconnector): Parametere for innkommende koblinger, blant annet SenderIPAddresses og TreatMessagesAsInternal.

4.  [Configure mail flow using connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow): Samspillet mellom koblingstypene og når hvilken brukes.

5.  [RFC 1918: Address Allocation for Private Internets](https://www.rfc-editor.org/rfc/rfc1918): Definerer de private adresseområdene som du må skille fra offentlige adresser i analysen.
