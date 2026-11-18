---
title: "Paperless-ngx 3.4.1, 3.4.2, and 3.5.0: stricter ALLOWED_HOSTS, fixed OIDC 2FA, asynchronous bulk operations"
navTitle: "Paperless-ngx 3.5.0"
description: "With PAPERLESS_URL set, Paperless-ngx 3.5.0 allows only its hostname and localhost, fixes the ineffective AMR default value from 3.4.x, and runs bulk operations asynchronously. Before updating, check which hostnames are used to access Paperless-ngx and add them to PAPERLESS_ALLOWED_HOSTS."
date: "2026-11-17"
kategorie: "Paperless-ngx"
timeToRead: "7 min read"
themen:
  - paperless-ngx
produkte:
  - "paperless-ngx"
protokolle:
  - "releases"
slug: "paperless-ngx-3-4-1-3-4-2-and-3-5-0-stricter-allowed-hosts-fixed-oidc-2fa-asynchronous-bulk"
translationId: article-4c816dd3cd08ff6e
translationOf: paperless-ngx-3-5-0
url: https://rafaelpfister.ch/en/blog/paperless-ngx-3-4-1-3-4-2-and-3-5-0-stricter-allowed-hosts-fixed-oidc-2fa-asynchronous-bulk
translationSourceHash: 2c1c3db7502c455d8090a922dd17654a159d2d094be4f29d0da51f29eababfb4
translationModel: gpt-5.6-terra
translatedAt: 2026-11-17T21:10:43.560Z
translationReview: required
---

# Paperless-ngx 3.4.1, 3.4.2, and 3.5.0: stricter ALLOWED_HOSTS, fixed OIDC 2FA, asynchronous bulk operations

Since 3.4.0, the Paperless-ngx project has released three versions: the bugfix releases 3.4.1 on October 31, 2026, and 3.4.2 on November 3, 2026, as well as the feature release 3.5.0 on November 17, 2026. The two patch versions primarily fix display issues in the redesigned web interface and update Django and other dependencies. 3.5.0 changes the default value for allowed hostnames, fixes a bug that caused the setting introduced in 3.4.0, `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`, to have no effect, runs all bulk operations asynchronously, automatically cleans up expired sessions, and adds manually triggered workflows. None of the three releases mentions a security notice with a CVE or GHSA. The previous state is described in the article [Paperless-ngx 3.4.0: OIDC login without duplicate 2FA, thumbnails without ImageMagick](/blog/paperless-ngx-3-4-0). The updates are aimed at all operators of Paperless-ngx 3.x; installations accessed via internal hostnames and installations using OIDC sign-in are particularly affected.

## The versions at a glance

| Version | Date | Type | Key points |
|---|---|---|---|
| 3.4.1 | 10/31/2026 | Bugfix release | Fixes to the API schema and web interface, lower idle memory usage |
| 3.4.2 | 11/03/2026 | Bugfix release | Bulk edit tasks in Celery workers, Tesseract language installation before index building, filelock 4.2.6, updates to Django and frontend dependencies |
| 3.5.0 | 11/17/2026 | Feature release | ALLOWED_HOSTS without wildcard when PAPERLESS_URL is set, fixed AMR default value, asynchronous bulk operations, automatic session cleanup, manual workflow trigger, tag hierarchy without django-treenode |

## 3.4.1 and 3.4.2: fixes after the feature release

3.4.1 fixes the placement of a schema correction for `id` fields (PR #14604), removes duplicate permission checks on the open button of document cards (PR #14601), largely hides the new PDF controls (PR #14595), moves the preview button in the document list (PR #14594), and fixes the display of the checksum in light mode as well as the position of the custom-field dropdown on mobile devices (PR #14590). In addition, Paperless-ngx now loads more modules only when needed, reducing idle memory usage (PR #14599).

3.4.2 includes eleven bug fixes. The following are relevant for operation:

- Import tasks from `bulk_edit` run reliably in Celery workers (PR #14653).
- Search index and LLM index building waits until the Tesseract language packages are installed (PR #14670).
- filelock is at 4.2.6; the release notes cite ACL fixes as the reason for the update (PR #14656).
- Document history shows tag names instead of IDs (PR #14625), zoom buttons no longer submit forms (PR #14624), and the name of a saved view is retained when saving and closing (PR #14667).

For dependencies, 3.4.2 updates Django, django-guardian, django-filter, and drf-spectacular-sidecar (PR #14661), turbohtml to 1.15.3 (PR #14662), and pdfjs-dist to 6.4.299 (PR #14644). The project replaces the no-longer-maintained ngx-bootstrap-icons library with its own solution (PR #14665). In the example Compose files, Gotenberg is raised to 8.38 (PR #14634) and nginx to 1.31.7 (PR #14633). The release notes for 3.4.1 and 3.4.2 do not mark any of these entries as security updates.

## 3.5.0: ALLOWED_HOSTS without wildcard

The release notes for 3.5.0 begin with a notice that some users need to adjust their `PAPERLESS_ALLOWED_HOSTS` setting (PR #14735). Up to 3.4.2, the project documented the default value `*` for `PAPERLESS_ALLOWED_HOSTS`, meaning all hosts. Starting with 3.5.0, according to the configuration documentation:

- If neither `PAPERLESS_URL` nor `PAPERLESS_ALLOWED_HOSTS` is set, Paperless-ngx continues to accept every host.
- If only `PAPERLESS_URL` is set, Paperless-ngx accepts only the hostname from that URL and `localhost`.
- `PAPERLESS_ALLOWED_HOSTS` adds further hosts to `localhost` and the hostname from `PAPERLESS_URL`. `localhost` remains allowed at all times because the Docker health check requires it.
- `PAPERLESS_ALLOWED_HOSTS=*` allows every host, even if `PAPERLESS_URL` is set.

Affected installations have `PAPERLESS_URL` set and are also reached through other names, such as an internal hostname, an IP address, or the service name in a container network through which other services call the API. Django rejects such requests after the update. Before updating, add all additional names to `PAPERLESS_ALLOWED_HOSTS`, for example:

```shell
PAPERLESS_URL=https://paperless.example.com
PAPERLESS_ALLOWED_HOSTS=paperless.intern.example.com
```

You should set the wildcard value `*` only if Paperless-ngx is accessible exclusively in an isolated network; the documentation explicitly recommends defining the allowed hosts for installations exposed to the open internet.

## 3.5.0: OIDC sign-in and sessions

Version 3.4.0 introduced the `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR` setting, which makes Paperless-ngx skip the local two-factor prompt when the OIDC provider reports a trusted second factor in the `amr` claim. According to issue #14692, the list of trusted values was always empty in 3.4.x: the code passed `mfa` as the delimiter instead of as the default value. Without the variable set, and even with `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR=mfa`, Paperless-ngx therefore continued to ask for the local second factor. PR #14693 fixes this in 3.5.0. The documented default value is `mfa`; password logins still require the code from Paperless-ngx.

This means that the behavior change described in the article on 3.4.0 only actually takes effect with 3.5.0: users with locally configured 2FA whose identity provider supplies `mfa` in the `amr` claim no longer see a second prompt after the update. Before updating, check which AMR values your provider sends. The documentation requires entering only values that mean a second factor was actually used during sign-in. At the same time, django-allauth is updated to 65.21.1 (PR #14672).

Until now, Paperless-ngx never removed expired sessions from the database. The change was prompted by issue #14721: with `PAPERLESS_AUTO_LOGIN_USERNAME`, a new session was created every 30 seconds and never deleted. Starting with 3.5.0, a scheduled task removes expired sessions (PR #14722). `PAPERLESS_SESSION_CLEANUP_TASK_CRON` controls the schedule; the default is `0 3 * * *`, meaning daily at 3:00 AM.

## 3.5.0: bulk operations, workflows, and tags

- **Asynchronous bulk operations:** All bulk operations on documents and bulk editing of metadata run through the task queue (PR #14773, #14711) and appear in the task list. According to the API documentation, queued bulk operations respond with `{"result":"OK","task_id":"…"}`; retrieve the result through `/api/tasks/?task_id=<uuid>`. Once the bulk task has completed, follow-up tasks such as reprocessing files are only scheduled, not completed. Scripts that continue working with the result immediately after the call should be adjusted accordingly.
- **Manual workflow trigger:** Workflows can be applied to selected documents using the new manual trigger type, in bulk editing and in document details (PR #14704).
- **Tag hierarchy:** Paperless-ngx now stores the tag hierarchy as an adjacency list and no longer uses django-treenode (PR #14612). The update includes a database migration for this. In a prototype with 6,000 tags on SQLite, the author measured substantially shorter times, around 1.3 instead of 4,409 milliseconds for creating a tag; the values have not been independently verified. When deleting a tag, Paperless-ngx additionally checks permissions on child tags (PR #14687).
- **Other new features:** Notes on documents are rendered as Markdown (PR #14626). A locally cached Hugging Face embedding model loads without a connection to the Hub (PR #14728), and AI chat also filters across more than 32,700 documents (PR #14754).

## 3.5.0: further bug fixes

- **OCR arguments:** The API rejects arguments that Paperless-ngx itself passes to OCRmyPDF (`input_file_or_options`, `output_file` and `sidecar`); they are ignored in `PAPERLESS_OCR_USER_ARGS` (PR #14734). Check whether your configuration contains these arguments.
- **Workflows:** The filename filter for scheduled workflows uses a regular expression compatible with PostgreSQL (PR #14783).
- **Mail:** Saved emails are written with CRLF line endings (PR #14745), and PDF/A conversion of mail archives runs only once (PR #14698).
- **Thumbnails:** Paperless-ngx once again generates thumbnails for images without an archive version (PR #14741).
- **Permissions and accounts:** Permissions for the configuration view have been standardized (PR #14709), Paperless-ngx locks the affected database rows when registering the first user (PR #14688), and passwords are displayed with a constant mask (PR #14784). The author of PR #14709 explicitly does not classify the change as a vulnerability.

## Dependencies and security

There is no project security advisory for 3.4.1, 3.4.2, or 3.5.0 (as of November 17, 2026). The latest advisory, GHSA-c8f3-pqr2-rr24 of October 16, 2026, concerns AI classification workflows on documents without owners and has been fixed since 3.3.2; details are in the article on 3.4.0. Systems on an earlier 3.x version also close this gap by updating to 3.5.0.

## Known issues

- **Local 2FA with OIDC despite AMR `mfa` (3.4.0 through 3.4.2):** fixed in 3.5.0 (issue #14692, PR #14693).
- **Known issues for 3.5.0:** The release notes list none (as of November 17, 2026). The notice about `PAPERLESS_ALLOWED_HOSTS` is an intentional behavior change.

## Recommended procedure

The target version is 3.5.0. Before updating, determine which hostnames are used to access Paperless-ngx and add them to `PAPERLESS_ALLOWED_HOSTS`; also check the values in `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`. Stop running processing tasks such as document consumption and create a backup, as required by the administration documentation. For Docker installations using an image from Docker Hub, stop the stack with `docker compose down`, pull the new image with `docker compose pull`, and start with `docker compose up`; the database migrations, including the new tag hierarchy migration, run automatically. For bare-metal installations, update the Python packages with `pip install -r requirements.txt` and run the migrations in the `src` directory with `python3 manage.py migrate`; afterwards, `python3 manage.py document_index reindex --if-needed` and `python3 manage.py document_llmindex migrate` update the search index and LLM index if needed. When updating directly from 3.3.x, also follow the notes in the article on 3.4.0, especially regarding `pdftoppm`. The article [Running Paperless-ngx with limited storage space](/blog/paperless-dokumente-clouddienst-auslagern) describes a Docker installation with external document storage.

## Sources

1.  [GitHub: Paperless-ngx Release v3.5.0](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.5.0): Release notes with information about `PAPERLESS_ALLOWED_HOSTS`, new features, and bug fixes.

2.  [GitHub: Paperless-ngx Release v3.4.2](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.4.2): Bug fixes and updated dependencies.

3.  [GitHub: Paperless-ngx Release v3.4.1](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.4.1): Bug fixes and lower idle memory usage.

4.  [GitHub API: Paperless-ngx Releases](https://api.github.com/repos/paperless-ngx/paperless-ngx/releases): Release dates for 3.4.1, 3.4.2, and 3.5.0.

5.  [Paperless-ngx: Configuration documentation 3.5.0](https://github.com/paperless-ngx/paperless-ngx/blob/v3.5.0/docs/configuration.md): New behavior of `PAPERLESS_ALLOWED_HOSTS`, default value of `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`, `PAPERLESS_SESSION_CLEANUP_TASK_CRON`, and ignored OCR arguments.

6.  [Paperless-ngx: Configuration documentation 3.4.2](https://github.com/paperless-ngx/paperless-ngx/blob/v3.4.2/docs/configuration.md): Previous default value `*` for `PAPERLESS_ALLOWED_HOSTS`.

7.  [GitHub: Issue #14692, MFA trusted AMR always empty](https://github.com/paperless-ngx/paperless-ngx/issues/14692): Cause and symptoms of the ineffective AMR default value in 3.4.x.

8.  [GitHub: PR #14722, automatic cleanup of sessions](https://github.com/paperless-ngx/paperless-ngx/pull/14722): Background on sessions that were never deleted with auto-login.

9.  [GitHub: PR #14612, tag hierarchy as adjacency list](https://github.com/paperless-ngx/paperless-ngx/pull/14612): Replacement of django-treenode, migration, and the author's measurements.

10.  [Paperless-ngx: API documentation 3.5.0](https://github.com/paperless-ngx/paperless-ngx/blob/v3.5.0/docs/api.md): Asynchronous bulk operations and retrieval through `task_id`.

11.  [Paperless-ngx: Administration documentation 3.5.0](https://github.com/paperless-ngx/paperless-ngx/blob/v3.5.0/docs/administration.md): Update procedure for Docker and bare metal.

12.  [GitHub: Paperless-ngx Security Advisories](https://github.com/paperless-ngx/paperless-ngx/security/advisories): Overview of advisories, none for 3.4.1 through 3.5.0.
