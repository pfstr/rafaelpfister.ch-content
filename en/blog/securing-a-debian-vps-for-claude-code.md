---
title: "Run Claude Code securely on your own VPS"
navTitle: "VPS for Claude"
description: "A hardened Debian VPS keeps Claude Code sessions permanently accessible. This guide covers everything from user accounts and SSH keys to firewalls, data hygiene, tmux and secure access from an iPhone."
date: "2026-07-21"
kategorie: "Claude"
timeToRead: "12 min read"
themen:
  - "claude"
translationOf: "claude-code-vps-debian-absichern"
slug: "securing-a-debian-vps-for-claude-code"
url: "https://rafaelpfister.ch/en/blog/securing-a-debian-vps-for-claude-code"
---

On your own computer, a Claude Code session will eventually end unintentionally when the laptop goes to sleep or the network connection drops. A VPS keeps running and is accessible from multiple devices. At the same time, it is permanently connected to the public internet and is automatically scanned shortly after startup.

This guide combines both requirements: Claude Code remains available in a `tmux` session, while the Debian server offers only a key-protected SSH connection to the outside world. The hardening is not specific to Claude and is also suitable for other publicly accessible Linux servers.

## Why a VPS can make sense

Compared with a purely local installation, the server offers three practical advantages:

- **Persistence.** In a `tmux` session, Claude keeps running even if the SSH connection is disconnected. A task that takes ten minutes or an hour will complete without the laptop needing to stay open.
- **Accessibility.** The same session is accessible from a desktop, laptop and iPhone. You can start a task at your desk and check the result while out and about.
- **Data control.** You decide what is stored on the server. No sync service, no accidentally backed-up credentials, provided you take care during migration (see below).

`tmux` is purely an availability and convenience feature, not a security measure. The real work lies in securing the system.

## Starting point

The basis is Debian 13 (Trixie), installed minimally, without a desktop environment or additional network services. The provider supplies an upstream firewall that operates independently of the operating system. The aim is a server on which only SSH is reachable externally, and even that only with passphrase-protected keys.

## 1. Update the system

Immediately after installation, update all packages:

```bash
sudo apt update
sudo apt full-upgrade
```

Unlike `upgrade`, `full-upgrade` also resolves dependencies requiring packages to be installed or removed. On a fresh system, this is the right approach to install every available security update. Restart once after kernel updates.

## 2. Use a dedicated user instead of root

Working as root is unnecessarily risky: every typo affects the entire system, and direct root login is the first thing automated attacks attempt. Therefore, create a dedicated user (here `claude`) with sudo privileges for when they are needed:

```bash
sudo adduser claude
sudo usermod -aG sudo claude
```

From now on, all administration is performed through `claude` and `sudo`, no longer through direct root access.

## 3. Passphrase-protected Ed25519 keys, one per device

Login should be exclusively via SSH keys, not passwords. Ed25519 is the current standard: short, fast and cryptographically sound. Crucially, the key is generated on the client, i.e. on the PC rather than the server, and protected with a passphrase. The passphrase is the second line of defence should the private key ever fall into the wrong hands.

On the PC:

```bash
ssh-keygen -t ed25519 -C "pc-thinkpad"
```

The comment (`-C`) identifies the device. This pays off later: generate a separate key for every device, one for the PC and a separate one for the iPhone. If a device is lost, remove its public key specifically from `~/.ssh/authorized_keys` without having to redeploy all other access credentials.

Only the public key belongs on the server. The private key never leaves the device. In `authorized_keys`, there are ultimately only public keys, each with its device comment:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...pc  pc-thinkpad
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...ios iphone-15
```

Initially transfer the public PC key. As long as password login is still active, the easiest way is:

```bash
ssh-copy-id claude@SERVER
```

Then test that login with the key works before disabling password login in the next step. File permissions must be correct; otherwise sshd ignores the file: `~/.ssh` on `700`, `authorized_keys` on `600`.

## 4. Harden SSH: no root, no password

The server configuration is located in `/etc/ssh/sshd_config` and, on Debian 13, in drop-in files under `/etc/ssh/sshd_config.d/`. Changes belong in a dedicated drop-in file; this leaves the main file untouched and prevents package updates from overwriting anything. Create the file `/etc/ssh/sshd_config.d/99-haertung.conf`:

```text
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
```

This disables direct root login and password login. From now on, only someone with a matching private key can get in. Before reloading, check the configuration syntax:

```bash
sudo sshd -t
```

If `sshd -t` reports nothing, the file is valid. Only then reload:

```bash
sudo systemctl reload ssh
```

**Important:** Keep the existing SSH session open and test the new access in a second terminal. Only once key-based login has demonstrably worked there should you close the old session. This precaution reduces the risk of locking yourself out to practically zero. Otherwise, a configuration error costs you all access.

## 5. Move SSH to an uncommon port

The default port 22 is constantly probed by bots. Switching to a high, freely chosen port (`61417` in this example) causes most of this automated noise to come to nothing. This is explicitly not a security gain in the true sense: changing the port does not replace strong authentication; it merely reduces log volume and scan load. The key requirement from step 4 remains the real protection.

The port choice is not arbitrary. IANA distinguishes three ranges: **0–1023 (well-known ports)** are reserved for standard services (SSH itself on 22, HTTP on 80, HTTPS on 443), require root to bind and have no place on a self-selected SSH port; scanners expect these ports, as do standard services installed later. **1024–49151 (registered ports)** are assigned to individual applications on request, such as 3306 (MySQL), 5432 (PostgreSQL), 6379 (Redis) or 8080/8443 as common HTTP alternatives; a randomly chosen port from this range can easily clash later with software expecting its registered port. **49152–65535 (dynamic/private ports)** are not assigned to any service according to IANA and are intended for temporary, private purposes: the right range for a permanent, self-selected port.

One caveat remains: many Linux systems, including Debian, use part of the same range as source ports for their own outgoing connections (`net.ipv4.ip_local_port_range`, by default around 32768–60999). A permanently listening service does not genuinely clash with this, as the kernel does not assign a port that is already bound, but choosing a port above 60999 also avoids this theoretical ambiguity. The example in this article (`61417`) is deliberately in that range. Before changing it, also use `ss -lntup` (see step 7) to check that the selected port is not already in use on your server.

There is a pitfall on Debian 13: SSH can be started via systemd socket activation. If this is the case, the `Port` setting in `sshd_config` is simply ignored; the port must then be set on the socket. First check which case applies:

```bash
systemctl is-enabled ssh.socket
```

If the command responds with `enabled`, SSH is running via the socket. Change the port there:

```bash
sudo systemctl edit ssh.socket
```

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

If socket activation is not active (`disabled`), add `Port 61417` to the drop-in file from step 4 instead, followed by `sudo sshd -t` and `sudo systemctl restart ssh`.

The same applies here: first open the new port in the firewall (next step), then connect and test, and keep the old session open until access via the new port has been confirmed.

## 6. Firewall: closed by default

The upstream provider firewall is the most effective boundary because it intercepts packets before they even reach the operating system. Two basic rules apply:

- **Set the default incoming action to DROP.** Everything not explicitly allowed is discarded silently, without sending feedback to the sender.
- **One exception only:** incoming TCP to destination port `61417`. Nothing else needs to be reachable externally.

Outgoing traffic remains allowed. This is intentional: the server must download packages, synchronise time and reach the API for Claude Code. Restrictive outbound filtering provides little additional protection on a single server, while making operation noticeably more cumbersome.

If you want additional defence in depth, you can duplicate the same rules on the host with `nftables` or `ufw`. For the setup described, the provider firewall is sufficient.

## 7. Check the attack surface

After hardening, check what the server actually offers to the outside world. Two commands are enough. First: which services are listening on which addresses?

```bash
sudo ss -lntup
```

`ss` lists all listening TCP and UDP sockets together with their associated process (`sudo` is required to see process names). The address column is decisive: a service on `0.0.0.0` or `[::]` is externally reachable, while one on `127.0.0.1` or `[::1]` is local only. In the secured state, only SSH should appear publicly. Services such as `chronyd` (time synchronisation) may appear, but only bound to local addresses. If `chronyd` listens exclusively on `127.0.0.1` and `::1`, it cannot be reached externally and is therefore unproblematic.

Second: are there failed system services indicating a configuration problem?

```bash
systemctl --failed
```

The response should be `0 loaded units listed`: not a single failed service. Faulty units are not just an operational issue but potentially a security issue too, if they involve a partially started, incorrectly configured network service.

## 8. Install and run Claude Code

Claude Code requires a current Node.js runtime environment. After installing it, set up the CLI according to the official guide and authenticate afresh on the server rather than uploading local credentials (more on that shortly).

For persistent operation, use `tmux`:

```bash
tmux new -s claude
```

Start Claude within the session. With `Ctrl-b`, then `d`, you detach from the session without ending it; Claude continues running. Return with:

```bash
tmux attach -t claude
```

This allows a running task to survive disconnected connections, device changes and the laptop sleeping overnight.

## 9. Data hygiene during migration

The most sensitive part of moving to the server is not the technology but deciding what to bring along. Three rules:

- **No private keys on the server.** `authorized_keys` contains public keys only. Private keys remain on endpoint devices.
- **Do not copy credentials wholesale.** Sensitive local files such as `.credentials.json` do not belong on the VPS without review. Authenticate afresh on the server instead.
- **Move configuration to a migration folder first.** Do not write existing Claude memories and configuration directly to active configuration paths. Transfer them first to a separate migration folder and review what should actually be adopted. Anything no longer needed, such as old MCP entries or orphaned settings, is deliberately left behind rather than migrated uncritically.

## 10. Web previews via an SSH tunnel

For web previews, such as a local development server started by Claude, it is tempting simply to open another port. You should not do this. Every additional open port adds attack surface. Instead, run the preview through an encrypted SSH port tunnel: the service listens only locally on the server, and SSH forwards it to the client.

From the PC, make a service running locally on port 4321 accessible:

```bash
ssh -p 61417 -L 4321:localhost:4321 claude@SERVER
```

Then open `http://localhost:4321` in the local browser. Traffic runs entirely through the existing authenticated SSH connection, without opening even a single additional port in the firewall.

## Access from an iPhone

Access while away works with the same security model as from a PC. You only need an SSH client with key management. Common options include **Termius**, **Blink Shell** and **Secure ShellFish**; all can generate Ed25519 keys and store them in the iOS keychain, in some cases protected by Face ID.

The process corresponds to step 3, only on the iPhone:

1. Generate a dedicated Ed25519 key for the iPhone in the SSH client; do not copy the PC key. The private key remains in the device keychain.
2. Add the iPhone's public key as an additional line in `~/.ssh/authorized_keys` on the server, with a meaningful comment (`iphone-15`).
3. Create the connection in the client: server address, user `claude`, port `61417`, and the iPhone key for authentication.

This is precisely why a separate key per device is worthwhile: if the iPhone is lost, delete the one `iphone-15` line from `authorized_keys` on the server, and the device is locked out while PC access and all other keys continue working untouched.

After connecting, resume the running Claude session with `tmux attach -t claude` and continue where you left off at your desk. The port tunnel from step 10 also works from iOS; Termius and Secure ShellFish support port forwarding.

## Checklist

In summary, the complete process:

1. Debian 13 installed and fully updated with `apt full-upgrade`.
2. Dedicated user `claude` with sudo privileges; direct root login is no longer used.
3. Passphrase-protected Ed25519 keys, one per device, with public keys only in `authorized_keys`.
4. sshd hardened: `PermitRootLogin no`, `PasswordAuthentication no`; checked with `sshd -t` before reloading, keeping the existing session open until testing was complete.
5. SSH moved to port 61417, set on `ssh.socket` when socket activation is used, otherwise in the sshd configuration.
6. Provider firewall: incoming default DROP, sole exception TCP 61417; outgoing traffic allowed.
7. Attack surface checked with `ss -lntup` (only SSH public, `chronyd` local) and `systemctl --failed` (no errors).
8. Claude Code authenticated afresh on the server, running in a `tmux` session.
9. Data hygiene: no private keys or credentials on the server; configuration first reviewed through a migration folder.
10. No additional ports; web previews run through an SSH tunnel.

After this setup, only SSH on the specified port is reachable externally, and even there only with a passphrase-protected key. Claude Code runs independently of the endpoint device in `tmux`; web previews remain accessible through SSH tunnels without opening an additional port.

## Sources

1.  [OpenSSH Manual – sshd_config(5)](https://man.openbsd.org/sshd_config): Reference for all sshd directives, including `PermitRootLogin`, `PasswordAuthentication` and `PubkeyAuthentication`.

2.  [Debian Wiki – SSH](https://wiki.debian.org/SSH): Debian-specific notes on SSH configuration, including drop-in files under `/etc/ssh/sshd_config.d/`.

3.  [systemd.socket(5) – freedesktop.org](https://www.freedesktop.org/software/systemd/man/latest/systemd.socket.html): How socket activation and the `ListenStream=` directive work, relevant to changing the SSH port on Debian 13.

4.  [ss(8) – iproute2 Manpage](https://man7.org/linux/man-pages/man8/ss.8.html): Options for `ss` to list listening sockets together with their process and bound address.

5.  [Claude Code – Official documentation](https://docs.claude.com/en/docs/claude-code/overview): Installation, authentication and operation of Claude Code.
