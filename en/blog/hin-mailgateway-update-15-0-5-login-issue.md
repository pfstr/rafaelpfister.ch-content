---
title: "HIN Mailgateway 15.0.5: Fixing Login Failures After a Cluster Update"
navTitle: "Login Failure 15.0.5"
description: "After updating an HIN Mailgateway cluster to version 15.0.5, login fails on both nodes after a few minutes. This procedure restores the appliances to operation in a controlled manner."
date: "2026-06-19"
kategorie: "HIN Gateway"
timeToRead: "3 min read"
themen:
  - hin-gateway
slug: "hin-mailgateway-update-15-0-5-login-issue"
translationOf: "hin-update-issue-version-15.0.5"
draft: false
translationId: article-bd1908eec39f9c26
translatedAt: 2026-09-05T08:02:38.871Z
translationReview: automatic
translationSourceHash: 03805a55e2acb45a3453acda621e39cc3ddce6ae00cfe85f76ca2cc322cde6fa
translationModel: gpt-5.6-terra
url: https://rafaelpfister.ch/en/blog/hin-mailgateway-update-15-0-5-login-issue
---

# HIN Mailgateway 15.0.5: Fixing Login Failures After a Cluster Update

When updating an HIN Mailgateway from 14.1.4.2 to 15.0.5, an error in cluster replication can cause login failures on both appliances. Standalone systems are not affected. The manufacturer is aware of the issue and plans a fix for a future version.

**Update from July 29, 2026:** The announced fix is now available. Patch release 15.0.6 suppresses password rehashing when cluster members run different firmware versions. This is exactly the scenario that triggered the failure described here. Context is available in the article on [SEPPmail 15.0.6 and 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1); the following recovery procedure remains relevant for clusters that are still being updated to 15.0.5.

## Symptoms

Immediately after the update, the web interface can still be opened. About ten minutes later, login fails on both cluster nodes. The fact that the error occurs after a delay and on both systems points to the replicated cluster configuration as the cause.

## Recovery

The following steps modify the cluster configuration. Current backups and the cluster identifier must be available beforehand.

1. Restore the snapshots of both cluster nodes that were created at the same time.
2. After the restore, leave one node powered off.
3. On the running node, first download the cluster identifier and then dissolve the cluster.
4. Warning: After dissolving it, the appliance immediately reboots without further confirmation.

![](../images/hin-update-issue-version-15.0.5/YSaXyzS9jLOD9utH0H2AEDOdnjI.png)

5. Update the first node to version 15.0.5 and then shut it down.
6. Start the second node and repeat the same steps there.
7. Only once both systems work individually and have the same version, rebuild the cluster according to the manufacturer’s documentation.

This procedure prevents a faulty configuration from being replicated between the nodes again during the update.

## Sources

1. [SEPPmail Documentation – “Cluster / High Availability”](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): Cluster types and replication of the configuration across all nodes.
2. [SEPPmail Documentation – “Administration”](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): Update order in the cluster (frontend before backend) and the requirement for identical versions.
3. [HIN Mailgateway: Backup & Disaster Recovery in the Cluster](/blog/hin-mailgateway-backup-disaster-recovery): In-depth discussion of cluster replication, backup, and restore.
