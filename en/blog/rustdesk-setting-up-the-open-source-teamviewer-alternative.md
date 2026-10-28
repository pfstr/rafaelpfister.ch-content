---
title: "RustDesk: Setting Up the Open-Source TeamViewer Alternative"
navTitle: "Set up RustDesk"
description: "RustDesk is open-source remote support software under the AGPL, free of charge and self-hostable. Learn how to install the client on Windows (including unattended deployment via MSI), how connections work through the public rendezvous server, your own server, or a direct connection, which features everyday support needs, and where the limits of free use lie."
date: "2026-09-01"
kategorie: "Remote Support and Assistance"
timeToRead: "9 min read"
themen:
  - fernwartung
produkte:
  - "rustdesk"
protokolle:
  - "haertung"
slug: "rustdesk-setting-up-the-open-source-teamviewer-alternative"
translationId: "article-425ae4b8d562ae41"
translationOf: rustdesk-teamviewer-alternative
translationSourceHash: 3c44a4d5fc03c777c33f77525bcaebe75596ddde249f697041599b54e77cae2b
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T09:58:42.603Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/rustdesk-setting-up-the-open-source-teamviewer-alternative
---

# RustDesk: Setting Up the Open-Source TeamViewer Alternative

TeamViewer and AnyDesk provide reliable remote support, but they require a license for commercial use, and prices rise with the number of managed devices. RustDesk is an alternative licensed under AGPL-3.0: open source, free of charge, and without mandatory licensing. The client runs on Windows, macOS, Linux, Android, and iOS, as well as in the browser. It is written in Rust, with the interface built in Flutter.

The key difference from commercial products lies in connection brokering: RustDesk separates the client from the server infrastructure. You can use the free public rendezvous server, operate your own server, or establish a direct connection without a rendezvous server at all. This lets you use RustDesk from a single workstation to a self-hosted support platform, without connection data having to pass through a vendor.

## The Three Connection Types

Before installing, you should determine the connection type, as it affects the configuration and required open ports.

| Connection type | How it works | When it makes sense |
|---|---|---|
| Public rendezvous server | Two clients locate each other via the ID (a nine-digit number) on the RustDesk server; the connection is direct or goes through a relay | Quick start, testing, occasional private support |
| Your own server (self-hosted) | You operate the server components `hbbs` (rendezvous) and `hbbr` (relay) yourself, and all clients enter their address | Commercial use, many devices, full data sovereignty |
| Direct connection (Direct IP Access) | The client connects directly to the remote device's IP address without a rendezvous server | Both devices can reach each other on the same network or through a VPN |

The public server is expressly intended for testing and private use. For production commercial operation, the project recommends running your own server, also because the public service is rate-limited and does not provide an availability guarantee.

## Installation on Windows

Download the installer from the official source, the project's GitHub releases (`github.com/rustdesk/rustdesk`). Windows offers an executable file and an MSI package. Double-clicking is sufficient for interactive installation. If you want to deploy RustDesk to multiple computers or in the background, use the MSI with a silent installation. The file name includes the version number; version 1.5.0 has been current since September 30, 2026:

```powershell
msiexec /i rustdesk-1.5.0-x86_64.msi /qn /norestart
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `/i <paket>` | Installs the specified MSI package |
| `/qn` | No interface, no dialogs (silent) |
| `/norestart` | Prevents an automatic restart after installation |

</details>

The silent installation sets up the `RustDesk` service, which runs at system startup and enables unattended access. After installation, retrieve the device ID through the command line without opening the interface:

```powershell
& 'C:\Program Files\RustDesk\rustdesk.exe' --get-id
```

You can also set a permanent password for unattended access through the command line. Use a separate, sufficiently long password, not the user's sign-in password:

```powershell
& 'C:\Program Files\RustDesk\rustdesk.exe' --password "IhrLangesEinmalpasswort"
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `--get-id` | Outputs the device's nine-digit RustDesk ID |
| `--password <wert>` | Sets the permanent password for unattended access |
| `--silent-install` | Installs the executable version (`.exe`) as a service without an interface |

</details>

## Entering Your Own Server

If you operate your own rendezvous server, enter its address and public key on the clients. In the interface, these are listed in the network settings as ID Server, Relay Server, and Key. For mass deployment, configuration can also be supplied as a file or through environment variables, so every client starts preconfigured.

Your own server requires the two components `hbbs` and `hbbr`, which usually run as Docker containers. Both require open ports so clients can register and use a relay.

| Port | Protocol | Component and purpose |
|---|---|---|
| 21114 | TCP | Web interface of the Pro edition (only available there) |
| 21115 | TCP | `hbbs`, NAT type test |
| 21116 | TCP and UDP | `hbbs`, registration (UDP) and connection establishment (TCP) |
| 21117 | TCP | `hbbr`, relay traffic |
| 21118, 21119 | TCP | Support for web clients |

Open only the ports your connection type actually needs, and use the firewall to restrict access to the networks from which support is provided.

## Direct Connection Without a Rendezvous Server

If both devices can reach each other on the same network or through a VPN, RustDesk works without a rendezvous server altogether. To do so, enable direct access on the target device (in the interface, under Security, as "Enable IP direct access"; internally, the `direct-server` switch). The client then listens on the default port 21118 (TCP). In the connection window, enter the remote device's IP address instead of its ID.

Use the firewall to limit direct access to the network from which you connect. If access is through a VPN, open the port only for the VPN address range, not for the entire internet.

## Features for Everyday Support

RustDesk covers the features needed for everyday remote support:

- Screen sharing and remote control of the keyboard and mouse, with monitor selection for multiple displays.
- Bidirectional file transfer through a split-pane window.
- Text chat during the session.
- Unattended access through a permanent password, for devices without a user present.
- Session recording as a video file, automatically if desired.
- TCP tunneling and forwarding to access individual services on the remote device locally.
- An address book and multiple saved devices, stored locally in the free edition and shared server-side in the Pro edition.

For assisted support, this is important: by default, RustDesk asks the person on the other end to accept the connection and indicates during the session that access is active. The person at the device is therefore aware. Only a permanent password for unattended access removes the prompt. Use unattended access only on devices whose users know that the software is installed and what it is for.

## Limitations and Boundaries

RustDesk replaces TeamViewer in many cases, but it has limitations you should know before using it:

- The public rendezvous server is rate-limited, has no availability guarantee, and is not intended for ongoing commercial operation. If you need reliable operation, self-host.
- Operating your own server means administrative overhead: containers, open ports, certificates, and updates are your responsibility.
- A server-side shared address book, centralized user management, and the management web interface are part of the Pro edition, which requires payment above a certain number of devices. The client itself and basic operation remain free.
- Without a permanent password, unattended access is not possible. This is appropriate for assisted support, but it prevents spontaneous access to an unoccupied device.
- The breadth of features and stability on individual platforms, especially mobile devices, do not match commercial products in every respect. Check the features that matter to you before switching.
- Some security programs flag remote support software as potentially unwanted. Add an exception if necessary and document why the software is installed.

For private use and support of individual devices, the free edition with the public server or a direct connection is sufficient. Once you manage many devices, work commercially, or need full data sovereignty, you need your own server, with the corresponding administrative overhead in exchange for independence.

## Change Log

- **October 25, 2026:** Updated the installation example for RustDesk 1.5.0 of September 30, 2026.

## Sources

1.  [RustDesk on GitHub](https://github.com/rustdesk/rustdesk): Source code, releases with installers, and the AGPL-3.0 license.

2.  [RustDesk documentation](https://rustdesk.com/docs/): Installation, your own server, ports, and client configuration.

3.  [rustdesk-server on GitHub](https://github.com/rustdesk/rustdesk-server): Server components `hbbs` and `hbbr`, including the port overview for self-hosting.
