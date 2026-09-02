---
title: "Microsoft Entra Domain Services: LDAP and Kerberos for Cloud-Only Environments"
navTitle: "Entra Domain Services"
description: "Entra ID does not speak LDAP or Kerberos. Microsoft Entra Domain Services provides a managed Active Directory domain that synchronizes users from Entra ID and offers traditional protocols. How it works, limitations, costs, and a real-world email gateway use case."
date: "2026-07-30"
kategorie: "Active Directory / Entra"
timeToRead: "8 min read"
themen:
  - active-directory-entra
slug: "microsoft-entra-domain-services-ldap-and-kerberos-for-cloud-only-environments"
translationId: "article-9c271900a94406b8"
draft: false
translationOf: microsoft-entra-domain-services-ldap-kerberos
translationSourceHash: 6360f60ed2e92d286f0e279f487b62a86fa9a987c2f574b0a53af0d31f0d736b
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:20:02.220Z
translationReview: automatic
url: https://rafaelpfister.ch/en/blog/microsoft-entra-domain-services-ldap-and-kerberos-for-cloud-only-environments
---

# Microsoft Entra Domain Services: LDAP and Kerberos for Cloud-Only Environments

Anyone who has moved their users entirely to Microsoft Entra ID (formerly Azure Active Directory) will notice it at the latest when dealing with their first appliance or legacy application: Entra ID responds to queries through Microsoft Graph and modern authentication protocols such as OAuth and SAML, but not through LDAP, Kerberos, or NTLM. An LDAP bind against Entra ID simply is not possible. For anything that expects a traditional Active Directory, Microsoft offers its own service: Microsoft Entra Domain Services, formerly Azure AD Domain Services.

## What the Service Provides

Entra Domain Services is a managed Active Directory domain. Microsoft operates two Windows domain controllers for it in an Azure VNet, handles patching, replication, and backups, and automatically synchronizes users and groups from Entra ID to the domain. Synchronization runs in only one direction, from Entra ID to the managed domain; changes made directly in the domain do not flow back.

Externally, the domain behaves like a standard Active Directory: it responds to LDAP and LDAPS queries, supports Kerberos and NTLM authentication, allows VMs to join the domain, and provides limited Group Policy. Applications and devices do not need to be adapted; they see a domain controller.

## What You Need It For

The service is aimed at environments that are essentially cloud-only but run individual components with traditional directory requirements:

- **Appliances and line-of-business applications with LDAP integration:** Devices that look up users through LDAP, evaluate group memberships, or verify sign-ins through LDAP binds.
- **Lift-and-shift migrations:** Server workloads that must remain domain-joined (Kerberos, NTLM, domain join) without requiring you to operate your own domain controllers in Azure.
- **Environments without on-premises AD:** Where an Active Directory never existed or has been decommissioned, the managed domain replaces the need to build your own DCs and their operational overhead.

An important distinction: if you still operate an on-premises Active Directory with Entra Connect synchronization, you generally do not need the service; the appliance then queries the existing AD. Entra Domain Services fills the gap when Entra ID is the only user source.

## Architecture and Setup

The managed domain is deployed in its own VNet or subnet and receives two fixed domain controller addresses. Workloads in other VNets reach it through VNet peering; the DNS servers of the participating VNets must point to the domain controllers so that the domain name and objects can be resolved. Access is restricted through Network Security Groups to the sources and ports actually required.

Some characteristics of the managed domain that are relevant when connecting applications:

- Synchronized users are located in the **AADDC Users** OU, and without separate configuration, the domain uses the **onmicrosoft.com** suffix. Search Base and Bind DNs must reflect this structure, for example CN=svc-ldap,OU=AADDC Users,DC=example,DC=onmicrosoft,DC=com.
- There is no Domain Administrator. Administration is handled through the delegated AAD DC Administrators group; schema extensions are not possible.
- A dedicated, unprivileged account is sufficient for LDAP bind accounts; Directory Readers is the role for read-only directory queries in Entra ID.

## The Password Hash Problem

One issue regularly costs time during testing: Kerberos and NTLM sign-ins as well as LDAP binds require password hashes in the managed domain. For cloud-only accounts, Entra ID creates these hashes only when the password is next changed after the service has been enabled. A newly synchronized user is therefore visible in the directory but cannot sign in until they have changed their password once. For hybrid accounts, the hashes must also be synchronized from the on-premises AD through Entra Connect.

## Secure LDAP Step by Step

Within the domain, LDAP runs unencrypted over port 389 by default. For sign-ins and any access outside strictly isolated networks, the connection belongs on Secure LDAP (LDAPS, port 636); the service offers access from outside the VNet only in encrypted form anyway. Setup consists of four steps.

**1. Obtain a certificate.** Secure LDAP requires its own certificate, uploaded as a PFX including the private key. The Subject or SAN must cover the managed domain as a wildcard (for example, *.example.onmicrosoft.com), since requests can land on either of the two domain controllers. Options include a public CA, your own PKI, or a separately created self-signed certificate. With a self-signed certificate, the chain must be installed as trusted on every requesting system; not every appliance allows this. If you have a choice, your own PKI or a public CA is the safer option.

**2. Enable Secure LDAP.** In the portal under Settings > Secure LDAP, enable the feature and upload the PFX along with its password. You can optionally enable access over the internet there; the managed domain will then receive a public IP address.

**3. Network and DNS.** The external IP address is listed under Properties. The associated NSG rule opens TCP/636 and should be restricted to the source IP addresses actually needed, not Any. For name resolution, a DNS record (for example, ldaps.example.com) points to this IP address; it must match the certificate. Internal access continues directly against the domain controller addresses.

**4. Test the connection.** Before switching the application over, test against port 636 with an LDAP browser, ldp.exe, or ldapsearch: bind with the service account, then search under the AADDC Users OU. Only once bind and search work correctly should you move on to the application.

To set up the service itself, the portal account needs the Application Administrator, Domain Services Contributor, and Groups Administrator roles; deployment of the managed domain takes a little over an hour. TLS 1.2 can also be enforced as the minimum in the security settings.

## Costs

Entra Domain Services is an ongoing operating expense: the service is billed hourly by SKU, plus VNet, peering, and any test VMs. For one small LDAP use case, that is a substantial price; however, the alternative of operating your own domain controllers as VMs comes with patching, backup, and availability responsibilities in exchange for the savings.

## Real-World Use Case: Email Gateway with LDAP Integration

A concrete example in the appliance category is the SEPPmail Secure E-Mail Gateway. It uses LDAP for user provisioning and authorization queries and, since firmware 15.0.6, also for [sign-in to the Admin GUI](/blog/seppmail-admin-gui-ldap-authentifizierung). An appliance in the Azure VNet reaches the managed domain through VNet peering using a dedicated bind account (Directory Readers), secured through NSGs. At the latest for Admin GUI sign-in, whose TLS option is enabled by default, the connection belongs on Secure LDAP.

## Conclusion

Entra Domain Services is not a replacement for Entra ID, but a bridge: the service translates a cloud user base into a traditional AD domain for anything requiring LDAP, Kerberos, or domain join. If you only need to connect a single application, you should weigh the ongoing costs against modernizing the application. Once the service is in place, appliances and legacy applications behave as they would in a familiar AD environment, including the described specifics regarding OU structure, permissions, and password hashes.

## Sources

1.  [Microsoft Learn – “What is Microsoft Entra Domain Services?”](https://learn.microsoft.com/de-de/entra/identity/domain-services/overview): Scope of the managed domain, supported protocols, and distinction from Entra ID and self-operated domain controllers.

2.  [Microsoft Learn – “Synchronization in Entra Domain Services”](https://learn.microsoft.com/de-de/entra/identity/domain-services/synchronization): One-way synchronization, OU structure, and password hash behavior for cloud-only and hybrid accounts.

3.  [Microsoft Learn – “Configure Secure LDAP”](https://learn.microsoft.com/de-de/entra/identity/domain-services/tutorial-configure-ldaps): LDAPS with a dedicated certificate for encrypted LDAP access.

4.  [Connect the SEPPmail Admin GUI to Active Directory](/blog/seppmail-admin-gui-ldap-authentifizierung): Setting up LDAP sign-in for the Admin GUI starting with firmware 15.0.6.
