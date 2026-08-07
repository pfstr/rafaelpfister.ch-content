---
title: "Kerberos: tickets, KDC, and authentication in Active Directory"
blatt: "kerberos"
description: "How Kerberos works and why Active Directory is built on it: tickets instead of passwords, KDC and TGT, service principal names and keytabs, the time synchronization requirement, and the typical failure patterns."
fakten:
  - { label: "Purpose", wert: "Network authentication with tickets instead of password transmission" }
  - { label: "Port", wert: "88 (TCP/UDP), password service 464" }
  - { label: "Standard", wert: "Kerberos V5, RFC 4120", href: "https://datatracker.ietf.org/doc/html/rfc4120" }
  - { label: "In Active Directory", wert: "Default mechanism; domain controllers act as the KDCs", href: "https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview" }
  - { label: "Cloud", wert: "Entra ID uses OAuth/OIDC; Kerberos via Entra Domain Services" }
  - { label: "Requirement", wert: "Time synchronization (five-minute tolerance)" }
  - { label: "Tools", wert: "klist, setspn, ktpass, kinit" }
werbung: ["newsletter"]
ctaThemen: ["active-directory-entra"]
---

# Kerberos: tickets, KDC, and authentication in Active Directory

A user who signs in to a domain in the morning and then reaches file servers, printers, and the intranet without any further password prompts is using Kerberos. The protocol is the backbone of authentication in Active Directory, and it stays invisible until something breaks. At that point, a handful of basic concepts is worth more than any search for the error message.

## Tickets instead of passwords

The core idea is that the password never leaves the client. Instead, the client obtains a **Ticket Granting Ticket (TGT)** from the **Key Distribution Center (KDC)**, which in Active Directory means the domain controllers. Using that TGT, it requests a **service ticket** for each service and presents it as proof of identity. Services validate tickets cryptographically without querying the KDC themselves. That makes Kerberos fast, scalable, and resistant to password interception, as long as tickets remain short-lived (ten hours by default).

## SPNs and keytabs: where it gets practical

Before a client can request a ticket for a service, that service must carry a **Service Principal Name (SPN)** in the directory, for example `HTTP/intranet.example.ch`. Missing or duplicate SPNs are the most common Kerberos fault in day-to-day Active Directory operations: the service then either falls back to NTLM silently or refuses the request. **Keytabs** are the counterpart for non-Windows systems: a file containing the service keys that lets a Linux server or an appliance identify itself as a Kerberos service, generated with `ktpass` or equivalent tools.

## Time is part of the protocol

Kerberos tickets carry timestamps as a defense against replay attacks. Once a system clock drifts beyond the tolerated five minutes, sign-in fails with errors that offer little guidance. NTP monitoring is therefore part of basic Kerberos hygiene: when "suddenly nobody can sign in anymore," system time is one of the first things to check, particularly on virtualized domain controllers.

## Kerberos and the cloud

**Entra ID does not speak Kerberos**; it uses OAuth and OpenID Connect. For legacy components that require Kerberos or LDAP, **Microsoft Entra Domain Services** provides a managed domain with the classic endpoints. Appliances and line-of-business applications that depend on a Kerberos binding rarely move into a cloud-only environment without this building block.

## Quick diagnostics

```powershell
klist                       # Which tickets does the current session hold?
setspn -Q HTTP/intranet.example.ch   # Is the SPN registered exactly once?
w32tm /query /status        # Is the system time correct?
```

