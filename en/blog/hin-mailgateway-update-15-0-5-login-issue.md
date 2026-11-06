---
title: "HIN Mailgateway 15.0.5: Fixing login failures after a cluster update"
navTitle: "Login error 15.0.5"
description: "After updating an HIN Mailgateway cluster to version 15.0.5, logins fail on both nodes after a few minutes. This procedure restores the appliances to operation in a controlled manner."
date: "2026-06-19"
kategorie: "HIN Gateway"
timeToRead: "3 min read"
themen:
  - hin-gateway
slug: "hin-mailgateway-update-15-0-5-login-issue"
translationOf: "hin-update-issue-version-15.0.5"
draft: false
translationId: article-bd1908eec39f9c26
translatedAt: 2026-11-06T03:28:36.145Z
translationReview: automatic
translationSourceHash: 126aded044bfb102e020aaf45b7b77f6658f77a6de88a71d1554b1fabeefce2a
translationModel: gpt-5.6-terra
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/hin-mailgateway-update-15-0-5-login-issue
---

# HIN Mailgateway 15.0.5: Fixing login failures after a cluster update

When updating an HIN Mailgateway from 14.1.4.2 to 15.0.5, an error in cluster replication can cause logins to fail on both appliances. Individual systems are not affected. The vendor is aware of the issue and plans a fix for a subsequent version.

**Update of July 29, 2026:** The announced fix is now available. Patch release 15.0.6 suppresses password rehashing when cluster members run different firmware versions. This is precisely the scenario that triggered the outage described here. The classification is provided in the article on [SEPPmail 15.0.6 and 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1); the following recovery procedure remains relevant for clusters that are still updating to 15.0.5.

## Symptoms

Immediately after the update, the web interface can still be opened. About ten minutes later, login fails on both cluster nodes. The fact that the error occurs with a delay and on both systems points to the replicated cluster configuration as the cause.

## Recovery

The following steps modify the cluster configuration. Current backups and the cluster identifier must be available beforehand.

1. Restore the snapshots of both cluster nodes that were created at the same time.
2. After the restore, leave one node powered off.
3. On the running node, first download the cluster identifier and then dissolve the cluster.
4. Warning: After dissolving it, the appliance restarts immediately without further confirmation.

![](../images/hin-update-issue-version-15.0.5/YSaXyzS9jLOD9utH0H2AEDOdnjI.png)

5. Update the first node to version 15.0.5 and then shut it down.
6. Start the second node and repeat the same steps there.
7. Only once both systems work individually and have the same version, rebuild the cluster according to the vendor documentation.

This procedure prevents a faulty configuration from being replicated between the nodes again during the update.

## Change log

- **October 25, 2026:** Corrected: According to the SEPPmail documentation, the update order in the cluster is backend before frontend, not the other way around.

## Sources

1. [SEPPmail documentation – “Cluster / High Availability”](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): Cluster types and replication of the configuration across all nodes.
2. [SEPPmail documentation – “Administration”](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): Update order in the cluster (backend before frontend) and the requirement for identical version levels.
3. [HIN Mailgateway: Backup & Disaster Recovery in the cluster](/blog/hin-mailgateway-backup-disaster-recovery): An in-depth look at cluster replication, backup, and restore.
