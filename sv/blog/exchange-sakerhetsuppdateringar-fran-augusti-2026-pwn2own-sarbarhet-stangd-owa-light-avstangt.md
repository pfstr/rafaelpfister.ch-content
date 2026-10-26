---
title: "Exchange-säkerhetsuppdateringar för augusti 2026: Pwn2Own-sårbarheten åtgärdad, OWA Light avstängt"
navTitle: "Exchange SU 08/2026"
description: "Augusti-SU åtgärdar sju sårbarheter, däribland Exchange-exploiten som demonstrerades vid Pwn2Own 2026, och inaktiverar OWA Light permanent. Microsoft förklarar också varför Exchange-SU:er nu kommer varje månad och varför Exchange SE CU1 fortfarande dröjer."
date: "2026-08-19"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "6 min. lästid"
themen:
  - exchange-updates
produkte:
  - "exchange"
protokolle:
  - "releases"
  - "powershell"
slug: "exchange-sakerhetsuppdateringar-fran-augusti-2026-pwn2own-sarbarhet-stangd-owa-light-avstangt"
translationId: "article-b07bfd4074212673"
draft: false
translationOf: exchange-security-updates-august-2026
translationSourceHash: 4c2345cf2955df229b8713cf288ec21bba3e1bd43aef297ecad12536e9bf459a
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T08:56:28.869Z
translationReview: required
url: https://rafaelpfister.ch/sv/blog/exchange-sakerhetsuppdateringar-fran-augusti-2026-pwn2own-sarbarhet-stangd-owa-light-avstangt
---

# Exchange-säkerhetsuppdateringar för augusti 2026: Pwn2Own-sårbarheten åtgärdad, OWA Light avstängt

Microsoft släppte säkerhetsuppdateringar (SU:er) för Exchange Server den 11 augusti 2026, för fjärde månaden i rad. Uppdateringarna åtgärdar sju sårbarheter. Ingen av dem var offentligt känd i förväg, ingen utnyttjas aktivt enligt nuvarande information, och Microsoft bedömer exploatering av samtliga sju som «Exploitation Less Likely». Det är ändå ingen rutinmässig patchdag, av tre skäl: uppdateringen åtgärdar Exchange-sårbarheten som demonstrerades i hackingtävlingen Pwn2Own, den **stänger av OWA Light permanent efter nästan tjugo år**, och Exchange-teamet har efteråt förklarat varför den månatliga rytmen tills vidare blir det normala.

## Vilka Exchange-versioner uppdateringen finns för

SU:erna finns tillgängliga för följande versioner:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121573, version 15.2.2562.46; som en reguljärt tillgänglig offentlig uppdatering.
- **Exchange Server 2019 CU15**: KB5121574, version 15.2.1748.49; endast via **Period-2-ESU-programmet**.
- **Exchange Server 2019 CU14**: KB5121575, version 15.2.1544.44; endast via Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5121576, version 15.1.2507.72; endast via Period 2 ESU.

Situationen är densamma som i juli: Exchange 2016 och 2019 stöds inte längre. SU:erna från maj till oktober 2026 får endast de som är registrerade i Period-2-ESU-programmet. Alla andra förblir opatchade med fjorton öppna, delvis högt rankade sårbarheter; en övergång till Exchange SE kan inte längre vänta. Exchange Online är redan skyddat; i hybridmiljöer måste SU:t ändå installeras på alla Exchange-servrar, även på rena hanteringsservrar och på datorer där endast Exchange Management Tools är installerade.

Det kända problemet med *wrapper-meddelanden* i delade postlådor i hybridmiljöer kvarstår även med augusti-SU:t; enligt Microsoft är en korrigering planerad för en kommande uppdatering. Det finns åtminstone ett lugnande besked i kommentarsfältet till versionsannonseringen: den som har angett den dokumenterade SettingOverride som workaround behöver **inte** skapa den på nytt efter installationen av augusti-SU:t. Uppdateringen lämnar override-inställningen orörd, vilket Exchange-teamet bekräftar där.

## Översikt över de sju sårbarheterna

| CVE | Typ | CVSS |
| --- | --- | --- |
| CVE-2026-62913 | Remote Code Execution | 8.8 |
| CVE-2026-62911 | Elevation of Privilege | 8.0 |
| CVE-2026-62914 | Spoofing | 7.3 |
| CVE-2026-62910 | Elevation of Privilege | 7.2 |
| CVE-2026-62912 | Denial of Service | 6.5 |
| CVE-2026-62915 | Security Feature Bypass | 6.5 |
| CVE-2026-65813 | Elevation of Privilege | 6.5 |

Tre av dem förtjänar en närmare titt.

**[CVE-2026-62913](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62913)** har månadens högsta värde med CVSS 8.8: en Remote Code Execution som en autentiserad angripare med enkla behörigheter kan utlösa utan någon användarinteraktion. Ett valfritt komprometterat postlådekonto räcker som utgångspunkt; i tider av nätfiske och credential stuffing är «autentiserad» ingen hög tröskel.

**[CVE-2026-62911](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62911)** är månadens enda sårbarhet som Microsoft klassar som *Critical* (Elevation of Privilege, CVSS 8.0). Bakom det sakliga numret döljer sig mer historia än man kan ana: på frågan om Exchange-exploiten som Orange Tsai demonstrerade vid **Pwn2Own 2026** nu hade åtgärdats, hänvisar Exchange-teamet i kommentarsfältet till versionsannonseringen just till denna CVE. Tävlingsfyndet är därmed åtgärdat: ytterligare ett skäl att inte låta augusti-SU:t ligga, eftersom Pwn2Own-tekniker vanligtvis publiceras i detalj efter att sekretessfristerna har löpt ut. Det har nu precis skett: ett Proof-of-Concept är offentligt, och BSI rapporterar att omkring 85 procent av On-Premises-servrarna i Tyskland är sårbara. Hur angreppet fungerar tekniskt (MRSProxy utan Channel Binding, NTLM-relä) och vad siffrorna bygger på beskrivs i [den utförliga artikeln om CVE-2026-62911](/blog/cve-2026-62911-exchange-ntlm-relay).

**[CVE-2026-62914](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62914)** (Spoofing, CVSS 7.3) är den direkta anledningen till att OWA Light stängs av, mer om det strax.

De övriga sårbarheterna: CVE-2026-62910 (EoP, 7.2) kräver redan höga behörigheter, CVE-2026-62912 (DoS), CVE-2026-62915 (Security Feature Bypass) och CVE-2026-65813 (EoP) har CVSS 6.5. Detaljerna finns som vanligt i Security Update Guide (filtrera på «Server Software» för Exchange SE respektive «ESU» för 2016/2019).

## OWA Light: efter nästan tjugo år är det slut

### Vad uppdateringen ändrar

När augusti-SU:t installeras **inaktiveras OWA Light permanent**: på varje server där uppdateringen (eller en senare) installeras. Den som öppnar Light-gränssnittet hamnar framöver i vanliga Outlook on the web. Avstängningen är en del av själva uppdateringen och kan inte återställas med någon inställning; Microsoft hade meddelat den några veckor tidigare i ett separat blogginlägg.

OWA Light kommer från Exchange 2007-eran: ett avsiktligt enkelt webbgränssnitt som reservlösning för gamla webbläsare och långsamma anslutningar, officiellt deprecated sedan augusti 2024. Motiveringen till att det upphör är säkerhetsdriven: en separat äldre renderingsväg vid sidan av moderna OWA ökar komplexiteten och därmed attackytan; CVE-2026-62914 är ett konkret bevis på detta. Den som har läst [juliartikeln](/blog/exchange-security-updates-juli-2026) minns dessutom: redan CVE-2026-42897-mitigeringen från maj hade gjort OWA Light obrukbart som bieffekt. Gränssnittet var alltså redan på väg att försvinna.

### För dem som inte kan patcha: stäng av OWA Light manuellt

Viktigt för alla som (ännu) inte kan installera augusti-SU:t, exempelvis eftersom ESU-aktiveringen saknas: Microsoft rekommenderar uttryckligen att OWA Light i så fall **inaktiveras manuellt** för att mildra CVE-2026-62914. Det görs via OWA-postlådeprincipen och inloggningssidan:

```powershell
Get-OwaMailboxPolicy | Set-OwaMailboxPolicy -OWALightEnabled $false
Get-OwaVirtualDirectory | Set-OwaVirtualDirectory -LogonPageLightSelectionEnabled $false
```

Det första kommandot stänger av Light-versionen för alla postlådor i respektive princip, det andra tar bort alternativet «Använd Light-versionen» från OWA-inloggningssidan. Ändringar i den virtuella OWA-katalogen träder tillförlitligt i kraft först efter en återvinning av OWA-app-poolen respektive en `iisreset`.

### Vad administratörer bör kontrollera nu

Avstängningen är tekniskt trivial, men inte alltid organisatoriskt: OWA Light var den tysta reservlösningen för nischscenarier. Det är nu värt att kontrollera bokmärken och helpdesk-anvisningar som har `?layout=light` hårdkodat, kiosk- och terminalenheter med gamla webbläsare samt interna anvisningar för användare som har använt Light-versionen av tillgänglighetsskäl. Moderna Outlook on the web fungerar i alla aktuella webbläsare och har egna tillgänglighetsfunktioner; men den som inte informerar berörda användare i förväg skapar supportärenden.

## Varför ett SU nu kommer varje månad och var Exchange SE CU1 blir av

Två dagar efter lanseringen besvarade Exchange-teamet i ett anmärkningsvärt öppet blogginlägg («Where is Exchange SE CU1 anyway?») frågan som många administratörer ställer. Kortversionen: Microsoft använder AI-verktyg i hela koncernen för att hitta sårbarheter i sina egna produkter. Teamen, inklusive Exchange, arbetar för närvarande igenom de rapporterade fynden: validerar, reproducerar, åtgärdar, testar regressioner och levererar varje månad. Sedan maj 2026 har ett Exchange-SU släppts varje månad, och Microsoft säger uttryckligen att detta högre tempo kommer att fortsätta.

Det efterlängtade **CU1 för Exchange SE** fördröjs just av denna anledning. Ursprungligen utannonserat för första halvåret 2026 och sedan flyttat till det andra, finns det nu inte längre något måldatum. Microsoft vill släppa CU1 först när en månad utan brådskande säkerhetsleverans ligger emellan; ett CU som omedelbart ersätts av ett SU skulle innebära dubbelt uppdateringsarbete för många organisationer. Fram till dess införlivas den månatliga säkerhetsnyttolasten löpande i den interna CU1-versionen.

I praktiken betyder detta två saker. För det första: att vänta på CU1 är ingen strategi, varken för migreringen till SE eller för installationen av SU:erna. För det andra: ett **månatligt underhållsfönster** för Exchange hör från och med nu fast hemma i driftkalendern, precis som det sedan länge är självklart för Windows-servrar.

## Installation och efterarbete

Processen är fortsatt den beprövade: inventera först med [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) vilka servrar som har vilken CU/SU-nivå och om manuella åtgärder återstår. Installera sedan SU:t (om CU-nivån är föråldrad visar [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) vägen), starta om servern och kontrollera att alla Exchange-tjänster har startat korrekt. Om tjänster står som *inaktiverade* avbröts installationen; då hjälper den dokumenterade workarounden i Microsofts supportartikel om File Version-felet respektive [SetupAssist-skriptet](https://aka.ms/ExSetupAssist). Avsluta med att köra Health Checker igen.

SU:er är kumulativa: den som hoppade över juli-SU:t kan installera augusti-SU:t direkt. Och för hybridmiljöer gäller det välkända tillägget: om Auth-certifikatet byts efter SU-installationen bör Hybrid Configuration Wizard köras igen.

Ett efterarbete från juli är fortfarande aktuellt: den som fortfarande har CVE-2026-42897-mitigeringen (M2.1.0) aktiv bör ta bort den nu; hur det görs korrekt beskrivs i [artikeln om juli-SU:t](/blog/exchange-security-updates-juli-2026).

## Rekommenderat tillvägagångssätt

Kort sammanfattat: installera augusti-SU:t snarast på alla Exchange-servrar och datorer med Management Tools: Pwn2Own-sårbarheten och RCE:n med 8.8 är skäl nog att inte vänta till nästa patchdag. Den som inte kan patcha direkt: OWA Light kan inaktiveras manuellt som en omedelbar åtgärd mot CVE-2026-62914. Identifiera och informera berörda användargrupper före avstängningen av OWA Light (gamla bokmärken, kioskwebbläsare, tillgänglighetsarbetsflöden). Kör sedan Health Checker, utför återstående efterarbete från juli och planera in ett månatligt Exchange-underhållsfönster, eftersom rytmen fortsätter.

## Källor

1.  [Released: August 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-august-2026-exchange-server-security-updates/4543951): Officiell versionsannonsering med versioner som stöds, information om OWA Light, Known Issues och FAQ; i kommentarerna finns bekräftelserna om Pwn2Own-korrigeringen (CVE-2026-62911) och den kvarstående Wrapper-SettingOverride.

2.  [Upcoming retirement of OWA Light in Exchange Server – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/upcoming-retirement-of-owa-light-in-exchange-server/4534943): Förhandsmeddelandet om avstängningen samt Microsofts rekommendation att manuellt inaktivera OWA Light om patchen uteblir.

3.  [Where is Exchange SE CU1 anyway? – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/where-is-exchange-se-cu1-anyway/4546837): Exchange-teamet om AI-stödd sårbarhetssökning, den fortsatta månatliga rytmen för SU:er och CU1-fördröjningen.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Referens för versionsnumren för augusti-SU:erna.

5.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Villkor och löptid (maj till oktober 2026) för ESU-programmet.

6.  [Wrapper messages appear in shared mailbox inbox in hybrid environments – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Det hybridproblem som är känt sedan juni samt SettingOverride-workarounden.

7.  [Neue Sicherheitsupdates für Exchange Server (August 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-august-2026/): Tyskspråkig genomgång av de sju CVE:erna med CVSS-värden och versioner.

8.  [Set-OwaMailboxPolicy (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-owamailboxpolicy): Parametern `OWALightEnabled` för manuell inaktivering av Light-versionen.

9.  [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventering av CU/SU-nivåer och öppna manuella åtgärder före och efter installationen.
