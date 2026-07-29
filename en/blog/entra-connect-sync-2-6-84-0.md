---
title: "Entra Connect Sync 2.6.84.0: What is changing and who should update now"
navTitle: "Entra Connect 2.6.84"
description: "The security release brings passkey support and changes to application authentication, PowerShell and Password Hash Sync. The previous version was withdrawn, so the update requires a phased decision."
date: "2026-07-17"
kategorie: "Microsoft Entra"
timeToRead: "11 min read"
themen:
  - microsoft-entra
  - active-directory-entra
slug: "entra-connect-sync-2-6-84-0"
translationOf: "entra-connect-2-6-84-0"
draft: false
url: "https://rafaelpfister.ch/en/blog/entra-connect-sync-2-6-84-0"
translationId: article-85bd27acb917e406
translatedAt: 2026-07-28T11:10:30.445Z
translationReview: automatic
translationSourceHash: e4dc8f6498301c03d85afdba4b310d0af7ba497f7ee781448e2d02e5c62d26d9
---

# Entra Connect Sync 2.6.84.0: What is changing and who should update now

Microsoft released Entra Connect Sync 2.6.84.0 as a security release on 7 July 2026 and recommends upgrading promptly. At the same time, the direct predecessor, 2.6.79.0, was withdrawn because of an installer issue discovered afterwards. The consequence is neither ‘install it everywhere immediately’ nor ‘wait and ignore it’: affected systems and those soon to fall out of support should move quickly, while everyone else can test the update in a controlled manner first.

## Why this release warrants particular caution

The 2.6 branch of Entra Connect Sync has had a bumpy start. A brief look back, because it is relevant to the update decision:

- **2.6.1.0** (February 2026) fixed, among other things, an issue where editing the Entra ID connector configuration in Synchronization Service Manager deleted the Application-Based Authentication parameters, causing the wizard and certificate rotation to fail. All 2.5 versions were therefore subject to the remarkable recommendation simply not to use the product’s management interface.
- **2.6.3.0** (March 2026) was a hotfix for an issue where auto-upgrade could unexpectedly stop the Entra Connect server. The workaround at the time: auto-upgrade detects manually modified configuration files and simply skips those servers.
- **2.6.79.0** (June 2026) was completely withdrawn after release. The installer is no longer available; according to Microsoft, anyone who installed the version should uninstall it and install 2.6.84.0. Microsoft does not document exactly what the issue was.

As of today, version 2.6.84.0 is available only for download through the Microsoft Entra admin centre (‘Released for download’). An auto-upgrade rollout has not yet been announced. That too is a signal: Microsoft itself is not yet distributing the version across existing installations on a broad scale.

## New features

### Phishing-resistant sign-in in the setup wizard (Preview)

The setup wizard now supports signing in with passkeys and FIDO2 security keys through Windows Web Account Manager (WAM). The background is that Microsoft has been progressively enforcing MFA for sign-ins to Azure and Entra management interfaces since 2024/2025, and many organisations have restricted their admin accounts to phishing-resistant methods (FIDO2, passkeys, certificate-based authentication) through Conditional Access. Until now, precisely these properly secured accounts could not sign in to the Entra Connect wizard because the embedded sign-in dialogue did not support the methods. In practice, this led to unattractive workarounds, such as dedicated ‘setup accounts’ with weaker authentication requirements just to get the wizard through. This gap is now being closed, albeit initially as a preview.

### Support for the French Sovereign Cloud

2.6.84.0 adds support for the French Sovereign Cloud environment, including Pass-through Authentication, Seamless Single Sign-On, Password Writeback and Health Agent monitoring. In line with this, an issue was fixed where the Application Proxy cloud name in the France Cloud was not resolved correctly and PTA registration failed with ‘EnvironmentName attribute is invalid’.

## Behavioural changes in detail

The most interesting part of the release is not the new features, but the changed behaviours. Several of them correct design decisions that have caused surprises in practice.

### Auto-upgrade no longer destroys customised configuration files

This is the change with the longest history. Until now, auto-upgrade completely overwrote the `miiserver.exe.config` file during an update. Manual changes were lost. That may sound like an edge case, but it was not: Microsoft itself had instructed administrators in FIPS environments to edit this exact file so that Password Hash Synchronization works with FIPS mode enabled. Anyone following the official guidance therefore had a ‘modified’ configuration file.

The consequences became apparent during upgrades to 2.5.190.0 and 2.6.1.0 as a known issue: if the installer detects a modified `miiserver.exe.config`, it leaves the file untouched; however, the new assembly binding is then missing, and the synchronisation service dies after the upgrade with `System.IO.FileLoadException: Could not load file or assembly 'System.Diagnostics.DiagnosticSource, Version=6.0.0.1'`. The documented workaround: manually add a bindingRedirect in the `assemblyBinding` section of `miiserver.exe.config` (under `%programfiles%\Microsoft Azure AD Sync\Bin`):

```xml
<dependentAssembly>
  <assemblyIdentity name="System.Diagnostics.DiagnosticSource" publicKeyToken="cc7b13ffcd2ddd51" culture="neutral" />
  <bindingRedirect oldVersion="0.0.0.0-8.0.0.0" newVersion="8.0.0.0" />
</dependentAssembly>
```

Then restart the ADSync service. Hotfix 2.6.3.0 mitigated the issue only for auto-upgrade: affected servers were simply skipped and remained on the old version. With 2.6.84.0 comes the actual solution: the upgrade process merges customer customisations with the new configuration and validates the result before applying it. Anyone manually upgrading from an affected version should nevertheless check the state of their `miiserver.exe.config` beforehand and back up the file: the merge mechanism is new and has therefore not yet been proven in practice itself.

### Application-Based Authentication: no more silent fallback or silent migration

As a reminder: since 2.5.76.0, Application-Based Authentication (ABA) has been generally available and is the default. Rather than using the old Directory Synchronization Account (a cloud account with a stored password), the sync server authenticates as an Entra ID application using a certificate, ideally protected by the TPM. This is a much more robust architecture: no password that can leak, and a credential tied to the machine.

2.6.84.0 addresses two behaviours that have undermined this security gain:

**No more silent fallback.** If ABA setup failed in the wizard, setup previously fell back to the legacy account without comment. The result: the administrator believed they had certificate-based sign-in, while the server was actually running with the old password account. A classic fail-open pattern. The wizard now stops with a clear error message (‘Microsoft Entra Connect could not configure application-based authentication for this server. Setup cannot continue.’), so that the root cause is fixed rather than concealed.

**No more automatic background migration.** Previously, Entra Connect automatically migrated existing servers from the legacy account to ABA while sync operations were running. Well intended from a security perspective, but an operational nightmare: an authentication method changes unprompted, without a change window and without anyone knowing. And if something goes wrong (TPM issues, Conditional Access conflicts, firewall), synchronisation stops. Going forward, only new installations configure ABA automatically; existing servers change only when an administrator starts the wizard and explicitly selects **Configure application-based authentication to Microsoft Entra ID**. The migration is therefore back where it belongs: in a planned change.

In addition, TPM handling has been improved: setup now tests a certificate’s signing capability in advance and handles TPM signature verification correctly. On servers with faulty TPM firmware that cannot generate a valid signature, setup falls back in a controlled manner to a software-based certificate. This too has a history: TPM-related ABA failures occurred across several earlier releases (2.5.79.0, 2.5.190.0), including because of incompatibilities between TPM implementations and the standard signing method used by the MSAL library.

### PowerShell cmdlets now require an explicit admin sign-in

A change that script operators need to be aware of: the cmdlets `Set-ADSyncAADCompanyFeature` and `Set-ADSyncAADPasswordSyncState`, which change cloud configuration, now require the `-AADUsername` parameter for interactive admin authentication. The wizard itself also no longer writes cloud changes using stored service credentials, but through an interactive MSAL sign-in. And the uninstall wizard requests admin credentials to clean up cloud configuration; if this is skipped, only local cleanup takes place.

The background follows the same theme as ABA: actions against the tenant should be attributable to a real, traceable administrator identity rather than an anonymous service account. This aligns with a bug fix in the same release: previously, admin audit logging recorded the service account identity rather than that of the administrator who actually made changes to synchronisation rules: an audit trail that fails its purpose. Only together do the two changes provide usable auditing. The practical consequence: anyone who previously called these cmdlets unattended in scripts must redesign those processes; interactive authentication and automation do not mix.

### PHS self-healing removed

The most inconspicuous, but conceptually most interesting change: Password Hash Synchronization no longer reactivates its cloud feature flag automatically in the background. If the flag is disabled, an administrator must explicitly enable it again.

Previously, if PHS was disabled at tenant level (deliberately or accidentally), the feature ‘healed’ itself and switched back on. For environments that had intentionally disabled PHS (for example, for compliance reasons because password hashes must not flow to the cloud, or during a migration phase), this was a feature that overrode a documented administrator decision. The fact that a mechanism synchronising password hashes reactivated itself autonomously was difficult to justify.

However, the downside should not be concealed: self-healing also saved environments where the flag was disabled by an error or a failed script without anyone noticing. That safeguard is now gone. Anyone using PHS in production, even only as a fallback for emergency sign-in, should actively monitor PHS status in future, for example through Entra Connect Health or by checking synchronisation heartbeat values.

### Updated components: SQL LocalDB 2022, MSAL, VC++ runtime

Less spectacular but overdue is the modernisation of the bundled components:

- **SQL Server LocalDB 2019 → 2022.** Entra Connect’s internal database was previously based on SQL Server 2019 Express LocalDB, a version whose mainstream support ended in February 2025. With SQL Server 2022, the installation is once again on a version with active support.
- **MSAL 4.64.1 → 4.83.3.** Microsoft Authentication Library is the core component for all token acquisition (ABA, wizard sign-in, PowerShell). The jump of around twenty minor versions brings accumulated fixes and improvements from the library.
- **Visual C++ Redistributable 2013 → 2015–2022 (14.42).** What is notable here is less the update than the legacy dependency: until this release, Entra Connect required a runtime environment whose support expired in April 2024. The VC++ 2013 dependency has now been completely removed.

This fits with the general note in the release notes that ‘multiple security vulnerabilities in bundled third-party dependencies’ have been fixed. That is likely the main reason for classification as a security release: outdated bundled components are not a cosmetic issue in a product running with near-Domain Admin rights at the centre of identity infrastructure.

## The remaining bug fixes

For completeness, the remaining corrections:

- **Metaverse search in Synchronization Service Manager** fixed. After the warning not to use the interface at all in earlier versions, it now appears to be maintained again.
- **PowerShell diagnostic report (HTML)** renders correctly again; relevant to anyone using `Invoke-ADSyncDiagnostics` for support cases.
- **Generic SQL Connector:** Profile creation failed because mandatory parameters were not populated during configuration. This affects environments connecting additional directories through the GSQL connector.
- **China Cloud:** The instance name was not resolved correctly by the discovery endpoint API, which could cause cloud instance detection to fail.
- **Admin audit logging** now records the actual administrator rather than the service account for changes to synchronisation rules (see above).

## Support deadlines: who still needs to act now

Since March 2023, Entra Connect Sync 2.x has followed a strict retirement policy: each version falls out of support twelve months after the successor version is released. The current deadlines are:

| Version | End of support |
| --- | --- |
| 2.5.3.0 | **31 July 2026** |
| 2.5.76.0 | 1 September 2026 |
| 2.5.79.0 | 23 October 2026 |
| 2.5.190.0 | 2 February 2027 |
| 2.6.1.0 | 10 March 2027 |
| 2.6.3.0 | 7 July 2027 |

Anyone still running 2.5.3.0 therefore has only two weeks of support remaining. The question here is not whether to update, but only to which version. Microsoft also stresses that unsupported versions can ‘unexpectedly’ stop working; for retired 1.x versions, synchronisation has in fact since been disabled server-side. The minimum requirements remain .NET Framework 4.7.2 and TLS 1.2; the installer is available exclusively in the Entra admin centre (Entra ID → Entra Connect → Get started), no longer in the Download Centre.

## Recommendation by starting version

Microsoft recommends updating ‘as soon as possible’. However, this exact recommendation also accompanied version 2.6.79.0, the version that was subsequently withdrawn. The recent release history (withdrawn installer, hotfix due to stopped servers, UI warnings across several versions) justifies a sober assessment rather than a reflex.

My assessment for typical environments:

**Waiting a few weeks is justifiable** if you are running a still-supported version (2.5.190.0 or later), none of the fixed issues affects you urgently, and none of the new features is needed. Based on the release notes, the fixed security vulnerabilities are in bundled third-party components; an Entra Connect server should in any case be sufficiently isolated (no internet access except to Microsoft endpoints, no interactive sign-ins, Tier 0 treatment) for this time window to be defensible. If the version remains available without being recalled for a few weeks and Microsoft starts the auto-upgrade rollout, that is a much better quality signal than any announcement.

**You should act promptly** if any of these points applies:

- **You have 2.6.79.0 installed.** The instruction is then clear: uninstall it and install 2.6.84.0; do not wait.
- **You are running 2.5.3.0** (end of support 31 July 2026) or an even older version that has already expired.
- **One of the fixed issues affects you specifically**, such as ABA setup on TPM servers, the GSQL connector, or the audit requirement that rule changes are attributed to the correct administrator.

For the upgrade itself, the usual approach applies, and is particularly advisable given this release history: export the configuration beforehand (the wizard offers **View or export current configuration**), deploy the update first to a staging-mode server and test sync cycles, the wizard and certificate rotation there, then proceed with the active server. Anyone with a customised `miiserver.exe.config` should back it up before the update and check afterwards whether the new merge mechanism has correctly incorporated the customisations. And anyone running scripts with `Set-ADSyncAADCompanyFeature` or `Set-ADSyncAADPasswordSyncState` should test them before the production rollout; otherwise, they will fail on the new mandatory parameter.

## Sources

1. [Microsoft Entra Connect: Version release history – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history): Official release notes for 2.6.84.0, including the recall notice for 2.6.79.0, retirement table and the known issue with modified miiserver.exe.config.
1. [Microsoft Entra Connect: Upgrade from a previous version to the latest – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-upgrade-previous-version): Upgrade procedure, including swing migration via a staging-mode server.
1. [Authenticate to Microsoft Entra ID by using application identity – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/authenticate-application-id): How Application-Based Authentication works, replacing the legacy service account.
1. [Microsoft Entra Connect: Phishing-resistant authentication – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-passwordless-authentication): The new passkey/FIDO2 sign-in in the setup wizard via Windows Web Account Manager.
1. [Microsoft Entra Connect: Automatic upgrade – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-install-automatic-upgrade): The mechanics and requirements of auto-upgrade, whose rollout for 2.6.84.0 is still pending.
1. [Auditing administrator events in Microsoft Entra Connect Sync – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/admin-audit-logging): Admin audit logging, whose identity attribution for synchronisation rules was corrected in this release.
1. [SQL Server 2019 – Microsoft Lifecycle](https://learn.microsoft.com/en-us/lifecycle/products/sql-server-2019): Support dates for the previously bundled LocalDB base, whose mainstream support ended in February 2025.
