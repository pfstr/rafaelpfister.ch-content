---
title: "SEPPmail 15.0.6 and 15.0.6.1: security fixes and new admin features"
navTitle: "SEPPmail 15.0.6"
description: "SEPPmail released patch release 15.0.6 and hotfix 15.0.6.1 in July 2026. Alongside fixed vulnerabilities in PDF generation and PGP processing, the releases introduce a separate MFA field, LDAP authentication for the admin GUI and fixes to the RuleEngine, webmail and REST API."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "5 min read"
themen:
  - seppmail
slug: "seppmail-releases-15-0-6-and-15-0-6-1"
translationOf: "seppmail-releases-15-0-6-und-15-0-6-1"
draft: false
translationId: article-3046fc35b259929b
translationSourceHash: 5cf19b84bb90403b0a7e2795222b8f853c29c3fe562429df8538e703e565217a
translatedAt: 2026-07-30T12:49:03.863Z
translationReview: automatic
url: https://rafaelpfister.ch/en/blog/seppmail-releases-15-0-6-and-15-0-6-1
translationModel: gpt-5.6-terra
---

# SEPPmail 15.0.6 and 15.0.6.1: security fixes and new admin features

SEPPmail released patch release 15.0.6 on 21 July 2026, followed by hotfix 15.0.6.1 a day later. The patch release closes several vulnerabilities, updates OpenSSH and OpenSSL, and brings noticeable improvements for administration. The hotfix fixes two errors in the RuleEngine that were introduced or became visible with 15.0.6. The changes also affect appliances operated as HIN Mailgateway, as these are based on the same SEPPmail firmware.

## Hotfix 15.0.6.1 of 22 July 2026

The hotfix resolves two issues in the RuleEngine. Firstly, an undefined value in the message object prevented log entries from being written to the mail log. Affected messages therefore passed through the system without being logged. Secondly, the RuleEngine now identifies the direction of archived emails so that their delivery is handled correctly.

Anyone who has already installed 15.0.6 or is planning the update should upgrade directly to 15.0.6.1.

The HIN appliances also appear to have received the hotfix: a HIN Mailgateway with version 15.0.6-RC-42-g278c81f84 installed now reports 15.0.6-RC-88-g916e513cc as the next version in the 15.0 branch. The RC designations of the HIN firmware cannot be mapped directly to a SEPPmail release, but the timing of the offer suggests the hotfix.

## Security fixes in 15.0.6

The most important part of the patch release consists of three changes to the security architecture:

- A possible path traversal vulnerability in PDF generation has been closed. It was found by InfoGuard.
- All content decrypted using PGP is now Base64-encoded to prevent MIME structure injection.
- The hashencrypt function has been switched to AES-256-CBC with PBKDF2.

Updated libraries are also included: OpenSSH 10.4 and OpenSSL 3.0.21 fix more than twenty CVEs between them. These points alone make the update advisable for production systems.

## New features for administration

Three changes in the admin GUI stand out in day-to-day use:

- **Separate MFA input field:** The second factor no longer has to be appended to the password, but has its own field. This removes a long-standing login pitfall.
- **LDAP authentication for the admin GUI:** Administrators can now authenticate against an external LDAP server instead of maintaining local accounts on the appliance. Setup is described in the article on [connecting the admin GUI to Active Directory](/blog/seppmail-admin-gui-ldap-authentifizierung). I am still testing whether the HIN Mailgateway has also received this feature and will add this to the article afterwards; as HIN uses the same firmware base, I assume it has.
- **AutoRenew button for MPKI:** In the MPKI connector settings, automatic certificate renewal can be triggered manually using «Trigger AutoRenew... ».

In addition, the appliance now consistently uses valid time zones (default: Europe/Zurich), and the System Object ID under System >> Advanced View is validated as a valid OID.

## Mail processing and webmail

Four issues have been fixed in the RuleEngine. Subject handling now also works with unknown encoding. Messages are bounced when a signature is explicitly requested but cannot be created; previously, such messages could continue unsigned. Archive copies now pass through the delivery function and therefore receive ARC headers. In addition, MDC errors are ignored for PGP messages without MDC data rather than disrupting processing.

Four errors have been fixed in webmail (GINA): automatic deletion of unregistered accounts after the grace period has expired works again, the hashdecrypt function returned a false-positive decryption result in certain cases, adding an attachment cleared the To and CC fields, and the time output in the SMS logs was incorrect.

## REST API, cluster and backup

The REST API receives fixes to several endpoints: /system/ifaliasconfig (handling of null values), /system/applySysconfig (access configuration), /crypto/domain/{domainName} (uploading domain certificates), and GET and POST /ssl/csr. The timeout for REST calls has been increased from 300 to 900 seconds, making long-running requests such as larger configuration changes more reliable.

In cluster operation, an existing CARP IP previously blocked the IP settings of a newly added member; this has been fixed. Before creating the daily snapshot, the backup now also checks for a corrupt database before writing the snapshot.

## Relation to the login outage in 15.0.5

When updating a cluster to 15.0.5, login could fail on both nodes. The symptoms and recovery are described in the article on the [login outage after the 15.0.5 update](/blog/hin-update-issue-version-15.0.5). The manufacturer was already aware of the problem at the time and announced a fix for a subsequent version.

The release notes for 15.0.6 now contain exactly one entry that matches these symptoms: «prevent password rehashing when cluster members use different firmware versions». During a cluster update, the nodes inevitably run temporarily with different firmware versions. If one node recalculates password hashes at this stage and replicates them to the cluster, the hashes no longer match on the other version and login fails on both nodes, exactly as in the outage observed at the time. The release notes do not explicitly mention the login outage, but the entry precisely covers the configuration that caused it. The cause is therefore addressed in 15.0.6; the emergency procedure involving cluster dissolution that was necessary in 15.0.5 should no longer be required for future updates.

## Minor fixes

In the mail log, date sorting has been corrected, as it previously sorted alphabetically rather than chronologically, and the displayed size of LFT messages is correct again. Access to non-existent X-Headers is no longer logged. The CertCentral connector for MPKI handles input and REST errors more robustly.

## Assessment

The two RuleEngine errors addressed by the hotfix suggest skipping 15.0.6 and deploying 15.0.6.1 directly. For clusters, create snapshots of both nodes before updating and follow the update sequence in the manufacturer documentation. The login outage in 15.0.5 showed why this preparation is not a mere formality.

## Sources

1.  [SEPPmail documentation – «Revision History»](https://docs.seppmail.com/ch/20_revision-history.html): Official release notes for 15.0.6 and 15.0.6.1 with all individual items.

2.  [HIN Mailgateway 15.0.5: Fixing the login outage after the cluster update](/blog/hin-update-issue-version-15.0.5): Why snapshots and the correct update sequence in the cluster are crucial.
