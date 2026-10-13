---
title: "Claude et Claude Code : architecture, API et exploitation sécurisée des agents"
blatt: "claude"
description: "Claude et Claude Code pour les administrateurs de plateformes, de sécurité et d’automatisation : limites des modèles et de l’API, Messages sans état, Content Blocks, streaming, Tool Use et MCP, runtime local des agents, workspaces et clés, limites de débit, caching, batches, observabilité, protection des données, prompt injection et reprise."
fakten:
  - label: Rôle du système
    wert: Claude est une famille de modèles de langage génératifs ; les applications fournissent le contexte et traitent des Content Blocks générés de manière probabiliste
    href: https://platform.claude.com/docs/en/about-claude/models/overview
  - label: Fournisseur
    wert: Anthropic développe les modèles, l’API Claude directe, les applications Claude et Claude Code
    href: https://www.anthropic.com/news/introducing-claude
  - label: API principale
    wert: POST /v1/messages traite des messages structurés ; l’état de la conversation est renvoyé par le client à chaque requête
    href: https://platform.claude.com/docs/en/api/messages/create
  - label: Transport
    wert: HTTPS/JSON ; le streaming utilise Server-Sent Events sans modifier le contrat sémantique des Messages
    href: https://platform.claude.com/docs/en/build-with-claude/streaming
  - label: Sortie
    wert: un message contient des Content Blocks typés, des mesures d’utilisation et stop_reason, plutôt qu’un texte libre garanti
    href: https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons
  - label: Association au modèle
    wert: Les IDs de modèle sont des contrats épinglés ; les capacités et les limites sont déterminées via l’API Models et la documentation
    href: https://platform.claude.com/docs/en/about-claude/models/model-ids-and-versions
  - label: Contrat des outils
    wert: Claude génère tool_use avec des arguments JSON ; le code client exécute et renvoie tool_result
    href: https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works
  - label: MCP
    wert: protocole client-serveur ouvert pour les outils, les ressources et le contexte ; chaque serveur étend les droits sur les données et les actions
    href: https://docs.anthropic.com/en/docs/mcp
  - label: Claude Code
    wert: agent local pour les dépôts, fichiers, shell et outils ; l’inférence API reste un service externe
    href: https://docs.anthropic.com/en/docs/claude-code/getting-started
  - label: Modèle multi-tenant
    wert: Organisation → Workspace → membres, clés, limites et ressources liées au workspace
    href: https://platform.claude.com/docs/en/manage-claude/workspaces
  - label: Capacité
    wert: Spend Limits ainsi que limites de requêtes, de tokens d’entrée et de sortie ; 429 et retry-after pilotent le backoff
    href: https://platform.claude.com/docs/en/api/rate-limits
  - label: Conservation des données
    wert: La conservation dépend du produit, du workspace, de la fonctionnalité, du contrat et de la classification de sécurité, et est vérifiée avant utilisation
    href: https://privacy.claude.com/en/articles/7996866-how-long-do-you-store-my-organization-s-data
werbung:
  - newsletter
ctaThemen:
  - claude
translationSourceHash: a5189670b409d16f452c042199e3237347d4a9a1483be7c38ebbc07a8d640c5f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-10T10:08:42.154Z
translationReview: automatic
---

# Claude et Claude Code : architecture, API et exploitation sécurisée des agents

Claude désigne une famille de grands modèles de langage génératifs d’Anthropic, ainsi que plusieurs produits qui utilisent ces modèles. La distinction administrative essentielle est la suivante : **le modèle, l’API, l’application et le runtime de l’agent ne constituent pas le même système**. Le modèle génère, à partir de tokens d’entrée, une suite probabiliste de tokens de sortie. L’API Messages encapsule ce processus sous la forme d’un contrat HTTPS. Les applications Claude ajoutent des comptes, une mémoire de conversation et des interfaces utilisateur. Claude Code ajoute, sur un endpoint, un runtime local avec accès aux fichiers, au shell et aux outils.

La première version de Claude a été présentée en 2023 comme service de chat et d’API ([Présentation de Claude](https://www.anthropic.com/news/introducing-claude)). Anthropic décrit Claude comme un assistant conçu pour adopter un comportement utile, honnête et inoffensif. Cet alignement, une grande fenêtre de contexte ou un texte formulé de manière convaincante ne constituent toutefois pas des preuves d’exactitude factuelle, d’autorisation ou d’exécution sûre. Un système de production doit traiter les réponses du modèle comme des entrées non fiables, contrôlées par des schémas et des politiques.

L’explication commence par une requête au modèle et suit la réponse via l’API Messages, les Content Blocks et les appels d’outils. Ce n’est qu’une fois ce déroulement clair que Claude Code, MCP, les autorisations, la prompt injection, l’exploitation et la reprise sont approfondis.

Claude n’est pas un système d’exploitation qui agit de façon autonome, mais un service d’inférence. L’application environnante fournit le contexte, vérifie les réponses, exécute les outils approuvés et assume la responsabilité des identités, de l’accès aux données et de l’audit.

## Approche d’architecture : service d’inférence et application de contrôle

Claude fonctionne normalement comme un service d’inférence externe. Un client envoie des instructions système, des messages de conversation, des Content Blocks, des définitions d’outils et des paramètres de génération. Le service authentifie et limite la requête, tokenise le contexte, effectue l’inférence et renvoie un message ou un flux d’événements. L’application décide ensuite d’afficher le texte, de valider le JSON, d’exécuter un outil, de renvoyer un résultat ou d’interrompre l’exécution.

L’API directe n’est pas le seul chemin de déploiement. Des modèles Claude sont également proposés via des plateformes cloud ; l’identité, l’endpoint, les régions, les quotas, la journalisation et les conditions contractuelles y diffèrent. L’architecture applicative sépare les adaptateurs de fournisseurs et le workflow métier. Un changement de modèle n’est pas une simple commutation DNS lorsque les types d’outils, les limites de contexte, les Stop Reasons ou les fonctions du fournisseur diffèrent.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1116" src="/images/kb-interaktiv-claude.svg?v=20260813" title="Interaktive Infografik: Claude von Benutzer, Anwendung und Workspace über Messages API, Tokenisierung und Modellinferenz bis zu Content Blocks, Tool Use, MCP, Claude Code, Rate Limits, Logging, Datenschutz und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-claude.svg?v=20260813">Ouvrir directement le graphique interactif</a>.
</iframe>

## Couche de modèle et sémantique probabiliste

Les modèles Claude traitent du texte, du code et, selon le modèle, d’autres modalités. La [vue d’ensemble des modèles](https://platform.claude.com/docs/en/about-claude/models/overview) documente les familles de modèles, les fenêtres de contexte et les capacités. Ces valeurs ne doivent pas figurer comme faits durablement figés dans un article statique. Avant le déploiement, un client interroge les capacités documentées via l’API Models ou au moment de la compilation et enregistre l’identifiant de modèle réellement employé avec chaque résultat.

Un modèle de langage n’est ni une base de données ni un moteur de règles. Des entrées identiques peuvent générer des sorties différentes selon l’échantillonnage, le contexte système, la révision du modèle et les résultats d’outils. Même à basse température, les propriétés suivantes demeurent :

- Les faits peuvent être absents, obsolètes ou inventés.
- Les instructions peuvent être pondérées de manière ambiguë.
- Les longs contextes peuvent masquer des détails pertinents.
- Un texte ressemblant à une structure n’est pas un objet valide sans validation de schéma.
- Une explication plausible ne prouve pas qu’un outil a effectivement été exécuté.
- Un modèle ne connaît aucune autorisation en dehors des informations et des limites d’outils appliquées par l’application.

La réception technique utilise donc des cas d’évaluation, des schémas attendus, des contrôles déterministes et des sources métier. La réussite d’un benchmark de modèle ne remplace pas une mesure spécifique à l’application de l’exactitude, de la latence, des coûts et du préjudice en cas d’erreurs.

## Entraînement, Constitution et responsabilités

Anthropic a développé **Constitutional AI** comme complément à l’apprentissage supervisé et à l’apprentissage par renforcement. Le modèle critique et révise ses réponses au regard de principes explicites ; une seconde étape d’entraînement utilise du feedback généré par l’IA. Anthropic explique cette approche et ses limites dans [la Constitution de Claude](https://www.anthropic.com/research/claudes-constitution). La [fiche de modèle Claude 3](https://assets.anthropic.com/m/61e7d27f8c8f5919/original/Claude-3-Model-Card.pdf) documente, pour une génération concrète de modèles, l’entraînement, l’évaluation et les mesures de sécurité.

Pour les administrateurs, cette histoire est pertinente parce qu’elle explique le comportement, et non parce qu’elle remplace un contrôle d’exécution. L’entraînement à la sécurité peut réduire les réponses nuisibles, mais ne peut imposer ni la séparation des tenants ni l’autorisation des outils. Les refus constituent des sorties possibles normales du modèle. L’application doit les traiter comme `stop_reason` ou comme type de contenu et ne doit pas déduire de l’absence de refus qu’une action est sûre.

Le modèle probabiliste ne devient un service administrable qu’au travers de l’API Messages. Chaque requête transmet à nouveau le contexte de conversation requis et reçoit des Content Blocks structurés en réponse.

## API Messages : contrat de conversation sans état

`POST /v1/messages` reçoit une liste de messages `user` et `assistant` et génère le prochain tour de l’assistant. Un prompt système se trouve dans le champ de niveau supérieur distinct `system`; un rôle `system` dans `messages` n’existe pas. Le contenu peut être envoyé comme chaîne ou comme liste de blocs typés ([Créer un message](https://platform.claude.com/docs/en/api/messages/create)).

L’API est **sans état** en l’absence de mémoire de thread supplémentaire. Pour un dialogue à plusieurs tours, le client renvoie l’historique requis. Cela a des conséquences directes :

1. L’application est propriétaire de l’ID de conversation, de l’ordre et de la conservation.
2. Tronquer, résumer ou omettre modifie le contexte du modèle.
3. Le prompt système, les définitions d’outils et l’historique comptent dans le budget d’entrée.
4. Un ID de requête du fournisseur ne remplace pas un ID métier de tâche.
5. Une nouvelle tentative de la même requête peut produire une nouvelle sortie et des coûts supplémentaires.

L’opération métier reçoit donc son propre ID de corrélation et d’idempotence. Avant une nouvelle tentative, l’orchestrateur vérifie si un résultat ou un effet irréversible d’outil existe déjà.

## Content Blocks et Stop Reasons

Une réponse réussie n’est pas nécessairement un texte unique. `content` est une liste ordonnée de blocs typés, tels que du texte, Tool Use, Thinking ou des blocs d’outils serveur. `usage` indique les classes de tokens. `stop_reason` décrit pourquoi la génération s’est terminée. La [documentation Stop Reason](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons) distingue notamment `end_turn`, `max_tokens`, `stop_sequence`, `tool_use`, `pause_turn`, `refusal` et `model_context_window_exceeded`.

Seul `end_turn` indique un tour achevé naturellement ; il ne prouve pas l’exhaustivité métier. `max_tokens` et `model_context_window_exceeded` signalent des données potentiellement tronquées. `tool_use` est une demande adressée à l’orchestrateur, et non un effet exécuté. `pause_turn` exige une continuation conforme au protocole. De nouvelles valeurs d’énumération peuvent être ajoutées ; les parseurs doivent donc rejeter visiblement les types inconnus ou les ignorer de façon sûre, plutôt que de les faire tomber dans une réussite par défaut.

## Version d’API et version de modèle

Chaque requête API directe comporte un en-tête `anthropic-version`. Cette version d’API stabilise les champs et la sémantique de streaming, mais ne doit pas empêcher de nouvelles entrées facultatives, de nouveaux champs de sortie ou de nouvelles variantes d’énumération ([Versionnage d’API](https://platform.claude.com/docs/en/api/versioning)). Le parseur client est donc conçu pour être compatible avec les évolutions futures et journalise les blocs inconnus.

Elle est distincte de l’**ID de modèle**. Anthropic garantit, pour un ID épinglé, une version de modèle constante durant sa période de vie ; les alias pratiques peuvent obéir à d’autres règles ([IDs de modèles et versionnage](https://platform.claude.com/docs/en/about-claude/models/model-ids-and-versions)). Pour l’audit et la reproduction, un système enregistre :

- la version d’API et les en-têtes bêta,
- l’ID de modèle plutôt que le seul nom d’affichage,
- la version du prompt système et de la politique,
- la version du jeu d’outils et du schéma JSON,
- les versions de retrieval et de documents,
- les paramètres de génération,
- la corrélation de requête, de tâche et d’utilisateur,
- le Stop Reason, l’utilisation et le résultat de validation.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Claude-Modellinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$headers = @{
  'x-api-key' = $env:ANTHROPIC_API_KEY
  'anthropic-version' = '2023-06-01'
}
Invoke-RestMethod -Headers $headers -Uri 'https://api.anthropic.com/v1/models' |
  ConvertTo-Json -Depth 12</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">curl --fail --silent --show-error https://api.anthropic.com/v1/models \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H 'anthropic-version: 2023-06-01' | jq .</code></pre>
  </div>
</div>

[`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod) et [`ConvertTo-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertto-json) traitent Windows ; [`curl`](https://curl.se/docs/manpage.html) et [`jq`](https://jqlang.org/manual/) traitent Unix. La clé provient d’un magasin de secrets et n’est ni écrite dans la ligne de commande ni affichée.

## Tokenisation, contexte et coûts

Les fenêtres de contexte se mesurent en tokens, et non en caractères ou en fichiers. Les schémas d’outils, le prompt système, les messages, les images, les documents et les résultats d’outils contribuent à l’entrée ; le texte généré et, le cas échéant, le Thinking contribuent à la sortie. L’[API Token Counting](https://platform.claude.com/docs/en/build-with-claude/token-counting) accepte la même structure d’entrée que Messages et fournit une estimation préalable.

Un grand contexte n’est pas une archive. Plus le client envoie de données non pertinentes, plus les coûts, la latence et le risque d’instructions contradictoires augmentent. Les systèmes de production construisent donc une **pipeline d’assemblage du contexte** :

1. Vérifier les droits utilisateur et tenant avant le retrieval.
2. Récupérer les documents au moyen d’IDs et de versions stables.
3. Marquer le contenu non fiable comme données, et non comme instructions système.
4. Limiter le volume de données, les types de fichiers et le budget de tokens.
5. Inclure les métadonnées des sources et les hash.
6. Valider la réponse par rapport aux mêmes sources et schémas.

Les Search Result Content Blocks peuvent transmettre des sources RAG avec titre et provenance afin que Claude génère des citations ([Résultats de recherche](https://platform.claude.com/docs/en/build-with-claude/search-results)). Ces citations ne sont fiables qu’à la hauteur du retrieval, de l’identité du document et des métadonnées transmises.

## Prompt Caching

Le Prompt Caching stocke des préfixes réutilisables et réduit les coûts de traitement et la latence. La hiérarchie du cache suit `tools` → `system` → `messages`; les modifications d’une partie antérieure l’invalident ainsi que les niveaux suivants. La [documentation Prompt Caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) décrit des breakpoints automatiques et explicites ainsi que des TTL courts.

Un cache hit prouve uniquement qu’un préfixe identique a été réutilisé. Il ne garantit ni des données source actuelles ni une sortie identique. Les métriques de cache sont collectées séparément pour les créations et les lectures. Les workspaces isolent les Prompt Caches sur l’API Claude directe ([Workspaces](https://platform.claude.com/docs/en/manage-claude/workspaces)). Les secrets ne doivent pas figurer dans les prompts malgré la brièveté du TTL ; cache et rétention sont des mécanismes distincts.

## Streaming avec Server-Sent Events

Avec `stream: true`, l’API fournit des Server-Sent Events incrémentiels. Les Content Blocks sont commencés, étendus par des deltas puis clôturés ; les informations finales du message et d’utilisation restent nécessaires à un traitement correct. La [documentation Streaming](https://platform.claude.com/docs/en/build-with-claude/streaming) décrit les types d’événements et les accumulateurs SDK.

Un delta de texte visible n’est pas un commit. En cas de coupure de connexion, l’utilisateur peut déjà avoir vu un texte partiel alors que le client ne possède pas de message complet. Les arguments JSON ou d’outil ne doivent être utilisés qu’après réception du bloc complet et validation du schéma. Les gateways et proxys doivent gérer correctement les longues connexions HTTP, le buffering, les timeouts et la backpressure.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Claude-Messages und Streaming">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$body = @{
  model = $env:CLAUDE_MODEL
  max_tokens = 256
  messages = @(@{ role = 'user'; content = 'Antworte mit einem Satz.' })
} | ConvertTo-Json -Depth 8
curl.exe --no-buffer --fail-with-body https://api.anthropic.com/v1/messages `
  -H "x-api-key: $env:ANTHROPIC_API_KEY" `
  -H 'anthropic-version: 2023-06-01' -H 'content-type: application/json' `
  --data-raw $body</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">jq -n --arg model "$CLAUDE_MODEL" '{
  model: $model, max_tokens: 256,
  messages: [{role: "user", content: "Antworte mit einem Satz."}]
}' | curl --no-buffer --fail-with-body https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H 'anthropic-version: 2023-06-01' -H 'content-type: application/json' \
  --data-binary @-</code></pre>
  </div>
</div>

La requête d’exemple utilise volontairement un ID de modèle lu depuis la configuration. Pour SSE, `stream: true` est également défini et le flux d’événements nommé est analysé conformément au protocole ; une simple lecture ligne par ligne ne suffit pas pour du code de production.

La sortie texte ne modifie encore aucun système. Seul Tool Use relie une proposition du modèle à une action — et c’est précisément à cet endroit que l’application, l’autorisation et l’approbation humaine doivent intervenir.

## Tool Use : le modèle demande, l’application agit

Tool Use est un contrat entre le modèle et l’orchestrateur. L’application décrit un outil par un nom, un objectif et un schéma JSON. Claude peut alors générer un bloc `tool_use` avec des arguments. Le client valide le nom et l’entrée, autorise l’appel concret, l’exécute et renvoie un bloc `tool_result` avec le même ID Tool Use. Seul un tour ultérieur du modèle peut former une réponse à partir de celui-ci ([Fonctionnement de Tool Use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works)).

La [vue d’ensemble des outils](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview) distingue :

- **Client Tools :** l’application propriétaire exécute.
- **Anthropic Schema Tools :** Anthropic définit le schéma, mais le client exécute toujours.
- **Server Tools :** Anthropic exécute sur son infrastructure et fournit des blocs de résultats.
- **MCP Connector :** l’API se connecte à un serveur MCP distant.

Cette distinction détermine les chemins réseau, les secrets, la journalisation, la protection des données et le périmètre de défaillance. `strict: true` impose la conformité au schéma, mais non l’exactitude métier ou l’autorisation. Un appel syntaxiquement valide `delete_user(id)` peut toujours supprimer le mauvais utilisateur.

### Boucle d’outils prête pour la production

Pour chaque Tool Use, un orchestrateur sûr effectue les étapes suivantes :

1. Vérifier le nom de l’outil par rapport à une allowlist et à la phase du workflow.
2. Valider strictement le schéma JSON et rejeter les champs inconnus.
3. Revérifier côté serveur les autorisations utilisateur, tenant et objet.
4. Normaliser les entrées ; limiter les chemins, URL, IDs et tailles.
5. Classer les actions en lecture, écriture, message externe et action destructive.
6. Exiger, pour les risques élevés, une approbation métier ou le principe des quatre yeux.
7. Exécuter avec une identité éphémère et minimale dans un runtime isolé.
8. Définir un timeout, une limite de sortie et une clé d’idempotence.
9. Nettoyer le résultat des secrets et des instructions non fiables.
10. Auditer de manière immuable le Tool Use, la décision, l’effet et le résultat.

La boucle limite les tours, les outils parallèles, les coûts cumulés et les erreurs répétées. Un résultat d’outil est à son tour du contenu non fiable. Une ligne de base de données ou une page web peut contenir une prompt injection et ne doit pas remplacer la politique de l’orchestrateur.

## MCP comme frontière de protocole

Le Model Context Protocol standardise la connexion des applications d’IA aux outils, ressources et prompts. Anthropic décrit MCP comme un protocole client-serveur ouvert ([vue d’ensemble MCP](https://docs.anthropic.com/en/docs/mcp)) ; la [spécification MCP](https://modelcontextprotocol.io/specification/2025-06-18) normative définit les messages et capacités.

MCP rend les intégrations interchangeables, mais pas automatiquement dignes de confiance. Un serveur peut lire des données, déclencher des actions, fournir de très grands résultats ou retourner le contenu d’un tiers. Son ajout à un fichier de configuration constitue donc une décision logicielle et d’autorisation. Il faut inventorier l’opérateur, le transport, l’endpoint, l’authentification, les outils, les ressources, les scopes, les classes de données, la version, le timeout, la limite de sortie et la révocation.

Le [MCP Connector de l’API Messages](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector) est un outil serveur sur l’infrastructure d’Anthropic. À l’inverse, un serveur stdio lancé localement dans Claude Code s’exécute sur l’endpoint. Même protocole, autres chemins de données et autres périmètres de défaillance.

## Claude Code : runtime local des agents

Claude Code est un agent pour le terminal et les dépôts. Il collecte le contexte du projet, l’envoie à un endpoint de modèle, interprète les propositions du modèle et utilise des outils locaux pour lire, modifier et exécuter. La [configuration](https://docs.anthropic.com/en/docs/claude-code/getting-started) officielle documente Windows via WSL ou Git Bash, ainsi que macOS et Linux. La [référence CLI](https://docs.anthropic.com/en/docs/claude-code/cli-usage) décrit le fonctionnement interactif, le Print Mode, la sortie JSON, la reprise de session, le choix de modèle et les règles d’outils.

La principale frontière de confiance est le processus local. Claude Code ne peut agir qu’avec les droits du système d’exploitation et les credentials accessibles de l’utilisateur qui l’a lancé, mais ces droits peuvent être très étendus : dépôt, agent SSH, CLI cloud, registre de paquets, Kubernetes, cookies de navigateur, variables d’environnement ou tunnel de production. « L’agent a demandé » n’est pas une isolation.

La [documentation de sécurité de Claude Code](https://docs.anthropic.com/en/docs/claude-code/security) décrit les valeurs par défaut en lecture seule, les demandes d’autorisation, les limites de projet et la protection contre la prompt injection. Elle précise également qu’aucun système n’est totalement immunisé. L’exploitation non interactive nécessite des politiques plus strictes, car le contenu du dépôt, le texte des issues, les logs de build et les sorties d’outils peuvent contenir des instructions d’attaquants.

### Profil d’exécution et de droits

Un endpoint géré ou un runner CI utilise :

- un compte utilisateur ou de workload dédié,
- un répertoire de travail spécifique au projet,
- des droits minimaux de système de fichiers et de réseau,
- des credentials éphémères, limités à la cible et à l’action,
- des outils autorisés et interdits définis par une politique centrale,
- un conteneur, une VM ou une sandbox pour les builds non fiables,
- un Secret Scanning avant l’ajout au contexte,
- des tours, un temps, une sortie et des coûts limités,
- un Git Diff, des tests et une approbation avant commit ou déploiement,
- la corrélation des sessions, outils et fournisseurs dans le journal d’audit.

Le commutateur `--dangerously-skip-permissions` n’est pas une stratégie d’automatisation. Il supprime une couche de protection et n’est justifiable que dans une sandbox en amont dotée de sa propre politique.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Claude-Code-Inventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Command claude | Format-List Source,Version
claude doctor
Get-FileHash (Get-Command claude).Source -Algorithm SHA256
Get-ChildItem .\.claude -Force -Recurse | Select-Object FullName,Length</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">command -v claude
claude doctor
file "$(command -v claude)"
sha256sum "$(command -v claude)"
find ./.claude -maxdepth 3 -type f -printf '%p %s\n'</code></pre>
  </div>
</div>

[`Get-Command`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/get-command), [`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash) et [`Get-ChildItem`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-childitem) inventorient Windows. Les commandes POSIX [`command`](https://pubs.opengroup.org/onlinepubs/9799919799/utilities/command.html), [`file`](https://man7.org/linux/man-pages/man1/file.1.html), [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) et [`find`](https://man7.org/linux/man-pages/man1/find.1.html) prennent en charge Unix. Un hash n’est évalué que par rapport à une source de version ou de paquet digne de confiance.

## Exploitation non interactive et CI

Le Print Mode rend Claude Code scriptable. `--output-format json` ou `stream-json` fournit des résultats lisibles par machine ; `--max-turns` limite l’exécution de l’agent. Les données d’entrée et le code de sortie restent partie intégrante du protocole de tâche. Une sortie en texte libre n’est pas évaluée comme script shell.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für begrenzten Claude-Code-Print-Mode">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Content .\review-request.txt -Raw |
  claude -p --max-turns 3 --output-format json `
    --allowedTools 'Read' --disallowedTools 'Bash' 'Edit' |
  Tee-Object -FilePath .\claude-review.json |
  ConvertFrom-Json</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">cat ./review-request.txt |
  claude -p --max-turns 3 --output-format json \
    --allowedTools Read --disallowedTools Bash Edit |
  tee ./claude-review.json | jq .</code></pre>
  </div>
</div>

[`Get-Content`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-content), [`Tee-Object`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/tee-object) et [`ConvertFrom-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json) traitent Windows. [`cat`](https://www.gnu.org/software/coreutils/manual/html_node/cat-invocation.html) et [`tee`](https://www.gnu.org/software/coreutils/manual/html_node/tee-invocation.html) prennent en charge Unix. Un job CI reçoit en outre des règles fixes de répertoire de travail, réseau, secrets et branche.

## Settings, Hooks et politiques

Claude Code combine des paramètres utilisateur, projet et gérés. Les fichiers de projet peuvent se trouver dans le dépôt et font donc partie de la code review. Les hooks exécutent des commandes à des points définis du cycle de vie ; ils sont du code exécutable, et non une configuration de prompt inoffensive. Les serveurs MCP étendent les systèmes accessibles. Une politique d’entreprise doit donc contrôler conjointement les settings, hooks, plugins, skills, MCP et motifs shell autorisés.

La CLI peut autoriser des permissions une fois ou durablement. De larges wildcards accélèrent le travail, mais augmentent le blast radius. Une bonne règle n’autorise pas « Bash », mais une fonction de lecture limitée ou un outil typé placé en amont. Pour les opérations d’écriture, le chemin cible, la branche et le diff sont vérifiés. Les déploiements ou messages externes restent soumis à une approbation distincte, hors du modèle.

## Identité, organisation et workspaces

La Claude Platform rattache l’utilisation à une organisation et à des workspaces. Les clés de workspace sont limitées aux ressources et à l’utilisation de ce workspace ; les membres possèdent des rôles de workspace. Des workspaces distincts pour le développement, le test et la production séparent les clés, limites, batches, fichiers et Prompt Caches ([Workspaces](https://platform.claude.com/docs/en/manage-claude/workspaces)).

L’API Admin et l’API Inference utilisent différents types de clés. La [documentation de l’API Admin](https://platform.claude.com/docs/en/manage-claude/admin-api) décrit les membres, invitations, workspaces et clés API. Une clé administrateur ne doit jamais se trouver dans une application qui doit uniquement envoyer des Messages. Le départ d’un collaborateur révoque l’accès utilisateur, les clés personnelles Claude Code et, le cas échéant, les clés de service créées séparément.

Claude Code peut s’authentifier via Console/OAuth, les plans Claude ou un fournisseur d’entreprise. L’élément décisif est le **chemin réel de facturation et de données** de la session. Dans le cas contraire, des comptes personnels dans un dépôt d’entreprise contournent les contrôles de workspace, la rétention, les centres de coûts ou l’audit.

## Limites de débit, dépenses et backpressure

Anthropic distingue les Spend Limits et les Rate Limits. Les Messages sont limités selon les requêtes par minute, les tokens d’entrée par minute et les tokens de sortie par minute. Les limites utilisent des token buckets ; de courtes rafales peuvent donc déclencher des 429 malgré une moyenne par minute apparemment conforme. `retry-after` et les en-têtes de réponse fournissent le cadre de backoff ([Rate limits](https://platform.claude.com/docs/en/api/rate-limits)).

Une gateway implémente :

- une file d’attente et des priorités par tenant et workflow,
- le comptage des tokens avant l’acceptation de grands jobs,
- un backoff exponentiel avec jitter et `retry-after`,
- des limites de parallélisme globales et propres au workspace,
- des circuit breakers pour les 5xx et les erreurs réseau,
- des alertes de budget et une limite stricte de coûts,
- des métriques distinctes pour les lectures de cache, écritures de cache, entrées et sorties,
- un fallback contrôlé avec un changement de qualité documenté.

La [documentation des erreurs API](https://platform.claude.com/docs/en/api/errors) distingue 400, 401, 402, 403, 404, 413, 429, 500 et 529 et fournit `request_id`. Les SDK répètent automatiquement certaines erreurs transitoires. Ces nouvelles tentatives sont prises en compte dans la planification de capacité et des coûts.

## Batches et traitement asynchrone

L’API Message Batches traite de nombreuses requêtes indépendantes de manière asynchrone. Les batches sont liés à un workspace ; chaque résultat peut réussir ou échouer. La disponibilité des résultats et la conservation côté serveur diffèrent du chemin synchrone. La [documentation Batch](https://platform.claude.com/docs/en/build-with-claude/batch-processing) indique les limites de taille, de durée d’exécution et de récupération.

Un job batch possède un `custom_id` par objet métier, un manifeste d’entrée, un curseur de récupération et un contrôle des résultats. « Batch completed » signifie seulement que toutes les entrées ont un état final. L’import traite chaque résultat de manière idempotente, détecte les IDs absents et archive les versions de modèle, de prompt et de schéma. Pour les données personnelles ou réglementées, il faut vérifier si le chemin batch est admissible au regard du contrat et du modèle de conservation des données.

Après la fonction et l’échelle vient la question des données : quels contenus quittent le système propriétaire, combien de temps sont-ils conservés et quels opérateurs supplémentaires sont impliqués sur les plateformes cloud ?

## Conservation des données, confidentialité et plateformes tierces

La conservation des données dépend de l’interface et du contrat. Pour l’utilisation commerciale de l’API, Anthropic décrit une suppression standard des entrées et sorties dans les 30 jours, mais mentionne des exceptions pour les fonctions avec conservation plus longue, les accords divergents, l’application de la sécurité et les obligations légales ([Conservation des données commerciales](https://privacy.claude.com/en/articles/7996866-how-long-do-you-store-my-organization-s-data)). Les applications Claude stockent les conversations selon leurs propres règles pour les fonctions produit.

La Zero Data Retention n’est pas un commutateur global pour chaque produit. La [documentation ZDR](https://privacy.claude.com/en/articles/8956058-i-have-a-zero-data-retention-agreement-with-anthropic-what-products-does-it-apply-to) décrit les API, organisations et exceptions admissibles. En cas d’accès via un fournisseur cloud, son propre chemin de données, ses régions, sa gestion des clés et son contrat s’appliquent en plus. MCP, Web Search ou des outils externes peuvent transmettre des données à d’autres responsables.

Avant la mise en production, une matrice des flux de données est créée :

| Chemin | Données | Contrôle responsable |
|---|---|---|
| Client → endpoint de modèle | Prompt, fichiers, images, schéma d’outil | Classification, minimisation, contrat, région |
| Endpoint de modèle → client | Content Blocks, Usage, Request ID | Validation, redaction, journalisation |
| Client → outil/MCP | Arguments d’outil, contexte utilisateur | Autorisation, scope, DPA, audit |
| Outil → modèle | Résultat et contenu non fiable | Filtre, limite de sortie, protection contre l’injection |
| Claude Code local | Dépôt, shell, credentials | Politique d’endpoint, sandbox, secrets |
| Logs/tracing | Prompts, résultats, métadonnées | Redaction, accès, conservation |

## Prompt injection et contenu non fiable

La prompt injection survient lorsque des données tentent de devenir des instructions. Les surfaces d’attaque comprennent les pages web, e-mails, documents, issues, commentaires du code source, ressources MCP, résultats d’outils et sortie du terminal. L’attaque n’a pas besoin de « convaincre » le modèle si l’application accorde de toute façon des droits étendus à un Tool Use non contrôlé.

La défense est multicouche :

1. Imposer la politique système et les autorisations en dehors du texte du modèle.
2. Filtrer le retrieval selon les droits utilisateur et tenant.
3. Marquer les blocs de données avec leur provenance, niveau de confiance et finalité.
4. Autoriser les outils de manière minimale, typée et liée à l’objet.
5. Approuver séparément les actions d’écriture, d’envoi, de paiement et de suppression.
6. Sandboxer le réseau et le système de fichiers de l’exécution.
7. Ne pas transmettre de secrets dans le contexte, les résultats d’outils ou les logs visibles.
8. Valider la sortie et les arguments d’outils par rapport au schéma et aux règles métier.
9. Limiter les boucles d’agents par le temps, les tours, les coûts et les effets.
10. Réaliser des tests adversariaux avec injection indirecte dans les véritables chemins de données.

Un Human-in-the-Loop n’est efficace que si la personne voit la cible, l’effet et les données pertinentes. Un message générique « Autoriser ? » conduit à la fatigue d’approbation. Les actions à haut risque présentent des paramètres normalisés et sont contrôlées par un Policy Enforcement Point indépendant.

## Chemin réseau et exploitation par proxy

L’API Claude et Claude Code nécessitent un accès HTTPS. Un proxy d’entreprise peut assurer l’authentification, l’inspection TLS, le contrôle de l’egress et la journalisation, mais devient ainsi partie de la chaîne de confidentialité et de disponibilité. La [documentation proxy pour Claude Code](https://docs.anthropic.com/en/docs/claude-code/corporate-proxy) indique les variables de proxy prises en charge, les bundles CA et les cibles requises.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Claude-DNS-, TCP- und TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName api.anthropic.com
Test-NetConnection api.anthropic.com -Port 443
curl.exe -sS -D - -o NUL https://api.anthropic.com/v1/models
Get-NetTCPConnection -State Established | Where-Object RemotePort -eq 443</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig +short api.anthropic.com A api.anthropic.com AAAA
curl -sS -D - -o /dev/null https://api.anthropic.com/v1/models
ss -ntp state established '( dport = :443 )'
openssl s_client -connect api.anthropic.com:443 -servername api.anthropic.com -brief</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname), [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) et [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) vérifient Windows. [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility), [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) et [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) vérifient Unix. Un 401 sans clé confirme que DNS, TCP, TLS et HTTP atteignent le chemin API attendu ; ce n’est pas un test d’inférence réussi.

Une exploitation sûre des agents ne doit pas seulement compter les appels au modèle, mais rendre traçable l’intégralité du chemin, de l’entrée à l’effet externe en passant par la décision d’outil.

## Observabilité et audit

Une requête de production génère des métriques métier, techniques et de coûts. Les éléments suivants sont au minimum enregistrés :

- l’heure, le workspace, l’application, le workflow et une corrélation utilisateur pseudonyme,
- l’ID de modèle, la version d’API, la version de prompt, de jeu d’outils et de schéma,
- la latence jusqu’aux en-têtes, au premier token et au message complet,
- les tokens d’entrée, de création de cache, de lecture de cache et de sortie,
- le Stop Reason, les types de Content Blocks et le statut de validation,
- la classe d’erreur, le nombre de retries, `request_id` et `retry-after`,
- le nom de l’outil, la décision d’autorisation, la durée, l’effet et la classe de résultat,
- les coûts estimés et facturés,
- la classe de redaction et de conservation.

Les prompts et les réponses du modèle ne sont pas automatiquement journalisés dans leur intégralité. Ils peuvent contenir des données personnelles, des secrets, du code source ou des payloads d’attaquants. Les métadonnées d’audit et le contenu de débogage reçoivent des stockages, rôles et délais de suppression distincts. L’[API Usage Report](https://platform.claude.com/docs/en/api/admin/usage_report) fournit des données agrégables sur l’utilisation de l’API et de Claude Code ; les effets locaux des outils doivent néanmoins être audités dans le système propriétaire.

## Gestion des erreurs et reprise

Une requête au modèle n’est pas automatiquement idempotente. En cas de timeout, le fournisseur peut avoir terminé la génération alors que le client n’a reçu aucune réponse. Avec Tool Use, l’effet externe peut déjà avoir eu lieu. L’orchestrateur enregistre donc les transitions d’état :

`accepted → context_built → request_sent → response_received → validated → tool_authorized → tool_executed → result_returned → completed`

Chaque transition possède un ID de corrélation, une heure, des versions et un checkpoint. Une nouvelle tentative avant `tool_executed` peut répéter le tour du modèle ; après cela, elle vérifie d’abord l’effet métier. Les outils externes prennent en charge des clés d’idempotence ou une preuve Read-after-Write. En cas d’état inconnu, il faut escalader, et non exécuter aveuglément une nouvelle fois.

Les modèles de fallback modifient la qualité, les coûts, la limite de contexte et le comportement des outils. Ils constituent un chemin distinct testé. L’application ne cache pas un fallback lorsqu’il modifie la portée de l’affirmation ou la conformité.

## Backup et recovery

Le modèle lui-même est un service externe qui n’est pas sauvegardé par le client. Les données et configurations de contrôle propriétaires doivent pouvoir être restaurées :

- prompts système, politiques et jeux d’évaluation,
- schémas d’outils, implémentations et règles d’autorisation,
- inventaire MCP et versions de serveurs approuvées,
- provisionnement des workspaces et des clés sous forme de code ou de runbook,
- sources de retrieval, embeddings/index et versions de documents,
- états de conversation et de jobs ainsi que données d’idempotence,
- métadonnées de redaction, d’audit et de coûts,
- settings de projet Claude Code, hooks et skills vérifiés,
- adaptateurs de fournisseurs et migrations de modèles testées.

Les clés API ne sont pas « restaurées » depuis les backups ; elles sont révoquées et nouvellement provisionnées. Un test de disaster recovery construit un nouveau runtime, attribue des identités minimales, envoie un cas de test connu, valide le schéma et les sources, exécute un outil de test inoffensif et corrèle les logs du fournisseur, de la gateway et des outils. Les bases figurent sous [Backup et Disaster Recovery](/kb/backup-dr), [APIs](/kb/apis), [Durcissement](/kb/haertung) et [Dépannage](/kb/troubleshooting).

## Histoire technique

Anthropic a été créée en 2021 comme entreprise de recherche et de produits pour des systèmes d’IA sûrs. Claude a été présenté en mars 2023, après une phase fermée avec des partenaires, comme assistant de chat et d’API ([Présentation de Claude](https://www.anthropic.com/news/introducing-claude)). Claude 2 a étendu le contexte et la disponibilité publique en juillet 2023 ([Claude 2](https://www.anthropic.com/news/claude-2)).

En 2024, la famille Claude 3 a établi les dénominations Haiku, Sonnet et Opus comme échelons de vitesse, de coûts et de performance ([Claude 3](https://www.anthropic.com/news/claude-3-family)). Ces noms de produits ne constituent pas une vérification stable des capacités ; les applications lient des IDs épinglés et interrogent les limites séparément.

En février 2025, Claude Code est apparu comme Research Preview avec un modèle de raisonnement hybride ([Claude 3.7 et Claude Code](https://www.anthropic.com/news/claude-3-7-sonnet)). En mai 2025, Claude Code est devenu généralement disponible ; Anthropic a simultanément complété l’API avec des composants orientés agents tels que Code Execution, MCP Connector et un Prompt Caching plus long ([Claude 4](https://www.anthropic.com/news/claude-4)). La question opérationnelle est ainsi passée de « Comment consommer du texte ? » à « Quelles données et quels effets une couche de planification probabiliste peut-elle atteindre ? »

La plateforme a ensuite développé un versionnage de modèles plus fort, les workspaces, les API d’utilisation et de coûts, les schémas d’outils, le caching, les batches et les SDK d’agents. Le point d’architecture durable reste le même : le modèle propose ; les API et frameworks d’agents transportent ; l’application de contrôle autorise, exécute, valide et audite.

## Checklist administrateur

Les nombreuses interfaces de Claude deviennent maîtrisables lorsque le modèle, l’API, l’outil et le processus local sont recensés séparément. La checklist résume les contrôles qui doivent être démontrés avant un usage en production.

- **Couches :** inventorier séparément le modèle, l’API, l’application Claude, la gateway, l’outil et l’endpoint Claude Code.
- **Contrats :** traiter la version d’API, l’ID de modèle épinglé, les Content Blocks, les Stop Reasons et les types inconnus.
- **Contexte :** minimiser les données, autoriser le retrieval, compter les tokens et versionner les sources.
- **Outils :** imposer schéma, autorisation métier, idempotence, timeout, limite de sortie et audit.
- **MCP :** vérifier la provenance du serveur, le transport, les scopes, les classes de données et le risque de prompt injection.
- **Claude Code :** limiter les droits utilisateur, les secrets, le réseau, la sandbox, les settings, les hooks et la politique des outils.
- **Identité :** séparer proprement organisation, workspace, clés utilisateur, API et administrateur.
- **Capacité :** surveiller RPM, ITPM, OTPM, dépenses, cache et coûts des retries.
- **Confidentialité :** évaluer séparément l’interface produit, le fournisseur, la rétention, ZDR, les outils tiers et les logs.
- **Qualité :** utiliser les évaluations, les contrôles de schéma et de sources ainsi que l’approbation métier avant tout effet.
- **Recovery :** maintenir les prompts, politiques, outils, retrieval, jobs et provisionnement de manière reproductible.
- **Preuve :** exécuter un test complet, de l’authentification au rapport d’audit et de coûts en passant par Message et l’outil.

## Sources

- [Anthropic – Présentation de Claude](https://www.anthropic.com/news/introducing-claude)
- [Claude Platform – Vue d’ensemble des modèles](https://platform.claude.com/docs/en/about-claude/models/overview)
- [Anthropic – Constitution de Claude](https://www.anthropic.com/research/claudes-constitution)
- [Anthropic – Fiche de modèle Claude 3](https://assets.anthropic.com/m/61e7d27f8c8f5919/original/Claude-3-Model-Card.pdf)
- [Claude API – Créer un message](https://platform.claude.com/docs/en/api/messages/create)
- [Claude Platform – Stop Reasons](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons)
- [Claude Platform – Versionnage d’API](https://platform.claude.com/docs/en/api/versioning)
- [Claude Platform – IDs et versions de modèles](https://platform.claude.com/docs/en/about-claude/models/model-ids-and-versions)
- [Microsoft – Invoke-RestMethod](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod)
- [Microsoft – ConvertTo-Json](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertto-json)
- [curl – Manuel](https://curl.se/docs/manpage.html)
- [jq – Manuel](https://jqlang.org/manual/)
- [Claude Platform – Comptage des tokens](https://platform.claude.com/docs/en/build-with-claude/token-counting)
- [Claude Platform – Résultats de recherche et citations](https://platform.claude.com/docs/en/build-with-claude/search-results)
- [Claude Platform – Prompt Caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [Claude Platform – Workspaces](https://platform.claude.com/docs/en/manage-claude/workspaces)
- [Claude Platform – Streaming des messages](https://platform.claude.com/docs/en/build-with-claude/streaming)
- [Claude Platform – Fonctionnement de Tool Use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works)
- [Claude Platform – Vue d’ensemble de Tool Use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)
- [Anthropic – Model Context Protocol](https://docs.anthropic.com/en/docs/mcp)
- [Model Context Protocol – Spécification](https://modelcontextprotocol.io/specification/2025-06-18)
- [Claude Platform – MCP Connector](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector)
- [Anthropic – Configurer Claude Code](https://docs.anthropic.com/en/docs/claude-code/getting-started)
- [Anthropic – Référence CLI de Claude Code](https://docs.anthropic.com/en/docs/claude-code/cli-usage)
- [Anthropic – Sécurité de Claude Code](https://docs.anthropic.com/en/docs/claude-code/security)
- [Microsoft – Get-Command](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/get-command)
- [Microsoft – Get-FileHash](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash)
- [Microsoft – Get-ChildItem](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-childitem)
- [POSIX – command](https://pubs.opengroup.org/onlinepubs/9799919799/utilities/command.html)
- [Linux man-pages – file(1)](https://man7.org/linux/man-pages/man1/file.1.html)
- [GNU Coreutils – utilitaires sha2](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Linux man-pages – find(1)](https://man7.org/linux/man-pages/man1/find.1.html)
- [Microsoft – Get-Content](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-content)
- [Microsoft – Tee-Object](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/tee-object)
- [Microsoft – ConvertFrom-Json](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json)
- [GNU Coreutils – cat](https://www.gnu.org/software/coreutils/manual/html_node/cat-invocation.html)
- [GNU Coreutils – tee](https://www.gnu.org/software/coreutils/manual/html_node/tee-invocation.html)
- [Claude Platform – API Admin](https://platform.claude.com/docs/en/manage-claude/admin-api)
- [Claude Platform – Rate limits](https://platform.claude.com/docs/en/api/rate-limits)
- [Claude Platform – Erreurs API](https://platform.claude.com/docs/en/api/errors)
- [Claude Platform – Traitement par batch](https://platform.claude.com/docs/en/build-with-claude/batch-processing)
- [Anthropic Privacy Center – Conservation des données commerciales](https://privacy.claude.com/en/articles/7996866-how-long-do-you-store-my-organization-s-data)
- [Anthropic Privacy Center – Zero Data Retention](https://privacy.claude.com/en/articles/8956058-i-have-a-zero-data-retention-agreement-with-anthropic-what-products-does-it-apply-to)
- [Anthropic – Proxy d’entreprise Claude Code](https://docs.anthropic.com/en/docs/claude-code/corporate-proxy)
- [Microsoft – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [Microsoft – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [Microsoft – Get-NetTCPConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Linux man-pages – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Claude API – Usage Report](https://platform.claude.com/docs/en/api/admin/usage_report)
- [Anthropic – Claude 2](https://www.anthropic.com/news/claude-2)
- [Anthropic – Famille Claude 3](https://www.anthropic.com/news/claude-3-family)
- [Anthropic – Claude 3.7 et Claude Code](https://www.anthropic.com/news/claude-3-7-sonnet)
- [Anthropic – Claude 4 et disponibilité générale de Claude Code](https://www.anthropic.com/news/claude-4)
