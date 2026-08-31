---
title: "Test de charge SMTP avec Apache JMeter en pratique : 10'000 e-mails, cinq chemins de règles, un rapport HTML"
navTitle: "Test de charge JMeter"
description: "Un test de charge réalisé de A à Z : plan de test avec un mélange de messages suivant les chemins de ruleset d’une passerelle de chiffrement, configuration portable sans installation, 10'000 e-mails en rafale et évaluation via le rapport HTML de JMeter, y compris les problèmes réellement rencontrés."
date: "2026-08-24"
kategorie: "SMTP et flux de messagerie"
timeToRead: "11 min de lecture"
themen:
  - smtp-mailflow
  - testing
  - totemomail
produkte:
  - "uebergreifend"
  - "totemomail"
  - "apache-james"
protokolle:
  - "testing"
  - "smtp"
  - "troubleshooting"
related:
  - mail-lasttest-tools-linux-windows-vergleich
image: "../images/jmeter-report-dashboard.png"
slug: "test-de-charge-smtp-avec-apache-jmeter-en-pratique-10-000-e-mails-cinq-chemins-de-regles-un"
translationId: "article-fc3f25272e051f92"
aiPrompt: |
  Du bist mein Assistent für Mail-Lasttests mit Apache JMeter. Hilf mir Schritt für Schritt, einen SMTP-Lasttest aufzubauen: portables Setup (JRE + JMeter ohne Installation), lokale SMTP-Senke mit aiosmtpd, Testplan mit Thread Group, Throughput Controllern für den Nachrichtenmix und SMTP Samplern, Lauf im CLI-Modus mit HTML-Report und Auswertung der Perzentile pro Nachrichtenklasse. Frage zuerst nach Zielsystem, Nachrichtenklassen und gewünschtem Volumen.
translationOf: jmeter-smtp-lasttest-html-report
translationSourceHash: 26c09e391d2252b6203dceb5dc45edd23beba797820fe0b95273bf48a9afc181
translationModel: gpt-5.6-terra
translatedAt: 2026-08-30T09:22:46.819Z
translationReview: required
url: https://rafaelpfister.ch/fr/blog/test-de-charge-smtp-avec-apache-jmeter-en-pratique-10-000-e-mails-cinq-chemins-de-regles-un
---

# Test de charge SMTP avec Apache JMeter en pratique : 10'000 e-mails, cinq chemins de règles, un rapport HTML

L’[article de synthèse sur les tests de charge de messagerie](/blog/mail-lasttest-tools-linux-windows-vergleich) a comparé les outils et esquissé le plan de test. Voici la mise en pratique : un test de charge JMeter complet avec 10'000 e-mails, un mélange de messages suivant de véritables chemins de règles de passerelle et le rapport HTML comme évaluation. Toutes les valeurs présentées proviennent de l’exécution réelle, y compris les erreurs survenues en cours de route.

Le scénario s’inspire d’un projet réel : une passerelle de chiffrement des e-mails basée sur Apache James (Totemomail) est placée en boucle de smarthost derrière Exchange Online et décide, pour chaque message, du chiffrement, de la signature et du routage spécial. Le ruleset Mailet comporte plusieurs chemins à cet effet : des déclencheurs dans l’objet tels que (sec), (sign) et (unsec), des mots-clés comme VERTRAULICH pour le routage vers une passerelle sectorielle, ainsi que le chemin standard avec vérification des certificats et repli en texte clair. Un test de charge qui ne soumettrait qu’un seul type de message mesurerait toujours le même chemin dans cet ensemble de règles ; le plan de test représente donc cinq classes dont le ratio correspond au trafic attendu.

Important pour l’interprétation : ce plan de test génère le profil de charge de nombreux expéditeurs indépendants, car JMeter ouvre une connexion distincte pour chaque message (le contexte figure dans la délimitation à la fin). C’est le modèle approprié pour démontrer qu’un ensemble de règles fonctionne correctement et suffisamment rapidement sous un trafic mixte parallèle. En revanche, le plan ne reproduit pas la charge de pointe d’un seul expéditeur de masse avec des sessions ouvertes ; pour ce profil de charge, `smtp-source` de l’[article de synthèse](/blog/mail-lasttest-tools-linux-windows-vergleich) est l’outil adéquat.

## Les principales options de jmeter

Pour s’orienter, voici les options de ligne de commande utilisées dans cet article, traduites librement de la documentation :

<details class="options-details">
<summary>Vue d’ensemble des options</summary>

| Option | Signification |
|---|---|
| `-n` | Mode CLI (sans interface graphique) : exécute le plan de test sans interface graphique |
| `-t datei` | Chemin vers le fichier JMX contenant le plan de test |
| `-l datei` | Chemin vers le fichier de résultats JTL dans lequel les mesures sont enregistrées |
| `-e` | Génère directement le rapport de tableau de bord HTML après l’exécution |
| `-o verzeichnis` | Répertoire cible du rapport ; il doit être vide ou ne pas encore exister |
| `-g datei` | Génère ultérieurement le rapport à partir d’un fichier JTL existant, sans nouvelle exécution |
| `-J<property>=<wert>` | Définit une propriété JMeter uniquement pour cet appel |

</details>

La liste complète est affichée par `jmeter -?` ; les options sont décrites dans le chapitre consacré au fonctionnement sans interface graphique du [manuel utilisateur de JMeter](https://jmeter.apache.org/usermanual/get-started.html).

## La configuration : ne rien devoir installer

Le test a été exécuté sur une machine Windows sans Java ni JMeter. Les deux peuvent être utilisés de manière portable, ce qui est déterminant sur les postes d’administration aux droits d’installation limités : JRE Temurin au format ZIP depuis Adoptium, JMeter au format ZIP depuis apache.org, décompresser les deux, définir `JAVA_HOME` sur le répertoire JRE, et c’est tout.

```bash
export JAVA_HOME="$PWD/jdk-21-jre"
export PATH="$JAVA_HOME/bin:$PATH"
./apache-jmeter-5.6.3/bin/jmeter -n -t gateway-lasttest.jmx -l lauf.jtl -e -o report
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `export JAVA_HOME=…` | Pointe vers le répertoire JRE décompressé ; JMeter y trouve l’environnement d’exécution Java sans installation |
| `export PATH=…` | Place les binaires JRE au début du chemin de recherche |
| `-n` | Mode CLI sans interface graphique |
| `-t gateway-lasttest.jmx` | Le plan de test à exécuter |
| `-l lauf.jtl` | Fichier de résultats contenant les mesures de chaque sampler |
| `-e` | Génère le rapport HTML directement après l’exécution |
| `-o report` | Répertoire cible du rapport |

</details>

Une boîte noire SMTP locale basée sur aiosmtpd a servi de puits, avec une quarantaine de lignes Python : elle accepte chaque message avec `250`, rejette le contenu, effectue le comptage et attribue chaque e-mail à une classe selon sa ligne d’objet. Ce comptage indépendant côté réception constitue l’essai de contrôle du test ; si les nombres du générateur et du puits ne concordent pas, quelque chose a été perdu en chemin.

```python
from aiosmtpd.controller import Controller

class SinkHandler:
    def __init__(self):
        self.count = 0

    async def handle_DATA(self, server, session, envelope):
        self.count += 1
        # Extraire l’objet de l’en-tête pour les statistiques par classe,
        # Le contenu est supprimé
        return "250 Message accepted for delivery"

controller = Controller(SinkHandler(), hostname="127.0.0.1", port=2525)
controller.start()
```

Important pour l’interprétation : le générateur et le puits s’exécutaient sur la même machine, sans TLS et sans réseau entre les deux. Les chiffres mesurés ne constituent donc pas une évaluation d’une passerelle, mais l’auto-test du générateur présenté dans l’article de synthèse : la preuve que le montage de charge peut effectivement générer le débit cible, ainsi que la limite supérieure à laquelle les mesures ultérieures seront comparées avec le véritable système de test.

## Le plan de test : cinq classes de messages, un ratio

Le cœur du plan est un Thread Group de 20 threads, avec une montée en charge de 10 secondes et 500 boucles, soit 10'000 itérations. Il contient cinq Throughput Controller en mode « Percent Executions », chacun avec exactement un SMTP Sampler :

| Classe (libellé du sampler) | Part | Chemin de règles dans la passerelle |
|---|---|---|
| 01 Standard sans déclencheur | 60 % | Vérification AutoGenerated, vérification des certificats, repli en texte clair |
| 02 Déclencheur (sec) | 15 % | Enveloppe TRE pour les destinataires sans certificat |
| 03 Déclencheur (sign) | 10 % | Certificate Exchange : signer, joindre la clé |
| 04 Mot-clé VERTRAULICH | 10 % | Routage spécial vers la passerelle sectorielle |
| 05 Déclencheur (unsec) | 5 % | Texte clair forcé |

La répartition en cinq samplers distincts plutôt qu’en un sampler avec un objet variable a une raison concrète : le rapport HTML regroupe toutes les métriques par libellé de sampler. Cinq libellés donnent cinq lignes dans les statistiques, avec leurs propres percentiles par classe ; un seul sampler avec un objet alimenté par CSV ne produirait qu’une ligne agrégée, et la différence entre les chemins de règles serait invisible dans l’évaluation.

Chaque sampler renseigne les champs habituels : hôte cible et port comme variables définies par l’utilisateur (`${zielhost}`, `${zielport}`), afin que le même plan puisse être exécuté sans modification contre le puits, l’environnement de test ou la préproduction, ainsi que l’expéditeur, le destinataire, l’objet avec un marqueur clair (ici le mot LOADTEST dans l’objet) et un corps de texte d’environ 1 à 2 Ko. L’option « Include timestamp in subject » ajoute l’heure de soumission en millisecondes ; lors d’une exécution ultérieure contre un véritable système à plusieurs niveaux, elle permet de calculer, avec les heures de réception du puits, la latence de bout en bout par message.

Une erreur de cette exécution qui peut être généralisée : la première tentative a échoué avec 10'000 erreurs en 10 secondes, toutes avec `java.lang.ClassCastException ... NullProperty cannot be cast to ... CollectionProperty` au lieu d’une réponse SMTP. La cause était un fichier JMX construit à la main dans lequel la liste d’en-têtes du sampler manquait ; le sampler exige cette propriété, même si elle est vide. La leçon porte moins sur cette propriété précise que sur le modèle à suivre : créer les plans de test dans l’interface graphique et les enregistrer, plutôt que d’écrire le XML à la main, puis effectuer une exécution minimale avant chaque rafale et vérifier côté puits que l’objet et le contenu arrivent réellement. Un compteur d’erreurs à 100 % avec un temps de réponse de 0 ms signifie presque toujours que l’erreur se produit avant le réseau, et que le test n’a donc jamais atteint le système cible.

## L’exécution

La mesure elle-même s’effectue en mode CLI ; l’interface graphique sert uniquement d’éditeur. Un seul appel produit l’exécution, les données brutes et le rapport :

```bash
jmeter -n -t gateway-lasttest.jmx -l lauf-10k.jtl -e -o report-10k
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-n` | Mode CLI : le plan de test s’exécute sans interface graphique, seul le Summariser écrit sur la console |
| `-t gateway-lasttest.jmx` | Le plan de test créé dans l’interface graphique |
| `-l lauf-10k.jtl` | Données brutes de l’exécution ; le rapport peut être régénéré ultérieurement à partir de ce fichier |
| `-e` | Génère le rapport immédiatement après l’exécution |
| `-o report-10k` | Répertoire cible du rapport HTML |

</details>

Le Summariser dans la console affiche l’évolution en direct, ainsi que le résultat final de l’exécution :

```text
summary =  10000 in 00:00:13 =  781.8/s Avg: 6 Min: 1 Max: 27 Err: 0 (0.00%)
```

10'000 messages en 12.8 secondes, 782 messages par seconde en moyenne, aucune erreur. Le puits a confirmé indépendamment exactement 10'000 e-mails acceptés avec le mélange 6000 / 1500 / 1000 / 1000 / 500 ; le ratio des Throughput Controller a donc été respecté au message près.

## Le rapport HTML

L’argument en faveur de JMeter face à des générateurs plus légers tels que smtp-source est l’évaluation, et le rapport de tableau de bord la fournit sans travail supplémentaire :

![Tableau de bord JMeter de l’exécution : APDEX 1.000 pour les cinq classes, Requests Summary avec 100 % PASS, tableau statistique avec percentiles par classe de messages](../images/jmeter-report-dashboard.png)

Le tableau statistique est la partie la plus importante du rapport. Pour chaque libellé de sampler, donc pour chaque classe de messages, il indique le nombre, le taux d’erreur, la moyenne, la médiane, les 90e, 95e et 99e percentiles, le maximum et le débit. Dans l’exécution concrète : médiane de 7 ms, p95 à 11 ms, p99 à 12 ms, maximum de 27 ms, de manière pratiquement identique pour les cinq classes. Avec un puits local qui traite chaque message de la même manière, c’est exactement le résultat attendu et en même temps la valeur de référence : si le même plan est ensuite exécuté contre la véritable passerelle et que la classe (sec) affiche soudainement un multiple de la médiane standard, il s’agit du travail supplémentaire du chemin de chiffrement, proprement isolé par branche de règles.

Le bloc APDEX au-dessus condense la même information en un chiffre par classe (ici partout 1.000, car toutes les réponses étaient très en dessous du seuil de tolérance de 500 ms) ; les seuils peuvent être adaptés à ses propres objectifs de service dans les propriétés du rapport. Le bloc Errors reste vide dans cette exécution, mais il constitue le premier point de départ lors de tests contre des systèmes réels : il regroupe les erreurs par texte de réponse, de sorte qu’une limitation `421` du système cible peut immédiatement être distinguée des interruptions de connexion.

Une erreur d’évaluation typique se présente également ici, et elle concerne toute courte rafale : par défaut, les graphiques temporels du rapport travaillent avec une granularité d’une minute. Une exécution de 13 secondes se réduit ainsi à un seul point de données, et les courbes sous « Charts » donnent l’impression d’une erreur de mesure. Le rapport peut être régénéré depuis le fichier JTL existant, sans nouvelle exécution, avec une résolution plus fine :

```bash
jmeter -g lauf-10k.jtl -o report-fein -Jjmeter.reportgenerator.overall_granularity=1000
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-g lauf-10k.jtl` | Génère le rapport à partir du fichier JTL existant, sans réexécuter le test |
| `-o report-fein` | Nouveau répertoire cible ; le répertoire du rapport existant reste inchangé |
| `-Jjmeter.reportgenerator.overall_granularity=1000` | Définit la granularité des graphiques pour cet appel à 1'000 ms au lieu de la minute par défaut |

</details>

Avec une granularité à la seconde, le point unique devient le profil de charge réel :

![Hits per Second avec une granularité d’une seconde : montée pendant les 10 secondes de ramp-up vers un plateau d’environ 840 messages par seconde, puis forte chute à la fin du test](../images/jmeter-report-hits-per-second.png)

La courbe montre la montée en charge de 10 secondes, un plateau d’environ 840 messages par seconde et la chute finale lorsque les premiers threads ont terminé leurs 500 boucles. Pour l’interprétation, c’est le plateau qui compte, et non la moyenne sur l’ensemble de l’exécution : la moyenne de 782/s inclut la montée en charge et la phase de sortie, et sous-estime le débit soutenu atteint.

## Ce que cette exécution démontre, et ce qu’elle ne démontre pas

Cette exécution démontre que le plan de test est fonctionnellement correct (exécution minimale avec contrôle du contenu au puits), que le ratio est exactement respecté et que le générateur atteint au moins 840 messages par seconde sur cette machine, sans TLS. Pour tester avec cela une passerelle conçue pour 100 e-mails par seconde, on dispose d’une réserve d’un facteur huit et l’on peut raisonnablement attribuer les goulots d’étranglement au système cible.

Tout le reste n’est pas démontré, et cette délimitation doit figurer dans chaque rapport de test : aucune affirmation sur le coût des handshakes TLS (le chemin réel utilise STARTTLS), aucune sur le comportement de la file d’attente de la passerelle, aucune sur le temps de traitement des chemins de règles. Pour cela, le même plan, avec les variables `zielhost`/`zielport` modifiées, pointe vers l’environnement de test de la passerelle ; l’évaluation se déroule alors de manière identique, complétée par les journaux de la passerelle et l’observation de la file d’attente de l’article de synthèse. C’est précisément cette réutilisabilité — un plan pour le puits, l’environnement de test et la préproduction avec une évaluation identique — qui justifie l’effort investi une fois dans un plan JMeter propre.

Une limite de l’outil lui-même doit également être mentionnée dans cette délimitation : JMeter ne peut pas maintenir des sessions SMTP ouvertes. Le SMTP Sampler ouvre une nouvelle connexion pour chaque message, parcourt EHLO, éventuellement STARTTLS et AUTH, puis la ferme après exactement une transaction avec QUIT. Les 840 messages par seconde comprennent donc un établissement complet de connexion par message. Un expéditeur de masse qui envoie des centaines de messages via une session ouverte génère sur la passerelle un autre profil de charge, avec moins de connexions et davantage de transactions par connexion, et les limites de connexions s’appliquent donc plus tôt avec la charge JMeter. La raison est l’architecture du framework : JMeter mesure chaque sampler comme une unité indépendante et autonome afin que les temporisateurs, assertions et percentiles fonctionnent de la même manière pour tous les protocoles pris en charge, et le SMTP Sampler est une surcouche de la bibliothèque JavaMail, qui se connecte et se déconnecte pour chaque envoi en tant qu’API cliente. Il n’existe pas de réutilisation de connexion pour SMTP comparable au Keep-Alive du sampler HTTP. Pour le profil de charge d’un expéditeur en masse avec session ouverte, `smtp-source` ou un script propre sont mieux adaptés ; la comparaison des outils dans l’article de synthèse le situe.

## Sources

1.  [Manuel utilisateur Apache JMeter : référence des composants, SMTP Sampler](https://jmeter.apache.org/usermanual/component_reference.html) : référence des champs du sampler, y compris les en-têtes, l’option d’horodatage et l’envoi EML.

2.  [Apache JMeter : génération du rapport de tableau de bord](https://jmeter.apache.org/usermanual/generating-dashboard.html) : génération du rapport HTML depuis l’exécution ou ultérieurement depuis le JTL, y compris les propriétés de granularité et APDEX.

3.  [Apache JMeter : plan de test, contrôleurs logiques](https://jmeter.apache.org/usermanual/component_reference.html#Throughput_Controller) : fonctionnement du Throughput Controller en mode Percent Executions pour le mélange de messages.

4.  [aiosmtpd, documentation](https://aiosmtpd.aio-libs.org/) : le serveur SMTP basé sur asyncio permettant de créer le puits en quelques lignes de Python.

5.  [Eclipse Temurin, Adoptium](https://adoptium.net/temurin/releases/) : archives JRE portables pour utiliser JMeter sans installation de Java.

6.  [Apache JMeter : prise en main, mode sans interface graphique](https://jmeter.apache.org/usermanual/get-started.html) : vue d’ensemble des options de ligne de commande pour le fonctionnement CLI, y compris `-n`, `-t`, `-l`, `-e`, `-o`, `-g` et `-J`.
