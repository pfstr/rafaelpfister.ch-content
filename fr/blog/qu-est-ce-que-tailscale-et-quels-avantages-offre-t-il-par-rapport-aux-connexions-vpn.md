---
title: "Qu’est-ce que Tailscale et quels avantages offre-t-il par rapport aux connexions VPN traditionnelles ?"
navTitle: "Tailscale vs. VPN"
description: "Tailscale s’appuie sur WireGuard pour créer un VPN maillé dans lequel les appareils se connectent directement au lieu de passer par un concentrateur VPN central. Découvrez comment les serveurs de coordination, la traversée NAT et les relais DERP interagissent, quels sont les avantages par rapport aux VPN IPsec et SSL, et quelles dépendances et limites vous devriez connaître avant de le déployer."
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
aiPrompt: |
  Du bist mein Netzwerk-Assistent. Hilf mir einzuschätzen, ob Tailscale unser bestehendes VPN ganz oder teilweise ersetzen kann: Ist-Zustand aufnehmen (VPN-Gateway, Benutzer, Standorte, erreichbare Netze), Zugriffsregeln nach dem Prinzip der minimalen Rechte als Tailscale-Policy entwerfen, Subnet Router und Exit Nodes planen und Abhängigkeiten wie Identity Provider, Datenschutz nach revDSG und Koexistenz mit anderen VPN-Clients prüfen.
translationOf: tailscale-vorteile-vpn
url: https://rafaelpfister.ch/fr/blog/qu-est-ce-que-tailscale-et-quels-avantages-offre-t-il-par-rapport-aux-connexions-vpn
translationSourceHash: 965ff1000bee1a9e9899d6ffd750d28abbea09dd50896551eae369b7bf20f162
translationModel: gpt-5.6-terra
translatedAt: 2026-10-02T10:04:38.965Z
translationReview: required
---

# Qu’est-ce que Tailscale et quels avantages offre-t-il par rapport aux connexions VPN traditionnelles ?

Tailscale est un service VPN qui relie des appareils à un réseau privé, appelé Tailnet. Techniquement, il repose sur WireGuard. La différence avec un VPN d’entreprise classique réside dans la topologie : les appareils établissent directement entre eux leurs tunnels chiffrés (maillage). Il n’y a pas de concentrateur VPN central par lequel transite l’ensemble du trafic. Seule l’administration reste centralisée : un serveur de coordination distribue les clés publiques, les adresses et les règles d’accès, mais ne voit lui-même aucune donnée utile.

Pour démarrer, il suffit d’un compte auprès d’un fournisseur d’identité (Microsoft, Google, GitHub, Apple ou un fournisseur OIDC) et du client sur chaque appareil. En règle générale, aucune ouverture de port dans le pare-feu n’est nécessaire. Tailscale est ainsi répandu aussi bien pour les réseaux domestiques que pour l’accès à distance aux serveurs et aux systèmes clients.

## Fonctionnement d’un VPN traditionnel

Un VPN d’accès à distance classique fonctionne selon le principe hub-and-spoke. Une passerelle VPN (pare-feu ou appliance), accessible depuis Internet, se trouve à la périphérie du réseau d’entreprise. Le client sur l’ordinateur portable établit un tunnel vers cette passerelle ; IPsec/IKEv2 (UDP 500 et 4500), les variantes de VPN SSL via TCP 443 ou OpenVPN sont courants. Après l’authentification, l’appareil reçoit une adresse d’un pool et des routes vers les réseaux internes.

Ce modèle a fait ses preuves pendant des décennies, mais il présente des caractéristiques structurelles qui occasionnent des efforts dans l’exploitation actuelle :

- **Passerelle accessible publiquement :** le concentrateur VPN doit être accessible depuis Internet et constitue donc une cible privilégiée. Ces dernières années, des vulnérabilités dans des appliances VPN ont été activement exploitées à plusieurs reprises ; en janvier 2024, l’agence américaine CISA a même ordonné, par l’Emergency Directive 24-01, la déconnexion du réseau des passerelles Ivanti concernées.
- **Point de défaillance unique et goulot d’étranglement :** l’ensemble du trafic transite par la passerelle. En cas de panne ou de saturation de la bande passante, tous les utilisateurs sont affectés simultanément.
- **Détours (hairpinning) :** si deux collaborateurs en télétravail travaillent sur le même serveur dans le cloud, le trafic passe d’abord par le centre de données avant d’en ressortir.
- **Droits d’accès trop larges :** une fois la connexion établie, un segment de réseau entier est souvent accessible à l’appareil. Des règles fines par utilisateur et par service sont possibles, mais rarement maintenues de manière cohérente.
- **Effort site à site :** chaque site supplémentaire nécessite son propre tunnel avec des paramètres harmonisés (propositions de phase 1/phase 2, clés prépartagées ou certificats, adresses IP publiques fixes).

## Architecture de Tailscale

Tailscale sépare le plan de contrôle du plan de données. Le plan de contrôle est le serveur de coordination, que Tailscale exploite comme service cloud. Le plan de données correspond aux tunnels WireGuard entre les appareils.

| Composant | Rôle |
|---|---|
| Client (`tailscaled`) | Génère la paire de clés localement, établit des tunnels WireGuard vers les autres nœuds, applique localement les règles d’accès |
| Serveur de coordination | Authentifie les appareils via le fournisseur d’identité, distribue les clés publiques, les adresses, les paramètres DNS et la politique à tous les nœuds |
| Relais DERP | Transmettent les paquets chiffrés lorsqu’une connexion directe ne peut pas être établie |
| Relais pairs | Appareils propres au sein du Tailnet servant de relais avec un débit supérieur ; ils sont privilégiés par rapport à DERP |

La clé privée d’un appareil ne quitte jamais l’appareil. Le serveur de coordination ne connaît que les clés publiques et ne peut donc pas déchiffrer le trafic. Chaque appareil reçoit une adresse fixe dans la plage `100.64.0.0/10` (l’espace d’adressage destiné au NAT de niveau opérateur), ainsi qu’une adresse IPv6 dans `fd7a:115c:a1e0::/48`. Grâce à MagicDNS, les appareils sont également accessibles sous leur nom, par exemple `nas` ou `nas.tailnet-name.ts.net`.

### Traversée NAT : pourquoi aucune ouverture de port n’est nécessaire

La plupart des appareils se trouvent derrière un routeur NAT ou un pare-feu et ne sont pas directement accessibles depuis l’extérieur. Tailscale résout ce problème par la traversée NAT : les deux parties déterminent via STUN leur adresse publique et le port attribué, échangent ces informations via le serveur de coordination et s’envoient simultanément des paquets UDP. Les paquets sortants créent une entrée d’état sur les deux pare-feu, par laquelle les paquets de l’autre partie peuvent ensuite entrer (UDP Hole Punching).

Si cela échoue, par exemple avec des pare-feu restrictifs qui bloquent les UDP sortants ou avec certaines formes de NAT de niveau opérateur, le trafic passe via un relais DERP en HTTPS. Là aussi, il reste chiffré de bout en bout avec WireGuard ; le relais ne voit que des paquets chiffrés. Le prix à payer est une latence plus élevée et un débit moindre. Les relais pairs réduisent cet inconvénient en confiant le relais à un appareil propre disposant d’une bonne connectivité.

## Les avantages par rapport à un VPN traditionnel

| Critère | VPN traditionnel | Tailscale |
|---|---|---|
| Topologie | Hub-and-spoke via une passerelle centrale | Maillage, connexions directes entre les appareils |
| Ports entrants | La passerelle doit être accessible depuis Internet | Aucune ouverture de port entrant nécessaire |
| Authentification | Comptes locaux, RADIUS, certificats, souvent MFA séparée | Connexion via le fournisseur d’identité existant, y compris sa MFA |
| Droits d’accès | Souvent par segment réseau | Par utilisateur, groupe, appareil et port dans une politique centrale |
| Nouveau site | Tunnel site à site avec paramètres harmonisés | Installer un client ou un routeur de sous-réseau |
| Panne du centre | Plus aucun accès pour tous | Les connexions existantes continuent ; les nouveaux appareils et les modifications de politique attendent |
| Protocole | IPsec, VPN SSL, OpenVPN | WireGuard |

### Surface d’attaque réduite

Comme les clients établissent leurs connexions de manière sortante, aucun appareil n’a besoin d’un port ouvert sur Internet. Un serveur qui ne doit être accessible que via le Tailnet peut lier ses services exclusivement à l’interface Tailscale. Il devient alors invisible aux scanners de ports sur Internet. WireGuard lui-même, avec environ 4 000 lignes de code noyau, est nettement plus petit que les implémentations IPsec ou VPN SSL typiques et utilise un ensemble fixe de procédés modernes (Curve25519, ChaCha20-Poly1305, BLAKE2s). Il n’existe pas de négociation de suites de chiffrement, comme celle qui entraîne régulièrement des erreurs de configuration avec IPsec.

### L’identité plutôt que l’adresse réseau

Chaque appareil du Tailnet est associé à un utilisateur ou à une étiquette. La connexion s’effectue via le fournisseur d’identité que vous utilisez déjà ; la MFA et l’accès conditionnel de Microsoft Entra ID s’appliquent donc aussi à l’accès réseau. De nouveaux appareils ne peuvent rejoindre le Tailnet qu’après une authentification réussie auprès du fournisseur d’identité, et chaque clé d’appareil expire par défaut après 180 jours. Lorsqu’un collaborateur quitte l’entreprise, vous bloquez son compte auprès du fournisseur d’identité ; avec le provisionnement SCIM (à partir de l’offre Standard), l’utilisateur est automatiquement désactivé dans le Tailnet et ses appareils perdent l’accès.

### Règles d’accès selon le principe du moindre privilège

Par défaut, dans un nouveau Tailnet, chaque appareil peut joindre tous les autres. Pour une utilisation en production, vous définissez les droits d’accès dans un fichier de politique central (HuJSON). L’exemple suivant autorise le groupe des administrateurs à accéder en SSH et HTTPS à tous les serveurs portant l’étiquette `tag:server`, et n’autorise les autres utilisateurs à accéder en HTTPS qu’au serveur intranet :

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
| `tagOwners` | Définit qui peut attribuer une étiquette aux appareils ; les appareils étiquetés n’appartiennent pas à un utilisateur, mais à l’étiquette |
| `grants` | Liste des connexions autorisées ; tout ce qui n’est pas expressément autorisé est bloqué |
| `src` | Source de la connexion : utilisateur, groupe, étiquette ou `autogroup:member` (tous les utilisateurs du Tailnet) |
| `dst` | Destination de la connexion : étiquette, alias d’hôte, appareil ou sous-réseau |
| `ip` | Protocoles et ports autorisés, par exemple `tcp:22` ou `*` pour tout |
| `hosts` | Noms d’alias pour les adresses ou sous-réseaux du Tailnet, utilisés dans les règles |

</details>

Chaque client applique les règles localement. Un paquet non autorisé par la politique est rejeté dès l’appareil de destination. Le serveur de coordination ne fait que distribuer la politique.

### Connexions directes plutôt que détours

Puisque les tunnels existent directement entre les appareils, le trafic emprunte le chemin le plus court. Deux appareils dans le même bureau communiquent localement, et un ordinateur portable en télétravail atteint directement un serveur cloud. Cela réduit la latence et soulage la connexion Internet du site principal.

### Connecter les réseaux existants

Tous les appareils ne peuvent pas exécuter un client Tailscale, par exemple les imprimantes, les systèmes NAS anciens ou les systèmes de contrôle industriels. Dans ces cas, un routeur de sous-réseau assume le rôle de passerelle : un serveur Linux dans le réseau cible annonce le sous-réseau local dans le Tailnet (advertise), et les appareils autorisés y accèdent par son intermédiaire.

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
| `net.ipv4.ip_forward = 1` | Autorise le noyau Linux à transférer des paquets entre les interfaces ; sans ce paramètre, aucun routeur de sous-réseau ne fonctionne |
| `sysctl -p <datei>` | Charge immédiatement le paramètre, sans redémarrage |
| `tailscale up` | Connecte l’appareil au Tailnet ; lors du premier appel, un lien de connexion apparaît |
| `--advertise-routes=<subnetz>` | Annonce le sous-réseau indiqué dans le Tailnet ; plusieurs sous-réseaux sont séparés par des virgules |
| `--advertise-tags=<tag>` | Attribue à l’appareil une étiquette à laquelle se réfèrent les règles d’accès |

</details>

La route annoncée doit ensuite être approuvée dans la console d’administration, sauf si une approbation automatique (`autoApprovers`) est configurée dans la politique. De même, un appareil peut fonctionner comme nœud de sortie avec `--advertise-exit-node`. Les clients qui sélectionnent ce nœud de sortie y acheminent alors l’ensemble de leur trafic Internet, ce qui correspond au mode tunnel complet d’un VPN classique.

### Moins de charge d’exploitation

Le serveur de coordination gère la rotation des clés, l’attribution des adresses, le DNS et le routage. Un nouveau site a besoin d’un routeur de sous-réseau avec accès Internet, mais ni d’une adresse IP publique fixe ni d’une coordination des paramètres IPsec avec la partie opposée. Pour l’accès à distance aux serveurs, Tailscale SSH (connexion par identité Tailnet sans clés SSH distribuées) ainsi que `tailscale serve` (partage d’un service web local dans le Tailnet) sont également disponibles.

## Limites et dépendances

Tailscale remplace la passerelle VPN classique, mais transfère une partie de la responsabilité vers un service externe. Vous devriez examiner les points suivants avant toute mise en place.

| Sujet | À prendre en compte |
|---|---|
| Dépendance au fournisseur | Le plan de contrôle est un service cloud de Tailscale Inc. En cas de panne, les connexions existantes continuent de fonctionner, mais les nouveaux appareils, connexions et modifications de politique sont impossibles jusqu’au rétablissement |
| Métadonnées | Les noms des appareils, les adresses Tailnet, les adresses IP publiques, les comptes utilisateurs et les horaires de connexion sont traités par le fournisseur ; pas les données utiles |
| Confiance dans la distribution des clés | Le serveur de coordination détermine quelles clés publiques un appareil accepte. Tailnet Lock exige en plus une signature par des appareils propres de confiance |
| Code source | Le cœur du client est open source (BSD-3-Clause), mais pas le serveur de coordination. Headscale est une alternative open source auto-hébergée aux fonctionnalités réduites |
| Conflits d’adresses | `100.64.0.0/10` est également utilisé par certains fournisseurs pour le NAT de niveau opérateur et par d’autres produits VPN ; les chevauchements entraînent des problèmes de routage |
| Coexistence avec d’autres clients VPN | Un second client VPN avec tunnel complet peut rediriger le trafic Tailscale. La plage `100.64.0.0/10` et le service Tailscale doivent alors être exclus du tunnel |
| Performances via les relais | Si aucune connexion directe ne peut être établie, le débit baisse sensiblement via DERP ; `tailscale netcheck` indique si les UDP sortants fonctionnent |
| Ne remplace pas le filtrage web | Tailscale régule l’accès aux ressources internes. Il ne prend pas en charge le filtrage de contenu ni l’inspection du trafic Internet |

Pour les entreprises en Suisse, la loi fédérale révisée sur la protection des données (revLPD) s’applique. Comme des données personnelles des utilisateurs (comptes, appareils, métadonnées de connexion) sont traitées par le fournisseur, inscrivez Tailscale dans le registre des activités de traitement si votre entreprise doit en tenir un (à partir de 250 collaborateurs ou en cas de traitements présentant un risque élevé). Vérifiez le contrat de sous-traitance (Data Processing Addendum) du fournisseur ainsi que la base juridique de la communication à l’étranger conformément à l’art. 16 revLPD, par exemple une certification au titre du Swiss-U.S. Data Privacy Framework ou des clauses contractuelles types. Si tout traitement de données chez le fournisseur est exclu, Headscale reste une solution de plan de contrôle exploitée en interne.

> **Note UE :** pour les succursales dans l’UE, le RGPD et ses règles relatives aux transferts vers des pays tiers (art. 44 et suivants du RGPD) s’appliquent. L’examen est semblable sur le fond ; le cadre pertinent est alors le EU-U.S. Data Privacy Framework.

## Coûts

L’offre Personal est gratuite et comprend jusqu’à six utilisateurs, un nombre illimité d’appareils utilisateurs et 50 appareils étiquetés (état en octobre 2026). Pour un usage professionnel, l’offre Standard coûte 8 USD et l’offre Premium 18 USD par utilisateur et par mois. Premium ajoute notamment les journaux Network Flow Logs, le streaming de journaux et des options étendues pour Tailscale SSH. Enterprise est proposé sur mesure.

## À qui Tailscale convient-il ?

Tailscale convient particulièrement lorsque les utilisateurs et les ressources sont répartis : télétravail, serveurs cloud auprès de plusieurs fournisseurs, petits sites distants sans adresse IP fixe ou maintenance à distance chez les clients. Pour les petites et moyennes entreprises, il peut remplacer complètement une passerelle VPN. Dans les environnements plus grands, il fonctionne souvent en parallèle du VPN existant, par exemple pour l’accès d’administration aux serveurs, où les règles d’accès granulaires apportent le plus grand bénéfice.

Il est moins adapté lorsque les exigences imposent une infrastructure entièrement exploitée en interne et que Headscale ne couvre pas l’étendue fonctionnelle requise, ou lorsque l’ensemble du trafic Internet doit être filtré de manière centralisée. Dans ce cas, une passerelle web sécurisée reste nécessaire en complément de Tailscale.

Pour le tester, il suffit d’installer le client sur deux appareils et de se connecter avec le même compte. Avec `tailscale status`, vous voyez ensuite tous les appareils du Tailnet ; avec `tailscale ping <gerät>`, vous vérifiez si une connexion directe ou un relais est utilisé.

## Sources

1.  [Tailscale: How Tailscale works](https://tailscale.com/blog/how-tailscale-works): architecture avec serveur de coordination, maillage WireGuard et application locale des règles.

2.  [Tailscale: How NAT traversal works](https://tailscale.com/blog/how-nat-traversal-works): explication détaillée de STUN, de l’UDP Hole Punching et des cas dans lesquels un relais est nécessaire.

3.  [Tailscale Docs: DERP servers](https://tailscale.com/kb/1232/derp-servers): fonctionnement et chiffrement des serveurs relais.

4.  [Tailscale Docs: Tailscale Peer Relays](https://tailscale.com/kb/1591/peer-relays): appareils propres utilisés comme relais, prioritaires par rapport à DERP.

5.  [Tailscale Docs: Grants](https://tailscale.com/kb/1324/grants): syntaxe des règles d’accès dans le fichier de politique.

6.  [Tailscale Docs: Subnet routers](https://tailscale.com/kb/1019/subnets): configuration des routeurs de sous-réseau, y compris le transfert IP et l’approbation des routes.

7.  [Tailscale Docs: Exit nodes](https://tailscale.com/kb/1103/exit-nodes): acheminer l’ensemble du trafic Internet via un appareil du Tailnet.

8.  [Tailscale Docs: Tailnet Lock](https://tailscale.com/kb/1226/tailnet-lock): signature de nouveaux appareils par des nœuds propres de confiance.

9.  [Tailscale: Pricing](https://tailscale.com/pricing): tarifs et limites, consultés le 1er octobre 2026.

10.  [WireGuard: Next Generation Kernel Network Tunnel (Whitepaper)](https://www.wireguard.com/papers/wireguard.pdf): conception du protocole et procédés cryptographiques utilisés.

11.  [GitHub: tailscale/tailscale](https://github.com/tailscale/tailscale): code source du client sous licence BSD-3-Clause.

12.  [GitHub: juanfont/headscale](https://github.com/juanfont/headscale): implémentation open source du serveur de coordination pour une exploitation autonome.

13.  [CISA: Emergency Directive 24-01](https://www.cisa.gov/news-events/directives/ed-24-01-mitigate-ivanti-connect-secure-and-ivanti-policy-secure-vulnerabilities): instruction de déconnecter les passerelles VPN Ivanti vulnérables en janvier 2024.

14.  [Fedlex: Loi fédérale sur la protection des données (LPD)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 16 relatif à la communication de données personnelles à l’étranger.
