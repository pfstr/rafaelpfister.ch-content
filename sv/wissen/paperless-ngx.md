---
title: "Paperless-ngx: dokumentpipeline, OCR och arkivdrift"
blatt: "paperless-ngx"
description: "Paperless-ngx för administratörer av DMS, plattformar och meddelandetjänster: Django-/Angular-arkitektur, consumer- och OCR-pipeline, original- och arkivfiler, metadata, Tantivy-sökning, Celery och Redis, PostgreSQL, Tika/Gotenberg, behörigheter, e-posthämtning, API, drift, backup, restore och teknisk historia."
fakten:
  - label: Systemroll
    wert: självhostat dokumenthanteringssystem för insamling, OCR, klassificering, sökning, lagring och export
    href: https://docs.paperless-ngx.com/
  - label: Teknikstack
    wert: Python och Django i backend, Angular i webbläsaren, Celery-uppgifter, OCRmyPDF/Tesseract och Tantivy-sökindex
    href: https://github.com/paperless-ngx/paperless-ngx
  - label: Obligatoriska tjänster
    wert: webb-/uppgiftsprocesser, Redis-kompatibel broker, databas och persistenta Data-/Media-kataloger
    href: https://docs.paperless-ngx.com/configuration/
  - label: Databas
    wert: PostgreSQL rekommenderas för nya installationer; MariaDB och SQLite stöds också
    href: https://docs.paperless-ngx.com/setup/
  - label: Dokumentobjekt
    wert: databaspost plus original, valfri arkiv-PDF, miniatyrbild, fulltextindex och kontrollsummor
    href: https://docs.paperless-ngx.com/faq/
  - label: Ingångar
    wert: Consume-katalog, uppladdning via webbläsare/API och regelbaserad IMAP-e-posthämtning
    href: https://docs.paperless-ngx.com/usage/
  - label: OCR
    wert: OCRmyPDF orkestrerar rastrering och Tesseract; läge, språk, sidgräns och arkivformat är driftsparametrar
    href: https://docs.paperless-ngx.com/configuration/
  - label: Office och e-post
    wert: valfri konvertering via Apache Tika och Gotenberg; de båda tjänsterna utgör ett eget felområde
    href: https://docs.paperless-ngx.com/configuration/#optional-services
  - label: Sökning
    wert: Tantivy-index bredvid den relationella databasen; indexet är härledbart, databasen och originalen är auktoritativa
    href: https://docs.paperless-ngx.com/migration-v3/
  - label: Behörigheter
    wert: globala Django-rättigheter plus objektbaserade Owner-, View- och Change-rättigheter för användare och grupper
    href: https://docs.paperless-ngx.com/usage/#permissions
  - label: Kärnan i backupen
    wert: konsekvent export eller samordnad säkerhetskopiering av databas, Media, Data, konfiguration och hemligheter
    href: https://docs.paperless-ngx.com/administration/#making-backups
  - label: Godkännanderegel
    wert: restore till en tom, lämplig instans; inloggning, dokumentantal, stickprov, sökning, kontrollsummor och en ny Consume-körning måste fungera
    href: https://docs.paperless-ngx.com/administration/#restoring
werbung:
  - newsletter
ctaThemen:
  - paperless-ngx
translationSourceHash: a172bf8c7ed80cecd7c990d657cfdf7d06afe1c0b349350655549c21ed144dfc
translationModel: gpt-5.6-terra
translatedAt: 2026-10-10T10:44:00.616Z
translationReview: automatic
---

# Paperless-ngx: dokumentpipeline, OCR och arkivdrift

Paperless-ngx är ett självhostat dokumenthanteringssystem för inkommande filer och skannade pappersdokument. Det tar emot dokument från en övervakad katalog, från webben respektive REST-API:t eller från e-postlådor, skapar sökbar text och vid behov ett normaliserat arkivexemplar, tilldelar metadata och gör beståndet sökbart via webbgränssnitt och API. [Projektöversikten](https://docs.paperless-ngx.com/) beskriver tjänsten som ett sökbart onlinearkiv; [källkodsförrådet](https://github.com/paperless-ngx/paperless-ngx) anger samtidigt att den är efterföljaren till Paperless och Paperless-ng som drivs av communityn.

För administratörer är Paperless-ngx inte en enskild webbapplikation, utan en sammansättning av databas, broker, webb- och bakgrundsprocesser, filbestånd, sökindex och externa konverterare. En lyckad HTTP-status bevisar bara att webbprocessen svarar. Den säger inte om consumer-processen upptäcker nya filer, Celery-uppgifter levereras, OCR fungerar, arkivfiler skrivs, indexet är aktuellt eller en export kan importeras igen.

Förklaringen följer ett dokument från inkommande leverans via consumer, parser och OCR till databas, arkivfil och sökindex. Därefter följer automatisering, säkerhet, drift och en återställning som kontrollerar alla delar tillsammans.

Paperless-ngx bearbetar ett dokument som en pipeline: upptäcka ingången, överta filen, skapa OCR och metadata, uppdatera databasen och sökindexet samt tillhandahålla resultatet via webb eller API. Artikeln följer denna väg och härleder drift, backup och felsökning ur den.

## Arkitektur: styrflöde, arbetsflöde och dataflöde

Arkitekturen kan delas upp i tre flöden:

1. **Styrflöde:** Webbläsaren eller API-klienten kommunicerar med Django-applikationen via HTTP respektive HTTPS. Angular-gränssnittet är den interaktiva klienten; Django tillhandahåller autentisering, affärslogik, administrativt gränssnitt och REST-API. Att projektet består av en Django-applikation och ett Angular-gränssnitt dokumenteras i [utvecklingsdokumentationen](https://docs.paperless-ngx.com/development/).
2. **Arbetsflöde:** Consumer, scheduler och worker skapar och bearbetar asynkrona uppgifter. En Redis-kompatibel broker förmedlar mellan processer; Celery utför bland annat konsumtion, e-posthämtning, indexunderhåll och återkommande underhåll. [Användningsdokumentationen](https://docs.paperless-ngx.com/usage/) beskriver consumer, task processor och broker som separata roller.
3. **Dataflöde:** Den relationella databasen innehåller dokumentobjekt, metadata, användare, regler och uppgiftsstatus. Original, arkivversioner och miniatyrbilder finns i Media-katalogen; hjälpstatus och sökdata finns i Data-katalogen. Sökindexet är en projektion av detta auktoritativa bestånd.

Separationen är operativt avgörande. Databasen och Media-katalogen måste höra till samma logiska tidpunkt. Ett nytt index kan byggas upp, men en förlorad originalfil kan inte återskapas från indexet. På samma sätt räcker inte en databassäkerhetskopia utan tillhörande filer för en DMS-restore.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-paperless-ngx.svg?v=20260813" title="Interaktive Infografik: Paperless-ngx von Eingang, Consumer und Task-Broker über Parser, OCR, Archivierung, Metadaten und Suche bis zu Datenbank, Dateien, Backup und Restore" loading="lazy">
  <a href="/images/kb-interaktiv-paperless-ngx.svg?v=20260813">Öppna den interaktiva grafiken direkt</a>.
</iframe>

## Teknikstack och processroller

Backendkoden är huvudsakligen implementerad i Python och bygger på Django. Webbgränssnittet använder Angular. Beroenden som Django, [Celery](https://docs.celeryq.dev/en/stable/), OCRmyPDF och andra parser kan spåras i [Python-projektmanifestet](https://github.com/paperless-ngx/paperless-ngx/blob/dev/pyproject.toml); imagebygge och inbyggda OCR-/PDF-beroenden visas i [Dockerfile](https://github.com/paperless-ngx/paperless-ngx/blob/dev/Dockerfile). Följande roller är relevanta för drift:

| Roll | Huvuduppgift | Avgörande datakälla | Typisk störning |
|---|---|---|---|
| Webbserver | UI, REST-API, autentisering, uppladdning | Session och DB-transaktion | 4xx/5xx, proxy- eller CSRF-fel |
| Consumer | upptäcka nya filer och initiera uppdrag | Consume-katalog och uppgift | Filen ligger kvar, polling/inotify eller rättigheter |
| Task Worker | parser, OCR, klassificering och arkivsteg | uppgiftsstatus, tillfälliga filer, DB | Uppgift pending/failed, minnes-/CPU-gräns |
| Scheduler | e-posthämtning och periodiskt underhåll | tidplan i uppgiftssystemet | inga periodiska jobb |
| Broker | leverans av asynkrona uppgifter | flyktiga kö-/brokerdata | webben fungerar, bearbetningen står still |
| Databas | dokument, metadata, identiteter, regler | relationellt primärbestånd | transaktions-, schema- eller kapacitetsfel |
| Fillagring | original, arkiv, miniatyrbild | Media-filer | fil saknas, skrivskyddad, inkonsekvent mount |
| Sökindex | fulltextprojektion | härledbara indexdata | dokument finns, sökning ofullständig |
| Tika/Gotenberg | valfri Office-/e-postkonvertering | tillfälligt konverterat resultat | endast vissa filtyper misslyckas |

Den officiella [konfigurationsreferensen](https://docs.paperless-ngx.com/configuration/) betecknar den [Redis-kompatibla brokern](https://redis.io/docs/latest/) som en nödvändig tjänst. [PostgreSQL](https://www.postgresql.org/docs/current/), MariaDB och SQLite stöds som databaser; för nya installationer rekommenderar [installationsguiden](https://docs.paperless-ngx.com/setup/) PostgreSQL. Tika och Gotenberg är valfria och behövs tillsammans för Office-format och bearbetning av e-postfiler ([valfria tjänster](https://docs.paperless-ngx.com/configuration/#optional-services)).

## Tjänsteinventering och faktisk konfiguration

En robust inventering omfattar image eller release, databastyp, brokermål, mountar, effektiva miljövariabler, UID/GID, reverse proxy, tidszon, OCR-språk och externa parser. Docker Compose är en vanlig paketering, men inte något arkitekturbevis: Flera processroller kan finnas i en containerimage medan deras tillstånd ligger i separata volymer och tjänster.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Paperless-Dienstinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker compose config --services
docker compose ps
docker compose images
docker compose config
docker inspect paperless-webserver</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker compose config --services
docker compose ps
docker compose images
docker compose config
docker inspect paperless-webserver</code></pre>
  </div>
</div>

[`docker compose config`](https://docs.docker.com/reference/cli/docker/compose/config/) återger den faktiskt sammanslagna Compose-konfigurationen; [`docker compose ps`](https://docs.docker.com/reference/cli/docker/compose/ps/) visar processer och publicerade portar; [`docker compose images`](https://docs.docker.com/reference/cli/docker/compose/images/) identifierar images; [`docker inspect`](https://docs.docker.com/reference/cli/docker/inspect/) ger mountar, nätverk och körtidsparametrar. Utdata kan innehålla hemligheter och ska endast läggas in i ärenden i maskerad form.

## Ingångar och leveransgränser

Paperless-ngx har flera ingångar som före den gemensamma pipelinen har olika felområden:

- **Consume-katalogen** övervakas med filsystemhändelser. På filsystem utan tillförlitlig inotify, exempelvis vissa NFS-mountar, måste polling aktiveras; detta anges i [setup-dokumentationen](https://docs.paperless-ngx.com/setup/#file-systems-without-inotify-support-eg-nfs).
- **Webbläsare och REST-API** skriver uppladdningar via webbsökvägen. Reverse proxy, gräns för begärans storlek, timeout och temporär lagring ligger därmed före uppgiftssystemet.
- **E-posthämtningen** läser regelbaserat från konfigurerade konton och skapar uppgifter för matchande bilagor. IMAP-anslutning, TLS, OAuth/lösenord, mapp, sökkriterium och åtgärd efter bearbetning hör till leveransavtalet.

Consume-katalogen är en överlämningszon och inte ett dokumentarkiv. Framgångsrikt konsumerade filer tas bort därifrån, medan Paperless skapar nya hanterade filer i Media-beståndet. [FAQ](https://docs.paperless-ngx.com/faq/) varnar uttryckligen för att hantera Media-katalogen manuellt.

När processer och ingångar är kända börjar den faktiska bearbetningen. Varje steg skapar andra filer, databasposter och felsignaler.

## Consume-pipeline: från ingång till dokumentobjekt

Det verksamhetsmässiga committet sker inte när en fil upptäcks, utan först efter flera steg. Beroende på filtyp och konfiguration omfattar pipelinen vanligtvis:

1. **Identifiering och stabilisering:** Filnamn, tillåten typ, filstorlek och avslutad skrivning kontrolleras.
2. **Skapande av uppdrag:** Consumer-processen respektive webbservern överlämnar en uppgift via brokern.
3. **Parsing och förbearbetning:** Parser extraherar befintlig text eller konverterar ingångsdokumentet. Office- och e-postfiler kan behöva Tika och Gotenberg.
4. **OCR:** OCRmyPDF avgör utifrån OCR-läget om textigenkänning ska hoppas över, köras igen eller tvingas fram; Tesseract känner igen konfigurerade språk.
5. **Arkivversion och miniatyrbild:** En visnings-/arkivfil och förhandsvisningsdata skapas i den mån filtyp och inställningar medger det.
6. **Persistens:** Original och derivat skrivs; databasobjekt, kontrollsummor, text och tekniska egenskaper lagras.
7. **Metadata och arbetsflöden:** Regler tilldelar taggar, korrespondent, dokumenttyp, lagringssökväg, ägare eller andra fält.
8. **Indexering:** Fulltextbeståndet uppdateras i sökindexet.
9. **Avslut:** Uppgiftsstatus och logg visar framgång eller fel; först nu är dokumentet verksamhetsmässigt sökbart.

Pre- och post-consume-skript kan utöka flödet. [Dokumentationen om Consume Hooks](https://docs.paperless-ngx.com/advanced_usage/#consume-hooks) klargör att sådana skript körs i Paperless-processens säkerhets- och felkontext. De ska versionshanteras, köras med timeout och tydliga exitkoder; en felaktig hook kan blockera hela inmatningen.

## OCR, arkivformat och kvalitetsgränser

OCRmyPDF orkestrerar PDF-bearbetning och Tesseract-OCR. Läget avgör om befintlig text hoppas över, bearbetas på nytt eller tvingas fram. Språk, sidgräns, bildrensning, rotationskorrigering och utdataformat påverkar CPU-tid, filstorlek och igenkänningskvalitet. Inställningarna beskrivs i [Paperless OCR-konfiguration](https://docs.paperless-ngx.com/configuration/#ocr-settings) och i [OCRmyPDF-dokumentationen](https://ocrmypdf.readthedocs.io/en/latest/); språkmodeller och enginebeteende dokumenteras av [Tesseract](https://tesseract-ocr.github.io/tessdoc/).

Originalet behålls som bevisobjekt. En arkiv-PDF är en härledd version som kan skapas på nytt vid senare ändringar av OCR eller arkivering. Denna skillnad ska ingå i koncepten för bevarande och bevisvärde:

| Objekt | Syfte | Återställbarhet |
|---|---|---|
| Original | oförändrad inmatningsfil | kan inte rekonstrueras från OCR-text |
| Arkivfil | normaliserad, ofta sökbar version | kan skapas på nytt från original och lämplig pipeline |
| Miniatyrbild | UI-förhandsvisning | härledbar |
| extraherad text | sökning och klassificering | kan extraheras igen, resultatet kan skilja sig beroende på version |
| Kontrollsumma | integritetsjämförelse | har bara bevisvärde tillsammans med algoritm och referens |

OCR-framgång betyder inte verksamhetsmässig läsbarhet. Stickprov bör jämföra sidantal, orientering, texttäckning, specialtecken, tabeller, inbäddade signaturer och filstorlek. Toppar i CPU- och RAM-användning uppstår ofta vid rendering av stora eller bildrika PDF-filer, inte vid den avslutande databasskrivningen.

## Original, arkiv, miniatyrbilder och sökvägsformat

Ett Paperless-dokument består av flera kopplade delar. Databasen känner till intern ID, titel, skapande- och tilläggsdatum, korrespondent, dokumenttyp, taggar, lagringssökväg, innehåll, ägare och ytterligare fält. Originalet och eventuellt arkiv- och miniatyrfiler ligger i Media-katalogen. Konfigurerbara filnamn och Storage Paths ändrar den fysiska ordningen, inte databasobjektets identitet.

Administratörer undviker direkta namnbyten eller flyttningar i Media-katalogen. Den officiella [`document_renamer`](https://docs.paperless-ngx.com/administration/#document-renamer) tillämpar det konfigurerade formatet konsekvent. Ett externt filsynkverktyg får säkerhetskopiera beståndet, men inte självständigt organisera om det vid sidan av den körande applikationen. För grunder om lagring, se [Storage](/kb/storage), för kopiering och verifiering [Rclone](/kb/rclone).

## Metadata, matchning och arbetsflöden

Korrespondenter, dokumenttyper, taggar, lagringssökvägar, anpassade fält och datumvärden bildar en fasetterad ordning. De ersätter inte ett filträd, utan beskriver ett dokument längs flera oberoende dimensioner. Automatisk matchning ger förslag respektive tilldelningar baserat på igenkänt innehåll och tidigare data; den är ingen oföränderlig sanningskälla.

Arbetsflöden består av utlösare, valfria filter och sekventiella åtgärder. De kan vid konsumtion eller andra händelser sätta metadata, behörigheter och ytterligare egenskaper. [Workflow-dokumentationen](https://docs.paperless-ngx.com/usage/#workflows) beskriver utlösare och åtgärder. För administratörer följer tre regler:

- Varje automatik behöver en definierad inmatningsmängd och ett observerbart resultat.
- Regelordning och flera träffar testas som policykod.
- Efter ändringar utvärderas befintliga dokument på nytt endast kontrollerat med [`document_retagger`](https://docs.paperless-ngx.com/administration/#document-retagger).

## Sökning: databasfråga och Tantivy-projektion

Fulltextsökningen använder ett separat Tantivy-index. Databasen är fortsatt auktoritativ för dokumentobjekt och metadata; indexet snabbar upp sökning och rankning. Därför kan ett dokument vara synligt i databasen men saknas i fulltextsökningar. Migreringen till Tantivy och ändrade sökfält dokumenteras i [migrationsguiden](https://docs.paperless-ngx.com/migration-v3/#search-index-whoosh-tantivy).

Efter uppgraderingar eller vid misstanke om drift kan indexet byggas om med managementkommandot. Omindexering är en reparation av en projektion, inte en återställning av originalen. Kapacitetsplanering tar hänsyn till indexstorlek, temporärt utrymme under en ombyggnad och I/O-belastningen parallellt med normal konsumtion.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Paperless-Suche und Wartung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker compose exec webserver document_index list
docker compose exec webserver document_index reindex --if-needed
docker compose exec webserver document_sanity_checker
docker compose exec webserver document_thumbnails --help</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker compose exec webserver document_index list
docker compose exec webserver document_index reindex --if-needed
docker compose exec webserver document_sanity_checker
docker compose exec webserver document_thumbnails --help</code></pre>
  </div>
</div>

Managementkommandon startas i Compose via [`docker compose exec`](https://docs.docker.com/reference/cli/docker/compose/exec/) i webbservercontainern. Bare-metal-installationer använder `python3 manage.py`; det stödda anropsformatet och den inbyggda hjälpen dokumenteras i [administrationen](https://docs.paperless-ngx.com/administration/#management-utilities).

OCR-filer och metadata bildar ännu ingen konsekvent tjänst. Databas, taskbroker och sökprojektion måste motsvara samma bearbetningsstatus.

## Databas, broker och konsistens

PostgreSQL är standardvägen för nya installationer. MariaDB och SQLite är möjliga, men skiljer sig åt när det gäller backupverktyg, parallellbeteende och driftsgränser. Databasmigreringar är en del av en Paperless-uppgradering; en återkopierad äldre binärfil passar inte nödvändigtvis till ett redan migrerat schema. Databasdrift behandlas därför som en separat livscykel ([PostgreSQL-backup och restore](https://www.postgresql.org/docs/current/backup.html), [MariaDB Backup and Restore](https://mariadb.com/docs/server/server-usage/backing-up-and-restoring-databases/)).

Brokern transporterar uppgifter mellan processer. Redis-/Valkey-persistens ersätter inte backup av databas och dokument. Efter brokerförlust kan pågående eller levererade arbeten initieras på nytt, förloras eller synas som inkonsekvent uppgiftsstatus. Godkännandet sker därför på dokumentobjektet och dess filer, inte på en tom kö.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Datenbank- und Brokertests">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker compose exec db pg_isready
docker compose exec broker redis-cli PING
docker compose logs --since 30m webserver
docker compose logs --since 30m broker db</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker compose exec db pg_isready
docker compose exec broker redis-cli PING
docker compose logs --since 30m webserver
docker compose logs --since 30m broker db</code></pre>
  </div>
</div>

[`pg_isready`](https://www.postgresql.org/docs/current/app-pg-isready.html) kontrollerar om PostgreSQL-anslutningar accepteras, inte schema eller verksamhetsmässiga frågor. [`redis-cli`](https://redis.io/docs/latest/develop/tools/cli/) testar brokeråtkomst med `PING`. [`docker compose logs`](https://docs.docker.com/reference/cli/docker/compose/logs/) sammanför tidsbegränsade tjänstloggar; credentials, dokumenttitlar och sökvägar maskeras före vidarebefordran.

## Konfiguration, Secret Key och sökvägsupplösning

Med Docker levereras inställningar vanligtvis som miljövariabler respektive Compose-env-fil; `paperless.conf` gäller inte där. Bare-metal-installationer söker konfiguration i en dokumenterad ordning. Vissa inställningar kan dessutom lagras i applikationen och åsidosätta miljövärden. Den effektiva konfigurationen är därför mer än en enskild fil ([konfigurationsordning](https://docs.paperless-ngx.com/configuration/#configuration)).

`PAPERLESS_SECRET_KEY` signerar sessioner och känsliga värden och är obligatorisk. `PAPERLESS_URL`, Allowed Hosts, CSRF- och CORS-inställningar definierar webbgränsen. Credentials för databas, e-post, OAuth, Tika och valfri AI är hemligheter. Docker-konfigurationer kan använda `_FILE`-varianten för många värden och därmed mountade hemligheter; detta dokumenteras i [setup-guiden](https://docs.paperless-ngx.com/setup/#docker-compose-install).

Konfigurationsbackup omfattar minst:

- Compose-/tjänstdefinition och pinnad image-referens,
- miljö utan klartextkopior i oskyddade repositories,
- Secret Key och externa hemlighetsreferenser,
- UID/GID, volym- och bind-mount-tilldelning,
- proxy-, DNS-, certifikat- och e-postkontokonfiguration,
- OCR-språk, filnamnsformat, arbetsflöden och anpassade hooks.

## Webbgräns, TLS och reverse proxy

Applikationen bör vara åtkomlig via HTTPS bakom en korrekt konfigurerad reverse proxy. Hostheader, ursprungs-URL, CSRF Trusted Origins, Forwarded Headers och uppladdningsgränser måste stämma överens. Proxyn får endast sätta Remote-User-header när den tillförlitligt autentiserar den föregående identiteten och tar bort klientheaders med samma namn.

[Säkerhetsdokumentationen](https://docs.paperless-ngx.com/configuration/#hosting-security) beskriver Secret Key, URL-, host-, CORS- och CSRF-parametrar. Grunder för certifikat och transport finns under [TLS](/kb/tls), systematisk härdning under [Härtning](/kb/haertung). Projektet varnar i [repositoryt](https://github.com/paperless-ngx/paperless-ngx#important-note) för att dokument lagras i klartext och att Paperless-ngx inte bör drivas på en opålitlig värd.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Paperless-Web- und Portdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection paperless.example.net -Port 443
curl.exe -sS -D - -o NUL https://paperless.example.net/
Resolve-DnsName paperless.example.net
docker compose port webserver 8000</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ss -lntp
curl -sS -D - -o /dev/null https://paperless.example.net/
getent ahosts paperless.example.net
docker compose port webserver 8000</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) kontrollerar TCP-åtkomst i Windows; [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) visar lokala lyssnare; [`curl`](https://curl.se/docs/manpage.html) visar status, omdirigering och headers; [`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) respektive [`getent`](https://man7.org/linux/man-pages/man1/getent.1.html) kontrollerar namnupplösning; [`docker compose port`](https://docs.docker.com/reference/cli/docker/compose/port/) visar den faktiska portmappningen.

## Identiteter, globala rättigheter och objekträttigheter

Paperless-ngx använder Django-modellen för användare och grupper. Globala behörigheter styr vilka objekttyper och UI-/API-områden ett konto får använda. Objektbaserade rättigheter styr Owner, View och Change för ett visst dokument, en tagg eller ett annat objekt. [Behörighetsdokumentationen](https://docs.paperless-ngx.com/usage/#permissions) betonar att en taggs rättigheter inte automatiskt övergår till dokument som markerats med den.

Superusers får se alla objekt. Staff-/adminstatus ska skiljas från detta och möjliggör bland annat backend- och systemvyer. För den dagliga driften används vanliga konton och grupper; tjänstekonton får endast nödvändiga API- och objekträttigheter. Arbetsflöden kan sätta ägare och rättigheter vid konsumtion, vilket är särskilt viktigt för ägarlösa dokument från Consume-kataloger.

Autentisering kan ske lokalt, med reverse-proxy-Remote-User eller med konfigurerade social-/OIDC-förfaranden. Remote User aktiveras endast bakom en sluten proxygräns. MFA-, sessions- och tokenlivscykel inkluderas i det övergripande identitetskonceptet.

## REST-API och automationsgräns

REST-API:t är åtkomligt under applikationsprefixet; en interaktiv schemavy finns under `/api/schema/view/`. Bland annat stöds Basic-, Session-, Token-, Remote-User- och Headless-OIDC-autentisering. [API-dokumentationen](https://docs.paperless-ngx.com/api/) beskriver sökning, objektbehörigheter, bulkoperationer och versionsheaders.

En API-token är en hemlighet och tilldelas separat för varje automation. Klienter tar hänsyn till paginering, filter, behörigheter, API-version och idempotenta upprepningar. Ett lyckat uppladdningssvar innebär ännu inte att OCR är slutförd; automationer korrelerar dokument- respektive task-ID fram till verksamhetsmässigt slutstatus. Principer för avtal, autentisering och retrybeteende finns under [API:er](/kb/apis).

## E-posthämtning som meddelandegränssnitt

E-postkonton och e-postregler kopplar Paperless-ngx till en IMAP-brevlåda. Schedulern kontrollerar som standard regelbundet konfigurerade konton, tillämpar regler och överlämnar matchande bilagor till Consume-pipelinen. Managementkommandot `mail_fetcher` kan utlösa hämtningen manuellt; detta dokumenteras i [administrationen för e-posthämtning](https://docs.paperless-ngx.com/administration/#fetching-e-mail).

För meddelandeadministratörer ska följande värden ingå i runbooken:

- server-FQDN, port och TLS-läge,
- autentiseringsmetod, utgång av hemlighet/token och tjänstekonto,
- IMAP-mapp, sökkriterium, filtyper och maximal meddelandestorlek,
- åtgärd efter lyckad eller misslyckad import,
- hantering av flera leveranser av samma meddelande och identiska bilagor,
- korrelation mellan Message-ID, Paperless-task och dokument-ID.

IMAP-semantik definieras av [RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051). En lyckad IMAP-hämtning bevisar bara ingången till uppgiftsflödet; OCR- och arkivslutförande övervakas separat.

## Office-dokument, Tika och Gotenberg

Apache Tika extraherar innehåll och metadata från många filtyper; Gotenberg tillhandahåller konverteringstjänster. Paperless behöver båda när den valfria Office-/e-postbearbetningen är aktiverad. Deras ändpunkter konfigureras separat ([Tika Server](https://tika.apache.org/3.2.3/gettingstarted.html#Using_Tika_as_a_service), [Gotenberg-konfiguration](https://gotenberg.dev/docs/configuration)).

De båda tjänsterna ökar attackytan, resursbehovet och felutrymmet. De görs endast åtkomliga internt, får CPU-/RAM-/timeoutgränser och publiceras inte som ett allmänt konverterings-API. Om PDF-filer fungerar men DOCX eller EML konsekvent misslyckas börjar felsökningen i denna sidoväg, inte hos Tantivy eller databasen.

## Container-, bare-metal- och lagringsdrift

De officiella vägarna är installationsskript, Docker Compose och bare metal. Compose ger reproducerbara tjänstdefinitioner och separata volymer; bare metal ger mer kontroll över processer, paket och systemd, men kräver uttrycklig livscykelhantering. [Setup-matrisen](https://docs.paperless-ngx.com/setup/#overview) ordnar vägarna efter arbetsinsats och användning.

Persistenta kataloger förväxlas inte med containerlivscykeln. Med Compose löses Media, Data, Consume, Export och databassökvägen medvetet upp. Fjärr- eller nätverksfilsystem kontrolleras för POSIX-beteende, atomiskt namnbyte, låsning, latens, inotify och tillgänglighet. Ett kort avbrott kan annars se ut som en permanent dokumentförlust. Grunder om mountar finns under [Container](/kb/container) och [Storage](/kb/storage).

Vid en störning fastställs först i vilken pipelinefas dokumentet befinner sig. Därefter kontrolleras kö, worker, extern konverterare, databas och fillagring målmedvetet.

## Observerbarhet och uppgiftsdiagnostik

En meningsfull driftöversikt skiljer mellan fyra signaler:

- **Tjänstehälsa:** Processen körs, porten lyssnar, databas och broker är åtkomliga.
- **Genomströmning:** nya uppgifter, slutförda dokument, körtider och ködjup.
- **Fel:** uppgifter per steg och filtyp, parser-/OCR-exit, e-post- och lagringsfel.
- **Beståndsintegritet:** saknade eller skadade original/arkiv/miniatyrbilder, föräldralösa filer, indexdrift.

Den inbyggda Sanity Checker upptäcker bland annat saknade, oläsbara eller jämfört med den lagrade kontrollsumman förändrade filer samt föräldralösa Media-filer. Den fullständiga listan finns i [Sanity Checker-dokumentationen](https://docs.paperless-ngx.com/administration/#sanity-checker). Ett larm om antal fel utan dokument-/task-ID och pipelinefas är knappast användbart.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Paperless-Task- und Storage-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker compose logs --since 1h --timestamps webserver
docker compose exec webserver document_sanity_checker
docker compose exec webserver mail_fetcher
docker stats --no-stream
docker system df -v</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker compose logs --since 1h --timestamps webserver
docker compose exec webserver document_sanity_checker
docker compose exec webserver mail_fetcher
docker stats --no-stream
docker system df -v</code></pre>
  </div>
</div>

[`docker stats`](https://docs.docker.com/reference/cli/docker/container/stats/) visar momentan CPU-/minnesanvändning, [`docker system df`](https://docs.docker.com/reference/cli/docker/system/df/) Docker-diskanvändning. Båda är ögonblicksbilder. För orsaker korreleras tidsserier, task-ID och Paperless-loggar. Allmän metodik finns under [felsökning](/kb/troubleshooting).

## Kapacitet, parallellitet och gränsvärden

Kapacitetsplanering börjar med sidor per dag, inmatningsformat, genomsnittlig och maximal filstorlek, OCR-språk, parallella uppgifter, bevarandetid och sökvolym. Ett dokument kan samtidigt skapa original, arkiv-PDF, miniatyrbild, databaspost, fulltextindex och tillfälliga renderfiler. Det lediga utrymmet måste även täcka omindexering, export och uppgradering.

Fler parallella workers snabbar upp CPU-bunden OCR endast tills CPU, RAM och I/O är mättade. Samtidigt ökar antalet tillfälliga filer och databasanslutningar. E-posthämtning och consumer får skapa fler uppgifter än workers kan bearbeta; därför övervakas köålder i stället för bara aktuell kölängd. Gränser testas med representativa dokument, inte med små text-PDF:er.

Säkerhetskopieringsomfattningen följer datamodellen: Original, arkiv, databas, konfiguration, Secret Key och eventuella ytterligare tjänster måste kunna återställas tillsammans.

## Backup: vilka objekt som säkerhetskopieras tillsammans

Den officiella [backupdokumentationen](https://docs.paperless-ngx.com/administration/#making-backups) anger två huvudvägar: Document Exporter eller en samordnad säkerhetskopiering av volymer/kataloger och databas. Exportern skriver dokument, miniatyrbilder, metadata och databasinnehåll till en portabel katalog. API-token följer inte med; export och import bör vara versionskompatibla.

En fullständig recoveryuppsättning innehåller:

| Objekt | Varför det behövs | Konsistensregel |
|---|---|---|
| Databas | ID:n, metadata, användare, regler, uppgiftsreferenser | matchar Media-snapshoten |
| Media | original, arkiv, miniatyrbilder | inga halvfärdiga skrivningar |
| Data | index och hjälpstatus | delvis härledbart, men användbart för snabb återstart |
| Konfiguration | sökvägar, URL, OCR, e-post, proxy, databas | samma namngivnings- och mountavtal |
| Hemligheter | Secret Key, DB-/e-post-/OIDC-credentials | skyddade, separata och återställningsbara |
| Exportmanifest | portabel objekt- och filtilldelning | hålls tillsammans med exporterade filer |
| Runbooks | ordning, version, godkännandetester | tillgängliga offline |

Exportern kan uppdatera befintliga exporter och valfritt jämföra kontrollsummor. Det sparar tid, men ersätter inte en separat, versionshanterad kopia: Med `--delete` kan raderade dokument tas bort från exportmålet. Detaljer och alternativ dokumenteras av [`document_exporter`](https://docs.paperless-ngx.com/administration/#document-exporter).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Paperless-Export und Recoveryprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker compose exec -T webserver document_exporter ../export --compare-checksums
Get-ChildItem -Recurse .\export | Measure-Object -Property Length -Sum
Get-FileHash .\export\manifest.json -Algorithm SHA256
docker compose exec webserver document_sanity_checker</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker compose exec -T webserver document_exporter ../export --compare-checksums
du -sh ./export
sha256sum ./export/manifest.json
docker compose exec webserver document_sanity_checker</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash) och [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) säkrar överföringsbeviset för manifestet. En kontrollsumma bevisar bara likhet mellan två byteuppsättningar, inte importerbarhet eller verksamhetsmässig fullständighet. Grundläggande backupprinciper finns under [Backup och Disaster Recovery](/kb/backup-dr).

## Restore och godkännande

[`document_importer`](https://docs.paperless-ngx.com/administration/#document-importer) förutsätter en tom installation respektive, med `--data-only`, en tom databas. Ett restoretest använder därför isolerade målsökvägar, en Paperless-version som passar säkerhetskopian och inga produktionsvolymer.

Godkännandet omfattar minst:

1. Lägg in konfiguration och hemligheter i målmiljön.
2. Tillhandahåll tom databas och tomma Media-/Data-sökvägar.
3. Importera exporten eller återställ databas och filer konsekvent.
4. Kör migreringar och vid behov indexuppbyggnad kontrollerat.
5. Kontrollera användarinloggning, grupper och behörigheter.
6. Jämför dokumentantal, metadata och stickprov från original och arkiv.
7. Testa fulltextsökning och visning av miniatyrbilder.
8. Kör Sanity Checker utan oförklarade fynd.
9. Konsumera ett nytt testdokument via den verkliga ingången.
10. Dokumentera RPO och RTO med tidsstämplar.

En restore anses framgångsrik först efter detta end-to-end-test. ”Containern startar” är inget godkännandekriterium.

## Uppdateringar, databasmigreringar och rollbackgräns

Före en uppdatering avslutas aktiva Consume-uppgifter, en säkerhetskopia skapas och release notes kontrolleras för breaking changes. [Uppdateringsguiden](https://docs.paperless-ngx.com/administration/#updating-paperless) skiljer mellan Docker- och bare-metal-förfaranden och beskriver databasmigrering samt behovsstyrd omindexering. En image pinnas till en godkänd serie eller en digest; `latest` är ingen reproducerbar rollbackpunkt.

Efter en databasmigrering är rollback normalt en restore av databasen och det matchande filbeståndet, inte bara start av den gamla imagen. Nya arkiv eller metadata som tillkommit efter säkerhetskopieringspunkten måste hanteras verksamhetsmässigt före återgången. Allmänna principer för release och change finns under [Releases](/kb/releases) och [Migration](/kb/migration).

[Changelog](https://docs.paperless-ngx.com/changelog/) är den tidsberoende källan för nya funktioner, breaking changes och korrigeringar. Denna artikel beskriver däremot stabila arkitektur- och driftsprinciper.

## Felbilder per pipelinefas

Eftersom ett dokument genomgår flera bearbetningssteg visar det synliga symptomet ofta bara den senast lyckade fasen. Tabellen leder från symptom till nästa robusta bevis.

| Symptom | Trolig gräns | Första belägg |
|---|---|---|
| Filen ligger kvar i Consume-katalogen | watcher, polling, UID/GID, avslutad skrivning | mount, rättigheter, consumerlogg |
| Uppladdningen svarar, men inget dokument skapas | broker eller task worker | task-ID, broker-PING, workerlogg |
| endast DOCX/EML misslyckas | Tika/Gotenberg | ändpunkt, timeout, konverterarlogg |
| OCR är mycket långsam eller processen avslutas | CPU/RAM, sidantal, språkpaket | containerstatistik, kernel-/runtimehändelse, sidstorlek |
| Dokument synligt, fulltext saknas | parser/OCR eller sökindex | extraherad text, tasklogg, omindexeringsstatus |
| Miniatyrbild saknas, original finns | härledd fil | Sanity Checker, miniatyrkommando |
| Inloggning/POST bakom proxy fungerar inte | URL, host, CSRF, Forwarded Header | HTTP-header, effektiv konfiguration |
| periodisk e-posthämtning står still | scheduler, broker, e-postautentisering | schedulerlogg, `mail_fetcher`, IMAP-test |
| Filer försvinner efter omstart | felaktig eller flyktig mount | `docker inspect`, volyminventering |
| Restore importerar inte | versions-, sökvägs- eller tomhetsvillkor | exportmanifest, målversion, importlogg |

Den officiella [felsökningssamlingen](https://docs.paperless-ngx.com/troubleshooting/) kompletterar med konkreta felmeddelanden. Administratörer sparar alltid task-ID, dokumentnamn, tidsfönster, effektiv konfiguration, container-/processstatus och tillståndet för alla beroende tjänster före ändringar.

## Teknisk historia

Det ursprungliga Paperless-projektet etablerade grundmodellen med Consume-katalog, OCR och sökbart arkiv. Paperless-ng fortsatte utvecklingen som en fork. Paperless-ngx uppstod därefter som den officiella efterföljaren som drivs av communityn, för att fördela underhåll och ansvar över flera medverkande; denna härkomst beskrivs i [projektets README](https://github.com/paperless-ngx/paperless-ngx#paperless-ngx).

Plattformen utvecklades från en främst serversidig Django-applikation till ett DMS med Angular-gränssnitt, REST-API, objektbaserade behörigheter, arbetsflödesmotor och parallell uppgiftsbearbetning. OCRmyPDF/Tesseract förblev OCR-grunden; Tika och Gotenberg utökade de stödda inmatningsformaten. Sökvägen bytte från Whoosh till Tantivy, vilket gjorde indexet synligt som en egen migrations- och driftdomän ([Tantivy-migrering](https://docs.paperless-ngx.com/migration-v3/)).

Denna historia förklarar dagens arkitektur: Ett Paperless-ngx-dokument är samtidigt ett relationellt objekt, en uppsättning hanterade filer, ett OCR-resultat och en sökprojektion. Underhåll och recovery måste beakta alla fyra perspektiv.

## Checklista för administratörer

Avslutningsvis kontrolleras ingångar, workers, lagring, sökning och backup gemensamt. Checklistan hjälper till att inte förbise något externt beroende utanför Paperless-containern.

- **Inventering:** Image/commit, Django-/Angular-applikation, worker, scheduler, broker, databas, Tika/Gotenberg, proxy och mountar.
- **Dataobjekt:** Nämn databas, original, arkiv, miniatyrbild, index, konfiguration, hemligheter och exportmanifest separat.
- **Ingångar:** Dokumentera Consume, uppladdning/API och e-post med regler för storlek, typ, autentisering och fel.
- **Pipeline:** Korrelera task-ID genom parser, OCR, arkivering, metadata och index fram till avslut.
- **Behörigheter:** Testa globala och objektbaserade rättigheter, superusers, grupper och arbetsflödestilldelningar.
- **Kapacitet:** Övervaka köålder, OCR-körtid, temporärt utrymme, Media-/indextillväxt och databasanslutningar.
- **Säkerhet:** Inkludera HTTPS, proxyförtroende, Secret Key, token, klartextlagring och externa parser i hotmodellen.
- **Backup:** Säkerhetskopiera databas och Media konsekvent; glöm inte konfiguration och hemligheter.
- **Restore:** Kontrollera tom målinstans, lämplig version, stickprov, sökning, Sanity Checker och ny Consume-körning.
- **Ändring:** Dokumentera release notes, schema-/indexmigrering, ledigt utrymme och faktisk rollbackgräns i förväg.

## Källor

- [Paperless-ngx – dokumentation](https://docs.paperless-ngx.com/)
- [Paperless-ngx – GitHub-repository](https://github.com/paperless-ngx/paperless-ngx)
- [Paperless-ngx – utveckling](https://docs.paperless-ngx.com/development/)
- [Paperless-ngx – användning](https://docs.paperless-ngx.com/usage/)
- [Celery](https://docs.celeryq.dev/en/stable/)
- [Paperless-ngx – Python-projektmanifest](https://github.com/paperless-ngx/paperless-ngx/blob/dev/pyproject.toml)
- [Paperless-ngx – Dockerfile](https://github.com/paperless-ngx/paperless-ngx/blob/dev/Dockerfile)
- [Paperless-ngx – konfiguration](https://docs.paperless-ngx.com/configuration/)
- [Redis-kompatibel broker](https://redis.io/docs/latest/)
- [PostgreSQL](https://www.postgresql.org/docs/current/)
- [Paperless-ngx – installation](https://docs.paperless-ngx.com/setup/)
- [Docker – compose config](https://docs.docker.com/reference/cli/docker/compose/config/)
- [Docker – compose ps](https://docs.docker.com/reference/cli/docker/compose/ps/)
- [Docker – compose images](https://docs.docker.com/reference/cli/docker/compose/images/)
- [Docker – inspect](https://docs.docker.com/reference/cli/docker/inspect/)
- [Paperless-ngx – FAQ](https://docs.paperless-ngx.com/faq/)
- [Paperless-ngx – Consume Hooks](https://docs.paperless-ngx.com/advanced_usage/)
- [OCRmyPDF – dokumentation](https://ocrmypdf.readthedocs.io/en/latest/)
- [Tesseract – dokumentation](https://tesseract-ocr.github.io/tessdoc/)
- [Paperless-ngx – Document Renamer](https://docs.paperless-ngx.com/administration/)
- [Paperless-ngx – Tantivy-migrering](https://docs.paperless-ngx.com/migration-v3/)
- [Docker – compose exec](https://docs.docker.com/reference/cli/docker/compose/exec/)
- [PostgreSQL – Backup and Restore](https://www.postgresql.org/docs/current/backup.html)
- [MariaDB – Backup and Restore](https://mariadb.com/docs/server/server-usage/backing-up-and-restoring-databases/)
- [PostgreSQL – pg_isready](https://www.postgresql.org/docs/current/app-pg-isready.html)
- [Redis – redis-cli](https://redis.io/docs/latest/develop/tools/cli/)
- [Docker – compose logs](https://docs.docker.com/reference/cli/docker/compose/logs/)
- [Microsoft – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [Linux man-pages – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html)
- [curl – handbok](https://curl.se/docs/manpage.html)
- [Microsoft – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [Linux man-pages – getent(1)](https://man7.org/linux/man-pages/man1/getent.1.html)
- [Docker – compose port](https://docs.docker.com/reference/cli/docker/compose/port/)
- [Paperless-ngx – REST-API](https://docs.paperless-ngx.com/api/)
- [RFC 9051 – IMAP4rev2](https://datatracker.ietf.org/doc/html/rfc9051)
- [Apache Tika – serverdrift](https://tika.apache.org/3.2.3/gettingstarted.html)
- [Gotenberg – konfiguration](https://gotenberg.dev/docs/configuration)
- [Docker – stats](https://docs.docker.com/reference/cli/docker/container/stats/)
- [Docker – system df](https://docs.docker.com/reference/cli/docker/system/df/)
- [Microsoft – Get-FileHash](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – sha2 utilities](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Paperless-ngx – Changelog](https://docs.paperless-ngx.com/changelog/)
- [Paperless-ngx – felsökning](https://docs.paperless-ngx.com/troubleshooting/)
