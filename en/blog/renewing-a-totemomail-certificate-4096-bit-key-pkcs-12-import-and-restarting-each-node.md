---
title: "Renewing a Totemomail Certificate: 4096-Bit Key, PKCS#12 Import, and Restarting Each Node"
navTitle: "Renew certificate"
description: "Totemomail’s request dialog generates only 2048-bit keys without alternative names. However, many internal CAs now sign only 4096-bit keys. The key and request are therefore created with openssl, followed by ordering from the PKI team, PKCS#12 import, connector binding, and restarting each node."
date: "2026-10-06"
kategorie: "Totemomail"
timeToRead: "12 min read"
themen:
  - totemomail
  - e-mail-verschluesselung
produkte:
  - "totemomail"
protokolle:
  - "tls"
  - "smtp"
slug: "renewing-a-totemomail-certificate-4096-bit-key-pkcs-12-import-and-restarting-each-node"
translationId: "article-1e59c4ee01e408a3"
translationOf: totemomail-zertifikat-erneuern
url: https://rafaelpfister.ch/en/blog/renewing-a-totemomail-certificate-4096-bit-key-pkcs-12-import-and-restarting-each-node
translationSourceHash: 3ba1992d93abcd33fda47f86cb3b1ea4c8884c36fcfa41fa5c098f4aff9dff32
translationModel: gpt-5.6-terra
translatedAt: 2026-10-06T10:31:20.473Z
translationReview: required
---

# Renewing a Totemomail Certificate: 4096-Bit Key, PKCS#12 Import, and Restarting Each Node

Renewing the server certificate of a Totemomail cluster (now Kiteworks Email Protection Gateway) looks like a routine task: generate a request in the management interface, have it signed, import the response. In practice, this approach often fails at the very first step. The “New PKCS#10” dialog always generates a 2048-bit key and provides no field for alternative names (Subject Alternative Names). Many internal certification authorities now sign only 4096-bit keys, and current TLS peers do not even recognize a certificate without alternative names as valid for a name.

The following procedure proved effective during a replacement in fall 2026 in pre-production and production: create the key and request with openssl on the gateway, order it from the PKI team, import it as PKCS#12, bind it to the connectors, and restart each node individually. Several peculiarities became apparent during the replacement, including an interface error that occurs when unbinding the old certificate.

## Two certificates with different purposes

A Totemomail cluster behind Exchange Online generally needs two types of certificates.

| Type | Purpose | Issuer | Validity |
|---|---|---|---|
| Internal | Web interface, administration, internal SMTP connections. Contains the internal names of the nodes. | internal PKI, such as Active Directory Certificate Services | selectable, typically 12 to 13 months |
| Public | Connection between Exchange Online and the gateway, if Exchange Online is to validate the certificate | public certification authority | no more than 200 days since 03/15/2026, no more than 100 days starting 03/15/2027 |

A single certificate cannot serve both purposes. Public certification authorities do not issue internal names or short names without a domain, and every public certificate appears in Certificate Transparency logs. Conversely, Exchange Online does not trust an internal chain. The article [Mail loop with encryption gateway behind EXO](https://rafaelpfister.ch/blog/verschluesselungsgateway-hinter-exchange-online) describes how Exchange Online handles the connection to an encryption gateway.

The following steps apply to the internal certificate. For the public certificate, the process is identical up to the import; the differences are covered in the final section.

## Process overview

1. Create the key and request with openssl on one node.
2. Order the request from the PKI team.
3. Verify the delivered certificate.
4. Combine the certificate, key, and intermediate certificate into a PKCS#12 file and retrieve it to the computer with the browser.
5. Import it into the Totemomail interface and bind it to the connectors.
6. Restart and verify each node individually.
7. Test mail flow, then clean up.

Plan at least three weeks before expiration. As long as the old certificate is valid, there is a way back. If pre-production is available in your environment, replace it there first and use that run as a reference.

## Step 1: Key and request with openssl

Work on one node in the cluster using your personal account, not the `totemo` service account. This lets you retrieve the PKCS#12 file directly later with `scp`. No root permissions are required for any of these steps.

```bash
umask 077
mkdir -m 700 ~/csr-2026
cd ~/csr-2026
```

The configuration contains the subject, intended uses, and all alternative names. The names in the example are placeholders: three nodes and the service names under which the cluster is addressed internally.

```bash
cat > intern.cnf <<'EOF'
[ req ]
default_md         = sha256
prompt             = no
distinguished_name = dn
req_extensions     = ext

[ dn ]
C  = CH
O  = Beispiel AG
CN = SecureMail

[ ext ]
basicConstraints = critical, CA:FALSE
keyUsage         = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName   = @alt

[ alt ]
DNS.1 = gw01.intern.example.ch
DNS.2 = gw02.intern.example.ch
DNS.3 = gw03.intern.example.ch
DNS.4 = securemail.intern.example.ch
EOF
```

<details class="options-details">
<summary>Options explained</summary>

| Entry | Effect |
|---|---|
| `default_md = sha256` | Hash algorithm for signing the request |
| `prompt = no` | Use values from the file instead of prompting interactively |
| `req_extensions = ext` | Write extensions from the `[ ext ]` section into the request |
| `basicConstraints = critical, CA:FALSE` | End-entity certificate, not a certification authority |
| `keyUsage` | Key for signing and key exchange, as customary for TLS servers |
| `extendedKeyUsage = serverAuth, clientAuth` | Server and client authentication. The gateway is a server on some connections and a client on others. |
| `subjectAltName = @alt` | Alternative names from the `[ alt ]` section |

</details>

Include only fully qualified names. Many registration authorities reject short names without a domain, and you should not rely on them ending up in the certificate.

Create the key without a passphrase and protect it with file permissions. It remains on the node only until import and is deleted afterward. A passphrase on a key that exists for only a few days offers little protection and creates a new risk: if it is lost, the key becomes unusable and the certificate must be reissued.

```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -out intern.key
openssl req -new -key intern.key -config intern.cnf -out intern.csr
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `genpkey -algorithm RSA` | Generate a new RSA private key |
| `-pkeyopt rsa_keygen_bits:4096` | Key length of 4096 bits |
| `-out intern.key` | Key file, without a passphrase |
| `req -new` | Generate a new certificate signing request (CSR) |
| `-key intern.key` | Use the existing key |
| `-config intern.cnf` | Subject, extensions, and names from the configuration |
| `-out intern.csr` | Request file |

</details>

Verify the request before it leaves the machine:

```bash
openssl req -in intern.csr -noout -verify -subject
openssl req -in intern.csr -noout -text | grep -E "Public-Key|Signature Algorithm" | head -2
openssl req -in intern.csr -noout -text | grep -o "DNS:[^,]*" | wc -l
```

Expected are `verify OK`, the correct subject, `4096 bit`, and the number of your names. Also record the fingerprint of the public key. You can later use it to unambiguously match the delivered certificate to this key, even if two requests have the same subject:

```bash
openssl req -in intern.csr -noout -pubkey |
  openssl pkey -pubin -outform DER |
  openssl dgst -sha256
```

## Step 2: Ordering from the PKI team

The team issuing the certificate needs the following information for each request:

- the CSR as text
- the SHA-256 fingerprint of the public key
- the type: internal or public, and which environment if there are multiple environments
- the name list for copying, one name per line
- key length 4096, extended use Server Authentication and Client Authentication
- the date by which the certificate must be available, at least two weeks before expiration

When replacing certificates across multiple environments and certificate types, it is worth including an overview at the start of the email: which certificates exist, which are issued internally and which publicly, and the order in which they are needed. Issuance requires no system change. Certificates for all environments can therefore be issued at the same time, even if they are installed one after another.

During the replacement in fall 2026, several points became apparent at the registration authority (RA) that are likely to occur similarly in many environments:

- **The RA sets the subject itself.** The certificate contained only country, organization, and Common Name, even though the request included organizational unit, locality, and canton.
- **No short names.** Names without a domain were absent from the issued certificate.
- **Names entered manually.** The RA did not take the alternative names from the CSR; they were typed into the form. One name arrived truncated. The delivered name list must therefore always be verified.
- **Wrong profile.** An RA serving both the internal CA and a public CA initially issued requests for public certificates through the internal profile. The issuer was the internal CA. Such certificates are useless for the connection to Exchange Online.
- **Only one name for public single certificates.** A single-domain product fails with a request containing two names, for example with “Only one Subject Alternative Name is allowed”. However, one name is sufficient; see the section on the public certificate.

The PKI team should subsequently revoke certificates resulting from incorrect issuance. They have a valid key and would otherwise remain valid for a year.

## Step 3: Verify delivery

Place the delivered certificate in the same directory as the key, for example with `cat > intern.crt`, paste the contents, `Strg+D`. Then verify it:

```bash
openssl x509 -in intern.crt -noout -subject -issuer -serial -dates
openssl x509 -in intern.crt -noout -ext subjectAltName,extendedKeyUsage,keyUsage
```

Check the issuer, the complete name list without typos or duplicates, and both intended uses. Comparing the fingerprints shows whether the certificate and key belong together. Both values must be identical:

```bash
openssl x509 -in intern.crt -noout -pubkey |
  openssl pkey -pubin -outform DER |
  openssl dgst -sha256
openssl pkey -in intern.key -pubout -outform DER |
  openssl dgst -sha256
```

For a public certificate, add two checks. The policy OID `2.23.140.1.2.2` denotes an organization-validated certificate under CA/Browser Forum rules, and the certificate must contain embedded Certificate Transparency proofs. A few minutes after issuance, it appears under its name on crt.sh. If either is missing, it is not a public certificate, regardless of how the delivery is labeled.

## Step 4: Build the PKCS#12 file

Totemomail imports the certificate and key together as PKCS#12. To do so, obtain the issuing intermediate certificate. Its address is listed in the certificate under `Authority Information Access`:

```bash
openssl x509 -in intern.crt -noout -ext authorityInfoAccess
curl -sS -o issuing.crt http://pki.example.ch/crt/Issuing-CA.crt
file issuing.crt
```

If `file` does not report `PEM certificate`, the file is DER-encoded and must be converted:

```bash
openssl x509 -inform DER -in issuing.crt -out issuing.pem
mv issuing.pem issuing.crt
```

The Subject Key Identifier of the intermediate certificate must match the Authority Key Identifier of the certificate. Then build the file. The command prompts for an export password, which you need during import:

```bash
openssl pkcs12 -export \
  -inkey intern.key \
  -in intern.crt \
  -certfile issuing.crt \
  -name "SecureMail" \
  -out intern.p12
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-export` | Create a PKCS#12 file |
| `-inkey intern.key` | Private key |
| `-in intern.crt` | Issued certificate |
| `-certfile issuing.crt` | Additional chain certificates, here the intermediate certificate |
| `-name "SecureMail"` | Display name of the entry in the file |
| `-out intern.p12` | Target file, protected with the export password |

</details>

Verify it: two entries must appear, the certificate and the intermediate certificate:

```bash
openssl pkcs12 -in intern.p12 -nokeys 2>/dev/null | grep -E "subject=|issuer="
```

The Totemomail interface runs in a browser, typically on a jump host. The file must be transferred there. On Windows, the OpenSSH client with `scp` is available:

```powershell
New-Item -ItemType Directory -Path "$env:USERPROFILE\Downloads\zert" -Force | Out-Null
scp benutzer@gw01.intern.example.ch:csr-2026/intern.p12 "$env:USERPROFILE\Downloads\zert\"
scp benutzer@gw01.intern.example.ch:csr-2026/issuing.crt "$env:USERPROFILE\Downloads\zert\"
```

If you created the key as `totemo`, it is located under `/opt/totemomail`, and your user cannot read it. In that case, briefly copy the PKCS#12 file to `/tmp`, retrieve it there, and delete it again immediately. The workaround using the clipboard with Base64 works, but is error-prone with a line of around 10,000 characters.

## Step 5: Import and bind in Totemomail

Under `Key Management`:

1. **`Issuer Certificates`**: import the intermediate certificate `issuing.crt`.
2. **`Own Server Certificates`**, button **`Import`**: The dialog offers two paths. On the left, “Import certificate” is for a certificate with a key, i.e., the PKCS#12 file. On the right, “Import a PKCS#10 certificate reply” is only for responses to requests generated by Totemomail itself. Choose the left option and enter the export password in the second step.
3. Open the new certificate (pencil icon). Under `Connector` all connectors should be selected; in the installation described, these are `8443=Admin`, `443=SecMail`, `7444=MailAPI`, `10443=SENDIT`, and `8444=AdminAPI`. `Host` is set to `*`.
4. Open the old certificate and deselect all connectors except one.

Item 4 involves the error encountered during the replacement: if you deselect **all** connectors on the old certificate, the interface reports “Could not edit selected server certificate”. The log contains:

```
ERROR [EditServerCertBean] Could not edit certificate
ch.totemo.core.actions.ActionException: Failed to edit key.
Caused by: java.lang.NullPointerException
```

Totemomail does not allow a server certificate without a connector. Therefore, leave one connector in place, such as `8444=AdminAPI`, and delete the old certificate after the observation period. Deleting instead of deselecting works; the certificate ends up under `Deleted Certificates`.

Three observations help explain the binding:

- **The connector list applies only to web services.** Port 25 does not appear there. If no certificate of type SMTPS is present, SMTP uses the HTTPS certificate. Therefore, port 25 presents the new certificate after replacement even though SMTPS is not selected in the list.
- **An explicit assignment takes precedence over the asterisk.** The connector left on the old certificate continues to present the old one, even if the new one is registered for all with `*`.
- **The list is shared across the cluster.** It looks the same on every node, regardless of whether that node has already applied the change. Only a query on the ports shows what a node actually presents.

## Step 6: Restart each node individually

Totemomail loads certificates at startup. After import, all nodes continue to present the old certificate until they are restarted. Restart each node individually, never all at once, so that nodes remain active behind the load balancer at all times. Start with the nodes you are not logged into, and leave the node with the open interface until last.

On the node as `totemo`:

```bash
totemomail stop
totemomail start
```

Further guidance on controlled shutdown is available in the article [Most important controls for Totemomail admins](https://rafaelpfister.ch/blog/totemomail-server-stoppen-queues-bereinigen).

The web services are reachable again after about one minute, but the SMTP service on port 25 only after several minutes. An empty response on port 25 shortly after startup therefore does not yet indicate an error. Move on to the next node only once a node has switched over on all ports.

Check all nodes and ports:

```bash
for h in gw01 gw02 gw03; do
  for p in 25 443 8443; do
    if [ "$p" = 25 ]; then s="-starttls smtp"; else s=""; fi
    c=$(echo | openssl s_client -connect "$h:$p" $s 2>/dev/null |
        openssl x509 -noout -serial -enddate 2>/dev/null | tr '\n' ' ')
    printf "%-6s %-5s %s\n" "$h" "$p" "$c"
  done
done
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `s_client -connect host:port` | Establish a TLS connection to the service |
| `-starttls smtp` | On port 25, first perform the SMTP dialog and then switch to TLS with STARTTLS |
| `x509 -noout` | Read the certificate without outputting it |
| `-serial -enddate` | Display serial number and expiration date |

</details>

If port 25 remains empty after five minutes, `ss -lnt | grep ':25 '` shows whether the service is listening, and the log under `/opt/totemomail` shows why.

The `-showcerts` option shows whether Totemomail sends the intermediate certificate:

```bash
echo | openssl s_client -connect gw01:25 -starttls smtp -showcerts 2>/dev/null | grep -E " s:| i:"
```

In the replacement described, only the end-entity certificate appeared, even though the intermediate certificate was in the PKCS#12 file and under `Issuer Certificates`. This has no consequences for internal connections where no one validates the certificate. For validation by Exchange Online, however, the chain must be complete.

## Step 7: Tests, rollback, cleanup

After the final restart, test mail flow in both directions: a message from outside through the loop and one to the outside through the gateway. In Exchange Online message tracing, both legs must appear as delivered, and nothing may be stuck in the queue toward the gateway.

The **rollback** consists of selecting the connectors on the old certificate again, deselecting all but one on the new certificate, and restarting the nodes individually again. This works only while the old certificate remains valid.

After successful tests:

- delete the PKCS#12 file on the jump host
- delete the working directory on the node; the key is now in the Totemomail key store
- after a few days, delete the old certificate and restart the nodes individually once more so that the last connector also switches over
- have the PKI team revoke incorrectly issued certificates
- add the new expiration date to your reminder system

Notify administrators about the replacement if the new certificate no longer contains short names. Anyone who previously accessed the interface using `https://gw01:8443` will then see a certificate warning. This also applies to monitoring and scripts that use short names.

## Peculiarities at a glance

| Observation | Consequence | Handling |
|---|---|---|
| “New PKCS#10” generates 2048 bits without alternative names | Request unusable for 4096-bit CAs | Create key and request with openssl |
| Changes take effect only after restart | Nodes continue to present the old certificate | Restart each node individually |
| Port 25 comes up several minutes after web services | Empty response shortly after startup | Wait before proceeding to the next node |
| Certificate without a connector triggers a NullPointerException | Old certificate cannot be fully unbound | Leave one connector in place, delete later |
| Explicit assignment takes precedence over `*` | A connector continues to present the old certificate | Delete the old certificate after observation |
| Only the end-entity certificate is sent | Peers that validate cannot build the chain | Clarify before validation by Exchange Online |
| RA enters names manually | Typos and missing names are possible | Verify the name list in the delivery |

## The public certificate for the connection to Exchange Online

For Exchange Online to validate the connection to the gateway in addition to encrypting it, the gateway needs a public certificate on port 25. Only then can the outbound connector be changed to `TlsSettings DomainValidation` with a `TlsDomain`, and the inbound connector bound to the certificate through `TlsSenderCertificateName`.

A single name is sufficient for this. Exchange Online compares only the name in the certificate with the configured value in both directions; it does not validate the address behind it. A certificate for the name under which the gateway is publicly known covers both directions. When ordering, explicitly ask for Client Authentication: several public certification authorities removed this use from TLS certificates in 2026. It is required for the return path, where the gateway identifies itself as a client.

Based on the observations above, the following approach appears appropriate for use in Totemomail: import the public certificate as type SMTPS so that port 25 presents it while the web services retain the internal certificate. Two points must be clarified first. The chain must be sent completely, and port 25 will then present the public certificate to all senders, including internal ones such as an upstream gateway. If a sender expects a specific certificate, that connection will fail.

## Sources

1.  [CA/Browser Forum: Ballot SC081v3](https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/): Timeline for public TLS certificate validity, 200 days starting in March 2026, 100 days starting in March 2027, and 47 days starting in March 2029.

2.  [Microsoft Learn: Set-OutboundConnector](https://learn.microsoft.com/powershell/module/exchange/set-outboundconnector): Parameters `TlsSettings` and `TlsDomain` for validating the certificate on the remote side.

3.  [Microsoft Learn: Set-InboundConnector](https://learn.microsoft.com/powershell/module/exchange/set-inboundconnector): Parameter `TlsSenderCertificateName` for assignment through the sender's certificate.

4.  [OpenSSL documentation: openssl-req](https://docs.openssl.org/master/man1/openssl-req/): Structure of the configuration file and options for certificate requests.

5.  [OpenSSL documentation: openssl-pkcs12](https://docs.openssl.org/master/man1/openssl-pkcs12/): Creating and verifying PKCS#12 files.

6.  [crt.sh](https://crt.sh): Search Certificate Transparency logs to verify the issuance of a public certificate.
