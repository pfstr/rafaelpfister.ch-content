---
title: "Operar montajes de Rclone en Docker de forma fiable"
navTitle: "Rclone en Docker"
description: "Para que un montaje FUSE de un contenedor también funcione en el host y en otros contenedores, deben coordinarse la propagación de montajes, AppArmor y la recuperación tras fallos."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "9 min de lectura"
themen:
  - rclone
related:
  - paperless-dokumente-clouddienst-auslagern
slug: "operar-montajes-de-rclone-en-docker-de-forma-fiable"
translationOf: "rclone-mount-in-docker-container"
url: "https://rafaelpfister.ch/es/blog/operar-montajes-de-rclone-en-docker-de-forma-fiable"
translationId: article-a08b15399e144547
translationModel: gpt-5.6-terra
translatedAt: 2026-07-29T07:02:42.741Z
translationReview: automatic
translationSourceHash: 9b1f0ebdc53ebc1f61e127ca462d0b92c4e48e717c4ac91778c59fa1f6915823
---

Un montaje de Rclone se ejecuta en un contenedor Docker, pero también debe estar disponible en el host y en otros contenedores. Para ello, los eventos de montaje deben atravesar varios espacios de nombres. Una sola opción de Compose no es suficiente.

En una prueba práctica con Ubuntu 25.10, kernel 6.17 y Docker 29.6 aparecieron tres errores independientes entre sí: Docker degradó `rshared` sin avisar, AppArmor bloqueó `fusermount3`, y un contenedor consumidor siguió aferrado al montaje antiguo tras un reinicio. El caso de uso concreto fue un [almacenamiento en la nube para Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern); los mismos mecanismos también se aplican a otras herramientas FUSE como sshfs.

## 1. La fuente del host debe ser ella misma `shared`

Para que un montaje del contenedor llegue al host, el bind necesita la propagación `rshared`:

```yaml
    volumes:
      - type: bind
        source: /srv/storage/media
        target: /data
        bind:
          propagation: rshared
```

`rshared` solo funciona si la fuente del bind en el host **es a su vez un punto de montaje con propagación shared**. Un directorio normal no cumple este requisito. Aun así, Docker no muestra ningún error, sino que utiliza silenciosamente una propagación más débil. Puede verse en `/proc/self/mountinfo` dentro del contenedor:

```text
1938 2077 8:2 /srv/storage/media /data rw,relatime master:1 - ext4 /dev/sda2 rw
```

`master:1` significa propagación slave: los montajes entran desde el host, pero nunca salen hacia él. Lo correcto sería `shared:N`. La solución es hacer un bind de la fuente sobre sí misma y marcarlo como shared:

```bash
mount --bind /srv/storage/media /srv/storage/media
mount --make-shared /srv/storage/media
```

Para que sobreviva a un reinicio, debe incluirse en una unidad systemd con `Before=docker.service`. Comprobación: `findmnt -no PROPAGATION /srv/storage/media` debe devolver `shared`.

## 2. AppArmor también verifica `fusermount3` dentro del contenedor

Con la propagación correcta llegó la siguiente sorpresa. El montaje en la ruta compartida seguía fallando:

```text
NOTICE: mount helper error: fusermount3: mount failed: Permission denied
CRITICAL: Fatal error: failed to mount FUSE fs: fusermount: exit status 1
```

Los permisos adicionales habituales del contenedor no cambiaron nada: ni `CAP_SYS_ADMIN` y `/dev/fuse` ni `unconfined` o incluso `--privileged`. Un montaje tmpfs funcionaba en el mismo destino, y FUSE funcionaba en otras rutas. Solo el registro de auditoría del kernel mostró la causa real:

```text
audit: type=1400 apparmor="DENIED" operation="mount" class="mount"
  info="failed mntpnt match" error=-13 profile="fusermount3"
  name="/data/documents/originals/" fstype="fuse.rclone"
```

Ubuntu incluye un **perfil de AppArmor para el binario `fusermount3`** que solo permite montajes FUSE en una lista positiva de patrones de puntos de montaje. Este perfil también se aplica a fusermount3 **dentro del contenedor**. Lo decisivo es la ruta tal como la ve el contenedor:

```text
mount fstype=@{fuse_types} ... -> @{HOME}/**/,
mount fstype=@{fuse_types} ... -> /mnt/{,**/},
mount fstype=@{fuse_types} ... -> /media/**/,
mount fstype=@{fuse_types} ... -> /tmp/**/,
```

`/data` no está en la lista, tampoco `/srv`. Que el contenedor se ejecute sin confinamiento no ayuda: el perfil está vinculado al archivo ejecutable, no al contenedor.

La salida aprovecha que solo fusermount3 está sujeto al perfil, mientras que un `mount --bind` normal no lo está: montar FUSE en una ruta **permitida** y publicarlo desde allí mediante un bind en la ruta compartida.

```sh
rclone mount remote:ruta /mnt/inner/documentos --allow-other --vfs-cache-mode full &
# esperar hasta que el montaje responda y, después:
mount --bind /mnt/inner/documentos /data/documentos
```

El bind es una llamada mount(2) normal y se propaga al host como cualquier otra a través de la ruta shared. Se pudo verificar hasta en un segundo contenedor, que podía leer los archivos como uid 1000. `--allow-other` es obligatorio cuando un usuario distinto del que realiza el montaje accede a los archivos; para ello, en el contenedor de Rclone debe figurar `user_allow_other` en `/etc/fuse.conf` (ya es el caso en la imagen oficial).

## 3. Los consumidores necesitan `rslave`

La tercera trampa afecta al otro lado. Si el proceso de Rclone muere y el montaje se reconstruye, el host lo ve de inmediato. Sin embargo, un contenedor que haya incluido la ruta mediante un bind normal no lo ve:

```text
ls: cannot access '/usr/src/app/media': Transport endpoint is not connected
```

Docker utiliza `rprivate` de forma predeterminada para los bind mounts: un montaje que surge en el host **después** de iniciar el contenedor nunca llega a su espacio de nombres de montaje. El contenedor queda atascado en el montaje FUSE muerto hasta que se vuelve a crear. La solución cuesta una línea:

```yaml
    volumes:
      - type: bind
        source: /srv/storage/media
        target: /usr/src/app/media
        bind:
          propagation: rslave
```

Con `rslave`, el host reenvía nuevos eventos de montaje al contenedor. En la prueba, tras finalizar bruscamente un montaje y reconstruirlo, el consumidor volvió a ver todos los archivos **sin reiniciarse**. El contador de reinicios se mantuvo en cero.

## Recuperación sin intervención manual

De los tres componentes resulta un patrón global robusto que no requiere un demonio watchdog:

1. El contenedor de montaje comprueba sus montajes en un bucle. Si uno deja de responder, termina con un código de error.
2. `restart: unless-stopped` hace que Docker reinicie el contenedor.
3. Al iniciarse, el contenedor elimina primero los **montajes huérfanos de la ejecución anterior**: de lo contrario, un bind muerto en la ruta de destino bloquearía su publicación de nuevo, y un usuario sin privilegios no puede eliminarlo desde el host. Dentro del contenedor sí es posible, y el umount se propaga hacia fuera:

```sh
while grep -q " /data/documentos " /proc/self/mountinfo; do
    umount -l /data/documentos 2>/dev/null || break
done
```

4. Después, montar y publicar normalmente; los consumidores con `rslave` adoptan el montaje nuevo por sí mismos.

En la prueba, toda la cadena duró 160 segundos: se terminó el proceso de Rclone, se detectó el error, se reinició el contenedor, se eliminó el montaje huérfano y se volvió a publicar el nuevo montaje. El contenedor consumidor siguió ejecutándose durante ese tiempo y solo notó una breve interrupción.

Quien ejecute Rclone directamente **en el host** mediante systemd evita los dos primeros problemas y solo necesita `rslave` en los contenedores consumidores. El contenedor adicional merece la pena sobre todo si el host debe permanecer libre de instalaciones de Rclone o si se quieren gestionar varios montajes de forma uniforme. En ese caso, los tres niveles deben configurarse deliberadamente.

## Fuentes

1.  [Docker: Bind mounts: configurar la propagación de bind](https://docs.docker.com/engine/storage/bind-mounts/#configure-bind-propagation): los modos de propagación rprivate, rslave y rshared, y su comportamiento predeterminado.

2.  [Documentación del kernel: Shared Subtrees](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt): la propagación de montajes del kernel Linux en la que se basan las opciones bind de Docker.

3.  [Rclone mount](https://rclone.org/commands/rclone_mount/): modos de caché VFS, --allow-other y los límites del montaje FUSE.

4.  [Documentación de AppArmor (Ubuntu)](https://documentation.ubuntu.com/server/how-to/security/apparmor/): cómo se vinculan los perfiles a archivos ejecutables; el perfil de fusermount3 se encuentra en /etc/apparmor.d/fusermount3.
