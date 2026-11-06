---
title: "Back up and restore the HIN Mailgateway after an outage"
navTitle: "Backup & Recovery"
description: "A cluster protects the HIN Mailgateway against node failures, but it does not replace a backup. Configuration, key material, restore order, and the changes introduced by Stargate are critical."
date: "2026-07-08"
kategorie: "HIN Gateway"
timeToRead: "15 min read"
themen:
  - hin-gateway
slug: "hin-mail-gateway-backup-disaster-recovery"
translationOf: "hin-mailgateway-backup-disaster-recovery"
translationId: article-845fb4bd0e4c592a
translatedAt: 2026-11-06T03:21:59.974Z
translationReview: required
translationSourceHash: cd8f7c8633d8cf882725ad88feed7be5a1730ce563724481e25b4b9ee2ec84e4
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/hin-mail-gateway-backup-disaster-recovery
translationModel: gpt-5.6-terra
---

# Back up and restore the HIN Mailgateway after an outage

Many production HIN Mailgateways run as a cluster. If one node fails, the other takes over. However, this redundancy does not help against an incorrect rule, a deleted certificate, or a corrupted import: [System-relevant data is replicated to all nodes](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html), including unwanted changes.

For reliable recovery, a separate backup is therefore required. Since the HIN Mailgateway is technically based on a SEPPmail appliance with GINA, its documented backup and restore mechanisms apply.

## What data is stored on the gateway

The gateway processes incoming and outgoing emails according to a central set of rules and encrypts them using S/MIME, OpenPGP, or TLS depending on the recipient; the web-based GINA method is used for recipients without their own key material. For backups, it is crucial that [message contents are not stored persistently on the gateway](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): The appliance processes emails in transit without archiving them.

  

## What the cluster replicates

SEPPmail supports several [cluster variants—high availability, load balancing, and geo-clusters](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html); system parameters, user data, and key material are synchronized across all nodes. In a [frontend/backend cluster, the frontend has no configuration database of its own](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html): It can be operated in a DMZ without storing data and receives only the data required for current processing; the database, including keys, resides on the backend. There is [an exception for Large File Transfer (LFT)](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html): Each partner, including frontends, is assigned an equally sized disk, and LFT data is synchronized to all nodes.

  

## Why replication is not a backup

> *Replication copies the current state, including an incorrect one. A backup preserves a known working state.*

An incorrect import, a deleted key, or a disabled domain is replicated to partner nodes within seconds. Without an independent backup, there is no recovery point afterwards. How closely availability and consistency are linked in a cluster was demonstrated by the [login issues after updating to 15.0.5](/blog/hin-update-issue-version-15.0.5), which were caused by disrupted cluster replication.

  

## What is and is not included in the backup

The [SEPPmail backup is deliberately lean](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): It includes configuration and cryptographic key material only: [no messages, no mail queue, and explicitly no logs](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) (logs should therefore be sent to an external system via Syslog). Since firmware 14.0.0, the appliance creates the backup [automatically at midnight as backup.tgz](https://docs.seppmail.com/de/07_mi_11_adm__administration.html); it can be retrieved via `Download`, `Send Backup` (email to the backup group), or SCP.

| **Included in the backup** | **Not included in the backup** |
| --- | --- |
| System configuration and rule set | Email content / message text |
| User and GINA accounts | Current mail queue |
| Key material: S/MIME, X.509, OpenPGP | System and mail logs (back up externally via Syslog) |
| TLS and certificate configuration | Operating system / VM image |


This means that, because the operating system is not included in the configuration backup, a complete DR strategy also needs a way to restore the appliance base (redeployment from a vendor image or VM snapshot). The configuration backup then restores the configuration and keys.

  

## Snapshots are not cluster backups

Since firmware 14.0.0, the appliance also creates [local snapshots, but only if an LFT partition with a database exists](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). A full snapshot is created on Sundays, with one incremental snapshot each day from Monday through Saturday; retention is 14 days.

A key consideration for DR planning: In cluster operation, these snapshots run in the background, but no restore from them is offered. Snapshots are therefore a local rollback aid on standalone systems, not cluster recovery. The reliable safeguard remains the encrypted configuration backup.

  

## Set up backups

A backup password set under [Administration › Backup › Change password](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) is required for every retrieval method; without it, the backup can neither be downloaded, sent, nor made available via SCP. By default, the nightly backup is emailed to the [“backup (Backup Operator)” group](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html); a dedicated backup user requires a valid internal email address.

-   Set a backup password and [store it separately from the backup](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html): The backup contains private keys.
    
-   For automated storage, [retrieve backups via SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html): Store the public `SSH-RSA`\ key in Administration and retrieve the `backup.tgz` made available at midnight through the OS user `backup`.
    
-   Back up logs separately (external Syslog), as they are [intentionally not part of the backup](https://docs.seppmail.com/de/07_mi_11_adm__administration.html).
    

  

## Backup strategy in cluster operation

In cluster operation, orderly backups and consistent version management are critical.

-   **Daily**: retrieve the encrypted configuration backup [via SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html) and store it externally with versioning
    
-   **Weekly**: full VM or system backup of both nodes, staggered rather than simultaneous (the operating system is not part of the configuration backup)
    
-   **Before maintenance or an update**: stop mail acceptance using [Preempt](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): Incoming emails are then temporarily rejected with a configurable SMTP return code (default `421`); the setting remains active even after a restart.
    

  

Regarding version management: In a frontend/backend cluster, SEPPmail updates [the frontend before the backend](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), and for multi-step updates, all partners must be on the same version before proceeding to the next release. After a major update, regenerating the rule set may be necessary (message: *“Current ruleset created for another version”*).

  

## Restore and disaster recovery

The basic case is straightforward: [Import backup file](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), restart, and the gateway then operates with full functionality. Note the version rule: Only the backup from the immediately preceding firmware version can be imported into the current version (then regenerate the rule set); importing a backup from a newer firmware version into an older version is not possible.

There is an important restriction in a cluster:

-   **Never restore an individual node directly**: A [restore of a single cluster partner is not supported](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). Instead, remove the defective machine from the cluster, set up a new VM, and add it back: Configuration and keys are replicated automatically from the intact partner.
    
-   **Total loss of all nodes**: Redeploy the appliance from the base image, then import the last known working configuration backup and restart.
    

A backup is only as reliable as the last successful restore test. A test restore should be performed in an isolated environment at least twice a year, not against the production cluster.

  

### Restore checklist for an emergency

1.  Remove the defective node from the cluster (do not directly restore a partner).
    
2.  Set up a new VM or, in the event of total loss, provision the appliance from the base image/VM snapshot.
    
3.  Only in the event of total loss: Import the latest working configuration backup (have the password ready and observe the version rule).
    
4.  Test the node in isolation: SMTP acceptance, TLS, GINA, rule set.
    
5.  Add it to the cluster and monitor replication; regenerate the rule set if prompted.
    
6.  Document the incident and update the backup interval and version levels.
    

  

Two maintenance actions require special caution and always a prior backup: [Expanding the LFT partition shuts down the appliance](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), and Factory Reset overwrites the hard drive ten times (the security prompt requires entering the code in reverse order).

  

## What changes with “Stargate”

HIN is gradually replacing the existing Mailgateway with the [new HIN Gateway](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm) (the “Stargate” project, listed by Zug-based [Vereign AG as “Verimesh”](https://www.vereign.com/)). This is not a one-to-one replacement for the appliance, but an architectural change that fundamentally affects backup and disaster recovery:

-   **From centralized to decentralized**: Nodes communicate directly with each other; a central distribution hub is no longer needed.
    
-   **Decentralized Key Management System (DKMS)**: Each organization manages its own cryptographic identity without a central Certificate Authority.
    
-   **End-to-end encryption** with message fragmentation.
    
-   **Resilience from the network**: If one node fails, the mesh remains functional.
    
-   **Open deployment repository**: The [Stargate Deployment](https://code.vereign.com/svdh/stargate-deployment) repository containing the services of a Stargate node is publicly accessible under AGPLv3; the Vereign Client Library (vcl), on the other hand, is now only available after registration.
    

Timeline: The decentralized infrastructure has been [in production in the Swiss healthcare sector since April 2025](https://www.vereign.com/); 2026 is planned for the gradual replacement of the existing Mailgateways and a broad rollout. Organizations with HIN-owned domains (`@hin.ch`, `@verband-hin.ch`) run on HIN infrastructure and are barely affected by the transition.

  

For the operations manual, this means that the traditional discipline of “exporting appliance configuration and keys and restoring them to a replacement node” is becoming less important. It is being replaced by node enrollment, identity and key custody in the mesh, and reconnecting nodes to the network.

  

## The most important distinction

As long as HIN MGW runs on SEPPmail technology, the following applies: The cluster compensates for hardware failures, but responsibility for configuration and key integrity remains with the operator. The lean configuration backup must be secured independently of the cluster (via SCP, versioned, with the password stored separately), snapshots do not replace it in the cluster, versions remain synchronized, and restores are tested regularly in isolation. The transition to Stargate should be incorporated into DR planning early, as it shifts resilience and key custody into the decentralized network.

## Change log

- **October 25, 2026:** Added: The Vereign Client Library is now only available after registration; the Stargate Deployment repository is now listed as the publicly accessible source under AGPLv3.

## Sources

1.  [SEPPmail documentation – “Backup / Restore”](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): Backup contents (configuration and key material only), nightly creation, automatic cluster recovery through replication.
    
2.  [SEPPmail documentation – “Administration”](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): Detailed reference: Backup menu (Download / Send Backup / Change password, `backup.tgz` at midnight), LFT snapshots (14 days, no restore in the cluster), restore rules and cluster procedure, Preempt (SMTP return code, default 421), Device Cloning, update channels and update order (frontend before backend), Factory Reset, bulk import/export.
    
3.  [SEPPmail documentation – “Create backup user”](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html): “backup (Backup operator)” group, encryption, and password management.
    
4.  [SEPPmail documentation – “Copy backup via SCP”](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html): Retrieving `backup.tgz` via SCP through the OS user `backup` instead of sending it by email.
    
5.  [SEPPmail documentation – “Cluster / High Availability”](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): Cluster types and data synchronized across all nodes (system parameters, user data, key material).
    
6.  [SEPPmail documentation – “Frontend/backend cluster”](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html): Frontend without a configuration database, DMZ operation, on-demand data; backend as the data holder.
    
7.  [SEPPmail documentation – “Data storage in the cluster (LFT)”](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html): Equally sized additional disk per partner, synchronization of LFT data to all nodes.
    
8.  [HIN AG – “From Mailgateway to Data Mesh”](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): HIN communication about Stargate as the successor: decentralized nodes, Data Mesh concept, timeline, end-to-end encryption.
    
9.  [Vereign AG – “Verimesh”](https://vereign.com/): Verimesh (formerly the Stargate project) in production use in the Swiss healthcare sector since April 2025.
    
10. [Vereign GitLab – Stargate Deployment](https://code.vereign.com/svdh/stargate-deployment): Public deployment repository under AGPLv3 containing the services of a Stargate node and a link to the [deployment documentation](https://health-info-net-ag.github.io/Stargate-deployment/).
