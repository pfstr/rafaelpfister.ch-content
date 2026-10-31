---
title: "Running Claude Code Securely on Your Own VPS"
navTitle: "VPS for Claude"
description: "A hardened Debian VPS keeps Claude Code sessions permanently available. This guide covers everything from user accounts and SSH keys to firewalls, data hygiene, tmux, and secure access from an iPhone."
date: "2026-07-21"
kategorie: "Claude"
timeToRead: "12 min read"
themen:
  - claude
translationOf: "claude-code-vps-debian-absichern"
slug: "securing-a-debian-vps-for-claude-code"
translationId: article-f932e9e537d7704a
translatedAt: 2026-10-31T03:41:36.938Z
translationReview: automatic
translationSourceHash: 0c584eb684bd92b439d97b20aae84cd15468c5c30d1e277d7fde7bd3fab9a37e
translationModel: gpt-5.6-terra
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/securing-a-debian-vps-for-claude-code
---

A Claude Code session on your own computer can end unexpectedly as soon as the laptop sleeps or the network connection drops. A VPS keeps running and can be accessed from multiple devices. At the same time, it is permanently connected to the public internet and is automatically scanned shortly after startup.

This guide combines both requirements: Claude Code remains available in a `tmux` session, while the Debian server offers only an SSH connection protected by keys to the outside world. The hardening is not Claude-specific and is also suitable for other publicly accessible Linux servers.

## Why a VPS can make sense

Compared to a purely local installation, the server offers three practical advantages:

- **Persistence.** In a `tmux` session, Claude keeps running even when the SSH connection is disconnected. A task that takes ten minutes or an hour finishes without the laptop having to stay open.
- **Accessibility.** The same session is accessible from a desktop, laptop, and iPhone. You start a task at your desk and check the result while on the go.
- **Data control.** You decide what resides on the server. No sync service, no accidentally backed-up credentials—provided you proceed carefully during migration (see below).

`tmux` is purely an availability and convenience feature, not a security measure. The real work lies in securing the system.

## Starting point

The basis is Debian 13 (Trixie), installed minimally, without a desktop and without additional network services. The provider supplies an upstream firewall that applies independently of the operating system. The goal is a server on which only SSH is reachable from the outside, and even that only with passphrase-protected keys.

## 1. Update the system

Immediately after installation, update all packages:

```bash
sudo apt update
sudo apt full-upgrade
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `update` | Reloads the package lists from all configured Sources |
| `full-upgrade` | Updates all packages and may also install new packages or remove existing ones |

</details>

Unlike `upgrade`, `full-upgrade` also resolves dependencies that require packages to be added or removed. On a fresh system, this is the right way to install all available security updates. Restart once after kernel updates.

## 2. Use a dedicated user instead of root

Working as root is unnecessarily risky: every typo affects the entire system, and direct root login is the first thing automated attacks try. Therefore, create a dedicated user (here `claude`) with sudo privileges for cases where they are needed:

```bash
sudo adduser claude
sudo usermod -aG sudo claude
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-a` | Append: adds to the user's group list instead of replacing it; valid only together with `-G` |
| `-G sudo` | Supplementary group(s) to which the user is added |
| `claude` | The affected user; with `adduser` the name of the account to create |

</details>

From now on, all administration is performed through `claude` and `sudo`, no longer through direct root access.

## 3. Passphrase-protected Ed25519 keys, one per device

Login should work exclusively through SSH keys, not passwords. Ed25519 is the current standard: short, fast, and cryptographically sound. The key point is that the key is generated on the client—that is, on the PC, not the server—and protected with a passphrase. The passphrase is the second line of defense should the private key ever fall into the wrong hands.

On the PC:

```bash
ssh-keygen -t ed25519 -C "pc-thinkpad"
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-t ed25519` | Key type, here the Ed25519 elliptic-curve algorithm |
| `-C "pc-thinkpad"` | Comment appended to the public key |

</details>

The comment (`-C`) identifies the device. This pays off later: generate a separate key for every device—one for the PC and a separate one for the iPhone. If a device is lost, remove its public key specifically from `~/.ssh/authorized_keys` without having to redeploy all other access.

Only the public key belongs on the server. The private key never leaves the device. In `authorized_keys`, only public keys remain, each with its device comment:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...pc  pc-thinkpad
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...ios iphone-15
```

Transfer the public PC key initially. As long as password login is still active, this is easiest with:

```bash
ssh-copy-id claude@SERVER
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `claude@SERVER` | User and target host; the default public key is appended there to `~/.ssh/authorized_keys` |

</details>

Then test that key-based login works before disabling password login in the next step. File permissions must be correct; otherwise sshd ignores the file: `~/.ssh` at `700`, `authorized_keys` at `600`.

## 4. Harden SSH: no root, no password

The server configuration is located in `/etc/ssh/sshd_config` and (on Debian 13) in drop-in files under `/etc/ssh/sshd_config.d/`. Changes belong in a dedicated drop-in file; this leaves the main file untouched and package updates overwrite nothing. Create file `/etc/ssh/sshd_config.d/99-haertung.conf`:

```text
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
```

This disables direct root login and password login. From now on, only someone who possesses a matching private key can get in. Before reloading, check the configuration syntax:

```bash
sudo sshd -t
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-t` | Test mode: checks configuration files and keys for validity without starting the service |

</details>

If `sshd -t` reports nothing, the file is valid. Only then reload:

```bash
sudo systemctl reload ssh
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `reload` | Instructs the service to reload its configuration without dropping existing connections |
| `ssh` | The target unit, here the OpenSSH service |

</details>

**Important:** Leave the existing SSH session open and test the new access in a second terminal. Only once key-based login demonstrably works there may you close the old session. This precaution reduces the risk of locking yourself out to virtually zero. A configuration error would otherwise cost you all access.

## 5. Move SSH to an uncommon port

The standard port 22 is probed by bots around the clock. Switching to a high, freely chosen port (in this example `61417`) causes most of this automated noise to go nowhere. This is explicitly not a security improvement in the strict sense: changing the port does not replace strong authentication; it only reduces log volume and scanning load. The key requirement from step 4 remains the actual security measure.

Which port to use is not arbitrary. IANA distinguishes three ranges: **0–1023 (well-known ports)** are reserved for standard services (SSH itself on 22, HTTP on 80, HTTPS on 443), require root to bind, and have no place on a self-chosen SSH port; scanners expect these ports, as do standard services installed later. **1024–49151 (registered ports)** are assigned to individual applications on request, such as 3306 (MySQL), 5432 (PostgreSQL), 6379 (Redis), or 8080/8443 as common HTTP alternatives; a randomly selected port from this range can easily conflict later with software that expects its registered port. **49152–65535 (dynamic/private ports)** are not assigned to any service according to IANA and are intended for temporary, private purposes—the right range for a permanent, self-chosen port.

One caveat remains: many Linux systems, including Debian, use part of this same range as source ports for their own outgoing connections (`net.ipv4.ip_local_port_range`, by default around 32768–60999). A permanently listening service does not truly conflict with this, as the kernel does not assign an already bound port, but a port above 60999 also avoids this theoretical ambiguity. The example in this article (`61417`) is deliberately in that range. Before switching, also use `ss -lntup` (see step 7) to check that the selected port is not already in use on your server.

There is a peculiarity in Debian 13: SSH may be started through systemd socket activation. If that is the case, the `Port` setting in `sshd_config` is simply ignored; the port must then be set on the socket. First check which case applies:

```bash
systemctl is-enabled ssh.socket
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `is-enabled` | Shows whether the unit is enabled for system startup |
| `ssh.socket` | The SSH service socket unit |

</details>

If the command responds with `enabled`, SSH runs through the socket. Then change the port there:

```bash
sudo systemctl edit ssh.socket
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `edit` | Creates a drop-in override file for the unit and opens it in the editor |
| `ssh.socket` | The socket unit to override |

</details>

Enter the following lines in the editor. The first, empty `ListenStream=` line clears the preset port 22; the second sets the new one:

```text
[Socket]
ListenStream=
ListenStream=61417
```

Then apply the changes:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `daemon-reload` | Reloads all unit files, including the override file just created |
| `restart ssh.socket` | Restarts the socket unit so it listens on the new port |

</details>

If socket activation is not active (`disabled`), add `Port 61417` to the drop-in file from step 4 instead, followed by `sudo sshd -t` and `sudo systemctl restart ssh`.

The same applies here: first open the new port in the firewall (next step), then connect and test it, and leave the old session open until access through the new port is confirmed.

## 6. Firewall: deny by default

The upstream provider firewall is the most effective boundary because it intercepts packets before they even reach the operating system. Two basic rules apply:

- **Set the default inbound action to DROP.** Everything not explicitly permitted is discarded silently, without feedback to the sender.
- **One single exception:** inbound TCP to destination port `61417`. Nothing else needs to be reachable from the outside.

Outbound traffic remains allowed. This is intentional: the server must download packages, synchronize time, and reach the API for Claude Code. Restrictive outbound filtering provides little additional protection for a single server but makes operation noticeably more cumbersome.

If you want additional defense in depth, you can duplicate the same rules on the host with `nftables` or `ufw`. For the setup described, the provider firewall is sufficient.

## 7. Check the attack surface

After hardening, check what the server actually offers to the outside. Two commands are enough. First: which services are listening on which addresses?

```bash
sudo ss -lntup
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-l` | Shows only listening sockets |
| `-n` | Numeric output: ports and addresses are not resolved to names |
| `-t` | Includes TCP sockets |
| `-u` | Includes UDP sockets |
| `-p` | Shows the process behind each socket; this requires `sudo` |

</details>

The address column is crucial: a service on `0.0.0.0` or `[::]` is reachable from the outside, while one on `127.0.0.1` or `[::1]` is local only. In the secured state, only SSH should appear publicly. Services such as `chronyd` (time synchronization) may appear, but only bound to local addresses. If `chronyd` listens exclusively on `127.0.0.1` and `::1`, it cannot be reached from the outside and is therefore not an issue.

Second: are there failed system services that indicate a configuration problem?

```bash
systemctl --failed
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `--failed` | Lists only units in a failed state |

</details>

The response should be `0 loaded units listed`, with not a single failed service. Failed units are not only an operational issue but potentially a security issue as well if they involve a partially started, misconfigured network service.

## 8. Install and run Claude Code

Claude Code does not require a separate Node.js runtime environment. Anthropic recommends the native installer (`curl -fsSL https://claude.ai/install.sh | bash`), which updates Claude Code itself in the background; for Debian, a signed apt repository is also available and is updated through normal system updates. After installation, authenticate the CLI fresh on the server rather than uploading local credentials (more on that shortly).

For persistent operation, use `tmux`:

```bash
tmux new -s claude
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `new` | Creates a new session |
| `-s claude` | Assigns the session name used to resume it later |

</details>

Start Claude within the session. Press `Ctrl-b`, then `d` to detach from the session without ending it; Claude keeps running. Return with:

```bash
tmux attach -t claude
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `attach` | Reconnects the terminal to a running session |
| `-t claude` | Selects the target session by name |

</details>

This lets a running task survive disconnected connections, device changes, and the laptop sleeping overnight.

## 9. Data hygiene during migration

The most sensitive part of moving to the server is not the technology but deciding what to bring along. Three rules:

- **No private keys on the server.** Only public keys belong in `authorized_keys`. Private keys remain on endpoint devices.
- **Do not copy credentials indiscriminately.** Sensitive local files such as a `.credentials.json` do not belong on the VPS without review. Instead, authenticate fresh on the server.
- **Put configuration in a migration folder first.** Do not write existing Claude memories and configuration directly to active configuration paths. First transfer them to a separate migration folder and review what should actually be retained. Anything no longer needed, such as old MCP entries or orphaned settings, is intentionally left behind rather than moved blindly.

## 10. Web previews through an SSH tunnel

For web previews, such as a local development server started by Claude, it is tempting to simply open another port. You should not do that. Every additional open port is additional attack surface. Instead, the preview runs through an encrypted SSH port tunnel: the service listens only locally on the server, and SSH forwards it to the client.

Make a service running locally on port 4321 accessible from the PC:

```bash
ssh -p 61417 -L 4321:localhost:4321 claude@SERVER
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-p 61417` | Port on which the SSH server listens (the one selected in step 5) |
| `-L 4321:localhost:4321` | Local port forwarding: connections on local port 4321 are forwarded through the tunnel to `localhost:4321` from the server's perspective |
| `claude@SERVER` | User and target host of the SSH connection |

</details>

Then open `http://localhost:4321` in the local browser. All traffic runs through the existing, authenticated SSH connection, without having to open even one additional port in the firewall.

## Access from an iPhone

Access while on the go follows the same security model as from the PC. You only need an SSH client with key management. Common options are **Termius**, **Blink Shell**, and **Secure ShellFish**; all can generate Ed25519 keys and store them in the iOS keychain, in some cases protected by Face ID.

The process corresponds to step 3, only on the iPhone:

1. Generate a dedicated Ed25519 key for the iPhone in the SSH client; do not copy the PC key. The private key remains in the device keychain.
2. Add the iPhone's public key as an additional line in `~/.ssh/authorized_keys` on the server, with a descriptive comment (`iphone-15`).
3. Create the connection in the client: server address, user `claude`, port `61417`, and the iPhone key for authentication.

This is exactly why a separate key per device is worthwhile: if the iPhone is lost, delete the one `iphone-15` line from `authorized_keys` on the server, and that device is locked out while PC access and all other keys continue unaffected.

After connecting, resume the running Claude session with `tmux attach -t claude` and continue where you left off at your desk. The port tunnel from step 10 also works from iOS; Termius and Secure ShellFish support port forwarding.

## Checklist

In summary, the complete process:

1. Debian 13 installed and fully updated with `apt full-upgrade`.
2. Dedicated user `claude` with sudo privileges; direct root login no longer used.
3. Passphrase-protected Ed25519 keys, one per device, with only public keys in `authorized_keys`.
4. sshd hardened: `PermitRootLogin no`, `PasswordAuthentication no`; checked with `sshd -t` before reloading, and left the existing session open until testing was complete.
5. SSH moved to port 61417, set on `ssh.socket` when socket activation is used, otherwise in the sshd configuration.
6. Provider firewall: inbound default DROP, sole exception TCP 61417; outbound allowed.
7. Attack surface checked with `ss -lntup` (only SSH public, `chronyd` local) and `systemctl --failed` (no errors).
8. Claude Code freshly authenticated on the server, running in a `tmux` session.
9. Data hygiene: no private keys or credentials on the server; configuration reviewed through a migration folder first.
10. No additional ports; web previews run through an SSH tunnel.

After this setup, only SSH on the specified port is reachable from the outside, and even there only with a passphrase-protected key. Claude Code runs independently of the endpoint device in `tmux`; web previews remain accessible through SSH tunnels without opening an additional port.

## Change log

- **October 25, 2026:** Corrected: Claude Code does not require a separate Node.js runtime environment; added the recommended native installer and the signed apt repository for Debian. Updated source links to the new Claude Code documentation addresses.

## Sources

1.  [OpenSSH Manual – sshd_config(5)](https://man.openbsd.org/sshd_config): Reference for all sshd directives, including `PermitRootLogin`, `PasswordAuthentication`, and `PubkeyAuthentication`.

2.  [Debian Wiki – SSH](https://wiki.debian.org/SSH): Debian-specific notes on SSH configuration, including the drop-in files under `/etc/ssh/sshd_config.d/`.

3.  [systemd.socket(5) – freedesktop.org](https://www.freedesktop.org/software/systemd/man/latest/systemd.socket.html): How socket activation and the `ListenStream=` directive work, relevant to changing the SSH port on Debian 13.

4.  [ss(8) – iproute2 Manpage](https://man7.org/linux/man-pages/man8/ss.8.html): Options for `ss` to list listening sockets along with their process and bind address.

5.  [Claude Code – Official documentation](https://code.claude.com/docs/en/overview): Installation, authentication, and operation of Claude Code.

6.  [Claude Code – Advanced setup](https://code.claude.com/docs/en/setup): System requirements, native installer, apt repository, and auto-updates.
