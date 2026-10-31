---
title: "Tester le Cold Storage avec Rclone : un plan de test pratique"
navTitle: "Tester Rclone"
description: "Avant qu’un service ne lise ses fichiers depuis le cloud via un montage Rclone, vous devriez vérifier davantage que l’accès aux répertoires. Ce plan de test couvre les lectures à froid, les lectures à chaud, les opérations d’écriture, le comportement du cache, l’intégrité des fichiers et les pannes."
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
translatedAt: 2026-10-31T03:25:57.679Z
translationReview: required
translationSourceHash: 820965b014837c0d0b5b5bf629a02842f2ebbbb26f62d343d8f1630278cd3bec
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/tester-le-stockage-a-froid-avec-rclone-un-plan-de-test-pratique
---

Un montage Rclone se met rapidement en place. Le remote apparaît comme un répertoire, `ls` affiche des fichiers et le premier test fonctionnel est réussi. Cela ne dit toutefois pas grand-chose sur l’exploitation en production.

Dès qu’un service accède au montage, d’autres questions se posent : combien de temps prend le premier accès à un fichier ? Quels accès sont servis par le cache local ? Que se passe-t-il avec un fichier qui n’a pas encore été téléversé si Rclone plante ? Un conteneur en cours d’exécution voit-il de nouveau le montage reconstruit ? Et comment le service réagit-il si le cloud est temporairement inaccessible ?

Cet article fournit un plan de test général à cet effet. Vous pouvez l’utiliser pour une archive de documents, un serveur multimédia, une gestion de photos ou tout autre service qui récupère des fichiers rarement nécessaires via Rclone depuis un Cold Storage.

## Les principales options de rclone

Pour vous orienter, voici les options Rclone utilisées dans ce plan de test, traduites librement depuis la documentation :

<details class="options-details">
<summary>Aperçu des options</summary>

| Option | Signification |
|---|---|
| `--seed n` | Valeur initiale du générateur aléatoire avec `rclone test makefiles` ; une même seed produit la même arborescence de fichiers |
| `--files n` | Nombre de fichiers de test à générer |
| `--files-per-directory n` | Nombre moyen de fichiers par répertoire |
| `--min-file-size grösse` | Taille minimale des fichiers générés (suffixes tels que K, M, G autorisés) |
| `--max-file-size grösse` | Taille maximale des fichiers générés |
| `--progress` | Affichage de progression pendant le transfert |
| `--stats dauer` | Intervalle d’affichage des statistiques de transfert |
| `--log-file datei` | Écrit le journal dans le fichier indiqué |
| `--log-level stufe` | Niveau de détail du journal : DEBUG, INFO, NOTICE ou ERROR |
| `--one-way` | Avec `rclone check`, vérifie uniquement que tous les fichiers source existent dans la destination et sont identiques ; les fichiers supplémentaires dans la destination ne sont pas considérés comme une erreur |
| `--download` | Télécharge effectivement les données lors de la comparaison, au lieu de comparer uniquement les hashes |
| `--vfs-cache-mode modus` | Stratégie de cache de la couche VFS ; `full` met en cache localement les accès en lecture et en écriture |
| `--cache-dir verzeichnis` | Répertoire du cache local |
| `--vfs-cache-max-size grösse` | Limite souple pour la taille totale du cache VFS |
| `--vfs-cache-poll-interval dauer` | Intervalle auquel Rclone vérifie et nettoie le cache |
| `--vfs-write-back dauer` | Délai après la fermeture d’un fichier avant le début du téléversement vers le remote |
| `--vfs-read-ahead grösse` | Quantité supplémentaire de données lues à l’avance au-delà de la position demandée avec `full` |
| `--poll-interval dauer` | Intervalle auquel Rclone interroge le remote pour détecter des modifications (uniquement pour les backends prenant en charge le polling) |
| `--dir-cache-time dauer` | Durée de validité des listes de répertoires mises en cache |
| `--allow-other` | Autorise les utilisateurs autres que celui ayant monté le système à accéder au montage FUSE |

</details>

Les listes complètes figurent dans la documentation Rclone, notamment sous [rclone mount](https://rclone.org/commands/rclone_mount/) et dans l’aperçu des [flags globaux](https://rclone.org/flags/).

## Définir d’abord ce que vous souhaitez atteindre

Le Cold Storage ne signifie pas automatiquement la même chose pour chaque application. Un serveur multimédia lit généralement de gros fichiers de manière séquentielle. Une gestion de photos charge de nombreuses petites miniatures et saute à différents endroits. Une archive de documents ouvre des fichiers relativement petits, mais souvent une seule fois.

Avant le test, notez les caractéristiques principales de votre véritable corpus :

- taille de fichier typique et plus grand fichier présent
- nombre de fichiers par répertoire
- lecture complète ou accès aléatoires à certaines zones
- rapport entre les accès en lecture et en écriture
- nombre d’utilisateurs ou de processus simultanés
- modifications intervenant directement dans le remote en dehors du montage
- délai acceptable pour une lecture à froid
- espace maximal disponible pour le cache local

Ce n’est qu’à partir de là que des critères de réussite pertinents émergent. Ouvrir un seul fichier en 1,2 seconde peut être parfaitement acceptable pour une archive et inutilisable pour une application interactive.

## Générer un corpus de test reproductible

Rclone fournit déjà un outil adapté à cet effet. `rclone test makefiles` génère toujours la même arborescence de fichiers avec une seed fixe :

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
| `--seed 42` | Valeur initiale fixe du générateur aléatoire ; chaque exécution génère le même corpus |
| `--files 250` | 250 fichiers de test au total |
| `--files-per-directory 25` | En moyenne 25 fichiers par répertoire |
| `--min-file-size 16K` | Plus petit fichier : 16 KiB |
| `--max-file-size 32M` | Plus grand fichier : 32 MiB |

</details>

Adaptez le nombre et les tailles à votre véritable corpus. Ne testez pas uniquement des fichiers moyens. Quelques très petits fichiers montrent le coût des accès aux métadonnées ; quelques gros fichiers rendent visibles le débit, la lecture anticipée et le comportement du cache.

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
| `printf '…\n' > datei` | Écrit le texte indiqué comme contenu du fichier ; la redirection crée le fichier au nom problématique |

</details>

Le dernier test est particulièrement important si le système de fichiers local et le backend cloud traitent différemment les majuscules et les minuscules.

Si votre service n’accepte que certains formats, des fichiers binaires quelconques ne suffisent pas. Générez alors en complément des fichiers synthétiques exactement dans ces formats. Avec Paperless-ngx, il s’agissait de PDF avec une véritable couche de texte, afin que le test ne mesure pas par inadvertance les performances OCR plutôt que le chemin de stockage. Pour une gestion de photos, le corpus doit comporter différentes tailles et différents formats d’image ; pour un serveur multimédia, des fichiers courts avec différents codecs.

## Une mesure de référence sans montage

Avant que FUSE et le cache VFS n’entrent en jeu, vous devriez mesurer directement le backend. Copiez le corpus vers le remote de test avec Rclone et enregistrez un journal détaillé :

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
| `./testdata` | Source de la copie : le corpus de test généré localement |
| `remote:cold-storage-test` | Destination : chemin dans le remote configuré |
| `--progress` | Affichage de progression dans le terminal |
| `--stats 5s` | Statistiques de transfert toutes les 5 secondes au lieu de l’intervalle par défaut |
| `--log-file rclone-copy.log` | Journal complet dans un fichier pour une analyse ultérieure |
| `--log-level INFO` | Journalise les transferts, les tentatives et les erreurs, sans le volume de DEBUG |

</details>

Vérifiez ensuite que la source et la destination correspondent :

```bash
rclone check ./testdata remote:cold-storage-test \
  --one-way \
  --download
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `./testdata` | Référence : le corpus original local |
| `remote:cold-storage-test` | Élément contrôlé : le corpus fraîchement téléversé dans le remote |
| `--one-way` | Vérifie uniquement que tous les fichiers source existent dans la destination et sont identiques ; les fichiers supplémentaires dans la destination ne sont pas signalés |
| `--download` | Télécharge les données et compare leur contenu, au lieu de se fier aux hashes |

</details>

`--download` est ici déterminant, car certains backends ne fournissent pas de hashes adaptés. La comparaison dure plus longtemps, mais fournit une base exploitable pour le test d’intégrité ultérieur.

Notez le temps de téléversement, le débit de transfert, le nombre de tentatives et les erreurs d’API. Si l’accès direct est déjà instable, le montage ne peut pas le réparer.

## Séparer le montage de test du cache de production

Utilisez un point de montage et un répertoire de cache distincts pour les mesures :

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
| `remote:cold-storage-test` | Remote et chemin à monter |
| `/mnt/rclone-test` | Point de montage sur le système de test |
| `--vfs-cache-mode full` | Met entièrement en cache les accès en lecture et en écriture dans le cache local |
| `--cache-dir /var/cache/rclone-test` | Répertoire de cache dédié, séparé du cache de production |
| `--vfs-cache-max-size 10G` | Limite souple de 10 GiB pour le cache VFS |
| `--vfs-cache-poll-interval 1m` | Vérification et nettoyage du cache toutes les minutes |
| `--allow-other` | Les utilisateurs autres que celui ayant monté le système peuvent également y accéder ; nécessaire pour les services et conteneurs |
| `--log-file /var/log/rclone-test.log` | Journal dans un fichier pour suivre les accès pendant les tests |
| `--log-level INFO` | Niveau de détail intermédiaire du journal |

</details>

Les valeurs sont un exemple et non une recommandation générale. La séparation est déterminante : un cache de test vide rend les lectures à froid reproductibles, sans devoir supprimer des fichiers d’un cache de production actif.

`--vfs-cache-mode full` est généralement le mode de test le plus révélateur pour les applications. Rclone met alors les accès en lecture et en écriture en cache localement et peut mieux reproduire les accès aux fichiers impossibles avec un simple stockage objet. Cette compatibilité supplémentaire consomme de l’espace de stockage local.

## Toujours vérifier du point de vue du véritable service

Un montage peut fonctionner pour votre utilisateur tout en restant inutilisable pour le service. Les causes fréquentes sont un autre ID utilisateur, l’absence de `--allow-other`, les limites des conteneurs ou une propagation de montage incorrecte.

Exécutez donc au moins un accès complet en lecture avec la même identité que celle sous laquelle l’application s’exécutera ensuite :

```bash
sudo -u <service-user> sha256sum /mnt/rclone-test/pfad/zur/datei
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-u <service-user>` | Exécute la commande en tant que l’utilisateur indiqué, et non root |
| `/mnt/rclone-test/pfad/zur/datei` | Fichier à lire ; `sha256sum` force une lecture complète |

</details>

Si le service s’exécute dans Docker, le test doit être effectué dans le conteneur :

```bash
docker exec --user <uid>:<gid> <app-container> \
  sha256sum /pfad/im/container/datei
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `--user <uid>:<gid>` | Exécute la commande dans le conteneur avec cet ID utilisateur et de groupe, indépendamment de l’utilisateur par défaut de l’image |
| `<app-container>` | Nom ou ID du conteneur d’application en cours d’exécution |
| `sha256sum /pfad/im/container/datei` | Commande à exécuter ; le chemin est le montage vu depuis le conteneur |

</details>

Mieux encore : effectuez un véritable test de l’application. Ouvrez le fichier via l’interface web ou l’API du service. C’est le seul moyen de constater si l’application lance, par exemple, plusieurs lectures parallèles, saute à la fin du fichier ou attend des métadonnées supplémentaires.

## Mesurer séparément les lectures à froid et à chaud

Avec `--vfs-cache-mode full`, trois couches se situent entre l’application et le cloud :

| Couche | Ce qui s’y trouve |
|---|---|
| Remote | le fichier complet dans le service cloud |
| Cache VFS | zones stockées localement de fichiers déjà lus |
| Cache de pages Linux | données récemment utilisées en RAM |

Pour une lecture à froid, choisissez un fichier dont le contenu n’a encore jamais été lu via le montage de test. Lors de la lecture à chaud qui suit directement, il se trouve dans le cache VFS et généralement aussi en RAM.

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
| `date +%s%3N` | Horodatage en millisecondes : secondes Unix suivies directement des trois premiers chiffres de la fraction en nanosecondes (GNU date) |
| `cat "$file" > /dev/null` | Lit entièrement le fichier et ignore la sortie ; seul le temps de lecture est mesuré |
| `"$1"`, `"$2"` | Arguments de la fonction shell : chemin du fichier et libellé de la ligne de mesure |

</details>

Ne mesurez pas un seul fichier. Utilisez au moins dix fichiers encore non lus de tailles différentes et notez la médiane, la valeur la plus lente et la taille du fichier. Une seule meilleure valeur ne constitue pas une base de décision.

Une lecture à chaud n’est pas un simple test de disque, car le noyau peut conserver des parties du fichier en RAM. Pour la plupart des scénarios de Cold Storage, ce n’est pas un problème. L’essentiel est ce qu’un utilisateur constate lors de la première ouverture et des ouvertures répétées. Si vous souhaitez évaluer séparément la RAM et le disque local, vous devez également contrôler et vider de manière vérifiable le cache de pages.

## Ne pas tester uniquement les lectures complètes

`cat` lit un fichier du début à la fin. Beaucoup d’applications se comportent différemment :

- Un lecteur vidéo lit d’abord l’en-tête et l’index, saute ensuite à une autre position puis poursuit la lecture séquentielle.
- Une gestion d’images lit les métadonnées puis génère une miniature.
- Un programme d’archivage peut commencer par lire la fin du fichier.
- Plusieurs workers peuvent accéder simultanément à différents fichiers.

Testez ces déroulements avec l’application réelle. Observez parallèlement le journal Rclone et le cache. Avec les gros fichiers, il est intéressant de voir quelle quantité Rclone stocke réellement localement et si `--vfs-read-ahead` correspond au schéma d’accès.

Un montage Rclone n’est par ailleurs pas un emplacement de stockage pertinent pour des bases de données ou d’autres fichiers nécessitant un verrouillage fiable et des modifications fréquentes dans le même fichier. La couche VFS compense les différences entre système de fichiers et stockage objet, mais ne transforme pas le backend en système de fichiers local.

## Valider séparément le chemin d’écriture

Si votre service ne fait que lire, montez le remote en lecture seule si possible. S’il doit écrire, testez séparément la création, l’écrasement, le renommage et la suppression.

Un fichier écrit n’apparaît pas nécessairement immédiatement dans le remote. Avec un cache VFS actif, le téléversement ne commence qu’après la fermeture du fichier et l’expiration de `--vfs-write-back`. Vérifiez donc les deux états :

1. L’application a fermé le fichier avec succès.
2. Le fichier est ensuite lisible dans le remote par un accès direct Rclone.

```bash
printf 'writeback-test\n' > /mnt/rclone-test/writeback-test.txt

# Après expiration de --vfs-write-back :
rclone cat remote:cold-storage-test/writeback-test.txt
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `/mnt/rclone-test/writeback-test.txt` | Fichier cible dans le montage ; la redirection écrit via le cache VFS |
| `remote:cold-storage-test/writeback-test.txt` | Accès direct sans passer par le montage : `rclone cat` lit le fichier depuis le remote et l’envoie sur stdout |

</details>

Répétez le test avec un gros fichier et arrêtez Rclone pendant que le téléversement est encore en cours. Redémarrez ensuite avec le même répertoire de cache et vérifiez si le téléversement reprend. Cette fenêtre temporelle détermine précisément combien de données sont menacées en cas de panne du serveur.

Testez également le renommage et la suppression. Beaucoup de backends cloud représentent ces opérations différemment d’un système de fichiers local. Ce qui importe n’est pas seulement que la commande se termine avec succès, mais aussi le moment où la modification devient visible via un accès direct au remote et pour les autres clients.

## Tester les modifications hors du montage

Les fichiers peuvent être modifiés via l’interface web du fournisseur, un second processus Rclone ou un autre serveur. Le montage ne voit pas toujours ces modifications immédiatement, car les informations de répertoires sont mises en cache.

Créez donc un fichier directement dans le remote avec un second appel Rclone :

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
| `remote:cold-storage-test/external-change.txt` | Destination avec le nom de fichier exact ; `copyto` copie un seul fichier sous ce nom précis, au lieu de copier comme `copy` dans un répertoire |

</details>

Mesurez le moment où le fichier apparaît dans le montage. Répétez le test pour une modification et une suppression. Le résultat dépend du backend, de sa prise en charge du polling ainsi que de `--poll-interval` et `--dir-cache-time`. Si l’application doit voir immédiatement les modifications récentes, ce comportement doit explicitement faire partie des critères de validation.

Lorsque l’interface Remote Control est activée, vous pouvez invalider de manière ciblée le cache de répertoires :

```bash
rclone rc vfs/forget
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `vfs/forget` | Commande Remote Control à exécuter : invalide l’arborescence de répertoires mise en cache par le VFS afin que l’accès suivant interroge de nouveau le remote |

</details>

C’est utile pour un test manuel, mais ne remplace pas une stratégie d’exploitation adaptée.

## Mettre le cache sous pression

Un cache presque vide est le cas le plus simple. Dans un second cycle de test, définissez délibérément `--vfs-cache-max-size` à une faible valeur et lisez plus de données qu’il ne peut en contenir.

```bash
du -sh /var/cache/rclone-test/vfs
du -sh --apparent-size /var/cache/rclone-test/vfs
find /var/cache/rclone-test/vfs -type f | wc -l
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `du -s` | Résume l’espace utilisé sur une seule ligne au lieu de lister chaque sous-répertoire |
| `du -h` | Affichage en unités lisibles (K, M, G) |
| `du --apparent-size` | Affiche la taille logique du fichier au lieu de l’espace de stockage réellement occupé |
| `find … -type f` | Ne prend en compte que les fichiers ordinaires, pas les répertoires |
| `wc -l` | Compte les lignes de sortie, donc ici le nombre de fichiers du cache |

</details>

Les deux tailles peuvent fortement diverger. En mode Full, Rclone utilise des fichiers sparse : un fichier affiche sa taille logique complète alors que seules les zones lues occupent de l’espace local.

La limite du cache est en outre souple. Rclone la vérifie au rythme de `--vfs-cache-poll-interval`, et les fichiers ouverts ne peuvent pas être supprimés. Le cache peut donc dépasser temporairement la limite. Il devrait toutefois diminuer à nouveau après la fermeture des fichiers et le prochain cycle de nettoyage.

Consignez le pic, la valeur après nettoyage et le temps nécessaire. Vous pourrez ainsi dimensionner raisonnablement l’espace de stockage local requis.

## Simuler deux pannes distinctes

Un cloud inaccessible et un processus Rclone planté sont deux erreurs distinctes :

| Panne | Ce que vous vérifiez |
|---|---|
| Backend ou réseau inaccessible, Rclone continue de fonctionner | Comportement lors des tentatives, des timeouts et avec les fichiers déjà mis en cache |
| Processus Rclone arrêté | Comportement du montage FUSE et rétablissement du point de montage |

Simulez ces deux cas uniquement dans l’environnement de test. Pour le second cas, vous pouvez arrêter brutalement un conteneur Rclone :

```bash
docker kill --signal KILL <rclone-container>
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `--signal KILL` | Envoie SIGKILL (déjà le signal par défaut avec `docker kill`, indiqué ici à titre de clarification) ; le processus n’a aucune possibilité de nettoyer |
| `<rclone-container>` | Nom ou ID du conteneur Rclone |

</details>

Pendant la panne, vérifiez l’application et pas seulement le point de montage :

- Quelles fonctions restent disponibles ?
- Combien de temps un accès attend-il avant qu’une erreur apparaisse ?
- Les fichiers déjà entièrement mis en cache restent-ils accessibles ?
- L’application arrête-t-elle les nouvelles opérations d’écriture ?
- Un message d’erreur compréhensible apparaît-il ou seulement un processus bloqué ?
- Votre monitoring se déclenche-t-il ?

Un service d’écriture ne doit pas écrire sans le savoir dans le répertoire local sous-jacent lorsqu’un montage est absent. Après le retour du montage, ces fichiers seraient masqués. Une protection simple avant chaque tâche d’écriture est :

```bash
mountpoint -q /mnt/rclone-test || exit 1
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-q` | Aucune sortie ; le résultat n’est présent que dans le code de sortie |
| `/mnt/rclone-test` | Chemin à vérifier ; code de sortie 0 uniquement si un montage y est effectivement actif |
| `\|\| exit 1` | Arrête le script si le chemin n’est pas un point de montage |

</details>

Après le redémarrage de Rclone, vérifiez le montage sur l’hôte et depuis chaque conteneur consommateur. Un montage reconstruit n’atteint un conteneur déjà en cours d’exécution qu’avec une propagation de montage adaptée. Pour Docker, `rslave` est généralement nécessaire du côté consommateur. Les détails figurent dans l’article [Exploiter de manière fiable les montages Rclone dans Docker](/blog/rclone-mount-in-docker-container).

## Un exemple concret avec Paperless-ngx

Pour mon test Paperless, j’ai généré 40 PDF totalisant 13,9 MB. Un document encore jamais ouvert nécessitait environ 1,8 seconde, tandis qu’un accès répété directement prenait 19 à 24 millisecondes. Un cache VFS limité à 4 MB est brièvement monté à 12,7 MiB, puis a été nettoyé lors du cycle suivant.

Pendant que le remote était inaccessible, la liste des documents, la recherche plein texte et les miniatures continuaient de fonctionner, car ces données étaient locales. Seul l’original ne pouvait pas être ouvert. Après la reconstruction du montage, le conteneur Paperless en cours d’exécution pouvait de nouveau accéder aux fichiers sans devoir être redémarré lui-même.

Ces chiffres ne constituent pas un benchmark pour Rclone ou Proton Drive. Ce qui est intéressant est le comportement : le Hot Storage est resté disponible localement, les lectures à froid étaient plus lentes mais prévisibles, et le service s’est remis de la panne.

## Ce que doit contenir le protocole de test

Un résultat traçable ultérieurement contient au minimum :

- version de Rclone et backend utilisé
- système d’exploitation, variante FUSE et système de fichiers du répertoire de cache
- commande de montage complète sans données d’accès
- nombre, distribution des tailles et structure des fichiers de test
- valeurs de lecture à froid et à chaud pour plusieurs fichiers
- durée d’écriture jusqu’à la visibilité dans le remote
- pic du cache et durée du nettoyage
- résultat de `rclone check --download`
- comportement en cas de panne du backend et d’arrêt du processus Rclone
- temps de rétablissement du point de vue de l’application
- tentatives, timeouts, limitations et erreurs d’authentification du journal

Définissez à l’avance une valeur limite pour chaque point. Le test se termine alors par une décision et non par une simple collection de chiffres intéressants.

## Quand la configuration est prête

Un montage de Cold Storage est prêt à être utilisé lorsque vous pouvez répondre oui à ces questions :

- Les lectures à froid sont-elles suffisamment rapides pour le service prévu ?
- Le cache accélère-t-il les accès répétés comme prévu ?
- Les besoins en espace local restent-ils maîtrisables même sous charge ?
- Tous les fichiers correspondent-ils après un téléchargement complet ?
- Toutes les opérations nécessaires sur les fichiers fonctionnent-elles avec le backend choisi ?
- L’application se comporte-t-elle de manière contrôlée lors d’une panne du cloud ?
- Les opérations d’écriture sont-elles stoppées de manière sûre lorsqu’un montage est absent ?
- Un montage reconstruit atteint-il tous les consommateurs en cours d’exécution ?
- Le monitoring indique-t-il la panne avant qu’un utilisateur ne la signale ?

Si une réponse manque, vous savez au moins exactement sur quoi vous devez encore travailler. C’est bien plus utile qu’un montage qui semblait correct lors du premier `ls` et ne révèle ses limites qu’en exploitation.

## Historique des modifications

- **25 octobre 2026 :** Correction : avec docker kill, SIGKILL est déjà le signal par défaut ; l’option --signal KILL sert uniquement à le préciser.

## Sources

1.  [Rclone test makefiles](https://rclone.org/commands/rclone_test_makefiles/) : fichiers de test et structures de répertoires reproductibles avec des tailles configurables.

2.  [Rclone mount](https://rclone.org/commands/rclone_mount/) : modes de cache VFS, writeback, fichiers sparse, limites de cache et cache de répertoires.

3.  [Rclone check](https://rclone.org/commands/rclone_check/) : comparaison de la source et de la destination, y compris la vérification complète avec `--download`.

4.  [Rclone Remote Control](https://rclone.org/rc/) : invalidation ciblée du cache de répertoires VFS avec `vfs/forget`.

5.  [Rclone Global Flags](https://rclone.org/flags/) : référence complète des options globales, notamment la journalisation, les statistiques et les paramètres VFS.
