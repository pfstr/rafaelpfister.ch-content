---
title: "Rclone: remotos, comparación de datos y funcionamiento de VFS"
blatt: "rclone"
description: "Rclone para administradores de almacenamiento, plataformas y mensajería: arquitectura Go y modelo de backend uniforme, modelo de remotos y configuración, Copy/Sync/Move/Bisync, comparación y sumas de comprobación, Crypt, montaje y caché VFS, Retry/Pacer, filtros, API RC, registros, seguridad, recuperación e historia técnica."
fakten:
  - label: Función del sistema
    wert: Herramienta de CLI y API para copia, sincronización, verificación, montaje y publicación entre almacenamientos de archivos y objetos
    href: https://rclone.org/
  - label: Pila tecnológica
    wert: Proyecto de código abierto en Go; los backends y comandos se integran en un binario rclone portable
    href: https://github.com/rclone/rclone
  - label: Modelo de direccionamiento
    wert: remoto con nombre más dos puntos y ruta; las rutas locales permanecen sin prefijo de remoto
    href: https://rclone.org/docs/
  - label: Contrato de backend
    wert: interfaz Fs/Object común con capacidades opcionales para Hash, ModTime, Copy, Move, Polling, Metadata y ListR
    href: https://rclone.org/overview/
  - label: Dirección
    wert: copy añade sin limpiar el destino · sync hace que el destino refleje la fuente · move elimina la fuente solo después de una copia correcta
    href: https://rclone.org/commands/rclone_sync/
  - label: Comparación
    wert: tamaño más ModTime o hash común; los flags y las capacidades del backend modifican la huella
    href: https://rclone.org/overview/
  - label: Prueba de integridad
    wert: verificación de transferencia, rclone check, cryptcheck o comparación mediante descarga; el código de salida y el registro final forman parte del resultado
    href: https://rclone.org/commands/rclone_check/
  - label: Ruta de montaje
    wert: VFS mediante FUSE/WinFsp; Directory Cache, Buffer y File Cache se sitúan entre la aplicación y el remoto
    href: https://rclone.org/commands/rclone_mount/
  - label: Cifrado
    wert: Crypt cifra los contenidos con NaCl SecretBox y, de forma predeterminada, también los nombres; la configuración o contraseña y el salt son objetos de recuperación
    href: https://rclone.org/crypt/
  - label: Control
    wert: API Remote Control y endpoint OpenMetrics; la accesibilidad equivale en la práctica a permisos de shell del usuario de rclone
    href: https://rclone.org/rc/
  - label: Regla de errores
    wert: Retry de bajo nivel, Pacer y reintentos de comandos; solo hay éxito tras el estado final y el código de salida, no tras un mensaje de error aislado
    href: https://rclone.org/docs/#retries
  - label: Objeto de recuperación
    wert: Binario, configuración, credenciales, filtros, definición del trabajo, archivos de trabajo de Bisync, caché VFS, versiones de destino e informe de verificación
    href: https://rclone.org/docs/#configuration-encryption
werbung:
  - newsletter
ctaThemen:
  - rclone
translationSourceHash: bb7131a94d721cf6984869b9d907a4d2089758b80e651c8b03bd86f72841d42e
translationModel: gpt-5.6-terra
translatedAt: 2026-10-15T08:50:33.707Z
translationReview: automatic
---

# Rclone: remotos, comparación de datos y funcionamiento de VFS

Rclone es una herramienta de movimiento de datos de código abierto para sistemas de archivos locales, protocolos de red, unidades en la nube y almacenamiento de objetos. La [página del proyecto Rclone](https://rclone.org/) lo describe como un programa de línea de comandos para gestionar archivos en almacenamientos en la nube. Proporciona a estos sistemas una interfaz común para listing, copia, sincronización, verificación, montaje, cifrado y acceso de servicio. Esta uniformidad es útil, pero no debe confundirse con una semántica de almacenamiento uniforme: los hashes, las horas de modificación, las mayúsculas/minúsculas, los directorios vacíos, los metadatos, las copias del lado del servidor y la consistencia difieren según el backend.

Para los administradores, rclone no es un «explorador de nube», sino una canalización de datos. Su corrección depende de la fuente y el destino, la dirección, el criterio de comparación, el conjunto de filtros, la regla de eliminación, el estado de reintento, las credenciales, las capacidades del backend y la verificación final. Un proceso puede transferir el 100 por ciento de los bytes y aun así ser técnicamente incorrecto si utiliza el espacio de nombres equivocado, una vista de listado obsoleta o un conjunto de filtros incompleto.

La explicación sigue un comando de rclone desde la fuente y el remoto, pasando por la comparación, los checkers y las transferencias, hasta el backend. A continuación se explican filtros, Crypt, VFS, automatización, seguridad y verificación a lo largo de la misma ruta del trabajo.

Rclone traduce un modelo uniforme de archivos y directorios a las API de servicios de almacenamiento muy distintos. Por ello, el artículo empieza por los remotos y las capacidades de backend, después aborda la semántica de copia y la verificación, y termina con el montaje, la seguridad y la recuperación.

## Arquitectura: comando, modelo de sistema de archivos y backend

El programa escrito en Go conecta varias capas:

1. **Comando y definición del trabajo:** `copy`, `sync`, `move`, `check`, `bisync`, `mount`, `serve` o una llamada RC definen la dirección y la operación.
2. **Filtro y comparación:** las reglas de ruta, el tamaño, la hora de modificación, el hash, los metadatos y los flags determinan el conjunto de objetos considerado y la igualdad.
3. **Motor de transferencia:** los listers, checkers y trabajadores de transferencia planifican operaciones, limitan el paralelismo y recopilan errores.
4. **Modelo de interfaces Fs/Object:** Rclone modela directorios, objetos y funciones opcionales de backend mediante interfaces Go comunes.
5. **Adaptadores de backend:** S3, SFTP, WebDAV, Drive, OneDrive, sistemas de archivos locales y otros servicios traducen el contrato a sus API.
6. **Transporte y proveedor:** HTTP/TLS, SSH/SFTP o I/O local se enfrentan a límites de tasa, consistencia, versionado y políticas del proveedor.

La vista general oficial de backends muestra explícitamente qué sistemas admiten hashes, ModTime, nombres que no distinguen mayúsculas/minúsculas, duplicados, tipos MIME y metadatos ([Overview of cloud storage systems](https://rclone.org/overview/)). El árbol de fuentes separa backends, comandos y el modelo de interfaces común `fs`; las compilaciones oficiales integran los adaptadores seleccionados en un binario ([rclone source repository](https://github.com/rclone/rclone), [Install from source](https://github.com/rclone/rclone/blob/master/docs/content/install.md#source)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-rclone.svg?v=20260813" title="Interaktive Infografik: rclone-Job von Quelle, Remote und Filter über Listing, Vergleich, Checker, Transfer, Backend und Provider bis zu VFS, Crypt, Logs, Exitcode, Verifikation und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-rclone.svg?v=20260813">Abrir directamente el gráfico interactivo</a>.
</iframe>

## Sintaxis de remotos y espacio de nombres

Un remoto configurado se accede como `name:pfad`. Los dos puntos separan el nombre del remoto de la ruta; una ruta local sin prefijo de remoto utiliza el backend Local. Las cadenas de conexión como `:sftp,host=example.net:/data` pueden instanciar un backend sin un nombre de remoto guardado. Esta flexibilidad también aumenta el riesgo de que un error tipográfico designe una ruta local en vez de remota ([Rclone documentation – remotes](https://rclone.org/docs/)).

En comandos de directorio, Rclone transfiere el **contenido** de la ruta de origen, no automáticamente el nombre del directorio de origen. La semántica corresponde a la barra final conocida de `rsync`. Por ello, la fuente, el destino y el árbol de destino esperado se documentan como ejemplos concretos en el runbook ([rclone copy](https://rclone.org/commands/rclone_copy/)).

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

[`rclone version`](https://rclone.org/commands/rclone_version/) identifica el binario, el sistema operativo y la compilación; [`rclone config file`](https://rclone.org/commands/rclone_config_file/) muestra la configuración realmente cargada; [`rclone listremotes`](https://rclone.org/commands/rclone_listremotes/) inventaría los remotos; [`rclone backend features`](https://rclone.org/commands/rclone_backend/) proporciona las capacidades del backend resuelto; [`rclone about`](https://rclone.org/commands/rclone_about/) consulta información de capacidad, si el servicio la ofrece.

## Capacidades de backend y diferencias transparentes

La interfaz común solo puede utilizar capacidades que proporciona el backend o un wrapper. Las diferencias importantes son:

| Capacidad | Significado | Efecto si falta o difiere |
|---|---|---|
| hash común | huella de contenido del proveedor | la comparación recurre a tamaño/ModTime o descarga |
| ModTime modificable | la hora de origen se puede establecer sin volver a cargar | se puede transferir de nuevo contenido idéntico |
| Copy/Move del lado del servidor | los datos permanecen en el proveedor | de lo contrario, descarga/carga o Copy más Delete |
| ListR/Fast List | listado recursivo en menos llamadas | más solicitudes API o más memoria del cliente |
| Change Polling | señala cambios externos | VFS no ve los cambios hasta que expire la caché |
| directorios vacíos | objeto de directorio/marcador real | las carpetas vacías desaparecen en modelos de bucket |
| distinción entre mayúsculas/minúsculas | `A` y `a` distinguibles | sincronización infinita o colisión de nombres en el destino |
| nombres duplicados | varios objetos con el mismo nombre | la ruta no es un identificador único |
| Metadata/MIME | atributos adicionales de lectura/escritura | las ACL o metadatos de negocio no se siguen automáticamente |

Los niveles de soporte evalúan los backends según pruebas y capacidades, no según una promesa general de funcionalidad ([Backend Support Tiers](https://rclone.org/tiers/)). Por tanto, la documentación del proveedor y del backend forma parte de la definición del trabajo. Un servicio compatible con S3 puede tener límites distintos de multipart, hash o copia del lado del servidor que otro proveedor S3.

## Configuración, credenciales y orden de resolución

`rclone.conf` almacena definiciones de remotos y, a menudo, tokens OAuth o credenciales. Los campos de contraseña «obscured» no son un cifrado seguro. Toda la configuración puede cifrarse con una contraseña; para trabajos desatendidos, un Password Command externo puede obtener el secreto de un Secret Store ([Configuration encryption](https://rclone.org/docs/#configuration-encryption), [rclone config encryption](https://rclone.org/commands/rclone_config_encryption/)).

Rclone puede obtener opciones de backend desde la línea de comandos, variables de entorno específicas del remoto, variables globales del backend y el archivo de configuración. El orden de prioridad debe documentarse para servicios y contenedores, porque una variable heredada puede sobrescribir el `rclone.conf` esperado ([Environment variables](https://rclone.org/docs/#environment-variables)). Los registros de depuración muestran opciones resueltas, pero pueden contener endpoints o rutas sensibles.

Los administradores protegen como mínimo:

- `rclone.conf` y sus permisos de archivo;
- contraseña de configuración o Password Command;
- tokens de actualización OAuth, secretos de cliente, claves de acceso y claves privadas SSH;
- contraseña Crypt y salt opcional;
- permisos de cuenta de servicio, bucket y carpeta;
- definiciones de filtros y trabajos como parte del control de cambios.

Una copia de seguridad de la configuración sin el secreto de descifrado no es una recuperación probada. A la inversa, una configuración cifrada con una contraseña débil no debe almacenarse públicamente.

Una vez aclarados el remoto, el espacio de nombres y las credenciales, el comando elegido determina el efecto sobre los datos. En particular, `sync` no es una copia inocua, sino que impone una parte como la verdad deseada.

## Copy, Sync y Move: la dirección es una política

[`rclone copy`](https://rclone.org/commands/rclone_copy/) copia archivos nuevos o diferentes al destino y no elimina allí objetos adicionales. [`rclone sync`](https://rclone.org/commands/rclone_sync/) convierte el destino en una réplica de la fuente y puede eliminar objetos de destino para ello. [`rclone move`](https://rclone.org/commands/rclone_move/) copia o mueve y elimina la fuente solo después de una operación de archivo correcta.

Estos comandos no son intercambiables:

| Operación | Sobrescritura de destino | Elimina objetos adicionales del destino | Elimina la fuente | Uso típico |
|---|---:|---:|---:|---|
| `copy` | para objetos evaluados como diferentes | no | no | importación, seed, replicación aditiva |
| `sync` | sí | sí | no | réplica unidireccional autoritativa |
| `move` | sí | no | tras una transferencia correcta | ingestión, entrega de archivo |
| `copyto`/`moveto` | objeto individual con nombre de destino explícito | no | solo `moveto` | operación atómica de archivo individual |
| `bisync` | en ambos lados | depende del delta determinado | posible en ambos lados | replicación bidireccional controlada |

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

`--dry-run` evita cambios, pero solo demuestra los listados y opciones visibles en el momento de la prueba. `--check-first` separa las fases de comparación y transferencia. `--backup-dir` mueve objetos de destino sobrescritos o eliminados a una ruta separada, si el backend admite la operación necesaria del lado del servidor ([Global options – backup-dir and retries](https://rclone.org/docs/)). El informe Combined clasifica las rutas iguales, ausentes, diferentes y erróneas.

## ¿Qué significa «idéntico»?

De forma predeterminada, Rclone omite archivos basándose en el tamaño y la hora de modificación, o utiliza un hash común cuando el comando y el backend lo permiten. La huella exacta depende de las capacidades y los flags. `--checksum`, `--size-only`, `--ignore-times`, `--update`, `--ignore-size`, las opciones de metadatos y la precisión de ModTime modifican la decisión ([Overview – Hash and ModTime](https://rclone.org/overview/)).

Un ETag del proveedor no es universalmente un MD5 de contenido. La carga multipart, el cifrado o la implementación del proveedor pueden cambiar su significado. Rclone utiliza la capacidad de hash declarada explícitamente por el backend, no una suposición general sobre ETag.

Entre Linux con distinción de mayúsculas/minúsculas y Windows, normalmente sin esa distinción, `report.pdf` y `Report.pdf` no pueden representarse sin pérdida. Las unidades en la nube pueden permitir nombres duplicados o IDs de objeto propios. Por ello, una comprobación previa de colisiones de nombres y caracteres no permitidos forma parte de toda migración.

## Motor de transferencia, paralelismo y Pacer

El motor de transferencia lista la fuente y el destino, compara entradas, planifica comprobaciones y transferencias, y ejecuta operaciones de backend en paralelo. `--checkers` controla las comprobaciones paralelas, y `--transfers` las transferencias. `--fast-list` puede reducir las llamadas API, pero requiere más memoria porque recopila listados grandes ([Rclone documentation – performance options](https://rclone.org/docs/)).

El paralelismo no es una mera opción de velocidad. Más trabajadores aumentan los archivos abiertos, la memoria, las solicitudes API, los estados multipart y la presión sobre el proveedor. Con objetos pequeños domina la latencia de listado/solicitud; con objetos grandes, el ancho de banda y la configuración multipart. Las copias de nube a nube pueden permanecer del lado del servidor o pasar por el host de rclone; esto se comprueba en el registro y en las capacidades del backend.

Los backends implementan pacing para límites de tasa de API. Los reintentos de bajo nivel gestionan solicitudes individuales; los de alto nivel repiten una operación global fallida. Solo el estado final decide si los errores previos se compensaron correctamente ([Retries](https://rclone.org/docs/#retries)). Un planificador evalúa el código de salida del proceso y no la aparición de la palabra `ERROR` en cualquier parte del registro.

## Los filtros definen el conjunto de datos

Las reglas Include, Exclude, Size, Age y Files-from determinan qué rutas son visibles. Los filtros se refieren a la raíz indicada; un glob de shell en la ruta de origen no sustituye una regla de filtro de rclone. El orden de Include y Exclude puede ser relevante ([Rclone Filtering](https://rclone.org/filtering/)).

Si cambia un filtro, cambia el conjunto funcional de datos. Con `sync`, objetos antes visibles pueden aparecer como eliminados; `--delete-excluded` es especialmente peligroso. Los archivos de filtros se versionan, se hashean y se aprueban junto con el informe de Dry Run y la definición del trabajo. Una prueba segura utiliza `rclone ls` o `lsf` con las mismas reglas antes de ejecutar un comando de escritura.

Un código de salida sin errores de transferencia solo demuestra la ejecución, no la igualdad completa de ambos conjuntos. `check` compara fuentes y destinos con las capacidades del backend correspondiente.

## Verificación: Check en lugar de contadores de transferencia

[`rclone check`](https://rclone.org/commands/rclone_check/) no modifica datos. Compara tamaño y hashes comunes, puede generar informes para coincidencias, diferencias y rutas ausentes, y con `--download` puede leer contenidos cuando no existe un hash remoto común. Para Crypt existe [`rclone cryptcheck`](https://rclone.org/commands/rclone_cryptcheck/), que compara adecuadamente la fuente en texto claro y el destino cifrado.

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

Una comparación de hashes demuestra igualdad para el algoritmo y el espacio de nombres utilizados. No demuestra automáticamente ACL, retención, versiones de objetos del proveedor, metadatos de directorios o legibilidad funcional. Para archivos críticos, una prueba de restauración complementa la comprobación técnica.

## Crypt: cifrado de datos y nombres

Crypt es un backend wrapper sobre otro remoto. El contenido de los archivos se cifra con NaCl SecretBox basado en XSalsa20 y Poly1305; los nombres de archivos y directorios también se transforman de forma predeterminada ([Crypt backend](https://rclone.org/crypt/)). El proveedor ve el texto cifrado, los patrones de tamaño, las horas de acceso y el número de objetos; por tanto, Crypt protege contenido y nombres, no proporciona anonimato completo del tráfico ni de los metadatos.

La contraseña y el salt opcional se almacenan en la configuración solo como «obscured», a menos que toda la configuración esté cifrada. La pérdida de ambas informaciones de recuperación hace que los datos sean ilegibles. Antes de rotar o cambiar el cifrado de nombres, se prueba un segundo remoto de solo lectura contra los mismos datos.

Crypt cambia los tamaños y las posibilidades de hash. Por ello, un `check` normal entre texto claro y texto cifrado no puede esperar simplemente el mismo hash del proveedor. `cryptcheck` o una ruta de descarga controlada es el nivel de comprobación correcto.

Los comandos de copia trabajan explícitamente con objetos. En cambio, un montaje VFS debe reconstruir operaciones continuas de sistema de archivos a partir de ellos y necesita para ello caché, modo de escritura y supuestos claros de consistencia.

## Montaje VFS: el almacenamiento de objetos no es un sistema de archivos POSIX

[`rclone mount`](https://rclone.org/commands/rclone_mount/) presenta un remoto como sistema de archivos mediante FUSE o WinFsp. La capa VFS traduce operaciones de archivos en operaciones de objetos. Como los objetos en la nube normalmente no pueden modificarse en medio ni extenderse arbitrariamente, Rclone almacena en búfer o reemplaza objetos.

Cuatro modos de caché establecen contratos distintos:

| Modo | Caché local de archivos | Comportamiento de escritura | Límite operativo |
|---|---|---|---|
| `off` | ninguna | directo y limitado | no hay Seek/Read-Write como en un archivo normal; es más difícil repetir una carga fallida |
| `minimal` | solo en casos necesarios de Read-Write | parcialmente almacenado en búfer | no toda semántica de Open/Seek es posible |
| `writes` | archivos en escritura locales | carga tras Close/Writeback | la caché local aún no es un commit remoto |
| `full` | Reads y Writes locales, parcialmente sparse | alta compatibilidad de sistema de archivos | supervisar necesidad de disco, Eviction y consistencia de caché |

La caché de directorios mantiene nombres y metadatos en memoria. Los cambios externos solo aparecen mediante Polling o tras expiración o Flush. Buffer es por archivo abierto; la necesidad máxima de memoria crece con `buffer-size × open files`. Dos procesos no deben utilizar la misma caché VFS superpuesta con File Caching activo, porque esto puede dañar datos ([VFS file caching](https://rclone.org/commands/rclone_mount/#vfs-file-caching)).

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

En Windows, [WinFsp](https://winfsp.dev/doc/WinFsp-Tutorial/) proporciona el puente de sistema de archivos; en Linux y otros sistemas Unix, Rclone utiliza FUSE. Un servicio espera la disponibilidad de red, configuración y montaje, dispone de una ruta de desmontaje controlada y supervisa el volumen de caché local. Los montajes en contenedores requieren permisos adicionales de dispositivo y propagación de montaje, y amplían la superficie de privilegios ([Rclone mount in Docker](https://rclone.org/install/#docker)).

## Bisync: dos fuentes de verdad

[`rclone bisync`](https://rclone.org/bisync/) detecta cambios en ambos lados mediante listados almacenados y transfiere deltas bidireccionalmente. Esto crea un estado de trabajo persistente fuera de los datos de usuario. Un directorio de trabajo perdido o inconsistente puede requerir una fase de resync.

Bisync cuenta con comprobaciones de seguridad para la accesibilidad y un número inusualmente alto de eliminaciones. Los cambios de filtros modifican el conjunto de datos visible y fuerzan deliberadamente un resync. Si las aplicaciones cambian el contenido sin modificar tamaño y ModTime, la detección estándar puede fallar. Por ello, la sincronización bidireccional solo se utiliza cuando están documentadas la política de conflictos, el comportamiento de reloj/timestamp, las comprobaciones de acceso, Max-Delete, la copia de seguridad de archivos de trabajo y la resolución humana.

Bisync no es un sistema de archivos multi-master con bloqueo distribuido. Las ediciones simultáneas, los cambios de nombre y los periodos sin conexión siguen siendo conflictos funcionales.

## API Remote Control, métricas y modos Serve

Con `--rc` o [`rclone rcd`](https://rclone.org/commands/rclone_rcd/) Rclone abre una API HTTP. Puede iniciar trabajos, configurar remotos, leer o escribir archivos locales y, en determinadas circunstancias, ejecutar comandos del sistema operativo en el contexto del usuario de rclone. Por ello, la documentación considera que el acceso a RC equivale al acceso de shell ([Remote Control / API](https://rclone.org/rc/)).

RC escucha localmente de forma predeterminada. Si se habilita para la red, se requieren autenticación fuerte, TLS o un reverse proxy protegido, una allowlist de firewall y privilegios de proceso mínimos. `--rc-no-auth` no debe estar en una interfaz accesible. El endpoint OpenMetrics puede exponer métricas de transferencias y trabajos; también allí los nombres de remotos y las rutas pueden ser metadatos sensibles.

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

[`Invoke-WebRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest) y [`curl`](https://curl.se/docs/manpage.html) comprueban el endpoint HTTP local. [`rclone rc`](https://rclone.org/commands/rclone_rc/) utiliza la misma API y adopta parámetros de autenticación de la configuración del trabajo.

Rclone también puede ofrecer remotos mediante modos Serve compatibles con HTTP, WebDAV, SFTP, FTP, NFS, DLNA o S3. Con ello, la función del sistema cambia de cliente a servidor. La dirección de escucha, la autenticación, la seguridad del protocolo, la caché VFS y los derechos sobre los datos se endurecen entonces como en un servicio independiente ([Rclone serve](https://rclone.org/commands/rclone_serve/)).

## Registro, estadísticas y códigos de salida

Los registros distinguen DEBUG, INFO, NOTICE y ERROR. Para procesamiento automático hay disponible registro JSON. Las estadísticas muestran checks, transferencias, errores, bytes, velocidad y ETA. Un contador breve de `Transferred` no es una prueba de auditoría permanente; un trabajo guarda la hora de inicio, la raíz de fuente/destino, el hash de filtro, la compilación de Rclone, las opciones resueltas no sensibles, las estadísticas finales y el código de salida.

Los códigos de salida documentados distinguen éxito, errores de sintaxis, rutas ausentes, errores temporales, menos graves y fatales, así como límites de transferencia o tiempo alcanzados ([List of exit codes](https://rclone.org/docs/#list-of-exit-codes)). `--error-on-no-transfer` puede señalar por separado una ejecución correcta sin transferencia. Los planificadores y la monitorización tratan los códigos según el tipo de trabajo; un trabajo incremental previsiblemente vacío se evalúa de forma distinta que una exportación diaria ausente.

Un reintento de alto nivel puede resolver correctamente mensajes de error tempranos. A la inversa, un proceso puede transferir mucho y terminar con un código distinto de cero debido a pocos objetos sin resolver. Se alerta según el estado final, el número de errores, el agotamiento de reintentos, la duración del trabajo, la ausencia de heartbeat y el resultado de verificación.

## Seguridad de la ruta de datos

Rclone hereda los permisos de su cuenta de proceso y de las credenciales remotas. Un `sync` con permisos de Delete es un agente de eliminación altamente privilegiado. El principio de mínimo privilegio separa fuente de lectura, destino de escritura, derechos de versionado/retención y acciones administrativas de bucket.

La validación de certificados TLS permanece activa. `--no-check-certificate` desactiva la validación de cadena y nombre de host, y según la documentación está destinado solo a pruebas ([TLS options](https://rclone.org/docs/#no-check-certificate)). Los backends SFTP validan las claves de host con una política mantenida de known hosts o huellas digitales; los endpoints API no se redirigen mediante proxies no controlados.

Los filtros, `--backup-dir`, el versionado de objetos y Object Lock limitan riesgos distintos. Un directorio de copia de seguridad en la misma cuenta no protege contra el compromiso de la cuenta ni contra la eliminación por parte del proveedor. [Storage](/kb/storage), [Cifrado](/kb/verschluesselung) y [Hardening](/kb/haertung) tratan las superficies de control subyacentes.

El diagnóstico separa la ruta local, el filtro de Rclone, la API de backend, la autenticación, el límite de tasa y la ruta de red. Solo así se hace visible si un archivo nunca fue seleccionado, no se transfirió o el destino lo representa de otra forma.

## Diagnóstico de red y backend

Un mensaje de error de rclone puede proceder de la resolución de nombres, TCP, TLS, proxy, autenticación del proveedor, límite de tasa de API, traducción de backend, sistema de archivos local o lógica de comparación. El diagnóstico sigue la ruta:

1. Confirmar el binario, la ruta de configuración y el nombre de remoto resuelto.
2. Comprobar DNS, puerto y TLS del endpoint concreto.
3. Utilizar `rclone lsd` o `lsjson` como prueba mínima de backend en modo lectura.
4. Con `-vv`, registrar la fase de solicitud/reintento y las capacidades de backend sin publicar secretos.
5. Comprobar fuente/destino, filtros y decisión de huella.
6. Correlacionar estado final, código de salida y registro de auditoría del proveedor.

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

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname), [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection), [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility), [`nc`](https://man.openbsd.org/nc), [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/), [`rclone lsd`](https://rclone.org/commands/rclone_lsd/) y [`rclone lsjson`](https://rclone.org/commands/rclone_lsjson/) separan transporte, autenticación, listado y visibilidad de objetos. Los volcados de headers se revisan antes de compartirlos para detectar tokens, firmas, cookies y rutas. [DNS](/kb/dns), [TCP](/kb/tcp), [TLS](/kb/tls) y [Troubleshooting](/kb/troubleshooting) profundizan en las fases.

## Patrones de error habituales

Rclone suele informar del paso en el que un trabajo falla visiblemente, no necesariamente de la causa original. Por ello, la tabla vincula el síntoma, el desencadenante probable y la prueba que confirma o refuta la hipótesis.

| Síntoma | Causa probable | Prueba | Solución rápida errónea |
|---|---|---|---|
| Los archivos se copian de nuevo en cada ejecución | ModTime/hash no común o precisión distinta | capacidades de backend, `-vv`, informe Check | establecer a ciegas `--size-only` |
| Los objetos de destino desaparecen | dirección de Sync, filtro u opción Delete | informe Dry Run/Combined, definición del trabajo | simplemente iniciar el trabajo de nuevo |
| El montaje muestra nombres antiguos | Directory Cache o falta de Polling | estadísticas VFS, RC `vfs/forget`, listado del proveedor | finalizar el proceso de forma forzada |
| Falta una escritura del montaje en remoto | archivo abierto, Writeback pendiente, caché llena | caché VFS y registro | eliminar la caché local |
| No es posible comprobar hashes | no hay hash común, Crypt o semántica multipart | vista general de backend, `hashsum`, `check --download` | asumir que ETag es MD5 |
| RC responde externamente | dirección de escucha demasiado amplia/falta de autenticación | lista de sockets, firewall, flags RC | mantener la URL en secreto solamente |
| Bisync se interrumpe tras cambiar filtros | el estado de trabajo protege contra eliminaciones erróneas | registro Bisync y hash de filtro | borrar archivos de trabajo e iniciar normalmente |
| Código de salida distinto de cero pese a muchos archivos transferidos | errores residuales tras reintentos | registro final, informe de errores | considerar el contador de transferencias como éxito |

## Copia de seguridad, recuperación y reconciliación

Rclone forma a menudo parte de una copia de seguridad, pero no es en sí mismo un modelo de copia de seguridad inmutable. `sync` replica eliminaciones y daños por cifrado. `copy` conserva objetos adicionales de destino, pero puede sobrescribir versiones existentes. La protección se logra mediante versionado de destino, retención/Object Lock, credenciales separadas, `--backup-dir`, snapshots inmutables y pruebas de restauración.

El conjunto de recuperación incluye:

- binario de rclone aprobado o artefacto de versión verificable;
- `rclone.conf` cifrado más secreto de descifrado separado;
- credenciales del proveedor y su recuperación;
- contraseña Crypt y salt;
- definiciones de trabajo, filtros y entorno;
- archivos de trabajo de Bisync;
- objetos de caché VFS no cargados y su asignación;
- informes de Dry Run, Combined, Check y código de salida;
- procedimientos de versionado, retención y restauración del destino.

Una restauración se ejecuta en una ruta separada y se comprueba posteriormente con hash, número, tamaño, muestra de metadatos y prueba de aplicación. La reconciliación compara la fuente de verdad, las versiones de destino, el Writeback de VFS y el estado del planificador. [Backup y recuperación ante desastres](/kb/backup-dr) describe RPO/RTO y la aceptación de forma transversal.

## Historia técnica

Nick Craig-Wood inició el proyecto con un primer commit en noviembre de 2012. El fundador documentó este origen en el decimocuarto aniversario y enlazó el estado inicial del código fuente ([Happy 14th Birthday rclone](https://forum.rclone.org/t/happy-14th-birthday-rclone/53005), [initial commit](https://github.com/rclone/rclone/commit/e9ae4f89a44b1cac8e83ede1572f15dace692c67)).

Rclone adoptó la conocida sintaxis de dirección de `rsync`, pero sustituyó sus supuestos de delta por bloques por operaciones de objetos y proveedores. El almacenamiento en la nube normalmente no ofrece una interfaz de sumas de comprobación rodantes rsync del lado del servidor; Rclone compara objetos completos mediante las huellas disponibles y los transfiere en consecuencia.

Con el crecimiento del número de backends se desarrolló el modelo central de interfaces Fs/Object con funciones opcionales. Crypt añadió cifrado del lado del cliente; Mount y VFS hicieron accesible el almacenamiento de objetos para aplicaciones basadas en archivos; la API Remote Control se amplió considerablemente en 2018; Bisync añadió la conciliación bidireccional persistente. El [Changelog](https://rclone.org/changelog/) documenta esta evolución y correcciones relevantes para la seguridad. Por ello, para la operación sigue siendo el complemento dependiente del tiempo de este artículo de arquitectura.

La línea técnica se mantiene coherente: un binario Go portable, muchos backends integrados, comandos comunes y diferencias entre proveedores explícitamente transparentes. Precisamente estas diferencias hacen que Rclone sea potente para administradores y, al mismo tiempo, requiera revisión.

## Lista de comprobación para administradores

Antes de un trabajo productivo, como mínimo se deben documentar los siguientes puntos:

- El origen del binario, la ruta de configuración y la resolución del remoto son inequívocos.
- La fuente, el destino, la dirección y el árbol de destino esperado figuran en el runbook.
- Se han comprobado las capacidades del backend para hash, ModTime, Case, Metadata, Copy/Move y Listing.
- El archivo de filtros y los parámetros del trabajo están versionados; se aprobaron Dry Run e informe Combined.
- Las políticas de Delete, Max-Delete, Backup-Dir, versionado y retención son coherentes entre sí.
- Las credenciales poseen solo los permisos necesarios; la configuración, la contraseña Crypt y el salt son recuperables.
- Transfers, Checkers, Pacer y ancho de banda respetan el host y el proveedor.
- La caché VFS tiene un volumen propio, alerta de capacidad, parada ordenada y comprobación de Writeback.
- RC/Serve escucha solo de forma controlada y está protegido como acceso de shell.
- El éxito se demuestra mediante estado final, código de salida, informe Check y prueba de restauración.

## Fuentes

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
