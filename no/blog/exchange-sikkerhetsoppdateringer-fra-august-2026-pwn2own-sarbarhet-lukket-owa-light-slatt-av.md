---
title: "Exchange-sikkerhetsoppdateringer fra august 2026: Pwn2Own-sårbarhet lukket, OWA Light deaktivert"
navTitle: "Exchange SU 08/2026"
description: "August-SU-en lukker sju sårbarheter, inkludert Exchange-utnyttelsen som ble demonstrert på Pwn2Own 2026, og deaktiverer OWA Light permanent. Microsoft forklarer også hvorfor Exchange-SU-er nå kommer månedlig, og hvorfor Exchange SE CU1 fortsatt lar vente på seg."
date: "2026-08-19"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "6 min lesetid"
themen:
  - exchange-updates
produkte:
  - "exchange"
protokolle:
  - "releases"
  - "powershell"
slug: "exchange-sikkerhetsoppdateringer-fra-august-2026-pwn2own-sarbarhet-lukket-owa-light-slatt-av"
translationId: "article-b07bfd4074212673"
draft: false
translationOf: exchange-security-updates-august-2026
translationSourceHash: 4c2345cf2955df229b8713cf288ec21bba3e1bd43aef297ecad12536e9bf459a
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T08:57:11.888Z
translationReview: required
url: https://rafaelpfister.ch/no/blog/exchange-sikkerhetsoppdateringer-fra-august-2026-pwn2own-sarbarhet-lukket-owa-light-slatt-av
---

# Exchange-sikkerhetsoppdateringer fra august 2026: Pwn2Own-sårbarhet lukket, OWA Light deaktivert

Microsoft publiserte sikkerhetsoppdateringer (SU-er) for Exchange Server 11. august 2026, allerede for fjerde måned på rad. Oppdateringene lukker sju sårbarheter. Ingen av dem var offentlig kjent på forhånd, ingen blir aktivt utnyttet etter dagens status, og Microsoft vurderer utnyttelse av alle sju som «Exploitation Less Likely». Likevel er dette ikke en vanlig patchdag, av tre grunner: Oppdateringen lukker Exchange-sårbarheten som ble demonstrert i hackingkonkurransen Pwn2Own, den **deaktiverer OWA Light permanent etter nesten tjue år**, og Exchange-teamet har i etterkant forklart hvorfor den månedlige rytmen foreløpig blir normalen.

## Hvilke Exchange-versjoner oppdateringen er tilgjengelig for

SU-ene er tilgjengelige for følgende versjoner:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121573, build 15.2.2562.46; som en vanlig tilgjengelig, offentlig oppdatering.
- **Exchange Server 2019 CU15**: KB5121574, build 15.2.1748.49; kun via **Period-2-ESU-programmet**.
- **Exchange Server 2019 CU14**: KB5121575, build 15.2.1544.44; kun via Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5121576, build 15.1.2507.72; kun via Period 2 ESU.

Situasjonen er den samme som i juli: Exchange 2016 og 2019 er out of support. SU-ene fra mai til oktober 2026 får bare de som er registrert i Period-2-ESU-programmet. Alle andre forblir upatchet på et nivå med nå fjorten åpne, delvis høyt vurderte sårbarheter; overgang til Exchange SE kan ikke lenger utsettes. Exchange Online er allerede beskyttet; i hybridmiljøer må SU-en likevel installeres på alle Exchange-servere, også rene administrasjonsservere og maskiner som bare har Exchange Management Tools installert.

Det kjente problemet med *wrapper-meldinger* i delte postbokser i hybridmiljøer består også med august-SU-en; ifølge Microsoft er en løsning planlagt i en kommende oppdatering. Det er i det minste beroligende nytt fra kommentarfeltet i utgivelseskunngjøringen: De som har satt den dokumenterte SettingOverride-løsningen som en workaround, trenger **ikke** å opprette den på nytt etter installasjon av august-SU-en. Oppdateringen lar override-innstillingen være urørt, slik Exchange-teamet bekrefter der.

## Oversikt over de sju sårbarhetene

| CVE | Type | CVSS |
| --- | --- | --- |
| CVE-2026-62913 | Remote Code Execution | 8.8 |
| CVE-2026-62911 | Elevation of Privilege | 8.0 |
| CVE-2026-62914 | Spoofing | 7.3 |
| CVE-2026-62910 | Elevation of Privilege | 7.2 |
| CVE-2026-62912 | Denial of Service | 6.5 |
| CVE-2026-62915 | Security Feature Bypass | 6.5 |
| CVE-2026-65813 | Elevation of Privilege | 6.5 |

Tre av dem fortjener en nærmere titt.

**[CVE-2026-62913](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62913)** har månedens høyeste verdi, CVSS 8.8: en Remote Code Execution som en autentisert angriper med enkle rettigheter kan utløse uten noen brukerinteraksjon. En vilkårlig kompromittert postbokkonto er nok som utgangspunkt; i en tid med phishing og credential stuffing er «autentisert» ingen høy terskel.

**[CVE-2026-62911](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62911)** er månedens eneste sårbarhet som Microsoft klassifiserer som *Critical* (Elevation of Privilege, CVSS 8.0). Bak den nøkterne nummereringen ligger det mer historie enn man skulle tro: På spørsmålet om Exchange-utnyttelsen demonstrert av Orange Tsai på **Pwn2Own 2026** nå var rettet, viser Exchange-teamet i kommentarfeltet til utgivelseskunngjøringen nettopp til denne CVE-en. Konkurransefunnet er dermed lukket: enda en grunn til ikke å la august-SU-en ligge, for Pwn2Own-teknikker publiseres vanligvis i detalj etter at sperrefristene er utløpt. Nå har nettopp dette skjedd: En proof-of-concept er offentlig, og BSI rapporterer rundt 85 prosent sårbare On-Premises-servere i Tyskland. Hvordan angrepet fungerer teknisk (MRSProxy uten Channel Binding, NTLM-relé) og hva som ligger bak tallene, står i [den utførlige artikkelen om CVE-2026-62911](/blog/cve-2026-62911-exchange-ntlm-relay).

**[CVE-2026-62914](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62914)** (Spoofing, CVSS 7.3) er den direkte årsaken til at OWA Light deaktiveres, mer om det straks.

De øvrige sårbarhetene: CVE-2026-62910 (EoP, 7.2) forutsetter allerede høye rettigheter, CVE-2026-62912 (DoS), CVE-2026-62915 (Security Feature Bypass) og CVE-2026-65813 (EoP) har CVSS 6.5. Detaljene finnes som vanlig i Security Update Guide (filtrer på «Server Software» for Exchange SE eller «ESU» for 2016/2019).

## OWA Light: etter nesten tjue år er det slutt

### Hva oppdateringen endrer

Når august-SU-en installeres, blir **OWA Light deaktivert permanent**: på hver server der oppdateringen (eller en senere oppdatering) installeres. De som åpner Light-grensesnittet, havner fremover i vanlig Outlook on the web. Deaktiveringen er en del av selve oppdateringen og kan ikke angres med en bryter; Microsoft kunngjorde den noen uker tidligere i et eget blogginnlegg.

OWA Light stammer fra Exchange 2007-æraen: et bevisst enkelt webgrensesnitt som reserve for gamle nettlesere og trege forbindelser, offisielt deprecated siden august 2024. Begrunnelsen for slutten er sikkerhetsdrevet: En separat eldre renderingsbane ved siden av moderne OWA øker kompleksiteten og dermed angrepsflaten; CVE-2026-62914 gir et konkret bevis på dette. De som har lest [juli-artikkelen](/blog/exchange-security-updates-juli-2026), husker dessuten: Allerede CVE-2026-42897-mitigeringen fra mai hadde samtidig gjort OWA Light ubrukelig. Grensesnittet var altså allerede på vei ut.

### For dem som ikke kan patche: Deaktiver OWA Light manuelt

Viktig for alle som (ennå) ikke kan installere august-SU-en, for eksempel fordi ESU-aktiveringen mangler: Microsoft anbefaler uttrykkelig å **deaktivere OWA Light manuelt** i dette tilfellet for å redusere CVE-2026-62914. Dette gjøres via OWA-postboks-policyen og påloggingssiden:

```powershell
Get-OwaMailboxPolicy | Set-OwaMailboxPolicy -OWALightEnabled $false
Get-OwaVirtualDirectory | Set-OwaVirtualDirectory -LogonPageLightSelectionEnabled $false
```

Den første kommandoen deaktiverer Light-versjonen for alle postbokser med den aktuelle policyen, den andre fjerner valget «Bruk Light-versjon» fra OWA-påloggingssiden. Endringer i den virtuelle OWA-katalogen trer først pålitelig i kraft etter resirkulering av OWA-app-poolen eller en `iisreset`.

### Hva administratorer bør kontrollere nå

Deaktiveringen er teknisk triviell, men ikke alltid organisatorisk: OWA Light var den stille reserveløsningen for nisjescenarier. Det er nå verdt å kontrollere bokmerker og helpdesk-veiledninger som har `?layout=light` hardkodet, kiosk- og terminalenheter med gamle nettlesere samt interne veiledninger for brukere som brukte Light-versjonen av hensyn til tilgjengelighet. Moderne Outlook on the web fungerer i alle aktuelle nettlesere og har egne tilgjengelighetsfunksjoner; men de som ikke informerer berørte brukere på forhånd, vil få saker.

## Hvorfor det nå kommer en SU hver måned, og hvor Exchange SE CU1 blir av

To dager etter utgivelsen besvarte Exchange-teamet i et bemerkelsesverdig åpent blogginnlegg («Where is Exchange SE CU1 anyway?») spørsmålet mange administratorer stiller. Kortversjonen: Microsoft bruker KI-verktøy på tvers av konsernet for å finne sårbarheter i egne produkter. Teamene, inkludert Exchange, behandler nå de rapporterte funnene: validerer, reproduserer, retter, tester for regresjoner og leverer månedlig. Siden mai 2026 har det dermed kommet en Exchange-SU hver måned, og Microsoft sier uttrykkelig: Dette høyere tempoet vil fortsette.

Det lenge ventede **CU1 for Exchange SE** forsinkes nettopp av denne grunnen. Opprinnelig annonsert for første halvår 2026, deretter utsatt til andre halvår, finnes det nå ikke lenger noen måldato. Microsoft vil først publisere CU1 når det er en måned uten en presserende sikkerhetsleveranse imellom; en CU som umiddelbart blir innhentet av en SU, ville påføre mange organisasjoner dobbelt oppdateringsarbeid. Frem til da blir den månedlige sikkerhetspakken fortløpende inkludert i den interne CU1-byggingen.

I praksis betyr dette to ting. For det første: Å vente på CU1 er ingen strategi, verken for migrering til SE eller for installasjon av SU-er. For det andre: Et **månedlig vedlikeholdsvindu** for Exchange hører fra nå av fast hjemme i driftskalenderen, slik det lenge har vært en selvfølge for Windows-servere.

## Installasjon og oppfølging

Fremgangsmåten er den velprøvde: Start med å bruke [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) for å kartlegge hvilke servere som har hvilken CU/SU-status, og om det gjenstår manuelle trinn. Installer deretter SU-en (ved utdatert CU-status viser [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) veien), start serveren på nytt og kontroller at alle Exchange-tjenester har startet korrekt. Hvis tjenester står som *deaktivert*, ble installasjonen avbrutt; da hjelper den dokumenterte workarouden i Microsofts støtteartikkel om File-Version-feilen, eller [SetupAssist-skriptet](https://aka.ms/ExSetupAssist). Kjør til slutt Health Checker på nytt.

SU-er er kumulative: De som hoppet over juli-SU-en, kan installere august-SU-en direkte. Og for hybridmiljøer gjelder det kjente tillegget: Hvis Auth-sertifikatet byttes etter SU-installasjonen, bør Hybrid Configuration Wizard kjøres på nytt.

En oppfølging fra juli er fortsatt aktuell: De som fortsatt har CVE-2026-42897-mitigeringen (M2.1.0) aktiv, bør fjerne den nå; hvordan dette gjøres på en korrekt måte, står i [artikkelen om juli-SU-en](/blog/exchange-security-updates-juli-2026).

## Anbefalt fremgangsmåte

Kort oppsummert: Installer august-SU-en raskt på alle Exchange-servere og maskiner med Management Tools: Pwn2Own-sårbarheten og 8.8-RCE-en er grunn nok til ikke å vente på neste patchdag. De som ikke kan patche umiddelbart: OWA Light kan deaktiveres manuelt som et straks-tiltak mot CVE-2026-62914. Før OWA Light deaktiveres, identifiser og informer berørte brukergrupper (gamle bokmerker, kiosknettlesere, arbeidsflyter for tilgjengelighet). Kjør deretter Health Checker, utfør gjenstående oppfølging fra juli, og planlegg et månedlig Exchange-vedlikeholdsvindu, for rytmen fortsetter.

## Kilder

1.  [Released: August 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-august-2026-exchange-server-security-updates/4543951): Offisiell utgivelseskunngjøring med støttede versjoner, merknad om OWA Light, Known Issues og FAQ; i kommentarene finnes bekreftelsene på Pwn2Own-rettingen (CVE-2026-62911) og den fortsatt gjeldende Wrapper-SettingOverride.

2.  [Upcoming retirement of OWA Light in Exchange Server – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/upcoming-retirement-of-owa-light-in-exchange-server/4534943): Forhåndsvarslingen om deaktiveringen, inkludert Microsofts anbefaling om å deaktivere OWA Light manuelt dersom patchen uteblir.

3.  [Where is Exchange SE CU1 anyway? – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/where-is-exchange-se-cu1-anyway/4546837): Exchange-teamet om KI-støttet søk etter sårbarheter, den vedvarende månedlige rytmen for SU-er og CU1-forsinkelsen.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Referanse for build-numrene til august-SU-ene.

5.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Vilkår og varighet (mai til oktober 2026) for ESU-programmet.

6.  [Wrapper messages appear in shared mailbox inbox in hybrid environments – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Hybridproblemet kjent siden juni, inkludert SettingOverride-workaround.

7.  [Neue Sicherheitsupdates für Exchange Server (August 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-august-2026/): Tyskspråklig gjennomgang av de sju CVE-ene med CVSS-verdier og builds.

8.  [Set-OwaMailboxPolicy (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-owamailboxpolicy): Parameteren `OWALightEnabled` for manuell deaktivering av Light-versjonen.

9.  [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Kartlegging av CU/SU-status og åpne manuelle trinn før og etter installasjonen.
