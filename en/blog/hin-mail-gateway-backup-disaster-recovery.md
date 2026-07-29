---
title: "Backing up and restoring the HIN Mail Gateway after a failure"
navTitle: "Backup & Recovery"
description: "A cluster protects the HIN Mail Gateway against node failures, but it does not replace a backup. Configuration, key material, restore sequence and the changes introduced by Stargate are crucial."
date: "2026-07-08"
kategorie: "HIN Gateway"
timeToRead: "15 min read"
themen:
  - hin-gateway
slug: "hin-mail-gateway-backup-disaster-recovery"
translationOf: "hin-mailgateway-backup-disaster-recovery"
url: "https://rafaelpfister.ch/en/blog/hin-mail-gateway-backup-disaster-recovery"
translationId: article-845fb4bd0e4c592a
translatedAt: 2026-07-28T11:10:30.445Z
translationReview: automatic
translationSourceHash: 39ecd30339131eb74d0748f4bfb31ead3f98aefbd47621974b1e032f1a96b345
---

# Backing up and restoring the HIN Mail Gateway after a failure

Many production HIN Mail Gateways run as a cluster. If one node fails, the other takes over. However, this redundancy does not help against an incorrect rule, a deleted certificate or a corrupted import: [System-relevant data is replicated to all nodes](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html), including unwanted changes.

A separate backup is therefore needed for reliable recovery. As the HIN Mail Gateway is technically based on a SEPPmail appliance with GINA, its documented backup and restore mechanisms apply.

## What data is stored on the gateway

The gateway processes incoming and outgoing emails according to a central rule set and encrypts them using S/MIME, OpenPGP or TLS depending on the recipient; the web-based GINA method is used for recipients without their own key material. The key point for backups is that [message content is not stored persistently on the gateway](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): the appliance processes emails in transit without archiving them.

  

## What the cluster replicates

SEPPmail supports several [cluster variants – high availability, load balancing and geo-clusters](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html); system parameters, user data and key material are synchronised across all nodes. In a [frontend/backend cluster, the frontend has no configuration database of its own](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html): it can be operated in a DMZ without storing data and receives only the data required for current processing; the database, including keys, resides on the backend. There is an exception for [Large File Transfer (LFT)](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html): each partner, including frontends, is assigned an equally sized disk, and LFT data is synchronised across all nodes.

  

## Why replication is not a backup

> *Replication copies the current state, including an erroneous one. A backup preserves a known working state.*

An incorrect import, a deleted key or a disabled domain is replicated to partner nodes within seconds. Without an independent backup, there is then no recovery point left. How closely availability and consistency are linked in the cluster became apparent with the [login problems after the update to 15.0.5](/blog/hin-update-issue-version-15.0.5), which were caused by disrupted cluster replication.

  

## What is and is not included in the backup

The [SEPPmail backup is deliberately lean](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): it includes only configuration and cryptographic key material: [no messages, no mail queue and explicitly no logs](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) (logs should therefore be sent to an external system via Syslog). Since firmware 14.0.0, the appliance creates the backup [automatically at midnight each night as backup.tgz](https://docs.seppmail.com/de/07_mi_11_adm__administration.html); it can be retrieved via `Download`, `Send Backup` (email to the backup group) or SCP.

| **Included in the backup** | **Not included in the backup** |
| --- | --- |
| System configuration and rule set | Email content / message text |
| User and GINA accounts | Current mail queue |
| Key material: S/MIME, X.509, OpenPGP | System and mail logs (back up externally via Syslog) |
| TLS and certificate configuration | Operating system / VM image |


Consequently, because the operating system is not included in the configuration backup, a complete DR strategy also requires a way to restore the appliance base system (redeployment from the vendor image or a VM snapshot). The configuration backup then restores the configuration and keys.

  

## Snapshots are not cluster backups

Since firmware 14.0.0, the appliance also creates [local snapshots, but only if an LFT partition with a database is present](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). A full snapshot is created on Sundays, with one incremental snapshot each from Monday to Saturday; the retention period is 14 days.

Crucial for DR planning: although these snapshots run in the background in cluster operation, no restore is offered from them. Snapshots are therefore a local rollback aid on standalone systems, not cluster recovery. The reliable safeguard remains the encrypted configuration backup.

  

## Setting up backups

A backup password must be set under [Administration › Backup › Change password](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) for every retrieval method; without this password, the backup can neither be downloaded, sent nor provided via SCP. By default, the nightly backup is emailed to the [“backup (Backup Operator)” group](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html); a dedicated backup user requires a valid internal email address.

-   Set a backup password and [store it separately from the backup](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html): the backup contains private keys.
    
-   For automated storage, [retrieve backups via SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html): store the public `SSH-RSA`\-keys in Administration and download the `backup.tgz` made available at midnight using the OS user `backup`.
    
-   Back up logs separately (external Syslog), as they are [deliberately not part of the backup](https://docs.seppmail.com/de/07_mi_11_adm__administration.html).
    

  

## Backup strategy in cluster operation

In cluster operation, orderly backups and consistent version management are crucial.

-   **Daily**: retrieve the encrypted configuration backup [via SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html) and store it externally with versioning
    
-   **Weekly**: full VM or system backup of both nodes, staggered rather than simultaneous (the operating system is not part of the configuration backup)
    
-   **Before maintenance or an update**: stop accepting mail using [Preempt](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): incoming emails are then temporarily rejected with a configurable SMTP return code (default `421`); the setting remains active even after a restart.
    

  

Regarding version management: in a frontend/backend cluster, SEPPmail updates [the frontend before the backend](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), and with multi-stage updates all partners must be on the same version before moving to the next release. After a major update, it may be necessary to regenerate the rule set (message: *“Current ruleset created for another version”*).

  

## Restore and disaster recovery

The basic case is straightforward: [Import backup file](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), restart, and the gateway then operates with full functionality. The version rule must be observed: only the backup from the immediately preceding firmware version can be imported into the current one (then regenerate the rule set); importing a backup from newer firmware into an older version is not possible.

An important restriction applies in a cluster:

-   **Never restore an individual node directly**: A [restore of an individual cluster partner is not intended](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). Instead, remove the faulty machine from the cluster, set up a new VM and add it again: configuration and keys are automatically replicated from the intact partner.
    
-   **Total loss across all nodes**: Redeploy the appliance from the base image, then import the last known working configuration backup and restart.
    

A backup is only as reliable as the last successful restore test. A test restore should be performed in an isolated environment at least twice a year, not against the production cluster.

  

### Restore checklist for an emergency

1.  Remove the faulty node from the cluster (do not directly restore a partner).
    
2.  Set up a new VM or, in the event of total loss, provide the appliance from the base image/VM snapshot.
    
3.  Only in the event of total loss: import the last working configuration backup (have the password ready and observe the version rule).
    
4.  Check the node in isolation: SMTP acceptance, TLS, GINA, rule set.
    
5.  Add it to the cluster and monitor replication; regenerate the rule set if prompted.
    
6.  Document the incident and update the backup interval and version levels.
    

  

Two maintenance operations require particular caution and always a prior backup: [expanding the LFT partition shuts down the appliance](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), and a factory reset overwrites the hard disk ten times (the security prompt requires the code in reverse order).

  

## What changes with “Stargate”

HIN is gradually replacing the existing Mail Gateway with the [new HIN Gateway](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm) (the “Stargate” project, operated by Zug-based [Vereign AG as “Verimesh”](https://www.vereign.com/)). This is not a one-to-one replacement for the appliance, but an architectural shift that fundamentally affects backup and disaster recovery:

-   **From centralised to decentralised**: nodes communicate directly with one another; there is no central distribution hub.
    
-   **Decentralised key management (DKMS)**: each organisation manages its own cryptographic identity, without a central certificate authority.
    
-   **End-to-end encryption** with message fragmentation.
    
-   **Resilience from the network**: if one node fails, the mesh remains operational.
    
-   **Open reference implementation**: the [Vereign Client Library (vcl)](https://code.vereign.com/code/vcl/-/tree/0.4-rc1) is open source under AGPLv3.
    

Timeline: the decentralised infrastructure has been [in production use in Swiss healthcare since April 2025](https://www.vereign.com/); 2026 is planned to see the gradual replacement of existing Mail Gateways and a broad rollout. Organisations with HIN-owned domains (`@hin.ch`, `@verband-hin.ch`) run on HIN infrastructure and are barely affected by the transition.

  

For the operations manual, this means that the traditional discipline of “exporting appliance configuration and keys and restoring them to a replacement node” becomes less important. It is replaced by node enrolment, identity and key custody in the mesh, and rejoining nodes to the network.

  

## The key distinction

As long as the HIN MGW runs on SEPPmail technology, the following applies: the cluster compensates for hardware failures, but responsibility for configuration and key integrity remains with the operator. The lean configuration backup must be secured independently of the cluster (via SCP, versioned, with the password stored separately), snapshots do not replace it in the cluster, versions must remain synchronised, and restores must be tested regularly in isolation. The move to Stargate should be incorporated into DR planning at an early stage, as it shifts resilience and key custody into the decentralised network.

## Sources

1.  [SEPPmail documentation – “Backup / Restore”](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): Backup contents (configuration and key material only), nightly creation, automatic cluster recovery through replication.
    
2.  [SEPPmail documentation – “Administration”](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): Detailed reference: Backup menu (Download / Send Backup / Change password, `backup.tgz` at midnight), LFT snapshots (14 days, no restore in the cluster), restore rules and cluster procedure, Preempt (SMTP return code, default 421), device cloning, update channels and update sequence (frontend before backend), factory reset, bulk import/export.
    
3.  [SEPPmail documentation – “Create backup user”](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html): “backup (Backup operator)” group, encryption and password management.
    
4.  [SEPPmail documentation – “Copy backup via SCP”](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html): Retrieving `backup.tgz` via SCP using the OS user `backup` instead of sending it by email.
    
5.  [SEPPmail documentation – “Cluster / High availability”](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): Cluster types and data synchronised across all nodes (system parameters, user data, key material).
    
6.  [SEPPmail documentation – “Frontend/backend cluster”](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html): Frontend without a configuration database, DMZ operation, on-demand data; backend as data holder.
    
7.  [SEPPmail documentation – “Data storage in the cluster (LFT)”](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html): Equally sized additional disk for each partner, synchronisation of LFT data across all nodes.
    
8.  [HIN AG – “From Mail Gateway to Data Mesh”](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): HIN communication on the Stargate successor: decentralised nodes, data mesh concept, timeline, end-to-end encryption.
    
9.  [Vereign AG – “Verimesh” / Vereign Client Library (vcl, tag 0.4-rc1)](https://code.vereign.com/code/vcl/-/tree/0.4-rc1): Technical foundation of Stargate: decentralised key management (DKMS), end-to-end encryption with message fragmentation, open source under AGPLv3.
