---
title: "NatJack : explication des attaques contre les tables NAT, personnes concernées et mesures nécessaires"
navTitle: "NatJack expliqué"
description: "NatJack est une nouvelle classe d’attaques contre les appareils NAT : un attaquant placé derrière le même routeur, pare-feu ou pont Docker peut détourner des connexions TCP tierces, intercepter des réponses DNS ou paralyser le NAT. Fonctionnement de l’attaque, produits concernés, mesures nécessaires et efficaces."
date: "2026-10-28"
kategorie: "Vulnérabilités de sécurité"
timeToRead: "10 min de lecture"
themen:
  - sicherheitsluecken
produkte:
  - "sicherheitsluecken"
protokolle:
  - "tcp"
  - "dns"
  - "container"
  - "haertung"
hauptthema: "sicherheitsluecken"
slug: "natjack-explication-des-attaques-contre-les-tables-nat-personnes-concernees-et-mesures"
translationId: "article-223ba44d60e8cbc8"
translationOf: natjack-nat-tabellen-angriff
url: https://rafaelpfister.ch/fr/blog/natjack-explication-des-attaques-contre-les-tables-nat-personnes-concernees-et-mesures
translationSourceHash: c29a779430bbe591ef42a7db22926ad80fea50613dbc60458b9e88e40edadc02
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T21:30:11.559Z
translationReview: automatic
---

# NatJack : explication des attaques contre les tables NAT, personnes concernées et mesures nécessaires

NatJack est une classe d’attaques visant les appareils qui utilisent la translation d’adresses réseau (NAT) : routeurs, pare-feu, réseaux d’hyperviseurs, ponts de conteneurs et passerelles NAT cloud. Un attaquant qui se trouve derrière le même appareil NAT que sa victime manipule sa table NAT afin que les réponses provenant d’Internet n’arrivent plus à la victime, mais chez lui. Cela permet de détourner des connexions TCP existantes, d’intercepter et de falsifier des réponses DNS ou de bloquer l’appareil NAT pour tous les autres utilisateurs. La recherche a été publiée par Malcolm Stagg (SODIUM-24, LLC) le 6 août 2026 à Black Hat USA, coordonnée via CERT/CC (VU#268870) avec 13 fabricants. Dans 23 produits testés, 95 constats individuels ont été signalés ; selon le livre blanc, chaque appareil testé avec NAT était vulnérable à au moins une partie des attaques.

En bref : aucune action n’est requise dans un réseau domestique composé de ses propres appareils. Des mesures sont nécessaires là où des utilisateurs et des charges de travail externes ou non fiables utilisent le même appareil NAT que vos systèmes de production : Wi-Fi invités sur le pare-feu de l’entreprise, hôtes Docker et Kubernetes avec des images tierces, VM de test dans le réseau NAT d’un hôte Hyper-V, environnements d’hébergement et de coworking. La mesure la plus efficace est le chiffrement (TLS également en interne, DNS sécurisé), suivie de la séparation des zones de confiance sur des appareils NAT distincts.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Assistance pour l’évaluation</p>
<p>Si vous souhaitez vérifier si votre environnement de pare-feu, de conteneurs ou d’hyperviseur est concerné, veuillez utiliser le <a href="https://adeptio.ch/en">formulaire de contact sur adeptio.ch</a>.</p>
</div>

## Rôle d’une table NAT

Derrière un appareil NAT, les équipements disposent d’adresses privées issues de la RFC 1918, par exemple `10.0.10.21`. Lorsqu’un appareil établit une connexion vers l’extérieur, l’appareil NAT remplace l’adresse et le port source par son adresse externe et un port externe, puis mémorise l’association dans la table NAT. Lorsque la réponse du serveur revient, l’appareil consulte la table pour savoir à quel équipement interne elle appartient. Sous Linux, Netfilter s’en charge avec son suivi des connexions `conntrack` ; la plupart des routeurs commerciaux, OpenWRT, Docker et Kubernetes utilisent ce code.

À l’origine, le NAT était une solution transitoire face à la pénurie d’adresses IPv4 et non une limite de sécurité. En pratique, il en est toutefois devenu une : au niveau 2, les VLAN, l’isolation des ports, Dynamic ARP Inspection et l’isolation des vSwitch séparent proprement les appareils, et l’ARP spoofing est désormais rarement possible dans les réseaux bien administrés. Aux niveaux 3/4, les réseaux séparés convergent cependant souvent à nouveau dans le même appareil NAT. C’est précisément là que NatJack intervient.

![Appareil NAT partagé avec deux VLAN séparés : l’entrée NAT de la victime est remplacée par une entrée de l’attaquant, puis les réponses du serveur cible sont envoyées à l’attaquant](../images/natjack-nat-tabellen-angriff/geteiltes-nat.svg)

## Fonctionnement de l’attaque

Toutes les variantes de NatJack suivent le même schéma : l’attaquant amène l’appareil NAT à supprimer l’entrée de table de la victime, puis crée immédiatement sa propre entrée avec les mêmes paramètres externes. L’appareil NAT transmet alors la réponse suivante du serveur à l’attaquant. Ni le serveur ni la victime ne reçoivent de message d’erreur.

Cela est rendu possible par des comportements conçus pour un réseau coopératif, où ils sont pertinents :

- **Loose-Mode :** Par défaut, Netfilter ajoute également à la table des connexions TCP dont il n’a pas vu la négociation initiale. Cela vise à préserver les connexions après un redémarrage du routeur ou un changement de chemin, mais permet aussi de créer des entrées sans établissement de connexion valide.
- **Vérification tolérante des paquets de réinitialisation :** L’appareil NAT vérifie les numéros de séquence moins strictement que les points terminaux eux-mêmes afin de ne pas rejeter de paquets valides. Des paquets qu’un point terminal ignorerait peuvent encore être considérés par l’appareil NAT comme une fin de connexion.
- **Port Preservation et Endpoint-Independent Mapping :** Netfilter reprend si possible le port interne comme port externe et réutilise pour toutes les destinations une association déjà attribuée. Le comportement lors de conflits de ports permet de déduire quels ports sont actuellement utilisés par d’autres appareils.
- **Table finie :** Lorsque la table est pleine, Netfilter supprime aléatoirement des entrées non encore confirmées, par exemple des requêtes DNS ouvertes. Les connexions confirmées peuvent rester présentes jusqu’à cinq jours.

Pour la plupart des variantes, l’attaquant doit pouvoir envoyer des paquets avec une adresse source falsifiée. Si l’attaquant et la victime sont dans le même sous-réseau, un appareil ou un conteneur disposant de droits de socket brut lui suffit (`NET_RAW`). La variante upstream fonctionne également au-delà des limites de sous-réseau et de VLAN ; l’attaquant a toutefois aussi besoin de son propre serveur sur Internet, capable d’envoyer des paquets avec une adresse source falsifiée. Le livre blanc décrit les séquences de paquets et le code de preuve de concept ; le rapport technique annoncé avec le framework de test n’a pas encore été publié.

## Aperçu des cinq vulnérabilités

Le livre blanc décrit cinq vulnérabilités. Les évaluations sont des valeurs typiques ; elles varient selon le produit.

| Vulnérabilité | Conséquence | CVSS 3.1 (typique) |
|---|---|---|
| Déni de service | L’attaquant remplit la table NAT d’entrées confirmées. Aucun autre appareil ne peut établir de nouvelles connexions, avec Netfilter jusqu’à cinq jours ou jusqu’au redémarrage. | 7.4 (High) |
| Divulgation de port et d’IP | L’attaquant détecte qu’un autre appareil est connecté à un serveur précis et détermine son port et son adresse interne. Cela sert surtout d’étape préalable aux autres attaques. | 4.7 (Medium) |
| Détournement DNS via UDP | Avec une certaine probabilité, l’attaquant intercepte la réponse DNS destinée à un autre appareil et envoie une réponse falsifiée, par exemple avec l’adresse de son propre serveur. | 10.0 (Critical) |
| Détournement TCP (downstream) | L’attaquant prend le contrôle d’une connexion TCP existante d’un appareil du même sous-réseau, lit les réponses du serveur et envoie des données au nom de la victime. | 9.6 (Critical) |
| Détournement TCP (upstream) | Comme ci-dessus, mais au-delà des limites de sous-réseau et de VLAN avec l’aide d’un serveur externe de l’attaquant. | 10.0 (Critical) |

Tous les protocoles sans chiffrement fort sont concernés : HTTP, DNS sur UDP, protocoles d’administration basés sur Telnet ainsi que les protocoles de messagerie tels que SMTP et IMAP lorsqu’ils fonctionnent sans TLS. Dans le cas d’une connexion TLS, l’attaquant peut interrompre la connexion, mais n’obtient aucune donnée lisible et ne peut rien injecter. Le livre blanc souligne par ailleurs que les paquets d’attaque peuvent être envoyés avec une TTL réduite afin d’atteindre l’appareil NAT sans pouvoir aller plus loin dans le réseau. Cela complique la détection.

Par rapport à l’ARP spoofing, NatJack produit un effet similaire, mais contourne les mécanismes de protection établis au niveau 2. Des travaux antérieurs de Yang et al. (CVE-2023-30305 à CVE-2023-30314) ont montré le détournement TCP via les tables NAT sur des routeurs Wi-Fi avec des options non standard non sécurisées ; NatJack fonctionne dans la configuration par défaut.

## Qui est concerné

Selon le livre blanc, pratiquement tous les routeurs basés sur Linux sont vulnérables, de même que des systèmes reposant sur une base de code entièrement différente. L’aperçu suivant résume l’état de la matrice des correctifs sur natjack.io (dernière mise à jour le 19 août 2026).

| Domaine | Produits | État |
|---|---|---|
| Linux / Netfilter | Noyau Linux, OpenWRT, DD-WRT, presque tous les routeurs basés sur Linux | Correctif partiel avec CVE-2026-63913 à partir de Linux 7.1 ; augmente la difficulté, sans protection complète. OpenWRT et DD-WRT ont besoin des modifications de Netfilter. |
| BSD | FreeBSD pf, OpenBSD pf | FreeBSD est fortement atténué à partir de 15.0.0 grâce à une modification issue d’OpenBSD ; les versions antérieures sont concernées. OpenBSD est nettement mieux protégé depuis 2016. |
| Windows | Windows NAT (vSwitch Hyper-V avec NAT) | CVE-2026-56181 (juillet 2026) et CVE-2026-56179 (août 2026), tous deux classés comme modérés. L’atténuation de CVE-2026-56179 est désactivée par défaut et s’active via le Registre. Windows Internet Connection Sharing est en cours d’examen. |
| Équipements réseau | Cisco Firepower et Meraki, NETGEAR Nighthawk, Google Nest WiFi Pro, SpaceX Starlink, Amazon eero | Selon le fabricant : protection partielle, recommandations de configuration ou aucune mise à jour. eero a publié sa propre communication. |
| Hotspots | Hotspot personnel iOS, partage Internet macOS, hotspot Android | concernés |
| Virtualisation et conteneurs | Hyper-V, KVM/QEMU, Apple Virtualization Framework, Docker (Bridge), FreeBSD Jails, GKE, MicroK8s avec Calico ou Flannel, Azure AKS | concernés ; protection surtout par la configuration (voir ci-dessous) |
| Cloud | AWS NAT Gateway et NLB, Google Cloud NAT, Cloud Run, Azure Firewall, Azure NAT Gateway, Azure Functions | AWS a renforcé le service côté serveur, les clients n’ont rien à faire. AWS Lambda est protégé en l’absence de `NET_RAW`. Vérifiez l’état sur natjack.io pour les autres services. |

## Des mesures sont-elles nécessaires ?

Le point décisif est de savoir qui se trouve derrière le même appareil NAT ; que votre modèle de routeur figure ou non dans la liste est secondaire. Chaque variante suppose que l’attaquant contrôle déjà un appareil, une VM ou un conteneur derrière le NAT. Lorsque seuls vos propres appareils fiables sont concernés, un attaquant disposant de cet accès aurait de toute façon des moyens plus simples. Le livre blanc indique donc que la plupart des utilisateurs à domicile ne sont pas sérieusement touchés, tandis que les conséquences sont nettement plus graves pour les entreprises et les prestataires.

![Trois niveaux : aucune pression d’action avec ses propres appareils uniquement, vérification et durcissement pour des réseaux invités séparés sur le même pare-feu, mesures nécessaires avec des utilisateurs ou charges de travail externes sur le même NAT](../images/natjack-nat-tabellen-angriff/massnahmen-noetig.svg)

Dans les PME suisses, le niveau intermédiaire est le cas le plus fréquent : le Wi-Fi invités est séparé par VLAN, mais passe par le même pare-feu vers Internet que le réseau de l’entreprise. La séparation par VLAN empêche l’ARP spoofing et l’accès direct, mais les deux réseaux partagent tout de même la table NAT. Pour la variante downstream, l’attaquant devrait se trouver dans le même sous-réseau que sa victime ; la variante upstream fonctionne au-delà de la limite VLAN, mais elle est plus complexe et nécessite un serveur externe. Des mesures sont clairement nécessaires là où des tiers se trouvent dans le même sous-réseau (Wi-Fi d’hôtel, de cabinet ou d’école sans isolation des clients) ainsi que sur les hôtes de conteneurs et Kubernetes exécutant des charges de travail de niveaux de confiance différents. Un paquet npm ou PyPI compromis dans un conteneur y suffit comme point de départ.

Pour les services de messagerie, il vaut la peine d’examiner spécifiquement les connexions internes sans TLS : scanners, applications métier et systèmes de monitoring qui envoient sans chiffrement via le port 25 vers un relais interne, ainsi qu’IMAP sans TLS. L’article [Envoi d’e-mails via un relais : vérifier TLS et l’authentification](/blog/mail-relay-tls-authentisierung-pruefen) explique comment vérifier TLS et l’authentification sur le relais.

## Mesures

Les recommandations du livre blanc peuvent être organisées en cinq couches. Elles se distinguent par les vulnérabilités contre lesquelles elles sont efficaces.

![Cinq couches de protection : chiffrement, séparation des zones de confiance, anti-spoofing, durcissement du NAT et surveillance, avec leurs effets respectifs sur le détournement, la divulgation de ports et le déni de service](../images/natjack-nat-tabellen-angriff/schutzschichten.svg)

### Chiffrement

TLS sur toutes les connexions, y compris entre les systèmes internes, rend inutiles les connexions TCP détournées : l’attaquant peut les interrompre, mais ne peut ni les lire ni injecter des données. Pour DNS, le livre blanc recommande DNSSEC contre les réponses falsifiées ainsi que DNS over HTTPS ou DNS over TLS contre l’interception. Le chiffrement ne protège ni contre la divulgation de ports ni contre le déni de service.

### Séparation des zones de confiance

Exploitez les appareils et charges de travail non fiables dans leurs propres sous-réseaux et VLAN, avec des ACL L3/L4 qui bloquent tout accès aux hôtes fiables. Un appareil NAT distinct ou une adresse externe distincte par zone de confiance est plus efficace, car sans table NAT commune, la base de toutes les variantes disparaît. Dans le cloud, cela signifie : ne pas exploiter des charges de travail fiables et externes derrière la même passerelle NAT, et utiliser des adresses IP dédiées plutôt que partagées pour les services serverless. Dans Kubernetes, les pods non fiables s’exécutent sur leurs propres nœuds.

### Anti-spoofing

Activez IP Source Guard ou une fonction comparable sur le commutateur ou le point d’accès, ainsi que uRPF (Reverse Path Filtering) sur le routeur, afin que les clients ne puissent envoyer qu’avec leur propre adresse. Dans les conteneurs, retirer la capability `NET_RAW` empêche un processus compromis de générer des paquets falsifiés. Docker attribue cette capability par défaut.

```bash
docker run --cap-drop=NET_RAW --user 1000:1000 example/app:1.4
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `run` | Démarre un nouveau conteneur à partir d’une image |
| `--cap-drop=NET_RAW` | Retire au conteneur le droit d’utiliser des sockets bruts ; il ne peut alors plus générer de paquets IP falsifiés |
| `--user 1000:1000` | Exécute le processus avec l’identifiant utilisateur et groupe 1000 au lieu de root |
| `example/app:1.4` | Image et tag du conteneur (exemple) |

</details>

`ping` nécessite `NET_RAW` dans certaines images ; sur les noyaux récents, il fonctionne généralement aussi sans cette capability grâce aux sockets ICMP non privilégiés. Dans Kubernetes, le `securityContext` du conteneur s’en charge :

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - NET_RAW
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `runAsNonRoot: true` | Kubernetes ne démarre pas le conteneur s’il devait s’exécuter en tant que root |
| `allowPrivilegeEscalation: false` | Empêche le processus d’acquérir des droits supplémentaires, par exemple via des programmes setuid |
| `capabilities.drop: [NET_RAW]` | Retire la capability de socket brut, comme `--cap-drop=NET_RAW` avec Docker |

</details>

Le Pod Security Standard `restricted` exige que les conteneurs abandonnent toutes les capabilities (`drop: [ALL]`) et couvre donc `NET_RAW`.

### Durcissement du NAT

Installez les mises à jour disponibles : noyau ou firmware avec le correctif Netfilter, mises à jour Windows de juillet et août 2026 sur les hôtes Hyper-V avec réseau NAT, et FreeBSD à partir de 15.0.0. Le changelog du paquet du noyau indique si le noyau de votre distribution contient le correctif pour CVE-2026-63913.

Sur les routeurs Linux et les hôtes de conteneurs, il est en outre possible de désactiver le Loose-Mode. Netfilter n’ajoute alors à la table que les connexions TCP avec une négociation complète.

```bash
sysctl net.netfilter.nf_conntrack_tcp_loose
sudo sysctl -w net.netfilter.nf_conntrack_tcp_loose=0
echo "net.netfilter.nf_conntrack_tcp_loose = 0" | sudo tee /etc/sysctl.d/90-natjack.conf
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `sysctl <schlüssel>` | Affiche la valeur actuelle ; `1` signifie que le Loose-Mode est actif (par défaut) |
| `-w` | Écrit immédiatement la valeur dans le noyau en cours d’exécution, sans redémarrage |
| `nf_conntrack_tcp_loose=0` | Désactive la prise en charge de connexions dont la négociation initiale n’a pas été observée |
| `tee /etc/sysctl.d/90-natjack.conf` | Enregistre durablement le réglage afin qu’il survive à un redémarrage |

</details>

Le Loose-Mode existe pour une raison : après le redémarrage du routeur ou un basculement dans un cluster avec `conntrackd`, les connexions existantes s’interrompent sans lui. Vérifiez ce point avant le changement, surtout avec des pare-feu à haute disponibilité. Le livre blanc souligne explicitement que plusieurs mesures impliquent un arbitrage entre sécurité, disponibilité et compatibilité des protocoles.

Autres réglages, dans la mesure où votre produit les propose :

- **Limite de connexions par client :** Le livre blanc recommande une limite d’environ 10 000 connexions simultanées par client. Sur les pare-feu basés sur pf tels qu’OPNsense et pfSense, ce réglage se trouve dans les options avancées de la règle de pare-feu (pf : `max-src-states`).
- **Désactiver Port Preservation et Endpoint-Independent Mapping :** Rend la détermination des ports plus difficile. Avec pf, une règle NAT sans `static-port` remplace aléatoirement le port source. Notez que certaines applications VoIP et pair à pair reposent sur des associations fixes.
- **Randomisation ISN :** Dans pf, via `modulate state` au lieu de `keep state`. Évitez les règles avec `flags any`, car elles autorisent les connexions sans négociation valide.

### Surveillance

Le livre blanc cite plusieurs signes de tentatives d’attaque : une table NAT pleine ou presque pleine, des flots de paquets d’un client sur une grande plage de ports source, la même adresse IP associée à deux adresses MAC ou une adresse MAC associée à plusieurs adresses IP, des flots de paquets RST avec des numéros de séquence changeants, des numéros de séquence largement en dehors de la fenêtre TCP et des paquets avec une TTL anormalement basse. Sous Linux, le taux de remplissage de la table s’affiche par comparaison de deux valeurs :

```bash
cat /proc/sys/net/netfilter/nf_conntrack_count /proc/sys/net/netfilter/nf_conntrack_max
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `nf_conntrack_count` | Nombre d’entrées actuellement occupées dans la table |
| `nf_conntrack_max` | Taille maximale de la table ; dépend de la mémoire vive (262 144 entrées avec 4 Go ou plus) |

</details>

Si la première valeur reste durablement proche de la seconde, cela justifie une investigation, indépendamment de NatJack. De nombreux pare-feu affichent le même indicateur sous « State Table » ou « Sessions » dans le tableau de bord et peuvent déclencher une alerte à partir d’un seuil.

## Mise en perspective

NatJack exploite des hypothèses présentes depuis des décennies dans les implémentations NAT, et non une erreur de programmation isolée. C’est pourquoi aucun correctif ne peut éliminer entièrement cette classe d’attaques : Linux et Windows ont accru la difficulté, AWS a apporté des améliorations côté serveur, et de nombreux appareils restent sans mise à jour. Les mesures durables relèvent de votre propre architecture : protocoles chiffrés également dans le réseau interne et aucun appareil NAT commun pour des systèmes auxquels vous accordez des niveaux de confiance différents. Ceux qui ont mis en œuvre ces deux points peuvent traiter NatJack selon leur cycle normal de correctifs.

## Sources

1.  [NatJack (natjack.io)](https://natjack.io/): page du projet du chercheur avec matrice des correctifs par produit, liste des CVE et recommandations ; état de la matrice au 19 août 2026.

2.  [NatJack White Paper 1.0 (PDF)](https://natjack.io/natjack_white_paper_1.0.pdf): Malcolm Stagg, SODIUM-24, LLC, août 2026 ; description technique des cinq vulnérabilités, évaluations CVSS, mesures d’atténuation et indicateurs.

3.  [Black Hat USA 2026: Breaking Trust Boundaries](https://blackhat.com/us-26/briefings/schedule/#breaking-trust-boundaries-exploiting-design-assumptions-in-network-infrastructure-53311): conférence du 6 août 2026 ayant rendu publique cette classe d’attaques.

4.  [Microsoft: CVE-2026-56181](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-56181): Windows NAT, variante downstream sur Hyper-V ; mise à jour de juillet 2026.

5.  [Microsoft: CVE-2026-56179](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-56179): Windows NAT, variante upstream sur Hyper-V ; mise à jour d’août 2026, atténuation à activer via le Registre.

6.  [Noyau Linux : commit Netfilter pour CVE-2026-63913](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/net/netfilter/nf_conntrack_proto_tcp.c?id=2bb6d82b586ea5a4cb73bbdd6b7432e96096bc77): atténuation partielle dans le suivi TCP de Netfilter.

7.  [FreeBSD : commit pf](https://github.com/freebsd/freebsd-src/commit/9485d22286603ef2ac2e2009dd7dfcc6a59e2283): modification reprise d’OpenBSD, qui complique fortement la plupart des variantes dans FreeBSD 15.0.0.

8.  [eero : Protecting eero customers from NatJack](https://eero.com/blog/articles/protecting-eero-customers-from-natjack): communication du fabricant concernant ses propres appareils.

9.  [AWS : Troubleshoot NAT gateways](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-troubleshooting.html): indications sur l’utilisation des ports après le durcissement côté serveur, auxquelles AWS renvoie dans sa prise de position.

10.  [Documentation du noyau Linux : Netfilter Conntrack Sysctl](https://docs.kernel.org/networking/nf_conntrack-sysctl.html): description de `nf_conntrack_tcp_loose`, `nf_conntrack_max` et des autres paramètres Conntrack.
