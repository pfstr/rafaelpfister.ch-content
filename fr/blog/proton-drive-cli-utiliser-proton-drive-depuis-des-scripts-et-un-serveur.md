---
title: "Proton Drive CLI : utiliser Proton Drive depuis des scripts et un serveur"
navTitle: "Proton Drive CLI"
description: "Depuis juin 2026, Proton propose un outil officiel en ligne de commande pour Proton Drive. Cet article décrit les commandes, la connexion sur les serveurs sans bureau, les stratégies de conflit pour les scripts et les limites par rapport à Rclone."
date: "2026-10-01"
kategorie: "Proton Drive"
timeToRead: "9 min de lecture"
themen:
  - proton-drive
produkte:
  - "proton-drive"
protokolle:
  - "storage"
  - "backup-dr"
related:
  - proton-drive-linux-status
  - rclone-mount-in-docker-container
slug: "proton-drive-cli-utiliser-proton-drive-depuis-des-scripts-et-un-serveur"
translationId: "article-97376b998fdaec4c"
translationOf: proton-drive-cli
translationSourceHash: ce73cccdecdcdea611f1ead28872dab00bc33a7106e2eaefd51bee4241d8631c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-27T09:15:02.414Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/proton-drive-cli-utiliser-proton-drive-depuis-des-scripts-et-un-serveur
---

Le 9 juin 2026, Proton a publié la **Proton Drive CLI**, un outil officiel en ligne de commande pour Windows, macOS et Linux. Elle repose sur le même SDK que les applications Drive officielles, chiffre de bout en bout et est disponible sous la forme d’un unique fichier exécutable `proton-drive`. Le code source se trouve dans le dépôt public du SDK, sous `cli/`.

La CLI est conçue pour des opérations ponctuelles : téléverser des fichiers après un build, sauvegarder un dossier selon un calendrier, vérifier ou révoquer des partages. Elle ne synchronise pas en arrière-plan et ne monte pas de système de fichiers. La version actuelle est la **0.9.0 du 5 octobre 2026** ; son numéro de version montre que les commandes et options peuvent encore évoluer (la version 0.8.0 a renommé les stratégies de conflit avec un changement incompatible).

La présentation des autres options sous Linux (Rclone, client de bureau annoncé) figure dans l’article d’état [Proton Drive sous Linux](/blog/proton-drive-linux-status).

## Vue d’ensemble des commandes

Les commandes sont organisées en groupes : `proton-drive <gruppe> <befehl> [optionen] [argumente]`. Les noms des groupes peuvent être abrégés tant qu’ils restent univoques ; pour `filesystem`, l’alias `fs` est également disponible. Sans argument, une interface interactive est lancée. L’aide complète est fournie par `proton-drive help` ou `proton-drive <gruppe> <befehl> --help`.

<details class="options-details">
<summary>Aperçu des options</summary>

| Groupe / commande | Effet |
|---|---|
| `auth login` / `auth logout` | Connexion via le navigateur ; la déconnexion supprime les identifiants locaux et les caches |
| `filesystem list <pfad>` | Lister le contenu d’un dossier ; `/` affiche les zones racines |
| `filesystem info` / `size` | Métadonnées d’un élément ou taille d’un dossier, y compris le contenu de la corbeille |
| `filesystem upload` / `download` | Téléverser ou télécharger des fichiers et dossiers |
| `filesystem create-folder`, `rename`, `copy`, `move` | Créer, renommer, copier, déplacer des dossiers |
| `filesystem trash` / `restore` | Mettre dans la corbeille ou restaurer |
| `filesystem delete` / `empty-trash` | Supprimer définitivement ou vider `/trash` |
| `sharing status <pfad>` | Afficher les membres, les invitations ouvertes et les paramètres des liens |
| `sharing invite` / `remove` | Inviter des personnes par e-mail ou révoquer leur accès |
| `sharing set-url` / `remove-url` | Créer, modifier ou supprimer un lien public |
| `sharing leave` / `report` | Quitter un partage qui vous a été attribué ou le signaler comme abusif |
| `invitation list` / `accept` / `reject` | Gérer les invitations reçues |
| `album …`, `photo timeline`, `photo upload`, `photo download` | Proton Photos : albums et chronologie |
| `takeout run` | Exporter une copie hors ligne du compte dans un dossier local (depuis la version 0.9.0) |
| `version` | Afficher les versions de la CLI et du SDK |
| `--json` (`-j`) | Sortie JSON lisible par machine, pour chaque commande |
| `--verbose` (`-v`) | Afficher les journaux directement dans la console |
| `--help` (`-h`) | Aide relative à la commande concernée |

</details>

Les chemins dans Proton Drive sont toujours des chemins POSIX, y compris sous Windows. La racine `/` contient des zones virtuelles : `/my-files` (fichiers personnels), `/devices` (ordinateurs sauvegardés), `/shared-by-me`, `/shared-with-me`, `/trash`, ainsi que les zones Photos `/photos`, `/albums`, `/photos-shared-by-me`, `/photos-shared-with-me` et `/photos-trash`.

## Installation sous Linux

Proton fournit les builds sur une page de téléchargement dédiée, chacun avec une somme de contrôle SHA-512. Il existe cinq variantes pour Linux :

| Build | Utilisation |
|---|---|
| `linux/x64` | Standard pour les systèmes x86-64 récents |
| `linux/x64-baseline` | x86-64 sans AVX2, par exemple les NAS et processeurs de serveur plus anciens |
| `linux/arm64` | Serveurs ARM et ordinateurs monocarte avec glibc |
| `linux/x64-musl`, `linux/arm64-musl` | Distributions utilisant musl plutôt que glibc, par exemple Alpine Linux et les images de conteneurs qui en dérivent |

Si le build standard s’interrompt au démarrage avec `Illegal instruction`, le processeur ne dispose pas de l’extension AVX2 ; le build `x64-baseline` est alors le bon choix. Le fichier intègre l’environnement d’exécution Bun et ne requiert aucune autre dépendance :

```bash
chmod +x proton-drive
sudo install -m 0755 proton-drive /usr/local/bin/proton-drive
proton-drive version
```

Sans droits administrateur, il suffit de copier le fichier dans `~/.local/bin`, à condition que ce répertoire figure dans le `PATH`.

## Connexion, même sur des serveurs sans bureau

`auth login` ne demande pas de mot de passe sur la ligne de commande. La CLI tente d’ouvrir un navigateur et affiche également l’URL de connexion. Cette URL peut être ouverte **sur un autre appareil** ; le terminal attend que la connexion y soit terminée. L’authentification à deux facteurs se déroule normalement dans le navigateur. La connexion fonctionne donc aussi via SSH sur un serveur sans interface graphique.

```bash
proton-drive auth login
```

Après une connexion réussie, la CLI enregistre la session, et non le mot de passe. Son emplacement est déterminé par la variable d’environnement `PROTON_DRIVE_CREDENTIALS_STORE` :

| Valeur | Emplacement de stockage |
|---|---|
| `keychain` (par défaut) | Magasin de clés du système d’exploitation : Windows Credential Manager, macOS Keychain, sous Linux libsecret (GNOME Keyring, KWallet) |
| `pass` | Entrée chiffrée par GPG `ch.proton.drive/drive-sdk-cli/auth-session` dans le gestionnaire de mots de passe [pass](https://www.passwordstore.org/) |
| `unsafe_file` | Fichier en clair `auth-session.json` dans le répertoire de données ; selon Proton, uniquement pour les tests |

Sur un serveur sans session de bureau, aucun trousseau libsecret déverrouillé n’est généralement disponible. Pour ce cas, l’option `pass` existe depuis la version 0.6.0. L’utilisateur sous lequel les scripts s’exécutent a besoin d’un magasin de mots de passe initialisé et d’une clé GPG que le `gpg-agent` peut déverrouiller sans saisie interactive. La session doit être retrouvée à chaque appel grâce à la même variable, qui doit donc aussi être définie dans les tâches Cron et les unités systemd :

```bash
export PROTON_DRIVE_CREDENTIALS_STORE=pass
proton-drive auth login
```

C’est un progrès par rapport à Rclone : le mot de passe et la clé TOTP ne se trouvent pas sur le serveur, et `auth logout` met fin à l’accès. La session conserve toutefois l’intégralité des droits du compte. Il n’existe aucune restriction à certains dossiers ni accès en lecture seule. Pour les tâches automatisées, un compte Proton distinct reste donc l’option la plus sûre.

Sous Linux, le cache, les données de l’application et les journaux se trouvent dans les répertoires XDG (`~/.cache/proton-drive-cli`, `~/.local/share/proton-drive-cli`, `~/.local/state/proton-drive-cli`). Avec `PROTON_DRIVE_CACHE_DIR`, les trois peuvent être placés dans un seul répertoire, par exemple pour un conteneur avec volume monté. Par défaut, la CLI écrit les journaux avec le niveau `DEBUG` ; `PROTON_DRIVE_LOG_LEVEL=WARNING` en réduit le volume.

## Téléverser et télécharger dans des scripts

En mode interactif, la CLI demande quoi faire pour chaque conflit de nom. Cela n’est pas possible dans les scripts : avec `--json`, la question interactive est désactivée. Définissez donc toujours explicitement la stratégie de conflit pour les fichiers et les dossiers.

```bash
proton-drive filesystem upload --json \
  --file-conflict-strategy create-new-revision \
  --folder-conflict-strategy merge \
  --skip-thumbnails \
  /srv/export/berichte /my-files/backup
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `--json` (`-j`) | Produire le résultat au format JSON ; désactive les questions interactives |
| `--file-conflict-strategy` (`-f`) | Comportement lorsqu’un fichier du même nom existe : `create-new-revision` (nouvelle version du fichier existant), `rename` (ajouter un suffixe), `replace` (placer le fichier distant dans la corbeille, téléverser le fichier local), `skip` |
| `--folder-conflict-strategy` (`-d`) | Comportement lorsqu’un dossier existe déjà : `merge` (fusionner les contenus), `rename`, `replace`, `skip` |
| `--skip-thumbnails` (`-t`) | Ne pas générer de miniatures ; économise du temps de calcul pour les images |
| `/srv/export/berichte` | Source locale ; plusieurs sources sont possibles |
| `/my-files/backup` | Dossier de destination dans Proton Drive (dernier argument) |

</details>

`create-new-revision` est le bon choix pour les sauvegardes : Proton Drive conserve les versions antérieures d’un fichier, et depuis la version 0.7.0, la CLI ignore automatiquement les fichiers dont le contenu n’a pas changé. La CLI ne fait toutefois pas de synchronisation : les fichiers supprimés localement restent dans Proton Drive. Pour obtenir un miroir avec suppressions, `rclone sync` reste nécessaire.

Le téléchargement fonctionne de manière symétrique. Les stratégies diffèrent parce que la partie locale est ici écrasée :

```bash
proton-drive filesystem download --json \
  --file-conflict-strategy remove \
  --folder-conflict-strategy merge \
  /my-files/backup/berichte /srv/restore
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `--file-conflict-strategy` (`-f`) | `rename`, `remove` (supprimer le fichier local et télécharger la version distante) ou `skip` |
| `--folder-conflict-strategy` (`-d`) | `merge`, `rename`, `remove` ou `skip` |
| `/my-files/backup/berichte` | Source dans Proton Drive ; plusieurs sources sont possibles |
| `/srv/restore` | Dossier de destination local (dernier argument) |

</details>

La CLI ignore Proton Docs et Proton Sheets lors du téléchargement ; ils ne peuvent actuellement pas être exportés sous forme de fichiers.

Un téléversement régulier peut être planifié avec un timer systemd ou Cron. La sortie JSON peut ensuite être analysée avec `jq`, par exemple pour transmettre une alerte à la supervision.

## Gérer les partages

Pour l’offboarding ou les audits, la gestion des partages est souvent plus utile que le transfert de fichiers. `sharing status` affiche pour un élément tous les membres, les invitations ouvertes et les paramètres d’un lien public :

```bash
proton-drive sharing status --json /my-files/projekte/kunde-a
```

Une invitation avec droits de lecture :

```bash
proton-drive sharing invite \
  --user person@example.com \
  --role viewer \
  /my-files/projekte/kunde-a
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `--user` (`-u`) | Adresse e-mail de la personne invitée ; peut être indiquée plusieurs fois |
| `--role` (`-r`) | Rôle, `viewer` par défaut ; autres rôles selon `--help` (par ex. `editor`) |
| `--message` (`-m`) | Message dans l’e-mail d’invitation ; envoyé en **clair** |
| `--include-node-name` (`-n`) | Inclure le nom de l’élément dans l’e-mail d’invitation ; également en clair |
| `/my-files/projekte/kunde-a` | Élément à partager |

</details>

Un lien public avec mot de passe et date d’expiration :

```bash
proton-drive sharing set-url \
  --role viewer \
  --password 'Linkpasswort' \
  --expiration 2026-12-31 \
  /my-files/projekte/kunde-a/bericht.pdf
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `--role` | `viewer` (par défaut) ou `editor` |
| `--password` | Mot de passe propre au lien |
| `--expiration` | Date d’expiration au format ISO (`JJJJ-MM-TT`) |
| `/my-files/…/bericht.pdf` | Élément pour lequel le lien est créé ou modifié |

</details>

Un mot de passe transmis sur la ligne de commande figure dans l’historique du shell et est visible dans la liste des processus pendant l’exécution. Dans les scripts, il doit donc provenir d’une variable ou d’un magasin de secrets. `sharing remove-url` supprime à nouveau le lien, sans affecter les membres directs ; `sharing remove --user …` révoque l’accès de personnes individuelles.

## Takeout : export du compte

Depuis la version 0.9.0 du 5 octobre 2026, la CLI comprend la commande `takeout run`. Elle exporte une copie hors ligne du compte dans un dossier local, au choix avec `--include my-files`, `devices`, `photos` et `revisions` (toutes les versions antérieures des fichiers). Sans au moins une indication `--include`, elle n’exporte rien, et `revisions` nécessite `my-files` ou `devices`. Pour chaque dossier, elle écrit un fichier `manifest.json` qui décrit l’exportation ; le compte n’est pas modifié. La commande ne prend pas encore en charge Proton Docs et Proton Sheets. Pour une sauvegarde locale complète du contenu de Proton Drive, c’est la solution évidente.

## Limites par rapport à Rclone

| Exigence | Proton Drive CLI 0.9.0 | Rclone (backend `protondrive`) |
|---|---|---|
| Prise en charge officielle | Oui, par Proton, Open Source | Non, ingénierie inverse, bêta |
| Connexion | Navigateur, même sur un autre appareil ; session dans le magasin de clés ou `pass` | Mot de passe et clé TOTP dans le fichier de configuration |
| Téléversement, téléchargement | Oui, avec stratégies de conflit et versionnage | Oui |
| Miroir avec suppressions (`sync`) | Non | Oui |
| Monter un système de fichiers (FUSE) | Non | Oui |
| Partages, invitations, liens | Oui | Non |
| Proton Photos | Oui | Non |
| Accès aux droits limités | Non | Non |

Pour les sauvegardes, les artefacts de build et la gestion des partages, la CLI est le meilleur choix, car elle est officiellement prise en charge et ne nécessite pas de mot de passe enregistré. Pour un montage, comme celui dont a besoin par exemple une [archive documentaire Paperless](/blog/paperless-dokumente-clouddienst-auslagern), et pour les miroirs avec suppressions, Rclone reste pour l’instant nécessaire. Dans les deux cas, la plus grande lacune reste la même : Proton ne propose pas d’accès machine pouvant être limité à certains dossiers ou à la lecture seule.

## Historique des modifications

- **25 octobre 2026 :** Mise à jour vers la version 0.9.0 du 5 octobre 2026. Ajout : la commande takeout run est désormais publiée, avec des indications sur les options --include nécessaires et l’absence de prise en charge de Proton Docs et Sheets.

## Sources

1.  [Proton: Introducing Proton Drive CLI](https://proton.me/blog/proton-drive-cli) : l’annonce du 9 juin 2026 avec les cas d’utilisation et la sortie JSON.

2.  [Proton Support: Using Proton Drive CLI](https://proton.me/support/drive-cli) : guide de téléchargement, de connexion et des commandes de base.

3.  [Proton Drive CLI: Downloads](https://proton.me/download/drive/cli/index.html) : version actuelle 0.9.0, tous les builds pour plateformes avec sommes de contrôle SHA-512.

4.  [ProtonDriveApps/sdk: cli/README.md](https://github.com/ProtonDriveApps/sdk/blob/main/cli/README.md) : variables d’environnement, emplacements de stockage, magasins d’identifiants et indication concernant le build `x64-baseline`.

5.  [ProtonDriveApps/sdk: cli/CHANGELOG.md](https://github.com/ProtonDriveApps/sdk/blob/main/cli/CHANGELOG.md) : historique des versions de 0.4.2 à 0.9.0, notamment la prise en charge de `pass` (0.6.0), l’ignorance des fichiers inchangés (0.7.0) et la commande Takeout (0.9.0).

6.  [ProtonDriveApps/sdk: cli/src/commands](https://github.com/ProtonDriveApps/sdk/tree/main/cli/src/commands) : code source des commandes avec les options, stratégies de conflit et la commande Takeout.

7.  [Proton for Business: Proton Drive CLI](https://proton.me/business/drive/cli) : scénarios d’utilisation de Proton pour les entreprises, comme révoquer les partages lors du départ d’employés.

8.  [Rclone: Proton Drive](https://rclone.org/protondrive/) : le backend communautaire avec fonctions de montage et de synchronisation à titre de comparaison.
