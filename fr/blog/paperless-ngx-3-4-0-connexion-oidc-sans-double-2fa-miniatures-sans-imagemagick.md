---
title: "Paperless-ngx 3.4.0 : connexion OIDC sans double 2FA, miniatures sans ImageMagick"
navTitle: "Paperless-ngx 3.4.0"
description: "Paperless-ngx 3.4.0 ignore la 2FA locale lorsqu’un fournisseur OIDC signale un second facteur fiable, génère les miniatures PDF avec Poppler au lieu d’ImageMagick et met à jour plusieurs dépendances pour des raisons de sécurité. Avant la mise à jour, vérifiez le nouveau paramètre AMR et, sur les systèmes Bare Metal, que pdftoppm est installé."
date: "2026-10-30"
kategorie: "Paperless-ngx"
timeToRead: "5 min de lecture"
themen:
  - paperless-ngx
produkte:
  - "paperless-ngx"
protokolle:
  - "releases"
slug: "paperless-ngx-3-4-0-connexion-oidc-sans-double-2fa-miniatures-sans-imagemagick"
translationId: article-57582656ad9c441b
translationOf: paperless-ngx-3-4-0
url: https://rafaelpfister.ch/fr/blog/paperless-ngx-3-4-0-connexion-oidc-sans-double-2fa-miniatures-sans-imagemagick
translationSourceHash: fe1a5dd3d088e7e544b9448ecc7099233f8287856a5eded7af7488e06c8f290a
translationModel: gpt-5.6-terra
translatedAt: 2026-10-31T03:19:06.515Z
translationReview: automatic
---

# Paperless-ngx 3.4.0 : connexion OIDC sans double 2FA, miniatures sans ImageMagick

Le projet Paperless-ngx a publié la version 3.4.0 le 29 octobre 2026, une semaine après la 3.3.4. Cette version fonctionnelle apporte un paramètre qui supprime la demande locale de double authentification lors des connexions OIDC lorsque le fournisseur d’identité a déjà confirmé un second facteur. Paperless-ngx génère désormais les miniatures des documents PDF avec les outils Poppler ; ImageMagick n’est plus une dépendance. S’y ajoutent des extensions pour les workflows et les Custom Fields, des modifications de l’upload et des notifications dans l’interface web, huit corrections de bugs ainsi que des mises à jour de dépendances frontend et Python que le projet répertorie comme mises à jour de sécurité. Cette version ne mentionne pas de propre avis de sécurité avec CVE ou GHSA. Elle s’adresse à tous les opérateurs de Paperless-ngx 3.x, en particulier aux installations avec connexion OIDC et aux installations Bare Metal dont vous gérez vous-même les paquets système.

## Les versions en bref

| Version | Date | Type | Points principaux |
|---|---|---|---|
| 3.4.0 | 29.10.2026 | Version fonctionnelle | ignorer la 2FA locale avec un AMR OIDC fiable, miniatures PDF via Poppler, ImageMagick supprimé, Custom Fields dans les titres de workflows, mises à jour de sécurité des dépendances |

## Connexion OIDC : 2FA locale avec AMR fiable

Lors d’une connexion via un fournisseur OIDC, Paperless-ngx évalue désormais le claim `amr` du jeton ID (PR #14403). S’il contient l’une des valeurs de `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`, Paperless-ngx ignore sa propre demande de double authentification pour cette connexion. Selon la documentation de configuration, la valeur par défaut de la variable est `mfa`. La documentation cite comme exemples Authelia, qui signale `mfa`, et Pocket ID, qui signale `phr` pour les passkeys. Pour les codes à usage unique par e-mail, Pocket ID envoie la valeur `otp` ; la documentation la considère comme non fiable.

```shell
PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR=mfa
```

La valeur par défaut étant déjà définie, le comportement change après la mise à jour pour les utilisateurs ayant configuré une 2FA locale dès que leur fournisseur d’identité fournit `mfa` dans le claim `amr`. Vérifiez donc quelles valeurs AMR votre fournisseur envoie réellement et si ces valeurs ne sont définies qu’en cas de véritable second facteur. La documentation exige explicitement de ne renseigner que des valeurs indiquant qu’un second facteur a effectivement été utilisé lors de la connexion. En parallèle, la version 3.4.0 met à jour django-allauth vers 65.20 et oauthlib vers 4.0 (PR #14570).

## Miniatures PDF avec Poppler, ImageMagick supprimé

Jusqu’à présent, Paperless-ngx appelait ImageMagick (`convert -density 300`) pour la miniature d’un PDF, qui rendait la première page via Ghostscript à 300 DPI. À partir de la version 3.4.0, `pdftoppm` des outils Poppler rend la première page, Pillow redimensionne l’image et l’enregistre au format WebP (PR #14420). Paperless-ngx calcule la résolution à partir de la taille de la page, qu’il détermine via `pdfinfo`, avec un maximum de 300 DPI. Le fallback Ghostscript pour les miniatures est supprimé, tandis que la réparation via qpdf demeure. L’auteur de la PR a mesuré des temps d’exécution nettement plus courts et une consommation de mémoire réduite, pour un document A0 vide par exemple 0,21 au lieu de 12,9 secondes ; ces valeurs n’ont pas été vérifiées indépendamment.

Avec la PR #14440, ImageMagick est entièrement supprimé. Les tâches restantes, principalement la suppression d’un canal alpha des images, sont prises en charge par un outil basé sur Pillow ; il enregistre alors les fichiers WebP sans perte. La politique ImageMagick, qui devait jusqu’ici être adaptée pour le traitement des PDF, n’est donc plus nécessaire. Ghostscript reste installé, car OCRmyPDF en a toujours besoin.

Pour les administrateurs, cela a deux conséquences :

- **Bare Metal :** `pdftoppm` doit être disponible. S’il manque, Paperless-ngx émet un avertissement au démarrage. Comparez les paquets système installés avec la liste des dépendances mise à jour dans la documentation d’installation.
- **Configuration :** `PAPERLESS_CONVERT_MEMORY_LIMIT` et `PAPERLESS_CONVERT_TMPDIR` sont, selon la PR #14420, obsolètes et sans effet ; `PAPERLESS_GS_BINARY` a également été marqué comme obsolète. Vous pouvez supprimer ces entrées de `paperless.conf` ou du fichier `.env`.

## Workflows, Custom Fields et interface web

- **Workflows :** les modèles de titre dans les workflows peuvent utiliser des Custom Fields (PR #14564). Dans les champs de tags des workflows, les tags parents et enfants peuvent être sélectionnés indépendamment (PR #14400).
- **Custom Fields :** les requêtes sur les champs de date peuvent être relatives (PR #14549).
- **Upload et notifications :** la barre de navigation dispose d’un bouton d’upload (PR #14442). Le bouton affiche le nombre d’uploads en cours et renvoie vers l’aperçu, tandis que le nombre de notifications diminue (PR #14501, #14500).
- **Interface :** les détails des documents ont été remaniés (PR #14485), le badge des tâches se charge plus rapidement (PR #14552) et le chat IA affiche le Markdown (PR #14409, #14415).

## Corrections de bugs

- Le `document_exporter` échouait lors de l’exportation vers un dossier avec `--delete` lorsqu’un `manifest.json.tmp` obsolète était présent (PR #14526). Les personnes qui utilisent l’exportateur avec `--delete` pour les sauvegardes devraient vérifier après la mise à jour que les exports se terminent à nouveau complètement.
- Lors de la modification d’un document, l’index de recherche n’est mis à jour qu’après l’exécution des workflows (PR #14558).
- Lorsqu’un code-barres ne produit pas de séparation de documents, les tags ne sont plus attribués avec retard (PR #14497).
- Les tâches annulées ou expirées sont affichées de manière plus compréhensible (PR #14566).
- La vue du schéma de l’API ne déclenche plus de vérifications d’autorisation sur les documents (PR #14503), et le type de liste des filtres pour les objets document dans le schéma de l’API est corrigé (PR #14524).
- Les longues désignations de version sont tronquées dans la liste déroulante et la taille des listes déroulantes est uniformisée (PR #14578, #14510).

## Dépendances et sécurité

Les notes de version présentent les mises à jour des dépendances frontend (PR #14581) ainsi que de brotli, djangorestframework, filelock, msgpack et setuptools (PR #14575) comme des mises à jour de sécurité. Les notes de version ne précisent pas quelles vulnérabilités sont corrigées dans ces paquets. Les composants Angular sont en version 22.2.2. Il n’existe pas d’avis de sécurité propre au projet concernant la version 3.4.0 (état au 30 octobre 2026).

Le dernier avis GHSA-c8f3-pqr2-rr24, du 16 octobre 2026, concerne les versions 3.0 à antérieures à 3.3.2 et est corrigé dans la version 3.3.2. Lorsque les fonctions IA étaient activées, les workflows de classification IA pour des documents sans propriétaire pouvaient transmettre des extraits et métadonnées de documents privés d’autres utilisateurs au fournisseur IA configuré (CVSS 3.1 : 4.3, sans CVE). Les systèmes fonctionnant encore avec une ancienne version 3.x corrigent également cette faille en passant à la version 3.4.0.

## Procédure recommandée

La version cible est la 3.4.0. Avant la mise à jour, arrêtez les traitements en cours tels que l’importation de documents et créez une sauvegarde, comme l’exige la documentation d’administration. Pour les installations Docker utilisant une image Docker Hub, arrêtez la pile avec `docker compose down`, téléchargez la nouvelle image avec `docker compose pull` et démarrez avec `docker compose up`, les migrations de base de données étant exécutées automatiquement ; si tout fonctionne, démarrez la pile en arrière-plan avec `-d`. Pour les installations Bare Metal, installez d’abord les nouvelles dépendances système, notamment Poppler avec `pdftoppm`, mettez à jour les paquets Python avec `pip install -r requirements.txt` et exécutez les migrations dans le répertoire `src` avec `python3 manage.py migrate`. Avant la première connexion OIDC après la mise à jour, définissez les valeurs qui doivent figurer dans `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`. Une installation Docker avec stockage de documents externalisé est décrite dans l’article [Utiliser Paperless-ngx avec peu d’espace de stockage](/blog/paperless-dokumente-clouddienst-auslagern). Les notes de version de la 3.4.0 ne mentionnent ni problèmes connus ni changements incompatibles (état au 30 octobre 2026).

## Sources

1.  [GitHub : version Paperless-ngx v3.4.0](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.4.0) : notes de version avec la date, les nouvelles fonctions, les corrections de bugs et les dépendances.

2.  [Paperless-ngx : Changelog](https://github.com/paperless-ngx/paperless-ngx/blob/main/docs/changelog.md) : changelog dans la documentation du projet avec les entrées pour les versions 3.3.x et 3.4.0.

3.  [GitHub : PR #14403, ignorer la 2FA locale pour un AMR OIDC fiable](https://github.com/paperless-ngx/paperless-ngx/pull/14403) : nouvelle évaluation du claim AMR lors de la connexion OIDC.

4.  [Paperless-ngx : documentation de configuration](https://github.com/paperless-ngx/paperless-ngx/blob/main/docs/configuration.md) : description et valeur par défaut de `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`.

5.  [GitHub : PR #14420, miniatures PDF via Poppler](https://github.com/paperless-ngx/paperless-ngx/pull/14420) : passage à `pdftoppm`, paramètres obsolètes et mesures de l’auteur.

6.  [GitHub : PR #14440, ImageMagick supprimé comme dépendance](https://github.com/paperless-ngx/paperless-ngx/pull/14440) : remplacement des appels ImageMagick restants par Pillow.

7.  [GitHub : avis de sécurité GHSA-c8f3-pqr2-rr24](https://github.com/paperless-ngx/paperless-ngx/security/advisories/GHSA-c8f3-pqr2-rr24) : divulgation d’informations dans des workflows de classification IA, versions affectées de 3.0 à antérieures à 3.3.2, CVSS 4.3.

8.  [Paperless-ngx : documentation d’administration](https://github.com/paperless-ngx/paperless-ngx/blob/main/docs/administration.md) : procédure de mise à jour pour Docker et Bare Metal, option de l’exportateur `--delete`.

9.  [API GitHub : versions de Paperless-ngx](https://api.github.com/repos/paperless-ngx/paperless-ngx/releases) : dates de publication des versions 3.3.4 et 3.4.0.
