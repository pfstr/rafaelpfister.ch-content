---
title: "Entra Connect Sync 2.6.84.0: What’s Changing and Who Should Update Now"
navTitle: "Entra Connect 2.6.84"
description: "The security release brings Passkey support and changes to app authentication, PowerShell, and Password Hash Sync. The previous version was withdrawn, so the update requires a phased decision."
date: "2026-07-17"
kategorie: "Microsoft Entra"
timeToRead: "11 min read"
themen:
  - microsoft-entra
  - active-directory-entra
slug: "entra-connect-sync-2-6-84-0"
translationOf: "entra-connect-2-6-84-0"
draft: false
translationId: article-85bd27acb917e406
translatedAt: 2026-09-04T08:49:01.269Z
translationReview: required
translationSourceHash: da16eeec10c227af5ba6f33ae138e0148db5b34736874eeed6b2b60c0b469a81
url: https://rafaelpfister.ch/en/blog/entra-connect-sync-2-6-84-0
translationModel: gpt-5.6-terra
---

# Entra Connect Sync 2.6.84.0: What’s Changing and Who Should Update Now

Microsoft released Entra Connect Sync 2.6.84.0 as a security release on July 7, 2026, and recommends upgrading promptly. At the same time, the direct predecessor, 2.6.79.0, was withdrawn due to an installer issue discovered afterward. The consequence is neither “install it everywhere immediately” nor “wait and ignore it”: affected systems and systems that will soon fall out of support should move quickly, while everyone else can test the update in a controlled manner first.

## Why This Release Deserves Special Caution

The 2.6 line of Entra Connect Sync has had a rough start. A brief review is relevant to the update decision:

- **2.6.1.0** (February 2026) fixed, among other things, an issue where editing the Entra ID connector configuration in Synchronization Service Manager deleted the Application-Based Authentication parameters, causing the wizard and certificate rotation to fail. Therefore, the notable recommendation for all 2.5 versions was simply not to use the product’s management interface.
- **2.6.3.0** (March 2026) was a hotfix for an issue where auto-upgrade could unexpectedly stop the Entra Connect server. The workaround at the time: auto-upgrade detects manually modified configuration files and simply skips those servers.
- **2.6.79.0** (June 2026) was completely withdrawn after release. The installer is no longer available; according to Microsoft, anyone who installed the version should uninstall it and install 2.6.84.0. Microsoft does not document exactly what the issue was.

As of today, version 2.6.84.0 is available only for download through the Microsoft Entra admin center (“Released for download”). An auto-upgrade rollout has not yet been announced. This, too, is a signal: Microsoft itself is not yet broadly distributing the version to existing installations.

## New Features

### Phishing-Resistant Sign-In in the Setup Wizard (Preview)

The Setup Wizard now supports sign-in with Passkeys and FIDO2 security keys through the Windows Web Account Manager (WAM). The background: since 2024/2025, Microsoft has been progressively enforcing MFA for sign-ins to Azure and Entra administration interfaces, and many organizations have restricted their admin accounts through Conditional Access to phishing-resistant methods (FIDO2, Passkeys, certificate-based authentication). Until now, these properly secured accounts could not sign in to the Entra Connect wizard because the embedded sign-in dialog did not support those methods. In practice, this led to unattractive workarounds, such as separate “setup accounts” with weaker authentication requirements just to get through the wizard. This gap is now being closed, albeit initially as a preview.

### Support for the French Sovereign Cloud

Version 2.6.84.0 adds support for the French Sovereign Cloud environment, including Pass-through Authentication, Seamless Single Sign-On, Password Writeback, and Health agent monitoring. Correspondingly, it fixes an issue where the Application Proxy cloud name in the France Cloud was not resolved correctly and PTA registration failed with “EnvironmentName attribute is invalid.”

## Behavior Changes in Detail

The most interesting part of this release is not its new features, but its changed behaviors. Several of them correct design decisions that have caused surprises in practice.

### Auto-Upgrade No Longer Destroys Customized Configuration Files

This is the change with the longest history. Until now, auto-upgrade completely overwrote the `miiserver.exe.config` file during an update. Manual customizations were lost. That may sound like an edge case, but it was not: Microsoft itself had instructed administrators in FIPS environments to edit exactly this file so that Password Hash Synchronization works with FIPS mode enabled. Anyone following the official instructions therefore had a “modified” configuration file.

The consequences became apparent as a known issue when upgrading to 2.5.190.0 and 2.6.1.0: if the installer detects an altered `miiserver.exe.config`, it leaves the file untouched; however, the new assembly binding is then missing, and the synchronization service fails after the upgrade with `System.IO.FileLoadException: Could not load file or assembly 'System.Diagnostics.DiagnosticSource, Version=6.0.0.1'`. The documented workaround: manually add a bindingRedirect in the `assemblyBinding` section of `miiserver.exe.config` (under `%programfiles%\Microsoft Azure AD Sync\Bin`):

```xml
<dependentAssembly>
  <assemblyIdentity name="System.Diagnostics.DiagnosticSource" publicKeyToken="cc7b13ffcd2ddd51" culture="neutral" />
  <bindingRedirect oldVersion="0.0.0.0-8.0.0.0" newVersion="8.0.0.0" />
</dependentAssembly>
```

Then restart the ADSync service. Hotfix 2.6.3.0 mitigated the issue only for auto-upgrade: affected servers were simply skipped and remained on the old version. Version 2.6.84.0 provides the actual solution: the upgrade process merges customer customizations with the new configuration and validates the result before applying it. Anyone manually upgrading from an affected version should still check the state of their `miiserver.exe.config` beforehand and back up the file: the merge mechanism is new and therefore has not yet been proven in practice.

### Application-Based Authentication: No More Silent Fallbacks or Silent Switching

As a reminder: since 2.5.76.0, Application-Based Authentication (ABA) has been generally available and is the default. Instead of the old Directory Synchronization Account (a cloud account with a stored password), the sync server authenticates as an Entra ID application with a certificate, ideally protected by the TPM. This is a significantly more robust architecture: no password that can leak and a credential tied to the machine.

Version 2.6.84.0 cleans up two behaviors that undermined this security gain:

**No more silent fallback.** If ABA setup failed in the wizard, setup previously fell back to the legacy account without comment. The result: the administrator believed they had certificate-based sign-in, while the server was actually running with the old password account. A classic fail-open pattern. The wizard now stops with a clear error message (“Microsoft Entra Connect could not configure application-based authentication for this server. Setup cannot continue.”), so that the underlying cause is fixed rather than concealed.

**No more automatic background switching.** Previously, Entra Connect independently switched existing servers from the legacy account to ABA during ongoing sync operations. Well-intentioned from a security perspective, but a significant operational risk: an authentication method changes without asking, without a change window, and without anyone knowing. And if something goes wrong (TPM issues, Conditional Access conflicts, firewall), synchronization stops. The new rule is: only new installations configure ABA automatically; existing servers switch only when an administrator starts the wizard and explicitly selects **Configure application-based authentication to Microsoft Entra ID**. The switch is thus back where it belongs: in a planned change.

In addition, TPM handling has been improved: setup now tests a certificate’s signing capability in advance and handles TPM signature verification correctly. On servers with faulty TPM firmware that cannot produce a valid signature, setup falls back in a controlled way to a software-based certificate. This also has a history: TPM-related ABA failures spanned several earlier releases (2.5.79.0, 2.5.190.0), including due to incompatibilities between TPM implementations and the default signing method of the MSAL library.

### PowerShell Cmdlets Now Require an Explicit Admin Sign-In

A change script operators need to know about: the cmdlets `Set-ADSyncAADCompanyFeature` and `Set-ADSyncAADPasswordSyncState`, which change cloud configuration, now require the `-AADUsername` parameter for interactive admin authentication. The wizard itself also no longer writes cloud changes using stored service credentials, but through an interactive MSAL sign-in. And the uninstall wizard asks for admin credentials to clean up cloud configuration; if you skip this, cleanup is performed locally only.

The underlying theme is the same as with ABA: actions against the tenant should be attributable to a real, traceable administrator identity rather than an anonymous service account. This aligns with a bug fix in the same release: until now, admin audit logging recorded the identity of the service account instead of the administrator actually making changes to synchronization rules—an audit trail that failed its purpose. Together, the two changes result in usable auditing. The practical consequence: anyone who previously ran these cmdlets unattended in scripts must redesign those processes; interactive authentication and automation do not mix.

### PHS Self-Healing Removed

The least conspicuous but conceptually most interesting change: Password Hash Synchronization no longer reactivates its cloud feature flag automatically in the background. If the flag is disabled, an administrator must explicitly enable it again.

Previously, if PHS was disabled at the tenant level—deliberately or accidentally—the feature “healed” itself and turned back on. For environments that had deliberately disabled PHS, such as for compliance reasons because password hashes must not flow to the cloud, or during a migration phase, this was a feature overriding a documented administrator decision. It was difficult to justify that a mechanism synchronizing password hashes would reactivate itself on its own authority.

However, the downside should not be overlooked: self-healing also saved environments where the flag was disabled by an error or a failed script without anyone noticing. That safeguard is now gone. Anyone using PHS in production, even only as a fallback for emergency sign-in, should actively monitor PHS status going forward, for example through Entra Connect Health or by checking synchronization heartbeat values.

### Updated Components: SQL LocalDB 2022, MSAL, VC++ Runtime

Less spectacular but overdue is the modernization of bundled components:

- **SQL Server LocalDB 2019 → 2022.** Entra Connect’s internal database was previously based on SQL Server 2019 Express LocalDB, a version whose mainstream support ended in February 2025. SQL Server 2022 puts the installation back on a version with ongoing support.
- **MSAL 4.64.1 → 4.83.3.** The Microsoft Authentication Library is the central component for all token acquisition (ABA, wizard sign-in, PowerShell). The jump of around twenty minor versions brings the accumulated fixes and improvements of the library.
- **Visual C++ Redistributable 2013 → 2015–2022 (14.42).** What is notable here is less the update than the legacy dependency: until this release, Entra Connect required a runtime environment whose support ended in April 2024. The VC++ 2013 dependency has now been completely removed.

This also fits with the general note in the release notes that “multiple security vulnerabilities in bundled third-party dependencies” have been fixed. That is likely the main reason for classifying this as a security release: outdated bundled components are not a cosmetic issue in a product that runs with near-Domain Admin privileges at the center of identity infrastructure.

## The Remaining Bug Fixes

For completeness, here are the remaining fixes:

- **Metaverse search in Synchronization Service Manager** has been repaired. After the warning not to use the interface at all in older versions, it now apparently receives maintenance again.
- **PowerShell diagnostic report (HTML)** renders correctly again; relevant to anyone using `Invoke-ADSyncDiagnostics` for support cases.
- **Generic SQL Connector:** Profile creation failed because required parameters were not populated during configuration. This affects environments connecting additional directories through the GSQL connector.
- **China Cloud:** The instance name was not correctly resolved by the discovery endpoint API, which could cause cloud instance detection to fail.
- **Admin audit logging** now records the actual administrator instead of the service account when synchronization rules are changed (see above).

## Support Deadlines: Who Still Needs to Act Now

Since March 2023, a strict retirement policy has applied to Entra Connect Sync 2.x: each version falls out of support twelve months after its successor is released. The current deadlines are:

| Version | End of support |
| --- | --- |
| 2.5.3.0 | **July 31, 2026** |
| 2.5.76.0 | September 1, 2026 |
| 2.5.79.0 | October 23, 2026 |
| 2.5.190.0 | February 2, 2027 |
| 2.6.1.0 | March 10, 2027 |
| 2.6.3.0 | July 7, 2027 |

Anyone still running 2.5.3.0 therefore has only two weeks of support remaining. The question here is not whether to update, but only which version to update to. Microsoft also emphasizes that versions outside support can “unexpectedly” stop working; synchronization for retired 1.x versions has in fact since been disabled server-side. The minimum requirements remain .NET Framework 4.7.2 and TLS 1.2; the installer is available exclusively in the Entra admin center (Entra ID → Entra Connect → Get started), no longer in the Download Center.

## Recommendation by Starting Version

Microsoft recommends updating “as soon as possible.” However, this exact recommendation also appeared above version 2.6.79.0, the version that was subsequently withdrawn. The recent release history—withdrawn installer, hotfix due to stopped servers, UI warnings across multiple versions—justifies a sober assessment instead of a reflex.

My assessment for typical environments:

**Waiting a few weeks is reasonable** if you are running a still-supported version (2.5.190.0 or later), none of the fixed issues affect you urgently, and none of the new features are needed. Based on the release notes, the fixed security vulnerabilities are in bundled third-party components; an Entra Connect server should in any case be isolated well enough—no internet access except to Microsoft endpoints, no interactive sign-ins, Tier 0 treatment—that the time window is defensible. If the version remains without a recall for several weeks and Microsoft starts the auto-upgrade rollout, that is a much better quality signal than any announcement.

**You should act promptly** if any of the following applies:

- **You have 2.6.79.0 installed.** The instruction is then clear: uninstall it and install 2.6.84.0; do not wait.
- **You are running 2.5.3.0** (end of support July 31, 2026) or an even older version that has already expired.
- **One of the fixed issues directly affects you**, such as ABA setup on TPM servers, the GSQL connector, or the audit requirement that rule changes be attributed to the correct administrator.

For the upgrade itself, the usual procedure applies—and is especially advisable given this release history: export the configuration beforehand (the wizard offers **View or export current configuration**), install the update first on a staging mode server and test sync cycles, the wizard, and certificate rotation there, then update the active server. Anyone with a customized `miiserver.exe.config` should back it up before the update and verify afterward that the new merge mechanism correctly retained the customizations. And anyone running scripts with `Set-ADSyncAADCompanyFeature` or `Set-ADSyncAADPasswordSyncState` should test them before the production rollout; otherwise, they will fail on the new required parameter.

## Sources

1. [Microsoft Entra Connect: Version release history – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history): Official release notes for 2.6.84.0, including the recall notice for 2.6.79.0, the retirement table, and the known issue with modified miiserver.exe.config.
1. [Microsoft Entra Connect: Upgrade from a previous version to the latest – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-upgrade-previous-version): Upgrade procedure, including swing migration through a staging mode server.
1. [Authenticate to Microsoft Entra ID by using application identity – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/authenticate-application-id): How Application-Based Authentication works, replacing the legacy service account.
1. [Microsoft Entra Connect: Phishing-resistant authentication – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-passwordless-authentication): The new Passkey/FIDO2 sign-in in the Setup Wizard through Windows Web Account Manager.
1. [Microsoft Entra Connect: Automatic upgrade – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-install-automatic-upgrade): The mechanics and prerequisites of auto-upgrade, whose rollout for 2.6.84.0 is still pending.
1. [Auditing administrator events in Microsoft Entra Connect Sync – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/admin-audit-logging): Admin audit logging, whose identity attribution for synchronization rules was corrected in this release.
1. [SQL Server 2019 – Microsoft Lifecycle](https://learn.microsoft.com/en-us/lifecycle/products/sql-server-2019): Support dates for the previously bundled LocalDB foundation, whose mainstream support ended in February 2025.
