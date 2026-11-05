---
title: "RustDesk 1.5.1: Signed Updates, Trusted Devices V2, and Hardened File Transfers"
navTitle: "RustDesk 1.5.1"
description: "RustDesk 1.5.1 verifies desktop updates using signed metadata, introduces Trusted Devices V2, and hardens file transfers, the Windows installer, and auto-login. It also includes around 35 bug fixes, primarily for DRM and Wayland capture. Updating clients is recommended; the server remains unchanged."
date: "2026-11-02"
kategorie: "Remote Support and Assistance"
timeToRead: "5 min read"
themen:
  - fernwartung
produkte:
  - "rustdesk"
protokolle:
  - "releases"
slug: "rustdesk-1-5-1-signed-updates-trusted-devices-v2-and-hardened-file-transfers"
translationId: article-c95ef39dfcc09875
translationOf: rustdesk-1-5-1
url: https://rafaelpfister.ch/en/blog/rustdesk-1-5-1-signed-updates-trusted-devices-v2-and-hardened-file-transfers
translationSourceHash: aa77142cfee63f49b15c18d6d1aa69368ec9e6ac75dee970c2907402fc21fc91
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:34:30.682Z
translationReview: required
---

# RustDesk 1.5.1: Signed Updates, Trusted Devices V2, and Hardened File Transfers

RustDesk released client version 1.5.1 on November 1, 2026, just over a month after 1.5.0 (September 30, 2026). The release includes five new features, including verification of signed update metadata and a new version of Trusted Devices for two-factor authentication, three changes, and around 35 bug fixes. Several of them harden security-relevant processes: download path validation, protected installer handoff on Windows, a warning for unencrypted API servers, and more narrowly scoped auto-login. The vendor does not mention a CVE or Security Advisory for this release (as of November 2, 2026). The release is aimed at everyone running RustDesk clients on Windows, macOS, Linux, Android, or iOS, especially with their own server. Setting up and operating your own RustDesk server is covered in the article [RustDesk: Setting Up the Open-Source TeamViewer Alternative](/blog/rustdesk-teamviewer-alternative).

## Versions at a Glance

| Version | Date | Type | Key points |
|---|---|---|---|
| 1.5.1 | 11/01/2026 | Client release | signed update metadata, Trusted Devices V2, keyboard shortcuts, download path validation, Windows installer hardening, HTTP API server warning, fixes for DRM, Wayland, X11, and QSV |

The open-source server (`rustdesk-server`) is not affected by this release. Its latest version is 1.1.16, dated July 20, 2026.

## Signed Update Metadata

With PR #15914, the desktop updater on Windows and macOS obtains its information from signed release metadata. Each release includes the assets `rustdesk-update.json` (version, URL, filename, size, and SHA-256 for each artifact) and `rustdesk-update.json.sig` (Ed25519 signature). The client verifies the signature before evaluating the metadata and accepts download URLs only under the release from `rustdesk/rustdesk`. If verification fails, the update is aborted; there is no fallback to unsigned data. Downloaded installers are kept in a protected directory until launch and hashed again before execution with elevated privileges.

According to the PR, the signature proves authenticity and integrity, but not freshness: protection against replaying older, correctly signed releases (rollback) is not included. The portable Windows version continues to extract and launch from a user-writable directory; according to the PR, hardening for this has been moved to a separate branch.

## Trusted Devices V2

PR #16548 replaces the previous hardware-ID-based trust mechanism with tokens issued by the host. A new registration still requires the permanent password and TOTP; afterward, a Trusted Device token only saves the TOTP entry. The host stores only the SHA-256 hash of the token in encrypted form, while the controller binds its token to the host's verified public key. No trust option is offered for direct IP connections because the host identity is not verified there.

The following points from the PR are relevant for operation:

- **Validity:** configurable for 1, 7, 30, or 90 days, or indefinitely; the default is 90 days. Trust relationships can be renewed or removed individually in the settings, or deleted in bulk.
- **Revocation:** Changing the password, 2FA, or the Trusted Device setting revokes the affected tokens. The host maintains a maximum of 128 entries.
- **Mixed versions:** Existing, unexpired trust relationships remain valid after updating the host. New registrations require 1.5.1 on both sides; in mixed sessions, RustDesk prompts for TOTP and does not offer “Trust this device.”
- **Downgrade:** According to the PR, not safe because older versions cannot read imported trust relationships.
- **Known limitations:** A revocation is stored locally first and applied asynchronously to the root configuration; restarting before synchronization may restore an older trust relationship. The expiration date is based on the host's system clock.

Therefore, plan the update so that hosts and controllers move to 1.5.1 promptly, and avoid reverting to 1.5.0.

## Additional Security-Relevant Fixes

- **Download paths (PR #16601):** Downloads reject unsafe filenames and paths that escape the selected destination directory or traverse symbolic links. Remote file lists are checked before downloading, and overwriting existing files or resuming interrupted downloads requires confirmation.
- **Windows installer (PR #16554, #16562):** The scripts for elevated installer handoff are no longer stored in `System32`, but in a directory under Windows Temp with a restricted DACL (Administrators and SYSTEM). The service is controlled through native Service Control. One reason was that Bitdefender blocked the temporary batch script (Issue #16557).
- **Unencrypted API server (PR #16616):** The server settings display a warning if the API server is empty or uses a remote HTTP address. HTTPS and local addresses do not trigger a warning. The connection is not blocked.
- **Auto-login (PR #16520):** Automatic login now enters the operating system password only on the login or lock screen.
- **Privacy Mode (PR #16507):** A rejected Privacy Mode request no longer disables Privacy Mode for another connection.

If you operate your own server with a `http://` address as the API server, your users will see the new warning. Switch the API server to HTTPS before distributing 1.5.1 to avoid questions.

## New Features and Changes

In addition to the points already mentioned, 1.5.1 adds keyboard shortcuts (PR #14933), a button in the address book to open the web console, and direct file transfers to SAF folders on Android. Changes include encoding static image content after scrolling (sharper display) and MediaCodec refresh budgets. Processing of `ConfigUpdate` was removed.

## Bug Fixes by Platform

- **Linux, DRM capture:** correct size for rotated displays, no waking sleeping displays at startup, discarding duplicate frames, and a frame rate adapted to the encoder.
- **Linux, Wayland:** PipeWire capture without wait times, correct mapping of the portal stream to the output, keyboard layout during input, and an open portal request no longer blocks the server.
- **Linux, general:** correct active user under Flatpak, crash when closing the GTK window, sudo prompt without the incorrect “Sorry, try again.” and with support for sudo-rs, xrandr output names, RandR monitor queries under X11.
- **Windows:** reading `host=` from program paths with non-ASCII characters, explicitly named executable for the Privacy window, QSV encoder hangs.
- **macOS:** link to Screen Recording settings, mouse button release.
- **Mobile:** no more delayed mouse-down after releasing the floating button.
- **Network:** HTTP API error 407 through HTTPS proxies, IPv6 ID servers without brackets when deriving the API server, one error message instead of two when both WebRTC connection attempts fail.
- **Other:** deleted printers are retained, replay buffer in the terminal, keyboard navigation in the file list, loading peers without an app directory set, clicks on peer cards in the web client.

## Recommended Approach

The target version for all clients is 1.5.1. Update hosts and controllers within the same time frame whenever possible so that new Trusted Device registrations work; according to PR #16548, a downgrade to 1.5.0 is not safe. Before rollout, check whether your own API server is accessible via HTTPS. The open-source server does not need to be updated for this release. The release notes contain no information on known issues or support periods (as of November 2, 2026).

## Sources

1.  [GitHub: RustDesk Release 1.5.1](https://github.com/rustdesk/rustdesk/releases/tag/1.5.1): Release notes with date, new features, changes, and bug fixes.

2.  [GitHub: RustDesk Release 1.5.0](https://github.com/rustdesk/rustdesk/releases/tag/1.5.0): Previous version from September 30, 2026.

3.  [GitHub: PR #15914, verify signed update metadata](https://github.com/rustdesk/rustdesk/pull/15914): Signature verification process, missing rollback protection, portable version.

4.  [GitHub: PR #16548, Trusted Devices V2](https://github.com/rustdesk/rustdesk/pull/16548): Token model, compatibility with older versions, downgrade, and known limitations.

5.  [GitHub: PR #16601, Download path validation](https://github.com/rustdesk/rustdesk/pull/16601): Validation of filenames, paths, and symbolic links during downloads.

6.  [GitHub: PR #16554, Installer scripts in protected Temp directories](https://github.com/rustdesk/rustdesk/pull/16554): Move from System32, DACL, and connection to Bitdefender.

7.  [GitHub: PR #16616, Warning for insecure API server](https://github.com/rustdesk/rustdesk/pull/16616): Conditions and text of the new warning.

8.  [GitHub: RustDesk Security Advisories](https://github.com/rustdesk/rustdesk/security/advisories): No published advisories (as of November 2, 2026).

9.  [GitHub: rustdesk-server Releases](https://github.com/rustdesk/rustdesk-server/releases): Latest server version 1.1.16 from July 20, 2026.

10.  [RustDesk: Setting Up the Open-Source TeamViewer Alternative](/blog/rustdesk-teamviewer-alternative): Setting up the client and your own server.
