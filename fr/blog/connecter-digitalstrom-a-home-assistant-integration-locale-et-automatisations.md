---
title: "Connecter digitalSTROM à Home Assistant : intégration locale et automatisations"
navTitle: "digitalSTROM et HA"
description: "Une intégration Home Assistant locale pour le serveur digitalSTROM, avec éclairage sur détection de mouvement, mode douche, musique par 4 pressions et départ automatique via FRITZ!Box. Avec les problèmes rencontrés en pratique."
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
  - midea-portasplit-home-assistant
slug: "connecter-digitalstrom-a-home-assistant-integration-locale-et-automatisations"
translationId: "article-271967fa6d61231d"
translationOf: digitalstrom-home-assistant
translationSourceHash: 162e47131f3dd2ebe7da91e342e930e442e8b0130eb234edbce882409781e07b
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T09:22:41.388Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/connecter-digitalstrom-a-home-assistant-integration-locale-et-automatisations
---

Dans les installations digitalSTROM, les lumières, stores et autres consommateurs sont raccordés à des bornes dans le tableau électrique, commandés par des boutons-poussoirs et un serveur digitalSTROM (dSS20). Lorsque d’autres systèmes tels que Philips Hue, Sonos et une FRITZ!Box s’y ajoutent, il est logique de tout réunir localement dans Home Assistant, sans compte cloud et sans enregistrer le mot de passe du dSS dans Home Assistant. J’ai donc écrit une petite intégration et l’ai publiée, avec les automatisations correspondantes, sous licence MIT sous [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local).

En bref : le dSS propose une API JSON utilisable avec des événements. En l’utilisant avec précaution, il est possible de transmettre sans délai à Home Assistant les boutons-poussoirs, scènes et activités Départ et Arrivée, puis de déclencher des automatisations que digitalSTROM ne connaît pas seul.

## Comment fonctionne l’intégration

Le dSS met son API à disposition sur le port 8080 via HTTPS, avec un certificat auto-signé. Pour les programmes, digitalSTROM prévoit des jetons d’application : le programme demande un jeton, l’utilisateur l’autorise dans le Configurator sous Système > Autorisation d’accès, puis le programme se connecte avec celui-ci. Le mot de passe du dSS reste chez l’utilisateur.

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
| `-s` | pas d’affichage de progression |
| `-k` | accepter le certificat auto-signé du dSS |
| `requestApplicationToken` | crée un jeton d’application qui doit être autorisé dans le Configurator |
| `loginApplication` | échange le jeton d’application autorisé contre un jeton de session |
| `event/subscribe` | s’abonne à un événement (`callScene`, `stateChange`, `buttonClick` …) sous un identifiant librement choisi |
| `event/get` avec `timeout` | attend jusqu’à 30 secondes de nouveaux événements (long polling) |

</details>

L’intégration maintient en permanence une telle connexion de long polling ouverte. Chaque scène déclenchée par un bouton-poussoir, l’application ou le Configurator arrive sous forme d’événement. Les états tels que « lumière allumée dans la pièce » ou la présence se trouvent dans le cache du dSS sous `/usr/states` et peuvent être interrogés sans solliciter les bornes.

Les requêtes directes des valeurs de sortie (`device/getOutputValue`) passent en revanche par le bus dS485 jusqu’à la borne et prennent une demi-seconde à une seconde par valeur. De nombreuses requêtes de ce type à intervalles rapprochés peuvent surcharger les compteurs. L’intégration lit donc directement uniquement les positions des stores et les sorties Joker, toutes les 15 minutes et environ une minute après un déplacement.

| Plateforme | Contenu |
|---|---|
| `light` | une lumière par pièce, commandée par des scènes de pièce comme les boutons-poussoirs ; luminosité pour les bornes à variateur, uniquement marche/arrêt pour les bornes commutées |
| `cover` | stores avec montée, descente, arrêt et position |
| `scene` | ambiances nommées dans le Configurator |
| `button` | Départ (scène 72) et Arrivée (scène 71) |
| `binary_sensor` | présence, alarme vent, détecteurs de mouvement, sorties Joker |
| `sensor` | consommation totale |

En outre, l’intégration signale chaque scène de pièce ainsi que Départ et Arrivée comme événement `digitalstrom_local_event` à Home Assistant. Il est ainsi possible d’attribuer librement des fonctions à 2 ou 4 pressions sur un interrupteur d’éclairage ordinaire.

## Automatisations sous forme de blueprints

Les automatisations suivantes sont incluses dans le dépôt sous forme de blueprints et peuvent être importées dans Home Assistant via le bouton d’importation du README.

### Éclairage sur mouvement qui n’éteint pas une lumière allumée manuellement

En cas de mouvement, la lumière s’allume puis s’éteint après une durée réglable sans mouvement. Cela peut être limité en option à un seuil de luminosité, à une plage horaire ou à un éclairage tamisé la nuit. Si quelqu’un allume la lumière au bouton-poussoir, elle reste allumée. Un interrupteur auxiliaire (`input_boolean`) mémorise si l’automatisation a allumé la lumière ; elle ne l’éteint à nouveau que dans ce cas.

Un détail n’apparaît qu’en fonctionnement : le déclencheur « détecteur calme depuis 2 minutes » est un compteur que Home Assistant réinitialise à chaque redémarrage. Si le détecteur a détecté un mouvement pour la dernière fois avant le redémarrage, aucun nouveau passage à « calme » ne se produit et la lumière reste allumée. Le blueprint vérifie donc également chaque minute si une lumière qu’il a allumée est toujours allumée alors que le détecteur est calme depuis suffisamment longtemps.

### Éteindre la lumière en quittant la pièce

Le délai d’extinction de l’éclairage sur mouvement est un compromis : trop court, la lumière s’éteint alors que quelqu’un reste immobile dans la pièce ; trop long, elle continue de brûler pendant des minutes après le départ. Un autre blueprint utilise donc un second détecteur à l’extérieur de la pièce, généralement dans le couloir. Si celui-ci détecte un mouvement, que le détecteur dans la pièce a lui aussi détecté un mouvement peu avant (dans les 30 secondes) et qu’il reste ensuite calme dans la pièce, la lumière de la pièce s’éteint immédiatement. Avec des détecteurs Hue, qui signalent « calme » environ 10 secondes après le dernier mouvement, cela représente environ 10 à 15 secondes après la sortie.

Afin que personne ne se retrouve dans le noir, la règle ne s’applique que sous certaines conditions : la lumière doit avoir été allumée par l’automatisation de mouvement, un mode douche ne doit pas être actif, et il ne doit y avoir au maximum qu’une personne à la maison. Le nombre de personnes est déterminé à partir des téléphones que Home Assistant connaît comme personnes. La condition de 30 secondes protège en outre le cas où quelqu’un reste immobile longtemps dans la pièce tandis qu’une autre personne passe dans le couloir.

### Mode douche par 2 pressions

Dans la douche, un détecteur de mouvement ne détecte généralement personne et, après la durée définie, l’obscurité s’installe. 2 pressions sur le bouton-poussoir de la salle de bains déclenchent l’ambiance 2 (scène 17) dans digitalSTROM. L’automatisation reconnaît cette scène dans la pièce et active un mode douche qui bloque l’extinction. Les mouvements durant les 2 premières minutes sont ignorés (se déshabiller, entrer dans la douche). Le premier mouvement suivant met fin au mode douche ; le délai d’extinction normal s’applique alors à nouveau. Après une durée maximale réglable (20 minutes par défaut), il se termine de lui-même.

### 4 pressions démarrent Sonos

Lorsque l’on appuie plusieurs fois rapidement sur un bouton-poussoir, digitalSTROM fait défiler les ambiances dans l’ordre : scène 5, 17, 18, puis 19 à la quatrième pression. Si la scène 19 est réglée dans le Configurator sur « ne pas modifier la sortie » pour toutes les lampes, elle est libre pour d’autres usages. Un blueprint réagit à cette scène, éteint la lumière de la pièce et démarre le haut-parleur Sonos de la pièce ou, dans les pièces sans haut-parleur, plusieurs haut-parleurs ensemble.

Le script correspondant tient compte de trois points : si un haut-parleur est déjà en lecture, la nouvelle pièce est ajoutée à son groupe afin que la lecture reste synchronisée. S’il n’est pas possible de reprendre une lecture, une station de radio du répertoire Radio Browser est diffusée. Enfin, le même volume de démarrage est défini avant chaque lancement ; autrement, une pièce diffuse faiblement et l’autre aussi fort que lors de la dernière écoute de musique.

### Départ et Arrivée via FRITZ!Box

L’intégration FRITZ!Box Tools indique si un téléphone est connecté au Wi-Fi ; elle évalue la liste des appareils de la box et couvre ainsi simultanément le réseau 2,4 GHz, le réseau 5 GHz et le LAN. S’il n’y a plus personne à la maison et que Départ n’a pas été activé, un blueprint déclenche Départ. Au retour à la maison, Arrivée est déclenchée et, s’il fait sombre, une lumière d’accueil s’allume.

En complément, une automatisation peut éteindre les lampes d’autres systèmes (par exemple Hue) et mettre Sonos en pause lors du départ. Le dSS éteint lui-même les lampes digitalSTROM.

## Plan au sol comme tableau de bord

Une présentation claire dans Home Assistant consiste en un plan au sol utilisant la carte intégrée `picture-elements`. Pour chaque pièce, un SVG transparent est superposé au plan et remplacé par une version légèrement jaune lorsque la lumière est allumée ; toucher la pièce allume ou éteint la lumière. Les lampes, haut-parleurs, stores et détecteurs sont représentés par des symboles à leur emplacement. Les instructions et un exemple de configuration se trouvent dans le dépôt sous `docs/floor-plan-dashboard.md`.

## Problèmes rencontrés en pratique

| Problème | Cause | Solution |
|---|---|---|
| Bouton Départ sans effet | dSS non connecté au réseau, les activités sur plusieurs circuits électriques passent par le serveur | connexion réseau du dSS rétablie |
| Les stores ne réagissent pas | l’alarme vent était définie sur « active », bien qu’aucun capteur de vent ne soit présent | scène 87 (« pas de vent ») déclenchée pour toutes les pièces |
| Les stores ne montent pas lors du départ | scène 72 réglée sur « ne pas modifier la sortie » (`dontCare`) | `device/setSceneMode` avec `dontCare=0`; la valeur `false` a été acceptée, mais ignorée |
| La lumière ne peut pas être variée | tube fluorescent sur une borne commutée (mode de sortie 35) | l’intégration reconnaît les bornes commutées et ne propose que marche/arrêt |
| La lumière reste allumée après un redémarrage | le compteur « calme depuis X minutes » est perdu au redémarrage | vérification supplémentaire chaque minute |
| Le téléphone est considéré absent | l’iPhone utilise une adresse Wi-Fi privée changeante et se déconnecte brièvement du Wi-Fi en veille | adresse Wi-Fi privée réglée sur « Fixe », délai de grâce de 10 minutes |
| DECT émet en permanence | « DECT Eco » n’est pas disponible dès qu’un appareil FRITZ! Smart Home est enregistré | renoncer aux prises DECT si DECT Eco est souhaité |

Une recherche d’appareils sur le pont Hue ajoute tous les appareils qui se trouvent alors en mode d’appairage. Vérifiez ensuite dans la liste des appareils que seules vos propres lampes ont été ajoutées.

Les tables de scènes des bornes peuvent être lues et modifiées via l’API, par exemple avec `device/getSceneMode` et `device/saveScene`. Chacun de ces appels passe par le bus. Avant toute modification, sauvegardez les anciennes valeurs, par exemple dans un fichier CSV, afin de pouvoir les restaurer si nécessaire.

## Historique des modifications

- **25 octobre 2026 :** lien des Sources mis à jour vers la nouvelle adresse de la base de connaissances FRITZ!.

## Sources

1.  [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local): intégration et blueprints de cet article, licence MIT.

2.  [digitalSTROM : manuel d’utilisation et de réglage](https://www.digitalstrom.com/wp-content/uploads/2021/08/AHB_DE_A1121D001V013_neu.pdf): activité Départ (maintenir 3 secondes), réglage « ne pas modifier la sortie » au chapitre 3.5.2.

3.  [digitalSTROM : documentation des boutons-poussoirs des bornes](https://www.digitalstrom.com/wp-content/uploads/2021/08/A0818D078V001_Tastendokumentation.pdf): réglages d’usine des fonctions des boutons-poussoirs selon le type de borne.

4.  [digitalSTROM : modes d’emploi](https://www.digitalstrom.com/bedienungsanleitungen/): aperçu des manuels, documents de planification et d’installation.

5.  [Home Assistant : FRITZ!Box Tools](https://www.home-assistant.io/integrations/fritz/): détection de présence, interrupteurs Wi-Fi, options de l’intégration.

6.  [Home Assistant : carte Picture Elements](https://www.home-assistant.io/dashboards/picture-elements/): base du tableau de bord avec plan au sol.

7.  [Home Assistant : Blueprints](https://www.home-assistant.io/docs/automation/using_blueprints/): importation et utilisation des blueprints.

8.  [Base de connaissances FRITZ! : une prise FRITZ! perd sa connexion](https://fritz.com/de-lu/apps/knowledge-base/FRITZ-Smart-Energy-200/3538_FRITZ-Steckdose-verliert-haufig-die-Verbindung-zur-FRITZ-Box/): indications sur la portée DECT et la puissance d’émission pour les prises Smart Home.

9.  [Radio Browser](https://www.radio-browser.info/): répertoire libre de flux radio, intégré à Home Assistant comme source multimédia.
