---
title: "Tester le stockage à froid avec Rclone : un plan de test pratique"
navTitle: "Tester Rclone"
description: "Avant qu’un service ne lise ses fichiers depuis le cloud via un montage Rclone, vérifiez davantage que l’accès aux répertoires. Ce plan de test couvre les lectures à froid, les lectures à chaud, les opérations d’écriture, le comportement du cache, l’intégrité des fichiers et les pannes."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "11 min de lecture"
themen:
  - "rclone"
related:
  - "rclone-mount-in-docker-container"
  - "paperless-dokumente-clouddienst-auslagern"
slug: "tester-le-stockage-a-froid-avec-rclone-un-plan-de-test-pratique"
translationOf: "cloud-mount-testen-dummy-pdfs"
url: "https://rafaelpfister.ch/fr/blog/tester-le-stockage-a-froid-avec-rclone-un-plan-de-test-pratique"
---

Un montage Rclone se configure rapidement. Le remote apparaît comme un répertoire, `ls` affiche les fichiers et le premier test fonctionnel est réussi. Cela ne dit encore pas grand-chose sur l’exploitation en production.

Dès qu’un service accède au montage, d’autres questions se posent : combien de temps prend le premier accès à un fichier ? Quels accès sont servis par le cache local ? Que se passe-t-il avec un fichier qui n’a pas encore été téléversé si Rclone tombe en panne ? Un conteneur en cours d’exécution voit-il à nouveau le montage reconstruit ? Et comment le service réagit-il lorsque le cloud est temporairement inaccessible ?

Cet article fournit un plan de test général à cet effet. Vous pouvez l’utiliser pour une archive de documents, un serveur multimédia, une gestion de photos ou tout autre service qui accède, via Rclone, à des fichiers rarement nécessaires stockés à froid.

## Définir d’abord ce que vous voulez atteindre

Le stockage à froid ne signifie pas automatiquement la même chose pour chaque application. Un serveur multimédia lit généralement de gros fichiers de manière séquentielle. Une gestion de photos charge de nombreuses petites vignettes et saute à différents endroits. Une archive de documents ouvre des fichiers relativement petits, mais souvent une seule fois.

Avant le test, notez les principales caractéristiques de votre jeu de données réel :

- taille de fichier typique et plus gros fichier présent
- nombre de fichiers par répertoire
- lecture complète ou accès aléatoire à certaines zones
- proportion entre les accès en lecture et en écriture
- nombre d’utilisateurs ou de processus simultanés
- modifications effectuées directement dans le remote, hors du montage
- délai d’attente acceptable pour une lecture à froid
- espace maximal disponible pour le cache local

Ce n’est qu’à partir de là que des critères de réussite pertinents peuvent être définis. Ouvrir un fichier en 1,2 seconde peut être tout à fait acceptable pour une archive et inutilisable pour une application interactive.

## Créer un jeu de test reproductible

Rclone fournit déjà un outil adapté à cet effet. `rclone test makefiles` génère à chaque fois la même arborescence de fichiers avec une graine fixe :

```bash
rclone test makefiles ./testdata \
  --seed 42 \
  --files 250 \
  --files-per-directory 25 \
  --min-file-size 16K \
  --max-file-size 32M
```

Adaptez le nombre et les tailles à votre jeu de données réel. Ne testez pas uniquement des fichiers moyens. Quelques très petits fichiers montrent le coût des accès aux métadonnées ; quelques gros fichiers rendent visibles le débit, le read-ahead et le comportement du cache.

Ajoutez également des noms de fichiers susceptibles de poser problème en pratique :

```bash
mkdir -p "testdata/Cas particuliers/Sous-dossier"
printf 'Espaces\n' > "testdata/Cas particuliers/Fichier avec espaces.txt"
printf 'Caractères accentués\n' > "testdata/Cas particuliers/Taille et modification.txt"
printf 'Majuscules\n' > "testdata/Cas particuliers/Test.txt"
printf 'Minuscules\n' > "testdata/Cas particuliers/test.txt"
```

Le dernier test est particulièrement important lorsque le système de fichiers local et le backend cloud traitent différemment les majuscules et les minuscules.

Si votre service n’accepte que certains formats, des fichiers binaires quelconques ne suffisent pas. Générez alors aussi des fichiers synthétiques dans ces formats précis. Avec Paperless-ngx, il s’agissait de PDF dotés d’une véritable couche de texte, afin que le test ne mesure pas par inadvertance les performances d’OCR plutôt que le chemin de stockage. Pour une gestion de photos, le jeu de données doit inclure différentes tailles et formats d’images ; pour un serveur multimédia, de courts fichiers utilisant différents codecs.

## Une mesure de référence sans montage

Avant que FUSE et le cache VFS n’entrent en jeu, mesurez directement le backend. Copiez le jeu de données vers le remote de test avec Rclone et enregistrez un journal détaillé :

```bash
rclone copy ./testdata remote:cold-storage-test \
  --progress \
  --stats 5s \
  --log-file rclone-copy.log \
  --log-level INFO
```

Vérifiez ensuite que la source et la destination correspondent :

```bash
rclone check ./testdata remote:cold-storage-test \
  --one-way \
  --download
```

Avec `--download`, Rclone lit effectivement les données et les compare, même si le backend ne fournit pas de hash adaptés. Cela prend plus de temps, mais fournit une base utile pour le test d’intégrité ultérieur.

Consignez le temps de téléversement, le débit de transfert, le nombre de tentatives et les erreurs d’API. Si l’accès direct est déjà instable, le montage ne pourra pas le réparer.

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

Les valeurs ne sont qu’un exemple et non une recommandation générale. Ce qui compte est la séparation : un cache de test vide rend les lectures à froid reproductibles sans devoir supprimer de fichiers d’un cache de production en cours d’utilisation.

`--vfs-cache-mode full` est généralement le mode de test le plus révélateur pour les applications. Rclone met alors localement en tampon les accès en lecture et en écriture et peut mieux reproduire des accès aux fichiers impossibles avec un simple stockage objet. Cette compatibilité supplémentaire consomme de l’espace de stockage local.

## Toujours vérifier du point de vue du véritable service

Un montage peut fonctionner pour votre utilisateur tout en étant inutilisable pour le service. Les causes fréquentes sont un autre ID utilisateur, l’absence de `--allow-other`, les limites des conteneurs ou une propagation de montage incorrecte.

Effectuez donc au moins un accès complet en lecture avec la même identité que celle sous laquelle l’application s’exécutera ensuite :

```bash
sudo -u <service-user> sha256sum /mnt/rclone-test/chemin/vers/fichier
```

Si le service s’exécute dans Docker, le test doit être réalisé dans le conteneur :

```bash
docker exec --user <uid>:<gid> <app-container> \
  sha256sum /chemin/dans/conteneur/fichier
```

Mieux encore : effectuez un véritable test de l’application. Ouvrez le fichier via l’interface web ou l’API du service. C’est la seule façon de constater si l’application lance par exemple plusieurs lectures parallèles, saute à la fin du fichier ou attend des métadonnées supplémentaires.

## Mesurer séparément les lectures à froid et à chaud

Avec `--vfs-cache-mode full`, il existe trois niveaux entre l’application et le cloud :

| Niveau | Ce qui s’y trouve |
|---|---|
| Remote | le fichier complet dans le service cloud |
| Cache VFS | zones localement stockées de fichiers déjà lus |
| Cache de pages Linux | données récemment utilisées en RAM |

Pour une lecture à froid, choisissez un fichier dont le contenu n’a encore jamais été lu via le montage de test. Lors de la lecture à chaud effectuée directement après, il se trouve dans le cache VFS et, le plus souvent, aussi en RAM.

```bash
measure_read() {
  file="$1"
  label="$2"
  start=$(date +%s%3N)
  cat "$file" > /dev/null
  end=$(date +%s%3N)
  printf '%s: %s ms\n' "$label" "$((end - start))"
}

measure_read "/mnt/rclone-test/gros-fichier.bin" "Lecture à froid"
measure_read "/mnt/rclone-test/gros-fichier.bin" "Lecture à chaud"
```

Ne mesurez pas un seul fichier. Utilisez au moins dix fichiers de tailles différentes qui n’ont pas encore été lus et notez la médiane, la valeur la plus lente et la taille du fichier. Un unique meilleur résultat ne constitue pas une base de décision.

Une lecture à chaud n’est pas un simple test du disque, car le noyau peut conserver certaines parties du fichier en RAM. Pour la plupart des scénarios de stockage à froid, ce n’est pas un problème. L’essentiel est ce qu’un utilisateur constate lors de la première ouverture et lors d’une ouverture répétée. Si vous voulez évaluer séparément la RAM et le disque local, vous devez également contrôler et vider de manière vérifiable le cache de pages.

## Ne pas tester uniquement les lectures complètes

`cat` lit un fichier du début à la fin. De nombreuses applications se comportent autrement :

- Un lecteur vidéo lit d’abord l’en-tête et l’index, saute ensuite à une autre position puis poursuit la lecture séquentielle.
- Une gestion d’images lit les métadonnées puis génère une vignette.
- Un logiciel d’archivage peut commencer par lire la fin du fichier.
- Plusieurs workers peuvent accéder simultanément à différents fichiers.

Testez ces flux avec l’application réelle. Observez parallèlement le journal Rclone et le cache. Pour les gros fichiers, il est intéressant de voir quelle quantité Rclone stocke réellement en local et si `--vfs-read-ahead` correspond au modèle d’accès.

Un montage Rclone n’est par ailleurs pas un emplacement de stockage pertinent pour les bases de données ou d’autres fichiers nécessitant un verrouillage fiable et des modifications fréquentes au sein d’un même fichier. La couche VFS compense les différences entre système de fichiers et stockage objet, mais ne transforme pas le backend en système de fichiers local.

## Valider séparément le chemin d’écriture

Si votre service ne fait que lire, montez le remote en lecture seule lorsque cela est possible. S’il doit écrire, testez séparément la création, l’écrasement, le renommage et la suppression.

Un fichier écrit n’apparaît pas forcément immédiatement dans le remote. Lorsque le cache VFS est actif, le téléversement ne commence qu’après la fermeture du fichier et l’expiration de `--vfs-write-back`. Vérifiez donc les deux états :

1. L’application a bien fermé le fichier.
2. Le fichier est ensuite lisible dans le remote via un accès Rclone direct.

```bash
printf 'writeback-test\n' > /mnt/rclone-test/writeback-test.txt

# Après expiration de --vfs-write-back :
rclone cat remote:cold-storage-test/writeback-test.txt
```

Répétez le test avec un gros fichier et arrêtez Rclone pendant que le téléversement est encore en cours. Redémarrez ensuite avec le même répertoire de cache et vérifiez si le téléversement reprend. C’est précisément cette fenêtre temporelle qui détermine la quantité de données menacées en cas de panne du serveur.

Testez aussi le renommage et la suppression. De nombreux backends cloud représentent ces opérations différemment d’un système de fichiers local. Ce qui importe n’est pas seulement que la commande se termine avec succès, mais aussi le moment où la modification devient visible lors d’un accès direct au remote et pour les autres clients.

## Tester les modifications hors du montage

Des fichiers peuvent être modifiés via l’interface web du fournisseur, un second processus Rclone ou un autre serveur. Le montage ne voit pas toujours ces changements immédiatement, car les informations de répertoire sont mises en cache.

Créez donc un fichier directement dans le remote avec un second appel Rclone :

```bash
printf 'external-change\n' > external-change.txt
rclone copyto external-change.txt \
  remote:cold-storage-test/external-change.txt
```

Mesurez quand le fichier apparaît dans le montage. Répétez le test pour une modification et une suppression. Le résultat dépend du backend, de sa prise en charge du polling ainsi que de `--poll-interval` et `--dir-cache-time`. Si l’application doit voir immédiatement les modifications récentes, ce comportement doit figurer explicitement dans les critères de validation.

Lorsque l’interface Remote Control est activée, vous pouvez vider de manière ciblée le cache de répertoires :

```bash
rclone rc vfs/forget
```

C’est utile pour un test manuel, mais ne remplace pas une stratégie d’exploitation adaptée.

## Mettre le cache sous pression

Un cache presque vide est le cas le plus simple. Lors d’une deuxième phase de test, définissez volontairement `--vfs-cache-max-size` à une valeur faible et lisez plus de données qu’il ne peut en contenir.

```bash
du -sh /var/cache/rclone-test/vfs
du -sh --apparent-size /var/cache/rclone-test/vfs
find /var/cache/rclone-test/vfs -type f | wc -l
```

Les deux tailles peuvent fortement différer. En mode Full, Rclone utilise des fichiers clairsemés : un fichier affiche sa taille logique complète, bien que seules les zones lues occupent de l’espace local.

La limite du cache est en outre souple. Rclone la vérifie au rythme de `--vfs-cache-poll-interval`, et les fichiers ouverts ne peuvent pas être supprimés. Le cache peut donc brièvement dépasser la limite. Il devrait toutefois diminuer à nouveau après la fermeture des fichiers et le prochain cycle de nettoyage.

Consignez la valeur maximale, la valeur après nettoyage et le temps nécessaire. Vous pourrez ainsi dimensionner raisonnablement l’espace de stockage local requis.

## Simuler deux pannes distinctes

Un cloud inaccessible et un processus Rclone qui s’est arrêté sont deux erreurs différentes :

| Panne | Ce que vous vérifiez |
|---|---|
| Backend ou réseau inaccessible, Rclone continue de fonctionner | comportement lors des tentatives, des délais d’attente et pour les fichiers déjà mis en cache |
| Processus Rclone arrêté | comportement du montage FUSE et restauration du point de montage |

Simulez les deux uniquement dans l’environnement de test. Pour le deuxième cas, vous pouvez arrêter brutalement un conteneur Rclone :

```bash
docker kill --signal KILL <rclone-container>
```

Pendant la panne, vérifiez l’application et pas seulement le point de montage :

- Quelles fonctions restent disponibles ?
- Combien de temps un accès attend-il avant qu’une erreur apparaisse ?
- Les fichiers déjà entièrement mis en cache restent-ils accessibles ?
- L’application arrête-t-elle les nouvelles opérations d’écriture ?
- Un message d’erreur compréhensible apparaît-il, ou seulement un processus bloqué ?
- Votre supervision se déclenche-t-elle ?

Un service d’écriture ne doit pas écrire silencieusement dans le répertoire local sous-jacent lorsque le montage est absent. Après le retour du montage, ces fichiers seraient masqués. Une protection simple avant chaque tâche d’écriture est :

```bash
mountpoint -q /mnt/rclone-test || exit 1
```

Après le redémarrage de Rclone, vérifiez le montage sur l’hôte et depuis chaque conteneur consommateur. Un montage reconstruit n’atteint un conteneur déjà en cours d’exécution qu’avec une propagation de montage appropriée. Pour Docker, `rslave` est généralement nécessaire du côté consommateur. Les détails figurent dans l’article [Exploiter de manière fiable les montages Rclone dans Docker](/blog/rclone-mount-in-docker-container).

## Un exemple concret avec Paperless-ngx

Pour mon test Paperless, j’ai généré 40 PDF totalisant 13,9 MB. Un document qui n’avait encore jamais été ouvert nécessitait environ 1,8 seconde, tandis qu’un accès répété immédiatement prenait 19 à 24 millisecondes. Un cache VFS limité à 4 MB est brièvement monté à 12,7 MiB, puis a été nettoyé lors du cycle suivant.

Pendant que le remote était inaccessible, la liste des documents, la recherche plein texte et les vignettes ont continué à fonctionner, car ces données étaient stockées localement. Seul l’original ne pouvait pas être ouvert. Après la reconstruction du montage, le conteneur Paperless en cours d’exécution a de nouveau pu accéder aux fichiers sans devoir être redémarré lui-même.

Ces chiffres ne constituent pas un benchmark pour Rclone ou Proton Drive. Ce qui importe est le comportement : le stockage à chaud est resté disponible localement, les lectures à froid étaient plus lentes mais prévisibles, et le service s’est rétabli après la panne.

## Ce qui doit figurer dans le protocole de test

Un résultat traçable ultérieurement contient au minimum :

- version de Rclone et backend utilisé
- système d’exploitation, variante FUSE et système de fichiers du répertoire de cache
- commande de montage complète sans identifiants
- nombre, répartition des tailles et structure des fichiers de test
- valeurs de lecture à froid et à chaud pour plusieurs fichiers
- durée d’écriture jusqu’à la visibilité dans le remote
- valeur maximale du cache et durée du nettoyage
- résultat de `rclone check --download`
- comportement en cas de panne du backend et d’arrêt du processus Rclone
- temps de rétablissement du point de vue de l’application
- tentatives, délais d’attente, limitations et erreurs d’authentification du journal

Définissez à l’avance une valeur limite pour chaque point. Le test aboutira alors à une décision, et non seulement à une collection de chiffres intéressants.

## Quand l’architecture est prête

Un montage de stockage à froid est prêt à l’emploi si vous pouvez répondre oui à ces questions :

- Les lectures à froid sont-elles assez rapides pour le service prévu ?
- Le cache accélère-t-il les accès répétés comme prévu ?
- L’espace local requis reste-t-il maîtrisable, même sous charge ?
- Tous les fichiers correspondent-ils après un téléchargement complet ?
- Toutes les opérations de fichiers nécessaires fonctionnent-elles avec le backend choisi ?
- L’application se comporte-t-elle de manière contrôlée lors d’une panne du cloud ?
- Les opérations d’écriture sont-elles arrêtées de manière sûre lorsque le montage est absent ?
- Un montage reconstruit atteint-il tous les consommateurs en cours d’exécution ?
- La supervision signale-t-elle la panne avant qu’un utilisateur ne la remonte ?

S’il manque une réponse, vous savez au moins exactement sur quoi poursuivre le travail. C’est bien plus utile qu’un montage qui semblait bon lors du premier `ls` et qui ne révèle ses limites qu’en exploitation.

## Sources

1.  [Rclone test makefiles](https://rclone.org/commands/rclone_test_makefiles/): fichiers de test et structures de répertoires reproductibles avec des tailles configurables.

2.  [Rclone mount](https://rclone.org/commands/rclone_mount/): modes de cache VFS, writeback, fichiers clairsemés, limites du cache et cache de répertoires.

3.  [Rclone check](https://rclone.org/commands/rclone_check/): comparaison de la source et de la destination, y compris une vérification complète avec `--download`.

4.  [Rclone Remote Control](https://rclone.org/rc/): vidage ciblé du cache de répertoires VFS avec `vfs/forget`.
