---
title: "Tailscale : comparaison entre les nœuds de sortie et les routes de sous-réseau, et leur fonctionnement technique"
navTitle: "Nœud de sortie ou sous-réseau"
description: "Dans Tailscale, les nœuds de sortie et les routeurs de sous-réseau sont deux modes de fonctionnement apparentés, mais différents. Un routeur de sous-réseau ouvre de manière ciblée certaines plages d’adresses IP, tandis qu’un nœud de sortie achemine l’ensemble du trafic Internet par son intermédiaire. Découvrez ce que cette différence implique en pratique, comment Tailscale l’implémente via WireGuard, l’approbation des routes et le SNAT, ainsi que les limites de chaque variante."
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
aiPrompt: |
  Du bist mein Netzwerkassistent. Erkläre mir den Unterschied zwischen einem Tailscale-Subnetz-Router und einem Exit-Node, wann ich welchen brauche, und wie Tailscale das technisch umsetzt (WireGuard-Data-Plane, Routen-Freigabe über den Coordination Server, IP-Weiterleitung und SNAT auf dem Router-Node). Hilf mir, die richtige Variante zu wählen und einzurichten.
translationOf: tailscale-exit-node-subnet-routes
url: https://rafaelpfister.ch/fr/blog/tailscale-comparaison-entre-les-n-uds-de-sortie-et-les-routes-de-sous-reseau-et-leur
translationSourceHash: f05a193f13dd2b8aba3c9d049ea1c0a1fcc25b12c420a1d520f99854b7883a79
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:00:07.791Z
translationReview: automatic
---

# Tailscale : comparaison entre les nœuds de sortie et les routes de sous-réseau, et leur fonctionnement technique

Un nœud Tailscale ne représente d’abord que lui-même : il est accessible via son adresse Tailscale, et rien d’autre. Pour qu’un nœud donne à d’autres appareils accès à davantage que lui-même, il existe deux modes de fonctionnement souvent confondus : le **routeur de sous-réseau** et le **nœud de sortie**. Tous deux étendent la portée d’un nœud, mais dans des directions différentes. Connaître la différence permet de choisir la variante adaptée et d’éviter de faire passer par inadvertance tout son trafic via un ordinateur tiers.

En bref : un routeur de sous-réseau ouvre **de manière ciblée certaines plages d’adresses IP** situées derrière le nœud, par exemple le réseau local avec un NAS et une imprimante. Un nœud de sortie fait passer **l’ensemble du trafic Internet** d’un appareil par son intermédiaire, comme un VPN classique en tunnel complet. Techniquement, les deux reposent sur le même mécanisme : l’annonce de routes. Le nœud de sortie est en substance un cas particulier du routeur de sous-réseau, dans lequel la route par défaut est annoncée.

## Routeur de sous-réseau : accès ciblé à un réseau

Un routeur de sous-réseau annonce une ou plusieurs plages d’adresses IP qu’il peut atteindre sur le réseau local. Les autres appareils du tailnet qui acceptent ces routes peuvent ainsi joindre les appareils de la plage annoncée, même si Tailscale n’y est pas installé. C’est la solution pour rendre accessible un NAS, une imprimante ou une interface d’administration sans installer de client VPN sur chaque appareil.

La plage est annoncée sur le nœud routeur :

```powershell
tailscale set --advertise-routes=192.168.1.0/24
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `--advertise-routes=<CIDR>` | Annonce une ou plusieurs plages d’adresses IP (séparées par des virgules) que ce nœud achemine |
| `--snat-subnet-routes=false` | Achemine sans NAT de source afin que les appareils de destination voient l’adresse source Tailscale réelle ; exige une route de retour sur le réseau local |
| `--advertise-exit-node` | Forme abrégée qui annonce `0.0.0.0/0` et `::/0`, proposant ainsi le nœud comme nœud de sortie |

</details>

Le trafic ne circule qu’après **approbation** de la route dans l’administration Tailscale. Il ne suffit pas de l’annoncer ; c’est l’erreur la plus fréquente : la route n’apparaît dans la table de routage des appareils qui l’acceptent qu’après son approbation.

## Nœud de sortie : tout le trafic via un nœud

Un nœud de sortie annonce la route par défaut (`0.0.0.0/0` et `::/0`). Lorsqu’un appareil sélectionne ce nœud de sortie, **tout** son trafic Internet sortant passe par ce nœud, et non seulement le trafic vers un réseau déterminé. C’est utile pour accéder à Internet depuis un emplacement disposant d’une adresse IP fixe ou pour diriger le trafic, depuis un réseau non fiable, vers une sortie de confiance.

La différence avec une route de sous-réseau réside dans la sélection côté client : une route de sous-réseau est utilisée automatiquement dès que l’appareil accepte la route et contacte une destination de cette plage. Un nœud de sortie, en revanche, doit être sélectionné activement ; il s’applique alors à l’ensemble du trafic :

```powershell
tailscale set --exit-node=100.100.10.10 --exit-node-allow-lan-access
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `--exit-node=<IP oder Name>` | Sélectionne un nœud de sortie ; une valeur vide (`--exit-node=`) le désactive à nouveau |
| `--exit-node-allow-lan-access` | Conserve l’accès au réseau local de l’appareil malgré un nœud de sortie actif |

</details>

C’est précisément pour cette raison qu’au quotidien du support, il était erroné de cocher le nœud de sortie pour accéder à un seul NAS : cela aurait redirigé tout le trafic de l’utilisateur via l’ordinateur tiers au lieu d’ouvrir uniquement la plage concernée.

## Comparaison

| Caractéristique | Routeur de sous-réseau | Nœud de sortie |
|---|---|---|
| Route annoncée | Plages ciblées, p. ex. `192.168.1.0/24` | Route par défaut `0.0.0.0/0`, `::/0` |
| Utilisation côté client | Automatique pour les destinations de la plage | Doit être sélectionné activement comme nœud de sortie |
| Étendue | Uniquement les réseaux annoncés | Tout le trafic Internet |
| Approbation dans l’administration | Par sous-réseau | Séparément comme nœud de sortie |
| Usage typique | Rendre accessibles des services internes | Faire passer le trafic sortant par un emplacement |

## Comment Tailscale l’implémente techniquement

Les deux modes reposent sur la même base. Il est utile de distinguer les différentes couches.

**Plan de données via WireGuard.** Chaque nœud possède une paire de clés WireGuard. Le trafic réel entre deux nœuds circule directement sous forme de paquets WireGuard chiffrés via UDP, si possible de pair à pair après traversée de NAT, sinon via un serveur relais DERP comme solution de repli. Tailscale ne réinvente pas le chiffrement, mais utilise WireGuard comme transport.

**Plan de contrôle via le serveur de coordination.** Un serveur de coordination central distribue les clés publiques et une carte du réseau indiquant quel nœud possède quelles adresses et routes. Le serveur de coordination voit les métadonnées (qui est autorisé à communiquer avec qui, quelles routes sont approuvées), mais pas le contenu des paquets WireGuard. Lorsque vous annoncez une route, le nœud le signale au plan de contrôle ; ce n’est qu’après approbation que la route devient partie intégrante de la carte du réseau reçue par tous les nœuds.

**Sur le nœud routeur.** Pour qu’un nœud achemine le trafic d’autres appareils, le transfert IP doit être activé et il doit relayer les paquets entre l’interface Tailscale et le réseau local. Par défaut, Tailscale masque le trafic relayé par NAT de source (SNAT) : les appareils de destination du réseau local voient comme expéditeur l’adresse locale du nœud routeur, et non l’adresse Tailscale de l’appareil accédant à la ressource. C’est le cas simple, car les paquets de réponse retrouvent ainsi automatiquement le chemin vers le routeur. Si vous désactivez le SNAT, les appareils de destination voient l’adresse source Tailscale réelle, mais le réseau local doit alors savoir comment renvoyer la plage Tailscale vers le routeur.

**Côté client.** Un appareil n’utilise des routes tierces que s’il les accepte. Sur les clients graphiques pour Windows et macOS, l’acceptation des routes de sous-réseau est activée par défaut ; sous Linux, elle s’active avec `--accept-routes`. Lorsque le client accepte une route, il l’inscrit dans sa table de routage et la dirige vers l’interface Tailscale. Les paquets destinés à une cible de cette plage sont alors encapsulés dans WireGuard et envoyés au nœud routeur. Avec un nœud de sortie, le mécanisme est identique, sauf que la route par défaut pointe ici vers le nœud de sortie, raison pour laquelle tout le trafic y transite.

**L’approbation.** Le fait que les routes ne prennent effet qu’après approbation est une fonction de sécurité, et non un détour : un nœud quelconque ne doit pas pouvoir attirer sans autorisation le trafic de réseaux entiers. L’approbation peut être effectuée manuellement dans l’administration ou automatiquement via `autoApprovers` dans les règles de contrôle d’accès (ACL). Les nœuds de sortie et les routes de sous-réseau sont approuvés séparément.

## Limites

Les deux variantes présentent des limites qui influencent le choix :

- **Le nœud routeur constitue un goulot d’étranglement et un point unique de défaillance.** Tout le trafic destiné au réseau annoncé passe par ce seul nœud, son chiffrement WireGuard et sa connexion. Pour assurer la disponibilité, plusieurs nœuds peuvent annoncer la même route ; Tailscale en utilise alors un et bascule en cas de panne.
- **Le SNAT masque la source.** Avec le NAT de source activé par défaut, tous les accès apparaissent sous l’adresse du nœud routeur. Pour la journalisation ou les règles d’accès sur les appareils de destination qui nécessitent la source réelle, vous devez désactiver le SNAT et configurer la route de retour sur le réseau local.
- **Un nœud de sortie achemine réellement tout.** Tout le trafic passe par le nœud, avec les conséquences correspondantes en matière de débit, de latence et de confidentialité. L’exploitant du nœud de sortie voit le trafic à l’endroit où il quitte le tailnet. Utilisez comme nœud de sortie uniquement des nœuds auxquels vous faites confiance.
- **Les sous-réseaux qui se chevauchent posent problème.** Si deux emplacements annoncent la même plage privée, par exemple `192.168.1.0/24`, un client ne peut pas les distinguer. Tailscale propose pour cela une réécriture via IPv6 (`4via6`), qui rend les plages univoques.
- **L’expiration des clés interrompt le relais.** Si la clé du nœud routeur expire, tout le réseau situé derrière lui n’est plus accessible. Pour un nœud routeur permanent, désactivez l’expiration des clés dans l’administration.

Pour l’accès ciblé à des services internes, le routeur de sous-réseau est presque toujours le bon choix : il n’ouvre que ce qui est nécessaire. Utilisez le nœud de sortie lorsque vous souhaitez délibérément faire passer tout le trafic sortant par un emplacement donné.

## Sources

1.  [Tailscale: Subnet routers](https://tailscale.com/kb/1019/subnets): annonce des routes, approbation, comportement du SNAT et haute disponibilité avec plusieurs routeurs.

2.  [Tailscale: Exit nodes](https://tailscale.com/kb/1103/exit-nodes): annonce de la route par défaut, sélection côté client et accès au propre réseau local.

3.  [Tailscale: How Tailscale works](https://tailscale.com/blog/how-tailscale-works): interaction entre le plan de données WireGuard, le serveur de coordination et les relais DERP.

4.  [WireGuard: Vue d’ensemble du protocole](https://www.wireguard.com/protocol/): la base cryptographique du plan de données que Tailscale utilise comme transport.
