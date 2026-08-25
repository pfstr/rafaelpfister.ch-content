---
title: "Test de charge SMTP avec Apache JMeter en pratique : 10'000 e-mails, cinq chemins de règles, un rapport HTML"
navTitle: "Test de charge JMeter"
description: "Un test de charge réalisé de A à Z : plan de test avec un mix de messages suivant les chemins de ruleset d’une passerelle de chiffrement, configuration portable sans installation, 10'000 e-mails en rafale et analyse via le rapport HTML de JMeter, y compris les écueils réellement rencontrés."
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
url: https://rafaelpfister.ch/fr/blog/test-de-charge-smtp-avec-apache-jmeter-en-pratique-10-000-e-mails-cinq-chemins-de-regles-un
translationSourceHash: a41d58b7a4a717db179b3fec1ef8fac7961ff3ee12069f65627ddb48338aef0a
translationModel: gpt-5.6-terra
translatedAt: 2026-08-25T04:08:32.274Z
translationReview: required
---

# Test de charge SMTP avec Apache JMeter en pratique : 10'000 e-mails, cinq chemins de règles, un rapport HTML

L’[article d’aperçu sur les tests de charge de messagerie](/blog/mail-lasttest-tools-linux-windows-vergleich) a comparé les outils et esquissé le plan de test. Cet article passe à l’épreuve pratique : un test de charge JMeter entièrement réalisé avec 10'000 e-mails, un mix de messages suivant de véritables chemins de règles de passerelle et le rapport HTML comme outil d’analyse. Toutes les valeurs présentées proviennent de l’exécution réelle, y compris les erreurs survenues en cours de route.

Le scénario s’inspire d’un projet réel : une passerelle de chiffrement des e-mails basée sur Apache James (Totemomail) est placée comme boucle de smarthost derrière Exchange Online et décide pour chaque message du chiffrement, de la signature et du routage spécial. Le ruleset Mailet prévoit plusieurs chemins à cet effet : des déclencheurs dans l’objet tels que (sec), (sign) et (unsec), des mots-clés comme VERTRAULICH pour le routage vers une passerelle sectorielle, ainsi que le chemin standard avec vérification des certificats et repli en clair. Un test de charge qui n’injecterait qu’un seul type de message mesurerait toujours le même chemin dans ce jeu de règles ; le plan de test représente donc cinq classes dont la répartition correspond au trafic attendu.

## La configuration : aucune installation nécessaire

Le test a été exécuté sur une machine Windows sans Java ni JMeter. Les deux peuvent être utilisés de manière portable, ce qui est déterminant sur les postes d’administration aux droits d’installation limités : JRE Temurin au format ZIP depuis Adoptium, JMeter au format ZIP depuis apache.org, décompresser les deux, définir `JAVA_HOME` sur le répertoire de la JRE, et c’est prêt.

```bash
export JAVA_HOME="$PWD/jdk-21-jre"
export PATH="$JAVA_HOME/bin:$PATH"
./apache-jmeter-5.6.3/bin/jmeter -n -t gateway-lasttest.jmx -l lauf.jtl -e -o report
```

Une boîte noire SMTP locale basée sur aiosmtpd a servi de puits, avec un peu plus de 40 lignes de Python : elle accepte chaque message avec `250`, en rejette le contenu, les compte et attribue chaque e-mail à une classe à partir de sa ligne d’objet. Ce comptage indépendant côté réception constitue le contrôle du test ; si les nombres du générateur et du puits ne correspondent pas, quelque chose a été perdu en chemin.

```python
from aiosmtpd.controller import Controller

class SinkHandler:
    def __init__(self):
        self.count = 0

    async def handle_DATA(self, server, session, envelope):
        self.count += 1
        # Extraire l’objet de l’en-tête pour les statistiques par classe,
        # Le contenu est rejeté
        return "250 Message accepted for delivery"

controller = Controller(SinkHandler(), hostname="127.0.0.1", port=2525)
controller.start()
```

Point important pour l’interprétation : le générateur et le puits fonctionnaient sur la même machine, sans TLS et sans réseau entre les deux. Les chiffres mesurés ne disent donc rien sur une passerelle, mais constituent l’auto-test du générateur présenté dans l’article d’aperçu : la preuve que la configuration de charge peut effectivement produire le débit cible, et la limite supérieure à laquelle les mesures ultérieures seront comparées avec le véritable système de test.

## Le plan de test : cinq classes de messages, une répartition

Le cœur du plan est un Thread Group avec 20 threads, un ramp-up de 10 secondes et 500 boucles, soit 10'000 itérations. Il contient cinq Throughput Controllers en mode « Percent Executions », chacun avec exactement un SMTP Sampler :

| Classe (libellé du sampler) | Part | Chemin de règle dans la passerelle |
|---|---|---|
| 01 Standard sans déclencheur | 60 % | Vérification AutoGenerated, vérification des certificats, repli en clair |
| 02 Déclencheur (sec) | 15 % | Enveloppe TRE pour les destinataires sans certificat |
| 03 Déclencheur (sign) | 10 % | Certificate Exchange : signer, joindre la clé |
| 04 Mot-clé VERTRAULICH | 10 % | Routage spécial vers la passerelle sectorielle |
| 05 Déclencheur (unsec) | 5 % | Texte en clair forcé |

La répartition entre cinq samplers distincts plutôt qu’un sampler unique avec objet variable a une raison concrète : le rapport HTML regroupe tous les indicateurs selon le libellé du sampler. Cinq libellés produisent cinq lignes dans les statistiques avec leurs propres percentiles par classe ; un seul sampler avec un objet alimenté par CSV produirait une unique ligne agrégée, et la différence entre les chemins de règles serait invisible dans l’analyse.

Chaque sampler renseigne les champs habituels : hôte cible et port sous forme de variables définies par l’utilisateur (`${zielhost}`, `${zielport}`), afin que le même plan puisse être exécuté sans modification contre le puits, l’environnement de test ou la préproduction, ainsi que l’expéditeur, le destinataire, l’objet avec un marqueur clair (ici le mot LOADTEST dans l’objet) et un corps de texte d’environ 1 à 2 Ko. L’option « Include timestamp in subject » ajoute l’heure d’injection en millisecondes ; lors d’une exécution ultérieure contre un véritable système à plusieurs niveaux, elle permet de calculer, avec les heures de réception du puits, la latence de bout en bout par message.

Un écueil rencontré lors de cette exécution et généralisable : la première tentative a échoué avec 10'000 erreurs en 10 secondes, toutes avec `java.lang.ClassCastException ... NullProperty cannot be cast to ... CollectionProperty` au lieu d’une réponse SMTP. La cause était un fichier JMX construit à la main, dans lequel la liste des en-têtes du sampler manquait ; le sampler attend impérativement cette propriété, même vide. La leçon concerne moins la propriété précise que le schéma à suivre : construire et enregistrer les plans de test dans l’interface graphique plutôt que d’écrire le XML à la main, et effectuer une exécution minimale avant chaque rafale afin de vérifier côté puits que l’objet et le contenu arrivent réellement. Un compteur d’erreurs à 100 % avec un temps de réponse de 0 ms signifie presque toujours que l’erreur survient avant le réseau et que le test n’a donc jamais atteint le système cible.

## L’exécution

La mesure elle-même s’exécute en mode CLI ; l’interface graphique sert uniquement d’éditeur. Un seul appel génère l’exécution, les données brutes et le rapport :

```bash
jmeter -n -t gateway-lasttest.jmx -l lauf-10k.jtl -e -o report-10k
```

Le summariser dans la console affiche la progression en direct, ainsi que le résultat final de l’exécution :

```text
summary =  10000 in 00:00:13 =  781.8/s Avg: 6 Min: 1 Max: 27 Err: 0 (0.00%)
```

10'000 messages en 12.8 secondes, 782 messages par seconde en moyenne, aucune erreur. Le puits a confirmé indépendamment exactement 10'000 e-mails acceptés avec la répartition 6000 / 1500 / 1000 / 1000 / 500 ; la répartition des Throughput Controllers correspondait donc au message près.

## Le rapport HTML

L’argument en faveur de JMeter par rapport à des générateurs plus légers tels que smtp-source est l’analyse, et le rapport Dashboard la fournit sans travail supplémentaire :

![Tableau de bord JMeter de l’exécution : APDEX 1.000 pour les cinq classes, Requests Summary à 100 % PASS, tableau statistique avec percentiles par classe de messages](../images/jmeter-report-dashboard.png)

Le tableau statistique est la partie la plus importante du rapport. Pour chaque libellé de sampler, donc pour chaque classe de messages, il indique le nombre, le taux d’erreur, la moyenne, la médiane, les percentiles 90, 95 et 99, le maximum et le débit. Dans l’exécution concrète : médiane à 7 ms, p95 à 11 ms, p99 à 12 ms, maximum à 27 ms, de manière pratiquement identique pour les cinq classes. Avec un puits local qui traite tous les messages de la même manière, c’est exactement le résultat attendu et, simultanément, la valeur de référence : si le même plan est exécuté plus tard contre la véritable passerelle et que la classe (sec) présente soudainement un multiple de la médiane standard, cela correspond au travail supplémentaire du chemin de chiffrement, proprement isolé par branche de règle.

Le bloc APDEX au-dessus condense la même information en un chiffre par classe (ici 1.000 partout, car toutes les réponses étaient très en dessous du seuil de tolérance de 500 ms) ; les seuils peuvent être adaptés aux propres objectifs de service dans les propriétés du rapport. Le bloc Errors reste vide dans cette exécution, mais il est le premier point de départ lors des tests contre des systèmes réels : il regroupe les erreurs selon le texte de réponse, de sorte qu’une limitation `421` du système cible se distingue immédiatement des coupures de connexion.

Un autre écueil, qui concerne toute courte rafale : les graphiques de séries temporelles du rapport utilisent par défaut une granularité d’une minute. Une exécution de 13 secondes se réduit ainsi à un seul point de données, et les courbes sous « Charts » ressemblent à une erreur de mesure. Le rapport peut être régénéré à partir du fichier JTL existant, sans nouvelle exécution et avec une résolution plus fine :

```bash
jmeter -g lauf-10k.jtl -o report-fein -Jjmeter.reportgenerator.overall_granularity=1000
```

Avec une granularité à la seconde, le point unique devient le véritable profil de charge :

![Hits per Second avec une granularité d’une seconde : montée durant le ramp-up de 10 secondes jusqu’à un plateau d’environ 840 messages par seconde, puis baisse abrupte à la fin du test](../images/jmeter-report-hits-per-second.png)

La courbe montre le ramp-up de 10 secondes, un plateau autour de 840 messages par seconde et la baisse à la fin, lorsque les premiers threads ont achevé leurs 500 boucles. Pour l’interprétation, c’est le plateau qui compte, non la moyenne sur l’ensemble de l’exécution : la moyenne de 782/s inclut le ramp-up et la phase de fin, et sous-estime le débit soutenu atteint.

## Ce que cette exécution démontre, et ce qu’elle ne démontre pas

Cette exécution démontre que le plan de test est fonctionnellement correct (exécution minimale avec contrôle du contenu sur le puits), que la répartition est exacte et que le générateur atteint sur cette machine au moins 840 messages par seconde sans TLS. Quiconque souhaite ainsi tester une passerelle conçue pour 100 e-mails par seconde dispose d’une marge d’un facteur huit et peut attribuer les goulots d’étranglement au système cible en toute confiance.

Elle ne démontre rien d’autre, et cette délimitation doit figurer dans chaque rapport de test : aucune indication sur le coût des handshakes TLS (le véritable chemin utilise STARTTLS), aucune sur le comportement de la file d’attente de la passerelle, aucune sur le temps de traitement des chemins de règles. Le même plan, avec les variables `zielhost`/`zielport` pointant vers l’environnement de test de la passerelle, permet précisément cela ; l’analyse s’exécute alors à l’identique, complétée par les journaux de la passerelle et l’observation de la file d’attente de l’article d’aperçu. Cette réutilisabilité — un plan pour le puits, l’environnement de test et la préproduction avec une analyse identique — est la véritable raison d’investir une fois l’effort nécessaire dans un plan JMeter propre.

## Sources

1.  [Apache JMeter User's Manual: Component Reference, SMTP Sampler](https://jmeter.apache.org/usermanual/component_reference.html): référence des champs du sampler, y compris les en-têtes, l’option d’horodatage et l’envoi EML.

2.  [Apache JMeter: Generating Dashboard Report](https://jmeter.apache.org/usermanual/generating-dashboard.html): génération du rapport HTML à partir de l’exécution ou ultérieurement depuis le JTL, y compris les propriétés de granularité et d’APDEX.

3.  [Apache JMeter: Test Plan, Logic Controllers](https://jmeter.apache.org/usermanual/component_reference.html#Throughput_Controller): fonctionnement du Throughput Controller en mode Percent Executions pour le mix de messages.

4.  [aiosmtpd, documentation](https://aiosmtpd.aio-libs.org/): le serveur SMTP basé sur asyncio, qui permet de créer le puits en quelques lignes de Python.

5.  [Eclipse Temurin, Adoptium](https://adoptium.net/temurin/releases/): archives JRE portables pour utiliser JMeter sans installation de Java.
