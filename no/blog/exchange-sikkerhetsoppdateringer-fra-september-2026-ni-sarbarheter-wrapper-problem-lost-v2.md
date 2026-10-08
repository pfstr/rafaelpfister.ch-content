---
title: "Exchange-sikkerhetsoppdateringer fra september 2026: ni sårbarheter, wrapper-problem løst, v2 lansert i etterkant"
navTitle: "Exchange SU 09/2026"
description: "September-SU-en lukker ni sårbarheter i Exchange SE og 2019 (åtte i Exchange 2016), inkludert en spoofing-sårbarhet med CVSS 9.3, og løser wrapper-problemet i hybridmiljøer. Den 2. oktober kom v2 med en ytterligere CVE; i tillegg finnes det tre kjente problemer med midlertidige løsninger og en SettingOverride som nå bør fjernes."
date: "2026-10-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "7 min. lesetid"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
  - "powershell"
slug: "exchange-sikkerhetsoppdateringer-fra-september-2026-ni-sarbarheter-wrapper-problem-lost-v2"
translationId: article-53db0c02bc33f9bd
translationOf: exchange-security-updates-september-2026
url: https://rafaelpfister.ch/no/blog/exchange-sikkerhetsoppdateringer-fra-september-2026-ni-sarbarheter-wrapper-problem-lost-v2
translationSourceHash: d0738f61713a26973457a9e536720b9787af35b22867c648dbe113d2aeb3f4ea
translationModel: gpt-5.6-terra
translatedAt: 2026-10-08T10:47:06.828Z
translationReview: automatic
---

# Exchange-sikkerhetsoppdateringer fra september 2026: ni sårbarheter, wrapper-problem løst, v2 lansert i etterkant

Microsoft publiserte 8. september 2026 sikkerhetsoppdateringer (SU-er) for Exchange Server. De lukker ni sårbarheter i Exchange SE og Exchange 2019, og åtte i Exchange 2016. Ingen av dem var offentlig kjent på forhånd, ingen utnyttes aktivt ifølge Security Update Guide, og Microsoft klassifiserer alle som *Important* med «Exploitation Less Likely». Den høyeste CVSS-verdien på 9.3 ligger imidlertid betydelig over forrige måneds verdi. Denne måneden skiller seg ut av tre grunner: SU-en løser problemet med *wrapper-meldinger* i delte postbokser, som har vært åpent siden juni, den medfører tre nye eller vedvarende Known Issues, og 2. oktober lanserte Microsoft en **versjon 2** som lukker en ekstra sårbarhet.

## Hvilke Exchange-versjoner oppdateringen er tilgjengelig for

SU-ene fra 8. september 2026 er tilgjengelige for følgende versjoner:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121608, build 15.2.2562.49; offentlig tilgjengelig.
- **Exchange Server 2019 CU15**: KB5121609, build 15.2.1748.51; bare via **Period-2-ESU-programmet**.
- **Exchange Server 2019 CU14**: KB5121610, build 15.2.1544.46; bare via Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5121611, build 15.1.2507.73; bare via Period 2 ESU.

Exchange 2016 og 2019 støttes ikke lenger. Ifølge Microsoft mottar bare organisasjoner som er registrert i Period-2-ESU-programmet, SU-ene fra mai til oktober 2026. Ifølge KB-artiklene gjelder denne berettigelsen til oktober 2026. Det kommer også press fra Exchange Online: Siden den andre uken i september har Exchange Online begrenset og blokkert hybrid-e-postflyt fra servere med et nivå eldre enn oktober 2025, se detaljer i [artikkelen om transporthåndheving](/blog/exchange-online-transport-enforcement-hybrid-server). Exchange Online er allerede beskyttet ifølge kunngjøringen; i hybridmiljøer trenger likevel hver Exchange-server SU-en, i tillegg til maskiner med Exchange Management Tools.

Du kan kontrollere din egen versjon mot oversikten over [Exchange-buildnumre](/tools/exchange-builds).

## Oversikt over sårbarhetene

| CVE | Type | CVSS |
| --- | --- | --- |
| CVE-2026-69356 | Spoofing (Cross-Site-Scripting) | 9.3 |
| CVE-2026-69641 | Elevation of Privilege | 9.1 |
| CVE-2026-69355 | Remote Code Execution | 8.8 |
| CVE-2026-55007 | Remote Code Execution | 8.1 |
| CVE-2026-69380 | Elevation of Privilege | 8.1 |
| CVE-2026-69378 | Denial of Service | 7.5 |
| CVE-2026-69361 | Spoofing (Server-Side Request Forgery) | 6.5 |
| CVE-2026-69375 | Tampering | 6.5 |
| CVE-2026-69382 | Information Disclosure | 5.9 |

CVE-2026-55007 berører ikke Exchange 2016; Security Update Guide oppgir for denne bare Exchange SE og 2019 CU14/CU15. En detalj i dokumentasjonen: CVE-2026-69380 mangler i CVE-listen i KB-artiklene for de fire september-SU-ene. Security Update Guide oppgir imidlertid nøyaktig de fire september-buildene som rettelse for denne CVE-en (per 7. oktober 2026).

**[CVE-2026-69356](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69356)** har den høyeste verdien med CVSS 9.3. Ifølge Microsoft kan en uautentisert angriper sende en manipulert kalenderinvitasjon med en skadelig møtelenke; når mottakeren åpner møtet og velger lenken for å delta, utløses Cross-Site-Scripting. Det kreves altså brukerinteraksjon, men ingen konto i organisasjonen.

**[CVE-2026-69380](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69380)** (Elevation of Privilege, CVSS 8.1) krever bare en konto med lave rettigheter og en tilordnet postboks. Ifølge FAQ-en i Security Update Guide kan en angriper, gjennom svakheter i kontrollen av forespørsler og identity tokens, utgi seg for å være en annen bruker og overta postboksene til alle Exchange-brukere: lese og sende e-post samt laste ned vedlegg. Én kompromittert brukerkonto er tilstrekkelig som utgangspunkt.

**[CVE-2026-69641](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69641)** (Elevation of Privilege, CVSS 9.1) fører til samme resultat, overtakelse av alle postbokser, men forutsetter medlemskap i en høyt privilegert rollegruppe.

Utgangspunktet er forskjellig for de to Remote Code Execution-sårbarhetene: [CVE-2026-69355](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69355) (CVSS 8.8) krever en autentisert konto med lave rettigheter, mens [CVE-2026-55007](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-55007) (CVSS 8.1) kan utløses uten pålogging med et manipulert Visio-vedlegg, men ifølge Microsoft forutsetter vedvarende lite tilgjengelig arbeidsminne på målsystemet. De øvrige fire sårbarhetene: CVE-2026-69378 (DoS gjennom ukontrollert rekursjon, uten pålogging), CVE-2026-69361 (SSRF, der serveren sender HTTP-forespørsler til interne systemer eller loopback-systemer), CVE-2026-69375 (en autentisert angriper kan erstatte filinnhold) og CVE-2026-69382 (utlevering av påloggingsdata via en svak kryptografisk algoritme, krever en allerede stjålet autentiseringsinformasjonskapsel).

## Versjon 2 fra 2. oktober: CVE-2026-96940 levert i etterkant

2. oktober 2026 publiserte Microsoft «versjon 2» av september-SU-ene. Ifølge kunngjøringen er den eneste forskjellen fra første versjon den ekstra rettelsen for **[CVE-2026-96940](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940). Denne Elevation-of-Privilege-sårbarheten har CVSS 8.8, er verken offentlig kjent eller utnyttet, men Microsoft vurderer den som den eneste av de ti CVE-ene med **«Exploitation More Likely»**. En autentisert angriper kan dermed få tilgang til andre postbokser i samme organisasjon og lese e-post med vedlegg. Exchange Online er allerede korrigert på serversiden.

| Versjon | KB | Build v2 |
| --- | --- | --- |
| Exchange SE RTM | KB5129955 | 15.2.2562.53 |
| Exchange 2019 CU15 | KB5129956 | 15.2.1748.53 |
| Exchange 2019 CU14 | KB5129957 | 15.2.1544.48 |
| Exchange 2016 CU23 | KB5129958 | 15.1.2507.75 |

I praksis betyr dette: Rettelsen for CVE-2026-96940 finnes bare i v2-buildene. Servere som allerede kjører SU-en fra 8. september, trenger v2 i tillegg. De som først oppdaterer nå, kan installere v2 direkte, ettersom SU-er er kumulative. KB-artiklene sier ikke uttrykkelig om servere med den første september-SU-en må installere v2; siden den nye CVE-en bare lukkes med v2, er det den nærliggende tolkningen.

## Wrapper-problem løst: Fjern SettingOverride nå

Det kjente problemet siden juni-SU-en, der *wrapper-meldinger* dukker opp i innboksen til delte postbokser i hybridmiljøer, er løst med september-SU-en i alle fire versjoner. I [august-artikkelen](/blog/exchange-security-updates-august-2026) sto det fortsatt at SettingOverride-en som var dokumentert som midlertidig løsning, kunne beholdes. Etter installasjon av september-SU-en gjelder det motsatte: Microsoft anbefaler i den tilhørende supportartikkelen å kontrollere og fjerne overstyringen.

```powershell
Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
```

<details class="options-details">
<summary>Forklaring av alternativene</summary>

| Kommando | Virkning |
|---|---|
| `Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Kontrollerer om workaround-overstyringen er satt i organisasjonen. |
| `Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Fjerner overstyringen når september-SU-en er installert. |

</details>

Hvis den første kommandoen rapporterer at objektet `DisableBlockSharedAndUserMailboxHeaders` ikke ble funnet, er det ifølge Microsoft ikke nødvendig å gjøre noe mer.

For Exchange SE løser SU-en dessuten en feil i hybrid-ledig/opptatt-forespørsler via Microsoft Graph: Lokale brukere så opptatte tider for Exchange Online-postbokser forskjøvet med sin egen UTC-forskyvning, uten feilmelding.

## Kjente problemer

**Publiserte kalendere (.ics) gir HTTP 500 i kalenderapper.** Problemet har eksistert siden august-SU-en (Exchange SE fra build 15.2.2562.46 samt Exchange 2019 og 2016) og er heller ikke løst i september-SU-en eller v2. Abonnementer på anonymt publiserte kalendere oppdateres ikke lenger; samme URL fungerer i nettleseren. Ifølge Microsoft er årsaken at Exchange gjenkjenner klienter via User-Agent, og kalenderapper uten nettleseridentifikasjon havner i en kodedel som august-SU-en har deaktivert. Som workaround beskriver Microsoft en URL rewrite-regel på nettstedet «Exchange Back End» i IIS, som for .ics-forespørsler under `/owa/calendar/` legger til parameteren `layout=premium`; IIS-modulen URL Rewrite må være installert. De nøyaktige trinnene (via IIS Manager eller direkte i `applicationHost.config`) finnes i [supportartikkelen KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672). Microsoft oppgir ikke noe tidspunkt for rettelsen (per 7. oktober 2026).

**Ledig/opptatt for delegerte postbokser i hybridmiljøer (bare Exchange SE).** Hvis tilgjengelighetsforespørselen utelukkende er konfigurert via Graph API, mislykkes forespørsler til Exchange Online-postbokser via delegert lokal tilgang. Outlook melder «Your server location could not be determined», OWA viser «No information», og EWS-loggene inneholder `(403) Forbidden`. Den dokumenterte workaround-en leder forespørslene tilbake via EWS i stedet for Graph:

```powershell
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=False"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Forklaring av alternativene</summary>

| Alternativ | Virkning |
|---|---|
| `-Identity EnableRouteThroughMSGraphFeature` | Overstyringen som styrer videresendingen av tilgjengelighetsforespørsler via Microsoft Graph. |
| `-Parameters "Enabled=False"` | Slår av Graph-banen; forespørslene går igjen via EWS. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Retter diagnosekallet mot topologitjenesten. |
| `-Component VariantConfiguration -Argument Refresh` | Laster Variant Configuration på nytt, slik at overstyringen virker uten ventetid. |

</details>

I kunngjøringen av v2 oppgir Microsoft dette problemet blant de løste problemene. De som har satt workaround-en, bør etter installasjon av v2 kontrollere i [supportartikkelen KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) om den bør trekkes tilbake; en veiledning for dette var ennå ikke tilgjengelig der per 7. oktober 2026.

**ContentEngine-deadlock på grunn av manglende koreanske WordBreaker-filer (bare Exchange SE).** Med september-SU-en (build 15.2.2562.49 og v2-build 15.2.2562.53) installeres ikke regelfilene for den oppdaterte koreanske WordBreaker-en. Konsekvensene er manglende søkeresultater, forsinket e-postlevering og Outlook- eller MAPI-klienter som henger eller mister forbindelsen. Ifølge kunngjøringen berører dette meldinger på koreansk. Workaround-en i [supportartikkelen KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098) består i å hente de to filene `ko.token.rule.bin` og `ko.complex.rule.bin` fra SQL Server 2025 Express RTM, kontrollere SHA256-hashene, kopiere dem til katalogen `Native` i Exchange-installasjonen og starte tjenesten Search Host Controller på nytt. Microsoft undersøker fortsatt problemet.

## Installasjon og oppfølging

Microsoft anbefaler den kjente fremgangsmåten: inventariser med [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), finn oppdateringsbanen med [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) dersom versjonen er utdatert, installer SU-en, start serveren på nytt og kontroller at alle Exchange-tjenester kjører. Health Checker viser også etterpå om SU-en er riktig installert. Security Update Guide angir at oppdateringene krever omstart.

Etter installasjonen er det tre oppfølgingsoppgaver:

1. Fjern wrapper-SettingOverride-en `DisableBlockSharedAndUserMailboxHeaders` hvis den er satt (se ovenfor).

2. På Exchange SE må du kontrollere om problemene med ledig/opptatt og WordBreaker oppstår, og eventuelt bruke workaround-ene.

3. For publiserte kalendere med eksterne abonnenter må URL rewrite-regelen fra KB5126672 settes opp, dersom dette ikke allerede er gjort siden august-SU-en.

Fra juli gjenstår dessuten kontrollen av om CVE-2026-42897-mitigeringen (M2.1.0) fortsatt er aktiv; hvordan den fjernes, finner du i [artikkelen om juli-SU-en](/blog/exchange-security-updates-juli-2026).

## Anbefalt fremgangsmåte

Installer v2-buildene fra 2. oktober direkte på alle Exchange-servere og maskiner med Management Tools; servere med SU-en fra 8. september trenger v2 i tillegg for CVE-2026-96940. Spoofing-sårbarheten med CVSS 9.3 og overtakelsen av postbokser via en brukerkonto med lave rettigheter (CVE-2026-69380) er grunn nok til ikke å vente på neste patchdag. Fjern deretter wrapper-overstyringen, kontroller de tre Known Issues og kjør Health Checker. ESU-programmet for Exchange 2016 og 2019 avsluttes i oktober 2026; migreringen til Exchange SE kan ikke utsettes lenger.

## Kilder

1.  [Released: September 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-september-2026-exchange-server-security-updates/4554411): Offisiell utgivelseskunngjøring med støttede versjoner, ESU-merknad, Known Issues, løste problemer og installasjonsprosess (åpnet via RSS-feeden til Exchange Team Blog).

2.  [Released: September 2026 V2 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-september-2026-v2-exchange-server-security-updates/ba-p/4561718): Kunngjøring av v2 fra 2. oktober 2026; eneste forskjell er CVE-2026-96940.

3.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM: September 8, 2026 (KB5121608) – Microsoft Support](https://support.microsoft.com/help/5121608): CVE-liste, løste problemer og de tre Known Issues for Exchange SE.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU15: September 8, 2026 (KB5121609) – Microsoft Support](https://support.microsoft.com/help/5121609): KB-artikkel for Exchange 2019 CU15.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU14: September 8, 2026 (KB5121610) – Microsoft Support](https://support.microsoft.com/help/5121610): KB-artikkel for Exchange 2019 CU14.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: September 8, 2026 (KB5121611) – Microsoft Support](https://support.microsoft.com/help/5121611): KB-artikkel for Exchange 2016 CU23, uten CVE-2026-55007.

7.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Type, CVSS, alvorlighetsgrad, vurdering av utnyttelse og FAQ for alle ni CVE-ene fra september og CVE-2026-96940, inkludert berørte builds per CVE.

8.  [Description of version 2 of the security update for Microsoft Exchange Server Subscription Edition RTM October 2, 2026 (KB5129955) – Microsoft Support](https://support.microsoft.com/help/5129955): KB-artikkel om v2 for Exchange SE; v2-artiklene for 2019 og 2016 er KB5129956 til KB5129958.

9.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Buildnumre for september-SU-ene og v2 fra 2. oktober 2026.

10. [Wrapper messages appear in shared mailbox in hybrid environments after installing the June 2026 Security Update – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Rettelse i september-SU-en og veiledning for å fjerne SettingOverride-en.

11. [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Årsak og URL rewrite-workaround.

12. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Symptomer og SettingOverride-workaround for Exchange SE.

13. [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Berørte SE-builds og manuell workaround.

14. [Hybrid free/busy through Microsoft Graph incorrectly shifts busy times by requester timezone – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5125804): Tidssonefeilen i Exchange SE som ble løst med september-SU-en.

15. [Neue Sicherheitsupdates für Exchange Server (September 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-september-2026/): Tyskspråklig gjennomgang av de ni CVE-ene med CVSS-verdier og builds.

16. [Exchange Server: Sicherheitsupdates 8. September 2026 – Borns Tech and Windows World](https://borncity.com/blog/?p=329371): Tyskspråklig oppsummering med henvisninger til den senere erstatningsoppdateringen.
