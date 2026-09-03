---
title: "Tester le stockage à froid avec Rclone : un plan de test pratique"
navTitle: "Tester Rclone"
description: "Avant qu’un service lise ses fichiers depuis le cloud via un montage Rclone, il convient de vérifier davantage que l’accès aux répertoires. Ce plan de test couvre les lectures à froid, les lectures à chaud, les écritures, le comportement du cache, l’intégrité des fichiers et les pannes."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "11 min de lecture"
themen:
  - rclone
  - testing
related:
  - rclone-mount-in-docker-container
  - paperless-dokumente-clouddienst-auslagern
slug: "tester-le-stockage-a-froid-avec-rclone-un-plan-de-test-pratique"
translationOf: "cloud-mount-testen-dummy-pdfs"
translationId: article-8592f808b2e93cd4
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:19:15.650Z
translationReview: required
translationSourceHash: 27bc45a50d8e84fc785eaf04ec6814054e327f516587d0f9f9a101c989ce2aa1
url: https://rafaelpfister.ch/fr/blog/tester-le-stockage-a-froid-avec-rclone-un-plan-de-test-pratique
---

Un montage Rclone se met rapidement en place. Le remote apparaît comme un répertoire, `ls` affiche des fichiers et le premier test fonctionnel est réussi. Cela ne dit toutefois pas grand-chose sur l’exploitation en production.

Dès qu’un service accède au montage, d’autres questions se posent : combien de temps prend le premier accès à un fichier ? Quels accès sont servis par le cache local ? Que se passe-t-il avec un fichier pas encore téléversé si Rclone plante ? Un conteneur en cours d’exécution voit-il à nouveau le montage recréé ? Et comment le service réagit-il lorsque le cloud est temporairement inaccessible ?

Cet article fournit un plan de test général à cet effet. Vous pouvez l’utiliser pour une archive de documents, un serveur multimédia, une gestion de photos ou tout autre service qui accède à des fichiers rarement nécessaires via Rclone depuis un stockage à froid.

## Les principales options de rclone

Pour vous orienter, voici les options Rclone utilisées dans ce plan de test, traduites de manière libre depuis la documentation :

<details class="options-details">
<summary>Aperçu des options</summary>

| Option | Signification |
|---|---|
| `--seed n` | Valeur initiale du générateur aléatoire dans `rclone test makefiles` ; la même seed produit la même arborescence de fichiers |
| `--files n` | Nombre de fichiers de test à générer |
| `--files-per-directory n` | Nombre moyen de fichiers par répertoire |
| `--min-file-size grösse` | Taille minimale des fichiers générés (suffixes tels que K, M, G autorisés) |
| `--max-file-size grösse` | Taille maximale des fichiers générés |
| `--progress` | Affichage continu de la progression pendant le transfert |
| `--stats dauer` | Intervalle auquel les statistiques de transfert sont affichées |
| `--log-file datei` | Écrit le journal dans le fichier indiqué |
| `--log-level stufe` | Niveau de détail du journal : DEBUG, INFO, NOTICE ou ERROR |
| `--one-way` | Vérifie avec `rclone check` uniquement que tous les fichiers source existent dans la destination et sont identiques ; les fichiers supplémentaires dans la destination ne sont pas considérés comme une erreur |
| `--download` | Télécharge réellement les données lors de la comparaison, au lieu de comparer uniquement les hash |
| `--vfs-cache-mode modus` | Stratégie de cache de la couche VFS ; `full` met en tampon localement les accès en lecture et en écriture |
| `--cache-dir verzeichnis` | Répertoire du cache local |
| `--vfs-cache-max-size grösse` | Limite souple de la taille totale du cache VFS |
| `--vfs-cache-poll-interval dauer` | Intervalle auquel Rclone vérifie et nettoie le cache |
| `--vfs-write-back dauer` | Délai après la fermeture d’un fichier avant le début de son téléversement vers le remote |
| `--vfs-read-ahead grösse` | Quantité supplémentaire de données lues à l’avance au-delà de la position demandée avec `full` |
| `--poll-interval dauer` | Intervalle auquel Rclone interroge le remote pour détecter les modifications (uniquement pour les backends prenant en charge le polling) |
| `--dir-cache-time dauer` | Durée de validité des listes de répertoires mises en cache |
| `--allow-other` | Autorise des utilisateurs autres que celui qui effectue le montage à accéder au montage FUSE |

</details>

Les listes complètes sont disponibles dans la documentation Rclone, notamment sous [rclone mount](https://rclone.org/commands/rclone_mount/) et dans l’aperçu des [flags globaux](https://rclone.org/flags/).

## Définir d’abord ce que vous souhaitez atteindre

Le stockage à froid ne signifie pas automatiquement la même chose pour chaque application. Un serveur multimédia lit généralement de gros fichiers de manière séquentielle. Une gestion de photos charge de nombreuses petites vignettes et saute à différents endroits. Une archive de documents ouvre des fichiers relativement petits, mais souvent une seule fois.

Avant le test, notez les principales caractéristiques de votre véritable jeu de données :

- taille de fichier typique et fichier le plus volumineux
- nombre de fichiers par répertoire
- lecture complète ou accès aléatoire à certaines zones
- rapport entre les accès en lecture et en écriture
- nombre d’utilisateurs ou de processus simultanés
- modifications effectuées directement dans le remote en dehors du montage
- délai d’attente acceptable pour une lecture à froid
- espace maximal disponible pour le cache local

Ce n’est qu’à partir de là que des critères de réussite pertinents peuvent être définis. Ouvrir un seul fichier en 1,2 seconde peut être parfaitement acceptable pour une archive, mais inutilisable pour une application interactive.

## Créer un jeu de test reproductible

Rclone intègre déjà un outil adapté à cet effet. `rclone test makefiles` génère à chaque fois la même arborescence de fichiers avec une seed fixe :

```bash
rclone test makefiles ./testdata \
  --seed 42 \
  --files 250 \
  --files-per-directory 25 \
  --min-file-size 16K \
  --max-file-size 32M
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `./testdata` | Répertoire cible dans lequel l’arborescence de test est créée |
| `--seed 42` | Valeur initiale fixe du générateur aléatoire ; chaque exécution génère le même jeu de données |
| `--files 250` | 250 fichiers de test au total |
| `--files-per-directory 25` | En moyenne 25 fichiers par répertoire |
| `--min-file-size 16K` | Plus petit fichier : 16 KiB |
| `--max-file-size 32M` | Plus gros fichier : 32 MiB |

</details>

Adaptez le nombre et les tailles à votre véritable jeu de données. Ne testez pas uniquement des fichiers moyens. Quelques très petits fichiers montrent le coût des accès aux métadonnées ; quelques gros fichiers révèlent le débit, la lecture anticipée et le comportement du cache.

Ajoutez également des noms de fichiers susceptibles de poser problème en pratique :

```bash
mkdir -p "testdata/Sonderfälle/Unterordner"
printf 'Leerzeichen\n' > "testdata/Sonderfälle/Datei mit Leerzeichen.txt"
printf 'Umlaute\n' > "testdata/Sonderfälle/Grösse und Änderung.txt"
printf 'Grossschreibung\n' > "testdata/Sonderfälle/Test.txt"
printf 'Kleinschreibung\n' > "testdata/Sonderfälle/test.txt"
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `mkdir -p` | Crée également les répertoires parents manquants et ne signale pas d’erreur si le répertoire existe |
| `printf '…\n' > datei` | Écrit le texte indiqué comme contenu du fichier ; la redirection crée le fichier portant le nom problématique |

</details>

Le dernier test est particulièrement important lorsque le système de fichiers local et le backend cloud traitent différemment les majuscules et minuscules.

Si votre service n’accepte que certains formats, des fichiers binaires quelconques ne suffisent pas. Générez alors en complément des fichiers synthétiques exactement dans ces formats. Avec Paperless-ngx, il s’agissait de PDF avec une véritable couche de texte, afin que le test ne mesure pas par inadvertance les performances de l’OCR au lieu du chemin de stockage. Pour une gestion de photos, le jeu de données doit inclure différentes tailles et différents formats d’image ; pour un serveur multimédia, de courts fichiers utilisant différents codecs.

## Une mesure de référence sans montage

Avant que FUSE et le cache VFS n’entrent en jeu, vous devriez mesurer le backend directement. Copiez le jeu de données vers le remote de test avec Rclone et enregistrez un journal détaillé :

```bash
rclone copy ./testdata remote:cold-storage-test \
  --progress \
  --stats 5s \
  --log-file rclone-copy.log \
  --log-level INFO
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `./testdata` | Source de la copie : le jeu de données de test généré localement |
| `remote:cold-storage-test` | Destination : chemin dans le remote configuré |
| `--progress` | Affichage continu de la progression dans le terminal |
| `--stats 5s` | Statistiques de transfert toutes les 5 secondes au lieu de l’intervalle par défaut |
| `--log-file rclone-copy.log` | Journal complet dans un fichier pour une analyse ultérieure |
| `--log-level INFO` | Journalise les transferts, les nouvelles tentatives et les erreurs, sans le volume de DEBUG |

</details>

Vérifiez ensuite si la source et la destination correspondent :

```bash
rclone check ./testdata remote:cold-storage-test \
  --one-way \
  --download
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `./testdata` | Référence : le jeu de données original local |
| `remote:cold-storage-test` | Élément testé : le jeu de données fraîchement téléversé dans le remote |
| `--one-way` | Vérifie uniquement que tous les fichiers source existent dans la destination et sont identiques ; les fichiers supplémentaires dans la destination ne sont pas signalés |
| `--download` | Télécharge les données et compare leur contenu, au lieu de se fier aux hash |

</details>

`--download` est ici essentiel, car certains backends ne fournissent pas de hash adaptés. La comparaison est plus longue, mais elle fournit une base utile pour le test d’intégrité ultérieur.

Consignez le temps de téléversement, le débit de transfert, le nombre de nouvelles tentatives et les erreurs d’API. Si l’accès direct est déjà instable, le montage ne pourra pas le corriger.

## Séparer le montage de test du cache de production

Pour les mesures, utilisez un point de montage et un répertoire de cache distincts :

```bash
rclone mount remote:cold-storage-test /mnt/rclone-test \
  --vfs-cache-mode full \
  --cache-dir /var/cache/rclone-test \
  --vfs-cache-max-size 10G \
  --vfs-cache-poll-interval 1m \
  --allow-other \
  --log-file /var/log/rclone-test.log \
  --log-level INFO
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `remote:cold-storage-test` | Remote, chemin inclus, à monter |
| `/mnt/rclone-test` | Point de montage sur le système de test |
| `--vfs-cache-mode full` | Met intégralement en tampon les accès en lecture et en écriture dans le cache local |
| `--cache-dir /var/cache/rclone-test` | Répertoire de cache distinct, séparé du cache de production |
| `--vfs-cache-max-size 10G` | Limite souple de 10 GiB pour le cache VFS |
| `--vfs-cache-poll-interval 1m` | Vérification et nettoyage du cache chaque minute |
| `--allow-other` | Les utilisateurs autres que celui qui effectue le montage peuvent aussi accéder au montage ; nécessaire pour les services et les conteneurs |
| `--log-file /var/log/rclone-test.log` | Journal dans un fichier afin de suivre les accès durant les tests |
| `--log-level INFO` | Niveau de détail moyen du journal |

</details>

Ces valeurs constituent un exemple, et non une recommandation générale. La séparation est déterminante : un cache de test vide rend les lectures à froid reproductibles sans devoir supprimer des fichiers d’un cache de production actif.

`--vfs-cache-mode full` est généralement le mode de test le plus révélateur pour les applications. Rclone met alors en tampon localement les accès en lecture et en écriture, et peut mieux reproduire des accès aux fichiers qui ne seraient pas possibles avec un simple stockage objet. Cette compatibilité supplémentaire consomme de l’espace de stockage local.

## Toujours vérifier du point de vue du véritable service

Un montage peut fonctionner pour votre utilisateur et être malgré tout inutilisable pour le service. Les causes fréquentes sont un autre identifiant utilisateur, l’absence de `--allow-other`, les limites des conteneurs ou une propagation de montage incorrecte.

Effectuez donc au moins un accès complet en lecture avec la même identité que celle sous laquelle l’application s’exécutera ultérieurement :

```bash
sudo -u <service-user> sha256sum /mnt/rclone-test/pfad/zur/datei
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-u <service-user>` | Exécute la commande en tant qu’utilisateur indiqué, et non en tant que root |
| `/mnt/rclone-test/pfad/zur/datei` | Fichier à lire ; `sha256sum` force une lecture complète |

</details>

Si le service s’exécute dans Docker, le test doit être réalisé dans le conteneur :

```bash
docker exec --user <uid>:<gid> <app-container> \
  sha256sum /pfad/im/container/datei
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `--user <uid>:<gid>` | Exécute la commande dans le conteneur avec cet identifiant utilisateur et de groupe, indépendamment de l’utilisateur par défaut de l’image |
| `<app-container>` | Nom ou ID du conteneur d’application en cours d’exécution |
| `sha256sum /pfad/im/container/datei` | Commande à exécuter ; le chemin correspond au montage vu depuis le conteneur |

</details>

Mieux encore : effectuez un véritable test de l’application. Ouvrez le fichier via l’interface web ou l’API du service. C’est le seul moyen de constater si l’application lance par exemple plusieurs lectures parallèles, saute à la fin du fichier ou attend des métadonnées supplémentaires.

## Mesurer séparément les lectures à froid et à chaud

Avec `--vfs-cache-mode full`, trois couches se trouvent entre l’application et le cloud :

| Couche | Ce qui s’y trouve |
|---|---|
| Remote | le fichier complet dans le service cloud |
| Cache VFS | zones stockées localement de fichiers déjà lus |
| Cache de pages Linux | données récemment utilisées en RAM |

Pour une lecture à froid, choisissez un fichier dont le contenu n’a encore jamais été lu via le montage de test. Lors de la lecture à chaud effectuée juste après, il se trouve dans le cache VFS et le plus souvent aussi en RAM.

```bash
measure_read() {
  file="$1"
  label="$2"
  start=$(date +%s%3N)
  cat "$file" > /dev/null
  end=$(date +%s%3N)
  printf '%s: %s ms\n' "$label" "$((end - start))"
}

measure_read "/mnt/rclone-test/grosse-datei.bin" "Cold Read"
measure_read "/mnt/rclone-test/grosse-datei.bin" "Warm Read"
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `date +%s%3N` | Horodatage en millisecondes : secondes Unix immédiatement suivies des trois premiers chiffres de la partie nanoseconde (GNU date) |
| `cat "$file" > /dev/null` | Lit entièrement le fichier et ignore la sortie ; seule la durée de lecture est mesurée |
| `"$1"`, `"$2"` | Arguments de la fonction shell : chemin du fichier et libellé de la ligne de mesure |

</details>

Ne mesurez pas un seul fichier. Utilisez au moins dix fichiers encore jamais lus, de tailles différentes, et notez la médiane, la valeur la plus lente et la taille du fichier. Un seul meilleur résultat ne constitue pas une base de décision.

Une lecture à chaud n’est pas un simple test de disque, car le noyau peut conserver des parties du fichier en RAM. Pour la plupart des scénarios de stockage à froid, ce n’est pas un problème. L’important est ce qu’un utilisateur constate lors de la première ouverture et lors des ouvertures répétées. Si vous souhaitez évaluer séparément la RAM et le disque local, vous devez également contrôler et vider de façon vérifiable le cache de pages.

## Ne pas tester uniquement les lectures complètes

`cat` lit un fichier du début à la fin. De nombreuses applications se comportent différemment :

- Un lecteur vidéo lit d’abord l’en-tête et l’index, saute ensuite à une autre position, puis poursuit la lecture séquentielle.
- Une gestion d’images lit les métadonnées puis génère une vignette.
- Un programme d’archivage peut commencer par lire la fin du fichier.
- Plusieurs workers peuvent accéder simultanément à différents fichiers.

Testez ces déroulements avec l’application réelle. Observez en parallèle le journal Rclone et le cache. Pour les gros fichiers, il est intéressant de voir combien Rclone stocke réellement en local et si `--vfs-read-ahead` correspond au modèle d’accès.

Un montage Rclone n’est par ailleurs pas un emplacement de stockage pertinent pour des bases de données ou d’autres fichiers nécessitant un verrouillage fiable et des modifications fréquentes au sein du même fichier. La couche VFS compense les différences entre système de fichiers et stockage objet, mais elle ne transforme pas le backend en système de fichiers local.

## Valider séparément le chemin d’écriture

Si votre service lit uniquement, montez si possible le remote en lecture seule. S’il doit écrire, testez séparément la création, l’écrasement, le renommage et la suppression.

Un fichier écrit n’apparaît pas nécessairement immédiatement dans le remote. Lorsque le cache VFS est activé, le téléversement ne commence qu’après la fermeture du fichier et l’expiration de `--vfs-write-back`. Vérifiez donc les deux états :

1. L’application a fermé le fichier avec succès.
2. Le fichier est ensuite lisible dans le remote via un accès Rclone direct.

```bash
printf 'writeback-test\n' > /mnt/rclone-test/writeback-test.txt

# Après l’expiration de --vfs-write-back :
rclone cat remote:cold-storage-test/writeback-test.txt
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `/mnt/rclone-test/writeback-test.txt` | Fichier cible dans le montage ; la redirection écrit via le cache VFS |
| `remote:cold-storage-test/writeback-test.txt` | Accès direct sans passer par le montage : `rclone cat` lit le fichier depuis le remote et l’écrit sur stdout |

</details>

Répétez le test avec un gros fichier et arrêtez Rclone pendant que le téléversement est encore en cours. Redémarrez ensuite avec le même répertoire de cache et vérifiez si le téléversement reprend. C’est précisément cette fenêtre de temps qui détermine la quantité de données menacées lors d’une panne de serveur.

Testez également le renommage et la suppression. De nombreux backends cloud représentent ces opérations différemment d’un système de fichiers local. L’important n’est pas seulement que la commande se termine avec succès, mais aussi à quel moment la modification devient visible via un accès direct au remote et pour les autres clients.

## Tester les modifications en dehors du montage

Les fichiers peuvent être modifiés via l’interface web du fournisseur, un deuxième processus Rclone ou un autre serveur. Le montage ne voit pas toujours ces modifications immédiatement, car les informations de répertoire sont mises en cache.

Créez donc un fichier directement dans le remote avec un deuxième appel Rclone :

```bash
printf 'external-change\n' > external-change.txt
rclone copyto external-change.txt \
  remote:cold-storage-test/external-change.txt
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `external-change.txt` | Source : le fichier créé localement |
| `remote:cold-storage-test/external-change.txt` | Destination avec le nom de fichier exact ; `copyto` copie un fichier unique sous exactement ce nom, plutôt que, comme `copy`, dans un répertoire |

</details>

Mesurez à quel moment le fichier apparaît dans le montage. Répétez le test pour une modification et une suppression. Le résultat dépend du backend, de sa prise en charge du polling ainsi que de `--poll-interval` et `--dir-cache-time`. Si l’application doit voir immédiatement les modifications actuelles, ce comportement doit faire explicitement partie des critères de validation.

Si l’interface de contrôle à distance est activée, vous pouvez invalider de manière ciblée le cache de répertoires :

```bash
rclone rc vfs/forget
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `vfs/forget` | Commande de contrôle à distance à exécuter : invalide l’arborescence de répertoires mise en cache par le VFS, afin que le prochain accès interroge à nouveau le remote |

</details>

C’est utile pour un test manuel, mais cela ne remplace pas une stratégie d’exploitation adaptée.

## Mettre le cache sous pression

Un cache presque vide est le cas le plus simple. Lors d’une deuxième série de tests, réduisez volontairement `--vfs-cache-max-size` et lisez plus de données qu’il ne peut en contenir.

```bash
du -sh /var/cache/rclone-test/vfs
du -sh --apparent-size /var/cache/rclone-test/vfs
find /var/cache/rclone-test/vfs -type f | wc -l
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `du -s` | Résume l’utilisation de l’espace sur une ligne, au lieu de lister chaque sous-répertoire |
| `du -h` | Affichage en unités lisibles par l’humain (K, M, G) |
| `du --apparent-size` | Affiche la taille logique du fichier plutôt que l’espace disque réellement occupé |
| `find … -type f` | Ne prend en compte que les fichiers ordinaires, pas les répertoires |
| `wc -l` | Compte les lignes de la sortie, donc ici le nombre de fichiers dans le cache |

</details>

Les deux tailles peuvent fortement différer. En mode Full, Rclone utilise des fichiers creux : un fichier affiche sa taille logique complète, bien que seules les zones lues occupent de l’espace local.

La limite du cache est en outre souple. Rclone la vérifie selon l’intervalle de `--vfs-cache-poll-interval`, et les fichiers ouverts ne peuvent pas être supprimés. Le cache peut donc temporairement dépasser la limite. Il devrait toutefois diminuer à nouveau après la fermeture des fichiers et le prochain cycle de nettoyage.

Consignez le pic, la valeur après le nettoyage et le temps nécessaire. Vous pourrez ainsi dimensionner raisonnablement l’espace de stockage local requis.

## Simuler deux pannes différentes

Un cloud inaccessible et un processus Rclone qui plante sont deux erreurs distinctes :

| Panne | Ce que vous vérifiez |
|---|---|
| Backend ou réseau inaccessible, Rclone continue de fonctionner | Comportement lors des nouvelles tentatives, des délais d’attente et pour les fichiers déjà mis en cache |
| Processus Rclone arrêté | Comportement du montage FUSE et restauration du point de montage |

Simulez les deux uniquement dans l’environnement de test. Vous pouvez arrêter brutalement un conteneur Rclone pour le deuxième cas :

```bash
docker kill --signal KILL <rclone-container>
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `--signal KILL` | Envoie SIGKILL au lieu du signal standard SIGTERM ; le processus n’a aucune possibilité de nettoyage |
| `<rclone-container>` | Nom ou ID du conteneur Rclone |

</details>

Pendant la panne, vérifiez l’application, et pas seulement le point de montage :

- Quelles fonctions restent disponibles ?
- Combien de temps un accès attend-il avant qu’une erreur apparaisse ?
- Les fichiers déjà entièrement mis en cache restent-ils accessibles ?
- L’application arrête-t-elle les nouvelles écritures ?
- Un message d’erreur compréhensible apparaît-il, ou seulement un processus bloqué ?
- Votre supervision se déclenche-t-elle ?

Un service d’écriture ne doit pas écrire sans que cela soit détecté dans le répertoire local sous-jacent lorsqu’il manque le montage. Après le retour du montage, ces fichiers seraient masqués. Une protection simple avant chaque tâche d’écriture est :

```bash
mountpoint -q /mnt/rclone-test || exit 1
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-q` | Aucune sortie ; le résultat figure uniquement dans le code de sortie |
| `/mnt/rclone-test` | Chemin à vérifier ; code de sortie 0 uniquement si un montage est réellement actif à cet emplacement |
| `\|\| exit 1` | Interrompt le script si le chemin n’est pas un point de montage |

</details>

Après le redémarrage de Rclone, vérifiez le montage sur l’hôte et depuis chaque conteneur consommateur. Un montage recréé n’atteint un conteneur déjà en cours d’exécution qu’avec une propagation de montage adaptée. Pour Docker, `rslave` est généralement nécessaire du côté consommateur. Les détails figurent dans l’article [Exploiter de manière fiable les montages Rclone dans Docker](/blog/rclone-mount-in-docker-container).

## Un exemple concret avec Paperless-ngx

Pour mon test Paperless, j’ai généré 40 PDF totalisant 13,9 MB. Un document jamais ouvert auparavant nécessitait environ 1,8 seconde, tandis qu’un accès répété immédiatement prenait 19 à 24 millisecondes. Un cache VFS limité à 4 MB est brièvement monté à 12,7 MiB, puis a été nettoyé lors du cycle suivant.

Pendant que le remote était inaccessible, la liste de documents, la recherche plein texte et les aperçus continuaient de fonctionner, car ces données étaient stockées localement. Seul l’original ne pouvait pas être ouvert. Après la recréation du montage, le conteneur Paperless en cours d’exécution a de nouveau pu accéder aux fichiers sans devoir être redémarré.

Ces chiffres ne constituent pas un benchmark pour Rclone ni pour Proton Drive. Le comportement est intéressant : le stockage à chaud restait disponible localement, les lectures à froid étaient plus lentes mais prévisibles, et le service récupérait après la panne.

## Ce qui doit figurer dans le protocole de test

Un résultat vérifiable ultérieurement contient au minimum :

- version de Rclone et backend utilisé
- système d’exploitation, variante FUSE et système de fichiers du répertoire de cache
- commande de montage complète sans données d’accès
- nombre, répartition des tailles et structure des fichiers de test
- valeurs de lecture à froid et à chaud pour plusieurs fichiers
- durée d’écriture jusqu’à la visibilité dans le remote
- pic du cache et durée du nettoyage
- résultat de `rclone check --download`
- comportement en cas de panne du backend et d’arrêt du processus Rclone
- temps de récupération du point de vue de l’application
- nouvelles tentatives, délais d’attente, limitations et erreurs d’authentification du journal

Définissez à l’avance une valeur limite pour chaque point. Le test se conclura alors par une décision, et non par une simple collection de chiffres intéressants.

## Quand l’architecture est prête

Un montage de stockage à froid est prêt à être utilisé si vous pouvez répondre oui à ces questions :

- Les lectures à froid sont-elles suffisamment rapides pour le service prévu ?
- Le cache accélère-t-il les accès répétés comme prévu ?
- L’espace local requis reste-t-il maîtrisable, même sous charge ?
- Tous les fichiers correspondent-ils après un téléchargement complet ?
- Toutes les opérations sur fichiers nécessaires fonctionnent-elles avec le backend choisi ?
- L’application se comporte-t-elle de manière maîtrisée lors d’une panne du cloud ?
- Les écritures sont-elles arrêtées en toute sécurité en l’absence de montage ?
- Un montage recréé atteint-il tous les consommateurs en cours d’exécution ?
- La supervision signale-t-elle la panne avant qu’un utilisateur ne la rapporte ?

S’il manque une réponse, vous savez au moins précisément sur quoi vous devez encore travailler. C’est bien plus utile qu’un montage qui semblait fonctionner lors du premier `ls` et qui ne révèle ses limites qu’en exploitation.

## Sources

1.  [Rclone test makefiles](https://rclone.org/commands/rclone_test_makefiles/) : fichiers de test et structures de répertoires reproductibles avec des tailles configurables.

2.  [Rclone mount](https://rclone.org/commands/rclone_mount/) : modes de cache VFS, écriture différée, fichiers creux, limites du cache et cache de répertoires.

3.  [Rclone check](https://rclone.org/commands/rclone_check/) : comparaison entre source et destination, y compris la vérification complète avec `--download`.

4.  [Rclone Remote Control](https://rclone.org/rc/) : invalidation ciblée du cache de répertoires VFS avec `vfs/forget`.

5.  [Rclone Global Flags](https://rclone.org/flags/) : référence complète des options globales, dont la journalisation, les statistiques et les paramètres VFS.
