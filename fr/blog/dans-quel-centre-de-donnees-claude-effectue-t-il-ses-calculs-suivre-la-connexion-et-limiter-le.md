---
title: "Dans quel centre de données Claude effectue-t-il ses calculs ? Suivre la connexion et limiter le traitement à la Suisse ou à l’UE"
navTitle: "Centre de données Claude"
description: "Les étapes qu’une requête à Claude traverse, pourquoi la trace s’arrête à l’edge Cloudflare et ce qu’Anthropic révèle sur ses centres de données. Comparaison des voies avec un lieu de traitement fixe (API Anthropic, AWS Bedrock, Google Vertex AI, Microsoft Foundry), limites d’une exploitation exclusivement suisse et coût de l’ancrage régional."
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
translationSourceHash: 82442938b810b052119b19044fb328ab5edcdebb9350352ff47207901949421b
translationModel: gpt-5.6-terra
translatedAt: 2026-10-20T08:44:46.332Z
translationReview: required
url: https://rafaelpfister.ch/fr/blog/dans-quel-centre-de-donnees-claude-effectue-t-il-ses-calculs-suivre-la-connexion-et-limiter-le
aktualisiert: 2026-10-25
---

Toute personne qui utilise Claude via claude.ai, Claude Code ou l’API ne sait pas dans quel centre de données le modèle traite sa requête. Seule la première partie de la connexion, jusqu’au nœud edge le plus proche, est visible. Pour les entreprises qui transmettent des données personnelles ou des contenus confidentiels à un modèle de langage, cela ne suffit pas : elles doivent pouvoir démontrer à leurs clients, conseillers en protection des données ou à l’audit où le traitement a lieu.

Cet article montre jusqu’où la connexion peut être retracée, ce qui est publiquement connu des centres de données d’Anthropic et par quelles voies le traitement peut être limité de manière contraignante à l’UE ou à l’espace UE plus Suisse. Les prix et les régions correspondent à la situation au 2 octobre 2026.

**Focus Suisse :** La loi fédérale révisée sur la protection des données (revLPD) est déterminante. Une communication de données personnelles à l’étranger est autorisée selon l’art. 16 revLPD lorsque le Conseil fédéral atteste un niveau de protection adéquat dans l’État destinataire (annexe 1 de l’ordonnance sur la protection des données, OPDo) ou lorsque des garanties appropriées, telles que les clauses contractuelles types, existent. Les États de l’UE et de l’EEE figurent sur cette liste, les États-Unis uniquement pour les entreprises certifiées au titre du Swiss-U.S. Data Privacy Framework. Un lieu de traitement dans l’UE simplifie donc considérablement la justification, sans remplacer les autres obligations (contrat de sous-traitance, information des personnes concernées, sécurité des données).

**Note UE :** Pour les entreprises établies dans l’UE, les art. 44 ss. RGPD s’appliquent à la place. La Commission européenne a reconnu à la Suisse un niveau de protection des données adéquat (décision d’adéquation 2000/518/CE, confirmée en janvier 2024) ; un traitement à Zurich ne constitue donc pas un transfert vers un pays tiers problématique pour les entreprises de l’UE.

<aside class="pilot-card">
  <span class="pilot-card__tag">Nouveau par adeptio · Entreprises pilotes recherchées</span>
  <p class="pilot-card__titel">Utiliser Claude et OpenAI sans que les données personnelles quittent la Suisse</p>
  <p class="pilot-card__lead">Notre couche intermédiaire se place entre vos applications en Suisse et les modèles d’Anthropic et d’OpenAI, et anonymise chaque requête avant qu’elle quitte le pays. Pour l’exploitation en conditions réelles, nous recherchons des entreprises qui utilisent le produit au quotidien.</p>
  <div class="pilot-card__spalten">
    <div class="pilot-card__spalte">
      <p class="pilot-card__spalte-titel">Votre entreprise</p>
      <ul>
        <li>Les données personnelles ne quittent la Suisse qu’après anonymisation</li>
        <li>Justification simplifiée vis-à-vis de la protection des données et de l’audit</li>
        <li>Accès avant la disponibilité générale et participation aux choix fonctionnels</li>
        <li>Contact direct avec les développeurs</li>
      </ul>
    </div>
    <div class="pilot-card__spalte">
      <p class="pilot-card__spalte-titel">adeptio</p>
      <ul>
        <li>Enseignements tirés de vrais prompts plutôt que de données de test</li>
        <li>Mesures de latence et de qualité des réponses sous charge réelle</li>
        <li>Références sur le marché suisse</li>
      </ul>
    </div>
  </div>
  <div class="pilot-card__aktionen">
    <a class="pilot-card__cta" href="/kontakt">S’inscrire comme entreprise pilote</a>
    <a class="pilot-card__mehr" href="#anonymisierung-vor-dem-modell-pilotunternehmen-gesucht">Détails et qualification juridique</a>
  </div>
</aside>

## Les étapes qu’une requête à Claude traverse

Une requête vers `claude.ai` ou `api.anthropic.com` passe par trois étapes :

| Étape | Exploitant | Visible pour vous ? |
|---|---|---|
| Nœud edge (terminaison TLS, protection contre les abus) | Cloudflare, au nom d’Anthropic | Oui, emplacement lisible dans le header |
| Réseau interne jusqu’au backend | Cloudflare et Anthropic | Non |
| Inférence (le modèle calcule) | Anthropic, sur des ressources de calcul d’AWS, Google Cloud et de ses propres centres de données | Non, uniquement l’indication générale `us` ou `global` |

Les noms d’hôte `claude.ai` et `api.anthropic.com` pointent tous deux vers l’adresse `160.79.104.10`. Elle appartient au préfixe `160.79.104.0/23`, enregistré au nom d’Anthropic (AS399358) et dont l’origine Anthropic est valide selon RPKI. Dans le routage mondial, le préfixe est toutefois annoncé exclusivement via Cloudflare (AS13335) : Anthropic injecte ses propres adresses IP dans le réseau Anycast de Cloudflare. La même adresse répond ainsi depuis chaque site Cloudflare dans le monde, et la connexion aboutit au nœud edge le plus proche.

## Déterminer le nœud edge lui-même

Cloudflare révèle le site qui assure le service dans une page de diagnostic et dans le header `cf-ray`. L’abréviation après l’identifiant Ray est un code aéroportuaire IATA : `ZRH` correspond à Zurich, `GVA` à Genève, `FRA` à Francfort, `MRS` à Marseille.

```bash
curl -s https://api.anthropic.com/cdn-cgi/trace
```

Dans la sortie, les lignes `colo=` (site edge) et `loc=` (pays auquel Cloudflare attribue votre adresse source) sont pertinentes. Sous Windows, PowerShell fournit le même résultat :

```powershell
$trace = Invoke-WebRequest -Uri "https://api.anthropic.com/cdn-cgi/trace" -UseBasicParsing
$trace.Content
```

Vous lisez le header avec une requête HEAD :

```bash
curl -sI https://api.anthropic.com/ \
  | grep -i -E '^(server|cf-ray):'
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-s` | Supprime l’indicateur de progression et les messages d’erreur de curl |
| `-I` | Envoie une requête HEAD et affiche uniquement les headers de réponse |
| `grep -i` | Recherche sans tenir compte des majuscules et minuscules |
| `-E '^(server\|cf-ray):'` | Expression régulière étendue : uniquement les lignes commençant par `server:` ou `cf-ray:` |

</details>

Depuis un réseau suisse, il faut généralement s’attendre à `ZRH` ou `GVA`. Si la sortie Internet se trouve ailleurs, par exemple dans un réseau d’entreprise dont la sortie centrale est à l’étranger, le nœud edge répond là-bas, par exemple avec `colo=FRA` ou `colo=MRS`. Le site edge suit donc votre sortie réseau, et non le lieu de traitement.

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
| `-TraceRoute` | Détermine les sauts de routeur sur le chemin vers la cible |
| `-Hops 20` | Nombre maximal de sauts vérifiés |

</details>

Sous Linux et macOS, cela correspond à `traceroute -n api.anthropic.com` (`-n` supprime la résolution DNS des sauts). Lors de la mesure, les derniers sauts avant `160.79.104.10` se trouvaient dans la plage d’adresses `162.158.0.0/15`, qui appartient à Cloudflare. La trace s’arrête ensuite : la connexion est terminée au nœud edge, le transfert vers le backend s’effectue dans une nouvelle connexion interne, invisible de l’extérieur.

RIPEstat indique à qui appartient le préfixe et par quel réseau il est annoncé :

```bash
curl -s "https://stat.ripe.net/data/prefix-overview/data.json?resource=160.79.104.10"
curl -s "https://stat.ripe.net/data/bgp-state/data.json?resource=160.79.104.0/23"
```

Dans le deuxième résultat, tous les chemins AS se terminent par `13335 399358` : Cloudflare est le seul upstream du préfixe Anthropic.

Selon mes recherches, il n’existe pas de mesure publique permettant de retracer le chemin jusqu’à un centre de données d’inférence et, en raison de la terminaison à l’edge, cela n’est pas non plus possible avec des outils réseau. Les services de mesure tels que llmlatency.dev ne mesurent que le temps de réponse jusqu’au premier octet à l’edge (état de septembre 2026 : médiane de 96 ms depuis US-Central, 199 ms depuis l’Allemagne). Le délai jusqu’au premier token (Time to First Token) pourrait certes être comparé depuis plusieurs emplacements, mais il dépend davantage du modèle, de la longueur du prompt et de la charge que de la distance. Il n’est donc pas possible d’en déduire de façon fiable un emplacement.

## Ce que l’on sait des centres de données d’Anthropic

Anthropic n’exploite à ce jour aucun centre de données d’inférence propre publiquement associé à une adresse. Les ressources de calcul proviennent de partenariats recensés par Tim Cadenbach dans le blog TCDEV :

| Site / partenaire | Données clés | Rôle connu |
|---|---|---|
| Project Rainier, New Carlisle (Indiana, États-Unis), Amazon Web Services | Ouvert en octobre 2025, environ 11 milliards USD, quelque 500 000 puces Trainium2, plus de 2,2 GW prévus | Anthropic est le locataire principal ; surtout entraînement |
| Google Cloud | Accord d’octobre 2025 portant sur jusqu’à 1 million de TPU, plus de 1 GW à partir de 2026 | Régions non publiées |
| Centres de données propres avec Fluidstack | Annoncés en novembre 2025, 50 milliards USD, sites au Texas et à New York, mise en service à partir de 2026 | Encore en construction |

L’entraînement et l’inférence ne se déroulent pas nécessairement au même endroit. L’inférence peut être exécutée dans n’importe quelle région AWS ou Google Cloud, et Anthropic ne révèle pas quel site calcule la réponse pour un utilisateur en Suisse. Seul ce que la politique de confidentialité établit est certain : le cocontractant et responsable du traitement pour les clients de l’EEE, du Royaume-Uni et de la Suisse est Anthropic Ireland, Limited à Dublin ; les données sont transférées vers des serveurs aux États-Unis ou dans d’autres pays hors EEE, sur la base de clauses contractuelles types.

## Le contrôle directement chez Anthropic : États-Unis ou mondial uniquement

Depuis les modèles de génération 4.6, l’API Claude connaît le paramètre `inference_geo`. Il possède exactement deux valeurs :

| Valeur | Effet | Prix |
|---|---|---|
| `global` (standard) | Inférence dans toute région disponible | Prix catalogue |
| `us` | Inférence exclusivement aux États-Unis | Prix catalogue × 1,1 |

Il n’existe aucune valeur pour l’UE ou la Suisse. Le lieu de stockage du workspace (`workspace geo`) ne peut actuellement être défini que sur `us`. La réponse indique dans le champ `usage.inference_geo` le paramètre appliqué, mais pas de région précise. Pour Haiku 4.5 et les modèles plus anciens, l’API répond au paramètre par une erreur 400.

Pour claude.ai et Claude Code avec un abonnement (Pro, Max, Team, Enterprise), il n’existe aucun réglage du lieu de traitement. Les offres directes d’Anthropic ne permettent donc de limiter le traitement ni à l’UE ni à la Suisse. Cela n’est possible qu’auprès des fournisseurs de cloud qui exploitent Claude dans leurs propres régions.

## AWS Bedrock : espace UE plus Suisse depuis Zurich

AWS Bedrock exploite Claude sur l’infrastructure AWS. Selon AWS, Anthropic n’a pas accès aux prompts, réponses ou logs des clients. Vous choisissez la région via la région source de votre appel API et le profil d’inférence :

| Type d’endpoint | Préfixe de l’ID de modèle | Lieu de traitement |
|---|---|---|
| Global Cross-Region Inference | `global.` | Toute région AWS commerciale dans le monde |
| Geographic Cross-Region Inference | `eu.` | Uniquement les régions de la zone géographique |
| In-Region | sans préfixe | Uniquement la région appelée |

Pour la Suisse, la région `eu-central-2` (Zurich) est déterminante. Opus 5.5, Sonnet 5.5 et Haiku 4.5 y sont disponibles, toutefois uniquement avec un profil Global ou UE, et non In-Region. Il n’existe pas de profil calculant exclusivement à Zurich. Si le profil UE est appelé depuis Zurich, AWS répartit les requêtes, selon la fiche modèle (documenté pour Haiku 4.5 et Sonnet 4.6), entre les régions suivantes :

| Région | Site |
|---|---|
| `eu-central-2` | Zurich |
| `eu-central-1` | Francfort |
| `eu-north-1` | Stockholm |
| `eu-south-1` | Milan |
| `eu-south-2` | Espagne |
| `eu-west-1` | Irlande |
| `eu-west-3` | Paris |

Zurich n’est une destination possible que si la requête provient de Zurich. Le traitement reste ainsi dans l’espace UE plus Suisse. Pour Opus 5.5 et Sonnet 5.5, la fiche modèle ne mentionne pas les régions cibles ; l’API renvoie la liste réelle :

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
| `bedrock get-inference-profile` | Lit la définition d’un profil d’inférence |
| `--region eu-central-2` | Région source Zurich ; la liste de destinations dépend de la région source |
| `--inference-profile-identifier` | ID du profil UE pour Sonnet 5.5 |
| `--query "models[].modelArn"` | Affiche uniquement les ARN de modèles ; la région figure dans chaque ARN |

</details>

Par défaut, les données ne sont stockées que dans la région source. Une exception concerne les contenus conservés pour la détection des abus : ils se trouvent dans la région cible. Le transport entre régions est chiffré sur le réseau AWS.

### Prouver la région de traitement pour chaque requête

Bedrock est la seule des voies décrites à enregistrer la région de traitement effective pour chaque requête. CloudTrail l’écrit dans la région source, dans le champ `additionalEventData.inferenceRegion` :

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
| `cloudtrail lookup-events` | Parcourt les événements CloudTrail des 90 derniers jours |
| `--region eu-central-2` | Région source dans laquelle les appels sont consignés |
| `--lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock.amazonaws.com` | Filtre les événements Bedrock |
| `--max-results 20` | Limite la sortie à 20 événements |
| `--query "Events[].CloudTrailEvent"` | Affiche uniquement l’événement complet sous forme de texte JSON |
| `--output text` | Sortie sans enveloppe JSON, un événement par ligne |
| `jq -r '… \| @tsv'` | Extrait l’horodatage, l’action et la région de traitement sous forme de ligne séparée par des tabulations |

</details>

En outre, une Service Control Policy (SCP) dans AWS Organizations permet de bloquer toutes les régions hors UE et Suisse. Si une région cible d’un profil est bloquée, la requête échoue au lieu de basculer vers une autre région. Avec cette combinaison, le lieu de traitement est imposé techniquement et démontrable pour chaque requête.

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
| `AWS_REGION=eu-central-2` | Région source Zurich ; il en résulte le profil UE |

Le préfixe peut être remplacé avec `ANTHROPIC_BEDROCK_REGION_PREFIX`. L’authentification AWS s’effectue via les mécanismes habituels (profil, SSO, variables d’environnement). La facturation passe par le compte AWS, et non par un abonnement Claude.

## Google Vertex AI : uniquement l’UE, explicitement sans la Suisse

Sur Google Cloud (Vertex AI, désormais sous le nom Gemini Enterprise Agent Platform), la situation est moins favorable pour la Suisse :

| Endpoint | Modèles (sélection) | Lieu de traitement |
|---|---|---|
| `global` | tous | Toute région Google Cloud, sans garantie |
| Multi-région `eu` (`aiplatform.eu.rep.googleapis.com`) | Opus 5.5, Sonnet 5.5, Opus 5, Sonnet 5, Opus 4.8, Opus 4.7 | Uniquement les États membres de l’UE |
| Région `europe-west1` (Belgique) | Haiku 4.5, Sonnet 4.6, Opus 4.6 et versions antérieures | Selon la page modèle, multi-région Europe |
| Région `europe-west6` (Zurich) | aucun | Claude non disponible |

Google exclut explicitement la Suisse de l’endpoint multi-région UE : il ne couvre que les États membres de l’UE ; le Royaume-Uni et la Suisse n’en font pas partie. Plusieurs guides suisses mentionnent `europe-west6` comme moyen d’accéder à Claude à Zurich ; selon le tableau officiel des emplacements de Google, aucun modèle Claude n’y est disponible. Google ne documente pas de champ de journal indiquant la région de traitement effective par requête. Pour l’endpoint global, Google précise que la région de traitement ne peut être ni contrôlée ni déterminée.

Vertex AI convient donc à un traitement exclusivement dans l’UE, mais pas à un traitement en Suisse.

## Microsoft Foundry : aucune option UE à ce jour

Microsoft propose Claude dans Foundry en deux variantes. « Hosted on Azure » calcule sur une infrastructure Azure, en Global Standard ou Data Zone Standard ; la Data Zone n’existe pour Claude qu’aux États-Unis. « Hosted on Anthropic » calcule sur l’infrastructure Anthropic, et Microsoft indique que les données peuvent être traitées hors d’Azure et hors de la région sélectionnée. Anthropic indique Foundry pour l’Europe comme « Coming soon ». Dans Microsoft 365 Copilot et Copilot Studio, les modèles Anthropic sont exclus de l’EU Data Boundary et désactivés par défaut dans l’UE, l’AELE (donc aussi en Suisse) et au Royaume-Uni.

## Vue d’ensemble : quelle voie garantit quel lieu de traitement

| Voie | Suisse uniquement | UE uniquement | Espace UE plus Suisse | Région démontrable par requête |
|---|---|---|---|---|
| claude.ai, Claude Code (abonnement) | non | non | non | non |
| API Claude avec `inference_geo` | non | non | non (uniquement `us`) | uniquement `us`/`global` |
| AWS Bedrock, source `eu-central-2`, profil `eu.` | non | non | oui | oui (CloudTrail) |
| AWS Bedrock, source dans l’UE, profil `eu.` | non | oui | oui | oui (CloudTrail) |
| Google Vertex AI, endpoint `eu` | non | oui | (partie UE uniquement) | non |
| Microsoft Foundry | non | non | non | non |

Aucun fournisseur ne propose actuellement une voie garantissant que le traitement de Claude soit limité à la Suisse. AWS Bedrock avec la région source Zurich et le profil UE s’en approche le plus. S’il faut en outre exclure que les données quittent l’UE, par exemple en raison d’engagements contractuels envers des clients de l’UE, le profil UE peut être appelé depuis une région UE telle que Francfort ; Zurich n’est alors plus une destination.

## Anonymisation avant le modèle : entreprises pilotes recherchées

Dans la plupart des cas, le lieu de traitement n’est pertinent que parce que le prompt contient des données personnelles ou des informations confidentielles. C’est à ce stade qu’intervient un nouveau produit de mon entreprise, adeptio gmbh : il se place comme couche intermédiaire entre les applications en Suisse et les modèles d’Anthropic et d’OpenAI, et anonymise les requêtes avant qu’elles quittent la Suisse. Le modèle ne reçoit plus que des contenus sans lien avec une personne, quel que soit le centre de données où il calcule.

**Focus Suisse :** Les informations qui ne peuvent plus être attribuées à une personne déterminée ou déterminable ne sont pas des données personnelles au sens de l’art. 5, let. a revLPD. Les règles relatives à la communication à l’étranger (art. 16 s. revLPD) ne s’appliquent pas à ces contenus. La question de savoir si une anonymisation atteint ce seuil dépend de sa mise en œuvre et des données concrètes ; c’est précisément ce que nous voulons démontrer en exploitation réelle.

Pour le lancement, nous recherchons des entreprises qui utilisent Claude ou des modèles d’OpenAI en production, ou qui prévoient de les introduire, et qui emploieront le produit au quotidien comme entreprise pilote. Les deux parties en bénéficient :

| Votre entreprise | adeptio |
|---|---|
| Les données personnelles et les informations confidentielles ne quittent la Suisse qu’après anonymisation. La justification auprès des conseillers en protection des données, des clients et de l’audit s’en trouve simplifiée. | Enseignements tirés de vrais prompts et flux de travail plutôt que de données de test : quelles informations doivent être détectées et où l’anonymisation retire trop ou trop peu d’éléments. |
| Le choix entre Anthropic, OpenAI, Bedrock ou Vertex AI dépend à nouveau de l’offre de modèles et du prix, et non plus uniquement du lieu de traitement. | Mesures de latence et de qualité des réponses sous charge réelle. |
| Accès au produit avant sa disponibilité générale et influence sur les types de données et cas d’usage qu’il couvre en premier. | Retours de la protection des données et de l’audit, qui orientent les preuves fournies par le produit. |
| Contact direct avec les développeurs plutôt qu’un support par tickets. | Références sur le marché suisse pour la disponibilité générale. |

<aside class="offer-box">
  <span class="offer-box__tag">Entreprises pilotes recherchées</span>
  <p><strong>Utilisez-vous Claude ou OpenAI avec des données personnelles ?</strong> Utilisez la couche d’anonymisation d’adeptio en exploitation réelle et participez à son évolution.</p>
  <a class="offer-box__cta" href="/kontakt">Prendre contact</a>
</aside>

## Ce que coûte l’ancrage régional

Les trois fournisseurs de cloud et Anthropic elle-même facturent un supplément de 10 % par rapport à l’endpoint global pour l’ancrage à une région ou une zone géographique. Chez Bedrock, les prix dépendent de la région source ; Zurich, Francfort et Northern Virginia coûtent la même chose. Le routage entre régions n’est pas facturé en supplément.

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

L’ancrage régional coûte donc 10 % de plus que l’endpoint global. Par rapport au prix catalogue chez Anthropic directement avec `inference_geo: "us"`, un profil UE sur Bedrock coûte le même prix. Les coûts annexes pèsent davantage dans le choix : un compte AWS ou Google Cloud avec politiques d’organisation, journalisation et alertes budgétaires doit être mis en place et exploité, et les prix mensuels fixes des abonnements Claude sont remplacés par une facturation purement à l’usage. La rentabilité du changement dépend du volume ; un abonnement a un prix fixe par personne, mais n’offre aucun contrôle sur le lieu de traitement.

## Conservation et entraînement

Outre le lieu, la durée de conservation des données compte :

- **API Anthropic :** Les prompts et réponses ne sont pas utilisés pour l’entraînement sans consentement explicite. La Zero Data Retention (ZDR) est disponible sur demande par organisation et peut être combinée avec `inference_geo`. Les contenus signalés par la détection des abus restent conservés jusqu’à deux ans, même avec ZDR. Pour les modèles Fable et Mythos, une conservation de 30 jours est obligatoire.
- **AWS Bedrock :** Les contenus ne sont pas transmis à Anthropic. La conservation est configurable par modèle entre `none`, `default` et `aws_review`; Fable requiert `aws_review` avec une conservation pouvant aller jusqu’à 30 jours pour vérification par AWS.
- **Google Vertex AI :** Google n’utilise les données pour l’entraînement ou le fine-tuning qu’avec consentement préalable.

## Sources

1.  [Claude Platform Docs: Data residency](https://platform.claude.com/docs/en/manage-claude/data-residency): paramètre `inference_geo`, valeurs `us` et `global`, géographie du workspace, modèles pris en charge, champ `usage.inference_geo`.

2.  [Claude Platform Docs: Pricing](https://platform.claude.com/docs/en/about-claude/pricing): prix catalogue de l’API Anthropic et facteur 1,1 pour `inference_geo: "us"`.

3.  [Claude Platform Docs: API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention): Zero Data Retention, exceptions pour les contenus signalés et pour Fable/Mythos.

4.  [Anthropic: Privacy Policy](https://www.anthropic.com/legal/privacy): Anthropic Ireland comme responsable pour l’EEE, le Royaume-Uni et la Suisse, transfert vers les États-Unis sur la base de clauses contractuelles types.

5.  [Anthropic: Regional compliance](https://claude.com/regional-compliance): aperçu des plateformes offrant la résidence des données dans chaque région ; Foundry Europe indiqué comme « Coming soon ».

6.  [Claude Platform Docs: Claude in Amazon Bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock): types d’endpoint par région, supplément de 10 % pour les endpoints régionaux.

7.  [AWS: Fiche modèle Claude Sonnet 4.6](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-4-6.html): régions cibles du profil UE selon chaque région source, y compris Zurich.

8.  [AWS: Fiche modèle Claude Sonnet 5.5](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5-5.html): disponibilité dans `eu-central-2` uniquement en profils Geo et Global.

9.  [AWS: Geographic cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/geographic-cross-region-inference.html): maintien des données dans la zone géographique, lieu de stockage, blocage des régions cibles par SCP.

10.  [AWS: Cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html): champ `additionalEventData.inferenceRegion` dans CloudTrail, prix selon la région source.

11.  [AWS: Data protection in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html): aucun accès des fournisseurs de modèles aux prompts, réponses et logs.

12.  [AWS: Amazon Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/): prix catalogue pour profils Global et Geo, identiques pour Zurich, Francfort et Northern Virginia.

13.  [AWS Alps Blog: Cross-region inference for EU data processing in Switzerland](https://aws.amazon.com/blogs/alps/unlocking-ai-flexibility-in-switzerland-a-guide-to-cross-region-inference-for-eu-data-processing-and-model-access/): guide d’AWS Suisse relatif au profil UE depuis Zurich.

14.  [Claude Code Docs: Amazon Bedrock](https://code.claude.com/docs/en/amazon-bedrock): variables d’environnement et préfixe automatique `eu.` dans les régions UE.

15.  [Google Cloud: Generative AI locations](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/locations): tableau des emplacements des modèles Claude, aucune disponibilité dans `europe-west6`.

16.  [Google Cloud: Data residency](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/data-residency): multi-région UE réservée aux États membres de l’UE, Suisse et Royaume-Uni exclus.

17.  [Google Cloud: Vertex AI Pricing](https://cloud.google.com/vertex-ai/generative-ai/pricing): prix catalogue global, multi-région UE et `europe-west1`.

18.  [Microsoft Learn: Claude models hosting comparison](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/claude-models-hosting-comparison): Hosted on Azure et Hosted on Anthropic, Data Zone uniquement aux États-Unis.

19.  [Microsoft Learn: Anthropic as AI subprocessor in Microsoft 365 Copilot](https://learn.microsoft.com/en-us/microsoft-365/copilot/connect-to-ai-subprocessor): exception à l’EU Data Boundary, configuration par défaut dans l’UE/AELE.

20.  [TCDEV Blog: Where Are Claude's Data Centers?](https://www.tcdev.de/blog/where-are-claudes-data-centers/): récapitulatif de Tim Cadenbach sur Project Rainier, l’accord Google TPU et les propres centres de données avec Fluidstack.

21.  [Anthropic: Investissement de 50 milliards USD dans l’infrastructure américaine](https://www.anthropic.com/news/anthropic-invests-50-billion-in-american-ai-infrastructure): annonce de centres de données propres au Texas et à New York avec Fluidstack.

22.  [Claude Platform Docs: IP addresses](https://platform.claude.com/docs/en/api/ip-addresses): plage d’adresses entrante `160.79.104.0/23`.

23.  [RIPEstat](https://stat.ripe.net/): aperçu du préfixe et statut BGP pour `160.79.104.0/23` (AS399358, upstream AS13335).

24.  [llmlatency.dev: Anthropic](https://llmlatency.dev/provider/anthropic): temps de réponse à l’edge selon le site de mesure.

25.  [Fedlex: Ordonnance sur la protection des données (OPDo), annexe 1](https://www.fedlex.admin.ch/eli/cc/2022/568/de): liste des États offrant une protection des données adéquate, y compris UE/EEE et États-Unis pour les entreprises certifiées.
