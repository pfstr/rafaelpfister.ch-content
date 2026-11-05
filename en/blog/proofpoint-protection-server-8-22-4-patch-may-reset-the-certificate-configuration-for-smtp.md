---
title: "Proofpoint Protection Server 8.22.4: Patch may reset the certificate configuration for SMTP Client"
navTitle: "PPS 8.22.4 Certificate"
description: "If the “SMTP Client” assignment under System > Certificates > Services in Proofpoint Protection Server is set to “-none-”, Sendmail delivers without a TLS client certificate. Peers that validate certificates will then reject it, such as Exchange Online with 5.7.51 or a smarthost with “Relay access denied.” Here is how to check and fix it."
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
url: https://rafaelpfister.ch/en/blog/proofpoint-protection-server-8-22-4-patch-may-reset-the-certificate-configuration-for-smtp
translationSourceHash: a6ce9c6e7e8b62910100ae9ff2323179510735b20326f5b916838ea5df74de02
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:26:06.964Z
translationReview: required
---

Today, I investigated an outage in the outbound mail flow of a Proofpoint Protection Server (PPS) 8.22.4. In short: After an automatically installed patch, the **SMTP Client** assignment under **System > Certificates > Services** was set to `-none-`. Sendmail then established outbound connections without a TLS client certificate. All peers that identify Proofpoint through this certificate rejected the emails. Regular recipients using opportunistic TLS continued accepting mail, so the issue was only noticed when individual partners could no longer be reached.

If you operate PPS with automatic patches and have partners that require a client certificate, check this setting after every patch.

## Symptom

Only connections where the peer validates Proofpoint's client certificate are affected. Two typical cases:

- **Exchange Online with a partner connector** restricted to a certificate via `RestrictDomainsToCertificate`. The rejection is `550 5.7.51 TenantInboundAttribution; Rejecting. Recipient has a partner connector with RestrictDomainsToIPAddresses or RestrictDomainsToCertificate set.`
- **Smarthost that permits relaying only with a client certificate.** The response is `554 5.7.1 <empfaenger@example.com>: Relay access denied`.

All other destinations continue receiving their mail. The test matrix therefore shows an inconsistent picture: mail to external recipients gets through, while mail to individual partners or through a specific smarthost fails.

The agent's mail log contains one of the following two messages for every outbound connection. If the file is missing entirely:

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

PPS assigns a certificate to each service on each server. Under **System > Certificates > Services**, each server lists, among other things, `SMTP Server`, `SMTP Client`, `Administration Server` and `End User Web Application / Encryption`. From the assignment for `SMTP Client`, PPS generates the file `/etc/pki/tls/certs/sendmail-client.pem`, which Sendmail uses as its client certificate. If it is set to `-none-`, PPS writes the file as empty or does not write it at all.

In the environment investigated, the assignment was set to `-none-` on all servers after an automatic patch. Under **System > Config History**, the patch was visible as an entry `configuration changes for patch-…` created by `system`, and the files under `/etc/pki/tls/certs/` had the same timestamp. Whether Proofpoint introduced the assignment with this patch or whether the patch reset an existing assignment is not described in the publicly available release notes. I will update this article as soon as Proofpoint comments on it.

A rollback under **Config History** to the patch snapshot did not help. PPS rewrote the file afterward, but with 0 bytes.

## Check

In the Admin GUI on the master, go to **System > Certificates > Services**. At the top, select each agent that delivers outbound mail under **Server**, one at a time, and inspect the `SMTP Client` row.

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
| `--time-style=full-iso` | Timestamp with date, time, and time zone for comparison with the patch time |
| `grep -E` | Extended regular expressions so both search terms can be combined with `\|` |
| `tail -5` | Display only the last five matches |

</details>

If `sendmail-client.pem` is missing or the file is 0 bytes while `sendmail.pem` is several thousand bytes in size, no certificate is assigned to the `SMTP Client` service.

## Fix

Under **System > Certificates > Services**, select a valid certificate for `SMTP Client` on each affected agent and save it with **Save Changes**. This is usually the same public certificate assigned to `SMTP Server`.

The selection list displays all imported certificates, including expired ones with the same name. They can only be distinguished by the serial number in parentheses. Before saving, compare the serial number with the list under **System > Certificates > Certificates**, which shows the issue and expiration dates.

After saving, PPS distributes the configuration to the agents. Verify on the agent:

```bash
ls -l /etc/pki/tls/certs/sendmail-client.pem
grep -E 'STARTTLS=client|stat=' /var/log/maillog | tail -20
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `ls -l` | Shows the file size and permissions; after the correction, it is no longer empty |
| `grep -E 'STARTTLS=client\|stat='` | Filters TLS messages for outbound connections and the delivery result |
| `tail -20` | Display only the last twenty lines |

</details>

The messages `SSL_CTX_… failed` and `unsafe` must no longer appear after saving, and deliveries to partners must end with `stat=Sent`. Sendmail delivers mail that was previously deferred with a 4xx code from the queue on its own. Mail rejected with a 5xx code must be resent by the senders.

Manually copying `sendmail.pem` to `sendmail-client.pem` also bridges the outage. However, PPS overwrites the file again during the next configuration push as long as the assignment is missing in the GUI.

## Check after every patch

Until it is clear why the patch changes the assignment, I recommend checking the **Services** page after every patch. Monitoring that alerts on the message `sendmail-client.pem` in the agents' mail logs is easier. Under **Licenses and Updates > Settings**, you can also specify when patches are installed. A maintenance window in which someone can check the mail flow significantly reduces the duration of such an outage.

## Client Authentication disappears from public certificates

Independently of the patch, a second change affects precisely these connections. Until now, public TLS certificates have generally contained both purposes in the `Extended Key Usage` extension: `TLS Web Server Authentication` and `TLS Web Client Authentication`. Mail gateways therefore use their server certificate as a client certificate as well.

The Chrome Root Program requires public certificates issued from March 15, 2027 onward to contain only `serverAuth`. Since October 1, 2025, DigiCert has issued public TLS certificates without Client Authentication by default; the purpose remains available upon request until March 1, 2027. Other CAs have comparable schedules. OpenSSL shows whether a certificate contains the client purpose:

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
| `-subject` | Display the subject (CN and additional fields) |
| `-enddate` | Display the expiration date |
| `-ext extendedKeyUsage` | Display only the extension containing the purposes |

</details>

In Windows, double-clicking the file shows the same information: the **Details** tab, **Enhanced Key Usage** field.

What this means for outbound connections depends on the peer:

| What the peer validates | Result with a certificate without Client Authentication |
|---|---|
| Name in the certificate (for example, Exchange Online via `TlsSenderCertificateName`) | Whether Exchange Online validates the purpose is not documented; test before changing the certificate |
| Purpose of the client certificate | Rejection even if the name and CA match |
| Fingerprint of the client certificate | Rejection with every certificate change until the new fingerprint is configured |

In addition, public certificates have a shorter validity period. Under CA/Browser Forum decision SC-081v3, it will gradually decrease from 398 days to 47 days by March 2029; the first stage of 200 days has applied since March 2026. Every certificate change is a point at which a peer may not recognize the new certificate.

For connections where partners identify Proofpoint through a client certificate, I therefore recommend:

1.  **Check existing certificates:** Do current gateway certificates still include the client purpose, and when do they expire?

2.  **Plan a dedicated client certificate** that permanently includes the client purpose, such as one from a private PKI or a CA offering for server-to-server connections. Partners must know it.

3.  **Ask partners what they validate:** Name, purpose, issuer, or fingerprint. This determines whether a certificate change constitutes a change for them.

4.  **Manage certificate changes as changes**, with a list of all peers that must know the certificate and a test for each peer.

## Sources

1.  [Set-InboundConnector (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/exchange/set-inboundconnector): Parameters `RestrictDomainsToCertificate` and `TlsSenderCertificateName` for partner connectors in Exchange Online.

2.  [Chrome Root Program Policy](https://googlechrome.github.io/chromerootprogram/crp/policy): Section 1.3.2 on dedicated TLS server hierarchies; certificates issued from March 15, 2027 onward may contain only `serverAuth`.

3.  [DigiCert: Removing the client authentication EKU from public TLS certificates](https://knowledge.digicert.com/alerts/sunsetting-client-authentication-eku-from-digicert-public-tls-certificates): Default without Client Authentication since October 1, 2025, available upon request until March 1, 2027, and alternatives for server-to-server connections.

4.  [GeoCerts Change Log](https://www.geocerts.com/docs/change-log/): Overview of postponed dates for Client Authentication and validity stages at DigiCert and Sectigo.

5.  [CA/Browser Forum: Ballot SC081v3](https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/): Decision on the gradual reduction of public TLS certificate validity periods through March 2029.
