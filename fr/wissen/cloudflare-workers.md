---
title: "Cloudflare Workers : isolates, bindings et exploitation en périphérie"
blatt: "cloudflare-workers"
description: "Cloudflare Workers pour les administrateurs d’infrastructure, de messagerie et de plateformes : chemin des requêtes, workerd et isolats V8, gestionnaires et API Web, bindings de capacité, KV, D1, R2 et Durable Objects, routage, Compatibility Dates, limites, observabilité, déploiements, reprise et histoire technique."
fakten:
  - label: Rôle système
    wert: plateforme de calcul événementielle dans le réseau Cloudflare pour HTTP, planifications, files d’attente, e-mails et services internes
    href: https://developers.cloudflare.com/workers/runtime-apis/handlers/
  - label: Runtime
    wert: workerd avec isolats V8 ; aucun processus serveur persistant propre ni instance de conteneur par application
    href: https://developers.cloudflare.com/workers/reference/how-workers-works/
  - label: Modèle de programmation
    wert: points d’entrée de modules ES avec des API standard du Web telles que Request, Response, Fetch, Streams et Web Crypto
    href: https://developers.cloudflare.com/workers/runtime-apis/
  - label: Langages
    wert: JavaScript, TypeScript, Python et Rust ; autres langages via WebAssembly
    href: https://developers.cloudflare.com/workers/languages/
  - label: Voies d’entrée
    wert: workers.dev, route, Custom Domain ainsi que les événements Fetch, Scheduled, Queue, E-Mail, Alarm et Tail
    href: https://developers.cloudflare.com/workers/wrangler/configuration/
  - label: Bindings
    wert: capacité et API pour les ressources de plateforme sans clés d’accès exposées dans le code
    href: https://developers.cloudflare.com/workers/runtime-apis/bindings/
  - label: Modèles d’état
    wert: cache local · KV eventual-consistent · D1 SQL · Durable Object fortement cohérent · objet R2 · file d’attente
    href: https://developers.cloudflare.com/workers/platform/storage-options/
  - label: Compatibilité
    wert: Compatibility Date et flags facultatifs figent les changements de runtime affectant le comportement pour chaque déploiement
    href: https://developers.cloudflare.com/workers/configuration/compatibility-flags/
  - label: Control Plane
    wert: configuration Wrangler, API Workers, objets de version, déploiements, secrets et bindings de ressources
    href: https://developers.cloudflare.com/workers/configuration/
  - label: Déploiement progressif
    wert: téléverser une version, créer un déploiement, répartir progressivement le trafic et revenir à une version stable
    href: https://developers.cloudflare.com/workers/versions-and-deployments/gradual-deployments/
  - label: Signaux d’exploitation
    wert: résultat d’invocation, statut, exception, CPU et Wall Time, sous-requêtes, logs, traces et ID de version
    href: https://developers.cloudflare.com/workers/observability/logs/workers-logs/
  - label: Objets de reprise
    wert: artefact de code, configuration, Compatibility Date, bindings, secrets, exports de données, cycle de vie des DO et rollback testé
    href: https://developers.cloudflare.com/workers/versions-and-deployments/
werbung:
  - newsletter
ctaThemen:
  - cloudflare-workers
translationSourceHash: 4c5445ed6d4ead36638b1de03ed9d67a4f82304b58f28d83a9c72e29e760d065
translationModel: gpt-5.6-terra
translatedAt: 2026-10-06T10:49:39.487Z
translationReview: automatic
---

# Cloudflare Workers : isolates, bindings et exploitation en périphérie

Cloudflare Workers est une plateforme de calcul événementielle au sein du réseau Cloudflare. Un Worker peut répondre à des requêtes HTTP ou les transférer, exécuter des tâches planifiées, consommer des messages de file d’attente, traiter des e-mails entrants et servir de service interne à d’autres Workers. L’exploitant ne gère ni processus d’écoute ni VM individuelle. Il gère le code, les points d’entrée, les routes, la compatibilité de la runtime, les autorisations d’accès aux ressources de la plateforme, les versions et les données d’exploitation.

Le terme « Edge » ne décrit qu’un emplacement d’exécution possible, et non l’architecture complète. Un Worker HTTP peut s’exécuter près de l’utilisateur ; Smart Placement peut rapprocher le calcul d’un backend ; les Durable Objects ont chacun un emplacement d’état unique ; D1, KV, R2 et les Queues possèdent chacun leurs propres modèles de réplication et de cohérence. Pour les administrateurs, la caractéristique déterminante n’est donc pas « serverless », mais la répartition entre le chemin de données, le Control Plane, les services d’état et les points de défaillance possibles.

L’explication suit une requête depuis le point de terminaison public jusqu’à la runtime Workers, puis vers les bindings, le stockage et les upstreams. Le déploiement, la sécurité, l’observabilité et la reprise sont ensuite situés dans la perspective de l’exploitation de plateforme.

## Architecture : chemin de données, runtime et Control Plane

Un système Workers se compose de plusieurs couches :

1. **Ingress et routage :** Cloudflare reçoit une requête via DNS, TLS et HTTP et associe un nom d’hôte ou un motif de chemin à un Worker ou à un asset statique.
2. **Dispatch d’événements :** la plateforme crée un événement Fetch, Scheduled, Queue, E-Mail, Alarm ou Tail et appelle le point d’entrée approprié.
3. **Runtime :** `workerd` fournit des isolats V8, des API Web, la compatibilité Node.js et des limites d’exécution.
4. **Bindings :** l’objet `env` confère au code des capacités pour KV, D1, R2, Durable Objects, Queues, Secrets, Assets, d’autres Workers et d’autres services de la plateforme.
5. **État et upstreams :** les données résident hors de l’isolat volatile ou dans des stockages Durable Object contrôlés ; les appels sortants utilisent Fetch, Service Bindings ou des API Socket prises en charge.
6. **Control Plane :** Wrangler, le Dashboard et l’API gèrent la configuration, les relations entre ressources, les secrets, les versions, les déploiements et l’observabilité.

Cloudflare décrit les isolats, le temps de calcul par requête et l’exécution distribuée comme les trois principales différences par rapport aux runtimes serveur classiques ([How Workers works](https://developers.cloudflare.com/workers/reference/how-workers-works/)). La référence de la plateforme distingue le Worker lui-même des produits de stockage et de plateforme développeur connectés ([Cloudflare Workers documentation](https://developers.cloudflare.com/workers/)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-cloudflare-workers.svg?v=20260813" title="Interaktive Infografik: Cloudflare-Workers-Aufruf von DNS, TLS und Routing über Event Dispatch, workerd und V8-Isolate bis zu Bindings, Storage, Upstreams, Observability, Versionen und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-cloudflare-workers.svg?v=20260813">Ouvrir directement le graphique interactif</a>.
</iframe>

## Chemin de requête et emplacement d’exécution

Pour une requête HTTP, la configuration Cloudflare décide d’abord si un nom d’hôte `workers.dev`-hostname, une route ou un Custom Domain déclenche le Worker. Une route se place devant une origin existante et peut modifier, répondre à ou transférer des requêtes. Un Custom Domain relie directement un Worker à un nom d’hôte. Les assets statiques peuvent être évalués avant ou après le Worker ; `run_worker_first` modifie cet ordre ([Wrangler configuration – routes](https://developers.cloudflare.com/workers/wrangler/configuration/), [Static Assets – configuration and bindings](https://developers.cloudflare.com/workers/static-assets/binding/)).

Le chemin externe peut être modélisé comme `Client → DNS → Cloudflare Edge → TLS/HTTP → Route → Worker/Asset → Binding oder Origin`. La connexion vers l’origin constitue un nouveau contexte de transport. Le certificat client, l’en-tête `Authorization`, la clé de cache, l’en-tête Host et l’adresse IP source doivent donc être traités consciemment à chaque frontière. Un Service Binding contourne en revanche la résolution de nom publique et fournit un chemin Worker-à-Worker explicitement autorisé ([Service bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/service-bindings/)).

Smart Placement peut exécuter un Worker plus près de ses backends, au lieu de privilégier uniquement la proximité avec l’utilisateur. Cela est utile pour les appels intensifs en base de données, mais peut allonger le mauvais chemin pour les assets ou les transformations Edge pures. Cloudflare documente donc des recommandations différentes pour Assets-first et Worker-first ([Placement](https://developers.cloudflare.com/workers/configuration/placement/)).

## workerd et isolats V8

`workerd` est la runtime JavaScript/WebAssembly orientée serveur derrière Workers. V8 fournit des isolats comme contextes d’exécution séparés. De nombreux isolats peuvent partager un processus sans que chaque application nécessite son propre processus JavaScript ou conteneur. Les API Web sont largement fournies nativement par la runtime ([Introducing workerd](https://blog.cloudflare.com/workerd-open-source-workers-runtime/), [Workers security model](https://developers.cloudflare.com/workers/reference/security-model/)).

Un isolat n’est pas une machine persistante. Il peut être réutilisé, occupé en parallèle par plusieurs requêtes ou supprimé. Deux requêtes consécutives ne doivent pas nécessairement atteindre la même instance ni le même emplacement. Les objets globaux conviennent aux structures d’aide immuables et coûteuses à créer ou aux caches opportunistes, mais pas comme source de vérité. Toute correction qui dépend d’une variable globale entre les requêtes est une erreur d’architecture.

L’isolation V8 ne remplace pas tous les contrôles de sécurité. Cloudflare ajoute une protection de la runtime et des processus, des limites de ressources et des défenses Spectre spécifiques. L’exploitant reste responsable de la validation des entrées, de l’autorisation, du périmètre des secrets, des listes d’autorisation de destinations, des limites de sortie et de la classification des données. La frontière de l’isolat ne protège pas contre une application qui lit ou écrit de mauvaises données avec ses bindings légitimes.

## Modèle d’événements et gestionnaires

Les Workers implémentent des points d’entrée pour différents types d’événements. Le catalogue général des gestionnaires comprend Fetch, Scheduled, Queue, E-Mail, Alarm et Tail ([Handlers](https://developers.cloudflare.com/workers/runtime-apis/handlers/)).

| Gestionnaire | Déclencheur | Frontière de retour et d’erreur | Question d’administration typique |
|---|---|---|---|
| `fetch()` | Requête HTTP ou appel de service | `Response`, flux ou exception | Quelle route et quelle version ont traité la requête ? |
| `scheduled()` | Déclencheur Cron | Résultat Promise/invocation | L’exécution planifiée a-t-elle été déclenchée et terminée ? |
| `queue()` | Lot de messages | Ack, retry ou comportement Dead Letter | Quel message est idempotent, réessayable ou poison ? |
| `email()` | Email Routing | Distribuer, rejeter, transférer | Quelle frontière d’enveloppe et de politique s’applique ? |
| `alarm()` | Alarme Durable Object | Exécution locale à l’objet | Quel objet possède l’alarme et l’état ? |
| `tail()` | Événement de trace d’un Worker producteur | Export asynchrone | L’observabilité peut-elle elle-même échouer ou devenir récursive ? |

Le gestionnaire Fetch reçoit `Request`, `env` et `ctx` et renvoie une `Response` d’API Web ([Fetch handler](https://developers.cloudflare.com/workers/runtime-apis/handlers/fetch/)). `ctx.waitUntil()` enregistre du travail qui peut se poursuivre après l’envoi de la réponse ; il reste soumis aux limites d’exécution et d’erreur documentées et n’est pas un serveur de tâches durable ([Context – waitUntil](https://developers.cloudflare.com/workers/runtime-apis/context/)). Pour des processus durables et répétables, les Queues ou Workflows sont mieux adaptés qu’une chaîne de Background Promises non observées.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für ein minimales Workers-Modul">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-typescript">export interface Env {
  UPSTREAM: Fetcher;
  RELEASE: string;
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise&lt;Response&gt; {
    const started = Date.now();
    const response = await env.UPSTREAM.fetch(request);
    ctx.waitUntil(Promise.resolve().then(() =&gt;
      console.log(JSON.stringify({ release: env.RELEASE, status: response.status, ms: Date.now() - started }))
    ));
    return response;
  },
} satisfies ExportedHandler&lt;Env&gt;;</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-typescript">export interface Env {
  UPSTREAM: Fetcher;
  RELEASE: string;
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise&lt;Response&gt; {
    const started = Date.now();
    const response = await env.UPSTREAM.fetch(request);
    ctx.waitUntil(Promise.resolve().then(() =&gt;
      console.log(JSON.stringify({ release: env.RELEASE, status: response.status, ms: Date.now() - started }))
    ));
    return response;
  },
} satisfies ExportedHandler&lt;Env&gt;;</code></pre>
  </div>
</div>

L’exemple utilise un Service Binding plutôt qu’une URL librement configurable. L’accessibilité et l’autorisation relèvent ainsi de la configuration de déploiement, tandis que le code ne voit qu’une capacité `Fetcher`. Les API de runtime sont orientées Web et comprennent notamment Fetch, Streams, Web Crypto, WebSockets, HTMLRewriter et TCP Sockets ([Runtime APIs](https://developers.cloudflare.com/workers/runtime-apis/)).

Le code ne s’exécute pas dans un environnement Node local librement choisi, mais selon un contrat de runtime versionné. La Compatibility Date détermine les changements de comportement applicables à un déploiement.

## Compatibility Date et contrat de runtime

Une Compatibility Date n’est ni une note de version ni une indication de l’« état actuel » de l’article. Elle fait partie du contrat de runtime d’un déploiement Worker concret. Les changements susceptibles de rompre le comportement existant sont activés par date ou par flag. Une date plus ancienne fixe un comportement compatible ; une date plus récente doit être testée et déployée comme une mise à jour de dépendance ([Compatibility flags](https://developers.cloudflare.com/workers/configuration/compatibility-flags/)).

Les administrateurs traitent donc ces champs ensemble :

- commit de code et artefact bundlé ;
- Compatibility Date et flags explicites ;
- chaîne d’outils Wrangler et de build ;
- définitions de bindings, routes et placement ;
- ID de version et poids de déploiement ;
- migrations de données ou de Durable Objects.

La compatibilité Node.js est opt-in et n’implémente qu’une partie documentée des API Node. Certains modules sont complets, d’autres partiels ou seulement disponibles comme stub d’importation. `nodejs_compat` ne doit donc pas être assimilé à une « runtime Node complète » ([Node.js compatibility](https://developers.cloudflare.com/workers/runtime-apis/nodejs/)).

## Langages et pile technologique

Cloudflare documente JavaScript, TypeScript, Python et Rust comme parcours de langage de premier plan ; WebAssembly ouvre l’accès à d’autres langages sources ([Languages](https://developers.cloudflare.com/workers/languages/)). La pile technologique résultante distingue développement et exécution :

| Couche | Technologie typique | Pertinence opérationnelle |
|---|---|---|
| Code source | TypeScript/JavaScript, Python, Rust | Chaîne d’outils du langage, dépendances, tests |
| Build | Wrangler/esbuild ou adaptateur de framework | Bundle, Source Maps, externals, builds déterministes |
| Runtime | workerd, V8, API Web et Node facultatives | Compatibility Date, CPU/mémoire, Event Loop |
| Capacité | bindings `env` | Moindre privilège, ID de ressource, séparation des environnements |
| État | Cache, KV, D1, R2, Durable Objects, Queues | Cohérence, emplacement des données, sauvegarde, reprise |
| Ingress | Route, Custom Domain, workers.dev, trigger | DNS, TLS, ordre, Fail-open/closed |
| Control Plane | Wrangler, API, Dashboard, CI/CD | Authentification, versionnage, déploiement progressif, audit |

Rust est intégré via `workers-rs` et WebAssembly ; le module Wasm importé reste une partie de l’artefact Worker ([Rust language support](https://developers.cloudflare.com/workers/languages/rust/)). Les Python Workers utilisent leurs propres classes de points d’entrée et une intégration de runtime, et non un processus CPython librement administrable.

## Configuration et Wrangler

La configuration Wrangler est le lien déclaratif entre le code et la plateforme. Elle définit au minimum le nom, le point d’entrée, la Compatibility Date, les routes et les bindings. Les Named Environments peuvent posséder des valeurs différentes ; les bindings doivent être vérifiés consciemment pour chaque environnement. Cloudflare documente les champs et l’héritage dans le schéma Wrangler ([Workers configuration](https://developers.cloudflare.com/workers/configuration/), [Wrangler configuration reference](https://developers.cloudflare.com/workers/wrangler/configuration/)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Workers-Konfigurations- und Typprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">npx wrangler whoami
npx wrangler types
npx wrangler deploy --dry-run
npx wrangler deployments status</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">npx wrangler whoami
npx wrangler types
npx wrangler deploy --dry-run
npx wrangler deployments status</code></pre>
  </div>
</div>

[`npx`](https://docs.npmjs.com/cli/commands/npx) lance la CLI figée localement au projet ; les [`Wrangler Commands`](https://developers.cloudflare.com/workers/wrangler/commands/) documentent l’authentification, la génération de types, le déploiement, les commandes de version et de logs. `wrangler types` génère des types à partir de la configuration réelle des bindings. `deploy --dry-run` vérifie le bundle et la configuration, mais ne prouve pas le fonctionnement des ressources distantes ni la correction des autorisations.

Une fois la runtime et le build clarifiés, la question de l’accès se pose. Un Worker accède aux bases de données, aux files d’attente, aux secrets et à d’autres Workers via des bindings déclarés plutôt que via des informations d’accès distribuées librement.

## Bindings comme modèle de capacité

Un binding est à la fois une autorisation et une API de runtime. Le Worker reçoit par exemple `env.ARCHIVE` comme bucket R2, `env.DB` comme base de données D1 ou `env.AUTH` comme service interne. Le credential de plateforme sous-jacent n’est pas révélé au code d’application sous forme de clé API réutilisable ([Bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/)).

Le modèle de capacité déplace la question centrale d’administration de « Quel secret se trouve dans la variable d’environnement ? » vers « Quelle ressource est liée, sous quel nom, dans quel environnement et à quelle version ? ». Un mauvais ID de bucket ou une mauvaise version de Service Binding peut provoquer une perte de données ou un accès inter-environnements avec un code identique.

Les bindings doivent être séparés selon la tâche et l’environnement. Les Workers de production et de test ne partagent pas de buckets, files d’attente ou bases de données inscriptibles, sauf si cela fait explicitement partie d’un test d’intégration contrôlé. Des noms génériques tels que `DB` suffisent dans le code lorsque le déploiement lie la ressource réelle de manière univoque et vérifiable.

### Secrets et variables

Les variables ordinaires sont de la configuration, pas des secrets. Les secrets sont des bindings texte chiffrés et sont gérés hors du code source. Les noms de secrets requis peuvent être déclarés sans écrire leurs valeurs dans le fichier de configuration ([Environment variables](https://developers.cloudflare.com/workers/configuration/environment-variables/), [Secrets](https://developers.cloudflare.com/workers/configuration/secrets/)).

Un changement de secret est un changement d’état de l’application. Les clients dérivés globalement de `env` peuvent conserver une ancienne valeur dans des isolats réutilisés ; Cloudflare recommande de créer de telles dérivations par requête. La rotation comprend donc la définition, la vérification du déploiement et du binding, une validité parallèle si le système cible l’exige, la télémétrie et la suppression contrôlée.

## Service Bindings et architecture Worker-à-Worker

Les Service Bindings relient explicitement des Workers, sans nécessiter de point de terminaison HTTP publiquement adressable. Le service appelé peut être exposé via Fetch ou RPC. Cela réduit la configuration réseau et des credentials, mais n’élimine pas les problèmes de versions et de contrats. Lors de déploiements séparés, l’appelant et l’appelé peuvent utiliser des états d’API différents ([Service bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/service-bindings/), [Gradual deployments](https://developers.cloudflare.com/workers/versions-and-deployments/gradual-deployments/)).

Les administrateurs documentent pour chaque binding :

- appelant et service cible ;
- environnement et version cible ou stratégie de déploiement progressif ;
- accessibilité publique du Worker cible ;
- sémantique des délais d’attente, erreurs et retries ;
- transmission de l’ID de requête et corrélation de traces ;
- compatibilité des contrats lors de déploiements indépendants.

Une chaîne de nombreux Workers répartit une requête sur davantage de points de défaillance possibles. Réduire la surface d’attaque publique est précieux ; une granularité trop fine peut toutefois augmenter la profondeur des sous-requêtes, l’effort de débogage et le décalage de versions.

## Services d’état : le modèle avant le nom du produit

Le heap de l’isolat est volatil. L’état persistant réside dans des services liés, dont la sémantique diffère fortement. La vue d’ensemble du stockage Cloudflare classe les produits selon le modèle d’accès et la cohérence ([Choosing a data or storage product](https://developers.cloudflare.com/workers/platform/storage-options/)).

| Service | Modèle | Frontière de cohérence/emplacement | Adapté à | Question d’administration critique |
|---|---|---|---|---|
| Cache API | Cache de réponses HTTP | local au centre de calcul, éphémère | réponses répétables | La clé de cache est-elle complète et sûre ? |
| Workers KV | Store clé-valeur lisible globalement | eventual consistent, Read Cache | configuration, données à forte lecture | Le workflow tolère-t-il des lectures anciennes ou négatives ? |
| D1 | SQL géré basé sur SQLite | sémantique liée à la base et à la session | tables d’application relationnelles | Quelle garantie de réplica/session est nécessaire pour la lecture ? |
| Durable Objects | Instance d’objet unique avec stockage | fortement cohérent et sérialisable par objet | coordination, locks, sessions, WebSockets | La clé de partition est-elle correctement choisie ? |
| R2 | Stockage d’objets compatible S3 | fortement cohérent par objet | blobs, archives, payloads volumineux | Comment versionner ensemble objet, métadonnées et index ? |
| Queues | Messages asynchrones | traitement orienté au moins une fois et politique de retry | découplage, lissage de charge | Le consommateur est-il idempotent et existe-t-il une DLQ ? |

### Cache API

La Cache API fonctionne localement dans le centre de calcul où le Worker traite la requête. `cache.put()` n’est donc ni une réplication globale ni une écriture persistante en base de données. `caches.default` partage le contexte de cache standard ; les caches nommés créent des espaces de noms, mais aucune cohérence globale ([How the Cache works](https://developers.cloudflare.com/workers/reference/how-the-cache-works/)).

Les réponses personnalisées ne doivent être mises en cache qu’avec une clé représentant tous les attributs pertinents d’identité et de variante. `Authorization`, les cookies, la langue, l’encodage et l’appartenance au tenant sont des frontières de fuite typiques. La purge de cache et la validation de l’origin font partie de la reprise, pas seulement les TTL.

### Workers KV

KV réplique les données entre des stores centraux et des caches Edge. Les changements peuvent devenir visibles avec un décalage dans d’autres emplacements ; même « non trouvé » peut être mis en cache. Cloudflare indique des retards pouvant atteindre 60 secondes ou plus pour les emplacements distants et déconseille KV pour les transactions atomiques lecture-modification-écriture ([How KV works](https://developers.cloudflare.com/kv/concepts/how-kv-works/)).

KV convient à une configuration principalement lue, à des listes d’autorisation/refus ou à des caches lorsque le caractère obsolète est explicitement toléré. Les limites de débit globales, les séquences uniques ou la révocation immédiate de credentials ne doivent pas être placées dans KV sans coordination supplémentaire.

### Durable Objects

Un Durable Object relie un ID d’objet mondialement unique à une coordination sérielle et à un stockage privé. Les requêtes pour le même ID sont routées vers la même instance logique. Les nouveaux espaces de noms utilisent le chemin de stockage SQLite ; celui-ci offre un stockage transactionnel fortement cohérent et des fonctions de reprise à un instant donné ([Durable Objects overview](https://developers.cloudflare.com/durable-objects/), [SQLite-backed Durable Object Storage](https://developers.cloudflare.com/durable-objects/api/sqlite-storage-api/)).

Même un Durable Object n’est pas un serveur fonctionnant en permanence. L’état en mémoire peut disparaître lors d’une hibernation ou d’une éviction et doit, si nécessaire, être reconstruit depuis le stockage. Il n’existe pas de hook d’arrêt fiable ([Durable Object lifecycle](https://developers.cloudflare.com/durable-objects/concepts/durable-object-lifecycle/)). Les modifications aux classes et aux affectations de stockage sont des opérations de cycle de vie ; elles sont traitées comme des migrations de données ([Durable Object class lifecycle](https://developers.cloudflare.com/durable-objects/reference/durable-objects-migrations/)).

### D1, R2 et Queues

D1 fournit SQL pour les données relationnelles. R2 stocke de grands objets non structurés et expose une API compatible S3 à l’extérieur ou une API de binding à l’intérieur des Workers. Les Queues découplent producteur et consommateur. Ces services ne se remplacent pas : un objet R2 peut porter le payload, D1 l’index, une Queue déclencher le traitement et un Durable Object coordonner des modifications concurrentes.

Le commit commun de ces quatre états n’est pas automatiquement atomique. Les administrateurs planifient des clés d’idempotence, des modèles Outbox/Inbox, la réconciliation et le traitement Dead Letter. La documentation produit et les limites de chaque service pour [D1](https://developers.cloudflare.com/d1/), [R2](https://developers.cloudflare.com/r2/) et [Queues](https://developers.cloudflare.com/queues/) font partie du runbook.

## Assets statiques et Workers full-stack

Un Worker peut distribuer des assets statiques conjointement avec le code. Le binding Assets permet au code de récupérer explicitement un asset. L’option de routage détermine si les assets existants contournent le Worker ou si le Worker s’exécute d’abord ([Static Assets](https://developers.cloudflare.com/workers/static-assets/)).

L’ordre est une décision de sécurité. Avec Assets-first, un chemin de fichier présent par hasard ne doit pas contourner l’authentification. Avec Worker-first, chaque requête d’asset augmente la charge de calcul et de dépendances. Les adaptateurs de framework masquent partiellement cet ordre ; l’artefact de déploiement construit et la configuration Wrangler restent la vérité de référence.

## Vue réseau et protocole

Workers se situe dans le chemin applicatif au-dessus de DNS, TLS et HTTP. La plateforme termine le transport externe ; le Worker traite des requêtes d’API Web. Un `fetch()` sortant crée un nouvel appel HTTP. Les TCP Sockets permettent certains protocoles sortants, mais ne transforment pas le Worker en serveur TCP généralement accessible ([Workers protocols](https://developers.cloudflare.com/workers/reference/protocols/), [TCP sockets](https://developers.cloudflare.com/workers/runtime-apis/tcp-sockets/)).

Pour les administrateurs de messagerie, trois frontières sont importantes :

- Un gestionnaire d’e-mail est déclenché par Cloudflare Email Routing ; il n’écoute pas lui-même sur [SMTP](/kb/smtp).
- Un Worker qui appelle une API de messagerie doit traiter l’audience OAuth, le cycle de vie des tokens et les retries comme tout autre client API.
- Les erreurs DNS, TLS et HTTP se produisent avant le point d’entrée ; les erreurs de binding, d’authentification et d’application après. Un timeout `fetch()` ne permet pas de savoir sans corrélation quelle phase a échoué.

## Frontières de sécurité et de confiance

Le modèle de sécurité comporte au moins cinq frontières de confiance distinctes :

1. **Requête publique :** en-têtes, taille de body, méthode et identité arbitraires.
2. **Configuration Cloudflare :** zone, route, WAF, Access, certificats et Fail-open/closed.
3. **Artefact Worker :** code, dépendances, Source Maps, Compatibility Date et chaîne d’approvisionnement.
4. **Bindings :** autorisations de ressources concrètes, secrets et chemins de service internes.
5. **Upstreams et données stockées :** autorisation, cohérence et reprise propres.

Le moindre privilège est atteint principalement grâce à de petites surfaces de bindings et à des environnements séparés. Un Worker ayant un accès en écriture à tous les buckets et bases de données reste un principal hautement privilégié, même s’il ne possède pas de clés d’accès visibles. La documentation de sécurité Workers explique l’isolation de runtime ; elle ne remplace pas une architecture de sécurité applicative ([Security model](https://developers.cloudflare.com/workers/reference/security-model/)).

Les contrôles de chaîne d’approvisionnement comprennent des versions de paquets fixées, un lockfile, un build reproductible, l’analyse des dépendances, le secret scanning, des autorisations de téléversement minimales et des environnements de déploiement protégés. Le bundle généré est vérifié avant le téléversement ; la revue du code source seule ne suffit pas lorsque des bundlers ou frameworks incorporent des modules supplémentaires.

La proximité Edge ne supprime pas les limites de ressources. Le temps CPU, les sous-requêtes, la mémoire et les limites de plateforme doivent être pris en compte dès la conception et le modèle de charge.

## Les limites comme paramètres d’architecture

Workers limite notamment le temps CPU, la mémoire, le temps de démarrage, les sous-requêtes, la taille du bundle, le volume de logs, le nombre de routes et les assets statiques. Les valeurs diffèrent selon le plan et le type d’invocation et peuvent changer. Un article statique ne doit donc pas contenir une table de limites copiée présentée comme une vérité intemporelle ; la page officielle [Workers Limits](https://developers.cloudflare.com/workers/platform/limits/) est la source opérationnelle.

Les administrateurs distinguent :

- **CPU Time :** temps de calcul actif ; l’attente d’E/S ne compte pas comme le CPU.
- **Wall Time :** durée écoulée de l’événement ; les règles diffèrent entre Fetch, Cron, Queue et Durable Object.
- **Startup Time :** évaluation des modules et initialisation avant le gestionnaire.
- **Subrequests :** opérations Fetch et de plateforme par invocation.
- **Memory :** heap, streams, buffers et état des bibliothèques de l’isolat.
- **Log Budget :** les payloads trop grands ou secrets sont à la fois un problème de coûts et de protection des données.

Une erreur `1102` indique des limites de ressources dépassées ; `10021` peut survenir lors d’une initialisation de démarrage trop coûteuse. Le numéro d’erreur est un point de départ, pas une cause racine. Le profil CPU, le type d’invocation, la version, la classe d’entrée et le graphe des sous-requêtes doivent être considérés ensemble.

## Développement local et tests

Le développement local exécute le code avec `workerd` via Miniflare. Les bindings sont simulés localement par défaut ; certains Remote Bindings peuvent être reliés de façon ciblée à de vraies ressources de plateforme. L’exécution entièrement distante de `wrangler dev --remote` reste disponible pour les cas spécifiques au réseau, mais n’est plus le chemin standard ([Local development](https://developers.cloudflare.com/workers/local-development/), [Bindings per development mode](https://developers.cloudflare.com/workers/local-development/bindings-per-env/)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für lokale Workers-Tests">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">npx wrangler dev --local
$response = Invoke-WebRequest 'http://127.0.0.1:8787/health'
$response.StatusCode
$response.Headers['content-type']
npx vitest run</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">npx wrangler dev --local
curl --silent --show-error --fail --dump-header - http://127.0.0.1:8787/health
npx vitest run</code></pre>
  </div>
</div>

[`Invoke-WebRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest) et [`curl`](https://curl.se/docs/manpage.html) vérifient le statut et les en-têtes du chemin HTTP local. L’intégration Workers-Vitest exécute les tests au sein de `workerd` et fournit des assistants pour les bindings, les requêtes et les Durable Objects ([Vitest integration](https://developers.cloudflare.com/workers/testing/vitest-integration/), [Workers test APIs](https://developers.cloudflare.com/workers/testing/vitest-integration/test-apis/)).

Un test unitaire vert ne prouve ni une route correcte, ni une autorisation distante, ni une migration de données correcte. La pyramide de tests comprend la logique pure, l’intégration de runtime, la sémantique des bindings, la route de staging, un smoke test de production contrôlé et un exercice de reprise.

## Déploiement, version et rollout

Une **version** est un objet immuable de code/configuration. Un **déploiement** distribue le trafic sur une ou plusieurs versions. Les Gradual Deployments déplacent les poids progressivement et permettent l’observation ainsi que le retour à une version stable ([Versions and deployments](https://developers.cloudflare.com/workers/versions-and-deployments/), [Gradual deployments](https://developers.cloudflare.com/workers/versions-and-deployments/gradual-deployments/)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Workers-Versionen und Logs">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">npx wrangler versions list
npx wrangler deployments status
npx wrangler tail --format json
npx wrangler check startup</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">npx wrangler versions list
npx wrangler deployments status
npx wrangler tail --format json
npx wrangler check startup</code></pre>
  </div>
</div>

Avant un rollout, l’association `Git commit → Bundlehash → Worker-Version → Deploymentgewicht` est enregistrée. Avec plusieurs Service Bindings, le décalage de versions fait partie des tests. Les Durable Objects nécessitent des règles de migration et de rollout particulières, car des états de classes arbitraires ne peuvent pas être simultanément actifs pour un ID d’objet.

Un rollback restaure le code et la configuration, mais pas automatiquement les données externes. Un nouveau Worker peut avoir écrit des données dans un format que l’ancien ne comprend pas. Les changements de schéma utilisent Expand/Contract, la compatibilité ascendante ou une restauration des données testée séparément. « Rollback possible » n’est prouvé que lorsque les aspects code, binding et données ont été vérifiés ensemble.

## Observabilité et preuve opérationnelle

Workers Logs collecte les logs d’invocation, les logs propres, les erreurs et les exceptions non gérées. Les Tail Workers ou Logpush peuvent exporter des événements ; les cibles OpenTelemetry proposent un chemin d’export plus direct pour les logs et traces ([Workers Logs](https://developers.cloudflare.com/workers/observability/logs/workers-logs/), [Tail Workers](https://developers.cloudflare.com/workers/observability/logs/tail-workers/), [Traces](https://developers.cloudflare.com/workers/observability/traces/)).

Chaque invocation devrait permettre au minimum une corrélation de :

- nom du Worker, ID de version et environnement ;
- type d’événement et route ;
- ID de requête/message/corrélation ;
- résultat, statut HTTP ou Ack/Retry ;
- CPU et Wall Time ;
- cible et latence des sous-requêtes sans secrets ;
- classe de binding/ressource, non le contenu complet sensible ;
- clé d’idempotence métier pour le traitement asynchrone.

`console.log()` n’est pas un dépôt de preuves illimité. Les limites de logs, l’échantillonnage et la rédaction influencent la visibilité. Les credentials, en-têtes Authorization, cookies, contenus complets d’e-mails et payloads à caractère personnel ne sont pas journalisés. Le monitoring surveille également le chemin d’export : un Tail Worker défaillant ne doit pas empêcher le diagnostic d’une erreur du producteur.

Le diagnostic suit le chemin de publication : route et DNS, version active, runtime et bindings, dépendances sortantes, puis enfin logs et traces.

## Diagnostic d’administration par phases

Les erreurs Workers sont efficacement séparées le long du chemin d’exécution :

1. **DNS :** l’hôte attendu résout-il vers Cloudflare et la zone est-elle active ?
2. **TLS/HTTP :** vérifier le certificat, SNI, le protocole et le statut.
3. **Route/Asset :** l’hôte/chemin atteint-il le Worker, un asset ou l’origin ?
4. **Version :** quelle version et quel poids de déploiement ont traité la requête ?
5. **Runtime :** vérifier le démarrage, CPU, mémoire, exception et sortie du gestionnaire.
6. **Binding :** la ressource existe-t-elle dans cet environnement et possède-t-elle la capacité attendue ?
7. **Upstream/Storage :** examiner timeout, authentification, cohérence, état des données et retry.
8. **Recovery :** confirmer la version stable, la compatibilité des données et la réconciliation.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für DNS-, TLS- und Workers-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName $env:WORKER_HOST
Test-NetConnection $env:WORKER_HOST -Port 443
$r = Invoke-WebRequest "https://$env:WORKER_HOST/health" -Headers @{ 'x-correlation-id' = [guid]::NewGuid() }
$r.StatusCode
$r.Headers
npx wrangler deployments status</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig +short "$WORKER_HOST"
nc -vz "$WORKER_HOST" 443
openssl s_client -connect "$WORKER_HOST:443" -servername "$WORKER_HOST" -brief &lt;/dev/null
curl --silent --show-error --fail --dump-header - "https://$WORKER_HOST/health"
npx wrangler deployments status</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname), [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection), [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility), [`nc`](https://man.openbsd.org/nc), [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) et [`curl`](https://curl.se/docs/manpage.html) distinguent DNS, TCP, TLS et HTTP avant la runtime. [DNS](/kb/dns), [TCP](/kb/tcp), [TLS](/kb/tls), [APIs](/kb/apis) et [Troubleshooting](/kb/troubleshooting) traitent ces couches en profondeur.

### Symptômes typiques d’erreur

| Symptôme | Phase probable | Preuve | Erreur de raisonnement fréquente |
|---|---|---|---|
| Réponse d’origin au lieu de réponse Worker | Route/Asset/Fail-open | Route, hôte, chemin, marqueur de réponse | « Déploiement réussi » signifie « route active » |
| 1101 ou exception | Application/binding | Workers Logs, version, stack | redéployer seulement au lieu de vérifier la classe d’entrée |
| 1102 | Limite de ressources | CPU/Wall Time, profil, type d’invocation | assimiler attente d’E/S et CPU |
| ancienne valeur KV sporadique | Cohérence KV | clé, emplacement, TTL de cache, heure d’écriture | traiter KV comme une transaction globale |
| vert en local, erreur de binding à distance | Environnement/capacité | ID de ressource concret et nom de binding | la simulation prouve l’état IAM/de ressource |
| seul une partie du trafic échoue | Gradual Deployment | ID de version et poids | agréger les métriques sans dimension de version |
| le rollback corrige le code, pas les données | Frontière schéma/état | format écrit, migration, réconciliation | le déploiement serait une reprise complète |

## Sauvegarde et Disaster Recovery

Le serverless n’élimine pas la sauvegarde. Il déplace les objets à protéger :

- dépôt Git, lockfile et définition de build reproductible ;
- configuration Wrangler, routes, Compatibility Date et flags ;
- inventaire de tous les bindings et ressources cibles ;
- valeurs de secrets ou leur source externe de vérité et processus de rotation ;
- exports de données et procédures de restauration pour D1, R2, KV et les systèmes externes ;
- classes Durable Object, état du cycle de vie/des migrations et, le cas échéant, PITR ;
- état Queue/DLQ, clés d’idempotence et réconciliation ;
- historique des versions/déploiements ainsi qu’un chemin de rollback testé.

Tous les produits n’ont pas la même sémantique d’export et de restauration. Une sauvegarde d’objet R2 ne protège pas l’index D1 ; une restauration D1 ne rétablit pas un message Queue déjà confirmé ; un Worker revenu à une version antérieure peut mal interpréter de nouveaux formats d’objet. [Backup et Disaster Recovery](/kb/backup-dr) doit donc définir un point applicatif cohérent au lieu de simples copies de produits individuels.

La reprise est testée par scénarios : mauvais secret, route supprimée, version défectueuse, schéma D1 incompatible, objet R2 perdu, message Queue poison, classe Durable Object défectueuse et upstream inaccessible. Les RTO et RPO sont définis pour chaque service d’état et pour l’application composite.

## Histoire technique

Cloudflare a présenté Workers en 2017 comme une exécution programmable sur le réseau mondial. Le point de départ était la limite de latence des centres de calcul centralisés et l’idée d’exécuter du code près du chemin des données ([Code Everywhere: Why We Built Cloudflare Workers](https://blog.cloudflare.com/code-everywhere-cloudflare-workers/)).

L’architecture s’est appuyée très tôt sur les isolats V8 plutôt que sur un conteneur ou processus par fonction. En 2018, Cloudflare a décrit les différences économiques et techniques de ce modèle, mais aussi sa limitation aux langages capables de JavaScript ou WebAssembly ([Cloud Computing without Containers](https://blog.cloudflare.com/cloud-computing-without-containers/)).

Workers KV a ajouté un état lisible globalement, eventual-consistent. Les Durable Objects ont été annoncés en 2020 comme modèle opposé pour des états d’objet coordonnés et fortement cohérents ([Introducing Workers Durable Objects](https://blog.cloudflare.com/introducing-workers-durable-objects/)). Workers a ainsi évolué de la pure transformation de requêtes vers une plateforme applicative dotée de plusieurs modèles de cohérence explicites.

En 2022, Cloudflare a publié `workerd` sous Apache 2.0. La runtime ouverte partage du code avec le système de production et a amélioré la précision du développement local ; elle ne constitue toutefois pas l’ensemble de la plateforme Cloudflare avec son routage, son orchestration et son exploitation de sécurité ([Introducing workerd](https://blog.cloudflare.com/workerd-open-source-workers-runtime/), [workerd repository](https://github.com/cloudflare/workerd)).

Des extensions ultérieures ont apporté les points d’entrée de modules ES, les Compatibility Dates, les Service/RPC Bindings, D1, R2, Queues, Workflows, Python, une compatibilité Node étendue, les objets de version et les déploiements progressifs. Cette histoire explique le cœur actuel : Workers n’est ni du JavaScript de navigateur sur un CDN ni un serveur Linux quelconque, mais une runtime événementielle fondée sur les capacités, avec des produits de données et de Control Plane séparés.

## Liste de contrôle d’administration

Avant la mise en production, les points suivants sont au minimum établis :

- La route, le Custom Domain, l’ordre des assets et Fail-open/closed sont documentés.
- Compatibility Date, flags, version de Wrangler, bundle et Source Maps sont reproductibles.
- Chaque binding possède un propriétaire, un environnement, un ID de ressource, des droits et un chemin de reprise.
- Les sémantiques de Cache, KV, D1, R2, Durable Object et Queue ne sont pas confondues.
- Les secrets sont hors du dépôt, peuvent être tournés et ne sont pas visibles dans les logs.
- Les timeouts, retries, l’idempotence et le comportement Dead Letter sont définis par type d’événement.
- Les logs et traces contiennent la version et l’ID de corrélation, mais aucun payload sensible.
- Gradual Deployment et rollback tiennent compte des versions de Service Bindings et des formats de données.
- Les limites sont surveillées depuis la page officielle de la plateforme, pas depuis une table copiée.
- La sauvegarde, la restauration et la réconciliation ont été testées pour l’application composite.

## Sources

- [How Workers works](https://developers.cloudflare.com/workers/reference/how-workers-works/)
- [Cloudflare Workers documentation](https://developers.cloudflare.com/workers/)
- [Wrangler configuration reference](https://developers.cloudflare.com/workers/wrangler/configuration/)
- [Static Assets – configuration and bindings](https://developers.cloudflare.com/workers/static-assets/binding/)
- [Service bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/service-bindings/)
- [Placement](https://developers.cloudflare.com/workers/configuration/placement/)
- [Introducing workerd](https://blog.cloudflare.com/workerd-open-source-workers-runtime/)
- [Workers security model](https://developers.cloudflare.com/workers/reference/security-model/)
- [Handlers](https://developers.cloudflare.com/workers/runtime-apis/handlers/)
- [Fetch handler](https://developers.cloudflare.com/workers/runtime-apis/handlers/fetch/)
- [Context – waitUntil](https://developers.cloudflare.com/workers/runtime-apis/context/)
- [Runtime APIs](https://developers.cloudflare.com/workers/runtime-apis/)
- [Compatibility flags](https://developers.cloudflare.com/workers/configuration/compatibility-flags/)
- [Node.js compatibility](https://developers.cloudflare.com/workers/runtime-apis/nodejs/)
- [Languages](https://developers.cloudflare.com/workers/languages/)
- [Rust language support](https://developers.cloudflare.com/workers/languages/rust/)
- [Workers configuration](https://developers.cloudflare.com/workers/configuration/)
- [npx documentation](https://docs.npmjs.com/cli/commands/npx)
- [Wrangler Commands](https://developers.cloudflare.com/workers/wrangler/commands/)
- [Bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/)
- [Environment variables](https://developers.cloudflare.com/workers/configuration/environment-variables/)
- [Secrets](https://developers.cloudflare.com/workers/configuration/secrets/)
- [Gradual deployments](https://developers.cloudflare.com/workers/versions-and-deployments/gradual-deployments/)
- [Choosing a data or storage product](https://developers.cloudflare.com/workers/platform/storage-options/)
- [How the Cache works](https://developers.cloudflare.com/workers/reference/how-the-cache-works/)
- [How KV works](https://developers.cloudflare.com/kv/concepts/how-kv-works/)
- [Durable Objects overview](https://developers.cloudflare.com/durable-objects/)
- [SQLite-backed Durable Object Storage](https://developers.cloudflare.com/durable-objects/api/sqlite-storage-api/)
- [Durable Object lifecycle](https://developers.cloudflare.com/durable-objects/concepts/durable-object-lifecycle/)
- [Durable Object class lifecycle](https://developers.cloudflare.com/durable-objects/reference/durable-objects-migrations/)
- [D1](https://developers.cloudflare.com/d1/)
- [R2](https://developers.cloudflare.com/r2/)
- [Queues](https://developers.cloudflare.com/queues/)
- [Static Assets](https://developers.cloudflare.com/workers/static-assets/)
- [Workers protocols](https://developers.cloudflare.com/workers/reference/protocols/)
- [TCP sockets](https://developers.cloudflare.com/workers/runtime-apis/tcp-sockets/)
- [Workers Limits](https://developers.cloudflare.com/workers/platform/limits/)
- [Local development](https://developers.cloudflare.com/workers/local-development/)
- [Bindings per development mode](https://developers.cloudflare.com/workers/local-development/bindings-per-env/)
- [Invoke-WebRequest](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest)
- [curl manual](https://curl.se/docs/manpage.html)
- [Vitest integration](https://developers.cloudflare.com/workers/testing/vitest-integration/)
- [Workers test APIs](https://developers.cloudflare.com/workers/testing/vitest-integration/test-apis/)
- [Versions and deployments](https://developers.cloudflare.com/workers/versions-and-deployments/)
- [Workers Logs](https://developers.cloudflare.com/workers/observability/logs/workers-logs/)
- [Tail Workers](https://developers.cloudflare.com/workers/observability/logs/tail-workers/)
- [Traces](https://developers.cloudflare.com/workers/observability/traces/)
- [Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [dig manual](https://bind9.readthedocs.io/en/latest/manpages.html)
- [nc(1)](https://man.openbsd.org/nc)
- [openssl-s_client(1)](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Code Everywhere: Why We Built Cloudflare Workers](https://blog.cloudflare.com/code-everywhere-cloudflare-workers/)
- [Cloud Computing without Containers](https://blog.cloudflare.com/cloud-computing-without-containers/)
- [Introducing Workers Durable Objects](https://blog.cloudflare.com/introducing-workers-durable-objects/)
- [workerd repository](https://github.com/cloudflare/workerd)
