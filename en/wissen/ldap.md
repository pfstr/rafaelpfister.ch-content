---
title: "Understanding LDAP: the directory protocol behind AD, Entra, and mail gateways"
blatt: "ldap"
description: "What LDAP is and how it works: a directory rather than a database, DIT and distinguished names, bind and search, LDAPS vs. StartTLS, and what of that really counts in Active Directory, Entra, and secure mail gateways."
fakten:
  - { label: "Full name", wert: "Lightweight Directory Access Protocol" }
  - { label: "Purpose", wert: "Access to directory services: search, authenticate, modify" }
  - { label: "Introduced", wert: "1993 · current version LDAPv3 (since 1997)" }
  - { label: "OSI layer", wert: "Application (layer 7)" }
  - { label: "Transport", wert: "TCP", href: "https://datatracker.ietf.org/doc/html/rfc9293" }
  - { label: "Ports", wert: "389 (StartTLS) · 636 (LDAPS)", href: "https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=ldap" }
  - { label: "Standard", wert: "RFC 4510–4519", href: "https://datatracker.ietf.org/doc/html/rfc4510" }
  - { label: "Typical server", wert: "Active Directory (Global Catalog: 3268/3269)", href: "https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/config-firewall-for-ad-domains-and-trusts" }
  - { label: "Entra ID", wert: "no native LDAP, only via Domain Services", href: "/en/blog/microsoft-entra-domain-services-ldap-and-kerberos-for-cloud-only-environments" }
  - { label: "Tools", wert: "ldapsearch, ldp.exe", href: "https://www.openldap.org/software/man.cgi?query=ldapsearch" }
werbung: ["newsletter"]
ctaThemen: ["active-directory-entra", "seppmail"]
---

# Understanding LDAP: the directory protocol behind AD, Entra, and mail gateways

Few protocols carry as many logins, address books, and permissions as LDAP, and few are used as routinely without ever being encountered directly. Connecting a secure mail gateway to Active Directory, centralizing authentication for an appliance GUI, or cleaning up license counts by way of directory groups all come down to LDAP in the end. What follows covers the protocol to the depth that actually matters in operation.

## A directory, not a database

LDAP (Lightweight Directory Access Protocol) is the access protocol for directory services. A directory is a specialized form of data storage: heavily read-optimized, hierarchically organized, and designed for small queries by the million, such as "What is the mail address of user X?" or "Is Y a member of group Z?" Write operations are possible, but rare and expensive. That is what separates a directory from a relational database: no joins, no transactions spanning many objects, but fast, filterable searches across a tree.

The "lightweight" in the name is historical. LDAP emerged in the 1990s as a slim way into X.500 directories, whose original protocol (DAP) required the complete OSI stack. LDAP brought the same idea to TCP/IP and became the standard that every product speaks today: Active Directory, OpenLDAP, 389 Directory Server, and practically every appliance that lists "AD integration" on its data sheet.

## The data model: tree, DN, and objectClass

An LDAP directory is a tree, the **Directory Information Tree (DIT)**. Every entry has a unique name, its **Distinguished Name (DN)**, assembled from its path read from the bottom up:

```text
CN=Rafael Pfister,OU=Users,OU=Zurich,DC=example,DC=ch
```

The leftmost component (`CN=Rafael Pfister`) is the **Relative Distinguished Name (RDN)**; `OU` denotes organizational units and `DC` the domain components. Every entry carries **attributes** (`mail`, `sAMAccountName`, `memberOf`, `proxyAddresses` …), and which attributes are permitted or mandatory is determined by its **objectClasses** (`user`, `group`, `organizationalUnit`): the schema of the directory.

In practice this means that configuring an LDAP integration almost always comes down to the same three questions. **Where** does the search begin (base DN)? **What** is it filtered on (search filter)? **As whom** is it performed (bind account)?

## The operations: bind, search, modify

Three operations cover nearly everything that comes up in daily operation:

- **Bind** is authentication against the directory. A *simple bind* transmits the DN and the password. Without transport encryption this happens in the clear, which is why it is acceptable only over LDAPS or StartTLS. SASL mechanisms, typically Kerberos in an Active Directory environment, are the more robust alternative. Many products verify user passwords simply by attempting a bind with the supplied credentials: if the bind succeeds, the password is correct.
- **Search** is the most frequent operation: starting from a base DN, with a scope (`base`, `one`, `sub`) and a filter such as `(&(objectClass=user)(mail=*@example.ch))`. Everything a mail gateway wants to know about recipients ultimately amounts to a search.
- **Modify/Add/Delete** change entries. They are less common as direct administrative actions, but they are the basis of every provisioning script.

## LDAPS or StartTLS?

Both encrypt with TLS; the path differs. **LDAPS** (port 636) establishes the TLS session immediately, whereas **StartTLS** begins in the clear on port 389 and upgrades the existing connection by command. LDAPS is the older, "unofficial" approach and still the more dependable standard in practice, because a glance at the firewall settles the question: port 636 is always encrypted. Since Microsoft's hardening of **LDAP channel binding and signing**, the conclusion is even clearer: unencrypted LDAP against Active Directory is on its way out. One certificate detail matters: the hostname in the bind target must match the server certificate, and the appliance must trust the issuing CA. That is the most common source of error behind "LDAP stopped working" after a certificate renewal.

## Where LDAP appears in messaging environments

- **Active Directory** is the LDAP server par excellence: ports 389/636, plus the **Global Catalog** on 3268/3269 for forest-wide searches.
- **Microsoft Entra ID does not speak LDAP.** For legacy LDAP consumers, cloud-only environments need **Entra Domain Services**, a managed domain with LDAP(S) and Kerberos endpoints.
- **Secure mail gateways** (SEPPmail, Totemomail, Cisco ESA/SMA) use LDAP in several places at once: recipient verification before acceptance, group memberships for policies, address book comparison for license counting, and increasingly authentication of the admin GUI itself against the directory.

## Quick start: ldapsearch

The fastest way to verify an integration is the command line. The following is an LDAPS search against Active Directory:

```bash
ldapsearch -H ldaps://dc1.example.ch:636 \
  -D "CN=svc-gateway,OU=Service,DC=example,DC=ch" -W \
  -b "OU=Users,DC=example,DC=ch" \
  "(mail=rafael.pfister@example.ch)" mail memberOf
```

If the server responds with the entry, then network, certificate, bind account, and filter are all correct. If it does not, the error message narrows the problem down cleanly: `err=49` is a failed bind (in Active Directory with a sub-code, for example `52e` = wrong password, `775` = account locked out), certificate errors point to the trust store or the hostname, and timeouts point to the firewall or the wrong port.

