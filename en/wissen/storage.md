---
title: "Storage and Sync: Cloud Storage in Technical Terms"
blatt: "storage"
description: "How cloud storage works technically: object storage versus file system, access through sync, copy, and mount, client-side encryption, storage classes from hot to cold, and integrity checking of transferred data."
fakten:
  - { label: "Basic types", wert: "Object storage (S3-style) and file-based services" }
  - { label: "Access patterns", wert: "Sync, copy, mount (each with different guarantees)" }
  - { label: "General-purpose tool", wert: "rclone (more than 70 backends)", href: "https://rclone.org/docs/" }
  - { label: "Encryption", wert: "server-side, transport-level, client-side (zero-knowledge)" }
  - { label: "Storage classes", wert: "Hot, cool/infrequent access, cold/archive" }
  - { label: "Integrity", wert: "Checksums (MD5/SHA) depending on the backend" }
werbung: ["newsletter"]
ctaThemen: ["rclone", "proton-drive"]
---

# Storage and Sync: Cloud Storage in Technical Terms

Cloud storage looks uniform from the outside: files sit "in the cloud". Technically, very different systems hide behind that phrase, and the differences determine which tools work, which guarantees apply, and what changing providers costs.

## Object storage versus file system

**Object storage** (the S3 model) has no real folders, only objects with keys, metadata, and checksums in flat buckets. It scales almost without limit, is inexpensive, and forms the basis of most backup and archive services. **File-based services** (drive models) behave like a remote file system with folders, versions, and shares, and are designed for end-user clients. The distinction explains many quirks: renaming in object storage is a copy operation, change detection relies on checksums or timestamps, and not every service even exposes a documented API for third-party tools.

## Sync, copy, mount: three access patterns

**Copy** transfers data in one direction and leaves existing data in place; **sync** makes a destination match a source exactly and in doing so also deletes whatever is missing at the source; a **mount** presents remote storage as a local file system. The patterns differ fundamentally in risk: a sync command with source and destination swapped deletes data, and a mount simulates local semantics that the remote service cannot always deliver (latency, locking, partial writes). Tools such as **rclone** implement all three patterns uniformly across many services and add checksum verification (`rclone check`) and client-side encryption (the `crypt` backend).

## Encryption: where the key lives

Three levels need to be distinguished: **transport encryption** (TLS, standard everywhere), **server-side encryption** (the provider encrypts but holds the keys), and **client-side or end-to-end encryption** (only the customer holds the keys, and the provider sees random data). Zero-knowledge services and client-side tools trade convenience for this: server-side full-text search, previews, and web-based editing cannot work on encrypted content by design.

## Storage classes and retrieval

Providers tier prices by access pattern: **hot** for active data, **cool** for infrequent access, **cold/archive** for retention with retrieval fees and sometimes hours of lead time. For archive data, cold storage is cheaper by orders of magnitude; the price is that restoring costs money and time and therefore needs to be tested before anyone relies on it.
