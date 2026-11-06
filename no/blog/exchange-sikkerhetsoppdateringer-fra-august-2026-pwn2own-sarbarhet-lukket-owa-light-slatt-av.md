---
title: "Exchange-sikkerhetsoppdateringer for august 2026: Pwn2Own-sårbarhet lukket, OWA Light deaktivert"
navTitle: "Exchange SU 08/2026"
description: "August-SU-en lukker sju sårbarheter, deriblant Exchange-utnyttelsen som ble demonstrert på Pwn2Own 2026, og deaktiverer OWA Light permanent. Microsoft forklarer også hvorfor Exchange-SU-er nå kommer månedlig, og hvorfor Exchange SE CU1 fortsatt lar vente på seg."
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
translationSourceHash: fcce9c9e87a2b2cd669e4d2647b79283a9e29727194593a7208e977c56763c97
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T15:53:14.203Z
translationReview: required
url: https://rafaelpfister.ch/no/blog/exchange-sikkerhetsoppdateringer-fra-august-2026-pwn2own-sarbarhet-lukket-owa-light-slatt-av
---

# Exchange-sikkerhetsoppdateringer for august 2026: Pwn2Own-sårbarhet lukket, OWA Light deaktivert

Microsoft publiserte sikkerhetsoppdateringer (SU-er) for Exchange 11. august 2026, for fjerde måned på rad. Oppdateringene lukker sju sårbarheter. Ingen av dem var offentlig kjent på forhånd, ingen blir etter det som er kjent aktivt utnyttet, og Microsoft vurderer utnyttelse av alle sju som «Exploitation Less Likely». Likevel er dette ikke en rutinemessig patchdag, av tre grunner: Oppdateringen lukker Exchange-sårbarheten som ble demonstrert i hackingkonkurransen Pwn2Own, den **deaktiverer OWA Light permanent etter nesten tjue år**, og Exchange-teamet har i etterkant forklart hvorfor den månedlige rytmen foreløpig blir normalen.

## Hvilke Exchange-versjoner oppdateringen er tilgjengelig for

SU-ene er tilgjengelige for følgende versjoner:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121573, build 15.2.2562.46; som en vanlig, offentlig tilgjengelig oppdatering.
- **Exchange Server 2019 CU15**: KB5121574, build 15.2.1748.49; kun via **Period-2-ESU-programmet**.
- **Exchange Server 2019 CU14**: KB5121575, build 15.2.1544.44; kun via Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5121576, build 15.1.2507.72; kun via Period 2 ESU.

Situasjonen er den samme som i juli: Exchange 2016 og 2019 har nådd slutten av støtteperioden. SU-ene fra mai til oktober 2026 er kun tilgjengelige for dem som er registrert i Period-2-ESU-programmet. Alle andre forblir uten oppdateringer på et nivå med nå fjorten åpne, delvis høyt vurderte sårbarheter; migrering til Exchange SE kan ikke lenger utsettes. Exchange Online er allerede beskyttet; i hybridmiljøer må SU-en likevel installeres på alle Exchange-servere, også rene administrasjonsservere og maskiner der kun Exchange Management Tools er installert.

Det kjente problemet med *wrapper-meldinger* i delte postbokser i hybridmiljøer består også med august-SU-en; ifølge Microsoft er en løsning planlagt i en kommende oppdatering. Det finnes i det minste en avklaring i kommentarfeltet til lanseringskunngjøringen: De som har satt den dokumenterte SettingOverride som workaround, trenger **ikke** å opprette den på nytt etter installasjon av august-SU-en. Oppdateringen lar overstyringen være urørt, slik Exchange-teamet bekrefter der.

## De sju sårbarhetene i oversikt

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

**[CVE-2026-62913](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62913)** har månedens høyeste verdi med CVSS 8.8: en Remote Code Execution som en autentisert angriper med enkle rettigheter kan utløse uten noen brukerinteraksjon. En hvilken som helst kompromittert postbokkonto er tilstrekkelig som utgangspunkt; i en tid med phishing og credential stuffing er «autentisert» ingen høy terskel.

**[CVE-2026-62911](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62911)** er månedens eneste sårbarhet som Microsoft vurderer som *Critical* (Elevation of Privilege, CVSS 8.0). Bak dette ligger det mer historie enn det nøkterne nummeret avslører: På spørsmål om Exchange-utnyttelsen som Orange Tsai demonstrerte under **Pwn2Own 2026**, nå var rettet, viser Exchange-teamet i kommentarfeltet til lanseringskunngjøringen nettopp til denne CVE-en. Konkurransefunnet er dermed lukket: enda en grunn til ikke å la august-SU-en ligge, for Pwn2Own-teknikker publiseres vanligvis i detalj etter at sperrefristene er utløpt. Nå har nettopp dette skjedd: Et proof-of-concept er offentlig, og BSI rapporterer om rundt 85 prosent sårbare on-premises-servere i Tyskland. Hvordan angrepet fungerer teknisk (MRSProxy uten Channel Binding, NTLM-relay), og hva som ligger bak tallene, finner du i [den detaljerte artikkelen om CVE-2026-62911](/blog/cve-2026-62911-exchange-ntlm-relay).

**[CVE-2026-62914](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62914)** (Spoofing, CVSS 7.3) er den direkte årsaken til at OWA Light deaktiveres, mer om det straks.

De øvrige sårbarhetene: CVE-2026-62910 (EoP, 7.2) krever allerede høye rettigheter, CVE-2026-62912 (DoS), CVE-2026-62915 (Security Feature Bypass) og CVE-2026-65813 (EoP) har CVSS 6.5. Detaljene finnes som vanlig i Security Update Guide (filtrer på «Server Software» for Exchange SE eller «ESU» for 2016/2019).

## OWA Light: slutt etter nesten tjue år

### Hva oppdateringen endrer

Med installasjon av august-SU-en blir **OWA Light deaktivert permanent**: på hver server der oppdateringen (eller en senere oppdatering) installeres. De som åpner Light-grensesnittet, vil fremover komme til vanlig Outlook on the web. Deaktiveringen er en del av selve oppdateringen og kan ikke angres med en bryter; Microsoft hadde kunngjort dette noen uker tidligere i et eget blogginnlegg.

OWA Light stammer fra Exchange 2007-æraen: et bevisst enkelt webgrensesnitt som reserve for gamle nettlesere og langsomme forbindelser, offisielt deprecated siden august 2024. Begrunnelsen for avviklingen er drevet av sikkerhet: En separat, eldre renderingsbane ved siden av moderne OWA øker kompleksiteten og dermed angrepsflaten; CVE-2026-62914 gir et konkret bevis på dette. De som har lest [juli-artikkelen](/blog/exchange-security-updates-juli-2026), husker dessuten at CVE-2026-42897-mitigationen fra mai allerede hadde gjort OWA Light ubrukelig som bieffekt. Grensesnittet var altså allerede på lånt tid.

### For dem som ikke kan patche: Deaktiver OWA Light manuelt

Viktig for alle som (ennå) ikke kan installere august-SU-en, for eksempel fordi ESU-aktiveringen mangler: Microsoft anbefaler uttrykkelig å **deaktivere OWA Light manuelt** i dette tilfellet for å redusere CVE-2026-62914. Dette gjøres via OWA-postboks-policyen og påloggingssiden:

```powershell
Get-OwaMailboxPolicy | Set-OwaMailboxPolicy -OWALightEnabled $false
Get-OwaVirtualDirectory | Set-OwaVirtualDirectory -LogonPageLightSelectionEnabled $false
```

Den første kommandoen deaktiverer Light-versjonen for alle postbokser i den aktuelle policyen, den andre fjerner valget «Bruk Light-versjon» fra OWA-påloggingssiden. Endringer i den virtuelle OWA-katalogen trer først pålitelig i kraft etter resirkulering av OWA-appbassenget eller en `iisreset`.

### Hva administratorer bør kontrollere nå

Deaktiveringen er teknisk triviell, men ikke alltid organisatorisk: OWA Light var den stille reserveløsningen for nisjescenarier. Det er nå verdt å kontrollere bokmerker og helpdesk-veiledninger som har `?layout=light` fast innebygd, kiosk- og terminalenheter med gamle nettlesere samt interne veiledninger for brukere som har brukt Light-versjonen av tilgjengelighetshensyn. Moderne Outlook on the web kjører i alle oppdaterte nettlesere og har egne tilgjengelighetsfunksjoner; men de som ikke informerer berørte brukere på forhånd, vil skape saker.

## Hvorfor det nå kommer en SU hver måned, og hvor Exchange SE CU1 blir av

To dager etter lanseringen besvarte Exchange-teamet spørsmålet mange administratorer stiller, i et bemerkelsesverdig åpent blogginnlegg («Where is Exchange SE CU1 anyway?»). Kortversjonen: Microsoft bruker KI-verktøy på tvers av konsernet for å finne sårbarheter i egne produkter. Teamene, inkludert Exchange, arbeider nå med de rapporterte funnene: validerer, reproduserer, retter, tester for regresjoner og leverer månedlig. Siden mai 2026 har det dermed kommet en Exchange-SU hver måned, og Microsoft sier uttrykkelig at dette økte tempoet vil fortsette.

Det lenge ventede **CU1 for Exchange SE** blir forsinket nettopp av denne grunn. Opprinnelig kunngjort for første halvår 2026, deretter utsatt til andre halvår, finnes det nå ikke lenger noen måldato. Microsoft vil først publisere CU1 når det finnes en måned uten en presserende sikkerhetsleveranse imellom; en CU som umiddelbart blir innhentet av en SU, ville påføre mange organisasjoner dobbelt oppdateringsarbeid. Frem til da blir den månedlige sikkerhetslasten fortløpende integrert i den interne CU1-byggingen.

I praksis betyr dette to ting. For det første: Å vente på CU1 er ingen strategi, verken for migrering til SE eller installasjon av SU-er. For det andre: Et **månedlig vedlikeholdsvindu** for Exchange må fra nå av være en fast del av driftskalenderen, slik det lenge har vært en selvfølge for Windows-servere.

## Installasjon og oppfølging

Fremgangsmåten er den velprøvde: Først inventariserer du med [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) hvilke servere som har hvilken CU/SU-status, og om manuelle tiltak fortsatt gjenstår. Installer deretter SU-en (ved utdatert CU-nivå viser [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) veien), start serveren på nytt og kontroller at alle Exchange-tjenester har startet uten problemer. Hvis tjenester står som *deaktivert*, ble installasjonen avbrutt; da hjelper den dokumenterte workarouden i Microsofts supportartikkel om File-Version-feilen eller [SetupAssist-skriptet](https://aka.ms/ExSetupAssist). Til slutt kjører du Health Checker på nytt.

SU-er er kumulative: De som hoppet over juli-SU-en, kan installere august-SU-en direkte. Og for hybridmiljøer gjelder det kjente tillegget: Hvis Auth-sertifikatet byttes etter SU-installasjonen, bør Hybrid Configuration Wizard kjøres på nytt.

Et etterarbeid fra juli er fortsatt aktuelt: De som fremdeles har CVE-2026-42897-mitigationen (M2.1.0) aktiv, bør fjerne den nå; hvordan dette gjøres korrekt, finner du i [artikkelen om juli-SU-en](/blog/exchange-security-updates-juli-2026).

## Anbefalt fremgangsmåte

Kort oppsummert: Installer august-SU-en raskt på alle Exchange-servere og maskiner med Management Tools: Pwn2Own-sårbarheten og RCE-en med 8.8 er grunn nok til ikke å vente på neste patchdag. De som ikke kan patche umiddelbart, kan deaktivere OWA Light manuelt som et umiddelbart tiltak mot CVE-2026-62914. Identifiser og informer berørte brukergrupper før OWA Light deaktiveres (gamle bokmerker, kiosknettlesere, arbeidsflyter for tilgjengelighet). Kjør deretter Health Checker, fullfør utestående etterarbeid fra juli og planlegg et månedlig vedlikeholdsvindu for Exchange, for rytmen fortsetter.

## Kilder

1.  [Released: August 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-august-2026-exchange-server-security-updates/4543951): Offisiell lanseringskunngjøring med støttede versjoner, informasjon om OWA Light, Known Issues og FAQ; i kommentarene finnes bekreftelsene på Pwn2Own-rettelsen (CVE-2026-62911) og den fortsatt gjeldende Wrapper-SettingOverride.

2.  [Upcoming retirement of OWA Light in Exchange Server – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/upcoming-retirement-of-owa-light-in-exchange-server/4534943): Forhåndskunngjøringen av deaktiveringen, inkludert Microsofts anbefaling om å deaktivere OWA Light manuelt dersom oppdatering uteblir.

3.  [Where is Exchange SE CU1 anyway? – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/where-is-exchange-se-cu1-anyway/4546837): Exchange-teamet om KI-støttet søk etter sårbarheter, den vedvarende månedlige rytmen for SU-er og CU1-forsinkelsen.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Referanse for build-numrene til august-SU-ene.

5.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Vilkår og varighet (mai til oktober 2026) for ESU-programmet.

6.  [Wrapper messages appear in shared mailbox inbox in hybrid environments – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Det kjente hybridproblemet siden juni, inkludert SettingOverride-workaround.

7.  [Neue Sicherheitsupdates für Exchange Server (August 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-august-2026/): Tyskspråklig gjennomgang av de sju CVE-ene med CVSS-verdier og builds.

8.  [Set-OwaMailboxPolicy (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-owamailboxpolicy): Parameteren `OWALightEnabled` for manuell deaktivering av Light-versjonen.

9.  [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventarisering av CU/SU-status og åpne manuelle tiltak før og etter installasjon.
