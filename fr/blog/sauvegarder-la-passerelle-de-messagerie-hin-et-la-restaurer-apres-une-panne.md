---
title: "Sauvegarder et restaurer la passerelle de messagerie HIN après une panne"
navTitle: "Sauvegarde et restauration"
description: "Un cluster protège la passerelle de messagerie HIN contre la défaillance d’un nœud, mais ne remplace pas une sauvegarde. La configuration, le matériel de clés, l’ordre de restauration et les changements liés à Stargate sont déterminants."
date: "2026-07-08"
kategorie: "Passerelle HIN"
timeToRead: "15 min de lecture"
themen:
  - hin-gateway
slug: "sauvegarder-la-passerelle-de-messagerie-hin-et-la-restaurer-apres-une-panne"
translationOf: "hin-mailgateway-backup-disaster-recovery"
translationId: article-845fb4bd0e4c592a
translationReview: required
translationSourceHash: cd8f7c8633d8cf882725ad88feed7be5a1730ce563724481e25b4b9ee2ec84e4
translatedAt: 2026-11-06T03:22:27.429Z
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/sauvegarder-la-passerelle-de-messagerie-hin-et-la-restaurer-apres-une-panne
translationModel: gpt-5.6-terra
---

# Sauvegarder et restaurer la passerelle de messagerie HIN après une panne

De nombreuses passerelles de messagerie HIN en production fonctionnent en cluster. Lorsqu’un nœud tombe en panne, l’autre prend le relais. Cette redondance ne protège toutefois pas contre une règle erronée, un certificat supprimé ou une importation corrompue : [Les données essentielles au système sont répliquées sur tous les nœuds](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html), y compris les modifications indésirables.

Une sauvegarde distincte est donc nécessaire pour une restauration fiable. La passerelle de messagerie HIN reposant techniquement sur une appliance SEPPmail avec GINA, les mécanismes documentés de sauvegarde et de restauration de celle-ci s’appliquent.

## Quelles données se trouvent sur la passerelle

La passerelle traite les e-mails entrants et sortants selon un ensemble central de règles et les chiffre, selon le destinataire, via S/MIME, OpenPGP ou TLS ; pour les destinataires ne disposant pas de leur propre matériel de clés, le procédé GINA basé sur le Web est utilisé. Pour la sauvegarde, il est essentiel de savoir que [le contenu des messages n’est pas stocké de manière persistante sur la passerelle](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html) : l’appliance traite les e-mails en transit, sans les archiver.

  

## Ce que le cluster réplique

SEPPmail connaît plusieurs [configurations de cluster – haute disponibilité, équilibrage de charge et géo-cluster](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html) ; les paramètres système, les données utilisateur et le matériel de clés sont synchronisés sur tous les nœuds. Dans un [cluster frontend/backend, le frontend ne dispose pas de sa propre base de données de configuration](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html) : il peut être exploité dans une DMZ sans stockage de données et ne reçoit que les données nécessaires au traitement en cours ; la base de données avec les clés se trouve sur le backend. Une exception s’applique au [Large File Transfer (LFT)](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html) : un disque de même taille est attribué à chaque partenaire, y compris aux frontends, et les données LFT sont synchronisées sur tous les nœuds.

  

## Pourquoi la réplication n’est pas une sauvegarde

> *La réplication copie l’état actuel, y compris les erreurs. Une sauvegarde préserve un état connu et fonctionnel.*

Une importation erronée, une clé supprimée ou un domaine désactivé est répliqué en quelques secondes sur le nœud partenaire. Sans sauvegarde indépendante, il n’existe alors plus de point de restauration. Le lien étroit entre disponibilité et cohérence dans le cluster s’est manifesté lors des [problèmes de connexion après la mise à jour vers la version 15.0.5](/blog/hin-update-issue-version-15.0.5), causés par une réplication de cluster perturbée.

  

## Ce qui est inclus dans la sauvegarde et ce qui ne l’est pas

La [sauvegarde SEPPmail est volontairement légère](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html) : elle comprend exclusivement la configuration et le matériel de clés cryptographiques : [aucun message, aucune file d’attente de messagerie et expressément aucun journal](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) (les journaux doivent donc être envoyés via Syslog vers un système externe). Depuis le firmware 14.0.0, l’appliance crée la sauvegarde [automatiquement la nuit à minuit sous le nom backup.tgz](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) ; elle peut être récupérée via `Download`, `Send Backup` (e-mail au groupe de sauvegarde) ou SCP.

| **Inclus dans la sauvegarde** | **Non inclus dans la sauvegarde** |
| --- | --- |
| Configuration système et ensemble de règles | Contenu des e-mails / texte des messages |
| Comptes utilisateur et GINA | File d’attente de messagerie actuelle |
| Matériel de clés : S/MIME, X.509, OpenPGP | Journaux système et de messagerie (à sauvegarder en externe via Syslog) |
| Configuration TLS et des certificats | Système d’exploitation / image VM |


Il en résulte que, le système d’exploitation n’étant pas inclus dans la sauvegarde de configuration, une stratégie de reprise après sinistre complète nécessite également un moyen de restaurer la base de l’appliance (redéploiement à partir de l’image du fabricant ou instantané VM). La sauvegarde de configuration complète ensuite la configuration et les clés.

  

## Les instantanés ne sont pas une sauvegarde de cluster

Depuis le firmware 14.0.0, l’appliance crée en plus [des instantanés locaux, mais uniquement si une partition LFT avec base de données est présente](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). Un instantané complet est créé le dimanche, puis un instantané incrémentiel chaque jour du lundi au samedi ; la durée de conservation est de 14 jours.

Point décisif pour la planification de la reprise après sinistre : en mode cluster, ces instantanés s’exécutent certes en arrière-plan, mais aucune restauration n’est proposée à partir de ceux-ci. Les instantanés constituent donc une aide au retour en arrière local sur des systèmes isolés, et non une récupération de cluster. La sauvegarde fiable reste la sauvegarde de configuration chiffrée.

  

## Configurer la sauvegarde

La condition préalable à chaque méthode de récupération est la définition d’un mot de passe de sauvegarde sous [Administration › Backup › Change password](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) ; sans ce mot de passe, aucune sauvegarde ne peut être téléchargée, envoyée ou mise à disposition via SCP. Par défaut, la sauvegarde nocturne est envoyée par e-mail au [groupe « backup (Backup Operator) »](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html) ; un utilisateur de sauvegarde dédié nécessite une adresse e-mail interne valide.

-   Définir le mot de passe de sauvegarde et le [conserver séparément de la sauvegarde](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html) : la sauvegarde contient des clés privées.
    
-   Pour le stockage automatisé, [récupérer les sauvegardes via SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html) : enregistrer la clé publique `SSH-RSA`\- dans l’administration et récupérer à minuit `backup.tgz` via l’utilisateur OS `backup`.
    
-   Sauvegarder les journaux séparément (Syslog externe), car ils [ne font volontairement pas partie de la sauvegarde](https://docs.seppmail.com/de/07_mi_11_adm__administration.html).
    

  

## Stratégie de sauvegarde en mode cluster

En mode cluster, une sauvegarde ordonnée et une gestion cohérente des versions sont essentielles.

-   **Quotidiennement** : récupérer la sauvegarde de configuration chiffrée [via SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html) et l’archiver en externe avec versionnage
    
-   **Hebdomadairement** : sauvegarde complète de la VM ou du système des deux nœuds, décalée dans le temps plutôt que simultanée (le système d’exploitation ne fait pas partie de la sauvegarde de configuration)
    
-   **Avant une maintenance ou une mise à jour** : arrêter l’acceptation des e-mails via [Preempt](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) : les e-mails entrants sont alors temporairement refusés avec un code de retour SMTP configurable (par défaut `421`) ; ce paramètre reste actif même après un redémarrage.
    

  

Concernant la gestion des versions : dans un cluster frontend/backend, SEPPmail met à jour [le frontend avant le backend](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), et lors de mises à jour en plusieurs étapes, tous les partenaires doivent avoir le même niveau de version avant de passer à la version suivante. Après une mise à jour majeure, il peut être nécessaire de régénérer l’ensemble de règles (message *« Current ruleset created for another version »*).

  

## Restauration et reprise après sinistre

Le cas de base est simple : [Import backup file](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), redémarrer, puis la passerelle fonctionne à nouveau avec toutes ses fonctionnalités. La règle de version doit être respectée : seule la sauvegarde de la version de firmware immédiatement précédente peut être importée dans la version actuelle (puis régénérer l’ensemble de règles) ; il n’est pas possible d’importer la sauvegarde d’un firmware plus récent dans une version plus ancienne.

Une restriction importante s’applique dans le cluster :

-   **Ne jamais restaurer directement un nœud isolé** : une [restauration d’un seul partenaire de cluster n’est pas prévue](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). Il faut plutôt retirer la machine défectueuse du cluster, déployer une nouvelle VM et l’ajouter à nouveau : la configuration et les clés proviennent automatiquement du partenaire intact par réplication.
    
-   **Perte totale de tous les nœuds** : redéployer l’appliance à partir de l’image de base, puis importer la dernière sauvegarde de configuration connue comme fonctionnelle et redémarrer.
    

Une sauvegarde n’est fiable qu’à la mesure du dernier test de restauration réussi. Un test de restauration devrait être effectué au moins deux fois par an dans un environnement isolé, et non sur le cluster de production.

  

### Liste de contrôle de restauration en cas d’urgence

1.  Retirer le nœud défectueux du cluster (pas de restauration directe d’un partenaire).
    
2.  Déployer une nouvelle VM ou, en cas de perte totale, fournir l’appliance à partir de l’image de base/de l’instantané VM.
    
3.  Uniquement en cas de perte totale : importer la dernière sauvegarde de configuration fonctionnelle (préparer le mot de passe, respecter la règle de version).
    
4.  Vérifier le nœud de manière isolée : acceptation SMTP, TLS, GINA, ensemble de règles.
    
5.  L’intégrer au cluster et surveiller la réplication ; régénérer l’ensemble de règles si le message apparaît.
    
6.  Documenter l’incident et ajuster l’intervalle de sauvegarde ainsi que les niveaux de version.
    

  

Deux opérations de maintenance exigent une prudence particulière et toujours une sauvegarde préalable : [l’agrandissement de la partition LFT arrête l’appliance](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), et la réinitialisation d’usine écrase le disque dur dix fois (la confirmation de sécurité exige le code écrit à l’envers).

  

## Ce qui change avec « Stargate »

HIN remplace progressivement la passerelle de messagerie actuelle par la [nouvelle passerelle HIN](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm) (projet « Stargate », désigné chez la société zougoise [Vereign AG sous le nom « Verimesh »](https://www.vereign.com/)). Il ne s’agit pas d’un remplacement à l’identique de l’appliance, mais d’un changement d’architecture qui touche au cœur même de la sauvegarde et de la reprise après sinistre :

-   **Du centralisé au décentralisé** : les nœuds communiquent directement entre eux ; un centre de distribution central disparaît.
    
-   **Gestion décentralisée des clés (DKMS)** : chaque organisation gère sa propre identité cryptographique, sans autorité de certification centrale.
    
-   **Chiffrement de bout en bout** avec fragmentation des messages.
    
-   **Résilience par le réseau** : si un nœud tombe en panne, le maillage reste opérationnel.
    
-   **Dépôt de déploiement ouvert** : le dépôt [Stargate Deployment](https://code.vereign.com/svdh/stargate-deployment) contenant les services d’un nœud Stargate est publiquement consultable sous AGPLv3 ; la Vereign Client Library (vcl), en revanche, n’est plus accessible qu’après inscription.
    

Calendrier : l’infrastructure décentralisée est [en production dans le secteur suisse de la santé depuis avril 2025](https://www.vereign.com/) ; pour 2026, le remplacement progressif des passerelles de messagerie existantes et un déploiement à grande échelle sont prévus. Les organisations disposant de domaines propres à HIN (`@hin.ch`, `@verband-hin.ch`) fonctionnent sur l’infrastructure HIN et sont peu concernées par cette transition.

  

Pour le manuel d’exploitation, cela signifie que la discipline classique consistant à « exporter la configuration et les clés de l’appliance, puis à les restaurer sur un nœud de remplacement » perd de son importance. Elle est remplacée par l’enrôlement des nœuds, la conservation des identités et des clés dans le maillage ainsi que la réintégration des nœuds dans le réseau.

  

## La distinction essentielle

Tant que la passerelle de messagerie HIN repose sur la technologie SEPPmail, la règle est la suivante : le cluster compense les défaillances matérielles, mais la responsabilité de l’intégrité de la configuration et des clés reste à la charge de l’exploitant. La sauvegarde légère de configuration doit être protégée indépendamment du cluster (via SCP, avec versionnage et un mot de passe conservé séparément), les instantanés ne la remplacent pas dans le cluster, les niveaux de version restent synchronisés et la restauration est régulièrement testée de manière isolée. La migration vers Stargate devrait être intégrée suffisamment tôt à la planification de reprise après sinistre, car elle déplace la résilience et la conservation des clés vers le réseau décentralisé.

## Historique des modifications

- **25 octobre 2026 :** Ajout : la Vereign Client Library n’est plus accessible qu’après inscription ; le dépôt Stargate Deployment est désormais indiqué comme source publiquement consultable sous AGPLv3.

## Sources

1.  [Documentation SEPPmail – « Backup / Restore »](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html) : contenu de la sauvegarde (uniquement configuration et matériel de clés), création nocturne, restauration automatique du cluster par réplication.
    
2.  [Documentation SEPPmail – « Administration »](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) : référence détaillée : menu de sauvegarde (Download / Send Backup / Change password, `backup.tgz` à minuit), instantanés LFT (14 jours, pas de restauration dans le cluster), règles de restauration et procédure de cluster, Preempt (code de retour SMTP, par défaut 421), clonage d’appareil, canaux de mise à jour et ordre des mises à jour (frontend avant backend), réinitialisation d’usine, importation/exportation en masse.
    
3.  [Documentation SEPPmail – « Créer un utilisateur de sauvegarde »](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html) : groupe « backup (Backup operator) », chiffrement et gestion des mots de passe.
    
4.  [Documentation SEPPmail – « Copier une sauvegarde via SCP »](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html) : récupération de `backup.tgz` via SCP par l’utilisateur OS `backup` au lieu de l’envoi par e-mail.
    
5.  [Documentation SEPPmail – « Cluster / haute disponibilité »](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html) : types de cluster et données synchronisées sur tous les nœuds (paramètres système, données utilisateur, matériel de clés).
    
6.  [Documentation SEPPmail – « Cluster frontend/backend »](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html) : frontend sans base de données de configuration, fonctionnement en DMZ, données à la demande ; backend comme détenteur des données.
    
7.  [Documentation SEPPmail – « Stockage de données dans le cluster (LFT) »](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html) : disque supplémentaire de même taille pour chaque partenaire, synchronisation des données LFT sur tous les nœuds.
    
8.  [HIN AG – « De la passerelle de messagerie au Data Mesh »](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm) : communication HIN sur le successeur Stargate : nœuds décentralisés, concept de Data Mesh, calendrier, chiffrement de bout en bout.
    
9.  [Vereign AG – « Verimesh »](https://vereign.com/) : Verimesh (anciennement projet Stargate) en production dans le secteur suisse de la santé depuis avril 2025.
    
10. [Vereign GitLab – Stargate Deployment](https://code.vereign.com/svdh/stargate-deployment) : dépôt de déploiement public sous AGPLv3 avec les services d’un nœud Stargate et renvoi vers la [documentation de déploiement](https://health-info-net-ag.github.io/Stargate-deployment/).
