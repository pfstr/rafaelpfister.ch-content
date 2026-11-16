---
title: "SQLite : comprendre et éviter « database is locked »"
navTitle: "Verrous SQLite"
description: "Pourquoi SQLite répond avec SQLITE_BUSY et « database is locked », comment le mode WAL, busy_timeout, des transactions courtes et les nouvelles tentatives évitent les verrous, et pourquoi des supports lents et une charge d’E/S sur un NAS déclenchent le problème. Avec un cas pratique dans Grafana."
date: "2026-11-12"
kategorie: "Bases de données"
timeToRead: "11 min de lecture"
themen:
  - datenbanken
produkte:
  - "sqlite"
protokolle:
  - "troubleshooting"
  - "container"
slug: "sqlite-comprendre-et-eviter-database-is-locked"
translationId: "article-2a328dabf9164dae"
translationOf: sqlite-database-locked-wal-io-last
url: https://rafaelpfister.ch/fr/blog/sqlite-comprendre-et-eviter-database-is-locked
translationSourceHash: 121b1c0f6dc58cb2249e9e0276937397b52725621cd1553b455a37069f5958de
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T13:50:44.933Z
translationReview: automatic
---

SQLite est utilisé dans de nombreux services auto-hébergés : Grafana, Home Assistant, Vaultwarden, Uptime Kuma, Jellyfin et de nombreuses autres applications stockent leurs données par défaut dans un fichier unique. Cela fonctionne jusqu’à ce que le journal affiche le message `database is locked`. Il indique qu’un accès n’a pas obtenu de verrou dans le délai d’attente autorisé ; le fichier de base de données lui-même reste intact. Que cela arrive rarement ou en permanence dépend de trois éléments : le mode de journalisation, le comportement de l’application en cas de conflit et la latence du support de stockage.

## Cas pratique : Grafana sur un NAS

Grafana s’exécutait dans un conteneur Docker sur un NAS Synology ; le fichier SQLite se trouvait dans un volume nommé. Sur le même NAS tournaient Prometheus, Loki, Paperless-ngx et plusieurs sauvegardes restic. Les supports étaient des disques durs à enregistrement magnétique en bardeaux (SMR).

Au début, le point de terminaison de santé de Grafana signalait `"database": "failing"`. Le journal contenait à intervalles rapprochés des entrées de ce type :

```text
level=error msg="Failed to save history batch"
  error="database is locked (5) (SQLITE_BUSY)"
```

Le message provient de l’historique des états d’alerte : Grafana enregistre par défaut les changements d’état des règles d’alerte sous forme d’annotations dans sa propre base de données. Avec un grand nombre de règles, cela génère continuellement de petites transactions d’écriture. Après un redémarrage du conteneur, Grafana ne parvenait plus à démarrer et redémarrait en boucle :

```text
Failed to stop all modules ... kv get:
  [sqlstore.max-retries-reached] retry 1: database is locked
```

Le problème a été résolu avec deux réglages dans le `docker-compose.yml` : activer le mode WAL et demander à Grafana de réessayer plusieurs fois en cas de verrou. La raison pour laquelle ces deux valeurs fonctionnent découle du fonctionnement des verrous SQLite.

## Comment SQLite gère les verrous

SQLite ne possède pas de processus serveur qui coordonne les accès. Chaque connexion ouvre elle-même le fichier et gère l’accès partagé au moyen des verrous de fichiers du système d’exploitation. Il en découle la caractéristique la plus importante : **une seule connexion peut écrire à la fois.** Cela s’applique dans tous les modes de journalisation.

Dans le mode classique de journal rollback (par défaut, `journal_mode=DELETE`), une transaction d’écriture passe par plusieurs niveaux de verrouillage :

| Verrou | Signification |
|---|---|
| `SHARED` | La connexion lit. Un nombre quelconque de lecteurs simultanés est autorisé. |
| `RESERVED` | La connexion veut écrire et prépare les modifications. Les lecteurs peuvent continuer à lire, mais pas un second écrivain. |
| `PENDING` | La connexion attend la fin des accès en lecture en cours. Les nouveaux lecteurs sont refusés. |
| `EXCLUSIVE` | La connexion écrit dans le fichier de base de données. Personne d’autre ne peut lire ni écrire. |

Pour valider la transaction, l’écrivain a donc besoin du verrou exclusif, qu’il n’obtient que lorsqu’aucun lecteur n’est encore actif. Une lecture de longue durée bloque ainsi tout écrivain, et pendant la validation, tous les lecteurs sont bloqués.

Lorsqu’une connexion n’obtient pas le verrou requis, SQLite renvoie le code d’erreur `SQLITE_BUSY` (5) avec le texte `database is locked`. Il faut le distinguer de `SQLITE_LOCKED` (6, `database table is locked`) : ce code concerne les conflits au sein d’une même connexion ou en mode cache partagé et a d’autres causes.

Le facteur déterminant est le temps pendant lequel SQLite attend avant d’afficher ce message. Sans configuration, SQLite n’attend pas du tout : la valeur par défaut de `busy_timeout` est 0, et le message apparaît immédiatement dès qu’une autre connexion détient le verrou. De nombreuses bibliothèques définissent leur propre valeur ; le module Python `sqlite3` attend par exemple 5 secondes.

## Mode WAL : séparer lecteurs et écrivains

En mode Write-Ahead Log (`journal_mode=WAL`), SQLite ajoute les modifications à un fichier distinct `<name>-wal` au lieu de les écrire directement dans le fichier de base de données. Les lecteurs voient l’état au début de leur transaction de lecture et lisent pour cela dans le fichier de base de données et le WAL. Il en résulte deux avantages :

- Les lecteurs ne bloquent pas l’écrivain, et l’écrivain ne bloque pas les lecteurs.
- Une validation consiste à ajouter des données à un fichier plutôt qu’à effectuer plusieurs écritures dispersées avec un journal. Chaque transaction génère moins d’E/S et le verrou d’écriture est détenu moins longtemps.

Le mode WAL ne change rien au fait qu’un seul écrivain peut être actif simultanément. Il réduit toutefois le temps pendant lequel le verrou est détenu et élimine les conflits entre lecture et écriture. Pour une application comme Grafana, qui sert simultanément des tableaux de bord et enregistre les états d’alerte, c’est la mesure la plus efficace.

Le mode est défini une seule fois et reste enregistré dans le fichier :

```sql
PRAGMA journal_mode = WAL;
PRAGMA busy_timeout = 5000;
PRAGMA synchronous = NORMAL;
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `journal_mode = WAL` | Passe au mode Write-Ahead Log. Le paramètre est persistant ; les nouvelles connexions ouvrent automatiquement le fichier en mode WAL. |
| `busy_timeout = 5000` | Délai d’attente en millisecondes pendant lequel SQLite tente à répétition d’obtenir un verrou occupé. Ne s’applique qu’à la connexion actuelle et doit être défini à nouveau à chaque ouverture. |
| `synchronous = NORMAL` | En mode WAL, pas de `fsync` à chaque validation, mais uniquement lors des points de contrôle. La base de données reste cohérente en cas de panne de courant ; les dernières transactions validées peuvent être perdues. S’applique par connexion. |

</details>

Le mode WAL a des prérequis. Tous les processus qui accèdent au fichier doivent s’exécuter sur la même machine, car ils se coordonnent au moyen d’un fichier de mémoire partagée `<name>-shm`. Le répertoire doit disposer des droits d’écriture pour les deux fichiers auxiliaires, y compris pour les processus qui ne font que lire. Lors des sauvegardes, les fichiers doivent aussi être considérés ensemble (voir plus bas).

## Points de contrôle : lorsque le WAL grossit

Le WAL grossit jusqu’à ce qu’un point de contrôle transfère les modifications dans le véritable fichier de base de données. SQLite le déclenche automatiquement dès que le WAL contient environ 1000 pages (`wal_autocheckpoint`, soit environ 4 Mo avec des pages de 4 KiB). Le point de contrôle écrit les pages à leur emplacement dans le fichier de base de données, puis appelle `fsync`. Sur un support lent, cette étape est précisément celle qui dure le plus longtemps.

Un point de contrôle ne peut transférer des pages qu’à condition qu’aucun lecteur n’ait besoin d’un état plus ancien. Si au moins une transaction de lecture est constamment active, le point de contrôle reste incomplet et le WAL continue de grossir. Un WAL de plusieurs centaines de mégaoctets indique des transactions de lecture de longue durée ou des points de contrôle qui ne se terminent pas en raison de la charge d’E/S. Vous pouvez le vérifier ainsi :

```sql
PRAGMA wal_checkpoint(TRUNCATE);
```

La sortie se compose de trois nombres : si le point de contrôle a été bloqué (`1` = oui), le nombre de pages présentes dans le WAL et le nombre de pages transférées. Avec `TRUNCATE`, le WAL est ensuite réduit à zéro octet, à condition que le point de contrôle ait été complet.

## Délai d’attente busy et nouvelles tentatives

Avec un délai d’attente busy, SQLite attend en cas de conflit au lieu d’abandonner immédiatement. L’erreur n’apparaît plus que si le verrou est détenu plus longtemps que le délai configuré. Pour les applications qui effectuent peu d’écritures courtes, 5 à 10 secondes suffisent.

Il existe un cas dans lequel le délai d’attente busy ne s’applique pas. Si une transaction commence par lire (`BEGIN` sans ajout, donc `DEFERRED`) et veut ensuite écrire alors qu’un autre écrivain a entre-temps terminé une validation, SQLite renvoie immédiatement `SQLITE_BUSY`. La transaction travaille sur un état obsolète et ne peut pas devenir une transaction d’écriture ; attendre n’y changerait rien. En mode rollback, une situation comparable existe lorsque deux connexions veulent simultanément passer de `SHARED` à un verrou d’écriture.

La solution consiste à désigner les transactions d’écriture comme telles dès le départ :

```sql
BEGIN IMMEDIATE;
UPDATE alert_state SET state = 'firing' WHERE id = 42;
COMMIT;
```

`BEGIN IMMEDIATE` demande le verrou d’écriture au démarrage. S’il est occupé, le délai d’attente busy s’applique normalement. En Python, cela ressemble à ceci :

```python
import sqlite3

con = sqlite3.connect("app.db", timeout=10.0,
                      isolation_level=None)
con.execute("PRAGMA journal_mode = WAL")
con.execute("BEGIN IMMEDIATE")
con.execute("UPDATE jobs SET status = ? WHERE id = ?",
            ("done", 42))
con.execute("COMMIT")
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `"app.db"` | Chemin vers le fichier de base de données. |
| `timeout=10.0` | Délai d’attente busy en secondes ; correspond à `PRAGMA busy_timeout = 10000`. |
| `isolation_level=None` | Désactive la gestion automatique des transactions du module afin que `BEGIN IMMEDIATE` et `COMMIT` puissent être définis explicitement. |

</details>

Les nouvelles tentatives au niveau de l’application complètent le délai d’attente busy. Elles interceptent `SQLITE_BUSY`, attendent brièvement et exécutent à nouveau l’ensemble de la transaction. Elles couvrent aussi le cas d’un état de lecture obsolète, car la répétition démarre avec un nouvel état. Grafana propose ses propres réglages à cet effet (voir la section sur la solution).

## Transactions courtes et un seul écrivain

La durée pendant laquelle un verrou d’écriture est détenu est déterminée par l’application. Trois règles permettent de la réduire :

- **Aucun travail sans rapport avec la base de données dans la transaction.** Les requêtes HTTP, accès aux fichiers ou calculs doivent avoir lieu avant `BEGIN` ou après `COMMIT`.
- **Regrouper de nombreuses petites écritures.** 500 validations individuelles coûtent 500 cycles de verrouillage et, en mode rollback, plusieurs appels à `fsync` 500 fois. Une transaction comportant 500 lignes détient le verrou une fois, un peu plus longtemps.
- **Fermer les curseurs de lecture.** Un résultat de requête qui n’a pas été entièrement récupéré maintient la transaction de lecture ouverte et bloque le point de contrôle en mode WAL.

Lorsque plusieurs threads ou processus écrivent, un seul écrivain avec une file d’attente est la solution la plus fiable : un thread ou processus possède l’unique connexion d’écriture, tous les autres lui transmettent des tâches. Les conflits de verrouillage entre écrivains disparaissent ainsi complètement, tandis que les lecteurs continuent à travailler en parallèle en mode WAL.

À l’inverse, plusieurs conteneurs qui ouvrent le même fichier SQLite via un volume partagé sont une cause fréquente de verrous. Chaque conteneur est un écrivain distinct, et aucun ne connaît les autres. Si une application est mise à l’échelle horizontalement, un serveur de base de données tel que PostgreSQL ou MySQL est le choix approprié.

## Pas de systèmes de fichiers réseau

SQLite repose sur le bon fonctionnement atomique des verrous de fichiers et sur le fait qu’un `fsync` écrit effectivement les données sur le support. Avec NFS, SMB/CIFS et de nombreux systèmes de fichiers FUSE, aucune de ces deux garanties n’est assurée. Les développeurs de SQLite déconseillent explicitement d’utiliser une base de données via un système de fichiers réseau depuis plusieurs ordinateurs. Les conséquences vont de messages de verrouillage permanents à une base de données endommagée, et le mode WAL n’y fonctionne fondamentalement pas, car la synchronisation par mémoire partagée n’est possible que localement.

Pour Docker, cela signifie qu’un volume nommé ou un bind mount sur un support local convient. Un bind mount vers un partage SMB ou NFS monté ne convient pas, même s’il semble fonctionner en fonctionnement normal.

## Charge d’E/S : pourquoi le verrou dure soudainement trop longtemps

Le verrou n’est qu’un symptôme. Il devient un problème lorsqu’il est détenu plus longtemps que le délai d’attente busy ne l’autorise, ce qui dépend presque toujours de la latence d’écriture du support.

Chaque validation se termine par un `fsync`. SQLite ne libère le verrou qu’une fois que le support a confirmé que les données ont été écrites de manière durable. Sur un SSD, cela prend moins d’une milliseconde ; sur un disque dur peu sollicité, quelques millisecondes. Si le support doit traiter simultanément d’autres tâches, le `fsync` attend dans la file d’attente et le verrou reste détenu pendant ce temps. Tous les autres écrivains attendent également jusqu’à l’expiration de leur délai d’attente busy.

Plusieurs facteurs font augmenter cette latence sur un NAS :

- **Disques durs SMR.** Avec l’enregistrement magnétique en bardeaux, les pistes se chevauchent. Les écritures aléatoires arrivent d’abord dans une zone de cache, puis sont réorganisées ultérieurement dans les zones qui se chevauchent. Si la zone de cache est pleine sous charge continue, la latence d’écriture passe de quelques millisecondes à plusieurs secondes. Les bases de données comportant de nombreuses petites écritures dispersées représentent le profil de charge le plus défavorable.
- **Sauvegardes parallèles.** restic, Hyper Backup ou rsync lisent de grandes quantités de données et concurrencent chaque `fsync` sur les mêmes plateaux.
- **Scrubbing et contrôles de cohérence RAID.** Un scrub de support lit l’ensemble des données et s’exécute selon une planification sur de nombreux systèmes NAS, souvent pendant plusieurs heures.
- **Autres bases de données sur le même volume.** Prometheus et Loki écrivent continuellement des blocs et les compressent périodiquement. Ces compactages génèrent des pics de charge.
- **Points de contrôle en mode WAL.** Le point de contrôle écrit de nombreuses pages à la fois et se termine par `fsync`. S’il survient pendant une période de forte charge, il s’allonge d’autant.

Dans le cas pratique, plusieurs de ces facteurs se sont cumulés : disques SMR, sauvegardes et environnement de supervision avec plusieurs services effectuant des écritures. Grafana tournait en mode rollback sans nouvelles tentatives. Chaque validation de l’historique des états d’alerte nécessitait plusieurs appels à `fsync`, et chacun attendait derrière le reste de la charge. Au démarrage, Grafana lit notamment des valeurs dans son magasin clé-valeur de la base de données. Sans répétition, le démarrage échouait au premier conflit de verrouillage et le conteneur était redémarré.

### Mesurer la latence

Vous pouvez mesurer directement si les supports sont à l’origine du problème. `iostat` du paquet sysstat affiche l’utilisation par support :

```bash
iostat -x 5
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-x` | Statistiques étendues par périphérique, notamment `r_await`/`w_await` (temps d’attente moyen en ms) et `%util` (utilisation). |
| `5` | Intervalle en secondes ; la première sortie affiche les moyennes depuis le démarrage du système et n’est pas représentative de la situation actuelle. |

</details>

Des valeurs de `w_await` de l’ordre de centaines de millisecondes et un `%util` proche de 100 indiquent que le support doit mettre les tâches en attente. La latence des écritures synchrones individuelles peut être vérifiée avec `dd` dans le répertoire de la base de données :

```bash
dd if=/dev/zero of=./fsync-test bs=4k count=500 \
   oflag=dsync
rm ./fsync-test
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `if=/dev/zero` | Source : octets nuls ; seules les performances d’écriture sont mesurées. |
| `of=./fsync-test` | Fichier cible dans le répertoire actuel ; celui-ci doit se trouver sur le volume de la base de données. |
| `bs=4k` | Taille de bloc de 4 KiB, correspondant à la taille de page SQLite habituelle. |
| `count=500` | Nombre de blocs, soit 2 Mo au total. |
| `oflag=dsync` | Chaque bloc est écrit de manière synchrone, comparable à un `fsync` par opération d’écriture. |

</details>

À la fin, `dd` indique la durée et le débit. 500 blocs en 2 à 5 secondes correspondent à 4 à 10 ms par écriture synchrone et sont normaux pour des disques durs. Si le même test dure 60 secondes ou davantage pendant une sauvegarde, la cause des verrous se trouve au niveau du support.

## Solution dans le cas pratique

Grafana prend en charge les deux mesures via des paramètres de configuration dans la section `[database]`, qui peuvent être définis comme variables d’environnement :

```yaml
services:
  grafana:
    image: grafana/grafana-oss:13.0.2
    environment:
      GF_DATABASE_WAL: "true"
      GF_DATABASE_QUERY_RETRIES: "5"
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  grafana-data:
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `GF_DATABASE_WAL` | Correspond à `wal = true` : Grafana ouvre le fichier SQLite en mode WAL. La valeur par défaut est `false`. |
| `GF_DATABASE_QUERY_RETRIES` | Correspond à `query_retries` : nombre de répétitions d’une requête lorsque SQLite signale `database is locked`. La valeur par défaut est `0`. |
| `grafana-data:/var/lib/grafana` | Volume nommé sur le support local du NAS ; il contient `grafana.db`, ainsi qu’après la migration `grafana.db-wal` et `grafana.db-shm`. |

</details>

Pour les transactions, `transaction_retries` est également disponible (valeur par défaut `5`), mais il n’a pas dû être modifié dans le cas pratique. Après le redémarrage avec les deux réglages, Grafana a démarré sans erreur, le point de terminaison de santé a de nouveau signalé `"database": "ok"`, et les messages SQLITE_BUSY de l’historique des états d’alerte ont cessé. Les nouveaux fichiers dans le volume permettent de vérifier que la migration a pris effet :

```bash
docker exec grafana ls -l /var/lib/grafana
```

Si `grafana.db-wal` et `grafana.db-shm` apparaissent, la base de données fonctionne en mode WAL.

Sur les systèmes fortement chargés, deux mesures supplémentaires peuvent être envisagées. L’historique des états d’alerte peut être écrit dans Loki plutôt que dans la base de données (`[unified_alerting.state_history]` avec `backend = loki`), lorsqu’un Loki est déjà en service. Et pour les installations avec de nombreux utilisateurs ou plusieurs instances Grafana, Grafana recommande une base de données externe (MySQL ou PostgreSQL) plutôt que SQLite.

## Sauvegarder sans conflits de verrouillage

Une sauvegarde de fichier de `grafana.db` pendant son utilisation capture un état intermédiaire arbitraire. En mode WAL, tous les changements non encore transférés manquent également sans le fichier `-wal`. SQLite génère lui-même une copie cohérente :

```bash
sqlite3 /var/lib/grafana/grafana.db \
  ".backup /backup/grafana-$(date +%F).db"
```

L’API de sauvegarde en ligne copie la base de données page par page et prend en compte les écritures en cours. Alternativement, `VACUUM INTO '/backup/grafana.db'` génère une copie compacte. Les deux variantes peuvent être exécutées avant la sauvegarde de fichiers proprement dite, qui sauvegarde alors la copie au lieu du fichier actif. Planifiez les sauvegardes et les scrubs de manière à ce qu’ils ne se chevauchent pas et ne coïncident pas avec les pics de charge connus d’autres services.

## Liste de contrôle

- Activer le mode WAL, à condition que tous les accès proviennent de la même machine.
- Définir le délai d’attente busy sur chaque connexion, avec 5 à 10 secondes comme valeur de départ.
- Commencer les transactions d’écriture par `BEGIN IMMEDIATE` et les garder courtes.
- Prévoir des répétitions pour `SQLITE_BUSY` au niveau de l’application, dans Grafana via `query_retries`.
- N’exécuter qu’un seul processus d’écriture par fichier de base de données.
- Stocker les fichiers de base de données uniquement sur des systèmes de fichiers locaux, jamais sur des partages SMB ou NFS.
- Mesurer la latence d’écriture lorsque les verrous se multiplient ; éviter les disques SMR pour les bases de données ou déplacer les bases de données vers un SSD.
- Créer les sauvegardes via `.backup` ou `VACUUM INTO` et les séparer dans le temps des scrubs.

## Sources

1.  [SQLite: Result and Error Codes](https://www.sqlite.org/rescode.html): signification de `SQLITE_BUSY` (5), `SQLITE_LOCKED` (6) et des codes étendus tels que `SQLITE_BUSY_SNAPSHOT`.

2.  [SQLite: File Locking And Concurrency In SQLite Version 3](https://www.sqlite.org/lockingv3.html): les niveaux de verrouillage SHARED, RESERVED, PENDING et EXCLUSIVE en mode rollback.

3.  [SQLite: Write-Ahead Logging](https://www.sqlite.org/wal.html): fonctionnement, points de contrôle, `wal_autocheckpoint` et la limitation aux systèmes de fichiers locaux.

4.  [SQLite: PRAGMA Statements](https://www.sqlite.org/pragma.html): `journal_mode`, `busy_timeout`, `synchronous` et `wal_checkpoint`.

5.  [SQLite: BEGIN TRANSACTION](https://www.sqlite.org/lang_transaction.html): différence entre DEFERRED, IMMEDIATE et EXCLUSIVE.

6.  [SQLite: SQLite Over a Network, Caveats and Considerations](https://www.sqlite.org/useovernet.html): pourquoi les systèmes de fichiers réseau ne conviennent pas à SQLite.

7.  [SQLite: How To Corrupt An SQLite Database File](https://www.sqlite.org/howtocorrupt.html): verrous de fichiers défectueux et `fsync` comme causes de bases de données endommagées.

8.  [SQLite: SQLite Backup API](https://www.sqlite.org/backup.html): copies cohérentes d’une base de données pendant son fonctionnement.

9.  [Python: sqlite3 DB-API 2.0 interface](https://docs.python.org/3/library/sqlite3.html): paramètres `timeout` et `isolation_level` de `sqlite3.connect()`.

10.  [Grafana: Configure Grafana, section database](https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/#database): paramètres `wal`, `query_retries` et `transaction_retries`, ainsi que leur correspondance avec les variables d’environnement `GF_`.

11.  [Wikipedia: Shingled magnetic recording](https://en.wikipedia.org/wiki/Shingled_magnetic_recording): structure des disques durs SMR et leur comportement lors d’écritures aléatoires.
