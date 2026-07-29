---
title: "Running Paperless-ngx with limited storage: offload documents to a cloud service"
navTitle: "Paperless with a cloud service"
description: "Paperless-ngx only needs the database, search index and preview images locally; the documents themselves can reside in a cloud service. What the practical test revealed and how to set it up using the ready-made template in three commands."
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
url: "https://rafaelpfister.ch/en/blog/offloading-paperless-documents-to-cloud-storage"
translationId: article-2f00e7c17fc45664
translationModel: gpt-5.6-terra
translatedAt: 2026-07-28T21:05:48.854Z
translationReview: automatic
translationSourceHash: 81212f097221ec6213025dc5de54f583369799181f72747549102e2b4246e021
---

Paperless-ngx stores its documents in a local directory, and that directory grows with every scan. Yet Paperless hardly needs the files in day-to-day use: searches run against the database, the list displays preview images, and the actual file is only read when it is opened. So I tested whether the storage could be moved to a cloud service. The tool for this is Rclone, which Plex users have been using for years to mount entire media collections from the cloud.

The result: **It works both ways**, and the setup has now been reduced to three commands. This article summarises what the test revealed and how to set up the configuration yourself. The technical details are covered in separate articles linked at the end: Docker mount propagation, AppArmor pitfalls, two-factor authentication and the test methodology.

## The principle: hot storage stays local, cold storage lives in the cloud

| Component | Location | Why |
|---|---|---|
| Database (contains the OCR text) | local | requires proper locking |
| Search index, preview images | local | constant access |
| **Document files** | **cloud** | are rarely read |
| Cache (recently opened documents) | local, limited | repeated access remains fast |

In Paperless, the directory name is misleading of all things: `archive/` is **not cold storage**, but contains the PDF/A version delivered whenever a document is viewed. Despite its name, it belongs to hot storage. The rarely needed originals under `originals/` are the actual cold storage. If you want to save as much space as possible, disable the archive copy entirely with `PAPERLESS_ARCHIVE_FILE_GENERATION=never`; full-text search remains unaffected because the text is stored in the database.

Paperless-ngx does not include its own cloud integration, incidentally, neither S3 nor `django-storages`. A filesystem mount via Rclone is currently the only option, and it works with each of the more than 70 services supported by Rclone. Proton Drive was my test choice because of its end-to-end encryption; S3-compatible storage is the more robust alternative.

## What the test revealed

Tested with an isolated Paperless instance, 40 generated test PDFs (13.9 MB) and a dedicated Proton account:

| Operation | Result |
|---|---|
| Open a document for the first time (from the cloud) | ~1.8 s |
| Open the same document again (from the cache) | ~20 ms |
| Add a new document until it is in the cloud | ~20 s |
| Document list, full-text search | 39 ms / 272 ms, also works **without** a cloud connection |
| Integrity check (checksum of every file) | passed, no discrepancies |
| Mount failure | self-healing without restarting Paperless, verified |

Local storage requirements are therefore decoupled from the size of the archive: the collection may grow, but the disk does not have to.

## How to set it up

The complete configuration is available as a template on GitHub: [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage). On the server:

```bash
git clone https://github.com/pfstr/paperless-cloud-storage.git
cd paperless-cloud-storage
sudo ./setup.sh      # one-off, prepares the host (the only root step)
./wizard.sh          # guided: choose provider, credentials, round-trip test
```

The wizard queries the cloud service (Proton, S3, Backblaze B2, WebDAV, SFTP or “Not in the list” for any other Rclone service), tests the connection with a real upload and download test, and starts the storage container. Afterwards:

- **New installation:** `docker compose -f paperless.yml up -d`, done.
- **Existing Paperless instance:** the database and settings remain untouched; the instructions in [RETROFIT.md](https://github.com/pfstr/paperless-cloud-storage/blob/main/docs/RETROFIT.md) describe uploading existing documents and the required change to your Compose file.

I deliberately chose not to provide a web interface. Rclone’s web GUI was initially used, but SSH tunnels, CORS and ephemeral mounts made it worse than the command line it was meant to replace. Three questions in the terminal are faster.

## Keeping the mount stable in day-to-day use

The template addresses four points that you must also consider if building your own setup:

1. **`propagation: rslave`** on the Paperless container’s media bind mount, otherwise the container will not survive a mount restart. Details and the AppArmor pitfall behind it: [Rclone mount in a Docker container](/blog/rclone-mount-in-docker-container).
2. **Stop Paperless when the mount is missing.** Otherwise, it writes documents to an empty local directory, and the returning mount invisibly covers them. A watchdog script is included in the template.
3. **An account that can sign in unattended.** With Proton, this means storing the TOTP key in the Rclone configuration. Why this does not undermine two-factor authentication and Proton’s overall status on Linux: [Proton Drive on Linux](/blog/proton-drive-linux-status).
4. **Disable scheduled full-read tasks** (`PAPERLESS_SANITY_TASK_CRON=disable`), as otherwise the integrity check regularly reads the entire collection from the cloud.

## What to weigh up before using it

A newly added document exists only in the local cache for a few seconds until the upload completes. If the machine dies during this window, the file is missing. The cache limit is soft and may be exceeded significantly for short periods during access spikes. And Rclone’s Proton backend is officially beta; it showed signs of throttling under rapid API calls. Since long-term data from continuous operation is still lacking, the template is labelled experimental.

The methodology article explains how the measurements were obtained, which failures were simulated and how such a setup can be tested properly: [Testing cloud mounts with generated PDFs](/blog/cloud-mount-testen-dummy-pdfs).

## Conclusion

Paperless-ngx on a small disk with cloud storage is feasible and suitable for day-to-day use: just under two seconds when opening a document for the first time, then cache speed; search and the interface remain independent of the cloud, and the setup heals itself after failures. However, if you only want to save a few gigabytes on a normally sized server, you should do the maths: in my case, the entire storage occupied 71 MB, while the operating system used several gigabytes. The benefit is not the space saved immediately, but that the collection can grow without the disk having to grow with it.

## Sources

1.  [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage): the template from this article: setup.sh, wizard.sh, Compose files, watchdog and retrofit instructions.

2.  [Rclone: Overview of cloud storage systems](https://rclone.org/overview/): the more than 70 supported services and a comparison of their capabilities.

3.  [Paperless-ngx: Configuration](https://docs.paperless-ngx.com/configuration/): `PAPERLESS_ARCHIVE_FILE_GENERATION`, `PAPERLESS_SANITY_TASK_CRON` and the other settings used.

4.  [Paperless-ngx: Administration](https://docs.paperless-ngx.com/administration/): sanity checker, export and import, and scheduled background tasks.
