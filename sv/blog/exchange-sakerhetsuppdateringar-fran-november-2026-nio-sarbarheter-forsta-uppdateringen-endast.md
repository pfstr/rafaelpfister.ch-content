---
title: "Exchange-säkerhetsuppdateringar från november 2026: nio sårbarheter, första uppdateringen endast för Exchange SE"
navTitle: "Exchange SU 11/2026"
description: "November-SU åtgärdar nio sårbarheter i Exchange Server SE, däribland en kritisk fjärrkörning av kod där ett konto med låga behörigheter räcker. Exchange 2016 och 2019 får inte längre några uppdateringar. Exchange-avbrotten efter Windows-uppdateringarna i oktober åtgärdas av Windows-uppdateringen från november."
date: "2026-11-11"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "6 min lästid"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
slug: "exchange-sakerhetsuppdateringar-fran-november-2026-nio-sarbarheter-forsta-uppdateringen-endast"
translationId: article-cfb739155e545cc3
translationOf: exchange-security-updates-november-2026
url: https://rafaelpfister.ch/sv/blog/exchange-sakerhetsuppdateringar-fran-november-2026-nio-sarbarheter-forsta-uppdateringen-endast
translationSourceHash: 1ce238b3219fc9c3670ec3137e807538c62e566ad22bd60fbee45dcf6dce9193
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:25:17.207Z
translationReview: automatic
---

# Exchange-säkerhetsuppdateringar från november 2026: nio sårbarheter, första uppdateringen endast för Exchange SE

Microsoft publicerade den 10 november 2026 en säkerhetsuppdatering (SU) för Exchange Server SE. Enligt KB-artikeln åtgärdar den nio sårbarheter. Ingen av dem var offentligt känd i förväg, och enligt Security Update Guide utnyttjas ingen aktivt. Microsoft klassar en fjärrkörning av kod som *Critical* och som ”Exploitation More Likely”; för den räcker ett konto med låga behörigheter. Den här månaden är särskild av två skäl: Det är den första SU:n som endast släpps för Exchange SE, och Exchange-avbrotten efter Windows Server-uppdateringarna i oktober har åtgärdats med Windows-uppdateringarna från november.

## Vilka Exchange-versioner uppdateringen är tillgänglig för

SU:n från den 10 november 2026 är endast tillgänglig för en version:

- **Exchange Server Subscription Edition (SE) RTM**: KB5133453, build 15.2.2562.57; offentligt tillgänglig.

Du kan jämföra din egen version med översikten över [Exchange-buildnummer](/tools/exchange-builds).

För Exchange 2016 och 2019 finns ingen uppdatering. [Oktober-SU:n](/blog/exchange-security-updates-oktober-2026) var den sista, även för kunder i Period 2 ESU-programmet, och enligt tillkännagivandet har programmet avslutats. Enligt Microsoft är Exchange Online redan skyddat; i hybridmiljöer behöver dock varje Exchange-server SU:n, även rena hanteringsservrar och datorer med Exchange Management Tools.

## Översikt över sårbarheterna

| CVE | Typ | CVSS |
| --- | --- | --- |
| CVE-2026-118207 | Remote Code Execution | 8.8 |
| CVE-2026-118149 | Remote Code Execution | 8.8 |
| CVE-2026-118150 | Elevation of Privilege | 8.8 |
| CVE-2026-118154 | Elevation of Privilege | 8.8 |
| CVE-2026-118153 | Denial of Service | 7.5 |
| CVE-2026-118151 | Elevation of Privilege | 7.2 |
| CVE-2026-118157 | Elevation of Privilege | 7.2 |
| CVE-2026-118208 | Spoofing | 7.1 |
| CVE-2026-118152 | Spoofing | 6.5 |

Alla nio sårbarheter kan nås via nätverket (AV:N) och har enligt CVSS-vektorn låg attackkomplexitet (AC:L). Security Update Guide listar endast Exchange Server SE RTM som berörd produkt.

**[CVE-2026-118207](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118207) är månadens enda sårbarhet som Microsoft klassar som *Critical* (Remote Code Execution, CVSS 8.8), och den enda med bedömningen ”Exploitation More Likely”. Enligt FAQ:n i Security Update Guide kan en autentiserad angripare med låga behörigheter skicka en preparerad begäran till en berörd server och därmed köra kod på servern; ingen användarinteraktion krävs. Microsoft anger otillräcklig indatavalidering (CWE-706) som orsak. För attacken räcker alltså ett vanligt postlådekonto, till exempel ett med ett lösenord som har stulits genom nätfiske.

**[CVE-2026-118149](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118149) är den andra Remote Code Execution-sårbarheten (CVSS 8.8, *Important*). Även här skickar en autentiserad angripare enligt FAQ:n en preparerad begäran till en berörd tjänst och kan därefter köra kod på målsystemet; orsaken är en code injection (CWE-94). Microsoft bedömer utnyttjandet som ”Exploitation Unlikely”.

**[CVE-2026-118153](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118153) (Denial of Service, CVSS 7.5) är den enda sårbarheten som varken kräver ett konto eller användarinteraktion. Enligt beskrivningen kan en oautentiserad angripare slå ut tjänsten via nätverket; orsaken är ineffektiv algoritmisk komplexitet (CWE-407). Enligt vektorn påverkas endast tillgängligheten, inte konfidentialiteten eller integriteten.

De fyra behörighetshöjningarna skiljer sig åt i förutsättningarna: För [CVE-2026-118150](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118150) och [CVE-2026-118154](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118154) (båda CVSS 8.8) räcker ett konto med låga behörigheter, medan angriparen för [CVE-2026-118151](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118151) och [CVE-2026-118157](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118157) (båda CVSS 7.2) redan behöver höga behörigheter. Vid CVE-2026-118151 kan angriparen enligt FAQ:n få SYSTEM-behörigheter.

Av de två spoofing-sårbarheterna kräver [CVE-2026-118208](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118208) (CVSS 7.1) ett konto med låga behörigheter. [CVE-2026-118152](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118152) (CVSS 6.5) är en Cross-Site Request Forgery: Angriparen behöver inget konto, men en användare måste medverka (UI:R). Alla sårbarheter utom CVE-2026-118207 och CVE-2026-118149 är klassade som ”Exploitation Less Likely”.

## Exchange 2016 och 2019: inga uppdateringar, inga fler tester

Tillkännagivandet nämner numera Exchange 2016 och 2019 endast i en notis: Dessa versioner testas inte längre för sårbarheter och får inga fler uppdateringar. Microsoft skriver att de bör betraktas som permanent sårbara (”permanently vulnerable”) och att organisationen omedelbart bör uppgraderas. Att de nio CVE:erna från november endast listar Exchange SE säger därför inget om huruvida de äldre versionerna är berörda. Frankys Web påpekar att Exchange SE och 2019 CU15 tekniskt ligger nära varandra och att åtminstone en del av sårbarheterna sannolikt även finns i Exchange 2019.

För drift innebär detta:

1. **Exchange 2019 CU15 och äldre, Exchange 2016**: Versionen förblir fryst på oktober-SU:n (med ESU) eller äldre. De nio sårbarheterna från november förblir öppna på dessa servrar om de finns där, och Microsoft testar inte längre detta.

2. **Migrering**: Microsoft rekommenderar omedelbar övergång till Exchange SE.

3. **Hybridmiljöer**: Exchange Online begränsar och blockerar e-postflödet från föråldrade lokala servrar; reglerna finns i [artikeln om Transport Enforcement](/blog/exchange-online-transport-enforcement-hybrid-server).

## Åtgärdade problem

**Exchange-avbrott efter Windows Server-uppdateringarna i oktober** (alla versioner). Problemet med `MapiExceptionMdbOffline` (ec=1142) åtgärdas inte av Exchange-SU:n: Enligt tillkännagivandet ska Windows Server-uppdateringarna från november 2026 installeras. Release Notes för dessa uppdateringar listar korrigeringen under ”[Exchange Server]” och anger problemet som löst med uppdateringar från och med den 10 november 2026: KB5131080 för Windows Server 2025, KB5131090 för Windows Server 2022, KB5131085 för Windows Server 2019 och KB5131291 för Windows Server 2016. Enligt tillkännagivandet spelar ordningen i vilken du installerar Exchange-SU:n och Windows-uppdateringen ingen roll. Om du har satt registervärdet `EpmapSavedContextCap` från [artikeln om Exchange-avbrotten](/blog/exchange-windows-updates-oktober-2026), kan du låta det vara kvar; efter installationen av novemberuppdateringarna för Windows Server behövs det enligt Microsoft inte längre.

**Felaktig ledig/upptagen-visning via Microsoft Graph i hybridmiljöer** (endast Exchange SE). Enligt [KB5135867](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5135867) såg användare med lokala postlådor i klassiska Outlook inte korrekt ledig/upptagen-information för Exchange Online-postlådor när Graph API var aktiverat; endast tidsblock visades. Med Exchange Web Services (EWS) i stället för Graph API stämde visningen. Problemet är åtgärdat med KB5133453 eller en senare uppdatering. Supportartikeln nämner inget efterarbete. Om du på grund av detta fel har växlat tillbaka till EWS, kontrollera efter installationen ledig/upptagen-sökningen för en Exchange Online-postlåda innan du aktiverar Graph API igen.

## Kända problem

Varken tillkännagivandet eller KB5133453 nämner några Known Issues för november-SU:n (status 11 november 2026). Tillkännagivandet har inga kommentarer från Exchange-teamet, och Microsoft påpekar att dokumentationen fortfarande kan vara ofullständig vid publiceringstillfället. Inget Known Issue från tidigare månader är längre öppet: Problemen med septemberuppdateringarna är åtgärdade sedan oktober-SU:n, och Windows-problemet från oktober är åtgärdat med novemberuppdateringarna för Windows Server (se ovan).

## Installation och uppföljning

Microsoft anger den välkända processen i tillkännagivandet: inventera med [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) vilka servrar som behöver uppdateringar eller manuella steg, installera SU:n, starta om servern och kontrollera att alla Exchange-tjänster körs. Om tjänster är inaktiverade efter installationen avbröts installationen. Vid fel under eller efter installationen hänvisar Microsoft till [SetupAssist-skriptet](https://aka.ms/ExSetupAssist). Enligt KB5133453 visar Health Checker efter installationen om SU:n är korrekt installerad och om ytterligare åtgärder behövs.

Installationsfilen heter enligt KB5133453 `ExchangeSubscriptionEdition-KB5133453-x64-en.exe` (Download Center: ”SU12”). SU:n ersätter oktober-SU:n KB5128895. SU:er är kumulativa: Den som har en äldre SU installerad på Exchange SE RTM installerar november-SU:n direkt.

För hybridmiljöer gäller enligt FAQ:n: Om Auth-certifikatet byts efter installationen av en SU måste Hybrid Configuration Wizard köras igen. På Windows Server 2025 visas installerade Exchange-SU:er inte i Kontrollpanelen; Microsoft avråder från att avinstallera SU:er.

Från tidigare månader återstår följande efterarbete, om det ännu inte har utförts: återställ ledig/upptagen-override:n `EnableRouteThroughMSGraphFeature` (se [oktoberartikeln](/blog/exchange-security-updates-oktober-2026)), ta bort Wrapper-SettingOverride `DisableBlockSharedAndUserMailboxHeaders` (se [septemberartikeln](/blog/exchange-security-updates-september-2026)) och kontrollera om CVE-2026-42897-mitigeringen (M2.1.0) fortfarande är aktiv (se [artikeln om juli-SU:n](/blog/exchange-security-updates-juli-2026)).

## Rekommenderad åtgärd

Installera november-SU:n snarast på alla servrar med Exchange SE och på datorerna med Management Tools: För den kritiska Remote Code Execution-sårbarheten CVE-2026-118207 räcker ett konto med låga behörigheter, och Microsoft anser att dess utnyttjande är mer sannolikt. Installera Windows Server-uppdateringarna från november under samma underhållsfönster så att orsaken till avbrotten i oktober åtgärdas. För Exchange 2016 och 2019 finns det inte längre någon patch; enligt Microsoft ska dessa servrar betraktas som permanent sårbara och omedelbart ersättas med Exchange SE.

## Källor

1.  [Released: November 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-november-2026-exchange-server-security-updates/4570368): Tillkännagivande från den 10 november 2026 med version som stöds, information om Exchange 2016/2019, det åtgärdade Windows-problemet och FAQ (inga kommentarer från Exchange-teamet, status 11 november 2026).

2.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM November 10, 2026 (KB5133453) – Microsoft Support](https://support.microsoft.com/help/5133453): CVE-lista, åtgärdat problem, installationsfil, ersatt uppdatering och information om Health Checker.

3.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Typ, CVSS, allvarlighetsgrad, bedömning av utnyttjande, FAQ och berörd produkt för de nio CVE:erna från november.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Buildnummer för november-SU:n.

5.  [Microsoft graph displays incorrect free/busy availability information in Microsoft Exchange server SE hybrid setup – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5135867): Symtom och korrigering med KB5133453.

6.  [Exchange servers may become unstable after installation of October 2026 Windows updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/exchange-servers-may-become-unstable-after-installation-of/ba-p/4565382): Exchange-teamets inlägg om Windows-problemet från oktober, som tillkännagivandet hänvisar till.

7.  [November 10, 2026: KB5131080 (OS Build 26100.33822) – Microsoft Support](https://support.microsoft.com/help/5131080): Release Notes för Windows Server 2025 med Exchange-korrigeringen och Known Issue som markerats som löst.

8.  [November 10, 2026: KB5131090 (OS Build 20348.5870) – Microsoft Support](https://support.microsoft.com/help/5131090): Release Notes för Windows Server 2022 med samma korrigering.

9.  [November 10, 2026: KB5131085 (OS Build 17763.9487) – Microsoft Support](https://support.microsoft.com/help/5131085): Release Notes för Windows Server 2019 med samma korrigering.

10. [November 10, 2026: KB5131291 (OS Build 14393.9688) – Microsoft Support](https://support.microsoft.com/help/5131291): Release Notes för Windows Server 2016 med samma korrigering.

11. [Nya säkerhetsuppdateringar för Exchange Server (november 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-november-2026/): Tyskspråkig genomgång av de nio CVE:erna och bedömning för Exchange 2016 och 2019.

12. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventering av CU/SU-versioner före och efter installationen.
