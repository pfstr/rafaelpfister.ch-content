---
title: "Connecter digitalSTROM à Home Assistant : intégration locale et automatisations"
navTitle: "digitalSTROM et HA"
description: "Une intégration Home Assistant locale pour le serveur digitalSTROM, avec éclairage sur mouvement, mode douche, musique par 4× pressions et départ automatique via FRITZ!Box. Avec les problèmes rencontrés en pratique."
date: "2026-10-06"
kategorie: "Home Assistant et IoT"
timeToRead: "13 min de lecture"
themen:
  - smart-home-iot
produkte:
  - "home-assistant"
protokolle:
  - "apis"
  - "troubleshooting"
related:
  - midea-portasplit-home-assistant-einrichten
slug: "connecter-digitalstrom-a-home-assistant-integration-locale-et-automatisations"
translationId: "article-271967fa6d61231d"
translationOf: digitalstrom-home-assistant
url: https://rafaelpfister.ch/fr/blog/connecter-digitalstrom-a-home-assistant-integration-locale-et-automatisations
translationSourceHash: 3eb347c3b61cdbf39b6aad2c7b4e99ca0b364c68a4c632de652726ff9bc144d0
translationModel: gpt-5.6-terra
translatedAt: 2026-10-07T10:23:56.825Z
translationReview: automatic
---

Dans les installations digitalSTROM, les luminaires, stores et autres consommateurs sont raccordés à des bornes dans le tableau électrique, commandés par des boutons-poussoirs et un serveur digitalSTROM (dSS20). Lorsque d’autres systèmes tels que Philips Hue, Sonos et une FRITZ!Box s’y ajoutent, il est naturel de tout réunir localement dans Home Assistant, sans compte cloud et sans enregistrer le mot de passe du dSS dans Home Assistant. J’ai pour cela développé une petite intégration et l’ai publiée, avec les automatisations correspondantes, sous [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local) sous licence MIT.

En bref : le dSS propose une API JSON utilisable avec des événements. En l’utilisant avec modération, les boutons-poussoirs, scènes et activités Départ et Arrivée sont transmis sans délai à Home Assistant et permettent de déclencher des automatisations que digitalSTROM ne connaît pas seul.

## Fonctionnement de l’intégration

Le dSS met son API à disposition sur le port 8080 via HTTPS, avec un certificat autosigné. Pour les programmes, digitalSTROM prévoit des jetons d’application : le programme demande un jeton, l’utilisateur l’autorise dans le Configurator sous Système > Autorisation d’accès, puis le programme s’y connecte. Le mot de passe du dSS reste chez l’utilisateur.

```bash
DSS=https://dss.local:8080/json
curl -sk "$DSS/system/requestApplicationToken?applicationName=Home%20Assistant"
curl -sk "$DSS/system/loginApplication?loginToken=<APP-TOKEN>"
curl -sk "$DSS/event/subscribe?name=callScene&subscriptionID=42&token=<SESSION>"
curl -sk "$DSS/event/get?subscriptionID=42&timeout=30000&token=<SESSION>"
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-s` | aucune indication de progression |
| `-k` | accepter le certificat autosigné du dSS |
| `requestApplicationToken` | crée un jeton d’application qui doit être autorisé dans le Configurator |
| `loginApplication` | échange le jeton d’application autorisé contre un jeton de session |
| `event/subscribe` | s’abonne à un événement (`callScene`, `stateChange`, `buttonClick` …) sous un identifiant librement choisi |
| `event/get` avec `timeout` | attend jusqu’à 30 secondes de nouveaux événements (long polling) |

</details>

L’intégration maintient en permanence une telle connexion de long polling ouverte. Chaque scène déclenchée par un bouton-poussoir, l’application ou le Configurator arrive comme événement. Les états tels que « lumière allumée dans la pièce » ou la présence se trouvent dans le cache du dSS sous `/usr/states` et peuvent être interrogés sans solliciter les bornes.

Les interrogations directes des valeurs de sortie (`device/getOutputValue`) passent en revanche par le bus dS485 jusqu’à la borne et prennent une demi-seconde à une seconde par valeur. Un grand nombre de telles interrogations à de courts intervalles peut surcharger les compteurs. L’intégration ne lit donc directement que les positions des stores et les sorties Joker, toutes les 15 minutes et environ une minute après un déplacement.

| Plateforme | Contenu |
|---|---|
| `light` | une lumière par pièce, commandée par des scènes de pièce comme les boutons-poussoirs ; luminosité pour les bornes à variateur, uniquement marche/arrêt pour les bornes commutées |
| `cover` | stores avec montée, descente, arrêt et position |
| `scene` | ambiances nommées dans le Configurator |
| `button` | Départ (scène 72) et Arrivée (scène 71) |
| `binary_sensor` | présence, alarme vent, détecteurs de mouvement, sorties Joker |
| `sensor` | consommation totale |

L’intégration signale en outre chaque scène de pièce ainsi que Départ et Arrivée comme événement `digitalstrom_local_event` à Home Assistant. Cela permet d’attribuer librement une fonction à 2× ou 4× pressions sur un interrupteur d’éclairage ordinaire.

## Automatisations sous forme de Blueprints

Les automatisations suivantes sont incluses dans le dépôt sous forme de Blueprints et peuvent être importées dans Home Assistant via le bouton d’importation du README.

### Éclairage sur mouvement qui n’éteint pas une lumière allumée manuellement

En cas de mouvement, la lumière s’allume, puis s’éteint après une durée configurable sans mouvement. En option, cela ne s’applique que sous un seuil de luminosité, dans une plage horaire ou la nuit avec variation. Si quelqu’un allume la lumière au bouton-poussoir, elle reste allumée. Un interrupteur auxiliaire (`input_boolean`) mémorise à cette fin si l’automatisation a allumé la lumière ; elle ne l’éteint que dans ce cas.

Un détail ne devient apparent qu’en fonctionnement : le déclencheur « détecteur calme depuis 2 minutes » est un compteur que Home Assistant réinitialise à chaque redémarrage. Si le détecteur a détecté un mouvement pour la dernière fois avant le redémarrage, aucun nouveau passage à « calme » ne se produit et la lumière reste allumée. Le Blueprint vérifie donc aussi chaque minute si une lumière qu’il a allumée est encore allumée alors que le détecteur est calme depuis suffisamment longtemps.

### Éteindre la lumière en quittant la pièce

Le délai d’extinction de l’éclairage sur mouvement est un compromis : trop court, la lumière s’éteint alors que quelqu’un reste immobile dans la pièce ; trop long, elle reste allumée plusieurs minutes après le départ. Un autre Blueprint utilise donc un second détecteur à l’extérieur de la pièce, généralement dans le couloir. Si celui-ci détecte un mouvement, que le détecteur dans la pièce a encore détecté un mouvement peu auparavant (dans les 30 secondes) et qu’il reste ensuite calme dans la pièce, la lumière de la pièce s’éteint immédiatement. Avec des détecteurs Hue, qui signalent « calme » environ 10 secondes après le dernier mouvement, cela représente environ 10 à 15 secondes après la sortie.

Afin que personne ne reste dans le noir, la règle ne s’applique que sous certaines conditions : la lumière doit avoir été allumée par l’automatisation de mouvement, un mode douche ne doit pas être actif et il ne doit y avoir au maximum qu’une personne à la maison. Le nombre de personnes est déterminé à partir des téléphones que Home Assistant connaît comme personnes. La condition de 30 secondes protège en outre contre le cas où une personne reste immobile dans la pièce et une autre passe dans le couloir.

### Mode douche par 2× pressions

Dans la douche, un détecteur de mouvement ne détecte généralement personne et, après le délai réglé, il fait sombre. 2× pressions sur le bouton-poussoir de la salle de bains déclenchent l’ambiance 2 (scène 17) dans digitalSTROM. L’automatisation reconnaît cette scène dans la pièce et active un mode douche qui bloque l’extinction. Les mouvements durant les 2 premières minutes sont ignorés (se déshabiller, entrer dans la douche). Le premier mouvement qui suit désactive le mode douche ; à partir de là, le délai d’extinction normal s’applique à nouveau. Après une durée maximale configurable (20 minutes par défaut), il se termine de lui-même.

### 4× pressions lancent Sonos

Lorsqu’on appuie rapidement plusieurs fois sur un bouton-poussoir, digitalSTROM fait défiler les ambiances dans l’ordre : scène 5, 17, 18 et, à la quatrième pression, 19. Si la scène 19 est réglée dans le Configurator sur « ne pas modifier la sortie » pour toutes les lampes, elle est libre pour d’autres usages. Un Blueprint réagit à cette scène, éteint l’éclairage de la pièce et démarre le haut-parleur Sonos dans la pièce ou, dans les pièces sans haut-parleur, plusieurs haut-parleurs ensemble.

Le script associé tient compte de trois points : si un haut-parleur fonctionne déjà, la nouvelle pièce est ajoutée à son groupe afin que la lecture reste synchronisée. S’il n’est pas possible de reprendre quoi que ce soit, une station de radio du répertoire Radio Browser est lue. Et le même volume de départ est réglé avant chaque démarrage ; sinon, une pièce joue doucement et l’autre aussi fort que la dernière fois que quelqu’un y a écouté de la musique.

### Départ et Arrivée via FRITZ!Box

L’intégration FRITZ!Box Tools indique si un téléphone est connecté au WLAN ; elle évalue la liste des appareils de la box et couvre ainsi simultanément les réseaux 2,4 GHz, 5 GHz et le LAN. Lorsque plus personne n’est à la maison et que Départ n’a pas été activé, un Blueprint déclenche Départ. Au retour à la maison, Arrivée suit et, s’il fait sombre, une lumière d’accueil.

En complément, une automatisation peut éteindre les lampes d’autres systèmes (par exemple Hue) et mettre Sonos en pause lors du départ. Le dSS éteint lui-même les lampes digitalSTROM.

## Plan au sol comme tableau de bord

Une représentation claire dans Home Assistant est un plan au sol utilisant la carte intégrée `picture-elements`. Pour chaque pièce, un SVG transparent est placé au-dessus du plan et remplacé par une version légèrement jaune lorsque la lumière est allumée ; une pression sur la pièce commande la lumière. Les lampes, haut-parleurs, stores et détecteurs sont représentés par des symboles à leur emplacement. Les instructions avec une configuration d’exemple se trouvent dans le dépôt sous `docs/floor-plan-dashboard.md`.

## Problèmes rencontrés en pratique

| Problème | Cause | Solution |
|---|---|---|
| Bouton Départ sans effet | dSS non connecté au réseau, les activités sur plusieurs circuits électriques passent par le serveur | connexion réseau du dSS rétablie |
| Les stores ne réagissent pas | l’alarme vent était réglée sur « active », bien qu’aucun capteur de vent ne soit présent | scène 87 (« pas de vent ») déclenchée pour toutes les pièces |
| Les stores ne montent pas au départ | scène 72 sur « ne pas modifier la sortie » (`dontCare`) | `device/setSceneMode` avec `dontCare=0`; la valeur `false` a été acceptée, mais ignorée |
| Impossible de varier la lumière | tube fluorescent sur une borne commutée (mode de sortie 35) | l’intégration reconnaît les bornes commutées et n’y propose que marche/arrêt |
| La lumière reste allumée après un redémarrage | le compteur « calme depuis X minutes » est perdu au redémarrage | vérification supplémentaire chaque minute |
| Le téléphone est considéré absent | l’iPhone utilise une adresse WLAN privée changeante et se déconnecte brièvement du WLAN en veille | adresse WLAN privée réglée sur « fixe », délai de grâce de 10 minutes |
| DECT émet en permanence | « DECT Eco » n’est pas disponible dès qu’un appareil FRITZ! Smart Home est enregistré | renoncer aux prises DECT si DECT Eco est souhaité |

Une recherche d’appareils sur le pont Hue ajoute tous les appareils qui se trouvent alors en mode d’appairage. Vérifiez ensuite dans la liste des appareils que seules vos propres lampes ont été ajoutées.

Les tables de scènes des bornes peuvent être lues et modifiées via l’API, par exemple avec `device/getSceneMode` et `device/saveScene`. Chacun de ces appels passe par le bus. Avant toute modification, sauvegardez les anciennes valeurs, par exemple dans un fichier CSV, afin de pouvoir les restaurer si nécessaire.

## Sources

1.  [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local): intégration et Blueprints de cet article, licence MIT.

2.  [digitalSTROM : manuel d’utilisation et de réglage](https://www.digitalstrom.com/wp-content/uploads/2021/08/AHB_DE_A1121D001V013_neu.pdf): activité Départ (maintenir 3 secondes), réglage « ne pas modifier la sortie » au chapitre 3.5.2.

3.  [digitalSTROM : documentation des boutons-poussoirs des bornes](https://www.digitalstrom.com/wp-content/uploads/2021/08/A0818D078V001_Tastendokumentation.pdf): réglages d’usine des fonctions des boutons-poussoirs par type de borne.

4.  [digitalSTROM : modes d’emploi](https://www.digitalstrom.com/bedienungsanleitungen/): aperçu des manuels, documents de planification et d’installation.

5.  [Home Assistant : FRITZ!Box Tools](https://www.home-assistant.io/integrations/fritz/): détection de présence, interrupteurs WLAN, options de l’intégration.

6.  [Home Assistant : carte Picture Elements](https://www.home-assistant.io/dashboards/picture-elements/): base du tableau de bord avec plan au sol.

7.  [Home Assistant : Blueprints](https://www.home-assistant.io/docs/automation/using_blueprints/): importation et utilisation de Blueprints.

8.  [Base de connaissances FRITZ! : la prise FRITZ! perd la connexion](https://lu.fritz.com/service/wissensdatenbank/dok/FRITZ-Smart-Energy-200/3538_FRITZ-Steckdose-verliert-haufig-die-Verbindung-zur-FRITZ-Box/): indications sur la portée et la puissance radio DECT des prises Smart Home.

9.  [Radio Browser](https://www.radio-browser.info/): répertoire libre de flux radio, intégré dans Home Assistant comme source multimédia.
