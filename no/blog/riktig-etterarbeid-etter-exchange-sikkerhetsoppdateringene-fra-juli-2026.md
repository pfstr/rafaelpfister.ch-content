---
title: "Riktig oppfølging av Exchange-sikkerhetsoppdateringene fra juli 2026"
navTitle: "Exchange SU 07/2026"
description: "Etter installasjonen må to oppryddingsoppgaver utføres: kontrollert fjerning av den gamle CVE-2026-42897-mitigeringen og kontroll av overprivilegerte eldre grupper i Active Directory."
date: "2026-07-14"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "6 min lesetid"
themen:
  - exchange-updates
  - active-directory-entra
slug: "riktig-etterarbeid-etter-exchange-sikkerhetsoppdateringene-fra-juli-2026"
translationOf: "exchange-security-updates-juli-2026"
translationId: article-731b5b840aee096c
translationReview: automatic
translationSourceHash: e5d9295515965d3e7801752cd605f6d2a78cacfc9fb965e0f63d645658b39e9b
translatedAt: 2026-09-05T07:54:05.296Z
url: https://rafaelpfister.ch/no/blog/riktig-etterarbeid-etter-exchange-sikkerhetsoppdateringene-fra-juli-2026
translationModel: gpt-5.6-terra
---

# Riktig oppfølging av Exchange-sikkerhetsoppdateringene fra juli 2026

Med installasjonen av Exchange-sikkerhetsoppdateringene fra 14. juli 2026 er arbeidet ikke fullført. Deretter bør administratorer fjerne to etterlatenskaper: mitigeringen for **CVE-2026-42897**, som ble aktivert i mai, og to historiske Exchange-sikkerhetsgrupper med omfattende rettigheter i Active Directory.

Begge oppgavene er lette å overse. Mitigeringen blir med hensikt værende til den fjernes kontrollert. Gruppene kan på sin side ha overlevd enhver migrering ubemerket i mange år.

## Hvilke Exchange-versjoner oppdateringen er tilgjengelig for

SU-ene er tilgjengelige for følgende versjoner:

- **Exchange Server Subscription Edition (SE) RTM**: som en ordinært tilgjengelig, offentlig oppdatering.
- **Exchange Server 2019 CU14 og CU15**: kun for organisasjoner som er registrert i **Period-2-ESU-programmet**.
- **Exchange Server 2016 CU23**: også kun via Period 2 ESU.

Exchange 2016 og 2019 er ute av støtte. De som ikke er med i Period-2-ESU-programmet (gyldig fra mai til oktober 2026), får ikke lenger disse oppdateringene og bør ikke utsette overgangen til Exchange SE lenger. Exchange Online-miljøer er allerede beskyttet; i hybridoppsett må SU-et likevel installeres på alle Exchange-servere, også på rene administrasjonsservere. Hvilke konkrete CVE-er som håndteres, står som vanlig i Security Update Guide (filteret «Server Software» for Exchange SE og «ESU» for 2016/2019).

Det finnes et kjent problem i den nåværende utgivelsen: I hybridmiljøer kan såkalte *wrapper-meldinger* dukke opp i innboksen til delte postbokser. Se den aktuelle Microsoft-støtteartikkelen for detaljer.

## Fjern CVE-2026-42897-mitigeringen etter installasjonen

### Kort tilbakeblikk

CVE-2026-42897 ble offentliggjort 14. mai 2026: en Cross-Site Scripting-sårbarhet (spoofing) i Outlook Web Access. En angriper sender en spesialpreparert e-post; dersom offeret åpner den i OWA og bestemte interaksjonsbetingelser er oppfylt, kan vilkårlig JavaScript kjøres i nettleserkonteksten. Exchange 2016, 2019 og SE var berørt på *alle* oppdateringsnivåer. Microsoft publiserte samme dag en nødmitigering (ID **M2.1.x**, den konkrete IIS-regelen heter **M2.1.0**) og leverte den faktiske rettelsen med SU-et for juni 2026.

### Hvorfor julioppdateringen *ikke* fjerner mitigeringen automatisk

Dette er punktet som overrasker de fleste: Selv etter installasjon av juli-SU-et forblir en allerede anvendt mitigering aktiv. Årsaken ligger i mekanismen. Mitigeringen er en **Content Security Policy-basert IIS URL Rewrite-regel** som ble lagt inn *utenfor* MSI-installasjonsprogrammet, enten av Emergency Mitigation Service (EM Service) eller av EOMT-skriptet. MSI-oppdateringen erstatter binærfiler, men administrerer ikke disse IIS-reglene som er satt utenfor oppdateringen. Derfor er fjerning et eget, manuelt trinn.

For øvrig: Mitigeringen beskyttet uansett aldri IE-klienter og Edge i IE-modus, fordi Internet Explorer ikke støtter CSP. De som hadde slike klienter i bruk, var aldri sikret av mitigeringen alene. Det er enda et argument for å oppdatere raskt i stedet for å stole på mitigeringen.

### Det vanskelige punktet: EM Service legger inn mitigeringen på nytt

En regel som slettes for tidlig, forblir ikke fjernet permanent. EM Service kjører hver time og sammenligner faktisk tilstand med spesifikasjonene som leveres av Office Config Service (Flighting). Tilordningen «hvilken build trenger hvilken mitigering» ligger på serversiden. Først en endring på serversiden markerer juli 2026-builden som «mitigering ikke lenger nødvendig». Ifølge Microsoft ble denne endringen først rullet helt ut rundt 16. juli 2026. Frem til da legger EM Service bare inn en slettet M2.1.0-regel på nytt ved neste timekjøring.

I praksis betyr det: Enten venter man med manuell fjerning til etter 16. juli, eller så blokkerer man mitigeringen eksplisitt slik at den ikke aktiveres på nytt.

### Slik fjerner du mitigeringen korrekt (EM Service-banen)

Kontroller først hva som faktisk er anvendt:

```powershell
Get-ExchangeServer -Identity <Servername> | Format-List Name,MitigationsApplied,MitigationsBlocked
```

For å forhindre reaktivering settes mitigeringens ID på blokkeringslisten: Oppføringer der ignoreres av EM Service i den timevise kjøringen.

```powershell
Set-ExchangeServer -Identity <Servername> -MitigationsBlocked @("M2.1.0")
```

Fjern deretter selve IIS-regelen. Greit å vite og sjelden dokumentert: EM Service oppretter URL Rewrite-reglene sine med **prefikset «EEMS `<Mitigation-ID>` `<Beschreibung>`»**. Dermed finner du dem entydig i IIS Manager under URL Rewrite (eller via `appcmd`/PowerShell i `applicationHost.config`) uten å måtte gjette hvilken regel som hører til mitigeringen. Etter at endringen på serversiden er rullet ut, kan du oppheve blokkeringen igjen (`-MitigationsBlocked @()`), dersom den kun ble satt som en midlertidig løsning.

### EOMT-banen (isolerte eller air-gapped miljøer)

Dersom mitigeringen ble satt via det nedlastbare **EOMT-skriptet** (https://aka.ms/UnifiedEOMT), reverseres den med rollback-bryteren:

```powershell
.\EOMT.ps1 -RollbackMitigation -CVE "CVE-2026-42897"
```

Også her er det en lite kjent detalj: EOMT lagrer IIS-utgangstilstanden i en **CVE-spesifikk JSON-sikkerhetskopifil** under `%WINDIR%\System32\inetsrv\config\` før hver endring. Rollback leser nettopp denne filen og gjenoppretter de opprinnelige innstillingene. Viktig: En mitigering som er satt med et eldre skript (EOMTv2 osv.), må også fjernes med sin egen rollback-mekanisme: Sikkerhetskopiformatene er ikke kompatible.

### Hvorfor det lønner seg å fjerne den

Mitigeringen er ikke «gratis». Så lenge den er aktiv, følger de kjente bivirkningene med: OWA-funksjonen «Skriv ut kalender» virker ikke, innebygde bilder kan i enkelte tilfeller ikke vises riktig i OWA-leseruten, OWA Light (`/?layout=light`) er defekt (og blir uansett snart avviklet), og publiserte kalendere gir delvis feil 500. Særlig vanskelig for overvåking: Healthset **OWACalendar.Proxy** kan bli *unhealthy* og dermed utløse falske alarmer i overvåkingen. De som har installert SU-et, men lar mitigeringen stå, ender opp med å lete etter feil som ikke finnes. Så snart oppdateringen er installert *og* mitigeringen er fjernet, forsvinner også disse kjente problemene.

Et spesialtilfelle: I blandede miljøer kan servere som ennå ikke er oppdatert, beholde mitigeringen. Men du bør være klar over at Office Online Server-integrasjonen (OOS) i enkelte tilfeller først fungerer korrekt igjen når *alle* Exchange-servere i organisasjonen er på juli-nivået.

## Health Checker: finn urgamle sikkerhetsgrupper

Det andre punktet, uavhengig av SU-utgivelsen: **Exchange Health Checker** (https://aka.ms/ExchangeHealthChecker) kontrollerer nå om to for lengst foreldede sikkerhetsgrupper finnes: **«Exchange Domain Servers»** og **«Exchange Enterprise Servers»**.

### Hvor disse gruppene kommer fra, og hvorfor de utgjør en risiko

Disse to gruppene stammer fra tillatelsesmodellen i Exchange 2000/2003 og har vært foreldet siden Exchange 2007. Med Exchange 2007/2010 kom Split Permissions- og RBAC-modellen, og siden da har de rett og slett ikke vært brukt. Problemet er at de ikke dermed forsvant. I mange kataloger har de ligget uoppdaget i rundt to tiår og delvis fortsatt omfattende ACL-er fra den gamle modellen, altså flere rettigheter enn en moderne Exchange-sikkerhetsgruppe noensinne ville hatt.

Nettopp dette gjør dem til en angrepsvektor. En inaktiv gruppe med stående, brede tillatelser er en klassisk eskaleringskjede: Den som klarer å legge seg selv (eller en kontrollert konto) til i en slik gruppe, arver gruppens rettigheter i katalogen. Siden ingen overvåker gruppen aktivt, blir slik manipulering knapt oppdaget.

### Hvorfor de fleste administratorer ikke kjenner dem

Disse gruppene er en blind flekk av flere grunner: De har vært inaktive i rundt 20 år, eksisterte som regel før dagens team tiltrådte, overlever uten problemer enhver migrering og har hittil aldri blitt rapportert av Health Checker. Særlig alvorlig: De overlever selv *fullstendig* avvikling av lokal Exchange. De som har fjernet den siste Exchange-serveren, rydder vanligvis bort serverobjektene, men overser disse eldre gruppene fullstendig.

### Opprydding

Health Checker vil fremover rapportere gruppene automatisk. Manuelt finner du dem i Active Directory (vanligvis i `Users`-containeren) eller via PowerShell:

```powershell
Get-ADGroup -Filter "Name -eq 'Exchange Domain Servers' -or Name -eq 'Exchange Enterprise Servers'"
```

Fremgangsmåte: Kontroller medlemskap og eventuelle egendefinerte ACL-referanser, forsikre deg om at ingenting i produksjon refererer til dem, og slett deretter gruppene. Siden de har vært foreldet siden 2007, kan de fjernes uten risiko i det overveldende flertallet av miljøer. De som ikke lenger drifter lokal Exchange, bør samtidig planlegge en mer omfattende AD-opprydding i henhold til Microsofts offisielle veiledning.

Hayes Jupe har skrevet en detaljert veiledning om å fjerne gruppene i blogginnlegget [Latest Exchange health check script and deprecated groups](https://www.hayesjupe.com/latest-exchange-health-check-script-and-deprecated-groups/).

## Anbefalt fremgangsmåte

Kort oppsummert er den praktiske rekkefølgen: Inventer først miljøet med Health Checker (den viser manglende CU-er/SU-er, åpne manuelle trinn *og* nå de eldre gruppene). Installer deretter gjeldende CU og juli-SU-et, start serveren på nytt og kontroller at alle Exchange-tjenester har startet korrekt. Kjør deretter Health Checker på nytt, fjern CVE-2026-42897-mitigeringen (etter 16. juli eller med forhåndsblokkering av ID-en M2.1.0) og rydd til slutt opp i de foreldede sikkerhetsgruppene. SU-er er kumulative: De som er på en støttet CU, trenger ikke installere hvert mellomliggende SU, men installerer direkte den nyeste.

## Kilder

1.  [Released: July 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-july-2026-exchange-server-security-updates/4534146): Offisiell kunngjøring av juliutgivelsen med de støttede versjonene og det kjente problemet med wrapper-meldinger.

2.  [Addressing Exchange Server May 2026 vulnerability CVE-2026-42897 – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/addressing-exchange-server-may-2026-vulnerability-cve-2026-42897/4518498): Opprinnelig sikkerhetsmelding, inkludert nødmitigeringen og de kjente bivirkningene i OWA.

3.  [Released: June 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-june-2026-exchange-server-security-updates/4524491): Juniutgivelsen som leverte den faktiske rettelsen for CVE-2026-42897.

4.  [Exchange Emergency Mitigation Service (Exchange EM Service) – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-emergency-mitigation-service): Hvordan EM Service fungerer, inkludert timevis sammenligning av mitigeringer og gjenoppretting av en regel som ble slettet for tidlig.

5.  [Set-ExchangeServer (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-exchangeserver): Parameterne `MitigationsApplied` og `MitigationsBlocked` for å kontrollere mitigeringer og hindre reaktivering.

6.  [Exchange On-premises Mitigation Tool (EOMT) – Microsoft CSS-Exchange](https://microsoft.github.io/CSS-Exchange/Security/EOMT/): EOMT-skriptet, inkludert rollback-bryteren og CVE-spesifikk JSON-sikkerhetskopi av IIS-utgangstilstanden.

7.  [CVE-2026-42897 Detail – NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-42897): Teknisk beskrivelse og vurdering av sårbarheten i National Vulnerability Database.
