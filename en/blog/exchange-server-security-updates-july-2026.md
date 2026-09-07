---
title: "Properly follow up on the July 2026 Exchange security updates"
navTitle: "Exchange SU 07/2026"
description: "Two cleanup tasks are needed after installation: safely remove the old CVE-2026-42897 mitigation and review overprivileged legacy groups in Active Directory."
date: "2026-07-14"
kategorie: "Exchange On-Premises / Hybrid"
timeToRead: "6 min read"
themen:
  - exchange-updates
  - active-directory-entra
slug: "exchange-server-security-updates-july-2026"
translationOf: "exchange-security-updates-juli-2026"
translationId: article-731b5b840aee096c
translatedAt: 2026-09-05T07:51:19.968Z
translationReview: automatic
translationSourceHash: e5d9295515965d3e7801752cd605f6d2a78cacfc9fb965e0f63d645658b39e9b
url: https://rafaelpfister.ch/en/blog/exchange-server-security-updates-july-2026
translationModel: gpt-5.6-terra
---

# Properly follow up on the July 2026 Exchange security updates

Installing the Exchange security updates from July 14, 2026 does not complete the work. Administrators should then address two legacy items: the mitigation for **CVE-2026-42897** activated in May and two historical Exchange security groups with extensive permissions in Active Directory.

Both tasks are easy to overlook. The mitigation intentionally remains in place until it is removed in a controlled manner. Meanwhile, the groups may have survived every migration unnoticed for many years.

## Which Exchange versions the update is available for

The SUs are available for the following versions:

- **Exchange Server Subscription Edition (SE) RTM**: as a regularly available public update.
- **Exchange Server 2019 CU14 and CU15**: only for organizations enrolled in the **Period 2 ESU program**.
- **Exchange Server 2016 CU23**: also only through Period 2 ESU.

Exchange 2016 and 2019 are out of support. Those not enrolled in the Period 2 ESU program (valid from May through October 2026) will no longer receive these updates and should not delay moving to Exchange SE any longer. Exchange Online environments are already protected; in hybrid setups, however, the SU must still be installed on all Exchange servers, including management-only servers. As usual, the specific CVEs addressed are listed in the Security Update Guide (filter for “Server Software” for Exchange SE or “ESU” for 2016/2019).

There is a known issue in the current release: in hybrid environments, so-called *wrapper messages* may appear in the Inbox of shared mailboxes. See the relevant Microsoft support article for details.

## Remove the CVE-2026-42897 mitigation after installation

### A brief recap

CVE-2026-42897 was disclosed on May 14, 2026: a cross-site scripting vulnerability (spoofing) in Outlook Web Access. An attacker sends a specially crafted email; if the victim opens it in OWA and certain interaction conditions are met, arbitrary JavaScript can be executed in the browser context. Exchange 2016, 2019, and SE at *any* patch level were affected. Microsoft released an emergency mitigation the same day (ID **M2.1.x**, with the specific IIS rule named **M2.1.0**) and delivered the actual fix with the June 2026 SU.

### Why the July update does *not* remove the mitigation automatically

This is the point that surprises most people: even after installing the July SU, an already applied mitigation remains active. The reason lies in how it works. The mitigation is a **Content Security Policy-based IIS URL Rewrite rule** deployed *outside* the MSI installer, either by the Emergency Mitigation Service (EM Service) or by the EOMT script. The MSI patch replaces binaries but does not manage these out-of-band IIS rules. Therefore, removing it is a separate manual step.

Incidentally, the mitigation never protected IE clients or Edge in IE mode anyway, because Internet Explorer does not support CSP. Anyone using such clients was never secured by the mitigation alone. This is another reason to patch promptly rather than relying on the mitigation.

### The tricky part: the EM Service reapplies the mitigation

A rule deleted prematurely does not stay removed permanently. The EM Service runs hourly and compares the current state with the requirements provided by the Office Config Service (flighting). The mapping of “which build needs which mitigation” is maintained server-side. Only a server-side change marks the July 2026 build as “mitigation no longer required.” According to Microsoft, this change was not fully rolled out until around July 16, 2026. Until then, the EM Service simply adds a deleted M2.1.0 rule back during its next hourly run.

In practical terms, this means either waiting until after July 16 before removing it manually, or explicitly blocking the mitigation so that it cannot be reactivated.

### How to remove the mitigation cleanly (EM Service path)

First, check what has been applied:

```powershell
Get-ExchangeServer -Identity <Servername> | Format-List Name,MitigationsApplied,MitigationsBlocked
```

To prevent reactivation, add the mitigation ID to the block list: entries there are ignored by the EM Service during its hourly run.

```powershell
Set-ExchangeServer -Identity <Servername> -MitigationsBlocked @("M2.1.0")
```

Then remove the actual IIS rule. A useful and rarely documented detail: the EM Service creates its URL Rewrite rules with the **prefix “EEMS `<Mitigation-ID>` `<Beschreibung>`”**. This makes them easy to identify in IIS Manager under URL Rewrite (or via `appcmd`/PowerShell in `applicationHost.config`) without having to guess which rule belongs to the mitigation. After the server-side change has rolled out, you can remove the block again (`-MitigationsBlocked @()`), provided it was set only as a temporary measure.

### EOMT path (isolated or air-gapped environments)

If the mitigation was applied using the downloadable **EOMT script** (https://aka.ms/UnifiedEOMT), roll it back using the rollback switch:

```powershell
.\EOMT.ps1 -RollbackMitigation -CVE "CVE-2026-42897"
```

Here, too, there is a little-known detail: before every change, EOMT saves the original IIS state in a **CVE-specific JSON backup file** under `%WINDIR%\System32\inetsrv\config\`. The rollback reads exactly this file and restores the original settings. Important: a mitigation applied with a legacy script (EOMTv2, etc.) must also be removed using its own rollback mechanism: the backup formats are not compatible.

### Why removal is worthwhile

The mitigation is not “free.” As long as it remains active, its known side effects remain: the OWA “Print calendar” feature does not work, inline images may not display correctly in the OWA reading pane, OWA Light (`/?layout=light`) is broken (and will be retired soon anyway), and published calendars sometimes return error 500. Particularly tricky for monitoring: the **OWACalendar.Proxy** health set can turn *unhealthy*, triggering false alerts in monitoring. Anyone who installs the SU but leaves the mitigation in place may end up investigating errors that are not real. Once the update is installed *and* the mitigation is removed, these known issues disappear as well.

A special case: in mixed environments, servers that have not yet been updated may retain the mitigation. However, you should be aware that Office Online Server (OOS) integration may not work properly again until *all* Exchange servers in the organization are at the July level.

## Health Checker: finding ancient security groups

The second point, independent of the SU release: the **Exchange Health Checker** (https://aka.ms/ExchangeHealthChecker) now checks for the existence of two long-deprecated security groups: **“Exchange Domain Servers”** and **“Exchange Enterprise Servers.”**

### Where these groups came from and why they are a risk

These two groups originate from the Exchange 2000/2003 permissions model and have been deprecated since Exchange 2007. Exchange 2007/2010 introduced the split permissions and RBAC model, and these groups have simply not been used since then. The problem is that they did not disappear. In many directories, they have sat unnoticed for around two decades and may still carry extensive ACLs from the old model—that is, more permissions than a modern Exchange security group would ever have.

That is exactly what makes them an attack vector. An inactive group with broad standing permissions is a classic escalation path: anyone who manages to add themselves (or a controlled account) to such a group inherits its directory permissions. Since no one actively monitors the group, such manipulation is unlikely to be noticed.

### Why most admins do not know about them

These groups are a blind spot for several reasons: they have been inactive for about 20 years, usually existed before the current team’s tenure, survive every migration without issue, and have never previously been reported by Health Checker. Particularly concerning: they even survive the *complete* decommissioning of on-premises Exchange. Anyone who removes the last Exchange server typically cleans up the server objects but overlooks these legacy groups entirely.

### Cleanup

Health Checker will report the groups automatically going forward. You can find them manually in Active Directory (usually in the `Users` container) or with PowerShell:

```powershell
Get-ADGroup -Filter "Name -eq 'Exchange Domain Servers' -or Name -eq 'Exchange Enterprise Servers'"
```

The process: review membership and any custom ACL references, ensure that nothing in production refers to them, then delete the groups. Since they have been deprecated since 2007, they can be safely removed in the vast majority of environments. Those no longer operating any on-premises Exchange should use the opportunity to plan a more comprehensive AD cleanup following Microsoft’s official guidance.

Hayes Jupe provided detailed instructions for removing the groups in his blog post [Latest Exchange health check script and deprecated groups](https://www.hayesjupe.com/latest-exchange-health-check-script-and-deprecated-groups/).

## Recommended approach

In short, the practical sequence is as follows: first, inventory the environment with Health Checker (it shows missing CUs/SUs, outstanding manual steps, *and* now the legacy groups). Then install the current CU and the July SU, restart the server, and verify that all Exchange services started cleanly. Next, run Health Checker again, remove the CVE-2026-42897 mitigation (after July 16 or after first blocking ID M2.1.0), and finally clean up the deprecated security groups. SUs are cumulative: if you are on a supported CU, you do not need to install every intervening SU; install the latest one directly.

## Sources

1.  [Released: July 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-july-2026-exchange-server-security-updates/4534146): Official announcement of the July release, including supported versions and the known wrapper-message issue.

2.  [Addressing Exchange Server May 2026 vulnerability CVE-2026-42897 – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/addressing-exchange-server-may-2026-vulnerability-cve-2026-42897/4518498): Original security advisory, including the emergency mitigation and known OWA side effects.

3.  [Released: June 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-june-2026-exchange-server-security-updates/4524491): The June release that delivered the actual fix for CVE-2026-42897.

4.  [Exchange Emergency Mitigation Service (Exchange EM Service) – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-emergency-mitigation-service): How the EM Service works, comparing mitigations hourly and re-adding a rule deleted prematurely.

5.  [Set-ExchangeServer (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-exchangeserver): Parameters `MitigationsApplied` and `MitigationsBlocked` for checking mitigations and preventing reactivation.

6.  [Exchange On-premises Mitigation Tool (EOMT) – Microsoft CSS-Exchange](https://microsoft.github.io/CSS-Exchange/Security/EOMT/): The EOMT script, including the rollback switch and CVE-specific JSON backup of the original IIS state.

7.  [CVE-2026-42897 Detail – NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-42897): Technical description and assessment of the vulnerability in the National Vulnerability Database.
