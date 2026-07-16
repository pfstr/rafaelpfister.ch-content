---
title: "Login is no longer possible after updating the HIN Mailgateway from version 14.1.4.2 to 15.0.5 (affects only systems in a cluster)"
navTitle: "Login issue 15.0.5"
description: "HIN MGW Login Issues After Update to 15.0.5 Due to Faulty Cluster Replication"
date: "2026-06-19"
kategorie: "HIN-Gateway"
timeToRead: "3 min to read"
themen:
  - "hin-gateway"
slug: "hin-mailgateway-update-15-0-5-login-issue"
translationOf: "hin-update-issue-version-15.0.5"
url: "https://rafaelpfister.ch/en/blog/hin-mailgateway-update-15-0-5-login-issue"
---

# Login is no longer possible after updating the HIN Mailgateway from version 14.1.4.2 to 15.0.5 (affects only systems in a cluster)

## Fault Symptoms

After updating to version 15.0.5, it is still possible to log in to the appliance for a few minutes. After about 10 minutes, login attempts fail on both cluster members. The error pattern suggests that cluster replication is causing the problems. According to the manufacturer, this is a known issue.

## Solution

1.  Restore snapshots from both cluster members simultaneously
    
2.  A cluster member must remain shut down after the restore
    
3.  Dissolve the cluster (download the cluster identifier first)
    
4.  The system boots up immediately and without warning!
    
    ![](../images/YSaXyzS9jLOD9utH0H2AEDOdnjI.png)
    
5.  Update System 1
    
6.  Shut down System 1
    
7.  Repeat the same game using System 2

## Sources

1.  [SEPPmail Documentation – “Cluster / High Availability”](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html) — Cluster types and configuration replication across all nodes.
    
2.  [SEPPmail Documentation – “Administration”](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) — Update order within the cluster (frontend before backend) and the requirement for identical version numbers.
    
3.  [HIN Mailgateway: Backup & Disaster Recovery im Cluster](/en/blog/hin-mail-gateway-backup-disaster-recovery) — An in-depth look at cluster replication, backup, and restore.
