---
title: "Rclone: The Universal Tool for Cloud Storage"
blatt: "rclone"
description: "An overview of the command-line tool for cloud storage: remotes and backends, the core commands sync, copy and mount, client-side encryption with crypt, and integrity checking by checksum."
fakten:
  - { label: "Project", wert: "Rclone (open source, Go)", href: "https://rclone.org/" }
  - { label: "Purpose", wert: "Transferring, synchronizing, mounting and verifying cloud storage" }
  - { label: "Backends", wert: "more than 70 services and protocols (S3, Drive, SFTP, WebDAV …)", href: "https://rclone.org/overview/" }
  - { label: "Core commands", wert: "copy, sync, mount, check, lsd/lsl" }
  - { label: "Encryption", wert: "crypt backend (client-side, names and contents)" }
  - { label: "Platforms", wert: "Linux, macOS, Windows; also in containers" }
werbung: ["newsletter"]
ctaThemen: ["rclone"]
---

# Rclone: The Universal Tool for Cloud Storage

Rclone is to cloud storage what a Swiss Army knife is to everyday tasks: a single command-line tool that operates more than seventy storage services and protocols in a uniform way, from an S3 bucket through Google Drive to an SFTP server. It is the de facto standard whenever data has to be moved between local systems and clouds in a scriptable way.

## Remotes: the basic idea

Rclone abstracts every service as a **remote**: a named configuration with a backend type and credentials, created through the interactive `rclone config` dialog. After that, all services are addressed identically (`remote:path`), and commands work across backends, including cloud to cloud. Remotes can be stacked: a `crypt` remote is layered on top of any other one and encrypts file names and contents on the client side before they leave the machine; the service behind it sees only random data.

## The core commands and their semantics

**`copy`** transfers files, overwrites nothing newer and never deletes. **`sync`** makes the destination match the source exactly, including deletions; the direction is part of the command and warrants corresponding care (`--dry-run` shows in advance what would happen). **`mount`** exposes a remote as a file system and brings cloud storage to applications that only understand file paths; caching modes control the trade-off between compatibility and immediacy. **`check`** compares source and destination by checksum or size and answers the question of whether a transfer really was complete and intact. Everyday helpers such as `lsd`, `lsl`, `size` and `ncdu` round out the set.

## Operational characteristics

Rclone transfers in parallel, resumes interrupted runs idempotently and respects bandwidth and transfer limits (`--bwlimit`, `--transfers`). Filter rules include or exclude paths. For services with API quotas, Rclone throttles itself on request, which permits continuous operation alongside productive use. For unattended deployment there are structured logs and a remote control mode with an HTTP API; in containers Rclone runs as a sidecar or mount service, where mounts demand particular attention to startup order and permissions.

## Assessment

The tool reaches its limits where services offer no API or only unofficial ones, or where end-to-end encryption rules out access by third-party clients; there, solutions often appear only with a delay or by way of workarounds. Within its field, Rclone is regarded as mature, conservatively maintained and unusually well documented.
