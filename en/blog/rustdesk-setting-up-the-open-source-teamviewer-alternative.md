---
title: "RustDesk: Setting Up the Open-Source TeamViewer Alternative"
navTitle: "Set Up RustDesk"
description: "RustDesk is open-source remote support software licensed under the AGPL, free of charge and self-hostable. Learn how to install the client on Windows (including unattended deployment via MSI), how connections work through the public rendezvous server, your own server, or a direct connection, which features are needed for day-to-day support, and where the limits of free use lie."
date: "2026-09-01"
kategorie: "Remote Support and Help Desk"
timeToRead: "9 min read"
themen:
  - fernwartung
produkte:
  - "rustdesk"
protokolle:
  - "haertung"
slug: "rustdesk-setting-up-the-open-source-teamviewer-alternative"
translationId: "article-425ae4b8d562ae41"
aiPrompt: |
  Du bist mein IT-Support-Assistent. Hilf mir, RustDesk als quelloffene TeamViewer-Alternative einzurichten: Client installieren, Verbindungsart wählen (öffentlicher Vermittlungsserver, eigener Server oder Direktverbindung über ein privates Netz), unbeaufsichtigten Zugriff absichern und die Grenzen der kostenlosen Nutzung einordnen.
translationOf: rustdesk-teamviewer-alternative
url: https://rafaelpfister.ch/en/blog/rustdesk-setting-up-the-open-source-teamviewer-alternative
translationSourceHash: f812fc4b04abe0aa92cca47b285a30a18f5cd1e99ab328593b224ee26051a7f3
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T08:48:16.644Z
translationReview: automatic
---

# RustDesk: Setting Up the Open-Source TeamViewer Alternative

TeamViewer and AnyDesk provide reliable remote support, but require a license for commercial use, and prices increase with the number of managed devices. RustDesk is an alternative licensed under AGPL-3.0: open source, free of charge, and with no licensing requirement. The client runs on Windows, macOS, Linux, Android, and iOS, as well as in the browser. It is written in Rust, with the interface built in Flutter.

The key difference from commercial products lies in connection brokering: RustDesk separates the client from the server infrastructure. You can use the free public rendezvous server, operate your own server, or establish a direct connection without any rendezvous server. This lets you run RustDesk from a single workstation to a self-hosted support platform, without connection data having to pass through a provider.

## The Three Connection Types

Before installing, you should decide on the connection type, as configuration and open ports depend on it.

| Connection type | How it works | When it makes sense |
|---|---|---|
| Public rendezvous server | Two clients find each other via the ID (a nine-digit number) on the RustDesk server, and the connection runs directly or through a relay | Quick start, testing, occasional private support |
| Own server (self-hosted) | You operate the server components `hbbs` (rendezvous) and `hbbr` (relay) yourself, and all clients enter their address | Commercial use, many devices, full control over data |
| Direct connection (Direct IP Access) | The client connects directly to the remote system's IP address without a rendezvous server | Both devices can reach each other on the same network or through a VPN |

The public server is explicitly intended for testing and private use. For production commercial operation, the project recommends running your own server, partly because the public service is throttled and carries no availability guarantee.

## Installation on Windows

Download the installer from the official source, the project's GitHub releases (`github.com/rustdesk/rustdesk`). Windows offers an executable file and an MSI package. For interactive installation, a double-click is sufficient. If you want to deploy RustDesk on multiple computers or in the background, use the MSI with a silent installation:

```powershell
msiexec /i rustdesk-1.4.9-x86_64.msi /qn /norestart
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `/i <paket>` | Installs the specified MSI package |
| `/qn` | No interface, no dialog boxes (silent) |
| `/norestart` | Prevents an automatic restart after installation |

</details>

The silent installation sets up the `RustDesk` service, which runs at system startup and enables unattended access. After installation, retrieve the device ID from the command line without opening the interface:

```powershell
& 'C:\Program Files\RustDesk\rustdesk.exe' --get-id
```

You can also set a permanent password for unattended access from the command line. Use a separate, sufficiently long password, not the user's sign-in password:

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

## Adding Your Own Server

If you operate your own rendezvous server, enter its address and public key on the clients. In the interface, these are listed in the network settings as ID Server, Relay Server, and Key. For mass deployment, configuration can also be supplied as a file or through environment variables, so every client starts preconfigured.

Your own server requires the two components `hbbs` and `hbbr`, which usually run as Docker containers. Both require open ports so clients can register and use a relay.

| Port | Protocol | Component and purpose |
|---|---|---|
| 21114 | TCP | Web interface of the Pro version (only available there) |
| 21115 | TCP | `hbbs`, NAT type test |
| 21116 | TCP and UDP | `hbbs`, registration (UDP) and connection setup (TCP) |
| 21117 | TCP | `hbbr`, relay traffic |
| 21118, 21119 | TCP | Support for web clients |

Open only the ports your connection type actually requires, and use the firewall to limit access to the networks from which support is provided.

## Direct Connection Without a Rendezvous Server

If both devices can reach each other on the same network or through a VPN, RustDesk works entirely without a rendezvous server. To do this, enable direct access on the target device (in the interface, under security, as "Enable direct IP access"; internally, the `direct-server` switch). The client then listens on standard port 21118 (TCP). In the connection window, enter the remote system's IP address instead of its ID.

Use the firewall to limit direct access to the network from which you connect. If access runs through a VPN, allow the port only for the VPN address range, not for the entire internet.

## Features for Day-to-Day Support

RustDesk covers the features needed for everyday remote support:

- Screen sharing and remote control of the keyboard and mouse, with monitor selection for multiple displays.
- Two-way file transfer through a split-pane window.
- Text chat during the session.
- Unattended access through a permanent password, for devices without a user present.
- Session recording as a video file, optionally automatic.
- TCP tunneling and forwarding to access individual services on the remote system locally.
- Address book and multiple saved devices, stored locally in the free version and shared server-side in the Pro version.

For attended support, this is important: by default, RustDesk asks on the remote side whether the connection should be accepted and indicates during the session that access is active. The person using the device is therefore aware of it. Only a permanent password for unattended access removes this prompt. Use unattended access only on devices whose users know that the software is installed and what it is for.

## Limitations and Boundaries

RustDesk replaces TeamViewer in many cases, but it has limitations you should know before using it:

- The public rendezvous server is throttled, has no availability guarantee, and is not intended for ongoing commercial operation. Anyone who needs reliable operation should self-host.
- Running your own server means operational effort: containers, open ports, certificates, and updates are your responsibility.
- A server-side shared address book, centralized user management, and the web interface for administration belong to the Pro version, which costs money beyond a certain number of devices. The client itself and basic operation remain free.
- Without a permanent password, unattended access is not possible. This is appropriate for attended support, but prevents spontaneous access to an unattended device.
- The range of features and stability on individual platforms, especially mobile devices, do not match commercial products in every respect. Check the features important to you before switching.
- Some security programs flag remote support software as potentially unwanted. If necessary, create an exception and document why the software is installed.

For private use and support of individual devices, the free version with the public server or a direct connection is sufficient. Once you manage many devices, work commercially, or need full control over data, you need your own server, with the corresponding operational effort in exchange for independence.

## Sources

1.  [RustDesk on GitHub](https://github.com/rustdesk/rustdesk): Source code, releases with installers, and the AGPL-3.0 license.

2.  [RustDesk documentation](https://rustdesk.com/docs/): Installation, own server, ports, and client configuration.

3.  [rustdesk-server on GitHub](https://github.com/rustdesk/rustdesk-server): Server components `hbbs` and `hbbr`, including the port overview for self-hosting.
