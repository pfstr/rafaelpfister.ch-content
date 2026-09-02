---
title: "Redirection de port avec netsh portproxy : accéder à des services internes via un hôte de rebond"
navTitle: "netsh portproxy"
description: "Windows intègre une redirection de port TCP avec netsh interface portproxy. Associée à un VPN tel que Tailscale, elle permet d’accéder depuis l’extérieur à un service interne, par exemple l’interface d’un NAS, sans l’exposer publiquement. Découvrez comment configurer, sécuriser et supprimer la redirection, ainsi que ses limites : pas d’UDP, pas de chiffrement supplémentaire, et des pièges liés aux certificats et aux redirections."
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
aiPrompt: |
  Du bist mein Windows-Administrationsassistent. Hilf mir, mit netsh interface portproxy eine TCP-Portweiterleitung über einen Windows-Jumphost einzurichten, um einen internen Dienst (z. B. eine NAS-Weboberfläche) über ein VPN zu erreichen: Weiterleitung anlegen, Firewall auf den VPN-Bereich beschränken, prüfen, wieder entfernen, und die Grenzen (kein UDP, keine Verschlüsselung, Zertifikats- und Redirect-Probleme) einordnen.
translationOf: windows-portproxy-portweiterleitung
url: https://rafaelpfister.ch/fr/blog/redirection-de-port-avec-netsh-portproxy-acceder-a-des-services-internes-via-un-hote-de-rebond
translationSourceHash: a4888a85b953fbf7b2248232b7b7361e752300872cdb570d6fd15b1cb806ef89
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:02:59.318Z
translationReview: automatic
---

# Redirection de port avec netsh portproxy : accéder à des services internes via un hôte de rebond

Un service interne n’écoute souvent que sur le réseau local : l’interface web d’un NAS, le panneau d’une imprimante, une page d’administration. Si vous souhaitez y accéder depuis l’extérieur sans exposer le service sur Internet, vous avez besoin d’un chemin passant par un ordinateur qui voit les deux côtés. Windows propose pour cela un outil intégré : `netsh interface portproxy` redirige les connexions TCP entrantes vers une autre destination. En combinaison avec un VPN tel que Tailscale ou WireGuard, un ordinateur du réseau cible devient un hôte de rebond par lequel vous accédez au service interne.

Exemple concret : un NAS dont l’interface web est accessible sur `10.0.0.245:5000` n’est joignable que depuis le réseau local. Sur ce même réseau se trouve un PC Windows également accessible via VPN. Configurez sur ce PC une redirection de port depuis son adresse VPN vers le NAS, puis ouvrez l’interface du NAS dans le navigateur via l’adresse VPN du PC. Le service reste sur le réseau interne ; seul l’hôte de rebond est accessible via le VPN.

## Fonctionnement de portproxy

`portproxy` fait partie du service IP Helper (`iphlpsvc`). Le service accepte les connexions sur un port local et les transmet à une destination. Il s’agit d’un simple relais TCP au niveau applicatif : ce n’est pas une règle NAT du pare-feu, mais un processus qui copie les octets entre deux connexions. Si `iphlpsvc` n’est pas en cours d’exécution, aucune redirection ne fonctionne. Le service est présent par défaut ; son type de démarrage devrait être réglé sur automatique afin que la redirection survive à un redémarrage.

## Configuration

Une redirection nécessite deux étapes : la règle portproxy et une règle de pare-feu qui autorise l’accès au listener. Exécutez les deux dans une invite de commandes ou dans PowerShell avec des droits d’administrateur.

Commencez par la redirection. Elle se lie à une adresse locale et à un port, et pointe vers l’adresse IP et le port de destination :

```powershell
netsh interface portproxy add v4tov4 listenaddress=100.100.10.10 listenport=5000 connectaddress=10.0.0.245 connectport=5000
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `v4tov4` | IPv4 écoute, IPv4 se connecte ; également possible : `v4tov6`, `v6tov4`, `v6tov6` |
| `listenaddress` | Adresse locale sur laquelle l’écoute est effectuée ; ici, l’adresse VPN de l’hôte de rebond afin que les connexions n’arrivent que via le VPN |
| `listenport` | Port local sur lequel l’écoute est effectuée |
| `connectaddress` | Adresse IP de destination vers laquelle le trafic est transmis (le service interne) |
| `connectport` | Port de destination du service interne |

</details>

La liaison à l’adresse VPN plutôt qu’à `0.0.0.0` constitue la première mesure de sécurité : le listener n’apparaît que sur l’interface VPN, et non sur toutes les cartes réseau de l’hôte de rebond. La deuxième mesure est le pare-feu. N’ouvrez le port du listener que pour la plage d’adresses de votre VPN, et non pour toutes les adresses :

```powershell
New-NetFirewallRule -DisplayName "NAS-Proxy (VPN)" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 5000 -RemoteAddress 100.64.0.0/10
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Direction Inbound` | Règle pour le trafic entrant |
| `-Protocol TCP` | portproxy ne redirige que TCP, donc TCP |
| `-LocalPort 5000` | Le port du listener issu de la règle portproxy |
| `-RemoteAddress 100.64.0.0/10` | Seules les sources de cette plage sont autorisées ; ici, la plage Tailscale, sinon le bloc CIDR de votre VPN |

</details>

## Vérifier et utiliser

Vérifiez d’abord sur l’hôte de rebond lui-même que le service interne est joignable, puis affichez la redirection active :

```powershell
Test-NetConnection -ComputerName 10.0.0.245 -Port 5000
netsh interface portproxy show v4tov4
```

Si la destination répond et que la règle figure dans la liste, testez depuis votre appareil distant. Le service est désormais accessible via l’adresse et le port de l’hôte de rebond :

```powershell
Test-NetConnection -ComputerName 100.100.10.10 -Port 5000
```

Ouvrez ensuite `http://100.100.10.10:5000` dans le navigateur. Si vous avez besoin de plusieurs ports pour le même service, par exemple 5000 et 5001 pour HTTP et HTTPS, créez une règle portproxy distincte et l’autorisation de pare-feu correspondante pour chaque port.

## Aperçu de type manpage

Les principaux sous-commandes de `netsh interface portproxy` :

<details class="options-details">
<summary>Aperçu des options</summary>

| Commande | Objectif |
|---|---|
| `add v4tov4 …` | Créer une redirection (listenaddress/listenport → connectaddress/connectport) |
| `show v4tov4` | Afficher les redirections IPv4 actives |
| `show all` | Afficher toutes les redirections de toutes les variantes de protocole |
| `delete v4tov4 listenaddress=… listenport=…` | Supprimer une redirection |
| `reset` | Supprimer toutes les règles portproxy |

</details>

Les règles sont enregistrées dans le Registre sous `HKLM\SYSTEM\CurrentControlSet\Services\PortProxy` et survivent à un redémarrage. Elles ne sont visibles que via `netsh` ou directement dans le Registre, et non dans l’interface graphique du pare-feu.

## Alternatives

`portproxy` est pratique si l’hôte de rebond est déjà sous Windows et que vous ne voulez rien installer. Deux alternatives résolvent le même problème avec des caractéristiques différentes.

Un tunnel SSH avec redirection locale (`ssh -L 5000:10.0.0.245:5000 benutzer@jumphost`) chiffre le trajet jusqu’à l’hôte de rebond lui-même et fonctionne sur plusieurs plateformes. Il nécessite un serveur SSH sur l’hôte de rebond et n’existe que tant que la session SSH reste ouverte.

Un routeur de sous-réseau Tailscale (`tailscale up --advertise-routes=10.0.0.0/24`) rend l’ensemble du sous-réseau interne accessible à vos appareils VPN. Vous adressez alors directement le service interne à son adresse IP réelle, sans redirection par port. C’est la solution la plus directe si vous souhaitez atteindre plusieurs appareils internes, mais elle requiert l’approbation de la route dans l’administration Tailscale.

## Limites

Une redirection de port avec portproxy résout l’accès, mais elle présente des limites claires que vous devriez connaître avant de l’utiliser :

- **TCP uniquement.** `portproxy` ne redirige que TCP. Les services nécessitant UDP (DNS, de nombreux protocoles VPN et de jeu, certaines transmissions vidéo) ne peuvent pas être pris en charge ainsi.
- **Pas de chiffrement supplémentaire.** La redirection copie les octets sans les modifier. La confidentialité du trajet est assurée uniquement par le VPN via lequel vous atteignez l’hôte de rebond. Sur un réseau de transport non chiffré, le trafic ne serait pas protégé.
- **Avertissement de certificat avec HTTPS via l’adresse IP.** Si vous redirigez un service HTTPS et l’appelez via l’adresse IP de l’hôte de rebond, le certificat de destination ne correspond pas à l’adresse appelée. Le navigateur affiche un avertissement. Cela peut être acceptable pour un test rapide, mais pas pour une utilisation durable.
- **Redirections et adresses absolues.** Certaines interfaces web redirigent elles-mêmes vers leur nom d’hôte ou un autre port, ou construisent des liens absolus avec leur adresse interne. L’accès via l’hôte de rebond échoue alors, même si la redirection est en place. Ces services nécessitent un véritable proxy inverse plutôt qu’un simple relais de port.
- **Liaison à une adresse qui doit exister au démarrage.** Si la règle se lie à une `listenaddress` précise, cette adresse doit être présente au démarrage du service. Si l’interface VPN ne s’active que plus tard, la liaison peut échouer jusqu’à ce que le service ou la règle soit réinitialisé.
- **Un accès supplémentaire au réseau interne.** Chaque redirection constitue un chemin depuis l’extérieur vers un service interne. Limitez strictement le pare-feu à la plage VPN, liez-vous à l’adresse VPN et supprimez la redirection dès que vous n’en avez plus besoin.

## Supprimer la configuration

Une fois le travail terminé, supprimez la redirection et la règle de pare-feu :

```powershell
netsh interface portproxy delete v4tov4 listenaddress=100.100.10.10 listenport=5000
Remove-NetFirewallRule -DisplayName "NAS-Proxy (VPN)"
```

Une redirection de port est un outil destiné à un accès ciblé et limité dans le temps, et non à un canal durablement ouvert. Pour exploiter durablement un service interne via Internet, un proxy inverse avec un certificat valide ou un VPN avec routage de sous-réseau constitue une solution plus propre.

## Sources

1.  [netsh interface portproxy (Microsoft Learn)](https://learn.microsoft.com/en-us/windows-server/networking/technologies/netsh/netsh-interface-portproxy): Référence des sous-commandes, des variantes de protocole et de la dépendance au service IP Helper.

2.  [New-NetFirewallRule (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/netsecurity/new-netfirewallrule): Paramètres de la règle de pare-feu, y compris la restriction aux plages d’adresses via RemoteAddress.

3.  [Tailscale: Subnet routers](https://tailscale.com/kb/1019/subnets): Rendre un sous-réseau entier accessible via le VPN, comme alternative à la redirection par port.
