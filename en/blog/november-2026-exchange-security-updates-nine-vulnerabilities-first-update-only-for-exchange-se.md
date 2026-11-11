---
title: "November 2026 Exchange security updates: nine vulnerabilities, first update only for Exchange SE"
navTitle: "Exchange SU 11/2026"
description: "The November SU closes nine vulnerabilities in Exchange Server SE, including a critical remote code execution for which a low-privileged account is sufficient. Exchange 2016 and 2019 no longer receive updates. The November Windows updates fix the Exchange outages caused by the October Windows updates."
date: "2026-11-11"
kategorie: "Exchange On-Premises / Hybrid"
timeToRead: "6 min read"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
slug: "november-2026-exchange-security-updates-nine-vulnerabilities-first-update-only-for-exchange-se"
translationId: article-cfb739155e545cc3
translationOf: exchange-security-updates-november-2026
url: https://rafaelpfister.ch/en/blog/november-2026-exchange-security-updates-nine-vulnerabilities-first-update-only-for-exchange-se
translationSourceHash: 1ce238b3219fc9c3670ec3137e807538c62e566ad22bd60fbee45dcf6dce9193
translationModel: gpt-5.6-terra
translatedAt: 2026-11-11T10:00:21.129Z
translationReview: automatic
---

# November 2026 Exchange security updates: nine vulnerabilities, first update only for Exchange SE

Microsoft released a security update (SU) for Exchange Server SE on November 10, 2026. According to the KB article, it closes nine vulnerabilities. None of them was publicly known in advance, and according to the Security Update Guide, none is being actively exploited. Microsoft rates one remote code execution as *Critical* and “Exploitation More Likely”; a low-privileged account is sufficient for it. This month is notable for two reasons: it is the first SU released only for Exchange SE, and the Exchange outages following the October Windows Server updates are fixed by the November Windows updates.

## Which Exchange versions the update is available for

The November 10, 2026 SU is available for only one version:

- **Exchange Server Subscription Edition (SE) RTM**: KB5133453, build 15.2.2562.57; publicly available.

You can compare your own version against the overview of [Exchange build numbers](/tools/exchange-builds).

There is no update for Exchange 2016 and 2019. The [October SU](/blog/exchange-security-updates-oktober-2026) was the last one, including for customers in the Period 2 ESU program, and according to the announcement, this program has ended. According to Microsoft, Exchange Online is already protected; however, in hybrid environments, every Exchange server still needs the SU, including management-only servers and machines with the Exchange Management Tools.

## Overview of the vulnerabilities

| CVE | Type | CVSS |
| --- | --- | --- |
| CVE-2026-118207 | Remote Code Execution | 8.8 |
| CVE-2026-118149 | Remote Code Execution | 8.8 |
| CVE-2026-118150 | Elevation of Privilege | 8.8 |
| CVE-2026-118154 | Elevation of Privilege | 8.8 |
| CVE-2026-118153 | Denial of Service | 7.5 |
| CVE-2026-118151 | Elevation of Privilege | 7.2 |
| CVE-2026-118157 | Elevation of Privilege | 7.2 |
| CVE-2026-118208 | Spoofing | 7.1 |
| CVE-2026-118152 | Spoofing | 6.5 |

All nine vulnerabilities are reachable over the network (AV:N) and have low attack complexity (AC:L) according to their CVSS vector. The Security Update Guide lists Exchange Server SE RTM as the only affected product.

**[CVE-2026-118207](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118207) is the month's only vulnerability that Microsoft rates as *Critical* (Remote Code Execution, CVSS 8.8), and the only one assessed as “Exploitation More Likely.” According to the FAQ in the Security Update Guide, an authenticated low-privileged attacker can send a specially crafted request to an affected server and execute code on the server; no user interaction is required. Microsoft identifies insufficient input validation (CWE-706) as the cause. A regular mailbox account is therefore sufficient for the attack, for example one with a password obtained through phishing.

**[CVE-2026-118149](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118149) is the second remote code execution (CVSS 8.8, *Important*). Here too, according to the FAQ, an authenticated attacker sends a specially crafted request to an affected service and can then execute code on the target system; the cause is code injection (CWE-94). Microsoft rates exploitation as “Exploitation Unlikely.”

**[CVE-2026-118153](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118153) (Denial of Service, CVSS 7.5) is the only vulnerability that requires neither an account nor user interaction. According to the description, an unauthenticated attacker can take the service offline over the network; the cause is inefficient algorithmic complexity (CWE-407). According to the vector, only availability is affected, not confidentiality or integrity.

The four elevation-of-privilege vulnerabilities differ in their prerequisites: [CVE-2026-118150](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118150) and [CVE-2026-118154](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118154) (both CVSS 8.8) require only a low-privileged account, while [CVE-2026-118151](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118151) and [CVE-2026-118157](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118157) (both CVSS 7.2) require the attacker to already have high privileges. According to the FAQ, CVE-2026-118151 can allow the attacker to obtain SYSTEM privileges.

Of the two spoofing vulnerabilities, [CVE-2026-118208](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118208) (CVSS 7.1) requires a low-privileged account. [CVE-2026-118152](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118152) (CVSS 6.5) is a cross-site request forgery: the attacker does not need an account, but a user must participate (UI:R). All vulnerabilities except CVE-2026-118207 and CVE-2026-118149 are rated as “Exploitation Less Likely.”

## Exchange 2016 and 2019: no update, no further testing

The announcement mentions Exchange 2016 and 2019 only in a notice: these versions are no longer tested for vulnerabilities, and no further updates will be released. Microsoft writes that they should be considered permanently vulnerable (“permanently vulnerable”) and that organizations should upgrade immediately. The fact that the nine November CVEs list only Exchange SE therefore says nothing about whether older versions are affected. Frankys Web points out that Exchange SE and 2019 CU15 are technically close, and that at least some of the vulnerabilities are likely also present in Exchange 2019.

For operations, this means:

1. **Exchange 2019 CU15 and earlier, Exchange 2016**: Their state remains frozen at the October SU (with ESU) or earlier. The nine November vulnerabilities remain unpatched on these servers if they are present there, and Microsoft no longer tests this.

2. **Migration**: Microsoft recommends immediately moving to Exchange SE.

3. **Hybrid environments**: Exchange Online throttles and blocks mail flow from outdated on-premises servers; the rules are in the [article on transport enforcement](/blog/exchange-online-transport-enforcement-hybrid-server).

## Fixed issues

**Exchange outages after the October Windows Server updates** (all versions). The issue with `MapiExceptionMdbOffline` (ec=1142) is not fixed by the Exchange SU: according to the announcement, the November 2026 Windows Server updates must be installed. The release notes for these updates list the fix under “[Exchange Server]” and state that the issue is resolved with updates from November 10, 2026: KB5131080 for Windows Server 2025, KB5131090 for Windows Server 2022, KB5131085 for Windows Server 2019, and KB5131291 for Windows Server 2016. According to the announcement, the order in which you install the Exchange SU and the Windows update does not matter. If you set the registry value `EpmapSavedContextCap` from the [article on the Exchange outages](/blog/exchange-windows-updates-oktober-2026), you can leave it in place; according to Microsoft, it is no longer needed after installing the November Windows Server updates.

**Incorrect free/busy display through Microsoft Graph in hybrid environments** (Exchange SE only). According to [KB5135867](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5135867), users with an on-premises mailbox did not see correct free/busy information for Exchange Online mailboxes in classic Outlook when the Graph API was enabled; only time blocks were displayed. The display was correct with Exchange Web Services (EWS) instead of the Graph API. The issue is fixed with KB5133453 or a newer update. The support article does not mention any follow-up action. If you switched back to EWS because of this issue, check the free/busy lookup for an Exchange Online mailbox after installation before re-enabling the Graph API.

## Known issues

Neither the announcement nor KB5133453 lists known issues for the November SU (as of November 11, 2026). The announcement has no comments from the Exchange team, and Microsoft notes that the documentation may still be incomplete at the time of publication. No known issue from previous months remains unresolved: the issues in the September updates have been resolved since the October SU, and the October Windows issue is fixed by the November Windows Server updates (see above).

## Installation and follow-up

In the announcement, Microsoft outlines the familiar process: use the [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) to inventory which servers need updates or manual steps, install the SU, restart the server, and check that all Exchange services are running. If services are disabled after setup, the installation was interrupted. For errors during or after installation, Microsoft refers to the [SetupAssist script](https://aka.ms/ExSetupAssist). According to KB5133453, after installation, Health Checker shows whether the SU was installed correctly and whether further steps are needed.

According to KB5133453, the installation file is called `ExchangeSubscriptionEdition-KB5133453-x64-en.exe` (Download Center: “SU12”). The SU replaces the October SU KB5128895. SUs are cumulative: if you have an older SU installed on Exchange SE RTM, install the November SU directly.

For hybrid environments, the FAQ states: if the Auth certificate is changed after installing an SU, the Hybrid Configuration Wizard must be run again. On Windows Server 2025, installed Exchange SUs do not appear in Control Panel; Microsoft advises against uninstalling SUs.

The following follow-up tasks from previous months remain open if not yet completed: remove the free/busy override `EnableRouteThroughMSGraphFeature` (see [October article](/blog/exchange-security-updates-oktober-2026)), remove the wrapper SettingOverride `DisableBlockSharedAndUserMailboxHeaders` (see [September article](/blog/exchange-security-updates-september-2026)), and check whether the CVE-2026-42897 mitigation (M2.1.0) is still active (see [article on the July SU](/blog/exchange-security-updates-juli-2026)).

## Recommended approach

Install the November SU promptly on all servers running Exchange SE and on machines with the Management Tools: for the critical remote code execution CVE-2026-118207, a low-privileged account is sufficient, and Microsoft considers its exploitation more likely. In the same maintenance window, install the November Windows Server updates so that the cause of the October outages is fixed. There is no patch for Exchange 2016 and 2019; according to Microsoft, these servers should be considered permanently vulnerable and replaced with Exchange SE immediately.

## Sources

1.  [Released: November 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-november-2026-exchange-server-security-updates/4570368): Announcement from November 10, 2026, with the supported version, notice about Exchange 2016/2019, the fixed Windows issue, and FAQ (no comments from the Exchange team, as of November 11, 2026).

2.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM November 10, 2026 (KB5133453) – Microsoft Support](https://support.microsoft.com/help/5133453): CVE list, fixed issue, installation file, replaced update, and notice about Health Checker.

3.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Type, CVSS, severity, exploitation assessment, FAQ, and affected product for the nine November CVEs.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Build number of the November SU.

5.  [Microsoft graph displays incorrect free/busy availability information in Microsoft Exchange server SE hybrid setup – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5135867): Symptoms and fix with KB5133453.

6.  [Exchange servers may become unstable after installation of October 2026 Windows updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/exchange-servers-may-become-unstable-after-installation-of/ba-p/4565382): Exchange team post about the October Windows issue referenced by the announcement.

7.  [November 10, 2026: KB5131080 (OS Build 26100.33822) – Microsoft Support](https://support.microsoft.com/help/5131080): Release notes for Windows Server 2025 with the Exchange fix and the known issue marked as resolved.

8.  [November 10, 2026: KB5131090 (OS Build 20348.5870) – Microsoft Support](https://support.microsoft.com/help/5131090): Release notes for Windows Server 2022 with the same fix.

9.  [November 10, 2026: KB5131085 (OS Build 17763.9487) – Microsoft Support](https://support.microsoft.com/help/5131085): Release notes for Windows Server 2019 with the same fix.

10. [November 10, 2026: KB5131291 (OS Build 14393.9688) – Microsoft Support](https://support.microsoft.com/help/5131291): Release notes for Windows Server 2016 with the same fix.

11. [New security updates for Exchange Server (November 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-november-2026/): German-language breakdown of the nine CVEs and assessment for Exchange 2016 and 2019.

12. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventory of CU/SU versions before and after installation.
