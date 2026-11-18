---
title: "Etter Exchange Online-utfallet: Krev SLA-kreditt hos Microsoft"
navTitle: "SLA-kreditt"
description: "Exchange Online-utfallet 12. og 13. november 2026 varte i rundt 22 timer og ligger dermed for mange leietakere under den garanterte tilgjengeligheten på 99,9 prosent. Microsoft krediterer ikke beløpet automatisk; det må søkes om innen 31. desember 2026. Slik beregner du tilgjengeligheten, hvilken dokumentasjon Microsoft krever, og hvorfor små bedrifter bør gå via partneren sin mens store bedrifter går direkte til Microsoft."
date: "2026-11-18"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min lesetid"
themen:
  - microsoft-365-exchange
produkte:
  - "exchange-online"
protokolle:
  - "lizenzierung"
  - "powershell"
slug: "etter-exchange-online-utfallet-krev-sla-kreditt-hos-microsoft"
translationId: "article-0e96d35615b0d8d9"
translationOf: microsoft-sla-gutschrift-einfordern
url: https://rafaelpfister.ch/no/blog/etter-exchange-online-utfallet-krev-sla-kreditt-hos-microsoft
translationSourceHash: 537caf78f05fd364041d0a3155f020be11b80f4c936d732e61f0a01f443853c1
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T09:25:16.564Z
translationReview: required
---

# Etter Exchange Online-utfallet: Krev SLA-kreditt hos Microsoft

Fra 12. november 2026 kl. 18.54 til 13. november 2026 kl. 17.10 (sveitsisk tid) hadde mange organisasjoner ingen eller bare periodisk tilgang til Exchange Online-postboksene sine. Microsoft førte hendelsen i Admin Center under EX1550543 (Exchange Online) og MO1550821 (andre Microsoft 365-tjenester). Jeg har dokumentert forløpet i [liveoppdateringen om utfallet](https://rafaelpfister.ch/liveticker/exchange-online-ausfall-2026-11). For Exchange Online garanterer Microsoft en månedlig tilgjengelighet på 99,9 prosent. Et utfall på godt over 22 timer ligger betydelig under dette. Du får imidlertid bare en kreditt dersom du søker om den, og fremgangsmåten avhenger av hvordan organisasjonen din har kjøpt Microsoft 365.

## Hva SLA-en garanterer

Det avgjørende dokumentet er «Service Level Agreement for Microsoft Online Services», som Microsoft publiserer på nytt hver måned. Denne artikkelen bygger på versjonen av 17. november 2026. For Exchange Online definerer SLA-en nedetid som enhver periode der brukere med Outlook på nettet ikke kan sende eller motta e-post. Det finnes ingen planlagte vedlikeholdsvinduer for Exchange Online.

Tilgjengeligheten beregnes i brukerminutter:

```text
Verfügbarkeit (%) = (Benutzerminuten - Ausfallminuten) / Benutzerminuten x 100
```

Brukerminutter er antall minutter i måneden multiplisert med antall brukere. Nedetidsminutter er varigheten av hver hendelse multiplisert med antall brukere som ble berørt. Dersom tilgjengeligheten faller under en terskel, gis det en kreditt på månedsavgiften for den berørte tjenesten:

| Tilgjengelighet i måneden | Kreditt |
|---|---|
| under 99,9 % | 25 % |
| under 99 % | 50 % |
| under 95 % | 100 % |

For Microsoft Teams, SharePoint Online og OneDrive gjelder de samme tersklene, men med egne definisjoner av nedetid. For Teams teller det bare når brukere ikke kan chatte eller starte møter. Under EX1550543 fungerte chatter og møter ifølge Microsoft stort sett, mens kalender og invitasjoner var forstyrret. En Teams-søknad har derfor trolig liten utsikt til å lykkes.

Exchange Online har i tillegg til postbokstilgjengelighet andre tjenestenivåer, som leveringstid innen Microsoft 365 og mottak av innkommende e-post. Dersom en hendelse rammer flere av disse, må du ifølge SLA-en velge ett av dem. For november-utfallet er postbokstilgjengeligheten via Outlook på nettet den mest nærliggende målestokken.

## Hva utfallet betyr i prosent

November 2026 har 30 dager, altså 43'200 minutter per bruker. Dette gir følgende terskelverdier per bruker:

| Terskel | Tillatt nedetid i november |
|---|---|
| 99,9 % | 43 minutter |
| 99 % | 7 timer og 12 minutter |
| 95 % | 36 timer |

Forstyrrelsen varte 1'336 minutter fra første melding til den ble løst. To regneeksempler for en organisasjon med 200 postbokser:

| Tilfelle | Nedetidsminutter | Tilgjengelighet | Kreditt |
|---|---|---|---|
| Alle 200 brukere uten tilgang under hele forstyrrelsen | 200 x 1'336 = 267'200 | 96,91 % | 50 % |
| 120 brukere uten tilgang i 10 timer | 120 x 600 = 72'000 | 99,17 % | 25 % |

Det totale antallet brukerminutter er i begge tilfeller 200 x 43'200 = 8'640'000. Det avgjørende er hvor mange brukere som faktisk var berørt, og hvor lenge. Microsoft kontrollerer dette ut fra egen telemetri, men krever opplysningene dine i søknaden. Mange leietakere i Europa ble først berørt sent på kvelden 12. november eller bare periodevis. Beregn derfor med tidene du kan dokumentere, og ikke med den samlede varigheten i Microsoft-meldingen.

## Hva kreditten er og ikke er

SLA-en regulerer noen punkter som bør være klare før du søker:

1.  **Kun etter søknad.** Microsoft vurderer og innvilger ikke kreditter på eget initiativ. En statusmelding fra Microsoft regnes ifølge SLA-en uttrykkelig ikke som en bekreftelse på at et tjenestenivå ikke ble oppfylt.

2.  **Kreditt, ikke tilbakebetaling.** Kreditten motregnes på en fremtidig faktura. Den er begrenset til månedsavgiften for den berørte tjenesten.

3.  **Eneste rettsmiddel.** Ifølge SLA-en er kreditter det eneste rettsmiddelet ved tilgjengelighetsproblemer. Microsoft erstatter ikke tapt omsetning, overtid eller andre følgekostnader. Du kan ikke redusere fakturaer på egen hånd.

4.  **Kun den berørte tjenesten.** Har du en pakke som Microsoft 365 E3 eller Business Premium, gjelder kreditten Exchange Online-andelen, ikke hele lisensprisen. Microsoft fastsetter størrelsen på denne andelen ved vurderingen; SLA-en oppgir ingen formel for dette.

5.  **Ingen kreditt for gratis- og prøvetilbud.** Prøveversjoner, forhåndsvisninger og gratis tjenester er unntatt, det samme gjelder donerte lisenser til ideelle organisasjoner. Privatkunder med Microsoft 365 Personal eller Family omfattes ikke av denne SLA-en.

6.  **Open-avtaler i dager.** De som har lisensiert via Open, Open Value eller Open Value Subscription, får eventuell kreditt som ekstra bruksdager i stedet for et pengebeløp.

7.  **Frist.** For Microsoft 365-tjenester må søknaden med alle opplysninger være mottatt innen utgangen av måneden etter hendelsesmåneden. For november-utfallet er det **31. desember 2026**. Azure har en egen frist på 60 dager.

## Hvilke opplysninger Microsoft krever

SLA-en oppgir fem obligatoriske opplysninger. Mangler de, avviser Microsoft søknaden:

1.  en nøyaktig beskrivelse av hendelsen,
2.  tidspunkt og varighet for nedetiden,
3.  de berørte ressursene (for Exchange Online: leietaker og tjeneste),
4.  antall og lokasjoner for de berørte brukerne,
5.  en beskrivelse av feilene som oppsto.

Hendelses-ID-en (EX1550543) og leietaker-ID-en skal alltid inkluderes. Begge kan hentes med Microsoft Graph PowerShell:

```powershell
Connect-MgGraph -Scopes "ServiceHealth.Read.All", "Organization.Read.All"
(Get-MgOrganization).Id
Get-MgServiceAnnouncementIssue -ServiceHealthIssueId "EX1550543" |
    Select-Object Id, Title, StartDateTime, EndDateTime, Classification
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-Scopes "ServiceHealth.Read.All"` | tillatelse til å lese meldinger om tjenestetilstand |
| `-Scopes "Organization.Read.All"` | tillatelse til å lese organisasjonsdata, inkludert leietaker-ID-en |
| `(Get-MgOrganization).Id` | skriver ut leietaker-ID-en |
| `-ServiceHealthIssueId "EX1550543"` | henter nøyaktig denne hendelsen |
| `Select-Object …` | viser ID, tittel, start, slutt og klassifisering for hendelsen |

</details>

Klassifiseringen (`Classification`) viser om Microsoft fører hendelsen som `incident` eller som `advisory`. Varsler på nivået «Advisory» gir ifølge Microsofts partnerdokumentasjon normalt ikke rett til kreditt. EX1550543 var klassifisert som Incident.

For antall og lokasjon for brukerne er det tilstrekkelig med en oversikt over lisensierte postbokser etter brukssted:

```powershell
Connect-MgGraph -Scopes "User.Read.All"
$abfrage = @{
    All              = $true
    Filter           = 'assignedLicenses/$count ne 0'
    ConsistencyLevel = 'eventual'
    CountVariable    = 'anzahl'
    Property         = 'UsageLocation'
}
Get-MgUser @abfrage |
    Group-Object UsageLocation |
    Select-Object Name, Count
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `$abfrage = @{ … }` | samler parameterne for `Get-MgUser` i en hashtabell (splatting), slik at linjen holdes kort |
| `All` | leser alle brukere, ikke bare første side |
| `Filter 'assignedLicenses/$count ne 0'` | kun brukere med minst én tildelt lisens |
| `ConsistencyLevel eventual` | nødvendig for `$count`-filteret (utvidet spørring) |
| `CountVariable anzahl` | lagrer totalantallet i variabelen `$anzahl` |
| `Property UsageLocation` | henter også bruksstedet |
| `Group-Object UsageLocation` | teller brukerne per land |

</details>

Dette tallet er øvre grense. Hvor mange brukere som faktisk ikke hadde tilgang, dokumenterer du med egne data: helpdesk-saker med tidsstempel, feilmeldinger fra Outlook og Outlook på nettet (skjermbilder), resultater fra tilgjengelighetstester i overvåkingen din samt påloggings- og tilgangslogger. Jo mer presise disse opplysningene er, desto mindre avhenger resultatet bare av Microsofts telemetri.

## Små bedrifter: vanligvis via partneren

Små og mellomstore bedrifter kjøper vanligvis Microsoft 365 i Sveits via en partner i Cloud Solution Provider Program (CSP) eller direkte med kredittkort i Admin Center. Hvilken vei som gjelder, ser du i Microsoft 365 Admin Center under **Fakturering > Produktene dine**: Dersom en partner står oppført som kjøpskilde, er avtalen en CSP-avtale.

### Kjøpt via en CSP-partner

Microsoft mottar i dette tilfellet ikke søknaden fra deg. Ifølge Partner Center kan bare CSP-direkteleverandører og indirekte leverandører (distributører) søke om kreditter, ikke indirekte forhandlere. Dersom IT-leverandøren din er en indirekte forhandler, går søknaden derfor via denne til distributøren og derfra til Microsoft. Hvert ledd har egne, ofte tidligere frister. Ta derfor kontakt nå, og ikke først i slutten av desember.

Partneren din trenger fra deg:

1.  leietaker-ID-en,
2.  hendelses-ID-en (EX1550543),
3.  tidsrom, antall og lokasjoner for de berørte brukerne samt de observerte feilene,
4.  en e-post der du søker om SLA-kreditten. Den må komme fra en adresse i det berørte domenet; Microsoft godtar ikke e-post fra en privat adresse.

Microsoft kjenner selve hendelsen. Søknaden må dokumentere at organisasjonen din var berørt: Navn, leietaker-ID og saksnummer alene er ifølge Microsoft ikke tilstrekkelig. Dersom kreditten innvilges, krediterer Microsoft partneren, og partneren gir den videre til deg. Om og hvordan dette gjøres, reguleres av avtalen din med partneren. Spør dersom ingenting vises på neste faktura.

### Kjøpt direkte hos Microsoft

Dersom du har betalt Microsoft 365 direkte til Microsoft, søker du selv i Admin Center. Dette krever en administratorrolle:

1.  Åpne **Hjelp og støtte** nederst til høyre under `admin.cloud.microsoft`.
2.  Beskriv henvendelsen, for eksempel «Request an SLA service credit for a Microsoft 365 outage», og velg **Kontakt støtte**.
3.  Velg e-post som kontaktmåte. Tittelen skal inneholde «SLA service credit request» og ID-en EX1550543.
4.  Oppgi de fem obligatoriske opplysningene fra forrige avsnitt i beskrivelsen, og legg ved dokumentasjonen.

Det kreves en separat søknad per tjeneste og hendelse. Ifølge SLA-en behandler Microsoft vanligvis søknader innen 45 dager.

### Lønner det seg for små bedrifter

For ti brukere er kreditten liten: For en pakke teller bare Exchange Online-andelen, og av den 25 eller 50 prosent for én måned. Innsatsen er også liten dersom du samler opplysningene nå. Søknaden har en annen fordel: Den dokumenterer overfor partneren og ledelsen hvor sterkt organisasjonen din ble berørt. Det hjelper senere når dere vurderer om dere trenger en reserveordning for e-post.

## Store bedrifter: direkte med Microsoft, med egne måledata

Store organisasjoner lisensierer vanligvis Microsoft 365 via en Enterprise Agreement (EA) eller Microsoft Customer Agreement for Enterprise (MCA-E). Søknaden går her direkte til Microsoft, enten gjennom Admin Center som beskrevet over eller gjennom Unified Support-avtalen. Har du en Customer Success Account Manager (CSAM), informer vedkommende om søknaden; han eller hun kan følge behandlingen. Ved en EA via en Licensing Solution Partner (LSP) bør du i tillegg avklare hvilken faktura kreditten vises på.

Ved flere tusen brukere dreier det seg om betydelige beløp, og Microsoft gransker derfor opplysningene tilsvarende nøye. Forskjellen fra en liten leietaker ligger i dokumentasjonen:

1.  **Fastslå berørte brukere nøyaktig.** Store leietakere er fordelt over flere databaser og regioner i Microsofts datasentre, og ikke alle postbokser var forstyrret like lenge. Å oppgi alle brukere som berørt er sårbart ved store leietakere. Måledata er mer holdbare: syntetiske tester mot Outlook på nettet per lokasjon, tilgjengelighetsdata fra egen overvåking, analyser av helpdesk-saker etter lokasjon og tid.

2.  **Oppgi lokasjoner.** SLA-en krever lokasjonene til de berørte brukerne. For internasjonale organisasjoner lønner det seg å dele opp per land, fordi forstyrrelsen først rammet Nord- og Sør-Amerika og først senere andre regioner.

3.  **Avgrens hybridløsninger tydelig.** Forsinkelser i eget datasenter, for eksempel e-poster i kø på lokale Exchange-servere på vei til leietakeren, er unntatt fra SLA-en. Dokumenter at feilen lå i Microsoft 365. Køene på de lokale serverne er et godt bevis på dette, fordi de viser at Exchange Online ikke tok imot e-post eller bare tok den imot forsinket.

4.  **Velg tjenestenivå.** Dersom det hos dere først og fremst var leveringstidene som ble berørt, og postbokstilgangen i mindre grad, kan tjenestenivået for leveringstid være et bedre grunnlag. Det er bare tillatt med én søknad per hendelse og tjeneste, så valget bør tas før søknaden sendes.

5.  **Involver innkjøp og lisensadministrasjon.** Kreditten vises på fakturaen, ikke i IT-kostnadsstedet. Avklar med innkjøp hvem som sender søknaden, hvem som kontrollerer kreditten og hvordan den bokføres.

Store organisasjoner bør dessuten forberede prosessen for neste utfall: en mal med de fem obligatoriske opplysningene, overvåking som måler tilgjengeligheten til Outlook på nettet per lokasjon, og et tydelig ansvar for SLA-søknader.

## Mal for søknaden

Følgende mal inneholder de obligatoriske opplysningene fra SLA-en. Microsoft behandler søknader på engelsk raskest:

```text
Subject: SLA service credit request: EX1550543, Exchange Online

Tenant ID: <Tenant-ID>
Service: Exchange Online
Incident ID: EX1550543 (related: MO1550821)
Service level: Exchange Online availability (Outlook on the web)

Description of the incident:
Users were unable to send or receive email using Outlook on the web,
Outlook for Windows and mobile clients.

Time and duration of downtime:
From 2026-11-12 <hh:mm> UTC to 2026-11-13 <hh:mm> UTC (<n> minutes).

Affected users:
<n> of <total> licensed users, locations: <Land: Anzahl, ...>

Errors observed:
<Fehlermeldungen, z. B. Verbindungsfehler in Outlook, HTTP 5xx in Outlook on the web>

Uptime calculation (November 2026):
(<total> x 43,200 - <Ausfallminuten>) / (<total> x 43,200) x 100 = <xx.xx> %

Attachments: helpdesk ticket export, monitoring data, screenshots.
```

## Kilder

1.  [Service Level Agreement for Microsoft Online Services (oversikt over versjoner)](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services): versjon av 17. november 2026, definisjon av nedetid for Exchange Online, kredittrinn, obligatoriske opplysninger, frister og unntak.

2.  [Microsoft Learn: How and when to request a credit from Microsoft](https://learn.microsoft.com/en-us/partner-center/billing/request-credit): søknadsprosess for CSP-partnere, unntak for indirekte forhandlere, nødvendig dokumentasjon, klassifisering av Advisories.

3.  [Microsoft Learn: Get-MgServiceAnnouncementIssue](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.devices.serviceannouncement/get-mgserviceannouncementissue): cmdlet for å hente en tjenestetilstandshendelse via Microsoft Graph.

4.  [Microsoft 365 Status (status.cloud.microsoft)](https://status.cloud.microsoft/): offentlige statusmeldinger om EX1550543 og MO1550821.

5.  [Windows Latest: Microsoft 365 outage, you may get money off your bill](https://www.windowslatest.com/2026/11/13/microsoft-365-outage-you-may-get-money-off-your-bill-if-you-use-it-for-work-but-youll-have-to-ask-microsoft-for-it-as-a-server-failure-takes-down-outlook-teams-and-exchange/): søknadsprosess i Admin Center for november-utfallet, merknad om privatkunder.

6.  [Office Watch: Microsoft 365 went down? How to claim your SLA credit](https://office-watch.com/?p=78178): unntak for privat-, gratis- og ideelle lisenser, én søknad per tjeneste og hendelse.

7.  [Liveoppdatering: Exchange Online, globalt utfall i postbokstilgang](https://rafaelpfister.ch/liveticker/exchange-online-ausfall-2026-11): forløpet av forstyrrelsen 12. og 13. november 2026 med Microsoft-kronologi.
