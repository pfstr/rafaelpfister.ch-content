---
title: "Pi-hole 2026.11.0: FTL 6.9 y Core 6.6 corrigen once vulnerabilidades de seguridad"
navTitle: "Pi-hole 2026.11.0"
description: "La imagen Docker Pi-hole 2026.11.0 incluye FTL v6.9 y Core v6.6 y corrige, entre otros problemas, la ejecución de código por parte de administradores mediante files.gravity (CVSS 7.2), así como varias vulnerabilidades de enlaces simbólicos en el instalador. Los filtros regex, webserver.acl y la API de sesiones se comportan en parte de forma distinta; revise estos puntos tras la actualización."
date: "2026-11-12"
kategorie: "Redes y DNS"
timeToRead: "7 min de lectura"
themen:
  - smart-home-iot
produkte:
  - "pihole"
protokolle:
  - "releases"
  - "dns"
  - "container"
slug: "pi-hole-2026-11-0-ftl-6-9-y-core-6-6-corrigen-once-vulnerabilidades-de-seguridad"
translationId: article-0eb045e8f65bedd0
translationOf: pihole-2026-11-0
url: https://rafaelpfister.ch/es/blog/pi-hole-2026-11-0-ftl-6-9-y-core-6-6-corrigen-once-vulnerabilidades-de-seguridad
translationSourceHash: 44a4448ed57216f76890c2807ea763f7c340fcdc0fffa70b8e8e50816c28f4c4
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T13:42:56.205Z
translationReview: automatic
---

# Pi-hole 2026.11.0: FTL 6.9 y Core 6.6 corrigen once vulnerabilidades de seguridad

Pi-hole publicó la imagen Docker 2026.11.0 el 11 de noviembre de 2026. Incluye Pi-hole FTL v6.9 y Pi-hole Core v6.6, publicados el mismo día; según el fabricante, el Web Interface se mantiene en v6.6.1. Las dos versiones de los componentes corrigen once avisos de seguridad, incluidos cuatro de gravedad alta (CVSS 7.2 a 7.3). FTL v6.9 también actualiza dnsmasq, SQLite y el motor regex TRE, y modifica el comportamiento de los filtros regex, `webserver.acl` y partes de la API. La versión está dirigida a todos los operadores de Pi-hole, tanto en contenedores Docker como en instalaciones bare metal. Según las notas de la versión, la predecesora 2026.10.1, del 31 de octubre de 2026, incorporó principalmente una imagen base Alpine actualizada (3.24.3).

## Las versiones de un vistazo

| Versión | Fecha | Tipo | Puntos clave |
|---|---|---|---|
| Docker 2026.11.0 | 11.11.2026 | Lanzamiento de imagen | incluye FTL v6.9, Core v6.6 y Web v6.6.1; `setpriv` en lugar de `sudo`, logrotate mediante la crontab del contenedor |
| FTL v6.9 | 11.11.2026 | Lanzamiento menor | siete avisos (uno alto), dnsmasq v2.93+42, SQLite 3.54.1, actualización de TRE con cambios en el comportamiento regex |
| Core v6.6 | 11.11.2026 | Lanzamiento menor | cinco avisos (cuatro altos, uno medio), el instalador solo escribe en `/etc/pihole` como el usuario `pihole` |
| Web v6.6.1 | 17.10.2026 | sin cambios | no hay versión nueva en este lanzamiento |

## Correcciones de seguridad de gravedad alta

Los avisos se publicaron el 11 de noviembre de 2026. En ese momento no se había asignado una ID CVE para los avisos revisados; según el texto, se ha solicitado para el aviso sobre `files.gravity` (situación al 12 de noviembre de 2026).

- **GHSA-pj9m-9623-2g75 (FTL) y GHSA-cq9w-mv93-mxh7 (Core), CVSS 7.2:** Ejecución de código por parte de un administrador autenticado (CWE-78, CWE-89). FTL 6.8 había relajado la validación de rutas, de modo que `files.gravity` podía contener una comilla simple. `gravity.sh` insertaba este valor sin protección en una sentencia SQL `ATTACH`; así, mediante la shell sqlite3 de FTL, se podían ejecutar comandos del sistema operativo como el usuario `pihole`, tanto en instalaciones Docker como bare metal, según el aviso. Están afectadas las versiones de Core hasta la 6.5 y FTL 6.8. Core ahora escapa el valor y FTL vincula la ruta como parámetro en Teleporter. El fabricante recalca que ambos paquetes deben actualizarse conjuntamente.
- **GHSA-wp3p-45h3-2v82 (Core), CVSS 7.3:** Corrección incompleta de las vulnerabilidades de enlaces simbólicos de octubre (CWE-59, CWE-61). `pihole -r` y `pihole -up` seguían enlaces simbólicos al escribir `install.log`, `migration_backup_v6` y `hosts/custom.list` en `/etc/pihole`. El requisito es la ejecución de código como el usuario `pihole` y que un administrador ejecute posteriormente una reparación o actualización como root. Están afectadas las versiones hasta la 6.5.
- **GHSA-5w4v-f65m-323f (Core), CVSS 7.3:** root seguía enlaces simbólicos y rutas de la configuración al descargar `macvendor.db`, en `pihole checkout` y en `pihole uninstall`.
- **GHSA-jvg9-fm8x-2jqq (Core), CVSS 7.3:** `pihole -d` seguía enlaces simbólicos en directorios propiedad del usuario `pihole`. De este modo, root podía copiar archivos arbitrarios al registro de depuración y sobrescribir otros o cambiar su propietario.

Por ello, Core v6.6 solo escribe `install.log`, la copia de seguridad de migración, `hosts/custom.list` y `macvendor.db` como el usuario `pihole`. `pihole -d` tampoco opera ya como root en estos directorios.

## Hashes de contraseñas en el registro de depuración

**GHSA-qfq3-88xg-765w (Core), CVSS 5.3:** El registro de depuración de `pihole -d`, y por tanto también las cargas al servicio Tricorder, contenía el hash de la contraseña web, el hash de la contraseña de la aplicación y el secreto TOTP. Core v6.6 elimina estos valores del registro. Si en el pasado subió un registro de depuración, el fabricante recomienda cambiar la contraseña y las contraseñas de aplicación y volver a configurar la autenticación de dos factores.

## Correcciones de seguridad de gravedad baja en FTL

- **GHSA-x6xh-vxcm-g8rm, CVSS 3.8:** Las sesiones de aplicación sin `webserver.api.app_sudo` y las sesiones CLI podían listar todas las sesiones de API y finalizar cualquiera de ellas.
- **GHSA-87wp-wm3p-8cxm, CVSS 3.7:** Las entradas IPv4 en `webserver.acl` no se aplicaban a clientes IPv4 en un `webserver.port` de doble pila como `+80`.
- **GHSA-cg4p-5r3r-m54p, CVSS 3.1:** Un cambio de contraseña mediante `pihole-FTL --config` o `pihole setpassword` podía revertirse mediante una escritura simultánea de configuración a través de la API; la contraseña anterior y sus sesiones seguían siendo válidas.
- **GHSA-fj2f-pgf2-j888, CVSS 2.2:** Las consultas registradas con `misc.privacylevel` 3 podían llegar a la base de datos a largo plazo después de reducir el nivel.
- **GHSA-9mh9-95wj-4w4q y GHSA-9hh4-j8jf-6c96, ambos CVSS 2.2:** Un fallo al leer las métricas durante la recarga del archivo hosts, así como un use-after-free al importar un Teleporter en formato v5. Ambos requieren una sesión válida.

## Comportamiento modificado tras la actualización

Según las notas de la versión, FTL v6.9 no es deliberadamente una v6.8.1, porque la actualización de TRE modifica la evaluación de algunos filtros regex guardados:

- **Regex:** Un límite inferior ausente (`{,n}`, `{,}`) significa ahora `{0,n}` o `{0,}`. Se rechazan más de tres grupos anidados con coincidencia aproximada y referencias inversas en repeticiones ilimitadas.
- **DNS local:** Se rechazan las entradas en `dns.cnameRecords` con un nombre de dominio no válido o un TTL que no cabe en un entero.
- **webserver.acl:** Las entradas IPv4 se aplican en todos los puertos, también en puertos de doble pila. Las entradas IPv6, salvo `::ffff:a.b.c.d`, solo coinciden con clientes IPv6; por tanto, `[::]/0` ya no permite clientes IPv4.
- **API de sesiones:** Las sesiones de aplicación sin `webserver.api.app_sudo` y las sesiones CLI reciben el estado 403 en `GET /api/auth/sessions` y `DELETE /api/auth/session/{id}`. Una sesión puede seguir finalizándose a sí misma con `DELETE /api/auth`.
- **Servidor web:** Se rechazan las solicitudes con más de 64 cabeceras. `/api/info/system` devuelve un error en lugar de valores nulos si no se pueden leer las medias de carga.
- **API DHCP:** `DELETE /api/dhcp/leases/{ip}` responde con 503 en vez de 400 si el resolvedor no ha procesado la solicitud a tiempo.
- **Contraseña:** Una contraseña establecida mediante `pihole setpassword` o `pihole-FTL --config` ya no se sobrescribe con el siguiente cambio de configuración en el Web Interface.

Tras la actualización, revise los filtros regex con límites inferiores o referencias inversas, las entradas en `webserver.acl` (en particular `[::]/0`) y los scripts o integraciones que listen o finalicen sesiones con contraseñas de aplicación.

## DNS, DHCP y otras correcciones en FTL v6.9

El dnsmasq integrado pasa a v2.93+42 e incluye además una corrección en `blockdata_alloc()` que upstream todavía no ha incorporado. Según las notas de la versión, se corrigen, entre otros problemas, una lectura heap out-of-bounds en `zone_status()` (DNSSEC), una escritura heap out-of-bounds en `blockdata_read()`, un desbordamiento del búfer de nombres DNS, un off-by-one en la opción DHCPv6 FQDN, así como SERVFAIL en respuestas wildcard más profundas y respuestas perdidas con TCP pipelining. Con DNSSEC activado, dnsmasq reserva ranuras TCP adicionales para sus propias consultas de validación, para que esta funcione incluso si los clientes agotan el límite de conexiones TCP. El upstream sin filtrado de Quad9 se identifica ahora como validador de DNSSEC.

En la configuración y la API, FTL v6.9 corrige, entre otros problemas, cambios perdidos con dos modificaciones simultáneas de configuración, un worker TCP que mantenía `pihole.toml.lock` hasta 300 segundos e ignoraba variables `FTLCONF_` cuando FTL se iniciaba sin `pihole.toml`. El servidor web podía quedarse bloqueado anteriormente con una carga de CPU del 100 % y dejar de aceptar conexiones. En contenedores, `/api/info/system` informaba hasta ahora de la carga del host. También es relevante para la operación que `gravity.db` solo se vuelve a abrir cuando el archivo ha cambiado; según el fabricante, esto representaba la mayor parte de la carga de CPU de FTL en reposo. SQLite pasa a 3.54.1.

## Cambios en Core v6.6 y en la imagen Docker

Core v6.6 cambia de `sudo` a `setpriv` al reducir privilegios de root al usuario `pihole`; por tanto, `sudo` deja de ser una dependencia. Según el fabricante, los comandos administrativos como `pihole -up` siguen requiriendo privilegios de root. La rotación de registros pasa ahora a estar a cargo del logrotate diario del sistema; se elimina la rotación nocturna forzada de Pi-hole porque entraba en conflicto con la ejecución del sistema y rotaba los registros dos veces. Otras correcciones afectan a `pihole flush quiet`, `pihole updatechecker` y una advertencia cuando un repositorio local para git no está marcado como seguro (hasta ahora, `pihole -v` solo mostraba N/A). Se eliminó la capacidad innecesaria `CAP_IPC_LOCK`.

La imagen Docker adopta el cambio a `setpriv` y programa la rotación de registros mediante la crontab del contenedor. La imagen nocturna se compila ahora desde la rama `development`.

## Procedimiento recomendado

La versión objetivo es la imagen Docker 2026.11.0 o, en instalaciones bare metal, FTL v6.9 con Core v6.6. Dado que la vulnerabilidad mediante `files.gravity` solo se cierra con ambos componentes, debería actualizar FTL y Core conjuntamente. Antes, cree una copia de seguridad con Teleporter (Settings, Teleporter en el Web Interface o `pihole-FTL --teleporter`), tal como recomienda el fabricante, y lea los changelogs debido a los cambios de comportamiento. Actualice las instalaciones bare metal con `pihole -up`. Según la documentación, `pihole -up` y `pihole -r` están desactivados en el contenedor; allí debe sustituir la imagen. Con Docker Compose, ejecute los siguientes comandos en el directorio de `docker-compose.yml`, según la documentación:

```bash
docker compose pull
docker compose down
docker compose up -d
```

Para instalaciones con `docker run`, la documentación indica `docker pull pihole/pihole:latest`, después `docker stop pihole` y `docker rm pihole`, y un nuevo `docker run` con los argumentos anteriores. Si anteriormente subió un registro de depuración, cambie además las contraseñas y vuelva a configurar 2FA. Las notas de la versión no incluyen indicaciones sobre problemas conocidos ni plazos de soporte (situación al 12 de noviembre de 2026).

## Fuentes

1.  [GitHub: pi-hole/docker-pi-hole Release 2026.11.0](https://github.com/pi-hole/docker-pi-hole/releases/tag/2026.11.0): Notas de la versión de la imagen Docker con versiones de los componentes, cambios de Docker y lista de los once avisos.

2.  [GitHub: pi-hole/FTL Release v6.9](https://github.com/pi-hole/FTL/releases/tag/v6.9): Correcciones de seguridad, actualización de dnsmasq y SQLite, cambios de comportamiento en regex, ACL y API.

3.  [GitHub: pi-hole/pi-hole Release v6.6](https://github.com/pi-hole/pi-hole/releases/tag/v6.6): Correcciones de seguridad de Core, cambio a setpriv, modificaciones en la rotación de registros.

4.  [GitHub: pi-hole/web Release v6.6.1](https://github.com/pi-hole/web/releases/tag/v6.6.1): Estado del Web Interface del 17 de octubre de 2026, sin cambios en 2026.11.0.

5.  [GitHub: pi-hole/docker-pi-hole Release 2026.10.1](https://github.com/pi-hole/docker-pi-hole/releases/tag/2026.10.1): Imagen predecesora con Alpine 3.24.3.

6.  [GitHub Advisory GHSA-pj9m-9623-2g75](https://github.com/pi-hole/FTL/security/advisories/GHSA-pj9m-9623-2g75): Ejecución de código de administrador mediante files.gravity, CVSS 7.2, versiones afectadas y corregidas, CVE solicitada.

7.  [GitHub Advisory GHSA-wp3p-45h3-2v82](https://github.com/pi-hole/pi-hole/security/advisories/GHSA-wp3p-45h3-2v82): Corrección incompleta de enlaces simbólicos en el instalador, CVSS 7.3, requisitos.

8.  [Pi-hole Blog: Pi-hole FTL v6.9 and Core v6.6 released](https://pi-hole.net/blog/2026/11/11/pi-hole-ftl-v6-9-and-core-v6-6-released/): Anuncio con indicaciones de actualización y recomendación tras subir registros de depuración.

9.  [Pi-hole Docs: Upgrading Pi-hole in Docker](https://docs.pi-hole.net/docker/upgrading/): Comandos de actualización para Docker Compose y docker run; pihole -up está desactivado en el contenedor.
