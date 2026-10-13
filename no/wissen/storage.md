---
title: "Lagring: blokk, fil, objekt og dataintegritet"
blatt: "storage"
description: "Lagring for infrastruktur- og meldingsadministratorer: blokk-, fil- og objektlagring, I/O- og cachebane, filsystemer, RAID og Erasure Coding, replikering og konsistens, øyeblikksbilder, integritet, kryptering, kapasitet, ytelse, skymodeller, gjenoppretting og teknisk historie."
fakten:
  - label: Grunnmodeller
    wert: Blokk · fil · objekt; ulik adressering, metadata og konsistens
    href: https://www.snia.org/education/storage_networking_primer
  - label: Blokkbane
    wert: Applikasjon · filsystem/volumbehandler · blokklag · kontroller · medium
    href: https://docs.kernel.org/block/index.html
  - label: Filtilgang
    wert: lokalt via VFS eller eksternt blant annet via NFS og SMB
    href: https://www.rfc-editor.org/rfc/rfc8881.html
  - label: Objektmodell
    wert: Bucket/navnerom · nøkkel · data · metadata · API-operasjon
    href: https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html
  - label: Varighet
    wert: først etter definert commit-/flushgrense; et vellykket write-kall alene kan være utilstrekkelig
    href: https://man7.org/linux/man-pages/man2/fsync.2.html
  - label: Redundans
    wert: Mirror/RAID/Erasure Coding/replikering øker feiltoleransen, men er ikke sikkerhetskopiering
    href: https://www.snia.org/sites/default/files/technical-work/whitepapers/SNIA-Data-Protection-Best-Practices-White%20Paper.pdf
  - label: Konsistens
    wert: lokalt, distribuert og applikasjonsperspektiv holdes atskilt; dokumenter synkronisering, quorum og forsinkelse
    href: https://www.cs.cornell.edu/fbs/publications/SMSurvey.pdf
  - label: Integritet
    wert: kontrollsum · ende-til-ende-verifisering · scrub · reparasjonsbane
    href: https://openzfs.github.io/openzfs-docs/Basic%20Concepts/Checksums.html
  - label: Øyeblikksbilder
    wert: tidspunktsreferanse i samme feil- og administrasjonsområde; konsistens må etableres
    href: https://docs.kernel.org/filesystems/fiemap.html
  - label: Ytelsesakser
    wert: latens · IOPS · gjennomstrømning · Queue Depth · blokkstørrelse · lese-/skrivemiks
    href: https://fio.readthedocs.io/en/latest/fio_doc.html
  - label: Kapasitetsakser
    wert: brukbare byte · reservert plass · metadata · inoder/objekter · Thin Provisioning
    href: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/htmlsingle/configuring_and_managing_logical_volumes/configuring-lvm-on-san-one-machine_configuring-lvm-on-shared-storage
  - label: Gjenopprettingsobjekt
    wert: data, metadata, tillatelser, nøkler, versjon/øyeblikksbilde, applikasjonsskjema og avhengigheter
    href: https://csrc.nist.gov/pubs/sp/800/209/final
werbung:
  - newsletter
ctaThemen:
  - rclone
  - proton-drive
  - backup-dr
translationSourceHash: 97205e641b2bc37a7cbb4a5e7de137f527187daf42e5da7e951bd18246a3cede
translationModel: gpt-5.6-terra
translatedAt: 2026-10-13T11:37:07.821Z
translationReview: automatic
---

# Lagring: blokk, fil, objekt og dataintegritet

Lagring er tilstandsgrunnlaget for nesten all infrastruktur: postbokser, køer, søkeindekser, kataloger, databaser, virtuelle maskiner og sikkerhetskopier består av byte pluss metadata og et løfte om når endringer er synlige og varige. Mediet alene gir ikke dette løftet. Applikasjon, operativsystemcache, filsystem, volumbehandler, kontroller, nettverk, redundansmetode og ekstern replikering utgjør sammen databanen.

For administratorer er derfor tre spørsmål viktigere enn kapasitet og nominelle IOPS før ethvert produktvalg: **Hvilken tilgangsmodell ser applikasjonen? Hvilken commit- og konsistenssemantikk gjelder? Hvilke feilområder overlever en bekreftet endring?** Først deretter kan ytelse, sikkerhetskopiering, kryptering og kostnader vurderes fornuftig.

Forklaringen følger en skriveoperasjon fra applikasjonen via filsystem eller objekt-API til volum, kontroller og medium. Først deretter sammenlignes replikering, konsistens, ytelse, kryptering, sikkerhetskopiering og gjenoppretting.

Lagring begynner med en skriveforespørsel fra applikasjonen og avsluttes først når dataene, etter den lovede modellen, er varige og kan leses igjen. Mellom disse ligger cache, filsystem eller objekt-API, nettverk, kontroller og medium. Artikkelen følger denne databanen i stedet for bare å liste opp lagringstyper.

## Teknisk plassering og databane

Lagring klassifiseres etter grensesnittet en bruker får:

- **Blokklagring** adresserer områder på en logisk enhet. Filsystemet eller databasen eier den overordnede strukturen.
- **Fillagring** eksponerer hierarkiske navn, kataloger, attributter, tillatelser og operasjoner som åpning, lesing eller omdøping.
- **Objektlagring** adresserer objekter via navnerom/bucket og nøkkel; data og metadata administreres med API-operasjoner.

SNIA beskriver blokk-, fil- og objektlagring som ulike tjeneste- og tilgangsmodeller, ikke bare som tre produktklasser ([SNIA – Storage Networking Primer](https://www.snia.org/education/storage_networking_primer)). En S3-gateway foran et filsystem endrer for eksempel den eksterne objektmodellen uten at den interne datalagringen nødvendigvis er objektbasert.

En synkron skrivebane kan gå gjennom følgende lag:

1. Applikasjon og kjøretid med egen buffering eller Write-ahead Log;
2. Systemkall, Virtual File System eller rå blokktilgang;
3. Page Cache, filsystem og volumbehandler;
4. Kernel-blokklag og I/O-planlegger;
5. Enhetsdriver, HBA, nettverksprotokoll eller virtuelt lagringslag;
6. Kontrollercache, redundans-/Erasure-Coding-lag og medium;
7. Replikaer, journal, quorum eller eksternt sted.

Linux-blokklaget dokumenterer køer, forespørsler og enhetstopologi som et eget kjernelag ([Linux Kernel – Block subsystem](https://docs.kernel.org/block/index.html)). Windows abstraherer også fysiske datadisker, Storage Pools, virtuelle disker, partisjoner, volumer og filsystemer separat ([Microsoft Learn – Storage overview](https://learn.microsoft.com/en-us/windows-server/storage/storage)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-storage.svg?v=20260813" title="Interaktive Infografik: Storage-Datenpfad von Anwendung und Cache über Dateisystem, Volume, Controller und Medium bis zu Replikation sowie Block-, Datei-, Objekt-, Integritäts- und Recoverymodellen" loading="lazy">
  <a href="/images/kb-interaktiv-storage.svg?v=20260813">Åpne interaktiv grafikk direkte</a>.
</iframe>

## Blokklagring

Blokklagring tilbyr et lineært logisk adresserom. SCSI LUN, iSCSI Target, Fibre Channel-volum, NVMe Namespace, virtuell disk og skyvolum er ulike transportformer eller implementeringer av samme grunnmodell. Initiatoren ser blokker; filnavn, kataloger og ofte også låsesemantikk oppstår over dette.

iSCSI kapsler SCSI-kommandoer over TCP og skiller mellom Discovery/Login, Session og Connections. RFC 7143 definerer protokollsemantikken; multipathing, autentisering, nettverksdesign og filsystemets klyngeegnethet forblir ytterligere driftslag ([RFC 7143 – iSCSI](https://www.rfc-editor.org/rfc/rfc7143.html)). NVMe ble utformet for ikke-flyktig lagring over PCIe og bruker Submission/Completion Queues; NVMe over Fabrics overfører modellen blant annet over RDMA eller TCP ([NVM Express Specifications](https://nvmexpress.org/specifications/)).

Et blokkvolum må ikke monteres skrivbart samtidig fra flere verter med vanlige lokale filsystemer. Drift med delt blokktilgang krever et klyngefilsystem eller en applikasjon med egen distribuert koordinering. «SAN-et presenterer LUN-en for begge serverne» er ingen skrivelås.

## Filsystem og VFS

Et filsystem tildeler blokkområder eller andre backender til filer, kataloger og metadata. Virtual File System gir applikasjoner et felles systemkallgrensesnitt; konkrete filsystemer implementerer operasjoner, caching og varighet på ulike måter ([Linux Kernel – Overview of the Linux Virtual File System](https://docs.kernel.org/filesystems/vfs.html)).

Journaling logger metadata eller også data for å kunne gjenopprette en konsistent filsystemtilstand etter et krasj. Det er ikke en sikkerhetskopi og hindrer ikke at en faglig feil endring blir journalført korrekt. ext4 dokumenterer modi som `journal`, `ordered` og `writeback`; XFS bruker en metadatajournal og egne gjenopprettingsmekanismer ([Linux Kernel – ext4](https://docs.kernel.org/filesystems/ext4/index.html), [XFS Documentation](https://docs.kernel.org/filesystems/xfs/index.html)).

Copy-on-write-filsystemer som Btrfs og OpenZFS skriver endrede blokker til nye steder og oppdaterer referanser. Det muliggjør effektive øyeblikksbilder og kontrollsummemodeller, men krever plass til nye blokker, metadata og Garbage Collection ([Btrfs Documentation](https://btrfs.readthedocs.io/en/latest/), [OpenZFS – Basic Concepts](https://openzfs.github.io/openzfs-docs/Basic%20Concepts/index.html)).

## Eksterne filer: NFS og SMB

NFSv4.1 definerer en tilstandsbasert ekstern filprotokoll med Sessions, Opens, Locks, Delegations og en Compound-operasjon. Klientcache og servergjenoppretting er en del av protokollen; en NFS-mount er ikke bare «lokal POSIX over TCP» ([RFC 8881 – NFSv4.1](https://www.rfc-editor.org/rfc/rfc8881.html)).

SMB 2/3 bruker Credits, Sessions, Tree Connects, Opens og Leases/Oplocks. Microsoft publiserer protokollspesifikasjonen; funksjoner som Multichannel, Transparent Failover, Encryption og Persistent Handles endrer ytelses- og failoversemantikken ([Microsoft Open Specifications – MS-SMB2](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-smb2/5606ad47-5ee0-437a-817e-70c366052962)).

For eksterne filer dokumenteres minst:

- Navneoppløsning, mål/VIP og porter;
- Autentisering og identitetsmapping, for eksempel Kerberos, NTLM eller UID/GID;
- Dialect/protokollversjon, mountalternativer og kryptering;
- Cache-, låse-, lease- og failoveratferd;
- Semantikk ved nettverksbrudd, timeout og servergjenoppretting;
- Kvoter, øyeblikksbilder og ansvar for sikkerhetskopiering.

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

[`Get-SmbConnection`](https://learn.microsoft.com/en-us/powershell/module/smbshare/get-smbconnection) og [`Get-SmbMapping`](https://learn.microsoft.com/en-us/powershell/module/smbshare/get-smbmapping) viser aktive SMB-Dialects og mappings; [`Get-Volume`](https://learn.microsoft.com/en-us/powershell/module/storage/get-volume) supplerer den lokale volumvisningen. [`findmnt`](https://man7.org/linux/man-pages/man8/findmnt.8.html), [`nfsstat`](https://man7.org/linux/man-pages/man8/nfsstat.8.html) og [`smbclient`](https://www.samba.org/samba/docs/current/man-html/smbclient.1.html) kontrollerer Unix-mounts og protokollvisninger; [`jq`](https://jqlang.org/manual/) filtrerer JSON-utdataene deres. [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) og [`nc`](https://man.openbsd.org/nc) dokumenterer bare TCP-banen, ikke share-autorisasjon eller I/O-semantikk.

## Objektlagring

Objektlagring bruker ikke åpne filhåndtak i klassisk forstand. En operasjon leser, skriver, lister, kopierer eller sletter et objekt under en nøkkel. «Mapper» er ofte prefikser eller UI-projeksjoner. En omdøping kan derfor implementeres som serverbasert kopiering pluss sletting. AWS dokumenterer S3-API-et som et REST-orientert grensesnitt over Buckets, Objects og operasjoner ([Amazon S3 API Reference](https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html)).

Objektoperasjoner har egne grenser:

- Multipart Upload oppretter deler og en avsluttende commit; avbrutte opplastinger krever livssyklusrensing.
- ETag er ikke universelt en MD5-kontrollsum, særlig ikke for Multipart- eller krypterte objekter.
- Listing, Versioning, Delete Marker og Lifecycle er separate tilstander.
- Pre-signed URLs delegerer en konkret operasjon frem til utløp og må behandles som credentials.
- Object Lock/WORM krever en modell for retention, Legal Hold, nøkler og administrasjon.

Amazon S3 dokumenterer sterk Read-after-write- og listkonsistens for vellykkede PUT-er og Deletes. Dette produktløftet gjelder ikke automatisk for S3-kompatible implementeringer, cacher eller overlagrede gatewayer ([Amazon S3 – Data consistency model](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html#ConsistencyModel)).

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

[`aws s3api`](https://docs.aws.amazon.com/cli/latest/reference/s3api/) arbeider med S3-API-operasjoner; de viste underkommandoene kontrollerer bucket-tilgang, versjonering og objektattributter. [`rclone size`](https://rclone.org/commands/rclone_size/) teller objekter og byte over en konfigurert backend. Begge viser perspektivet til sine credentials og sitt endpoint; region, konto og alias loggføres derfor også.

Enten det gjelder blokk, fil eller objekt: En bekreftet skriveoperasjon må forstås i sammenheng med cache og flush. «Vellykket skrevet» betyr ikke det samme punktet for fysisk varighet i enhver stack.

## Cache, flush og varighet

Et vellykket `write()`-kall kan bare bety at byte er kopiert til en operativsystemcache. POSIX-lignende systemer bruker `fsync()` eller `fdatasync()` for å overføre endrede data og nødvendige metadata til enheten. Linux-manualsiden påpeker at også katalogen som inneholder en nyopprettet fil, kan synkroniseres, slik at katalogoppføringen blir varig ([fsync(2)](https://man7.org/linux/man-pages/man2/fsync.2.html)).

Under dette må kontroller og medium implementere Flush/FUA korrekt. En batteri- eller flashbeskyttet Write Cache kan bevare bekreftede Writes ved strømbrudd; en flyktig cache uten beskyttelse kan gi inntrykk av ytelse. Virtualisering og nettverkslagring forlenger kjeden. Administratoren trenger **ende-til-ende-commitløftet**, ikke bare en aktivert avkrysningsboks for «Write Cache».

Databaser bruker Write-ahead Logging, kontrollsummer og gjenopprettingsprotokoller fordi en filsystemcommit ikke definerer en faglig transaksjon. Et øyeblikksbilde eller replika blir først applikasjonskonsistent når applikasjonen bruker en støttet freeze-, sikkerhetskopierings- eller loggposisjoneringsmekanisme.

## Konsistens i distribuerte lagringssystemer

Distribuerte systemer replikerer data over prosesser og feilområder. En operasjon kan være bekreftet på én node, fortsatt usynlig på en annen eller uavgjørbar ved nettverkspartisjoner. Konsistensmodeller beskriver hvilke observasjonsrekkefølger klienter kan forvente. En oversikt av Viotti og Vukolić plasserer sterke og svakere modeller etter historikkgarantier ([Consistency in Non-Transactional Distributed Storage Systems](https://www.cs.cornell.edu/fbs/publications/SMSurvey.pdf)).

Praktiske parametere er:

- Antall og plassering av replikaer eller Erasure-Coding-fragmenter;
- Write- og Read-quorum samt bekreftelsesgrense;
- Synkron eller asynkron stedsreplikering;
- Atferd ved Split Brain, tapt flertall og rejoin;
- Repair, Backfill, Rebalance og Scrub;
- Løfte om staleness, Read-after-write og monotone reads;
- Konfliktløsning og foretrukket dataautoritet.

Ceph beskriver CRUSH som deterministisk plassering av objekter over hierarkiske Failure Domains og RADOS som distribuert objektgrunnlag for blokk-, fil- og objektgrensesnitt ([Ceph Architecture](https://docs.ceph.com/en/latest/architecture/)). Den konkrete konfigurasjonen av pool, replika, Erasure Coding og Failure Domain avgjør hvilke feil systemet faktisk overlever.

Konsistens beskriver visningen av data; redundans beskytter mot bortfall av enkeltkomponenter. RAID og Erasure Coding øker tilgjengeligheten, men erstatter ikke uavhengig sikkerhetskopiering.

## RAID, Erasure Coding og rebuild

RAID kombinerer flere enheter for tilgjengelighets- eller ytelsesegenskaper. Mirroring holder fullstendige kopier; Parity-RAID beregner redundansinformasjon. Det klassiske Berkeley-arbeidet etablerte RAID-nivåene som arkitekturalternativer ([Patterson, Gibson, Katz – A Case for Redundant Arrays of Inexpensive Disks](https://www.cs.cmu.edu/~garth/RAIDpaper/Patterson88.pdf)).

Erasure Coding deler data i data- og paritetsfragmenter; objektet kan rekonstrueres fra et minimumsantall. Det sparer kapasitet sammenlignet med full replikering, men øker kostnadene for koding, nettverk og reparasjon. Små tilfeldige Writes kan være særlig kostbare.

RAID og replikering beskytter ikke mot:

- Logisk sletting eller kryptering utført av autorisert programvare;
- Stille korrupsjon som replikeres korrekt;
- Kompromitterte administrator- eller Control Plane-credentials;
- Feil utenfor den valgte Failure Domain;
- Inkompatible programvare-/metadatatilstander;
- Tap av krypteringsnøkler.

SNIA beskriver RAID som en databeskyttelsesalgoritme over flere uavhengige enheter og behandler sikkerhetskopiering som en separat beskyttelsesteknikk; RAID gir derfor ingen sikkerhetskopieringssemantikk ([SNIA – Data Protection Best Practices](https://www.snia.org/sites/default/files/technical-work/whitepapers/SNIA-Data-Protection-Best-Practices-White%20Paper.pdf)). Under rebuild eller backfill reduseres sikkerhetsmarginen, og latenstid samt sannsynligheten for feil kan øke. Fremdrift, gjenværende redundans og feilrate hører hjemme i overvåkingen.

## Øyeblikksbilder og kloner

Et øyeblikksbilde er en referanse til en tilstand på et tidspunkt. Copy-on-write eller Redirect-on-write lagrer senere endringer separat; en klone kan opprette en skrivbar gren fra samme blokkgrunnlag. Øyeblikksbilder er raske og nyttige for gjenopprettingspunkter, testing og replikering.

De deler imidlertid ofte lagringsmaskinvare, Control Plane, credentials og feilområde med kilden. Tap av en pool eller en kompromittert lagringsadministrator kan ramme både kilde og øyeblikksbilder. Øyeblikksbildekjeder bruker metadata og kapasitet; sletting av et gammelt øyeblikksbilde kan utløse betydelig bakgrunnsarbeid.

Krasjkonsistens tilsvarer omtrent et brått strømbrudd. Applikasjonskonsistens krever flush, quiesce, Filesystem Freeze, VSS eller et produktspesifikt sikkerhetskopierings-API. Microsoft VSS koordinerer Requester, Writer og Provider for konsistente Shadow Copies i Windows ([Microsoft Learn – Volume Shadow Copy Service](https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service)).

## Kontrollsummer, scrub og ende-til-ende-integritet

Kontrollsummer oppdager bare endringer dersom en pålitelig referanseverdi finnes og det kontrollerte området er klart definert. Transport-CRC, enhets-ECC, filsystemkontrollsum, objektkontrollsum og applikasjonshash beskytter ulike strekninger. En ETag eller filtidsstempel er ikke et universelt integritetsbevis.

OpenZFS lagrer kontrollsummer for data og metadata i overordnede blokkpunkter og kan ved redundans oppdage og reparere en feilaktig kopi ([OpenZFS – Checksums](https://openzfs.github.io/openzfs-docs/Basic%20Concepts/Checksums.html)). Btrfs tilbyr kontrollsummer og scrub over sine enheter; scrub leser data og metadata, verifiserer og reparerer når en god kopi finnes ([Btrfs – Scrub](https://btrfs.readthedocs.io/en/latest/Scrub.html)).

En scrub gir I/O-belastning og krever tidsvinduer, prioritering og varsling. «Scrub fullført» er bare meningsfullt dersom feil som ikke kan korrigeres, reparerte feil, utelatte områder og berørte objekter vurderes.

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

[`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash) og [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) sammenligner kontrollerte filer. [`Get-StorageReliabilityCounter`](https://learn.microsoft.com/en-us/powershell/module/storage/get-storagereliabilitycounter) og [`smartctl`](https://www.smartmontools.org/browser/trunk/smartmontools/smartctl.8.in) leser enhetshelsedata; [`Repair-Volume`](https://learn.microsoft.com/en-us/powershell/module/storage/repair-volume), [`zpool status`](https://openzfs.github.io/openzfs-docs/man/master/8/zpool-status.8.html) og [`btrfs scrub`](https://btrfs.readthedocs.io/en/latest/btrfs-scrub.html) viser filsystem-/poolperspektiver. Disse verktøyene må bare kjøres mot tydelig identifiserte mål.

## Thin Provisioning, Discard og kapasitet

Thin Provisioning presenterer mer logisk enn fysisk tildelt plass. Det øker utnyttelsen, men skaper en ekstra feilgrense: En vert ser ledig plass mens poolen eller metadataområdet er oppbrukt. Vekst, forbruk av øyeblikksbilder, reserver og reclaim må overvåkes sammen ([Red Hat – Thinly-provisioned logical volumes](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/htmlsingle/configuring_and_managing_logical_volumes/configuring-lvm-on-san-one-machine_configuring-lvm-on-shared-storage)).

TRIM/UNMAP/Discard melder blokker som ikke lenger trengs til underliggende lag. Om informasjonen når arrayet eller skybackend, avhenger av filsystem, mountalternativ, hypervisor, protokoll og enhet. Reclaim kan påvirke ytelse og personvern; på krypterte volumer kan Discard synliggjøre bruksmønstre.

Kapasitetsovervåking omfatter:

- Logiske og fysiske byte samt overcommit;
- Filsystemblokker, inoder eller objektantall;
- Plass for øyeblikksbilde, journal, WAL og metadata;
- Reserved Space, rebuild-/backfillreserve og kvote;
- Papirkurv, Delete Marker, gamle versjoner og Multipart-rester;
- Vekst per dataklasse og tidspunkt til uttømming.

## HDD, flash og NVMe

HDD-er har mekaniske posisjoneringstider; tilfeldig I/O og sekvensiell gjennomstrømning skiller seg sterkt. SSD-er bruker Flash Translation Layer, Garbage Collection, Over-Provisioning og Wear Leveling. Skriveforsterkning og gjenværende levetid avhenger av arbeidslast og fastvare. NVMe reduserer protokoll- og køflaskehalser, men fjerner ikke spørsmål om medium eller varighet.

SMART-/NVMe-helseverdier er produsentspesifikke signaler, ikke garantier. Kritiske advarsler, Media/Data Integrity Errors, Unsafe Shutdowns, temperatur, Spare og Percentage Used vurderes sammen med applikasjonsfeil og kontrollerlogger. NVMe-prosjektet publiserer spesifikasjonene for kontrollere, administrasjon og transporter ([NVM Express Specifications](https://nvmexpress.org/specifications/)).

## Kryptering og nøkkelgrenser

Lagringskryptering kan skje på medium-, kontroller-, volum-, filsystem-, objekt- eller applikasjonsnivå. Hvert lag beskytter mot en annen trusselmodell:

| Nivå | Beskytter primært | Gjenværende grense |
|---|---|---|
| Self-Encrypting Drive | fjernet medium ved korrekt låsing | verten ser klartekst; nøkkel-/fastvaredrift |
| Volume Encryption | blokker lest offline | kjørende vert og privilegerte prosesser ser klartekst |
| Filsystem/fil | utvalgte navnerom og brukere | metadata kan fortsatt være synlige |
| serverside KMS | leverandørens medier/backender | leverandørens Control Plane og IAM |
| klientside objektkryptering | leverandøren ser krypteringstekst | søk/forhåndsvisning, nøkkeltap, metadata og migrering |

NIST SP 800-209 krever for lagringsinfrastrukturer blant annet tilgangskontroll, sikker administrasjon, dataintegritet, kryptering, isolasjon og overvåking som en sammenhengende sikkerhetsmodell ([NIST SP 800-209 – Security Guidelines for Storage Infrastructure](https://csrc.nist.gov/pubs/sp/800/209/final)).

Key Rotation og gjenoppretting testes sammen. En sikkerhetskopi uten nøklene og metadataene som var gyldige ved sikkerhetskopieringen, kan ikke gjenopprettes; en eksportert nøkkel uten IAM-/wrappingkontekst kan være like ubrukelig.

Kapasitet alene sier lite om en applikasjons atferd. Avgjørende er latenstid, IOPS, gjennomstrømning og køer langs hele databanen.

## Ytelse: latens, IOPS, gjennomstrømning og kø

Nominelle IOPS uten blokkstørrelse, lese-/skrivemiks, tilgangsmønster, Queue Depth, synkronitet og latenspercentiler kan ikke sammenlignes. Gjennomstrømning tilsvarer omtrentlig IOPS multiplisert med overføringsstørrelse, til buss-, nettverks-, kontroller- eller mediegrenser dominerer. Cachetreff kan omgå mediet fullstendig.

Det måles minst:

- Ende-til-ende-latens fra applikasjonens perspektiv, inkludert p95/p99;
- IOPS og byte/s atskilt etter lesing/skriving;
- Queue Depth, utnyttelse og ventetid per enhet/volum;
- Sync-Write-latens og flushrate;
- Cachetreff, Dirty Data og throttling;
- Backend-/replikeringsforsinkelse, rebuild eller Garbage Collection;
- Feil, retries, timeouts og bane-failover.

`fio` kan generere kontrollerte I/O-arbeidslaster; dokumentasjonen advarer implisitt gjennom de fleksible skrivealternativene: Feil mål ødelegger data. Tester kjøres bare på dedikerte testfiler eller godkjente enheter med verifisert bane ([fio Documentation](https://fio.readthedocs.io/en/latest/fio_doc.html)). Microsofts DiskSpd er et tilsvarende syntetisk lagringsverktøy for Windows ([Microsoft DiskSpd](https://github.com/microsoft/diskspd)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Storage-Performanceanalyse">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Counter '\\PhysicalDisk(*)\\Avg. Disk sec/Transfer','\\PhysicalDisk(*)\\Disk Transfers/sec' `
  -SampleInterval 2 -MaxSamples 15
diskspd.exe -c2G -d30 -r -w30 -b8K -o4 -t2 $env:TEST_FILE
Get-PhysicalDisk | Get-StorageReliabilityCounter</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">iostat -x -d 2 15
fio --name=controlled-test --filename="$TEST_FILE" --size=2G \\
  --rw=randrw --rwmixread=70 --bs=8k --iodepth=4 --runtime=30 --time_based
pidstat -d 2 15</code></pre>
  </div>
</div>

[`Get-Counter`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-counter), [`iostat`](https://man7.org/linux/man-pages/man1/iostat.1.html) og [`pidstat`](https://man7.org/linux/man-pages/man1/pidstat.1.html) observerer reell belastning. [`diskspd`](https://github.com/microsoft/diskspd/wiki/Command-line-and-parameters) og [`fio`](https://fio.readthedocs.io/en/latest/fio_doc.html) genererer syntetisk belastning og rettes aldri mot produksjonsblokkenenheter eller usikrede filer.

## Multipathing og Failure Domains

Flere baner kan overleve feil i kontroller, HBA, switch eller fabric dersom de faktisk er uavhengige. Multipathing oppdager flere veier til samme logiske enhet og utfører failover eller lastbalansering. Antall baner beviser ikke redundans: To sesjoner over samme switch og kontroller feiler fortsatt sammen.

Asymmetric Logical Unit Access kan markere foretrukne og ikke-foretrukne kontrollerbaner. Timeoutverdier må passe sammen på tvers av applikasjon, filsystem, multipath, transport og array; for korte timeouts skaper falske feil, for lange blokkerer køer.

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

[`Get-Disk`](https://learn.microsoft.com/en-us/powershell/module/storage/get-disk), [`Get-PhysicalDisk`](https://learn.microsoft.com/en-us/powershell/module/storage/get-physicaldisk), [`Get-MPIOPath`](https://learn.microsoft.com/en-us/powershell/module/mpio/get-mpiopath) og [`Get-IscsiSession`](https://learn.microsoft.com/en-us/powershell/module/iscsi/get-iscsisession) viser Windows-enheter og -baner. [`lsblk`](https://man7.org/linux/man-pages/man8/lsblk.8.html), [`multipath`](https://manpages.debian.org/bookworm/multipath-tools/multipath.8.en.html), [`iscsiadm`](https://manpages.debian.org/bookworm/open-iscsi/iscsiadm.8.en.html) og [`nvme`](https://github.com/linux-nvme/nvme-cli) viser Linux-tilordning, sesjoner og namespaces. Serienummer, WWN, LUN og mountpoint krysskontrolleres før hver endring.

## Skylagringsklasser og livssyklus

Skyobjektlagring tilbyr klasser med ulike priser, minimumsoppbevaring, hentekostnader og gjenopprettingstider. Arkiv kan rehydreres asynkront; Lifecycle Policies kan flytte objekter mellom klasser eller slette versjoner. Et lavt kapasitetstall uten pris for forespørsler, retrieval, egress og minimumsvarighet er ufullstendig.

AWS dokumenterer Storage Classes med tilgangsmønstre og begrensninger; andre leverandører bruker egne klasser og løfter ([Amazon S3 Storage Classes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html)). For en gjenopprettingsplan testes derfor RPO, RTO, gjenopprettingsbåndbredde, forespørselsrate, egresskostnader og nøkkeltilgjengelighet per dataklasse.

Replikering holder ytterligere kopier tilgjengelige, men overtar ofte også feil og slettinger. Sikkerhetskopiering og arkiv oppfyller andre mål for tid og uforanderlighet.

## Skill mellom sikkerhetskopiering, replikering og arkiv

**Replikering** holder én eller flere kopier tilgjengelige synkront eller tidsforskjøvet. **Øyeblikksbilde** refererer til et tidspunkt i samme lagringskontekst. **Sikkerhetskopiering** lager en administrert, gjenopprettbar kopi med oppbevaring og katalog. **Arkiv** bevarer data på lang sikt etter regler for governance og tilgang. Ett system kan kombinere flere roller; løftene forblir separate.

En robust beskyttelsesplan følger [Sikkerhetskopiering og Disaster Recovery](/kb/backup-dr) og inneholder:

- Applikasjonsquieszenz eller produktspesifikk semantikk for online-sikkerhetskopiering;
- Data, metadata, ACL-er, eiere, Extended Attributes og nøkler;
- Separate feil- og administrasjonsdomener;
- Retention, Immutability, modell for sletting og Legal Hold;
- Katalog og avhengigheter til programvare-/skemaversjoner;
- Regelmessig gjenoppretting til et rent målmiljø;
- Integritets- og faglig avstemming etter gjenoppretting.

## Datasletting og slutten på mediets levetid

Sletting av filer eller objekter fjerner ofte bare navne- eller metadatareferanser. Øyeblikksbilder, versjoner, replikaer, cacher, sikkerhetskopier, Flash Translation Layer og defekte medier kan beholde kopier. NIST SP 800-88 beskriver Clear, Purge og Destroy som ulike sanitiseringsprosedyrer avhengig av medium, beskyttelsesbehov og gjenbruk ([NIST SP 800-88 Rev. 1 – Media Sanitization](https://csrc.nist.gov/pubs/sp/800/88/r1/final)).

Ved kryptert lagring kan Crypto Erase gjøre nøkkelmateriale ubrukelig dersom alle data er effektivt beskyttet med de berørte nøklene og ingen klartekstkopier finnes. Key Hierarchy, Escrow, replikaer og sikkerhetskopier må inkluderes i dokumentasjonen.

## Teknisk historie

Tidlige datasystemer koblet lagring direkte til én enkelt datamaskin. Standardiserte blokkprotokoller som SCSI skilte vert og enhetsfunksjon; RAID kombinerte enheter til modeller for ytelse og redundans. Storage Area Networks over Fibre Channel og iSCSI sentraliserte blokkvolumer, mens Network Attached Storage leverte fildelinger over NFS og SMB.

Logiske volumbehandlere og Copy-on-write-filsystemer skilte fysiske enheter videre fra volumer, øyeblikksbilder og dataintegritet. Distribuerte filsystemer og objektlagring skalerte metadata og data over mange noder. Sky-API-er gjorde lagring til en tjeneste som klargjøres per forespørsel, med livssyklus, versioning og bruksbasert fakturering.

Flash og NVMe endret ytelsesgrensene: Parallelle køer og mikrosekundlatens gjorde programvarebane, CPU, NUMA og nettverk viktigere. Samtidig abstraherer programvaredefinert lagring og [Container Storage Interface](https://github.com/container-storage-interface/spec/blob/master/spec.md) den fysiske plasseringen. Det sentrale administrasjonsspørsmålet er uendret: Hvilke bekreftede byte overlever hvilken feil, og hvordan gjentas dette beviset regelmessig?

## Administratorsjekkliste på et øyeblikk

En lagringsgodkjenning er først forståelig når skrivebekreftelse, redundans, feilatferd og gjenoppretting er dokumentert samlet. Tabellen oppsummerer disse drifts spørsmålene uavhengig av produktnavn.

| Spørsmål | Driftsbevis |
|---|---|
| Hvilken tilgangsmodell? | Blokkenhet/LUN/Namespace, filsystem/share eller bucket/nøkkel/API |
| Hva er ende-til-ende-banen? | Applikasjon, cache, FS/volum, driver, transport, kontroller, medium, replika |
| Når er en Write varig? | Flush-/fsync-/FUA-semantikk, beskyttet cache, journal/WAL, quorum |
| Hvilken konsistens gjelder? | Lokal visning, Read-after-write, quorum, lag, konflikt- og Split-Brain-prosedyrer |
| Hvilke feilområder? | Enhet, kontroller, bane, node, rack, sone, sted og administrator-/credentialdomene |
| Hvor finnes redundans? | RAID/Mirror/EC, antall replikaer, Failure-Domain-plassering, rebuildreserve |
| Hvordan oppdages integritet? | Ende-til-ende-kontrollsum, scrub, reparasjonskilde, alarm og gjenopprettingssammenligning |
| Hvilken kapasitet teller? | Fysisk/logisk, Thin Pool, metadata, inoder/objekter, øyeblikksbilder/versjoner, reserve |
| Hvilket ytelsesløfte? | Blokkstørrelse, miks, synkronisering, kø, p95/p99-latens, IOPS og gjennomstrømning |
| Hvordan gjenopprettes det? | Applikasjonskonsistent sikkerhetskopiering, nøkler, metadata/ACL-er, katalog og ren gjenopprettingstest |

## Kilder

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
