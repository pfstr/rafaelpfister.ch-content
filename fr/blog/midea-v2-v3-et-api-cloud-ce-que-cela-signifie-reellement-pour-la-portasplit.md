---
title: "Midea V2, V3 et API Cloud : ce que cela signifie réellement pour la PortaSplit"
navTitle: "API Cloud Midea V2"
description: "Le protocole local des appareils, les points de terminaison privés des applications et l’API partenaire officielle utilisent des noms de version similaires. L’analyse des sources distingue ces niveaux et remet l’avertissement d’arrêt en contexte."
date: "2026-07-25"
kategorie: "Home Assistant et IoT"
timeToRead: "11 min de lecture"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant
  - midea-portasplit-home-assistant-einrichten
draft: false
slug: "midea-v2-v3-et-api-cloud-ce-que-cela-signifie-reellement-pour-la-portasplit"
translationOf: "midea-v2-cloud-api-portasplit-home-assistant"
translationId: article-f504b2af00493864
translationModel: gpt-5.6-terra
translatedAt: 2026-08-08T14:08:57.451Z
translationReview: automatic
translationSourceHash: e1fb72bbe36ce246f02afd3d44a07a5d954f27ea67eb0e0b0bb4a1967dac935c
url: https://rafaelpfister.ch/fr/blog/midea-v2-v3-et-api-cloud-ce-que-cela-signifie-reellement-pour-la-portasplit
---

Dans l’univers de la Midea PortaSplit, « V2 » désigne plusieurs choses indépendantes les unes des autres. Il existe un protocole local V2 pour les appareils, des numéros de version dans des points de terminaison privés d’applications et une API officielle Cloud-to-Cloud V2 destinée aux partenaires. Assimiler ces niveaux conduit inévitablement à des conclusions erronées sur le contrôle local.

Le projet `Midea AC LAN` avertit dans son [README](https://github.com/wuwentao/midea_ac_lan#1-important-notice) que les interfaces de jetons utilisées jusqu’ici seraient fermées et remplacées par une API V2 basée sur le cloud. L’examen des discussions, du code actuel et de la documentation officielle de Midea donne une image plus nuancée :

> Une API officielle Midea Cloud-to-Cloud V2 existe. Elle n’est toutefois identique ni à l’interface de jetons utilisée par Home Assistant, ni au protocole local V2 ou V3 des appareils. Aucun arrêt officiellement annoncé du contrôle local de la PortaSplit à une date précise n’est documenté. En juin 2026, il a en outre été démontré que l’API de jetons SmartHome supposément arrêtée fonctionnait toujours : la requête précédente de la bibliothèque communautaire était simplement incomplète.

Cet article est à jour au 25 juillet 2026.

## Pourquoi l’évaluation précédente doit être corrigée

Dans le [premier article sur la question des jetons cloud](/blog/midea-portasplit-home-assistant), j’avais présenté l’avertissement du projet `Midea AC LAN` comme l’annonce d’un arrêt des interfaces cloud. Cela correspondait au libellé du README du projet, mais la formulation était trop catégorique en tant qu’affirmation factuelle.

L’avertissement reste pertinent comme indication de risque. Il ne constitue toutefois pas une feuille de route publiée par Midea. Surtout, de nouveaux éléments techniques sont désormais disponibles et remettent en cause une part essentielle de l’interprétation précédente.

## Comment fonctionne le contrôle local de la PortaSplit

L’intégration Home Assistant `Midea Smart AC` décrit explicitement son architecture comme un contrôle local. Sur les appareils V3 plus récents, le cloud Midea n’est utilisé que lors de la configuration afin d’obtenir un jeton et une clé spécifiques à l’appareil. L’intégration stocke ensuite ces deux valeurs localement et ne nécessite plus de connexion cloud pour le contrôle proprement dit. Le projet le documente dans [« Note On Cloud Usage »](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

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

Pour les appareils V3 configurés manuellement, `Midea Smart AC` demande l’identifiant de l’appareil, l’adresse IP, le port, le jeton et la clé. Le port par défaut documenté est `6444/TCP`; le jeton et la clé sont indiqués comme comportant respectivement 128 et 64 caractères hexadécimaux. Ces informations figurent dans la [documentation de la configuration manuelle](https://github.com/mill1000/midea-ac-py#manual-configuration).

Une PortaSplit a par exemple été détectée dans le suivi des tickets de `Midea AC LAN` comme étant du type d’appareil `0xAC`, du modèle `00000Q1D` et de version de protocole 3. Le même utilisateur a ensuite pu l’ajouter à Home Assistant via NetHome Plus. Le déroulement concret est documenté dans [Issue #607](https://github.com/wuwentao/midea_ac_lan/issues/607).

La distinction est décisive :

- Le service cloud est utilisé pour obtenir les informations d’accès locales.
- Le contrôle ultérieur se fait directement sur le réseau local.
- Une défaillance du service de jetons empêche donc surtout de nouvelles configurations.
- Elle ne met pas automatiquement fin à une connexion locale déjà configurée.

Ce dernier point correspond également à la description explicite de [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

## D’où vient l’avertissement d’arrêt

Le texte d’avertissement visible aujourd’hui a été ajouté à la documentation le 19 mai 2025 avec la [Pull Request #578](https://github.com/wuwentao/midea_ac_lan/pull/578).

Son raisonnement peut être résumé comme suit :

- Les jetons locaux n’auraient pas de date d’expiration.
- Différents projets Home Assistant utiliseraient un chiffrement d’application imité ou extrait.
- Il en résulterait un problème de sécurité.
- Midea fermerait donc progressivement les services de jetons existants.
- À long terme, le contrôle local V1 serait remplacé par une API V2 basée sur le cloud.

En juillet 2025, la documentation a de nouveau été adaptée via la [Pull Request #639](https://github.com/wuwentao/midea_ac_lan/pull/639). Au lieu du cloud SmartHome, NetHome Plus était désormais mentionné comme source provisoire de jetons. L’avertissement d’arrêt lui-même a été conservé.

La discussion sous-jacente est toutefois formulée avec plus de prudence que le README.

Dans le [commentaire du mainteneur de Midea AC LAN](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457), il est indiqué, en substance, que NetHome Plus pourrait n’être qu’une solution temporaire et que Midea disposerait, selon sa compréhension, d’un nouveau service V2 entièrement basé sur le cloud.

Le mainteneur de `midea-msmart` a répondu qu’il soupçonnait lui aussi l’existence d’une nouvelle API V2, mais ne pouvait l’étudier que de manière limitée faute de posséder lui-même des appareils Midea. Cela figure dans le [commentaire de réponse direct](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109).

La situation des sources est donc plus claire :

- L’avertissement provient de développeurs communautaires expérimentés.
- Il se fonde sur des changements observés et sur leur évaluation technique.
- L’un des mainteneurs qualifie explicitement la migration V2 de sa propre compréhension.
- L’autre parle d’une supposition.
- Ni la Pull Request ni la discussion ne renvoient vers une annonce officielle de Midea sur un arrêt ou une date.

Cela ne rend pas l’avertissement inutile. Mais il s’agit d’une analyse de risque, et non d’une feuille de route confirmée du fabricant.

## La constatation décisive de juin 2026

Le 15 juin 2026, un correctif a été intégré dans la bibliothèque `midea-local`, modifiant sensiblement l’interprétation précédente.

Le point de départ était l’erreur suivante :

```json
{
  "code": "3004",
  "msg": "value is illegal."
}
```

Cette erreur s’était produite lors de la récupération du jeton et de la clé via le cloud SmartHome. La connexion et la liste des appareils continuaient de fonctionner, mais l’appel à `/v1/iot/secure/getToken` était rejeté.

Au départ, cela ressemblait à une interface arrêtée ou rendue inutilisable. Une analyse de la requête de l’application officielle SmartHome a toutefois révélé une autre cause : en plus de `udpid`, l’application envoyait le champ `applianceCodes`. La bibliothèque communautaire n’envoyait pas ce champ.

La requête corrigée contient désormais :

```python
data.update({
    "udpid": udp_id,
    "applianceCodes": str(appliance_id)
})
```

Le développeur a testé cette modification avec un vrai compte SmartHome et quatre climatiseurs V3 de type `0xAC` :

- Sans `applianceCodes`, le serveur répondait avec l’erreur 3004.
- Avec `applianceCodes`, il fournissait des jetons et des clés valides.
- Les valeurs renvoyées fonctionnaient ensuite pour l’authentification locale V3.

L’enquête complète, les résultats des tests et la différence de code sont documentés dans la [Pull Request #470 de `midea-local`](https://github.com/midea-lan/midea-local/pull/470). Le commit immuable associé est [`23312799`](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5).

Le code source actuel utilise toujours exactement ce point de terminaison :

```text
/v1/iot/secure/getToken
```

En outre, `applianceCodes` est désormais également envoyé. Cela peut être vérifié directement dans le [code actuel de `midealocal/cloud.py`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py).

La version actuelle de `Midea AC LAN` intègre `midea-local==6.11.0` et se déclare toujours comme une intégration `local_push`. Ces deux éléments figurent dans le [manifeste actuel`manifest.json`](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json).

L’affirmation générale selon laquelle l’API de jetons SmartHome aurait été fermée est ainsi réfutée, au moins pour les comptes et appareils testés en juin 2026. La formulation correcte serait :

> L’ancienne récupération des jetons a cessé de fonctionner après une modification du format de requête attendu. Après adaptation au format utilisé par l’application officielle, le même point de terminaison V1 a de nouveau fourni des informations d’accès locales valides.

Cela n’exclut pas des différences régionales, des comptes différents ou des types d’appareils non pris en charge. Mais il ne s’agissait manifestement pas d’un arrêt global.

## Pourquoi « V2 » est si facilement mal compris ici

Dans l’environnement Midea, au moins trois désignations de version indépendantes sont utilisées.

| Terme | Signification |
| --- | --- |
| Protocole local V2/V3 | Génération de communication directe entre l’intégration et l’appareil |
| Point de terminaison d’application V1/V2 | Numéro de version d’un point de terminaison HTTP individuel dans le backend des applications Midea |
| API Cloud-to-Cloud V2 | API partenaire officielle destinée aux entreprises tierces autorisées |

### V2 et V3 locaux

Dans le protocole local des appareils, V2 ou V3 désigne la génération de communication de l’appareil. Les appareils V3 plus récents nécessitent un jeton et une clé pour l’authentification locale. `Midea Smart AC` documente cette condition dans son [guide de configuration](https://github.com/mill1000/midea-ac-py#manual-configuration).

Cette version de protocole n’a rien à voir avec l’API officielle Cloud-to-Cloud V2.

### V1 et V2 dans les URL d’applications

Même au sein d’une même application, des points de terminaison comportant différents numéros de version peuvent être utilisés simultanément. Un `/v2/` dans le chemin d’une URL ne signifie donc pas que toute la plateforme a été migrée vers une nouvelle architecture.

Le code actuel de `midea-local` utilise toujours [`/v1/iot/secure/getToken`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) pour le jeton et la clé. D’autres fonctions peuvent néanmoins se trouver sous des chemins ayant d’autres versions.

### API officielle Cloud-to-Cloud V2

Midea documente effectivement une [API officielle Cloud-to-Cloud V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Elle utilise notamment :

- OAuth 2.0
- `client_id` et `client_secret`
- des jetons d’accès et des jetons de rafraîchissement à courte durée de vie
- des signatures HMAC-SHA256
- `/v2/open/oauth2/authorize`
- `/v2/open/oauth2/token`
- `/v2/open/device/list/get`
- des interrogations d’état et des commandes de contrôle basées sur le cloud

Il s’agit d’une interface partenaire contrôlée. Le `client_secret` requis est attribué par Midea à un fournisseur tiers. Un propriétaire ordinaire de PortaSplit ne l’obtient pas simplement via son compte MSmartHome. Les exigences et règles de signature sont décrites dans la [documentation officielle V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Cette API n’est d’ailleurs pas apparue en 2025. La documentation contient des exemples de requêtes avec des horodatages de 2018 et un commentaire Java du 18 avril 2019. L’interface partenaire V2 existait donc bien avant l’avertissement de `Midea AC LAN`.

## Midea remplace effectivement une API V1, mais une autre

Midea maintient également une ancienne interface officielle Cloud-to-Cloud sous `/v1/open/...`. Sa documentation indique explicitement qu’elle n’est plus recommandée, qu’elle pourrait être arrêtée à l’avenir et que la nouvelle documentation V2 doit être utilisée. Cela figure dans la [documentation de Midea sur l’ancienne API Cloud-to-Cloud](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html).

Cette indication correspond à une véritable migration officielle de V1 vers V2. Elle concerne toutefois les points de terminaison partenaires :

```text
/v1/open/...
           ↓
/v2/open/...
```

La récupération de jetons utilisée par les bibliothèques Home Assistant est en revanche la suivante :

```text
/v1/iot/secure/getToken
```

Et la connexion locale de la PortaSplit ne passe ensuite plus du tout par une telle URL cloud, mais directement par le réseau domestique.

Assimiler les trois interfaces uniquement sur la base du numéro de version « V1 » ne serait donc pas techniquement justifié.

## Existe-t-il déjà une intégration Home Assistant entièrement basée sur le cloud ?

Avec [`Midea Auto Cloud`](https://github.com/sususweet/midea_auto_cloud), il existe désormais une intégration communautaire qui contrôle les appareils Midea via le cloud au lieu de passer directement par le réseau local.

Cela ne prouve toutefois pas non plus que l’API partenaire officielle V2 ait déjà remplacé le contrôle local. Le code source actuel de `Midea Auto Cloud` utilise notamment :

```text
/v1/appliance/transparent/send
/mjl/v1/device/status/lua/get
/mjl/v1/device/lua/control
```

Ces points de terminaison sont consultables dans le [code cloud actuel de `core/cloud.py`](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py).

L’intégration reproduit donc des fonctions d’applications privées ou du cloud grand public. Elle n’utilise pas simplement l’interface partenaire documentée `/v2/open/...`.

Une alternative basée sur le cloud existe donc déjà. Elle implique toutefois aussi les dépendances habituelles d’une intégration cloud : accès à Internet, compte utilisateur fonctionnel, serveurs Midea disponibles et points de terminaison privés toujours compatibles.

## Que cela signifie-t-il concrètement pour les propriétaires de PortaSplit ?

### Contrôle local déjà configuré

Pour une PortaSplit déjà configurée, la situation est relativement sereine. `Midea Smart AC` stocke le jeton et la clé localement après la configuration et n’a, selon sa propre [documentation cloud](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage), plus besoin d’une connexion cloud pour le contrôle ultérieur.

Un arrêt de la seule récupération des jetons ne mettrait donc pas automatiquement fin à la connexion locale existante.

### Nouvelle configuration ou restauration

Le risque est plus important dans les cas suivants :

- une nouvelle installation de Home Assistant
- le passage à une autre intégration
- une sauvegarde perdue ou endommagée
- le remplacement du module Wi-Fi
- des modifications de l’association de l’appareil
- un nouvel appairage, si les informations d’accès à l’appareil changent à cette occasion

Dans de tels cas, l’intégration doit obtenir à nouveau le jeton et la clé, ou l’utilisateur doit les indiquer manuellement. Le fait que `Midea Smart AC` prenne en charge une configuration manuelle est décrit dans sa [documentation de configuration](https://github.com/mill1000/midea-ac-py#manual-configuration).

Il n’est pas officiellement documenté qu’une réinitialisation d’usine ou un nouvel appairage génère nécessairement de nouvelles informations d’accès pour chaque PortaSplit ; cela ne devrait donc pas être affirmé de manière générale.

### Un véritable arrêt du contrôle LAN

Pour qu’une PortaSplit déjà configurée cesse d’accepter ses informations d’accès stockées localement, le comportement de l’appareil ou du module Wi-Fi devrait également changer, par exemple via un nouveau firmware ou une procédure d’authentification modifiée.

Un simple arrêt du point de terminaison cloud `/v1/iot/secure/getToken` ne supprime pas automatiquement les informations d’accès déjà présentes dans l’appareil et dans Home Assistant. Cela découle de la séparation entre la récupération cloud unique et le contrôle LAN ultérieur, documentée par [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

Une telle modification future de l’appareil est techniquement possible. Je n’ai toutefois pas trouvé, dans les documents Midea accessibles au public, d’annonce concrète ou de date d’arrêt concernant spécifiquement la PortaSplit.

## Ce que je continuerais à recommander

Malgré ces conclusions nuancées, une sauvegarde reste judicieuse.

Pour les appareils V3, `Midea AC LAN` recommande explicitement de sauvegarder la configuration JSON générée en dehors de HAOS. La recommandation actuelle figure directement dans le [README du projet](https://github.com/wuwentao/midea_ac_lan#1-important-notice).

Les règles suivantes s’appliquent :

- Traiter le jeton et la clé comme des mots de passe.
- Ne pas téléverser le fichier JSON dans un dépôt Git public.
- Ne pas publier de journaux de débogage non masqués.
- Chiffrer la sauvegarde.
- Créer également une sauvegarde complète de Home Assistant.
- Vérifier le fonctionnement actuel avant les mises à jour du firmware et des intégrations.
- Tester à nouveau le contrôle local après les mises à jour.

Une sauvegarde constitue une protection raisonnable contre les changements dans le cloud, les problèmes d’intégration et les erreurs personnelles. Elle n’indique toutefois pas qu’un arrêt est imminent. La procédure pour configurer proprement une PortaSplit et la sécuriser sur le réseau domestique est décrite dans la [partie pratique consacrée à la configuration](/blog/midea-portasplit-home-assistant-einrichten).

## Évaluation sur la base des preuves disponibles

L’avertissement de `Midea AC LAN` doit être pris au sérieux, mais correctement mis en contexte.

Il documente un risque plausible à long terme : Midea pourrait considérer les jetons locaux sans expiration comme un problème de sécurité, restreindre davantage l’obtention de tels jetons ou lier plus fortement les futurs appareils au cloud.

En revanche, aucun arrêt officiellement annoncé et daté du contrôle local de la PortaSplit n’est établi.

L’état technique actuel montre même le contraire d’un arrêt déjà réalisé : en juin 2026, le point de terminaison de jetons V1 toujours utilisé a fourni des informations d’accès valides après que la requête a été adaptée au format de l’application officielle SmartHome. Le correctif correspondant fait aujourd’hui partie de la bibliothèque utilisée par `Midea AC LAN`.

L’API officielle Midea Cloud-to-Cloud V2 existe également. Mais il s’agit d’une ancienne interface partenaire à accès restreint, et non automatiquement de la successeure du protocole local de la PortaSplit.

La conclusion sobre est donc la suivante :

> Créez une sauvegarde, surveillez les intégrations et gardez à l’esprit les dépendances au cloud, mais n’écartez pas prématurément le contrôle local de la PortaSplit sur la base d’une hypothèse d’arrêt non confirmée.

## Sources

1.  [Midea AC LAN : README actuel et avertissement d’arrêt](https://github.com/wuwentao/midea_ac_lan#1-important-notice) : libellé de l’avertissement, recommandation de sauvegarde et distinction entre les anciens appareils V2 et les appareils V3 plus récents.

2.  [Midea AC LAN PR #578 du 19 mai 2025](https://github.com/wuwentao/midea_ac_lan/pull/578) : introduction de l’avertissement relatif à l’arrêt progressif des services de jetons et à la migration alléguée vers une API V2 basée sur le cloud.

3.  [Midea AC LAN PR #639](https://github.com/wuwentao/midea_ac_lan/pull/639) : modification de la source de jetons documentée vers NetHome Plus.

4.  [midea-msmart Issue #201](https://github.com/mill1000/midea-msmart/issues/201) : discussion sur la récupération défaillante des jetons SmartHome et l’utilisation temporaire de NetHome Plus.

5.  [Commentaire du mainteneur de Midea AC LAN concernant la migration V2 supposée](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457) : qualifie explicitement l’affirmation relative au nouveau cloud V2 de sa propre compréhension.

6.  [Réponse du mainteneur de midea-msmart](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109) : décrit l’existence d’une nouvelle API V2 comme une supposition et souligne les possibilités limitées de rétro-ingénierie.

7.  [midea-local PR #470 du 15 juin 2026](https://github.com/midea-lan/midea-local/pull/470) : analyse de l’erreur 3004, capture de la requête de l’application officielle, ajout de `applianceCodes` et test réussi avec quatre climatiseurs V3.

8.  [Commit immuable du correctif SmartHome-getToken](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5) : différence de code exacte du correctif intégré.

9.  [Code cloud midea-local actuel](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) : point de terminaison `/v1/iot/secure/getToken` toujours utilisé et champ de requête actuel `applianceCodes`.

10.  [Manifeste actuel de Midea AC LAN](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json) : version utilisée de `midea-local` et classification comme intégration push locale.

11.  [Midea Smart AC](https://github.com/mill1000/midea-ac-py) : documentation du contrôle local, de la récupération cloud unique pour les appareils V3 et de la configuration manuelle avec jeton et clé.

12.  [Midea AC LAN Issue #607 sur la PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/607) : exemple concret de PortaSplit avec le type d’appareil `0xAC`, le modèle `00000Q1D`, la version de protocole 3 et une configuration réussie via NetHome Plus.

13.  [API officielle Midea Cloud-to-Cloud V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html) : OAuth2, ID client, secret client, jetons d’accès et de rafraîchissement, méthode de signature et points de terminaison `/v2/open/...`.

14.  [API officielle Midea Cloud-to-Cloud V1](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html) : indication officielle selon laquelle l’ancienne interface partenaire `/v1/open/...` n’est plus recommandée et pourrait être arrêtée à l’avenir.

15.  [Midea Auto Cloud](https://github.com/sususweet/midea_auto_cloud) et [code cloud actuel](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py) : intégration communautaire pour un contrôle entièrement via le cloud et points de terminaison privés V1 des applications effectivement utilisés.
