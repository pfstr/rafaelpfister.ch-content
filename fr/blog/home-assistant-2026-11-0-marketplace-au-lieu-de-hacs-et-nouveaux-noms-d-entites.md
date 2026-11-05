---
title: "Home Assistant 2026.11.0 : Marketplace au lieu de HACS et nouveaux noms d’entités"
navTitle: "Home Assistant 2026.11.0"
description: "Home Assistant 2026.11.0 intègre HACS en tant que Marketplace, recompose les noms d’entités à partir de l’appareil et de l’entité et apporte de nombreux changements incompatibles avec les versions antérieures. Avant la mise à jour, vérifiez les tableaux de bord, les modèles et la détection de présence via la FRITZ!Box."
date: "2026-11-05"
kategorie: "Home Assistant et IoT"
timeToRead: "6 min de lecture"
themen:
  - smart-home-iot
produkte:
  - "home-assistant"
protokolle:
  - "releases"
slug: "home-assistant-2026-11-0-marketplace-au-lieu-de-hacs-et-nouveaux-noms-d-entites"
translationId: article-6fe587829541b0ff
translationOf: home-assistant-2026-11-0
url: https://rafaelpfister.ch/fr/blog/home-assistant-2026-11-0-marketplace-au-lieu-de-hacs-et-nouveaux-noms-d-entites
translationSourceHash: e560e2600ff8f900817c5c323e9f6a0c0bb7be09065e7794e695e6fbe12d1f52
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:24:09.396Z
translationReview: automatic
---

# Home Assistant 2026.11.0 : Marketplace au lieu de HACS et nouveaux noms d’entités

Home Assistant 2026.11.0 est paru le 4 novembre 2026 et succède à la version 2026.10.4. Cette version intègre l’ancien Home Assistant Community Store (HACS) sous forme de Marketplace directement dans Home Assistant, modifie la création des noms d’entités et contient une longue liste de changements incompatibles avec les versions antérieures, notamment des modifications apportées aux Device Trackers de FRITZ!Box Tools et à l’authentification via des adresses non locales. Selon le billet de publication, 38 nouvelles intégrations sont ajoutées. Les notes de version ne mentionnent aucune correction de sécurité avec CVE. Cette version s’adresse à tous les exploitants de Home Assistant, particulièrement aux installations utilisant des Custom Integrations issues de HACS, comme celles employées dans les articles sur [Midea PortaSplit](/blog/midea-portasplit-home-assistant) et [digitalSTROM](/blog/digitalstrom-home-assistant).

## Les versions en bref

| Version | Date | Type | Points clés |
|---|---|---|---|
| 2026.11.0 | 04.11.2026 | Version mensuelle | HACS devient Marketplace, nouveaux noms d’entités, Device Trackers des scanners réseau toujours associés à un appareil, contrôle d’authentification pour les adresses non locales, plus de dossier `deps`, 38 nouvelles intégrations |

## Marketplace : HACS devient partie intégrante de Home Assistant

Avec la version 2026.11, HACS devient, selon le billet de publication, une composante de Home Assistant et s’appelle désormais Marketplace. Les administrateurs le trouvent sous Paramètres > Marketplace et y installent des intégrations, cartes de tableau de bord, thèmes et modèles ; aucun compte GitHub n’est nécessaire. Après l’installation d’une intégration, aucun redémarrage n’est requis, mais les mises à jour nécessitent toujours un redémarrage.

Home Assistant migre les installations HACS existantes au premier démarrage après la mise à jour. Les projets installés restent installés et continuent de recevoir des mises à jour ; les Custom Repositories et les entités de mise à jour sont repris. Selon le fabricant, aucune intervention n’est nécessaire. L’installateur HACS prend fin avec Home Assistant 2026.11, une installation HACS distincte n’est plus possible avec cette version et l’ancien HACS ne recevra plus de nouvelles versions. Signalez désormais les erreurs dans le gestionnaire d’issues de Home Assistant.

Lors du premier accès, le Marketplace affiche un avertissement indiquant que les projets communautaires ne sont ni testés, ni vérifiés, ni pris en charge par le projet Home Assistant. Vous devez confirmer cet avertissement avant toute installation ou mise à jour. Les projets hors catalogue peuvent être ajoutés via « Add from link » ; cela requiert une connexion à GitHub par Device Sign-in, qui lit uniquement des informations publiques.

Pour Midea PortaSplit, cela concerne la Custom Integration `Midea AC LAN` et la carte `apexcharts-card`, à condition que toutes deux aient été installées via HACS. Selon les informations de migration, elles restent installées et continuent de recevoir des mises à jour. Le risque lié à la chaîne d’approvisionnement des Custom Integrations, décrit dans l’article sur la [sécurisation de la PortaSplit](/blog/midea-portasplit-home-assistant-absichern), subsiste avec le Marketplace, comme le montre l’avertissement de risque du fabricant. Après la mise à jour, vérifiez sous Paramètres > Marketplace que tous les projets et Custom Repositories auparavant installés via HACS sont présents.

## Nouveaux noms d’entités

À partir de la version 2026.11, les noms d’entités sont composés du nom de l’entité et de son contexte, généralement du nom de l’appareil. Les noms attribués manuellement sont migrés selon les mêmes règles et l’ancien nom est conservé comme alias. Les cartes, badges et lignes sans nom propre affichent désormais le nom de l’appareil suivi du nom de l’entité au lieu du Friendly Name. Selon les notes de version, les ID d’entité, automatisations et scripts ne sont pas concernés.

Pour les modèles, le fabricant recommande les fonctions `entity_name()`, `device_name()` et `area_name()`; l’attribut `friendly_name` devrait ultérieurement être marqué comme obsolète. Après la mise à jour, vérifiez les libellés modifiés dans vos tableaux de bord et adaptez les modèles qui évaluent `friendly_name`.

## FRITZ!Box Tools et Device Tracker

Les trackers de scanners réseau, parmi lesquels FRITZ!Box Tools, UniFi Network, NETGEAR et Nmap Tracker sont explicitement cités, sont désormais toujours associés à un appareil. Les entités de tracker renommées reçoivent le nom de l’appareil au début de leur nom. Les nouveaux trackers de ces intégrations sont toujours créés désactivés ; auparavant, ils étaient actifs si une autre intégration connaissait déjà l’adresse MAC de l’appareil. Les trackers existants conservent leur état, activé ou désactivé.

Pour les automatisations de présence via la FRITZ!Box, telles que décrites dans l’[article digitalSTROM](/blog/digitalstrom-home-assistant) pour le départ automatique, cela signifie que les trackers existants continuent de fonctionner. En revanche, vous devez d’abord activer le tracker d’un smartphone nouvellement ajouté avant qu’il ne fournisse un état aux automatisations.

Le changelog répertorie également plusieurs corrections pour FRITZ!Box Tools, notamment pour la création des interrupteurs Wi-Fi, le filtrage des hôtes FRITZ!Box sans adresse IP des Device Trackers, une boucle de rechargement pour les appareils sans données de trafic et un message d’erreur plus compréhensible en cas d’identifiants invalides dans l’assistant de configuration. En cas d’autorisations manquantes, l’intégration FRITZ!Box Call Monitor signale désormais une erreur de configuration.

## Midea et digitalSTROM

Les améliorations apportées à Midea dans le billet de publication concernent l’intégration officielle `Midea`, qui fait partie de Core depuis Home Assistant 2026.8. La Custom Integration `Midea AC LAN` (domaine `midea_ac_lan`) en est distincte. Sont nouveaux : des commandes et un capteur de progression pour les lave-linge et sèche-linge, des capteurs de porte pour les réfrigérateurs, des entités de chauffe-eau, un interrupteur marche/arrêt pour les purificateurs d’air et un interrupteur pour le signal sonore des déshumidificateurs. Pour les climatiseurs, le changelog mentionne un bouton `reset_filter`, qui permet de marquer le filtre comme nettoyé. L’intégration a atteint le niveau de qualité Platinum. Selon la page de l’intégration, les commandes et les requêtes d’état fonctionnent localement ; le cloud n’est requis que pour le token et la clé des appareils V3.

Le changelog de 2026.11.0 ne contient aucune entrée concernant digitalSTROM. L’intégration décrite dans l’[article digitalSTROM](/blog/digitalstrom-home-assistant) est une Custom Integration ; si elle est intégrée via HACS comme Custom Repository, la reprise dans le Marketplace décrite ci-dessus s’applique.

## Autres changements incompatibles avec les versions antérieures

La liste complète figure dans le billet de publication. Pour l’exploitation, les points suivants sont particulièrement pertinents :

- **Auth :** En cas d’accès via une adresse IP non locale, le schéma, l’hôte et le port dans le navigateur doivent correspondre à l’adresse vue par Home Assistant, sans quoi l’authentification échoue avec « Invalid client id ». Le contrôle précédent acceptait ces adresses à cause d’une erreur dès lors qu’elles contenaient un port. Dans ce cas, ajoutez votre Reverse Proxy aux Trusted Proxies de l’intégration HTTP ou utilisez un nom de domaine.
- **Dépendances Python :** Home Assistant n’installe plus de paquets dans le dossier `deps` du répertoire de configuration. Seules les installations Core hors environnement virtuel sont concernées ; les installations OS, Supervised et Container ne le sont pas. Après la mise à jour, le dossier `deps` peut être supprimé.
- **SMTP :** Une pièce jointe illisible provoque désormais une erreur ; auparavant, le message était envoyé sans pièce jointe.
- **Telegram :** L’ancienne plateforme `notify` est supprimée. Utilisez les entités Notify du bot Telegram et l’action `notify.send_message`.
- **Z-Wave :** zwave-js-server 3.11.0 ou version ultérieure (schéma 51), l’application Z-Wave JS à partir de 1.9.0 ou Z-Wave JS UI à partir de v11.24.2 sont requis.
- **Uptime Kuma :** Les versions antérieures à 2.0.0 ne sont plus prises en charge.
- **Philips Hue :** L’effet lumineux `"None"` est supprimé ; utilisez à la place `"off"`.
- **Matter et Electra Smart :** `climate.turn_on` reprend le dernier mode actif. Pour utiliser un mode précis, employez `climate.set_hvac_mode`.
- **Intervalle de polling :** Pour Hayward Omnilogic, Hive, Keenetic NDMS2, Kraken, Plaato et Risco, un intervalle personnalisé est ignoré et l’intervalle par défaut est utilisé. Control4 interroge toujours toutes les 5 secondes ; pour un autre intervalle, désactivez le polling et appelez `homeassistant.update_entity` depuis une automatisation.

L’intégration Flipr a été supprimée, car l’entreprise qui la développait a été liquidée.

## Nouvelles fonctionnalités

Outre le Marketplace, le billet de publication mentionne notamment l’exclusion d’entités, appareils, zones, étages ou labels individuels d’une cible, le copier-coller de sections de tableau de bord, une carte de date et une carte d’alerte avec éditeur visuel. Le tableau de bord Énergie peut dériver un capteur d’énergie d’un capteur de puissance. Pour le dépannage, un filtre « Triggered by » est disponible dans l’activité, ainsi que des journaux en direct par intégration et la suppression de statistiques sur une période donnée. Les sauvegardes et médias peuvent être stockés sur des supports locaux. Sont également pris en charge les caméras Matter 1.5 avec WebRTC, ainsi que LoRaWAN (ChirpStack, The Things Network, SenseCAP) et Zigbee Green Power.

Dans la carte d’alerte, l’option `color` s’appelle désormais `alert_color`; vous ne devez adapter que les tableaux de bord que vous avez créés vous-même en YAML. La nouvelle intégration Debug console exécute du code Python au sein de Home Assistant, dispose d’un accès complet et doit donc être activée dans `configuration.yaml`; seuls les administrateurs peuvent l’utiliser.

## Procédure recommandée

La version cible est 2026.11.0. Avant la mise à jour, lisez les changements incompatibles avec les versions antérieures dans le billet de publication pour les intégrations que vous utilisez. Après la mise à jour, vérifiez dans le Marketplace que tous les anciens projets HACS ont été repris, contrôlez vos tableaux de bord et modèles pour les nouveaux noms d’entités et activez, si nécessaire, les Device Trackers nouvellement créés. En cas d’accès via un Reverse Proxy, testez l’authentification. Les avis de sécurité dans le dépôt GitHub de Home Assistant Core ne contiennent, au 5 novembre 2026, aucune entrée associée à la version 2026.11.0 ; le billet de publication ne donne aucune indication sur les problèmes connus.

## Sources

1.  [Home Assistant: 2026.11: A lot in store!](https://www.home-assistant.io/blog/2026/11/04/release-202611/): Billet de publication avec Marketplace, noms d’entités, nouvelles intégrations et changements incompatibles avec les versions antérieures.

2.  [Home Assistant: Full changelog for Home Assistant 2026.11](https://www.home-assistant.io/changelogs/core-2026.11): Changelog complet, y compris les entrées concernant FRITZ!Box Tools, FRITZ!Box Call Monitor et Midea.

3.  [GitHub: home-assistant/core Release 2026.11.0](https://github.com/home-assistant/core/releases/tag/2026.11.0): Tag de publication avec sortie le 4 novembre 2026.

4.  [Home Assistant: Midea](https://www.home-assistant.io/integrations/midea/): Documentation de l’intégration officielle Midea, introduite en 2026.8, avec contrôle local.

5.  [GitHub: Security Advisories home-assistant/core](https://github.com/home-assistant/core/security): Avis de sécurité publiés, état au 5 novembre 2026 sans entrée concernant 2026.11.0.

6.  [Midea PortaSplit dans Home Assistant : configuration et tableau de bord](/blog/midea-portasplit-home-assistant): Installation de Midea AC LAN et apexcharts-card via HACS.

7.  [Sécuriser Midea PortaSplit dans Home Assistant](/blog/midea-portasplit-home-assistant-absichern): Risque lié à la chaîne d’approvisionnement des Custom Integrations.

8.  [Connecter digitalSTROM à Home Assistant](/blog/digitalstrom-home-assistant): Intégration digitalSTROM locale et présence via la FRITZ!Box.
