---
title: "Exchange Online limite et bloque les serveurs Exchange 2016 et 2019 obsolètes dès septembre 2026 : fonctionnement du Transport Enforcement"
navTitle: "EXO-Enforcement 09/2026"
description: "À partir de la deuxième semaine de septembre 2026, Exchange Online exige au minimum la SU d’octobre 2025 des serveurs hybrides, faute de quoi le flux de messagerie est limité puis bloqué. Contexte du Transport Enforcement en place depuis 2023, niveaux d’escalade avec codes SMTP, rapport dans l’Admin Center, pause de 90 jours via PowerShell et raisons pour lesquelles la prochaine hausse ne laissera passer que les clients ESU et Exchange SE."
date: "2026-09-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "9 min de lecture"
themen:
  - exchange-onprem-hybrid
  - exchange-updates
produkte:
  - "exchange-hybrid"
  - "exchange-online"
  - "hybrid-mailfluss"
  - "exchange-updates"
protokolle:
  - "smtp"
  - "migration"
  - "releases"
slug: "exchange-online-limite-et-bloque-les-versions-obsoletes-d-exchange-2016-et-2019-a-partir-de"
translationId: "article-fff0c5efce59ef76"
draft: false
translationOf: exchange-online-transport-enforcement-hybrid-server
translationSourceHash: d2bb7589193052361cd6a083ace0d3dbc58552e5e66691c352cd82b693fae6bd
translationModel: gpt-5.6-terra
translatedAt: 2026-10-27T09:44:05.942Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/exchange-online-limite-et-bloque-les-versions-obsoletes-d-exchange-2016-et-2019-a-partir-de
---

# Exchange Online limite et bloque les serveurs Exchange 2016 et 2019 obsolètes dès septembre 2026 : fonctionnement du Transport Enforcement

Le 2 septembre 2026, l’équipe Exchange a annoncé le relèvement de la version minimale requise pour Exchange 2016 et Exchange 2019 dans le flux de messagerie hybride. À partir de la deuxième semaine de septembre 2026, Exchange Online exigera des serveurs qui remettent des messages via un connecteur entrant de type `OnPremises` au minimum le niveau de la dernière mise à jour de sécurité publique d’octobre 2025. Tout niveau inférieur sera limité puis bloqué. En bref : les organisations qui n’ont pas corrigé leurs serveurs hybrides depuis octobre 2025 perdront progressivement, au cours des prochaines semaines, la remise des messages vers Exchange Online. Et le prochain relèvement, que Microsoft annonce pour les mois à venir, se situera au-dessus de toute mise à jour disponible publiquement : seuls les clients du programme ESU payant ou les environnements exécutant Exchange Server Subscription Edition (SE) satisferont alors à l’exigence.

L’annonce elle-même est brève. Sa portée concrète découle du système d’enforcement que Microsoft met progressivement en place depuis 2023 : quelles réponses SMTP votre serveur recevra, comment vérifier son état dans l’Admin Center et via PowerShell, et quelles options restent pendant la transition jusqu’à la fin du programme ESU en octobre 2026.

## Ce qui s’applique à partir de la deuxième semaine de septembre 2026

Le nouveau seuil minimal correspond aux mises à jour de sécurité du 14 octobre 2025. Il s’agissait du dernier Patch Tuesday lors duquel Microsoft a publié publiquement des mises à jour pour Exchange 2016 et 2019 ; toutes les SU depuis décembre 2025 ne sont disponibles que via le programme ESU.

| Version | Niveau minimal | KB | Build |
|---|---|---|---|
| Exchange 2019 CU15 | SU d’octobre 2025 (CU15 SU5) | KB5066367 | 15.2.1748.39 |
| Exchange 2016 CU23 | SU d’octobre 2025 (CU23 SU19) | KB5066369 | 15.1.2507.61 |

Une SU d’octobre 2025 existe également pour Exchange 2019 CU14 (KB5066368, build 15.2.1544.36). Toutefois, l’article Microsoft indique explicitement CU15 SU5 comme version minimale ; CU14 n’est de toute façon plus un niveau recommandé depuis la publication de CU15 en février 2025. Prévoyez donc également le passage à CU15 pour CU14.

Trois précisions sont importantes :

- **Seul le flux de messagerie hybride est concerné.** Exchange Online vérifie la version des serveurs remettant des messages pour ceux qui arrivent via un connecteur entrant de type `OnPremises`. Il s’agit de la configuration hybride classique créée par le Hybrid Configuration Wizard. Les messages arrivant via une passerelle tierce ou un connecteur de type `Partner` ne passent pas par cet enforcement.
- **La version est lue dans les en-têtes.** Un serveur Exchange inscrit son build dans la ligne `Received` de chaque message qu’il transmet (`… with Microsoft SMTP Server … id 15.1.2507.61`). Exchange Online évalue cette information. C’est donc le niveau du serveur qui remet effectivement le message à Exchange Online qui compte, soit dans de nombreux environnements le serveur Edge Transport ou le serveur Mailbox doté du connecteur d’envoi vers `*.mail.protection.outlook.com`.
- **Exchange SE n’est pas concerné.** L’enforcement s’applique à Exchange 2016 et 2019 ; Exchange Server SE est au-dessus de tout seuil minimal tant qu’il est régulièrement corrigé.

## Contexte : le Transport Enforcement depuis 2023

L’annonce de septembre ne constitue pas une nouvelle mesure, mais l’étape suivante d’un système présenté par Microsoft en mars 2023 sous le titre «Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online». Microsoft définit comme «persistently vulnerable» tout serveur Exchange ayant atteint la fin de support ou restant non corrigé face à des vulnérabilités connues. L’objectif est de protéger les destinataires Exchange Online des messages provenant de serveurs susceptibles d’être compromis, tout en poussant les exploitants à corriger ou désactiver leurs serveurs.

Le système a été activé progressivement par version :

| Date | Version concernée |
|---|---|
| Août 2023 | Exchange 2007 |
| Septembre 2023 | Exchange 2010 |
| Décembre 2023 | Exchange 2013 |
| Mars 2024 | Exchange 2016 et 2019 (niveaux de SU très obsolètes) |
| Septembre 2026 | Exchange 2016 et 2019 : seuil minimal = SU d’octobre 2025 |
| «dans quelques mois» | Exchange 2016 et 2019 : seuil minimal supérieur à la dernière mise à jour publique |

Jusqu’à présent, le seuil minimal pour Exchange 2016 et 2019 concernait les niveaux «significantly behind on security updates». La nouveauté est que Microsoft fixe la limite à la dernière mise à jour publique et touche ainsi pour la première fois des serveurs qui étaient encore entièrement corrigés il y a moins d’un an.

## Les niveaux d’escalade

L’enforcement fonctionne selon trois mécanismes que Microsoft appelle «reporting», «throttling» et «blocking». Dès qu’un serveur passe sous le seuil minimal, un cycle de 90 jours commence. Les niveaux de l’article de référence de 2023 :

| Période | Mesure | Réponse SMTP |
|---|---|---|
| Jour 0 à 30 | Rapport uniquement dans l’Exchange Admin Center | aucune |
| Jour 30 à 40 | Limitation 5 minutes par heure | `450 4.7.230` |
| Jour 40 à 50 | Limitation 10 minutes par heure | `450 4.7.230` |
| Jour 50 à 60 | Limitation 20 minutes par heure | `450 4.7.230` |
| Jour 60 à 70 | Limitation 30 minutes par heure, plus blocage 5 minutes par heure | `450 4.7.230` et `550 5.7.230` |
| Jour 70 à 80 | Blocage 10 minutes par heure | `550 5.7.230` |
| Jour 80 à 90 | Blocage 20 minutes par heure | `550 5.7.230` |
| à partir du jour 90 | Blocage complet | `550 5.7.230` |

Les deux réponses sont formulées comme suit :

```text
450 4.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online throttled for n mins/hr.

550 5.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online blocked for n mins/hr.
```

La différence est déterminante pour l’exploitation. Avec `450`, Exchange Online refuse temporairement la connexion ; le serveur On-Premises conserve le message dans sa file d’attente et réessaie. Les utilisateurs ne remarquent d’abord que des retards ; dans le Queue Viewer ou dans `Get-Queue`, la file d’attente vers le connecteur d’envoi vers Exchange Online augmente avec l’état `Retry` et le message 4.7.230 comme `LastError`. Avec `550`, le refus est définitif : l’expéditeur reçoit un NDR avec le code 5.7.230 et le message est perdu, sauf s’il est envoyé à nouveau. Étant donné que le blocage n’est initialement actif que quelques minutes par heure, le symptôme semble d’abord sporadique : une partie des messages arrive, une autre échoue avec un NDR. En cas d’un tel schéma dans le suivi des messages, vérifiez d’abord la version avant de rechercher des problèmes de réseau ou de certificat.

L’annonce ne précise pas si Microsoft démarrera le cycle complet de 90 jours à partir de la deuxième semaine de septembre pour le nouveau seuil minimal ou s’il commencera déjà à un niveau ultérieur. L’article de référence indique que le système reprend, après une pause, au niveau précédemment atteint. Ne comptez donc pas sur un délai de grâce de 30 jours.

## Rapport dans l’Exchange Admin Center et via PowerShell

Exchange Online répertorie les serveurs On-Premises détectés avec leur version dans un rapport dédié : dans l’Exchange Admin Center, sous *Reports*, *Mail flow*, rapport sur les serveurs Exchange On-Premises connectés obsolètes («out-of-date connecting on-premises Exchange servers»). Pour chaque serveur, le rapport affiche le build détecté, s’il se situe sous le seuil minimal et à quel niveau d’enforcement il se trouve.

Exchange Online PowerShell fournit les mêmes informations :

```powershell
Connect-ExchangeOnline
Get-OnPremServerReportInfo
```

<details class="options-details">
<summary>Options expliquées</summary>

| Commande | Effet |
|---|---|
| `Connect-ExchangeOnline` | Ouvre la session vers Exchange Online (module `ExchangeOnlineManagement`). |
| `Get-OnPremServerReportInfo` | Répertorie les serveurs On-Premises détectés par Exchange Online avec leur build, statut d’enforcement et niveau. |

</details>

Le rapport ne connaît que les serveurs qui remettent réellement des messages à Exchange Online. Un serveur de gestion sans flux de messagerie ou une machine équipée uniquement des Management Tools n’y apparaît pas. Cela est sans importance pour l’enforcement, mais pas pour la sécurité : ces systèmes nécessitent également les SU.

## Mettre l’enforcement en pause : 90 jours par an

Pour les environnements qui ne peuvent pas atteindre le seuil minimal à court terme, Microsoft propose une pause. Elle peut être activée pour un total de 90 jours par an, d’un seul tenant ou en plusieurs périodes :

```powershell
Get-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer

New-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer `
  -NumberOfDays 30
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Get-TenantExemptionInfo` | Indique si une pause est active pour le tenant et sa durée. |
| `New-TenantExemptionInfo` | Crée une nouvelle pause. |
| `-BlockingScenario UnpatchedOnPremServer` | Sélectionne le scénario «serveur On-Premises obsolète» ; il n’existe actuellement aucun autre scénario pour ce cmdlet. |
| `-NumberOfDays 30` | Durée de la pause en jours. Le contingent est de 90 jours par an et la durée indiquée en est déduite. |

</details>

Deux caractéristiques de la pause sont importantes en pratique. Premièrement, l’enforcement reprend à l’expiration au niveau auquel il avait été arrêté ; la pause ne réinitialise pas le cycle de 90 jours. Deuxièmement, aucun cmdlet ne permet de mettre fin prématurément à une pause en cours : si vous créez une pause de 90 jours et terminez les correctifs après deux semaines, le contingent annuel est épuisé. Créez donc une pause aussi courte que possible et prolongez-la si nécessaire.

La pause n’est par ailleurs une solution que pour le seuil minimal actuel. Si Microsoft relève la limite dans quelques mois au-dessus de la dernière mise à jour publique, un contingent épuisé ne sera plus utile.

## Pourquoi le prochain relèvement constitue la véritable échéance

Exchange 2016 et 2019 sont hors support depuis le 14 octobre 2025. Microsoft a ensuite lancé deux périodes ESU payantes : la période 1 jusqu’en avril 2026, la période 2 de mai à octobre 2026. En annonçant la période 2 le 15 avril 2026, l’équipe Exchange a précisé qu’il n’y aurait pas de prolongation supplémentaire. Les SU de décembre 2025 à octobre 2026 (dernières en date : build 15.2.1748.55 pour 2019 CU15 et 15.1.2507.77 pour 2016 CU23, toutes deux du 13 octobre 2026) sont exclusivement disponibles pour les clients ESU et ne sont pas proposées publiquement au téléchargement.

La situation est donc la suivante :

- **Aujourd’hui**, un serveur équipé de la SU d’octobre 2025 satisfait au seuil minimal, avec ou sans ESU.
- **Lors du prochain relèvement**, le seuil minimal sera, selon Microsoft, supérieur au niveau d’octobre 2025. Sans contrat ESU, il n’existe aucun moyen légal d’atteindre ce niveau. Le flux de messagerie hybride de ces serveurs sera alors limité puis bloqué, quelle que soit la qualité de gestion du reste de l’environnement.
- **Le 31 octobre 2026**, la période 2 prend également fin. Microsoft a publié les dernières mises à jour pour Exchange 2016 et 2019 avec la SU d’octobre du 13 octobre 2026 ; il n’y aura ensuite plus de SU pour personne. Au plus tard le relèvement suivant touchera donc également les clients ESU.

Le programme ESU n’offre donc, dans le meilleur des cas, que quelques mois. Le seul état durable accepté par l’enforcement est Exchange Server SE. Microsoft a en outre annoncé qu’Exchange SE CU2 (initialement prévu pour le second semestre 2026 ; Microsoft ne donnant actuellement aucune date pour le CU1 qui le précède, aucune date n’est non plus fixée pour CU2) mettra fin à la coexistence avec Exchange 2016 et 2019 : l’installation échouera si des serveurs plus anciens sont détectés dans l’organisation. La migration est donc nécessaire non seulement en raison du flux de messagerie, mais aussi pour pouvoir continuer à installer des mises à jour pour SE.

Pour les environnements qui ne conservent Exchange On-Premises que pour gérer des attributs dans une configuration hybride, l’alternative consiste à supprimer le dernier serveur : depuis Exchange 2019 CU12, les attributs de destinataire peuvent être gérés avec les Management Tools sans serveur Exchange en fonctionnement. Il n’y a alors plus de flux de messagerie On-Premises et l’enforcement ne s’applique plus.

## Déterminer la version

`Get-ExchangeServer` n’affiche dans `AdminDisplayVersion` que le CU, pas la SU. La version de fichier de `ExSetup.exe` ou le [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), qui signale aussi les étapes manuelles manquantes, sont fiables. Pour un aperçu rapide de tous les serveurs :

```powershell
Get-ExchangeServer | ForEach-Object {
  $path = "\\$($_.Name)\C$\Program Files\Microsoft\Exchange Server\V15\bin\ExSetup.exe"
  [pscustomobject]@{
    Server  = $_.Name
    Version = (Get-Item $path).VersionInfo.ProductVersion
  }
}
```

<details class="options-details">
<summary>Options expliquées</summary>

| Élément | Effet |
|---|---|
| `Get-ExchangeServer` | Répertorie tous les serveurs Exchange de l’organisation. |
| `\\<Server>\C$\...\ExSetup.exe` | Chemin de partage administratif vers le fichier Setup ; adaptez-le si le chemin d’installation diffère. |
| `VersionInfo.ProductVersion` | Version de fichier correspondant au build SU installé (par exemple `15.1.2507.61`). |

</details>

Si la version est inférieure à `15.2.1748.39` (2019 CU15) ou à `15.1.2507.61` (2016 CU23), le serveur passera sous le seuil minimal à partir de la deuxième semaine de septembre.

## Procédure recommandée

1. **Inventoriez les niveaux** comme décrit ci-dessus, y compris les serveurs Edge Transport et les serveurs de gestion.

2. **Vérifiez le rapport dans Exchange Online.** `Get-OnPremServerReportInfo` indique quels serveurs Exchange Online voit réellement et si un niveau d’enforcement est déjà actif. Comparez la liste avec l’inventaire : les serveurs absents ne remettent pas de messages via le connecteur `OnPremises`.

3. **Installez au minimum la SU d’octobre 2025.** KB5066367 (2019 CU15) et KB5066369 (2016 CU23) restent disponibles publiquement dans le Microsoft Download Center. Les SU sont cumulatives ; un serveur au niveau d’août 2025 peut être mis directement à niveau vers octobre 2025. Pour CU14, installez d’abord CU15. Après l’installation, redémarrez, contrôlez l’état des services et exécutez de nouveau le Health Checker.

4. **Utilisez la pause uniquement comme transition.** Si la mise à jour ne peut pas être effectuée durant la première moitié de septembre, créez `New-TenantExemptionInfo` avec une durée courte et ne considérez pas la pause comme une réserve de planification pour le prochain relèvement.

5. **Planifiez la migration vers Exchange SE.** Sans contrat ESU, le prochain relèvement est l’échéance impérative ; avec ESU, c’est le 31 octobre 2026. Exchange 2019 CU15 peut être mis à niveau sur place vers SE ; Exchange 2016 nécessite le détour par une nouvelle installation de SE et le déplacement des boîtes aux lettres ou des rôles. Les organisations qui exploitent Exchange uniquement pour la gestion des attributs suppriment le dernier serveur et continuent avec les Management Tools.

## Historique des modifications

- **25 octobre 2026 :** Ajout : les SU d’octobre du 13 octobre 2026 (builds 15.2.1748.55 et 15.1.2507.77) sont les dernières mises à jour pour Exchange 2016 et 2019. Aucune date n’est fixée pour Exchange SE CU2 en raison du report de CU1.

## Sources

1.  [Exchange 2016/2019: Throttling and Blocking up to the Final Public Update Baseline – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/exchange-20162019-throttling-and-blocking-up-to-the-final-public-update-baseline/4552717): L’annonce du 2 septembre 2026 avec le nouveau seuil minimal (SU d’octobre 2025), le démarrage durant la deuxième semaine de septembre et l’indication d’un prochain relèvement au-delà de la dernière mise à jour publique.

2.  [Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/throttling-and-blocking-email-from-persistently-vulnerable-exchange-servers-to-e/3815328): L’article de référence de 2023 avec la définition de «persistently vulnerable», les niveaux Reporting, Throttling et Blocking, le cycle de 90 jours, les réponses SMTP 4.7.230 et 5.7.230 ainsi que le plan de déploiement par version.

3.  [How to pause throttling and blocking of out-of-date on-premises Exchange Servers – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/how-to-pause-throttling-and-blocking-of-out-of-date-on-premises-exchange-servers/4007169): Les cmdlets `Get-OnPremServerReportInfo`, `Get-TenantExemptionInfo` et `New-TenantExemptionInfo`, ainsi que le rapport dans l’Exchange Admin Center et le contingent annuel de 90 jours.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Les numéros de build des SU d’octobre 2025 et des mises à jour ESU ultérieures jusqu’en octobre 2026 ; comprend aussi l’indication que seules les clients ESU reçoivent les SU à partir de décembre 2025.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU15: October 14, 2025 (KB5066367) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2019-cu15-october-14-2025-kb5066367-19a1091c-e0b3-4078-be6b-312463063d05): L’article KB relatif au niveau minimal pour Exchange 2019 CU15.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: October 14, 2025 (KB5066369) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2016-cu23-october-14-2025-kb5066369-8ca2ab99-dfde-4329-896a-3faa677d2603): L’article KB relatif au niveau minimal pour Exchange 2016 CU23.

7.  [Support for Exchange Server 2016 and Exchange Server 2019 ends today – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/support-for-exchange-server-2016-and-exchange-server-2019-ends-today/4461192): La fin du support le 14 octobre 2025.

8.  [Announcing Exchange 2016 / 2019 Extended Security Update program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-exchange-2016--2019-extended-security-update-program/4433495): Les conditions de la première période ESU.

9.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): La période de mai à octobre 2026, l’indication qu’il n’y aura pas de prolongation supplémentaire et l’ajout du 13 octobre 2026 relatif aux dernières mises à jour pour Exchange 2016 et 2019.

10. [Exchange Online transport enforcement system explained – CodeTwo Admin's Blog](https://www.codetwo.com/admins-blog/persistently-vulnerable-exchange-server/): Décomposition sous forme de tableau des huit niveaux d’enforcement et des dates de déploiement pour chaque version d’Exchange ; source tierce.

11. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventaire des niveaux CU/SU et des étapes manuelles en attente.

12. [Where is Exchange SE CU1 anyway? – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/where-is-exchange-se-cu1-anyway/4546837): L’équipe Exchange sur le report d’Exchange SE CU1 sans date cible.
