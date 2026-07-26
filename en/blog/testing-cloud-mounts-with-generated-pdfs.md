---
title: "Testing cloud mounts for real: generating PDFs, measuring access, simulating outages"
navTitle: "Testing cloud mounts"
description: "A reproducible testing method for cloud mounts under Paperless-ngx: reproducible PDF files, separated cache layers, integrity checks and hard outage tests."
date: "2026-07-26"
kategorie: "Paperless-ngx"
timeToRead: "8 min to read"
themen:
  - "paperless-ngx"
related:
  - "offloading-paperless-documents-to-cloud-storage"
draft: true
translationOf: "cloud-mount-testen-dummy-pdfs"
slug: "testing-cloud-mounts-with-generated-pdfs"
url: "https://rafaelpfister.ch/en/blog/testing-cloud-mounts-with-generated-pdfs"
---

A cloud mount is not production-ready just because `ls` works. What matters is how long a cold access takes, what the application can still do during an outage, and whether every file comes back unchanged.

The following method emerged from [testing cloud storage for Paperless-ngx](/en/blog/offloading-paperless-documents-to-cloud-storage). It uses artificial documents, measures at the application level rather than just the file-system level, and verifies recovery after a hard mount interruption. The figures come from one specific machine and show orders of magnitude, not generally valid benchmarks.

## Generate test documents instead of uploading real ones

Private documents have no place in a test environment. For transfer and cache measurements, what counts is file size, page count and text layer; the content itself is irrelevant.

The following script writes PDFs directly in PDF syntax, without a single library. Every page gets a block of random text as a content stream; the page count controls the file size. The **fixed seed** makes the distribution reproducible. A **real text layer** stops the document management system from kicking off OCR: what should be measured is the storage path, not Tesseract.

```python
import os, random
random.seed(42)                      # fixed seed: reproducible distribution
outdir = "dummy"
os.makedirs(outdir, exist_ok=True)
WORDS = ("Invoice Receipt Contract Customer Amount Date Position Quantity "
         "Price Total Tax Net Gross Delivery Payment Account Reference").split()

def build(idx, npages):
    objects, kids, objnum = {}, [], 4
    for p in range(1, npages + 1):
        lines = ["Test document %03d - page %d of %d" % (idx, p, npages)]
        for _ in range(45):
            lines.append(" ".join(random.choice(WORDS) for _ in range(11))
                         + " %d" % random.randint(100, 99999))
        parts = ["BT /F1 11 Tf 40 800 Td 14 TL"]
        parts += ["(%s) Tj T*" % l for l in lines]
        parts.append("ET")
        stream = "\n".join(parts).encode("latin-1")
        cnum, pnum = objnum, objnum + 1
        objnum += 2
        objects[cnum] = (b"<< /Length " + str(len(stream)).encode()
                         + b" >>\nstream\n" + stream + b"\nendstream")
        objects[pnum] = ("<< /Type /Page /Parent 2 0 R /MediaBox [0 0 595 842] "
                         "/Resources << /Font << /F1 3 0 R >> >> "
                         "/Contents %d 0 R >>" % cnum).encode()
        kids.append(pnum)
    objects[1] = b"<< /Type /Catalog /Pages 2 0 R >>"
    objects[2] = ("<< /Type /Pages /Kids [%s] /Count %d >>"
                  % (" ".join("%d 0 R" % k for k in kids), npages)).encode()
    objects[3] = b"<< /Type /Font /Subtype /Type1 /BaseFont /Helvetica >>"
    mx = max(objects)
    buf, off = bytearray(b"%PDF-1.4\n"), {}
    for n in range(1, mx + 1):
        off[n] = len(buf)
        buf += ("%d 0 obj\n" % n).encode() + objects[n] + b"\nendobj\n"
    x = len(buf)
    buf += ("xref\n0 %d\n" % (mx + 1)).encode() + b"0000000000 65535 f \n"
    for n in range(1, mx + 1):
        buf += ("%010d 00000 n \n" % off[n]).encode()
    buf += ("trailer\n<< /Size %d /Root 1 0 R >>\nstartxref\n%d\n%%%%EOF\n"
            % (mx + 1, x)).encode()
    return bytes(buf)

for i in range(1, 41):
    pages = random.choice([10, 25, 40, 60, 90, 130])
    open(os.path.join(outdir, "dummy-%03d.pdf" % i), "wb").write(build(i, pages))
```

The result is 40 files of 45 to 596 KB, 13.9 MB in total. That is a realistic distribution for a private document archive. Paperless consumed them at a good 8 seconds per document, with `pdftotext exited 0` in the log confirming the working text layer.

The test instance itself ran fully separated from production: its own database, its own volumes, its own port. One detail that cost time: with `postgres:18` the volume must point to `/var/lib/postgresql`, no longer `/var/lib/postgresql/data`. Otherwise the container restarts in an endless loop.

## Where cold and warm live in this system

Before measuring, it must be clear **which cache layers** sit between the application and the cloud. Otherwise you measure something other than what you claim. With an rclone mount using `--vfs-cache-mode full` there are three:

| Layer | Location | What lives there |
|---|---|---|
| Cloud | at the provider | the only complete truth |
| VFS cache | local disk (`--cache-dir`) | copies of recently read files, bounded by `--vfs-cache-max-size` |
| Page cache | RAM | held transparently by the kernel, both for the file as it is read through the FUSE mount and for the VFS copy itself |

**Cold** in this setup means the file is not in the VFS cache and comes over the network. **Warm** means the VFS copy sits locally, and on repeated reads practically always in RAM as well. The simple two-point measurement therefore captures the network/local boundary:

```bash
S=$(date +%s%3N); cat "$D/$F" > /dev/null; echo "cold: $(( $(date +%s%3N) - S )) ms"
S=$(date +%s%3N); cat "$D/$F" > /dev/null; echo "warm: $(( $(date +%s%3N) - S )) ms"
```

In the test: 1765 ms cold versus 19 to 24 ms warm. That boundary is also the one that matters for the cloud question. Strictly speaking, though, "warm" here means "local, RAM-assisted" and not "from disk".

## Measurement hygiene: drop the page cache and prove the drop

To separate the local layers (RAM versus disk), you have to clear the page cache before measuring and **prove it**. The usual way is `sync` followed by `vmtouch -e`, with `fincore` from util-linux as the check:

```bash
sync
vmtouch -e file.pdf           # evict this file's pages from RAM
fincore file.pdf              # proof: RES must be 0
```

If `vmtouch` is not installed, three lines without any extra package do the job:

```python
import os
fd = os.open("file.pdf", os.O_RDONLY)
os.posix_fadvise(fd, 0, 0, os.POSIX_FADV_DONTNEED)
os.close(fd)
```

Two peculiarities of this setup make the proof mandatory rather than optional:

**Eviction has to hit every path to the same file**: the file as the mount presents it *and* the VFS copy under `--cache-dir`. This lives in the `vfs/` subdirectory; `vfsMeta/` holds only metadata.

**And it can fail silently.** In my control run, `fincore` showed the VFS copy fully resident in RAM after `sync` and `POSIX_FADV_DONTNEED` (600K resident, before and after): rclone keeps the cache file open, and the kernel did not release the pages. Without the `fincore` proof I would have published a "from disk" number that in truth came from RAM. To really isolate the disk layer, you have to stop the rclone process for the measurement or use `echo 3 > /proc/sys/vm/drop_caches` as root.

Put in perspective: at typical document sizes the separation barely pays off. Hot reads through the mount sat at 19 to 35 ms for me, because the FUSE round trips dominate, not the storage medium. The layer that determines the user experience remains network versus local at a factor of 60 to 90. But you may only claim that once the measurement demonstrably keeps the layers apart.

The same applies to the generated test files, by the way: right after generation they sit fully in the page cache. For the upload measurement that is irrelevant, because the network is the bottleneck. But anyone who wants to derive local read numbers from them has to evict first.

Ongoing observation additionally includes a look at the VFS cache after every step (`du -sh` on the cache directory, `find … -type f | wc -l`) and the rclone log via `--log-file` and `--log-level INFO`, which records the eviction behaviour. It also revealed that the configured cache limit is soft: 4 MB configured, 12.7 MiB peak until the once-a-minute cleanup pass kicked in.

## Verify at application level, not just in the file system

That `ls` shows files does not mean the application can read them. User permissions on FUSE mounts are a topic of their own. The access should therefore be verified through the application itself; for Django applications like Paperless via a shell in the container:

```bash
docker compose exec webserver python3 -c "
import os; os.environ.setdefault('DJANGO_SETTINGS_MODULE','paperless.settings')
import django; django.setup()
from documents.models import Document
d = Document.objects.first()
print(os.path.exists(d.source_path), len(open(d.source_path,'rb').read()))
"
```

The hardest check ships with the application: Paperless' `document_sanity_checker` reads every file in full and compares its checksum against the database. "No issues detected" after a run across the cloud mount proves that every file comes back bit for bit. One quirk applies to the write path: the test file must be **unique**, because Paperless rejects a copy of an existing document as a duplicate.

## Simulate outages hard and prove the recovery

An outage test that merely stops the service cleanly tests nothing. Hard means: kill the rclone process, unmount, and then compare what host and application each see:

```bash
pkill -f "rclone mount" ; fusermount3 -u /path/to/mount
docker compose exec webserver ls /usr/src/paperless/media/documents/originals
```

Two things proved measurably important. First, the difference between perspectives: after a re-created mount the host immediately saw everything again, while the container without the right mount propagation stayed stuck on `Transport endpoint is not connected`. The details are in the [article on rclone in Docker](/en/blog/rclone-mount-inside-docker-container). Second, the **restart counter as evidence**: `docker inspect -f 'restarts={{.RestartCount}}'` before and after the test proves whether a container really recovered without a restart or whether Docker quietly helped out.

Equally important: check what still works **without** the cloud. With the mount removed, document list, full-text search and thumbnails ran unchanged. The recognised text sat in the database, close to 244,000 characters for one test document. Only opening the original file failed. Such negative probes belong in every test protocol because they show what damage an outage actually does.

## Evaluate logs with precise patterns

When evaluating logs, search patterns need to be precise enough. A search for `422` returned 39 supposed errors in my test. In fact, the hits came from harmless size figures such as `422504 bytes`. A word-boundary pattern (`grep -E '\b422\b'`) or searching for the complete error message avoids such false alarms.

## Sources

1.  [Paperless-ngx: Administration](https://docs.paperless-ngx.com/administration/): the sanity checker and the other management commands that served as verification tools here.

2.  [rclone mount](https://rclone.org/commands/rclone_mount/): the VFS cache modes and logging options whose behaviour was measured here.

3.  [PDF 1.7 reference (Adobe)](https://opensource.adobe.com/dc-acrobat-sdk-docs/pdfstandards/PDF32000_2008.pdf): the structure of objects, content streams and the xref table that the generator writes directly.
