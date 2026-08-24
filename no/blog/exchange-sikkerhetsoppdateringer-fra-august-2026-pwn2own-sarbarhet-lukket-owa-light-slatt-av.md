---
title: "Exchange-sikkerhetsoppdateringer fra august 2026: Pwn2Own-sårbarhet lukket, OWA Light slått av"
navTitle: "Exchange SU 08/2026"
description: "August-SU-en lukker sju sårbarheter, inkludert Exchange-utnyttelsen som ble demonstrert på Pwn2Own 2026, og deaktiverer OWA Light permanent. Microsoft forklarer også hvorfor Exchange-SU-er nå kommer månedlig, og hvorfor Exchange SE CU1 fortsatt lar vente på seg."
date: "2026-08-19"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "6 min. lesetid"
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
url: https://rafaelpfister.ch/no/blog/exchange-sikkerhetsoppdateringer-fra-august-2026-pwn2own-sarbarhet-lukket-owa-light-slatt-av
translationSourceHash: a41c24b533c3b19bf6226ac5d16e7b9668d83d13b53588da7109f5567e79db51
translationModel: gpt-5.6-terra
translatedAt: 2026-08-20T04:05:38.925Z
translationReview: required
---

# Exchange-sikkerhetsoppdateringer fra august 2026: Pwn2Own-sårbarhet lukket, OWA Light slått av

Microsoft publiserte sikkerhetsoppdateringer (SU-er) for Exchange Server 11. august 2026 — for fjerde måned på rad. Oppdateringene lukker sju sårbarheter. Ingen av dem var offentlig kjent på forhånd, ingen utnyttes aktivt etter dagens kjennskap, og Microsoft vurderer utnyttelse av alle sju som «Exploitation Less Likely». Likevel er dette ikke en rutinemessig patchdag, av tre grunner: Oppdateringen lukker Exchange-sårbarheten som ble demonstrert i hackingkonkurransen Pwn2Own, den **slår av OWA Light permanent etter nesten tjue år**, og Exchange-teamet har i etterkant forklart hvorfor den månedlige rytmen foreløpig blir normalen.

## Hvilke Exchange-versjoner oppdateringen er tilgjengelig for

SU-ene er tilgjengelige for følgende versjoner:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121573, build 15.2.2562.46 — som en ordinært tilgjengelig, offentlig oppdatering.
- **Exchange Server 2019 CU15**: KB5121574, build 15.2.1748.49 — kun via **Period 2 ESU-programmet**.
- **Exchange Server 2019 CU14**: KB5121575, build 15.2.1544.44 — kun via Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5121576, build 15.1.2507.72 — kun via Period 2 ESU.

Situasjonen er den samme som i juli: Exchange 2016 og 2019 har ikke lenger støtte. SU-ene fra mai til oktober 2026 får bare de som er registrert i Period 2 ESU-programmet. Alle andre forblir upatchet på et nivå med nå fjorten åpne, delvis høyt vurderte sårbarheter — overgangen til Exchange SE kan ikke lenger utsettes. Exchange Online er allerede beskyttet; i hybridmiljøer må SU-en likevel installeres på alle Exchange-servere, også rene administrasjonsservere og maskiner der bare Exchange Management Tools er installert.

Det kjente problemet med *wrapper-meldinger* i delte postbokser i hybridmiljøer består også med august-SU-en; ifølge Microsoft er løsningen planlagt for en kommende oppdatering. Det finnes i det minste en god nyhet i kommentarfeltet til utgivelseskunngjøringen: De som har angitt den dokumenterte SettingOverride-en som en midlertidig løsning, trenger **ikke** å opprette den på nytt etter installasjon av august-SU-en — oppdateringen lar overstyringen være urørt, slik Exchange-teamet bekrefter der.

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

**[CVE-2026-62913](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62913)** har månedens høyeste verdi med CVSS 8.8: en Remote Code Execution som en autentisert angriper med enkle rettigheter kan utløse uten noen brukerinteraksjon. En hvilken som helst kompromittert postbokkonto er tilstrekkelig som utgangspunkt — i en tid med phishing og credential stuffing er «autentisert» ingen høy terskel.

**[CVE-2026-62911](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62911)** er månedens eneste sårbarhet som Microsoft vurderer som *Critical* (Elevation of Privilege, CVSS 8.0). Bak dette ligger det mer historie enn det nøkterne nummeret avslører: På spørsmålet om Exchange-utnyttelsen som Orange Tsai demonstrerte på **Pwn2Own 2026**, nå var rettet, viser Exchange-teamet i kommentarfeltet til utgivelseskunngjøringen nettopp til denne CVE-en. Konkurransefunnet er dermed lukket — enda en grunn til ikke å la august-SU-en ligge, ettersom Pwn2Own-teknikker vanligvis publiseres i detalj etter at sperrefristene har utløpt.

**[CVE-2026-62914](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62914)** (Spoofing, CVSS 7.3) er den direkte årsaken til at OWA Light slås av — mer om dette straks.

De øvrige sårbarhetene: CVE-2026-62910 (EoP, 7.2) krever allerede høye rettigheter, CVE-2026-62912 (DoS), CVE-2026-62915 (Security Feature Bypass) og CVE-2026-65813 (EoP) har CVSS 6.5. Detaljene finnes som vanlig i Security Update Guide (filtrer på «Server Software» for Exchange SE eller «ESU» for 2016/2019).

## OWA Light: etter nesten tjue år er det slutt

### Hva oppdateringen endrer

Ved installasjon av august-SU-en blir **OWA Light deaktivert permanent** — på alle servere der oppdateringen (eller en senere oppdatering) installeres. De som åpner Light-grensesnittet, kommer fremover til vanlig Outlook on the web. Deaktiveringen er en del av selve oppdateringen og kan ikke angres med en bryter; Microsoft varslet den noen uker tidligere i et eget blogginnlegg.

OWA Light stammer fra Exchange 2007-æraen: et bevisst enkelt webgrensesnitt som reserve for gamle nettlesere og langsomme forbindelser, og offisielt deprecated siden august 2024. Begrunnelsen for slutten er drevet av sikkerhet: En separat eldre renderingsbane ved siden av moderne OWA øker kompleksiteten og dermed angrepsflaten — CVE-2026-62914 gir et konkret bevis på dette. De som har lest [juli-artikkelen](/blog/exchange-security-updates-juli-2026), husker dessuten: Allerede CVE-2026-42897-mitigeringen fra mai hadde samtidig gjort OWA Light ubrukelig. Grensesnittet var altså allerede på overtid.

### De som ikke kan patche: Slå av OWA Light manuelt

Viktig for alle som (ennå) ikke kan installere august-SU-en — for eksempel fordi ESU-aktiveringen mangler: Microsoft anbefaler uttrykkelig å **deaktivere OWA Light manuelt** i dette tilfellet, for å redusere CVE-2026-62914. Dette gjøres via OWA-postboksregelen og påloggingssiden:

```powershell
Get-OwaMailboxPolicy | Set-OwaMailboxPolicy -OWALightEnabled $false
Get-OwaVirtualDirectory | Set-OwaVirtualDirectory -LogonPageLightSelectionEnabled $false
```

Den første kommandoen slår av Light-versjonen for alle postbokser i den aktuelle policyen, den andre fjerner valget «Bruk Light-versjonen» fra OWA-påloggingssiden. Endringer i den virtuelle OWA-katalogen trer først pålitelig i kraft etter en resirkulering av OWA-appbassenget eller en `iisreset`.

### Hva administratorer bør kontrollere nå

Deaktiveringen er teknisk triviell, men ikke alltid organisatorisk: OWA Light var den stille redningslinen for nisjescenarier. Nå bør man kontrollere bokmerker og helpdesk-veiledninger som har `?layout=light` hardkodet, kiosk- og terminalenheter med gamle nettlesere samt interne veiledninger for brukere som har brukt Light-versjonen av tilgjengelighetshensyn. Moderne Outlook on the web kjører i alle aktuelle nettlesere og har egne tilgjengelighetsfunksjoner — men de som ikke informerer berørte brukere på forhånd, skaper supporthenvendelser.

## Hvorfor det nå kommer en SU hver måned — og hvor Exchange SE CU1 blir av

To dager etter utgivelsen besvarte Exchange-teamet spørsmålet mange administratorer stiller, i et bemerkelsesverdig åpent blogginnlegg («Where is Exchange SE CU1 anyway?»). Kortversjonen: Microsoft bruker KI-verktøy på tvers av konsernet for å finne sårbarheter i egne produkter. Teamene — inkludert Exchange — behandler nå funnene som rapporteres: validerer, reproduserer, retter, tester for regresjoner og leverer månedlig. Siden mai 2026 har det dermed kommet en Exchange-SU hver måned, og Microsoft sier uttrykkelig at dette økte tempoet vil fortsette.

Det lenge ventede **CU1 for Exchange SE** forsinkes nettopp av denne grunn. Opprinnelig annonsert for første halvår 2026, deretter flyttet til andre halvår, finnes det nå ikke lenger noen måldato. Microsoft vil først publisere CU1 når det er en måned uten et presserende sikkerhetsbidrag imellom — en CU som umiddelbart blir innhentet av en SU, ville påføre mange organisasjoner dobbelt oppdateringsarbeid. Frem til da integreres den månedlige sikkerhetslasten fortløpende i den interne CU1-byggingen.

I praksis betyr dette to ting. For det første: Å vente på CU1 er ingen strategi — verken for migreringen til SE eller for installasjon av SU-er. For det andre: Et **månedlig vedlikeholdsvindu** for Exchange hører fra nå av fast hjemme i driftskalenderen, slik det lenge har vært en selvfølge for Windows-servere.

## Installasjon og oppfølging

Fremgangsmåten er fortsatt den velprøvde: Først inventariseres det med [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) hvilke servere som har hvilken CU/SU-versjon, og om manuelle trinn gjenstår. Installer deretter SU-en (ved utdatert CU-versjon viser [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) veien), start serveren på nytt og kontroller at alle Exchange-tjenester har startet korrekt. Hvis tjenester står som *deaktivert*, ble installasjonen avbrutt — da hjelper den dokumenterte midlertidige løsningen i Microsofts støtteartikkel om File-Version-feilen, eller [SetupAssist-skriptet](https://aka.ms/ExSetupAssist). Avslutt med å kjøre Health Checker på nytt.

SU-er er kumulative: De som hoppet over juli-SU-en, installerer august-SU-en direkte. Og for hybridmiljøer gjelder det kjente tillegget: Hvis Auth-sertifikatet byttes etter SU-installasjonen, bør Hybrid Configuration Wizard kjøres på nytt.

Et etterarbeid fra juli er fortsatt aktuelt: De som fortsatt har CVE-2026-42897-mitigeringen (M2.1.0) aktiv, bør fjerne den nå — hvordan dette gjøres riktig, står i [artikkelen om juli-SU-en](/blog/exchange-security-updates-juli-2026).

## Anbefalt fremgangsmåte

Kort oppsummert: Installer august-SU-en raskt på alle Exchange-servere og maskiner med Management Tools — Pwn2Own-sårbarheten og RCE-en med 8.8 er grunn nok til ikke å vente på neste patchdag. De som ikke kan patche umiddelbart, deaktiverer OWA Light manuelt som et umiddelbart tiltak mot CVE-2026-62914. Identifiser og informer berørte brukergrupper før OWA Light slås av (gamle bokmerker, kiosknettlesere, tilgjengelighetsarbeidsflyter). Kjør deretter Health Checker, utfør gjenstående etterarbeid fra juli — og planlegg et månedlig Exchange-vedlikeholdsvindu, for rytmen fortsetter.

## Kilder

1.  [Released: August 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-august-2026-exchange-server-security-updates/4543951): Offisiell utgivelseskunngjøring med støttede versjoner, merknad om OWA Light, kjente problemer og FAQ; i kommentarene finnes bekreftelsene på Pwn2Own-fiksen (CVE-2026-62911) og den fortsatt gjeldende Wrapper-SettingOverride-en.

2.  [Upcoming retirement of OWA Light in Exchange Server – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/upcoming-retirement-of-owa-light-in-exchange-server/4534943): Forhåndsvarselet om deaktiveringen, inkludert Microsofts anbefaling om å deaktivere OWA Light manuelt dersom oppdateringen uteblir.

3.  [Where is Exchange SE CU1 anyway? – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/where-is-exchange-se-cu1-anyway/4546837): Exchange-teamet om KI-støttet søk etter sårbarheter, den vedvarende månedlige rytmen for SU-er og CU1-forsinkelsen.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Referanse for build-numrene til august-SU-ene.

5.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Vilkår og varighet (mai til oktober 2026) for ESU-programmet.

6.  [Wrapper messages appear in shared mailbox inbox in hybrid environments – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Det kjente hybridproblemet siden juni, inkludert SettingOverride-midlertidig løsning.

7.  [Neue Sicherheitsupdates für Exchange Server (August 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-august-2026/): Tyskspråklig oversikt over de sju CVE-ene med CVSS-verdier og builds.

8.  [Set-OwaMailboxPolicy (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-owamailboxpolicy): Parameteren `OWALightEnabled` for manuell deaktivering av Light-versjonen.

9.  [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventarisering av CU/SU-versjoner og åpne manuelle trinn før og etter installasjonen.
