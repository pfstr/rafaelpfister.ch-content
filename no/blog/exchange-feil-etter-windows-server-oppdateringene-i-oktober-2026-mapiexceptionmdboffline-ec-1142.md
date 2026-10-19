---
title: "Exchange-feil etter Windows Server-oppdateringene i oktober 2026: MapiExceptionMdbOffline (ec=1142)"
navTitle: "Exchange-feil 10/2026"
description: "Etter Windows Server-oppdateringene 13. oktober 2026 rapporterer Exchange-servere MapiExceptionMdbOffline (ec=1142) etter rundt 24 timer: OWA feiler, køene vokser, og DAG-kopier replikerer ikke lenger. Årsaken er en ny grense i RPC Endpoint Mapper; Microsoft anbefaler registerverdien EpmapSavedContextCap siden 17. oktober 2026, men en rettelse mangler fortsatt. Symptomer, berørte KB-er, kontrollkommandoer og fremgangsmåte."
date: "2026-10-16"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "10 min lesetid"
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
translationOf: exchange-windows-updates-oktober-2026
translationSourceHash: 47c39c05826230354e6b431330725587440880dac2b6a33c00f5d34da2afe2d7
translationModel: gpt-5.6-terra
translatedAt: 2026-10-19T07:51:53.840Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/exchange-feil-etter-windows-server-oppdateringene-i-oktober-2026-mapiexceptionmdboffline-ec-1142
---

# Exchange-feil etter Windows Server-oppdateringene i oktober 2026: MapiExceptionMdbOffline (ec=1142)

Siden patchdagen 13. oktober 2026 har Exchange-servere sluttet å fungere etter rundt 24 timers drift: Klienter mister forbindelsen til postboksen, OWA rapporterer `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)`, e-post blir liggende i køene, og i DAG-er replikeres ikke databasekopiene lenger. Microsoft supplerte kunngjøringen om Exchange-SU-ene i oktober 15. oktober 2026 med en merknad: Utløseren er oktoberoppdateringene for Windows Server, og installasjon av Exchange-SU-en er ikke nødvendig for at problemet skal oppstå. Siden 17. oktober 2026 er årsaken og et offisielt mottiltak via registeret kjent; en rettelse finnes fortsatt ikke. Detaljer i neste avsnitt.

## Oppdatering 19. oktober 2026: Årsak og registermottiltak

Exchange-teamet publiserte 17. oktober 2026 et eget innlegg om problemet. I versjonsmerknadene for Windows-oppdateringene (f.eks. KB5126646) er problemet nå oppført som et Known Issue med statusen «uløst».

### Årsak

Oktoberoppdateringene begrenser minnet RPC Endpoint Mapper bruker for endpoint-oppslagskall. Som standard er bare 512 lagrede kontekster tillatt; ifølge Microsoft er dette ikke tilstrekkelig for belastninger som Exchange. Exchange samler disse kontekstene under drift til grensen nås. Deretter feiler RPC-forbindelsene til postboksdatabasen, og OWA rapporterer `MapiExceptionMdbOffline`. En leser hos Born har påvist den nye konfigurasjonsverdien `EpmapSavedContextCap` i `RpcEpMap.dll` for oktoberoppdateringene. Frankys Web knytter grensen til sårbarheten CVE-2026-96300 i RPC Endpoint Mapper, en denial-of-service-sårbarhet som ble lukket på samme patchdag.

Dette forklarer flere observasjoner fra de første dagene:

- Feilen oppstår først etter flere timer, og tidspunktet avhenger av belastningen. Ifølge Microsoft blir enkelte servere aldri berørt.
- En omstart av serveren nullstiller telleren og hjelper derfor bare til grensen nås igjen. Ifølge en leser hos Born frigjør ikke en omstart av bare Exchange-tjenestene det opptatte minnet.
- Grensen ligger i de kumulative Windows-oppdateringene. Exchange-SU og .NET-oppdateringen er ikke involvert.
- Den mistenkte sammenhengen med databasevedlikeholdsoppgavene og replikeringstjenesten er dermed ikke årsaken.

### Mottiltak

Microsoft anbefaler å øke grensen via registeret og deretter starte serveren på nytt. I en ledetekst med administratorrettigheter:

```cmd
reg add "HKLM\SOFTWARE\Microsoft\Rpc" /v EpmapSavedContextCap /t REG_DWORD /d 65536 /f
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `"HKLM\SOFTWARE\Microsoft\Rpc"` | Registernøkkel for RPC-konfigurasjonen. |
| `/v EpmapSavedContextCap` | Navn på verdien som angir grensen for lagrede Endpoint Mapper-kontekster (uten oppføring gjelder 512). |
| `/t REG_DWORD` | Datatype: 32-bits tall. |
| `/d 65536` | Ny grense, slik Microsoft anbefaler. |
| `/f` | Overskriver en eksisterende verdi uten spørsmål. |

</details>

Endpoint Mapper leser verdien bare ved oppstart; endringen virker ikke uten omstart av serveren. Hvis problemet oppstår igjen etterpå, anbefaler Microsoft å øke verdien ytterligere. Ifølge Microsoft har tiltaket løst problemet i kundesakene som hittil er behandlet. Ifølge Born omtaler Nino Bilic (Microsoft) oppføringen som en midlertidig løsning som ikke medfører en sikkerhetsrisiko, men som ikke er den endelige løsningen. En leser hos Born rapporterer om en server som har kjørt uten avbrudd i over 26 timer med 65536. Enkelte administratorer bruker høyere verdier som 16777215; det finnes foreløpig ikke et dokumentert tilfelle der 65536 ikke var tilstrekkelig etter en omstart.

Denne spørringen i PowerShell viser om verdien er satt:

```powershell
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Rpc" -Name EpmapSavedContextCap
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-Path "HKLM:\SOFTWARE\Microsoft\Rpc"` | Registernøkkelen der verdien ligger. |
| `-Name EpmapSavedContextCap` | Leser bare denne verdien; mangler den, rapporterer PowerShell en feil, og standardgrensen på 512 gjelder. |

</details>

Sett verdien på alle Exchange-servere, også de som hittil ikke har feilet. På servere som ennå ikke har oktoberoppdateringene, kan du sette verdien før installasjonen; omstarten etter installasjonen aktiverer den da. I en DAG starter du medlemmene på nytt etter tur, slik det beskrives nedenfor. Frankys Web tilbyr også et PowerShell-skript som setter verdien, sikkerhetskopierer den tidligere tilstanden og kan reverseres med `-Remove`.

Ifølge Microsoft skal Windows-oppdateringene uttrykkelig ikke avinstalleres. En leser hos Frankys Web rapporterer dessuten RDP-påloggingsproblemer (CredSSP) etter avinstallering av KB5126868. Hvis registertiltaket ikke er tilstrekkelig, tilbyr Microsoft et ytterligere mottiltak via Support for business. Så snart en rettelse kommer, bør du fjerne registerverdien igjen, fordi den høyere grensen svekker beskyttelsen mot CVE-2026-96300.

### Uavklarte punkter

- **Rettelse**: Microsoft oppgir ingen dato. En leser hos Born antar at Microsoft kan trekke tilbake grensen via en Known Issue Rollback (Feature-ID 63096240); dette er ikke bekreftet.
- **Verdi**: Det er uavklart om 65536 også er tilstrekkelig i store miljøer frem til neste patchdag. Overvåk derfor fortsatt Event ID 1006.
- **Andre serverroller**: En leser hos Born ser også servere uten Exchange som berørt. Microsoft uttaler seg bare om Exchange.
- **Sikkerhetskopiering**: Det er uavklart om den rapporterte VSS Writer-feilen er en følge av driftsavbruddet.

## Symptomer

Symptomene som hittil er rapportert:

- OWA feiler med `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)`; Event ID 1006 står i Application Log.
- Outlook-klienter mister forbindelsen til postboksen.
- E-post leveres verken internt eller eksternt, og transportkøene vokser.
- I DAG-er replikeres ikke databasekopiene lenger; ifølge leserrapporter hos Born feiler switchover ved overgang til vedlikeholdsmodus.
- Exchange Admin Center viser likevel databasene som montert (Frankys Web).

Feilen oppstår ikke rett etter installasjonen, men ifølge rapporter etter omtrent 22 til 30 timers drift. En omstart av serveren gjenoppretter driften, men feilen gjentar seg etter omtrent samme tid. I kommentarer hos Born observeres det at postboksreplikeringstjenesten stopper under eller etter databasevedlikeholdsoppgavene, og at MdbOffline-feilene deretter begynner; etter den nå kjente årsaken er dette en følgevirkning, ikke utløseren.

## Berørte oppdateringer

Microsoft nevner disse kumulative Windows-oppdateringene i tillegget til Exchange-kunngjøringen:

| Windows Server | Oppdatering |
| --- | --- |
| Windows Server 2025 | KB5126646 |
| Windows Server 2022 | KB5126655 |
| Windows Server 2019 | KB5126651 |
| Windows Server 2016 | KB5126868 |

På samme patchdag ble også Exchange-SU-ene (Exchange SE: KB5128895, «SU11») og kumulative oppdateringer for .NET Framework utgitt. På en berørt Exchange SE under Windows Server 2019 var disse tre pakkene installert:

- Security Update For Exchange Server Subscription Edition (KB5128895)
- 2026-10 Cumulative Update for .NET Framework 3.5, 4.7.2 and 4.8 for Windows Server 2019 for x64 (KB5128923)
- 2026-10 Cumulative Update for Windows Server 2019 for x64-based Systems (KB5126651)

Microsoft nevner ikke .NET-oppdateringen i tillegget. Lesere hos Born rapporterer imidlertid om feil med kombinasjonen av Windows- og .NET-oppdatering (der KB5128928 under Windows Server 2022) og om en server med KB5126651 og Exchange-SU, men uten .NET-oppdatering, som fortsatt kjørte etter over 24 timer. Det er dermed uavklart om .NET-oppdateringen er involvert.

Om Exchange-SU-en: I fora og kommentarer heter det at situasjonen blir bedre dersom Exchange SU11 er installert. Dette kan ikke bekreftes; feilene oppsto også på en server med SU11. Omvendt rapporterer flere administratorer om feil uten installert Exchange-SU, noe som stemmer med Microsofts uttalelse. Exchange-SU-en lukker seks sårbarheter, deriblant en kritisk remote code execution via en manipulert e-post; detaljer i [artikkelen om oktober-SU-en](/blog/exchange-security-updates-oktober-2026). Å holde den tilbake løser ikke problemet.

## Kontroller status

Finn først ut på hver Exchange-server hvilke av oktoberoppdateringene som er installert:

```powershell
$kbs = "KB5126646", "KB5126655", "KB5126651", "KB5126868", "KB5128923"
Get-HotFix -Id $kbs -ErrorAction SilentlyContinue |
  Select-Object PSComputerName, HotFixID, InstalledOn
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `$kbs`, `-Id` | Liste over søkte KB-numre: de fire Windows-CU-ene fra Microsofts tillegg og .NET-oppdateringen for Windows Server 2019. |
| `-ErrorAction SilentlyContinue` | Undertrykker feilmeldingen for KB-numre som ikke er installert på denne serveren. |
| `Select-Object PSComputerName, HotFixID, InstalledOn` | Viser server, KB-nummer og installasjonsdato; datoen gjør det mulig å anslå når neste feil kan forventes. |

</details>

`Get-HotFix` lister ikke .NET-oppdateringer pålitelig på alle systemer; ved tvil viser oppdateringsloggen i Innstillinger den fullstendige statusen. Kontroller Exchange-byggversjonen i Exchange Management Shell med `Get-ExchangeServer | Format-Table Name, AdminDisplayVersion` og sammenlign den med [Exchange-byggnumrene](/tools/exchange-builds).

Kontroller deretter statusen for databaser, kopier og køer i Exchange Management Shell:

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
| `CopyQueueLength`, `ReplayQueueLength` | Antall loggfiler som ennå ikke er kopiert eller spilt inn; økende verdier viser at replikeringen står stille. |
| `ContentIndexState` | Tilstand for kopiens søkeindeks. |
| `Get-Queue \| Where-Object MessageCount -gt 0` | Viser bare køer som inneholder e-post. |
| `LastError` | Siste feil i køen; ved dette problemet typisk forbindelsesfeil til postboksdatabasen. |
| `-FilterHashtable @{LogName='Application'; Id=1006; …}` | Søker i Application Log etter Event ID 1006, slik Microsoft angir. |
| `StartTime=(Get-Date).AddDays(-2)` | Begrenser søket til de siste to dagene, altså tiden siden patchdagen. |

</details>

Flere Exchange-komponenter logger Event ID 1006; det avgjørende er oppføringen med MdbOffline-meldingen i teksten.

## Fremgangsmåte frem til rettelsen

Følgende punkter stammer fra 16. oktober 2026, da det ennå ikke fantes et offisielt mottiltak. Siden 17. oktober 2026 står registertiltaket fra oppdateringen ovenfor først; punkt 3 til 6 er fortsatt gyldige som reservealternativ.

1. **Exchange-servere som ennå ikke er oppdatert**: Opprinnelig var anbefalingen å utsette Windows-oktoberoppdateringene. Med registertiltaket er dette ikke lenger nødvendig: Sett verdien og installer oppdateringene, som også lukker sikkerhetshull.

2. **Installer likevel Exchange-SU**: Exchange-SU KB5128895 (eller ESU-pakkene for 2016/2019) er ifølge Microsoft ikke utløseren og lukker et kritisk hull som kan utnyttes via e-post uten konto.

3. **Allerede berørte servere**: En omstart gjenoppretter driften i rundt én dag. Planlegg kontrollerte omstarter med klart mindre enn 22 timers mellomrom, for eksempel hver natt, i stedet for å vente på neste feil. Ifølge lesere hos Born hjelper en omstart av bare Exchange-tjenestene også midlertidig; hvor lenge er ikke dokumentert.

4. **DAG**: Start medlemmene på nytt etter tur, og flytt først de aktive databasene til en nylig omstartet server. Kontroller etter hver omstart med `Get-MailboxDatabaseCopyStatus` at kopiene igjen er `Healthy` og at køene reduseres før du starter neste server på nytt.

5. **Overvåk køer**: Hold øye med `Get-Queue` og Event ID 1006, helst med varsling, slik at en feil ikke først oppdages gjennom brukermeldinger.

6. **Kontroller sikkerhetskopiering**: En leser hos Born rapporterer en mislykket VSS Writer i sikkerhetskopieringen etter feilen. Kontroller sikkerhetskopiene fra de siste nettene.

Om avinstallering av Windows-oppdateringene: Siden 2021 har Microsoft levert den kumulative oppdateringen sammen med Servicing Stack Update som en kombinert pakke. `wusa /uninstall` fjerner ikke en slik pakke; dette krever `DISM /Online /Remove-Package` med pakkenavnet fra `DISM /Online /Get-Packages`. Microsoft fraråder uttrykkelig avinstallering siden 17. oktober 2026; med registertiltaket er det heller ingen grunn til det.

Microsoft fører nå problemet som et Known Issue i versjonsmerknadene for Windows-oppdateringene (status 19. oktober 2026, uløst). Denne artikkelen oppdateres så snart en rettelse foreligger.

## Kilder

1.  [Released: October 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-october-2026-exchange-server-security-updates/4563014): Tillegg fra 15. oktober 2026 med berørte Windows-oppdateringer, feilmelding, Event ID 1006 og uttalelsen om at Exchange-SU-en ikke er nødvendig for problemet.

2.  [Exchange Server SE: Problems with the October updates for Windows Server – Frankys Web](https://www.frankysweb.de/en/exchange-server-se-problems-with-the-october-update-kb5128895/): Feil etter omtrent 24 timer, databaser fortsatt vist som montert i EAC, omstart som midlertidig løsning, kommentarer om DAG-omstarter.

3.  [Achtung: Exchange Server SE Okt. 2026-SU führen zu Ausfällen – Borns IT- und Windows-Blog](https://borncity.com/blog/2026/10/15/achtung-exchange-server-okt-2026-su-fuehren-zu-ausfaellen/): Leserrapporter om tidspunkt, involverte KB-er (også .NET), replikeringstjeneste, vedlikeholdsmodus og VSS-feil.

4.  [Oktober-Updates setzen Exchange-Server ausser Betrieb – IT Magazine](https://www.itmagazine.ch/artikel/88263/Oktober-Updates_setzen_Exchange-Server_ausser_Betrieb.html): Sveitsisk oppsummering av Microsofts bekreftelse.

5.  [Exchange servers may become unstable after installation of October 2026 Windows updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/exchange-servers-may-become-unstable-after-installation-of-october-2026-windows-/4565382): Innlegg fra Exchange-teamet 17. oktober 2026 med registerverdien EpmapSavedContextCap, anbefalingen om ikke å avinstallere Windows-oppdateringene og merknaden om belastningsavhengige konsekvenser.

6.  [October 13, 2026: KB5126646 (OS Build 26100.33650) – Microsoft Support](https://support.microsoft.com/en-us/servicing/os/windows-server/2026/10/kb5126646-windows-server-2025-security-update): Known Issue med grensen på 512 endpoint-oppslagskall, status uløst, ytterligere mottiltak via Microsoft Support for business.

7.  [Microsoft analysiert Exchange-Server-Probleme durch Windows-Oktober-2026-Updates – Borns IT- und Windows-Blog](https://borncity.com/blog/2026/10/17/microsoft-analysiert-exchange-server-probleme-durch-windows-oktober-2026-updates/): Leseranalyse av RpcEpMap.dll (EpmapSavedContextCap, Feature-ID), uttalelse fra Nino Bilic, erfaringer med 65536 og høyere verdier.

8.  [CVE-2026-96300 – OpenCVE](https://app.opencve.io/cve/CVE-2026-96300): RPC Endpoint Mapper Service Denial of Service Vulnerability, CWE-770, publisert 13. oktober 2026.

9.  [Exchange-Sicherheitsupdates vom Oktober 2026 – rafaelpfister.ch](/blog/exchange-security-updates-oktober-2026): Innholdet i Exchange-SU-en KB5128895 og ESU-pakkene, sårbarheter og installasjonsmerknader.
