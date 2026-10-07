---
title: "Home Assistant : architecture, modèle de données et exploitation"
blatt: "home-assistant"
description: "Home Assistant pour les administrateurs de plateformes, réseaux et IoT : Core Python événementiel, intégrations et registres, Home Assistant OS et conteneur, passerelles de protocoles et radio, exécution des automatisations, Recorder, API, authentification, observabilité, sauvegarde et reprise."
fakten:
  - label: Rôle système
    wert: plateforme centrale de contrôle et d’automatisation événementielle pour les appareils locaux, les réseaux radio et les services externes
    href: https://developers.home-assistant.io/docs/architecture_index/
  - label: Core
    wert: Event Bus, State Machine, Service Registry et Timer constituent le cœur d’exécution
    href: https://developers.home-assistant.io/docs/architecture/core/
  - label: Pile technologique
    wert: Home Assistant Core et ses intégrations sont implémentés en Python ; asyncio assure le traitement I/O concurrent
    href: https://github.com/home-assistant/core
  - label: Modèle d’extension
    wert: les intégrations se composent d’une logique de domaine et de plateformes ; les Config Entries contrôlent leur cycle de vie persistant
    href: https://developers.home-assistant.io/docs/architecture_components/
  - label: Modèle d’objet
    wert: Config Entry → appareil → entité → état ; les registres stabilisent les identités, les noms et les attributions
    href: https://developers.home-assistant.io/docs/architecture/devices-and-services/
  - label: Installation prise en charge
    wert: Home Assistant OS comme appliance gérée ou Home Assistant Container sur un hôte exploité en propre
    href: https://www.home-assistant.io/faq/ha-vs-hassio/
  - label: Pile HAOS
    wert: Buildroot, Linux, systemd, Docker, Supervisor, Core et Apps ; RAUC met à jour le système d’exploitation
    href: https://developers.home-assistant.io/docs/operating-system/
  - label: Interfaces
    wert: REST via /api et WebSocket via /api/websocket sur le même point de terminaison HTTP que le frontend
    href: https://developers.home-assistant.io/docs/api/rest/
  - label: Port standard
    wert: TCP 8123 pour le frontend, REST et WebSocket ; TLS ou un reverse proxy modifient le chemin d’accès externe
    href: https://www.home-assistant.io/integrations/http/
  - label: Historique
    wert: Recorder écrit les états et certains événements via SQLAlchemy dans SQLite par défaut ; MariaDB, MySQL et PostgreSQL sont pris en charge
    href: https://www.home-assistant.io/integrations/recorder/
  - label: Automatisations
    wert: les triggers démarrent une exécution, les conditions décident et les actions utilisent la même sémantique de séquence que les scripts
    href: https://www.home-assistant.io/docs/automation/basics/
  - label: Reprise
    wert: des sauvegardes chiffrées peuvent restaurer la configuration, le Core et les Apps ; les clés, contrôleurs radio et bases de données externes restent des dépendances distinctes
    href: https://www.home-assistant.io/common-tasks/general/
werbung:
  - newsletter
ctaThemen:
  - smart-home-iot
translationSourceHash: 204801ccce2af55eaa473c7a7599bd0744a8e9db9b0e1dbdb99efe8949a6fff1
translationModel: gpt-5.6-terra
translatedAt: 2026-10-07T10:48:29.535Z
translationReview: automatic
---

# Home Assistant : architecture, modèle de données et exploitation

Home Assistant est une plateforme centrale de contrôle et d’automatisation pour les appareils, réseaux radio, services IP et interfaces utilisateur. L’instance collecte les états via des intégrations, les normalise en entités, distribue les changements via un Event Bus et exécute des actions à partir de ceux-ci. « Local » désigne ici une préférence architecturale, et non une propriété générale de chaque intégration : une ampoule Zigbee peut être entièrement accessible localement, tandis qu’une intégration constructeur peut obtenir ses états exclusivement depuis une API cloud. La [vue d’ensemble officielle de l’architecture](https://developers.home-assistant.io/docs/architecture_index/) distingue système d’exploitation, Supervisor et Core ; l’[architecture des intégrations](https://developers.home-assistant.io/docs/architecture_components/) décrit l’extension du Core par des composants Python.

Pour les administrateurs, Home Assistant n’est donc ni seulement un tableau de bord ni un convertisseur universel de protocoles. C’est un orchestrateur avec état comportant plusieurs points de défaillance possibles : environnement d’exécution Python, intégrations, registres, base de données, authentification, réseaux locaux, contrôleurs radio, brokers, clouds des constructeurs et, le cas échéant, apps du Supervisor. Une interface verte prouve uniquement que le chemin du frontend fonctionne. Elle ne prouve pas que les événements arrivent à temps, que les appareils sont accessibles, que les automatisations s’exécutent de façon déterministe ou qu’une sauvegarde, y compris les dépendances externes, peut être restaurée.

L’explication suit un événement d’appareil à travers l’intégration, l’Event Bus et la State Machine jusqu’à l’automatisation et l’action. Elle situe ensuite persistance, add-ons, sécurité, surveillance et restauration.

## Approche architecturale : nœud central d’événements et d’états

Home Assistant Core est événementiel. Quatre composants documentés forment le cœur ([Core architecture](https://developers.home-assistant.io/docs/architecture/core/)) :

1. L’**Event Bus** distribue les événements aux écouteurs enregistrés.
2. La **State Machine** conserve le dernier état connu de chaque entité chargée et publie `state_changed`.
3. La **Service Registry** gère les actions appelables et traite les appels de services.
4. Le **Timer** génère des événements temporels pour le traitement dépendant du temps.

Les intégrations traduisent les états d’appareils ou de services dans ce modèle. Une intégration peut interroger périodiquement, recevoir des événements push, utiliser des bibliothèques locales ou appeler une API distante. Home Assistant unifie l’état résultant, pas le transport. C’est la principale frontière d’exploitation : deux entités ayant le même type de domaine, par exemple `light`, peuvent avoir des chemins de latence, d’authentification et de reprise totalement différents.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1116" src="/images/kb-interaktiv-home-assistant.svg?v=20260813" title="Interaktive Infografik: Home Assistant von Geräten und Protokollbrücken über Integrationen, Registries, Event Bus, State Machine, Automationen und Recorder bis zu APIs, Supervisor, Monitoring und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-home-assistant.svg?v=20260813">Ouvrir directement le graphique interactif</a>.
</iframe>

## Couches d’exécution et modèles d’installation

Home Assistant propose deux modèles d’installation pris en charge. **Home Assistant OS** est une appliance gérée. **Home Assistant Container** exécute Home Assistant Core dans un conteneur sur un hôte dont l’opérateur est responsable. La comparaison officielle recommande HAOS pour presque toutes les installations et décrit Container comme une installation autonome du Core sans apps du Supervisor ([HAOS ou Container](https://www.home-assistant.io/faq/ha-vs-hassio/)).

### Home Assistant OS

HAOS est construit avec Buildroot et comprend Linux, GNU C Library, systemd et Docker. SquashFS porte les zones système en lecture seule, ZRAM les systèmes de fichiers temporaires et le swap, AppArmor limite les processus et RAUC met à jour le système d’exploitation ([Home Assistant Operating System](https://developers.home-assistant.io/docs/operating-system/)). Au-dessus, le **Supervisor** gère Core, Apps, DNS, audio, mDNS, sauvegardes et mises à jour ([Supervisor](https://developers.home-assistant.io/docs/supervisor/)).

Le modèle d’appliance réduit les variantes, mais confie au Supervisor de vastes responsabilités. Une erreur peut se situer à au moins cinq niveaux : slot de démarrage/OS, Docker Engine, Supervisor, conteneur Core ou app individuelle. Le Supervisor peut revenir en arrière après un chemin de mise à jour du Core échoué ; il ne détecte toutefois pas automatiquement un comportement erroné au niveau fonctionnel d’un appareil ou de la base de données.

### Home Assistant Container

Container ne fournit que le Core. Le système d’exploitation hôte, le moteur de conteneurs, le réseau, les volumes, la base de données, le broker, le serveur radio, le reverse proxy, la sauvegarde et les mises à jour relèvent de l’opérateur. Les apps du Supervisor sont des services empaquetés séparément. Dans une conception avec conteneurs, Mosquitto, Matter Server, Zigbee2MQTT, Z-Wave JS UI, PostgreSQL ou un reverse proxy sont exploités comme charges de travail distinctes avec leurs propres volumes, versions et contrôles de santé.

L’avantage réside dans une architecture de plateforme explicite ; le prix est une surface d’exploitation plus grande. Une sauvegarde du volume Core ne contient par exemple ni la base de données Recorder externe, ni l’état du broker, la NVM radio ou les clés du reverse proxy s’ils sont externes.

### Formes d’installation historiques

L’ancienne installation **Core** dans un environnement Python et l’installation **Supervised** sur un Linux autogéré ont été abandonnées en 2025. Depuis la version 2025.12, elles ne sont plus prises en charge ; les architectures 32 bits `i386`, `armhf` et `armv7` ont simultanément perdu leur voie de publication. L’annonce du projet désigne HAOS et Container comme les modèles restants ([Abandon de Core et Supervised](https://www.home-assistant.io/blog/2025/05/22/deprecating-core-and-supervised-installation-methods-and-32-bit-systems/)). Un nom d’installation historique ne doit donc pas être confondu avec le composant logiciel **Home Assistant Core**, qui continue de fonctionner aussi dans HAOS et Container.

## Pile technologique

Home Assistant Core est une application Python sous licence Apache-2.0. Le [dépôt Core officiel](https://github.com/home-assistant/core) montre Python, asyncio et la structure modulaire des intégrations. Les intégrations intensives en I/O ne doivent pas bloquer : les règles de qualité privilégient les dépendances asynchrones afin que les appels réseau et aux appareils ne bloquent pas la boucle d’événements commune ([async dependency](https://developers.home-assistant.io/docs/core/integration-quality-scale/rules/async-dependency/)). Le code de bibliothèque bloquant est délégué à des threads d’exécuteur ; le travail intensif en CPU ou insuffisamment borné reste néanmoins un risque de capacité et de latence.

La pile visible comprend plus que Python :

| Couche | Technologie typique | Importance opérationnelle |
|---|---|---|
| Frontend | Application navigateur, HTTP et WebSocket | Chemin utilisateur et temps réel |
| Core | Python, asyncio, intégrations | États, événements, actions, authentification |
| Persistance | Stockages de configuration basés sur JSON, YAML, SQLAlchemy/SQL | Configuration, registres, historique |
| HAOS | Buildroot, Linux, systemd, Docker, AppArmor, RAUC | Cycle de vie de l’appliance et isolation |
| Services | Apps du Supervisor ou conteneurs/hôtes externes | MQTT, Matter, base de données, proxy, partages de fichiers |
| Edge | Contrôleurs radio, passerelles de protocoles, API d’appareils et cloud | Accessibilité physique et provenance des données |

L’[Integration Quality Scale](https://developers.home-assistant.io/docs/core/integration-quality-scale/) évalue les intégrations selon le flux de configuration, les tests, le typage, les diagnostics, l’utilisation efficace des données et le comportement asynchrone. Un niveau élevé améliore la maintenabilité attendue, mais ne constitue pas un SLA de disponibilité pour l’appareil ou le fournisseur cloud sous-jacent.

Après le modèle d’installation et l’exécution vient le modèle de données. Ce n’est qu’en distinguant Config Entry, appareil, entité et état que les entités dupliquées, les appareils manquants et les automatisations erronées peuvent être expliqués proprement.

## Modèle d’objet : Config Entry, appareil, entité et état

La clé d’inventaire opérationnel n’est pas la tuile visible, mais la chaîne de configuration, d’identité de l’appareil et d’entité.

### Config Entries

Une **Config Entry** stocke la configuration persistante d’une instance d’intégration. Un flux de configuration dans l’UI la crée ; les options, Reconfigure, Reload, Unload, Removal et Migration sont des opérations de cycle de vie définies. Les intégrations ne doivent pas modifier directement les données de l’entrée, mais utiliser le gestionnaire de Config Entries ([Config entries](https://developers.home-assistant.io/docs/config_entries_index/)). Une erreur d’authentification, une entrée non chargée et un homologue inaccessible sont donc des états distincts.

### Appareils et registres

Le **Device Registry** regroupe les points de terminaison techniques en appareils. Les identifiants ou connexions, par exemple numéro de série et adresse MAC, servent à la correspondance ; `via_device` peut représenter une relation de pont ou de parenté ([Device registry](https://developers.home-assistant.io/docs/device_registry_index/)). Un capteur Zigbee peut ainsi apparaître comme appareil connecté via un coordinateur, sans que le coordinateur constitue son état applicatif.

L’**Entity Registry** attribue aux entités une identité durable avec `unique_id` et empêche les collisions d’Entity IDs. L’adresse IP, le nom d’hôte, l’URL, le nom d’utilisateur ou l’adresse e-mail ne sont explicitement pas considérés comme des Unique IDs stables ([Entity registry](https://developers.home-assistant.io/docs/entity_registry_index/)). Cela explique pourquoi le renommage manuel d’un hôte ne doit pas remplacer l’identité de l’appareil et pourquoi les migrations d’intégrations nécessitent des identifiants constructeur stables.

### Entité et état

Une **entité** représente une fonction ou une grandeur mesurée : `sensor`, `switch`, `light`, `climate`, `binary_sensor` ou un autre domaine. Son état comprend un State principal, des attributs, des heures de modification et un Context. La State Machine ne conserve que le dernier état connu. `unavailable` signifie que l’entité n’est actuellement pas alimentée par un objet d’entité actif ; `unknown` signifie qu’aucune valeur exploitable n’est disponible. La « dernière valeur » n’est donc pas automatiquement une « mesure récente ».

L’[interaction documentée des appareils et services](https://developers.home-assistant.io/docs/architecture/devices-and-services/) distingue Entity Integration, Entity Component, Entity Platform et intégration spécifique au constructeur. Le diagnostic pose donc toujours les questions suivantes :

- Quelle Config Entry possède l’entité ?
- Par quelle intégration et quelle plateforme est-elle créée ?
- Quelle Device ID et quelle Entity ID stables relient l’historique et la configuration ?
- Les données sont-elles interrogées ou poussées ?
- Quel modèle de temps et de disponibilité possède la valeur source ?
- Quelle passerelle, bibliothèque, API cloud ou liaison radio se trouve en amont ?

## Intégrations et isolation des erreurs

Une intégration définit un domaine et peut fournir des plateformes telles que `sensor`, `light` ou `switch`. La plateforme abstrait le type d’entité ; l’intégration d’appareil communique avec le protocole concret. Les intégrations intégrées sont livrées avec le Core et testées par son processus de publication. Les **Custom Integrations**, en revanche, s’exécutent dans le même processus Python et peuvent affecter les imports, l’Event Loop, le temps de démarrage ou la consommation mémoire. Le répertoire `/config/custom_components` fait donc partie de l’inventaire, de la gestion des changements et de la reprise.

La [vue d’ensemble officielle des intégrations](https://www.home-assistant.io/integrations/) distingue notamment des classes IoT telles que Local Push, Local Polling, Cloud Push et Cloud Polling. Cette classification est plus utile pour les modèles d’exploitation qu’une longue liste de constructeurs :

| Classe | Chemin de données | Domaine de défaillance typique |
|---|---|---|
| Local Push | L’appareil ou la passerelle envoie dans le LAN | Multicast, pare-feu, passerelle, sous-réseau |
| Local Polling | Le Core interroge l’appareil local | Latence, timeout, intervalle d’interrogation, capacité de l’appareil |
| Cloud Push | Le cloud envoie ou diffuse des événements | Internet, compte, token, flux du fournisseur |
| Cloud Polling | Le Core interroge l’API du fournisseur | Rate limit, token, Internet, modifications de l’API |
| Calculated/Internal | Le Core calcule l’état | Données d’entrée, templates, temps, état après redémarrage |

L’intégration n’est pas un isolateur de processus. Une bonne délimitation des erreurs désactive ou recharge de manière ciblée la Config Entry concernée avant de redémarrer l’ensemble du Core. Un redémarrage détruit des éléments de preuve volatils et peut réinitialiser les temporisateurs d’automatisation dépendants du temps.

## Modèle de protocoles et de réseau

Pour Home Assistant, un **graphe de dépendances** est plus utile qu’une table OSI générale. La plateforme se trouve au niveau applicatif, mais ses chemins de données se ramifient :

- Frontend, REST et WebSocket fonctionnent sur HTTP via TCP, par défaut sur le port 8123.
- DNS résout les hôtes et services cloud ; mDNS et SSDP découvrent les appareils sur le réseau local.
- MQTT utilise un broker distinct et un modèle publication/abonnement sur TCP ou WebSocket.
- Zigbee, Z-Wave, Thread et Bluetooth nécessitent des contrôleurs radio ou des proxys réseau.
- Matter utilise la communication IP, mais requiert un serveur Matter et, le cas échéant, un Thread Border Router pour le provisioning et l’exploitation de la fabric.
- Les intégrations constructeur peuvent utiliser HTTPS, des protocoles locaux propriétaires ou des flux cloud.

Les intégrations Discovery intégrées documentent [mDNS/Zeroconf](https://www.home-assistant.io/integrations/zeroconf/) et [SSDP](https://www.home-assistant.io/integrations/ssdp/). Les deux dépendent du segment et du multicast. Un reverse proxy pour le frontend ne répare pas la découverte au-delà des frontières VLAN. Multicast relay, IGMP snooping, isolation des clients WLAN, IPv6 RA, suffixes DNS et règles de pare-feu doivent être vérifiés pour chaque chemin réel d’appareil.

### MQTT comme espace d’état distinct

MQTT n’est pas l’Event Bus interne. C’est un service de broker externe avec lequel une intégration communique. L’[intégration MQTT officielle](https://www.home-assistant.io/integrations/mqtt/) décrit les Discovery Topics, retained messages, Birth/Last Will, Availability, TLS et MQTT 5. Une Discovery retenue peut recréer des appareils après un redémarrage, mais peut aussi conserver des Ghost Entities obsolètes. La disponibilité nécessite sa propre sémantique ; un State retenu présent ne prouve pas que le publisher vit encore.

Une exploitation MQTT robuste inventorie le broker, les Client IDs, l’authentification, la CA, les Topics, QoS, Retain, Expiry, Birth/Will et l’origine de Discovery. La sauvegarde du broker et celle du Core sont des objets de protection distincts.

### Chemins radio et passerelles

[ZHA](https://www.home-assistant.io/integrations/zha/) intègre un coordinateur Zigbee, [Z-Wave JS](https://www.home-assistant.io/integrations/zwave_js/) utilise un serveur Z-Wave JS séparé et [Matter](https://www.home-assistant.io/integrations/matter/) connecte un serveur Matter. [Thread](https://www.home-assistant.io/integrations/thread/) gère les références aux Border Routers et aux réseaux, mais n’est pas identique à Matter. Les appareils radio, le firmware des contrôleurs, les données réseau, le matériel de clés et la configuration des appareils forment chacun un ensemble de reprise. Déplacer une clé USB ou remplacer un coordinateur n’est pas un simple changement d’adresse IP.

Les intégrations fournissent des états et événements ; les automatisations y réagissent. Leur déroulement, composé de trigger, conditions et actions, doit donc être diagnostiqué séparément de la configuration des appareils.

## Exécution des automatisations : Trigger, Condition, Action

Une automatisation est une définition d’exécution réactive. Les [bases des automatisations](https://www.home-assistant.io/docs/automation/basics/) distinguent les triggers, les Conditions facultatives et les Actions. Le trigger crée une exécution, les Conditions vérifient l’état à l’entrée, les Actions utilisent la sémantique de séquence des Scripts ([Actions](https://www.home-assistant.io/docs/automation/action/)).

Le moment est important : State, attributs et valeurs de template peuvent changer entre le trigger et une Action ultérieure. Un délai ne maintient pas une transaction ouverte. Plusieurs exécutions de la même automatisation nécessitent donc un mode tel que Single, Restart, Queued ou Parallel et un modèle de conflit conscient. Les actionneurs physiques sont rarement transactionnels ; une exécution partiellement réalisée nécessite éventuellement des actions compensatoires.

La [documentation des triggers](https://www.home-assistant.io/docs/automation/trigger/) indique que les temps d’attente `for` ne survivent pas à un redémarrage ou au rechargement de l’automatisation. Si une échéance doit survivre aux redémarrages, un instant doit être persisté, par exemple dans `input_datetime`, puis utilisé comme déclencheur. Les Conditions ne sont que des vérifications dans l’exécution en cours ; la [sémantique des Conditions](https://www.home-assistant.io/docs/scripts/conditions/) n’en fait pas un verrou contre les modifications parallèles.

Les templates sont évalués dans Home Assistant avec des expressions Jinja. Les erreurs d’entrée et de type, `unknown`, `unavailable`, les fuseaux horaires et la conversion implicite de chaînes doivent faire partie des tests. La [documentation sur le templating](https://www.home-assistant.io/docs/automation/templating/) décrit les variables dépendantes du trigger. Un administrateur ne teste pas seulement le happy path, mais aussi le redémarrage, l’entité manquante, l’événement tardif, le trigger en double et l’erreur d’actionneur.

## Configuration, registres et source de vérité

Home Assistant combine Config Entries pilotées par l’UI, données de registre et YAML. `configuration.yaml` est la racine de la configuration manuelle, mais pas la source de vérité complète. La [vue d’ensemble officielle de la configuration](https://www.home-assistant.io/docs/configuration/) distingue UI et YAML ; les Packages peuvent structurer des blocs YAML associés ([Packages](https://www.home-assistant.io/docs/configuration/packages/)).

Seule la partie textuelle débarrassée des secrets convient à Git et à la revue. `secrets.yaml` sépare les valeurs du YAML, mais ne les chiffre pas ; le [guide de durcissement](https://www.home-assistant.io/docs/configuration/securing/) le souligne expressément. L’état de l’UI, les registres, tokens et Config Entries sont conservés dans le stockage de configuration et modifiés par les voies UI/API prises en charge. La modification directe de fichiers de stockage internes pendant l’exécution du Core contourne la logique de schéma, de cycle de vie et de cohérence.

Un inventaire de configuration comprend :

- YAML, Packages, Blueprints et Custom Components,
- Config Entries avec leur origine, propriétaire et authentification,
- attributions Device, Entity et Area,
- automatisations, Scripts, scènes et tableaux de bord,
- utilisateurs, tokens, MFA et fournisseurs d’identité externes,
- Apps du Supervisor ou services externes,
- contrôleurs radio, broker, base de données et proxy,
- secrets, certificats et clés de reprise.

## Recorder, historique et statistiques à long terme

La State Machine conserve l’état actuel en mémoire. L’historique n’est créé que par le **Recorder**. Il écrit les changements d’état et certains événements dans une base de données via SQLAlchemy ; History, Activity, graphiques et statistiques à long terme les lisent depuis celle-ci. La [documentation officielle du Recorder](https://www.home-assistant.io/integrations/recorder/) cite SQLite comme valeur par défaut et recommandation, ainsi que MariaDB, MySQL et PostgreSQL comme alternatives prises en charge.

Les données du Recorder ne sont pas une source d’événements pour la commande en temps réel. Une base de données indisponible peut affecter l’historique et les statistiques tandis que les états actuels et les automatisations continuent parfois de fonctionner. Inversement, un historique complet ne prouve pas qu’une Action ait réussi sur l’appareil physique.

Les principaux paramètres opérationnels sont :

- `purge_keep_days` pour l’historique brut,
- filtres Include/Exclude pour les entités et événements,
- `commit_interval` comme rapport entre I/O et fenêtre de perte,
- taille de la base de données, espace libre et latence d’écriture,
- Purge et Repack,
- ordre de démarrage et accessibilité des bases de données externes,
- statistiques à long terme et cohérence des métadonnées.

Un changement de base de données Recorder ne migre pas l’historique existant de manière prise en charge. Les bases de données externes exigent leurs propres sauvegardes cohérentes et tests de restauration. Pour SQLite, la documentation indique un espace libre d’au moins 2,5 fois la taille de la base de données afin de traiter la corruption. Le stockage et le Recorder constituent ainsi un chemin distinct de capacité et de reprise, et non un simple cache facultatif.

## API, WebSocket et authentification

Le frontend et les API partagent par défaut le même listener HTTP. L’[API REST](https://developers.home-assistant.io/docs/api/rest/) utilise JSON et des Bearer Tokens ; le chemin de base est `/api/`. L’[API WebSocket](https://developers.home-assistant.io/docs/api/websocket/) se trouve sous `/api/websocket`, passe par `auth_required`, `auth` et `auth_ok`, et corrèle les commandes au moyen d’IDs numériques. WebSocket fournit les flux d’événements et les registres plus efficacement que les interrogations REST répétées.

Les tokens longue durée sont des identifiants utilisateur. L’[Authentication API](https://developers.home-assistant.io/docs/auth_api/) décrit OAuth/IndieAuth, Refresh Tokens, Long-Lived Access Tokens et Signed Paths de courte durée. Un token hérite du contexte de son utilisateur ; un Long-Lived Token valable dix ans doit être conservé dans un magasin de secrets, et non dans YAML, l’historique du shell, une URL ou le JavaScript d’un tableau de bord.

Un moniteur API vérifie au minimum l’authentification, `/api/config`, les entités attendues, `last_updated`, un abonnement WebSocket et un chemin de lecture/action inoffensif. Un HTTP 200 sur `/` ne vérifie que l’accessibilité du frontend.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Home-Assistant-API-Inventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$headers = @{ Authorization = "Bearer $env:HA_TOKEN" }
Invoke-RestMethod -Headers $headers -Uri "https://ha.example.net/api/config"
Invoke-RestMethod -Headers $headers -Uri "https://ha.example.net/api/states/sensor.uptime" |
  ConvertTo-Json -Depth 8</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">curl --fail --silent --show-error \
  -H "Authorization: Bearer $HA_TOKEN" \
  https://ha.example.net/api/config | jq .
curl --fail --silent --show-error \
  -H "Authorization: Bearer $HA_TOKEN" \
  https://ha.example.net/api/states/sensor.uptime | jq .</code></pre>
  </div>
</div>

[`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod) et [`ConvertTo-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertto-json) traitent l’interrogation Windows ; [`curl`](https://curl.se/docs/manpage.html) et [`jq`](https://jqlang.org/manual/) font de même sous Unix. Le token n’est montré que comme variable d’environnement du processus ; en production, il provient d’un magasin de secrets contrôlé.

## HTTP, TLS et reverse proxy

Le point de terminaison HTTP écoute par défaut sur TCP 8123. TLS direct, reverse proxy et Home Assistant Cloud sont des modèles d’accès différents. Avec un reverse proxy traditionnel, `use_x_forwarded_for` et `trusted_proxies` doivent être définis de manière appropriée ; sinon, l’IP client est erronée ou la requête est refusée ([HTTP integration](https://www.home-assistant.io/integrations/http/)). Une liste de confiance de proxys trop large permet de falsifier les informations Forwarded-For.

Le [guide de sécurité](https://www.home-assistant.io/docs/configuration/securing/) recommande des mots de passe uniques, MFA, des privilèges administratifs minimaux et un accès distant sécurisé plutôt qu’une exposition directe à Internet. TLS ne sécurise que le transport. Les droits des tokens, en-têtes proxy, mises à niveau WebSocket, rate limits, DNS, renouvellement des certificats et sécurité de l’IdP en amont restent des contrôles distincts.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für DNS-, TCP- und TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName ha.example.net
Test-NetConnection ha.example.net -Port 443
curl.exe -sS -D - -o NUL https://ha.example.net/api/
Get-NetTCPConnection -State Established | Where-Object RemotePort -eq 443</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig +short ha.example.net A ha.example.net AAAA
curl -sS -D - -o /dev/null https://ha.example.net/api/
ss -ntp state established '( dport = :443 )'
openssl s_client -connect ha.example.net:443 -servername ha.example.net -brief</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname), [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) et [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) vérifient Windows ; [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility), [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) et [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) vérifient Unix. L’appel non authentifié à `/api/` peut renvoyer 401 ; la résolution de noms, l’identité TLS, la route proxy et la limite d’authentification attendue sont déterminantes.

## Diagnostic MQTT

L’état du broker est vérifié en dehors de Home Assistant. Un subscriber observe Discovery, Availability et State sans modifier les Topics. Un test de publication utilise un chemin de test spécifiquement réservé ; les Command Topics de production ne sont pas utilisés de manière incidente.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für MQTT-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">mosquitto_sub.exe -h mqtt.example.net -p 8883 --cafile .\ca.pem `
  -u ha-observer -P $env:MQTT_PASSWORD -v -t "homeassistant/#"
mosquitto_pub.exe -h mqtt.example.net -p 8883 --cafile .\ca.pem `
  -u ha-probe -P $env:MQTT_PASSWORD -t "ops/probe" -m "online" -q 1</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">mosquitto_sub -h mqtt.example.net -p 8883 --cafile ./ca.pem \
  -u ha-observer -P "$MQTT_PASSWORD" -v -t 'homeassistant/#'
mosquitto_pub -h mqtt.example.net -p 8883 --cafile ./ca.pem \
  -u ha-probe -P "$MQTT_PASSWORD" -t 'ops/probe' -m 'online' -q 1</code></pre>
  </div>
</div>

[`mosquitto_sub`](https://mosquitto.org/man/mosquitto_sub-1.html) et [`mosquitto_pub`](https://mosquitto.org/man/mosquitto_pub-1.html) sont les clients officiels du broker. Les mots de passe sur la ligne de commande peuvent apparaître dans les listes de processus ou l’historique ; les exemples illustrent le chemin, tandis que l’appel de production utilise un fichier de mots de passe, un magasin de secrets du système d’exploitation ou des identifiants de courte durée.

## Exploitation de HAOS et Container

HAOS fournit la commande `ha` via l’accès Terminal/SSH. Les installations Container sont exploitées avec les outils du runtime choisi. Un paquet de diagnostic regroupe les informations système, le journal Core, les diagnostics d’intégration, l’état des conteneurs, l’espace libre et l’horodatage.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Home-Assistant-Laufzeitdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker inspect homeassistant | ConvertFrom-Json
docker logs --since 30m --timestamps homeassistant 2&gt;&amp;1 |
  Select-String -Pattern 'ERROR|WARNING|unavailable|timeout'
docker stats --no-stream homeassistant</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker inspect homeassistant | jq '.[0].State, .[0].Mounts, .[0].NetworkSettings.Networks'
docker logs --since 30m --timestamps homeassistant 2&gt;&amp;1 | grep -E 'ERROR|WARNING|unavailable|timeout'
docker stats --no-stream homeassistant
df -h /path/to/config &amp;&amp; du -sh /path/to/config</code></pre>
  </div>
</div>

[`docker inspect`](https://docs.docker.com/reference/cli/docker/inspect/), [`docker logs`](https://docs.docker.com/reference/cli/docker/container/logs/) et [`docker stats`](https://docs.docker.com/reference/cli/docker/container/stats/) fournissent l’état des conteneurs. [`ConvertFrom-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json) et [`Select-String`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/select-string) traitent les sorties Windows ; [`grep`](https://www.gnu.org/software/grep/manual/grep.html), [`df`](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html) et [`du`](https://www.gnu.org/software/coreutils/manual/html_node/du-invocation.html) complètent Unix. Un conteneur en cours d’exécution n’est que la première vérification ; viennent ensuite les chemins d’intégration, de registre, d’événement et d’appareil.

Pour le dépannage, le chemin de signal est lu à rebours : action, trace d’automatisation, changement d’état, intégration, protocole réseau et appareil physique.

## Observabilité et diagnostic systématique

**System Health** collecte le type d’installation, l’architecture et les informations Python, Core et frontend, et fournit des fonctions de diagnostic via Paramètres > Système > Réparations ([System Health](https://www.home-assistant.io/integrations/system_health/)). L’[intégration Logger](https://www.home-assistant.io/integrations/logger/) contrôle les niveaux de journalisation globaux et spécifiques aux composants. La journalisation de débogage est limitée dans le temps et aux espaces de noms concernés ; les tempêtes radio ou d’événements peuvent sinon dominer la mémoire et les I/O.

Une chaîne de diagnostic robuste est la suivante :

1. **Symptôme et état attendu :** quelle entité, Action, automatisation ou interface est concernée ?
2. **Temps et périmètre :** depuis quand, pour quels appareils, utilisateurs, réseaux et instances d’intégration ?
3. **Identité de l’objet :** préserver Config Entry, Device ID, Entity ID, Unique ID et référence de passerelle.
4. **Exécution :** vérifier Core, Event Loop, mémoire, CPU, système de fichiers et base de données.
5. **Intégration :** vérifier l’état de l’entrée, l’authentification, le statut du coordinateur/de l’interrogation et le téléchargement des diagnostics.
6. **Transport :** vérifier Discovery, DNS, TCP, TLS, broker, contrôleur radio ou API constructeur.
7. **Automatisation :** vérifier trace, données de trigger, Conditions, Run Mode et résultat de l’Action.
8. **Persistance :** évaluer séparément le retard du Recorder et l’historique par rapport à l’état en direct.
9. **Test contrôlé :** utiliser une entité de test en lecture seule ou inoffensive.
10. **Reprise :** Reload avant Restart, Restart avant Restore ; conserver les éléments de preuve auparavant.

Une entité `unavailable` peut provenir d’une Config Entry déchargée, d’une passerelle absente, d’une perte radio ou d’un timeout de source. Un ancien chiffre visible est plus dangereux, car il semble plausible. La surveillance nécessite donc des limites de fraîcheur, et pas seulement des limites de valeur.

## Mises à jour, versions et Custom Integrations

Home Assistant publie fréquemment des versions du Core et documente les changements incompatibles avec les versions antérieures. Un article de référence statique ne fige volontairement pas un état de version momentané. Le déploiement vérifie plutôt, au moment de la maintenance, les notes de version, les changements d’intégration et les dépendances cibles.

Un chemin de mise à jour contrôlé comprend :

1. Confirmer la sauvegarde et le téléchargement indépendant ou l’emplacement de stockage externe.
2. Vérifier l’espace libre, l’état de la base de données et System Health.
3. Évaluer les notes de version ainsi que les intégrations et Custom Components concernés.
4. Inventorier les dépendances radio, broker, base de données et proxy.
5. Mettre à jour le Core ou HAOS et les Apps dans un ordre défini.
6. Vérifier le journal de démarrage, les réparations et les migrations de registres.
7. Tester les chemins critiques de capteurs, actionneurs, automatisations, API et accès distant.
8. Déterminer la limite d’erreur, puis seulement déclencher un rollback ou une restauration.

HAOS utilise RAUC avec deux slots de système d’exploitation ; `ha os info` et `rauc status` rendent l’état du slot visible ([HAOS update system](https://developers.home-assistant.io/docs/operating-system/update-system/)). Ce mécanisme protège le chemin de mise à jour de l’OS, mais pas automatiquement la configuration du Core, les données Recorder ou l’état du réseau radio.

Lorsque l’exécution et le chemin de données sont connus, l’étendue de la sauvegarde peut être déterminée. Configuration, registres, secrets, base de données et états d’add-ons doivent correspondre ensemble au modèle d’installation choisi.

## Sauvegarde et reprise

Home Assistant peut écrire des sauvegardes automatiques et manuelles, chiffrées, vers des cibles locales ou externes. Le [guide officiel de sauvegarde et restauration](https://www.home-assistant.io/common-tasks/general/) décrit les emplacements de sauvegarde, l’Emergency Kit, le téléchargement, la restauration lors de l’onboarding et la migration vers un autre matériel. Depuis 2026, le modèle cryptographique des sauvegardes a été modernisé ; l’[annonce sur le chiffrement des sauvegardes](https://www.home-assistant.io/blog/2026/03/26/modernizing-encryption-of-home-assistant-backups/) documente le changement de format et les limites de compatibilité.

Une sauvegarde n’est complète que relativement au modèle d’installation :

| Objet | Sauvegarde HAOS | Responsabilité Container/externe |
|---|---|---|
| Configuration Core et registres | peut être incluse | sauvegarder le volume de configuration |
| Apps du Supervisor | données d’apps incluses possibles | conteneurs et volumes séparés |
| Recorder SQLite | dans la zone de configuration | sauvegarde DB cohérente avec une DB externe |
| Broker MQTT | uniquement avec la sélection d’app appropriée | configuration et persistance du broker séparées |
| Zigbee/Z-Wave/Matter | données d’intégration partielles | vérifier séparément sauvegarde contrôleur/serveur et clés |
| TLS/proxy/DNS | uniquement si dans les données choisies | infrastructure externe séparée |
| Clés de sauvegarde | pas suffisamment dans la sauvegarde chiffrée elle-même | conserver l’Emergency Kit séparément |

Un test de restauration ne s’arrête pas à la connexion. Les critères d’acceptation sont : Config Entries chargées, registres cohérents, accès utilisateur possible, base de données sans erreur, brokers et passerelles connectés, appareils radio contrôlables, automatisations critiques testées et accès distant disponible avec le bon certificat. Les appareils sur batterie peuvent initialement dormir après une migration ; l’absence immédiate de valeur ne doit pas être précipitamment interprétée comme une perte de données.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Backupinventar und Prüfsummen">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-ChildItem .\ha-backups -File -Recurse |
  Get-FileHash -Algorithm SHA256 |
  Export-Csv .\ha-backups-manifest.csv -NoTypeInformation
Get-Content .\ha-backups-manifest.csv -First 5</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">find ./ha-backups -type f -print0 | sort -z | xargs -0 sha256sum &gt; ha-backups-manifest.sha256
head -n 5 ha-backups-manifest.sha256
tar -tf ./ha-backups/example-backup.tar | head</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash), [`Export-Csv`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/export-csv) et [`Get-Content`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-content) créent ou lisent le manifeste Windows. [`find`](https://man7.org/linux/man-pages/man1/find.1.html), [`sort`](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html), [`xargs`](https://man7.org/linux/man-pages/man1/xargs.1.html), [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) et [`tar`](https://www.gnu.org/software/tar/manual/html_node/index.html) assurent la partie Unix. Une somme de contrôle prouve l’intégrité de l’archive ; seul le test de restauration prouve le déchiffrement et la reprise fonctionnelle.

## RPO, RTO et haute disponibilité

Home Assistant est, dans son exploitation habituelle, une instance unique avec état. Deux instances Core actives face aux mêmes appareils, registres ou commandes de broker ne créent pas une haute disponibilité automatiquement coordonnée. Des automatisations dupliquées peuvent commuter plusieurs fois les actionneurs ; les contrôleurs radio et appareils locaux n’autorisent souvent qu’une propriété active.

Un modèle de résilience réaliste combine :

- un nœud unique fiable ou une VM aux ressources surveillées,
- un onduleur et un stockage approprié plutôt que des supports flash sensibles,
- des sauvegardes distinctes, automatiques et chiffrées,
- du matériel de remplacement documenté ou une plateforme cible VM,
- des états et clés de contrôleurs radio exportables,
- des services externes reproductibles,
- une restauration contrôlée avec reprise claire des appareils et du réseau.

Le **RPO** dépend de la dernière copie sauvegardée de la configuration, du registre, des apps et des services externes. L’historique Recorder peut avoir un RPO différent de celui de la configuration des automatisations. Le **RTO** englobe non seulement le démarrage du Core, mais aussi DNS, proxy, base de données, broker, contrôleur radio, reconnexion des appareils, capteurs endormis et tests d’acceptation.

## Sécurité et frontières de confiance

Home Assistant peut piloter des portes, le chauffage, des systèmes d’alarme et des flux énergétiques. Son périmètre d’influence est donc physique. La conception de la sécurité sépare :

- utilisateurs et administrateurs,
- sessions navigateur, application Companion et API,
- Long-Lived Tokens et webhooks,
- Core et Custom Integrations,
- Apps du Supervisor ou conteneurs externes,
- segments IoT, management et utilisateurs,
- appareils locaux et clouds des constructeurs,
- réseaux radio et leurs clés,
- cibles de sauvegarde et Emergency Kit.

MFA protège les comptes interactifs, mais pas un Long-Lived Token volé. La segmentation réseau limite les mouvements latéraux, mais ne doit pas bloquer sans contrôle les canaux de découverte et de retour nécessaires. Les Custom Integrations bénéficient d’une proximité de processus avec le Core et sont traitées comme des déploiements de code. Les secrets n’apparaissent ni dans Git, ni dans les fichiers de diagnostic, ni dans les publications de support. Les contrôles généraux figurent sous [Durcissement](/kb/haertung), les fondamentaux du transport sous [TLS](/kb/tls) et les modèles de contrat API sous [API](/kb/apis).

## Histoire technique

Home Assistant a commencé en 2013 comme projet Python de Paulus Schoutsen. La rétrospective des dix ans décrit l’évolution d’une petite application locale d’automatisation vers un grand projet open source ([10 ans de Home Assistant](https://www.home-assistant.io/blog/2023/09/17/10-years-home-assistant/)). Le Core Python et le modèle d’intégration sont restés le centre fonctionnel, tandis que frontend, clients mobiles, Supervisor, HAOS, matériel d’appareils et options cloud se développaient autour.

Avec Hass.io, devenu ensuite Home Assistant puis Home Assistant OS et Supervisor, une pile d’appliance composée de système d’exploitation, gestion de conteneurs, Core et services supplémentaires a émergé. La séparation a été clarifiée plusieurs fois dans la terminologie : les « Add-ons » s’appellent aujourd’hui **Apps**, tandis que les « intégrations » restent des extensions Python du Core. Ces termes désignent des frontières d’exécution et de sécurité différentes.

En 2024, Home Assistant est passé sous l’égide de la fondation à but non lucratif Open Home Foundation ; Nabu Casa est resté partenaire commercial. L’annonce du projet concernant l’[écosystème Open Home](https://www.home-assistant.io/blog/2024/08/08/works-with-home-assistant-becomes-part-ohf/) décrit propriété et gouvernance. En 2025, le projet a réduit les variantes d’installation prises en charge à HAOS et Container. La tendance historique ne va donc pas vers un cluster distribué, mais vers un Core central plus stable, avec des paquets d’exécution clairement pris en charge et des serveurs de protocoles autonomes.

## Liste de contrôle pour administrateurs

Un tableau de bord vert ne suffit pas comme preuve d’exploitation. La liste de contrôle relie installation, chemins d’appareils, automatisations, stockage des données et reprise dans une vue globale vérifiable.

- **Installation :** documenter HAOS ou Container, l’architecture, l’hôte, le stockage, le réseau et l’ownership.
- **Pile :** séparer Core, Supervisor, Apps/conteneurs externes, base de données, broker, proxy et serveurs radio.
- **Inventaire :** relever Config Entry, Device ID, Entity ID, Unique ID, Area et `via_device`.
- **Provenance des données :** indiquer Local/Cloud ainsi que Push/Polling pour chaque intégration critique.
- **État :** distinguer `unknown`, `unavailable`, valeur obsolète et succès confirmé sur l’appareil.
- **Automatisation :** tester trigger, Context, Condition, Run Mode, comportement au redémarrage et compensation.
- **API :** contrôler contexte utilisateur, stockage des tokens, WebSocket, reverse proxy et TLS.
- **Recorder :** surveiller base de données, filtres, intervalle de commit, purge, I/O, croissance et sauvegarde.
- **Réseau IoT :** vérifier explicitement mDNS, SSDP, MQTT, VLAN, IPv6 et les chemins radio/passerelle.
- **Mises à jour :** regrouper notes de version, Custom Integrations, sauvegarde, déploiement et réception.
- **Reprise :** tester ensemble Core, services externes, état radio, clés et Emergency Kit.
- **Preuve :** vérifier de bout en bout non seulement l’UI et les conteneurs, mais aussi au moins un chemin de capteur, d’actionneur, d’automatisation et d’API.

## Sources

- [Home Assistant Developer Docs – Vue d’ensemble de l’architecture](https://developers.home-assistant.io/docs/architecture_index/)
- [Home Assistant Developer Docs – Architecture des intégrations](https://developers.home-assistant.io/docs/architecture_components/)
- [Home Assistant Developer Docs – Core architecture](https://developers.home-assistant.io/docs/architecture/core/)
- [Home Assistant – HAOS ou Container](https://www.home-assistant.io/faq/ha-vs-hassio/)
- [Home Assistant Developer Docs – Operating System](https://developers.home-assistant.io/docs/operating-system/)
- [Home Assistant Developer Docs – Supervisor](https://developers.home-assistant.io/docs/supervisor/)
- [Home Assistant – Abandon de Core, Supervised et 32 bits](https://www.home-assistant.io/blog/2025/05/22/deprecating-core-and-supervised-installation-methods-and-32-bit-systems/)
- [GitHub – Home Assistant Core](https://github.com/home-assistant/core)
- [Home Assistant Developer Docs – Async dependency](https://developers.home-assistant.io/docs/core/integration-quality-scale/rules/async-dependency/)
- [Home Assistant Developer Docs – Integration Quality Scale](https://developers.home-assistant.io/docs/core/integration-quality-scale/)
- [Home Assistant Developer Docs – Config entries](https://developers.home-assistant.io/docs/config_entries_index/)
- [Home Assistant Developer Docs – Device registry](https://developers.home-assistant.io/docs/device_registry_index/)
- [Home Assistant Developer Docs – Entity registry](https://developers.home-assistant.io/docs/entity_registry_index/)
- [Home Assistant Developer Docs – Appareils et services](https://developers.home-assistant.io/docs/architecture/devices-and-services/)
- [Home Assistant – Intégrations](https://www.home-assistant.io/integrations/)
- [Home Assistant – Zeroconf](https://www.home-assistant.io/integrations/zeroconf/)
- [Home Assistant – SSDP](https://www.home-assistant.io/integrations/ssdp/)
- [Home Assistant – MQTT](https://www.home-assistant.io/integrations/mqtt/)
- [Home Assistant – ZHA](https://www.home-assistant.io/integrations/zha/)
- [Home Assistant – Z-Wave JS](https://www.home-assistant.io/integrations/zwave_js/)
- [Home Assistant – Matter](https://www.home-assistant.io/integrations/matter/)
- [Home Assistant – Thread](https://www.home-assistant.io/integrations/thread/)
- [Home Assistant – Bases des automatisations](https://www.home-assistant.io/docs/automation/basics/)
- [Home Assistant – Automation actions](https://www.home-assistant.io/docs/automation/action/)
- [Home Assistant – Automation triggers](https://www.home-assistant.io/docs/automation/trigger/)
- [Home Assistant – Conditions](https://www.home-assistant.io/docs/scripts/conditions/)
- [Home Assistant – Automation templating](https://www.home-assistant.io/docs/automation/templating/)
- [Home Assistant – Configuration](https://www.home-assistant.io/docs/configuration/)
- [Home Assistant – Packages](https://www.home-assistant.io/docs/configuration/packages/)
- [Home Assistant – Sécuriser Home Assistant](https://www.home-assistant.io/docs/configuration/securing/)
- [Home Assistant – Recorder](https://www.home-assistant.io/integrations/recorder/)
- [Home Assistant Developer Docs – REST API](https://developers.home-assistant.io/docs/api/rest/)
- [Home Assistant Developer Docs – WebSocket API](https://developers.home-assistant.io/docs/api/websocket/)
- [Home Assistant Developer Docs – Authentication API](https://developers.home-assistant.io/docs/auth_api/)
- [Microsoft – Invoke-RestMethod](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod)
- [Microsoft – ConvertTo-Json](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertto-json)
- [curl – Manuel](https://curl.se/docs/manpage.html)
- [jq – Manuel](https://jqlang.org/manual/)
- [Home Assistant – HTTP integration](https://www.home-assistant.io/integrations/http/)
- [Microsoft – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [Microsoft – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [Microsoft – Get-NetTCPConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Linux man-pages – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Eclipse Mosquitto – mosquitto_sub](https://mosquitto.org/man/mosquitto_sub-1.html)
- [Eclipse Mosquitto – mosquitto_pub](https://mosquitto.org/man/mosquitto_pub-1.html)
- [Docker – inspect](https://docs.docker.com/reference/cli/docker/inspect/)
- [Docker – logs](https://docs.docker.com/reference/cli/docker/container/logs/)
- [Docker – stats](https://docs.docker.com/reference/cli/docker/container/stats/)
- [Microsoft – ConvertFrom-Json](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json)
- [Microsoft – Select-String](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/select-string)
- [GNU Grep – Manuel](https://www.gnu.org/software/grep/manual/grep.html)
- [GNU Coreutils – df](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html)
- [GNU Coreutils – du](https://www.gnu.org/software/coreutils/manual/html_node/du-invocation.html)
- [Home Assistant – System Health](https://www.home-assistant.io/integrations/system_health/)
- [Home Assistant – Logger](https://www.home-assistant.io/integrations/logger/)
- [Home Assistant Developer Docs – HAOS update system](https://developers.home-assistant.io/docs/operating-system/update-system/)
- [Home Assistant – Sauvegarde et restauration](https://www.home-assistant.io/common-tasks/general/)
- [Home Assistant – Chiffrement des sauvegardes modernisé](https://www.home-assistant.io/blog/2026/03/26/modernizing-encryption-of-home-assistant-backups/)
- [Microsoft – Get-FileHash](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash)
- [Microsoft – Export-Csv](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/export-csv)
- [Microsoft – Get-Content](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-content)
- [Linux man-pages – find(1)](https://man7.org/linux/man-pages/man1/find.1.html)
- [GNU Coreutils – sort](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html)
- [Linux man-pages – xargs(1)](https://man7.org/linux/man-pages/man1/xargs.1.html)
- [GNU Coreutils – utilitaires sha2](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [GNU Tar – Manuel](https://www.gnu.org/software/tar/manual/html_node/index.html)
- [Home Assistant – 10 ans de Home Assistant](https://www.home-assistant.io/blog/2023/09/17/10-years-home-assistant/)
- [Home Assistant – Open Home Foundation et gouvernance](https://www.home-assistant.io/blog/2024/08/08/works-with-home-assistant-becomes-part-ohf/)
