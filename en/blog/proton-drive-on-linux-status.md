---
title: "Proton Drive on Linux: The State of Play in July 2026"
navTitle: "Proton Drive & Linux"
description: "The official Linux client has been announced but is not yet available. On servers, Proton Drive can currently be mounted with Rclone; the new SDK indicates the technical direction. What is still missing is machine access restricted to individual folders or tasks."
date: "2026-07-26"
kategorie: "Proton Drive"
timeToRead: "8 min read"
themen:
  - "proton-drive"
  - "rclone"
related:
  - "offloading-paperless-documents-to-cloud-storage"
  - "rclone-mount-inside-docker-container"
translationOf: "proton-drive-linux-status"
slug: "proton-drive-on-linux-status"
url: "https://rafaelpfister.ch/en/blog/proton-drive-on-linux-status"
---

Proton Drive has offered dedicated sync clients for Windows and macOS since 2023. On Linux, there is currently only the web interface, community tools and an official SDK in preview. The situation is even more difficult on a server, where neither desktop sync nor interactive sign-in is a good fit.

This overview describes the state of play in July 2026. Alongside the published roadmaps, it is based on a practical test of the Rclone backend [as document storage for Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern).

## The Linux client has been announced, but has no date yet

In June 2026, Proton explicitly confirmed for the first time that it is developing a Linux client. It is being built on the new unified SDK and is intended to use the same technical foundation as the Windows and macOS applications. There is no release date or public beta yet.

Important context: this will be a **desktop sync client**. It solves the problem for desktop use. For server applications, however, a sync client is the wrong tool, because a service needs to read files directly from Proton Drive and write them there. A sync client maintains a complete local copy, precisely what you want to avoid when storage is limited.

## Rclone does the practical work today

On Linux, Rclone with its `protondrive` backend is currently the most versatile tool. It can copy and synchronise files and, as the only available solution, provide Proton Drive as a local directory via a **FUSE mount**. Two limitations are important:

**It is beta and uses a reverse-engineered API.** Proton does not publicly document its Drive API; the backend is based on reverse engineering. In testing, it worked reliably, but throttled rapid sequences of calls with inconsistent directory listings.

**For unattended operation, Rclone asks for the TOTP secret.** The configuration wizard labels the field `otp_secret_key`. This means the persistent secret from the 2FA setup, not the six-digit code currently displayed by an authenticator app. Rclone stores this value obfuscated and generates a valid TOTP code from it itself for every sign-in.

Anyone who accidentally enters a current one-time code can complete the initial sign-in. However, the next reauthentication fails with error 8002 because Rclone cannot use the same code again.

This keeps the account protected against an isolated stolen password. However, a compromised server exposes both the password and the TOTP secret. For automated access, a **dedicated Proton account** is therefore recommended.

How such a mount behaves in Docker environments, including two undocumented pitfalls, is covered in the [separate article on Rclone in containers](/blog/rclone-mount-in-docker-container).

## The official SDK shows where development is heading

In parallel, Proton is moving its applications to an **official SDK** for JavaScript and C#, with bindings for Swift and Kotlin. The public repository also contains a command-line tool. Its sign-in model is cleaner than that of the Rclone backend:

- `auth login` opens the browser; sign-in proceeds normally **including two-factor authentication**
- the session is stored in the **operating system's key store** (Keychain, Credential Manager, libsecret), and the SDK renews it itself
- after that: list files with machine-readable JSON output, upload files and check shares

This means that neither the password nor the TOTP secret needs to be stored in a configuration file. Three limitations nevertheless remain for server use: the CLI **cannot mount a file system**, signing in opens a browser, and Proton does not yet consider the SDK production-ready for third-party applications. Release is planned for late 2026 to early 2027.

## The real gap: machine access

The core of the problem lies one level deeper than client or SDK: **Proton has no machine access.** No app password, no service account, no token with a limited scope. Every automation, whether a backup script, server mount or CI job, must work with the account's full credentials.

By comparison, access-key pairs are standard for S3-compatible storage, revocable and restrictable to buckets or prefixes. Google and Microsoft offer app passwords and service accounts. With Proton, however, it is all or nothing: anyone who wants to give a server access to one folder gives it access to the entire account.

To be fair, this is more difficult with an end-to-end encrypted service than with S3, because limited access would also have to mean limited key material. The SDK sessions do show, however, that Proton can handle such constructs. A session is already a derived, revocable access method. An official “machine token for this specific folder, read-only” would be the single greatest improvement for server use, far ahead of any client.

## Recommendation by use case

| Use case | State in July 2026 |
|---|---|
| Desktop sync on Linux | Wait for the announced client; until then, use Rclone sync or the web interface |
| Server backup (uploading files) | Rclone with `copy` or `sync`; works, but account for its beta status |
| File-system mount for services | Rclone with `mount`, a stored TOTP secret and a dedicated account; the only [field-tested approach](/blog/paperless-dokumente-clouddienst-auslagern) |
| Script automation with clean sign-in | Keep an eye on the SDK CLI; still too early for production |

On the Linux desktop, you can wait for the announced client or use Rclone for now. On servers, Rclone remains the only practical mounting solution. However, a working workaround will only become a robust platform when Proton offers restricted machine access and an officially supported mount.

## Sources

1.  [OMG Ubuntu: Proton Drive client is (finally) coming to Linux](https://www.omgubuntu.co.uk/2026/06/proton-drive-linux-client): confirmation from June 2026 that the Linux client is in development, without a date.

2.  [Proton: Product roadmaps for spring and summer 2026](https://proton.me/blog/2026-spring-summer-roadmaps): the roadmap with the Linux client without a time frame and the SDK as the foundation of Proton's own apps.

3.  [ProtonDriveApps/sdk on GitHub](https://github.com/ProtonDriveApps/sdk): the public SDK repository, including the CLI with browser sign-in and a session stored in the key store.

4.  [Proton Drive SDK preview](https://proton.me/blog/proton-drive-sdk-preview): Proton's own assessment: not yet production-ready for third-party applications.

5.  [Rclone: Proton Drive](https://rclone.org/protondrive/): the backend including the beta notice and the `otp_secret_key` option for unattended sign-in.
