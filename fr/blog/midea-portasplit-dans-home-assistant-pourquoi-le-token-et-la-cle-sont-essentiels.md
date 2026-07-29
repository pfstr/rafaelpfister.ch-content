---
title: "Midea PortaSplit dans Home Assistant : pourquoi le token et la clé sont essentiels"
navTitle: "PortaSplit et token"
description: "La commande locale nécessite deux valeurs issues du cloud Midea. Voici comment obtenir le token et la clé, pourquoi leur perte est problématique et comment les propriétaires peuvent sécuriser leur installation existante."
date: "2026-07-24"
kategorie: "Home Assistant et IoT"
timeToRead: "9 min de lecture"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant-einrichten
  - serverloser-newsletter-cloudflare-workers-d1
image: "../images/midea-portasplit-home-assistant/home-assistant-dashboard-portasplit.png"
slug: "midea-portasplit-dans-home-assistant-pourquoi-le-token-et-la-cle-sont-essentiels"
translationOf: "midea-portasplit-home-assistant"
url: "https://rafaelpfister.ch/fr/blog/midea-portasplit-dans-home-assistant-pourquoi-le-token-et-la-cle-sont-essentiels"
translationId: article-a02e26cce22063f1
translationReview: automatic
translationSourceHash: a02265cf4b8fde907361c3551326fd3283c83d660cf9fdfb40451a9e78ca690b
translatedAt: 2026-07-29T12:29:38.937Z
---

<aside class="article-update">
  <p class="article-update__label">Ce que les propriétaires de PortaSplit devraient faire maintenant</p>
  <p>Lors de la configuration, Home Assistant récupère le token et la clé propres à l’appareil via des interfaces cloud privées. Le projet Midea AC LAN met en garde contre d’éventuelles modifications depuis le 19 mai 2025. Aucune date précise d’arrêt par le fabricant n’est toutefois documentée. Pour les propriétaires, cela signifie :</p>
  <ol>
    <li><strong>Ne pas supprimer inutilement une configuration existante.</strong> Seule l’obtention des identifiants nécessite le cloud Midea. De futures modifications du point de terminaison privé pourraient compliquer une nouvelle configuration.</li>
    <li><strong>Sauvegarder le token, la clé et la configuration sous forme chiffrée.</strong> Si leur récupération ne fonctionne plus ultérieurement, la sauvegarde reste le moyen le plus fiable de restaurer la configuration.</li>
    <li><strong>Ne pas dissocier l’appareil sans nécessité.</strong> La réinitialisation d’usine, la suppression du compte Midea ou le remplacement d’un module Wi-Fi imposent une nouvelle obtention du token, qui pourrait échouer à l’avenir.</li>
  </ol>
  <p>Les appareils déjà configurés sont commandés localement. Les modifications de l’interface cloud concernent donc d’abord l’ajout et la restauration, et non chaque commande en cours. Les étapes concrètes sont décrites dans <a href="/blog/midea-portasplit-home-assistant-einrichten">l’article pratique sur l’intégration et la sécurisation</a>.</p>
</aside>

![Exemple de tableau de bord Home Assistant d’une Midea PortaSplit avec température ambiante et de consigne, humidité de l’air, puissance absorbée, consommation d’énergie et temps de fonctionnement du compresseur au cours des dernières 24 heures.](../images/midea-portasplit-home-assistant/home-assistant-dashboard-portasplit.png)

La commande locale de la Midea PortaSplit repose sur deux valeurs propres à l’appareil : le token et la clé. Lors de la configuration, l’intégration Home Assistant récupère les deux via un point de terminaison cloud Midea privé. Elle envoie ensuite les commandes de contrôle directement sur le réseau local.

Le projet Midea AC LAN met en garde contre d’éventuelles modifications de ces interfaces cloud. Des analyses plus récentes montrent toutefois qu’il n’est pas possible d’en déduire une feuille de route confirmée du fabricant ni une date précise d’arrêt. Cet article explique cette dépendance technique ; [l’analyse détaillée de l’API](/blog/midea-v2-cloud-api-portasplit-home-assistant) replace les différentes appellations « V2 » et la situation actuelle dans leur contexte.

## La question du token en détail

### Pourquoi Home Assistant pouvait-il jusqu’ici obtenir le token ?

Le point intéressant est le suivant : la communauté n’a jamais calculé le token. Elle a plutôt analysé le trafic réseau de l’application officielle et constaté que celle-ci ne génère pas elle-même le token, mais le récupère depuis le cloud :

```text
App
   ↓
Midea Cloud
   ↓
Cloud liefert Token
   ↓
App verwendet Token lokal
```

L’intégration Home Assistant a réimplémenté précisément cet appel cloud. Elle se connecte au cloud avec les mêmes points de terminaison et le même déroulement que l’application, et obtient ainsi le même token et la même clé. Le véritable fondement n’est donc pas un calcul ingénieux, mais une récupération reproduite. Si le point de terminaison disparaît, l’obtention disparaît également.

### Peut-on extraire le token de l’application officielle ?

En théorie, oui. L’application doit connaître le token à un moment donné, sans quoi elle ne pourrait pas communiquer localement avec l’appareil. Les pistes envisageables sont notamment :

- l’ingénierie inverse de l’application,
- l’interception du trafic réseau, s’il n’est pas protégé en plus,
- l’instrumentation de l’application à l’exécution, par exemple avec Frida ou Objection,
- le hooking des fonctions qui traitent le token.

C’est précisément ce que vise l’affirmation du développeur de Midea AC LAN selon laquelle la conception actuelle constitue, du point de vue de Midea, un problème de sécurité : un secret durable qui peut être extrait avec un effort raisonnable d’une application largement distribuée est difficile à contrôler. Pour chaque utilisateur, ces méthodes restent toutefois complexes et ne remplacent pas la récupération pratique depuis le cloud.

### Peut-on obtenir le token directement de l’appareil ?

Ce serait la solution la plus élégante. Si l’appareil échangeait une clé publique lors du premier appairage local ou utilisait un code d’appairage unique via Bluetooth, le cloud ne serait pas nécessaire. De nombreux appareils IoT modernes fonctionnent précisément ainsi.

Midea a toutefois conçu le protocole LAN d’origine différemment : l’appareil n’accepte les commandes locales qu’avec les identifiants appropriés liés au cloud. Il n’existe aucun mécanisme d’appairage local documenté qui fournirait le token sans passer par le cloud. Le cloud n’est donc pas seulement une commodité, mais la seule voie prévue par l’architecture pour obtenir le token.

### La communauté pourrait-elle contourner des modifications du point de terminaison du token ?

Cela ne serait possible que si l’une des options suivantes était trouvée :

- une nouvelle API cloud qui continue à fournir des tokens,
- une méthode d’appairage local jusqu’ici inconnue,
- une faille dans l’appareil,
- ou la publication par Midea d’une API locale officielle.

En revanche, il est très peu probable qu’il soit possible de « recalculer » simplement le token. Si cela était possible, la communauté l’aurait probablement mis en œuvre depuis longtemps et n’aurait jamais dépendu de l’API cloud. Le fait même que ce détour par le cloud ait été mis en place est l’indice le plus fort qu’il n’existe pas de solution locale plus simple.

## L’avertissement de Midea AC LAN

Le dépôt de `Midea AC LAN` contient un avis « Important Notice » placé bien en évidence. Selon le développeur, Midea a déjà fermé les API de token côté serveur dans les clouds Meiju et SmartHome. L’intégration utilise donc actuellement les interfaces de token du cloud NetHome Plus, et celles-ci devraient elles aussi être progressivement fermées. Il en résulterait que les appareils déjà configurés continueraient à fonctionner localement, mais que de nouveaux appareils ne pourraient plus être ajoutés. Le développeur va plus loin et écrit que Midea souhaiterait à long terme passer à une nouvelle API Cloud Control, rendant ainsi inutilisable l’ancienne API LAN V1.

L’avertissement a une brève histoire. Le très visible « Important Notice » a été ajouté au README le 19 mai 2025 (pull request no 578) et désignait alors le cloud SmartHome comme solution de repli pour l’ajout de nouveaux appareils. Le 14 juillet 2025 (no 639), il a été mis à jour ; depuis, il renvoie au cloud NetHome Plus, car Midea avait fermé d’autres points de terminaison. Le fond est resté inchangé dans les deux versions : les interfaces de token disparaissent progressivement, seul le cloud encore utilisable change à chaque fois.

Il faut considérer cela avec nuance. Il s’agit de l’évaluation d’un projet open source, et non d’une feuille de route contraignante de Midea, et le calendrier est inconnu. Une future mise à jour du firmware peut modifier les fonctions locales ; un token déjà enregistré peut continuer à fonctionner, mais pas nécessairement pour toujours. Une réinitialisation d’usine, le remplacement du module Wi-Fi ou un nouvel appareil peuvent nécessiter une nouvelle obtention du token.

Les trois mesures présentées dans l’encadré au début de l’article en découlent, chacune pour la raison suivante :

- **Ne pas remplacer une configuration fonctionnelle sans raison.** L’obtention du token est la seule étape qui passe obligatoirement par le cloud Midea. Les modifications du point de terminaison privé peuvent surtout affecter une nouvelle configuration ultérieure.
- **Sauvegarder les identifiants.** Home Assistant enregistre localement le token et la clé. Un système défectueux, une restauration ratée ou une intégration supprimée par inadvertance peut néanmoins rendre la commande locale inutilisable en l’absence de sauvegarde externe.
- **Ne pas dissocier l’appareil à la légère.** Il n’est pas entièrement documenté si une réinitialisation d’usine ou la suppression du compte Midea impose de nouveaux identifiants pour chaque modèle. Une sauvegarde avant de tels changements est donc impérative.

Le fonctionnement en cours n’est pour l’instant pas concerné : la commande locale utilise les valeurs déjà enregistrées et n’a plus besoin du point de terminaison du token. Un risque résiduel demeure si un futur firmware modifie le protocole local ou l’authentification. La procédure de sauvegarde du token, de la clé et de la configuration est décrite dans [l’article pratique sur la configuration](/blog/midea-portasplit-home-assistant-einrichten#backup-der-konfiguration).

## Ce que cela signifie pour la sécurité

Outre la disponibilité, l’avertissement comporte un aspect central de sécurité. Selon `Midea AC LAN`, l’ancienne architecture LAN repose sur une hypothèse problématique : la communication client était initialement considérée comme suffisamment protégée, raison pour laquelle les tokens émis par le cloud n’avaient pas de date d’expiration.

Un token sans expiration n’est pas, en soi, une vulnérabilité. Il devient problématique s’il se retrouve dans des journaux ou des sauvegardes non protégées, parvient à des tiers ou ne peut être ni révoqué ni renouvelé. Le développeur de `Midea AC LAN` suppose que Midea réagit à ces risques en modifiant les services de token et en adoptant une architecture davantage basée sur le cloud. Aucune annonce du fabricant avec un calendrier n’est toutefois établie.

La précision des termes est importante ici. L’intégration communautaire ne « pirate » pas le climatiseur. Elle implémente un protocole propriétaire reconstitué par ingénierie inverse. Le problème de sécurité vient du fait que des secrets durables peuvent être utilisés et enregistrés en dehors de l’application initialement prévue.

Pour l’exploitation sur son propre réseau, il importe surtout de savoir ce que permettent le token et la clé. Tous deux authentifient la communication locale avec l’appareil. S’ils tombent entre de mauvaises mains, un attaquant pourrait, selon le protocole et sa position sur le réseau, identifier l’appareil, s’authentifier auprès de lui, lire les informations d’état, modifier les réglages, allumer ou éteindre le climatiseur, changer de mode de fonctionnement et modifier la température de consigne. En règle générale, l’attaquant doit néanmoins pouvoir établir une connexion réseau avec l’appareil ; la seule possession du token et de la clé ne permet pas une attaque depuis l’ensemble d’Internet. Le token et la clé doivent donc être traités comme un mot de passe. La manière d’intégrer l’appareil au réseau pour que ces valeurs causent peu de dégâts même en cas d’incident est le sujet de la [deuxième partie](/blog/midea-portasplit-home-assistant-einrichten#die-portasplit-sicher-betreiben).

## Ce qui reste en pratique

La commande locale de la PortaSplit dépend entièrement du token et de la clé, qui ne peuvent actuellement être obtenus que via le cloud Midea. Ce détour fait partie de la conception du protocole : les commandes locales sont liées à des identifiants associés au cloud. Comme le point de terminaison est privé et non documenté, la disponibilité à long terme de l’intégration non officielle reste incertaine.

En pratique, cela signifie : sauvegarder les identifiants et la configuration, ne pas dissocier inutilement une configuration fonctionnelle et surveiller les modifications de l’intégration et du firmware. Les appareils déjà configurés continuent à fonctionner localement. La configuration, la sauvegarde et la protection réseau sont décrites dans [l’article pratique sur la PortaSplit](/blog/midea-portasplit-home-assistant-einrichten).

## Sources

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: intégration `Midea AC LAN` avec l’« Important Notice » (depuis le 19 mai 2025, mise à jour le 14 juillet 2025), l’explication fondée sur les tokens sans expiration et le chiffrement client reconstitué, ainsi que la description de l’obtention du token basée sur le cloud.

2.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: intégration `Midea Smart AC`: description de l’obtention du token et de la clé basée sur le cloud pour les appareils V3 et de l’enregistrement local des valeurs.

3.  [Midea SmartHome](https://www.midea.com/global/smarthome): informations du fabricant sur l’écosystème SmartHome et sur les normes de sécurité et de protection des données référencées.
