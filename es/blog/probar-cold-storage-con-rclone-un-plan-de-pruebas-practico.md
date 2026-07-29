---
title: "Probar Cold Storage con Rclone: un plan de pruebas práctico"
navTitle: "Probar Rclone"
description: "Antes de que un servicio lea sus archivos desde la nube mediante un montaje de Rclone, deberías comprobar algo más que el acceso a directorios. Este plan de pruebas cubre lecturas en frío, lecturas en caliente, operaciones de escritura, comportamiento de la caché, integridad de archivos y fallos."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "11 min de lectura"
themen:
  - "rclone"
related:
  - "rclone-mount-in-docker-container"
  - "paperless-dokumente-clouddienst-auslagern"
slug: "probar-cold-storage-con-rclone-un-plan-de-pruebas-practico"
translationOf: "cloud-mount-testen-dummy-pdfs"
url: "https://rafaelpfister.ch/es/blog/probar-cold-storage-con-rclone-un-plan-de-pruebas-practico"
---

Un montaje de Rclone se configura rápidamente. El remoto aparece como un directorio, `ls` muestra archivos y la primera prueba funcional está superada. Para el funcionamiento en producción, eso aún dice muy poco.

En cuanto un servicio accede al montaje, surgen más preguntas: ¿cuánto tarda el primer acceso a un archivo? ¿Qué accesos atiende la caché local? ¿Qué ocurre con un archivo que aún no se ha subido si Rclone se bloquea? ¿Un contenedor en ejecución vuelve a ver el montaje reconstruido? ¿Y cómo reacciona el servicio si la nube no está disponible temporalmente?

Este artículo proporciona un plan de pruebas general. Puedes usarlo para un archivo de documentos, un servidor multimedia, una gestión de fotos o cualquier otro servicio que obtenga mediante Rclone archivos poco necesarios desde un Cold Storage.

## Primero define qué quieres conseguir

Cold Storage no significa automáticamente lo mismo para cada aplicación. Un servidor multimedia suele leer archivos grandes de forma secuencial. Una gestión de fotos carga muchos archivos pequeños de vista previa y salta a distintas posiciones. Un archivo de documentos abre archivos relativamente pequeños, pero a menudo solo una vez.

Antes de la prueba, anota las características principales de tu conjunto real:

- tamaño típico de archivo y archivo más grande existente
- número de archivos por directorio
- lectura completa o accesos aleatorios a áreas individuales
- proporción entre accesos de lectura y escritura
- número de usuarios o procesos simultáneos
- cambios que se producen directamente en el remoto fuera del montaje
- tiempo de espera aceptable para una lectura en frío
- espacio máximo disponible para la caché local

Solo a partir de esto surgen criterios de éxito útiles. Abrir un único archivo en 1.2 segundos puede ser perfectamente aceptable para un archivo documental e inutilizable para una aplicación interactiva.

## Crear un conjunto de pruebas reproducible

Rclone ya incluye una herramienta adecuada para ello. `rclone test makefiles` genera siempre el mismo árbol de archivos con una semilla fija:

```bash
rclone test makefiles ./datos-prueba \
  --seed 42 \
  --files 250 \
  --files-per-directory 25 \
  --min-file-size 16K \
  --max-file-size 32M
```

Adapta la cantidad y los tamaños a tu conjunto real. No pruebes solo archivos promedio. Algunos archivos muy pequeños muestran cuánto cuestan los accesos a metadatos; algunos archivos grandes hacen visibles el rendimiento, la lectura anticipada y el comportamiento de la caché.

Añade además nombres de archivo que puedan causar problemas en la práctica:

```bash
mkdir -p "datos-prueba/Casos especiales/Subdirectorio"
printf 'Espacios\n' > "datos-prueba/Casos especiales/Archivo con espacios.txt"
printf 'Acentos\n' > "datos-prueba/Casos especiales/Tamaño y cambio.txt"
printf 'Mayúsculas\n' > "datos-prueba/Casos especiales/Prueba.txt"
printf 'Minúsculas\n' > "datos-prueba/Casos especiales/prueba.txt"
```

La última prueba es especialmente importante si el sistema de archivos local y el backend en la nube tratan las mayúsculas y minúsculas de forma diferente.

Si tu servicio solo acepta determinados formatos, no bastan archivos binarios arbitrarios. En ese caso, crea además archivos sintéticos en esos formatos exactos. En Paperless-ngx eran PDF con una capa de texto real, para que la prueba no midiera accidentalmente el rendimiento del OCR en lugar de la ruta de almacenamiento. Para una gestión de fotos, el conjunto debe incluir diferentes tamaños y formatos de imagen; para un servidor multimedia, archivos cortos con distintos códecs.

## Una medición de referencia sin montaje

Antes de que entren en juego FUSE y la caché VFS, deberías medir el backend directamente. Copia el conjunto al remoto de pruebas con Rclone y guarda un registro detallado:

```bash
rclone copy ./datos-prueba remote:cold-storage-test \
  --progress \
  --stats 5s \
  --log-file rclone-copy.log \
  --log-level INFO
```

Comprueba después si el origen y el destino coinciden:

```bash
rclone check ./datos-prueba remote:cold-storage-test \
  --one-way \
  --download
```

Con `--download` Rclone lee realmente los datos y los compara, incluso si el backend no proporciona hashes adecuados. Tarda más, pero ofrece una base útil para la prueba de integridad posterior.

Anota el tiempo de subida, la tasa de transferencia, el número de reintentos y los errores de API. Si el acceso directo ya es inestable, el montaje no puede arreglarlo.

## Separar el montaje de pruebas de la caché de producción

Para la medición, utiliza un punto de montaje propio y un directorio de caché independiente:

```bash
rclone mount remote:cold-storage-test /mnt/rclone-test \
  --vfs-cache-mode full \
  --cache-dir /var/cache/rclone-test \
  --vfs-cache-max-size 10G \
  --vfs-cache-poll-interval 1m \
  --allow-other \
  --log-file /var/log/rclone-test.log \
  --log-level INFO
```

Los valores son un ejemplo y no una recomendación general. Lo decisivo es la separación: una caché de pruebas vacía hace reproducibles las lecturas en frío sin que tengas que borrar archivos de una caché de producción en uso.

`--vfs-cache-mode full` suele ser el modo de prueba más revelador para las aplicaciones. Rclone almacena en búfer localmente los accesos de lectura y escritura y puede representar mejor accesos a archivos que no serían posibles con un almacenamiento de objetos puro. La compatibilidad adicional consume espacio local.

## Comprueba siempre desde la perspectiva del servicio real

Un montaje puede funcionar para tu usuario y, aun así, ser inutilizable para el servicio. Las causas frecuentes son un ID de usuario distinto, la falta de `--allow-other`, límites de contenedores o una propagación de montaje incorrecta.

Por ello, realiza al menos una lectura completa con la misma identidad con la que se ejecutará posteriormente la aplicación:

```bash
sudo -u <usuario-servicio> sha256sum /mnt/rclone-test/ruta/al/archivo
```

Si el servicio se ejecuta en Docker, la prueba debe hacerse dentro del contenedor:

```bash
docker exec --user <uid>:<gid> <contenedor-aplicacion> \
  sha256sum /ruta/en/el/contenedor/archivo
```

Aún mejor es una prueba real de la aplicación. Abre el archivo mediante la interfaz web o la API del servicio. Solo así notarás si la aplicación, por ejemplo, inicia varias lecturas paralelas, salta al final del archivo o espera metadatos adicionales.

## Medir por separado lecturas en frío y en caliente

Con `--vfs-cache-mode full` hay tres capas entre la aplicación y la nube:

| Capa | Qué contiene |
|---|---|
| Remoto | el archivo completo en el servicio en la nube |
| Caché VFS | áreas almacenadas localmente de archivos ya leídos |
| Caché de páginas de Linux | datos usados recientemente en la RAM |

Para una lectura en frío, elige un archivo cuyo contenido aún no se haya leído nunca mediante el montaje de pruebas. En la lectura en caliente inmediatamente posterior, se encuentra en la caché VFS y normalmente también en la RAM.

```bash
measure_read() {
  file="$1"
  label="$2"
  start=$(date +%s%3N)
  cat "$file" > /dev/null
  end=$(date +%s%3N)
  printf '%s: %s ms\n' "$label" "$((end - start))"
}

measure_read "/mnt/rclone-test/archivo-grande.bin" "Lectura en frío"
measure_read "/mnt/rclone-test/archivo-grande.bin" "Lectura en caliente"
```

No midas solo un archivo. Utiliza al menos diez archivos de distintos tamaños que aún no se hayan leído y anota la mediana, el valor más lento y el tamaño de archivo. Un único mejor resultado no es una base para decidir.

Una lectura en caliente no es una prueba pura de disco, porque el kernel puede mantener partes del archivo en la RAM. Para la mayoría de escenarios de Cold Storage, esto no es un problema. Lo importante es lo que experimenta un usuario al abrir por primera vez y repetidamente. Si quieres evaluar por separado la RAM y el disco local, debes controlar y vaciar de forma verificable la caché de páginas adicionalmente.

## No pruebes solo lecturas completas

`cat` lee un archivo de principio a fin. Muchas aplicaciones se comportan de otra forma:

- Un reproductor de vídeo lee primero las cabeceras y el índice, salta después a otra posición y continúa cargando secuencialmente.
- Una gestión de imágenes lee metadatos y luego genera una miniatura.
- Un programa de archivado puede leer primero el final del archivo.
- Varios workers pueden acceder simultáneamente a archivos distintos.

Prueba estos flujos con la aplicación real. Observa en paralelo el registro de Rclone y la caché. En archivos grandes, interesa saber cuánto almacena Rclone realmente de forma local y si `--vfs-read-ahead` se ajusta al patrón de acceso.

Además, un montaje de Rclone no es un lugar de almacenamiento adecuado para bases de datos u otros archivos que necesiten bloqueo fiable y cambios frecuentes dentro del mismo archivo. La capa VFS compensa diferencias entre el sistema de archivos y el almacenamiento de objetos, pero no convierte el backend en un sistema de archivos local.

## Validar por separado la ruta de escritura

Si tu servicio solo lee, monta el remoto en modo de solo lectura siempre que sea posible. Si debe escribir, prueba por separado la creación, sobrescritura, cambio de nombre y eliminación.

Un archivo escrito no aparece necesariamente de inmediato en el remoto. Con la caché VFS activa, la subida comienza solo después de cerrar el archivo y de que haya transcurrido `--vfs-write-back`. Por ello, comprueba ambos estados:

1. La aplicación ha cerrado correctamente el archivo.
2. Después, el archivo se puede leer en el remoto mediante acceso directo con Rclone.

```bash
printf 'prueba-de-writeback\n' > /mnt/rclone-test/prueba-de-writeback.txt

# Después de que transcurra --vfs-write-back:
rclone cat remote:cold-storage-test/prueba-de-writeback.txt
```

Repite la prueba con un archivo grande y finaliza Rclone mientras la subida todavía está en curso. Después reinicia con el mismo directorio de caché y comprueba si la subida continúa. Esta ventana temporal determina exactamente cuántos datos están en riesgo durante una caída del servidor.

Prueba también el cambio de nombre y la eliminación. Muchos backends en la nube representan estas operaciones de forma distinta a un sistema de archivos local. No solo importa si el comando termina correctamente, sino cuándo se hace visible el cambio con acceso directo al remoto y para otros clientes.

## Probar cambios fuera del montaje

Los archivos pueden modificarse a través de la interfaz web del proveedor, un segundo proceso de Rclone u otro servidor. El montaje no siempre ve estos cambios de inmediato, porque la información de directorios se almacena en caché.

Por ello, crea un archivo directamente en el remoto con una segunda llamada de Rclone:

```bash
printf 'cambio-externo\n' > cambio-externo.txt
rclone copyto cambio-externo.txt \
  remote:cold-storage-test/cambio-externo.txt
```

Mide cuándo aparece el archivo en el montaje. Repite la prueba para modificación y eliminación. El resultado depende del backend, de su compatibilidad con polling, así como de `--poll-interval` y `--dir-cache-time`. Si la aplicación debe ver los cambios actuales de inmediato, este comportamiento debe formar parte explícita de los criterios de aceptación.

Con la interfaz Remote Control activada, puedes descartar específicamente la caché de directorios:

```bash
rclone rc vfs/forget
```

Esto es útil para una prueba manual, pero no sustituye una estrategia operativa adecuada.

## Someter la caché a presión

Una caché casi vacía es el caso más sencillo. En una segunda ronda de pruebas, establece `--vfs-cache-max-size` deliberadamente bajo y lee más datos de los que caben en ella.

```bash
du -sh /var/cache/rclone-test/vfs
du -sh --apparent-size /var/cache/rclone-test/vfs
find /var/cache/rclone-test/vfs -type f | wc -l
```

Los dos tamaños pueden diferir mucho. En modo full, Rclone utiliza archivos dispersos: un archivo muestra su tamaño lógico completo aunque solo las áreas leídas ocupen espacio local.

Además, el límite de caché es flexible. Rclone lo comprueba al ritmo de `--vfs-cache-poll-interval`, y los archivos abiertos no se pueden eliminar. Por tanto, la caché puede superar brevemente el límite. Sin embargo, debería volver a reducirse tras cerrar los archivos y realizar la siguiente limpieza.

Registra el valor máximo, el valor después de la limpieza y el tiempo necesario. Así se puede dimensionar razonablemente el almacenamiento local necesario.

## Simular dos fallos distintos

Una nube inaccesible y un proceso de Rclone bloqueado son dos errores diferentes:

| Fallo | Qué compruebas con ello |
|---|---|
| Backend o red inaccesibles, Rclone sigue ejecutándose | comportamiento ante reintentos, tiempos de espera y archivos ya almacenados en caché |
| Proceso de Rclone finalizado | comportamiento del montaje FUSE y recuperación del punto de montaje |

Simula ambos solo en el entorno de pruebas. Puedes finalizar de forma forzada un contenedor de Rclone para el segundo caso:

```bash
docker kill --signal KILL <contenedor-rclone>
```

Durante el fallo, comprueba la aplicación y no solo el punto de montaje:

- ¿Qué funciones siguen disponibles?
- ¿Cuánto espera un acceso antes de que aparezca un error?
- ¿Los archivos ya almacenados completamente en caché siguen siendo accesibles?
- ¿La aplicación detiene nuevas operaciones de escritura?
- ¿Aparece un mensaje de error comprensible o solo un proceso bloqueado?
- ¿Se activa tu monitorización?

Un servicio de escritura no debe escribir inadvertidamente en el directorio local subyacente cuando falta el montaje. Tras el retorno del montaje, estos archivos quedarían ocultos. Una protección sencilla antes de cada tarea de escritura es:

```bash
mountpoint -q /mnt/rclone-test || exit 1
```

Tras reiniciar Rclone, comprueba el montaje en el host y desde cada contenedor consumidor. Un montaje reconstruido solo llega a un contenedor que ya está en ejecución con la propagación de montaje adecuada. Para Docker, normalmente se necesita `rslave` en el lado consumidor. Los detalles están en el artículo [Operar montajes de Rclone en Docker de forma fiable](/blog/rclone-mount-in-docker-container).

## Un ejemplo concreto de Paperless-ngx

Para mi prueba de Paperless, generé 40 PDF con un total de 13.9 MB. Un documento que aún no se había abierto necesitó alrededor de 1.8 segundos; un acceso repetido inmediatamente tardó de 19 a 24 milisegundos. Una caché VFS limitada a 4 MB subió temporalmente a 12.7 MiB y volvió a limpiarse en la siguiente ejecución.

Mientras el remoto no estuvo disponible, la lista de documentos, la búsqueda de texto completo y las miniaturas siguieron funcionando porque esos datos estaban almacenados localmente. Solo no se podía abrir el original. Tras reconstruir el montaje, el contenedor de Paperless en ejecución pudo volver a acceder a los archivos sin tener que reiniciarse.

Estas cifras no son un benchmark de Rclone ni de Proton Drive. Lo interesante es el comportamiento: el Hot Storage siguió disponible localmente, las lecturas en frío fueron más lentas pero previsibles y el servicio se recuperó tras el fallo.

## Qué debe incluir el protocolo de pruebas

Un resultado que se pueda entender posteriormente incluye como mínimo:

- versión de Rclone y backend utilizado
- sistema operativo, variante de FUSE y sistema de archivos del directorio de caché
- comando completo de montaje sin credenciales
- cantidad, distribución de tamaños y estructura de los archivos de prueba
- valores de lectura en frío y en caliente para varios archivos
- duración de escritura hasta la visibilidad en el remoto
- valor máximo de caché y duración de la limpieza
- resultado de `rclone check --download`
- comportamiento ante fallo del backend y proceso de Rclone finalizado
- tiempo de recuperación desde la perspectiva de la aplicación
- reintentos, tiempos de espera, limitaciones y errores de autenticación del registro

Define previamente un valor límite para cada punto. Así, la prueba termina con una decisión y no solo con una colección de cifras interesantes.

## Cuándo está listo el montaje

Un montaje de Cold Storage está listo para usarse si puedes responder sí a estas preguntas:

- ¿Las lecturas en frío son suficientemente rápidas para el servicio previsto?
- ¿La caché acelera los accesos repetidos como se espera?
- ¿El consumo de espacio local sigue siendo controlable incluso bajo carga?
- ¿Todos los archivos coinciden tras una descarga completa?
- ¿Funcionan todas las operaciones de archivo necesarias con el backend elegido?
- ¿La aplicación se comporta de forma controlada ante un fallo de la nube?
- ¿Las operaciones de escritura se detienen de forma segura cuando falta el montaje?
- ¿Un montaje reconstruido alcanza a todos los consumidores en ejecución?
- ¿La monitorización muestra el fallo antes de que lo comunique un usuario?

Si falta una respuesta, al menos sabes exactamente en qué debes seguir trabajando. Eso es mucho más útil que un montaje que parecía funcionar bien con el primer `ls` y solo muestra sus límites durante el funcionamiento.

## Fuentes

1.  [Rclone test makefiles](https://rclone.org/commands/rclone_test_makefiles/): archivos de prueba y estructuras de directorios reproducibles con tamaños configurables.

2.  [Rclone mount](https://rclone.org/commands/rclone_mount/): modos de caché VFS, writeback, archivos dispersos, límites de caché y caché de directorios.

3.  [Rclone check](https://rclone.org/commands/rclone_check/): comparación de origen y destino, incluida la comprobación completa con `--download`.

4.  [Rclone Remote Control](https://rclone.org/rc/): descarte específico de la caché de directorios VFS con `vfs/forget`.
