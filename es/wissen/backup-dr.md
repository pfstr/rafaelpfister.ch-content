---
title: "Copia de seguridad y recuperación ante desastres: estados, objetivos y reanudación"
blatt: "backup-dr"
description: "Copia de seguridad y recuperación ante desastres para administradores de mensajería: diferenciación respecto a instantáneas, replicación y alta disponibilidad; RPO, RTO y MTD; protección coherente de buzones, colas, configuración, claves e índices; recuperación aislada, orden de reanudación, pruebas de restauración y diagnóstico."
fakten:
  - label: Objetivo
    wert: Restaurar servicios y datos a un estado conocido y fiable
    href: https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf
  - label: Base de planificación
    wert: Análisis de impacto en el negocio e inventario de dependencias
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final
  - label: MTD
    wert: interrupción máxima tolerable del proceso de negocio
    href: https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf
  - label: RTO
    wert: indisponibilidad máxima de un recurso antes de un impacto inaceptable
    href: https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf
  - label: RPO
    wert: momento hasta el cual deben restaurarse los datos tras el incidente
    href: https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf
  - label: Copia de seguridad
    wert: copia recuperable con marca temporal; complementa la replicación
    href: https://learn.microsoft.com/en-us/azure/reliability/concept-redundancy-replication-backup
  - label: Coherencia
    wert: la aplicación, el software de copia de seguridad y el almacenamiento deben coordinar el punto de copia
    href: https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service
  - label: Capas de datos
    wert: Buzón/blob · metadatos · cola · índice · configuración · claves
    href: https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf
  - label: Recuperación cibernética
    wert: copia aislada y restauración a un estado fiable
    href: https://cas8.docs.cisecurity.org/en/latest/source/Controls11/
  - label: Inmutabilidad
    wert: la retención impide eliminar o sobrescribir determinadas versiones de objetos
    href: https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html
  - label: Regla de claves
    wert: planificar la recuperación según el tipo de clave y el propósito de uso
    href: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r5.pdf
  - label: Criterio de aceptación
    wert: restauración satisfactoria y oportuna de un servicio empresarial
    href: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-184.pdf
werbung:
  - tools
  - newsletter
ctaThemen:
  - backup
  - disaster-recovery
  - messaging
translationSourceHash: dec2b9d3915258979e801cb9d3ea573f054ae89136cd0631af06b681f36c3ca9
translationModel: gpt-5.6-terra
translatedAt: 2026-10-03T09:38:10.898Z
translationReview: required
---

# Copia de seguridad y recuperación ante desastres: estados, objetivos y reanudación

Un trabajo de copia de seguridad finalizado correctamente solo demuestra inicialmente que una herramienta ha escrito datos. Aún no indica si el punto de copia está completo, es coherente con la aplicación, está protegido frente al mismo fallo y puede restaurarse como servicio utilizable dentro del plazo acordado. **Copia de seguridad** designa la copia recuperable de un estado anterior; la **recuperación ante desastres** incluye además personas, prioridades, infraestructura de destino, dependencias, validación y el retorno controlado a la operación. Por ello, NIST distingue entre copia de seguridad, estrategia de restauración, procedimientos de recuperación, pruebas y mantenimiento continuo del plan ([NIST SP 800-34 Rev. 1](https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final)).

Para las plataformas de mensajería, «la base de datos» no es un objeto de protección completo. Un estado utilizable puede distribuirse entre almacenamiento de buzones o blobs, metadatos relacionales, colas de transporte, índices de búsqueda, configuraciones, reglas de enrutamiento, referencias de directorio, certificados, claves privadas, DNS, licencias y automatización. Algunas partes son autoritativas, otras solo proyecciones y otras estados efímeros. El plan de recuperación debe especificar para cada parte si se **restaura, reconstruye, vuelve a emitir o se descarta deliberadamente**. Además de los datos de usuario, NIST exige estado del sistema, software, inventario, licencias y documentación relevante para la seguridad; PostgreSQL, por ejemplo, señala expresamente que el archivado de WAL no guarda sus archivos de configuración ([NIST SP 800-34 Rev. 1, CP-9](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf), [PostgreSQL: archivado continuo y PITR](https://www.postgresql.org/docs/17/continuous-archiving.html)).

Por tanto, el criterio operativo de aceptación no es «backup montado», sino, por ejemplo: un remitente externo puede entregar un mensaje, se aplica la política correcta, el mensaje aparece en el buzón previsto, se puede buscar y responder, y el monitoreo y la auditoría registran el proceso. NIST menciona restauraciones correctas y puntuales, objetivos de recuperación alcanzados y usuarios o sistemas nuevamente disponibles como resultados medibles de recuperación ([NIST SP 800-184](https://www.nist.gov/publications/guide-cybersecurity-event-recovery), [NIST SP 800-184, métricas de recuperación](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-184.pdf)).

La explicación comienza con el proceso de negocio que debe volver a funcionar tras un fallo. De ello surgen RTO y RPO, después la cadena de copias adecuada; el resultado final no es el trabajo de copia de seguridad, sino una prueba de restauración medida.

## Una copia de seguridad no es alta disponibilidad

La copia de seguridad, la instantánea, la replicación y la alta disponibilidad resuelven clases de error diferentes. Microsoft describe explícitamente la copia de seguridad y la replicación como complementarias: la replicación mantiene una copia actual para la operación continua, pero también reproduce eliminaciones lógicas o daños; una copia de seguridad con marca temporal permite volver a un estado anterior ([Microsoft Azure Reliability: redundancia, replicación y copia de seguridad](https://learn.microsoft.com/en-us/azure/reliability/concept-redundancy-replication-backup)).

| Mecanismo | Beneficio principal | Lo que no demuestra por sí solo |
|---|---|---|
| Copia de seguridad | estado histórico recuperable con retención | tiempo de conmutación corto o plataforma de destino inmediatamente operativa |
| Instantánea de almacenamiento | estado rápido de un volumen en un momento determinado | coherencia de la aplicación, dominio de fallo separado o conservación a largo plazo |
| Replicación | estado actual de los datos en un segundo destino | protección frente a eliminación, cifrado o corrupción silenciosa replicados |
| Alta disponibilidad | continuidad del servicio ante fallos definidos de componentes | vuelta histórica o reconstrucción tras el compromiso de un administrador |
| Archivo | conservación de datos seleccionados durante plazos largos | reconstrucción completa del servicio y sus dependencias |
| Recuperación ante desastres | reanudación coordinada tras eventos de sitio, plataforma o seguridad | recuperación de datos sin copias adecuadas y verificadas |

Una instantánea puede ser un componente de la copia de seguridad. Sin embargo, solo se convierte en una fuente de recuperación fiable mediante coherencia, exportación o replicación a un almacenamiento operado independientemente, retención, catálogo y procedimiento de restauración. La API de instantáneas de Kubernetes, por ejemplo, no garantiza por sí misma la coherencia de la aplicación; las aplicaciones deben prepararse adecuadamente antes de la instantánea. En Windows, VSS asume esta coordinación únicamente cuando requester, writer y provider interactúan correctamente ([Kubernetes: instantáneas de volúmenes y coherencia de aplicaciones](https://kubernetes.io/blog/2020/12/10/kubernetes-1.20-volume-snapshot-moves-to-ga/), [Microsoft: Volume Shadow Copy Service](https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service)).

## MTD, RTO y RPO corresponden a funciones empresariales y del sistema

El **Maximum Tolerable Downtime (MTD)** es la interrupción más larga que el proceso de negocio en conjunto puede tolerar. El **Recovery Time Objective (RTO)** describe cuánto tiempo puede estar indisponible un recurso concreto del sistema antes de afectar de forma inaceptable a otros recursos, al proceso respaldado o a su MTD. El **Recovery Point Objective (RPO)** designa el momento anterior al evento hasta el que deben restaurarse los datos. Por regla general, el RTO debe ser menor que el MTD, ya que tras la reanudación técnica aún deben procesarse datos y verificarse funcionalmente el servicio ([NIST SP 800-34 Rev. 1, sección 3.2](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf)).

Un único «RTO de correo» oculta diferencias relevantes. Una plataforma puede aceptar SMTP aunque el acceso de usuarios o la búsqueda todavía no estén disponibles. Una puerta de enlace puede almacenar mensajes en búfer aunque el servicio de buzones posterior esté caído. A la inversa, una interfaz web puede ser accesible mientras faltan claves, consultas de directorio o conectores salientes. Por ello, los objetivos deben definirse por **función empresarial y dependencia**.

| Función | Estado que se debe medir | Pregunta típica de RPO | El RTO finaliza solo cuando |
|---|---|---|---|
| recepción externa | MX, TLS, listener SMTP, política y cola | ¿Qué mensajes aceptados pueden faltar? | la recepción está controlada y el procesamiento de la cola es verificable |
| entrega saliente | enrutamiento, DNS, política TLS, reintentos y DSN | ¿Qué entradas de cola pueden perderse? | se logra la entrega correcta o una demora conforme a norma |
| acceso al buzón | identidad, metadatos, blob y protocolo | ¿Qué último estado del buzón es necesario? | el inicio de sesión y la lectura, escritura y operaciones de carpeta funcionan |
| búsqueda | índice y proyecciones | ¿Debe protegerse el índice o regenerarse? | el alcance de datos definido vuelve a ser localizable |
| cifrado | política, certificados, claves y confianza | ¿Qué contenidos anteriores deben seguir siendo descifrables? | un mensaje de prueba definido puede cifrarse y descifrarse |
| administración | plano de control, roles, auditoría y monitoreo | ¿Qué cambio de configuración puede faltar? | los cambios autorizados, las alertas y la auditoría son trazables |

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-backup-dr.svg?v=20260813" title="Interaktive Infografik: Backup- und Disaster-Recovery-Kette für Messaging-Plattformen von produktiven Zuständen über konsistente, isolierte Sicherungen bis zur geprüften Wiederherstellung" loading="lazy">
  <a href="/images/kb-interaktiv-backup-dr.svg?v=20260813">Abrir directamente el gráfico interactivo</a>.
</iframe>

Una vez definidos los objetivos, se debe inventariar el estado real de la plataforma. Una base de datos de buzones por sí sola no restaura ni enrutamiento, ni identidades, ni claves, ni índices de búsqueda.

## Inventario de estados de una plataforma de mensajería

Una política de copias de seguridad comienza con un inventario de estados y dependencias, no con el catálogo de productos del fabricante de backup. Para cada estado se documentan la **fuente autoritativa, el mecanismo de coherencia, RPO, retención, dominio de protección, método de restauración, orden y paso de verificación**. El análisis de impacto en el negocio de NIST identifica procesos críticos, recursos y su prioridad de recuperación; NIST SP 800-184 añade escenarios realistas y dependencias descubiertas durante la recuperación ([NIST SP 800-34 Rev. 1](https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final), [NIST SP 800-184](https://www.nist.gov/publications/guide-cybersecurity-event-recovery)).

| Estado | Característica | Decisión de recuperación | Paso de verificación funcional |
|---|---|---|---|
| Buzones y blobs de mensajes | datos de usuario autoritativos | restaurar de forma coherente o reconstruir desde una fuente inmutable | leer un mensaje conocido junto con sus adjuntos MIME |
| Base de datos de metadatos | transacciones, asignaciones, ACL, UID | utilizar backup base más reproducción de logs o restauración específica de la aplicación | coinciden carpetas, permisos y referencias de mensajes |
| Cola de transporte | estado de entrega efímero, pero relevante para el negocio | proteger, asumir ordenadamente o volver a entregar deliberadamente | no hay laguna silenciosa y se gestionan los duplicados de forma controlada |
| Índice de búsqueda y proyecciones | normalmente derivados y posiblemente coherentes | proteger solo si la reconstrucción incumple el RTO; de lo contrario, reindexar | una muestra definida se puede localizar íntegramente |
| Configuración y política | declarativa, exportada o respaldada por base de datos | proteger exportación versionada más estado de esquema/producto | se aplican enrutamiento, filtros, límites y separación de inquilinos |
| Identidades y referencias de directorio | a menudo autoritativas externamente | restaurar el directorio por separado; conservar bindings, ID y claims | funcionan el inicio de sesión de servicio y de usuario |
| Certificados, claves y secretos | muy sensibles, parcialmente no exportables | proteger, volver a emitir o reconstruir mediante HSM/KMS según tipo de clave | se verifican TLS, firma, descifrado y rotación |
| DNS, tiempo, red y balanceador de carga | plano externo de control y nombres | mantener como código/exportación y documentar con el proveedor | coinciden nombres, puertos, nombres de certificados y tiempo |
| Software, imágenes, IaC y licencias | base de ejecución reproducible | mantener artefactos, versiones y dependencias fiables | se inicia una compilación idéntica o compatible aprobada |
| Logs, auditoría, catálogo de copias y runbooks | evidencia y control | mantener disponibles fuera del dominio administrativo afectado | el incidente, el punto de restauración y las aprobaciones son trazables |

La recuperación de colas es un caso especial. SMTP exige ejecutar de forma fiable la responsabilidad aceptada, pero en caso de interrupciones de conexión permite situaciones en las que emisor y receptor evalúan de forma diferente la finalización. Por ello, un estado de cola restaurado puede volver a entregar mensajes. Los runbooks requieren una estrategia definida para duplicados, ID de cola, ventanas temporales y comunicación con destinatarios; copiar simplemente un directorio de spool no es una restauración conforme a norma ([RFC 5321, colas y mensajes duplicados](https://datatracker.ietf.org/doc/html/rfc5321)).

## La coherencia se crea en la capa de aplicación

Un punto de copia **coherente ante fallos** contiene el estado que un sistema vería tras una pérdida repentina de alimentación. El sistema de archivos y los bloques individuales pueden ser coherentes internamente mientras que bases de datos, blobs y colas relacionados representan momentos diferentes. Un punto de copia **coherente con la aplicación** coordina búferes de escritura, registros de transacciones, puntos de comprobación y, cuando corresponde, varios volúmenes, de modo que la aplicación tenga una ruta de recuperación definida.

VSS muestra explícitamente esta arquitectura: el requester de backup solicita la copia, el writer específico de la aplicación prepara un conjunto de datos coherente y el provider crea la copia de sombra. Exchange proporciona para ello su propio VSS Writer; por eso, un backup compatible con Exchange es más que una instantánea de sus archivos de base de datos ([Microsoft: Volume Shadow Copy Service](https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service), [Microsoft: Windows Server Backup para Exchange](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/windows-server-backup)).

PostgreSQL utiliza un modelo de recuperación diferente pero comparable. Un backup base proporciona la base inicial y una secuencia ininterrumpida de segmentos archivados de Write-Ahead Log la prolonga hasta el momento deseado. Un `pg_dump` es una exportación lógica y no sustituye la cadena de backup base/WAL necesaria para PITR. Los archivos de configuración como `postgresql.conf` y `pg_hba.conf` también quedan fuera de esta recuperación WAL y requieren una ruta de copia separada ([PostgreSQL: archivado continuo y PITR](https://www.postgresql.org/docs/17/continuous-archiving.html)).

Para productos distribuidos, la documentación del producto debe indicar si los backends se protegen de forma independiente, como grupo de coherencia o mediante funciones de exportación propias de la aplicación. Una instantánea de almacenamiento simultánea de varios volúmenes no constituye automáticamente un corte coherente de base de datos, almacenamiento de objetos, cola e índice de búsqueda. El administrador debe conocer la **fuente de verdad** y la ruta de reconstrucción permitida para cada proyección.

### Inventariar la capacidad y los artefactos de copia de seguridad

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kapazitäts- und Sicherungsinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-Volume | Sort-Object DriveLetter |
  Select-Object DriveLetter, FileSystemLabel, Size, SizeRemaining
Get-ChildItem \\backup.example.ch\mail -File -Recurse |
  Select-Object FullName, Length, LastWriteTime
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
df -hT
find /backup/mail -type f -printf '%TY-%Tm-%TdT%TH:%TM:%TS %s %p\n'
```

  </div>
</div>

[`Get-Volume`](https://learn.microsoft.com/powershell/module/storage/get-volume) y [`df`](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html) muestran la capacidad utilizada y libre. [`Get-ChildItem`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-childitem) y [`find`](https://www.gnu.org/software/findutils/find) inventarían artefactos y marcas temporales. Ninguno de los dos demuestra coherencia de la aplicación ni recuperabilidad; para ello se requieren evidencias de catálogo, logs y restauración.

## Arquitectura de protección: separada, aislada y verificable

Una cadena de copia de seguridad fiable tiene al menos cuatro roles diferenciables:

1. **Captura:** la aplicación o función de exportación genera un estado definido.
2. **Catálogo y manifiesto:** ID de backup, fuente, momento, versión de software, logs necesarios, referencias de claves y sumas de verificación hacen que el conjunto sea localizable y verificable.
3. **Almacenamiento de recuperación:** las copias versionadas se encuentran fuera del dominio de fallo primario y, en lo posible, también del dominio administrativo.
4. **Plano de control de recuperación:** identidades separadas, runbooks, infraestructura de destino y aprobaciones permiten la restauración cuando la producción no es fiable.

CIS Control 11 exige datos de recuperación protegidos de forma equivalente, una instancia aislada, por ejemplo sin conexión, en la nube o fuera de la ubicación, así como pruebas de restauración periódicas. CISA recomienda para escenarios de ransomware copias de seguridad sin conexión o aisladas de otro modo, cifradas y probadas regularmente, además de imágenes limpias y un entorno de recuperación separado ([CIS Control 11: recuperación de datos](https://cas8.docs.cisecurity.org/en/latest/source/Controls11/), [CISA: guía StopRansomware](https://www.cisa.gov/stopransomware/ransomware-guide)).

La **inmutabilidad** y el **aislamiento** no son lo mismo. S3 Object Lock puede proteger determinadas versiones de objetos en el modelo WORM frente a eliminación y sobrescritura durante una retención o una retención legal. Los modos de gobernanza y cumplimiento ofrecen posibilidades de omisión diferentes. Esto protege las versiones almacenadas, pero no demuestra ni credenciales separadas ni un extremo de restauración limpio, una cadena de aplicación completa o claves de descifrado accesibles ([Amazon S3: Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)).

### Verificar sumas de comprobación y manifiesto

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Prüfsummen von Sicherungsartefakten">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-FileHash .\mail-backup-2026-08-08.tar.zst -Algorithm SHA256
Get-FileHash .\mail-backup-2026-08-08.manifest.json -Algorithm SHA256
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
sha256sum mail-backup-2026-08-08.tar.zst
sha256sum mail-backup-2026-08-08.manifest.json
```

  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) y [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) detectan cambios en un artefacto si el hash esperado procede de una fuente fiable. Una suma de comprobación no sustituye la autenticación del manifiesto ni una prueba de restauración. NIST menciona hashes criptográficos y firmas digitales como mecanismos para proteger la integridad de la información de backup ([NIST SP 800-34 Rev. 1, CP-9](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf)).

## Las claves y los secretos requieren un plan de recuperación propio

«Hacer copia de todas las claves privadas» es tan erróneo como «los certificados se pueden volver a emitir». Lo decisivo es el propósito de uso:

- Una **clave de servidor TLS** perdida normalmente puede sustituirse mediante un nuevo par de claves y certificado; no obstante, la conmutación debe ajustarse al RTO y deben comprobarse las dependencias asociadas de confianza o pinning.
- Una **clave de descifrado** para datos S/MIME, OpenPGP o backups almacenados debe seguir disponible mientras el texto cifrado protegido deba poder leerse.
- Según NIST, la copia de seguridad de una **clave privada de firma** generalmente no es deseable, ya que su reutilización puede afectar el valor probatorio de la firma; las excepciones justificadas requieren una recuperación especialmente segura y una sustitución rápida.
- Una **clave HSM/KMS no exportable** necesita la vía de redundancia, backup o reprovisionamiento prevista por el sistema. Exportar un certificado sin clave privada no constituye una copia de seguridad de clave.
- La **clave que cifra el backup** no debe encontrarse exclusivamente dentro del backup cifrado ni del dominio de producción comprometido.

NIST exige una decisión según el tipo de clave, metadatos asociados, una política de recuperación de claves y controles de confidencialidad, integridad, disponibilidad y auditoría para el material de recuperación. Si se pierde una clave de descifrado, el texto cifrado ya no se puede convertir nuevamente en texto claro ([NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final), [NIST SP 800-57 Part 1 Rev. 5, recuperación de claves](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r5.pdf)).

### Comprobar el tiempo y la resolución de nombres antes de la restauración

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Zeit- und DNS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
w32tm /query /status
Resolve-DnsName -Type MX example.ch
Resolve-DnsName backup.example.ch
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
timedatectl status
dig +short MX example.ch
dig +short A backup.example.ch
```

  </div>
</div>

[`w32tm`](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings) y [`timedatectl`](https://www.freedesktop.org/software/systemd/man/latest/timedatectl.html) comprueban la base temporal para certificados, Kerberos, logs y puntos de recuperación. [`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) y [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) muestran si los nombres MX, de servicio y de repositorio se resuelven según lo previsto en la zona de recuperación. La fuente de [DNS](/kb/dns) y su autorización de modificación deben figurar por sí mismas en el inventario de dependencias.

Un backup coherente es solo la mitad del plan. Durante la restauración, la identidad, DNS, la base de datos, la cola, las claves y las aplicaciones deben volver en un orden justificado.

## La reanudación sigue el grafo de dependencias

Un orden fijo de productos sería inventado. El orden fiable resulta del BIA, el inventario de recursos y las dependencias reales. NIST exige una lista priorizada de recursos del sistema y escenarios de prueba realistas; NIST SP 800-184 requiere incorporar a la documentación las dependencias recién identificadas durante la restauración ([NIST SP 800-34 Rev. 1, prioridades de recuperación](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf), [NIST SP 800-184, ejecución de recuperación](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-184.pdf)). Para una plataforma de mensajería típica, esto suele dar lugar a la siguiente cadena, que debe validarse localmente:

1. **Delimitar el incidente:** distinguir entre fallo o compromiso, preservar evidencias, determinar un punto de recuperación limpio conocido y la aprobación.
2. **Establecer el plano de control de recuperación:** proporcionar identidades de administrador separadas, MFA, runbooks, catálogo de copias y acceso al descifrado.
3. **Validar servicios básicos:** comprobar red, enrutamiento, [DNS](/kb/dns), tiempo, [LDAP](/kb/ldap) o [Kerberos](/kb/kerberos), PKI/KMS y balanceador de carga en la zona de destino.
4. **Restaurar la persistencia:** reconstruir almacenamiento de objetos/buzones, bases de datos y los logs de transacciones necesarios en un corte coherente.
5. **Iniciar la aplicación y la política:** aplicar imágenes aprobadas, configuración, secretos, conectores y roles; sin permitir todavía un flujo externo de correo no controlado.
6. **Activar la cola y el enrutamiento de forma controlada:** evaluar antigüedad, destinatarios, estado de reintentos y posibles duplicados; liberar por separado entrada y salida.
7. **Regenerar proyecciones:** crear índices de búsqueda, cachés e informes a partir de fuentes autoritativas y supervisar el retraso de reconstrucción.
8. **Aceptar la transacción de negocio:** probar entrega, acceso al buzón, búsqueda, [TLS](/kb/tls), cifrado, monitoreo y auditoría frente a criterios definidos.

Para un incidente de seguridad, «el sistema se inicia» expresamente no basta. NIST describe la reconstitución a un estado seguro conocido con parámetros seguros, parches, configuración, software fiable, backup limpio conocido y prueba completa. CIS formula el mismo objetivo como restauración a un «pre-incident and trusted state» ([NIST SP 800-34 Rev. 1, CP-10](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf), [CIS Control 11: recuperación de datos](https://cas8.docs.cisecurity.org/en/latest/source/Controls11/)).

### Alcanzar los extremos de recuperación y TLS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Recovery-Endpunkt- und TLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection backup.example.ch -Port 443 -InformationLevel Detailed
curl.exe --verbose https://backup.example.ch/health
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz backup.example.ch 443
openssl s_client -connect backup.example.ch:443 \
  -servername backup.example.ch -verify_return_error
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) y [`nc`](https://man.openbsd.org/nc) comprueban la ruta TCP. [`curl`](https://curl.se/docs/manpage.html) y [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) muestran el comportamiento de HTTP y TLS, respectivamente. Un extremo de estado accesible solo demuestra el plano de control, no la legibilidad de todos los conjuntos de backup.

La reanudación solo finaliza cuando un usuario o una contraparte puede utilizar realmente el servicio. Un medio de backup leído correctamente no es una prueba suficiente.

## Las pruebas de restauración miden el servicio, no el medio

Una prueba completa se realiza en un entorno de destino aislado con punto de partida documentado, medición de tiempo y criterios de aceptación. Comprueba al menos:

- que el catálogo, las credenciales, las claves de descifrado y los artefactos son accesibles sin producción;
- que se puede proporcionar un sistema de destino compatible a partir de imágenes fiables;
- que el backup base, los logs de transacciones, el almacenamiento de blobs y la configuración producen el mismo estado funcional;
- que las colas se procesan de forma controlada y se detectan duplicados;
- que funcionan identidad, DNS, TLS, flujo de correo, acceso al buzón, búsqueda y monitoreo;
- que la pérdida de datos medida y el tiempo de reanudación medido cumplen RPO y RTO;
- que el servicio puede aprobarse como fiable tras incidentes de seguridad.

CIS Control 11.5 evalúa una muestra de backups restaurados y que después funcionan realmente. NIST SP 800-184 mide restauraciones correctas y puntuales y exige escenarios realistas, revisión posterior y mejora del plan ([CIS Control 11: prueba de recuperación de datos](https://cas8.docs.cisecurity.org/en/latest/source/Controls11/), [NIST SP 800-184](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-184.pdf)). La frecuencia de las pruebas depende del riesgo, la tasa de cambios y los requisitos; una prueba completa anual puede complementarse con muestras automatizadas más frecuentes y restauraciones por componentes, pero no sustituirse por estadísticas exitosas de trabajos.

### Listeners y estado de almacenamiento tras la restauración

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Listener- und Speicherprüfung nach Restore">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-NetTCPConnection -State Listen |
  Sort-Object LocalPort |
  Select-Object LocalAddress, LocalPort, OwningProcess
Get-Volume | Select-Object DriveLetter, FileSystemLabel, SizeRemaining
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
ss -lntup
df -hT
```

  </div>
</div>

[`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) y [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) muestran los listeners locales y los procesos asociados. [`Get-Volume`](https://learn.microsoft.com/powershell/module/storage/get-volume) y [`df`](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html) muestran el espacio libre. A continuación, la comprobación debe continuar en el nivel de protocolo: un listener en el puerto 25 todavía no es una transacción [SMTP](/kb/smtp) funcional.

## Patrones de fallo y alcance de recuperación adecuado

Un fallo determina hasta dónde debe llegar una restauración. Por ello, la tabla vincula el evento observado con el alcance mínimo razonable de recuperación y el supuesto erróneo que se produce con especial frecuencia.

| Evento | Riesgo principal | Alcance de recuperación adecuado | Supuesto erróneo frecuente |
|---|---|---|---|
| eliminación accidental | daño lógico menor | restaurar selectivamente objeto, buzón, política o punto en el tiempo | revertir toda la plataforma y perder datos correctos más recientes |
| nodo o soporte de datos individual | fallo de infraestructura local | conmutación por error de HA, réplica o restauración por componente | confundir la conmutación por error con un backup histórico |
| pérdida de ubicación o proveedor | dominio de fallo físico o administrativo compartido | zona/región/sitio alternativo más copias externas y conmutación DNS/red | considerar DR una copia de datos sin capacidad de destino accesible |
| ransomware o compromiso de administrador | datos, identidades, software y backups no son fiables | plano de control de recuperación aislado, compilación limpia, punto de restauración conocido | seguir usando la identidad comprometida para desbloquear todos los backups |
| pérdida de clave | texto cifrado permanentemente ilegible o identidad inutilizable | recuperación específica por tipo de clave, reemisión o procedimiento HSM/KMS | confundir certificado público con clave privada |
| cambio de configuración defectuoso | datos correctos, comportamiento incorrecto | revertir configuración versionada y validar selectivamente | elegir la restauración de base de datos o buzón como primera medida |

En caso de compromiso, el momento limpio conocido no tiene por qué ser el punto de copia más reciente. Los backups más nuevos pueden contener el estado del atacante; los más antiguos pueden traer vulnerabilidades conocidas o versiones de software incompatibles. Por tanto, la recuperación combina análisis forense, estado de parches, línea base de configuración, rotación de claves y restauración funcional de datos. CISA recomienda, entre otras medidas, «golden images» limpios, definiciones de infraestructura mantenidas sin conexión y una zona de red de recuperación para que los sistemas no vuelvan a infectarse durante la reconstrucción ([CISA: guía StopRansomware](https://www.cisa.gov/stopransomware/ransomware-guide)).

## Evolución técnica

La cinta magnética se introdujo a principios de la década de 1950 como medio rápido de almacenamiento de datos para ordenadores y sigue siendo un medio de backup debido a su coste, capacidad y separabilidad física ([IBM: cinta magnética](https://www.ibm.com/history/magnetic-tape)). Las arquitecturas de backup posteriores separaron cada vez más el punto lógico de copia del medio de destino: las bases de datos combinaron backups base con logs de transacciones y recuperación a un punto en el tiempo; los sistemas de almacenamiento permitieron instantáneas rápidas; la deduplicación y el almacenamiento de objetos cambiaron la transferencia y la retención.

Para las aplicaciones Windows en ejecución, Microsoft introdujo con VSS un modelo coordinado de requester, writer y provider; la tecnología apareció con Windows XP y Windows Server 2003, respectivamente. En plataformas distribuidas y en contenedores se estandarizaron APIs de instantáneas y orquestación, sin resolver automáticamente por ello la coherencia de las aplicaciones ([Microsoft: Volume Shadow Copy Service](https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service), [Kubernetes: instantáneas de volúmenes](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)).

La recuperación cibernética volvió a desplazar el foco. El versionado y las copias externas no bastan si identidades con privilegios elevados pueden eliminar todos los destinos o si vuelven imágenes comprometidas. Instancias de recuperación aisladas, identidades separadas, versiones inmutables de objetos, infraestructura declarativa y zonas limpias de reanudación complementan los backups clásicos completos, incrementales y basados en logs. Amazon S3 Object Lock se introdujo en 2018 como protección WORM para versiones de objetos; esta función ilustra la transición, pero sigue sin sustituir ni la coherencia de aplicaciones ni las pruebas de restauración ([AWS: introducción de S3 Object Lock](https://aws.amazon.com/about-aws/whats-new/2018/11/s3-object-lock/), [Amazon S3: Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)).

## Lista de verificación para administradores

La planificación solo es fiable cuando los objetivos, las copias, los accesos y las pruebas se documentan conjuntamente. La lista de verificación resume estas dependencias para la revisión y los ejercicios de restauración.

- [ ] Los procesos de negocio, MTD, RTO y RPO por función del sistema están aprobados.
- [ ] Se han inventariado todos los estados autoritativos y derivados de la plataforma de mensajería.
- [ ] La coherencia de la aplicación, la cadena de logs y los grupos de coherencia están documentados de forma específica para el producto.
- [ ] Están regulados la restauración de colas, los posibles duplicados y la liberación de entrada y salida.
- [ ] La configuración, políticas, DNS, certificados, claves, secretos, licencias y runbooks están dentro del alcance.
- [ ] Al menos una copia de recuperación está separada de producción y de las cuentas administrativas principales.
- [ ] Se han evaluado por separado inmutabilidad, aislamiento, cifrado y acceso a claves.
- [ ] El plano de control de recuperación y la capacidad de destino funcionan sin la producción comprometida.
- [ ] El orden de reanudación sigue un grafo de dependencias mantenido.
- [ ] Las pruebas restauran una transacción empresarial completa de correo y miden RPO/RTO.
- [ ] Los resultados, las nuevas dependencias y las desviaciones se incorporan al runbook y a la arquitectura.

## Fuentes

- [NIST – SP 800-34 Rev. 1, guía de planificación de contingencias](https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final)
- [NIST – SP 800-34 Rev. 1, PDF](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf)
- [PostgreSQL – archivado continuo y recuperación a un punto en el tiempo](https://www.postgresql.org/docs/17/continuous-archiving.html)
- [NIST – SP 800-184, guía para la recuperación de eventos de ciberseguridad](https://www.nist.gov/publications/guide-cybersecurity-event-recovery)
- [NIST – SP 800-184, PDF](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-184.pdf)
- [Microsoft Azure Reliability – redundancia, replicación y copia de seguridad](https://learn.microsoft.com/en-us/azure/reliability/concept-redundancy-replication-backup)
- [Kubernetes – instantáneas de volúmenes y coherencia de aplicaciones](https://kubernetes.io/blog/2020/12/10/kubernetes-1.20-volume-snapshot-moves-to-ga/)
- [Microsoft Learn – Volume Shadow Copy Service](https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service)
- [IETF RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [Microsoft Learn – Windows Server Backup para Exchange](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/windows-server-backup)
- [Microsoft Learn – Get-Volume](https://learn.microsoft.com/powershell/module/storage/get-volume)
- [GNU Coreutils – df](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html)
- [Microsoft Learn – Get-ChildItem](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-childitem)
- [GNU Findutils – find](https://www.gnu.org/software/findutils/find)
- [CIS – Control 11: recuperación de datos](https://cas8.docs.cisecurity.org/en/latest/source/Controls11/)
- [CISA – guía StopRansomware](https://www.cisa.gov/stopransomware/ransomware-guide)
- [Amazon S3 – Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)
- [Microsoft Learn – Get-FileHash](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – sha256sum](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [NIST – SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)
- [NIST – SP 800-57 Part 1 Rev. 5, PDF](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r5.pdf)
- [Microsoft Learn – w32tm](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings)
- [systemd – timedatectl](https://www.freedesktop.org/software/systemd/man/latest/timedatectl.html)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [ISC BIND 9 – página de manual de dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – página de manual de nc](https://man.openbsd.org/nc)
- [curl – página de manual de línea de comandos](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection)
- [Linux man-pages – ss](https://man7.org/linux/man-pages/man8/ss.8.html)
- [IBM – cinta magnética](https://www.ibm.com/history/magnetic-tape)
- [Kubernetes – instantáneas de volúmenes](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
- [AWS – introducción de S3 Object Lock](https://aws.amazon.com/about-aws/whats-new/2018/11/s3-object-lock/)
