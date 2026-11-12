---
title: "Rclone 1.76.0 : nombreuses corrections de sécurité, correctifs VFS et nouveaux backends"
navTitle: "Rclone 1.76.0"
description: "Rclone 1.76.0 du 6 novembre 2026 comble des failles de sécurité dans serve, rc et plusieurs backends, dont une faille de proxy d’authentification dans serve sftp avec un CVSS de 8.1, corrige cinq erreurs dans la couche VFS et bloque par défaut les redirections HTTPS vers HTTP. Les opérateurs de montages, de services serve et de rc devraient mettre à jour et vérifier les valeurs par défaut modifiées."
date: "2026-11-11"
kategorie: "Rclone"
timeToRead: "7 min de lecture"
themen:
  - rclone
produkte:
  - "rclone"
protokolle:
  - "releases"
  - "storage"
slug: "rclone-1-76-0-nombreuses-corrections-de-securite-correctifs-vfs-et-nouveaux-backends"
translationId: article-aa7be1fdff77b8cc
translationOf: rclone-1-76-0
url: https://rafaelpfister.ch/fr/blog/rclone-1-76-0-nombreuses-corrections-de-securite-correctifs-vfs-et-nouveaux-backends
translationSourceHash: 79bd08de9bbf74eb195b3bdc4093b8e92f84cd358eb2a4b0a950ad3109a08e56
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:29:14.150Z
translationReview: automatic
---

# Rclone 1.76.0 : nombreuses corrections de sécurité, correctifs VFS et nouveaux backends

Rclone 1.76.0 est paru le 6 novembre 2026, environ quatre semaines après la version 1.75.2 (9 octobre 2026). Le journal des modifications compte vingt entrées dans la section Security, dont une faille dans le proxy d’authentification de `rclone serve sftp` avec un CVSS de 8.1, des fuites d’identifiants lors de redirections dans les backends Swift, NetStorage, Seafile et Ulozto, ainsi que plusieurs sorties via symlink dans le backend `local`. S’y ajoutent cinq corrections dans la couche VFS, qui prend en charge `rclone mount` et les commandes serve, des extensions du backend S3, deux nouveaux backends (Dosya et Infomaniak kDrive) et la nouvelle commande `index`. Certaines valeurs par défaut changent : les redirections HTTPS vers HTTP sont refusées, les écouteurs TCP multipath sont désactivés et les motifs glob dans les filtres sont évalués plus strictement. Cette version s’adresse à toutes les personnes qui exploitent Rclone pour des montages, des synchronisations, des sauvegardes ou comme serveur avec `rclone serve` et `rclone rcd`. Le contexte des montages en conteneur est présenté dans l’article [Exploiter de manière fiable des montages Rclone dans Docker](/blog/rclone-mount-in-docker-container).

## Aperçu des versions

| Version | Date | Type | Points clés |
|---|---|---|---|
| 1.76.0 | 06.11.2026 | Version fonctionnelle | Corrections de sécurité dans serve sftp (CVSS 8.1), rc, lib/http, local et quatre backends ; Go 1.27.3 ; cinq correctifs VFS ; protection contre les redirections `--allow-https-downgrade` ; nouveaux backends Dosya et Infomaniak kDrive ; commande `index` |

La version la plus élevée jusqu’alors était 1.75.2. Aucune version corrective 1.76.x n’avait été publiée au 11 novembre 2026.

## Corrections de sécurité avec advisory

Le projet Rclone a publié les failles suivantes le 6 novembre 2026 sous forme de GitHub Security Advisories. Toutes indiquent 1.76.0 comme version corrigée ; aucune ne possède d’identifiant CVE (état au 11 novembre 2026, marqué « CVE-PENDING » dans le journal des modifications).

- **GHSA-66qv-66mg-9h74, CVSS 8.1 (High) :** Avec `rclone serve sftp` et `--auth-proxy`, la clé de cache du proxy d’authentification ne contenait pas le type d’authentification. Une clé publique connue pouvait ainsi être réutilisée comme mot de passe si le proxy d’authentification traite différemment les connexions par clé et par mot de passe, que l’attaquant et la victime utilisent le même nom d’utilisateur et la même adresse IP cliente, et que l’entrée de cache est encore valide (300 secondes d’inactivité par défaut). Des adresses IP clientes identiques peuvent par exemple survenir derrière un NAT, un hôte bastion ou un équilibreur de charge. Selon l’advisory, les autres protocoles serve ne sont pas concernés. Le champ relatif aux versions affectées indique « >= 1.51.0, <= 1.76.0 », celui de la version corrigée indique 1.76.0 ; l’entrée de correction du journal des modifications de 1.76.0 fait foi.
- **GHSA-vmgx-g9c5-669p, CVSS 4.2 :** Avec `serve sftp`, `serve ftp`, `serve http` et `serve webdav` utilisant `--auth-proxy`, une révocation dans le proxy d’authentification restait sans effet tant qu’une réponse était en cache. À partir de 1.76.0, une révocation prend effet dans les cinq minutes. `serve s3` n’est pas concerné.
- **GHSA-hwj7-wp9g-wrv9, CVSS 5.3 :** Le temps de réponse de la vérification bcrypt permettait de distinguer des noms d’utilisateur valides de noms invalides avec `--htpasswd` et `--user`. Sont notamment concernés `serve http`, `serve webdav`, `serve restic`, `serve s3`, `rcd` avec `--rc-htpasswd` et le point de terminaison Metrics avec `--metrics-htpasswd`, dans les versions 1.56.0 à 1.75.2. Les mots de passe ne sont pas divulgués.
- **GHSA-f8p4-8g4j-q3m2, CVSS 5.5 :** Si le serveur rc fonctionnait sans authentification HTTP, un client ayant accès au port pouvait épuiser la mémoire du processus au moyen d’une grande requête JSON vers une méthode non autorisée (déni de service, versions 1.46.0 à 1.75.2). La version 1.76.0 limite la taille des requêtes et refuse les appels non autorisés avant de lire le corps. L’advisory recommande comme mesure de contournement une authentification HTTP pour rc et une limite de taille du corps au niveau du proxy inverse ; selon l’advisory, la liaison à localhost ne sépare pas les comptes utilisateurs locaux les uns des autres.
- **GHSA-m97j-7x65-wh93, CVSS 4.8 :** Les backends Swift, NetStorage et Seafile transmettaient les identifiants lors de redirections vers un autre hôte ou de HTTPS vers HTTP. L’exploitation n’est possible que si le point de terminaison configuré effectue lui-même une redirection, par exemple via un proxy. L’advisory recommande comme mesure de contournement d’utiliser exclusivement des points de terminaison `https://` et aucun proxy qui rétrograde vers HTTP.
- **GHSA-3xpx-hjg2-vrjh, CVSS 4.8 :** Le backend Ulozto transmettait les en-têtes `X-Auth-Token` et `X-User-Token` lors de redirections de liens de téléchargement. Pour l’exploiter, un attaquant doit disposer d’une position réseau lui permettant de rediriger le trafic CDN.
- **GHSA-r5wv-vqqh-2r9c, CVSS 4.0 :** Avec `--links` et `--metadata`, le backend `local` pouvait, via un symlink injecté, écrire des attributs `user.*` en dehors du répertoire cible. Jusqu’à la mise à jour, l’advisory recommande de ne pas utiliser ces deux options conjointement lorsque des tiers peuvent influencer la cible.

En outre, la version 1.76.0 corrige dans le backend `local` d’autres cas sans advisory propre, dans lesquels `MkdirMetadata`, `Remove`, `Rmdir` et `DirMove` pouvaient agir via des symlinks en dehors du répertoire racine.

## Dépendances et valeurs par défaut modifiées

Rclone 1.76.0 est compilé avec Go 1.27.3. Le journal des modifications mentionne à ce sujet CVE-2026-111943 : Go ajoute dans `crypto/x509` une liste de blocage pour les certificats intermédiaires compromis (CVSS 6.8 selon l’évaluation CISA-ADP, Go affecté avant 1.26.10 et de 1.27.0 jusqu’avant 1.27.3). En outre, `golang.org/x/net` a été mis à jour vers v0.62.0 et `github.com/andybalholm/brotli` vers v1.2.7, ce dernier en raison de CVE-2026-111497 ; le contexte de cette faille est présenté dans l’article sur [CVE-2026-111497 dans le décodeur Brotli](/blog/brotli-cve-2026-111497).

Les écouteurs TCP multipath sont désactivés par défaut à partir de 1.76.0. Le journal des modifications justifie cela par CVE-2026-98432 ; l’entrée CVE décrit un use-after-free dans la pile TCP du noyau Linux (CVSS 9.8, corrigé entre autres dans 6.1.190, 6.6.159, 6.12.113 et 6.18.56). Les mises à jour du noyau de la distribution fournissent elles-mêmes la correction ; Rclone réduit uniquement sa propre surface d’attaque. Le journal des modifications ne mentionne aucun indicateur pour les réactiver.

Aperçu des modifications du comportement par défaut :

- **Redirections :** Rclone refuse les redirections de HTTPS vers HTTP. Le nouveau flag `--allow-https-downgrade` permet à nouveau de les autoriser.
- **Filtres :** `[!...]` dans les motifs glob est désormais une classe de caractères négative, `[^...]` exclut `/`. Vérifiez les fichiers de filtres et les règles `--include`/`--exclude` utilisant de telles classes avec `--dry-run` avant la mise en production.
- **local :** `--copy-links` ne suit plus les boucles de symlinks. Sans privilèges root, Rclone ne définit plus que les droits de propriété qu’un utilisateur normal est autorisé à définir. Un manque d’espace disque interrompt désormais les transferts multithread avec une erreur fatale.
- **Validation des paramètres :** Les tailles trop élevées, NaN et les durées provoquant un dépassement sont refusés, de même que les indications de temps `--bwlimit` sans deux-points ou avec un signe.
- **bisync :** `--max-delete` prend en compte `--track-renames`; les valeurs supérieures à 100 sont refusées.
- **operations :** `--immutable` s’applique désormais également à `copyto`, `moveto` et à la copie de fichiers individuels.
- **Obsolète :** `--rc-web-gui` ainsi que, pour `serve s3`, l’option `--no-cleanup`, qui est désormais un alias du nouveau `--directory-markers`.

Après la mise à jour, vous devriez exécuter une fois les scripts et unités qui appellent Rclone avec des paramètres fixes et vérifier les messages d’erreur concernant des valeurs refusées.

## Montage et cache VFS

Pour `rclone mount`, le journal des modifications indique une correction : les instances VFS n’étaient pas libérées lors d’échecs de montage ni lors du démontage. Dans la couche VFS, utilisée en commun par Mount et les commandes serve, la version 1.76.0 corrige cinq erreurs :

- Un truncate réduisant la taille d’un fichier non mis en cache était perdu.
- Le renommage d’un répertoire pendant l’expiration du cache de répertoires provoquait un plantage.
- Des entrées virtuelles persistaient dans des répertoires supprimés.
- Après un renommage qui ne différait que par la casse ou la normalisation Unicode, des entrées apparaissaient en double.
- Les téléversements d’un fichier modifié ne démarraient pas tant que le fichier continuait à être lu.

Le dernier point concerne les montages avec cache VFS, sur lesquels un fichier modifié continue à être lu alors qu’il doit être téléversé. Dans ces configurations, surveillez après la mise à jour la file d’attente des téléversements et effectuez un test d’écriture et de lecture, comme décrit dans le [plan de test pour les montages Rclone](/blog/cloud-mount-testen-dummy-pdfs).

## Backend S3 et Proton Drive

Dans le backend S3, Rclone ne met plus en mémoire tampon les téléversements avec Object Lock lorsque la valeur MD5 de la source est connue. La nouvelle commande de backend `link` permet des overrides signés pour les en-têtes de réponse. S’ajoutent les régions Fastly EU Central 1 et Scaleway IT-MIL, plusieurs régions Huawei OBS ainsi que les fournisseurs Autonomys Auto Drive, Clever Cloud Cellar, Infomaniak Swiss Backup, Neon Object Storage et RelAix.Storage. Les téléversements multithread calculent les sommes de contrôle par lectures de 1 MiB et, après une restauration, la classe de stockage réelle est affichée. Ont été corrigés `rmdir` au niveau racine avec `--s3-directory-markers` ainsi que les URL présignées pour GCS, Linode et Ceph.

Pour `rclone serve s3`, la version 1.76.0 apporte le flag `--directory-markers` ainsi que des modifications concernant les tags, ETags, ETags multipart et écritures conditionnelles. Il est également corrigé que les buckets ne pouvaient pas être supprimés après un téléversement multipart interrompu.

Le journal des modifications de 1.76.0 ne contient aucune entrée concernant le backend Proton Drive. Les dernières corrections à ce sujet remontent à 1.75.0 (logique de nouvelle tentative, blocages lors d’erreurs de validation, heures de modification incorrectes) et à 1.75.1 (les fichiers téléversés avec 1.75.0 ne pouvaient pas être lus dans les applications Proton). L’état de la prise en charge de Proton Drive sous Linux est décrit dans l’article [Proton Drive sous Linux](/blog/proton-drive-linux-status).

## Autres nouveautés

Les deux nouveaux backends sont Dosya (Kaya) et Infomaniak kDrive. La nouvelle commande `index` écrit des listes de répertoires statiques dans un remote et peut, lors des exécutions suivantes, ne réindexer que les répertoires modifiés. Rclone prend en charge CRC-32C comme hash, `sync` peut contrôler l’ordre des transferts selon des motifs, et la vue des répertoires de `serve http` et `serve webdav` est triée et remaniée. Selon le journal des modifications, l’interface graphique n’est plus considérée comme bêta. Les statistiques ne comptent plus `totalBytes` deux fois lors de copies côté serveur, et `serve sftp` renvoie une erreur lors de `rmdir` sur un fichier au lieu de supprimer ce fichier.

## Procédure recommandée

La version cible est 1.76.0. La mise à jour est particulièrement urgente sur les systèmes avec `rclone serve sftp --auth-proxy`, avec `rclone rcd` ou `--rc` sans authentification HTTP, avec des services serve protégés par `--htpasswd`, ainsi qu’avec des remotes de type Swift, NetStorage, Seafile ou Ulozto. Pour les installations à partir des binaires officiels, `rclone selfupdate` met à jour le fichier exécutable et vérifie le téléchargement au moyen d’un hash et d’une signature ; avec `rclone selfupdate --check`, Rclone indique au préalable quelle version serait installée. Pour les installations à partir de paquets deb ou rpm, la documentation mentionne `--package deb` ou `--package rpm`. Les montages et processus serve en cours n’utilisent la nouvelle version qu’après un redémarrage. Vérifiez ensuite les règles de filtrage avec classes de caractères, les planifications `--bwlimit` et les configurations qui dépendent de redirections HTTP. Le journal des modifications et la page de publication ne mentionnent aucun problème connu pour 1.76.0 (état au 11 novembre 2026).

## Sources

1.  [Rclone : Changelog](https://rclone.org/changelog/) : entrées relatives à 1.76.0 (6 novembre 2026), ainsi qu’à 1.75.0 à 1.75.2 avec corrections de sécurité et modifications de Mount, VFS, S3 et Proton Drive.

2.  [GitHub : rclone v1.76.0](https://github.com/rclone/rclone/releases/tag/v1.76.0) : page de publication avec binaires et renvoi vers le journal des modifications.

3.  [GitHub : Security Advisories de rclone](https://github.com/rclone/rclone/security/advisories) : aperçu des advisories publiés le 6 novembre 2026.

4.  [GHSA-66qv-66mg-9h74](https://github.com/rclone/rclone/security/advisories/GHSA-66qv-66mg-9h74) : serve sftp, cache du proxy d’authentification sans type d’authentification, CVSS 8.1.

5.  [GHSA-vmgx-g9c5-669p](https://github.com/rclone/rclone/security/advisories/GHSA-vmgx-g9c5-669p) : la révocation dans le proxy d’authentification reste sans effet avec une réponse en cache, CVSS 4.2.

6.  [GHSA-hwj7-wp9g-wrv9](https://github.com/rclone/rclone/security/advisories/GHSA-hwj7-wp9g-wrv9) : détection de noms d’utilisateur via le timing bcrypt avec `--htpasswd`, CVSS 5.3.

7.  [GHSA-f8p4-8g4j-q3m2](https://github.com/rclone/rclone/security/advisories/GHSA-f8p4-8g4j-q3m2) : épuisement de la mémoire du serveur rc par des requêtes JSON trop volumineuses, CVSS 5.5.

8.  [GHSA-m97j-7x65-wh93](https://github.com/rclone/rclone/security/advisories/GHSA-m97j-7x65-wh93) : fuites d’identifiants lors de redirections dans Swift, NetStorage et Seafile, CVSS 4.8.

9.  [GHSA-3xpx-hjg2-vrjh](https://github.com/rclone/rclone/security/advisories/GHSA-3xpx-hjg2-vrjh) : fuite de token lors de redirections dans le backend Ulozto, CVSS 4.8.

10.  [GHSA-r5wv-vqqh-2r9c](https://github.com/rclone/rclone/security/advisories/GHSA-r5wv-vqqh-2r9c) : sortie via symlink avec `--links --metadata` dans le backend local, CVSS 4.0.

11.  [CVE.org : CVE-2026-111943](https://www.cve.org/CVERecord?id=CVE-2026-111943) : liste de blocage pour les certificats intermédiaires compromis dans Go `crypto/x509`, corrigé dans Go 1.27.3.

12.  [CVE.org : CVE-2026-98432](https://www.cve.org/CVERecord?id=CVE-2026-98432) : use-after-free dans la pile TCP du noyau Linux, CVSS 9.8, motif de la désactivation des écouteurs TCP multipath.

13.  [Rclone : rclone selfupdate](https://rclone.org/commands/rclone_selfupdate/) : documentation sur l’auto-mise à jour, `--check` et `--package`.

14.  [CVE-2026-111497 : faille dans le décodeur Brotli](/blog/brotli-cve-2026-111497) : contexte de la faille Brotli traitée par 1.76.0 via la mise à jour de dépendance.
