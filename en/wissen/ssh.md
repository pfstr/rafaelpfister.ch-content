---
title: "SSH: Remote Access, Keys, and Hardening"
blatt: "ssh"
description: "Secure Shell in day-to-day administration: keys instead of passwords, agent and known_hosts, the essential hardening steps for exposed servers, and SFTP as a file transport."
fakten:
  - { label: "Full name", wert: "Secure Shell" }
  - { label: "Purpose", wert: "Encrypted remote access, commands, tunnels, and file transfer" }
  - { label: "Port", wert: "22 (TCP)" }
  - { label: "Standard", wert: "RFC 4251 through 4254", href: "https://datatracker.ietf.org/doc/html/rfc4251" }
  - { label: "Authentication", wert: "Public key (recommended), password (disable)" }
  - { label: "File transfer", wert: "SFTP and SCP over the same connection" }
  - { label: "Tools", wert: "ssh, ssh-keygen, ssh-agent, sshd_config" }
werbung: ["newsletter"]
ctaThemen: ["claude"]
---

# SSH: Remote Access, Keys, and Hardening

Every server, every appliance with a CLI, and every VPS is administered over SSH. The protocol is so ubiquitous that its configuration often stays untouched, and that is exactly where the problems begin: an exposed SSH port with password login is a standing invitation for automated attacks.

## Keys instead of passwords

SSH authenticates either with a password or with a key pair. For operations the rule is: **public key authentication is the standard**, and password login is disabled (`PasswordAuthentication no`). The private key stays on the workstation and is protected with a passphrase; `ssh-agent` keeps it unlocked in memory so that the passphrase is not required for every connection. Modern keys are created with `ssh-keygen -t ed25519`; RSA keys below 3072 bits should be replaced.

SSH raises the second question of trust itself: on the first connection, the server's host key is stored in `known_hosts`. If it changes later, SSH warns loudly. That warning is not a nuisance but the core of the protocol: it distinguishes a server migration from a man-in-the-middle.

## Hardening in five steps

For servers on the internet, a compact set of measures has proven itself: **password login off**, **root login off** (`PermitRootLogin no`, administration through a regular account plus `sudo`), **limit access** (`AllowUsers` or groups, and source IPs in the firewall where possible), **throttle failed attempts** (fail2ban or an equivalent mechanism), and **stay current** (apply OpenSSH updates promptly, because SSH is the door into the system). Changing the port replaces none of these measures, but it does reduce the background noise of scans in the logs.

## More than a shell

The same connection also carries files (**SFTP**, which rclone and many backup tools use as a backend) and tunnels (**port forwarding**), for example to reach a management interface that should not be publicly exposed. `ssh -L 8443:appliance.internal:443 jumphost` brings the GUI of an internal appliance safely to the local machine without a firewall exception.

## Quick check

```bash
ssh -o PasswordAuthentication=no admin@server.example.ch   # test enforced key-based login
sudo sshd -T | grep -E "passwordauthentication|permitrootlogin"
```

