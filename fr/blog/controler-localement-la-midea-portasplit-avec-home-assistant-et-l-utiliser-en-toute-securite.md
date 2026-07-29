---
title: "Contrôler localement la Midea PortaSplit avec Home Assistant et l’utiliser en toute sécurité"
navTitle: "Configurer PortaSplit"
description: "De l’intégration communautaire appropriée au VLAN IoT : comment configurer la PortaSplit, protéger le token et la clé, et limiter les accès au cloud et au réseau."
date: "2026-07-24"
kategorie: "Maison intelligente et IoT"
timeToRead: "14 min de lecture"
themen:
  - "smart-home-iot"
related:
  - "midea-portasplit-home-assistant"
  - "serverloser-newsletter-cloudflare-workers-d1"
slug: "controler-localement-la-midea-portasplit-avec-home-assistant-et-l-utiliser-en-toute-securite"
translationOf: "midea-portasplit-home-assistant-einrichten"
url: "https://rafaelpfister.ch/fr/blog/controler-localement-la-midea-portasplit-avec-home-assistant-et-l-utiliser-en-toute-securite"
---

La Midea PortaSplit peut être contrôlée directement sur le réseau local via Home Assistant après sa configuration. Pour cela, l’intégration communautaire nécessite deux identifiants spécifiques à l’appareil provenant du cloud Midea : un token et une clé.

Cet article présente la sélection, la configuration et la sécurisation de l’intégration. Les solutions décrites proviennent de la communauté et ne sont officiellement prises en charge ni par Midea ni par Home Assistant. Des modifications du firmware ou du cloud peuvent donc influencer leur fonctionnement à tout moment. Les détails concernant l’interface des tokens et l’avertissement ambigu relatif à son arrêt figurent dans [l’analyse des API cloud de Midea](/blog/midea-v2-cloud-api-portasplit-home-assistant).

## Comment fonctionne le contrôle local

Après la configuration, les commandes de contrôle sont envoyées directement par Home Assistant à la PortaSplit :

```text
Home Assistant → lokales Netzwerk → Midea PortaSplit
```

Une commande de commutation ne doit pas passer par un serveur Midea externe, le temps de réponse est court, une panne du cloud Midea ne paralyse pas nécessairement le contrôle local déjà configuré, et l’appareil reste en principe contrôlable même sans accès à Internet.

Toutefois, sur les appareils récents utilisant le protocole dit V3, la PortaSplit n’accepte pas les commandes locales sans protection. Home Assistant a besoin de deux valeurs spécifiques à l’appareil, un token et une clé, utilisés pour l’authentification et le chiffrement de la connexion locale. Lors de la première configuration, l’intégration les récupère une fois via une interface cloud Midea, puis les stocke localement ; aucune connexion cloud n’est nécessaire pour le contrôle ultérieur.

De manière simplifiée, le processus se déroule ainsi :

1. La PortaSplit est connectée à MSmartHome.
2. Home Assistant se connecte à un cloud Midea.
3. Home Assistant obtient l’ID de l’appareil, le token et la clé.
4. Le token et la clé sont stockés localement.
5. Home Assistant contrôle la PortaSplit directement sur le LAN.

## Quelle intégration choisir

### Midea Smart AC

Le dépôt <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> se concentre sur les climatiseurs Midea et les modèles OEM associés, et prend en charge les types d’appareils `0xAC` et `0xCC`. Il propose un contrôle local, une configuration graphique, une détection automatique, une configuration manuelle avec token et clé, ainsi qu’une interrogation automatique des capacités de l’appareil. Le « Out Silent Mode » de la PortaSplit est explicitement pris en charge.

Comme indice de compatibilité, le projet mentionne notamment les applications Artic King, Midea Air, NetHome Plus, SmartHome ou MSmartHome, Toshiba AC NA et 美的美居. La PortaSplit utilise généralement MSmartHome en Europe et s’inscrit donc dans cet écosystème.

### Midea AC LAN

Le dépôt <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> prend en charge non seulement les climatiseurs, mais aussi de nombreuses autres catégories d’appareils Midea : déshumidificateurs, ventilateurs, purificateurs d’air, lave-linge, sèche-linge, lave-vaisselle, chauffe-eau, pompes à chaleur, réfrigérateurs et bien d’autres, parfois aussi sous des marques tierces telles que Carrier ou Electrolux. Il propose également une communication locale, une détection automatique des appareils et des capteurs supplémentaires ; selon la description du projet, il maintient une connexion TCP plus longue avec l’appareil afin de synchroniser rapidement les changements d’état. Home Assistant 2024.4.1 au minimum est requis.

Le principal inconvénient actuel est l’avertissement du développeur : les API cloud de token utilisées pour ajouter de nouveaux appareils sont progressivement désactivées. L’ajout ultérieur de nouveaux appareils peut ainsi devenir impossible.

### Recommandation

Pour une installation PortaSplit seule, je commencerais par `Midea Smart AC` et garderais `Midea AC LAN` comme alternative à l’esprit. `Midea Smart AC` est davantage spécialisé dans les climatiseurs et documente explicitement les fonctions actuelles de la PortaSplit.

Il n’est pas judicieux d’utiliser simultanément et durablement les deux intégrations avec le même appareil. Plusieurs connexions parallèles entraînent des problèmes d’état, du trafic réseau inutile et un comportement difficile à comprendre.

## Ce que l’intégration apporte

Après la configuration, la PortaSplit apparaît comme une entité `climate` dans Home Assistant. Selon le firmware et l’intégration, les fonctions suivantes sont notamment disponibles :

- Mise sous et hors tension
- Réglage de la température de consigne
- Lecture de la température ambiante actuelle
- Refroidissement, déshumidification et ventilation seule
- Réglage de la vitesse du ventilateur
- Contrôle de la fonction Swing
- Modes Eco et Boost
- Lecture de l’humidité de l’air
- Affichage des codes d’erreur
- Lecture des valeurs d’énergie et de puissance
- Affichage des valeurs du compresseur
- Activation du mode silencieux de l’unité extérieure

Les entités qui apparaissent réellement dépendent du modèle, du firmware, du protocole utilisé et de l’intégration concernée. `Midea Smart AC` interroge les capacités rapportées par l’appareil et masque les fonctions que le modèle ne prend pas en charge. `Midea AC LAN` documente également de nombreuses entités de climatisation, notamment la température, l’humidité, la puissance actuelle, l’énergie totale, la fréquence du compresseur, l’état de la pompe et divers modes de fonctionnement, et indique des méthodes propres à certains sous-types de PortaSplit pour décoder les données énergétiques.

Toutes les mesures affichées ne sont pas forcément correctes. La consommation d’énergie et la puissance, en particulier, sont transmises dans différents formats selon les modèles Midea. Si Home Assistant affiche des valeurs manifestement erronées, il faut généralement adapter la méthode de décodage utilisée ; l’appareil n’est pas forcément défectueux.

## Prérequis

Il faut une Midea PortaSplit dotée du Wi-Fi, un réseau Wi-Fi 2,4 GHz, l’application MSmartHome, un compte utilisateur Midea, Home Assistant, HACS et un accès réseau entre Home Assistant et la PortaSplit. La PortaSplit doit d’abord être connectée normalement via l’application MSmartHome, puis ajoutée à Home Assistant.

## Étape 1 : connecter la PortaSplit à MSmartHome

1. Installer l’application MSmartHome.
2. Créer un compte Midea ou se connecter.
3. Mettre la PortaSplit en mode d’appairage Wi-Fi.
4. Connecter l’appareil au Wi-Fi 2,4 GHz.
5. Vérifier que la PortaSplit peut être contrôlée depuis l’application.

De nombreux appareils IoT ne prennent toujours en charge que le 2,4 GHz. Si le routeur utilise le même SSID pour le 2,4 et le 5 GHz, la configuration fonctionne généralement malgré tout. En cas de problème, il peut être utile de fournir temporairement un réseau Wi-Fi 2,4 GHz distinct.

## Étape 2 : installer HACS

HACS est le magasin communautaire de Home Assistant. Il permet d’installer des intégrations communautaires qui ne font pas partie de Home Assistant Core. Après l’installation de HACS, ouvrez HACS, accédez aux intégrations, recherchez `Midea Smart AC`, téléchargez l’intégration et redémarrez Home Assistant. Vous pouvez aussi rechercher `Midea AC LAN`.

HACS simplifie l’installation et les mises à jour. Il ne transforme toutefois pas une intégration personnalisée en composant Home Assistant officiellement vérifié. Cette distinction est importante du point de vue de la sécurité et sera abordée plus loin.

## Étape 3 : ajouter Midea Smart AC

Après le redémarrage, allez dans Paramètres, Appareils et services, puis Ajouter une intégration, recherchez `Midea Smart AC`, puis `Discover devices`. L’intégration peut soit parcourir l’ensemble du réseau local, soit interroger directement l’adresse IP de la PortaSplit.

Lorsque l’appareil est trouvé, l’intégration requiert, pour les appareils V3 récents, la région, le compte Midea, le mot de passe et l’ID de l’appareil, ainsi que le token et la clé qui en sont dérivés. La région cloud doit correspondre au compte utilisé. En cas de problème, le projet recommande également d’essayer les autres régions proposées.

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

Pour les appareils V3, la documentation indique que le token est une chaîne hexadécimale de 128 caractères et la clé une chaîne hexadécimale de 64 caractères. Ces deux valeurs sont des secrets et doivent être traitées comme tels. Si vous ne souhaitez pas obtenir les identifiants via la détection, vous pouvez les récupérer avec votre propre compte via l’interface CLI `msmart-ng`.

## Utiliser la PortaSplit en toute sécurité

Contrôler la PortaSplit localement permet de reprendre une partie du contrôle au cloud du fabricant, mais déplace aussi la responsabilité vers votre propre réseau. Les points suivants garantissent que le token et la clé causent peu de dégâts même en cas d’incident et que l’appareil reste correctement isolé.

### Le token et la clé sont des secrets

Le token et la clé authentifient la communication locale avec l’appareil et doivent être traités comme un mot de passe. Pour l’exploitation, l’essentiel est le suivant : ils ne doivent pas apparaître dans les journaux, les sauvegardes non chiffrées ni un dépôt.

### Aucun transfert de port vers la PortaSplit

L’erreur évitable la plus fréquente serait de rendre le port local de l’appareil directement accessible depuis Internet. Une règle telle que celle-ci serait dangereuse :

```text
Internet → TCP 6444 → PortaSplit
```

Il n’existe aucune bonne raison de rendre la PortaSplit directement accessible depuis Internet. Home Assistant se trouve déjà sur le réseau local et sert d’instance de contrôle. Le routeur ne devrait disposer d’aucune redirection de port vers la PortaSplit, UPnP devrait être limité ou désactivé si possible, les connexions entrantes devraient être bloquées par défaut et aucune autorisation DMZ ne devrait être utilisée pour l’appareil.

### VLAN IoT dédié

La meilleure architecture réseau consiste à utiliser un réseau IoT distinct :

```text
VLAN 10: vertrauenswürdige Clients
VLAN 20: Server und Home Assistant
VLAN 30: IoT-Geräte
VLAN 40: Gäste
```

La PortaSplit se trouve dans le VLAN IoT. Home Assistant peut accéder spécifiquement à l’appareil, mais la PortaSplit ne peut pas accéder librement aux PC, NAS et autres systèmes internes. Une logique de pare-feu possible :

```text
Home Assistant → PortaSplit: erlauben
PortaSplit → Home Assistant: etablierte Verbindungen erlauben
PortaSplit → interne Clients: blockieren
PortaSplit → NAS: blockieren
PortaSplit → Management-Netz: blockieren
Internet → PortaSplit: blockieren
```

Lors de la première configuration, l’appareil nécessite un accès à Internet vers le cloud Midea. Une fois la configuration locale réussie, vous pouvez tester si l’accès sortant à Internet peut être bloqué. Il ne faut toutefois pas imposer immédiatement un blocage définitif. Vérifiez d’abord que le contrôle local fonctionne toujours, que l’appareil reste accessible après un redémarrage, qu’il résiste à un redémarrage du routeur, qu’il répond toujours après plusieurs jours, que l’application MSmartHome est encore nécessaire et que les mises à jour de firmware sont toujours proposées. Si vous souhaitez continuer à utiliser le cloud et les mises à jour du firmware, vous pouvez autoriser temporairement l’accès sortant à Internet, puis le bloquer à nouveau.

### La segmentation réseau peut empêcher la détection

La recherche automatique d’appareils repose souvent sur du trafic broadcast ou multicast, qui n’est normalement pas routé à travers les frontières de VLAN. Home Assistant peut donc ne pas détecter automatiquement la PortaSplit, même si une connexion IP normale serait autorisée.

Dans ce cas, vous pouvez configurer temporairement la PortaSplit dans le même VLAN que Home Assistant, indiquer manuellement l’adresse IP de l’appareil, utiliser une fonction de relais broadcast appropriée ou définir des règles de pare-feu ciblées après la configuration. La configuration manuelle est souvent même la meilleure option du point de vue de la sécurité, car elle ne nécessite pas d’autoriser du trafic broadcast supplémentaire entre les réseaux.

### Attribution DHCP statique

La PortaSplit devrait recevoir une attribution DHCP fixe dans le routeur :

```text
PortaSplit → 192.168.30.25
```

Une réservation DHCP est généralement préférable à une adresse IP statique définie dans l’appareil. Home Assistant trouve l’appareil de manière fiable, les règles de pare-feu peuvent être limitées à une adresse fixe, l’analyse des erreurs est simplifiée et l’attribution reste stable après le redémarrage du routeur ou de l’appareil. Une règle de pare-feu peut ainsi être formulée de manière très restrictive :

```text
Home-Assistant-IP → 192.168.30.25:6444/TCP
```

Le port réellement requis doit être vérifié à partir de l’intégration et de votre propre appareil.

### Home Assistant comme ancrage de confiance central

Contrôler la PortaSplit localement revient à déplacer une partie de la confiance du cloud Midea vers Home Assistant. Si Home Assistant est compromis, un attaquant peut potentiellement contrôler non seulement le climatiseur, mais aussi l’ensemble de la maison intelligente.

Home Assistant doit donc être mis à jour régulièrement, ne pas être exposé par une redirection de port non protégée, être protégé par un mot de passe fort et unique, utiliser l’authentification multifacteur, créer des sauvegardes chiffrées, ne contenir que les modules complémentaires nécessaires et ne pas autoriser d’accès SSH inutile depuis Internet. Pour l’accès à distance, un VPN, Home Assistant Cloud ou un proxy inverse correctement configuré sont de meilleures options qu’une simple redirection de port vers le port 8123.

### HACS et le risque de chaîne d’approvisionnement

`Midea Smart AC` et `Midea AC LAN` sont des intégrations personnalisées. Elles s’exécutent au sein de Home Assistant et bénéficient donc d’un accès étendu à son environnement d’exécution. Une intégration malveillante ou compromise pourrait théoriquement lire des données de configuration, extraire des secrets, établir des connexions réseau, analyser les appareils du réseau local, lire les états d’autres entités, transmettre des données à des systèmes externes et nuire à la disponibilité de Home Assistant.

Cela ne signifie pas que les intégrations mentionnées sont malveillantes. Les deux projets sont publics, activement développés et disposent d’une communauté visible. L’open source n’est toutefois pas une garantie de sécurité automatique. Avant l’installation, il vaut au moins la peine de vérifier si le dépôt est activement maintenu, s’il existe des publications régulières, combien de personnes contribuent au code, s’il existe des problèmes de sécurité ouverts, si les responsables de maintenance ou les propriétaires du dépôt ont récemment changé, si HACS renvoie vers le dépôt attendu et si une mise à jour contient des modifications inhabituellement importantes ou inexplicables.

Les mises à jour ne devraient pas être installées aveuglément dès leur publication. Pour les systèmes de maison intelligente sensibles en matière de sécurité, il est judicieux d’attendre quelques jours et de consulter les notes de version ainsi que les problèmes signalés.

### Sécuriser le compte cloud

Tant que le cloud Midea est utilisé pour la configuration ou le contrôle par l’application, le compte Midea reste lui aussi une partie du modèle de sécurité. Il doit disposer d’un mot de passe unique, non partagé avec d’autres services, d’un gestionnaire de mots de passe, de l’authentification multifacteur lorsqu’elle est proposée, de la suppression des anciens smartphones et des sessions, de l’absence de comptes partagés et d’un contrôle régulier des appareils enregistrés dans le compte.

Si l’intégration Home Assistant demande un nom d’utilisateur et un mot de passe pendant la configuration, vérifiez si les identifiants servent uniquement à récupérer le token une fois ou s’ils sont stockés durablement. Les développeurs de `Midea Smart AC` indiquent que les appareils ne sont pas associés à des comptes d’intégration intégrés après la configuration et que le token et la clé peuvent aussi être obtenus manuellement via votre propre compte avec la CLI. Lorsque cela est possible, votre propre compte doit être préféré à des comptes tiers ou à des comptes mutualisés intégrés.

### Bloquer le cloud ou non ?

Après une configuration réussie, la question se pose de savoir si l’accès Internet de la PortaSplit doit être entièrement bloqué. En faveur d’un blocage : moins de télémétrie, une dépendance réduite aux services externes, une surface d’attaque réduite via le cloud du fabricant, le fait que l’appareil ne peut contacter aucune destination externe arbitraire et un impact moindre des changements côté cloud.

À l’inverse, l’application MSmartHome peut ne plus fonctionner hors du réseau domestique, les mises à jour de firmware ne peuvent plus être téléchargées, les fonctions d’horloge ou de cloud peuvent cesser de fonctionner, une nouvelle connexion ou une restauration peut devenir plus difficile et certains appareils peuvent réagir de manière inattendue après une longue période hors ligne.

Une approche pragmatique : configurer normalement l’appareil, tester Home Assistant et l’application, sauvegarder le token et la configuration, bloquer l’accès à Internet, redémarrer l’appareil et Home Assistant, observer pendant plusieurs jours et, si nécessaire, réautoriser l’accès à Internet uniquement de manière temporaire.

### Mises à jour du firmware : gain de sécurité ou risque pour l’intégration ?

Les mises à jour du firmware constituent un dilemme pour les appareils IoT. Elles peuvent corriger des vulnérabilités connues, améliorer la stabilité, moderniser les mécanismes de sécurité et apporter de nouvelles fonctions. Mais elles peuvent aussi modifier les interfaces locales, casser les intégrations issues de rétro-ingénierie, invalider les tokens, désactiver l’API locale et introduire de nouvelles dépendances au cloud.

Le firmware PortaSplit livré en janvier 2026 a par exemple apporté un nouveau mode silencieux pour l’unité extérieure, réduisant le bruit d’environ 6 décibels. Les intégrations communautaires ont d’abord dû l’analyser et l’implémenter, ce qui est documenté dans une issue GitHub dédiée à la PortaSplit.

Il en résulte que les mises à jour de firmware ne doivent pas être empêchées par principe : avant une mise à jour, vérifiez si d’autres utilisateurs de Home Assistant signalent des problèmes, sauvegardez au préalable la configuration et le token, créez une sauvegarde de Home Assistant et testez complètement le contrôle local après la mise à jour. La sécurité ne signifie pas « ne jamais mettre à jour ». Un firmware obsolète peut être plus dangereux qu’une intégration temporairement incompatible.

### Les journaux de débogage contiennent des données sensibles

En cas de problème, les projets open source demandent souvent des journaux de débogage. La documentation de `Midea AC LAN` indique comment activer la journalisation pour les deux composants concernés :

```yaml
logger:
  default: warn
  logs:
    custom_components.midea_ac_lan: debug
    midealocal: debug
```

Les journaux peuvent ensuite être téléchargés via Paramètres, Système et Journaux. Selon l’intégration et le cas d’erreur, ces journaux peuvent contenir des adresses IP locales, l’ID de l’appareil, le numéro de série, l’identifiant du modèle, des réponses du cloud, des informations de compte, le token ou des parties de celui-ci, des paquets réseau ainsi que des horodatages et des informations d’usage. Avant de les téléverser dans une issue GitHub publique, ils doivent donc être vérifiés et les valeurs sensibles doivent être masquées.

Une fois le dépannage terminé, la journalisation de débogage doit être désactivée. Une journalisation de débogage active en permanence augmente non seulement l’utilisation du stockage, mais aussi la quantité d’informations sensibles dans les sauvegardes.

### Ce que Midea indique elle-même au sujet de la sécurité

Midea fait la promotion de son écosystème SmartHome en indiquant s’aligner sur plusieurs normes de sécurité et de protection des données, notamment EN 303 645, UK PSTI, NIST, le traitement des données conforme au RGPD et les exigences de la directive européenne sur les équipements radio. Ce sont des signaux positifs, mais ils ne disent rien sur la manière dont chaque firmware PortaSplit, chaque point de terminaison cloud et chaque API locale sont réellement implémentés. Les affirmations de certification et de marketing ne remplacent pas une vérification technique de l’appareil concret.

De même, il serait erroné de déduire de l’avertissement d’une intégration communautaire que la PortaSplit est globalement non sécurisée. Le problème décrit concerne l’architecture des tokens durables et leur utilisation par des clients non officiels.

### Risque selon le scénario

| Scénario | Risque | Justification |
| --- | --- | --- |
| Réseau domestique normal sans redirection de port | modéré | Un attaquant doit d’abord obtenir l’accès au Wi-Fi, à Home Assistant ou à une sauvegarde. |
| Réseau domestique plat avec de nombreux appareils IoT non sécurisés | moyen | Un autre appareil IoT compromis peut atteindre la PortaSplit ou Home Assistant sur le même réseau. |
| PortaSplit directement accessible depuis Internet | élevé | L’appareil ne doit jamais être exposé par redirection de port. |
| Token et clé publics sur GitHub | élevé | Les secrets doivent être considérés comme compromis ; il n’est pas garanti qu’ils puissent être révoqués. |
| VLAN IoT séparé, pare-feu restrictif, contrôle local | relativement faible | Même en cas de vulnérabilité dans l’appareil, sa liberté de mouvement sur le réseau est fortement limitée. |

## Sauvegarde de la configuration

La sauvegarde du token, de la clé et de la configuration est l’étape unique la plus importante : une fois les interfaces cloud de tokens fermées, une sauvegarde est le seul moyen de procéder à une nouvelle configuration. `Midea AC LAN` enregistre un fichier de configuration JSON pour les appareils V3 après une configuration réussie. Le chemin documenté est :

```text
/config/.storage/midea_ac_lan/
```

Le fichier porte l’ID de l’appareil comme nom de fichier :

```text
<device-id>.json
```

Ce fichier n’est pas une simple note de texte. Il peut contenir l’ID de l’appareil, le numéro de série, l’adresse IP, le token, la clé, des informations de protocole ainsi que des paramètres cloud et appareil. En conséquence :

- Ne pas le téléverser dans un dépôt GitHub public.
- Ne pas le publier dans des forums.
- Ne pas le partager sous forme de capture d’écran non masquée.
- Ne pas l’envoyer par e-mail non chiffré.

Même un dépôt Git privé n’est pas automatiquement le bon emplacement de stockage, car les secrets restent dans l’historique Git, même s’ils sont ensuite supprimés du fichier actuel. Une sauvegarde chiffrée, un gestionnaire de mots de passe avec pièce jointe, une sauvegarde NAS chiffrée, un support hors ligne chiffré ou une archive chiffrée avec mot de passe stocké séparément sont plus appropriés.

Pour effectuer la sauvegarde via le terminal Home Assistant :

```bash
cd /config/.storage/midea_ac_lan
ls -la
```

Afficher le fichier :

```bash
cat <device-id>.json
```

Pour la copie, le fichier ne doit pas être transféré via un service web public. Une archive chiffrée, ensuite placée dans une sauvegarde chiffrée, est préférable :

```bash
tar -czf /config/midea-ac-lan-backup.tar.gz \
  /config/.storage/midea_ac_lan
```

Les fichiers dans `.storage` ne doivent pas être modifiés manuellement. Le développeur recommande explicitement de ne ni supprimer ni modifier directement le fichier JSON en cas de problème, mais de le renommer et de le sauvegarder avant toute modification.

Une sauvegarde complète de Home Assistant contient également ces fichiers. Une copie séparée reste toutefois judicieuse, car les sauvegardes Home Assistant peuvent être endommagées, une restauration peut écraser l’intégration, le fichier peut être nécessaire spécifiquement pour une nouvelle configuration ultérieure et une sauvegarde ne devrait jamais se trouver uniquement sur le même système.

## Retirer des secrets d’un dépôt Git publié

Si un fichier JSON a été publié par erreur sur GitHub, une suppression normale suivie d’un nouveau commit ne suffit pas. Le fichier reste accessible dans l’historique Git. Au minimum, les étapes suivantes sont nécessaires :

1. Rendre immédiatement le dépôt privé, si possible.
2. Retirer le fichier de l’ensemble de l’historique Git.
3. Tenir compte des caches GitHub et des forks.
4. Considérer le token comme compromis.
5. Retirer l’appareil du compte Midea et le reconnecter si cela génère de nouvelles clés.
6. Reconfigurer l’intégration Home Assistant.
7. Modifier le mot de passe du compte Midea si les identifiants ont également été concernés.

La génération effective d’un nouveau token lors d’un nouvel appairage varie selon l’appareil et l’architecture cloud. Il ne faut pas compter sur le fait que la modification du mot de passe du compte invalide automatiquement le token local de l’appareil.

## Automatisations utiles

Après une intégration réussie, la PortaSplit peut être utilisée de manière nettement plus intelligente. Les ID d’entités doivent être adaptés à votre propre installation.

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

Préréfrigérer avant le coucher :

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

## Configuration recommandée en bref

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

La Midea PortaSplit s’intègre étonnamment bien dans Home Assistant. Après une configuration réussie, elle peut être contrôlée localement et intégrée dans des automatisations, ce qui élimine une grande partie de la dépendance au cloud pour l’utilisation quotidienne.

Du point de vue de la sécurité, l’intégration est défendable si quelques règles de base sont respectées : aucune redirection de port, garder le token et la clé secrets, chiffrer les sauvegardes, vérifier les journaux de débogage avant publication, sécuriser Home Assistant, segmenter les appareils IoT, limiter l’accès sortant à Internet à ce qui est nécessaire et ne pas installer aveuglément les mises à jour du firmware et de HACS. Utilisée ainsi, la PortaSplit reste un climatiseur performant tout en devenant un élément utilement intégrable à une maison intelligente contrôlée localement.

## Sources

1.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> : intégration `Midea Smart AC` : types d’appareils pris en charge `0xAC` et `0xCC`, PortaSplit avec « Out Silent Mode », utilisation du cloud pour obtenir le token et la clé sur les appareils V3, configuration manuelle et port standard 6444.

2.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> : intégration `Midea AC LAN` : catégories d’appareils prises en charge, connexion TCP prolongée pour la synchronisation de l’état et version minimale Home Assistant 2024.4.1.

3.  [midea_ac_lan : documentation des entités de climatisation](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/AC.md) : entités et attributs pour les climatiseurs, notamment la puissance, l’énergie totale, la fréquence du compresseur et les méthodes de décodage des valeurs énergétiques de certains sous-types.

4.  [midea_ac_lan : indications de débogage et de configuration](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/debug.md) : emplacement de la configuration de l’appareil sous `/config/.storage/midea_ac_lan/`, recommandation de sauvegarder plutôt que de supprimer le fichier JSON et configuration des enregistreurs pour les journaux de débogage.

5.  [Issue 779 : Out Silent Mode de la PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/779) : demande de prise en charge du mode silencieux de l’unité extérieure introduit avec la mise à jour du firmware de janvier 2026, qui réduit le bruit d’environ 6 décibels.

6.  [Midea SmartHome](https://www.midea.com/global/smarthome) : informations du fabricant concernant les normes de sécurité et de protection des données EN 303 645, PSTI, NIST, RGPD et RED DA.

7.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/) : installation et gestion d’intégrations personnalisées qui ne font pas partie de Home Assistant Core.
