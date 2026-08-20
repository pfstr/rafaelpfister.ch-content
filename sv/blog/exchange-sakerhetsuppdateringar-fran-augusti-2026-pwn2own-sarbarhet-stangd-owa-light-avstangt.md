---
title: "Exchange-säkerhetsuppdateringar från augusti 2026: Pwn2Own-sårbarhet stängd, OWA Light avstängt"
navTitle: "Exchange SU 08/2026"
description: "Augusti-SU:t åtgärdar sju sårbarheter, däribland Exchange-exploiten som demonstrerades vid Pwn2Own 2026, och inaktiverar OWA Light permanent. Microsoft förklarar även varför Exchange-SU:er nu släpps varje månad och varför Exchange SE CU1 fortfarande dröjer."
date: "2026-08-19"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "6 min lästid"
themen:
  - exchange-onprem-hybrid
produkte:
  - "exchange"
protokolle:
  - "releases"
  - "powershell"
slug: "exchange-sakerhetsuppdateringar-fran-augusti-2026-pwn2own-sarbarhet-stangd-owa-light-avstangt"
translationId: "article-b07bfd4074212673"
draft: false
translationOf: exchange-security-updates-august-2026
url: https://rafaelpfister.ch/sv/blog/exchange-sakerhetsuppdateringar-fran-augusti-2026-pwn2own-sarbarhet-stangd-owa-light-avstangt
translationSourceHash: a41c24b533c3b19bf6226ac5d16e7b9668d83d13b53588da7109f5567e79db51
translationModel: gpt-5.6-terra
translatedAt: 2026-08-20T04:05:07.390Z
translationReview: required
---

# Exchange-säkerhetsuppdateringar från augusti 2026: Pwn2Own-sårbarhet stängd, OWA Light avstängt

Microsoft släppte säkerhetsuppdateringar (SU:er) för Exchange Server den 11 augusti 2026 — för fjärde månaden i rad. Uppdateringarna åtgärdar sju sårbarheter. Ingen av dem var offentligt känd i förväg, ingen utnyttjas aktivt enligt nuvarande uppgifter, och Microsoft bedömer exploatering av samtliga sju som «Exploitation Less Likely». Det är ändå ingen rutinmässig patchdag, av tre skäl: Uppdateringen stänger Exchange-sårbarheten som demonstrerades vid hackingtävlingen Pwn2Own, den **stänger av OWA Light permanent efter nästan tjugo år**, och Exchange-teamet har efteråt förklarat varför månadsrytmen tills vidare blir det normala.

## Vilka Exchange-versioner uppdateringen är tillgänglig för

SU:erna finns tillgängliga för följande versioner:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121573, build 15.2.2562.46 — som en vanligt tillgänglig, offentlig uppdatering.
- **Exchange Server 2019 CU15**: KB5121574, build 15.2.1748.49 — endast via **Period 2 ESU-programmet**.
- **Exchange Server 2019 CU14**: KB5121575, build 15.2.1544.44 — endast via Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5121576, build 15.1.2507.72 — endast via Period 2 ESU.

Situationen är densamma som i juli: Exchange 2016 och 2019 är out of support. SU:erna från maj till oktober 2026 får endast den som är registrerad i Period 2 ESU-programmet. Alla andra förblir opatchade på en nivå med nu fjorton öppna, delvis högt bedömda sårbarheter — migrering till Exchange SE tål inte längre någon uppskjutning där. Exchange Online är redan skyddat; i hybridmiljöer måste SU:t ändå installeras på alla Exchange-servrar, även rena hanteringsservrar och maskiner där endast Exchange Management Tools är installerade.

Det kända problemet med *wrapper-meddelanden* i delade postlådor i hybridmiljöer kvarstår även med augusti-SU:t; enligt Microsoft är korrigeringen planerad för en kommande uppdatering. Det finns åtminstone goda nyheter i kommentarsfältet till release-meddelandet: Den som har satt den dokumenterade SettingOverride-lösningen behöver **inte** skapa den på nytt efter installationen av augusti-SU:t — uppdateringen lämnar override-inställningen orörd, vilket Exchange-teamet bekräftar där.

## De sju sårbarheterna i korthet

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

**[CVE-2026-62913](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62913)** har med CVSS 8.8 månadens högsta värde: en Remote Code Execution som en autentiserad angripare med enkla behörigheter kan utlösa utan någon användarinteraktion. Ett valfritt komprometterat postlådekonto räcker som utgångspunkt — i tider av phishing och credential stuffing är «autentiserad» ingen hög tröskel.

**[CVE-2026-62911](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62911)** är månadens enda sårbarhet som Microsoft klassar som *Critical* (Elevation of Privilege, CVSS 8.0). Bakom det sakliga numret finns mer historia än man kan tro: På frågan om Exchange-exploiten som Orange Tsai demonstrerade vid **Pwn2Own 2026** nu hade åtgärdats hänvisar Exchange-teamet i kommentarsfältet till release-meddelandet just till denna CVE. Tävlingsfyndet är därmed stängt — ytterligare ett skäl att inte låta augusti-SU:t vänta, eftersom Pwn2Own-tekniker normalt publiceras i detalj efter att spärrfristerna löpt ut.

**[CVE-2026-62914](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62914)** (Spoofing, CVSS 7.3) är den direkta anledningen till att OWA Light stängs av — mer om det strax.

De övriga sårbarheterna: CVE-2026-62910 (EoP, 7.2) kräver redan höga behörigheter, CVE-2026-62912 (DoS), CVE-2026-62915 (Security Feature Bypass) och CVE-2026-65813 (EoP) har CVSS 6.5. Detaljerna finns som vanligt i Security Update Guide (filtrera på «Server Software» för Exchange SE respektive «ESU» för 2016/2019).

## OWA Light: slut efter nästan tjugo år

### Vad uppdateringen ändrar

När augusti-SU:t installeras blir **OWA Light permanent inaktiverat** — på varje server som får uppdateringen (eller en senare). Den som öppnar Light-gränssnittet hamnar framöver i vanliga Outlook on the web. Avstängningen är en del av själva uppdateringen och kan inte återställas med en inställning; Microsoft hade annonserat den några veckor tidigare i ett separat blogginlägg.

OWA Light kommer från Exchange 2007-eran: ett medvetet enkelt webbgränssnitt som reserv för gamla webbläsare och långsamma anslutningar, officiellt deprecated sedan augusti 2024. Motiveringen till slutet är säkerhetsdriven: En separat äldre renderingsväg vid sidan av moderna OWA ökar komplexiteten och därmed attackytan — CVE-2026-62914 är det konkreta beviset. Den som läst [juliartikeln](/blog/exchange-security-updates-juli-2026) minns dessutom: Redan CVE-2026-42897-mitigeringen från maj hade gjort OWA Light obrukbart som bieffekt. Gränssnittet var alltså redan på väg bort.

### Om du inte kan patcha: stäng av OWA Light manuellt

Viktigt för alla som (ännu) inte kan installera augusti-SU:t — exempelvis eftersom ESU-aktiveringen saknas: Microsoft rekommenderar uttryckligen att i detta fall **inaktivera OWA Light manuellt** för att minska CVE-2026-62914. Det görs via OWA-postlådepolicyn och inloggningssidan:

```powershell
Get-OwaMailboxPolicy | Set-OwaMailboxPolicy -OWALightEnabled $false
Get-OwaVirtualDirectory | Set-OwaVirtualDirectory -LogonPageLightSelectionEnabled $false
```

Det första kommandot stänger av Light-versionen för alla postlådor i respektive policy, det andra tar bort alternativet «Använd Light-versionen» från OWA-inloggningssidan. Ändringar i den virtuella OWA-katalogen träder tillförlitligt i kraft först efter en återvinning av OWA-app-poolen respektive en `iisreset`.

### Vad administratörer bör kontrollera nu

Avstängningen är tekniskt trivial, men inte alltid organisatoriskt: OWA Light var en tyst livlina för nischscenarier. Kontrollera nu bokmärken och helpdesk-anvisningar som har `?layout=light` hårdkodat, kiosk- och terminalenheter med gamla webbläsare samt interna instruktioner för användare som använt Light-versionen av tillgänglighetsskäl. Moderna Outlook on the web fungerar i alla aktuella webbläsare och har egna tillgänglighetsfunktioner — men den som inte informerar berörda användare i förväg skapar supportärenden.

## Varför ett SU nu kommer varje månad — och var Exchange SE CU1 är

Två dagar efter releasen besvarade Exchange-teamet frågan som många administratörer ställer i ett anmärkningsvärt öppet blogginlägg («Where is Exchange SE CU1 anyway?»). Kortversionen: Microsoft använder AI-verktyg i hela koncernen för att hitta sårbarheter i sina egna produkter. Teamen — inklusive Exchange — arbetar för närvarande igenom de rapporterade fynden: validerar, reproducerar, åtgärdar, testar för regressioner och levererar månadsvis. Sedan maj 2026 har därför ett Exchange-SU släppts varje månad, och Microsoft säger uttryckligen: Detta högre tempo kommer att fortsätta.

Det länge efterlängtade **CU1 för Exchange SE** försenas just därför. Ursprungligen aviserat för första halvåret 2026 och sedan flyttat till det andra, finns det nu inte längre något måldatum. Microsoft vill släppa CU1 först när det finns en månad utan en akut säkerhetsleverans emellan — ett CU som omedelbart överträffas av ett SU skulle innebära dubbelt uppdateringsarbete för många organisationer. Fram till dess förs den månatliga säkerhetspayloaden löpande in i den interna CU1-bygget.

I praktiken innebär det två saker. För det första: Att vänta på CU1 är ingen strategi — varken för migreringen till SE eller för installationen av SU:erna. För det andra: Ett **månatligt underhållsfönster** för Exchange ska från och med nu vara en fast del av driftkalendern, precis som det sedan länge är självklart för Windows-servrar.

## Installation och efterarbete

Förfarandet är det beprövade: Inventera först med [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) vilka servrar som har vilken CU/SU-nivå och om manuella steg återstår. Installera sedan SU:t (om CU-nivån är föråldrad visar [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) vägen), starta om servern och kontrollera att alla Exchange-tjänster har startat korrekt. Om tjänster är *inaktiverade* avbröts installationen — då hjälper den dokumenterade lösningen i Microsoft-supportartikeln om File-Version-felet respektive [SetupAssist-skriptet](https://aka.ms/ExSetupAssist). Avsluta med att köra Health Checker igen.

SU:er är kumulativa: Den som hoppade över juli-SU:t installerar augusti-SU:t direkt. Och för hybridmiljöer gäller det välkända tillägget: Om Auth-certifikatet byts efter SU-installationen bör Hybrid Configuration Wizard köras igen.

Ett efterarbete från juli är fortfarande aktuellt: Den som fortfarande har CVE-2026-42897-mitigeringen (M2.1.0) aktiv bör ta bort den nu — hur detta görs korrekt finns i [artikeln om juli-SU:t](/blog/exchange-security-updates-juli-2026).

## Rekommenderat tillvägagångssätt

Kort sammanfattat: Installera augusti-SU:t snarast på alla Exchange-servrar och maskiner med Management Tools — Pwn2Own-sårbarheten och RCE:n med 8.8 är tillräckliga skäl att inte vänta på nästa patchdag. Den som inte kan patcha omedelbart inaktiverar OWA Light manuellt som en omedelbar åtgärd mot CVE-2026-62914. Identifiera och informera berörda användargrupper före avstängningen av OWA Light (gamla bokmärken, kioskwebbläsare, tillgänglighetsarbetsflöden). Kör sedan Health Checker, slutför kvarvarande efterarbete från juli — och planera in ett månatligt Exchange-underhållsfönster, eftersom rytmen består.

## Källor

1.  [Released: August 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-august-2026-exchange-server-security-updates/4543951): Officiellt release-meddelande med versioner som stöds, information om OWA Light, Known Issues och FAQ; i kommentarerna finns bekräftelserna om Pwn2Own-korrigeringen (CVE-2026-62911) och den kvarstående Wrapper-SettingOverride-inställningen.

2.  [Upcoming retirement of OWA Light in Exchange Server – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/upcoming-retirement-of-owa-light-in-exchange-server/4534943): Förhandsmeddelandet om avstängningen samt Microsofts rekommendation att inaktivera OWA Light manuellt om patchen uteblir.

3.  [Where is Exchange SE CU1 anyway? – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/where-is-exchange-se-cu1-anyway/4546837): Exchange-teamet om AI-stödd sårbarhetssökning, den fortsatta månatliga rytmen för SU:er och CU1-förseningen.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Referens för build-numren för augusti-SU:erna.

5.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Villkor och löptid (maj till oktober 2026) för ESU-programmet.

6.  [Wrapper messages appear in shared mailbox inbox in hybrid environments – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Det sedan juni kända hybridproblemet och SettingOverride-lösningen.

7.  [Neue Sicherheitsupdates für Exchange Server (August 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-august-2026/): Tyskspråkig genomgång av de sju CVE:erna med CVSS-värden och builds.

8.  [Set-OwaMailboxPolicy (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-owamailboxpolicy): Parametern `OWALightEnabled` för manuell inaktivering av Light-versionen.

9.  [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventering av CU/SU-nivåer och återstående manuella steg före och efter installationen.
