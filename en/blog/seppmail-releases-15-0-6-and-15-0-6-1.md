---
title: "SEPPmail 15.0.6 and 15.0.6.1: Security Fixes and New Admin Features"
navTitle: "SEPPmail 15.0.6"
description: "In July 2026, SEPPmail released patch release 15.0.6 and hotfix 15.0.6.1. In addition to fixing vulnerabilities in PDF generation and PGP processing, the releases introduce a separate MFA field, LDAP authentication for the Admin GUI, and fixes for RuleEngine, webmail, and the REST API."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "5 min read"
themen:
  - seppmail
slug: "seppmail-releases-15-0-6-and-15-0-6-1"
translationOf: "seppmail-releases-15-0-6-und-15-0-6-1"
draft: false
translationId: article-3046fc35b259929b
translationSourceHash: 636a7246234584a2b5797f53239fe65129de0f4463b8f773d0a7d9ed06d61f91
translatedAt: 2026-09-03T08:14:31.540Z
translationReview: automatic
translationModel: gpt-5.6-terra
url: https://rafaelpfister.ch/en/blog/seppmail-releases-15-0-6-and-15-0-6-1
---

# SEPPmail 15.0.6 and 15.0.6.1: Security Fixes and New Admin Features

On July 21, 2026, SEPPmail released patch release 15.0.6, followed one day later by hotfix 15.0.6.1. The patch release closes several vulnerabilities, updates OpenSSH and OpenSSL, and delivers noticeable administration improvements. The hotfix corrects two RuleEngine errors that were introduced or became visible with 15.0.6. The changes also affect appliances operated as HIN Mailgateway systems, as they are based on the same SEPPmail firmware.

## Hotfix 15.0.6.1 of July 22, 2026

The hotfix fixes two RuleEngine issues. First, an undefined value in the Message object prevented log entries from being written to the mail log. Affected messages therefore passed through the system without being logged. Second, RuleEngine now recognizes the direction of archived emails so that their delivery is handled correctly.

Anyone who has already installed 15.0.6 or plans to update should go directly to 15.0.6.1.

The HIN appliances apparently received the hotfix as well: A HIN Mailgateway with version 15.0.6-RC-42-g278c81f84 installed now reports 15.0.6-RC-88-g916e513cc as the next version in the 15.0 branch. The RC designations of the HIN firmware cannot be mapped directly to a SEPPmail release, but the timing of the offer points to the hotfix.

## Security Fixes in 15.0.6

The most important part of the patch release consists of three fixes to the security architecture:

- A potential path traversal vulnerability in PDF generation was closed. It was discovered by InfoGuard.
- All content decrypted via PGP is now Base64-encoded to prevent MIME structure injection.
- The hashencrypt function was switched to AES-256-CBC with PBKDF2.

Updated libraries are also included: OpenSSH 10.4 and OpenSSL 3.0.21 fix more than twenty CVEs combined. These items alone make the update advisable for production systems.

## New Features for Administration

Three changes in the Admin GUI stand out in day-to-day use:

- **Separate MFA input field:** The second factor no longer needs to be appended to the password, but has its own field. This eliminates a long-standing source of login errors.
- **LDAP authentication for the Admin GUI:** Administrators can now authenticate against an external LDAP server instead of maintaining local accounts on the appliance. Setup is described in the article on [connecting the Admin GUI to Active Directory](/blog/seppmail-admin-gui-ldap-authentifizierung). Whether the HIN Mailgateway has also received this feature is still being tested and will be added to the article afterward; since HIN uses the same firmware base, I assume it has.
- **AutoRenew button for MPKI:** In the MPKI connector settings, automatic certificate renewal can be triggered manually using “Trigger AutoRenew...”.

In addition, the appliance now consistently uses valid time zones (default: Europe/Zurich), and the System Object ID under System >> Advanced View is validated as a valid OID.

## Mail Processing and Webmail

Four items were corrected in RuleEngine. Subject handling now also works with unknown encoding. Messages are bounced if a signature is explicitly requested but cannot be created; previously, such messages could continue unsigned. Archive copies now pass through the delivery function and therefore receive ARC headers. And for PGP messages without MDC data, MDC errors are ignored instead of disrupting processing.

Four errors were fixed in webmail (GINA): automatic deletion of unregistered accounts after the grace period has expired works again, the hashdecrypt function returned a false-positive decryption result in certain cases, adding an attachment cleared the To and CC fields, and the time output in the SMS logs was incorrect.

## REST API, Cluster, and Backup

The REST API receives fixes to several endpoints: /system/ifaliasconfig (handling null values), /system/applySysconfig (access configuration), /crypto/domain/{domainName} (uploading domain certificates), and GET and POST /ssl/csr. The timeout for REST calls was increased from 300 to 900 seconds, making long-running requests such as larger configuration changes more reliable.

In cluster operation, an existing CARP IP previously blocked the IP settings of a newly added member; this has been fixed. Before creating the daily snapshot, the backup now also checks for a corrupt database before writing the snapshot.

## Relation to the Login Outage in 15.0.5

When updating a cluster to 15.0.5, login could fail on both nodes. The symptoms and recovery are described in the article on [login outage after the 15.0.5 update](/blog/hin-update-issue-version-15.0.5). The manufacturer was already aware of the problem at the time and announced a fix for a subsequent version.

The 15.0.6 release notes now contain exactly one entry that matches this issue: “prevent password rehashing when cluster members use different firmware versions.” During a cluster update, the nodes inevitably run temporarily on different firmware versions. If one node recalculates password hashes during this phase and replicates them to the cluster, the hashes no longer match on the other version, and login fails on both nodes, exactly as in the outage observed at the time. The release notes do not explicitly mention the login outage, but the entry precisely covers the configuration that triggered it. The cause is therefore addressed in 15.0.6; the emergency procedure required in 15.0.5 involving cluster dissolution should no longer be necessary for future updates.

## Minor Fixes

In the mail log, date sorting was corrected, which had previously sorted alphabetically instead of chronologically, and the displayed size of LFT messages is correct again. Access to nonexistent X-headers is no longer logged. The CertCentral connector for MPKI handles input and REST errors more robustly.

## Assessment

The two RuleEngine errors addressed by the hotfix suggest skipping 15.0.6 and deploying 15.0.6.1 directly. For clusters, create snapshots of both nodes before updating and follow the update order in the manufacturer documentation. The login outage in 15.0.5 showed why this preparation is not merely a formality.

## Sources

1.  [SEPPmail documentation – “Revision History”](https://docs.seppmail.com/ch/20_revision-history.html): Official release notes for 15.0.6 and 15.0.6.1 with all individual items.

2.  [HIN Mailgateway 15.0.5: Fixing the login outage after the cluster update](/blog/hin-update-issue-version-15.0.5): Why snapshots and the correct update order in the cluster are crucial.
