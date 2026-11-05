---
title: "Multipath TCP (MPTCP) sous Linux : fonctionnement, historique et vérification avant désactivation"
navTitle: "MPTCP sous Linux"
description: "Multipath TCP répartit une connexion TCP sur plusieurs chemins réseau. Depuis Go 1.24, de nombreux services Go sous Linux écoutent via MPTCP sans que personne ne l’ait configuré. Avec CVE-2026-98432, cela est devenu pertinent. Cet article explique le protocole, son historique et comment vérifier si vous pouvez désactiver MPTCP."
date: "2026-11-02"
kategorie: "Noyau Linux"
timeToRead: "10 min de lecture"
themen:
  - linux-kernel
  - sicherheitsluecken
produkte:
  - "linux-kernel"
  - "sicherheitsluecken"
protokolle:
  - "tcp"
  - "haertung"
hauptthema: "linux-kernel"
slug: "multipath-tcp-mptcp-sous-linux-fonctionnement-historique-et-verification-avant-desactivation"
translationId: "article-1caab37cb0d7fce7"
translationOf: multipath-tcp-mptcp-erklaert
url: https://rafaelpfister.ch/fr/blog/multipath-tcp-mptcp-sous-linux-fonctionnement-historique-et-verification-avant-desactivation
translationSourceHash: 0c0b43321d6db2e6da54028fa5339a41dcd664ca8d2264fec44309b32d7809b1
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:31:51.259Z
translationReview: automatic
---

En vérifiant mon serveur domestique pour CVE-2026-98432, j’ai constaté que plusieurs services y écoutaient via Multipath TCP, notamment Prometheus-node-exporter, Pi-hole-Exporter et `tailscaled`, tous écrits en Go. Je n’avais jamais configuré cela. L’exploit root décrit publiquement jusqu’à présent pour cette vulnérabilité nécessite précisément un tel listener MPTCP.

En bref : MPTCP fait partie du noyau officiel Linux depuis le noyau 5.6 et est autorisé par défaut. Depuis Go 1.24, les programmes Go ouvrent automatiquement leurs listeners en tant que sockets MPTCP. Deux commandes et quelques compteurs du noyau permettent de déterminer si vous pouvez désactiver MPTCP. Sur la plupart des serveurs, aucun client n’utilise le protocole, et Go revient de lui-même au TCP normal après sa désactivation.

## Ce qu’est Multipath TCP

Une connexion TCP normale est liée à une seule paire d’adresses : IP source et port, IP de destination et port. Lorsqu’un smartphone passe du Wi-Fi au réseau mobile, son IP source change et la connexion est interrompue. Multipath TCP étend TCP afin qu’une connexion puisse utiliser plusieurs chemins simultanément ou successivement, par exemple le Wi-Fi et le réseau mobile.

Techniquement, une connexion MPTCP se compose de plusieurs `Subflows`. Chaque sous-flux est, pour le réseau, une connexion TCP classique via un chemin. L’application ne le voit pas : elle utilise un seul socket et le noyau répartit les données entre les sous-flux. Afin que les données arrivent dans le bon ordre, MPTCP ajoute son propre numéro de séquence 64 bits au niveau de la connexion (Data Sequence Number), en plus des numéros de séquence des différents sous-flux.

MPTCP est négocié lors du handshake TCP normal. SYN, SYN/ACK et le premier ACK portent l’option TCP `MP_CAPABLE` (Option Kind 30), avec laquelle les deux parties indiquent qu’elles parlent MPTCP et échangent des clés. D’autres sous-flux rejoignent ensuite la connexion avec l’option `MP_JOIN`. Si `MP_CAPABLE` est absent de la réponse, parce que le pair ne connaît pas MPTCP ou qu’une middlebox retire l’option, la connexion se poursuit comme un TCP ordinaire. Ce repli fait partie de la norme et explique pourquoi les serveurs compatibles MPTCP fonctionnent avec tous les clients TCP.

Sous Linux, MPTCP est opt-in par socket : une application doit créer explicitement le socket avec le protocole `IPPROTO_MPTCP` (valeur 262). Un programme qui ne le fait pas continue à utiliser le TCP normal, même si MPTCP est actif dans le noyau.

## L’historique

MPTCP est né dans les universités et a été déployé à grande échelle par Apple. L’implémentation Linux est restée hors du noyau officiel pendant plus d’une décennie, avant d’être intégrée sous une nouvelle forme.

| Date | Événement |
|---|---|
| vers 2009 | Des chercheurs de l’Université catholique de Louvain (UCLouvain) développent la première implémentation Linux sous forme de branche distincte du noyau (multipath-tcp.org). |
| mars 2011 | RFC 6182 décrit l’architecture de Multipath TCP (Informational). |
| janvier 2013 | RFC 6824 spécifie MPTCP version 0 comme protocole expérimental. |
| septembre 2013 | iOS 7 paraît avec MPTCP, première utilisation commerciale à grande échelle. Apple utilise notamment MPTCP pour Siri. |
| juin 2015 | KT (Korea Telecom) lance un service commercial qui agrège Wi-Fi et LTE via MPTCP et permet aux smartphones d’atteindre 1 Gbit/s ; il est présenté à l’IETF 93. |
| 2017 | iOS 11 ouvre MPTCP aux développeurs d’applications via `URLSession` (`multipathServiceType`). |
| mars 2020 | RFC 8684 définit MPTCP version 1 comme Proposed Standard et remplace RFC 6824. |
| 29 mars 2020 | Linux 5.6 paraît avec une première prise en charge de MPTCP dans le noyau officiel, réécrite et réservée à la version 1. Les sous-flux supplémentaires ont été ajoutés dans les versions suivantes. |
| 2020 | Dans la Release 16, 3GPP spécifie ATSSS (Access Traffic Steering, Switching, Splitting) pour la 5G ; MPTCP y répartit le trafic entre réseau mobile et Wi-Fi. |
| août 2023 | Go 1.21 introduit `SetMultipathTCP` pour les dialers et listeners, désactivé par défaut. |
| février 2025 | Go 1.24 active MPTCP par défaut pour les listeners (`ListenConfig`), les dialers restant en TCP. |
| octobre 2026 | CVE-2026-98432 : use-after-free dans le code IPv4-TCP ; l’exploit root présenté nécessite un listener MPTCP. |

La branche hors arbre d’UCLouvain prenait en charge les versions 0 et 1. Le noyau officiel ne connaît que la version 1 ; le projet recommande aujourd’hui exclusivement cette implémentation.

## Pourquoi MPTCP est actif sur de nombreux serveurs

Deux réglages par défaut se rejoignent. Le noyau autorise les sockets MPTCP par défaut : le réglage sysctl `net.mptcp.enabled` est défini sur `1`. Et les programmes Go à partir de la version 1.24 créent chaque listener comme socket MPTCP, si le noyau l’autorise. Les services Go tels que les exportateurs Prometheus ou Tailscale écoutent donc via MPTCP dès lors qu’ils sont compilés avec Go 1.24 ou une version plus récente.

Cela ne change rien pour les clients ordinaires : ils n’envoient pas de `MP_CAPABLE`, et la connexion s’effectue en TCP. La surface d’attaque supplémentaire apparaît toutefois côté serveur, car tout client qui envoie `MP_CAPABLE` atteint le code MPTCP dans le noyau.

## CVE-2026-98432

CVE-2026-98432 est un use-after-free lors de l’établissement de connexions IPv4-TCP avec des options IP (CVSS 3.1 : 9.8). Selon l’entrée CVE, Linux est concerné à partir de la version 4.4. L’erreur elle-même concerne tout listener TCP, et pas seulement MPTCP. Toutefois, seule une chaîne passant par un listener MPTCP a jusqu’ici été démontrée pour obtenir des droits root ; selon lilting.ch, Anthropic l’a présentée le 27 octobre 2026 dans un environnement de laboratoire. La possibilité d’obtenir des droits root sans MPTCP reste ouverte.

En amont, l’erreur est corrigée dans les noyaux Stable 5.10.272, 5.15.223, 6.1.190, 6.6.159, 6.12.113, 6.18.56 et 7.2.10, ainsi que dans Mainline 7.3 (commit « ipv4: free inet_opt and ireq_opt after an RCU grace period »). Le 30 octobre 2026, Ubuntu indiquait encore les versions LTS 22.04, 24.04 et 26.04 comme « Vulnerable, work in progress » ; pour 26.04, le correctif est prévu dans `linux` 7.0.0-43.43. Les distributions rétroportent les correctifs ; c’est donc la version du paquet indiquée dans l’advisory de la distribution qui fait foi, et non la version majeure du noyau.

Désactiver MPTCP élimine la condition nécessaire à l’exploit présenté. La vulnérabilité dans le code IPv4-TCP demeure jusqu’à ce que le noyau mis à jour soit installé et démarré.

## Vérifier si MPTCP est actif et utilisé

### 1. Le noyau autorise-t-il MPTCP ?

```bash
sysctl net.mptcp.enabled
```

`1` signifie que les applications peuvent créer des sockets MPTCP. Si `sysctl` indique que la clé est inconnue, MPTCP n’est pas compilé dans le noyau et n’est pas disponible sur cet hôte.

### 2. Quels services écoutent via MPTCP ?

```bash
sudo ss -Mlnp
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-M` | afficher uniquement les sockets MPTCP |
| `-l` | afficher uniquement les sockets en écoute (listeners) |
| `-n` | adresses et ports numériques, sans résolution de noms |
| `-p` | afficher le processus associé au socket ; des droits root sont nécessaires pour les processus d’autres utilisateurs |

</details>

Chaque ligne correspond à un listener MPTCP. Le même port apparaît aussi comme listener TCP avec `ss -tln` ; c’est normal, car le noyau maintient en interne un socket TCP pour chaque listener MPTCP. Les noms de processus indiquent quels programmes sont concernés. Il s’agit généralement de programmes Go.

### 3. Un client a-t-il déjà utilisé MPTCP ?

Le noyau compte les handshakes MPTCP depuis le dernier démarrage. Les compteurs pertinents :

```bash
nstat -asz |
  grep -E 'MPTcpExt(MPCapable|MPCurrEstab)'
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-a` | valeurs absolues depuis le démarrage du système au lieu de la différence depuis le dernier appel |
| `-s` | ne pas mettre à jour le fichier d’historique de `nstat` |
| `-z` | afficher également les compteurs dont la valeur est 0 |

</details>

| Compteur | Signification |
|---|---|
| `MPTcpExtMPCapableSYNRX` | SYN entrants avec `MP_CAPABLE` sur un listener MPTCP |
| `MPTcpExtMPCapableACKRX` | connexions MPTCP entrantes dont le handshake est terminé |
| `MPTcpExtMPCapableSYNTX` | SYN sortants avec `MP_CAPABLE`, soit les connexions que cet hôte voulait établir via MPTCP |
| `MPTcpExtMPCapableSYNACKRX` | connexions sortantes pour lesquelles le pair a confirmé MPTCP |
| `MPTcpExtMPCapableFallbackACK` | connexions revenues à TCP côté serveur durant le handshake |
| `MPTcpExtMPCapableFallbackSYNACK` | connexions revenues à TCP côté client |
| `MPTcpExtMPCurrEstab` | connexions MPTCP actuellement établies |

Si `MPCapableSYNRX` et `MPCapableACKRX` restent à 0 après une durée de fonctionnement représentative, aucun client n’a utilisé MPTCP. Si `MPCapableSYNTX` est supérieur à 0, une application sur l’hôte établit elle-même des connexions MPTCP sortantes ; identifiez cette application avant la désactivation. `ss -Mani` affiche les connexions MPTCP existantes avec les détails des sous-flux.

Les compteurs s’appliquent depuis le dernier redémarrage. Un serveur démarré il y a seulement une heure ne fournit donc pas d’indication fiable. De plus, le noyau compte par espace de noms réseau ; davantage à ce sujet dans la section suivante.

### 4. Vérifier séparément les conteneurs

`net.mptcp.enabled` s’applique par espace de noms réseau. Chaque conteneur doté de son propre réseau (bridge Docker, Podman, pods Kubernetes) possède sa propre valeur, et un nouvel espace de noms commence avec la valeur par défaut `1`, indépendamment du réglage de l’hôte. Sur mon serveur domestique, après la désactivation sur l’hôte, tous les conteneurs bridge restaient sur `1` ; seuls les conteneurs avec `network_mode: host` adoptaient la valeur `0`.

```bash
for c in $(docker ps -q); do
  pid=$(docker inspect -f '{{.State.Pid}}' "$c")
  name=$(docker inspect -f '{{.Name}}' "$c")
  echo "$name $(sudo nsenter -t "$pid" -n sysctl -n net.mptcp.enabled)"
done
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `docker ps -q` | identifiants de tous les conteneurs en cours d’exécution |
| `docker inspect -f '{{.State.Pid}}'` | PID du processus principal dans le conteneur, par lequel `nsenter` trouve l’espace de noms |
| `nsenter -t <pid>` | entrer dans les espaces de noms du processus indiqué |
| `-n` | entrer uniquement dans l’espace de noms réseau ; le système de fichiers et les outils restent ceux de l’hôte |
| `sysctl -n` | afficher uniquement la valeur, sans le nom de la clé |

</details>

`nsenter` a l’avantage que ni `sysctl` ni `cat` ne doivent être présents dans le conteneur ; de nombreuses images (distroless, scratch) ne contiennent pas de shell. De la même manière, vous pouvez interroger `ss -Mln` et les compteurs `nstat` dans l’espace de noms d’un conteneur.

## Quand vous pouvez désactiver MPTCP

La désactivation ne pose pas de problème si les compteurs de l’étape 3, sur l’hôte comme dans les conteneurs, ne montrent aucune utilisation de MPTCP. Les programmes Go en écoute ne perdent rien : Go tente le socket MPTCP, reçoit une erreur du noyau et crée silencieusement un socket TCP. Les autres programmes qui demandent explicitement `IPPROTO_MPTCP` reçoivent l’erreur `ENOPROTOOPT`. La question de savoir s’ils reviennent ensuite à TCP ou s’arrêtent dépend du logiciel ; testez ces services après le changement.

La prudence est de mise dans les cas suivants :

- **Accès hybride et agrégation de liens :** les routeurs ou proxys qui agrègent DSL, fibre et réseau mobile via MPTCP (par exemple OpenMPTCProuter ou des passerelles Hybrid Access de fournisseurs) dépendent de MPTCP.
- **Serveurs pour applications mobiles :** une application iOS propriétaire avec `multipathServiceType` ne profite du changement de réseau que si le serveur parle MPTCP. Les compteurs `MPCapableACKRX` indiquent si de tels clients existent.
- **MPTCP sortant :** les programmes démarrés avec `mptcpize` ou exécutés en Go avec `GODEBUG=multipathtcp=1` ou `3` établissent des connexions MPTCP sortantes. Cela est identifiable lorsque `MPCapableSYNTX` est supérieur à 0.

## Désactiver MPTCP

Il existe trois possibilités, selon la portée souhaitée.

### Pour tout l’hôte

```bash
echo "net.mptcp.enabled = 0" |
  sudo tee /etc/sysctl.d/99-no-mptcp.conf
sudo sysctl --system
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `/etc/sysctl.d/99-no-mptcp.conf` | fichier de configuration lu au démarrage ; le réglage persiste après les redémarrages |
| `sysctl --system` | recharger tous les fichiers sous `/etc/sysctl.d/` et les autres chemins standards, puis appliquer immédiatement les réglages |

</details>

Le réglage empêche seulement la création de nouveaux sockets MPTCP. Les listeners déjà ouverts restent des sockets MPTCP jusqu’au redémarrage du service. Le plus simple est de redémarrer l’hôte ; ensuite, `ss -Mln` doit retourner une liste vide.

### Pour les conteneurs

Dans les conteneurs ayant leur propre réseau, définissez la valeur par conteneur. Docker autorise les sysctls de la zone `net.*`, tant que le conteneur n’utilise pas le réseau de l’hôte :

```yaml
services:
  app:
    sysctls:
      net.mptcp.enabled: 0
```

En appel direct, cela correspond à `docker run --sysctl net.mptcp.enabled=0 …`. La modification prend effet lors de la prochaine recréation du conteneur (`docker compose up -d`).

### Uniquement pour certains programmes Go

Au lieu de désactiver MPTCP dans le noyau, vous pouvez rétablir le comportement standard de Go par service. La variable d’environnement `GODEBUG=multipathtcp=0` désactive MPTCP pour les listeners et les dialers. Pour un service systemd, cela se fait par un drop-in :

```bash
sudo systemctl edit node_exporter
```

```ini
[Service]
Environment=GODEBUG=multipathtcp=0
```

Si le service définit déjà une variable `GODEBUG`, les entrées sont combinées dans une seule variable, séparées par des virgules. Cette possibilité n’agit que sur les programmes Go et seulement si le code n’appelle pas lui-même `SetMultipathTCP` ; un appel explicite dans le programme prévaut sur `GODEBUG`. Pour vos propres projets Go, le même réglage est disponible sous forme de `godebug multipathtcp=0` dans `go.mod` ou sous forme de `//go:debug multipathtcp=0` dans le paquet principal.

## Contrôle après le changement

```bash
sysctl net.mptcp.enabled
ss -Mln
```

Les valeurs attendues sont `0` et aucune ligne sous l’en-tête du tableau de `ss`. Les services continuent alors d’écouter sur les mêmes ports, visibles avec `ss -tln`. Pour annuler le changement, supprimez le fichier sous `/etc/sysctl.d/` et redémarrez l’hôte.

## Sources

1.  [RFC 8684: TCP Extensions for Multipath Operation with Multiple Addresses](https://datatracker.ietf.org/doc/html/rfc8684): spécification de MPTCP version 1 (mars 2020), avec sous-flux, `MP_CAPABLE`, Data Sequence Number et repli.

2.  [RFC 6824](https://datatracker.ietf.org/doc/rfc6824/): MPTCP version 0, expérimental (janvier 2013), remplacé par RFC 8684.

3.  [RFC 6182: Architectural Guidelines for Multipath TCP Development](https://datatracker.ietf.org/doc/rfc6182/): document d’architecture de mars 2011.

4.  [Linux Kernel Documentation: Multipath TCP](https://docs.kernel.org/networking/mptcp.html): opt-in via `IPPROTO_MPTCP`, codes d’erreur lors de la création de sockets, gestionnaire de chemins.

5.  [Linux Kernel Documentation: MPTCP Sysfs variables](https://docs.kernel.org/networking/mptcp-sysctl.html): signification et valeur par défaut de `net.mptcp.enabled` et des autres sysctls MPTCP.

6.  [mptcp.dev](https://www.mptcp.dev/): site web du projet upstream avec FAQ, guide de débogage (`ss -Mani`, `nstat`) et informations sur macOS et iOS.

7.  [KernelNewbies: Linux 5.6](https://kernelnewbies.org/Linux_5.6): aperçu de la version avec la première prise en charge de MPTCP dans le noyau officiel.

8.  [Go 1.21 Release Notes](https://go.dev/doc/go1.21): introduction de la prise en charge de MPTCP dans le paquet `net`, désactivée par défaut, avec repli silencieux sur TCP.

9.  [Go 1.24 Release Notes](https://go.dev/doc/go1.24): MPTCP activé par défaut pour `ListenConfig` sous Linux.

10.  [Go: paramètres GODEBUG](https://go.dev/doc/godebug): valeurs de 0 à 3 de `multipathtcp`, ainsi que `godebug` dans `go.mod` et `//go:debug`.

11.  [Wikipedia: Multipath TCP](https://en.wikipedia.org/wiki/Multipath_TCP): aperçu des implémentations et usages (UCLouvain, Apple, KT, 3GPP ATSSS).

12.  [Ubuntu Security: CVE-2026-98432](https://ubuntu.com/security/CVE-2026-98432): statut des noyaux Ubuntu par version et versions de correctifs prévues.

13.  [lilting.ch: CVE-2026-98432 Linux TCP Use-After-Free](https://lilting.ch/en/articles/linux-ipv4-tcp-options-uaf-mptcp-rce): analyse de l’exploitabilité, rôle de MPTCP et versions corrigées des noyaux Stable.
