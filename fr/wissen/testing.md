---
title: "Tests fonctionnels et tests de charge : vérifier l’infrastructure dans des conditions contrôlées"
blatt: "testing"
description: "Méthodologie pour les tests fonctionnels et de charge dans les environnements de messagerie et d’infrastructure : métriques, déroulement des tests avec baseline, burst et soak, générateurs de charge et puits, marquage des exécutions de test et évaluation par percentiles plutôt que par moyennes."
fakten:
  - label: Déroulement de base
    wert: Test fonctionnel → Baseline → Burst → Soak
    href: https://jmeter.apache.org/usermanual/best-practices.html
  - label: Quatre métriques
    wert: Taux d’acceptation · Latence de session · Latence de bout en bout · Comportement de la file d’attente
  - label: Évaluation
    wert: Percentiles p50 · p95 · p99, pas les moyennes
    href: https://sre.google/sre-book/monitoring-distributed-systems/
  - label: Générateurs de charge
    wert: smtp-source · Postal · JMeter · scripts propres
    href: https://www.postfix.org/smtp-source.1.html
  - label: Puits
    wert: smtp-sink · bhm · aiosmtpd · transport discard
    href: https://www.postfix.org/smtp-sink.1.html
  - label: Test de contrôle
    wert: comptage indépendant côté réception
  - label: Marquage
    wert: en-tête unique ou marqueur d’objet par exécution de test
  - label: Auto-test du générateur
    wert: une exécution contre le puits prouve le taux cible atteignable
  - label: Règle de base
    wert: uniquement des systèmes propres ou expressément autorisés
werbung:
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: f6d1ef180221f41242e1d32b5fb8fb2e631894e3ccd32a3b3322a33b808da210
translationModel: gpt-5.6-terra
translatedAt: 2026-10-14T09:39:29.339Z
translationReview: automatic
---

# Tests fonctionnels et tests de charge : vérifier l’infrastructure dans des conditions contrôlées

La capacité d’un environnement à supporter ses pics de charge, le comportement identique d’un ensemble de règles après une modification, ou la tenue d’un montage de stockage face à des schémas d’accès réels ne se vérifient pas dans une fiche technique, mais par des tests. Cette page rassemble la méthodologie commune à tous les types de tests : ce qui est mesuré, comment se déroule un test, quels outils génèrent et absorbent la charge, et comment transformer des données brutes en conclusions fiables.

La règle la plus importante précède toute méthodologie : les tests de charge et de perturbation doivent être effectués exclusivement sur ses propres systèmes ou dans des environnements expressément autorisés à cet effet. Un burst contre une infrastructure tierce est une attaque ; un test de messagerie avec des expéditeurs fictifs vers des cibles de production génère du backscatter et des inscriptions sur des listes de blocage.

## Séparer test fonctionnel et test de charge

Un test répond toujours à une seule question, et les deux questions fondamentales exigent des configurations différentes. Le **test fonctionnel** vérifie qu’un chemin fonctionne correctement : une transaction SMTP unique avec un contrôle total de chaque étape, un accès unique à un fichier, un parcours unique de règles à travers une passerelle. Le **test de charge** vérifie le comportement du système sous volume : débit, distribution des latences, files d’attente, consommation des ressources.

L’ordre est impératif : d’abord le test fonctionnel, puis la charge. Un burst sur un chemin défaillant ne mesure rien ; il ne produit que dix mille erreurs identiques. Un compteur d’erreurs à 100 % avec des temps de réponse proches de zéro signifie presque toujours que l’erreur se produit avant le réseau et que le test n’atteint jamais le système cible.

## Les quatre métriques

Les tests de charge dans des environnements à plusieurs niveaux mesurent quatre éléments différents, et chacun exige son propre angle d’analyse :

1. **Taux d’acceptation :** combien d’opérations par seconde le premier saut accepte-t-il ? C’est la valeur que les générateurs de charge fournissent directement.
2. **Latence de session :** combien de temps dure une transaction individuelle ? Sous charge, cette valeur augmente souvent bien avant que le taux d’acceptation ne s’effondre.
3. **Latence de bout en bout :** combien de temps une opération met-elle pour atteindre la cible en passant par toutes les étapes intermédiaires ? C’est la métrique perçue par les utilisateurs.
4. **Comportement de la file d’attente :** jusqu’à quel niveau la file d’attente augmente-t-elle pendant le pic, et à quelle vitesse se vide-t-elle ensuite ? Un système qui accepte tout et traite les opérations pendant des heures réussit le test d’acceptation tout en échouant malgré tout.

## Déroulement du test : baseline, burst, soak

Un test fiable se déroule en trois étapes, chacune avec sa propre question. La **baseline** établit, avec une charge modérée et connue, les valeurs de référence pour la latence et la consommation des ressources, et révèle les erreurs de configuration avant qu’elles ne disparaissent dans la mesure de pointe. Le **burst** est la véritable mesure de charge de pointe ; plusieurs exécutions avec un parallélisme croissant montrent où le taux d’acceptation plafonne et où la latence augmente fortement. Le **soak** applique la charge continue attendue pendant des heures ; c’est seulement ici que se révèlent les fuites de mémoire, les partitions de spool saturées et les limites de connexion qu’un burst court n’atteint jamais.

Trois règles pratiques s’y ajoutent. Premièrement, l’**auto-test du générateur** : avant la première exécution contre le système cible, le générateur est exécuté directement contre le puits ; cela prouve que la configuration peut effectivement produire le taux cible et fournit la limite supérieure à laquelle comparer les mesures ultérieures. Deuxièmement, le **marquage** : chaque opération de test porte un marqueur univoque (en-tête propre, convention d’objet, numéro d’exécution) afin de séparer clairement les exécutions dans les journaux et de pouvoir supprimer intégralement les données de test par la suite. Troisièmement, le **test de contrôle** : un comptage indépendant côté réception confirme que ce que le générateur a envoyé est bien arrivé ; si les nombres ne correspondent pas, quelque chose s’est perdu en chemin.

## Outils : générateur et puits

Un test de charge se compose toujours de trois éléments : générateur de charge, système de test et puits contrôlé. Pour la charge SMTP, les générateurs courants sont [smtp-source](https://www.postfix.org/smtp-source.1.html) du paquet Postfix (charge brute maximale avec un effort minimal), [Postal](https://doc.coker.com.au/projects/postal/) (charge continue avec un taux cible et des messages variables), [Apache JMeter](https://jmeter.apache.org/) (profils de charge, percentiles et rapport HTML, premier choix sous Windows) et des scripts propres lorsque le test nécessite des mélanges d’opérations réalistes. [swaks](https://www.jetmore.org/john/code/swaks/) n’est pas un générateur de charge, mais doit servir de testeur fonctionnel avant chaque burst.

Côté réception, un puits accepte tout et rejette tout : [smtp-sink](https://www.postfix.org/smtp-sink.1.html), le `bhm` fourni avec Postal ou un petit serveur [aiosmtpd](https://aiosmtpd.aio-libs.org/) en quelques lignes de Python, qui effectue aussi le comptage de contrôle. Les outils de développement avec interface Web ne conviennent pas comme puits pour les tests de charge ; avec des dizaines de milliers d’opérations, c’est sinon l’outil d’analyse qui est mesuré plutôt que le système de test.

## Évaluation : percentiles, pivot, évolution

Les données brutes ne deviennent des conclusions qu’à travers trois étapes d’évaluation. **Percentiles plutôt que moyennes :** une moyenne de deux secondes peut signifier que tout dure deux secondes, ou que 95 % des opérations se terminent en une demi-seconde et que le reste a pris nettement plus longtemps ; p50, p95 et p99 distinguent ces cas. **Pivoter les erreurs :** la distribution des codes d’erreur dans le temps montre quand un système commence à limiter et quel mécanisme de protection intervient en premier. **Évolution plutôt qu’état final :** le débit et la profondeur de la file d’attente représentés dans le temps montrent la montée en charge, le plateau et la décroissance ; pour l’interprétation, c’est le plateau qui compte, pas la moyenne sur toute l’exécution, car celle-ci inclut la montée et la décroissance et sous-estime le taux continu atteint.

Enfin, l’évaluation comprend une délimitation honnête : chaque rapport de test indique ce que l’exécution démontre et ce qu’elle ne démontre pas. Une mesure sans TLS ne dit rien des coûts de négociation TLS ; un puits local ne dit rien du comportement de la file d’attente de la véritable cible ; un burst ne dit rien des fuites de mémoire en fonctionnement continu. Ces limites définissent les prochaines exécutions de test.

Les articles ci-dessous appliquent la méthodologie à des cas concrets : comparaison d’outils et planification de tests pour les tests de charge de messagerie, une exécution JMeter complète avec rapport HTML, diagnostic SMTP par couches avec les outils intégrés, un plan de test pour les montages de stockage cloud et la méthode scientifique derrière les expériences contrôlées dans le flux de messagerie.
