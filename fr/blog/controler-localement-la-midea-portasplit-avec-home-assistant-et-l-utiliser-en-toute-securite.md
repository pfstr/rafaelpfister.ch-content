---
title: "Midea PortaSplit dans Home Assistant : configuration et tableau de bord"
navTitle: "Configurer PortaSplit"
description: "Pas à pas, de l’association avec MSmartHome à l’intégration Midea AC LAN, jusqu’au tableau de bord complet avec indicateurs, commandes et graphiques d’historique."
date: "2026-07-24"
kategorie: "Home Assistant et IoT"
timeToRead: "10 min de lecture"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant-absichern
  - midea-v2-cloud-api-portasplit-home-assistant
slug: "controler-localement-la-midea-portasplit-avec-home-assistant-et-l-utiliser-en-toute-securite"
translationOf: "midea-portasplit-home-assistant"
translationId: article-36e7710abe426781
translationReview: automatic
translationSourceHash: 6b0bf224030d5fca539c146523bb8de015a6bab426c4e35ab116423719d9c232
translatedAt: 2026-10-09T10:51:35.751Z
translationModel: gpt-5.6-terra
image: ../images/midea-portasplit-home-assistant/portasplit-dashboard.png
url: https://rafaelpfister.ch/fr/blog/controler-localement-la-midea-portasplit-avec-home-assistant-et-l-utiliser-en-toute-securite
---

La Midea PortaSplit peut être commandée directement sur le réseau local via Home Assistant grâce à une intégration communautaire. En sept étapes, de l’association dans l’application au tableau de bord, vous obtenez une commande locale avec indicateurs et graphiques d’historique. Le tableau de bord, les capteurs auxiliaires et le thème sont disponibles dans le dépôt <a class="gh-badge" href="https://github.com/pfstr/ha-portasplit-dashboard" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">pfstr/ha-portasplit-dashboard</span></a>.

![Tableau de bord Home Assistant de la Midea PortaSplit en mode refroidissement : indicateurs en haut, thermostat réglé sur 22 °C, historiques de la température ambiante, de la puissance absorbée, de l’énergie quotidienne, de la fréquence du compresseur, du fonctionnement du compresseur et de la vitesse du ventilateur, suivis des valeurs techniques et de l’état.](../images/midea-portasplit-home-assistant/portasplit-dashboard.png)

L’illustration montre le tableau de bord final en mode refroidissement avec les indicateurs, les commandes et les historiques des dernières 24 heures.

La série compte trois parties : la partie 1 décrit la configuration, la [partie 2](/blog/midea-portasplit-home-assistant-absichern) traite de la sécurisation du token, de la clé et du réseau domestique, tandis que la [partie 3](/blog/midea-v2-cloud-api-portasplit-home-assistant) met en contexte les avertissements concernant l’API cloud de Midea.

## Fonctionnement de la commande locale

Après la configuration, les commandes sont envoyées directement de Home Assistant à la PortaSplit, sans passer par un serveur Midea. Sur les appareils utilisant le protocole V3, la PortaSplit n’accepte toutefois les commandes locales qu’avec deux valeurs propres à l’appareil : le token et la clé. Lors de la configuration, l’intégration récupère une seule fois les deux via le cloud Midea et les enregistre localement :

```text
Einrichtung:   Home Assistant → Midea-Cloud → Token und Key
Betrieb:       Home Assistant → lokales Netz (6444/TCP) → PortaSplit
```

Les intégrations décrites sont issues de la communauté et ne sont officiellement prises en charge ni par Midea ni par Home Assistant. Des modifications du firmware ou du cloud peuvent influencer leur fonctionnement.

## Quelle intégration choisir

Deux intégrations communautaires prennent en charge la PortaSplit :

| Intégration | Spécificité |
|---|---|
| <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> (`Midea AC LAN`) | nombreuses catégories d’appareils Midea ; fournit 21 capteurs pour la PortaSplit, dont la fréquence, le courant et la tension du compresseur, ainsi que les températures de l’évaporateur, du condenseur et du gaz chaud |
| <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> (`Midea Smart AC`) | adaptée aux climatiseurs (`0xAC`, `0xCC`), interroge les capacités de l’appareil et prend en charge le mode silencieux de l’unité extérieure |

Pour ma PortaSplit, j’utilise `Midea AC LAN`; les instructions et le tableau de bord reposent sur ses entités. Avec `Midea Smart AC`, les entités portent d’autres noms ; le tableau de bord ne peut alors être utilisé qu’après adaptation. Utiliser les deux intégrations simultanément avec le même appareil entraîne des problèmes d’état et n’est pas pertinent.

## Prérequis

- Midea PortaSplit avec fonction Wi-Fi et un réseau Wi-Fi 2,4 GHz
- Application MSmartHome avec compte Midea
- Home Assistant à partir de la version 2024.10 (testé avec 2026.7), avec accès au répertoire de configuration `/config`, par exemple via l’add-on File editor, Samba ou SSH
- HACS pour la carte graphique `apexcharts-card`
- Accès réseau de Home Assistant à la PortaSplit sur le port 6444/TCP

## Étape 1 : connecter la PortaSplit à MSmartHome

1. Installez l’application MSmartHome et connectez-vous avec le compte Midea.
2. Placez la PortaSplit en mode d’association Wi-Fi et connectez-la au réseau Wi-Fi 2,4 GHz.
3. Vérifiez que la PortaSplit peut être commandée via l’application.
4. Créez une réservation DHCP pour la PortaSplit dans le routeur afin qu’elle reçoive durablement la même adresse IP.

Si le routeur utilise le même SSID pour les bandes 2,4 et 5 GHz, l’association fonctionne généralement malgré tout. En cas de problème, un réseau Wi-Fi 2,4 GHz séparé peut aider temporairement.

## Étape 2 : installer Midea AC LAN

**Via HACS :** ouvrez HACS, recherchez `Midea AC LAN`, téléchargez l’intégration et redémarrez Home Assistant.

**Sans HACS :** téléchargez directement l’archive de publication dans le répertoire `custom_components`. Dans une installation Docker, cela se fait avec `docker exec -it homeassistant bash` dans le conteneur ; sur Home Assistant OS, utilisez l’add-on Terminal :

```bash
mkdir -p /config/custom_components
cd /config/custom_components
wget https://github.com/wuwentao/midea_ac_lan/releases/download/v2026.9.2/midea_ac_lan.zip
unzip midea_ac_lan.zip -d midea_ac_lan
rm midea_ac_lan.zip
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `mkdir -p` | crée le répertoire s’il n’existe pas et ne signale pas d’erreur s’il existe déjà |
| `wget <url>` | télécharge l’archive de publication de la version indiquée depuis GitHub |
| `unzip <archiv>` | décompresse l’archive |
| `-d midea_ac_lan` | répertoire cible ; les fichiers se trouvent dans l’archive sans sous-répertoire et doivent se retrouver dans `custom_components/midea_ac_lan/` |

</details>

Le numéro de version actuel se trouve sur la page des publications du projet. Redémarrez ensuite Home Assistant via Paramètres, Système et Redémarrer. L’avertissement `We found a custom integration midea_ac_lan which has not been tested by Home Assistant` dans le journal est normal pour toute intégration personnalisée.

## Étape 3 : ajouter la PortaSplit

Sous Paramètres, Appareils et services, Ajouter une intégration, recherchez `Midea AC LAN`. L’assistant de configuration demande successivement :

1. **Action :** `Discover automatically`.
2. **Adresse IP :** `auto` parcourt le réseau local. Si la PortaSplit se trouve dans un autre VLAN, saisissez son adresse IP, car la recherche par diffusion ne traverse pas les limites de VLAN.
3. **Appareil :** la PortaSplit apparaît comme `<Geräte-ID> (Air Conditioner)`.
4. **Connexion :** compte, mot de passe et serveur. Pour un compte de l’application MSmartHome, sélectionnez `SmartHome`; si la connexion échoue, essayez `NetHome Plus` avec les mêmes identifiants. L’intégration récupère ainsi une seule fois le token et la clé.

L’appareil apparaît ensuite avec une seule entité `climate.<geräte-id>_climate`. L’identifiant de l’appareil est un nombre à 15 chiffres et est désigné ci-après par `DEVICE_ID`.

## Étape 4 : activer les capteurs

`Midea AC LAN` ne crée par défaut que l’entité de climatisation. Sous Paramètres, Appareils et services, Midea AC LAN, Configurer, la boîte de dialogue d’options s’ouvre :

| Champ | Réglage |
|---|---|
| Adresse IP | laisser inchangé |
| Refresh interval | 30 secondes (valeur par défaut) |
| Sensors | sélectionner tous les capteurs proposés |
| Switches | au minimum `Power`, `ECO Mode`, `Sleep Mode`, `Swing Vertical`, `Swing Horizontal`, `Screen Display`, `Prompt Tone` et `Fan Speed Percent` |
| Customize | laisser vide |

Après l’enregistrement, l’intégration crée les entités sans redémarrage, selon le modèle `sensor.DEVICE_ID_indoor_temperature`. Une PortaSplit (type d’appareil `0xAC`, protocole V3) avec `Midea AC LAN` v2026.9.2 fournissait les valeurs suivantes en veille :

| Entité | Signification | Valeur en veille |
|---|---|---|
| `indoor_temperature` | température ambiante | 23,0 °C |
| `outdoor_temperature` | température extérieure au niveau de l’unité extérieure | 23,5 °C |
| `realtime_power` | puissance absorbée actuelle | 1,5 W |
| `total_energy_consumption` | compteur d’énergie depuis la mise en service | 90,54 kWh |
| `compressor_frequency`, `target_compressor_frequency` | fréquence réelle et cible du compresseur | 0 Hz |
| `compressor_voltage`, `compressor_current`, `compressor_power` | tension, courant et puissance du compresseur | 230 V, 1 A, 4 W |
| `indoor_coil_temperature` (T2), `outdoor_coil_temperature` (T3) | évaporateur, condenseur | 23,5 °C |
| `discharge_pipe_temperature` (TP) | conduite de gaz chaud | 23 °C |
| `indoor_fan_speed` | vitesse du ventilateur | 0 rpm |
| `error_code`, `full_dust` | code d’erreur, filtre encrassé | 0, on |
| `indoor_humidity` | humidité de l’air | unknown |

La PortaSplit ne possède pas de capteur d’humidité ; `indoor_humidity` reste donc vide. L’entité de climatisation connaît les modes Arrêt, Auto, Refroidissement, Déshumidification, Chauffage et Ventilation, des températures de consigne de 16 à 30 °C par pas de 0,5 °C ainsi que les vitesses de ventilation Silent, Low, Medium, High, Full et Auto.

## Étape 5 : installer la carte graphique

Les graphiques d’historique utilisent `apexcharts-card`. Dans HACS, recherchez `apexcharts-card` et téléchargez-la. HACS enregistre la carte comme ressource de tableau de bord ; rechargez ensuite le navigateur une fois.

## Étape 6 : configurer les capteurs auxiliaires et le thème

Le tableau de bord nécessite cinq capteurs auxiliaires : compresseur activé/désactivé, vitesse du ventilateur sous forme numérique pour le graphique à paliers, durée de fonctionnement et énergie depuis minuit, ainsi que l’heure du dernier rapport. Activez d’abord les packages et les thèmes dans `configuration.yaml`, si ce n’est pas déjà fait :

```yaml
homeassistant:
  packages: !include_dir_named packages

frontend:
  themes: !include_dir_merge_named themes
```

Copiez ensuite `packages/portasplit.yaml` depuis le dépôt vers `/config/packages/` et `themes/portasplit.yaml` vers `/config/themes/`, puis remplacez dans le fichier de package chaque `DEVICE_ID` par votre propre identifiant d’appareil. Le package contient :

```yaml
template:
  - binary_sensor:
      - name: "PortaSplit Kompressor"
        unique_id: portasplit_kompressor
        icon: mdi:heat-pump-outline
        device_class: running
        state: >
          {% set f = states('sensor.DEVICE_ID_compressor_frequency') %}
          {% if f | is_number %}{{ f | float > 0 }}
          {% else %}{{ states('sensor.DEVICE_ID_realtime_power') | float(0) > 150 }}{% endif %}
  - sensor:
      - name: "PortaSplit Lüfterstufe"
        unique_id: portasplit_luefterstufe_num
        icon: mdi:fan
        state: >
          {% set m = state_attr('climate.DEVICE_ID_climate', 'fan_mode') %}
          {{ {'silent': 1, 'low': 2, 'medium': 3, 'high': 4,
              'full': 5, 'auto': 6}.get(m, none) }}
      - name: "PortaSplit letzte Meldung"
        unique_id: portasplit_letzte_meldung
        icon: mdi:sync
        device_class: timestamp
        state: "{{ states.climate['DEVICE_ID_climate'].last_reported }}"

sensor:
  - platform: history_stats
    name: "PortaSplit Laufzeit heute"
    unique_id: portasplit_laufzeit_heute
    entity_id: binary_sensor.portasplit_kompressor
    state: "on"
    type: time
    start: "{{ today_at() }}"
    end: "{{ now() }}"

utility_meter:
  portasplit_energie_heute:
    name: "PortaSplit Energie heute"
    unique_id: portasplit_energie_heute
    source: sensor.DEVICE_ID_total_energy_consumption
    cycle: daily
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `template: binary_sensor` | le compresseur est considéré comme en marche lorsque sa fréquence dépasse 0 Hz ; si un appareil ne transmet pas la fréquence, une puissance supérieure à 150 W sert de critère |
| `template: sensor` (vitesse du ventilateur) | convertit le mode de ventilation en un nombre de 1 (Silent) à 6 (Auto), afin que le graphique puisse dessiner des paliers |
| `states.climate['…']` | la notation entre crochets est nécessaire car l’ID de l’objet commence par un chiffre ; `states.climate.123…` n’est pas valide pour Jinja |
| `last_reported` | heure du dernier rapport de l’appareil, même si aucune valeur n’a changé |
| `history_stats` avec `type: time` | additionne le temps durant lequel le capteur du compresseur était `on` |
| `start` / `end` | période allant de minuit à maintenant |
| `utility_meter` avec `cycle: daily` | crée, à partir du compteur total, un compteur quotidien remis à 0 à minuit |

</details>

Vérifiez la configuration sous Outils de développement, YAML, Vérifier la configuration, puis redémarrez Home Assistant ; `utility_meter` ne peut pas être chargé par rechargement. Ensuite, `binary_sensor.portasplit_kompressor`, `sensor.portasplit_lufterstufe`, `sensor.portasplit_laufzeit_heute`, `sensor.portasplit_energie_heute` et `sensor.portasplit_letzte_meldung` existent. Home Assistant supprime le tréma lors de la création de l’ID d’entité, d’où `lufterstufe`.

## Étape 7 : créer le tableau de bord

1. Paramètres, Tableaux de bord, Ajouter un tableau de bord, Nouveau tableau de bord à partir de zéro, nom `PortaSplit`.
2. Ouvrez le nouveau tableau de bord et passez en mode édition via le crayon.
3. Ouvrez l’éditeur de configuration brute via le menu à trois points.
4. Remplacez entièrement le contenu par `dashboard.yaml` du dépôt, après avoir remplacé chaque `DEVICE_ID` par votre propre identifiant d’appareil.
5. Enregistrez et quittez le mode édition.

Le tableau de bord utilise la vue « Sections » à quatre colonnes et le thème `PortaSplit Dark`. Huit indicateurs se trouvent en haut, à gauche les commandes avec thermostat, mode de fonctionnement et vitesse du ventilateur, et à droite les historiques des dernières 24 heures. En dessous figurent les valeurs techniques du circuit frigorifique ainsi qu’un bloc d’état avec le code d’erreur, l’état du filtre et le dernier rapport.

Juste après la configuration, les graphiques sont vides et se remplissent au fil du temps. `Energie heute` affiche `Unbekannt`, jusqu’à ce que le compteur d’énergie augmente pour la première fois.

## Après la configuration

Le token et la clé se trouvent désormais dans Home Assistant. S’il n’est plus possible de les obtenir via le cloud par la suite, une sauvegarde est le seul moyen de procéder à une nouvelle configuration. La [partie 2 : sécuriser la PortaSplit](/blog/midea-portasplit-home-assistant-absichern) explique comment sauvegarder le token, la clé et la configuration, et isoler la PortaSplit sur le réseau domestique.

## Sources

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: intégration `Midea AC LAN` : catégories d’appareils prises en charge, installation via HACS, version minimale de Home Assistant 2024.4.1.

2.  [midea_ac_lan: Releases](https://github.com/wuwentao/midea_ac_lan/releases): archives de publication pour l’installation sans HACS, testées avec v2026.9.2.

3.  [midea_ac_lan : documentation des entités de climatisation](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/AC.md): entités et attributs des climatiseurs, dont la puissance, l’énergie totale et la fréquence du compresseur.

4.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: intégration `Midea Smart AC` : types d’appareils pris en charge `0xAC` et `0xCC`, PortaSplit avec « Out Silent Mode », utilisation du cloud pour obtenir le token et la clé sur les appareils V3, et port standard 6444.

5.  <a class="gh-badge" href="https://github.com/pfstr/ha-portasplit-dashboard" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">pfstr/ha-portasplit-dashboard</span></a>: tableau de bord, package de capteurs auxiliaires et thème de ce guide, avec la liste des entités qu’une PortaSplit avec `Midea AC LAN` v2026.9.2 transmet.

6.  [apexcharts-card](https://github.com/RomRider/apexcharts-card): carte graphique pour les graphiques d’historique.

7.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): installation d’intégrations personnalisées et de cartes frontend.

8.  [Home Assistant : Packages](https://www.home-assistant.io/docs/configuration/packages/): regroupement de la configuration Template, Sensor et Utility Meter dans un fichier sous `/config/packages/`.

9.  [Home Assistant : History Stats](https://www.home-assistant.io/integrations/history_stats/): plateforme de capteurs pour la durée de fonctionnement du compresseur depuis minuit.

10.  [Home Assistant : Utility Meter](https://www.home-assistant.io/integrations/utility_meter/): compteur quotidien basé sur le compteur total d’énergie.
