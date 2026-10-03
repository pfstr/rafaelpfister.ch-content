---
title: "Apache James : serveur de messagerie modulaire et plateforme de Mailets"
blatt: "apache-james"
description: "Présentation technique d’Apache James : protocoles et rôles de messagerie, architecture basée sur les composants, file d’attente et pipeline de Mailets, modèle de boîtes aux lettres et de stockage, variantes d’exploitation de PostgreSQL à Cassandra ainsi que l’évolution du projet Java Apache vers une plateforme de messagerie JVM."
fakten:
  - label: Nom complet
    wert: Java Apache Mail Enterprise Server
    href: https://james.apache.org/
  - label: Catégorie
    wert: MTA, MDA, serveur de boîtes aux lettres et plateforme d’applications de messagerie
    href: https://james.apache.org/documentation.html
  - label: Projet
    wert: Apache Software Foundation
    href: https://projects.apache.org/committee.html?james
  - label: Runtime
    wert: JVM · Java 21 à partir de la version 3.9
    href: https://james.apache.org/james/update/2025/09/25/james-3.9.0.html
  - label: Langages
    wert: principalement Java, certains modules en Scala
    href: https://github.com/apache/james-project
  - label: Protocoles
    wert: SMTP, LMTP, IMAP, POP3, ManageSieve, JMAP
    href: https://james.apache.org/server/feature-protocols.html
  - label: Style architectural
    wert: modulaire, basé sur les composants, Inversion of Control, événementiel
    href: https://james.apache.org/
  - label: Backends
    wert: PostgreSQL/JPA ou Cassandra · OpenSearch · RabbitMQ · S3
    href: https://james.apache.org/download.cgi
  - label: Configuration
    wert: conf/*.xml et *.properties · variables d’environnement
    href: https://james.apache.org/server/config.html
  - label: Conditionnement
    wert: distributions ZIP et images Docker officielles
    href: https://james.apache.org/download.cgi
  - label: Administration
    wert: WebAdmin REST API, CLI et métriques
    href: https://james.apache.org/server/manage-webadmin.html
  - label: Supervision
    wert: Health Checks, Prometheus, JMX, journaux et Grafana
    href: https://james.apache.org/server/metrics.html
  - label: Licence
    wert: Apache License 2.0
    href: https://www.apache.org/licenses/LICENSE-2.0
werbung:
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: adbfb005b83b16086ba55e53dd469f3aff1e5642364da5ab8b2da5d265a1ce51
translationModel: gpt-5.6-terra
translatedAt: 2026-10-03T09:22:44.346Z
translationReview: required
---

# Apache James : serveur de messagerie modulaire et plateforme de Mailets

Apache James est un serveur de messagerie open source et, à la fois, une boîte à outils pour les applications dont la logique métier repose sur l’e-mail. Son nom signifie **Java Apache Mail Enterprise Server**. James peut accepter et relayer des messages via SMTP, gérer des boîtes aux lettres locales, les rendre disponibles via IMAP, POP3 ou JMAP et piloter l’ensemble du flux de messages au moyen de composants de traitement librement combinables. Le projet ne se décrit donc pas seulement comme un serveur, mais comme une **plateforme d’inversion de contrôle sur la JVM**, composable de manière modulaire ([Apache James – Présentation du projet](https://james.apache.org/)).

Ce double rôle distingue James des agents de transfert de courrier classiques tels que Postfix et des appliances de sécurité prêtes à l’emploi. Un administrateur peut exploiter James comme simple relais SMTP, comme serveur de boîtes aux lettres complet ou comme moteur de messagerie intégré à un produit. L’antispam, le chiffrement, l’archivage ou le routage spécifique à un domaine ne résultent pas d’un bloc fonctionnel rigide, mais d’un pipeline de **Matchers** et de **Mailets**. James est ainsi exceptionnellement adaptable, mais transfère une partie de la responsabilité produit du fabricant vers l’organisation qui l’exploite.

L’explication suit un message à travers James : des serveurs de protocoles à la file d’attente et au pipeline de Mailets, jusqu’au stockage des boîtes aux lettres. Les variantes d’exploitation, le diagnostic et enfin l’évolution technique du projet s’appuient sur cette base.

## Positionnement : MTA, MDA et plateforme applicative

Dans un système de messagerie, chaque composant ne remplit pas le même rôle. Un **Mail User Agent** (MUA) est le client de l’utilisateur, par exemple Thunderbird. Un **Mail Transfer Agent** (MTA) transporte les messages entre systèmes. Un **Mail Delivery Agent** (MDA) dépose un message dans la boîte aux lettres cible. James peut être simultanément MTA et MDA ; par ses modules de protocoles et de boîtes aux lettres, il fournit en outre des services côté serveur aux MUA. La vue d’ensemble officielle des composants mentionne à cet effet des projets distincts pour le serveur, les protocoles, les Mailets, les boîtes aux lettres et les tests ([Apache James – Composants logiciels](https://james.apache.org/documentation.html)).

| Rôle | Mise en œuvre dans James | Point de transfert |
|---|---|---|
| Transport de messages | Serveurs SMTP et LMTP, file d’attente, Mailet de livraison distante | autres MTA, relais et passerelles |
| Livraison locale | Pipeline de Mailets et Mailbox API | utilisateurs, domaines et quotas |
| Accès aux boîtes aux lettres | IMAP, POP3 et JMAP | clients de messagerie et applications web |
| Logique de filtrage | Matchers, Mailets, Processors et Sieve | règles internes et services de contrôle externes |
| Administration | WebAdmin REST API, CLI, Health Checks et métriques | automatisation et supervision |

James n’est donc **pas un client de messagerie**, ni une passerelle de messagerie sécurisée préconfigurée. Il fournit des briques pour le transport, la livraison, le stockage et le traitement. La distribution et la configuration choisies déterminent s’il devient un simple relais, un service de messagerie mutualisé ou une passerelle spécifique à un produit.

## Protocoles, TLS et ports

James fournit SMTP, LMTP, IMAP, POP3 et ManageSieve comme services basés sur TCP ; JMAP et WebAdmin utilisent HTTP ([Apache James – Serveurs de protocoles](https://james.apache.org/server/feature-protocols.html)). Selon l’écouteur, TLS protège une connexion chiffrée dès le départ ou est inséré dans une session existante via StartTLS. DNS ne fait pas partie du processus James, mais il est indispensable à un MTA public : les enregistrements MX déterminent la destination, les enregistrements A et AAAA ses adresses, et les enregistrements PTR influencent la réputation des connexions sortantes.

Le seul numéro de port ne décrit pas encore la sémantique de sécurité. Le port 25 est destiné au transport de serveur à serveur ; la soumission authentifiée par les clients relève du port 587 selon la [RFC 6409](https://datatracker.ietf.org/doc/html/rfc6409). Depuis la [RFC 8314](https://datatracker.ietf.org/doc/html/rfc8314), le port 465 est à nouveau enregistré pour la soumission de messages avec chiffrement implicite. Les mêmes deux modèles s’appliquent à IMAP et POP3 : connexion en clair avec StartTLS possible ou établissement immédiat de TLS.

| Service | Ports typiques | Standard | Rôle dans James |
|---|---:|---|---|
| SMTP | 25, 587, 465 | [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321) | acceptation, relais et soumission |
| LMTP | configurable, 24 enregistré | [RFC 2033](https://datatracker.ietf.org/doc/html/rfc2033) | transfert local avec statut par destinataire |
| IMAP4rev2 | 143, 993 | [RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051) | accès synchrone aux boîtes aux lettres |
| POP3 | 110, 995 | [RFC 1939](https://datatracker.ietf.org/doc/html/rfc1939) | récupération simple des messages |
| ManageSieve | 4190 | [RFC 5804](https://datatracker.ietf.org/doc/html/rfc5804) | gestion des règles Sieve propres aux utilisateurs |
| JMAP Mail | généralement 443 | [RFC 8621](https://datatracker.ietf.org/doc/html/rfc8621) | accès HTTP aux boîtes aux lettres pour les clients modernes |

Les ports sont configurables ; la combinaison de l’écouteur, du protocole, du mode TLS et de l’authentification est déterminante. La [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) reste la référence pour les affectations enregistrées.

## Approche architecturale

James suit une **architecture basée sur les composants**. Les serveurs de protocoles, la file d’attente, la logique de traitement, les boîtes aux lettres, la gestion des utilisateurs, l’index de recherche et l’administration sont séparés par des API et assemblés via l’injection de dépendances. Les distributions documentées pour James 3.9 reposent à cet effet sur Google Guice ; l’architecture Spring appartient à une génération antérieure. Ce découplage ne relève pas seulement de l’organisation du code : il permet d’utiliser la même [Mailbox API](https://james.apache.org/mailbox/mailbox-api.html) avec différentes couches de persistance et la même logique de Mailets dans des profils de serveur très différents.

Le chemin de données central est asynchrone. Un écouteur SMTP n’a pas besoin de livrer intégralement un message accepté avant de répondre à la connexion. Il place un objet mail dans une file d’attente ; un **Spooler** l’en retire ultérieurement et l’exécute à travers le conteneur de Mailets. La file d’attente sépare ainsi la charge de réception, la durée de traitement et la disponibilité des systèmes en aval. La documentation d’exploitation distribuée la désigne donc à juste titre comme un composant obligatoire d’un serveur SMTP ([Apache James – Exploitation d’un serveur distribué](https://james.apache.org/server/manage-guice-distributed-james.html)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 976" src="/images/apache-james-architektur.svg?v=20260813" title="Interaktive Infografik: technische Architektur und Nachrichtenfluss von Apache James" loading="lazy">
  <a href="/images/apache-james-architektur.svg?v=20260813">Ouvrir l’infographie sur l’architecture technique</a>
</iframe>

### Le parcours de traitement d’un message

1. **Acceptation par protocole :** SMTP ou LMTP vérifie la session, l’authentification, l’expéditeur d’enveloppe et les destinataires. Après la fin de `DATA`, un objet interne `Mail` est créé avec l’enveloppe, le contenu MIME et les attributs.
2. **File d’attente :** l’objet est mis en file de manière persistante ou volatile. Ce n’est qu’à partir de ce point que l’acceptation est découplée du traitement.
3. **Spooler :** des workers retirent les entrées de la file d’attente et les transmettent au conteneur de Mailets.
4. **Processor :** un Processor nommé contient une liste ordonnée de paires Matcher/Mailet. Le Processor obligatoire `root` constitue le point d’entrée.
5. **Matcher :** un Matcher ne modifie pas le message, mais renvoie le sous-ensemble des destinataires pour lesquels une condition s’applique.
6. **Mailet :** le Mailet associé modifie le message ou l’enveloppe, déclenche un effet secondaire, livre localement ou à distance, ou bifurque vers un autre Processor.
7. **Résultat :** le message aboutit dans une boîte aux lettres utilisateur, dans la livraison sortante, dans un Mail Repository pour traitement ultérieur, ou est terminé après une action réussie.

Un détail important est la **division par destinataire**. Si un Matcher ne correspond qu’à une partie des destinataires, le conteneur divise le traitement entre ensembles de destinataires correspondants et non correspondants. Les règles ne s’appliquent donc pas nécessairement à un message MIME complet. Un Mailet peut en outre sauter directement vers un autre Processor via `ToProcessor` ; le pipeline ressemble ainsi davantage à un graphe de traitement orienté qu’à une simple liste linéaire. La [documentation officielle du conteneur de Mailets](https://james.apache.org/server/feature-mailetcontainer.html) décrit précisément ce modèle.

Un modèle minimal et simplifié se présente ainsi :

```xml
<processor state="root" enableJmx="true">
  <mailet match="RelayLimit=30" class="ToRepository">
    <repositoryPath>cassandra://var/mail/relay-denied/</repositoryPath>
  </mailet>
  <mailet match="RecipientIsLocal" class="LocalDelivery" />
  <mailet match="All" class="RemoteDelivery" />
</processor>
```

L’ordre fait partie de la sémantique. Une règle très large placée au début peut rendre les règles suivantes inaccessibles ; une boucle infinie entre Processors peut bloquer le Spooler. James propose donc un comportement d’erreur configurable pour chaque Matcher et Mailet, ainsi que des Processors d’erreur dédiés ([Configuration du conteneur de Mailets](https://james.apache.org/server/config-mailetcontainer.html)).

L’architecture par composants devient concrète dès qu’un message atteint la file d’attente. Les Processors, Matchers et Mailets déterminent alors les étapes de traitement suivantes et la destination du résultat.

## Structure technique

L’architecture décrit le parcours des messages ; pour l’installation et l’exploitation, elle doit maintenant se traduire par une vue concrète des composants. Le point décisif est de savoir quels runtimes, stockages et services supplémentaires le profil James choisi nécessite réellement.

### Stack technologique et vue d’ensemble pour l’administrateur

Pour une première évaluation du produit, les limites d’exploitation sont plus importantes que les noms de classes. La vue d’ensemble suivante synthétise le stack autour des questions qui doivent être clarifiées avant l’installation, l’intégration ou la reprise d’un environnement existant :

| Domaine | Technologie ou artefact | Ce que l’administrateur doit savoir |
|---|---|---|
| Runtime | Java 21, JVM ; code source majoritairement Java, quelques modules Scala | le heap, le garbage collection, le threading et les correctifs JVM font partie de l’exploitation du serveur |
| Build et paquet | projet Maven multimodule ; ZIP et images Docker | les Mailets personnalisés doivent correspondre aux générations de James, Java et Jakarta |
| Câblage | Guice dans la génération 3.9 ; Spring dans les installations plus anciennes | la distribution choisie détermine les modules et fichiers de configuration disponibles |
| Configuration | `conf/*.xml`, `conf/*.properties`, variables d’environnement | particulièrement importants : `smtpserver.xml`, `mailetcontainer.xml`, `webadmin.properties`, fichiers JMAP et backend |
| Traitement | MailQueue, Spooler, Processor, Matcher, Mailet | acceptation, traitement et livraison finale sont des états séparés |
| Données | PostgreSQL/JPA ou Cassandra ; S3, OpenSearch, RabbitMQ en option | source, projection, file d’attente et contenu blob requièrent des plans de reprise distincts |
| Administration | WebAdmin REST API et `james-cli` | REST est plus puissant ; la CLI est incluse dans chaque variante de câblage |
| Observabilité | Health Checks, Dropwizard Metrics, Prometheus, JMX, journaux, Grafana | la file d’attente, les Mailets, les Matchers, les protocoles et les backends possèdent leurs propres métriques |
| Sécurité | keystores TLS, SMTP AUTH, JWT pour WebAdmin, segmentation réseau | WebAdmin sans JWT activé n’est pas protégé par défaut |

Selon le projet, tous les fichiers de configuration se trouvent dans `conf` ou `conf/META-INF`; ceux qui s’appliquent réellement dépendent du câblage et du backend. Les valeurs peuvent être récupérées depuis l’environnement avec `${env:VARIABLE}` ([Apache James – Configuration](https://james.apache.org/server/config.html)). C’est pratique pour les conteneurs, mais ne remplace pas une gestion des secrets : les certificats, clés privées, clés JWT et mots de passe de base de données devraient être fournis comme secrets montés ou via la plateforme d’orchestration.

### Couche protocolaire

Le projet Protocols fournit des implémentations de serveurs extensibles pour SMTP, LMTP, IMAP, POP3, ManageSieve et JMAP ([James Protocols](https://james.apache.org/server/feature-protocols.html)). Les écouteurs ne sont pas câblés de façon fixe à un stockage précis. IMAP et JMAP accèdent via la Mailbox API ; SMTP transmet les messages acceptés à la file d’attente et au conteneur de Mailets. Les protocoles peuvent ainsi être mis à l’échelle ou désactivés indépendamment de la topologie du backend.

### Boîte aux lettres, Mail Repository et stockage Blob

James distingue trois notions de stockage qui ne devraient pas être confondues en exploitation :

| Stockage | Contenu | Visibilité | Restauration typique |
|---|---|---|---|
| **Mailbox** | dossiers, messages, flags, UID, ACL et quotas d’un utilisateur | IMAP/JMAP/POP3 | restauration ou réplication du backend de boîtes aux lettres |
| **Mail Repository** | messages issus de chemins de traitement tels que `error`, `relay-denied` ou la quarantaine | administration uniquement | corriger la cause et retraiter le message |
| **Blob Store** | contenu MIME binaire ou objets volumineux | référencé indirectement par les métadonnées | sauvegarde cohérente avec métadonnées et références |

La [documentation sur la persistance](https://james.apache.org/server/feature-persistence.html) souligne qu’un Mail Repository n’est précisément **pas** la boîte aux lettres utilisateur. Cette séparation est précieuse pour la réponse aux incidents : un message défectueux peut être isolé, examiné et réinjecté dans le pipeline après correction, sans contourner le modèle de boîtes aux lettres.

### Bus d’événements, recherche et projections

Les opérations sur les boîtes aux lettres produisent des événements, par exemple `MailboxAdded`, `MessageMoveEvent`, `FlagsUpdated` ou des modifications de quota. Des listeners mettent à jour les quotas, les index de recherche et d’autres projections. Dans le profil distribué, RabbitMQ assure la communication, OpenSearch la recherche et Cassandra les métadonnées ; les contenus binaires sont stockés dans un Object Store compatible S3. Cette séparation permet une mise à l’échelle horizontale, mais crée une **cohérence éventuelle** entre la source et les projections. Les événements de listener échoués arrivent dans une Event Dead Letter et doivent être surveillés et, si nécessaire, redélivrés ([Distributed James – Mailbox Event Bus](https://james.apache.org/server/manage-guice-distributed-james.html#Mailbox_Event_Bus)).

### MIME, Sieve et authentification des expéditeurs

Le projet James englobe plus que le serveur. **Apache Mime4J** analyse les structures MIME en flux ou comme modèle objet ; **jSieve** implémente le langage de filtrage Sieve ; **jSPF** et **jDKIM** fournissent des bibliothèques Java pour la vérification d’expéditeurs ainsi que la signature et la vérification DKIM. Ces modules sont des projets autonomes et peuvent également être utilisés en dehors d’un serveur James complet ([Apache James – Composants](https://james.apache.org/documentation.html)).

La question de savoir quels composants fonctionnent sur un nœud ou sont répartis n’est pas une simple question de performance. Ce choix détermine aussi la cohérence, le redémarrage et le nombre de backends à surveiller.

## Variantes d’exploitation et mise à l’échelle

Pour James 3.9.0, Apache documente plusieurs profils. Il ne s’agit pas seulement d’installateurs différents, mais de modèles de cohérence, de mise à l’échelle et d’exploitation distincts. Dans cet état, la variante JPA est explicitement désignée comme **legacy** ; une distribution PostgreSQL et une distribution distribuée sont également proposées ([Apache James – Téléchargements](https://james.apache.org/download.cgi)). Les points indiqués dans le graphique comme **déduction d’exploitation** sont des recommandations dérivées et non des affirmations littérales du fabricant.

<iframe class="kb-infographic" style="aspect-ratio: 1280 / 956" src="/images/apache-james-betriebsmodelle.svg?v=20260813" title="Interaktive Infografik: Apache-James-Betriebsmodelle und Technologiestacks" loading="lazy">
  <a href="/images/apache-james-betriebsmodelle.svg?v=20260813">Ouvrir l’infographie comparant les modèles d’exploitation</a>
</iframe>

| Profil | Persistance et services | Adapté à | Conséquence opérationnelle |
|---|---|---|---|
| JPA/Guice (legacy) | base H2 embarquée ou base SQL externe ; modèle classique à serveur unique | laboratoire, migration d’installations anciennes, petites solutions spécifiques | peu de composants, mais trajectoire stratégique limitée et mise à l’échelle verticale |
| PostgreSQL | PostgreSQL comme cœur ; OpenSearch, RabbitMQ et stockage compatible S3 en option | nouvelles installations à un ou plusieurs nœuds sur base relationnelle | sauvegarde et HA bien connues ; introduire les services supplémentaires seulement en cas de besoin de mise à l’échelle |
| Distributed/Guice | Cassandra, RabbitMQ, OpenSearch et Object Store compatible S3 | services importants et horizontalement extensibles | plusieurs domaines de défaillance, projections, Dead Letters et contrôles de cohérence plus complexes |
| Memory | composants In-Memory volatils | tests et développement | aucune conservation des données de production |

La version 3.9 met en avant l’implémentation PostgreSQL performante comme une nouveauté majeure et la décrit comme capable de fonctionner de manière autonome tout en étant extensible avec RabbitMQ, OpenSearch et S3 ([Apache James 3.9.0](https://james.apache.org/james/update/2025/09/25/james-3.9.0.html)). Pour les nouvelles installations, c’est généralement le point de départ le plus compréhensible : commencer par la cohérence relationnelle et les méthodes de sauvegarde connues, puis ajouter des services uniquement pour des exigences concrètement mesurées.

## Modèle de sécurité

James fournit TLS, l’authentification SMTP, des contrôles de protocoles et des Mailets cryptographiques. Cela ne garantit toutefois pas automatiquement une exploitation de production sûre. Le chiffrement de transport protège un saut ; il ne remplace ni le chiffrement de bout en bout ni une vérification contraignante des destinataires. La [configuration TLS](https://james.apache.org/server/config-ssl-tls.html) distingue le keystore, les suites de chiffrement activées, StartTLS et TLS implicite pour chaque écouteur. Un changement de certificat doit donc être suivi séparément pour SMTP, IMAP, POP3 et HTTP.

**WebAdmin** mérite une attention particulière. L’API REST peut modifier les domaines, utilisateurs, boîtes aux lettres, files d’attente, repositories, quotas et tâches de maintenance. Selon la [documentation WebAdmin](https://james.apache.org/server/manage-webadmin.html), l’authentification JWT est désactivée par défaut ; sans protection supplémentaire, l’API ne doit donc jamais être accessible depuis un réseau non contrôlé. Les endpoints de santé et la documentation API peuvent en outre être volontairement hors authentification.

Un durcissement minimal de production comprend :

- lier WebAdmin à un réseau d’administration, activer JWT et limiter en plus l’accès via un pare-feu ou un reverse proxy ;
- empêcher les relais ouverts par des règles explicites de relais, d’authentification et de destinataires ;
- exploiter la soumission et le SMTP serveur-à-serveur sur des écouteurs séparés avec des politiques différentes ;
- supprimer les domaines de démonstration, utilisateurs d’exemple et mots de passe par défaut des images de conteneurs avant le premier démarrage externe ;
- gérer les clés privées hors de la couche de conteneur et surveiller les dates d’expiration ;
- traiter les Mailets personnalisés comme du code applicatif : vérifier les dépendances, exécuter les tests et limiter les droits d’exécution ;
- concevoir consciemment le contrôle antispam et antimalware. James est une plateforme ; les scanners externes et services de réputation sont intégrés via des Mailets ou des transferts de protocoles.

Pour le dépannage, le parcours du message est à nouveau contrôlé dans le même ordre : écouteur, file d’attente, pipeline de Mailets, repository, boîte aux lettres et livraison sortante.

## Exploitation et dépannage

Avec un serveur de messagerie modulaire, « le service fonctionne » n’est pas un état suffisant. Les Health Checks WebAdmin distinguent `healthy`, `degraded` et `unhealthy`; en mode strict, un seul composant dégradé entraîne déjà un HTTP 503. Selon le profil, les contrôles couvrent notamment JPA ou Cassandra, OpenSearch, RabbitMQ, le cycle de vie Guice, les Event Dead Letters et une livraison de test complète ([WebAdmin Health Checks](https://james.apache.org/server/manage-webadmin.html#HealthCheck)).

Pour le diagnostic, une approche par couches est plus efficace qu’une recherche globale dans les journaux :

1. **Connexion :** le client atteint-il le bon écouteur et TLS fonctionne-t-il avec le certificat et le nom d’hôte attendus ?
2. **Transaction SMTP :** quel code de réponse a été fourni pour `MAIL FROM`, `RCPT TO` et `DATA` ? Un `250` après `DATA` signifie une acceptation, pas nécessairement la livraison finale.
3. **File d’attente :** le nombre d’entrées en attente augmente-t-il, leur âge s’accroît-il ou la même erreur distante se répète-t-elle ?
4. **Pipeline de Mailets :** quel Processor et quelle paire Matcher/Mailet ont traité le message ? L’identifiant du mail sert de clé de corrélation.
5. **Repository :** le message se trouve-t-il dans `error`, `address-error`, `relay-denied` ou dans un repository propre ? Corriger d’abord la cause, puis retraiter.
6. **Boîte aux lettres et événements :** le message est-il présent dans le stockage Mailbox principal, mais absent de l’index de recherche ou de JMAP ? Dans ce cas, les listeners, Dead Letters et la réindexation sont plus pertinents que SMTP.
7. **Remote Delivery :** pour la livraison sortante, vérifier séparément DNS, routage, TLS, code du pair, plan de retry et génération de bounce.

Un contrôle synthétique compact peut relier le plan d’administration et le plan de données :

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für den Health Check">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$headers = @{ Authorization = "Bearer $env:JAMES_ADMIN_JWT" }
Invoke-RestMethod `
  -Uri "https://james-admin.example.net/healthcheck?strict" `
  -Headers $headers</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">curl --fail --silent \
  -H "Authorization: Bearer $JAMES_ADMIN_JWT" \
  "https://james-admin.example.net/healthcheck?strict"</code></pre>
  </div>
</div>

Sous Windows, [`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod) appelle l’endpoint REST ; sous Linux et Unix, [`curl`](https://curl.se/docs/manpage.html) effectue le même contrôle HTTP. Ces deux commandes testent ici exclusivement le Health Check WebAdmin documenté et ne remplacent aucune transaction SMTP ou Mailbox synthétique.

Il convient en outre d’alerter au minimum sur la profondeur et l’âge de la file d’attente, les repositories d’erreurs, les Event Dead Letters, le retard d’indexation OpenSearch, les latences de backend, les classes de réponses SMTP, la mémoire JVM et les durées de validité des certificats. Dans la variante distribuée, un processus James au vert alors que RabbitMQ ou OpenSearch est perturbé ne constitue qu’un succès partiel.

### Outils pour le poste de travail de l’administrateur

James fournit un client en ligne de commande pour les domaines, utilisateurs, boîtes aux lettres, mappings, quotas et la réindexation ; dans les conteneurs Guice, il est disponible sous le nom `james-cli` ([James CLI](https://james.apache.org/server/manage-cli.html)). Pour un diagnostic fiable, quelques outils neutres vis-à-vis des protocoles doivent également être présents sur le poste de travail de l’administrateur :

| Outil | Utilisation avec James |
|---|---|
| [`swaks`](https://www.jetmore.org/john/code/swaks/) | transaction SMTP et de soumission complète avec AUTH, TLS, enveloppe et en-têtes librement définis |
| [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) | vérifier la chaîne de certificats, SNI, le chiffrement et StartTLS sur SMTP, IMAP ou POP3 |
| [`curl`](https://curl.se/docs/manpage.html) et [`jq`](https://jqlang.org/manual/) | interroger automatiquement WebAdmin, Health Checks, tâches et métriques |
| [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) ou [`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) | contrôler MX, A/AAAA, PTR, SPF, DKIM et DMARC |
| [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) ou [Wireshark](https://www.wireshark.org/docs/wsug_html_chunked/) | distinguer handshake, retransmissions, interruptions de connexion et dialogues de protocoles |
| [Prometheus](https://prometheus.io/docs/introduction/overview/) et [Grafana](https://grafana.com/docs/grafana/latest/) | surveiller les métriques de file d’attente et de protocoles, les percentiles de latence, les temps d’exécution des Mailets/Matchers et les états des backends |
| [JMX](https://docs.oracle.com/en/java/javase/21/management/java-management-extensions-jmx-user-guide.html), [VisualVM](https://visualvm.github.io/documentation.html) et [`jcmd`](https://docs.oracle.com/en/java/javase/21/docs/specs/man/jcmd.html) | analyser le heap, les threads, le garbage collection et les métriques internes de la JVM |

La [documentation native des métriques](https://james.apache.org/server/metrics.html) répertorie notamment les connexions SMTP, IMAP et LMTP actives, les entrées de file d’attente, les messages envoyés et livrés, les temps de réponse par protocole ainsi que les durées d’exécution de Mailets et Matchers individuels. Ces métriques sont plus significatives qu’un simple uptime de processus, car elles représentent le parcours d’un message dans l’architecture.

## Histoire technique

James n’est pas né comme portage d’un MTA Unix existant. Les plus anciennes pages de projet conservées, datant de **1997/1998**, décrivent d’abord un serveur Java prévu, encore inutilisable, fondé sur des packages communs du Java Apache Project. Une interface de protocole commune, un stockage JDBC et une interface **MailServlet** inspirée des Servlets étaient prévus ; l’infrastructure de l’environnement Apache-JServ servait de travail préparatoire technique ([Archives James-1.0](https://svn.apache.org/repos/asf/james/server/tags/james_1_0/docs/index.html?p=1400000)). La Mailet API ultérieure a conservé l’idée fondamentale de petits composants de traitement déployables, sans devenir partie de la spécification Java Servlet.

| Période | Étape de développement technique |
|---|---|
| 1997–1998 | conception au sein du Java Apache Project : serveur Java pur, interfaces communes de protocoles et de ressources, idée MailServlet |
| Février 2001 | migration du Java Apache Project vers le projet Jakarta ([Jakarta News 2001](https://jakarta.apache.org/site/news/news-2001.html#20010311.1)) |
| James 1.x/2.x | serveur SMTP/POP3 stable, NNTP temporairement ; moteur de Mailets, stockage fichiers et RDBMS ; conteneur de composants Avalon/Phoenix ([Archives de documentation](https://james.apache.org/server/archive/document_archive.html)) |
| début des années 2000 | passage de sous-projet Jakarta à projet Top-Level autonome de l’Apache Software Foundation ([James 2.1.3 – page de projet archivée](https://svn.apache.org/repos/asf/james/server/tags/deprecated/build_2_2_0_RC1/www/index.html?p=1400000)) |
| 2010 | James 3.0 M1 avec prise en charge IMAP complète, SMTP/LMTP, Mailet API révisée et stockage Maildir, JPA et JCR ([Annonce de publication](https://james.apache.org/james/update/2010/11/05/james-3.0-M1.html)) |
| James 3.x | remplacement d’Avalon/Phoenix par Spring, puis orientation stratégique vers Guice ; développement d’IMAP, JMAP, administration REST et backends distribués |
| Septembre 2025 | James 3.9.0 : passage de `javax` à `jakarta`, Java 21 et nouvelle implémentation PostgreSQL ([Annonce de publication](https://james.apache.org/james/update/2025/09/25/james-3.9.0.html)) |

Le code source se trouve dans le dépôt officiel [apache/james-project](https://github.com/apache/james-project). La génération 3.9 considérée ici est principalement constituée de Java ; certains modules utilisent Scala. La compilation repose sur un vaste projet Maven multimodule. Cette longue histoire explique pourquoi plusieurs générations restent visibles dans la documentation et les installations : termes Phoenix et Spring dans les anciens textes, Guice dans la documentation 3.x, JPA comme voie legacy et profils PostgreSQL ou Cassandra pour les déploiements distribués.

## Adéquation et limites

James est particulièrement adapté lorsque l’e-mail constitue **une partie de l’application** et non seulement une infrastructure : traitement fondé sur des règles, Mailets propres, protocoles ouverts, JMAP, stockage maîtrisable ou mise à l’échelle horizontale sans cœur de serveur propriétaire. Les API publiques permettent de faire évoluer séparément le transport, les boîtes aux lettres et la logique métier.

James est moins adapté aux organisations qui attendent une appliance clé en main avec une interface graphique complète, une défense antispam et antimalware préconfigurée, des SLA éditeur et un objet de sauvegarde unique. La liberté modulaire génère du travail d’intégration. Le profil distribué exige en particulier une expérience d’exploitation avec plusieurs systèmes de données ainsi qu’une définition claire de la source, des projections, de la reconstruction et du point de reprise.

La question d’architecture décisive est donc la suivante : **faut-il exploiter l’e-mail comme système de protocoles configurable ou comme produit fini ?** Dans le premier cas, James fournit une boîte à outils ouverte d’une profondeur inhabituelle. Dans le second cas, un produit davantage préconfiguré est souvent plus économique.

## Sources

- [Apache Projects – James Committee](https://projects.apache.org/committee.html?james)
- [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)
- [Microsoft Learn – Invoke-RestMethod](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod)
- [curl – Manpage](https://curl.se/docs/manpage.html)
- [SWAKS – Swiss Army Knife for SMTP](https://www.jetmore.org/john/code/swaks/)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [jq – Manual](https://jqlang.org/manual/)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [tcpdump – Manpage](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [Wireshark – User’s Guide](https://www.wireshark.org/docs/wsug_html_chunked/)
- [Prometheus – Overview](https://prometheus.io/docs/introduction/overview/)
- [Grafana – Documentation](https://grafana.com/docs/grafana/latest/)
- [Oracle – JMX User Guide](https://docs.oracle.com/en/java/javase/21/management/java-management-extensions-jmx-user-guide.html)
- [VisualVM – Documentation](https://visualvm.github.io/documentation.html)
- [Oracle – jcmd](https://docs.oracle.com/en/java/javase/21/docs/specs/man/jcmd.html)
- [Apache James – Présentation du projet](https://james.apache.org/) – auto-description, JVM, protocoles, modules et objectifs d’architecture.
- [Apache James – Composants logiciels](https://james.apache.org/documentation.html) – projets serveur, Mailet, Mailbox, Protocols et sous-projets.
- [Apache James – Serveurs de protocoles](https://james.apache.org/server/feature-protocols.html) – services de protocoles pris en charge.
- [RFC 6409](https://datatracker.ietf.org/doc/html/rfc6409)
- [RFC 8314](https://datatracker.ietf.org/doc/html/rfc8314)
- [IETF: SMTP](https://datatracker.ietf.org/doc/html/rfc5321), [LMTP](https://datatracker.ietf.org/doc/html/rfc2033), [Message Submission](https://datatracker.ietf.org/doc/html/rfc6409), [IMAP4rev2](https://datatracker.ietf.org/doc/html/rfc9051), [POP3](https://datatracker.ietf.org/doc/html/rfc1939), [ManageSieve](https://datatracker.ietf.org/doc/html/rfc5804) et [JMAP Mail](https://datatracker.ietf.org/doc/html/rfc8621) – normes de protocoles normatives.
- [RFC 2033](https://datatracker.ietf.org/doc/html/rfc2033)
- [RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051)
- [RFC 1939](https://datatracker.ietf.org/doc/html/rfc1939)
- [RFC 5804](https://datatracker.ietf.org/doc/html/rfc5804)
- [RFC 8621](https://datatracker.ietf.org/doc/html/rfc8621)
- [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) – ports enregistrés.
- [Mailbox API](https://james.apache.org/mailbox/mailbox-api.html)
- [Apache James – Managing Distributed James](https://james.apache.org/server/manage-guice-distributed-james.html) – Cassandra, S3, OpenSearch, RabbitMQ, bus d’événements et exploitation.
- [Apache James – Mailet Container](https://james.apache.org/server/feature-mailetcontainer.html) – Matchers, Mailets, Processors, Spooler et division par destinataire.
- [Apache James – Mailet Container Configuration](https://james.apache.org/server/config-mailetcontainer.html) – configuration et gestion des erreurs du pipeline.
- [Apache James – Configuration](https://james.apache.org/server/config.html) – répertoire de configuration, fichiers et variables d’environnement.
- [Apache James – Persistence](https://james.apache.org/server/feature-persistence.html) – distinction entre Mailbox et Mail Repository.
- [Apache James – Downloads](https://james.apache.org/download.cgi) – profils de serveur et téléchargements officiels.
- [Apache James Server 3.9.0](https://james.apache.org/james/update/2025/09/25/james-3.9.0.html) – Java 21, transition Jakarta et implémentation PostgreSQL.
- [Apache James – SSL/TLS Configuration](https://james.apache.org/server/config-ssl-tls.html) – modes TLS et configuration des écouteurs.
- [Apache James – WebAdmin](https://james.apache.org/server/manage-webadmin.html) – administration REST, indication JWT et Health Checks.
- [Apache James – Command Line](https://james.apache.org/server/manage-cli.html) – CLI pour domaines, utilisateurs, boîtes aux lettres, mappings, quotas et réindexation.
- [Apache James – Metrics](https://james.apache.org/server/metrics.html) – Prometheus, JMX et métriques d’exploitation disponibles.
- [Archives James-1.0 du Java Apache Project](https://svn.apache.org/repos/asf/james/server/tags/james_1_0/docs/index.html?p=1400000) – premières planifications d’architecture et de MailServlet.
- [Jakarta Project News 2001](https://jakarta.apache.org/site/news/news-2001.html) – migration du projet James vers Jakarta.
- [Apache James Document Archive](https://james.apache.org/server/archive/document_archive.html) – documentation des versions 1.x et 2.x.
- [James 2.1.3 – page de projet archivée](https://svn.apache.org/repos/asf/james/server/tags/deprecated/build_2_2_0_RC1/www/index.html?p=1400000)
- [Apache James 3.0 M1](https://james.apache.org/james/update/2010/11/05/james-3.0-M1.html) – IMAP, profils de stockage et Mailet API de la génération 3.x.
- [Apache James – Dépôt GitHub](https://github.com/apache/james-project) – code source, build et structure des modules.
