---
title: "Proofpoint Protection Server 8.22.4: Patch 5807 leaves SMTP Client without a certificate"
navTitle: "PPS 8.22.4 certificate"
description: "Patches 5806 and 5807 introduce a separate “SMTP Client” assignment under System > Certificates > Services in Proofpoint Protection Server 8.22.4. If it is set to “-none-”, Sendmail delivers without a TLS client certificate. Peers that validate certificates then reject messages, such as Exchange Online with 5.7.51 or a smarthost with “Relay access denied”. Here is how to check and fix it."
date: "2026-11-05"
kategorie: "Proofpoint"
timeToRead: "7 min read"
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
slug: "proofpoint-protection-server-8-22-4-patch-kan-tilbakestille-sertifikatkonfigurasjonen-for-smtp"
translationId: "article-6f99807d289cc394"
warnung: true
featured: "2026-11-05"
translationOf: proofpoint-smtp-client-zertifikat
translationSourceHash: fcb36b2ae8af7420156aca54a99d61f4702aeb47781e1081e27edf7419e77db8
translationModel: gpt-5.6-terra
translatedAt: 2026-11-07T03:21:56.397Z
translationReview: required
aktualisiert: 2026-11-06
url: https://rafaelpfister.ch/no/blog/proofpoint-protection-server-8-22-4-patch-kan-tilbakestille-sertifikatkonfigurasjonen-for-smtp
---

I investigated an outage in the outbound mail flow of a Proofpoint Protection Server (PPS) 8.22.4 today. In short: after an automatically installed patch, the **SMTP Client** assignment under **System > Certificates > Services** was set to `-none-`. Sendmail subsequently established outbound connections without a TLS client certificate. All peers that identify Proofpoint through this certificate rejected the emails. Regular recipients using opportunistic TLS continued to accept mail, which is why the problem only became apparent when individual partners could no longer be reached.

If you operate PPS with automatic patches and have partners that require a client certificate, check this setting after every patch.

## Symptom

Only connections where the peer validates Proofpoint's client certificate are affected. Two typical cases:

- **Exchange Online with a partner connector** restricted to a certificate via `RestrictDomainsToCertificate`. The rejection reads `550 5.7.51 TenantInboundAttribution; Rejecting. Recipient has a partner connector with RestrictDomainsToIPAddresses or RestrictDomainsToCertificate set.`
- **Smarthost that permits relay only with a client certificate.** The response reads `554 5.7.1 <empfaenger@example.com>: Relay access denied`.

All other destinations continue to receive their mail. The test matrix therefore shows an inconsistent picture: mail to external recipients goes through, while mail to individual partners or through a particular smarthost fails.

The agent's mail log contains one of the following two messages for every outbound connection. If the file is missing altogether:

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

Immediately afterwards, Sendmail still establishes the connection using TLS (`verify=OK`), but without a client certificate. To the peer, this looks like an unknown sender.

## Cause

PPS assigns a certificate to each service on each server. Under **System > Certificates > Services**, each server lists, among other things, `SMTP Server`, `SMTP Client`, `Administration Server` and `End User Web Application / Encryption`. PPS generates the file `/etc/pki/tls/certs/sendmail-client.pem` from the assignment at `SMTP Client`, which Sendmail uses as its client certificate. If it is set to `-none-`, PPS writes the file empty or does not write it at all.

The `SMTP Client` assignment has only existed in 8.22.4 since patches 5806 and 5807, which Proofpoint released on 29 October 2026. Both are titled “TLS client and server certificate authentication improvement” and, according to the Knowledge Base, fix the inability of PPS 8.22.4 to assign separate certificates for the SMTP server and SMTP client. The feature is included as standard in release 8.23.0 (Release Notes, section “SMTP Client Certificate”).

| Patch | Content according to the Knowledge Base |
|---|---|
| 5806 | Updates, among other things, `SysCertificates.pm`, `Certificate.pm`, `TLS.pm`, `pp-sendmail.pl` and the template `sendmail.mc`, restarts the Admin GUI (`admind`), and updates the RPM `pps-pfpt` |
| 5807 | Requires 5806, changes no files and restarts no processes; it only applies the configuration changes required for separate server and client certificates |

Proofpoint therefore introduced the assignment rather than resetting an existing one. The patch does not carry over a certificate: after installation, the new line is set to `-none-`, and Sendmail no longer uses a client certificate from that point on. The patch descriptions contain no indication of this.

In the environment investigated, the assignment was subsequently set to `-none-` on all servers. Under **System > Config History**, patch 5807 was visible as an entry `configuration changes for patch-…` created by `system`, while the files under `/etc/pki/tls/certs/` had the same timestamp. In contrast, 5807 did not appear under **Software Patch History**. This is consistent with 5807 not installing software, but only changing configuration. **Config History** therefore shows the time of the change, while the actual software change belongs to patch 5806.

Rolling back under **Config History** to the patch snapshot did not help. PPS rewrote the file afterwards, but with 0 bytes.

## Check

In the Admin GUI on the master, go to **System > Certificates > Services**, then select each agent that delivers externally in turn at the top under **Server**, and inspect the `SMTP Client` line.

On the agent itself, two commands show the status:

```bash
ls -l --time-style=full-iso /etc/pki/tls/certs/
grep -E 'sendmail-client.pem|SSL_CTX_' /var/log/maillog | tail -5
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-l` | Long format with permissions, owner, size and date |
| `--time-style=full-iso` | Timestamp with date, time and time zone for comparison with the patch time |
| `grep -E` | Extended regular expressions, allowing both search terms to be combined with `\|` |
| `tail -5` | Show only the last five matches |

</details>

If `sendmail-client.pem` is missing or the file has 0 bytes while `sendmail.pem` is several thousand bytes in size, no certificate is assigned to the `SMTP Client` service.

## Fix

Under **System > Certificates > Services**, select a valid certificate for `SMTP Client` on every affected agent and save it using **Save Changes**. In most cases, this is the same public certificate shown for `SMTP Server`.

The selection list may contain multiple certificates with the same name, including expired ones. They can only be distinguished by the serial number in parentheses. Before saving, compare the serial number with the list under **System > Certificates > Certificates**, where the issue and expiry dates are shown.

The list under `SMTP Client` does not necessarily show all imported certificates. In the environment investigated, some certificates were missing although they were valid and expired copies had already been deleted. Neither the patch articles nor the 8.23 Release Notes describe the criteria PPS uses to filter the list. If a certificate is missing, check two points:

- **Private key:** If the certificate was imported as PEM without a key, PPS cannot assign it to a service. PKCS12 files always contain the key.
- **Intended use:** Does the certificate contain the purpose `TLS Web Client Authentication` in `Extended Key Usage`? Many public certificates issued since autumn 2025 no longer contain it (see below). A certificate without this purpose is unsuitable for the `SMTP Client` service anyway.

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
| `tail -20` | Show only the last twenty lines |

</details>

The messages `SSL_CTX_… failed` and `unsafe` must no longer appear after saving, and deliveries to partners must end with `stat=Sent`. Sendmail delivers mail previously deferred with a 4xx code from the queue itself. Senders must resend mail rejected with a 5xx code.

Manually copying `sendmail.pem` to `sendmail-client.pem` also bridges the outage. However, PPS overwrites the file again at the next configuration push as long as the assignment is missing in the GUI.

## Check after every patch

Patch 5807 changed the configuration without the patch description indicating the consequences. I therefore recommend checking the **Services** page after every patch and looking in **Config History** to see whether the patch included configuration changes. If patch 5807 has not yet been installed on an 8.22.4 system, assign a certificate to `SMTP Client` immediately after installation. Even easier is monitoring that alerts on the message `sendmail-client.pem` in the agents' mail log. Under **Licenses and Updates > Settings**, you can also specify when patches are installed. A maintenance window during which someone can check the mail flow significantly shortens such an outage.

## Client Authentication is disappearing from public certificates

Independently of the patch, a second change affects precisely these connections. Until now, public TLS certificates have generally contained both purposes in the `Extended Key Usage` extension: `TLS Web Server Authentication` and `TLS Web Client Authentication`. Mail gateways therefore also use their server certificate as a client certificate.

The Chrome Root Program requires public certificates issued from 15 March 2027 onwards to contain only `serverAuth`. DigiCert has issued public TLS certificates without Client Authentication by default since 1 October 2025; the purpose is still available on request until 1 March 2027. Other CAs have comparable schedules. OpenSSL shows whether a certificate contains the client purpose:

```bash
openssl x509 -in zertifikat.pem -noout \
  -subject -enddate -ext extendedKeyUsage
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `x509` | Subcommand for viewing and editing X.509 certificates |
| `-in zertifikat.pem` | Certificate file in PEM format |
| `-noout` | Do not output the certificate itself again |
| `-subject` | Show subject (CN and additional fields) |
| `-enddate` | Show expiry date |
| `-ext extendedKeyUsage` | Show only the extension containing the intended uses |

</details>

On Windows, double-clicking the file shows the same information: the **Details** tab, **Enhanced Key Usage** field.

What this means for outbound connections depends on the peer:

| Peer validates | Consequence with a certificate without Client Authentication |
|---|---|
| Name in the certificate (e.g. Exchange Online via `TlsSenderCertificateName`) | Whether Exchange Online validates the intended use is not documented; test before changing the certificate |
| Intended use of the client certificate | Rejection even if the name and CA match |
| Fingerprint of the client certificate | Rejection at every certificate change until the new fingerprint has been configured |

There is also the shorter lifetime of public certificates. Following CA/Browser Forum resolution SC-081v3, it is gradually reduced from 398 days to 47 days by March 2029; the first stage of 200 days has applied since March 2026. Every certificate change is a point at which a peer may not know the new certificate.

For connections where partners identify Proofpoint through a client certificate, I therefore recommend:

1.  **Check existing certificates:** Do current gateway certificates still contain the client purpose, and when do they expire?

2.  **Plan a dedicated client certificate** that permanently contains the client purpose, for example from a private PKI or a CA offering for server-to-server connections. Partners must know it.

3.  **Ask partners what they validate:** Name, intended use, issuer or fingerprint. This determines whether a certificate change constitutes a change for them.

4.  **Handle certificate changes as a change**, with a list of all peers that must know the certificate and a test for each peer.

## Kilder

1.  [Patch 5806 - TLS client and server certificate authentication improvement - Fix for 8.22.4 (Proofpoint Knowledge Base, login required)](https://proofpoint.my.site.com/community/s/article/Patch-5806---TLS-client-and-server-certificate-authentication-improvement---Fix-for-8-22-4): Article 000027579 of 29 October 2026, modified files and Admin GUI restart.

2.  [Patch 5807 - TLS client and server certificate authentication improvement - Fix for 8.22.4 (Proofpoint Knowledge Base, login required)](https://proofpoint.my.site.com/community/s/article/Patch-5807---TLS-client-and-server-certificate-authentication-improvement---Fix-for-8-22-4): Article 000027580 of 29 October 2026, configuration-only change dependent on 5806.

3.  [Proofpoint Enterprise Protection 8.23.X Release Notes (Proofpoint Knowledge Base, login required)](https://proofpoint.my.site.com/community/s/article/Email-Protection-PPS-PoD-Proofpoint-Enterprise-Protection-8-23-X-Release-Notes): Article 000027117, section “SMTP Client Certificate” on page 5 of revision A.

4.  [Set-InboundConnector (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/exchange/set-inboundconnector): Parameters `RestrictDomainsToCertificate` and `TlsSenderCertificateName` for partner connectors in Exchange Online.

2.  [Chrome Root Program Policy](https://googlechrome.github.io/chromerootprogram/crp/policy): Section 1.3.2 on dedicated TLS server hierarchies; certificates from 15 March 2027 onwards only with `serverAuth`.

3.  [DigiCert: Removing the client authentication EKU from public TLS certificates](https://knowledge.digicert.com/alerts/sunsetting-client-authentication-eku-from-digicert-public-tls-certificates): Default without Client Authentication since 1 October 2025, available on request until 1 March 2027, alternatives for server-to-server connections.

4.  [GeoCerts Change Log](https://www.geocerts.com/docs/change-log/): Overview of postponed dates for Client Authentication and lifetime stages at DigiCert and Sectigo.

5.  [CA/Browser Forum: Ballot SC081v3](https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/): Resolution on the gradual reduction of public TLS certificate lifetimes until March 2029.
