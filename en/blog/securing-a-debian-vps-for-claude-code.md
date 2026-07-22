---
title: "Running Claude Code on Your Own VPS and Hardening the Server Properly"
navTitle: "VPS for Claude"
description: "Why your own VPS is a solid base for Claude Code, and the complete hardening checklist: a dedicated user, SSH with passphrase-protected Ed25519 keys only, a hardened sshd configuration, a restrictive firewall, attack-surface checks, and access from your iPhone."
date: "2026-07-21"
kategorie: "Linux"
timeToRead: "12 min to read"
themen:
  - "server-sicherheit"
translationOf: "claude-code-vps-debian-absichern"
slug: "securing-a-debian-vps-for-claude-code"
url: "https://rafaelpfister.ch/en/blog/securing-a-debian-vps-for-claude-code"
aiPrompt: |
  You are my Linux server assistant. Help me harden a freshly installed Debian 13 VPS step by step:
  1. Fully update the system and create a dedicated user with sudo rights.
  2. Switch SSH login to passphrase-protected Ed25519 keys, disable root and password login.
  3. Move SSH to a high port (mind systemd socket activation) and validate the config with `sshd -t` before I reload.
  4. Set the firewall so that only the SSH port is open inbound (default DROP), outbound allowed.
  5. Check with `ss -lntup` and `systemctl --failed` which services listen publicly and whether anything is failing.
  Ask me before any step that could lock me out, and keep my existing session open until the new access is tested.
---

Starting Claude Code locally on your workstation is convenient, until the laptop lid closes, the Wi-Fi changes, or a long-running task dies in the middle of the night. Your own VPS solves that: a session that keeps running, reachable from your PC and your iPhone, with full control over what data lives there. The price is that the server is publicly reachable from the first minute, and therefore a target for automated scans. Running it is still worthwhile, and a freshly installed Debian server can be hardened to withstand that constant background noise.

None of the steps are Claude-specific. It is the same hardening checklist that applies to any publicly reachable Linux server.

## Why your own VPS?

Three reasons favour a dedicated server over a local install:

- **Persistence.** Inside a `tmux` session, Claude keeps running even when the SSH connection drops. A task that takes ten minutes or an hour runs to completion without the laptop having to stay open.
- **Reachability.** The same session is accessible from your desktop, your laptop, and your iPhone. You kick off a task at your desk and check the result on the move.
- **Data control.** What lives on the server is up to you. No sync service, no accidentally backed-up credentials, provided you migrate carefully (see below).

`tmux` here is purely an availability and convenience feature, not a security measure. The real work is in the hardening.

## Starting point

The base is Debian 13 (Trixie), a minimal install with no desktop and no extra network services. The provider offers an upstream firewall that takes effect independently of the operating system. The goal is a server on which only SSH is reachable from outside, and even that only with passphrase-protected keys.

## 1. Update the system

Right after installation, bring the whole package set up to date:

```bash
sudo apt update
sudo apt full-upgrade
```

Unlike `upgrade`, `full-upgrade` also resolves dependencies that require installing or removing packages. On a fresh system, that is the right way to actually pull in every available security update. Reboot once after kernel updates.

## 2. A dedicated user instead of root

Working as root is needlessly risky: every typo acts system-wide, and direct root login is the first thing automated attacks try. Hence a dedicated user (here `claude`) with sudo rights for the cases where they are needed:

```bash
sudo adduser claude
sudo usermod -aG sudo claude
```

From now on, all administration runs through `claude` and `sudo`, no longer through direct root access.

## 3. Ed25519 keys with a passphrase, one per device

Login should run exclusively via SSH keys, never passwords. Ed25519 is the current standard: short, fast, cryptographically sound. What matters is that the key is generated on the client (on the PC, not the server) and protected with a passphrase. The passphrase is the second line of defence should the private key ever fall into the wrong hands.

On the PC:

```bash
ssh-keygen -t ed25519 -C "pc-thinkpad"
```

The comment (`-C`) names the device. That pays off later: generate a separate key for each device: one for the PC, a distinct one for the iPhone. If a device is lost, you remove exactly its public key from `~/.ssh/authorized_keys` without having to roll out every other login again.

Only the public key belongs on the server. The private key never leaves the device. In the end, `authorized_keys` contains public keys only, each with its device comment:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...pc  pc-thinkpad
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...ios iphone-15
```

Transfer the PC's public key initially. While password login is still active, the easiest way is:

```bash
ssh-copy-id claude@SERVER
```

Then verify that key login works before disabling password authentication in the next step. The file permissions have to be right, otherwise sshd ignores the file: `~/.ssh` set to `700`, `authorized_keys` to `600`.

## 4. Harden SSH: no root, no password

The server configuration lives in `/etc/ssh/sshd_config` and (on Debian 13) in drop-in files under `/etc/ssh/sshd_config.d/`. Changes belong in a dedicated drop-in file; that way the main file stays untouched and package updates overwrite nothing. Create the file `/etc/ssh/sshd_config.d/99-hardening.conf`:

```text
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
```

This disables direct root login and password authentication. From now on, only whoever holds a matching private key gets in. Before reloading, validate the configuration syntactically:

```bash
sudo sshd -t
```

If `sshd -t` reports nothing, the file is valid. Only then reload:

```bash
sudo systemctl reload ssh
```

**Important:** keep your existing SSH session open and test the new access in a second terminal. Only once key login demonstrably works there may the old session be closed. This precaution cuts the lockout risk to practically zero. A mistake in the configuration would otherwise cost you all access.

## 5. Move SSH to an unusual port

The default port 22 is probed by bots around the clock. Moving to a high, freely chosen port (`61417` in the example) lets most of that automated noise run into nothing. This is explicitly not a security gain in the strict sense: a port change does not replace strong authentication, it merely reduces log volume and scan load. The key requirement from step 4 remains the actual protection.

Which port to pick isn't arbitrary. IANA distinguishes three zones. **0–1023 (well-known ports)** are reserved for standard services (SSH itself on 22, HTTP on 80, HTTPS on 443), require root to bind, and have no business hosting a custom SSH port: those are exactly the ports scanners and future software both expect. **1024–49151 (registered ports)** are assigned to specific applications on request, for example 3306 (MySQL), 5432 (PostgreSQL), 6379 (Redis), or 8080/8443 as common HTTP alternatives; a randomly chosen port in this range can later collide with software that expects its registered port. **49152–65535 (dynamic/private ports)** are not assigned to any service by IANA and are meant for temporary, private use, the right zone for a permanent, custom port.

One caveat: many Linux systems, Debian included, also use part of that same range as the source port for their own outbound connections (`net.ipv4.ip_local_port_range`, commonly around 32768–60999). A permanently listening service doesn't actually collide with that (the kernel never hands out a port already bound to a listener), but sitting above 60999 sidesteps even that theoretical fuzziness. The example in this article (`61417`) sits there deliberately. Before switching, also check with `ss -lntup` (see step 7) that the chosen port isn't already taken on your own server.

On Debian 13 there is a pitfall here: SSH can be started via systemd socket activation. If it is, the `Port` directive in `sshd_config` is simply ignored; the port then has to be set on the socket. First check which case applies:

```bash
systemctl is-enabled ssh.socket
```

If the command answers `enabled`, SSH runs via the socket. Then change the port there:

```bash
sudo systemctl edit ssh.socket
```

Enter the following lines in the editor. The first, empty `ListenStream=` line clears the preset port 22, the second sets the new one:

```text
[Socket]
ListenStream=
ListenStream=61417
```

Then apply:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

If socket activation is not active (`disabled`), put `Port 61417` into the drop-in file from step 4 instead, followed by `sudo sshd -t` and `sudo systemctl restart ssh`.

The same rule applies here: open the new port in the firewall first (next step), then connect and test, and keep the old session open until access via the new port is confirmed.

## 6. Firewall: closed by default

The upstream provider firewall is the most effective boundary because it intercepts packets before they even reach the operating system. Two ground rules:

- **Inbound default action set to DROP.** Anything not explicitly allowed is discarded, silently, with no reply to the sender.
- **A single exception:** inbound TCP on destination port `61417`. Nothing else needs to be reachable from outside.

Outbound traffic stays allowed. That is deliberate: the server has to pull packages, synchronise time, and reach the API for Claude Code. Restrictive egress filtering adds little protection on a single server while making operations noticeably more cumbersome.

If you want additional defence in depth, you can mirror the same rules host-side with `nftables` or `ufw`. For the setup described, the provider firewall is enough.

## 7. Check the attack surface

After hardening, verify what the server actually exposes. Two commands suffice. First: which services listen on which addresses?

```bash
sudo ss -lntup
```

`ss` lists all listening TCP and UDP sockets along with the owning process (`sudo` is needed to see the process names). The address column is what counts: a service on `0.0.0.0` or `[::]` is reachable from outside, one on `127.0.0.1` or `[::1]` only locally. In the hardened state, only SSH should appear publicly. Services such as `chronyd` (time synchronisation) may show up, but only bound to local addresses. If `chronyd` listens exclusively on `127.0.0.1` and `::1`, it is not reachable from outside and therefore uncritical.

Second: are there failed system services pointing to a configuration problem?

```bash
systemctl --failed
```

The answer should be `0 loaded units listed`, not a single failed service. Faulty units are not just an operational issue but potentially a security one too, if behind them sits a half-started, misconfigured network service.

## 8. Install and run Claude Code

Claude Code needs a current Node.js runtime. After installing it, set up the CLI per the official documentation and authenticate freshly on the server, do not upload your local credentials (more on that shortly).

For persistent operation, `tmux`:

```bash
tmux new -s claude
```

Start Claude inside the session. With `Ctrl-b`, then `d`, you detach from the session without ending it; Claude keeps running. Return with:

```bash
tmux attach -t claude
```

This way a running task survives dropped connections, device switches, and the laptop's night off.

## 9. Data hygiene during migration

The trickiest part of moving to the server is not the technology but the question of what you bring along. Three rules:

- **No private keys on the server.** `authorized_keys` holds public keys only. Private keys stay on the end devices.
- **Do not copy credentials wholesale.** Sensitive local files such as a `.credentials.json` do not belong on the VPS unexamined. Authenticate freshly on the server instead.
- **Configuration goes to a migration folder first.** Do not write existing Claude memories and configuration straight into the active configuration paths; transfer them into a separate migration folder first and review there what should actually be adopted. Whatever you no longer need, such as old MCP entries or orphaned settings, is deliberately left behind instead of migrating unseen.

## 10. Web previews over an SSH tunnel

For web previews, say a local development server that Claude starts, the temptation is to simply open another port. Do not. Every additional open port is additional attack surface. Instead, the preview runs through an encrypted SSH port tunnel: the service listens only locally on the server, and SSH forwards it to the client.

From the PC, make a service running locally on port 4321 reachable:

```bash
ssh -p 61417 -L 4321:localhost:4321 claude@SERVER
```

Then open `http://localhost:4321` in the local browser. The traffic runs entirely through the existing, authenticated SSH connection, without opening even a single additional port in the firewall.

## Access from the iPhone

Access on the go works with the same security model as from the PC. You just need an SSH client with key management. Common ones are **Termius**, **Blink Shell**, and **Secure ShellFish**; all can generate Ed25519 keys and store them in the iOS keychain, some protected by Face ID.

The procedure mirrors step 3, only on the iPhone:

1. In the SSH client, generate a dedicated Ed25519 key for the iPhone rather than copying the PC key. The private key stays in the device keychain.
2. Add the iPhone's public key as an additional line in `~/.ssh/authorized_keys` on the server, with a descriptive comment (`iphone-15`).
3. Create the connection in the client: server address, user `claude`, port `61417`, the iPhone key for authentication.

This is exactly why the separate key per device pays off: if the iPhone is lost, you delete the single `iphone-15` line from `authorized_keys` on the server, and the device is locked out, while PC access and every other key carry on untouched.

After connecting, pull the running Claude session back with `tmux attach -t claude` and continue where you left off at your desk. The port tunnel from step 10 works from iOS too; Termius and Secure ShellFish both support port forwarding.

## Checklist

The full sequence at a glance:

1. Debian 13 installed, fully updated with `apt full-upgrade`.
2. Dedicated user `claude` with sudo rights; direct root login no longer used.
3. Passphrase-protected Ed25519 keys, one per device, public keys only in `authorized_keys`.
4. sshd hardened: `PermitRootLogin no`, `PasswordAuthentication no`; validated with `sshd -t` before reload, existing session kept open until tested.
5. SSH on port 61417, set on `ssh.socket` under socket activation, otherwise in the sshd configuration.
6. Provider firewall: inbound default DROP, sole exception TCP 61417; outbound allowed.
7. Attack surface checked with `ss -lntup` (only SSH public, `chronyd` local) and `systemctl --failed` (no errors).
8. Claude Code freshly authenticated on the server, run inside a `tmux` session.
9. Data hygiene: no private keys and no credentials on the server, configuration vetted through a migration folder first.
10. No additional ports; web previews run through an SSH tunnel.

The effort is one-off, the result lasting: a server that shows the outside world only a single, key-protected door, and a Claude session that is always reachable, whether from the desk or on the move.

## Sources

1.  [OpenSSH Manual – sshd_config(5)](https://man.openbsd.org/sshd_config) — Reference for all sshd directives, including `PermitRootLogin`, `PasswordAuthentication`, and `PubkeyAuthentication`.

2.  [Debian Wiki – SSH](https://wiki.debian.org/SSH) — Debian-specific notes on SSH configuration, including the drop-in files under `/etc/ssh/sshd_config.d/`.

3.  [systemd.socket(5) – freedesktop.org](https://www.freedesktop.org/software/systemd/man/latest/systemd.socket.html) — How socket activation and the `ListenStream=` directive work, relevant to the SSH port change on Debian 13.

4.  [ss(8) – iproute2 manpage](https://man7.org/linux/man-pages/man8/ss.8.html) — Options of `ss` for listing listening sockets together with process and bind address.

5.  [Claude Code – Official Documentation](https://docs.claude.com/en/docs/claude-code/overview) — Installing, authenticating, and running Claude Code.
