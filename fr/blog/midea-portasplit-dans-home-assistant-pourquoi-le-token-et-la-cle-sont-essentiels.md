---
title: "Sécuriser la Midea PortaSplit dans Home Assistant : token, clé et réseau domestique"
navTitle: "Sécuriser la PortaSplit"
description: "Le token et la clé de la PortaSplit proviennent du cloud Midea et n’expirent jamais. Voici comment sécuriser ces valeurs, isoler l’appareil sur le réseau domestique et maintenir Home Assistant, l’intégration et le firmware à jour de manière contrôlée."
date: "2026-07-24"
kategorie: "Home Assistant et IoT"
timeToRead: "14 min de lecture"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant
  - midea-v2-cloud-api-portasplit-home-assistant
image: "../images/midea-portasplit-home-assistant/portasplit-dashboard.png"
slug: "midea-portasplit-dans-home-assistant-pourquoi-le-token-et-la-cle-sont-essentiels"
translationOf: "midea-portasplit-home-assistant-absichern"
translationId: article-a02e26cce22063f1
translationReview: automatic
translationSourceHash: d9e59ab5647df261814123d662df93c68c9f6cd35c20cec518de0a9cf54d6e22
translatedAt: 2026-10-31T03:37:50.148Z
translationModel: gpt-5.6-terra
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/midea-portasplit-dans-home-assistant-pourquoi-le-token-et-la-cle-sont-essentiels
---

<aside class="article-update">
  <p class="article-update__label">Ce que les propriétaires de PortaSplit devraient faire maintenant</p>
  <p>Lors de la configuration, Home Assistant récupère le token et la clé de la PortaSplit via des interfaces cloud privées. Le projet Midea AC LAN avertit depuis le 19 mai 2025 de possibles changements ; aucune date d’arrêt par le fabricant n’est documentée. Pour les propriétaires, cela signifie :</p>
  <ol>
    <li><strong>Sauvegarder le token, la clé et la configuration de manière chiffrée.</strong> Si leur récupération ne fonctionne plus par la suite, la sauvegarde est le seul moyen de restaurer la configuration.</li>
    <li><strong>Ne pas dissocier l’appareil sans nécessité.</strong> Le rétablissement des paramètres d’usine, la suppression du compte Midea ou le remplacement du module Wi-Fi imposent une nouvelle obtention du token.</li>
    <li><strong>Isoler la PortaSplit sur le réseau domestique.</strong> Aucune redirection de port, un VLAN IoT dédié, accès uniquement depuis Home Assistant.</li>
  </ol>
</aside>

La commande locale de la Midea PortaSplit repose sur deux valeurs spécifiques à l’appareil : le token et la clé. Elles authentifient la connexion entre Home Assistant et l’appareil et ne peuvent actuellement être obtenues que via le cloud Midea. Cela implique deux tâches : sauvegarder ces valeurs afin qu’une nouvelle configuration reste possible sans le cloud, et exploiter l’appareil ainsi que Home Assistant de façon à limiter les dégâts potentiels de ces valeurs en cas d’incident.

La série comprend trois parties : la [partie 1](/blog/midea-portasplit-home-assistant) décrit la configuration jusqu’au tableau de bord, cette partie traite de la sécurisation, et la [partie 3](/blog/midea-v2-cloud-api-portasplit-home-assistant) explique le contexte des avertissements concernant l’API cloud.

![Tableau de bord Home Assistant de la Midea PortaSplit en mode refroidissement : indicateurs en haut, thermostat à 22 °C, courbes de la température ambiante, de la puissance absorbée, de l’énergie quotidienne, de la fréquence du compresseur, du fonctionnement du compresseur et de la vitesse du ventilateur, puis valeurs techniques et état.](../images/midea-portasplit-home-assistant/portasplit-dashboard.png)

## Origine du token et de la clé

Sur les appareils utilisant le protocole V3, la PortaSplit n’accepte les commandes locales qu’avec un token et une clé. Ces valeurs ne sont pas générées par l’appareil, mais par le cloud Midea ; l’application officielle les récupère également à cet endroit. Les intégrations communautaires ont réimplémenté cet appel cloud : elles se connectent aux mêmes points de terminaison que l’application, obtiennent le token et la clé, puis les enregistrent localement. Aucune connexion cloud n’est ensuite nécessaire pour le fonctionnement courant.

Il n’existe pas de mécanisme d’appairage local documenté qui fournisse ces valeurs sans passer par le cloud. Théoriquement, il serait possible de les extraire de l’application, par exemple par rétro-ingénierie ou instrumentation à l’exécution ; pour un utilisateur individuel, cette opération est complexe et ne remplace pas la récupération via le cloud. Si le point de terminaison disparaît, l’obtention des valeurs disparaît donc également.

Le projet `Midea AC LAN` avertit dans son README que Midea pourrait progressivement fermer les interfaces de token à l’avenir. Depuis la révision du README du 3 septembre 2026, l’intégration récupère le token et la clé via les clouds Meiju et SmartHome. Selon le README, cela ne fonctionne qu’avec un compte personnel auquel l’appareil est associé ; il n’est actuellement pas possible de récupérer des tokens via NetHome Plus. Depuis le 9 septembre 2026, le README indique en outre qu’un compte SmartHome migré depuis une autre application ne fournit aucun token ; un compte nouvellement créé avec des appareils ajoutés à nouveau est nécessaire. Les appareils déjà configurés continuent de fonctionner localement ; les nouveaux appareils et les nouvelles configurations seraient concernés. Il ne s’agit pas d’une feuille de route contraignante de Midea. En juin 2026, il s’est également avéré que l’API de token SmartHome prétendument fermée continuait de fonctionner ; la requête de la bibliothèque communautaire était simplement incomplète. L’analyse de cet avertissement et des différentes appellations « V2 » figure dans la [partie 3](/blog/midea-v2-cloud-api-portasplit-home-assistant).

## Ce que permettent le token et la clé

Le token et la clé n’ont pas de date d’expiration. Selon `Midea AC LAN`, la communication client était initialement considérée comme suffisamment protégée, raison pour laquelle le cloud fournissait des tokens sans expiration. Cela n’est pas une vulnérabilité en soi ; le problème survient lorsque ces valeurs se retrouvent dans des journaux ou des sauvegardes non protégées, sont transmises à des tiers ou ne peuvent être ni révoquées ni renouvelées.

Quiconque possède le token et la clé et peut joindre l’appareil sur le réseau peut s’authentifier auprès de la PortaSplit, lire les informations d’état, l’allumer et l’éteindre, changer les modes de fonctionnement et modifier la température de consigne. Ces valeurs seules ne permettent pas une attaque depuis Internet ; l’attaquant a aussi besoin d’une connexion réseau vers l’appareil. Le token et la clé doivent donc être traités comme un mot de passe, et le réseau devrait autant que possible n’autoriser cette connexion qu’à Home Assistant.

L’intégration communautaire n’attaque pas le climatiseur. Elle implémente un protocole propriétaire qui a été compris par rétro-ingénierie. Le risque provient du fait que des secrets durables sont stockés en dehors de l’application prévue à cet effet.

## Sauvegarder le token, la clé et la configuration

La sauvegarde du token, de la clé et de la configuration est l’étape unique la plus importante : une fois les interfaces cloud de token fermées, une sauvegarde est le seul moyen d’effectuer une nouvelle configuration. `Midea AC LAN` crée, après une configuration réussie pour les appareils V3, un fichier de configuration JSON. Le chemin documenté est :

```text
/config/.storage/midea_ac_lan/
```

Le fichier porte l’identifiant de l’appareil comme nom de fichier :

```text
<device-id>.json
```

Ce fichier n’est pas une simple note de texte. Il peut contenir l’identifiant de l’appareil, le numéro de série, l’adresse IP, le token, la clé, des informations de protocole ainsi que des paramètres cloud et appareil. Les règles suivantes s’appliquent donc :

- Ne pas le téléverser dans un dépôt GitHub public.
- Ne pas le publier sur des forums.
- Ne pas partager de capture d’écran non caviardée.
- Ne pas l’envoyer par e-mail non chiffré.

Même un dépôt Git privé n’est pas automatiquement le bon emplacement, car les secrets restent dans l’historique Git, même s’ils sont ensuite supprimés du fichier actuel. Une sauvegarde chiffrée, un gestionnaire de mots de passe avec pièce jointe, une sauvegarde NAS chiffrée, un support hors ligne chiffré ou une archive chiffrée dont le mot de passe est stocké séparément sont plus adaptés.

Pour la sauvegarde via le terminal Home Assistant :

```bash
cd /config/.storage/midea_ac_lan
ls -la
```

Afficher le fichier :

```bash
cat <device-id>.json
```

Pour la copie, le fichier ne doit pas être transféré via un service Web public. Une archive chiffrée, ensuite placée dans une sauvegarde chiffrée, est préférable :

```bash
tar -czf /config/midea-ac-lan-backup.tar.gz \
  /config/.storage/midea_ac_lan
```

Les fichiers dans `.storage` ne doivent pas être modifiés manuellement. Le développeur recommande expressément, en cas de problème, de ne pas supprimer ni modifier directement le fichier JSON, mais de le renommer et de le sauvegarder avant toute modification.

Une sauvegarde complète de Home Assistant contient également ces fichiers. Une copie séparée reste néanmoins utile, car les sauvegardes Home Assistant peuvent être endommagées, une restauration peut écraser l’intégration, le fichier peut être nécessaire spécifiquement pour une nouvelle configuration ultérieure et une sauvegarde ne devrait jamais se trouver uniquement sur le même système.

### Retirer des secrets d’un dépôt Git publié

Si un fichier JSON a été publié par erreur sur GitHub, une suppression normale suivie d’un nouveau commit ne suffit pas. Le fichier reste accessible dans l’historique Git. Au minimum, les étapes suivantes sont nécessaires :

1. Rendre immédiatement le dépôt privé, si possible.
2. Retirer le fichier de l’ensemble de l’historique Git.
3. Prendre en compte les caches GitHub et les forks.
4. Considérer le token comme compromis.
5. Retirer l’appareil du compte Midea et le reconnecter si cela génère de nouvelles clés.
6. Reconfigurer l’intégration Home Assistant.
7. Modifier le mot de passe du compte Midea si les identifiants ont également été concernés.

Le fait qu’un nouvel appairage génère effectivement un nouveau token varie selon l’appareil et l’architecture cloud. Il ne faut pas compter sur le fait que la modification du mot de passe du compte invalide automatiquement le token local de l’appareil.

## Isoler la PortaSplit sur le réseau

### Pas de redirection de port vers la PortaSplit

L’erreur évitable la plus fréquente serait de rendre le port local de l’appareil directement accessible depuis Internet. Une règle telle que celle-ci serait dangereuse :

```text
Internet → TCP 6444 → PortaSplit
```

Il n’y a aucune bonne raison de rendre la PortaSplit directement accessible depuis Internet. Home Assistant se trouve déjà sur le réseau local et sert d’instance de contrôle. Le routeur ne devrait disposer d’aucune redirection de port vers la PortaSplit, limiter ou désactiver UPnP si possible, bloquer les connexions entrantes par défaut et ne pas utiliser de règle DMZ pour l’appareil.

### VLAN IoT dédié

La meilleure architecture réseau est un réseau IoT séparé :

```text
VLAN 10: vertrauenswürdige Clients
VLAN 20: Server und Home Assistant
VLAN 30: IoT-Geräte
VLAN 40: Gäste
```

La PortaSplit se trouve dans le VLAN IoT. Home Assistant peut accéder spécifiquement à l’appareil, mais la PortaSplit ne doit pas pouvoir accéder librement aux PC, NAS et autres systèmes internes. Une logique de pare-feu possible :

```text
Home Assistant → PortaSplit: erlauben
PortaSplit → Home Assistant: etablierte Verbindungen erlauben
PortaSplit → interne Clients: blockieren
PortaSplit → NAS: blockieren
PortaSplit → Management-Netz: blockieren
Internet → PortaSplit: blockieren
```

Lors de la configuration initiale, l’appareil a besoin d’un accès Internet au cloud Midea. Après une configuration locale réussie, il est possible de tester si l’accès Internet sortant peut être bloqué. Il ne faut pas appliquer immédiatement un blocage définitif. Vérifiez d’abord si la commande locale continue de fonctionner, si l’appareil reste accessible après un redémarrage, s’il résiste à un redémarrage du routeur, s’il répond toujours après plusieurs jours, si l’application MSmartHome reste nécessaire et si des mises à jour du firmware sont encore proposées. Ceux qui souhaitent continuer à utiliser le cloud et les mises à jour du firmware peuvent autoriser temporairement l’accès Internet sortant, puis le bloquer à nouveau.

### La segmentation réseau peut empêcher la découverte

La recherche automatique d’appareils repose souvent sur le trafic broadcast ou multicast, lequel n’est généralement pas routé au-delà des frontières entre VLAN. Home Assistant risque donc de ne pas détecter automatiquement la PortaSplit, même si une connexion IP normale serait autorisée.

Dans ce cas, il peut être utile de configurer temporairement la PortaSplit dans le même VLAN que Home Assistant, d’indiquer manuellement l’adresse IP de l’appareil, d’utiliser une fonction de relais broadcast appropriée ou de définir des règles de pare-feu ciblées après la configuration. La configuration manuelle est souvent même la meilleure solution du point de vue de la sécurité, car aucun trafic broadcast supplémentaire entre les réseaux ne doit être autorisé.

### Attribution DHCP statique

La PortaSplit devrait recevoir une attribution DHCP fixe dans le routeur :

```text
PortaSplit → 192.168.30.25
```

Une réservation DHCP est généralement préférable à une adresse IP statique configurée dans l’appareil. Home Assistant trouve l’appareil de manière fiable, les règles de pare-feu peuvent être limitées à une adresse fixe, l’analyse des erreurs devient plus simple et l’attribution reste stable après le redémarrage du routeur ou de l’appareil. Une règle de pare-feu peut ainsi être formulée de manière très restrictive :

```text
Home-Assistant-IP → 192.168.30.25:6444/TCP
```

Le port réellement requis doit être vérifié à partir de l’intégration et de votre propre appareil.

## Sécuriser Home Assistant et les intégrations

### Home Assistant comme ancre de confiance centrale

En commandant la PortaSplit localement, on déplace une partie de la confiance du cloud Midea vers Home Assistant. Si Home Assistant est compromis, un attaquant peut éventuellement contrôler non seulement le climatiseur, mais l’ensemble de la maison connectée.

Home Assistant devrait donc être mis à jour régulièrement, ne pas être publié via une redirection de port non protégée, être protégé par un mot de passe fort et unique, utiliser l’authentification multifacteur, créer des sauvegardes chiffrées, ne contenir que les add-ons nécessaires et ne pas autoriser d’accès SSH inutile depuis Internet. Pour l’accès à distance, un VPN, Home Assistant Cloud ou un reverse proxy correctement configuré sont de meilleures options qu’une simple redirection de port vers le port 8123.

### HACS et le risque de chaîne d’approvisionnement

`Midea Smart AC` et `Midea AC LAN` sont des Custom Integrations. Elles s’exécutent au sein de Home Assistant et disposent ainsi d’un accès étendu à son environnement d’exécution. Une intégration malveillante ou compromise pourrait théoriquement lire des données de configuration, extraire des secrets, établir des connexions réseau, analyser les appareils du réseau local, lire les états d’autres entités, transmettre des données à des systèmes externes et affecter la disponibilité de Home Assistant.

Cela ne signifie pas que les intégrations citées sont malveillantes. Les deux projets sont publics, activement développés et disposent d’une communauté visible. L’open source n’est toutefois pas une garantie automatique de sécurité. Avant l’installation, il vaut au minimum la peine de vérifier si le dépôt est activement maintenu, s’il existe des versions régulières, combien de personnes contribuent au code, s’il y a des problèmes de sécurité ouverts, si les responsables de maintenance ou le propriétaire du dépôt ont récemment changé, si HACS renvoie vers le dépôt attendu et si une mise à jour contient des changements inhabituellement importants ou inexplicables.

Les mises à jour ne devraient pas être installées aveuglément immédiatement après leur publication. Surtout pour les systèmes de maison connectée critiques du point de vue de la sécurité, il est judicieux d’attendre quelques jours et de consulter les notes de version ainsi que les problèmes signalés.

### Les journaux de débogage contiennent des données sensibles

En cas de problème, les projets open source demandent souvent des journaux de débogage. La documentation de `Midea AC LAN` montre comment activer la journalisation pour les deux composants concernés :

```yaml
logger:
  default: warn
  logs:
    custom_components.midea_ac_lan: debug
    midealocal: debug
```

Les journaux peuvent ensuite être téléchargés via Paramètres, Système et Journaux. Selon l’intégration et l’erreur, ces journaux peuvent contenir des adresses IP locales, l’identifiant de l’appareil, le numéro de série, l’identifiant de modèle, des réponses cloud, des informations de compte, le token ou une partie de celui-ci, des paquets réseau ainsi que des horodatages et des habitudes d’utilisation. Avant de les téléverser dans une issue GitHub publique, il faut donc les vérifier et masquer les valeurs sensibles.

Une fois le dépannage terminé, la journalisation de débogage doit être supprimée. Une journalisation de débogage activée en permanence augmente non seulement l’utilisation du stockage, mais aussi la quantité d’informations sensibles dans les sauvegardes.

## Cloud et firmware

### Sécuriser le compte cloud

Tant que le cloud Midea est utilisé pour la configuration ou le contrôle via l’application, le compte Midea reste lui aussi une partie du modèle de sécurité. Il doit disposer d’un mot de passe unique, non partagé avec d’autres services, d’un gestionnaire de mots de passe, d’une authentification multifacteur si elle est proposée, de la suppression des anciens smartphones et sessions, de l’absence de comptes partagés et d’un contrôle régulier des appareils enregistrés dans le compte.

Si l’intégration Home Assistant demande un nom d’utilisateur et un mot de passe lors de la configuration, il faut vérifier si les identifiants sont utilisés uniquement pour la récupération ponctuelle du token ou s’ils sont stockés durablement. Les développeurs de `Midea Smart AC` écrivent que les appareils ne sont pas liés à des comptes d’intégration intégrés après la configuration et que le token et la clé peuvent aussi être obtenus manuellement via son propre compte au moyen de la CLI. Lorsqu’il est possible, il faut privilégier son propre compte aux comptes tiers ou collectifs intégrés. Pour `Midea AC LAN`, un compte personnel Meiju ou SmartHome auquel l’appareil est associé est, depuis septembre 2026, une condition préalable à la récupération du token.

### Bloquer le cloud ou non ?

Après une configuration réussie, la question se pose de savoir si l’accès Internet de la PortaSplit devrait être entièrement bloqué. En faveur d’un blocage : moins de télémétrie, une dépendance réduite aux services externes, une surface d’attaque moindre via le cloud du fabricant, le fait que l’appareil ne peut pas contacter des destinations externes arbitraires et un impact réduit des changements côté cloud.

En revanche, l’application MSmartHome pourrait ne plus fonctionner en dehors du réseau domestique, les mises à jour du firmware pourraient ne plus être téléchargées, les fonctions d’horloge ou cloud pourraient ne plus fonctionner, une nouvelle connexion ou restauration pourrait devenir plus difficile et certains appareils pourraient réagir de manière inattendue après une longue période hors ligne.

Un ordre pragmatique : configurer l’appareil normalement, tester Home Assistant et l’application, sauvegarder le token et la configuration, bloquer l’accès Internet, redémarrer l’appareil et Home Assistant, observer pendant plusieurs jours et, si nécessaire, ne réautoriser l’accès Internet que temporairement.

### Mises à jour du firmware : gain de sécurité ou risque d’intégration ?

Les mises à jour du firmware constituent un dilemme pour les appareils IoT. Elles peuvent corriger des vulnérabilités connues, améliorer la stabilité, moderniser les mécanismes de sécurité et apporter de nouvelles fonctions. Mais elles peuvent aussi modifier les interfaces locales, casser les intégrations issues de rétro-ingénierie, invalider les tokens, désactiver l’API locale et introduire de nouvelles dépendances au cloud.

Le firmware de la PortaSplit publié en janvier 2026 a par exemple apporté un nouveau mode silencieux pour l’unité extérieure, qui réduit le bruit d’environ 6 décibels. Les intégrations communautaires ont d’abord dû l’analyser et l’implémenter, ce qui est documenté dans une issue GitHub dédiée à la PortaSplit.

Il en découle qu’il ne faut pas empêcher systématiquement les mises à jour du firmware, mais vérifier avant une mise à jour si d’autres utilisateurs de Home Assistant signalent des problèmes, sauvegarder la configuration et le token au préalable, créer une sauvegarde Home Assistant et tester entièrement la commande locale après la mise à jour. La sécurité ne signifie pas « ne jamais mettre à jour ». Un firmware obsolète peut être plus dangereux qu’une intégration temporairement incompatible.

### Ce que Midea dit elle-même sur la sécurité

Midea fait la promotion de son écosystème SmartHome en se référant à plusieurs normes de sécurité et de protection des données, notamment EN 303 645, UK PSTI, NIST, un traitement des données conforme au RGPD et les exigences de la directive européenne sur les équipements radio. Ce sont des signaux positifs, mais ils ne disent rien de la manière dont chaque firmware PortaSplit, chaque point de terminaison cloud et chaque API locale sont réellement implémentés. Les certifications et déclarations marketing ne remplacent pas un examen technique de l’appareil concret.

De même, il serait erroné de déduire de l’avertissement d’une intégration communautaire que la PortaSplit est généralement non sécurisée. Le problème décrit concerne l’architecture des tokens durables et leur utilisation par des clients non officiels.

## Risque selon le scénario

| Scénario | Risque | Justification |
| --- | --- | --- |
| Réseau domestique normal sans redirection de port | modéré | Un attaquant doit d’abord accéder au Wi-Fi, à Home Assistant ou à une sauvegarde. |
| Réseau domestique plat avec de nombreux appareils IoT non sécurisés | moyen | Un autre appareil IoT compromis peut joindre la PortaSplit ou Home Assistant sur le même réseau. |
| PortaSplit directement accessible depuis Internet | élevé | L’appareil ne devrait jamais être publié via une redirection de port. |
| Token et clé publics sur GitHub | élevé | Les secrets sont considérés comme compromis ; leur révocation n’est pas garantie. |
| VLAN IoT séparé, pare-feu restrictif, commande locale | relativement faible | Même en cas de vulnérabilité de l’appareil, la liberté de mouvement sur le réseau est fortement limitée. |

## Liste de contrôle

```text
1. Home-Assistant-Backup anfertigen
2. Token- und Konfigurationsdaten verschlüsselt sichern
3. DHCP-Reservation für die PortaSplit einrichten
4. Keine Portweiterleitung, UPnP einschränken
5. PortaSplit in ein separates IoT-VLAN verschieben
6. Zugriff von Home Assistant zur PortaSplit erlauben
7. Zugriff der PortaSplit auf interne Netze blockieren
8. Internetzugriff testweise blockieren
9. lokale Steuerung nach Neustarts prüfen
10. Firmware- und Integrationsupdates kontrolliert durchführen
```

Le sens de communication souhaité :

```text
Home Assistant
    │
    │ gezielt erlaubt
    ▼
Midea PortaSplit
    │
    ├── kein Zugriff auf PCs
    ├── kein Zugriff auf NAS
    ├── kein Zugriff auf Management-Netz
    └── Internet nur bei Bedarf
```

Exploitée ainsi, la commande locale est acceptable du point de vue de la sécurité : le token et la clé restent secrets et sauvegardés, l’appareil n’est accessible que par Home Assistant et les mises à jour du firmware et de l’intégration sont appliquées de manière contrôlée.

## Historique des modifications

- **25 octobre 2026 :** Ajout : depuis la révision du README de septembre 2026, Midea AC LAN exige pour la récupération du token un compte personnel Meiju ou SmartHome auquel l’appareil est associé ; NetHome Plus et les comptes SmartHome migrés ne fournissent aucun token.

## Sources

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: intégration `Midea AC LAN` avec l’« Important Notice » (depuis le 19 mai 2025, révisée les 3 et 9 septembre 2026), l’explication concernant les tokens sans expiration et la description de l’obtention cloud du token via un compte personnel Meiju ou SmartHome.

2.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: intégration `Midea Smart AC` : obtention cloud du token et de la clé pour les appareils V3, stockage local des valeurs, port standard 6444.

3.  [midea_ac_lan : indications de débogage et de configuration](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/debug.md): emplacement de la configuration de l’appareil sous `/config/.storage/midea_ac_lan/`, recommandation de sauvegarder plutôt que supprimer le fichier JSON et configuration du logger pour les journaux de débogage.

4.  [Issue 779 : mode silencieux extérieur de la PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/779): demande de prise en charge du mode silencieux de l’unité extérieure introduit avec la mise à jour du firmware de janvier 2026, qui réduit le bruit d’environ 6 décibels.

5.  [Midea SmartHome](https://www.midea.com/global/smarthome): informations du fabricant concernant les normes de sécurité et de protection des données EN 303 645, PSTI, NIST, RGPD et RED DA.

6.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): installation et gestion de Custom Integrations qui ne font pas partie de Home Assistant Core.
