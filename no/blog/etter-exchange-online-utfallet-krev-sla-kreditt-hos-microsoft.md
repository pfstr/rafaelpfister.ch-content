---
title: "Etter Exchange Online-utfallet: Krev SLA-kreditt fra Microsoft"
navTitle: "SLA-kreditt"
description: "Exchange Online-utfallet 12. og 13. november 2026 varte i rundt 22 timer og ligger dermed for mange leiere under den garanterte tilgjengeligheten på 99,9 prosent. Microsoft gir ikke kreditt automatisk; den må søkes om innen 31. desember 2026. Slik beregner du tilgjengeligheten, hvilken dokumentasjon Microsoft krever, og hvorfor små bedrifter bør gå via partneren sin mens store bedrifter bør henvende seg direkte til Microsoft."
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
translationSourceHash: e2a540fa8b0a15956e2a9f4af6b604f517d4f077ec2d566bd2dd187a1d423216
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T11:24:38.753Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/etter-exchange-online-utfallet-krev-sla-kreditt-hos-microsoft
---

# Etter Exchange Online-utfallet: Krev SLA-kreditt fra Microsoft

<div class="notfall-hinweis notfall-hinweis--gross">
<p class="notfall-hinweis__titel">Kostnadsfri hjelp med SLA-søknaden</p>
<p>Vi hjelper deg kostnadsfritt med å søke om kreditt hos Microsoft eller partneren din for Exchange Online-utfallet 12. og 13. november 2026: beregne tilgjengelighet, samle dokumentasjon og sende inn søknaden. Fristen er 31. desember 2026. Ta kontakt via <a href="https://rafaelpfister.ch/kontakt">kontaktskjemaet</a>.</p>
</div>

Fra 12. november 2026 kl. 18:54 til 13. november 2026 kl. 17:10 (sveitsisk tid) hadde mange organisasjoner ingen eller bare periodisk tilgang til Exchange Online-postboksene sine. Microsoft førte feilen i Admin Center under EX1550543 (Exchange Online) og MO1550821 (andre Microsoft 365-tjenester). Jeg dokumenterte forløpet i [livestudioet om utfallet](https://rafaelpfister.ch/liveticker/exchange-online-ausfall-2026-11). For Exchange Online garanterer Microsoft en månedlig tilgjengelighet på 99,9 prosent. Et utfall på godt over 22 timer ligger klart under dette. Du får imidlertid bare kreditt dersom du søker om den, og fremgangsmåten avhenger av hvordan organisasjonen din har kjøpt Microsoft 365.

## Hva SLA-en garanterer

Det avgjørende er «Service Level Agreement for Microsoft Online Services», som Microsoft publiserer på nytt hver måned. Denne artikkelen bygger på versjonen av 17. november 2026. For Exchange Online definerer SLA-en nedetid som ethvert tidsrom der brukere ikke kan sende eller motta e-post med Outlook på nettet. Det finnes ingen planlagte vedlikeholdsvinduer for Exchange Online.

Tilgjengeligheten beregnes i brukerminutter:

```text
Verfügbarkeit (%) = (Benutzerminuten - Ausfallminuten) / Benutzerminuten x 100
```

Brukerminutter er antall minutter i måneden multiplisert med antall brukere. Nedetidsminutter er varigheten av hver hendelse multiplisert med antall berørte brukere. Dersom tilgjengeligheten faller under en terskel, gis det kreditt på månedsavgiften for den berørte tjenesten:

| Tilgjengelighet i måneden | Kreditt |
|---|---|
| under 99,9 % | 25 % |
| under 99 % | 50 % |
| under 95 % | 100 % |

For Microsoft Teams, SharePoint Online og OneDrive gjelder de samme tersklene, med egne definisjoner av nedetid. For Teams teller det bare dersom brukere ikke kan chatte eller starte møter. Under EX1550543 fungerte chatter og møter ifølge Microsoft i stor grad, mens kalender og invitasjoner var forstyrret. En Teams-søknad vil derfor trolig ha liten utsikt til å lykkes.

I tillegg til postbokstilgjengelighet har Exchange Online flere tjenestenivåer, for eksempel leveringstid innenfor Microsoft 365 og mottak av innkommende e-post. Dersom en hendelse rammer flere av disse, må du ifølge SLA-en velge ett. For novemberutfallet er postbokstilgjengelighet via Outlook på nettet det nærliggende målet.

## Hva utfallet betyr i prosent

November 2026 har 30 dager, altså 43'200 minutter per bruker. Dette gir følgende grenseverdier per bruker:

| Terskel | Tillatt nedetid i november |
|---|---|
| 99,9 % | 43 minutter |
| 99 % | 7 timer og 12 minutter |
| 95 % | 36 timer |

Feilen varte 1'336 minutter fra første melding til den var løst. To regneeksempler for en organisasjon med 200 postbokser:

| Tilfelle | Nedetidsminutter | Tilgjengelighet | Kreditt |
|---|---|---|---|
| Alle 200 brukere uten tilgang under hele feilen | 200 x 1'336 = 267'200 | 96,91 % | 50 % |
| 120 brukere uten tilgang i 10 timer | 120 x 600 = 72'000 | 99,17 % | 25 % |

Det samlede antallet brukerminutter er i begge tilfeller 200 x 43'200 = 8'640'000. Det avgjørende er hvor mange brukere som faktisk var berørt, og hvor lenge. Microsoft kontrollerer dette ved hjelp av egen telemetri, men krever at du oppgir dette i søknaden. Mange leiere i Europa ble først rammet sent på kvelden 12. november eller bare periodevis. Beregn derfor med tidene du kan dokumentere, ikke med den samlede varigheten av Microsoft-meldingen.

## Hva kreditten er – og ikke er

SLA-en regulerer flere punkter som bør være klare før søknaden:

1.  **Kun ved søknad.** Microsoft vurderer og innvilger ikke kreditt på eget initiativ. Ifølge SLA-en er en statusmelding fra Microsoft uttrykkelig ikke en bekreftelse på at et tjenestenivå ikke ble oppfylt.

2.  **Kreditt, ikke refusjon.** Kreditten motregnes mot en fremtidig faktura. Den er begrenset til månedsavgiften for den berørte tjenesten.

3.  **Eneste rettsmiddel.** Ifølge SLA-en er kreditt det eneste rettsmiddelet ved tilgjengelighetsproblemer. Microsoft erstatter ikke tapt omsetning, overtid eller andre følgekostnader. Du kan ikke redusere fakturaer på eget initiativ.

4.  **Kun den berørte tjenesten.** Har du en pakke som Microsoft 365 E3 eller Business Premium, gjelder kreditten andelen for Exchange Online, ikke hele lisensprisen. Microsoft fastsetter denne andelen ved vurderingen; SLA-en oppgir ingen formel for dette.

5.  **Ingen kreditt for gratis- og prøveversjoner.** Prøveversjoner, forhåndsversjoner og gratis tjenester er unntatt, i tillegg til donerte lisenser for ideelle organisasjoner. Privatkunder med Microsoft 365 Personal eller Family omfattes ikke av denne SLA-en.

6.  **Open-avtaler i dager.** De som har lisensiert via Open, Open Value eller Open Value Subscription, får eventuell kreditt som ekstra bruksdager i stedet for et pengebeløp.

7.  **Frist.** For Microsoft 365-tjenester må søknaden med alle opplysninger være mottatt innen utgangen av måneden etter hendelsesmåneden. For novemberutfallet er fristen **31. desember 2026**. Azure har en egen frist på 60 dager.

## Hvilke opplysninger Microsoft krever

SLA-en angir fem obligatoriske opplysninger. Mangler de, avviser Microsoft søknaden:

1.  en nøyaktig beskrivelse av hendelsen,
2.  tidspunkt og varighet for nedetiden,
3.  de berørte ressursene (for Exchange Online: leier og tjeneste),
4.  antall og plasseringer for berørte brukere,
5.  en beskrivelse av feilene som oppstod.

Hendelses-ID-en (EX1550543) og leier-ID-en skal alltid være med. Begge kan hentes med Microsoft Graph PowerShell:

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
| `-Scopes "ServiceHealth.Read.All"` | Tillatelse til å lese meldinger om tjenestetilstand |
| `-Scopes "Organization.Read.All"` | Tillatelse til å lese organisasjonsdata, inkludert leier-ID-en |
| `(Get-MgOrganization).Id` | skriver ut leier-ID-en |
| `-ServiceHealthIssueId "EX1550543"` | henter nøyaktig denne hendelsen |
| `Select-Object …` | viser ID, tittel, start, slutt og klassifisering for hendelsen |

</details>

Klassifiseringen (`Classification`) viser om Microsoft fører hendelsen som `incident` eller som `advisory`. Meldinger på nivået «Advisory» er ifølge Microsofts partnerdokumentasjon som regel ikke kredittberettigede. EX1550543 var klassifisert som Incident.

For antall og plassering av brukere er det tilstrekkelig med en oversikt over lisensierte postbokser etter brukssted:

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
| `All` | leser alle brukere, ikke bare den første siden |
| `Filter 'assignedLicenses/$count ne 0'` | kun brukere med minst én tilordnet lisens |
| `ConsistencyLevel eventual` | nødvendig for `$count`-filteret (utvidet spørring) |
| `CountVariable anzahl` | lagrer totalantallet i variabelen `$anzahl` |
| `Property UsageLocation` | henter også bruksstedet |
| `Group-Object UsageLocation` | teller brukerne per land |

</details>

Dette tallet er øvre grense. Hvor mange brukere som faktisk ikke hadde tilgang, dokumenterer du med egne data: helpdesk-saker med tidsstempel, feilmeldinger fra Outlook og Outlook på nettet (skjermbilder), resultater fra tilgjengelighetstester i overvåkingen din samt påloggings- og tilgangslogger. Jo mer presise disse opplysningene er, desto mindre avhenger resultatet utelukkende av Microsofts telemetri.

## Små bedrifter: som regel via partneren

Små og mellomstore bedrifter i Sveits kjøper vanligvis Microsoft 365 via en partner i Cloud Solution Provider Program (CSP) eller direkte med kredittkort i Admin Center. Hvilken vei som gjelder, ser du i Microsoft 365 Admin Center under **Fakturering > Dine produkter**: Står en partner oppført som kjøpskilde, er avtalen en CSP-avtale.

### Kjøpt via en CSP-partner

Microsoft mottar i dette tilfellet ikke søknaden direkte fra deg. Ifølge Partner Center kan bare CSP-direkteleverandører og indirekte leverandører (distributører) søke om kreditt, ikke indirekte forhandlere. Dersom IT-tjenesteleverandøren din er en indirekte forhandler, går søknaden via denne til distributøren og derfra til Microsoft. Hvert ledd har egne, ofte tidligere frister. Ta derfor kontakt nå, ikke først i slutten av desember.

Partneren din trenger fra deg:

1.  leier-ID-en,
2.  hendelses-ID-en (EX1550543),
3.  tidsrom, antall og plasseringer for de berørte brukerne samt de observerte feilene,
4.  en e-post der du søker om SLA-kreditt. Den må komme fra en adresse i det berørte domenet; Microsoft godtar ikke e-post fra en privat adresse.

Microsoft kjenner selve hendelsen. Søknaden må dokumentere at organisasjonen din var berørt: Navn, leier-ID og saksnummer alene er ifølge Microsoft ikke nok. Dersom kreditten innvilges, krediterer Microsoft partneren, og partneren viderefører den til deg. Om og hvordan dette skjer, reguleres av avtalen din med partneren. Spør dersom ingenting vises på neste faktura.

### Kjøpt direkte fra Microsoft

Dersom du har betalt Microsoft 365 direkte til Microsoft, sender du selv inn søknaden i Admin Center. Dette krever en administratorrolle:

1.  Åpne **Hjelp og støtte** nederst til høyre under `admin.cloud.microsoft`.
2.  Beskriv henvendelsen, for eksempel «Request an SLA service credit for a Microsoft 365 outage», og velg **Kontakt kundestøtte**.
3.  Velg e-post som kontaktmetode. Tittelen skal inneholde «SLA service credit request» og ID-en EX1550543.
4.  Oppgi de fem obligatoriske opplysningene fra forrige avsnitt i beskrivelsen, og legg ved dokumentasjonen.

Det kreves en egen søknad per tjeneste og hendelse. Ifølge SLA-en behandler Microsoft normalt søknader innen 45 dager.

### Lønner det seg for små bedrifter?

For ti brukere er kreditten liten: For en pakke teller bare Exchange-andelen, og av den 25 eller 50 prosent for én måned. Arbeidet er også lite dersom du samler opplysningene nå. Søknaden har en annen fordel: Den dokumenterer overfor partner og ledelse hvor hardt organisasjonen din ble rammet. Det hjelper senere når det skal vurderes om det trengs en reserveordning for e-post.

## Store bedrifter: direkte med Microsoft, med egne måledata

Store organisasjoner lisensierer vanligvis Microsoft 365 via en Enterprise Agreement (EA) eller Microsoft Customer Agreement for Enterprise (MCA-E). Søknaden sendes her direkte til Microsoft, enten via Admin Center som beskrevet ovenfor eller via Unified Support-avtalen. Har du en Customer Success Account Manager (CSAM), bør du informere vedkommende om søknaden; han eller hun kan følge behandlingen. Ved EA gjennom en Licensing Solution Partner (LSP) må du i tillegg avklare hvilken faktura kreditten vil vises på.

Når det er flere tusen brukere, gjelder det betydelige beløp, og Microsoft kontrollerer opplysningene tilsvarende nøye. Forskjellen fra en liten leier ligger i dokumentasjonen:

1.  **Fastslå berørte brukere nøyaktig.** Store leiere er fordelt på flere databaser og regioner i Microsofts datasentre; ikke alle postbokser var forstyrret like lenge. Å oppgi alle brukere som berørte uten nærmere grunnlag er sårbart i store leiere. Måledata er mer holdbare: syntetiske tester mot Outlook på nettet per sted, tilgjengelighetsdata fra egen overvåking, analyser av helpdesk-saker etter sted og tid.

2.  **Oppgi steder.** SLA-en krever plasseringene til de berørte brukerne. For internasjonale organisasjoner lønner det seg å dele opp etter land, fordi feilen først rammet Nord- og Sør-Amerika og senere andre regioner.

3.  **Avgrens hybridmiljøet tydelig.** Forsinkelser i eget datasenter, for eksempel e-poster i kø på lokale Exchange-servere på vei til leieren, er unntatt fra SLA-en. Dokumenter at feilen lå i Microsoft 365. Køene på de lokale serverne er god dokumentasjon på dette, fordi de viser at Exchange Online ikke tok imot e-post eller bare tok den imot forsinket.

4.  **Velg tjenestenivået.** Dersom det hos dere først og fremst var leveringstidene som var berørt, og i mindre grad postbokstilgangen, kan tjenestenivået for leveringstid være et bedre grunnlag. Det er kun tillatt med én søknad per hendelse og tjeneste, så valget bør tas før søknaden sendes.

5.  **Involver innkjøp og lisensadministrasjon.** Kreditten vises på fakturaen, ikke i IT-kostnadsstedet. Avstem med innkjøp hvem som sender søknaden, hvem som kontrollerer kreditten, og hvordan den bokføres.

Store organisasjoner bør dessuten forberede prosessen til neste utfall: en mal med de fem obligatoriske opplysningene, overvåking som måler tilgjengeligheten til Outlook på nettet per sted, og et tydelig ansvar for SLA-søknader.

## Mal for søknaden

Følgende mal inneholder de obligatoriske opplysningene fra SLA-en. Microsoft behandler søknader raskest på engelsk:

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

1.  [Service Level Agreement for Microsoft Online Services (oversikt over versjoner)](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services): Versjon av 17. november 2026, definisjon av nedetid for Exchange Online, kredittrinn, obligatoriske opplysninger, frister og unntak.

2.  [Microsoft Learn: How and when to request a credit from Microsoft](https://learn.microsoft.com/en-us/partner-center/billing/request-credit): Søknadsprosess for CSP-partnere, unntak for indirekte forhandlere, nødvendig dokumentasjon, klassifisering av Advisories.

3.  [Microsoft Learn: Get-MgServiceAnnouncementIssue](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.devices.serviceannouncement/get-mgserviceannouncementissue): Cmdlet for å hente en hendelse for tjenestetilstand via Microsoft Graph.

4.  [Microsoft 365 Status (status.cloud.microsoft)](https://status.cloud.microsoft/): offentlige statusmeldinger om EX1550543 og MO1550821.

5.  [Windows Latest: Microsoft 365 outage, you may get money off your bill](https://www.windowslatest.com/2026/11/13/microsoft-365-outage-you-may-get-money-off-your-bill-if-you-use-it-for-work-but-youll-have-to-ask-microsoft-for-it-as-a-server-failure-takes-down-outlook-teams-and-exchange/): Søknadsprosess i Admin Center for novemberutfallet, merknad om privatkunder.

6.  [Office Watch: Microsoft 365 went down? How to claim your SLA credit](https://office-watch.com/?p=78178): Unntak for private, gratis- og ideelle lisenser, én søknad per tjeneste og hendelse.

7.  [Livestudio: Exchange Online, verdensomspennende utfall i postbokstilgang](https://rafaelpfister.ch/liveticker/exchange-online-ausfall-2026-11): Forløpet av feilen 12. og 13. november 2026 med Microsoft-kronologi.
