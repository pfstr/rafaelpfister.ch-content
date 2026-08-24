---
title: "Finaliser correctement les mises à jour de sécurité Exchange de juillet 2026"
navTitle: "Exchange SU 07/2026"
description: "Après l’installation, deux opérations de nettoyage sont nécessaires : supprimer de manière contrôlée l’ancienne mitigation CVE-2026-42897 et vérifier les groupes hérités disposant de privilèges excessifs dans Active Directory."
date: "2026-07-14"
kategorie: "Exchange OnPrem / Hybride"
timeToRead: "6 min de lecture"
themen:
  - exchange-updates
  - active-directory-entra
slug: "finaliser-correctement-les-mises-a-jour-de-securite-exchange-de-juillet-2026"
translationOf: "exchange-security-updates-juli-2026"
url: "https://rafaelpfister.ch/fr/blog/finaliser-correctement-les-mises-a-jour-de-securite-exchange-de-juillet-2026"
translationId: article-731b5b840aee096c
translationReview: automatic
translationSourceHash: c4f0a68a6d0b88997bcc5dadd9f5c2423dcb61c7986e179a099460335042a23a
translatedAt: 2026-07-29T12:29:38.934Z
---

# Finaliser correctement les mises à jour de sécurité Exchange de juillet 2026

L’installation des mises à jour de sécurité Exchange du 14 juillet 2026 ne clôt pas le travail. Les administrateurs doivent ensuite éliminer deux héritages : la mitigation activée en mai pour **CVE-2026-42897** et deux groupes de sécurité Exchange historiques disposant de droits étendus dans Active Directory.

Ces deux tâches sont faciles à manquer. La mitigation reste volontairement en place jusqu’à ce qu’elle soit supprimée de manière contrôlée. Quant aux groupes, ils peuvent avoir survécu inaperçus à chaque migration pendant de nombreuses années.

## Versions d’Exchange pour lesquelles la mise à jour est disponible

Les SU sont disponibles pour les versions suivantes :

- **Exchange Server Subscription Edition (SE) RTM** : en tant que mise à jour publique normalement disponible.
- **Exchange Server 2019 CU14 et CU15** : uniquement pour les organisations inscrites au **programme ESU période 2**.
- **Exchange Server 2016 CU23** : également uniquement via l’ESU période 2.

Exchange 2016 et 2019 ne sont plus pris en charge. Les organisations qui ne participent pas au programme ESU période 2 (valable de mai à octobre 2026) ne reçoivent plus ces mises à jour et ne devraient plus reporter le passage à Exchange SE. Les environnements Exchange Online sont déjà protégés ; dans les configurations hybrides, le SU doit néanmoins être installé sur tous les serveurs Exchange, y compris les serveurs dédiés uniquement à l’administration. Les CVE concrètement corrigées sont répertoriées, comme d’habitude, dans le Security Update Guide (filtre « Server Software » pour Exchange SE ou « ESU » pour 2016/2019).

Un problème connu existe dans la version actuelle : dans les environnements hybrides, des *messages wrapper* peuvent apparaître dans la boîte de réception des boîtes aux lettres partagées. Les détails figurent dans l’article de support Microsoft correspondant.

## Supprimer la mitigation CVE-2026-42897 après l’installation

### Bref retour en arrière

CVE-2026-42897 a été annoncée le 14 mai 2026 : une vulnérabilité de type cross-site scripting (usurpation) dans Outlook Web Access. Un attaquant envoie un e-mail spécialement conçu ; si la victime l’ouvre dans OWA et que certaines conditions d’interaction sont réunies, du JavaScript arbitraire peut être exécuté dans le contexte du navigateur. Exchange 2016, 2019 et SE étaient concernés à *tous* les niveaux de correctifs. Microsoft a publié le jour même une mitigation d’urgence (ID **M2.1.x**, la règle IIS concrète s’appelle **M2.1.0**) et a fourni le correctif proprement dit avec le SU de juin 2026.

### Pourquoi la mise à jour de juillet ne supprime *pas* automatiquement la mitigation

C’est le point qui surprend le plus : même après l’installation du SU de juillet, une mitigation déjà appliquée reste active. La raison tient au mécanisme. La mitigation est une **règle IIS URL Rewrite basée sur Content-Security-Policy**, appliquée *en dehors* de l’installateur MSI, soit par l’Emergency Mitigation Service (service EM), soit par le script EOMT. Le correctif MSI remplace les fichiers binaires, mais ne gère pas ces règles IIS définies hors bande. La suppression constitue donc une étape manuelle distincte.

À noter : la mitigation n’a jamais protégé les clients IE ni Edge en mode IE, car Internet Explorer ne prend pas en charge CSP. Les organisations utilisant de tels clients n’ont donc jamais été protégées uniquement par la mitigation. C’est un argument supplémentaire pour appliquer rapidement les correctifs plutôt que de s’appuyer sur la mitigation.

### Le piège : le service EM réapplique la mitigation

Toute personne supprimant prématurément la règle aura une surprise. Le service EM s’exécute toutes les heures et compare l’état actuel aux paramètres fournis par l’Office Config Service (Flighting). L’association « quelle build nécessite quelle mitigation » est gérée côté serveur. Seule une modification côté serveur marque la build de juillet 2026 comme « mitigation non nécessaire ». Selon Microsoft, cette modification n’a été entièrement déployée qu’autour du 16 juillet 2026. Jusqu’alors, le service EM réintroduit simplement une règle M2.1.0 supprimée lors de son prochain cycle horaire.

En pratique, cela signifie : soit attendre après le 16 juillet pour la suppression manuelle, soit bloquer explicitement la mitigation afin qu’elle ne soit pas réactivée.

### Supprimer proprement la mitigation (chemin du service EM)

Commencez par vérifier ce qui est effectivement appliqué :

```powershell
Get-ExchangeServer -Identity <NomDuServeur> | Format-List Name,MitigationsApplied,MitigationsBlocked
```

Pour empêcher la réactivation, l’ID de mitigation est ajoutée à la liste de blocage : les entrées de cette liste sont ignorées par le service EM lors de son cycle horaire.

```powershell
Set-ExchangeServer -Identity <NomDuServeur> -MitigationsBlocked @("M2.1.0")
```

Supprimez ensuite la règle IIS proprement dite. Bon à savoir, bien que rarement documenté : le service EM crée ses règles URL Rewrite avec le **préfixe « EEMS `<Mitigation-ID>` `<Beschreibung>` »**. Cela permet de les identifier sans ambiguïté dans le gestionnaire IIS sous URL Rewrite (ou via `appcmd`/PowerShell dans le `applicationHost.config`), sans devoir deviner quelle règle appartient à la mitigation. Après le déploiement de la modification côté serveur, le blocage peut être levé à nouveau (`-MitigationsBlocked @()`) s’il n’avait été défini que comme solution temporaire.

### Chemin EOMT (environnements isolés ou air-gapped)

Si la mitigation a été appliquée à l’aide du **script EOMT** téléchargeable (https://aka.ms/UnifiedEOMT), la restauration s’effectue avec le commutateur de rollback :

```powershell
.\EOMT.ps1 -RollbackMitigation -CVE "CVE-2026-42897"
```

Ici aussi, un détail peu connu : avant chaque modification, EOMT sauvegarde l’état initial IIS dans un **fichier de sauvegarde JSON spécifique à la CVE** sous `%WINDIR%\System32\inetsrv\config\`. Le rollback lit précisément ce fichier et restaure les paramètres d’origine. Important : une mitigation définie avec un script hérité (EOMTv2, etc.) doit aussi être supprimée avec son propre mécanisme de rollback : les formats de sauvegarde ne sont pas compatibles.

### Pourquoi la suppression en vaut la peine

La mitigation n’est pas « gratuite ». Tant qu’elle est active, elle entraîne ses effets secondaires connus : la fonction OWA « Imprimer le calendrier » ne fonctionne pas, les images en ligne peuvent ne pas s’afficher correctement dans le volet de lecture OWA, OWA Light (`/?layout=light`) est défectueux (il sera de toute façon prochainement désactivé), et les calendriers publiés renvoient parfois des erreurs 500. Particulièrement trompeur pour la supervision : le health set **OWACalendar.Proxy** peut passer à l’état *unhealthy* et déclencher ainsi de fausses alertes dans la surveillance. Ceux qui ont installé le SU mais laissent la mitigation en place finissent par poursuivre des fantômes. Dès que la mise à jour est installée *et* que la mitigation est supprimée, ces problèmes connus disparaissent également.

Cas particulier : dans les environnements mixtes, les serveurs pas encore mis à jour peuvent conserver la mitigation. Il faut toutefois savoir que l’intégration d’Office Online Server (OOS) peut ne fonctionner correctement à nouveau que lorsque *tous* les serveurs Exchange de l’organisation sont au niveau de juillet.

## Health Checker : repérer des groupes de sécurité très anciens

Le deuxième point, indépendant de la publication du SU : l’**Exchange Health Checker** (https://aka.ms/ExchangeHealthChecker) vérifie désormais l’existence de deux groupes de sécurité abandonnés depuis longtemps : **« Exchange Domain Servers »** et **« Exchange Enterprise Servers »**.

### Origine de ces groupes et raison de leur risque

Ces deux groupes proviennent du modèle d’autorisations d’Exchange 2000/2003 et sont obsolètes depuis Exchange 2007. Exchange 2007/2010 a introduit le modèle de Split Permissions, puis RBAC, et ils ne sont plus utilisés depuis. Le problème : ils n’ont pas pour autant disparu. Dans de nombreux annuaires, ils sont restés ignorés pendant près de deux décennies et portent parfois encore des ACL étendues de l’ancien modèle, donc davantage de droits qu’un groupe de sécurité Exchange moderne n’en aurait jamais.

C’est précisément ce qui en fait un vecteur d’attaque. Un groupe dormant doté d’autorisations larges persistantes constitue une chaîne d’escalade classique : toute personne qui parvient à s’ajouter (ou à ajouter un compte qu’elle contrôle) à un tel groupe hérite de ses droits dans l’annuaire. Comme personne ne surveille activement le groupe, une telle manipulation passe difficilement inaperçue.

### Pourquoi la plupart des administrateurs n’y pensent pas

Ces groupes constituent un angle mort pour plusieurs raisons : ils sont inactifs depuis environ 20 ans, existaient généralement avant l’arrivée de l’équipe actuelle, survivent sans problème à toutes les migrations et n’étaient jusqu’ici jamais signalés par le Health Checker. Point particulièrement délicat : ils survivent même à la mise hors service *complète* d’Exchange on-premises. Les personnes qui retirent le dernier serveur Exchange suppriment généralement les objets serveur, mais passent entièrement à côté de ces groupes hérités.

### Nettoyage

Le Health Checker signalera désormais automatiquement ces groupes. Manuellement, ils peuvent être trouvés dans Active Directory (généralement dans le conteneur `Users`) ou via PowerShell :

```powershell
Get-ADGroup -Filter "Name -eq 'Serveurs de domaine Exchange' -or Name -eq 'Serveurs d’entreprise Exchange'"
```

Procédure : vérifiez les appartenances et les éventuelles références ACL personnalisées, assurez-vous qu’aucun élément de production ne s’y réfère, puis supprimez les groupes. Étant obsolètes depuis 2007, ils peuvent être supprimés sans risque dans l’immense majorité des environnements. Les organisations qui n’exploitent plus aucun Exchange on-premises devraient également prévoir un nettoyage AD plus complet conformément à la documentation officielle Microsoft.

Hayes Jupe a publié dans son billet de blog [Latest Exchange health check script and deprecated groups](https://www.hayesjupe.com/latest-exchange-health-check-script-and-deprecated-groups/) une procédure détaillée pour supprimer ces groupes.

## Procédure recommandée

En résumé, voici le déroulement pratique : commencez par inventorier l’environnement avec le Health Checker (il indique les CU/SU manquants, les étapes manuelles en attente *et* désormais les groupes hérités). Installez ensuite le CU actuel et le SU de juillet, redémarrez le serveur et vérifiez que tous les services Exchange ont démarré correctement. Exécutez ensuite de nouveau le Health Checker, supprimez la mitigation CVE-2026-42897 (après le 16 juillet ou après avoir préalablement bloqué l’ID M2.1.0), puis nettoyez les groupes de sécurité obsolètes. Les SU sont cumulatives : si vous utilisez un CU pris en charge, il n’est pas nécessaire d’installer chaque SU intermédiaire ; installez directement la plus récente.

## Sources

1.  [Released: July 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-july-2026-exchange-server-security-updates/4534146): Annonce officielle de la version de juillet avec les versions prises en charge et le problème connu des messages wrapper.

2.  [Addressing Exchange Server May 2026 vulnerability CVE-2026-42897 – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/addressing-exchange-server-may-2026-vulnerability-cve-2026-42897/4518498): Avis de sécurité initial, incluant la mitigation d’urgence et les effets secondaires connus dans OWA.

3.  [Released: June 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-june-2026-exchange-server-security-updates/4524491): Version de juin qui a fourni le correctif proprement dit pour CVE-2026-42897.

4.  [Exchange Emergency Mitigation Service (Exchange EM Service) – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-emergency-mitigation-service): Fonctionnement du service EM, qui compare les mitigations toutes les heures et réintroduit une règle supprimée prématurément.

5.  [Set-ExchangeServer (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-exchangeserver): Paramètres `MitigationsApplied` et `MitigationsBlocked` permettant de vérifier les mitigations et d’empêcher leur réactivation.

6.  [Exchange On-premises Mitigation Tool (EOMT) – Microsoft CSS-Exchange](https://microsoft.github.io/CSS-Exchange/Security/EOMT/): Le script EOMT, incluant le commutateur de rollback et la sauvegarde JSON spécifique à la CVE de l’état initial IIS.

7.  [CVE-2026-42897 Detail – NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-42897): Description technique et évaluation de la vulnérabilité dans la National Vulnerability Database.
