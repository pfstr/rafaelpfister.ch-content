---
title: "Renewing a Certificate on the Cisco SMA"
navTitle: "SMA Certificate"
description: "Certificates can only be installed on the Cisco SMA through the CLI, and current AsyncOS versions validate the entire chain during import: without a stored root CA, it fails. This article covers the options for obtaining a new key pair, the OpenSSL approach in detail, handling OpenSSL 3's RC2-40-CBC error, and importing the internal root CA into the appliance's trust store."
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
url: https://rafaelpfister.ch/en/blog/renewing-a-certificate-on-the-cisco-sma
translationSourceHash: 0c12510db6a327680d08d3f4eb6924738cef4987860e42c41043ce66467d4249
translationModel: gpt-5.6-terra
translatedAt: 2026-08-05T06:05:02.169Z
translationReview: automatic
---

# Renewing a Certificate on the Cisco SMA

The Cisco SMA (Security Management Appliance, now known as Cisco Secure Email and Web Manager) provides centralized spam quarantine and reporting for Secure Email Gateways in many mail environments. Its HTTPS certificate covers the admin GUI and the quarantine page where end users review and release their held messages. When it expires, mail flow does not stop. However, the expiration is immediately visible: every visit to the quarantine page results in a certificate warning in the browser, and the very users whom awareness training teaches not to click through such warnings are then expected to ignore them.

During a renewal in a customer project, there were two stumbling blocks: first, OpenSSL 3 responded to the internal CA's PFX file with a cryptic error concerning `RC2-40-CBC`, then the appliance refused to import the finished certificate because it did not know the issuing root CA. Both hurdles and their solutions are covered below.

## What the SMA does differently from the ESA

On the ESA, the entire certificate lifecycle can be handled through the GUI (`Network > Certificates`). The SMA cannot do this: the server certificate is installed exclusively through the CLI, using the `certconfig` command in an SSH session. The SMA GUI only displays certificates; only the lists of trusted certificate authorities can be managed there, more on that later.

There are also two other particularities:

- The paste dialog accepts only PEM format. A PFX file (PKCS#12) must be converted before installation; current AsyncOS versions also offer direct PKCS#12 import, but the file must first be transferred to the appliance.
- Older AsyncOS versions (as documented in the Cisco technical note) neither generate keys nor CSRs themselves, so the key pair must be created externally; the three viable options are described below. Current versions can use `certconfig > CERTIFICATE > NEW` to generate a self-signed certificate along with a CSR directly on the appliance. However, this does not help with a shared certificate across multiple appliances because the private key never leaves the appliance.

A single certificate can either serve all services (inbound and outbound TLS, HTTPS management access, LDAPS) or be assigned separately for each service. This is controlled in the `certconfig` dialog; the command header always shows the active assignment (`Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.`). There is no separate assignment screen as on the ESA, and it cannot be changed through the GUI. In most environments, one certificate for everything is the pragmatic choice: the name list covers the appliances' FQDNs anyway, and separate key pairs multiply the effort with every renewal.

That the dialog on a quarantine appliance asks about inbound and outbound TLS is initially confusing, since the SMA is not in an MX path. However, it still uses SMTP in both directions. Inbound (Receiving) is the receiving side: the ESAs deliver quarantined messages by SMTP to the SMA, to the centralized spam quarantine on port 6025 and to the centralized policy, virus, and outbreak quarantines on port 7025; the latter connections are TLS-encrypted by default, and the SMA presents this exact certificate. Outbound (Delivery) is the sending side: when a user releases a message from quarantine, the SMA itself delivers it back into mail flow through its SMTP routes, and the appliance also sends quarantine notifications, scheduled reports, and alerts as its own emails. For renewal, HTTPS is what matters in practice; the two SMTP services simply come along when using a certificate for all services.

## Defining names: CN and SAN

Regardless of how the key pair is obtained, start by defining the name list. The Common Name belongs to the hostname under which users access the quarantine page. The SAN list should additionally include the appliances' FQDNs so that direct access to the admin GUI also works without warnings. For an environment with two appliances, the name list looks like this:

| Field | Value |
|---|---|
| CN | `spam-quarantine.example.ch` |
| SAN | `spam-quarantine.example.ch` |
| SAN | `sma01.example.ch` |
| SAN | `sma02.example.ch` |

Two notes on this: browsers have long evaluated only SAN entries; the CN alone is not sufficient. The quarantine hostname must therefore also appear as a SAN. And short hostnames without a domain component (such as `SMA01`) are issued only by an internal CA; public CAs do not sign internal names.

## Three ways to obtain a new key pair

For a certificate that covers multiple appliances and the quarantine hostname, the key pair must be created outside the appliance. Three approaches have become established:

1. Generate the key and CSR with OpenSSL within your own environment. The private key is created where it is needed and never leaves the environment. This is the recommended approach; details follow in the next section.
2. The CA generates the key pair and provides a PFX file. This works, but has two drawbacks: the key passes through other hands (the password should therefore be sent through a separate channel, not in the same email as the file), and depending on the CA tool, an RC2-encrypted PFX may be returned, which OpenSSL 3 opens only with additional effort; more on that below.
3. The detour through an ESA, documented in the Cisco technical note: create a certificate there under `Network > Certificates` with the SMA's CN, download the CSR and have it signed by the CA, upload the signed certificate back to the ESA, and export the whole thing as a PFX. Here, too, PEM conversion is required at the end.

## Starting OpenSSL on Windows

All following steps use OpenSSL on a system within the environment, such as an admin server. The Light edition of the Windows builds from Shining Light Productions is sufficient; the installer is about 6 MB and can be verified against the checksum list published by slproweb.

The installer places everything under `C:\Program Files\OpenSSL-Win64` and the executable is located in `bin\openssl.exe`. It does not add itself to the search path: anyone typing `openssl` in a fresh command prompt gets an error message. Three options lead to the goal:

- Launch the `Win64 OpenSSL Command Prompt` entry from the Start menu. It starts `start.bat` from the installation directory, sets up the environment, and greets you with the output of `openssl version -a`. In this window, `openssl` works directly.
- Specify the full path: `"C:\Program Files\OpenSSL-Win64\bin\openssl.exe" version`.
- Permanently add `C:\Program Files\OpenSSL-Win64\bin` to the `Path` environment variable; afterward, `openssl` is available in every shell.

Those already using Git for Windows do not need any additional installation: it includes its own OpenSSL (`C:\Program Files\Git\mingw64\bin\openssl.exe`), and it is immediately in the search path in Git Bash. Current Git versions ship with OpenSSL 3.5 and an active legacy provider, so `-legacy` from the PFX conversion section also works there. You can verify it as follows:

```bash
openssl list -providers -provider legacy
```

However, Git Bash has one pitfall: it treats arguments beginning with `/` as paths and rewrites them. `-subj "/C=CH/O=Example AG/CN=..."` becomes `C:/Program Files/Git/C=CH/O=Example AG/CN=...`, and OpenSSL aborts:

```text
req: subject name is expected to be in the format /type0=value0/type1=value1/type2=...
where characters may be escaped by \. This name is not in that format:
'C:/Program Files/Git/C=CH/O=Example AG/CN=spam-quarantine.example.ch'
```

A preceding `MSYS_NO_PATHCONV=1` disables rewriting for that individual invocation. The problem does not occur in Command Prompt, PowerShell, or the OpenSSL command prompt.

## Generating the key and CSR with OpenSSL

A single command generates the key and CSR with the complete SAN list:

```bash
openssl req -new -newkey rsa:2048 -noenc \
  -keyout spam-quarantine.example.ch.key \
  -out spam-quarantine.example.ch.csr \
  -subj "/C=CH/O=Example AG/CN=spam-quarantine.example.ch" \
  -addext "subjectAltName=DNS:spam-quarantine.example.ch,DNS:sma01.example.ch,DNS:sma02.example.ch"
```

Send the CSR file to the CA; the key remains on the server. The CA returns the signed certificate along with the intermediate, usually directly as PEM. This leaves everything ready for installation, and PFX conversion is completely unnecessary with this approach.

The key file is unencrypted (`-noenc`), because `certconfig` expects it exactly that way. Until installation, keep it on the server under restrictive permissions; afterward, delete it or move it to password management.

## Converting PFX to PEM

This and the next section apply to approaches 2 and 3, which result in a PFX file. `certconfig` expects the certificate and private key as PEM, with the key unencrypted. A single OpenSSL command handles both:

```bash
openssl pkcs12 -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

`-noenc` (up to OpenSSL 3.0, the option was called `-nodes`) writes the private key without a passphrase to the output file. The import password prompt has no echo, and no asterisks appear. The resulting PEM file contains the certificate, key, and supplied chain certificates in one file and must be protected accordingly: delete it after installation or move it to password management.

## When OpenSSL 3 refuses the PFX file

With older PFX files, conversion under OpenSSL 3.x aborts with this message:

```text
Error outputting keys and certificates
error:0308010C:digital envelope routines:inner_evp_generic_fetch:unsupported:
crypto/evp/evp_fetch.c: Global default library context, Algorithm (RC2-40-CBC : 0)
```

The cause is not a damaged file but a design decision: OpenSSL 3 moved legacy algorithms such as RC2, RC4, and DES into a separate legacy provider that is not loaded by default. However, many PFX exports from older Windows systems and CA tools encrypt the certificate part of the container using RC2-40-CBC. OpenSSL 1.1 opened such files without issue; OpenSSL 3 rejects them.

The solution is one additional option:

```bash
openssl pkcs12 -legacy -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

`-legacy` loads the legacy provider for this invocation, after which conversion completes successfully. This requires an OpenSSL installation that includes the legacy provider; this is the case with common Windows builds.

To eliminate the error permanently, address it at the source and have the PFX file exported with modern encryption: current export dialogs and CA tools offer AES-256, which eliminates the legacy workaround entirely.

XCA (X Certificate and Key Management) works as a graphical alternative: import the PFX file through `Importieren > PKCS#12`, then export the certificate as PEM on the `Zertifikate` tab and the key separately as unencrypted PEM on the `Private Schlüssel` tab. Both exports are required; `certconfig` asks for the certificate and key separately. XCA includes its own cryptographic library and also opens containers with legacy algorithms.

One more word on the download source: the OpenSSL project does not publish Windows binaries itself, but refers to third-party builds such as Win64 OpenSSL from Shining Light Productions. Download portals with their own installers are the wrong source for a cryptographic tool.

## Import the internal root CA into the appliance trust store first

Current AsyncOS versions validate the complete chain when creating a certificate profile. If the certificate comes from an internal CA whose root is unknown to the appliance, import aborts with this message:

```text
Cannot import certificate: The certificate authority validation could not be
trusted for certificate because unable to get local issuer certificate
```

The appliance maintains two lists of trusted certificate authorities: the supplied system list and a custom list for your own CAs. The internal root CA belongs in the custom list, before the server certificate is installed. Only the public CA certificate as a PEM file is required (`-----BEGIN CERTIFICATE-----` through `-----END CERTIFICATE-----`), not a private key.

This is how to get the root CA onto the appliance through the web interface:

1. Open `Network > Certificates`.
2. In the `Certificate Authorities` section, click `Edit Settings`.
3. Under `Custom List`, select the `Enable` option.
4. Upload the PEM file through `Choose File`.
5. Perform `Submit` and then `Commit Changes`.
6. Under `Network > Certificates > Manage Trusted Root Certificates`, verify that the CA appears in the list of custom certificates.

If a custom list already exists, export it first and append the new CA to the existing PEM bundle: the import replaces the list, otherwise previously stored CAs will disappear. For a chain with an intermediate, import the root CA first, then the intermediate CA. During import, AsyncOS checks, among other things, the expiration date, duplicates, and the `CA:TRUE` flag, and rejects an intermediate as long as the associated root is missing. The same import can also be done through the CLI: `certconfig > CERTAUTHORITY > CUSTOM > IMPORT`, then `commit`.

Two distinctions are worth noting: for updates through a TLS-inspecting proxy, the SMA uses a separate trust store (`updateconfig > TRUSTED_CERTIFICATES > ADD`), to which the custom CA list does not apply. And the root CA on the SMA does not eliminate browser warnings: clients still need the root through their own certificate distribution, typically via GPO, and the appliance must deliver the server certificate along with the intermediate.

## Installation with certconfig

Log in to the SMA through SSH and start `certconfig`. On current AsyncOS versions, the dialog works with certificate profiles:

```text
sma01.example.ch> certconfig

Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.

Choose the operation you want to perform:
- CERTIFICATE - Import, Create a request, Edit or Remove Certificate Profiles
- CERTAUTHORITY - Manage System and Customized Authorities
- CRL - Manage Certificate Revocation Lists
[]> certificate
```

Behind `CERTIFICATE` are the operations `IMPORT` (PKCS#12 file previously uploaded to the appliance), `PASTE` (paste certificate into the CLI), `NEW` (generate a self-signed certificate along with a CSR), `EDIT`, `EXPORT`, `DELETE`, and `PRINT` (shows assignment to services). The usual path through SSH is `PASTE`: the dialog asks for a name for the profile, then for the certificate, private key, and optionally the CA intermediate certificate, each as a PEM block, terminated by a single `.` on its own line. The final question about FQDN verification of the Common Name can be answered with the default value. The intermediate must be included in the profile; otherwise, the client chain is incomplete and, depending on the browser, the warning may remain despite a valid certificate.

Older AsyncOS versions (as documented in the Cisco technical note) instead show a `SETUP` dialog. It begins with the question `Do you want to use one certificate/key for receiving, delivery, HTTPS management access, and LDAPS?`: a `y` assigns the same pair to all four services, while a `n` goes through the certificate, key, and intermediate prompts once for each service. The paste procedure is identical.

Two points determine success or failure: do not end the session with Ctrl+C, as that discards all changes immediately. And run `commit` at the end; only then does the certificate become active. With two appliances, repeat the process on both, as certificate configuration is not synchronized between SMAs.

## Verification

The quickest test runs externally against the quarantine page. By default, end-user access to spam quarantine is on HTTPS port 83, unless a different setting was configured when it was enabled:

```bash
openssl s_client -connect spam-quarantine.example.ch:83 \
  -servername spam-quarantine.example.ch </dev/null 2>/dev/null |
  openssl x509 -noout -subject -enddate
```

The output must show the new subject and expiration date. On the appliance, `certconfig` with the `PRINT` operation lists the active certificates, and browser checks against the admin GUI and quarantine page confirm that the chain is built correctly.

## Sources

1.  [How to Generate and Install a Certificate on an SMA](https://www.cisco.com/c/en/us/support/docs/security/content-security-management-appliance/118460-technote-sma-00.html): Cisco technical note with the certconfig procedure for older AsyncOS versions, the PEM requirement, and options for certificate generation through ESA or OpenSSL.

2.  [Common Administrative Tasks, AsyncOS 16.0 for Cisco Secure Email and Web Manager](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_01011.html): Admin Guide chapter on managing Certificate Authority lists (system and custom lists), including checks performed during CA import.

3.  [Comprehensive Spam Quarantine Setup Guide on ESA and SMA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118692-configure-esa-00.html): Cisco guide to spam quarantine, including end-user access through HTTPS on port 83.

4.  [openssl-req](https://docs.openssl.org/master/man1/openssl-req/): Reference for generating keys and CSRs, including `-addext` for the SAN list.

5.  [openssl-pkcs12](https://docs.openssl.org/master/man1/openssl-pkcs12/): Reference for conversion options, including `-noenc` (previously `-nodes`) and `-legacy`.

6.  [OpenSSL 3.0 Migration Guide](https://docs.openssl.org/3.0/man7/migration_guide/): Background on moving legacy algorithms into the legacy provider.

7.  [XCA: X Certificate and Key Management](https://hohnstaedt.de/xca/): Open-source tool for importing and exporting PKCS#12 and PEM structures.

8.  [Win32/Win64 OpenSSL](https://slproweb.com/products/Win32OpenSSL.html): Windows builds from Shining Light Productions referenced by the OpenSSL project, including a published checksum list.

9.  [MSYS2: Filesystem Paths](https://www.msys2.org/docs/filesystem-paths/): Description of automatic path rewriting, which changes the `-subj` argument in Git Bash, along with `MSYS_NO_PATHCONV`.
