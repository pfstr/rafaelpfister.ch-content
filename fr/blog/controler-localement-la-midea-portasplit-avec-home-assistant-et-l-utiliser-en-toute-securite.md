---
title: "Contrôler localement et exploiter en toute sécurité le Midea PortaSplit avec Home Assistant"
navTitle: "Configurer PortaSplit"
description: "De la bonne intégration communautaire au VLAN IoT : configurez le PortaSplit, protégez le token et la clé, et limitez les accès au cloud et au réseau."
date: "2026-07-24"
kategorie: "Home Assistant et IoT"
timeToRead: "14 min de lecture"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant
  - serverloser-newsletter-cloudflare-workers-d1
slug: "controler-localement-la-midea-portasplit-avec-home-assistant-et-l-utiliser-en-toute-securite"
translationOf: "midea-portasplit-home-assistant-einrichten"
translationId: article-36e7710abe426781
translationReview: automatic
translationSourceHash: bbe70b67dd255184cf0db69f7308c756937dc961c3c83e152268ee668f93dd07
translatedAt: 2026-09-04T08:32:33.662Z
translationModel: gpt-5.6-terra
url: https://rafaelpfister.ch/fr/blog/controler-localement-la-midea-portasplit-avec-home-assistant-et-l-utiliser-en-toute-securite
---

Le Midea PortaSplit peut être contrôlé directement sur le réseau local via Home Assistant après sa configuration. Pour cela, l’intégration communautaire nécessite deux identifiants spécifiques à l’appareil provenant du cloud Midea : un token et une clé.

Cet article vous guide dans le choix, la configuration et la sécurisation de l’intégration. Les solutions décrites proviennent de la communauté et ne sont officiellement prises en charge ni par Midea ni par Home Assistant. Des modifications du firmware ou du cloud peuvent donc à tout moment affecter leur fonctionnement. Le contexte relatif à l’interface des tokens et à l’avertissement ambigu concernant son arrêt est présenté dans [l’analyse des API du cloud Midea](/blog/midea-v2-cloud-api-portasplit-home-assistant).

## Comment fonctionne le contrôle local

Après la configuration, les commandes de contrôle sont envoyées directement de Home Assistant au PortaSplit :

```text
Home Assistant → lokales Netzwerk → Midea PortaSplit
```

Une commande n’a pas besoin de passer par un serveur Midea externe, le temps de réponse est court, une panne du cloud Midea n’interrompt pas nécessairement le contrôle local déjà configuré, et l’appareil reste en principe contrôlable même sans accès à Internet.

Toutefois, sur les appareils récents utilisant le protocole dit V3, le PortaSplit n’accepte pas les commandes locales sans protection. Home Assistant a besoin de deux valeurs spécifiques à l’appareil, un token et une clé, qui servent à l’authentification et au chiffrement de la connexion locale. Lors de la configuration initiale, l’intégration les récupère une seule fois via une interface du cloud Midea et les enregistre ensuite localement ; aucune connexion au cloud n’est nécessaire pour le contrôle ultérieur.

Le déroulement est simplifié comme suit :

1. Le PortaSplit est connecté à MSmartHome.
2. Home Assistant se connecte à un cloud Midea.
3. Home Assistant obtient l’ID de l’appareil, le token et la clé.
4. Le token et la clé sont enregistrés localement.
5. Home Assistant contrôle directement le PortaSplit sur le LAN.

## Quelle intégration choisir

### Midea Smart AC

Le dépôt <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> se concentre sur les climatiseurs Midea et les modèles OEM apparentés, et prend en charge les types d’appareils `0xAC` et `0xCC`. Il offre un contrôle local, une configuration graphique, une détection automatique, une configuration manuelle avec token et clé, ainsi qu’une interrogation automatique des capacités de l’appareil. Le « Out Silent Mode » du PortaSplit est explicitement pris en charge.

Le projet cite notamment comme indice de compatibilité les applications Artic King, Midea Air, NetHome Plus, SmartHome ou MSmartHome, Toshiba AC NA et 美的美居. En Europe, le PortaSplit utilise généralement MSmartHome et s’inscrit donc dans cet écosystème.

### Midea AC LAN

Le dépôt <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> prend en charge non seulement les climatiseurs, mais aussi de nombreuses autres catégories d’appareils Midea : déshumidificateurs, ventilateurs, purificateurs d’air, lave-linge, sèche-linge, lave-vaisselle, chauffe-eau, pompes à chaleur, réfrigérateurs et plus encore, parfois aussi sous des marques tierces telles que Carrier ou Electrolux. Il offre également une communication locale, une détection automatique des appareils et des capteurs supplémentaires. Selon la description du projet, il maintient une connexion TCP plus longue avec l’appareil afin de synchroniser rapidement les changements d’état. Home Assistant 2024.4.1 au minimum est requis.

Le principal inconvénient est actuellement l’avertissement du développeur : les API de tokens cloud utilisées pour ajouter de nouveaux appareils sont progressivement désactivées. Il pourrait ainsi devenir impossible d’ajouter ultérieurement de nouveaux appareils.

### Recommandation

Pour une installation PortaSplit uniquement, je commencerais par `Midea Smart AC` et considérerais `Midea AC LAN` comme alternative. `Midea Smart AC` est davantage spécialisé dans les climatiseurs et documente explicitement les fonctions actuelles du PortaSplit.

Il n’est pas judicieux d’utiliser simultanément et durablement les deux intégrations avec le même appareil. Plusieurs connexions parallèles entraînent des problèmes d’état, du trafic réseau inutile et des comportements difficiles à comprendre.

## Ce que l’intégration apporte

Après la configuration, le PortaSplit apparaît comme une entité `climate` dans Home Assistant. Selon le firmware et l’intégration, les fonctions suivantes sont notamment disponibles :

- Allumer et éteindre
- Régler la température de consigne
- Lire la température ambiante actuelle
- Refroidissement, déshumidification et ventilation seule
- Régler la vitesse du ventilateur
- Contrôler la fonction Swing
- Mode Eco et Boost
- Lire l’humidité de l’air
- Afficher les codes d’erreur
- Lire les valeurs d’énergie et de puissance
- Afficher les valeurs du compresseur
- Activer le mode silencieux de l’unité extérieure

Les entités effectivement affichées dépendent du modèle, du firmware, du protocole utilisé et de l’intégration concernée. `Midea Smart AC` interroge les capacités signalées par l’appareil et masque les fonctions que le modèle ne prend pas en charge. `Midea AC LAN` documente également de nombreuses entités de climatisation, notamment la température, l’humidité, la puissance actuelle, l’énergie totale, la fréquence du compresseur, l’état de la pompe et différents modes de fonctionnement, et mentionne des méthodes propres à certains sous-types de PortaSplit pour décoder les données énergétiques.

Toutes les mesures affichées ne sont pas nécessairement correctes. La consommation d’énergie et la puissance, en particulier, sont transmises dans différents formats selon les modèles Midea. Si Home Assistant affiche des valeurs manifestement erronées, il faut généralement adapter la méthode de décodage utilisée plutôt que considérer l’appareil comme défectueux.

## Prérequis

Vous avez besoin d’un Midea PortaSplit avec fonction Wi-Fi, d’un réseau Wi-Fi 2,4 GHz, de l’application MSmartHome, d’un compte utilisateur Midea, de Home Assistant, de HACS et d’un accès réseau entre Home Assistant et le PortaSplit. Le PortaSplit doit d’abord être connecté normalement avec l’application MSmartHome, puis ajouté à Home Assistant.

## Étape 1 : connecter le PortaSplit à MSmartHome

1. Installer l’application MSmartHome.
2. Créer un compte Midea ou se connecter.
3. Mettre le PortaSplit en mode d’appairage Wi-Fi.
4. Connecter l’appareil au réseau Wi-Fi 2,4 GHz.
5. Vérifier que le PortaSplit peut être contrôlé avec l’application.

De nombreux appareils IoT ne prennent encore en charge que le 2,4 GHz. Si le routeur utilise le même SSID pour les bandes 2,4 et 5 GHz, la configuration fonctionne généralement tout de même. En cas de problème, il peut être utile de fournir temporairement un réseau Wi-Fi 2,4 GHz séparé.

## Étape 2 : installer HACS

HACS est le Community Store de Home Assistant. Il permet d’installer des intégrations communautaires qui ne font pas partie de Home Assistant Core. Après l’installation de HACS, ouvrez HACS, accédez aux intégrations, recherchez `Midea Smart AC`, téléchargez l’intégration et redémarrez Home Assistant. Vous pouvez aussi rechercher `Midea AC LAN`.

HACS simplifie l’installation et les mises à jour. Il ne transforme toutefois pas une Custom Integration en composant Home Assistant officiellement vérifié. Cette différence est essentielle du point de vue de la sécurité et sera abordée plus loin.

## Étape 3 : ajouter Midea Smart AC

Après le redémarrage, accédez à Paramètres, Appareils et services, puis Ajouter une intégration et recherchez `Midea Smart AC`, puis `Discover devices`. L’intégration peut soit rechercher sur l’ensemble du réseau local, soit interroger directement l’adresse IP du PortaSplit.

Si l’appareil est détecté, l’intégration requiert pour les appareils V3 récents la région, le compte Midea, le mot de passe et l’ID de l’appareil, ainsi que le token et la clé qui en sont dérivés. La région cloud doit correspondre au compte utilisé. En cas de problème, le projet recommande également d’essayer les autres régions proposées.

### Configuration manuelle

Si la configuration automatique échoue, l’appareil peut être configuré manuellement. Pour `Midea Smart AC`, les informations suivantes sont nécessaires :

```text
Device ID
IP-Adresse
Port
Gerätetyp
Token
Key
```

Le port standard documenté est :

```text
6444/TCP
```

Pour les appareils V3, la documentation indique que le token est une chaîne hexadécimale de 128 caractères et la clé une chaîne hexadécimale de 64 caractères. Ces deux valeurs sont des secrets et doivent être traitées en conséquence. Si vous ne souhaitez pas obtenir les identifiants via la découverte, vous pouvez les récupérer avec votre propre compte au moyen de l’interface CLI `msmart-ng`.

## Exploiter le PortaSplit en toute sécurité

En contrôlant le PortaSplit localement, vous récupérez une partie du contrôle depuis le cloud du fabricant, mais transférez aussi la responsabilité vers votre propre réseau. Les mesures suivantes permettent de limiter les dommages en cas d’incident impliquant le token ou la clé et d’isoler correctement l’appareil.

### Le token et la clé sont des secrets

Le token et la clé authentifient la communication locale avec l’appareil et doivent être traités comme un mot de passe. Pour l’exploitation, l’essentiel est qu’ils ne doivent figurer ni dans les journaux, ni dans des sauvegardes non chiffrées, ni dans un dépôt.

### Aucun transfert de port vers le PortaSplit

L’erreur évitable la plus fréquente serait de rendre le port local de l’appareil directement accessible depuis Internet. Une règle comme celle-ci serait dangereuse :

```text
Internet → TCP 6444 → PortaSplit
```

Il n’existe aucune bonne raison de rendre le PortaSplit directement accessible depuis Internet. Home Assistant se trouve déjà sur le réseau local et agit comme instance de contrôle. Le routeur ne doit pas posséder de redirection de port vers le PortaSplit, UPnP doit être limité ou désactivé dans la mesure du possible, les connexions entrantes doivent être bloquées par défaut et aucune autorisation DMZ ne doit être utilisée pour l’appareil.

### VLAN IoT dédié

La meilleure architecture réseau consiste à disposer d’un réseau IoT distinct :

```text
VLAN 10: vertrauenswürdige Clients
VLAN 20: Server und Home Assistant
VLAN 30: IoT-Geräte
VLAN 40: Gäste
```

Le PortaSplit se trouve dans le VLAN IoT. Home Assistant peut accéder de manière ciblée à l’appareil, mais le PortaSplit ne doit pas pouvoir accéder librement aux PC, NAS et autres systèmes internes. Une logique de pare-feu possible :

```text
Home Assistant → PortaSplit: erlauben
PortaSplit → Home Assistant: etablierte Verbindungen erlauben
PortaSplit → interne Clients: blockieren
PortaSplit → NAS: blockieren
PortaSplit → Management-Netz: blockieren
Internet → PortaSplit: blockieren
```

Lors de la configuration initiale, l’appareil nécessite un accès à Internet vers le cloud Midea. Une fois la configuration locale réussie, vous pouvez tester si l’accès sortant à Internet peut être bloqué. Il ne faut toutefois pas appliquer immédiatement un blocage définitif. Vérifiez d’abord que le contrôle local fonctionne toujours, que l’appareil reste accessible après un redémarrage, qu’il résiste au redémarrage du routeur, qu’il répond encore après plusieurs jours, que l’application MSmartHome est toujours nécessaire et que les mises à jour du firmware sont encore proposées. Si vous souhaitez continuer à utiliser le cloud et les mises à jour du firmware, vous pouvez autoriser temporairement l’accès sortant à Internet, puis le bloquer de nouveau.

### La segmentation réseau peut empêcher la découverte

La recherche automatique d’appareils repose souvent sur le trafic de diffusion broadcast ou multicast, qui n’est normalement pas routé au-delà des frontières entre VLAN. Home Assistant pourrait donc ne pas détecter automatiquement le PortaSplit, même si une connexion IP normale est autorisée.

Dans ce cas, vous pouvez configurer temporairement le PortaSplit dans le même VLAN que Home Assistant, indiquer manuellement l’adresse IP de l’appareil, utiliser une fonction de relais broadcast appropriée ou définir des règles de pare-feu ciblées après la configuration. La configuration manuelle est même souvent la meilleure variante du point de vue de la sécurité, car elle évite d’autoriser un trafic broadcast supplémentaire entre les réseaux.

### Attribution DHCP statique

Le routeur doit attribuer une réservation DHCP fixe au PortaSplit :

```text
PortaSplit → 192.168.30.25
```

Une réservation DHCP est généralement préférable à une adresse IP statique définie sur l’appareil. Home Assistant détecte ainsi l’appareil de manière fiable, les règles de pare-feu peuvent être limitées à une adresse fixe, l’analyse des erreurs est simplifiée et l’attribution reste stable après le redémarrage du routeur ou de l’appareil. Une règle de pare-feu peut alors être formulée de façon très restrictive :

```text
Home-Assistant-IP → 192.168.30.25:6444/TCP
```

Le port réellement nécessaire doit être vérifié à partir de l’intégration et de votre propre appareil.

### Home Assistant comme point d’ancrage central de confiance

En contrôlant le PortaSplit localement, vous transférez partiellement la confiance du cloud Midea à Home Assistant. Si Home Assistant est compromis, un attaquant peut potentiellement contrôler non seulement le climatiseur, mais aussi l’ensemble de la maison connectée.

Home Assistant doit donc être mis à jour régulièrement, ne pas être exposé par une redirection de port non protégée, être protégé par un mot de passe fort et unique, utiliser l’authentification multifacteur, créer des sauvegardes chiffrées, ne contenir que les add-ons nécessaires et ne pas autoriser d’accès SSH inutile depuis Internet. Pour l’accès à distance, un VPN, Home Assistant Cloud ou un reverse proxy correctement configuré sont de meilleures options qu’une simple redirection vers le port 8123.

### HACS et le risque lié à la chaîne d’approvisionnement

`Midea Smart AC` et `Midea AC LAN` sont des Custom Integrations. Elles s’exécutent au sein de Home Assistant et disposent donc d’un accès étendu à son environnement d’exécution. Une intégration malveillante ou compromise pourrait théoriquement lire des données de configuration, extraire des secrets, établir des connexions réseau, analyser les appareils du réseau local, lire les états d’autres entités, transmettre des données à des systèmes externes et affecter la disponibilité de Home Assistant.

Cela ne signifie pas que les intégrations mentionnées sont malveillantes. Les deux projets sont publics, activement développés et disposent d’une communauté visible. L’open source n’est toutefois pas une garantie de sécurité automatique. Avant l’installation, il vaut au moins la peine de vérifier si le dépôt est activement maintenu, s’il existe des versions régulières, combien de personnes contribuent au code, s’il existe des problèmes de sécurité ouverts, si les responsables ou propriétaires du dépôt ont récemment changé, si HACS renvoie vers le dépôt attendu et si une mise à jour contient des changements inhabituellement importants ou inexplicables.

Les mises à jour ne devraient pas être installées aveuglément dès leur publication. Surtout pour les systèmes domotiques critiques pour la sécurité, il est judicieux d’attendre quelques jours et de consulter les notes de version ainsi que les problèmes signalés.

### Sécuriser le compte cloud

Tant que le cloud Midea est utilisé pour la configuration ou le contrôle via l’application, le compte Midea reste également partie intégrante du modèle de sécurité. Il doit utiliser un mot de passe unique, non partagé avec d’autres services, un gestionnaire de mots de passe, l’authentification multifacteur si elle est proposée, la suppression des anciens smartphones et des sessions, l’absence de comptes partagés et un contrôle régulier des appareils enregistrés dans le compte.

Si l’intégration Home Assistant demande un nom d’utilisateur et un mot de passe lors de la configuration, vérifiez si les identifiants sont utilisés uniquement pour récupérer le token une seule fois ou s’ils sont enregistrés durablement. Les développeurs de `Midea Smart AC` indiquent que les appareils ne sont pas associés à des comptes intégrés à l’intégration après leur configuration et que le token et la clé peuvent également être obtenus manuellement via CLI avec votre propre compte. Lorsque cela est possible, votre propre compte est préférable à des comptes tiers ou à des comptes groupés intégrés.

### Faut-il bloquer le cloud ?

Une fois la configuration réussie, la question se pose de savoir si l’accès à Internet du PortaSplit doit être entièrement bloqué. En faveur d’un blocage : moins de télémétrie, une dépendance moindre aux services externes, une surface d’attaque réduite via le cloud du fabricant, le fait que l’appareil ne peut pas contacter des destinations externes arbitraires et un impact moindre des changements côté cloud.

En revanche, l’application MSmartHome peut ne plus fonctionner en dehors du réseau domestique, les mises à jour du firmware peuvent ne plus être téléchargées, les fonctions d’horloge ou cloud peuvent cesser de fonctionner, une nouvelle connexion ou une restauration peut devenir plus difficile et certains appareils peuvent réagir de manière inattendue après une longue période hors ligne.

Une séquence pragmatique : configurez normalement l’appareil, testez Home Assistant et l’application, sauvegardez le token et la configuration, bloquez l’accès à Internet, redémarrez l’appareil et Home Assistant, observez pendant plusieurs jours et, si nécessaire, ne réactivez l’accès à Internet que temporairement.

### Mises à jour du firmware : gain de sécurité ou risque pour l’intégration ?

Les mises à jour du firmware constituent un dilemme pour les appareils IoT. Elles peuvent corriger des vulnérabilités connues, améliorer la stabilité, moderniser les mécanismes de sécurité et apporter de nouvelles fonctions. Mais elles peuvent aussi modifier les interfaces locales, casser les intégrations issues du rétro-ingénierie, invalider les tokens, désactiver l’API locale et introduire de nouvelles dépendances au cloud.

Le firmware PortaSplit distribué en janvier 2026 a par exemple apporté un nouveau mode silencieux pour l’unité extérieure, qui réduit le bruit d’environ 6 décibels. Les intégrations communautaires ont d’abord dû l’analyser et l’implémenter, comme le documente une issue GitHub spécifique au PortaSplit.

Il en résulte qu’il ne faut pas empêcher systématiquement les mises à jour du firmware, mais vérifier avant une mise à jour si d’autres utilisateurs de Home Assistant signalent des problèmes, sauvegarder au préalable la configuration et le token, créer une sauvegarde de Home Assistant et tester entièrement le contrôle local après la mise à jour. La sécurité ne signifie pas « ne jamais mettre à jour ». Un firmware obsolète peut être plus dangereux qu’une intégration temporairement incompatible.

### Les journaux de débogage contiennent des données sensibles

En cas de problème, les projets open source demandent fréquemment des journaux de débogage. La documentation de `Midea AC LAN` montre comment activer la journalisation pour les deux composants concernés :

```yaml
logger:
  default: warn
  logs:
    custom_components.midea_ac_lan: debug
    midealocal: debug
```

Les journaux peuvent ensuite être téléchargés via Paramètres, Système et Journaux. Selon l’intégration et le type d’erreur, ces journaux peuvent contenir des adresses IP locales, l’ID de l’appareil, le numéro de série, l’identifiant de modèle, des réponses du cloud, des informations de compte, le token ou une partie de celui-ci, des paquets réseau ainsi que des horodatages et des habitudes d’utilisation. Avant de les envoyer dans une issue GitHub publique, ils doivent donc être vérifiés et les valeurs sensibles doivent être masquées.

Une fois le dépannage terminé, la journalisation de débogage doit être supprimée. Une journalisation de débogage active en permanence augmente non seulement l’utilisation du stockage, mais aussi la quantité d’informations sensibles dans les sauvegardes.

### Ce que Midea dit lui-même sur la sécurité

Midea promeut son écosystème SmartHome en indiquant son alignement sur plusieurs normes de sécurité et de protection des données, dont EN 303 645, UK PSTI, NIST, le traitement des données conforme au RGPD et les exigences de la directive européenne sur les équipements radio. Ce sont des signaux positifs, mais ils ne disent rien de la manière dont chaque firmware PortaSplit, chaque point de terminaison cloud et chaque API locale sont réellement implémentés. Les déclarations de certification et de marketing ne remplacent pas un examen technique de l’appareil concret.

De même, il serait erroné de déduire de l’avertissement d’une intégration communautaire que le PortaSplit est généralement peu sûr. Le problème décrit concerne l’architecture des tokens à longue durée de vie et leur utilisation par des clients non officiels.

### Risque selon le scénario

| Scénario | Risque | Justification |
| --- | --- | --- |
| Réseau domestique normal sans redirection de port | modéré | Un attaquant doit d’abord accéder au Wi-Fi, à Home Assistant ou à une sauvegarde. |
| Réseau domestique plat avec de nombreux appareils IoT non sécurisés | moyen | Un autre appareil IoT compromis peut atteindre le PortaSplit ou Home Assistant sur le même réseau. |
| PortaSplit directement accessible depuis Internet | élevé | L’appareil ne doit jamais être exposé par redirection de port. |
| Token et clé publics sur GitHub | élevé | Les secrets doivent être considérés comme compromis ; leur révocation n’est pas garantie. |
| VLAN IoT séparé, pare-feu restrictif, contrôle local | relativement faible | Même en cas de vulnérabilité dans l’appareil, la liberté de mouvement sur le réseau est fortement limitée. |

## Sauvegarde de la configuration

La sauvegarde du token, de la clé et de la configuration est l’étape ponctuelle la plus importante : une fois les interfaces de tokens cloud fermées, une sauvegarde est le seul moyen de procéder à une nouvelle configuration. `Midea AC LAN` crée après une configuration réussie un fichier de configuration JSON pour les appareils V3. Le chemin documenté est :

```text
/config/.storage/midea_ac_lan/
```

Le fichier porte l’ID de l’appareil comme nom de fichier :

```text
<device-id>.json
```

Ce fichier n’est pas une simple note textuelle. Il peut contenir l’ID de l’appareil, le numéro de série, l’adresse IP, le token, la clé, des informations de protocole ainsi que des paramètres cloud et appareil. Par conséquent :

- Ne le téléversez pas dans un dépôt GitHub public.
- Ne le publiez pas sur des forums.
- Ne le partagez pas sous forme de capture d’écran non masquée.
- Ne l’envoyez pas par e-mail non chiffré.

Même un dépôt Git privé n’est pas automatiquement le bon emplacement de stockage, car les secrets restent dans l’historique Git, même s’ils sont ultérieurement supprimés du fichier actuel. Une sauvegarde chiffrée, un gestionnaire de mots de passe avec pièce jointe, une sauvegarde NAS chiffrée, un support hors ligne chiffré ou une archive chiffrée dont le mot de passe est stocké séparément conviennent mieux.

Pour effectuer une sauvegarde via le terminal Home Assistant :

```bash
cd /config/.storage/midea_ac_lan
ls -la
```

Afficher le fichier :

```bash
cat <device-id>.json
```

Pour le copier, le fichier ne doit pas être transféré via un service Web public. Il est préférable de créer une archive chiffrée, puis de la placer dans une sauvegarde chiffrée :

```bash
tar -czf /config/midea-ac-lan-backup.tar.gz \
  /config/.storage/midea_ac_lan
```

Les fichiers dans `.storage` ne doivent pas être modifiés manuellement. Le développeur recommande expressément de ne pas supprimer ni modifier directement le fichier JSON en cas de problème, mais de le renommer et de le sauvegarder avant toute modification.

Une sauvegarde complète de Home Assistant contient également ces fichiers. Une copie distincte reste néanmoins utile, car les sauvegardes Home Assistant peuvent être endommagées, une restauration peut écraser l’intégration, le fichier peut être nécessaire spécifiquement pour une nouvelle configuration ultérieure et une sauvegarde ne doit jamais se trouver uniquement sur le même système.

## Retirer des secrets d’un dépôt Git publié

Si un fichier JSON a été publié par erreur sur GitHub, une suppression normale suivie d’un nouveau commit ne suffit pas. Le fichier reste accessible dans l’historique Git. Les étapes suivantes sont au minimum nécessaires :

1. Rendre immédiatement le dépôt privé, si possible.
2. Retirer le fichier de l’ensemble de l’historique Git.
3. Tenir compte des caches et des forks GitHub.
4. Considérer le token comme compromis.
5. Retirer l’appareil du compte Midea et le reconnecter, si cela génère de nouvelles clés.
6. Reconfigurer l’intégration Home Assistant.
7. Modifier le mot de passe du compte Midea si les identifiants étaient également concernés.

Le fait qu’un nouvel appairage génère réellement un nouveau token varie selon l’appareil et l’architecture cloud. Il ne faut pas compter sur le fait que la modification du mot de passe du compte invalide automatiquement le token local de l’appareil.

## Automatisations utiles

Après une intégration réussie, le PortaSplit peut être exploité de manière nettement plus intelligente. Les ID d’entité doivent être adaptés à votre propre installation.

Refroidir uniquement lorsque les fenêtres sont fermées :

```yaml
alias: PortaSplit nur bei geschlossenen Fenstern
triggers:
  - trigger: state
    entity_id: binary_sensor.wohnzimmer_fenster
    to: "on"

actions:
  - action: climate.turn_off
    target:
      entity_id: climate.portasplit
```

Allumer lorsque la température ambiante est élevée :

```yaml
alias: PortaSplit bei Hitze einschalten
triggers:
  - trigger: numeric_state
    entity_id: sensor.wohnzimmer_temperatur
    above: 27

conditions:
  - condition: state
    entity_id: binary_sensor.wohnzimmer_fenster
    state: "off"
  - condition: state
    entity_id: person.rafael
    state: "home"

actions:
  - action: climate.set_hvac_mode
    target:
      entity_id: climate.portasplit
    data:
      hvac_mode: cool

  - action: climate.set_temperature
    target:
      entity_id: climate.portasplit
    data:
      temperature: 24
```

Prérefroidir avant le coucher :

```yaml
alias: Schlafzimmer vorkühlen
triggers:
  - trigger: time
    at: "21:00:00"

conditions:
  - condition: numeric_state
    entity_id: sensor.schlafzimmer_temperatur
    above: 25

actions:
  - action: climate.set_temperature
    target:
      entity_id: climate.portasplit
    data:
      temperature: 23
```

Éteindre lorsqu’il n’y a personne à la maison :

```yaml
alias: PortaSplit bei Abwesenheit ausschalten
triggers:
  - trigger: state
    entity_id: zone.home
    to: "0"
    for:
      minutes: 10

actions:
  - action: climate.turn_off
    target:
      entity_id: climate.portasplit
```

## Configuration recommandée en un coup d’œil

```text
1. PortaSplit mit MSmartHome einrichten
2. Midea Smart AC über HACS installieren
3. PortaSplit automatisch oder manuell hinzufügen
4. DHCP-Reservation erstellen
5. Home-Assistant-Backup anfertigen
6. Token- und Konfigurationsdaten verschlüsselt sichern
7. PortaSplit in ein separates IoT-VLAN verschieben
8. Zugriff von Home Assistant zur PortaSplit erlauben
9. Zugriff der PortaSplit auf interne Netze blockieren
10. Internetzugriff testweise blockieren
11. lokale Steuerung nach Neustarts prüfen
12. Firmware- und Integrationsupdates kontrolliert durchführen
```

La direction de communication souhaitée est donc la suivante :

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

## État de fonctionnement recommandé

Le Midea PortaSplit s’intègre bien à Home Assistant. Une fois la configuration réussie, il peut être contrôlé localement et intégré dans des automatisations, ce qui supprime une grande partie de la dépendance au cloud pour l’utilisation quotidienne.

Du point de vue de la sécurité, l’intégration est défendable si quelques règles de base sont respectées : aucune redirection de port, garder le token et la clé secrets, chiffrer les sauvegardes, vérifier les journaux de débogage avant publication, sécuriser Home Assistant, segmenter les appareils IoT, limiter l’accès sortant à Internet au nécessaire et ne pas installer aveuglément les mises à jour du firmware et de HACS. Utilisé ainsi, le PortaSplit reste un climatiseur performant tout en devenant un élément intégrable de façon pertinente dans une maison connectée contrôlée localement.

## Sources

1.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: intégration `Midea Smart AC` : types d’appareils pris en charge `0xAC` et `0xCC`, PortaSplit avec « Out Silent Mode », utilisation du cloud pour obtenir le token et la clé pour les appareils V3, configuration manuelle et port standard 6444.

2.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: intégration `Midea AC LAN` : catégories d’appareils prises en charge, connexion TCP plus longue pour la synchronisation de l’état et version minimale Home Assistant 2024.4.1.

3.  [midea_ac_lan : documentation des entités de climatisation](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/AC.md): entités et attributs pour climatiseurs, notamment la puissance, l’énergie totale, la fréquence du compresseur et les méthodes de décodage des valeurs énergétiques de certains sous-types.

4.  [midea_ac_lan : indications de débogage et de configuration](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/debug.md): emplacement de la configuration de l’appareil sous `/config/.storage/midea_ac_lan/`, recommandation de sauvegarder plutôt que de supprimer le fichier JSON et configuration des loggers pour les journaux de débogage.

5.  [Issue 779 : Out Silent Mode du PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/779): demande de prise en charge du mode silencieux de l’unité extérieure introduit par la mise à jour du firmware de janvier 2026, qui réduit le bruit d’environ 6 décibels.

6.  [Midea SmartHome](https://www.midea.com/global/smarthome): informations du fabricant sur les normes de sécurité et de protection des données EN 303 645, PSTI, NIST, RGPD et RED DA.

7.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): installation et gestion de Custom Integrations qui ne font pas partie de Home Assistant Core.
