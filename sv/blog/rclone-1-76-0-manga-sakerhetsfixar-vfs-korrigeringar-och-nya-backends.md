---
title: "Rclone 1.76.0: många säkerhetsfixar, VFS-korrigeringar och nya backends"
navTitle: "Rclone 1.76.0"
description: "Rclone 1.76.0 från den 6 november 2026 stänger säkerhetsluckor i serve, rc och flera backends, däribland en Auth-Proxy-lucka i serve sftp med CVSS 8.1, korrigerar fem fel i VFS-lagret och blockerar HTTPS-till-HTTP-omdirigeringar som standard. Operatörer av mountar, serve-tjänster och rc bör uppdatera och kontrollera de ändrade standardinställningarna."
date: "2026-11-11"
kategorie: "Rclone"
timeToRead: "7 min lästid"
themen:
  - rclone
produkte:
  - "rclone"
protokolle:
  - "releases"
  - "storage"
slug: "rclone-1-76-0-manga-sakerhetsfixar-vfs-korrigeringar-och-nya-backends"
translationId: article-aa7be1fdff77b8cc
translationOf: rclone-1-76-0
url: https://rafaelpfister.ch/sv/blog/rclone-1-76-0-manga-sakerhetsfixar-vfs-korrigeringar-och-nya-backends
translationSourceHash: 79bd08de9bbf74eb195b3bdc4093b8e92f84cd358eb2a4b0a950ad3109a08e56
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:31:05.401Z
translationReview: automatic
---

# Rclone 1.76.0: många säkerhetsfixar, VFS-korrigeringar och nya backends

Rclone 1.76.0 släpptes den 6 november 2026, omkring fyra veckor efter 1.75.2 (9 oktober 2026). Changeloggen innehåller tjugo poster i avsnittet Security, däribland en lucka i Auth-Proxy för `rclone serve sftp` med CVSS 8.1, läckor av autentiseringsuppgifter vid omdirigeringar i backends Swift, NetStorage, Seafile och Ulozto samt flera symlink-utbrott i backend `local`. Därtill kommer fem korrigeringar i VFS-lagret, som används av `rclone mount` och serve-kommandona, utökningar av S3-backend, två nya backends (Dosya och Infomaniak kDrive) samt det nya kommandot `index`. Vissa standardinställningar ändras: HTTPS-till-HTTP-omdirigeringar avvisas, Multipath-TCP-lyssnare är inaktiverade och glob-mönster i filter utvärderas striktare. Versionen riktar sig till alla som använder Rclone för mountar, synkronisering, säkerhetskopior eller som server med `rclone serve` och `rclone rcd`. Bakgrund om mountar i containrar finns i artikeln [Använd Rclone-mountar i Docker tillförlitligt](/blog/rclone-mount-in-docker-container).

## Versionerna i översikt

| Version | Datum | Typ | Huvudpunkter |
|---|---|---|---|
| 1.76.0 | 06.11.2026 | Funktionsversion | Säkerhetskorrigeringar i serve sftp (CVSS 8.1), rc, lib/http, local och fyra backends; Go 1.27.3; fem VFS-korrigeringar; omdirigeringsskydd `--allow-https-downgrade`; nya backends Dosya och Infomaniak kDrive; kommandot `index` |

Den tidigare högsta versionen var 1.75.2. Någon patchversion 1.76.x hade inte släppts fram till den 11 november 2026.

## Säkerhetskorrigeringar med advisory

Följande sårbarheter publicerade Rclone-projektet den 6 november 2026 som GitHub Security Advisories. Alla anger 1.76.0 som patchad version och ingen har tilldelats ett CVE-ID (status 11 november 2026, markerat som ”CVE-PENDING” i changeloggen).

- **GHSA-66qv-66mg-9h74, CVSS 8.1 (High):** I `rclone serve sftp` med `--auth-proxy` innehöll cache-nyckeln för Auth-Proxy inte autentiseringstypen. En känd publik nyckel kunde därför återanvändas som lösenord om Auth-Proxy behandlar nyckel- och lösenordsautentisering olika, angripare och offer använder samma användarnamn och samma klient-IP samt cacheposten fortfarande är giltig (standard: 300 sekunders inaktivitet). Samma klient-IP kan exempelvis uppstå bakom NAT, en bastionvärd eller en lastbalanserare. Andra serve-protokoll berörs enligt advisorn inte. Fältet för berörda versioner anger ”>= 1.51.0, <= 1.76.0”, medan fältet för patchad version anger 1.76.0; avgörande är fixposten i changeloggen för 1.76.0.
- **GHSA-vmgx-g9c5-669p, CVSS 4.2:** I `serve sftp`, `serve ftp`, `serve http` och `serve webdav` med `--auth-proxy` fungerade en återkallelse i Auth-Proxy inte så länge ett cachat svar fanns. Från och med 1.76.0 träder en återkallelse i kraft inom fem minuter. `serve s3` berörs inte.
- **GHSA-hwj7-wp9g-wrv9, CVSS 5.3:** Via svarstiden för bcrypt-kontrollen gick det att skilja giltiga från ogiltiga användarnamn vid `--htpasswd` och `--user`. Bland de berörda finns `serve http`, `serve webdav`, `serve restic`, `serve s3`, `rcd` med `--rc-htpasswd` och Metrics-ändpunkten med `--metrics-htpasswd`, versionerna 1.56.0 till 1.75.2. Lösenord exponeras inte.
- **GHSA-f8p4-8g4j-q3m2, CVSS 5.5:** Om rc-servern körs utan HTTP-autentisering kunde en klient med åtkomst till porten tömma processens minne med en stor JSON-begäran till en obehörig metod (denial of service, versionerna 1.46.0 till 1.75.2). 1.76.0 begränsar begärans storlek och avvisar obehöriga anrop innan bodyn läses. Som lösning anger advisorn HTTP-autentisering för rc och en body-gräns i reverse proxy; bindning till localhost separerar enligt advisorn inte lokala användarkonton från varandra.
- **GHSA-m97j-7x65-wh93, CVSS 4.8:** Backends Swift, NetStorage och Seafile vidarebefordrade autentiseringsuppgifter vid omdirigeringar till en annan värd eller från HTTPS till HTTP. Detta kan endast utnyttjas om den konfigurerade ändpunkten själv omdirigerar, exempelvis via en proxy. Som lösning anger advisorn uteslutande `https://`-ändpunkter och ingen proxy som nedgraderar till HTTP.
- **GHSA-3xpx-hjg2-vrjh, CVSS 4.8:** Backend Ulozto skickade med rubrikerna `X-Auth-Token` och `X-User-Token` vid omdirigeringar från nedladdningslänkar. För detta behöver en angripare en nätverksposition från vilken CDN-trafiken kan omdirigeras.
- **GHSA-r5wv-vqqh-2r9c, CVSS 4.0:** Med `--links` och `--metadata` kunde backend `local` skriva `user.*`-attribut utanför målkatalogen via en insmugglad symlink. Fram till uppdateringen rekommenderar advisorn att inte använda båda alternativen tillsammans när tredje parter kan påverka målet.

Dessutom åtgärdar 1.76.0 i backend `local` ytterligare fall utan eget advisory, där `MkdirMetadata`, `Remove`, `Rmdir` och `DirMove` kunde verka via symlänkar utanför rotkatalogen.

## Beroenden och ändrade standardinställningar

Rclone 1.76.0 är byggt med Go 1.27.3. Changeloggen nämner CVE-2026-111943: Go lägger till en spärrlista för komprometterade mellanliggande certifikat i `crypto/x509` (CVSS 6.8 enligt CISA-ADP-bedömningen, berör Go före 1.26.10 och 1.27.0 till före 1.27.3). Dessutom har `golang.org/x/net` uppdaterats till v0.62.0 och `github.com/andybalholm/brotli` till v1.2.7, det senare på grund av CVE-2026-111497; bakgrund om denna sårbarhet finns i artikeln om [CVE-2026-111497 i Brotli-avkodaren](/blog/brotli-cve-2026-111497).

Multipath-TCP-lyssnare är inaktiverade som standard från och med 1.76.0. Changeloggen motiverar detta med CVE-2026-98432; CVE-posten beskriver en use-after-free i TCP-stacken i Linux-kärnan (CVSS 9.8, korrigerad bland annat i 6.1.190, 6.6.159, 6.12.113 och 6.18.56). Korrigeringen levereras av distributionens kärnuppdateringar; Rclone minskar bara sin egen attackyta. Changeloggen nämner ingen flagga för att aktivera funktionen igen.

Ändringarna i standardbeteendet i översikt:

- **Omdirigeringar:** Rclone avvisar omdirigeringar från HTTPS till HTTP. Den nya flaggan `--allow-https-downgrade` tillåter dem igen.
- **Filter:** `[!...]` i glob-mönster är nu en negerad teckenklass, `[^...]` exkluderar `/`. Kontrollera filterfiler och `--include`/`--exclude`-regler som använder sådana klasser före produktionsdrift med `--dry-run`.
- **local:** `--copy-links` följer inte längre symlink-loopar. Utan root-rättigheter anger Rclone nu endast de ägarförhållanden som en vanlig användare får ange. Brist på lagringsutrymme avbryter nu flertrådiga överföringar med ett fatalt fel.
- **Parameterkontroll:** För stora storleksangivelser, NaN och överflödande tidsintervall avvisas, liksom tidsangivelser för `--bwlimit` utan kolon eller med tecken.
- **bisync:** `--max-delete` tar hänsyn till `--track-renames`; värden över 100 avvisas.
- **operations:** `--immutable` påverkar nu också `copyto`, `moveto` och kopiering av enskilda filer.
- **Föråldrat:** `--rc-web-gui` samt alternativet `--no-cleanup` för `serve s3`, som nu är ett alias för det nya `--directory-markers`.

Skript och units som anropar Rclone med fasta parametrar bör köras en gång efter uppdateringen och kontrolleras för felmeddelanden om avvisade värden.

## Mount och VFS-cache

För `rclone mount` nämner changeloggen en korrigering: VFS-instanser frigjordes inte vid misslyckade mountar och vid unmount. I VFS-lagret, som mount och serve-kommandona använder gemensamt, åtgärdar 1.76.0 fem fel:

- En förminskande truncate av en fil som inte var cachad gick förlorad.
- Att byta namn på en katalog medan katalogcachen löpte ut ledde till en krasch.
- Virtuella poster blev kvar i borttagna kataloger.
- Efter ett namnbyte som endast skilde sig i versaler/gemener eller Unicode-normalisering visades poster dubbelt.
- Uppladdningar av en ändrad fil startade inte så länge filen fortfarande lästes.

Den sista punkten berör mountar med VFS-cache där en ändrad fil fortsätter att läsas medan den ska laddas upp. Övervaka i sådana installationer uppladdningskön efter uppdateringen och genomför ett skriv- och lästest, såsom beskrivs i [Testplan för Rclone-mountar](/blog/cloud-mount-testen-dummy-pdfs).

## S3-backend och Proton Drive

I S3-backend buffrar Rclone inte längre uppladdningar med Object Lock om källans MD5-värde är känt. Nytt är backend-kommandot `link` med signerade overrides för response-rubriker. Dessutom tillkommer regionerna Fastly EU Central 1 och Scaleway IT-MIL, flera Huawei OBS-regioner samt leverantörerna Autonomys Auto Drive, Clever Cloud Cellar, Infomaniak Swiss Backup, Neon Object Storage och RelAix.Storage. Flertrådiga uppladdningar beräknar kontrollsummor i läsoperationer på 1 MiB, och efter en restore visas den faktiska Storage Class. Korrigerat har `rmdir` på rotnivå med `--s3-directory-markers` samt presigned URLs för GCS, Linode och Ceph.

För `rclone serve s3` ger 1.76.0 flaggan `--directory-markers` samt ändringar för taggar, ETags, Multipart-ETags och Conditional Writes. Dessutom är det åtgärdat att buckets inte kunde tas bort efter en avbruten Multipart-upload.

Changeloggen för 1.76.0 innehåller ingen post om backend Proton Drive. De senaste korrigeringarna för detta kommer från 1.75.0 (retry-logik, hängningar vid valideringsfel, felaktiga ändringstider) och 1.75.1 (filer som laddats upp med 1.75.0 kunde inte läsas i Proton-apparna). Statusen för Linux-stödet för Proton Drive beskrivs i artikeln [Proton Drive under Linux](/blog/proton-drive-linux-status).

## Fler nyheter

De två nya backends är Dosya (Kaya) och Infomaniak kDrive. Det nya kommandot `index` skriver statiska kataloglistor till en remote och kan vid efterföljande körningar endast indexera ändrade kataloger på nytt. Rclone stöder CRC-32C som hash, `sync` kan styra ordningen på överföringar utifrån mönster, och katalogvyn i `serve http` och `serve webdav` är sorterad och omgjord. GUI:t betraktas enligt changeloggen inte längre som beta. Statistiken räknar inte längre `totalBytes` dubbelt vid kopior på serversidan, och `serve sftp` rapporterar ett fel vid `rmdir` på en fil i stället för att radera filen.

## Rekommenderat tillvägagångssätt

Målversionen är 1.76.0. Uppdateringen är särskilt brådskande på system med `rclone serve sftp --auth-proxy`, med `rclone rcd` eller `--rc` utan HTTP-autentisering, med serve-tjänster skyddade av `--htpasswd` samt vid remotes av typen Swift, NetStorage, Seafile eller Ulozto. Vid installationer från de officiella binärfilerna uppdaterar `rclone selfupdate` den körbara filen och kontrollerar nedladdningen med hash och signatur; med `rclone selfupdate --check` visar Rclone i förväg vilken version som skulle installeras. För installationer från deb- eller rpm-paket nämner dokumentationen `--package deb` respektive `--package rpm`. Aktiva mountar och serve-processer använder den nya versionen först efter en omstart. Kontrollera därefter filterregler med teckenklasser, tidsscheman för `--bwlimit` och installationer som är beroende av HTTP-omdirigeringar. Changeloggen och releasesidan nämner inga kända problem för 1.76.0 (status 11 november 2026).

## Källor

1.  [Rclone: Changelog](https://rclone.org/changelog/): Poster om 1.76.0 (6 november 2026) samt 1.75.0 till 1.75.2 med säkerhetskorrigeringar, ändringar för mount, VFS, S3 och Proton Drive.

2.  [GitHub: rclone v1.76.0](https://github.com/rclone/rclone/releases/tag/v1.76.0): Releasesida med binärfiler och hänvisning till changeloggen.

3.  [GitHub: Security Advisories från rclone](https://github.com/rclone/rclone/security/advisories): Översikt över advisories som publicerades den 6 november 2026.

4.  [GHSA-66qv-66mg-9h74](https://github.com/rclone/rclone/security/advisories/GHSA-66qv-66mg-9h74): serve sftp, Auth-Proxy-cache utan autentiseringstyp, CVSS 8.1.

5.  [GHSA-vmgx-g9c5-669p](https://github.com/rclone/rclone/security/advisories/GHSA-vmgx-g9c5-669p): Återkallelse i Auth-Proxy fungerar inte vid cachat svar, CVSS 4.2.

6.  [GHSA-hwj7-wp9g-wrv9](https://github.com/rclone/rclone/security/advisories/GHSA-hwj7-wp9g-wrv9): Identifiering av användarnamn via bcrypt-timing vid `--htpasswd`, CVSS 5.3.

7.  [GHSA-f8p4-8g4j-q3m2](https://github.com/rclone/rclone/security/advisories/GHSA-f8p4-8g4j-q3m2): Minnesuttömning i rc-servern genom överstora JSON-begäranden, CVSS 5.5.

8.  [GHSA-m97j-7x65-wh93](https://github.com/rclone/rclone/security/advisories/GHSA-m97j-7x65-wh93): Läckor av autentiseringsuppgifter vid omdirigeringar i Swift, NetStorage och Seafile, CVSS 4.8.

9.  [GHSA-3xpx-hjg2-vrjh](https://github.com/rclone/rclone/security/advisories/GHSA-3xpx-hjg2-vrjh): Tokenläcka vid omdirigeringar i backend Ulozto, CVSS 4.8.

10.  [GHSA-r5wv-vqqh-2r9c](https://github.com/rclone/rclone/security/advisories/GHSA-r5wv-vqqh-2r9c): Symlink-utbrott med `--links --metadata` i backend local, CVSS 4.0.

11.  [CVE.org: CVE-2026-111943](https://www.cve.org/CVERecord?id=CVE-2026-111943): Spärrlista för komprometterade mellanliggande certifikat i Go `crypto/x509`, åtgärdat i Go 1.27.3.

12.  [CVE.org: CVE-2026-98432](https://www.cve.org/CVERecord?id=CVE-2026-98432): Use-after-free i TCP-stacken i Linux-kärnan, CVSS 9.8, anledning till de inaktiverade Multipath-TCP-lyssnarna.

13.  [Rclone: rclone selfupdate](https://rclone.org/commands/rclone_selfupdate/): Dokumentation om självuppdatering, `--check` och `--package`.

14.  [CVE-2026-111497: Sårbarhet i Brotli-avkodaren](/blog/brotli-cve-2026-111497): Bakgrund om Brotli-sårbarheten som 1.76.0 hanterar genom beroendeuppdateringen.
