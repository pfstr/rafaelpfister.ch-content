---
title: "Pi-hole 2026.11.0: FTL 6.9 och Core 6.6 åtgärdar elva säkerhetsbrister"
navTitle: "Pi-hole 2026.11.0"
description: "Docker-imagen Pi-hole 2026.11.0 samlar FTL v6.9 och Core v6.6 och åtgärdar bland annat kodkörning för administratörer via files.gravity (CVSS 7.2) samt flera symlink-sårbarheter i installationsprogrammet. Regex-filter, webserver.acl och Sessions-API:et beter sig delvis annorlunda; kontrollera dessa punkter efter uppdateringen."
date: "2026-11-12"
kategorie: "Nätverk och DNS"
timeToRead: "7 min lästid"
themen:
  - smart-home-iot
produkte:
  - "pihole"
protokolle:
  - "releases"
  - "dns"
  - "container"
slug: "pi-hole-2026-11-0-ftl-6-9-och-core-6-6-atgardar-elva-sakerhetsbrister"
translationId: article-0eb045e8f65bedd0
translationOf: pihole-2026-11-0
url: https://rafaelpfister.ch/sv/blog/pi-hole-2026-11-0-ftl-6-9-och-core-6-6-atgardar-elva-sakerhetsbrister
translationSourceHash: 44a4448ed57216f76890c2807ea763f7c340fcdc0fffa70b8e8e50816c28f4c4
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T13:43:26.989Z
translationReview: automatic
---

# Pi-hole 2026.11.0: FTL 6.9 och Core 6.6 åtgärdar elva säkerhetsbrister

Pi-hole publicerade Docker-imagen 2026.11.0 den 11 november 2026. Den innehåller Pi-hole FTL v6.9 och Pi-hole Core v6.6, som släpptes samma dag; Web Interface ligger enligt tillverkaren kvar på v6.6.1. De båda komponentreleaserna åtgärdar elva säkerhetsråd, varav fyra med allvarlighetsgraden High (CVSS 7.2 till 7.3). FTL v6.9 uppdaterar dessutom dnsmasq, SQLite och regexmotorn TRE samt ändrar beteendet för regexfilter, `webserver.acl` och delar av API:et. Releasen riktar sig till alla som driver Pi-hole, både i Docker-containrar och som Bare Metal-installation. Föregångaren 2026.10.1 från den 31 oktober 2026 innehöll enligt Release Notes främst en uppdaterad Alpine-basavbildning (3.24.3).

## Versionerna i korthet

| Version | Datum | Typ | Huvudpunkter |
|---|---|---|---|
| Docker 2026.11.0 | 11.11.2026 | Image-release | samlar FTL v6.9, Core v6.6 och Web v6.6.1; `setpriv` i stället för `sudo`, logrotate via containerns Crontab |
| FTL v6.9 | 11.11.2026 | Minor-release | sju säkerhetsråd (ett High), dnsmasq v2.93+42, SQLite 3.54.1, TRE-uppdatering med ändrat regexbeteende |
| Core v6.6 | 11.11.2026 | Minor-release | fem säkerhetsråd (fyra High, ett Medium), installationsprogrammet skriver i `/etc/pihole` endast som användaren `pihole` |
| Web v6.6.1 | 17.10.2026 | oförändrad | ingen ny version i denna release |

## Säkerhetskorrigeringar med allvarlighetsgraden High

Säkerhetsråden har publicerats sedan den 11 november 2026. Något CVE-ID hade vid denna tidpunkt inte tilldelats för de granskade säkerhetsråden; enligt texten har det ansökts om ett för säkerhetsrådet om `files.gravity` (status den 12 november 2026).

- **GHSA-pj9m-9623-2g75 (FTL) och GHSA-cq9w-mv93-mxh7 (Core), CVSS 7.2:** Kodkörning genom en autentiserad administratör (CWE-78, CWE-89). FTL 6.8 hade lättat på sökvägskontrollen, så att `files.gravity` fick innehålla ett enkelt citattecken. `gravity.sh` infogade detta värde oskyddat i en SQL-`ATTACH`-sats; via FTL:s sqlite3-skal kunde därmed operativsystemkommandon köras som användaren `pihole`, enligt säkerhetsrådet både i Docker- och Bare Metal-installationer. Core upp till 6.5 och FTL 6.8 påverkas. Core maskerar nu värdet och FTL binder sökvägen som en parameter i Teleporter. Tillverkaren betonar att båda paketen måste uppdateras tillsammans.
- **GHSA-wp3p-45h3-2v82 (Core), CVSS 7.3:** Ofullständig korrigering av symlink-sårbarheterna från oktober (CWE-59, CWE-61). `pihole -r` och `pihole -up` följde fortfarande symlänkar vid skrivning av `install.log`, `migration_backup_v6` och `hosts/custom.list` i `/etc/pihole`. Förutsättningen är kodkörning som användaren `pihole` och en administratör som därefter startar reparation eller uppdatering som root. Versioner upp till 6.5 påverkas.
- **GHSA-5w4v-f65m-323f (Core), CVSS 7.3:** root följde symlänkar och sökvägar från konfigurationen vid nedladdning av `macvendor.db`, vid `pihole checkout` och vid `pihole uninstall`.
- **GHSA-jvg9-fm8x-2jqq (Core), CVSS 7.3:** `pihole -d` följde symlänkar i kataloger som ägs av användaren `pihole`. root kunde på så sätt kopiera godtyckliga filer till debugloggen och skriva över andra eller ändra deras ägare.

Core v6.6 skriver därför `install.log`, migrationsbackupen, `hosts/custom.list` och `macvendor.db` endast som användaren `pihole`. Även `pihole -d` arbetar inte längre som root i dessa kataloger.

## Lösenordshashar i debugloggen

**GHSA-qfq3-88xg-765w (Core), CVSS 5.3:** Debugloggen från `pihole -d` och därmed även uppladdningar till Tricorder-tjänsten innehöll hashen för webblösenordet, hashen för applösenordet och TOTP-hemligheten. Core v6.6 tar bort dessa värden ur loggen. Om du tidigare har laddat upp en debuglogg rekommenderar tillverkaren att du ändrar lösenordet och applösenorden samt konfigurerar tvåfaktorsautentisering på nytt.

## Säkerhetskorrigeringar med allvarlighetsgraden Low i FTL

- **GHSA-x6xh-vxcm-g8rm, CVSS 3.8:** Appsessioner utan `webserver.api.app_sudo` och CLI-sessioner kunde lista alla API-sessioner och avsluta valfria sådana.
- **GHSA-87wp-wm3p-8cxm, CVSS 3.7:** IPv4-poster i `webserver.acl` gällde inte för IPv4-klienter på en Dual Stack-`webserver.port` som `+80`.
- **GHSA-cg4p-5r3r-m54p, CVSS 3.1:** En lösenordsändring via `pihole-FTL --config` eller `pihole setpassword` kunde återställas av en samtidig konfigurationsskrivning via API:et; det gamla lösenordet och dess sessioner förblev då giltiga.
- **GHSA-fj2f-pgf2-j888, CVSS 2.2:** Frågor som registrerades med `misc.privacylevel` 3 kunde hamna i långtidsdatabasen efter att nivån sänkts.
- **GHSA-9mh9-95wj-4w4q och GHSA-9hh4-j8jf-6c96, vardera CVSS 2.2:** En krasch vid läsning av mätvärden under omladdning av hosts-filen samt en use-after-free vid en Teleporter-import i v5-format. Båda kräver en giltig session.

## Ändrat beteende efter uppdateringen

Enligt Release Notes är FTL v6.9 medvetet inte v6.8.1, eftersom TRE-uppdateringen förändrar utvärderingen av vissa sparade regexfilter:

- **Regex:** En saknad undergräns (`{,n}`, `{,}`) betyder nu `{0,n}` respektive `{0,}`. Fler än tre kapslade grupper med approximativ matchning och back-references i obegränsade upprepningar avvisas.
- **Local DNS:** Poster i `dns.cnameRecords` med ogiltigt domännamn eller en TTL som inte ryms i ett heltal avvisas.
- **webserver.acl:** IPv4-poster gäller på alla portar, även på Dual Stack-portar. IPv6-poster utom `::ffff:a.b.c.d` matchar endast IPv6-klienter; `[::]/0` tillåter därmed inte längre IPv4-klienter.
- **Sessions-API:** Appsessioner utan `webserver.api.app_sudo` och CLI-sessioner får status 403 vid `GET /api/auth/sessions` och `DELETE /api/auth/session/{id}`. En session kan fortfarande avsluta sig själv med `DELETE /api/auth`.
- **Webbserver:** Förfrågningar med fler än 64 headers avvisas. `/api/info/system` returnerar ett fel i stället för nollvärden om Load Averages inte kan läsas.
- **DHCP-API:** `DELETE /api/dhcp/leases/{ip}` svarar med 503 i stället för 400 när resolvern inte har behandlat förfrågan i tid.
- **Lösenord:** Ett lösenord som anges via `pihole setpassword` eller `pihole-FTL --config` skrivs inte längre över av nästa konfigurationsändring i Web Interface.

Kontrollera efter uppdateringen regexfilter med undergränser eller back-references, posterna i `webserver.acl` (särskilt `[::]/0`) samt skript eller integrationer som listar eller avslutar sessioner med applösenord.

## DNS, DHCP och ytterligare korrigeringar i FTL v6.9

Den inbäddade dnsmasq är på v2.93+42 och innehåller dessutom en korrigering i `blockdata_alloc()` som ännu inte har tagits uppströms. Enligt Release Notes har bland annat en heap-out-of-bounds-läsning i `zone_status()` (DNSSEC), en heap-out-of-bounds-skrivning i `blockdata_read()`, ett överflöde i DNS-namnbufferten, ett off-by-one-fel i DHCPv6-FQDN-alternativet samt SERVFAIL vid djupare wildcard-svar och förlorade svar vid TCP-pipelining åtgärdats. Med DNSSEC aktiverat reserverar dnsmasq ytterligare TCP-slots för egna valideringsfrågor, så att valideringen fungerar även när klienter utnyttjar TCP-anslutningsgränsen fullt ut. Den ofiltrerade uppströmsservern från Quad9 är nu markerad som DNSSEC-validerande.

För konfiguration och API åtgärdar FTL v6.9 bland annat förlorade ändringar vid två samtidiga konfigurationsändringar, en TCP-worker som höll `pihole.toml.lock` i upp till 300 sekunder och ignorerade `FTLCONF_`-variabler när FTL startade utan `pihole.toml`. Webbservern kunde tidigare fastna med 100 % CPU-belastning och inte längre acceptera anslutningar. I containrar rapporterade `/api/info/system` tidigare värdens belastning. För driften är det dessutom relevant att `gravity.db` nu endast öppnas på nytt när filen har ändrats; enligt tillverkaren utgjorde detta den största delen av FTL:s CPU-belastning vid tomgång. SQLite är på 3.54.1.

## Ändringar i Core v6.6 och Docker-imagen

Core v6.6 byter vid rättighetsnedtrappning från root till användaren `pihole` från `sudo` till `setpriv`; `sudo` är därmed inte längre ett beroende. Enligt tillverkaren kräver administrativa kommandon som `pihole -up` fortfarande root-rättigheter. Loggrotationen hanteras nu av systemets dagliga logrotate; Pi-holes nattliga tvingade rotation tas bort eftersom den kolliderade med systemkörningen och roterade loggar dubbelt. Ytterligare korrigeringar rör `pihole flush quiet`, `pihole updatechecker` och en varning när ett lokalt git-arkiv inte är markerat som säkert (tidigare visade `pihole -v` då endast N/A). Den onödiga capabilityn `CAP_IPC_LOCK` har tagits bort.

Docker-imagen tar över bytet till `setpriv` och schemalägger loggrotation via containerns Crontab. Den nattliga imagen byggs nu från grenen `development`.

## Rekommenderat tillvägagångssätt

Målversionen är Docker-imagen 2026.11.0 respektive FTL v6.9 med Core v6.6 för Bare Metal-installationer. Eftersom sårbarheten via `files.gravity` först stängs med båda komponenterna bör du uppdatera FTL och Core tillsammans. Skapa först en säkerhetskopia med Teleporter (Settings, Teleporter i Web Interface eller `pihole-FTL --teleporter`), som tillverkaren rekommenderar, och läs ändringsloggarna med anledning av beteendeförändringarna. Bare Metal-installationer uppdateras med `pihole -up`. I containern är `pihole -up` och `pihole -r` enligt dokumentationen inaktiverade; där ersätter du imagen. Med Docker Compose kör du enligt dokumentationen följande kommandon i katalogen för `docker-compose.yml`:

```bash
docker compose pull
docker compose down
docker compose up -d
```

För installationer med `docker run` anger dokumentationen `docker pull pihole/pihole:latest`, därefter `docker stop pihole` och `docker rm pihole` samt en ny `docker run` med de tidigare argumenten. Om du tidigare har laddat upp en debuglogg ska du dessutom ändra lösenorden och konfigurera 2FA på nytt. Release Notes innehåller inga uppgifter om Known Issues eller supportfrister (status den 12 november 2026).

## Källor

1.  [GitHub: pi-hole/docker-pi-hole Release 2026.11.0](https://github.com/pi-hole/docker-pi-hole/releases/tag/2026.11.0): Release Notes för Docker-imagen med komponentversioner, Docker-ändringar och lista över de elva säkerhetsråden.

2.  [GitHub: pi-hole/FTL Release v6.9](https://github.com/pi-hole/FTL/releases/tag/v6.9): Säkerhetskorrigeringar, uppdateringar av dnsmasq och SQLite, beteendeförändringar för regex, ACL och API.

3.  [GitHub: pi-hole/pi-hole Release v6.6](https://github.com/pi-hole/pi-hole/releases/tag/v6.6): Säkerhetskorrigeringar i Core, byte till setpriv, ändringar i loggrotationen.

4.  [GitHub: pi-hole/web Release v6.6.1](https://github.com/pi-hole/web/releases/tag/v6.6.1): Status för Web Interface från den 17 oktober 2026, oförändrat i 2026.11.0.

5.  [GitHub: pi-hole/docker-pi-hole Release 2026.10.1](https://github.com/pi-hole/docker-pi-hole/releases/tag/2026.10.1): Föregående image med Alpine 3.24.3.

6.  [GitHub Advisory GHSA-pj9m-9623-2g75](https://github.com/pi-hole/FTL/security/advisories/GHSA-pj9m-9623-2g75): Admin-kodkörning via files.gravity, CVSS 7.2, berörda och korrigerade versioner, CVE ansökt.

7.  [GitHub Advisory GHSA-wp3p-45h3-2v82](https://github.com/pi-hole/pi-hole/security/advisories/GHSA-wp3p-45h3-2v82): Ofullständig symlink-korrigering i installationsprogrammet, CVSS 7.3, förutsättningar.

8.  [Pi-hole Blog: Pi-hole FTL v6.9 and Core v6.6 released](https://pi-hole.net/blog/2026/11/11/pi-hole-ftl-v6-9-and-core-v6-6-released/): Tillkännagivande med uppdateringsanvisningar och rekommendation efter uppladdningar av debugloggar.

9.  [Pi-hole Docs: Upgrading Pi-hole in Docker](https://docs.pi-hole.net/docker/upgrading/): Uppdateringskommandon för Docker Compose och docker run, pihole -up inaktiverat i containern.
