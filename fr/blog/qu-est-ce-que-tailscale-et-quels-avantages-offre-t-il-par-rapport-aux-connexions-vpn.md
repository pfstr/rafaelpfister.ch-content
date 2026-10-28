---
title: "Qu’est-ce que Tailscale et quels avantages offre-t-il par rapport aux connexions VPN traditionnelles ?"
navTitle: "Tailscale vs. VPN"
description: "Tailscale s’appuie sur WireGuard pour créer un VPN maillé dans lequel les appareils se connectent directement plutôt que via un concentrateur VPN central. Découvrez comment les serveurs de coordination, la traversée NAT et les relais DERP interagissent, quels sont les avantages par rapport aux VPN IPsec et SSL, et quelles dépendances et limites vous devez connaître avant de l’utiliser."
date: "2026-10-01"
kategorie: "VPN et accès à distance"
timeToRead: "11 min de lecture"
themen:
  - vpn-fernzugriff
produkte:
  - "tailscale"
protokolle:
  - "tcp"
  - "haertung"
slug: "qu-est-ce-que-tailscale-et-quels-avantages-offre-t-il-par-rapport-aux-connexions-vpn"
translationId: "article-91fddf1e0239f5c4"
translationOf: tailscale-vorteile-vpn
translationSourceHash: f7642d64432f28cd0e53fd833c1accbd7a2a206550be04e07ace9cea8d23a04f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-27T09:21:13.968Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/qu-est-ce-que-tailscale-et-quels-avantages-offre-t-il-par-rapport-aux-connexions-vpn
---

# Qu’est-ce que Tailscale et quels avantages offre-t-il par rapport aux connexions VPN traditionnelles ?

Tailscale est un service VPN qui relie des appareils à un réseau privé, appelé Tailnet. Techniquement, il repose sur WireGuard. La différence avec un VPN d’entreprise classique réside dans la topologie : les appareils établissent directement entre eux leurs tunnels chiffrés (mesh). Il n’y a pas de concentrateur VPN central par lequel transite l’ensemble du trafic. Seule l’administration reste centralisée : un serveur de coordination distribue les clés publiques, les adresses et les règles d’accès, mais ne voit lui-même aucune donnée utile.

Pour démarrer, il suffit d’avoir un compte auprès d’un fournisseur d’identité (Microsoft, Google, GitHub, Apple ou un fournisseur OIDC) et le client sur chaque appareil. En règle générale, aucune ouverture de port dans le pare-feu n’est nécessaire. Tailscale est donc répandu aussi bien pour les réseaux domestiques que pour l’accès à distance aux serveurs et aux systèmes clients.

## Comment fonctionne un VPN traditionnel

Un VPN d’accès à distance classique fonctionne selon le principe hub-and-spoke. Une passerelle VPN (pare-feu ou appliance) se trouve à la périphérie du réseau d’entreprise et est accessible depuis Internet. Le client sur l’ordinateur portable établit un tunnel vers cette passerelle ; les solutions courantes sont IPsec/IKEv2 (UDP 500 et 4500), les variantes SSL-VPN via TCP 443 ou OpenVPN. Après l’authentification, l’appareil reçoit une adresse provenant d’un pool ainsi que des routes vers les réseaux internes.

Ce modèle a fait ses preuves pendant des décennies, mais il présente des caractéristiques structurelles qui génèrent des efforts dans l’exploitation actuelle :

- **Passerelle accessible publiquement :** Le concentrateur VPN doit être accessible depuis Internet et constitue donc une cible privilégiée. Ces dernières années, des vulnérabilités dans des appliances VPN ont été activement exploitées à plusieurs reprises ; en janvier 2024, l’agence américaine CISA a même ordonné, par l’Emergency Directive 24-01, de déconnecter du réseau les passerelles Ivanti concernées.
- **Point de défaillance unique et goulot d’étranglement :** Tout le trafic passe par la passerelle. En cas de panne ou de saturation de la bande passante, tous les utilisateurs sont affectés simultanément.
- **Détours (hairpinning) :** Si deux collaborateurs en télétravail accèdent au même serveur dans le cloud, le trafic passe d’abord par le centre de données, puis ressort de celui-ci.
- **Droits d’accès grossiers :** Après l’établissement de la connexion, un segment de réseau entier est souvent accessible à l’appareil. Des règles fines par utilisateur et par service sont possibles, mais rarement maintenues de manière systématique.
- **Effort site à site :** Chaque site supplémentaire nécessite son propre tunnel avec des paramètres harmonisés (propositions Phase 1/Phase 2, clés prépartagées ou certificats, adresses IP publiques fixes).

## Architecture de Tailscale

Tailscale sépare le plan de contrôle du plan de données. Le plan de contrôle est le serveur de coordination, que Tailscale exploite comme service cloud. Le plan de données est constitué des tunnels WireGuard entre les appareils.

| Composant | Rôle |
|---|---|
| Client (`tailscaled`) | Génère la paire de clés localement, établit des tunnels WireGuard vers les autres nœuds et applique localement les règles d’accès |
| Serveur de coordination | Authentifie les appareils via le fournisseur d’identité, distribue les clés publiques, les adresses, les paramètres DNS et la politique à tous les nœuds |
| Relais DERP | Transmettent les paquets chiffrés lorsqu’une connexion directe ne peut pas être établie |
| Relais pairs | Appareils propres au sein du Tailnet servant de relais à débit supérieur ; ils sont privilégiés par rapport à DERP |

La clé privée d’un appareil ne quitte jamais l’appareil. Le serveur de coordination ne connaît que les clés publiques et ne peut donc pas déchiffrer le trafic. Chaque appareil reçoit une adresse fixe de la plage `100.64.0.0/10` (l’espace d’adressage pour le NAT de niveau opérateur) ainsi qu’une adresse IPv6 de `fd7a:115c:a1e0::/48`. Grâce à MagicDNS, les appareils sont en outre accessibles par leur nom, par exemple `nas` ou `nas.tailnet-name.ts.net`.

### Traversée NAT : pourquoi aucune ouverture de port n’est nécessaire

La plupart des appareils se trouvent derrière un routeur NAT ou un pare-feu et ne sont pas directement accessibles depuis l’extérieur. Tailscale résout cela par la traversée NAT : les deux parties déterminent, via STUN, leur adresse publique et le port attribué, échangent ces informations par l’intermédiaire du serveur de coordination et s’envoient simultanément des paquets UDP. Les paquets sortants ouvrent une entrée d’état dans les deux pare-feu, par laquelle les paquets de l’autre partie peuvent ensuite entrer (UDP Hole Punching).

Si cela échoue, par exemple avec des pare-feu restrictifs qui bloquent les UDP sortants ou avec certaines formes de NAT de niveau opérateur, le trafic passe par un relais DERP via HTTPS. Là aussi, il reste chiffré de bout en bout avec WireGuard ; le relais ne voit que des paquets chiffrés. Le prix à payer est une latence plus élevée et un débit moindre. Les relais pairs réduisent cet inconvénient en confiant le transfert à un appareil propre disposant d’une bonne connectivité.

## Les avantages par rapport à un VPN traditionnel

| Critère | VPN traditionnel | Tailscale |
|---|---|---|
| Topologie | Hub-and-spoke via une passerelle centrale | Mesh, connexions directes entre les appareils |
| Ports entrants | La passerelle doit être accessible depuis Internet | Aucune ouverture de port entrant nécessaire |
| Authentification | Comptes locaux, RADIUS, certificats, souvent MFA distinct | Connexion via le fournisseur d’identité existant, y compris son MFA |
| Droits d’accès | Souvent par segment de réseau | Par utilisateur, groupe, appareil et port dans une politique centrale |
| Nouveau site | Tunnel site à site avec paramètres harmonisés | Installer un client ou un routeur de sous-réseau |
| Panne du centre | Plus aucun accès pour tous | Les connexions existantes continuent ; les nouveaux appareils et les modifications de politique attendent |
| Protocole | IPsec, SSL-VPN, OpenVPN | WireGuard |

### Surface d’attaque réduite

Comme les clients établissent leurs connexions de manière sortante, aucun appareil n’a besoin d’un port ouvert sur Internet. Un serveur qui ne doit être accessible que via le Tailnet peut lier ses services exclusivement à l’interface Tailscale. Il est alors invisible pour les scanners de ports sur Internet. WireGuard lui-même, avec environ 4 000 lignes de code noyau, est nettement plus petit que les implémentations IPsec ou SSL-VPN typiques et utilise un ensemble fixe de procédés modernes (Curve25519, ChaCha20-Poly1305, BLAKE2s). Il n’y a pas de négociation de suites cryptographiques, comme c’est le cas avec IPsec et conduit régulièrement à des erreurs de configuration.

### L’identité plutôt que l’adresse réseau

Chaque appareil du Tailnet est associé à un utilisateur ou à un tag. La connexion s’effectue via le fournisseur d’identité que vous utilisez déjà ; le MFA et l’accès conditionnel de Microsoft Entra ID s’appliquent donc également à l’accès réseau. Les nouveaux appareils ne peuvent rejoindre le Tailnet qu’après une authentification réussie auprès du fournisseur d’identité, et chaque clé d’appareil expire par défaut après 180 jours. Lorsqu’un collaborateur quitte l’entreprise, vous bloquez son compte auprès du fournisseur d’identité ; avec le provisionnement SCIM (à partir de l’offre Standard), l’utilisateur est automatiquement désactivé dans le Tailnet et ses appareils perdent l’accès.

### Règles d’accès selon le principe du moindre privilège

Par défaut, dans un nouveau Tailnet, chaque appareil peut joindre tous les autres. Pour une utilisation en production, vous définissez les droits d’accès dans un fichier de politique central (HuJSON). L’exemple suivant autorise le groupe des administrateurs à accéder en SSH et HTTPS à tous les serveurs portant le tag `tag:server`, et tous les autres utilisateurs uniquement en HTTPS au serveur intranet :

```json
{
  "groups": {
    "group:admins": ["admin@example.com"]
  },
  "tagOwners": {
    "tag:server": ["group:admins"]
  },
  "grants": [
    {
      "src": ["group:admins"],
      "dst": ["tag:server"],
      "ip":  ["tcp:22", "tcp:443"]
    },
    {
      "src": ["autogroup:member"],
      "dst": ["intranet"],
      "ip":  ["tcp:443"]
    }
  ],
  "hosts": {
    "intranet": "100.101.102.103"
  }
}
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `groups` | Définit des groupes d’utilisateurs ; les membres sont indiqués par leur adresse de connexion auprès du fournisseur d’identité |
| `tagOwners` | Définit qui peut attribuer un tag aux appareils ; les appareils tagués n’appartiennent à aucun utilisateur, mais au tag |
| `grants` | Liste des connexions autorisées ; tout ce qui n’est pas explicitement autorisé est bloqué |
| `src` | Source de la connexion : utilisateur, groupe, tag ou `autogroup:member` (tous les utilisateurs du Tailnet) |
| `dst` | Destination de la connexion : tag, alias d’hôte, appareil ou sous-réseau |
| `ip` | Protocoles et ports autorisés, par exemple `tcp:22` ou `*` pour tout |
| `hosts` | Noms d’alias pour les adresses Tailnet ou les sous-réseaux utilisés dans les règles |

</details>

Chaque client applique les règles localement. Un paquet non autorisé par la politique est déjà rejeté par l’appareil de destination. Le serveur de coordination ne fait que distribuer la politique.

### Connexions directes plutôt que détours

Comme les tunnels existent directement entre les appareils, le trafic emprunte le chemin le plus court. Deux appareils dans le même bureau communiquent localement, tandis qu’un ordinateur portable en télétravail atteint directement un serveur cloud. Cela réduit la latence et soulage la connexion Internet du site principal.

### Connecter des réseaux existants

Tous les appareils ne peuvent pas exécuter un client Tailscale, comme les imprimantes, les systèmes NAS anciens ou les contrôleurs industriels. Dans ces cas, un routeur de sous-réseau joue le rôle de passerelle : un serveur Linux du réseau cible annonce le sous-réseau local dans le Tailnet (advertise), et les appareils autorisés y accèdent par son intermédiaire.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
echo 'net.ipv4.ip_forward = 1' | \
  sudo tee /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
sudo tailscale up \
  --advertise-routes=192.168.10.0/24 \
  --advertise-tags=tag:server
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `curl -fsSL …/install.sh \| sh` | Télécharge le script d’installation officiel et configure le dépôt de paquets de la distribution |
| `net.ipv4.ip_forward = 1` | Autorise le noyau Linux à transférer des paquets entre interfaces ; sans ce paramètre, aucun routeur de sous-réseau ne fonctionne |
| `sysctl -p <datei>` | Charge immédiatement le paramètre, sans redémarrage |
| `tailscale up` | Connecte l’appareil au Tailnet ; un lien de connexion apparaît lors du premier appel |
| `--advertise-routes=<subnetz>` | Annonce le sous-réseau indiqué dans le Tailnet ; plusieurs sous-réseaux sont séparés par des virgules |
| `--advertise-tags=<tag>` | Attribue à l’appareil un tag auquel se rapportent les règles d’accès |

</details>

La route annoncée doit ensuite être approuvée dans la console d’administration, sauf si une approbation automatique (`autoApprovers`) est configurée dans la politique. De même, un appareil peut être utilisé comme nœud de sortie avec `--advertise-exit-node`. Les clients qui sélectionnent ce nœud de sortie y acheminent alors l’ensemble de leur trafic Internet, ce qui correspond au mode tunnel complet d’un VPN classique.

### Moins d’efforts d’exploitation

La rotation des clés, l’attribution des adresses, le DNS et le routage sont gérés par le serveur de coordination. Un nouveau site nécessite un routeur de sous-réseau avec accès à Internet, mais pas d’adresse IP publique fixe ni d’harmonisation des paramètres IPsec avec la partie opposée. Pour l’accès à distance aux serveurs, Tailscale SSH (connexion via l’identité Tailnet sans clés SSH distribuées) et `tailscale serve` (partage d’un service web local dans le Tailnet) sont également disponibles.

## Limites et dépendances

Tailscale remplace la passerelle VPN classique, mais transfère une partie de la responsabilité vers un service externe. Vous devriez examiner les points suivants avant une introduction.

| Sujet | Points à considérer |
|---|---|
| Dépendance au fournisseur | Le plan de contrôle est un service cloud de Tailscale Inc. En cas de panne, les connexions existantes continuent de fonctionner, mais les nouveaux appareils, connexions et modifications de politique sont impossibles jusqu’au rétablissement |
| Métadonnées | Les noms d’appareils, adresses Tailnet, adresses IP publiques, comptes utilisateurs et horaires de connexion sont traités par le fournisseur ; pas les données utiles |
| Confiance dans la distribution des clés | Le serveur de coordination détermine quelles clés publiques un appareil accepte. Tailnet Lock exige pour cela une signature supplémentaire d’appareils propres dignes de confiance |
| Code source | Le cœur du client est open source (BSD-3-Clause), le serveur de coordination ne l’est pas. Headscale est une alternative open source auto-hébergée aux fonctionnalités réduites |
| Conflits d’adresses | `100.64.0.0/10` est également utilisé par certains fournisseurs pour le NAT de niveau opérateur et par d’autres produits VPN ; les chevauchements entraînent des problèmes de routage |
| Coexistence avec d’autres clients VPN | Un second client VPN avec tunnel complet peut rediriger le trafic Tailscale. La plage `100.64.0.0/10` et le service Tailscale doivent alors être exclus du tunnel |
| Performances via les relais | Lorsqu’aucune connexion directe ne peut être établie, le débit via DERP diminue sensiblement ; `tailscale netcheck` indique si l’UDP sortant fonctionne |
| Ne remplace pas le filtrage web | Tailscale régit l’accès aux ressources internes. Il ne prend pas en charge le filtrage de contenu ni l’inspection du trafic Internet |

Pour les entreprises en Suisse, la loi fédérale révisée sur la protection des données (nLPD) s’applique. Comme des données personnelles des utilisateurs sont traitées par le fournisseur (comptes, appareils, métadonnées de connexion), inscrivez Tailscale dans le registre des activités de traitement si votre entreprise doit en tenir un (à partir de 250 collaborateurs ou en cas de traitements à risque élevé). Vérifiez l’accord de sous-traitance (Data Processing Addendum) du fournisseur ainsi que la base juridique de la communication à l’étranger conformément à l’art. 16 nLPD, par exemple une certification dans le cadre du Swiss-U.S. Data Privacy Framework ou des clauses contractuelles types. Si tout traitement de données par le fournisseur est exclu, Headscale reste une solution de plan de contrôle auto-exploitée.

> **Remarque UE :** Pour les établissements situés dans l’UE, le RGPD et ses règles concernant les transferts vers des pays tiers (art. 44 et suivants du RGPD) s’appliquent. L’examen est similaire sur le fond ; le EU-U.S. Data Privacy Framework y est déterminant.

## Coûts

L’offre Personal est gratuite et comprend jusqu’à six utilisateurs, un nombre illimité d’appareils utilisateurs et 50 appareils tagués (état en octobre 2026). Pour une utilisation professionnelle, l’offre Standard coûte 8 USD et Premium 18 USD par utilisateur et par mois. Premium ajoute notamment les Network Flow Logs, le streaming de journaux et des options avancées pour Tailscale SSH. Enterprise est proposé sur mesure.

## À qui Tailscale convient

Tailscale convient particulièrement lorsque les utilisateurs et les ressources sont répartis : télétravail, serveurs cloud chez plusieurs fournisseurs, petits sites distants sans adresse IP fixe ou maintenance à distance chez les clients. Pour les petites et moyennes entreprises, il peut remplacer entièrement une passerelle VPN. Dans les environnements plus grands, il fonctionne souvent en parallèle du VPN existant, par exemple pour l’accès administratif aux serveurs, où les règles d’accès fines apportent le plus grand bénéfice.

Il est moins adapté lorsque des exigences imposent une infrastructure entièrement auto-exploitée et que Headscale ne couvre pas les fonctionnalités nécessaires, ou lorsque l’ensemble du trafic Internet doit être filtré de manière centralisée. Dans ce cas, une Secure Web Gateway reste nécessaire en complément de Tailscale.

Pour le tester, il suffit d’installer le client sur deux appareils et de se connecter avec le même compte. Avec `tailscale status`, vous voyez ensuite tous les appareils du Tailnet ; avec `tailscale ping <gerät>`, vous vérifiez si une connexion directe ou un relais est utilisé.

## Historique des modifications

- **25 octobre 2026 :** Liens des sources mis à jour vers les nouvelles adresses de Tailscale.

## Sources

1.  [Tailscale: How Tailscale works](https://tailscale.com/blog/how-tailscale-works): architecture avec serveur de coordination, mesh WireGuard et application locale des règles.

2.  [Tailscale: How NAT traversal works](https://tailscale.com/blog/how-nat-traversal-works): explication détaillée de STUN, UDP Hole Punching et des cas où un relais devient nécessaire.

3.  [Tailscale Docs: DERP servers](https://tailscale.com/docs/reference/derp-servers): fonction et chiffrement des serveurs relais.

4.  [Tailscale Docs: Tailscale Peer Relays](https://tailscale.com/docs/features/peer-relay): appareils propres utilisés comme relais, prioritaires sur DERP.

5.  [Tailscale Docs: Grants](https://tailscale.com/docs/features/access-control/grants): syntaxe des règles d’accès dans le fichier de politique.

6.  [Tailscale Docs: Subnet routers](https://tailscale.com/docs/features/subnet-routers): configuration des routeurs de sous-réseau, y compris l’IP forwarding et l’approbation des routes.

7.  [Tailscale Docs: Exit nodes](https://tailscale.com/docs/features/exit-nodes): acheminer l’ensemble du trafic Internet via un appareil du Tailnet.

8.  [Tailscale Docs: Tailnet Lock](https://tailscale.com/docs/features/tailnet-lock): signature des nouveaux appareils par des nœuds propres et dignes de confiance.

9.  [Tailscale: Pricing](https://tailscale.com/pricing): offres et limites, consultées le 1er octobre 2026.

10.  [WireGuard: Next Generation Kernel Network Tunnel (Whitepaper)](https://www.wireguard.com/papers/wireguard.pdf): conception du protocole et procédés cryptographiques utilisés.

11.  [GitHub: tailscale/tailscale](https://github.com/tailscale/tailscale): code source du client sous BSD-3-Clause.

12.  [GitHub: juanfont/headscale](https://github.com/juanfont/headscale): implémentation open source du serveur de coordination à auto-héberger.

13.  [CISA: Emergency Directive 24-01](https://www.cisa.gov/news-events/directives/ed-24-01-mitigate-ivanti-connect-secure-and-ivanti-policy-secure-vulnerabilities): ordre de déconnexion des passerelles VPN Ivanti vulnérables en janvier 2024.

14.  [Fedlex: Loi fédérale sur la protection des données (LPD)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 16 relatif à la communication de données personnelles à l’étranger.
