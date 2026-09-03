---
title: "Running Paperless-ngx with Limited Storage: Offload Documents to a Cloud Service"
navTitle: "Paperless with Cloud Service"
description: "Paperless-ngx only needs the database, search index, and thumbnails locally; the documents themselves can reside in a cloud service. What the practical test showed and how to set it up with the ready-made template in three commands."
date: "2026-07-26"
kategorie: "Paperless-ngx"
timeToRead: "6 min read"
themen:
  - paperless-ngx
related:
  - rclone-mount-in-docker-container
  - proton-drive-linux-status
  - cloud-mount-testen-dummy-pdfs
translationOf: "paperless-dokumente-clouddienst-auslagern"
slug: "offloading-paperless-documents-to-cloud-storage"
translationId: article-2f00e7c17fc45664
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:23:53.343Z
translationReview: automatic
translationSourceHash: 1df015c7f06b7e3e850423bc79663fcd1ac13e66ec5ecd46eb430a0dc5ab3ad1
url: https://rafaelpfister.ch/en/blog/offloading-paperless-documents-to-cloud-storage
---

Paperless-ngx stores its documents in a local directory, and that directory grows with every scan. Yet Paperless barely needs the files in day-to-day use: search runs against the database, the list displays thumbnails, and the actual file is only read when opened. So I tested whether storage can be moved to a cloud service. The tool for this is Rclone, which Plex users have used for years to mount entire media collections from the cloud.

The result: **It works both ways**, and setup has now been reduced to three commands. This article summarizes what the test showed and how to set up the configuration yourself. The technical details are covered in separate articles linked at the end: Docker mount propagation, AppArmor specifics, two-factor authentication, and the testing methodology.

## The principle: Hot storage stays local, cold storage is in the cloud

| Component | Location | Why |
|---|---|---|
| Database (contains the OCR text) | local | requires real locking |
| Search index, thumbnails | local | constant access |
| **Document files** | **cloud** | rarely read |
| Cache (most recently opened documents) | local, limited | repeated access remains fast |

In Paperless, the directory name of all things is misleading: `archive/` is **not cold storage**, but contains the PDF/A version delivered with every view. Despite its name, it belongs to hot storage. The rarely needed originals under `originals/` are the actual cold storage. If you want to save as much as possible, disable the archive copy entirely with `PAPERLESS_ARCHIVE_FILE_GENERATION=never`; full-text search remains unaffected because the text is stored in the database.

Paperless-ngx does not include its own cloud integration, by the way—neither S3 nor `django-storages`. A filesystem mount via Rclone is currently the only option, and it works with each of the more than 70 services supported by Rclone. Proton Drive was my test choice because of its end-to-end encryption; S3-compatible storage is the more robust alternative.

## What the test showed

Tested with an isolated Paperless instance, 40 generated test PDFs (13.9 MB), and a dedicated Proton account:

| Operation | Result |
|---|---|
| Open a document for the first time (from the cloud) | ~1.8 s |
| Open the same document again (from the cache) | ~20 ms |
| Add a new document until it is in the cloud | ~20 s |
| Document list, full-text search | 39 ms / 272 ms, works even **without** a cloud connection |
| Integrity check (checksum of every file) | passed, no discrepancies |
| Mount failure | self-healing without restarting Paperless, verified |

This decouples local storage requirements from the size of the archive: the collection can grow, but the disk does not have to.

## How to set it up

The complete configuration is available as a template on GitHub: [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage). On the server:

```bash
git clone https://github.com/pfstr/paperless-cloud-storage.git
cd paperless-cloud-storage
sudo ./setup.sh      # einmalig, bereitet den Host vor (einziger Root-Schritt)
./wizard.sh          # geführt: Anbieter wählen, Zugangsdaten, Rundlauf-Test
```

The wizard asks for the cloud service (Proton, S3, Backblaze B2, WebDAV, SFTP, or “Not in the list” for any other Rclone service), checks the connection with an actual upload and download test, and starts the storage container. Then:

- **New installation:** `docker compose -f paperless.yml up -d`, done.
- **Existing Paperless instance:** The database and settings remain untouched; the guide [RETROFIT.md](https://github.com/pfstr/paperless-cloud-storage/blob/main/docs/RETROFIT.md) describes uploading existing documents and the required change to your Compose file.

I deliberately chose not to provide a web interface. Rclone’s web GUI was initially used, but SSH tunnels, CORS, and transient mounts made it worse than the command line it was supposed to replace. Three questions in the terminal are faster.

## Keeping the mount stable in day-to-day use

The template handles four points that you must also consider in your own setup:

1. **`propagation: rslave`** on the media bind mount of the Paperless container; otherwise, the container will not survive a mount restart. Details and the underlying AppArmor issue: [Rclone mount in the Docker container](/blog/rclone-mount-in-docker-container).
2. **Stop Paperless when the mount is missing.** Otherwise, it writes documents to an empty local directory, and the returning mount invisibly covers them up. A watchdog script is included with the template.
3. **An account that can log in unattended.** With Proton, this means storing the TOTP key in the Rclone configuration. Why this does not undermine two-factor authentication and Proton’s overall Linux status: [Proton Drive on Linux](/blog/proton-drive-linux-status).
4. **Disable scheduled full-read tasks** (`PAPERLESS_SANITY_TASK_CRON=disable`), because otherwise the integrity check regularly reads the entire collection from the cloud.

## What to consider before using it

A newly added document exists only in the local cache for a few seconds until the upload finishes. If the machine fails during exactly this window, the file is missing. The cache limit is soft and can be exceeded significantly for a short time during access spikes. And Rclone’s Proton backend is officially beta; under rapid API requests, it showed throttling symptoms. Since long-term data from continuous operation is still unavailable, the template is marked as experimental.

The methodology article explains how the measurements were obtained, which failures were simulated, and how such a setup can be tested properly: [Testing cloud mounts with generated PDFs](/blog/cloud-mount-testen-dummy-pdfs).

## Conclusion

Paperless-ngx on a small disk with cloud storage is feasible and suitable for everyday use: just under two seconds for the first open, then cache speed; search and the interface remain independent of the cloud, and the setup heals itself after failures. However, if you only want to save a few gigabytes on a normally sized server, you should do the math: in my case, the entire storage used 71 MB, while the operating system used several gigabytes. The benefit is not the space saved immediately, but that the collection can grow without the disk having to grow with it.

## Sources

1.  [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage): the template from this article: setup.sh, wizard.sh, Compose files, watchdog, and retrofit guide.

2.  [Rclone: Overview of cloud storage systems](https://rclone.org/overview/): the more than 70 supported services and a comparison of their capabilities.

3.  [Paperless-ngx: Configuration](https://docs.paperless-ngx.com/configuration/): `PAPERLESS_ARCHIVE_FILE_GENERATION`, `PAPERLESS_SANITY_TASK_CRON`, and the other settings used.

4.  [Paperless-ngx: Administration](https://docs.paperless-ngx.com/administration/): sanity checker, export and import, and the scheduled background tasks.
