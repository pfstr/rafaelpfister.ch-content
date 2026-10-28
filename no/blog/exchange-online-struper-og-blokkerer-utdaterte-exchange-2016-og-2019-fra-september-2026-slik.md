---
title: "Exchange Online struper og blokkerer utdaterte Exchange 2016- og 2019-servere fra september 2026: Slik fungerer transporthåndhevingen"
navTitle: "EXO-håndheving 09/2026"
description: "Fra den andre septemberuken i 2026 krever Exchange Online minst SU-en fra oktober 2025 fra hybridservere, ellers blir e-postflyten strupet og senere blokkert. Bakgrunnen for transporthåndhevingen siden 2023, eskaleringstrinnene med SMTP-koder, rapporten i Admin Center, 90-dagerspausen via PowerShell og hvorfor neste heving bare slipper gjennom ESU-kunder og Exchange SE."
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
translationSourceHash: d2bb7589193052361cd6a083ace0d3dbc58552e5e66691c352cd82b693fae6bd
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T09:52:00.388Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/exchange-online-struper-og-blokkerer-utdaterte-exchange-2016-og-2019-fra-september-2026-slik
---

# Exchange Online struper og blokkerer utdaterte Exchange 2016- og 2019-servere fra september 2026: Slik fungerer transporthåndhevingen

Exchange-teamet kunngjorde 2. september 2026 at minsteversjonen for Exchange 2016 og Exchange 2019 i hybrid e-postflyt heves. Fra den andre septemberuken i 2026 krever Exchange Online minst nivået fra den siste offentlig tilgjengelige sikkerhetsoppdateringen fra oktober 2025 fra servere som leverer via en innkommende kobling av typen `OnPremises`. Alt under dette blir strupet og senere blokkert. Kort oppsummert: De som ikke har oppdatert hybridserverne sine siden oktober 2025, mister gradvis e-postlevering til Exchange Online de kommende ukene. Og neste heving, som Microsoft varsler for de kommende månedene, ligger over enhver offentlig tilgjengelig oppdatering: Da oppfyller bare kunder i det betalte ESU-programmet eller miljøer med Exchange Server Subscription Edition (SE) kravet.

Selve kunngjøringen er kort. Hva den betyr i praksis, fremgår av håndhevingssystemet som Microsoft har bygget opp gradvis siden 2023: Hvilke SMTP-svar serveren din får, hvordan du kontrollerer statusen i Admin Center og via PowerShell, og hvilke alternativer som gjenstår i overgangen frem til slutten av ESU-programmet i oktober 2026.

## Hva som gjelder fra den andre septemberuken i 2026

Den nye nedre grensen tilsvarer sikkerhetsoppdateringene fra 14. oktober 2025. Det var siste Patch Tuesday da Microsoft offentlig gjorde oppdateringer tilgjengelige for Exchange 2016 og 2019; alle SU-er siden desember 2025 er kun tilgjengelige gjennom ESU-programmet.

| Versjon | Minstenivå | KB | Build |
|---|---|---|---|
| Exchange 2019 CU15 | SU fra oktober 2025 (CU15 SU5) | KB5066367 | 15.2.1748.39 |
| Exchange 2016 CU23 | SU fra oktober 2025 (CU23 SU19) | KB5066369 | 15.1.2507.61 |

For Exchange 2019 CU14 finnes det også en SU fra oktober 2025 (KB5066368, build 15.2.1544.36). Microsoft-innlegget angir imidlertid uttrykkelig CU15 SU5 som minsteversjon; CU14 har uansett ikke vært en anbefalt versjon siden CU15 ble lansert i februar 2025. Planlegg også oppgraderingen til CU15 dersom du har CU14.

Tre avgrensninger er viktige:

- **Bare hybrid e-postflyt er berørt.** Exchange Online kontrollerer versjonen til leverende servere for meldinger som ankommer via en innkommende kobling av typen `OnPremises`. Dette er den klassiske hybridkonfigurasjonen som Hybrid Configuration Wizard oppretter. E-poster som ankommer via en tredjepartsgateway eller en kobling av typen `Partner` går ikke gjennom denne håndhevingen.
- **Versjonen leses fra meldingshodene.** En Exchange-server skriver build-versjonen sin i `Received`-linjen i hver melding den videresender (`… with Microsoft SMTP Server … id 15.1.2507.61`). Exchange Online evaluerer denne informasjonen. Det er dermed nivået til serveren som faktisk overleverer meldingen til Exchange Online som teller, altså i mange miljøer Edge Transport Server eller Mailbox-serveren med Send Connector til `*.mail.protection.outlook.com`.
- **Exchange SE er ikke berørt.** Håndhevingen gjelder Exchange 2016 og 2019; Exchange Server SE ligger over enhver nedre grense så lenge den oppdateres regelmessig.

## Bakgrunn: transporthåndhevingen siden 2023

Kunngjøringen fra september er ikke et nytt tiltak, men neste trinn i et system som Microsoft presenterte i mars 2023 under tittelen «Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online». Microsoft definerer «persistently vulnerable» som enhver Exchange-server som enten har nådd slutten av supportperioden eller forblir uoppdatert mot kjente sårbarheter. Målet er å beskytte Exchange Online-mottakere mot meldinger fra kompromitterbare servere og samtidig legge press på driftsansvarlige for å oppdatere eller slå av serverne.

Systemet ble aktivert versjon for versjon:

| Tidspunkt | Berørt versjon |
|---|---|
| August 2023 | Exchange 2007 |
| September 2023 | Exchange 2010 |
| Desember 2023 | Exchange 2013 |
| Mars 2024 | Exchange 2016 og 2019 (betydelig utdaterte SU-nivåer) |
| September 2026 | Exchange 2016 og 2019: Nedre grense = SU fra oktober 2025 |
| «om noen måneder» | Exchange 2016 og 2019: Nedre grense over siste offentlige oppdatering |

For Exchange 2016 og 2019 har den nedre grensen hittil ligget ved nivåer som var «significantly behind on security updates». Det nye er at Microsoft setter grensen ved siste offentlige oppdatering, og dermed for første gang rammer servere som for mindre enn ett år siden fortsatt var fullt oppdatert.

## Eskaleringstrinnene

Håndhevingen opererer med tre funksjoner som Microsoft kaller «reporting», «throttling» og «blocking». Så snart en server faller under den nedre grensen, starter en 90-dagers syklus. Trinnene fra grunninnlegget fra 2023:

| Periode | Tiltak | SMTP-svar |
|---|---|---|
| Dag 0 til 30 | Kun rapport i Exchange Admin Center | ingen |
| Dag 30 til 40 | Struping 5 minutter per time | `450 4.7.230` |
| Dag 40 til 50 | Struping 10 minutter per time | `450 4.7.230` |
| Dag 50 til 60 | Struping 20 minutter per time | `450 4.7.230` |
| Dag 60 til 70 | Struping 30 minutter per time, i tillegg blokkering 5 minutter per time | `450 4.7.230` og `550 5.7.230` |
| Dag 70 til 80 | Blokkering 10 minutter per time | `550 5.7.230` |
| Dag 80 til 90 | Blokkering 20 minutter per time | `550 5.7.230` |
| fra dag 90 | Fullstendig blokkering | `550 5.7.230` |

De to svarene har følgende ordlyd:

```text
450 4.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online throttled for n mins/hr.

550 5.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online blocked for n mins/hr.
```

Forskjellen er avgjørende for driften. Ved `450` avviser Exchange Online forbindelsen midlertidig; den lokale serveren beholder meldingen i køen og prøver igjen. Brukerne merker først bare forsinkelser, mens køen til Send Connector mot Exchange Online vokser i Queue Viewer eller i `Get-Queue` med statusen `Retry` og 4.7.230-meldingen som `LastError`. Ved `550` er avvisningen endelig: Avsenderen mottar en NDR med koden 5.7.230, og meldingen er tapt dersom den ikke sendes på nytt. Fordi blokkeringen i starten bare er aktiv noen minutter per time, fremstår feilen først som sporadisk: Noen meldinger kommer frem, mens andre feiler med NDR. Den som ser et slikt mønster i Message Tracking, bør først kontrollere versjonsnivået før det letes etter nettverks- eller sertifikatproblemer.

Kunngjøringen sier ikke om Microsoft starter hele 90-dagerssyklusen fra den andre septemberuken for den nye nedre grensen, eller om den allerede starter på et senere trinn. Grunninnlegget fastslår at systemet fortsetter på det tidligere oppnådde trinnet etter en pause. Ikke stol på 30 dagers frist.

## Rapport i Exchange Admin Center og via PowerShell

Exchange Online lister de oppdagede lokale serverne med versjon i en egen rapport: I Exchange Admin Center under *Reports*, *Mail flow*, rapporten om utdaterte tilkoblede lokale Exchange-servere («out-of-date connecting on-premises Exchange servers»). Rapporten viser registrert build per server, om den ligger under den nedre grensen og hvilket håndhevingstrinn den befinner seg på.

Den samme informasjonen leveres av Exchange Online PowerShell:

```powershell
Connect-ExchangeOnline
Get-OnPremServerReportInfo
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Kommando | Virkning |
|---|---|
| `Connect-ExchangeOnline` | Åpner økten mot Exchange Online (modul `ExchangeOnlineManagement`). |
| `Get-OnPremServerReportInfo` | Lister lokale servere som Exchange Online har oppdaget, med build, håndhevingsstatus og trinn. |

</details>

Rapporten kjenner bare servere som faktisk overleverer e-post til Exchange Online. En administrasjonsserver uten e-postflyt eller en maskin med kun Management Tools vises ikke. Dette er irrelevant for håndhevingen, men ikke for sikkerheten: Også disse systemene trenger SU-ene.

## Pause håndhevingen: 90 dager per år

For miljøer som ikke kan nå den nedre grensen på kort sikt, tilbyr Microsoft en pause. Den kan aktiveres i totalt 90 dager per år, sammenhengende eller i flere perioder:

```powershell
Get-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer

New-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer `
  -NumberOfDays 30
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Get-TenantExemptionInfo` | Viser om og hvor lenge en pause er aktiv for tenant-en. |
| `New-TenantExemptionInfo` | Oppretter en ny pause. |
| `-BlockingScenario UnpatchedOnPremServer` | Velger scenarioet «utdatert lokal server»; det finnes for tiden ingen andre scenarioer for denne cmdleten. |
| `-NumberOfDays 30` | Varighet på pausen i dager. Kvoten er 90 dager per år, og angivelsen trekkes fra denne. |

</details>

To egenskaper ved pausen er viktige i praksis. For det første fortsetter håndhevingen på trinnet den ble stoppet på etter utløp; pausen tilbakestiller ikke 90-dagerssyklusen. For det andre finnes det ingen cmdlet for å avslutte en pågående pause før tiden: Den som oppretter 90 dager og fullfører oppdateringen etter to uker, har brukt opp årskvoten. Opprett derfor pausen så kort som mulig og forleng ved behov.

Pausen er dessuten bare en løsning for den aktuelle nedre grensen. Hvis Microsoft hever grensen over siste offentlige oppdatering om noen måneder, hjelper ikke en oppbrukt kvote lenger.

## Hvorfor neste heving er den egentlige fristen

Exchange 2016 og 2019 har vært utenfor support siden 14. oktober 2025. Microsoft opprettet deretter to betalte ESU-perioder: Periode 1 til april 2026, periode 2 fra mai til oktober 2026. Da periode 2 ble kunngjort 15. april 2026, presiserte Exchange-teamet at det ikke kommer noen ytterligere forlengelse. SU-ene fra desember 2025 til oktober 2026 (senest build 15.2.1748.55 for 2019 CU15 og 15.1.2507.77 for 2016 CU23, begge fra 13. oktober 2026) er utelukkende tilgjengelige for ESU-kunder og tilbys ikke offentlig for nedlasting.

Dette gir følgende situasjon:

- **I dag** oppfyller en server med SU fra oktober 2025 den nedre grensen, med eller uten ESU.
- **Ved neste heving** ligger den nedre grensen ifølge Microsoft over nivået fra oktober 2025. Uten ESU-avtale finnes det ingen lovlig måte å nå dette nivået på. Hybrid e-postflyt fra disse serverne blir da strupet og blokkert, uavhengig av hvor godt resten av miljøet driftes.
- **31. oktober 2026** avsluttes også periode 2. Microsoft publiserte de siste oppdateringene for Exchange 2016 og 2019 med SU-en fra 13. oktober 2026; etter det kommer det ikke flere SU-er, for noen. Senest den påfølgende hevingen vil dermed også ramme ESU-kunder.

ESU-programmet kjøper dermed i beste fall noen måneder. Den eneste varige versjonen som håndhevingen slipper gjennom, er Exchange Server SE. Microsoft har dessuten kunngjort at Exchange SE CU2 (opprinnelig planlagt for andre halvdel av 2026; siden Microsoft for tiden ikke oppgir noen dato for foregående CU1, finnes det heller ingen dato for CU2) avslutter sameksistens med Exchange 2016 og 2019: Installasjonen avbrytes dersom eldre servere oppdages i organisasjonen. Migreringen er derfor ikke bare nødvendig på grunn av e-postflyten, men også for å kunne installere videre oppdateringer for SE.

For miljøer som bare beholder Exchange On-Premises for å administrere attributter i en hybridkonstellasjon, er alternativet å fjerne den siste serveren: Siden Exchange 2019 CU12 kan mottakerattributtene vedlikeholdes med Management Tools uten en kjørende Exchange-server. Da finnes det ikke lenger lokal e-postflyt, og håndhevingen er irrelevant.

## Finne versjonsnivået

`Get-ExchangeServer` viser i `AdminDisplayVersion` bare CU-en, ikke SU-en. Det er pålitelig å kontrollere filversjonen til `ExSetup.exe` eller bruke [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), som i tillegg rapporterer manglende manuelle trinn. For en rask oversikt over alle servere:

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

| Element | Virkning |
|---|---|
| `Get-ExchangeServer` | Lister alle Exchange-servere i organisasjonen. |
| `\\<Server>\C$\...\ExSetup.exe` | Administrativ delingsbane til oppsettsfilen; tilpass ved avvikende installasjonsbane. |
| `VersionInfo.ProductVersion` | Filversjon som tilsvarer installert SU-build (f.eks. `15.1.2507.61`). |

</details>

Hvis versjonen er under `15.2.1748.39` (2019 CU15) eller `15.1.2507.61` (2016 CU23), faller serveren under den nedre grensen fra den andre septemberuken.

## Anbefalt fremgangsmåte

1. **Kartlegg nivået** som beskrevet over, inkludert Edge Transport Server og administrasjonsservere.

2. **Kontroller rapporten i Exchange Online.** `Get-OnPremServerReportInfo` viser hvilke servere Exchange Online faktisk ser, og om et håndhevingstrinn allerede er aktivt. Sammenlign listen med inventaret: Servere som mangler der, leverer ikke via `OnPremises`-koblingen.

3. **Installer minst SU-en fra oktober 2025.** KB5066367 (2019 CU15) og KB5066369 (2016 CU23) er fortsatt offentlig tilgjengelige i Microsoft Download Center. SU-er er kumulative; en server på august 2025-nivå kan oppdateres direkte til oktober 2025. Ved CU14 må du først installere CU15. Etter installasjonen må du starte på nytt, kontrollere tjenestestatus og kjøre Health Checker på nytt.

4. **Bruk pause bare som en overgangsløsning.** Hvis oppdateringen ikke lykkes i første halvdel av september, opprett `New-TenantExemptionInfo` med kort varighet, og ikke betrakt pausen som en planleggingsreserve for neste heving.

5. **Planlegg migrering til Exchange SE.** Uten ESU-avtale er neste heving den harde fristen, med ESU er det 31. oktober 2026. Exchange 2019 CU15 kan oppgraderes på stedet til SE; Exchange 2016 krever omveien via en ny SE-installasjon samt flytting av postbokser eller roller. De som kun drifter Exchange for attributtadministrasjon, fjerner den siste serveren og fortsetter med Management Tools.

## Endringslogg

- **25. oktober 2026:** Lagt til: SU-ene fra 13. oktober 2026 (buildene 15.2.1748.55 og 15.1.2507.77) er de siste oppdateringene for Exchange 2016 og 2019. For Exchange SE CU2 finnes ingen dato på grunn av utsatt CU1.

## Kilder

1.  [Exchange 2016/2019: Throttling and Blocking up to the Final Public Update Baseline – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/exchange-20162019-throttling-and-blocking-up-to-the-final-public-update-baseline/4552717): Kunngjøringen fra 2. september 2026 med den nye nedre grensen (SU fra oktober 2025), startdatoen i den andre septemberuken og merknaden om den kommende hevingen over siste offentlige oppdatering.

2.  [Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/throttling-and-blocking-email-from-persistently-vulnerable-exchange-servers-to-e/3815328): Grunninnlegget fra 2023 med definisjonen «persistently vulnerable», trinnene Reporting, Throttling og Blocking, 90-dagerssyklusen, SMTP-svarene 4.7.230 og 5.7.230 samt utrullingsplanen per versjon.

3.  [How to pause throttling and blocking of out-of-date on-premises Exchange Servers – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/how-to-pause-throttling-and-blocking-of-out-of-date-on-premises-exchange-servers/4007169): Cmdletene `Get-OnPremServerReportInfo`, `Get-TenantExemptionInfo` og `New-TenantExemptionInfo` samt rapporten i Exchange Admin Center og årskvoten på 90 dager.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Build-numrene for SU-ene fra oktober 2025 og de påfølgende ESU-oppdateringene til oktober 2026; inneholder også merknaden om at bare ESU-kunder får SU-er fra desember 2025.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU15: October 14, 2025 (KB5066367) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2019-cu15-october-14-2025-kb5066367-19a1091c-e0b3-4078-be6b-312463063d05): KB-artikkelen om minstenivået for Exchange 2019 CU15.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: October 14, 2025 (KB5066369) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2016-cu23-october-14-2025-kb5066369-8ca2ab99-dfde-4329-896a-3faa677d2603): KB-artikkelen om minstenivået for Exchange 2016 CU23.

7.  [Support for Exchange Server 2016 and Exchange Server 2019 ends today – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/support-for-exchange-server-2016-and-exchange-server-2019-ends-today/4461192): Slutten på support 14. oktober 2025.

8.  [Announcing Exchange 2016 / 2019 Extended Security Update program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-exchange-2016--2019-extended-security-update-program/4433495): Vilkårene for første ESU-periode.

9.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Varighet fra mai til oktober 2026, opplysningen om at det ikke kommer noen ytterligere forlengelse, og tillegget fra 13. oktober 2026 om de siste oppdateringene for Exchange 2016 og 2019.

10. [Exchange Online transport enforcement system explained – CodeTwo Admin's Blog](https://www.codetwo.com/admins-blog/persistently-vulnerable-exchange-server/): Tabellarisk oversikt over de åtte håndhevingstrinnene og utrullingsdatoene per Exchange-versjon; tredjepartskilde.

11. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Kartlegging av CU/SU-nivåer og åpne manuelle trinn.

12. [Where is Exchange SE CU1 anyway? – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/where-is-exchange-se-cu1-anyway/4546837): Exchange-teamet om utsettelsen av Exchange SE CU1 uten måldato.
