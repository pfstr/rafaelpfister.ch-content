---
title: "Exchange-feil etter Windows Server-oppdateringene i oktober 2026: MapiExceptionMdbOffline (ec=1142)"
navTitle: "Exchange-feil 10/2026"
description: "Etter Windows Server-oppdateringene 13. oktober 2026 rapporterer Exchange-servere MapiExceptionMdbOffline (ec=1142) etter rundt 24 timer: OWA feiler, køene vokser og DAG-kopier replikerer ikke lenger. Microsoft bekrefter sammenhengen med Windows-oppdateringene, men en løsning mangler fortsatt. Symptomer, berørte KB-er, kontrollkommandoer og fremgangsmåte."
date: "2026-10-16"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "7 min lesetid"
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
slug: "exchange-feil-etter-windows-server-oppdateringene-i-oktober-2026-mapiexceptionmdboffline-ec-1142"
featured: "2026-10-16"
warnung: true
translationId: "article-8fdf2b2744bdbd41"
aiPrompt: |
  Du bist mein Assistent für Exchange Server On-Premises. Seit den Windows-Server-Updates vom Oktober 2026 (KB5126646, KB5126655, KB5126651, KB5126868) melden Exchange-Server MapiExceptionMdbOffline (hr=0x80004005, ec=1142), Event ID 1006, wachsende Warteschlangen und nicht replizierende DAG-Kopien. Hilf mir Schritt für Schritt: ermitteln, welche Oktober-Updates (Windows, .NET Framework, Exchange SU) auf jedem Exchange-Server installiert sind, den Zustand von Datenbanken, Kopien und Warteschlangen prüfen, den Betrieb mit kontrollierten Neustarts überbrücken und die Windows-Updates auf noch nicht betroffenen Servern zurückstellen. Frage zuerst nach Exchange-Version, Windows-Server-Version, Anzahl Server, ob eine DAG im Einsatz ist und wie Updates verteilt werden (WSUS, Intune, manuell).
translationOf: exchange-windows-updates-oktober-2026
url: https://rafaelpfister.ch/no/blog/exchange-feil-etter-windows-server-oppdateringene-i-oktober-2026-mapiexceptionmdboffline-ec-1142
translationSourceHash: 562adaae54d08dd594a344f91f79a05160af956ab320978cef34097a075cab52
translationModel: gpt-5.6-terra
translatedAt: 2026-10-16T08:19:56.558Z
translationReview: automatic
---

# Exchange-feil etter Windows Server-oppdateringene i oktober 2026: MapiExceptionMdbOffline (ec=1142)

Siden oppdateringsdagen 13. oktober 2026 har Exchange-servere falt ut etter rundt 24 timers drift: Klienter mister forbindelsen til postboksen, OWA rapporterer `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)`, e-poster blir liggende i køene, og databasekopiene replikerer ikke lenger i DAG-er. Microsoft la 15. oktober 2026 til en merknad i kunngjøringen av Exchange-SU-ene for oktober: Årsaken er Windows Server-oppdateringene for oktober, og installasjon av Exchange-SU er ikke nødvendig for at problemet skal oppstå. Det finnes fortsatt ingen løsning eller offisiell midlertidig løsning (per 16. oktober 2026).

## Symptomer

Symptomene som hittil er rapportert:

- OWA feiler med `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)`; Event ID 1006 står i Application Log.
- Outlook-klienter mister forbindelsen til postboksen.
- E-poster leveres verken internt eller eksternt, og transportkøene vokser.
- I DAG-er replikerer ikke databasekopiene lenger; ifølge leserrapporter hos Born feiler switchover ved overgang til vedlikeholdsmodus.
- Exchange Admin Center viser likevel databasene som montert (Frankys Web).

Feilen oppstår ikke umiddelbart etter installasjonen, men ifølge rapporter etter omtrent 22 til 30 timers drift. En omstart av serveren gjenoppretter driften, men feilen gjentar seg etter omtrent samme tid. I kommentarer hos Born observeres det at postboksreplikeringstjenesten avsluttes under eller etter databasevedlikeholdsoppgavene, og at MdbOffline-feilene deretter begynner; dette er ikke bekreftet.

## Berørte oppdateringer

Microsoft oppgir disse kumulative Windows-oppdateringene i tillegget til Exchange-kunngjøringen:

| Windows Server | Oppdatering |
| --- | --- |
| Windows Server 2025 | KB5126646 |
| Windows Server 2022 | KB5126655 |
| Windows Server 2019 | KB5126651 |
| Windows Server 2016 | KB5126868 |

På samme oppdateringsdag ble også Exchange-SU-ene (Exchange SE: KB5128895, «SU11») og kumulative oppdateringer for .NET Framework utgitt. På en berørt Exchange SE med Windows Server 2019 var disse tre pakkene installert:

- Security Update For Exchange Server Subscription Edition (KB5128895)
- 2026-10 Cumulative Update for .NET Framework 3.5, 4.7.2 and 4.8 for Windows Server 2019 for x64 (KB5128923)
- 2026-10 Cumulative Update for Windows Server 2019 for x64-based Systems (KB5126651)

Microsoft nevner ikke .NET-oppdateringen i tillegget. Lesere hos Born rapporterer imidlertid om feil med kombinasjonen av Windows- og .NET-oppdatering (der KB5128928 på Windows Server 2022), og om en server med KB5126651 og Exchange-SU, men uten .NET-oppdatering, som fortsatt kjørte etter over 24 timer. Det er derfor uavklart om .NET-oppdateringen er involvert.

Om Exchange-SU: I forum og kommentarer heter det at situasjonen forbedres hvis Exchange SU11 er installert. Dette kan ikke bekreftes; feilene oppstod også på en server med SU11. Omvendt rapporterer flere administratorer om feil uten installert Exchange-SU, noe som stemmer med Microsofts uttalelse. Exchange-SU-en lukker seks sårbarheter, inkludert en kritisk Remote Code Execution via en manipulert e-post; detaljer i [artikkelen om SU-en for oktober](/blog/exchange-security-updates-oktober-2026). Å holde den tilbake løser ikke problemet.

## Kontroller status

Finn først ut hvilke av oktober-oppdateringene som er installert på hver Exchange-server:

```powershell
$kbs = "KB5126646", "KB5126655", "KB5126651", "KB5126868", "KB5128923"
Get-HotFix -Id $kbs -ErrorAction SilentlyContinue |
  Select-Object PSComputerName, HotFixID, InstalledOn
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `$kbs`, `-Id` | Liste over KB-numrene det søkes etter: de fire Windows-CU-ene fra Microsofts tillegg og .NET-oppdateringen for Windows Server 2019. |
| `-ErrorAction SilentlyContinue` | Undertrykker feilmeldingen for KB-numre som ikke er installert på denne serveren. |
| `Select-Object PSComputerName, HotFixID, InstalledOn` | Viser server, KB-nummer og installasjonsdato; datoen kan brukes til å anslå når neste feil kan forventes. |

</details>

`Get-HotFix` viser ikke .NET-oppdateringer pålitelig på alle systemer; ved tvil viser oppdateringsloggen i Innstillinger fullstendig status. Du kontrollerer Exchange-byggversjonen i Exchange Management Shell med `Get-ExchangeServer | Format-Table Name, AdminDisplayVersion` og sammenligner den med [Exchange-byggnumrene](/tools/exchange-builds).

Kontroller deretter status for databaser, kopier og køer i Exchange Management Shell:

```powershell
Get-MailboxDatabaseCopyStatus -Server $env:COMPUTERNAME |
  Format-Table Name, Status, CopyQueueLength, ReplayQueueLength, ContentIndexState
Get-Queue | Where-Object MessageCount -gt 0 |
  Format-Table Identity, DeliveryType, Status, MessageCount, LastError
Get-WinEvent -FilterHashtable @{LogName='Application'; Id=1006; StartTime=(Get-Date).AddDays(-2)} |
  Select-Object TimeCreated, ProviderName, Message
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Get-MailboxDatabaseCopyStatus -Server $env:COMPUTERNAME` | Status for alle databasekopier på den lokale serveren; uten DAG vises bare den aktive kopien. |
| `CopyQueueLength`, `ReplayQueueLength` | Antall loggfiler som ennå ikke er kopiert eller avspilt; økende verdier viser at replikeringen har stoppet. |
| `ContentIndexState` | Status for kopiens søkeindeks. |
| `Get-Queue \| Where-Object MessageCount -gt 0` | Viser bare køer som inneholder e-post. |
| `LastError` | Siste feil i køen; ved dette problemet typisk forbindelsesfeil til postboksdatabasen. |
| `-FilterHashtable @{LogName='Application'; Id=1006; …}` | Søker i Application Log etter Event ID 1006, slik Microsoft oppgir. |
| `StartTime=(Get-Date).AddDays(-2)` | Begrenses søket til de siste to dagene, altså tiden siden oppdateringsdagen. |

</details>

Flere Exchange-komponenter logger Event ID 1006; det avgjørende er oppføringen med MdbOffline-meldingen i teksten.

## Fremgangsmåte frem til løsningen

Microsoft undersøker ifølge egen uttalelse problemet sammen med Windows-teamet og har så langt ikke oppgitt verken en midlertidig løsning eller tilbakeføring. For mellomtiden:

1. **Exchange-servere som ennå ikke er oppdatert**: Utsett Windows-oppdateringene for oktober (og .NET-oppdateringen fra samme dag) på Exchange-servere, i WSUS, Intune eller egen oppdateringsadministrasjon. Dette er en avveiing: Windows-oppdateringene lukker også sikkerhetshull. Exchange-servere som er tilgjengelige fra internett, bør derfor følges ekstra tett.

2. **Installer likevel Exchange-SU**: Exchange-SU KB5128895 (eller ESU-pakkene for 2016/2019) er ifølge Microsoft ikke årsaken og lukker et kritisk sikkerhetshull som kan utnyttes via e-post uten konto.

3. **Allerede berørte servere**: En omstart gjenoppretter driften i rundt én dag. Planlegg kontrollerte omstarter med betydelig mindre enn 22 timers intervall, for eksempel hver natt, i stedet for å vente på neste feil. Ifølge lesere hos Born hjelper også en omstart av bare Exchange-tjenestene midlertidig; hvor lenge er ikke dokumentert.

4. **DAG**: Start medlemmene på nytt etter tur, og flytt de aktive databasene til en nylig omstartet server på forhånd. Kontroller etter hver omstart med `Get-MailboxDatabaseCopyStatus` at kopiene igjen er `Healthy` og at køene reduseres før du starter neste server på nytt.

5. **Overvåk køer**: Følg med på `Get-Queue` og Event ID 1006, helst med varsling, slik at en feil ikke først oppdages gjennom brukermeldinger.

6. **Kontroller sikkerhetskopi**: En leser hos Born rapporterer om en mislykket VSS-skriver i sikkerhetskopien etter feilen. Kontroller sikkerhetskopiene fra de siste nettene.

Om avinstallering av Windows-oppdateringene: Siden 2021 har Microsoft levert den kumulative oppdateringen sammen med Servicing Stack Update som en kombinert pakke. `wusa /uninstall` fjerner ikke en slik pakke; til dette kreves `DISM /Online /Remove-Package` med pakkenavnet fra `DISM /Online /Get-Packages`. Microsoft har ikke godkjent avinstalleringen som en midlertidig løsning, og det foreligger foreløpig ingen rapporter om vellykket tilbakeføring med påfølgende stabil drift. Test dette i så fall først på én enkelt server.

Ifølge Frankys Web oppfører Microsoft ennå ikke problemet som et Known Issue for Windows-oppdateringene (per 16. oktober 2026). Følg med på Microsoft-kunngjøringen; Microsoft fører tillegg der nederst under «Updates to this blog post». Denne artikkelen oppdateres så snart en løsning eller offisiell midlertidig løsning foreligger.

## Kilder

1.  [Released: October 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-october-2026-exchange-server-security-updates/4563014): Tillegg fra 15. oktober 2026 med berørte Windows-oppdateringer, feilmelding, Event ID 1006 og uttalelsen om at Exchange-SU ikke er nødvendig for problemet.

2.  [Exchange Server SE: Problems with the October updates for Windows Server – Frankys Web](https://www.frankysweb.de/en/exchange-server-se-problems-with-the-october-update-kb5128895/): Feil etter omtrent 24 timer, databaser fortsatt vist som montert i EAC, omstart som midlertidig avhjelping og kommentarer om DAG-omstarter.

3.  [Achtung: Exchange Server SE Okt. 2026-SU führen zu Ausfällen – Borns IT- und Windows-Blog](https://borncity.com/blog/2026/10/15/achtung-exchange-server-okt-2026-su-fuehren-zu-ausfaellen/): Leserrapporter om tidspunkt, involverte KB-er (også .NET), replikeringstjeneste, vedlikeholdsmodus og VSS-feil.

4.  [Oktober-Updates setzen Exchange-Server ausser Betrieb – IT Magazine](https://www.itmagazine.ch/artikel/88263/Oktober-Updates_setzen_Exchange-Server_ausser_Betrieb.html): Sveitsisk oppsummering av Microsofts bekreftelse.

5.  [Exchange-Sicherheitsupdates vom Oktober 2026 – rafaelpfister.ch](/blog/exchange-security-updates-oktober-2026): Innholdet i Exchange-SU KB5128895 og ESU-pakkene, sårbarheter og installasjonsmerknader.
