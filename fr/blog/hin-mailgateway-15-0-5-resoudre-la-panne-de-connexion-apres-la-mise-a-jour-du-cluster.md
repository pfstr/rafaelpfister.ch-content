---
slug: "hin-mailgateway-15-0-5-resoudre-la-panne-de-connexion-apres-la-mise-a-jour-du-cluster"
title: "HIN Mailgateway 15.0.5 : résoudre la panne de connexion après la mise à jour du cluster"
navTitle: "Erreur de connexion 15.0.5"
description: "Après la mise à jour d’un cluster HIN Mailgateway vers la version 15.0.5, la connexion échoue sur les deux nœuds après quelques minutes. Cette procédure permet de remettre les appliances en service de manière contrôlée."
date: "2026-06-19"
kategorie: "Passerelle HIN"
timeToRead: "3 min de lecture"
themen:
  - hin-gateway
draft: false
translationOf: "hin-update-issue-version-15.0.5"
url: "https://rafaelpfister.ch/fr/blog/hin-mailgateway-15-0-5-resoudre-la-panne-de-connexion-apres-la-mise-a-jour-du-cluster"
---

# HIN Mailgateway 15.0.5 : résoudre la panne de connexion après la mise à jour du cluster

Lors de la mise à jour d’un HIN Mailgateway de la version 14.1.4.2 vers 15.0.5, une erreur dans la réplication du cluster peut paralyser la connexion sur les deux appliances. Les systèmes isolés ne sont pas concernés. Le fabricant connaît le problème et prévoit un correctif dans une version ultérieure.

## Symptômes

Immédiatement après la mise à jour, l’interface web peut encore être ouverte. Environ dix minutes plus tard, la connexion échoue sur les deux nœuds du cluster. Le fait que l’erreur survienne avec un décalage et sur les deux systèmes indique que la configuration de cluster répliquée en est la cause.

## Restauration

Les étapes suivantes modifient la configuration du cluster. Des sauvegardes récentes et l’identifiant du cluster doivent être disponibles au préalable.

1. Restaurer les snapshots créés simultanément des deux nœuds du cluster.
2. Après la restauration, laisser un nœud éteint.
3. Sur le nœud en cours d’exécution, télécharger d’abord l’identifiant du cluster, puis dissoudre le cluster.
4. Attention : après sa dissolution, l’appliance redémarre immédiatement, sans autre confirmation.

![](../images/hin-update-issue-version-15.0.5/YSaXyzS9jLOD9utH0H2AEDOdnjI.png)

5. Mettre à jour le premier nœud vers la version 15.0.5, puis l’arrêter.
6. Démarrer le second nœud et y répéter les mêmes étapes.
7. Ne reconstruire le cluster conformément à la documentation du fabricant que lorsque les deux systèmes fonctionnent individuellement et disposent de la même version.

Cette procédure empêche qu’une configuration défectueuse soit à nouveau répliquée entre les nœuds pendant la mise à jour.

## Sources

1. [Documentation SEPPmail – « Cluster / haute disponibilité »](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): types de cluster et réplication de la configuration sur tous les nœuds.
2. [Documentation SEPPmail – « Administration »](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): ordre de mise à jour dans le cluster (frontend avant backend) et exigence de versions identiques.
3. [HIN Mailgateway : sauvegarde et reprise après sinistre dans le cluster](/blog/hin-mailgateway-backup-disaster-recovery): analyse approfondie de la réplication du cluster, de la sauvegarde et de la restauration.
