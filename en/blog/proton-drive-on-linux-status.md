---
title: "Proton Drive on Linux: The State of Play in October 2026"
navTitle: "Proton Drive & Linux"
description: "The official Linux client has been announced but is not yet available. The official Proton Drive CLI has been available for scripts and servers since June 2026; Proton Drive can still only be mounted with Rclone. What is missing is machine access limited to individual folders or tasks."
date: "2026-10-01"
kategorie: "Proton Drive"
timeToRead: "8 min read"
themen:
  - proton-drive
  - rclone
related:
  - proton-drive-cli
  - paperless-dokumente-clouddienst-auslagern
  - rclone-mount-in-docker-container
translationOf: "proton-drive-linux-status"
slug: "proton-drive-on-linux-status"
translationId: article-ca282447e0b9acff
translationModel: gpt-5.6-terra
translatedAt: 2026-10-02T10:01:22.424Z
translationReview: automatic
translationSourceHash: 73500e1be526e5e93bbd7bf789b1cc500976140cb4cb9542404e1e7a56b40ce0
url: https://rafaelpfister.ch/en/blog/proton-drive-on-linux-status
---

Proton Drive has offered its own sync clients for Windows and macOS since 2023. On Linux, there is currently the web interface, community tools, and, since June 2026, an official command-line application, but no sync client yet. The situation is even more difficult on a server, where neither desktop sync nor interactive sign-in is a good fit.

This overview describes the state of play as of October 1, 2026. It is based on the published roadmaps, the source code of the Proton Drive CLI, and a practical test of the Rclone backend [as document storage for Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern).

**Update from October 1, 2026:** The first version from July 26 described the command-line application only as a tool in the SDK repository. However, Proton had already officially released it as the **Proton Drive CLI** on June 9, 2026, with ready-made builds for Windows, macOS, and Linux. The relevant section and the recommendation table have been revised accordingly; details are in the separate article on the [Proton Drive CLI](/blog/proton-drive-cli).

## The Linux client has been announced, but there is still no date

In June 2026, Proton explicitly confirmed for the first time that a Linux client is being developed. It is being built on the new unified SDK and is intended to use the same technical foundation as the Windows and macOS applications. As of early October 2026, there is still neither a release date nor a public beta.

Important context: This will be a **desktop sync client**. It solves the problem for the desktop. For server applications, however, a sync client is the wrong tool because a service should read files directly from Proton Drive and write them there. A sync client maintains a complete local copy, exactly what you want to avoid when storage is limited.

## Rclone remains necessary for mounts and mirroring

On Linux, Rclone with its `protondrive` backend is currently the most versatile tool. It can copy and sync files and, as the only available solution, make Proton Drive available like a local directory through a **FUSE mount**. Two limitations are important:

**It is beta software using a reverse-engineered API.** Proton does not publicly document its Drive API; the backend is based on reverse engineering. In testing, it worked reliably but throttled rapid sequences of calls with inconsistent directory listings.

**For unattended operation, Rclone asks for the TOTP key.** The configuration wizard labels the field `otp_secret_key`. This means the permanent key from 2FA setup, not the six-digit code currently displayed by an authenticator app. Rclone stores this value obfuscated and generates a valid TOTP code from it for each sign-in.

Anyone who accidentally enters a current one-time code can complete the initial sign-in. However, the next reauthentication fails with error 8002 because Rclone cannot use the same code again.

This keeps the account protected against an isolated stolen password. A compromised server, however, exposes both the password and the TOTP key. A **dedicated Proton account** is therefore recommended for automated access.

How such a mount behaves in Docker environments, including two undocumented issues, is covered in the [separate article on Rclone in containers](/blog/rclone-mount-in-docker-container).

## The official CLI covers scripts and backups

On June 9, 2026, Proton released the **Proton Drive CLI**, a single executable file `proton-drive` for Windows, macOS, and Linux. It is based on the same SDK as the official apps; the source code is in the public SDK repository. The current version is 0.8.0 from August 13, 2026, with builds for x86-64 (including without AVX2), ARM64, and musl distributions such as Alpine.

The sign-in model is cleaner than that of the Rclone backend:

- `auth login` outputs a sign-in URL that can also be opened **on another device**; sign-in proceeds normally **including two-factor authentication**, and thus also works over SSH on a server without a desktop
- the session is stored in the **operating system's keychain** (Keychain, Credential Manager, libsecret) or, since version 0.6.0, in the password manager `pass`, which is more practical on servers without a desktop session
- after that: upload and download files, move them, send them to the trash, manage shares, invitations, and public links, and use Proton Photos; each with `--json` for machine-readable output

This means the password and TOTP key do not need to be stored on the server. For backups and build artifacts, the CLI is therefore the better choice than Rclone today. Two limitations remain: The CLI **cannot mount a file system** and **cannot create a mirror with deletions**; it uploads and downloads but does not synchronize. A command `takeout` for a complete local export is already included in the repository but has not yet been released.

The SDK itself still classifies Proton as not production-ready for third-party applications; release is planned for late 2026 to early 2027. The CLI is not affected because Proton publishes it itself.

## The real gap: machine access

The core problem is one layer below the client or SDK: **Proton has no machine access.** No app password, no service account, no token with limited scope. Every automation, whether a backup script, server mount, or CI job, must work with the account's full credentials.

For comparison: With S3-compatible storage, access key pairs are standard, revocable, and can be restricted to buckets or prefixes. Google and Microsoft offer app passwords and service accounts. With Proton, by contrast, it is all or nothing: Anyone who wants to give a server access to one folder gives it the entire account.

With an end-to-end encrypted service, this is more difficult than with S3 because limited access would also require limited key material. However, the CLI sessions show that Proton can handle such constructs. A session is already a derived, revocable access method, just with the full scope of the account. An official “machine token for exactly this folder, read-only” would be the single biggest improvement for server use, far ahead of any client.

## Recommendation by use case

| Use case | Status as of October 2026 |
|---|---|
| Desktop sync on Linux | Wait for the announced client; until then, use Rclone sync or the web interface |
| Server backup (upload files) | [Proton Drive CLI](/blog/proton-drive-cli) with `filesystem upload` and conflict strategy `create-new-revision`; officially supported, without a stored password |
| Mirror with deletions | Rclone with `sync`; account for beta status |
| File system mount for services | Rclone with `mount`, stored TOTP key, and a dedicated account; the only [field-tested approach](/blog/paperless-dokumente-clouddienst-auslagern) |
| Script automation, managing shares | Proton Drive CLI with `--json`; version 0.x, commands may still change |

On the Linux desktop, you can wait for the announced client or use Rclone for now. On servers, the official CLI now handles backups and automation; for a mount, Rclone remains the only practical solution. However, a working workaround will only become a robust platform once Proton offers limited machine access and an officially supported mount.

## Sources

1.  [OMG Ubuntu: Proton Drive client is (finally) coming to Linux](https://www.omgubuntu.co.uk/2026/06/proton-drive-linux-client): the June 2026 confirmation that the Linux client is in development, without a date.

2.  [Proton: Product roadmaps for spring and summer 2026](https://proton.me/blog/2026-spring-summer-roadmaps): the roadmap with the Linux client without a timeframe and the SDK as the foundation of its own apps.

3.  [ProtonDriveApps/sdk on GitHub](https://github.com/ProtonDriveApps/sdk): the public SDK repository including the CLI source code and changelog.

4.  [Proton: Introducing Proton Drive CLI](https://proton.me/blog/proton-drive-cli): the official release of the CLI on June 9, 2026.

5.  [Proton Drive CLI: Downloads](https://proton.me/download/drive/cli/index.html): current version 0.8.0 from August 13, 2026, with all platform builds.

6.  [Proton Drive SDK preview](https://proton.me/blog/proton-drive-sdk-preview): Proton's own assessment: not yet production-ready for third-party applications.

7.  [Rclone: Proton Drive](https://rclone.org/protondrive/): the backend including the beta notice and the option `otp_secret_key` for unattended sign-in.
