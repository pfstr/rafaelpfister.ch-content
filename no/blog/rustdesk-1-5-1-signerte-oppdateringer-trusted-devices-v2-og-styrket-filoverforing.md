---
title: "RustDesk 1.5.1: signerte oppdateringer, Trusted Devices V2 og styrket filoverføring"
navTitle: "RustDesk 1.5.1"
description: "RustDesk 1.5.1 kontrollerer skrivebordsoppdateringer ved hjelp av signerte metadata, introduserer Trusted Devices V2 og styrker filoverføring, Windows-installasjonsprogrammet og automatisk pålogging. I tillegg kommer rundt 35 feilrettinger, særlig for DRM- og Wayland-opptak. Det anbefales å oppdatere klientene, mens serveren forblir uendret."
date: "2026-11-02"
kategorie: "Fjernvedlikehold og support"
timeToRead: "5 min lesetid"
themen:
  - fernwartung
produkte:
  - "rustdesk"
protokolle:
  - "releases"
slug: "rustdesk-1-5-1-signerte-oppdateringer-trusted-devices-v2-og-styrket-filoverforing"
translationId: article-c95ef39dfcc09875
translationOf: rustdesk-1-5-1
url: https://rafaelpfister.ch/no/blog/rustdesk-1-5-1-signerte-oppdateringer-trusted-devices-v2-og-styrket-filoverforing
translationSourceHash: aa77142cfee63f49b15c18d6d1aa69368ec9e6ac75dee970c2907402fc21fc91
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:36:11.439Z
translationReview: automatic
---

# RustDesk 1.5.1: signerte oppdateringer, Trusted Devices V2 og styrket filoverføring

RustDesk lanserte klientversjon 1.5.1 1. november 2026, godt over én måned etter 1.5.0 (30. september 2026). Utgivelsen har fem nye funksjoner, blant annet kontroll av signerte oppdateringsmetadata og en ny versjon av Trusted Devices for tofaktorinnlogging, tre endringer og rundt 35 feilrettinger. Flere av disse styrker sikkerhetsrelevante prosesser: validering av nedlastingsstier, beskyttet overlevering til installasjonsprogrammet i Windows, advarsel ved ukrypterte API-servere og mer begrenset automatisk pålogging. Produsenten oppgir ingen CVE eller Security Advisory for dette (per 2. november 2026). Utgivelsen er rettet mot alle som kjører RustDesk-klienter på Windows, macOS, Linux, Android eller iOS, særlig med egen server. Oppsett og drift av en egen RustDesk-server er beskrevet i artikkelen [RustDesk: sett opp det åpne TeamViewer-alternativet](/blog/rustdesk-teamviewer-alternative).

## Versjonene i oversikt

| Versjon | Dato | Type | Hovedpunkter |
|---|---|---|---|
| 1.5.1 | 01.11.2026 | Klientutgivelse | signerte oppdateringsmetadata, Trusted Devices V2, hurtigtaster, validering av nedlastingssti, styrking av installasjonsprogrammet i Windows, advarsel ved HTTP-API-server, rettinger for DRM, Wayland, X11 og QSV |

Open-source-serveren (`rustdesk-server`) påvirkes ikke av denne utgivelsen. Den siste versjonen er 1.1.16 fra 20. juli 2026.

## Signerte oppdateringsmetadata

Med PR #15914 henter skrivebordsoppdatereren i Windows og macOS informasjonen sin fra signerte utgivelsesmetadata. Hver utgivelse inneholder ressursene `rustdesk-update.json` (versjon, URL, filnavn, størrelse og SHA-256 per artefakt) og `rustdesk-update.json.sig` (Ed25519-signatur). Klienten kontrollerer signaturen før den evaluerer metadataene, og godtar bare nedlastings-URL-er under utgivelsen fra `rustdesk/rustdesk`. Hvis kontrollen mislykkes, avbrytes oppdateringen; det finnes ingen tilbakefall til usignerte data. Nedlastede installasjonsprogrammer lagres i en beskyttet katalog frem til oppstart og hashes på nytt før kjøring med forhøyede rettigheter.

Ifølge PR-en dokumenterer signaturen autentisitet og integritet, men ikke aktualitet: Beskyttelse mot å legge inn eldre, korrekt signerte utgivelser på nytt (rollback) er ikke inkludert. Den portable varianten i Windows pakker fortsatt ut og starter fra en katalog som brukeren kan skrive til; styrkingen for dette er ifølge PR-en flyttet til en separat gren.

## Trusted Devices V2

PR #16548 erstatter den tidligere tillitsrelasjonen basert på maskinvare-ID med tokens som verten utsteder. En ny registrering krever fortsatt det permanente passordet og TOTP; et Trusted-Device-token sparer deretter bare TOTP-inntastingen. Verten lagrer bare SHA-256-hashen av tokenet i kryptert form, mens kontrolløren binder tokenet sitt til vertens kontrollerte offentlige nøkkel. Ved direkte IP-forbindelser tilbys ingen tillitsrelasjon, siden vertens identitet ikke kontrolleres der.

Følgende punkter fra PR-en er relevante for drift:

- **Gyldighet:** kan settes til 1, 7, 30 eller 90 dager eller ubegrenset, med 90 dager som standard. Tillitsrelasjoner kan fornyes eller fjernes enkeltvis i innstillingene, eller slettes samlet.
- **Tilbakekalling:** Endring av passord, 2FA eller Trusted-Device-innstillingen tilbakekaller de berørte tokenene. Verten har maksimalt 128 oppføringer.
- **Blandede versjoner:** Eksisterende, ikke-utløpte tillitsrelasjoner forblir gyldige etter oppdatering av verten. Nye registreringer krever 1.5.1 på begge sider; i blandede økter ber RustDesk om TOTP og tilbyr ikke «Trust this device».
- **Nedgradering:** Ifølge PR-en ikke sikker, ettersom eldre versjoner ikke kan lese de importerte tillitsrelasjonene.
- **Kjente begrensninger:** En tilbakekalling lagres først lokalt og overføres asynkront til rotkonfigurasjonen; en omstart før synkroniseringen kan gjenopprette en eldre tillitsrelasjon. Utløpsdatoen følger vertens systemklokke.

Planlegg derfor oppdateringen slik at verter og kontrollører går over til 1.5.1 innen kort tid, og unngå å gå tilbake til 1.5.0.

## Andre sikkerhetsrelevante rettinger

- **Nedlastingsstier (PR #16601):** Nedlastinger avviser usikre filnavn og stier som fører ut av den valgte målkatalogen eller går via symbolske lenker. Eksterne fillister kontrolleres før nedlasting, og overskriving av eksisterende filer samt gjenopptakelse av avbrutte nedlastinger krever bekreftelse.
- **Windows-installasjonsprogram (PR #16554, #16562):** Skriptene for overlevering til installasjonsprogrammet med forhøyede rettigheter ligger ikke lenger i `System32`, men i en katalog under Windows Temp med begrenset DACL (Administrators og SYSTEM). Tjenesten styres via den innebygde Service Control. Bakgrunnen var blant annet at Bitdefender blokkerte det midlertidige batch-skriptet (Issue #16557).
- **Ukryptert API-server (PR #16616):** Serverinnstillingene viser en advarsel dersom API-serveren er tom eller bruker en ekstern HTTP-adresse. HTTPS og lokale adresser utløser ingen advarsel. Forbindelsen blokkeres ikke.
- **Automatisk pålogging (PR #16520):** Automatisk pålogging skriver nå bare inn operativsystempassordet på innloggings- eller låseskjermen.
- **Privacy Mode (PR #16507):** En avvist Privacy-Mode-forespørsel deaktiverer ikke lenger Privacy Mode for en annen forbindelse.

Hvis du driver en egen server med en `http://`-adresse som API-server, vises den nye advarselen for brukerne dine. Bytt API-serveren til HTTPS før du distribuerer 1.5.1, for å unngå spørsmål.

## Nye funksjoner og endringer

I tillegg til punktene som allerede er nevnt, gir 1.5.1 hurtigtaster (PR #14933), en knapp i adresseboken for å åpne nettkonsollen og direkte filoverføring til SAF-mapper i Android. Endret er kodingen av statisk bildeinnhold etter rulling (skarpere visning) og oppdateringsbudsjettene for MediaCodec. Behandlingen av `ConfigUpdate` er fjernet.

## Feilrettinger etter plattform

- **Linux, DRM-opptak:** riktig størrelse ved roterte skjermer, ingen oppvekking av sovende skjermer ved oppstart, forkasting av dupliserte bilder og bildefrekvens tilpasset koderen.
- **Linux, Wayland:** PipeWire-opptak uten ventetider, riktig tilordning av portalstrømmen til utgangen, tastaturoppsett ved inntasting, og en åpen portalforespørsel blokkerer ikke lenger serveren.
- **Linux, generelt:** riktig aktiv bruker under Flatpak, krasj ved lukking av GTK-vinduet, sudo-forespørsel uten feilaktig «Sorry, try again.» og med støtte for sudo-rs, xrandr-utgangsnavn, RandR-skjermforespørsel under X11.
- **Windows:** lesing av `host=` fra programstier med ikke-ASCII-tegn, eksplisitt navngitt kjørbar fil for Privacy-vinduet, heng i QSV-koderen.
- **macOS:** lenke til innstillingene for Screen Recording, slipp av museknappen.
- **Mobil:** ikke lenger forsinket Mouse-Down etter at den flytende knappen slippes.
- **Nettverk:** HTTP-API-feil 407 via HTTPS-proxyer, IPv6-ID-server uten klammer ved utledning av API-serveren, én feilmelding i stedet for to når begge WebRTC-forbindelsesforsøkene mislykkes.
- **Andre:** slettede skrivere beholdes, replay-buffer i terminalen, tastaturnavigasjon i fillisten, lasting av peers uten angitt app-katalog, klikk på peer-kort i nettklienten.

## Anbefalt fremgangsmåte

Målversjonen for alle klienter er 1.5.1. Oppdater verter og kontrollører så langt som mulig i samme periode, slik at nye Trusted-Device-registreringer fungerer; en nedgradering til 1.5.0 er ifølge PR #16548 ikke sikker. Kontroller før utrullingen om din egen API-server er tilgjengelig via HTTPS. Open-source-serveren trenger ikke å oppdateres for denne utgivelsen. Utgivelsesnotatene inneholder ingen opplysninger om Known Issues eller supportfrister (per 2. november 2026).

## Kilder

1.  [GitHub: RustDesk Release 1.5.1](https://github.com/rustdesk/rustdesk/releases/tag/1.5.1): Release Notes med dato, nye funksjoner, endringer og feilrettinger.

2.  [GitHub: RustDesk Release 1.5.0](https://github.com/rustdesk/rustdesk/releases/tag/1.5.0): Forgjengerversjon fra 30. september 2026.

3.  [GitHub: PR #15914, verify signed update metadata](https://github.com/rustdesk/rustdesk/pull/15914): Prosess for signaturkontroll, manglende rollback-beskyttelse, portabel variant.

4.  [GitHub: PR #16548, Trusted Devices V2](https://github.com/rustdesk/rustdesk/pull/16548): Token-modell, kompatibilitet med eldre versjoner, nedgradering og kjente begrensninger.

5.  [GitHub: PR #16601, Download path validation](https://github.com/rustdesk/rustdesk/pull/16601): Kontroll av filnavn, stier og symbolske lenker ved nedlastinger.

6.  [GitHub: PR #16554, Installer-skript i beskyttede Temp-kataloger](https://github.com/rustdesk/rustdesk/pull/16554): Flytting fra System32, DACL og forbindelse til Bitdefender.

7.  [GitHub: PR #16616, advarsel ved usikker API-server](https://github.com/rustdesk/rustdesk/pull/16616): Betingelser og tekst for den nye advarselen.

8.  [GitHub: RustDesk Security Advisories](https://github.com/rustdesk/rustdesk/security/advisories): ingen publiserte advisories (per 2. november 2026).

9.  [GitHub: rustdesk-server Releases](https://github.com/rustdesk/rustdesk-server/releases): siste serverversjon 1.1.16 fra 20. juli 2026.

10.  [RustDesk: sett opp det åpne TeamViewer-alternativet](/blog/rustdesk-teamviewer-alternative): Oppsett av klient og egen server.
