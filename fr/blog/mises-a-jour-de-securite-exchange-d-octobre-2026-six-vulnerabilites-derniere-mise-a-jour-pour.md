---
title: "Mises à jour de sécurité Exchange d’octobre 2026 : six vulnérabilités, dernière mise à jour pour Exchange 2016 et 2019"
navTitle: "Exchange SU 10/2026"
description: "La SU d’octobre corrige six vulnérabilités, dont une exécution de code à distance critique via un e-mail spécialement conçu, ainsi que l’erreur .ics et le blocage de WordBreaker. Il s’agit de la dernière mise à jour pour Exchange 2016 et 2019, y compris dans le programme ESU."
date: "2026-10-14"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "7 min de lecture"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
  - "powershell"
slug: "mises-a-jour-de-securite-exchange-d-octobre-2026-six-vulnerabilites-derniere-mise-a-jour-pour"
translationId: article-173db9431239a95f
translationOf: exchange-security-updates-oktober-2026
url: https://rafaelpfister.ch/fr/blog/mises-a-jour-de-securite-exchange-d-octobre-2026-six-vulnerabilites-derniere-mise-a-jour-pour
translationSourceHash: ba4b8563ea0b25f5badf03f09e69f9b8329bb75bf4c2422cb5c3fbb97f55d137
translationModel: gpt-5.6-terra
translatedAt: 2026-10-14T08:50:01.784Z
translationReview: automatic
---

# Mises à jour de sécurité Exchange d’octobre 2026 : six vulnérabilités, dernière mise à jour pour Exchange 2016 et 2019

Microsoft a publié le 13 octobre 2026 des mises à jour de sécurité (SU) pour Exchange Server SE, Exchange 2019 et Exchange 2016. Selon les articles KB, elles corrigent six vulnérabilités. Aucune n’était connue publiquement à l’avance et, selon le Security Update Guide, aucune n’est activement exploitée. Microsoft classe toutefois deux d’entre elles dans la catégorie « Exploitation More Likely », dont une exécution de code à distance évaluée *Critical*, qu’un attaquant non authentifié peut déclencher au moyen d’un e-mail spécialement conçu. Ce mois-ci se distingue pour deux raisons : la SU corrige les deux problèmes connus encore ouverts des mises à jour de septembre (HTTP 500 pour les calendriers publiés et le blocage de ContentEngine dans Exchange SE), et, selon Microsoft, il s’agit de la toute dernière mise à jour de sécurité pour Exchange 2016 et 2019, y compris pour les clients du programme ESU de période 2.

## Versions d’Exchange pour lesquelles la mise à jour est disponible

Les SU du 13 octobre 2026 sont disponibles pour les versions suivantes :

- **Exchange Server Subscription Edition (SE) RTM** : KB5128895, build 15.2.2562.56 ; disponible publiquement.
- **Exchange Server 2019 CU15** : KB5128896, build 15.2.1748.55 ; uniquement via le **programme ESU de période 2**.
- **Exchange Server 2019 CU14** : KB5128897, build 15.2.1544.50 ; uniquement via l’ESU de période 2.
- **Exchange Server 2016 CU23** : KB5128898, build 15.1.2507.77 ; uniquement via l’ESU de période 2.

Vous pouvez comparer votre version avec l’aperçu des [numéros de build Exchange](/tools/exchange-builds).

Exchange 2016 et 2019 ne sont plus pris en charge. Selon les articles KB, les organisations inscrites au programme ESU de période 2 reçoivent des mises à jour de sécurité jusqu’à fin octobre 2026 ; toutes les autres doivent migrer vers Exchange SE. Selon l’annonce, Exchange Online est déjà protégé ; dans les environnements hybrides, chaque serveur Exchange a néanmoins besoin de la SU, y compris les serveurs de gestion uniquement et les machines équipées des Exchange Management Tools. La section [Dernière mise à jour pour Exchange 2016 et 2019](#letztes-update-für-exchange-2016-und-2019) explique concrètement ce que signifie la fin de l’ESU.

## Vue d’ensemble des vulnérabilités

| CVE | Type | CVSS |
| --- | --- | --- |
| CVE-2026-70279 | Usurpation (Cross-Site Scripting) | 9.3 |
| CVE-2026-77910 | Élévation de privilèges | 8.8 |
| CVE-2026-70571 | Exécution de code à distance | 8.1 |
| CVE-2026-87692 | Élévation de privilèges | 7.8 |
| CVE-2026-87695 | Élévation de privilèges | 7.8 |
| CVE-2026-87696 | Élévation de privilèges | 7.5 |

Les quatre articles KB répertorient les mêmes six CVE. Dans le Security Update Guide, Exchange 2019 CU15 manque toutefois pour CVE-2026-87692 ; seuls Exchange SE, 2019 CU14 et 2016 CU23 y sont indiqués comme affectés (état au 14 octobre 2026). Frankys Web l’interprète comme indiquant que 2019 CU15 n’est pas affecté par cette vulnérabilité. La différence n’a aucune incidence sur l’installation, puisque la SU pour CU15 corrige de toute façon toutes les autres vulnérabilités.

**[CVE-2026-70571](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-70571)** est la seule vulnérabilité du mois que Microsoft classe comme *Critical* (exécution de code à distance, CVSS 8.1). La cause est un dépassement de tampon sur la pile. Selon la FAQ du Security Update Guide, un attaquant non authentifié peut envoyer un e-mail spécialement conçu à un serveur Exchange affecté et déclencher ainsi un processus de conversion interne ; aucune interaction de l’utilisateur n’est nécessaire. Selon le vecteur CVSS, la complexité de l’attaque est élevée (AC:H) ; Microsoft précise qu’une exploitation réussie n’est pas garantie et exige une connaissance approfondie de l’environnement cible. Microsoft estime néanmoins que l’exploitation est « Exploitation More Likely ». L’attaquant n’a pas besoin d’un compte dans l’organisation : il suffit que le serveur accepte les e-mails.

**[CVE-2026-70279](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-70279)** obtient le score le plus élevé avec un CVSS de 9.3 et est également classée « Exploitation More Likely ». Il s’agit d’un Cross-Site Scripting : un attaquant non authentifié envoie un lien spécialement conçu ; si un administrateur l’ouvre, l’attaquant peut exécuter des actions avec ses droits et ainsi accéder aux données de l’organisation ou les modifier. Selon la FAQ, cela requiert une interaction de l’utilisateur, généralement via de l’ingénierie sociale. Le niveau de gravité est *Important*.

**[CVE-2026-77910](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-77910)** (élévation de privilèges, CVSS 8.8) ne nécessite qu’un compte disposant de faibles privilèges. Selon la FAQ, un attaquant peut manipuler un cache partagé du Exchange Control Panel et faire ainsi installer un complément de messagerie malveillant dans la boîte aux lettres d’un autre utilisateur, sans que celui-ci ait à intervenir. Microsoft décrit le même vecteur d’attaque pour [CVE-2026-87696](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-87696) (CVSS 7.5), comme conséquence d’une vérification insuffisante des signatures cryptographiques et avec une complexité d’attaque élevée.

Les deux autres vulnérabilités, [CVE-2026-87692](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-87692) (désérialisation de données non fiables) et [CVE-2026-87695](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-87695), sont des élévations locales de privilèges avec un CVSS de 7.8 : selon Microsoft, un attaquant connecté disposant de faibles privilèges peut obtenir les droits SYSTEM. Toutes deux sont classées « Exploitation Less Likely », tout comme CVE-2026-77910 et CVE-2026-87696.

Concernant **CVE-2026-96940** de la v2 de septembre : l’annonce d’octobre ne mentionne pas la v2. Selon la FAQ de l’annonce, les SU sont cumulatives ; si vous êtes sur une CU prise en charge, vous installez uniquement la SU la plus récente. Frankys Web confirme que la SU d’octobre contient le correctif de la v2. Les serveurs exécutant encore la première SU de septembre du 8 septembre peuvent donc installer directement la SU d’octobre ; les détails sur la v2 figurent dans l’[article sur la v2 de septembre](/blog/exchange-security-updates-september-2026-v2).

## Dernière mise à jour pour Exchange 2016 et 2019

L’annonce est claire sur ce point : octobre 2026 est le dernier mois pendant lequel les clients du programme ESU de période 2 reçoivent de nouvelles mises à jour de sécurité pour Exchange 2016 et 2019. À partir de la prochaine version, les mises à jour ne seront publiées que pour Exchange SE. Les futures annonces et notifications CVE ne mentionneront plus 2016 ni 2019, et Microsoft écrit explicitement : « We will stop testing Exchange 2016 and 2019 for vulnerabilities. » Le fait qu’une CVE ne mentionne pas ces versions ne signifie donc pas qu’elles ne sont pas affectées ; Microsoft souligne dans l’annonce que les anciennes versions n’en deviennent pas sûres pour autant. Frankys Web indique également qu’Exchange SE et 2019 CU15 sont techniquement très similaires ; les futures vulnérabilités d’Exchange SE affecteront donc très probablement aussi 2019.

Selon l’annonce, l’ESU de période 2 peut encore être achetée pour une durée limitée. Cela ne permet toutefois à une organisation d’atteindre que le niveau de la mise à jour d’octobre 2026 ; aucune autre mise à jour ne suivra. Selon les articles KB, les organisations ayant déjà acheté une ESU et ayant des questions sur l’accès peuvent contacter `ExchangeandSfBServerESUInquiry@service.microsoft.com`.

Pour l’exploitation, cela signifie :

1. **Exchange 2016 et 2019 avec ESU** : installez la SU d’octobre. Ensuite, la version restera figée et toute vulnérabilité révélée ultérieurement restera non corrigée.

2. **Exchange 2016 et 2019 sans ESU** : ces serveurs n’ont plus reçu de mises à jour de sécurité depuis mai 2026. Microsoft recommande de migrer vers Exchange SE dès que possible.

3. **Environnements hybrides** : outre l’absence de correctif, Exchange Online limite et bloque le flux de messagerie provenant de serveurs On-Premises obsolètes ; les règles sont indiquées dans l’[article sur l’application des règles de transport](/blog/exchange-online-transport-enforcement-hybrid-server).

## Problèmes corrigés et actions à effectuer ensuite

**Les calendriers publiés (.ics) renvoient HTTP 500 aux applications de calendrier** (toutes les versions). Le problème existant depuis la SU d’août est corrigé avec la SU d’octobre ; [KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672) indique désormais comme solution l’installation des quatre mises à jour d’octobre ou d’une version plus récente. L’article de support ne décrit plus la solution de contournement précédente de réécriture d’URL sur le site « Exchange Back End », et Microsoft ne fournit pas d’instructions pour supprimer la règle (état au 14 octobre 2026). Si vous avez configuré cette règle, documentez les serveurs sur lesquels elle est active et testez un abonnement à un calendrier avant de la supprimer.

**Blocage de ContentEngine dû à l’absence de fichiers WordBreaker coréens** (Exchange SE uniquement). Selon [KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098), le problème est corrigé avec KB5128895 ou une mise à jour ultérieure, et après l’installation de la mise à jour d’octobre, « no additional action » n’est nécessaire. Cela vaut également pour les serveurs sur lesquels vous avez appliqué la solution de contournement manuelle de l’[article de septembre](/blog/exchange-security-updates-september-2026#workaround-für-den-wordbreaker-deadlock) : Microsoft n’exige pas d’annuler la copie des fichiers de règles. Les serveurs sans solution de contournement ne doivent plus ajouter les fichiers manuellement.

**Libre/Occupé pour les boîtes aux lettres déléguées dans les environnements hybrides avec une configuration Graph API uniquement** (Exchange SE uniquement). Ce problème n’est pas corrigé par la SU d’octobre : [KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) indique désormais « The issue has been fixed in the service ». La contradiction avec l’[article v2](/blog/exchange-security-updates-september-2026-v2) est ainsi résolue. Si vous avez défini la solution de contournement, supprimez-la conformément à KB5127092 :

```powershell
Get-SettingOverride -Identity EnableRouteThroughMSGraphFeature
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=True"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Get-SettingOverride -Identity EnableRouteThroughMSGraphFeature` | Affiche la valeur actuelle de l’override ; la commande suivante n’est nécessaire que si la valeur est `Enabled=False`. |
| `-Parameters "Enabled=True"` | Réactive le transfert des requêtes de disponibilité via Microsoft Graph. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Dirige l’appel de diagnostic vers le service de topologie. |
| `-Component VariantConfiguration -Argument Refresh` | Recharge la Variant Configuration afin que la modification prenne effet immédiatement. |

</details>

Vérifiez ensuite la requête Libre/Occupé d’une boîte aux lettres Exchange Online déléguée depuis Outlook ou OWA.

## Problèmes connus

Ni l’annonce ni les quatre articles KB ne mentionnent de problèmes connus pour la SU d’octobre (état au 14 octobre 2026). Les trois problèmes encore ouverts des mises à jour de septembre sont donc résolus : deux par la SU d’octobre, un par une correction dans Exchange Online (voir ci-dessus). Le problème de wrapper dans les boîtes aux lettres partagées était déjà corrigé avec la SU de septembre.

## Installation et suivi

Dans l’annonce, Microsoft indique la procédure habituelle : inventorier avec le [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), déterminer le chemin avec le [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) si la CU est obsolète, installer la SU, redémarrer le serveur et vérifier que tous les services Exchange sont en cours d’exécution. En cas d’erreurs, Microsoft renvoie vers le [script SetupAssist](https://aka.ms/ExSetupAssist). Après l’installation, le Health Checker indique, selon KB5128895, si la SU est correctement installée et si d’autres étapes sont nécessaires. Le Security Update Guide n’indique aucun redémarrage requis pour les builds d’octobre ; l’annonce le recommande néanmoins après l’installation.

Selon KB5128895, le fichier d’installation pour Exchange SE s’appelle `ExchangeSubscriptionEdition-KB5128895-x64-en.exe` (Download Center : « SU11 »), pour Exchange 2019 CU15 `Exchange2019-KB5128896-x64-en.exe`, pour CU14 `Exchange2019-KB5128897-x64-en.exe` et pour Exchange 2016 `Exchange2016-KB5128898-x64-en.exe`. Les packages sont spécifiques à chaque CU.

Selon la FAQ, dans les environnements hybrides, si le certificat d’authentification est modifié après l’installation d’une SU, il faut exécuter de nouveau le Hybrid Configuration Wizard. Sous Windows Server 2025, les SU Exchange installées n’apparaissent pas dans le Panneau de configuration ; Microsoft déconseille de désinstaller les SU.

Des mois précédents, les actions de suivi suivantes restent à effectuer si ce n’est pas déjà fait : supprimer le SettingOverride de wrapper `DisableBlockSharedAndUserMailboxHeaders` (voir l’[article de septembre](/blog/exchange-security-updates-september-2026)) et vérifier si la mitigation CVE-2026-42897 (M2.1.0) est toujours active (voir l’[article sur la SU de juillet](/blog/exchange-security-updates-juli-2026)).

## Procédure recommandée

Installez rapidement la SU d’octobre sur tous les serveurs Exchange et les machines équipées des Management Tools : l’exécution de code à distance critique CVE-2026-70571 peut être déclenchée sans compte par un e-mail, et Microsoft considère son exploitation tout aussi probable que celle du Cross-Site Scripting CVE-2026-70279. Les serveurs disposant de la première SU de septembre reçoivent ainsi également le correctif pour CVE-2026-96940. Supprimez ensuite l’override Libre/Occupé s’il est défini et vérifiez les calendriers publiés. Pour Exchange 2016 et 2019, il s’agit de la dernière mise à jour ; planifiez dès maintenant la migration vers Exchange SE, car ces serveurs ne recevront plus de mises à jour de sécurité à partir de novembre.

## Sources

1.  [Released: October 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-october-2026-exchange-server-security-updates/ba-p/4563014) : annonce du 13 octobre 2026 avec les versions prises en charge, les problèmes corrigés, la fin des mises à jour pour Exchange 2016/2019 et la FAQ (consultée via le flux RSS du Exchange Team Blog ; aucun commentaire de l’équipe Exchange, état au 14 octobre 2026).

2.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM October 13, 2026 (KB5128895) – Microsoft Support](https://support.microsoft.com/help/5128895) : liste des CVE, problèmes corrigés, fichier d’installation et référence au Health Checker.

3.  [Description of the security update for Microsoft Exchange Server 2019 CU15 October 13, 2026 (KB5128896) – Microsoft Support](https://support.microsoft.com/help/5128896) : article KB pour Exchange 2019 CU15 avec indication ESU.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU14 October 13, 2026 (KB5128897) – Microsoft Support](https://support.microsoft.com/help/5128897) : article KB pour Exchange 2019 CU14 avec adresse de contact ESU.

5.  [Description of the security update for Microsoft Exchange Server 2016 CU23 October 13, 2026 (KB5128898) – Microsoft Support](https://support.microsoft.com/help/5128898) : article KB pour Exchange 2016 CU23.

6.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/) : type, CVSS, gravité, évaluation de l’exploitation, FAQ et builds affectés des six CVE d’octobre ainsi que de CVE-2026-96940.

7.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates) : numéros de build des SU d’octobre et des mises à jour précédentes.

8.  [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672) : correctif avec les mises à jour d’octobre.

9.  [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098) : correctif avec KB5128895, aucune autre action nécessaire.

10. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) : correction dans le service et commandes pour réinitialiser le SettingOverride.

11. [Nouvelles mises à jour de sécurité pour Exchange Server (octobre 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-oktober-2026/) : analyse en allemand des six CVE, indication du correctif v2 inclus et évaluation de la fin de l’ESU.

12. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker) : inventaire des versions CU/SU avant et après l’installation.
