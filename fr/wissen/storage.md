---
title: "Stockage : bloc, fichier, objet et intégrité des données"
blatt: "storage"
description: "Stockage pour les administrateurs d’infrastructure et de messagerie : stockage par blocs, fichiers et objets, chemin des E/S et du cache, systèmes de fichiers, RAID et Erasure Coding, réplication et cohérence, snapshots, intégrité, chiffrement, capacité, performances, modèles cloud, récupération et histoire technique."
fakten:
  - label: Modèles fondamentaux
    wert: Bloc · fichier · objet ; adressage, métadonnées et cohérence différents
    href: https://www.snia.org/education/storage_networking_primer
  - label: Chemin des blocs
    wert: Application · système de fichiers/gestionnaire de volumes · couche bloc · contrôleur · support
    href: https://docs.kernel.org/block/index.html
  - label: Accès aux fichiers
    wert: local via VFS ou distant, notamment via NFS et SMB
    href: https://www.rfc-editor.org/rfc/rfc8881.html
  - label: Modèle objet
    wert: Bucket/namespace · clé · données · métadonnées · opération d’API
    href: https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html
  - label: Durabilité
    wert: uniquement après la limite de commit/flush définie ; un appel d’écriture réussi peut ne pas suffire
    href: https://man7.org/linux/man-pages/man2/fsync.2.html
  - label: Redondance
    wert: Mirror/RAID/Erasure Coding/réplication augmentent la tolérance aux pannes, mais ne constituent pas une sauvegarde
    href: https://www.snia.org/sites/default/files/technical-work/whitepapers/SNIA-Data-Protection-Best-Practices-White%20Paper.pdf
  - label: Cohérence
    wert: vue locale, distribuée et applicative distinctes ; documenter sync, quorum et lag
    href: https://www.cs.cornell.edu/fbs/publications/SMSurvey.pdf
  - label: Intégrité
    wert: somme de contrôle · vérification de bout en bout · scrub · chemin de réparation
    href: https://openzfs.github.io/openzfs-docs/Basic%20Concepts/Checksums.html
  - label: Snapshots
    wert: référence temporelle dans le même domaine de défaillance et d’administration ; la cohérence doit être établie
    href: https://docs.kernel.org/filesystems/fiemap.html
  - label: Axes de performance
    wert: latence · IOPS · débit · profondeur de file d’attente · taille de bloc · mix lecture/écriture
    href: https://fio.readthedocs.io/en/latest/fio_doc.html
  - label: Axes de capacité
    wert: octets utilisables · espace réservé · métadonnées · inodes/objets · Thin Provisioning
    href: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/htmlsingle/configuring_and_managing_logical_volumes/configuring-lvm-on-san-one-machine_configuring-lvm-on-shared-storage
  - label: Objet de récupération
    wert: données, métadonnées, autorisations, clés, version/snapshot, schéma applicatif et dépendances
    href: https://csrc.nist.gov/pubs/sp/800/209/final
werbung:
  - newsletter
ctaThemen:
  - rclone
  - proton-drive
  - backup-dr
translationSourceHash: 97205e641b2bc37a7cbb4a5e7de137f527187daf42e5da7e951bd18246a3cede
translationModel: gpt-5.6-terra
translatedAt: 2026-10-13T11:31:59.912Z
translationReview: automatic
---

# Stockage : bloc, fichier, objet et intégrité des données

Le stockage constitue la base de l’état de presque toute infrastructure : boîtes aux lettres, files d’attente, index de recherche, répertoires, bases de données, machines virtuelles et sauvegardes se composent d’octets, de métadonnées et d’une promesse quant au moment où les modifications deviennent visibles et durables. Le support seul ne fournit pas cette promesse. L’application, le cache du système d’exploitation, le système de fichiers, le gestionnaire de volumes, le contrôleur, le réseau, les mécanismes de redondance et la réplication distante forment ensemble le chemin des données.

Pour les administrateurs, trois questions sont donc plus importantes que la capacité et les IOPS nominales avant tout choix de produit : **Quel modèle d’accès l’application voit-elle ? Quelle sémantique de commit et de cohérence s’applique ? Quels domaines de défaillance une modification confirmée survit-elle ?** Ce n’est qu’ensuite que les performances, la sauvegarde, le chiffrement et les coûts peuvent être évalués utilement.

L’explication suit une opération d’écriture depuis l’application, via le système de fichiers ou l’API objet, jusqu’au volume, au contrôleur et au support. Ce n’est qu’ensuite que sont comparés la réplication, la cohérence, les performances, le chiffrement, la sauvegarde et la récupération.

Le stockage commence par une demande d’écriture de l’application et ne se termine que lorsque les données sont durablement stockées et à nouveau lisibles selon le modèle promis. Entre les deux se trouvent le cache, le système de fichiers ou l’API objet, le réseau, le contrôleur et le support. L’article suit ce chemin des données plutôt que de simplement énumérer des types de stockage.

## Positionnement technique et chemin des données

Le stockage est classé selon l’interface qu’un consommateur reçoit :

- **Le stockage par blocs** adresse des zones d’un périphérique logique. Le système de fichiers ou la base de données possède la structure supérieure.
- **Le stockage de fichiers** expose des noms hiérarchiques, des répertoires, des attributs, des autorisations et des opérations telles qu’ouvrir, lire ou renommer.
- **Le stockage objet** adresse des objets via un namespace/bucket et une clé ; les données et métadonnées sont gérées par des opérations d’API.

La SNIA décrit le stockage par blocs, fichiers et objets comme des modèles de service et d’accès différents, et non simplement comme trois catégories de produits ([SNIA – Storage Networking Primer](https://www.snia.org/education/storage_networking_primer)). Par exemple, une passerelle S3 devant un système de fichiers modifie le modèle objet externe sans que le stockage interne soit nécessairement fondé sur des objets.

Un chemin d’écriture synchrone peut traverser les couches suivantes :

1. l’application et son environnement d’exécution avec son propre buffering ou Write-ahead Log ;
2. l’appel système, le Virtual File System ou l’accès brut aux blocs ;
3. le Page Cache, le système de fichiers et le gestionnaire de volumes ;
4. la couche bloc du noyau et le planificateur d’E/S ;
5. le pilote de périphérique, le HBA, le protocole réseau ou la couche de stockage virtuelle ;
6. le cache du contrôleur, la couche de redondance/d’Erasure Coding et le support ;
7. les réplicas, le journal, le quorum ou le site distant.

La couche bloc de Linux documente les files d’attente, les requêtes et la topologie des périphériques comme une couche distincte du noyau ([Linux Kernel – Block subsystem](https://docs.kernel.org/block/index.html)). Windows abstrait également séparément les disques physiques, les pools de stockage, les disques virtuels, les partitions, les volumes et les systèmes de fichiers ([Microsoft Learn – Storage overview](https://learn.microsoft.com/en-us/windows-server/storage/storage)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-storage.svg?v=20260813" title="Interaktive Infografik: Storage-Datenpfad von Anwendung und Cache über Dateisystem, Volume, Controller und Medium bis zu Replikation sowie Block-, Datei-, Objekt-, Integritäts- und Recoverymodellen" loading="lazy">
  <a href="/images/kb-interaktiv-storage.svg?v=20260813">Ouvrir directement le graphique interactif</a>.
</iframe>

## Stockage par blocs

Le stockage par blocs fournit un espace d’adressage logique linéaire. Une LUN SCSI, une cible iSCSI, un volume Fibre Channel, un namespace NVMe, un disque virtuel et un volume cloud sont des transports ou implémentations différents du même modèle de base. L’initiateur voit des blocs ; les noms de fichiers, répertoires et souvent la sémantique de verrouillage sont construits au-dessus.

iSCSI encapsule les commandes SCSI sur TCP et distingue Discovery/Login, Session et Connections. La RFC 7143 définit la sémantique du protocole ; le multipathing, l’authentification, la conception réseau et la capacité de clustering du système de fichiers restent des couches opérationnelles supplémentaires ([RFC 7143 – iSCSI](https://www.rfc-editor.org/rfc/rfc7143.html)). NVMe a été conçu pour le stockage non volatile via PCIe et utilise des files Submission/Completion ; NVMe over Fabrics transporte ce modèle notamment via RDMA ou TCP ([NVM Express Specifications](https://nvmexpress.org/specifications/)).

Un volume bloc ne doit pas être monté simultanément en écriture depuis plusieurs hôtes avec des systèmes de fichiers locaux ordinaires. Le fonctionnement en blocs partagés requiert un système de fichiers en cluster ou une application disposant de sa propre coordination distribuée. « Le SAN présente la LUN aux deux serveurs » n’est pas un verrouillage d’écriture.

## Système de fichiers et VFS

Un système de fichiers associe des plages de blocs ou d’autres backends à des fichiers, des répertoires et des métadonnées. Le Virtual File System fournit aux applications une interface commune d’appels système ; les systèmes de fichiers concrets implémentent différemment les opérations, le cache et la durabilité ([Linux Kernel – Overview of the Linux Virtual File System](https://docs.kernel.org/filesystems/vfs.html)).

La journalisation consigne les métadonnées ou, en plus, les données afin de restaurer un état cohérent du système de fichiers après un crash. Ce n’est pas une sauvegarde et elle n’empêche pas qu’une modification fonctionnellement erronée soit correctement journalisée. ext4 documente des modes tels que `journal`, `ordered` et `writeback`; XFS utilise un journal de métadonnées et ses propres mécanismes de récupération ([Linux Kernel – ext4](https://docs.kernel.org/filesystems/ext4/index.html), [XFS Documentation](https://docs.kernel.org/filesystems/xfs/index.html)).

Les systèmes de fichiers Copy-on-write tels que Btrfs et OpenZFS écrivent les blocs modifiés à de nouveaux emplacements et mettent à jour les références. Cela permet des snapshots efficaces et des modèles de sommes de contrôle, mais exige de l’espace pour les nouveaux blocs, les métadonnées et le Garbage Collection ([Btrfs Documentation](https://btrfs.readthedocs.io/en/latest/), [OpenZFS – Basic Concepts](https://openzfs.github.io/openzfs-docs/Basic%20Concepts/index.html)).

## Fichiers distants : NFS et SMB

NFSv4.1 définit un protocole de fichiers distants avec état, comprenant sessions, ouvertures, verrous, délégations et une opération compound. Le cache client et la récupération du serveur font partie du protocole ; un montage NFS n’est pas simplement du « POSIX local sur TCP » ([RFC 8881 – NFSv4.1](https://www.rfc-editor.org/rfc/rfc8881.html)).

SMB 2/3 utilise des crédits, des sessions, des Tree Connects, des ouvertures et des Leases/Oplocks. Microsoft publie la spécification du protocole ; des fonctionnalités comme Multichannel, Transparent Failover, Encryption et Persistent Handles modifient les performances et la sémantique de basculement ([Microsoft Open Specifications – MS-SMB2](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-smb2/5606ad47-5ee0-437a-817e-70c366052962)).

Pour les fichiers distants, il convient au minimum de documenter :

- la résolution de noms, la cible/VIP et les ports ;
- l’authentification et le mappage des identités, par exemple Kerberos, NTLM ou UID/GID ;
- la version du dialecte/protocole, les options de montage et le chiffrement ;
- le comportement du cache, des verrous, des leases et du failover ;
- la sémantique en cas de coupure réseau, timeout et récupération du serveur ;
- les quotas, snapshots et la responsabilité de sauvegarde.

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

[`Get-SmbConnection`](https://learn.microsoft.com/en-us/powershell/module/smbshare/get-smbconnection) et [`Get-SmbMapping`](https://learn.microsoft.com/en-us/powershell/module/smbshare/get-smbmapping) affichent les dialectes SMB actifs et les mappages ; [`Get-Volume`](https://learn.microsoft.com/en-us/powershell/module/storage/get-volume) complète la vue des volumes locaux. [`findmnt`](https://man7.org/linux/man-pages/man8/findmnt.8.html), [`nfsstat`](https://man7.org/linux/man-pages/man8/nfsstat.8.html) et [`smbclient`](https://www.samba.org/samba/docs/current/man-html/smbclient.1.html) vérifient les montages Unix et les vues de protocole ; [`jq`](https://jqlang.org/manual/) filtre leur sortie JSON. [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) et [`nc`](https://man.openbsd.org/nc) prouvent uniquement le chemin TCP, pas l’autorisation de partage ni la sémantique des E/S.

## Stockage objet

Le stockage objet n’utilise pas de handles de fichiers ouverts au sens classique. Une opération lit, écrit, liste, copie ou supprime un objet sous une clé. Les « dossiers » sont souvent des préfixes ou des projections d’interface. Un renommage peut donc être implémenté comme une copie côté serveur suivie d’une suppression. AWS documente l’API S3 comme une interface orientée REST reposant sur des buckets, des objets et des opérations ([Amazon S3 API Reference](https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html)).

Les opérations sur les objets ont leurs propres limites :

- Multipart Upload crée des parties et un commit final ; les uploads interrompus nécessitent un nettoyage par Lifecycle.
- ETag n’est pas universellement une somme de contrôle MD5, en particulier pour les objets multipart ou chiffrés.
- Listing, Versioning, Delete Marker et Lifecycle sont des états distincts.
- Les URL pré-signées délèguent une opération spécifique jusqu’à expiration et doivent être traitées comme des credentials.
- Object Lock/WORM requiert un modèle de retention, Legal Hold, clés et administration.

Amazon S3 documente une cohérence forte Read-after-write et List pour les PUT et Deletes réussis. Cette garantie de produit ne s’applique pas automatiquement aux implémentations compatibles S3, aux caches ou aux passerelles superposées ([Amazon S3 – Data consistency model](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html#ConsistencyModel)).

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

[`aws s3api`](https://docs.aws.amazon.com/cli/latest/reference/s3api/) agit sur les opérations de l’API S3 ; les sous-commandes présentées vérifient l’accès au bucket, le versioning et les attributs d’objet. [`rclone size`](https://rclone.org/commands/rclone_size/) compte les objets et les octets via un backend configuré. Les deux affichent la vue de leurs credentials et de leur endpoint ; la région, le compte et l’alias sont donc également consignés.

Qu’il s’agisse de bloc, fichier ou objet : une opération d’écriture confirmée doit être comprise dans le contexte du cache et du flush. « Écrit avec succès » ne signifie pas le même point de durabilité physique dans chaque pile.

## Cache, flush et durabilité

Un appel `write()` réussi peut seulement signifier que des octets ont été copiés dans un cache du système d’exploitation. Les systèmes proches de POSIX utilisent `fsync()` ou `fdatasync()` afin de transférer les données modifiées et les métadonnées nécessaires jusqu’au périphérique. La page de manuel Linux indique que, pour un fichier nouvellement créé, le répertoire contenant le fichier peut aussi devoir être synchronisé afin que l’entrée de répertoire soit durable ([fsync(2)](https://man7.org/linux/man-pages/man2/fsync.2.html)).

En dessous, le contrôleur et le support doivent implémenter correctement Flush/FUA. Un Write Cache protégé par batterie ou flash peut préserver les écritures confirmées lors d’une perte d’alimentation ; un cache volatil non protégé peut simuler des performances. La virtualisation et le stockage réseau allongent la chaîne. L’administrateur a besoin de la **garantie de commit de bout en bout**, pas seulement d’une case « Write Cache » activée.

Les bases de données utilisent Write-ahead Logging, sommes de contrôle et protocoles de récupération, car un commit du système de fichiers ne définit pas une transaction métier. Un snapshot ou un réplica ne devient cohérent pour l’application que lorsque celle-ci utilise un mécanisme pris en charge de freeze, de sauvegarde ou de positionnement du journal.

## Cohérence dans les systèmes de stockage distribués

Les systèmes distribués répliquent les données entre processus et domaines de défaillance. Une opération peut être confirmée sur un nœud, encore invisible sur un autre ou indécidable lors de partitions réseau. Les modèles de cohérence décrivent les ordres d’observation auxquels les clients peuvent s’attendre. Une vue d’ensemble de Viotti et Vukolić classe les modèles forts et plus faibles selon les garanties d’historique ([Consistency in Non-Transactional Distributed Storage Systems](https://www.cs.cornell.edu/fbs/publications/SMSurvey.pdf)).

Les paramètres pratiques sont :

- le nombre et l’emplacement des réplicas ou fragments d’Erasure Coding ;
- les quorums d’écriture et de lecture ainsi que la limite d’acknowledgement ;
- la réplication de site synchrone ou asynchrone ;
- le comportement en cas de Split Brain, de perte de majorité et de rejoin ;
- Repair, Backfill, Rebalance et Scrub ;
- la garantie de Staleness, Read-after-write et monotone-read ;
- la résolution des conflits et la souveraineté privilégiée des données.

Ceph décrit CRUSH comme un placement déterministe des objets sur des Failure Domains hiérarchiques et RADOS comme une base objet distribuée pour des interfaces bloc, fichier et objet ([Ceph Architecture](https://docs.ceph.com/en/latest/architecture/)). La configuration concrète des pools, réplicas, de l’Erasure Coding et des Failure Domains détermine les défaillances auxquelles le système survit réellement.

La cohérence décrit la vue des données ; la redondance protège contre la panne de composants individuels. RAID et Erasure Coding augmentent la disponibilité, mais ne remplacent pas une sauvegarde indépendante.

## RAID, Erasure Coding et reconstruction

RAID combine plusieurs périphériques pour obtenir des caractéristiques de disponibilité ou de performance. Le mirroring maintient des copies complètes ; le RAID à parité calcule des informations de redondance. Le travail classique de Berkeley a établi les niveaux RAID comme alternatives architecturales ([Patterson, Gibson, Katz – A Case for Redundant Arrays of Inexpensive Disks](https://www.cs.cmu.edu/~garth/RAIDpaper/Patterson88.pdf)).

L’Erasure Coding divise les données en fragments de données et de parité ; l’objet peut être reconstruit à partir d’un nombre minimal d’entre eux. Il économise de la capacité par rapport à une réplication complète, mais augmente les coûts d’encodage, de réseau et de réparation. Les petites écritures aléatoires peuvent être particulièrement coûteuses.

RAID et la réplication ne protègent pas contre :

- la suppression logique ou le chiffrement par un logiciel autorisé ;
- une corruption silencieuse correctement répliquée ;
- des credentials d’administrateur ou de Control Plane compromis ;
- des erreurs en dehors du Failure Domain choisi ;
- des états de logiciel/métadonnées incompatibles ;
- la perte de clés de chiffrement.

La SNIA décrit RAID comme un algorithme de protection des données sur plusieurs périphériques indépendants et traite la sauvegarde comme une technique de protection distincte ; RAID n’implique donc aucune sémantique de sauvegarde ([SNIA – Data Protection Best Practices](https://www.snia.org/sites/default/files/technical-work/whitepapers/SNIA-Data-Protection-Best-Practices-White%20Paper.pdf)). Pendant la reconstruction ou le Backfill, la marge de sécurité diminue et la latence ainsi que la probabilité de défaillance peuvent augmenter. La progression, la redondance restante et le taux d’erreurs doivent faire partie du monitoring.

## Snapshots et clones

Un snapshot est une référence à un état à un instant donné. Copy-on-write ou Redirect-on-write stocke séparément les modifications ultérieures ; un clone peut créer une branche inscriptible à partir de la même base de blocs. Les snapshots sont rapides et utiles pour les points de récupération, les tests et la réplication.

Toutefois, ils partagent souvent le matériel de stockage, le Control Plane, les credentials et le domaine de défaillance avec la source. Une perte de pool ou un administrateur de stockage compromis peut affecter simultanément la source et les snapshots. Les chaînes de snapshots consomment des métadonnées et de la capacité ; la suppression d’un ancien snapshot peut déclencher un travail d’arrière-plan considérable.

La cohérence après crash correspond approximativement à une coupure d’alimentation brutale. La cohérence applicative requiert un flush, un quiesce, un Filesystem Freeze, VSS ou une API de sauvegarde spécifique au produit. Microsoft VSS coordonne Requester, Writer et Provider pour des Shadow Copies cohérentes sous Windows ([Microsoft Learn – Volume Shadow Copy Service](https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service)).

## Sommes de contrôle, scrub et intégrité de bout en bout

Les sommes de contrôle ne détectent les modifications que si une valeur de référence fiable existe et que la zone vérifiée est clairement définie. Le CRC de transport, l’ECC du périphérique, la somme de contrôle du système de fichiers, la somme de contrôle d’objet et le hash applicatif protègent des segments différents. Un ETag ou l’horodatage d’un fichier ne constitue pas une preuve universelle d’intégrité.

OpenZFS stocke des sommes de contrôle pour les données et les métadonnées dans des pointeurs de blocs parents et peut détecter et réparer une copie incorrecte en présence de redondance ([OpenZFS – Checksums](https://openzfs.github.io/openzfs-docs/Basic%20Concepts/Checksums.html)). Btrfs fournit des sommes de contrôle et un scrub sur ses périphériques ; Scrub lit les données et métadonnées, les vérifie et les répare si une bonne copie existe ([Btrfs – Scrub](https://btrfs.readthedocs.io/en/latest/Scrub.html)).

Un scrub représente une charge d’E/S et requiert des fenêtres temporelles, une priorisation et des alertes. « Scrub terminé » n’est significatif que si les erreurs irrécupérables, les erreurs réparées, les zones ignorées et les objets concernés sont évalués.

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

[`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash) et [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) comparent des fichiers contrôlés. [`Get-StorageReliabilityCounter`](https://learn.microsoft.com/en-us/powershell/module/storage/get-storagereliabilitycounter) et [`smartctl`](https://www.smartmontools.org/browser/trunk/smartmontools/smartctl.8.in) lisent les données de santé des périphériques ; [`Repair-Volume`](https://learn.microsoft.com/en-us/powershell/module/storage/repair-volume), [`zpool status`](https://openzfs.github.io/openzfs-docs/man/master/8/zpool-status.8.html) et [`btrfs scrub`](https://btrfs.readthedocs.io/en/latest/btrfs-scrub.html) affichent les perspectives du système de fichiers/pool. Ces outils ne doivent être exécutés que sur des cibles clairement identifiées.

## Thin Provisioning, Discard et capacité

Le Thin Provisioning présente davantage d’espace logique que d’espace physiquement alloué. Cela accroît l’utilisation, mais crée une limite de défaillance supplémentaire : un hôte voit de l’espace libre alors que le pool ou la zone de métadonnées est épuisé. La croissance, la consommation des snapshots, les réserves et le reclaim doivent être surveillés ensemble ([Red Hat – Thinly-provisioned logical volumes](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/htmlsingle/configuring_and_managing_logical_volumes/configuring-lvm-on-san-one-machine_configuring-lvm-on-shared-storage)).

TRIM/UNMAP/Discard signale aux couches inférieures les blocs qui ne sont plus nécessaires. La transmission de cette information jusqu’à l’array ou au backend cloud dépend du système de fichiers, des options de montage, de l’hyperviseur, du protocole et du périphérique. Le reclaim peut affecter les performances et la protection des données ; sur des volumes chiffrés, Discard peut révéler des schémas d’occupation.

Le monitoring de capacité comprend :

- les octets logiques et physiques ainsi que l’overcommit ;
- les blocs du système de fichiers, les inodes ou le nombre d’objets ;
- l’espace des snapshots, journaux, WAL et métadonnées ;
- l’espace réservé, la réserve de reconstruction/Backfill et les quotas ;
- la corbeille, les Delete Markers, les anciennes versions et les restes multipart ;
- la croissance par classe de données et le moment jusqu’à l’épuisement.

## HDD, flash et NVMe

Les HDD ont des temps de positionnement mécaniques ; les E/S aléatoires et le débit séquentiel diffèrent fortement. Les SSD utilisent une Flash Translation Layer, le Garbage Collection, l’Over-Provisioning et le Wear Leveling. L’amplification d’écriture et la durée de vie restante dépendent de la charge de travail et du firmware. NVMe réduit les goulots d’étranglement de protocole et de files d’attente, mais n’élimine pas les questions de support ou de durabilité.

Les valeurs SMART/NVMe Health sont des signaux propres au fabricant, non une garantie. Les avertissements critiques, Media/Data Integrity Errors, Unsafe Shutdowns, la température, Spare et Percentage Used sont évalués avec les erreurs applicatives et les logs du contrôleur. Le projet NVMe publie les spécifications relatives aux contrôleurs, à la gestion et aux transports ([NVM Express Specifications](https://nvmexpress.org/specifications/)).

## Chiffrement et limites des clés

Le chiffrement du stockage peut intervenir au niveau du support, du contrôleur, du volume, du système de fichiers, de l’objet ou de l’application. Chaque niveau protège contre un modèle de menace différent :

| Niveau | Protège principalement | Limite persistante |
|---|---|---|
| Self-Encrypting Drive | support retiré avec verrouillage correct | l’hôte voit le texte en clair ; exploitation des clés/du firmware |
| Volume Encryption | blocs lus hors ligne | l’hôte en fonctionnement et les processus privilégiés voient le texte en clair |
| Système de fichiers/fichier | namespaces et utilisateurs sélectionnés | les métadonnées peuvent rester visibles |
| KMS côté serveur | supports/backends du fournisseur | Control Plane et IAM du fournisseur |
| chiffrement objet côté client | le fournisseur voit le Ciphertext | recherche/aperçu, perte de clés, métadonnées et migration |

NIST SP 800-209 exige pour les infrastructures de stockage notamment le contrôle d’accès, une administration sécurisée, l’intégrité des données, le chiffrement, l’isolation et le monitoring comme modèle de sécurité cohérent ([NIST SP 800-209 – Security Guidelines for Storage Infrastructure](https://csrc.nist.gov/pubs/sp/800/209/final)).

La rotation des clés et la restauration sont testées ensemble. Une sauvegarde dépourvue des clés et métadonnées valides au moment de la sauvegarde n’est pas restaurable ; une clé exportée sans contexte IAM/wrapping peut être tout aussi inutile.

La capacité seule indique peu le comportement d’une application. La latence, les IOPS, le débit et les files d’attente sur l’ensemble du chemin des données sont déterminants.

## Performance : latence, IOPS, débit et file d’attente

Les IOPS nominales ne sont pas comparables sans taille de bloc, mix lecture/écriture, modèle d’accès, Queue Depth, synchronisme et percentiles de latence. Le débit est approximativement égal aux IOPS multipliées par la taille de transfert, jusqu’à ce que les limites du bus, du réseau, du contrôleur ou du support dominent. Les cache hits peuvent contourner entièrement le support.

Il convient de mesurer au minimum :

- la latence de bout en bout du point de vue de l’application, y compris p95/p99 ;
- les IOPS et octets/s séparément pour la lecture et l’écriture ;
- la Queue Depth, l’utilisation et le temps d’attente par périphérique/volume ;
- la latence des écritures synchrones et le taux de flush ;
- les cache hits, Dirty Data et le throttling ;
- le lag du backend/de la réplication, la reconstruction ou le Garbage Collection ;
- les erreurs, retries, timeouts et basculements de chemin.

`fio` peut générer des charges d’E/S contrôlées ; sa documentation avertit implicitement, par ses options d’écriture flexibles : une mauvaise cible détruit des données. Les tests ne sont exécutés que sur des fichiers de test dédiés ou des périphériques autorisés avec un chemin vérifié ([fio Documentation](https://fio.readthedocs.io/en/latest/fio_doc.html)). DiskSpd de Microsoft est un outil de stockage synthétique équivalent pour Windows ([Microsoft DiskSpd](https://github.com/microsoft/diskspd)).

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

[`Get-Counter`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-counter), [`iostat`](https://man7.org/linux/man-pages/man1/iostat.1.html) et [`pidstat`](https://man7.org/linux/man-pages/man1/pidstat.1.html) observent la charge réelle. [`diskspd`](https://github.com/microsoft/diskspd/wiki/Command-line-and-parameters) et [`fio`](https://fio.readthedocs.io/en/latest/fio_doc.html) génèrent une charge synthétique et ne doivent jamais cibler des périphériques bloc de production ou des fichiers non sauvegardés.

## Multipathing et Failure Domains

Plusieurs chemins peuvent survivre à des défaillances de contrôleur, HBA, switch ou fabric s’ils sont réellement indépendants. Le multipathing détecte plusieurs voies vers le même périphérique logique et effectue le failover ou le Load Balancing. Le nombre de chemins seul ne prouve pas la redondance : deux sessions via le même switch et contrôleur échouent toujours ensemble.

Asymmetric Logical Unit Access peut identifier des chemins de contrôleur préférés et non préférés. Les valeurs de timeout doivent correspondre entre l’application, le système de fichiers, le multipath, le transport et l’array ; des timeouts trop courts provoquent de fausses défaillances, tandis que des valeurs trop longues bloquent les files d’attente.

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

[`Get-Disk`](https://learn.microsoft.com/en-us/powershell/module/storage/get-disk), [`Get-PhysicalDisk`](https://learn.microsoft.com/en-us/powershell/module/storage/get-physicaldisk), [`Get-MPIOPath`](https://learn.microsoft.com/en-us/powershell/module/mpio/get-mpiopath) et [`Get-IscsiSession`](https://learn.microsoft.com/en-us/powershell/module/iscsi/get-iscsisession) affichent les périphériques et chemins Windows. [`lsblk`](https://man7.org/linux/man-pages/man8/lsblk.8.html), [`multipath`](https://manpages.debian.org/bookworm/multipath-tools/multipath.8.en.html), [`iscsiadm`](https://manpages.debian.org/bookworm/open-iscsi/iscsiadm.8.en.html) et [`nvme`](https://github.com/linux-nvme/nvme-cli) affichent l’association Linux, les sessions et les namespaces. Le numéro de série, le WWN, la LUN et le point de montage sont vérifiés avant toute modification.

## Classes de stockage cloud et Lifecycle

Le stockage objet cloud propose des classes aux prix, durées minimales de conservation, coûts de récupération et délais de restauration différents. Les archives peuvent être réhydratées de manière asynchrone ; les Lifecycle Policies peuvent déplacer les objets entre classes ou supprimer des versions. Un chiffre de capacité bon marché sans prix des requêtes, de récupération, d’egress et de durée minimale est incomplet.

AWS documente les Storage Classes avec leurs modèles d’accès et restrictions ; d’autres fournisseurs utilisent leurs propres classes et garanties ([Amazon S3 Storage Classes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html)). Pour un plan de récupération, le RPO, le RTO, la bande passante de restauration, le taux de requêtes, les coûts d’egress et la disponibilité des clés sont donc testés pour chaque classe de données.

La réplication conserve des copies supplémentaires pour la disponibilité, mais propage souvent aussi les erreurs et les suppressions. La sauvegarde et l’archivage répondent à d’autres objectifs de durée et d’immuabilité.

## Distinguer sauvegarde, réplication et archivage

**La réplication** maintient une ou plusieurs copies disponibles, de façon synchrone ou différée. **Le snapshot** référence un instant dans le même contexte de stockage. **La sauvegarde** crée une copie gérée et restaurable, assortie d’une rétention et d’un catalogue. **L’archivage** conserve les données à long terme selon des règles de gouvernance et d’accès. Un système peut combiner plusieurs rôles ; les garanties restent distinctes.

Un plan de protection solide suit [Sauvegarde et Disaster Recovery](/kb/backup-dr) et comprend :

- la mise au repos de l’application ou une sémantique de sauvegarde en ligne spécifique au produit ;
- les données, métadonnées, ACL, propriétaires, Extended Attributes et clés ;
- un domaine de défaillance et d’administration séparé ;
- la rétention, l’immutabilité et le modèle de suppression et de Legal Hold ;
- le catalogue et les dépendances vis-à-vis des versions de logiciel/schéma ;
- une restauration régulière dans un environnement cible propre ;
- un contrôle d’intégrité et fonctionnel après la restauration.

## Suppression des données et fin de vie des supports

La suppression d’un fichier ou d’un objet ne retire souvent que des références de nom ou de métadonnées. Les snapshots, versions, réplicas, caches, sauvegardes, Flash Translation Layer et supports défectueux peuvent conserver des copies. NIST SP 800-88 décrit Clear, Purge et Destroy comme des procédures de sanitization distinctes selon le support, le besoin de protection et la réutilisation ([NIST SP 800-88 Rev. 1 – Media Sanitization](https://csrc.nist.gov/pubs/sp/800/88/r1/final)).

Dans un stockage chiffré, Crypto Erase peut rendre le matériel de clés inutilisable si toutes les données sont effectivement protégées par les clés concernées et qu’aucune copie en clair n’existe. La hiérarchie des clés, l’escrow, les réplicas et les sauvegardes doivent être inclus dans la preuve.

## Histoire technique

Les premiers systèmes informatiques reliaient le stockage directement à un ordinateur unique. Les protocoles blocs standardisés tels que SCSI ont séparé l’hôte de la fonction du périphérique ; RAID a combiné des périphériques dans des modèles de performance et de redondance. Les Storage Area Networks via Fibre Channel et iSCSI ont centralisé les volumes blocs, tandis que le Network Attached Storage fournissait des partages de fichiers via NFS et SMB.

Les Logical Volume Managers et systèmes de fichiers Copy-on-write ont davantage séparé les périphériques physiques des volumes, snapshots et de l’intégrité des données. Les systèmes de fichiers distribués et le stockage objet ont mis à l’échelle les métadonnées et les données sur de nombreux nœuds. Les API cloud ont fait du stockage un service provisionné par requête, avec Lifecycle, Versioning et facturation à l’usage.

Le flash et NVMe ont modifié les limites de performance : les files parallèles et les latences en microsecondes ont rendu le chemin logiciel, le CPU, NUMA et le réseau plus importants. Parallèlement, le stockage défini par logiciel et l’[Container Storage Interface](https://github.com/container-storage-interface/spec/blob/master/spec.md) abstraient l’emplacement physique. La question centrale de l’administrateur reste inchangée : quels octets confirmés survivent à quelle défaillance, et comment cette preuve est-elle répétée régulièrement ?

## Checklist administrateur en un coup d’œil

Un partage de stockage n’est réellement compris que lorsque la confirmation d’écriture, la redondance, le comportement en cas de panne et la restauration sont documentés ensemble. Le tableau résume ces questions opérationnelles indépendamment du nom du produit.

| Question | Preuve opérationnelle |
|---|---|
| Quel modèle d’accès ? | périphérique bloc/LUN/namespace, système de fichiers/partage ou bucket/clé/API |
| Quel est le chemin de bout en bout ? | application, cache, FS/volume, pilote, transport, contrôleur, support, réplica |
| Quand une écriture est-elle durable ? | sémantique Flush/fsync/FUA, cache protégé, journal/WAL, quorum |
| Quelle cohérence s’applique ? | vue locale, Read-after-write, quorum, lag, procédures de conflit et de Split Brain |
| Quels domaines de défaillance ? | périphérique, contrôleur, chemin, nœud, rack, zone, site et domaine administrateur/credentials |
| Où se situe la redondance ? | RAID/Mirror/EC, nombre de réplicas, placement des Failure Domains, réserve de reconstruction |
| Comment l’intégrité est-elle détectée ? | somme de contrôle de bout en bout, scrub, source de réparation, alerte et comparaison de restauration |
| Quelle capacité compte ? | physique/logique, Thin Pool, métadonnées, inodes/objets, snapshots/versions, réserve |
| Quelle garantie de performance ? | taille de bloc, mix, sync, queue, latence p95/p99, IOPS et débit |
| Comment la restauration est-elle effectuée ? | sauvegarde cohérente avec l’application, clés, métadonnées/ACL, catalogue et test de restauration propre |

## Sources

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
