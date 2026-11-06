---
slug: "hin-mailgateway-15-0-5-resoudre-la-panne-de-connexion-apres-la-mise-a-jour-du-cluster"
title: "HIN Mailgateway 15.0.5 : corriger la panne de connexion après la mise à jour du cluster"
navTitle: "Erreur de connexion 15.0.5"
description: "Après la mise à jour d’un cluster HIN Mailgateway vers la version 15.0.5, la connexion échoue après quelques minutes sur les deux nœuds. Cette procédure permet de remettre les appliances en service de manière contrôlée."
date: "2026-06-19"
kategorie: "Passerelle HIN"
timeToRead: "3 min de lecture"
themen:
  - hin-gateway
draft: false
translationOf: "hin-update-issue-version-15.0.5"
translationId: article-bd1908eec39f9c26
translationReview: automatic
translationSourceHash: 126aded044bfb102e020aaf45b7b77f6658f77a6de88a71d1554b1fabeefce2a
translatedAt: 2026-11-06T03:28:42.751Z
translationModel: gpt-5.6-terra
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/hin-mailgateway-15-0-5-resoudre-la-panne-de-connexion-apres-la-mise-a-jour-du-cluster
---

# HIN Mailgateway 15.0.5 : corriger la panne de connexion après la mise à jour du cluster

Lors de la mise à jour d’une HIN Mailgateway de la version 14.1.4.2 vers 15.0.5, une erreur dans la réplication du cluster peut entraîner l’échec de la connexion sur les deux appliances. Les systèmes isolés ne sont pas concernés. Le fabricant connaît le problème et prévoit un correctif dans une version ultérieure.

**Mise à jour du 29 juillet 2026 :** Le correctif annoncé est disponible. La version corrective 15.0.6 empêche le rehashage des mots de passe lorsque les membres du cluster utilisent des versions de firmware différentes. C’est exactement la configuration qui avait provoqué la panne décrite ici. Le contexte est présenté dans l’article sur [SEPPmail 15.0.6 et 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1); la procédure de restauration suivante reste pertinente pour les clusters qui effectuent encore la mise à jour vers 15.0.5.

## Symptômes

Immédiatement après la mise à jour, l’interface web peut encore être ouverte. Environ dix minutes plus tard, la connexion échoue sur les deux nœuds du cluster. Le fait que l’erreur survienne avec un décalage et sur les deux systèmes indique que la configuration répliquée du cluster en est la cause.

## Restauration

Les étapes suivantes modifient la configuration du cluster. Des sauvegardes récentes et l’identifiant du cluster doivent être disponibles au préalable.

1. Restaurer les snapshots créés simultanément des deux nœuds du cluster.
2. Après la restauration, laisser un nœud éteint.
3. Sur le nœud en cours d’exécution, télécharger d’abord l’identifiant du cluster, puis dissoudre le cluster.
4. Attention : après la dissolution, l’appliance redémarre immédiatement et sans autre confirmation.

![](../images/hin-update-issue-version-15.0.5/YSaXyzS9jLOD9utH0H2AEDOdnjI.png)

5. Mettre à jour le premier nœud vers la version 15.0.5, puis l’arrêter.
6. Démarrer le deuxième nœud et y répéter les mêmes étapes.
7. Ne reconstruire le cluster conformément à la documentation du fabricant que lorsque les deux systèmes fonctionnent individuellement et disposent de la même version.

Cette procédure empêche qu’une configuration défectueuse soit à nouveau répliquée entre les nœuds pendant la mise à jour.

## Historique des modifications

- **25 octobre 2026 :** Corrigé : selon la documentation SEPPmail, l’ordre de mise à jour dans le cluster est backend avant frontend, et non l’inverse.

## Sources

1. [Documentation SEPPmail – « Cluster / Haute disponibilité »](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): types de clusters et réplication de la configuration sur tous les nœuds.
2. [Documentation SEPPmail – « Administration »](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): ordre de mise à jour dans le cluster (backend avant frontend) et exigence de versions identiques.
3. [HIN Mailgateway : sauvegarde et reprise après sinistre dans le cluster](/blog/hin-mailgateway-backup-disaster-recovery): analyse approfondie de la réplication du cluster, de la sauvegarde et de la restauration.
