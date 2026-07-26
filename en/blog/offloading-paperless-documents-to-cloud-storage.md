---
title: "Running Paperless-ngx on little storage: offloading documents to a cloud service"
navTitle: "Paperless with cloud service"
description: "Paperless-ngx only needs the database, search index and thumbnails locally; the documents themselves can live in a cloud service. What the hands-on test showed, and how to set it up yourself with the ready-made template in three commands."
date: "2026-07-26"
kategorie: "Paperless-ngx"
timeToRead: "6 min to read"
themen:
  - "paperless-ngx"
related:
  - "rclone-mount-inside-docker-container"
  - "proton-drive-on-linux-status"
  - "testing-cloud-mounts-with-generated-pdfs"
translationOf: "paperless-dokumente-clouddienst-auslagern"
slug: "offloading-paperless-documents-to-cloud-storage"
url: "https://rafaelpfister.ch/en/blog/offloading-paperless-documents-to-cloud-storage"
---

Paperless-ngx stores its documents in a local directory, and that directory grows with every scan. Yet Paperless barely needs the files day to day: search runs against the database, the list renders thumbnails, and the actual file is only read when opened. So I tested whether the store can be moved into a cloud service. The tool for the job is Rclone, which Plex users have relied on for years to pull entire media collections in from the cloud.

The result: **it works in both directions**, and the setup has shrunk to three commands. This article summarises what the test showed and how to set it up yourself. The technical depths live in their own articles, linked at the end: Docker mount propagation, AppArmor traps, two-factor authentication and the measurement methodology.

## The principle: hot stays local, cold goes to the cloud

| Component | Location | Why |
|---|---|---|
| Database (holds the OCR text) | local | needs real locking |
| Search index, thumbnails | local | constant access |
| **Document files** | **cloud** | rarely read |
| Cache (recently opened documents) | local, bounded | repeated access stays fast |

In Paperless, of all places, the directory name is misleading: `archive/` is **not** the cold archive but holds the PDF/A version served on every view. Despite its name, it is the hottest file in the system. Cold is `originals/`. If you want maximum savings, turn the archive copy off entirely with `PAPERLESS_ARCHIVE_FILE_GENERATION=never`; full-text search is unaffected because the text lives in the database.

Paperless-ngx ships no cloud storage support of its own, by the way: no S3, no `django-storages`. A file-system mount via Rclone is currently the only way, and it works with any of the 70+ services Rclone supports. Proton Drive was my test choice for its end-to-end encryption; an S3-compatible store is the more robust alternative.

## What the test showed

Tested with an isolated Paperless instance, 40 generated test PDFs (13.9 MB) and a dedicated Proton account:

| Operation | Result |
|---|---|
| Opening a document for the first time (from the cloud) | ~1.8 s |
| Opening the same document again (from the cache) | ~20 ms |
| Consuming a new document until it sits in the cloud | ~20 s |
| Document list, full-text search | 39 ms / 272 ms, works even **without** a cloud connection |
| Integrity check (checksum of every file) | passed, no discrepancy |
| Mount outage | self-healing without a Paperless restart, verified |

That decouples local storage demand from the size of the archive: the collection may grow, the disk does not have to.

## How to set it up

The complete configuration is available as a template on GitHub: [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage). On the server:

```bash
git clone https://github.com/pfstr/paperless-cloud-storage.git
cd paperless-cloud-storage
sudo ./setup.sh      # once, prepares the host (the only root step)
./wizard.sh          # guided: pick provider, credentials, round-trip test
```

The wizard asks for your cloud service (Proton, S3, Backblaze B2, WebDAV, SFTP, or "Not in the list" for any other Rclone backend), verifies the connection with a real upload/download test and starts the storage container. Then:

- **New installation:** `docker compose -f paperless.yml up -d`, done.
- **Existing Paperless instance:** database and settings stay untouched; the guide [RETROFIT.md](https://github.com/pfstr/paperless-cloud-storage/blob/main/docs/RETROFIT.md) covers uploading the existing documents and the one required change to your compose file.

I deliberately skipped a web interface. Rclone's web GUI was in place at first, but SSH tunnels, CORS and ephemeral mounts made it worse than the command line it was meant to replace. Three questions in the terminal are faster.

## The four rules for stable operation

The template implements all of them; if you build your own, you should know them:

1. **`propagation: rslave`** on the Paperless container's media bind mount, otherwise the container does not survive a mount restart. Details and the AppArmor trap behind it: [Rclone mounts inside Docker containers](/en/blog/rclone-mount-inside-docker-container).
2. **Stop Paperless while the mount is missing.** Otherwise it consumes documents into a bare local directory, and the returning mount then shadows them invisibly. A watchdog script ships with the template.
3. **An account that can sign in unattended.** For Proton that means storing the TOTP secret in the Rclone configuration. Why this does not devalue two-factor authentication, and where Proton stands on Linux overall: [Proton Drive on Linux](/en/blog/proton-drive-on-linux-status).
4. **Disable scheduled full-read tasks** (`PAPERLESS_SANITY_TASK_CRON=disable`), because the integrity check otherwise downloads the complete collection from the cloud regularly.

## Limits that remain

A freshly consumed document lives in the local cache for a few seconds until the upload completes. If the machine dies exactly in that window, the file is missing. The cache limit is soft and can be exceeded considerably during access bursts. And Rclone's Proton backend is officially beta; under rapid API calls it showed throttling symptoms. Because long-term data from continuous operation is still missing, the template is marked experimental.

How the measurements came about, which outages were simulated and how to test such a setup seriously at all is covered in the methodology article: [Testing cloud mounts with generated PDFs](/en/blog/testing-cloud-mounts-with-generated-pdfs).

## Conclusion

Paperless-ngx on a small disk with a cloud store is feasible and usable day to day: just under two seconds on first open, cache speed after that, search and interface stay cloud-independent, and the setup heals itself after outages. If you merely want to save a few gigabytes on a normally sized server, do the maths: in my case the entire store occupied 71 MB while the operating system took several gigabytes. The gain is not the space saved immediately, but that the collection may grow without the disk having to grow with it.

## Sources

1.  [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage) — the template from this article: setup.sh, wizard.sh, compose files, watchdog and retrofit guide.

2.  [Rclone: Overview of cloud storage systems](https://rclone.org/overview/) — the 70+ supported services and their capabilities compared.

3.  [Paperless-ngx: Configuration](https://docs.paperless-ngx.com/configuration/) — `PAPERLESS_ARCHIVE_FILE_GENERATION`, `PAPERLESS_SANITY_TASK_CRON` and the other settings used.

4.  [Paperless-ngx: Administration](https://docs.paperless-ngx.com/administration/) — sanity checker, export and import, and the scheduled background tasks.
