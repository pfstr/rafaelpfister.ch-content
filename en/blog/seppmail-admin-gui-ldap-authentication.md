---
title: "Connect the SEPPmail Admin GUI to Active Directory: Set Up LDAP Authentication from 15.0.6"
navTitle: "Admin LDAP Login"
description: "Since firmware version 15.0.6, administrators of the SEPPmail appliance can authenticate against an external LDAP server such as Active Directory, including group mapping to the local admin group. Setup under User > Advanced Settings, step by step."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "7 min read"
themen:
  - seppmail
slug: "seppmail-admin-gui-ldap-authentication"
translationOf: "seppmail-admin-gui-ldap-authentifizierung"
draft: false
translationId: article-21092a3dad6b84cb
translationSourceHash: b9a1bdfe75f57458eb57c2e1dcb32fdfd25e1279eb720d4cd4754d49b5aac624
translatedAt: 2026-10-31T03:23:06.084Z
translationReview: automatic
translationModel: gpt-5.6-terra
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/seppmail-admin-gui-ldap-authentication
---

# Connect the SEPPmail Admin GUI to Active Directory: Set Up LDAP Authentication from 15.0.6

Up to firmware version 15.0.5, the administration interface of the SEPPmail Secure E-Mail Gateway supported only local accounts. Those who wanted to manage things properly created a separate local user for every administrator and added it to the admin group. This works, but has the usual drawbacks of local accounts: separate passwords for each appliance, no centralized offboarding, and no enforcement of password policies from the directory service. That changes with patch release 15.0.6. The Admin GUI can optionally authenticate administrators against an external LDAP server such as Active Directory and map AD groups to local appliance groups.

The remaining release changes are summarized in the article on [SEPPmail 15.0.6 and 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1). This article covers only the new external authentication.

## What the feature does

According to the Extended Release Notes, version 15.0.6 adds a new **External Authentication** section under **User > Advanced Settings**. This lets the Admin GUI authenticate against an external LDAP server and map external groups, such as AD security groups, to local appliance groups.

Externally authenticated users appear locally on the appliance and behave like local users, with one difference: their password cannot be changed on the appliance because it resides on the external LDAP server. Password authority therefore moves entirely to the directory.

An important distinction: the appliance already supported external authentication, but only for the GINA web interface, configured per Managed Domain (the External authentication section in the domain configuration). What is new in 15.0.6 is that access to the administration interface itself can also use LDAP.

I am still testing whether the HIN Mailgateway has also received LDAP login and will update the article afterward. Since HIN appliances are based on the same SEPPmail firmware, I assume so.

## Prerequisites

Three things should be in place before setup:

- **Firmware 15.0.7 or later:** The feature was introduced in 15.0.6. Since September 2, 2026, patch release 15.0.7 has been available, fixing, among other things, an MFA requirement bypass (SEC-100) and a command injection in configuration processing (CVE-2026-84830). The appliance should run at least this version; the 15.1 feature branch (currently hotfix 15.1.0.1 from September 29, 2026) also includes the feature.
- **An LDAP-capable directory:** Active Directory, OpenLDAP, or equivalent. If users exist only in Entra ID, which does not itself speak LDAP, [Microsoft Entra Domain Services](/blog/microsoft-entra-domain-services-ldap-kerberos) provides the bridge.
- **A bind account in the directory:** A dedicated, unprivileged service account with read access that the appliance uses to perform LDAP searches. Not a domain administrator.
- **An AD group for gateway administrators:** For example, a security group named SEPPmail-Admins, which is later mapped to the local admin group. Membership in this group then determines full administrative access.

TLS is enabled by default in the connection settings and should remain enabled; administrator credentials do not belong on the network unencrypted. The appliance must be able to reach the LDAP server on the configured port (typically 636 for LDAPS).

## Setup under User > Advanced Settings

The configuration is located in the Admin GUI under **User > Advanced Settings**, in the **External Authentication** section, and consists of four blocks.

**1. Connection Settings:** The *Authenticate users to external LDAP server (e.g. Active Directory)* checkbox enables the feature. This is followed by the server address, port, the *TLS required* option, and the Bind DN and Bind Password for the service account.

**2. User Attributes:** This defines how the appliance finds user objects: the LDAP Object Class (typically person for Active Directory), the Search Base (the OU or container containing administrator accounts), and the email attribute (default: mail).

**3. Group Attributes:** Similarly, these settings specify group objects so that the appliance can resolve group memberships.

**4. Mapping Settings:** The crucial part. Under *Remote Group*, select the group from the LDAP server; under *Local Group*, select one or more local groups to which it is mapped. For full administrative access, this is the admin group; its members have the same privileges as the default admin user. For more granular access, map instead to restricted groups such as readonly admin or to appliance groups associated with specific functions.

Before saving, it is worth using the built-in **Login Test**: enter the username and password of a test account to verify that the connection, search, and authentication work before the configuration becomes active.

## Example configurations

The following values must be adapted to your own environment (example domain: example.com). The field names correspond to the appliance's External Authentication section.

### Active Directory

| Field | Value |
|---|---|
| Server | dc01.example.com |
| Port | 636 |
| TLS required | enabled |
| Bind DN | CN=svc-seppmail,OU=ServiceAccounts,DC=example,DC=com |
| Bind Password | Service account password |
| User: LDAP Object Class | person |
| User: Search Base | OU=IT,DC=example,DC=com |
| User: E-Mail Attribute | mail |
| Group: LDAP Object Class | group |
| Group: Search Base | OU=Groups,DC=example,DC=com |
| Mapping: Remote Group | SEPPmail-Admins |
| Mapping: Local Group | admin |

Notes on Active Directory: Any reachable domain controller can serve as the server; in environments with multiple locations, a DC at the same location or an alias pointing to multiple DCs is recommended. Port 636 is LDAPS; the appliance must be able to validate the DC certificate. The Search Base should be narrow enough to include the administrator accounts without covering the entire directory. The mail attribute must be populated on the AD accounts.

### OpenLDAP

| Field | Value |
|---|---|
| Server | ldap01.example.com |
| Port | 636 |
| TLS required | enabled |
| Bind DN | cn=seppmail,ou=services,dc=example,dc=com |
| Bind Password | Service account password |
| User: LDAP Object Class | inetOrgPerson |
| User: Search Base | ou=people,dc=example,dc=com |
| User: E-Mail Attribute | mail |
| Group: LDAP Object Class | groupOfNames |
| Group: Search Base | ou=groups,dc=example,dc=com |
| Mapping: Remote Group | seppmail-admins |
| Mapping: Local Group | admin |

Notes on OpenLDAP: In typical setups, users are stored as inetOrgPerson objects under ou=people. For groups, groupOfNames is the reliable choice because membership is represented there through the member attribute using the full DN. By contrast, posixGroup groups list members only as memberUid (username rather than DN); whether the appliance resolves this is undocumented and should be verified with the Login Test before switching. If the server supports only STARTTLS on port 389, enter the corresponding port in the Server field; the connection should never be unencrypted.

## Operational notes

Three points deserve attention before LDAP login becomes the only way to access the appliance:

- **Keep local emergency access.** External users' passwords reside on the LDAP server. If the directory is unavailable (due to a network problem, AD maintenance, or because the gateway is meant to fix a problem with that very network), a local administrator account with a securely stored password is still needed. The default admin user should therefore not be removed, but maintained as a documented emergency access method.
- **MFA remains relevant.** Version 15.0.6 also revised MFA login: the second factor is no longer appended to the password, but requested in a separate field. External authentication does not replace the second factor. A vulnerability that could, under certain conditions, allow access before all MFA steps had been completed was not fixed until 15.0.7.
- **Offboarding through the directory.** This is the real benefit of the integration: when an administrator leaves the company, disabling the AD account or removing it from the mapped group is sufficient. There is no longer any need to maintain local accounts on every appliance. The locally visible, externally authenticated user objects should nevertheless be periodically reconciled with the directory.

## Conclusion

LDAP authentication for the Admin GUI closes a long-standing gap in the appliance: administrator access can now be managed centrally in the directory rather than per device. Together with the separate MFA field, version 15.0.6 significantly improves login to the administration interface in a single release. Those introducing the feature should keep group mapping deliberately restrictive and retain local emergency access.

## Change history

- **October 25, 2026:** Raised the recommended minimum version from 15.0.6.1 to 15.0.7, which fixes an MFA requirement bypass (SEC-100) and CVE-2026-84830. Added: the 15.1 feature branch (currently 15.1.0.1) also includes the feature.

## Sources

1.  [SEPPmail Extended Release Notes 15.0](https://downloads.seppmail.com/extrelnotes/150/ERN15.0.html): Entry on Admin GUI authentication with a feature description, configuration location, and the behavior of externally authenticated users; security fixes in 15.0.7 (SEC-100, CVE-2026-84830).

2.  [SEPPmail documentation – “User > Advanced Settings”](https://docs.seppmail.com/ch/07_mi_15_usr_04_advanced-settings.html): Reference for the fields in the External Authentication section (Connection, User Attributes, Group Attributes, Mapping, Login Test).

3.  [SEPPmail documentation – “Groups”](https://docs.seppmail.com/ch/07_mi_16_groups.html): Predefined appliance groups; members of the admin group have unrestricted administrative access.

4.  [SEPPmail documentation – “Revision History”](https://docs.seppmail.com/ch/20_revision-history.html): Official release notes for 15.0.6 containing the entry on Admin GUI authentication against external LDAP servers, as well as subsequent releases 15.0.7, 15.1.0, and 15.1.0.1.

5.  [SEPPmail 15.0.6 and 15.0.6.1: Security fixes and new admin features](/blog/seppmail-releases-15-0-6-und-15-0-6-1): Overview of all changes in both releases.
