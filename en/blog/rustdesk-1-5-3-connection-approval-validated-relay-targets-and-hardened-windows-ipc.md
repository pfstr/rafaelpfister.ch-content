---
title: "RustDesk 1.5.3: Connection Approval, Validated Relay Targets, and Hardened Windows IPC"
navTitle: "RustDesk 1.5.3"
description: "RustDesk 1.5.3 adds passwordless connections approved by the remote side, accepts relay requests only for offered addresses, and authenticates IPC pipes on Windows. Server policies can no longer change sensitive settings. Updating all clients is recommended."
date: "2026-11-11"
kategorie: "Remote Maintenance and Support"
timeToRead: "5 min read"
themen:
  - fernwartung
produkte:
  - "rustdesk"
protokolle:
  - "releases"
  - "haertung"
slug: "rustdesk-1-5-3-connection-approval-validated-relay-targets-and-hardened-windows-ipc"
translationId: article-557eece28ec889f1
translationOf: rustdesk-1-5-3
url: https://rafaelpfister.ch/en/blog/rustdesk-1-5-3-connection-approval-validated-relay-targets-and-hardened-windows-ipc
translationSourceHash: 3d06a4983979b612a9c43bd0a3eacbb0e13d9692a772e95478c351673d707815
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:21:26.556Z
translationReview: required
---

# RustDesk 1.5.3: Connection Approval, Validated Relay Targets, and Hardened Windows IPC

On November 9, 2026, RustDesk released client version 1.5.3, eight days after 1.5.1. Version 1.5.2 is not listed on the release page. The release adds a new feature (establishing connections with approval from the remote side), changes relay logic, and includes 22 bug fixes. Several affect security-relevant processes: validation of relay addresses and relay UUIDs, authentication of named pipes on Windows, locked configuration keys in server policies, and once again, installer handoff on Windows. The vendor does not mention a CVE or Security Advisory for this (as of November 11, 2026). The release is aimed at everyone running RustDesk clients, especially on Windows and with their own server. The previous status with signed updates and Trusted Devices V2 is described in the article [RustDesk 1.5.1: Signed Updates, Trusted Devices V2, and File Transfer Hardening](/blog/rustdesk-1-5-1).

## Versions at a Glance

| Version | Date | Type | Key points |
|---|---|---|---|
| 1.5.3 | 11/09/2026 | Client release | Request approval, relay only to offered addresses, IPC authentication on Windows, locked keys in policies, installer fixes (F-Secure, junction), fixes for macOS, DRM, Wayland, Android, and iOS |

The open-source server (`rustdesk-server`) has not received a new release; the latest version remains 1.1.16 from July 20, 2026.

## New Feature: Request Approval

With PR #16784, a connection can be established without a RustDesk password. The remote side must accept it in a dialog. According to the PR, in this mode the client does not send a password from any source, including local storage or the address book. An explicitly supplied password is ignored. The option appears as the “Request approval” submenu in the ID dropdown and in peer card menus, each showing the connection types supported by the platform and remote side (connection, file transfer, and others).

For scripts and shortcuts, the PR specifies the command-line parameter `--request-approval` and the URI parameter `request-approval=true`. The mode persists when reconnecting, moving a session to another window, and making further connections from the remote toolbar. Normal connections retain their existing password behavior.

According to the PR, the following limitations apply:

- Available authentication methods are not negotiated with the remote side. If only password sign-in is permitted there, or restrictions apply at the login screen, these continue to apply.
- If a window already exists for the same peer and connection type, RustDesk activates it without changing the sign-in method. To create a new connection with approval, first close the existing session.
- With tunnel multiplexing, an already approved tunnel can be reused without prompting again.

The PR does not provide information on compatibility with older versions on the remote side.

## Relay Only to Offered Addresses

PR #16858 closes a gap in relay negotiation. Previously, the controlled side accepted the relay address and UUID from a forwarded `RequestRelay` message without validation and connected to the specified address. Now, the controlled side stores the relay addresses offered to it, as well as the relay it would select itself (at most 32 entries, valid for ten minutes from the last message). It discards requests using a different relay before name resolution. Both sides also reject UUIDs that are empty, longer than 128 bytes, or contain characters other than ASCII letters, digits, `-`, and `_`. According to the PR, this matches the validation hbbs already performs.

A rejected request receives no response, so the controller's relay attempt times out instead of showing an error message. A relay you remove from the configuration is no longer accepted no later than ten minutes after its last message.

For operators of their own server, one limitation is relevant: If hbbs runs with the deprecated option `--mask`, fallback to a relay after a failed WebRTC attempt is rejected when hbbs has rewritten the relay address for the LAN. This issue was still open at merge time. Check whether your hbbs instance uses `--mask` and test relay connections after the update.

## Additional Security-Relevant Fixes

- **Windows IPC (PR #16755):** According to the PR, other users could previously create additional server instances of the named pipes for main and service IPC, allowing them to intercept connections to RustDesk, read IPC data, or forge configuration responses. Clients now open the pipes without this permission and authenticate the server before exchanging data. Service pipes must be owned by SYSTEM. According to the PR, a denial of service remains possible if an attacker claims the pipe name first, and older versions with more permissive access rules cannot connect until updated.
- **Policies (PR #16696):** Policies distributed by the server via heartbeat can no longer change sensitive settings. Locked settings include `api-server`, `custom-rendezvous-server`, `relay-server`, `key`, `2fa`, `bot`, `enable-trusted-devices`, `trusted-device-expiration-days` and `websocket-port`. Settings that can still be configured include `approve-mode`, `verification-method`, `access-mode`, `whitelist`, `id-whitelist` and `allow-auto-update`. Rejected keys are logged without their values. The author explicitly describes the change as limited hardening: policy responses remain unauthenticated.
- **Windows Installer (PR #16889, #16908, #16871):** The named-pipe handoff introduced with 1.5.1 has been removed again because, according to the PR, it prevented installation and “Stop service” on systems with F-Secure DeepGuard (error `0x800700E8`). Protection for prepared files now relies on the restrictive DACLs from 1.5.1 and dedicated staging directories whose handles remain open during handoff. If `C:\Windows\Temp` is redirected through a junction, the installer uses the system's ProgramData folder instead of aborting with an error.
- **macOS (PR #16729, #16879):** Privacy Mode is disabled at the end of a session before locking, and the service daemon starts directly rather than through `sh`.
- **IP Allowlist (PR #16692):** Whitespace around IP allowlist entries is ignored.

If installation fails on Windows with F-Secure, test 1.5.3 on an affected device before a broad rollout. According to PR #16889, confirmation of the F-Secure fix by the reporter was still pending at merge time. If you have been distributing configuration values such as API, ID, or relay servers through policies, you must use another method going forward.

## Bug Fixes by Platform

- **Windows:** Uninstallation continues after configuration synchronization errors, service IPC is retried before the GUI exits, and the clipboard owner marker is written first.
- **Linux:** The login screen under greetd with Wayland is provided through DRM, unchanged screen content is not re-encoded under DRM, and unassigned Wayland capture errors display their actual message rather than “x11 expected”.
- **Android:** No more crash with clipboard content over 1 MB, frames are copied before the image is released, and Hebrew and Indonesian are correctly mapped from legacy locale codes.
- **iOS:** The splash screen follows the system Dark Mode setting.
- **Other:** Port forwarding keeps the KCP handle open and sends a close reason, sidebars in the acceptance window remain on the current monitor in Windows, audio input devices are searched only under input devices, and the minimum size of the Sciter main window is no longer scaled twice.

## Known Issues

- **Rollback protection for signed updates (from 1.5.1):** Open. The 1.5.3 release notes mention no change.
- **Portable version on Windows (from 1.5.1):** According to PR #15914, hardening was moved to a separate branch. The 1.5.3 release notes mention no change.
- **Trusted Devices V2 (from 1.5.1):** The known limitations (asynchronous revocation, expiration based on the host system clock) are not addressed in the 1.5.3 release notes.
- **Relay fallback with `--mask` (new):** Open according to PR #16858.
- **Pipe name claimed in advance (new):** Denial of service remains possible according to PR #16755.
- **Canceled UAC prompt (new):** According to PR #16889, prepared files remain in the protected directory and can be removed by an administrator.
- **Policy responses (new):** Still unauthenticated. In deployments using HTTP instead of HTTPS, the options that remain allowed can be manipulated, according to PR #16696.

## Recommended Approach

The target version for all clients is 1.5.3. Because, according to PR #16755, older versions cannot connect to hardened IPC pipes until updated, update all RustDesk components on a Windows device together. Before rollout, check whether you distribute server addresses or keys through policies, whether hbbs runs with `--mask`, and whether F-Secure is in use on target systems. The API server should be accessible over HTTPS, as already recommended for 1.5.1. The open-source server does not need to be updated for this release. The release notes do not provide information on support periods (as of November 11, 2026).

## Sources

1.  [GitHub: RustDesk Release 1.5.3](https://github.com/rustdesk/rustdesk/releases/tag/1.5.3): Release notes with date, new feature, change, and bug fixes.

2.  [GitHub: RustDesk Releases](https://github.com/rustdesk/rustdesk/releases): Version list without release 1.5.2.

3.  [GitHub: PR #16784, Request Approval](https://github.com/rustdesk/rustdesk/pull/16784): Process, parameters, and limitations of connections with approval.

4.  [GitHub: PR #16858, Relay Only to Offered Addresses](https://github.com/rustdesk/rustdesk/pull/16858): Allowlist for relay addresses, UUID validation, and limitation with `--mask`.

5.  [GitHub: PR #16755, Windows IPC Server Auth](https://github.com/rustdesk/rustdesk/pull/16755): Named pipe hardening, compatibility, and remaining risk.

6.  [GitHub: PR #16696, Locked Keys in Policies](https://github.com/rustdesk/rustdesk/pull/16696): Locked and still permitted settings, missing authentication of responses.

7.  [GitHub: PR #16889, Removal of Pipe Handoff in the Installer](https://github.com/rustdesk/rustdesk/pull/16889): Relation to F-Secure DeepGuard and behavior when a UAC prompt is canceled.

8.  [GitHub: PR #16908, Installer Directory Protection](https://github.com/rustdesk/rustdesk/pull/16908): Dedicated staging directories with open handles.

9.  [GitHub: PR #16871, Installer Staging with Junction](https://github.com/rustdesk/rustdesk/pull/16871): Fallback to ProgramData when Windows Temp is redirected.

10.  [GitHub: RustDesk Security Advisories](https://github.com/rustdesk/rustdesk/security/advisories): No published advisories (as of November 11, 2026).

11.  [GitHub: rustdesk-server Releases](https://github.com/rustdesk/rustdesk-server/releases): Latest server version 1.1.16 from July 20, 2026.

12.  [RustDesk 1.5.1: Signed Updates, Trusted Devices V2, and File Transfer Hardening](/blog/rustdesk-1-5-1): Previous release status.
