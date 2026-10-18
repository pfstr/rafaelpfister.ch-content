---
title: "Exchange-avbrott efter Windows Server-uppdateringarna i oktober 2026: MapiExceptionMdbOffline (ec=1142)"
navTitle: "Exchange-avbrott 10/2026"
description: "Efter Windows Server-uppdateringarna den 13 oktober 2026 rapporterar Exchange-servrar efter cirka 24 timmar MapiExceptionMdbOffline (ec=1142): OWA misslyckas, köer växer och DAG-kopior replikerar inte längre. Microsoft bekräftar sambandet med Windows-uppdateringarna, men någon lösning saknas fortfarande. Symptom, berörda KB:er, kontrollkommandon och tillvägagångssätt."
date: "2026-10-16"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "7 min. lästid"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
  - "exchange-on-premises"
protokolle:
  - "troubleshooting"
  - "releases"
  - "powershell"
slug: "exchange-avbrott-efter-windows-server-uppdateringarna-i-oktober-2026-mapiexceptionmdboffline-ec"
featured: "2026-10-16"
warnung: true
translationId: "article-8fdf2b2744bdbd41"
aiPrompt: |
  Du bist mein Assistent für Exchange Server On-Premises. Seit den Windows-Server-Updates vom Oktober 2026 (KB5126646, KB5126655, KB5126651, KB5126868) melden Exchange-Server MapiExceptionMdbOffline (hr=0x80004005, ec=1142), Event ID 1006, wachsende Warteschlangen und nicht replizierende DAG-Kopien. Hilf mir Schritt für Schritt: ermitteln, welche Oktober-Updates (Windows, .NET Framework, Exchange SU) auf jedem Exchange-Server installiert sind, den Zustand von Datenbanken, Kopien und Warteschlangen prüfen, den Betrieb mit kontrollierten Neustarts überbrücken und die Windows-Updates auf noch nicht betroffenen Servern zurückstellen. Frage zuerst nach Exchange-Version, Windows-Server-Version, Anzahl Server, ob eine DAG im Einsatz ist und wie Updates verteilt werden (WSUS, Intune, manuell).
translationOf: exchange-windows-updates-oktober-2026
url: https://rafaelpfister.ch/sv/blog/exchange-avbrott-efter-windows-server-uppdateringarna-i-oktober-2026-mapiexceptionmdboffline-ec
translationSourceHash: 562adaae54d08dd594a344f91f79a05160af956ab320978cef34097a075cab52
translationModel: gpt-5.6-terra
translatedAt: 2026-10-16T08:19:31.557Z
translationReview: automatic
---

# Exchange-avbrott efter Windows Server-uppdateringarna i oktober 2026: MapiExceptionMdbOffline (ec=1142)

Sedan patchdagen den 13 oktober 2026 slutar Exchange-servrar att fungera efter cirka 24 timmars drifttid: klienter förlorar anslutningen till postlådan, OWA rapporterar `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)`, e-post blir liggande i köerna och databasrepliker i DAG:er replikeras inte längre. Microsoft kompletterade tillkännagivandet av Exchange-SU:erna för oktober den 15 oktober 2026 med en anmärkning: orsaken är oktoberuppdateringarna för Windows Server, och installation av Exchange-SU:t krävs inte för att problemet ska uppstå. Det finns ännu ingen lösning eller officiell workaround (status 16 oktober 2026).

## Symptom

De symptom som hittills har rapporterats:

- OWA misslyckas med `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)`; i Application Log finns Event ID 1006.
- Outlook-klienter förlorar anslutningen till postlådan.
- E-post levereras varken internt eller externt, och transportköerna växer.
- I DAG:er replikeras databasrepliker inte längre; enligt läsarrapporter hos Born misslyckas switchovers vid övergång till underhållsläge.
- Exchange Admin Center visar ändå databaserna som monterade (Frankys Web).

Felet uppstår inte direkt efter installationen, utan enligt rapporter efter cirka 22 till 30 timmars drifttid. En omstart av servern återställer driften, men avbrottet upprepas efter ungefär samma tid. I kommentarer hos Born observeras att postlåde-replikeringstjänsten avslutas under eller efter databasunderhållsuppgifterna och att MdbOffline-felen därefter börjar; detta är inte bekräftat.

## Berörda uppdateringar

Microsoft nämner följande kumulativa Windows-uppdateringar i tillägget till Exchange-tillkännagivandet:

| Windows Server | Uppdatering |
| --- | --- |
| Windows Server 2025 | KB5126646 |
| Windows Server 2022 | KB5126655 |
| Windows Server 2019 | KB5126651 |
| Windows Server 2016 | KB5126868 |

Samma patchdag släpptes även Exchange-SU:erna (Exchange SE: KB5128895, «SU11») och kumulativa uppdateringar för .NET Framework. På en berörd Exchange SE med Windows Server 2019 var dessa tre paket installerade:

- Security Update For Exchange Server Subscription Edition (KB5128895)
- 2026-10 Cumulative Update for .NET Framework 3.5, 4.7.2 and 4.8 for Windows Server 2019 for x64 (KB5128923)
- 2026-10 Cumulative Update for Windows Server 2019 for x64-based Systems (KB5126651)

Microsoft nämner inte .NET-uppdateringen i tillägget. Läsare hos Born rapporterar dock avbrott med kombinationen av Windows- och .NET-uppdatering (där KB5128928 på Windows Server 2022) och en server med KB5126651 och Exchange-SU, men utan .NET-uppdatering, som fortfarande körde efter mer än 24 timmar. Det är därför oklart om .NET-uppdateringen är inblandad.

Om Exchange-SU:t: I forum och kommentarer sägs det att läget förbättras om Exchange SU11 installeras. Det kan inte bekräftas; avbrotten inträffade även på en server med SU11. Omvänt rapporterar flera administratörer avbrott utan installerat Exchange-SU, vilket stämmer överens med Microsofts uttalande. Exchange-SU:t åtgärdar sex sårbarheter, däribland en kritisk Remote Code Execution via ett preparerat e-postmeddelande; detaljer i [artikeln om oktober-SU:t](/blog/exchange-security-updates-oktober-2026). Att hålla tillbaka det löser inte problemet.

## Kontrollera status

Fastställ först på varje Exchange-server vilka av oktoberuppdateringarna som är installerade:

```powershell
$kbs = "KB5126646", "KB5126655", "KB5126651", "KB5126868", "KB5128923"
Get-HotFix -Id $kbs -ErrorAction SilentlyContinue |
  Select-Object PSComputerName, HotFixID, InstalledOn
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `$kbs`, `-Id` | Lista över sökta KB-nummer: de fyra Windows-CU:erna från Microsofts tillägg och .NET-uppdateringen för Windows Server 2019. |
| `-ErrorAction SilentlyContinue` | Undertrycker felmeddelandet för KB-nummer som inte är installerade på denna server. |
| `Select-Object PSComputerName, HotFixID, InstalledOn` | Visar server, KB-nummer och installationsdatum; utifrån datumet går det att uppskatta när nästa avbrott kan förväntas. |

</details>

`Get-HotFix` listar inte .NET-uppdateringar tillförlitligt på alla system; vid osäkerhet visar uppdateringshistoriken i Inställningar den fullständiga statusen. Kontrollera Exchange-versionen i Exchange Management Shell med `Get-ExchangeServer | Format-Table Name, AdminDisplayVersion` och jämför den med [Exchange-buildnummer](/tools/exchange-builds).

Kontrollera sedan statusen för databaser, repliker och köer i Exchange Management Shell:

```powershell
Get-MailboxDatabaseCopyStatus -Server $env:COMPUTERNAME |
  Format-Table Name, Status, CopyQueueLength, ReplayQueueLength, ContentIndexState
Get-Queue | Where-Object MessageCount -gt 0 |
  Format-Table Identity, DeliveryType, Status, MessageCount, LastError
Get-WinEvent -FilterHashtable @{LogName='Application'; Id=1006; StartTime=(Get-Date).AddDays(-2)} |
  Select-Object TimeCreated, ProviderName, Message
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `Get-MailboxDatabaseCopyStatus -Server $env:COMPUTERNAME` | Status för alla databasrepliker på den lokala servern; utan DAG visas endast den aktiva repliken. |
| `CopyQueueLength`, `ReplayQueueLength` | Antal loggfiler som ännu inte har kopierats respektive spelats upp; stigande värden visar att replikeringen står stilla. |
| `ContentIndexState` | Status för repliken sökindex. |
| `Get-Queue \| Where-Object MessageCount -gt 0` | Visar endast köer som innehåller e-post. |
| `LastError` | Senaste fel för kön; vid detta problem vanligtvis anslutningsfel till postlådedatabasen. |
| `-FilterHashtable @{LogName='Application'; Id=1006; …}` | Söker i Application Log efter Event ID 1006, enligt Microsofts anvisning. |
| `StartTime=(Get-Date).AddDays(-2)` | Begränsar sökningen till de senaste två dagarna, alltså tiden sedan patchdagen. |

</details>

Flera Exchange-komponenter loggar Event ID 1006; det avgörande är posten med MdbOffline-meddelandet i texten.

## Tillvägagångssätt tills en lösning finns

Microsoft undersöker enligt egen uppgift problemet tillsammans med Windows-teamet och nämner hittills varken workaround eller rollback. Under tiden:

1. **Exchange-servrar som ännu inte har uppdaterats**: Pausa Windows-oktoberuppdateringarna (och .NET-uppdateringen från samma dag) på Exchange-servrar, i WSUS, Intune eller den egna uppdateringshanteringen. Det är en avvägning: Windows-uppdateringarna åtgärdar också säkerhetsbrister. Exchange-servrar som är åtkomliga från internet bör därför följas särskilt noga.

2. **Installera ändå Exchange-SU:t**: Exchange-SU KB5128895 (respektive ESU-paketen för 2016/2019) är enligt Microsoft inte orsaken och åtgärdar en kritisk sårbarhet som kan utnyttjas via e-post utan konto.

3. **Redan berörda servrar**: En omstart återställer driften i ungefär en dag. Planera kontrollerade omstarter med ett intervall klart under 22 timmar, till exempel varje natt, i stället för att vänta på nästa avbrott. Enligt läsare hos Born hjälper även en omstart av endast Exchange-tjänsterna tillfälligt; hur länge är inte belagt.

4. **DAG**: Starta om medlemmarna en i taget och flytta de aktiva databaserna i förväg till en nyligen omstartad server. Kontrollera efter varje omstart med `Get-MailboxDatabaseCopyStatus` att replikerna åter är `Healthy` och att köerna minskar innan nästa server startas om.

5. **Övervaka köer**: Håll ett öga på `Get-Queue` och Event ID 1006, helst med larm, så att ett avbrott inte först upptäcks genom användarrapporter.

6. **Kontrollera backup**: En läsare hos Born rapporterar en misslyckad VSS-writer i backupen efter avbrottet. Kontrollera säkerhetskopiorna från de senaste nätterna.

Om avinstallation av Windows-uppdateringarna: Sedan 2021 levererar Microsoft den kumulativa uppdateringen tillsammans med Servicing Stack Update som ett kombinerat paket. `wusa /uninstall` tar inte bort ett sådant paket; i stället krävs `DISM /Online /Remove-Package` med paketnamnet från `DISM /Online /Get-Packages`. Microsoft har inte godkänt avinstallationen som workaround, och det finns ännu inga rapporter om en framgångsrik rollback med efterföljande stabil drift. Testa detta, om alls, först på en enskild server.

Enligt Frankys Web listar Microsoft ännu inte problemet som ett Known Issue för Windows-uppdateringarna (status 16 oktober 2026). Håll Microsofts tillkännagivande under uppsikt; Microsoft listar tillägg där längst ned under «Updates to this blog post». Denna artikel uppdateras så snart en lösning eller officiell workaround finns.

## Källor

1.  [Released: October 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-october-2026-exchange-server-security-updates/4563014): Tillägg från den 15 oktober 2026 med de berörda Windows-uppdateringarna, felmeddelande, Event ID 1006 och uppgiften att Exchange-SU:t inte krävs för problemet.

2.  [Exchange Server SE: Problems with the October updates for Windows Server – Frankys Web](https://www.frankysweb.de/en/exchange-server-se-problems-with-the-october-update-kb5128895/): Avbrott efter cirka 24 timmar, databaser visas fortfarande som monterade i EAC, omstart som tillfällig åtgärd, kommentarer om DAG-omstarter.

3.  [Achtung: Exchange Server SE Okt. 2026-SU führen zu Ausfällen – Borns IT- und Windows-Blog](https://borncity.com/blog/2026/10/15/achtung-exchange-server-okt-2026-su-fuehren-zu-ausfaellen/): Läsarrapporter om tidpunkt, berörda KB:er (även .NET), replikeringstjänst, underhållsläge och VSS-fel.

4.  [Oktober-Updates setzen Exchange-Server ausser Betrieb – IT Magazine](https://www.itmagazine.ch/artikel/88263/Oktober-Updates_setzen_Exchange-Server_ausser_Betrieb.html): Schweizisk sammanfattning av Microsofts bekräftelse.

5.  [Exchange-säkerhetsuppdateringar från oktober 2026 – rafaelpfister.ch](/blog/exchange-security-updates-oktober-2026): Innehåll i Exchange-SU:t KB5128895 och ESU-paketen, sårbarheter och installationsanvisningar.
