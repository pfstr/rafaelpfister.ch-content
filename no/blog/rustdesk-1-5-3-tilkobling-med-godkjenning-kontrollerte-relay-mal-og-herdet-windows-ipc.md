---
title: "RustDesk 1.5.3: Tilkobling med godkjenning, kontrollerte relay-mål og herdet Windows-IPC"
navTitle: "RustDesk 1.5.3"
description: "RustDesk 1.5.3 gir tilkoblinger uten passord med godkjenning fra motparten, aksepterer bare relay-forespørsler for tilbudte adresser og autentiserer IPC-pipene i Windows. Serverstrategier kan ikke lenger endre sensitive innstillinger. Oppdatering av alle klienter anbefales."
date: "2026-11-11"
kategorie: "Fjernvedlikehold og support"
timeToRead: "5 min. lesetid"
themen:
  - fernwartung
produkte:
  - "rustdesk"
protokolle:
  - "releases"
  - "haertung"
slug: "rustdesk-1-5-3-tilkobling-med-godkjenning-kontrollerte-relay-mal-og-herdet-windows-ipc"
translationId: article-557eece28ec889f1
translationOf: rustdesk-1-5-3
url: https://rafaelpfister.ch/no/blog/rustdesk-1-5-3-tilkobling-med-godkjenning-kontrollerte-relay-mal-og-herdet-windows-ipc
translationSourceHash: 3d06a4983979b612a9c43bd0a3eacbb0e13d9692a772e95478c351673d707815
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:35:15.385Z
translationReview: automatic
---

# RustDesk 1.5.3: Tilkobling med godkjenning, kontrollerte relay-mål og herdet Windows-IPC

RustDesk publiserte klientversjon 1.5.3 9. november 2026, åtte dager etter 1.5.1. Versjon 1.5.2 er ikke oppført på releasesiden. Utgivelsen gir en ny funksjon (opprettelse av tilkobling med godkjenning fra motparten), en endring i relay-logikken og 22 feilrettinger. Flere av disse gjelder sikkerhetsrelevante prosesser: kontroll av relay-adresser og relay-UUID-er, autentisering av Named Pipes i Windows, låste konfigurasjonsnøkler i serverstrategier og igjen installeroverføringen i Windows. Produsenten oppgir verken en CVE eller en Security Advisory for dette (per 11. november 2026). Utgivelsen er rettet mot alle som drifter RustDesk-klienter, særlig under Windows og med egen server. Forrige status med signerte oppdateringer og Trusted Devices V2 er beskrevet i artikkelen [RustDesk 1.5.1: signerte oppdateringer, Trusted Devices V2 og herding av filoverføring](/blog/rustdesk-1-5-1).

## Versjonene i oversikt

| Versjon | Dato | Type | Hovedpunkter |
|---|---|---|---|
| 1.5.3 | 09.11.2026 | Klientutgivelse | Request approval, relay kun til tilbudte adresser, IPC-autentisering i Windows, låste nøkler i strategier, installerrettinger (F-Secure, Junction), rettinger for macOS, DRM, Wayland, Android og iOS |

Open-source-serveren (`rustdesk-server`) har ikke fått noen ny utgivelse; siste versjon er fortsatt 1.1.16 fra 20. juli 2026.

## Ny funksjon: Request approval

Med PR #16784 kan en tilkobling opprettes uten RustDesk-passord. Motparten må godkjenne den i en dialog. Ifølge PR-en sender klienten i denne modusen ikke noe passord fra noen kilde, heller ikke fra lokal lagring eller adresseboken. Et eksplisitt oppgitt passord ignoreres. Valget vises som undermenyen «Request approval» i ID-rullegardinmenyen og i menyene på peer-kortene, hver gang med tilkoblingstypene som plattformen og motparten støtter (tilkobling, filoverføring og flere).

For skript og snarveier oppgir PR-en kommandolinjeparameteren `--request-approval` og URI-parameteren `request-approval=true`. Modusen beholdes ved ny tilkobling, ved flytting av en økt til et annet vindu og ved flere tilkoblinger fra den eksterne verktøylinjen. Vanlige tilkoblinger beholder sin tidligere passordatferd.

Ifølge PR-en gjelder følgende begrensninger:

- De tilgjengelige autentiseringsmetodene forhandles ikke med motparten. Hvis bare passordinnlogging er tillatt der, eller det gjelder begrensninger på påloggingsskjermen, gjelder disse som før.
- Hvis det allerede finnes et vindu for samme peer og samme tilkoblingstype, aktiverer RustDesk dette uten å endre innloggingsmetoden. Lukk først den eksisterende økten for å opprette en ny tilkobling med godkjenning.
- Ved tunnel-multipleksing kan en allerede godkjent tunnel gjenbrukes uten ny forespørsel.

PR-en inneholder ingen opplysninger om kompatibilitet med eldre versjoner hos motparten.

## Relay kun til tilbudte adresser

PR #16858 lukker et hull i relay-forhandlingen. Tidligere tok den kontrollerte siden relay-adressen og UUID-en fra en videresendt `RequestRelay`-melding uten kontroll og koblet til den angitte adressen. Nå husker den kontrollerte siden relay-adressene som ble tilbudt den, samt relayet den selv ville valgt (maksimalt 32 oppføringer, gyldig i ti minutter fra siste melding). Den forkaster forespørsler med et annet relay før navneoppløsningen. Begge sider avviser dessuten UUID-er som er tomme eller lengre enn 128 byte, eller som inneholder andre tegn enn ASCII-bokstaver, sifre, `-` og `_`. Ifølge PR-en tilsvarer dette kontrollen hbbs allerede utfører.

En avvist forespørsel får ikke noe svar; controllerens relay-forsøk ender derfor i en tidsavbrudd i stedet for en feilmelding. Et relay du fjerner fra konfigurasjonen, aksepteres senest ti minutter etter siste melding ikke lenger.

For operatører av egen server er én begrensning relevant: Hvis hbbs kjører med det som er markert som foreldet, alternativet `--mask`, avvises tilbakefall til et relay etter et mislykket WebRTC-forsøk dersom hbbs har skrevet om relay-adressen for LAN-et. Dette punktet var åpent ved sammenslåingen. Kontroller om hbbs-instansen din bruker `--mask`, og test relay-tilkoblinger etter oppdateringen.

## Andre sikkerhetsrelevante rettinger

- **Windows-IPC (PR #16755):** Ifølge PR-en kunne andre brukere tidligere opprette flere serverinstanser av Named Pipes for hoved- og tjeneste-IPC og dermed fange opp tilkoblinger til RustDesk, lese IPC-data eller forfalske konfigurasjonssvar. Klienter åpner nå pipene uten denne rettigheten og autentiserer serveren før datautvekslingen. Tjenestepiper må eies av SYSTEM. Ifølge PR-en er en tjenestenekt fortsatt mulig dersom en angriper opptar pipe-navnet først, og eldre versjoner med mer åpne tilgangsregler kan ikke opprette tilkobling før oppdateringen.
- **Strategier (PR #16696):** Strategier som serveren distribuerer via heartbeat, kan ikke lenger endre sensitive innstillinger. Blant de låste er `api-server`, `custom-rendezvous-server`, `relay-server`, `key`, `2fa`, `bot`, `enable-trusted-devices`, `trusted-device-expiration-days` og `websocket-port`. Fortsatt konfigurerbare er blant annet `approve-mode`, `verification-method`, `access-mode`, `whitelist`, `id-whitelist` og `allow-auto-update`. Avviste nøkler loggføres uten verdien. Forfatteren beskriver uttrykkelig endringen som begrenset herding: strategisvar er fortsatt ikke autentisert.
- **Windows-installer (PR #16889, #16908, #16871):** Overføringen via en Named Pipe, innført med 1.5.1, er fjernet igjen, fordi den ifølge PR-en forhindret installasjon på systemer med F-Secure DeepGuard og «Stop service» (feil `0x800700E8`). Beskyttelsen av de forberedte filene bygger nå på de restriktive DACL-ene fra 1.5.1 og på egne staging-kataloger, der handle-ene holdes åpne under overføringen. Hvis `C:\Windows\Temp` er omdirigert via en Junction, bruker installasjonsprogrammet i stedet systemets ProgramData-mappe, fremfor å avbryte med en feil.
- **macOS (PR #16729, #16879):** Privacy Mode slås av ved slutten av økten før låsing, og tjenestedaemonen starter direkte i stedet for via `sh`.
- **IP-hviteliste (PR #16692):** Mellomrom rundt oppføringer i IP-hvitelisten ignoreres.

Hvis installasjonen avbrytes under Windows med F-Secure, test 1.5.3 på en berørt enhet før bred utrulling. Ifølge PR #16889 var bekreftelsen på F-Secure-rettingen fra rapportøren fortsatt utestående ved sammenslåingen. Hvis du hittil har distribuert konfigurasjonsverdier som API-, ID- eller relay-server via strategier, må du fremover angi disse på en annen måte.

## Feilrettinger etter plattform

- **Windows:** Avinstalleringen fortsetter etter feil ved konfigurasjonsavstemming, tjeneste-IPC forsøkes på nytt før GUI-et avsluttes, Clipboard Owner-markøren skrives først.
- **Linux:** Påloggingsskjermen under greetd med Wayland leveres via DRM, uendrede skjermbilder kodes ikke på nytt under DRM, ikke-tilordnede Wayland-capture-feil vises med sin faktiske melding i stedet for «x11 expected».
- **Android:** Ikke lenger krasj ved utklippstavleinnhold over 1 MB, rammer kopieres før bildet frigis, hebraisk og indonesisk tilordnes korrekt fra gamle locale-koder.
- **iOS:** Startskjermen følger systemets Dark Mode.
- **Andre:** Port forwarding holder KCP-håndtaket åpent og sender en avslutningsårsak, sidepanelene i akseptvinduet forblir på gjeldende skjerm under Windows, lydinndataenheter søkes bare etter blant inndataenheter, minimumsstørrelsen for Sciter-hovedvinduet skaleres ikke lenger dobbelt.

## Kjente problemer

- **Rollback-beskyttelse ved signerte oppdateringer (fra 1.5.1):** Åpen. Release Notes for 1.5.3 nevner ingen endring i dette.
- **Portabel variant under Windows (fra 1.5.1):** Herdingen var ifølge PR #15914 flyttet til en separat gren. Release Notes for 1.5.3 nevner ingen endring i dette.
- **Trusted Devices V2 (fra 1.5.1):** De kjente begrensningene (asynkron tilbakekalling, utløp etter vertsens systemklokke) behandles ikke i Release Notes for 1.5.3.
- **Relay-tilbakefall med `--mask` (nytt):** Åpent ifølge PR #16858.
- **Pipe-navn opptatt på forhånd (nytt):** Tjenestenekt er fortsatt mulig ifølge PR #16755.
- **Avbrutt UAC-forespørsel (nytt):** De forberedte filene blir ifølge PR #16889 liggende i den beskyttede katalogen og kan fjernes av en administrator.
- **Strategisvar (nytt):** Fortsatt ikke autentisert. Ved utrullinger med HTTP i stedet for HTTPS kan de fortsatt tillatte alternativene manipuleres ifølge PR #16696.

## Anbefalt fremgangsmåte

Målversjonen for alle klienter er 1.5.3. Siden eldre versjoner ifølge PR #16755 ikke kan opprette tilkobling til de herdede IPC-pipene før oppdateringen, bør alle RustDesk-komponenter på en Windows-enhet oppdateres samtidig. Kontroller før utrulling om du distribuerer serveradresser eller nøkler via strategier, om hbbs kjører med `--mask`, og om F-Secure er i bruk på målsystemene. API-serveren bør være tilgjengelig via HTTPS, slik det allerede ble anbefalt for 1.5.1. Open-source-serveren trenger ikke oppdateres for denne utgivelsen. Release Notes inneholder ingen opplysninger om supportperioder (per 11. november 2026).

## Kilder

1.  [GitHub: RustDesk Release 1.5.3](https://github.com/rustdesk/rustdesk/releases/tag/1.5.3): Release Notes med dato, ny funksjon, endring og feilrettinger.

2.  [GitHub: RustDesk Releases](https://github.com/rustdesk/rustdesk/releases): Versjonsliste uten release 1.5.2.

3.  [GitHub: PR #16784, request approval](https://github.com/rustdesk/rustdesk/pull/16784): Forløp, parametere og begrensninger for tilkobling med godkjenning.

4.  [GitHub: PR #16858, relay kun til tilbudte adresser](https://github.com/rustdesk/rustdesk/pull/16858): Allowlist for relay-adresser, UUID-kontroll og begrensning med `--mask`.

5.  [GitHub: PR #16755, Windows IPC server auth](https://github.com/rustdesk/rustdesk/pull/16755): Sikring av Named Pipes, kompatibilitet og gjenstående risiko.

6.  [GitHub: PR #16696, låste nøkler i strategier](https://github.com/rustdesk/rustdesk/pull/16696): låste og fortsatt tillatte alternativer, manglende autentisering av svarene.

7.  [GitHub: PR #16889, fjerning av pipe-overføringen i installasjonsprogrammet](https://github.com/rustdesk/rustdesk/pull/16889): sammenheng med F-Secure DeepGuard og oppførsel ved avbrutt UAC-forespørsel.

8.  [GitHub: PR #16908, beskyttelse av installasjonskatalogen](https://github.com/rustdesk/rustdesk/pull/16908): egne staging-kataloger med åpne handle-er.

9.  [GitHub: PR #16871, installer-staging ved Junction](https://github.com/rustdesk/rustdesk/pull/16871): bruk av ProgramData ved omdirigert Windows Temp.

10.  [GitHub: RustDesk Security Advisories](https://github.com/rustdesk/rustdesk/security/advisories): ingen publiserte advisories (per 11. november 2026).

11.  [GitHub: rustdesk-server Releases](https://github.com/rustdesk/rustdesk-server/releases): siste serverversjon 1.1.16 fra 20. juli 2026.

12.  [RustDesk 1.5.1: signerte oppdateringer, Trusted Devices V2 og herding av filoverføring](/blog/rustdesk-1-5-1): forrige release-status.
