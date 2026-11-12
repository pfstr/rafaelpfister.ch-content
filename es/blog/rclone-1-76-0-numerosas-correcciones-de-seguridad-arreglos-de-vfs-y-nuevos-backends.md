---
title: "Rclone 1.76.0: numerosas correcciones de seguridad, arreglos de VFS y nuevos backends"
navTitle: "Rclone 1.76.0"
description: "Rclone 1.76.0, del 6 de noviembre de 2026, cierra vulnerabilidades en serve, rc y varios backends, entre ellas una vulnerabilidad del proxy de autenticación en serve sftp con CVSS 8.1, corrige cinco errores en la capa VFS y bloquea por defecto las redirecciones de HTTPS a HTTP. Los operadores de montajes, servicios serve y rc deberían actualizar y revisar los valores predeterminados modificados."
date: "2026-11-11"
kategorie: "Rclone"
timeToRead: "7 min de lectura"
themen:
  - rclone
produkte:
  - "rclone"
protokolle:
  - "releases"
  - "storage"
slug: "rclone-1-76-0-numerosas-correcciones-de-seguridad-arreglos-de-vfs-y-nuevos-backends"
translationId: article-aa7be1fdff77b8cc
translationOf: rclone-1-76-0
url: https://rafaelpfister.ch/es/blog/rclone-1-76-0-numerosas-correcciones-de-seguridad-arreglos-de-vfs-y-nuevos-backends
translationSourceHash: 79bd08de9bbf74eb195b3bdc4093b8e92f84cd358eb2a4b0a950ad3109a08e56
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:30:25.456Z
translationReview: automatic
---

# Rclone 1.76.0: numerosas correcciones de seguridad, arreglos de VFS y nuevos backends

Rclone 1.76.0 se publicó el 6 de noviembre de 2026, unas cuatro semanas después de la 1.75.2 (9 de octubre de 2026). El changelog incluye veinte entradas en la sección Security, entre ellas una vulnerabilidad en el proxy de autenticación de `rclone serve sftp` con CVSS 8.1, filtraciones de credenciales en redirecciones de los backends Swift, NetStorage, Seafile y Ulozto, así como varias salidas mediante enlaces simbólicos en el backend `local`. Además, incluye cinco correcciones en la capa VFS, que utilizan `rclone mount` y los comandos serve, ampliaciones del backend S3, dos nuevos backends (Dosya e Infomaniak kDrive) y el nuevo comando `index`. Cambian algunos valores predeterminados: se rechazan las redirecciones de HTTPS a HTTP, los listeners TCP multipath están desactivados y los patrones glob de los filtros se evalúan de forma más estricta. La versión está dirigida a todos quienes utilizan Rclone para montajes, sincronización, copias de seguridad o como servidor con `rclone serve` y `rclone rcd`. Encontrará información sobre los montajes en contenedores en el artículo [Operar montajes de Rclone en Docker de forma fiable](/blog/rclone-mount-in-docker-container).

## Las versiones de un vistazo

| Versión | Fecha | Tipo | Puntos clave |
|---|---|---|---|
| 1.76.0 | 06.11.2026 | Versión de funcionalidades | Correcciones de seguridad en serve sftp (CVSS 8.1), rc, lib/http, local y cuatro backends; Go 1.27.3; cinco arreglos de VFS; protección de redirecciones `--allow-https-downgrade`; nuevos backends Dosya e Infomaniak kDrive; comando `index` |

La versión más alta hasta entonces era la 1.75.2. No hubo ninguna versión de parche 1.76.x hasta el 11 de noviembre de 2026.

## Correcciones de seguridad con advisory

El proyecto Rclone publicó las siguientes vulnerabilidades como GitHub Security Advisory el 6 de noviembre de 2026. Todas indican 1.76.0 como versión corregida; ninguna tiene asignado un identificador CVE (a 11 de noviembre de 2026, marcado como «CVE-PENDING» en el changelog).

- **GHSA-66qv-66mg-9h74, CVSS 8.1 (High):** En `rclone serve sftp` con `--auth-proxy`, la clave de caché del proxy de autenticación no incluía el tipo de inicio de sesión. Por ello, una clave pública conocida podía reutilizarse como contraseña si el proxy de autenticación trataba de forma diferente la autenticación por clave y contraseña, el atacante y la víctima usaban el mismo nombre de usuario y la misma IP de cliente, y la entrada de caché seguía siendo válida (300 segundos de inactividad de forma predeterminada). Las mismas IP de cliente se dan, por ejemplo, detrás de NAT, un host bastión o un balanceador de carga. Según el advisory, los demás protocolos serve no se ven afectados. El campo de versiones afectadas indica «>= 1.51.0, <= 1.76.0» y el de versión corregida, 1.76.0; prevalece la entrada de corrección del changelog de 1.76.0.
- **GHSA-vmgx-g9c5-669p, CVSS 4.2:** En `serve sftp`, `serve ftp`, `serve http` y `serve webdav` con `--auth-proxy`, una revocación en el proxy de autenticación no surtía efecto mientras existiera una respuesta en caché. Desde la 1.76.0, una revocación se aplica en un plazo de cinco minutos. `serve s3` no se ve afectado.
- **GHSA-hwj7-wp9g-wrv9, CVSS 5.3:** Mediante el tiempo de respuesta de la comprobación bcrypt se podían distinguir nombres de usuario válidos de inválidos con `--htpasswd` y `--user`. Entre los afectados se encuentran `serve http`, `serve webdav`, `serve restic`, `serve s3`, `rcd` con `--rc-htpasswd` y el endpoint de métricas con `--metrics-htpasswd`, en versiones de la 1.56.0 a la 1.75.2. Las contraseñas no se revelan.
- **GHSA-f8p4-8g4j-q3m2, CVSS 5.5:** Si el servidor rc se ejecutaba sin autenticación HTTP, un cliente con acceso al puerto podía agotar la memoria del proceso mediante una solicitud JSON grande a un método no autorizado (denegación de servicio, versiones de la 1.46.0 a la 1.75.2). La 1.76.0 limita el tamaño de las solicitudes y rechaza las llamadas no autorizadas antes de leer el cuerpo. Como solución temporal, el advisory menciona la autenticación HTTP para rc y un límite de cuerpo en el proxy inverso; según el advisory, la vinculación a localhost no separa entre sí las cuentas de usuario locales.
- **GHSA-m97j-7x65-wh93, CVSS 4.8:** Los backends Swift, NetStorage y Seafile transmitían credenciales en redirecciones a otro host o de HTTPS a HTTP. Solo se puede explotar si el propio endpoint configurado redirige, por ejemplo, a través de un proxy. Como solución temporal, el advisory recomienda exclusivamente endpoints `https://` y ningún proxy que degrade a HTTP.
- **GHSA-3xpx-hjg2-vrjh, CVSS 4.8:** El backend Ulozto enviaba los encabezados `X-Auth-Token` y `X-User-Token` en redirecciones desde enlaces de descarga. Para ello, un atacante necesita una posición de red desde la que pueda redirigir el tráfico de CDN.
- **GHSA-r5wv-vqqh-2r9c, CVSS 4.0:** Con `--links` y `--metadata`, el backend `local` podía escribir atributos `user.*` fuera del directorio de destino mediante un enlace simbólico inyectado. Hasta actualizar, el advisory recomienda no utilizar conjuntamente ambas opciones cuando terceros puedan influir en el destino.

Además, la 1.76.0 corrige en el backend `local` otros casos sin advisory propio en los que `MkdirMetadata`, `Remove`, `Rmdir` y `DirMove` podían actuar fuera del directorio raíz a través de enlaces simbólicos.

## Dependencias y valores predeterminados modificados

Rclone 1.76.0 está compilado con Go 1.27.3. El changelog menciona CVE-2026-111943: Go añade en `crypto/x509` una lista de bloqueo para certificados intermedios comprometidos (CVSS 6.8 según la evaluación de CISA-ADP; afectados Go anterior a 1.26.10 y de 1.27.0 hasta antes de 1.27.3). Además, se actualizaron `golang.org/x/net` a v0.62.0 y `github.com/andybalholm/brotli` a v1.2.7, este último por CVE-2026-111497; encontrará contexto sobre esta vulnerabilidad en el artículo sobre [CVE-2026-111497 en el decodificador Brotli](/blog/brotli-cve-2026-111497).

Los listeners TCP multipath están desactivados de forma predeterminada desde la 1.76.0. El changelog lo justifica con CVE-2026-98432; la entrada CVE describe un use-after-free en la pila TCP del kernel Linux (CVSS 9.8, corregido, entre otras versiones, en 6.1.190, 6.6.159, 6.12.113 y 6.18.56). Las actualizaciones del kernel de la distribución proporcionan la corrección propiamente dicha; Rclone solo reduce su propia superficie de ataque. El changelog no menciona ninguna bandera para volver a activarlos.

Resumen de los cambios en el comportamiento predeterminado:

- **Redirecciones:** Rclone rechaza las redirecciones de HTTPS a HTTP. La nueva bandera `--allow-https-downgrade` permite volver a aceptarlas.
- **Filtros:** `[!...]` en patrones glob es ahora una clase de caracteres negada, `[^...]` excluye `/`. Antes de utilizarlos en producción, compruebe los archivos de filtros y las reglas `--include`/`--exclude` que empleen tales clases con `--dry-run`.
- **local:** `--copy-links` ya no sigue bucles de enlaces simbólicos. Sin privilegios de root, Rclone solo establece los propietarios que puede establecer un usuario normal. La falta de espacio de almacenamiento ahora interrumpe las transferencias multihilo con un error fatal.
- **Validación de parámetros:** Se rechazan indicaciones de tamaño demasiado grandes, NaN y duraciones desbordadas, así como indicaciones de hora `--bwlimit` sin dos puntos o con signo.
- **bisync:** `--max-delete` tiene en cuenta `--track-renames`; se rechazan valores superiores a 100.
- **operations:** `--immutable` ahora también se aplica a `copyto`, `moveto` y a la copia de archivos individuales.
- **Obsoleto:** `--rc-web-gui`, así como la opción `serve s3` `--no-cleanup`, que ahora es un alias del nuevo `--directory-markers`.

Debería ejecutar una vez tras la actualización los scripts y unidades que invocan Rclone con parámetros fijos, y comprobar si muestran mensajes de error por valores rechazados.

## Montaje y caché VFS

Para `rclone mount`, el changelog indica una corrección: las instancias VFS no se liberaban en montajes fallidos ni al desmontar. En la capa VFS, que comparten Mount y los comandos serve, la 1.76.0 corrige cinco errores:

- Se perdía un truncate que reducía una archivo no almacenado en caché.
- Renombrar un directorio durante la expiración de la caché de directorios provocaba un fallo.
- Las entradas virtuales permanecían en directorios eliminados.
- Tras un cambio de nombre que solo se diferenciaba en mayúsculas/minúsculas o normalización Unicode, las entradas aparecían duplicadas.
- Las cargas de un archivo modificado no se iniciaban mientras se seguía leyendo el archivo.

El último punto afecta a montajes con caché VFS en los que se sigue leyendo un archivo modificado mientras debe cargarse. En estas configuraciones, observe la cola de cargas después de la actualización y realice una prueba de escritura y lectura, como se describe en el [plan de pruebas para montajes de Rclone](/blog/cloud-mount-testen-dummy-pdfs).

## Backend S3 y Proton Drive

En el backend S3, Rclone deja de almacenar temporalmente las cargas con Object Lock cuando se conoce el valor MD5 de la fuente. Es nuevo el comando de backend `link` con overrides firmados para encabezados de respuesta. Se añaden las regiones Fastly EU Central 1 y Scaleway IT-MIL, varias regiones Huawei OBS y los proveedores Autonomys Auto Drive, Clever Cloud Cellar, Infomaniak Swiss Backup, Neon Object Storage y RelAix.Storage. Las cargas multihilo calculan sumas de comprobación en operaciones de lectura de 1 MiB; tras una restauración se muestra la Storage Class real. Se corrigieron `rmdir` en el nivel raíz con `--s3-directory-markers` y las URL prefirmadas para GCS, Linode y Ceph.

Para `rclone serve s3`, la 1.76.0 aporta la bandera `--directory-markers` y cambios en tags, ETags, ETags multipart y escrituras condicionales. También se ha corregido que los buckets no pudieran eliminarse tras una carga multipart cancelada.

El changelog de la 1.76.0 no contiene ninguna entrada sobre el backend Proton Drive. Las últimas correcciones al respecto proceden de la 1.75.0 (lógica de reintentos, bloqueos en errores de validación y horas de modificación incorrectas) y de la 1.75.1 (los archivos cargados con la 1.75.0 no se podían leer en las aplicaciones de Proton). El artículo [Proton Drive en Linux](/blog/proton-drive-linux-status) describe el estado de la compatibilidad de Proton Drive con Linux.

## Otras novedades

Los dos nuevos backends son Dosya (Kaya) e Infomaniak kDrive. El nuevo comando `index` escribe listas estáticas de directorios en un remoto y, en ejecuciones posteriores, puede reindexar únicamente los directorios modificados. Rclone admite CRC-32C como hash, `sync` puede controlar el orden de las transferencias por patrones, y la vista de directorios de `serve http` y `serve webdav` está ordenada y rediseñada. Según el changelog, la GUI ya no se considera beta. Las estadísticas ya no cuentan `totalBytes` por duplicado en las copias del lado del servidor, y `serve sftp` informa de un error al usar `rmdir` sobre un archivo, en lugar de eliminarlo.

## Procedimiento recomendado

La versión objetivo es la 1.76.0. La actualización es especialmente urgente en sistemas con `rclone serve sftp --auth-proxy`, con `rclone rcd` o `--rc` sin autenticación HTTP, con servicios serve protegidos mediante `--htpasswd`, así como con remotos de tipo Swift, NetStorage, Seafile o Ulozto. En instalaciones desde los binarios oficiales, `rclone selfupdate` actualiza el archivo ejecutable y comprueba la descarga mediante hash y firma; con `rclone selfupdate --check`, Rclone muestra previamente qué versión se instalaría. Para instalaciones desde paquetes deb o rpm, la documentación indica `--package deb` y `--package rpm`, respectivamente. Los montajes y procesos serve en ejecución no utilizarán la nueva versión hasta reiniciarse. Después, compruebe las reglas de filtros con clases de caracteres, los horarios `--bwlimit` y las configuraciones que dependen de redirecciones HTTP. El changelog y la página de lanzamiento no mencionan problemas conocidos de la 1.76.0 (a 11 de noviembre de 2026).

## Fuentes

1.  [Rclone: Changelog](https://rclone.org/changelog/): entradas sobre la 1.76.0 (6 de noviembre de 2026), así como de la 1.75.0 a la 1.75.2 con correcciones de seguridad y cambios en Mount, VFS, S3 y Proton Drive.

2.  [GitHub: rclone v1.76.0](https://github.com/rclone/rclone/releases/tag/v1.76.0): página de lanzamiento con binarios y referencia al changelog.

3.  [GitHub: Security Advisories de rclone](https://github.com/rclone/rclone/security/advisories): resumen de los advisories publicados el 6 de noviembre de 2026.

4.  [GHSA-66qv-66mg-9h74](https://github.com/rclone/rclone/security/advisories/GHSA-66qv-66mg-9h74): serve sftp, caché del proxy de autenticación sin tipo de inicio de sesión, CVSS 8.1.

5.  [GHSA-vmgx-g9c5-669p](https://github.com/rclone/rclone/security/advisories/GHSA-vmgx-g9c5-669p): la revocación en el proxy de autenticación no surte efecto con una respuesta en caché, CVSS 4.2.

6.  [GHSA-hwj7-wp9g-wrv9](https://github.com/rclone/rclone/security/advisories/GHSA-hwj7-wp9g-wrv9): detección de nombres de usuario mediante timing bcrypt con `--htpasswd`, CVSS 5.3.

7.  [GHSA-f8p4-8g4j-q3m2](https://github.com/rclone/rclone/security/advisories/GHSA-f8p4-8g4j-q3m2): agotamiento de memoria en el servidor rc mediante solicitudes JSON sobredimensionadas, CVSS 5.5.

8.  [GHSA-m97j-7x65-wh93](https://github.com/rclone/rclone/security/advisories/GHSA-m97j-7x65-wh93): filtraciones de credenciales en redirecciones de Swift, NetStorage y Seafile, CVSS 4.8.

9.  [GHSA-3xpx-hjg2-vrjh](https://github.com/rclone/rclone/security/advisories/GHSA-3xpx-hjg2-vrjh): filtración de token en redirecciones en el backend Ulozto, CVSS 4.8.

10.  [GHSA-r5wv-vqqh-2r9c](https://github.com/rclone/rclone/security/advisories/GHSA-r5wv-vqqh-2r9c): salida mediante enlace simbólico con `--links --metadata` en el backend local, CVSS 4.0.

11.  [CVE.org: CVE-2026-111943](https://www.cve.org/CVERecord?id=CVE-2026-111943): lista de bloqueo para certificados intermedios comprometidos en Go `crypto/x509`, corregido en Go 1.27.3.

12.  [CVE.org: CVE-2026-98432](https://www.cve.org/CVERecord?id=CVE-2026-98432): use-after-free en la pila TCP del kernel Linux, CVSS 9.8, motivo de los listeners TCP multipath desactivados.

13.  [Rclone: rclone selfupdate](https://rclone.org/commands/rclone_selfupdate/): documentación sobre autoactualización, `--check` y `--package`.

14.  [CVE-2026-111497: vulnerabilidad en el decodificador Brotli](/blog/brotli-cve-2026-111497): contexto sobre la vulnerabilidad de Brotli que la 1.76.0 aborda mediante la actualización de dependencias.
