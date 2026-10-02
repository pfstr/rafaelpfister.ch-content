---
title: "Proton Drive sous Linux : état des lieux en octobre 2026"
navTitle: "Proton Drive et Linux"
description: "Le client Linux officiel est annoncé, mais pas encore disponible. Depuis juin 2026, il existe la CLI officielle de Proton Drive pour les scripts et les serveurs ; Proton Drive ne peut toujours être monté qu’avec Rclone. Il manque un accès machine limité à certains dossiers ou tâches."
date: "2026-10-01"
kategorie: "Proton Drive"
timeToRead: "8 min de lecture"
themen:
  - proton-drive
  - rclone
related:
  - proton-drive-cli
  - paperless-dokumente-clouddienst-auslagern
  - rclone-mount-in-docker-container
slug: "proton-drive-sous-linux-etat-des-lieux-en-juillet-2026"
translationOf: "proton-drive-linux-status"
translationId: article-ca282447e0b9acff
translationModel: gpt-5.6-terra
translatedAt: 2026-10-02T10:01:46.530Z
translationReview: required
translationSourceHash: 73500e1be526e5e93bbd7bf789b1cc500976140cb4cb9542404e1e7a56b40ce0
url: https://rafaelpfister.ch/fr/blog/proton-drive-sous-linux-etat-des-lieux-en-juillet-2026
---

Pour Windows et macOS, Proton Drive propose ses propres clients de synchronisation depuis 2023. Sous Linux, il n’existe jusqu’à présent que l’interface web, des outils communautaires et, depuis juin 2026, une application officielle en ligne de commande, mais pas encore de client de synchronisation. Sur un serveur, la situation est encore plus difficile, car ni une synchronisation de bureau ni une connexion interactive ne conviennent vraiment.

Cet aperçu décrit la situation au 1er octobre 2026. Il s’appuie sur les feuilles de route publiées, le code source de la CLI Proton Drive et un test pratique du backend Rclone [comme espace de stockage documentaire pour Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern).

**Mise à jour du 1er octobre 2026 :** La première version du 26 juillet ne décrivait l’application en ligne de commande que comme un outil du dépôt SDK. Proton l’avait toutefois déjà publiée officiellement le 9 juin 2026 sous le nom de **Proton Drive CLI**, avec des builds prêts à l’emploi pour Windows, macOS et Linux. La section correspondante et le tableau de recommandations ont été révisés en conséquence ; les détails figurent dans l’article dédié à la [Proton Drive CLI](/blog/proton-drive-cli).

## Le client Linux est annoncé, mais sans date

En juin 2026, Proton a confirmé explicitement pour la première fois qu’un client Linux était en cours de développement. Il repose sur le nouveau SDK unifié et doit utiliser la même base technique que les applications pour Windows et macOS. Début octobre 2026, il n’existe toujours ni date ni bêta publique.

Il est important de le préciser : il s’agira d’un **client de synchronisation de bureau**. Pour le poste de travail, cela résout le problème. Pour les applications serveur, un client de synchronisation est en revanche le mauvais outil, car un service doit lire directement des fichiers depuis Proton Drive et y écrire. Un client de synchronisation conserve une copie locale complète, précisément ce qu’il faut éviter lorsque l’espace de stockage est limité.

## Rclone reste nécessaire pour les montages et les miroirs

Sous Linux, Rclone et son backend `protondrive` sont actuellement l’outil le plus polyvalent. Il peut copier et synchroniser des fichiers et, en tant que seule solution disponible, mettre Proton Drive à disposition comme un répertoire local via un **montage FUSE**. Deux limites sont importantes :

**Il est en bêta et repose sur une API reconstituée.** Proton ne documente pas publiquement son API Drive ; le backend repose sur de l’ingénierie inverse. Lors du test, il a fonctionné de manière fiable, mais limitait les séquences d’appels rapides avec des listes de répertoires incohérentes.

**Pour fonctionner sans surveillance, Rclone demande la clé TOTP.** L’assistant de configuration désigne ce champ comme `otp_secret_key`. Il s’agit de la clé permanente issue de la configuration de l’authentification à deux facteurs, et non du code à six chiffres actuellement affiché par une application d’authentification. Rclone enregistre cette valeur de manière obfusquée et génère lui-même un code TOTP valide à chaque connexion.

Toute personne qui saisit par erreur un code à usage unique actuel peut terminer la première connexion. La prochaine réauthentification échouera toutefois avec l’erreur 8002, car Rclone ne peut pas réutiliser le même code.

Le compte reste ainsi protégé contre le vol isolé d’un mot de passe. Un serveur compromis révèle toutefois le mot de passe et la clé TOTP. Pour les accès automatisés, il est donc recommandé d’utiliser un **compte Proton dédié**.

Le comportement d’un tel montage dans des environnements Docker, y compris deux problèmes non documentés, est décrit dans l’[article dédié à Rclone dans les conteneurs](/blog/rclone-mount-in-docker-container).

## La CLI officielle couvre les scripts et les sauvegardes

Le 9 juin 2026, Proton a publié la **Proton Drive CLI**, un unique fichier exécutable `proton-drive` pour Windows, macOS et Linux. Elle repose sur le même SDK que les applications officielles ; le code source se trouve dans le dépôt SDK public. La version actuelle est la 0.8.0 du 13 août 2026, avec des builds pour x86-64 (y compris sans AVX2), ARM64 et les distributions musl telles qu’Alpine.

Le modèle de connexion est plus propre que celui du backend Rclone :

- `auth login` affiche une URL de connexion qui peut aussi être ouverte **sur un autre appareil** ; la connexion se déroule normalement, **authentification à deux facteurs comprise**, donc également via SSH sur un serveur sans bureau
- la session est enregistrée dans le **trousseau du système d’exploitation** (Keychain, Credential Manager, libsecret) ou, depuis la version 0.6.0, dans le gestionnaire de mots de passe `pass`, plus pratique sur les serveurs sans session de bureau
- ensuite : téléverser et télécharger des fichiers, les déplacer, les mettre à la corbeille, gérer les partages, les invitations et les liens publics, utiliser Proton Photos ; à chaque fois avec `--json` pour une sortie lisible par machine

Le mot de passe et la clé TOTP ne doivent donc pas être stockés sur le serveur. Pour les sauvegardes et les artefacts de build, la CLI est donc aujourd’hui un meilleur choix que Rclone. Deux limites subsistent : la CLI ne peut **pas monter de système de fichiers** ni **créer un miroir avec suppressions** ; elle téléverse et télécharge, mais ne synchronise pas. Une commande `takeout` pour une exportation locale complète est déjà présente dans le dépôt, mais n’est pas encore publiée.

Le SDK lui-même est toujours considéré par Proton comme non prêt pour la production pour les applications tierces ; sa publication est prévue entre la fin 2026 et le début 2027. La CLI n’est pas concernée, puisque Proton la publie lui-même.

## La véritable lacune : les accès machine

Le cœur du problème se situe à un niveau plus profond que le client ou le SDK : **Proton ne propose pas d’accès machine.** Ni mot de passe d’application, ni compte de service, ni jeton à portée limitée. Toute automatisation, qu’il s’agisse d’un script de sauvegarde, d’un montage serveur ou d’une tâche CI, doit utiliser les identifiants complets du compte.

À titre de comparaison : avec les stockages compatibles S3, les paires de clés d’accès sont la norme, révocables et limitées à des buckets ou préfixes. Google et Microsoft proposent des mots de passe d’application et des comptes de service. Chez Proton, en revanche, c’est tout ou rien : donner à un serveur l’accès à un dossier revient à lui donner accès à l’ensemble du compte.

Avec un service chiffré de bout en bout, c’est plus difficile qu’avec S3, car un accès limité devrait aussi impliquer un matériel de clés limité. Les sessions de la CLI montrent toutefois que Proton maîtrise de telles constructions. Une session est déjà un accès dérivé et révocable, mais avec toute la portée du compte. Un « jeton machine officiel pour ce dossier précis, en lecture seule » constituerait la plus grande avancée individuelle pour l’utilisation sur serveur, bien avant n’importe quel client.

## Recommandation selon le cas d’utilisation

| Cas d’utilisation | État en octobre 2026 |
|---|---|
| Synchronisation de bureau sous Linux | Attendre le client annoncé ; d’ici là, synchronisation Rclone ou interface web |
| Sauvegarde serveur (téléverser des fichiers) | [Proton Drive CLI](/blog/proton-drive-cli) avec `filesystem upload` et la stratégie de conflit `create-new-revision`; officiellement prise en charge, sans mot de passe enregistré |
| Miroir avec suppressions | Rclone avec `sync`; tenir compte du statut bêta |
| Montage de système de fichiers pour des services | Rclone avec `mount`, une clé TOTP enregistrée et un compte dédié ; la seule [solution éprouvée en pratique](/blog/paperless-dokumente-clouddienst-auslagern) |
| Automatisation par script, gestion des partages | Proton Drive CLI avec `--json`; version 0.x, les commandes peuvent encore changer |

Sur le poste Linux, on peut attendre le client annoncé ou utiliser Rclone pour l’instant. Sur les serveurs, la CLI officielle prend désormais en charge les sauvegardes et l’automatisation ; pour un montage, Rclone reste la seule solution praticable. Mais un palliatif fonctionnel ne deviendra une plateforme solide que lorsque Proton proposera des accès machine limités et un montage officiellement pris en charge.

## Sources

1.  [OMG Ubuntu: Proton Drive client is (finally) coming to Linux](https://www.omgubuntu.co.uk/2026/06/proton-drive-linux-client): la confirmation de juin 2026 que le client Linux est en développement, sans date.

2.  [Proton: Product roadmaps for spring and summer 2026](https://proton.me/blog/2026-spring-summer-roadmaps): la feuille de route avec le client Linux sans calendrier et le SDK comme fondement des propres applications.

3.  [ProtonDriveApps/sdk sur GitHub](https://github.com/ProtonDriveApps/sdk): le dépôt SDK public avec le code source et le changelog de la CLI.

4.  [Proton: Introducing Proton Drive CLI](https://proton.me/blog/proton-drive-cli): la publication officielle de la CLI le 9 juin 2026.

5.  [Proton Drive CLI: Downloads](https://proton.me/download/drive/cli/index.html): version actuelle 0.8.0 du 13 août 2026 avec tous les builds de plateforme.

6.  [Proton Drive SDK preview](https://proton.me/blog/proton-drive-sdk-preview): l’évaluation de Proton lui-même : pas encore prêt pour la production pour les applications tierces.

7.  [Rclone: Proton Drive](https://rclone.org/protondrive/): le backend avec l’avertissement bêta et l’option `otp_secret_key` pour la connexion sans surveillance.
