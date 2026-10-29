---
title: "git.kernel.org expliqué : structure du site et utilisation pratique"
navTitle: "git.kernel.org"
description: "git.kernel.org est l’hébergement Git du développement du noyau Linux. Cette vue d’ensemble explique la liste des dépôts, le schéma des chemins, les vues summary, log, tree et commit, les paramètres de recherche dans l’URL et les URL courtes auxquelles renvoient les CVE du noyau."
date: "2026-10-29"
kategorie: "Noyau Linux"
timeToRead: "9 min de lecture"
themen:
  - linux-kernel
produkte:
  - "linux-kernel"
protokolle:
  - "releases"
  - "troubleshooting"
slug: "git-kernel-org-explique-structure-du-site-et-utilisation-pratique"
translationId: "article-91dab903d58c6173"
translationOf: git-kernel-org-erklaert
url: https://rafaelpfister.ch/fr/blog/git-kernel-org-explique-structure-du-site-et-utilisation-pratique
translationSourceHash: 34a3d0d525d79d1ef4c5ff31824dabdd6e910a1dd5a22e60c77b202f97a71b7d
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T14:32:25.882Z
translationReview: automatic
---

[git.kernel.org](https://git.kernel.org/) est le serveur Git de kernel.org. On y trouve la branche principale de développement de Linus Torvalds, les noyaux Stable, linux-next, des centaines d’arbres de sous-systèmes et de développeurs, ainsi que des outils comme `git` lui-même, `e2fsprogs` ou les pages de manuel Linux. L’interface web est une version adaptée de [cgit](https://git.zx2c4.com/cgit/about/) (indiquée dans le pied de page comme `cgit 1.3.1-korg`). Elle est en lecture seule : il n’y a ni comptes, ni issues, ni pull requests. Le développement du noyau passe par des listes de diffusion, dont l’archive se trouve sur [lore.kernel.org](https://lore.kernel.org/) ; git.kernel.org affiche le résultat, c’est-à-dire les commits que les mainteneurs ont intégrés dans leurs arbres.

On le consulte généralement à partir d’une entrée CVE renvoyant vers un commit du noyau, pour déterminer si un correctif donné est inclus dans une version du noyau, ou pour lire un fichier source précis sans cloner tout le dépôt.

## La page d’accueil : liste des dépôts

La page d’accueil répertorie tous les dépôts publics dans un tableau. Le regroupement suit le chemin des fichiers sur le serveur : chaque intertitre tel que `pub/scm/fs` ou `pub/scm/linux` correspond à un répertoire, sous lequel figurent les dépôts avec leur chemin relatif à ce répertoire.

<details class="options-details">
<summary>Vue d’ensemble des colonnes</summary>

| Colonne | Contenu |
|---|---|
| Name | Chemin du dépôt sous le groupe, par exemple `xfs/xfs-linux.git` sous `pub/scm/fs` |
| Description | Contenu du fichier `description` dans le dépôt. « Unnamed repository; edit this file … » est le texte standard de Git et signifie seulement que l’opérateur n’a pas défini de description |
| Owner | personne ou groupe responsable (par ex. « XFS FS Group », « Infra Team ») |
| Idle | temps écoulé depuis le dernier commit sur une branche quelconque ; une faible valeur indique un arbre activement maintenu |
| Links | accès directs aux vues `summary`, `log` et `tree` du dépôt |

</details>

Les principaux groupes :

| Groupe | Contenu |
|---|---|
| `pub/scm/linux` | Arbres du noyau : Mainline, Stable, linux-next, ainsi que les arbres personnels et de sous-systèmes des mainteneurs |
| `pub/scm/fs` | Systèmes de fichiers avec leurs outils en espace utilisateur (ext2/3/4, XFS, fscrypt, fsverity) |
| `pub/scm/libs` | Bibliothèques telles que `libcap`, `libgpiod`, `libtraceevent` |
| `pub/scm/git` | Le projet Git lui-même (`git.git`) avec documentation HTML et pages de manuel |
| `pub/scm/docs` | Sources de documentation, dont `man-pages` et les clés PGP des développeurs du noyau (`kernel/pgpkeys.git`) |
| `pub/scm/infra` | Infrastructure de kernel.org, par ex. le fork de cgit et les journaux de transparence du serveur Gitolite |
| `pub/scm/bluetooth`, `devel`, `editors` | BlueZ, outils de développement tels que `sparse` et `pahole`, `uemacs` de Linus Torvalds |

La liste est très longue, car sous `pub/scm/linux/kernel/git/` à lui seul, chaque mainteneur disposant d’un compte peut déposer ses propres arbres. Le champ de recherche en haut à droite filtre la liste selon le nom, la description et le propriétaire. La recherche peut aussi être appelée directement via une URL, par exemple `https://git.kernel.org/?q=xfs`.

## Le schéma des chemins des arbres du noyau

Les dépôts du noyau suivent le modèle :

```text
https://git.kernel.org/pub/scm/linux/kernel/git/<konto>/<repository>.git/
```

`<konto>` est le nom d’utilisateur du mainteneur sur kernel.org, et pas nécessairement son nom complet (`akpm` est Andrew Morton, `acme` est Arnaldo Carvalho de Melo). Pour la plupart des questions, trois arbres suffisent :

| Arbre | Chemin | Objectif |
|---|---|---|
| Mainline | `torvalds/linux.git` | Branche de développement de Linus Torvalds ; c’est ici que sont créées les versions `v7.3`, `v7.4-rc1` etc. |
| Stable | `stable/linux.git` | Versions maintenues avec correctifs (`v7.3.1`, `v7.3.2` …) et noyaux Longterm ; une branche par série de versions, par ex. `linux-7.3.y` |
| linux-next | `next/linux-next.git` | État quotidiennement fusionné de tous les arbres de sous-systèmes pour la prochaine fenêtre de fusion |

Un correctif passe généralement par ces arbres dans cet ordre : il arrive d’abord dans l’arbre du sous-système responsable, apparaît dans linux-next, est intégré à la Mainline pendant la fenêtre de fusion par Linus Torvalds via un e-mail de pull request, puis est rétroporté vers les branches Stable. L’arbre de sous-système responsable est indiqué dans le fichier `MAINTAINERS` du code source du noyau (lignes contenant `T: git …`).

## Les vues d’un dépôt

Un clic sur le nom d’un dépôt ouvre la vue `summary`. La barre d’onglets au-dessus mène aux autres vues ; à sa gauche, une liste déroulante sélectionne la branche à laquelle toutes les vues se rapportent.

<details class="options-details">
<summary>Vue d’ensemble des vues</summary>

| Vue | Contenu |
|---|---|
| about | README du dépôt, s’il est présent |
| summary | Branches avec le dernier commit, tags les plus récents avec lien de téléchargement, dix derniers commits et URL de clonage |
| refs | Liste complète de toutes les branches et de tous les tags |
| log | Historique des commits avec champ de recherche ; colonnes âge, message de commit, auteur, fichiers et lignes modifiés |
| tree | Arborescence des fichiers pour la branche ou le commit sélectionné, avec liens vers `plain` (fichier brut) et `log` (historique du fichier) |
| commit | Métadonnées et diff d’un commit individuel |
| diff | Différence entre deux commits ou entre un commit et son prédécesseur |
| stats | Statistiques de commits par auteur sur une période sélectionnable |

</details>

Dans la vue `summary`, le bloc des tags permet de connaître la version actuelle : la Mainline affiche les release candidates (`v7.3-rc7`) et les versions publiées (`v7.3`). Les liens de téléchargement génèrent une archive à la demande et ne sont donc pas toujours disponibles ; pour les versions finalisées, les tarballs sur [kernel.org](https://kernel.org/) ou `cdn.kernel.org` constituent la méthode de téléchargement prévue, avec signature PGP.

À la fin de la vue `summary` figurent les URL de clonage. Outre `https://` et `git://` non chiffré, un miroir chez Google (`kernel.googlesource.com`) est indiqué et fournit le même contenu.

## Rechercher dans l’historique

Le champ de recherche de la vue `log` propose une liste déroulante avec quatre modes. Chaque mode est aussi disponible comme paramètre d’URL, ce qui permet d’enregistrer les recherches comme favoris ou de créer des liens dans des tickets :

```text
https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?qt=grep&q=CVE
```

<details class="options-details">
<summary>Explication des options</summary>

| Paramètre | Effet |
|---|---|
| `qt=grep` | recherche dans les messages de commit (sélection « log msg ») |
| `qt=author` | recherche dans le champ auteur (nom ou adresse e-mail) |
| `qt=committer` | recherche dans le champ committer, c’est-à-dire la personne ayant intégré le commit |
| `qt=range` | affiche une plage en notation Git, par ex. `q=v7.3-rc6..v7.3-rc7` pour tous les commits entre deux tags |
| `q=` | terme de recherche ou plage |
| `h=` | branche à laquelle se rapporte la recherche, par ex. `h=linux-7.3.y` dans l’arbre Stable |
| `ofs=` | décalage pour la pagination (git.kernel.org affiche 200 commits par page) |

</details>

La plage `v7.3-rc6..v7.3-rc7` répond directement dans le navigateur à la question « Qu’est-ce qui a changé entre deux versions ? ». Dans l’arbre Stable, `qt=range&q=v7.3.1..v7.3.2&h=linux-7.3.y` affiche tous les correctifs ajoutés avec une mise à jour Stable.

## Lire un commit

Un commit individuel est adressable par son hash SHA-1 complet :

```text
https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=<sha>
```

La vue affiche l’auteur, le committer, la date, le commit parent (`parent`), le message de commit complet et le diff. À la fin d’un message de commit du noyau figurent des trailers tels que `Signed-off-by:`, `Reviewed-by:` et `Fixes: <sha> ("Titel")`. La ligne `Fixes:` indique le commit qui a introduit l’erreur ; elle permet ainsi de déterminer à partir de quelle version un système est affecté. `Link:` renvoie généralement vers la discussion correspondante sur lore.kernel.org, `Cc: stable@vger.kernel.org` marque le correctif à rétroporter vers les noyaux Stable.

En remplaçant `commit` dans l’URL par `patch`, le serveur fournit le commit sous forme de fichier patch au format e-mail (`text/plain`), applicable avec `git am` ou `patch`.

## URL courtes dans les entrées CVE

Depuis que le projet du noyau attribue lui-même des numéros CVE, les entrées dans la liste CVE et la NVD renvoient vers des URL courtes de la forme `https://git.kernel.org/stable/c/<sha>`. git.kernel.org les redirige via HTTP 301 vers la vue du commit dans l’arbre correspondant :

| URL courte | Destination |
|---|---|
| `https://git.kernel.org/stable/c/<sha>` | Commit dans l’arbre Stable `stable/linux.git` |
| `https://git.kernel.org/linus/<sha>` | Commit dans la Mainline `torvalds/linux.git` |

Une entrée CVE mentionne généralement plusieurs liens de ce type : un par branche Stable dans laquelle le correctif a été rétroporté, et un pour la Mainline. Les hash diffèrent parce que chaque rétroportage constitue un commit distinct. Le message de commit du rétroportage indique le hash de l’original dans la Mainline, soit sous forme de ligne `commit <sha> upstream.`, soit sous forme de `[ Upstream commit <sha> ]`.

## Vérifier si un correctif est inclus dans une version

L’interface web n’indique pas dans quels tags un commit est inclus. Un clone local est nécessaire pour cela. À cette fin, un clone sans contenu de fichiers, qui ne télécharge que l’historique des commits, suffit :

```bash
git clone --filter=blob:none --no-checkout https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `clone` | crée une copie locale du dépôt |
| `--filter=blob:none` | Clone partiel : télécharge les commits et les structures de répertoires, les contenus de fichiers (blobs) seulement à la demande |
| `--no-checkout` | ne crée pas de répertoire de travail ; inutile pour interroger l’historique |
| URL | URL de clonage HTTPS de la vue `summary` de l’arbre Stable |

</details>

Ensuite, une commande dans le répertoire cloné répond à la question de la première version contenant le correctif :

```bash
git describe --contains <sha>
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `describe` | décrit un commit par rapport à un tag |
| `--contains` | recherche le tag le plus ancien contenant le commit, plutôt que le tag le plus récent qui le précède |
| `<sha>` | Hash du commit, par ex. provenant de l’URL courte dans l’entrée CVE |

</details>

La sortie a la forme `v7.3.2~14` : le correctif est inclus à partir de `v7.3.2`. `git tag --contains <sha>` répertorie tous les tags contenant le commit. Si l’on a seulement besoin de lire un fichier individuel ou un état donné, on peut utiliser les vues `tree` et `plain` dans le navigateur plutôt qu’un clone.

Pour disposer du code source complet à compiler, un clone superficiel d’un tag suffit :

```bash
git clone --depth 1 --branch v7.3 https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `--depth 1` | ne télécharge que le dernier commit sans historique ; le téléchargement passe de plusieurs Go à quelques centaines de Mo |
| `--branch v7.3` | sélectionne le tag ou la branche à extraire |
| URL | URL de clonage de la Mainline |

</details>

## Suivre les modifications

Chaque dépôt propose un flux Atom de la branche par défaut sous `…/<repository>.git/atom/`, donc pour la Mainline `https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/atom/`. Un lecteur de flux affiche ainsi les nouveaux commits sans qu’il soit nécessaire de consulter régulièrement la page. Pour la Mainline, le flux génère de nombreuses entrées par jour durant la fenêtre de fusion ; pour les versions publiées et les mises à jour Stable, l’aperçu des versions sur [kernel.org](https://kernel.org/) ou son flux constitue une source plus calme.

## Sources

1.  [git.kernel.org](https://git.kernel.org/) : liste des dépôts avec groupes, colonnes Owner et Idle ; champ de recherche de la page d’accueil.

2.  [torvalds/linux.git](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/) : Mainline avec les vues summary, log, tree, les URL de clonage et le flux Atom.

3.  [stable/linux.git](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/) : noyaux Stable et Longterm, une branche par série de versions.

4.  [cgit](https://git.zx2c4.com/cgit/about/) : amont de l’interface web, documentation des paramètres d’URL.

5.  [Kernel Docs: Everything you ever wanted to know about Linux -stable releases](https://docs.kernel.org/process/stable-kernel-rules.html) : règles de rétroportage, trailer `Cc: stable` et référence au commit amont.

6.  [Kernel Docs: CVEs](https://docs.kernel.org/process/cve.html) : attribution des CVE par le projet du noyau et structure des entrées.

7.  [Kernel Docs: Submitting patches](https://docs.kernel.org/process/submitting-patches.html) : signification des trailers `Fixes:`, `Link:`, `Signed-off-by:`.

8.  [git-describe](https://git-scm.com/docs/git-describe) : page de manuel, option `--contains`.

9.  [git-clone](https://git-scm.com/docs/git-clone) : page de manuel, options `--filter`, `--depth`, `--branch`, `--no-checkout`.

10.  [lore.kernel.org](https://lore.kernel.org/) : archive des listes de diffusion du noyau, vers laquelle renvoient les trailers `Link:`.
