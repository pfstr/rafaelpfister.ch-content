---
title: "Proton Drive sous Linux : état des lieux en juillet 2026"
navTitle: "Proton Drive & Linux"
description: "Le client Linux officiel est annoncé, mais pas encore disponible. Sur les serveurs, Proton Drive peut actuellement être monté avec Rclone ; le nouveau SDK indique la direction technique. Il manque toujours un accès machine limité à certains dossiers ou tâches."
date: "2026-07-26"
kategorie: "Proton Drive"
timeToRead: "8 min de lecture"
themen:
  - proton-drive
  - rclone
related:
  - paperless-dokumente-clouddienst-auslagern
  - rclone-mount-in-docker-container
slug: "proton-drive-sous-linux-etat-des-lieux-en-juillet-2026"
translationOf: "proton-drive-linux-status"
url: "https://rafaelpfister.ch/fr/blog/proton-drive-sous-linux-etat-des-lieux-en-juillet-2026"
translationId: article-ca282447e0b9acff
translationModel: gpt-5.6-terra
translatedAt: 2026-07-28T21:47:16.269Z
translationReview: automatic
translationSourceHash: 1b0af572e102121912376d523c1785ed1563e4ca5c17eee8d605c5000096b57e
---

Pour Windows et macOS, Proton Drive propose ses propres clients de synchronisation depuis 2023. Sous Linux, il n’existe jusqu’à présent que l’interface web, des outils communautaires et un SDK officiel en préversion. Sur un serveur, la situation est encore plus difficile, car ni la synchronisation de bureau ni une connexion interactive ne conviennent vraiment.

Cet aperçu décrit la situation en juillet 2026. Outre les feuilles de route publiées, il s’appuie sur un test pratique du backend Rclone [comme dépôt de documents pour Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern).

## Le client Linux est annoncé, mais sans date

En juin 2026, Proton a confirmé pour la première fois explicitement qu’un client Linux était en cours de développement. Il repose sur le nouveau SDK unifié et devrait utiliser la même base technique que les applications pour Windows et macOS. Aucune date ni bêta publique n’ont encore été annoncées.

Il est important de le préciser : il s’agira d’un **client de synchronisation de bureau**. Pour le poste de travail, il résout le problème. Pour les applications serveur, un client de synchronisation est en revanche le mauvais outil, car un service doit lire directement les fichiers depuis Proton Drive et y écrire. Un client de synchronisation conserve une copie locale complète, précisément ce que l’on veut éviter lorsque l’espace de stockage est limité.

## Aujourd’hui, Rclone fait le travail pratique

Sous Linux, Rclone et son backend `protondrive` constituent actuellement l’outil le plus polyvalent. Il peut copier et synchroniser des fichiers et, en tant que seule solution disponible, proposer Proton Drive comme un répertoire local via un **montage FUSE**. Deux limites sont importantes :

**Il est en bêta sur une API reconstruite.** Proton ne documente pas publiquement son API Drive ; le backend repose sur de l’ingénierie inverse. Lors du test, il a fonctionné de manière fiable, mais a limité les séquences d’appels rapides avec des listes de répertoires incohérentes.

**Pour un fonctionnement non supervisé, Rclone demande la clé TOTP.** L’assistant de configuration désigne ce champ comme `otp_secret_key`. Il s’agit de la clé permanente issue de la configuration 2FA, et non du code à six chiffres actuellement affiché par une application d’authentification. Rclone enregistre cette valeur sous une forme obscurcie et génère lui-même un code TOTP valide à chaque connexion.

Si vous saisissez par erreur un code à usage unique actuel, vous pouvez terminer la première connexion. La prochaine réauthentification échouera toutefois avec l’erreur 8002, car Rclone ne peut pas réutiliser le même code.

Le compte reste ainsi protégé contre un mot de passe volé isolément. Un serveur compromis révèle toutefois le mot de passe et la clé TOTP. Pour les accès automatisés, il est donc recommandé d’utiliser un **compte Proton dédié**.

Le comportement d’un tel montage dans des environnements Docker, y compris deux pièges non documentés, est décrit dans [l’article dédié à Rclone dans les conteneurs](/blog/rclone-mount-in-docker-container).

## Le SDK officiel montre la direction prise par le développement

Parallèlement, Proton migre ses applications vers un **SDK officiel** pour JavaScript et C#, avec des bindings pour Swift et Kotlin. Le dépôt public contient également un outil en ligne de commande. Son modèle de connexion est plus propre que celui du backend Rclone :

- `auth login` ouvre le navigateur ; la connexion se déroule normalement, **authentification à deux facteurs comprise**
- la session est enregistrée dans le **magasin de clés du système d’exploitation** (Keychain, Credential Manager, libsecret), et le SDK la renouvelle lui-même
- ensuite : lister des fichiers avec une sortie JSON lisible par machine, téléverser des fichiers et vérifier les partages

Le mot de passe et la clé TOTP n’ont donc pas besoin de figurer dans un fichier de configuration. Trois limites subsistent néanmoins pour l’exploitation sur serveur : la CLI ne peut **pas monter de système de fichiers**, la connexion ouvre un navigateur et Proton ne considère pas encore le SDK comme prêt pour la production dans des applications tierces. La sortie est prévue entre fin 2026 et début 2027.

## La véritable lacune : les accès machine

Le cœur du problème se situe à un niveau plus profond que le client ou le SDK : **Proton ne propose pas d’accès machine.** Ni mot de passe d’application, ni compte de service, ni jeton à portée limitée. Toute automatisation, qu’il s’agisse d’un script de sauvegarde, d’un montage serveur ou d’une tâche CI, doit fonctionner avec les identifiants complets du compte.

À titre de comparaison : avec les stockages compatibles S3, les paires de clés d’accès sont la norme, révocables et limitées à des buckets ou préfixes. Google et Microsoft proposent des mots de passe d’application et des comptes de service. Chez Proton, c’est en revanche tout ou rien : donner à un serveur l’accès à un dossier revient à lui donner accès à l’ensemble du compte.

Il faut reconnaître que c’est plus difficile pour un service chiffré de bout en bout que pour S3, car un accès limité devrait aussi impliquer un matériel de clés limité. Les sessions du SDK montrent toutefois que Proton maîtrise de telles constructions. Une session est déjà un accès dérivé et révocable. Un « jeton machine officiel pour ce dossier précis, en lecture seule » constituerait la plus grande avancée individuelle pour l’utilisation sur serveur, bien avant n’importe quel client.

## Recommandation selon le cas d’utilisation

| Cas d’utilisation | Situation en juillet 2026 |
|---|---|
| Synchronisation de bureau sous Linux | Attendre le client annoncé ; d’ici là, synchronisation Rclone ou interface web |
| Sauvegarde serveur (téléversement de fichiers) | Rclone avec `copy` ou `sync` ; fonctionne, tenir compte du statut bêta |
| Montage de système de fichiers pour des services | Rclone avec `mount`, clé TOTP enregistrée et compte dédié ; la seule [solution éprouvée en pratique](/blog/paperless-dokumente-clouddienst-auslagern) |
| Automatisation par script avec connexion propre | Garder un œil sur la CLI du SDK ; encore trop tôt pour la production |

Sur le poste Linux, on peut attendre le client annoncé ou utiliser Rclone en attendant. Sur les serveurs, Rclone reste la seule solution de montage praticable. Toutefois, cette solution de secours fonctionnelle ne deviendra une plateforme robuste que lorsque Proton proposera des accès machine limités et un montage officiellement pris en charge.

## Sources

1.  [OMG Ubuntu: le client Proton Drive arrive (enfin) sous Linux](https://www.omgubuntu.co.uk/2026/06/proton-drive-linux-client): la confirmation de juin 2026 que le client Linux est en développement, sans date.

2.  [Proton: feuilles de route produit pour le printemps et l’été 2026](https://proton.me/blog/2026-spring-summer-roadmaps): la feuille de route avec le client Linux sans calendrier et le SDK comme fondement des applications propres à Proton.

3.  [ProtonDriveApps/sdk sur GitHub](https://github.com/ProtonDriveApps/sdk): le dépôt SDK public, y compris la CLI avec connexion par navigateur et session dans le magasin de clés.

4.  [Aperçu du SDK Proton Drive](https://proton.me/blog/proton-drive-sdk-preview): l’évaluation de Proton lui-même : pas encore prêt pour la production dans des applications tierces.

5.  [Rclone : Proton Drive](https://rclone.org/protondrive/): le backend, avec l’indication bêta et l’option `otp_secret_key` pour la connexion non supervisée.
