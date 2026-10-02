---
title: "Proton Drive CLI : utiliser Proton Drive depuis des scripts et un serveur"
navTitle: "Proton Drive CLI"
description: "Depuis juin 2026, Proton propose un outil officiel en ligne de commande pour Proton Drive. Cet article décrit les commandes, l’authentification sur des serveurs sans bureau, les stratégies de conflit pour les scripts et les limites par rapport à Rclone."
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
url: https://rafaelpfister.ch/fr/blog/proton-drive-cli-utiliser-proton-drive-depuis-des-scripts-et-un-serveur
translationSourceHash: e71e82deea6466312d0d95edc2c96890fc0810219a21308b4cb9346df3f32341
translationModel: gpt-5.6-terra
translatedAt: 2026-10-02T09:58:14.437Z
translationReview: automatic
---

Le 9 juin 2026, Proton a publié **Proton Drive CLI**, un outil officiel en ligne de commande pour Windows, macOS et Linux. Il repose sur le même SDK que les applications Drive officielles, chiffre de bout en bout et est disponible sous la forme d’un unique fichier exécutable `proton-drive`. Le code source se trouve dans le dépôt public du SDK, sous `cli/`.

La CLI est conçue pour des opérations individuelles et ponctuelles : téléverser des fichiers après un build, sauvegarder un dossier selon un calendrier, vérifier ou retirer des partages. Elle ne synchronise pas en arrière-plan et ne monte pas de système de fichiers. La version actuelle est la **0.8.0 du 13 août 2026** ; son numéro indique que les commandes et options peuvent encore changer (la version 0.8.0 a renommé les stratégies de conflit avec une modification incompatible).

La place de cet outil parmi les autres options Linux (Rclone, client de bureau annoncé) est décrite dans l’article de statut [Proton Drive sous Linux](/blog/proton-drive-linux-status).

## Aperçu des commandes

Les commandes sont organisées en groupes : `proton-drive <gruppe> <befehl> [optionen] [argumente]`. Les noms de groupes peuvent être abrégés tant qu’ils restent non ambigus ; pour `filesystem`, l’alias `fs` est également disponible. Sans argument, un shell interactif démarre. L’aide complète est fournie par `proton-drive help` ou `proton-drive <gruppe> <befehl> --help`.

<details class="options-details">
<summary>Aperçu des options</summary>

| Groupe / commande | Effet |
|---|---|
| `auth login` / `auth logout` | Connexion via le navigateur ; la déconnexion supprime les identifiants locaux et les caches |
| `filesystem list <pfad>` | Lister le contenu d’un dossier ; `/` affiche les zones racines |
| `filesystem info` / `size` | Métadonnées d’un élément ou taille d’un dossier, y compris le contenu de la corbeille |
| `filesystem upload` / `download` | Téléverser ou télécharger des fichiers et dossiers |
| `filesystem create-folder`, `rename`, `copy`, `move` | Créer, renommer, copier, déplacer des dossiers |
| `filesystem trash` / `restore` | Déplacer vers la corbeille ou restaurer |
| `filesystem delete` / `empty-trash` | Supprimer définitivement ou vider `/trash` |
| `sharing status <pfad>` | Afficher les membres, les invitations en attente et les paramètres des liens |
| `sharing invite` / `remove` | Inviter des personnes par e-mail ou retirer leur accès |
| `sharing set-url` / `remove-url` | Créer, modifier ou supprimer un lien public |
| `sharing leave` / `report` | Quitter un partage reçu ou le signaler comme abusif |
| `invitation list` / `accept` / `reject` | Gérer les invitations reçues |
| `album …`, `photo timeline`, `photo upload`, `photo download` | Proton Photos : albums et chronologie |
| `version` | Afficher les versions de la CLI et du SDK |
| `--json` (`-j`) | Sortie JSON lisible par machine, pour chaque commande |
| `--verbose` (`-v`) | Sortie de journalisation directement dans la console |
| `--help` (`-h`) | Aide relative à la commande concernée |

</details>

Les chemins dans Proton Drive sont toujours des chemins POSIX, y compris sous Windows. La racine `/` contient des zones virtuelles : `/my-files` (fichiers personnels), `/devices` (ordinateurs sauvegardés), `/shared-by-me`, `/shared-with-me`, `/trash` ainsi que les zones Photos `/photos`, `/albums`, `/photos-shared-by-me`, `/photos-shared-with-me` et `/photos-trash`.

## Installation sous Linux

Proton propose les builds sur une page de téléchargement dédiée, chacun avec une somme de contrôle SHA-512. Cinq variantes sont disponibles pour Linux :

| Build | Usage |
|---|---|
| `linux/x64` | Standard pour les systèmes x86-64 actuels |
| `linux/x64-baseline` | x86-64 sans AVX2, p. ex. appareils NAS et processeurs de serveur plus anciens |
| `linux/arm64` | Serveurs ARM et ordinateurs monocarte avec glibc |
| `linux/x64-musl`, `linux/arm64-musl` | Distributions utilisant musl au lieu de glibc, p. ex. Alpine Linux et les images de conteneurs qui en dérivent |

Si le build standard s’interrompt au démarrage avec `Illegal instruction`, l’extension AVX2 manque au processeur ; le build `x64-baseline` est alors le bon choix. Le fichier embarque l’environnement d’exécution Bun et ne nécessite aucune autre dépendance :

```bash
chmod +x proton-drive
sudo install -m 0755 proton-drive /usr/local/bin/proton-drive
proton-drive version
```

Sans droits d’administrateur, il suffit de copier le fichier dans `~/.local/bin`, à condition que ce répertoire se trouve dans le `PATH`.

## Connexion, y compris sur des serveurs sans bureau

`auth login` ne demande pas de mot de passe en ligne de commande. La CLI tente d’ouvrir un navigateur et affiche également l’URL de connexion. Cette URL peut être ouverte **sur un autre appareil** ; le terminal attend que la connexion y soit terminée. L’authentification à deux facteurs s’effectue normalement dans le navigateur. La connexion fonctionne ainsi également par SSH sur un serveur sans interface graphique.

```bash
proton-drive auth login
```

Après une connexion réussie, la CLI enregistre la session, et non le mot de passe. Son emplacement est déterminé par la variable d’environnement `PROTON_DRIVE_CREDENTIALS_STORE` :

| Valeur | Emplacement de stockage |
|---|---|
| `keychain` (standard) | Gestionnaire de clés du système d’exploitation : Windows Credential Manager, macOS Keychain, sous Linux libsecret (GNOME Keyring, KWallet) |
| `pass` | Entrée chiffrée avec GPG `ch.proton.drive/drive-sdk-cli/auth-session` dans le gestionnaire de mots de passe [pass](https://www.passwordstore.org/) |
| `unsafe_file` | Fichier texte brut `auth-session.json` dans le répertoire de données ; selon Proton, uniquement destiné aux tests |

Sur un serveur sans session de bureau, il n’y a généralement pas de trousseau libsecret déverrouillé. Depuis la version 0.6.0, l’option `pass` est prévue pour ce cas. L’utilisateur exécutant les scripts doit disposer d’un magasin de mots de passe initialisé et d’une clé GPG que le `gpg-agent` peut déverrouiller sans saisie interactive. La session doit être trouvée via la même variable à chaque appel, qui doit donc aussi être définie dans les tâches Cron et les unités systemd :

```bash
export PROTON_DRIVE_CREDENTIALS_STORE=pass
proton-drive auth login
```

C’est un progrès par rapport à Rclone : ni mot de passe ni clé TOTP ne se trouvent sur le serveur, et `auth logout` met fin à l’accès. La session conserve toutefois l’étendue complète du compte. Il n’existe aucune restriction à des dossiers individuels ou à un accès en lecture seule. Pour les processus automatisés, un compte Proton distinct reste donc l’option la plus sûre.

Sous Linux, le cache, les données d’application et les journaux se trouvent dans les répertoires XDG (`~/.cache/proton-drive-cli`, `~/.local/share/proton-drive-cli`, `~/.local/state/proton-drive-cli`). Avec `PROTON_DRIVE_CACHE_DIR`, les trois peuvent être placés dans un seul répertoire, par exemple pour un conteneur avec un volume monté. Par défaut, la CLI écrit les journaux au niveau `DEBUG` ; `PROTON_DRIVE_LOG_LEVEL=WARNING` réduit leur quantité.

## Téléversement et téléchargement dans des scripts

En mode interactif, la CLI demande quoi faire pour chaque conflit de noms. Cela n’est pas possible dans les scripts : avec `--json`, la question interactive est désactivée. Définissez donc toujours explicitement la stratégie de conflit pour les fichiers et dossiers.

```bash
proton-drive filesystem upload --json \
  --file-conflict-strategy create-new-revision \
  --folder-conflict-strategy merge \
  --skip-thumbnails \
  /srv/export/berichte /my-files/backup
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `--json` (`-j`) | Afficher le résultat au format JSON ; désactive les questions interactives |
| `--file-conflict-strategy` (`-f`) | Comportement lorsqu’un fichier du même nom existe : `create-new-revision` (nouvelle version du fichier existant), `rename` (ajouter un suffixe), `replace` (fichier distant dans la corbeille, téléversement du fichier local), `skip` |
| `--folder-conflict-strategy` (`-d`) | Comportement lorsqu’un dossier existe déjà : `merge` (fusionner les contenus), `rename`, `replace`, `skip` |
| `--skip-thumbnails` (`-t`) | Ne pas créer de miniatures ; économise du temps de calcul pour les images |
| `/srv/export/berichte` | Source locale ; plusieurs sources sont possibles |
| `/my-files/backup` | Dossier cible dans Proton Drive (dernier argument) |

</details>

`create-new-revision` est le choix approprié pour les sauvegardes : Proton Drive conserve les versions antérieures d’un fichier et, depuis la version 0.7.0, la CLI ignore automatiquement les fichiers dont le contenu n’a pas changé. La CLI ne réalise toutefois pas de synchronisation : les fichiers supprimés localement restent dans Proton Drive. Pour un miroir incluant les suppressions, il faut toujours recourir à `rclone sync`.

Le téléchargement fonctionne de façon symétrique. Les stratégies diffèrent car le côté local est ici écrasé :

```bash
proton-drive filesystem download --json \
  --file-conflict-strategy remove \
  --folder-conflict-strategy merge \
  /my-files/backup/berichte /srv/restore
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `--file-conflict-strategy` (`-f`) | `rename`, `remove` (supprimer le fichier local et télécharger la version distante) ou `skip` |
| `--folder-conflict-strategy` (`-d`) | `merge`, `rename`, `remove` ou `skip` |
| `/my-files/backup/berichte` | Source dans Proton Drive ; plusieurs sources sont possibles |
| `/srv/restore` | Dossier cible local (dernier argument) |

</details>

La CLI ignore Proton Docs et Proton Sheets lors du téléchargement ; ils ne peuvent actuellement pas être exportés en tant que fichiers.

Un téléversement régulier peut être planifié avec un minuteur systemd ou Cron. La sortie JSON peut ensuite être analysée avec `jq`, par exemple pour envoyer une notification à la supervision.

## Gérer les partages

Pour l’offboarding ou les audits, la gestion des partages est souvent plus utile que le transfert de fichiers. `sharing status` affiche pour un élément tous les membres, les invitations en attente et les paramètres d’un lien public :

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
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `--user` (`-u`) | Adresse e-mail de la personne invitée ; peut être indiquée plusieurs fois |
| `--role` (`-r`) | Rôle, par défaut `viewer`; autres rôles selon `--help` (p. ex. `editor`) |
| `--message` (`-m`) | Message dans l’e-mail d’invitation ; envoyé en **texte brut** |
| `--include-node-name` (`-n`) | Inclure le nom de l’élément dans l’e-mail d’invitation ; également en texte brut |
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
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `--role` | `viewer` (par défaut) ou `editor` |
| `--password` | Mot de passe propre au lien |
| `--expiration` | Date d’expiration au format ISO (`JJJJ-MM-TT`) |
| `/my-files/…/bericht.pdf` | Élément pour lequel le lien est créé ou modifié |

</details>

Un mot de passe transmis en ligne de commande apparaît dans l’historique du shell et reste visible dans la liste des processus pendant l’exécution. Dans les scripts, il devrait donc provenir d’une variable ou d’un stockage de secrets. `sharing remove-url` supprime à nouveau le lien sans affecter les membres directs ; `sharing remove --user …` retire l’accès à certaines personnes.

## En cours : Takeout

Depuis le 10 septembre 2026, le dépôt du SDK contient une commande supplémentaire `takeout run`. Elle exporte une copie hors ligne du compte vers un dossier local, au choix avec `--include my-files`, `devices`, `photos` et `revisions` (toutes les versions antérieures des fichiers). Pour chaque dossier, elle écrit un fichier `manifest.json` décrivant l’export ; elle ne modifie rien au compte. La commande n’est pas encore incluse dans la version publiée 0.8.0. Dès qu’elle paraîtra, elle sera la solution évidente pour une sauvegarde locale complète du contenu de Proton Drive.

## Limites par rapport à Rclone

| Exigence | Proton Drive CLI 0.8.0 | Rclone (`protondrive`-backend) |
|---|---|---|
| Prise en charge officielle | Oui, par Proton, Open Source | Non, ingénierie inverse, bêta |
| Connexion | Navigateur, également sur un autre appareil ; session dans le gestionnaire de clés ou `pass` | Mot de passe et clé TOTP dans le fichier de configuration |
| Téléversement, téléchargement | Oui, avec stratégies de conflit et versionnage | Oui |
| Miroir incluant les suppressions (`sync`) | Non | Oui |
| Monter un système de fichiers (FUSE) | Non | Oui |
| Partages, invitations, liens | Oui | Non |
| Proton Photos | Oui | Non |
| Accès à portée limitée | Non | Non |

Pour les sauvegardes, les artefacts de build et la gestion des partages, la CLI est le meilleur choix, car elle est officiellement prise en charge et ne nécessite pas de mot de passe enregistré. Pour un montage, tel que le nécessite par exemple une [archive documentaire Paperless](/blog/paperless-dokumente-clouddienst-auslagern), et pour les miroirs incluant les suppressions, Rclone reste pour l’instant indispensable. Dans les deux cas, la principale lacune demeure la même : Proton ne propose aucun accès machine pouvant être limité à certains dossiers ou à la lecture seule.

## Sources

1.  [Proton: Introducing Proton Drive CLI](https://proton.me/blog/proton-drive-cli): annonce du 9 juin 2026 avec les cas d’usage et la sortie JSON.

2.  [Proton Support: Using Proton Drive CLI](https://proton.me/support/drive-cli): guide pour le téléchargement, la connexion et les commandes de base.

3.  [Proton Drive CLI: Downloads](https://proton.me/download/drive/cli/index.html): version actuelle 0.8.0, tous les builds de plateforme avec sommes de contrôle SHA-512.

4.  [ProtonDriveApps/sdk: cli/README.md](https://github.com/ProtonDriveApps/sdk/blob/main/cli/README.md): variables d’environnement, emplacements de stockage, gestionnaires d’identifiants et indication concernant le build `x64-baseline`.

5.  [ProtonDriveApps/sdk: cli/CHANGELOG.md](https://github.com/ProtonDriveApps/sdk/blob/main/cli/CHANGELOG.md): historique des versions de 0.4.2 à 0.8.0, notamment la prise en charge de `pass` (0.6.0) et l’ignorance des fichiers inchangés (0.7.0).

6.  [ProtonDriveApps/sdk: cli/src/commands](https://github.com/ProtonDriveApps/sdk/tree/main/cli/src/commands): code source des commandes avec options, stratégies de conflit et commande Takeout encore non publiée.

7.  [Proton for Business: Proton Drive CLI](https://proton.me/business/drive/cli): cas d’usage de Proton pour les entreprises, par exemple retirer les partages lors du départ de collaborateurs.

8.  [Rclone: Proton Drive](https://rclone.org/protondrive/): backend communautaire avec fonctions de montage et de synchronisation à titre de comparaison.
