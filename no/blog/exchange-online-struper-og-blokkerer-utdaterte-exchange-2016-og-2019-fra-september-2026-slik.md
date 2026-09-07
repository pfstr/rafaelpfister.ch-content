---
title: "Exchange Online struper og blokkerer utdaterte Exchange 2016 og 2019 fra september 2026: Slik fungerer Transport Enforcement"
navTitle: "EXO-Enforcement 09/2026"
description: "Fra den andre uken i september 2026 krever Exchange Online minst oktober 2025-SU fra hybridservere, ellers blir e-postflyten strupet og senere blokkert. Bakgrunn om Transport Enforcement siden 2023, eskaleringstrinnene med SMTP-koder, rapporten i Admin Center, 90-dagers pausen via PowerShell og hvorfor neste heving bare slipper gjennom ESU-kunder og Exchange SE."
date: "2026-09-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "9 min lesetid"
themen:
  - exchange-onprem-hybrid
  - exchange-updates
produkte:
  - "exchange-hybrid"
  - "exchange-online"
  - "hybrid-mailfluss"
  - "exchange-updates"
protokolle:
  - "smtp"
  - "migration"
  - "releases"
slug: "exchange-online-struper-og-blokkerer-utdaterte-exchange-2016-og-2019-fra-september-2026-slik"
translationId: "article-fff0c5efce59ef76"
draft: false
translationOf: exchange-online-transport-enforcement-hybrid-server
url: https://rafaelpfister.ch/no/blog/exchange-online-struper-og-blokkerer-utdaterte-exchange-2016-og-2019-fra-september-2026-slik
translationSourceHash: bd79f97bff8aa7047f498355cc9cd65834e03cb24b8d552a841035216f63187c
translationModel: gpt-5.6-terra
translatedAt: 2026-09-07T08:30:52.921Z
translationReview: automatic
---

# Exchange Online struper og blokkerer utdaterte Exchange 2016 og 2019 fra september 2026: Slik fungerer Transport Enforcement

Exchange-teamet kunngjorde 2. september 2026 at minimumsversjonen for Exchange 2016 og Exchange 2019 i hybrid e-postflyt heves. Fra den andre uken i september 2026 krever Exchange Online minst nivået til den siste offentlige sikkerhetsoppdateringen fra oktober 2025 fra servere som leverer via en innkommende connector av typen `OnPremises`. Alt under dette blir strupet og senere blokkert. Kort oppsummert: De som ikke har patchet hybridserverne sine siden oktober 2025, vil i løpet av de neste ukene gradvis miste e-postlevering til Exchange Online. Og neste heving, som Microsoft varsler for de kommende månedene, ligger over alle offentlig tilgjengelige oppdateringer: Da oppfyller bare kunder i det betalte ESU-programmet eller miljøer med Exchange Server Subscription Edition (SE) kravet.

Selve kunngjøringen er kort. Hva den betyr i praksis, fremgår av enforcement-systemet som Microsoft har bygget opp trinnvis siden 2023: Hvilke SMTP-svar serveren din får, hvordan du kontrollerer statusen i Admin Center og med PowerShell, og hvilke alternativer som gjenstår i overgangen frem til ESU-programmet avsluttes i oktober 2026.

## Hva som gjelder fra den andre uken i september 2026

Den nye nedre grensen tilsvarer sikkerhetsoppdateringene fra 14. oktober 2025. Dette var siste Patch Tuesday da Microsoft offentlig gjorde oppdateringer tilgjengelig for Exchange 2016 og 2019; alle SUs siden desember 2025 er bare tilgjengelige gjennom ESU-programmet.

| Versjon | Minimumsnivå | KB | Build |
|---|---|---|---|
| Exchange 2019 CU15 | Oktober 2025-SU (CU15 SU5) | KB5066367 | 15.2.1748.39 |
| Exchange 2016 CU23 | Oktober 2025-SU (CU23 SU19) | KB5066369 | 15.1.2507.61 |

Det finnes også en oktober 2025-SU for Exchange 2019 CU14 (KB5066368, build 15.2.1544.36). I Microsoft-innlegget er imidlertid CU15 SU5 uttrykkelig angitt som minimumsversjon; CU14 har uansett ikke vært en anbefalt versjon siden CU15 ble lansert i februar 2025. Planlegg også oppgraderingen til CU15 dersom du bruker CU14.

Tre avgrensninger er viktige:

- **Bare hybrid e-postflyt er berørt.** Exchange Online kontrollerer versjonen til serverne som leverer meldinger som kommer inn via en innkommende connector av typen `OnPremises`. Dette er den klassiske hybridkonfigurasjonen som Hybrid Configuration Wizard oppretter. E-poster som mottas via en tredjepartsgateway eller en connector av typen `Partner` går ikke gjennom denne enforcementen.
- **Versjonen leses fra meldingshodene.** En Exchange-server skriver builden sin i `Received`-linjen i hver melding den videresender (`… with Microsoft SMTP Server … id 15.1.2507.61`). Exchange Online evaluerer denne opplysningen. Det er dermed statusen til serveren som faktisk overleverer meldingen til Exchange Online som teller, altså i mange miljøer Edge Transport Server eller Mailbox-serveren med Send Connector til `*.mail.protection.outlook.com`.
- **Exchange SE er ikke berørt.** Enforcementen gjelder Exchange 2016 og 2019; Exchange Server SE ligger over enhver nedre grense så lenge den patches regelmessig.

## Bakgrunn: Transport Enforcement siden 2023

Kunngjøringen fra september er ikke et nytt tiltak, men neste trinn i et system Microsoft presenterte i mars 2023 under tittelen «Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online». Microsoft definerer «persistently vulnerable» som enhver Exchange-server som enten har nådd slutten av støtteperioden eller forblir upatchet for kjente sårbarheter. Målet er å beskytte Exchange Online-mottakere mot meldinger fra servere som kan kompromitteres, samtidig som driftsansvarlige presses til å patche eller slå av serverne.

Systemet ble aktivert per versjon:

| Tidspunkt | Berørt versjon |
|---|---|
| August 2023 | Exchange 2007 |
| September 2023 | Exchange 2010 |
| Desember 2023 | Exchange 2013 |
| Mars 2024 | Exchange 2016 og 2019 (betydelig utdaterte SU-nivåer) |
| September 2026 | Exchange 2016 og 2019: nedre grense = oktober 2025-SU |
| «om noen måneder» | Exchange 2016 og 2019: nedre grense over siste offentlige oppdatering |

For Exchange 2016 og 2019 har den nedre grensen hittil vært versjoner som var «significantly behind on security updates». Det nye er at Microsoft setter grensen til siste offentlige oppdatering, og dermed for første gang rammer servere som for mindre enn ett år siden fortsatt var fullt patchet.

## Eskaleringstrinnene

Enforcementen fungerer med tre funksjoner som Microsoft kaller «reporting», «throttling» og «blocking». Så snart en server faller under den nedre grensen, starter en 90-dagers syklus. Trinnene fra grunninnlegget fra 2023:

| Periode | Tiltak | SMTP-svar |
|---|---|---|
| Dag 0 til 30 | Kun rapport i Exchange Admin Center | ingen |
| Dag 30 til 40 | Struping 5 minutter per time | `450 4.7.230` |
| Dag 40 til 50 | Struping 10 minutter per time | `450 4.7.230` |
| Dag 50 til 60 | Struping 20 minutter per time | `450 4.7.230` |
| Dag 60 til 70 | Struping 30 minutter per time, i tillegg blokkering 5 minutter per time | `450 4.7.230` og `550 5.7.230` |
| Dag 70 til 80 | Blokkering 10 minutter per time | `550 5.7.230` |
| Dag 80 til 90 | Blokkering 20 minutter per time | `550 5.7.230` |
| Fra dag 90 | Fullstendig blokkering | `550 5.7.230` |

De to svarene lyder ordrett:

```text
450 4.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online throttled for n mins/hr.

550 5.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online blocked for n mins/hr.
```

Forskjellen er avgjørende for driften. Ved `450` avviser Exchange Online forbindelsen midlertidig; On-Premises-serveren beholder meldingen i køen og prøver på nytt. Brukere merker først bare forsinkelser, mens køen til Send Connector mot Exchange Online vokser i Queue Viewer eller i `Get-Queue` med statusen `Retry` og 4.7.230-meldingen som `LastError`. Ved `550` er avvisningen endelig: Avsenderen mottar en NDR med koden 5.7.230, og meldingen går tapt dersom den ikke sendes på nytt. Fordi blokkeringen til å begynne med bare er aktiv noen minutter per time, virker feilen først sporadisk: Noen meldinger kommer frem, mens andre feiler med NDR. Den som ser et slikt mønster i Message Tracking, bør først kontrollere versjonsnivået før det letes etter nettverks- eller sertifikatproblemer.

Kunngjøringen sier ikke om Microsoft starter hele 90-dagers syklusen fra den andre uken i september for den nye nedre grensen, eller allerede begynner på et senere trinn. Grunninnlegget fastslår at systemet fortsetter på tidligere oppnådd trinn etter en pause. Ikke stol på 30 dagers respitt.

## Rapport i Exchange Admin Center og med PowerShell

Exchange Online lister opp oppdagede On-Premises-servere med versjon i en egen rapport: I Exchange Admin Center under *Reports*, *Mail flow*, rapporten om utdaterte tilkoblede On-Premises-servere («out-of-date connecting on-premises Exchange servers»). Rapporten viser for hver server oppdaget build, om den ligger under den nedre grensen og hvilket enforcement-trinn den befinner seg på.

Exchange Online PowerShell gir den samme informasjonen:

```powershell
Connect-ExchangeOnline
Get-OnPremServerReportInfo
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Kommando | Effekt |
|---|---|
| `Connect-ExchangeOnline` | Åpner økten mot Exchange Online (modulen `ExchangeOnlineManagement`). |
| `Get-OnPremServerReportInfo` | Lister opp On-Premises-serverne som Exchange Online har oppdaget, med build, enforcement-status og trinn. |

</details>

Rapporten kjenner bare servere som faktisk overleverer e-post til Exchange Online. En administrasjonsserver uten e-postflyt eller en maskin med bare Management Tools vises ikke. Dette er uten betydning for enforcementen, men ikke for sikkerheten: Også disse systemene trenger SUs.

## Sett enforcement på pause: 90 dager per år

Microsoft tilbyr en pause for miljøer som ikke kan nå den nedre grensen på kort sikt. Den kan aktiveres i totalt 90 dager per år, sammenhengende eller fordelt på flere perioder:

```powershell
Get-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer

New-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer `
  -NumberOfDays 30
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `Get-TenantExemptionInfo` | Viser om og hvor lenge en pause er aktiv for tenant-en. |
| `New-TenantExemptionInfo` | Oppretter en ny pause. |
| `-BlockingScenario UnpatchedOnPremServer` | Velger scenarioet «utdatert On-Premises-server»; andre scenarioer finnes for øyeblikket ikke for denne cmdleten. |
| `-NumberOfDays 30` | Varighet på pausen i dager. Kvoten er 90 dager per år, og oppgitt antall trekkes fra denne. |

</details>

To egenskaper ved pausen er viktige i praksis. For det første fortsetter enforcementen etter utløp på trinnet den ble stanset på; pausen tilbakestiller ikke 90-dagers syklusen. For det andre finnes det ingen cmdlet for å avslutte en aktiv pause før tiden: Den som oppretter 90 dager og er ferdig med patchingen etter to uker, har brukt opp årskvoten. Opprett derfor pausen så kort som mulig, og forleng ved behov.

Pausen er dessuten bare en løsning for den gjeldende nedre grensen. Når Microsoft om noen måneder hever grensen over den siste offentlige oppdateringen, hjelper ikke en oppbrukt kvote lenger.

## Hvorfor neste heving er den egentlige fristen

Exchange 2016 og 2019 har vært ute av støtte siden 14. oktober 2025. Microsoft opprettet deretter to betalte ESU-perioder: Periode 1 til april 2026, periode 2 fra mai til oktober 2026. Med kunngjøringen av periode 2 15. april 2026 presiserte Exchange-teamet at det ikke kommer noen ytterligere forlengelse. SUs fra desember 2025 til august 2026 (senest build 15.2.1748.49 for 2019 CU15 og 15.1.2507.72 for 2016 CU23) er kun tilgjengelige for ESU-kunder og tilbys ikke offentlig for nedlasting.

Dette gir følgende situasjon:

- **I dag** oppfyller en server med oktober 2025-SU den nedre grensen, med eller uten ESU.
- **Ved neste heving** ligger den nedre grensen ifølge Microsoft over oktober 2025-nivået. Uten ESU-avtale finnes det ingen lovlig måte å nå dette nivået på. Hybrid e-postflyt fra disse serverne vil da bli strupet og blokkert, uavhengig av hvor godt resten av miljøet driftes.
- **31. oktober 2026** avsluttes også periode 2. Deretter finnes det ikke flere SUs for Exchange 2016 og 2019, for noen. Senest den påfølgende hevingen rammer dermed også ESU-kunder.

ESU-programmet kjøper dermed i beste fall noen få måneder. Det eneste varige nivået som slipper gjennom enforcementen, er Exchange Server SE. Microsoft har dessuten kunngjort at Exchange SE CU2 (planlagt for andre halvår 2026) avslutter sameksistens med Exchange 2016 og 2019: Installasjonen avbrytes dersom eldre servere finnes i organisasjonen. Migreringen er dermed ikke bare nødvendig på grunn av e-postflyten, men også for i det hele tatt å kunne installere videre oppdateringer for SE.

For miljøer som bare beholder Exchange On-Premises for å administrere attributter i en hybridkonfigurasjon, er alternativet å fjerne den siste serveren: Siden Exchange 2019 CU12 kan mottakerattributtene administreres med Management Tools uten en kjørende Exchange-server. Da finnes det ingen On-Premises-e-postflyt lenger, og enforcementen er irrelevant.

## Fastslå versjonsnivået

`Get-ExchangeServer` viser i `AdminDisplayVersion` bare CU, ikke SU. Pålitelig er filversjonen til `ExSetup.exe` eller [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), som også rapporterer manglende manuelle trinn. For en rask oversikt over alle servere:

```powershell
Get-ExchangeServer | ForEach-Object {
  $path = "\\$($_.Name)\C$\Program Files\Microsoft\Exchange Server\V15\bin\ExSetup.exe"
  [pscustomobject]@{
    Server  = $_.Name
    Version = (Get-Item $path).VersionInfo.ProductVersion
  }
}
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Element | Effekt |
|---|---|
| `Get-ExchangeServer` | Lister opp alle Exchange-servere i organisasjonen. |
| `\\<Server>\C$\...\ExSetup.exe` | Administrativ delingsbane til Setup-filen; tilpass ved avvikende installasjonsbane. |
| `VersionInfo.ProductVersion` | Filversjon som tilsvarer installert SU-build (f.eks. `15.1.2507.61`). |

</details>

Dersom versjonen er under `15.2.1748.39` (2019 CU15) eller `15.1.2507.61` (2016 CU23), faller serveren under den nedre grensen fra den andre uken i september.

## Anbefalt fremgangsmåte

1. **Kartlegg nivået** som beskrevet ovenfor, inkludert Edge Transport Server og administrasjonsservere.

2. **Kontroller rapporten i Exchange Online.** `Get-OnPremServerReportInfo` viser hvilke servere Exchange Online faktisk ser, og om et enforcement-trinn allerede er aktivt. Sammenlign listen med inventaret: Servere som mangler der, leverer ikke inn via `OnPremises`-connectoren.

3. **Installer minst oktober 2025-SU.** KB5066367 (2019 CU15) og KB5066369 (2016 CU23) er fortsatt offentlig tilgjengelige i Microsoft Download Center. SUs er kumulative; en server på august 2025-nivå kan oppgraderes direkte til oktober 2025. Ved CU14 må CU15 installeres først. Etter installasjonen: Start på nytt, kontroller tjenestestatusen og kjør Health Checker på nytt.

4. **Bruk pause bare som en overgangsløsning.** Hvis oppdateringen ikke lykkes i første halvdel av september, opprett `New-TenantExemptionInfo` med kort varighet, og ikke forstå pausen som en planleggingsreserve for neste heving.

5. **Planlegg migrering til Exchange SE.** Uten ESU-avtale er neste heving den harde fristen, med ESU er det 31. oktober 2026. Exchange 2019 CU15 kan oppgraderes til SE med en in-place-oppgradering; Exchange 2016 krever omveien via en ny SE-installasjon og flytting av postbokser eller roller. De som kun drifter Exchange for attributtadministrasjon, fjerner den siste serveren og fortsetter å arbeide med Management Tools.

## Kilder

1.  [Exchange 2016/2019: Throttling and Blocking up to the Final Public Update Baseline – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/exchange-20162019-throttling-and-blocking-up-to-the-final-public-update-baseline/4552717): Kunngjøringen fra 2. september 2026 med den nye nedre grensen (oktober 2025-SU), startdatoen i den andre septemberuken og henvisningen til den kommende hevingen over siste offentlige oppdatering.

2.  [Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/throttling-and-blocking-email-from-persistently-vulnerable-exchange-servers-to-e/3815328): Grunninnlegget fra 2023 med definisjonen «persistently vulnerable», trinnene Reporting, Throttling og Blocking, 90-dagers syklusen, SMTP-svarene 4.7.230 og 5.7.230 samt utrullingsplanen per versjon.

3.  [How to pause throttling and blocking of out-of-date on-premises Exchange Servers – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/how-to-pause-throttling-and-blocking-of-out-of-date-on-premises-exchange-servers/4007169): Cmdletene `Get-OnPremServerReportInfo`, `Get-TenantExemptionInfo` og `New-TenantExemptionInfo` sammen med rapporten i Exchange Admin Center og årskvoten på 90 dager.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Build-numrene til oktober 2025-SUene og de påfølgende ESU-oppdateringene frem til august 2026; inneholder også opplysningen om at SUs fra desember 2025 kun gis til ESU-kunder.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU15: October 14, 2025 (KB5066367) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2019-cu15-october-14-2025-kb5066367-19a1091c-e0b3-4078-be6b-312463063d05): KB-artikkelen om minimumsnivået for Exchange 2019 CU15.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: October 14, 2025 (KB5066369) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2016-cu23-october-14-2025-kb5066369-8ca2ab99-dfde-4329-896a-3faa677d2603): KB-artikkelen om minimumsnivået for Exchange 2016 CU23.

7.  [Support for Exchange Server 2016 and Exchange Server 2019 ends today – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/support-for-exchange-server-2016-and-exchange-server-2019-ends-today/4461192): Slutten på støtteperioden 14. oktober 2025.

8.  [Announcing Exchange 2016 / 2019 Extended Security Update program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-exchange-2016--2019-extended-security-update-program/4433495): Vilkårene for den første ESU-perioden.

9.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Varighet fra mai til oktober 2026 og uttalelsen om at det ikke kommer noen ytterligere forlengelse.

10. [Exchange Online transport enforcement system explained – CodeTwo Admin's Blog](https://www.codetwo.com/admins-blog/persistently-vulnerable-exchange-server/): Tabellarisk oversikt over de åtte enforcement-trinnene og utrullingsdatoene per Exchange-versjon; tredjepartskilde.

11. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Kartlegging av CU/SU-nivåer og åpne manuelle trinn.
