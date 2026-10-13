---
title: "Lagring: block, fil, objekt och dataintegritet"
blatt: "storage"
description: "Lagring för infrastruktur- och meddelandeadministratörer: block-, fil- och objektlagring, I/O- och cacheväg, filsystem, RAID och Erasure Coding, replikering och konsistens, ögonblicksbilder, integritet, kryptering, kapacitet, prestanda, molnmodeller, återställning och teknisk historia."
fakten:
  - label: Grundmodeller
    wert: Block · fil · objekt; olika adressering, metadata och konsistens
    href: https://www.snia.org/education/storage_networking_primer
  - label: Blockväg
    wert: Applikation · filsystem/volymhanterare · blocklager · styrenhet · medium
    href: https://docs.kernel.org/block/index.html
  - label: Filåtkomst
    wert: lokalt via VFS eller på distans bland annat via NFS och SMB
    href: https://www.rfc-editor.org/rfc/rfc8881.html
  - label: Objektmodell
    wert: Bucket/namnområde · nyckel · data · metadata · API-operation
    href: https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html
  - label: Varaktighet
    wert: först efter definierad commit-/flushgräns; ett lyckat write-anrop kan ensamt vara otillräckligt
    href: https://man7.org/linux/man-pages/man2/fsync.2.html
  - label: Redundans
    wert: Mirror/RAID/Erasure Coding/replikering ökar feltoleransen, men är ingen säkerhetskopia
    href: https://www.snia.org/sites/default/files/technical-work/whitepapers/SNIA-Data-Protection-Best-Practices-White%20Paper.pdf
  - label: Konsistens
    wert: lokal, distribuerad och applikationsvy separat; dokumentera Sync, quorum och fördröjning
    href: https://www.cs.cornell.edu/fbs/publications/SMSurvey.pdf
  - label: Integritet
    wert: kontrollsumma · end-to-end-verifiering · scrub · reparationsväg
    href: https://openzfs.github.io/openzfs-docs/Basic%20Concepts/Checksums.html
  - label: Ögonblicksbilder
    wert: tidpunktsreferens i samma fel- och administrationsdomän; konsistens måste etableras
    href: https://docs.kernel.org/filesystems/fiemap.html
  - label: Prestandadimensioner
    wert: latens · IOPS · genomströmning · Queue Depth · blockstorlek · läs-/skrivmix
    href: https://fio.readthedocs.io/en/latest/fio_doc.html
  - label: Kapacitetsdimensioner
    wert: användbara byte · reserverat utrymme · metadata · inoder/objekt · Thin Provisioning
    href: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/htmlsingle/configuring_and_managing_logical_volumes/configuring-lvm-on-san-one-machine_configuring-lvm-on-shared-storage
  - label: Återställningsobjekt
    wert: data, metadata, behörigheter, nycklar, version/ögonblicksbild, applikationsschema och beroenden
    href: https://csrc.nist.gov/pubs/sp/800/209/final
werbung:
  - newsletter
ctaThemen:
  - rclone
  - proton-drive
  - backup-dr
translationSourceHash: 97205e641b2bc37a7cbb4a5e7de137f527187daf42e5da7e951bd18246a3cede
translationModel: gpt-5.6-terra
translatedAt: 2026-10-13T11:35:45.089Z
translationReview: automatic
---

# Lagring: block, fil, objekt och dataintegritet

Lagring är tillståndsgrunden för nästan varje infrastruktur: postlådor, köer, sökindex, kataloger, databaser, virtuella maskiner och säkerhetskopior består av byte plus metadata och ett löfte om när ändringar blir synliga och beständiga. Mediet i sig ger inte detta löfte. Applikation, operativsystemcache, filsystem, volymhanterare, styrenhet, nätverk, redundansmekanismer och fjärrreplikering utgör tillsammans datavägen.

För administratörer är därför tre frågor viktigare än kapacitet och nominella IOPS före varje produktval: **Vilken åtkomstmodell ser applikationen? Vilken commit- och konsistenssemantik gäller? Vilka feldomäner överlever en bekräftad ändring?** Först därefter kan prestanda, säkerhetskopiering, kryptering och kostnader bedömas meningsfullt.

Förklaringen följer en skrivoperation från applikationen via filsystem eller objekt-API till volym, styrenhet och medium. Först därefter jämförs replikering, konsistens, prestanda, kryptering, säkerhetskopiering och återställning.

Lagring börjar med en skrivbegäran från applikationen och slutar först när data enligt den utlovade modellen är beständiga och kan läsas igen. Däremellan finns cache, filsystem eller objekt-API, nätverk, styrenhet och medium. Artikeln följer denna dataväg i stället för att bara räkna upp lagringstyper.

## Teknisk klassificering och dataväg

Lagring klassificeras efter gränssnittet som en konsument får:

- **Blocklagring** adresserar områden på en logisk enhet. Filsystemet eller databasen äger den högre strukturen.
- **Fillagring** exponerar hierarkiska namn, kataloger, attribut, behörigheter och operationer som att öppna, läsa eller byta namn.
- **Objektlagring** adresserar objekt via namnområde/bucket och nyckel; data och metadata hanteras genom API-operationer.

SNIA beskriver block-, fil- och objektlagring som olika tjänste- och åtkomstmodeller, inte bara som tre produktklasser ([SNIA – Storage Networking Primer](https://www.snia.org/education/storage_networking_primer)). En S3-gateway framför ett filsystem ändrar exempelvis den externa objektmodellen utan att den interna datalagringen nödvändigtvis är objektbaserad.

En synkron skrivväg kan passera följande lager:

1. Applikation och körtid med egen buffring eller write-ahead-logg;
2. Systemanrop, Virtual File System eller rå blockåtkomst;
3. Page Cache, filsystem och volymhanterare;
4. Kernel-blocklager och I/O-schemaläggare;
5. Enhetsdrivrutin, HBA, nätverksprotokoll eller virtuellt lagringslager;
6. Styrenhetscache, redundans-/Erasure-Coding-lager och medium;
7. Repliker, journal, quorum eller fjärrplats.

Linux blocklager dokumenterar köer, begäranden och enhetstopologi som ett eget kärnlager ([Linux Kernel – Block subsystem](https://docs.kernel.org/block/index.html)). Windows abstraherar också fysiska datadiskar, Storage Pools, virtuella diskar, partitioner, volymer och filsystem separat ([Microsoft Learn – Storage overview](https://learn.microsoft.com/en-us/windows-server/storage/storage)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-storage.svg?v=20260813" title="Interaktive Infografik: Storage-Datenpfad von Anwendung und Cache über Dateisystem, Volume, Controller und Medium bis zu Replikation sowie Block-, Datei-, Objekt-, Integritäts- und Recoverymodellen" loading="lazy">
  <a href="/images/kb-interaktiv-storage.svg?v=20260813">Öppna den interaktiva grafiken direkt</a>.
</iframe>

## Blocklagring

Blocklagring tillhandahåller ett linjärt logiskt adressutrymme. SCSI LUN, iSCSI Target, Fibre Channel-volym, NVMe-namnområde, virtuell disk och molnvolym är olika transporter eller implementeringar av samma grundmodell. Initiatorn ser block; filnamn, kataloger och ofta även låssemantik uppstår ovanpå detta.

iSCSI kapslar in SCSI-kommandon över TCP och separerar Discovery/Login, Session och Connections. RFC 7143 definierar protokollsemantiken; multipathing, autentisering, nätverksdesign och filsystemets klusterförmåga förblir ytterligare driftslager ([RFC 7143 – iSCSI](https://www.rfc-editor.org/rfc/rfc7143.html)). NVMe utformades för icke-flyktig lagring via PCIe och använder Submission/Completion Queues; NVMe over Fabrics överför modellen bland annat via RDMA eller TCP ([NVM Express Specifications](https://nvmexpress.org/specifications/)).

En blockvolym får inte monteras skrivbart av flera värdar samtidigt med vanliga lokala filsystem. Delad blockdrift kräver ett klusterfilsystem eller en applikation med egen distribuerad koordinering. ”SAN:et presenterar LUN:en för båda servrarna” är inget skrivlås.

## Filsystem och VFS

Ett filsystem tilldelar blockområden eller andra backend-system till filer, kataloger och metadata. Virtual File System tillhandahåller applikationer ett gemensamt systemanropsgränssnitt; konkreta filsystem implementerar operationer, cachning och varaktighet på olika sätt ([Linux Kernel – Overview of the Linux Virtual File System](https://docs.kernel.org/filesystems/vfs.html)).

Journaling loggar metadata eller även data för att efter en krasch återställa ett konsistent filsystemstillstånd. Det är ingen säkerhetskopia och förhindrar inte att en sakligt felaktig ändring journalförs korrekt. ext4 dokumenterar lägen som `journal`, `ordered` och `writeback`; XFS använder en metadatajournal och egna återställningsmekanismer ([Linux Kernel – ext4](https://docs.kernel.org/filesystems/ext4/index.html), [XFS Documentation](https://docs.kernel.org/filesystems/xfs/index.html)).

Copy-on-write-filsystem som Btrfs och OpenZFS skriver ändrade block till nya platser och uppdaterar referenser. Det möjliggör effektiva ögonblicksbilder och kontrollsummemodeller, men kräver utrymme för nya block, metadata och garbage collection ([Btrfs Documentation](https://btrfs.readthedocs.io/en/latest/), [OpenZFS – Basic Concepts](https://openzfs.github.io/openzfs-docs/Basic%20Concepts/index.html)).

## Fjärrfiler: NFS och SMB

NFSv4.1 definierar ett tillståndsfullt fjärrfilsprotokoll med Sessions, Opens, Locks, Delegations och en Compound-operation. Klientcache och serveråterställning är en del av protokollet; en NFS-montering är inte bara ”lokal POSIX över TCP” ([RFC 8881 – NFSv4.1](https://www.rfc-editor.org/rfc/rfc8881.html)).

SMB 2/3 använder Credits, Sessions, Tree Connects, Opens och Leases/Oplocks. Microsoft publicerar protokollspecifikationen; funktioner som Multichannel, Transparent Failover, Encryption och Persistent Handles förändrar prestanda- och failoversemantik ([Microsoft Open Specifications – MS-SMB2](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-smb2/5606ad47-5ee0-437a-817e-70c366052962)).

För fjärrfiler dokumenteras minst:

- Namnupplösning, mål/VIP och portar;
- Autentisering och identitetsmappning, exempelvis Kerberos, NTLM eller UID/GID;
- Dialect/protokollversion, monteringsalternativ och kryptering;
- Cache-, lock-, lease- och failoverbeteende;
- Semantik vid nätverksavbrott, timeout och serveråterställning;
- Kvoter, ögonblicksbilder och ansvar för säkerhetskopiering.

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

[`Get-SmbConnection`](https://learn.microsoft.com/en-us/powershell/module/smbshare/get-smbconnection) och [`Get-SmbMapping`](https://learn.microsoft.com/en-us/powershell/module/smbshare/get-smbmapping) visar aktiva SMB-Dialects och mappningar; [`Get-Volume`](https://learn.microsoft.com/en-us/powershell/module/storage/get-volume) kompletterar den lokala volymvyn. [`findmnt`](https://man7.org/linux/man-pages/man8/findmnt.8.html), [`nfsstat`](https://man7.org/linux/man-pages/man8/nfsstat.8.html) och [`smbclient`](https://www.samba.org/samba/docs/current/man-html/smbclient.1.html) kontrollerar Unix-monteringar och protokollvyer; [`jq`](https://jqlang.org/manual/) filtrerar deras JSON-utdata. [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) och [`nc`](https://man.openbsd.org/nc) visar endast TCP-vägen, inte share-auktorisering eller I/O-semantik.

## Objektlagring

Objektlagring använder inte öppna filhandtag i klassisk mening. En operation läser, skriver, listar, kopierar eller tar bort ett objekt under en nyckel. ”Mappar” är ofta prefix eller UI-projektioner. Ett namnbyte kan därför implementeras som en kopiering på serversidan följt av borttagning. AWS dokumenterar S3-API:t som ett REST-orienterat gränssnitt över Buckets, Objects och Operations ([Amazon S3 API Reference](https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html)).

Objektoperationer har egna gränser:

- Multipart Upload skapar delar och en avslutande commit; avbrutna uppladdningar behöver livscykelrensning.
- ETag är inte universellt en MD5-kontrollsumma, särskilt inte för multipart- eller krypterade objekt.
- Listing, Versioning, Delete Marker och Lifecycle är separata tillstånd.
- Pre-signed URLs delegerar en konkret operation fram till utgången och ska hanteras som credentials.
- Object Lock/WORM kräver en modell för retention, Legal Hold, nycklar och administration.

Amazon S3 dokumenterar stark Read-after-write- och List-konsistens för lyckade PUTs och Deletes. Detta produktlöfte gäller inte automatiskt för S3-kompatibla implementationer, cacheminnen eller överlagrade gateways ([Amazon S3 – Data consistency model](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html#ConsistencyModel)).

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

[`aws s3api`](https://docs.aws.amazon.com/cli/latest/reference/s3api/) arbetar med S3-API-operationer; de visade underkommandona kontrollerar bucketåtkomst, versionshantering och objektattribut. [`rclone size`](https://rclone.org/commands/rclone_size/) räknar objekt och byte via en konfigurerad backend. Båda visar vyn för sina credentials och sin endpoint; region, konto och alias loggas därför också.

Oavsett om det gäller block, fil eller objekt måste en bekräftad skrivoperation förstås i sitt cache- och flushsammanhang. ”Skrivet utan fel” betyder inte samma punkt av fysisk varaktighet i varje stack.

## Cache, flush och varaktighet

Ett lyckat `write()`-anrop kan bara betyda att byte kopierats till en operativsystemcache. POSIX-liknande system använder `fsync()` respektive `fdatasync()` för att överföra ändrade data och nödvändiga metadata till enheten. Linux-manualsidan påpekar att den innehållande katalogen dessutom kan synkroniseras för en ny skapad fil, så att katalogposten blir beständig ([fsync(2)](https://man7.org/linux/man-pages/man2/fsync.2.html)).

Under detta måste styrenhet och medium implementera Flush/FUA korrekt. En batteri- eller flashskyddad write cache kan bevara bekräftade writes vid strömavbrott; en flyktig cache utan skydd kan simulera prestanda. Virtualisering och nätverkslagring förlänger kedjan. Administratören behöver **end-to-end-commitlöftet**, inte bara en aktiverad kryssruta för ”Write Cache”.

Databaser använder write-ahead logging, checksummor och återställningsprotokoll eftersom en filsystemcommit inte definierar en verksamhetstransaktion. En ögonblicksbild eller replik blir applikationskonsistent först när applikationen använder en stödd mekanism för freeze, säkerhetskopiering eller loggpositionering.

## Konsistens i distribuerade lagringssystem

Distribuerade system replikerar data över processer och feldomäner. En operation kan bekräftas på en nod, fortfarande vara osynlig på en annan eller vara oavgörbar vid nätverkspartitioner. Konsistensmodeller beskriver vilka observationsordningar klienter kan förvänta sig. En översikt av Viotti och Vukolić klassificerar starka och svagare modeller utifrån garantier för förlopp ([Consistency in Non-Transactional Distributed Storage Systems](https://www.cs.cornell.edu/fbs/publications/SMSurvey.pdf)).

Praktiska parametrar är:

- Antal och placering av repliker respektive Erasure-Coding-fragment;
- Write- och Read-quorum samt bekräftelsegräns;
- Synkron eller asynkron platsreplikering;
- Beteende vid Split Brain, förlorad majoritet och återanslutning;
- Repair, Backfill, Rebalance och Scrub;
- Löfte om staleness, Read-after-write och monotona läsningar;
- Konfliktlösning och föredragen dataauktoritet.

Ceph beskriver CRUSH som deterministisk placering av objekt över hierarkiska Failure Domains och RADOS som distribuerad objektbas för block-, fil- och objektgränssnitt ([Ceph Architecture](https://docs.ceph.com/en/latest/architecture/)). Den konkreta konfigurationen av pool, replik, Erasure Coding och Failure Domain avgör vilka fel systemet faktiskt överlever.

Konsistens beskriver synen på data; redundans skyddar mot fel i enskilda komponenter. RAID och Erasure Coding ökar tillgängligheten, men ersätter ingen oberoende säkerhetskopia.

## RAID, Erasure Coding och återuppbyggnad

RAID kombinerar flera enheter för tillgänglighets- eller prestandaegenskaper. Mirroring behåller fullständiga kopior; Parity-RAID beräknar redundansinformation. Det klassiska Berkeley-arbetet etablerade RAID-nivåerna som arkitekturalternativ ([Patterson, Gibson, Katz – A Case for Redundant Arrays of Inexpensive Disks](https://www.cs.cmu.edu/~garth/RAIDpaper/Patterson88.pdf)).

Erasure Coding delar upp data i data- och paritetsfragment; från ett minsta antal kan objektet rekonstrueras. Jämfört med full replikering sparar det kapacitet, men ökar kostnaderna för kodning, nätverk och reparation. Små slumpmässiga writes kan vara särskilt kostsamma.

RAID och replikering skyddar inte mot:

- Logisk radering eller kryptering genom auktoriserad programvara;
- Tyst korruption som replikeras korrekt;
- Komprometterade administratörs- eller control-plane-credentials;
- Fel utanför den valda Failure Domain;
- Inkompatibla programvaru-/metadatatillstånd;
- Förlust av krypteringsnycklar.

SNIA beskriver RAID som en dataskyddsalgoritm över flera oberoende enheter och behandlar säkerhetskopiering som en separat skyddsteknik; RAID medför därför ingen säkerhetskopieringssemantik ([SNIA – Data Protection Best Practices](https://www.snia.org/sites/default/files/technical-work/whitepapers/SNIA-Data-Protection-Best-Practices-White%20Paper.pdf)). Under återuppbyggnad eller backfill minskar säkerhetsmarginalen, och latens samt sannolikheten för fel kan öka. Framsteg, återstående redundans och felfrekvens ska ingå i övervakningen.

## Ögonblicksbilder och kloner

En ögonblicksbild är en referens till ett tillstånd vid en tidpunkt. Copy-on-write eller Redirect-on-write lagrar senare ändringar separat; en klon kan skapa en skrivbar gren från samma blockbas. Ögonblicksbilder är snabba och användbara för återställningspunkter, tester och replikering.

De delar dock ofta lagringshårdvara, control plane, credentials och feldomän med källan. En poolförlust eller komprometterad lagringsadministratör kan drabba både källa och ögonblicksbilder. Ögonblicksbildskedjor förbrukar metadata och kapacitet; borttagning av en gammal ögonblicksbild kan utlösa betydande bakgrundsarbete.

Kraschkonsistens motsvarar ungefär ett abrupt strömavbrott. Applikationskonsistens kräver Flush, Quiesce, Filesystem Freeze, VSS eller ett produktspecifikt backup-API. Microsoft VSS samordnar Requester, Writer och Provider för konsistenta Shadow Copies i Windows ([Microsoft Learn – Volume Shadow Copy Service](https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service)).

## Kontrollsummor, scrub och end-to-end-integritet

Kontrollsummor upptäcker ändringar endast om ett betrott referensvärde finns och det kontrollerade området är tydligt. Transport-CRC, enhets-ECC, filsystemkontrollsumma, objektkontrollsumma och applikationshash skyddar olika sträckor. En ETag eller filtid är inget universellt bevis på integritet.

OpenZFS lagrar kontrollsummor för data och metadata i överordnade blockpekare och kan vid redundans upptäcka och reparera en felaktig kopia ([OpenZFS – Checksums](https://openzfs.github.io/openzfs-docs/Basic%20Concepts/Checksums.html)). Btrfs erbjuder kontrollsummor och scrub över sina enheter; scrub läser data och metadata, verifierar och reparerar när en korrekt kopia finns ([Btrfs – Scrub](https://btrfs.readthedocs.io/en/latest/Scrub.html)).

En scrub är I/O-belastning och kräver tidsfönster, prioritering och larm. ”Scrub slutförd” är endast meningsfullt om icke-korrigerbara fel, reparerade fel, överhoppade områden och berörda objekt utvärderas.

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

[`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash) och [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) jämför kontrollerade filer. [`Get-StorageReliabilityCounter`](https://learn.microsoft.com/en-us/powershell/module/storage/get-storagereliabilitycounter) och [`smartctl`](https://www.smartmontools.org/browser/trunk/smartmontools/smartctl.8.in) läser enhetshälsodata; [`Repair-Volume`](https://learn.microsoft.com/en-us/powershell/module/storage/repair-volume), [`zpool status`](https://openzfs.github.io/openzfs-docs/man/master/8/zpool-status.8.html) och [`btrfs scrub`](https://btrfs.readthedocs.io/en/latest/btrfs-scrub.html) visar filsystems-/poolperspektiv. Dessa verktyg får endast köras mot tydligt identifierade mål.

## Thin Provisioning, Discard och kapacitet

Thin Provisioning presenterar mer logiskt än fysiskt tilldelat utrymme. Det ökar utnyttjandet, men skapar en ytterligare felgräns: En värd ser ledigt utrymme medan poolen eller metadataområdet är uttömt. Tillväxt, förbrukning av ögonblicksbilder, reserver och reclaim måste övervakas tillsammans ([Red Hat – Thinly-provisioned logical volumes](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/htmlsingle/configuring_and_managing_logical_volumes/configuring-lvm-on-san-one-machine_configuring-lvm-on-shared-storage)).

TRIM/UNMAP/Discard meddelar underliggande lager om block som inte längre behövs. Om informationen når arrayen eller molnbackend beror på filsystem, monteringsalternativ, hypervisor, protokoll och enhet. Reclaim kan påverka prestanda och dataskydd; på krypterade volymer kan Discard synliggöra beläggningsmönster.

Kapacitetsövervakning omfattar:

- Logiska och fysiska byte samt overcommit;
- Filsystemblock, inoder eller antal objekt;
- Utrymme för ögonblicksbilder, journal, WAL och metadata;
- Reserved Space, reserv för återuppbyggnad/backfill och kvot;
- Papperskorg, Delete Marker, gamla versioner och multipartrester;
- Tillväxt per dataklass och tidpunkt till utarmning.

## HDD, flash och NVMe

HDD:er har mekaniska positioneringstider; slumpmässig I/O och sekventiell genomströmning skiljer sig kraftigt. SSD:er använder Flash Translation Layer, Garbage Collection, Over-Provisioning och Wear Leveling. Skrivförstärkning och återstående livslängd beror på arbetslast och firmware. NVMe minskar protokoll- och köflaskhalsar, men eliminerar inte frågor om medium eller varaktighet.

SMART-/NVMe-hälsovärden är tillverkarspecifika signaler, ingen garanti. Kritiska varningar, Media/Data Integrity Errors, Unsafe Shutdowns, temperatur, Spare och Percentage Used utvärderas tillsammans med applikationsfel och styrenhetsloggar. NVMe-projektet publicerar specifikationerna för styrenheter, hantering och transporter ([NVM Express Specifications](https://nvmexpress.org/specifications/)).

## Kryptering och nyckelgränser

Lagringskryptering kan ske på medium, styrenhet, volym, filsystem, objekt eller applikation. Varje nivå skyddar mot en annan hotmodell:

| Nivå | Skyddar främst | Kvarstående gräns |
|---|---|---|
| Self-Encrypting Drive | borttaget medium vid korrekt låsning | värden ser klartext; nyckel-/firmwaredrift |
| Volume Encryption | offline-lästa block | aktiv värd och privilegierade processer ser klartext |
| Filesystem/fil | utvalda namnområden och användare | metadata kan förbli synliga |
| server-side KMS | leverantörens medier/backends | leverantörens control plane och IAM |
| client-side objektkryptering | leverantören ser chiffertext | sökning/förhandsvisning, nyckelförlust, metadata och migrering |

NIST SP 800-209 kräver för lagringsinfrastrukturer bland annat åtkomstkontroll, säker hantering, dataintegritet, kryptering, isolering och övervakning som en sammanhängande säkerhetsmodell ([NIST SP 800-209 – Security Guidelines for Storage Infrastructure](https://csrc.nist.gov/pubs/sp/800/209/final)).

Key Rotation och återställning testas tillsammans. En säkerhetskopia utan de nycklar och metadata som var giltiga vid säkerhetskopieringstillfället går inte att återställa; en exporterad nyckel utan IAM-/wrappingkontext kan vara lika oanvändbar.

Kapacitet i sig säger lite om en applikations beteende. Avgörande är latens, IOPS, genomströmning och köer längs hela datavägen.

## Prestanda: latens, IOPS, genomströmning och kö

Nominella IOPS utan blockstorlek, läs-/skrivmix, åtkomstmönster, Queue Depth, synkronicitet och latenspercentiler är inte jämförbara. Genomströmningen motsvarar ungefär IOPS gånger överföringsstorleken tills gränser för buss, nätverk, styrenhet eller medium dominerar. Cacheträffar kan helt kringgå mediet.

Minst följande mäts:

- End-to-end-latens ur applikationens perspektiv, inklusive p95/p99;
- IOPS och byte/s separat för läsning/skrivning;
- Queue Depth, utnyttjande och väntetid per enhet/volym;
- Sync-write-latens och flushfrekvens;
- Cacheträff, Dirty Data och throttling;
- Backend-/replikeringslag, återuppbyggnad eller garbage collection;
- Fel, retries, timeouts och path failover.

`fio` kan skapa kontrollerade I/O-arbetslaster; dess dokumentation varnar indirekt genom de flexibla skrivalternativen: Ett felaktigt mål förstör data. Tester körs endast mot dedikerade testfiler eller godkända enheter med verifierad sökväg ([fio Documentation](https://fio.readthedocs.io/en/latest/fio_doc.html)). Microsofts DiskSpd är ett motsvarande syntetiskt lagringsverktyg för Windows ([Microsoft DiskSpd](https://github.com/microsoft/diskspd)).

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

[`Get-Counter`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-counter), [`iostat`](https://man7.org/linux/man-pages/man1/iostat.1.html) och [`pidstat`](https://man7.org/linux/man-pages/man1/pidstat.1.html) övervakar verklig belastning. [`diskspd`](https://github.com/microsoft/diskspd/wiki/Command-line-and-parameters) och [`fio`](https://fio.readthedocs.io/en/latest/fio_doc.html) skapar syntetisk belastning och får aldrig riktas mot produktiva blockenheter eller osäkrade filer.

## Multipathing och Failure Domains

Flera vägar kan överleva fel på styrenhet, HBA, switch eller fabrik, om de verkligen är oberoende. Multipathing identifierar flera vägar till samma logiska enhet och utför failover eller load balancing. Antalet vägar bevisar i sig ingen redundans: Två sessioner via samma switch och styrenhet faller fortfarande bort tillsammans.

Asymmetric Logical Unit Access kan markera föredragna och icke-föredragna styrenhetsvägar. Timeoutvärden måste passa ihop över applikation, filsystem, multipath, transport och array; för korta timeouts skapar falska fel, för långa blockerar köer.

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

[`Get-Disk`](https://learn.microsoft.com/en-us/powershell/module/storage/get-disk), [`Get-PhysicalDisk`](https://learn.microsoft.com/en-us/powershell/module/storage/get-physicaldisk), [`Get-MPIOPath`](https://learn.microsoft.com/en-us/powershell/module/mpio/get-mpiopath) och [`Get-IscsiSession`](https://learn.microsoft.com/en-us/powershell/module/iscsi/get-iscsisession) visar Windows-enheter och vägar. [`lsblk`](https://man7.org/linux/man-pages/man8/lsblk.8.html), [`multipath`](https://manpages.debian.org/bookworm/multipath-tools/multipath.8.en.html), [`iscsiadm`](https://manpages.debian.org/bookworm/open-iscsi/iscsiadm.8.en.html) och [`nvme`](https://github.com/linux-nvme/nvme-cli) visar Linux-mappning, sessioner och namnområden. Serienummer, WWN, LUN och monteringspunkt dubbelkontrolleras före varje ändring.

## Molnlagringsklasser och livscykel

Molnobjektlagring erbjuder klasser med olika priser, minsta lagringstid, kostnader för hämtning och återställningstider. Arkiv kan rehydreras asynkront; Lifecycle Policies kan flytta objekt mellan klasser eller ta bort versioner. Ett billigt kapacitetspris utan pris för request, retrieval, egress och minsta lagringstid är ofullständigt.

AWS dokumenterar Storage Classes med åtkomstmönster och begränsningar; andra leverantörer använder egna klasser och löften ([Amazon S3 Storage Classes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html)). Därför testas RPO, RTO, återställningsbandbredd, requesthastighet, egresskostnader och nyckeltillgänglighet för varje dataklass i en återställningsplan.

Replikering håller ytterligare kopior tillgängliga för tillgänglighet, men överför ofta även fel och raderingar. Säkerhetskopiering och arkiv uppfyller andra mål för tid och oföränderlighet.

## Separera säkerhetskopiering, replikering och arkiv

**Replikering** håller en eller flera kopior tillgängliga synkront eller fördröjt. **Ögonblicksbild** refererar till en tidpunkt i samma lagringskontext. **Säkerhetskopia** skapar en hanterad, återställningsbar kopia med bevarande och katalog. **Arkiv** bevarar data långsiktigt enligt governance- och åtkomstregler. Ett system kan kombinera flera roller; löftena förblir separata.

En robust skyddsplan följer [Backup och Disaster Recovery](/kb/backup-dr) och innehåller:

- Applikationsquiescence eller produktspecifik semantik för onlinebackup;
- Data, metadata, ACL:er, ägare, Extended Attributes och nycklar;
- Separat fel- och administrationsdomän;
- Retention, Immutability, modell för radering och Legal Hold;
- Katalog och beroenden till programvaru-/schemaversioner;
- Regelbunden återställning till en ren målmiljö;
- Integritets- och verksamhetsmässig avstämning efter återställning.

## Dataradering och mediernas slut

Filradering eller Object Delete tar ofta bara bort namn- eller metadatareferenser. Ögonblicksbilder, versioner, repliker, cacheminnen, säkerhetskopior, Flash Translation Layer och defekta medier kan behålla kopior. NIST SP 800-88 beskriver Clear, Purge och Destroy som olika sanitization-metoder beroende på medium, skyddsbehov och återanvändning ([NIST SP 800-88 Rev. 1 – Media Sanitization](https://csrc.nist.gov/pubs/sp/800/88/r1/final)).

I krypterad lagring kan Crypto Erase göra nyckelmaterial obrukbart om all data är effektivt skyddad med de berörda nycklarna och inga klartextkopior finns. Key Hierarchy, Escrow, repliker och säkerhetskopior måste inkluderas i beviset.

## Teknisk historia

Tidiga datorsystem kopplade lagring direkt till en enskild dator. Standardiserade blockprotokoll som SCSI separerade värd- och enhetsfunktion; RAID kombinerade enheter till modeller för prestanda och redundans. Storage Area Networks via Fibre Channel och iSCSI centraliserade blockvolymer, medan Network Attached Storage tillhandahöll filresurser via NFS och SMB.

Logiska volymhanterare och copy-on-write-filsystem separerade fysiska enheter ytterligare från volymer, ögonblicksbilder och dataintegritet. Distribuerade filsystem och objektlagring skalade metadata och data över många noder. Moln-API:er gjorde lagring till en tjänst som provisioneras per request, med livscykel, versionering och användningsbaserad debitering.

Flash och NVMe förändrade prestandagränserna: parallella köer och mikrosekundslatenser gjorde programvaruväg, CPU, NUMA och nätverk viktigare. Samtidigt abstraherar Software-defined Storage och [Container Storage Interface](https://github.com/container-storage-interface/spec/blob/master/spec.md) den fysiska placeringen. Den centrala administratörsfrågan förblev oförändrad: Vilka bekräftade byte överlever vilket fel, och hur upprepas detta bevis regelbundet?

## Administratörschecklista i korthet

En lagringsresurs är först begriplig när skrivbekräftelse, redundans, felbeteende och återställning dokumenteras tillsammans. Tabellen sammanfattar dessa driftsfrågor oberoende av produktnamn.

| Fråga | Driftsbevis |
|---|---|
| Vilken åtkomstmodell? | Blockenhet/LUN/namnområde, filsystem/share eller bucket/nyckel/API |
| Hur ser end-to-end-vägen ut? | Applikation, cache, FS/volym, drivrutin, transport, styrenhet, medium, replik |
| När är en write beständig? | Flush-/fsync-/FUA-semantik, skyddad cache, journal/WAL, quorum |
| Vilken konsistens gäller? | Lokal vy, Read-after-write, quorum, lag, konflikt- och Split-Brain-metod |
| Vilka feldomäner? | Enhet, styrenhet, väg, nod, rack, zon, plats samt admin-/credentialdomän |
| Var finns redundansen? | RAID/Mirror/EC, antal repliker, Failure-Domain-placering, återuppbyggnadsreserv |
| Hur upptäcks integritet? | End-to-end-checksum, scrub, reparationskälla, larm och återställningsjämförelse |
| Vilken kapacitet räknas? | Fysisk/logisk, Thin Pool, metadata, inoder/objekt, ögonblicksbilder/versioner, reserv |
| Vilket prestandalöfte? | Blockstorlek, mix, sync, kö, p95/p99-latens, IOPS och genomströmning |
| Hur återställs data? | Applikationskonsistent backup, nycklar, metadata/ACL:er, katalog och rent återställningstest |

## Källor

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
