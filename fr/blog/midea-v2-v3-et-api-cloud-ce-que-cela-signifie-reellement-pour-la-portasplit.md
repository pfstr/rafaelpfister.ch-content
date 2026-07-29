---
title: "Midea V2, V3 et API cloud : ce que cela signifie réellement pour la PortaSplit"
navTitle: "API cloud Midea V2"
description: "Le protocole local des appareils, les points de terminaison privés des applications et l’API partenaire officielle utilisent des noms de version similaires. L’analyse des sources distingue ces niveaux et replace l’avertissement d’arrêt dans son contexte."
date: "2026-07-25"
kategorie: "Maison connectée & IoT"
timeToRead: "11 min de lecture"
themen:
  - "smart-home-iot"
related:
  - "midea-portasplit-home-assistant"
  - "midea-portasplit-home-assistant-einrichten"
draft: false
slug: "midea-v2-v3-et-api-cloud-ce-que-cela-signifie-reellement-pour-la-portasplit"
translationOf: "midea-v2-cloud-api-portasplit-home-assistant"
url: "https://rafaelpfister.ch/fr/blog/midea-v2-v3-et-api-cloud-ce-que-cela-signifie-reellement-pour-la-portasplit"
---

Dans l’univers de la Midea PortaSplit, « V2 » désigne plusieurs choses indépendantes les unes des autres. Il existe un protocole local V2 pour les appareils, des numéros de version dans des points de terminaison privés d’applications et une API officielle cloud-à-cloud V2 destinée aux partenaires. Confondre ces niveaux conduit inévitablement à de fausses conclusions sur le contrôle local.

Le projet `Midea AC LAN` avertit dans son [README](https://github.com/wuwentao/midea_ac_lan#1-important-notice) que les interfaces de jetons utilisées jusqu’à présent seraient fermées et remplacées par une API V2 basée sur le cloud. L’examen des discussions, du code actuel et de la documentation officielle de Midea donne une image plus nuancée :

> Une API officielle Midea cloud-à-cloud V2 existe. Mais elle n’est identique ni à l’interface de jetons utilisée par Home Assistant, ni au protocole local V2 ou V3 des appareils. Aucun arrêt officiellement annoncé du contrôle local de la PortaSplit à une date précise n’est documenté. En juin 2026, il a en outre été démontré que l’API de jetons SmartHome prétendument arrêtée fonctionnait toujours : la requête antérieure de la bibliothèque communautaire était simplement incomplète.

Cet article est à jour au 25 juillet 2026.

## Pourquoi l’interprétation précédente doit être corrigée

Dans le [premier article sur la question des jetons cloud](/blog/midea-portasplit-home-assistant), j’avais rapporté l’avertissement du projet `Midea AC LAN` en substance comme l’annonce d’un arrêt des interfaces cloud. Cela correspondait au libellé du README du projet, mais était formulé de manière trop catégorique comme une affirmation factuelle.

L’avertissement reste pertinent en tant qu’indication de risque. Il ne constitue toutefois pas une feuille de route publiée par Midea. Surtout, de nouveaux éléments techniques sont désormais disponibles et remettent en question une partie essentielle de l’interprétation précédente.

## Comment fonctionne le contrôle local de la PortaSplit

L’intégration Home Assistant `Midea Smart AC` décrit explicitement son architecture comme un contrôle local. Pour les appareils V3 plus récents, le cloud Midea est utilisé uniquement lors de la configuration afin d’obtenir un jeton et une clé propres à l’appareil. L’intégration enregistre ensuite ces deux valeurs localement et n’a plus besoin d’aucune connexion cloud pour le contrôle proprement dit. Le projet le documente sous [« Note On Cloud Usage »](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

Schématiquement, le processus se présente ainsi :

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

Pour les appareils V3 configurés manuellement, `Midea Smart AC` exige l’identifiant de l’appareil, l’adresse IP, le port, le jeton et la clé. Le port standard documenté est `6444/TCP`; le jeton et la clé sont indiqués avec respectivement 128 et 64 caractères hexadécimaux. Ces informations figurent dans la [documentation relative à la configuration manuelle](https://github.com/mill1000/midea-ac-py#manual-configuration).

Dans le suivi des tickets de `Midea AC LAN`, une PortaSplit a par exemple été détectée comme appareil de type `0xAC`, modèle `00000Q1D` et version de protocole 3. Le même utilisateur a ensuite pu l’ajouter à Home Assistant via NetHome Plus. Le déroulement concret est documenté dans [l’issue n° 607](https://github.com/wuwentao/midea_ac_lan/issues/607).

La distinction est essentielle :

- Le service cloud est utilisé pour obtenir les identifiants d’accès locaux.
- Le contrôle ultérieur s’effectue directement sur le réseau local.
- Une défaillance du service de jetons empêche donc avant tout les nouvelles configurations.
- Elle ne met pas automatiquement fin à une connexion locale déjà configurée.

Ce dernier point correspond également à la description explicite de [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

## D’où vient l’avertissement d’arrêt

Le texte d’avertissement actuellement visible a été ajouté à la documentation le 19 mai 2025 avec la [pull request n° 578](https://github.com/wuwentao/midea_ac_lan/pull/578).

Son raisonnement peut être résumé comme suit :

- Les jetons locaux n’auraient pas de date d’expiration.
- Différents projets Home Assistant utiliseraient un chiffrement d’application reproduit ou extrait.
- Il en résulterait un problème de sécurité.
- Midea fermerait donc progressivement les services de jetons existants.
- À long terme, le contrôle local V1 serait remplacé par une API V2 basée sur le cloud.

En juillet 2025, la documentation a de nouveau été adaptée via la [pull request n° 639](https://github.com/wuwentao/midea_ac_lan/pull/639). Au lieu du cloud SmartHome, NetHome Plus était désormais mentionné comme source temporaire de jetons. L’avertissement d’arrêt proprement dit est resté en place.

La discussion sous-jacente est toutefois formulée avec plus de prudence que le README.

Dans le [commentaire du mainteneur de Midea-AC-LAN](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457), il est indiqué en substance que NetHome Plus pourrait n’être qu’une solution temporaire et que Midea disposerait, selon sa compréhension, d’un nouveau service V2 entièrement basé sur le cloud.

Le mainteneur de `midea-msmart` a répondu qu’il soupçonnait également l’existence d’une nouvelle API V2, mais qu’il ne pouvait l’étudier que de manière limitée faute de posséder lui-même des appareils Midea. Cela figure dans le [commentaire de réponse direct](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109).

La situation des sources est donc plus claire :

- L’avertissement provient de développeurs expérimentés de la communauté.
- Il repose sur des changements observés et leur évaluation technique.
- L’un des mainteneurs qualifie explicitement la migration V2 de sa propre compréhension.
- L’autre parle d’une supposition.
- Ni la pull request ni la discussion ne renvoient vers une annonce officielle de Midea concernant un arrêt, ni vers une date.

Cela ne rend pas l’avertissement inutile. Mais cela en fait une analyse de risque, et non une feuille de route confirmée par le fabricant.

## La découverte déterminante de juin 2026

Le 15 juin 2026, un correctif a été intégré dans la bibliothèque `midea-local`, ce qui modifie considérablement l’interprétation précédente.

Le point de départ était l’erreur suivante :

```json
{
  "code": "3004",
  "msg": "value is illegal."
}
```

Cette erreur s’était produite lors de la récupération du jeton et de la clé via le cloud SmartHome. La connexion et la liste des appareils continuaient de fonctionner, mais l’appel à `/v1/iot/secure/getToken` était rejeté.

Au départ, cela ressemblait à une interface arrêtée ou rendue inutilisable. L’analyse de la requête de l’application officielle SmartHome a toutefois révélé une autre cause : en plus de `udpid`, l’application envoyait le champ `applianceCodes`. La bibliothèque communautaire n’envoyait pas ce champ.

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

L’enquête complète, les résultats des tests et le diff du code sont documentés dans la [pull request n° 470 de `midea-local`](https://github.com/midea-lan/midea-local/pull/470). Le commit immuable associé est [`23312799`](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5).

Le code source actuel utilise toujours exactement ce point de terminaison :

```text
/v1/iot/secure/getToken
```

En outre, `applianceCodes` est désormais également envoyé. Cela peut être vérifié directement dans le [`midealocal/cloud.py`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) actuel.

La version actuelle de `Midea AC LAN` intègre `midea-local==6.11.0` et se déclare toujours comme une intégration `local_push`. Ces deux informations figurent dans le [`manifest.json`](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json) actuel.

L’affirmation générale selon laquelle l’API de jetons SmartHome aurait été fermée est donc réfutée, au moins pour les comptes et appareils testés en juin 2026. La formulation correcte serait :

> L’ancienne récupération de jetons ne fonctionnait plus après une modification du format de requête attendu. Après adaptation au format utilisé par l’application officielle, le même point de terminaison V1 a de nouveau fourni des identifiants d’accès locaux valides.

Cela n’exclut pas les différences régionales, les comptes différents ou les types d’appareils non pris en charge. Mais il ne s’agissait manifestement pas d’un arrêt global.

## Pourquoi « V2 » est si facile à mal comprendre ici

Dans l’environnement Midea, au moins trois désignations de version indépendantes les unes des autres sont utilisées.

| Terme | Signification |
| --- | --- |
| Protocole local V2/V3 | Génération de la communication directe entre l’intégration et l’appareil |
| Point de terminaison d’application V1/V2 | Numéro de version d’un point de terminaison HTTP individuel dans le backend des applications Midea |
| API cloud-à-cloud V2 | API partenaire officielle destinée aux entreprises tierces autorisées |

### V2 et V3 locaux

Dans le protocole local des appareils, V2 et V3 désignent la génération de communication de l’appareil. Les appareils V3 plus récents nécessitent un jeton et une clé pour l’authentification locale. `Midea Smart AC` documente cette condition dans son [guide de configuration](https://github.com/mill1000/midea-ac-py#manual-configuration).

Cette version de protocole n’a rien à voir avec l’API officielle cloud-à-cloud V2.

### V1 et V2 dans les URL des applications

Même au sein d’une même application, des points de terminaison ayant différents numéros de version peuvent être utilisés simultanément. Un `/v2/` dans le chemin d’URL ne signifie donc pas que l’ensemble de la plateforme a été migré vers une nouvelle architecture.

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

Il s’agit d’une interface partenaire contrôlée. Le `client_secret` requis est attribué à un fournisseur tiers par Midea. Un propriétaire normal de PortaSplit ne l’obtient pas simplement via son compte MSmartHome. Les exigences et règles de signature sont décrites dans la [documentation officielle V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Cette API n’est par ailleurs pas apparue seulement en 2025. La documentation contient des exemples de requêtes avec des horodatages datant de 2018 et un commentaire Java du 18 avril 2019. L’interface partenaire V2 existait donc déjà bien avant l’avertissement de `Midea AC LAN`.

## Midea remplace effectivement une API V1, mais une autre

Midea propose également une ancienne interface officielle cloud-à-cloud sous `/v1/open/...`. Sa documentation comporte explicitement l’indication qu’elle n’est plus recommandée, qu’elle pourrait être arrêtée à l’avenir et qu’il convient d’utiliser la nouvelle documentation V2. Cela figure dans la [documentation Midea de l’ancienne API cloud-à-cloud](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html).

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

Et la connexion locale à la PortaSplit ne passe ensuite plus du tout par une telle URL cloud, mais directement par le réseau domestique.

Assimiler les trois interfaces uniquement en raison du numéro de version « V1 » ne serait donc pas techniquement justifié.

## Existe-t-il déjà une intégration Home Assistant entièrement basée sur le cloud ?

Avec [`Midea Auto Cloud`](https://github.com/sususweet/midea_auto_cloud), il existe désormais une intégration communautaire qui contrôle les appareils Midea via le cloud plutôt que directement par le réseau local.

Cela ne constitue toutefois pas non plus la preuve que l’API partenaire officielle V2 aurait déjà remplacé le contrôle local. Le code source actuel de `Midea Auto Cloud` utilise notamment :

```text
/v1/appliance/transparent/send
/mjl/v1/device/status/lua/get
/mjl/v1/device/lua/control
```

Ces points de terminaison sont consultables dans le [`core/cloud.py`](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py) actuel.

L’intégration reproduit ainsi des fonctions privées d’applications ou du cloud grand public. Elle n’utilise pas simplement l’interface partenaire documentée `/v2/open/...`.

Une alternative basée sur le cloud existe donc déjà. Mais elle entraîne également les dépendances habituelles d’une intégration cloud : accès à Internet, compte utilisateur fonctionnel, serveurs Midea disponibles et points de terminaison privés restant compatibles.

## Qu’est-ce que cela signifie concrètement pour les propriétaires de PortaSplit ?

### Contrôle local déjà configuré

Pour une PortaSplit déjà configurée, la situation est relativement sereine. `Midea Smart AC` enregistre le jeton et la clé localement après la configuration et, selon sa propre [documentation cloud](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage), n’a besoin d’aucune connexion cloud pour la suite du contrôle.

Un arrêt de la simple récupération de jetons ne mettrait donc pas automatiquement fin à la connexion locale existante.

### Nouvelle configuration ou restauration

Le risque est plus important dans les cas suivants :

- une nouvelle installation de Home Assistant
- le passage à une autre intégration
- une sauvegarde perdue ou endommagée
- le remplacement du module Wi-Fi
- des modifications de l’association de l’appareil
- un nouvel appairage si les identifiants d’accès de l’appareil changent à cette occasion

Dans ces cas, l’intégration doit obtenir à nouveau le jeton et la clé, ou l’utilisateur doit les fournir manuellement. Le fait que `Midea Smart AC` prend en charge une configuration manuelle est décrit dans sa [documentation de configuration](https://github.com/mill1000/midea-ac-py#manual-configuration).

Il n’est pas officiellement documenté qu’une réinitialisation d’usine ou un nouvel appairage génère obligatoirement de nouveaux identifiants d’accès pour chaque PortaSplit ; il ne faut donc pas l’affirmer de manière générale.

### Un véritable arrêt du contrôle par LAN

Pour qu’une PortaSplit déjà configurée n’accepte plus ses identifiants d’accès enregistrés localement, le comportement de l’appareil ou du module Wi-Fi devrait aussi changer, par exemple via un nouveau micrologiciel ou une procédure d’authentification modifiée.

Un simple arrêt du point de terminaison cloud `/v1/iot/secure/getToken` ne supprime pas automatiquement les identifiants d’accès déjà présents dans l’appareil et dans Home Assistant. Cela découle de la séparation entre la récupération unique dans le cloud et le contrôle ultérieur par LAN, documentée par [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

Une telle modification future des appareils est techniquement possible. Je n’ai toutefois trouvé dans la documentation Midea publiquement accessible aucune annonce concrète ni date d’arrêt spécifique à la PortaSplit.

## Ce que je continuerais à recommander

Malgré ces éléments relativisant la situation, une sauvegarde reste judicieuse.

Pour les appareils V3, `Midea AC LAN` recommande explicitement de sauvegarder la configuration JSON générée en dehors de HAOS. La recommandation actuelle figure directement dans le [README du projet](https://github.com/wuwentao/midea_ac_lan#1-important-notice).

Il convient de :

- Traiter le jeton et la clé comme des mots de passe.
- Ne pas téléverser le fichier JSON dans un dépôt Git public.
- Ne pas publier de journaux de débogage non masqués.
- Chiffrer la sauvegarde.
- Créer également une sauvegarde complète de Home Assistant.
- Vérifier le fonctionnement actuel avant les mises à jour de micrologiciel et d’intégration.
- Tester à nouveau le contrôle local après les mises à jour.

Une sauvegarde constitue une protection raisonnable contre les changements du cloud, les problèmes d’intégration et les erreurs personnelles. Elle ne signifie pas qu’un arrêt est imminent. La manière de configurer proprement une PortaSplit et de la sécuriser sur le réseau domestique est expliquée dans la [partie pratique consacrée à la configuration](/blog/midea-portasplit-home-assistant-einrichten).

## Évaluation sur la base des preuves disponibles

L’avertissement de `Midea AC LAN` doit être pris au sérieux, mais interprété correctement.

Il documente un risque plausible à long terme : Midea pourrait considérer les jetons locaux sans expiration comme un problème de sécurité, restreindre davantage l’obtention de tels jetons ou lier plus fortement les futurs appareils au cloud.

En revanche, aucun arrêt officiellement annoncé et daté du contrôle local de la PortaSplit n’est étayé.

L’état technique actuel montre même le contraire d’un arrêt déjà réalisé : en juin 2026, le point de terminaison de jetons V1 toujours utilisé a fourni des identifiants valides après adaptation de la requête au format de l’application officielle SmartHome. Le correctif correspondant fait aujourd’hui partie de la bibliothèque utilisée par `Midea AC LAN`.

L’API officielle Midea cloud-à-cloud V2 existe également. Mais il s’agit d’une ancienne interface partenaire à accès restreint, et non automatiquement du successeur du protocole local de la PortaSplit.

La conclusion sobre est donc la suivante :

> Créez une sauvegarde, surveillez les intégrations et gardez les dépendances cloud à l’esprit, mais n’abandonnez pas prématurément le contrôle local de la PortaSplit sur la base d’une hypothèse d’arrêt non confirmée.

## Sources

1.  [Midea AC LAN : README actuel et avertissement d’arrêt](https://github.com/wuwentao/midea_ac_lan#1-important-notice) : libellé de l’avertissement, recommandation de sauvegarde et distinction entre les anciens appareils V2 et les appareils V3 plus récents.

2.  [Midea AC LAN PR n° 578 du 19 mai 2025](https://github.com/wuwentao/midea_ac_lan/pull/578) : introduction de l’avertissement concernant l’arrêt progressif des services de jetons et la migration prétendue vers une API V2 basée sur le cloud.

3.  [Midea AC LAN PR n° 639](https://github.com/wuwentao/midea_ac_lan/pull/639) : remplacement de la source de jetons documentée par NetHome Plus.

4.  [midea-msmart issue n° 201](https://github.com/mill1000/midea-msmart/issues/201) : discussion sur la récupération défaillante des jetons SmartHome et l’utilisation temporaire de NetHome Plus.

5.  [Commentaire du mainteneur de Midea-AC-LAN sur la migration V2 supposée](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457) : qualifie explicitement l’affirmation concernant le nouveau cloud V2 de sa propre compréhension.

6.  [Réponse du mainteneur de midea-msmart](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109) : décrit l’existence d’une nouvelle API V2 comme une supposition et souligne les possibilités limitées de rétro-ingénierie.

7.  [midea-local PR n° 470 du 15 juin 2026](https://github.com/midea-lan/midea-local/pull/470) : analyse de l’erreur 3004, capture de la requête de l’application officielle, ajout de `applianceCodes` et test concluant avec quatre climatiseurs V3.

8.  [Commit immuable du correctif SmartHome-getToken](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5) : diff exact du code du correctif intégré.

9.  [Code cloud midea-local actuel](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) : point de terminaison `/v1/iot/secure/getToken` toujours utilisé et champ de requête actuel `applianceCodes`.

10.  [Manifeste actuel de Midea AC LAN](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json) : version utilisée de `midea-local` et classification en tant qu’intégration push locale.

11.  [Midea Smart AC](https://github.com/mill1000/midea-ac-py) : documentation du contrôle local, de la récupération cloud unique pour les appareils V3 et de la configuration manuelle avec jeton et clé.

12.  [Midea AC LAN issue n° 607 concernant la PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/607) : exemple concret de PortaSplit avec appareil de type `0xAC`, modèle `00000Q1D`, version de protocole 3 et configuration réussie via NetHome Plus.

13.  [API officielle Midea cloud-à-cloud V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html) : OAuth2, identifiant client, secret client, jetons d’accès et de rafraîchissement, procédure de signature et points de terminaison `/v2/open/...`.

14.  [API officielle Midea cloud-à-cloud V1](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html) : indication officielle selon laquelle l’ancienne interface partenaire `/v1/open/...` n’est plus recommandée et pourrait être arrêtée à l’avenir.

15.  [Midea Auto Cloud](https://github.com/sususweet/midea_auto_cloud) et [code cloud actuel](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py) : intégration communautaire pour un contrôle entièrement basé sur le cloud et points de terminaison privés V1 des applications effectivement utilisés.
