---
title: "Tailscale : comparaison entre nœud de sortie et routes de sous-réseau, et leur fonctionnement technique"
navTitle: "Nœud de sortie vs sous-réseau"
description: "Le nœud de sortie et le routeur de sous-réseau sont deux modes de fonctionnement apparentés, mais distincts dans Tailscale. Un routeur de sous-réseau ouvre de manière ciblée certaines plages d’adresses IP, tandis qu’un nœud de sortie achemine l’ensemble du trafic Internet à travers lui. Découvrez la différence en pratique, comment Tailscale met cela en œuvre via WireGuard, l’autorisation des routes et le SNAT, ainsi que les limites de chaque variante."
date: "2026-09-02"
kategorie: "Réseau et VPN"
timeToRead: "11 min de lecture"
themen:
  - tailscale
produkte:
  - "tailscale"
protokolle:
  - "tcp"
  - "haertung"
slug: "tailscale-comparaison-entre-les-n-uds-de-sortie-et-les-routes-de-sous-reseau-et-leur"
translationId: "article-c26cca4d635b9a04"
translationOf: tailscale-exit-node-subnet-routes
translationSourceHash: cefcefd45f617dfb668890986aaad256dbeca2b84d7c3472cd9eb5d6bb049911
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T09:52:41.849Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/tailscale-comparaison-entre-les-n-uds-de-sortie-et-les-routes-de-sous-reseau-et-leur
---

# Tailscale : comparaison entre nœud de sortie et routes de sous-réseau, et leur fonctionnement technique

Un nœud Tailscale ne représente d’abord que lui-même : il est accessible via son adresse Tailscale, et rien d’autre. Pour qu’un nœud donne à d’autres appareils accès à davantage que lui-même, il existe deux modes de fonctionnement souvent confondus : le **routeur de sous-réseau** et le **nœud de sortie**. Tous deux étendent la portée d’un nœud, mais dans des directions différentes. Comprendre la différence permet de choisir la variante adaptée et d’éviter d’acheminer par erreur tout le trafic via un ordinateur tiers.

En bref : un routeur de sous-réseau ouvre **de manière ciblée certaines plages d’adresses IP** derrière le nœud, par exemple le réseau local avec un NAS et une imprimante. Un nœud de sortie achemine **l’ensemble du trafic Internet** d’un appareil à travers lui, comme un VPN classique à tunnel complet. Techniquement, les deux reposent sur le même mécanisme : l’annonce de routes. Le nœud de sortie est en substance un cas particulier du routeur de sous-réseau, dans lequel la route par défaut est annoncée.

## Routeur de sous-réseau : accès ciblé à un réseau

Un routeur de sous-réseau annonce une ou plusieurs plages d’adresses IP auxquelles il peut accéder sur le réseau local. Les autres appareils du tailnet qui acceptent ces routes peuvent ainsi atteindre les appareils de la plage annoncée, même si Tailscale n’y est pas installé. C’est la solution pour rendre accessible un NAS, une imprimante ou une interface d’administration sans installer un client VPN sur chaque appareil.

La plage est annoncée sur le nœud routeur :

```powershell
tailscale set --advertise-routes=192.168.1.0/24
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `--advertise-routes=<CIDR>` | Annonce une ou plusieurs plages d’adresses IP (séparées par des virgules) que ce nœud achemine |
| `--snat-subnet-routes=false` | Achemine sans NAT source afin que les appareils cibles voient la véritable adresse source Tailscale ; nécessite une route de retour sur le réseau local |
| `--advertise-exit-node` | Forme courte qui annonce `0.0.0.0/0` et `::/0`, proposant ainsi le nœud comme nœud de sortie |

</details>

Le trafic ne circule qu’après **autorisation** de la route dans l’administration Tailscale. La simple annonce ne suffit pas ; c’est l’erreur la plus fréquente : la route n’apparaît dans la table de routage des appareils qui l’acceptent qu’après son autorisation.

## Nœud de sortie : tout le trafic via un nœud

Un nœud de sortie annonce la route par défaut (`0.0.0.0/0` et `::/0`). Lorsqu’un appareil sélectionne ce nœud de sortie, **tout** son trafic Internet sortant transite par ce nœud, et non uniquement le trafic destiné à un réseau précis. Cela est utile pour accéder à Internet via un emplacement disposant d’une adresse IP fixe ou pour diriger le trafic, sur un réseau non sûr, vers une sortie de confiance.

La différence avec la route de sous-réseau réside dans la sélection côté client : une route de sous-réseau est utilisée automatiquement dès que l’appareil accepte la route et contacte une destination de cette plage. En revanche, un nœud de sortie doit être sélectionné activement, et s’applique alors à tout le trafic :

```powershell
tailscale set --exit-node=100.100.10.10 --exit-node-allow-lan-access
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `--exit-node=<IP oder Name>` | Sélectionne un nœud de sortie ; vide (`--exit-node=`) le désactive à nouveau |
| `--exit-node-allow-lan-access` | Permet l’accès au réseau local propre malgré un nœud de sortie actif |

</details>

C’est précisément pourquoi, dans le cadre du support, il aurait été erroné d’activer le nœud de sortie pour accéder à un seul NAS : cela aurait redirigé tout le trafic de l’utilisateur via l’ordinateur tiers, au lieu de n’ouvrir qu’une seule plage.

## Comparaison

| Caractéristique | Routeur de sous-réseau | Nœud de sortie |
|---|---|---|
| Route annoncée | Plages ciblées, par ex. `192.168.1.0/24` | Route par défaut `0.0.0.0/0`, `::/0` |
| Utilisation côté client | Automatique pour les destinations de la plage | Doit être activement sélectionné comme nœud de sortie |
| Portée | Uniquement les réseaux annoncés | Tout le trafic Internet |
| Autorisation dans l’administration | Par sous-réseau | Séparément comme nœud de sortie |
| Usage typique | Rendre les services internes accessibles | Acheminer le trafic sortant via un emplacement |

## Comment Tailscale le met en œuvre techniquement

Les deux modes de fonctionnement reposent sur la même base. Il est utile de distinguer les couches.

**Plan de données via WireGuard.** Chaque nœud possède une paire de clés WireGuard. Le trafic effectif entre deux nœuds circule directement sous forme de paquets WireGuard chiffrés via UDP, si possible en pair à pair après traversée NAT, ou sinon via un serveur relais DERP comme solution de repli. Tailscale ne réinvente pas le chiffrement, mais utilise WireGuard comme transport.

**Plan de contrôle via le serveur de coordination.** Un serveur de coordination central distribue les clés publiques et une carte réseau qui indique quel nœud possède quelles adresses et quelles routes. Le serveur de coordination voit les métadonnées (qui est autorisé à parler avec qui, quelles routes sont autorisées), mais pas le contenu des paquets WireGuard. Lorsque vous annoncez une route, le nœud le signale au plan de contrôle ; ce n’est qu’après autorisation que la route devient partie intégrante de la carte réseau reçue par tous les nœuds.

**Sur le nœud routeur.** Pour qu’un nœud achemine le trafic destiné à d’autres appareils, le transfert IP doit être activé et les paquets doivent pouvoir transiter entre l’interface Tailscale et le réseau local. Par défaut, Tailscale masque le trafic transféré avec du NAT source (SNAT) : les appareils cibles du réseau local voient l’adresse locale du nœud routeur comme expéditeur, et non l’adresse Tailscale de l’appareil qui accède au service. C’est le cas simple, car les paquets de réponse reviennent alors automatiquement vers le routeur. Si vous désactivez le SNAT, les appareils cibles voient la véritable adresse source Tailscale, mais le réseau local doit alors savoir comment router la plage Tailscale en retour vers le routeur.

**Côté client.** Un appareil n’utilise les routes d’autres nœuds que s’il les accepte. Sur les clients graphiques pour Windows et macOS, l’acceptation des routes de sous-réseau est activée par défaut ; sous Linux, elle s’active avec `--accept-routes`. Lorsqu’un client accepte une route, il l’ajoute à sa table de routage et la fait pointer vers l’interface Tailscale. Les paquets destinés à une cible de cette plage sont alors encapsulés dans WireGuard et envoyés au nœud routeur. Pour le nœud de sortie, le mécanisme est identique, mais la route par défaut pointe alors vers le nœud de sortie, raison pour laquelle tout le trafic passe par lui.

**L’autorisation.** Le fait que les routes ne prennent effet qu’après autorisation est une fonctionnalité de sécurité, et non un détour : un nœud quelconque ne doit pas pouvoir attirer sans demande le trafic de réseaux entiers. L’autorisation peut être accordée manuellement dans l’administration ou automatiquement via `autoApprovers` dans les règles de contrôle d’accès (ACL). Les nœuds de sortie et les routes de sous-réseau sont autorisés séparément.

## Limites

Les deux variantes présentent des limites qui influencent le choix :

- **Le nœud routeur est un goulot d’étranglement et un point de défaillance unique.** Tout le trafic du réseau annoncé passe par ce seul nœud, par son chiffrement WireGuard et sa connexion. Pour assurer la disponibilité, plusieurs nœuds peuvent annoncer la même route ; Tailscale en utilise alors un et bascule en cas de panne.
- **Le SNAT masque la source.** Avec le NAT source activé par défaut, tous les accès apparaissent sous l’adresse du nœud routeur. Pour la journalisation ou les règles d’accès sur les appareils cibles nécessitant la véritable source, vous devez désactiver le SNAT et configurer la route de retour sur le réseau local.
- **Un nœud de sortie achemine réellement tout.** Tout le trafic passe par le nœud, avec les conséquences correspondantes en matière de débit, de latence et de confidentialité. L’opérateur du nœud de sortie voit le trafic à l’endroit où il quitte le tailnet. N’utilisez comme nœud de sortie que des nœuds auxquels vous faites confiance.
- **Les sous-réseaux qui se chevauchent posent problème.** Si deux emplacements annoncent la même plage privée, par exemple `192.168.1.0/24`, un client ne peut pas les distinguer. Tailscale propose pour cela une réécriture via IPv6 (`4via6`), qui rend les plages univoques.
- **L’expiration des clés bloque le transfert.** Si la clé du nœud routeur expire, tout le réseau situé derrière lui n’est plus accessible. Pour un nœud routeur permanent, désactivez l’expiration des clés dans l’administration.

Pour un accès ciblé aux services internes, le routeur de sous-réseau est presque toujours le bon choix : il n’ouvre que ce qui est nécessaire. Choisissez le nœud de sortie si vous souhaitez délibérément faire passer tout le trafic sortant par un emplacement donné.

## Historique des modifications

- **25 octobre 2026 :** liens des Sources mis à jour vers les nouvelles adresses de Tailscale.

## Sources

1.  [Tailscale : routeurs de sous-réseau](https://tailscale.com/docs/features/subnet-routers): annonce de routes, autorisation, comportement du SNAT et haute disponibilité avec plusieurs routeurs.

2.  [Tailscale : nœuds de sortie](https://tailscale.com/docs/features/exit-nodes): annonce de la route par défaut, sélection côté client et accès au réseau local propre.

3.  [Tailscale : fonctionnement de Tailscale](https://tailscale.com/blog/how-tailscale-works): interaction entre le plan de données WireGuard, le serveur de coordination et les relais DERP.

4.  [WireGuard : aperçu du protocole](https://www.wireguard.com/protocol/): fondement cryptographique du plan de données que Tailscale utilise comme transport.
