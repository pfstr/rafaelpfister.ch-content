---
title: "totemomail 9.5.1: Account takeover in the Web Portal and SSRF via incoming email fixed"
navTitle: "totemomail 9.5.1"
description: "totemomail 9.5.1 fixes eight security vulnerabilities and brings totemomail in line with the Kiteworks advisories for Email Protection Gateway published at the end of September. Two vulnerabilities can be exploited without authentication: a flaw in the Web Portal password reset process that allows administrator accounts to be taken over, and server-side retrieval of URLs from incoming messages. The release also includes path traversal fixes in the Admin API and Mail API, as well as updated Tomcat and Batik libraries."
date: "2026-10-31"
kategorie: "Totemomail"
timeToRead: "5 min read"
themen:
  - totemomail
  - sicherheitsluecken
produkte:
  - "totemomail"
  - "sicherheitsluecken"
protokolle:
  - "releases"
  - "haertung"
slug: "totemomail-9-5-1-account-takeover-in-the-web-portal-and-ssrf-via-incoming-email-fixed"
translationId: "article-2bd15f63632ab728"
draft: false
translationOf: totemomail-9-5-1-sicherheitsupdate
url: https://rafaelpfister.ch/en/blog/totemomail-9-5-1-account-takeover-in-the-web-portal-and-ssrf-via-incoming-email-fixed
translationSourceHash: 5d441bdc7ec39d7fcef423e826bf14772f11bca2aa3dbdf4e1e3c83297131635
translationModel: gpt-5.6-terra
translatedAt: 2026-10-31T19:46:02.667Z
translationReview: automatic
---

# totemomail 9.5.1: Account takeover in the Web Portal and SSRF via incoming email fixed

totemo has released totemomail 9.5.1 (Build 101). The patch contains eight security fixes, one improvement, and three bug fixes. Two vulnerabilities can be exploited without authentication: a flaw in the Web Portal password reset process that can also be used to take over administrator accounts, and server-side retrieval of URLs contained in incoming messages. Both affect components that are reachable from the internet in most installations. You should therefore schedule the update promptly.

## From 9.2.1 directly to 9.5.1

The release notes list 9.5.1 directly after 9.2.1; versions 9.3 and 9.4 do not appear in the totemomail history. Nevertheless, an entry in 9.5.1 refers to “security hardening introduced in 9.3.1.” The intermediate versions come from Kiteworks Email Protection Gateway (EPG), which is based on the same codebase: Kiteworks security advisories list EPG 9.3.1, 9.4.0, 9.4.1, 9.5.0, and 9.5.1 as fixed versions. For totemomail installations running 9.2.1, 9.5.1 is the next update; the release notes list no additional patch in between.

## Vulnerabilities without authentication

**Server-Side Request Forgery through message content.** totemomail retrieved URLs from processed messages on the server side without sufficiently restricting the destination. An unauthenticated sender could craft a message that caused the gateway to send requests to internal services or metadata endpoints of cloud instances and return the responses. All that is required is an email processed by the gateway; for a gateway in the inbound mail flow, any external sender can do this. This is particularly sensitive for installations in Azure, AWS, or other clouds, where the metadata endpoint `169.254.169.254` provides configuration data and, depending on the platform, instance access tokens. The release notes do not list a CVE number for this vulnerability. Kiteworks lists five EPG advisories for access to internal network resources (CVE-2026-102095, CVE-2026-102102 through CVE-2026-102105, each with CVSS 9.1, fixed in EPG 9.5.0); the brief advisory texts do not reveal which of them corresponds to the totemomail entry.

**Password reset in the Web Portal (CVE-2026-85066).** A flaw in the password reset flow made it possible to bypass verification steps and set a new password for an existing account. The release notes explicitly include administrator accounts. Kiteworks rates the vulnerability CVSS 9.8 (critical) in the related advisory GHSA-rwpq-5xfv-54pv and lists EPG 9.5.0 as the fixed version. The Web Portal is intended for external recipients of encrypted messages and is therefore usually publicly accessible.

## Vulnerabilities requiring administrator privileges

Three additional fixes require authentication with administrative privileges. In combination with the account takeover in the Web Portal, however, those requirements carry less weight.

- **Admin API, queue management (CVE-2026-34843):** Insufficiently protected file uploads could be used to write files to arbitrary locations in the file system. According to the release notes, this enabled remote code execution. Advisory GHSA-mhc9-4pph-2r9x rates the vulnerability CVSS 7.2; it has been fixed in EPG since 9.4.0.
- **Rule Designer:** When importing a rule configuration, the processing pipeline could load and execute code outside the approved mail processing components. Permissions to configure mail rules are required.
- **Mail API:** A path traversal vulnerability existed when extracting uploaded ZIP files. Files are now extracted only to the temporary directory.

The fourth fix concerns the property `security.login.requestingKeysAvailable`. Previously, it only hid the link for requesting public keys in the Web Portal; the feature itself remained accessible through a direct request. Starting with 9.5.1, the property disables the feature. If you set the property to prevent key requests, it will only work as expected after this update.

## What the Kiteworks registry lists in addition

On September 30, 2026, Kiteworks published 125 security advisories in its advisory registry on GitHub, including 28 for Email Protection Gateway. The totemomail 9.5.1 release notes list significantly fewer items. The registry does not distinguish between EPG and totemomail; it does not indicate whether every EPG vulnerability also affects totemomail. The critical EPG entries are:

| CVE | CVSS | Impact according to the advisory | Fixed in EPG starting with |
|---|---|---|---|
| CVE-2026-54154 | 10.0 | Unauthenticated code execution with root privileges | 9.4.1 |
| CVE-2026-85066 | 9.8 | User account takeover (password reset) | 9.5.0 |
| CVE-2026-85065 | 9.8 | User account takeover | 9.5.0 |
| CVE-2026-102149 | 9.4 | User account takeover | 9.5.1 |
| CVE-2026-102106 | 9.1 | Modification of the administration configuration | 9.5.0 |
| CVE-2026-102095, CVE-2026-102102 through 102105 | 9.1 | Access to internal network resources | 9.5.0 |

There are also 13 entries rated high, most of them code execution with administrator privileges (CVSS 7.2). The highest listed fixed version is 9.5.1; Kiteworks described 9.5.1 as the current version at the end of September, stating that it addressed all known vulnerabilities. Updating totemomail to 9.5.1 therefore brings it in line with all EPG advisories in the registry.

## Updated libraries

The bundled Apache Tomcat and Apache XML Graphics Batik libraries have been updated. The release notes list CVE-2026-50229, CVE-2026-53404, CVE-2026-53434, CVE-2026-55276, CVE-2026-55955, CVE-2026-55956, and CVE-2026-59083, without assigning them individually to the two libraries. In addition, woodstox-core-asl was updated because of a vulnerability; no CVE number is provided for it.

## Improvement and bug fixes

- **EPG migration:** When migrating from an Oracle database, totemomail now supports Oracle Wallet. If no database password is configured, the migration connects through the wallet.
- **RMS:** Messages could get stuck in processing because RMS actions waited indefinitely for authentication. The wait time is now limited to one minute. This affects installations using Microsoft Purview Information Protection (RMS).
- **TRE:** The “Open Online” feature for Registered Envelopes works again. It had stopped working since the aforementioned hardening in 9.3.1.
- **License report:** For cloud licenses, the report once again correctly counts internal users with predefined feature types. The article on the [license limit and cleanup via LDAP](/blog/totemomail-licensed-user-limit-ldap-cleanup) describes how orphaned accounts affect license counting.

## Installation

The patch can be installed on systems running totemomail 7 b985 or later on Windows, Linux, Solaris, and FreeBSD. Sign in to the administration console, upload the included TPF file under `[Instanzname]` > “Update,” and start the update. For multiple instances, this must be done separately for each instance. totemomail will restart itself afterward if possible; schedule a maintenance window and create a backup or snapshot beforehand.

Until the update is installed, two measures reduce the risk: make the admin console and Admin API accessible only from the management network, and restrict the gateway’s outbound connections to the destinations actually required. For cloud installations, the metadata endpoint `169.254.169.254` is among the destinations the gateway does not need to reach. Neither measure replaces the patch; only the update fixes the password reset vulnerability in the Web Portal.

## Sources

1.  totemomail Release Notes 9.5.1 (Build 101), totemo ag: installation guide and release notes as a PDF, available to customers through vendor support; basis for all information in this article.

2.  [Kiteworks Security Advisories on GitHub](https://github.com/kiteworks/security-advisories/security/advisories): Official registry with CVSS scores and fixed versions for Email Protection Gateway, Core, MFT, and Advanced Forms.

3.  [GHSA-rwpq-5xfv-54pv: Account takeover (CVE-2026-85066)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-rwpq-5xfv-54pv): Advisory on the password reset flaw, CVSS 9.8, fixed in EPG 9.5.0.

4.  [GHSA-5xhq-9wq3-rvj6: Arbitrary code execution (CVE-2026-54154)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-5xhq-9wq3-rvj6): Most critical EPG vulnerability with CVSS 10.0, fixed in EPG 9.4.1.

5.  [P.K. Sharma: Kiteworks published 125 advisories](https://www.pk-sharma.com/briefing/kiteworks-125-advisories-which-is-the-shutdown-flaw): Analysis of the September 30, 2026 publication, including counts by product and severity, and Kiteworks’ statements about 9.5.1.

6.  [Understanding mail routing between totemomail and Exchange Online](/blog/totemomail-m365): How totemomail processes incoming messages; relevant to the question of which emails can trigger the SSRF vulnerability.

7.  [Totemomail license limit reached: clean up orphaned users via LDAP](/blog/totemomail-licensed-user-limit-ldap-cleanup): Background on the license counting affected by the license report fix.
