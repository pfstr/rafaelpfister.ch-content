---
title: "Paperless-ngx 3.4.1, 3.4.2 et 3.5.0 : ALLOWED_HOSTS plus strict, 2FA OIDC corrigée, opérations en masse asynchrones"
navTitle: "Paperless-ngx 3.5.0"
description: "Avec PAPERLESS_URL défini, Paperless-ngx 3.5.0 n’autorise plus que son nom d’hôte et localhost, corrige la valeur AMR par défaut inefficace de la version 3.4.x et exécute les opérations en masse de manière asynchrone. Avant la mise à jour, vérifiez les noms d’hôte utilisés pour accéder à Paperless-ngx et ajoutez-les à PAPERLESS_ALLOWED_HOSTS."
date: "2026-11-17"
kategorie: "Paperless-ngx"
timeToRead: "7 min de lecture"
themen:
  - paperless-ngx
produkte:
  - "paperless-ngx"
protokolle:
  - "releases"
slug: "paperless-ngx-3-4-1-3-4-2-et-3-5-0-allowed-hosts-plus-strict-2fa-oidc-corrigee-operations-en"
translationId: article-4c816dd3cd08ff6e
translationOf: paperless-ngx-3-5-0
url: https://rafaelpfister.ch/fr/blog/paperless-ngx-3-4-1-3-4-2-et-3-5-0-allowed-hosts-plus-strict-2fa-oidc-corrigee-operations-en
translationSourceHash: 2c1c3db7502c455d8090a922dd17654a159d2d094be4f29d0da51f29eababfb4
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T03:21:39.673Z
translationReview: required
---

# Paperless-ngx 3.4.1, 3.4.2 et 3.5.0 : ALLOWED_HOSTS plus strict, 2FA OIDC corrigée, opérations en masse asynchrones

Depuis la version 3.4.0, le projet Paperless-ngx a publié trois versions : les correctifs 3.4.1, le 31 octobre 2026, et 3.4.2, le 3 novembre 2026, ainsi que la version fonctionnelle 3.5.0, le 17 novembre 2026. Les deux versions correctives corrigent principalement des erreurs d’affichage de l’interface web remaniée et mettent à jour Django ainsi que d’autres dépendances. La version 3.5.0 modifie la valeur par défaut des noms d’hôte autorisés, corrige une erreur qui rendait le réglage `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR` introduit dans la version 3.4.0 sans effet, exécute toutes les opérations en masse de manière asynchrone, nettoie automatiquement les sessions expirées et apporte des workflows déclenchables manuellement. Aucune des trois versions ne mentionne d’avis de sécurité avec CVE ou GHSA. L’état précédent est décrit dans l’article [Paperless-ngx 3.4.0 : connexion OIDC sans double 2FA, miniatures sans ImageMagick](/blog/paperless-ngx-3-4-0). Les mises à jour s’adressent à tous les exploitants de Paperless-ngx 3.x ; elles concernent particulièrement les installations accessibles via des noms d’hôte internes et celles utilisant une connexion OIDC.

## Vue d’ensemble des versions

| Version | Date | Type | Points essentiels |
|---|---|---|---|
| 3.4.1 | 31.10.2026 | Version corrective | Corrections du schéma API et de l’interface web, consommation mémoire réduite au repos |
| 3.4.2 | 03.11.2026 | Version corrective | Tâches de modification en masse dans les workers Celery, installation des langues Tesseract avant la création de l’index, filelock 4.2.6, mises à jour de Django et des dépendances frontend |
| 3.5.0 | 17.11.2026 | Version fonctionnelle | ALLOWED_HOSTS sans joker lorsque PAPERLESS_URL est défini, valeur AMR par défaut corrigée, opérations en masse asynchrones, nettoyage automatique des sessions, déclencheur manuel de workflow, hiérarchie des tags sans django-treenode |

## 3.4.1 et 3.4.2 : corrections après la version fonctionnelle

La version 3.4.1 corrige le placement d’une correction de schéma pour les champs `id` (PR #14604), supprime les contrôles d’autorisation en double pour le bouton d’ouverture des cartes de documents (PR #14601), masque largement les nouveaux contrôles PDF (PR #14595), déplace le bouton d’aperçu dans la liste des documents (PR #14594) et corrige l’affichage de la somme de contrôle en mode clair ainsi que la position du menu déroulant des champs personnalisés sur les appareils mobiles (PR #14590). En outre, Paperless-ngx ne charge davantage de modules qu’en cas de besoin, ce qui réduit la consommation mémoire au repos (PR #14599).

La version 3.4.2 comprend onze corrections de bogues. Les éléments pertinents pour l’exploitation sont les suivants :

- Les tâches d’importation de `bulk_edit` s’exécutent de manière fiable dans les workers Celery (PR #14653).
- La création de l’index de recherche et de l’index LLM attend que les paquets linguistiques Tesseract soient installés (PR #14670).
- filelock passe à la version 4.2.6 ; les notes de version justifient cette mise à jour par des corrections liées aux ACL (PR #14656).
- L’historique des documents affiche les noms des tags au lieu des ID (PR #14625), les boutons de zoom ne soumettent plus les formulaires (PR #14624) et le nom d’une vue enregistrée est conservé lors de l’enregistrement et de la fermeture (PR #14667).

Côté dépendances, la version 3.4.2 met à jour Django, django-guardian, django-filter et drf-spectacular-sidecar (PR #14661), turbohtml vers 1.15.3 (PR #14662) et pdfjs-dist vers 6.4.299 (PR #14644). Le projet remplace la bibliothèque non maintenue ngx-bootstrap-icons par sa propre solution (PR #14665). Dans les fichiers Compose d’exemple, Gotenberg passe à 8.38 (PR #14634) et nginx à 1.31.7 (PR #14633). Les notes de version de 3.4.1 et 3.4.2 ne qualifient aucun de ces éléments de mise à jour de sécurité.

## 3.5.0 : ALLOWED_HOSTS sans joker

Les notes de version de 3.5.0 commencent par l’indication que certains utilisateurs doivent adapter leur réglage `PAPERLESS_ALLOWED_HOSTS` (PR #14735). Jusqu’à la version 3.4.2, le projet documentait pour `PAPERLESS_ALLOWED_HOSTS` la valeur par défaut `*`, c’est-à-dire tous les hôtes. À partir de la version 3.5.0, la documentation de configuration indique :

- Si ni `PAPERLESS_URL` ni `PAPERLESS_ALLOWED_HOSTS` ne sont définis, Paperless-ngx continue d’accepter tout hôte.
- Si seul `PAPERLESS_URL` est défini, Paperless-ngx n’accepte que le nom d’hôte de cette URL et `localhost`.
- `PAPERLESS_ALLOWED_HOSTS` ajoute des hôtes supplémentaires à `localhost` et au nom d’hôte de `PAPERLESS_URL`. `localhost` reste toujours autorisé, car le contrôle d’état Docker en a besoin.
- `PAPERLESS_ALLOWED_HOSTS=*` autorise tout hôte, même si `PAPERLESS_URL` est défini.

Sont concernées les installations avec `PAPERLESS_URL` défini, accessibles en outre sous d’autres noms, par exemple via un nom d’hôte interne, une adresse IP ou le nom de service d’un réseau de conteneurs par lequel d’autres services appellent l’API. Django rejettera ces requêtes après la mise à jour. Avant la mise à jour, ajoutez tous les noms supplémentaires à `PAPERLESS_ALLOWED_HOSTS`, par exemple :

```shell
PAPERLESS_URL=https://paperless.example.com
PAPERLESS_ALLOWED_HOSTS=paperless.intern.example.com
```

Vous ne devriez définir la valeur joker `*` que si Paperless-ngx est accessible exclusivement dans un réseau isolé ; pour les installations sur Internet ouvert, la documentation recommande explicitement de définir les hôtes autorisés.

## 3.5.0 : connexion OIDC et sessions

La version 3.4.0 a introduit le réglage `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`, qui permet à Paperless-ngx d’ignorer la demande locale de deuxième facteur lorsque le fournisseur OIDC signale un deuxième facteur fiable dans la revendication `amr`. Selon l’issue #14692, la liste des valeurs fiables restait toujours vide en 3.4.x : le code transmettait `mfa` comme séparateur au lieu de valeur par défaut. Sans variable définie, et même avec `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR=mfa`, Paperless-ngx demandait donc toujours le deuxième facteur local. La PR #14693 corrige cela dans la version 3.5.0. La valeur par défaut documentée est `mfa` ; les connexions par mot de passe exigent toujours le code de Paperless-ngx.

Le changement de comportement décrit dans l’article sur la version 3.4.0 ne prend donc réellement effet qu’avec la version 3.5.0 : les utilisateurs ayant configuré localement la 2FA et dont le fournisseur d’identité fournit `mfa` dans la revendication `amr` ne verront plus de seconde demande après la mise à jour. Avant la mise à jour, vérifiez quelles valeurs AMR votre fournisseur envoie. La documentation exige de n’indiquer que les valeurs signifiant qu’un deuxième facteur a effectivement été utilisé lors de la connexion. Parallèlement, django-allauth passe à la version 65.21.1 (PR #14672).

Jusqu’à présent, Paperless-ngx ne supprimait jamais les sessions expirées de la base de données. Le changement a été déclenché par l’issue #14721 : avec `PAPERLESS_AUTO_LOGIN_USERNAME`, une nouvelle session était créée toutes les 30 secondes et n’était jamais supprimée. À partir de la version 3.5.0, une tâche planifiée supprime les sessions expirées (PR #14722). Le moment est contrôlé par `PAPERLESS_SESSION_CLEANUP_TASK_CRON`, avec `0 3 * * *` comme valeur par défaut, soit chaque jour à 03:00.

## 3.5.0 : opérations en masse, workflows et tags

- **Opérations en masse asynchrones :** Toutes les opérations en masse sur les documents et la modification en masse des métadonnées passent par la file d’attente des tâches (PR #14773, #14711) et apparaissent dans la liste des tâches. Selon la documentation de l’API, les opérations en masse mises en file répondent avec `{"result":"OK","task_id":"…"}`; vous interrogez le résultat via `/api/tasks/?task_id=<uuid>`. Lorsque la tâche en masse est terminée, les tâches suivantes, telles que le retraitement de fichiers, sont seulement planifiées et non terminées. Adaptez en conséquence les scripts qui exploitent directement le résultat après l’appel.
- **Déclencheur manuel de workflow :** Les workflows peuvent être appliqués manuellement à des documents sélectionnés avec le nouveau type de déclencheur, dans la modification en masse et dans les détails du document (PR #14704).
- **Hiérarchie des tags :** Paperless-ngx enregistre désormais la hiérarchie des tags sous forme de liste d’adjacence et renonce à django-treenode (PR #14612). La mise à jour comprend une migration de base de données à cet effet. Dans un prototype avec 6 000 tags sur SQLite, l’auteur mesure des temps nettement plus courts, par exemple environ 1,3 au lieu de 4 409 millisecondes pour créer un tag ; ces valeurs n’ont pas été vérifiées indépendamment. Lors de la suppression d’un tag, Paperless-ngx vérifie en outre les autorisations sur les tags enfants (PR #14687).
- **Autres nouveautés :** Les notes sur les documents sont affichées en Markdown (PR #14626). Un modèle d’embedding Hugging Face mis en cache localement se charge sans connexion au Hub (PR #14728), et le chat IA filtre aussi parmi plus de 32 700 documents (PR #14754).

## 3.5.0 : autres corrections de bogues

- **Arguments OCR :** L’API rejette les arguments que Paperless-ngx transmet lui-même à OCRmyPDF (`input_file_or_options`, `output_file` et `sidecar`) ; ils sont ignorés dans `PAPERLESS_OCR_USER_ARGS` (PR #14734). Vérifiez si votre configuration contient ces arguments.
- **Workflows :** Le filtre de nom de fichier des workflows planifiés utilise une expression régulière compatible avec PostgreSQL (PR #14783).
- **Courrier :** Les e-mails enregistrés sont écrits avec des fins de ligne CRLF (PR #14745), et la conversion PDF/A des archives de courrier ne s’exécute qu’une seule fois (PR #14698).
- **Miniatures :** Pour les images sans version archivée, Paperless-ngx génère de nouveau des miniatures (PR #14741).
- **Autorisations et comptes :** Les autorisations de la vue de configuration sont harmonisées (PR #14709), Paperless-ngx verrouille les lignes de base de données concernées lors de l’enregistrement du premier utilisateur (PR #14688), et les mots de passe sont affichés avec un masque constant (PR #14784). L’auteur de la PR #14709 ne qualifie explicitement pas ce changement de vulnérabilité.

## Dépendances et sécurité

Il n’existe aucun avis de sécurité du projet concernant les versions 3.4.1, 3.4.2 ou 3.5.0 (état au 17 novembre 2026). L’avis le plus récent, GHSA-c8f3-pqr2-rr24 du 16 octobre 2026, concerne les workflows de classification IA sur des documents sans propriétaire et est corrigé depuis la version 3.3.2 ; les détails figurent dans l’article sur la version 3.4.0. Les systèmes sur une version 3.x plus ancienne corrigent également cette faille en passant à la version 3.5.0.

## Problèmes connus

- **2FA locale avec OIDC malgré AMR `mfa` (3.4.0 à 3.4.2) :** corrigé dans la version 3.5.0 (issue #14692, PR #14693).
- **Problèmes connus de 3.5.0 :** les notes de version n’en mentionnent aucun (état au 17 novembre 2026). L’indication concernant `PAPERLESS_ALLOWED_HOSTS` correspond à un changement de comportement intentionnel.

## Procédure recommandée

La version cible est 3.5.0. Avant la mise à jour, déterminez les noms d’hôte utilisés pour accéder à Paperless-ngx et ajoutez-les à `PAPERLESS_ALLOWED_HOSTS` ; vérifiez également les valeurs dans `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`. Arrêtez les traitements en cours, tels que l’importation de documents, et créez une sauvegarde, comme l’exige la documentation d’administration. Pour les installations Docker utilisant une image de Docker Hub, arrêtez la pile avec `docker compose down`, téléchargez la nouvelle image avec `docker compose pull` et démarrez avec `docker compose up` ; les migrations de base de données, dont la nouvelle migration de la hiérarchie des tags, s’exécutent alors automatiquement. Pour les installations Bare Metal, mettez à jour les paquets Python avec `pip install -r requirements.txt` et exécutez les migrations dans le répertoire `src` avec `python3 manage.py migrate` ; ensuite, `python3 manage.py document_index reindex --if-needed` et `python3 manage.py document_llmindex migrate` mettent au besoin à jour l’index de recherche et l’index LLM. En cas de mise à jour directe depuis la version 3.3.x, les indications de l’article sur la version 3.4.0 s’appliquent également, notamment concernant `pdftoppm`. L’article [Exploiter Paperless-ngx avec peu d’espace de stockage](/blog/paperless-dokumente-clouddienst-auslagern) décrit une installation Docker avec stockage de documents externalisé.

## Sources

1.  [GitHub : Paperless-ngx Release v3.5.0](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.5.0) : notes de version avec indication sur `PAPERLESS_ALLOWED_HOSTS`, nouvelles fonctions et corrections de bogues.

2.  [GitHub : Paperless-ngx Release v3.4.2](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.4.2) : corrections de bogues et dépendances mises à jour.

3.  [GitHub : Paperless-ngx Release v3.4.1](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.4.1) : corrections de bogues et consommation mémoire réduite au repos.

4.  [API GitHub : versions de Paperless-ngx](https://api.github.com/repos/paperless-ngx/paperless-ngx/releases) : dates de publication des versions 3.4.1, 3.4.2 et 3.5.0.

5.  [Paperless-ngx : documentation de configuration 3.5.0](https://github.com/paperless-ngx/paperless-ngx/blob/v3.5.0/docs/configuration.md) : nouveau comportement de `PAPERLESS_ALLOWED_HOSTS`, valeur par défaut de `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`, `PAPERLESS_SESSION_CLEANUP_TASK_CRON` et arguments OCR ignorés.

6.  [Paperless-ngx : documentation de configuration 3.4.2](https://github.com/paperless-ngx/paperless-ngx/blob/v3.4.2/docs/configuration.md) : ancienne valeur par défaut `*` pour `PAPERLESS_ALLOWED_HOSTS`.

7.  [GitHub : issue #14692, MFA trusted AMR toujours vide](https://github.com/paperless-ngx/paperless-ngx/issues/14692) : cause et symptômes de la valeur AMR par défaut inefficace dans la version 3.4.x.

8.  [GitHub : PR #14722, nettoyage automatique des sessions](https://github.com/paperless-ngx/paperless-ngx/pull/14722) : contexte des sessions jamais supprimées en cas de connexion automatique.

9.  [GitHub : PR #14612, hiérarchie des tags sous forme de liste d’adjacence](https://github.com/paperless-ngx/paperless-ngx/pull/14612) : remplacement de django-treenode, migration et mesures de l’auteur.

10.  [Paperless-ngx : documentation API 3.5.0](https://github.com/paperless-ngx/paperless-ngx/blob/v3.5.0/docs/api.md) : opérations en masse asynchrones et interrogation via `task_id`.

11.  [Paperless-ngx : documentation d’administration 3.5.0](https://github.com/paperless-ngx/paperless-ngx/blob/v3.5.0/docs/administration.md) : procédure de mise à jour pour Docker et Bare Metal.

12.  [GitHub : avis de sécurité de Paperless-ngx](https://github.com/paperless-ngx/paperless-ngx/security/advisories) : aperçu des avis, aucun ne concernant les versions 3.4.1 à 3.5.0.
