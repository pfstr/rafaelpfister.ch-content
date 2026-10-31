---
title: "Paperless-ngx 3.4.0: OIDC Login Without Duplicate 2FA, Thumbnails Without ImageMagick"
navTitle: "Paperless-ngx 3.4.0"
description: "Paperless-ngx 3.4.0 skips local 2FA when the OIDC provider reports a trusted second factor, generates PDF thumbnails with Poppler instead of ImageMagick, and updates several dependencies for security reasons. Before updating, check the new AMR setting and, on bare-metal systems, whether pdftoppm is installed."
date: "2026-10-30"
kategorie: "Paperless-ngx"
timeToRead: "5 min read"
themen:
  - paperless-ngx
produkte:
  - "paperless-ngx"
protokolle:
  - "releases"
slug: "paperless-ngx-3-4-0-oidc-login-without-duplicate-2fa-thumbnails-without-imagemagick"
translationId: article-57582656ad9c441b
translationOf: paperless-ngx-3-4-0
url: https://rafaelpfister.ch/en/blog/paperless-ngx-3-4-0-oidc-login-without-duplicate-2fa-thumbnails-without-imagemagick
translationSourceHash: fe1a5dd3d088e7e544b9448ecc7099233f8287856a5eded7af7488e06c8f290a
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T06:48:54.666Z
translationReview: required
---

# Paperless-ngx 3.4.0: OIDC Login Without Duplicate 2FA, Thumbnails Without ImageMagick

The Paperless-ngx project released version 3.4.0 on October 29, 2026, one week after 3.3.4. The feature release adds a setting that eliminates the local two-factor prompt for OIDC logins when the identity provider has already confirmed a second factor. Paperless-ngx now generates thumbnails for PDF documents with the Poppler tools; ImageMagick is no longer a dependency. It also includes enhancements for workflows and custom fields, changes to uploads and notifications in the web interface, eight bug fixes, and updates to frontend and Python dependencies that the project lists as security updates. The release does not mention a separate security advisory with a CVE or GHSA. It is aimed at all operators of Paperless-ngx 3.x, especially installations using OIDC authentication and bare-metal installations whose system packages you manage yourself.

## Versions at a glance

| Version | Date | Type | Key points |
|---|---|---|---|
| 3.4.0 | October 29, 2026 | Feature release | skip local 2FA for trusted OIDC AMR, PDF thumbnails via Poppler, ImageMagick removed, custom fields in workflow titles, dependency security updates |

## OIDC authentication: local 2FA for trusted AMR

When logging in through an OIDC provider, Paperless-ngx now evaluates the `amr` claim in the ID token (PR #14403). If it contains one of the values from `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`, Paperless-ngx skips its own two-factor prompt for that login. According to the configuration documentation, the variable's default value is `mfa`. The documentation lists Authelia, which reports `mfa`, and Pocket ID, which reports `phr` for passkeys, as examples. For one-time codes by email, Pocket ID sends the value `otp`; the documentation classifies this as untrusted.

```shell
PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR=mfa
```

Because the default value is already set, behavior changes after the update for users with locally configured 2FA as soon as their identity provider supplies `mfa` in the `amr` claim. Therefore, check which AMR values your provider actually sends and whether it sets those values only when a genuine second factor is used. The documentation explicitly requires entering only values that mean a second factor was actually used during login. At the same time, 3.4.0 updates django-allauth to 65.20 and oauthlib to 4.0 (PR #14570).

## PDF thumbnails with Poppler, ImageMagick removed

Previously, Paperless-ngx called ImageMagick (`convert -density 300`) to create a PDF thumbnail, rendering the first page through Ghostscript at 300 DPI. Starting with 3.4.0, `pdftoppm` from the Poppler tools renders the first page, while Pillow scales the image and saves it as WebP (PR #14420). Paperless-ngx calculates the resolution from the page size, which it determines with `pdfinfo`, at a maximum of 300 DPI. The Ghostscript fallback for thumbnails is removed, while qpdf repair remains. The PR author measured substantially shorter runtimes and lower memory usage: for an empty A0 document, approximately 0.21 rather than 12.9 seconds; these figures have not been independently verified.

PR #14440 removes ImageMagick completely. A Pillow-based helper now handles the remaining tasks, primarily removing an alpha channel from images; it saves WebP files losslessly. This means the ImageMagick policy that previously had to be adjusted for PDF processing is no longer necessary. Ghostscript remains installed because OCRmyPDF still needs it.

This has two implications for admins:

- **Bare metal:** `pdftoppm` must be available. If it is missing, Paperless-ngx displays a warning at startup. Compare the installed system packages with the updated dependency list in the installation documentation.
- **Configuration:** According to PR #14420, `PAPERLESS_CONVERT_MEMORY_LIMIT` and `PAPERLESS_CONVERT_TMPDIR` are deprecated and have no effect; `PAPERLESS_GS_BINARY` has also been marked deprecated. You can remove these entries from `paperless.conf` or the `.env` file.

## Workflows, custom fields, and web interface

- **Workflows:** Title templates in workflows can use custom fields (PR #14564). In workflow tag fields, parent and child tags can be selected independently of one another (PR #14400).
- **Custom fields:** Queries on date fields can be relative (PR #14549).
- **Uploads and notifications:** The navigation bar has an upload button (PR #14442). The button shows the number of active uploads and links to the overview, while the notification count decreases (PR #14501, #14500).
- **Interface:** Document details have been revised (PR #14485), the task badge loads faster (PR #14552), and the AI chat renders Markdown (PR #14409, #14415).

## Bug fixes

- The `document_exporter` failed when exporting to a folder with `--delete` if an outdated `manifest.json.tmp` was present (PR #14526). Anyone using the exporter with `--delete` for backups should check after updating whether exports complete fully again.
- When editing a document, the search index is updated only after workflow runs (PR #14558).
- If a barcode does not result in document separation, tags are no longer assigned with a delay (PR #14497).
- Canceled or expired tasks are displayed more clearly (PR #14566).
- The API schema view no longer triggers permission queries on documents (PR #14503), and the list type of filters for document objects in the API schema has been corrected (PR #14524).
- Long version labels are shortened in the dropdown, and dropdown sizes have been standardized (PR #14578, #14510).

## Dependencies and security

The release notes list updates to frontend dependencies (PR #14581) as well as brotli, djangorestframework, filelock, msgpack, and setuptools (PR #14575) as security updates. The release notes do not state which vulnerabilities are fixed in these packages. The Angular components are at 22.2.2. There is no separate project security advisory for 3.4.0 (as of October 30, 2026).

The most recent advisory, GHSA-c8f3-pqr2-rr24 from October 16, 2026, affects versions 3.0 through before 3.3.2 and is fixed in 3.3.2. With AI features enabled, AI classification workflows for documents without an owner could transmit excerpts and metadata from other users' private documents to the configured AI provider (CVSS 3.1: 4.3, no CVE). Systems still running an older 3.x version also close this gap by updating to 3.4.0.

## Recommended procedure

The target version is 3.4.0. Before updating, stop ongoing processing such as document consumption and create a backup, as required by the administration documentation. For Docker installations using an image from Docker Hub, stop the stack with `docker compose down`, pull the new image with `docker compose pull`, and start it with `docker compose up`, with database migrations running automatically; once everything is working, start the stack in the background with `-d`. For bare-metal installations, first install the new system dependencies (especially Poppler with `pdftoppm`), update the Python packages with `pip install -r requirements.txt`, and run the migrations in the `src` directory with `python3 manage.py migrate`. Before the first OIDC login after the update, decide which values should be set in `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`. The article [Running Paperless-ngx with limited storage](/blog/paperless-dokumente-clouddienst-auslagern) describes a Docker installation with external document storage. The release notes for 3.4.0 do not mention known issues or breaking changes (as of October 30, 2026).

## Sources

1.  [GitHub: Paperless-ngx Release v3.4.0](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.4.0): Release notes with date, new features, bug fixes, and dependencies.

2.  [Paperless-ngx: Changelog](https://github.com/paperless-ngx/paperless-ngx/blob/main/docs/changelog.md): Changelog in the project documentation with entries for 3.3.x and 3.4.0.

3.  [GitHub: PR #14403, skip local 2FA for trusted OIDC AMR](https://github.com/paperless-ngx/paperless-ngx/pull/14403): New evaluation of the AMR claim during OIDC login.

4.  [Paperless-ngx: Configuration documentation](https://github.com/paperless-ngx/paperless-ngx/blob/main/docs/configuration.md): Description and default value of `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`.

5.  [GitHub: PR #14420, PDF thumbnails via Poppler](https://github.com/paperless-ngx/paperless-ngx/pull/14420): Switch to `pdftoppm`, deprecated settings, and the author's measurements.

6.  [GitHub: PR #14440, ImageMagick removed as a dependency](https://github.com/paperless-ngx/paperless-ngx/pull/14440): Replaces remaining ImageMagick calls with Pillow.

7.  [GitHub: Security Advisory GHSA-c8f3-pqr2-rr24](https://github.com/paperless-ngx/paperless-ngx/security/advisories/GHSA-c8f3-pqr2-rr24): Information disclosure in AI classification workflows, affecting 3.0 through before 3.3.2, CVSS 4.3.

8.  [Paperless-ngx: Administration documentation](https://github.com/paperless-ngx/paperless-ngx/blob/main/docs/administration.md): Update procedure for Docker and bare metal, exporter option `--delete`.

9.  [GitHub API: Paperless-ngx Releases](https://api.github.com/repos/paperless-ngx/paperless-ngx/releases): Release dates for 3.3.4 and 3.4.0.
