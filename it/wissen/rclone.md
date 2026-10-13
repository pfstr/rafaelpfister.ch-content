---
title: "Rclone: remoti, confronto dei dati e funzionamento VFS"
blatt: "rclone"
description: "Rclone per amministratori di storage, piattaforme e messaggistica: architettura Go e modello backend uniforme, modello di remoti e configurazione, Copy/Sync/Move/Bisync, confronto e checksum, Crypt, montaggio VFS e cache, retry/pacer, filtri, API RC, log, sicurezza, recovery e storia tecnica."
fakten:
  - label: Ruolo di sistema
    wert: Strumento CLI e API per copia, sincronizzazione, verifica, montaggio e pubblicazione tra archivi di file e oggetti
    href: https://rclone.org/
  - label: Stack tecnologico
    wert: Progetto open source in Go; backend e comandi sono integrati in un binario rclone portabile
    href: https://github.com/rclone/rclone
  - label: Modello di indirizzamento
    wert: remoto denominato più due punti e percorso; i percorsi locali rimangono senza prefisso remoto
    href: https://rclone.org/docs/
  - label: Contratto backend
    wert: interfaccia Fs/Object comune con capacità opzionali per hash, ModTime, Copy, Move, Polling, Metadata e ListR
    href: https://rclone.org/overview/
  - label: Direzione
    wert: copy aggiunge senza pulizia della destinazione · sync rende la destinazione uguale alla sorgente · move elimina la sorgente solo dopo una copia riuscita
    href: https://rclone.org/commands/rclone_sync/
  - label: Confronto
    wert: dimensione più ModTime oppure hash comune; flag e capacità del backend modificano il fingerprint
    href: https://rclone.org/overview/
  - label: Prova di integrità
    wert: verifica del trasferimento, rclone check, cryptcheck o confronto tramite download; exit code e log finale fanno parte del risultato
    href: https://rclone.org/commands/rclone_check/
  - label: Percorso di montaggio
    wert: VFS tramite FUSE/WinFsp; Directory Cache, Buffer e File Cache sono posti tra applicazione e remoto
    href: https://rclone.org/commands/rclone_mount/
  - label: Crittografia
    wert: Crypt cifra i contenuti con NaCl SecretBox e per impostazione predefinita anche i nomi; configurazione oppure password e salt sono oggetti di recovery
    href: https://rclone.org/crypt/
  - label: Controllo
    wert: API Remote Control ed endpoint OpenMetrics; l'accessibilità equivale praticamente ai diritti di shell dell'utente rclone
    href: https://rclone.org/rc/
  - label: Regola degli errori
    wert: retry a basso livello, pacer e retry del comando; successo solo dopo stato finale ed exit code, non dopo un singolo messaggio di errore
    href: https://rclone.org/docs/#retries
  - label: Oggetto di recovery
    wert: binario, configurazione, credenziali, filtri, definizione del job, file di lavoro Bisync, cache VFS, versioni di destinazione e report di verifica
    href: https://rclone.org/docs/#configuration-encryption
werbung:
  - newsletter
ctaThemen:
  - rclone
translationSourceHash: bb7131a94d721cf6984869b9d907a4d2089758b80e651c8b03bd86f72841d42e
translationModel: gpt-5.6-terra
translatedAt: 2026-10-12T11:31:41.996Z
translationReview: automatic
---

# Rclone: remoti, confronto dei dati e funzionamento VFS

Rclone è uno strumento open source per lo spostamento dei dati per file system locali, protocolli di rete, unità cloud e archivi di oggetti. La [pagina del progetto Rclone](https://rclone.org/) lo descrive come un programma da riga di comando per la gestione di file negli archivi cloud. Offre a questi sistemi un'interfaccia comune per listing, copia, sincronizzazione, verifica, mount, crittografia e accesso al servizio. Questa uniformità è utile, ma non deve essere confusa con una semantica di storage uniforme: hash, tempi di modifica, maiuscole/minuscole, directory vuote, metadati, copie lato server e coerenza differiscono a seconda del backend.

Per gli amministratori, rclone non è un «Cloud Explorer», ma una pipeline di dati. La sua correttezza dipende da sorgente e destinazione, direzione, criterio di confronto, insieme di filtri, regola di eliminazione, stato dei retry, credenziali, capacità del backend e verifica finale. Un processo può avere trasferito il 100 percento dei byte ed essere comunque errato dal punto di vista funzionale, se utilizza il namespace sbagliato, una vista di listing obsoleta o un insieme di filtri incompleto.

La spiegazione segue un comando rclone dalla sorgente e dal remoto attraverso confronto, checker e transfer fino al backend. Filtri, Crypt, VFS, automazione, sicurezza e verifica vengono poi illustrati lungo lo stesso percorso del job.

Rclone traduce un modello uniforme di file e directory nelle API di servizi di storage molto diversi. L'articolo parte quindi da remoto e capacità del backend, tratta poi la semantica di copia e verifica e termina con mount, sicurezza e recovery.

## Architettura: comando, modello del file system e backend

Il programma scritto in Go collega più livelli:

1. **Comando e definizione del job:** `copy`, `sync`, `move`, `check`, `bisync`, `mount`, `serve` oppure una chiamata RC definiscono direzione e operazione.
2. **Filtri e confronto:** regole dei percorsi, dimensione, tempo di modifica, hash, metadata e flag determinano l'insieme di oggetti considerato e l'uguaglianza.
3. **Motore di trasferimento:** lister, checker e worker di trasferimento pianificano le operazioni, limitano il parallelismo e raccolgono gli errori.
4. **Modello di interfaccia Fs/Object:** Rclone modella directory, oggetti e funzioni backend opzionali tramite interfacce Go comuni.
5. **Adattatori backend:** S3, SFTP, WebDAV, Drive, OneDrive, file system locali e altri servizi traducono il contratto nelle rispettive API.
6. **Trasporto e provider:** HTTP/TLS, SSH/SFTP oppure I/O locale incontrano rate limit, coerenza, versioning e policy del provider.

La panoramica ufficiale dei backend mostra esplicitamente quali sistemi supportano hash, ModTime, nomi case-insensitive, duplicati, tipi MIME e metadati ([Overview of cloud storage systems](https://rclone.org/overview/)). L'albero dei sorgenti separa backend, comandi e il modello di interfaccia comune `fs`; le build ufficiali raccolgono gli adattatori selezionati in un binario ([rclone source repository](https://github.com/rclone/rclone), [Install from source](https://github.com/rclone/rclone/blob/master/docs/content/install.md#source)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-rclone.svg?v=20260813" title="Interaktive Infografik: rclone-Job von Quelle, Remote und Filter über Listing, Vergleich, Checker, Transfer, Backend und Provider bis zu VFS, Crypt, Logs, Exitcode, Verifikation und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-rclone.svg?v=20260813">Apri direttamente la grafica interattiva</a>.
</iframe>

## Sintassi del remoto e namespace

Un remoto configurato viene indirizzato come `name:pfad`. I due punti separano il nome del remoto dal percorso; un percorso locale senza prefisso remoto utilizza il backend Local. Le stringhe di connessione come `:sftp,host=example.net:/data` possono istanziare un backend senza un nome remoto memorizzato. Questa flessibilità aumenta al contempo il rischio che un errore di digitazione indichi un percorso locale anziché remoto ([Rclone documentation – remotes](https://rclone.org/docs/)).

Con i comandi per directory, Rclone trasferisce il **contenuto** del percorso sorgente, non automaticamente il nome della directory sorgente. La semantica corrisponde allo slash finale noto da `rsync`. Sorgente, destinazione e albero di destinazione previsto vengono pertanto annotati nel runbook come esempi concreti ([rclone copy](https://rclone.org/commands/rclone_copy/)).

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

[`rclone version`](https://rclone.org/commands/rclone_version/) identifica binario, sistema operativo e build; [`rclone config file`](https://rclone.org/commands/rclone_config_file/) mostra la configurazione effettivamente caricata; [`rclone listremotes`](https://rclone.org/commands/rclone_listremotes/) inventaria i remoti; [`rclone backend features`](https://rclone.org/commands/rclone_backend/) restituisce le capacità del backend risolto; [`rclone about`](https://rclone.org/commands/rclone_about/) interroga le informazioni sulla capacità, se offerte dal servizio.

## Capacità del backend e differenze trasparenti

L'interfaccia comune può utilizzare soltanto le capacità offerte dal backend o da un wrapper. Differenze importanti sono:

| Capacità | Significato | Effetto in caso di assenza o differenza |
|---|---|---|
| hash comune | impronta del contenuto lato provider | il confronto ricade su dimensione/ModTime o download |
| ModTime scrivibile | l'ora della sorgente può essere impostata senza reupload | contenuti identici possono essere trasferiti nuovamente |
| Copy/Move lato server | i dati rimangono presso il provider | altrimenti download/upload oppure Copy più Delete |
| ListR/Fast List | listing ricorsivo in meno chiamate | più richieste API o più memoria client |
| Change Polling | segnala modifiche esterne | VFS vede le modifiche solo alla scadenza della cache |
| directory vuote | oggetto directory/marker reale | cartelle vuote scompaiono nei modelli bucket |
| Case Sensitivity | `A` e `a` distinguibili | sincronizzazione infinita o collisione di nomi nella destinazione |
| Duplicate Names | più oggetti con lo stesso nome | il percorso non è un identificatore univoco |
| Metadata/MIME | attributi aggiuntivi leggibili/scrivibili | ACL o metadati applicativi non vengono trasferiti automaticamente |

I livelli di supporto valutano i backend sulla base di test e capacità, non di una promessa generica di funzionalità ([Backend Support Tiers](https://rclone.org/tiers/)). La documentazione del provider e del backend fa quindi parte della definizione del job. Un servizio compatibile con S3 può avere limiti diversi per multipart, hash o Server-Side Copy rispetto a un altro provider S3.

## Configurazione, credenziali e ordine di risoluzione

`rclone.conf` memorizza definizioni dei remoti e spesso token OAuth o credenziali di accesso. I campi password «obscured» non sono una crittografia sicura. L'intera configurazione può essere cifrata con una password; per job non supervisionati, un Password Command esterno può fornire il segreto da un Secret Store ([Configuration encryption](https://rclone.org/docs/#configuration-encryption), [rclone config encryption](https://rclone.org/commands/rclone_config_encryption/)).

Rclone può ottenere opzioni backend dalla riga di comando, da variabili d'ambiente specifiche del remoto, da variabili valide per tutto il backend e dal file di configurazione. L'ordine di priorità deve essere documentato per servizi e container, poiché una variabile ereditata può sovrascrivere il previsto `rclone.conf` ([Environment variables](https://rclone.org/docs/#environment-variables)). I log di debug mostrano le opzioni risolte, ma possono contenere endpoint o percorsi sensibili.

Gli amministratori proteggono almeno:

- `rclone.conf` e i relativi permessi file;
- password di configurazione o Password Command;
- token di refresh OAuth, client secret, access key e chiavi private SSH;
- password Crypt e salt opzionale;
- autorizzazioni di account di servizio, bucket e cartella;
- definizioni di filtri e job come parte del Change Control.

Un backup della configurazione senza il segreto di decrittazione non è un recovery testato. Viceversa, una configurazione cifrata con password debole non deve essere archiviata pubblicamente.

Una volta chiariti remoto, namespace e credenziali, il comando scelto determina l'effetto sui dati. In particolare, `sync` non è una copia innocua, ma impone un lato come verità desiderata.

## Copy, Sync e Move: la direzione è policy

[`rclone copy`](https://rclone.org/commands/rclone_copy/) copia file nuovi o diversi nella destinazione e non elimina oggetti aggiuntivi presenti lì. [`rclone sync`](https://rclone.org/commands/rclone_sync/) rende la destinazione una replica della sorgente e può a tale scopo eliminare oggetti di destinazione. [`rclone move`](https://rclone.org/commands/rclone_move/) copia o sposta e rimuove la sorgente solo dopo un'operazione sul file riuscita.

Questi comandi non sono intercambiabili:

| Operazione | Sovrascrive la destinazione | Elimina oggetti aggiuntivi di destinazione | Elimina la sorgente | Uso tipico |
|---|---:|---:|---:|---|
| `copy` | per oggetti valutati come diversi | no | no | importazione, seed, replica additiva |
| `sync` | sì | sì | no | mirroring unidirezionale autorevole |
| `move` | sì | no | dopo trasferimento riuscito | ingest, consegna all'archivio |
| `copyto`/`moveto` | singolo oggetto con nome di destinazione esplicito | no | solo `moveto` | operazione atomica su singolo file |
| `bisync` | su entrambi i lati | dipende dal delta rilevato | possibile su entrambi i lati | replica bidirezionale controllata |

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

`--dry-run` impedisce modifiche, ma dimostra soltanto i listing e le opzioni visibili al momento del test. `--check-first` separa la fase di confronto da quella di trasferimento. `--backup-dir` sposta oggetti di destinazione sovrascritti o eliminati in un percorso separato, se il backend supporta la necessaria operazione lato server ([Global options – backup-dir and retries](https://rclone.org/docs/)). Il Combined Report classifica percorsi uguali, mancanti, diversi e difettosi.

## Cosa significa «identico»?

Per impostazione predefinita Rclone salta i file in base a dimensione e tempo di modifica oppure utilizza un hash comune, quando comando e backend lo consentono. L'impronta esatta dipende da capacità e flag. `--checksum`, `--size-only`, `--ignore-times`, `--update`, `--ignore-size`, opzioni dei metadati e precisione ModTime modificano la decisione ([Overview – Hash and ModTime](https://rclone.org/overview/)).

Un ETag lato provider non è universalmente un MD5 del contenuto. Upload multipart, crittografia o implementazione del provider possono modificarne il significato. Rclone utilizza la capacità hash esplicitamente dichiarata dal backend, non un'assunzione generica sugli ETag.

Tra Linux case-sensitive e Windows generalmente case-insensitive, `report.pdf` e `Report.pdf` non possono essere mappati senza perdite. Le unità cloud possono consentire nomi duplicati o ID oggetto propri. Una verifica preventiva delle collisioni di nomi e dei caratteri non consentiti fa quindi parte di ogni migrazione.

## Motore di trasferimento, parallelismo e Pacer

Il motore di trasferimento elenca sorgente e destinazione, confronta le voci, pianifica check e trasferimenti ed esegue operazioni backend in parallelo. `--checkers` controlla le verifiche parallele, `--transfers` i trasferimenti. `--fast-list` può ridurre le chiamate API, ma richiede più memoria perché raccoglie listing grandi ([Rclone documentation – performance options](https://rclone.org/docs/)).

Il parallelismo non è una semplice opzione di velocità. Più worker aumentano file aperti, memoria, richieste API, stati multipart e pressione sul provider. Per oggetti piccoli domina la latenza di listing/richieste, per oggetti grandi la larghezza di banda e la configurazione multipart. Le copie cloud-to-cloud possono rimanere lato server o transitare dall'host rclone; ciò viene verificato nel log e nelle capacità del backend.

I backend implementano pacing per i rate limit API. I retry a basso livello gestiscono singole richieste; quelli ad alto livello ripetono un'intera operazione fallita. Solo lo stato finale decide se gli errori precedenti sono stati compensati con successo ([Retries](https://rclone.org/docs/#retries)). Uno scheduler valuta l'exit code del processo e non la comparsa della parola `ERROR` da qualche parte nel log.

## I filtri definiscono l'insieme di dati

Le regole Include, Exclude, Size, Age e Files-from determinano quali percorsi siano visibili. I filtri si riferiscono alla root indicata; un glob della shell nel percorso sorgente non sostituisce una regola di filtro rclone. L'ordine di Include e Exclude può essere rilevante ([Rclone Filtering](https://rclone.org/filtering/)).

Quando un filtro cambia, cambia l'insieme di dati funzionale. Con `sync`, oggetti precedentemente visibili possono apparire come eliminati; `--delete-excluded` è particolarmente pericoloso. I file di filtro vengono versionati, sottoposti ad hash e approvati insieme al report Dry Run e alla definizione del job. Un test sicuro usa `rclone ls` oppure `lsf` con le stesse regole, prima di eseguire un comando in scrittura.

Un exit code senza errori di trasferimento dimostra soltanto l'esecuzione, non la completa uguaglianza di entrambi gli insiemi. `check` confronta sorgenti e destinazioni con le capacità del rispettivo backend.

## Verifica: Check invece del contatore di trasferimento

[`rclone check`](https://rclone.org/commands/rclone_check/) non modifica dati. Confronta dimensione e hash comuni, può generare report per corrispondenze, differenze e percorsi mancanti e, tramite `--download`, può leggere contenuti se non esiste un hash remoto comune. Per Crypt esiste [`rclone cryptcheck`](https://rclone.org/commands/rclone_cryptcheck/), che confronta in modo idoneo sorgente in chiaro e destinazione cifrata.

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

Un confronto hash dimostra l'uguaglianza per l'algoritmo e il namespace utilizzati. Non dimostra automaticamente ACL, retention, versioni di oggetti del provider, metadati delle directory o leggibilità funzionale. Per archivi critici, un test di ripristino integra il controllo tecnico.

## Crypt: crittografia dei dati e dei nomi

Crypt è un backend wrapper sopra un altro remoto. Il contenuto del file viene cifrato con NaCl SecretBox basato su XSalsa20 e Poly1305; anche nomi di file e directory vengono trasformati per impostazione predefinita ([Crypt backend](https://rclone.org/crypt/)). Il provider vede ciphertext, pattern di dimensione, tempi di accesso e numero di oggetti; Crypt è quindi protezione dei contenuti e dei nomi, non anonimizzazione completa di traffico o metadati.

Password e salt opzionale sono memorizzati nella configurazione solo come «obscured», a meno che l'intera configurazione non sia cifrata. La perdita di entrambe le informazioni di recovery rende i dati illeggibili. Prima della rotazione o della modifica della crittografia dei nomi, viene testato un secondo remoto in lettura sugli stessi dati.

Crypt modifica dimensioni e possibilità di hash. Un normale `check` tra testo in chiaro e ciphertext non può quindi semplicemente aspettarsi lo stesso hash del provider. `cryptcheck` oppure un percorso di download controllato è il livello di verifica appropriato.

I comandi di copia operano esplicitamente su oggetti. Un mount VFS deve invece ricostruire operazioni continue del file system e per questo necessita di cache, modalità di scrittura e chiare ipotesi di coerenza.

## Mount VFS: l'object storage non è un file system POSIX

[`rclone mount`](https://rclone.org/commands/rclone_mount/) presenta un remoto come file system tramite FUSE o WinFsp. Il livello VFS traduce le operazioni sui file in operazioni sugli oggetti. Poiché gli oggetti cloud in genere non possono essere modificati nel mezzo né estesi arbitrariamente, Rclone mette in buffer o sostituisce gli oggetti.

Quattro modalità di cache definiscono contratti diversi:

| Modalità | File Cache locale | Comportamento di scrittura | Limite operativo |
|---|---|---|---|
| `off` | nessuna | diretto e limitato | nessun seek/read-write come per un file normale; upload fallito più difficile da ripetere |
| `minimal` | solo nei casi read-write necessari | parzialmente in buffer | non è possibile ogni semantica Open/Seek |
| `writes` | file in scrittura locali | upload dopo Close/Writeback | la cache locale non è ancora un commit remoto |
| `full` | letture e scritture locali, in parte sparse | alta compatibilità con il file system | monitorare spazio disco, eviction e coerenza della cache |

La Directory Cache mantiene in memoria nomi e metadati. Le modifiche esterne compaiono solo tramite polling o dopo scadenza o flush. Buffer è per file aperto; il fabbisogno massimo di memoria cresce con `buffer-size × open files`. Due processi non possono utilizzare la stessa cache VFS sovrapposta con File Caching attivo, perché ciò può danneggiare i dati ([VFS file caching](https://rclone.org/commands/rclone_mount/#vfs-file-caching)).

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

In Windows, [WinFsp](https://winfsp.dev/doc/WinFsp-Tutorial/) fornisce il ponte del file system; in Linux e altri sistemi Unix Rclone utilizza FUSE. Un servizio attende rete, configurazione e disponibilità del mount, dispone di un percorso di unmount controllato e monitora il volume di cache locale. I mount dei container richiedono diritti aggiuntivi per device e mount propagation e ampliano la superficie dei privilegi ([Rclone mount in Docker](https://rclone.org/install/#docker)).

## Bisync: due fonti di verità

[`rclone bisync`](https://rclone.org/bisync/) rileva modifiche su entrambi i lati in base a listing memorizzati e trasferisce i delta in entrambe le direzioni. Questo crea uno stato persistente del job al di fuori dei dati utente. Una directory di lavoro persa o incoerente può richiedere una fase di resync.

Bisync dispone di controlli di sicurezza per raggiungibilità e un numero insolitamente elevato di eliminazioni. Le modifiche ai filtri alterano l'insieme di dati visibile e impongono consapevolmente un resync. Se le applicazioni modificano il contenuto senza modificare dimensione e ModTime, il rilevamento standard può fallire. La sincronizzazione bidirezionale viene pertanto impiegata solo se sono documentati conflict policy, comportamento di clock/timestamp, Access Checks, Max-Delete, backup dei file di lavoro e risoluzione umana.

Bisync non è un file system multi-master con locking distribuito. Modifiche simultanee, rinomine e fasi offline rimangono conflitti funzionali.

## API Remote Control, metriche e modalità Serve

Con `--rc` oppure [`rclone rcd`](https://rclone.org/commands/rclone_rcd/), Rclone apre un'API HTTP. Può avviare job, configurare remoti, leggere o scrivere file locali e, in certe circostanze, eseguire comandi del sistema operativo nel contesto dell'utente rclone. La documentazione considera pertanto l'accesso a RC equivalente all'accesso shell ([Remote Control / API](https://rclone.org/rc/)).

RC ascolta per impostazione predefinita in locale. In caso di pubblicazione in rete sono necessarie autenticazione forte, TLS o un reverse proxy protetto, una allowlist firewall e privilegi di processo minimi. `--rc-no-auth` non deve essere esposto su un'interfaccia raggiungibile. L'endpoint OpenMetrics può esporre metriche di trasferimento e job; anche qui nomi remoti e percorsi possono essere metadati sensibili.

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

[`Invoke-WebRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest) e [`curl`](https://curl.se/docs/manpage.html) verificano l'endpoint HTTP locale. [`rclone rc`](https://rclone.org/commands/rclone_rc/) utilizza la stessa API e rileva i parametri di autenticazione dalla configurazione del job.

Rclone può inoltre offrire remoti tramite modalità Serve HTTP, WebDAV, SFTP, FTP, NFS, DLNA o compatibili S3. In tal modo il ruolo del sistema passa da client a server. Indirizzo listener, autenticazione, sicurezza del protocollo, cache VFS e diritti sui dati vengono quindi protetti come per un servizio autonomo ([Rclone serve](https://rclone.org/commands/rclone_serve/)).

## Logging, statistiche ed exit code

I log distinguono DEBUG, INFO, NOTICE ed ERROR. Per l'elaborazione automatica è disponibile il logging JSON. Le statistiche mostrano check, trasferimenti, errori, byte, velocità ed ETA. Un breve contatore `Transferred` non costituisce una prova di audit duratura; un job memorizza ora di inizio, root sorgente/destinazione, hash dei filtri, build Rclone, opzioni risolte non sensibili, statistiche finali ed exit code.

Gli exit code documentati distinguono successo, errori di sintassi, percorsi mancanti, errori temporanei, meno gravi e fatali, nonché limiti di trasferimento o tempo raggiunti ([List of exit codes](https://rclone.org/docs/#list-of-exit-codes)). `--error-on-no-transfer` può segnalare separatamente un'esecuzione riuscita senza trasferimento. Scheduler e monitoraggio trattano i codici in base al tipo di job; un job incrementale prevedibilmente vuoto va valutato diversamente da un export giornaliero mancato.

Un retry ad alto livello può correggere con successo messaggi di errore iniziali. Viceversa, un processo può trasferire molto e terminare nonzero a causa di pochi oggetti irrisolti. L'allarme si basa su stato finale, numero di errori, esaurimento dei retry, durata del job, heartbeat mancante e risultato della verifica.

## Sicurezza del percorso dati

Rclone eredita i diritti del proprio account di processo e delle credenziali remote. Un `sync` con diritti di Delete è un agente di eliminazione altamente privilegiato. Il principio del privilegio minimo separa sorgente in lettura, destinazione in scrittura, diritti di versioning/retention e azioni amministrative sui bucket.

La verifica del certificato TLS rimane attiva. `--no-check-certificate` disattiva la verifica della catena e del nome host ed è destinato dalla documentazione solo ai test ([TLS options](https://rclone.org/docs/#no-check-certificate)). I backend SFTP verificano le host key con una policy Known Hosts o fingerprint mantenuta; gli endpoint API non vengono reindirizzati tramite proxy non controllati.

Filtri, `--backup-dir`, versioning degli oggetti e Object Lock limitano rischi diversi. Una directory di backup nello stesso account non protegge dalla compromissione dell'account o dall'eliminazione da parte del provider. [Storage](/kb/storage), [Crittografia](/kb/verschluesselung) e [Hardening](/kb/haertung) trattano le superfici di controllo sottostanti.

La diagnosi separa percorso locale, filtro Rclone, API backend, autenticazione, rate limit e percorso di rete. Solo così diventa evidente se un file non è mai stato selezionato, non è stato trasferito o è stato rappresentato diversamente dalla destinazione.

## Diagnosi di rete e backend

Un messaggio di errore rclone può provenire da risoluzione dei nomi, TCP, TLS, proxy, autenticazione del provider, rate limit API, traduzione backend, file system locale o logica di confronto. La diagnosi segue il percorso:

1. Confermare binario, percorso della configurazione e nome remoto risolto.
2. Verificare DNS, porta e TLS dell'endpoint concreto.
3. Utilizzare `rclone lsd` oppure `lsjson` come test backend minimo in lettura.
4. Con `-vv` acquisire fase richiesta/retry e capacità backend senza pubblicare segreti.
5. Verificare sorgente/destinazione, filtri e decisione di fingerprint.
6. Correlare stato finale, exit code e audit log del provider.

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

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname), [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection), [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility), [`nc`](https://man.openbsd.org/nc), [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/), [`rclone lsd`](https://rclone.org/commands/rclone_lsd/) e [`rclone lsjson`](https://rclone.org/commands/rclone_lsjson/) separano trasporto, autenticazione, listing e visione degli oggetti. Prima della condivisione, i dump degli header vengono controllati per token, firme, cookie e percorsi. [DNS](/kb/dns), [TCP](/kb/tcp), [TLS](/kb/tls) e [Troubleshooting](/kb/troubleshooting) approfondiscono le fasi.

## Errori tipici

Rclone spesso segnala il passaggio in cui un job fallisce visibilmente, non automaticamente la causa originaria. La tabella collega quindi sintomo, causa probabile e prova che conferma o confuta l'ipotesi.

| Sintomo | Causa probabile | Prova | Falsa soluzione rapida |
|---|---|---|---|
| I file vengono copiati nuovamente a ogni esecuzione | ModTime/hash non comuni o precisione differente | capacità backend, `-vv`, report Check | impostare ciecamente `--size-only` |
| Gli oggetti di destinazione scompaiono | direzione Sync, filtro oppure opzione Delete | report Dry Run/Combined, definizione del job | riavviare semplicemente il job |
| Il mount mostra nomi vecchi | Directory Cache o polling mancante | statistiche VFS, RC `vfs/forget`, listing del provider | terminare forzatamente il processo |
| La scrittura sul mount manca nel remoto | file aperto, Writeback in attesa, cache piena | cache VFS e log | eliminare la cache locale |
| Il controllo hash non è possibile | nessun hash comune, Crypt o semantica multipart | panoramica backend, `hashsum`, `check --download` | assumere ETag come MD5 |
| RC risponde esternamente | indirizzo listener troppo ampio/autenticazione assente | elenco socket, firewall, flag RC | mantenere segreto solo l'URL |
| Bisync si interrompe dopo una modifica del filtro | lo stato di lavoro protegge da eliminazioni errate | log Bisync e hash del filtro | eliminare i file di lavoro e avviare normalmente |
| Exit code nonzero nonostante molti file trasferiti | errori residui dopo i retry | log finale, report errori | considerare il contatore dei trasferimenti come successo |

## Backup, recovery e reconciliation

Rclone fa spesso parte di un backup, ma non è esso stesso un modello di backup immutabile. `sync` replica eliminazioni e danni da crittografia. `copy` conserva oggetti aggiuntivi nella destinazione, ma può sovrascrivere versioni esistenti. La protezione deriva da versioning della destinazione, retention/Object Lock, credenziali separate, `--backup-dir`, snapshot immutabili e test di ripristino.

Il set di recovery comprende:

- binario rclone approvato o artefatto di release verificabile;
- `rclone.conf` cifrato più segreto di decrittazione separato;
- credenziali del provider e relativa riacquisizione;
- password e salt Crypt;
- definizioni di job, filtri e ambiente;
- file di lavoro Bisync;
- oggetti della cache VFS non caricati e relativa associazione;
- report Dry Run, Combined, Check ed exit code;
- procedure di versioning, retention e ripristino della destinazione.

Un ripristino viene eseguito in un percorso separato e quindi verificato con hash, numero, dimensione, campione di metadati e test dell'applicazione. La reconciliation confronta Source of Truth, versioni di destinazione, Writeback VFS e stato dello scheduler. [Backup e Disaster Recovery](/kb/backup-dr) descrive RPO/RTO e accettazione trasversale.

## Storia tecnica

Nick Craig-Wood ha avviato il progetto con un primo commit nel novembre 2012. Il fondatore ha documentato questa origine al quattordicesimo anniversario e ha collegato lo stato iniziale dei sorgenti ([Happy 14th Birthday rclone](https://forum.rclone.org/t/happy-14th-birthday-rclone/53005), [initial commit](https://github.com/rclone/rclone/commit/e9ae4f89a44b1cac8e83ede1572f15dace692c67)).

Rclone ha adottato la nota sintassi direzionale di `rsync`, ma ha sostituito le sue ipotesi delta basate su blocchi con operazioni su oggetti e provider. Il cloud storage non offre normalmente un'interfaccia server-side per rolling checksum rsync; Rclone confronta oggetti completi tramite le impronte disponibili e li trasferisce di conseguenza.

Con l'aumento del numero di backend, si è evoluto il modello centrale di interfaccia Fs/Object con funzioni opzionali. Crypt ha aggiunto la crittografia lato client; Mount e VFS hanno reso l'object storage accessibile alle applicazioni basate su file; l'API Remote Control è stata notevolmente ampliata nel 2018; Bisync ha aggiunto il confronto bidirezionale persistente. Il [Changelog](https://rclone.org/changelog/) documenta questa evoluzione e le correzioni rilevanti per la sicurezza. Per l'esercizio rimane quindi l'integrazione dipendente dal tempo a questo articolo architetturale.

La linea tecnica resta coerente: un binario Go portabile, molti backend integrati, comandi comuni e differenze tra provider esplicitamente trasparenti. Sono proprio queste differenze a rendere Rclone potente per gli amministratori e al tempo stesso soggetto a revisione.

## Checklist amministrativa

Prima di un job produttivo devono essere comprovati almeno i seguenti punti:

- Origine del binario, percorso di configurazione e risoluzione del remoto sono univoci.
- Sorgente, destinazione, direzione e albero di destinazione previsto sono indicati nel runbook.
- Le capacità backend per hash, ModTime, Case, Metadata, Copy/Move e listing sono verificate.
- File di filtro e parametri del job sono versionati; Dry Run e Combined Report sono stati approvati.
- Le policy Delete, Max-Delete, Backup-Dir, versioning e retention sono coerenti.
- Le credenziali hanno solo i diritti necessari; configurazione, password Crypt e salt sono recuperabili.
- Transfers, Checkers, Pacer e larghezza di banda rispettano host e provider.
- La cache VFS dispone di volume dedicato, allarme di capacità, arresto ordinato e verifica Writeback.
- RC/Serve ascolta solo in modo controllato ed è protetto come accesso shell.
- Il successo è comprovato da stato finale, exit code, report Check e test di ripristino.

## Fonti

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
