---
title: "Riktig etterarbeid etter Exchange-sikkerhetsoppdateringene fra juli 2026"
navTitle: "Exchange SU 07/2026"
description: "Etter installasjonen er to oppryddingsoppgaver nødvendige: Kontroller fjerningen av den gamle CVE-2026-42897-avbøtningen og undersøk overprivilegerte eldre grupper i Active Directory."
date: "2026-07-14"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "6 min. lesetid"
themen:
  - exchange-onprem-hybrid
  - active-directory-entra
slug: "riktig-etterarbeid-etter-exchange-sikkerhetsoppdateringene-fra-juli-2026"
translationOf: "exchange-security-updates-juli-2026"
url: "https://rafaelpfister.ch/no/blog/riktig-etterarbeid-etter-exchange-sikkerhetsoppdateringene-fra-juli-2026"
translationId: article-731b5b840aee096c
translationReview: automatic
translationSourceHash: c4f0a68a6d0b88997bcc5dadd9f5c2423dcb61c7986e179a099460335042a23a
translatedAt: 2026-07-29T12:29:38.971Z
---

# Riktig etterarbeid etter Exchange-sikkerhetsoppdateringene fra juli 2026

Med installasjonen av Exchange-sikkerhetsoppdateringene 14. juli 2026 er arbeidet ikke ferdig. Administratorer bør deretter fjerne to eldre etterlatenskaper: avbøtningen for **CVE-2026-42897**, som ble aktivert i mai, og to historiske Exchange-sikkerhetsgrupper med omfattende rettigheter i Active Directory.

Begge oppgavene er lette å overse. Avbøtningen blir bevisst stående til den fjernes kontrollert. Gruppene kan på sin side ha overlevd enhver migrering ubemerket i mange år.

## Hvilke Exchange-versjoner oppdateringen er tilgjengelig for

SU-ene er tilgjengelige for følgende versjoner:

- **Exchange Server Subscription Edition (SE) RTM**: som en ordinært tilgjengelig, offentlig oppdatering.
- **Exchange Server 2019 CU14 og CU15**: kun for organisasjoner som er registrert i **Period-2-ESU-programmet**.
- **Exchange Server 2016 CU23**: også kun via Period 2 ESU.

Exchange 2016 og 2019 støttes ikke lenger. De som ikke er med i Period-2-ESU-programmet (gyldig fra mai til oktober 2026), mottar ikke lenger disse oppdateringene og bør ikke utsette overgangen til Exchange SE lenger. Exchange Online-miljøer er allerede beskyttet; i hybridoppsett må SU-en likevel installeres på alle Exchange-servere, også rene administrasjonsservere. Hvilke konkrete CVE-er som håndteres, står som vanlig i Security Update Guide (filteret «Server Software» for Exchange SE eller «ESU» for 2016/2019).

Det finnes et kjent problem i den aktuelle utgivelsen: I hybridmiljøer kan såkalte *wrapper-meldinger* vises i innboksen til delte postbokser. Se den aktuelle Microsoft-støtteartikkelen for detaljer.

## Fjern CVE-2026-42897-avbøtningen etter installasjonen

### Kort tilbakeblikk

CVE-2026-42897 ble offentliggjort 14. mai 2026: en cross-site scripting-sårbarhet (spoofing) i Outlook Web Access. En angriper sender en spesialpreparert e-post; hvis offeret åpner den i OWA og bestemte interaksjonsbetingelser er oppfylt, kan vilkårlig JavaScript kjøres i nettleserkonteksten. Exchange 2016, 2019 og SE var berørt på *alle* patchnivåer. Microsoft publiserte samme dag en nødavbøtning (ID **M2.1.x**, den konkrete IIS-regelen heter **M2.1.0**) og leverte den faktiske rettelsen med SU-en fra juni 2026.

### Hvorfor julioppdateringen *ikke* fjerner avbøtningen automatisk

Dette er punktet som overrasker de fleste: Selv etter installasjon av juli-SU-en forblir en allerede anvendt avbøtning aktiv. Årsaken ligger i mekanismen. Avbøtningen er en **Content-Security-Policy-basert IIS URL Rewrite-regel** som ble lagt inn *utenfor* MSI-installasjonsprogrammet, enten av Emergency Mitigation Service (EM Service) eller EOMT-skriptet. MSI-patchen bytter binærfiler, men administrerer ikke disse IIS-reglene som er satt out-of-band. Derfor er fjerningen et eget, manuelt trinn.

For øvrig: Avbøtningen beskyttet aldri IE-klienter og Edge i IE-modus, fordi Internet Explorer ikke støtter CSP. De som hadde slike klienter i bruk, var aldri sikret av avbøtningen alene. Dette er enda et argument for å patche raskt i stedet for å stole på avbøtningen.

### Fallgruven: EM Service legger inn avbøtningen på nytt

De som sletter regelen for tidlig, får en overraskelse. EM Service kjører hver time og sammenligner faktisk tilstand med spesifikasjonene som leveres av Office Config Service (flighting). Tilordningen «hvilken build trenger hvilken avbøtning» ligger på serversiden. Først en endring på serversiden markerer juli 2026-builden som «avbøtning ikke lenger nødvendig». Ifølge Microsoft ble denne endringen først fullt utrullet rundt 16. juli 2026. Frem til da legger EM Service ganske enkelt inn en slettet M2.1.0-regel igjen ved neste timekjøring.

I praksis betyr dette: Enten venter man med manuell fjerning til etter 16. juli, eller så blokkerer man avbøtningen eksplisitt slik at den ikke reaktiveres.

### Slik fjerner du avbøtningen på en ryddig måte (EM Service-bane)

Kontroller først hva som faktisk er anvendt:

```powershell
Get-ExchangeServer -Identity <servernavn> | Format-List Name,MitigationsApplied,MitigationsBlocked
```

For å forhindre reaktivering settes avbøtnings-ID-en på blokkeringslisten: Oppføringer der ignoreres av EM Service i timekjøringen.

```powershell
Set-ExchangeServer -Identity <servernavn> -MitigationsBlocked @("M2.1.0")
```

Fjern deretter selve IIS-regelen. Godt å vite, og sjelden dokumentert: EM Service oppretter sine URL Rewrite-regler med **prefikset «EEMS `<Mitigation-ID>` `<Beschreibung>`»**. Dermed finner du dem entydig i IIS Manager under URL Rewrite (eller via `appcmd`/PowerShell i `applicationHost.config`) uten å måtte gjette hvilken regel som hører til avbøtningen. Etter utrullingen av endringen på serversiden kan du oppheve blokkeringen igjen (`-MitigationsBlocked @()`) dersom den kun ble satt som en midlertidig løsning.

### EOMT-bane (isolerte eller air-gapped-miljøer)

Hvis avbøtningen ble satt via det nedlastbare **EOMT-skriptet** (https://aka.ms/UnifiedEOMT), utføres tilbakeføringen med rollback-bryteren:

```powershell
.\EOMT.ps1 -RollbackMitigation -CVE "CVE-2026-42897"
```

Også her finnes en lite kjent detalj: EOMT sikkerhetskopierer IIS-utgangstilstanden i en **CVE-spesifikk JSON-sikkerhetskopifil** under `%WINDIR%\System32\inetsrv\config\` før hver endring. Rollback leser nøyaktig denne filen og gjenoppretter de opprinnelige innstillingene. Viktig: En avbøtning som er satt med et eldre skript (EOMTv2 osv.), må også fjernes med sin egen rollback-mekanisme: Sikkerhetskopiformatene er ikke kompatible.

### Hvorfor det lønner seg å fjerne den

Avbøtningen er ikke «gratis». Så lenge den er aktiv, tar du med deg de kjente bivirkningene: OWA-funksjonen «Skriv ut kalender» fungerer ikke, innebygde bilder kan i enkelte tilfeller ikke vises riktig i OWA-leseruten, OWA Light (`/?layout=light`) er defekt (og blir uansett snart avviklet), og publiserte kalendere gir delvis feil 500. Særlig vanskelig for overvåking: Healthset **OWACalendar.Proxy** kan bli *unhealthy* og dermed utløse falske alarmer i overvåkingen. De som har installert SU-en, men lar avbøtningen stå, ender opp med å jage spøkelser. Så snart oppdateringen er installert *og* avbøtningen er fjernet, forsvinner også disse kjente problemene.

Et spesialtilfelle: I blandede miljøer kan servere som ennå ikke er oppdatert, beholde avbøtningen. Du bør imidlertid vite at Office Online Server-integrasjonen (OOS) i enkelte tilfeller først fungerer riktig igjen når *alle* Exchange-servere i organisasjonen er på juli-nivå.

## Health Checker: Finn eldgamle sikkerhetsgrupper

Det andre punktet, som er uavhengig av SU-utgivelsen: **Exchange Health Checker** (https://aka.ms/ExchangeHealthChecker) kontrollerer nå om to for lengst avviklede sikkerhetsgrupper finnes: **«Exchange Domain Servers»** og **«Exchange Enterprise Servers»**.

### Hvor disse gruppene kommer fra, og hvorfor de utgjør en risiko

Disse to gruppene stammer fra rettighetsmodellen i Exchange 2000/2003 og har vært avviklet siden Exchange 2007. Med Exchange 2007/2010 kom modellen for delte rettigheter og RBAC, og siden da har de rett og slett ikke vært brukt. Problemet er at de ikke forsvant av den grunn. I mange kataloger har de ligget ubemerket i omtrent to tiår og har delvis fortsatt omfattende ACL-er fra den gamle modellen, altså flere rettigheter enn en moderne Exchange-sikkerhetsgruppe noen gang ville hatt.

Nettopp dette gjør dem til en angrepsvektor. En sovende gruppe med eksisterende, brede rettigheter er en klassisk eskaleringskjede: Den som klarer å legge seg selv (eller en kontrollert konto) til i en slik gruppe, arver rettighetene dens i katalogen. Siden ingen aktivt overvåker gruppen, blir en slik manipulering knapt oppdaget.

### Hvorfor de fleste administratorer ikke har dem på radaren

Disse gruppene er en blindflekk av flere grunner: De har vært inaktive i rundt 20 år, eksisterte som regel allerede før dagens team tiltrådte, overlever uten problemer enhver migrering og har hittil aldri blitt vist av Health Checker. Særlig kritisk: De overlever til og med *fullstendig* avvikling av lokal Exchange. De som har fjernet den siste Exchange-serveren, rydder som regel bort serverobjektene, men overser disse eldre gruppene fullstendig.

### Opprydding

Health Checker vil fremover automatisk rapportere gruppene. Manuelt finner du dem i Active Directory (vanligvis i `Users`-containeren) eller via PowerShell:

```powershell
Get-ADGroup -Filter "Name -eq 'Exchange Domain Servers' -or Name -eq 'Exchange Enterprise Servers'"
```

Fremgangsmåte: Kontroller medlemskap og eventuelle egendefinerte ACL-referanser, sørg for at ingenting i produksjon refererer til dem, og slett deretter gruppene. Siden de har vært avviklet siden 2007, kan de fjernes uten risiko i det overveldende flertallet av miljøer. De som ikke lenger drifter lokal Exchange i det hele tatt, bør samtidig planlegge en mer omfattende AD-opprydding i henhold til Microsofts offisielle veiledning.

Hayes Jupe har skrevet en detaljert veiledning om fjerning av gruppene i blogginnlegget [Latest Exchange health check script and deprecated groups](https://www.hayesjupe.com/latest-exchange-health-check-script-and-deprecated-groups/).

## Anbefalt fremgangsmåte

Kort oppsummert er den praktiske rekkefølgen: Inventer først miljøet med Health Checker (den viser manglende CU-er/SU-er, åpne manuelle trinn *og* nå de eldre gruppene). Installer deretter gjeldende CU og juli-SU-en, start serveren på nytt og kontroller at alle Exchange-tjenester har startet riktig. Kjør så Health Checker på nytt, fjern CVE-2026-42897-avbøtningen (etter 16. juli eller med forutgående blokkering av ID M2.1.0), og rydd til slutt opp i de avviklede sikkerhetsgruppene. SU-er er kumulative: De som er på en støttet CU, trenger ikke installere alle mellomliggende SU-er, men installerer direkte den nyeste.

## Kilder

1.  [Released: July 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-july-2026-exchange-server-security-updates/4534146): Offisiell kunngjøring av juliutgivelsen med støttede versjoner og det kjente problemet med wrapper-meldinger.

2.  [Addressing Exchange Server May 2026 vulnerability CVE-2026-42897 – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/addressing-exchange-server-may-2026-vulnerability-cve-2026-42897/4518498): Opprinnelig sikkerhetsmelding inkludert nødavbøtning og kjente bivirkninger i OWA.

3.  [Released: June 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-june-2026-exchange-server-security-updates/4524491): Juniutgivelsen som leverte den faktiske rettelsen for CVE-2026-42897.

4.  [Exchange Emergency Mitigation Service (Exchange EM Service) – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-emergency-mitigation-service): Hvordan EM Service fungerer, inkludert timevis sammenligning av avbøtninger og gjenoppretting av en regel som er slettet for tidlig.

5.  [Set-ExchangeServer (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-exchangeserver): Parametrene `MitigationsApplied` og `MitigationsBlocked` for å kontrollere avbøtninger og forhindre reaktivering.

6.  [Exchange On-premises Mitigation Tool (EOMT) – Microsoft CSS-Exchange](https://microsoft.github.io/CSS-Exchange/Security/EOMT/): EOMT-skriptet, inkludert rollback-bryteren og CVE-spesifikk JSON-sikkerhetskopi av IIS-utgangstilstanden.

7.  [CVE-2026-42897 Detail – NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-42897): Teknisk beskrivelse og vurdering av sårbarheten i National Vulnerability Database.
