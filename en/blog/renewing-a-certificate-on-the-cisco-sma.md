---
title: "Renewing a Certificate on the Cisco SMA"
navTitle: "SMA Certificate"
description: "Certificates can only be installed on the Cisco SMA through the CLI, and current AsyncOS versions validate the entire chain during import: without a stored root CA, the import fails. This article shows the paths to a new key pair, the OpenSSL approach in detail, how to handle OpenSSL 3's RC2-40-CBC error, and how to import the internal root CA into the appliance's trust store."
date: "2026-08-04"
kategorie: "Cisco ESA / SMA"
timeToRead: "11 min read"
themen:
  - cisco-esa-sma
  - smtp-mailflow
hauptthema: "cisco-esa-sma"
slug: "renewing-a-certificate-on-the-cisco-sma"
translationId: "article-69d93a1e5e081848"
aiPrompt: |
  Du bist mein Assistent für die Zertifikatserneuerung auf einer Cisco SMA (Secure Email and Web Manager). Führe mich Schritt für Schritt durch den Ablauf aus diesem Artikel: 1. Wahl des Wegs zum Schlüsselpaar (OpenSSL-CSR in der eigenen Umgebung, PFX von der CA oder Umweg über eine ESA), 2. CN- und SAN-Liste für meine Hostnamen, 3. je nach Weg CSR-Erzeugung mit OpenSSL oder Konvertierung der PFX-Datei nach PEM inklusive Umgang mit dem Fehler RC2-40-CBC, 4. bei interner CA Import der Root-CA in die Custom-Liste der Appliance, 5. Installation über certconfig in der CLI, 6. Kontrolle. Frage mich zuerst nach den Hostnamen meiner Appliances und der Quarantäneseite, ob die ausstellende CA intern oder öffentlich ist und welche OpenSSL-Version ich installiert habe. Passe alle Befehle an meine Dateinamen an und erinnere mich vor dem Abschluss daran, die certconfig-Session nicht mit Ctrl+C zu beenden und die Änderung mit commit zu aktivieren.
translationOf: cisco-sma-zertifikat-erneuern
translationSourceHash: c99ce64a5e63875b84c7b6f14a7f2fb7e51290fedbdc93d99201cdc97a743508
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T09:05:59.415Z
translationReview: automatic
url: https://rafaelpfister.ch/en/blog/renewing-a-certificate-on-the-cisco-sma
---

# Renewing a Certificate on the Cisco SMA

The Cisco SMA (Security Management Appliance, now marketed as Cisco Secure Email and Web Manager) provides centralized spam quarantine and reporting for Secure Email Gateways in many mail environments. Its HTTPS certificate covers the admin GUI and the quarantine page, where end users review and release their held messages. When it expires, mail flow does not break. The expiration is immediately visible nonetheless: every visit to the quarantine page ends with a certificate warning in the browser, and the very users whose awareness training teaches them not to click through such warnings are then expected to ignore them.

During a renewal in a customer project, there were two issues right away: first, OpenSSL 3 responded to the internal CA's PFX file with a cryptic error related to `RC2-40-CBC`, then the appliance refused to import the completed certificate because the issuing root CA was unknown to it. Both hurdles and their solutions are covered below.

## What the SMA does differently from the ESA

On the ESA, the complete certificate lifecycle can be handled through the GUI (`Network > Certificates`). The SMA cannot do that: the server certificate is installed exclusively through the CLI, using the `certconfig` command in an SSH session. The SMA GUI only displays certificates; only the lists of trusted certificate authorities can be maintained there, as discussed later.

There are also two other peculiarities:

- The paste dialog accepts only PEM format. A PFX file (PKCS#12) must be converted before installation; current AsyncOS versions also offer direct PKCS#12 import, but the file must first be transferred to the appliance.
- Older AsyncOS versions (as documented in the Cisco technical note) neither generate keys nor CSRs themselves, so the key pair must be created externally; the three viable approaches are described below. Current versions can use `certconfig > CERTIFICATE > NEW` to generate a self-signed certificate and CSR directly on the appliance. However, this does not help with a shared certificate across multiple appliances, because the private key never leaves the appliance in that case.

A single certificate can either serve all services (inbound and outbound TLS, HTTPS administrative access, LDAPS) or be configured separately for each service. This is controlled in the `certconfig` dialog; the command header always shows the active assignment (`Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.`). There is no separate assignment screen as on the ESA, and it cannot be changed through the GUI. In most environments, one certificate for everything is the pragmatic choice: the name list covers the appliances' FQDNs anyway, and separate key pairs multiply the effort for every renewal.

That the dialog on a quarantine appliance asks about inbound and outbound TLS may seem confusing at first, since the SMA is not in any MX path. It still speaks SMTP in both directions. Inbound (Receiving) is the receiving side: the ESAs deliver quarantined messages to the SMA by SMTP, to the centralized spam quarantine on port 6025 and to the centralized policy, virus, and outbreak quarantines on port 7025; the latter connections are TLS-encrypted by default, and the SMA presents this exact certificate. Outbound (Delivery) is the sending side: when a user releases a message from quarantine, the SMA itself delivers it back into mail flow via its SMTP routes, and the appliance also sends quarantine notifications, scheduled reports, and alerts as its own emails. For the renewal, this means HTTPS is the practical concern; both SMTP services simply come along when using one certificate for all services.

## Defining names: CN and SAN

Regardless of how the key pair is created, first define the name list. The Common Name belongs to the hostname users use to access the quarantine page. The SAN list should additionally include the appliances' FQDNs so direct access to the admin GUI also works without warnings. For an environment with two appliances, the name list looks like this:

| Field | Value |
|---|---|
| CN | `spam-quarantine.example.ch` |
| SAN | `spam-quarantine.example.ch` |
| SAN | `sma01.example.ch` |
| SAN | `sma02.example.ch` |

Two notes: browsers have long evaluated only SAN entries; the CN alone is not enough. The quarantine hostname must therefore also be included as a SAN. And short hostnames without a domain component (such as `SMA01`) are issued only by an internal CA; public CAs do not sign internal names.

## Three paths to a new key pair

For a certificate that covers multiple appliances and the quarantine hostname, the key pair must be created outside the appliance. Three approaches have become established:

1. Generate the key and CSR with OpenSSL within your own environment. The private key is created where it is needed and never leaves the environment. This is the recommended approach; details are in the next section.
2. The CA generates the key pair and provides a PFX file. This works, but has two drawbacks: the key passes through external hands (the password should therefore be sent through a separate channel, not in the same email as the file), and depending on the CA tool, you may receive an RC2-encrypted PFX that OpenSSL 3 can open only with extra effort; more on that below.
3. The detour through an ESA, documented in the Cisco technical note: under `Network > Certificates`, create a certificate with the SMA's CN, download the CSR and have the CA sign it, upload the signed certificate back to the ESA, and export everything as a PFX. Here too, the final step is conversion to PEM.

## The most important openssl options

For orientation, here are the subcommands and options of `openssl` used in this article, translated in meaning from the OpenSSL documentation:

<details class="options-details">
<summary>Option overview</summary>

| Option | Meaning |
|---|---|
| `req` | Subcommand for certificate requests (CSRs): generate, display, verify |
| `-new` | Generates a new request |
| `-newkey rsa:2048` | Also generates a new 2048-bit RSA key pair |
| `-noenc` | Writes the private key unencrypted (up to OpenSSL 3.0: `-nodes`) |
| `-keyout datei` | Target file for the private key |
| `-out datei` | Target file for output, here CSR or PEM |
| `-subj text` | Request subject in `/C=…/O=…/CN=…` format |
| `-addext text` | Adds an extension to the request, here the SAN list |
| `pkcs12` | Subcommand for PKCS#12 containers (PFX): create and unpack |
| `-in datei` | Input file |
| `-legacy` | Also loads the legacy provider for legacy algorithms such as RC2 |
| `list` | Subcommand to display the installation's capabilities |
| `-providers` | Lists loaded providers |
| `-provider name` | Also loads the specified provider for this invocation |
| `s_client` | Subcommand: TLS test client for connections to a server |
| `-connect host:port` | Target host and port of the TLS connection |
| `-servername name` | Sets Server Name Indication (SNI) in the TLS handshake |
| `x509` | Subcommand for displaying and processing certificates |
| `-noout` | Suppresses output of the encoded certificate |
| `-subject` | Outputs the certificate subject |
| `-enddate` | Outputs the expiration date (notAfter) |

</details>

The OpenSSL documentation provides the complete references as a separate man page for each subcommand: `openssl-req(1)`, `openssl-pkcs12(1)`, `openssl-s_client(1)` and `openssl-x509(1)`.

## Starting OpenSSL on Windows

All subsequent steps use OpenSSL on a system within the environment, such as an admin server. The Light Edition of the Windows builds from Shining Light Productions is sufficient; the installer is about 6 MB and can be verified against the checksum list published by slproweb.

The installer places everything under `C:\Program Files\OpenSSL-Win64`, and the executable is located in `bin\openssl.exe`. It does not add itself to the search path: anyone typing `openssl` in a fresh command prompt receives an error message. There are three options:

- Open the `Win64 OpenSSL Command Prompt` entry from the Start menu. It starts `start.bat` from the installation directory, sets the environment, and greets you with the output of `openssl version -a`. In this window, `openssl` works directly.
- Specify the full path: `"C:\Program Files\OpenSSL-Win64\bin\openssl.exe" version`.
- Permanently add `C:\Program Files\OpenSSL-Win64\bin` to the `Path` environment variable; `openssl` will then be available in every shell.

Those already using Git for Windows need no additional installation: it includes its own OpenSSL (`C:\Program Files\Git\mingw64\bin\openssl.exe`), and it is immediately in the search path in Git Bash. Current Git versions ship with OpenSSL 3.5 and an active Legacy provider, so `-legacy` from the PFX conversion section also works there. You can verify it as follows:

```bash
openssl list -providers -provider legacy
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `list` | Displays the capabilities of the OpenSSL installation |
| `-providers` | Lists loaded providers with name, version, and status |
| `-provider legacy` | Also loads the `legacy` provider for this invocation; if it appears in the list, it is available |

</details>

Git Bash has one peculiarity, however: it treats arguments beginning with `/` as paths and rewrites them. `-subj "/C=CH/O=Example AG/CN=..."` becomes `C:/Program Files/Git/C=CH/O=Example AG/CN=...`, and OpenSSL aborts:

```text
req: subject name is expected to be in the format /type0=value0/type1=value1/type2=...
where characters may be escaped by \. This name is not in that format:
'C:/Program Files/Git/C=CH/O=Example AG/CN=spam-quarantine.example.ch'
```

A preceding `MSYS_NO_PATHCONV=1` disables rewriting for the individual invocation. The issue does not occur in Command Prompt, PowerShell, or the OpenSSL Command Prompt.

## Generating a key and CSR with OpenSSL

A single invocation generates the key and CSR with the complete SAN list:

```bash
openssl req -new -newkey rsa:2048 -noenc \
  -keyout spam-quarantine.example.ch.key \
  -out spam-quarantine.example.ch.csr \
  -subj "/C=CH/O=Example AG/CN=spam-quarantine.example.ch" \
  -addext "subjectAltName=DNS:spam-quarantine.example.ch,DNS:sma01.example.ch,DNS:sma02.example.ch"
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-new` | Generates a new certificate request (CSR) |
| `-newkey rsa:2048` | Also generates a new 2048-bit RSA key pair |
| `-noenc` | Writes the private key unencrypted to the file |
| `-keyout …` | Target file for the private key |
| `-out …` | Target file for the CSR |
| `-subj …` | Subject with country, organization, and Common Name |
| `-addext …` | Appends the SAN extension containing all DNS names to the request |

</details>

The CSR file goes to the CA, while the key remains on the server. The signed certificate and intermediate are returned, usually directly as PEM. Everything needed for installation is then ready, and PFX conversion is completely unnecessary with this approach.

The key file is unencrypted (`-noenc`), because `certconfig` expects it that way. Until installation, keep it on the server with restrictive permissions; then delete it or move it to password management.

## Converting PFX to PEM

This section and the next apply to approaches 2 and 3, which result in a PFX file. `certconfig` expects the certificate and private key as PEM, with the key unencrypted. A single OpenSSL invocation handles both:

```bash
openssl pkcs12 -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `pkcs12` | Subcommand for creating and unpacking PKCS#12 containers |
| `-in …` | The PFX input file |
| `-out …` | The PEM output file containing certificate, key, and chain certificates |
| `-noenc` | Writes the private key without a passphrase (up to OpenSSL 3.0, the option was called `-nodes`) |

</details>

The import password prompt has no echo, and no asterisks are displayed either. The resulting PEM file contains the certificate, key, and supplied chain certificates in one file and must be protected accordingly: delete it after installation or move it to password management.

## When OpenSSL 3 rejects the PFX file

With older PFX files, conversion under OpenSSL 3.x aborts with this message:

```text
Error outputting keys and certificates
error:0308010C:digital envelope routines:inner_evp_generic_fetch:unsupported:
crypto/evp/evp_fetch.c: Global default library context, Algorithm (RC2-40-CBC : 0)
```

The cause is not a corrupted file but a design decision: OpenSSL 3 moved legacy algorithms such as RC2, RC4, and DES into a separate Legacy provider, which is not loaded by default. Many PFX exports from older Windows systems and CA tools encrypt the certificate portion of the container specifically with RC2-40-CBC. OpenSSL 1.1 opened such files without issue; OpenSSL 3 rejects them.

The solution is one additional option:

```bash
openssl pkcs12 -legacy -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-legacy` | Loads the Legacy provider for this invocation; legacy algorithms such as RC2-40-CBC become available again and conversion completes |

</details>

The prerequisite is an OpenSSL installation that includes the Legacy provider; this is the case for common Windows builds.

To eliminate the error permanently, address it at the source and export the PFX file using modern encryption: current export dialogs and CA tools offer AES-256, which eliminates the Legacy workaround entirely.

As a graphical alternative, XCA (X Certificate and Key Management) works: import the PFX file through `Importieren > PKCS#12`, then export the certificate as PEM from the `Zertifikate` tab and the key separately as an unencrypted PEM from the `Private Schlüssel` tab. Both exports are needed; `certconfig` prompts for the certificate and key separately. XCA includes its own cryptographic library and can also open containers with legacy algorithms.

One more note on sources: the OpenSSL project does not publish Windows binaries itself, but refers to third-party builds such as Win64 OpenSSL from Shining Light Productions. Download portals with their own installers are the wrong place for a cryptographic tool.

## Import the internal root CA into the appliance trust store first

Current AsyncOS versions validate the complete chain when creating a certificate profile. If the certificate originates from an internal CA whose root is unknown to the appliance, import aborts with this message:

```text
Cannot import certificate: The certificate authority validation could not be
trusted for certificate because unable to get local issuer certificate
```

The appliance maintains two lists of trusted certificate authorities: the supplied system list and a Custom list for your own CAs. The internal root CA belongs in the Custom list, before the server certificate is installed. Only the public CA certificate as a PEM file is required (`-----BEGIN CERTIFICATE-----` through `-----END CERTIFICATE-----`), not a private key.

This is how to upload the root CA to the appliance through the web interface:

1. Open `Network > Certificates`.
2. In the `Certificate Authorities` section, click `Edit Settings`.
3. Under `Custom List`, select `Enable`.
4. Upload the PEM file through `Choose File`.
5. Run `Submit` and then `Commit Changes`.
6. Under `Network > Certificates > Manage Trusted Root Certificates`, verify that the CA appears in the list of custom certificates.

If a Custom list already exists, export it first and append the new CA to the existing PEM bundle: importing replaces the list, otherwise previously configured CAs disappear. For a chain with an intermediate, import the root CA first and then the intermediate CA. During import, AsyncOS checks the expiration date, duplicates, and the `CA:TRUE` flag, among other things, and rejects an intermediate as long as its associated root is missing. The same import can also be performed through the CLI: `certconfig > CERTAUTHORITY > CUSTOM > IMPORT`, followed by `commit`.

Two distinctions: for updates through a TLS-inspecting proxy, the SMA maintains a separate trust store (`updateconfig > TRUSTED_CERTIFICATES > ADD`), and the Custom CA list does not apply there. And placing the root CA on the SMA does not eliminate browser warnings: clients still need the root through their own certificate distribution, typically through GPO, and the appliance must deliver the server certificate including its intermediate.

## Installation with certconfig

Log in to the SMA via SSH and start `certconfig`. On current AsyncOS versions, the dialog works with certificate profiles:

```text
sma01.example.ch> certconfig

Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.

Choose the operation you want to perform:
- CERTIFICATE - Import, Create a request, Edit or Remove Certificate Profiles
- CERTAUTHORITY - Manage System and Customized Authorities
- CRL - Manage Certificate Revocation Lists
[]> certificate
```

Behind `CERTIFICATE` are the operations `IMPORT` (PKCS#12 file previously uploaded to the appliance), `PASTE` (paste the certificate into the CLI), `NEW` (generate a self-signed certificate and CSR), `EDIT`, `EXPORT`, `DELETE`, and `PRINT` (shows service assignment). The usual path through SSH is `PASTE`: the dialog prompts for a profile name, then for the certificate, private key, and optionally the CA intermediate certificate, each as a PEM block terminated by a single `.` on its own line. The final question about FQDN validation of the Common Name can be answered with the default value. The intermediate must be included in the profile; otherwise, clients lack the chain and, depending on the browser, the warning remains despite a valid certificate.

Older AsyncOS versions (as documented in the Cisco technical note) instead display a `SETUP` dialog. It starts with the question `Do you want to use one certificate/key for receiving, delivery, HTTPS management access, and LDAPS?`: `y` assigns the same pair to all four services, while `n` goes through the certificate, key, and intermediate prompts once for each service. The paste principle is identical.

Two points determine success or failure: do not end the session with Ctrl+C, as that immediately discards all changes. And run `commit` at the end; only then is the certificate active. With two appliances, repeat the process on both, as certificate configuration is not synchronized between SMAs.

## Verification

The fastest test runs externally against the quarantine page. By default, end-user access to the spam quarantine uses HTTPS port 83, unless something else was configured when it was enabled:

```bash
openssl s_client -connect spam-quarantine.example.ch:83 \
  -servername spam-quarantine.example.ch </dev/null 2>/dev/null |
  openssl x509 -noout -subject -enddate
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `s_client` | TLS test client: establishes the connection and passes through the presented certificate |
| `-connect …:83` | Target host and port, here the HTTPS port of the spam quarantine |
| `-servername …` | Sets Server Name Indication (SNI) so the server delivers the matching certificate |
| `x509` | Processes the passed-through certificate |
| `-noout` | Suppresses output of the encoded certificate |
| `-subject` | Outputs the certificate subject |
| `-enddate` | Outputs the expiration date (notAfter) |

</details>

The output must show the new subject and the new expiration date. On the appliance, `certconfig` with the `PRINT` operation lists the active certificates, and a browser check against the admin GUI and quarantine page confirms that the chain is built correctly.

## Sources

1.  [How to Generate and Install a Certificate on an SMA](https://www.cisco.com/c/en/us/support/docs/security/content-security-management-appliance/118460-technote-sma-00.html): Cisco technical note covering the certconfig workflow of older AsyncOS versions, the PEM requirement, and ways to generate certificates through ESA or OpenSSL.

2.  [Common Administrative Tasks, AsyncOS 16.0 for Cisco Secure Email and Web Manager](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_01011.html): Admin Guide chapter covering management of Certificate Authority lists (system and Custom lists), including checks during CA import.

3.  [Comprehensive Spam Quarantine Setup Guide on ESA and SMA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118692-configure-esa-00.html): Cisco guide to spam quarantine, including end-user access through HTTPS on port 83.

4.  [openssl-req](https://docs.openssl.org/master/man1/openssl-req/): Reference for generating keys and CSRs, including `-addext` for the SAN list.

5.  [openssl-pkcs12](https://docs.openssl.org/master/man1/openssl-pkcs12/): Reference for conversion options, including `-noenc` (formerly `-nodes`) and `-legacy`.

6.  [OpenSSL 3.0 Migration Guide](https://docs.openssl.org/3.0/man7/migration_guide/): Background on moving legacy algorithms to the Legacy provider.

7.  [XCA: X Certificate and Key Management](https://hohnstaedt.de/xca/): Open-source tool for importing and exporting PKCS#12 and PEM structures.

8.  [Win32/Win64 OpenSSL](https://slproweb.com/products/Win32OpenSSL.html): Windows builds from Shining Light Productions, referenced by the OpenSSL project, including its published checksum list.

9.  [MSYS2: Filesystem Paths](https://www.msys2.org/docs/filesystem-paths/): Description of automatic path rewriting, which changes the `-subj` argument in Git Bash, along with `MSYS_NO_PATHCONV`.

10.  [openssl-s_client](https://docs.openssl.org/master/man1/openssl-s_client/): Reference for the TLS test client, including `-connect` and `-servername`.

11.  [openssl-x509](https://docs.openssl.org/master/man1/openssl-x509/): Reference for display options, including `-noout`, `-subject`, and `-enddate`.
