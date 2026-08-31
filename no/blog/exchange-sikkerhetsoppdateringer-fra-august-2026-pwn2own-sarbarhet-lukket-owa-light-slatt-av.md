---
title: "Exchange-sikkerhetsoppdateringer for august 2026: Pwn2Own-sårbarhet lukket, OWA Light slått av"
navTitle: "Exchange SU 08/2026"
description: "August-SU-en lukker sju sårbarheter, inkludert Exchange-utnyttelsen demonstrert på Pwn2Own 2026, og deaktiverer OWA Light permanent. Microsoft forklarer også hvorfor Exchange-SU-er nå kommer månedlig, og hvorfor Exchange SE CU1 fortsatt lar vente på seg."
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
translationSourceHash: 41e10101798a88902017688d719457fce48959ba3acd2b3f1c757867b1b368d7
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T10:00:36.589Z
translationReview: required
url: https://rafaelpfister.ch/no/blog/exchange-sikkerhetsoppdateringer-fra-august-2026-pwn2own-sarbarhet-lukket-owa-light-slatt-av
---

# Exchange-sikkerhetsoppdateringer for august 2026: Pwn2Own-sårbarhet lukket, OWA Light slått av

Microsoft publiserte sikkerhetsoppdateringer (SU-er) for Exchange Server 11. august 2026, for fjerde måned på rad. Oppdateringene lukker sju sårbarheter. Ingen av dem var offentlig kjent på forhånd, ingen blir etter dagens status aktivt utnyttet, og Microsoft vurderer utnyttelse av alle sju som «Exploitation Less Likely». Likevel er dette ingen ordinær patchdag, av tre grunner: Oppdateringen lukker Exchange-sårbarheten som ble demonstrert i hackingkonkurransen Pwn2Own, den **slår av OWA Light permanent etter nesten tjue år**, og Exchange-teamet har i etterkant forklart hvorfor den månedlige rytmen foreløpig blir normalen.

## Hvilke Exchange-versjoner oppdateringen er tilgjengelig for

SU-ene er tilgjengelige for følgende versjoner:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121573, build 15.2.2562.46; som en ordinært tilgjengelig, offentlig oppdatering.
- **Exchange Server 2019 CU15**: KB5121574, build 15.2.1748.49; kun gjennom **Period 2 ESU-programmet**.
- **Exchange Server 2019 CU14**: KB5121575, build 15.2.1544.44; kun gjennom Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5121576, build 15.1.2507.72; kun gjennom Period 2 ESU.

Situasjonen er den samme som i juli: Exchange 2016 og 2019 er out of support. SU-ene fra mai til oktober 2026 får kun de som er registrert i Period 2 ESU-programmet. Alle andre forblir upatchet på et nivå med nå fjorten åpne, delvis høyt vurderte sårbarheter; overgangen til Exchange SE kan ikke lenger utsettes. Exchange Online er allerede beskyttet; i hybridmiljøer må SU-en likevel installeres på alle Exchange-servere, også på rene administrasjonsservere og på maskiner der bare Exchange Management Tools er installert.

Det kjente problemet med *wrapper-meldinger* i delte postbokser i hybridmiljøer vedvarer også med august-SU-en; ifølge Microsoft er løsningen planlagt for en kommende oppdatering. Det finnes i det minste en beroligelse i kommentarfeltet til utgivelseskunngjøringen: De som har satt den dokumenterte SettingOverride som en midlertidig løsning, trenger **ikke** å opprette den på nytt etter installasjon av august-SU-en. Oppdateringen lar overstyringen stå urørt, slik Exchange-teamet bekrefter der.

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

**[CVE-2026-62913](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62913)** har månedens høyeste verdi med CVSS 8.8: en Remote Code Execution som en autentisert angriper med enkle rettigheter kan utløse uten noen brukerinteraksjon. En vilkårlig kompromittert postbokkonto er tilstrekkelig som utgangspunkt; i en tid med phishing og credential stuffing er «autentisert» ingen høy terskel.

**[CVE-2026-62911](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62911)** er månedens eneste sårbarhet som Microsoft vurderer som *Critical* (Elevation of Privilege, CVSS 8.0). Det ligger mer historie bak enn det nøkterne nummeret antyder: På spørsmålet om Exchange-utnyttelsen som Orange Tsai demonstrerte på **Pwn2Own 2026**, nå var rettet, viser Exchange-teamet i kommentarfeltet til utgivelseskunngjøringen nettopp til denne CVE-en. Konkurransefunnet er dermed lukket: enda en grunn til å ikke la august-SU-en ligge, for Pwn2Own-teknikker publiseres vanligvis i detalj etter at sperrefristene er utløpt.

**[CVE-2026-62914](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62914)** (Spoofing, CVSS 7.3) er den direkte årsaken til at OWA Light slås av, mer om det straks.

De øvrige sårbarhetene: CVE-2026-62910 (EoP, 7.2) forutsetter allerede høye rettigheter, CVE-2026-62912 (DoS), CVE-2026-62915 (Security Feature Bypass) og CVE-2026-65813 (EoP) har CVSS 6.5. Detaljene finnes som vanlig i Security Update Guide (filtrer på «Server Software» for Exchange SE eller «ESU» for 2016/2019).

## OWA Light: slutt etter nesten tjue år

### Hva oppdateringen endrer

Når august-SU-en installeres, blir **OWA Light deaktivert permanent**: på hver server der oppdateringen (eller en senere oppdatering) installeres. De som åpner Light-grensesnittet, vil heretter havne i vanlig Outlook on the web. Deaktiveringen er en del av selve oppdateringen og kan ikke angres med en bryter; Microsoft varslet den noen uker tidligere i et eget blogginnlegg.

OWA Light stammer fra Exchange 2007-æraen: et bevisst enkelt webgrensesnitt som reserve for gamle nettlesere og langsomme forbindelser, offisielt deprecated siden august 2024. Begrunnelsen for slutten er drevet av sikkerhet: En separat eldre renderingsbane ved siden av moderne OWA øker kompleksiteten og dermed angrepsflaten; CVE-2026-62914 er et konkret bevis på dette. De som har lest [juli-artikkelen](/blog/exchange-security-updates-juli-2026), husker dessuten: Allerede CVE-2026-42897-mitigeringen fra mai hadde gjort OWA Light ubrukelig som en bieffekt. Grensesnittet var altså allerede på vei ut.

### For dem som ikke kan patche: Slå av OWA Light manuelt

Viktig for alle som (ennå) ikke kan installere august-SU-en, for eksempel fordi ESU-aktiveringen mangler: Microsoft anbefaler uttrykkelig å **deaktivere OWA Light manuelt** i dette tilfellet for å begrense CVE-2026-62914. Dette gjøres via OWA-postbokspolicyen og påloggingssiden:

```powershell
Get-OwaMailboxPolicy | Set-OwaMailboxPolicy -OWALightEnabled $false
Get-OwaVirtualDirectory | Set-OwaVirtualDirectory -LogonPageLightSelectionEnabled $false
```

Den første kommandoen slår av Light-versjonen for alle postbokser i den aktuelle policyen, den andre fjerner valget «Bruk Light-versjonen» fra OWA-påloggingssiden. Endringer i den virtuelle OWA-katalogen trer først pålitelig i kraft etter resirkulering av OWA-appbassenget eller en `iisreset`.

### Hva administratorer bør kontrollere nå

Deaktiveringen er teknisk triviell, men ikke alltid organisatorisk: OWA Light var den stille reserveløsningen for nisjescenarier. Det er nå verdt å kontrollere bokmerker og helpdesk-veiledninger som har `?layout=light` hardkodet, kiosk- og terminalenheter med gamle nettlesere samt interne veiledninger for brukere som har brukt Light-versjonen av tilgjengelighetshensyn. Moderne Outlook on the web fungerer i alle aktuelle nettlesere og har egne tilgjengelighetsfunksjoner; men den som ikke informerer berørte brukergrupper på forhånd, skaper saker.

## Hvorfor det nå kommer en SU hver måned, og hvor Exchange SE CU1 blir av

To dager etter utgivelsen besvarte Exchange-teamet i et bemerkelsesverdig åpent blogginnlegg («Where is Exchange SE CU1 anyway?») spørsmålet mange administratorer stiller. Kortversjonen: Microsoft bruker KI-verktøy på tvers av konsernet for å finne sårbarheter i egne produkter. Teamene, inkludert Exchange, behandler nå de rapporterte funnene: validerer, reproduserer, retter, tester for regresjoner og leverer månedlig. Siden mai 2026 har det dermed kommet en Exchange-SU hver måned, og Microsoft sier uttrykkelig: Dette økte tempoet vil fortsette.

Det lenge ventede **CU1 for Exchange SE** forsinkes nettopp av denne grunnen. Opprinnelig annonsert for første halvår 2026, deretter flyttet til andre halvår, finnes det nå ikke lenger noen måldato. Microsoft vil først publisere CU1 når det er en måned uten en hastende sikkerhetsleveranse imellom; en CU som umiddelbart blir innhentet av en SU, ville påføre mange organisasjoner dobbelt oppdateringsarbeid. Frem til da blir den månedlige sikkerhetslasten fortløpende integrert i den interne CU1-byggen.

I praksis betyr dette to ting. For det første: Å vente på CU1 er ingen strategi, verken for migrering til SE eller for installering av SU-er. For det andre: Et **månedlig vedlikeholdsvindu** for Exchange må fra nå av inn i driftskalenderen, slik det lenge har vært en selvfølge for Windows-servere.

## Installasjon og etterarbeid

Prosessen forblir den velprøvde: Først kartlegger du med [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) hvilke servere som har hvilket CU/SU-nivå og om manuelle trinn gjenstår. Installer deretter SU-en (ved utdatert CU-nivå viser [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) veien), start serveren på nytt og kontroller at alle Exchange-tjenester har startet riktig. Hvis tjenester står som *deaktivert*, ble installasjonen avbrutt; da hjelper den dokumenterte midlertidige løsningen i Microsofts støtteartikkel om File Version-feilen eller [SetupAssist-skriptet](https://aka.ms/ExSetupAssist). Kjør til slutt Health Checker på nytt.

SU-er er kumulative: De som hoppet over juli-SU-en, kan installere august-SU-en direkte. Og for hybridmiljøer gjelder det kjente tillegget: Hvis Auth-sertifikatet byttes etter SU-installasjonen, bør Hybrid Configuration Wizard kjøres på nytt.

Et etterarbeid fra juli er fortsatt aktuelt: De som fortsatt har CVE-2026-42897-mitigeringen (M2.1.0) aktiv, bør fjerne den nå; hvordan dette gjøres riktig, står i [artikkelen om juli-SU-en](/blog/exchange-security-updates-juli-2026).

## Anbefalt fremgangsmåte

Kort oppsummert: Installer august-SU-en snarlig på alle Exchange-servere og maskiner med Management Tools: Pwn2Own-sårbarheten og RCE-en med 8.8 er grunn nok til ikke å vente på neste patchdag. De som ikke kan patche umiddelbart: OWA Light kan deaktiveres manuelt som et umiddelbart tiltak mot CVE-2026-62914. Før OWA Light slås av, bør berørte brukergrupper identifiseres og informeres (gamle bokmerker, kiosknettlesere, tilgjengelighetsarbeidsflyter). Kjør deretter Health Checker, fullfør utestående etterarbeid fra juli og planlegg et månedlig Exchange-vedlikeholdsvindu, for rytmen fortsetter.

## Kilder

1.  [Released: August 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-august-2026-exchange-server-security-updates/4543951): Offisiell utgivelseskunngjøring med støttede versjoner, merknad om OWA Light, Known Issues og FAQ; i kommentarene finnes bekreftelsene på Pwn2Own-rettingen (CVE-2026-62911) og den fortsatt gjeldende Wrapper-SettingOverride.

2.  [Upcoming retirement of OWA Light in Exchange Server – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/upcoming-retirement-of-owa-light-in-exchange-server/4534943): Forhåndsvarselet om deaktiveringen, inkludert Microsofts anbefaling om å deaktivere OWA Light manuelt dersom patchen uteblir.

3.  [Where is Exchange SE CU1 anyway? – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/where-is-exchange-se-cu1-anyway/4546837): Exchange-teamet om KI-støttet sårbarhetssøk, den vedvarende månedlige rytmen for SU-er og CU1-forsinkelsen.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Referanse for build-numrene til august-SU-ene.

5.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Vilkår og varighet (mai til oktober 2026) for ESU-programmet.

6.  [Wrapper messages appear in shared mailbox inbox in hybrid environments – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Det kjente hybridproblemet siden juni, inkludert SettingOverride-løsningen.

7.  [Neue Sicherheitsupdates für Exchange Server (August 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-august-2026/): Tyskspråklig gjennomgang av de sju CVE-ene med CVSS-verdier og builds.

8.  [Set-OwaMailboxPolicy (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-owamailboxpolicy): Parameteren `OWALightEnabled` for manuell deaktivering av Light-versjonen.

9.  [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Kartlegging av CU/SU-nivåer og utestående manuelle trinn før og etter installasjonen.
