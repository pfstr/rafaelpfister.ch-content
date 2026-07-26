---
title: "Proton Drive on Linux: status as of July 2026"
navTitle: "Proton Drive & Linux"
description: "The official Linux client has been announced but is not yet available. On servers, Proton Drive can currently be mounted with rclone; the new SDK points to the technical direction ahead. What's still missing is machine access scoped to individual folders or tasks."
date: "2026-07-26"
kategorie: "Proton Drive"
timeToRead: "8 min to read"
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

For Windows and macOS, Proton Drive has offered its own sync clients since 2023. On Linux there is so far only the web interface, community tooling, and an official SDK still in preview. On a server the situation is even harder, since neither a desktop sync client nor an interactive login fits well there.

This overview describes the state as of July 2026. Alongside the published roadmaps, it draws on a hands-on test of the rclone backend [as a document store for Paperless-ngx](/en/blog/offloading-paperless-documents-to-cloud-storage).

## The Linux client is announced, but still without a date

In June 2026 Proton confirmed explicitly for the first time that a Linux client is in development. It is being built on the new, unified SDK and is meant to share the same technical foundation as the Windows and macOS applications. There is no date yet, and no public beta.

Important for context: this will be a **desktop sync client**. It solves the problem for the desktop. For server applications, though, a sync client is the wrong tool, because a service needs to read and write files directly from Proton Drive. A sync client keeps a full local copy, exactly what you want to avoid when disk space is tight.

## Today rclone carries the practical work

On Linux, rclone with its `protondrive` backend is currently the most versatile tool. It can copy and sync files, and as the only available solution it can expose Proton Drive as a local directory via **FUSE mount**. Two limitations matter here:

**It is beta on a reverse-engineered API.** Proton does not document its Drive API publicly; the backend is built on reverse engineering. In my test it worked reliably but throttled rapid call sequences with inconsistent directory listings.

**For unattended operation, rclone asks for the TOTP secret.** The configuration wizard labels the field `otp_secret_key`. That means the permanent secret from the 2FA setup, not the six-digit code an authenticator app currently displays. rclone stores this value obscured and generates a valid TOTP code from it itself at every sign-in.

Anyone who accidentally enters a current one-time code can complete the first sign-in. The next re-authentication, however, fails with error 8002, because rclone cannot reuse the same code a second time.

That keeps the account protected against a password stolen in isolation. A compromised server, however, exposes both the password and the TOTP secret. A **dedicated Proton account** is therefore recommended for automated access.

How such a mount behaves in Docker environments, including two undocumented traps, is covered in the [dedicated article on rclone in containers](/en/blog/rclone-mount-inside-docker-container).

## The official SDK shows where things are headed

In parallel, Proton is rebuilding its applications on an **official SDK** for JavaScript and C#, with bindings for Swift and Kotlin. The public repository also contains a command-line tool. Its authentication model is cleaner than the rclone backend's:

- `auth login` opens the browser; sign-in happens normally, **including two-factor authentication**
- the session lands in the **operating system's secret store** (Keychain, Credential Manager, libsecret), renewed by the SDK itself
- after that: list files, upload and check shares, with machine-readable JSON output

That means the password and TOTP secret no longer have to sit in a configuration file. Three limits remain for server operation, though: the CLI **cannot mount a file system**, sign-in opens a browser, and Proton does not yet rate the SDK as production-ready for third-party applications. General availability is planned for late 2026 to early 2027.

## The real gap: machine credentials

The core of the problem sits one level below client or SDK: **Proton has no machine credentials.** No app passwords, no service accounts, no scoped tokens. Every piece of automation, whether backup script, server mount or CI job, has to work with the account's full credentials.

For comparison: with S3-compatible storage, access key pairs are the norm, revocable and restrictable to buckets or prefixes. Google and Microsoft have app passwords and service accounts. With Proton, on the other hand, it is all or nothing: give a server access to one folder and you give it the whole account.

To be fair, this is harder for an end-to-end encrypted service than for S3, because limited access would also have to mean limited key material. The SDK sessions show, however, that Proton can build such constructs. A session is already a derived, revocable credential. An official "machine token for exactly this folder, read-only" would be the single biggest step forward for server use, well ahead of any client.

## Recommendation by use case

| Use case | State as of July 2026 |
|---|---|
| Desktop sync on Linux | Wait for the announced client; until then rclone sync or the web interface |
| Server backup (uploading files) | rclone `copy`/`sync`; works, factor in the beta status |
| File-system mount for services | rclone `mount` with a stored TOTP secret and a dedicated account; the only [proven-in-practice way](/en/blog/offloading-paperless-documents-to-cloud-storage) |
| Script automation with clean auth | Keep an eye on the SDK CLI; too early for production |

On the Linux desktop, you can either wait for the announced client or use rclone for now. On servers, rclone remains the only practical mount solution. A working stopgap only becomes a solid platform once Proton offers scoped machine credentials and an officially supported mount, though.

## Sources

1.  [OMG Ubuntu: Proton Drive client is (finally) coming to Linux](https://www.omgubuntu.co.uk/2026/06/proton-drive-linux-client): the June 2026 confirmation that the Linux client is in development, without a date.

2.  [Proton: Product roadmaps for spring and summer 2026](https://proton.me/blog/2026-spring-summer-roadmaps): the roadmap with the Linux client lacking a time frame and the SDK as the foundation of Proton's own apps.

3.  [ProtonDriveApps/sdk on GitHub](https://github.com/ProtonDriveApps/sdk): the public SDK repository including the CLI with browser login and session in the secret store.

4.  [Proton Drive SDK preview](https://proton.me/blog/proton-drive-sdk-preview): Proton's own assessment: not yet production-ready for third-party applications.

5.  [rclone: Proton Drive](https://rclone.org/protondrive/): the backend including the beta notice and the `otp_secret_key` option for unattended sign-in.
