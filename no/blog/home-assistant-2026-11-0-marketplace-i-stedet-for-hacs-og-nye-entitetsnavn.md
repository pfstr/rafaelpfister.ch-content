---
title: "Home Assistant 2026.11.0: Marketplace i stedet for HACS og nye entitetsnavn"
navTitle: "Home Assistant 2026.11.0"
description: "Home Assistant 2026.11.0 overtar HACS som en integrert Marketplace, setter sammen entitetsnavn på nytt av enhet og entitet og bringer en rekke bakoverinkompatible endringer. Kontroller før oppdateringen dashboards, maler og tilstedeværelsesregistrering via FRITZ!Box."
date: "2026-11-05"
kategorie: "Home Assistant og IoT"
timeToRead: "6 min lesetid"
themen:
  - smart-home-iot
produkte:
  - "home-assistant"
protokolle:
  - "releases"
slug: "home-assistant-2026-11-0-marketplace-i-stedet-for-hacs-og-nye-entitetsnavn"
translationId: article-6fe587829541b0ff
translationOf: home-assistant-2026-11-0
url: https://rafaelpfister.ch/no/blog/home-assistant-2026-11-0-marketplace-i-stedet-for-hacs-og-nye-entitetsnavn
translationSourceHash: e560e2600ff8f900817c5c323e9f6a0c0bb7be09065e7794e695e6fbe12d1f52
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:25:50.633Z
translationReview: required
---

# Home Assistant 2026.11.0: Marketplace i stedet for HACS og nye entitetsnavn

Home Assistant 2026.11.0 ble utgitt 4. november 2026 og følger etter 2026.10.4. Utgivelsen integrerer den tidligere Home Assistant Community Store (HACS) som Marketplace direkte i Home Assistant, endrer hvordan entitetsnavn dannes og inneholder en lang liste med bakoverinkompatible endringer, deriblant endringer i Device Trackers fra FRITZ!Box Tools og ved innlogging via ikke-lokale adresser. Ifølge utgivelsesblogginnlegget kommer det 38 nye integrasjoner. Release Notes nevner ingen sikkerhetsrettelse med CVE. Utgivelsen retter seg mot alle som drifter Home Assistant, særlig installasjoner med Custom Integrations fra HACS, slik de brukes i artiklene om [Midea PortaSplit](/blog/midea-portasplit-home-assistant) og [digitalSTROM](/blog/digitalstrom-home-assistant).

## Versjonene i oversikt

| Versjon | Dato | Type | Hovedpunkter |
|---|---|---|---|
| 2026.11.0 | 04.11.2026 | Månedlig utgivelse | HACS blir Marketplace, nye entitetsnavn, Device Trackers fra nettverksskannere alltid med enhet, Auth-kontroll ved ikke-lokale adresser, ingen `deps`-mappe lenger, 38 nye integrasjoner |

## Marketplace: HACS blir en del av Home Assistant

Med 2026.11 blir HACS ifølge utgivelsesblogginnlegget en del av Home Assistant og heter nå Marketplace. Administratorer finner den under Innstillinger > Marketplace og installerer integrasjoner, dashboard-kort, temaer og maler der; en GitHub-konto er ikke nødvendig. Etter installasjon av en integrasjon er omstart ikke lenger nødvendig, men oppdateringer krever fortsatt omstart.

Home Assistant migrerer eksisterende HACS-installasjoner ved første oppstart etter oppdateringen. Installerte prosjekter forblir installert og mottar fortsatt oppdateringer, og Custom Repositories og oppdateringsentiteter overføres. Ifølge produsenten kreves ingen handling. HACS-installasjonsprogrammet avsluttes med Home Assistant 2026.11, en separat HACS-installasjon er ikke lenger mulig på denne versjonen, og den tidligere HACS får ingen nye utgivelser. Rapporter feil fremover i Home Assistants Issue Tracker.

Ved første åpning viser Marketplace et varsel om at fellesskapsprosjekter verken testes, kontrolleres eller støttes av Home Assistant-prosjektet. Du må bekrefte varselet før installasjon eller oppdatering. Prosjekter utenfor katalogen kan legges til via «Add from link»; dette krever en GitHub-tilkobling via Device Sign-in, som bare leser offentlig informasjon.

For Midea PortaSplit gjelder dette Custom Integration `Midea AC LAN` og kortet `apexcharts-card`, dersom begge er installert via HACS. Ifølge migreringsopplysningene forblir de installert og mottar fortsatt oppdateringer. Supply-chain-risikoen ved Custom Integrations, som artikkelen om [sikring av PortaSplit](/blog/midea-portasplit-home-assistant-absichern) beskriver, består med Marketplace, slik produsentens risikovarsel viser. Kontroller etter oppdateringen under Innstillinger > Marketplace at alle prosjekter og Custom Repositories som tidligere ble installert via HACS, er tilgjengelige.

## Nye entitetsnavn

Fra 2026.11 settes entitetsnavn sammen av navnet på entiteten og konteksten den befinner seg i, vanligvis enhetsnavnet. Selvvalgte navn migreres etter de samme reglene, og det gamle navnet beholdes som alias. Kort, merker og rader uten eget navn viser nå enhetsnavnet etterfulgt av entitetsnavnet i stedet for Friendly Name. Entity IDs, automasjoner og skript påvirkes ikke ifølge Release Notes.

For maler anbefaler produsenten funksjonene `entity_name()`, `device_name()` og `area_name()`; attributtet `friendly_name` skal senere markeres som foreldet. Kontroller dashboardene etter oppdateringen for endrede etiketter, og tilpass maler som evaluerer `friendly_name`.

## FRITZ!Box Tools og Device Trackers

Trackere fra nettverksskannere, deriblant uttrykkelig FRITZ!Box Tools, UniFi Network, NETGEAR og Nmap Tracker, har nå alltid en enhet. Omdøpte tracker-entiteter får enhetsnavnet først i navnet sitt. Nye trackere fra disse integrasjonene opprettes alltid deaktivert; tidligere var de aktive hvis en annen integrasjon allerede kjente MAC-adressen til enheten. Eksisterende trackere beholder tilstanden sin, enten aktivert eller deaktivert.

For tilstedeværelsesautomasjoner via FRITZ!Box, som beskrevet for automatisk avreise i [digitalSTROM-artikkelen](/blog/digitalstrom-home-assistant), betyr dette: Eksisterende trackere fortsetter å fungere. Trackeren til en nylig lagt til smarttelefon må du derimot først aktivere før den leverer en tilstand i automasjoner.

Changelogen lister dessuten flere rettelser for FRITZ!Box Tools, blant annet for oppretting av WLAN-brytere, filtrering av FRITZ!Box-verter uten IP-adresse fra Device Trackers, en omlastingssløyfe for enheter uten trafikkdata og en mer forståelig feilmelding ved ugyldig innlogging i oppsettsdialogen. Integrasjonen FRITZ!Box Call Monitor rapporterer nå en konfigurasjonsfeil ved manglende tillatelser.

## Midea og digitalSTROM

Forbedringene av Midea i utgivelsesblogginnlegget gjelder den offisielle integrasjonen `Midea`, som har vært en del av Core siden Home Assistant 2026.8. Custom Integration `Midea AC LAN` (domene `midea_ac_lan`) er separat fra denne. Nytt er kontroller og en fremdriftssensor for vaskemaskiner og tørketromler, dørsensorer for kjøleskap, vannvarmerentiteter, en av/på-bryter for luftrensere og en bryter for lydsignalet på avfuktere. For klimaanlegg nevner changelogen en knapp `reset_filter`, som kan brukes til å markere filteret som rengjort. Integrasjonen har nådd kvalitetsnivået Platinum. Ifølge integrasjonssiden foregår styring og statusforespørsler lokalt; skyen trengs bare for token og nøkkel til V3-enheter.

Changelogen for 2026.11.0 inneholder ingen oppføring om digitalSTROM. Integrasjonen som beskrives i [digitalSTROM-artikkelen](/blog/digitalstrom-home-assistant), er en Custom Integration; dersom den er lagt til via HACS som et Custom Repository, gjelder den beskrevne overføringen til Marketplace også for den.

## Flere bakoverinkompatible endringer

Den fullstendige listen finnes i utgivelsesblogginnlegget. For drift er særlig disse punktene relevante:

- **Auth:** Ved tilgang via en ikke-lokal IP-adresse må skjema, vert og port i nettleseren samsvare med adressen Home Assistant ser, ellers mislykkes innloggingen med «Invalid client id». Den tidligere kontrollen aksepterte slike adresser på grunn av en feil så snart de inneholdt en port. Legg i dette tilfellet Reverse Proxy-en din inn under Trusted Proxies i HTTP-integrasjonen, eller bruk et domenenavn.
- **Python-avhengigheter:** Home Assistant installerer ikke lenger pakker i mappen `deps` i konfigurasjonskatalogen. Bare Core-installasjoner utenfor et virtuelt miljø påvirkes; OS-, Supervised- og Container-installasjoner påvirkes ikke. Etter oppdateringen kan mappen `deps` slettes.
- **SMTP:** Et vedlegg som ikke kan leses, fører nå til en feil; tidligere ble meldingen sendt uten vedlegget.
- **Telegram:** Den gamle `notify`-plattformen utgår. Bruk Telegram-botens Notify-entiteter og handlingen `notify.send_message`.
- **Z-Wave:** zwave-js-server 3.11.0 eller nyere (skjema 51), Z-Wave JS App fra 1.9.0 eller Z-Wave JS UI fra v11.24.2 kreves.
- **Uptime Kuma:** Versjoner før 2.0.0 støttes ikke lenger.
- **Philips Hue:** Lyseffekten `"None"` utgår; bruk i stedet `"off"`.
- **Matter og Electra Smart:** `climate.turn_on` fortsetter den sist aktive modusen. For en bestemt modus bruker du `climate.set_hvac_mode`.
- **Polling-intervall:** For Hayward Omnilogic, Hive, Keenetic NDMS2, Kraken, Plaato og Risco ignoreres et eget intervall og standardverdien brukes. Control4 spør alltid hvert femte sekund; for et annet intervall må du deaktivere polling og kalle `homeassistant.update_entity` fra en automasjon.

Integrasjonen Flipr er fjernet fordi selskapet bak den ble avviklet.

## Nye funksjoner

I tillegg til Marketplace nevner utgivelsesblogginnlegget blant annet utelukkelse av enkelte entiteter, enheter, områder, etasjer eller etiketter fra et mål, kopiering og innliming av dashboard-seksjoner, et datokort og et Alert-kort med visuell editor. Energi-dashboardet kan utlede en energisensor fra en effektsensor. For feilsøking finnes et «Triggered by»-filter i aktivitet, Live Logs per integrasjon og sletting av statistikk i et tidsrom. Sikkerhetskopier og medier kan lagres på lokale datalagringsenheter. Matter 1.5-kameraer med WebRTC samt LoRaWAN (ChirpStack, The Things Network, SenseCAP) og Zigbee Green Power støttes nå.

For Alert-kortet heter alternativet `color` nå `alert_color`; bare dashboards du selv har opprettet i YAML, må tilpasses. Den nye integrasjonen Debug console kjører Python-kode i Home Assistant, har full tilgang og må derfor aktiveres i `configuration.yaml`; bare administratorer kan bruke den.

## Anbefalt fremgangsmåte

Målversjonen er 2026.11.0. Les de bakoverinkompatible endringene i utgivelsesblogginnlegget for integrasjonene du bruker, før du oppdaterer. Etter oppdateringen kontrollerer du i Marketplace at alle tidligere HACS-prosjekter er overført, kontrollerer dashboards og maler for de nye entitetsnavnene og aktiverer ved behov nylig opprettede Device Trackers. Ved tilgang via en Reverse Proxy bør du teste innloggingen. Sikkerhetsmerknadene i GitHub-repositoriet til Home Assistant Core inneholder per 5. november 2026 ingen oppføring knyttet til 2026.11.0; utgivelsesblogginnlegget inneholder ingen informasjon om Known Issues.

## Kilder

1.  [Home Assistant: 2026.11: A lot in store!](https://www.home-assistant.io/blog/2026/11/04/release-202611/): Utgivelsesblogginnlegg med Marketplace, entitetsnavn, nye integrasjoner og de bakoverinkompatible endringene.

2.  [Home Assistant: Full changelog for Home Assistant 2026.11](https://www.home-assistant.io/changelogs/core-2026.11): Fullstendig changelog, inkludert oppføringene om FRITZ!Box Tools, FRITZ!Box Call Monitor og Midea.

3.  [GitHub: home-assistant/core Release 2026.11.0](https://github.com/home-assistant/core/releases/tag/2026.11.0): Release-tag med publisering 4. november 2026.

4.  [Home Assistant: Midea](https://www.home-assistant.io/integrations/midea/): Dokumentasjon av den offisielle Midea-integrasjonen, introdusert i 2026.8, lokal styring.

5.  [GitHub: Security Advisories home-assistant/core](https://github.com/home-assistant/core/security): Publiserte sikkerhetsmerknader, per 5. november 2026 uten oppføring om 2026.11.0.

6.  [Midea PortaSplit i Home Assistant: Oppsett og dashboard](/blog/midea-portasplit-home-assistant): Installasjon av Midea AC LAN og apexcharts-card via HACS.

7.  [Sikre Midea PortaSplit i Home Assistant](/blog/midea-portasplit-home-assistant-absichern): Supply-chain-risiko ved Custom Integrations.

8.  [Koble digitalSTROM til Home Assistant](/blog/digitalstrom-home-assistant): Lokal digitalSTROM-integrasjon og tilstedeværelse via FRITZ!Box.
