---
title: "Exchange Online stryper och blockerar föråldrade Exchange 2016- och 2019-servrar från september 2026: Så fungerar Transport Enforcement"
navTitle: "EXO-Enforcement 09/2026"
description: "Från den andra septemberveckan 2026 kräver Exchange Online minst oktober 2025-SU från hybridservrar, annars stryps e-postflödet och blockeras senare. Bakgrund till Transport Enforcement sedan 2023, eskaleringsstegen med SMTP-koder, rapporten i Admin Center, 90-dagarsuppehållet via PowerShell och varför nästa höjning bara släpper igenom ESU-kunder och Exchange SE."
date: "2026-09-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "9 min. lästid"
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
slug: "exchange-online-stryper-och-blockerar-foraldrade-exchange-2016-och-2019-fran-september-2026-sa"
translationId: "article-fff0c5efce59ef76"
draft: false
translationOf: exchange-online-transport-enforcement-hybrid-server
translationSourceHash: d2bb7589193052361cd6a083ace0d3dbc58552e5e66691c352cd82b693fae6bd
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T09:51:14.069Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/exchange-online-stryper-och-blockerar-foraldrade-exchange-2016-och-2019-fran-september-2026-sa
---

# Exchange Online stryper och blockerar föråldrade Exchange 2016- och 2019-servrar från september 2026: Så fungerar Transport Enforcement

Exchange-teamet meddelade den 2 september 2026 att minimiversionen för Exchange 2016 och Exchange 2019 i hybrid-e-postflödet höjs. Från den andra septemberveckan 2026 kräver Exchange Online minst nivån för den senaste offentliga säkerhetsuppdateringen från oktober 2025 från servrar som levererar via en inkommande anslutningsapp av typen `OnPremises`. Allt under denna nivå stryps och blockeras senare. Kort sagt: Den som inte har patchat sina hybridservrar sedan oktober 2025 kommer stegvis att förlora e-postleveransen till Exchange Online under de kommande veckorna. Och nästa höjning, som Microsoft aviserar för de kommande månaderna, ligger över varje offentligt tillgänglig uppdatering: Då uppfyller endast kunder i det avgiftsbelagda ESU-programmet eller miljöer med Exchange Server Subscription Edition (SE) kravet.

Meddelandet i sig är kort. Vad det innebär i praktiken framgår av Enforcement-systemet som Microsoft gradvis har byggt upp sedan 2023: vilka SMTP-svar din server får, hur du kontrollerar statusen i Admin Center och via PowerShell samt vilka alternativ som återstår under övergången fram till slutet av ESU-programmet i oktober 2026.

## Vad som gäller från den andra septemberveckan 2026

Den nya lägstanivån motsvarar säkerhetsuppdateringarna från den 14 oktober 2025. Det var den sista Patch Tuesday då Microsoft gjorde uppdateringar för Exchange 2016 och 2019 offentligt tillgängliga; alla SU:er sedan december 2025 finns endast via ESU-programmet.

| Version | Lägstanivå | KB | Build |
|---|---|---|---|
| Exchange 2019 CU15 | Oktober 2025-SU (CU15 SU5) | KB5066367 | 15.2.1748.39 |
| Exchange 2016 CU23 | Oktober 2025-SU (CU23 SU19) | KB5066369 | 15.1.2507.61 |

För Exchange 2019 CU14 finns också en oktober 2025-SU (KB5066368, build 15.2.1544.36). I Microsofts inlägg anges dock uttryckligen CU15 SU5 som minimiversion; CU14 är ändå inte längre en rekommenderad nivå sedan CU15 släpptes i februari 2025. Planera även för uppgraderingen till CU15 om du kör CU14.

Tre avgränsningar är viktiga:

- **Endast hybrid-e-postflödet berörs.** Exchange Online kontrollerar versionen på de levererande servrarna för meddelanden som anländer via en inkommande anslutningsapp av typen `OnPremises`. Det är den klassiska hybridkonfiguration som Hybrid Configuration Wizard skapar. E-post som kommer via en tredjepartsgateway eller en anslutningsapp av typen `Partner` passerar inte denna Enforcement.
- **Versionen läses från meddelandehuvudena.** En Exchange-server skriver sin build i raden `Received` för varje meddelande som den vidarebefordrar (`… with Microsoft SMTP Server … id 15.1.2507.61`). Exchange Online utvärderar denna uppgift. Därmed är det versionen på den server som faktiskt överlämnar meddelandet till Exchange Online som räknas, alltså i många miljöer Edge Transport Server eller Mailbox-servrarna med Send Connector till `*.mail.protection.outlook.com`.
- **Exchange SE berörs inte.** Enforcement gäller Exchange 2016 och 2019; Exchange Server SE ligger över varje lägstanivå så länge den patchas regelbundet.

## Bakgrund: Transport Enforcement sedan 2023

Meddelandet i september är ingen ny åtgärd, utan nästa steg i ett system som Microsoft presenterade i mars 2023 under titeln «Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online». Microsoft definierar «persistently vulnerable» som varje Exchange-server som antingen har nått slutet av sin supportperiod eller förblir opatchad för kända sårbarheter. Målet är att skydda Exchange Online-mottagare från meddelanden från servrar som kan komprometteras och samtidigt sätta press på administratörer att patcha eller stänga av servrarna.

Systemet aktiverades per version:

| Tidpunkt | Berörd version |
|---|---|
| Augusti 2023 | Exchange 2007 |
| September 2023 | Exchange 2010 |
| December 2023 | Exchange 2013 |
| Mars 2024 | Exchange 2016 och 2019 (kraftigt föråldrade SU-nivåer) |
| September 2026 | Exchange 2016 och 2019: lägstanivå = oktober 2025-SU |
| «om några månader» | Exchange 2016 och 2019: lägstanivå över den senaste offentliga uppdateringen |

För Exchange 2016 och 2019 har lägstanivån hittills avsett versioner som låg «significantly behind on security updates». Nytt är att Microsoft sätter gränsen vid den senaste offentliga uppdateringen och därmed för första gången träffar servrar som för mindre än ett år sedan fortfarande var helt patchade.

## Eskaleringsstegen

Enforcement arbetar i tre funktioner som Microsoft kallar «reporting», «throttling» och «blocking». Så snart en server hamnar under lägstanivån börjar en 90-dagarscykel. Stegen från grundinlägget från 2023:

| Period | Åtgärd | SMTP-svar |
|---|---|---|
| Dag 0 till 30 | Endast rapport i Exchange Admin Center | inget |
| Dag 30 till 40 | Strypning 5 minuter per timme | `450 4.7.230` |
| Dag 40 till 50 | Strypning 10 minuter per timme | `450 4.7.230` |
| Dag 50 till 60 | Strypning 20 minuter per timme | `450 4.7.230` |
| Dag 60 till 70 | Strypning 30 minuter per timme, dessutom blockering 5 minuter per timme | `450 4.7.230` och `550 5.7.230` |
| Dag 70 till 80 | Blockering 10 minuter per timme | `550 5.7.230` |
| Dag 80 till 90 | Blockering 20 minuter per timme | `550 5.7.230` |
| från dag 90 | Fullständig blockering | `550 5.7.230` |

De två svaren har följande ordalydelse:

```text
450 4.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online throttled for n mins/hr.

550 5.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online blocked for n mins/hr.
```

Skillnaden är avgörande i drift. Vid `450` avvisar Exchange Online anslutningen tillfälligt; den lokala servern behåller meddelandet i sin kö och försöker igen. Användarna märker först bara förseningar, medan kön till Send Connector mot Exchange Online växer i Queue Viewer eller i `Get-Queue` med statusen `Retry` och 4.7.230-meddelandet som `LastError`. Vid `550` är avvisningen slutgiltig: avsändaren får ett NDR med koden 5.7.230, och meddelandet går förlorat om det inte skickas igen. Eftersom blockeringen till en början bara är aktiv några minuter per timme verkar felbilden först sporadisk: en del meddelanden kommer fram, andra misslyckas med NDR. Den som ser ett sådant mönster i Message Tracking bör först kontrollera versionsnivån innan nätverks- eller certifikatproblem undersöks.

Meddelandet säger inte om Microsoft startar hela 90-dagarscykeln från den andra septemberveckan för den nya lägstanivån eller redan börjar på ett senare steg. Grundinlägget anger att systemet fortsätter på den tidigare uppnådda nivån efter ett uppehåll. Räkna därför inte med 30 dagars respit.

## Rapport i Exchange Admin Center och via PowerShell

Exchange Online listar identifierade lokala servrar inklusive version i en egen rapport: i Exchange Admin Center under *Reports*, *Mail flow*, rapporten om föråldrade anslutande lokala Exchange-servrar («out-of-date connecting on-premises Exchange servers»). Rapporten visar identifierad build per server, om den ligger under lägstanivån och i vilket Enforcement-steg den befinner sig.

Samma information tillhandahålls av Exchange Online PowerShell:

```powershell
Connect-ExchangeOnline
Get-OnPremServerReportInfo
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Kommando | Funktion |
|---|---|
| `Connect-ExchangeOnline` | Öppnar sessionen till Exchange Online (modulen `ExchangeOnlineManagement`). |
| `Get-OnPremServerReportInfo` | Listar de lokala servrar som Exchange Online identifierat, med build, Enforcement-status och steg. |

</details>

Rapporten känner endast till servrar som faktiskt överlämnar e-post till Exchange Online. En hanteringsserver utan e-postflöde eller en maskin med endast Management Tools visas inte. Det är irrelevant för Enforcement, men inte för säkerheten: även dessa system behöver SU:erna.

## Pausa Enforcement: 90 dagar per år

För miljöer som inte kan nå lägstanivån på kort sikt erbjuder Microsoft ett uppehåll. Det kan aktiveras sammanlagt 90 dagar per år, i ett stycke eller i flera delar:

```powershell
Get-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer

New-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer `
  -NumberOfDays 30
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `Get-TenantExemptionInfo` | Visar om och hur länge ett uppehåll är aktivt för klientorganisationen. |
| `New-TenantExemptionInfo` | Skapar ett nytt uppehåll. |
| `-BlockingScenario UnpatchedOnPremServer` | Väljer scenariot «föråldrad lokal server»; det finns för närvarande inga andra scenarier för denna cmdlet. |
| `-NumberOfDays 30` | Uppehållets längd i dagar. Kvoten är 90 dagar per år och den angivna tiden dras av från den. |

</details>

Två egenskaper hos uppehållet är viktiga i praktiken. För det första fortsätter Enforcement efter utgången på det steg där det pausades; uppehållet återställer inte 90-dagarscykeln. För det andra finns ingen cmdlet för att avsluta ett pågående uppehåll i förtid: den som skapar 90 dagar och är klar med patchningen efter två veckor har förbrukat årskvoten. Skapa därför uppehållet så kort som möjligt och förläng vid behov.

Uppehållet är dessutom bara en lösning för den aktuella lägstanivån. Om Microsoft om några månader höjer gränsen över den senaste offentliga uppdateringen hjälper inte en förbrukad kvot längre.

## Varför nästa höjning är den verkliga tidsfristen

Exchange 2016 och 2019 har varit out of support sedan den 14 oktober 2025. Microsoft har därefter lanserat två avgiftsbelagda ESU-perioder: period 1 till april 2026, period 2 från maj till oktober 2026. När period 2 tillkännagavs den 15 april 2026 klargjorde Exchange-teamet att någon ytterligare förlängning inte kommer. SU:erna från december 2025 till oktober 2026 (senast build 15.2.1748.55 för 2019 CU15 och 15.1.2507.77 för 2016 CU23, båda från den 13 oktober 2026) är uteslutande tillgängliga för ESU-kunder och erbjuds inte för offentlig nedladdning.

Detta ger följande läge:

- **I dag** uppfyller en server med oktober 2025-SU lägstanivån, med eller utan ESU.
- **Vid nästa höjning** ligger lägstanivån enligt Microsoft över oktober 2025-nivån. Utan ESU-avtal finns ingen laglig väg att nå denna nivå. Hybrid-e-postflödet från dessa servrar kommer då att strypas och blockeras, oberoende av hur väl resten av miljön drivs.
- **Den 31 oktober 2026** slutar även period 2. Microsoft släppte de sista uppdateringarna för Exchange 2016 och 2019 med oktober-SU den 13 oktober 2026; därefter finns inga fler SU:er, för någon. Senast nästa höjning därefter träffar alltså även ESU-kunder.

ESU-programmet köper därmed i bästa fall några månader. Den enda långsiktiga nivå som Enforcement släpper igenom är Exchange Server SE. Microsoft har dessutom meddelat att Exchange SE CU2 (ursprungligen planerad för andra halvåret 2026; eftersom Microsoft för närvarande inte anger något datum för föregående CU1 finns det heller inget datum för CU2) avslutar samexistensen med Exchange 2016 och 2019: installationen avbryts om äldre servrar hittas i organisationen. Migreringen är alltså inte bara nödvändig på grund av e-postflödet, utan också för att över huvud taget kunna installera fortsatta uppdateringar för SE.

För miljöer som endast behåller Exchange On-Premises för att hantera attribut i en hybridkonfiguration är alternativet att ta bort den sista servern: sedan Exchange 2019 CU12 kan mottagarattribut hanteras med Management Tools utan en aktiv Exchange-server. Då finns inget lokalt e-postflöde längre och Enforcement saknar relevans.

## Fastställ versionsnivån

`Get-ExchangeServer` visar i `AdminDisplayVersion` endast CU, inte SU. Tillförlitlig är filversionen för `ExSetup.exe` eller [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), som dessutom rapporterar saknade manuella steg. För en snabb överblick över alla servrar:

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
<summary>Förklaring av alternativ</summary>

| Element | Funktion |
|---|---|
| `Get-ExchangeServer` | Listar alla Exchange-servrar i organisationen. |
| `\\<Server>\C$\...\ExSetup.exe` | Administrativ utdelningssökväg till installationsfilen; anpassa vid avvikande installationssökväg. |
| `VersionInfo.ProductVersion` | Filversion som motsvarar den installerade SU-builden (t.ex. `15.1.2507.61`). |

</details>

Om versionen ligger under `15.2.1748.39` (2019 CU15) respektive `15.1.2507.61` (2016 CU23) hamnar servern under lägstanivån från den andra septemberveckan.

## Rekommenderat tillvägagångssätt

1. **Inventera nivån** enligt beskrivningen ovan, inklusive Edge Transport Server och hanteringsservrar.

2. **Kontrollera rapporten i Exchange Online.** `Get-OnPremServerReportInfo` visar vilka servrar Exchange Online faktiskt ser och om ett Enforcement-steg redan är aktivt. Jämför listan med inventeringen: servrar som saknas där levererar inte via anslutningsappen `OnPremises`.

3. **Installera minst oktober 2025-SU.** KB5066367 (2019 CU15) och KB5066369 (2016 CU23) är fortfarande offentligt tillgängliga i Microsoft Download Center. SU:er är kumulativa; en server på augusti 2025-nivån kan uppgraderas direkt till oktober 2025. Installera först CU15 om du kör CU14. Kontrollera efter installationen omstart och tjänststatus, och kör Health Checker igen.

4. **Använd endast uppehållet som en övergångslösning.** Om uppdateringen inte kan genomföras under första halvan av september, skapa `New-TenantExemptionInfo` med kort varaktighet och betrakta inte uppehållet som en planeringsreserv för nästa höjning.

5. **Schemalägg migreringen till Exchange SE.** Utan ESU-avtal är nästa höjning den hårda tidsfristen, med ESU är det den 31 oktober 2026. Exchange 2019 CU15 kan uppgraderas till SE med en in-place-uppgradering; Exchange 2016 kräver en omväg via en ny SE-installation och flytt av postlådor respektive roller. Den som endast använder Exchange för attributadministration tar bort den sista servern och fortsätter att arbeta med Management Tools.

## Ändringshistorik

- **25 oktober 2026:** Tillagt: Oktober-SU:erna från den 13 oktober 2026 (build 15.2.1748.55 och 15.1.2507.77) är de sista uppdateringarna för Exchange 2016 och 2019. Det finns inget datum för Exchange SE CU2 på grund av det uppskjutna CU1.

## Källor

1.  [Exchange 2016/2019: Throttling and Blocking up to the Final Public Update Baseline – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/exchange-20162019-throttling-and-blocking-up-to-the-final-public-update-baseline/4552717): Meddelandet från den 2 september 2026 med den nya lägstanivån (oktober 2025-SU), startdatumet under den andra septemberveckan och upplysningen om den kommande höjningen över den senaste offentliga uppdateringen.

2.  [Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/throttling-and-blocking-email-from-persistently-vulnerable-exchange-servers-to-e/3815328): Grundinlägget från 2023 med definitionen «persistently vulnerable», stegen Reporting, Throttling och Blocking, 90-dagarscykeln, SMTP-svaren 4.7.230 och 5.7.230 samt utrullningsplanen per version.

3.  [How to pause throttling and blocking of out-of-date on-premises Exchange Servers – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/how-to-pause-throttling-and-blocking-of-out-of-date-on-premises-exchange-servers/4007169): Cmdletarna `Get-OnPremServerReportInfo`, `Get-TenantExemptionInfo` och `New-TenantExemptionInfo`, inklusive rapporten i Exchange Admin Center och årskvoten på 90 dagar.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Buildnummer för oktober 2025-SU:erna och de efterföljande ESU-uppdateringarna till oktober 2026; innehåller även upplysningen att endast ESU-kunder får SU:er från december 2025.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU15: October 14, 2025 (KB5066367) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2019-cu15-october-14-2025-kb5066367-19a1091c-e0b3-4078-be6b-312463063d05): KB-artikeln om lägstanivån för Exchange 2019 CU15.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: October 14, 2025 (KB5066369) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2016-cu23-october-14-2025-kb5066369-8ca2ab99-dfde-4329-896a-3faa677d2603): KB-artikeln om lägstanivån för Exchange 2016 CU23.

7.  [Support for Exchange Server 2016 and Exchange Server 2019 ends today – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/support-for-exchange-server-2016-and-exchange-server-2019-ends-today/4461192): Slutet på supporten den 14 oktober 2025.

8.  [Announcing Exchange 2016 / 2019 Extended Security Update program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-exchange-2016--2019-extended-security-update-program/4433495): Villkor för den första ESU-perioden.

9.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Perioden från maj till oktober 2026, beskedet att ingen ytterligare förlängning följer samt tillägget från den 13 oktober 2026 om de sista uppdateringarna för Exchange 2016 och 2019.

10. [Exchange Online transport enforcement system explained – CodeTwo Admin's Blog](https://www.codetwo.com/admins-blog/persistently-vulnerable-exchange-server/): Tabellöversikt över de åtta Enforcement-stegen och utrullningsdatumen per Exchange-version; tredjepartskälla.

11. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventering av CU/SU-nivåer och öppna manuella steg.

12. [Where is Exchange SE CU1 anyway? – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/where-is-exchange-se-cu1-anyway/4546837): Exchange-teamet om uppskjutningen av Exchange SE CU1 utan måldatum.
