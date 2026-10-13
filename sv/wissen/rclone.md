---
title: "Rclone: fjärrlagringar, datajämförelse och VFS-drift"
blatt: "rclone"
description: "Rclone för lagrings-, plattforms- och meddelandeadministratörer: Go-arkitektur och enhetlig backendmodell, modell för fjärrlagringar och konfiguration, Copy/Sync/Move/Bisync, jämförelse och kontrollsummor, Crypt, VFS-mount och cache, Retry/Pacer, filter, RC API, loggar, säkerhet, återställning och teknisk historik."
fakten:
  - label: Systemroll
    wert: CLI- och API-verktyg för kopiering, synkronisering, verifiering, mount och tillhandahållande mellan fil- och objektlagringar
    href: https://rclone.org/
  - label: Teknikstack
    wert: Open-source-projekt i Go; backends och kommandon byggs in i en portabel rclone-binärfil
    href: https://github.com/rclone/rclone
  - label: Adressmodell
    wert: namngiven fjärrlagring plus kolon och sökväg; lokala sökvägar förblir utan fjärrprefix
    href: https://rclone.org/docs/
  - label: Backendkontrakt
    wert: gemensamt Fs/Object-gränssnitt med valfria funktioner för hash, ModTime, Copy, Move, Polling, Metadata och ListR
    href: https://rclone.org/overview/
  - label: Riktning
    wert: copy lägger till utan att rensa målet · sync gör målet till en avbild av källan · move raderar källan först efter lyckad kopiering
    href: https://rclone.org/commands/rclone_sync/
  - label: Jämförelse
    wert: storlek plus ModTime eller gemensam hash; flaggor och backendfunktioner ändrar fingeravtrycket
    href: https://rclone.org/overview/
  - label: Integritetsbevis
    wert: överföringskontroll, rclone check, cryptcheck eller nedladdningsjämförelse; exitkod och slutlogg ingår i resultatet
    href: https://rclone.org/commands/rclone_check/
  - label: Mountsökväg
    wert: VFS via FUSE/WinFsp; Directory Cache, Buffer och File Cache ligger mellan applikationen och fjärrlagringen
    href: https://rclone.org/commands/rclone_mount/
  - label: Kryptering
    wert: Crypt krypterar innehåll med NaCl SecretBox och normalt även namn; konfiguration respektive lösenord och salt är återställningsobjekt
    href: https://rclone.org/crypt/
  - label: Styrning
    wert: Remote Control API och OpenMetrics-slutpunkt; åtkomst motsvarar i praktiken rclone-användarens shellbehörigheter
    href: https://rclone.org/rc/
  - label: Felregel
    wert: Low-Level-Retry, Pacer och Command-Retry; framgång först efter slutstatus och exitkod, inte efter ett enskilt felmeddelande
    href: https://rclone.org/docs/#retries
  - label: Återställningsobjekt
    wert: binärfil, konfiguration, autentiseringsuppgifter, filter, jobbdefinition, Bisync-arbetsfiler, VFS-cache, målversioner och verifieringsrapport
    href: https://rclone.org/docs/#configuration-encryption
werbung:
  - newsletter
ctaThemen:
  - rclone
translationSourceHash: bb7131a94d721cf6984869b9d907a4d2089758b80e651c8b03bd86f72841d42e
translationModel: gpt-5.6-terra
translatedAt: 2026-10-12T11:37:10.908Z
translationReview: automatic
---

# Rclone: fjärrlagringar, datajämförelse och VFS-drift

Rclone är ett verktyg med öppen källkod för dataförflyttning för lokala filsystem, nätverksprotokoll, molnlagringar och objektlagring. [Rclone-projektsidan](https://rclone.org/) beskriver det som ett kommandoradsprogram för hantering av filer på molnlagringar. Det ger dessa system ett gemensamt gränssnitt för listning, kopiering, synkronisering, verifiering, mount, kryptering och tjänsteåtkomst. Denna enhetlighet är användbar, men får inte förväxlas med enhetlig lagringssemantik: hashar, ändringstider, versal-/gemenkänslighet, tomma kataloger, metadata, serverbaserade kopior och konsistens skiljer sig mellan backends.

För administratörer är rclone inte en ”molnutforskare”, utan en datapipeline. Dess korrekthet beror på källa och mål, riktning, jämförelsekriterium, filteruppsättning, raderingsregel, retry-tillstånd, autentiseringsuppgifter, backendfunktioner och avslutande verifiering. En process kan ha överfört 100 procent av bytena och ändå vara sakligt felaktig om den använder fel namnrymd, en inaktuell listningsvy eller en ofullständig filteruppsättning.

Förklaringen följer ett rclone-kommando från källa och fjärrlagring via jämförelse, checker och transfers till backend. Filter, Crypt, VFS, automatisering, säkerhet och verifiering förklaras därefter längs samma jobbflöde.

Rclone översätter en enhetlig fil- och katalogmodell till API:erna hos mycket olika lagringstjänster. Artikeln börjar därför med fjärrlagring och backendfunktioner, behandlar sedan kopieringssemantik och verifiering och avslutar med mount, säkerhet och återställning.

## Arkitektur: kommando, filsystemmodell och backend

Programmet, som är skrivet i Go, kopplar samman flera lager:

1. **Kommando och jobbdefinition:** `copy`, `sync`, `move`, `check`, `bisync`, `mount`, `serve` eller ett RC-anrop definierar riktning och operation.
2. **Filter och jämförelse:** sökvägsregler, storlek, ändringstid, hash, metadata och flaggor avgör den betraktade objektmängden och likheten.
3. **Överföringsmotor:** lister, checker och transferworkers planerar operationer, begränsar parallellitet och samlar fel.
4. **Fs/Object-gränssnittsmodell:** Rclone modellerar kataloger, objekt och valfria backendfunktioner via gemensamma Go-gränssnitt.
5. **Backendadaptrar:** S3, SFTP, WebDAV, Drive, OneDrive, lokala filsystem och andra tjänster översätter kontraktet till sina API:er.
6. **Transport och leverantör:** HTTP/TLS, SSH/SFTP eller lokal I/O möter rate limits, konsistens, versionshantering och leverantörspolicy.

Den officiella backendöversikten visar uttryckligen vilka system som stöder hashar, ModTime, skiftlägesokänsliga namn, dubbletter, MIME-typer och metadata ([Overview of cloud storage systems](https://rclone.org/overview/)). Källträdet separerar backends, kommandon och den gemensamma `fs`-gränssnittsmodellen; officiella byggen paketerar de valda adaptrarna i en binärfil ([rclone source repository](https://github.com/rclone/rclone), [Install from source](https://github.com/rclone/rclone/blob/master/docs/content/install.md#source)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-rclone.svg?v=20260813" title="Interaktive Infografik: rclone-Job von Quelle, Remote und Filter über Listing, Vergleich, Checker, Transfer, Backend und Provider bis zu VFS, Crypt, Logs, Exitcode, Verifikation und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-rclone.svg?v=20260813">Öppna interaktiv grafik direkt</a>.
</iframe>

## Syntax för fjärrlagring och namnrymd

En konfigurerad fjärrlagring adresseras som `name:pfad`. Kolonet skiljer fjärrlagringsnamnet från sökvägen; en lokal sökväg utan fjärrprefix använder Local-backenden. Anslutningssträngar som `:sftp,host=example.net:/data` kan instansiera en backend utan ett sparat fjärrlagringsnamn. Denna flexibilitet ökar samtidigt risken att ett skrivfel betecknar en lokal i stället för en fjärrsökväg ([Rclone documentation – remotes](https://rclone.org/docs/)).

Vid katalogkommandon överför Rclone **innehållet** i källsökvägen, inte automatiskt namnet på källkatalogen. Semantiken motsvarar det avslutande snedstreck som är välbekant från `rsync`. Källa, mål och förväntat målträd dokumenteras därför som konkreta exempel i runbooken ([rclone copy](https://rclone.org/commands/rclone_copy/)).

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

[`rclone version`](https://rclone.org/commands/rclone_version/) identifierar binärfil, operativsystem och bygge; [`rclone config file`](https://rclone.org/commands/rclone_config_file/) visar den faktiskt laddade konfigurationen; [`rclone listremotes`](https://rclone.org/commands/rclone_listremotes/) inventerar fjärrlagringar; [`rclone backend features`](https://rclone.org/commands/rclone_backend/) ger funktionerna för den upplösta backenden; [`rclone about`](https://rclone.org/commands/rclone_about/) hämtar kapacitetsinformation, om tjänsten tillhandahåller den.

## Backendfunktioner och genomlysande skillnader

Det gemensamma gränssnittet kan bara använda funktioner som backenden eller en wrapper tillhandahåller. Viktiga skillnader är:

| Funktion | Betydelse | Effekt vid frånvaro eller avvikelse |
|---|---|---|
| gemensam hash | leverantörsspecifikt innehållsfingeravtryck | jämförelsen faller tillbaka på storlek/ModTime eller nedladdning |
| skrivbar ModTime | källtiden kan anges utan ny uppladdning | identiskt innehåll kan överföras igen |
| serverbaserad Copy/Move | data stannar hos leverantören | annars nedladdning/uppladdning eller Copy plus Delete |
| ListR/Fast List | rekursiv listning med färre anrop | fler API-förfrågningar eller mer klientminne |
| Change Polling | signalerar externa ändringar | VFS ser ändringar först efter att cachen löpt ut |
| tomma kataloger | verkligt katalogobjekt/markör | tomma mappar försvinner i bucketmodeller |
| Case Sensitivity | `A` och `a` kan särskiljas | oändlig synkronisering eller namnkonflikt vid målet |
| Duplicate Names | flera objekt med samma namn | sökvägen är ingen entydig identifierare |
| Metadata/MIME | extra attribut kan läsas/skrivas | ACL:er eller verksamhetsmetadata följer inte automatiskt |

Supportnivåerna bedömer backends utifrån tester och funktioner, inte ett generellt funktionslöfte ([Backend Support Tiers](https://rclone.org/tiers/)). Leverantörs- och backenddokumentation är därför del av jobbdefinitionen. En S3-kompatibel tjänst kan ha andra gränser för multipart, hash eller Server-Side Copy än en annan S3-leverantör.

## Konfiguration, autentiseringsuppgifter och upplösningsordning

`rclone.conf` lagrar definitioner av fjärrlagringar och ofta OAuth-token eller åtkomstuppgifter. ”Obscured” lösenordsfält är ingen säker kryptering. Hela konfigurationen kan krypteras med ett lösenord; för obevakade jobb kan ett externt Password Command leverera hemligheten från en Secret Store ([Configuration encryption](https://rclone.org/docs/#configuration-encryption), [rclone config encryption](https://rclone.org/commands/rclone_config_encryption/)).

Rclone kan hämta backendalternativ från kommandoraden, fjärrspecifika miljövariabler, backendövergripande variabler och konfigurationsfilen. Prioritetsordningen måste dokumenteras för tjänster och containrar, eftersom en ärvd variabel kan skriva över den förväntade `rclone.conf` ([Environment variables](https://rclone.org/docs/#environment-variables)). Debugloggar visar upplösta alternativ, men kan innehålla känsliga slutpunkter eller sökvägar.

Administratörer skyddar minst:

- `rclone.conf` och dess filbehörigheter;
- konfigurationslösenordet respektive Password Command;
- OAuth-refresh-token, Client Secrets, Access Keys och SSH Private Keys;
- Crypt-lösenord och valfritt salt;
- behörigheter för tjänstekonto, bucket och mapp;
- filter- och jobbdefinitioner som del av Change Control.

En konfigurationsbackup utan dekrypteringshemlighet är ingen testad återställning. Omvänt får en krypterad konfiguration med svagt lösenord inte lagras offentligt.

När fjärrlagring, namnrymd och åtkomstuppgifter har klargjorts avgör det valda kommandot dataeffekten. Särskilt `sync` är ingen ofarlig kopiering, utan genomdriver ena sidan som önskad sanning.

## Copy, Sync och Move: riktning är policy

[`rclone copy`](https://rclone.org/commands/rclone_copy/) kopierar nya eller olika filer till målet och raderar inga extra objekt där. [`rclone sync`](https://rclone.org/commands/rclone_sync/) gör målet till en avbild av källan och kan då radera målobjekt. [`rclone move`](https://rclone.org/commands/rclone_move/) kopierar respektive flyttar och tar bort källan först efter en lyckad filoperation.

Dessa kommandon är inte utbytbara:

| Operation | Skriver över mål | Raderar extra målobjekt | Raderar källa | Typisk användning |
|---|---:|---:|---:|---|
| `copy` | för objekt som bedömts vara olika | nej | nej | import, seed, additiv replikering |
| `sync` | ja | ja | nej | auktoritativ enkelriktad spegling |
| `move` | ja | nej | efter lyckad överföring | inmatning, arkivöverlämning |
| `copyto`/`moveto` | enskilt objekt med explicit målnamn | nej | endast `moveto` | atomisk operation för enskild fil |
| `bisync` | på båda sidor | beroende på fastställt delta | möjligt på båda sidor | kontrollerad dubbelriktad replikering |

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

`--dry-run` förhindrar ändringar, men bevisar endast de listningar och alternativ som var synliga vid testtillfället. `--check-first` separerar jämförelse- och överföringsfasen. `--backup-dir` flyttar överskrivna eller raderade målobjekt till en separat sökväg, förutsatt att backenden stöder den nödvändiga serverbaserade operationen ([Global options – backup-dir and retries](https://rclone.org/docs/)). Combined Report klassificerar lika, saknade, olika och felaktiga sökvägar.

## Vad betyder ”identisk”?

Rclone hoppar som standard över filer baserat på storlek och ändringstid eller använder en gemensam hash när kommando och backend tillåter det. Det exakta fingeravtrycket beror på funktioner och flaggor. `--checksum`, `--size-only`, `--ignore-times`, `--update`, `--ignore-size`, metadataalternativ och ModTime-precision ändrar beslutet ([Overview – Hash and ModTime](https://rclone.org/overview/)).

En leverantörsspecifik ETag är inte universellt en innehålls-MD5. Multipartuppladdning, kryptering eller leverantörsimplementering kan förändra dess betydelse. Rclone använder den hashfunktion som backenden uttryckligen deklarerar, inte ett generellt ETag-antagande.

Mellan skiftlägeskänsligt Linux och normalt skiftlägesokänsligt Windows kan `report.pdf` och `Report.pdf` inte mappas utan förlust. Molnlagringar kan tillåta dubbla namn eller egna objekt-ID:n. En förkontroll av namnkonflikter och otillåtna tecken är därför del av varje migrering.

## Överföringsmotor, parallellitet och Pacer

Överföringsmotorn listar källa och mål, jämför poster, planerar kontroller och överföringar samt kör backendoperationer parallellt. `--checkers` styr parallella kontroller, `--transfers` överföringarna. `--fast-list` kan minska API-anrop, men kräver mer minne eftersom stora listningar samlas in ([Rclone documentation – performance options](https://rclone.org/docs/)).

Parallellitet är inte enbart ett hastighetsalternativ. Fler workers ökar öppna filer, minne, API-förfrågningar, multiparttillstånd och belastning på leverantören. För små objekt dominerar listnings-/förfrågningslatens, för stora objekt bandbredd och multipartkonfiguration. Moln-till-moln-kopior kan stanna på serversidan eller gå via rclone-värden; detta kontrolleras i loggen och backendfunktionen.

Backends implementerar pacing för API-rate limits. Low-Level-Retries hanterar enskilda förfrågningar; High-Level-Retries upprepar en misslyckad helhetsoperation. Först slutstatus avgör om tidigare fel kompenserades framgångsrikt ([Retries](https://rclone.org/docs/#retries)). En scheduler utvärderar processens exitkod och inte förekomsten av ordet `ERROR` någonstans i loggen.

## Filter definierar datamängden

Include-, Exclude-, Size-, Age- och Files-from-regler avgör vilka sökvägar som är synliga. Filter relaterar till den angivna roten; ett shellglob i källsökvägen ersätter ingen rclone-filterregel. Ordningen mellan Include och Exclude kan vara relevant ([Rclone Filtering](https://rclone.org/filtering/)).

Ändras ett filter ändras den sakliga datamängden. Vid `sync` kan tidigare synliga objekt framstå som raderade; `--delete-excluded` är särskilt farligt. Filterfiler versionshanteras, hashas och godkänns tillsammans med Dry-Run-rapport och jobbdefinition. Ett säkert test använder `rclone ls` eller `lsf` med samma regler innan ett skrivande kommando följer.

En exitkod utan överföringsfel bevisar endast körningen, inte fullständig likhet mellan båda bestånden. `check` jämför källor och mål med respektive backends funktioner.

## Verifiering: Check i stället för överföringsräknare

[`rclone check`](https://rclone.org/commands/rclone_check/) ändrar inga data. Det jämför storlek och gemensamma hashar, kan skapa rapporter för träffar, skillnader och saknade sökvägar samt läsa innehåll med `--download` när ingen gemensam fjärrhash finns. För Crypt finns [`rclone cryptcheck`](https://rclone.org/commands/rclone_cryptcheck/), som lämpligt jämför klartextkälla och krypterat mål.

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

En hashjämförelse bevisar likhet för den använda algoritmen och namnrymden. Den bevisar inte automatiskt ACL:er, retention, leverantörsobjektversioner, katalogmetadata eller verksamhetsmässig läsbarhet. För kritiska arkiv kompletterar ett återställningstest den tekniska kontrollen.

## Crypt: kryptering av data och namn

Crypt är en wrapperbackend ovanpå en annan fjärrlagring. Filinnehåll krypteras med NaCl SecretBox baserat på XSalsa20 och Poly1305; fil- och katalognamn transformeras normalt också ([Crypt backend](https://rclone.org/crypt/)). Leverantören ser chiffertext, storleksmönster, åtkomsttider och antal objekt; Crypt är därför skydd för innehåll och namn, inte fullständig anonymisering av trafik eller metadata.

Lösenord och valfritt salt lagras endast ”obscured” i konfigurationen, om inte hela konfigurationen är krypterad. Förlust av båda återställningsuppgifterna gör data oläsbara. Före rotation eller ändring av namnkryptering testas en andra, läsande fjärrlagring mot samma data.

Crypt ändrar storlekar och hashmöjligheter. En vanlig `check` mellan klartext och chiffertext kan därför inte bara förvänta sig samma leverantörshash. `cryptcheck` eller en kontrollerad nedladdningssökväg är rätt kontrollnivå.

Kopieringskommandon arbetar uttryckligen med objekt. En VFS-mount måste däremot återskapa löpande filsystemoperationer och behöver för detta cache, skrivläge och tydliga konsistensantaganden.

## VFS-mount: objektlagring är inget POSIX-filsystem

[`rclone mount`](https://rclone.org/commands/rclone_mount/) presenterar en fjärrlagring som ett filsystem via FUSE respektive WinFsp. VFS-lagret översätter filoperationer till objektoperationer. Eftersom molnobjekt vanligtvis inte kan ändras i mitten eller utökas godtyckligt buffrar eller ersätter Rclone objekt.

Fyra cachelägen ger olika kontrakt:

| Läge | Lokal filcache | Skrivbeteende | Driftgräns |
|---|---|---|---|
| `off` | ingen | direkt och begränsat | ingen Seek/Read-Write som för normal fil; misslyckad uppladdning svårare att upprepa |
| `minimal` | endast vid nödvändiga Read-Write-fall | delvis buffrat | inte all Open-/Seek-semantik är möjlig |
| `writes` | skrivna filer lokalt | uppladdning efter Close/Writeback | lokal cache är ännu ingen fjärrcommit |
| `full` | Reads och Writes lokalt, delvis sparse | hög filsystemkompatibilitet | övervaka diskbehov, Eviction och cachekonsistens |

Directory Cache håller namn och metadata i minnet. Externa ändringar visas först via polling eller efter utgång respektive flush. Buffer gäller per öppen fil; maximalt minnesbehov ökar med `buffer-size × open files`. Två processer får inte använda samma överlappande VFS-cache med aktiv File Caching, eftersom detta kan skada data ([VFS file caching](https://rclone.org/commands/rclone_mount/#vfs-file-caching)).

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

I Windows tillhandahåller [WinFsp](https://winfsp.dev/doc/WinFsp-Tutorial/) filsystembryggan; i Linux och andra Unixsystem använder Rclone FUSE. En tjänst väntar på nätverk, konfiguration och mountberedskap, har en kontrollerad unmountsökväg och övervakar lokalt cachevolume. Container-mounts kräver ytterligare enhets- och mount-propagation-behörigheter och utökar behörighetsytan ([Rclone mount in Docker](https://rclone.org/install/#docker)).

## Bisync: två sanningskällor

[`rclone bisync`](https://rclone.org/bisync/) identifierar ändringar på båda sidor utifrån sparade listningar och överför deltan dubbelriktat. Därmed skapas beständigt jobbtillstånd utanför användardata. En förlorad eller inkonsekvent arbetskatalog kan kräva en resync-fas.

Bisync har säkerhetskontroller för nåbarhet och ovanligt många raderingar. Filterändringar ändrar den synliga datamängden och tvingar medvetet fram en resync. Om applikationer ändrar innehåll utan att ändra storlek och ModTime kan standardidentifieringen misslyckas. Dubbelriktad synkronisering används därför bara när konfliktpolicy, clock-/timestampbeteende, Access Checks, Max-Delete, backup av arbetsfiler och mänsklig lösning är dokumenterade.

Bisync är inget multi-master-filsystem med distribuerad låsning. Samtidiga redigeringar, namnändringar och offlineperioder förblir verksamhetsmässiga konflikter.

## Remote Control API, Metrics och Serve-lägen

Med `--rc` eller [`rclone rcd`](https://rclone.org/commands/rclone_rcd/) öppnar Rclone ett HTTP-API. Det kan starta jobb, konfigurera fjärrlagringar, läsa eller skriva lokala filer och under vissa omständigheter köra OS-kommandon i rclone-användarens kontext. Dokumentationen bedömer därför åtkomst till RC som likvärdig med shellåtkomst ([Remote Control / API](https://rclone.org/rc/)).

RC lyssnar som standard lokalt. Vid nätverksexponering krävs stark autentisering, TLS eller en skyddad reverse proxy, brandväggsallowlist och minimala processbehörigheter. `--rc-no-auth` hör inte hemma på ett nåbart gränssnitt. OpenMetrics-slutpunkten kan exponera överförings- och jobbmetrik; även där kan fjärrlagringsnamn och sökvägar vara känsliga metadata.

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

[`Invoke-WebRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest) och [`curl`](https://curl.se/docs/manpage.html) kontrollerar den lokala HTTP-slutpunkten. [`rclone rc`](https://rclone.org/commands/rclone_rc/) använder samma API och tar över autentiseringsparametrar från jobbkonfigurationen.

Rclone kan dessutom erbjuda fjärrlagringar via HTTP, WebDAV, SFTP, FTP, NFS, DLNA eller S3-kompatibla Serve-lägen. Därmed ändras systemrollen från klient till server. Lyssnaradress, autentisering, protokollsäkerhet, VFS-cache och datarättigheter härdas då som för en fristående tjänst ([Rclone serve](https://rclone.org/commands/rclone_serve/)).

## Loggning, statistik och exitkoder

Loggar skiljer mellan DEBUG, INFO, NOTICE och ERROR. För maskinbearbetning finns JSON-loggning. Statistik visar checks, transfers, fel, byte, hastighet och ETA. En kort `Transferred`-räknare är inget varaktigt revisionsbevis; ett jobb lagrar starttid, käll-/målrot, filterhash, Rclone-bygge, upplösta icke-känsliga alternativ, slutstatistik och exitkod.

De dokumenterade exitkoderna skiljer mellan framgång, syntaxfel, saknade sökvägar, tillfälliga, mindre allvarliga och fatala fel samt uppnådda överförings- eller tidsgränser ([List of exit codes](https://rclone.org/docs/#list-of-exit-codes)). `--error-on-no-transfer` kan separat signalera en lyckad körning utan överföring. Schedulers och övervakning behandlar koderna efter jobbtyp; ett förväntat tomt inkrementellt jobb bedöms annorlunda än en utebliven daglig export.

En High-Level-Retry kan lösa tidiga felmeddelanden framgångsrikt. Omvänt kan en process överföra mycket och ändå avslutas nonzero på grund av få olösta objekt. Larm utlöses för slutstatus, antal fel, uttömda retries, jobblängd, utebliven heartbeat och verifieringsresultat.

## Säkerhet i datavägen

Rclone ärver behörigheterna för sitt პროცესskonto och fjärrlagringsuppgifterna. En `sync` med Delete-rättigheter är en högt privilegierad raderingsagent. Least Privilege separerar läsande källa, skrivande mål, versionerings-/retentionsrättigheter och administrativa bucketåtgärder.

TLS-certifikatkontroll förblir aktiv. `--no-check-certificate` inaktiverar kedje- och värdnamnskontroll och är enligt dokumentationen endast avsett för tester ([TLS options](https://rclone.org/docs/#no-check-certificate)). SFTP-backends kontrollerar Host Keys med en underhållen Known-Hosts- eller fingerprintpolicy; API-slutpunkter omdirigeras inte genom okontrollerade proxier.

Filter, `--backup-dir`, objektversionshantering och Object Lock begränsar olika risker. En backupkatalog i samma konto skyddar inte mot kontokompromettering eller radering hos leverantören. [Storage](/kb/storage), [Kryptering](/kb/verschluesselung) och [Härdning](/kb/haertung) behandlar de underliggande kontrollytorna.

Diagnosen skiljer mellan lokal sökväg, Rclone-filter, backend-API, autentisering, rate limit och nätverksväg. Bara så blir det synligt om en fil aldrig valdes, inte överfördes eller visades annorlunda av målet.

## Nätverks- och backenddiagnostik

Ett rclone-felmeddelande kan komma från namnupplösning, TCP, TLS, proxy, leverantörsautentisering, API-rate limit, backendöversättning, lokalt filsystem eller jämförelselogik. Diagnosen följer vägen:

1. Bekräfta binärfil, konfigurationssökväg och upplöst fjärrlagringsnamn.
2. Kontrollera DNS, port och TLS för den konkreta slutpunkten.
3. Använd `rclone lsd` eller `lsjson` som minimalt läsande backendtest.
4. Samla request-/retryfas och backendfunktioner med `-vv` utan att publicera hemligheter.
5. Kontrollera källa/mål, filter och fingeravtrycksbeslut.
6. Korrelera slutstatus, exitkod och leverantörens revisionslogg.

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

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname), [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection), [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility), [`nc`](https://man.openbsd.org/nc), [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/), [`rclone lsd`](https://rclone.org/commands/rclone_lsd/) och [`rclone lsjson`](https://rclone.org/commands/rclone_lsjson/) separerar transport, autentisering, listning och objektsynlighet. Headerdumps granskas före vidarebefordran för token, signaturer, cookies och sökvägar. [DNS](/kb/dns), [TCP](/kb/tcp), [TLS](/kb/tls) och [Felsökning](/kb/troubleshooting) fördjupar faserna.

## Typiska felbilder

Rclone rapporterar ofta steget där ett jobb synligt misslyckas, inte automatiskt grundorsaken. Tabellen kopplar därför symptom, sannolik utlösare och beviset som bekräftar eller motbevisar antagandet.

| Symptom | Sannolik orsak | Bevis | Felaktig snabblösning |
|---|---|---|---|
| Filer kopieras på nytt vid varje körning | ModTime/hash saknas gemensamt eller precisionen avviker | backendfunktioner, `-vv`, checkrapport | ange blint `--size-only` |
| Målobjekt försvinner | synkriktning, filter eller Delete-alternativ | Dry-Run-/Combined-rapport, jobbdefinition | starta bara om jobbet |
| Mount visar gamla namn | Directory Cache eller saknad polling | VFS-statistik, RC `vfs/forget`, leverantörslistning | avsluta processen hårt |
| Mountskrivning saknas på fjärren | fil öppen, Writeback väntar, cache full | VFS-cache och logg | radera lokal cache |
| Hashcheck är inte möjlig | ingen gemensam hash, Crypt eller multipartsemantik | backendöversikt, `hashsum`, `check --download` | anta att ETag är MD5 |
| RC svarar externt | för bred lyssnaradress/saknad autentisering | socketlista, brandvägg, RC-flaggor | bara hålla URL:en hemlig |
| Bisync avbryts efter filterändring | arbetstillstånd skyddar mot felaktiga raderingar | Bisync-logg och filterhash | radera arbetsfiler och starta normalt |
| Exitkod nonzero trots många överförda filer | kvarstående fel efter retries | slutlogg, felrapport | tolka överföringsräknare som framgång |

## Backup, återställning och reconciliation

Rclone ingår ofta i en backup, men är självt ingen oföränderlig backupmodell. `sync` replikerar raderingar och krypteringsskador. `copy` bevarar extra målobjekt, men kan skriva över befintliga versioner. Skydd uppstår genom målversionshantering, Retention/Object Lock, separata autentiseringsuppgifter, `--backup-dir`, oföränderliga snapshots och återställningstester.

Återställningsuppsättningen omfattar:

- godkänd rclone-binärfil eller verifierbar releaseartefakt;
- krypterad `rclone.conf` plus separat dekrypteringshemlighet;
- leverantörsautentiseringsuppgifter och deras återanskaffning;
- Crypt-lösenord och salt;
- jobb-, filter- och miljödefinitioner;
- Bisync-arbetsfiler;
- VFS-cacheobjekt som inte laddats upp och deras tilldelning;
- Dry-Run-, Combined-, Check- och exitkodrapporter;
- målets versionerings-, retention- och återställningsförfarande.

En återställning utförs till en separat sökväg och kontrolleras sedan med hash, antal, storlek, stickprov på metadata och applikationstest. Reconciliation jämför Source of Truth, målversioner, VFS-Writeback och schedulertillstånd. [Backup och Disaster Recovery](/kb/backup-dr) beskriver RPO/RTO och övergripande godkännande.

## Teknisk historia

Nick Craig-Wood inledde projektet med en första commit i november 2012. Grundaren dokumenterade detta ursprung på fjortonde årsdagen och länkade till det ursprungliga källläget ([Happy 14th Birthday rclone](https://forum.rclone.org/t/happy-14th-birthday-rclone/53005), [initial commit](https://github.com/rclone/rclone/commit/e9ae4f89a44b1cac8e83ede1572f15dace692c67)).

Rclone övertog den välbekanta riktningssyntaxen från `rsync`, men ersatte dess blockbaserade deltaantaganden med objekt- och leverantörsoperationer. Molnlagringar erbjuder normalt inget serverbaserat gränssnitt för rsync rolling checksum; Rclone jämför kompletta objekt baserat på tillgängliga fingeravtryck och överför dem därefter.

Med det växande antalet backends utvecklades den centrala Fs/Object-gränssnittsmodellen med valfria funktioner. Crypt lade till kryptering på klientsidan; Mount och VFS gjorde objektlagring tillgänglig för filbaserade applikationer; Remote Control API byggdes ut kraftigt 2018; Bisync lade till beständig dubbelriktad avstämning. [Changelog](https://rclone.org/changelog/) dokumenterar denna utveckling och säkerhetsrelevanta korrigeringar. För drift är den därför det tidsberoende komplementet till denna arkitekturartikel.

Den tekniska linjen förblir konsekvent: en portabel Go-binärfil, många inbyggda backends, gemensamma kommandon och uttryckligen genomlysande leverantörsskillnader. Just dessa skillnader gör Rclone kraftfullt för administratörer och samtidigt revisionskrävande.

## Checklista för administratörer

Före ett produktionsjobb finns minst följande punkter dokumenterade:

- Binärfilens ursprung, konfigurationssökväg och fjärrupplösning är entydiga.
- Källa, mål, riktning och förväntat målträd finns i runbooken.
- Backendfunktioner för hash, ModTime, Case, Metadata, Copy/Move och listning är kontrollerade.
- Filterfil och jobbparametrar är versionshanterade; Dry Run och Combined Report har godkänts.
- Delete-, Max-Delete-, Backup-Dir-, versionshanterings- och retentionpolicy passar ihop.
- Autentiseringsuppgifter har endast nödvändiga rättigheter; Config, Crypt-lösenord och salt kan återställas.
- Transfers, Checkers, Pacer och bandbredd respekterar värd och leverantör.
- VFS-cache har egen volym, kapacitetslarm, ordnad stoppning och Writeback-kontroll.
- RC/Serve lyssnar endast kontrollerat och skyddas som shellåtkomst.
- Framgång bevisas genom slutstatus, exitkod, checkrapport och återställningstest.

## Källor

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
