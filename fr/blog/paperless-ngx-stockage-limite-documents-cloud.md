---
title: "Utiliser Paperless-ngx avec peu d’espace de stockage : externaliser les documents vers un service cloud"
navTitle: "Paperless avec service cloud"
description: "Paperless-ngx ne nécessite en local que la base de données, l’index de recherche et les aperçus ; les documents eux-mêmes peuvent être hébergés dans un service cloud. Résultats du test pratique et configuration avec le modèle prêt à l’emploi en trois commandes."
date: "2026-07-26"
kategorie: "Paperless-ngx"
timeToRead: "6 min de lecture"
themen:
  - paperless-ngx
related:
  - rclone-mount-in-docker-container
  - proton-drive-linux-status
  - cloud-mount-testen-dummy-pdfs
slug: "paperless-ngx-stockage-limite-documents-cloud"
translationOf: "paperless-dokumente-clouddienst-auslagern"
url: "https://rafaelpfister.ch/fr/blog/paperless-ngx-stockage-limite-documents-cloud"
translationId: article-2f00e7c17fc45664
translationModel: gpt-5.6-terra
translatedAt: 2026-07-28T21:10:15.498Z
translationReview: automatic
translationSourceHash: 81212f097221ec6213025dc5de54f583369799181f72747549102e2b4246e021
---

Paperless-ngx stocke ses documents dans un répertoire local, et ce répertoire grossit à chaque numérisation. Or, Paperless a rarement besoin des fichiers au quotidien : la recherche interroge la base de données, la liste affiche des aperçus, et le fichier lui-même n’est lu qu’à son ouverture. J’ai donc testé s’il était possible de déplacer le stockage vers un service cloud. L’outil utilisé est Rclone, avec lequel les utilisateurs de Plex montent depuis des années des collections multimédias entières depuis le cloud.

Le résultat : **cela fonctionne dans les deux sens**, et la configuration ne requiert désormais plus que trois commandes. Cet article résume les résultats du test et explique comment mettre en place vous-même cette configuration. Les détails techniques font l’objet d’articles distincts, liés à la fin : propagation des montages Docker, pièges d’AppArmor, authentification à deux facteurs et méthodologie de mesure.

## Le principe : le stockage chaud reste local, le stockage froid est dans le cloud

| Composant | Emplacement | Pourquoi |
|---|---|---|
| Base de données (contient le texte OCR) | local | nécessite un véritable verrouillage |
| Index de recherche, aperçus | local | accès permanents |
| **Fichiers de documents** | **cloud** | rarement lus |
| Cache (documents ouverts récemment) | local, limité | les accès répétés restent rapides |

Dans Paperless, c’est justement le nom du répertoire qui est trompeur : `archive/` n’est **pas un stockage froid**, mais contient la version PDF/A qui est fournie à chaque consultation. Malgré son nom, elle fait partie du stockage chaud. Les originaux rarement nécessaires sous `originals/` constituent le véritable stockage froid. Si vous voulez économiser au maximum, désactivez complètement la copie d’archive avec `PAPERLESS_ARCHIVE_FILE_GENERATION=never` ; la recherche plein texte n’en est pas affectée, car le texte se trouve dans la base de données.

Paperless-ngx ne propose d’ailleurs pas d’intégration cloud native, ni S3 ni `django-storages`. Un montage de système de fichiers via Rclone est actuellement la seule solution, et elle fonctionne avec chacun des plus de 70 services pris en charge par Rclone. Proton Drive a été mon choix de test en raison du chiffrement de bout en bout ; un stockage compatible S3 est l’alternative plus robuste.

## Résultats du test

Testé avec une instance Paperless isolée, 40 PDF de test générés (13,9 Mo) et un compte Proton dédié :

| Opération | Résultat |
|---|---|
| Ouvrir un document pour la première fois (depuis le cloud) | ~1,8 s |
| Ouvrir à nouveau le même document (depuis le cache) | ~20 ms |
| Ajouter un nouveau document, jusqu’à ce qu’il soit dans le cloud | ~20 s |
| Liste de documents, recherche plein texte | 39 ms / 272 ms, fonctionne aussi **sans** connexion cloud |
| Vérification d’intégrité (somme de contrôle de chaque fichier) | réussie, aucun écart |
| Indisponibilité du montage | auto-rétablissement sans redémarrer Paperless, vérifié |

Les besoins en stockage local sont ainsi découplés de la taille des archives : la collection peut grandir, pas le disque.

## Configuration

La configuration complète est disponible sous forme de modèle sur GitHub : [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage). Sur le serveur :

```bash
git clone https://github.com/pfstr/paperless-cloud-storage.git
cd paperless-cloud-storage
sudo ./setup.sh      # une seule fois, prépare l’hôte (seule étape nécessitant root)
./wizard.sh          # guidé : choisir le fournisseur, identifiants, test complet
```

L’assistant demande le service cloud (Proton, S3, Backblaze B2, WebDAV, SFTP ou « Pas dans la liste » pour tout autre service Rclone), vérifie la connexion par un test réel d’envoi et de téléchargement, puis démarre le conteneur de stockage. Ensuite :

- **Nouvelle installation :** `docker compose -f paperless.yml up -d`, terminé.
- **Instance Paperless existante :** la base de données et les paramètres restent intacts ; le guide [RETROFIT.md](https://github.com/pfstr/paperless-cloud-storage/blob/main/docs/RETROFIT.md) décrit l’envoi des documents existants et la modification nécessaire dans votre fichier Compose.

J’ai volontairement renoncé à une interface web. L’interface graphique web de Rclone a d’abord été utilisée, mais les tunnels SSH, CORS et les montages éphémères la rendaient pire que la ligne de commande qu’elle devait remplacer. Trois questions dans le terminal sont plus rapides.

## Pour que le montage reste stable au quotidien

Le modèle gère quatre points que vous devez également prendre en compte dans une configuration personnalisée :

1. **`propagation: rslave`** sur le montage bind du média du conteneur Paperless, sans quoi le conteneur ne survit pas à un redémarrage du montage. Détails et piège AppArmor associé : [Montage Rclone dans un conteneur Docker](/blog/rclone-mount-in-docker-container).
2. **Arrêter Paperless si le montage est absent.** Sinon, il écrit les documents dans un répertoire local vide, et le montage qui revient les recouvre de manière invisible. Un script watchdog est inclus dans le modèle.
3. **Un compte capable de se connecter sans surveillance.** Avec Proton, cela signifie enregistrer la clé TOTP dans la configuration Rclone. Pourquoi cela ne vide pas l’authentification à deux facteurs de sa substance et où en est Proton sous Linux : [Proton Drive sous Linux](/blog/proton-drive-linux-status).
4. **Désactiver les tâches planifiées de lecture complète** (`PAPERLESS_SANITY_TASK_CRON=disable`), car la vérification d’intégrité lirait sinon régulièrement l’ensemble du contenu depuis le cloud.

## Ce qu’il faut considérer avant de l’utiliser

Un document fraîchement ajouté ne se trouve que quelques secondes dans le cache local, jusqu’à la fin de l’envoi. Si la machine tombe en panne précisément pendant cette fenêtre, le fichier manque. La limite du cache est souple et peut être nettement dépassée brièvement lors de pics d’accès. De plus, le backend Proton de Rclone est officiellement en bêta ; il a présenté des symptômes de limitation lors d’appels API rapides. Comme les données de long terme en fonctionnement continu manquent encore, le modèle est signalé comme expérimental.

L’article méthodologique explique comment les mesures ont été obtenues, quelles pannes ont été simulées et comment tester sérieusement une telle configuration : [Tester des montages cloud avec des PDF générés](/blog/cloud-mount-testen-dummy-pdfs).

## Conclusion

Faire fonctionner Paperless-ngx sur un petit disque avec un stockage cloud est possible et adapté à un usage quotidien : près de deux secondes à la première ouverture, puis la vitesse du cache ; la recherche et l’interface restent indépendantes du cloud, et la configuration se rétablit d’elle-même après des pannes. Si vous souhaitez seulement économiser quelques gigaoctets sur un serveur normalement dimensionné, vous devriez toutefois faire le calcul : dans mon cas, l’ensemble du stockage occupait 71 Mo, tandis que le système d’exploitation en occupait plusieurs gigaoctets. Le bénéfice ne réside pas dans l’espace économisé immédiatement, mais dans le fait que les archives peuvent grandir sans que le disque doive grandir avec elles.

## Sources

1.  [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage) : le modèle présenté dans cet article : setup.sh, wizard.sh, fichiers Compose, watchdog et guide de rétrofit.

2.  [Rclone: Overview of cloud storage systems](https://rclone.org/overview/) : les plus de 70 services pris en charge et une comparaison de leurs fonctionnalités.

3.  [Paperless-ngx: Configuration](https://docs.paperless-ngx.com/configuration/) : `PAPERLESS_ARCHIVE_FILE_GENERATION`, `PAPERLESS_SANITY_TASK_CRON` et les autres paramètres utilisés.

4.  [Paperless-ngx: Administration](https://docs.paperless-ngx.com/administration/) : vérificateur de cohérence, export et import, ainsi que les tâches d’arrière-plan planifiées.
