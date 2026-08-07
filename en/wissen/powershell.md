---
title: "PowerShell and Microsoft Graph: Automation for Mail Admins"
blatt: "powershell"
description: "The automation layer above Exchange, Entra and Microsoft 365: Exchange Online PowerShell, Microsoft Graph as an API, app registrations with certificate authentication, and where the classic modules are being retired."
fakten:
  - { label: "Purpose", wert: "Administration and automation of Exchange, Entra and M365" }
  - { label: "Shell", wert: "PowerShell 7 (cross-platform), Windows PowerShell 5.1 being phased out" }
  - { label: "Exchange Online", wert: "ExchangeOnlineManagement module (REST-based)", href: "https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell-v2" }
  - { label: "Microsoft Graph", wert: "Unified API for M365 at graph.microsoft.com", href: "https://learn.microsoft.com/en-us/graph/overview" }
  - { label: "Authentication", wert: "Entra app registration, app-only with certificate" }
  - { label: "Superseded", wert: "Basic auth, remote PowerShell sessions, MSOnline and AzureAD modules" }
  - { label: "Transport", wert: "HTTPS (443)" }
werbung: ["newsletter"]
ctaThemen: ["microsoft-365-exchange", "active-directory-entra"]
---

# PowerShell and Microsoft Graph: Automation for Mail Admins

Beyond a certain environment size, the administrative interface is little more than illustration material. Mailbox permissions across a hundred accounts, transport rules, reports, connecting an appliance to a mailbox: the actual work happens in PowerShell and, increasingly, through Microsoft Graph. Understanding both layers and their authentication is what makes automation secure instead of dependent on stored passwords.

## Exchange Online PowerShell

The **ExchangeOnlineManagement** module is the standard route for everything Exchange-specific: `Get-Mailbox`, `Set-TransportRule`, connectors, migration batches. It works over REST, no longer requires the classic remote sessions, and `Connect-ExchangeOnline` supports both interactive sign-in and unattended scenarios. For scripts the rule is: no basic auth, no cleartext passwords. The clean approach is **certificate-based app-only authentication**: an app registration in Entra, a certificate instead of a secret, and the `Exchange.ManageAsApp` permission together with a matching management role.

## Microsoft Graph: the single API

**Microsoft Graph** consolidates the interfaces of nearly all M365 services under `https://graph.microsoft.com`: users and groups, mail and calendar, devices, reports. Instead of product-specific legacy modules (MSOnline and AzureAD are deprecated), a single versioned REST API is addressed, either directly over HTTP or through the Graph PowerShell modules. Particularly relevant for mail scenarios: applications can be granted `Mail.Send` or mailbox access, which allows appliances and line-of-business applications to send mail through M365 without the legacy burden of SMTP auth.

The permission model distinguishes **delegated permissions** (on behalf of a signed-in user) from **application permissions** (app-only, without a user). For automation and system integrations, app-only is the appropriate route, and **application access policies** narrow access to exactly the mailboxes an application actually needs. Without this restriction, an app holding mailbox permissions may access every mailbox in the tenant, a point auditors rightly raise.

## Certificate instead of secret

App registrations accept client secrets and certificates. Secrets are passwords with an expiry date and, in practice, end up in scripts and pipelines. Certificates are the more robust standard: private key in the certificate store or a vault, public portion in the app registration, expiry monitoring as for any other certificate in the environment. Connecting a gateway or a line-of-business application to an M365 mailbox therefore comes down to three steps: register the app, add the certificate, assign permissions granularly.

## Getting started in practice

```powershell
Connect-ExchangeOnline -AppId $appId -CertificateThumbprint $thumb -Organization example.onmicrosoft.com
Get-EXOMailbox -ResultSize 25 | Select-Object DisplayName, PrimarySmtpAddress
```

