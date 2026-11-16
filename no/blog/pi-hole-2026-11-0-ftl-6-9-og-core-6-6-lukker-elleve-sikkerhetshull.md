---
title: "Pi-hole 2026.11.0: FTL 6.9 og Core 6.6 lukker elleve sikkerhetshull"
navTitle: "Pi-hole 2026.11.0"
description: "Docker-imaget Pi-hole 2026.11.0 samler FTL v6.9 og Core v6.6 og retter blant annet kodekjøring for administratorer via files.gravity (CVSS 7.2) samt flere symlink-sårbarheter i installasjonsprogrammet. Regex-filtre, webserver.acl og Sessions-API-et oppfører seg delvis annerledes; kontroller disse punktene etter oppdateringen."
date: "2026-11-12"
kategorie: "Nettverk og DNS"
timeToRead: "7 min lesetid"
themen:
  - smart-home-iot
produkte:
  - "pihole"
protokolle:
  - "releases"
  - "dns"
  - "container"
slug: "pi-hole-2026-11-0-ftl-6-9-og-core-6-6-lukker-elleve-sikkerhetshull"
translationId: article-0eb045e8f65bedd0
translationOf: pihole-2026-11-0
url: https://rafaelpfister.ch/no/blog/pi-hole-2026-11-0-ftl-6-9-og-core-6-6-lukker-elleve-sikkerhetshull
translationSourceHash: 44a4448ed57216f76890c2807ea763f7c340fcdc0fffa70b8e8e50816c28f4c4
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T13:43:58.462Z
translationReview: automatic
---

# Pi-hole 2026.11.0: FTL 6.9 og Core 6.6 lukker elleve sikkerhetshull

Pi-hole publiserte Docker-imaget 2026.11.0 den 11. november 2026. Det inneholder Pi-hole FTL v6.9 og Pi-hole Core v6.6, som ble utgitt samme dag; Web Interface forblir ifølge produsenten på v6.6.1. De to komponentutgivelsene retter elleve sikkerhetsråd, inkludert fire med alvorlighetsgrad High (CVSS 7.2 til 7.3). FTL v6.9 oppdaterer dessuten dnsmasq, SQLite og regex-motoren TRE og endrer oppførselen til regex-filtre, `webserver.acl` og deler av API-et. Utgivelsen retter seg mot alle som drifter Pi-hole, både i Docker-containere og som bare-metal-installasjon. Forgjengeren 2026.10.1 fra 31. oktober 2026 ga ifølge Release Notes først og fremst et oppdatert Alpine-basisimage (3.24.3).

## Versjonene i oversikt

| Versjon | Dato | Type | Hovedpunkter |
|---|---|---|---|
| Docker 2026.11.0 | 11.11.2026 | Image-utgivelse | samler FTL v6.9, Core v6.6 og Web v6.6.1; `setpriv` i stedet for `sudo`, logrotate via containerens crontab |
| FTL v6.9 | 11.11.2026 | Minor-utgivelse | sju sikkerhetsråd (ett High), dnsmasq v2.93+42, SQLite 3.54.1, TRE-oppdatering med endret regex-oppførsel |
| Core v6.6 | 11.11.2026 | Minor-utgivelse | fem sikkerhetsråd (fire High, ett Medium), installasjonsprogrammet skriver i `/etc/pihole` bare som brukeren `pihole` |
| Web v6.6.1 | 17.10.2026 | uendret | ingen ny versjon i denne utgivelsen |

## Sikkerhetsrettelser med alvorlighetsgrad High

Sikkerhetsrådene har vært publisert siden 11. november 2026. Det var ikke tildelt noen CVE-ID for de gjennomgåtte sikkerhetsrådene på dette tidspunktet; ifølge teksten er den søkt om for sikkerhetsrådet om `files.gravity` (per 12. november 2026).

- **GHSA-pj9m-9623-2g75 (FTL) og GHSA-cq9w-mv93-mxh7 (Core), CVSS 7.2:** Kodekjøring av en autentisert administrator (CWE-78, CWE-89). FTL 6.8 hadde løsnet på sti-kontrollen, slik at `files.gravity` kunne inneholde et enkelt anførselstegn. `gravity.sh` satte denne verdien ubeskyttet inn i en SQL-`ATTACH`-setning; via FTLs sqlite3-skall kunne det dermed kjøres operativsystemkommandoer som brukeren `pihole`, ifølge sikkerhetsrådet både i Docker- og bare-metal-installasjoner. Core opptil 6.5 og FTL 6.8 er berørt. Core maskerer nå verdien, FTL binder stien som parameter i Teleporter. Produsenten understreker at begge pakkene må oppdateres samtidig.
- **GHSA-wp3p-45h3-2v82 (Core), CVSS 7.3:** Ufullstendig retting av symlink-sårbarhetene fra oktober (CWE-59, CWE-61). `pihole -r` og `pihole -up` fulgte fortsatt symlinker ved skriving av `install.log`, `migration_backup_v6` og `hosts/custom.list` i `/etc/pihole`. Forutsetningen er kodekjøring som brukeren `pihole` og at en administrator deretter starter reparasjon eller oppdatering som root. Versjoner opptil 6.5 er berørt.
- **GHSA-5w4v-f65m-323f (Core), CVSS 7.3:** root fulgte symlinker og stier fra konfigurasjonen ved nedlasting av `macvendor.db`, ved `pihole checkout` og ved `pihole uninstall`.
- **GHSA-jvg9-fm8x-2jqq (Core), CVSS 7.3:** `pihole -d` fulgte symlinker i kataloger som eies av brukeren `pihole`. root kunne dermed kopiere vilkårlige filer inn i debugloggen og overskrive andre eller endre eieren deres.

Core v6.6 skriver derfor `install.log`, migreringsbackupen, `hosts/custom.list` og `macvendor.db` bare som brukeren `pihole`. Også `pihole -d` arbeider ikke lenger som root i disse katalogene.

## Passordhasher i debugloggen

**GHSA-qfq3-88xg-765w (Core), CVSS 5.3:** Debugloggen fra `pihole -d` og dermed også opplastinger til Tricorder-tjenesten inneholdt hashen til webpassordet, hashen til app-passordet og TOTP-hemmeligheten. Core v6.6 fjerner disse verdiene fra loggen. Dersom du tidligere har lastet opp en debuglogg, anbefaler produsenten å endre passordet og app-passordene samt sette opp tofaktorautentisering på nytt.

## Sikkerhetsrettelser med alvorlighetsgrad Low i FTL

- **GHSA-x6xh-vxcm-g8rm, CVSS 3.8:** App-økter uten `webserver.api.app_sudo` og CLI-økter kunne liste alle API-økter og avslutte vilkårlige av dem.
- **GHSA-87wp-wm3p-8cxm, CVSS 3.7:** IPv4-oppføringer i `webserver.acl` gjaldt ikke for IPv4-klienter på en dual-stack-`webserver.port` som `+80`.
- **GHSA-cg4p-5r3r-m54p, CVSS 3.1:** En passordendring via `pihole-FTL --config` eller `pihole setpassword` kunne tilbakeføres av en samtidig konfigurasjonsskriving via API-et; det gamle passordet og øktene knyttet til det forble da gyldige.
- **GHSA-fj2f-pgf2-j888, CVSS 2.2:** Forespørsler som ble registrert med `misc.privacylevel` 3, kunne havne i langtidsdatabasen etter at nivået ble senket.
- **GHSA-9mh9-95wj-4w4q og GHSA-9hh4-j8jf-6c96, begge CVSS 2.2:** Et krasj ved lesing av målinger under en nyinnlasting av hosts-filen samt use-after-free ved en Teleporter-import i v5-format. Begge forutsetter en gyldig økt.

## Endret oppførsel etter oppdateringen

FTL v6.9 er ifølge Release Notes bevisst ikke en v6.8.1, fordi TRE-oppdateringen endrer tolkningen av enkelte lagrede regex-filtre:

- **Regex:** En manglende nedre grense (`{,n}`, `{,}`) betyr nå `{0,n}` eller `{0,}`. Mer enn tre nestede grupper med approximativ matching og back-references i ubegrensede gjentakelser blir avvist.
- **Local DNS:** Oppføringer i `dns.cnameRecords` med ugyldig domenenavn eller en TTL som ikke får plass i en integer, blir avvist.
- **webserver.acl:** IPv4-oppføringer gjelder på alle porter, også på dual-stack-porter. IPv6-oppføringer unntatt `::ffff:a.b.c.d` matcher nå bare IPv6-klienter; `[::]/0` tillater dermed ikke lenger IPv4-klienter.
- **Sessions-API:** App-økter uten `webserver.api.app_sudo` og CLI-økter får status 403 ved `GET /api/auth/sessions` og `DELETE /api/auth/session/{id}`. En økt kan fortsatt avslutte seg selv med `DELETE /api/auth`.
- **Webserver:** Forespørsler med mer enn 64 headere blir avvist. `/api/info/system` returnerer en feil i stedet for nullverdier dersom load averages ikke kan leses.
- **DHCP-API:** `DELETE /api/dhcp/leases/{ip}` svarer med 503 i stedet for 400 dersom resolveren ikke har behandlet forespørselen i tide.
- **Passord:** Et passord som er satt via `pihole setpassword` eller `pihole-FTL --config`, blir ikke lenger overskrevet av neste konfigurasjonsendring i Web Interface.

Kontroller etter oppdateringen regex-filtre med nedre grenser eller back-references, oppføringene i `webserver.acl` (særlig `[::]/0`) og skript eller integrasjoner som lister eller avslutter økter med app-passord.

## DNS, DHCP og ytterligere rettelser i FTL v6.9

Den innebygde dnsmasq er på v2.93+42 og inneholder i tillegg en rettelse i `blockdata_alloc()`, som ennå ikke er tatt inn oppstrøms. Ifølge Release Notes er blant annet en heap-out-of-bounds-read i `zone_status()` (DNSSEC), en heap-out-of-bounds-write i `blockdata_read()`, en overflyt i DNS-navnebufferen, en off-by-one-feil i DHCPv6-FQDN-alternativet samt SERVFAIL ved dypere wildcard-svar og tapte svar ved TCP-pipelining rettet. Med DNSSEC aktivert holder dnsmasq ekstra TCP-spor ledige for egne valideringsforespørsler, slik at valideringen også fungerer når klienter bruker opp TCP-tilkoblingsgrensen. Den ufiltrerte upstream-serveren fra Quad9 er nå merket som DNSSEC-validerende.

For konfigurasjon og API retter FTL v6.9 blant annet tapte endringer ved to samtidige konfigurasjonsendringer, en TCP-worker som holdt `pihole.toml.lock` i opptil 300 sekunder, og ignorerte `FTLCONF_`-variabler når FTL startet uten `pihole.toml`. Webserveren kunne tidligere henge med 100 % CPU-belastning og ikke lenger ta imot tilkoblinger. I containere rapporterte `/api/info/system` hittil belastningen på verten. Også relevant for drift er at `gravity.db` bare åpnes på nytt når filen er endret; ifølge produsenten utgjorde dette den største delen av FTLs CPU-belastning i hvile. SQLite er på 3.54.1.

## Endringer i Core v6.6 og Docker-imaget

Core v6.6 bytter ved rettighetsreduksjon fra root til brukeren `pihole` fra `sudo` til `setpriv`; `sudo` er dermed ikke lenger en avhengighet. Administrative kommandoer som `pihole -up` trenger ifølge produsenten fortsatt root-rettigheter. Loggrotasjonen overtas nå av systemets daglige logrotate; Pi-holes nattlige tvungne rotasjon faller bort fordi den kolliderte med systemkjøringen og roterte logger dobbelt. Ytterligere rettelser gjelder `pihole flush quiet`, `pihole updatechecker` og en advarsel når et lokalt repository for git ikke er markert som sikkert (tidligere viste `pihole -v` da bare N/A). Den unødvendige capability-en `CAP_IPC_LOCK` er fjernet.

Docker-imaget overtar byttet til `setpriv` og planlegger loggrotasjonen via containerens crontab. Det nattlige imaget bygges nå fra branchen `development`.

## Anbefalt fremgangsmåte

Målversjonen er Docker-imaget 2026.11.0, eller FTL v6.9 med Core v6.6 for bare-metal-installasjoner. Siden sårbarheten via `files.gravity` først er lukket med begge komponentene, bør du oppdatere FTL og Core samtidig. Opprett på forhånd en sikkerhetskopi med Teleporter (Settings, Teleporter i Web Interface eller `pihole-FTL --teleporter`), slik produsenten anbefaler, og les endringsloggene på grunn av atferdsendringene. Bare-metal-installasjoner oppdateres med `pihole -up`. I containeren er `pihole -up` og `pihole -r` deaktivert ifølge dokumentasjonen; der erstatter du imaget. Med Docker Compose kjører du følgende kommandoer i katalogen til `docker-compose.yml` ifølge dokumentasjonen:

```bash
docker compose pull
docker compose down
docker compose up -d
```

For installasjoner med `docker run` oppgir dokumentasjonen `docker pull pihole/pihole:latest`, deretter `docker stop pihole` og `docker rm pihole` og en ny `docker run` med de tidligere argumentene. Dersom du tidligere har lastet opp en debuglogg, må du i tillegg endre passordene og sette opp 2FA på nytt. Release Notes inneholder ingen opplysninger om kjente problemer eller støttefrister (per 12. november 2026).

## Kilder

1.  [GitHub: pi-hole/docker-pi-hole Release 2026.11.0](https://github.com/pi-hole/docker-pi-hole/releases/tag/2026.11.0): Release Notes for Docker-imaget med komponentversjoner, Docker-endringer og liste over de elleve sikkerhetsrådene.

2.  [GitHub: pi-hole/FTL Release v6.9](https://github.com/pi-hole/FTL/releases/tag/v6.9): Sikkerhetsrettelser, dnsmasq- og SQLite-oppdatering, atferdsendringer for regex, ACL og API.

3.  [GitHub: pi-hole/pi-hole Release v6.6](https://github.com/pi-hole/pi-hole/releases/tag/v6.6): Core-sikkerhetsrettelser, bytte til setpriv, endringer i loggrotasjonen.

4.  [GitHub: pi-hole/web Release v6.6.1](https://github.com/pi-hole/web/releases/tag/v6.6.1): Status for Web Interface per 17. oktober 2026, uendret i 2026.11.0.

5.  [GitHub: pi-hole/docker-pi-hole Release 2026.10.1](https://github.com/pi-hole/docker-pi-hole/releases/tag/2026.10.1): Forgjengerimage med Alpine 3.24.3.

6.  [GitHub Advisory GHSA-pj9m-9623-2g75](https://github.com/pi-hole/FTL/security/advisories/GHSA-pj9m-9623-2g75): Admin-kodekjøring via files.gravity, CVSS 7.2, berørte og rettede versjoner, CVE søkt om.

7.  [GitHub Advisory GHSA-wp3p-45h3-2v82](https://github.com/pi-hole/pi-hole/security/advisories/GHSA-wp3p-45h3-2v82): Ufullstendig symlink-retting i installasjonsprogrammet, CVSS 7.3, forutsetninger.

8.  [Pi-hole Blog: Pi-hole FTL v6.9 and Core v6.6 released](https://pi-hole.net/blog/2026/11/11/pi-hole-ftl-v6-9-and-core-v6-6-released/): Kunngjøring med oppdateringsmerknader og anbefaling etter opplasting av debuglogger.

9.  [Pi-hole Docs: Upgrading Pi-hole in Docker](https://docs.pi-hole.net/docker/upgrading/): Oppdateringskommandoer for Docker Compose og docker run, pihole -up deaktivert i containeren.
