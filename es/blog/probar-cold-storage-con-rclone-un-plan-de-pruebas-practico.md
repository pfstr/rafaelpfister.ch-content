---
title: "Probar el almacenamiento en frío con Rclone: un plan de pruebas práctico"
navTitle: "Probar Rclone"
description: "Antes de que un servicio lea sus archivos de la nube mediante un montaje de Rclone, conviene comprobar más que el acceso al directorio. Este plan de pruebas cubre lecturas en frío, lecturas en caliente, operaciones de escritura, comportamiento de la caché, integridad de los archivos y fallos."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "11 min de lectura"
themen:
  - rclone
  - testing
related:
  - rclone-mount-in-docker-container
  - paperless-dokumente-clouddienst-auslagern
slug: "probar-cold-storage-con-rclone-un-plan-de-pruebas-practico"
translationOf: "cloud-mount-testen-dummy-pdfs"
translationId: article-8592f808b2e93cd4
translationModel: gpt-5.6-terra
translatedAt: 2026-10-31T03:27:23.039Z
translationReview: automatic
translationSourceHash: 820965b014837c0d0b5b5bf629a02842f2ebbbb26f62d343d8f1630278cd3bec
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/probar-cold-storage-con-rclone-un-plan-de-pruebas-practico
---

Un montaje de Rclone se configura rápidamente. El remoto aparece como un directorio, `ls` muestra archivos y la primera prueba funcional está superada. Para el funcionamiento en producción, eso todavía dice poco.

En cuanto un servicio accede al montaje, surgen más preguntas: ¿cuánto tarda el primer acceso a un archivo? ¿Qué accesos atiende la caché local? ¿Qué ocurre con un archivo que aún no se ha subido si Rclone se bloquea? ¿Un contenedor en ejecución vuelve a ver el montaje recreado? ¿Y cómo reacciona el servicio si la nube no está disponible temporalmente?

Este artículo proporciona un plan de pruebas general. Puede utilizarlo para un archivo de documentos, un servidor multimedia, una gestión de fotos o cualquier otro servicio que obtenga archivos poco necesarios mediante Rclone desde un almacenamiento en frío.

## Las opciones más importantes de rclone

Como orientación, a continuación se incluyen las opciones de Rclone que aparecen en este plan de pruebas, traducidas libremente de la documentación:

<details class="options-details">
<summary>Resumen de opciones</summary>

| Opción | Significado |
|---|---|
| `--seed n` | Valor inicial del generador de números aleatorios en `rclone test makefiles`; la misma semilla genera el mismo árbol de archivos |
| `--files n` | Número de archivos de prueba que se crearán |
| `--files-per-directory n` | Número medio de archivos por directorio |
| `--min-file-size grösse` | Tamaño mínimo de archivo generado (se permiten sufijos como K, M, G) |
| `--max-file-size grösse` | Tamaño máximo de archivo generado |
| `--progress` | Indicador de progreso durante la transferencia |
| `--stats dauer` | Intervalo en el que se emiten estadísticas de transferencia |
| `--log-file datei` | Escribe el registro en el archivo indicado |
| `--log-level stufe` | Nivel de detalle del registro: DEBUG, INFO, NOTICE o ERROR |
| `--one-way` | En `rclone check`, comprueba únicamente que todos los archivos fuente estén presentes e idénticos en el destino; los archivos adicionales en el destino no se consideran un error |
| `--download` | Descarga realmente los datos durante la comparación, en lugar de comparar solo hashes |
| `--vfs-cache-mode modus` | Estrategia de caché de la capa VFS; `full` almacena localmente en búfer los accesos de lectura y escritura |
| `--cache-dir verzeichnis` | Directorio de la caché local |
| `--vfs-cache-max-size grösse` | Límite flexible para el tamaño total de la caché VFS |
| `--vfs-cache-poll-interval dauer` | Intervalo en el que Rclone comprueba y limpia la caché |
| `--vfs-write-back dauer` | Tiempo de espera tras cerrar un archivo antes de que comience su subida al remoto |
| `--vfs-read-ahead grösse` | Cantidad adicional de datos que se lee por adelantado más allá de la posición solicitada con `full` |
| `--poll-interval dauer` | Intervalo con el que Rclone consulta cambios en el remoto (solo para backends compatibles con polling) |
| `--dir-cache-time dauer` | Período de validez de las listas de directorios almacenadas en caché |
| `--allow-other` | Permite que otros usuarios además del que monta accedan al montaje FUSE |

</details>

Las listas completas se encuentran en la documentación de Rclone, especialmente en [rclone mount](https://rclone.org/commands/rclone_mount/) y en la vista general de las [opciones globales](https://rclone.org/flags/).

## Primero defina lo que quiere conseguir

El almacenamiento en frío no significa automáticamente lo mismo para todas las aplicaciones. Un servidor multimedia suele leer archivos grandes de forma secuencial. Una gestión de fotos carga muchos datos de miniaturas pequeños y salta a distintas posiciones. Un archivo de documentos abre archivos relativamente pequeños, pero a menudo solo una vez.

Antes de la prueba, anote las características más importantes de su inventario real:

- tamaño típico de archivo y archivo más grande existente
- número de archivos por directorio
- lectura completa o accesos aleatorios a áreas concretas
- proporción entre accesos de lectura y escritura
- número de usuarios o procesos simultáneos
- cambios que se producen directamente en el remoto fuera del montaje
- tiempo de espera aceptable para una lectura en frío
- espacio máximo disponible para la caché local

Solo a partir de ello surgen criterios de éxito útiles. Abrir un único archivo en 1.2 segundos puede ser totalmente aceptable para un archivo y resultar inutilizable para una aplicación interactiva.

## Generar un conjunto de pruebas reproducible

Rclone ya incluye una herramienta adecuada para ello. `rclone test makefiles` genera siempre el mismo árbol de archivos con una semilla fija:

```bash
rclone test makefiles ./testdata \
  --seed 42 \
  --files 250 \
  --files-per-directory 25 \
  --min-file-size 16K \
  --max-file-size 32M
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `./testdata` | Directorio de destino en el que se crea el árbol de pruebas |
| `--seed 42` | Valor inicial fijo del generador de números aleatorios; cada ejecución genera el mismo conjunto |
| `--files 250` | 250 archivos de prueba en total |
| `--files-per-directory 25` | Una media de 25 archivos por directorio |
| `--min-file-size 16K` | Archivo mínimo de 16 KiB |
| `--max-file-size 32M` | Archivo máximo de 32 MiB |

</details>

Adapte la cantidad y los tamaños a su inventario real. No pruebe solo archivos medios. Algunos archivos muy pequeños muestran cuánto cuestan los accesos a metadatos; algunos archivos grandes revelan el rendimiento, la lectura anticipada y el comportamiento de la caché.

Añada también nombres de archivo que puedan causar problemas en la práctica:

```bash
mkdir -p "testdata/Sonderfälle/Unterordner"
printf 'Leerzeichen\n' > "testdata/Sonderfälle/Datei mit Leerzeichen.txt"
printf 'Umlaute\n' > "testdata/Sonderfälle/Grösse und Änderung.txt"
printf 'Grossschreibung\n' > "testdata/Sonderfälle/Test.txt"
printf 'Kleinschreibung\n' > "testdata/Sonderfälle/test.txt"
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `mkdir -p` | Crea también los directorios superiores que falten y no informa de error si el directorio existe |
| `printf '…\n' > datei` | Escribe el texto indicado como contenido del archivo; la redirección crea el archivo con el nombre problemático |

</details>

La última prueba es especialmente importante si el sistema de archivos local y el backend en la nube tratan de forma distinta las mayúsculas y minúsculas.

Si su servicio solo acepta determinados formatos, los archivos binarios arbitrarios no bastan. En ese caso, genere también archivos sintéticos exactamente en esos formatos. En Paperless-ngx, eran PDF con una capa de texto real, para que la prueba no midiera por accidente el rendimiento de OCR en lugar de la ruta de almacenamiento. En una gestión de fotos, el conjunto debe incluir distintos tamaños y formatos de imagen; en un servidor multimedia, archivos cortos con diferentes códecs.

## Una medición de referencia sin montaje

Antes de que entren en juego FUSE y la caché VFS, debe medir directamente el backend. Copie el conjunto con Rclone al remoto de prueba y guarde un registro detallado:

```bash
rclone copy ./testdata remote:cold-storage-test \
  --progress \
  --stats 5s \
  --log-file rclone-copy.log \
  --log-level INFO
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `./testdata` | Fuente de la copia: el conjunto de pruebas generado localmente |
| `remote:cold-storage-test` | Destino: ruta en el remoto configurado |
| `--progress` | Indicador de progreso en el terminal |
| `--stats 5s` | Estadísticas de transferencia cada 5 segundos en lugar del intervalo predeterminado |
| `--log-file rclone-copy.log` | Registro completo en un archivo para su evaluación posterior |
| `--log-level INFO` | Registra transferencias, reintentos y errores, sin el volumen de DEBUG |

</details>

Compruebe después si la fuente y el destino coinciden:

```bash
rclone check ./testdata remote:cold-storage-test \
  --one-way \
  --download
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `./testdata` | Referencia: el conjunto original local |
| `remote:cold-storage-test` | Objeto de comprobación: el conjunto recién subido en el remoto |
| `--one-way` | Comprueba solo que todos los archivos fuente estén presentes e idénticos en el destino; no señala los archivos adicionales en el destino |
| `--download` | Descarga los datos y compara los contenidos, en lugar de depender de hashes |

</details>

`--download` es decisivo aquí porque algunos backends no proporcionan hashes adecuados. La comparación tarda más, pero proporciona una base útil para la prueba de integridad posterior.

Registre el tiempo de subida, la velocidad de transferencia, el número de reintentos y los errores de API. Si el acceso directo ya es inestable, el montaje no podrá repararlo.

## Separar el montaje de prueba de la caché de producción

Utilice un punto de montaje y un directorio de caché propios para la medición:

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

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `remote:cold-storage-test` | Remoto y ruta que se montarán |
| `/mnt/rclone-test` | Punto de montaje en el sistema de pruebas |
| `--vfs-cache-mode full` | Almacena completamente en búfer los accesos de lectura y escritura en la caché local |
| `--cache-dir /var/cache/rclone-test` | Directorio de caché propio, separado de la caché de producción |
| `--vfs-cache-max-size 10G` | Límite flexible de 10 GiB para la caché VFS |
| `--vfs-cache-poll-interval 1m` | Comprobación y limpieza de la caché cada minuto |
| `--allow-other` | También pueden acceder otros usuarios además del que monta; necesario para servicios y contenedores |
| `--log-file /var/log/rclone-test.log` | Registro en un archivo para seguir los accesos durante las pruebas |
| `--log-level INFO` | Nivel medio de detalle del registro |

</details>

Los valores son un ejemplo, no una recomendación general. Lo decisivo es la separación: una caché de prueba vacía hace reproducibles las lecturas en frío sin tener que eliminar archivos de una caché de producción en uso.

`--vfs-cache-mode full` suele ser el modo de prueba más revelador para las aplicaciones. Rclone almacena localmente en búfer los accesos de lectura y escritura y puede representar mejor los accesos a archivos que no serían posibles con un almacenamiento de objetos puro. La compatibilidad adicional consume espacio de almacenamiento local.

## Compruebe siempre desde la perspectiva del servicio real

Un montaje puede funcionar para su usuario y aun así ser inutilizable para el servicio. Las causas frecuentes son un ID de usuario distinto, la ausencia de `--allow-other`, los límites de los contenedores o una propagación de montaje incorrecta.

Por ello, realice al menos una lectura completa con la misma identidad con la que se ejecutará posteriormente la aplicación:

```bash
sudo -u <service-user> sha256sum /mnt/rclone-test/pfad/zur/datei
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-u <service-user>` | Ejecuta el comando como el usuario indicado, no como root |
| `/mnt/rclone-test/pfad/zur/datei` | Archivo que se leerá; `sha256sum` fuerza una lectura completa |

</details>

Si el servicio se ejecuta en Docker, la prueba debe hacerse en el contenedor:

```bash
docker exec --user <uid>:<gid> <app-container> \
  sha256sum /pfad/im/container/datei
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `--user <uid>:<gid>` | Ejecuta el comando en el contenedor con este ID de usuario y grupo, independientemente del usuario predeterminado de la imagen |
| `<app-container>` | Nombre o ID del contenedor de la aplicación en ejecución |
| `sha256sum /pfad/im/container/datei` | Comando que se ejecutará; la ruta es el montaje desde la perspectiva del contenedor |

</details>

Mejor aún es una prueba real de la aplicación. Abra el archivo mediante la interfaz web o la API del servicio. Solo así advertirá, por ejemplo, si la aplicación inicia varias lecturas paralelas, salta al final del archivo o espera metadatos adicionales.

## Medir por separado lecturas en frío y en caliente

Con `--vfs-cache-mode full`, hay tres capas entre la aplicación y la nube:

| Capa | Qué contiene |
|---|---|
| Remoto | el archivo completo en el servicio en la nube |
| Caché VFS | áreas almacenadas localmente de archivos ya leídos |
| Caché de páginas de Linux | datos usados recientemente en la RAM |

Para una lectura en frío, elija un archivo cuyo contenido nunca se haya leído a través del montaje de prueba. En la lectura en caliente inmediatamente posterior, estará en la caché VFS y normalmente también en la RAM.

```bash
measure_read() {
  file="$1"
  label="$2"
  start=$(date +%s%3N)
  cat "$file" > /dev/null
  end=$(date +%s%3N)
  printf '%s: %s ms\n' "$label" "$((end - start))"
}

measure_read "/mnt/rclone-test/grosse-datei.bin" "Cold Read"
measure_read "/mnt/rclone-test/grosse-datei.bin" "Warm Read"
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `date +%s%3N` | Marca temporal en milisegundos: segundos Unix seguidos directamente de los tres primeros dígitos de la fracción de nanosegundos (GNU date) |
| `cat "$file" > /dev/null` | Lee completamente el archivo y descarta la salida; solo se mide el tiempo de lectura |
| `"$1"`, `"$2"` | Argumentos de la función shell: ruta del archivo y etiqueta de la línea de medición |

</details>

No mida solo un archivo. Utilice al menos diez archivos de distintos tamaños que todavía no se hayan leído y anote la mediana, el valor más lento y el tamaño del archivo. Un único mejor resultado no sirve como base para una decisión.

Una lectura en caliente no es una prueba pura del disco, porque el kernel puede conservar partes del archivo en la RAM. Para la mayoría de escenarios de almacenamiento en frío, esto no supone un problema. Lo decisivo es lo que experimenta un usuario al abrir por primera vez y repetidamente. Si desea evaluar por separado la RAM y el disco local, debe controlar adicionalmente la caché de páginas y vaciarla de forma verificable.

## No pruebe solo lecturas completas

`cat` lee un archivo de principio a fin. Muchas aplicaciones se comportan de otra forma:

- Un reproductor de vídeo lee primero las cabeceras y el índice, después salta a otra posición y continúa leyendo secuencialmente.
- Una gestión de imágenes lee metadatos y genera después una miniatura.
- Un programa de archivado puede leer primero el final del archivo.
- Varios workers pueden acceder simultáneamente a archivos distintos.

Pruebe estos flujos con la aplicación real. Observe en paralelo el registro de Rclone y la caché. En archivos grandes, interesa saber cuánto almacena realmente Rclone de forma local y si `--vfs-read-ahead` se adapta al patrón de acceso.

Además, un montaje de Rclone no es un lugar de almacenamiento adecuado para bases de datos u otros archivos que requieran bloqueos fiables y cambios frecuentes dentro del mismo archivo. La capa VFS compensa las diferencias entre el sistema de archivos y el almacenamiento de objetos, pero no convierte el backend en un sistema de archivos local.

## Valide por separado la ruta de escritura

Si su servicio solo lee, monte el remoto en modo de solo lectura siempre que sea posible. Si debe escribir, pruebe por separado la creación, sobrescritura, cambio de nombre y eliminación.

Un archivo escrito no aparece necesariamente de inmediato en el remoto. Con la caché VFS activa, la subida comienza solo después de cerrar el archivo y de que haya transcurrido `--vfs-write-back`. Por ello, compruebe ambos estados:

1. La aplicación ha cerrado correctamente el archivo.
2. Posteriormente, el archivo se puede leer en el remoto mediante acceso directo con Rclone.

```bash
printf 'writeback-test\n' > /mnt/rclone-test/writeback-test.txt

# Después de que transcurra --vfs-write-back:
rclone cat remote:cold-storage-test/writeback-test.txt
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `/mnt/rclone-test/writeback-test.txt` | Archivo de destino en el montaje; la redirección escribe mediante la caché VFS |
| `remote:cold-storage-test/writeback-test.txt` | Acceso directo sin pasar por el montaje: `rclone cat` lee el archivo desde el remoto y lo envía a stdout |

</details>

Repita la prueba con un archivo grande y finalice Rclone mientras la subida aún está en curso. Reinicie después con el mismo directorio de caché y compruebe si la subida continúa. Esta ventana temporal determina exactamente cuántos datos están en riesgo ante un fallo del servidor.

Pruebe también el cambio de nombre y la eliminación. Muchos backends en la nube representan estas operaciones de forma distinta a un sistema de archivos local. No solo importa si el comando termina correctamente, sino cuándo el cambio resulta visible mediante acceso directo al remoto y para otros clientes.

## Probar cambios fuera del montaje

Los archivos pueden modificarse mediante la interfaz web del proveedor, un segundo proceso de Rclone u otro servidor. El montaje no siempre ve estos cambios de inmediato, porque la información de los directorios se almacena en caché.

Por ello, cree un archivo directamente en el remoto con una segunda llamada a Rclone:

```bash
printf 'external-change\n' > external-change.txt
rclone copyto external-change.txt \
  remote:cold-storage-test/external-change.txt
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `external-change.txt` | Fuente: el archivo generado localmente |
| `remote:cold-storage-test/external-change.txt` | Destino con el nombre de archivo exacto; `copyto` copia un solo archivo con exactamente ese nombre, en lugar de hacerlo como `copy` en un directorio |

</details>

Mida cuándo aparece el archivo en el montaje. Repita la prueba para modificaciones y eliminaciones. El resultado depende del backend, de su compatibilidad con polling, así como de `--poll-interval` y `--dir-cache-time`. Si la aplicación debe ver los cambios actuales de inmediato, este comportamiento debe incluirse explícitamente en los criterios de aceptación.

Con la interfaz Remote Control activada, puede invalidar específicamente la caché de directorios:

```bash
rclone rc vfs/forget
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `vfs/forget` | Comando Remote Control que se ejecutará: descarta el árbol de directorios VFS almacenado en caché, de modo que el siguiente acceso vuelve a consultar el remoto |

</details>

Esto resulta útil para una prueba manual, pero no sustituye una estrategia operativa adecuada.

## Someter la caché a presión

Una caché casi vacía es el caso más sencillo. En una segunda ronda de pruebas, configure deliberadamente `--vfs-cache-max-size` con un valor pequeño y lea más datos de los que caben en ella.

```bash
du -sh /var/cache/rclone-test/vfs
du -sh --apparent-size /var/cache/rclone-test/vfs
find /var/cache/rclone-test/vfs -type f | wc -l
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `du -s` | Resume el uso de espacio en una sola línea, en lugar de listar cada subdirectorio |
| `du -h` | Salida en unidades legibles para humanos (K, M, G) |
| `du --apparent-size` | Muestra el tamaño lógico del archivo en lugar del espacio realmente ocupado |
| `find … -type f` | Solo considera archivos regulares, no directorios |
| `wc -l` | Cuenta las líneas de salida, en este caso el número de archivos en caché |

</details>

Los dos tamaños pueden diferir mucho. En modo Full, Rclone utiliza archivos dispersos: un archivo muestra su tamaño lógico completo aunque solo las áreas leídas ocupen espacio local.

Además, el límite de la caché es flexible. Rclone lo comprueba al ritmo de `--vfs-cache-poll-interval`, y los archivos abiertos no pueden eliminarse. Por eso, la caché puede superar brevemente el límite. Sin embargo, debería volver a reducirse tras cerrar los archivos y ejecutar la siguiente limpieza.

Registre el valor máximo, el valor tras la limpieza y el tiempo necesario. Así se puede dimensionar razonablemente el almacenamiento local necesario.

## Simular dos fallos distintos

Una nube inaccesible y un proceso de Rclone bloqueado son dos errores diferentes:

| Fallo | Qué comprueba |
|---|---|
| Backend o red inaccesibles, Rclone sigue ejecutándose | Comportamiento ante reintentos, tiempos de espera y archivos ya almacenados en caché |
| Proceso de Rclone finalizado | Comportamiento del montaje FUSE y recuperación del punto de montaje |

Simule ambos solo en el entorno de pruebas. Puede finalizar de forma forzada un contenedor de Rclone para el segundo caso:

```bash
docker kill --signal KILL <rclone-container>
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `--signal KILL` | Envía SIGKILL (que ya es el valor predeterminado en `docker kill`; aquí se indica para aclararlo); el proceso no tiene oportunidad de realizar tareas de limpieza |
| `<rclone-container>` | Nombre o ID del contenedor de Rclone |

</details>

Durante el fallo, compruebe la aplicación y no solo el punto de montaje:

- ¿Qué funciones permanecen disponibles?
- ¿Cuánto espera un acceso antes de que aparezca un error?
- ¿Siguen siendo accesibles los archivos ya completamente almacenados en caché?
- ¿La aplicación detiene las nuevas operaciones de escritura?
- ¿Aparece un mensaje de error comprensible o solo un proceso bloqueado?
- ¿Se activa su monitorización?

Un servicio de escritura no debe escribir inadvertidamente en el directorio local subyacente cuando falta el montaje. Tras el regreso del montaje, estos archivos quedarían ocultos. Una protección sencilla antes de cada tarea de escritura es:

```bash
mountpoint -q /mnt/rclone-test || exit 1
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-q` | Sin salida; el resultado solo figura en el código de salida |
| `/mnt/rclone-test` | Ruta que se comprobará; código de salida 0 solo si allí hay realmente un montaje activo |
| `\|\| exit 1` | Interrumpe el script si la ruta no es un punto de montaje |

</details>

Después de reiniciar Rclone, compruebe el montaje en el host y desde cada contenedor consumidor. Un montaje recreado solo llega a un contenedor que ya está en ejecución con una propagación de montaje adecuada. Para Docker, normalmente se necesita `rslave` en el lado consumidor. Los detalles se encuentran en el artículo [Operar montajes de Rclone en Docker de forma fiable](/blog/rclone-mount-in-docker-container).

## Un ejemplo concreto de Paperless-ngx

Para mi prueba de Paperless, generé 40 PDF con un total de 13.9 MB. Un documento que no se había abierto antes tardó unos 1.8 segundos; un acceso repetido inmediatamente tardó de 19 a 24 milisegundos. Una caché VFS limitada a 4 MB aumentó brevemente a 12.7 MiB y se limpió de nuevo en la siguiente ejecución.

Mientras el remoto no estaba disponible, la lista de documentos, la búsqueda de texto completo y las miniaturas siguieron funcionando porque esos datos eran locales. Solo el original no se podía abrir. Tras recrear el montaje, el contenedor Paperless en ejecución pudo volver a acceder a los archivos sin necesidad de reiniciarse.

Estas cifras no son un benchmark para Rclone ni Proton Drive. Lo interesante es el comportamiento: el almacenamiento en caliente siguió disponible localmente, las lecturas en frío eran más lentas pero predecibles, y el servicio se recuperó tras el fallo.

## Qué debe incluir el protocolo de pruebas

Un resultado que pueda revisarse posteriormente incluye como mínimo:

- versión de Rclone y backend utilizado
- sistema operativo, variante de FUSE y sistema de archivos del directorio de caché
- comando de montaje completo sin credenciales
- número, distribución de tamaños y estructura de los archivos de prueba
- valores de lectura en frío y en caliente para varios archivos
- duración de escritura hasta que sea visible en el remoto
- valor máximo de caché y duración de la limpieza
- resultado de `rclone check --download`
- comportamiento ante fallo del backend y finalización del proceso de Rclone
- tiempo de recuperación desde la perspectiva de la aplicación
- reintentos, tiempos de espera, limitaciones y errores de autenticación del registro

Defina de antemano un valor límite para cada punto. Así, la prueba termina con una decisión y no solo con una recopilación de cifras interesantes.

## Cuándo está listo el montaje

Un montaje de almacenamiento en frío está listo para usarse cuando puede responder afirmativamente a estas preguntas:

- ¿Las lecturas en frío son suficientemente rápidas para el servicio previsto?
- ¿La caché acelera los accesos repetidos como se esperaba?
- ¿El consumo de espacio local sigue siendo controlable incluso bajo carga?
- ¿Coinciden todos los archivos tras una descarga completa?
- ¿Funcionan todas las operaciones de archivo necesarias con el backend elegido?
- ¿La aplicación se comporta de forma controlada ante un fallo de la nube?
- ¿Las operaciones de escritura se detienen de forma segura si falta el montaje?
- ¿Un montaje recreado llega a todos los consumidores en ejecución?
- ¿La monitorización muestra el fallo antes de que lo informe un usuario?

Si falta una respuesta, al menos sabrá exactamente en qué debe seguir trabajando. Eso es mucho más útil que un montaje que parecía correcto al primer `ls` y solo muestra sus límites en funcionamiento.

## Historial de cambios

- **25 de octubre de 2026:** Corregido: en docker kill, SIGKILL ya es la señal predeterminada; la opción --signal KILL sirve únicamente para aclararlo.

## Fuentes

1.  [Rclone test makefiles](https://rclone.org/commands/rclone_test_makefiles/): archivos de prueba y estructuras de directorios reproducibles con tamaños configurables.

2.  [Rclone mount](https://rclone.org/commands/rclone_mount/): modos de caché VFS, writeback, archivos dispersos, límites de caché y caché de directorios.

3.  [Rclone check](https://rclone.org/commands/rclone_check/): comparación de fuente y destino, incluida la comprobación completa con `--download`.

4.  [Rclone Remote Control](https://rclone.org/rc/): invalidación selectiva de la caché de directorios VFS con `vfs/forget`.

5.  [Rclone Global Flags](https://rclone.org/flags/): referencia completa de las opciones globales, incluidos el registro, las estadísticas y los parámetros VFS.
