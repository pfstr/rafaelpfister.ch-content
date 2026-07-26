---
title: "HIN Mailgateway 15.0.5: Fixing the Login Outage After a Cluster Update"
navTitle: "Login issue 15.0.5"
description: "After updating a HIN Mailgateway cluster to version 15.0.5, login fails on both nodes within a few minutes. This procedure brings the appliances back into controlled operation."
date: "2026-06-19"
kategorie: "HIN-Gateway"
timeToRead: "3 min to read"
themen:
  - "hin-gateway"
slug: "hin-mailgateway-update-15-0-5-login-issue"
translationOf: "hin-update-issue-version-15.0.5"
url: "https://rafaelpfister.ch/en/blog/hin-mailgateway-update-15-0-5-login-issue"
---

# HIN Mailgateway 15.0.5: Fixing the Login Outage After a Cluster Update

When updating a HIN Mailgateway from 14.1.4.2 to 15.0.5, a bug in cluster replication can knock out the login on both appliances. Standalone systems are not affected. The vendor is aware of the issue and plans a fix in a later version.

## Fault Symptoms

Immediately after the update, the web interface can still be opened. About ten minutes later, login fails on both cluster nodes. The fact that the error appears with a delay and on both systems points to the replicated cluster configuration as the cause.

## Recovery

These steps modify the cluster configuration. Before you begin, make sure current backups and the cluster identifier are available.

1. Restore the snapshots taken simultaneously on both cluster nodes.
2. After the restore, leave one node powered off.
3. On the running node, download the cluster identifier first, then dissolve the cluster.
4. Warning: after dissolving the cluster, the appliance reboots immediately without any further confirmation prompt.

![](../images/hin-mailgateway-update-15-0-5-login-issue/YSaXyzS9jLOD9utH0H2AEDOdnjI.png)

5. Update the first node to version 15.0.5 and then shut it down.
6. Start the second node and repeat the same steps there.
7. Only once both systems work individually and are on the same version, rebuild the cluster according to the vendor documentation.

This procedure prevents a faulty configuration from being replicated between the nodes again during the update.

## Sources

1. [SEPPmail Documentation – “Cluster / High Availability”](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): Cluster types and configuration replication across all nodes.
2. [SEPPmail Documentation – “Administration”](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): Update order within the cluster (frontend before backend) and the requirement for identical version numbers.
3. [HIN Mailgateway: Backup & Disaster Recovery im Cluster](/en/blog/hin-mail-gateway-backup-disaster-recovery): An in-depth look at cluster replication, backup, and restore.
