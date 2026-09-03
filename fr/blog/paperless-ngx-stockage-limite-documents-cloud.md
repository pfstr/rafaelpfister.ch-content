---
title: "Exploiter Paperless-ngx avec peu d’espace de stockage : externaliser les documents vers un service cloud"
navTitle: "Paperless avec un service cloud"
description: "Paperless-ngx n’a besoin de conserver localement que la base de données, l’index de recherche et les aperçus ; les documents eux-mêmes peuvent être stockés dans un service cloud. Résultats du test pratique et installation du modèle prêt à l’emploi en trois commandes."
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
translationId: article-2f00e7c17fc45664
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:24:14.612Z
translationReview: automatic
translationSourceHash: 1df015c7f06b7e3e850423bc79663fcd1ac13e66ec5ecd46eb430a0dc5ab3ad1
url: https://rafaelpfister.ch/fr/blog/paperless-ngx-stockage-limite-documents-cloud
---

Paperless-ngx stocke ses documents dans un répertoire local, et ce répertoire grandit à chaque scan. Pourtant, Paperless n’a guère besoin des fichiers au quotidien : la recherche interroge la base de données, la liste affiche des aperçus, et le fichier proprement dit n’est lu qu’à son ouverture. J’ai donc testé s’il était possible de déplacer le stockage vers un service cloud. L’outil utilisé est Rclone, avec lequel les utilisateurs de Plex montent depuis des années des collections multimédias entières depuis le cloud.

Le résultat : **cela fonctionne dans les deux sens**, et la configuration ne nécessite désormais plus que trois commandes. Cet article résume les résultats du test et explique comment mettre en place vous-même cette configuration. Les détails techniques sont traités dans des articles distincts liés à la fin : propagation des mounts Docker, particularités d’AppArmor, authentification à deux facteurs et méthodologie de mesure.

## Le principe : le Hot Storage reste local, le Cold Storage est dans le cloud

| Composant | Emplacement | Pourquoi |
|---|---|---|
| Base de données (contient le texte OCR) | local | nécessite un véritable verrouillage |
| Index de recherche, aperçus | local | accès constants |
| **Fichiers de documents** | **Cloud** | rarement lus |
| Cache (documents ouverts récemment) | local, limité | les accès répétés restent rapides |

Dans Paperless, c’est précisément le nom du répertoire qui prête à confusion : `archive/` n’est **pas du Cold Storage**, mais contient la version PDF/A fournie à chaque affichage. Malgré son nom, elle fait partie du Hot Storage. Les originaux rarement nécessaires sous `originals/` constituent le véritable Cold Storage. Si vous souhaitez économiser au maximum, désactivez complètement la copie d’archive avec `PAPERLESS_ARCHIVE_FILE_GENERATION=never` ; la recherche en texte intégral n’en est pas affectée, car le texte se trouve dans la base de données.

Paperless-ngx ne propose d’ailleurs aucune connexion cloud intégrée, ni S3 ni `django-storages`. Un mount de système de fichiers via Rclone est actuellement la seule solution, et il fonctionne avec chacun des plus de 70 services pris en charge par Rclone. Proton Drive a été mon choix de test en raison du chiffrement de bout en bout ; un stockage compatible S3 est l’alternative la plus robuste.

## Ce que le test a montré

Testé avec une instance Paperless isolée, 40 PDF de test générés (13.9 MB) et un compte Proton dédié :

| Opération | Résultat |
|---|---|
| Ouvrir un document pour la première fois (depuis le cloud) | ~1.8 s |
| Rouvrir le même document (depuis le cache) | ~20 ms |
| Ajouter un nouveau document, jusqu’à ce qu’il soit dans le cloud | ~20 s |
| Liste des documents, recherche en texte intégral | 39 ms / 272 ms, fonctionne aussi **sans** connexion cloud |
| Vérification d’intégrité (somme de contrôle de chaque fichier) | réussie, aucun écart |
| Défaillance du mount | auto-rétablissement sans redémarrage de Paperless, vérifié |

Les besoins de stockage local sont ainsi découplés de la taille des archives : la collection peut grandir, pas le disque.

## Comment l’installer

La configuration complète est disponible sous forme de modèle sur GitHub : [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage). Sur le serveur :

```bash
git clone https://github.com/pfstr/paperless-cloud-storage.git
cd paperless-cloud-storage
sudo ./setup.sh      # einmalig, bereitet den Host vor (einziger Root-Schritt)
./wizard.sh          # geführt: Anbieter wählen, Zugangsdaten, Rundlauf-Test
```

L’assistant demande le service cloud (Proton, S3, Backblaze B2, WebDAV, SFTP ou « Not in the list » pour tout autre service Rclone), vérifie la connexion au moyen d’un véritable test d’envoi et de téléchargement, puis démarre le conteneur de stockage. Ensuite :

- **Nouvelle installation :** `docker compose -f paperless.yml up -d`, terminé.
- **Instance Paperless existante :** la base de données et les paramètres restent intacts ; le guide [RETROFIT.md](https://github.com/pfstr/paperless-cloud-storage/blob/main/docs/RETROFIT.md) décrit l’envoi des documents existants et la modification nécessaire de votre fichier Compose.

J’ai volontairement renoncé à une interface web. L’interface graphique web de Rclone a d’abord été utilisée, mais les tunnels SSH, CORS et les mounts éphémères la rendaient pire que la ligne de commande qu’elle était censée remplacer. Trois questions dans le terminal sont plus rapides.

## Pour que le mount reste stable au quotidien

Le modèle prend en charge quatre points que vous devez également considérer pour votre propre configuration :

1. **`propagation: rslave`** sur le media bind mount du conteneur Paperless, sans quoi le conteneur ne survivra pas au redémarrage du mount. Détails et problème AppArmor sous-jacent : [Mount Rclone dans un conteneur Docker](/blog/rclone-mount-in-docker-container).
2. **Arrêter Paperless lorsque le mount est absent.** Sinon, il écrit les documents dans un répertoire local vide, et le mount qui revient les recouvre de manière invisible. Un script watchdog est fourni avec le modèle.
3. **Un compte capable de se connecter sans surveillance.** Avec Proton, cela signifie enregistrer la clé TOTP dans la configuration Rclone. Pourquoi cela ne dévalorise pas l’authentification à deux facteurs et quelle est la situation globale de Proton sous Linux : [Proton Drive sous Linux](/blog/proton-drive-linux-status).
4. **Désactiver les tâches de lecture complète planifiées** (`PAPERLESS_SANITY_TASK_CRON=disable`), car la vérification d’intégrité lirait sinon régulièrement l’ensemble du contenu depuis le cloud.

## Ce que vous devez évaluer avant de l’utiliser

Un document fraîchement ajouté ne se trouve que quelques secondes dans le cache local, jusqu’à la fin de l’envoi. Si la machine tombe en panne précisément durant cette fenêtre, le fichier est perdu. La limite du cache est souple et peut être nettement dépassée pendant de courtes périodes lors de pics d’accès. De plus, le backend Proton de Rclone est officiellement en bêta ; il a montré des symptômes de limitation lors d’appels API rapides. Comme des données de long terme issues d’une utilisation continue font encore défaut, le modèle est indiqué comme expérimental.

La façon dont les mesures ont été obtenues, les pannes simulées et les moyens de tester sérieusement une telle configuration sont décrits dans l’article méthodologique : [Tester les mounts cloud avec des PDF générés](/blog/cloud-mount-testen-dummy-pdfs).

## Conclusion

Utiliser Paperless-ngx sur un petit disque avec un stockage cloud est réalisable et adapté au quotidien : près de deux secondes à la première ouverture, puis la vitesse du cache ; la recherche et l’interface restent indépendantes du cloud, et la configuration se rétablit elle-même après des pannes. Si vous souhaitez seulement économiser quelques gigaoctets sur un serveur normalement dimensionné, vous devriez toutefois faire le calcul : dans mon cas, l’ensemble du stockage occupait 71 MB, contre plusieurs gigaoctets pour le système d’exploitation. Le gain ne réside pas dans l’espace immédiatement économisé, mais dans le fait que le stock peut grandir sans que le disque doive grandir avec lui.

## Sources

1.  [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage) : le modèle de cet article : setup.sh, wizard.sh, fichiers Compose, watchdog et guide de rétrofit.

2.  [Rclone: Overview of cloud storage systems](https://rclone.org/overview/) : les plus de 70 services pris en charge et une comparaison de leurs capacités.

3.  [Paperless-ngx: Configuration](https://docs.paperless-ngx.com/configuration/) : `PAPERLESS_ARCHIVE_FILE_GENERATION`, `PAPERLESS_SANITY_TASK_CRON` et les autres paramètres utilisés.

4.  [Paperless-ngx: Administration](https://docs.paperless-ngx.com/administration/) : Sanity Checker, export et import ainsi que les tâches d’arrière-plan planifiées.
