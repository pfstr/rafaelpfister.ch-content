---
title: "Cisco Secure Email: ESA and SMA in operation"
blatt: "cisco"
description: "The Cisco email security platform from an administrator's perspective: the Email Security Appliance (ESA) as a gateway, the Security Management Appliance (SMA) for administration and quarantine, AsyncOS, working on the CLI, and certificate maintenance."
fakten:
  - { label: "Products", wert: "Secure Email Gateway (ESA) · Secure Email and Web Manager (SMA)" }
  - { label: "Vendor", wert: "Cisco", href: "https://www.cisco.com/c/en/us/support/security/email-security-appliance/series.html" }
  - { label: "Operating system", wert: "AsyncOS (separate release line per product)" }
  - { label: "Purpose", wert: "Spam, malware, and policy filtering in the mail flow; the SMA centralizes administration and quarantine" }
  - { label: "Integration", wert: "SMTP in the mail path · LDAP(S) to the directory" }
  - { label: "Administration", wert: "Web GUI and CLI (SSH); some certificate tasks only via CLI" }
  - { label: "Form factors", wert: "Hardware, virtual (ESAV/SMAV), cloud" }
werbung: ["newsletter"]
ctaThemen: ["cisco-esa-sma"]
---

# Cisco Secure Email: ESA and SMA in operation

Cisco's email security line consists of two roles that are often mentioned in the same breath and yet serve different purposes. The **Email Security Appliance (ESA)**, today the Secure Email Gateway, sits in the mail flow and filters: spam, malware, policies, content filters. The **Security Management Appliance (SMA)**, today the Secure Email and Web Manager, sits alongside it and centralizes: administration of multiple ESAs, reporting, message tracking, and the central **spam quarantine** in which end users review their held messages. Publishing the SMA quarantine page within an organization turns the SMA into a user-facing system, including the associated certificate and availability maintenance.

## AsyncOS and the two administration worlds

Both appliances run **AsyncOS**, Cisco's own operating system with separate release lines per product. Administration happens through the web GUI and the CLI, and the balance between them is typical for Cisco: much can be done in the GUI, but some operational tasks necessarily go through the **CLI over SSH**, most prominently SMA certificate management via the `certconfig` dialog. The CLI follows a question-and-answer pattern concluded by `commit`; aborting a session with Ctrl+C discards the change. For recurring tasks, a documented transcript of the dialog per appliance is worthwhile, because the menus change between AsyncOS versions.

## Certificates: the recurring obligation

The ESA and SMA carry several TLS roles at the same time: inbound and outbound SMTP TLS, HTTPS for the management interface and the quarantine page, and LDAPS to the directory. Current AsyncOS versions validate the **complete certificate chain** on import; an internal corporate CA therefore has to be placed in the appliance trust store first, otherwise the import of the organization's own certificate fails. A new key pair can be obtained via an OpenSSL CSR, via a PFX file from the CA (with the well-known conversion pitfalls of older encryption algorithms), or by exporting from an ESA. Expiry dates for all roles belong in the certificate inventory, because an expired quarantine page is noticed by end users first.

## Directory and policies

The **LDAP(S) connection** supplies the platform with recipient validation (rejecting invalid addresses during the SMTP dialog), groups for mail policies, and authentication. Policies work on a modular principle: sender and recipient groups meet mail policies with scanning engines and content filters. Established practice is to apply changes first on a policy with a test group, because filtering decisions in live mail flow are hard to take back.

## Operational view

Four topics dominate day-to-day operation: **certificates** with their expiry dates, the **directory connection** including the reachability of the domain controllers, **updates** to AsyncOS with their migration notes, and **monitoring of the mail flow** through message tracking and reporting. When faults occur, the central operational question is always the same: does the appliance accept the message, does it pass it on, and which policy touched it? Message tracking answers that per message; the system logs answer the rest.
