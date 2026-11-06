---
title: "Teams-talepost via e-post: ny avsender og nytt emne fra mars 2027 (MC1511072)"
navTitle: "Talepostavsender"
description: "Microsoft endrer leveringen av Teams-talepost via e-post: Avsenderen blir fremover «Microsoft Teams» fra ditt eget leierdomene, og anropsdataene flyttes til emnefeltet. Artikkelen viser hvem som berøres, hvordan du kontrollerer transportregler, tillatelseslister og automatiseringer, og hva som må tilpasses før mars 2027."
date: "2026-11-02"
kategorie: "Microsoft Teams"
timeToRead: "4 min. lesetid"
themen:
  - microsoft-teams
  - microsoft-365-exchange
produkte:
  - "teams"
  - "exchange-online"
protokolle:
  - "smtp"
  - "mail-auth"
  - "powershell"
hauptthema: "microsoft-teams"
slug: "teams-talepost-via-e-post-ny-avsender-og-nytt-emne-fra-mars-2027-mc1511072"
translationId: "article-b75c975f086dd0d5"
translationOf: teams-voicemail-mail-absender
url: https://rafaelpfister.ch/no/blog/teams-talepost-via-e-post-ny-avsender-og-nytt-emne-fra-mars-2027-mc1511072
translationSourceHash: 32089440ce20d39865ec421317bed6588c48eace59eb8f114015c626da412eba
translationModel: gpt-5.6-terra
translatedAt: 2026-11-06T03:19:22.896Z
translationReview: automatic
---

Med Message Center-meldingen MC1511072 av 16. oktober 2026 varsler Microsoft en ny levering av talepost-e-poster fra Teams. I henhold til gjeldende plan starter utrullingen tidlig i mars 2027 og skal være fullført innen utgangen av juni 2027; Microsoft forbeholder seg retten til å endre tidspunktene og vil ettersende bindende datoer samt eventuelle kundetiltak i en ny melding. Bakgrunnen er støtte for egne avsenderdomener (Bring Your Own Domain), slik at talepost-e-postene følger organisasjonens SPF-, DKIM- og DMARC-retningslinjer.

## Hva som endres

«Fra»-feltet vil fremover verken inneholde navn, telefonnummer eller e-postadresse til innringeren. Disse opplysningene vil i stedet stå i emnefeltet og fortsatt i meldingsteksten. I Teams forblir visningen uendret.

| Egenskap | I dag | Etter utrulling |
|---|---|---|
| Avsender (personlig talepost) | Navn eller nummer til innringeren | `"Microsoft Teams" no-reply@<Standard-Tenant-Domain>` eller eget domene, hvis konfigurert |
| Avsenderdomene | `skype.voicemail.microsoft.com` ved PSTN-anrop, leierdomene ved Teams-til-Teams-anrop | Standard leierdomene eller eget domene |
| Emne (personlig talepost) | `Voice mail (22 seconds)` | `Identified: John Smith (+19371817273) - Voicemail (00:22)` eller `Unidentified: +19371817273 - Voicemail (00:22)` |
| Anropskø (delt talepost) | `Shared Voicemail (<CQ-Name>)` | `John Smith (+37251223123) - Shared voicemail in <CQ-Name> (00:05)` |
| Automatisk sentralbord (delt talepost) | `Shared Voicemail (<AA-Name>)` | `John Smith - Shared voicemail in <AA-Name> (00:11)` |

For brukerne får dette to synlige konsekvenser: I innboksen vises alle taleposter med samme avsender, «Microsoft Teams», og fordi adressen er en no-reply-adresse, når ikke «Svar»-funksjonen lenger frem til innringeren.

## Hvem som berøres

Alle organisasjoner som bruker Teams Calling med talepost, berøres, altså Teams Phone med Calling Plan, Operator Connect eller Direct Routing samt anropskøer og automatiske sentralbord med delt talepost. Meldingen nevner ingen lisens- eller skybegrensninger; utrullingen er varslet globalt.

Det er behov for tiltak der noe er avhengig av dagens avsender eller emne:

- Exchange-transportregler som kontrollerer `skype.voicemail.microsoft.com`, innringeren som avsender eller emnet `Voice mail` eller `Shared Voicemail`
- oppføringer i tillatelseslister: Tenant Allow/Block List, spoofing-tillatelser, `AllowedSenderDomains` i antispam-retningslinjer
- innboksregler for brukere og delte postbokser for anropskøer som sorterer eller videresender taleposter basert på avsender eller emne
- Power-Automate-flyter, sakssystemer og CRM-integrasjoner som leser innringerens telefonnummer fra avsenderen
- oppstrøms eller nedstrøms gatewayer (for eksempel ved sentralisert e-postflyt i hybridmiljøer) som inneholder regler eller unntak for talepostdomenet

## Slik kontrollerer du om du er berørt

Følgende spørringer kjører i en Exchange Online-PowerShell-økt (`Connect-ExchangeOnline`).

### Søk i transportregler

```powershell
$muster = 'skype\.voicemail|voice ?mail|shared voicemail'
Get-TransportRule |
  Where-Object { ($_ | Format-List * | Out-String) -match $muster } |
  Select-Object Name, State, Priority
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `$muster` | Regulært uttrykk for dagens avsenderdomene og dagens emnetekster |
| `Format-List *` | Viser alle egenskapene til regelen, slik at betingelser og unntak også gjennomsøkes |
| `Out-String` | Konverterer utdataene til tekst som `-match` brukes på |
| `Select-Object Name, State, Priority` | Viser bare navn, status og rekkefølge for de funne reglene |

</details>

For hver funnet regel kontrollerer du betingelser som `FromAddressContainsWords`, `SenderDomainIs` eller `SubjectContainsWords`.

### Tillatelseslister og antispam-retningslinjer

```powershell
Get-TenantAllowBlockListItems -ListType Sender |
  Where-Object Value -match 'voicemail'
Get-TenantAllowBlockListSpoofItems |
  Where-Object SpoofedUser -match 'voicemail'
Get-HostedContentFilterPolicy |
  Select-Object Name, AllowedSenderDomains, AllowedSenders
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `-ListType Sender` | Leser avsenderoppføringene i Tenant Allow/Block List |
| `Where-Object Value -match 'voicemail'` | Filtrerer oppføringer der verdien inneholder talepostdomenet |
| `Get-TenantAllowBlockListSpoofItems` | Viser spoofing-tillatelser og -blokker |
| `AllowedSenderDomains, AllowedSenders` | Tillatelser direkte i antispam-retningslinjene |

</details>

### Fastslå faktisk e-postvolum

Meldingssporing viser om og hvor taleposter leveres i dag:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddDays(-10) -EndDate (Get-Date) -ResultSize 5000 |
  Where-Object { $_.SenderAddress -like '*@skype.voicemail.microsoft.com' -or $_.Subject -match 'voice ?mail' } |
  Group-Object RecipientAddress | Sort-Object Count -Descending |
  Select-Object Count, Name
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `-StartDate` / `-EndDate` | Søkeperiode; Exchange Online leverer maksimalt de siste 90 dagene, og høyst 10 dager per spørring |
| `-ResultSize 5000` | Maksimalt antall treff per spørring |
| `Where-Object …` | Beholder e-poster fra PSTN-talepostdomenet eller med talepostemne |
| `Group-Object RecipientAddress` | Teller talepostene per mottaker |

</details>

Mottakerlisten viser hvilke postbokser og delte postbokser som mottar taleposter. Der er det verdt å se på innboksregler (`Get-InboxRule -Mailbox <Adresse>`) og tilknyttede flyter.

## Hva som må gjøres

1.  Legg om regler til emne i stedet for avsender: Tilpass nye betingelser til de fremtidige emnemønstrene `- Voicemail (` og `- Shared voicemail in `. Inntil utrullingen har nådd din egen tenant, bør gamle og nye betingelser brukes parallelt.

2.  Fjern tillatelser for `skype.voicemail.microsoft.com` etter utrullingen. E-poster fra eget leierdomene trenger ikke noe unntak; en tillatelse for eget domene vil være en sikkerhetsrisiko fordi den forenkler spoofing.

3.  Tilpass automatiseringer som leser innringerens nummer fra avsenderen. Nummeret vil fremover stå i parentes i emnefeltet, eller ved ukjente innringere rett etter `Unidentified:`.

4.  Forbered eget avsenderdomene: Microsoft varsler at konfigurasjonsveiledningen for sending under eget domene kommer i oppfølgingsmeldingen. Hvis DMARC for dette domenet er satt til `p=reject`, må SPF og DKIM dekke sending fra Exchange Online, ellers havner talepostene i karantene.

5.  Informer brukere og servicedesk: Avsender «Microsoft Teams», innringerdata i emnefeltet, og det er ikke lenger mulig å svare innringeren via e-post.

## Kilder

1.  [MC1511072: Microsoft Teams: Upcoming change to voicemail message delivery by email](https://mc.merill.net/message/MC1511072): arkivert tekst av Message Center-meldingen med tidsplan og sammenligning av dagens og fremtidig avsender og emne.

2.  [Microsoft Learn: Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2): parametere og begrensninger for meldingssporing i Exchange Online.

3.  [Microsoft Learn: Get-TenantAllowBlockListItems](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-tenantallowblocklistitems): lesing av Tenant Allow/Block List.

4.  [Microsoft Learn: Set up voicemail for Teams](https://learn.microsoft.com/en-us/microsoftteams/set-up-phone-system-voicemail): grunnleggende informasjon om Teams-talepost og levering via e-post.
