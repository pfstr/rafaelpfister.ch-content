---
title: "Proper follow-up for the July 2026 Exchange security updates"
navTitle: "Exchange SU 07/2026"
description: "Two clean-up tasks are required after installation: verify the removal of the old CVE-2026-42897 mitigation and check for overprivileged legacy groups in Active Directory."
date: "2026-07-14"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "6 min read"
themen:
  - exchange-onprem-hybrid
  - active-directory-entra
slug: "exchange-server-security-updates-july-2026"
translationOf: "exchange-security-updates-juli-2026"
url: "https://rafaelpfister.ch/en/blog/exchange-server-security-updates-july-2026"
translationId: article-731b5b840aee096c
translatedAt: 2026-07-28T11:10:30.445Z
translationReview: automatic
translationSourceHash: c4f0a68a6d0b88997bcc5dadd9f5c2423dcb61c7986e179a099460335042a23a
---

# Proper follow-up for the July 2026 Exchange security updates

Installing the Exchange security updates released on 14 July 2026 does not complete the work. Administrators should then clear up two remnants: the mitigation for **CVE-2026-42897** activated in May, and two historical Exchange security groups with extensive permissions in Active Directory.

Both tasks are easy to overlook. The mitigation deliberately remains in place until it is removed in a controlled manner. The groups, meanwhile, may have survived every migration unnoticed for many years.

## Which Exchange versions the update is available for

The SUs are available for the following versions:

- **Exchange Server Subscription Edition (SE) RTM**: as a regularly available public update.
- **Exchange Server 2019 CU14 and CU15**: only for organisations enrolled in the **Period 2 ESU programme**.
- **Exchange Server 2016 CU23**: likewise only through Period 2 ESU.

Exchange 2016 and 2019 are out of support. Anyone not in the Period 2 ESU programme (valid from May to October 2026) will no longer receive these updates and should no longer postpone moving to Exchange SE. Exchange Online environments are already protected; in hybrid setups, however, the SU must still be installed on all Exchange servers, including management-only servers. As usual, the specific CVEs addressed are listed in the Security Update Guide (filter for ‘Server Software’ for Exchange SE or ‘ESU’ for 2016/2019).

There is a known issue in the current release: so-called *wrapper messages* can appear in the Inbox of shared mailboxes in hybrid environments. Details are available in the relevant Microsoft support article.

## Remove the CVE-2026-42897 mitigation after installation

### Brief recap

CVE-2026-42897 was announced on 14 May 2026: a cross-site scripting vulnerability (spoofing) in Outlook Web Access. An attacker sends a specially crafted email; if the victim opens it in OWA and certain interaction conditions are met, arbitrary JavaScript can be executed in the browser context. Exchange 2016, 2019 and SE were affected at *every* patch level. Microsoft released an emergency mitigation on the same day (ID **M2.1.x**, with the specific IIS rule named **M2.1.0**) and delivered the actual fix with the June 2026 SU.

### Why the July update does *not* remove the mitigation automatically

This is the point that surprises most people: a mitigation already applied remains active even after installing the July SU. The reason is the mechanism involved. The mitigation is a **Content Security Policy-based IIS URL Rewrite rule** applied *outside* the MSI installer, either by the Emergency Mitigation Service (EM Service) or by the EOMT script. The MSI patch replaces binaries but does not manage these out-of-band IIS rules. Removing it is therefore a separate manual step.

Incidentally, the mitigation never protected IE clients or Edge in IE mode anyway, because Internet Explorer does not support CSP. Anyone using such clients was never protected by the mitigation alone. This is another argument for patching promptly rather than relying on the mitigation.

### The pitfall: the EM Service reapplies the mitigation

Anyone who deletes the rule prematurely will encounter a surprise. The EM Service runs hourly and compares the current state with the requirements supplied by the Office Config Service (flighting). The mapping of ‘which build requires which mitigation’ is held server-side. Only a server-side change marks the July 2026 build as ‘mitigation no longer required’. According to Microsoft, this change was not fully rolled out until around 16 July 2026. Until then, the EM Service simply recreates a deleted M2.1.0 rule during its next hourly run.

In practice, this means either waiting until after 16 July before removing it manually, or explicitly blocking the mitigation so that it cannot be reactivated.

### How to remove the mitigation cleanly (EM Service path)

First check what has actually been applied:

```powershell
Get-ExchangeServer -Identity <ServerName> | Format-List Name,MitigationsApplied,MitigationsBlocked
```

To prevent reactivation, add the mitigation ID to the block list: entries there are ignored by the EM Service during its hourly run.

```powershell
Set-ExchangeServer -Identity <ServerName> -MitigationsBlocked @("M2.1.0")
```

Then remove the actual IIS rule. Useful to know and rarely documented: the EM Service creates its URL Rewrite rules with the **prefix ‘EEMS `<Mitigation-ID>` `<Beschreibung>`’**. This makes it possible to identify them unambiguously in IIS Manager under URL Rewrite (or via `appcmd`/PowerShell in `applicationHost.config`) without having to guess which rule belongs to the mitigation. Once the server-side change has been rolled out, you can remove the block again (`-MitigationsBlocked @()`) if it was only applied as a temporary measure.

### EOMT path (isolated or air-gapped environments)

If the mitigation was applied using the downloadable **EOMT script** (https://aka.ms/UnifiedEOMT), it is rolled back using the rollback switch:

```powershell
.\EOMT.ps1 -RollbackMitigation -CVE "CVE-2026-42897"
```

Here too, there is a little-known detail: before every change, EOMT saves the original IIS state in a **CVE-specific JSON backup file** under `%WINDIR%\System32\inetsrv\config\`. The rollback reads exactly that file and restores the original settings. Important: a mitigation applied with a legacy script (EOMTv2 etc.) must also be removed using that script's own rollback mechanism; the backup formats are not compatible.

### Why removal is worthwhile

The mitigation is not ‘free’. As long as it remains active, it carries its known side effects: the OWA ‘Print calendar’ function does not work, inline images may not display correctly in the OWA reading pane, OWA Light (`/?layout=light`) is broken (and will be retired soon anyway), and published calendars sometimes return error 500. Particularly troublesome for monitoring: the **OWACalendar.Proxy** health set can become *unhealthy*, triggering false alerts in monitoring. Anyone who installs the SU but leaves the mitigation in place ends up chasing ghosts. Once the update is installed *and* the mitigation has been removed, these known issues disappear too.

One special case: in mixed environments, servers that have not yet been updated may retain the mitigation. However, be aware that Office Online Server (OOS) integration may only work properly again once *all* Exchange servers in the organisation are at the July level.

## Health Checker: identify ancient security groups

The second point, independent of the SU release: the **Exchange Health Checker** (https://aka.ms/ExchangeHealthChecker) now checks for the existence of two long-deprecated security groups: **‘Exchange Domain Servers’** and **‘Exchange Enterprise Servers’**.

### Where these groups come from and why they are a risk

These two groups originate from the Exchange 2000/2003 permissions model and have been deprecated since Exchange 2007. Exchange 2007/2010 introduced the split-permissions and RBAC model, and they have simply not been used since. The problem is that they did not disappear. In many directories, they have been sitting unnoticed for around two decades and may still carry extensive ACLs from the old model — more permissions than a modern Exchange security group would ever have.

This is exactly what makes them an attack vector. A dormant group with persistent, broad permissions is a classic escalation chain: anyone who manages to add themselves (or a controlled account) to such a group inherits its directory permissions. Because nobody actively monitors the group, such manipulation is unlikely to be noticed.

### Why most admins do not have them on their radar

These groups are a blind spot for several reasons: they have been inactive for around 20 years, usually existed before the current team's tenure, survive every migration without complaint, and were never previously reported by Health Checker. Particularly problematic: they even survive the *complete* decommissioning of on-premises Exchange. Anyone who removes the last Exchange server will generally clean up the server objects but overlook these legacy groups entirely.

### Clean-up

Health Checker will report the groups automatically in future. You can find them manually in Active Directory (usually in the `Users` container) or using PowerShell:

```powershell
Get-ADGroup -Filter "Name -eq 'Exchange Domain Servers' -or Name -eq 'Exchange Enterprise Servers'"
```

Procedure: check membership and any custom ACL references, ensure that nothing in production refers to them, then delete the groups. Since they have been deprecated since 2007, they can be removed safely in the vast majority of environments. If you no longer run any on-premises Exchange at all, you should also plan a more comprehensive AD clean-up following the official Microsoft guidance.

Hayes Jupe has written a detailed guide to removing the groups in his blog post [Latest Exchange health check script and deprecated groups](https://www.hayesjupe.com/latest-exchange-health-check-script-and-deprecated-groups/).

## Recommended procedure

In short, the practical sequence is as follows: first, use Health Checker to inventory the environment (it shows missing CUs/SUs, outstanding manual steps *and* now the legacy groups). Then install the current CU and the July SU, restart the server, and check that all Exchange services have started properly. Afterwards, run Health Checker again, remove the CVE-2026-42897 mitigation (after 16 July or by first blocking ID M2.1.0), and finally clean up the deprecated security groups. SUs are cumulative: if you are on a supported CU, you do not need to install every intermediate SU, but can install the latest one directly.

## Sources

1.  [Released: July 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-july-2026-exchange-server-security-updates/4534146): Official announcement of the July release, including supported versions and the known wrapper-message issue.

2.  [Addressing Exchange Server May 2026 vulnerability CVE-2026-42897 – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/addressing-exchange-server-may-2026-vulnerability-cve-2026-42897/4518498): Original security notice, including the emergency mitigation and known side effects in OWA.

3.  [Released: June 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-june-2026-exchange-server-security-updates/4524491): The June release that delivered the actual fix for CVE-2026-42897.

4.  [Exchange Emergency Mitigation Service (Exchange EM Service) – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-emergency-mitigation-service): How the EM Service works, comparing mitigations hourly and recreating a rule deleted prematurely.

5.  [Set-ExchangeServer (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-exchangeserver): Parameters `MitigationsApplied` and `MitigationsBlocked` for checking mitigations and preventing reactivation.

6.  [Exchange On-premises Mitigation Tool (EOMT) – Microsoft CSS-Exchange](https://microsoft.github.io/CSS-Exchange/Security/EOMT/): The EOMT script, including the rollback switch and CVE-specific JSON backup of the original IIS state.

7.  [CVE-2026-42897 Detail – NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-42897): Technical description and assessment of the vulnerability in the National Vulnerability Database.
