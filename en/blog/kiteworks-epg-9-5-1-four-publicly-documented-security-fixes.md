---
title: "Kiteworks EPG 9.5.1: Four publicly documented security fixes"
navTitle: "Kiteworks EPG 9.5.1"
description: "Kiteworks Email Protection Gateway 9.5.1 fixes four publicly documented vulnerabilities, including an account takeover issue with a CVSS score of 9.4. Installations prior to 9.5.1 should be updated to this or a later version."
date: "2026-11-06"
kategorie: "Totemomail"
timeToRead: "3 min read"
themen:
  - totemomail
produkte:
  - "totemomail"
protokolle:
  - "releases"
slug: "kiteworks-epg-9-5-1-four-publicly-documented-security-fixes"
translationId: article-d4b9d984c72a54ea
translationOf: kiteworks-epg-releases-9-5-1
url: https://rafaelpfister.ch/en/blog/kiteworks-epg-9-5-1-four-publicly-documented-security-fixes
translationSourceHash: 7d961f479006e9eb0da1a74925d77e25e7205a64a1ed783dccb55b6eed8c2c5c
translationModel: gpt-5.6-terra
translatedAt: 2026-11-07T03:18:55.163Z
translationReview: required
---

# Kiteworks EPG 9.5.1: Four publicly documented security fixes

Kiteworks continues the former totemo technology as Email Protection Gateway (EPG). Version 9.5.1 was the current Kiteworks version no later than September 25, 2026; the vendor does not state a public release date. The vendor advisories published on September 30 document four fixes for EPG 9.5.1: an account takeover, two unauthenticated flaws for writing files and triggering a denial of service, respectively, and a flaw that allowed administrators to write arbitrary files. The classification for the identically named totemomail version is available in the article on [totemomail 9.5.1](/blog/totemomail-9-5-1-sicherheitsupdate).

## Versions at a glance

| Version | Date | Type | Key points |
|---|---|---|---|
| 9.5.1 | publicly confirmed no later than September 25, 2026 | security update | Four publicly documented EPG fixes, including CVE-2026-102149 with CVSS 9.4 |

## Security fixes in 9.5.1

According to Kiteworks, all four advisories affect all EPG versions before 9.5.1 and list 9.5.1 as the fixed version. The advisories were published on September 30, 2026. They retrospectively describe the security status of the release, rather than its exact release date.

- **CVE-2026-102149, CVSS 9.4 (critical):** A weakness in access control could allow an attacker to gain access to user accounts over the network. The CVSS rating requires neither privileges nor user interaction. Kiteworks classifies the cause as CWE-306 (missing authentication for critical functions) and CWE-639 (authorization bypass through a user-controlled key).
- **CVE-2026-102143, CVSS 7.5 (high):** An unauthenticated attacker could write files to the appliance. Kiteworks cites an authentication issue as well as CWE-306 and CWE-434 (unrestricted upload of dangerous file types).
- **CVE-2026-102146, CVSS 6.5 (medium):** An administrator could write arbitrary files. Kiteworks describes a privilege issue; the CVSS metric requires high privileges. The vendor lists CWE-73 (external control of file name or path) and CWE-1336 (insufficient neutralization in a template engine) as weaknesses.
- **CVE-2026-102144, CVSS 5.3 (medium):** An attacker on the network could trigger a denial of service. The advisory cites missing authentication for critical functions and uncontrolled resource consumption (CWE-306, CWE-400). This rating also requires neither privileges nor user interaction.

For operators, the account takeover is the priority fix because, according to the advisory, it is possible without an existing account. The other two flaws that do not require login also affect all versions before 9.5.1. The vendor advisories do not disclose technical details about endpoints, configurations, or workarounds.

## Recommended approach

Upgrade EPG to at least 9.5.1 if an older version is in use. Kiteworks explicitly identifies upgrading to 9.5.1 or later as the remedy for all four advisories. If a later vendor version has already been approved, it also meets this minimum version requirement. Kiteworks has not published a public update path, known issues, or support timelines for EPG 9.5.1 in the sources reviewed.

## Sources

1.  [Kiteworks: Precautionary Shutdown Advisory for Customers](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): Vendor notice dated September 25, 2026, naming 9.5.1 as the current version.

2.  [Kiteworks Security Advisories](https://github.com/kiteworks/security-advisories/security/advisories): Official overview of the advisories published on September 30, 2026, including the EPG entries for 9.5.1.

3.  [GHSA-c9w5-4frw-7wqq: Account takeover (CVE-2026-102149)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-c9w5-4frw-7wqq): Critical access-control weakness, CVSS 9.4, fixed in EPG 9.5.1.

4.  [GHSA-3p9g-jh62-8f89: Unauthorized file modification (CVE-2026-102143)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-3p9g-jh62-8f89): Unauthenticated file writing, CVSS 7.5, fixed in EPG 9.5.1.

5.  [GHSA-5pgq-v8g2-rg2f: Unauthorized file modification (CVE-2026-102146)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-5pgq-v8g2-rg2f): Writing arbitrary files with administrator privileges, CVSS 6.5, fixed in EPG 9.5.1.

6.  [GHSA-wwhf-5862-rjxq: Denial of service (CVE-2026-102144)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-wwhf-5862-rjxq): Unauthenticated denial of service, CVSS 5.3, fixed in EPG 9.5.1.

7.  [Kiteworks: EPG with totemo Integration](https://www.kiteworks.com/video-why-kiteworks-and-totemo-are-better-together/): Vendor statement on the continuation and renaming of the totemo technology as Email Protection Gateway.
