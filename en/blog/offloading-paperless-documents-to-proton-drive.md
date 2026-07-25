---
title: "Offloading Paperless documents to Proton Drive: a proof of concept"
navTitle: "Paperless & Proton Drive"
description: "Can the Paperless-ngx document store be moved into Proton Drive via rclone and served through a local cache? A test setup with measured access times, behaviour during a cloud outage, and four reasons why this is not production-ready yet."
date: "2026-07-25"
kategorie: "Paperless-ngx"
timeToRead: "11 min to read"
themen:
  - "paperless-ngx"
draft: true
translationOf: "paperless-dokumente-proton-drive-auslagern"
slug: "offloading-paperless-documents-to-proton-drive"
url: "https://rafaelpfister.ch/en/blog/offloading-paperless-documents-to-proton-drive"
aiPrompt: |
  You are my infrastructure assistant. Help me assess whether I can offload the document store of a self-hosted service into cloud storage:
  1. First establish which file system access patterns the application relies on (atomic renames, locking, random writes, many small metadata calls).
  2. Separate hot data (database, search index, thumbnails) from cold data (fully processed original files).
  3. Check whether the cloud storage can be mounted through rclone with a VFS cache, and how the session renews unattended.
  4. Measure cold and warm access times as well as the local disk footprint of the cache under load.
  5. Explicitly test a failure of the mount and whether the application recovers without a restart.
  6. Finally, judge honestly whether the effort justifies the reclaimed disk space, or whether a larger local disk is the better answer.
---

Paperless-ngx stores its documents in a local directory, and that directory keeps growing. The obvious question: could the fully processed files be moved into cloud storage and pulled back through a cache when needed? That is exactly what I built and measured with Proton Drive and rclone. The result up front: it works better than expected technically, but for four specific reasons it is not production-ready.

## The idea: separating hot and cold data

The approach comes from classic hierarchical storage management. Once a document has been consumed, Paperless does not touch the PDF file again during normal operation. Search runs against the search index and the text stored in the database, the document list renders thumbnails. The original file is only read when someone opens or downloads a specific document.

That yields a clean split:

- **Staying local** are the database, the search index and the thumbnails. This data is small, constantly read, and requires real locking.
- **Offloaded** is the directory holding the original files. It accounts for most of the volume and is rarely touched.
- **A local cache** keeps recently opened documents at hand so repeated access stays fast.

One thing matters here: the files cannot simply be copied away. Paperless stores the paths in the database and opens them directly. If a file disappears, the integrity check reports it as missing. The path must therefore still resolve, and that is precisely what a transparent mount provides.

## The test setup

The basis was a small home server running Ubuntu 25.10, Docker and a 29 GB SSD. To leave the production instance untouched, the test ran in a completely separate Paperless instance with its own database, its own volumes and its own port.

Instead of real documents, 40 generated PDF files were used, 13.9 MB in total at 45 to 596 KB per file. There are two reasons for this: the cloud account credentials live on the server in this setup, and for measuring transfer times the size distribution matters, not the content.

On the cloud side a **dedicated Proton account** was used, not the main account. More on that below, because this point decides feasibility.

The mount was created with rclone 1.74.4 and its Proton Drive backend:

```bash
rclone mount proton:paperless-poc/originals /path/media/documents/originals \
  --vfs-cache-mode full --vfs-cache-max-size 4M --vfs-cache-max-age 1h \
  --dir-cache-time 1h --allow-other --cache-dir /path/cache/originals --daemon
```

`--vfs-cache-mode full` is the decisive switch: it ensures that files being read are cached locally and that random access is possible at all. I deliberately set the cache limit to 4 MB, well below the total of 13.9 MB, so that eviction becomes measurable. For the container to read inside the FUSE mount, `user_allow_other` must be present in `/etc/fuse.conf` on the host.

## The measurements

| Operation | Result |
|---|---|
| Upload to Proton Drive (13.9 MiB, 40 files) | 7 s |
| First read of a 596 KB file (empty cache) | 1765 ms |
| Repeated read of the same file (from cache) | 19–24 ms |
| Reading all 40 files cold | 58 s (avg 1468 ms per file) |
| Document list (40 entries) | 39 ms |
| Full-text search | 272 ms |
| Local disk footprint before | 14 MB of original files |
| Local disk footprint after | 5.3 MB cache, plus 3.2 MB thumbnails |

The difference between cold and warm access is roughly a factor of 80. Opening a document from the cloud takes about one and a half to two seconds, opening one already read takes practically no time. For an archive in which individual documents are looked up, that is perfectly usable.

## What worked reliably

**Paperless reads the files transparently.** The container sees all 40 files in the mount, and the application opens them through its normal paths without any adjustment being necessary.

**The integrity check passes.** The bundled sanity checker computes the checksum of every file and compares it against the database. It completed fully and reported no discrepancies. That is the strongest evidence that the offloading is sound: not only do individual accesses work, every single file comes back from the cloud bit for bit.

**Search and the interface do not need the cloud.** The recognised text lives in the database, close to 244,000 characters for a single document in this test. List, search and thumbnails therefore kept working even after I removed the mount entirely. Only opening the original file failed. The web interface stayed reachable.

**Cache eviction works.** After the full read the cache held 24 files, older entries had been removed. The local footprint dropped from 14 MB to 5.3 MB.

## Where it breaks

### 1. Two-factor authentication prevents unattended operation

The first attempt used an account with two-factor authentication enabled. Upload and initial access worked, then everything failed:

```text
422 POST https://drive-api.proton.me/auth/v4/2fa:
Invalid credentials (Code=8002)
```

The session lasted around 35 minutes. When re-authentication became necessary, rclone sent the one-time code entered during setup again, and that code had long been used. A telling symptom was shifting directory listings: first 40 files were visible, then 17, then none.

There are two ways out, and both come at a price. You can store the permanent TOTP seed in the rclone configuration, in which case rclone generates the codes itself. But then the password and the second factor live in the same place, which effectively cancels two-factor authentication for that account. Or you deliberately do without it for a dedicated account, as in this test. For your main account you should do neither.

### 2. The cache limit is not a hard boundary

The configured limit was 4 MB. During the full read the cache grew to 12.7 MiB according to the log, before the cleanup run cut it back to 3.5 MiB:

```text
vfs cache: cleaned: objects 9 (was 38) in use 1, total size 3.471Mi (was 12.698Mi)
```

Cleanup runs at an interval of roughly one minute. During a burst of access the cache can therefore briefly hold almost the entire offloaded set locally. Anyone budgeting disk space tightly has to plan for that peak, not for the configured limit.

### 3. Routine tasks pull the entire set across the network

The sanity checker verifies checksums and reads every file in full to do so. In this test that took 58 seconds for 13.9 MB. Extrapolated to a real archive of 10 GB, this becomes an operation of several hours with the corresponding data volume. The same applies to the document exporter, to regenerating thumbnails and to running text recognition again.

Anyone operating such a setup therefore has to disable these scheduled tasks deliberately or stretch them considerably. Otherwise a background process regularly undoes the offloading entirely.

### 4. After an outage the container does not recover on its own

This is the most serious point. I removed the mount to simulate a cloud outage, then set it up again. On the host all 40 files were immediately back. Inside the container they were not:

```text
ls: cannot access '/usr/src/paperless/media/documents/originals':
Transport endpoint is not connected
```

The container holds on to the dead mount. Only restarting the container, which in this setup took 95 seconds to reach a healthy state, restored access. Translated to real operation: every expired session, every network hiccup and every restart of the mount requires manual intervention. Without monitoring that checks the mount and restarts the container automatically, this is not operationally sound.

## Two side findings

**Exports are version-bound.** Attempting to import an export created with version 2.20.3 into a 3.0.2 instance failed with `SavedView has no field named 'show_on_dashboard'`. An export therefore belongs to the version that created it. For backups that means: export again after an upgrade.

**Archive files are missing in this test.** The generated PDF files already contained a text layer, so Paperless in `skip` mode produced no archive version. With real scans that directory is often larger than the one holding the originals. The offloaded volume in real operation would therefore be considerably higher than measured here, though the behaviour would be identical.

## Conclusion

As a proof of concept the question is answered: **yes, it works.** Paperless-ngx runs with a document store in Proton Drive, serves documents transparently, passes its own integrity check, and both search and interface remain independent of the cloud. The price is just under two seconds when opening a document for the first time, which is acceptable for an archive.

For production that is not enough. The missing self-healing after a mount failure is the point at which you discover on a weekend that no document has been openable for days. Add to that an authentication scheme that either fails unattended or devalues the second factor, and a backend the vendor explicitly labels as beta.

The more honest conclusion, however, concerns the original question. In the case that prompted this test, the document store occupied 71 MB, while the system journal took 2.9 GB and old package revisions 1.4 GB. The disk problem was not with the documents at all. Anyone who genuinely needs space is better served by a larger disk or storage on the local network: both provide a real file system with low latency instead of emulating one on top of an object interface.

Proton Drive remains very much worthwhile for this purpose, just in a different place: as an encrypted target for a regular backup of the document export. That is what the interface is built for, latency is irrelevant there, and an expired token then prevents at most one backup run rather than access to the archive.

## Sources

1.  [rclone: Proton Drive](https://rclone.org/protondrive/) — documentation of the backend including the two-factor authentication options and the note about its beta status.

2.  [rclone mount](https://rclone.org/commands/rclone_mount/) — description of the VFS cache modes, the size and age limits, and their constraints.

3.  [Paperless-ngx: Administration](https://docs.paperless-ngx.com/administration/) — sanity checker, export and import, and the scheduled background tasks.

4.  [Paperless-ngx: Releases](https://github.com/paperless-ngx/paperless-ngx/releases) — version 3.0.2 and the breaking changes introduced in 3.0.

5.  [Proton Drive SDK preview](https://proton.me/blog/proton-drive-sdk-preview) — status of the official SDK, which targets file transfer and according to Proton is not yet released for third parties.
