---
title: "Exchange-säkerhetsuppdateringar i oktober 2026: sex sårbarheter, sista uppdateringen för Exchange 2016 och 2019"
navTitle: "Exchange SU 10/2026"
description: "Oktober-SU:t åtgärdar sex sårbarheter, däribland en kritisk fjärrkörning av kod via ett manipulerat e-postmeddelande, samt .ics-felet och WordBreaker-låsningen. För Exchange 2016 och 2019 är detta den sista uppdateringen, även i ESU-programmet."
date: "2026-10-14"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "7 min läsning"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
  - "powershell"
slug: "exchange-sakerhetsuppdateringar-i-oktober-2026-sex-sarbarheter-sista-uppdateringen-for-exchange"
translationId: article-173db9431239a95f
translationOf: exchange-security-updates-oktober-2026
url: https://rafaelpfister.ch/sv/blog/exchange-sakerhetsuppdateringar-i-oktober-2026-sex-sarbarheter-sista-uppdateringen-for-exchange
translationSourceHash: ba4b8563ea0b25f5badf03f09e69f9b8329bb75bf4c2422cb5c3fbb97f55d137
translationModel: gpt-5.6-terra
translatedAt: 2026-10-14T08:51:45.344Z
translationReview: automatic
---

# Exchange-säkerhetsuppdateringar i oktober 2026: sex sårbarheter, sista uppdateringen för Exchange 2016 och 2019

Microsoft publicerade den 13 oktober 2026 säkerhetsuppdateringar (SU:er) för Exchange Server SE, Exchange 2019 och Exchange 2016. Enligt KB-artiklarna åtgärdar de sex sårbarheter. Ingen av dem var offentligt känd i förväg och enligt Security Update Guide utnyttjas ingen aktivt. Microsoft klassar dock två som «Exploitation More Likely», däribland en fjärrkörning av kod som klassas som *Critical* och som en oautentiserad angripare kan utlösa med ett manipulerat e-postmeddelande. Den här månaden är särskild av två skäl: SU:t åtgärdar de två öppna Known Issues från septemberuppdateringarna (HTTP 500 för publicerade kalendrar och ContentEngine-låsningen i Exchange SE), och för Exchange 2016 och 2019 är det enligt Microsoft den sista säkerhetsuppdateringen över huvud taget, även för kunder i Period 2 ESU-programmet.

## Vilka Exchange-versioner uppdateringen finns för

SU:erna från den 13 oktober 2026 finns för följande versioner:

- **Exchange Server Subscription Edition (SE) RTM**: KB5128895, build 15.2.2562.56; offentligt tillgänglig.
- **Exchange Server 2019 CU15**: KB5128896, build 15.2.1748.55; endast via **Period 2 ESU-programmet**.
- **Exchange Server 2019 CU14**: KB5128897, build 15.2.1544.50; endast via Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5128898, build 15.1.2507.77; endast via Period 2 ESU.

Du kan jämföra din egen version med översikten över [Exchange-byggnummer](/tools/exchange-builds).

Exchange 2016 och 2019 har nått slutet av supportperioden. Enligt KB-artiklarna får organisationer i Period 2 ESU-programmet säkerhetsuppdateringar till slutet av oktober 2026; alla andra bör migrera till Exchange SE. Exchange Online är enligt tillkännagivandet redan skyddat, men i hybridmiljöer behöver ändå varje Exchange-server SU:t, även rena hanteringsservrar och maskiner med Exchange Management Tools. Vad ESU-slutet konkret innebär beskrivs i avsnittet [Sista uppdateringen för Exchange 2016 och 2019](#letztes-update-für-exchange-2016-und-2019).

## Översikt över sårbarheterna

| CVE | Typ | CVSS |
| --- | --- | --- |
| CVE-2026-70279 | Spoofing (Cross-Site-Scripting) | 9.3 |
| CVE-2026-77910 | Elevation of Privilege | 8.8 |
| CVE-2026-70571 | Remote Code Execution | 8.1 |
| CVE-2026-87692 | Elevation of Privilege | 7.8 |
| CVE-2026-87695 | Elevation of Privilege | 7.8 |
| CVE-2026-87696 | Elevation of Privilege | 7.5 |

Alla fyra KB-artiklar listar samma sex CVE:er. I Security Update Guide saknas dock Exchange 2019 CU15 för CVE-2026-87692; där anges endast Exchange SE, 2019 CU14 och 2016 CU23 som berörda (status den 14 oktober 2026). Frankys Web tolkar detta som att 2019 CU15 inte påverkas av denna sårbarhet. För installationen spelar skillnaden ingen roll, eftersom SU:t på CU15 ändå åtgärdar alla övriga sårbarheter.

**[CVE-2026-70571](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-70571)** är månadens enda sårbarhet som Microsoft klassar som *Critical* (Remote Code Execution, CVSS 8.1). Orsaken är ett stackbaserat buffertspill. Enligt FAQ:n i Security Update Guide kan en oautentiserad angripare skicka ett manipulerat e-postmeddelande till en berörd Exchange-server och därmed utlösa en intern konverteringsprocess; ingen användarinteraktion krävs. Angreppskomplexiteten är enligt CVSS-vektorn hög (AC:H); Microsoft skriver att en lyckad exploatering inte är garanterad och kräver djup förståelse för målmiljön. Trots detta bedömer Microsoft exploatering som «Exploitation More Likely». Angriparen behöver inget konto i organisationen, det räcker att servern tar emot e-post.

**[CVE-2026-70279](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-70279)** har med CVSS 9.3 det högsta värdet och klassas också som «Exploitation More Likely». Det handlar om Cross-Site-Scripting: En oautentiserad angripare skickar en manipulerad länk; om en administratör öppnar den kan angriparen utföra åtgärder med dennes behörigheter och därigenom komma åt eller ändra organisationens data. Enligt FAQ:n krävs användarinteraktion, vanligtvis genom social engineering. Allvarlighetsgraden är *Important*.

**[CVE-2026-77910](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-77910)** (Elevation of Privilege, CVSS 8.8) kräver endast ett konto med låga behörigheter. Enligt FAQ:n kan en angripare manipulera en delad cache i Exchange Control Panel och därigenom installera ett skadligt e-posttillägg i en annan användares postlåda, utan att denne behöver göra något. Microsoft beskriver samma angreppsväg för [CVE-2026-87696](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-87696) (CVSS 7.5), där som följd av otillräcklig kontroll av kryptografiska signaturer och med hög angreppskomplexitet.

De två återstående sårbarheterna, [CVE-2026-87692](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-87692) (deserialisering av opålitliga data) och [CVE-2026-87695](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-87695), är lokala behörighetshöjningar med CVSS 7.8: En inloggad angripare med låga behörigheter kan enligt Microsoft få SYSTEM-behörigheter. Båda klassas som «Exploitation Less Likely», liksom CVE-2026-77910 och CVE-2026-87696.

Om **CVE-2026-96940** från september-v2: Oktoberannonseringen nämner inte v2. Enligt FAQ:n i tillkännagivandet är SU:er kumulativa; den som använder en version med stöd installerar bara den senaste SU:n. Frankys Web bekräftar att oktober-SU:t innehåller korrigeringen från v2. Servrar som fortfarande kör det första september-SU:t från den 8 september kan alltså installera oktober-SU:t direkt; information om v2 finns i [artikeln om september-v2](/blog/exchange-security-updates-september-2026-v2).

## Sista uppdateringen för Exchange 2016 och 2019

Tillkännagivandet är tydligt på denna punkt: Oktober 2026 är den sista månaden då kunder i Period 2 ESU-programmet får nya säkerhetsuppdateringar för Exchange 2016 och 2019. Från och med nästa version publiceras uppdateringar endast för Exchange SE. Framtida tillkännagivanden och CVE-meddelanden kommer inte längre att nämna 2016 och 2019, och Microsoft skriver uttryckligen: «We will stop testing Exchange 2016 and 2019 for vulnerabilities.» Att en CVE inte nämner dessa versioner innebär alltså inte att de inte påverkas; Microsoft betonar i tillkännagivandet att de gamla versionerna inte blir säkra av detta. Frankys Web påpekar dessutom att Exchange SE och 2019 CU15 tekniskt sett är mycket lika, så framtida SE-sårbarheter med hög sannolikhet också berör 2019.

Enligt tillkännagivandet går Period 2 ESU fortfarande att köpa under begränsad tid. Det tar dock bara organisationen till nivån för uppdateringen från oktober 2026, och inga ytterligare uppdateringar följer. Den som redan har köpt en ESU och har frågor om åtkomst kan enligt KB-artiklarna kontakta `ExchangeandSfBServerESUInquiry@service.microsoft.com`.

För drift innebär detta:

1. **Exchange 2016 och 2019 med ESU**: Installera oktober-SU:t. Därefter fryser versionen, och varje sårbarhet som senare blir känd förblir öppen.

2. **Exchange 2016 och 2019 utan ESU**: Dessa servrar har inte fått några säkerhetsuppdateringar sedan maj 2026. Microsoft rekommenderar migrering till Exchange SE så snart som möjligt.

3. **Hybridmiljöer**: Utöver avsaknaden av patchar stryper och blockerar Exchange Online e-postflödet från föråldrade lokala servrar; reglerna finns i [artikeln om transport enforcement](/blog/exchange-online-transport-enforcement-hybrid-server).

## Åtgärdade problem och vad du behöver göra efteråt

**Publicerade kalendrar (.ics) ger HTTP 500 i kalenderappar** (alla versioner). Problemet, som har funnits sedan augusti-SU:t, är åtgärdat med oktober-SU:t; [KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672) anger nu installation av de fyra oktoberuppdateringarna eller nyare som lösning. Supportartikeln beskriver inte längre den tidigare URL-rewrite-lösningen på webbplatsen «Exchange Back End», och Microsoft ger ingen anvisning för hur regeln tas bort (status den 14 oktober 2026). Om du har konfigurerat regeln ska du dokumentera vilka servrar den är aktiv på och testa en kalenderprenumeration innan du tar bort den.

**ContentEngine-låsning på grund av saknade koreanska WordBreaker-regelfiler** (endast Exchange SE). Enligt [KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098) är problemet åtgärdat med KB5128895 eller en senare uppdatering, och efter installationen av oktoberuppdateringen behövs «no additional action». Detta gäller även servrar där du har tillämpat den manuella lösningen från [septemberartikeln](/blog/exchange-security-updates-september-2026#workaround-für-den-wordbreaker-deadlock): Microsoft kräver inte att de kopierade regelfilerna återställs. Servrar utan workaround behöver inte längre lägga till filerna manuellt.

**Ledig/upptagen för delegerade postlådor i hybridmiljöer med enbart Graph API-konfiguration** (endast Exchange SE). Detta problem åtgärdas inte av oktober-SU:t: [KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) anger nu «The issue has been fixed in the service». Motsägelsen från [v2-artikeln](/blog/exchange-security-updates-september-2026-v2) är därmed löst. Om du har angett workarounden ska du återställa den enligt KB5127092:

```powershell
Get-SettingOverride -Identity EnableRouteThroughMSGraphFeature
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=True"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `Get-SettingOverride -Identity EnableRouteThroughMSGraphFeature` | Visar override-värdets aktuella värde; endast vid `Enabled=False` behövs nästa kommando. |
| `-Parameters "Enabled=True"` | Aktiverar vidarebefordran av tillgänglighetsfrågor via Microsoft Graph igen. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Riktar diagnostikanropet till topologitjänsten. |
| `-Component VariantConfiguration -Argument Refresh` | Läser in Variant Configuration på nytt så att ändringen får effekt direkt. |

</details>

Kontrollera därefter ledig/upptagen-frågan för en delegerad Exchange Online-postlåda från Outlook eller OWA.

## Kända problem

Varken tillkännagivandet eller de fyra KB-artiklarna nämner Known Issues för oktober-SU:t (status den 14 oktober 2026). De tre öppna problemen från septemberuppdateringarna är därmed lösta: två genom oktober-SU:t, ett genom en korrigering i Exchange Online (se ovan). Wrapper-problemet i delade postlådor åtgärdades redan med september-SU:t.

## Installation och efterarbete

Microsoft anger det välkända förloppet i tillkännagivandet: inventera med [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), fastställ sökvägen med [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) vid en föråldrad CU-version, installera SU:t, starta om servern och kontrollera att alla Exchange-tjänster körs. Vid fel hänvisar Microsoft till [SetupAssist-skriptet](https://aka.ms/ExSetupAssist). Efter installationen visar Health Checker enligt KB5128895 om SU:t har installerats korrekt och om ytterligare steg krävs. Security Update Guide anger ingen obligatorisk omstart för oktoberbyggena, men tillkännagivandet rekommenderar den ändå efter installationen.

Installationsfilen för Exchange SE heter enligt KB5128895 `ExchangeSubscriptionEdition-KB5128895-x64-en.exe` (Download Center: «SU11»), för Exchange 2019 CU15 `Exchange2019-KB5128896-x64-en.exe`, för CU14 `Exchange2019-KB5128897-x64-en.exe` och för Exchange 2016 `Exchange2016-KB5128898-x64-en.exe`. Paketen är CU-specifika.

För hybridmiljöer gäller enligt FAQ:n: Om autentiseringscertifikatet byts efter installationen av en SU måste Hybrid Configuration Wizard köras igen. På Windows Server 2025 visas installerade Exchange-SU:er inte i Kontrollpanelen; Microsoft avråder från att avinstallera SU:er.

Från tidigare månader återstår följande efterarbete, om det ännu inte har gjorts: ta bort Wrapper-SettingOverride `DisableBlockSharedAndUserMailboxHeaders` (se [septemberartikeln](/blog/exchange-security-updates-september-2026)) och kontrollera om CVE-2026-42897-mitigeringen (M2.1.0) fortfarande är aktiv (se [artikeln om juli-SU:t](/blog/exchange-security-updates-juli-2026)).

## Rekommenderat tillvägagångssätt

Installera oktober-SU:t snarast på alla Exchange-servrar och maskiner med Management Tools: Den kritiska fjärrkörningen av kod CVE-2026-70571 kan utlösas utan konto via ett e-postmeddelande, och Microsoft bedömer att exploatering av den är lika sannolik som för Cross-Site-Scripting-sårbarheten CVE-2026-70279. Servrar med det första september-SU:t får därmed också korrigeringen för CVE-2026-96940. Återställ därefter ledig/upptagen-overriden om den är angiven och kontrollera publicerade kalendrar. För Exchange 2016 och 2019 är detta den sista uppdateringen; planera migreringen till Exchange SE nu, eftersom dessa servrar inte längre får säkerhetsuppdateringar från och med november.

## Källor

1.  [Released: October 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-october-2026-exchange-server-security-updates/ba-p/4563014): Tillkännagivande från den 13 oktober 2026 med versioner som stöds, åtgärdade problem, slutet på uppdateringar för Exchange 2016/2019 samt FAQ (granskat via RSS-flödet för Exchange Team Blog; inga kommentarer från Exchange-teamet, status den 14 oktober 2026).

2.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM October 13, 2026 (KB5128895) – Microsoft Support](https://support.microsoft.com/help/5128895): CVE-lista, åtgärdade problem, installationsfil och anmärkning om Health Checker.

3.  [Description of the security update for Microsoft Exchange Server 2019 CU15 October 13, 2026 (KB5128896) – Microsoft Support](https://support.microsoft.com/help/5128896): KB-artikel för Exchange 2019 CU15 med ESU-information.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU14 October 13, 2026 (KB5128897) – Microsoft Support](https://support.microsoft.com/help/5128897): KB-artikel för Exchange 2019 CU14 med ESU-kontaktadress.

5.  [Description of the security update for Microsoft Exchange Server 2016 CU23 October 13, 2026 (KB5128898) – Microsoft Support](https://support.microsoft.com/help/5128898): KB-artikel för Exchange 2016 CU23.

6.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Typ, CVSS, allvarlighetsgrad, exploateringsbedömning, FAQ och berörda byggen för de sex CVE:erna från oktober samt CVE-2026-96940.

7.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Byggnummer för oktober-SU:erna och föregående uppdateringar.

8.  [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Korrigering med oktoberuppdateringarna.

9.  [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Korrigering med KB5128895, inga ytterligare steg krävs.

10. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Korrigering i tjänsten och kommandon för att återställa SettingOverride.

11. [Neue Sicherheitsupdates für Exchange Server (Oktober 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-oktober-2026/): Tyskspråkig genomgång av de sex CVE:erna, information om den inkluderade v2-korrigeringen och bedömning av ESU-slutet.

12. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventering av CU/SU-versioner före och efter installationen.
