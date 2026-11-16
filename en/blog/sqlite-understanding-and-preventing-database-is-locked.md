---
title: "SQLite: Understanding and preventing “database is locked”"
navTitle: "SQLite locks"
description: "Why SQLite responds with SQLITE_BUSY and “database is locked,” how WAL mode, busy_timeout, short transactions, and retries avoid locks, and why slow storage and I/O load on a NAS trigger the issue in the first place. Includes a real-world Grafana case."
date: "2026-11-12"
kategorie: "Databases"
timeToRead: "11 min read"
themen:
  - datenbanken
produkte:
  - "sqlite"
protokolle:
  - "troubleshooting"
  - "container"
slug: "sqlite-understanding-and-preventing-database-is-locked"
translationId: "article-2a328dabf9164dae"
translationOf: sqlite-database-locked-wal-io-last
url: https://rafaelpfister.ch/en/blog/sqlite-understanding-and-preventing-database-is-locked
translationSourceHash: 121b1c0f6dc58cb2249e9e0276937397b52725621cd1553b455a37069f5958de
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T13:50:00.363Z
translationReview: automatic
---

SQLite is used by many self-hosted services: Grafana, Home Assistant, Vaultwarden, Uptime Kuma, Jellyfin, and many other applications store their data in a single file by default. This works until the log reports `database is locked`. It means that an access attempt did not obtain a lock within the permitted wait time; the database file itself remains intact. Whether this happens rarely or continuously depends on three things: the journal mode, how the application handles conflicts, and storage latency.

## Real-world case: Grafana on a NAS

Grafana was running as a Docker container on a Synology NAS, with the SQLite file stored in a named volume. Prometheus, Loki, Paperless-ngx, and several restic backups were running on the same NAS. The drives were Shingled Magnetic Recording (SMR) hard disks.

At first, Grafana’s health endpoint reported `"database": "failing"`. The log contained entries like this at short intervals:

```text
level=error msg="Failed to save history batch"
  error="database is locked (5) (SQLITE_BUSY)"
```

The message comes from the alert state history: by default, Grafana writes state changes of alert rules as annotations to its own database. With many rules, this creates a steady stream of small write transactions. After restarting the container, Grafana no longer came up and kept restarting in a loop:

```text
Failed to stop all modules ... kv get:
  [sqlstore.max-retries-reached] retry 1: database is locked
```

The issue was resolved with two settings in the `docker-compose.yml`: enable WAL mode and have Grafana retry several times when locks occur. Why exactly these two values work follows from how SQLite locking works.

## How SQLite locks

SQLite has no server process coordinating access. Each connection opens the file itself and manages shared access through operating-system file locks. This leads to the most important property: **Only one connection may write at any given time.** This applies in every journal mode.

In classic rollback journal mode (the default, `journal_mode=DELETE`), a write transaction goes through several locking stages:

| Lock | Meaning |
|---|---|
| `SHARED` | The connection is reading. Any number of simultaneous readers are allowed. |
| `RESERVED` | The connection intends to write and is preparing changes. Readers may continue reading, but a second writer may not proceed. |
| `PENDING` | The connection is waiting for active reads to finish. New readers are rejected. |
| `EXCLUSIVE` | The connection writes to the database file. Nobody else may read or write. |

For the commit, the writer therefore needs an exclusive lock, which it gets only when no readers are active. A long-running read thus holds up every writer, and all readers are blocked during the commit.

If a connection cannot obtain the required lock, SQLite returns error code `SQLITE_BUSY` (5) with the text `database is locked`. This must be distinguished from `SQLITE_LOCKED` (6, `database table is locked`): this code concerns conflicts within the same connection or in shared-cache mode and has different causes.

What matters is how long SQLite waits before reporting this message. Without configuration, SQLite does not wait at all: the default value of `busy_timeout` is 0, so the message appears immediately when another connection holds the lock. Many libraries set their own value; Python’s `sqlite3` module, for example, waits 5 seconds.

## WAL mode: separating readers and writers

In Write-Ahead Log mode (`journal_mode=WAL`), SQLite appends changes to a separate `<name>-wal` file instead of writing them directly to the database file. Readers see the state from the beginning of their read transaction and read from both the database file and the WAL. This provides two advantages:

- Readers do not block the writer, and the writer does not block readers.
- A commit is an append to a file rather than several scattered journal writes. Each transaction involves less I/O, and the write lock is held for less time.

WAL mode does not change the fact that only one writer can be active at a time. However, it shortens the time the lock is held and eliminates conflicts between reads and writes. For an application like Grafana, which serves dashboards while saving alert states, this is the most effective measure.

The mode is set once and remains stored in the file:

```sql
PRAGMA journal_mode = WAL;
PRAGMA busy_timeout = 5000;
PRAGMA synchronous = NORMAL;
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `journal_mode = WAL` | Switches to Write-Ahead Log mode. The setting is persistent; new connections automatically open the file in WAL mode. |
| `busy_timeout = 5000` | Wait time in milliseconds during which SQLite repeatedly tries to obtain a held lock. Applies only to the current connection and must be set again every time it is opened. |
| `synchronous = NORMAL` | In WAL mode, no `fsync` is performed for each commit, only at checkpoints. The database remains consistent after a power failure; the most recently committed transactions may be lost. Applies per connection. |

</details>

WAL mode has prerequisites. All processes accessing the file must run on the same computer because they coordinate through a shared-memory file `<name>-shm`. The directory needs write permissions for both additional files, including for processes that only read. And backups must account for the files together (more on this below).

## Checkpoints: when the WAL grows

The WAL grows until a checkpoint transfers the changes to the actual database file. SQLite triggers one automatically as soon as the WAL contains about 1000 pages (`wal_autocheckpoint`, about 4 MB with 4-KiB pages). The checkpoint writes the pages to their positions in the database file and then calls `fsync`. On slow storage, this is precisely the step that takes the longest.

A checkpoint can transfer pages only as long as no reader requires an older state. If at least one read transaction is constantly active, the checkpoint remains incomplete and the WAL continues to grow. A WAL of several hundred megabytes indicates long-running read transactions or checkpoints that cannot finish because of I/O load. You can check it like this:

```sql
PRAGMA wal_checkpoint(TRUNCATE);
```

The output consists of three numbers: whether the checkpoint was blocked (`1` = yes), how many pages are in the WAL, and how many of them were transferred. With `TRUNCATE`, the WAL is then truncated to zero bytes, provided the checkpoint was complete.

## Busy timeout and retries

With a busy timeout, SQLite waits when a conflict occurs instead of aborting immediately. The error appears only when the lock is held longer than the configured time. For applications with few, short write operations, 5 to 10 seconds are sufficient.

There is one case in which the busy timeout does not apply. If a transaction begins as a read (`BEGIN` without an addition, that is, `DEFERRED`) and later wants to write while another writer has completed a commit in the meantime, SQLite immediately returns `SQLITE_BUSY`. The transaction is working from an outdated state and cannot become a write transaction; waiting would not change that. In rollback mode, there is a comparable situation when two connections simultaneously want to move from `SHARED` to a write lock.

The remedy is to designate write transactions as such from the beginning:

```sql
BEGIN IMMEDIATE;
UPDATE alert_state SET state = 'firing' WHERE id = 42;
COMMIT;
```

`BEGIN IMMEDIATE` requests the write lock at the start. If it is held, the busy timeout applies normally. In Python, it looks like this:

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
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `"app.db"` | Path to the database file. |
| `timeout=10.0` | Busy timeout in seconds; corresponds to `PRAGMA busy_timeout = 10000`. |
| `isolation_level=None` | Disables the module’s automatic transaction control so that `BEGIN IMMEDIATE` and `COMMIT` can be set explicitly. |

</details>

Application-level retries complement the busy timeout. They catch `SQLITE_BUSY`, wait briefly, and execute the entire transaction again. This also covers the case of an outdated read state because the retry starts from a new state. Grafana provides dedicated settings for this (see the solution section).

## Short transactions and one writer

The application determines how long a write lock is held. Three rules keep it short:

- **Do no non-database work inside the transaction.** HTTP calls, file access, or calculations belong before `BEGIN` or after `COMMIT`.
- **Bundle many small write operations.** 500 individual commits cost 500 lock cycles and, in rollback mode, several `fsync` calls 500 times. One transaction with 500 rows holds the lock once, for somewhat longer.
- **Close read cursors.** A query result that has not been fully fetched keeps the read transaction open and blocks the checkpoint in WAL mode.

Where multiple threads or processes write, a single writer with a queue is the most reliable solution: one thread or process owns the only writing connection, while all others pass it jobs. This eliminates lock conflicts among writers entirely, while readers continue operating in parallel in WAL mode.

Multiple containers opening the same SQLite file through a shared volume, on the other hand, are a frequent cause of locks. Each container is a separate writer, and neither knows about the other. If an application scales horizontally, a database server such as PostgreSQL or MySQL is the appropriate choice.

## No network file systems

SQLite relies on file locks working correctly and atomically, and on an `fsync` actually bringing data to storage. Neither is guaranteed with NFS, SMB/CIFS, and many FUSE file systems. SQLite’s developers explicitly advise against using a database over a network file system from multiple computers. The consequences range from persistent lock messages to a corrupted database, and WAL mode fundamentally does not work there because shared-memory coordination is possible only locally.

For Docker, this means that a named volume or a bind mount on local storage is suitable. A bind mount on a mounted SMB or NFS share is not, even if it appears to work in normal operation.

## I/O load: why the lock suddenly takes too long

The lock is only a symptom. It becomes a problem when it is held longer than the busy timeout allows, and this almost always depends on storage write latency.

Every commit ends with an `fsync`. SQLite releases the lock only once the storage device confirms that the data has been written persistently. On an SSD, this takes less than a millisecond; on an idle hard drive, a few milliseconds. If the drive must process other jobs at the same time, the `fsync` waits in the queue, and the lock remains held for that long. All other writers wait as well until their busy timeout expires.

Several factors increase this latency on a NAS:

- **SMR hard drives.** In Shingled Magnetic Recording, tracks overlap. Random writes first land in a cache area and are later rearranged into the overlapping zones. If the cache area fills under sustained load, write latency rises from milliseconds to several seconds. Databases with many small, scattered writes are the least favorable workload profile for this.
- **Concurrent backups.** restic, Hyper Backup, or rsync read large amounts of data and compete with every `fsync` for the same spindles.
- **Scrubbing and RAID consistency checks.** A disk scrub reads the entire data set and runs on a schedule on many NAS systems, often for hours.
- **Other databases on the same volume.** Prometheus and Loki continuously write blocks and periodically compact them. These compactions create load spikes.
- **Checkpoints in WAL mode.** The checkpoint writes many pages at once and finishes with `fsync`. If it occurs during a period of high load, it takes correspondingly longer.

In the real-world case, several of these factors coincided: SMR hard drives, backups, and a monitoring environment with several writing services. Grafana was running in rollback mode without retries. Every commit of the alert state history required several `fsync` calls, and each one waited behind the other load. At startup, Grafana reads, among other things, values from its key-value store in the database. Without retries, startup failed at the first lock conflict and the container was restarted.

### Measuring latency

Whether the drives are the problem can be measured directly. `iostat` from the sysstat package shows utilization per drive:

```bash
iostat -x 5
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-x` | Extended statistics per device, including `r_await`/`w_await` (average wait time in ms) and `%util` (utilization). |
| `5` | Interval in seconds; the first output shows averages since system startup and is not meaningful for the current situation. |

</details>

Values for `w_await` in the triple-digit millisecond range and `%util` close to 100 indicate that the drive must queue jobs. The latency of individual synchronous writes can be checked with `dd` in the database directory:

```bash
dd if=/dev/zero of=./fsync-test bs=4k count=500 \
   oflag=dsync
rm ./fsync-test
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `if=/dev/zero` | Source: zero bytes; only write performance is measured. |
| `of=./fsync-test` | Target file in the current directory; it must be on the database volume. |
| `bs=4k` | Block size of 4 KiB, matching the usual SQLite page size. |
| `count=500` | Number of blocks, 2 MB total. |
| `oflag=dsync` | Each block is written synchronously, comparable to one `fsync` per write operation. |

</details>

`dd` reports duration and throughput at the end. 500 blocks in 2 to 5 seconds correspond to 4 to 10 ms per synchronous write operation and are normal for hard drives. If the same test takes 60 seconds or longer during a backup, the storage device is the cause of the locks.

## Solution in the real-world case

Grafana supports both measures through configuration parameters in the `[database]` section, which can be set as environment variables:

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
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `GF_DATABASE_WAL` | Corresponds to `wal = true`: Grafana opens the SQLite file in WAL mode. The default is `false`. |
| `GF_DATABASE_QUERY_RETRIES` | Corresponds to `query_retries`: number of query retries when SQLite reports `database is locked`. The default is `0`. |
| `grafana-data:/var/lib/grafana` | Named volume on the NAS’s local storage; it contains `grafana.db` and, after the change, `grafana.db-wal` and `grafana.db-shm`. |

</details>

For transactions, there is also `transaction_retries` (default `5`), which did not need adjustment in this case. After restarting with both settings, Grafana started without errors, the health endpoint again reported `"database": "ok"`, and the SQLITE_BUSY messages from the alert state history stopped. The new files in the volume show whether the change took effect:

```bash
docker exec grafana ls -l /var/lib/grafana
```

If `grafana.db-wal` and `grafana.db-shm` appear, the database is operating in WAL mode.

For heavily loaded systems, two additional measures can be considered. The alert state history can be written to Loki instead of the database (`[unified_alerting.state_history]` with `backend = loki`) if Loki is already running. And for installations with many users or multiple Grafana instances, Grafana recommends an external database (MySQL or PostgreSQL) instead of SQLite.

## Backing up without lock conflicts

A file backup of `grafana.db` while it is running captures an arbitrary intermediate state. In WAL mode, all changes not yet transferred are also missing without the `-wal` file. SQLite itself creates a consistent copy:

```bash
sqlite3 /var/lib/grafana/grafana.db \
  ".backup /backup/grafana-$(date +%F).db"
```

The online backup API copies the database page by page while accounting for ongoing write operations. Alternatively, `VACUUM INTO '/backup/grafana.db'` creates a compact copy. Both approaches can be run before the actual file backup, which then backs up the copy instead of the live file. Schedule backups and scrubs so that they do not overlap and do not coincide with known load peaks from other services.

## Checklist

- Enable WAL mode, provided all access comes from the same computer.
- Set a busy timeout on every connection, with 5 to 10 seconds as a starting value.
- Start write transactions with `BEGIN IMMEDIATE` and keep them short.
- Provide application-level retries for `SQLITE_BUSY`; in Grafana, use `query_retries`.
- Run only one writing process per database file.
- Store database files only on local file systems, never on SMB or NFS shares.
- Measure write latency when locks occur frequently; avoid SMR hard drives for databases or move databases to an SSD.
- Create backups with `.backup` or `VACUUM INTO` and schedule them separately from scrubs.

## Sources

1.  [SQLite: Result and Error Codes](https://www.sqlite.org/rescode.html): meaning of `SQLITE_BUSY` (5), `SQLITE_LOCKED` (6), and extended codes such as `SQLITE_BUSY_SNAPSHOT`.

2.  [SQLite: File Locking And Concurrency In SQLite Version 3](https://www.sqlite.org/lockingv3.html): the SHARED, RESERVED, PENDING, and EXCLUSIVE lock stages in rollback mode.

3.  [SQLite: Write-Ahead Logging](https://www.sqlite.org/wal.html): how it works, checkpoints, `wal_autocheckpoint`, and the limitation to local file systems.

4.  [SQLite: PRAGMA Statements](https://www.sqlite.org/pragma.html): `journal_mode`, `busy_timeout`, `synchronous`, and `wal_checkpoint`.

5.  [SQLite: BEGIN TRANSACTION](https://www.sqlite.org/lang_transaction.html): difference between DEFERRED, IMMEDIATE, and EXCLUSIVE.

6.  [SQLite: SQLite Over a Network, Caveats and Considerations](https://www.sqlite.org/useovernet.html): why network file systems are unsuitable for SQLite.

7.  [SQLite: How To Corrupt An SQLite Database File](https://www.sqlite.org/howtocorrupt.html): faulty file locks and `fsync` as causes of corrupted databases.

8.  [SQLite: SQLite Backup API](https://www.sqlite.org/backup.html): consistent copies of a database while it is running.

9.  [Python: sqlite3 DB-API 2.0 interface](https://docs.python.org/3/library/sqlite3.html): `timeout` and `isolation_level` parameters of `sqlite3.connect()`.

10.  [Grafana: Configure Grafana, database section](https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/#database): `wal`, `query_retries`, and `transaction_retries` parameters, as well as their mapping to `GF_` environment variables.

11.  [Wikipedia: Shingled magnetic recording](https://en.wikipedia.org/wiki/Shingled_magnetic_recording): construction of SMR hard drives and their behavior with random write operations.
