---
title: "Microsoft Entra Connect Sync 2.6.84.0: Security Fixes, New Rules for App-Based Authentication and PHS – and the Lesson from the Recalled Predecessor"
navTitle: "Entra Connect 2.6.84"
description: "On July 7, 2026, Microsoft released Entra Connect Sync 2.6.84.0 – with security fixes, passkey support in the wizard, and a number of behavior changes around Application-Based Authentication, PowerShell cmdlets, and Password Hash Sync. Its direct predecessor 2.6.79.0 was recalled after release. What the release contains, the background behind it, and why a controlled waiting period is defensible despite the security fixes."
date: "2026-07-17"
kategorie: "Microsoft Entra"
timeToRead: "11 min to read"
themen:
  - "microsoft-entra"
  - "active-directory-ldap"
slug: "entra-connect-sync-2-6-84-0"
translationOf: "entra-connect-2-6-84-0"
url: "https://rafaelpfister.ch/en/blog/entra-connect-sync-2-6-84-0"
aiPrompt: |
  You are my Entra Connect administration assistant. Help me plan and execute the update to Microsoft Entra Connect Sync 2.6.84.0 cleanly. Proceed step by step:
  1. Determine the current version and its support deadline (note the 12-month retirement policy; 2.5.3.0 expires on July 31, 2026).
  2. Check whether miiserver.exe.config was modified manually (e.g. the FIPS/PHS adjustment) and whether the workaround with the bindingRedirect for System.Diagnostics.DiagnosticSource is needed.
  3. Check whether the server still uses the legacy Directory Synchronization Account or already uses Application-Based Authentication, and whether any scripts call Set-ADSyncAADCompanyFeature or Set-ADSyncAADPasswordSyncState (new mandatory parameter: -AADUsername).
  4. Test the update on a staging-mode server first, export the configuration beforehand, then move to production.
  Ask me for the values only I know (current version, staging server available, FIPS enabled, sovereign cloud).
---

# Microsoft Entra Connect Sync 2.6.84.0

**On July 7, 2026, Microsoft released version 2.6.84.0 of Entra Connect Sync and classifies it as a security release: "We recommend upgrading to this version as soon as possible." At the same time, the very same document notes that the direct predecessor 2.6.79.0 was recalled after release because an issue was identified in the installer afterwards. Taken together, the realistic assessment is: the update is necessary, but a day-one rollout to your production server is not – unless you fall into one of the exceptions described below.**

## The Starting Point: a Release with History

The 2.6 line of Entra Connect Sync has had a bumpy start. A short recap, because it matters for the update decision:

- **2.6.1.0** (February 2026) fixed, among other things, a bug where editing the Entra ID connector configuration in the Synchronization Service Manager deleted the Application-Based Authentication parameters – with the consequence that the wizard and certificate rotation failed. For all 2.5 versions, the remarkable recommendation was therefore to simply not use the product's own management UI.
- **2.6.3.0** (March 2026) was a hotfix for an issue where auto-upgrade could stop the Entra Connect server unexpectedly. The stopgap back then: auto-upgrade detects manually modified configuration files and simply skips those servers.
- **2.6.79.0** (June 2026) was completely withdrawn after publication. The installer is no longer available; according to Microsoft, anyone who installed this version should uninstall it and install 2.6.84.0. What exactly the problem was is not documented by Microsoft.

As of today, version 2.6.84.0 is only available as a download via the Microsoft Entra admin center ("Released for download"). An auto-upgrade rollout has not been announced yet – which is also a signal: Microsoft itself is not yet distributing this version broadly to existing installations.

## New Features

### Phishing-Resistant Sign-In in the Setup Wizard (Preview)

The setup wizard now supports signing in with passkeys and FIDO2 security keys via the Windows Web Account Manager (WAM). The background: since 2024/2025, Microsoft has been gradually enforcing MFA for sign-ins to Azure and Entra management surfaces, and many organizations have restricted their admin accounts to phishing-resistant methods (FIDO2, passkeys, certificate-based authentication) via Conditional Access. It was precisely these properly secured accounts that could not sign in to the Entra Connect wizard, because the embedded sign-in dialog did not support those methods. In practice, this led to ugly workarounds – such as dedicated "setup accounts" with weaker auth requirements, just to get the wizard through. That gap is now closing, albeit initially as a preview.

### Support for the France Sovereign Cloud

2.6.84.0 adds support for the France sovereign cloud environment, including Pass-through Authentication, Seamless Single Sign-On, password writeback, and Health Agent monitoring. Fittingly, a bug was fixed where the Application Proxy cloud name was not resolved correctly in the France cloud, causing PTA registration to fail with "EnvironmentName attribute is invalid". For Swiss environments this has no direct relevance, but it shows where things are heading: Microsoft is expanding its sovereign cloud offerings, and the hybrid components are following suit.

## Behavior Changes in Detail

The most interesting part of the release is not the new features but the changed behaviors. Several of them correct design decisions that caused surprises in practice.

### Auto-Upgrade No Longer Destroys Customized Configuration Files

This is the change with the longest history. Until now, auto-upgrade completely overwrote the `miiserver.exe.config` file during an update – manual customizations were lost. That sounds like an edge case, but it wasn't: Microsoft itself had instructed administrators in FIPS environments to edit exactly this file so that Password Hash Synchronization works with FIPS mode enabled. Anyone who followed the official guidance therefore had a "modified" configuration file.

The consequences showed up as a known issue when upgrading to 2.5.190.0 and 2.6.1.0: if the installer detects a modified `miiserver.exe.config`, it leaves the file untouched – but then the new assembly binding is missing, and the synchronization service dies after the upgrade with `System.IO.FileLoadException: Could not load file or assembly 'System.Diagnostics.DiagnosticSource, Version=6.0.0.1'`. The documented workaround: manually add a bindingRedirect to the `assemblyBinding` section of `miiserver.exe.config` (under `%programfiles%\Microsoft Azure AD Sync\Bin`):

```xml
<dependentAssembly>
  <assemblyIdentity name="System.Diagnostics.DiagnosticSource" publicKeyToken="cc7b13ffcd2ddd51" culture="neutral" />
  <bindingRedirect oldVersion="0.0.0.0-8.0.0.0" newVersion="8.0.0.0" />
</dependentAssembly>
```

Then restart the ADSync service. The 2.6.3.0 hotfix only defused the problem for auto-upgrade – affected servers were simply skipped and stayed on the old version. With 2.6.84.0 comes the actual solution: the upgrade process merges customer modifications with the new configuration and validates the result before applying it. If you are upgrading manually from an affected version, you should still check the state of your `miiserver.exe.config` beforehand and back up the file – the merge mechanism is new and therefore not yet proven in the field itself.

### Application-Based Authentication: No More Silent Fallback and No More Silent Migration

As a reminder: since 2.5.76.0, Application-Based Authentication (ABA) has been generally available and the default. Instead of the old Directory Synchronization Account – a cloud account with a stored password – the sync server authenticates as an Entra ID application with a certificate, ideally TPM-protected. That is the considerably more robust architecture: no password that can leak, and a credential bound to the machine.

2.6.84.0 cleans up two behaviors that undermined this security gain:

**No more silent fallback.** If ABA setup failed in the wizard, setup previously fell back to the legacy account without comment. The result: the administrator believed they had certificate-based authentication, while the server was actually running on the old password account. A classic fail-open pattern. Now the wizard stops with a clear error message ("Microsoft Entra Connect could not configure application-based authentication for this server. Setup cannot continue.") so the underlying cause gets fixed instead of hidden.

**No more automatic migration in the background.** Until now, Entra Connect switched existing servers from the legacy account to ABA on its own during regular sync operation. Well-intentioned from a security perspective, a nightmare from an operations perspective: an authentication method changes unasked, with no change window, without anyone knowing – and if something goes wrong in the process (TPM issues, Conditional Access conflicts, firewall), synchronization is down. The new rule: only new installations configure ABA automatically; existing servers switch only when an administrator runs the wizard and explicitly selects **Configure application-based authentication to Microsoft Entra ID**. The switch is thus back where it belongs – in a planned change.

In addition, TPM handling was improved: setup now tests a certificate's signing capability upfront and handles TPM signature verification correctly. On servers with non-conforming TPM firmware that cannot produce a valid signature, setup falls back to a software-based certificate in a controlled manner. This too has history: TPM-related ABA failures ran through several earlier releases (2.5.79.0, 2.5.190.0), among other things due to incompatibilities between TPM implementations and the MSAL library's default signing method.

### PowerShell Cmdlets Now Require an Explicit Admin Sign-In

A change that script operators need to have on their radar: the cmdlets `Set-ADSyncAADCompanyFeature` and `Set-ADSyncAADPasswordSyncState`, which modify cloud configuration, now require the `-AADUsername` parameter for interactive admin authentication. The wizard itself also no longer writes cloud changes with stored service credentials but via an interactive MSAL sign-in. And the uninstall wizard prompts for admin credentials to clean up the cloud configuration; if you skip that, only local cleanup proceeds.

The background is the same thread as with ABA: actions against the tenant should be attributable to a real, traceable administrator identity rather than an anonymous service account. This matches a bug fix in the same release: until now, admin audit logging recorded the service account's identity instead of the administrator actually performing the action when synchronization rules were changed – an audit trail that misses its purpose. Only both together produce usable auditing. The practical consequence: anyone who has been calling these cmdlets unattended in scripts needs to rework those procedures – interactive authentication and automation do not mix.

### PHS Self-Healing Removed

The most inconspicuous but conceptually most interesting change: Password Hash Synchronization no longer re-enables its cloud feature flag on its own in the background. If the flag is disabled, an administrator must explicitly re-enable it.

Until now: if PHS was disabled at the tenant level – deliberately or accidentally – the feature "healed" itself and switched back on. For environments that had deliberately disabled PHS (for compliance reasons, say, because no password hashes may flow to the cloud, or during a migration phase), this was a feature that overrode a documented administrator decision. That of all things a mechanism synchronizing password hashes would reactivate itself was a hard sell.

But the flip side should not be concealed: self-healing also rescued environments where the flag was disabled by a bug or a botched script – without anyone noticing. That safety net is now gone. If you use PHS in production (even if only as a fallback for emergency sign-in), you should actively monitor the PHS status going forward, for example via Entra Connect Health or by checking the synchronization heartbeat values.

### Updated Components: SQL LocalDB 2022, MSAL, VC++ Runtime

Less spectacular but overdue is the modernization of the bundled components:

- **SQL Server LocalDB 2019 → 2022.** Entra Connect's internal database was previously based on SQL Server 2019 Express LocalDB – a version whose mainstream support ended in February 2025. With SQL Server 2022, the installation is back on a version with active support.
- **MSAL 4.64.1 → 4.83.3.** The Microsoft Authentication Library is the central component for all token acquisition (ABA, wizard sign-in, PowerShell). The jump across roughly twenty minor versions brings the library's accumulated fixes and improvements.
- **Visual C++ Redistributable 2013 → 2015–2022 (14.42).** What is remarkable here is less the update than the legacy: up to this release, Entra Connect required a runtime whose support expired in April 2024. The VC++ 2013 dependency has now been removed entirely.

This matches the blanket note in the release notes that "multiple security vulnerabilities in bundled third-party dependencies" were fixed. That is likely the main reason for the security-release classification – outdated bundled components are no cosmetic issue for a product running with near-domain-admin privileges at the center of your identity infrastructure.

## The Remaining Bug Fixes

For completeness, the remaining fixes:

- **Metaverse search in the Synchronization Service Manager** repaired. After the warning not to use the UI at all in older versions, it is evidently being maintained again.
- **PowerShell diagnostics report (HTML)** renders correctly again – relevant for anyone using `Invoke-ADSyncDiagnostics` for support cases.
- **Generic SQL connector:** profile creation failed because required parameters were not populated during configuration. Affects environments that connect additional directories via the GSQL connector.
- **China cloud:** the instance name was not resolved correctly by the Discovery Endpoint API, which could cause cloud instance detection to fail.
- **Admin audit logging** now records the actual administrator instead of the service account when synchronization rules are changed (see above).

## Support Deadlines: Who Has to Act Now Regardless

Since March 2023, a strict retirement policy applies to Entra Connect Sync 2.x: each version falls out of support twelve months after the release of its successor. The current deadlines:

| Version | End of support |
| --- | --- |
| 2.5.3.0 | **July 31, 2026** |
| 2.5.76.0 | September 1, 2026 |
| 2.5.79.0 | October 23, 2026 |
| 2.5.190.0 | February 2, 2027 |
| 2.6.1.0 | March 10, 2027 |
| 2.6.3.0 | July 7, 2027 |

If you are still on 2.5.3.0, you have only two weeks of support left – here the question is not whether to update but only to which version. Microsoft also emphasizes that retired versions might stop working "unexpectedly"; for the discontinued 1.x versions, synchronization has in fact been switched off server-side by now. The minimum requirements remain .NET Framework 4.7.2 and TLS 1.2; the installer is available exclusively in the Entra admin center (Entra ID → Entra Connect → Get started), no longer in the Download Center.

## Recommendation: Wait in a Controlled Manner Instead of a Day-One Upgrade

Microsoft recommends upgrading "as soon as possible". However, that recommendation stood word for word above version 2.6.79.0 as well – the version that was subsequently recalled. The recent release history (recalled installer, hotfix for stopped servers, UI warnings across several versions) justifies a sober assessment rather than a reflex.

My assessment for typical environments:

**Waiting a few weeks is defensible** if you are running a still-supported version (2.5.190.0 or newer), none of the fixed issues affects you acutely, and none of the new features is needed. According to the release notes, the fixed security vulnerabilities sit in bundled third-party components; an Entra Connect server should be locked down anyway (no internet access except to the Microsoft endpoints, no interactive sign-ins, tier-0 treatment), so the time window is justifiable. If the version remains available for a few weeks without a recall and Microsoft starts the auto-upgrade rollout, that is a much better quality signal than any announcement.

**You should act quickly** if any of these points applies:

- **You have 2.6.79.0 installed.** Then the instruction is unambiguous: uninstall and install 2.6.84.0 – do not wait.
- **You are running 2.5.3.0** (end of support July 31, 2026) or an even older, already retired version.
- **One of the fixed issues affects you concretely** – for example ABA setup on TPM servers, the GSQL connector, or the audit requirement that rule changes be attributed to the correct administrator.

For the upgrade itself, the usual procedure applies, and with this release history it is particularly advisable: export the configuration beforehand (the wizard offers **View or export current configuration**), install the update on a staging-mode server first and verify sync cycles, wizard, and certificate rotation there, and only then the active server. If you have a customized `miiserver.exe.config`, back it up before the update and check afterwards whether the new merge mechanism carried your modifications over correctly. And if you operate scripts using `Set-ADSyncAADCompanyFeature` or `Set-ADSyncAADPasswordSyncState`, test them before the production rollout – otherwise they will break on the new mandatory parameter.

## Sources

1.  [Microsoft Entra Connect: Version release history – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history) — Official release notes for 2.6.84.0 including the recall notice for 2.6.79.0, the retirement table, and the known issue with a modified miiserver.exe.config.

2.  [Microsoft Entra Connect: Upgrade from a previous version to the latest – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-upgrade-previous-version) — Upgrade procedures including swing migration via a staging-mode server.

3.  [Authenticate to Microsoft Entra ID by using application identity – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/authenticate-application-id) — How Application-Based Authentication works, replacing the legacy service account.

4.  [Microsoft Entra Connect: Phishing-resistant authentication – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-passwordless-authentication) — The new passkey/FIDO2 sign-in in the setup wizard via the Windows Web Account Manager.

5.  [Microsoft Entra Connect: Automatic upgrade – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-install-automatic-upgrade) — The mechanics and prerequisites of auto-upgrade, whose rollout for 2.6.84.0 is still pending.

6.  [Auditing administrator events in Microsoft Entra Connect Sync – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/admin-audit-logging) — Admin audit logging, whose identity attribution for synchronization rules was corrected in this release.

7.  [SQL Server 2019 – Microsoft Lifecycle](https://learn.microsoft.com/en-us/lifecycle/products/sql-server-2019) — Support dates for the previously bundled LocalDB base, whose mainstream support ended in February 2025.
