---
title: "Paperless-ngx: canalización de documentos, OCR y operación del archivo"
blatt: "paperless-ngx"
description: "Paperless-ngx para administradores de DMS, plataformas y mensajería: arquitectura Django/Angular, canalización de Consumer y OCR, archivos originales y de archivo, metadatos, búsqueda Tantivy, Celery y Redis, PostgreSQL, Tika/Gotenberg, permisos, recuperación de correo, API, operación, copia de seguridad, restauración e historia técnica."
fakten:
  - label: Función del sistema
    wert: sistema de gestión documental autoalojado para captura, OCR, clasificación, búsqueda, conservación y exportación
    href: https://docs.paperless-ngx.com/
  - label: Pila tecnológica
    wert: Python y Django en el backend, Angular en el navegador, tareas Celery, OCRmyPDF/Tesseract e índice de búsqueda Tantivy
    href: https://github.com/paperless-ngx/paperless-ngx
  - label: Servicios obligatorios
    wert: procesos web/de tareas, bróker compatible con Redis, base de datos y directorios persistentes Data/Media
    href: https://docs.paperless-ngx.com/configuration/
  - label: Base de datos
    wert: PostgreSQL recomendado para instalaciones nuevas; también se admiten MariaDB y SQLite
    href: https://docs.paperless-ngx.com/setup/
  - label: Objeto documental
    wert: registro de base de datos más original, PDF de archivo opcional, miniatura, índice de texto completo y sumas de comprobación
    href: https://docs.paperless-ngx.com/faq/
  - label: Entradas
    wert: directorio Consume, carga desde navegador/API y recuperación de correo IMAP basada en reglas
    href: https://docs.paperless-ngx.com/usage/
  - label: OCR
    wert: OCRmyPDF orquesta la rasterización y Tesseract; el modo, idioma, límite de páginas y formato de archivo son parámetros operativos
    href: https://docs.paperless-ngx.com/configuration/
  - label: Office y correo electrónico
    wert: conversión opcional mediante Apache Tika y Gotenberg; ambos servicios constituyen un ámbito de fallo propio
    href: https://docs.paperless-ngx.com/configuration/#optional-services
  - label: Búsqueda
    wert: índice Tantivy junto a la base de datos relacional; el índice es derivable, la base de datos y los originales son la fuente principal
    href: https://docs.paperless-ngx.com/migration-v3/
  - label: Permisos
    wert: permisos globales de Django más permisos de Owner, View y Change por objeto para usuarios y grupos
    href: https://docs.paperless-ngx.com/usage/#permissions
  - label: Núcleo de la copia de seguridad
    wert: exportación coherente o copia coordinada de base de datos, Media, Data, configuración y secretos
    href: https://docs.paperless-ngx.com/administration/#making-backups
  - label: Regla de aceptación
    wert: restauración en una instancia vacía y compatible; deben funcionar el inicio de sesión, el número de documentos, las muestras, la búsqueda, las sumas de comprobación y una nueva ejecución de Consume
    href: https://docs.paperless-ngx.com/administration/#restoring
werbung:
  - newsletter
ctaThemen:
  - paperless-ngx
translationSourceHash: a172bf8c7ed80cecd7c990d657cfdf7d06afe1c0b349350655549c21ed144dfc
translationModel: gpt-5.6-terra
translatedAt: 2026-10-10T10:42:40.541Z
translationReview: automatic
---

# Paperless-ngx: canalización de documentos, OCR y operación del archivo

Paperless-ngx es un sistema de gestión documental autoalojado para archivos entrantes y documentos en papel escaneados. Recibe documentos desde un directorio supervisado, desde la web o la API REST, o desde buzones de correo electrónico; genera texto consultable y, si es necesario, una copia de archivo normalizada, asigna metadatos y permite buscar el fondo documental mediante la interfaz web y la API. La [visión general del proyecto](https://docs.paperless-ngx.com/) describe el servicio como un archivo en línea consultable; el [repositorio fuente](https://github.com/paperless-ngx/paperless-ngx) también lo identifica como el sucesor comunitario de Paperless y Paperless-ng.

Para los administradores, Paperless-ngx no es una única aplicación web, sino un conjunto de base de datos, bróker, procesos web y en segundo plano, archivos, índice de búsqueda y convertidores externos. Un estado HTTP correcto solo demuestra que el proceso web responde. No indica si el Consumer detecta archivos nuevos, si se entregan tareas de Celery, si funciona el OCR, si se escriben archivos de archivo, si el índice está actualizado o si se puede volver a importar una exportación.

La explicación acompaña un documento desde su entrada, pasando por Consumer, parser y OCR, hasta la base de datos, el archivo de archivo y el índice de búsqueda. Después se abordan la automatización, la seguridad, la operación y una restauración que comprueba conjuntamente todas las partes.

Paperless-ngx procesa un documento como una canalización: detectar la entrada, adoptar el archivo, generar OCR y metadatos, actualizar la base de datos y el índice de búsqueda, y proporcionar el resultado a través de la web o la API. El artículo sigue este recorrido y deriva de él la operación, la copia de seguridad y el diagnóstico.

## Arquitectura: ruta de control, ruta de trabajo y ruta de datos

La arquitectura puede dividirse en tres rutas:

1. **Ruta de control:** El navegador o cliente de API se comunica mediante HTTP o HTTPS con la aplicación Django. El frontend Angular es el cliente interactivo; Django proporciona autenticación, lógica de negocio, interfaz de administración y API REST. La [documentación de desarrollo](https://docs.paperless-ngx.com/development/) documenta que el proyecto consta de una aplicación Django y un frontend Angular.
2. **Ruta de trabajo:** Consumer, Scheduler y Worker generan y procesan tareas asíncronas. Un bróker compatible con Redis media entre procesos; Celery ejecuta, entre otras cosas, el consumo, la recuperación de correo, el mantenimiento del índice y tareas de mantenimiento recurrentes. La [documentación de uso](https://docs.paperless-ngx.com/usage/) describe Consumer, Task Processor y Broker como roles separados.
3. **Ruta de datos:** La base de datos relacional contiene objetos documentales, metadatos, usuarios, reglas y estados de tareas. Los originales, versiones de archivo y miniaturas se encuentran en el directorio Media; los estados auxiliares y los datos de búsqueda se encuentran en el directorio Data. El índice de búsqueda es una proyección de este fondo principal.

La separación es decisiva desde el punto de vista operativo. La base de datos y el directorio Media deben pertenecer al mismo momento lógico. Puede reconstruirse un índice nuevo, pero no puede reconstruirse un archivo original perdido a partir del índice. Del mismo modo, una copia de seguridad de la base de datos sin los archivos correspondientes no basta para restaurar un DMS.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-paperless-ngx.svg?v=20260813" title="Interaktive Infografik: Paperless-ngx von Eingang, Consumer und Task-Broker über Parser, OCR, Archivierung, Metadaten und Suche bis zu Datenbank, Dateien, Backup und Restore" loading="lazy">
  <a href="/images/kb-interaktiv-paperless-ngx.svg?v=20260813">Abrir directamente el gráfico interactivo</a>.
</iframe>

## Pila tecnológica y roles de proceso

El código de backend está implementado principalmente en Python y se basa en Django. El frontend del navegador utiliza Angular. Dependencias como Django, [Celery](https://docs.celeryq.dev/en/stable/), OCRmyPDF y otros parsers se pueden consultar en el [manifiesto del proyecto Python](https://github.com/paperless-ngx/paperless-ngx/blob/dev/pyproject.toml); la creación de la imagen y las dependencias nativas de OCR/PDF se muestran en el [Dockerfile](https://github.com/paperless-ngx/paperless-ngx/blob/dev/Dockerfile). Para la operación son relevantes los siguientes roles:

| Rol | Tarea principal | Fuente de datos determinante | Incidencia típica |
|---|---|---|---|
| Servidor web | UI, API REST, autenticación, carga | Sesión y transacción de BD | 4xx/5xx, errores de proxy o CSRF |
| Consumer | detectar archivos nuevos e iniciar la tarea | Directorio Consume y tarea | El archivo queda pendiente, polling/inotify o permisos |
| Task Worker | parser, OCR, clasificación y pasos de archivo | Estado de tarea, archivos temporales, BD | Tarea pending/failed, límite de memoria/CPU |
| Scheduler | recuperación de correo y mantenimiento periódico | Planificación en el sistema de tareas | No hay trabajos periódicos |
| Broker | entrega de tareas asíncronas | Datos efímeros de cola/bróker | La web funciona, el procesamiento se detiene |
| Base de datos | documentos, metadatos, identidades, reglas | Fondo principal relacional | Errores de transacción, esquema o capacidad |
| Almacenamiento de archivos | original, archivo, miniatura | Archivos Media | Falta un archivo, solo lectura, montaje incoherente |
| Índice de búsqueda | proyección de texto completo | Datos de índice derivables | El documento existe, búsqueda incompleta |
| Tika/Gotenberg | conversión opcional de Office/correo | Convertido temporal | Solo fallan determinados tipos de archivo |

La [referencia de configuración](https://docs.paperless-ngx.com/configuration/) oficial designa el [bróker compatible con Redis](https://redis.io/docs/latest/) como servicio obligatorio. Se admiten [PostgreSQL](https://www.postgresql.org/docs/current/), MariaDB y SQLite como bases de datos; para instalaciones nuevas, la [guía de instalación](https://docs.paperless-ngx.com/setup/) recomienda PostgreSQL. Tika y Gotenberg son opcionales y se necesitan conjuntamente para formatos de Office y el procesamiento de archivos de correo ([servicios opcionales](https://docs.paperless-ngx.com/configuration/#optional-services)).

## Inventario de servicios y configuración efectiva

Un inventario sólido incluye imagen o versión, tipo de base de datos, destino del bróker, montajes, variables de entorno efectivas, UID/GID, proxy inverso, zona horaria, idiomas de OCR y parsers externos. Docker Compose es una forma de empaquetado habitual, pero no prueba la arquitectura: varios roles de proceso pueden estar en una imagen de contenedor, mientras que sus estados residen en volúmenes y servicios separados.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Paperless-Dienstinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker compose config --services
docker compose ps
docker compose images
docker compose config
docker inspect paperless-webserver</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker compose config --services
docker compose ps
docker compose images
docker compose config
docker inspect paperless-webserver</code></pre>
  </div>
</div>

[`docker compose config`](https://docs.docker.com/reference/cli/docker/compose/config/) representa la configuración Compose realmente combinada; [`docker compose ps`](https://docs.docker.com/reference/cli/docker/compose/ps/) muestra procesos y puertos publicados; [`docker compose images`](https://docs.docker.com/reference/cli/docker/compose/images/) identifica imágenes; [`docker inspect`](https://docs.docker.com/reference/cli/docker/inspect/) proporciona montajes, red y parámetros de ejecución. Las salidas pueden contener secretos y solo deben incluirse anonimizadas en tickets.

## Entradas y límites de entrega

Paperless-ngx conoce varias entradas que, antes de la canalización común, poseen ámbitos de fallo diferentes:

- El **directorio Consume** se supervisa mediante eventos del sistema de archivos. En sistemas de archivos sin inotify fiable, como algunos montajes NFS, debe activarse el polling; así lo indica la [documentación de configuración](https://docs.paperless-ngx.com/setup/#file-systems-without-inotify-support-eg-nfs).
- **Navegador y API REST** escriben cargas a través de la ruta web. Por tanto, el proxy inverso, el límite de tamaño de solicitud, el tiempo de espera y el almacenamiento temporal se sitúan antes del sistema de tareas.
- La **recuperación de correo** lee cuentas configuradas según reglas y genera tareas para los adjuntos coincidentes. La conexión IMAP, TLS, OAuth/contraseña, carpeta, criterio de búsqueda y acción posterior forman parte del contrato de entrega.

El directorio Consume es una zona de transferencia, no un archivo documental. Los archivos consumidos correctamente se eliminan de allí, mientras Paperless genera nuevos archivos gestionados en el fondo Media. La [FAQ](https://docs.paperless-ngx.com/faq/) advierte expresamente de no gestionar manualmente el directorio Media.

Una vez conocidos los procesos y las entradas, comienza el procesamiento real. Cada paso genera archivos, entradas de base de datos y señales de error diferentes.

## Canalización de Consume: de la entrada al objeto documental

El commit funcional no se produce al detectar un archivo, sino solo tras varios pasos. Según el tipo de archivo y la configuración, la canalización suele incluir:

1. **Detección y estabilización:** se comprueban el nombre de archivo, el tipo permitido, el tamaño y la finalización de la escritura.
2. **Creación de la tarea:** el Consumer o el servidor web entrega una tarea mediante el bróker.
3. **Parsing y preprocesamiento:** los parsers extraen texto existente o convierten el documento de entrada. Los archivos de Office y correo pueden requerir Tika y Gotenberg.
4. **OCR:** OCRmyPDF decide según el modo de OCR si omite el texto existente, lo procesa de nuevo o lo fuerza. Tesseract reconoce los idiomas configurados.
5. **Versión de archivo y miniatura:** se generan un archivo de visualización/archivo y datos de vista previa, en la medida en que lo prevean el tipo de archivo y la configuración.
6. **Persistencia:** se escriben el original y los derivados; se almacenan el objeto de base de datos, las sumas de comprobación, el texto y las propiedades técnicas.
7. **Metadatos y flujos de trabajo:** las reglas asignan etiquetas, corresponsal, tipo documental, ruta de almacenamiento, propietario u otros campos.
8. **Indexación:** se actualiza el fondo de texto completo en el índice de búsqueda.
9. **Finalización:** el estado de la tarea y el registro muestran éxito o error; solo entonces el documento es localizable funcionalmente.

Los scripts pre- y post-Consume pueden ampliar la ruta. La [documentación sobre hooks de Consume](https://docs.paperless-ngx.com/advanced_usage/#consume-hooks) deja claro que dichos scripts se ejecutan en el contexto de seguridad y error del proceso Paperless. Deben versionarse y operarse con tiempo de espera y códigos de salida inequívocos; un hook defectuoso puede bloquear toda la ingesta.

## OCR, formato de archivo y límites de calidad

OCRmyPDF orquesta el procesamiento de PDF y el OCR de Tesseract. El modo decide si se omite, se vuelve a procesar o se fuerza el texto existente. El idioma, el límite de páginas, la limpieza de imágenes, la corrección de rotación y el formato de salida influyen en el tiempo de CPU, el tamaño de archivo y la calidad de reconocimiento. Las configuraciones se describen en la [configuración OCR de Paperless](https://docs.paperless-ngx.com/configuration/#ocr-settings) y en la [documentación de OCRmyPDF](https://ocrmypdf.readthedocs.io/en/latest/); [Tesseract](https://tesseract-ocr.github.io/tessdoc/) documenta los modelos de idioma y el comportamiento del motor.

El original se conserva como objeto probatorio. Un PDF de archivo es una versión derivada que puede regenerarse mediante cambios posteriores de OCR/archivo. Esta distinción debe formar parte de los conceptos de conservación y valor probatorio:

| Objeto | Finalidad | Recuperabilidad |
|---|---|---|
| Original | archivo de entrada sin modificar | no reconstruible a partir del texto OCR |
| Archivo de archivo | versión normalizada, frecuentemente consultable | regenerable a partir del original y una canalización adecuada |
| Miniatura | vista previa de UI | derivable |
| Texto extraído | búsqueda y clasificación | extraíble de nuevo; el resultado puede variar según la versión |
| Suma de comprobación | comparación de integridad | solo probatoria junto con algoritmo y referencia |

El éxito del OCR no implica legibilidad funcional. Las comprobaciones por muestreo deben comparar número de páginas, orientación, cobertura de texto, caracteres especiales, tablas, firmas incrustadas y tamaño de archivo. Los picos de CPU y RAM suelen producirse al renderizar PDF grandes o ricos en imágenes, no en la escritura final de la base de datos.

## Originales, archivos, miniaturas y formato de ruta

Un documento de Paperless consta de varias partes acopladas. La base de datos conoce ID interno, título, fecha de creación y de incorporación, corresponsal, tipo documental, etiquetas, ruta de almacenamiento, contenido, propietario y otros campos. En el directorio Media se encuentran el original y, en su caso, el archivo de archivo y la miniatura. Los nombres de archivo y Storage Paths configurables cambian la disposición física, no la identidad del objeto de base de datos.

Los administradores evitan renombrar o mover directamente elementos en el directorio Media. El [`document_renamer`](https://docs.paperless-ngx.com/administration/#document-renamer) oficial aplica coherentemente el formato configurado. Una herramienta externa de sincronización de archivos puede proteger el fondo, pero no debe reorganizarlo por iniciativa propia junto a la aplicación en ejecución. Para fundamentos de almacenamiento, consulte [Storage](/kb/storage), y para copia y verificación, [Rclone](/kb/rclone).

## Metadatos, matching y flujos de trabajo

Los corresponsales, tipos documentales, etiquetas, rutas de almacenamiento, campos personalizados y valores de fecha forman una organización facetada. No sustituyen a un árbol de archivos, sino que describen un documento a lo largo de varias dimensiones independientes. El matching automático proporciona propuestas o asignaciones basadas en el contenido reconocido y los datos existentes; no es un servicio de verdad inmutable.

Los flujos de trabajo constan de desencadenantes, filtros opcionales y acciones secuenciales. Pueden establecer metadatos, permisos y otras propiedades durante el consumo o en otros eventos. La [documentación de flujos de trabajo](https://docs.paperless-ngx.com/usage/#workflows) describe los desencadenantes y las acciones. Para los administradores se derivan tres reglas:

- Toda automatización necesita un conjunto de entrada definido y un resultado observable.
- El orden de las reglas y las coincidencias múltiples se prueban como código de políticas.
- Tras los cambios, los documentos existentes solo se reevalúan de forma controlada con [`document_retagger`](https://docs.paperless-ngx.com/administration/#document-retagger).

## Búsqueda: consulta de base de datos y proyección Tantivy

La búsqueda de texto completo utiliza un índice Tantivy independiente. La base de datos sigue siendo la fuente principal para objetos documentales y metadatos; el índice acelera la consulta y la clasificación. Por ello, un documento puede ser visible en la base de datos pero faltar en las búsquedas de texto completo. La guía de migración documenta la migración a Tantivy y los campos de búsqueda modificados ([guía de migración](https://docs.paperless-ngx.com/migration-v3/#search-index-whoosh-tantivy)).

Después de actualizaciones o si se sospecha deriva, el índice puede reconstruirse mediante el comando de gestión. La reindexación repara una proyección, no restaura los originales. La planificación de capacidad tiene en cuenta el tamaño del índice, el espacio temporal durante una reconstrucción y la carga de E/S en paralelo al consumo normal.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Paperless-Suche und Wartung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker compose exec webserver document_index list
docker compose exec webserver document_index reindex --if-needed
docker compose exec webserver document_sanity_checker
docker compose exec webserver document_thumbnails --help</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker compose exec webserver document_index list
docker compose exec webserver document_index reindex --if-needed
docker compose exec webserver document_sanity_checker
docker compose exec webserver document_thumbnails --help</code></pre>
  </div>
</div>

Los comandos de gestión se inician con Compose en el contenedor del servidor web mediante [`docker compose exec`](https://docs.docker.com/reference/cli/docker/compose/exec/). Las instalaciones bare metal usan `python3 manage.py`; la [administración](https://docs.paperless-ngx.com/administration/#management-utilities) documenta la forma de invocación admitida y las ayudas integradas.

Los archivos OCR y los metadatos por sí solos aún no forman un servicio coherente. La base de datos, el bróker de tareas y la proyección de búsqueda deben corresponder al mismo estado de procesamiento.

## Base de datos, bróker y coherencia

PostgreSQL es la ruta estándar para instalaciones nuevas. MariaDB y SQLite son posibles, pero difieren en herramientas de copia de seguridad, comportamiento en paralelo y límites operativos. Las migraciones de base de datos forman parte de una actualización de Paperless; un binario antiguo restaurado no tiene por qué ser compatible con un esquema ya migrado. Por ello, la operación de la base de datos se trata como un ciclo de vida propio ([copia de seguridad y restauración de PostgreSQL](https://www.postgresql.org/docs/current/backup.html), [MariaDB Backup and Restore](https://mariadb.com/docs/server/server-usage/backing-up-and-restoring-databases/)).

El bróker transporta tareas entre procesos. La persistencia de Redis/Valkey no sustituye la copia de seguridad de la base de datos y los documentos. Tras la pérdida del bróker, los trabajos en curso o entregados pueden reiniciarse, perderse o hacerse visibles como un estado de tarea incoherente. Por ello, la aceptación se realiza sobre el objeto documental y sus archivos, no sobre una cola vacía.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Datenbank- und Brokertests">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker compose exec db pg_isready
docker compose exec broker redis-cli PING
docker compose logs --since 30m webserver
docker compose logs --since 30m broker db</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker compose exec db pg_isready
docker compose exec broker redis-cli PING
docker compose logs --since 30m webserver
docker compose logs --since 30m broker db</code></pre>
  </div>
</div>

[`pg_isready`](https://www.postgresql.org/docs/current/app-pg-isready.html) comprueba la aceptación de conexiones PostgreSQL, no el esquema ni las consultas funcionales. [`redis-cli`](https://redis.io/docs/latest/develop/tools/cli/) prueba la accesibilidad del bróker con `PING`. [`docker compose logs`](https://docs.docker.com/reference/cli/docker/compose/logs/) reúne registros de servicio acotados en el tiempo; las credenciales, títulos de documentos y rutas se anonimizan antes de compartirlos.

## Configuración, Secret Key y resolución de rutas

Con Docker, los ajustes suelen proporcionarse como archivo Environment o Compose-Env; `paperless.conf` no se aplica en ese caso. Las instalaciones bare metal buscan la configuración en un orden documentado. Algunos ajustes también pueden almacenarse en la aplicación y sobrescribir valores Environment. Por eso, la configuración efectiva es más que un único archivo ([orden de configuración](https://docs.paperless-ngx.com/configuration/#configuration)).

`PAPERLESS_SECRET_KEY` firma sesiones y valores sensibles y es obligatorio. `PAPERLESS_URL`, Allowed Hosts y los ajustes CSRF y CORS definen el límite web. Las credenciales de base de datos, correo, OAuth, Tika y AI opcional son secretos. Para numerosos valores, las configuraciones Docker pueden usar la variante `_FILE` y, por tanto, secretos montados; así lo documenta la [guía de configuración](https://docs.paperless-ngx.com/setup/#docker-compose-install).

La copia de seguridad de configuración incluye como mínimo:

- definición Compose/de servicio y referencia de imagen fijada,
- Environment sin copias en texto claro en repositorios no protegidos,
- Secret Key y referencias de secretos externos,
- asignación de UID/GID, volumen y bind mount,
- configuración de proxy, DNS, certificados y cuenta de correo,
- idiomas OCR, formato de nombres de archivo, flujos de trabajo y Custom Hooks.

## Límite web, TLS y proxy inverso

La aplicación debería estar accesible mediante HTTPS detrás de un proxy inverso configurado correctamente. El encabezado Host, la URL de origen, CSRF Trusted Origins, Forwarded Headers y los límites de carga deben coincidir. El proxy solo debe establecer encabezados Remote-User si autentica de forma fiable la identidad previa y elimina encabezados de cliente homónimos.

La [documentación de seguridad](https://docs.paperless-ngx.com/configuration/#hosting-security) describe los parámetros de Secret Key, URL, Host, CORS y CSRF. Los fundamentos de certificados y transporte se encuentran en [TLS](/kb/tls), y la protección sistemática en [endurecimiento](/kb/haertung). El proyecto advierte en el [repositorio](https://github.com/paperless-ngx/paperless-ngx#important-note) que los documentos se almacenan en texto claro y que Paperless-ngx no debe ejecutarse en un host no confiable.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Paperless-Web- und Portdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection paperless.example.net -Port 443
curl.exe -sS -D - -o NUL https://paperless.example.net/
Resolve-DnsName paperless.example.net
docker compose port webserver 8000</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ss -lntp
curl -sS -D - -o /dev/null https://paperless.example.net/
getent ahosts paperless.example.net
docker compose port webserver 8000</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) comprueba la accesibilidad TCP en Windows; [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) muestra listeners locales; [`curl`](https://curl.se/docs/manpage.html) hace visibles el estado, la redirección y los encabezados; [`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) o [`getent`](https://man7.org/linux/man-pages/man1/getent.1.html) comprueban la resolución de nombres; [`docker compose port`](https://docs.docker.com/reference/cli/docker/compose/port/) muestra el mapeo de puertos real.

## Identidades, permisos globales y permisos de objeto

Paperless-ngx utiliza el modelo de usuarios y grupos de Django. Los permisos globales controlan qué tipos de objeto y áreas de UI/API puede utilizar una cuenta. Los permisos relativos a objetos controlan Owner, View y Change para un documento, etiqueta u otro objeto específico. La [documentación de permisos](https://docs.paperless-ngx.com/usage/#permissions) subraya que los permisos de una etiqueta no se transfieren automáticamente a los documentos marcados con ella.

Los superusuarios pueden ver todos los objetos. El estado Staff/Admin debe distinguirse de ello y permite, entre otras cosas, vistas de backend y sistema. Para la operación diaria se usan cuentas y grupos normales; las cuentas de servicio reciben solo los permisos API y de objeto necesarios. Los flujos de trabajo pueden establecer propietario y permisos durante el consumo, algo especialmente importante para documentos sin propietario procedentes de directorios Consume.

La autenticación es posible de forma local, mediante Remote-User de proxy inverso o mediante procedimientos Social/OIDC configurados. Remote User solo se activa detrás de un límite de proxy cerrado. El ciclo de vida de MFA, sesión y token se incorpora al concepto general de identidad.

## API REST y límite de automatización

La API REST es accesible bajo el prefijo de la aplicación; una vista de esquema interactiva se encuentra en `/api/schema/view/`. Se admiten, entre otros, autenticación Basic, Session, Token, Remote-User y Headless-OIDC. La [documentación de API](https://docs.paperless-ngx.com/api/) describe búsqueda, permisos de objeto, operaciones masivas y encabezados de versión.

Un token de API es un secreto y se asigna por separado a cada automatización. Los clientes tienen en cuenta paginación, filtros, permisos, versión de API y repetición idempotente. Una respuesta de carga correcta no equivale aún a OCR finalizado; las automatizaciones correlacionan el ID de documento o tarea hasta alcanzar el estado final funcional. Los principios sobre contrato, autenticación y comportamiento de reintento se encuentran en [APIs](/kb/apis).

## Recuperación de correo como interfaz de mensajería

Las cuentas y reglas de correo conectan Paperless-ngx con un buzón IMAP. El Scheduler comprueba periódicamente las cuentas configuradas de forma predeterminada, aplica reglas y entrega los adjuntos coincidentes a la canalización Consume. El comando de gestión `mail_fetcher` puede activar manualmente la recuperación; así lo documenta la [administración de recuperación de correo](https://docs.paperless-ngx.com/administration/#fetching-e-mail).

Para los administradores de mensajería, los siguientes valores deben incluirse en el runbook:

- FQDN del servidor, puerto y modo TLS,
- método de autenticación, caducidad de secreto/token y cuenta de servicio,
- carpeta IMAP, criterio de búsqueda, tipos de archivo y tamaño máximo de mensaje,
- acción tras una importación correcta o fallida,
- tratamiento de mensajes entregados varias veces y adjuntos idénticos,
- correlación entre Message-ID, tarea Paperless e ID de documento.

La semántica IMAP está definida por [RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051). Una recuperación IMAP correcta solo demuestra la entrada en la ruta de tareas; la finalización de OCR y archivo se supervisa por separado.

## Documentos Office, Tika y Gotenberg

Apache Tika extrae contenido y metadatos de numerosos tipos de archivo; Gotenberg proporciona servicios de conversión. Paperless necesita ambos cuando está activado el procesamiento opcional de Office/correo. Sus endpoints se configuran por separado ([Tika Server](https://tika.apache.org/3.2.3/gettingstarted.html#Using_Tika_as_a_service), [configuración de Gotenberg](https://gotenberg.dev/docs/configuration)).

Ambos servicios amplían la superficie de ataque, las necesidades de recursos y el espacio de errores. Solo se hacen accesibles internamente, reciben límites de CPU/RAM/tiempo de espera y no se publican como API de conversión general. Si funcionan los PDF pero DOCX o EML fallan de forma coherente, el diagnóstico comienza en esta ruta secundaria y no en Tantivy ni en la base de datos.

## Operación de contenedores, bare metal y almacenamiento

Las vías oficiales son script de instalación, Docker Compose y bare metal. Compose proporciona definiciones de servicio reproducibles y volúmenes separados; bare metal ofrece más control sobre procesos, paquetes y systemd, pero exige una gestión explícita del ciclo de vida. La [matriz de configuración](https://docs.paperless-ngx.com/setup/#overview) clasifica las vías según esfuerzo y uso.

Los directorios persistentes no se confunden con el ciclo de vida del contenedor. Con Compose, Media, Data, Consume, Export y la ruta de base de datos se resuelven explícitamente. Los sistemas de archivos remotos o de red se comprueban respecto al comportamiento POSIX, renombrado atómico, bloqueos, latencia, inotify y disponibilidad. De lo contrario, una breve interrupción puede parecer una pérdida permanente de documentos. Los fundamentos sobre montajes se encuentran en [Container](/kb/container) y [Storage](/kb/storage).

Ante una incidencia, primero se determina en qué fase de la canalización se encuentra el documento. Después se comprueban de forma específica la cola, el Worker, el convertidor externo, la base de datos y el almacenamiento de archivos.

## Observabilidad y diagnóstico de tareas

Una visión operativa útil separa cuatro señales:

- **Salud del servicio:** el proceso se ejecuta, el puerto escucha, la base de datos y el bróker son accesibles.
- **Rendimiento:** tareas nuevas, documentos finalizados, tiempos de ejecución y profundidad de cola.
- **Errores:** tareas por etapa y tipo de archivo, salida de parser/OCR, errores de correo y almacenamiento.
- **Integridad del fondo:** originales, archivos de archivo o miniaturas ausentes o dañados, archivos huérfanos, deriva de índice.

El Sanity Checker integrado detecta, entre otras cosas, archivos ausentes, ilegibles o modificados respecto a la suma de comprobación almacenada, así como archivos Media huérfanos. La lista completa se encuentra en la [documentación de Sanity Checker](https://docs.paperless-ngx.com/administration/#sanity-checker). Una alerta sobre número de errores sin ID de documento/tarea y fase de canalización apenas permite actuar.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Paperless-Task- und Storage-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker compose logs --since 1h --timestamps webserver
docker compose exec webserver document_sanity_checker
docker compose exec webserver mail_fetcher
docker stats --no-stream
docker system df -v</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker compose logs --since 1h --timestamps webserver
docker compose exec webserver document_sanity_checker
docker compose exec webserver mail_fetcher
docker stats --no-stream
docker system df -v</code></pre>
  </div>
</div>

[`docker stats`](https://docs.docker.com/reference/cli/docker/container/stats/) muestra el uso actual de CPU/memoria, [`docker system df`](https://docs.docker.com/reference/cli/docker/system/df/) la ocupación de almacenamiento de Docker. Ambos son instantáneas. Para identificar causas se correlacionan serie temporal, ID de tarea y registros de Paperless. La metodología general se encuentra en [Troubleshooting](/kb/troubleshooting).

## Capacidad, paralelismo y límites

La planificación de capacidad comienza con páginas por día, formatos de entrada, tamaño de archivo medio y máximo, idiomas OCR, tareas paralelas, tiempo de conservación y volumen de búsqueda. Un documento puede generar simultáneamente original, PDF de archivo, miniatura, registro de base de datos, índice de texto completo y archivos temporales de renderizado. El espacio libre también debe tener en cuenta reindexación, exportación y actualización.

Un mayor paralelismo de Worker acelera el OCR ligado a CPU solo hasta que se saturan CPU, RAM y E/S. Al mismo tiempo, aumenta el número de archivos temporales y conexiones de base de datos. La recuperación de correo y Consumer pueden generar más tareas de las que los Worker procesan; por ello se observa la antigüedad de la cola, no solo su longitud actual. Los límites se prueban con documentos representativos, no con PDF de texto pequeños.

El alcance de la copia de seguridad se deriva del modelo de datos: originales, archivos de archivo, base de datos, configuración, Secret Key y, en su caso, servicios adicionales deben poder restaurarse conjuntamente.

## Copia de seguridad: qué objetos deben protegerse conjuntamente

La [documentación de copia de seguridad](https://docs.paperless-ngx.com/administration/#making-backups) oficial menciona dos vías principales: Document Exporter o una copia coordinada de volúmenes/directorios y base de datos. El Exporter escribe documentos, miniaturas, metadatos y contenido de base de datos en un directorio transportable. Los tokens de API no se transfieren; la exportación y la importación deben ser compatibles en versión.

Un conjunto de recuperación completo contiene:

| Objeto | Por qué es necesario | Regla de coherencia |
|---|---|---|
| Base de datos | ID, metadatos, usuarios, reglas, referencias de tareas | debe corresponder al snapshot de Media |
| Media | originales, archivos de archivo, miniaturas | sin escrituras a medio terminar |
| Data | índice y estados auxiliares | parcialmente derivable, pero útil para una recuperación rápida |
| Configuración | rutas, URL, OCR, correo, proxy, base de datos | mismo contrato de nombres y montajes |
| Secretos | Secret Key, credenciales de BD/correo/OIDC | protegidos, separados y aptos para restauración |
| Manifiesto de exportación | asignación transportable de objetos y archivos | conservar junto con los archivos exportados |
| Runbooks | secuencia, versión, pruebas de aceptación | disponibles sin conexión |

El Exporter puede actualizar exportaciones existentes y comparar opcionalmente sumas de comprobación. Esto ahorra tiempo, pero no sustituye una copia separada y versionada: con `--delete` se pueden eliminar documentos borrados del destino de exportación. El [`document_exporter`](https://docs.paperless-ngx.com/administration/#document-exporter) documenta los detalles y opciones.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Paperless-Export und Recoveryprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker compose exec -T webserver document_exporter ../export --compare-checksums
Get-ChildItem -Recurse .\export | Measure-Object -Property Length -Sum
Get-FileHash .\export\manifest.json -Algorithm SHA256
docker compose exec webserver document_sanity_checker</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker compose exec -T webserver document_exporter ../export --compare-checksums
du -sh ./export
sha256sum ./export/manifest.json
docker compose exec webserver document_sanity_checker</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash) y [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) garantizan la prueba de transferencia del manifiesto. Una suma de comprobación solo demuestra la igualdad de dos conjuntos de bytes, no la importabilidad ni la integridad funcional. Los fundamentos de las copias de seguridad se encuentran en [copia de seguridad y recuperación ante desastres](/kb/backup-dr).

## Restauración y aceptación

El [`document_importer`](https://docs.paperless-ngx.com/administration/#document-importer) espera una instalación vacía o, con `--data-only`, una base de datos vacía. Por ello, una prueba de restauración utiliza rutas de destino aisladas, una versión de Paperless compatible con la copia de seguridad y ningún volumen de producción.

La aceptación incluye como mínimo:

1. Aplicar configuración y secretos al entorno de destino.
2. Proporcionar una base de datos vacía y rutas Media/Data vacías.
3. Importar la exportación o restaurar de forma coherente la base de datos y los archivos.
4. Ejecutar de forma controlada las migraciones y, si procede, la construcción del índice.
5. Comprobar el inicio de sesión de usuarios, grupos y permisos.
6. Comparar número de documentos, metadatos y muestras del original y del archivo.
7. Probar la búsqueda de texto completo y la visualización de miniaturas.
8. Ejecutar Sanity Checker sin hallazgos sin explicación.
9. Consumir un nuevo documento de prueba mediante la entrada real.
10. Documentar RPO y RTO con marcas de tiempo.

Una restauración solo se considera correcta después de esta prueba integral. «El contenedor arranca» no es un criterio de aceptación.

## Actualizaciones, migraciones de base de datos y límite de rollback

Antes de una actualización se finalizan las tareas Consume activas, se crea una copia de seguridad y se revisan las notas de versión para detectar cambios incompatibles. La [guía de actualización](https://docs.paperless-ngx.com/administration/#updating-paperless) separa los procedimientos Docker y bare metal y describe la migración de la base de datos y la reindexación según necesidad. Una imagen se fija a una serie publicada o a un digest; `latest` no es un punto de rollback reproducible.

Después de una migración de base de datos, un rollback normalmente consiste en restaurar la base de datos y el fondo de archivos correspondiente, no solo en iniciar la imagen antigua. Los archivos de archivo o metadatos nuevos creados después del punto de copia de seguridad deben tratarse funcionalmente antes del regreso. Los principios generales de versiones y cambios se encuentran en [Releases](/kb/releases) y [Migration](/kb/migration).

El [changelog](https://docs.paperless-ngx.com/changelog/) es la fuente dependiente del tiempo para nuevas funciones, cambios incompatibles y correcciones. En cambio, este artículo describe principios estables de arquitectura y operación.

## Patrones de error por fase de canalización

Dado que un documento atraviesa varios pasos de procesamiento, el síntoma visible suele mostrar solo la última fase correcta. La tabla conduce del síntoma a la siguiente evidencia sólida.

| Síntoma | Límite probable | Primeras evidencias |
|---|---|---|
| El archivo permanece en el directorio Consume | Watcher, polling, UID/GID, finalización de escritura | Montaje, permisos, registro de Consumer |
| La carga responde, pero no se crea ningún documento | Bróker o Task Worker | ID de tarea, PING del bróker, registro del Worker |
| Solo falla DOCX/EML | Tika/Gotenberg | Endpoint, tiempo de espera, registro del convertidor |
| OCR muy lento o proceso finalizado | CPU/RAM, número de páginas, paquetes de idioma | Estadísticas de contenedor, evento de kernel/runtime, tamaño de página |
| Documento visible, falta texto completo | Parser/OCR o índice de búsqueda | Texto extraído, registro de tarea, estado de reindexación |
| Falta miniatura, existe original | Archivo derivado | Sanity Checker, comando de miniaturas |
| Error de inicio de sesión/POST detrás del proxy | URL, Host, CSRF, Forwarded Header | Encabezados HTTP, configuración efectiva |
| La recuperación periódica de correo está detenida | Scheduler, bróker, autenticación de correo | Registro de Scheduler, `mail_fetcher`, prueba IMAP |
| Archivos desaparecidos tras reinicio | Montaje incorrecto o efímero | `docker inspect`, inventario de volúmenes |
| La restauración no importa | Condición de versión, ruta o vaciedad | Manifiesto de exportación, versión de destino, registro de importación |

La [colección oficial de Troubleshooting](https://docs.paperless-ngx.com/troubleshooting/) añade mensajes de error concretos. Antes de realizar cambios, los administradores siempre guardan ID de tarea, nombre de documento, ventana temporal, configuración efectiva, estado de contenedor/proceso y estado de todos los servicios dependientes.

## Historia técnica

El proyecto Paperless original estableció el modelo básico de directorio Consume, OCR y archivo consultable. Paperless-ng continuó el desarrollo como fork. Paperless-ngx surgió posteriormente como sucesor comunitario oficial para distribuir el mantenimiento y la responsabilidad entre varios colaboradores; el [README del proyecto](https://github.com/paperless-ngx/paperless-ngx#paperless-ngx) describe esta procedencia.

La plataforma evolucionó de una aplicación Django principalmente del lado del servidor a un DMS con frontend Angular, API REST, permisos relativos a objetos, motor de flujos de trabajo y procesamiento paralelo de tareas. OCRmyPDF/Tesseract siguieron siendo la base OCR; Tika y Gotenberg ampliaron los formatos de entrada admitidos. La ruta de búsqueda cambió de Whoosh a Tantivy, haciendo visible el índice como dominio propio de migración y operación ([migración a Tantivy](https://docs.paperless-ngx.com/migration-v3/)).

Esta historia explica la arquitectura actual: un documento Paperless-ngx es simultáneamente objeto relacional, conjunto de archivos gestionados, resultado OCR y proyección de búsqueda. El mantenimiento y la recuperación deben considerar las cuatro perspectivas.

## Lista de comprobación para administradores

Para finalizar, se comprueban conjuntamente entradas, Worker, almacenamiento, búsqueda y copia de seguridad. La lista ayuda a no pasar por alto ninguna dependencia externa fuera del contenedor Paperless.

- **Inventario:** imagen/commit, aplicación Django/Angular, Worker, Scheduler, Broker, base de datos, Tika/Gotenberg, proxy y montajes.
- **Objetos de datos:** nombrar por separado base de datos, original, archivo, miniatura, índice, configuración, secretos y manifiesto de exportación.
- **Entradas:** documentar Consume, carga/API y correo con reglas de tamaño, tipo, autenticación y error.
- **Canalización:** correlacionar el ID de tarea mediante parser, OCR, archivado, metadatos e índice hasta la finalización.
- **Permisos:** probar permisos globales y relativos a objetos, superusuario, grupos y asignaciones de flujo de trabajo.
- **Capacidad:** supervisar antigüedad de cola, tiempo OCR, espacio temporal, crecimiento de Media/índice y conexiones de base de datos.
- **Seguridad:** incluir HTTPS, confianza en el proxy, Secret Key, tokens, almacenamiento en texto claro y parsers externos en el modelo de amenazas.
- **Copia de seguridad:** proteger de forma coherente la base de datos y Media; no olvidar configuración y secretos.
- **Restauración:** comprobar instancia destino vacía, versión compatible, muestras, búsqueda, Sanity Checker y nueva ejecución Consume.
- **Cambio:** registrar de antemano notas de versión, migración de esquema/índice, espacio libre y límite de rollback real.

## Fuentes

- [Paperless-ngx – Documentación](https://docs.paperless-ngx.com/)
- [Paperless-ngx – Repositorio GitHub](https://github.com/paperless-ngx/paperless-ngx)
- [Paperless-ngx – Desarrollo](https://docs.paperless-ngx.com/development/)
- [Paperless-ngx – Uso](https://docs.paperless-ngx.com/usage/)
- [Celery](https://docs.celeryq.dev/en/stable/)
- [Paperless-ngx – Manifiesto de proyecto Python](https://github.com/paperless-ngx/paperless-ngx/blob/dev/pyproject.toml)
- [Paperless-ngx – Dockerfile](https://github.com/paperless-ngx/paperless-ngx/blob/dev/Dockerfile)
- [Paperless-ngx – Configuración](https://docs.paperless-ngx.com/configuration/)
- [Bróker compatible con Redis](https://redis.io/docs/latest/)
- [PostgreSQL](https://www.postgresql.org/docs/current/)
- [Paperless-ngx – Instalación](https://docs.paperless-ngx.com/setup/)
- [Docker – compose config](https://docs.docker.com/reference/cli/docker/compose/config/)
- [Docker – compose ps](https://docs.docker.com/reference/cli/docker/compose/ps/)
- [Docker – compose images](https://docs.docker.com/reference/cli/docker/compose/images/)
- [Docker – inspect](https://docs.docker.com/reference/cli/docker/inspect/)
- [Paperless-ngx – FAQ](https://docs.paperless-ngx.com/faq/)
- [Paperless-ngx – Consume Hooks](https://docs.paperless-ngx.com/advanced_usage/)
- [OCRmyPDF – Documentación](https://ocrmypdf.readthedocs.io/en/latest/)
- [Tesseract – Documentación](https://tesseract-ocr.github.io/tessdoc/)
- [Paperless-ngx – Document Renamer](https://docs.paperless-ngx.com/administration/)
- [Paperless-ngx – Migración a Tantivy](https://docs.paperless-ngx.com/migration-v3/)
- [Docker – compose exec](https://docs.docker.com/reference/cli/docker/compose/exec/)
- [PostgreSQL – Backup and Restore](https://www.postgresql.org/docs/current/backup.html)
- [MariaDB – Backup and Restore](https://mariadb.com/docs/server/server-usage/backing-up-and-restoring-databases/)
- [PostgreSQL – pg_isready](https://www.postgresql.org/docs/current/app-pg-isready.html)
- [Redis – redis-cli](https://redis.io/docs/latest/develop/tools/cli/)
- [Docker – compose logs](https://docs.docker.com/reference/cli/docker/compose/logs/)
- [Microsoft – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [Linux man-pages – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html)
- [curl – Manual](https://curl.se/docs/manpage.html)
- [Microsoft – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [Linux man-pages – getent(1)](https://man7.org/linux/man-pages/man1/getent.1.html)
- [Docker – compose port](https://docs.docker.com/reference/cli/docker/compose/port/)
- [Paperless-ngx – API REST](https://docs.paperless-ngx.com/api/)
- [RFC 9051 – IMAP4rev2](https://datatracker.ietf.org/doc/html/rfc9051)
- [Apache Tika – Operación de servidor](https://tika.apache.org/3.2.3/gettingstarted.html)
- [Gotenberg – Configuración](https://gotenberg.dev/docs/configuration)
- [Docker – stats](https://docs.docker.com/reference/cli/docker/container/stats/)
- [Docker – system df](https://docs.docker.com/reference/cli/docker/system/df/)
- [Microsoft – Get-FileHash](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – utilidades sha2](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Paperless-ngx – Changelog](https://docs.paperless-ngx.com/changelog/)
- [Paperless-ngx – Troubleshooting](https://docs.paperless-ngx.com/troubleshooting/)
