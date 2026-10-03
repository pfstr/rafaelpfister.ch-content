---
title: "API : contrats, identités et limites d’erreur distribuées"
blatt: "apis"
description: "API pour les administrateurs d’infrastructure et de messagerie : REST, HTTP et JSON, RPC, GraphQL, gRPC et API événementielles, OpenAPI et schémas, OAuth et identités de charge de travail, passerelles, délais d’attente, tentatives, idempotence, pagination, limites de débit, webhooks, observabilité, versionnage et histoire technique."
fakten:
  - label: Rôle système
    wert: interface de gestion ou de données lisible par machine entre composants séparés
    href: https://ics.uci.edu/~fielding/pubs/dissertation/net_app_arch.htm
  - label: REST
    wert: style architectural avec contraintes ; ne signifie pas HTTP plus JSON
    href: https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm
  - label: Modèle HTTP
    wert: ressource/URI · méthode · champs · représentation · statut
    href: https://www.rfc-editor.org/rfc/rfc9110.html
  - label: Contrat
    wert: décrire les opérations, schémas, erreurs, sécurité et cycle de vie de manière lisible par machine
    href: https://spec.openapis.org/oas/
  - label: Formats de données
    wert: JSON · XML · Protocol Buffers · formats binaires et de streaming
    href: https://www.rfc-editor.org/rfc/rfc8259.html
  - label: Styles d’interaction
    wert: requête/réponse · RPC · requête · flux · événement/webhook
    href: https://ics.uci.edu/~fielding/pubs/dissertation/net_arch_styles.htm
  - label: Identité
    wert: API-Key, certificat client ou jeton ; credential et autorisation sont distincts
    href: https://www.rfc-editor.org/rfc/rfc9700.html
  - label: Répétition
    wert: uniquement avec une sémantique connue ; un timeout ne signifie pas que rien ne s’est produit côté serveur
    href: https://www.rfc-editor.org/rfc/rfc9110.html
  - label: Concurrence
    wert: ETag/If-Match ou numéro de version métier évite les Lost Updates
    href: https://www.rfc-editor.org/rfc/rfc9110.html
  - label: Format d’erreur
    wert: statut HTTP plus type de problème stable lisible par machine et Request-ID
    href: https://www.rfc-editor.org/rfc/rfc9457.html
  - label: État opérationnel
    wert: latence · taux d’erreur · saturation · quota · expiration du jeton · Queue/Consumer Lag
    href: https://opentelemetry.io/docs/specs/otel/trace/
  - label: Preuve pour l’administrateur
    wert: client · identité · scope · endpoint · version du contrat · Request-ID · résultat
    href: https://www.rfc-editor.org/rfc/rfc9110.html
werbung:
  - newsletter
ctaThemen:
  - cloudflare-workers
  - powershell
  - automatisierung
translationSourceHash: 4b29095b24ea1576608e147b1904a65a72c2b09b2e72f991e3e27a64dbcb3332
translationModel: gpt-5.6-terra
translatedAt: 2026-10-03T09:28:28.791Z
translationReview: required
---

# API : contrats, identités et limites d’erreur distribuées

Une interface de programmation applicative est une **limite de contrat et de confiance** entre des composants exploités indépendamment. Le contrat définit les opérations et les données qui existent ; l’exécution détermine le transport, l’identité, l’autorisation, le comportement temporel et les erreurs. Pour les administrateurs, cette séparation est centrale : une requête JSON syntaxiquement valide peut aboutir au mauvais tenant, échouer avec un jeton valide auprès de la mauvaise audience, ou avoir néanmoins été exécutée avec succès côté serveur après un timeout client.

Les API sont omniprésentes dans les environnements de messagerie : entre le client d’administration et la plateforme de messagerie, la passerelle et l’annuaire, la supervision et le backend de télémétrie, l’application cloud et le destinataire du webhook. Une interface graphique peut utiliser la même interface, mais elle ne représente généralement qu’une partie de ses états et de ses erreurs. Une exploitation robuste ne commence donc pas par une commande curl isolée, mais par le **style d’interface, le contrat, le modèle de ressources, l’identité, les transitions d’état et la sémantique de récupération**.

L’explication suit un appel API du client jusqu’à la réponse métier. Elle aborde d’abord le transport et le contrat, puis l’identité, la gestion des erreurs et les événements ; viennent ensuite l’exploitation de la passerelle, la sécurité et le diagnostic.

## Classification en tant que système distribué

Une API réseau n’est pas seulement du code applicatif. Un appel typique parcourt :

1. une bibliothèque cliente, une CLI ou un processus d’automatisation ;
2. la résolution de noms, le routage et l’établissement de connexion ;
3. [TLS](/kb/tls), proxy ou Service Mesh ;
4. un Load Balancer, une API Gateway ou un Reverse Proxy ;
5. l’authentification, la validation du jeton et l’autorisation ;
6. le service applicatif, le cache, la file d’attente et la base de données ;
7. le chemin de réponse, la sérialisation et l’évaluation côté client.

Le travail architectural de Roy Fielding distingue explicitement les systèmes fondés sur le réseau de l’exécution locale transparente : la communication réseau possède ses propres latences, coûts et modes de défaillance. Un style architectural est un ensemble coordonné de contraintes qui fait émerger certaines propriétés et certains compromis ([Fielding – Network-based Application Architectures](https://ics.uci.edu/~fielding/pubs/dissertation/net_app_arch.htm), [Fielding – Network-based Architectural Styles](https://ics.uci.edu/~fielding/pubs/dissertation/net_arch_styles.htm)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-apis.svg?v=20260813" title="Interaktive Infografik: API-Aufruf über DNS, TLS, Gateway, Identität und Dienst sowie Vertrag, Fehler- und Wiederholungssemantik, Events und Betriebsnachweis" loading="lazy">
  <a href="/images/kb-interaktiv-apis.svg?v=20260813">Ouvrir directement le graphique interactif</a>.
</iframe>

La pile pertinente est documentée pour chaque intégration :

| Niveau | Exemples | Périmètre de défaillance typique |
|---|---|---|
| Identification | URI, Service Discovery, DNS | mauvais hôte, région, tenant ou chemin API |
| Transport | TCP/TLS, HTTP/1.1, HTTP/2, HTTP/3 | timeout, proxy, certificat, ALPN, Connection Pool |
| Interaction | REST, RPC, GraphQL, gRPC, webhook/event | mauvaise sémantique, retry inadapté, interruption de streaming |
| Représentation | JSON, XML, Protobuf, Multipart, données binaires | erreur de schéma, d’encodage, de taille ou de compatibilité |
| Contrat | OpenAPI, JSON Schema, Protobuf IDL, GraphQL SDL, AsyncAPI | Breaking Change, dérive entre documentation et exécution |
| Identité | API-Key, jeton OAuth, mTLS, requête signée | expiration, scope, audience, rotation de clés, Clock Skew |
| Politique | Gateway, WAF, RBAC/ABAC, quota | 401/403/429, normalisation des en-têtes, mauvais principal |
| État | service, cache, file d’attente, base de données | exécution partielle, retard de réplication, Eventual Consistency |
| Preuve | Request-ID, trace, journal d’audit, métriques | corrélation manquante, échantillonnage, protection des données |

## REST est un style architectural, pas un format de données

REST désigne les contraintes décrites par Fielding pour les systèmes hypermédias distribués : client/serveur, absence d’état, cache, interface uniforme, architecture en couches et Code-on-Demand facultatif. L’interface uniforme comprend l’identification des ressources, la manipulation via des représentations, les messages auto-descriptifs et l’hypermédia comme machine à états. La standardisation de l’interface améliore la visibilité et le développement indépendant, mais peut être moins efficace que des protocoles spécialisés ([Fielding – Representational State Transfer](https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)).

Une API HTTP avec JSON et des chemins tels que `/v1/getUser` n’est donc pas automatiquement REST. Elle peut simplement être du RPC sur HTTP. Ce n’est pas fondamentalement mauvais ; le problème survient lorsque les opérateurs attendent des propriétés que le style réel ne fournit pas. Par exemple, un client ne peut pas répéter un POST de manière sûre simplement parce que l’endpoint est appelé « REST ».

### URI, ressource et représentation

Une URI identifie une ressource ; elle ne garantit ni son accessibilité ni une opération donnée. La RFC 3986 sépare explicitement l’identification de l’interaction. Le schéma, l’autorité, le chemin, la requête et le fragment ont une syntaxe définie, tandis que l’API concrète définit la sémantique de ses ressources ([RFC 3986 – URI Generic Syntax](https://www.rfc-editor.org/rfc/rfc3986.html)).

Une ressource n’est pas son fichier JSON. La même ressource peut être représentée en JSON, XML ou dans un autre format selon l’en-tête `Accept`. `Content-Type` décrit le corps envoyé, `Accept` la réponse préférée. Le statut, les champs et le corps constituent ensemble le message ; ne journaliser que le corps JSON fait disparaître des informations de diagnostic importantes.

## Sémantique HTTP : la méthode avant le nom du chemin

La RFC 9110 sépare l’identification de la ressource de la sémantique de la requête. La méthode définit l’opération envisagée ; le seul chemin URI ne le fait pas. « Safe » signifie que le client ne vise aucune modification d’état. « Idempotent » signifie que plusieurs requêtes identiques ont le même effet intentionnel qu’une seule requête ; des effets secondaires tels que la journalisation peuvent néanmoins se produire plusieurs fois ([RFC 9110 – HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)).

| Méthode | Safe | Idempotent | Sémantique API typique | Retry sans information supplémentaire |
|---|---:|---:|---|---|
| GET | oui | oui | lire une représentation | généralement possible, mais tenir compte de la charge/des quotas |
| HEAD | oui | oui | métadonnées sans corps | généralement possible |
| OPTIONS | oui | oui | capacités/options de communication | généralement possible |
| PUT | non | oui | remplacer l’état sous une URI connue | possible si le contrat respecte réellement la sémantique PUT |
| DELETE | non | oui | supprimer une association | effet répétable ; le statut de réponse peut changer |
| POST | non | non | traiter, déclencher une action ou créer une nouvelle ressource | ne pas répéter aveuglément |
| PATCH | non | pas en général | modification partielle | uniquement avec une sémantique de patch et d’idempotence documentée |

L’idempotence décrit l’**effet voulu côté serveur**, et non la réponse de transport. Un PUT peut être terminé sur le serveur alors que la réponse est perdue. Un nouveau PUT est alors sémantiquement acceptable ; un nouveau POST peut créer un deuxième objet ou un second message. Pour les opérations non idempotentes, un identifiant d’opération généré par le client, une clé Idempotency-Key spécifique au fournisseur ou une consultation ultérieure du statut sont nécessaires.

### Les codes de statut sont des catégories, pas un diagnostic complet

- `2xx` : la requête a été traitée de la manière définie par le statut ; tout `202 Accepted` n’est pas encore nécessairement achevé sur le plan métier.
- `3xx` : une autre action ou représentation est requise ; les redirections peuvent modifier la méthode et le flux des credentials.
- `400` : la requête est erronée du point de vue du serveur.
- `401` : credentials d’authentification absents ou invalides ; la réponse utilise en principe `WWW-Authenticate`.
- `403` : le serveur comprend la requête, mais la refuse.
- `404` : ressource introuvable ou volontairement masquée ; ce n’est pas une preuve certaine de son inexistence.
- `409` : conflit avec l’état actuel.
- `412` : une précondition telle que `If-Match` n’est pas satisfaite.
- `429` : trop de requêtes dans une fenêtre de temps ; `Retry-After` peut indiquer une durée d’attente.
- `5xx` : le serveur n’a pas pu satisfaire une requête fondamentalement valide ; elle n’est pas automatiquement réessayable.

La RFC 6585 définit `429 Too Many Requests`, mais ni la portée du quota ni le compteur. Ceux-ci peuvent s’appliquer par credential, utilisateur, tenant, ressource, région ou cluster ([RFC 6585 – Additional HTTP Status Codes](https://www.rfc-editor.org/rfc/rfc6585.html)). Le client enregistre donc le statut, les champs de réponse pertinents, la Request-ID et le corps d’erreur tronqué.

## Variantes de transport et coûts de connexion

La sémantique HTTP est distincte de la version Wire utilisée. HTTP/1.1 emploie des règles textuelles de framing des messages, HTTP/2 des flux multiplexés et un framing binaire, HTTP/3 implémente HTTP sur QUIC. Une API Gateway peut accepter HTTP/2 côté client et parler HTTP/1.1 avec le backend ; le protocole côté client ne prouve pas l’ensemble du chemin vers le backend ([RFC 9112 – HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112.html), [RFC 9113 – HTTP/2](https://www.rfc-editor.org/rfc/rfc9113.html), [RFC 9114 – HTTP/3](https://www.rfc-editor.org/rfc/rfc9114.html)).

Les administrateurs n’observent pas seulement la latence de requête, mais aussi les temps DNS et de connexion, le handshake TLS, la réutilisation des connexions, la version HTTP/ALPN, le temps du proxy/de la passerelle, le Time to First Byte, le transfert du corps, le nombre de tentatives et la durée Wall-Clock totale.

### Vérifier le chemin des noms, TCP et TLS

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) et [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) vérifient [DNS](/kb/dns). [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) et [`nc`](https://man.openbsd.org/nc) vérifient TCP. [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) affiche le handshake TLS, le chemin de certificat et ALPN ; aucun de ces tests ne prouve une autorisation API réussie.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für API-Netz- und TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
    </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
    </code></pre>
  </div>
</div>
Une fois la méthode, le transport et la représentation compris, vient la première particularité distribuée : une modification réussie ne doit pas être immédiatement visible sur chaque chemin de lecture.

## Caching et Read-after-write

Le caching HTTP stocke des représentations à l’aide de clés de cache et de directives. `Cache-Control`, `Vary`, les validators et les règles d’authentification déterminent si et comment elles sont réutilisées. Un `200` peut provenir d’un cache ; un GET immédiatement consécutif à un PUT peut, selon l’architecture, encore voir l’ancien état ([RFC 9111 – HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html)).

Questions d’administration :

- Un cache de navigateur, proxy, CDN, passerelle ou application se trouve-t-il sur le chemin ?
- Quels en-têtes composent la clé de cache, notamment Authorization et tenant ?
- La représentation est-elle privée, publique ou non cacheable ?
- Pendant combien de temps les réponses négatives sont-elles cacheables ?
- Existe-t-il Read-your-writes ou eventual consistency ?
- Quelle région/réplique lit le GET suivant ?
- ETag représente-t-il une version du contenu ou seulement un cache validator ?

La purge du cache n’est pas une réparation universelle. Elle peut provoquer des pics de charge et masquer l’incohérence réelle.

## Limites de débit, quotas et saturation

Une limite de débit, un quota et une limite de concurrence sont des contrôles distincts :

- **Débit :** requêtes ou points de coût par fenêtre de temps.
- **Quota :** consommation totale par jour, mois ou abonnement.
- **Concurrence :** requêtes/flux simultanément actifs.
- **Limite de payload :** taille du corps, de l’objet, du batch ou de la réponse.
- **Limite de complexité :** profondeur de requête, coût GraphQL ou relations développées.

`429` peut fournir `Retry-After`, mais les en-têtes de limite de débit spécifiques aux fournisseurs ne sont pas uniformes. Le client traite les champs documentés comme faisant partie du contrat concret, et non comme une norme universelle. Il limite localement le débit d’interrogation, répartit le budget entre les charges de travail et enregistre la portée, le budget restant et l’heure de réinitialisation.

Le throttling est un signal de protection, pas un mode de débit normal. Des vagues persistantes de 429 indiquent une pagination inadaptée, l’absence de cache, un parallélisme trop élevé ou une capacité insuffisante.

## Corps d’erreur et corrélation

Un statut HTTP est trop grossier pour l’automatisation métier. La RFC 9457 définit Problem Details avec une URI `type` stable, `title`, `status`, `detail` et `instance`, ainsi que des champs d’extension. Le type de problème est l’identité lisible par machine ; le texte formulé librement n’est pas destiné aux parseurs ([RFC 9457 – Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457.html)).

Un bon contrat d’erreur fournit un type d’erreur/de problème stable, un statut HTTP/RPC, une indication de détail sûre, les chemins de champs concernés, une Request-ID/Correlation-ID, la possibilité de retry et un lien de documentation. Le client ne journalise ni jetons complets, ni en-têtes Authorization, ni payloads confidentiels.

### Capturer la réponse d’erreur avec les en-têtes

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für API-Fehler- und Korrelationsdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
    </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
    </code></pre>
  </div>
</div>

## RPC, GraphQL et gRPC

Toutes les API ne correspondent pas au style fondé sur les ressources.

| Style | Centre du contrat | Force | Limite opérationnelle |
|---|---|---|---|
| REST/HTTP | ressource, représentation, sémantique HTTP | intermédiaires web, caching, large prise en charge des outils | conventions de détail hétérogènes |
| RPC | service et opération | représentation directe des actions métier | retry/idempotence explicites pour chaque méthode |
| GraphQL | schéma typé et requête client | sélection flexible de données liées | coût des requêtes, N+1, souvent HTTP 200 malgré des erreurs de champ |
| gRPC | service/message Protobuf | génération de code, HTTP/2, unary et streaming | framing binaire, proxys, statut/trailers gRPC |
| API événementielle | canal, message, type d’événement | découplage et traitement asynchrone | ordre, déduplication, replay, Consumer Lag |

La spécification GraphQL définit le langage, le système de types, la validation et l’exécution ; le transport, l’authentification, les limites de débit et les coûts opérationnels des requêtes sont des contrats supplémentaires ([GraphQL Specification](https://spec.graphql.org/September2025/)). Des erreurs de champ peuvent survenir avec des données partielles ; un statut HTTP seul ne décrit pas le résultat.

gRPC mappe les channels, RPC et messages préfixés par longueur sur des flux HTTP/2. Le statut gRPC est transmis dans les trailers et doit être distingué du statut HTTP. Les appels ne sont pas automatiquement idempotents ; deadline, cancellation et politique de retry sont compris par service ([gRPC – What is gRPC?](https://grpc.io/docs/what-is-grpc/), [gRPC over HTTP/2 protocol](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)). Protocol Buffers fournit un modèle d’interface et de sérialisation ; les numéros de champ sont des ancres de compatibilité et ne doivent pas être réutilisés avec une autre signification après suppression ([Protocol Buffers – Language Guide](https://protobuf.dev/programming-guides/proto3/)).

[`grpcurl`](https://github.com/fullstorydev/grpcurl) peut utiliser Server Reflection ou des descripteurs locaux pour examiner les services gRPC. Reflection constitue elle-même une surface exposée et ne doit pas être activée publiquement sans vérification.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für gRPC-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
    </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
    </code></pre>
  </div>
</div>

## Représentations : JSON est une syntaxe, le schéma est le contrat

La RFC 8259 définit JSON comme format d’échange avec objets, tableaux, nombres, chaînes, booléens et null. JSON ne définit pas quel champ est un ID stable, si un champ absent et `null` ont le même sens, quel fuseau horaire possède un timestamp ou si des propriétés inconnues sont tolérées ([RFC 8259 – JSON](https://www.rfc-editor.org/rfc/rfc8259.html)).

Cette sémantique appartient à un schéma et à la documentation du contrat :

- nom du champ, type, format et unité ;
- required, optional, nullable et valeur par défaut ;
- read-only/write-only et généré par le serveur ;
- valeurs d’enum et comportement face aux valeurs inconnues ;
- format horaire, fuseau horaire et précision ;
- ID stable par rapport au nom d’affichage ;
- sémantique de référence, d’imbrication et de suppression ;
- règle de compatibilité pour les champs nouveaux ou supprimés.

JSON Schema définit des vocabulaires permettant de valider les instances JSON. Un schéma peut vérifier la structure, mais ne remplace pas les invariants métier ni l’autorisation ([JSON Schema – Specification](https://json-schema.org/specification)). La spécification OpenAPI peut décrire de manière lisible par machine les opérations HTTP, les paramètres, les schémas de requête/réponse et les Security Schemes ; elle ne prouve pas que l’implémentation exécutée correspond au document ([OpenAPI Specification](https://spec.openapis.org/oas/)).

### Inspecter la réponse et les champs sans interface graphique

[`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod) désérialise les réponses structurées ; [`Invoke-WebRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest) retourne davantage de détails HTTP. [`curl`](https://curl.se/docs/manpage.html) affiche requête/réponse et timing, [`jq`](https://jqlang.org/manual/) filtre JSON.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für API-Response-Inspektion">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
    </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
    </code></pre>
  </div>
</div>

## Contrats et dérive de contrat

Un contrat API complet comprend plus que des schémas de Happy Path :

| Domaine contractuel | Doit être défini |
|---|---|
| Discovery | Base URL, région, tenant, endpoint de service/métadonnées |
| Opération | méthode/RPC/événement, binding des paramètres, effet secondaire |
| Données | schéma, ID, ordre, sémantique null/par défaut, limites de taille |
| sécurité | Authflow, type de credential, audience, scope/rôle, durée de vie du jeton |
| Erreur | statut/code/type de problème, possibilité de retry, Request-ID |
| Cohérence | Read-after-write, retard de réplication, cache et ETag |
| Quantité | pagination, filtre, tri, sémantique de snapshot/curseur |
| Temps | deadline client/passerelle/serveur, Retry-After, Clock Skew |
| Cycle de vie | version du contrat, deprecation, sunset et chemin de migration |
| Exploitation | quota, SLO, maintenance, page de statut, corrélation avec le support |

La dérive de contrat survient lorsque le document, le SDK et l’implémentation de production divergent. La spécification publiée est donc stockée comme artefact versionné, validée dans la CI et vérifiée contre un environnement de test réel. Les clients générés réduisent le travail de saisie, mais propagent aussi les erreurs et les Breaking Changes du schéma vers de nombreux consommateurs.

Les tests pilotés par les consommateurs peuvent rendre visibles les hypothèses d’un client. Ils ne remplacent pas la sémantique du fournisseur : un mock peut fournir un `200`, alors que la production renvoie après une mise à jour de la passerelle un autre en-tête, une autre pagination ou un nouvel enum.

Jusqu’ici, l’appel était techniquement valide. Le contrôle de sécurité détermine s’il peut également être exécuté par la bonne identité sur le bon objet.

## L’authentification n’est pas l’autorisation

Une API-Key ou un jeton répond d’abord à la question de savoir **quel client ou principal** parle. L’autorisation détermine ensuite quelle action est permise sur quelle ressource et dans quel scope. Un jeton valide peut donc être correctement rejeté avec `403`.

| Procédure | Force et usage | Risque opérationnel |
|---|---|---|
| API-Key | identification simple du client ou ancre de quota | souvent valable longtemps, peu de scope, facilement copiable |
| Basic Auth | nom d’utilisateur/mot de passe via TLS | cycle de vie du mot de passe, limites MFA/délégation |
| mTLS | TLS mutuel, certificat client | PKI, rotation, terminaison proxy, mapping vers le principal |
| OAuth Access Token | autorisation déléguée ou de charge de travail avec scope/audience | obtention du jeton, expiration, consentement, replay |
| requête signée | intégrité de parties sélectionnées du message | Canonicalization, Clock Skew, magasin de nonce/replay |
| identité réseau | réseaux privés, Service Mesh, certificats de charge de travail | ne doit pas remplacer silencieusement le RBAC métier |

OAuth 2.0 définit des rôles et mécanismes de grant pour l’émission d’Access Tokens ; les Bearer Tokens peuvent être utilisés par toute personne qui les possède ([RFC 6749 – OAuth 2.0](https://www.rfc-editor.org/rfc/rfc6749.html), [RFC 6750 – Bearer Token Usage](https://www.rfc-editor.org/rfc/rfc6750.html)). Le BCP de sécurité RFC 9700 exige des privilèges minimaux, une Audience Restriction et la protection des flux de redirection ; il interdit le Resource Owner Password Credentials Grant. Pour la protection contre le replay, il cite les jetons liés à l’expéditeur via mTLS ou DPoP ([RFC 9700 – OAuth 2.0 Security BCP](https://www.rfc-editor.org/rfc/rfc9700.html), [RFC 8705 – OAuth mTLS](https://www.rfc-editor.org/rfc/rfc8705.html), [RFC 9449 – DPoP](https://www.rfc-editor.org/rfc/rfc9449.html)).

OpenID Connect ajoute une couche d’identité à OAuth ; un ID Token est destiné au client et n’est pas automatiquement un Access Token pour une API ([OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)). Un JWT n’est qu’un format compact de claims. La seule vérification de signature ne suffit pas : l’algorithme, l’issuer, l’audience, les time claims, la sélection de clé et les claims spécifiques à l’application doivent être validés ([RFC 7519 – JSON Web Token](https://www.rfc-editor.org/rfc/rfc7519.html), [RFC 8725 – JWT Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725.html)).

### Obtenir un jeton de charge de travail et appeler l’API

Le flux Client Credentials ne convient que lorsque l’application agit en son propre nom et peut protéger son credential de manière sûre. Le secret, le certificat ou l’identité de charge de travail fédérée, l’endpoint de jeton, l’audience/la ressource et le scope relèvent de la documentation de la plateforme concernée.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für OAuth-Client-Credentials-Test">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
    </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
    </code></pre>
  </div>
</div>

Les secrets n’apparaissent ni dans la ligne de commande, ni dans la transcription, ni dans le journal de débogage. L’exemple montre le flux de protocole, et non un transport de secret approprié pour les processus de production.

## API Gateway et limites de confiance

Une passerelle peut terminer TLS, valider les jetons, centraliser le routage, les quotas, les filtres de schéma, les règles WAF et l’observabilité. Elle constitue ainsi un point de contrôle et un périmètre de défaillance. Le service backend ne doit pas supposer silencieusement que chaque requête est passée par ce chemin de passerelle précis.

- **Client → Gateway :** identité d’hôte publique, TLS, DDoS/quota, credential client.
- **Gateway → Service :** identité mTLS ou de charge de travail propre ; aucune hypothèse de confiance aveugle fondée sur l’IP source.
- **Identity Provider → Validator :** métadonnées d’issuer, JWKS, rotation de clés, cache et horloge.
- **Service → stockage de données :** autorisation métier et limite de tenant.
- **Fournisseur de webhook → Receiver :** signature, fenêtre temporelle, Event-ID et vérification de replay.

La RFC 9700 met explicitement en garde contre les en-têtes de forwarding entrants non vérifiés lors de l’utilisation de Reverse Proxies. Le proxy doit nettoyer les champs importants pour la sécurité ; le lien interne doit être protégé contre l’écoute, l’injection et le replay ([RFC 9700 – OAuth 2.0 Security BCP](https://www.rfc-editor.org/rfc/rfc9700.html)).

Le statut de passerelle `200` ne prouve pas qu’une file d’attente ou une réplication en aval est saine. Inversement, un backend peut être sain tandis que DNS, certificat, validateur de jeton ou quota bloque chaque client.

Après la passerelle et le contrôle des autorisations subsiste la question opérationnelle la plus difficile : que s’est-il passé lorsque le client n’obtient pas de réponse à temps ? Un timeout ne prouve pas que le serveur n’a rien modifié.

## Timeouts, deadlines et exécution partielle

Un « timeout » n’est pas un résultat serveur. Le client sait seulement qu’aucune réponse exploitable n’est arrivée dans son délai. La requête peut avoir échoué avant l’établissement de connexion, avoir été rejetée par la passerelle, être encore active dans le service ou déjà avoir été commitée alors que seule la réponse a été perdue.

Chaque couche peut avoir son propre délai : DNS, connexion, TLS, temps total client, proxy, passerelle, upstream, base de données et file d’attente. La deadline externe doit être coordonnée avec les délais internes ; sinon, le client abandonne après 30 secondes pendant que le serveur continue à travailler 60 secondes et qu’un retry lance en parallèle la même action.

Un service propage autant que possible une deadline restante plutôt que de recommencer le temps complet à chaque hop. La cancellation est best effort : elle ne prouve pas qu’un effet secondaire déjà commité a été annulé.

## Retries, backoff et idempotence

La répétition automatique n’est autorisée que lorsque **la classe d’erreur et l’opération** le permettent. Un client robuste clarifie :

1. Une connexion a-t-elle seulement été établie ?
2. Un statut ou une erreur de protocole est-il présent ?
3. L’opération est-elle safe/idempotente ou protégée par déduplication ?
4. Le serveur fournit-il `Retry-After` ou une indication de backoff propre au produit ?
5. Reste-t-il suffisamment de deadline de bout en bout ?
6. Un retry aggrave-t-il une surcharge ?

Un backoff exponentiel avec jitter évite les vagues de retries synchrones. Le nombre de tentatives est limité et fait partie de la latence totale. `401` ou `403` ne sont pas réparés par des répétitions plus fréquentes ; `429` exige le respect du quota ; un `500` après un POST peut laisser un effet secondaire partiel malgré le corps d’erreur.

Pour une opération métier, le client conserve un ID d’opération stable. Le serveur conserve le résultat ou l’état de déduplication au moins aussi longtemps que la fenêtre de retry maximale. En l’absence d’un tel engagement, le client relit avant le retry à l’aide d’un ID d’objet stable ou d’une condition de recherche.

## Concurrence optimiste

Un read-modify-write sans condition de version provoque des Lost Updates :

```text
Client A liest Version 7     Client B liest Version 7
Client A schreibt Änderung  → Version 8
Client B schreibt alten Stand plus Änderung → A geht verloren
```

HTTP prend en charge les requêtes conditionnelles avec des validators comme `ETag`. Le client lit l’ETag et envoie `If-Match` lors de la modification ; si la représentation a changé, le serveur répond avec `412 Precondition Failed` au lieu d’écraser un état tiers. L’API concrète doit documenter si l’ETag est suffisamment fort pour cette sémantique ([RFC 9110 – HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für bedingte API-Änderung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
    </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
    </code></pre>
  </div>
</div>

## Pagination, filtres et ensembles cohérents

Un endpoint qui fournit aujourd’hui 50 objets peut en fournir 50'000 demain. La pagination fait partie du contrat :

- **Offset/Page :** simple, mais les insertions et suppressions peuvent créer des doublons ou des lacunes.
- **Cursor/Continuation Token :** encode la progression côté serveur ; le token est opaque et ne doit pas être interprété.
- **Keyset :** trié selon un ID de continuation stable et unique.
- **Snapshot :** maintient une vue cohérente sur plusieurs pages, mais nécessite un état serveur ou une ancre de version.

Le client suit le Next-Link ou le curseur documenté et ne le construit pas à partir d’hypothèses. La RFC 8288 définit les liens typés, mais pas de pagination universelle ; la relation concrète et la forme du corps restent un contrat API ([RFC 8288 – Web Linking](https://www.rfc-editor.org/rfc/rfc8288.html)).

Les filtres et le tri doivent rester stables d’une page à l’autre. Un tri portant uniquement sur un timestamp non unique est insuffisant ; un tie-breaker tel qu’un ID immuable doit s’y ajouter. Pour les API Delta/Change, le curseur, son expiration et le chemin de resynchronisation sont documentés.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für API-Pagination">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
    </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
    </code></pre>
  </div>
</div>


## Webhooks, événements et API asynchrones

Un succès HTTP synchrone et un traitement métier achevé sont des états différents. Un `202 Accepted` confirme selon la RFC 9110 uniquement que le serveur a accepté le traitement ; la tâche peut encore échouer plus tard. Un destinataire de webhook confirme inversement souvent seulement l’acceptation persistée d’un événement. Assimiler `2xx` à « processus métier terminé » fait perdre précisément les états intermédiaires pertinents avec les files d’attente, retries et défaillances partielles ([RFC 9110 – HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)).

Un événement exploitable doit contenir au minimum un Event-ID stable, le type d’événement et la version du schéma, l’heure de création, le producteur, l’ID de ressource ainsi que, lorsque l’ordre a une importance métier, une version de ressource ou de séquence. CloudEvents standardise à cette fin une enveloppe d’événement indépendante du fournisseur ; AsyncAPI décrit les canaux de messages et les opérations de manière lisible par machine, de façon analogue au rôle d’OpenAPI pour les API requête/réponse ([CloudEvents Specification](https://github.com/cloudevents/spec), [AsyncAPI Specification](https://www.asyncapi.com/docs/reference/specification/latest)).

Les webhooks sont exploités comme un client externe qui répète ses requêtes :

- L’expéditeur signe le **corps de requête non modifié** avec des métadonnées de temps ou de nonce ; le destinataire valide la signature, la fenêtre temporelle acceptée et le contexte cible avant le parsing. Les HTTP Message Signatures standardisées peuvent lier cryptographiquement des composants et des champs dérivés ([RFC 9421 – HTTP Message Signatures](https://www.rfc-editor.org/rfc/rfc9421.html)).
- Le destinataire déduplique à l’aide de l’Event-ID et persiste l’acceptation avant l’accusé positif. Le traitement est conçu de manière idempotente.
- Les retries possèdent une durée limitée, un backoff et un chemin Dead-Letter ou de quarantaine. Un replay est journalisé et ne crée pas une nouvelle identité métier.
- Une **exécution de réconciliation** périodique compare le système source et l’état local. Les webhooks sont des accélérateurs, mais pas nécessairement l’unique source de vérité.

## Sécurité des API : objet, fonction et flux de données

Une vérification réussie du jeton répond uniquement à qui, ou quelle charge de travail, parle et pour quelle audience le credential est destiné. Pour chaque objet et chaque opération, l’application doit en plus décider si cette identité peut lire ou modifier précisément ce tenant, cet utilisateur, cette clé ou cet ensemble de messages. L’OWASP API Security Top 10 souligne donc notamment Broken Object Level Authorization, Broken Authentication, la consommation non limitée de ressources, SSRF et un inventaire API défaillant comme classes de risques distinctes ([OWASP API Security Top 10](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)).

Pour les API d’infrastructure, il en résulte des contrôles concrets :

- **Référence à l’objet :** les ancres de tenant et d’objet proviennent d’un contexte validé côté serveur, et non seulement d’un champ de chemin ou de corps librement sélectionnable.
- **Limites d’entrée :** Content-Type, schéma, longueurs de champ, imbrication, taille totale, taux de compression et temps de traitement sont limités.
- **Connexions sortantes :** les URL issues de requêtes ou de webhooks passent par une allowlist, un contrôle DNS/IP et une Egress Policy ; les redirections sont vérifiées à nouveau.
- **Credentials :** les jetons n’apparaissent ni dans l’URI ni dans les logs ; les secrets font l’objet de rotations, sont limités à l’audience cible et aux scopes minimaux, et ne sont pas intégrés dans les artefacts clients.
- **Trust Hops :** si une passerelle termine [TLS](/kb/tls), le hop backend doit être authentifié et autorisé séparément. Un en-tête Forwarded digne de confiance ne naît qu’à une limite de proxy contrôlée.
- **Audit :** les modifications privilégiées consignent le client, le principal, l’objet cible, l’action, la référence avant/après, la Request-ID et le résultat, sans secret ni payload sensible complet.

OAuth 2.0 Security Best Current Practice déconseille notamment le Resource Owner Password Credentials Grant, exige des comparaisons exactes des Redirect URI et privilégie les jetons liés à l’expéditeur ou de courte durée lorsque le modèle de menace l’exige ([RFC 9700 – Best Current Practice for OAuth 2.0 Security](https://www.rfc-editor.org/rfc/rfc9700.html)). Mutual TLS et DPoP sont deux procédures distinctes de liaison à l’expéditeur ; toutes deux modifient l’exploitation des clés et le diagnostic des erreurs, et ne sont pas de simples interrupteurs sur la passerelle ([RFC 8705 – OAuth 2.0 Mutual-TLS Client Authentication](https://www.rfc-editor.org/rfc/rfc8705.html), [RFC 9449 – OAuth 2.0 Demonstrating Proof of Possession](https://www.rfc-editor.org/rfc/rfc9449.html)).

Les retries, la pagination et les événements génèrent plusieurs opérations techniques pour une action métier. La corrélation et l’audit doivent les relier à nouveau en un déroulement traçable.

## Observabilité et appels prouvables

Les métriques montrent le volume, les logs les décisions individuelles et les traces le chemin d’une requête au-delà des frontières de processus. OpenTelemetry modélise une trace comme un ensemble causal de spans et définit les ID de trace et de span pour la corrélation ([OpenTelemetry – Traces](https://opentelemetry.io/docs/specs/otel/trace/)). À des fins d’administration, un appel API devrait permettre de reconstruire au minimum les faits suivants :

| Dimension | Preuve opérationnelle |
|---|---|
| Appelant | Client-ID, charge de travail ou utilisateur ; méthode d’authentification ; rôles/scopes effectifs |
| Cible | hôte, tenant, version API/contrat, méthode ou opération, identifiant de ressource stable |
| Exécution | heure de début, durée totale, temps DNS/connexion/TLS lorsque disponibles, deadline, numéro de retry |
| Résultat | statut de transport, code d’erreur métier, taille de réponse, état de Rate Limit/quota |
| Corrélation | Request-ID du serveur, Trace-ID, Job-/Event-ID et, dans le contexte de messagerie, Message-ID |

Les ID sont transmis aux frontières de processus, mais ne sont pas repris aveuglément depuis des clients externes quelconques comme autorité interne. Les labels de métriques évitent les ID d’utilisateurs, les chemins complets et les autres valeurs à haute cardinalité. Les payloads, en-têtes Authorization, cookies et signatures de webhook ne font généralement pas partie de la télémétrie. Une trace peut prouver le chemin, mais ne remplace pas une preuve d’audit protégée contre la manipulation concernant une modification privilégiée.

## Versionnage, deprecation et sunset

Un numéro de version n’est pas un cycle de vie. Il faut d’abord distinguer les **extensions compatibles** des **Breaking Changes**. De nouveaux champs facultatifs, des valeurs enum supplémentaires ou un ordre modifié peuvent casser les clients malgré une apparente rétrocompatibilité lorsque ceux-ci implémentent le contrat de manière trop stricte. Les tests consommateurs et le schema diffing ne vérifient donc pas seulement les chemins, mais aussi la sémantique, les autorisations, les erreurs et les valeurs limites.

Les versions peuvent figurer dans le chemin, l’hôte, l’en-tête ou le type de média ; l’essentiel est que le routage, la documentation, la télémétrie et le support désignent sans ambiguïté la même variante. Pour l’abandon progressif, la RFC 9745 standardise le champ HTTP `Deprecation`; la RFC 8594 définit `Sunset` comme le moment à partir duquel une ressource ne répondra vraisemblablement plus. Aucun des deux ne remplace des instructions de migration, un lien alternatif ou un inventaire de clients prouvé ([RFC 9745 – The Deprecation HTTP Response Header Field](https://www.rfc-editor.org/rfc/rfc9745.html), [RFC 8594 – The Sunset HTTP Header Field](https://www.rfc-editor.org/rfc/rfc8594.html)).

Un processus d’abandon robuste comprend l’inventaire des consommateurs, les métriques d’utilisation par version et par client, les dates annoncées, l’exploitation parallèle, un environnement de test, un chemin de repli et une décision explicite de désactivation. « Annoncé dans le wiki » ne prouve pas que les automatisations non surveillées ont été migrées.

## Modèles d’exploitation : local, cloud et Control Plane

L’emplacement d’une API ne détermine pas à lui seul la sécurité ou la maîtrise. Une interface locale peut être directement liée à des comptes système privilégiés, à des clés de longue durée et à des réseaux peu segmentés. Une Cloud-Control-Plane peut en revanche offrir de fortes identités de charge de travail et des journaux d’audit, mais reste dépendante du chemin Internet, de l’IAM du fournisseur, de la configuration du tenant, des quotas et de la disponibilité du service. Le facteur déterminant est l’espace concret de défaillance et de confiance.

| Modèle | Limite typique | Questions d’administration |
|---|---|---|
| API de processus/hôte locale | Unix Socket, Named Pipe, Loopback ou réseau de gestion | Quelle identité OS s’applique ? Qui possède le socket/l’ACL ? L’accès distant est-il réellement exclu ? |
| API de service interne | segment, Service Mesh, passerelle ou Load Balancer | Où se terminent TLS et l’autorisation ? Comment sont exploités les identités de service et DNS ? |
| SaaS-Control-Plane | endpoint du fournisseur et Tenant-IAM | Quelle région, quels chemins de quota, d’audit et de jeton s’appliquent ? Comment fonctionne Break Glass ? |
| plan de données plus Control Plane | la configuration pilote des workers ou appliances séparés | Quand une modification est-elle distribuée ? Comment détecter la dérive, le rollback et les états partiels ? |
| intégration événement/webhook | producteur, broker ou callback public | Qui gère la livraison, le retry, la signature, la DLQ et la réconciliation ? |

Les sauvegardes ne sécurisent pas automatiquement une API externe. Pour le redémarrage, on inventorie plutôt les contrats, la configuration client, les références de secret, les certificats, les règles de passerelle, l’état d’idempotence, les jobs ouverts et la capacité de réconciliation. Les tests de récupération doivent également couvrir les jetons expirés, les cibles DNS modifiées et les Continuation Tokens réinitialisés.

## Histoire technique

Les premières interfaces distribuées étaient souvent étroitement liées à Remote Procedure Call et à des stubs spécifiques à un langage. SOAP 1.2 a ensuite défini une enveloppe de messages fondée sur XML avec un modèle de traitement extensible et s’est répandu avec WSDL et WS-* dans les plateformes d’entreprise ([W3C – SOAP Version 1.2 Part 1](https://www.w3.org/TR/soap12-part1/)). La thèse de Roy Fielding a décrit REST en 2000 comme un style architectural pour les systèmes hypermédias distribués et a dérivé les contraintes des exigences du Web, et non d’une recette « HTTP plus JSON » ([Fielding – Architectural Styles and the Design of Network-based Software Architectures](https://ics.uci.edu/~fielding/pubs/dissertation/top.htm)).

HTTP a évolué parallèlement, des connexions TCP persistantes de HTTP/1.1 aux flux multiplexés de HTTP/2 jusqu’à HTTP/3 sur QUIC. La méthodologie et la sémantique des statuts sont décrites indépendamment du transport dans la RFC 9110 ; les formats Wire se trouvent dans les RFC 9112, RFC 9113 et RFC 9114 ([RFC 9112 – HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112.html), [RFC 9113 – HTTP/2](https://www.rfc-editor.org/rfc/rfc9113.html), [RFC 9114 – HTTP/3](https://www.rfc-editor.org/rfc/rfc9114.html)).

JSON a été standardisé comme format d’échange léger ; JSON Schema et OpenAPI ont complété les contrats de structure et d’opération lisibles par machine. GraphQL décrit un modèle typé de requête et d’exécution dans lequel les clients sélectionnent les champs ; gRPC associe des définitions RPC orientées services à Protocol Buffers et à un framing fondé sur HTTP/2 ([JSON Schema Specification](https://json-schema.org/specification), [OpenAPI Specification](https://spec.openapis.org/oas/), [GraphQL Specification](https://spec.graphql.org/September2025/), [gRPC – What is gRPC?](https://grpc.io/docs/what-is-grpc/), [Protocol Buffers – Language Guide](https://protobuf.dev/programming-guides/proto3/)). Les modèles événementiels et de streaming complètent Request/Response, mais n’éliminent ni les contrats ni les questions de livraison et de cohérence.

## Checklist d’administration en un coup d’œil

Après le contrat, l’exécution et l’exploitation, la checklist suivante condense les questions auxquelles il faut répondre avant la mise en service d’une API. Elle est conçue comme aide à la réception, non comme remplacement des explications précédentes.

| Question | Preuve ou artefact |
|---|---|
| Quel style d’interaction est exploité ? | OpenAPI/schéma GraphQL/Proto/AsyncAPI, opération concrète et profil de transport |
| Quel endpoint s’applique ? | Scheme, FQDN, port, Base Path, région/tenant, preuve DNS et certificat |
| Qui appelle ? | Client-/Workload-ID, type de credential, Token-Issuer, audience, scopes/rôles, possession de clé |
| Quel est le contrat ? | méthodes, schémas, catalogue de statuts et d’erreurs, limites, pagination, idempotence et cycle de vie |
| Quand une répétition est-elle permise ? | deadline, sémantique ou clé idempotente, backoff, budget de retry et opération de consultation |
| Comment empêcher les Lost Updates ? | ETag/`If-Match`, numéro de version métier ou opération transactionnelle |
| Comment identifier les états partiels ? | statut de job/d’événement, Request-ID, trace, Queue-/Consumer-Lag, réconciliation |
| Comment les modifications sont-elles effectuées ? | staging/canary, tests de contrat et consommateurs, rollback, deprecation/sunset |
| Comment restaurer ? | configuration, contrats, références de secret/certificat, curseurs/jobs, test de replay et de réconciliation |
| Que doit contenir le Runbook ? | codes d’erreur connus, chemins 401/403/404/409/412/429/5xx, contacts et données d’escalade |

## Sources

- [Fielding – Network-based Application Architectures](https://ics.uci.edu/~fielding/pubs/dissertation/net_app_arch.htm)
- [Fielding – Network-based Architectural Styles](https://ics.uci.edu/~fielding/pubs/dissertation/net_arch_styles.htm)
- [Fielding – Representational State Transfer](https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)
- [RFC 3986 – Uniform Resource Identifier](https://www.rfc-editor.org/rfc/rfc3986.html)
- [RFC 9110 – HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
- [RFC 6585 – Additional HTTP Status Codes](https://www.rfc-editor.org/rfc/rfc6585.html)
- [RFC 9112 – HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112.html)
- [RFC 9113 – HTTP/2](https://www.rfc-editor.org/rfc/rfc9113.html)
- [RFC 9114 – HTTP/3](https://www.rfc-editor.org/rfc/rfc9114.html)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [RFC 9111 – HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html)
- [RFC 9457 – Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457.html)
- [GraphQL Specification](https://spec.graphql.org/September2025/)
- [gRPC – What is gRPC?](https://grpc.io/docs/what-is-grpc/)
- [gRPC over HTTP/2 Protocol](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)
- [Protocol Buffers – Language Guide](https://protobuf.dev/programming-guides/proto3/)
- [grpcurl](https://github.com/fullstorydev/grpcurl)
- [RFC 8259 – JSON](https://www.rfc-editor.org/rfc/rfc8259.html)
- [JSON Schema Specification](https://json-schema.org/specification)
- [OpenAPI Specification](https://spec.openapis.org/oas/)
- [Microsoft Learn – Invoke-RestMethod](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod)
- [Microsoft Learn – Invoke-WebRequest](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest)
- [curl manual](https://curl.se/docs/manpage.html)
- [jq manual](https://jqlang.org/manual/)
- [RFC 6749 – OAuth 2.0 Authorization Framework](https://www.rfc-editor.org/rfc/rfc6749.html)
- [RFC 6750 – OAuth 2.0 Bearer Token Usage](https://www.rfc-editor.org/rfc/rfc6750.html)
- [RFC 9700 – Best Current Practice for OAuth 2.0 Security](https://www.rfc-editor.org/rfc/rfc9700.html)
- [RFC 8705 – OAuth 2.0 Mutual-TLS Client Authentication](https://www.rfc-editor.org/rfc/rfc8705.html)
- [RFC 9449 – OAuth 2.0 Demonstrating Proof of Possession](https://www.rfc-editor.org/rfc/rfc9449.html)
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [RFC 7519 – JSON Web Token](https://www.rfc-editor.org/rfc/rfc7519.html)
- [RFC 8725 – JSON Web Token Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725.html)
- [RFC 8288 – Web Linking](https://www.rfc-editor.org/rfc/rfc8288.html)
- [CloudEvents Specification](https://github.com/cloudevents/spec)
- [AsyncAPI Specification](https://www.asyncapi.com/docs/reference/specification/latest)
- [RFC 9421 – HTTP Message Signatures](https://www.rfc-editor.org/rfc/rfc9421.html)
- [OWASP API Security Top 10](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)
- [OpenTelemetry – Traces](https://opentelemetry.io/docs/specs/otel/trace/)
- [RFC 9745 – Deprecation](https://www.rfc-editor.org/rfc/rfc9745.html)
- [RFC 8594 – Sunset](https://www.rfc-editor.org/rfc/rfc8594.html)
- [W3C – SOAP Version 1.2 Part 1](https://www.w3.org/TR/soap12-part1/)
- [Fielding – Architectural Styles and the Design of Network-based Software Architectures](https://ics.uci.edu/~fielding/pubs/dissertation/top.htm)
