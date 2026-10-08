---
title: "Exchange-säkerhetsuppdateringar från september 2026: nio sårbarheter, wrapper-problem åtgärdat, v2 släppt i efterhand"
navTitle: "Exchange SU 09/2026"
description: "September-SU:t åtgärdar nio sårbarheter i Exchange SE och 2019 (åtta i Exchange 2016), däribland en spoofing-sårbarhet med CVSS 9.3, och löser wrapper-problemet i hybridmiljöer. Den 2 oktober följde en v2 med ytterligare en CVE; dessutom finns tre Known Issues med lösningar och en SettingOverride som nu bör tas bort."
date: "2026-10-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "7 min lästid"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
  - "powershell"
slug: "exchange-sakerhetsuppdateringar-fran-september-2026-nio-sarbarheter-wrapper-problem-atgardat-v2"
translationId: article-53db0c02bc33f9bd
translationOf: exchange-security-updates-september-2026
url: https://rafaelpfister.ch/sv/blog/exchange-sakerhetsuppdateringar-fran-september-2026-nio-sarbarheter-wrapper-problem-atgardat-v2
translationSourceHash: d0738f61713a26973457a9e536720b9787af35b22867c648dbe113d2aeb3f4ea
translationModel: gpt-5.6-terra
translatedAt: 2026-10-08T10:46:25.381Z
translationReview: automatic
---

# Exchange-säkerhetsuppdateringar från september 2026: nio sårbarheter, wrapper-problem åtgärdat, v2 släppt i efterhand

Microsoft publicerade säkerhetsuppdateringar (SU:er) för Exchange Server den 8 september 2026. De åtgärdar nio sårbarheter i Exchange SE och Exchange 2019, och åtta i Exchange 2016. Ingen av dem var offentligt känd i förväg, ingen utnyttjas aktivt enligt Security Update Guide, och Microsoft klassar alla som *Important* med «Exploitation Less Likely». Det högsta CVSS-värdet på 9.3 ligger dock betydligt över föregående månads. Den här månaden är särskild av tre skäl: SU:t löser problemet med *wrapper-meddelanden* i delade postlådor, som varit öppet sedan juni, det medför tre nya eller kvarstående Known Issues, och den 2 oktober släppte Microsoft en **Version 2** som åtgärdar ytterligare en sårbarhet.

## Vilka Exchange-versioner uppdateringen är tillgänglig för

SU:erna från den 8 september 2026 finns tillgängliga för följande versioner:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121608, build 15.2.2562.49; offentligt tillgänglig.
- **Exchange Server 2019 CU15**: KB5121609, build 15.2.1748.51; endast via **Period-2-ESU-programmet**.
- **Exchange Server 2019 CU14**: KB5121610, build 15.2.1544.46; endast via Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5121611, build 15.1.2507.73; endast via Period 2 ESU.

Exchange 2016 och 2019 är out of support. Enligt Microsoft får endast organisationer som är registrerade i Period-2-ESU-programmet SU:erna från maj till oktober 2026. Enligt KB-artiklarna gäller denna behörighet till oktober 2026. Därtill kommer trycket från Exchange Online: sedan den andra veckan i september begränsar och blockerar Exchange Online hybrid-e-postflödet från servrar under nivån från oktober 2025, detaljer finns i [artikeln om transport-enforcement](/blog/exchange-online-transport-enforcement-hybrid-server). Exchange Online är enligt tillkännagivandet redan skyddat; i hybridmiljöer behöver dock varje Exchange-server SU:t, liksom datorer med Exchange Management Tools.

Du kan jämföra din egen version med översikten över [Exchange-buildnummer](/tools/exchange-builds).

## Översikt över sårbarheterna

| CVE | Typ | CVSS |
| --- | --- | --- |
| CVE-2026-69356 | Spoofing (Cross-Site Scripting) | 9.3 |
| CVE-2026-69641 | Elevation of Privilege | 9.1 |
| CVE-2026-69355 | Remote Code Execution | 8.8 |
| CVE-2026-55007 | Remote Code Execution | 8.1 |
| CVE-2026-69380 | Elevation of Privilege | 8.1 |
| CVE-2026-69378 | Denial of Service | 7.5 |
| CVE-2026-69361 | Spoofing (Server-Side Request Forgery) | 6.5 |
| CVE-2026-69375 | Tampering | 6.5 |
| CVE-2026-69382 | Information Disclosure | 5.9 |

CVE-2026-55007 påverkar inte Exchange 2016; Security Update Guide listar endast Exchange SE och 2019 CU14/CU15 för denna sårbarhet. En detalj om dokumentationen: CVE-2026-69380 saknas i CVE-listan i KB-artiklarna för de fyra SU:erna från september. Security Update Guide anger dock exakt de fyra septemberbyggena som åtgärd för denna CVE (per den 7 oktober 2026).

**[CVE-2026-69356](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69356)** har det högsta värdet med CVSS 9.3. Enligt Microsoft kan en oautentiserad angripare skicka en preparerad kalenderinbjudan med en skadlig möteslänk; när mottagaren öppnar mötet och väljer länken för att ansluta utlöses Cross-Site Scripting. Det krävs alltså användarinteraktion, men inget konto i organisationen.

**[CVE-2026-69380](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69380)** (Elevation of Privilege, CVSS 8.1) kräver endast ett konto med begränsade rättigheter och en tilldelad postlåda. Enligt FAQ:n i Security Update Guide kan en angripare, genom svagheter i kontrollen av förfrågningar och identity tokens, utge sig för att vara en annan användare och ta över alla Exchange-användares postlådor: läsa och skicka e-post samt ladda ned bilagor. Ett enda komprometterat användarkonto räcker som utgångspunkt.

**[CVE-2026-69641](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69641)** (Elevation of Privilege, CVSS 9.1) leder till samma resultat, övertagande av alla postlådor, men kräver medlemskap i en rollgrupp med höga behörigheter.

För de två Remote Code Execution-sårbarheterna är utgångsläget olika: [CVE-2026-69355](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69355) (CVSS 8.8) kräver ett autentiserat konto med låga rättigheter, medan [CVE-2026-55007](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-55007) (CVSS 8.1) kan utlösas utan inloggning genom en preparerad Visio-bilaga, men enligt Microsoft förutsätter ihållande brist på arbetsminne på målsystemet. De övriga fyra sårbarheterna: CVE-2026-69378 (DoS genom okontrollerad rekursion, utan inloggning), CVE-2026-69361 (SSRF, servern skickar HTTP-förfrågningar till interna system eller loopback-system), CVE-2026-69375 (en autentiserad angripare kan ersätta filinnehåll) och CVE-2026-69382 (röjande av inloggningsuppgifter genom en svag kryptografisk algoritm, förutsätter en redan stulen autentiseringscookie).

## Version 2 från den 2 oktober: CVE-2026-96940 tillagd

Den 2 oktober 2026 publicerade Microsoft «Version 2» av SU:erna från september. Enligt tillkännagivandet är den enda skillnaden jämfört med den första versionen den ytterligare korrigeringen för **[CVE-2026-96940](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940)**. Denna Elevation of Privilege-sårbarhet har CVSS 8.8, är varken offentligt känd eller utnyttjad, men är den enda av de tio CVE:erna som Microsoft bedömer som **«Exploitation More Likely»**. En autentiserad angripare kan därmed få åtkomst till andra postlådor i samma organisation och läsa e-post med bilagor. Exchange Online är redan korrigerat på serversidan.

| Version | KB | Build v2 |
| --- | --- | --- |
| Exchange SE RTM | KB5129955 | 15.2.2562.53 |
| Exchange 2019 CU15 | KB5129956 | 15.2.1748.53 |
| Exchange 2019 CU14 | KB5129957 | 15.2.1544.48 |
| Exchange 2016 CU23 | KB5129958 | 15.1.2507.75 |

I praktiken innebär detta: korrigeringen för CVE-2026-96940 finns endast i v2-byggena. Servrar som redan kör SU:t från den 8 september behöver även v2. Den som patchar först nu kan installera v2 direkt, eftersom SU:er är kumulativa. KB-artiklarna säger inte uttryckligen om servrar med det första september-SU:t måste installera v2; eftersom den nya CVE:n endast åtgärdas med v2 är det den rimliga tolkningen.

## Wrapper-problemet åtgärdat: ta nu bort SettingOverride

Det sedan juni-SU:t kända problemet att *wrapper-meddelanden* visas i inkorgen för delade postlådor i hybridmiljöer är åtgärdat med september-SU:t på alla fyra versioner. I [augustiartikeln](/blog/exchange-security-updates-august-2026) stod det fortfarande att den dokumenterade SettingOverride-lösningen kunde vara kvar. Efter installationen av september-SU:t gäller motsatsen: Microsoft rekommenderar i den tillhörande supportartikeln att kontrollera och ta bort override-inställningen.

```powershell
Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Kommando | Effekt |
|---|---|
| `Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Kontrollerar om workaround-override-inställningen är satt i organisationen. |
| `Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Tar bort override-inställningen så snart september-SU:t är installerat. |

</details>

Om det första kommandot rapporterar att objektet `DisableBlockSharedAndUserMailboxHeaders` inte hittades behöver inget ytterligare göras enligt Microsoft.

För Exchange SE åtgärdar SU:t dessutom ett fel vid hybridfrågor om ledig/upptagen via Microsoft Graph: On-premises-användare såg upptagna tider i Exchange Online-postlådor förskjutna med sin egen UTC-förskjutning, utan felmeddelande.

## Kända problem

**Publicerade kalendrar (.ics) ger HTTP 500 i kalenderappar.** Problemet finns sedan augusti-SU:t (Exchange SE från build 15.2.2562.46 samt Exchange 2019 och 2016) och är inte åtgärdat vare sig i september-SU:t eller i v2. Prenumerationer på anonymt publicerade kalendrar uppdateras inte längre; samma URL fungerar i webbläsaren. Orsaken enligt Microsoft: Exchange identifierar klienter via User-Agent, och kalenderappar utan webbläsaridentifiering hamnar i en kodväg som augusti-SU:t har inaktiverat. Som workaround beskriver Microsoft en URL-rewrite-regel på webbplatsen «Exchange Back End» i IIS, som vid .ics-förfrågningar under `/owa/calendar/` lägger till parametern `layout=premium`; IIS-modulen URL Rewrite måste vara installerad. Exakta steg (via IIS Manager eller direkt i `applicationHost.config`) finns i [supportartikeln KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672). Microsoft anger inget datum för korrigeringen (per den 7 oktober 2026).

**Ledig/upptagen för delegerade postlådor i hybridmiljöer (endast Exchange SE).** Om tillgänglighetsfrågan är konfigurerad enbart via Graph API misslyckas frågor för Exchange Online-postlådor via delegerad On-premises-åtkomst. Outlook rapporterar «Your server location could not be determined», OWA visar «No information», och i EWS-loggarna visas `(403) Forbidden`. Den dokumenterade workaround-lösningen leder frågorna åter via EWS i stället för Graph:

```powershell
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=False"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-Identity EnableRouteThroughMSGraphFeature` | Override-inställningen som styr vidarebefordran av tillgänglighetsfrågor via Microsoft Graph. |
| `-Parameters "Enabled=False"` | Stänger av Graph-sökvägen, så att frågorna åter går via EWS. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Riktar diagnostikanropet till topologitjänsten. |
| `-Component VariantConfiguration -Argument Refresh` | Läser in Variant Configuration på nytt så att override-inställningen tillämpas utan väntan. |

</details>

I tillkännagivandet om v2 listar Microsoft detta problem bland de åtgärdade problemen. Den som har satt workaround-inställningen bör efter installation av v2 kontrollera i [supportartikeln KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) om den bör återställas; där fanns ännu ingen anvisning för detta per den 7 oktober 2026.

**ContentEngine-deadlock på grund av saknade koreanska WordBreaker-regelfiler (endast Exchange SE).** Med september-SU:t (build 15.2.2562.49 och v2-build 15.2.2562.53) installeras inte regelfilerna för den uppdaterade koreanska WordBreaker. Följderna är saknade sökresultat, fördröjd e-postleverans och Outlook- eller MAPI-klienter som hänger sig eller förlorar anslutningen. Enligt tillkännagivandet berör det meddelanden på koreanska. Workaround-lösningen i [supportartikeln KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098) består i att hämta de två filerna `ko.token.rule.bin` och `ko.complex.rule.bin` från SQL Server 2025 Express RTM, kontrollera deras SHA256-hashar, kopiera dem till katalogen `Native` i Exchange-installationen och starta om tjänsten Search Host Controller. Microsoft undersöker fortfarande problemet.

## Installation och efterarbete

Microsoft rekommenderar det välkända förfarandet: inventera med [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), fastställ uppdateringsvägen vid en föråldrad version med [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard), installera SU:t, starta om servern och kontrollera att alla Exchange-tjänster körs. Health Checker visar sedan även om SU:t har installerats korrekt. Security Update Guide anger att uppdateringarna kräver omstart.

Efter installationen finns tre uppgifter att utföra:

1. Ta bort Wrapper-SettingOverride `DisableBlockSharedAndUserMailboxHeaders` om den är satt (se ovan).

2. Kontrollera på Exchange SE om problemen med ledig/upptagen och WordBreaker uppstår, och tillämpa vid behov workaround-lösningarna.

3. För publicerade kalendrar med externa prenumeranter: konfigurera URL-rewrite-regeln från KB5126672, om detta inte redan gjorts sedan augusti-SU:t.

Från juli återstår även kontrollen av om CVE-2026-42897-mitigeringen (M2.1.0) fortfarande är aktiv; hur den tas bort beskrivs i [artikeln om juli-SU:t](/blog/exchange-security-updates-juli-2026).

## Rekommenderat tillvägagångssätt

Installera v2-byggena från den 2 oktober direkt på alla Exchange-servrar och datorer med Management Tools; servrar med SU:t från den 8 september behöver dessutom v2 för CVE-2026-96940. Spoofing-sårbarheten med CVSS 9.3 och övertagandet av postlådor via ett användarkonto med låga rättigheter (CVE-2026-69380) är tillräckliga skäl att inte vänta till nästa patchdag. Ta sedan bort wrapper-override-inställningen, kontrollera de tre Known Issues och kör Health Checker. ESU-programmet för Exchange 2016 och 2019 upphör i oktober 2026; migreringen till Exchange SE kan inte skjutas upp längre.

## Källor

1.  [Released: September 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-september-2026-exchange-server-security-updates/4554411): Officiellt release-tillkännagivande med versioner som stöds, ESU-information, Known Issues, åtgärdade problem och installationsförfarande (konsulterad via RSS-flödet för Exchange Team Blog).

2.  [Released: September 2026 V2 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-september-2026-v2-exchange-server-security-updates/ba-p/4561718): Tillkännagivande om v2 från den 2 oktober 2026; enda skillnaden är CVE-2026-96940.

3.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM: September 8, 2026 (KB5121608) – Microsoft Support](https://support.microsoft.com/help/5121608): CVE-lista, åtgärdade problem och de tre Known Issues för Exchange SE.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU15: September 8, 2026 (KB5121609) – Microsoft Support](https://support.microsoft.com/help/5121609): KB-artikel för Exchange 2019 CU15.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU14: September 8, 2026 (KB5121610) – Microsoft Support](https://support.microsoft.com/help/5121610): KB-artikel för Exchange 2019 CU14.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: September 8, 2026 (KB5121611) – Microsoft Support](https://support.microsoft.com/help/5121611): KB-artikel för Exchange 2016 CU23, utan CVE-2026-55007.

7.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Typ, CVSS, allvarlighetsgrad, bedömning av utnyttjande och FAQ om alla nio CVE:er från september samt CVE-2026-96940, inklusive berörda byggen för varje CVE.

8.  [Description of version 2 of the security update for Microsoft Exchange Server Subscription Edition RTM October 2, 2026 (KB5129955) – Microsoft Support](https://support.microsoft.com/help/5129955): KB-artikel om v2 för Exchange SE; v2-artiklarna för 2019 och 2016 är KB5129956 till KB5129958.

9.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Buildnummer för september-SU:erna och v2 från den 2 oktober 2026.

10. [Wrapper messages appear in shared mailbox in hybrid environments after installing the June 2026 Security Update – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Korrigering i september-SU:t och anvisningar för att ta bort SettingOverride.

11. [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Orsak och URL-rewrite-workaround.

12. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Symtom och SettingOverride-workaround för Exchange SE.

13. [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Berörda SE-byggen och manuell workaround.

14. [Hybrid free/busy through Microsoft Graph incorrectly shifts busy times by requester timezone – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5125804): Tidszonsfelet i Exchange SE som åtgärdats med september-SU:t.

15. [Neue Sicherheitsupdates für Exchange Server (September 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-september-2026/): Tyskspråkig genomgång av de nio CVE:erna med CVSS-värden och byggen.

16. [Exchange Server: Sicherheitsupdates 8. September 2026 – Borns Tech and Windows World](https://borncity.com/blog/?p=329371): Tyskspråkig sammanfattning med hänvisningar till den senare ersättningsuppdateringen.
