---
title: "Contrôler localement le Midea PortaSplit avec Home Assistant et l’utiliser en toute sécurité"
navTitle: "Configurer PortaSplit"
description: "De l’intégration communautaire adaptée au VLAN IoT : configurez la PortaSplit, protégez le token et la clé, et limitez les accès au cloud et au réseau."
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
translationSourceHash: 859c24ec38af3b4b931702c7be50cf2224580d30045559ba089224d0de25339c
translatedAt: 2026-08-08T14:18:53.130Z
url: https://rafaelpfister.ch/fr/blog/controler-localement-la-midea-portasplit-avec-home-assistant-et-l-utiliser-en-toute-securite
translationModel: gpt-5.6-terra
---

Le Midea PortaSplit peut être contrôlé directement sur le réseau local via Home Assistant après sa configuration. Pour cela, l’intégration communautaire nécessite deux identifiants spécifiques à l’appareil provenant du cloud Midea : un token et une clé.

Cet article présente le choix, la configuration et la sécurisation de l’intégration. Les solutions décrites proviennent de la communauté et ne sont officiellement prises en charge ni par Midea ni par Home Assistant. Des modifications du firmware ou du cloud peuvent donc en influencer le fonctionnement à tout moment. Le contexte de l’interface de tokens et de l’avertissement ambigu concernant son arrêt est présenté dans l’[analyse des API du cloud Midea](/blog/midea-v2-cloud-api-portasplit-home-assistant).

## Fonctionnement du contrôle local

Une fois la configuration terminée, les commandes de contrôle sont envoyées directement par Home Assistant à la PortaSplit :

```text
Home Assistant → lokales Netzwerk → Midea PortaSplit
```

Une commande de commutation ne doit pas passer par un serveur Midea externe, le temps de réponse est court, une panne du cloud Midea ne paralyse pas nécessairement le contrôle local déjà configuré, et l’appareil reste en principe contrôlable même sans accès à Internet.

Sur les appareils récents utilisant le protocole dit V3, la PortaSplit n’accepte toutefois pas les commandes locales sans protection. Home Assistant a besoin de deux valeurs spécifiques à l’appareil, un token et une clé, qui servent à l’authentification et au chiffrement de la connexion locale. Lors de la première configuration, l’intégration les récupère une seule fois via une interface cloud Midea, puis les enregistre localement ; aucune connexion au cloud n’est nécessaire pour le contrôle ultérieur.

De manière simplifiée, le processus est le suivant :

1. La PortaSplit est connectée à MSmartHome.
2. Home Assistant se connecte à un cloud Midea.
3. Home Assistant reçoit l’ID de l’appareil, le token et la clé.
4. Le token et la clé sont enregistrés localement.
5. Home Assistant contrôle la PortaSplit directement sur le LAN.

## Quelle intégration choisir

### Midea Smart AC

Le dépôt <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> se concentre sur les climatiseurs Midea et les modèles OEM associés, et prend en charge les types d’appareils `0xAC` et `0xCC`. Il propose un contrôle local, une configuration graphique, une détection automatique, une configuration manuelle avec token et clé, ainsi qu’une interrogation automatique des capacités de l’appareil. Le « Out Silent Mode » de la PortaSplit est explicitement pris en charge.

Le projet cite notamment les applications Artic King, Midea Air, NetHome Plus, SmartHome ou MSmartHome, Toshiba AC NA et 美的美居 comme indicateurs de compatibilité. En Europe, la PortaSplit utilise généralement MSmartHome et s’intègre donc dans cet écosystème.

### Midea AC LAN

Le dépôt <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> prend en charge non seulement les climatiseurs, mais aussi de nombreuses autres catégories d’appareils Midea : déshumidificateurs, ventilateurs, purificateurs d’air, lave-linge, sèche-linge, lave-vaisselle, appareils de production d’eau chaude, pompes à chaleur, réfrigérateurs et bien d’autres, parfois également sous des marques tierces comme Carrier ou Electrolux. Il propose également une communication locale, une détection automatique des appareils et des capteurs supplémentaires et, selon la description du projet, maintient une connexion TCP plus longue avec l’appareil afin de synchroniser rapidement les changements d’état. Home Assistant 2024.4.1 au minimum est requis.

Le principal inconvénient actuel est l’avertissement du développeur : les API de tokens cloud utilisées pour ajouter de nouveaux appareils sont progressivement désactivées. L’ajout ultérieur de nouveaux appareils pourrait ainsi devenir impossible.

### Recommandation

Pour une installation PortaSplit uniquement, je commencerais par `Midea Smart AC` et garderais `Midea AC LAN` comme alternative. `Midea Smart AC` est davantage ciblée sur les climatiseurs et documente explicitement les fonctions actuelles de la PortaSplit.

Il n’est pas judicieux d’utiliser les deux intégrations simultanément et durablement avec le même appareil. Plusieurs connexions parallèles entraînent des problèmes d’état, du trafic réseau inutile et un comportement difficile à comprendre.

## Ce que l’intégration apporte

Après la configuration, la PortaSplit apparaît comme une entité `climate` dans Home Assistant. Selon le firmware et l’intégration, les fonctions suivantes sont notamment disponibles :

- Allumer et éteindre
- Régler la température de consigne
- Lire la température ambiante actuelle
- Refroidissement, déshumidification et mode ventilation seule
- Régler la vitesse du ventilateur
- Contrôler la fonction Swing
- Modes Eco et Boost
- Lire l’humidité de l’air
- Afficher les codes d’erreur
- Lire les valeurs d’énergie et de puissance
- Afficher les valeurs du compresseur
- Activer le mode silencieux de l’unité extérieure

Les entités effectivement disponibles dépendent du modèle, du firmware, du protocole utilisé et de l’intégration concernée. `Midea Smart AC` interroge les capacités signalées par l’appareil et masque les fonctions que le modèle ne prend pas en charge. `Midea AC LAN` documente également de nombreuses entités de climatisation, dont la température, l’humidité, la puissance actuelle, l’énergie totale, la fréquence du compresseur, l’état de la pompe et différents modes de fonctionnement, et indique des méthodes propres à certains sous-types de PortaSplit pour décoder les données énergétiques.

Toutes les mesures affichées ne sont pas nécessairement correctes. La consommation d’énergie et la puissance, en particulier, sont transmises dans des formats différents selon les modèles Midea. Si Home Assistant affiche des valeurs manifestement erronées, il faut généralement adapter la méthode de décodage utilisée plutôt que conclure à une défaillance de l’appareil.

## Prérequis

Il faut une Midea PortaSplit avec Wi-Fi, un réseau Wi-Fi 2,4 GHz, l’application MSmartHome, un compte utilisateur Midea, Home Assistant, HACS et un accès réseau entre Home Assistant et la PortaSplit. La PortaSplit doit d’abord être connectée normalement à l’application MSmartHome, avant d’être ajoutée à Home Assistant.

## Étape 1 : connecter la PortaSplit à MSmartHome

1. Installer l’application MSmartHome.
2. Créer un compte Midea ou se connecter.
3. Mettre la PortaSplit en mode d’appairage Wi-Fi.
4. Connecter l’appareil au réseau Wi-Fi 2,4 GHz.
5. Vérifier que la PortaSplit peut être contrôlée via l’application.

De nombreux appareils IoT ne prennent toujours en charge que le 2,4 GHz. Si le routeur utilise le même SSID pour le 2,4 et le 5 GHz, la configuration fonctionne généralement tout de même. En cas de problème, il peut être utile de fournir temporairement un réseau Wi-Fi 2,4 GHz séparé.

## Étape 2 : installer HACS

HACS est le magasin communautaire de Home Assistant. Il permet d’installer des intégrations communautaires qui ne font pas partie de Home Assistant Core. Après l’installation de HACS, ouvrez HACS, accédez aux intégrations, recherchez `Midea Smart AC`, téléchargez l’intégration et redémarrez Home Assistant. Vous pouvez également rechercher `Midea AC LAN`.

HACS simplifie l’installation et les mises à jour. Il ne transforme toutefois pas une intégration personnalisée en composant Home Assistant officiellement vérifié. Cette différence est essentielle du point de vue de la sécurité et sera abordée plus loin.

## Étape 3 : ajouter Midea Smart AC

Après le redémarrage, allez dans Paramètres, Appareils et services, puis Ajouter une intégration, recherchez `Midea Smart AC`, puis `Discover devices`. L’intégration peut soit parcourir l’ensemble du réseau local, soit interroger directement l’adresse IP de la PortaSplit.

Si l’appareil est trouvé, l’intégration requiert, pour les appareils V3 récents, la région, le compte Midea, le mot de passe et l’ID de l’appareil, ainsi que le token et la clé qui en sont dérivés. La région cloud doit correspondre au compte utilisé. En cas de problème, le projet recommande également d’essayer les autres régions proposées.

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

Pour les appareils V3, la documentation indique un token sous la forme d’une chaîne hexadécimale de 128 caractères et une clé sous la forme d’une chaîne hexadécimale de 64 caractères. Ces deux valeurs sont des secrets et doivent être traitées comme tels. Ceux qui ne souhaitent pas obtenir les identifiants via Discovery peuvent les récupérer avec leur propre compte via la CLI `msmart-ng`.

## Utiliser la PortaSplit en toute sécurité

Contrôler la PortaSplit localement permet de reprendre une partie du contrôle au cloud du fabricant, mais déplace aussi la responsabilité vers votre propre réseau. Les points suivants garantissent que le token et la clé causent peu de dommages même en cas d’incident et que l’appareil reste correctement isolé.

### Le token et la clé sont des secrets

Le token et la clé authentifient la communication locale avec l’appareil et doivent être traités comme un mot de passe. Pour l’exploitation, l’essentiel est le suivant : ils n’ont leur place ni dans les journaux, ni dans des sauvegardes non chiffrées, ni dans un dépôt.

### Pas de redirection de port vers la PortaSplit

L’erreur évitable la plus fréquente serait de rendre le port local de l’appareil directement accessible depuis Internet. Une règle comme celle-ci serait dangereuse :

```text
Internet → TCP 6444 → PortaSplit
```

Il n’existe aucune bonne raison de rendre la PortaSplit directement accessible depuis Internet. Home Assistant se trouve déjà sur le réseau local et sert d’instance de contrôle. Le routeur ne devrait avoir aucune redirection de port vers la PortaSplit, UPnP devrait être restreint ou désactivé dans la mesure du possible, les connexions entrantes devraient être bloquées par défaut et aucune autorisation DMZ ne devrait être utilisée pour l’appareil.

### VLAN IoT dédié

La meilleure architecture réseau est un réseau IoT séparé :

```text
VLAN 10: vertrauenswürdige Clients
VLAN 20: Server und Home Assistant
VLAN 30: IoT-Geräte
VLAN 40: Gäste
```

La PortaSplit se trouve dans le VLAN IoT. Home Assistant peut accéder de manière ciblée à l’appareil, mais la PortaSplit ne doit pas pouvoir accéder librement aux PC, NAS et autres systèmes internes. Une logique de pare-feu possible :

```text
Home Assistant → PortaSplit: erlauben
PortaSplit → Home Assistant: etablierte Verbindungen erlauben
PortaSplit → interne Clients: blockieren
PortaSplit → NAS: blockieren
PortaSplit → Management-Netz: blockieren
Internet → PortaSplit: blockieren
```

Lors de la première configuration, l’appareil nécessite un accès à Internet vers le cloud Midea. Une fois la configuration locale réussie, vous pouvez tester si l’accès sortant à Internet peut être bloqué. Il ne faut toutefois pas imposer immédiatement un blocage définitif. Vérifiez d’abord que le contrôle local continue de fonctionner, que l’appareil reste accessible après un redémarrage, qu’il résiste à un redémarrage du routeur, qu’il répond toujours après plusieurs jours, que l’application MSmartHome est toujours nécessaire et que les mises à jour du firmware sont encore proposées. Si vous souhaitez continuer à utiliser le cloud et les mises à jour du firmware, vous pouvez autoriser temporairement l’accès sortant à Internet, puis le bloquer à nouveau.

### La segmentation réseau peut empêcher Discovery

La recherche automatique d’appareils repose souvent sur le trafic broadcast ou multicast, qui n’est normalement pas routé au-delà des limites des VLAN. Home Assistant peut donc ne pas trouver automatiquement la PortaSplit, même si une connexion IP classique serait autorisée.

Dans ce cas, il peut être utile de configurer temporairement la PortaSplit dans le même VLAN que Home Assistant, de saisir manuellement l’adresse IP de l’appareil, d’utiliser une fonction de relais broadcast adaptée ou de définir des règles de pare-feu ciblées après la configuration. La configuration manuelle est même souvent préférable du point de vue de la sécurité, car elle évite d’autoriser un trafic broadcast supplémentaire entre les réseaux.

### Attribution DHCP statique

La PortaSplit devrait recevoir une attribution DHCP fixe dans le routeur :

```text
PortaSplit → 192.168.30.25
```

Une réservation DHCP est généralement préférable à une adresse IP statique définie sur l’appareil. Home Assistant trouve l’appareil de manière fiable, les règles de pare-feu peuvent être limitées à une adresse fixe, l’analyse des erreurs est simplifiée et l’attribution reste stable après le redémarrage du routeur ou de l’appareil. Une règle de pare-feu peut ainsi être formulée de manière très restrictive :

```text
Home-Assistant-IP → 192.168.30.25:6444/TCP
```

Le port réellement requis doit être vérifié selon l’intégration et votre appareil.

### Home Assistant comme point d’ancrage central de confiance

Contrôler la PortaSplit localement déplace une partie de la confiance du cloud Midea vers Home Assistant. Si Home Assistant est compromis, un attaquant peut potentiellement contrôler non seulement la climatisation, mais aussi l’ensemble de la maison connectée.

Home Assistant doit donc être mis à jour régulièrement, ne pas être publié via une redirection de port non protégée, être protégé par un mot de passe fort et unique, utiliser l’authentification multifacteur, créer des sauvegardes chiffrées, ne contenir que les extensions nécessaires et ne pas permettre d’accès SSH inutile depuis Internet. Pour l’accès à distance, un VPN, Home Assistant Cloud ou un reverse proxy correctement configuré constituent de meilleures options qu’une simple redirection de port vers le port 8123.

### HACS et le risque lié à la chaîne d’approvisionnement

`Midea Smart AC` et `Midea AC LAN` sont des intégrations personnalisées. Elles s’exécutent au sein de Home Assistant et bénéficient donc d’un accès étendu à son environnement d’exécution. Une intégration malveillante ou compromise pourrait théoriquement lire les données de configuration, extraire des secrets, établir des connexions réseau, analyser les appareils du réseau local, lire les états d’autres entités, transférer des données vers des systèmes externes et nuire à la disponibilité de Home Assistant.

Cela ne signifie pas que les intégrations mentionnées sont malveillantes. Les deux projets sont publiquement consultables, activement développés et disposent d’une communauté visible. L’open source n’est toutefois pas une garantie automatique de sécurité. Avant l’installation, il vaut au minimum la peine de vérifier si le dépôt est activement maintenu, s’il existe des versions régulières, combien de personnes contribuent au code, s’il y a des problèmes de sécurité ouverts, si les responsables ou propriétaires du dépôt ont récemment changé, si HACS pointe vers le dépôt attendu et si une mise à jour contient des changements anormalement importants ou inexplicables.

Les mises à jour ne devraient pas être installées aveuglément immédiatement après leur publication. En particulier pour les systèmes de maison connectée critiques pour la sécurité, il est judicieux d’attendre quelques jours et de consulter les notes de version ainsi que les problèmes signalés.

### Sécuriser le compte cloud

Tant que le cloud Midea est utilisé pour la configuration ou le contrôle par application, le compte Midea fait lui aussi partie du modèle de sécurité. Il doit utiliser un mot de passe unique, non partagé avec d’autres services, un gestionnaire de mots de passe, l’authentification multifacteur si elle est proposée, la suppression des anciens smartphones et sessions, l’absence de comptes partagés et un contrôle régulier des appareils enregistrés dans le compte.

Si l’intégration Home Assistant demande un nom d’utilisateur et un mot de passe pendant la configuration, il faut vérifier si les identifiants sont utilisés uniquement pour récupérer le token une seule fois ou s’ils sont enregistrés durablement. Les développeurs de `Midea Smart AC` indiquent que les appareils ne sont pas associés à des comptes d’intégration intégrés après leur configuration et que le token et la clé peuvent aussi être obtenus manuellement via la CLI avec son propre compte. Lorsque c’est possible, votre propre compte doit être préféré à des comptes tiers ou intégrés partagés.

### Bloquer le cloud ou non ?

Après une configuration réussie, la question se pose de savoir si l’accès à Internet de la PortaSplit doit être entièrement bloqué. En faveur du blocage, on peut citer une télémétrie réduite, une moindre dépendance aux services externes, une surface d’attaque plus réduite via le cloud du fabricant, le fait que l’appareil ne peut pas contacter des destinations externes arbitraires et un impact moindre des changements côté cloud.

En revanche, l’application MSmartHome peut ne plus fonctionner hors du réseau domestique, les mises à jour du firmware ne peuvent plus être téléchargées, les fonctions d’horloge ou de cloud peuvent cesser de fonctionner, une nouvelle connexion ou une restauration peut devenir plus difficile et certains appareils peuvent réagir de manière inattendue après une longue période hors ligne.

Une séquence pragmatique : configurez normalement l’appareil, testez Home Assistant et l’application, sauvegardez le token et la configuration, bloquez l’accès à Internet, redémarrez l’appareil et Home Assistant, observez pendant plusieurs jours et, si nécessaire, réautorisez l’accès à Internet uniquement temporairement.

### Mises à jour du firmware : gain de sécurité ou risque pour l’intégration ?

Les mises à jour du firmware constituent un dilemme pour les appareils IoT. Elles peuvent corriger des vulnérabilités connues, améliorer la stabilité, moderniser les mécanismes de sécurité et apporter de nouvelles fonctions. Mais elles peuvent aussi modifier les interfaces locales, casser les intégrations issues de rétro-ingénierie, invalider les tokens, désactiver l’API locale et introduire de nouvelles dépendances au cloud.

Le firmware PortaSplit distribué en janvier 2026 a par exemple introduit un nouveau mode silencieux pour l’unité extérieure, qui réduit le bruit d’environ 6 décibels. Les intégrations communautaires ont d’abord dû l’analyser et l’implémenter, ce qui est documenté dans une issue GitHub spécifique à la PortaSplit.

Il en résulte qu’il ne faut pas empêcher systématiquement les mises à jour du firmware, mais vérifier avant une mise à jour si d’autres utilisateurs de Home Assistant signalent des problèmes, sauvegarder la configuration et le token au préalable, créer une sauvegarde Home Assistant et tester intégralement le contrôle local après la mise à jour. La sécurité ne signifie pas « ne jamais mettre à jour ». Un firmware obsolète peut être plus dangereux qu’une intégration temporairement incompatible.

### Les journaux de débogage contiennent des données sensibles

En cas de problème, les projets open source demandent souvent des journaux de débogage. La documentation de `Midea AC LAN` montre comment activer la journalisation pour les deux composants concernés :

```yaml
logger:
  default: warn
  logs:
    custom_components.midea_ac_lan: debug
    midealocal: debug
```

Les journaux peuvent ensuite être téléchargés via Paramètres, Système et Journaux. Selon l’intégration et le type d’erreur, de tels journaux peuvent contenir des adresses IP locales, l’ID de l’appareil, le numéro de série, l’identifiant de modèle, des réponses cloud, des informations de compte, le token ou des parties de celui-ci, des paquets réseau ainsi que des horodatages et des informations sur le comportement d’utilisation. Il convient donc de les vérifier et de masquer les valeurs sensibles avant de les téléverser dans une issue GitHub publique.

Une fois le dépannage terminé, la journalisation de débogage doit être supprimée. Une journalisation de débogage activée en permanence augmente non seulement l’utilisation de stockage, mais aussi la quantité d’informations sensibles présentes dans les sauvegardes.

### Ce que Midea indique elle-même sur la sécurité

Midea promeut son écosystème SmartHome en mettant en avant son alignement sur plusieurs normes de sécurité et de protection des données, notamment EN 303 645, UK PSTI, NIST, le traitement des données conforme au RGPD et les exigences de la directive européenne sur les équipements radioélectriques. Ce sont des signaux positifs, mais ils ne disent rien de la manière dont chaque firmware PortaSplit, chaque point de terminaison cloud et chaque API locale sont réellement implémentés. Les certifications et déclarations marketing ne remplacent pas un examen technique de l’appareil concret.

De même, il serait erroné de déduire de l’avertissement d’une intégration communautaire que la PortaSplit est globalement non sécurisée. Le problème décrit concerne l’architecture des tokens à longue durée de vie et leur utilisation par des clients non officiels.

### Risque selon le scénario

| Scénario | Risque | Justification |
| --- | --- | --- |
| Réseau domestique normal sans redirection de port | modéré | Un attaquant doit d’abord accéder au Wi-Fi, à Home Assistant ou à une sauvegarde. |
| Réseau domestique plat avec de nombreux appareils IoT non sécurisés | moyen | Un autre appareil IoT compromis peut atteindre la PortaSplit ou Home Assistant sur le même réseau. |
| PortaSplit directement accessible depuis Internet | élevé | L’appareil ne doit jamais être publié via une redirection de port. |
| Token et clé publics sur GitHub | élevé | Les secrets doivent être considérés comme compromis ; leur révocation n’est pas garantie. |
| VLAN IoT séparé, pare-feu restrictif, contrôle local | relativement faible | Même en cas de vulnérabilité de l’appareil, la liberté de mouvement sur le réseau est fortement limitée. |

## Sauvegarde de la configuration

La sauvegarde du token, de la clé et de la configuration est l’étape unique la plus importante : une fois les interfaces cloud de tokens fermées, une sauvegarde est le seul moyen de procéder à une nouvelle configuration. `Midea AC LAN` crée un fichier de configuration JSON pour les appareils V3 après une configuration réussie. Le chemin documenté est :

```text
/config/.storage/midea_ac_lan/
```

Le fichier porte l’ID de l’appareil comme nom de fichier :

```text
<device-id>.json
```

Ce fichier n’est pas une simple note de texte. Il peut contenir l’ID de l’appareil, le numéro de série, l’adresse IP, le token, la clé, des informations de protocole ainsi que des paramètres cloud et d’appareil. En conséquence :

- Ne pas le téléverser dans un dépôt GitHub public.
- Ne pas le publier dans des forums.
- Ne pas le partager sous forme de capture d’écran non masquée.
- Ne pas l’envoyer par e-mail non chiffré.

Même un dépôt Git privé n’est pas automatiquement le bon emplacement, car les secrets restent dans l’historique Git, même s’ils sont ultérieurement supprimés du fichier actuel. Une sauvegarde chiffrée, un gestionnaire de mots de passe avec pièce jointe, une sauvegarde NAS chiffrée, un support hors ligne chiffré ou une archive chiffrée avec un mot de passe stocké séparément sont plus appropriés.

Pour sauvegarder via le terminal Home Assistant :

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

Les fichiers dans `.storage` ne doivent pas être modifiés manuellement. Le développeur recommande expressément de ne ni supprimer ni modifier directement le fichier JSON en cas de problème, mais de le renommer et de le sauvegarder avant toute modification.

Une sauvegarde Home Assistant complète contient également ces fichiers. Une copie distincte reste néanmoins utile, car les sauvegardes Home Assistant peuvent être endommagées, une restauration peut écraser l’intégration, le fichier peut être nécessaire de manière ciblée pour une future nouvelle configuration et une sauvegarde ne devrait jamais se trouver uniquement sur le même système.

## Retirer des secrets d’un dépôt Git publié

Si un fichier JSON a été publié par erreur sur GitHub, une suppression normale suivie d’un nouveau commit ne suffit pas. Le fichier reste accessible dans l’historique Git. Au minimum, les étapes suivantes sont nécessaires :

1. Rendre immédiatement le dépôt privé, si possible.
2. Retirer le fichier de l’ensemble de l’historique Git.
3. Prendre en compte les caches et les forks GitHub.
4. Considérer le token comme compromis.
5. Retirer l’appareil du compte Midea et le reconnecter si cela génère de nouvelles clés.
6. Reconfigurer l’intégration Home Assistant.
7. Modifier le mot de passe du compte Midea si les identifiants ont également été concernés.

La génération effective d’un nouveau token après un nouvel appairage varie selon l’appareil et l’architecture cloud. Il ne faut pas supposer que la modification du mot de passe du compte invalide automatiquement le token local de l’appareil.

## Automatisations utiles

Après une intégration réussie, la PortaSplit peut être utilisée de manière beaucoup plus intelligente. Les ID d’entité doivent être adaptés à votre installation.

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

Éteindre lorsque personne n’est à la maison :

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

Le sens de communication souhaité est donc le suivant :

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

Le Midea PortaSplit s’intègre étonnamment bien à Home Assistant. Après une configuration réussie, il peut être contrôlé localement et intégré à des automatisations, ce qui élimine une grande partie de la dépendance au cloud pour l’utilisation quotidienne.

Du point de vue de la sécurité, l’intégration est acceptable si quelques règles de base sont respectées : aucune redirection de port, maintien secret du token et de la clé, chiffrement des sauvegardes, vérification des journaux de débogage avant publication, sécurisation de Home Assistant, segmentation des appareils IoT, limitation de l’accès sortant à Internet à ce qui est nécessaire et absence d’installation aveugle des mises à jour du firmware et de HACS. Utilisée ainsi, la PortaSplit reste une climatisation performante tout en devenant un élément intégrable de manière pertinente dans une maison connectée contrôlée localement.

## Sources

1.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> : intégration `Midea Smart AC` : types d’appareils pris en charge `0xAC` et `0xCC`, PortaSplit avec « Out Silent Mode », utilisation du cloud pour obtenir le token et la clé sur les appareils V3, configuration manuelle et port standard 6444.

2.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> : intégration `Midea AC LAN` : catégories d’appareils prises en charge, connexion TCP prolongée pour la synchronisation des états et version minimale Home Assistant 2024.4.1.

3.  [midea_ac_lan : documentation des entités de climatisation](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/AC.md) : entités et attributs pour les appareils de climatisation, dont la puissance, l’énergie totale, la fréquence du compresseur et les méthodes de décodage des valeurs énergétiques de certains sous-types.

4.  [midea_ac_lan : indications de débogage et de configuration](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/debug.md) : emplacement de la configuration de l’appareil sous `/config/.storage/midea_ac_lan/`, recommandation de sauvegarder plutôt que supprimer le fichier JSON et configuration du logger pour les journaux de débogage.

5.  [Issue 779 : Out Silent Mode de la PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/779) : demande de prise en charge du mode silencieux de l’unité extérieure introduit avec la mise à jour du firmware de janvier 2026, qui réduit le bruit d’environ 6 décibels.

6.  [Midea SmartHome](https://www.midea.com/global/smarthome) : informations du fabricant sur les normes de sécurité et de protection des données EN 303 645, PSTI, NIST, RGPD et RED DA.

7.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/) : installation et gestion d’intégrations personnalisées qui ne font pas partie de Home Assistant Core.
