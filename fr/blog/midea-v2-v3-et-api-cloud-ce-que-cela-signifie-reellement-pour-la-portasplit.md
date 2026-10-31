---
title: "Midea V2, V3 et API cloud : ce qui est réellement concerné pour la PortaSplit"
navTitle: "API cloud Midea V2"
description: "Le protocole local des appareils, les points de terminaison privés des applications et l’API partenaire officielle utilisent des noms de version similaires. L’analyse des sources distingue ces niveaux et replace l’avertissement d’arrêt dans son contexte."
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
translatedAt: 2026-10-31T03:30:51.126Z
translationReview: automatic
translationSourceHash: 551c9e10e7e9c6a84b6e439dd388d775dae5246f96cd6c71e46c6065e84c5fd7
image: ../images/midea-portasplit-home-assistant/portasplit-dashboard.png
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/midea-v2-v3-et-api-cloud-ce-que-cela-signifie-reellement-pour-la-portasplit
---

Dans l’environnement de la Midea PortaSplit, « V2 » désigne plusieurs choses indépendantes les unes des autres. Il existe un protocole local d’appareil V2, des numéros de version dans des points de terminaison privés d’applications et une API V2 cloud-à-cloud officielle destinée aux partenaires. Assimiler ces niveaux conduit inévitablement à des conclusions erronées sur le pilotage local.

Le projet `Midea AC LAN` avertit dans son [README](https://github.com/wuwentao/midea_ac_lan#1-important-notice) que les interfaces de jetons utilisées jusqu’à présent seraient fermées et remplacées par une API V2 basée sur le cloud. L’examen des discussions, du code actuel et de la documentation officielle de Midea aboutit à un constat plus nuancé :

> Une API V2 officielle de Midea cloud-à-cloud existe. Elle n’est toutefois identique ni à l’interface de jetons utilisée par Home Assistant, ni au protocole local d’appareil V2 ou V3. Aucun arrêt officiellement annoncé du pilotage local de la PortaSplit à une date précise n’est documenté. En juin 2026, il a en outre été démontré que l’API de jetons SmartHome prétendument arrêtée fonctionnait toujours : la requête jusqu’alors utilisée par la bibliothèque communautaire était simplement incomplète.

Ceci est la partie 3 de la série ; [la partie 1](/blog/midea-portasplit-home-assistant) décrit la configuration jusqu’au tableau de bord, [la partie 2](/blog/midea-portasplit-home-assistant-absichern) la sécurisation du jeton, de la clé et du réseau domestique. Cet article est à jour au 25 octobre 2026.

![Tableau de bord Home Assistant de la Midea PortaSplit en mode refroidissement : indicateurs en haut, thermostat à 22 °C, historiques de température ambiante, consommation électrique, énergie quotidienne, fréquence du compresseur, fonctionnement du compresseur et vitesse du ventilateur, puis valeurs techniques et état.](../images/midea-portasplit-home-assistant/portasplit-dashboard.png)

## Pourquoi la précédente interprétation doit être corrigée

Dans une version antérieure de [l’article sur le jeton et la clé](/blog/midea-portasplit-home-assistant-absichern), j’avais présenté l’avertissement du projet `Midea AC LAN` comme l’annonce de l’arrêt des interfaces cloud. Cela correspondait au libellé du README du projet, mais la formulation était trop catégorique en tant qu’affirmation factuelle.

L’avertissement demeure pertinent en tant qu’indication de risque. Il ne constitue toutefois pas une feuille de route Midea publiée. Surtout, de nouveaux éléments techniques sont désormais disponibles et remettent en question une part essentielle de l’interprétation précédente.

## Comment fonctionne le pilotage local de la PortaSplit

L’intégration Home Assistant `Midea Smart AC` décrit explicitement son architecture comme un pilotage local. Pour les appareils V3 plus récents, le cloud Midea n’est utilisé que durant la configuration afin d’obtenir un jeton et une clé spécifiques à l’appareil. L’intégration enregistre ensuite ces deux valeurs localement et ne nécessite plus de connexion cloud pour le pilotage proprement dit. Le projet le documente sous [« Note On Cloud Usage »](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

De manière simplifiée, le processus est le suivant :

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

Pour les appareils V3 configurés manuellement, `Midea Smart AC` exige l’identifiant de l’appareil, l’adresse IP, le port, le jeton et la clé. Le port standard documenté est `6444/TCP`; le jeton et la clé sont indiqués comme comportant respectivement 128 et 64 caractères hexadécimaux. Ces informations figurent dans la [documentation de configuration manuelle](https://github.com/mill1000/midea-ac-py#manual-configuration).

Une PortaSplit a par exemple été détectée dans le gestionnaire d’incidents de `Midea AC LAN` comme appareil de type `0xAC`, modèle `00000Q1D` et version de protocole 3. Le même utilisateur a ensuite pu l’ajouter à Home Assistant via NetHome Plus. Le déroulement précis est documenté dans [l’issue n° 607](https://github.com/wuwentao/midea_ac_lan/issues/607).

La séparation est déterminante :

- Le service cloud sert à obtenir les identifiants d’accès locaux.
- Le pilotage ultérieur s’effectue directement sur le réseau local.
- Une défaillance du service de jetons empêche donc avant tout les nouvelles configurations.
- Elle ne met pas automatiquement fin à une connexion locale déjà configurée.

Ce dernier point correspond également à la description explicite de [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

## D’où vient l’avertissement d’arrêt

Le texte d’avertissement aujourd’hui visible a été ajouté à la documentation le 19 mai 2025 via [la pull request n° 578](https://github.com/wuwentao/midea_ac_lan/pull/578).

Le raisonnement peut être résumé comme suit :

- Les jetons locaux n’auraient pas de date d’expiration.
- Différents projets Home Assistant utiliseraient un chiffrement d’application reproduit ou extrait.
- Il en résulterait un problème de sécurité.
- Midea fermerait donc progressivement les services de jetons existants.
- À long terme, le pilotage local V1 serait évincé par une API V2 basée sur le cloud.

En juillet 2025, la documentation a de nouveau été adaptée via [la pull request n° 639](https://github.com/wuwentao/midea_ac_lan/pull/639). Au lieu du cloud SmartHome, NetHome Plus était alors cité comme source de jetons temporairement utilisée. L’avertissement d’arrêt proprement dit est resté en place.

La discussion sous-jacente est toutefois formulée plus prudemment que le README.

Dans le [commentaire du mainteneur de Midea AC LAN](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457), il est dit en substance que NetHome Plus n’est peut-être qu’une solution temporaire et que, selon sa compréhension, Midea dispose d’un nouveau service V2 entièrement basé sur le cloud.

Le mainteneur de `midea-msmart` a répondu qu’il avait lui aussi supposé l’existence d’une nouvelle API V2, mais qu’il ne pouvait l’étudier que de manière limitée faute de posséder ses propres appareils Midea. Cela figure dans le [commentaire de réponse direct](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109).

La situation des sources est ainsi plus claire :

- L’avertissement émane de développeurs communautaires expérimentés.
- Il repose sur des changements observés et leur appréciation technique.
- L’un des mainteneurs qualifie explicitement la migration V2 de sa compréhension.
- L’autre parle d’une supposition.
- Ni la pull request ni la discussion ne renvoient à une annonce officielle de Midea concernant un arrêt ou à une date.

Cela ne rend pas l’avertissement sans valeur. Cela en fait toutefois une analyse de risque, et non une feuille de route confirmée par le fabricant.

## La découverte décisive de juin 2026

Le 15 juin 2026, un correctif a été intégré à la bibliothèque `midea-local`, ce qui modifie sensiblement l’interprétation précédente.

Le point de départ était l’erreur :

```json
{
  "code": "3004",
  "msg": "value is illegal."
}
```

Cette erreur s’était produite lors de la récupération du jeton et de la clé via le cloud SmartHome. La connexion et la liste des appareils fonctionnaient toujours, mais l’appel de `/v1/iot/secure/getToken` était rejeté.

Au départ, cela ressemblait à une interface arrêtée ou rendue inutilisable. Une analyse de la requête de l’application officielle SmartHome a toutefois révélé une autre cause : en plus de `udpid`, l’application envoyait le champ `applianceCodes`. La bibliothèque communautaire n’envoyait pas ce champ.

La requête corrigée contient désormais :

```python
data.update({
    "udpid": udp_id,
    "applianceCodes": str(appliance_id)
})
```

Le développeur a testé la modification avec un véritable compte SmartHome et quatre climatiseurs V3 de type `0xAC` :

- Sans `applianceCodes`, le serveur répondait avec l’erreur 3004.
- Avec `applianceCodes`, il fournissait des jetons et clés valides.
- Les valeurs renvoyées fonctionnaient ensuite pour l’authentification locale V3.

L’enquête complète, les résultats des tests et le diff de code sont documentés dans [la pull request n° 470 de `midea-local`](https://github.com/midea-lan/midea-local/pull/470). Le commit immuable correspondant est [`23312799`](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5).

Le code source actuel utilise toujours exactement ce point de terminaison :

```text
/v1/iot/secure/getToken
```

En outre, `applianceCodes` est désormais également envoyé. Cela peut être vérifié directement dans le [code actuel de `midealocal/cloud.py`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py).

Depuis le 12 août 2026, `Midea AC LAN` utilise sa propre bibliothèque `midea-lan` au lieu de `midea-local`; la version actuelle intègre `midea-lan==2026.10.0` et se déclare toujours comme une intégration `local_push`. Ces deux éléments figurent dans le [`manifest.json`](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json) actuel. `midea-lan` récupère lui aussi le jeton et la clé auprès du cloud SmartHome via `/v1/iot/secure/getToken` avec `applianceCodes`, comme le montre le [code de `midealan/cloud.py`](https://github.com/wuwentao/midea-lan/blob/main/midealan/cloud.py).

L’affirmation générale selon laquelle l’API de jetons SmartHome aurait été fermée est donc réfutée, au moins pour les comptes et appareils testés en juin 2026. La formulation correcte serait :

> L’ancienne récupération de jetons ne fonctionnait plus après une modification du format de requête attendu. Après adaptation au format utilisé par l’application officielle, le même point de terminaison V1 a de nouveau fourni des identifiants locaux valides.

Cela n’exclut pas des différences régionales, des comptes différents ou des types d’appareils non pris en charge. Il ne s’agissait toutefois manifestement pas d’un arrêt global.

La situation a depuis évolué pour d’autres clouds Midea. Depuis août 2026, le cloud chinois Meiju répond à `/v1/iot/secure/getToken` avec l’erreur `40404`; il ne fournit plus le jeton et la clé que par le point de terminaison de l’application `/v2/iot/secure/getToken`, qui exige en outre `homegroupId`. Les bibliothèques ont été adaptées en conséquence fin août 2026, comme documenté dans [la pull request n° 717 de `midea-local`](https://github.com/midea-lan/midea-local/pull/717). Selon le [README de `Midea AC LAN`](https://github.com/wuwentao/midea_ac_lan#1-important-notice), il n’est actuellement plus possible de récupérer des jetons avec des comptes NetHome Plus ; seuls les comptes personnels Meiju et SmartHome auxquels l’appareil est associé peuvent être utilisés. Ce changement concerne lui aussi un point de terminaison privé d’application, et non l’API V2 officielle cloud-à-cloud.

## Pourquoi « V2 » est si facilement mal compris ici

Dans l’environnement Midea, au moins trois désignations de version indépendantes les unes des autres sont utilisées.

| Terme | Signification |
| --- | --- |
| Protocole local V2/V3 | Génération de communication directe entre l’intégration et l’appareil |
| Point de terminaison d’application V1/V2 | Numéro de version d’un point de terminaison HTTP individuel dans le backend des applications Midea |
| API V2 cloud-à-cloud | API partenaire officielle destinée aux entreprises tierces autorisées |

### V2 et V3 locaux

Dans le protocole local des appareils, V2 ou V3 désigne la génération de communication de l’appareil. Les appareils V3 plus récents nécessitent un jeton et une clé pour l’authentification locale. `Midea Smart AC` documente cette condition dans son [guide de configuration](https://github.com/mill1000/midea-ac-py#manual-configuration).

Cette version de protocole n’a rien à voir avec l’API V2 officielle cloud-à-cloud.

### V1 et V2 dans les URL d’applications

Même au sein d’une même application, des points de terminaison portant des numéros de version différents peuvent être utilisés simultanément. Un `/v2/` dans le chemin de l’URL ne signifie donc pas que l’ensemble de la plateforme a été migré vers une nouvelle architecture.

Le code actuel de `midea-local` utilise toujours [`/v1/iot/secure/getToken`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) pour le jeton et la clé. D’autres fonctions peuvent néanmoins se trouver sous des chemins dont les versions diffèrent.

### API V2 officielle cloud-à-cloud

Midea documente bel et bien une [API V2 officielle cloud-à-cloud](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Elle utilise notamment :

- OAuth 2.0
- `client_id` et `client_secret`
- des jetons d’accès et des jetons de rafraîchissement de courte durée
- des signatures HMAC-SHA256
- `/v2/open/oauth2/authorize`
- `/v2/open/oauth2/token`
- `/v2/open/device/list/get`
- des requêtes d’état et commandes de pilotage basées sur le cloud

Il s’agit d’une interface partenaire contrôlée. Le `client_secret` requis est attribué par Midea à un fournisseur tiers. Un propriétaire normal de PortaSplit ne l’obtient pas simplement via son compte MSmartHome. Les exigences et règles de signature sont décrites dans la [documentation V2 officielle](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Cette API n’est d’ailleurs pas apparue seulement en 2025. La documentation contient des exemples de requêtes avec des horodatages de 2018 ainsi qu’un commentaire Java du 18 avril 2019. L’interface partenaire V2 existait donc déjà bien avant l’avertissement dans `Midea AC LAN`.

## Midea remplace bien une API V1, mais une autre

Midea maintient également une ancienne interface officielle cloud-à-cloud sous `/v1/open/...`. Sa documentation indique explicitement qu’elle n’est plus recommandée, qu’elle pourrait être arrêtée à l’avenir et que la nouvelle documentation V2 devrait être utilisée. Cela figure dans la [documentation Midea de l’ancienne API cloud-à-cloud](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html).

Cette indication correspond à une véritable migration officielle de V1 vers V2. Elle concerne toutefois les points de terminaison partenaires :

```text
/v1/open/...
           ↓
/v2/open/...
```

La récupération de jetons utilisée par les bibliothèques Home Assistant est en revanche :

```text
/v1/iot/secure/getToken
```

Et la connexion locale de la PortaSplit ne passe ensuite plus du tout par une telle URL cloud, mais directement par le réseau domestique.

Assimiler les trois interfaces uniquement en raison du numéro de version « V1 » ne serait donc pas techniquement justifié.

## Existe-t-il déjà une intégration Home Assistant entièrement basée sur le cloud ?

Avec [`Midea Auto Cloud`](https://github.com/sususweet/midea_auto_cloud), il existe désormais une intégration communautaire qui pilote les appareils Midea via le cloud plutôt que directement par le LAN.

Cela ne prouve toutefois pas non plus que l’API partenaire V2 officielle aurait déjà remplacé le pilotage local. Le code source actuel de `Midea Auto Cloud` utilise notamment :

```text
/v1/appliance/transparent/send
/mjl/v1/device/status/lua/get
/mjl/v1/device/lua/control
```

Ces points de terminaison sont visibles dans le [code actuel de `core/cloud.py`](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py).

L’intégration reproduit donc des fonctions privées d’application ou de cloud grand public. Elle n’utilise pas simplement l’interface partenaire documentée `/v2/open/...`.

Une alternative basée sur le cloud existe donc déjà. Elle apporte toutefois les dépendances habituelles d’une intégration cloud : accès Internet, compte utilisateur fonctionnel, serveurs Midea disponibles et points de terminaison privés toujours compatibles.

## Qu’est-ce que cela signifie concrètement pour les propriétaires de PortaSplit ?

### Pilotage local déjà configuré

Pour une PortaSplit déjà configurée, la situation est relativement peu critique. `Midea Smart AC` enregistre le jeton et la clé localement après la configuration et, selon sa propre [documentation cloud](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage), ne nécessite plus de connexion cloud pour le pilotage ultérieur.

L’arrêt de la seule récupération de jetons ne mettrait donc pas automatiquement fin à la connexion locale existante.

### Nouvelle configuration ou restauration

Le risque est plus important dans les cas suivants :

- une nouvelle installation Home Assistant
- le passage à une autre intégration
- une sauvegarde perdue ou endommagée
- le remplacement du module Wi-Fi
- des modifications de l’association de l’appareil
- un nouvel appairage, si les identifiants de l’appareil changent à cette occasion

Dans de tels cas, l’intégration doit récupérer à nouveau le jeton et la clé, ou l’utilisateur doit les fournir manuellement. Le fait que `Midea Smart AC` prenne en charge une configuration manuelle est décrit dans sa [documentation de configuration](https://github.com/mill1000/midea-ac-py#manual-configuration).

Il n’est pas officiellement documenté qu’une réinitialisation d’usine ou un nouvel appairage génère obligatoirement de nouveaux identifiants pour chaque PortaSplit ; il ne faut donc pas l’affirmer de manière générale.

### Un véritable arrêt du pilotage LAN

Pour qu’une PortaSplit déjà configurée cesse d’accepter ses identifiants stockés localement, le comportement de l’appareil ou du module Wi-Fi devrait également changer, par exemple via un nouveau micrologiciel ou une procédure d’authentification modifiée.

Le simple arrêt du point de terminaison cloud `/v1/iot/secure/getToken` ne supprime pas automatiquement les identifiants déjà présents dans l’appareil et dans Home Assistant. Cela découle de la séparation, documentée par [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage), entre récupération unique via le cloud et pilotage ultérieur sur le LAN.

Une telle modification future de l’appareil est techniquement possible. Je n’ai toutefois trouvé dans les documents Midea accessibles au public aucune annonce concrète ni date d’arrêt spécifiquement pour la PortaSplit.

## Ce que je continuerais à recommander

Malgré ces constats relativisant la situation, une sauvegarde reste utile.

Pour les appareils V3, `Midea AC LAN` recommande explicitement de sauvegarder la configuration JSON générée en dehors de HAOS. La recommandation actuelle figure directement dans le [README du projet](https://github.com/wuwentao/midea_ac_lan#1-important-notice).

Une sauvegarde constitue une protection raisonnable contre les changements du cloud, les problèmes d’intégration et les erreurs personnelles, mais elle n’indique pas qu’un arrêt est imminent. La [partie 2](/blog/midea-portasplit-home-assistant-absichern#token-key-und-konfiguration-sichern) explique comment sauvegarder le jeton, la clé et la configuration.

## Évaluation sur la base des éléments disponibles

L’avertissement de `Midea AC LAN` doit être pris au sérieux, mais correctement replacé dans son contexte.

Il documente un risque plausible à long terme : Midea pourrait considérer les jetons locaux sans expiration comme un problème de sécurité, limiter davantage l’obtention de tels jetons ou lier plus étroitement les futurs appareils au cloud.

En revanche, un arrêt officiellement annoncé et daté du pilotage local de la PortaSplit n’est pas établi.

L’état technique actuel montre même le contraire d’un arrêt déjà effectif : en juin 2026, le point de terminaison V1 de jetons toujours utilisé a fourni des identifiants valides après adaptation de la requête au format de l’application SmartHome officielle. Le correctif correspondant fait aujourd’hui partie de la bibliothèque utilisée par `Midea AC LAN`.

L’API V2 officielle Midea cloud-à-cloud existe également. Il s’agit toutefois d’une interface partenaire plus ancienne et à accès restreint, et non automatiquement du successeur du protocole local de la PortaSplit.

La conclusion sobre est donc la suivante :

> Créez une sauvegarde, surveillez les intégrations et gardez les dépendances au cloud à l’esprit, mais ne condamnez pas prématurément le pilotage local de la PortaSplit sur la base d’une hypothèse d’arrêt non confirmée.

## Historique des modifications

- **25 octobre 2026 :** Ajout : depuis août 2026, Midea AC LAN utilise la bibliothèque midea-lan (actuellement 2026.10.0), et le cloud Meiju ne fournit plus le jeton et la clé que via le point de terminaison /v2/iot/secure/getToken avec homegroupId. Article mis à jour au 25 octobre 2026, sources complétées.

## Sources

1.  [Midea AC LAN : README actuel et avertissement d’arrêt](https://github.com/wuwentao/midea_ac_lan#1-important-notice) : libellé de l’avertissement, recommandation de sauvegarde et distinction entre les anciens appareils V2 et les appareils V3 plus récents.

2.  [Midea AC LAN PR n° 578 du 19 mai 2025](https://github.com/wuwentao/midea_ac_lan/pull/578) : introduction de l’avertissement concernant l’arrêt progressif des services de jetons et la migration prétendue vers une API V2 basée sur le cloud.

3.  [Midea AC LAN PR n° 639](https://github.com/wuwentao/midea_ac_lan/pull/639) : changement de la source de jetons documentée vers NetHome Plus.

4.  [midea-msmart issue n° 201](https://github.com/mill1000/midea-msmart/issues/201) : discussion sur la récupération défaillante des jetons SmartHome et l’utilisation temporaire de NetHome Plus.

5.  [Commentaire du mainteneur de Midea AC LAN sur la migration V2 supposée](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457) : qualifie explicitement l’affirmation sur le nouveau cloud V2 de compréhension personnelle.

6.  [Réponse du mainteneur de midea-msmart](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109) : décrit l’existence d’une nouvelle API V2 comme une supposition et souligne les possibilités limitées de rétro-ingénierie.

7.  [midea-local PR n° 470 du 15 juin 2026](https://github.com/midea-lan/midea-local/pull/470) : analyse de l’erreur 3004, capture de la requête de l’application officielle, ajout de `applianceCodes` et test réussi avec quatre climatiseurs V3.

8.  [Commit immuable du correctif SmartHome-getToken](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5) : diff de code exact du correctif intégré.

9.  [Code cloud actuel de midea-local](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) : point de terminaison toujours utilisé `/v1/iot/secure/getToken` et champ de requête actuel `applianceCodes`.

10.  [Manifeste actuel de Midea AC LAN](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json) : version utilisée de `midea-lan` et classification comme intégration locale Push.

11.  [Midea Smart AC](https://github.com/mill1000/midea-ac-py) : documentation du pilotage local, de la récupération unique via le cloud pour les appareils V3 et de la configuration manuelle avec jeton et clé.

12.  [Midea AC LAN issue n° 607 sur la PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/607) : exemple concret de PortaSplit avec appareil de type `0xAC`, modèle `00000Q1D`, version de protocole 3 et configuration réussie via NetHome Plus.

13.  [API V2 officielle Midea cloud-à-cloud](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html) : OAuth2, Client-ID, Client-Secret, jetons d’accès et de rafraîchissement, procédé de signature et points de terminaison `/v2/open/...`.

14.  [API V1 officielle Midea cloud-à-cloud](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html) : indication officielle selon laquelle l’ancienne interface partenaire `/v1/open/...` n’est plus recommandée et pourrait être arrêtée à l’avenir.

15.  [Midea Auto Cloud](https://github.com/sususweet/midea_auto_cloud) et [code cloud actuel](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py) : intégration communautaire pour le pilotage entièrement via le cloud et points de terminaison privés d’application V1 effectivement utilisés.

16.  [midea-local PR n° 717](https://github.com/midea-lan/midea-local/pull/717) : passage du cloud Meiju de `/v1/iot/secure/getToken` (erreur 40404) à `/v2/iot/secure/getToken` avec `homegroupId`.

17.  [Code cloud actuel de midea-lan](https://github.com/wuwentao/midea-lan/blob/main/midealan/cloud.py) : bibliothèque utilisée par Midea AC LAN depuis août 2026, récupération de jetons SmartHome via `/v1/iot/secure/getToken` avec `applianceCodes`.
