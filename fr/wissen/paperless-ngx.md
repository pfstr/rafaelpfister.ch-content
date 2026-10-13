---
title: "Paperless-ngx : pipeline documentaire, OCR et exploitation de l’archive"
blatt: "paperless-ngx"
description: "Paperless-ngx pour les administrateurs DMS, de plateformes et de messagerie : architecture Django/Angular, pipeline Consumer et OCR, fichiers originaux et d’archive, métadonnées, recherche Tantivy, Celery et Redis, PostgreSQL, Tika/Gotenberg, autorisations, récupération des e-mails, API, exploitation, sauvegarde, restauration et histoire technique."
fakten:
  - label: Rôle du système
    wert: système de gestion documentaire auto-hébergé pour la capture, l’OCR, la classification, la recherche, la conservation et l’exportation
    href: https://docs.paperless-ngx.com/
  - label: Stack technologique
    wert: Python et Django côté backend, Angular dans le navigateur, tâches Celery, OCRmyPDF/Tesseract et index de recherche Tantivy
    href: https://github.com/paperless-ngx/paperless-ngx
  - label: Services obligatoires
    wert: processus Web/de tâches, broker compatible Redis, base de données et répertoires Data/Media persistants
    href: https://docs.paperless-ngx.com/configuration/
  - label: Base de données
    wert: PostgreSQL recommandé pour les nouvelles installations ; MariaDB et SQLite sont également pris en charge
    href: https://docs.paperless-ngx.com/setup/
  - label: Objet document
    wert: enregistrement de base de données plus original, PDF d’archive facultatif, miniature, index plein texte et sommes de contrôle
    href: https://docs.paperless-ngx.com/faq/
  - label: Entrées
    wert: répertoire Consume, envoi via navigateur/API et récupération IMAP d’e-mails basée sur des règles
    href: https://docs.paperless-ngx.com/usage/
  - label: OCR
    wert: OCRmyPDF orchestre la rastérisation et Tesseract ; le mode, la langue, la limite de pages et le format d’archive sont des paramètres d’exploitation
    href: https://docs.paperless-ngx.com/configuration/
  - label: Office et e-mail
    wert: conversion facultative via Apache Tika et Gotenberg ; les deux services constituent un domaine de défaillance distinct
    href: https://docs.paperless-ngx.com/configuration/#optional-services
  - label: Recherche
    wert: index Tantivy à côté de la base de données relationnelle ; l’index est dérivable, la base de données et les originaux font foi
    href: https://docs.paperless-ngx.com/migration-v3/
  - label: Autorisations
    wert: droits Django globaux plus droits Owner, View et Change liés aux objets pour les utilisateurs et les groupes
    href: https://docs.paperless-ngx.com/usage/#permissions
  - label: Noyau de sauvegarde
    wert: export cohérent ou sauvegarde coordonnée de la base de données, de Media, de Data, de la configuration et des secrets
    href: https://docs.paperless-ngx.com/administration/#making-backups
  - label: Règle de réception
    wert: restauration dans une instance vide et compatible ; connexion, nombre de documents, échantillons, recherche, sommes de contrôle et nouvelle exécution Consume doivent fonctionner
    href: https://docs.paperless-ngx.com/administration/#restoring
werbung:
  - newsletter
ctaThemen:
  - paperless-ngx
translationSourceHash: a172bf8c7ed80cecd7c990d657cfdf7d06afe1c0b349350655549c21ed144dfc
translationModel: gpt-5.6-terra
translatedAt: 2026-10-10T10:37:57.993Z
translationReview: automatic
---

# Paperless-ngx : pipeline documentaire, OCR et exploitation de l’archive

Paperless-ngx est un système de gestion documentaire auto-hébergé destiné aux fichiers entrants et aux documents papier numérisés. Il reçoit les documents depuis un répertoire surveillé, le Web ou l’API REST, ou encore depuis des boîtes aux lettres électroniques, produit du texte consultable et, si nécessaire, un exemplaire d’archive normalisé, attribue des métadonnées et rend le fonds consultable par l’interface Web et l’API. La [présentation du projet](https://docs.paperless-ngx.com/) décrit le service comme une archive en ligne consultable ; le [dépôt source](https://github.com/paperless-ngx/paperless-ngx) le présente également comme le successeur communautaire de Paperless et Paperless-ng.

Pour les administrateurs, Paperless-ngx n’est pas une application Web unique, mais un ensemble composé d’une base de données, d’un broker, de processus Web et d’arrière-plan, d’un fonds de fichiers, d’un index de recherche et de convertisseurs externes. Un statut HTTP réussi prouve seulement que le processus Web répond. Il n’indique pas si le Consumer détecte de nouveaux fichiers, si les tâches Celery sont distribuées, si l’OCR fonctionne, si les fichiers d’archive sont écrits, si l’index est à jour ou si une exportation peut être réimportée.

L’explication accompagne un document depuis son entrée, en passant par le Consumer, les analyseurs et l’OCR, jusqu’à la base de données, au fichier d’archive et à l’index de recherche. Viennent ensuite l’automatisation, la sécurité, l’exploitation et une restauration qui contrôle tous les composants ensemble.

Paperless-ngx traite un document comme un pipeline : détecter l’entrée, prendre en charge le fichier, produire l’OCR et les métadonnées, mettre à jour la base de données et l’index de recherche, puis fournir le résultat par le Web ou l’API. L’article suit ce chemin et en déduit l’exploitation, la sauvegarde et le diagnostic.

## Architecture : chemin de contrôle, chemin de traitement et chemin de données

L’architecture peut être décomposée en trois chemins :

1. **Chemin de contrôle :** le navigateur ou le client API communique en HTTP ou HTTPS avec l’application Django. Le frontend Angular est le client interactif ; Django fournit l’authentification, la logique métier, l’interface d’administration et l’API REST. La [documentation de développement](https://docs.paperless-ngx.com/development/) indique que le projet est composé d’une application Django et d’un frontend Angular.
2. **Chemin de traitement :** le Consumer, le planificateur et les workers créent et traitent des tâches asynchrones. Un broker compatible Redis assure la médiation entre les processus ; Celery exécute notamment la consommation, la récupération des e-mails, la maintenance de l’index et la maintenance récurrente. La [documentation d’utilisation](https://docs.paperless-ngx.com/usage/) décrit le Consumer, le processeur de tâches et le broker comme des rôles distincts.
3. **Chemin de données :** la base de données relationnelle conserve les objets document, les métadonnées, les utilisateurs, les règles et les statuts des tâches. Les originaux, versions d’archive et miniatures se trouvent dans le répertoire Media ; les états auxiliaires et les données de recherche se trouvent dans le répertoire Data. L’index de recherche est une projection de ce fonds de référence.

Cette séparation est décisive sur le plan opérationnel. La base de données et le répertoire Media doivent appartenir au même instant logique. Un nouvel index peut être reconstruit, mais un fichier original perdu ne peut pas être reconstruit à partir de l’index. De même, une sauvegarde de base de données sans les fichiers associés ne suffit pas pour restaurer un DMS.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-paperless-ngx.svg?v=20260813" title="Interaktive Infografik: Paperless-ngx von Eingang, Consumer und Task-Broker über Parser, OCR, Archivierung, Metadaten und Suche bis zu Datenbank, Dateien, Backup und Restore" loading="lazy">
  <a href="/images/kb-interaktiv-paperless-ngx.svg?v=20260813">Ouvrir directement le graphique interactif</a>.
</iframe>

## Stack technologique et rôles des processus

Le code backend est principalement implémenté en Python et repose sur Django. Le frontend du navigateur utilise Angular. Les dépendances telles que Django, [Celery](https://docs.celeryq.dev/en/stable/), OCRmyPDF et d’autres analyseurs sont répertoriées dans le [manifeste du projet Python](https://github.com/paperless-ngx/paperless-ngx/blob/dev/pyproject.toml) ; la construction de l’image et les dépendances OCR/PDF natives sont présentées dans le [Dockerfile](https://github.com/paperless-ngx/paperless-ngx/blob/dev/Dockerfile). Les rôles suivants sont pertinents pour l’exploitation :

| Rôle | Tâche principale | Source de données déterminante | Incident typique |
|---|---|---|---|
| Serveur Web | UI, API REST, authentification, envoi | Session et transaction DB | 4xx/5xx, erreur de proxy ou CSRF |
| Consumer | détecter de nouveaux fichiers et déclencher une tâche | répertoire Consume et tâche | fichier restant en place, polling/inotify ou droits |
| Worker de tâches | analyseurs, OCR, classification et étapes d’archivage | statut de tâche, fichiers temporaires, DB | tâche pending/failed, limite mémoire/CPU |
| Planificateur | récupération des e-mails et maintenance périodique | calendrier du système de tâches | aucune tâche périodique |
| Broker | distribution des tâches asynchrones | données volatiles de file d’attente/broker | le Web fonctionne, le traitement est arrêté |
| Base de données | documents, métadonnées, identités, règles | fonds primaire relationnel | erreur de transaction, de schéma ou de capacité |
| Stockage de fichiers | original, archive, miniature | fichiers Media | fichier manquant, lecture seule, montage incohérent |
| Index de recherche | projection plein texte | données d’index dérivables | document présent, recherche incomplète |
| Tika/Gotenberg | conversion Office/e-mail facultative | conversion temporaire | seuls certains types de fichier échouent |

La [référence de configuration](https://docs.paperless-ngx.com/configuration/) désigne le [broker compatible Redis](https://redis.io/docs/latest/) comme un service requis. [PostgreSQL](https://www.postgresql.org/docs/current/), MariaDB et SQLite sont pris en charge comme bases de données ; pour les nouvelles installations, le [guide d’installation](https://docs.paperless-ngx.com/setup/) recommande PostgreSQL. Tika et Gotenberg sont facultatifs et nécessaires ensemble pour les formats Office et le traitement des fichiers e-mail ([services facultatifs](https://docs.paperless-ngx.com/configuration/#optional-services)).

## Inventaire des services et configuration effective

Un inventaire fiable comprend l’image ou la version, le type de base de données, la cible du broker, les montages, les variables d’environnement effectives, les UID/GID, le reverse proxy, le fuseau horaire, les langues OCR et les analyseurs externes. Docker Compose est un empaquetage courant, mais ne constitue pas une preuve d’architecture : plusieurs rôles de processus peuvent résider dans une image de conteneur, tandis que leurs états se trouvent dans des volumes et services distincts.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Paperless-Dienstinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker compose config --services
docker compose ps
docker compose images
docker compose config
docker inspect paperless-webserver</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker compose config --services
docker compose ps
docker compose images
docker compose config
docker inspect paperless-webserver</code></pre>
  </div>
</div>

[`docker compose config`](https://docs.docker.com/reference/cli/docker/compose/config/) affiche la configuration Compose effectivement fusionnée ; [`docker compose ps`](https://docs.docker.com/reference/cli/docker/compose/ps/) affiche les processus et les ports publiés ; [`docker compose images`](https://docs.docker.com/reference/cli/docker/compose/images/) identifie les images ; [`docker inspect`](https://docs.docker.com/reference/cli/docker/inspect/) fournit les montages, le réseau et les paramètres d’exécution. Les sorties peuvent contenir des secrets et ne doivent être intégrées aux tickets qu’après expurgation.

## Entrées et limites de livraison

Paperless-ngx connaît plusieurs entrées qui, avant le pipeline commun, possèdent des domaines de défaillance différents :

- Le **répertoire Consume** est surveillé au moyen d’événements de système de fichiers. Sur les systèmes de fichiers dépourvus d’inotify fiable, comme certains montages NFS, le polling doit être activé ; la [documentation de configuration](https://docs.paperless-ngx.com/setup/#file-systems-without-inotify-support-eg-nfs) le mentionne.
- **Le navigateur et l’API REST** écrivent les envois par le chemin Web. Le reverse proxy, la limite de taille des requêtes, le délai d’attente et le stockage temporaire se situent donc avant le système de tâches.
- La **récupération des e-mails** lit selon des règles des comptes configurés et crée des tâches pour les pièces jointes correspondantes. La connexion IMAP, TLS, OAuth/mot de passe, le dossier, le critère de recherche et l’action de post-traitement font partie du contrat de livraison.

Le répertoire Consume est une zone de transfert et non une archive documentaire. Les fichiers consommés avec succès y sont supprimés, tandis que Paperless crée de nouveaux fichiers gérés dans le fonds Media. La [FAQ](https://docs.paperless-ngx.com/faq/) met explicitement en garde contre la gestion manuelle du répertoire Media.

Une fois les processus et les entrées connus, le traitement proprement dit commence. Chaque étape produit des fichiers, des entrées de base de données et des signaux d’erreur différents.

## Pipeline Consume : de l’entrée à l’objet document

La validation métier n’a pas lieu lors de la détection d’un fichier, mais seulement après plusieurs étapes. Selon le type de fichier et la configuration, le pipeline comprend généralement :

1. **Détection et stabilisation :** le nom de fichier, le type autorisé, la taille du fichier et la fin de l’écriture sont vérifiés.
2. **Création de la tâche :** le Consumer ou le serveur Web transmet une tâche via le broker.
3. **Analyse et prétraitement :** les analyseurs extraient le texte existant ou convertissent le document entrant. Les fichiers Office et e-mail peuvent nécessiter Tika et Gotenberg.
4. **OCR :** OCRmyPDF détermine la reconnaissance de texte selon le mode OCR ; Tesseract reconnaît les langues configurées.
5. **Version d’archive et miniature :** un fichier d’affichage/d’archive et des données d’aperçu sont générés, dans la mesure où le type de fichier et les paramètres le prévoient.
6. **Persistance :** l’original et les dérivés sont écrits ; l’objet de base de données, les sommes de contrôle, le texte et les propriétés techniques sont enregistrés.
7. **Métadonnées et workflows :** les règles attribuent des tags, un correspondant, un type de document, un chemin de stockage, un propriétaire ou d’autres champs.
8. **Indexation :** le fonds plein texte est mis à jour dans l’index de recherche.
9. **Finalisation :** le statut de tâche et le journal indiquent le succès ou l’erreur ; ce n’est qu’à ce stade que le document est trouvable sur le plan métier.

Les scripts pre- et post-consume peuvent étendre le chemin. La [documentation sur les hooks Consume](https://docs.paperless-ngx.com/advanced_usage/#consume-hooks) précise que ces scripts s’exécutent dans le contexte de sécurité et d’erreur du processus Paperless. Ils doivent être versionnés, exécutés avec un délai d’attente et des codes de sortie non ambigus ; un hook défaillant peut bloquer toute l’ingestion.

## OCR, format d’archive et limites de qualité

OCRmyPDF orchestre le traitement PDF et l’OCR Tesseract. Le mode détermine si le texte existant est ignoré, retraité ou forcé. La langue, la limite de pages, le nettoyage d’image, la correction de rotation et le format de sortie influencent le temps CPU, la taille des fichiers et la qualité de reconnaissance. Les paramètres sont décrits dans la [configuration OCR de Paperless](https://docs.paperless-ngx.com/configuration/#ocr-settings) et dans la [documentation OCRmyPDF](https://ocrmypdf.readthedocs.io/en/latest/) ; les modèles linguistiques et le comportement du moteur sont documentés par [Tesseract](https://tesseract-ocr.github.io/tessdoc/).

L’original est conservé comme objet de preuve. Un PDF d’archive est une version dérivée qui peut être régénérée à la suite de modifications ultérieures de l’OCR ou de l’archivage. Cette distinction doit figurer dans les concepts de conservation et de valeur probante :

| Objet | Objectif | Possibilité de restauration |
|---|---|---|
| Original | fichier entrant non modifié | non reconstructible à partir du texte OCR |
| Fichier d’archive | version normalisée, souvent consultable | peut être régénérée à partir de l’original et du pipeline approprié |
| Miniature | aperçu UI | dérivable |
| Texte extrait | recherche et classification | extractible à nouveau, le résultat peut varier selon la version |
| Somme de contrôle | comparaison d’intégrité | probante uniquement avec l’algorithme et la référence correspondante |

Le succès de l’OCR ne garantit pas la lisibilité métier. Les contrôles par échantillonnage doivent comparer le nombre de pages, l’orientation, la couverture de texte, les caractères spéciaux, les tableaux, les signatures intégrées et la taille du fichier. Les pics CPU et RAM surviennent souvent lors du rendu de PDF volumineux ou riches en images, et non lors de l’écriture finale dans la base de données.

## Originaux, archives, miniatures et format de chemin

Un document Paperless se compose de plusieurs éléments couplés. La base de données connaît l’ID interne, le titre, les dates de création et d’ajout, le correspondant, le type de document, les tags, le chemin de stockage, le contenu, le propriétaire et d’autres champs. Le répertoire Media contient l’original et, le cas échéant, les fichiers d’archive et de miniature. Les noms de fichiers configurables et les Storage Paths modifient l’organisation physique, mais non l’identité de l’objet de base de données.

Les administrateurs évitent de renommer ou de déplacer directement des éléments dans le répertoire Media. Le [`document_renamer`](https://docs.paperless-ngx.com/administration/#document-renamer) applique le format configuré de manière cohérente. Un outil externe de synchronisation de fichiers peut sauvegarder le fonds, mais ne doit pas le réorganiser de sa propre initiative parallèlement à l’application en cours d’exécution. Pour les principes de stockage, voir [Storage](/kb/storage), et pour la copie et la vérification, [Rclone](/kb/rclone).

## Métadonnées, matching et workflows

Les correspondants, types de document, tags, chemins de stockage, champs personnalisés et valeurs de date forment une organisation à facettes. Ils ne remplacent pas une arborescence de fichiers, mais décrivent un document selon plusieurs dimensions indépendantes. Le matching automatique fournit des suggestions ou des attributions sur la base du contenu reconnu et des données précédentes ; ce n’est pas un service de vérité immuable.

Les workflows se composent de déclencheurs, de filtres facultatifs et d’actions séquentielles. Ils peuvent définir des métadonnées, des autorisations et d’autres propriétés lors de la consommation ou d’autres événements. La [documentation des workflows](https://docs.paperless-ngx.com/usage/#workflows) décrit les déclencheurs et les actions. Pour les administrateurs, trois règles en découlent :

- Chaque automatisation nécessite un ensemble d’entrée défini et un résultat observable.
- L’ordre des règles et les correspondances multiples sont testés comme du code de politique.
- Après des modifications, les documents existants ne sont réévalués de manière contrôlée qu’avec le [`document_retagger`](https://docs.paperless-ngx.com/administration/#document-retagger).

## Recherche : requête de base de données et projection Tantivy

La recherche plein texte utilise un index Tantivy distinct. La base de données reste la référence pour les objets document et les métadonnées ; l’index accélère la recherche et le classement. Un document peut donc être visible dans la base de données, mais manquer dans les recherches plein texte. La migration vers Tantivy et les champs de recherche modifiés sont documentés dans le [guide de migration](https://docs.paperless-ngx.com/migration-v3/#search-index-whoosh-tantivy).

Après des mises à niveau ou en cas de suspicion de dérive, l’index peut être reconstruit au moyen de la commande de gestion. La réindexation répare une projection, ce n’est pas une restauration des originaux. La planification de capacité tient compte de la taille de l’index, de l’espace temporaire pendant une reconstruction et de la charge d’E/S parallèlement à la consommation normale.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Paperless-Suche und Wartung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker compose exec webserver document_index list
docker compose exec webserver document_index reindex --if-needed
docker compose exec webserver document_sanity_checker
docker compose exec webserver document_thumbnails --help</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker compose exec webserver document_index list
docker compose exec webserver document_index reindex --if-needed
docker compose exec webserver document_sanity_checker
docker compose exec webserver document_thumbnails --help</code></pre>
  </div>
</div>

Les commandes de gestion sont lancées avec Compose dans le conteneur Webserver via [`docker compose exec`](https://docs.docker.com/reference/cli/docker/compose/exec/). Les installations Bare Metal utilisent `python3 manage.py`; la forme d’appel prise en charge et les aides intégrées sont documentées dans l’[administration](https://docs.paperless-ngx.com/administration/#management-utilities).

Les fichiers OCR et les métadonnées ne constituent pas encore à eux seuls un service cohérent. La base de données, le broker de tâches et la projection de recherche doivent correspondre au même état de traitement.

## Base de données, broker et cohérence

PostgreSQL constitue le chemin standard pour les nouvelles installations. MariaDB et SQLite sont possibles, mais diffèrent en matière d’outils de sauvegarde, de comportement en parallèle et de limites d’exploitation. Les migrations de base de données font partie d’une mise à niveau de Paperless ; un ancien binaire restauré ne correspond pas nécessairement à un schéma déjà migré. L’exploitation de la base de données est donc traitée comme un cycle de vie séparé ([PostgreSQL Backup et Restore](https://www.postgresql.org/docs/current/backup.html), [MariaDB Backup and Restore](https://mariadb.com/docs/server/server-usage/backing-up-and-restoring-databases/)).

Le broker transporte les tâches entre les processus. La persistance Redis/Valkey ne remplace pas la sauvegarde de la base de données et des documents. Après la perte du broker, des travaux en cours ou distribués peuvent être relancés, perdus ou apparaître comme des statuts de tâche incohérents. La réception s’effectue donc sur l’objet document et ses fichiers, et non sur une file d’attente vide.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Datenbank- und Brokertests">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker compose exec db pg_isready
docker compose exec broker redis-cli PING
docker compose logs --since 30m webserver
docker compose logs --since 30m broker db</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker compose exec db pg_isready
docker compose exec broker redis-cli PING
docker compose logs --since 30m webserver
docker compose logs --since 30m broker db</code></pre>
  </div>
</div>

[`pg_isready`](https://www.postgresql.org/docs/current/app-pg-isready.html) vérifie l’acceptation des connexions PostgreSQL, et non le schéma ni les requêtes métier. [`redis-cli`](https://redis.io/docs/latest/develop/tools/cli/) teste l’accessibilité du broker avec `PING`. [`docker compose logs`](https://docs.docker.com/reference/cli/docker/compose/logs/) rassemble les journaux de service limités dans le temps ; les identifiants, titres de documents et chemins sont expurgés avant transmission.

## Configuration, Secret Key et résolution des chemins

Avec Docker, les paramètres sont généralement fournis sous forme d’environnement ou de fichier Compose-Env ; `paperless.conf` ne s’applique pas dans ce cas. Les installations Bare Metal recherchent la configuration selon un ordre documenté. Certains paramètres peuvent en outre être enregistrés dans l’application et remplacer les valeurs d’environnement. La configuration effective est donc plus qu’un fichier unique ([ordre de configuration](https://docs.paperless-ngx.com/configuration/#configuration)).

`PAPERLESS_SECRET_KEY` signe les sessions et les valeurs sensibles et est obligatoire. `PAPERLESS_URL`, Allowed Hosts, ainsi que les paramètres CSRF et CORS définissent la limite Web. Les identifiants de base de données, de messagerie, OAuth, Tika et AI facultatifs sont des secrets. Pour de nombreuses valeurs, les configurations Docker peuvent utiliser la variante `_FILE` et donc des secrets montés ; le [guide de configuration](https://docs.paperless-ngx.com/setup/#docker-compose-install) le documente.

La sauvegarde de configuration comprend au minimum :

- la définition Compose/de service et la référence d’image épinglée,
- l’environnement sans copies en clair dans des dépôts non protégés,
- le Secret Key et les références de secrets externes,
- les UID/GID, l’affectation des volumes et bind mounts,
- la configuration de proxy, DNS, certificats et comptes de messagerie,
- les langues OCR, le format de nom de fichier, les workflows et les hooks personnalisés.

## Limite Web, TLS et reverse proxy

L’application devrait être accessible derrière un reverse proxy correctement configuré en HTTPS. Les en-têtes Host, l’URL d’origine, CSRF Trusted Origins, les en-têtes Forwarded et les limites d’envoi doivent être cohérents. Le proxy ne doit définir des en-têtes Remote-User que s’il authentifie de façon fiable l’identité en amont et supprime les en-têtes client portant le même nom.

La [documentation de sécurité](https://docs.paperless-ngx.com/configuration/#hosting-security) décrit le Secret Key et les paramètres URL, Host, CORS et CSRF. Les bases relatives aux certificats et au transport sont disponibles sous [TLS](/kb/tls), et la sécurisation systématique sous [Durcissement](/kb/haertung). Le projet avertit dans le [dépôt](https://github.com/paperless-ngx/paperless-ngx#important-note) que les documents sont stockés en clair et que Paperless-ngx ne doit pas être exploité sur un hôte non fiable.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Paperless-Web- und Portdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection paperless.example.net -Port 443
curl.exe -sS -D - -o NUL https://paperless.example.net/
Resolve-DnsName paperless.example.net
docker compose port webserver 8000</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ss -lntp
curl -sS -D - -o /dev/null https://paperless.example.net/
getent ahosts paperless.example.net
docker compose port webserver 8000</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) vérifie l’accessibilité TCP sous Windows ; [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) affiche les écouteurs locaux ; [`curl`](https://curl.se/docs/manpage.html) rend visibles le statut, la redirection et les en-têtes ; [`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) ou [`getent`](https://man7.org/linux/man-pages/man1/getent.1.html) vérifient la résolution de nom ; [`docker compose port`](https://docs.docker.com/reference/cli/docker/compose/port/) affiche le mappage réel des ports.

## Identités, droits globaux et droits sur les objets

Paperless-ngx utilise le modèle d’utilisateurs et de groupes Django. Les autorisations globales contrôlent quels types d’objets et quelles zones UI/API un compte peut utiliser. Les droits liés aux objets contrôlent Owner, View et Change pour un document, un tag ou un autre objet donné. La [documentation des autorisations](https://docs.paperless-ngx.com/usage/#permissions) souligne que les droits d’un tag ne sont pas automatiquement transmis aux documents qui le portent.

Les superutilisateurs peuvent voir tous les objets. Le statut Staff/Admin doit en être distingué et permet notamment l’accès aux vues backend et système. Pour l’exploitation quotidienne, des comptes et groupes normaux sont utilisés ; les comptes de service ne reçoivent que les droits API et objet nécessaires. Les workflows peuvent définir le propriétaire et les droits lors de la consommation, ce qui est particulièrement important pour les documents sans propriétaire provenant des répertoires Consume.

L’authentification peut être locale, via Reverse-Proxy-Remote-User ou par les mécanismes Social/OIDC configurés. Remote User n’est activé que derrière une limite de proxy fermée. Le cycle de vie MFA, des sessions et des tokens est intégré au concept général d’identité.

## API REST et limite d’automatisation

L’API REST est accessible sous le préfixe d’application ; une vue de schéma interactive se trouve sous `/api/schema/view/`. Sont notamment pris en charge les authentifications Basic, Session, Token, Remote-User et Headless-OIDC. La [documentation API](https://docs.paperless-ngx.com/api/) décrit la recherche, les autorisations sur les objets, les opérations en masse et les en-têtes de version.

Un token API est un secret et est attribué séparément à chaque automatisation. Les clients tiennent compte de la pagination, des filtres, des autorisations, de la version de l’API et de la répétition idempotente. Une réponse d’envoi réussie n’équivaut pas encore à un OCR terminé ; les automatisations corrèlent l’ID du document ou de la tâche jusqu’à l’état final métier. Les principes de contrat, d’authentification et de comportement de nouvelle tentative sont disponibles sous [APIs](/kb/apis).

## Récupération d’e-mails comme interface de messagerie

Les comptes et règles de messagerie relient Paperless-ngx à une boîte aux lettres IMAP. Le planificateur vérifie périodiquement par défaut les comptes configurés, applique les règles et transmet les pièces jointes correspondantes au pipeline Consume. La commande de gestion `mail_fetcher` peut déclencher manuellement la récupération ; cela est documenté dans l’[administration de la récupération des e-mails](https://docs.paperless-ngx.com/administration/#fetching-e-mail).

Pour les administrateurs de messagerie, les valeurs suivantes doivent figurer dans le runbook :

- FQDN du serveur, port et mode TLS,
- méthode d’authentification, expiration des secrets/tokens et compte de service,
- dossier IMAP, critère de recherche, types de fichier et taille maximale des messages,
- action après importation réussie ou échouée,
- gestion des messages livrés plusieurs fois et des pièces jointes identiques,
- corrélation entre Message-ID, tâche Paperless et ID du document.

La sémantique IMAP est définie par la [RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051). Une récupération IMAP réussie ne prouve que l’entrée dans le chemin de tâche ; la finalisation OCR et archivage est surveillée séparément.

## Documents Office, Tika et Gotenberg

Apache Tika extrait le contenu et les métadonnées de nombreux types de fichiers ; Gotenberg fournit des services de conversion. Paperless nécessite les deux lorsque le traitement Office/e-mail facultatif est activé. Leurs points de terminaison sont configurés séparément ([Tika Server](https://tika.apache.org/3.2.3/gettingstarted.html#Using_Tika_as_a_service), [configuration Gotenberg](https://gotenberg.dev/docs/configuration)).

Les deux services élargissent la surface d’attaque, les besoins en ressources et l’espace d’erreur. Ils ne sont rendus accessibles qu’en interne, reçoivent des limites CPU/RAM/de délai d’attente et ne sont pas publiés comme API de conversion générale. Si les PDF fonctionnent, mais que DOCX ou EML échouent systématiquement, le diagnostic commence sur ce chemin latéral et non avec Tantivy ou la base de données.

## Exploitation des conteneurs, Bare Metal et stockage

Les voies officielles sont le script d’installation, Docker Compose et Bare Metal. Compose fournit des définitions de services reproductibles et des volumes séparés ; Bare Metal donne davantage de contrôle sur les processus, les paquets et systemd, mais exige une gestion explicite du cycle de vie. La [matrice de configuration](https://docs.paperless-ngx.com/setup/#overview) classe ces voies selon l’effort et l’usage.

Les répertoires persistants ne sont pas confondus avec le cycle de vie des conteneurs. Avec Compose, les chemins Media, Data, Consume, Export et de base de données sont délibérément résolus. Les systèmes de fichiers distants ou réseau sont contrôlés quant au comportement POSIX, au renommage atomique, aux verrous, à la latence, à inotify et à la disponibilité. Une courte interruption peut sinon ressembler à une perte de documents permanente. Les principes des montages sont disponibles sous [Container](/kb/container) et [Storage](/kb/storage).

En cas d’incident, il faut d’abord déterminer dans quelle phase du pipeline se trouve le document. La file d’attente, le worker, le convertisseur externe, la base de données et le stockage de fichiers sont ensuite contrôlés de manière ciblée.

## Observabilité et diagnostic des tâches

Une vue d’exploitation utile distingue quatre signaux :

- **Santé du service :** le processus est en cours d’exécution, le port écoute, la base de données et le broker sont accessibles.
- **Débit :** nouvelles tâches, documents finalisés, durées d’exécution et profondeur de la file d’attente.
- **Erreurs :** tâches par étape et type de fichier, sortie des analyseurs/OCR, erreurs de messagerie et de stockage.
- **Intégrité du fonds :** originaux/archives/miniatures manquants ou endommagés, fichiers orphelins, dérive de l’index.

Le Sanity Checker intégré détecte notamment les fichiers manquants, illisibles ou modifiés par rapport à la somme de contrôle enregistrée, ainsi que les fichiers Media orphelins. La liste complète figure dans la [documentation du Sanity Checker](https://docs.paperless-ngx.com/administration/#sanity-checker). Une alerte sur le nombre d’erreurs sans ID de document/de tâche ni phase de pipeline est difficilement exploitable.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Paperless-Task- und Storage-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker compose logs --since 1h --timestamps webserver
docker compose exec webserver document_sanity_checker
docker compose exec webserver mail_fetcher
docker stats --no-stream
docker system df -v</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker compose logs --since 1h --timestamps webserver
docker compose exec webserver document_sanity_checker
docker compose exec webserver mail_fetcher
docker stats --no-stream
docker system df -v</code></pre>
  </div>
</div>

[`docker stats`](https://docs.docker.com/reference/cli/docker/container/stats/) affiche l’utilisation instantanée du CPU/de la mémoire, [`docker system df`](https://docs.docker.com/reference/cli/docker/system/df/) l’occupation des disques Docker. Ces deux éléments sont des instantanés. Pour déterminer les causes, il faut corréler la série temporelle, l’ID de tâche et les journaux Paperless. La méthodologie générale est disponible sous [Troubleshooting](/kb/troubleshooting).

## Capacité, parallélisme et limites

La planification de capacité commence par le nombre de pages par jour, les formats entrants, la taille moyenne et maximale des fichiers, les langues OCR, les tâches parallèles, la durée de conservation et le volume de recherche. Un document peut générer simultanément un original, un PDF d’archive, une miniature, un enregistrement de base de données, un index plein texte et des fichiers de rendu temporaires. L’espace libre doit également tenir compte de la réindexation, de l’exportation et des mises à niveau.

Un parallélisme accru des workers n’accélère l’OCR lié au CPU que jusqu’à saturation du CPU, de la RAM et des E/S. Il augmente simultanément le nombre de fichiers temporaires et de connexions à la base de données. La récupération des e-mails et le Consumer peuvent générer davantage de tâches que les workers ne peuvent en traiter ; l’âge de la file d’attente est donc surveillé plutôt que sa seule longueur instantanée. Les limites sont testées avec des documents représentatifs, et non avec de petits PDF texte.

L’étendue de la sauvegarde découle du modèle de données : les originaux, archives, la base de données, la configuration, le Secret Key et, le cas échéant, des services supplémentaires doivent pouvoir être restaurés ensemble.

## Sauvegarde : quels objets sont sauvegardés ensemble

La [documentation de sauvegarde](https://docs.paperless-ngx.com/administration/#making-backups) officielle indique deux voies principales : le Document Exporter ou une sauvegarde coordonnée des volumes/répertoires et de la base de données. L’exportateur écrit les documents, miniatures, métadonnées et le contenu de la base de données dans un répertoire transportable. Les tokens API ne sont pas repris ; l’exportation et l’importation devraient être compatibles en version.

Un jeu de restauration complet contient :

| Objet | Pourquoi nécessaire | Règle de cohérence |
|---|---|---|
| Base de données | ID, métadonnées, utilisateurs, règles, références de tâches | cohérente avec le snapshot Media |
| Media | originaux, archives, miniatures | aucune écriture inachevée |
| Data | index et états auxiliaires | partiellement dérivable, mais utile pour un redémarrage rapide |
| Configuration | chemins, URL, OCR, messagerie, proxy, base de données | même contrat de nommage et de montage |
| Secrets | Secret Key, identifiants DB/messagerie/OIDC | protégés, séparés et restaurables |
| Manifeste d’export | affectation portable des objets et fichiers | à conserver avec les fichiers exportés |
| Runbooks | ordre, version, tests de réception | disponibles hors ligne |

L’exportateur peut mettre à jour les exportations existantes et comparer facultativement les sommes de contrôle. Cela économise du temps, mais ne remplace pas une copie distincte et versionnée : avec `--delete`, les documents supprimés peuvent être retirés de la cible d’exportation. Les détails et les options sont documentés par le [`document_exporter`](https://docs.paperless-ngx.com/administration/#document-exporter).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Paperless-Export und Recoveryprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker compose exec -T webserver document_exporter ../export --compare-checksums
Get-ChildItem -Recurse .\export | Measure-Object -Property Length -Sum
Get-FileHash .\export\manifest.json -Algorithm SHA256
docker compose exec webserver document_sanity_checker</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker compose exec -T webserver document_exporter ../export --compare-checksums
du -sh ./export
sha256sum ./export/manifest.json
docker compose exec webserver document_sanity_checker</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash) et [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) garantissent la preuve de transfert du manifeste. Une somme de contrôle prouve seulement l’égalité de deux ensembles d’octets, et non l’importabilité ou l’exhaustivité métier. Les principes de sauvegarde sont disponibles sous [Sauvegarde et reprise après sinistre](/kb/backup-dr).

## Restauration et réception

Le [`document_importer`](https://docs.paperless-ngx.com/administration/#document-importer) attend une installation vide ou, avec `--data-only`, une base de données vide. Un test de restauration utilise donc des chemins de destination isolés, une version Paperless correspondant à la sauvegarde et aucun volume de production.

La réception comprend au minimum :

1. Déployer la configuration et les secrets dans l’environnement cible.
2. Fournir une base de données vide et des chemins Media/Data vides.
3. Importer l’exportation ou restaurer la base de données et les fichiers de manière cohérente.
4. Exécuter de manière contrôlée les migrations et, le cas échéant, la construction de l’index.
5. Vérifier la connexion utilisateur, les groupes et les autorisations.
6. Comparer le nombre de documents, les métadonnées et des échantillons des originaux et archives.
7. Tester la recherche plein texte et l’affichage des miniatures.
8. Exécuter le Sanity Checker sans résultats inexpliqués.
9. Consommer un nouveau document de test par l’entrée réelle.
10. Documenter le RPO et le RTO avec des horodatages.

Une restauration n’est considérée comme réussie qu’après ce test de bout en bout. « Le conteneur démarre » n’est pas un critère de réception.

## Mises à jour, migrations de base de données et limite de rollback

Avant une mise à jour, les tâches Consume actives sont terminées, une sauvegarde est créée et les notes de version sont examinées pour détecter les breaking changes. Le [guide de mise à jour](https://docs.paperless-ngx.com/administration/#updating-paperless) distingue les procédures Docker et Bare Metal et décrit la migration de la base de données ainsi que la réindexation selon les besoins. Une image est épinglée sur une série validée ou un digest ; `latest` n’est pas un point de rollback reproductible.

Après une migration de base de données, un rollback est normalement une restauration de la base de données et du fonds de fichiers correspondant, et non le simple démarrage de l’ancienne image. Les nouvelles archives ou métadonnées créées après le point de sauvegarde doivent être traitées sur le plan métier avant le retour en arrière. Les principes généraux de release et de changement sont disponibles sous [Releases](/kb/releases) et [Migration](/kb/migration).

Le [changelog](https://docs.paperless-ngx.com/changelog/) est la source dépendante du temps pour les nouvelles fonctionnalités, les breaking changes et les corrections. En revanche, cet article décrit des principes stables d’architecture et d’exploitation.

## Scénarios d’erreur par phase de pipeline

Parce qu’un document traverse plusieurs étapes de traitement, le symptôme visible ne montre souvent que la dernière phase réussie. Le tableau mène du symptôme à la première preuve fiable.

| Symptôme | Limite probable | Premières preuves |
|---|---|---|
| Le fichier reste dans le répertoire Consume | watcher, polling, UID/GID, fin d’écriture | montage, droits, journal du Consumer |
| L’envoi répond, mais aucun document n’est créé | broker ou worker de tâches | ID de tâche, PING du broker, journal du worker |
| Seuls DOCX/EML échouent | Tika/Gotenberg | point de terminaison, délai d’attente, journal du convertisseur |
| OCR très lent ou processus arrêté | CPU/RAM, nombre de pages, paquets linguistiques | statistiques du conteneur, événement kernel/runtime, taille des pages |
| Document visible, plein texte absent | analyseur/OCR ou index de recherche | texte extrait, journal de tâche, statut de réindexation |
| Miniature absente, original présent | fichier dérivé | Sanity Checker, commande de miniature |
| Login/POST défaillant derrière le proxy | URL, Host, CSRF, en-tête Forwarded | en-têtes HTTP, configuration effective |
| Récupération périodique des e-mails arrêtée | planificateur, broker, authentification de messagerie | journal du planificateur, `mail_fetcher`, test IMAP |
| Fichiers disparus après redémarrage | montage erroné ou volatil | `docker inspect`, inventaire des volumes |
| La restauration n’importe pas | condition de version, de chemin ou de vide | manifeste d’exportation, version cible, journal d’importation |

La [collection de dépannage](https://docs.paperless-ngx.com/troubleshooting/) officielle complète ces informations par des messages d’erreur concrets. Avant toute modification, les administrateurs conservent toujours l’ID de tâche, le nom du document, la fenêtre temporelle, la configuration effective, l’état du conteneur/processus et l’état de tous les services dépendants.

## Histoire technique

Le projet Paperless d’origine a établi le modèle de base reposant sur un répertoire Consume, l’OCR et une archive consultable. Paperless-ng a poursuivi le développement en tant que fork. Paperless-ngx est ensuite devenu le successeur communautaire officiel afin de répartir la maintenance et la responsabilité entre plusieurs contributeurs ; cette filiation est décrite dans le [README du projet](https://github.com/paperless-ngx/paperless-ngx#paperless-ngx).

La plateforme est passée d’une application Django principalement côté serveur à un DMS avec frontend Angular, API REST, autorisations liées aux objets, moteur de workflows et traitement parallèle des tâches. OCRmyPDF/Tesseract sont restés la base OCR ; Tika et Gotenberg ont étendu les formats d’entrée pris en charge. Le chemin de recherche est passé de Whoosh à Tantivy, ce qui a rendu visible l’index comme domaine distinct de migration et d’exploitation ([migration Tantivy](https://docs.paperless-ngx.com/migration-v3/)).

Cette histoire explique l’architecture actuelle : un document Paperless-ngx est à la fois un objet relationnel, un ensemble de fichiers gérés, un résultat OCR et une projection de recherche. La maintenance et la reprise doivent tenir compte de ces quatre perspectives.

## Checklist administrateur

Pour finir, les entrées, workers, stockage, recherche et sauvegarde sont vérifiés ensemble. La checklist aide à ne pas oublier une dépendance externe hors du conteneur Paperless.

- **Inventaire :** image/commit, application Django/Angular, workers, planificateur, broker, base de données, Tika/Gotenberg, proxy et montages.
- **Objets de données :** nommer séparément base de données, original, archive, miniature, index, configuration, secrets et manifeste d’exportation.
- **Entrées :** documenter Consume, upload/API et e-mail avec les règles de taille, de type, d’authentification et d’erreur.
- **Pipeline :** corréler l’ID de tâche à travers l’analyseur, l’OCR, l’archivage, les métadonnées et l’index jusqu’à la finalisation.
- **Autorisations :** tester les droits globaux et liés aux objets, les superutilisateurs, groupes et attributions de workflow.
- **Capacité :** surveiller l’âge de la file d’attente, la durée OCR, l’espace temporaire, la croissance Media/index et les connexions de base de données.
- **Sécurité :** intégrer HTTPS, la confiance dans le proxy, le Secret Key, les tokens, le stockage en clair et les analyseurs externes au modèle de menace.
- **Sauvegarde :** sauvegarder de manière cohérente la base de données et Media ; ne pas oublier la configuration et les secrets.
- **Restauration :** vérifier l’instance cible vide, la version correspondante, les échantillons, la recherche, le Sanity Checker et une nouvelle exécution Consume.
- **Changement :** consigner à l’avance les notes de version, la migration du schéma/de l’index, l’espace libre et la véritable limite de rollback.

## Sources

- [Paperless-ngx – Documentation](https://docs.paperless-ngx.com/)
- [Paperless-ngx – dépôt GitHub](https://github.com/paperless-ngx/paperless-ngx)
- [Paperless-ngx – Développement](https://docs.paperless-ngx.com/development/)
- [Paperless-ngx – Utilisation](https://docs.paperless-ngx.com/usage/)
- [Celery](https://docs.celeryq.dev/en/stable/)
- [Paperless-ngx – manifeste de projet Python](https://github.com/paperless-ngx/paperless-ngx/blob/dev/pyproject.toml)
- [Paperless-ngx – Dockerfile](https://github.com/paperless-ngx/paperless-ngx/blob/dev/Dockerfile)
- [Paperless-ngx – Configuration](https://docs.paperless-ngx.com/configuration/)
- [broker compatible Redis](https://redis.io/docs/latest/)
- [PostgreSQL](https://www.postgresql.org/docs/current/)
- [Paperless-ngx – Installation](https://docs.paperless-ngx.com/setup/)
- [Docker – compose config](https://docs.docker.com/reference/cli/docker/compose/config/)
- [Docker – compose ps](https://docs.docker.com/reference/cli/docker/compose/ps/)
- [Docker – compose images](https://docs.docker.com/reference/cli/docker/compose/images/)
- [Docker – inspect](https://docs.docker.com/reference/cli/docker/inspect/)
- [Paperless-ngx – FAQ](https://docs.paperless-ngx.com/faq/)
- [Paperless-ngx – Consume Hooks](https://docs.paperless-ngx.com/advanced_usage/)
- [OCRmyPDF – Documentation](https://ocrmypdf.readthedocs.io/en/latest/)
- [Tesseract – Documentation](https://tesseract-ocr.github.io/tessdoc/)
- [Paperless-ngx – Document Renamer](https://docs.paperless-ngx.com/administration/)
- [Paperless-ngx – migration Tantivy](https://docs.paperless-ngx.com/migration-v3/)
- [Docker – compose exec](https://docs.docker.com/reference/cli/docker/compose/exec/)
- [PostgreSQL – Backup et Restore](https://www.postgresql.org/docs/current/backup.html)
- [MariaDB – Backup et Restore](https://mariadb.com/docs/server/server-usage/backing-up-and-restoring-databases/)
- [PostgreSQL – pg_isready](https://www.postgresql.org/docs/current/app-pg-isready.html)
- [Redis – redis-cli](https://redis.io/docs/latest/develop/tools/cli/)
- [Docker – compose logs](https://docs.docker.com/reference/cli/docker/compose/logs/)
- [Microsoft – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [Linux man-pages – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html)
- [curl – Manuel](https://curl.se/docs/manpage.html)
- [Microsoft – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [Linux man-pages – getent(1)](https://man7.org/linux/man-pages/man1/getent.1.html)
- [Docker – compose port](https://docs.docker.com/reference/cli/docker/compose/port/)
- [Paperless-ngx – API REST](https://docs.paperless-ngx.com/api/)
- [RFC 9051 – IMAP4rev2](https://datatracker.ietf.org/doc/html/rfc9051)
- [Apache Tika – exploitation du serveur](https://tika.apache.org/3.2.3/gettingstarted.html)
- [Gotenberg – Configuration](https://gotenberg.dev/docs/configuration)
- [Docker – stats](https://docs.docker.com/reference/cli/docker/container/stats/)
- [Docker – system df](https://docs.docker.com/reference/cli/docker/system/df/)
- [Microsoft – Get-FileHash](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – utilitaires sha2](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Paperless-ngx – Changelog](https://docs.paperless-ngx.com/changelog/)
- [Paperless-ngx – Troubleshooting](https://docs.paperless-ngx.com/troubleshooting/)
