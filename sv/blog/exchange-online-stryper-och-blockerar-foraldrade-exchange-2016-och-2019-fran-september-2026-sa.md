---
title: "Exchange Online stryper och blockerar föråldrade Exchange 2016 och 2019 från september 2026: Så fungerar transportenforcement"
navTitle: "EXO-enforcement 09/2026"
description: "Från och med den andra septemberveckan 2026 kräver Exchange Online att hybridservrar minst har SU från oktober 2025, annars stryps och senare blockeras e-postflödet. Bakgrund om transportenforcement sedan 2023, eskaleringsnivåerna med SMTP-koder, rapporten i Admin Center, 90 dagars paus via PowerShell och varför nästa höjning endast släpper igenom ESU-kunder och Exchange SE."
date: "2026-09-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "9 min. läsning"
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
url: https://rafaelpfister.ch/sv/blog/exchange-online-stryper-och-blockerar-foraldrade-exchange-2016-och-2019-fran-september-2026-sa
translationSourceHash: bd79f97bff8aa7047f498355cc9cd65834e03cb24b8d552a841035216f63187c
translationModel: gpt-5.6-terra
translatedAt: 2026-09-07T08:29:59.138Z
translationReview: automatic
---

# Exchange Online stryper och blockerar föråldrade Exchange 2016 och 2019 från september 2026: Så fungerar transportenforcement

Exchange-teamet meddelade den 2 september 2026 att minimiversionen för Exchange 2016 och Exchange 2019 i hybrid-e-postflödet höjs. Från och med den andra septemberveckan 2026 kräver Exchange Online att servrar som levererar via en inkommande connector av typen `OnPremises` minst har nivån från den senaste offentliga säkerhetsuppdateringen från oktober 2025. Allt under detta stryps och blockeras senare. Kort sammanfattat: Den som inte har patchat sina hybridservrar sedan oktober 2025 förlorar stegvis e-postleveransen till Exchange Online under de kommande veckorna. Och nästa höjning, som Microsoft aviserar för de kommande månaderna, ligger över alla offentligt tillgängliga uppdateringar: då uppfyller endast kunder i det avgiftsbelagda ESU-programmet eller miljöer med Exchange Server Subscription Edition (SE) kravet.

Själva tillkännagivandet är kort. Vad det innebär i praktiken framgår av enforcement-systemet som Microsoft successivt har byggt upp sedan 2023: vilka SMTP-svar din server får se, hur du kontrollerar statusen i Admin Center och med PowerShell samt vilka alternativ som återstår under övergången fram till slutet av ESU-programmet i oktober 2026.

## Vad som gäller från och med den andra septemberveckan 2026

Den nya miniminivån motsvarar säkerhetsuppdateringarna från den 14 oktober 2025. Det var den sista patchdagen då Microsoft offentliggjorde uppdateringar för Exchange 2016 och 2019; alla SU:er sedan december 2025 finns endast via ESU-programmet.

| Version | Miniminivå | KB | Build |
|---|---|---|---|
| Exchange 2019 CU15 | SU från oktober 2025 (CU15 SU5) | KB5066367 | 15.2.1748.39 |
| Exchange 2016 CU23 | SU från oktober 2025 (CU23 SU19) | KB5066369 | 15.1.2507.61 |

För Exchange 2019 CU14 finns också ett SU från oktober 2025 (KB5066368, build 15.2.1544.36). I Microsofts inlägg anges dock uttryckligen CU15 SU5 som minimiversion; CU14 har ändå inte varit en rekommenderad nivå sedan CU15 släpptes i februari 2025. Planera även för uppgraderingen till CU15 om du använder CU14.

Tre avgränsningar är viktiga:

- **Endast hybrid-e-postflödet påverkas.** Exchange Online kontrollerar versionen på de levererande servrarna för meddelanden som kommer via en inkommande connector av typen `OnPremises`. Detta är den klassiska hybridkonfigurationen som Hybrid Configuration Wizard skapar. E-post som kommer via en tredjepartsgateway eller en connector av typen `Partner` omfattas inte av denna enforcement.
- **Versionen läses från meddelandehuvudena.** En Exchange-server skriver sitt buildnummer i raden `Received` i varje meddelande som den vidarebefordrar (`… with Microsoft SMTP Server … id 15.1.2507.61`). Exchange Online utvärderar denna uppgift. Därmed är det versionen på servern som faktiskt överlämnar meddelandet till Exchange Online som räknas, alltså i många miljöer Edge Transport Server eller Mailbox-servern med Send Connector till `*.mail.protection.outlook.com`.
- **Exchange SE påverkas inte.** Enforcement gäller Exchange 2016 och 2019; Exchange Server SE ligger över varje miniminivå så länge den patchas regelbundet.

## Bakgrund: transportenforcement sedan 2023

Meddelandet från september är ingen ny åtgärd, utan nästa steg i ett system som Microsoft presenterade i mars 2023 under titeln «Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online». Microsoft definierar «persistently vulnerable» som varje Exchange-server som antingen har nått slutet av sin supportperiod eller förblir opatchad för kända sårbarheter. Målet är att skydda Exchange Online-mottagare från meddelanden från servrar som kan komprometteras och samtidigt sätta press på operatörerna att patcha eller stänga av servrarna.

Systemet har aktiverats versionsvis:

| Tidpunkt | Berörd version |
|---|---|
| Augusti 2023 | Exchange 2007 |
| September 2023 | Exchange 2010 |
| December 2023 | Exchange 2013 |
| Mars 2024 | Exchange 2016 och 2019 (kraftigt föråldrade SU-nivåer) |
| September 2026 | Exchange 2016 och 2019: miniminivå = SU från oktober 2025 |
| «om några månader» | Exchange 2016 och 2019: miniminivå över den senaste offentliga uppdateringen |

För Exchange 2016 och 2019 har miniminivån hittills gällt nivåer som låg «significantly behind on security updates». Det nya är att Microsoft sätter gränsen vid den senaste offentliga uppdateringen och därmed för första gången träffar servrar som var fullt patchade för mindre än ett år sedan.

## Eskaleringsnivåerna

Enforcement arbetar i tre funktioner som Microsoft kallar «reporting», «throttling» och «blocking». Så snart en server hamnar under miniminivån börjar en 90-dagarscykel. Nivåerna från grundläggande inlägget från 2023:

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

De två svaren lyder ordagrant:

```text
450 4.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online throttled for n mins/hr.

550 5.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online blocked for n mins/hr.
```

Skillnaden är avgörande för driften. Vid `450` avvisar Exchange Online anslutningen tillfälligt; den lokala servern behåller meddelandet i sin kö och försöker igen. Användare märker till en början endast fördröjningar, medan kön till Send Connector mot Exchange Online växer i Queue Viewer eller i `Get-Queue` med statusen `Retry` och 4.7.230-meddelandet som `LastError`. Vid `550` är avvisningen slutgiltig: avsändaren får en NDR med koden 5.7.230 och meddelandet är förlorat om det inte skickas på nytt. Eftersom blockeringen till en början endast är aktiv några minuter per timme framstår felbilden först som sporadisk: vissa meddelanden kommer fram, andra misslyckas med NDR. Den som ser ett sådant mönster i Message Tracking bör först kontrollera versionsnivån innan nätverks- eller certifikatproblem undersöks.

Om Microsoft startar den fulla 90-dagarscykeln från den andra septemberveckan för den nya miniminivån, eller redan börjar på en senare nivå, framgår inte av tillkännagivandet. Det grundläggande inlägget anger att systemet fortsätter på den tidigare uppnådda nivån efter en paus. Räkna därför inte med 30 dagars respit.

## Rapport i Exchange Admin Center och via PowerShell

Exchange Online listar identifierade lokala servrar och deras versioner i en särskild rapport: i Exchange Admin Center under *Reports*, *Mail flow*, rapporten om föråldrade anslutande lokala Exchange-servrar («out-of-date connecting on-premises Exchange servers»). Rapporten visar identifierat buildnummer per server, om den ligger under miniminivån och vilket enforcement-steg som är aktivt.

Samma information tillhandahålls av Exchange Online PowerShell:

```powershell
Connect-ExchangeOnline
Get-OnPremServerReportInfo
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Kommando | Effekt |
|---|---|
| `Connect-ExchangeOnline` | Öppnar sessionen till Exchange Online (modul `ExchangeOnlineManagement`). |
| `Get-OnPremServerReportInfo` | Listar de lokala servrar som Exchange Online identifierat med build, enforcement-status och nivå. |

</details>

Rapporten känner endast till servrar som faktiskt överlämnar e-post till Exchange Online. En hanteringsserver utan e-postflöde eller en dator med enbart Management Tools visas inte. Detta saknar betydelse för enforcement, men inte för säkerheten: även dessa system behöver SU:erna.

## Pausa enforcement: 90 dagar per år

För miljöer som kortsiktigt inte kan nå miniminivån erbjuder Microsoft en paus. Den kan aktiveras sammanlagt 90 dagar per år, i ett sammanhängande intervall eller i flera delar:

```powershell
Get-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer

New-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer `
  -NumberOfDays 30
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `Get-TenantExemptionInfo` | Visar om och hur länge en paus är aktiv för tenantet. |
| `New-TenantExemptionInfo` | Skapar en ny paus. |
| `-BlockingScenario UnpatchedOnPremServer` | Väljer scenariot «föråldrad lokal server»; det finns för närvarande inga andra scenarier för denna cmdlet. |
| `-NumberOfDays 30` | Pausens längd i dagar. Kvoten är 90 dagar per år och angivet antal dras från den. |

</details>

Två egenskaper hos pausen är viktiga i praktiken. För det första fortsätter enforcement efter utgången på den nivå där det stoppades; pausen återställer inte 90-dagarscykeln. För det andra finns ingen cmdlet för att avsluta en pågående paus i förtid: den som skapar 90 dagar och patchar klart efter två veckor har förbrukat årskvoten. Skapa därför pausen så kort som möjligt och förläng vid behov.

Pausen är dessutom endast en lösning för den aktuella miniminivån. Om Microsoft om några månader höjer gränsen över den senaste offentliga uppdateringen hjälper inte en förbrukad kvot längre.

## Varför nästa höjning är den verkliga tidsfristen

Exchange 2016 och 2019 har varit out of support sedan den 14 oktober 2025. Microsoft har därefter lanserat två avgiftsbelagda ESU-perioder: period 1 till april 2026, period 2 från maj till oktober 2026. När period 2 tillkännagavs den 15 april 2026 klargjorde Exchange-teamet att ingen ytterligare förlängning kommer.

SU:erna från december 2025 till augusti 2026 (senast build 15.2.1748.49 för 2019 CU15 och 15.1.2507.72 för 2016 CU23) är endast tillgängliga för ESU-kunder och erbjuds inte för offentlig nedladdning.

Detta innebär följande situation:

- **I dag** uppfyller en server med SU från oktober 2025 miniminivån, med eller utan ESU.
- **Vid nästa höjning** ligger miniminivån enligt Microsoft över nivån från oktober 2025. Utan ESU-avtal finns ingen laglig väg att nå denna nivå. Hybrid-e-postflödet för dessa servrar kommer då att strypas och blockeras, oavsett hur väl resten av miljön drivs.
- **Den 31 oktober 2026** upphör även period 2. Därefter kommer det inte längre några SU:er för Exchange 2016 och 2019, för någon. Senast nästa höjning efter den kommande träffar alltså även ESU-kunder.

ESU-programmet köper därmed i bästa fall några månader. Den enda långsiktiga nivå som släpps igenom av enforcement är Exchange Server SE. Microsoft har dessutom meddelat att Exchange SE CU2 (planerad för andra halvåret 2026) avslutar samexistens med Exchange 2016 och 2019: installationen avbryts om äldre servrar hittas i organisationen. Migreringen är alltså nödvändig inte bara på grund av e-postflödet, utan även för att över huvud taget kunna fortsätta installera uppdateringar för SE.

För miljöer som endast behåller Exchange On-Premises för att hantera attribut i en hybridkonfiguration är alternativet att ta bort den sista servern: sedan Exchange 2019 CU12 kan mottagarattribut hanteras med Management Tools utan en körande Exchange-server. Då finns inget lokalt e-postflöde längre och enforcement saknar relevans.

## Fastställ versionsnivån

`Get-ExchangeServer` visar i `AdminDisplayVersion` endast CU, inte SU. Tillförlitligt är filversionen för `ExSetup.exe` eller [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), som dessutom rapporterar saknade manuella steg. För en snabb överblick över alla servrar:

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
<summary>Förklarade alternativ</summary>

| Element | Effekt |
|---|---|
| `Get-ExchangeServer` | Listar alla Exchange-servrar i organisationen. |
| `\\<Server>\C$\...\ExSetup.exe` | Administrativ utdelningssökväg till installationsfilen; anpassa vid avvikande installationssökväg. |
| `VersionInfo.ProductVersion` | Filversion som motsvarar installerat SU-build (t.ex. `15.1.2507.61`). |

</details>

Om versionen är lägre än `15.2.1748.39` (2019 CU15) respektive `15.1.2507.61` (2016 CU23) hamnar servern under miniminivån från och med den andra septemberveckan.

## Rekommenderat tillvägagångssätt

1. **Inventera nivån** enligt beskrivningen ovan, inklusive Edge Transport Server och hanteringsservrar.

2. **Kontrollera rapporten i Exchange Online.** `Get-OnPremServerReportInfo` visar vilka servrar Exchange Online faktiskt ser och om en enforcement-nivå redan är aktiv. Jämför listan med inventeringen: servrar som saknas där levererar inte via connectorn `OnPremises`.

3. **Installera minst SU från oktober 2025.** KB5066367 (2019 CU15) och KB5066369 (2016 CU23) finns fortfarande offentligt tillgängliga i Microsoft Download Center. SU:er är kumulativa; en server på nivån från augusti 2025 kan uppdateras direkt till oktober 2025. Installera först CU15 om du använder CU14. Starta om efter installationen, kontrollera tjänststatus och kör Health Checker igen.

4. **Använd endast pausen som en tillfällig lösning.** Om uppdateringen inte kan genomföras under första halvan av september, skapa `New-TenantExemptionInfo` med kort löptid och betrakta inte pausen som en planeringsreserv inför nästa höjning.

5. **Schemalägg migreringen till Exchange SE.** Utan ESU-avtal är nästa höjning den hårda tidsfristen, med ESU är det den 31 oktober 2026. Exchange 2019 CU15 kan uppgraderas på plats till SE; Exchange 2016 kräver omvägen via en nyinstallation av SE samt flytt av postlådor respektive roller. Den som endast använder Exchange för attributadministration tar bort den sista servern och fortsätter arbeta med Management Tools.

## Källor

1.  [Exchange 2016/2019: Throttling and Blocking up to the Final Public Update Baseline – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/exchange-20162019-throttling-and-blocking-up-to-the-final-public-update-baseline/4552717): Tillkännagivandet från den 2 september 2026 med den nya miniminivån (SU från oktober 2025), startdatumet under den andra septemberveckan och informationen om den kommande höjningen över den senaste offentliga uppdateringen.

2.  [Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/throttling-and-blocking-email-from-persistently-vulnerable-exchange-servers-to-e/3815328): Det grundläggande inlägget från 2023 med definitionen «persistently vulnerable», nivåerna Reporting, Throttling och Blocking, 90-dagarscykeln, SMTP-svaren 4.7.230 och 5.7.230 samt den versionsvisa utrullningsplanen.

3.  [How to pause throttling and blocking of out-of-date on-premises Exchange Servers – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/how-to-pause-throttling-and-blocking-of-out-of-date-on-premises-exchange-servers/4007169): Cmdletarna `Get-OnPremServerReportInfo`, `Get-TenantExemptionInfo` och `New-TenantExemptionInfo` samt rapporten i Exchange Admin Center och årskvoten på 90 dagar.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Buildnummer för SU:erna från oktober 2025 och efterföljande ESU-uppdateringar till augusti 2026; innehåller även uppgiften att endast ESU-kunder får SU:er från och med december 2025.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU15: October 14, 2025 (KB5066367) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2019-cu15-october-14-2025-kb5066367-19a1091c-e0b3-4078-be6b-312463063d05): KB-artikeln om miniminivån för Exchange 2019 CU15.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: October 14, 2025 (KB5066369) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2016-cu23-october-14-2025-kb5066369-8ca2ab99-dfde-4329-896a-3faa677d2603): KB-artikeln om miniminivån för Exchange 2016 CU23.

7.  [Support for Exchange Server 2016 and Exchange Server 2019 ends today – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/support-for-exchange-server-2016-and-exchange-server-2019-ends-today/4461192): Supportslutet den 14 oktober 2025.

8.  [Announcing Exchange 2016 / 2019 Extended Security Update program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-exchange-2016--2019-extended-security-update-program/4433495): Villkoren för den första ESU-perioden.

9.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Löptiden från maj till oktober 2026 samt beskedet att ingen ytterligare förlängning följer.

10. [Exchange Online transport enforcement system explained – CodeTwo Admin's Blog](https://www.codetwo.com/admins-blog/persistently-vulnerable-exchange-server/): Tabelluppdelning av de åtta enforcement-nivåerna och utrullningsdatumen per Exchange-version; tredjepartskälla.

11. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventering av CU/SU-nivåer och öppna manuella steg.
