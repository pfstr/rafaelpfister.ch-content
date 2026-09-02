---
title: "RustDesk : configurer l’alternative open source à TeamViewer"
navTitle: "Configurer RustDesk"
description: "RustDesk est un logiciel de prise en main à distance open source sous AGPL, gratuit et auto-hébergeable. Découvrez comment installer le client sous Windows (y compris sans surveillance via MSI), comment établir une connexion via le serveur de médiation public, votre propre serveur ou une connexion directe, quelles fonctions sont nécessaires au support quotidien et où se situent les limites de l’utilisation gratuite."
date: "2026-09-01"
kategorie: "Prise en main à distance et support"
timeToRead: "9 min de lecture"
themen:
  - fernwartung
produkte:
  - "rustdesk"
protokolle:
  - "haertung"
slug: "rustdesk-configurer-l-alternative-open-source-a-teamviewer"
translationId: "article-425ae4b8d562ae41"
aiPrompt: |
  Du bist mein IT-Support-Assistent. Hilf mir, RustDesk als quelloffene TeamViewer-Alternative einzurichten: Client installieren, Verbindungsart wählen (öffentlicher Vermittlungsserver, eigener Server oder Direktverbindung über ein privates Netz), unbeaufsichtigten Zugriff absichern und die Grenzen der kostenlosen Nutzung einordnen.
translationOf: rustdesk-teamviewer-alternative
url: https://rafaelpfister.ch/fr/blog/rustdesk-configurer-l-alternative-open-source-a-teamviewer
translationSourceHash: f812fc4b04abe0aa92cca47b285a30a18f5cd1e99ab328593b224ee26051a7f3
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T08:48:42.193Z
translationReview: automatic
---

# RustDesk : configurer l’alternative open source à TeamViewer

TeamViewer et AnyDesk assurent de manière fiable la prise en main à distance, mais nécessitent une licence pour une utilisation commerciale, et les prix augmentent avec le nombre d’appareils pris en charge. RustDesk est une alternative sous licence AGPL-3.0 : open source, gratuite et sans obligation de licence. Le client fonctionne sous Windows, macOS, Linux, Android et iOS, ainsi que dans le navigateur. Il est écrit en Rust, avec une interface en Flutter.

La différence décisive avec les produits commerciaux réside dans la médiation : RustDesk sépare le client de l’infrastructure serveur. Vous pouvez utiliser le serveur de médiation public gratuit, exploiter votre propre serveur ou établir une connexion directe sans serveur de médiation. RustDesk peut ainsi être utilisé d’un poste individuel à une plateforme de support auto-hébergée, sans que les données de connexion ne transitent par un fournisseur.

## Les trois types de connexion

Avant l’installation, vous devriez définir le type de connexion, car la configuration et les ports à ouvrir en dépendent.

| Type de connexion | Fonctionnement | Quand l’utiliser |
|---|---|---|
| Serveur de médiation public | Deux clients se trouvent via l’ID (numéro à neuf chiffres) sur le serveur RustDesk ; la connexion est directe ou passe par un relais | Démarrage rapide, test, support occasionnel privé |
| Serveur propre (auto-hébergé) | Vous exploitez vous-même les composants serveur `hbbs` (médiation) et `hbbr` (relais), et tous les clients renseignent leur adresse | Utilisation commerciale, nombreux appareils, souveraineté totale des données |
| Connexion directe (Direct IP Access) | Le client se connecte directement à l’adresse IP de l’autre extrémité, sans serveur de médiation | Les deux appareils sont accessibles sur le même réseau ou via un VPN |

Le serveur public est explicitement destiné aux tests et à l’usage privé. Pour une exploitation commerciale en production, le projet recommande son propre serveur, notamment parce que le service public est bridé et ne comporte aucune garantie de disponibilité.

## Installation sous Windows

Téléchargez l’installateur depuis la source officielle, les releases GitHub du projet (`github.com/rustdesk/rustdesk`). Pour Windows, une version exécutable et un paquet MSI sont disponibles. Pour une installation interactive, un double-clic suffit. Si vous souhaitez déployer RustDesk sur plusieurs ordinateurs ou en arrière-plan, utilisez le MSI avec une installation silencieuse :

```powershell
msiexec /i rustdesk-1.4.9-x86_64.msi /qn /norestart
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `/i <paket>` | Installe le paquet MSI indiqué |
| `/qn` | Aucune interface, aucune boîte de dialogue (silencieux) |
| `/norestart` | Empêche un redémarrage automatique après l’installation |

</details>

L’installation silencieuse configure le service `RustDesk`, qui démarre avec le système et permet l’accès sans surveillance. Après l’installation, récupérez l’ID de l’appareil depuis la ligne de commande, sans ouvrir l’interface :

```powershell
& 'C:\Program Files\RustDesk\rustdesk.exe' --get-id
```

Vous pouvez également définir un mot de passe fixe pour l’accès sans surveillance depuis la ligne de commande. Choisissez un mot de passe distinct et suffisamment long, et non le mot de passe de connexion de l’utilisateur :

```powershell
& 'C:\Program Files\RustDesk\rustdesk.exe' --password "IhrLangesEinmalpasswort"
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `--get-id` | Affiche l’ID RustDesk à neuf chiffres de l’appareil |
| `--password <wert>` | Définit le mot de passe fixe pour l’accès sans surveillance |
| `--silent-install` | Installe la version exécutable (`.exe`) sans interface en tant que service |

</details>

## Configurer son propre serveur

Si vous exploitez votre propre serveur de médiation, renseignez son adresse et la clé publique dans les clients. Dans l’interface, cela se trouve dans les paramètres réseau sous ID Server, Relay Server et Key. Pour un déploiement de masse, la configuration peut également être fournie sous forme de fichier ou via des variables d’environnement, afin que chaque client démarre préconfiguré.

Un serveur propre nécessite les deux composants `hbbs` et `hbbr`, qui s’exécutent généralement sous forme de conteneurs Docker. Tous deux nécessitent des ports ouverts afin que les clients puissent s’enregistrer et utiliser un relais.

| Port | Protocole | Composant et objectif |
|---|---|---|
| 21114 | TCP | Interface web de l’édition Pro (uniquement dans celle-ci) |
| 21115 | TCP | `hbbs`, test du type de NAT |
| 21116 | TCP et UDP | `hbbs`, enregistrement (UDP) et établissement de la connexion (TCP) |
| 21117 | TCP | `hbbr`, trafic de relais |
| 21118, 21119 | TCP | Prise en charge des clients web |

N’ouvrez que les ports réellement nécessaires à votre type de connexion et limitez, via le pare-feu, l’accès aux réseaux depuis lesquels le support est assuré.

## Connexion directe sans serveur de médiation

Si les deux appareils sont accessibles sur le même réseau ou via un VPN, RustDesk peut fonctionner entièrement sans serveur de médiation. Activez pour cela l’accès direct sur l’appareil cible (dans l’interface, sous sécurité, « Activer l’accès direct par IP », le commutateur interne étant `direct-server`). Le client écoute alors sur le port standard 21118 (TCP). Dans la fenêtre de connexion, saisissez l’adresse IP de l’autre extrémité au lieu de l’ID.

Limitez l’accès direct, via le pare-feu, au réseau depuis lequel vous accédez à l’appareil. Si l’accès passe par un VPN, n’autorisez le port que pour la plage d’adresses VPN, et non pour l’ensemble d’Internet.

## Fonctions au quotidien dans le support

RustDesk couvre les fonctions nécessaires à la prise en main à distance au quotidien :

- Partage d’écran et contrôle à distance du clavier et de la souris, avec sélection du moniteur en cas de plusieurs écrans.
- Transfert de fichiers dans les deux sens via une fenêtre divisée en deux.
- Chat textuel pendant la session.
- Accès sans surveillance au moyen d’un mot de passe fixe, pour les appareils sans utilisateur présent.
- Enregistrement de session sous forme de fichier vidéo, automatiquement si souhaité.
- Tunnel TCP et redirection pour accéder localement à certains services de l’autre extrémité.
- Carnet d’adresses et plusieurs appareils enregistrés, localement dans la version gratuite et partagés côté serveur dans l’édition Pro.

Pour le support assisté, un point est important : par défaut, RustDesk demande du côté distant si la connexion doit être acceptée et indique pendant la session qu’un accès est en cours. La personne devant l’appareil est donc informée. Seul un mot de passe fixe pour l’accès sans surveillance supprime cette demande de confirmation. N’utilisez l’accès sans surveillance que sur des appareils dont les utilisateurs savent que le logiciel est installé et à quoi il sert.

## Restrictions et limites

RustDesk remplace TeamViewer dans de nombreux cas, mais présente des limites que vous devriez connaître avant son utilisation :

- Le serveur de médiation public est bridé, sans garantie de disponibilité et n’est pas prévu pour une exploitation commerciale continue. Pour travailler de manière fiable, hébergez vous-même le service.
- Un serveur propre implique une charge d’exploitation : conteneurs, ports ouverts, certificats et mises à jour sont à votre charge.
- Un carnet d’adresses partagé côté serveur, une gestion centralisée des utilisateurs et l’interface web d’administration font partie de l’édition Pro, qui est payante à partir d’un certain nombre d’appareils. Le client lui-même et le fonctionnement de base restent gratuits.
- Sans mot de passe fixe, aucun accès sans surveillance n’est possible, ce qui est correct pour le support assisté, mais empêche l’accès spontané à un appareil inoccupé.
- L’étendue fonctionnelle et la stabilité de certaines plateformes, en particulier sur les appareils mobiles, n’atteignent pas les produits commerciaux dans tous les détails. Vérifiez les fonctions importantes pour vous avant de migrer.
- Certains programmes de sécurité signalent les logiciels de prise en main à distance comme potentiellement indésirables. Ajoutez une exception si nécessaire et documentez pourquoi le logiciel est installé.

Pour un usage privé et le support d’appareils individuels, la version gratuite avec le serveur public ou une connexion directe suffit. Dès lors que vous gérez de nombreux appareils, travaillez de façon commerciale ou avez besoin d’une souveraineté totale sur les données, vous avez besoin de votre propre serveur, avec la charge d’exploitation correspondante en contrepartie de l’indépendance.

## Sources

1.  [RustDesk sur GitHub](https://github.com/rustdesk/rustdesk): code source, releases avec les installateurs et licence AGPL-3.0.

2.  [Documentation RustDesk](https://rustdesk.com/docs/): installation, serveur propre, ports et configuration des clients.

3.  [rustdesk-server sur GitHub](https://github.com/rustdesk/rustdesk-server): composants serveur `hbbs` et `hbbr`, y compris la vue d’ensemble des ports pour l’exploitation en propre.
