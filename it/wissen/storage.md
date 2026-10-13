---
title: "Storage: blocchi, file, oggetti e integrità dei dati"
blatt: "storage"
description: "Storage per amministratori di infrastrutture e messaggistica: archiviazione a blocchi, file e oggetti, percorso I/O e cache, file system, RAID ed Erasure Coding, replica e coerenza, snapshot, integrità, crittografia, capacità, prestazioni, modelli cloud, recovery e storia tecnica."
fakten:
  - label: Modelli fondamentali
    wert: Blocco · file · oggetto; indirizzamento, metadati e coerenza differenti
    href: https://www.snia.org/education/storage_networking_primer
  - label: Percorso a blocchi
    wert: Applicazione · file system/Volume Manager · block layer · controller · supporto
    href: https://docs.kernel.org/block/index.html
  - label: Accesso ai file
    wert: locale tramite VFS oppure remoto, tra l’altro tramite NFS e SMB
    href: https://www.rfc-editor.org/rfc/rfc8881.html
  - label: Modello a oggetti
    wert: Bucket/namespace · chiave · dati · metadati · operazione API
    href: https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html
  - label: Durabilità
    wert: solo dopo il limite di commit/flush definito; una Write Call riuscita da sola può non bastare
    href: https://man7.org/linux/man-pages/man2/fsync.2.html
  - label: Ridondanza
    wert: Mirror/RAID/Erasure Coding/replica aumentano la tolleranza ai guasti, ma non sono un backup
    href: https://www.snia.org/sites/default/files/technical-work/whitepapers/SNIA-Data-Protection-Best-Practices-White%20Paper.pdf
  - label: Coerenza
    wert: distinguere tra vista locale, distribuita e dell’applicazione; documentare Sync, Quorum e Lag
    href: https://www.cs.cornell.edu/fbs/publications/SMSurvey.pdf
  - label: Integrità
    wert: checksum · verifica end-to-end · scrub · percorso di riparazione
    href: https://openzfs.github.io/openzfs-docs/Basic%20Concepts/Checksums.html
  - label: Snapshot
    wert: riferimento temporale nello stesso dominio di errore e amministrativo; la coerenza deve essere stabilita
    href: https://docs.kernel.org/filesystems/fiemap.html
  - label: Dimensioni delle prestazioni
    wert: latenza · IOPS · throughput · Queue Depth · dimensione del blocco · mix lettura/scrittura
    href: https://fio.readthedocs.io/en/latest/fio_doc.html
  - label: Dimensioni della capacità
    wert: byte utilizzabili · spazio riservato · metadati · inode/oggetti · Thin Provisioning
    href: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/htmlsingle/configuring_and_managing_logical_volumes/configuring-lvm-on-san-one-machine_configuring-lvm-on-shared-storage
  - label: Oggetto di recovery
    wert: dati, metadati, autorizzazioni, chiavi, versione/snapshot, schema applicativo e dipendenze
    href: https://csrc.nist.gov/pubs/sp/800/209/final
werbung:
  - newsletter
ctaThemen:
  - rclone
  - proton-drive
  - backup-dr
translationSourceHash: 97205e641b2bc37a7cbb4a5e7de137f527187daf42e5da7e951bd18246a3cede
translationModel: gpt-5.6-terra
translatedAt: 2026-10-13T11:33:10.897Z
translationReview: automatic
---

# Storage: blocchi, file, oggetti e integrità dei dati

Lo storage è la base di stato di quasi ogni infrastruttura: mailbox, code, indici di ricerca, directory, database, macchine virtuali e backup consistono di byte più metadati e di una garanzia su quando le modifiche sono visibili e permanenti. Il solo supporto non fornisce questa garanzia. Applicazione, cache del sistema operativo, file system, Volume Manager, controller, rete, meccanismi di ridondanza e replica remota formano insieme il percorso dei dati.

Per gli amministratori, prima di qualsiasi scelta di prodotto, tre domande sono quindi più importanti della capacità e degli IOPS nominali: **Quale modello di accesso vede l’applicazione? Quale semantica di commit e coerenza si applica? Quali domini di guasto sopravvive una modifica confermata?** Solo dopo ha senso valutare prestazioni, backup, crittografia e costi.

La spiegazione segue un’operazione di scrittura dall’applicazione, attraverso il file system o l’API degli oggetti, fino a volume, controller e supporto. Solo successivamente vengono confrontati replica, coerenza, prestazioni, crittografia, backup e recovery.

Lo storage inizia con una richiesta di scrittura dell’applicazione e termina solo quando i dati sono permanenti e nuovamente leggibili secondo il modello garantito. Nel mezzo vi sono cache, file system o API degli oggetti, rete, controller e supporto. L’articolo segue questo percorso dei dati anziché limitarsi a elencare i tipi di storage.

## Inquadramento tecnico e percorso dei dati

Lo storage viene classificato in base all’interfaccia ricevuta da un consumer:

- **Storage a blocchi** indirizza aree di un dispositivo logico. Il file system o il database possiede la struttura superiore.
- **Storage di file** espone nomi gerarchici, directory, attributi, autorizzazioni e operazioni come apertura, lettura o ridenominazione.
- **Storage di oggetti** indirizza oggetti tramite namespace/bucket e chiave; dati e metadati sono gestiti tramite operazioni API.

SNIA descrive Block, File e Object Storage come modelli diversi di servizio e accesso, non semplicemente come tre classi di prodotto ([SNIA – Storage Networking Primer](https://www.snia.org/education/storage_networking_primer)). Un gateway S3 davanti a un file system, ad esempio, modifica il modello esterno a oggetti senza che la memorizzazione interna debba necessariamente essere basata su oggetti.

Un percorso di scrittura sincrono può attraversare i seguenti livelli:

1. applicazione e runtime con buffering proprio o Write-ahead Log;
2. chiamata di sistema, Virtual File System o accesso raw a blocchi;
3. Page Cache, file system e Volume Manager;
4. block layer del kernel e scheduler I/O;
5. driver di dispositivo, HBA, protocollo di rete o livello di storage virtuale;
6. cache del controller, livello di ridondanza/Erasure Coding e supporto;
7. repliche, journal, quorum o sede remota.

Il block layer Linux documenta code, richieste e topologia dei dispositivi come livello separato del kernel ([Linux Kernel – Block subsystem](https://docs.kernel.org/block/index.html)). Anche Windows astrae separatamente dischi fisici, Storage Pool, dischi virtuali, partizioni, volumi e file system ([Microsoft Learn – Storage overview](https://learn.microsoft.com/en-us/windows-server/storage/storage)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-storage.svg?v=20260813" title="Interaktive Infografik: Storage-Datenpfad von Anwendung und Cache über Dateisystem, Volume, Controller und Medium bis zu Replikation sowie Block-, Datei-, Objekt-, Integritäts- und Recoverymodellen" loading="lazy">
  <a href="/images/kb-interaktiv-storage.svg?v=20260813">Apri direttamente il grafico interattivo</a>.
</iframe>

## Storage a blocchi

Lo storage a blocchi fornisce uno spazio di indirizzamento logico lineare. LUN SCSI, target iSCSI, volume Fibre Channel, namespace NVMe, disco virtuale e volume cloud sono trasporti o implementazioni diversi dello stesso modello di base. L’iniziatore vede blocchi; nomi dei file, directory e spesso anche la semantica di blocco vengono creati a un livello superiore.

iSCSI incapsula comandi SCSI su TCP e separa discovery/login, sessione e connessioni. RFC 7143 definisce la semantica del protocollo; multipathing, autenticazione, progettazione della rete e capacità del file system in cluster restano ulteriori livelli operativi ([RFC 7143 – iSCSI](https://www.rfc-editor.org/rfc/rfc7143.html)). NVMe è stato progettato per storage non volatile su PCIe e utilizza Submission/Completion Queues; NVMe over Fabrics trasferisce il modello, tra l’altro, tramite RDMA o TCP ([NVM Express Specifications](https://nvmexpress.org/specifications/)).

Un volume a blocchi non deve essere montato simultaneamente in scrittura da più host con normali file system locali. L’uso condiviso a blocchi richiede un file system in cluster o un’applicazione con coordinamento distribuito proprio. «La SAN presenta la LUN a entrambi i server» non costituisce un blocco di scrittura.

## File system e VFS

Un file system assegna aree a blocchi o altri backend a file, directory e metadati. Il Virtual File System fornisce alle applicazioni un’interfaccia comune di chiamate di sistema; i file system concreti implementano in modo diverso operazioni, caching e persistenza ([Linux Kernel – Overview of the Linux Virtual File System](https://docs.kernel.org/filesystems/vfs.html)).

Il journaling registra metadati o anche dati aggiuntivi per ripristinare uno stato coerente del file system dopo un crash. Non è un backup e non impedisce che una modifica tecnicamente errata venga registrata correttamente nel journal. ext4 documenta modalità quali `journal`, `ordered` e `writeback`; XFS utilizza un journal dei metadati e meccanismi di recovery propri ([Linux Kernel – ext4](https://docs.kernel.org/filesystems/ext4/index.html), [XFS Documentation](https://docs.kernel.org/filesystems/xfs/index.html)).

I file system Copy-on-write quali Btrfs e OpenZFS scrivono i blocchi modificati in nuove posizioni e aggiornano i riferimenti. Ciò consente snapshot efficienti e modelli di checksum, ma richiede spazio per nuovi blocchi, metadati e garbage collection ([Btrfs Documentation](https://btrfs.readthedocs.io/en/latest/), [OpenZFS – Basic Concepts](https://openzfs.github.io/openzfs-docs/Basic%20Concepts/index.html)).

## File remoti: NFS e SMB

NFSv4.1 definisce un protocollo di file remoti con stato, con sessioni, aperture, lock, deleghe e un’operazione compound. Cache del client e recovery del server sono parte del protocollo; un mount NFS non è semplicemente «POSIX locale su TCP» ([RFC 8881 – NFSv4.1](https://www.rfc-editor.org/rfc/rfc8881.html)).

SMB 2/3 utilizza crediti, sessioni, Tree Connects, aperture e Leases/Oplocks. Microsoft pubblica la specifica del protocollo; funzionalità quali Multichannel, Transparent Failover, Encryption e Persistent Handles modificano le prestazioni e la semantica di failover ([Microsoft Open Specifications – MS-SMB2](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-smb2/5606ad47-5ee0-437a-817e-70c366052962)).

Per i file remoti si documentano almeno:

- risoluzione dei nomi, destinazione/VIP e porte;
- autenticazione e mappatura delle identità, ad esempio Kerberos, NTLM o UID/GID;
- dialect/versione del protocollo, opzioni di mount e crittografia;
- comportamento di cache, lock, lease e failover;
- semantica in caso di disconnessione di rete, timeout e recovery del server;
- quote, snapshot e responsabilità del backup.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Dateifreigabe-Inventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-SmbConnection | Select-Object ServerName,ShareName,Dialect,Encrypted,ContinuouslyAvailable
Get-SmbMapping
Test-NetConnection $env:FILESERVER -Port 445
Get-Volume | Sort-Object DriveLetter</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">findmnt --json | jq '.filesystems[] | select(.fstype | test("nfs|cifs"))'
nfsstat --client
smbclient --list="//$FILESERVER" --use-kerberos=required
nc -vz "$FILESERVER" 445</code></pre>
  </div>
</div>

[`Get-SmbConnection`](https://learn.microsoft.com/en-us/powershell/module/smbshare/get-smbconnection) e [`Get-SmbMapping`](https://learn.microsoft.com/en-us/powershell/module/smbshare/get-smbmapping) mostrano i dialect SMB attivi e i mapping; [`Get-Volume`](https://learn.microsoft.com/en-us/powershell/module/storage/get-volume) integra la vista locale dei volumi. [`findmnt`](https://man7.org/linux/man-pages/man8/findmnt.8.html), [`nfsstat`](https://man7.org/linux/man-pages/man8/nfsstat.8.html) e [`smbclient`](https://www.samba.org/samba/docs/current/man-html/smbclient.1.html) verificano mount Unix e viste del protocollo; [`jq`](https://jqlang.org/manual/) filtra il loro output JSON. [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) e [`nc`](https://man.openbsd.org/nc) attestano solo il percorso TCP, non l’autorizzazione della share né la semantica I/O.

## Storage di oggetti

Lo storage di oggetti non utilizza file handle aperti nel senso classico. Un’operazione legge, scrive, elenca, copia o elimina un oggetto sotto una chiave. Le «cartelle» sono spesso prefissi o proiezioni dell’interfaccia utente. Un rename può quindi essere implementato come copia lato server più eliminazione. AWS documenta l’API S3 come interfaccia orientata a REST su bucket, oggetti e operazioni ([Amazon S3 API Reference](https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html)).

Le operazioni sugli oggetti hanno limiti propri:

- Multipart Upload crea parti e un commit finale; gli upload interrotti richiedono una pulizia tramite Lifecycle.
- ETag non è universalmente un checksum MD5, in particolare per oggetti Multipart o crittografati.
- Listing, Versioning, Delete Marker e Lifecycle sono stati distinti.
- Le URL pre-firmate delegano un’operazione concreta fino alla scadenza e vanno trattate come credenziali.
- Object Lock/WORM richiede un modello di retention, Legal Hold, chiavi e amministrazione.

Amazon S3 documenta una forte coerenza read-after-write e di list per PUT e Delete riusciti. Questa garanzia di prodotto non si applica automaticamente a implementazioni compatibili con S3, cache o gateway sovrapposti ([Amazon S3 – Data consistency model](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html#ConsistencyModel)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Objektspeicher-Inventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">aws s3api head-bucket --bucket $env:BUCKET
aws s3api get-bucket-versioning --bucket $env:BUCKET
aws s3api get-object-attributes --bucket $env:BUCKET --key $env:OBJECT_KEY `
  --object-attributes ETag,Checksum,ObjectSize,StorageClass
rclone size $env:REMOTE</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">aws s3api head-bucket --bucket "$BUCKET"
aws s3api get-bucket-versioning --bucket "$BUCKET"
aws s3api get-object-attributes --bucket "$BUCKET" --key "$OBJECT_KEY" \
  --object-attributes ETag,Checksum,ObjectSize,StorageClass | jq .
rclone size "$REMOTE"</code></pre>
  </div>
</div>

[`aws s3api`](https://docs.aws.amazon.com/cli/latest/reference/s3api/) opera su operazioni API S3; i sottocomandi mostrati verificano accesso al bucket, versioning e attributi dell’oggetto. [`rclone size`](https://rclone.org/commands/rclone_size/) conta oggetti e byte tramite un backend configurato. Entrambi mostrano la vista delle rispettive credenziali e del relativo endpoint; regione, account e alias vengono pertanto registrati.

Che si tratti di blocchi, file o oggetti, un’operazione di scrittura confermata deve essere compresa nel contesto di cache e flush. «Scrittura riuscita» non indica in ogni stack lo stesso punto di durabilità fisica.

## Cache, flush e durabilità

Una chiamata `write()` riuscita può significare soltanto che i byte sono stati copiati in una cache del sistema operativo. I sistemi di tipo POSIX utilizzano `fsync()` oppure `fdatasync()` per trasferire al dispositivo dati modificati e metadati necessari. La manpage Linux segnala che, per un file appena creato, può inoltre essere necessario sincronizzare la directory contenente affinché la directory entry sia permanente ([fsync(2)](https://man7.org/linux/man-pages/man2/fsync.2.html)).

A un livello inferiore, controller e supporto devono implementare correttamente Flush/FUA. Una Write Cache protetta da batteria o flash può preservare le scritture confermate in caso di perdita di alimentazione; una cache volatile senza protezione può simulare prestazioni. Virtualizzazione e storage di rete allungano la catena. L’amministratore necessita della **garanzia di commit end-to-end**, non soltanto di una casella «Write Cache» attivata.

I database utilizzano Write-ahead Logging, checksum e protocolli di recovery, perché un commit del file system non definisce una transazione applicativa. Uno snapshot o una replica diventano coerenti con l’applicazione soltanto se questa utilizza un meccanismo supportato di freeze, backup o posizionamento del log.

## Coerenza nei sistemi di storage distribuiti

I sistemi distribuiti replicano dati tra processi e domini di guasto. Un’operazione può essere confermata su un nodo, ancora invisibile su un altro o non decidibile in caso di partizione di rete. I modelli di coerenza descrivono quali sequenze di osservazione i client possono aspettarsi. Una panoramica di Viotti e Vukolić classifica i modelli forti e più deboli in base alle garanzie sulla cronologia ([Consistency in Non-Transactional Distributed Storage Systems](https://www.cs.cornell.edu/fbs/publications/SMSurvey.pdf)).

I parametri pratici sono:

- numero e collocazione delle repliche oppure dei frammenti Erasure Coding;
- quorum di scrittura e lettura nonché soglia di acknowledgement;
- replica tra sedi sincrona o asincrona;
- comportamento in caso di Split Brain, maggioranza persa e rejoin;
- repair, backfill, rebalance e scrub;
- garanzia di staleness, read-after-write e lettura monotona;
- risoluzione dei conflitti e sovranità dei dati preferita.

Ceph descrive CRUSH come posizionamento deterministico degli oggetti su Failure Domain gerarchici e RADOS come base a oggetti distribuita per interfacce a blocchi, file e oggetti ([Ceph Architecture](https://docs.ceph.com/en/latest/architecture/)). La configurazione concreta di pool, repliche, Erasure Coding e Failure Domain determina quali guasti il sistema sopravvive effettivamente.

La coerenza descrive la vista sui dati; la ridondanza protegge dal guasto di singoli componenti. RAID ed Erasure Coding aumentano la disponibilità, ma non sostituiscono un backup indipendente.

## RAID, Erasure Coding e rebuild

RAID combina più dispositivi per ottenere caratteristiche di disponibilità o prestazioni. Il mirroring conserva copie complete; il RAID con parità calcola informazioni di ridondanza. Il classico lavoro di Berkeley ha definito i livelli RAID come alternative architetturali ([Patterson, Gibson, Katz – A Case for Redundant Arrays of Inexpensive Disks](https://www.cs.cmu.edu/~garth/RAIDpaper/Patterson88.pdf)).

Erasure Coding suddivide i dati in frammenti di dati e parità; l’oggetto può essere ricostruito da un numero minimo di essi. Risparmia capacità rispetto alla replica completa, ma aumenta i costi di codifica, rete e repair. Le piccole scritture casuali possono essere particolarmente costose.

RAID e replica non proteggono da:

- eliminazione logica o crittografia da parte di software autorizzato;
- corruzione silenziosa replicata correttamente;
- credenziali compromesse dell’amministratore o del Control Plane;
- errori al di fuori del Failure Domain selezionato;
- stati incompatibili di software/metadati;
- perdita delle chiavi di crittografia.

SNIA descrive RAID come algoritmo di protezione dei dati su più dispositivi indipendenti e tratta il backup come tecnica di protezione distinta; dal RAID non deriva quindi alcuna semantica di backup ([SNIA – Data Protection Best Practices](https://www.snia.org/sites/default/files/technical-work/whitepapers/SNIA-Data-Protection-Best-Practices-White%20Paper.pdf)). Durante rebuild o backfill il margine di sicurezza diminuisce, mentre latenza e probabilità di guasto possono aumentare. Avanzamento, ridondanza residua e tasso di errore devono far parte del monitoraggio.

## Snapshot e clone

Uno snapshot è un riferimento a uno stato in un determinato momento. Copy-on-write o Redirect-on-write memorizza separatamente le modifiche successive; un clone può creare un ramo scrivibile dalla stessa base a blocchi. Gli snapshot sono rapidi e utili per punti di recovery, test e replica.

Tuttavia, spesso condividono hardware di storage, Control Plane, credenziali e dominio di guasto con la sorgente. La perdita di un pool o un amministratore dello storage compromesso possono colpire insieme sorgente e snapshot. Le catene di snapshot consumano metadati e capacità; l’eliminazione di uno snapshot precedente può generare notevole lavoro in background.

La coerenza da crash corrisponde approssimativamente a un’improvvisa perdita di alimentazione. La coerenza dell’applicazione richiede flush, quiesce, freeze del file system, VSS o un’API di backup specifica del prodotto. Microsoft VSS coordina requester, writer e provider per Shadow Copies coerenti in Windows ([Microsoft Learn – Volume Shadow Copy Service](https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service)).

## Checksum, scrub e integrità end-to-end

I checksum rilevano modifiche solo quando esiste un valore atteso affidabile e l’area verificata è chiaramente definita. CRC di trasporto, ECC del dispositivo, checksum del file system, checksum dell’oggetto e hash dell’applicazione proteggono tratti differenti. Un ETag o l’ora del file non è una prova universale di integrità.

OpenZFS memorizza checksum per dati e metadati nei puntatori ai blocchi superiori e, in presenza di ridondanza, può rilevare e riparare una copia errata ([OpenZFS – Checksums](https://openzfs.github.io/openzfs-docs/Basic%20Concepts/Checksums.html)). Btrfs offre checksum e scrub sui propri dispositivi; lo scrub legge dati e metadati, verifica e ripara se è presente una copia valida ([Btrfs – Scrub](https://btrfs.readthedocs.io/en/latest/Scrub.html)).

Uno scrub genera carico I/O e richiede finestre temporali, prioritizzazione e allarmi. «Scrub completato» è significativo solo se vengono valutati errori non correggibili, errori corretti, aree saltate e oggetti interessati.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Integritätsprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-FileHash -Algorithm SHA256 $env:SOURCE_FILE
Get-FileHash -Algorithm SHA256 $env:RESTORED_FILE
Get-StorageReliabilityCounter -PhysicalDisk (Get-PhysicalDisk)
Repair-Volume -DriveLetter $env:DRIVE_LETTER -Scan</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sha256sum "$SOURCE_FILE" "$RESTORED_FILE"
smartctl --xall "$BLOCK_DEVICE"
zpool status -v "$ZPOOL"
btrfs scrub status "$BTRFS_MOUNT"</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash) e [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) confrontano file controllati. [`Get-StorageReliabilityCounter`](https://learn.microsoft.com/en-us/powershell/module/storage/get-storagereliabilitycounter) e [`smartctl`](https://www.smartmontools.org/browser/trunk/smartmontools/smartctl.8.in) leggono dati sullo stato di salute dei dispositivi; [`Repair-Volume`](https://learn.microsoft.com/en-us/powershell/module/storage/repair-volume), [`zpool status`](https://openzfs.github.io/openzfs-docs/man/master/8/zpool-status.8.html) e [`btrfs scrub`](https://btrfs.readthedocs.io/en/latest/btrfs-scrub.html) mostrano le prospettive del file system/pool. Questi strumenti devono essere eseguiti solo su destinazioni chiaramente identificate.

## Thin Provisioning, Discard e capacità

Thin Provisioning presenta più spazio logico di quello fisicamente allocato. Ciò aumenta l’utilizzo, ma crea un limite di guasto aggiuntivo: un host vede spazio libero mentre pool o area dei metadati sono esauriti. Crescita, consumo degli snapshot, riserve e reclaim devono essere monitorati insieme ([Red Hat – Thinly-provisioned logical volumes](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/htmlsingle/configuring_and_managing_logical_volumes/configuring-lvm-on-san-one-machine_configuring-lvm-on-shared-storage)).

TRIM/UNMAP/Discard segnala ai livelli sottostanti blocchi non più necessari. Se l’informazione raggiunge l’array o il backend cloud dipende da file system, opzioni di mount, hypervisor, protocollo e dispositivo. Il reclaim può influire su prestazioni e protezione dei dati; nei volumi crittografati, Discard può rendere visibili i modelli di occupazione.

Il monitoraggio della capacità comprende:

- byte logici e fisici nonché overcommit;
- blocchi del file system, inode o numero di oggetti;
- spazio per snapshot, journal, WAL e metadati;
- Reserved Space, riserva per rebuild/backfill e quota;
- trash, Delete Marker, versioni precedenti e residui Multipart;
- crescita per classe di dati e momento fino all’esaurimento.

## HDD, flash e NVMe

Gli HDD hanno tempi di posizionamento meccanici; I/O casuali e throughput sequenziale differiscono notevolmente. Gli SSD utilizzano Flash Translation Layer, Garbage Collection, Over-Provisioning e Wear Leveling. Write amplification e durata residua dipendono dal workload e dal firmware. NVMe riduce i colli di bottiglia del protocollo e delle code, ma non elimina le questioni relative al supporto o alla durabilità.

I valori SMART/NVMe Health sono segnali specifici del produttore, non una garanzia. Avvisi critici, Media/Data Integrity Errors, Unsafe Shutdowns, temperatura, Spare e Percentage Used vengono valutati insieme a errori dell’applicazione e log del controller. Il progetto NVMe pubblica le specifiche per controller, gestione e trasporti ([NVM Express Specifications](https://nvmexpress.org/specifications/)).

## Crittografia e confini delle chiavi

La crittografia dello storage può avvenire sul supporto, nel controller, nel volume, nel file system, nell’oggetto o nell’applicazione. Ogni livello protegge da un diverso modello di minaccia:

| Livello | Protegge principalmente | Limite residuo |
|---|---|---|
| Self-Encrypting Drive | supporto rimosso con locking corretto | l’host vede il testo in chiaro; gestione di chiavi/firmware |
| Volume Encryption | blocchi letti offline | host in esecuzione e processi privilegiati vedono il testo in chiaro |
| File system/file | namespace e utenti selezionati | i metadati possono restare visibili |
| KMS lato server | supporti/backend del provider | Control Plane e IAM del provider |
| crittografia di oggetti lato client | il provider vede il ciphertext | ricerca/anteprima, perdita delle chiavi, metadati e migrazione |

NIST SP 800-209 richiede per le infrastrutture di storage, tra l’altro, controllo degli accessi, gestione sicura, integrità dei dati, crittografia, isolamento e monitoraggio come modello di sicurezza coerente ([NIST SP 800-209 – Security Guidelines for Storage Infrastructure](https://csrc.nist.gov/pubs/sp/800/209/final)).

Key Rotation e restore vengono testati insieme. Un backup senza le chiavi e i metadati validi al momento del salvataggio non è ripristinabile; anche una chiave esportata senza contesto IAM/wrapping può essere inutilizzabile.

La sola capacità dice poco sul comportamento di un’applicazione. Determinanti sono latenza, IOPS, throughput e code lungo l’intero percorso dei dati.

## Prestazioni: latenza, IOPS, throughput e coda

Gli IOPS nominali senza dimensione del blocco, mix lettura/scrittura, schema di accesso, Queue Depth, sincronia e percentili di latenza non sono confrontabili. Il throughput risulta approssimativamente da IOPS per dimensione del trasferimento, finché non dominano i limiti di bus, rete, controller o supporto. I cache hit possono aggirare completamente il supporto.

Si misurano almeno:

- latenza end-to-end dal punto di vista dell’applicazione, inclusi p95/p99;
- IOPS e byte/s distinti tra lettura e scrittura;
- Queue Depth, utilizzo e tempo di attesa per dispositivo/volume;
- latenza delle scritture sincrone e frequenza di flush;
- cache hit, Dirty Data e throttling;
- lag di backend/replica, rebuild o garbage collection;
- errori, retry, timeout e failover del percorso.

`fio` può generare workload I/O controllati; la sua documentazione avverte implicitamente tramite le opzioni di scrittura flessibili: una destinazione errata distrugge i dati. I test vengono eseguiti solo su file di test dedicati o dispositivi autorizzati con percorso verificato ([fio Documentation](https://fio.readthedocs.io/en/latest/fio_doc.html)). DiskSpd di Microsoft è un corrispondente strumento di storage sintetico per Windows ([Microsoft DiskSpd](https://github.com/microsoft/diskspd)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Storage-Performanceanalyse">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Counter '\PhysicalDisk(*)\Avg. Disk sec/Transfer','\PhysicalDisk(*)\Disk Transfers/sec' `
  -SampleInterval 2 -MaxSamples 15
diskspd.exe -c2G -d30 -r -w30 -b8K -o4 -t2 $env:TEST_FILE
Get-PhysicalDisk | Get-StorageReliabilityCounter</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">iostat -x -d 2 15
fio --name=controlled-test --filename="$TEST_FILE" --size=2G \
  --rw=randrw --rwmixread=70 --bs=8k --iodepth=4 --runtime=30 --time_based
pidstat -d 2 15</code></pre>
  </div>
</div>

[`Get-Counter`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-counter), [`iostat`](https://man7.org/linux/man-pages/man1/iostat.1.html) e [`pidstat`](https://man7.org/linux/man-pages/man1/pidstat.1.html) osservano il carico reale. [`diskspd`](https://github.com/microsoft/diskspd/wiki/Command-line-and-parameters) e [`fio`](https://fio.readthedocs.io/en/latest/fio_doc.html) generano carico sintetico e non vengono mai indirizzati verso dispositivi a blocchi produttivi o file non protetti.

## Multipathing e Failure Domain

Più percorsi possono sopravvivere a guasti di controller, HBA, switch o fabric se sono effettivamente indipendenti. Il multipathing rileva più vie verso lo stesso dispositivo logico ed esegue failover o load balancing. Il solo numero di percorsi non dimostra la ridondanza: due sessioni sullo stesso switch e controller continuano a guastarsi insieme.

Asymmetric Logical Unit Access può identificare percorsi del controller preferiti e non preferiti. I valori di timeout devono essere coerenti tra applicazione, file system, multipath, trasporto e array; timeout troppo brevi generano falsi guasti, mentre timeout troppo lunghi bloccano le code.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Block- und Multipathinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Disk | Sort-Object Number
Get-PhysicalDisk | Sort-Object FriendlyName
Get-MPIOPath
Get-IscsiSession | Format-List *</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">lsblk --json -o NAME,KNAME,TYPE,SIZE,FSTYPE,MOUNTPOINTS,MODEL,SERIAL,WWN | jq .
multipath -ll
iscsiadm -m session -P 3
nvme list -o json | jq .</code></pre>
  </div>
</div>

[`Get-Disk`](https://learn.microsoft.com/en-us/powershell/module/storage/get-disk), [`Get-PhysicalDisk`](https://learn.microsoft.com/en-us/powershell/module/storage/get-physicaldisk), [`Get-MPIOPath`](https://learn.microsoft.com/en-us/powershell/module/mpio/get-mpiopath) e [`Get-IscsiSession`](https://learn.microsoft.com/en-us/powershell/module/iscsi/get-iscsisession) mostrano dispositivi e percorsi Windows. [`lsblk`](https://man7.org/linux/man-pages/man8/lsblk.8.html), [`multipath`](https://manpages.debian.org/bookworm/multipath-tools/multipath.8.en.html), [`iscsiadm`](https://manpages.debian.org/bookworm/open-iscsi/iscsiadm.8.en.html) e [`nvme`](https://github.com/linux-nvme/nvme-cli) mostrano mappatura Linux, sessioni e namespace. Numero di serie, WWN, LUN e mountpoint vengono ricontrollati prima di ogni modifica.

## Classi di storage cloud e Lifecycle

Lo storage di oggetti cloud offre classi con prezzi, conservazione minima, costi di recupero e tempi di ripristino differenti. L’archivio può essere reidratato in modo asincrono; le Lifecycle Policy possono spostare oggetti tra classi o eliminare versioni. Un costo di capacità basso senza prezzo di request, retrieval, egress e durata minima è incompleto.

AWS documenta le Storage Classes con schemi di accesso e limitazioni; altri provider utilizzano classi e garanzie proprie ([Amazon S3 Storage Classes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html)). Per un piano di recovery vengono quindi testati per ogni classe di dati RPO, RTO, larghezza di banda di ripristino, tasso di request, costi di egress e disponibilità delle chiavi.

La replica mantiene ulteriori copie disponibili per la disponibilità, ma spesso propaga anche errori ed eliminazioni. Backup e archivio soddisfano obiettivi diversi di tempo e immutabilità.

## Separare backup, replica e archivio

**La replica** mantiene disponibili una o più copie in modo sincrono o differito. **Lo snapshot** fa riferimento a un momento nello stesso contesto di storage. **Il backup** crea una copia gestita e ripristinabile con conservazione e catalogo. **L’archivio** conserva i dati a lungo termine secondo regole di governance e accesso. Un sistema può combinare più ruoli; le garanzie restano separate.

Un piano di protezione solido segue [Backup e Disaster Recovery](/kb/backup-dr) e include:

- quiescenza dell’applicazione o semantica di backup online specifica del prodotto;
- dati, metadati, ACL, proprietari, Extended Attributes e chiavi;
- dominio di errore e amministrativo separato;
- retention, immutabilità, modello di eliminazione e Legal Hold;
- catalogo e dipendenze da versioni di software/schema;
- restore regolare in un ambiente di destinazione pulito;
- verifica dell’integrità e funzionale dopo il restore.

## Eliminazione dei dati e fine vita del supporto

L’eliminazione di un file o Object Delete rimuove spesso soltanto riferimenti di nome o metadati. Snapshot, versioni, repliche, cache, backup, Flash Translation Layer e supporti difettosi possono conservare copie. NIST SP 800-88 descrive Clear, Purge e Destroy come procedure di sanitizzazione differenti in base al supporto, al fabbisogno di protezione e al riutilizzo ([NIST SP 800-88 Rev. 1 – Media Sanitization](https://csrc.nist.gov/pubs/sp/800/88/r1/final)).

Nello storage crittografato, Crypto Erase può rendere inutilizzabile il materiale delle chiavi se tutti i dati sono effettivamente protetti dalle chiavi interessate e non esistono copie in chiaro. Key Hierarchy, escrow, repliche e backup devono essere inclusi nella verifica.

## Storia tecnica

I primi sistemi informatici collegavano lo storage direttamente a un singolo computer. Protocolli a blocchi standardizzati quali SCSI separarono host e funzione del dispositivo; RAID combinò dispositivi in modelli di prestazioni e ridondanza. Storage Area Networks tramite Fibre Channel e iSCSI centralizzarono volumi a blocchi, mentre Network Attached Storage forniva condivisioni di file tramite NFS e SMB.

Logical Volume Manager e file system Copy-on-write separarono ulteriormente i dispositivi fisici da volumi, snapshot e integrità dei dati. File system distribuiti e storage di oggetti scalarono metadati e dati su molti nodi. Le API cloud resero lo storage un servizio provisionato per request con Lifecycle, Versioning e fatturazione basata sull’utilizzo.

Flash e NVMe hanno modificato i limiti prestazionali: code parallele e latenze in microsecondi hanno reso più importanti il percorso software, CPU, NUMA e rete. Allo stesso tempo, Software-defined Storage e [Container Storage Interface](https://github.com/container-storage-interface/spec/blob/master/spec.md) astraggono la posizione fisica. La domanda centrale per l’amministratore è rimasta invariata: quali byte confermati sopravvivono a quale guasto, e come viene ripetuta regolarmente questa verifica?

## Checklist per amministratori in sintesi

Una condivisione di storage è comprensibile solo quando conferma di scrittura, ridondanza, comportamento in caso di guasto e restore sono documentati insieme. La tabella riassume queste domande operative indipendentemente dal nome del prodotto.

| Domanda | Verifica operativa |
|---|---|
| Quale modello di accesso? | Dispositivo a blocchi/LUN/namespace, file system/share o bucket/chiave/API |
| Qual è il percorso end-to-end? | Applicazione, cache, FS/volume, driver, trasporto, controller, supporto, replica |
| Quando una scrittura è permanente? | Semantica Flush/fsync/FUA, cache protetta, journal/WAL, quorum |
| Quale coerenza si applica? | Vista locale, Read-after-write, quorum, lag, procedure per conflitti e Split Brain |
| Quali domini di guasto? | Dispositivo, controller, percorso, nodo, rack, zona, sede e dominio amministratore/credenziali |
| Dove si trova la ridondanza? | RAID/Mirror/EC, numero di repliche, collocazione nei Failure Domain, riserva di rebuild |
| Come viene rilevata l’integrità? | Checksum end-to-end, scrub, fonte di riparazione, allarme e confronto del restore |
| Quale capacità conta? | Fisica/logica, Thin Pool, metadati, inode/oggetti, snapshot/versioni, riserva |
| Quale garanzia di prestazioni? | Dimensione del blocco, mix, Sync, coda, latenza p95/p99, IOPS e throughput |
| Come viene ripristinato? | Backup coerente con l’applicazione, chiavi, metadati/ACL, catalogo e test di restore pulito |

## Fonti

- [Linux Kernel – FIEMAP ioctl](https://docs.kernel.org/filesystems/fiemap.html)
- [SNIA – Storage Networking Primer](https://www.snia.org/education/storage_networking_primer)
- [Linux Kernel – Block subsystem](https://docs.kernel.org/block/index.html)
- [Microsoft Learn – Storage overview](https://learn.microsoft.com/en-us/windows-server/storage/storage)
- [RFC 7143 – iSCSI](https://www.rfc-editor.org/rfc/rfc7143.html)
- [NVM Express Specifications](https://nvmexpress.org/specifications/)
- [Linux Kernel – Virtual File System](https://docs.kernel.org/filesystems/vfs.html)
- [Linux Kernel – ext4](https://docs.kernel.org/filesystems/ext4/index.html)
- [Linux Kernel – XFS](https://docs.kernel.org/filesystems/xfs/index.html)
- [Btrfs Documentation](https://btrfs.readthedocs.io/en/latest/)
- [OpenZFS – Basic Concepts](https://openzfs.github.io/openzfs-docs/Basic%20Concepts/index.html)
- [RFC 8881 – NFSv4.1](https://www.rfc-editor.org/rfc/rfc8881.html)
- [Microsoft Open Specifications – MS-SMB2](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-smb2/5606ad47-5ee0-437a-817e-70c366052962)
- [Microsoft Learn – Get-SmbConnection](https://learn.microsoft.com/en-us/powershell/module/smbshare/get-smbconnection)
- [Microsoft Learn – Get-SmbMapping](https://learn.microsoft.com/en-us/powershell/module/smbshare/get-smbmapping)
- [Microsoft Learn – Get-Volume](https://learn.microsoft.com/en-us/powershell/module/storage/get-volume)
- [findmnt(8)](https://man7.org/linux/man-pages/man8/findmnt.8.html)
- [nfsstat(8)](https://man7.org/linux/man-pages/man8/nfsstat.8.html)
- [Samba – smbclient](https://www.samba.org/samba/docs/current/man-html/smbclient.1.html)
- [jq manual](https://jqlang.org/manual/)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [Amazon S3 API Reference](https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html)
- [Amazon S3 – Data consistency model](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)
- [AWS CLI – s3api](https://docs.aws.amazon.com/cli/latest/reference/s3api/)
- [rclone size](https://rclone.org/commands/rclone_size/)
- [fsync(2)](https://man7.org/linux/man-pages/man2/fsync.2.html)
- [Consistency in Non-Transactional Distributed Storage Systems](https://www.cs.cornell.edu/fbs/publications/SMSurvey.pdf)
- [Ceph Architecture](https://docs.ceph.com/en/latest/architecture/)
- [Patterson, Gibson, Katz – A Case for RAID](https://www.cs.cmu.edu/~garth/RAIDpaper/Patterson88.pdf)
- [SNIA – Data Protection Best Practices](https://www.snia.org/sites/default/files/technical-work/whitepapers/SNIA-Data-Protection-Best-Practices-White%20Paper.pdf)
- [Microsoft Learn – Volume Shadow Copy Service](https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service)
- [OpenZFS – Checksums](https://openzfs.github.io/openzfs-docs/Basic%20Concepts/Checksums.html)
- [Btrfs – Scrub](https://btrfs.readthedocs.io/en/latest/Scrub.html)
- [Microsoft Learn – Get-FileHash](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU coreutils – SHA-2 utilities](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Microsoft Learn – Get-StorageReliabilityCounter](https://learn.microsoft.com/en-us/powershell/module/storage/get-storagereliabilitycounter)
- [smartctl(8)](https://www.smartmontools.org/browser/trunk/smartmontools/smartctl.8.in)
- [Microsoft Learn – Repair-Volume](https://learn.microsoft.com/en-us/powershell/module/storage/repair-volume)
- [OpenZFS – zpool status](https://openzfs.github.io/openzfs-docs/man/master/8/zpool-status.8.html)
- [Btrfs – btrfs scrub](https://btrfs.readthedocs.io/en/latest/btrfs-scrub.html)
- [Red Hat – Thinly-provisioned logical volumes](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/htmlsingle/configuring_and_managing_logical_volumes/configuring-lvm-on-san-one-machine_configuring-lvm-on-shared-storage)
- [NIST SP 800-209 – Security Guidelines for Storage Infrastructure](https://csrc.nist.gov/pubs/sp/800/209/final)
- [fio Documentation](https://fio.readthedocs.io/en/latest/fio_doc.html)
- [Microsoft DiskSpd](https://github.com/microsoft/diskspd)
- [Microsoft Learn – Get-Counter](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-counter)
- [iostat(1)](https://man7.org/linux/man-pages/man1/iostat.1.html)
- [pidstat(1)](https://man7.org/linux/man-pages/man1/pidstat.1.html)
- [DiskSpd command line](https://github.com/microsoft/diskspd/wiki/Command-line-and-parameters)
- [Microsoft Learn – Get-Disk](https://learn.microsoft.com/en-us/powershell/module/storage/get-disk)
- [Microsoft Learn – Get-PhysicalDisk](https://learn.microsoft.com/en-us/powershell/module/storage/get-physicaldisk)
- [Microsoft Learn – Get-MPIOPath](https://learn.microsoft.com/en-us/powershell/module/mpio/get-mpiopath)
- [Microsoft Learn – Get-IscsiSession](https://learn.microsoft.com/en-us/powershell/module/iscsi/get-iscsisession)
- [lsblk(8)](https://man7.org/linux/man-pages/man8/lsblk.8.html)
- [multipath(8)](https://manpages.debian.org/bookworm/multipath-tools/multipath.8.en.html)
- [iscsiadm(8)](https://manpages.debian.org/bookworm/open-iscsi/iscsiadm.8.en.html)
- [nvme-cli](https://github.com/linux-nvme/nvme-cli)
- [Amazon S3 Storage Classes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html)
- [NIST SP 800-88 Rev. 1 – Media Sanitization](https://csrc.nist.gov/pubs/sp/800/88/r1/final)
- [Container Storage Interface](https://github.com/container-storage-interface/spec/blob/master/spec.md)
