---
title: "RustDesk : configurer l’alternative open source à TeamViewer"
navTitle: "Configurer RustDesk"
description: "RustDesk est un logiciel de télémaintenance open source sous AGPL, gratuit et auto-hébergeable. Découvrez comment installer le client sous Windows (y compris de manière non surveillée via MSI), comment établir une connexion via le serveur de médiation public, votre propre serveur ou une connexion directe, quelles fonctions sont nécessaires au support quotidien et quelles sont les limites de l’utilisation gratuite."
date: "2026-09-01"
kategorie: "Télémaintenance et support"
timeToRead: "9 min de lecture"
themen:
  - fernwartung
produkte:
  - "rustdesk"
protokolle:
  - "haertung"
slug: "rustdesk-configurer-l-alternative-open-source-a-teamviewer"
translationId: "article-425ae4b8d562ae41"
translationOf: rustdesk-teamviewer-alternative
translationSourceHash: 3c44a4d5fc03c777c33f77525bcaebe75596ddde249f697041599b54e77cae2b
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T09:59:04.277Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/rustdesk-configurer-l-alternative-open-source-a-teamviewer
---

# RustDesk : configurer l’alternative open source à TeamViewer

TeamViewer et AnyDesk couvrent efficacement les besoins de télémaintenance, mais nécessitent une licence pour un usage professionnel, et les prix augmentent avec le nombre d’appareils gérés. RustDesk est une alternative sous licence AGPL-3.0 : open source, gratuite et sans obligation de licence. Le client fonctionne sous Windows, macOS, Linux, Android et iOS, ainsi que dans le navigateur. Il est écrit en Rust et son interface en Flutter.

La différence déterminante avec les produits commerciaux réside dans la médiation : RustDesk sépare le client de l’infrastructure serveur. Vous pouvez utiliser le serveur de médiation public gratuit, exploiter votre propre serveur ou établir une connexion directe sans aucun serveur de médiation. RustDesk peut ainsi être utilisé d’un poste individuel à une plateforme de support auto-hébergée, sans que les données de connexion doivent transiter par un fournisseur.

## Les trois types de connexion

Avant l’installation, vous devriez définir le type de connexion, car la configuration et les ports ouverts en dépendent.

| Type de connexion | Fonctionnement | Quand l’utiliser |
|---|---|---|
| Serveur de médiation public | Deux clients se trouvent via l’ID (numéro à neuf chiffres) sur le serveur RustDesk ; la connexion passe directement ou via un relais | Démarrage rapide, test, support privé occasionnel |
| Serveur propre (self-hosted) | Vous exploitez vous-même les composants serveur `hbbs` (médiation) et `hbbr` (relais), et tous les clients renseignent leur adresse | Usage professionnel, nombreux appareils, souveraineté totale sur les données |
| Connexion directe (Direct IP Access) | Le client se connecte directement à l’adresse IP de l’autre partie sans serveur de médiation | Les deux appareils sont joignables sur le même réseau ou via un VPN |

Le serveur public est explicitement destiné aux tests et à un usage privé. Pour une exploitation productive et professionnelle, le projet recommande un serveur propre, notamment parce que le service public est limité et ne comporte aucun engagement de disponibilité.

## Installation sous Windows

Téléchargez l’installateur depuis la source officielle, les versions GitHub du projet (`github.com/rustdesk/rustdesk`). Pour Windows, une version exécutable et un package MSI sont disponibles. Pour une installation interactive, un double-clic suffit. Si vous souhaitez déployer RustDesk sur plusieurs ordinateurs ou en arrière-plan, utilisez le MSI avec une installation silencieuse. Le nom du fichier contient le numéro de version ; la version actuelle est la 1.5.0 depuis le 30 septembre 2026 :

```powershell
msiexec /i rustdesk-1.5.0-x86_64.msi /qn /norestart
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `/i <paket>` | Installe le package MSI indiqué |
| `/qn` | Sans interface ni boîte de dialogue (silencieux) |
| `/norestart` | Empêche un redémarrage automatique après l’installation |

</details>

L’installation silencieuse configure le service `RustDesk`, qui s’exécute au démarrage du système et permet l’accès non surveillé. Après l’installation, vous pouvez lire l’ID de l’appareil depuis la ligne de commande sans ouvrir l’interface :

```powershell
& 'C:\Program Files\RustDesk\rustdesk.exe' --get-id
```

Vous pouvez également définir un mot de passe fixe pour l’accès non surveillé depuis la ligne de commande. Attribuez un mot de passe distinct et suffisamment long, et non le mot de passe de connexion de l’utilisateur :

```powershell
& 'C:\Program Files\RustDesk\rustdesk.exe' --password "IhrLangesEinmalpasswort"
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `--get-id` | Affiche l’ID RustDesk à neuf chiffres de l’appareil |
| `--password <wert>` | Définit le mot de passe fixe pour l’accès non surveillé |
| `--silent-install` | Installe la version exécutable (`.exe`) comme service, sans interface |

</details>

## Enregistrer son propre serveur

Si vous exploitez votre propre serveur de médiation, renseignez son adresse et sa clé publique dans les clients. Dans l’interface, ces paramètres se trouvent dans les réglages réseau sous ID Server, Relay Server et Key. Pour le déploiement en masse, la configuration peut aussi être définie dans un fichier ou au moyen de variables d’environnement, afin que chaque client démarre préconfiguré.

Un serveur propre nécessite les deux composants `hbbs` et `hbbr`, qui s’exécutent généralement dans des conteneurs Docker. Tous deux requièrent des ports ouverts afin que les clients puissent s’enregistrer et utiliser un relais.

| Port | Protocole | Composant et objectif |
|---|---|---|
| 21114 | TCP | Interface web de l’édition Pro (uniquement dans cette édition) |
| 21115 | TCP | `hbbs`, test du type de NAT |
| 21116 | TCP et UDP | `hbbs`, enregistrement (UDP) et établissement de la connexion (TCP) |
| 21117 | TCP | `hbbr`, trafic de relais |
| 21118, 21119 | TCP | Prise en charge des clients web |

N’ouvrez que les ports réellement nécessaires à votre type de connexion et limitez, au moyen du pare-feu, l’accès aux réseaux depuis lesquels le support est fourni.

## Connexion directe sans serveur de médiation

Si les deux appareils sont joignables sur le même réseau ou via un VPN, RustDesk fonctionne entièrement sans serveur de médiation. Pour cela, activez l’accès direct sur l’appareil cible (dans l’interface, sous sécurité, avec « Activer l’accès direct par IP », soit en interne le commutateur `direct-server`). Le client écoute alors sur le port standard 21118 (TCP). Dans la fenêtre de connexion, saisissez l’adresse IP de l’autre partie au lieu de l’ID.

Limitez l’accès direct au réseau depuis lequel vous accédez via le pare-feu. Si l’accès passe par un VPN, n’autorisez le port que pour la plage d’adresses VPN, et non pour l’ensemble d’Internet.

## Fonctions au quotidien pour le support

RustDesk couvre les fonctions nécessaires à la télémaintenance au quotidien :

- Partage d’écran et contrôle à distance du clavier et de la souris, avec sélection du moniteur en cas de plusieurs écrans.
- Transfert de fichiers dans les deux sens via une fenêtre scindée.
- Chat textuel pendant la session.
- Accès non surveillé au moyen d’un mot de passe fixe, pour les appareils sans utilisateur présent.
- Enregistrement de session sous forme de fichier vidéo, automatiquement si souhaité.
- Tunnel TCP et redirection permettant d’accéder localement à des services spécifiques de l’autre partie.
- Carnet d’adresses et plusieurs appareils enregistrés, localement dans l’édition gratuite et partagés côté serveur dans l’édition Pro.

Pour le support assisté, un point important est le suivant : par défaut, RustDesk demande à l’autre partie d’accepter la connexion et indique pendant la session qu’un accès est en cours. La personne devant l’appareil est donc informée. Seul un mot de passe fixe pour l’accès non surveillé supprime cette demande de confirmation. N’utilisez l’accès non surveillé que sur des appareils dont les utilisateurs savent que le logiciel est installé et à quoi il sert.

## Restrictions et limites

RustDesk remplace TeamViewer dans de nombreux cas, mais présente des limites à connaître avant son utilisation :

- Le serveur de médiation public est limité, sans engagement de disponibilité et non prévu pour un fonctionnement professionnel continu. Pour travailler de manière fiable, hébergez votre propre serveur.
- Un serveur propre implique une charge d’exploitation : conteneurs, ports ouverts, certificats et mises à jour sont à votre charge.
- Un carnet d’adresses partagé côté serveur, une gestion centralisée des utilisateurs et l’interface web d’administration font partie de l’édition Pro, qui devient payante à partir d’un certain nombre d’appareils. Le client lui-même et l’utilisation de base restent gratuits.
- Sans mot de passe fixe, aucun accès non surveillé n’est possible. C’est correct pour le support assisté, mais cela empêche un accès spontané à un appareil inoccupé.
- L’étendue des fonctions et la stabilité sur certaines plateformes, en particulier les appareils mobiles, n’atteignent pas les produits commerciaux dans tous les détails. Vérifiez les fonctions importantes pour vous avant de migrer.
- Certains logiciels de sécurité signalent les logiciels de télémaintenance comme potentiellement indésirables. Ajoutez si nécessaire une exception et documentez la raison pour laquelle le logiciel est installé.

Pour un usage privé et le support de quelques appareils, l’édition gratuite avec le serveur public ou une connexion directe suffit. Dès que vous gérez de nombreux appareils, travaillez à titre professionnel ou avez besoin d’une souveraineté totale sur les données, vous avez besoin de votre propre serveur, avec la charge d’exploitation correspondante en contrepartie de l’indépendance.

## Historique des modifications

- **25 octobre 2026 :** exemple d’installation mis à jour pour RustDesk 1.5.0 du 30 septembre 2026.

## Sources

1.  [RustDesk sur GitHub](https://github.com/rustdesk/rustdesk): code source, versions avec les installateurs et licence AGPL-3.0.

2.  [Documentation RustDesk](https://rustdesk.com/docs/): installation, serveur propre, ports et configuration des clients.

3.  [rustdesk-server sur GitHub](https://github.com/rustdesk/rustdesk-server): composants serveur `hbbs` et `hbbr`, y compris la vue d’ensemble des ports pour l’auto-hébergement.
