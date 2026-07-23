---
title: "Exchange Server Security Updates July 2026: Removing the CVE-2026-42897 Mitigation and Cleaning Up Legacy Security Groups"
navTitle: "Exchange SU 07/2026"
description: "On July 14, 2026, Microsoft released the Exchange Security Updates. Two easily overlooked points: the CVE-2026-42897 mitigation applied in May can now be removed cleanly (including the pitfall that the Emergency Mitigation Service will otherwise re-apply it), and the Health Checker newly reports two ancient, over-privileged legacy security groups in Active Directory."
date: "2026-07-14"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "6 min to read"
themen:
  - "exchange-onprem-hybrid"
  - "active-directory-entra"
slug: "exchange-server-security-updates-july-2026"
translationOf: "exchange-security-updates-juli-2026"
url: "https://rafaelpfister.ch/en/blog/exchange-server-security-updates-july-2026"
---

# Exchange Server Security Updates July 2026

On July 14, 2026, Microsoft released the regular Security Updates (SUs) for Exchange Server. Besides the actual security fixes, the release contains two points that are easily overlooked in practice but important: you can (and should) now remove the mitigation for **CVE-2026-42897** applied back in May, and the Exchange Health Checker newly checks for ancient, long-obsolete security groups in Active Directory. The official blog posts only touch on both points briefly.

## What the July release contains

The SUs are available for the following versions:

- **Exchange Server Subscription Edition (SE) RTM**: as a regularly available, public update.
- **Exchange Server 2019 CU14 and CU15**: only for organizations enrolled in the **Period 2 ESU program**.
- **Exchange Server 2016 CU23**: likewise only via Period 2 ESU.

Exchange 2016 and 2019 are out of support. If you are not enrolled in the Period 2 ESU program (valid from May to October 2026), you no longer receive these updates and should no longer postpone the move to Exchange SE. Exchange Online environments are already protected; in hybrid setups the SU must nevertheless be installed on all Exchange servers, including management-only servers. Which specific CVEs are addressed is, as usual, listed in the Security Update Guide (filter "Server Software" for Exchange SE, or "ESU" for 2016/2019).

There is a known issue in the current release: in hybrid environments, so-called *wrapper messages* can appear in the inbox of shared mailboxes. See the corresponding Microsoft support article for details.

## Removing the CVE-2026-42897 mitigation after installation

### A brief recap

CVE-2026-42897 was disclosed on May 14, 2026: a cross-site scripting (spoofing) vulnerability in Outlook Web Access. An attacker sends a specially crafted email; if the victim opens it in OWA and certain interaction conditions are met, arbitrary JavaScript can be executed in the browser context. Exchange 2016, 2019, and SE were affected at *any* patch level. Microsoft published an emergency mitigation the same day (ID **M2.1.x**; the concrete IIS rule is called **M2.1.0**) and delivered the actual fix with the June 2026 SU.

### Why the July update does *not* remove the mitigation automatically

This is the point that surprises most people: even after installing the July SU, an already-applied mitigation stays active. The reason lies in the mechanics. The mitigation is a **Content-Security-Policy-based IIS URL rewrite rule** that was applied *outside* the MSI installer, either by the Emergency Mitigation Service (EM Service) or by the EOMT script. The MSI patch swaps out binaries but does not manage these out-of-band IIS rules. Removing it is therefore a separate, manual step.

As an aside: the mitigation never protected IE clients or Edge in IE mode anyway, because Internet Explorer does not support CSP. Anyone running such clients was never secured by the mitigation alone. That is another argument to patch promptly instead of relying on the mitigation.

### The pitfall: the EM Service re-applies the mitigation

If you delete the rule prematurely, you are in for a surprise. The EM Service runs hourly and reconciles the actual state with the specifications delivered by the Office Config Service (flighting). The mapping of "which build needs which mitigation" resides on the server side. Only a server-side change marks the July 2026 build as "mitigation no longer needed." According to Microsoft, that change was not fully rolled out until around July 16, 2026. Until then, the EM Service simply re-adds a deleted M2.1.0 rule on its next hourly run.

In practice this means: either wait with the manual removal until after July 16, or explicitly block the mitigation so it is not reactivated.

### How to remove the mitigation cleanly (EM Service path)

First check what is actually applied:

```powershell
Get-ExchangeServer -Identity <ServerName> | Format-List Name,MitigationsApplied,MitigationsBlocked
```

To prevent reactivation, add the mitigation ID to the block list: entries there are ignored by the EM Service on its hourly run.

```powershell
Set-ExchangeServer -Identity <ServerName> -MitigationsBlocked @("M2.1.0")
```

Then remove the actual IIS rule. Good to know and rarely documented: the EM Service creates its URL rewrite rules with the **prefix "EEMS `<mitigation ID>` `<description>`"**. This lets you find them unambiguously in IIS Manager under URL Rewrite (or via `appcmd`/PowerShell in `applicationHost.config`) without guessing which rule belongs to the mitigation. After the server-side change has rolled out, you can lift the block again (`-MitigationsBlocked @()`), provided you set it only as an interim measure.

### EOMT path (isolated or air-gapped environments)

If the mitigation was applied via the downloadable **EOMT script** (https://aka.ms/UnifiedEOMT), roll it back using the rollback switch:

```powershell
.\EOMT.ps1 -RollbackMitigation -CVE "CVE-2026-42897"
```

Here, too, a little-known detail: before each change, EOMT backs up the initial IIS state in a **CVE-specific JSON backup file** under `%WINDIR%\System32\inetsrv\config\`. The rollback reads exactly this file and restores the original settings. Important: a mitigation applied with a legacy script (EOMTv2, etc.) must also be rolled back with that script's own rollback mechanism: the backup formats are not compatible.

### Why removal is worth it

The mitigation is not "free." As long as it is active, you carry its known side effects: the OWA "print calendar" function does not work, inline images may not display correctly in the OWA reading pane, OWA Light (`/?layout=light`) is broken (it will be retired soon anyway), published calendars partly return error 500. Particularly treacherous for monitoring: the **OWACalendar.Proxy** health set can flip to *unhealthy* and thus trigger false alarms in monitoring. If you installed the SU but leave the mitigation in place, you end up chasing ghosts. Once the update is installed *and* the mitigation is removed, these known issues disappear as well.

A special case: in mixed environments, servers that have not yet been updated may keep the mitigation. You should know, however, that Office Online Server (OOS) integration may only work cleanly again once *all* Exchange servers in the organization are at the July level.

## Health Checker: tracking down ancient security groups

The second point, independent of the SU release: the **Exchange Health Checker** (https://aka.ms/ExchangeHealthChecker) newly checks for the existence of two long-deprecated security groups: **"Exchange Domain Servers"** and **"Exchange Enterprise Servers"**.

### Where these groups come from and why they are a risk

These two groups originate from the Exchange 2000/2003 permissions model and have been deprecated since Exchange 2007. Exchange 2007/2010 introduced the split-permissions / RBAC model, and since then they are simply no longer used. The problem: they did not disappear. In many directories they have been sitting untouched for around two decades, and in some cases still carry far-reaching ACLs from the old model, i.e. more rights than a modern Exchange security group would ever have.

That is exactly what makes them an attack vector. A dormant group with standing, broad permissions is a classic escalation chain: whoever manages to add themselves (or a controlled account) to such a group inherits its rights in the directory. Since no one actively monitors the group, such manipulation is barely noticed.

### Why most admins are not aware of them

These groups are a blind spot for several reasons: they have been inactive for ~20 years, usually existed before the current team's tenure, survive every migration without complaint, and were never shown by the Health Checker until now. Particularly tricky: they even survive the *complete* decommissioning of on-premises Exchange. Whoever removed the last Exchange server usually clears away the server objects but completely overlooks these legacy groups.

### Cleaning up

The Health Checker will report the groups automatically from now on. Manually, you find them in Active Directory (usually in the `Users` container) or via PowerShell:

```powershell
Get-ADGroup -Filter "Name -eq 'Exchange Domain Servers' -or Name -eq 'Exchange Enterprise Servers'"
```

Procedure: check membership and any custom ACL references, make sure nothing productive relies on them, and then delete the groups. Since they have been deprecated since 2007, they can be removed safely in the vast majority of environments. If you no longer run any on-premises Exchange at all, plan a more comprehensive AD cleanup following the official Microsoft guidance at the same time.

Hayes Jupe has written a detailed guide to removing these groups in his blog post [Latest Exchange health check script and deprecated groups](https://www.hayesjupe.com/latest-exchange-health-check-script-and-deprecated-groups/).

## Recommended procedure

In short, the practical sequence: first inventory the environment with the Health Checker (it shows missing CUs/SUs, open manual steps *and*, newly, the legacy groups). Then install the current CU and the July SU, restart the server, and check that all Exchange services have started cleanly. Then run the Health Checker again, remove the CVE-2026-42897 mitigation (after July 16, or with the ID M2.1.0 blocked beforehand), and finally clean up the deprecated security groups. SUs are cumulative: if you are on a supported CU, you do not need to install every intermediate SU, just the latest one directly.

## Sources

1.  [Released: July 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-july-2026-exchange-server-security-updates/4534146) — Official announcement of the July release with the supported versions and the known wrapper-message issue.

2.  [Addressing Exchange Server May 2026 vulnerability CVE-2026-42897 – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/addressing-exchange-server-may-2026-vulnerability-cve-2026-42897/4518498) — Original security advisory including the emergency mitigation and the known OWA side effects.

3.  [Released: June 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-june-2026-exchange-server-security-updates/4524491) — The June release that delivered the actual fix for CVE-2026-42897.

4.  [Exchange Emergency Mitigation Service (Exchange EM Service) – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-emergency-mitigation-service) — How the EM Service works, reconciling mitigations hourly and re-adding a prematurely deleted rule.

5.  [Set-ExchangeServer (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-exchangeserver) — The `MitigationsApplied` and `MitigationsBlocked` parameters for checking mitigations and preventing reactivation.

6.  [Exchange On-premises Mitigation Tool (EOMT) – Microsoft CSS-Exchange](https://microsoft.github.io/CSS-Exchange/Security/EOMT/) — The EOMT script including the rollback switch and CVE-specific JSON backup of the initial IIS state.

7.  [CVE-2026-42897 Detail – NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-42897) — Technical description and scoring of the vulnerability in the National Vulnerability Database.
