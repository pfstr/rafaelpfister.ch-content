---
title: "Lorsque le journal remplit le disque : limiter correctement log4j2 RollingFile, avec l’exemple de totemomail"
navTitle: "Espace disque log4j2"
description: "Un volume de journaux saturé peut, dans le pire des cas, paralyser toute la passerelle. Pourquoi l’association d’une rotation temporelle et par taille sans %i produit un unique fichier gigantesque, comment strategy.max limite la conservation, quel rôle joue le niveau de journalisation et où totemomail masque ces valeurs."
date: "2026-09-04"
kategorie: "Totemomail"
timeToRead: "9 min de lecture"
themen:
  - totemomail
produkte:
  - "totemomail"
protokolle:
  - "troubleshooting"
  - "storage"
slug: "lorsque-le-journal-remplit-le-disque-limiter-correctement-log4j2-rollingfile-avec-l-exemple-de"
translationId: "article-c400eee99d90052d"
translationOf: log4j2-rollingfile-plattenplatz-totemomail
url: https://rafaelpfister.ch/fr/blog/lorsque-le-journal-remplit-le-disque-limiter-correctement-log4j2-rollingfile-avec-l-exemple-de
translationSourceHash: 39952348654f81231356634fc8b434cbfecdea73118db7ff1add02720283792b
translationModel: gpt-5.6-terra
translatedAt: 2026-09-04T08:16:20.582Z
translationReview: automatic
---

# Lorsque le journal remplit le disque : limiter correctement log4j2 RollingFile, avec l’exemple de totemomail

Une passerelle de messagerie basée sur Java écrit des volumes surprenants en mode DEBUG. Une seule journée de forte charge peut générer plusieurs gigaoctets de journaux de trace et, si le volume de journaux est peu dimensionné, il se remplit. La conséquence est désagréable : le processus Java ne peut plus écrire dans son journal, le framework de journalisation passe dans un état d’erreur et, même après avoir libéré de l’espace, il ne recommence à écrire qu’après un redémarrage. Sur une passerelle de messagerie, un disque plein peut en outre perturber le spooling et la livraison. Le déclencheur est presque toujours une rotation des journaux qui est certes configurée, mais ne fonctionne pas comme on le suppose.

L’article suivant explique la rotation de log4j2 précisément à ce niveau, d’abord de manière générale puis concrètement pour totemomail (qui repose sur Apache James et log4j2). Le point central est une indication unique, facile à négliger dans le modèle de fichier.

## Comment log4j2 effectue la rotation

Le `RollingFileAppender` de log4j2 combine deux éléments : une ou plusieurs **TriggeringPolicies** déterminent *quand* la rotation a lieu, tandis qu’une **RolloverStrategy** détermine *comment* les fichiers d’archive sont nommés et combien sont conservés. Deux policies sont généralement utilisées simultanément :

- `TimeBasedTriggeringPolicy` : effectue une rotation selon le temps, généralement chaque jour.
- `SizeBasedTriggeringPolicy` : effectue une rotation dès que le fichier actif atteint une taille donnée, par exemple 100 MB.

Lors de la rotation, le fichier actif est renommé et archivé. Le nom du fichier d’archive est défini par le `filePattern`, qui contient deux espaces réservés dont l’interaction fait toute la différence.

<details class="options-details">
<summary>Aperçu des options</summary>

| Espace réservé | Signification |
|---|---|
| `%d{...}` | Date/heure de la rotation selon le modèle indiqué, par ex. `%d{yyyy-MM-dd}` pour le jour |
| `%i` | Indice calculé du fichier d’archive, un compteur qui augmente à chaque rotation |
| `%03i` | Le même indice, complété par des zéros sur trois positions |
| `.gz` / `.zip` à la fin du modèle | L’archive est compressée lors de la rotation |

</details>

La référence complète se trouve dans la documentation log4j2 du Rolling File Appender ; le tableau ci-dessus ne présente que les éléments essentiels à la rotation par taille et par temps.

## Le piège de %i

C’est précisément ici que se trouve l’erreur qui remplit les disques. Si vous nommez les fichiers uniquement par date, donc `filePattern = trace.log.%d{yyyy-MM-dd}`, tout en configurant une policy de taille de 100 MB, vous n’obtiendrez pas de nombreux fichiers de 100 MB par jour, mais un seul fichier qui continue à grossir sans limite. La rotation par taille ne dispose pas de sa propre destination dans laquelle écrire la partie suivante, car le modèle ne contient aucun compteur. La documentation log4j2 est explicite à ce sujet :

> When combined with a time-based triggering policy, the filePattern attribute of the Appender should contain an `%i` conversion pattern. Otherwise, the target file will be overwritten on each rollover.

Sans `%i`, l’association de la rotation temporelle et par taille est donc défectueuse ; selon la stratégie, le fichier est soit écrasé, soit il dépasse la taille configurée. En pratique, cela signifie que la limite de 100 MB ne s’applique jamais, qu’une journée de forte charge écrit tout dans un seul fichier et que celui-ci atteint plusieurs gigaoctets. La correction consiste à compléter le modèle :

```text
filePattern = trace.log.%d{yyyy-MM-dd}.%i
```

Chaque rotation de 100 MB crée alors son propre fichier indexé (`trace.log.2026-09-04.1`, `.2`, `.3`), et la limitation par taille fonctionne comme prévu.

## Conservation via strategy.max

L’indice est également indispensable au fonctionnement de la conservation. La `DefaultRolloverStrategy` dispose d’un attribut `max`, qui indique le nombre maximal de fichiers d’archive conservés ; au-delà de cette limite, les plus anciens sont supprimés. Sans `%i`, il n’y a pas d’indice que `max` puisse compter ; rien n’est donc supprimé et les anciens fichiers datés s’accumulent.

<details class="options-details">
<summary>Options expliquées</summary>

| Attribut | Effet |
|---|---|
| `max` | Nombre maximal de fichiers d’archive conservés ; les plus anciens sont supprimés au-delà |
| `min` | Valeur d’indice minimale (1 par défaut) |
| `fileIndex="min"` | Le fichier le plus récent reçoit l’indice `min`, le plus ancien `max` |
| `fileIndex="max"` (par défaut) | Le fichier le plus ancien reçoit l’indice `min`, le plus récent `max` |
| `fileIndex="nomax"` | Aucune suppression n’a lieu ; les nouvelles archives reçoivent des indices continuellement croissants |

</details>

La taille et le nombre définissent la limite globale : 100 MB par fichier multipliés par `max=10` plafonnent le journal à environ un gigaoctet, indépendamment du volume écrit. Si vous avez besoin d’un contrôle plus fin selon l’âge plutôt que le nombre, ajoutez à la stratégie une action `Delete` avec `IfLastModified` (âge) ou `IfAccumulatedFileSize` (taille totale) ; dans la plupart des cas, la combinaison de la taille par fichier et de `max` suffit.

## Le niveau de journalisation, véritable facteur de volume

La rotation et la conservation limitent l’espace consommé, mais ne changent rien à la quantité effectivement écrite. Le levier principal est le niveau de journalisation. Une passerelle exécutée en DEBUG en production consigne chaque étape de traitement de chaque message, ce qui représente plusieurs gigaoctets par jour sous charge. Pour l’exploitation normale, le niveau doit être INFO ou supérieur ; DEBUG est un outil d’analyse ponctuelle, pas un mode de fonctionnement permanent. Si le niveau est réglé sur INFO et que la rotation par taille avec `%i` est correctement configurée, les deux mécanismes se complètent : INFO maintient le volume quotidien à un faible niveau, et la rotation limite même un pic DEBUG.

## Où totemomail stocke ces valeurs

Dans totemomail, ces réglages ne se trouvent pas dans un `log4j2.xml` local, ce qui peut facilement induire en erreur lors du dépannage. La configuration est générée au moment de l’exécution à partir de propriétés ayant le préfixe `totemo.log4j2.*`, et ces propriétés sont gérées de manière centralisée via la console de gestion (section Logging + Tracking). Une recherche de `log4j2.xml` dans le système de fichiers reste donc vaine ; un `log4j.xml` dans le répertoire de configuration appartient à un composant fourni (openjms) et n’a rien à voir avec le journal de trace.

Les propriétés pertinentes et leur signification :

<details class="options-details">
<summary>Options expliquées</summary>

| Propriété | Signification |
|---|---|
| `totemo.log4j2.appender.a1.filePattern` | Le modèle de fichier ; `%i` doit y figurer |
| `totemo.log4j2.appender.a1.policies.size.size` | Taille par fichier pour la SizeBasedTriggeringPolicy, par ex. `100MB` |
| `totemo.log4j2.appender.a1.strategy.max` | Nombre de fichiers d’archive conservés |
| `totemo.log4j2.rootLogger.level` | Niveau du logger racine log4j2 |
| `totemo.log.priority` | Priorité de journalisation globale de l’application, le véritable commutateur DEBUG |
| `totemo.tracking` | Niveau de détail du suivi des messages ; `debug` génère les lignes par Mailet |

</details>

La double nature est importante : les loggers log4j2 peuvent être réglés sur `warn` ou `error` tout en produisant un flot DEBUG dans le journal de trace, car `totemo.log.priority` et `totemo.tracking` agissent comme des commutateurs supérieurs indépendants. Pour réduire le volume, réglez `totemo.log.priority` sur INFO et faites passer `totemo.tracking` de `debug` à `on` ; cela supprime les lignes de traitement détaillées. Comme les valeurs sont gérées via la console, elles s’appliquent à l’ensemble du cluster, et certaines exigent le redémarrage de l’instance pour prendre effet (cela est indiqué pour chaque propriété).

## Le redémarrage après saturation

Un détail facile à négliger : après que le disque a été plein une fois, la journalisation ne reprend pas d’elle-même, même si vous libérez de l’espace. L’appender de fichier reste dans son état d’erreur jusqu’au redémarrage du processus Java. On le reconnaît au fait que la passerelle accepte et traite toujours les e-mails (la bannière SMTP affiche l’heure correcte), mais que le journal de trace reste bloqué à l’instant où le disque s’est rempli. Un redémarrage contrôlé de l’instance rétablit la journalisation et active en même temps les paramètres d’appender modifiés, comme le nouveau `filePattern`.

## Diagnostic en quelques commandes

La partition pleine et son responsable peuvent être rapidement identifiés. Commencez par déterminer quel système de fichiers est concerné :

```bash
df -h
```

Si le volume de journaux est à 100 %, une liste triée par taille désigne le principal responsable :

```bash
du -sh /pfad/zu/logs/* | sort -rh | head
```

Si vous y trouvez un seul fichier journalier de plusieurs gigaoctets au lieu de nombreuses petites archives indexées, il s’agit du piège `%i`. Après la correction et un redémarrage, la liste des fichiers confirme que la rotation fonctionne :

```bash
ls -laht /pfad/zu/logs/trace.log*
```

Vous devez obtenir `trace.log` ainsi que les archives indexées `trace.log.<datum>.1`, `.2` et ainsi de suite, chacune ayant approximativement la taille maximale configurée.

## Résumé

Toute personne utilisant log4j2 avec une rotation temporelle et par taille doit impérativement inclure un `%i` dans le `filePattern`, sans quoi un seul fichier croît sans limite et la limite de taille reste inefficace. Avec `strategy.max` (conjointement avec l’indice), le nombre d’archives plafonne l’espace consommé, et le niveau de journalisation détermine le volume à la source. Dans totemomail, ces valeurs se trouvent dans la console de gestion sous `totemo.log4j2.*`, ainsi que dans les commutateurs supérieurs `totemo.log.priority` et `totemo.tracking` ; après la saturation du disque, un redémarrage de l’instance est nécessaire pour que la journalisation écrive à nouveau.

## Sources

1.  [Apache Logging Services: Log4j RollingFileAppender](https://logging.apache.org/log4j/2.x/manual/appenders/rolling-file.html): Référence concernant filePattern, les TriggeringPolicies et la DefaultRolloverStrategy, y compris l’indication relative à `%i` lors d’une rotation temporelle.

2.  [Apache Logging Services: Log4j Architecture](https://logging.apache.org/log4j/2.x/manual/architecture.html): Présentation des appenders, layouts et de la hiérarchie des loggers, pour comprendre le logger racine et le niveau de journalisation.
