---
title: "Exchange-säkerhetsuppdateringar för augusti 2026: Pwn2Own-sårbarhet åtgärdad, OWA Light avstängt"
navTitle: "Exchange SU 08/2026"
description: "Augusti-SU:t åtgärdar sju sårbarheter, däribland Exchange-exploiten som demonstrerades vid Pwn2Own 2026, och inaktiverar OWA Light permanent. Microsoft förklarar också varför Exchange-SU:er nu kommer varje månad och varför Exchange SE CU1 fortfarande dröjer."
date: "2026-08-19"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "6 min läsning"
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
translationSourceHash: fcce9c9e87a2b2cd669e4d2647b79283a9e29727194593a7208e977c56763c97
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T15:52:44.569Z
translationReview: required
url: https://rafaelpfister.ch/sv/blog/exchange-sakerhetsuppdateringar-fran-augusti-2026-pwn2own-sarbarhet-stangd-owa-light-avstangt
---

# Exchange-säkerhetsuppdateringar för augusti 2026: Pwn2Own-sårbarhet åtgärdad, OWA Light avstängt

Microsoft släppte säkerhetsuppdateringar (SU:er) för Exchange den 11 augusti 2026, för fjärde månaden i rad. Uppdateringarna åtgärdar sju sårbarheter. Ingen av dem var offentligt känd i förväg, ingen utnyttjas aktivt enligt nuvarande uppgifter, och Microsoft bedömer sannolikheten för exploatering som «Exploitation Less Likely» för samtliga sju. Ändå är detta ingen rutinmässig patchdag, av tre skäl: Uppdateringen åtgärdar Exchange-sårbarheten som demonstrerades i hackingtävlingen Pwn2Own, den **stänger slutgiltigt av OWA Light efter nästan tjugo år**, och Exchange-teamet har i efterhand förklarat varför månadstakten tills vidare blir det normala.

## Vilka Exchange-versioner uppdateringen finns för

SU:erna finns tillgängliga för följande versioner:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121573, build 15.2.2562.46; som en reguljärt tillgänglig offentlig uppdatering.
- **Exchange Server 2019 CU15**: KB5121574, build 15.2.1748.49; endast via **Period 2 ESU-programmet**.
- **Exchange Server 2019 CU14**: KB5121575, build 15.2.1544.44; endast via Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5121576, build 15.1.2507.72; endast via Period 2 ESU.

Situationen är densamma som i juli: Exchange 2016 och 2019 har nått end of support. SU:erna från maj till oktober 2026 får endast de som är registrerade i Period 2 ESU-programmet. Alla andra förblir opatchade på en nivå med nu fjorton öppna, delvis högt klassade sårbarheter; migreringen till Exchange SE kan inte längre skjutas upp där. Exchange Online är redan skyddat; i hybridmiljöer måste SU:t ändå installeras på alla Exchange-servrar, även på rena administrationsservrar och på maskiner där endast Exchange Management Tools är installerade.

Det kända problemet med *wrapper-meddelanden* i delade postlådor i hybridmiljöer kvarstår även med augusti-SU:t; enligt Microsoft är en åtgärd planerad för en kommande uppdatering. Det finns åtminstone ett lugnande besked i kommentarsfältet till release-meddelandet: Den som har ställt in den dokumenterade SettingOverride-lösningen behöver **inte** skapa den på nytt efter installationen av augusti-SU:t. Uppdateringen lämnar override-inställningen orörd, vilket Exchange-teamet bekräftar där.

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

**[CVE-2026-62913](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62913)** har månadens högsta värde med CVSS 8.8: en Remote Code Execution som en autentiserad angripare med enkla behörigheter kan utlösa utan någon användarinteraktion. Ett valfritt komprometterat postlådekonto räcker som utgångspunkt; i tider av nätfiske och credential stuffing är «autentiserad» ingen hög tröskel.

**[CVE-2026-62911](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62911)** är månadens enda sårbarhet som Microsoft klassar som *Critical* (Elevation of Privilege, CVSS 8.0). Bakom den torra siffran finns mer historia än man kan ana: På frågan om Exchange-exploiten som Orange Tsai demonstrerade vid **Pwn2Own 2026** nu hade åtgärdats, hänvisar Exchange-teamet i kommentarsfältet till release-meddelandet just till denna CVE. Tävlingsfyndet är därmed åtgärdat: ytterligare ett skäl att inte låta augusti-SU:t ligga kvar, eftersom Pwn2Own-tekniker vanligtvis publiceras i detalj när spärrfristerna löper ut. Detta har nu också inträffat: Ett proof of concept är offentligt och BSI rapporterar omkring 85 procent sårbara on-premises-servrar i Tyskland. Hur attacken fungerar tekniskt (MRSProxy utan Channel Binding, NTLM-relay) och vad som ligger bakom siffrorna beskrivs i [den utförliga artikeln om CVE-2026-62911](/blog/cve-2026-62911-exchange-ntlm-relay).

**[CVE-2026-62914](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62914)** (Spoofing, CVSS 7.3) är den direkta orsaken till att OWA Light stängs av, mer om det strax.

De övriga sårbarheterna: CVE-2026-62910 (EoP, 7.2) förutsätter redan höga behörigheter, medan CVE-2026-62912 (DoS), CVE-2026-62915 (Security Feature Bypass) och CVE-2026-65813 (EoP) har CVSS 6.5. Detaljerna finns som vanligt i Security Update Guide (filtrera på «Server Software» för Exchange SE respektive «ESU» för 2016/2019).

## OWA Light: slut efter nästan tjugo år

### Vad uppdateringen ändrar

När augusti-SU:t installeras blir **OWA Light permanent inaktiverat**: på varje server där uppdateringen (eller en senare) installeras. Den som öppnar Light-gränssnittet kommer i fortsättningen till vanliga Outlook on the web. Avstängningen är en del av själva uppdateringen och kan inte återställas med någon växel; Microsoft hade annonserat den några veckor tidigare i ett separat blogginlägg.

OWA Light kommer från Exchange 2007-eran: ett medvetet enkelt webbgränssnitt som reservlösning för gamla webbläsare och långsamma anslutningar, officiellt deprecated sedan augusti 2024. Motiveringen till slutet är säkerhetsdriven: En separat äldre renderingsväg vid sidan av moderna OWA ökar komplexiteten och därmed attackytan; CVE-2026-62914 ger det konkreta beviset. Den som har läst [juliartikeln](/blog/exchange-security-updates-juli-2026) minns dessutom att CVE-2026-42897-mitigeringen från maj redan då hade gjort OWA Light obrukbart som bieffekt. Gränssnittet stod alltså redan på tur.

### För dem som inte kan patcha: stäng av OWA Light manuellt

Viktigt för alla som (ännu) inte kan installera augusti-SU:t, exempelvis eftersom ESU-aktiveringen saknas: Microsoft rekommenderar uttryckligen att OWA Light i så fall **inaktiveras manuellt** för att mildra CVE-2026-62914. Det görs via OWA-postlådepolicyn och inloggningssidan:

```powershell
Get-OwaMailboxPolicy | Set-OwaMailboxPolicy -OWALightEnabled $false
Get-OwaVirtualDirectory | Set-OwaVirtualDirectory -LogonPageLightSelectionEnabled $false
```

Det första kommandot stänger av Light-versionen för alla postlådor i respektive policy, det andra tar bort valet «Använd Light-versionen» från OWA-inloggningssidan. Ändringar i den virtuella OWA-katalogen får tillförlitligt genomslag först efter en återvinning av OWA-app-poolen eller en `iisreset`.

### Vad administratörer bör kontrollera nu

Avstängningen är tekniskt trivial, men organisatoriskt inte alltid: OWA Light var den tysta reservlösningen för nischscenarier. Det är nu värt att kontrollera bokmärken och helpdesk-anvisningar som har `?layout=light` fast inbyggt, kiosk- och terminalenheter med gamla webbläsare samt interna anvisningar för användare som använde Light-versionen av tillgänglighetsskäl. Moderna Outlook on the web fungerar i alla aktuella webbläsare och har egna tillgänglighetsfunktioner; men den som inte informerar berörda användare i förväg skapar supportärenden.

## Varför det nu kommer ett SU varje månad och var Exchange SE CU1 blir av

Två dagar efter releasen besvarade Exchange-teamet i ett anmärkningsvärt öppet blogginlägg («Where is Exchange SE CU1 anyway?») frågan som många administratörer ställer sig. Kortversionen: Microsoft använder AI-verktyg i hela koncernen för att hitta sårbarheter i sina egna produkter. Teamen, inklusive Exchange, hanterar för närvarande de rapporterade fynden: validerar, reproducerar, åtgärdar, testar för regressioner och levererar månadsvis. Sedan maj 2026 har ett Exchange-SU släppts varje månad, och Microsoft säger uttryckligen: Denna högre takt kommer att fortsätta.

Det länge efterlängtade **CU1 för Exchange SE** försenas just därför. Ursprungligen annonserat för första halvåret 2026 och sedan flyttat till det andra, finns det nu inte längre något måldatum. Microsoft vill släppa CU1 först när en månad utan brådskande säkerhetsleverans infaller; ett CU som omedelbart blir omsprunget av ett SU skulle innebära dubbelt uppdateringsarbete för många organisationer. Fram till dess integreras den månatliga säkerhetspayloaden löpande i den interna CU1-versionen.

I praktiken betyder det två saker. För det första: Att vänta på CU1 är ingen strategi, varken för migreringen till SE eller för installation av SU:er. För det andra: Ett **månatligt underhållsfönster** för Exchange hör från och med nu permanent hemma i driftkalendern, precis som det sedan länge är självklart för Windows-servrar.

## Installation och efterarbete

Processen förblir den beprövade: Inventera först med [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) vilka servrar som har vilken CU/SU-nivå och om manuella steg återstår. Installera sedan SU:t (om CU-nivån är föråldrad visar [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) vägen), starta om servern och kontrollera att alla Exchange-tjänster har startat korrekt. Om tjänster står som *inaktiverade* avbröts installationen; då hjälper den dokumenterade lösningen i Microsofts supportartikel om felet med filversionen eller [SetupAssist-skriptet](https://aka.ms/ExSetupAssist). Avsluta med att köra Health Checker igen.

SU:er är kumulativa: Den som hoppade över juli-SU:t kan installera augusti-SU:t direkt. Och för hybridmiljöer gäller det välkända tillägget: Om autentiseringscertifikatet byts efter SU-installationen bör Hybrid Configuration Wizard köras igen.

Ett efterarbete från juli är fortfarande aktuellt: Den som ännu har CVE-2026-42897-mitigeringen (M2.1.0) aktiv bör ta bort den nu; hur man gör det korrekt står i [artikeln om juli-SU:t](/blog/exchange-security-updates-juli-2026).

## Rekommenderat tillvägagångssätt

Kort sammanfattat: Installera augusti-SU:t snarast på alla Exchange-servrar och maskiner med Management Tools: Pwn2Own-sårbarheten och RCE:n med 8.8 är skäl nog att inte vänta till nästa patchdag. Den som inte kan patcha omedelbart kan manuellt inaktivera OWA Light som en omedelbar åtgärd mot CVE-2026-62914. Identifiera och informera berörda användargrupper före avstängningen av OWA Light (gamla bokmärken, kioskwebbläsare, tillgänglighetsarbetsflöden). Kör sedan Health Checker, slutför kvarstående efterarbete från juli och planera in ett månatligt Exchange-underhållsfönster, eftersom takten består.

## Källor

1.  [Released: August 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-august-2026-exchange-server-security-updates/4543951): Officiellt release-meddelande med versioner som stöds, information om OWA Light, kända problem och FAQ; i kommentarerna finns bekräftelserna av Pwn2Own-korrigeringen (CVE-2026-62911) och den kvarstående Wrapper-SettingOverride.

2.  [Upcoming retirement of OWA Light in Exchange Server – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/upcoming-retirement-of-owa-light-in-exchange-server/4534943): Förhandsannonseringen av avstängningen samt Microsofts rekommendation att inaktivera OWA Light manuellt om en patch uteblir.

3.  [Where is Exchange SE CU1 anyway? – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/where-is-exchange-se-cu1-anyway/4546837): Exchange-teamet om AI-stödd sårbarhetssökning, den fortsatta månadstakten för SU:er och CU1-förseningen.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Referens för buildnumren för augusti-SU:erna.

5.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Villkor och löptid (maj till oktober 2026) för ESU-programmet.

6.  [Wrapper messages appear in shared mailbox inbox in hybrid environments – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Hybridproblemet som varit känt sedan juni, inklusive workaround med SettingOverride.

7.  [Neue Sicherheitsupdates für Exchange Server (August 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-august-2026/): Tyskspråkig genomgång av de sju CVE:erna med CVSS-värden och builds.

8.  [Set-OwaMailboxPolicy (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-owamailboxpolicy): Parametern `OWALightEnabled` för manuell inaktivering av Light-versionen.

9.  [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventering av CU/SU-nivåer och öppna manuella steg före och efter installationen.
