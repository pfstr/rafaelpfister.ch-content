---
title: "Running Claude Code Securely on Your Own VPS"
navTitle: "VPS for Claude"
description: "A hardened Debian VPS keeps Claude Code sessions persistently available. This guide covers everything from user accounts and SSH keys to firewalls, data hygiene, tmux, and secure access from an iPhone."
date: "2026-07-21"
kategorie: "Claude"
timeToRead: "12 min read"
themen:
  - claude
translationOf: "claude-code-vps-debian-absichern"
slug: "securing-a-debian-vps-for-claude-code"
translationId: article-f932e9e537d7704a
translatedAt: 2026-09-04T08:42:04.965Z
translationReview: automatic
translationSourceHash: 011d5e16cec877d14e68e11ff48caee9b6ee849ee6235c889676cfe64ae81628
url: https://rafaelpfister.ch/en/blog/securing-a-debian-vps-for-claude-code
translationModel: gpt-5.6-terra
---

On your own computer, a Claude Code session ends unexpectedly at the latest when the laptop goes to sleep or the network connection drops. A VPS keeps running and can be reached from multiple devices. At the same time, it is permanently connected to the public internet and is automatically scanned shortly after startup.

This guide combines both requirements: Claude Code remains available in a `tmux` session, while the Debian server offers only a key-protected SSH connection to the outside world. The hardening is not specific to Claude and is also suitable for other publicly accessible Linux servers.

## Why a VPS can make sense

Compared with a purely local installation, the server offers three practical advantages:

- **Persistence.** In a `tmux` session, Claude keeps running even if the SSH connection is disconnected. A task that takes ten minutes or an hour finishes without the laptop needing to remain open.
- **Accessibility.** The same session can be accessed from a desktop, laptop, and iPhone. You start a task at your desk and check the result while on the go.
- **Data control.** You decide what is stored on the server. No sync service, no credentials accidentally backed up along with it—provided you migrate carefully (see below).

`tmux` is purely an availability and convenience feature, not a security measure. The actual work lies in securing the system.

## Starting point

The foundation is Debian 13 (Trixie), installed minimally, without a desktop environment or additional network services. The provider supplies an upstream firewall that operates independently of the operating system. The goal is a server on which only SSH is reachable from the outside—and even that only with passphrase-protected keys.

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
| `full-upgrade` | Updates all packages and may install new packages or remove existing ones if necessary |

</details>

Unlike `upgrade`, `full-upgrade` also resolves dependencies that require new or removed packages. On a fresh system, this is the right approach to install all available security updates. Restart once after kernel updates.

## 2. Use a dedicated user instead of root

Working as root is unnecessarily risky: every typo affects the entire system, and direct root login is the first thing automated attacks try. Therefore, create a dedicated user (here, `claude`) with sudo privileges for the cases where they are needed:

```bash
sudo adduser claude
sudo usermod -aG sudo claude
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-a` | Append: adds to the user's group list rather than replacing it; valid only together with `-G` |
| `-G sudo` | Supplementary group(s) to which the user is added |
| `claude` | The affected user; with `adduser` the name of the account to be created |

</details>

From now on, all administration is done through `claude` and `sudo`, no longer through direct root access.

## 3. Passphrase-protected Ed25519 keys, one per device

Logins should use SSH keys exclusively, not passwords. Ed25519 is the current standard: short, fast, and cryptographically sound. Crucially, the key is generated on the client—that is, on the PC, not the server—and protected with a passphrase. The passphrase is the second line of defense if the private key ever falls into the wrong hands.

On the PC:

```bash
ssh-keygen -t ed25519 -C "pc-thinkpad"
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-t ed25519` | Key type, in this case the Ed25519 elliptic-curve algorithm |
| `-C "pc-thinkpad"` | Comment appended to the public key |

</details>

The comment (`-C`) identifies the device. This pays off later: generate a separate key for every device—one for the PC and another for the iPhone. If a device is lost, remove only its public key from `~/.ssh/authorized_keys` without having to redeploy all other access keys.

Only the public key belongs on the server. The private key never leaves the device. In `authorized_keys`, there are ultimately only public keys, each with its device comment:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...pc  pc-thinkpad
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...ios iphone-15
```

Transfer the public PC key initially. As long as password login is still active, the easiest way is:

```bash
ssh-copy-id claude@SERVER
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `claude@SERVER` | User and target host; the default public key is appended there to `~/.ssh/authorized_keys` |

</details>

Then test that key-based login works before disabling password login in the next step. File permissions must be correct, otherwise sshd ignores the file: `~/.ssh` must be set to `700`, and `authorized_keys` to `600`.

## 4. Harden SSH: no root, no password

The server configuration is located in `/etc/ssh/sshd_config` and, on Debian 13, in drop-in files under `/etc/ssh/sshd_config.d/`. Changes belong in a dedicated drop-in file; this leaves the main file untouched and prevents package updates from overwriting anything. Create the file `/etc/ssh/sshd_config.d/99-haertung.conf`:

```text
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
```

This disables direct root login and password authentication. From now on, only someone with a matching private key can get in. Before reloading, check the configuration syntax:

```bash
sudo sshd -t
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-t` | Test mode: checks the configuration file and keys for validity without starting the service |

</details>

If `sshd -t` reports nothing, the file is valid. Only then reload:

```bash
sudo systemctl reload ssh
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `reload` | Tells the service to reload its configuration without disconnecting existing connections |
| `ssh` | The target unit, here the OpenSSH service |

</details>

**Important:** Keep the existing SSH session open and test the new access in a second terminal. Only close the old session once key-based login is demonstrably working there. This precaution reduces the risk of locking yourself out to virtually zero. A configuration error would otherwise cost you all access.

## 5. Move SSH to an uncommon port

Bots probe the standard port 22 around the clock. Switching to a high, freely chosen port (`61417` in this example) causes most of this automated noise to lead nowhere. This is explicitly not a security gain in the strict sense: changing ports does not replace strong authentication; it only reduces log volume and scanning load. The key requirement from step 4 remains the real protection.

The port choice is not arbitrary. IANA distinguishes three ranges: **0–1023 (well-known ports)** are reserved for standard services (SSH itself on 22, HTTP on 80, HTTPS on 443), require root privileges to bind, and have no place as a custom SSH port; scanners expect these ports, as do standard services installed later. **1024–49151 (registered ports)** are assigned to individual applications upon request, such as 3306 (MySQL), 5432 (PostgreSQL), 6379 (Redis), or 8080/8443 as common HTTP alternatives; a randomly chosen port in this range can easily conflict later with software expecting its registered port. **49152–65535 (dynamic/private ports)** are not assigned to any service according to IANA and are intended for temporary, private purposes—the right range for a permanent custom port.

One caveat remains: many Linux systems, including Debian, use part of the same range as source ports for their own outbound connections (`net.ipv4.ip_local_port_range`, by default around 32768–60999). A permanently listening service does not truly conflict with this, since the kernel does not assign an already bound port, but a port above 60999 also avoids this theoretical ambiguity. The example in this article (`61417`) is deliberately in that range. Before switching, also use `ss -lntup` (see step 7) to verify that the selected port is not already in use on your server.

There is a special consideration on Debian 13: SSH can be started through systemd socket activation. If that is the case, the `Port` setting in `sshd_config` is simply ignored; the port must instead be set on the socket. First check which case applies:

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

If the command returns `enabled`, SSH runs through the socket. Change the port there:

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

The same applies here: first open the new port in the firewall (next step), then connect and test it, keeping the old session open until access over the new port has been confirmed.

## 6. Firewall: closed by default

The upstream provider firewall is the most effective boundary because it intercepts packets before they even reach the operating system. Two basic rules apply:

- **Set the default action for incoming traffic to DROP.** Anything not explicitly allowed is discarded silently, without sending feedback to the sender.
- **One exception only:** incoming TCP traffic on target port `61417`. Nothing else needs to be reachable from the outside.

Outbound traffic remains allowed. This is intentional: the server must download packages, synchronize the time, and reach the API for Claude Code. Restricting outbound traffic offers little additional protection for a single server, but makes operation noticeably more cumbersome.

For additional defense in depth, you can duplicate the same rules on the host using `nftables` or `ufw`. For the setup described here, the provider firewall is sufficient.

## 7. Check the attack surface

After hardening, verify what the server is actually exposing externally. Two commands are enough. First: Which services are listening on which addresses?

```bash
sudo ss -lntup
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-l` | Show only listening sockets |
| `-n` | Numeric output: ports and addresses are not resolved to names |
| `-t` | Include TCP sockets |
| `-u` | Include UDP sockets |
| `-p` | Shows the process behind each socket; this requires `sudo` |

</details>

The address column is crucial: a service on `0.0.0.0` or `[::]` is reachable from the outside, while one on `127.0.0.1` or `[::1]` is local only. In the secured state, only SSH should appear publicly. Services such as `chronyd` (time synchronization) may appear, but only bound to local addresses. If `chronyd` listens exclusively on `127.0.0.1` and `::1`, it cannot be reached from the outside and is therefore not an issue.

Second: Are there failed system services that indicate a configuration problem?

```bash
systemctl --failed
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `--failed` | Lists only units in a failed state |

</details>

The response should be `0 loaded units listed`, with not a single failed service. Failed units are not only an operational issue, but potentially also a security issue if they involve a partially started, misconfigured network service.

## 8. Install and run Claude Code

Claude Code requires a current Node.js runtime environment. After installing it, set up the CLI according to the official documentation and authenticate anew on the server; do not upload local credentials (more on that in a moment).

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

Start Claude within the session. Pressing `Ctrl-b`, then `d` detaches from the session without ending it; Claude keeps running. Return with:

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

This allows a running task to survive disconnected connections, switching devices, and the laptop sleeping overnight.

## 9. Data hygiene during migration

The most sensitive part of moving to a server is not the technology, but deciding what to bring along. Three rules apply:

- **No private keys on the server.** Only public keys are stored in `authorized_keys`. Private keys remain on endpoint devices.
- **Do not copy credentials indiscriminately.** Sensitive local files such as a `.credentials.json` do not belong on the VPS without review. Instead, authenticate anew on the server.
- **Move configuration to a migration folder first.** Do not write existing Claude memories and configuration directly into active configuration paths. Instead, transfer them first to a separate migration folder and review what should actually be adopted. Anything no longer needed, such as old MCP entries or orphaned settings, is deliberately left behind rather than carried over without review.

## 10. Web previews through an SSH tunnel

For web previews, such as a local development server started by Claude, it is tempting to simply open another port. Do not do that. Every additional open port increases the attack surface. Instead, run the preview through an encrypted SSH port tunnel: the service listens only locally on the server, and SSH forwards it to the client.

From the PC, make a service running locally on port 4321 accessible:

```bash
ssh -p 61417 -L 4321:localhost:4321 claude@SERVER
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-p 61417` | Port on which the SSH server listens (the one selected in step 5) |
| `-L 4321:localhost:4321` | Local port forwarding: connections to local port 4321 are forwarded through the tunnel to `localhost:4321` from the server's perspective |
| `claude@SERVER` | User and target host of the SSH connection |

</details>

Then open `http://localhost:4321` in the local browser. All traffic runs through the existing, authenticated SSH connection, without opening even a single additional port in the firewall.

## Access from an iPhone

Access while on the go follows the same security model as from a PC. You only need an SSH client with key management. Popular choices include **Termius**, **Blink Shell**, and **Secure ShellFish**; all can generate Ed25519 keys and store them in the iOS keychain, in some cases protected with Face ID.

The procedure is the same as in step 3, only on the iPhone:

1. Generate a dedicated Ed25519 key for the iPhone in the SSH client; do not copy the PC key. The private key remains in the device's keychain.
2. Add the iPhone's public key as an additional line in `~/.ssh/authorized_keys` on the server, with a descriptive comment (`iphone-15`).
3. Create the connection in the client: server address, user `claude`, port `61417`, and the iPhone key for authentication.

This is exactly why a separate key per device is worthwhile: if the iPhone is lost, delete the single `iphone-15` line from `authorized_keys` on the server, and the device is locked out while PC access and all other keys continue to work unaffected.

After connecting, resume the running Claude session with `tmux attach -t claude` and continue where you left off at your desk. The port tunnel from step 10 also works from iOS; Termius and Secure ShellFish support port forwarding.

## Checklist

Here is the complete process in summary:

1. Installed Debian 13 and fully updated it with `apt full-upgrade`.
2. Created dedicated user `claude` with sudo privileges; direct root login is no longer used.
3. Used passphrase-protected Ed25519 keys, one per device; only public keys in `authorized_keys`.
4. Hardened sshd: `PermitRootLogin no`, `PasswordAuthentication no`; checked with `sshd -t` before reloading and kept the existing session open until testing was complete.
5. Moved SSH to port 61417, configured on `ssh.socket` when using socket activation, otherwise in the sshd configuration.
6. Provider firewall: incoming default DROP, with TCP 61417 as the only exception; outbound traffic allowed.
7. Checked the attack surface with `ss -lntup` (only SSH public, `chronyd` local) and `systemctl --failed` (no errors).
8. Authenticated Claude Code anew on the server and ran it in a `tmux` session.
9. Maintained data hygiene: no private keys or credentials on the server; reviewed configuration through a migration folder first.
10. No additional ports; web previews run through an SSH tunnel.

After this setup, only SSH on the specified port is reachable from the outside, and only with a passphrase-protected key. Claude Code runs independently of the endpoint device in `tmux`; web previews remain accessible through SSH tunnels without opening an additional port.

## Sources

1.  [OpenSSH Manual – sshd_config(5)](https://man.openbsd.org/sshd_config): Reference for all sshd directives, including `PermitRootLogin`, `PasswordAuthentication` and `PubkeyAuthentication`.

2.  [Debian Wiki – SSH](https://wiki.debian.org/SSH): Debian-specific notes on SSH configuration, including the drop-in files under `/etc/ssh/sshd_config.d/`.

3.  [systemd.socket(5) – freedesktop.org](https://www.freedesktop.org/software/systemd/man/latest/systemd.socket.html): How socket activation and the `ListenStream=` directive work, relevant to changing the SSH port on Debian 13.

4.  [ss(8) – iproute2 Manpage](https://man7.org/linux/man-pages/man8/ss.8.html): Options for `ss` to list listening sockets along with their process and bind address.

5.  [Claude Code – Official Documentation](https://docs.claude.com/en/docs/claude-code/overview): Installing, authenticating, and running Claude Code.
