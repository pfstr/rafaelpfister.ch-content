---
title: "Active Directory and Microsoft Entra: the identity foundation"
blatt: "entra"
description: "Why running mail means running a directory: Active Directory and Entra ID in combination, synchronization with Entra Connect, Kerberos and LDAP on the local network, Entra Domain Services for cloud-only environments, and app registrations for system access."
fakten:
  - { label: "Product family", wert: "Active Directory (AD DS), Entra ID, Entra Domain Services" }
  - { label: "Vendor", wert: "Microsoft", href: "https://learn.microsoft.com/en-us/entra/identity/" }
  - { label: "Purpose", wert: "Identities, groups, authentication, and authorization" }
  - { label: "On-premises protocols", wert: "LDAP(S), Kerberos, DNS-integrated" }
  - { label: "Cloud protocols", wert: "OAuth 2.0, OpenID Connect, SAML over HTTPS" }
  - { label: "Synchronization", wert: "Microsoft Entra Connect (Sync)", href: "https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/whatis-azure-ad-connect" }
  - { label: "LDAP in the cloud", wert: "only via Entra Domain Services", href: "https://learn.microsoft.com/en-us/entra/identity/domain-services/overview" }
werbung: ["newsletter"]
ctaThemen: ["active-directory-entra", "microsoft-entra"]
---

# Active Directory and Microsoft Entra: the identity foundation

No mail system works without a clean directory: address books, group policies for gateways, sign-ins to management interfaces, and application permissions all depend on identities. Operating messaging therefore always means operating a piece of an identity platform as well, today usually as a pairing of on-premises Active Directory and Entra ID.

## Two worlds, one set of objects

**Active Directory Domain Services (AD DS)** is the on-premises directory: domain controllers, organizational units, groups, Kerberos for network sign-ins, LDAP(S) for applications and appliances. **Entra ID** (formerly Azure AD) is the cloud identity platform: sign-ins to M365 and SaaS applications via OAuth, OpenID Connect, and SAML, plus Conditional Access and app registrations. The two speak different protocol worlds, and precisely that shapes many projects: an appliance that expects LDAP cannot authenticate directly against Entra ID, and a modern SaaS application does not understand Kerberos.

## The bridge: Entra Connect

In hybrid environments, **Microsoft Entra Connect** synchronizes users, groups, and attributes from the on-premises AD to Entra ID. The sync is inconspicuous but operationally critical: it determines which attributes (such as `proxyAddresses` for mail addresses) arrive in the cloud, it has its own release cycles with support deadlines, and its service account is among the most sensitive in the organization. The source of truth remains the on-premises AD: mail addresses and groups are maintained there, not in the portal.

## LDAP and Kerberos in a cloud-only world

When everything moves to the cloud, a gap remains: legacy applications and appliances that require LDAP or Kerberos. **Microsoft Entra Domain Services** closes it with a managed domain that is fed from Entra ID and provides classic endpoints: LDAP(S), Kerberos, NTLM, and group policy in outline. This is not a full replacement for a domain controller (no schema extensions, no domain administrator rights), but it is exactly the right tool for continuing to operate a secure mail gateway or a line-of-business application without dedicated DCs.

## App registrations: identities for systems

Modern system access no longer runs through service accounts with passwords but through **app registrations** in Entra: an application receives its own identity, ideally authenticates with a certificate, and is granted granular permissions, for example Graph access to exactly one mailbox. For mail admins this is the standard way to connect gateways, scanners, and scripts to M365, including expiry monitoring for the stored certificates.

## Operational view

Three things shape everyday operations. **Synchronization** has to run and to be monitored, because a stalled sync often surfaces only when new accounts are missing or disabled accounts still have access. **Expiry dates** on certificates and client secrets of app registrations belong under monitoring, because they lapse silently and break integrations without warning. And **role assignment** follows least privilege: grant privileged roles for a limited time and in a traceable way rather than permanently, with separate accounts for administrative work.
