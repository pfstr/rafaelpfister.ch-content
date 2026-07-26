---
title: "Proton Drive on Linux: the state of the official client, the Rclone backend and the SDK"
navTitle: "Proton Drive & Linux"
description: "Windows and macOS have their sync clients, Linux is still waiting — and servers are a different story again. What works today with Rclone, what the official SDK and its CLI can already do, why machine credentials are the real gap, and what Proton has announced."
date: "2026-07-25"
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

Proton Drive has had sync clients for Windows and macOS since 2023, Linux came away empty. Whoever wants to use Proton Drive on a Linux desktop, let alone on a server, navigates a patchwork of community tooling, an official SDK in preview and announcements. This article sorts out the state as of July 2026, and does so from hands-on experience: I have just tested the Rclone backend extensively [as a document store for Paperless-ngx](/en/blog/offloading-paperless-documents-to-cloud-storage).

## The official client is announced, nothing more

In June 2026 Proton confirmed for the first time that a Linux client is actually in development: "from the ground up" on the new, unified SDK, with the same architecture as the Windows and macOS applications. There is no date and no beta; the roadmap mentions the "highly anticipated Linux app" without a time frame.

Important for perspective: this will be a **desktop sync client**. It solves the desk problem. On the server, though, where a service is supposed to read and write files straight from Proton Drive, a sync client is the wrong tool: it keeps a full local copy, which is exactly what you want to avoid when disk space is the constraint.

## The Rclone backend carries the load today

The workhorse on Linux has been Rclone with its `protondrive` backend for years: copying, syncing and, as the only tool at all, a **FUSE mount** that lets applications see the cloud as a directory. Two limitations belong here:

**It is beta on a reverse-engineered API.** Proton does not document its Drive API publicly; the backend is built on reverse engineering. In my test it worked reliably but throttled rapid call sequences with inconsistent directory listings.

**Unattended sign-in needs a trick.** Whoever feeds two-factor authentication a one-time code during setup sees the session end after about 35 minutes:

```text
422 POST https://drive-api.proton.me/auth/v4/2fa:
Invalid credentials (Code=8002)
```

Rclone tries to re-authenticate with the long-consumed code. The fix is storing the permanent TOTP secret (the Base32 value from the 2FA setup) as `otp_secret_key` in the Rclone configuration, obscured via `rclone obscure`. Rclone then generates the codes itself and runs indefinitely. That is less delicate than it sounds: against leaked passwords the second factor keeps protecting unchanged, just not against a compromise of the server. That case it never defended anyway, since the password sits there too. A **dedicated account** just for the service in question remains mandatory regardless.

How such a mount behaves in Docker environments, including two undocumented traps, is covered in the [dedicated article on Rclone in containers](/en/blog/rclone-mount-inside-docker-container).

## SDK and CLI point the way

In parallel, Proton is rebuilding its applications on an **official SDK** (JavaScript and C#, with bindings for Swift and Kotlin). The repository is public, and recently a **command-line tool** appeared there. Its authentication model shows where things are heading:

- `auth login` opens the browser; sign-in happens normally, **including two-factor authentication**
- the session lands in the **operating system's secret store** (Keychain, Credential Manager, libsecret), renewed by the SDK itself
- after that: list files, upload, check shares, with machine-readable JSON output

No password on the command line, no TOTP secret in a configuration file. Exactly the model the Rclone backend lacks. But: the CLI can **not mount file systems**, the browser login fits headless servers poorly, and Proton explicitly declares the SDK not yet production-ready for third parties; general availability is targeted for late 2026 to early 2027.

## The real gap: machine credentials

The core of the problem sits one level below client or SDK: **Proton has no machine credentials.** No app passwords, no service accounts, no scoped tokens. Every piece of automation, from the backup script through the server mount to the CI job, has to work with the account's full credentials.

For comparison: with S3-compatible storage, access key pairs are the norm — revocable and restrictable to buckets or prefixes. Google and Microsoft have app passwords and service accounts. With Proton it is all or nothing: whoever gives a server access to one folder gives it the whole account.

Admittedly, this is not entirely simple for an end-to-end encrypted service, since unlike with S3, limited access here would also have to mean limited key material. The SDK sessions show, however, that Proton masters such constructs; a session already is a derived, revocable credential. An official "machine token for exactly this folder, read-only" would be the single biggest step forward for server use, well ahead of any client.

## Recommendation by use case

| Use case | State as of July 2026 |
|---|---|
| Desktop sync on Linux | Wait for the announced client; until then Rclone sync or the web interface |
| Server backup (uploading files) | Rclone `copy`/`sync`: works, factor in the beta status |
| File-system mount for services | Rclone `mount` with a stored TOTP secret and a dedicated account: the only way, [proven in practice](/en/blog/offloading-paperless-documents-to-cloud-storage) |
| Script automation with clean auth | Keep an eye on the SDK CLI; too early for production |

Linux users carry Proton Drive today with community tooling that reaches surprisingly far. What would turn "works" into "built for this" is official mount support and machine credentials, and both are still outstanding.

## Sources

1.  [OMG Ubuntu: Proton Drive client is (finally) coming to Linux](https://www.omgubuntu.co.uk/2026/06/proton-drive-linux-client) — the June 2026 confirmation that the Linux client is in development, without a date.

2.  [Proton: Product roadmaps for spring and summer 2026](https://proton.me/blog/2026-spring-summer-roadmaps) — the roadmap with the Linux client lacking a time frame and the SDK as the foundation of Proton's own apps.

3.  [ProtonDriveApps/sdk on GitHub](https://github.com/ProtonDriveApps/sdk) — the public SDK repository including the CLI with browser login and session in the secret store.

4.  [Proton Drive SDK preview](https://proton.me/blog/proton-drive-sdk-preview) — Proton's own assessment: not yet production-ready for third-party applications.

5.  [Rclone: Proton Drive](https://rclone.org/protondrive/) — the backend including the beta notice and the `otp_secret_key` option for unattended sign-in.
