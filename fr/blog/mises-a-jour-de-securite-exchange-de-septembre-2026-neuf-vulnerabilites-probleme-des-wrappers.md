---
title: "Mises à jour de sécurité Exchange de septembre 2026 : neuf vulnérabilités, problème des wrappers corrigé, v2 publiée"
navTitle: "Exchange SU 09/2026"
description: "La SU de septembre corrige neuf vulnérabilités dans Exchange SE et 2019 (huit dans Exchange 2016), dont une faille d’usurpation avec un CVSS de 9.3, et résout le problème des wrappers dans les environnements hybrides. Une v2 avec une CVE supplémentaire a suivi le 2 octobre ; trois problèmes connus avec solutions de contournement et un SettingOverride à supprimer s’y ajoutent."
date: "2026-10-07"
kategorie: "Exchange OnPrem / Hybride"
timeToRead: "7 min de lecture"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
  - "powershell"
slug: "mises-a-jour-de-securite-exchange-de-septembre-2026-neuf-vulnerabilites-probleme-des-wrappers"
translationId: article-53db0c02bc33f9bd
translationOf: exchange-security-updates-september-2026
url: https://rafaelpfister.ch/fr/blog/mises-a-jour-de-securite-exchange-de-septembre-2026-neuf-vulnerabilites-probleme-des-wrappers
translationSourceHash: d0738f61713a26973457a9e536720b9787af35b22867c648dbe113d2aeb3f4ea
translationModel: gpt-5.6-terra
translatedAt: 2026-10-08T10:44:32.829Z
translationReview: automatic
---

# Mises à jour de sécurité Exchange de septembre 2026 : neuf vulnérabilités, problème des wrappers corrigé, v2 publiée

Microsoft a publié des mises à jour de sécurité (SU) pour Exchange Server le 8 septembre 2026. Elles corrigent neuf vulnérabilités dans Exchange SE et Exchange 2019, et huit dans Exchange 2016. Aucune n’était connue publiquement à l’avance, aucune n’est activement exploitée selon le Security Update Guide, et Microsoft les classe toutes comme *Important* avec « Exploitation Less Likely ». La valeur CVSS maximale, de 9.3, est toutefois nettement supérieure à celle du mois précédent. Ce mois-ci se distingue pour trois raisons : la SU corrige le problème des *messages wrapper* dans les boîtes aux lettres partagées, ouvert depuis juin, elle apporte trois nouveaux problèmes connus ou persistants, et Microsoft a publié une **version 2** le 2 octobre, qui corrige une vulnérabilité supplémentaire.

## Versions d’Exchange pour lesquelles la mise à jour est disponible

Les SU du 8 septembre 2026 sont disponibles pour les versions suivantes :

- **Exchange Server Subscription Edition (SE) RTM** : KB5121608, build 15.2.2562.49 ; disponible publiquement.
- **Exchange Server 2019 CU15** : KB5121609, build 15.2.1748.51 ; uniquement via le **programme ESU de période 2**.
- **Exchange Server 2019 CU14** : KB5121610, build 15.2.1544.46 ; uniquement via ESU de période 2.
- **Exchange Server 2016 CU23** : KB5121611, build 15.1.2507.73 ; uniquement via ESU de période 2.

Exchange 2016 et 2019 ne sont plus pris en charge. Selon Microsoft, seules les organisations inscrites au programme ESU de période 2 reçoivent les SU de mai à octobre 2026. D’après les articles KB, cette éligibilité s’étend jusqu’en octobre 2026. À cela s’ajoute la pression d’Exchange Online : depuis la deuxième semaine de septembre, Exchange Online limite et bloque le flux de messagerie hybride depuis les serveurs antérieurs au niveau d’octobre 2025 ; voir les détails dans l’[article sur l’application des règles de transport](/blog/exchange-online-transport-enforcement-hybrid-server). Selon l’annonce, Exchange Online lui-même est déjà protégé ; dans les environnements hybrides, chaque serveur Exchange doit néanmoins recevoir la SU, de même que les machines dotées des Exchange Management Tools.

Vous pouvez comparer votre niveau actuel à la liste des [numéros de build Exchange](/tools/exchange-builds).

## Vue d’ensemble des vulnérabilités

| CVE | Type | CVSS |
| --- | --- | --- |
| CVE-2026-69356 | Usurpation (Cross-Site Scripting) | 9.3 |
| CVE-2026-69641 | Élévation de privilèges | 9.1 |
| CVE-2026-69355 | Exécution de code à distance | 8.8 |
| CVE-2026-55007 | Exécution de code à distance | 8.1 |
| CVE-2026-69380 | Élévation de privilèges | 8.1 |
| CVE-2026-69378 | Déni de service | 7.5 |
| CVE-2026-69361 | Usurpation (Server-Side Request Forgery) | 6.5 |
| CVE-2026-69375 | Altération | 6.5 |
| CVE-2026-69382 | Divulgation d’informations | 5.9 |

CVE-2026-55007 ne concerne pas Exchange 2016 ; le Security Update Guide n’indique à son sujet qu’Exchange SE et Exchange 2019 CU14/CU15. Une précision concernant la documentation : CVE-2026-69380 est absente de la liste des CVE dans les articles KB des quatre SU de septembre. Le Security Update Guide mentionne toutefois précisément les quatre builds de septembre comme correctif pour cette CVE (état au 7 octobre 2026).

**[CVE-2026-69356](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69356) présente la valeur la plus élevée, avec un CVSS de 9.3. Selon Microsoft, un attaquant non authentifié peut envoyer une invitation de calendrier spécialement conçue contenant un lien de réunion malveillant ; lorsque le destinataire ouvre la réunion et sélectionne le lien pour la rejoindre, le Cross-Site Scripting est déclenché. Une interaction utilisateur est donc nécessaire, mais aucun compte dans l’organisation.

**[CVE-2026-69380](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69380) (élévation de privilèges, CVSS 8.1) nécessite seulement un compte disposant de peu de droits et une boîte aux lettres attribuée. Selon la FAQ du Security Update Guide, un attaquant peut, en exploitant des faiblesses dans la vérification des requêtes et des jetons d’identité, se faire passer pour un autre utilisateur et prendre le contrôle des boîtes aux lettres de tous les utilisateurs Exchange : lire et envoyer des e-mails, ainsi que télécharger des pièces jointes. Un seul compte utilisateur compromis suffit comme point de départ.

**[CVE-2026-69641](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69641) (élévation de privilèges, CVSS 9.1) aboutit au même résultat, la prise de contrôle de toutes les boîtes aux lettres, mais requiert l’appartenance à un groupe de rôles hautement privilégié.

La situation initiale diffère pour les deux failles d’exécution de code à distance : [CVE-2026-69355](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69355) (CVSS 8.8) exige un compte authentifié disposant de peu de droits, tandis que [CVE-2026-55007](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-55007) (CVSS 8.1) peut être déclenchée sans authentification au moyen d’une pièce jointe Visio spécialement conçue, mais nécessite selon Microsoft une mémoire vive constamment faible sur le système cible. Les quatre autres failles sont les suivantes : CVE-2026-69378 (DoS par récursion non contrôlée, sans authentification), CVE-2026-69361 (SSRF, le serveur envoie des requêtes HTTP vers des systèmes internes ou de bouclage), CVE-2026-69375 (un attaquant authentifié peut remplacer le contenu de fichiers) et CVE-2026-69382 (divulgation d’informations d’identification via un algorithme cryptographique faible, nécessite un cookie d’authentification déjà dérobé).

## Version 2 du 2 octobre : CVE-2026-96940 ajoutée

Le 2 octobre 2026, Microsoft a publié la « Version 2 » des SU de septembre. Selon l’annonce, la seule différence avec la première version est le correctif supplémentaire pour **[CVE-2026-96940](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940). Cette faille d’élévation de privilèges a un CVSS de 8.8, n’est ni connue publiquement ni exploitée, mais Microsoft est la seule des dix CVE à l’évaluer comme **« Exploitation More Likely »**. Un attaquant authentifié peut ainsi accéder aux boîtes aux lettres d’autres utilisateurs de la même organisation et lire les e-mails, pièces jointes comprises. Exchange Online est déjà corrigé côté serveur.

| Version | KB | Build v2 |
| --- | --- | --- |
| Exchange SE RTM | KB5129955 | 15.2.2562.53 |
| Exchange 2019 CU15 | KB5129956 | 15.2.1748.53 |
| Exchange 2019 CU14 | KB5129957 | 15.2.1544.48 |
| Exchange 2016 CU23 | KB5129958 | 15.1.2507.75 |

En pratique, cela signifie que le correctif pour CVE-2026-96940 n’est inclus que dans les builds v2. Les serveurs sur lesquels la SU du 8 septembre est déjà installée nécessitent donc aussi v2. Ceux qui appliquent les correctifs seulement maintenant peuvent installer directement v2, les SU étant cumulatives. Les articles KB n’indiquent pas explicitement si les serveurs équipés de la première SU de septembre doivent obligatoirement installer v2 ; puisque la nouvelle CVE n’est corrigée qu’avec v2, c’est l’interprétation la plus logique.

## Problème des wrappers corrigé : supprimer maintenant le SettingOverride

Le problème connu depuis la SU de juin, qui faisait apparaître des *messages wrapper* dans la boîte de réception des boîtes aux lettres partagées dans les environnements hybrides, est corrigé avec la SU de septembre sur les quatre versions. L’[article d’août](/blog/exchange-security-updates-august-2026) indiquait encore que le SettingOverride documenté comme solution de contournement pouvait rester en place. Après l’installation de la SU de septembre, c’est l’inverse : Microsoft recommande dans l’article de support associé de vérifier puis de supprimer l’override.

```powershell
Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
```

<details class="options-details">
<summary>Explication des options</summary>

| Commande | Effet |
|---|---|
| `Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Vérifie si l’override de contournement est défini dans l’organisation. |
| `Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Supprime l’override dès que la SU de septembre est installée. |

</details>

Si la première commande indique que l’objet `DisableBlockSharedAndUserMailboxHeaders` est introuvable, Microsoft précise qu’aucune autre action n’est nécessaire.

Pour Exchange SE, la SU corrige en outre une erreur lors des requêtes de disponibilité hybride via Microsoft Graph : les utilisateurs on-premises voyaient les périodes occupées des boîtes aux lettres Exchange Online décalées de leur propre décalage UTC, sans message d’erreur.

## Problèmes connus

**Les calendriers publiés (.ics) renvoient HTTP 500 aux applications de calendrier.** Le problème existe depuis la SU d’août (Exchange SE à partir du build 15.2.2562.46 ainsi qu’Exchange 2019 et 2016) et n’est pas corrigé dans la SU de septembre ni dans v2. Les abonnements à des calendriers publiés anonymement ne se mettent plus à jour ; la même URL fonctionne dans le navigateur. Selon Microsoft, la cause est la suivante : Exchange identifie les clients à partir du User-Agent ; les applications de calendrier, sans identifiant de navigateur, empruntent un chemin de code que la SU d’août a désactivé. Comme solution de contournement, Microsoft décrit une règle de réécriture d’URL sur le site « Exchange Back End » dans IIS, qui ajoute le paramètre `layout=premium` aux requêtes .ics sous `/owa/calendar/` ; le module IIS URL Rewrite doit être installé. Les étapes exactes (via le gestionnaire IIS ou directement dans `applicationHost.config`) figurent dans l’[article de support KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672). Microsoft ne donne aucune date pour le correctif (état au 7 octobre 2026).

**Disponibilité des boîtes aux lettres déléguées dans les environnements hybrides (Exchange SE uniquement).** Si la requête de disponibilité est configurée exclusivement via l’API Graph, les requêtes vers les boîtes aux lettres Exchange Online via un accès on-premises délégué échouent. Outlook affiche « Your server location could not be determined », OWA affiche « No information » et les journaux EWS contiennent `(403) Forbidden`. La solution de contournement documentée redirige les requêtes via EWS au lieu de Graph :

```powershell
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=False"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-Identity EnableRouteThroughMSGraphFeature` | L’override qui contrôle le routage des requêtes de disponibilité via Microsoft Graph. |
| `-Parameters "Enabled=False"` | Désactive le chemin Graph ; les requêtes repassent par EWS. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Dirige l’appel de diagnostic vers le service de topologie. |
| `-Component VariantConfiguration -Argument Refresh` | Recharge la Variant Configuration afin que l’override prenne effet sans attendre. |

</details>

Dans l’annonce de v2, Microsoft classe ce problème parmi les problèmes corrigés. Ceux qui ont défini la solution de contournement devraient vérifier après l’installation de v2, dans l’[article de support KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092), s’il faut la retirer ; au 7 octobre 2026, aucune instruction à cet effet n’y figurait encore.

**Interblocage ContentEngine dû à l’absence de fichiers WordBreaker coréens (Exchange SE uniquement).** Avec la SU de septembre (build 15.2.2562.49 et build v2 15.2.2562.53), les fichiers de règles du WordBreaker coréen mis à jour ne sont pas installés. Conséquences : résultats de recherche manquants, distribution des e-mails retardée, clients Outlook ou MAPI qui se bloquent ou perdent la connexion. Selon l’annonce, cela concerne les messages en coréen. La solution de contournement dans l’[article de support KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098) consiste à extraire les deux fichiers `ko.token.rule.bin` et `ko.complex.rule.bin` de SQL Server 2025 Express RTM, à vérifier leurs hachages SHA256, à les copier dans le répertoire `Native` de l’installation Exchange et à redémarrer le service Search Host Controller. Microsoft étudie toujours le problème.

## Installation et suivi

Microsoft recommande la procédure habituelle : inventorier avec l’[Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), déterminer le chemin avec l’[Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) en cas de version obsolète, installer la SU, redémarrer le serveur et vérifier que tous les services Exchange fonctionnent. Le Health Checker indique ensuite également si la SU est correctement installée. Le Security Update Guide indique qu’un redémarrage est requis pour ces mises à jour.

Après l’installation, trois actions de suivi sont nécessaires :

1. Supprimer le SettingOverride des wrappers `DisableBlockSharedAndUserMailboxHeaders` s’il est défini (voir ci-dessus).

2. Sur Exchange SE, vérifier si les problèmes de disponibilité et de WordBreaker surviennent et, le cas échéant, appliquer les solutions de contournement.

3. Pour les calendriers publiés avec des abonnés externes, configurer la règle de réécriture d’URL de KB5126672, si cela n’a pas déjà été fait depuis la SU d’août.

Depuis juillet, il reste aussi à vérifier si l’atténuation de CVE-2026-42897 (M2.1.0) est encore active ; la procédure pour la supprimer figure dans l’[article sur la SU de juillet](/blog/exchange-security-updates-juli-2026).

## Procédure recommandée

Installez directement les builds v2 du 2 octobre sur tous les serveurs Exchange et toutes les machines avec les outils de gestion ; les serveurs équipés de la SU du 8 septembre ont besoin de v2 en supplément pour CVE-2026-96940. La faille d’usurpation avec un CVSS de 9.3 et la prise de contrôle de boîtes aux lettres au moyen d’un compte utilisateur disposant de peu de droits (CVE-2026-69380) suffisent à justifier de ne pas attendre le prochain Patch Tuesday. Supprimez ensuite l’override des wrappers, vérifiez les trois problèmes connus et exécutez le Health Checker. Pour Exchange 2016 et 2019, le programme ESU prend fin en octobre 2026 ; la migration vers Exchange SE ne peut plus être repoussée.

## Sources

1.  [Released: September 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-september-2026-exchange-server-security-updates/4554411): Annonce officielle de publication avec les versions prises en charge, l’indication ESU, les problèmes connus, les problèmes corrigés et la procédure d’installation (consultée via le flux RSS du Exchange Team Blog).

2.  [Released: September 2026 V2 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-september-2026-v2-exchange-server-security-updates/ba-p/4561718): Annonce de la v2 du 2 octobre 2026 ; la seule différence est CVE-2026-96940.

3.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM: September 8, 2026 (KB5121608) – Microsoft Support](https://support.microsoft.com/help/5121608): Liste des CVE, problèmes corrigés et les trois problèmes connus pour Exchange SE.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU15: September 8, 2026 (KB5121609) – Microsoft Support](https://support.microsoft.com/help/5121609): Article KB pour Exchange 2019 CU15.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU14: September 8, 2026 (KB5121610) – Microsoft Support](https://support.microsoft.com/help/5121610): Article KB pour Exchange 2019 CU14.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: September 8, 2026 (KB5121611) – Microsoft Support](https://support.microsoft.com/help/5121611): Article KB pour Exchange 2016 CU23, sans CVE-2026-55007.

7.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Type, CVSS, niveau de gravité, évaluation de l’exploitation et FAQ pour les neuf CVE de septembre et CVE-2026-96940, y compris les builds concernés pour chaque CVE.

8.  [Description of version 2 of the security update for Microsoft Exchange Server Subscription Edition RTM October 2, 2026 (KB5129955) – Microsoft Support](https://support.microsoft.com/help/5129955): Article KB sur la v2 pour Exchange SE ; les articles v2 pour 2019 et 2016 vont de KB5129956 à KB5129958.

9.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Numéros de build des SU de septembre et de la v2 du 2 octobre 2026.

10. [Wrapper messages appear in shared mailbox in hybrid environments after installing the June 2026 Security Update – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Correctif dans la SU de septembre et instructions pour supprimer le SettingOverride.

11. [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Cause et solution de contournement par réécriture d’URL.

12. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Symptômes et solution de contournement par SettingOverride pour Exchange SE.

13. [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Builds SE concernés et solution de contournement manuelle.

14. [Hybrid free/busy through Microsoft Graph incorrectly shifts busy times by requester timezone – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5125804): Erreur de fuseau horaire dans Exchange SE corrigée avec la SU de septembre.

15. [Neue Sicherheitsupdates für Exchange Server (September 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-september-2026/): Présentation en allemand des neuf CVE avec valeurs CVSS et builds.

16. [Exchange Server: Sicherheitsupdates 8. September 2026 – Borns Tech and Windows World](https://borncity.com/blog/?p=329371): Résumé en allemand avec des indications sur la mise à jour de remplacement publiée ultérieurement.
