---
title: "Almacenamiento: bloques, archivos, objetos e integridad de datos"
blatt: "storage"
description: "Almacenamiento para administradores de infraestructura y mensajería: almacenamiento en bloques, archivos y objetos, ruta de E/S y caché, sistemas de archivos, RAID y codificación de borrado, replicación y consistencia, snapshots, integridad, cifrado, capacidad, rendimiento, modelos de nube, recuperación e historia técnica."
fakten:
  - label: Modelos fundamentales
    wert: Bloque · archivo · objeto; direccionamiento, metadatos y consistencia diferentes
    href: https://www.snia.org/education/storage_networking_primer
  - label: Ruta de bloques
    wert: Aplicación · sistema de archivos/gestor de volúmenes · capa de bloques · controlador · medio
    href: https://docs.kernel.org/block/index.html
  - label: Acceso a archivos
    wert: local mediante VFS o remoto, entre otros, mediante NFS y SMB
    href: https://www.rfc-editor.org/rfc/rfc8881.html
  - label: Modelo de objetos
    wert: Bucket/espacio de nombres · clave · datos · metadatos · operación de API
    href: https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html
  - label: Durabilidad
    wert: solo después de un límite definido de commit/flush; una llamada de escritura correcta por sí sola puede no bastar
    href: https://man7.org/linux/man-pages/man2/fsync.2.html
  - label: Redundancia
    wert: Mirror/RAID/Erasure Coding/replicación aumentan la tolerancia a fallos, pero no son una copia de seguridad
    href: https://www.snia.org/sites/default/files/technical-work/whitepapers/SNIA-Data-Protection-Best-Practices-White%20Paper.pdf
  - label: Consistencia
    wert: la visión local, distribuida y de la aplicación se tratan por separado; documentar Sync, Quorum y Lag
    href: https://www.cs.cornell.edu/fbs/publications/SMSurvey.pdf
  - label: Integridad
    wert: suma de comprobación · verificación de extremo a extremo · Scrub · ruta de reparación
    href: https://openzfs.github.io/openzfs-docs/Basic%20Concepts/Checksums.html
  - label: Snapshots
    wert: referencia temporal en el mismo ámbito de fallos y administración; debe establecerse la consistencia
    href: https://docs.kernel.org/filesystems/fiemap.html
  - label: Ejes de rendimiento
    wert: latencia · IOPS · rendimiento · profundidad de cola · tamaño de bloque · mezcla de lectura/escritura
    href: https://fio.readthedocs.io/en/latest/fio_doc.html
  - label: Ejes de capacidad
    wert: bytes utilizables · espacio reservado · metadatos · inodos/objetos · Thin Provisioning
    href: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/htmlsingle/configuring_and_managing_logical_volumes/configuring-lvm-on-san-one-machine_configuring-lvm-on-shared-storage
  - label: Objeto de recuperación
    wert: datos, metadatos, permisos, claves, versión/snapshot, esquema de aplicación y dependencias
    href: https://csrc.nist.gov/pubs/sp/800/209/final
werbung:
  - newsletter
ctaThemen:
  - rclone
  - proton-drive
  - backup-dr
translationSourceHash: 97205e641b2bc37a7cbb4a5e7de137f527187daf42e5da7e951bd18246a3cede
translationModel: gpt-5.6-terra
translatedAt: 2026-10-13T11:34:18.643Z
translationReview: automatic
---

# Almacenamiento: bloques, archivos, objetos e integridad de datos

El almacenamiento es la base de estado de casi toda infraestructura: buzones, colas, índices de búsqueda, directorios, bases de datos, máquinas virtuales y copias de seguridad constan de bytes, metadatos y una garantía sobre cuándo los cambios son visibles y duraderos. El medio por sí solo no ofrece esta garantía. La aplicación, la caché del sistema operativo, el sistema de archivos, el gestor de volúmenes, el controlador, la red, los mecanismos de redundancia y la replicación remota conforman conjuntamente la ruta de datos.

Por ello, para los administradores hay tres preguntas más importantes que la capacidad y las IOPS nominales antes de elegir un producto: **¿Qué modelo de acceso ve la aplicación? ¿Qué semántica de commit y consistencia se aplica? ¿Qué ámbitos de fallo sobrevive un cambio confirmado?** Solo después pueden evaluarse de forma razonable el rendimiento, la copia de seguridad, el cifrado y los costes.

La explicación sigue una operación de escritura desde la aplicación, pasando por el sistema de archivos o la API de objetos, hasta el volumen, el controlador y el medio. Solo después se comparan la replicación, la consistencia, el rendimiento, el cifrado, la copia de seguridad y la recuperación.

El almacenamiento comienza con una solicitud de escritura de la aplicación y no termina hasta que los datos son duraderos y legibles de nuevo conforme al modelo prometido. Entre ambos puntos se encuentran la caché, el sistema de archivos o la API de objetos, la red, el controlador y el medio. El artículo sigue esta ruta de datos en lugar de limitarse a enumerar tipos de almacenamiento.

## Clasificación técnica y ruta de datos

El almacenamiento se clasifica según la interfaz que recibe un consumidor:

- **El almacenamiento en bloques** direcciona áreas de un dispositivo lógico. El sistema de archivos o la base de datos posee la estructura superior.
- **El almacenamiento de archivos** expone nombres jerárquicos, directorios, atributos, permisos y operaciones como abrir, leer o renombrar.
- **El almacenamiento de objetos** direcciona objetos mediante espacio de nombres/bucket y clave; los datos y metadatos se administran mediante operaciones de API.

SNIA describe Block, File y Object Storage como modelos de servicio y acceso diferentes, no simplemente como tres clases de productos ([SNIA – Storage Networking Primer](https://www.snia.org/education/storage_networking_primer)). Por ejemplo, una puerta de enlace S3 delante de un sistema de archivos cambia el modelo externo de objetos sin que el almacenamiento interno deba estar necesariamente basado en objetos.

Una ruta de escritura síncrona puede atravesar las siguientes capas:

1. aplicación y tiempo de ejecución con su propio buffering o Write-ahead Log;
2. llamada al sistema, Virtual File System o acceso Raw-Block;
3. Page Cache, sistema de archivos y gestor de volúmenes;
4. capa de bloques del kernel y planificador de E/S;
5. controlador de dispositivo, HBA, protocolo de red o capa de almacenamiento virtual;
6. caché del controlador, capa de redundancia/Erasure Coding y medio;
7. réplicas, journal, Quorum o ubicación remota.

La capa de bloques de Linux documenta colas, solicitudes y topología de dispositivos como una capa propia del kernel ([Linux Kernel – Block subsystem](https://docs.kernel.org/block/index.html)). Windows también abstrae por separado los soportes físicos, Storage Pools, discos virtuales, particiones, volúmenes y sistemas de archivos ([Microsoft Learn – Storage overview](https://learn.microsoft.com/en-us/windows-server/storage/storage)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-storage.svg?v=20260813" title="Interaktive Infografik: Storage-Datenpfad von Anwendung und Cache über Dateisystem, Volume, Controller und Medium bis zu Replikation sowie Block-, Datei-, Objekt-, Integritäts- und Recoverymodellen" loading="lazy">
  <a href="/images/kb-interaktiv-storage.svg?v=20260813">Abrir el gráfico interactivo directamente</a>.
</iframe>

## Almacenamiento en bloques

El almacenamiento en bloques proporciona un espacio de direcciones lógico lineal. Un LUN SCSI, destino iSCSI, volumen Fibre Channel, espacio de nombres NVMe, disco virtual y volumen en la nube son transportes o implementaciones distintos del mismo modelo básico. El iniciador ve bloques; los nombres de archivo, directorios y, a menudo, la semántica de bloqueo surgen por encima de ellos.

iSCSI encapsula comandos SCSI sobre TCP y separa Discovery/Login, Session y Connections. RFC 7143 define la semántica del protocolo; Multipathing, autenticación, diseño de red y capacidad de clúster del sistema de archivos siguen siendo capas operativas adicionales ([RFC 7143 – iSCSI](https://www.rfc-editor.org/rfc/rfc7143.html)). NVMe se diseñó para almacenamiento no volátil sobre PCIe y utiliza Submission/Completion Queues; NVMe over Fabrics transporta el modelo, entre otros medios, mediante RDMA o TCP ([NVM Express Specifications](https://nvmexpress.org/specifications/)).

Un volumen de bloques no debe montarse simultáneamente con permiso de escritura desde varios hosts utilizando sistemas de archivos locales convencionales. La operación de bloques compartidos requiere un sistema de archivos de clúster o una aplicación con coordinación distribuida propia. «La SAN presenta el LUN a ambos servidores» no constituye un bloqueo de escritura.

## Sistema de archivos y VFS

Un sistema de archivos asigna áreas de bloques u otros backends a archivos, directorios y metadatos. El Virtual File System proporciona a las aplicaciones una interfaz común de llamadas al sistema; los sistemas de archivos concretos implementan operaciones, caché y durabilidad de formas distintas ([Linux Kernel – Overview of the Linux Virtual File System](https://docs.kernel.org/filesystems/vfs.html)).

El journaling registra metadatos o también datos para restaurar un estado coherente del sistema de archivos tras un fallo. No es una copia de seguridad y no evita que un cambio funcionalmente erróneo se registre correctamente en el journal. ext4 documenta modos como `journal`, `ordered` y `writeback`; XFS utiliza un journal de metadatos y mecanismos de recuperación propios ([Linux Kernel – ext4](https://docs.kernel.org/filesystems/ext4/index.html), [XFS Documentation](https://docs.kernel.org/filesystems/xfs/index.html)).

Los sistemas de archivos Copy-on-write como Btrfs y OpenZFS escriben los bloques modificados en ubicaciones nuevas y actualizan las referencias. Esto permite snapshots y modelos de sumas de comprobación eficientes, pero exige espacio para nuevos bloques, metadatos y Garbage Collection ([Btrfs Documentation](https://btrfs.readthedocs.io/en/latest/), [OpenZFS – Basic Concepts](https://openzfs.github.io/openzfs-docs/Basic%20Concepts/index.html)).

## Archivos remotos: NFS y SMB

NFSv4.1 define un protocolo de archivos remotos con estado, con Sessions, Opens, Locks, Delegations y una operación Compound. La caché del cliente y la recuperación del servidor forman parte del protocolo; un montaje NFS no es simplemente «POSIX local sobre TCP» ([RFC 8881 – NFSv4.1](https://www.rfc-editor.org/rfc/rfc8881.html)).

SMB 2/3 utiliza Credits, Sessions, Tree Connects, Opens y Leases/Oplocks. Microsoft publica la especificación del protocolo; funciones como Multichannel, Transparent Failover, Encryption y Persistent Handles modifican el rendimiento y la semántica de failover ([Microsoft Open Specifications – MS-SMB2](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-smb2/5606ad47-5ee0-437a-817e-70c366052962)).

Para archivos remotos se documentan como mínimo:

- resolución de nombres, destino/VIP y puertos;
- autenticación y asignación de identidades, por ejemplo Kerberos, NTLM o UID/GID;
- versión de dialecto/protocolo, opciones de montaje y cifrado;
- comportamiento de caché, bloqueo, lease y failover;
- semántica ante desconexión de red, timeout y recuperación del servidor;
- cuotas, snapshots y responsabilidad de copia de seguridad.

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

[`Get-SmbConnection`](https://learn.microsoft.com/en-us/powershell/module/smbshare/get-smbconnection) y [`Get-SmbMapping`](https://learn.microsoft.com/en-us/powershell/module/smbshare/get-smbmapping) muestran dialectos SMB y asignaciones activos; [`Get-Volume`](https://learn.microsoft.com/en-us/powershell/module/storage/get-volume) añade la vista de volúmenes locales. [`findmnt`](https://man7.org/linux/man-pages/man8/findmnt.8.html), [`nfsstat`](https://man7.org/linux/man-pages/man8/nfsstat.8.html) y [`smbclient`](https://www.samba.org/samba/docs/current/man-html/smbclient.1.html) comprueban montajes Unix y vistas de protocolos; [`jq`](https://jqlang.org/manual/) filtra su salida JSON. [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) y [`nc`](https://man.openbsd.org/nc) solo prueban la ruta TCP, no la autorización del recurso compartido ni la semántica de E/S.

## Almacenamiento de objetos

El almacenamiento de objetos no utiliza manejadores de archivos abiertos en el sentido clásico. Una operación lee, escribe, enumera, copia o elimina un objeto bajo una clave. Las «carpetas» suelen ser prefijos o proyecciones de interfaz. Por ello, un cambio de nombre puede implementarse como copia en el servidor seguida de eliminación. AWS documenta la API S3 como una interfaz orientada a REST sobre buckets, objetos y operaciones ([Amazon S3 API Reference](https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html)).

Las operaciones de objetos tienen sus propios límites:

- Multipart Upload genera partes y un commit final; las cargas interrumpidas requieren limpieza mediante Lifecycle.
- ETag no es universalmente una suma MD5, especialmente en objetos multipartes o cifrados.
- Listing, Versioning, Delete Marker y Lifecycle son estados independientes.
- Las URL prefirmadas delegan una operación concreta hasta su vencimiento y deben tratarse como credenciales.
- Object Lock/WORM requiere un modelo de Retention, Legal Hold, claves y administración.

Amazon S3 documenta consistencia fuerte de Read-after-write y List para PUT y Delete correctos. Esta garantía de producto no se aplica automáticamente a implementaciones compatibles con S3, cachés o gateways superpuestos ([Amazon S3 – Data consistency model](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html#ConsistencyModel)).

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

[`aws s3api`](https://docs.aws.amazon.com/cli/latest/reference/s3api/) trabaja con operaciones de API S3; los subcomandos mostrados comprueban acceso al bucket, versionado y atributos del objeto. [`rclone size`](https://rclone.org/commands/rclone_size/) cuenta objetos y bytes a través de un backend configurado. Ambos muestran la vista de sus credenciales y su endpoint; por ello también se registran la región, la cuenta y el alias.

Ya sea bloque, archivo u objeto: una operación de escritura confirmada debe entenderse en el contexto de la caché y el flush. «Escrito correctamente» no significa el mismo punto de durabilidad física en todas las pilas.

## Caché, flush y durabilidad

Una llamada `write()` correcta puede significar únicamente que los bytes se copiaron a una caché del sistema operativo. Los sistemas tipo POSIX utilizan `fsync()` o `fdatasync()` para transferir los datos modificados y los metadatos necesarios al dispositivo. La página de manual de Linux indica que, para un archivo creado recientemente, también puede ser necesario sincronizar el directorio contenedor para que la entrada de directorio sea duradera ([fsync(2)](https://man7.org/linux/man-pages/man2/fsync.2.html)).

Por debajo, el controlador y el medio deben implementar correctamente Flush/FUA. Una Write Cache protegida por batería o flash puede conservar escrituras confirmadas ante una pérdida de alimentación; una caché volátil sin protección puede simular rendimiento. La virtualización y el almacenamiento de red prolongan la cadena. El administrador necesita la **garantía de commit de extremo a extremo**, no solo una casilla activada de «Write Cache».

Las bases de datos usan Write-ahead Logging, sumas de comprobación y protocolos de recuperación porque un commit del sistema de archivos no define una transacción funcional. Un snapshot o una réplica solo es coherente con la aplicación si esta utiliza un mecanismo compatible de Freeze, Backup o posicionamiento de logs.

## Consistencia en sistemas de almacenamiento distribuidos

Los sistemas distribuidos replican datos entre procesos y ámbitos de fallo. Una operación puede confirmarse en un nodo, seguir siendo invisible en otro o no poder decidirse ante particiones de red. Los modelos de consistencia describen qué secuencias de observación pueden esperar los clientes. Una visión general de Viotti y Vukolić clasifica modelos fuertes y más débiles basándose en garantías de historial ([Consistency in Non-Transactional Distributed Storage Systems](https://www.cs.cornell.edu/fbs/publications/SMSurvey.pdf)).

Los parámetros prácticos son:

- número y ubicación de réplicas o fragmentos de Erasure Coding;
- Quorum de escritura y lectura, así como límite de acknowledgement;
- replicación de ubicación síncrona o asíncrona;
- comportamiento ante Split Brain, pérdida de mayoría y rejoin;
- Repair, Backfill, Rebalance y Scrub;
- garantía de Staleness, Read-after-write y lecturas monotónicas;
- resolución de conflictos y soberanía de datos preferida.

Ceph describe CRUSH como colocación determinista de objetos mediante dominios de fallo jerárquicos y RADOS como base de objetos distribuida para interfaces de bloque, archivo y objeto ([Ceph Architecture](https://docs.ceph.com/en/latest/architecture/)). La configuración concreta de pool, réplicas, Erasure Coding y Failure Domain determina qué fallos sobrevive realmente el sistema.

La consistencia describe la vista de los datos; la redundancia protege contra el fallo de componentes individuales. RAID y Erasure Coding aumentan la disponibilidad, pero no sustituyen una copia de seguridad independiente.

## RAID, Erasure Coding y reconstrucción

RAID combina varios dispositivos para obtener características de disponibilidad o rendimiento. Mirroring mantiene copias completas; RAID con paridad calcula información de redundancia. El trabajo clásico de Berkeley acuñó los niveles RAID como alternativas arquitectónicas ([Patterson, Gibson, Katz – A Case for Redundant Arrays of Inexpensive Disks](https://www.cs.cmu.edu/~garth/RAIDpaper/Patterson88.pdf)).

Erasure Coding divide los datos en fragmentos de datos y paridad; el objeto puede reconstruirse a partir de una cantidad mínima. Ahorra capacidad frente a la replicación completa, pero aumenta los costes de codificación, red y Repair. Las escrituras aleatorias pequeñas pueden resultar especialmente costosas.

RAID y la replicación no protegen contra:

- eliminación lógica o cifrado por software autorizado;
- corrupción silenciosa que se replica correctamente;
- credenciales comprometidas de administrador o Control Plane;
- fallos fuera del Failure Domain elegido;
- estados incompatibles de software/metadatos;
- pérdida de claves de cifrado.

SNIA describe RAID como un algoritmo de protección de datos sobre varios dispositivos independientes y trata Backup como una técnica de protección separada; por tanto, RAID no implica semántica de copia de seguridad ([SNIA – Data Protection Best Practices](https://www.snia.org/sites/default/files/technical-work/whitepapers/SNIA-Data-Protection-Best-Practices-White%20Paper.pdf)). Durante Rebuild o Backfill disminuye el margen de seguridad, y pueden aumentar la latencia y la probabilidad de fallos. El progreso, la redundancia restante y la tasa de errores deben monitorizarse.

## Snapshots y clones

Un snapshot es una referencia a un estado en un momento determinado. Copy-on-write o Redirect-on-write guarda los cambios posteriores por separado; un clon puede generar una rama escribible a partir de la misma base de bloques. Los snapshots son rápidos y útiles para puntos de recuperación, pruebas y replicación.

Sin embargo, a menudo comparten hardware de almacenamiento, Control Plane, credenciales y ámbito de fallo con la fuente. La pérdida de un pool o un administrador de almacenamiento comprometido puede afectar conjuntamente a la fuente y a los snapshots. Las cadenas de snapshots consumen metadatos y capacidad; eliminar un snapshot antiguo puede desencadenar un trabajo considerable en segundo plano.

La consistencia ante fallos equivale aproximadamente a una pérdida de alimentación abrupta. La consistencia de aplicación requiere Flush, Quiesce, Filesystem Freeze, VSS o una API de Backup específica del producto. Microsoft VSS coordina Requester, Writer y Provider para Shadow Copies coherentes en Windows ([Microsoft Learn – Volume Shadow Copy Service](https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service)).

## Sumas de comprobación, Scrub e integridad de extremo a extremo

Las sumas de comprobación solo detectan cambios si existe un valor de referencia fiable y el área comprobada está claramente definida. CRC de transporte, ECC de dispositivo, suma de comprobación del sistema de archivos, suma de comprobación de objeto y hash de aplicación protegen tramos distintos. Un ETag o la hora de un archivo no son una prueba universal de integridad.

OpenZFS almacena sumas de comprobación para datos y metadatos en punteros de bloques superiores y puede detectar y reparar una copia defectuosa si existe redundancia ([OpenZFS – Checksums](https://openzfs.github.io/openzfs-docs/Basic%20Concepts/Checksums.html)). Btrfs ofrece sumas de comprobación y Scrub en sus dispositivos; Scrub lee datos y metadatos, verifica y repara si existe una copia correcta ([Btrfs – Scrub](https://btrfs.readthedocs.io/en/latest/Scrub.html)).

Un Scrub supone carga de E/S y requiere ventanas temporales, priorización y alertas. «Scrub completado» solo es significativo si se evalúan los errores no corregibles, los errores reparados, las áreas omitidas y los objetos afectados.

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

[`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash) y [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) comparan archivos controlados. [`Get-StorageReliabilityCounter`](https://learn.microsoft.com/en-us/powershell/module/storage/get-storagereliabilitycounter) y [`smartctl`](https://www.smartmontools.org/browser/trunk/smartmontools/smartctl.8.in) leen datos de estado de los dispositivos; [`Repair-Volume`](https://learn.microsoft.com/en-us/powershell/module/storage/repair-volume), [`zpool status`](https://openzfs.github.io/openzfs-docs/man/master/8/zpool-status.8.html) y [`btrfs scrub`](https://btrfs.readthedocs.io/en/latest/btrfs-scrub.html) muestran perspectivas del sistema de archivos/pool. Estas herramientas solo deben ejecutarse sobre destinos claramente identificados.

## Thin Provisioning, Discard y capacidad

Thin Provisioning presenta más espacio lógico que espacio físico asignado. Esto aumenta la utilización, pero introduce un límite de fallo adicional: un host ve espacio libre mientras que el pool o el área de metadatos están agotados. El crecimiento, el consumo de snapshots, las reservas y Reclaim deben monitorizarse conjuntamente ([Red Hat – Thinly-provisioned logical volumes](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/htmlsingle/configuring_and_managing_logical_volumes/configuring-lvm-on-san-one-machine_configuring-lvm-on-shared-storage)).

TRIM/UNMAP/Discard comunica los bloques que ya no se necesitan a las capas inferiores. Que la información llegue hasta el array o backend en la nube depende del sistema de archivos, las opciones de montaje, el hipervisor, el protocolo y el dispositivo. Reclaim puede afectar al rendimiento y a la protección de datos; en volúmenes cifrados, Discard puede revelar patrones de ocupación.

La monitorización de capacidad incluye:

- bytes lógicos y físicos, así como overcommit;
- bloques del sistema de archivos, inodos o número de objetos;
- espacio de snapshots, journal, WAL y metadatos;
- Reserved Space, reserva de Rebuild/Backfill y cuota;
- papelera, Delete Marker, versiones antiguas y restos multipartes;
- crecimiento por clase de datos y momento hasta el agotamiento.

## HDD, Flash y NVMe

Los HDD tienen tiempos de posicionamiento mecánicos; la E/S aleatoria y el rendimiento secuencial difieren mucho. Las SSD utilizan Flash Translation Layer, Garbage Collection, Over-Provisioning y Wear Leveling. La amplificación de escritura y la vida útil restante dependen de la carga de trabajo y el firmware. NVMe reduce los cuellos de botella de protocolo y colas, pero no elimina las cuestiones relativas al medio o la durabilidad.

Los valores SMART/NVMe-Health son señales específicas del fabricante, no una garantía. Las advertencias críticas, Media/Data Integrity Errors, Unsafe Shutdowns, temperatura, Spare y Percentage Used se evalúan junto con errores de aplicación y logs del controlador. El proyecto NVMe publica las especificaciones de controladores, gestión y transportes ([NVM Express Specifications](https://nvmexpress.org/specifications/)).

## Cifrado y límites de claves

El cifrado de almacenamiento puede realizarse en el medio, el controlador, el volumen, el sistema de archivos, el objeto o la aplicación. Cada capa protege frente a un modelo de amenazas distinto:

| Capa | Protege principalmente | Límite restante |
|---|---|---|
| Self-Encrypting Drive | medio retirado con bloqueo correcto | el host ve texto claro; operación de claves/firmware |
| Volume Encryption | bloques leídos sin conexión | el host en ejecución y los procesos privilegiados ven texto claro |
| Filesystem/archivo | espacios de nombres y usuarios seleccionados | los metadatos pueden seguir siendo visibles |
| KMS del lado del servidor | medios/backends del proveedor | Control Plane e IAM del proveedor |
| cifrado de objetos del lado del cliente | el proveedor ve Ciphertext | búsqueda/preview, pérdida de claves, metadatos y migración |

NIST SP 800-209 exige para infraestructuras de almacenamiento, entre otros aspectos, control de acceso, administración segura, integridad de datos, cifrado, aislamiento y monitorización como un modelo de seguridad integrado ([NIST SP 800-209 – Security Guidelines for Storage Infrastructure](https://csrc.nist.gov/pubs/sp/800/209/final)).

Key Rotation y Restore se prueban conjuntamente. Una copia de seguridad sin las claves y metadatos válidos en el momento de realizarse no se puede restaurar; una clave exportada sin contexto de IAM/Wrapping puede ser igualmente inútil.

La capacidad por sí sola dice poco sobre el comportamiento de una aplicación. Son decisivas la latencia, las IOPS, el rendimiento y las colas a lo largo de toda la ruta de datos.

## Rendimiento: latencia, IOPS, rendimiento y cola

Las IOPS nominales sin tamaño de bloque, mezcla de lectura/escritura, patrón de acceso, Queue Depth, sincronía y percentiles de latencia no son comparables. El rendimiento resulta aproximadamente de IOPS por tamaño de transferencia hasta que predominan los límites de bus, red, controlador o medio. Los aciertos de caché pueden evitar por completo el medio.

Se mide como mínimo:

- latencia de extremo a extremo desde la perspectiva de la aplicación, incluidos p95/p99;
- IOPS y bytes/s separados para lectura/escritura;
- Queue Depth, utilización y tiempo de espera por dispositivo/volumen;
- latencia de escritura síncrona y tasa de Flush;
- aciertos de caché, Dirty Data y Throttling;
- lag de backend/replicación, Rebuild o Garbage Collection;
- errores, reintentos, timeouts y failover de ruta.

`fio` puede generar cargas de E/S controladas; su documentación advierte implícitamente mediante las flexibles opciones de escritura: un destino incorrecto destruye datos. Las pruebas se ejecutan únicamente sobre archivos de prueba dedicados o dispositivos autorizados con ruta verificada ([fio Documentation](https://fio.readthedocs.io/en/latest/fio_doc.html)). DiskSpd de Microsoft es una herramienta de almacenamiento sintética equivalente para Windows ([Microsoft DiskSpd](https://github.com/microsoft/diskspd)).

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

[`Get-Counter`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-counter), [`iostat`](https://man7.org/linux/man-pages/man1/iostat.1.html) y [`pidstat`](https://man7.org/linux/man-pages/man1/pidstat.1.html) observan carga real. [`diskspd`](https://github.com/microsoft/diskspd/wiki/Command-line-and-parameters) y [`fio`](https://fio.readthedocs.io/en/latest/fio_doc.html) generan carga sintética y nunca deben dirigirse a dispositivos de bloques en producción ni a archivos sin protección.

## Multipathing y Failure Domains

Varios caminos pueden sobrevivir a fallos de controlador, HBA, switch o fabric si son realmente independientes. Multipathing detecta varias rutas hacia el mismo dispositivo lógico y realiza failover o Load Balancing. El número de rutas por sí solo no demuestra redundancia: dos sesiones a través del mismo switch y controlador siguen fallando juntas.

Asymmetric Logical Unit Access puede señalar rutas de controlador preferidas y no preferidas. Los valores de timeout deben ajustarse entre aplicación, sistema de archivos, Multipath, transporte y array; timeouts demasiado cortos provocan fallos falsos y los demasiado largos bloquean las colas.

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

[`Get-Disk`](https://learn.microsoft.com/en-us/powershell/module/storage/get-disk), [`Get-PhysicalDisk`](https://learn.microsoft.com/en-us/powershell/module/storage/get-physicaldisk), [`Get-MPIOPath`](https://learn.microsoft.com/en-us/powershell/module/mpio/get-mpiopath) y [`Get-IscsiSession`](https://learn.microsoft.com/en-us/powershell/module/iscsi/get-iscsisession) muestran dispositivos y rutas de Windows. [`lsblk`](https://man7.org/linux/man-pages/man8/lsblk.8.html), [`multipath`](https://manpages.debian.org/bookworm/multipath-tools/multipath.8.en.html), [`iscsiadm`](https://manpages.debian.org/bookworm/open-iscsi/iscsiadm.8.en.html) y [`nvme`](https://github.com/linux-nvme/nvme-cli) muestran la asignación de Linux, las sesiones y los espacios de nombres. El número de serie, WWN, LUN y punto de montaje se comprueban antes de cualquier cambio.

## Clases de almacenamiento en la nube y Lifecycle

El almacenamiento de objetos en la nube ofrece clases con distintos precios, retención mínima, costes de recuperación y tiempos de restauración. Archive puede rehidratarse de forma asíncrona; las Lifecycle Policies pueden mover objetos entre clases o eliminar versiones. Una cifra de capacidad barata sin precio de solicitudes, Retrieval, Egress y duración mínima está incompleta.

AWS documenta Storage Classes junto con patrones de acceso y restricciones; otros proveedores utilizan clases y garantías propias ([Amazon S3 Storage Classes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html)). Por ello, para un plan de recuperación se prueban por clase de datos el RPO, RTO, ancho de banda de restauración, tasa de solicitudes, costes de Egress y disponibilidad de claves.

La replicación mantiene más copias para disponibilidad, pero a menudo también propaga errores y eliminaciones. Backup y Archive cumplen objetivos distintos de tiempo e inmutabilidad.

## Separar Backup, replicación y Archive

**La replicación** mantiene disponible una o varias copias de forma síncrona o diferida. **Snapshot** referencia un momento en el mismo contexto de almacenamiento. **Backup** crea una copia administrada y restaurable con retención y catálogo. **Archive** conserva datos a largo plazo conforme a reglas de gobernanza y acceso. Un sistema puede combinar varias funciones; las garantías siguen siendo independientes.

Un plan de protección sólido sigue [Backup y recuperación ante desastres](/kb/backup-dr) e incluye:

- quiescencia de la aplicación o semántica de Backup online específica del producto;
- datos, metadatos, ACL, propietarios, Extended Attributes y claves;
- dominio de errores y administración separado;
- modelo de Retention, Immutability, eliminación y Legal Hold;
- catálogo y dependencias de versiones de software/esquema;
- Restore periódico en un entorno de destino limpio;
- comprobación de integridad y funcional tras el Restore.

## Eliminación de datos y fin de vida del medio

La eliminación de archivos u objetos suele retirar únicamente referencias de nombre o metadatos. Snapshots, versiones, réplicas, cachés, copias de seguridad, Flash Translation Layer y medios defectuosos pueden conservar copias. NIST SP 800-88 describe Clear, Purge y Destroy como procedimientos de sanitización distintos según el medio, el nivel de protección y la reutilización ([NIST SP 800-88 Rev. 1 – Media Sanitization](https://csrc.nist.gov/pubs/sp/800/88/r1/final)).

En almacenamiento cifrado, Crypto Erase puede inutilizar el material de claves si todos los datos están protegidos eficazmente con las claves afectadas y no existen copias en texto claro. La jerarquía de claves, Escrow, réplicas y copias de seguridad deben incluirse en la comprobación.

## Historia técnica

Los primeros sistemas informáticos conectaban el almacenamiento directamente a un único ordenador. Los protocolos de bloques estandarizados, como SCSI, separaron el host y la función de dispositivo; RAID combinó dispositivos en modelos de rendimiento y redundancia. Las Storage Area Networks mediante Fibre Channel e iSCSI centralizaron volúmenes de bloques, mientras que Network Attached Storage proporcionaba recursos de archivos a través de NFS y SMB.

Los gestores de volúmenes lógicos y los sistemas de archivos Copy-on-write separaron aún más los dispositivos físicos de los volúmenes, snapshots e integridad de datos. Los sistemas de archivos distribuidos y el almacenamiento de objetos escalaron metadatos y datos entre muchos nodos. Las API en la nube convirtieron el almacenamiento en un servicio aprovisionado por solicitud, con Lifecycle, Versioning y facturación basada en el uso.

Flash y NVMe cambiaron los límites de rendimiento: las colas paralelas y las latencias de microsegundos hicieron más importantes la ruta de software, CPU, NUMA y red. Al mismo tiempo, Software-defined Storage y [Container Storage Interface](https://github.com/container-storage-interface/spec/blob/master/spec.md) abstraen la ubicación física. La pregunta central del administrador no ha cambiado: ¿qué bytes confirmados sobreviven a qué fallo y cómo se repite periódicamente esta comprobación?

## Lista de comprobación para administradores

Una autorización de almacenamiento solo se entiende cuando se documentan conjuntamente la confirmación de escritura, la redundancia, el comportamiento ante fallos y la restauración. La tabla resume estas cuestiones operativas independientemente del nombre del producto.

| Pregunta | Evidencia operativa |
|---|---|
| ¿Qué modelo de acceso? | dispositivo de bloques/LUN/Namespace, sistema de archivos/Share o Bucket/Key/API |
| ¿Cuál es la ruta de extremo a extremo? | aplicación, caché, FS/volumen, controlador, transporte, controlador, medio, réplica |
| ¿Cuándo es duradera una escritura? | semántica Flush/fsync/FUA, caché protegida, Journal/WAL, Quorum |
| ¿Qué consistencia se aplica? | vista local, Read-after-write, Quorum, Lag, procedimiento de conflicto y Split Brain |
| ¿Qué ámbitos de fallo? | dispositivo, controlador, ruta, nodo, rack, zona, ubicación y dominio de administrador/credenciales |
| ¿Dónde está la redundancia? | RAID/Mirror/EC, número de réplicas, colocación de Failure Domain, reserva de Rebuild |
| ¿Cómo se detecta la integridad? | checksum de extremo a extremo, Scrub, fuente de reparación, alerta y comparación tras Restore |
| ¿Qué capacidad cuenta? | física/lógica, Thin Pool, metadatos, inodos/objetos, snapshots/versiones, reserva |
| ¿Qué garantía de rendimiento? | tamaño de bloque, mezcla, Sync, cola, latencia p95/p99, IOPS y rendimiento |
| ¿Cómo se restaura? | Backup coherente con la aplicación, claves, metadatos/ACL, catálogo y prueba de Restore limpia |

## Fuentes

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
- [Rclone size](https://rclone.org/commands/rclone_size/)
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
