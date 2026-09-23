---
title: "Exchange Online stryper och blockerar föråldrade Exchange 2016- och 2019-servrar från september 2026: Så fungerar Transport Enforcement"
navTitle: "EXO-Enforcement 09/2026"
description: "Från och med den andra veckan i september 2026 kräver Exchange Online minst oktober 2025-SU från hybridservrar, annars stryps och blockeras e-postflödet senare. Bakgrund till Transport Enforcement sedan 2023, eskaleringsnivåerna med SMTP-koder, rapporten i Admin Center, 90-dagarspausen via PowerShell och varför nästa höjning endast släpper igenom ESU-kunder och Exchange SE."
date: "2026-09-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "9 min lästid"
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
translationSourceHash: b2c2e61a4e97d49b1046134bf215a91e0d46ed817201023d8364945f3549d539
translationModel: gpt-5.6-terra
translatedAt: 2026-09-08T08:15:09.111Z
translationReview: automatic
url: https://rafaelpfister.ch/sv/blog/exchange-online-stryper-och-blockerar-foraldrade-exchange-2016-och-2019-fran-september-2026-sa
---

# Exchange Online stryper och blockerar föråldrade Exchange 2016- och 2019-servrar från september 2026: Så fungerar Transport Enforcement

Exchange-teamet meddelade den 2 september 2026 att minimiversionen för Exchange 2016 och Exchange 2019 i hybrid-e-postflödet höjs. Från och med den andra veckan i september 2026 kräver Exchange Online att servrar som levererar via en inkommande anslutning av typen `OnPremises` minst har nivån från den senaste offentliga säkerhetsuppdateringen från oktober 2025. Allt under detta stryps och blockeras senare. Kort sagt: Den som inte har patchat sina hybridservrar sedan oktober 2025 förlorar stegvis e-postleveransen till Exchange Online under de kommande veckorna. Och nästa höjning, som Microsoft aviserar för de kommande månaderna, ligger över alla offentligt tillgängliga uppdateringar: då uppfyller endast kunder i det avgiftsbelagda ESU-programmet eller miljöer med Exchange Server Subscription Edition (SE) kravet.

Själva tillkännagivandet är kort. Vad det innebär i praktiken framgår av enforcement-systemet som Microsoft stegvis har byggt upp sedan 2023: vilka SMTP-svar servern får, hur du kontrollerar statusen i Admin Center och via PowerShell samt vilka alternativ som återstår under övergången fram till ESU-programmets slut i oktober 2026.

## Vad som gäller från och med den andra veckan i september 2026

Den nya nedre gränsen motsvarar säkerhetsuppdateringarna från den 14 oktober 2025. Det var den sista Patch Tuesday då Microsoft tillhandahöll uppdateringar offentligt för Exchange 2016 och 2019; alla SU:er sedan december 2025 finns endast via ESU-programmet.

| Version | Miniminivå | KB | Build |
|---|---|---|---|
| Exchange 2019 CU15 | Oktober 2025-SU (CU15 SU5) | KB5066367 | 15.2.1748.39 |
| Exchange 2016 CU23 | Oktober 2025-SU (CU23 SU19) | KB5066369 | 15.1.2507.61 |

För Exchange 2019 CU14 finns också ett oktober 2025-SU (KB5066368, build 15.2.1544.36). I Microsofts inlägg anges dock uttryckligen CU15 SU5 som minimiversion; CU14 är ändå inte längre en rekommenderad nivå sedan CU15 släpptes i februari 2025. Planera därför även för uppgraderingen från CU14 till CU15.

Tre avgränsningar är viktiga:

- **Endast hybrid-e-postflödet påverkas.** Exchange Online kontrollerar versionen på de levererande servrarna för meddelanden som anländer via en inkommande anslutning av typen `OnPremises`. Det är den klassiska hybridkonfigurationen som Hybrid Configuration Wizard skapar. E-post som anländer via en tredjepartsgateway eller en anslutning av typen `Partner` passerar inte denna enforcement.
- **Versionen läses från meddelandehuvudena.** En Exchange-server skriver sin build i raden `Received` för varje meddelande som den vidarebefordrar (`… with Microsoft SMTP Server … id 15.1.2507.61`). Exchange Online utvärderar denna uppgift. Därmed är det statusen för den server som faktiskt överlämnar meddelandet till Exchange Online som räknas, alltså i många miljöer Edge Transport Server eller Mailbox-servern med Send Connector till `*.mail.protection.outlook.com`.
- **Exchange SE påverkas inte.** Enforcement gäller Exchange 2016 och 2019; Exchange Server SE ligger över varje nedre gräns så länge den patchas regelbundet.

## Bakgrund: Transport Enforcement sedan 2023

Tillkännagivandet i september är inte en ny åtgärd, utan nästa steg i ett system som Microsoft presenterade i mars 2023 under titeln «Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online». Microsoft definierar «persistently vulnerable» som varje Exchange-server som antingen har nått slutet av supportperioden eller förblir opatchad för kända sårbarheter. Målet är att skydda Exchange Online-mottagare mot meddelanden från servrar som kan komprometteras och samtidigt sätta press på operatörerna att patcha eller stänga av servrarna.

Systemet har aktiverats per version:

| Tidpunkt | Berörd version |
|---|---|
| Augusti 2023 | Exchange 2007 |
| September 2023 | Exchange 2010 |
| December 2023 | Exchange 2013 |
| Mars 2024 | Exchange 2016 och 2019 (kraftigt föråldrade SU-nivåer) |
| September 2026 | Exchange 2016 och 2019: nedre gräns = oktober 2025-SU |
| «om några månader» | Exchange 2016 och 2019: nedre gräns över den senaste offentliga uppdateringen |

För Exchange 2016 och 2019 låg den nedre gränsen hittills vid nivåer som var «significantly behind on security updates». Nytt är att Microsoft sätter gränsen vid den senaste offentliga uppdateringen och därmed för första gången träffar servrar som fortfarande var fullt patchade för mindre än ett år sedan.

## Eskaleringsnivåerna

Enforcement arbetar med tre funktioner som Microsoft kallar «reporting», «throttling» och «blocking». Så snart en server hamnar under den nedre gränsen startar en 90-dagarscykel. Nivåerna från grundläggande inlägget från 2023:

| Period | Åtgärd | SMTP-svar |
|---|---|---|
| Dag 0 till 30 | Endast rapport i Exchange Admin Center | inga |
| Dag 30 till 40 | Strypning 5 minuter per timme | `450 4.7.230` |
| Dag 40 till 50 | Strypning 10 minuter per timme | `450 4.7.230` |
| Dag 50 till 60 | Strypning 20 minuter per timme | `450 4.7.230` |
| Dag 60 till 70 | Strypning 30 minuter per timme, dessutom blockering 5 minuter per timme | `450 4.7.230` och `550 5.7.230` |
| Dag 70 till 80 | Blockering 10 minuter per timme | `550 5.7.230` |
| Dag 80 till 90 | Blockering 20 minuter per timme | `550 5.7.230` |
| Från dag 90 | Fullständig blockering | `550 5.7.230` |

De två svaren lyder ordagrant:

```text
450 4.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online throttled for n mins/hr.

550 5.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online blocked for n mins/hr.
```

Skillnaden är avgörande för driften. Vid `450` avvisar Exchange Online anslutningen tillfälligt; den lokala servern behåller meddelandet i sin kö och försöker igen. Användarna märker till en början endast förseningar, medan kön till Send Connector mot Exchange Online växer i Queue Viewer eller i `Get-Queue` med statusen `Retry` och meddelandet 4.7.230 som `LastError`. Vid `550` är avvisningen slutgiltig: avsändaren får ett NDR med koden 5.7.230 och meddelandet går förlorat om det inte skickas igen. Eftersom blockeringen inledningsvis endast är aktiv några minuter per timme framstår felbilden först som sporadisk: en del meddelanden kommer fram, andra misslyckas med NDR. Den som ser ett sådant mönster i Message Tracking bör först kontrollera versionsnivån innan nätverks- eller certifikatproblem undersöks.

Om Microsoft för den nya nedre gränsen startar hela 90-dagarscykeln från den andra veckan i september eller redan börjar på en senare nivå anges inte i tillkännagivandet. Grundläggande inlägget fastslår att systemet fortsätter på den tidigare uppnådda nivån efter en paus. Räkna därför inte med 30 dagars respit.

## Rapport i Exchange Admin Center och via PowerShell

Exchange Online listar de identifierade lokala servrarna med version i en egen rapport: i Exchange Admin Center under *Reports*, *Mail flow*, rapporten om föråldrade anslutande lokala Exchange-servrar («out-of-date connecting on-premises Exchange servers»). Rapporten visar per server identifierad build, om den ligger under den nedre gränsen och vilken enforcement-nivå som gäller.

Samma information tillhandahålls av Exchange Online PowerShell:

```powershell
Connect-ExchangeOnline
Get-OnPremServerReportInfo
```

<details class="options-details">
<summary>Förklarar alternativen</summary>

| Kommando | Funktion |
|---|---|
| `Connect-ExchangeOnline` | Öppnar sessionen till Exchange Online (modulen `ExchangeOnlineManagement`). |
| `Get-OnPremServerReportInfo` | Listar de lokala servrar som Exchange Online har identifierat med build, enforcement-status och nivå. |

</details>

Rapporten känner bara till servrar som faktiskt överlämnar e-post till Exchange Online. En hanteringsserver utan e-postflöde eller en dator med enbart Management Tools visas inte. För enforcement spelar detta ingen roll, men för säkerheten gör det det: även dessa system behöver SU:erna.

## Pausa enforcement: 90 dagar per år

För miljöer som inte kan nå den nedre gränsen på kort sikt erbjuder Microsoft en paus. Den kan aktiveras sammanlagt 90 dagar per år, i en följd eller i flera avsnitt:

```powershell
Get-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer

New-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer `
  -NumberOfDays 30
```

<details class="options-details">
<summary>Förklarar alternativen</summary>

| Alternativ | Funktion |
|---|---|
| `Get-TenantExemptionInfo` | Visar om och hur länge en paus är aktiv för klientorganisationen. |
| `New-TenantExemptionInfo` | Skapar en ny paus. |
| `-BlockingScenario UnpatchedOnPremServer` | Väljer scenariot «föråldrad lokal server»; andra scenarier finns för närvarande inte för denna cmdlet. |
| `-NumberOfDays 30` | Pausens längd i dagar. Kvoten är 90 dagar per år och angivelsen dras av från den. |

</details>

Två egenskaper hos pausen är viktiga i praktiken. För det första fortsätter enforcement efter utgången på den nivå där den pausades; pausen återställer inte 90-dagarscykeln. För det andra finns ingen cmdlet för att avsluta en pågående paus i förtid: den som skapar en paus på 90 dagar och patchar klart efter två veckor har förbrukat årskvoten. Skapa därför pausen så kort som möjligt och förläng vid behov.

Pausen är dessutom endast en lösning för den aktuella nedre gränsen. Om Microsoft om några månader höjer gränsen över den senaste offentliga uppdateringen hjälper inte en förbrukad kvot längre.

## Varför nästa höjning är den egentliga tidsfristen

Exchange 2016 och 2019 har varit out of support sedan den 14 oktober 2025. Microsoft har därefter infört två avgiftsbelagda ESU-perioder: period 1 till april 2026, period 2 från maj till oktober 2026. I samband med tillkännagivandet av period 2 den 15 april 2026 klargjorde Exchange-teamet att det inte blir någon ytterligare förlängning. SU:erna från december 2025 till augusti 2026 (senast build 15.2.1748.49 för 2019 CU15 och 15.1.2507.72 för 2016 CU23) är endast tillgängliga för ESU-kunder och erbjuds inte för offentlig nedladdning.

Detta ger följande läge:

- **I dag** uppfyller en server med oktober 2025-SU den nedre gränsen, med eller utan ESU.
- **Vid nästa höjning** ligger den nedre gränsen enligt Microsoft över nivån från oktober 2025. Utan ESU-avtal finns ingen laglig väg att nå den nivån. Dessa servrars hybrid-e-postflöde stryps och blockeras då, oavsett hur väl resten av miljön drivs.
- **Den 31 oktober 2026** upphör även period 2. Därefter finns inga fler SU:er för Exchange 2016 och 2019, för någon. Senast nästa höjning därefter träffar alltså även ESU-kunder.

ESU-programmet köper därmed i bästa fall några månader. Den enda hållbara nivån som släpps igenom av enforcement är Exchange Server SE. Microsoft har dessutom meddelat att Exchange SE CU2 (planerat för andra halvåret 2026) avslutar samexistens med Exchange 2016 och 2019: installationen avbryts om äldre servrar hittas i organisationen. Migreringen är alltså inte bara nödvändig på grund av e-postflödet, utan också för att över huvud taget kunna fortsätta installera uppdateringar för SE.

För miljöer som bara behåller Exchange On-Premises för att hantera attribut i en hybridkonstellation är alternativet att ta bort den sista servern: sedan Exchange 2019 CU12 kan mottagarattribut hanteras med Management Tools utan en körande Exchange-server. Då finns inget lokalt e-postflöde kvar och enforcement saknar relevans.

## Fastställa versionsnivån

`Get-ExchangeServer` visar i `AdminDisplayVersion` endast CU, inte SU. Tillförlitlig är filversionen för `ExSetup.exe` eller [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), som dessutom rapporterar saknade manuella steg. För en snabb översikt över alla servrar:

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
<summary>Förklarar alternativen</summary>

| Element | Funktion |
|---|---|
| `Get-ExchangeServer` | Listar organisationens alla Exchange-servrar. |
| `\\<Server>\C$\...\ExSetup.exe` | Administrativ delningssökväg till installationsfilen; anpassa vid en avvikande installationssökväg. |
| `VersionInfo.ProductVersion` | Filversion som motsvarar den installerade SU-builden (t.ex. `15.1.2507.61`). |

</details>

Om versionen ligger under `15.2.1748.39` (2019 CU15) respektive `15.1.2507.61` (2016 CU23) hamnar servern under den nedre gränsen från och med den andra veckan i september.

## Rekommenderat tillvägagångssätt

1. **Inventera nivån** enligt beskrivningen ovan, inklusive Edge Transport Server och hanteringsservrar.

2. **Kontrollera rapporten i Exchange Online.** `Get-OnPremServerReportInfo` visar vilka servrar Exchange Online faktiskt ser och om en enforcement-nivå redan är aktiv. Jämför listan med inventeringen: servrar som saknas där levererar inte via anslutningen `OnPremises`.

3. **Installera minst oktober 2025-SU.** KB5066367 (2019 CU15) och KB5066369 (2016 CU23) finns fortfarande offentligt tillgängliga i Microsoft Download Center. SU:er är kumulativa; en server på nivån från augusti 2025 kan uppdateras direkt till oktober 2025. Installera först CU15 vid CU14. Kontrollera efter installationen omstarten och tjänsternas status, och kör sedan Health Checker igen.

4. **Använd pausen endast som en tillfällig lösning.** Om uppdateringen inte lyckas under första halvan av september, skapa `New-TenantExemptionInfo` med kort löptid och betrakta inte pausen som planeringsreserv för nästa höjning.

5. **Schemalägg migreringen till Exchange SE.** Utan ESU-avtal är nästa höjning den hårda tidsfristen, med ESU är det den 31 oktober 2026. Exchange 2019 CU15 kan uppgraderas till SE med en in-place-uppgradering; Exchange 2016 kräver omvägen via en ny SE-installation och flytt av postlådor respektive roller. Den som endast använder Exchange för attributadministration tar bort den sista servern och fortsätter arbeta med Management Tools.

## Källor

1.  [Exchange 2016/2019: Throttling and Blocking up to the Final Public Update Baseline – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/exchange-20162019-throttling-and-blocking-up-to-the-final-public-update-baseline/4552717): Tillkännagivandet från den 2 september 2026 med den nya nedre gränsen (oktober 2025-SU), startdatumet under den andra veckan i september och upplysningen om den kommande höjningen över den senaste offentliga uppdateringen.

2.  [Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/throttling-and-blocking-email-from-persistently-vulnerable-exchange-servers-to-e/3815328): Det grundläggande inlägget från 2023 med definitionen «persistently vulnerable», nivåerna Reporting, Throttling och Blocking, 90-dagarscykeln, SMTP-svaren 4.7.230 och 5.7.230 samt utrullningsplanen per version.

3.  [How to pause throttling and blocking of out-of-date on-premises Exchange Servers – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/how-to-pause-throttling-and-blocking-of-out-of-date-on-premises-exchange-servers/4007169): Cmdletarna `Get-OnPremServerReportInfo`, `Get-TenantExemptionInfo` och `New-TenantExemptionInfo` samt rapporten i Exchange Admin Center och årskvoten på 90 dagar.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Buildnummer för SU:erna från oktober 2025 och efterföljande ESU-uppdateringar till augusti 2026; innehåller även upplysningen att endast ESU-kunder får SU:er från december 2025.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU15: October 14, 2025 (KB5066367) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2019-cu15-october-14-2025-kb5066367-19a1091c-e0b3-4078-be6b-312463063d05): KB-artikeln om miniminivån för Exchange 2019 CU15.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: October 14, 2025 (KB5066369) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2016-cu23-october-14-2025-kb5066369-8ca2ab99-dfde-4329-896a-3faa677d2603): KB-artikeln om miniminivån för Exchange 2016 CU23.

7.  [Support for Exchange Server 2016 and Exchange Server 2019 ends today – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/support-for-exchange-server-2016-and-exchange-server-2019-ends-today/4461192): Slutet på supporten den 14 oktober 2025.

8.  [Announcing Exchange 2016 / 2019 Extended Security Update program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-exchange-2016--2019-extended-security-update-program/4433495): Villkoren för den första ESU-perioden.

9.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Löptiden från maj till oktober 2026 och beskedet att ingen ytterligare förlängning följer.

10. [Exchange Online transport enforcement system explained – CodeTwo Admin's Blog](https://www.codetwo.com/admins-blog/persistently-vulnerable-exchange-server/): Tabellöversikt över de åtta enforcement-nivåerna och utrullningsdatumen per Exchange-version; tredjepartskälla.

11. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventering av CU/SU-nivåer och öppna manuella steg.
