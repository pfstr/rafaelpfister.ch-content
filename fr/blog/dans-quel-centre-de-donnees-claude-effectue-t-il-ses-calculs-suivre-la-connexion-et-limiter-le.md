---
title: "Dans quel centre de données Claude effectue-t-il ses calculs ? Suivre la connexion et limiter le traitement à la Suisse ou à l’UE"
navTitle: "Centre de données Claude"
description: "Quelles étapes une requête à Claude traverse, pourquoi la trace s’arrête à l’edge Cloudflare et ce qu’Anthropic révèle sur ses centres de données. Comparaison des parcours avec un lieu de traitement fixe (API Anthropic, AWS Bedrock, Google Vertex AI, Microsoft Foundry), limites d’une exploitation exclusivement suisse et coût du rattachement régional."
date: "2026-10-02"
kategorie: "Claude"
timeToRead: "12 min de lecture"
themen:
  - claude
produkte:
  - "claude"
protokolle:
  - "apis"
  - "tcp"
slug: "dans-quel-centre-de-donnees-claude-effectue-t-il-ses-calculs-suivre-la-connexion-et-limiter-le"
translationId: "article-d8b7299589ea96f4"
translationOf: claude-rechenzentrum-schweiz-eu
translationSourceHash: 93ccc2c90b4f24a12ada6d9fd9417465b4e821e9d371fa5d045ed64cfd70e07c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-19T14:49:50.777Z
translationReview: required
url: https://rafaelpfister.ch/fr/blog/dans-quel-centre-de-donnees-claude-effectue-t-il-ses-calculs-suivre-la-connexion-et-limiter-le
---

Quiconque utilise Claude via claude.ai, Claude Code ou l’API ne sait pas dans quel centre de données le modèle traite sa requête. Seule la première partie de la connexion, jusqu’au nœud edge le plus proche, est visible. Pour les entreprises qui transmettent des données personnelles ou des contenus confidentiels à un modèle de langage, cela ne suffit pas : elles doivent pouvoir démontrer à leurs clients, conseillers en protection des données ou à l’audit où le traitement a lieu.

Cet article montre jusqu’où il est possible de suivre la connexion elle-même, ce qui est publiquement connu des centres de données d’Anthropic et par quelles voies le traitement peut être limité de manière contraignante à l’UE ou à l’espace UE plus Suisse. Les prix et régions sont ceux en vigueur au 2 octobre 2026.

**Focus Suisse :** la loi révisée sur la protection des données (nLPD) est déterminante. La communication de données personnelles à l’étranger est autorisée selon l’art. 16 nLPD si le Conseil fédéral atteste un niveau de protection adéquat dans l’État destinataire (annexe 1 de l’ordonnance sur la protection des données, OPDo) ou s’il existe des garanties appropriées, telles que des clauses contractuelles types. Les États de l’UE et de l’EEE figurent sur cette liste ; les États-Unis n’y figurent que pour les entreprises certifiées selon le Swiss-U.S. Data Privacy Framework. Un lieu de traitement dans l’UE simplifie donc considérablement la justification, sans remplacer les autres obligations (contrat de sous-traitance, information des personnes concernées, sécurité des données).

**Remarque UE :** pour les entreprises établies dans l’UE, les art. 44 et suivants du RGPD s’appliquent à la place. La Commission européenne a reconnu à la Suisse un niveau de protection des données adéquat (décision d’adéquation 2000/518/CE, confirmée en janvier 2024) ; un traitement à Zurich ne constitue donc pas, pour les entreprises de l’UE, un transfert vers un pays tiers au sens problématique.

## Quelles étapes une requête à Claude traverse

Une requête vers `claude.ai` ou `api.anthropic.com` passe par trois étapes :

| Étape | Exploitant | Visible pour vous ? |
|---|---|---|
| Nœud edge (terminaison TLS, protection contre les abus) | Cloudflare, au nom d’Anthropic | Oui, emplacement lisible dans l’en-tête |
| Réseau interne jusqu’au backend | Cloudflare et Anthropic | Non |
| Inférence (le modèle calcule) | Anthropic, sur des ressources de calcul d’AWS, Google Cloud et de ses propres centres de données | Non, uniquement l’indication générale `us` ou `global` |

Les noms d’hôte `claude.ai` et `api.anthropic.com` renvoient tous deux vers l’adresse `160.79.104.10`. Elle appartient au préfixe `160.79.104.0/23`, enregistré au nom d’Anthropic (AS399358) et dont l’origine Anthropic est validée selon RPKI. Dans le routage mondial, le préfixe est toutefois annoncé exclusivement via Cloudflare (AS13335) : Anthropic intègre ses propres adresses IP au réseau Anycast de Cloudflare. La même adresse répond ainsi depuis chaque site Cloudflare dans le monde, et la connexion aboutit au nœud edge le plus proche.

## Déterminer le nœud edge lui-même

Cloudflare révèle le site qui répond dans une page de diagnostic et dans l’en-tête `cf-ray`. L’abréviation à la suite de l’identifiant Ray est un code d’aéroport IATA : `ZRH` désigne Zurich, `GVA` Genève, `FRA` Francfort et `MRS` Marseille.

```bash
curl -s https://api.anthropic.com/cdn-cgi/trace
```

Les lignes pertinentes dans la sortie sont `colo=` (emplacement edge) et `loc=` (pays auquel Cloudflare rattache votre adresse source). Sous Windows, PowerShell fournit le même résultat :

```powershell
$trace = Invoke-WebRequest -Uri "https://api.anthropic.com/cdn-cgi/trace" -UseBasicParsing
$trace.Content
```

Vous lisez l’en-tête avec une requête HEAD :

```bash
curl -sI https://api.anthropic.com/ \
  | grep -i -E '^(server|cf-ray):'
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-s` | Masque l’indicateur de progression et les messages d’erreur de curl |
| `-I` | Envoie une requête HEAD et affiche uniquement les en-têtes de réponse |
| `grep -i` | Recherche sans tenir compte de la casse |
| `-E '^(server\|cf-ray):'` | Expression régulière étendue : uniquement les lignes commençant par `server:` ou `cf-ray:` |

</details>

Depuis un réseau suisse, il faut généralement s’attendre à `ZRH` ou `GVA`. Si la sortie Internet se trouve ailleurs, par exemple dans un réseau d’entreprise doté d’une sortie centralisée à l’étranger, le nœud edge répondra là-bas, par exemple avec `colo=FRA` ou `colo=MRS`. L’emplacement edge suit donc votre sortie réseau, et non le lieu du traitement.

## Pourquoi le traceroute s’arrête à l’edge

Un traceroute montre le chemin jusqu’au nœud edge, et pas au-delà :

```powershell
Test-NetConnection -ComputerName api.anthropic.com -TraceRoute -Hops 20
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-ComputerName api.anthropic.com` | Hôte cible de la mesure |
| `-TraceRoute` | Détermine les sauts de routeurs sur le chemin vers la cible |
| `-Hops 20` | Nombre maximal de sauts à vérifier |

</details>

Sous Linux et macOS, l’équivalent est `traceroute -n api.anthropic.com` (`-n` désactive la résolution DNS des sauts). Lors de la mesure, les derniers sauts avant `160.79.104.10` se situaient dans la plage d’adresses `162.158.0.0/15`, qui appartient à Cloudflare. La trace s’arrête ensuite : la connexion est terminée au nœud edge, et le transfert vers le backend s’effectue dans une nouvelle connexion interne, invisible de l’extérieur.

RIPEstat indique à qui appartient le préfixe et par quel réseau il est annoncé :

```bash
curl -s "https://stat.ripe.net/data/prefix-overview/data.json?resource=160.79.104.10"
curl -s "https://stat.ripe.net/data/bgp-state/data.json?resource=160.79.104.0/23"
```

Dans le second résultat, tous les chemins AS se terminent par `13335 399358` : Cloudflare est le seul upstream du préfixe Anthropic.

D’après mes recherches, il n’existe aucune mesure publique permettant de retracer le chemin jusqu’à un centre de données d’inférence ; en raison de la terminaison à l’edge, cela n’est pas non plus possible avec des outils réseau. Des services de mesure comme llmlatency.dev ne mesurent que le temps de réponse jusqu’au premier octet à l’edge (en septembre 2026 : médiane de 96 ms depuis le centre des États-Unis, 199 ms depuis l’Allemagne). Il serait certes possible de comparer le temps jusqu’au premier token (Time to First Token) depuis plusieurs emplacements, mais celui-ci dépend davantage du modèle, de la longueur du prompt et de la charge que de la distance. Il n’est donc pas possible d’en déduire de façon fiable un emplacement.

## Ce que l’on sait des centres de données d’Anthropic

Anthropic n’exploite jusqu’ici aucun centre de données d’inférence propre publiquement associé à une adresse. La capacité de calcul provient de partenariats recensés par Tim Cadenbach dans le blog TCDEV :

| Site / partenaire | Caractéristiques | Rôle connu |
|---|---|---|
| Project Rainier, New Carlisle (Indiana, États-Unis), Amazon Web Services | Ouvert en octobre 2025, environ 11 milliards USD, quelque 500 000 puces Trainium2, plus de 2,2 GW prévus | Anthropic est locataire principal ; surtout entraînement |
| Google Cloud | Accord d’octobre 2025 portant sur jusqu’à 1 million de TPU, plus de 1 GW à partir de 2026 | Régions non publiées |
| Centres de données propres avec Fluidstack | Annoncés en novembre 2025, 50 milliards USD, sites au Texas et à New York, mise en service dès 2026 | Encore en construction |

L’entraînement et l’inférence ne se déroulent pas nécessairement au même endroit. L’inférence peut avoir lieu dans n’importe quelle région AWS ou Google Cloud, et Anthropic ne révèle pas quel site calcule la réponse pour un utilisateur en Suisse. Seul ce que la déclaration de confidentialité indique est établi : Anthropic Ireland, Limited à Dublin est le partenaire contractuel et responsable pour les clients de l’EEE, du Royaume-Uni et de Suisse ; les données sont transférées vers des serveurs aux États-Unis ou dans d’autres pays hors EEE, sur la base de clauses contractuelles types.

## Le pilotage direct chez Anthropic : États-Unis ou mondial uniquement

Depuis les modèles de génération 4.6, l’API Claude connaît le paramètre `inference_geo`. Il ne comporte précisément que deux valeurs :

| Valeur | Effet | Prix |
|---|---|---|
| `global` (par défaut) | Inférence dans toute région disponible | Prix catalogue |
| `us` | Inférence exclusivement aux États-Unis | Prix catalogue × 1,1 |

Il n’existe pas de valeur pour l’UE ou la Suisse. Le lieu de stockage du workspace (`workspace geo`) ne peut actuellement être défini que sur `us`. La réponse indique dans le champ `usage.inference_geo` le réglage appliqué, mais aucune région précise. Pour Haiku 4.5 et les modèles plus anciens, l’API renvoie une erreur 400 pour ce paramètre.

Pour claude.ai et Claude Code avec un abonnement (Pro, Max, Team, Enterprise), aucun réglage du lieu de traitement n’est disponible. Les offres directes d’Anthropic ne permettent donc de limiter le traitement ni à l’UE ni à la Suisse. Cela n’est possible qu’en passant par les fournisseurs cloud qui exploitent Claude dans leurs propres régions.

## AWS Bedrock : espace UE plus Suisse depuis Zurich

AWS Bedrock exploite Claude sur l’infrastructure AWS. Selon AWS, Anthropic n’a pas accès aux prompts, réponses ou logs des clients. Vous choisissez la région via la région source de votre appel API et via l’Inference Profile :

| Type de point de terminaison | Préfixe de l’ID de modèle | Lieu de traitement |
|---|---|---|
| Global Cross-Region Inference | `global.` | Toute région AWS commerciale dans le monde |
| Geographic Cross-Region Inference | `eu.` | Uniquement les régions situées dans la zone géographique |
| In-Region | sans préfixe | Uniquement la région appelée |

Pour la Suisse, la région `eu-central-2` (Zurich) est déterminante. Opus 5.5, Sonnet 5.5 et Haiku 4.5 y sont disponibles, mais uniquement en profil Global ou EU et non In-Region. Il n’existe aucun profil calculant exclusivement à Zurich. Lorsqu’un profil EU est appelé depuis Zurich, AWS répartit les requêtes, selon la carte du modèle (documenté pour Haiku 4.5 et Sonnet 4.6), entre les régions suivantes :

| Région | Emplacement |
|---|---|
| `eu-central-2` | Zurich |
| `eu-central-1` | Francfort |
| `eu-north-1` | Stockholm |
| `eu-south-1` | Milan |
| `eu-south-2` | Espagne |
| `eu-west-1` | Irlande |
| `eu-west-3` | Paris |

Zurich n’est une destination possible que lorsque la requête provient de Zurich. Le traitement reste ainsi dans l’espace UE plus Suisse. Pour Opus 5.5 et Sonnet 5.5, la carte du modèle ne mentionne pas les régions cibles ; l’API fournit la liste effective :

```bash
aws bedrock get-inference-profile \
  --region eu-central-2 \
  --inference-profile-identifier eu.anthropic.claude-sonnet-5-5 \
  --query "models[].modelArn"
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `bedrock get-inference-profile` | Lit la définition d’un Inference Profile |
| `--region eu-central-2` | Région source Zurich ; la liste des destinations dépend de la région source |
| `--inference-profile-identifier` | ID du profil EU pour Sonnet 5.5 |
| `--query "models[].modelArn"` | Affiche uniquement les ARN de modèles ; la région figure dans chaque ARN |

</details>

Par défaut, les données ne sont stockées que dans la région source. Les contenus conservés pour la détection des abus constituent une exception : ils se trouvent dans la région cible. Le transport entre les régions est chiffré via le réseau AWS.

### Prouver la région de traitement pour chaque requête

Bedrock est la seule des voies décrites qui consigne la région de traitement effective pour chaque requête. CloudTrail l’écrit dans la région source, dans le champ `additionalEventData.inferenceRegion` :

```bash
aws cloudtrail lookup-events \
  --region eu-central-2 \
  --lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock.amazonaws.com \
  --max-results 20 \
  --query "Events[].CloudTrailEvent" \
  --output text \
  | jq -r '[.eventTime, .eventName, .additionalEventData.inferenceRegion] | @tsv'
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `cloudtrail lookup-events` | Recherche les événements CloudTrail des 90 derniers jours |
| `--region eu-central-2` | Région source dans laquelle les appels sont consignés |
| `--lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock.amazonaws.com` | Filtre les événements Bedrock |
| `--max-results 20` | Limite la sortie à 20 événements |
| `--query "Events[].CloudTrailEvent"` | Affiche uniquement l’événement complet sous forme de texte JSON |
| `--output text` | Sortie sans enveloppe JSON, un événement par ligne |
| `jq -r '… \| @tsv'` | Extrait l’horodatage, l’action et la région de traitement dans une ligne séparée par des tabulations |

</details>

En outre, une Service Control Policy (SCP) dans AWS Organizations permet de bloquer toutes les régions hors UE et Suisse. Lorsqu’une région cible d’un profil est bloquée, la requête échoue au lieu de basculer vers une autre région. Cette combinaison impose donc techniquement le lieu de traitement et permet de le démontrer pour chaque requête.

### Claude Code via Bedrock dans l’UE

Claude Code peut également être exploité via Bedrock. Dans une région `eu-*`, Claude Code sélectionne automatiquement le préfixe `eu.` :

```bash
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=eu-central-2
claude
```

| Variable | Effet |
|---|---|
| `CLAUDE_CODE_USE_BEDROCK=1` | Bascule Claude Code de l’API Anthropic vers Bedrock |
| `AWS_REGION=eu-central-2` | Région source Zurich ; il en découle le profil EU |

Le préfixe peut être remplacé avec `ANTHROPIC_BEDROCK_REGION_PREFIX`. L’authentification AWS s’effectue via les mécanismes habituels (profil, SSO, variables d’environnement). La facturation passe par le compte AWS, et non par un abonnement Claude.

## Google Vertex AI : uniquement l’UE, explicitement sans la Suisse

Sur Google Cloud (Vertex AI, désormais sous le nom Gemini Enterprise Agent Platform), la situation est moins favorable pour la Suisse :

| Point de terminaison | Modèles (sélection) | Lieu de traitement |
|---|---|---|
| `global` | tous | Toute région Google Cloud, sans garantie |
| Multi-région `eu` (`aiplatform.eu.rep.googleapis.com`) | Opus 5.5, Sonnet 5.5, Opus 5, Sonnet 5, Opus 4.8, Opus 4.7 | États membres de l’UE uniquement |
| Région `europe-west1` (Belgique) | Haiku 4.5, Sonnet 4.6, Opus 4.6 et versions antérieures | Selon la page du modèle, multi-région Europe |
| Région `europe-west6` (Zurich) | aucun | Claude non disponible |

Google exclut explicitement la Suisse du point de terminaison multi-région UE : il ne couvre que les États membres de l’UE ; le Royaume-Uni et la Suisse n’en font pas partie. Plusieurs pages de conseil suisses citent `europe-west6` comme voie vers Claude à Zurich ; selon le tableau officiel des emplacements de Google, aucun modèle Claude n’y est disponible. Google ne documente pas de champ de journalisation indiquant la région de traitement effective par requête. Pour le point de terminaison mondial, Google précise qu’il est impossible de piloter ou de déterminer la région de traitement.

Vertex AI convient donc à un traitement exclusivement dans l’UE, mais pas à un traitement en Suisse.

## Microsoft Foundry : aucune option UE à ce jour

Microsoft propose Claude dans Foundry en deux variantes. « Hosted on Azure » calcule sur l’infrastructure Azure, en Global Standard ou Data Zone Standard ; la Data Zone n’existe pour Claude qu’aux États-Unis. « Hosted on Anthropic » calcule sur l’infrastructure d’Anthropic, et Microsoft indique que les données peuvent être traitées hors d’Azure et hors de la région sélectionnée. Anthropic indique que Foundry pour l’Europe est « Coming soon ». Dans Microsoft 365 Copilot et Copilot Studio, les modèles Anthropic sont exclus de l’EU Data Boundary et désactivés par défaut dans l’UE, l’AELE (donc aussi en Suisse) et au Royaume-Uni.

## Vue d’ensemble : quelle voie garantit quel lieu de traitement

| Voie | Suisse uniquement | UE uniquement | Espace UE plus Suisse | Région prouvable par requête |
|---|---|---|---|---|
| claude.ai, Claude Code (abonnement) | non | non | non | non |
| API Claude avec `inference_geo` | non | non | non (uniquement `us`) | uniquement `us`/`global` |
| AWS Bedrock, source `eu-central-2`, profil `eu.` | non | non | oui | oui (CloudTrail) |
| AWS Bedrock, source dans l’UE, profil `eu.` | non | oui | oui | oui (CloudTrail) |
| Google Vertex AI, point de terminaison `eu` | non | oui | (partie UE uniquement) | non |
| Microsoft Foundry | non | non | non | non |

Aucun fournisseur ne propose actuellement de voie garantissant que le traitement de Claude soit limité à la Suisse. AWS Bedrock avec la région source Zurich et un profil EU s’en approche le plus. S’il faut en outre exclure que les données quittent l’UE, par exemple en raison d’engagements contractuels envers des clients de l’UE, le profil EU peut être appelé depuis une région UE telle que Francfort ; Zurich ne sera alors plus une destination.

## Anonymisation avant le modèle : entreprises pilotes recherchées

Dans la plupart des cas, le lieu de traitement n’est pertinent que parce que le prompt contient des données personnelles ou des informations confidentielles. C’est là qu’intervient un nouveau produit de mon entreprise, adeptio gmbh : il se place comme couche intermédiaire entre les applications en Suisse et les modèles d’Anthropic et d’OpenAI, et anonymise les requêtes avant qu’elles ne quittent la Suisse. Le modèle ne reçoit plus que des contenus sans lien avec des personnes, quel que soit le centre de données dans lequel il calcule.

**Focus Suisse :** les informations qui ne peuvent plus être attribuées à une personne déterminée ou déterminable ne sont pas des données personnelles au sens de l’art. 5, let. a nLPD. Les règles relatives à la communication à l’étranger (art. 16 s. nLPD) ne s’appliquent pas à de tels contenus. La question de savoir si une anonymisation atteint ce seuil dépend de sa mise en œuvre et des données concrètes ; c’est précisément ce que nous voulons démontrer en exploitation réelle.

Pour le lancement, nous recherchons des entreprises qui utilisent Claude ou des modèles d’OpenAI en production, ou qui prévoient de les introduire, afin qu’elles utilisent le produit comme entreprise pilote au quotidien. Les deux parties en bénéficient :

| Votre entreprise | adeptio |
|---|---|
| Les données personnelles et informations confidentielles ne quittent la Suisse qu’après anonymisation. La justification auprès des conseils en protection des données, des clients et de l’audit est simplifiée. | Des enseignements tirés de prompts et de processus de travail réels plutôt que de données de test : quelles informations doivent être reconnues et où l’anonymisation retire trop ou trop peu. |
| Le choix entre Anthropic, OpenAI, Bedrock ou Vertex AI repose à nouveau sur l’offre de modèles et le prix, plutôt que sur le seul lieu de traitement. | Des mesures de latence et de qualité des réponses sous charge réelle. |
| Accès au produit avant sa disponibilité générale et influence sur les types de données et cas d’utilisation qu’il couvrira en premier. | Des retours des équipes de protection des données et d’audit, auxquels les preuves fournies par le produit seront alignées. |
| Contact direct avec les développeurs au lieu d’un support par tickets. | Des références du marché suisse pour la disponibilité générale. |

<aside class="offer-box">
  <span class="offer-box__tag">Entreprises pilotes recherchées</span>
  <p><strong>Utilisez-vous Claude ou OpenAI avec des données personnelles ?</strong> Utilisez la couche d’anonymisation d’adeptio en exploitation réelle et participez à son évolution.</p>
  <a class="offer-box__cta" href="/kontakt">Prendre contact</a>
</aside>

## Ce que coûte le rattachement régional

Les trois fournisseurs cloud et Anthropic lui-même appliquent un supplément de 10 % pour un rattachement à une région ou à une zone géographique par rapport au point de terminaison mondial. Chez Bedrock, les prix dépendent de la région source ; Zurich, Francfort et North Virginia coûtent le même prix. Le routage entre les régions ne fait pas l’objet d’un coût supplémentaire.

Prix catalogue en USD par million de tokens (entrée / sortie) :

| Modèle | API Anthropic, `global` | API Anthropic, `us` | Bedrock ou Vertex, global | Bedrock `eu.` ou Vertex `eu` |
|---|---|---|---|---|
| Opus 5.5 | 4.00 / 20.00 | 4.40 / 22.00 | 4.00 / 20.00 | 4.40 / 22.00 |
| Sonnet 5.5 | 2.00 / 10.00 | 2.20 / 11.00 | 2.00 / 10.00 | 2.20 / 11.00 |
| Haiku 4.5 | 1.00 / 5.00 | non disponible | 1.00 / 5.00 | 1.10 / 5.50 |
| Sonnet 4.6 | 3.00 / 15.00 | 3.30 / 16.50 | 3.00 / 15.00 | 3.30 / 16.50 |

Dans Vertex AI, le prix de la colonne UE s’applique à Haiku 4.5 et Sonnet 4.6 dans la région `europe-west1`. Exemple de calcul pour une application interne consommant 50 millions de tokens d’entrée et 10 millions de tokens de sortie par mois :

| Modèle | Global | Lié à l’UE | Surcoût mensuel |
|---|---|---|---|
| Sonnet 5.5 | 50 × 2 + 10 × 10 = 200 USD | 220 USD | 20 USD |
| Opus 5.5 | 50 × 4 + 10 × 20 = 400 USD | 440 USD | 40 USD |

Le rattachement régional coûte donc 10 % de plus que le point de terminaison mondial. Par rapport au prix catalogue directement chez Anthropic avec `inference_geo: "us"`, un profil EU sur Bedrock coûte le même prix. Les coûts annexes pèsent davantage dans le choix : un compte AWS ou Google Cloud avec politiques d’organisation, journalisation et alertes budgétaires doit être mis en place et exploité, tandis que les prix mensuels fixes des abonnements Claude sont remplacés par une facturation purement à l’usage. La rentabilité du changement dépend du volume ; un abonnement a un prix fixe par personne, mais n’offre aucun contrôle sur le lieu de traitement.

## Conservation et entraînement

Outre le lieu, la durée de conservation des données compte :

- **API Anthropic :** les prompts et réponses ne sont pas utilisés pour l’entraînement sans consentement explicite. La Zero Data Retention (ZDR) est disponible sur demande pour chaque organisation et peut être combinée à `inference_geo`. Les contenus signalés par la détection des abus restent stockés jusqu’à deux ans, même avec ZDR. Pour les modèles Fable et Mythos, une conservation de 30 jours est obligatoire.
- **AWS Bedrock :** les contenus ne sont pas transmis à Anthropic. La conservation peut être choisie par modèle entre `none`, `default` et `aws_review`; Fable exige `aws_review` avec jusqu’à 30 jours de conservation pour examen par AWS.
- **Google Vertex AI :** Google utilise les données pour l’entraînement ou le fine-tuning uniquement avec consentement préalable.

## Sources

1.  [Claude Platform Docs: Data residency](https://platform.claude.com/docs/en/manage-claude/data-residency): paramètre `inference_geo`, valeurs `us` et `global`, géographie du workspace, modèles pris en charge, champ `usage.inference_geo`.

2.  [Claude Platform Docs: Pricing](https://platform.claude.com/docs/en/about-claude/pricing): prix catalogue de l’API Anthropic et facteur 1,1 pour `inference_geo: "us"`.

3.  [Claude Platform Docs: API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention): Zero Data Retention, exceptions pour les contenus signalés et pour Fable/Mythos.

4.  [Anthropic: Privacy Policy](https://www.anthropic.com/legal/privacy): Anthropic Ireland comme responsable pour l’EEE, le Royaume-Uni et la Suisse, transfert vers les États-Unis sur la base de clauses contractuelles types.

5.  [Anthropic: Regional compliance](https://claude.com/regional-compliance): aperçu des plateformes offrant la résidence des données dans chaque région ; Foundry Europe comme « Coming soon ».

6.  [Claude Platform Docs: Claude in Amazon Bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock): types de points de terminaison par région, supplément de 10 % pour les points de terminaison régionaux.

7.  [AWS: carte de modèle Claude Sonnet 4.6](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-4-6.html): régions cibles du profil EU selon la région source, Zurich inclus.

8.  [AWS: carte de modèle Claude Sonnet 5.5](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5-5.html): disponibilité dans `eu-central-2` uniquement comme profil Geo et Global.

9.  [AWS: Geographic cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/geographic-cross-region-inference.html): conservation des données dans la zone géographique, lieu de stockage, blocage des régions cibles par SCP.

10.  [AWS: Cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html): champ `additionalEventData.inferenceRegion` dans CloudTrail, prix selon la région source.

11.  [AWS: Data protection in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html): aucun accès des fournisseurs de modèles aux prompts, réponses et logs.

12.  [AWS: Amazon Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/): prix catalogue pour les profils Global et Geo, identiques pour Zurich, Francfort et North Virginia.

13.  [AWS Alps Blog: Cross-region inference for EU data processing in Switzerland](https://aws.amazon.com/blogs/alps/unlocking-ai-flexibility-in-switzerland-a-guide-to-cross-region-inference-for-eu-data-processing-and-model-access/): guide d’AWS Suisse sur le profil EU depuis Zurich.

14.  [Claude Code Docs: Amazon Bedrock](https://code.claude.com/docs/en/amazon-bedrock): variables d’environnement et préfixe automatique `eu.` dans les régions UE.

15.  [Google Cloud: Generative AI locations](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/locations): tableau des emplacements des modèles Claude, aucune disponibilité dans `europe-west6`.

16.  [Google Cloud: Data residency](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/data-residency): multi-région UE uniquement pour les États membres de l’UE, Suisse et Royaume-Uni exclus.

17.  [Google Cloud: Vertex AI Pricing](https://cloud.google.com/vertex-ai/generative-ai/pricing): prix catalogue global, multi-région UE et `europe-west1`.

18.  [Microsoft Learn: Claude models hosting comparison](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/claude-models-hosting-comparison): Hosted on Azure et Hosted on Anthropic, Data Zone uniquement aux États-Unis.

19.  [Microsoft Learn: Anthropic as AI subprocessor in Microsoft 365 Copilot](https://learn.microsoft.com/en-us/microsoft-365/copilot/connect-to-ai-subprocessor): exception à l’EU Data Boundary, réglage par défaut dans l’UE/AELE.

20.  [TCDEV Blog: Where Are Claude's Data Centers?](https://www.tcdev.de/blog/where-are-claudes-data-centers/): aperçu de Tim Cadenbach sur Project Rainier, l’accord Google TPU et les centres de données propres avec Fluidstack.

21.  [Anthropic: investissement de 50 milliards USD dans l’infrastructure américaine](https://www.anthropic.com/news/anthropic-invests-50-billion-in-american-ai-infrastructure): annonce des centres de données propres au Texas et à New York avec Fluidstack.

22.  [Claude Platform Docs: IP addresses](https://platform.claude.com/docs/en/api/ip-addresses): plage d’adresses entrantes `160.79.104.0/23`.

23.  [RIPEstat](https://stat.ripe.net/): aperçu du préfixe et statut BGP pour `160.79.104.0/23` (AS399358, upstream AS13335).

24.  [llmlatency.dev: Anthropic](https://llmlatency.dev/provider/anthropic): temps de réponse à l’edge selon le site de mesure.

25.  [Fedlex: ordonnance sur la protection des données (OPDo), annexe 1](https://www.fedlex.admin.ch/eli/cc/2022/568/de): liste des États disposant d’une protection des données adéquate, y compris UE/EEE et États-Unis pour les entreprises certifiées.
