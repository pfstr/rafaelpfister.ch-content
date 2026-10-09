---
title: "Midea V2, V3 et API cloud : ce que cela signifie réellement pour la PortaSplit"
navTitle: "API cloud Midea V2"
description: "Le protocole local de l’appareil, les points de terminaison privés de l’application et l’API partenaire officielle utilisent des noms de version similaires. L’analyse des sources distingue ces niveaux et replace l’avertissement d’arrêt dans son contexte."
date: "2026-07-25"
kategorie: "Home Assistant et IoT"
timeToRead: "11 min de lecture"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant-absichern
  - midea-portasplit-home-assistant
draft: false
slug: "midea-v2-v3-et-api-cloud-ce-que-cela-signifie-reellement-pour-la-portasplit"
translationOf: "midea-v2-cloud-api-portasplit-home-assistant"
translationId: article-f504b2af00493864
translationModel: gpt-5.6-terra
translatedAt: 2026-10-09T10:44:10.161Z
translationReview: automatic
translationSourceHash: fb685f86fac63fa4efa6770539bc7b23d1376fc51c6cca5993d68a517872975f
image: ../images/midea-portasplit-home-assistant/portasplit-dashboard.png
url: https://rafaelpfister.ch/fr/blog/midea-v2-v3-et-api-cloud-ce-que-cela-signifie-reellement-pour-la-portasplit
---

Dans l’environnement de la Midea PortaSplit, « V2 » désigne plusieurs choses indépendantes les unes des autres. Il existe un protocole local V2 pour les appareils, des numéros de version dans des points de terminaison privés d’applications et une API officielle cloud-à-cloud V2 destinée aux partenaires. Assimiler ces niveaux conduit inévitablement à des conclusions erronées sur le contrôle local.

Le projet `Midea AC LAN` avertit dans son [README](https://github.com/wuwentao/midea_ac_lan#1-important-notice) que les interfaces de jetons utilisées jusqu’ici seraient fermées et remplacées par une API V2 basée sur le cloud. L’examen des discussions, du code actuel et de la documentation officielle de Midea donne une image plus nuancée :

> Une API officielle Midea cloud-à-cloud V2 existe. Toutefois, elle n’est identique ni à l’interface de jetons utilisée par Home Assistant ni au protocole local V2 ou V3 des appareils. Aucun arrêt officiellement annoncé du contrôle local de la PortaSplit à une date précise n’est documenté. En juin 2026, il a en outre été démontré que l’API de jetons SmartHome prétendument désactivée fonctionnait toujours : la requête précédente de la bibliothèque communautaire était simplement incomplète.

Ceci est la partie 3 de la série ; la [partie 1](/blog/midea-portasplit-home-assistant) décrit la configuration jusqu’au tableau de bord, la [partie 2](/blog/midea-portasplit-home-assistant-absichern) la sécurisation du jeton, de la clé et du réseau domestique. Cet article est à jour au 25 juillet 2026.

![Tableau de bord Home Assistant de la Midea PortaSplit en mode refroidissement : indicateurs en haut, thermostat à 22 °C, historiques de la température ambiante, de la puissance absorbée, de l’énergie quotidienne, de la fréquence du compresseur, du fonctionnement du compresseur et de la vitesse du ventilateur, suivis des valeurs techniques et de l’état.](../images/midea-portasplit-home-assistant/portasplit-dashboard.png)

## Pourquoi l’interprétation précédente doit être corrigée

Dans une version antérieure de l’[article sur le jeton et la clé](/blog/midea-portasplit-home-assistant-absichern), j’avais présenté l’avertissement du projet `Midea AC LAN` comme l’annonce, en substance, de l’arrêt des interfaces cloud. Cela correspondait au libellé du README du projet, mais était formulé de manière trop catégorique comme une affirmation factuelle.

L’avertissement reste pertinent en tant qu’indication de risque. Il ne constitue toutefois pas une feuille de route Midea publiée. Surtout, de nouveaux éléments techniques sont désormais disponibles et remettent en question une part essentielle de l’interprétation précédente.

## Fonctionnement du contrôle local de la PortaSplit

L’intégration Home Assistant `Midea Smart AC` décrit explicitement son architecture comme un contrôle local. Sur les appareils V3 plus récents, le cloud Midea n’est utilisé que durant la configuration afin d’obtenir un jeton et une clé propres à l’appareil. L’intégration enregistre ensuite ces deux valeurs localement et ne nécessite plus aucune connexion cloud pour le contrôle proprement dit. Le projet le documente sous [« Note On Cloud Usage »](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

Le déroulement peut être simplifié ainsi :

```text
Einrichtung:

Home Assistant
    │
    ├── Anmeldung an einer Midea-Cloud
    ├── Abruf von Geräte-ID, Token und Key
    └── lokale Speicherung der Zugangsdaten

Normalbetrieb:

Home Assistant
    │
    └── lokale TCP-Verbindung zur PortaSplit
```

Pour les appareils V3 configurés manuellement, `Midea Smart AC` exige l’identifiant de l’appareil, l’adresse IP, le port, le jeton et la clé. Le port standard documenté est `6444/TCP`; le jeton et la clé sont indiqués comme ayant respectivement 128 et 64 caractères hexadécimaux. Ces informations figurent dans la [documentation sur la configuration manuelle](https://github.com/mill1000/midea-ac-py#manual-configuration).

Une PortaSplit a par exemple été détectée dans le gestionnaire d’issues de `Midea AC LAN` comme type d’appareil `0xAC`, modèle `00000Q1D` et version de protocole 3. Le même utilisateur a ensuite pu l’ajouter à Home Assistant via NetHome Plus. Le déroulement précis est documenté dans l’[issue n° 607](https://github.com/wuwentao/midea_ac_lan/issues/607).

La séparation est déterminante :

- Le service cloud est utilisé pour obtenir les identifiants d’accès locaux.
- Le contrôle ultérieur s’effectue directement sur le réseau local.
- Une défaillance du service de jetons empêche donc avant tout les nouvelles configurations.
- Elle ne met pas automatiquement fin à une connexion locale déjà configurée.

Ce dernier point correspond également à la description explicite de [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

## Origine de l’avertissement d’arrêt

Le texte d’avertissement visible aujourd’hui a été ajouté à la documentation le 19 mai 2025 avec la [pull request n° 578](https://github.com/wuwentao/midea_ac_lan/pull/578).

Les motifs peuvent être résumés ainsi :

- Les jetons locaux n’auraient pas de date d’expiration.
- Différents projets Home Assistant utiliseraient un chiffrement d’application reproduit ou extrait.
- Il en résulterait un problème de sécurité.
- Midea fermerait donc progressivement les services de jetons existants.
- À long terme, le contrôle local V1 devrait être évincé par une API V2 basée sur le cloud.

En juillet 2025, la documentation a de nouveau été adaptée via la [pull request n° 639](https://github.com/wuwentao/midea_ac_lan/pull/639). Au lieu du cloud SmartHome, NetHome Plus était désormais mentionné comme source temporaire de jetons. L’avertissement d’arrêt proprement dit est resté en place.

La discussion sous-jacente est toutefois formulée avec plus de prudence que le README.

Dans le [commentaire du mainteneur de Midea-AC-LAN](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457), il est indiqué, en substance, que NetHome Plus n’est peut-être qu’une solution temporaire et que Midea disposerait, selon sa compréhension, d’un nouveau service V2 entièrement basé sur le cloud.

Le mainteneur de `midea-msmart` a répondu qu’il soupçonnait lui aussi l’existence d’une nouvelle API V2, mais ne pouvait l’étudier que de façon limitée faute de posséder ses propres appareils Midea. Cela figure dans le [commentaire de réponse direct](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109).

La situation des sources est ainsi plus claire :

- L’avertissement provient de développeurs communautaires expérimentés.
- Il repose sur des changements observés et sur leur évaluation technique.
- L’un des mainteneurs qualifie explicitement la migration V2 de sa compréhension.
- L’autre parle d’une supposition.
- Ni la pull request ni la discussion ne renvoient à une annonce officielle de Midea concernant un arrêt ou à une date.

Cela ne rend pas l’avertissement inutile. Mais cela en fait une analyse de risque, et non une feuille de route confirmée par le fabricant.

## La nouvelle constatation décisive de juin 2026

Le 15 juin 2026, un correctif a été intégré dans la bibliothèque `midea-local`, modifiant sensiblement l’interprétation précédente.

Le point de départ était l’erreur suivante :

```json
{
  "code": "3004",
  "msg": "value is illegal."
}
```

Cette erreur était apparue lors de la demande du jeton et de la clé via le cloud SmartHome. La connexion et la liste des appareils fonctionnaient toujours, mais l’appel de `/v1/iot/secure/getToken` était refusé.

Au départ, cela ressemblait à une interface désactivée ou rendue inutilisable. L’analyse de la requête de l’application officielle SmartHome a toutefois révélé une autre cause : outre `udpid`, l’application envoyait le champ `applianceCodes`. La bibliothèque communautaire n’envoyait pas ce champ.

La requête corrigée contient désormais :

```python
data.update({
    "udpid": udp_id,
    "applianceCodes": str(appliance_id)
})
```

Le développeur a testé la modification avec un véritable compte SmartHome et quatre climatiseurs V3 de type `0xAC` :

- Sans `applianceCodes`, le serveur répondait avec l’erreur 3004.
- Avec `applianceCodes`, il fournissait des jetons et des clés valides.
- Les valeurs renvoyées fonctionnaient ensuite pour l’authentification locale V3.

L’enquête complète, les résultats des tests et le diff du code sont documentés dans la [pull request n° 470 de `midea-local`](https://github.com/midea-lan/midea-local/pull/470). Le commit immuable correspondant est [`23312799`](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5).

Le code source actuel utilise encore exactement ce point de terminaison :

```text
/v1/iot/secure/getToken
```

De plus, `applianceCodes` est désormais également envoyé. Cela peut être vérifié directement dans le [code actuel de `midealocal/cloud.py`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py).

La version actuelle de `Midea AC LAN` intègre `midea-local==6.11.0` et se déclare toujours comme une intégration `local_push`. Ces deux éléments figurent dans le [manifeste actuel`manifest.json`](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json).

L’affirmation générale selon laquelle l’API de jetons SmartHome aurait été fermée est donc réfutée, au moins pour les comptes et appareils testés en juin 2026. La formulation correcte serait :

> La demande de jetons utilisée jusqu’ici ne fonctionnait plus après une modification du format de requête attendu. Après adaptation au format employé par l’application officielle, le même point de terminaison V1 a de nouveau fourni des identifiants d’accès locaux valides.

Cela n’exclut pas des différences régionales, des comptes différents ou des types d’appareils non pris en charge. Il ne s’agissait manifestement pas d’un arrêt global.

## Pourquoi « V2 » est si facile à mal comprendre ici

Dans l’univers Midea, au moins trois désignations de version indépendantes sont utilisées.

| Terme | Signification |
| --- | --- |
| Protocole local V2/V3 | Génération de la communication directe entre l’intégration et l’appareil |
| Point de terminaison d’application V1/V2 | Numéro de version d’un point de terminaison HTTP individuel dans le backend des applications Midea |
| API cloud-à-cloud V2 | API partenaire officielle pour les entreprises tierces autorisées |

### V2 et V3 locaux

Dans le protocole local de l’appareil, V2 ou V3 désigne la génération de communication de l’appareil. Les appareils V3 plus récents nécessitent un jeton et une clé pour l’authentification locale. `Midea Smart AC` documente cette condition dans son [guide de configuration](https://github.com/mill1000/midea-ac-py#manual-configuration).

Cette version de protocole n’a rien à voir avec l’API cloud-à-cloud officielle V2.

### V1 et V2 dans les URL des applications

Même au sein d’une même application, des points de terminaison avec des numéros de version différents peuvent être utilisés simultanément. Un `/v2/` dans le chemin de l’URL ne signifie donc pas que l’ensemble de la plateforme a été migré vers une nouvelle architecture.

Le code actuel de `midea-local` utilise toujours [`/v1/iot/secure/getToken`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) pour le jeton et la clé. D’autres fonctions peuvent néanmoins se trouver sous des chemins versionnés différemment.

### API officielle cloud-à-cloud V2

Midea documente bel et bien une [API officielle cloud-à-cloud V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Elle utilise notamment :

- OAuth 2.0
- `client_id` et `client_secret`
- des jetons d’accès et de rafraîchissement à courte durée de vie
- des signatures HMAC-SHA256
- `/v2/open/oauth2/authorize`
- `/v2/open/oauth2/token`
- `/v2/open/device/list/get`
- des requêtes d’état et commandes de contrôle basées sur le cloud

Il s’agit d’une interface partenaire contrôlée. Le `client_secret` nécessaire est attribué par Midea à un fournisseur tiers. Un propriétaire ordinaire de PortaSplit ne l’obtient pas simplement via son compte MSmartHome. Les exigences et les règles de signature sont décrites dans la [documentation officielle V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Cette API n’est d’ailleurs pas apparue seulement en 2025. La documentation contient des exemples de requêtes avec des horodatages de 2018 ainsi qu’un commentaire Java du 18 avril 2019. L’interface partenaire V2 existait donc déjà bien avant l’avertissement dans `Midea AC LAN`.

## Midea remplace effectivement une API V1, mais une autre

Midea maintient également une ancienne interface officielle cloud-à-cloud sous `/v1/open/...`. Sa documentation porte explicitement la mention selon laquelle elle n’est plus recommandée, pourrait être désactivée à l’avenir et devrait être remplacée par la nouvelle documentation V2. Cela figure dans la [documentation Midea de l’ancienne API cloud-à-cloud](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html).

Cette mention correspond à une véritable migration officielle de V1 vers V2. Elle concerne toutefois les points de terminaison partenaires :

```text
/v1/open/...
           ↓
/v2/open/...
```

La demande de jetons utilisée par les bibliothèques Home Assistant est en revanche :

```text
/v1/iot/secure/getToken
```

Et la connexion locale de la PortaSplit ne passe ensuite plus du tout par une telle URL cloud, mais directement par le réseau domestique.

Assimiler ces trois interfaces uniquement en raison du numéro de version « V1 » ne serait donc pas techniquement justifié.

## Existe-t-il déjà une intégration Home Assistant entièrement basée sur le cloud ?

Avec [`Midea Auto Cloud`](https://github.com/sususweet/midea_auto_cloud), il existe désormais une intégration communautaire qui contrôle les appareils Midea via le cloud plutôt que directement via le réseau local.

Cela ne prouve toutefois pas non plus que l’API partenaire officielle V2 ait déjà remplacé le contrôle local. Le code source actuel de `Midea Auto Cloud` utilise notamment :

```text
/v1/appliance/transparent/send
/mjl/v1/device/status/lua/get
/mjl/v1/device/lua/control
```

Ces points de terminaison sont consultables dans le [code cloud actuel de `core/cloud.py`](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py).

L’intégration reproduit ainsi des fonctions d’applications privées ou du cloud grand public. Elle n’utilise pas simplement l’interface partenaire documentée `/v2/open/...`.

Une alternative basée sur le cloud existe donc déjà. Elle s’accompagne toutefois aussi des dépendances habituelles d’une intégration cloud : accès à Internet, compte utilisateur fonctionnel, serveurs Midea disponibles et points de terminaison privés toujours compatibles.

## Que cela signifie-t-il concrètement pour les propriétaires de PortaSplit ?

### Contrôle local déjà configuré

Pour une PortaSplit déjà configurée, la situation est relativement peu critique. `Midea Smart AC` enregistre le jeton et la clé localement après la configuration et, selon sa propre [documentation cloud](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage), ne nécessite plus de connexion cloud pour les contrôles ultérieurs.

Un arrêt de la seule récupération des jetons ne mettrait donc pas automatiquement fin à la connexion locale existante.

### Nouvelle configuration ou restauration

Le risque est plus important dans les cas suivants :

- une nouvelle installation de Home Assistant
- le passage à une autre intégration
- une sauvegarde perdue ou endommagée
- le remplacement du module Wi-Fi
- des modifications de l’association de l’appareil
- un nouvel appairage si les identifiants d’accès à l’appareil changent à cette occasion

Dans de tels cas, l’intégration doit à nouveau obtenir le jeton et la clé, ou l’utilisateur doit les saisir manuellement. Le fait que `Midea Smart AC` prenne en charge une configuration manuelle est décrit dans sa [documentation de configuration](https://github.com/mill1000/midea-ac-py#manual-configuration).

Il n’est pas officiellement documenté qu’une réinitialisation d’usine ou un nouvel appairage génère obligatoirement de nouveaux identifiants d’accès pour chaque PortaSplit ; cela ne devrait donc pas être affirmé de manière générale.

### Un véritable arrêt du contrôle LAN

Pour qu’une PortaSplit déjà configurée n’accepte plus ses identifiants d’accès enregistrés localement, le comportement de l’appareil ou du module Wi-Fi devrait également changer, par exemple via un nouveau firmware ou une procédure d’authentification modifiée.

Un simple arrêt du point de terminaison cloud `/v1/iot/secure/getToken` ne supprime pas automatiquement les identifiants d’accès déjà présents dans l’appareil et dans Home Assistant. Cela découle de la séparation entre récupération unique dans le cloud et contrôle LAN ultérieur documentée par [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

Une telle modification future des appareils est techniquement possible. Je n’ai toutefois trouvé dans les documents Midea accessibles au public aucune annonce concrète ni date d’arrêt spécifique à la PortaSplit.

## Ce que je continuerais à recommander

Malgré ces conclusions relativisantes, une sauvegarde reste judicieuse.

Pour les appareils V3, `Midea AC LAN` recommande explicitement de sauvegarder la configuration JSON générée en dehors de HAOS. La recommandation actuelle figure directement dans le [README du projet](https://github.com/wuwentao/midea_ac_lan#1-important-notice).

Une sauvegarde constitue une protection raisonnable contre les changements du cloud, les problèmes d’intégration et les erreurs personnelles, mais elle n’indique pas qu’un arrêt soit imminent. La manière de sauvegarder le jeton, la clé et la configuration est décrite dans la [partie 2](/blog/midea-portasplit-home-assistant-absichern#token-key-und-konfiguration-sichern).

## Évaluation sur la base des éléments disponibles

L’avertissement de `Midea AC LAN` doit être pris au sérieux, mais interprété correctement.

Il documente un risque plausible à long terme : Midea pourrait considérer les jetons locaux sans expiration comme un problème de sécurité, restreindre davantage l’obtention de tels jetons ou lier plus étroitement les futurs appareils au cloud.

En revanche, aucun arrêt officiellement annoncé et daté du contrôle local de la PortaSplit n’est établi.

L’état technique actuel montre même l’inverse d’un arrêt déjà réalisé : en juin 2026, le point de terminaison V1 de jetons toujours utilisé a fourni des identifiants valides après adaptation de la requête au format de l’application officielle SmartHome. Le correctif correspondant fait aujourd’hui partie de la bibliothèque utilisée par `Midea AC LAN`.

L’API officielle Midea cloud-à-cloud V2 existe également. Mais il s’agit d’une interface partenaire plus ancienne, à accès restreint, et non automatiquement du successeur du protocole local de la PortaSplit.

La conclusion sobre est donc :

> Créez une sauvegarde, surveillez les intégrations et gardez les dépendances cloud à l’esprit, mais n’abandonnez pas prématurément le contrôle local de la PortaSplit sur la base d’une hypothèse d’arrêt non confirmée.

## Sources

1.  [Midea AC LAN : README actuel et avertissement d’arrêt](https://github.com/wuwentao/midea_ac_lan#1-important-notice) : libellé de l’avertissement, recommandation de sauvegarde et distinction entre les anciens appareils V2 et les appareils V3 plus récents.

2.  [Midea AC LAN PR n° 578 du 19 mai 2025](https://github.com/wuwentao/midea_ac_lan/pull/578) : introduction de l’avertissement concernant l’arrêt progressif des services de jetons et la migration alléguée vers une API V2 basée sur le cloud.

3.  [Midea AC LAN PR n° 639](https://github.com/wuwentao/midea_ac_lan/pull/639) : passage de la source de jetons documentée à NetHome Plus.

4.  [midea-msmart issue n° 201](https://github.com/mill1000/midea-msmart/issues/201) : discussion sur la demande de jetons SmartHome défectueuse et l’utilisation temporaire de NetHome Plus.

5.  [Commentaire du mainteneur de Midea-AC-LAN sur la migration V2 supposée](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457) : qualifie explicitement l’affirmation sur le nouveau cloud V2 de compréhension personnelle.

6.  [Réponse du mainteneur de midea-msmart](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109) : décrit l’existence d’une nouvelle API V2 comme une supposition et souligne les possibilités limitées de rétro-ingénierie.

7.  [midea-local PR n° 470 du 15 juin 2026](https://github.com/midea-lan/midea-local/pull/470) : analyse de l’erreur 3004, capture de la requête de l’application officielle, ajout de `applianceCodes` et test réussi avec quatre climatiseurs V3.

8.  [Commit immuable du correctif SmartHome-getToken](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5) : diff exact du code du correctif intégré.

9.  [Code cloud actuel de midea-local](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) : point de terminaison `/v1/iot/secure/getToken` toujours utilisé et champ de requête actuel `applianceCodes`.

10.  [Manifeste actuel de Midea AC LAN](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json) : version utilisée de `midea-local` et classification comme intégration push locale.

11.  [Midea Smart AC](https://github.com/mill1000/midea-ac-py) : documentation du contrôle local, de la récupération cloud unique pour les appareils V3 et de la configuration manuelle avec jeton et clé.

12.  [Midea AC LAN issue n° 607 sur la PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/607) : exemple concret de PortaSplit avec le type d’appareil `0xAC`, le modèle `00000Q1D`, la version de protocole 3 et une configuration réussie via NetHome Plus.

13.  [API officielle Midea cloud-à-cloud V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html) : OAuth2, ID client, secret client, jetons d’accès et de rafraîchissement, procédé de signature et points de terminaison `/v2/open/...`.

14.  [API officielle Midea cloud-à-cloud V1](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html) : indication officielle selon laquelle l’interface partenaire `/v1/open/...` ancienne n’est plus recommandée et pourrait être désactivée à l’avenir.

15.  [Midea Auto Cloud](https://github.com/sususweet/midea_auto_cloud) et [code cloud actuel](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py) : intégration communautaire pour un contrôle entièrement via le cloud et points de terminaison privés V1 d’application effectivement utilisés.
