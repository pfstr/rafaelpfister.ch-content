---
title: "HIN Mailgateway 15.0.5: Fixing login failures after the cluster update"
navTitle: "Login error 15.0.5"
description: "After updating an HIN Mailgateway cluster to version 15.0.5, login fails on both nodes after a few minutes. This procedure restores the appliances to operation in a controlled manner."
date: "2026-06-19"
kategorie: "HIN Gateway"
timeToRead: "3 min read"
themen:
  - "hin-gateway"
slug: "hin-mailgateway-update-15-0-5-login-issue"
translationOf: "hin-update-issue-version-15.0.5"
draft: false
url: "https://rafaelpfister.ch/en/blog/hin-mailgateway-update-15-0-5-login-issue"
---

# HIN Mailgateway 15.0.5: Fixing login failures after the cluster update

When updating an HIN Mailgateway from 14.1.4.2 to 15.0.5, an error in cluster replication can prevent login on both appliances. Individual systems are not affected. The manufacturer is aware of the issue and plans a fix for a future version.

**Update from July 29, 2026:** The announced fix has arrived. Patch release 15.0.6 suppresses password rehashing when cluster members run different firmware versions — exactly the constellation that triggered the outage described here. The assessment is in the article on [SEPPmail 15.0.6 and 15.0.6.1](/en/blog/seppmail-releases-15-0-6-and-15-0-6-1); the recovery procedure below remains relevant for clusters still updating to 15.0.5.

## Symptoms

Immediately after the update, the web interface can still be opened. Around ten minutes later, login fails on both cluster nodes. The fact that the issue occurs after a delay and on both systems indicates that the replicated cluster configuration is the cause.

## Recovery

The following steps modify the cluster configuration. Current backups and the cluster identifier must be available beforehand.

1. Restore the snapshots of both cluster nodes that were created at the same time.
2. After the restore, leave one node switched off.
3. On the running node, first download the cluster identifier and then dissolve the cluster.
4. Warning: Once dissolved, the appliance restarts immediately without further confirmation.

![](../images/hin-update-issue-version-15.0.5/YSaXyzS9jLOD9utH0H2AEDOdnjI.png)

5. Update the first node to version 15.0.5 and then shut it down.
6. Start the second node and repeat the same steps there.
7. Only once both systems work individually and have the same version, rebuild the cluster according to the manufacturer’s documentation.

This procedure prevents a faulty configuration from being replicated between the nodes again during the update.

## Sources

1. [SEPPmail documentation – “Cluster / High Availability”](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): Cluster types and replication of the configuration across all nodes.
2. [SEPPmail documentation – “Administration”](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): Update sequence in the cluster (frontend before backend) and the requirement for identical versions.
3. [HIN Mailgateway: Backup & Disaster Recovery in the Cluster](/blog/hin-mailgateway-backup-disaster-recovery): In-depth discussion of cluster replication, backup and restore.
