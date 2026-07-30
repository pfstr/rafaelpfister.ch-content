---
title: "Microsoft Entra Domain Services: LDAP and Kerberos for Cloud-Only Environments"
navTitle: "Entra Domain Services"
description: "Entra ID speaks neither LDAP nor Kerberos. Microsoft Entra Domain Services provides a managed Active Directory domain that synchronizes users from Entra ID and offers the classic protocols. How it works, its limits, costs, and a real-world case with an email gateway."
date: "2026-07-30"
kategorie: "Active Directory / Entra"
timeToRead: "8 min to read"
themen:
  - active-directory-entra
slug: "microsoft-entra-domain-services-ldap-kerberos"
translationOf: "microsoft-entra-domain-services-ldap-kerberos"
url: "https://rafaelpfister.ch/en/blog/microsoft-entra-domain-services-ldap-kerberos"
draft: false
translationId: microsoft-entra-domain-services-ldap-kerberos
translationReview: automatic
---

# Microsoft Entra Domain Services: LDAP and Kerberos for Cloud-Only Environments

If you have moved your users entirely to Microsoft Entra ID (formerly Azure Active Directory), you notice it at the latest with the first appliance or legacy application: Entra ID answers queries through Microsoft Graph and modern authentication protocols such as OAuth and SAML, but not through LDAP, Kerberos, or NTLM. An LDAP bind against Entra ID is simply not possible. For everything that expects a classic Active Directory, Microsoft offers a dedicated service: Microsoft Entra Domain Services, formerly Azure AD Domain Services.

## What the Service Provides

Entra Domain Services is a managed Active Directory domain. Microsoft operates two Windows domain controllers in an Azure VNet, takes care of patching, replication, and backups, and automatically synchronizes users and groups from Entra ID into the domain. Synchronization runs in one direction only, from Entra ID into the managed domain; changes made directly in the domain do not flow back.

To the outside, the domain behaves like an ordinary Active Directory: it answers LDAP and LDAPS queries, supports Kerberos and NTLM authentication, allows VMs to join the domain, and offers limited group policies. Applications and devices do not need to be adapted; they see a domain controller.

## What You Need It For

The service targets environments that are essentially cloud-only but run individual components with classic directory requirements:

- **Appliances and business applications with LDAP integration:** Devices that look up users via LDAP, evaluate group memberships, or verify logins via LDAP bind.
- **Lift-and-shift migrations:** Server workloads that must remain domain-joined (Kerberos, NTLM, domain join) without operating your own domain controllers in Azure.
- **Environments without on-premises AD:** Where an Active Directory never existed or has been decommissioned, the managed domain replaces building your own DCs and their operational burden.

An important distinction: if you still run an on-premises Active Directory with Entra Connect synchronization, you usually do not need the service; the appliance queries the existing AD. Entra Domain Services fills the gap when Entra ID is the only user source.

## Architecture and Setup

The managed domain is deployed into its own VNet or subnet and receives two fixed domain controller addresses. Workloads in other VNets reach it via VNet peering; the DNS servers of the involved VNets must point to the domain controllers so the domain name and its objects resolve. Access is restricted via network security groups to the sources and ports actually needed.

Some peculiarities of the managed domain that matter when connecting applications:

- Synchronized users live in the OU **AADDC Users**, and without custom configuration the domain carries the **onmicrosoft.com** suffix. Search base and bind DNs must reflect this structure, for example CN=svc-ldap,OU=AADDC Users,DC=example,DC=onmicrosoft,DC=com.
- There is no domain administrator. Management runs through the delegated AAD DC Administrators group; schema extensions are not possible.
- A dedicated, unprivileged account is sufficient for LDAP binds; for pure directory queries in Entra ID, the Directory Readers role.

## The Password Hash Trap

One point regularly costs time in testing: Kerberos and NTLM logins as well as LDAP binds need password hashes in the managed domain. For cloud-only accounts, Entra ID generates these hashes only at the next password change after the service is enabled. A freshly synchronized user is therefore visible in the directory but can only log in after changing their password once. For hybrid accounts, the hashes must be synchronized from the on-premises AD via Entra Connect.

## Secure LDAP Step by Step

Within the domain, LDAP runs unencrypted over port 389 by default. For logins and for any access outside strictly isolated networks, the connection belongs on Secure LDAP (LDAPS, port 636); access from outside the VNet is only offered encrypted anyway. The setup consists of four steps.

**1. Obtain a certificate.** Secure LDAP needs its own certificate, uploaded as a PFX including the private key. The subject or SAN must cover the managed domain as a wildcard (for example *.example.onmicrosoft.com), since requests can land on either of the two domain controllers. Possible sources are a public CA, your own PKI, or a purpose-made self-signed certificate. With the self-signed route, the chain must be trusted on every querying system; not every appliance allows that. If you have the choice, your own PKI or a public CA is the calmer path.

**2. Enable Secure LDAP.** In the portal under Settings > Secure LDAP, the feature is switched on and the PFX is uploaded with its password. Optionally, access over the internet can be enabled there; the managed domain then receives a public IP address.

**3. Network and DNS.** The external IP address is listed under Properties. The corresponding NSG rule opens TCP/636 and should be restricted to the source IPs actually needed, not Any. For name resolution, a DNS record (for example ldaps.example.com) points to this IP; it must match the certificate. Internal access continues to run directly against the domain controller addresses.

**4. Test the connection.** Before switching the application over, test with an LDAP browser, ldp.exe, or ldapsearch against port 636: bind with the service account, then a search under the AADDC Users OU. Only when bind and search work cleanly there is it the application's turn.

For setting up the service itself, the portal account needs the Application Administrator, Domain Services Contributor, and Groups Administrator roles; deploying the managed domain takes a good hour. The security settings also allow enforcing TLS 1.2 as the minimum.

## Costs

Entra Domain Services is a permanent operating cost: the service is billed per hour by SKU, plus VNet, peering, and any test VMs. For a single small LDAP use case, that is a steep price; the alternative of running your own domain controllers as VMs buys back the savings with patching, backup, and availability responsibility.

## Real-World Case: Email Gateway with LDAP Integration

A concrete example from the appliance category is the SEPPmail Secure E-Mail Gateway. It uses LDAP for user creation and authorization queries and, since firmware 15.0.6, also for [logging in to the admin GUI](/en/blog/seppmail-admin-gui-ldap-authentication). An appliance in an Azure VNet reaches the managed domain via VNet peering with a dedicated bind account (Directory Readers), secured via NSGs. At the latest for the admin GUI login, whose TLS option is enabled by default, the connection belongs on Secure LDAP.

## Conclusion

Entra Domain Services is not a replacement for Entra ID but a bridge: the service translates a cloud user base into a classic AD domain for everything that demands LDAP, Kerberos, or domain join. If you only need to connect a single application, weigh the running costs against modernizing the application. Once the service is in place, appliances and legacy applications behave as in a familiar AD environment, including the described peculiarities of OU structure, permissions, and password hashes.

## Sources

1.  [Microsoft Learn – "What is Microsoft Entra Domain Services?"](https://learn.microsoft.com/en-us/entra/identity/domain-services/overview): Feature scope of the managed domain, supported protocols, and the distinction from Entra ID and self-managed domain controllers.

2.  [Microsoft Learn – "Synchronization in Entra Domain Services"](https://learn.microsoft.com/en-us/entra/identity/domain-services/synchronization): One-way synchronization, OU structure, and the password hash behavior for cloud-only and hybrid accounts.

3.  [Microsoft Learn – "Configure Secure LDAP"](https://learn.microsoft.com/en-us/entra/identity/domain-services/tutorial-configure-ldaps): LDAPS with your own certificate for encrypted LDAP access.

4.  [Connecting the SEPPmail Admin GUI to Active Directory](/en/blog/seppmail-admin-gui-ldap-authentication): Setting up LDAP login for the admin GUI as of firmware 15.0.6.
