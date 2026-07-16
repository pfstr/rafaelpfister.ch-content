---
title: "HIN Mail Gateway Based on SEPPmail: Backup & Disaster Recovery in a Cluster—and What's Changing with Stargate"
navTitle: "Backup & Recovery"
description: "The HIN mail gateway is based on a SEPPmail appliance. For backup and disaster recovery, the key factors are what the appliance backs up (only configuration and key material, not emails), how cluster replication works, and why it does not replace a backup."
date: "2026-07-08"
kategorie: "HIN-Gateway"
timeToRead: "15 min to read"
themen:
  - "hin-gateway"
slug: "hin-mail-gateway-backup-disaster-recovery"
translationOf: "hin-mailgateway-backup-disaster-recovery"
url: "https://rafaelpfister.ch/en/blog/hin-mail-gateway-backup-disaster-recovery"
---

# HIN Mail Gateway Based on SEPPmail: Backup & Disaster Recovery in a Cluster—and What's Changing with Stargate

Almost every production HIN Mail Gateway (MGW) runs in a cluster—and some operators tacitly assume that this redundancy also handles data backup. That is a fallacy. A cluster protects against the failure of a node, not against a faulty rule change, a deleted certificate, or a corrupted import—because [System-critical data is reliably replicated to all nodes](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html), errors included.

  
  
  

Backup and disaster recovery can only be planned once the technical foundation is clear. The HIN Mail Gateway is based on a SEPPmail appliance using the GINA method; therefore, the documented SEPPmail mechanism applies, which has some specific characteristics when it comes to backup.

  
  
  

## What the HIN MGW Is, Technically Speaking

The gateway processes incoming and outgoing emails according to a set of central rules and encrypts them using S/MIME, OpenPGP, or TLS, depending on the recipient; for recipients without their own key material, the web-based GINA method is used. For backup purposes, it is crucial that [Message content is not stored persistently on the gateway](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): The appliance processes emails as they come in without archiving them.

  
  
  

## Cluster Architecture: What Is Replicated

SEPPmail supports several [Cluster Configurations – High Availability, Load Balancing, and Geo-Clusters](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html); system parameters, user data, and key material are synchronized across all nodes. When [In a front-end/back-end cluster, the front end does not have its own configuration database](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html): It can be run in a DMZ without data storage and receives only the data necessary for the current processing; the database, including the keys, is located on the backend. For [Large File Transfer (LFT) is an exception](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html): Each partner—including front-ends—is assigned a disk of equal size, and the LFT data is synchronized across all nodes.

  
  
  

## Why Replication Is Not a Backup

> *Replication copies the current state—even if it is faulty. A backup preserves a known, functioning state.*

A failed import, a deleted key, or a deactivated domain is replicated to the partner nodes within seconds. Without an independent backup, there is no longer a restore point. Just how closely availability and consistency are linked within the cluster became evident during the [Login Issues After Updating to 15.0.5](/en/blog/hin-mailgateway-update-15-0-5-login-issue), which were triggered by a failure in cluster replication.

  
  
  

## What's Included in the Backup—and What Isn't

The [SEPPmail Backup is intentionally streamlined](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): It consists exclusively of configuration and cryptographic key material – [No messages, no mail queue, and definitely no logs](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) (Logs should therefore be sent to an external system via Syslog.) Starting with firmware version 14.0.0, the appliance creates the backup [automatically at midnight as backup.tgz](https://docs.seppmail.com/de/07_mi_11_adm__administration.html); it can be accessed via `Download`, `Send Backup` (Email to the backup group) or SCP.

| Included in the backup | Not in the backup |
| --- | --- |
| System configuration and rule set | Email contents / message bodies |
| User and GINA accounts | Current mail queue |
| Key material: S/MIME, X.509, OpenPGP | System and mail logs (secure externally via syslog) |
| TLS and certificate configuration | Operating system / VM image |

It follows that, because the operating system is not included in the configuration backup, a comprehensive DR strategy must also include a way to restore the appliance base (by redeploying from a manufacturer image or a VM snapshot). The configuration backup then restores the configuration and keys.

  
  
  

## Snapshots are not cluster backups

Starting with firmware version 14.0.0, the appliance also creates [local snapshots—but only if there is an LFT partition containing a database](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). A full snapshot is created on Sundays, and an incremental snapshot is created each day from Monday through Saturday; snapshots are retained for 14 days.

Crucial for DR planning: Although these snapshots run in the background in cluster mode, **no restore is offered from them**. Snapshots are therefore a local rollback tool for individual systems, not a cluster recovery solution. The encrypted configuration backup remains the most reliable backup method.

  
  
  

## Set Up a Backup

A prerequisite for any retrieval method is that a backup password has been set under [Administration › Backup › Change password](https://docs.seppmail.com/de/07_mi_11_adm__administration.html); without this password, nothing will be downloaded, sent, or made available via SCP. By default, the nightly backup is sent via email to the ["backup (Backup Operator)" group](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html); a dedicated backup user must have a valid internal email address.

-   Set a backup password and [Store separately from the backup](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html) – The backup contains private keys.
    
-   For automated retrieval, [fetch the backups via SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html): store the public `SSH-RSA` key in the administration interface and use the OS user `backup` to pull the `backup.tgz` made available at midnight.
    
-   Back up logs separately (external syslog), since they [deliberately excluded from the backup](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) are.
    

  
  
  

## Backup Strategy in a Cluster Environment

In a cluster environment, orderly backups and consistent version control are crucial.

-   **Every day**: Encrypted configuration backup [per SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html) Retrieve and store externally with versioning
    
-   **Weekly**: Full VM or system backup of both nodes, performed at different times rather than simultaneously (the operating system is not included in the configuration backup)
    
-   **Before Maintenance or an Update**: Email submissions via [Stop Preempt](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) – Incoming emails are then marked with a configurable SMTP return code (default `421`) temporarily rejected; the setting remains active even after a restart.
    

  
  
  

About Version Management: SEPPmail Updates in the Frontend/Backend Cluster [the front end before the back end](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), and for multi-stage updates, all partners must be at the same version before upgrading to the next release. After a major update, it may be necessary to regenerate the ruleset (message *«Current ruleset created for another version»*).

  
  
  

## Restore und Disaster Recovery

The basic case is straightforward: [Import backup file](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), Restart—afterward, the gateway will be fully functional. Please note the version rule: Only the backup of the **immediately preceding** You can update the firmware to the current version (and then regenerate the ruleset); however, it is not possible to restore a backup of a newer firmware version to an older one.

There is one important restriction within the cluster:

-   **Never restore individual nodes directly**: One [Restoring a single cluster partner is not supported](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). Instead, remove the faulty machine from the cluster, set up a new VM, and add it back—the configuration and keys are automatically replicated from the healthy partner.
    
-   **Total loss on all nodes**: Re-deploy the appliance from the base image, then import the last known working configuration backup and restart it.
    

A backup is only as reliable as the last successful restore test. A test restore should be performed at least twice a year in an isolated environment, not against the production cluster.

  
  
  

### Restore Checklist for Emergencies

1.  Remove a faulty node from the cluster (do not perform a direct restore of a partner).
    
2.  Set up a new VM or, in the event of a total loss, provision the appliance from a base image or VM snapshot.
    
3.  Only in the event of total loss: Import the last working configuration backup (have your password ready; follow the version rules).
    
4.  Test nodes in isolation: SMTP acceptance, TLS, GINA, rule set.
    
5.  Add it to the cluster and monitor replication; if an error occurs, regenerate the ruleset.
    
6.  Document the incident and verify the backup interval and version statuses.
    

  
  
  

Two maintenance tasks require special caution and should always be preceded by a backup: The [Expanding the LFT partition shuts down the appliance](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), and the factory reset overwrites the hard drive ten times (the security prompt requires the code to be entered in reverse order).

  
  
  

## What's Changing with "Stargate"

HIN is gradually replacing the existing mail gateway with the [New HIN Gateway](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm) (The "Stargate" project, in Zug [Vereign AG as “Verimesh”](https://www.vereign.com/) (implemented). This is not a 1:1 replacement of the appliance, but rather an architectural change that fundamentally affects backup and disaster recovery:

-   **From Centralized to Decentralized**: Nodes communicate directly with one another; there is no central distribution center.
    
-   **Decentralized Key Management (DKMS)**: Each organization manages its own cryptographic identity without a central certificate authority.
    
-   **End-to-end encryption** with fragmented news reports.
    
-   **Resilience from the Web**: If a node fails, the mesh remains functional.
    
-   **Open Reference Implementation**: The [Vereign Client Library (vcl)](https://code.vereign.com/code/vcl/-/tree/0.4-rc1) is available as open source under the AGPLv3.
    

Timeline: Decentralized infrastructure in the Swiss healthcare system [In productive use since April 2025](https://www.vereign.com/); the phased replacement of the existing mail gateways and a broad rollout are planned for 2026. Organizations with HIN-owned domains (`@hin.ch`, `@verband-hin.ch`) run on HIN infrastructure and are hardly affected by the transition.

  
  
  

For the operations manual, this means that the traditional procedure of “configuring the appliance, exporting keys, and restoring them to a replacement node” is becoming less important. It is being replaced by node enrollment, identity and key management within the mesh, and the reintegration of nodes into the network.

  
  
  

## Conclusion

As long as the HIN MGW runs on SEPPmail technology, the following applies: The cluster compensates for hardware failures, but responsibility for configuration and key integrity remains with the operator. The lean configuration backup must be backed up independently of the cluster (via SCP, versioned, with a separately stored password); snapshots do not replace it within the cluster, the version states remain synchronized, and the restore is regularly tested in isolation. The transition to Stargate should be incorporated into disaster recovery (DR) planning at an early stage, as it shifts resilience and key storage to the decentralized network.

## Sources

1.  [SEPPmail Documentation – “Backup / Restore”](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html) — Backup content (configuration and key material only), nightly backups, automatic cluster recovery via replication.
    
2.  [SEPPmail Documentation – “Administration”](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) — Detailed reference: Backup menu (Download / Send Backup / Change password, `backup.tgz` at midnight), LFT snapshots (14 days, no restore in the cluster), restore rules and cluster procedures, preemption (SMTP return code, default 421), device cloning, update channels and update order (frontend before backend), factory reset, bulk import/export.
    
3.  [SEPPmail Documentation – “Create a Backup User”](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html) — "backup (Backup operator)" group, encryption, and password management.
    
4.  [SEPPmail Documentation – “Copying a Backup via SCP”](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html) — Pickup of the `backup.tgz` via SCP using the OS user `backup` instead of sending an email.
    
5.  [SEPPmail Documentation – “Cluster / High Availability”](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html) — Cluster types and the data synchronized across all nodes (system parameters, user data, key material).
    
6.  [SEPPmail Documentation – “Frontend/Backend Cluster”](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html) — Front end without a configuration database, DMZ operation, on-demand data; back end as a data store.
    
7.  [SEPPmail Documentation – “Data Storage in the Cluster (LFT)”](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html) — An additional plate of equal size for each partner; synchronization of LFT data across all nodes.
    
8.  [HIN AG – “From Mail Gateway to Data Mesh”](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm) — HIN Communication on the Successor to Stargate: Decentralized Nodes, Data Mesh Concept, Timeline, End-to-End Encryption.
    
9.  [Vereign AG – “Verimesh” / Vereign Client Library (vcl, Tag 0.4-rc1)](https://code.vereign.com/code/vcl/-/tree/0.4-rc1) — Technical foundation of Stargate: decentralized key management (DKMS), end-to-end encryption with message fragmentation, open source under AGPLv3.
