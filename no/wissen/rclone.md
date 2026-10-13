---
title: "Rclone: Remotes, datasammenligning og VFS-drift"
blatt: "rclone"
description: "Rclone for lagrings-, plattform- og meldingsadministratorer: Go-arkitektur og enhetlig backendmodell, remote- og konfigurasjonsmodell, Copy/Sync/Move/Bisync, sammenligning og kontrollsummer, Crypt, VFS-mount og cache, Retry/Pacer, filtre, RC API, logger, sikkerhet, recovery og teknisk historie."
fakten:
  - label: Systemrolle
    wert: CLI- og API-verktøy for kopiering, synkronisering, verifisering, mount og tilgjengeliggjøring mellom fil- og objektlagring
    href: https://rclone.org/
  - label: Teknologistakk
    wert: Åpen kildekode-prosjekt i Go; backender og kommandoer bygges inn i én portabel rclone-binærfil
    href: https://github.com/rclone/rclone
  - label: Adresseringsmodell
    wert: navngitt remote pluss kolon og sti; lokale stier forblir uten remote-prefiks
    href: https://rclone.org/docs/
  - label: Backendkontrakt
    wert: felles Fs/Object-grensesnitt med valgfrie funksjoner for Hash, ModTime, Copy, Move, Polling, Metadata og ListR
    href: https://rclone.org/overview/
  - label: Retning
    wert: copy legger til uten å rydde målet · sync gjør målet lik kilden · move sletter kilden først etter vellykket kopiering
    href: https://rclone.org/commands/rclone_sync/
  - label: Sammenligning
    wert: Størrelse pluss ModTime eller felles hash; flagg og backendfunksjoner endrer fingeravtrykket
    href: https://rclone.org/overview/
  - label: Integritetsbevis
    wert: Overføringskontroll, rclone check, cryptcheck eller nedlastingssammenligning; exitkode og sluttlogg er del av resultatet
    href: https://rclone.org/commands/rclone_check/
  - label: Mountsti
    wert: VFS via FUSE/WinFsp; Directory Cache, Buffer og File Cache ligger mellom applikasjon og remote
    href: https://rclone.org/commands/rclone_mount/
  - label: Kryptering
    wert: Crypt krypterer innhold med NaCl SecretBox og som standard også navn; konfigurasjon samt passord og salt er recoveryobjekter
    href: https://rclone.org/crypt/
  - label: Styring
    wert: Remote Control API og OpenMetrics-endepunkt; tilgjengelighet tilsvarer i praksis shell-rettighetene til rclone-brukeren
    href: https://rclone.org/rc/
  - label: Feilregel
    wert: Low-Level-Retry, Pacer og Command-Retry; suksess først etter sluttstatus og exitkode, ikke etter enkeltstående feilmelding
    href: https://rclone.org/docs/#retries
  - label: Recoveryobjekt
    wert: Binærfil, konfigurasjon, credentials, filtre, jobbdefinisjon, Bisync-arbeidsfiler, VFS-cache, målversjoner og verifikasjonsrapport
    href: https://rclone.org/docs/#configuration-encryption
werbung:
  - newsletter
ctaThemen:
  - rclone
translationSourceHash: bb7131a94d721cf6984869b9d907a4d2089758b80e651c8b03bd86f72841d42e
translationModel: gpt-5.6-terra
translatedAt: 2026-10-12T11:41:14.808Z
translationReview: automatic
---

# Rclone: Remotes, datasammenligning og VFS-drift

Rclone er et åpen kildekode-verktøy for dataflytting for lokale filsystemer, nettverksprotokoller, skylagringsstasjoner og objektlagring. [Rclone-prosjektsiden](https://rclone.org/) beskriver det som et kommandolinjeprogram for å administrere filer på skylagring. Det gir disse systemene et felles grensesnitt for listing, kopiering, synkronisering, verifisering, mount, kryptering og tjenestetilgang. Denne ensartetheten er nyttig, men må ikke forveksles med ensartet lagringssemantikk: Hashes, endringstider, store/små bokstaver, tomme kataloger, metadata, kopier på serversiden og konsistens varierer mellom backender.

For administratorer er rclone ikke en «skyutforsker», men en datapipeline. Korrektheten avhenger av kilde og mål, retning, sammenligningskriterium, filtermengde, slettingsregel, retrytilstand, credentials, backendfunksjoner og endelig verifisering. En prosess kan ha overført 100 prosent av bytene og likevel være faglig feil dersom den bruker feil navneområde, en utdatert listingvisning eller en ufullstendig filtermengde.

Forklaringen følger en rclone-kommando fra kilde og remote via sammenligning, checkers og transfers til backenden. Filtre, Crypt, VFS, automatisering, sikkerhet og verifisering forklares deretter langs samme jobbvei.

Rclone oversetter en enhetlig fil- og katalogmodell til API-ene for svært ulike lagringstjenester. Artikkelen starter derfor med remote og backendfunksjoner, behandler deretter kopieringssemantikk og verifisering og avslutter med mount, sikkerhet og recovery.

## Arkitektur: Kommando, filsystemmodell og backend

Programmet, som er skrevet i Go, kobler sammen flere lag:

1. **Kommando og jobbdefinisjon:** `copy`, `sync`, `move`, `check`, `bisync`, `mount`, `serve` eller et RC-kall definerer retning og operasjon.
2. **Filtre og sammenligning:** Stiregler, størrelse, endringstid, hash, metadata og flagg bestemmer objektmengden som vurderes, og hva som er likt.
3. **Overføringsmotor:** Listere, checkers og overføringsarbeidere planlegger operasjoner, begrenser parallellitet og samler feil.
4. **Fs/Object-grensesnittmodell:** Rclone modellerer kataloger, objekter og valgfrie backendfunksjoner gjennom felles Go-grensesnitt.
5. **Backendadaptere:** S3, SFTP, WebDAV, Drive, OneDrive, lokale filsystemer og andre tjenester oversetter kontrakten til sine API-er.
6. **Transport og leverandør:** HTTP/TLS, SSH/SFTP eller lokal I/O møter rate limits, konsistens, versjonering og leverandørpolicy.

Den offisielle backendoversikten viser eksplisitt hvilke systemer som støtter hasher, ModTime, navn som ikke skiller mellom store og små bokstaver, duplikater, MIME-typer og metadata ([Overview of cloud storage systems](https://rclone.org/overview/)). Kildetreet skiller mellom backender, kommandoer og den felles `fs`-grensesnittmodellen; offisielle builds pakker de valgte adapterne i én binærfil ([rclone source repository](https://github.com/rclone/rclone), [Install from source](https://github.com/rclone/rclone/blob/master/docs/content/install.md#source)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-rclone.svg?v=20260813" title="Interaktive Infografik: rclone-Job von Quelle, Remote und Filter über Listing, Vergleich, Checker, Transfer, Backend und Provider bis zu VFS, Crypt, Logs, Exitcode, Verifikation und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-rclone.svg?v=20260813">Åpne interaktiv grafikk direkte</a>.
</iframe>

## Remote-syntaks og navneområde

En konfigurert remote adresseres som `name:pfad`. Kolonet skiller remote-navnet fra stien; en lokal sti uten remote-prefiks bruker Local-backenden. Connection strings som `:sftp,host=example.net:/data` kan instansiere en backend uten lagret remote-navn. Denne fleksibiliteten øker samtidig risikoen for at en skrivefeil angir en lokal i stedet for ekstern sti ([Rclone documentation – remotes](https://rclone.org/docs/)).

Ved katalogkommandoer overfører Rclone **innholdet** i kildestien, ikke automatisk navnet på kildekatalogen. Semantikken tilsvarer den avsluttende skråstreken man kjenner fra `rsync`. Kilde, mål og forventet målstruktur dokumenteres derfor som konkrete eksempler i runbooken ([rclone copy](https://rclone.org/commands/rclone_copy/)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für rclone-Inventar und Konfigurationspfad">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">rclone version
rclone config file
rclone listremotes --long
rclone backend features archive:
rclone about archive:</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">rclone version
rclone config file
rclone listremotes --long
rclone backend features archive:
rclone about archive:</code></pre>
  </div>
</div>

[`rclone version`](https://rclone.org/commands/rclone_version/) identifiserer binærfil, operativsystem og build; [`rclone config file`](https://rclone.org/commands/rclone_config_file/) viser konfigurasjonen som faktisk er lastet; [`rclone listremotes`](https://rclone.org/commands/rclone_listremotes/) inventariserer remotes; [`rclone backend features`](https://rclone.org/commands/rclone_backend/) leverer funksjonene til den oppløste backenden; [`rclone about`](https://rclone.org/commands/rclone_about/) spør etter kapasitetsinformasjon dersom tjenesten tilbyr den.

## Backendfunksjoner og synlige forskjeller

Det felles grensesnittet kan bare bruke funksjoner som backenden eller en wrapper tilbyr. Viktige forskjeller er:

| Funksjon | Betydning | Konsekvens ved fravær eller avvik |
|---|---|---|
| felles hash | innholdsfingeravtrykk på leverandørsiden | sammenligning faller tilbake på størrelse/ModTime eller nedlasting |
| skrivbar ModTime | kildetid kan settes uten opplasting på nytt | identisk innhold kan overføres på nytt |
| Copy/Move på serversiden | data forblir hos leverandøren | ellers nedlasting/opplasting eller Copy pluss Delete |
| ListR/Fast List | rekursiv listing i færre kall | flere API-forespørsler eller mer klientminne |
| Change Polling | signaliserer eksterne endringer | VFS ser først endringer etter cacheutløp |
| tomme kataloger | ekte katalogobjekt/markør | tomme mapper forsvinner i bucketmodeller |
| Case Sensitivity | `A` og `a` kan skilles | endeløs synkronisering eller navnekollisjon i målet |
| Duplicate Names | flere objekter med samme navn | sti er ikke en entydig identifikator |
| Metadata/MIME | tilleggsattributter kan leses/skrives | ACL-er eller faglige metadata følger ikke automatisk |

Supportnivåene vurderer backender basert på tester og funksjoner, ikke et generelt funksjonsløfte ([Backend Support Tiers](https://rclone.org/tiers/)). Leverandør- og backenddokumentasjon er derfor del av jobbdefinisjonen. En S3-kompatibel tjeneste kan ha andre grenser for multipart, hash eller Server-Side Copy enn en annen S3-leverandør.

## Konfigurasjon, credentials og oppløsningsrekkefølge

`rclone.conf` lagrer remote-definisjoner og ofte OAuth-tokener eller tilgangsdata. «Obscured» passordfelt er ikke sikker kryptering. Hele konfigurasjonen kan krypteres med et passord; for ubetjente jobber kan en ekstern Password Command levere hemmeligheten fra en Secret Store ([Configuration encryption](https://rclone.org/docs/#configuration-encryption), [rclone config encryption](https://rclone.org/commands/rclone_config_encryption/)).

Rclone kan hente backendalternativer fra kommandolinjen, remote-spesifikke miljøvariabler, backendomfattende variabler og konfigurasjonsfilen. Prioritetsrekkefølgen må dokumenteres for tjenester og containere, fordi en arvet variabel kan overskrive den forventede `rclone.conf` ([Environment variables](https://rclone.org/docs/#environment-variables)). Debuglogger viser oppløste alternativer, men kan inneholde sensitive endepunkter eller stier.

Administratorer beskytter minst:

- `rclone.conf` og filrettighetene for denne;
- konfigurasjonspassord eller Password Command;
- OAuth-refresh-tokener, Client Secrets, Access Keys og SSH Private Keys;
- Crypt-passord og valgfritt salt;
- rettigheter for tjenestekonto, bucket og mappe;
- filter- og jobbdefinisjoner som del av change control.

En konfigurasjonsbackup uten dekrypteringshemmeligheten er ikke testet recovery. Omvendt må en kryptert konfigurasjon med svakt passord ikke lagres offentlig.

Når remote, navneområde og tilgangsdata er avklart, bestemmer den valgte kommandoen datavirkningen. Særlig `sync` er ikke en ufarlig kopi, men håndhever én side som ønsket sannhet.

## Copy, Sync og Move: Retning er policy

[`rclone copy`](https://rclone.org/commands/rclone_copy/) kopierer nye eller forskjellige filer til målet og sletter ingen ekstra objekter der. [`rclone sync`](https://rclone.org/commands/rclone_sync/) gjør målet til et bilde av kilden og kan slette målobjekter for å oppnå dette. [`rclone move`](https://rclone.org/commands/rclone_move/) kopierer eller flytter og fjerner først kilden etter en vellykket filoperasjon.

Disse kommandoene er ikke utbyttbare:

| Operasjon | Overskriver mål | Sletter ekstra målobjekter | Sletter kilde | Typisk bruk |
|---|---:|---:|---:|---|
| `copy` | ved objekter vurdert som forskjellige | nei | nei | import, seed, additiv replikering |
| `sync` | ja | ja | nei | autoritativ enveis speiling |
| `move` | ja | nei | etter vellykket overføring | inntak, arkivoverføring |
| `copyto`/`moveto` | enkeltobjekt med eksplisitt målnavn | nei | kun `moveto` | atomisk enkeltfiloperasjon |
| `bisync` | på begge sider | avhengig av beregnet delta | mulig på begge sider | kontrollert toveis replikering |

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für sicheren rclone-Probelauf">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">rclone sync 'D:\Export' 'archive:current' `
  --dry-run --check-first --backup-dir 'archive:history/prechange' `
  --combined '.\sync-plan.txt' --log-level INFO --log-file '.\sync-dry-run.log'
if ($LASTEXITCODE -ne 0) { throw "rclone dry-run failed: $LASTEXITCODE" }
Get-Content '.\sync-plan.txt' | Group-Object { $_[0] } | Select-Object Name,Count</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">rclone sync /srv/export archive:current \
  --dry-run --check-first --backup-dir archive:history/prechange \
  --combined ./sync-plan.txt --log-level INFO --log-file ./sync-dry-run.log
test "$?" -eq 0 || exit "$?"
cut -c1 ./sync-plan.txt | sort | uniq -c</code></pre>
  </div>
</div>

`--dry-run` hindrer endringer, men beviser bare listingene og alternativene som var synlige ved testtidspunktet. `--check-first` skiller sammenlignings- og overføringsfasen. `--backup-dir` flytter overskrevne eller slettede målobjekter til en separat sti, dersom backenden støtter den nødvendige serveroperasjonen ([Global options – backup-dir and retries](https://rclone.org/docs/)). Combined Report klassifiserer like, manglende, ulike og feilaktige stier.

## Hva betyr «identisk»?

Rclone hopper som standard over filer på grunnlag av størrelse og endringstid, eller bruker en felles hash når kommando og backend tillater det. Det nøyaktige fingeravtrykket avhenger av funksjoner og flagg. `--checksum`, `--size-only`, `--ignore-times`, `--update`, `--ignore-size`, metadataalternativer og ModTime-presisjon endrer beslutningen ([Overview – Hash and ModTime](https://rclone.org/overview/)).

En ETag på leverandørsiden er ikke universelt en innholds-MD5. Multipart-opplasting, kryptering eller leverandørimplementasjon kan endre dens betydning. Rclone bruker hashfunksjonen som backenden eksplisitt erklærer, ikke en generell ETag-antakelse.

Mellom Linux som skiller store og små bokstaver og vanligvis Windows som ikke gjør det, kan `report.pdf` og `Report.pdf` ikke avbildes uten tap. Skylagringsstasjoner kan tillate dupliserte navn eller egne objekt-ID-er. En forhåndskontroll av navnekollisjoner og ugyldige tegn er derfor del av enhver migrering.

## Overføringsmotor, parallellitet og Pacer

Overføringsmotoren lister kilde og mål, sammenligner oppføringer, planlegger kontroller og overføringer og utfører backendoperasjoner parallelt. `--checkers` styrer parallelle kontroller, `--transfers` styrer overføringene. `--fast-list` kan redusere API-kall, men trenger mer minne fordi store listings samles ([Rclone documentation – performance options](https://rclone.org/docs/)).

Parallellitet er ikke bare et hastighetsalternativ. Flere arbeidere øker åpne filer, minne, API-forespørsler, multiparttilstander og belastningen på leverandøren. For små objekter dominerer listing-/forespørselslatens, for store objekter båndbredde og multipartkonfigurasjon. Sky-til-sky-kopier kan forbli på serversiden eller gå via rclone-verten; dette kontrolleres i loggen og backendfunksjonene.

Backender implementerer pacing for API-rate limits. Low-Level-Retries håndterer enkeltforespørsler; High-Level-Retries gjentar en mislykket totaloperasjon. Først sluttstatus avgjør om tidligere feil ble kompensert ([Retries](https://rclone.org/docs/#retries)). En scheduler vurderer prosessens exitkode og ikke forekomsten av ordet `ERROR` et sted i loggen.

## Filtre definerer datamengden

Include-, Exclude-, Size-, Age- og Files-from-regler bestemmer hvilke stier som er synlige. Filtre gjelder den angitte roten; en shellglob i kildestien erstatter ikke en rclone-filterregel. Rekkefølgen på Include og Exclude kan være relevant ([Rclone Filtering](https://rclone.org/filtering/)).

Når et filter endres, endres den faglige datamengden. Ved `sync` kan tidligere synlige objekter fremstå som slettet; `--delete-excluded` er særlig farlig. Filterfiler versjoneres, hashes og godkjennes sammen med Dry-Run-rapport og jobbdefinisjon. En sikker test bruker `rclone ls` eller `lsf` med de samme reglene før en skrivende kommando følger.

En exitkode uten overføringsfeil beviser bare kjøringen, ikke fullstendig likhet mellom begge bestandene. `check` sammenligner kilder og mål med funksjonene til den aktuelle backenden.

## Verifisering: Check i stedet for overføringsteller

[`rclone check`](https://rclone.org/commands/rclone_check/) endrer ingen data. Den sammenligner størrelse og felles hasher, kan lage rapporter for treff, forskjeller og manglende stier og med `--download` lese innhold når ingen felles remote-hash finnes. For Crypt finnes [`rclone cryptcheck`](https://rclone.org/commands/rclone_cryptcheck/), som egnet sammenligner klartekstkilde og kryptert mål.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für rclone-Integritätsprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">rclone check 'D:\Export' 'archive:current' `
  --combined '.\check-report.txt' --checkers 8 `
  --log-level INFO --log-file '.\check.log'
$code = $LASTEXITCODE
Get-Content '.\check-report.txt' | Group-Object { $_[0] } | Select-Object Name,Count
if ($code -ne 0) { throw "Integrity check failed: $code" }</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">rclone check /srv/export archive:current \
  --combined ./check-report.txt --checkers 8 \
  --log-level INFO --log-file ./check.log
code=$?
cut -c1 ./check-report.txt | sort | uniq -c
test "$code" -eq 0 || exit "$code"</code></pre>
  </div>
</div>

En hashsammenligning beviser likhet for algoritmen og navneområdet som brukes. Den beviser ikke automatisk ACL-er, retention, leverandørens objektversjoner, katalogmetadata eller faglig lesbarhet. For kritiske arkiver supplerer en restoretest den tekniske kontrollen.

## Crypt: Data- og navnekryptering

Crypt er en wrapperbackend over en annen remote. Filinnhold krypteres med NaCl SecretBox basert på XSalsa20 og Poly1305; filnavn og katalognavn transformeres som standard også ([Crypt backend](https://rclone.org/crypt/)). Leverandøren ser ciphertext, størrelsesmønstre, tilgangstider og antall objekter; Crypt er derfor innholds- og navnebeskyttelse, ikke fullstendig trafikk- eller metadataanonymisering.

Passord og valgfritt salt lagres bare «obscured» i konfigurasjonen, med mindre hele konfigurasjonen er kryptert. Tap av begge recoveryopplysningene gjør dataene uleselige. Før rotasjon eller endring av navnekrypteringen testes en andre, lesende remote mot de samme dataene.

Crypt endrer størrelser og hashmuligheter. En vanlig `check` mellom klartekst og ciphertext kan derfor ikke uten videre forvente samme leverandørhash. `cryptcheck` eller en kontrollert nedlastingssti er riktig kontrollnivå.

Kopieringskommandoer arbeider eksplisitt med objekter. Et VFS-mount må derimot gjenskape løpende filsystemoperasjoner fra dette og trenger derfor cache, skrivemodus og klare konsistensforutsetninger.

## VFS-mount: Objektlagring er ikke et POSIX-filsystem

[`rclone mount`](https://rclone.org/commands/rclone_mount/) presenterer en remote som et filsystem via FUSE eller WinFsp. VFS-laget oversetter filoperasjoner til objektoperasjoner. Ettersom skyobjekter normalt ikke kan endres i midten eller utvides vilkårlig, bufferlagrer eller erstatter Rclone objekter.

Fire cachemoduser representerer ulike kontrakter:

| Modus | Lokal filcache | Skriveatferd | Driftsgrense |
|---|---|---|---|
| `off` | ingen | direkte og begrenset | ingen Seek/Read-Write som ved normal fil; mislykket opplasting er vanskeligere å gjenta |
| `minimal` | kun ved nødvendige Read-Write-tilfeller | delvis bufret | ikke all Open-/Seek-semantikk er mulig |
| `writes` | skrivende filer lokalt | opplasting etter Close/Writeback | lokal cache er ennå ikke remote-commit |
| `full` | Reads og Writes lokalt, delvis sparse | høy filsystemkompatibilitet | overvåk diskbehov, eviction og cachekonsistens |

Directory Cache holder navn og metadata i minnet. Eksterne endringer vises først via polling eller etter utløp eller flush. Buffer gjelder per åpen fil; maksimalt minnebehov øker med `buffer-size × open files`. To prosesser må ikke bruke samme overlappende VFS-cache med aktiv File Caching, fordi dette kan ødelegge data ([VFS file caching](https://rclone.org/commands/rclone_mount/#vfs-file-caching)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für kontrollierten rclone-Mount">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">rclone mount crypt-archive: X: `
  --vfs-cache-mode writes --cache-dir 'D:\rclone-cache\archive' `
  --vfs-cache-max-size 100G --vfs-cache-min-free-space 20G `
  --log-file 'D:\Logs\rclone-mount.log' --log-level INFO</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">rclone mount crypt-archive: /mnt/archive \
  --vfs-cache-mode writes --cache-dir /var/cache/rclone/archive \
  --vfs-cache-max-size 100G --vfs-cache-min-free-space 20G \
  --log-file /var/log/rclone-mount.log --log-level INFO</code></pre>
  </div>
</div>

På Windows leverer [WinFsp](https://winfsp.dev/doc/WinFsp-Tutorial/) filsystembroen; på Linux og andre Unix-systemer bruker Rclone FUSE. En tjeneste venter på nettverk, konfigurasjon og mountberedskap, har en kontrollert unmount-sti og overvåker lokalt cachevolum. Container-mounts krever ytterligere device- og mount-propagation-rettigheter og utvider privilegieflaten ([Rclone mount in Docker](https://rclone.org/install/#docker)).

## Bisync: To sannhetskilder

[`rclone bisync`](https://rclone.org/bisync/) oppdager endringer på begge sider ved hjelp av lagrede listings og overfører deltaer toveis. Dette oppretter vedvarende jobbtilstand utenfor brukerdataene. En tapt eller inkonsistent arbeidskatalog kan kreve en resync-fase.

Bisync har sikkerhetskontroller for tilgjengelighet og uvanlig mange slettinger. Filterendringer endrer den synlige datamengden og tvinger bevisst frem en resync. Dersom applikasjoner endrer innhold uten å endre størrelse og ModTime, kan standarddetektering svikte. Toveis synkronisering brukes derfor bare når konfliktpolicy, klokke-/timestampatferd, Access Checks, Max-Delete, backup av arbeidsfiler og menneskelig løsning er dokumentert.

Bisync er ikke et multi-master-filsystem med distribuert låsing. Samtidige redigeringer, omdøpinger og offlinefaser forblir faglige konflikter.

## Remote Control API, Metrics og Serve-moduser

Med `--rc` eller [`rclone rcd`](https://rclone.org/commands/rclone_rcd/) åpner Rclone et HTTP-API. Det kan starte jobber, konfigurere remotes, lese eller skrive lokale filer og under visse omstendigheter utføre OS-kommandoer i konteksten til rclone-brukeren. Dokumentasjonen vurderer derfor tilgang til RC som ekvivalent med shelltilgang ([Remote Control / API](https://rclone.org/rc/)).

RC lytter som standard lokalt. Ved nettverksdeling kreves sterk autentisering, TLS eller en beskyttet reverse proxy, brannmurallowlist og minimale prosessrettigheter. `--rc-no-auth` hører ikke hjemme på et tilgjengelig grensesnitt. OpenMetrics-endepunktet kan eksponere overførings- og jobbmetrikker; også her kan remote-navn og stier være sensitive metadata.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für rclone-RC-Status und Metriken">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">rclone rc core/version
rclone rc core/stats
$metrics = Invoke-WebRequest 'http://127.0.0.1:5572/metrics'
$metrics.StatusCode
$metrics.Content | Select-String 'rclone_' | Select-Object -First 20</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">rclone rc core/version
rclone rc core/stats
curl --silent --show-error --fail http://127.0.0.1:5572/metrics | grep '^rclone_' | head -n 20</code></pre>
  </div>
</div>

[`Invoke-WebRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest) og [`curl`](https://curl.se/docs/manpage.html) kontrollerer det lokale HTTP-endepunktet. [`rclone rc`](https://rclone.org/commands/rclone_rc/) bruker samme API og overtar autentiseringsparametere fra jobbkonfigurasjonen.

Rclone kan dessuten tilby remotes via HTTP, WebDAV, SFTP, FTP, NFS, DLNA eller S3-kompatible Serve-moduser. Systemrollen skifter da fra klient til server. Lytteradresse, autentisering, protokollsikkerhet, VFS-cache og datarettigheter herdes deretter som for en selvstendig tjeneste ([Rclone serve](https://rclone.org/commands/rclone_serve/)).

## Logging, statistikk og exitkoder

Logger skiller mellom DEBUG, INFO, NOTICE og ERROR. JSON-logging er tilgjengelig for maskinbehandling. Statistikk viser checks, transfers, feil, byte, hastighet og ETA. En kortvarig `Transferred`-teller er ikke et varig revisjonsbevis; en jobb lagrer starttid, kilde-/målrot, filterhash, Rclone-build, oppløste ikke-sensitive alternativer, sluttstatistikk og exitkode.

De dokumenterte exitkodene skiller mellom suksess, syntaksfeil, manglende stier, midlertidige, mindre alvorlige og fatale feil samt nådde overførings- eller tidsgrenser ([List of exit codes](https://rclone.org/docs/#list-of-exit-codes)). `--error-on-no-transfer` kan signalisere en vellykket kjøring uten overføring separat. Schedulere og overvåking behandler kodene etter jobbtype; en forventet tom inkrementell jobb vurderes annerledes enn en uteblitt daglig eksport.

En High-Level-Retry kan løse tidlige feilmeldinger. Omvendt kan en prosess overføre mye og til slutt ende nonzero på grunn av få uløste objekter. Det alarmeres på sluttstatus, feilantall, oppbrukte retries, jobbvarighet, manglende heartbeat og verifiseringsresultat.

## Sikkerhet i datastien

Rclone arver rettighetene til prosesskontoen og remote-credentials. En `sync` med sletterettigheter er en høyt privilegert sletteagent. Least Privilege skiller mellom lesende kilde, skrivende mål, versjonerings-/retentionrettigheter og administrative buckethandlinger.

TLS-sertifikatkontroll forblir aktiv. `--no-check-certificate` deaktiverer kontroll av kjede og vertsnavn og er ifølge dokumentasjonen kun beregnet for testing ([TLS options](https://rclone.org/docs/#no-check-certificate)). SFTP-backender kontrollerer Host Keys med en vedlikeholdt Known-Hosts- eller fingerprintpolicy; API-endepunkter omdirigeres ikke av ukontrollerte proxier.

Filtre, `--backup-dir`, objektversjonering og Object Lock begrenser ulike risikoer. En backupkatalog i samme konto beskytter ikke mot kontokompromittering eller sletting hos leverandøren. [Storage](/kb/storage), [Kryptering](/kb/verschluesselung) og [Herding](/kb/haertung) behandler de underliggende kontrollflatene.

Diagnosen skiller mellom lokal sti, Rclone-filter, backend-API, autentisering, rate limit og nettverksvei. Bare slik blir det synlig om en fil aldri ble valgt, ikke ble overført eller ble fremstilt annerledes av målet.

## Nettverks- og backenddiagnose

En rclone-feilmelding kan stamme fra navneoppløsning, TCP, TLS, proxy, leverandørautentisering, API-rate limit, backendoversettelse, lokalt filsystem eller sammenligningslogikk. Diagnosen følger stien:

1. Bekreft binærfil, konfigurasjonssti og oppløst remote-navn.
2. Kontroller DNS, port og TLS for det konkrete endepunktet.
3. Bruk `rclone lsd` eller `lsjson` som en minimal lesende backendtest.
4. Registrer forespørsels-/retryfase og backendfunksjoner med `-vv`, uten å publisere hemmeligheter.
5. Kontroller kilde/mål, filtre og fingeravtrykksbeslutning.
6. Korreler sluttstatus, exitkode og leverandørens revisjonslogg.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für rclone-Netzwerk- und Backenddiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName $env:RCLONE_ENDPOINT
Test-NetConnection $env:RCLONE_ENDPOINT -Port 443
rclone lsd archive: -vv --dump headers --log-file '.\rclone-debug.log'
rclone lsjson archive:health --max-depth 1 --metadata</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig +short "$RCLONE_ENDPOINT"
nc -vz "$RCLONE_ENDPOINT" 443
openssl s_client -connect "$RCLONE_ENDPOINT:443" -servername "$RCLONE_ENDPOINT" -brief &lt;/dev/null
rclone lsd archive: -vv --dump headers --log-file ./rclone-debug.log
rclone lsjson archive:health --max-depth 1 --metadata</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname), [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection), [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility), [`nc`](https://man.openbsd.org/nc), [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/), [`rclone lsd`](https://rclone.org/commands/rclone_lsd/) og [`rclone lsjson`](https://rclone.org/commands/rclone_lsjson/) skiller transport, autentisering, listing og objektsynlighet. Headerdumper kontrolleres for tokener, signaturer, cookies og stier før de deles. [DNS](/kb/dns), [TCP](/kb/tcp), [TLS](/kb/tls) og [Feilsøking](/kb/troubleshooting) utdyper fasene.

## Typiske feilbilder

Rclone rapporterer ofte trinnet der en jobb synlig feiler, ikke nødvendigvis den opprinnelige årsaken. Tabellen kobler derfor symptom, sannsynlig utløsende faktor og beviset som bekrefter eller avkrefter antakelsen.

| Symptom | Sannsynlig årsak | Bevis | Feil kortløsning |
|---|---|---|---|
| Filer kopieres på nytt ved hver kjøring | ModTime/hash er ikke felles eller presisjonen avviker | Backendfunksjoner, `-vv`, Check-rapport | angi `--size-only` blindt |
| Målobjekter forsvinner | Sync-retning, filter eller Delete-alternativ | Dry-Run-/Combined-rapport, jobbdefinisjon | bare starte jobben på nytt |
| Mount viser gamle navn | Directory Cache eller manglende polling | VFS-statistikk, RC `vfs/forget`, leverandørlisting | tvangsavslutte prosessen |
| Mount-skriving mangler på remote | filen er åpen, Writeback venter, cache er full | VFS-cache og logg | slette lokal cache |
| Hashkontroll ikke mulig | ingen felles hash, Crypt eller multipartsemantikk | Backendoversikt, `hashsum`, `check --download` | anta at ETag er MD5 |
| RC svarer eksternt | for bred lytteradresse/manglende autentisering | socketliste, brannmur, RC-flagg | bare holde URL-en hemmelig |
| Bisync avbryter etter filterendring | arbeidstilstand beskytter mot feil slettinger | Bisync-logg og filterhash | slette arbeidsfiler og starte normalt |
| Exitkode nonzero til tross for mange overførte filer | restfeil etter retries | sluttlogg, feilrapport | vurdere overføringsteller som suksess |

## Backup, recovery og reconciliation

Rclone er ofte del av en backup, men er i seg selv ikke en uforanderlig backupmodell. `sync` replikerer slettinger og krypteringsskade. `copy` bevarer ekstra målobjekter, men kan overskrive eksisterende versjoner. Beskyttelse oppstår gjennom målversjonering, Retention/Object Lock, separate credentials, `--backup-dir`, uforanderlige snapshots og restoretester.

Følgende inngår i recoverysettet:

- godkjent rclone-binærfil eller verifiserbart releaseartefakt;
- kryptert `rclone.conf` pluss separat dekrypteringshemmelighet;
- leverandørcredentials og gjenanskaffelse av dem;
- Crypt-passord og salt;
- jobb-, filter- og miljødefinisjoner;
- Bisync-arbeidsfiler;
- ikke-opplastede VFS-cacheobjekter og deres tilordning;
- Dry-Run-, Combined-, Check- og exitkoderapporter;
- målets prosedyrer for versjonering, retention og restore.

En restore utføres til en separat sti og kontrolleres deretter med hash, antall, størrelse, metadatastikkprøve og applikasjonstest. Reconciliation sammenligner Source of Truth, målversjoner, VFS-Writeback og schedulertilstand. [Backup og Disaster Recovery](/kb/backup-dr) beskriver RPO/RTO og overordnet akseptanse.

## Teknisk historie

Nick Craig-Wood startet prosjektet med en første commit i november 2012. Grunnleggeren dokumenterte denne opprinnelsen ved fjortenårsjubileet og lenket til den første kildetilstanden ([Happy 14th Birthday rclone](https://forum.rclone.org/t/happy-14th-birthday-rclone/53005), [initial commit](https://github.com/rclone/rclone/commit/e9ae4f89a44b1cac8e83ede1572f15dace692c67)).

Rclone tok i bruk den velkjente retningssyntaksen fra `rsync`, men erstattet dets blokkbasserte deltaforutsetninger med objekt- og leverandøroperasjoner. Skylagring tilbyr normalt ikke et servergrensesnitt for rsync Rolling Checksum; Rclone sammenligner hele objekter med tilgjengelige fingeravtrykk og overfører dem tilsvarende.

Med et voksende antall backender utviklet den sentrale Fs/Object-grensesnittmodellen seg, med valgfrie funksjoner. Crypt la til kryptering på klientsiden; Mount og VFS gjorde objektlagring tilgjengelig for filbaserte applikasjoner; Remote Control API ble kraftig utvidet i 2018; Bisync la til vedvarende toveis avstemming. [Changelog](https://rclone.org/changelog/) dokumenterer denne utviklingen og sikkerhetsrelevante rettelser. For drift er den derfor det tidsavhengige supplementet til denne arkitekturartikkelen.

Den tekniske linjen forblir konsistent: én portabel Go-binærfil, mange innebygde backender, felles kommandoer og eksplisitt synlige leverandørforskjeller. Nettopp disse forskjellene gjør Rclone kraftig for administratorer og samtidig revisjonskrevende.

## Administrator-sjekkliste

Før en produksjonsjobb må minst følgende punkter være dokumentert:

- Binærfilopprinnelse, konfigurasjonssti og remote-oppløsning er entydige.
- Kilde, mål, retning og forventet målstruktur står i runbooken.
- Backendfunksjoner for hash, ModTime, Case, Metadata, Copy/Move og Listing er kontrollert.
- Filterfil og jobbparametere er versjonert; Dry Run og Combined Report er godkjent.
- Delete-, Max-Delete-, Backup-Dir-, versjonerings- og retentionpolicy passer sammen.
- Credentials har bare nødvendige rettigheter; Config, Crypt-passord og salt kan gjenopprettes.
- Transfers, Checkers, Pacer og båndbredde respekterer vert og leverandør.
- VFS-cache har eget volum, kapasitetsalarm, kontrollert stopp og Writeback-kontroll.
- RC/Serve lytter bare kontrollert og er beskyttet som shelltilgang.
- Suksess bevises med sluttstatus, exitkode, Check-rapport og restoretest.

## Kilder

- [Rclone](https://rclone.org/)
- [Overview of cloud storage systems](https://rclone.org/overview/)
- [rclone source repository](https://github.com/rclone/rclone)
- [Install from source](https://github.com/rclone/rclone/blob/master/docs/content/install.md)
- [Rclone documentation – remotes](https://rclone.org/docs/)
- [rclone copy](https://rclone.org/commands/rclone_copy/)
- [rclone version](https://rclone.org/commands/rclone_version/)
- [rclone config file](https://rclone.org/commands/rclone_config_file/)
- [rclone listremotes](https://rclone.org/commands/rclone_listremotes/)
- [rclone backend features](https://rclone.org/commands/rclone_backend/)
- [rclone about](https://rclone.org/commands/rclone_about/)
- [Backend Support Tiers](https://rclone.org/tiers/)
- [rclone config encryption](https://rclone.org/commands/rclone_config_encryption/)
- [rclone sync](https://rclone.org/commands/rclone_sync/)
- [rclone move](https://rclone.org/commands/rclone_move/)
- [Rclone Filtering](https://rclone.org/filtering/)
- [rclone check](https://rclone.org/commands/rclone_check/)
- [rclone cryptcheck](https://rclone.org/commands/rclone_cryptcheck/)
- [Crypt backend](https://rclone.org/crypt/)
- [rclone mount](https://rclone.org/commands/rclone_mount/)
- [WinFsp documentation](https://winfsp.dev/doc/WinFsp-Tutorial/)
- [Rclone mount in Docker](https://rclone.org/install/)
- [rclone bisync](https://rclone.org/bisync/)
- [rclone rcd](https://rclone.org/commands/rclone_rcd/)
- [Remote Control / API](https://rclone.org/rc/)
- [Invoke-WebRequest](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest)
- [curl manual](https://curl.se/docs/manpage.html)
- [rclone rc](https://rclone.org/commands/rclone_rc/)
- [Rclone serve](https://rclone.org/commands/rclone_serve/)
- [Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [dig manual](https://bind9.readthedocs.io/en/latest/manpages.html)
- [nc(1)](https://man.openbsd.org/nc)
- [openssl-s_client(1)](https://docs.openssl.org/master/man1/openssl-s_client/)
- [rclone lsd](https://rclone.org/commands/rclone_lsd/)
- [rclone lsjson](https://rclone.org/commands/rclone_lsjson/)
- [Happy 14th Birthday rclone](https://forum.rclone.org/t/happy-14th-birthday-rclone/53005)
- [Initial rclone commit](https://github.com/rclone/rclone/commit/e9ae4f89a44b1cac8e83ede1572f15dace692c67)
- [Rclone Changelog](https://rclone.org/changelog/)
