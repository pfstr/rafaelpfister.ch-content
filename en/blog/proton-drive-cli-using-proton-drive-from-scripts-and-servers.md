---
title: "Proton Drive CLI: Using Proton Drive from Scripts and Servers"
navTitle: "Proton Drive CLI"
description: "Since June 2026, Proton has offered an official command-line tool for Proton Drive. This article describes commands, signing in on servers without a desktop, conflict strategies for scripts, and the limitations compared with Rclone."
date: "2026-10-01"
kategorie: "Proton Drive"
timeToRead: "9 min read"
themen:
  - proton-drive
produkte:
  - "proton-drive"
protokolle:
  - "storage"
  - "backup-dr"
related:
  - proton-drive-linux-status
  - rclone-mount-in-docker-container
slug: "proton-drive-cli-using-proton-drive-from-scripts-and-servers"
translationId: "article-97376b998fdaec4c"
translationOf: proton-drive-cli
translationSourceHash: ce73cccdecdcdea611f1ead28872dab00bc33a7106e2eaefd51bee4241d8631c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-27T09:14:25.949Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/proton-drive-cli-using-proton-drive-from-scripts-and-servers
---

On June 9, 2026, Proton released the **Proton Drive CLI**, an official command-line tool for Windows, macOS, and Linux. It is based on the same SDK as the official Drive apps, provides end-to-end encryption, and is available as a single executable file `proton-drive`. The source code is in the public SDK repository at `cli/`.

The CLI is intended for individual, time-bound operations: uploading files after a build, backing up a folder on a schedule, reviewing or revoking shares. It does not synchronize in the background or mount a file system. The current version is **0.9.0 from October 5, 2026**; the version number indicates that commands and options may still change (version 0.8.0 renamed the conflict strategies in an incompatible change).

Its place among the other Linux options (Rclone, the announced desktop client) is covered in the status article [Proton Drive on Linux](/blog/proton-drive-linux-status).

## Command overview

Commands are organized into groups: `proton-drive <gruppe> <befehl> [optionen] [argumente]`. Group names can be abbreviated as long as they remain unambiguous; `filesystem` also has the alias `fs`. Without arguments, it starts an interactive shell. Full help is available through `proton-drive help` or `proton-drive <gruppe> <befehl> --help`.

<details class="options-details">
<summary>Options at a glance</summary>

| Group / command | Effect |
|---|---|
| `auth login` / `auth logout` | Sign in through the browser; signing out deletes local credentials and caches |
| `filesystem list <pfad>` | List a folder's contents; `/` shows the root areas |
| `filesystem info` / `size` | Display an item's metadata or a folder's size including Trash contents |
| `filesystem upload` / `download` | Upload or download files and folders |
| `filesystem create-folder`, `rename`, `copy`, `move` | Create, rename, copy, move folders |
| `filesystem trash` / `restore` | Move to Trash or restore |
| `filesystem delete` / `empty-trash` | Permanently delete or empty `/trash` |
| `sharing status <pfad>` | Display members, pending invitations, and link settings |
| `sharing invite` / `remove` | Invite people by email or revoke access |
| `sharing set-url` / `remove-url` | Create, change, or remove a public link |
| `sharing leave` / `report` | Leave a share shared with you or report it as abuse |
| `invitation list` / `accept` / `reject` | Manage received invitations |
| `album …`, `photo timeline`, `photo upload`, `photo download` | Proton Photos: albums and timeline |
| `takeout run` | Export an offline copy of the account to a local folder (since 0.9.0) |
| `version` | Display CLI and SDK versions |
| `--json` (`-j`) | Machine-readable JSON output, for every command |
| `--verbose` (`-v`) | Log output directly in the console |
| `--help` (`-h`) | Help for the respective command |

</details>

Paths in Proton Drive are always POSIX paths, including on Windows. The root `/` contains virtual areas: `/my-files` (your files), `/devices` (backed-up computers), `/shared-by-me`, `/shared-with-me`, `/trash`, as well as the Photos areas `/photos`, `/albums`, `/photos-shared-by-me`, `/photos-shared-with-me`, and `/photos-trash`.

## Installation on Linux

Proton provides the builds on a dedicated download page, each with a SHA-512 checksum. Five variants are available for Linux:

| Build | Use case |
|---|---|
| `linux/x64` | Standard for current x86-64 systems |
| `linux/x64-baseline` | x86-64 without AVX2, such as NAS devices and older server CPUs |
| `linux/arm64` | ARM servers and single-board computers with glibc |
| `linux/x64-musl`, `linux/arm64-musl` | Distributions with musl rather than glibc, such as Alpine Linux and container images based on it |

If the standard build exits at startup with `Illegal instruction`, the CPU lacks the AVX2 extension; in that case, the `x64-baseline` build is the right choice. The file includes the Bun runtime and requires no further dependencies:

```bash
chmod +x proton-drive
sudo install -m 0755 proton-drive /usr/local/bin/proton-drive
proton-drive version
```

Without administrator privileges, simply copy the file to `~/.local/bin` if that directory is in `PATH`.

## Signing in, including on servers without a desktop

`auth login` does not ask for a password on the command line. The CLI tries to open a browser and also outputs the sign-in URL. You can open this URL **on another device**; the terminal waits until sign-in is completed there. Two-factor authentication proceeds normally in the browser. This also makes sign-in possible over SSH on a server without a graphical interface.

```bash
proton-drive auth login
```

After a successful sign-in, the CLI stores the session, not the password. The `PROTON_DRIVE_CREDENTIALS_STORE` environment variable determines where it is stored:

| Value | Storage location |
|---|---|
| `keychain` (default) | Operating system credential store: Windows Credential Manager, macOS Keychain, or libsecret on Linux (GNOME Keyring, KWallet) |
| `pass` | GPG-encrypted entry `ch.proton.drive/drive-sdk-cli/auth-session` in the [pass](https://www.passwordstore.org/) password manager |
| `unsafe_file` | Plain-text file `auth-session.json` in the data directory; according to Proton, for testing only |

On a server without a desktop session, there is usually no unlocked libsecret keyring. Version 0.6.0 introduced the `pass` option for this case. The user under which scripts run needs an initialized password store and a GPG key that `gpg-agent` can unlock without interactive input. The session must be found through the same variable on every invocation, so it must also be set in cron jobs and systemd units:

```bash
export PROTON_DRIVE_CREDENTIALS_STORE=pass
proton-drive auth login
```

Compared with Rclone, this is an improvement: the password and TOTP key are not stored on the server, and `auth logout` terminates access. However, the session still has the full scope of the account. It cannot be limited to individual folders or read-only access. For automated workflows, a separate Proton account therefore remains the more secure option.

On Linux, cache, application data, and logs are stored in the XDG directories (`~/.cache/proton-drive-cli`, `~/.local/share/proton-drive-cli`, `~/.local/state/proton-drive-cli`). `PROTON_DRIVE_CACHE_DIR` can place all three in a single directory, for example for a container with a mounted volume. By default, the CLI writes logs at the `DEBUG` level; `PROTON_DRIVE_LOG_LEVEL=WARNING` reduces the volume.

## Uploading and downloading in scripts

Interactively, the CLI asks what to do for every naming conflict. This is not possible in scripts: `--json` disables the interactive prompt. Therefore, always explicitly specify the conflict strategy for files and folders.

```bash
proton-drive filesystem upload --json \
  --file-conflict-strategy create-new-revision \
  --folder-conflict-strategy merge \
  --skip-thumbnails \
  /srv/export/berichte /my-files/backup
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `--json` (`-j`) | Output the result as JSON; disables interactive prompts |
| `--file-conflict-strategy` (`-f`) | Behavior when a file with the same name exists: `create-new-revision` (new version of the existing file), `rename` (append a suffix), `replace` (move remote file to Trash, upload local file), `skip` |
| `--folder-conflict-strategy` (`-d`) | Behavior when a folder already exists: `merge` (merge contents), `rename`, `replace`, `skip` |
| `--skip-thumbnails` (`-t`) | Do not generate thumbnails; saves processing time for images |
| `/srv/export/berichte` | Local source; multiple sources are possible |
| `/my-files/backup` | Destination folder in Proton Drive (last argument) |

</details>

`create-new-revision` is the right choice for backups: Proton Drive retains earlier versions of a file, and since version 0.7.0, the CLI automatically skips files with unchanged contents. However, the CLI does not reconcile: files deleted locally remain in Proton Drive. Anyone needing a mirror including deletions still needs `rclone sync`.

Downloading works in reverse. The strategies differ because the local side is overwritten here:

```bash
proton-drive filesystem download --json \
  --file-conflict-strategy remove \
  --folder-conflict-strategy merge \
  /my-files/backup/berichte /srv/restore
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `--file-conflict-strategy` (`-f`) | `rename`, `remove` (delete local file and download remote version), or `skip` |
| `--folder-conflict-strategy` (`-d`) | `merge`, `rename`, `remove`, or `skip` |
| `/my-files/backup/berichte` | Source in Proton Drive; multiple sources are possible |
| `/srv/restore` | Local destination folder (last argument) |

</details>

The CLI skips Proton Docs and Proton Sheets when downloading; they cannot currently be exported as files.

A regular upload can be scheduled with a systemd timer or Cron. The JSON output can then be evaluated with `jq`, for example to report to monitoring.

## Managing shares

For offboarding or audits, share management is often more useful than file transfer. `sharing status` displays all members, pending invitations, and public-link settings for an item:

```bash
proton-drive sharing status --json /my-files/projekte/kunde-a
```

An invitation with view permissions:

```bash
proton-drive sharing invite \
  --user person@example.com \
  --role viewer \
  /my-files/projekte/kunde-a
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `--user` (`-u`) | Email address of the invited person; can be specified multiple times |
| `--role` (`-r`) | Role, default `viewer`; additional roles according to `--help` (for example, `editor`) |
| `--message` (`-m`) | Message in the invitation email; sent in **plain text** |
| `--include-node-name` (`-n`) | Include the item's name in the invitation email; also plain text |
| `/my-files/projekte/kunde-a` | Item to share |

</details>

A public link with a password and expiration date:

```bash
proton-drive sharing set-url \
  --role viewer \
  --password 'Linkpasswort' \
  --expiration 2026-12-31 \
  /my-files/projekte/kunde-a/bericht.pdf
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `--role` | `viewer` (default) or `editor` |
| `--password` | Custom password for the link |
| `--expiration` | Expiration date in ISO format (`JJJJ-MM-TT`) |
| `/my-files/…/bericht.pdf` | Item for which the link is created or changed |

</details>

A password passed on the command line appears in shell history and is visible in the process list while executing. In scripts, it should therefore come from a variable or secret store. `sharing remove-url` removes the link without affecting direct members; `sharing remove --user …` revokes access for individual people.

## Takeout: Exporting the account

Since version 0.9.0 of October 5, 2026, the CLI includes the `takeout run` command. It exports an offline copy of the account to a local folder, optionally with `--include my-files`, `devices`, `photos`, and `revisions` (all earlier file versions). Without at least one `--include` specification, it exports nothing, and `revisions` requires `my-files` or `devices`. For each folder, it writes a `manifest.json` describing the export; it makes no changes to the account. The command does not yet support Proton Docs and Proton Sheets. It is the obvious route to a complete local backup of Proton Drive contents.

## Limitations compared with Rclone

| Requirement | Proton Drive CLI 0.9.0 | Rclone (`protondrive` backend) |
|---|---|---|
| Officially supported | Yes, by Proton, Open Source | No, reverse engineering, beta |
| Sign-in | Browser, including on another device; session in credential store or `pass` | Password and TOTP key in the configuration file |
| Uploading, downloading | Yes, with conflict strategies and versioning | Yes |
| Mirror including deletions (`sync`) | No | Yes |
| Mount file system (FUSE) | No | Yes |
| Shares, invitations, links | Yes | No |
| Proton Photos | Yes | No |
| Access with limited scope | No | No |

For backups, build artifacts, and share management, the CLI is the better choice because it is officially supported and does not require a stored password. Rclone remains necessary for a mount, such as one needed by a [Paperless document archive](/blog/paperless-dokumente-clouddienst-auslagern), and for mirrors including deletions. The biggest gap remains the same in both cases: Proton does not offer machine access that can be restricted to individual folders or read-only access.

## Change log

- **October 25, 2026:** Updated to version 0.9.0 from October 5, 2026. Added: the takeout run command is now released, including notes on the required --include specifications and the lack of support for Proton Docs and Sheets.

## Sources

1.  [Proton: Introducing Proton Drive CLI](https://proton.me/blog/proton-drive-cli): the June 9, 2026 announcement covering intended use and JSON output.

2.  [Proton Support: Using Proton Drive CLI](https://proton.me/support/drive-cli): instructions for downloading, signing in, and basic commands.

3.  [Proton Drive CLI: Downloads](https://proton.me/download/drive/cli/index.html): current version 0.9.0, all platform builds with SHA-512 checksums.

4.  [ProtonDriveApps/sdk: cli/README.md](https://github.com/ProtonDriveApps/sdk/blob/main/cli/README.md): environment variables, storage locations, credential stores, and the note on the `x64-baseline` build.

5.  [ProtonDriveApps/sdk: cli/CHANGELOG.md](https://github.com/ProtonDriveApps/sdk/blob/main/cli/CHANGELOG.md): version history from 0.4.2 to 0.9.0, including `pass` support (0.6.0), skipping unchanged files (0.7.0), and the Takeout command (0.9.0).

6.  [ProtonDriveApps/sdk: cli/src/commands](https://github.com/ProtonDriveApps/sdk/tree/main/cli/src/commands): source code for commands with options, conflict strategies, and the Takeout command.

7.  [Proton for Business: Proton Drive CLI](https://proton.me/business/drive/cli): Proton's business use cases, such as revoking shares when employees leave.

8.  [Rclone: Proton Drive](https://rclone.org/protondrive/): the community backend with mount and sync functionality for comparison.
