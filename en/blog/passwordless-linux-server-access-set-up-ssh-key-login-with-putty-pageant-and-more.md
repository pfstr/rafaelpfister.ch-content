---
title: "Passwordless Linux Server Access: Set Up SSH Key Login with PuTTY, Pageant, and More"
navTitle: "Passwordless PuTTY"
description: "Admins who access Linux servers daily have to enter a username and password every time when using password login. An SSH key pair reduces this to a double-click: generate a key with PuTTYgen, store the public key on the server, and load Pageant. The same key works in WinSCP, MobaXterm, and OpenSSH, and if desired, takes you straight to a service account shell."
date: "2026-08-28"
kategorie: "Linux and SSH"
timeToRead: "9 min read"
themen:
  - totemomail
  - windows-client
produkte:
  - "totemomail"
  - "uebergreifend"
protokolle:
  - "ssh"
  - "haertung"
slug: "passwordless-linux-server-access-set-up-ssh-key-login-with-putty-pageant-and-more"
translationId: "article-9f94fa6eb8b95bcf"
aiPrompt: |
  Du bist mein Linux- und SSH-Assistent. Hilf mir Schritt für Schritt, einen passwortlosen SSH-Login von Windows auf meine Linux-Server einzurichten: 1. Ein Ed25519-Schlüsselpaar mit PuTTYgen erzeugen und den Public Key in authorized_keys eintragen. 2. Die PuTTY-Session mit Schlüsseldatei und Auto-login username vervollständigen und Pageant mit Autostart einrichten. 3. Optional ein Remote command hinterlegen, das mich direkt in die Shell eines Service-Accounts bringt, samt minimaler NOPASSWD-Regel unter /etc/sudoers.d. Weise mich auf typische Fehler hin: Key im falschen Home-Verzeichnis, mehrzeiliger Public Key, falsche Berechtigungen, sudoers-Befehl stimmt nicht exakt mit dem Remote command überein.
translationOf: putty-ssh-login-service-account-shell
url: https://rafaelpfister.ch/en/blog/passwordless-linux-server-access-set-up-ssh-key-login-with-putty-pageant-and-more
translationSourceHash: e95b27e9a86f59dfb0808afee63664493f5961b983f807f31ef9ee7a36f6fb3e
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:35:30.046Z
translationReview: automatic
---

# Passwordless Linux Server Access: Set Up SSH Key Login with PuTTY, Pageant, and More

Admins who access Linux servers daily repeat the same entries with every password login: username, password, and, for service accounts, a sudo command afterward. An SSH key pair eliminates all of that. Once set up, double-clicking the saved session opens a ready-to-use shell, and the same key works in PuTTY, WinSCP, MobaXterm, and the Windows OpenSSH client.

This guide fully sets up passwordless login from Windows: generate a key pair, store the public key on the server, complete the PuTTY session, and configure Pageant as a key agent. It also covers troubleshooting the most common issue (the server continues to ask for the password despite the key) and, as an extension, logging directly into the shell of a service account such as `totemo`.

## Why use keys instead of passwords

The convenience gain is the most visible effect, but not the most important one. An Ed25519 key is practically immune to brute-force attacks, while a password is only as strong as its length and the discipline not to reuse it anywhere. On servers whose users have been fully migrated to keys, password authentication can be disabled entirely in the sshd configuration (`PasswordAuthentication no`), causing automated login attempts from the internet to go nowhere. Only disable password authentication once key login has been proven to work and a second access path (console, second key) is available.

The principle: the private key remains on your Windows computer, while the public key is stored on the server. When a connection is established, the server checks whether the other side possesses the matching private key, without that key ever leaving the computer.

## Step 1: Generate a key pair with PuTTYgen

1. Start **PuTTYgen** (part of the PuTTY package), select **Ed25519** as the type, click **Generate**, and move the mouse over the area.
2. Enter a **passphrase** in both fields and save the private key with **Save private key** as a `.ppk` file.
3. Copy the text field at the top in full (the line beginning with `ssh-ed25519 AAAA...`). This is the public key in the format expected by the server.

Save the private key with a passphrase. Without a passphrase, every copy of the file is a ready-made server login; with a passphrase, the file alone is worthless. Pageant (Step 4) almost completely eliminates the convenience drawback. A key without a passphrase is only acceptable for unattended automation, not for interactive login.

## Step 2: Store the public key on the server

On the server, logged in as the user you will connect as in the future:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo 'ssh-ed25519 AAAA... kommentar' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod go-w ~
```

<details class="options-details">
<summary>Options explained</summary>

| Command / Option | Effect |
|---|---|
| `mkdir -p ~/.ssh` | Creates the SSH directory; `-p` suppresses the error if it already exists |
| `chmod 700 ~/.ssh` | Only the owner may read, write, and enter the directory |
| `echo '...' >> ~/.ssh/authorized_keys` | Appends the public key as a new line to the file (`>>` rather than `>`, otherwise existing keys are overwritten) |
| `chmod 600 ~/.ssh/authorized_keys` | Only the owner may read and write the key file |
| `chmod go-w ~` | Revokes write permission on the home directory from the group and others |

</details>

The last line may look inconspicuous, but it is essential: a group-writable or world-writable home directory causes the SSH daemon to silently ignore the key, and the server continues asking for the password without giving a reason.

## Step 3: Complete the PuTTY session

1. Open PuTTY, select the saved session, and load it using **Load**.
2. Go to **Connection → SSH → Auth → Credentials** and select the `.ppk` file under **Private key file for authentication**.
3. Go to **Connection → Data** and enter the username under **Auto-login username**; otherwise, PuTTY will still ask for it every time you connect.
4. Return to the **Session** category, select the session name again, and click **Save**.

The most common user error here is skipping **Load** before making changes or **Save** afterward. Without Load, you are editing only the default settings; without Save, the change is lost the next time PuTTY starts.

## Step 4: Pageant, one passphrase per Windows session

Pageant is PuTTY's key agent. It keeps the decrypted private key in memory, so the passphrase is required only once per Windows session:

1. Start Pageant (an icon appears in the system tray).
2. Right-click the icon, choose **Add Key**, select the `.ppk` file, and enter the passphrase.
3. From then on, all connections proceed without prompting until the next restart.

To have Pageant start automatically, create a shortcut in the Startup folder (`Win+R`, then `shell:startup`) and pass the key as an argument:

```text
"C:\Program Files\PuTTY\pageant.exe" "C:\Pfad\zum\schluessel.ppk"
```

Windows will then ask for the passphrase once after sign-in, and the rest of the workday proceeds without it.

## If the server continues to ask for the password

Start troubleshooting in PuTTY's **Event Log** (right-click the title bar of the terminal session). It shows whether the key was offered at all:

| Event Log finding | Cause and solution |
|---|---|
| No entry about a public key | The `.ppk` is not configured in the saved session, or the wrong session was edited. Load the session, set the key, and save it. |
| `Server refused our key` | The server cannot find or accept the key: wrong home directory, wrong format, or wrong permissions (see below). |
| `Access granted`, followed by a password prompt | Key login worked; the prompt comes from a downstream program, typically sudo. See the extension below. |

The three most common causes of `Server refused our key`:

- **Key in the wrong home directory.** The public key must be in the `authorized_keys` of the user the connection is established as. If you already switched to another account with `sudo -u` or `su` before adding it, the file is placed in that account's home directory rather than your own. Running `whoami` before adding it shows whose home directory will receive the key.
- **Wrong format.** The public key must be a single line in `authorized_keys`, using the format from the text field at the top of PuTTYgen. The file produced by **Save public key** uses a different, multi-line format (`---- BEGIN SSH2 PUBLIC KEY ----`) and does not work in `authorized_keys`.
- **Permissions.** `~/.ssh` set to `700`, `authorized_keys` set to `600`, and the home directory must not be group-writable or world-writable.

If the finding remains unclear, check `/var/log/secure` or `journalctl -u sshd` on the server; the SSH daemon explains the rejection there.

## The same key in other tools

The server-side setup is tool-independent, and the key can be reused everywhere:

| Tool | Setup |
|---|---|
| **WinSCP** | Uses `.ppk` files directly (Login → Advanced → SSH → Authentication) and automatically uses Pageant when the key is loaded there |
| **MobaXterm** | Set `.ppk` under Session settings → SSH → Advanced → Use private key; it also understands the OpenSSH format |
| **FileZilla** | Configure `.ppk` under Settings → SFTP, or keep Pageant running |
| **OpenSSH (Windows Terminal, `ssh`)** | Requires the OpenSSH format: export it in PuTTYgen through **Conversions → Export OpenSSH key** and store it in `~/.ssh/` |

For convenient login with the OpenSSH client, add an entry to `~/.ssh/config` on the Windows computer:

```text
Host mailgw
    HostName server.example.com
    User mmuster
    IdentityFile ~/.ssh/id_ed25519
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Host mailgw` | Freely selectable alias; `ssh mailgw` is then sufficient to connect |
| `HostName` | Actual server name or IP address |
| `User` | Username, equivalent to Auto-login username in PuTTY |
| `IdentityFile` | Path to the private key in OpenSSH format |

</details>

## Extension: go straight to the service account shell

Many Linux servers in application and mail environments are not administered with a personal account but through a service account: TotemoMail runs under `totemo`, while other gateways and applications have their own functional accounts. After login, the same command therefore routinely follows:

```bash
sudo -u totemo /bin/bash -l
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `sudo` | Runs the following command with different privileges and logs the invocation |
| `-u totemo` | The target user is `totemo` rather than the default `root` |
| `/bin/bash` | The command to execute: a new Bash shell |
| `-l` | Starts Bash as a login shell; it loads the target user's profile (`.bash_profile`, environment variables, paths) |

</details>

The `-l` is crucial for service accounts: without a login shell, the environment variables from the functional account's profile are missing, such as paths to application directories or Java installations, and application-specific commands fail with misleading error messages.

Direct SSH login as the service account would be even shorter, but for good reason is usually not possible at all: functional accounts often have no password or a locked login shell, and direct access shared by several people would eliminate the personalized audit trail. Going through sudo makes it possible to trace which person switched to the service account shell and when. The following automation does not change that; it merely saves typing.

### Remote command in the PuTTY session

PuTTY can store a command per saved session that is run after login instead of the normal shell:

1. Load the session with **Load**.
2. Navigate in the tree on the left to **Connection → SSH** (the main node, not a subitem).
3. Enter the following in the **Remote command** field: `sudo -u totemo /bin/bash -l`
4. Save under **Session**.

There are three characteristics of the remote command you should know:

- An `exit` in the service account shell closes the connection completely instead of returning to your personal shell. The command replaces the login shell; it does not nest it.
- If you occasionally work on the server with your personal account, save a second session without a remote command (load the session, clear the field, and save it under a new name).
- File transfer tools such as WinSCP or `pscp` are not affected. They establish their own connections and ignore the remote command of the PuTTY session.

If the connection closes again immediately after being established or sudo reports a missing terminal: under **Connection → SSH → TTY**, check that **Don't allocate a pseudo-terminal** is not selected. By default, it is not. Important for Step 2 above: with a remote command enabled, you are already the service account when adding the public key; however, the key belongs in the personal user's home directory.

In the OpenSSH client, two lines in the `Host` entry do the same thing: `RequestTTY yes` and `RemoteCommand sudo -u totemo /bin/bash -l`.

### sudo without a password prompt

After the key and remote command, one entry remains: sudo's password prompt. It disappears only through the sudoers configuration on the server, which requires root privileges. On a managed company server, this is a request to the server administrator, not a PuTTY setting.

The rule belongs in a separate file under `/etc/sudoers.d/` and is edited with `visudo`:

```bash
visudo -f /etc/sudoers.d/totemo-shell
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `visudo` | Opens the sudoers file in an editor and checks the syntax before saving |
| `-f /etc/sudoers.d/totemo-shell` | Edits the specified file instead of the central `/etc/sudoers` |

</details>

Contents of the file, using the personal username (here `mmuster` as an example):

```text
mmuster ALL=(totemo) NOPASSWD: /bin/bash -l
```

<details class="options-details">
<summary>Options explained</summary>

| Component | Effect |
|---|---|
| `mmuster` | The rule applies only to this user |
| `ALL=` | On all hosts (relevant for centrally distributed sudoers files) |
| `(totemo)` | Only for commands as target user `totemo`, not as root |
| `NOPASSWD:` | No password prompt for the following commands |
| `/bin/bash -l` | Exactly this command with exactly this argument, nothing else |

</details>

Two points determine whether the rule applies and whether it is reasonable:

- **Exact match.** The command in the rule must match the remote command, including its argument. If PuTTY contains `sudo -u totemo /bin/bash -l`, the rule must allow `/bin/bash -l`. A rule for `/bin/bash` without `-l` does not cover an invocation with `-l`, and sudo will continue to ask for the password.
- **Minimal scope.** The rule permits one command as one target user. It grants neither root privileges nor arbitrary commands. In this form, it is also a common and justifiable request on managed servers. sudo logging remains fully intact; every switch to the service account shell is still recorded in the log.

`visudo` is not optional: it checks syntax before saving. A typo written directly into the file can render sudo unusable for all users of the system. For the same reason, a separate file under `/etc/sudoers.d/` is preferable to editing the central `/etc/sudoers`; it survives package updates and can be removed safely.

## The result

Once configured, login looks like this: double-click the PuTTY session, and the shell is ready. No username, no password, and with the extension, no sudo prompt either. The security posture has not worsened; in one respect, it has even improved:

| Aspect | Before | After |
|---|---|---|
| Authentication | Password at every login | Ed25519 key with passphrase, held in Pageant |
| Login identity | Personal account | Personal account unchanged |
| sudo logging | Every switch recorded in the log | Every switch still recorded in the log |
| NOPASSWD scope | None | One command, one target user, no root |

## Sources

1.  [PuTTY User Manual, Chapter 4: Configuring PuTTY](https://the.earth.li/~sgtatham/putty/0.83/htmldoc/Chapter4.html): Documentation of session settings, including Remote command (SSH panel), Auto-login username (Data panel), and Pseudo-Terminal (TTY panel).

2.  [PuTTY User Manual, Chapter 8: Using public keys for SSH authentication](https://the.earth.li/~sgtatham/putty/0.83/htmldoc/Chapter8.html): PuTTYgen, key types, passphrases, OpenSSH export, and adding the public key to the server.

3.  [PuTTY User Manual, Chapter 9: Using Pageant for authentication](https://the.earth.li/~sgtatham/putty/0.83/htmldoc/Chapter9.html): How the agent works, loading keys, and starting with a key as a command-line argument.

4.  [ssh_config(5) Manual](https://man.openbsd.org/ssh_config.5): OpenSSH client configuration, including host aliases, IdentityFile, RequestTTY, and RemoteCommand.

5.  [sudoers(5) Manual](https://www.sudo.ws/docs/man/sudoers.man/): Syntax of sudoers rules, Runas specification, and the NOPASSWD tag.

6.  [sshd(8) Manual, AUTHORIZED_KEYS FILE FORMAT section](https://man.openbsd.org/sshd.8): Format of the authorized_keys file and file permission requirements.
