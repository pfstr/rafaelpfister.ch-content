---
title: "Proton Drive CLI: Using Proton Drive from Scripts and Servers"
navTitle: "Proton Drive CLI"
description: "Since June 2026, Proton has offered an official command-line tool for Proton Drive. This article covers commands, signing in on servers without a desktop, conflict strategies for scripts, and its limitations compared with Rclone."
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
url: https://rafaelpfister.ch/en/blog/proton-drive-cli-using-proton-drive-from-scripts-and-servers
translationSourceHash: e71e82deea6466312d0d95edc2c96890fc0810219a21308b4cb9346df3f32341
translationModel: gpt-5.6-terra
translatedAt: 2026-10-02T09:57:33.327Z
translationReview: automatic
---

On June 9, 2026, Proton released the **Proton Drive CLI**, an official command-line tool for Windows, macOS, and Linux. It is based on the same SDK as the official Drive apps, provides end-to-end encryption, and is available as a single executable file `proton-drive`. The source code is available in the public SDK repository at `cli/`.

The CLI is intended for individual, time-bound tasks: uploading files after a build, backing up a folder on a schedule, checking or revoking shares. It does not synchronize in the background or mount a file system. The current version is **0.8.0 from August 13, 2026**; the version number indicates that commands and options may still change (version 0.8.0 renamed the conflict strategies in an incompatible change).

For how it fits among the other Linux options (Rclone, the announced desktop client), see the status article [Proton Drive on Linux](/blog/proton-drive-linux-status).

## Command overview

The commands are organized into groups: `proton-drive <gruppe> <befehl> [optionen] [argumente]`. Group names can be abbreviated as long as they remain unambiguous; `filesystem` also has the alias `fs`. Without arguments, it starts an interactive shell. Full help is available through `proton-drive help` and `proton-drive <gruppe> <befehl> --help`.

<details class="options-details">
<summary>Options at a glance</summary>

| Group / command | Effect |
|---|---|
| `auth login` / `auth logout` | Sign in through the browser; signing out deletes local credentials and caches |
| `filesystem list <pfad>` | List a folder's contents; `/` shows the root areas |
| `filesystem info` / `size` | Show an item's metadata or the size of a folder including Trash contents |
| `filesystem upload` / `download` | Upload or download files and folders |
| `filesystem create-folder`, `rename`, `copy`, `move` | Create folders, rename, copy, move |
| `filesystem trash` / `restore` | Move to Trash or restore |
| `filesystem delete` / `empty-trash` | Permanently delete or empty `/trash` |
| `sharing status <pfad>` | Show members, pending invitations, and link settings |
| `sharing invite` / `remove` | Invite people by email or revoke access |
| `sharing set-url` / `remove-url` | Create, edit, or remove a public link |
| `sharing leave` / `report` | Leave a share shared with you or report it as abuse |
| `invitation list` / `accept` / `reject` | Manage received invitations |
| `album …`, `photo timeline`, `photo upload`, `photo download` | Proton Photos: albums and timeline |
| `version` | Show CLI and SDK versions |
| `--json` (`-j`) | Machine-readable JSON output, for every command |
| `--verbose` (`-v`) | Log output directly in the console |
| `--help` (`-h`) | Help for the respective command |

</details>

Paths in Proton Drive are always POSIX paths, including on Windows. The root `/` contains virtual areas: `/my-files` (your own files), `/devices` (backed-up computers), `/shared-by-me`, `/shared-with-me`, `/trash`, plus the Photos areas `/photos`, `/albums`, `/photos-shared-by-me`, `/photos-shared-with-me`, and `/photos-trash`.

## Installation on Linux

Proton provides builds on a dedicated download page, each with an SHA-512 checksum. There are five variants for Linux:

| Build | Use case |
|---|---|
| `linux/x64` | Standard for current x86-64 systems |
| `linux/x64-baseline` | x86-64 without AVX2, such as NAS devices and older server CPUs |
| `linux/arm64` | ARM servers and single-board computers with glibc |
| `linux/x64-musl`, `linux/arm64-musl` | Distributions using musl instead of glibc, such as Alpine Linux and container images based on it |

If the standard build exits at startup with `Illegal instruction`, the CPU lacks the AVX2 extension; in that case, the `x64-baseline` build is the right choice. The file includes the Bun runtime and requires no additional dependencies:

```bash
chmod +x proton-drive
sudo install -m 0755 proton-drive /usr/local/bin/proton-drive
proton-drive version
```

Without administrator privileges, it is enough to copy the file to `~/.local/bin` if that directory is in `PATH`.

## Signing in, including on servers without a desktop

`auth login` does not request a password on the command line. The CLI attempts to open a browser and also displays the sign-in URL. This URL can be opened **on another device**; the terminal waits until sign-in has been completed there. Two-factor authentication proceeds normally in the browser. This also allows signing in over SSH on a server without a graphical interface.

```bash
proton-drive auth login
```

After a successful sign-in, the CLI stores the session, not the password. Its location is determined by the `PROTON_DRIVE_CREDENTIALS_STORE` environment variable:

| Value | Storage location |
|---|---|
| `keychain` (default) | Operating system key store: Windows Credential Manager, macOS Keychain, or libsecret on Linux (GNOME Keyring, KWallet) |
| `pass` | GPG-encrypted entry `ch.proton.drive/drive-sdk-cli/auth-session` in the [pass](https://www.passwordstore.org/) password manager |
| `unsafe_file` | Plain-text file `auth-session.json` in the data directory; according to Proton, only for testing |

A server without a desktop session generally lacks an unlocked libsecret keyring. Since version 0.6.0, `pass` has been available for this case. The user running the scripts needs an initialized password store and a GPG key that `gpg-agent` can unlock without interactive input. The session must be found through the same variable on every invocation, so it must also be set in Cron jobs and systemd units:

```bash
export PROTON_DRIVE_CREDENTIALS_STORE=pass
proton-drive auth login
```

This is an improvement over Rclone: the password and TOTP key are not stored on the server, and `auth logout` terminates access. However, the session still has the account's full scope. There is no restriction to individual folders or read-only access. For automated workflows, a separate Proton account therefore remains the safer option.

On Linux, cache, application data, and logs are stored in the XDG directories (`~/.cache/proton-drive-cli`, `~/.local/share/proton-drive-cli`, `~/.local/state/proton-drive-cli`). `PROTON_DRIVE_CACHE_DIR` can place all three in a single directory, for example for a container with a mounted volume. By default, the CLI writes logs at level `DEBUG`; `PROTON_DRIVE_LOG_LEVEL=WARNING` reduces their volume.

## Uploading and downloading in scripts

Interactively, the CLI asks what to do for every naming conflict. That is not possible in scripts: `--json` disables the interactive prompt. Therefore, always explicitly set the conflict strategy for files and folders.

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
| `--file-conflict-strategy` (`-f`) | Behavior when a file with the same name exists: `create-new-revision` (new version of the existing file), `rename` (append a suffix), `replace` (move remote file to Trash, upload local one), `skip` |
| `--folder-conflict-strategy` (`-d`) | Behavior when a folder already exists: `merge` (merge contents), `rename`, `replace`, `skip` |
| `--skip-thumbnails` (`-t`) | Do not generate thumbnails; saves processing time for images |
| `/srv/export/berichte` | Local source; multiple sources are possible |
| `/my-files/backup` | Destination folder in Proton Drive (last argument) |

</details>

`create-new-revision` is the right choice for backups: Proton Drive retains earlier versions of a file, and since version 0.7.0, the CLI automatically skips files with unchanged contents. However, the CLI does not synchronize: locally deleted files remain in Proton Drive. Anyone needing a mirror that includes deletions still needs `rclone sync`.

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

The CLI skips Proton Docs and Proton Sheets when downloading; they currently cannot be exported as files.

A regular upload can be scheduled with a systemd timer or Cron. The JSON output can then be evaluated with `jq`, for example to send a notification to monitoring.

## Managing shares

For offboarding or audits, share management is often more useful than file transfer. `sharing status` shows all members, pending invitations, and public-link settings for an item:

```bash
proton-drive sharing status --json /my-files/projekte/kunde-a
```

An invitation with viewer access:

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
| `--role` (`-r`) | Role, default `viewer`; additional roles according to `--help` (e.g., `editor`) |
| `--message` (`-m`) | Message in the invitation email; sent in **plain text** |
| `--include-node-name` (`-n`) | Include the item name in the invitation email; also plain text |
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

A password passed on the command line appears in shell history and is visible in the process list while execution is in progress. In scripts, it should therefore come from a variable or secret store. `sharing remove-url` removes the link without affecting direct members; `sharing remove --user …` revokes access for individual people.

## In progress: Takeout

Since September 10, 2026, the SDK repository has included another command, `takeout run`. It exports an offline copy of the account to a local folder, optionally with `--include my-files`, `devices`, `photos`, and `revisions` (all earlier file versions). For each folder, it writes a `manifest.json` describing the export; it does not modify the account. The command is not yet included in the released version 0.8.0. Once available, it will be the obvious route for a complete local backup of Proton Drive contents.

## Limitations compared with Rclone

| Requirement | Proton Drive CLI 0.8.0 | Rclone (`protondrive` backend) |
|---|---|---|
| Officially supported | Yes, by Proton, open source | No, reverse engineering, beta |
| Sign-in | Browser, including on another device; session in key store or `pass` | Password and TOTP key in the configuration file |
| Uploading, downloading | Yes, with conflict strategies and versioning | Yes |
| Mirror with deletions (`sync`) | No | Yes |
| Mount file system (FUSE) | No | Yes |
| Shares, invitations, links | Yes | No |
| Proton Photos | Yes | No |
| Limited-scope access | No | No |

For backups, build artifacts, and share management, the CLI is the better choice because it is officially supported and does not require a stored password. For a mount, such as one needed by a [Paperless document archive](/blog/paperless-dokumente-clouddienst-auslagern), and for mirrors with deletions, Rclone remains necessary for now. The largest gap is the same in both cases: Proton does not offer machine access that can be restricted to individual folders or read-only access.

## Sources

1.  [Proton: Introducing Proton Drive CLI](https://proton.me/blog/proton-drive-cli): the June 9, 2026 announcement covering use cases and JSON output.

2.  [Proton Support: Using Proton Drive CLI](https://proton.me/support/drive-cli): instructions for downloading, signing in, and basic commands.

3.  [Proton Drive CLI: Downloads](https://proton.me/download/drive/cli/index.html): current version 0.8.0, all platform builds with SHA-512 checksums.

4.  [ProtonDriveApps/sdk: cli/README.md](https://github.com/ProtonDriveApps/sdk/blob/main/cli/README.md): environment variables, storage locations, credential stores, and the note about the `x64-baseline` build.

5.  [ProtonDriveApps/sdk: cli/CHANGELOG.md](https://github.com/ProtonDriveApps/sdk/blob/main/cli/CHANGELOG.md): version history from 0.4.2 to 0.8.0, including `pass` support (0.6.0) and skipping unchanged files (0.7.0).

6.  [ProtonDriveApps/sdk: cli/src/commands](https://github.com/ProtonDriveApps/sdk/tree/main/cli/src/commands): source code for commands with options, conflict strategies, and the not-yet-released Takeout command.

7.  [Proton for Business: Proton Drive CLI](https://proton.me/business/drive/cli): Proton's business use cases, such as revoking shares when employees leave.

8.  [Rclone: Proton Drive](https://rclone.org/protondrive/): the community backend with mount and sync functionality for comparison.
