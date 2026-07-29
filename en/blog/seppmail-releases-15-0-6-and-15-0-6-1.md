---
title: "SEPPmail 15.0.6 and 15.0.6.1: Security Fixes and New Admin Features"
navTitle: "SEPPmail 15.0.6"
description: "In July 2026 SEPPmail released patch 15.0.6 and hotfix 15.0.6.1. Besides fixing vulnerabilities in PDF generation and PGP processing, the releases add a separate MFA field, LDAP authentication for the admin GUI, and corrections to the RuleEngine, webmail, and REST API."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "5 min to read"
themen:
  - "seppmail"
slug: "seppmail-releases-15-0-6-and-15-0-6-1"
translationOf: "seppmail-releases-15-0-6-und-15-0-6-1"
url: "https://rafaelpfister.ch/en/blog/seppmail-releases-15-0-6-and-15-0-6-1"
draft: false
---
# SEPPmail 15.0.6 and 15.0.6.1: Security Fixes and New Admin Features

On July 21, 2026, SEPPmail released patch 15.0.6, followed one day later by hotfix 15.0.6.1. The patch release closes several vulnerabilities, updates OpenSSH and OpenSSL, and brings noticeable improvements for administration. The hotfix corrects two RuleEngine issues that were introduced or surfaced with 15.0.6. The changes also affect appliances operated as HIN Mailgateways, since those run on the same SEPPmail firmware.

## Hotfix 15.0.6.1 from July 22, 2026

The hotfix addresses two points in the RuleEngine. First, an undefined value in the message object prevented log entries from being written to the mail log. Affected messages passed through the system without being logged. Second, the RuleEngine now detects the direction of archived emails so their delivery is handled correctly.

If you have already installed 15.0.6 or are planning the update, go straight to 15.0.6.1.

## Security Fixes in 15.0.6

The most important part of the patch release consists of three corrections to the security architecture:

- A possible path traversal vulnerability in PDF generation was closed. It was found by InfoGuard.
- All PGP-decrypted content is now Base64-encoded to prevent MIME structure injection.
- The hashencrypt function was refactored to use AES-256-CBC with PBKDF2.

On top of that come updated libraries: OpenSSH 10.4 and OpenSSL 3.0.21 together fix more than twenty CVEs. These points alone make the update advisable for production systems.

## New Administration Features

Three changes in the admin GUI stand out in daily use:

- **Separate MFA input field:** The second factor no longer needs to be appended to the password but has its own field. This removes a long-standing stumbling block at login.
- **LDAP authentication for the admin GUI:** Administrators can now authenticate against an external LDAP server instead of maintaining local accounts on the appliance. The setup is described in the article on [connecting the admin GUI to Active Directory](/en/blog/seppmail-admin-gui-ldap-authentication).
- **AutoRenew button for MPKI:** In the MPKI connector settings, automatic certificate renewal can be triggered manually via "Trigger AutoRenew...".

In addition, the appliance now consistently uses valid time zones (default: Europe/Zurich), and the System Object ID under System >> Advanced View is validated as a proper OID.

## Mail Processing and Webmail

Four points were corrected in the RuleEngine. Subject handling now works even when the encoding is unknown. Messages are bounced when signing is explicitly requested but cannot be performed; previously such messages could continue unsigned. Archive copies now pass through the delivery function and thus receive ARC headers. And for PGP messages without MDC data, MDC errors are ignored instead of disrupting processing.

Four bugs were fixed in the webmail (GINA): automatic deletion of unregistered accounts after the configured grace period works again, the hashdecrypt function returned a false-positive decryption result in certain cases, adding an attachment cleared the To and CC fields, and the time output in the SMS logs was incorrect.

## REST API, Cluster, and Backup

The REST API receives fixes to several endpoints: /system/ifaliasconfig (handling of null values), /system/applySysconfig (access configuration), /crypto/domain/{domainName} (domain certificate uploads), and GET and POST /ssl/csr. The timeout for REST calls was increased from 300 to 900 seconds, making long-running requests such as larger configuration changes more reliable.

In cluster operation, an existing CARP IP previously blocked the IP settings of a newly joined member; this is fixed. Password rehashing is also suppressed when cluster members run different firmware versions; more on that below. Before daily snapshot creation, the backup now additionally checks for a corrupt database before the snapshot is written.

## Connection to the Login Outage Under 15.0.5

When updating a cluster to 15.0.5, login could fail on both nodes. The fault symptoms and the recovery procedure are described in the article on the [login outage after the 15.0.5 update](/en/blog/hin-mailgateway-update-15-0-5-login-issue). The vendor was already aware of the problem at the time and announced a fix for a later version.

The release notes for 15.0.6 now contain exactly one entry that matches this fault pattern: "prevent password rehashing when cluster members use different firmware versions". During a cluster update, the nodes inevitably run different firmware versions for a while. If a node recomputes password hashes in this phase and replicates them into the cluster, the hashes no longer match on the other version, and login fails on both nodes, exactly as observed in the outage back then. The release notes do not mention the login outage explicitly, but the entry covers precisely the constellation that triggered it. The root cause is therefore addressed in 15.0.6; the emergency procedure with cluster dissolution that was necessary under 15.0.5 should no longer be needed for future updates.

## Minor Corrections

In the mail log, the date sorting that previously sorted alphabetically instead of chronologically was corrected, and the displayed size of LFT messages is accurate again. Access to missing X-headers is no longer logged. The MPKI CertCentral connector handles input and REST errors more robustly.

## Assessment

The two RuleEngine bugs addressed by the hotfix argue for skipping 15.0.6 and deploying 15.0.6.1 directly. For clusters, create snapshots of both nodes before the update and follow the update order from the vendor documentation. The login outage under 15.0.5 has shown why this preparation is not a formality.

## Sources

1.  [SEPPmail-Dokumentation – «Revision History»](https://docs.seppmail.com/ch/20_revision-history.html): Official release notes for 15.0.6 and 15.0.6.1 with all individual items.

2.  [HIN Mailgateway 15.0.5: Fixing the Login Outage After a Cluster Update](/en/blog/hin-mailgateway-update-15-0-5-login-issue): Why snapshots and the correct update order in a cluster are essential.
