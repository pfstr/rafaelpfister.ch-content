---
title: "Connecting the SEPPmail Admin GUI to Active Directory: Setting Up LDAP Authentication in 15.0.6"
navTitle: "Admin LDAP login"
description: "Since firmware 15.0.6, administrators of the SEPPmail appliance can authenticate against an external LDAP server such as Active Directory, including group mapping to the local admin group. The setup under User > Advanced Settings, step by step."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "7 min to read"
themen:
  - seppmail
slug: "seppmail-admin-gui-ldap-authentication"
translationOf: "seppmail-admin-gui-ldap-authentifizierung"
url: "https://rafaelpfister.ch/en/blog/seppmail-admin-gui-ldap-authentication"
draft: false
translationId: article-21092a3dad6b84cb
translationSourceHash: bede296801778ee8c5b09810cd9a6eca5522a50cc40c6f5a9e52f7cc24233ef9
translatedAt: 2026-07-29T15:07:29.049Z
translationReview: automatic
---

# Connecting the SEPPmail Admin GUI to Active Directory: Setting Up LDAP Authentication in 15.0.6

Up to firmware 15.0.5, the administration interface of the SEPPmail Secure E-Mail Gateway only knew local accounts. If you wanted to work cleanly, you created a dedicated local user for each administrator and added them to the admin group. That works, but it comes with the usual drawbacks of local accounts: separate passwords per appliance, no central offboarding, and no enforcement of the password policies from your directory service. Patch release 15.0.6 changes this. The admin GUI can now authenticate administrators against an external LDAP server such as Active Directory and map AD groups to local groups on the appliance.

The other changes in the release are summarized in the article on [SEPPmail 15.0.6 and 15.0.6.1](/en/blog/seppmail-releases-15-0-6-and-15-0-6-1). This article covers only the new external authentication.

## What the Feature Does

According to the Extended Release Notes, 15.0.6 adds a new **External Authentication** section under **User > Advanced Settings**. With it, the admin GUI authenticates against an external LDAP server, and external groups (such as AD security groups) are mapped to local groups on the appliance.

Externally authenticated users appear locally on the appliance and behave like local users, with one difference: their password cannot be changed on the appliance, because it lives in the external LDAP server. Password authority moves entirely into the directory.

An important distinction: the appliance already offered external authentication before, but only for the GINA web interface, configured per managed domain (the External authentication section in the domain configuration). What is new in 15.0.6 is that access to the administration interface itself can run through LDAP.

Whether the HIN Mailgateway has also received the LDAP login, I still need to test; I will update this article afterwards. Since the HIN appliances are based on the same SEPPmail firmware, I assume it has.

## Prerequisites

Three things should be in place before the setup:

- **Firmware 15.0.6.1:** The feature ships with 15.0.6; because of the two RuleEngine bugs in that release, going straight to hotfix 15.0.6.1 is the right choice.
- **An LDAP-capable directory:** An Active Directory, OpenLDAP, or comparable. If your users live only in Entra ID, which itself does not speak LDAP, [Microsoft Entra Domain Services](/en/blog/microsoft-entra-domain-services-ldap-kerberos) bridges the gap.
- **A bind account in the directory:** A dedicated, unprivileged service account with read access that the appliance uses for the LDAP search. Not a domain administrator.
- **An AD group for the gateway administrators:** For example, a security group SEPPmail-Admins that is later mapped to the local admin group. Membership in this group then decides who gets full administrative access.

TLS is enabled by default in the connection settings and should stay that way; administrator credentials do not belong on the network unencrypted. The appliance must be able to reach the LDAP server on the configured port (typically 636 for LDAPS).

## Setup Under User > Advanced Settings

The configuration lives in the admin GUI under **User > Advanced Settings** in the **External Authentication** section and consists of four blocks.

**1. Connection Settings:** The checkbox *Authenticate users to external LDAP server (e.g. Active Directory)* enables the feature. It is followed by the server address, port, the *TLS required* option, and the Bind DN and Bind Password of the service account.

**2. User Attributes:** This defines how the appliance finds user objects: the LDAP Object Class (typically person for Active Directory), the Search Base (the OU or container holding the administrator accounts), and the email attribute (default: mail).

**3. Group Attributes:** The same settings for group objects, so the appliance can resolve group memberships.

**4. Mapping Settings:** The decisive part. Under *Remote Group* you select the group from the LDAP server, under *Local Group* one or more local groups it is mapped to. For full administrative access, that is the admin group; its members are equal to the default admin user. If you want to differentiate, map to restricted groups such as readonly admin or to function-specific groups on the appliance instead.

Before saving, the built-in **Login Test** is worth using: with the username and password of a test account, you can verify that connection, search, and authentication work before the configuration goes live.

## Example Configurations

Adapt the following values to your own environment (example domain example.com). The field names match the External Authentication section on the appliance.

### Active Directory

| Field | Value |
|---|---|
| Server | dc01.example.com |
| Port | 636 |
| TLS required | enabled |
| Bind DN | CN=svc-seppmail,OU=ServiceAccounts,DC=example,DC=com |
| Bind Password | password of the service account |
| User: LDAP Object Class | person |
| User: Search Base | OU=IT,DC=example,DC=com |
| User: E-Mail Attribute | mail |
| Group: LDAP Object Class | group |
| Group: Search Base | OU=Groups,DC=example,DC=com |
| Mapping: Remote Group | SEPPmail-Admins |
| Mapping: Local Group | admin |

Notes on Active Directory: Any reachable domain controller works as the server; in multi-site environments, prefer a DC at the same site or an alias pointing to several DCs. Port 636 is LDAPS, which requires the DC certificate to be validatable by the appliance. Keep the search base narrow enough to contain the administrator accounts but not the whole directory. The mail attribute must be populated on the AD accounts.

### OpenLDAP

| Field | Value |
|---|---|
| Server | ldap01.example.com |
| Port | 636 |
| TLS required | enabled |
| Bind DN | cn=seppmail,ou=services,dc=example,dc=com |
| Bind Password | password of the service account |
| User: LDAP Object Class | inetOrgPerson |
| User: Search Base | ou=people,dc=example,dc=com |
| User: E-Mail Attribute | mail |
| Group: LDAP Object Class | groupOfNames |
| Group: Search Base | ou=groups,dc=example,dc=com |
| Mapping: Remote Group | seppmail-admins |
| Mapping: Local Group | admin |

Notes on OpenLDAP: In typical setups, users live as inetOrgPerson under ou=people. For groups, groupOfNames is the reliable choice, since membership is stored in the member attribute with the full DN. posixGroup entries list their members only as memberUid (username instead of DN); whether the appliance resolves that is not documented and should be verified with the Login Test before switching over. If the server only offers STARTTLS on port 389, that port belongs in the server field; the connection should never run unencrypted.

## Operational Notes

Three points deserve attention before LDAP login becomes the only way into the appliance:

- **Keep a local emergency account.** The passwords of external users live in the LDAP server. If the directory is unreachable (a network problem, AD maintenance, or the gateway is supposed to help fix an issue with that very network), you still need a local administrator account with a securely stored password. The default admin user should therefore not be retired but maintained as a documented emergency access.
- **MFA remains relevant.** 15.0.6 also reworked the MFA login: the second factor is no longer appended to the password but requested in its own field. External authentication does not replace a second factor.
- **Offboarding through the directory.** The real gain of the integration: when an administrator leaves the company, disabling the AD account or removing it from the mapped group is enough. Maintaining local accounts on every appliance is no longer necessary. The locally visible, externally authenticated user objects should still be reconciled with the directory periodically.

## Conclusion

LDAP authentication for the admin GUI closes a gap that existed on the appliance for a long time: administrator access can now be controlled centrally in the directory instead of per device. Together with the separate MFA field, 15.0.6 makes the login to the administration interface considerably more mature in a single release. If you introduce the feature, keep the group mapping deliberately restrictive and do not sacrifice the local emergency access.

## Sources

1.  [SEPPmail Extended Release Notes 15.0](https://downloads.seppmail.com/extrelnotes/150/ERN15.0.html): Entry on the admin GUI authentication with the feature description, configuration location, and the behavior of externally authenticated users.

2.  [SEPPmail-Dokumentation – «User > Advanced Settings»](https://docs.seppmail.com/ch/07_mi_15_usr_04_advanced-settings.html): Field reference for the External Authentication section (Connection, User Attributes, Group Attributes, Mapping, Login Test).

3.  [SEPPmail-Dokumentation – «Groups»](https://docs.seppmail.com/ch/07_mi_16_groups.html): Predefined groups on the appliance; members of the admin group have unrestricted administrative access.

4.  [SEPPmail-Dokumentation – «Revision History»](https://docs.seppmail.com/ch/20_revision-history.html): Official release notes for 15.0.6 with the entry on admin GUI authentication against external LDAP servers.

5.  [SEPPmail 15.0.6 and 15.0.6.1: Security Fixes and New Admin Features](/en/blog/seppmail-releases-15-0-6-and-15-0-6-1): Overview of all changes in the two releases.
