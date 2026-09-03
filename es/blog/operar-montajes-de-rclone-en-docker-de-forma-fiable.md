---
title: "Operar montajes de Rclone en Docker de forma fiable"
navTitle: "Rclone en Docker"
description: "Para que un montaje FUSE de un contenedor funcione también en el host y en otros contenedores, deben combinarse la propagación de montajes, AppArmor y la recuperación tras fallos."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "9 min de lectura"
themen:
  - rclone
related:
  - paperless-dokumente-clouddienst-auslagern
slug: "operar-montajes-de-rclone-en-docker-de-forma-fiable"
translationOf: "rclone-mount-in-docker-container"
translationId: article-a08b15399e144547
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:29:29.141Z
translationReview: automatic
translationSourceHash: 5cba6faedde80db33a3f35e999758cb09a93ccb85cfb9021a45026a99173bb26
url: https://rafaelpfister.ch/es/blog/operar-montajes-de-rclone-en-docker-de-forma-fiable
---

Un montaje de Rclone se ejecuta en un contenedor Docker, pero también debe estar disponible en el host y en otros contenedores. Para ello, los eventos de montaje deben atravesar varios espacios de nombres. Una sola opción de Compose no es suficiente.

En una prueba práctica con Ubuntu 25.10, kernel 6.17 y Docker 29.6 aparecieron tres fallos independientes: Docker degradaba `rshared` sin avisar, AppArmor bloqueaba `fusermount3`, y un contenedor consumidor seguía vinculado al montaje antiguo tras reiniciarse. El caso de uso concreto era un [almacenamiento en la nube para Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern); los mismos mecanismos también se aplican a otras herramientas FUSE como sshfs.

## 1. La fuente del host debe ser `shared` por sí misma

Para que un montaje desde el contenedor llegue al host, el bind necesita la propagación `rshared`:

```yaml
    volumes:
      - type: bind
        source: /srv/storage/media
        target: /data
        bind:
          propagation: rshared
```

`rshared` solo funciona si la fuente del bind en el host **es a su vez un punto de montaje con propagación shared**. Un directorio normal no cumple este requisito. Aun así, Docker no muestra ningún error, sino que utiliza silenciosamente una propagación más débil. Esto puede comprobarse en `/proc/self/mountinfo` dentro del contenedor:

```text
1938 2077 8:2 /srv/storage/media /data rw,relatime master:1 - ext4 /dev/sda2 rw
```

`master:1` significa propagación slave: los montajes entran desde el host, pero nunca salen. Lo correcto sería `shared:N`. La solución consiste en hacer un bind de la fuente sobre sí misma y marcarlo como shared:

```bash
mount --bind /srv/storage/media /srv/storage/media
mount --make-shared /srv/storage/media
```

<details class="options-details">
<summary>Explicación de las opciones</summary>

| Opción | Efecto |
|---|---|
| `--bind quelle ziel` | Vincula un directorio a una segunda ruta; aquí, a sí mismo, convirtiendo el directorio en un punto de montaje independiente |
| `--make-shared pfad` | Establece la propagación de este punto de montaje en shared, de modo que los eventos de montaje se reenvían en ambas direcciones |

</details>

Para que sobreviva a un reinicio, debe incluirse en una unidad systemd con `Before=docker.service`. Comprobación: `findmnt -no PROPAGATION /srv/storage/media` debe devolver `shared`.

## 2. AppArmor también comprueba `fusermount3` dentro del contenedor

Con la propagación correcta, surgió el siguiente problema. El montaje en la ruta compartida seguía fallando:

```text
NOTICE: mount helper error: fusermount3: mount failed: Permission denied
CRITICAL: Fatal error: failed to mount FUSE fs: fusermount: exit status 1
```

Los permisos adicionales habituales del contenedor no cambiaron nada: ni `CAP_SYS_ADMIN` ni `/dev/fuse`, ni `unconfined` ni siquiera `--privileged`. Un montaje tmpfs funcionaba en el mismo destino, y FUSE funcionaba en otras rutas. Solo el registro de auditoría del kernel mostró la causa real:

```text
audit: type=1400 apparmor="DENIED" operation="mount" class="mount"
  info="failed mntpnt match" error=-13 profile="fusermount3"
  name="/data/documents/originals/" fstype="fuse.rclone"
```

Ubuntu incluye un **perfil AppArmor para el binario `fusermount3`** que permite montajes FUSE únicamente en una lista positiva de patrones de puntos de montaje. Este perfil también se aplica a fusermount3 **dentro del contenedor**. Lo decisivo es la ruta tal como la ve el contenedor:

```text
mount fstype=@{fuse_types} ... -> @{HOME}/**/,
mount fstype=@{fuse_types} ... -> /mnt/{,**/},
mount fstype=@{fuse_types} ... -> /media/**/,
mount fstype=@{fuse_types} ... -> /tmp/**/,
```

`/data` no figura en la lista, `/srv` tampoco. Que el contenedor se ejecute sin confinamiento no ayuda: el perfil está ligado al archivo ejecutable, no al contenedor.

La salida aprovecha que solo fusermount3 está sujeto al perfil, mientras que un `mount --bind` normal no lo está: montar FUSE en una ruta **permitida** y publicarlo desde allí mediante un bind en la ruta compartida.

```sh
rclone mount remote:pfad /mnt/inner/dokumente --allow-other --vfs-cache-mode full &
# esperar hasta que el montaje responda; después:
mount --bind /mnt/inner/dokumente /data/dokumente
```

<details class="options-details">
<summary>Explicación de las opciones</summary>

| Opción | Efecto |
|---|---|
| `remote:pfad` | Remoto de Rclone y ruta que se van a montar |
| `/mnt/inner/dokumente` | Punto de montaje bajo `/mnt`, un patrón permitido por el perfil AppArmor |
| `--allow-other` | Permite que usuarios distintos del que realiza el montaje accedan al montaje FUSE |
| `--vfs-cache-mode full` | Almacena completamente en caché las operaciones de lectura y escritura locales |
| `&` | Inicia el montaje en segundo plano para que la shell quede libre para el bind |
| `mount --bind quelle ziel` | Publica el montaje FUSE mediante un bind en la ruta compartida; como llamada mount(2), no está sujeto al perfil de fusermount3 |

</details>

El bind es una llamada mount(2) normal y, como cualquier otra, se propaga al host a través de la ruta shared. Pudo verificarse incluso en un segundo contenedor, que podía leer los archivos como uid 1000. `--allow-other` es obligatorio en cuanto un usuario distinto del que realiza el montaje accede a los archivos; para ello, en el contenedor de Rclone debe figurar `user_allow_other` en `/etc/fuse.conf` (ya es el caso en la imagen oficial).

## 3. Los consumidores necesitan `rslave`

El tercer problema afecta al otro lado. Si el proceso de Rclone falla y se reconstruye el montaje, el host lo ve de inmediato. Sin embargo, un contenedor que ha integrado la ruta mediante un bind normal no lo ve:

```text
ls: cannot access '/usr/src/app/media': Transport endpoint is not connected
```

Docker utiliza `rprivate` de forma predeterminada para los bind mounts: un montaje que surge en el host **después** de iniciar el contenedor nunca llega a su espacio de nombres de montaje. El contenedor queda bloqueado en el montaje FUSE ya desconectado hasta que se recrea. La solución cuesta una línea:

```yaml
    volumes:
      - type: bind
        source: /srv/storage/media
        target: /usr/src/app/media
        bind:
          propagation: rslave
```

Con `rslave`, el host reenvía los nuevos eventos de montaje al contenedor. En la prueba, tras finalizar forzosamente y reconstruir el montaje, el consumidor volvió a ver todos los archivos **sin reiniciarse**. El contador de reinicios permaneció en cero.

## Recuperación sin intervención manual

De los tres elementos resulta un patrón global robusto que no requiere un demonio watchdog:

1. El contenedor de montaje comprueba sus montajes en un bucle. Si uno deja de responder, finaliza con un código de error.
2. `restart: unless-stopped` hace que Docker reinicie el contenedor.
3. Al iniciarse, el contenedor primero elimina **los montajes huérfanos de la ejecución anterior**: de lo contrario, un bind huérfano en la ruta de destino bloquearía la publicación de nuevo, y un usuario sin privilegios no puede eliminarlo desde el host. En el contenedor sí es posible, y el umount se propaga hacia fuera:

```sh
while grep -q " /data/dokumente " /proc/self/mountinfo; do
    umount -l /data/dokumente 2>/dev/null || break
done
```

<details class="options-details">
<summary>Explicación de las opciones</summary>

| Opción | Efecto |
|---|---|
| `grep -q` | Sin salida; solo el código de salida indica si la ruta sigue figurando como montaje en `/proc/self/mountinfo` |
| `umount -l` | Desmontaje diferido: elimina inmediatamente el montaje del árbol y solo limpia las referencias cuando deja de usarse |
| `2>/dev/null` | Suprime los mensajes de error de umount |
| `\|\| break` | Finaliza el bucle si falla un umount, en lugar de seguir ejecutándose indefinidamente |

</details>

4. Después, montar y publicar normalmente; los consumidores con `rslave` adoptan el montaje nuevo automáticamente.

En la prueba, toda la cadena duró 160 segundos: se terminó el proceso de Rclone, se detectó el fallo, se reinició el contenedor, se eliminó el montaje huérfano y se volvió a publicar el nuevo montaje. El contenedor consumidor siguió ejecutándose mientras tanto y solo notó una breve interrupción.

Quien ejecute Rclone directamente **en el host** mediante systemd evita los dos primeros problemas y solo necesita `rslave` en los contenedores consumidores. El contenedor adicional merece la pena sobre todo si el host debe permanecer libre de instalaciones de Rclone o si se quieren gestionar varios montajes de forma uniforme. En ese caso, los tres niveles deben configurarse conscientemente.

## Fuentes

1.  [Docker: Bind mounts: configure bind propagation](https://docs.docker.com/engine/storage/bind-mounts/#configure-bind-propagation): los modos de propagación rprivate, rslave y rshared, así como su comportamiento predeterminado.

2.  [Documentación del kernel: Shared Subtrees](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt): la propagación de montajes del kernel Linux en la que se basan las opciones bind de Docker.

3.  [Rclone mount](https://rclone.org/commands/rclone_mount/): los modos de caché VFS, --allow-other y las limitaciones del montaje FUSE.

4.  [Documentación de AppArmor (Ubuntu)](https://documentation.ubuntu.com/server/how-to/security/apparmor/): cómo se vinculan los perfiles a archivos ejecutables; el perfil fusermount3 se encuentra en /etc/apparmor.d/fusermount3.
