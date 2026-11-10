---
title: "Proofpoint Protection Server 8.22.4: Patch 5807 leaves SMTP Client without a certificate"
navTitle: "PPS 8.22.4 Certificate"
description: "Patches 5806 and 5807 introduce a separate “SMTP Client” assignment under System > Certificates > Services in Proofpoint Protection Server 8.22.4. If it is set to “-none-”, Sendmail delivers without a TLS client certificate. Peers that validate certificates then reject the messages, such as Exchange Online with 5.7.51 or a smarthost with “Relay access denied”. Here is how to check and fix it."
date: "2026-11-05"
kategorie: "Proofpoint"
timeToRead: "7 min to read"
themen:
  - proofpoint
  - smtp-mailflow
produkte:
  - "proofpoint-pps"
  - "exchange-online"
protokolle:
  - "tls"
  - "smtp"
  - "troubleshooting"
slug: "proofpoint-protection-server-8-22-4-patch-may-reset-the-certificate-configuration-for-smtp"
translationId: "article-6f99807d289cc394"
warnung: true
featured: "2026-11-05"
translationOf: proofpoint-smtp-client-zertifikat
translationSourceHash: fcb36b2ae8af7420156aca54a99d61f4702aeb47781e1081e27edf7419e77db8
translationModel: gpt-5.6-terra
translatedAt: 2026-11-07T03:19:14.931Z
translationReview: required
aktualisiert: 2026-11-06
url: https://rafaelpfister.ch/en/blog/proofpoint-protection-server-8-22-4-patch-may-reset-the-certificate-configuration-for-smtp
---

Today, I investigated an outage in the outbound mail flow of a Proofpoint Protection Server (PPS) 8.22.4. In short: After an automatically installed patch, the **SMTP Client** assignment under **System > Certificates > Services** was set to `-none-`. Sendmail then established outbound connections without a TLS client certificate. All peers that identify Proofpoint through this certificate rejected the emails. Regular recipients using opportunistic TLS continued accepting mail, so the issue was only noticed when individual partners could no longer be reached.

If you run PPS with automatic patches and have partners that require a client certificate, check this setting after every patch.

## Symptom

Only connections where the peer validates Proofpoint's client certificate are affected. Two typical cases:

- **Exchange Online with a partner connector** restricted to a certificate through `RestrictDomainsToCertificate`. The rejection is `550 5.7.51 TenantInboundAttribution; Rejecting. Recipient has a partner connector with RestrictDomainsToIPAddresses or RestrictDomainsToCertificate set.`
- **A smarthost that allows relaying only with a client certificate.** The response is `554 5.7.1 <empfaenger@example.com>: Relay access denied`.

All other destinations continue receiving their mail. The test matrix therefore shows an inconsistent picture: mail to external recipients gets through, while mail to individual partners or through a particular smarthost fails.

The agent mail log contains one of the following two messages for every outbound connection. If the file is missing entirely:

```text
sendmail[...]: STARTTLS=client: file /etc/pki/tls/certs/sendmail-client.pem unsafe:
  No such file or directory
```

If the file exists but is empty:

```text
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_use_PrivateKey_file(/etc/pki/tls/certs/sendmail-client.pem) failed
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_use_certificate_chain_file(/etc/pki/tls/certs/sendmail-client.pem) failed
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_check_private_key failed(/etc/pki/tls/certs/sendmail-client.pem): 0
```

Immediately afterward, Sendmail still establishes the connection with TLS (`verify=OK`), but without a client certificate. To the peer, this looks like an unknown sender.

## Cause

PPS assigns a certificate to each service on each server. Under **System > Certificates > Services**, each server has, among others, `SMTP Server`, `SMTP Client`, `Administration Server` and `End User Web Application / Encryption`. PPS generates the file `/etc/pki/tls/certs/sendmail-client.pem` from the assignment for `SMTP Client`, which Sendmail uses as its client certificate. If it is set to `-none-`, PPS writes the file as empty or does not write it at all.

The `SMTP Client` assignment has only existed in 8.22.4 since patches 5806 and 5807, which Proofpoint released on October 29, 2026. Both are titled “TLS client and server certificate authentication improvement” and, according to the Knowledge Base, fix the inability of PPS 8.22.4 to assign separate certificates to the SMTP server and SMTP client. The feature is included as standard in release 8.23.0 (Release Notes, section “SMTP Client Certificate”).

| Patch | Content according to the Knowledge Base |
|---|---|
| 5806 | Updates, among other things, `SysCertificates.pm`, `Certificate.pm`, `TLS.pm`, `pp-sendmail.pl` and the template `sendmail.mc`, restarts the Admin GUI (`admind`), and updates the RPM `pps-pfpt` |
| 5807 | Requires 5806, changes no files, and restarts no processes; it only applies the configuration changes required for separate server and client certificates |

Proofpoint therefore introduced the assignment rather than resetting an existing one. The patch does not carry over a certificate: After installation, the new row is set to `-none-`, and Sendmail no longer uses a client certificate from that point onward. The patch descriptions contain no notice of this.

In the environment investigated, the assignment was set to `-none-` on all servers afterward. Under **System > Config History**, patch 5807 was visible as an entry `configuration changes for patch-…` created by `system`, and the files under `/etc/pki/tls/certs/` had the same timestamp. However, 5807 did not appear under **Software Patch History**. This is consistent with 5807 not installing software but only changing configuration. **Config History** therefore shows when the change occurred, while the software change itself belongs to patch 5806.

A rollback under **Config History** to the patch snapshot did not help. PPS rewrote the file afterward, but with 0 bytes.

## Check

In the Admin GUI on the master, under **System > Certificates > Services**, select each agent that delivers mail externally in turn at the top under **Server**, and inspect the `SMTP Client` row.

On the agent itself, two commands show the status:

```bash
ls -l --time-style=full-iso /etc/pki/tls/certs/
grep -E 'sendmail-client.pem|SSL_CTX_' /var/log/maillog | tail -5
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-l` | Long format with permissions, owner, size, and date |
| `--time-style=full-iso` | Timestamp with date, time, and time zone, for comparison with the patch time |
| `grep -E` | Extended regular expressions, allowing both search terms to be combined with `\|` |
| `tail -5` | Display only the last five matches |

</details>

If `sendmail-client.pem` is missing or the file is 0 bytes while `sendmail.pem` is several thousand bytes in size, no certificate is assigned to the `SMTP Client` service.

## Fix

Under **System > Certificates > Services**, select a valid certificate for `SMTP Client` on each affected agent and save it with **Save Changes**. Usually, this is the same public certificate assigned to `SMTP Server`.

The selection list may contain multiple certificates with the same name, including expired ones. They can only be distinguished by the serial number in parentheses. Before saving, compare the serial number with the list under **System > Certificates > Certificates**, where the issue and expiration dates are shown.

The list for `SMTP Client` does not necessarily show all imported certificates. In the environment investigated, some certificates were missing even though they were valid and expired copies had already been deleted. Neither the patch articles nor the 8.23 Release Notes describe the criteria PPS uses to filter the list. If a certificate is missing, check two points:

- **Private key:** If the certificate was imported as a PEM without a key, PPS cannot assign it to a service. PKCS12 files always contain the key.
- **Purpose:** Does the certificate include the purpose `TLS Web Client Authentication` in `Extended Key Usage`? Many public certificates issued since fall 2025 no longer include it (see below). A certificate without this purpose is unsuitable for the `SMTP Client` service anyway.

If Proofpoint confirms the criteria, I will add them here.

After saving, PPS distributes the configuration to the agents. Check on the agent:

```bash
ls -l /etc/pki/tls/certs/sendmail-client.pem
grep -E 'STARTTLS=client|stat=' /var/log/maillog | tail -20
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `ls -l` | Shows the file size and permissions; after the fix, it is no longer empty |
| `grep -E 'STARTTLS=client\|stat='` | Filters TLS messages for outbound connections and the delivery result |
| `tail -20` | Display only the last twenty lines |

</details>

The messages `SSL_CTX_… failed` and `unsafe` must no longer appear after saving, and deliveries to partners must end with `stat=Sent`. Sendmail delivers messages that were deferred with a 4xx code from the queue automatically. Messages rejected with a 5xx code must be resent by the senders.

Manually copying `sendmail.pem` to `sendmail-client.pem` also bridges the outage. However, PPS overwrites the file again during the next configuration push as long as the assignment is missing in the GUI.

## Check after every patch

Patch 5807 changed the configuration without the patch description indicating the consequences. I therefore recommend checking the **Services** page after every patch and looking in **Config History** to see whether the patch included configuration changes. If patch 5807 is not yet installed on an 8.22.4 system, assign a certificate to `SMTP Client` immediately after installation. Even simpler is monitoring that alerts on the `sendmail-client.pem` message in the agents' mail logs. Under **Licenses and Updates > Settings**, you can also specify when patches are installed. A maintenance window during which someone can check mail flow substantially shortens such an outage.

## Client Authentication is disappearing from public certificates

Independently of the patch, a second change affects these exact connections. Public TLS certificates have generally contained both purposes in the `Extended Key Usage` extension: `TLS Web Server Authentication` and `TLS Web Client Authentication`. Mail gateways therefore also use their server certificate as a client certificate.

The Chrome Root Program requires that public certificates issued from March 15, 2027 onward contain only `serverAuth`. DigiCert has issued public TLS certificates without Client Authentication by default since October 1, 2025; the purpose is still available upon request until March 1, 2027. Other CAs have comparable schedules. OpenSSL shows whether a certificate includes the client purpose:

```bash
openssl x509 -in zertifikat.pem -noout \
  -subject -enddate -ext extendedKeyUsage
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `x509` | Subcommand for displaying and processing X.509 certificates |
| `-in zertifikat.pem` | Certificate file in PEM format |
| `-noout` | Do not output the certificate itself again |
| `-subject` | Display subject (CN and other fields) |
| `-enddate` | Display expiration date |
| `-ext extendedKeyUsage` | Display only the extension containing the purposes |

</details>

On Windows, double-clicking the file shows the same information: the **Details** tab, **Enhanced Key Usage** field.

What this means for outbound connections depends on the peer:

| The peer validates | Result with a certificate without Client Authentication |
|---|---|
| Name in the certificate (e.g., Exchange Online through `TlsSenderCertificateName`) | Whether Exchange Online validates the purpose is not documented; test before changing the certificate |
| Purpose of the client certificate | Rejection even if the name and CA are correct |
| Fingerprint of the client certificate | Rejection at every certificate change until the new fingerprint is configured |

Public certificates also have a shorter validity period. Under CA/Browser Forum decision SC-081v3, it is gradually decreasing from 398 days to 47 days by March 2029; the first stage of 200 days has applied since March 2026. Every certificate change is a point at which a peer may not recognize the new certificate.

For connections where partners identify Proofpoint using a client certificate, I therefore recommend:

1.  **Check existing certificates:** Do the current gateway certificates still contain the client purpose, and when do they expire?

2.  **Plan a dedicated client certificate** that permanently contains the client purpose, for example from a private PKI or a CA offering for server-to-server connections. Partners must know it.

3.  **Ask partners what they validate:** Name, purpose, issuer, or fingerprint. This determines whether a certificate change is a change for them.

4.  **Handle certificate changes as a change**, with a list of all peers that need to know the certificate and a test for each peer.

## Sources

1.  [Patch 5806 - TLS client and server certificate authentication improvement - Fix for 8.22.4 (Proofpoint Knowledge Base, sign-in required)](https://proofpoint.my.site.com/community/s/article/Patch-5806---TLS-client-and-server-certificate-authentication-improvement---Fix-for-8-22-4): Article 000027579 dated October 29, 2026; changed files and restart of the Admin GUI.

2.  [Patch 5807 - TLS client and server certificate authentication improvement - Fix for 8.22.4 (Proofpoint Knowledge Base, sign-in required)](https://proofpoint.my.site.com/community/s/article/Patch-5807---TLS-client-and-server-certificate-authentication-improvement---Fix-for-8-22-4): Article 000027580 dated October 29, 2026; configuration-only change dependent on 5806.

3.  [Proofpoint Enterprise Protection 8.23.X Release Notes (Proofpoint Knowledge Base, sign-in required)](https://proofpoint.my.site.com/community/s/article/Email-Protection-PPS-PoD-Proofpoint-Enterprise-Protection-8-23-X-Release-Notes): Article 000027117, section “SMTP Client Certificate” on page 5 of revision A.

4.  [Set-InboundConnector (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/exchange/set-inboundconnector): Parameters `RestrictDomainsToCertificate` and `TlsSenderCertificateName` for partner connectors in Exchange Online.

2.  [Chrome Root Program Policy](https://googlechrome.github.io/chromerootprogram/crp/policy): Section 1.3.2 on dedicated TLS server hierarchies; certificates issued from March 15, 2027 onward only with `serverAuth`.

3.  [DigiCert: Removing the client authentication EKU from public TLS certificates](https://knowledge.digicert.com/alerts/sunsetting-client-authentication-eku-from-digicert-public-tls-certificates): Default without Client Authentication since October 1, 2025, available upon request until March 1, 2027, and alternatives for server-to-server connections.

4.  [GeoCerts Change Log](https://www.geocerts.com/docs/change-log/): Overview of postponed dates for Client Authentication and validity-period stages at DigiCert and Sectigo.

5.  [CA/Browser Forum: Ballot SC081v3](https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/): Decision on the gradual reduction of public TLS certificate validity periods through March 2029.
