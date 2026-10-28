---
title: "Redirection de port avec netsh portproxy : accéder à des services internes via un hôte de rebond"
navTitle: "netsh portproxy"
description: "Avec netsh interface portproxy, Windows intègre une redirection de ports TCP. Combinée à un VPN tel que Tailscale, elle permet d’accéder depuis l’extérieur à un service interne, par exemple l’interface d’un NAS, sans l’exposer publiquement. Comment configurer, sécuriser et supprimer la redirection, ainsi que ses limites : pas d’UDP, pas de chiffrement supplémentaire, et des pièges liés aux certificats et aux redirections."
date: "2026-09-02"
kategorie: "Windows et réseau"
timeToRead: "9 min de lecture"
themen:
  - windows-client
produkte:
  - "windows-client"
protokolle:
  - "tcp"
  - "haertung"
slug: "redirection-de-port-avec-netsh-portproxy-acceder-a-des-services-internes-via-un-hote-de-rebond"
translationId: "article-236adcb4ae982572"
translationOf: windows-portproxy-portweiterleitung
translationSourceHash: 152d2f6d20774179128718ab4e218fcc0cd76cbcf0706060df7d60f30e012900
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T09:54:54.219Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/redirection-de-port-avec-netsh-portproxy-acceder-a-des-services-internes-via-un-hote-de-rebond
---

# Redirection de port avec netsh portproxy : accéder à des services internes via un hôte de rebond

Un service interne n’écoute souvent que sur le réseau local : l’interface web d’un NAS, le panneau d’une imprimante, une page d’administration. Si vous souhaitez y accéder depuis l’extérieur sans exposer le service à Internet, vous avez besoin d’un chemin via un ordinateur qui voit les deux côtés. Windows fournit pour cela un outil intégré : `netsh interface portproxy` redirige les connexions TCP entrantes vers une autre destination. Combiné à un VPN tel que Tailscale ou WireGuard, un ordinateur du réseau cible devient un hôte de rebond par lequel vous accédez au service interne.

Exemple concret : un NAS dont l’interface web se trouve sur `10.0.0.245:5000` n’est accessible que sur le réseau local. Un PC Windows se trouve sur le même réseau et est également accessible via VPN. Configurez sur ce PC une redirection de port depuis son adresse VPN vers le NAS, puis ouvrez l’interface du NAS dans le navigateur via l’adresse VPN du PC. Le service reste sur le réseau interne ; seul l’hôte de rebond est accessible via le VPN.

## Fonctionnement de portproxy

`portproxy` fait partie du service IP Helper (`iphlpsvc`). Le service accepte les connexions sur un port local et les transmet à une destination. Il s’agit d’un relais TCP pur au niveau applicatif : ce n’est pas une règle NAT du pare-feu, mais un processus qui copie les octets entre deux connexions. Si `iphlpsvc` n’est pas exécuté, aucune redirection ne fonctionne. Le service est présent par défaut ; son type de démarrage doit être réglé sur automatique si la redirection doit survivre à un redémarrage.

## Configuration

Une redirection nécessite deux étapes : la règle portproxy et une règle de pare-feu qui autorise l’accès à l’écouteur. Exécutez les deux dans une invite de commandes ou PowerShell avec des droits d’administrateur.

Commencez par la redirection. Elle se lie à une adresse et un port locaux et pointe vers une IP et un port de destination :

```powershell
netsh interface portproxy add v4tov4 listenaddress=100.100.10.10 listenport=5000 connectaddress=10.0.0.245 connectport=5000
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `v4tov4` | IPv4 écoute, IPv4 se connecte ; également possible : `v4tov6`, `v6tov4`, `v6tov6` |
| `listenaddress` | Adresse locale sur laquelle l’écoute est effectuée ; ici, l’adresse VPN de l’hôte de rebond, afin que les connexions n’arrivent que via le VPN |
| `listenport` | Port local sur lequel l’écoute est effectuée |
| `connectaddress` | IP de destination vers laquelle le trafic est transmis (le service interne) |
| `connectport` | Port de destination du service interne |

</details>

La liaison à l’adresse VPN plutôt qu’à `0.0.0.0` constitue la première mesure de sécurité : l’écouteur n’apparaît que sur l’interface VPN, et non sur toutes les cartes réseau de l’hôte de rebond. La deuxième mesure est le pare-feu. Ouvrez le port de l’écouteur uniquement pour la plage d’adresses de votre VPN, et non pour toutes les adresses :

```powershell
New-NetFirewallRule -DisplayName "NAS-Proxy (VPN)" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 5000 -RemoteAddress 100.64.0.0/10
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Direction Inbound` | Règle pour le trafic entrant |
| `-Protocol TCP` | portproxy ne transmet que TCP, donc TCP |
| `-LocalPort 5000` | Le port de l’écouteur de la règle portproxy |
| `-RemoteAddress 100.64.0.0/10` | Seules les sources de cette plage sont autorisées ; ici, la plage Tailscale, sinon le bloc CIDR de votre VPN |

</details>

## Vérifier et utiliser

Vérifiez d’abord sur l’hôte de rebond lui-même si le service interne est accessible, puis affichez la redirection active :

```powershell
Test-NetConnection -ComputerName 10.0.0.245 -Port 5000
netsh interface portproxy show v4tov4
```

Si la destination répond et que la règle apparaît dans la liste, testez depuis votre appareil distant. Le service est désormais accessible via l’adresse et le port de l’hôte de rebond :

```powershell
Test-NetConnection -ComputerName 100.100.10.10 -Port 5000
```

Dans le navigateur, ouvrez ensuite `http://100.100.10.10:5000`. Si vous avez besoin de plusieurs ports du même service, par exemple 5000 et 5001 pour HTTP et HTTPS, créez une règle portproxy distincte et l’autorisation de pare-feu correspondante pour chaque port.

## Vue d’ensemble de type manpage

Les principaux sous-commandes de `netsh interface portproxy` :

<details class="options-details">
<summary>Vue d’ensemble des options</summary>

| Commande | But |
|---|---|
| `add v4tov4 …` | Créer une redirection (listenaddress/listenport → connectaddress/connectport) |
| `show v4tov4` | Afficher les redirections IPv4 actives |
| `show all` | Afficher toutes les redirections de toutes les variantes de protocole |
| `delete v4tov4 listenaddress=… listenport=…` | Supprimer une redirection |
| `reset` | Supprimer toutes les règles portproxy |

</details>

Les règles sont stockées dans le registre sous `HKLM\SYSTEM\CurrentControlSet\Services\PortProxy` et persistent après un redémarrage. Elles ne sont visibles que via `netsh` ou directement dans le registre, pas dans l’interface graphique du pare-feu.

## Alternatives

`portproxy` est pratique lorsque l’hôte de rebond est déjà sous Windows et que vous ne souhaitez rien installer. Deux alternatives résolvent le même problème avec d’autres caractéristiques.

Un tunnel SSH avec redirection locale (`ssh -L 5000:10.0.0.245:5000 benutzer@jumphost`) chiffre le trajet jusqu’à l’hôte de rebond lui-même et fonctionne sur toutes les plateformes. Il nécessite un serveur SSH sur l’hôte de rebond et n’existe que tant que la session SSH est active.

Un routeur de sous-réseau Tailscale (`tailscale set --advertise-routes=10.0.0.0/24`) rend l’ensemble du sous-réseau interne accessible à vos appareils VPN. Vous adressez alors directement le service interne à sa véritable IP, sans redirection par port. C’est la solution la plus directe si vous souhaitez accéder à plusieurs appareils internes, mais elle exige l’approbation de la route dans l’administration Tailscale.

## Limites

Une redirection de port avec portproxy résout l’accès, mais elle présente des limites claires qu’il convient de connaître avant de l’utiliser :

- **TCP uniquement.** `portproxy` ne transmet que TCP. Les services nécessitant UDP (DNS, de nombreux protocoles VPN et de jeu, certaines transmissions vidéo) ne peuvent pas être pris en charge ainsi.
- **Pas de chiffrement supplémentaire.** La redirection copie les octets sans les modifier. La confidentialité du trajet est assurée uniquement par le VPN via lequel vous accédez à l’hôte de rebond. Sur un réseau de transport non chiffré, le trafic ne serait pas protégé.
- **Avertissement de certificat avec HTTPS via l’IP.** Si vous redirigez un service HTTPS et l’appelez via l’IP de l’hôte de rebond, le certificat de la destination ne correspond pas à l’adresse appelée. Le navigateur affiche un avertissement. C’est acceptable pour un test bref, mais pas pour une utilisation permanente.
- **Redirections et adresses absolues.** Certaines interfaces web redirigent elles-mêmes vers leur nom d’hôte ou un autre port, ou construisent des liens absolus avec leur adresse interne. L’accès via l’hôte de rebond échoue alors, même si la redirection est en place. Ces services nécessitent un véritable proxy inverse plutôt qu’un simple relais de port.
- **Liaison à une adresse qui doit exister au démarrage.** Si la règle se lie à une `listenaddress` précise, cette adresse doit être disponible au démarrage du service. Si l’interface VPN ne s’active que plus tard, la liaison peut échouer jusqu’au redémarrage du service ou à la recréation de la règle.
- **Un accès supplémentaire au réseau interne.** Chaque redirection constitue un chemin depuis l’extérieur vers un service interne. Restreignez strictement le pare-feu à la plage VPN, liez-vous à l’adresse VPN et supprimez la redirection dès que vous n’en avez plus besoin.

## Supprimer à nouveau

Après avoir terminé, supprimez la redirection et la règle de pare-feu :

```powershell
netsh interface portproxy delete v4tov4 listenaddress=100.100.10.10 listenport=5000
Remove-NetFirewallRule -DisplayName "NAS-Proxy (VPN)"
```

Une redirection de port est un outil destiné à un accès ciblé et limité dans le temps, et non à un canal ouvert en permanence. Pour l’exploitation durable d’un service interne via Internet, un proxy inverse avec un certificat valide ou un VPN avec routage de sous-réseau constitue une solution plus propre.

## Historique des modifications

- **25 octobre 2026 :** Correction : les routes de sous-réseau sont annoncées avec tailscale set --advertise-routes au lieu de tailscale up. Liens des Sources mis à jour vers les nouvelles adresses de Microsoft Learn et Tailscale.

## Sources

1.  [netsh interface (Microsoft Learn)](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/netsh-interface): référence des commandes de `netsh interface` avec la syntaxe des sous-commandes portproxy et des variantes de protocole.

2.  [New-NetFirewallRule (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/netsecurity/new-netfirewallrule?view=windowsserver2025-ps): paramètres de la règle de pare-feu, y compris la restriction aux plages d’adresses via RemoteAddress.

3.  [Tailscale: Subnet routers](https://tailscale.com/docs/features/subnet-routers): rendre tout un sous-réseau accessible via le VPN, comme alternative à la redirection par port ; les routes sont annoncées avec `tailscale set --advertise-routes`.
