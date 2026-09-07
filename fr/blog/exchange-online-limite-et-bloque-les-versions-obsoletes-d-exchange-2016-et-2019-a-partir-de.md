---
title: "Exchange Online limite et bloque les versions obsolètes d’Exchange 2016 et 2019 à partir de septembre 2026 : fonctionnement du transport enforcement"
navTitle: "EXO-Enforcement 09/2026"
description: "À partir de la deuxième semaine de septembre 2026, Exchange Online exige des serveurs hybrides au minimum la SU d’octobre 2025, faute de quoi le flux de messagerie est limité puis bloqué. Contexte du transport enforcement en place depuis 2023, niveaux d’escalade avec codes SMTP, rapport dans l’Admin Center, pause de 90 jours via PowerShell et raison pour laquelle la prochaine hausse n’autorisera plus que les clients ESU et Exchange SE."
date: "2026-09-07"
kategorie: "Exchange OnPrem / Hybride"
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
url: https://rafaelpfister.ch/fr/blog/exchange-online-limite-et-bloque-les-versions-obsoletes-d-exchange-2016-et-2019-a-partir-de
translationSourceHash: bd79f97bff8aa7047f498355cc9cd65834e03cb24b8d552a841035216f63187c
translationModel: gpt-5.6-terra
translatedAt: 2026-09-07T08:27:29.818Z
translationReview: automatic
---

# Exchange Online limite et bloque les versions obsolètes d’Exchange 2016 et 2019 à partir de septembre 2026 : fonctionnement du transport enforcement

Le 2 septembre 2026, l’équipe Exchange a annoncé le relèvement de la version minimale requise pour Exchange 2016 et Exchange 2019 dans le flux de messagerie hybride. À partir de la deuxième semaine de septembre 2026, Exchange Online exigera des serveurs qui remettent des messages via un connecteur entrant de type `OnPremises` au minimum le niveau de la dernière mise à jour de sécurité publique d’octobre 2025. Tout niveau inférieur sera limité, puis bloqué. En bref : les personnes qui n’ont pas corrigé leurs serveurs hybrides depuis octobre 2025 perdront progressivement, au cours des prochaines semaines, la remise des e-mails vers Exchange Online. Et le prochain relèvement, que Microsoft annonce pour les mois à venir, sera supérieur à toute mise à jour disponible publiquement : seuls les clients du programme ESU payant ou les environnements avec Exchange Server Subscription Edition (SE) satisferont alors à l’exigence.

L’annonce elle-même est brève. Ses conséquences pratiques découlent du système d’enforcement que Microsoft construit progressivement depuis 2023 : quelles réponses SMTP votre serveur recevra, comment vérifier l’état dans l’Admin Center et via PowerShell, et quelles options subsistent pendant la transition jusqu’à la fin du programme ESU en octobre 2026.

## Ce qui s’applique à partir de la deuxième semaine de septembre 2026

Le nouveau seuil correspond aux mises à jour de sécurité du 14 octobre 2025. Il s’agissait du dernier Patch Tuesday lors duquel Microsoft a publié publiquement des mises à jour pour Exchange 2016 et 2019 ; toutes les SU depuis décembre 2025 ne sont disponibles que via le programme ESU.

| Version | Niveau minimal | KB | Build |
|---|---|---|---|
| Exchange 2019 CU15 | SU d’octobre 2025 (CU15 SU5) | KB5066367 | 15.2.1748.39 |
| Exchange 2016 CU23 | SU d’octobre 2025 (CU23 SU19) | KB5066369 | 15.1.2507.61 |

Il existe également une SU d’octobre 2025 pour Exchange 2019 CU14 (KB5066368, build 15.2.1544.36). Toutefois, la publication Microsoft mentionne explicitement CU15 SU5 comme version minimale ; CU14 n’est de toute façon plus une version recommandée depuis la publication de CU15 en février 2025. Prévoyez également le passage à CU15 si vous utilisez CU14.

Trois restrictions sont importantes :

- **Seul le flux de messagerie hybride est concerné.** Exchange Online vérifie la version des serveurs émetteurs pour les messages reçus via un connecteur entrant de type `OnPremises`. Il s’agit de la configuration hybride classique créée par le Hybrid Configuration Wizard. Les e-mails qui arrivent via une passerelle tierce ou un connecteur de type `Partner` ne passent pas par cet enforcement.
- **La version est lue dans les en-têtes.** Un serveur Exchange inscrit son build dans la ligne `Received` de chaque message qu’il transmet (`… with Microsoft SMTP Server … id 15.1.2507.61`). Exchange Online analyse cette information. C’est donc le niveau du serveur qui remet effectivement le message à Exchange Online qui compte, soit, dans de nombreux environnements, l’Edge Transport Server ou le serveur Mailbox doté du connecteur d’envoi vers `*.mail.protection.outlook.com`.
- **Exchange SE n’est pas concerné.** L’enforcement s’applique à Exchange 2016 et 2019 ; Exchange Server SE est au-dessus de tout seuil minimal tant qu’il est régulièrement corrigé.

## Contexte : le transport enforcement depuis 2023

L’annonce de septembre n’est pas une nouvelle mesure, mais l’étape suivante d’un système que Microsoft a présenté en mars 2023 sous le titre «Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online». Microsoft définit comme «persistently vulnerable» tout serveur Exchange ayant atteint sa fin de support ou restant non corrigé face à des vulnérabilités connues. L’objectif est de protéger les destinataires Exchange Online des messages provenant de serveurs susceptibles d’être compromis, tout en incitant les exploitants à corriger ou à désactiver leurs serveurs.

Le système a été activé progressivement selon les versions :

| Période | Version concernée |
|---|---|
| Août 2023 | Exchange 2007 |
| Septembre 2023 | Exchange 2010 |
| Décembre 2023 | Exchange 2013 |
| Mars 2024 | Exchange 2016 et 2019 (versions SU nettement obsolètes) |
| Septembre 2026 | Exchange 2016 et 2019 : seuil minimal = SU d’octobre 2025 |
| «dans quelques mois» | Exchange 2016 et 2019 : seuil minimal supérieur à la dernière mise à jour publique |

Jusqu’à présent, le seuil minimal pour Exchange 2016 et 2019 concernait les niveaux «significantly behind on security updates». La nouveauté est que Microsoft place la limite à la dernière mise à jour publique et touche ainsi, pour la première fois, des serveurs qui étaient encore entièrement corrigés il y a moins d’un an.

## Les niveaux d’escalade

L’enforcement fonctionne selon trois mécanismes que Microsoft appelle «reporting», «throttling» et «blocking». Dès qu’un serveur passe sous le seuil minimal, un cycle de 90 jours commence. Les étapes décrites dans l’article de référence de 2023 sont les suivantes :

| Période | Mesure | Réponse SMTP |
|---|---|---|
| Jour 0 à 30 | Rapport uniquement dans l’Exchange Admin Center | aucune |
| Jour 30 à 40 | Limitation 5 minutes par heure | `450 4.7.230` |
| Jour 40 à 50 | Limitation 10 minutes par heure | `450 4.7.230` |
| Jour 50 à 60 | Limitation 20 minutes par heure | `450 4.7.230` |
| Jour 60 à 70 | Limitation 30 minutes par heure, plus blocage 5 minutes par heure | `450 4.7.230` et `550 5.7.230` |
| Jour 70 à 80 | Blocage 10 minutes par heure | `550 5.7.230` |
| Jour 80 à 90 | Blocage 20 minutes par heure | `550 5.7.230` |
| À partir du jour 90 | Blocage complet | `550 5.7.230` |

Les deux réponses sont formulées comme suit :

```text
450 4.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online throttled for n mins/hr.

550 5.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online blocked for n mins/hr.
```

La différence est déterminante pour l’exploitation. Avec `450`, Exchange Online refuse temporairement la connexion ; le serveur On-Premises conserve le message dans sa file d’attente et réessaie ultérieurement. Les utilisateurs ne constatent d’abord que des retards ; dans Queue Viewer ou dans `Get-Queue`, la file d’attente vers le connecteur d’envoi d’Exchange Online augmente avec l’état `Retry` et le message 4.7.230 comme `LastError`. Avec `550`, le refus est définitif : l’expéditeur reçoit un NDR avec le code 5.7.230 et le message est perdu, sauf s’il est renvoyé. Comme le blocage n’est d’abord actif que quelques minutes par heure, le symptôme semble initialement sporadique : une partie des messages arrive, une autre échoue avec un NDR. Si vous observez ce schéma dans le suivi des messages, vérifiez d’abord la version avant de rechercher des problèmes réseau ou de certificat.

L’annonce ne précise pas si Microsoft lancera le cycle complet de 90 jours à partir de la deuxième semaine de septembre pour le nouveau seuil, ou s’il commencera déjà à une étape ultérieure. L’article de référence indique que le système reprend au niveau précédemment atteint après une pause. Ne comptez donc pas sur un délai de grâce de 30 jours.

## Rapport dans l’Exchange Admin Center et via PowerShell

Exchange Online répertorie les serveurs On-Premises détectés, avec leur version, dans un rapport dédié : dans l’Exchange Admin Center, sous *Reports*, *Mail flow*, rapport sur les serveurs Exchange On-Premises connectés obsolètes («out-of-date connecting on-premises Exchange servers»). Le rapport indique pour chaque serveur le build détecté, s’il est sous le seuil minimal et à quel niveau se trouve l’enforcement.

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
| `Get-OnPremServerReportInfo` | Répertorie les serveurs On-Premises détectés par Exchange Online avec leur build, leur état d’enforcement et leur niveau. |

</details>

Le rapport ne connaît que les serveurs qui remettent effectivement des e-mails à Exchange Online. Un serveur de gestion sans flux de messagerie ou une machine ne disposant que des Management Tools n’y apparaissent pas. Cela est sans importance pour l’enforcement, mais pas pour la sécurité : ces systèmes ont eux aussi besoin des SU.

## Mettre l’enforcement en pause : 90 jours par an

Pour les environnements qui ne peuvent pas atteindre le seuil minimal à court terme, Microsoft propose une pause. Elle peut être activée pour un total de 90 jours par an, en une fois ou en plusieurs périodes :

```powershell
Get-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer

New-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer `
  -NumberOfDays 30
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Get-TenantExemptionInfo` | Indique si une pause est active pour le tenant et pendant combien de temps. |
| `New-TenantExemptionInfo` | Crée une nouvelle pause. |
| `-BlockingScenario UnpatchedOnPremServer` | Sélectionne le scénario «serveur On-Premises obsolète» ; il n’existe actuellement aucun autre scénario pour ce cmdlet. |
| `-NumberOfDays 30` | Durée de la pause en jours. Le contingent est de 90 jours par an et la valeur indiquée y est déduite. |

</details>

Deux caractéristiques de la pause sont importantes en pratique. Premièrement, après son expiration, l’enforcement reprend au niveau où il avait été arrêté ; la pause ne réinitialise pas le cycle de 90 jours. Deuxièmement, aucun cmdlet ne permet de mettre fin prématurément à une pause en cours : si vous créez une pause de 90 jours et terminez les correctifs après deux semaines, votre contingent annuel est épuisé. Créez donc la pause pour une durée aussi courte que possible et prolongez-la si nécessaire.

La pause n’est en outre une solution que pour le seuil minimal actuel. Lorsque Microsoft relèvera la limite, dans quelques mois, au-dessus de la dernière mise à jour publique, un contingent épuisé ne vous aidera plus.

## Pourquoi le prochain relèvement est la véritable échéance

Exchange 2016 et 2019 ne sont plus pris en charge depuis le 14 octobre 2025. Microsoft a ensuite mis en place deux périodes ESU payantes : la période 1 jusqu’en avril 2026, la période 2 de mai à octobre 2026. En annonçant la période 2 le 15 avril 2026, l’équipe Exchange a précisé qu’il n’y aurait pas de prolongation supplémentaire. Les SU de décembre 2025 à août 2026 (dernièrement le build 15.2.1748.49 pour 2019 CU15 et 15.1.2507.72 pour 2016 CU23) sont exclusivement disponibles pour les clients ESU et ne sont pas proposées publiquement en téléchargement.

La situation est donc la suivante :

- **Aujourd’hui**, un serveur doté de la SU d’octobre 2025 satisfait au seuil minimal, avec ou sans ESU.
- **Lors du prochain relèvement**, le seuil minimal sera, selon Microsoft, supérieur au niveau d’octobre 2025. Sans contrat ESU, il n’existe aucun moyen légal d’atteindre ce niveau. Le flux de messagerie hybride de ces serveurs sera alors limité puis bloqué, indépendamment de la qualité d’exploitation du reste de l’environnement.
- **Le 31 octobre 2026**, la période 2 prendra également fin. Après cette date, il n’y aura plus de SU pour Exchange 2016 et 2019, pour personne. Le relèvement suivant touchera donc au plus tard également les clients ESU.

Le programme ESU n’achète ainsi, au mieux, que quelques mois. Le seul niveau pérenne accepté par l’enforcement est Exchange Server SE. Microsoft a également annoncé qu’Exchange SE CU2 (prévu pour le second semestre 2026) mettra fin à la coexistence avec Exchange 2016 et 2019 : l’installation échouera si des serveurs plus anciens sont détectés dans l’organisation. La migration est donc nécessaire non seulement en raison du flux de messagerie, mais aussi pour pouvoir continuer à installer des mises à jour pour SE.

Pour les environnements qui conservent Exchange On-Premises uniquement afin de gérer des attributs dans une configuration hybride, l’alternative consiste à supprimer le dernier serveur : depuis Exchange 2019 CU12, les attributs de destinataire peuvent être gérés avec les Management Tools sans serveur Exchange en fonctionnement. Il n’y a alors plus de flux de messagerie On-Premises et l’enforcement devient sans objet.

## Déterminer la version installée

`Get-ExchangeServer` n’affiche dans `AdminDisplayVersion` que le CU, pas la SU. La version de fichier de `ExSetup.exe` est fiable, tout comme le [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), qui signale également les étapes manuelles manquantes. Pour obtenir rapidement une vue d’ensemble de tous les serveurs :

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
| `\\<Server>\C$\...\ExSetup.exe` | Chemin de partage administratif vers le fichier d’installation ; adaptez-le si le chemin d’installation diffère. |
| `VersionInfo.ProductVersion` | Version de fichier correspondant au build SU installé (par ex. `15.1.2507.61`). |

</details>

Si la version est inférieure à `15.2.1748.39` (2019 CU15) ou à `15.1.2507.61` (2016 CU23), le serveur sera sous le seuil minimal à partir de la deuxième semaine de septembre.

## Procédure recommandée

1. **Inventoriez les versions** comme décrit ci-dessus, y compris les Edge Transport Server et les serveurs de gestion.

2. **Vérifiez le rapport dans Exchange Online.** `Get-OnPremServerReportInfo` indique quels serveurs Exchange Online voit réellement et si un niveau d’enforcement est déjà actif. Comparez la liste avec l’inventaire : les serveurs absents ne remettent pas leurs messages via le connecteur `OnPremises`.

3. **Installez au minimum la SU d’octobre 2025.** KB5066367 (2019 CU15) et KB5066369 (2016 CU23) restent publiquement disponibles dans le Microsoft Download Center. Les SU sont cumulatives ; un serveur au niveau d’août 2025 peut être directement mis à niveau vers octobre 2025. Avec CU14, installez d’abord CU15. Après l’installation, redémarrez, contrôlez l’état des services et exécutez à nouveau le Health Checker.

4. **N’utilisez la pause que comme solution transitoire.** Si la mise à jour ne peut pas être effectuée durant la première moitié de septembre, créez `New-TenantExemptionInfo` avec une durée courte et ne considérez pas la pause comme une réserve de planification pour le prochain relèvement.

5. **Planifiez la migration vers Exchange SE.** Sans contrat ESU, le prochain relèvement constitue l’échéance impérative ; avec ESU, il s’agit du 31 octobre 2026. Exchange 2019 CU15 peut être mis à niveau sur place vers SE ; Exchange 2016 nécessite le détour par une nouvelle installation de SE et le déplacement des boîtes aux lettres ou des rôles. Les personnes qui n’utilisent Exchange que pour la gestion des attributs suppriment le dernier serveur et continuent à travailler avec les Management Tools.

## Sources

1.  [Exchange 2016/2019: Throttling and Blocking up to the Final Public Update Baseline – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/exchange-20162019-throttling-and-blocking-up-to-the-final-public-update-baseline/4552717): L’annonce du 2 septembre 2026 avec le nouveau seuil minimal (SU d’octobre 2025), la date de début durant la deuxième semaine de septembre et la mention du prochain relèvement au-delà de la dernière mise à jour publique.

2.  [Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/throttling-and-blocking-email-from-persistently-vulnerable-exchange-servers-to-e/3815328): L’article de référence de 2023 avec la définition de «persistently vulnerable», les niveaux Reporting, Throttling et Blocking, le cycle de 90 jours, les réponses SMTP 4.7.230 et 5.7.230, ainsi que le plan de déploiement par version.

3.  [How to pause throttling and blocking of out-of-date on-premises Exchange Servers – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/how-to-pause-throttling-and-blocking-of-out-of-date-on-premises-exchange-servers/4007169): Les cmdlets `Get-OnPremServerReportInfo`, `Get-TenantExemptionInfo` et `New-TenantExemptionInfo`, le rapport dans l’Exchange Admin Center et le contingent annuel de 90 jours.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Numéros de build des SU d’octobre 2025 et des mises à jour ESU suivantes jusqu’en août 2026 ; contient également l’indication que seules les clientes et clients ESU reçoivent les SU à partir de décembre 2025.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU15: October 14, 2025 (KB5066367) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2019-cu15-october-14-2025-kb5066367-19a1091c-e0b3-4078-be6b-312463063d05): L’article KB relatif au niveau minimal pour Exchange 2019 CU15.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: October 14, 2025 (KB5066369) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2016-cu23-october-14-2025-kb5066369-8ca2ab99-dfde-4329-896a-3faa677d2603): L’article KB relatif au niveau minimal pour Exchange 2016 CU23.

7.  [Support for Exchange Server 2016 and Exchange Server 2019 ends today – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/support-for-exchange-server-2016-and-exchange-server-2019-ends-today/4461192): La fin du support au 14 octobre 2025.

8.  [Announcing Exchange 2016 / 2019 Extended Security Update program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-exchange-2016--2019-extended-security-update-program/4433495): Conditions de la première période ESU.

9.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Durée de mai à octobre 2026 et indication qu’aucune prolongation supplémentaire ne suivra.

10. [Exchange Online transport enforcement system explained – CodeTwo Admin's Blog](https://www.codetwo.com/admins-blog/persistently-vulnerable-exchange-server/): Présentation tabulaire des huit niveaux d’enforcement et des dates de déploiement pour chaque version d’Exchange ; source tierce.

11. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventaire des niveaux CU/SU et des étapes manuelles en attente.
