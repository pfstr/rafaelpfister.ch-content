---
title: "Jusqu’à 10 millions de tokens gratuits par jour : utiliser le programme de partage de données d’OpenAI avec des garde-fous de coûts"
navTitle: "Tokens gratuits OpenAI"
description: "OpenAI crédite les organisations qui autorisent l’utilisation de leur trafic API pour l’entraînement d’un quota quotidien gratuit : jusqu’à 10 millions de tokens selon le tier. Avec un crédit prépayé, des limites de projet et un budget de tokens dans le code, l’utilisation reste durablement gratuite."
date: "2026-08-27"
kategorie: "API OpenAI"
timeToRead: "9 min de lecture"
themen:
  - openai-api
produkte:
  - "openai"
protokolle:
  - "apis"
  - "lizenzierung"
slug: "jusqu-a-10-millions-de-tokens-gratuits-par-jour-utiliser-le-programme-de-partage-de-donnees-d"
translationId: "article-dde41cbe2dd858e6"
aiPrompt: |
  Du bist mein Assistent für die OpenAI-Plattform. Prüfe mit mir Schritt für Schritt, ob mein OpenAI-Konto für das Data-Sharing-Programm mit Gratis-Tokens sauber abgesichert ist: 1) Billing: Prepaid-Guthaben statt Rechnung, Auto-Reload aus. 2) Data controls → Sharing: "Share inputs and outputs" nur für ein dediziertes Projekt aktiviert, Enrollment-Hinweis sichtbar. 3) Projekt: eigenes Spend-Limit gesetzt, nur ein restricted API-Key. 4) Limits: Spend-Alerts konfiguriert. 5) Code: tägliches Token-Budget deutlich unter Gratis-Kontingent und Tages-Rate-Limit. Frage mich nach meinem Usage-Tier und Modell und rechne mir mein Gratis-Kontingent aus.
translationOf: openai-gratis-tokens-data-sharing
url: https://rafaelpfister.ch/fr/blog/jusqu-a-10-millions-de-tokens-gratuits-par-jour-utiliser-le-programme-de-partage-de-donnees-d
translationSourceHash: 0f0fef78a8ab264b755061045a34cc765916b1f1b433473f99a5eb6e0538a6b0
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:43:03.272Z
translationReview: automatic
---

# Jusqu’à 10 millions de tokens gratuits par jour : utiliser le programme de partage de données d’OpenAI avec des garde-fous de coûts

OpenAI rémunère les données d’entraînement avec de la puissance de calcul plutôt qu’avec de l’argent : depuis décembre 2024, les organisations qui autorisent l’utilisation de leurs entrées et sorties API pour l’entraînement reçoivent un quota quotidien de tokens gratuits. Selon le Usage Tier et le groupe de modèles, il va de 250'000 à 10 millions de tokens par jour. Pour de nombreuses petites automatisations, cela suffit entièrement : une traduction batch nocturne, une tâche cron de classification ou l’indexation automatique d’une archive publique restent ainsi durablement gratuites.

Pour que cela reste gratuit, il faut des limites, au bon endroit. Un compteur de tokens dans son propre code est une fonction de confort ; seules les limites appliquées par OpenAI lui-même sont contraignantes.

## Le programme : des tokens contre des données d’entraînement

La participation s’effectue via le réglage **Share inputs and outputs with OpenAI** sous *Settings → Data controls → Sharing*. Seul l’Organization Owner peut le modifier, soit pour toute l’organisation, soit pour certains projets. Les personnes éligibles au programme voient sur cette page le message « You're eligible for free daily usage on traffic shared with OpenAI » ; après l’activation, il devient « You're enrolled for complimentary daily tokens ». En l’absence de ce message, l’organisation n’est actuellement pas éligible. Les comptes avec Zero Data Retention et les contrats Enterprise sont exclus du partage des entrées et sorties.

Le quota dépend du Usage Tier de l’organisation et est calculé par groupe de modèles :

| Groupe de modèles | Tier 1–2 | Tier 3–5 |
|---|---|---|
| Grands modèles (dont gpt-5.6-sol, gpt-5.x, série o, gpt-4.1, gpt-4o) | 250'000 tokens/jour | 1 million de tokens/jour |
| Petits modèles (dont gpt-5.6-terra, gpt-5.6-luna, variantes mini et nano) | 2,5 millions de tokens/jour | 10 millions de tokens/jour |

Les règles principales en détail :

- Les tokens d’entrée et de sortie sont comptés ensemble, répartis entre tous les modèles d’un groupe. Le compteur est réinitialisé chaque jour à 00:00 UTC.
- Les modèles fine-tunés, l’entraînement de fine-tuning, les évaluations et l’utilisation d’outils sont exclus.
- Le compte doit disposer d’un solde positif, sans quoi même les tokens gratuits ne fonctionnent pas.
- OpenAI se réserve le droit de mettre fin au programme avec un préavis de 30 jours.

La règle de facturation la plus importante : la requête qui dépasse le quota quotidien est facturée **intégralement** au tarif normal, et non seulement pour la partie excédentaire. Si l’on est à 975'000 tokens sur 1 million et que l’on envoie une requête de 30'000 tokens, les 30'000 tokens sont tous facturés. Pour sa propre planification budgétaire, cela signifie : prévoir une marge de sécurité, ne pas optimiser jusqu’au quota.

## Ce que l’on cède en échange

La contrepartie est sans ambiguïté : toutes les entrées et sorties des projets partagés sont transmises à OpenAI et peuvent être utilisées pour entraîner de futurs modèles. Des catégories entières de cas d’utilisation sont donc exclues. Les données clients, tickets de support, documents internes, code contenant des détails de configuration et tout contenu à caractère personnel ne doivent pas parvenir à un projet partagé ; pour les entreprises suisses, la nLPD révisée fixe déjà cette limite, avant même d’aborder la confidentialité envers les clients.

Les charges de travail portant sur des données déjà publiques conviennent bien. Un exemple est la traduction nocturne d’un blog public en plusieurs langues : les articles sont en ligne, chaque crawler peut déjà les lire aujourd’hui, et les traductions sont elles aussi publiées. Dans un tel cas, le partage ne révèle rien qui ne le soit déjà. Parmi les autres candidats figurent les textes alternatifs d’une archive d’images publique, l’indexation d’une documentation open source ou des résumés de notes de version publiques pour un changelog.

## Configurer les garde-fous de coûts dans le compte OpenAI

L’ordre est délibéré : les limites appliquées côté serveur par OpenAI viennent en premier. Elles s’appliquent même si son propre code contient une erreur, si une tâche cron s’exécute deux fois ou si une clé tombe entre de mauvaises mains.

**Crédit prépayé, rechargement automatique désactivé.** Configurer la facturation sur « Pay as you go » avec un crédit prépayé et désactiver le rechargement automatique. Le dommage maximal est ainsi limité au crédit restant : une fois celui-ci épuisé, l’API refuse les requêtes supplémentaires. Le programme exigeant un solde positif, un petit montant de base est nécessaire ; 5 à 10 dollars suffisent et restent intacts en cas de fonctionnement propre. Cette étape est la seule qui arrête réellement tout dans le pire des cas, c’est pourquoi elle vient en premier.

**Un projet dédié au trafic partagé.** Régler le partage sur « Enabled for selected projects » et n’autoriser qu’un projet créé spécialement à cet effet. Tous les autres projets de l’organisation restent exclus de l’entraînement, et le trafic accidentel provenant d’autres applications ne se retrouve ni dans le jeu de données d’entraînement ni dans le mauvais budget.

**Définir une limite de dépenses de projet basse.** Les projets ont leur propre limite mensuelle de dépenses, et elle est stricte : les requêtes échouent dès qu’elle est atteinte. Pour un projet qui coûte normalement 0 dollar, elle peut être très basse ; 5 dollars suffisent comme réserve au cas où une exécution unique dépasse le quota gratuit. La limite au niveau de l’organisation est en revanche conçue comme un plafond avec alertes ; les seuils d’avertissement (par exemple à 90 et 100 %) déclenchent des e-mails.

**Une clé restreinte par projet, uniquement comme secret CI.** La clé API est créée dans le projet, et non au niveau de l’organisation, et ne reçoit que les autorisations requises par la charge de travail. Pour un workflow CI, cela signifie : exactement une clé aux droits restreints, enregistrée comme secret dans l’environnement CI. Elle n’apparaît dans aucun dépôt, aucun shell local ni aucun second service.

**Choisir un modèle du groupe économique.** La différence entre les groupes est d’un facteur 10. En Tier 1, un modèle du petit groupe donne 2,5 millions de tokens par jour au lieu de 250'000. Pour des tâches structurées telles que la traduction, la classification ou l’extraction, le petit groupe suffit généralement.

## La deuxième ligne de défense dans le code

Les limites du compte empêchent les dommages financiers, mais elles entraînent des erreurs brutales : une limite de dépenses atteinte interrompt l’exécution au milieu du batch. Pour rester proprement dans le quota gratuit, il est donc possible de compter soi-même en complément. Un simple compteur quotidien a fait ses preuves, configuré par exemple ainsi :

```json
{
  "openai": {
    "model": "gpt-5.6-terra",
    "reasoningEffort": "none",
    "maxOutputTokens": 32000,
    "dailyTokenBudget": 1000000
  }
}
```

Le mécanisme repose sur quatre règles :

- Après chaque réponse, la tâche ajoute les `input_tokens` et `output_tokens` signalés par l’API à un compteur dans un fichier d’état. Il n’y a ni estimation ni seconde requête, uniquement les indications d’usage de la réponse elle-même.
- Avant chaque requête, elle vérifie le budget restant. S’il ne suffit plus avec certitude pour une réponse complète, l’exécution se termine normalement avec le motif d’arrêt `token-budget` plutôt qu’avec une erreur.
- Le compteur fonctionne avec des jours calendaires UTC et est ainsi synchronisé avec la réinitialisation du quota gratuit à 00:00 UTC.
- Indépendamment du budget, le nombre d’appels API par exécution est plafonné afin qu’une série de tentatives échouées ne puisse pas non plus épuiser le quota. Les erreurs de transport et de quota interrompent l’exécution, sans répétition automatique.

Le budget de cet exemple, à 1 million, est volontairement nettement inférieur au quota de 2,5 millions. Cet écart découle de deux particularités de la facturation. Premièrement, le compteur ne connaît pas à l’avance la taille de la prochaine requête ; un budget calculé au plus juste peut donc être dépassé de la taille d’une requête, et cette requête serait précisément facturée dans son intégralité selon la règle décrite plus haut. Deuxièmement, les limites de débit quotidiennes (TPD) sont, selon le tier et le modèle, inférieures au quota gratuit ; un budget supérieur à la limite TPD ne serait jamais atteint normalement, car l’API rejetterait auparavant avec HTTP 429.

## Contrôle : le tableau de bord doit afficher 0.00

Le tableau de bord Usage de la plateforme indique si le calcul est correct. Deux vues suffisent :

- La vue **Usage** compte tous les tokens, y compris ceux facturés gratuitement. Elle indique la consommation totale de la charge de travail.
- La vue **Costs** (et le champ « Monthly spend » dans la liste des projets) n’affiche que les tokens payants. Elle doit rester durablement à 0.00.

Pour plus de précision, il est possible de regrouper la vue Usage par *Service tier* : les tokens facturés gratuitement y apparaissent comme un poste distinct, « data sharing incentive tier ». Ajouter une fois par mois une entrée au calendrier pour vérifier ce tableau de bord complète la chaîne de garde-fous, car OpenAI peut mettre fin au programme avec un préavis de 30 jours et, à partir de ce jour, la même charge de travail serait facturée au tarif normal.

## Sources

1.  [OpenAI Help Center: Sharing feedback, evaluation and fine-tuning data, and API inputs and outputs](https://help.openai.com/en/articles/10306912-sharing-feedback-evaluation-and-fine-tuning-data-and-api-inputs-and-outputs-with-openai): description de référence du programme avec les groupes de modèles, les quotas par tier, la réinitialisation UTC et la règle de facturation des requêtes dépassant le quota.

2.  [OpenAI Developer Community: Extended: Free tokens on traffic shared with OpenAI](https://community.openai.com/t/good-news-extended-free-tokens-on-traffic-shared-with-openai/1241322): annonce de la prolongation du programme en avril 2025, avec la garantie du préavis de 30 jours.

3.  [OpenAI Platform: Data sharing settings](https://platform.openai.com/settings/organization/data-controls/sharing): interrupteur d’opt-in et statut d’inscription de sa propre organisation (connexion requise).

4.  [OpenAI Platform: Rate limits guide](https://platform.openai.com/docs/guides/rate-limits): explication des limites TPM, RPM et TPD qui s’appliquent en plus du quota gratuit.

5.  [OpenAI Platform: Pricing](https://platform.openai.com/docs/pricing): tarifs normaux auxquels les dépassements de quota sont facturés.
