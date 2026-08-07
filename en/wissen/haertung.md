---
title: "Hardening and access protection: reducing attack surface in operations"
blatt: "haertung"
description: "The principles behind every hardening checklist: minimize the attack surface, use central instead of local accounts, grant minimal privileges, apply updates consistently, and keep access traceable, from the appliance GUI to the VPS."
fakten:
  - { label: "Goal", wert: "Minimize attack surface and potential damage per system" }
  - { label: "Principles", wert: "Least privilege, central authentication, service minimization" }
  - { label: "Reference baselines", wert: "CIS Benchmarks", href: "https://www.cisecurity.org/cis-benchmarks" }
  - { label: "Complementary", wert: "BSI IT-Grundschutz, vendor hardening guides" }
  - { label: "Typical levers", wert: "MFA, keys instead of passwords, firewall zones, logging" }
  - { label: "Related", wert: "SSH hardening, LDAP admin integration", href: "/en/kb/ssh" }
werbung: ["newsletter"]
ctaThemen: ["seppmail", "claude"]
---

# Hardening and access protection: reducing attack surface in operations

Hardening is not a one-time checklist but a way of thinking: every system offers exactly as much attack surface as it has services, accounts, and reachable interfaces. Keeping these three quantities consistently small accomplishes the largest part of the work before the first specialized tool is installed.

## Access: central beats local

Local accounts are maintenance problem number one: separate passwords per system, no central offboarding, no enforced policies. Wherever a system allows it, authentication belongs to the central directory, whether an appliance GUI via LDAP, a server via Kerberos, or SaaS via SSO. The benefit is concrete: a departure locks all access at once, and the password policy applies everywhere. Added to this are **MFA for everything exposed** and **keys instead of passwords** wherever the protocols support it. Emergency accounts (break-glass) remain the documented, monitored exception.

## Privileges: as few as necessary

**Least privilege** in daily practice means administration through personal accounts with elevatable rights instead of shared superusers, separate accounts for daily work and administration, and service accounts that are permitted exactly one task. Modern platforms support this directly, for example with granular app permissions instead of full access. The test is the question: what could this account do if it were compromised today?

## Surface: services and reachability

Every listening port is attack surface. Therefore: disable services that are not needed, never expose management interfaces directly to the internet (place them behind a VPN, a jump host, or IP filtering), and segment network zones so that one compromised system does not automatically reach all the others. For internet-exposed systems the rules are stricter: minimal services, throttled login attempts, prompt updates, because that is where the constant barrage of scans arrives.

## Traceability: seeing what happens

Hardening without logging is flying blind. Sign-ins (failed ones in particular), privilege changes, and configuration changes belong in logs and ideally in central collection so that they survive an incident. The CIS Benchmarks and the vendor hardening guides supply the system-specific details; the principles above are the yardstick against which each individual recommendation can be measured.

