---
title: "Split Tunneling VPN pour Microsoft Teams : faire passer le trafic multimédia hors du tunnel"
navTitle: "Split Tunneling Teams"
description: "Les appels Teams via un VPN souffrent de latence, de gigue et du détour par la passerelle VPN. Cet article explique quels réseaux et ports Microsoft sont responsables du trafic multimédia, pourquoi le Split Tunneling basé sur les adresses IP est préférable à l’exclusion d’applications et comment le mettre en œuvre dans les VPN grand public, WireGuard, OpenVPN et les clients d’entreprise."
date: "2026-08-26"
kategorie: "Microsoft Teams"
timeToRead: "8 min de lecture"
themen:
  - microsoft-teams
  - microsoft-365-exchange
produkte:
  - "teams"
protokolle:
  - "tcp"
hauptthema: "microsoft-teams"
slug: "split-tunneling-vpn-pour-microsoft-teams-faire-passer-le-trafic-multimedia-hors-du-tunnel"
translationId: "article-d15f1e7ff6af231c"
aiPrompt: |
  Du bist mein Netzwerk-Assistent. Ich will Microsoft-Teams-Medienverkehr per Split Tunneling an meinem VPN vorbeiführen. Hilf mir Schritt für Schritt: 1. Frage mich, welchen VPN-Client ich einsetze (Consumer-VPN, WireGuard, OpenVPN, Enterprise-Client) und auf welchem Betriebssystem. 2. Nenne mir die passende Konfiguration für die drei Optimize-Netze 13.107.64.0/18, 52.112.0.0/14 und 52.122.0.0/15 (UDP 3478 bis 3481, TCP 443). 3. Erkläre mir, wie ich mit Find-NetRoute oder der Anrufintegrität in Teams prüfe, ob der Medienverkehr tatsächlich am Tunnel vorbeiläuft. 4. Weise mich auf die Sicherheitsabwägungen hin, bevor ich die Ausnahme produktiv setze.
translationOf: vpn-split-tunneling-microsoft-teams
url: https://rafaelpfister.ch/fr/blog/split-tunneling-vpn-pour-microsoft-teams-faire-passer-le-trafic-multimedia-hors-du-tunnel
translationSourceHash: 95e3cefa4946676022602866d6ef21ab92ef25ec8c5dd3ff4ab0219ba718a880
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:31:38.253Z
translationReview: automatic
---

Un appel Teams via une connexion VPN sonne souvent moins bien que sans VPN : la voix se coupe, la vidéo saccade, les partages d’écran s’affichent avec retard. La cause réside généralement dans le détour que le trafic temps réel effectue via le tunnel VPN, et non dans Teams lui-même. Microsoft recommande donc depuis des années de faire passer directement le trafic multimédia Teams vers Internet, hors du VPN, au moyen du Split Tunneling. Cette approche fonctionne avec pratiquement tous les produits VPN, du client grand public à la passerelle d’entreprise ; seule la configuration diffère dans les détails.

## Pourquoi le trafic temps réel souffre dans le tunnel

L’audio et la vidéo Teams utilisent SRTP, un protocole basé sur UDP qui dépend d’une faible latence et d’une gigue réduite. Microsoft indique comme objectifs un temps aller-retour inférieur à 100 ms jusqu’au point d’entrée réseau Microsoft le plus proche et une gigue inférieure à 30 ms. Un tunnel VPN dégrade ces deux valeurs à plusieurs titres.

Premièrement, le tunnel allonge le trajet : au lieu de rejoindre directement le point d’entrée Microsoft géographiquement le plus proche, le trafic passe d’abord par la passerelle VPN, qui peut se trouver dans le centre de données du fournisseur ou de l’entreprise, puis seulement ensuite chez Microsoft. Deuxièmement, la couche de chiffrement supplémentaire consomme du temps de calcul et augmente la surcharge par paquet ; le flux multimédia étant déjà chiffré avec SRTP, le chiffrement VPN s’y ajoute comme deuxième couche. Troisièmement, la passerelle VPN constitue un goulot d’étranglement partagé : aux heures de pointe, tous les utilisateurs se partagent sa bande passante et ses tampons de paquets, ce qui génère précisément la gigue à laquelle le trafic temps réel est le plus sensible. Quatrièmement, certaines configurations VPN bloquent entièrement UDP ou imposent TCP ; Teams bascule alors sur TCP 443, ce qui dégrade encore la qualité, car les retransmissions TCP ne conviennent pas aux médias temps réel.

Pour le reste du trafic Teams (authentification, chat, accès aux fichiers), tout cela joue à peine un rôle, car il n’est pas sensible au temps réel. Il suffit donc d’exclure spécifiquement le trafic multimédia.

## Les réseaux et ports concernés

Microsoft publie tous les points de terminaison Microsoft 365 dans un format lisible par machine et les répartit dans les catégories Optimize, Allow et Default. La catégorie Optimize est pertinente pour le Split Tunneling : elle regroupe les quelques points de terminaison critiques pour la latence, avec des réseaux IP fixes, qui représentent ensemble la majeure partie du volume. Pour les médias Teams, il s’agit des identifiants de point de terminaison 11 et 12 de la liste officielle :

| Réseau | Protocole et ports | Objectif |
|---|---|---|
| `13.107.64.0/18` | UDP 3478 à 3481, TCP 443 | Médias Teams (audio, vidéo, partage d’écran) |
| `52.112.0.0/14` | UDP 3478 à 3481, TCP 443 | Médias Teams et relais de transport |
| `52.122.0.0/15` | UDP 3478 à 3481, TCP 443 | Médias Teams et relais de transport |
| `2603:1063::/38` | UDP 3478 à 3481, TCP 443 | Les mêmes services via IPv6 |

Les quatre ports UDP correspondent aux catégories de médias audio (3478), vidéo (3479 et 3480) et partage d’écran (3481) ; TCP 443 est la voie de repli. Si IPv6 est utilisé, il convient d’exclure également le réseau IPv6, sinon une partie des connexions repassera malgré tout par le tunnel.

Ces réseaux sont volontairement stables : Microsoft annonce les modifications des points de terminaison Optimize via le service web Endpoint et maintient la liste courte, précisément pour que les entreprises puissent les intégrer dans des règles de routage et de pare-feu. Un rapprochement périodique avec la liste officielle doit néanmoins faire partie des procédures d’exploitation.

## Basé sur l’application ou sur l’adresse IP : deux approches aux atouts inégaux

De nombreux clients VPN proposent deux types de Split Tunneling : des exceptions par application ou des exceptions par adresse IP de destination.

L’exclusion d’application semble évidente, mais présente deux faiblesses avec Teams. Le nouveau Teams est une application WebView2 : le processus principal s’appelle `ms-teams.exe`, mais une partie du trafic passe par `msedgewebview2.exe`. Exclure uniquement le processus principal ne couvre donc pas tout le trafic ; exclure aussi WebView2 fait également passer hors du tunnel le trafic d’autres applications WebView2 (comme le nouveau Outlook). Et pour Teams dans le navigateur, l’exclusion d’application ne sert à rien, sauf à exclure le navigateur entier, ce qui contourne le VPN pour tout le trafic web.

L’exclusion basée sur l’adresse IP agit en revanche au niveau réseau et ne dépend donc pas du fait que le trafic provienne de l’application Teams, de WebView2 ou d’un onglet de navigateur. Elle exclut exactement ce qui est sensible à la latence et laisse l’authentification, le chat et le reste du trafic web dans le tunnel. Pour Teams, l’approche basée sur l’adresse IP est donc le meilleur choix ; l’exclusion d’application peut servir de complément si l’ensemble du trafic Teams doit réellement contourner le VPN.

## Mise en œuvre dans les produits VPN courants

Le principe est partout le même : les trois réseaux IPv4 (et, si nécessaire, le réseau IPv6) sont exclus du tunnel afin que les routes du système d’exploitation vers ces destinations pointent vers l’interface physique.

**VPN grand public (Proton VPN, NordVPN, Surfshark et similaires) :** Les clients Windows et Android proposent généralement un menu tel que « Split Tunneling » avec une liste d’exclusion pour les adresses IP ou les sous-réseaux. Saisissez-y les trois réseaux en notation CIDR et rétablissez la connexion VPN afin que les routes soient appliquées. Sur macOS et iOS, cette fonction est absente chez la plupart des fournisseurs, car les API système n’y permettent pas le Split Tunneling contrôlé par application sous cette forme.

**WireGuard :** WireGuard ne connaît pas de liste d’exclusion, seulement le paramètre `AllowedIPs` qui définit ce qui entre dans le tunnel. Les exceptions sont créées en remplaçant `0.0.0.0/0` par la liste de tous les réseaux qui ne contiennent pas la plage à exclure. Personne ne calcule cette liste complémentaire à la main ; des calculateurs en ligne tels que WireGuard AllowedIPs Calculator utilisent `0.0.0.0/0` comme base, les trois réseaux Microsoft comme « Disallowed IPs » et fournissent la ligne prête à l’emploi pour le fichier de configuration.

**OpenVPN :** Lorsque `redirect-gateway` est actif, les routes les plus spécifiques l’emportent. Trois lignes supplémentaires dans la configuration client font passer les réseaux Microsoft hors du tunnel :

```text
route 13.107.64.0 255.255.192.0 net_gateway
route 52.112.0.0 255.252.0.0 net_gateway
route 52.122.0.0 255.254.0.0 net_gateway
```

`net_gateway` désigne ici la passerelle par défaut du réseau local, et non la passerelle VPN.

**Clients d’entreprise (Cisco Secure Client/AnyConnect, Palo Alto GlobalProtect, Fortinet FortiClient) :** Ici, l’entreprise configure les exceptions de manière centralisée : chez Cisco sous forme de liste « Split Exclude » dans la stratégie de groupe, chez GlobalProtect sous forme d’« Exclude Access Route ». Microsoft documente explicitement cette démarche comme modèle recommandé pour le trafic Microsoft 365 et fournit la liste Optimize via le service web Endpoint, ce qui permet de maintenir les exceptions à jour automatiquement. Les employés derrière un VPN d’entreprise ne peuvent donc pas définir eux-mêmes l’exception, mais doivent la demander à l’équipe réseau ; le document Microsoft correspondant constitue une base d’argumentation appropriée.

**Outils intégrés à Windows :** Pour une connexion VPN configurée avec les outils intégrés de Windows en mode Split (`Set-VpnConnection -SplitTunneling $true`), seuls les réseaux ajoutés via `Add-VpnConnectionRoute` passent dans le tunnel. Tant que les réseaux Microsoft n’y apparaissent pas, ils passent automatiquement en direct ; une exclusion explicite est alors inutile.

## Compromis de sécurité : ce qui passe hors du tunnel

Le Split Tunneling constitue un assouplissement délibéré du principe consistant à faire passer tout le trafic par le tunnel. Avant la mise en œuvre, vous devriez clarifier trois points.

Votre propre adresse IP publique devient visible pour Microsoft, car c’est précisément l’objectif : le flux multimédia doit emprunter le chemin le plus court. Quiconque utilise principalement un VPN pour masquer sa localisation renonce à cette protection pour les appels Teams. Le contenu reste inchangé, car SRTP chiffre le flux multimédia de bout en bout entre le client et l’infrastructure Microsoft.

En environnement d’entreprise, la passerelle de sécurité centrale perd sa visibilité sur le trafic exclu : l’inspection TLS, les signatures IDS et l’analyse des volumes ne s’appliquent plus à ces réseaux. Comme l’exception est limitée à quelques réseaux fixes clairement attribués à Microsoft, avec des ports définis, Microsoft estime que ce risque résiduel est faible ; les points de terminaison Optimize sont précisément sélectionnés à cette fin. Une exclusion globale d’applications entières, voire du navigateur, présente en revanche une surface d’attaque nettement plus importante et doit être évitée en environnement d’entreprise.

Enfin, le Kill Switch : certains clients VPN n’appliquent les exceptions de Split Tunneling qu’après le rétablissement de la connexion ou se comportent différemment lorsque le Kill Switch est actif. Après chaque modification de la liste d’exclusion, il faut donc rétablir la connexion et effectuer un test de contrôle.

## Contrôle : le trafic multimédia passe-t-il vraiment en direct ?

Il est possible de vérifier si l’exception fonctionne à deux niveaux. Au niveau du routage, PowerShell indique quelle interface Windows choisit pour une destination dans les réseaux Microsoft :

```powershell
Find-NetRoute -RemoteIPAddress 52.112.1.1 |
  Select-Object InterfaceAlias, NextHop
```

Si l’interface physique (Ethernet ou Wi-Fi) apparaît à la place de l’adaptateur VPN, la route est correcte. Au niveau applicatif, Teams fournit lui-même la confirmation : pendant un appel, l’intégrité de l’appel (sous « Autres actions » dans la fenêtre d’appel) affiche le type de connexion négocié, le temps aller-retour et le taux de perte de paquets. Un temps aller-retour qui diminue sensiblement après le changement et le type de connexion UDP au lieu de TCP sont les deux indicateurs d’une exception fonctionnelle.

Si le trafic reste dans le tunnel malgré une route correcte, il vaut la peine de vérifier l’ordre des adaptateurs réseau et les particularités du client : certains clients VPN réimposent leurs routes avec une métrique plus faible après chaque établissement de connexion, et une liste d’exclusion obsolète ne se remarque que lorsque Microsoft ajoute un réseau. Le rapprochement avec la liste officielle des points de terminaison doit donc suivre le même rythme que les autres contrôles réseau récurrents.

## Sources

1.  [Microsoft: Office 365 URLs and IP address ranges](https://learn.microsoft.com/en-us/microsoft-365/enterprise/urls-and-ip-address-ranges): liste officielle des points de terminaison ; les réseaux de médias Teams figurent sous les identifiants 11 et 12 dans la catégorie Optimize.

2.  [Microsoft: Implementing VPN split tunneling for Microsoft 365](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-vpn-implement-split-tunnel): guide de mise en œuvre de Microsoft pour les VPN d’entreprise, y compris la justification de l’évaluation des risques.

3.  [Microsoft: Microsoft 365 network connectivity principles](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-network-connectivity-principles): les principes derrière la sortie Internet locale, y compris les objectifs de latence pour les médias temps réel.

4.  [Proton VPN: How to use split tunneling](https://protonvpn.com/support/protonvpn-split-tunneling/): exemple d’un client grand public avec Split Tunneling basé sur les adresses IP et les applications sous Windows et Android.

5.  [WireGuard AllowedIPs Calculator](https://www.procustodibus.com/blog/2021/03/wireguard-allowedips-calculator/): calculateur pour la liste complémentaire lorsque les exceptions doivent être représentées via AllowedIPs.
