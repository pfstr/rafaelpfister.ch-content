---
title: "Backup and recovery: bringing mail infrastructure back online"
blatt: "backup-dr"
description: "What really has to be backed up in mail systems: configurations of gateways and appliances, key material, RTO and RPO as planning parameters, and why a backup without a restore test is not a backup."
fakten:
  - { label: "Purpose", wert: "Resuming operation after an outage, operator error, or compromise" }
  - { label: "Planning parameters", wert: "RTO (recovery time) and RPO (tolerated data loss)" }
  - { label: "Gateway specifics", wert: "Configuration and key material instead of payload data" }
  - { label: "Reference", wert: "NIST SP 800-34 (Contingency Planning)", href: "https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final" }
  - { label: "Rule of thumb", wert: "A backup only becomes a backup after a successful restore" }
  - { label: "Storage", wert: "separate from the system, versioned, encrypted" }
werbung: ["newsletter"]
ctaThemen: ["hin-gateway"]
---

# Backup and recovery: bringing mail infrastructure back online

Backup discussions usually revolve around mailboxes and databases. In mail infrastructure the actual risk lies elsewhere: in the systems in between. An encryption gateway that has to be rebuilt after a hardware failure is a total loss without a backed-up configuration and without key material, even if not a single mailbox was lost. Decryptable historical messages, installed certificates, policies, LDAP connections: all gone.

## What really has to be backed up

For every system in the mail path, the list includes the **configuration** (policies, connectors, transport rules, user and group mappings), the **key material** (private keys of the TLS certificates, S/MIME domain certificates, PGP keyrings), and the **operational data** that cannot be reconstructed from other sources. For appliances this means using the vendor's own configuration export, regularly and automatically, and storing the export password separately. A gateway backup whose password went down with the gateway is decoration.

The counter-list is equally important: what is deliberately **not** in the backup because it lives elsewhere? Messages themselves reside in the mail system, users in the directory, DNS at the registrar. Recovery means reconnecting these sources, not duplicating them.

## RTO and RPO instead of gut feeling

Two parameters make recovery plannable. **RPO** (recovery point objective): how much configuration state may be lost? For a gateway whose policies change weekly, a daily configuration backup is comfortably sufficient. **RTO** (recovery time objective): how long may mail flow be interrupted? This number determines the architecture: is a reinstallation plus configuration import enough, is a standby appliance required, or is a cluster needed in which the failure of a single node is no longer a recovery scenario at all? Once RTO and RPO have been agreed with the business, budget discussions about redundancy become considerably more relaxed.

## The restore is the test

Backups fail silently: empty exports, expired credentials, changed formats after a firmware update. The old saying therefore holds literally: a backup is only a backup once the restore has been proven. An annual test is practical: import the configuration export onto a test instance or a virtual appliance, verify the LDAP bind and mail flow, and document any deviations. Such a test also surfaces the undocumented manual steps that otherwise exist only in one person's head, and precisely those steps belong in the recovery document: sequence, dependencies (directory first, then gateway, then connectors), and vendor contacts.

## Accounting for compromise

Conventional backups protect against outages and operator error, but not automatically against an attacker with administrative rights. A resilient concept therefore includes immutable backups, or at least separately authenticated ones, a copy outside the production domain, and the assumption that the backup system itself can be a target. Recovery then needs a path that works without the compromised environment: documented credentials outside the affected directory service, a clean target system, and a sequence that restores identity and network before the applications.
