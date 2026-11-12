---
title: "Rclone 1.76.0: numerous security fixes, VFS fixes, and new backends"
navTitle: "Rclone 1.76.0"
description: "Rclone 1.76.0, released on November 6, 2026, closes security vulnerabilities in serve, rc, and several backends, including an auth-proxy vulnerability in serve sftp with CVSS 8.1, fixes five bugs in the VFS layer, and blocks HTTPS-to-HTTP redirects by default. Operators of mounts, serve services, and rc should update and review the changed defaults."
date: "2026-11-11"
kategorie: "Rclone"
timeToRead: "7 min read"
themen:
  - rclone
produkte:
  - "rclone"
protokolle:
  - "releases"
  - "storage"
slug: "rclone-1-76-0-numerous-security-fixes-vfs-fixes-and-new-backends"
translationId: article-aa7be1fdff77b8cc
translationOf: rclone-1-76-0
url: https://rafaelpfister.ch/en/blog/rclone-1-76-0-numerous-security-fixes-vfs-fixes-and-new-backends
translationSourceHash: 79bd08de9bbf74eb195b3bdc4093b8e92f84cd358eb2a4b0a950ad3109a08e56
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:21:09.547Z
translationReview: required
---

# Rclone 1.76.0: numerous security fixes, VFS fixes, and new backends

Rclone 1.76.0 was released on November 6, 2026, around four weeks after 1.75.2 (October 9, 2026). The changelog lists twenty entries in the Security section, including a vulnerability in the auth proxy of `rclone serve sftp` with CVSS 8.1, credential leaks on redirects in the Swift, NetStorage, Seafile, and Ulozto backends, and several symlink escapes in the `local` backend. It also includes five fixes in the VFS layer that powers `rclone mount` and the serve commands, enhancements to the S3 backend, two new backends (Dosya and Infomaniak kDrive), and the new `index` command. Some defaults are changing: HTTPS-to-HTTP redirects are rejected, Multipath TCP listeners are disabled, and glob patterns in filters are evaluated more strictly. The release is aimed at everyone running Rclone for mounts, synchronization, backups, or as a server with `rclone serve` and `rclone rcd`. Background information on mounts in containers is available in the article [Run Rclone mounts reliably in Docker](/blog/rclone-mount-in-docker-container).

## Version overview

| Version | Date | Type | Key points |
|---|---|---|---|
| 1.76.0 | 11/06/2026 | Feature release | Security fixes in serve sftp (CVSS 8.1), rc, lib/http, local, and four backends; Go 1.27.3; five VFS fixes; redirect protection `--allow-https-downgrade`; new Dosya and Infomaniak kDrive backends; `index` command |

The previous highest version was 1.75.2. No 1.76.x patch release existed as of November 11, 2026.

## Security fixes with advisories

The following vulnerabilities were published by the Rclone project as GitHub Security Advisories on November 6, 2026. All list 1.76.0 as the patched version, and none has a CVE ID assigned (as of November 11, 2026; marked as “CVE-PENDING” in the changelog).

- **GHSA-66qv-66mg-9h74, CVSS 8.1 (High):** With `rclone serve sftp` using `--auth-proxy`, the auth proxy cache key did not include the authentication type. This allowed a known public key to be reused as a password if the auth proxy handled key and password authentication differently, the attacker and victim used the same username and client IP address, and the cache entry was still valid (default: 300 seconds of inactivity). Identical client IPs can occur behind NAT, a bastion host, or a load balancer. According to the advisory, other serve protocols are not affected. The affected versions field lists “>= 1.51.0, <= 1.76.0,” while the patched version field lists 1.76.0; the fix entry in the 1.76.0 changelog is authoritative.
- **GHSA-vmgx-g9c5-669p, CVSS 4.2:** With `serve sftp`, `serve ftp`, `serve http`, and `serve webdav` using `--auth-proxy`, revocation in the auth proxy did not take effect while a response was cached. Starting with 1.76.0, revocation takes effect within five minutes. `serve s3` is not affected.
- **GHSA-hwj7-wp9g-wrv9, CVSS 5.3:** The response time of bcrypt verification made it possible to distinguish valid from invalid usernames with `--htpasswd` and `--user`. Affected components include `serve http`, `serve webdav`, `serve restic`, `serve s3`, `rcd` with `--rc-htpasswd`, and the metrics endpoint with `--metrics-htpasswd`, in versions 1.56.0 through 1.75.2. Passwords are not disclosed.
- **GHSA-f8p4-8g4j-q3m2, CVSS 5.5:** If the rc server ran without HTTP authentication, a client with access to the port could exhaust process memory with a large JSON request to an unauthorized method (denial of service, versions 1.46.0 through 1.75.2). Version 1.76.0 limits request size and rejects unauthorized calls before reading the body. The advisory lists HTTP authentication for rc and a body limit on the reverse proxy as workarounds; binding to localhost does not isolate local user accounts from one another, according to the advisory.
- **GHSA-m97j-7x65-wh93, CVSS 4.8:** The Swift, NetStorage, and Seafile backends forwarded credentials on redirects to another host or from HTTPS to HTTP. This is exploitable only if the configured endpoint itself redirects, for example through a proxy. The advisory recommends using only `https://` endpoints and no proxy that downgrades to HTTP.
- **GHSA-3xpx-hjg2-vrjh, CVSS 4.8:** The Ulozto backend sent the `X-Auth-Token` and `X-User-Token` headers along with redirects from download links. An attacker requires a network position from which they can redirect CDN traffic.
- **GHSA-r5wv-vqqh-2r9c, CVSS 4.0:** With `--links` and `--metadata`, the `local` backend could write `user.*` attributes outside the target directory through an injected symlink. Until updating, the advisory recommends not using both options together when third parties can influence the target.

Additionally, 1.76.0 fixes further cases in the `local` backend without a separate advisory, where `MkdirMetadata`, `Remove`, `Rmdir`, and `DirMove` could operate outside the root directory through symlinks.

## Dependencies and changed defaults

Rclone 1.76.0 is built with Go 1.27.3. The changelog cites CVE-2026-111943: Go adds a blocklist for compromised intermediate certificates in `crypto/x509` (CVSS 6.8 according to the CISA-ADP rating; affected: Go before 1.26.10 and 1.27.0 through before 1.27.3). In addition, `golang.org/x/net` was updated to v0.62.0 and `github.com/andybalholm/brotli` to v1.2.7, the latter because of CVE-2026-111497; background on this vulnerability is available in the article on [CVE-2026-111497 in the Brotli decoder](/blog/brotli-cve-2026-111497).

Multipath TCP listeners are disabled by default starting with 1.76.0. The changelog attributes this to CVE-2026-98432; the CVE entry describes a use-after-free in the Linux kernel TCP stack (CVSS 9.8, fixed among others in 6.1.190, 6.6.159, 6.12.113, and 6.18.56). The fix itself is delivered by distribution kernel updates; Rclone only reduces its own attack surface. The changelog does not mention a flag to re-enable it.

An overview of changes to default behavior:

- **Redirects:** Rclone rejects redirects from HTTPS to HTTP. The new `--allow-https-downgrade` flag allows them again.
- **Filters:** `[!...]` in glob patterns is now a negated character class; `[^...]` excludes `/`. Before using them in production, check filter files and `--include`/`--exclude` rules that use such classes with `--dry-run`.
- **local:** `--copy-links` no longer follows symlink loops. Without root privileges, Rclone now only sets ownership that a regular user is permitted to set. Insufficient disk space now aborts multi-threaded transfers with a fatal error.
- **Parameter validation:** Excessively large size specifications, NaN, and overflowing durations are rejected, as are `--bwlimit` time specifications without a colon or with a sign.
- **bisync:** `--max-delete` takes `--track-renames` into account; values above 100 are rejected.
- **operations:** `--immutable` now also applies to `copyto`, `moveto`, and copying individual files.
- **Deprecated:** `--rc-web-gui`, as well as the `--no-cleanup` option for `serve s3`, which is now an alias for the new `--directory-markers`.

You should run scripts and units that invoke Rclone with fixed parameters once after updating and check for error messages concerning rejected values.

## Mount and VFS cache

For `rclone mount`, the changelog lists one fix: VFS instances were not released after failed mounts or during unmounting. In the VFS layer shared by Mount and the serve commands, 1.76.0 fixes five bugs:

- A shrinking truncate on a non-cached file was lost.
- Renaming a directory while the directory cache was expiring caused a crash.
- Virtual entries remained in deleted directories.
- Following a rename that differed only in letter case or Unicode normalization, entries appeared twice.
- Uploads of a modified file did not start while the file continued to be read.

The last point affects mounts with VFS cache where a modified file continues to be read while it is to be uploaded. In such setups, monitor the upload queue after updating and perform a write and read test as described in the [test plan for Rclone mounts](/blog/cloud-mount-testen-dummy-pdfs).

## S3 backend and Proton Drive

In the S3 backend, Rclone no longer buffers uploads with Object Lock when the MD5 value of the source is known. The new backend command is `link` with signed overrides for response headers. Added regions include Fastly EU Central 1 and Scaleway IT-MIL, several Huawei OBS regions, and the vendors Autonomys Auto Drive, Clever Cloud Cellar, Infomaniak Swiss Backup, Neon Object Storage, and RelAix.Storage. Multi-threaded uploads calculate checksums in 1 MiB read operations, and the actual Storage Class is displayed after a restore. Fixes include `rmdir` at the root level with `--s3-directory-markers` and presigned URLs for GCS, Linode, and Ceph.

For `rclone serve s3`, 1.76.0 introduces the `--directory-markers` flag as well as changes to tags, ETags, multipart ETags, and conditional writes. It also fixes an issue that prevented buckets from being deleted after an aborted multipart upload.

The changelog for 1.76.0 contains no entry for the Proton Drive backend. The most recent fixes for it came in 1.75.0 (retry logic, hangs on validation errors, incorrect modification times) and 1.75.1 (files uploaded with 1.75.0 could not be read in Proton apps). The article [Proton Drive on Linux](/blog/proton-drive-linux-status) describes the status of Proton Drive support on Linux.

## Other new features

The two new backends are Dosya (Kaya) and Infomaniak kDrive. The new `index` command writes static directory listings to a remote and can re-index only changed directories on subsequent runs. Rclone supports CRC-32C as a hash, `sync` can control transfer order by patterns, and the directory view of `serve http` and `serve webdav` is sorted and redesigned. According to the changelog, the GUI is no longer considered beta. Statistics no longer count `totalBytes` twice for server-side copies, and `serve sftp` reports an error for `rmdir` on a file instead of deleting the file.

## Recommended approach

The target version is 1.76.0. Updating is particularly urgent on systems with `rclone serve sftp --auth-proxy`, with `rclone rcd` or `--rc` without HTTP authentication, with `--htpasswd`-protected serve services, and with remotes of type Swift, NetStorage, Seafile, or Ulozto. For installations using the official binaries, `rclone selfupdate` updates the executable and verifies the download by hash and signature; with `rclone selfupdate --check`, Rclone shows which version would be installed beforehand. For installations from deb or rpm packages, the documentation specifies `--package deb` and `--package rpm`, respectively. Running mount and serve processes do not use the new version until restarted. Afterwards, check filter rules with character classes, `--bwlimit` schedules, and setups that depend on HTTP redirects. The changelog and release page do not list any known issues for 1.76.0 (as of November 11, 2026).

## Sources

1.  [Rclone: Changelog](https://rclone.org/changelog/): Entries for 1.76.0 (November 6, 2026) and 1.75.0 through 1.75.2 with security fixes and Mount, VFS, S3, and Proton Drive changes.

2.  [GitHub: rclone v1.76.0](https://github.com/rclone/rclone/releases/tag/v1.76.0): Release page with binaries and a link to the changelog.

3.  [GitHub: rclone Security Advisories](https://github.com/rclone/rclone/security/advisories): Overview of the advisories published on November 6, 2026.

4.  [GHSA-66qv-66mg-9h74](https://github.com/rclone/rclone/security/advisories/GHSA-66qv-66mg-9h74): serve sftp, auth proxy cache without authentication type, CVSS 8.1.

5.  [GHSA-vmgx-g9c5-669p](https://github.com/rclone/rclone/security/advisories/GHSA-vmgx-g9c5-669p): Revocation in the auth proxy does not take effect with a cached response, CVSS 4.2.

6.  [GHSA-hwj7-wp9g-wrv9](https://github.com/rclone/rclone/security/advisories/GHSA-hwj7-wp9g-wrv9): Username discovery through bcrypt timing with `--htpasswd`, CVSS 5.3.

7.  [GHSA-f8p4-8g4j-q3m2](https://github.com/rclone/rclone/security/advisories/GHSA-f8p4-8g4j-q3m2): Memory exhaustion in the rc server through oversized JSON requests, CVSS 5.5.

8.  [GHSA-m97j-7x65-wh93](https://github.com/rclone/rclone/security/advisories/GHSA-m97j-7x65-wh93): Credential leaks on redirects in Swift, NetStorage, and Seafile, CVSS 4.8.

9.  [GHSA-3xpx-hjg2-vrjh](https://github.com/rclone/rclone/security/advisories/GHSA-3xpx-hjg2-vrjh): Token leak on redirects in the Ulozto backend, CVSS 4.8.

10.  [GHSA-r5wv-vqqh-2r9c](https://github.com/rclone/rclone/security/advisories/GHSA-r5wv-vqqh-2r9c): Symlink escape with `--links --metadata` in the local backend, CVSS 4.0.

11.  [CVE.org: CVE-2026-111943](https://www.cve.org/CVERecord?id=CVE-2026-111943): Blocklist for compromised intermediate certificates in Go `crypto/x509`, fixed in Go 1.27.3.

12.  [CVE.org: CVE-2026-98432](https://www.cve.org/CVERecord?id=CVE-2026-98432): Use-after-free in the Linux kernel TCP stack, CVSS 9.8, reason for disabling Multipath TCP listeners.

13.  [Rclone: rclone selfupdate](https://rclone.org/commands/rclone_selfupdate/): Documentation on self-updating, `--check` and `--package`.

14.  [CVE-2026-111497: vulnerability in the Brotli decoder](/blog/brotli-cve-2026-111497): Background on the Brotli vulnerability addressed by 1.76.0 through the dependency update.
