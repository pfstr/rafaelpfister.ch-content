---
title: "Probar el almacenamiento en frío con Rclone: un plan de pruebas práctico"
navTitle: "Probar Rclone"
description: "Antes de que un servicio lea sus archivos de la nube mediante un montaje de Rclone, conviene comprobar algo más que el acceso a directorios. Este plan de pruebas abarca lecturas en frío, lecturas en caliente, operaciones de escritura, comportamiento de la caché, integridad de los archivos y fallos."
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
translatedAt: 2026-09-03T08:21:28.421Z
translationReview: automatic
translationSourceHash: 27bc45a50d8e84fc785eaf04ec6814054e327f516587d0f9f9a101c989ce2aa1
url: https://rafaelpfister.ch/es/blog/probar-cold-storage-con-rclone-un-plan-de-pruebas-practico
---

Un montaje de Rclone se configura rápidamente. El remoto aparece como un directorio, `ls` muestra archivos y la primera prueba funcional se ha superado. Sin embargo, eso dice poco sobre el funcionamiento en producción.

En cuanto un servicio accede al montaje, surgen más preguntas: ¿cuánto tarda el primer acceso a un archivo? ¿Qué accesos atiende la caché local? ¿Qué ocurre con un archivo que aún no se ha subido si Rclone falla? ¿Un contenedor en ejecución vuelve a ver el montaje reconstruido? ¿Y cómo reacciona el servicio si la nube deja de estar disponible temporalmente?

Este artículo ofrece un plan de pruebas general para ello. Puede utilizarlo para un archivo de documentos, un servidor multimedia, una gestión de fotos o cualquier otro servicio que obtenga archivos poco necesarios mediante Rclone desde un almacenamiento en frío.

## Las opciones más importantes de rclone

Como orientación, a continuación se muestran las opciones de Rclone que aparecen en este plan de pruebas, traducidas libremente de la documentación:

<details class="options-details">
<summary>Resumen de opciones</summary>

| Opción | Significado |
|---|---|
| `--seed n` | Valor inicial del generador aleatorio en `rclone test makefiles`; la misma semilla genera el mismo árbol de archivos |
| `--files n` | Número de archivos de prueba que se generarán |
| `--files-per-directory n` | Número medio de archivos por directorio |
| `--min-file-size grösse` | Tamaño mínimo de archivo generado (se permiten sufijos como K, M, G) |
| `--max-file-size grösse` | Tamaño máximo de archivo generado |
| `--progress` | Indicador de progreso durante la transferencia |
| `--stats dauer` | Intervalo en el que se muestran estadísticas de transferencia |
| `--log-file datei` | Escribe el registro en el archivo especificado |
| `--log-level stufe` | Nivel de detalle del registro: DEBUG, INFO, NOTICE o ERROR |
| `--one-way` | En `rclone check`, solo comprueba que todos los archivos de origen existan en el destino y sean idénticos; los archivos adicionales en el destino no se consideran un error |
| `--download` | Descarga los datos realmente durante la comparación, en lugar de limitarse a comparar hashes |
| `--vfs-cache-mode modus` | Estrategia de caché de la capa VFS; `full` almacena en búfer localmente los accesos de lectura y escritura |
| `--cache-dir verzeichnis` | Directorio para la caché local |
| `--vfs-cache-max-size grösse` | Límite flexible para el tamaño total de la caché VFS |
| `--vfs-cache-poll-interval dauer` | Intervalo con el que Rclone comprueba y limpia la caché |
| `--vfs-write-back dauer` | Tiempo de espera tras cerrar un archivo antes de iniciar la subida al remoto |
| `--vfs-read-ahead grösse` | Cantidad adicional de datos que se lee por adelantado más allá de la posición solicitada con `full` |
| `--poll-interval dauer` | Intervalo con el que Rclone consulta cambios en el remoto (solo en backends con compatibilidad con polling) |
| `--dir-cache-time dauer` | Tiempo de validez de las listas de directorios almacenadas en caché |
| `--allow-other` | Permite que usuarios distintos del que realiza el montaje accedan al montaje FUSE |

</details>

Las listas completas se encuentran en la documentación de Rclone, especialmente en [rclone mount](https://rclone.org/commands/rclone_mount/) y en la descripción general de las [global flags](https://rclone.org/flags/).

## Primero defina lo que quiere conseguir

El almacenamiento en frío no significa automáticamente lo mismo para todas las aplicaciones. Un servidor multimedia suele leer archivos grandes de forma secuencial. Una gestión de fotos carga muchas vistas previas pequeñas y salta a distintas posiciones. Un archivo de documentos abre archivos relativamente pequeños, pero a menudo solo una vez.

Antes de la prueba, anote las características más importantes de su conjunto de datos real:

- tamaño típico de archivo y archivo más grande existente
- número de archivos por directorio
- lectura completa o accesos aleatorios a áreas individuales
- proporción entre accesos de lectura y escritura
- número de usuarios o procesos simultáneos
- cambios realizados directamente en el remoto fuera del montaje
- tiempo de espera aceptable para una lectura en frío
- espacio máximo disponible para la caché local

Solo a partir de ahí surgen criterios de éxito útiles. Abrir un único archivo en 1.2 segundos puede ser perfectamente aceptable para un archivo y resultar inutilizable para una aplicación interactiva.

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
| `./testdata` | Directorio de destino en el que se crea el árbol de prueba |
| `--seed 42` | Valor inicial fijo del generador aleatorio; cada ejecución crea el mismo conjunto |
| `--files 250` | 250 archivos de prueba en total |
| `--files-per-directory 25` | Una media de 25 archivos por directorio |
| `--min-file-size 16K` | Archivo mínimo de 16 KiB |
| `--max-file-size 32M` | Archivo máximo de 32 MiB |

</details>

Adapte el número y los tamaños a su conjunto de datos real. No pruebe solo archivos medios. Algunos archivos muy pequeños muestran el coste de los accesos a metadatos; algunos archivos grandes hacen visible el rendimiento, la lectura anticipada y el comportamiento de la caché.

Además, añada nombres de archivo que puedan causar problemas en la práctica:

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
| `mkdir -p` | También crea directorios superiores inexistentes y no informa de error si el directorio ya existe |
| `printf '…\n' > datei` | Escribe el texto indicado como contenido del archivo; la redirección crea el archivo con el nombre problemático |

</details>

La última prueba es especialmente importante si el sistema de archivos local y el backend de nube tratan de forma distinta las mayúsculas y minúsculas.

Si su servicio solo acepta determinados formatos, los archivos binarios arbitrarios no bastan. En ese caso, genere también archivos sintéticos precisamente en esos formatos. En Paperless-ngx, se trataba de PDF con una capa de texto real, para que la prueba no midiera por error el rendimiento del OCR en lugar de la ruta de almacenamiento. En una gestión de fotos, el conjunto debe incluir distintos tamaños y formatos de imagen; en un servidor multimedia, archivos cortos con distintos códecs.

## Una medición de referencia sin montaje

Antes de que entren en juego FUSE y la caché VFS, debe medir el backend directamente. Copie el conjunto al remoto de prueba con Rclone y guarde un registro detallado:

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
| `./testdata` | Origen de la copia: el conjunto de pruebas generado localmente |
| `remote:cold-storage-test` | Destino: ruta en el remoto configurado |
| `--progress` | Indicador de progreso en ejecución en el terminal |
| `--stats 5s` | Estadísticas de transferencia cada 5 segundos en lugar del intervalo predeterminado |
| `--log-file rclone-copy.log` | Registro completo en un archivo para su análisis posterior |
| `--log-level INFO` | Registra transferencias, reintentos y errores sin el volumen de DEBUG |

</details>

A continuación, compruebe que el origen y el destino coinciden:

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
| `remote:cold-storage-test` | Elemento comprobado: el conjunto recién subido al remoto |
| `--one-way` | Solo comprueba que todos los archivos de origen existan en el destino y sean idénticos; no señala como problema los archivos adicionales en el destino |
| `--download` | Descarga los datos y compara el contenido, en lugar de confiar en hashes |

</details>

`--download` es decisivo aquí porque algunos backends no proporcionan hashes adecuados. La comparación tarda más, pero proporciona una base útil para la posterior prueba de integridad.

Registre el tiempo de subida, la velocidad de transferencia, el número de reintentos y los errores de API. Si el acceso directo ya es inestable, el montaje no puede arreglarlo.

## Separar el montaje de prueba de la caché de producción

Para las mediciones, utilice un punto de montaje y un directorio de caché propios:

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
| `/mnt/rclone-test` | Punto de montaje en el sistema de prueba |
| `--vfs-cache-mode full` | Almacena completamente en búfer los accesos de lectura y escritura en la caché local |
| `--cache-dir /var/cache/rclone-test` | Directorio de caché propio, separado de la caché de producción |
| `--vfs-cache-max-size 10G` | Límite flexible de 10 GiB para la caché VFS |
| `--vfs-cache-poll-interval 1m` | Comprobación y limpieza de la caché cada minuto |
| `--allow-other` | También pueden acceder otros usuarios distintos del que realiza el montaje; necesario para servicios y contenedores |
| `--log-file /var/log/rclone-test.log` | Registro en un archivo para seguir los accesos durante las pruebas |
| `--log-level INFO` | Nivel medio de detalle del registro |

</details>

Los valores son un ejemplo, no una recomendación general. Lo decisivo es la separación: una caché de prueba vacía hace reproducibles las lecturas en frío, sin tener que eliminar archivos de una caché de producción activa.

`--vfs-cache-mode full` suele ser el modo de prueba más revelador para las aplicaciones. Rclone almacena localmente en búfer los accesos de lectura y escritura, y puede representar mejor los accesos a archivos que no serían posibles con un almacenamiento de objetos puro. Esa compatibilidad adicional consume espacio de almacenamiento local.

## Compruebe siempre desde la perspectiva del servicio real

Un montaje puede funcionar para su usuario y, aun así, ser inutilizable para el servicio. Entre las causas frecuentes se encuentran otro ID de usuario, la ausencia de `--allow-other`, límites de contenedores o una propagación de montaje incorrecta.

Por tanto, realice al menos una lectura completa con la misma identidad con la que se ejecutará después la aplicación:

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

Si el servicio se ejecuta en Docker, la prueba debe realizarse dentro del contenedor:

```bash
docker exec --user <uid>:<gid> <app-container> \
  sha256sum /pfad/im/container/datei
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `--user <uid>:<gid>` | Ejecuta el comando en el contenedor con este ID de usuario y grupo, independientemente del usuario predeterminado de la imagen |
| `<app-container>` | Nombre o ID del contenedor de aplicación en ejecución |
| `sha256sum /pfad/im/container/datei` | Comando que se ejecutará; la ruta es el montaje visto desde el contenedor |

</details>

Aún mejor es una prueba real de la aplicación. Abra el archivo mediante la interfaz web o la API del servicio. Solo así detectará si la aplicación, por ejemplo, inicia varias lecturas en paralelo, salta al final del archivo o espera metadatos adicionales.

## Medir por separado las lecturas en frío y en caliente

Con `--vfs-cache-mode full`, hay tres capas entre la aplicación y la nube:

| Capa | Qué contiene |
|---|---|
| Remoto | el archivo completo en el servicio en la nube |
| Caché VFS | áreas almacenadas localmente de archivos ya leídos |
| Caché de páginas de Linux | datos usados recientemente en la RAM |

Para una lectura en frío, elija un archivo cuyo contenido no se haya leído nunca mediante el montaje de prueba. En la lectura en caliente inmediatamente posterior, estará en la caché VFS y, por lo general, también en la RAM.

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
| `date +%s%3N` | Marca de tiempo en milisegundos: segundos Unix seguidos directamente por las tres primeras cifras de la parte de nanosegundos (GNU date) |
| `cat "$file" > /dev/null` | Lee el archivo completo y descarta la salida; solo se mide el tiempo de lectura |
| `"$1"`, `"$2"` | Argumentos de la función de shell: ruta de archivo y etiqueta de la línea de medición |

</details>

No mida solo un archivo. Utilice al menos diez archivos de distintos tamaños que aún no se hayan leído y anote la mediana, el valor más lento y el tamaño del archivo. Un único mejor resultado no es base suficiente para tomar una decisión.

Una lectura en caliente no es una prueba pura de disco, porque el kernel puede conservar partes del archivo en la RAM. Para la mayoría de escenarios de almacenamiento en frío, esto no supone un problema. Lo decisivo es lo que experimenta un usuario al abrir un archivo por primera vez y al volver a abrirlo. Si desea evaluar por separado la RAM y el disco local, debe controlar y vaciar de forma verificable la caché de páginas.

## No pruebe solo lecturas completas

`cat` lee un archivo de principio a fin. Muchas aplicaciones se comportan de otro modo:

- Un reproductor de vídeo lee primero la cabecera y el índice, después salta a otra posición y continúa cargando secuencialmente.
- Una gestión de imágenes lee metadatos y luego genera una vista previa.
- Un programa de archivado puede leer primero el final del archivo.
- Varios workers pueden acceder simultáneamente a archivos diferentes.

Pruebe estos flujos con la aplicación real. Observe en paralelo el registro de Rclone y la caché. Con archivos grandes, resulta interesante cuánto almacena realmente Rclone de forma local y si `--vfs-read-ahead` se ajusta al patrón de acceso.

Además, un montaje de Rclone no es un lugar de almacenamiento adecuado para bases de datos u otros archivos que requieren bloqueos fiables y cambios frecuentes dentro del mismo archivo. La capa VFS compensa las diferencias entre el sistema de archivos y el almacenamiento de objetos, pero no convierte el backend en un sistema de archivos local.

## Valide por separado la ruta de escritura

Si su servicio solo lee, monte el remoto en modo de solo lectura siempre que sea posible. Si debe escribir, pruebe por separado la creación, sobrescritura, cambio de nombre y eliminación.

Un archivo escrito no aparece necesariamente de inmediato en el remoto. Con la caché VFS activa, la subida no empieza hasta que se cierra el archivo y transcurre `--vfs-write-back`. Por ello, compruebe ambos estados:

1. La aplicación ha cerrado correctamente el archivo.
2. Posteriormente, se puede leer el archivo en el remoto mediante acceso directo con Rclone.

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

Repita la prueba con un archivo grande y detenga Rclone mientras la subida aún está en curso. A continuación, reinícielo con el mismo directorio de caché y compruebe si la subida se reanuda. Precisamente este intervalo determina cuántos datos están en riesgo ante un fallo del servidor.

Pruebe también el cambio de nombre y la eliminación. Muchos backends de nube representan estas operaciones de forma distinta a un sistema de archivos local. No solo importa que el comando finalice correctamente, sino cuándo se hace visible el cambio mediante un acceso directo al remoto y para otros clientes.

## Probar cambios fuera del montaje

Los archivos pueden modificarse mediante la interfaz web del proveedor, un segundo proceso de Rclone u otro servidor. El montaje no siempre ve estos cambios de inmediato porque la información de directorios se almacena en caché.

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
| `external-change.txt` | Origen: el archivo generado localmente |
| `remote:cold-storage-test/external-change.txt` | Destino con el nombre de archivo exacto; `copyto` copia un único archivo con exactamente ese nombre, en lugar de hacerlo en un directorio como `copy` |

</details>

Mida cuándo aparece el archivo en el montaje. Repita la prueba para la modificación y la eliminación. El resultado depende del backend, de su compatibilidad con polling y de `--poll-interval` y `--dir-cache-time`. Si la aplicación necesita ver los cambios actuales de inmediato, este comportamiento debe formar parte explícita de los criterios de aceptación.

Con la interfaz de control remoto activada, puede descartar específicamente la caché de directorios:

```bash
rclone rc vfs/forget
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `vfs/forget` | Comando de control remoto que se ejecutará: descarta el árbol de directorios VFS almacenado en caché, de modo que el siguiente acceso vuelve a consultar el remoto |

</details>

Esto resulta útil para una prueba manual, pero no sustituye una estrategia operativa adecuada.

## Someter la caché a presión

Una caché casi vacía es el caso más sencillo. En una segunda ronda de pruebas, configure `--vfs-cache-max-size` deliberadamente bajo y lea más datos de los que caben en ella.

```bash
du -sh /var/cache/rclone-test/vfs
du -sh --apparent-size /var/cache/rclone-test/vfs
find /var/cache/rclone-test/vfs -type f | wc -l
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `du -s` | Resume el uso de espacio en una línea total, en lugar de listar cada subdirectorio |
| `du -h` | Salida en unidades legibles para humanos (K, M, G) |
| `du --apparent-size` | Muestra el tamaño lógico del archivo en lugar del espacio de disco realmente ocupado |
| `find … -type f` | Solo tiene en cuenta archivos regulares, no directorios |
| `wc -l` | Cuenta las líneas de la salida, en este caso el número de archivos de caché |

</details>

Ambos tamaños pueden diferir considerablemente. En modo Full, Rclone utiliza archivos dispersos: un archivo muestra su tamaño lógico completo aunque solo las áreas leídas ocupen espacio local.

Además, el límite de caché es flexible. Rclone lo comprueba al ritmo de `--vfs-cache-poll-interval`, y los archivos abiertos no se pueden eliminar. Por ello, la caché puede superar brevemente el límite. Sin embargo, debería volver a reducirse después de cerrar los archivos y tras la siguiente limpieza.

Registre el valor máximo, el valor tras la limpieza y el tiempo necesario. Así podrá dimensionar razonablemente el almacenamiento local requerido.

## Simular dos fallos diferentes

Una nube inaccesible y un proceso de Rclone que falla son dos errores distintos:

| Fallo | Qué comprueba con ello |
|---|---|
| Backend o red inaccesibles, Rclone sigue ejecutándose | Comportamiento ante reintentos, tiempos de espera y archivos ya almacenados en caché |
| Proceso de Rclone finalizado | Comportamiento del montaje FUSE y recuperación del punto de montaje |

Simule ambos solo en el entorno de pruebas. Para el segundo caso, puede finalizar de forma forzada un contenedor de Rclone:

```bash
docker kill --signal KILL <rclone-container>
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `--signal KILL` | Envía SIGKILL en lugar de la señal predeterminada SIGTERM; el proceso no tiene oportunidad de limpiar |
| `<rclone-container>` | Nombre o ID del contenedor de Rclone |

</details>

Durante el fallo, compruebe la aplicación y no solo el punto de montaje:

- ¿Qué funciones siguen disponibles?
- ¿Cuánto espera un acceso antes de que aparezca un error?
- ¿Siguen siendo accesibles los archivos ya almacenados completamente en caché?
- ¿La aplicación detiene nuevas operaciones de escritura?
- ¿Aparece un mensaje de error comprensible o solo un proceso bloqueado?
- ¿Se activa su monitorización?

Un servicio de escritura no debe escribir sin que se note en el directorio local subyacente cuando falta el montaje. Tras el regreso del montaje, esos archivos quedarían ocultos. Una protección sencilla antes de cada tarea de escritura es:

```bash
mountpoint -q /mnt/rclone-test || exit 1
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-q` | Sin salida; el resultado solo está en el código de salida |
| `/mnt/rclone-test` | Ruta que se comprobará; código de salida 0 solo si realmente hay un montaje activo allí |
| `\|\| exit 1` | Interrumpe el script si la ruta no es un punto de montaje |

</details>

Después de reiniciar Rclone, compruebe el montaje en el host y desde cada contenedor consumidor. Un montaje reconstruido solo llega a un contenedor que ya está en ejecución con la propagación de montaje adecuada. Para Docker, normalmente se requiere `rslave` en el lado consumidor. Los detalles se explican en el artículo [Operar montajes de Rclone en Docker de forma fiable](/blog/rclone-mount-in-docker-container).

## Un ejemplo concreto de Paperless-ngx

Para mi prueba de Paperless, generé 40 PDF con un total de 13.9 MB. Un documento que no se había abierto antes tardó unos 1.8 segundos, mientras que un acceso repetido inmediatamente tardó entre 19 y 24 milisegundos. Una caché VFS limitada a 4 MB subió temporalmente a 12.7 MiB y volvió a limpiarse en la siguiente ejecución.

Mientras el remoto no estaba disponible, la lista de documentos, la búsqueda de texto completo y las vistas previas siguieron funcionando porque esos datos estaban almacenados localmente. Solo no se podía abrir el original. Tras reconstruir el montaje, el contenedor de Paperless en ejecución volvió a poder acceder a los archivos sin tener que reiniciarse.

Estas cifras no son un benchmark de Rclone ni de Proton Drive. Lo interesante es el comportamiento: el almacenamiento en caliente permaneció disponible localmente, las lecturas en frío fueron más lentas pero predecibles y el servicio se recuperó tras el fallo.

## Qué debe incluir el protocolo de pruebas

Un resultado que pueda revisarse posteriormente incluye, como mínimo:

- versión de Rclone y backend utilizado
- sistema operativo, variante de FUSE y sistema de archivos del directorio de caché
- comando completo de montaje sin credenciales
- número, distribución de tamaños y estructura de los archivos de prueba
- valores de lectura en frío y en caliente para varios archivos
- duración de escritura hasta la visibilidad en el remoto
- valor máximo de caché y duración de la limpieza
- resultado de `rclone check --download`
- comportamiento ante un fallo del backend y un proceso de Rclone finalizado
- tiempo de recuperación desde la perspectiva de la aplicación
- reintentos, tiempos de espera, limitaciones y errores de autenticación del registro

Defina de antemano un valor límite para cada punto. Así, la prueba termina con una decisión y no solo con una colección de cifras interesantes.

## Cuándo está listo el montaje

Un montaje de almacenamiento en frío está listo para usarse si puede responder afirmativamente a estas preguntas:

- ¿Las lecturas en frío son suficientemente rápidas para el servicio previsto?
- ¿La caché acelera los accesos repetidos como se espera?
- ¿El uso de espacio local sigue siendo controlable incluso bajo carga?
- ¿Coinciden todos los archivos tras una descarga completa?
- ¿Funcionan todas las operaciones de archivo necesarias con el backend elegido?
- ¿La aplicación se comporta de forma controlada ante un fallo de la nube?
- ¿Las operaciones de escritura se detienen de forma segura cuando falta el montaje?
- ¿Un montaje reconstruido llega a todos los consumidores en ejecución?
- ¿La monitorización muestra el fallo antes de que lo notifique un usuario?

Si falta una respuesta, al menos sabrá exactamente en qué debe seguir trabajando. Eso es mucho más útil que un montaje que se veía bien con el primer `ls` y solo muestra sus límites durante el funcionamiento.

## Fuentes

1.  [Rclone test makefiles](https://rclone.org/commands/rclone_test_makefiles/): archivos de prueba y estructuras de directorios reproducibles con tamaños configurables.

2.  [Rclone mount](https://rclone.org/commands/rclone_mount/): modos de caché VFS, writeback, archivos dispersos, límites de caché y caché de directorios.

3.  [Rclone check](https://rclone.org/commands/rclone_check/): comparación entre origen y destino, incluida la comprobación completa con `--download`.

4.  [Rclone Remote Control](https://rclone.org/rc/): descarte específico de la caché de directorios VFS con `vfs/forget`.

5.  [Rclone Global Flags](https://rclone.org/flags/): referencia completa de las opciones globales, incluidos el registro, las estadísticas y los parámetros VFS.
