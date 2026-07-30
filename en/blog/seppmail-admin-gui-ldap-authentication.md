---
title: "Connecting the SEPPmail Admin GUI to Active Directory: Setting up LDAP authentication from 15.0.6"
navTitle: "Admin LDAP login"
description: "Since firmware 15.0.6, administrators of the SEPPmail appliance can authenticate against an external LDAP server such as Active Directory, including group mapping to the local admin group. A step-by-step setup guide under User > Advanced Settings."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "7 min read"
themen:
  - seppmail
slug: "seppmail-admin-gui-ldap-authentication"
translationOf: "seppmail-admin-gui-ldap-authentifizierung"
draft: false
translationId: article-21092a3dad6b84cb
translationSourceHash: bb8386d1f880934d4811eb317bcd51d47900fdd493dad90b1d7752bfc25ba55c
translatedAt: 2026-07-30T12:23:42.226Z
translationReview: automatic
url: https://rafaelpfister.ch/en/blog/seppmail-admin-gui-ldap-authentication
translationModel: gpt-5.6-terra
---

# Connecting the SEPPmail Admin GUI to Active Directory: Setting up LDAP authentication from 15.0.6

Until firmware 15.0.5, the administration interface of the SEPPmail Secure E-Mail Gateway supported local accounts only. Those wishing to manage it properly created a separate local user for each administrator and added them to the admin group. This works, but comes with the usual disadvantages of local accounts: separate passwords for each appliance, no central offboarding, and no enforcement of password policies from the directory service. This changes with patch release 15.0.6. The Admin GUI can now authenticate administrators against an external LDAP server such as Active Directory and map AD groups to local appliance groups.

The other changes in the release are summarised in the article on [SEPPmail 15.0.6 and 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1). This article covers only the new external authentication.

## What the feature does

According to the Extended Release Notes, 15.0.6 adds a new **External Authentication** section under **User > Advanced Settings**. This enables the Admin GUI to authenticate against an external LDAP server and maps external groups (such as AD security groups) to local appliance groups.

Externally authenticated users appear locally on the appliance and behave like local users, with one difference: their password cannot be changed on the appliance because it is stored on the external LDAP server. Password control therefore moves entirely to the directory.

An important distinction: the appliance already supported external authentication, but only for the GINA web interface, configured per Managed Domain (the External authentication section in the domain configuration). New in 15.0.6 is that access to the administration interface itself can also use LDAP.

I am still testing whether the HIN Mailgateway has also received LDAP login and will update this article afterwards. As HIN appliances are based on the same SEPPmail firmware, I assume it has.

## Prerequisites

Three things should be in place before configuration:

- **Firmware 15.0.6.1:** The feature arrives with 15.0.6; because of the release's two RuleEngine errors, the 15.0.6.1 hotfix is the right choice straight away.
- **An LDAP-capable directory:** Active Directory, OpenLDAP or equivalent. If users exist only in Entra ID, which does not itself speak LDAP, [Microsoft Entra Domain Services](/blog/microsoft-entra-domain-services-ldap-kerberos) provides the bridge.
- **A bind account in the directory:** A dedicated, unprivileged service account with read access, which the appliance uses for LDAP searches. Not a domain administrator.
- **An AD group for gateway administrators:** For example, a SEPPmail-Admins security group, which is subsequently mapped to the local admin group. Membership of this group then determines full administrative access.

TLS is enabled by default in the connection settings and should remain so; administrators' credentials do not belong on the network unencrypted. The appliance must be able to reach the LDAP server on the configured port (usually 636 for LDAPS).

## Setup under User > Advanced Settings

The configuration is located in the Admin GUI under **User > Advanced Settings**, in the **External Authentication** section, and consists of four blocks.

**1. Connection Settings:** The *Authenticate users to external LDAP server (e.g. Active Directory)* checkbox enables the feature. This is followed by the server address, port, the *TLS required* option, and the Bind DN and Bind Password for the service account.

**2. User Attributes:** This defines how the appliance finds user objects: the LDAP Object Class (typically person for Active Directory), the Search Base (the OU or container containing the administrator accounts), and the email attribute (default: mail).

**3. Group Attributes:** Similarly, these settings define group objects so that the appliance can resolve group memberships.

**4. Mapping Settings:** This is the crucial part. Under *Remote Group*, select the group from the LDAP server; under *Local Group*, select one or more local groups to which it is mapped. For full administrative access, this is the admin group; its members are equivalent to the default admin user. For more granular access, map instead to restricted groups such as readonly admin or to function-specific appliance groups.

Before saving, it is worth using the built-in **Login Test**: enter the username and password of a test account to verify that connection, search and authentication work before the configuration becomes active.

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

Notes on Active Directory: any reachable domain controller is suitable as the server; in environments with multiple sites, a DC at the same site or an alias pointing to several DCs is recommended. Port 636 is LDAPS; its DC certificate must be capable of validation by the appliance. The Search Base should be narrow enough to contain the administrator accounts without covering the entire directory. The mail attribute must be populated on the AD accounts.

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

Notes on OpenLDAP: in typical setups, users are stored as inetOrgPerson under ou=people. For groups, groupOfNames is the reliable choice because membership is represented through the member attribute using the full DN. By contrast, posixGroup groups list their members only as memberUid (a username rather than a DN); whether the appliance resolves this is not documented and should be checked with the Login Test before switching over. If the server uses STARTTLS only on port 389, enter the appropriate port in the Server field; the connection should never run unencrypted.

## Operational notes

Three points deserve attention before LDAP login becomes the only way into the appliance:

- **Keep local emergency access.** External users' passwords are stored on the LDAP server. If the directory is unavailable (because of a network issue, AD maintenance, or because the gateway is intended to resolve a problem with that very network), a local administrator account with a securely stored password is still required. The default admin user should therefore not be removed, but retained as a documented emergency access account.
- **MFA remains relevant.** 15.0.6 also revised MFA login: the second factor is no longer appended to the password, but requested in a separate field. External authentication does not replace the second factor.
- **Offboarding through the directory.** This is the real benefit of the integration: when an administrator leaves the organisation, it is sufficient to disable their AD account or remove them from the mapped group. The previously necessary maintenance of local accounts on every appliance is no longer required. However, the locally visible externally authenticated user objects should still be periodically reconciled with the directory.

## Conclusion

LDAP authentication for the Admin GUI closes a long-standing gap in the appliance: administrator access can now be managed centrally in the directory instead of on a per-device basis. Together with the separate MFA field, 15.0.6 makes logging into the administration interface significantly more mature in a single release. Anyone introducing the feature should keep group mapping deliberately restrictive and not sacrifice local emergency access.

## Sources

1.  [SEPPmail Extended Release Notes 15.0](https://downloads.seppmail.com/extrelnotes/150/ERN15.0.html): Entry on Admin GUI authentication with a feature description, configuration location and the behaviour of externally authenticated users.

2.  [SEPPmail documentation – «User > Advanced Settings»](https://docs.seppmail.com/ch/07_mi_15_usr_04_advanced-settings.html): Reference for the fields in the External Authentication section (Connection, User Attributes, Group Attributes, Mapping, Login Test).

3.  [SEPPmail documentation – «Groups»](https://docs.seppmail.com/ch/07_mi_16_groups.html): Predefined appliance groups; members of the admin group have unrestricted administrative access.

4.  [SEPPmail documentation – «Revision History»](https://docs.seppmail.com/ch/20_revision-history.html): Official release notes for 15.0.6, including the entry on Admin GUI authentication against external LDAP servers.

5.  [SEPPmail 15.0.6 and 15.0.6.1: Security fixes and new admin features](/blog/seppmail-releases-15-0-6-und-15-0-6-1): Overview of all changes in the two releases.
