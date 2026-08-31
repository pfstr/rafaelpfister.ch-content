---
title: "Contrôles essentiels pour les administrateurs TotemoMail : arrêter le serveur, vérifier les files d’attente et les purger de manière contrôlée"
navTitle: "Contrôles TotemoMail"
description: "Les principaux contrôles pour exploiter une passerelle TotemoMail : arrêter le service via systemd et le script de contrôle Tanuki, compter les messages en file d’attente par dépôt, inspecter des messages individuels, purger de manière contrôlée et redémarrer le service."
date: "2026-08-28"
kategorie: "TotemoMail"
timeToRead: "9 min de lecture"
themen:
  - totemomail
produkte:
  - "totemomail"
  - "apache-james"
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "controles-essentiels-pour-les-administrateurs-totemomail-arreter-le-serveur-verifier-les-files"
translationId: "article-3a0a526ab6e38a06"
translationOf: totemomail-server-stoppen-queues-bereinigen
url: https://rafaelpfister.ch/fr/blog/controles-essentiels-pour-les-administrateurs-totemomail-arreter-le-serveur-verifier-les-files
translationSourceHash: bc887dcd4aa82db7e020247f75b86528f0fa331e1643c28a215a1638587197a6
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:39:54.634Z
translationReview: automatic
---

# Contrôles essentiels pour les administrateurs TotemoMail : arrêter le serveur, vérifier les files d’attente et les purger de manière contrôlée

Pour exploiter une passerelle TotemoMail (aujourd’hui Kiteworks Email Protection Gateway), quatre étapes font partie des outils de base : arrêter proprement le service, établir l’état des files d’attente, inspecter des messages individuels et purger les files d’attente de manière contrôlée avant de redémarrer le service.

Ces étapes sont nécessaires aussi bien lors de maintenances planifiées qu’en cas de perturbations, par exemple lorsqu’une règle erronée, une destination inaccessible ou un test de charge a rempli les files d’attente. Cet article présente chaque étape avec les commandes concrètes, y compris la manière d’arrêter proprement le service. Le modèle de traitement sous-jacent (processeurs, dépôts, formats de fichiers) est décrit dans l’article [Comprendre le routage du courrier entre TotemoMail et Exchange Online](/blog/totemomail-m365) décrit.

Tous les chemins se réfèrent à une installation sous `/opt/totemomail` avec l’utilisateur de service `totemo`. Adaptez les chemins à votre environnement.

## Comment TotemoMail est démarré et arrêté

Avant d’arrêter un service, vous devez savoir comment il fonctionne. Avec TotemoMail, trois couches sont impliquées :

- Une **unité systemd** `totemomail.service` comme niveau de contrôle le plus externe.
- Le **script de contrôle** `/opt/totemomail/bin/totemomail`, qui appelle l’unité au démarrage et à l’arrêt.
- Le **Tanuki Java Service Wrapper** : un processus natif `wrapper`, qui démarre et surveille le processus Java proprement dit, et peut le redémarrer en cas de plantage.

Vous pouvez vérifier cette structure sur votre système sans devoir lire le fichier d’unité. `systemctl show` interroge directement les propriétés auprès de systemd et fonctionne même lorsque le fichier sous `/etc/systemd/system/` n’est lisible que par root :

```bash
systemctl show totemomail.service -p Type -p User -p ExecStart -p ExecStop \
  -p KillMode -p TimeoutStopUSec --no-pager
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `show totemomail.service` | Affiche les propriétés d’exécution de l’unité telles que systemd les a chargées |
| `-p <Property>` | Limite la sortie à la propriété indiquée ; peut être spécifiée plusieurs fois |
| `--no-pager` | Affiche directement dans la console au lieu d’ouvrir un pager comme `less` |

</details>

Une sortie typique ressemble à ceci :

```text
Type=oneshot
TimeoutStopUSec=1min 30s
ExecStart={ path=/opt/totemomail/bin/totemomail ; argv[]=/opt/totemomail/bin/totemomail start ; ... }
ExecStop={ path=/opt/totemomail/bin/totemomail ; argv[]=/opt/totemomail/bin/totemomail stop ; ... }
User=totemo
KillMode=control-group
```

Vous pouvez en déduire les propriétés importantes : `systemctl stop totemomail` appelle le script de contrôle avec l’argument `stop`, attend jusqu’à 90 secondes une fin propre, puis termine via `KillMode=control-group` tous les processus restants de l’unité. L’arrêt via systemd est donc équivalent à l’appel direct du script, mais effectue en plus un nettoyage si le script reste bloqué.

L’état `active (exited)` de `systemctl status totemomail` est normal dans cette configuration et ne constitue pas une erreur : l’unité est `Type=oneshot`, le script de démarrage se termine après le démarrage et le wrapper continue de s’exécuter comme démon autonome, géré seulement indirectement par systemd. La liste des processus, et non l’état de l’unité, indique donc si le service fonctionne réellement :

```bash
ps -ef | grep -E 'wrapper|TotemoBootStrapper' | grep -v grep
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-e` | Affiche tous les processus, et pas seulement ceux de la session en cours |
| `-f` | Format de sortie complet avec la ligne de commande intégrale |
| `grep -E 'wrapper\|TotemoBootStrapper'` | Filtre sur le processus wrapper et la classe principale Java |
| `grep -v grep` | Retire les processus grep eux-mêmes de la liste des résultats |

</details>

En fonctionnement normal, deux processus apparaissent : le `wrapper` natif (démarré avec `../conf/wrapper.conf` et le fichier PID `totemomail.pid`) et le processus Java avec la classe principale `ch.totemo.bootstrapper.TotemoBootStrapper`. Si l’un des deux manque, le service n’a pas complètement démarré.

## Étape 1 : arrêter le service

Pour toute intervention sur les files d’attente, arrêtez d’abord le service. Tant que TotemoMail fonctionne, il accepte les messages, traite les files d’attente et les distribue ; seul l’arrêt fige l’état pour l’analyse.

```bash
sudo systemctl stop totemomail
```

Vérifiez ensuite que les processus wrapper et Java sont terminés :

```bash
ps -ef | grep -E 'wrapper|TotemoBootStrapper' | grep -v grep
```

La sortie doit être vide. Le fichier PID `/opt/totemomail/bin/totemomail.pid` disparaît également. Si un processus reste actif après expiration du délai d’arrêt, systemd le termine via le groupe de contrôle ; dans ce cas, vérifiez `journalctl -u totemomail` avant de poursuivre.

N’oubliez pas le niveau amont : pendant l’arrêt, les nouveaux messages entrants s’accumulent sur le système expéditeur, par exemple dans la file d’attente Exchange ou sur le relais amont. C’est voulu. Les expéditeurs sérieux tentent automatiquement une nouvelle distribution après le redémarrage.

## Étape 2 : établir l’état des files d’attente

Les files d’attente de TotemoMail sont des dépôts de messagerie basés sur des fichiers de l’Apache James sous-jacent. Elles se trouvent sous le répertoire d’application James, ici `/opt/totemomail/mailer/apps/james/var/mail/`. Chaque sous-répertoire est un dépôt, et chaque message se compose de deux fichiers : `*.FileStreamStore` contient le message MIME complet, `*.FileObjectStore` l’objet d’état sérialisé avec les métadonnées.

Un décompte des fichiers `FileObjectStore` par répertoire fournit une vue d’ensemble :

```bash
for d in /opt/totemomail/mailer/apps/james/var/mail/*/; do \
  printf '%-22s %s\n' "$(basename "$d")" \
  "$(find "$d" -maxdepth 1 -name '*.FileObjectStore' | wc -l)"; \
done
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `for d in .../*/` | Itère sur tous les répertoires de dépôt (le `/` final limite aux répertoires) |
| `printf '%-22s %s\n'` | Formate la sortie en deux colonnes ; `%-22s` complète le nom à gauche sur 22 caractères |
| `basename "$d"` | Réduit le chemin complet au nom du répertoire |
| `find "$d" -maxdepth 1` | Recherche uniquement directement dans le répertoire, sans sous-répertoires |
| `-name '*.FileObjectStore'` | Compte un fichier par message ; l’équivalent Stream doublerait le nombre |
| `wc -l` | Compte les fichiers trouvés |

</details>

Le résultat comporte une ligne par file d’attente avec le nombre de messages, par exemple :

```text
DBUnavailable          0
error                  12
incoming               121
outgoing               0
spool                  0
```

Les dépôts standard ont la signification suivante : `spool` contient les messages acceptés mais non encore traités, `incoming` ceux à distribuer en interne, `outgoing` les messages sortants, `error` les messages ayant échoué et `DBUnavailable` les messages mis en attente en raison d’un backend inaccessible. Selon la configuration, d’autres dépôts existent pour des routes spécifiques ; ils suivent le même schéma de fichiers.

Si `find` est exécuté depuis un répertoire auquel l’utilisateur de service n’a pas accès (par exemple le répertoire personnel d’un autre utilisateur après `sudo -u totemo`), l’avertissement `Failed to restore initial working directory` apparaît à chaque appel. Il est sans gravité et disparaît après un `cd ~`.

## Étape 3 : examiner les messages

Les chiffres seuls ne suffisent pas pour prendre une décision. Avant de supprimer quoi que ce soit, vous devez savoir ce qui se trouve dans les files d’attente : des messages indésirables issus d’un incident, ou des e-mails légitimes qui doivent être distribués après le redémarrage ?

Les fichiers `FileStreamStore` sont des messages RFC 822 non modifiés. Les principaux en-têtes peuvent donc être lus directement :

```bash
for f in /opt/totemomail/mailer/apps/james/var/mail/incoming/*.FileStreamStore; do \
  awk 'BEGIN{IGNORECASE=1} /^(From|To|Subject|Date):/{print} /^\r?$/{exit}' "$f"; \
  echo ---; \
done | less
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `BEGIN{IGNORECASE=1}` | Compare les noms d’en-tête sans tenir compte de la casse (GNU awk) |
| `/^(From\|To\|Subject\|Date):/{print}` | Affiche uniquement les quatre lignes d’en-tête pertinentes |
| `/^\r?$/{exit}` | S’arrête à la ligne vide entre l’en-tête et le corps ; le contenu du message n’est pas lu |
| `echo ---` | Ligne de séparation entre les messages |
| `less` | Permet de feuilleter au lieu de faire défiler lorsqu’il y a beaucoup de messages |

</details>

Pour des volumes importants, la répartition est plus révélatrice que la vue individuelle. Pour afficher les expéditeurs les plus fréquents :

```bash
grep -him1 '^From:' /opt/totemomail/mailer/apps/james/var/mail/incoming/*.FileStreamStore \
  | sort | uniq -c | sort -rn | head
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-h` | Supprime les noms de fichiers de la sortie afin que les expéditeurs identiques soient regroupés |
| `-i` | Ignore la casse |
| `-m1` | Uniquement la première occurrence par fichier (l’en-tête, et non des lignes `From:` citées dans le corps) |
| `sort \| uniq -c` | Regroupe les lignes d’expéditeur identiques et les compte |
| `sort -rn \| head` | Trie par fréquence décroissante et affiche les dix plus fréquents |

</details>

Si un seul expéditeur ou un seul objet domine avec des centaines de copies, cela indique une boucle ou un envoi massif mal dirigé ; ces messages sont les candidats à la purge. Un regard sur les horodatages des fichiers (`ls -lt`) permet en outre de délimiter la période et de voir si des messages légitimes plus anciens s’y trouvent.

## Étape 4 : purger de manière contrôlée

Ce n’est qu’à présent que la suppression intervient, et même alors avec une étape intermédiaire : déplacez d’abord le contenu dans un répertoire de sauvegarde plutôt que de le supprimer directement. Le résultat pour l’exploitation de la messagerie est le même (la file d’attente est vide), mais l’opération est réversible, et des messages légitimes isolés peuvent ensuite être replacés depuis la sauvegarde ou réutilisés au format `.eml`.

```bash
mkdir -p /opt/totemomail/queue-backup-$(date +%F)
mv /opt/totemomail/mailer/apps/james/var/mail/incoming/* \
   /opt/totemomail/queue-backup-$(date +%F)/
```

Important : les répertoires de dépôt eux-mêmes restent en place ; seul leur contenu est déplacé. Les fichiers Stream et Object d’un message vont ensemble ; supprimer un seul des deux laisse des fichiers orphelins qui génèrent des erreurs dans le journal au prochain démarrage.

Une fois la sauvegarde vérifiée, ou lorsque le contenu n’a manifestement aucune valeur (par exemple s’il ne s’agit que de messages de test de charge), supprimez l’intégralité du contenu des files d’attente dans tous les dépôts :

```bash
find /opt/totemomail/mailer/apps/james/var/mail/ -mindepth 2 -maxdepth 2 -type f \
  \( -name '*.FileStreamStore' -o -name '*.FileObjectStore' \) -delete
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-mindepth 2 -maxdepth 2` | Cible uniquement les fichiers directement dans les répertoires de dépôt, pas `var/mail` lui-même ni les niveaux inférieurs |
| `-type f` | Uniquement les fichiers ordinaires ; les répertoires sont conservés |
| `\( -name ... -o -name ... \)` | Les deux types de fichiers d’un message, Stream et objet d’état |
| `-delete` | Supprime directement les éléments trouvés ; exécutez d’abord sans cette option afin de vérifier la liste des éléments trouvés |

</details>

Exécutez ensuite le même décompte qu’à l’étape 2 : tous les dépôts doivent afficher 0.

## Étape 5 : redémarrer le service

```bash
sudo systemctl start totemomail
```

Le démarrage appelle le script de contrôle avec `start`, qui daemonise le wrapper ; le wrapper démarre ensuite le processus Java. Vérifiez les deux via la liste des processus de la première section et consultez les fichiers journaux sous `/opt/totemomail/bin/` : `wrapper.log` consigne le démarrage du wrapper et de la JVM, tandis que `console.log` et `console.err` consignent les sorties de l’application elle-même.

Terminez par un test fonctionnel avec un unique message de test à travers la passerelle avant de réactiver le flux de messagerie normal. Et si une règle ou une boucle de messagerie avait rempli les files d’attente : corrigez d’abord la cause, puis autorisez à nouveau le trafic. Sinon, l’établissement de l’état des files d’attente recommence depuis le début.

## Résumé

| Étape | Commande | Vérification |
|---|---|---|
| Arrêter | `sudo systemctl stop totemomail` | Filtre `ps` vide, fichier PID disparu |
| Compter l’état | Boucle `find` sur `var/mail/*/` | Nombre par dépôt |
| Inspecter | Extraction des en-têtes `awk`, statistiques des expéditeurs `grep` | Séparer les messages indésirables des messages légitimes |
| Purger | `mv` vers la sauvegarde, puis `find ... -delete` | Le décompte affiche 0 partout |
| Démarrer | `sudo systemctl start totemomail` | Processus, `wrapper.log`, message de test |

## Sources

1.  [Apache James Server 2: Provided Mailets](https://james.apache.org/server/2/provided_mailets.html): Documentation des mailets et des dépôts sur lesquels repose la structure des files d’attente de TotemoMail.

2.  [Tanuki Software: Java Service Wrapper](https://wrapper.tanukisoftware.com/doc/english/introduction.html): Fonctionnement du wrapper qui démarre et surveille le processus Java de TotemoMail, y compris le fichier PID et `wrapper.conf`.

3.  [systemd.service(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html): Signification de `Type=oneshot`, `ExecStop` et `TimeoutStopSec` pour les unités qui appellent un script de contrôle externe.

4.  [systemd.kill(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.kill.html): `KillMode=control-group` comme mécanisme de protection qui termine les processus de l’unité restés actifs après le script d’arrêt.

5.  [RFC 5322: Internet Message Format](https://datatracker.ietf.org/doc/html/rfc5322): Structure des en-têtes de message lus lors de l’inspection des fichiers `FileStreamStore`.
