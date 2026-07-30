---
title: "Microsoft Entra Domain Services: LDAP and Kerberos for cloud-only environments"
navTitle: "Entra Domain Services"
description: "Entra ID does not speak LDAP or Kerberos. Microsoft Entra Domain Services provides a managed Active Directory domain that synchronises users from Entra ID and offers classic protocols. How it works, limitations, costs and a practical case involving an email gateway."
date: "2026-07-30"
kategorie: "Active Directory / Entra"
timeToRead: "8 min read"
themen:
  - active-directory-entra
slug: "microsoft-entra-domain-services-ldap-and-kerberos-for-cloud-only-environments"
translationId: "article-9c271900a94406b8"
draft: false
translationOf: microsoft-entra-domain-services-ldap-kerberos
url: https://rafaelpfister.ch/en/blog/microsoft-entra-domain-services-ldap-and-kerberos-for-cloud-only-environments
translationSourceHash: 00f01b9fa1426d692146e27b2e15e6926e04ea3cccd4855bd0b18c8c10e36e0d
translationModel: gpt-5.6-terra
translatedAt: 2026-07-30T12:21:22.632Z
translationReview: automatic
---

# Microsoft Entra Domain Services: LDAP and Kerberos for cloud-only environments

Anyone who has moved their users entirely to Microsoft Entra ID (formerly Azure Active Directory) will notice it at the latest when dealing with their first appliance or legacy application: Entra ID responds to queries via Microsoft Graph and modern authentication protocols such as OAuth and SAML, but not via LDAP, Kerberos or NTLM. An LDAP bind against Entra ID is simply not possible. For anything that expects a traditional Active Directory, Microsoft offers a dedicated service: Microsoft Entra Domain Services, formerly Azure AD Domain Services.

## What the service provides

Entra Domain Services is a managed Active Directory domain. Microsoft operates two Windows domain controllers for it in an Azure VNet, handles patching, replication and backups, and automatically synchronises users and groups from Entra ID into the domain. Synchronisation runs in one direction only, from Entra ID to the managed domain; changes made directly in the domain do not flow back.

Externally, the domain behaves like a standard Active Directory: it responds to LDAP and LDAPS queries, supports Kerberos and NTLM authentication, allows VMs to join the domain, and provides limited Group Policy. Applications and devices do not need to be adapted; they see a domain controller.

## What it is needed for

The service is aimed at environments that are actually cloud-only but run individual components with traditional directory requirements:

- **Appliances and line-of-business applications with LDAP integration:** Devices that look up users via LDAP, evaluate group memberships or verify logins using LDAP bind.
- **Lift-and-shift migrations:** Server workloads that must remain domain-bound (Kerberos, NTLM, domain join), without having to operate their own domain controllers in Azure.
- **Environments without on-premises AD:** Where Active Directory never existed or has been decommissioned, the managed domain replaces the need to build dedicated DCs and their operational overhead.

Important distinction: anyone still operating an on-premises Active Directory with Entra Connect synchronisation generally does not need the service; the appliance can query the existing AD instead. Entra Domain Services fills the gap when Entra ID is the only user source.

## Architecture and setup

The managed domain is deployed in its own VNet or subnet and receives two fixed domain controller addresses. Workloads in other VNets reach it through VNet peering; the DNS servers of the participating VNets must point to the domain controllers so that the domain name and objects can be resolved. Access is restricted via Network Security Groups to the sources and ports actually required.

Some characteristics of the managed domain that are relevant when connecting applications:

- Synchronised users are located in the **AADDC Users** OU, and the domain uses the **onmicrosoft.com** suffix without additional configuration. Search Base and bind DNs must reflect this structure, for example CN=svc-ldap,OU=AADDC Users,DC=example,DC=onmicrosoft,DC=com.
- There is no Domain Administrator. Administration is performed through the delegated AAD DC Administrators group; schema extensions are not possible.
- A dedicated, unprivileged account is sufficient for LDAP bind accounts; Directory Readers is the role for read-only directory queries in Entra ID.

## The password hash trap

One point regularly costs time in testing: Kerberos and NTLM logins as well as LDAP binds require password hashes in the managed domain. For cloud-only accounts, Entra ID only generates these hashes when the password is next changed after the service has been activated. A newly synchronised user is therefore visible in the directory but can only log in after changing their password once. For hybrid accounts, the hashes must also be synchronised from the on-premises AD via Entra Connect.

## Secure LDAP step by step

Within the domain, LDAP runs unencrypted by default over port 389. For logins and for any access outside strictly isolated networks, the connection should use Secure LDAP (LDAPS, port 636); the service only offers access from outside the VNet in encrypted form anyway. Setup consists of four steps.

**1. Obtain a certificate.** Secure LDAP requires a separate certificate, uploaded as a PFX including its private key. The Subject or SAN must cover the managed domain with a wildcard (for example *.example.onmicrosoft.com), as requests may reach either of the two domain controllers. Options include a public CA, your own PKI or a separately created self-signed certificate. With a self-signed certificate, the chain must be stored as trusted on every requesting system; not every appliance permits this. If there is a choice, your own PKI or a public CA is the safer option.

**2. Enable Secure LDAP.** In the portal, under Settings > Secure LDAP, enable the feature and upload the PFX together with its password. You can also enable internet access there; the managed domain then receives a public IP address.

**3. Network and DNS.** The external IP address is listed under Properties. The associated NSG rule opens TCP/636 and should be restricted to the source IP addresses actually required, rather than Any. For name resolution, a DNS record (for example ldaps.example.com) points to this IP address; it must match the certificate. Internal access continues to go directly to the domain controller addresses.

**4. Test the connection.** Before switching over the application, it is worth testing port 636 with an LDAP browser, ldp.exe or ldapsearch: bind with the service account, then search under the AADDC Users OU. Only once binding and searching work properly should the application be configured.

To set up the service itself, the portal account requires the Application Administrator, Domain Services Contributor and Groups Administrator roles; deployment of the managed domain takes well over an hour. TLS 1.2 can also be enforced as the minimum in the security settings.

## Costs

Entra Domain Services is an ongoing operating cost: the service is billed hourly by SKU, in addition to VNet, peering and any test VMs. For one small LDAP use case, that is a substantial price; however, the alternative of operating your own domain controllers as VMs trades the saving for responsibility for patching, backups and availability.

## Practical case: Email gateway with LDAP integration

A specific example of the appliance category is the SEPPmail Secure E-Mail Gateway. It uses LDAP for user provisioning and permission queries and, since firmware 15.0.6, also for [logging in to the Admin GUI](/blog/seppmail-admin-gui-ldap-authentifizierung). An appliance in the Azure VNet reaches the managed domain through VNet peering using a dedicated bind account (Directory Readers), secured via NSGs. Particularly for Admin GUI login, where the TLS option is enabled by default, the connection should use Secure LDAP.

## Conclusion

Entra Domain Services is not a replacement for Entra ID, but a bridge: the service translates a cloud user base into a traditional AD domain for anything requiring LDAP, Kerberos or domain join. Anyone who only needs to connect a single application should weigh the ongoing costs against modernising that application. Once the service is in place, appliances and legacy applications behave as they would in a familiar AD environment, including the characteristics described for OU structure, permissions and password hashes.

## Sources

1.  [Microsoft Learn – «What is Microsoft Entra Domain Services?»](https://learn.microsoft.com/de-de/entra/identity/domain-services/overview): Scope of the managed domain, supported protocols and distinction from Entra ID and self-operated domain controllers.

2.  [Microsoft Learn – «Synchronisation in Entra Domain Services»](https://learn.microsoft.com/de-de/entra/identity/domain-services/synchronization): One-way synchronisation, OU structure and the behaviour of password hashes for cloud-only and hybrid accounts.

3.  [Microsoft Learn – «Configure Secure LDAP»](https://learn.microsoft.com/de-de/entra/identity/domain-services/tutorial-configure-ldaps): LDAPS with a dedicated certificate for encrypted LDAP access.

4.  [Connect the SEPPmail Admin GUI to Active Directory](/blog/seppmail-admin-gui-ldap-authentifizierung): Setting up LDAP login for the Admin GUI from firmware 15.0.6.
