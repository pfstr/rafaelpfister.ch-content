---
title: "Split tunneling VPN pour Microsoft Teams : faire passer le trafic multimédia hors du tunnel"
navTitle: "Split tunneling Teams"
description: "Les appels Teams via un VPN souffrent de latence, de gigue et du détour par la passerelle VPN. Cet article indique quels réseaux et ports Microsoft sont responsables du trafic multimédia, pourquoi le split tunneling basé sur les adresses IP est préférable à l’exclusion d’applications et comment le mettre en œuvre avec des VPN grand public, WireGuard, OpenVPN et des clients d’entreprise."
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
translationOf: vpn-split-tunneling-microsoft-teams
translationSourceHash: 9a96126164a12f1f411065507313733f451161a25c0cd051778cc306fd32ed4c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:25:30.504Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/split-tunneling-vpn-pour-microsoft-teams-faire-passer-le-trafic-multimedia-hors-du-tunnel
---

Un appel Teams via une connexion VPN est souvent de moins bonne qualité que sans VPN : la voix se coupe, la vidéo saccade et les partages d’écran s’affichent avec retard. La cause réside généralement dans le détour que le trafic en temps réel effectue via le tunnel VPN, et non dans Teams lui-même. Microsoft recommande donc depuis des années d’acheminer directement le trafic multimédia Teams vers Internet, en contournant le VPN grâce au split tunneling. Cette approche fonctionne avec pratiquement tous les produits VPN, du client grand public à la passerelle d’entreprise ; seule la configuration diffère dans les détails.

## Pourquoi le trafic en temps réel souffre dans le tunnel

L’audio et la vidéo Teams utilisent SRTP, un protocole basé sur UDP qui dépend d’une faible latence et d’une gigue réduite. Microsoft indique comme valeurs cibles moins de 100 ms de temps aller-retour jusqu’au point d’entrée réseau Microsoft le plus proche et moins de 30 ms de gigue. Un tunnel VPN dégrade ces deux valeurs à plusieurs égards.

Premièrement, le tunnel allonge le trajet : au lieu de passer directement par le point d’entrée Microsoft géographiquement le plus proche, le trafic passe d’abord par la passerelle VPN, qui peut se trouver dans le centre de données du fournisseur ou de l’entreprise, puis seulement ensuite chez Microsoft. Deuxièmement, la couche de chiffrement supplémentaire consomme du temps de calcul et augmente la surcharge par paquet ; le flux multimédia est déjà chiffré avec SRTP, et le chiffrement VPN s’ajoute comme seconde couche. Troisièmement, la passerelle VPN constitue un goulot d’étranglement partagé : aux heures de pointe, tous les utilisateurs se partagent sa bande passante et ses tampons de paquets, ce qui génère précisément la gigue à laquelle le trafic en temps réel réagit le plus fortement. Quatrièmement, certaines configurations VPN bloquent entièrement UDP ou imposent TCP ; Teams bascule alors vers TCP 443, ce qui dégrade encore la qualité, car les retransmissions TCP ne conviennent pas aux médias en temps réel.

Pour le reste du trafic Teams (connexion, chat, accès aux fichiers), tout cela joue un rôle minime, car il n’est pas sensible au temps réel. Il suffit donc d’exclure spécifiquement le trafic multimédia.

## Les réseaux et ports pertinents

Microsoft publie tous les points de terminaison Microsoft 365 dans un format lisible par machine et les répartit dans les catégories Optimize, Allow et Default. La catégorie Optimize est pertinente pour le split tunneling : elle comprend les quelques points de terminaison critiques pour la latence, avec des réseaux IP fixes, qui représentent ensemble la majeure partie du volume. Pour les médias Teams, il s’agit de l’ID de point de terminaison 11 de la liste officielle. L’ID 12 comprend les mêmes réseaux IPv4 ainsi que d’autres réseaux IPv6 pour TCP 443 et 80, ainsi qu’UDP 443, mais relève de la catégorie Allow. Le réseau auparavant également indiqué `13.107.64.0/18` a été retiré par Microsoft des deux entrées en 2024 (version 2024053000 pour l’ID 12, 2024073100 pour l’ID 11) :

| Réseau | Protocole et ports | Objectif |
|---|---|---|
| `52.112.0.0/14` | UDP 3478 à 3481 (ID 11), TCP 443 (ID 12) | Médias Teams et relais de transport |
| `52.122.0.0/15` | UDP 3478 à 3481 (ID 11), TCP 443 (ID 12) | Médias Teams et relais de transport |
| `2603:1063::/38` | UDP 3478 à 3481 (ID 11), TCP 443 (ID 12) | Mêmes services via IPv6 |

Les quatre ports UDP correspondent aux classes de médias audio (3478), vidéo (3479 et 3480) et partage d’écran (3481) ; TCP 443 est la voie de repli. Si IPv6 est utilisé, le réseau IPv6 doit également être exclu, sans quoi une partie des connexions repassera malgré tout par le tunnel.

Ces réseaux sont délibérément stables : Microsoft annonce les modifications des points de terminaison Optimize via le service web Endpoint et maintient la liste courte, précisément afin que les entreprises puissent les intégrer dans des règles de routage et de pare-feu. Néanmoins, une vérification occasionnelle par rapport à la liste officielle doit faire partie de la routine d’exploitation.

## Basé sur l’application ou sur l’adresse IP : deux approches aux atouts inégaux

De nombreux clients VPN proposent deux types de split tunneling : des exceptions par application ou des exceptions par adresse IP de destination.

L’exclusion d’application paraît évidente, mais elle présente deux faiblesses avec Teams. Le nouveau Teams est une application WebView2 : le processus principal s’appelle `ms-teams.exe`, mais une partie du trafic passe par `msedgewebview2.exe`. Exclure uniquement le processus principal ne couvre donc pas tout le trafic ; exclure également WebView2 fait aussi contourner le tunnel au trafic d’autres applications WebView2, comme le nouveau Outlook. Et pour Teams dans le navigateur, l’exclusion d’application n’aide pas du tout, à moins d’exclure le navigateur entier, ce qui ferait contourner le VPN à l’ensemble du trafic web.

L’exclusion basée sur l’adresse IP agit en revanche au niveau réseau et ne dépend donc pas du fait que le trafic provienne de l’application Teams, de WebView2 ou d’un onglet de navigateur. Elle exclut exactement ce qui est critique pour la latence et laisse dans le tunnel la connexion, le chat et le reste du trafic web. Pour Teams, l’approche basée sur l’adresse IP est donc le meilleur choix ; l’exclusion d’application convient en complément lorsqu’il faut réellement que tout le trafic Teams contourne le VPN.

## Mise en œuvre dans les produits VPN courants

Le principe est partout le même : les deux réseaux IPv4, et le réseau IPv6 si nécessaire, sont exclus du tunnel afin que les routes du système d’exploitation vers ces destinations pointent vers l’interface physique.

**VPN grand public (Proton VPN, NordVPN, Surfshark et similaires) :** Les clients Windows et Android proposent généralement un menu tel que « Split Tunneling » avec une liste d’exclusion pour les adresses IP ou les sous-réseaux. Saisissez-y les réseaux en notation CIDR et rétablissez la connexion VPN pour que les routes prennent effet. Sur macOS et iOS, cette fonction est absente chez la plupart des fournisseurs, car les API système n’y permettent pas le split tunneling contrôlé par application sous cette forme.

**WireGuard :** WireGuard ne connaît pas de liste d’exclusion, mais uniquement le paramètre `AllowedIPs` qui définit ce qui passe dans le tunnel. Les exceptions sont créées en remplaçant `0.0.0.0/0` par la liste de tous les réseaux qui ne contiennent pas la plage à exclure. Personne ne calcule cette liste complémentaire à la main ; des calculateurs en ligne comme WireGuard AllowedIPs Calculator prennent `0.0.0.0/0` comme base, les réseaux Microsoft comme « Disallowed IPs » et fournissent la ligne prête à l’emploi pour le fichier de configuration.

**OpenVPN :** Lorsque `redirect-gateway` est actif, les routes plus spécifiques prévalent. Deux lignes supplémentaires dans la configuration cliente font contourner le tunnel aux réseaux Microsoft :

```text
route 52.112.0.0 255.252.0.0 net_gateway
route 52.122.0.0 255.254.0.0 net_gateway
```

`net_gateway` représente ici la passerelle par défaut du réseau local, et non la passerelle VPN.

**Clients d’entreprise (Cisco Secure Client/AnyConnect, Palo Alto GlobalProtect, Fortinet FortiClient) :** Ici, l’entreprise configure les exceptions de manière centralisée : chez Cisco sous forme de liste « Split Exclude » dans la stratégie de groupe, chez GlobalProtect sous forme d’« Exclude Access Route ». Microsoft documente explicitement cette procédure comme modèle recommandé pour le trafic Microsoft 365 et fournit la liste Optimize via le service web Endpoint, ce qui permet de maintenir les exceptions à jour automatiquement. En tant qu’employé derrière un VPN d’entreprise, vous ne pouvez donc pas définir vous-même l’exception, mais devez en faire la demande auprès de l’équipe réseau ; le document Microsoft correspondant constitue une bonne base d’argumentation.

**Outils intégrés à Windows :** Avec une connexion VPN configurée à l’aide des outils intégrés de Windows en mode split (`Set-VpnConnection -SplitTunneling $true`), seuls les réseaux enregistrés via `Add-VpnConnectionRoute` passent dans le tunnel. Tant que les réseaux Microsoft n’y apparaissent pas, ils passent automatiquement en direct ; une exclusion explicite est alors inutile.

## Arbitrage de sécurité : ce qui contourne le tunnel

Le split tunneling assouplit délibérément le principe consistant à faire passer tout le trafic par le tunnel. Avant la mise en œuvre, vous devriez clarifier trois points.

Votre propre adresse IP publique devient visible pour Microsoft, car c’est précisément l’objectif : le flux multimédia doit emprunter le chemin le plus court. Si vous utilisez un VPN principalement pour masquer votre localisation, vous renoncez à cette protection pour les appels Teams. Le contenu n’en est pas affecté, car SRTP chiffre le flux multimédia de bout en bout entre le client et l’infrastructure Microsoft.

En entreprise, la passerelle de sécurité centrale perd sa visibilité sur le trafic exclu : l’inspection TLS, les signatures IDS et l’analyse des volumes ne s’appliquent plus à ces réseaux. Comme l’exception est limitée à quelques réseaux clairement attribués à Microsoft et à des ports définis, Microsoft considère que ce risque résiduel est faible ; les points de terminaison Optimize sont précisément sélectionnés à cette fin. Une exception générale pour des applications entières, voire pour le navigateur, présente en revanche une surface d’attaque nettement plus grande et doit être évitée en entreprise.

Enfin, il y a le Kill Switch : certains clients VPN n’appliquent les exceptions de split tunneling qu’après le rétablissement de la connexion ou se comportent différemment lorsque le Kill Switch est actif. Après toute modification de la liste d’exclusion, il convient donc de rétablir la connexion et d’effectuer un test de contrôle.

## Vérification : le trafic multimédia passe-t-il vraiment en direct ?

Vous pouvez vérifier si l’exception s’applique à deux niveaux. Au niveau du routage, PowerShell indique quelle interface Windows choisit pour une destination dans les réseaux Microsoft :

```powershell
Find-NetRoute -RemoteIPAddress 52.112.1.1 |
  Select-Object InterfaceAlias, NextHop
```

Si l’interface physique, Ethernet ou WLAN, apparaît ici à la place de l’adaptateur VPN, la route est correcte. Au niveau de l’application, Teams fournit lui-même la confirmation : pendant un appel, la qualité de l’appel, sous « Autres actions » dans la fenêtre d’appel, affiche le type de connexion négocié, le temps aller-retour et le taux de perte de paquets. Une baisse sensible du temps aller-retour après la modification et le type de connexion UDP plutôt que TCP sont les deux signes d’une exception fonctionnelle.

Si le trafic reste dans le tunnel malgré une route correcte, il vaut la peine de vérifier l’ordre des adaptateurs réseau et les particularités du client : certains clients VPN imposent à nouveau leurs routes avec une métrique plus faible après chaque établissement de connexion, et une liste d’exclusion obsolète ne devient visible que lorsque Microsoft ajoute un réseau. La comparaison avec la liste officielle des points de terminaison doit donc suivre le même rythme que les autres vérifications réseau récurrentes.

## Historique des modifications

- **25 octobre 2026 :** Correction : Microsoft a retiré le réseau 13.107.64.0/18 de la liste des points de terminaison en 2024 ; il a été supprimé du tableau et de l’exemple OpenVPN. Également corrigé : seul l’ID 11 relève de la catégorie Optimize, l’ID 12 relève de la catégorie Allow.

## Sources

1.  [Microsoft: Office 365 URLs and IP address ranges](https://learn.microsoft.com/en-us/microsoft-365/enterprise/urls-and-ip-address-ranges): liste officielle des points de terminaison ; les réseaux multimédias Teams figurent sous l’ID 11 dans la catégorie Optimize, l’ID 12 avec les mêmes réseaux dans la catégorie Allow.

2.  [Microsoft: Implementing VPN split tunneling for Microsoft 365](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-vpn-implement-split-tunnel): guide de mise en œuvre de Microsoft pour les VPN d’entreprise, y compris la justification de l’évaluation des risques et un exemple avec les réseaux multimédias Teams `52.112.0.0/14`, `52.122.0.0/15` et `2603:1063::/38`.

3.  [Microsoft: Microsoft 365 network connectivity principles](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-network-connectivity-principles): les principes à l’origine de la sortie Internet locale, y compris les valeurs cibles de latence pour les médias en temps réel.

4.  [Proton VPN: How to use split tunneling](https://protonvpn.com/support/protonvpn-split-tunneling/): exemple de client grand public avec split tunneling basé sur les adresses IP et les applications sous Windows et Android.

5.  [WireGuard AllowedIPs Calculator](https://www.procustodibus.com/blog/2021/03/wireguard-allowedips-calculator/): calculateur de liste complémentaire lorsqu’il faut représenter des exceptions via AllowedIPs.

6.  [Microsoft 365 IP and URL web service: Änderungen Worldwide](https://endpoints.office.com/changes/worldwide/2024010100?clientrequestid=b10c5ed1-bad1-445f-b386-b919946339a7): journal des modifications de la liste des points de terminaison, notamment le retrait de `13.107.64.0/18` des ID 12 (version 2024053000) et 11 (version 2024073100).
