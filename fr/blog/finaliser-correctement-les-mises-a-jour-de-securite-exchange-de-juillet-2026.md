---
title: "Assurer correctement le suivi des mises à jour de sécurité Exchange de juillet 2026"
navTitle: "Exchange SU 07/2026"
description: "Après l’installation, deux tâches de nettoyage sont nécessaires : supprimer de manière contrôlée l’ancienne mitigation CVE-2026-42897 et vérifier les groupes hérités sur-privilégiés dans Active Directory."
date: "2026-07-14"
kategorie: "Exchange OnPrem / hybride"
timeToRead: "6 min de lecture"
themen:
  - exchange-updates
  - active-directory-entra
slug: "finaliser-correctement-les-mises-a-jour-de-securite-exchange-de-juillet-2026"
translationOf: "exchange-security-updates-juli-2026"
translationId: article-731b5b840aee096c
translationReview: automatic
translationSourceHash: e5d9295515965d3e7801752cd605f6d2a78cacfc9fb965e0f63d645658b39e9b
translatedAt: 2026-09-05T07:51:47.644Z
url: https://rafaelpfister.ch/fr/blog/finaliser-correctement-les-mises-a-jour-de-securite-exchange-de-juillet-2026
translationModel: gpt-5.6-terra
---

# Assurer correctement le suivi des mises à jour de sécurité Exchange de juillet 2026

L’installation des mises à jour de sécurité Exchange du 14 juillet 2026 ne clôt pas le travail. Les administrateurs devraient ensuite éliminer deux éléments hérités : la mitigation activée en mai pour **CVE-2026-42897** et deux groupes de sécurité Exchange historiques disposant de droits étendus dans Active Directory.

Ces deux tâches sont faciles à négliger. La mitigation reste volontairement en place jusqu’à ce qu’elle soit supprimée de manière contrôlée. Quant aux groupes, ils peuvent avoir survécu discrètement à toutes les migrations pendant de nombreuses années.

## Versions d’Exchange pour lesquelles la mise à jour est disponible

Les SU sont disponibles pour les versions suivantes :

- **Exchange Server Subscription Edition (SE) RTM** : sous forme de mise à jour publique disponible normalement.
- **Exchange Server 2019 CU14 et CU15** : uniquement pour les organisations inscrites au **programme ESU Period 2**.
- **Exchange Server 2016 CU23** : également uniquement via ESU Period 2.

Exchange 2016 et 2019 ne sont plus pris en charge. Les organisations qui ne participent pas au programme ESU Period 2 (valable de mai à octobre 2026) ne reçoivent plus ces mises à jour et ne devraient plus repousser la migration vers Exchange SE. Les environnements Exchange Online sont déjà protégés ; dans les configurations hybrides, le SU doit néanmoins être installé sur tous les serveurs Exchange, y compris les serveurs de gestion uniquement. Les CVE concrètement traitées figurent, comme d’habitude, dans le Security Update Guide (filtre « Server Software » pour Exchange SE ou « ESU » pour 2016/2019).

Un problème connu existe dans la version actuelle : dans les environnements hybrides, des *messages wrapper* peuvent apparaître dans la boîte de réception de boîtes aux lettres partagées. Les détails sont disponibles dans l’article de support Microsoft correspondant.

## Supprimer la mitigation CVE-2026-42897 après l’installation

### Bref retour en arrière

CVE-2026-42897 a été annoncée le 14 mai 2026 : une vulnérabilité de cross-site scripting (usurpation) dans Outlook Web Access. Un attaquant envoie un e-mail spécialement conçu ; si la victime l’ouvre dans OWA et que certaines conditions d’interaction sont remplies, du JavaScript arbitraire peut être exécuté dans le contexte du navigateur. Exchange 2016, 2019 et SE étaient concernés à *tous* les niveaux de correctifs. Microsoft a publié le jour même une mitigation d’urgence (ID **M2.1.x**, la règle IIS concrète s’appelle **M2.1.0**) et a livré le correctif proprement dit avec le SU de juin 2026.

### Pourquoi la mise à jour de juillet ne supprime *pas* automatiquement la mitigation

C’est le point qui surprend le plus : même après l’installation du SU de juillet, une mitigation déjà appliquée reste active. La raison réside dans son mécanisme. La mitigation est une **règle IIS URL Rewrite basée sur Content Security Policy**, appliquée *en dehors* de l’installateur MSI, soit par l’Emergency Mitigation Service (service EM), soit par le script EOMT. Le correctif MSI remplace les fichiers binaires, mais ne gère pas ces règles IIS définies hors bande. Leur suppression est donc une étape manuelle distincte.

À noter : la mitigation n’a de toute façon jamais protégé les clients IE et Edge en mode IE, car Internet Explorer ne prend pas en charge CSP. Les organisations utilisant de tels clients n’ont donc jamais été protégées par la seule mitigation. C’est un argument supplémentaire pour appliquer rapidement les correctifs plutôt que de se fier à la mitigation.

### Le point délicat : le service EM réapplique la mitigation

Une règle supprimée prématurément ne reste pas supprimée durablement. Le service EM s’exécute toutes les heures et compare l’état actuel aux spécifications fournies par le service Office Config (Flighting). L’association « quelle build requiert quelle mitigation » est gérée côté serveur. Seule une modification côté serveur marque la build de juillet 2026 comme « mitigation non nécessaire ». Selon Microsoft, cette modification n’a été entièrement déployée qu’aux alentours du 16 juillet 2026. Jusque-là, le service EM réajoute simplement une règle M2.1.0 supprimée lors de son prochain cycle horaire.

En pratique, cela signifie qu’il faut soit attendre après le 16 juillet avant de procéder à la suppression manuelle, soit bloquer explicitement la mitigation afin qu’elle ne soit pas réactivée.

### Supprimer proprement la mitigation (via le service EM)

Commencez par vérifier ce qui est réellement appliqué :

```powershell
Get-ExchangeServer -Identity <Servername> | Format-List Name,MitigationsApplied,MitigationsBlocked
```

Pour empêcher sa réactivation, ajoutez l’ID de mitigation à la liste de blocage : les entrées qui y figurent sont ignorées par le service EM lors du cycle horaire.

```powershell
Set-ExchangeServer -Identity <Servername> -MitigationsBlocked @("M2.1.0")
```

Supprimez ensuite la règle IIS elle-même. Bon à savoir, et rarement documenté : le service EM crée ses règles URL Rewrite avec le **préfixe « EEMS `<Mitigation-ID>` `<Beschreibung>`»**. Cela permet de les identifier sans ambiguïté dans le Gestionnaire IIS sous URL Rewrite (ou via `appcmd`/PowerShell dans `applicationHost.config`), sans avoir à deviner quelle règle appartient à la mitigation. Après le déploiement de la modification côté serveur, vous pouvez lever le blocage (`-MitigationsBlocked @()`), si vous ne l’aviez défini que comme solution temporaire.

### Parcours EOMT (environnements séparés ou isolés du réseau)

Si la mitigation a été appliquée au moyen du **script EOMT** téléchargeable (https://aka.ms/UnifiedEOMT), le retour en arrière s’effectue avec le commutateur de restauration :

```powershell
.\EOMT.ps1 -RollbackMitigation -CVE "CVE-2026-42897"
```

Ici aussi, un détail peu connu : avant chaque modification, EOMT sauvegarde l’état initial d’IIS dans un **fichier de sauvegarde JSON spécifique au CVE** sous `%WINDIR%\System32\inetsrv\config\`. La restauration lit précisément ce fichier et rétablit les paramètres d’origine. Important : une mitigation appliquée avec un script hérité (EOMTv2, etc.) doit également être supprimée avec son propre mécanisme de restauration : les formats de sauvegarde ne sont pas compatibles.

### Pourquoi il vaut la peine de la supprimer

La mitigation n’est pas « gratuite ». Tant qu’elle reste active, elle entraîne ses effets secondaires connus : la fonction OWA « Imprimer le calendrier » ne fonctionne pas, les images en ligne peuvent ne pas s’afficher correctement dans le volet de lecture OWA, OWA Light (`/?layout=light`) est défectueux (et sera de toute façon bientôt désactivé), et les calendriers publiés renvoient parfois des erreurs 500. Point particulièrement trompeur pour la supervision : le health set **OWACalendar.Proxy** peut passer à l’état *unhealthy*, déclenchant ainsi de fausses alertes. Une organisation qui a installé le SU mais laisse la mitigation en place finit par rechercher des erreurs qui n’en sont pas. Dès que la mise à jour est installée *et* la mitigation supprimée, ces problèmes connus disparaissent également.

Cas particulier : dans les environnements mixtes, les serveurs qui n’ont pas encore été mis à jour peuvent conserver la mitigation. Il faut toutefois savoir que l’intégration Office Online Server (OOS) peut ne fonctionner à nouveau correctement que lorsque *tous* les serveurs Exchange de l’organisation sont au niveau de juillet.

## Health Checker : détecter des groupes de sécurité très anciens

Le second point, indépendant de la publication du SU : l’**Exchange Health Checker** (https://aka.ms/ExchangeHealthChecker) vérifie désormais l’existence de deux groupes de sécurité obsolètes depuis longtemps : **« Exchange Domain Servers »** et **« Exchange Enterprise Servers »**.

### Origine de ces groupes et raisons pour lesquelles ils représentent un risque

Ces deux groupes proviennent du modèle d’autorisations d’Exchange 2000/2003 et sont obsolètes depuis Exchange 2007. Exchange 2007/2010 a introduit le modèle Split Permissions, ou RBAC, et depuis lors ils ne sont tout simplement plus utilisés. Le problème : ils n’ont pas pour autant disparu. Dans de nombreux annuaires, ils sont restés ignorés pendant près de deux décennies et portent parfois encore des ACL étendues issues de l’ancien modèle, soit davantage de droits qu’un groupe de sécurité Exchange moderne n’en aurait jamais.

C’est précisément ce qui en fait un vecteur d’attaque. Un groupe inactif doté d’autorisations larges persistantes constitue une chaîne d’escalade classique : quiconque parvient à s’ajouter, ou à ajouter un compte qu’il contrôle, à un tel groupe hérite de ses droits dans l’annuaire. Comme personne ne surveille activement ce groupe, une telle manipulation passe difficilement inaperçue.

### Pourquoi la plupart des administrateurs ne les connaissent pas

Ces groupes constituent un angle mort pour plusieurs raisons : ils sont inactifs depuis environ 20 ans, existaient généralement avant l’arrivée de l’équipe actuelle, survivent sans problème à toute migration et n’avaient jusqu’ici jamais été signalés par le Health Checker. Point particulièrement délicat : ils survivent même à la mise hors service *complète* d’Exchange on-premises. Lorsqu’un administrateur supprime le dernier serveur Exchange, il nettoie généralement les objets serveur, mais néglige entièrement ces groupes hérités.

### Nettoyage

Le Health Checker signalera désormais automatiquement ces groupes. Vous pouvez les trouver manuellement dans Active Directory (généralement dans le conteneur `Users`) ou avec PowerShell :

```powershell
Get-ADGroup -Filter "Name -eq 'Exchange Domain Servers' -or Name -eq 'Exchange Enterprise Servers'"
```

Procédure : vérifiez les appartenances et les éventuelles références ACL personnalisées, assurez-vous qu’aucun élément de production n’y fait référence, puis supprimez les groupes. Étant donné qu’ils sont obsolètes depuis 2007, ils peuvent être supprimés sans risque dans la très grande majorité des environnements. Les organisations qui n’exploitent plus du tout Exchange on-premises devraient en profiter pour planifier un nettoyage AD plus complet selon les instructions officielles de Microsoft.

Hayes Jupe a publié un guide détaillé pour supprimer ces groupes dans son article de blog [Latest Exchange health check script and deprecated groups](https://www.hayesjupe.com/latest-exchange-health-check-script-and-deprecated-groups/) écrit.

## Procédure recommandée

En résumé, voici la marche à suivre : commencez par inventorier l’environnement avec le Health Checker (il affiche les CU/SU manquants, les étapes manuelles en attente *et* désormais les groupes hérités). Installez ensuite le CU actuel ainsi que le SU de juillet, redémarrez le serveur et vérifiez que tous les services Exchange ont démarré correctement. Exécutez ensuite à nouveau le Health Checker, supprimez la mitigation CVE-2026-42897 (après le 16 juillet ou après avoir préalablement bloqué l’ID M2.1.0) et nettoyez enfin les groupes de sécurité obsolètes. Les SU sont cumulatifs : si vous utilisez un CU pris en charge, il n’est pas nécessaire d’installer chaque SU intermédiaire ; installez directement le plus récent.

## Sources

1.  [Released: July 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-july-2026-exchange-server-security-updates/4534146): Annonce officielle de la version de juillet avec les versions prises en charge et le problème connu des messages wrapper.

2.  [Addressing Exchange Server May 2026 vulnerability CVE-2026-42897 – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/addressing-exchange-server-may-2026-vulnerability-cve-2026-42897/4518498): Avis de sécurité initial avec la mitigation d’urgence et les effets secondaires connus dans OWA.

3.  [Released: June 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-june-2026-exchange-server-security-updates/4524491): Version de juin qui a livré le correctif proprement dit pour CVE-2026-42897.

4.  [Exchange Emergency Mitigation Service (Exchange EM Service) – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-emergency-mitigation-service): Fonctionnement du service EM, qui compare les mitigations chaque heure et réajoute une règle supprimée prématurément.

5.  [Set-ExchangeServer (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-exchangeserver): Paramètres `MitigationsApplied` et `MitigationsBlocked` permettant de vérifier les mitigations et d’empêcher leur réactivation.

6.  [Exchange On-premises Mitigation Tool (EOMT) – Microsoft CSS-Exchange](https://microsoft.github.io/CSS-Exchange/Security/EOMT/): Le script EOMT, y compris le commutateur de restauration et la sauvegarde JSON spécifique au CVE de l’état initial d’IIS.

7.  [CVE-2026-42897 Detail – NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-42897): Description technique et évaluation de la vulnérabilité dans la National Vulnerability Database.
