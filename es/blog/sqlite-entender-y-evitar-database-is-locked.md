---
title: "SQLite: entender y evitar «database is locked»"
navTitle: "Bloqueos de SQLite"
description: "Por qué SQLite responde con SQLITE_BUSY y «database is locked», cómo el modo WAL, busy_timeout, transacciones cortas y reintentos evitan bloqueos, y por qué los soportes lentos y la carga de E/S en una NAS desencadenan el problema. Con un caso práctico de Grafana."
date: "2026-11-12"
kategorie: "Bases de datos"
timeToRead: "11 min de lectura"
themen:
  - datenbanken
produkte:
  - "sqlite"
protokolle:
  - "troubleshooting"
  - "container"
slug: "sqlite-entender-y-evitar-database-is-locked"
translationId: "article-2a328dabf9164dae"
translationOf: sqlite-database-locked-wal-io-last
url: https://rafaelpfister.ch/es/blog/sqlite-entender-y-evitar-database-is-locked
translationSourceHash: 121b1c0f6dc58cb2249e9e0276937397b52725621cd1553b455a37069f5958de
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T13:52:11.349Z
translationReview: automatic
---

SQLite se utiliza en muchos servicios autogestionados: Grafana, Home Assistant, Vaultwarden, Uptime Kuma, Jellyfin y muchas otras aplicaciones almacenan sus datos de forma predeterminada en un único archivo. Esto funciona hasta que aparece en el registro el mensaje `database is locked`. Indica que un acceso no ha podido obtener un bloqueo dentro del tiempo de espera permitido; el archivo de base de datos permanece intacto. Que ocurra rara vez o constantemente depende de tres factores: el modo de diario, el comportamiento de la aplicación ante conflictos y la latencia del soporte de almacenamiento.

## Caso práctico: Grafana en una NAS

Grafana se ejecutaba como contenedor Docker en una NAS Synology, y el archivo SQLite estaba en un volumen con nombre. En la misma NAS se ejecutaban Prometheus, Loki, Paperless-ngx y varias copias de seguridad de restic. Las unidades eran discos duros con Shingled Magnetic Recording (SMR).

Al principio, el endpoint de salud de Grafana informaba `"database": "failing"`. En el registro aparecían a intervalos breves entradas de este tipo:

```text
level=error msg="Failed to save history batch"
  error="database is locked (5) (SQLITE_BUSY)"
```

El mensaje procede del historial de estados de alertas: Grafana guarda de forma predeterminada los cambios de estado de las reglas de alerta como anotaciones en su propia base de datos. Con muchas reglas, se generan continuamente pequeñas transacciones de escritura. Tras reiniciar el contenedor, Grafana ya no arrancaba y se reiniciaba en bucle:

```text
Failed to stop all modules ... kv get:
  [sqlstore.max-retries-reached] retry 1: database is locked
```

El caso se resolvió con dos ajustes en el `docker-compose.yml`: activar el modo WAL y permitir que Grafana reintente varias veces ante bloqueos. Por qué funcionan precisamente estos dos valores se desprende del funcionamiento de los bloqueos de SQLite.

## Cómo bloquea SQLite

SQLite no tiene un proceso de servidor que coordine los accesos. Cada conexión abre el archivo por sí misma y regula el acceso compartido mediante bloqueos de archivos del sistema operativo. De ello se deriva la característica más importante: **en todo momento, solo una conexión puede escribir.** Esto se aplica a todos los modos de diario.

En el modo clásico de diario de reversión (predeterminado, `journal_mode=DELETE`), una transacción de escritura pasa por varios niveles de bloqueo:

| Bloqueo | Significado |
|---|---|
| `SHARED` | La conexión lee. Se permite cualquier número de lectores simultáneos. |
| `RESERVED` | La conexión quiere escribir y prepara cambios. Los lectores pueden seguir leyendo, pero no puede haber un segundo escritor. |
| `PENDING` | La conexión espera a que terminen las lecturas en curso. Se rechazan nuevos lectores. |
| `EXCLUSIVE` | La conexión escribe en el archivo de base de datos. Nadie más puede leer ni escribir. |

Por tanto, para confirmar los cambios el escritor necesita el bloqueo exclusivo, y solo lo obtiene cuando ya no hay lectores activos. Una lectura de larga duración retrasa así a todos los escritores y, durante la confirmación, todos los lectores quedan bloqueados.

Si una conexión no obtiene el bloqueo necesario, SQLite devuelve el código de error `SQLITE_BUSY` (5) con el texto `database is locked`. Debe distinguirse de `SQLITE_LOCKED` (6, `database table is locked`): este código afecta a conflictos dentro de la misma conexión o en modo de caché compartida, y tiene otras causas.

Lo decisivo es cuánto tiempo espera SQLite antes de este mensaje. Sin configuración, SQLite no espera en absoluto: el valor predeterminado de `busy_timeout` es 0, y el mensaje aparece de inmediato en cuanto otra conexión mantiene el bloqueo. Muchas bibliotecas establecen su propio valor; por ejemplo, el módulo `sqlite3` de Python espera 5 segundos.

## Modo WAL: separar lectores y escritores

En el modo Write-Ahead Log (`journal_mode=WAL`), SQLite añade los cambios a un archivo independiente `<name>-wal` en vez de escribirlos directamente en el archivo de base de datos. Los lectores ven el estado existente al comienzo de su transacción de lectura y, para ello, leen del archivo de base de datos y del WAL. Esto aporta dos ventajas:

- Los lectores no bloquean al escritor, y el escritor no bloquea a los lectores.
- Una confirmación consiste en añadir datos a un archivo en lugar de varias operaciones de escritura dispersas con un diario. Hay menos E/S por transacción y el bloqueo de escritura se mantiene durante menos tiempo.

El modo WAL no cambia el hecho de que solo puede haber un escritor activo a la vez. Sin embargo, reduce el tiempo durante el cual se mantiene el bloqueo y elimina los conflictos entre lectura y escritura. Para una aplicación como Grafana, que sirve paneles y guarda estados de alertas simultáneamente, es la medida más eficaz.

El modo se establece una vez y queda almacenado en el archivo:

```sql
PRAGMA journal_mode = WAL;
PRAGMA busy_timeout = 5000;
PRAGMA synchronous = NORMAL;
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `journal_mode = WAL` | Cambia a Write-Ahead Log. El ajuste es persistente; las nuevas conexiones abren automáticamente el archivo en modo WAL. |
| `busy_timeout = 5000` | Tiempo de espera en milisegundos durante el cual SQLite intenta repetidamente obtener un bloqueo ocupado. Solo se aplica a la conexión actual y debe configurarse de nuevo cada vez que se abre. |
| `synchronous = NORMAL` | En modo WAL, no se realiza `fsync` en cada confirmación, sino solo en los checkpoints. La base de datos sigue siendo consistente ante un corte de energía; pueden perderse las transacciones confirmadas más recientes. Se aplica por conexión. |

</details>

El modo WAL tiene requisitos. Todos los procesos que acceden al archivo deben ejecutarse en el mismo ordenador, ya que se coordinan mediante un archivo de memoria compartida `<name>-shm`. El directorio necesita permisos de escritura para los dos archivos adicionales, incluso para procesos que solo leen. Y en las copias de seguridad, los archivos deben tratarse conjuntamente (más información al respecto más abajo).

## Checkpoints: cuando crece el WAL

El WAL crece hasta que un checkpoint transfiere los cambios al archivo de base de datos propiamente dicho. SQLite lo activa automáticamente en cuanto el WAL contiene unas 1000 páginas (`wal_autocheckpoint`, aproximadamente 4 MB con páginas de 4 KiB). El checkpoint escribe las páginas en sus posiciones del archivo de base de datos y después llama a `fsync`. En un soporte lento, este paso es precisamente el que más tarda.

Un checkpoint solo puede transferir páginas mientras ningún lector necesite un estado anterior. Si hay continuamente al menos una transacción de lectura en curso, el checkpoint queda incompleto y el WAL sigue creciendo. Un WAL de varios cientos de megabytes indica transacciones de lectura de larga duración o checkpoints que no terminan debido a la carga de E/S. Puede comprobarse así:

```sql
PRAGMA wal_checkpoint(TRUNCATE);
```

La salida consta de tres números: si el checkpoint fue bloqueado (`1` = sí), cuántas páginas hay en el WAL y cuántas de ellas se transfirieron. Con `TRUNCATE` el WAL se reduce después a cero bytes, siempre que el checkpoint se haya completado.

## Tiempo de espera ocupado y reintentos

Con un tiempo de espera ocupado, SQLite espera ante un conflicto en lugar de abortar inmediatamente. El error solo aparece si el bloqueo se mantiene más tiempo que el configurado. Para aplicaciones con pocas operaciones de escritura breves, bastan entre 5 y 10 segundos.

Hay un caso en el que el tiempo de espera ocupado no se aplica. Si una transacción comienza leyendo (`BEGIN` sin añadido, es decir, `DEFERRED`) y más tarde quiere escribir, mientras otro escritor ha completado una confirmación entretanto, SQLite devuelve de inmediato `SQLITE_BUSY`. La transacción trabaja sobre un estado obsoleto y no puede convertirse en una transacción de escritura; esperar no cambiaría nada. En modo de reversión existe una situación comparable cuando dos conexiones quieren pasar simultáneamente de `SHARED` a un bloqueo de escritura.

La solución consiste en marcar las transacciones de escritura como tales desde el principio:

```sql
BEGIN IMMEDIATE;
UPDATE alert_state SET state = 'firing' WHERE id = 42;
COMMIT;
```

`BEGIN IMMEDIATE` solicita el bloqueo de escritura al inicio. Si está ocupado, el tiempo de espera ocupado se aplica con normalidad. En Python, sería así:

```python
import sqlite3

con = sqlite3.connect("app.db", timeout=10.0,
                      isolation_level=None)
con.execute("PRAGMA journal_mode = WAL")
con.execute("BEGIN IMMEDIATE")
con.execute("UPDATE jobs SET status = ? WHERE id = ?",
            ("done", 42))
con.execute("COMMIT")
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `"app.db"` | Ruta al archivo de base de datos. |
| `timeout=10.0` | Tiempo de espera ocupado en segundos; equivale a `PRAGMA busy_timeout = 10000`. |
| `isolation_level=None` | Desactiva el control automático de transacciones del módulo para que `BEGIN IMMEDIATE` y `COMMIT` puedan establecerse explícitamente. |

</details>

Los reintentos a nivel de aplicación complementan el tiempo de espera ocupado. Capturan `SQLITE_BUSY`, esperan brevemente y vuelven a ejecutar toda la transacción. Esto también cubre el caso de un estado de lectura obsoleto, ya que el reintento comienza con un estado nuevo. Grafana ofrece sus propios ajustes para ello (véase la sección sobre la solución).

## Transacciones cortas y un escritor

La duración durante la cual se mantiene un bloqueo de escritura la determina la aplicación. Tres reglas ayudan a mantenerla breve:

- **No realizar dentro de la transacción trabajo que no tenga relación con la base de datos.** Las llamadas HTTP, los accesos a archivos o los cálculos deben hacerse antes de `BEGIN` o después de `COMMIT`.
- **Agrupar muchas operaciones de escritura pequeñas.** 500 confirmaciones individuales implican 500 ciclos de bloqueo y, en modo de reversión, 500 llamadas múltiples a `fsync`. Una transacción con 500 filas mantiene el bloqueo una vez y durante algo más de tiempo.
- **Cerrar los cursores de lectura.** Un resultado de consulta que no se ha recuperado por completo mantiene abierta la transacción de lectura y bloquea el checkpoint en modo WAL.

Cuando escriben varios hilos o procesos, un único escritor con cola es la solución más fiable: un hilo o proceso posee la única conexión de escritura, y todos los demás le entregan tareas. Así se eliminan por completo los conflictos de bloqueo entre escritores, mientras los lectores continúan trabajando en paralelo en modo WAL.

En cambio, varios contenedores que abren el mismo archivo SQLite mediante un volumen compartido son una causa frecuente de bloqueos. Cada contenedor es un escritor independiente y ninguno conoce al otro. Si una aplicación se escala horizontalmente, un servidor de base de datos como PostgreSQL o MySQL es la elección adecuada.

## Sin sistemas de archivos de red

SQLite depende de que los bloqueos de archivos funcionen correctamente y de forma atómica, y de que un `fsync` lleve realmente los datos al soporte de almacenamiento. En NFS, SMB/CIFS y muchos sistemas de archivos FUSE, ninguna de las dos cosas está garantizada. Los desarrolladores de SQLite desaconsejan expresamente usar una base de datos a través de un sistema de archivos de red desde varios ordenadores. Las consecuencias van desde mensajes de bloqueo permanentes hasta una base de datos dañada, y el modo WAL no funciona allí por principio, ya que la sincronización de memoria compartida solo es posible localmente.

Para Docker, esto significa que un volumen con nombre o un bind mount en un soporte local es adecuado. Un bind mount en un recurso SMB o NFS montado no lo es, aunque parezca funcionar en condiciones normales.

## Carga de E/S: por qué el bloqueo de repente dura demasiado

El bloqueo es solo un síntoma. Se convierte en un problema cuando se mantiene más tiempo del que permite el tiempo de espera ocupado, y esto casi siempre depende de la latencia de escritura del soporte de almacenamiento.

Cada confirmación termina con un `fsync`. SQLite no libera el bloqueo hasta que el soporte confirma que los datos se han escrito de forma permanente. En una SSD, esto tarda menos de un milisegundo; en un disco duro sin carga, unos pocos milisegundos. Si el soporte debe procesar simultáneamente otras tareas, el `fsync` espera en la cola, y el bloqueo permanece durante ese tiempo. Todos los demás escritores esperan hasta que vence su tiempo de espera ocupado.

Varios factores aumentan esta latencia en una NAS:

- **Discos duros SMR.** En Shingled Magnetic Recording, las pistas se solapan. Las escrituras aleatorias llegan primero a una zona de caché y más tarde se reorganizan en las zonas solapadas. Si la zona de caché se llena bajo carga continua, la latencia de escritura aumenta de milisegundos a varios segundos. Las bases de datos con muchas escrituras pequeñas y dispersas presentan el perfil de carga menos favorable para ello.
- **Copias de seguridad paralelas.** restic, Hyper Backup o rsync leen grandes cantidades de datos y compiten con cada `fsync` por los mismos discos.
- **Scrubbing y comprobaciones de consistencia RAID.** Un scrub de disco lee todo el conjunto de datos y se ejecuta en muchos sistemas NAS según una programación, a menudo durante horas.
- **Otras bases de datos en el mismo volumen.** Prometheus y Loki escriben bloques continuamente y los comprimen periódicamente. Estas compactaciones generan picos de carga.
- **Checkpoints en modo WAL.** El checkpoint escribe muchas páginas de una vez y termina con `fsync`. Si coincide con un periodo de carga elevada, se prolonga en consecuencia.

En el caso práctico coincidieron varios de estos factores: discos duros SMR, copias de seguridad y un entorno de monitorización con varios servicios que escribían. Grafana se ejecutaba en modo de reversión sin reintentos. Cada confirmación del historial de estados de alertas requería varias llamadas a `fsync`, y cada una esperaba detrás de la carga restante. Al iniciar, Grafana lee, entre otros elementos, valores de su almacén clave-valor en la base de datos. Sin reintentos, el inicio abortaba ante el primer conflicto de bloqueo y el contenedor se reiniciaba.

### Medir la latencia

Puede medirse directamente si los soportes de almacenamiento son el problema. `iostat` del paquete sysstat muestra la utilización por dispositivo:

```bash
iostat -x 5
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-x` | Estadísticas ampliadas por dispositivo, entre ellas `r_await`/`w_await` (tiempo medio de espera en ms) y `%util` (utilización). |
| `5` | Intervalo en segundos; la primera salida muestra valores medios desde el arranque del sistema y no es representativa de la situación actual. |

</details>

Valores de `w_await` en el rango de cientos de milisegundos y `%util` cerca de 100 indican que el soporte debe acumular tareas. La latencia de operaciones individuales de escritura síncrona puede comprobarse con `dd` en el directorio de la base de datos:

```bash
dd if=/dev/zero of=./fsync-test bs=4k count=500 \
   oflag=dsync
rm ./fsync-test
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `if=/dev/zero` | Origen: bytes nulos; solo se mide el rendimiento de escritura. |
| `of=./fsync-test` | Archivo de destino en el directorio actual; este debe estar en el volumen de la base de datos. |
| `bs=4k` | Tamaño de bloque de 4 KiB, equivalente al tamaño de página habitual de SQLite. |
| `count=500` | Número de bloques, 2 MB en total. |
| `oflag=dsync` | Cada bloque se escribe de forma síncrona, comparable a un `fsync` por operación de escritura. |

</details>

`dd` informa al final de la duración y el rendimiento. 500 bloques en 2 a 5 segundos equivalen a entre 4 y 10 ms por operación de escritura síncrona y son normales para discos duros. Si la misma prueba tarda 60 segundos o más durante una copia de seguridad, la causa de los bloqueos está en el soporte de almacenamiento.

## Solución en el caso práctico

Grafana admite ambas medidas mediante parámetros de configuración en la sección `[database]`, que pueden establecerse como variables de entorno:

```yaml
services:
  grafana:
    image: grafana/grafana-oss:13.0.2
    environment:
      GF_DATABASE_WAL: "true"
      GF_DATABASE_QUERY_RETRIES: "5"
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  grafana-data:
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `GF_DATABASE_WAL` | Equivale a `wal = true`: Grafana abre el archivo SQLite en modo WAL. El valor predeterminado es `false`. |
| `GF_DATABASE_QUERY_RETRIES` | Equivale a `query_retries`: número de reintentos de una consulta cuando SQLite informa `database is locked`. El valor predeterminado es `0`. |
| `grafana-data:/var/lib/grafana` | Volumen con nombre en el soporte local de la NAS; allí se encuentran `grafana.db` y, tras el cambio, `grafana.db-wal` y `grafana.db-shm`. |

</details>

Para las transacciones también existe `transaction_retries` (valor predeterminado `5`), que no tuvo que ajustarse en el caso práctico. Tras reiniciar con ambos ajustes, Grafana arrancó sin errores, el endpoint de salud volvió a informar `"database": "ok"`, y dejaron de aparecer los mensajes SQLITE_BUSY del historial de estados de alertas. Los nuevos archivos en el volumen muestran si el cambio se ha aplicado:

```bash
docker exec grafana ls -l /var/lib/grafana
```

Si aparecen `grafana.db-wal` y `grafana.db-shm`, la base de datos funciona en modo WAL.

En sistemas con alta carga se pueden evaluar dos medidas adicionales. El historial de estados de alertas puede escribirse en Loki en lugar de en la base de datos (`[unified_alerting.state_history]` con `backend = loki`), si ya se ejecuta un Loki. Y para instalaciones con muchos usuarios o varias instancias de Grafana, Grafana recomienda una base de datos externa (MySQL o PostgreSQL) en lugar de SQLite.

## Copias de seguridad sin conflictos de bloqueo

Una copia de archivos de `grafana.db` durante el funcionamiento captura un estado intermedio arbitrario. En modo WAL, sin el archivo `-wal` también faltan todos los cambios aún no transferidos. SQLite puede crear por sí mismo una copia consistente:

```bash
sqlite3 /var/lib/grafana/grafana.db \
  ".backup /backup/grafana-$(date +%F).db"
```

La API de copia de seguridad en línea copia la base de datos página por página y tiene en cuenta las operaciones de escritura en curso. Como alternativa, `VACUUM INTO '/backup/grafana.db'` crea una copia compacta. Ambas variantes pueden ejecutarse antes de la copia de archivos propiamente dicha, que entonces protege la copia en lugar del archivo activo. Debe programar las copias de seguridad y los scrubs de modo que no se solapen ni coincidan con picos de carga conocidos de otros servicios.

## Lista de comprobación

- Activar el modo WAL siempre que todos los accesos procedan del mismo ordenador.
- Configurar un tiempo de espera ocupado en cada conexión; entre 5 y 10 segundos como valor inicial.
- Iniciar las transacciones de escritura con `BEGIN IMMEDIATE` y mantenerlas breves.
- Prever reintentos para `SQLITE_BUSY` a nivel de aplicación; en Grafana mediante `query_retries`.
- Ejecutar un único proceso de escritura por archivo de base de datos.
- Almacenar los archivos de base de datos solo en sistemas de archivos locales, nunca en recursos SMB o NFS.
- Medir la latencia de escritura cuando los bloqueos sean frecuentes; evitar discos duros SMR para bases de datos o trasladarlas a una SSD.
- Crear copias de seguridad mediante `.backup` o `VACUUM INTO` y separarlas temporalmente de los scrubs.

## Fuentes

1.  [SQLite: Result and Error Codes](https://www.sqlite.org/rescode.html): significado de `SQLITE_BUSY` (5), `SQLITE_LOCKED` (6) y códigos extendidos como `SQLITE_BUSY_SNAPSHOT`.

2.  [SQLite: File Locking And Concurrency In SQLite Version 3](https://www.sqlite.org/lockingv3.html): los niveles de bloqueo SHARED, RESERVED, PENDING y EXCLUSIVE en modo de reversión.

3.  [SQLite: Write-Ahead Logging](https://www.sqlite.org/wal.html): funcionamiento, checkpoints, `wal_autocheckpoint` y la restricción a sistemas de archivos locales.

4.  [SQLite: PRAGMA Statements](https://www.sqlite.org/pragma.html): `journal_mode`, `busy_timeout`, `synchronous` y `wal_checkpoint`.

5.  [SQLite: BEGIN TRANSACTION](https://www.sqlite.org/lang_transaction.html): diferencia entre DEFERRED, IMMEDIATE y EXCLUSIVE.

6.  [SQLite: SQLite Over a Network, Caveats and Considerations](https://www.sqlite.org/useovernet.html): por qué los sistemas de archivos de red no son adecuados para SQLite.

7.  [SQLite: How To Corrupt An SQLite Database File](https://www.sqlite.org/howtocorrupt.html): bloqueos de archivos defectuosos y `fsync` como causas de bases de datos dañadas.

8.  [SQLite: SQLite Backup API](https://www.sqlite.org/backup.html): copias consistentes de una base de datos durante el funcionamiento.

9.  [Python: sqlite3 DB-API 2.0 interface](https://docs.python.org/3/library/sqlite3.html): parámetros `timeout` y `isolation_level` de `sqlite3.connect()`.

10.  [Grafana: Configure Grafana, sección database](https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/#database): parámetros `wal`, `query_retries` y `transaction_retries`, así como la correspondencia con variables de entorno `GF_`.

11.  [Wikipedia: Shingled magnetic recording](https://en.wikipedia.org/wiki/Shingled_magnetic_recording): estructura de los discos duros SMR y su comportamiento ante operaciones de escritura aleatorias.
