---
title: "Renewing a TotemoMail certificate: 4096-bit key, PKCS#12 import and restart per node"
navTitle: "Renew certificate"
description: "TotemoMail’s request dialog generates only 2048-bit keys without alternative names. However, many internal CAs now sign only 4096-bit keys. The key and request are therefore created with openssl, followed by ordering from the PKI authority, PKCS#12 import, port binding and restart per node."
date: "2026-10-06"
kategorie: "TotemoMail"
timeToRead: "12 min read"
themen:
  - totemomail
  - e-mail-verschluesselung
produkte:
  - "totemomail"
protokolle:
  - "tls"
  - "smtp"
slug: "renewing-a-totemomail-certificate-4096-bit-key-pkcs-12-import-and-restart-per-node"
translationId: "article-1e59c4ee01e408a3"
translationOf: totemomail-zertifikat-erneuern
url: https://rafaelpfister.ch/no/blog/renewing-a-totemomail-certificate-4096-bit-key-pkcs-12-import-and-restart-per-node
translationSourceHash: 3ba1992d93abcd33fda47f86cb3b1ea4c8884c36fcfa41fa5c098f4aff9dff32
translationModel: gpt-5.6-terra
translatedAt: 2026-10-06T10:34:59.801Z
translationReview: required
---

# Renewing a TotemoMail certificate: 4096-bit key, PKCS#12 import and restart per node

Renewing the server certificate of a TotemoMail cluster (now Kiteworks Email Protection Gateway) looks like a routine task: generate a request in the administration interface, have it signed, import the response. In practice, this approach often fails at the very first step. The “New PKCS#10” dialog always generates a 2048-bit key and provides no field for alternative names (Subject Alternative Names). Many internal certification authorities now sign only 4096-bit keys, and current TLS peers do not recognise a certificate without alternative names as valid for a name at all.

The following procedure proved effective during a replacement in autumn 2026 in pre-production and production: key and request with openssl on the gateway, order from the PKI authority, import as PKCS#12, bind to the ports and restart node by node. Several peculiarities emerged during the replacement, including an interface error that occurs when detaching the old certificate.

## Two certificates with different purposes

A TotemoMail cluster behind Exchange Online generally requires two types of certificates.

| Type | Purpose | Issuer | Validity |
|---|---|---|---|
| Internal | Web interface, administration, internal SMTP connections. Contains the internal node names. | Internal PKI, such as Active Directory Certificate Services | Freely selectable, typically 12 to 13 months |
| Public | Connection between Exchange Online and the gateway, if Exchange Online is to verify the certificate | Public certification authority | No more than 200 days since 15 March 2026, no more than 100 days from 15 March 2027 |

A single certificate cannot serve both purposes. Public certification authorities do not issue internal names or short names without a domain, and every public certificate appears in Certificate Transparency logs. Conversely, Exchange Online does not trust an internal chain. The article [Mail loop with encryption gateway behind EXO](https://rafaelpfister.ch/blog/verschluesselungsgateway-hinter-exchange-online) describes how Exchange Online handles the connection to an encryption gateway.

The following steps apply to the internal certificate. For the public one, the procedure is identical up to the import; the differences are covered in the final section.

## Overview of the process

1. Generate the key and request with openssl on one node.
2. Order the request from the PKI authority.
3. Check the delivered certificate.
4. Combine the certificate, key and intermediate certificate into a PKCS#12 file and transfer it to the computer with the browser.
5. Import it in the TotemoMail interface and bind it to the ports.
6. Restart each node individually and check it.
7. Test mail flow, then clean up.

Plan at least three weeks before expiry. As long as the old certificate remains valid, there is a fallback option. If pre-production is available in your environment, replace it there first and use the run as a reference.

## Step 1: Key and request with openssl

Work on one cluster node with your personal user account, not with the service account `totemo`. This lets you retrieve the PKCS#12 file directly later using `scp`. Root privileges are not required for any of the steps.

```bash
umask 077
mkdir -m 700 ~/csr-2026
cd ~/csr-2026
```

The configuration contains the subject, intended usages and all alternative names. The names in the example are placeholders: three nodes and the service names under which the cluster is addressed internally.

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
| `default_md = sha256` | Hash method for signing the request |
| `prompt = no` | Use values from the file instead of prompting interactively |
| `req_extensions = ext` | Write extensions from the section `[ ext ]` into the request |
| `basicConstraints = critical, CA:FALSE` | End-entity certificate, not a certification authority |
| `keyUsage` | Key for signing and key exchange, as is customary for TLS servers |
| `extendedKeyUsage = serverAuth, clientAuth` | Server and client authentication. The gateway is a server on some connections and a client on others. |
| `subjectAltName = @alt` | Alternative names from the section `[ alt ]` |

</details>

Include only fully qualified names. Many registration authorities reject short names without a domain, and you should not rely on them ending up in the certificate.

Generate the key without a passphrase and protect it through file permissions. It remains on the node only until import and is deleted afterwards. A passphrase on a key that exists for only a few days provides little protection and creates a new risk: if it is lost, the key becomes unusable and the certificate must be reissued.

```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -out intern.key
openssl req -new -key intern.key -config intern.cnf -out intern.csr
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `genpkey -algorithm RSA` | Generate a new private RSA key |
| `-pkeyopt rsa_keygen_bits:4096` | Key length of 4096 bits |
| `-out intern.key` | File for the key, without passphrase |
| `req -new` | Generate a new certificate signing request (CSR) |
| `-key intern.key` | Use the existing key |
| `-config intern.cnf` | Subject, extensions and names from the configuration |
| `-out intern.csr` | File for the request |

</details>

Check the request before it leaves the machine:

```bash
openssl req -in intern.csr -noout -verify -subject
openssl req -in intern.csr -noout -text | grep -E "Public-Key|Signature Algorithm" | head -2
openssl req -in intern.csr -noout -text | grep -o "DNS:[^,]*" | wc -l
```

Expected are `verify OK`, the correct subject, `4096 bit` and the number of your names. Also note the fingerprint of the public key. You can later use it to unambiguously assign the delivered certificate to this key, even if two requests have the same subject:

```bash
openssl req -in intern.csr -noout -pubkey |
  openssl pkey -pubin -outform DER |
  openssl dgst -sha256
```

## Step 2: Ordering from the PKI authority

The authority issuing the certificate needs these details for each request:

- the CSR as text
- the SHA-256 fingerprint of the public key
- the type: internal or public, and which one if multiple environments exist
- the list of names for copying, one name per line
- key length 4096, extended usages Server Authentication and Client Authentication
- the date by which the certificate must be available, at least two weeks before expiry

For replacements involving multiple environments and certificate types, an overview at the beginning of the email is worthwhile: which certificates exist, which are issued internally and which publicly, and in which order they are required. No system change is required for issuance. Certificates for all environments can therefore be issued simultaneously, even if they are installed sequentially.

During the replacement in autumn 2026, several points became apparent at the registration authority (RA) that are likely to occur similarly in many environments:

- **The RA sets the subject itself.** The certificate contained only country, organisation and Common Name, even though the request contained organisational unit, locality and canton.
- **No short names.** Names without a domain were absent from the issued certificate.
- **Names entered manually.** The RA did not take the alternative names from the CSR; they were entered in the form. One name was truncated. The delivered list of names must therefore always be checked.
- **Wrong profile.** An RA serving both the internal CA and a public CA initially issued requests for public certificates using the internal profile. The issuer showed the internal CA. Such certificates are worthless for the connection to Exchange Online.
- **Only one name for public single certificates.** A single-domain product aborts when given a request with two names, for example with “Only one Subject Alternative Name is allowed”. However, one name is sufficient; see the section on the public certificate.

The PKI authority should subsequently revoke certificates created through incorrect issuance. They contain a valid key and would otherwise remain valid for a year.

## Step 3: Check the delivery

Store the delivered certificate in the same directory as the key, for example with `cat > intern.crt`, paste the content, `Strg+D`. Then check it:

```bash
openssl x509 -in intern.crt -noout -subject -issuer -serial -dates
openssl x509 -in intern.crt -noout -ext subjectAltName,extendedKeyUsage,keyUsage
```

Check the issuer, the complete list of names without typos or duplicates, and both intended usages. Comparing the fingerprints shows whether the certificate and key belong together. Both values must be identical:

```bash
openssl x509 -in intern.crt -noout -pubkey |
  openssl pkey -pubin -outform DER |
  openssl dgst -sha256
openssl pkey -in intern.key -pubout -outform DER |
  openssl dgst -sha256
```

For a public certificate, two additional checks apply. The policy OID `2.23.140.1.2.2` stands for an organisation-validated certificate under the rules of the CA/Browser Forum, and the certificate must carry embedded Certificate Transparency proofs. A few minutes after issuance, it appears under its name on crt.sh. If either is missing, it is not a public certificate, regardless of how the delivery is labelled.

## Step 4: Build the PKCS#12 file

TotemoMail imports the certificate and key together as PKCS#12. To do this, obtain the certificate of the issuing intermediate CA. Its address is listed in the certificate under `Authority Information Access`:

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

The Subject Key Identifier of the intermediate CA must match the Authority Key Identifier of the certificate. Then build the file. The command prompts for an export password, which you need for the import:

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
| `-certfile issuing.crt` | Additional certificates in the chain, here the intermediate CA |
| `-name "SecureMail"` | Display name of the entry in the file |
| `-out intern.p12` | Target file, protected with the export password |

</details>

Check it; two entries must appear, the certificate and the intermediate CA:

```bash
openssl pkcs12 -in intern.p12 -nokeys 2>/dev/null | grep -E "subject=|issuer="
```

The TotemoMail interface runs in the browser, typically on a jump host. The file must be transferred there. On Windows, the OpenSSH client with `scp` is available:

```powershell
New-Item -ItemType Directory -Path "$env:USERPROFILE\Downloads\zert" -Force | Out-Null
scp benutzer@gw01.intern.example.ch:csr-2026/intern.p12 "$env:USERPROFILE\Downloads\zert\"
scp benutzer@gw01.intern.example.ch:csr-2026/issuing.crt "$env:USERPROFILE\Downloads\zert\"
```

If you generated the key as `totemo` after all, it is located under `/opt/totemomail`, and your user cannot read it. In that case, briefly copy the PKCS#12 file to `/tmp`, retrieve it there and delete it again immediately. The detour via the clipboard using Base64 works, but is error-prone for a line of around 10,000 characters.

## Step 5: Import and bind in TotemoMail

Under `Key Management`:

1. **`Issuer Certificates`**: import the intermediate CA `issuing.crt`.
2. **`Own Server Certificates`**, button **`Import`**: The dialog offers two methods. On the left, “Import certificate” is for a certificate with key, i.e. the PKCS#12 file. On the right, “Import a PKCS#10 certificate reply” is only for responses to requests generated by TotemoMail itself. Choose the left option and enter the export password in the second step.
3. Open the new certificate (pencil icon). Under `Connector` all ports should be selected; in the installation described, these are `8443=Admin`, `443=SecMail`, `7444=MailAPI`, `10443=SENDIT` and `8444=AdminAPI`. `Host` is set to `*`.
4. Open the old certificate and deselect all ports except one.

Point 4 involves the error that occurred during the replacement: if you deselect **all** ports on the old certificate, the interface reports “Could not edit selected server certificate”. The log contains:

```
ERROR [EditServerCertBean] Could not edit certificate
ch.totemo.core.actions.ActionException: Failed to edit key.
Caused by: java.lang.NullPointerException
```

TotemoMail does not tolerate a server certificate without a port. Therefore leave one port selected, for example `8444=AdminAPI`, and delete the old certificate after the observation period. Deleting instead of deselecting works; the certificate ends up under `Deleted Certificates`.

Three observations help explain the binding:

- **The port list applies only to web services.** Port 25 does not appear there. If no certificate of type SMTPS exists, SMTP uses the HTTPS certificate. This is why port 25 presents the new certificate after replacement, even though SMTPS is not selected in the list.
- **An explicit assignment takes precedence over the asterisk.** The port left on the old certificate continues to present the old certificate, even if the new one is registered for all using `*`.
- **The list is shared across the cluster.** It looks the same on every node, regardless of whether the node has already applied the change. Only a query on the ports shows what a node actually presents.

## Step 6: Restart node by node

TotemoMail reads certificates when it starts. After import, all nodes continue to show the old certificate until they are restarted. Restart every node individually, never all at once, so that nodes remain active behind the load balancer at all times. Begin with the nodes through which you are not logged in, and restart the node with the open interface last.

On the node as `totemo`:

```bash
totemomail stop
totemomail start
```

Further information on controlled shutdown is available in the article [Most important controls for TotemoMail admins](https://rafaelpfister.ch/blog/totemomail-server-stoppen-queues-bereinigen).

The web services become available again after about one minute, while the SMTP service on port 25 takes several minutes. An empty response on port 25 shortly after startup therefore does not yet indicate an error. Only once a node has switched on all ports should the next one be restarted.

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
| `-starttls smtp` | On port 25, perform the SMTP dialogue first, then switch to TLS with STARTTLS |
| `x509 -noout` | Read the certificate without outputting it |
| `-serial -enddate` | Display serial number and expiry date |

</details>

If port 25 remains empty after five minutes, `ss -lnt | grep ':25 '` shows whether the service is listening, and the log under `/opt/totemomail` shows the reason.

The option `-showcerts` shows whether TotemoMail sends the intermediate CA as well:

```bash
echo | openssl s_client -connect gw01:25 -starttls smtp -showcerts 2>/dev/null | grep -E " s:| i:"
```

In the replacement described, only the end-entity certificate appeared, although the intermediate CA was in the PKCS#12 file and under `Issuer Certificates`. This has no consequences for internal connections where no one verifies the certificate. For verification by Exchange Online, however, the chain must be complete.

## Step 7: Tests, fallback and cleanup

After the final restart, test mail flow in both directions: one message from external through the loop and one to external through the gateway. In Exchange Online message trace, both legs must appear as delivered, and nothing may remain in the queue towards the gateway.

The **fallback** consists of selecting the ports again on the old certificate, deselecting all but one on the new certificate, and restarting the nodes individually once more. This works only while the old certificate is valid.

After successful tests:

- delete the PKCS#12 file on the jump host
- delete the working directory on the node; the key is now in TotemoMail’s key store
- after a few days, delete the old certificate and restart the nodes individually once more so that the final port also switches over
- have incorrectly issued certificates revoked by the PKI authority
- enter the new expiry date in the reminder system

Notify administrators of the replacement if the new certificate no longer contains short names. Anyone who previously accessed the interface using `https://gw01:8443` will then see a certificate warning. This also applies to monitoring and scripts that use short names.

## Peculiarities at a glance

| Observation | Consequence | Handling |
|---|---|---|
| “New PKCS#10” generates 2048 bits without alternative names | Request unusable for 4096-bit CAs | Generate key and request with openssl |
| Changes take effect only after restart | Nodes continue to present the old certificate | Restart each node individually |
| Port 25 comes up several minutes after web services | Empty response shortly after startup | Wait before proceeding to the next node |
| Certificate without a port triggers a NullPointerException | Old certificate cannot be fully detached | Leave one port selected, delete later |
| Explicit assignment takes precedence over `*` | A port continues to present the old certificate | Delete old certificate after the observation period |
| Only the end-entity certificate is sent | Peers that verify cannot build the chain | Clarify before verification by Exchange Online |
| RA enters names manually | Typos and missing names are possible | Check the delivered list of names |

## The public certificate for the connection to Exchange Online

For Exchange Online to verify the connection to the gateway in addition to encrypting it, the gateway needs a public certificate on port 25. Only then can the outgoing connector be configured with `TlsSettings DomainValidation` and a `TlsDomain`, and the incoming connector bound to the certificate via `TlsSenderCertificateName`.

A single name is sufficient for this. Exchange Online compares only the name in the certificate with the configured value in both directions; it does not verify the address behind it. A certificate for the name under which the gateway is known externally covers both directions. When ordering, explicitly ask for Client Authentication: several public certification authorities removed this usage from TLS certificates in 2026. It is required for the return path, where the gateway identifies itself as a client.

For use in TotemoMail, the observations above suggest this approach: import the public certificate as type SMTPS so that port 25 presents it while the web services retain the internal one. Two points must be clarified beforehand. The chain must be sent in full, and port 25 will then present the public certificate to all senders, including internal ones such as an upstream gateway. If a sender expects a specific certificate, that connection will fail.

## Sources

1.  [CA/Browser Forum: Ballot SC081v3](https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/): Roadmap for the validity period of public TLS certificates: 200 days from March 2026, 100 days from March 2027, 47 days from March 2029.

2.  [Microsoft Learn: Set-OutboundConnector](https://learn.microsoft.com/powershell/module/exchange/set-outboundconnector): Parameters `TlsSettings` and `TlsDomain` for verifying the certificate on the remote side.

3.  [Microsoft Learn: Set-InboundConnector](https://learn.microsoft.com/powershell/module/exchange/set-inboundconnector): Parameter `TlsSenderCertificateName` for assignment via the sender’s certificate.

4.  [OpenSSL documentation: openssl-req](https://docs.openssl.org/master/man1/openssl-req/): Structure of the configuration file and options for certificate requests.

5.  [OpenSSL documentation: openssl-pkcs12](https://docs.openssl.org/master/man1/openssl-pkcs12/): Creating and checking PKCS#12 files.

6.  [crt.sh](https://crt.sh): Search Certificate Transparency logs to verify issuance of a public certificate.
