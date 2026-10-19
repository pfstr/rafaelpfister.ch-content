---
title: "Exchange-avbrott efter Windows Server-uppdateringarna i oktober 2026: MapiExceptionMdbOffline (ec=1142)"
navTitle: "Exchange-avbrott 10/2026"
description: "Efter Windows Server-uppdateringarna den 13 oktober 2026 rapporterar Exchange-servrar efter omkring 24 timmar MapiExceptionMdbOffline (ec=1142): OWA misslyckas, köerna växer och DAG-kopior replikerar inte längre. Orsaken är en ny gräns i RPC Endpoint Mapper; sedan den 17 oktober 2026 rekommenderar Microsoft registervärdet EpmapSavedContextCap, men någon korrigering saknas fortfarande. Symptom, berörda KB:er, kontrollkommandon och tillvägagångssätt."
date: "2026-10-16"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "10 min lästid"
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
translationSourceHash: 47c39c05826230354e6b431330725587440880dac2b6a33c00f5d34da2afe2d7
translationModel: gpt-5.6-terra
translatedAt: 2026-10-19T07:51:17.301Z
translationReview: automatic
url: https://rafaelpfister.ch/sv/blog/exchange-avbrott-efter-windows-server-uppdateringarna-i-oktober-2026-mapiexceptionmdboffline-ec
---

# Exchange-avbrott efter Windows Server-uppdateringarna i oktober 2026: MapiExceptionMdbOffline (ec=1142)

Sedan patchdagen den 13 oktober 2026 slutar Exchange-servrar att fungera efter cirka 24 timmars drift: klienter förlorar anslutningen till postlådan, OWA rapporterar `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)`, e-post fastnar i köerna och databas­kopior i DAG:ar replikerar inte längre. Microsoft kompletterade tillkännagivandet om oktober-SU:erna för Exchange den 15 oktober 2026 med en hänvisning: utlösaren är oktoberuppdateringarna för Windows Server, och installation av Exchange-SU krävs inte för att problemet ska uppstå. Sedan den 17 oktober 2026 är orsaken och en officiell registeråtgärd kända; någon korrigering finns ännu inte. Mer information finns i följande avsnitt.

## Uppdatering 19 oktober 2026: Orsak och registeråtgärd

Exchange-teamet publicerade ett eget inlägg om problemet den 17 oktober 2026. I versionsinformationen för Windows-uppdateringarna (t.ex. KB5126646) anges problemet nu som ett Known Issue med statusen ”olöst”.

### Orsak

Oktoberuppdateringarna begränsar det minne som RPC Endpoint Mapper använder för Endpoint-lookup-anrop. Som standard tillåts nu endast 512 sparade kontexter; enligt Microsoft räcker detta inte för belastningar som Exchange. Exchange samlar dessa kontexter under drift tills gränsen nås. Därefter misslyckas RPC-anslutningarna till postlåde­databasen och OWA rapporterar `MapiExceptionMdbOffline`. En läsare hos Born har påvisat det nya konfigurationsvärdet `EpmapSavedContextCap` i `RpcEpMap.dll` för oktoberuppdateringarna. Frankys Web kopplar gränsen till sårbarheten CVE-2026-96300 för överbelastningsattacker i RPC Endpoint Mapper, som stängdes samma patchdag.

Detta förklarar flera observationer från de första dagarna:

- Avbrottet inträffar först efter flera timmar, och tidpunkten beror på belastningen. Enligt Microsoft påverkas vissa servrar aldrig.
- En omstart av servern återställer räknaren och hjälper därför endast tills gränsen nås igen. Enbart en omstart av Exchange-tjänsterna frigör enligt en läsare hos Born inte det upptagna minnet.
- Gränsen finns i de kumulativa Windows-uppdateringarna. Exchange-SU och .NET-uppdateringen är inte inblandade.
- Det misstänkta sambandet med databasunderhållsuppgifterna och replikeringstjänsten är därmed inte orsaken.

### Åtgärd

Microsoft rekommenderar att höja gränsen via registret och sedan starta om servern. I en kommandotolk med administratörsbehörighet:

```cmd
reg add "HKLM\SOFTWARE\Microsoft\Rpc" /v EpmapSavedContextCap /t REG_DWORD /d 65536 /f
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `"HKLM\SOFTWARE\Microsoft\Rpc"` | Registernyckel för RPC-konfigurationen. |
| `/v EpmapSavedContextCap` | Namn på värdet som anger gränsen för sparade Endpoint Mapper-kontexter (utan post gäller 512). |
| `/t REG_DWORD` | Datatyp: 32-bitars tal. |
| `/d 65536` | Ny gräns enligt Microsofts rekommendation. |
| `/f` | Skriver över ett befintligt värde utan bekräftelse. |

</details>

Endpoint Mapper läser värdet endast vid start; ändringen får ingen effekt utan omstart av servern. Om problemet återkommer efteråt rekommenderar Microsoft att värdet höjs ytterligare. Enligt Microsoft har åtgärden löst problemet i de kundärenden som hittills hanterats. Enligt Born beskriver Nino Bilic (Microsoft) posten som en tillfällig lösning som inte innebär någon säkerhetsrisk, men som inte är den slutliga lösningen. En läsare hos Born rapporterar en server som har körts utan avbrott i över 26 timmar med 65536. Enskilda administratörer använder högre värden som 16777215; det finns hittills inget dokumenterat fall där 65536 inte räckte efter en omstart.

Följande fråga i PowerShell visar om värdet är angivet:

```powershell
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Rpc" -Name EpmapSavedContextCap
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-Path "HKLM:\SOFTWARE\Microsoft\Rpc"` | Registernyckeln där värdet finns. |
| `-Name EpmapSavedContextCap` | Läser endast detta värde; om det saknas rapporterar PowerShell ett fel, och standardgränsen 512 gäller. |

</details>

Ange värdet på alla Exchange-servrar, även på sådana som ännu inte har drabbats av avbrott. På servrar som ännu inte har oktoberuppdateringarna kan värdet anges före installationen; omstarten efter installationen aktiverar det då. I en DAG startar du om medlemmarna en i taget enligt beskrivningen nedan. Frankys Web tillhandahåller dessutom ett PowerShell-skript som anger värdet, säkerhetskopierar tidigare tillstånd och kan återställas med `-Remove`.

Enligt Microsoft ska Windows-uppdateringarna uttryckligen inte avinstalleras. En läsare hos Frankys Web rapporterar dessutom RDP-inloggningsproblem (CredSSP) efter avinstallation av KB5126868. Om registeråtgärden inte räcker erbjuder Microsoft en ytterligare åtgärd via Support for business. Så snart en korrigering kommer bör du ta bort registervärdet igen, eftersom den högre gränsen försvagar skyddet mot CVE-2026-96300.

### Öppna frågor

- **Korrigering**: Microsoft anger inget datum. En läsare hos Born misstänker att Microsoft kan återställa gränsen via Known Issue Rollback (Feature-ID 63096240); detta är inte bekräftat.
- **Värde**: Det är oklart om 65536 även räcker i stora miljöer fram till nästa patchdag. Fortsätt därför att övervaka Event ID 1006.
- **Andra serverroller**: En läsare hos Born ser även servrar utan Exchange som drabbade. Microsoft uttalar sig endast om Exchange.
- **Säkerhetskopiering**: Det är oklart om det rapporterade VSS Writer-felet är en följd av avbrottet.

## Symptom

De symptom som hittills rapporterats:

- OWA misslyckas med `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)`; Event ID 1006 finns i Application Log.
- Outlook-klienter förlorar anslutningen till postlådan.
- E-post levereras varken internt eller externt, och transportköerna växer.
- I DAG:ar replikerar databas­kopiorna inte längre; enligt läsarrapporter hos Born misslyckas switchover vid övergång till underhållsläge.
- Exchange Admin Center visar ändå databaserna som monterade (Frankys Web).

Felet uppstår inte direkt efter installationen utan enligt rapporter efter ungefär 22 till 30 timmars drift. En omstart av servern återställer driften, men avbrottet upprepas efter ungefär samma tid. I kommentarer hos Born observeras att postlåde­replikeringstjänsten avslutas under eller efter databasunderhållsuppgifterna och att MdbOffline-felen börjar därefter; enligt den numera kända orsaken är detta en bieffekt, inte utlösaren.

## Berörda uppdateringar

Microsoft nämner följande kumulativa Windows-uppdateringar i tillägget till Exchange-tillkännagivandet:

| Windows Server | Uppdatering |
| --- | --- |
| Windows Server 2025 | KB5126646 |
| Windows Server 2022 | KB5126655 |
| Windows Server 2019 | KB5126651 |
| Windows Server 2016 | KB5126868 |

Samma patchdag kom även Exchange-SU:erna (Exchange SE: KB5128895, ”SU11”) och kumulativa uppdateringar för .NET Framework. På en berörd Exchange SE med Windows Server 2019 var dessa tre paket installerade:

- Security Update For Exchange Server Subscription Edition (KB5128895)
- 2026-10 Cumulative Update for .NET Framework 3.5, 4.7.2 and 4.8 for Windows Server 2019 for x64 (KB5128923)
- 2026-10 Cumulative Update for Windows Server 2019 for x64-based Systems (KB5126651)

Microsoft nämner inte .NET-uppdateringen i tillägget. Läsare hos Born rapporterar dock avbrott med kombinationen av Windows- och .NET-uppdatering (där KB5128928 på Windows Server 2022) samt en server med KB5126651 och Exchange-SU men utan .NET-uppdatering som fortfarande körde efter över 24 timmar. Det är därför oklart om .NET-uppdateringen är inblandad.

Om Exchange-SU: I forum och kommentarer sägs läget förbättras om Exchange SU11 är installerat. Detta kan inte bekräftas; avbrott inträffade även på en server med SU11. Omvänt rapporterar flera administratörer avbrott utan installerat Exchange-SU, vilket stämmer med Microsofts uttalande. Exchange-SU åtgärdar sex sårbarheter, bland annat en kritisk Remote Code Execution via ett manipulerat e-postmeddelande; detaljer finns i [artikeln om oktober-SU](/blog/exchange-security-updates-oktober-2026). Att hålla tillbaka det löser inte problemet.

## Kontrollera status

Fastställ först på varje Exchange-server vilka oktoberuppdateringar som är installerade:

```powershell
$kbs = "KB5126646", "KB5126655", "KB5126651", "KB5126868", "KB5128923"
Get-HotFix -Id $kbs -ErrorAction SilentlyContinue |
  Select-Object PSComputerName, HotFixID, InstalledOn
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `$kbs`, `-Id` | Lista över de sökta KB-numren: de fyra Windows-CU:erna från Microsofts tillägg och .NET-uppdateringen för Windows Server 2019. |
| `-ErrorAction SilentlyContinue` | Undertrycker felmeddelandet för KB-nummer som inte är installerade på denna server. |
| `Select-Object PSComputerName, HotFixID, InstalledOn` | Visar server, KB-nummer och installationsdatum; datumet kan användas för att uppskatta när nästa avbrott kan förväntas. |

</details>

`Get-HotFix` listar inte .NET-uppdateringar tillförlitligt på alla system; vid tveksamhet visar uppdateringshistoriken i Inställningar den fullständiga statusen. Kontrollera Exchange-versionen i Exchange Management Shell med `Get-ExchangeServer | Format-Table Name, AdminDisplayVersion` och jämför den med [Exchange-buildnummer](/tools/exchange-builds).

Kontrollera därefter status för databaser, kopior och köer i Exchange Management Shell:

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
| `Get-MailboxDatabaseCopyStatus -Server $env:COMPUTERNAME` | Status för alla databaskopior på den lokala servern; utan DAG visas endast den aktiva kopian. |
| `CopyQueueLength`, `ReplayQueueLength` | Antal ännu inte kopierade respektive ännu inte uppspelade loggfiler; stigande värden visar att replikeringen har stannat. |
| `ContentIndexState` | Status för kopians sökindex. |
| `Get-Queue \| Where-Object MessageCount -gt 0` | Visar endast köer som innehåller e-post. |
| `LastError` | Senaste köfelet; vid detta problem vanligen anslutningsfel till postlåde­databasen. |
| `-FilterHashtable @{LogName='Application'; Id=1006; …}` | Söker i Application Log efter Event ID 1006, enligt Microsofts anvisning. |
| `StartTime=(Get-Date).AddDays(-2)` | Begränsar sökningen till de senaste två dagarna, alltså tiden sedan patchdagen. |

</details>

Flera Exchange-komponenter loggar Event ID 1006; avgörande är posten med MdbOffline-meddelandet i texten.

## Tillvägagångssätt fram till korrigeringen

Följande punkter är från den 16 oktober 2026, då det ännu inte fanns någon officiell åtgärd. Sedan den 17 oktober 2026 har registeråtgärden från uppdateringen ovan högsta prioritet; punkterna 3 till 6 gäller fortsatt som reservåtgärder.

1. **Exchange-servrar som ännu inte har uppdaterats**: Ursprungligen var rekommendationen att skjuta upp Windows-oktoberuppdateringarna. Med registeråtgärden är detta inte längre nödvändigt: ange värdet och installera uppdateringarna, som också åtgärdar säkerhetsbrister.

2. **Installera ändå Exchange-SU**: Exchange-SU KB5128895 (respektive ESU-paketen för 2016/2019) är enligt Microsoft inte utlösaren och åtgärdar en kritisk sårbarhet som kan utnyttjas via e-post utan konto.

3. **Redan drabbade servrar**: En omstart återställer driften i omkring ett dygn. Planera kontrollerade omstarter med betydligt mindre än 22 timmars intervall, till exempel varje natt, i stället för att vänta på nästa avbrott. Enbart en omstart av Exchange-tjänsterna hjälper enligt läsare hos Born också tillfälligt; hur länge är inte belagt.

4. **DAG**: Starta om medlemmarna en i taget och flytta de aktiva databaserna i förväg till en nyss omstartad server. Kontrollera efter varje omstart med `Get-MailboxDatabaseCopyStatus` att kopiorna åter är `Healthy` och att köerna minskar innan nästa server startas om.

5. **Övervaka köer**: Håll koll på `Get-Queue` och Event ID 1006, helst med larm, så att ett avbrott inte först upptäcks genom användarrapporter.

6. **Kontrollera säkerhetskopiering**: En läsare hos Born rapporterar en misslyckad VSS Writer i säkerhetskopieringen efter avbrottet. Kontrollera säkerhetskopiorna från de senaste nätterna.

Om avinstallation av Windows-uppdateringarna: Sedan 2021 levererar Microsoft den kumulativa uppdateringen tillsammans med Servicing Stack Update som ett kombinerat paket. `wusa /uninstall` tar inte bort ett sådant paket; för det krävs `DISM /Online /Remove-Package` med paketnamnet från `DISM /Online /Get-Packages`. Microsoft avråder uttryckligen från avinstallation sedan den 17 oktober 2026; med registeråtgärden finns det inte längre någon anledning till det.

Microsoft listar nu problemet som ett Known Issue i versionsinformationen för Windows-uppdateringarna (status 19 oktober 2026, olöst). Artikeln uppdateras så snart en korrigering finns tillgänglig.

## Källor

1.  [Released: October 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-october-2026-exchange-server-security-updates/4563014): Tillägg från den 15 oktober 2026 med berörda Windows-uppdateringar, felmeddelande, Event ID 1006 och uppgiften att Exchange-SU inte krävs för problemet.

2.  [Exchange Server SE: Problems with the October updates for Windows Server – Frankys Web](https://www.frankysweb.de/en/exchange-server-se-problems-with-the-october-update-kb5128895/): Avbrott efter cirka 24 timmar, databaser visas fortfarande som monterade i EAC, omstart som tillfällig lösning, kommentarer om DAG-omstarter.

3.  [Achtung: Exchange Server SE Okt. 2026-SU führen zu Ausfällen – Borns IT- und Windows-Blog](https://borncity.com/blog/2026/10/15/achtung-exchange-server-okt-2026-su-fuehren-zu-ausfaellen/): Läsarrapporter om tidpunkt, berörda KB:er (även .NET), replikeringstjänst, underhållsläge och VSS-fel.

4.  [Oktober-Updates setzen Exchange-Server ausser Betrieb – IT Magazine](https://www.itmagazine.ch/artikel/88263/Oktober-Updates_setzen_Exchange-Server_ausser_Betrieb.html): Schweizisk sammanfattning av Microsofts bekräftelse.

5.  [Exchange servers may become unstable after installation of October 2026 Windows updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/exchange-servers-may-become-unstable-after-installation-of-october-2026-windows-/4565382): Inlägg från Exchange-teamet den 17 oktober 2026 med registervärdet EpmapSavedContextCap, rekommendationen att inte avinstallera Windows-uppdateringarna och uppgiften om belastningsberoende effekter.

6.  [October 13, 2026: KB5126646 (OS Build 26100.33650) – Microsoft Support](https://support.microsoft.com/en-us/servicing/os/windows-server/2026/10/kb5126646-windows-server-2025-security-update): Known Issue med gränsen på 512 Endpoint-lookup-anrop, status olöst, ytterligare åtgärd via Microsoft Support for business.

7.  [Microsoft analysiert Exchange-Server-Probleme durch Windows-Oktober-2026-Updates – Borns IT- und Windows-Blog](https://borncity.com/blog/2026/10/17/microsoft-analysiert-exchange-server-probleme-durch-windows-oktober-2026-updates/): Läsaranalys av RpcEpMap.dll (EpmapSavedContextCap, Feature-ID), uttalande från Nino Bilic, erfarenheter med 65536 och högre värden.

8.  [CVE-2026-96300 – OpenCVE](https://app.opencve.io/cve/CVE-2026-96300): RPC Endpoint Mapper Service Denial of Service Vulnerability, CWE-770, publicerad den 13 oktober 2026.

9.  [Exchange-Sicherheitsupdates vom Oktober 2026 – rafaelpfister.ch](/blog/exchange-security-updates-oktober-2026): Innehåll i Exchange-SU KB5128895 och ESU-paketen, sårbarheter och installationsanvisningar.
