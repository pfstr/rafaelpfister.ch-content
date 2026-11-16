---
title: "Pi-hole 2026.11.0 : FTL 6.9 et Core 6.6 corrigent onze failles de sécurité"
navTitle: "Pi-hole 2026.11.0"
description: "L’image Docker Pi-hole 2026.11.0 regroupe FTL v6.9 et Core v6.6 et corrige notamment une exécution de code par un administrateur via files.gravity (CVSS 7.2), ainsi que plusieurs failles de liens symboliques dans l’installateur. Les filtres regex, webserver.acl et l’API des sessions se comportent en partie différemment ; vérifiez ces points après la mise à jour."
date: "2026-11-12"
kategorie: "Réseau et DNS"
timeToRead: "7 min de lecture"
themen:
  - smart-home-iot
produkte:
  - "pihole"
protokolle:
  - "releases"
  - "dns"
  - "container"
slug: "pi-hole-2026-11-0-ftl-6-9-et-core-6-6-corrigent-onze-failles-de-securite"
translationId: article-0eb045e8f65bedd0
translationOf: pihole-2026-11-0
url: https://rafaelpfister.ch/fr/blog/pi-hole-2026-11-0-ftl-6-9-et-core-6-6-corrigent-onze-failles-de-securite
translationSourceHash: 44a4448ed57216f76890c2807ea763f7c340fcdc0fffa70b8e8e50816c28f4c4
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T13:42:00.803Z
translationReview: automatic
---

# Pi-hole 2026.11.0 : FTL 6.9 et Core 6.6 corrigent onze failles de sécurité

Pi-hole a publié l’image Docker 2026.11.0 le 11 novembre 2026. Elle contient Pi-hole FTL v6.9 et Pi-hole Core v6.6, publiés le même jour ; selon le fabricant, le Web Interface reste en v6.6.1. Les deux versions de composants corrigent onze avis de sécurité, dont quatre de sévérité High (CVSS 7.2 à 7.3). FTL v6.9 met également à jour dnsmasq, SQLite et le moteur regex TRE, et modifie le comportement des filtres regex, de `webserver.acl` et de certaines parties de l’API. Cette version s’adresse à tous les exploitants de Pi-hole, aussi bien dans un conteneur Docker qu’en installation Bare Metal. Selon les notes de version, la précédente version 2026.10.1 du 31 octobre 2026 apportait surtout une image de base Alpine actualisée (3.24.3).

## Aperçu des versions

| Version | Date | Type | Points clés |
|---|---|---|---|
| Docker 2026.11.0 | 11.11.2026 | Publication d’image | regroupe FTL v6.9, Core v6.6 et Web v6.6.1 ; `setpriv` au lieu de `sudo`, logrotate via la Crontab du conteneur |
| FTL v6.9 | 11.11.2026 | Version mineure | sept avis (dont un High), dnsmasq v2.93+42, SQLite 3.54.1, mise à jour de TRE modifiant le comportement des regex |
| Core v6.6 | 11.11.2026 | Version mineure | cinq avis (quatre High, un Medium), l’installateur n’écrit plus dans `/etc/pihole` qu’en tant qu’utilisateur `pihole` |
| Web v6.6.1 | 17.10.2026 | inchangé | aucune nouvelle version dans cette publication |

## Correctifs de sécurité de sévérité High

Les avis sont publiés depuis le 11 novembre 2026. Aucun identifiant CVE n’avait été attribué aux avis examinés à cette date ; selon le texte de l’avis concernant `files.gravity`, une demande est en cours (état au 12 novembre 2026).

- **GHSA-pj9m-9623-2g75 (FTL) et GHSA-cq9w-mv93-mxh7 (Core), CVSS 7.2 :** exécution de code par un administrateur authentifié (CWE-78, CWE-89). FTL 6.8 avait assoupli la vérification du chemin afin que `files.gravity` puisse contenir une apostrophe. `gravity.sh` insérait cette valeur sans protection dans une instruction SQL `ATTACH` ; le shell sqlite3 de FTL permettait ainsi d’exécuter des commandes du système d’exploitation en tant qu’utilisateur `pihole`, dans les installations Docker comme Bare Metal selon l’avis. Sont affectés Core jusqu’à 6.5 et FTL 6.8. Core échappe désormais la valeur, tandis que FTL lie le chemin comme paramètre dans le Teleporter. Le fabricant souligne que les deux paquets doivent être mis à jour ensemble.
- **GHSA-wp3p-45h3-2v82 (Core), CVSS 7.3 :** correction incomplète des failles de liens symboliques d’octobre (CWE-59, CWE-61). `pihole -r` et `pihole -up` continuaient de suivre les liens symboliques lors de l’écriture de `install.log`, `migration_backup_v6` et `hosts/custom.list` dans `/etc/pihole`. Cela nécessite une exécution de code en tant qu’utilisateur `pihole` et qu’un administrateur lance ensuite une réparation ou une mise à jour en tant que root. Les versions jusqu’à 6.5 sont affectées.
- **GHSA-5w4v-f65m-323f (Core), CVSS 7.3 :** root suivait les liens symboliques et les chemins issus de la configuration lors du téléchargement de `macvendor.db`, dans `pihole checkout` et dans `pihole uninstall`.
- **GHSA-jvg9-fm8x-2jqq (Core), CVSS 7.3 :** `pihole -d` suivait les liens symboliques dans des répertoires appartenant à l’utilisateur `pihole`. root pouvait ainsi copier des fichiers arbitraires dans le journal de débogage et en écraser d’autres ou modifier leur propriétaire.

Core v6.6 écrit donc `install.log`, la sauvegarde de migration, `hosts/custom.list` et `macvendor.db` uniquement en tant qu’utilisateur `pihole`. `pihole -d` ne travaille plus non plus en tant que root dans ces répertoires.

## Hachages de mots de passe dans le journal de débogage

**GHSA-qfq3-88xg-765w (Core), CVSS 5.3 :** le journal de débogage de `pihole -d`, et donc également les envois au service Tricorder, contenait le hachage du mot de passe Web, le hachage du mot de passe d’application et le secret TOTP. Core v6.6 retire ces valeurs du journal. Si vous avez téléversé un journal de débogage par le passé, le fabricant recommande de modifier le mot de passe et les mots de passe d’application, puis de reconfigurer l’authentification à deux facteurs.

## Correctifs de sécurité de sévérité Low dans FTL

- **GHSA-x6xh-vxcm-g8rm, CVSS 3.8 :** les sessions d’application sans `webserver.api.app_sudo` et les sessions CLI pouvaient lister toutes les sessions API et mettre fin à n’importe laquelle.
- **GHSA-87wp-wm3p-8cxm, CVSS 3.7 :** les entrées IPv4 dans `webserver.acl` ne s’appliquaient pas aux clients IPv4 sur un `webserver.port` double pile tel que `+80`.
- **GHSA-cg4p-5r3r-m54p, CVSS 3.1 :** une modification de mot de passe via `pihole-FTL --config` ou `pihole setpassword` pouvait être annulée par une écriture simultanée de configuration via l’API ; l’ancien mot de passe et ses sessions restaient alors valides.
- **GHSA-fj2f-pgf2-j888, CVSS 2.2 :** des requêtes enregistrées avec `misc.privacylevel` 3 pouvaient atteindre la base de données à long terme après l’abaissement du niveau.
- **GHSA-9mh9-95wj-4w4q et GHSA-9hh4-j8jf-6c96, CVSS 2.2 chacun :** un plantage lors de la lecture des métriques pendant le rechargement du fichier hosts, ainsi qu’un use-after-free lors de l’importation d’un Teleporter au format v5. Les deux nécessitent une session valide.

## Changements de comportement après la mise à jour

Selon les notes de version, FTL v6.9 n’est volontairement pas une v6.8.1, car la mise à jour de TRE modifie l’évaluation de certains filtres regex enregistrés :

- **Regex :** une borne inférieure absente (`{,n}`, `{,}`) signifie désormais `{0,n}` ou `{0,}`. Plus de trois groupes imbriqués avec correspondance approximative et les back-references dans des répétitions non limitées sont rejetés.
- **DNS local :** les entrées de `dns.cnameRecords` contenant un nom de domaine invalide ou un TTL ne tenant pas dans un entier sont rejetées.
- **webserver.acl :** les entrées IPv4 s’appliquent sur tous les ports, y compris les ports double pile. Les entrées IPv6, à l’exception de `::ffff:a.b.c.d`, ne correspondent plus qu’aux clients IPv6 ; `[::]/0` n’autorise donc plus les clients IPv4.
- **API des sessions :** les sessions d’application sans `webserver.api.app_sudo` et les sessions CLI reçoivent l’état 403 avec `GET /api/auth/sessions` et `DELETE /api/auth/session/{id}`. Une session peut toujours se terminer elle-même avec `DELETE /api/auth`.
- **Serveur Web :** les requêtes comportant plus de 64 en-têtes sont rejetées. `/api/info/system` renvoie une erreur plutôt que des valeurs nulles lorsque les Load Averages ne sont pas lisibles.
- **API DHCP :** `DELETE /api/dhcp/leases/{ip}` répond par 503 au lieu de 400 lorsque le résolveur n’a pas traité la requête à temps.
- **Mot de passe :** un mot de passe défini avec `pihole setpassword` ou `pihole-FTL --config` n’est plus écrasé par la prochaine modification de configuration dans le Web Interface.

Après la mise à jour, vérifiez les filtres regex avec des bornes inférieures ou des back-references, les entrées dans `webserver.acl` (notamment `[::]/0`) ainsi que les scripts ou intégrations qui listent ou terminent des sessions avec des mots de passe d’application.

## DNS, DHCP et autres correctifs dans FTL v6.9

Le dnsmasq intégré passe à v2.93+42 et comprend en outre un correctif dans `blockdata_alloc()` qui n’a pas encore été repris en amont. Selon les notes de version, sont notamment corrigés une lecture heap out-of-bounds dans `zone_status()` (DNSSEC), une écriture heap out-of-bounds dans `blockdata_read()`, un dépassement du tampon de nom DNS, un off-by-one dans l’option DHCPv6 FQDN, ainsi que des SERVFAIL pour des réponses wildcard plus profondes et des réponses perdues avec le pipelining TCP. Lorsque DNSSEC est activé, dnsmasq réserve des slots TCP supplémentaires pour ses propres requêtes de validation, afin que la validation fonctionne également lorsque les clients épuisent la limite de connexions TCP. L’amont non filtré de Quad9 est désormais identifié comme validant DNSSEC.

Dans la configuration et l’API, FTL v6.9 corrige notamment des modifications perdues lors de deux changements de configuration simultanés, un worker TCP qui retenait `pihole.toml.lock` jusqu’à 300 secondes, ainsi que des variables `FTLCONF_` ignorées lorsque FTL démarrait sans `pihole.toml`. Le serveur Web pouvait auparavant rester bloqué à 100 % de charge CPU et ne plus accepter de connexions. Dans les conteneurs, `/api/info/system` signalait jusqu’ici la charge de l’hôte. Pour l’exploitation, il est également pertinent que `gravity.db` ne soit plus rouvert que si le fichier a changé ; selon le fabricant, cela représentait la majeure partie de la charge CPU de FTL au repos. SQLite passe à 3.54.1.

## Changements dans Core v6.6 et dans l’image Docker

Core v6.6 remplace `sudo` par `setpriv` pour l’abaissement des privilèges de root vers l’utilisateur `pihole` ; `sudo` n’est donc plus une dépendance. Selon le fabricant, les commandes administratives telles que `pihole -up` nécessitent toujours les droits root. La rotation des journaux est désormais assurée par le logrotate quotidien du système ; la rotation forcée nocturne de Pi-hole est supprimée, car elle entrait en conflit avec l’exécution du système et effectuait une double rotation des journaux. D’autres correctifs concernent `pihole flush quiet`, `pihole updatechecker` et un avertissement lorsqu’un dépôt local pour git n’est pas marqué comme sûr (jusqu’ici, `pihole -v` affichait alors seulement N/A). La capability inutile `CAP_IPC_LOCK` a été supprimée.

L’image Docker adopte le passage à `setpriv` et planifie la rotation des journaux via la Crontab du conteneur. L’image nocturne est désormais construite depuis la branche `development`.

## Procédure recommandée

La version cible est l’image Docker 2026.11.0 ou, pour les installations Bare Metal, FTL v6.9 avec Core v6.6. Comme la faille via `files.gravity` n’est corrigée qu’avec les deux composants, vous devez mettre à jour FTL et Core ensemble. Créez auparavant une sauvegarde avec Teleporter (Settings, Teleporter dans le Web Interface ou `pihole-FTL --teleporter`), comme le recommande le fabricant, et lisez les changelogs en raison des changements de comportement. Mettez à jour les installations Bare Metal avec `pihole -up`. Dans le conteneur, `pihole -up` et `pihole -r` sont désactivés selon la documentation ; remplacez-y l’image. Avec Docker Compose, exécutez les commandes suivantes dans le répertoire de `docker-compose.yml`, conformément à la documentation :

```bash
docker compose pull
docker compose down
docker compose up -d
```

Pour les installations avec `docker run`, la documentation indique `docker pull pihole/pihole:latest`, puis `docker stop pihole` et `docker rm pihole`, ainsi qu’un nouveau `docker run` avec les arguments existants. Si vous avez précédemment téléversé un journal de débogage, modifiez également les mots de passe et reconfigurez la 2FA. Les notes de version ne contiennent aucune indication sur des problèmes connus ou des périodes de support (état au 12 novembre 2026).

## Sources

1.  [GitHub : pi-hole/docker-pi-hole Release 2026.11.0](https://github.com/pi-hole/docker-pi-hole/releases/tag/2026.11.0) : notes de version de l’image Docker avec les versions des composants, les changements Docker et la liste des onze avis.

2.  [GitHub : pi-hole/FTL Release v6.9](https://github.com/pi-hole/FTL/releases/tag/v6.9) : correctifs de sécurité, mises à jour de dnsmasq et SQLite, changements de comportement pour les regex, ACL et l’API.

3.  [GitHub : pi-hole/pi-hole Release v6.6](https://github.com/pi-hole/pi-hole/releases/tag/v6.6) : correctifs de sécurité de Core, passage à setpriv, changements de la rotation des journaux.

4.  [GitHub : pi-hole/web Release v6.6.1](https://github.com/pi-hole/web/releases/tag/v6.6.1) : état du Web Interface au 17 octobre 2026, inchangé dans 2026.11.0.

5.  [GitHub : pi-hole/docker-pi-hole Release 2026.10.1](https://github.com/pi-hole/docker-pi-hole/releases/tag/2026.10.1) : image précédente avec Alpine 3.24.3.

6.  [GitHub Advisory GHSA-pj9m-9623-2g75](https://github.com/pi-hole/FTL/security/advisories/GHSA-pj9m-9623-2g75) : exécution de code par un administrateur via files.gravity, CVSS 7.2, versions affectées et corrigées, CVE demandée.

7.  [GitHub Advisory GHSA-wp3p-45h3-2v82](https://github.com/pi-hole/pi-hole/security/advisories/GHSA-wp3p-45h3-2v82) : correction incomplète de liens symboliques dans l’installateur, CVSS 7.3, conditions requises.

8.  [Blog Pi-hole : Pi-hole FTL v6.9 and Core v6.6 released](https://pi-hole.net/blog/2026/11/11/pi-hole-ftl-v6-9-and-core-v6-6-released/) : annonce avec conseils de mise à jour et recommandation après téléversement de journaux de débogage.

9.  [Pi-hole Docs : Upgrading Pi-hole in Docker](https://docs.pi-hole.net/docker/upgrading/) : commandes de mise à jour pour Docker Compose et docker run, pihole -up désactivé dans le conteneur.
