---
title: "Paperless-ngx: dokumentpipeline, OCR og arkivdrift"
blatt: "paperless-ngx"
description: "Paperless-ngx for DMS-, plattform- og meldingsadministratorer: Django-/Angular-arkitektur, consumer- og OCR-pipeline, original- og arkivfiler, metadata, Tantivy-søk, Celery og Redis, PostgreSQL, Tika/Gotenberg, tillatelser, e-posthenting, API, drift, backup, gjenoppretting og teknisk historie."
fakten:
  - label: Systemrolle
    wert: selvhostet dokumenthåndteringssystem for innhenting, OCR, klassifisering, søk, oppbevaring og eksport
    href: https://docs.paperless-ngx.com/
  - label: Teknologistakk
    wert: Python og Django i backend, Angular i nettleseren, Celery-oppgaver, OCRmyPDF/Tesseract og Tantivy-søkeindeks
    href: https://github.com/paperless-ngx/paperless-ngx
  - label: Obligatoriske tjenester
    wert: Web-/oppgaveprosesser, Redis-kompatibel broker, database og persistente Data-/Media-kataloger
    href: https://docs.paperless-ngx.com/configuration/
  - label: Database
    wert: PostgreSQL anbefales for nye installasjoner; MariaDB og SQLite støttes også
    href: https://docs.paperless-ngx.com/setup/
  - label: Dokumentobjekt
    wert: Databasepost pluss original, valgfri arkiv-PDF, miniatyrbilde, fulltekstindeks og kontrollsummer
    href: https://docs.paperless-ngx.com/faq/
  - label: Innganger
    wert: Consume-katalog, nettleser-/API-opplasting og regelbasert IMAP-e-posthenting
    href: https://docs.paperless-ngx.com/usage/
  - label: OCR
    wert: OCRmyPDF orkestrerer rasterisering og Tesseract; modus, språk, sidegrense og arkivformat er driftsparametere
    href: https://docs.paperless-ngx.com/configuration/
  - label: Office og e-post
    wert: valgfri konvertering via Apache Tika og Gotenberg; de to tjenestene utgjør et eget feilområde
    href: https://docs.paperless-ngx.com/configuration/#optional-services
  - label: Søk
    wert: Tantivy-indeks ved siden av den relasjonelle databasen; indeksen kan avledes, databasen og originalene er autoritative
    href: https://docs.paperless-ngx.com/migration-v3/
  - label: Tillatelser
    wert: globale Django-rettigheter pluss objektbaserte Owner-, View- og Change-rettigheter for brukere og grupper
    href: https://docs.paperless-ngx.com/usage/#permissions
  - label: Kjernen i backupen
    wert: konsistent eksport eller koordinert sikkerhetskopi av database, Media, Data, konfigurasjon og hemmeligheter
    href: https://docs.paperless-ngx.com/administration/#making-backups
  - label: Godkjenningsregel
    wert: Gjenoppretting i en tom, passende instans; pålogging, dokumentantall, stikkprøver, søk, kontrollsummer og en ny Consume-kjøring må fungere
    href: https://docs.paperless-ngx.com/administration/#restoring
werbung:
  - newsletter
ctaThemen:
  - paperless-ngx
translationSourceHash: a172bf8c7ed80cecd7c990d657cfdf7d06afe1c0b349350655549c21ed144dfc
translationModel: gpt-5.6-terra
translatedAt: 2026-10-11T10:34:38.369Z
translationReview: automatic
---

# Paperless-ngx: dokumentpipeline, OCR og arkivdrift

Paperless-ngx er et selvhostet dokumenthåndteringssystem for innkommende filer og skannede papirdokumenter. Det mottar dokumenter fra en overvåket katalog, fra nettet eller REST-API-et eller fra e-postkasser, oppretter søkbar tekst og ved behov et standardisert arkiveksemplar, tilordner metadata og gjør bestandet søkbart via webgrensesnitt og API. [Prosjektoversikten](https://docs.paperless-ngx.com/) beskriver tjenesten som et søkbart nettarkiv; [kildearkivet](https://github.com/paperless-ngx/paperless-ngx) angir den samtidig som fellesskapets etterfølger til Paperless og Paperless-ng.

For administratorer er Paperless-ngx ikke én enkelt webapplikasjon, men en sammensetning av database, broker, web- og bakgrunnsprosesser, filbestand, søkeindeks og eksterne konvertere. En vellykket HTTP-status beviser bare at webprosessen svarer. Den sier ikke om consumeren gjenkjenner nye filer, Celery-oppgaver blir levert, OCR fungerer, arkivfiler blir skrevet, indeksen er oppdatert eller en eksport kan importeres igjen.

Forklaringen følger et dokument fra inngangen via consumer, parser og OCR til database, arkivfil og søkeindeks. Deretter følger automatisering, sikkerhet, drift og en gjenoppretting som kontrollerer alle delene samlet.

Paperless-ngx behandler et dokument som en pipeline: gjenkjenne inngangen, overta filen, opprette OCR og metadata, oppdatere database og søkeindeks og gjøre resultatet tilgjengelig via web eller API. Artikkelen følger denne veien og utleder drift, backup og diagnose av den.

## Arkitektur: kontrollsti, arbeidssti og datasti

Arkitekturen kan deles inn i tre stier:

1. **Kontrollsti:** Nettleseren eller API-klienten kommuniserer via HTTP eller HTTPS med Django-applikasjonen. Angular-frontend er den interaktive klienten; Django tilbyr autentisering, forretningslogikk, administrasjonsgrensesnitt og REST-API. At prosjektet består av en Django-applikasjon og en Angular-frontend, dokumenteres i [utviklingsdokumentasjonen](https://docs.paperless-ngx.com/development/).
2. **Arbeidssti:** Consumer, scheduler og workere oppretter og behandler asynkrone oppgaver. En Redis-kompatibel broker formidler mellom prosesser; Celery utfører blant annet konsum, e-posthenting, indeksvedlikehold og tilbakevendende vedlikehold. [Bruksdokumentasjonen](https://docs.paperless-ngx.com/usage/) beskriver consumer, task processor og broker som separate roller.
3. **Datasti:** Den relasjonelle databasen lagrer dokumentobjekter, metadata, brukere, regler og oppgavestatus. Originaler, arkivversjoner og miniatyrbilder ligger i Media-katalogen; hjelpetilstander og søkedata ligger i Data-katalogen. Søkeindeksen er en projeksjon av dette autoritative bestandet.

Skillet er avgjørende operativt. Databasen og Media-katalogen må tilhøre samme logiske tidspunkt. En ny indeks kan bygges opp, men en tapt originalfil kan ikke rekonstrueres fra indeksen. På samme måte er en databasesikkerhetskopi uten tilhørende filer ikke tilstrekkelig for en DMS-gjenoppretting.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-paperless-ngx.svg?v=20260813" title="Interaktive Infografik: Paperless-ngx von Eingang, Consumer und Task-Broker über Parser, OCR, Archivierung, Metadaten und Suche bis zu Datenbank, Dateien, Backup und Restore" loading="lazy">
  <a href="/images/kb-interaktiv-paperless-ngx.svg?v=20260813">Åpne interaktiv grafikk direkte</a>.
</iframe>

## Teknologistakk og prosessroller

Backendkoden er hovedsakelig implementert i Python og er basert på Django. Nettleserfrontend bruker Angular. Avhengigheter som Django, [Celery](https://docs.celeryq.dev/en/stable/), OCRmyPDF og andre parsere kan spores i [Python-prosjektmanifestet](https://github.com/paperless-ngx/paperless-ngx/blob/dev/pyproject.toml); imagebygging og native OCR-/PDF-avhengigheter vises i [Dockerfile](https://github.com/paperless-ngx/paperless-ngx/blob/dev/Dockerfile). Følgende roller er relevante for drift:

| Rolle | Hovedoppgave | Autoritativ datakilde | Typisk feil |
|---|---|---|---|
| Webserver | UI, REST-API, autentisering, opplasting | Økt og DB-transaksjon | 4xx/5xx, proxy- eller CSRF-feil |
| Consumer | oppdage nye filer og starte oppdrag | Consume-katalog og oppgave | Fil blir liggende, polling/inotify eller rettigheter |
| Task worker | parser, OCR, klassifisering og arkivtrinn | Oppgavestatus, midlertidige filer, DB | Oppgave pending/failed, minne-/CPU-grense |
| Scheduler | e-posthenting og periodisk vedlikehold | Tidsplan i oppgavesystemet | Ingen periodiske jobber |
| Broker | levering av asynkrone oppgaver | flyktige kø-/brokerdata | Web fungerer, behandlingen står |
| Database | dokumenter, metadata, identiteter, regler | relasjonelt primærbestand | Transaksjons-, skjema- eller kapasitetsfeil |
| Fillagring | original, arkiv, miniatyrbilde | Media-filer | Fil mangler, skrivebeskyttet, inkonsistent mount |
| Søkeindeks | fulltekstprojeksjon | avledbare indeksdata | Dokument finnes, søket er ufullstendig |
| Tika/Gotenberg | valgfri Office-/e-postkonvertering | midlertidig konvertert fil | Bare bestemte filtyper feiler |

Den offisielle [konfigurasjonsreferansen](https://docs.paperless-ngx.com/configuration/) omtaler den [Redis-kompatible brokeren](https://redis.io/docs/latest/) som en påkrevd tjeneste. [PostgreSQL](https://www.postgresql.org/docs/current/), MariaDB og SQLite støttes som databaser; for nye installasjoner anbefaler [installasjonsveiledningen](https://docs.paperless-ngx.com/setup/) PostgreSQL. Tika og Gotenberg er valgfrie og kreves sammen for Office-formater og behandling av e-postfiler ([valgfrie tjenester](https://docs.paperless-ngx.com/configuration/#optional-services)).

## Tjenesteinventar og faktisk konfigurasjon

Et robust inventar inneholder image eller release, databasetype, brokermål, mounts, effektive miljøvariabler, UID/GID, reverse proxy, tidssone, OCR-språk og eksterne parsere. Docker Compose er en utbredt pakking, men ikke et bevis på arkitektur: Flere prosessroller kan ligge i samme containerimage, mens tilstandene deres ligger i separate volumer og tjenester.

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

[`docker compose config`](https://docs.docker.com/reference/cli/docker/compose/config/) gjengir den faktisk sammenslåtte Compose-konfigurasjonen; [`docker compose ps`](https://docs.docker.com/reference/cli/docker/compose/ps/) viser prosesser og publiserte porter; [`docker compose images`](https://docs.docker.com/reference/cli/docker/compose/images/) identifiserer images; [`docker inspect`](https://docs.docker.com/reference/cli/docker/inspect/) viser mounts, nettverk og kjøretidsparametere. Utdata kan inneholde hemmeligheter og skal bare legges uredigert i saker etter redigering.

## Innganger og leveringsgrenser

Paperless-ngx kjenner flere innganger som har ulike feilområder før den felles pipelinen:

- **Consume-katalogen** overvåkes via filsystemhendelser. På filsystemer uten pålitelig inotify, som enkelte NFS-mounts, må polling aktiveres; dette nevnes i [oppsettdokumentasjonen](https://docs.paperless-ngx.com/setup/#file-systems-without-inotify-support-eg-nfs).
- **Nettleser og REST-API** skriver opplastinger via webstien. Reverse proxy, grense for forespørselsstørrelse, timeout og midlertidig lagring ligger dermed foran oppgavesystemet.
- **E-posthentingen** leser regelbasert fra konfigurerte kontoer og oppretter oppgaver for matchende vedlegg. IMAP-forbindelse, TLS, OAuth/passord, mappe, søkekriterium og etterbehandlingshandling inngår i leveringskontrakten.

Consume-katalogen er en overleveringssone og ikke et dokumentarkiv. Vellykket konsumerte filer fjernes derfra, mens Paperless oppretter nye administrerte filer i Media-bestandet. [FAQ](https://docs.paperless-ngx.com/faq/) advarer uttrykkelig mot å administrere Media-katalogen manuelt.

Når prosesser og innganger er kjent, begynner selve behandlingen. Hvert trinn oppretter forskjellige filer, databaseoppføringer og feilsignaler.

## Consume-pipeline: fra inngang til dokumentobjekt

Det faglige commit-et skjer ikke når en fil oppdages, men først etter flere trinn. Avhengig av filtype og konfigurasjon omfatter pipelinen vanligvis:

1. **Gjenkjenning og stabilisering:** Filnavn, tillatt type, filstørrelse og fullført skriving kontrolleres.
2. **Opprettelse av oppdrag:** Consumeren eller webserveren overleverer en oppgave via brokeren.
3. **Parsing og forbehandling:** Parsere trekker ut eksisterende tekst eller konverterer inngangsdokumentet. Office- og e-postfiler kan kreve Tika og Gotenberg.
4. **OCR:** OCRmyPDF avgjør tekstgjenkjenning ut fra OCR-modusen; Tesseract gjenkjenner konfigurerte språk.
5. **Arkivversjon og miniatyrbilde:** En visnings-/arkivfil og forhåndsvisningsdata opprettes, så langt filtype og innstillinger tillater det.
6. **Persistens:** Original og avledninger skrives; dokumentobjekt, kontrollsummer, tekst og tekniske egenskaper lagres i databasen.
7. **Metadata og arbeidsflyter:** Regler tilordner tagger, korrespondent, dokumenttype, lagringssti, eier eller andre felt.
8. **Indeksering:** Fulltekstbestandet oppdateres i søkeindeksen.
9. **Avslutning:** Oppgavestatus og logg viser suksess eller feil; først nå er dokumentet faglig søkbart.

Pre- og post-consume-skript kan utvide stien. [Dokumentasjonen for Consume Hooks](https://docs.paperless-ngx.com/advanced_usage/#consume-hooks) tydeliggjør at slike skript kjøres i sikkerhets- og feilkonteksten til Paperless-prosessen. De skal versjoneres og kjøres med timeout og entydige exitkoder; en feilaktig hook kan blokkere hele inntaket.

## OCR, arkivformat og kvalitetsgrenser

OCRmyPDF orkestrerer PDF-behandling og Tesseract-OCR. Modusen avgjør om eksisterende tekst hoppes over, behandles på nytt eller tvinges gjennom. Språk, sidegrense, bildeopprydding, rotasjonskorrigering og utdataformat påvirker CPU-tid, filstørrelse og gjenkjenningskvalitet. Innstillingene beskrives i [Paperless-OCR-konfigurasjonen](https://docs.paperless-ngx.com/configuration/#ocr-settings) og i [OCRmyPDF-dokumentasjonen](https://ocrmypdf.readthedocs.io/en/latest/); språkmodeller og motoratferd dokumenteres av [Tesseract](https://tesseract-ocr.github.io/tessdoc/).

Originalen beholdes som bevisobjekt. En arkiv-PDF er en avledet versjon som kan opprettes på nytt ved senere endringer i OCR eller arkivering. Dette skillet inngår i konsepter for oppbevaring og bevisverdi:

| Objekt | Formål | Gjenopprettbarhet |
|---|---|---|
| Original | uendret inngangsfil | kan ikke rekonstrueres fra OCR-tekst |
| Arkivfil | normalisert, ofte søkbar versjon | kan opprettes på nytt fra original og passende pipeline |
| Miniatyrbilde | UI-forhåndsvisning | avledbar |
| Ekstrahert tekst | søk og klassifisering | kan ekstraheres på nytt, resultatet kan avvike avhengig av versjon |
| Kontrollsum | integritetssammenligning | kun beviskraftig sammen med algoritme og referanse |

OCR-suksess betyr ikke faglig lesbarhet. Stikkprøver bør sammenligne sidetall, orientering, tekstdekning, spesialtegn, tabeller, innebygde signaturer og filstørrelse. CPU- og RAM-topper oppstår ofte ved rendering av store eller bilderike PDF-er, ikke ved den avsluttende databaseskrivingen.

## Originaler, arkiver, miniatyrbilder og stiformat

Et Paperless-dokument består av flere koblede deler. Databasen kjenner intern ID, tittel, opprettelses- og tilleggsdato, korrespondent, dokumenttype, tagger, lagringssti, innhold, eier og andre felt. I Media-katalogen ligger originalen og eventuelt arkiv- og miniatyrbildefiler. Konfigurerbare filnavn og Storage Paths endrer den fysiske plasseringen, ikke identiteten til databaseobjektet.

Administratorer unngår direkte omdøping eller flytting i Media-katalogen. Den offisielle [`document_renamer`](https://docs.paperless-ngx.com/administration/#document-renamer) bruker det konfigurerte formatet konsekvent. Et eksternt filsynkroniseringsverktøy kan sikkerhetskopiere bestanden, men skal ikke omorganisere den på egen hånd ved siden av den kjørende applikasjonen. Se [Storage](/kb/storage) for grunnleggende lagring og [Rclone](/kb/rclone) for kopiering og verifikasjon.

## Metadata, matching og arbeidsflyter

Korrespondenter, dokumenttyper, tagger, lagringsstier, egendefinerte felt og datoverdier utgjør en fasettert struktur. De erstatter ikke et filtre, men beskriver et dokument langs flere uavhengige dimensjoner. Automatisk matching gir forslag eller tilordninger basert på gjenkjent innhold og tidligere data; det er ikke en uforanderlig sannhetstjeneste.

Arbeidsflyter består av triggere, valgfrie filtre og sekvensielle handlinger. De kan sette metadata, tillatelser og andre egenskaper ved konsum eller andre hendelser. [Arbeidsflytdokumentasjonen](https://docs.paperless-ngx.com/usage/#workflows) beskriver triggere og handlinger. For administratorer følger tre regler av dette:

- Hver automatisering trenger et definert inngangssett og et observerbart resultat.
- Regelrekkefølge og flere treff testes som policykode.
- Etter endringer vurderes eksisterende dokumenter på nytt bare kontrollert med [`document_retagger`](https://docs.paperless-ngx.com/administration/#document-retagger).

## Søk: databasespørring og Tantivy-projeksjon

Fulltekstsøket bruker en separat Tantivy-indeks. Databasen forblir autoritativ for dokumentobjekter og metadata; indeksen akselererer søk og rangering. Dermed kan et dokument være synlig i databasen, men mangle i fulltekstsøk. Migreringen til Tantivy og endrede søkefelt dokumenteres i [migreringsveiledningen](https://docs.paperless-ngx.com/migration-v3/#search-index-whoosh-tantivy).

Etter oppgraderinger eller ved mistanke om drift kan indeksen bygges på nytt via administrasjonskommandoen. Reindeksering reparerer en projeksjon, ikke en gjenoppretting av originaler. Kapasitetsplanlegging tar hensyn til indeksstørrelse, midlertidig plass under ombygging og I/O-belastning parallelt med normalt konsum.

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

Administrasjonskommandoer startes med Compose via [`docker compose exec`](https://docs.docker.com/reference/cli/docker/compose/exec/) i webservercontaineren. Bare-metal-installasjoner bruker `python3 manage.py`; den støttede kallformen og innebygde hjelpene dokumenteres i [administrasjonen](https://docs.paperless-ngx.com/administration/#management-utilities).

OCR-filer og metadata alene utgjør ennå ikke en konsistent tjeneste. Database, oppgavebroker og søkeprojeksjon må være på samme behandlingsnivå.

## Database, broker og konsistens

PostgreSQL er standardvalget for nye installasjoner. MariaDB og SQLite er mulige, men skiller seg i backupverktøy, parallellatferd og driftsgrenser. Databasemigreringer inngår i en Paperless-oppgradering; en tilbakekopiert gammel binærfil passer ikke nødvendigvis til et allerede migrert skjema. Databaseoperasjon behandles derfor som en egen livssyklus ([PostgreSQL-backup og gjenoppretting](https://www.postgresql.org/docs/current/backup.html), [MariaDB Backup and Restore](https://mariadb.com/docs/server/server-usage/backing-up-and-restoring-databases/)).

Brokeren transporterer oppgaver mellom prosesser. Redis-/Valkey-persistens erstatter ikke sikkerhetskopi av database og dokumenter. Etter brokertap kan pågående eller leverte arbeider startes på nytt, gå tapt eller bli synlige som inkonsistent oppgavestatus. Godkjenningen skjer derfor på dokumentobjektet og filene dets, ikke på en tom kø.

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

[`pg_isready`](https://www.postgresql.org/docs/current/app-pg-isready.html) kontrollerer om PostgreSQL-forbindelser aksepteres, ikke skjema eller faglige spørringer. [`redis-cli`](https://redis.io/docs/latest/develop/tools/cli/) tester brokertilgjengelighet med `PING`. [`docker compose logs`](https://docs.docker.com/reference/cli/docker/compose/logs/) samler tidsavgrensede tjenestelogger; legitimasjon, dokumenttitler og stier redigeres før videreformidling.

## Konfigurasjon, Secret Key og stioppløsning

Med Docker leveres innstillinger vanligvis som miljøvariabler eller Compose-env-fil; `paperless.conf` gjelder ikke der. Bare-metal-installasjoner søker etter konfigurasjon i en dokumentert rekkefølge. Enkelte innstillinger kan dessuten lagres i applikasjonen og overstyre miljøverdier. Den effektive konfigurasjonen er derfor mer enn én enkelt fil ([konfigurasjonsrekkefølge](https://docs.paperless-ngx.com/configuration/#configuration)).

`PAPERLESS_SECRET_KEY` signerer økter og sensitive verdier og er obligatorisk. `PAPERLESS_URL`, Allowed Hosts, CSRF- og CORS-innstillinger definerer webgrensen. Database-, e-post-, OAuth-, Tika- og valgfrie AI-legitimasjoner er hemmeligheter. Docker-oppsett kan bruke `_FILE`-varianten for en rekke verdier og dermed mountede hemmeligheter; dette dokumenteres i [oppsettsveiledningen](https://docs.paperless-ngx.com/setup/#docker-compose-install).

Konfigurasjonsbackup omfatter minst:

- Compose-/tjenestedefinisjon og festet image-referanse,
- miljøvariabler uten klartekstkopier i ubeskyttede repositories,
- Secret Key og eksterne hemmelighetsreferanser,
- UID/GID, volum- og bind-mount-tilordning,
- proxy-, DNS-, sertifikat- og e-postkontokonfigurasjon,
- OCR-språk, filnavnformat, arbeidsflyter og egendefinerte hooks.

## Webgrense, TLS og reverse proxy

Applikasjonen bør være tilgjengelig via HTTPS bak en riktig konfigurert reverse proxy. Host-header, opprinnelses-URL, CSRF Trusted Origins, Forwarded Headers og opplastingsgrenser må passe sammen. Proxyen må bare sette Remote-User-headere når den pålitelig autentiserer identiteten foran og fjerner klientheadere med samme navn.

[Sikkerhetsdokumentasjonen](https://docs.paperless-ngx.com/configuration/#hosting-security) beskriver Secret Key, URL-, host-, CORS- og CSRF-parametere. Sertifikat- og transportgrunnlag finnes under [TLS](/kb/tls), systematisk sikring under [herding](/kb/haertung). Prosjektet advarer i [repositoryet](https://github.com/paperless-ngx/paperless-ngx#important-note) om at dokumenter lagres i klartekst og at Paperless-ngx ikke bør driftes på en vert som ikke er til å stole på.

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

[`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) kontrollerer TCP-tilgjengelighet i Windows; [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) viser lokale lyttere; [`curl`](https://curl.se/docs/manpage.html) viser status, omdirigering og headere; [`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) eller [`getent`](https://man7.org/linux/man-pages/man1/getent.1.html) kontrollerer navneoppløsning; [`docker compose port`](https://docs.docker.com/reference/cli/docker/compose/port/) viser den faktiske porttilordningen.

## Identiteter, globale rettigheter og objektrettigheter

Paperless-ngx bruker Django-modellen for brukere og grupper. Globale tillatelser styrer hvilke objekttyper og UI-/API-områder en konto kan bruke. Objektbaserte rettigheter styrer Owner, View og Change for et bestemt dokument, en tagg eller et annet objekt. [Tillatelsesdokumentasjonen](https://docs.paperless-ngx.com/usage/#permissions) fremhever at rettighetene til en tagg ikke automatisk overføres til dokumenter som er merket med den.

Superbrukere kan se alle objekter. Staff-/adminstatus må skilles fra dette og gir blant annet tilgang til backend- og systemvisninger. I den daglige driften brukes vanlige kontoer og grupper; tjenestekontoer får kun nødvendige API- og objektrettigheter. Arbeidsflyter kan sette eier og rettigheter ved konsum, noe som er spesielt viktig for eierløse dokumenter fra Consume-kataloger.

Autentisering kan være lokal, via reverse-proxy-Remote-User eller gjennom konfigurerte Social-/OIDC-prosedyrer. Remote User aktiveres bare bak en lukket proxygrense. MFA-, økt- og tokenlivssyklus inngår i det generelle identitetskonseptet.

## REST-API og automatiseringsgrense

REST-API-et er tilgjengelig under applikasjonsprefikset; en interaktiv skjemavisning finnes under `/api/schema/view/`. Blant annet støttes Basic-, Session-, Token-, Remote-User- og Headless-OIDC-autentisering. [API-dokumentasjonen](https://docs.paperless-ngx.com/api/) beskriver søk, objekttillatelser, bulkoperasjoner og versjonsheadere.

Et API-token er en hemmelighet og tildeles separat for hver automatisering. Klienter tar hensyn til paginering, filtre, tillatelser, API-versjon og idempotent gjentakelse. Et vellykket opplastingssvar betyr ennå ikke at OCR er fullført; automatiseringer korrelerer dokument- eller oppgave-ID til den faglige sluttilstanden. Prinsipper for kontrakt, autentisering og retry-atferd finnes under [API-er](/kb/apis).

## E-posthenting som meldingsgrensesnitt

E-postkontoer og e-postregler kobler Paperless-ngx til en IMAP-postkasse. Scheduler-en kontrollerer som standard konfigurerte kontoer periodisk, anvender regler og overleverer matchende vedlegg til Consume-pipelinen. Administrasjonskommandoen `mail_fetcher` kan utløse hentingen manuelt; dette dokumenteres i [administrasjonen for e-posthenting](https://docs.paperless-ngx.com/administration/#fetching-e-mail).

For meldingsadministratorer inngår følgende verdier i runbooken:

- Server-FQDN, port og TLS-modus,
- autentiseringsmetode, utløp av hemmelighet/token og tjenestekonto,
- IMAP-mappe, søkekriterium, filtyper og maksimal meldingsstørrelse,
- handling etter vellykket eller mislykket import,
- håndtering av flere leverte meldinger og like vedlegg,
- korrelasjon mellom Message-ID, Paperless-oppgave og dokument-ID.

IMAP-semantikk defineres av [RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051). En vellykket IMAP-henting beviser bare inngangen til oppgavestien; fullføring av OCR og arkivering overvåkes separat.

## Office-dokumenter, Tika og Gotenberg

Apache Tika trekker ut innhold og metadata fra mange filtyper; Gotenberg tilbyr konverteringstjenester. Paperless krever begge når valgfri Office-/e-postbehandling er aktivert. Endepunktene deres konfigureres separat ([Tika Server](https://tika.apache.org/3.2.3/gettingstarted.html#Using_Tika_as_a_service), [Gotenberg-konfigurasjon](https://gotenberg.dev/docs/configuration)).

De to tjenestene øker angrepsflaten, ressursbehovet og feilrommet. De gjøres bare tilgjengelige internt, får CPU-/RAM-/timeoutgrenser og publiseres ikke som generelt konverterings-API. Hvis PDF-er fungerer, men DOCX eller EML konsekvent feiler, starter diagnosen ved denne sidestien og ikke ved Tantivy eller databasen.

## Container-, bare-metal- og lagringsdrift

De offisielle metodene er installasjonsskript, Docker Compose og Bare Metal. Compose gir reproduserbare tjenestedefinisjoner og separate volumer; Bare Metal gir mer kontroll over prosesser, pakker og systemd, men krever eksplisitt livssyklusadministrasjon. [Oppsettsmatrisen](https://docs.paperless-ngx.com/setup/#overview) klassifiserer metodene etter innsats og bruk.

Persistente kataloger må ikke forveksles med containerens livssyklus. Med Compose løses Media, Data, Consume, Export og databasestien bevisst. Fjern- eller nettverksfilsystemer kontrolleres for POSIX-atferd, atomisk omdøping, låsing, latens, inotify og tilgjengelighet. Et kort avbrudd kan ellers se ut som permanent dokumenttap. Grunnleggende om mounts finnes under [Container](/kb/container) og [Storage](/kb/storage).

Ved en feil fastslås først hvilken pipelinefase dokumentet befinner seg i. Deretter kontrolleres kø, worker, ekstern konverterer, database og fillagring målrettet.

## Observerbarhet og oppgavediagnose

En fornuftig driftsoversikt skiller mellom fire signaler:

- **Tjenestehelse:** Prosessen kjører, porten lytter, database og broker er tilgjengelige.
- **Gjennomstrømning:** Nye oppgaver, fullførte dokumenter, kjøretider og kødybde.
- **Feil:** Oppgaver etter trinn og filtype, parser-/OCR-exit, e-post- og lagringsfeil.
- **Bestandsintegritet:** Manglende eller skadede originaler/arkiver/miniatyrbilder, foreldreløse filer, indeksdrift.

Den innebygde Sanity Checker oppdager blant annet manglende, ulesbare eller filer som er endret i forhold til lagret kontrollsum, samt foreldreløse Media-filer. Den fullstendige listen finnes i [Sanity Checker-dokumentasjonen](https://docs.paperless-ngx.com/administration/#sanity-checker). En alarm på antall feil uten dokument-/oppgave-ID og pipelinefase er knapt handlingsdyktig.

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

[`docker stats`](https://docs.docker.com/reference/cli/docker/container/stats/) viser øyeblikkelig CPU-/minnebruk, [`docker system df`](https://docs.docker.com/reference/cli/docker/system/df/) viser Docker-diskbruken. Begge er øyeblikksbilder. For årsaker korreleres tidsserie, oppgave-ID og Paperless-logger. Generell metodikk finnes under [Troubleshooting](/kb/troubleshooting).

## Kapasitet, parallellitet og grenseverdier

Kapasitetsplanlegging starter med sider per dag, inngangsformater, gjennomsnittlig og maksimal filstørrelse, OCR-språk, parallelle oppgaver, oppbevaringstid og søkevolum. Et dokument kan samtidig opprette original, arkiv-PDF, miniatyrbilde, databasepost, fulltekstindeks og midlertidige renderfiler. Ledig plass må også ta høyde for reindeksering, eksport og oppgradering.

Mer workerparallellitet akselererer CPU-bundet OCR bare til CPU, RAM og I/O er mettet. Samtidig øker den antallet midlertidige filer og databaseforbindelser. E-posthenting og consumer kan opprette flere oppgaver enn workere behandler; derfor observeres køalder, ikke bare øyeblikkelig kølengde. Grenser testes med representative dokumenter, ikke små tekst-PDF-er.

Sikkerhetskopiomfanget følger datamodellen: Originaler, arkiver, database, konfigurasjon, Secret Key og eventuelt tilleggstjenester må kunne gjenopprettes samlet.

## Backup: Hvilke objekter som sikkerhetskopieres sammen

Den offisielle [backupdokumentasjonen](https://docs.paperless-ngx.com/administration/#making-backups) nevner to hovedmetoder: Document Exporter eller koordinert sikkerhetskopi av volumer/kataloger og database. Exporter-en skriver dokumenter, miniatyrbilder, metadata og databaseinnhold til en transportabel katalog. API-tokens overtas ikke; eksport og import bør være versjonskompatible.

Et fullstendig gjenopprettingssett inneholder:

| Objekt | Hvorfor nødvendig | Konsistensregel |
|---|---|---|
| Database | ID-er, metadata, brukere, regler, oppgavereferanser | må passe til Media-snapshot |
| Media | originaler, arkiver, miniatyrbilder | ingen halvferdige skrivinger |
| Data | indeks og hjelpetilstander | delvis avledbar, men nyttig for rask gjenoppstart |
| Konfigurasjon | stier, URL, OCR, e-post, proxy, database | samme navne- og mountkontrakt |
| Hemmeligheter | Secret Key, DB-/e-post-/OIDC-legitimasjon | beskyttet, separat og klar for gjenoppretting |
| Eksportmanifest | portabel objekt- og filtilordning | beholdes sammen med eksporterte filer |
| Runbooks | rekkefølge, versjon, godkjenningstester | tilgjengelig offline |

Exporter-en kan oppdatere eksisterende eksporter og sammenligne kontrollsummer valgfritt. Dette sparer tid, men erstatter ikke en separat, versjonert kopi: Med `--delete` kan slettede dokumenter fjernes fra eksportmålet. Detaljer og alternativer dokumenteres av [`document_exporter`](https://docs.paperless-ngx.com/administration/#document-exporter).

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

[`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash) og [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) sikrer overføringsbeviset for manifestet. En kontrollsum beviser bare likhet mellom to byte-sekvenser, ikke importerbarhet eller faglig fullstendighet. Grunnleggende om backup finnes under [Backup og disaster recovery](/kb/backup-dr).

## Gjenoppretting og godkjenning

[`document_importer`](https://docs.paperless-ngx.com/administration/#document-importer) forventer en tom installasjon eller, med `--data-only`, en tom database. En gjenopprettingstest bruker derfor isolerte målstier, en Paperless-versjon som passer sikkerhetskopien og ingen produksjonsvolumer.

Godkjenningen omfatter minst:

1. Legg inn konfigurasjon og hemmeligheter i målmiljøet.
2. Klargjør tom database og tomme Media-/Data-stier.
3. Importer eksporten eller gjenopprett database og filer konsistent.
4. Utfør migreringer og eventuelt indeksbygging kontrollert.
5. Kontroller brukerinnlogging, grupper og tillatelser.
6. Sammenlign dokumentantall, metadata og stikkprøver fra original og arkiv.
7. Test fulltekstsøk og visning av miniatyrbilder.
8. Kjør Sanity Checker uten uforklarte funn.
9. Konsumer et nytt testdokument gjennom den reelle inngangen.
10. Dokumenter RPO og RTO med tidsstempler.

En gjenoppretting anses først som vellykket etter denne ende-til-ende-testen. «Containeren starter» er ikke et godkjenningskriterium.

## Oppdateringer, databasemigreringer og rollbackgrense

Før en oppdatering fullføres aktive Consume-oppgaver, en sikkerhetskopi opprettes og release notes gjennomgås for breaking changes. [Oppdateringsveiledningen](https://docs.paperless-ngx.com/administration/#updating-paperless) skiller mellom Docker- og bare-metal-prosedyrer og beskriver databasemigrering og behovsbasert reindeksering. Et image festes til en utgitt serie eller en digest; `latest` er ikke et reproduserbart rollbackpunkt.

Etter en databasemigrering er rollback normalt en gjenoppretting av database og tilhørende filbestand, ikke bare oppstart av det gamle imaget. Nye arkiver eller metadata som er opprettet etter sikkerhetskopieringspunktet, må behandles faglig før tilbakespranget. Generelle prinsipper for release og endring finnes under [Releases](/kb/releases) og [Migration](/kb/migration).

[Endringsloggen](https://docs.paperless-ngx.com/changelog/) er den tidsavhengige kilden for nye funksjoner, breaking changes og rettelser. Denne artikkelen beskriver derimot stabile arkitektur- og driftsprinsipper.

## Feilbilder etter pipelinefase

Fordi et dokument går gjennom flere behandlingstrinn, viser det synlige symptomet ofte bare den siste vellykkede fasen. Tabellen leder fra symptomet til neste robuste bevis.

| Symptom | Sannsynlig grense | Første bevis |
|---|---|---|
| Fil blir liggende i Consume-katalogen | Watcher, polling, UID/GID, fullført skriving | Mount, rettigheter, consumerlogg |
| Opplasting svarer, men det opprettes ikke dokument | Broker eller task worker | Oppgave-ID, broker-PING, workerlogg |
| Bare DOCX/EML feiler | Tika/Gotenberg | Endepunkt, timeout, konvertererlogg |
| OCR er svært langsom eller prosessen avsluttes | CPU/RAM, sidetall, språkpakker | Containerstatistikk, kernel-/runtimehendelse, sidestørrelse |
| Dokument synlig, fulltekst mangler | Parser/OCR eller søkeindeks | ekstrahert tekst, oppgavelogg, reindekseringsstatus |
| Miniatyrbilde mangler, original finnes | avledet fil | Sanity Checker, miniatyrbildekommando |
| Innlogging/POST bak proxy feiler | URL, host, CSRF, Forwarded Header | HTTP-headere, effektiv konfigurasjon |
| Periodisk e-posthenting står | Scheduler, broker, e-postautentisering | Schedulerlogg, `mail_fetcher`, IMAP-test |
| Filer er borte etter omstart | feil eller flyktig mount | `docker inspect`, voluminventar |
| Gjenoppretting importerer ikke | versjons-, sti- eller tomhetsbetingelse | eksportmanifest, målversjon, importlogg |

Den offisielle [Troubleshooting-samlingen](https://docs.paperless-ngx.com/troubleshooting/) supplerer med konkrete feilmeldinger. Administratorer lagrer alltid oppgave-ID, dokumentnavn, tidsvindu, effektiv konfigurasjon, container-/prosessstatus og tilstanden til alle avhengige tjenester før endringer.

## Teknisk historie

Det opprinnelige Paperless-prosjektet etablerte grunnmodellen med Consume-katalog, OCR og søkbart arkiv. Paperless-ng fortsatte utviklingen som en fork. Paperless-ngx oppstod deretter som den offisielle fellesskapsetterfølgeren for å fordele vedlikehold og ansvar mellom flere bidragsytere; denne avstamningen beskrives i [prosjektets README](https://github.com/paperless-ngx/paperless-ngx#paperless-ngx).

Plattformen utviklet seg fra en primært serversidebasert Django-applikasjon til et DMS med Angular-frontend, REST-API, objektbaserte tillatelser, arbeidsflytmotor og parallell oppgavebehandling. OCRmyPDF/Tesseract forble OCR-grunnlaget; Tika og Gotenberg utvidet de støttede inngangsformatene. Søkestien gikk fra Whoosh til Tantivy, noe som gjorde indeksen synlig som et eget migrerings- og driftsdomene ([Tantivy-migrering](https://docs.paperless-ngx.com/migration-v3/)).

Denne historien forklarer dagens arkitektur: Et Paperless-ngx-dokument er samtidig et relasjonelt objekt, et sett med administrerte filer, et OCR-resultat og en søkeprojeksjon. Vedlikehold og gjenoppretting må ta hensyn til alle fire perspektivene.

## Sjekkliste for administratorer

Til slutt kontrolleres innganger, workere, lagring, søk og backup samlet. Sjekklisten bidrar til at ingen ekstern avhengighet utenfor Paperless-containeren overses.

- **Inventar:** Image/commit, Django-/Angular-applikasjon, worker, scheduler, broker, database, Tika/Gotenberg, proxy og mounts.
- **Dataobjekter:** Navngi database, original, arkiv, miniatyrbilde, indeks, konfigurasjon, hemmeligheter og eksportmanifest separat.
- **Innganger:** Dokumenter Consume, opplasting/API og e-post hver med regel for størrelse, type, autentisering og feil.
- **Pipeline:** Korreler oppgave-ID via parser, OCR, arkivering, metadata og indeks til fullføring.
- **Tillatelser:** Test globale og objektbaserte rettigheter, superbrukere, grupper og arbeidsflyttilordninger.
- **Kapasitet:** Overvåk køalder, OCR-kjøretid, midlertidig plass, Media-/indeksvekst og databaseforbindelser.
- **Sikkerhet:** Ta med HTTPS, proxytiltro, Secret Key, tokens, klartekstlagring og eksterne parsere i trusselmodellen.
- **Backup:** Sikkerhetskopier database og Media konsistent; ikke glem konfigurasjon og hemmeligheter.
- **Gjenoppretting:** Kontroller tom målforekomst, passende versjon, stikkprøver, søk, Sanity Checker og ny Consume-kjøring.
- **Endring:** Dokumenter release notes, skjema-/indeksmigrering, ledig plass og reell rollbackgrense på forhånd.

## Kilder

- [Paperless-ngx – dokumentasjon](https://docs.paperless-ngx.com/)
- [Paperless-ngx – GitHub-repository](https://github.com/paperless-ngx/paperless-ngx)
- [Paperless-ngx – utvikling](https://docs.paperless-ngx.com/development/)
- [Paperless-ngx – bruk](https://docs.paperless-ngx.com/usage/)
- [Celery](https://docs.celeryq.dev/en/stable/)
- [Paperless-ngx – Python-prosjektmanifest](https://github.com/paperless-ngx/paperless-ngx/blob/dev/pyproject.toml)
- [Paperless-ngx – Dockerfile](https://github.com/paperless-ngx/paperless-ngx/blob/dev/Dockerfile)
- [Paperless-ngx – konfigurasjon](https://docs.paperless-ngx.com/configuration/)
- [Redis-kompatibel broker](https://redis.io/docs/latest/)
- [PostgreSQL](https://www.postgresql.org/docs/current/)
- [Paperless-ngx – installasjon](https://docs.paperless-ngx.com/setup/)
- [Docker – compose config](https://docs.docker.com/reference/cli/docker/compose/config/)
- [Docker – compose ps](https://docs.docker.com/reference/cli/docker/compose/ps/)
- [Docker – compose images](https://docs.docker.com/reference/cli/docker/compose/images/)
- [Docker – inspect](https://docs.docker.com/reference/cli/docker/inspect/)
- [Paperless-ngx – FAQ](https://docs.paperless-ngx.com/faq/)
- [Paperless-ngx – Consume Hooks](https://docs.paperless-ngx.com/advanced_usage/)
- [OCRmyPDF – dokumentasjon](https://ocrmypdf.readthedocs.io/en/latest/)
- [Tesseract – dokumentasjon](https://tesseract-ocr.github.io/tessdoc/)
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
- [curl – håndbok](https://curl.se/docs/manpage.html)
- [Microsoft – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [Linux man-pages – getent(1)](https://man7.org/linux/man-pages/man1/getent.1.html)
- [Docker – compose port](https://docs.docker.com/reference/cli/docker/compose/port/)
- [Paperless-ngx – REST-API](https://docs.paperless-ngx.com/api/)
- [RFC 9051 – IMAP4rev2](https://datatracker.ietf.org/doc/html/rfc9051)
- [Apache Tika – serverdrift](https://tika.apache.org/3.2.3/gettingstarted.html)
- [Gotenberg – konfigurasjon](https://gotenberg.dev/docs/configuration)
- [Docker – stats](https://docs.docker.com/reference/cli/docker/container/stats/)
- [Docker – system df](https://docs.docker.com/reference/cli/docker/system/df/)
- [Microsoft – Get-FileHash](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – sha2-verktøy](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Paperless-ngx – endringslogg](https://docs.paperless-ngx.com/changelog/)
- [Paperless-ngx – Troubleshooting](https://docs.paperless-ngx.com/troubleshooting/)
