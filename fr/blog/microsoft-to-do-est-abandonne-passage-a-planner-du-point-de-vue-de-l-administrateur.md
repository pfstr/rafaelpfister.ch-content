---
title: "Microsoft To Do est abandonné : passage à Planner du point de vue de l’administrateur"
navTitle: "To Do est abandonné"
description: "Microsoft mettra fin aux applications To Do le 30 juin 2027. Les comptes professionnels passent à Planner, les tâches existantes ne sont pas migrées et Planner stocke les nouvelles tâches privées en dehors de la boîte aux lettres Exchange. Pour les tenants suisses, le lieu de stockage des données et la couverture de conformité changent ; Microsoft Graph permet d’identifier les utilisateurs concernés."
date: "2026-11-24"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min de lecture"
themen:
  - microsoft-365-exchange
produkte:
  - "planner"
  - "exchange-online"
protokolle:
  - "migration"
  - "powershell"
hauptthema: "microsoft-365-exchange"
slug: "microsoft-to-do-est-abandonne-passage-a-planner-du-point-de-vue-de-l-administrateur"
translationId: "article-7691ba2aae63c942"
translationOf: microsoft-to-do-planner-umstieg
url: https://rafaelpfister.ch/fr/blog/microsoft-to-do-est-abandonne-passage-a-planner-du-point-de-vue-de-l-administrateur
translationSourceHash: ae78db5df03b2677ad2bda294efdd3b1f31ee233d2de5ea0238c2de865396fe8
translationModel: gpt-5.6-terra
translatedAt: 2026-11-24T12:58:19.727Z
translationReview: automatic
---

# Microsoft To Do est abandonné : passage à Planner du point de vue de l’administrateur

Microsoft abandonne Microsoft To Do. Les applications Web, Windows, Mac, iOS et Android seront arrêtées le 30 juin 2027, aussi bien pour les comptes professionnels et scolaires que pour les comptes Microsoft personnels. Pour les comptes professionnels et scolaires, Microsoft Planner prend en charge les tâches personnelles ; la vue remaniée « Mes tâches » (My tasks) sera déployée à partir de décembre 2026. Microsoft a publié ce changement en novembre 2026 dans un article de support comprenant le calendrier, les différences et des indications destinées aux administrateurs.

En bref : il n’existe aucune migration automatique. Les tâches To Do existantes resteront visibles dans Planner jusqu’au 30 juin 2027 et continueront d’être stockées dans la boîte aux lettres Exchange Online. À partir du déploiement, Planner enregistre les nouvelles tâches privées dans un plan Planner privé, donc plus dans Exchange. Cela modifie le lieu de stockage des données et la couverture Purview pour les tenants suisses. Vérifiez d’ici mi-2027 les licences, les listes partagées, les intégrations et les obligations de conservation. Les comptes Microsoft personnels ne disposent d’aucune solution de remplacement.

## Calendrier

| Date | Changement |
|---|---|
| Décembre 2026 | Début du déploiement de la vue remaniée « Mes tâches » dans Planner pour les comptes professionnels et scolaires. À partir de cette date, Planner enregistre les tâches privées nouvellement créées dans un plan Planner privé. |
| avant le 30 juin 2027 | Microsoft fournira une fonction d’exportation des données To Do. Le format, l’étendue et les conditions ne sont pas encore publiés. |
| 30 juin 2027 | Les applications To Do pour le Web, Windows, Mac, iOS et Android sont abandonnées. Les tâches To Do existantes ne seront alors plus visibles non plus dans Planner. |
| ultérieurement, sans date | Le service To Do, les API et la synchronisation seront retirés séparément. Microsoft annoncera les dates distinctement. |

Planner est déjà disponible aujourd’hui pour les comptes professionnels sur le Web à l’adresse `planner.cloud.microsoft` et comme application dans Teams. Microsoft annonce qu’il adaptera les points d’accès existants de To Do dans Microsoft 365. Microsoft a précisé à Thurrott que les intégrations existantes, notamment Outlook et Teams, seront basculées vers Planner.

## Les tâches existantes ne sont pas migrées

Microsoft ne transfère pas les tâches et listes To Do existantes vers Planner. Elles peuvent encore être affichées et modifiées dans Planner jusqu’au 30 juin 2027, mais plus après. Elles ne sont pas reprises dans le nouveau plan privé.

Les données ne sont pas supprimées à la fin des applications. Selon Microsoft, le retrait des applications ne prévoit aucune suppression ; Microsoft traite séparément la conservation et l’accès. To Do stocke les tâches comme tâches Exchange dans la boîte aux lettres Exchange Online de l’utilisateur, dans la même boîte aux lettres que les e-mails, rendez-vous, contacts et notes. Les données restent donc provisoirement dans la boîte aux lettres après le 30 juin 2027.

Outlook classique n’est pas concerné par le retrait. Microsoft indique que sa fonction de tâches est distincte de To Do, continue de fonctionner et se synchronise avec Exchange. Les listes To Do apparaissent dans Outlook classique comme dossiers de tâches sous « Mes tâches » ; les utilisateurs d’Outlook classique peuvent donc continuer à y consulter les données. Microsoft n’a pas encore décrit ce qu’il adviendra des données lorsqu’il arrêtera ultérieurement le service To Do lui-même.

## Les nouvelles tâches sont stockées en dehors d’Exchange

Avec la nouvelle vue, le lieu de stockage change. Selon la FAQ destinée aux administrateurs Planner, les tâches de To Do et Outlook se trouvent dans Exchange, tandis que les plans et leurs tâches se trouvent dans Azure. Planner stocke les pièces jointes des tâches de plan dans le site SharePoint du groupe, et les plans Premium dans Dataverse. Les nouvelles tâches privées de « Mes tâches » ne se trouvent donc plus dans la boîte aux lettres de l’utilisateur. Cela a deux conséquences pour les administrateurs.

### Lieu de stockage des données

Exchange Online fait partie des services bénéficiant d’un engagement permanent concernant le lieu de stockage des données. Si un tenant est créé en Suisse, Microsoft stocke, selon les Product Terms, le contenu des boîtes aux lettres au repos (texte des e-mails, entrées de calendrier et pièces jointes) uniquement en Suisse. Cet engagement ne mentionne pas explicitement les tâches, mais elles se trouvent dans la même boîte aux lettres.

Microsoft classe Planner parmi les autres services Microsoft 365 sans engagement permanent. Pour les tenants dont le pays est la Suisse, le tableau indique « EUR1 », la macro-région Europe, comme lieu de stockage. Planner ne participe pas non plus à Advanced Data Residency. Les tâches personnelles que To Do stockait jusqu’ici dans la boîte aux lettres en Suisse sont stockées par Planner, après le changement, dans la macro-région Europe. Les lieux de stockage par service pour votre tenant sont indiqués dans le Centre d’administration Microsoft 365 sur la carte de localisation des données (Data Location Card). Les organisations qui ont garanti un lieu de stockage des données en Suisse contractuellement ou dans leurs directives internes doivent évaluer ce changement.

### Conservation et eDiscovery

Les stratégies de rétention Exchange dans Microsoft Purview ne couvrent les tâches de la boîte aux lettres que si elles ont une date de fin ; les tâches sans date de fin ne sont pas couvertes. Cette limitation s’applique déjà aujourd’hui aux tâches To Do.

Pour Planner, Purview prend en charge, selon Microsoft, la recherche eDiscovery et la conservation légale uniquement pour les tâches des plans partagés avec un groupe. Les plans personnels (plans Roster), les plans dans des composants Loop, les plans Premium ainsi que les plans dans les canaux Teams partagés et privés ne sont explicitement pas pris en charge. Microsoft déploie depuis la mi-juillet 2026 les stratégies de rétention et de suppression pour les contenus Planner ; la fin du déploiement était annoncée pour début août (MC1384412). Selon l’annonce, elles peuvent être appliquées aux plans basés sur les utilisateurs et les groupes. L’annonce ne précise pas à quel type de plan correspond techniquement le nouveau plan privé. Après le déploiement, vérifiez si vos stratégies de rétention et vos conservations légales couvrent les nouvelles tâches privées.

## Différences entre To Do et Planner

Microsoft énumère les différences suivantes :

| Domaine | Différence selon Microsoft |
|---|---|
| Organisation | Planner regroupe les tâches privées et le travail assigné sous « Mes tâches », aujourd’hui avec les vues Grille et Tableau. Une vue Liste suivra avec la refonte. |
| Rappels | Les notifications relatives aux dates d’échéance dans Planner diffèrent des rappels planifiés par tâche dans To Do. |
| Partage | Planner fonctionne avec des plans et des attributions. Les appartenances, attributions et autorisations des listes To Do partagées ne sont pas reprises. |
| Bureau et hors ligne | Planner est disponible sur le Web et dans Teams. Aucune application native pour Windows et Mac n’est prévue ; le Web et Teams n’offrent pas de fonction hors ligne comparable. |
| Catégories | Les étiquettes Planner fonctionnent différemment des catégories Outlook et ne sont pas synchronisées avec elles. |
| Pièces jointes | Les pièces jointes aux tâches ne sont actuellement pas prises en charge dans le nouveau plan Planner privé. |
| Étoile et priorité | L’étoile personnelle sert à la planification individuelle et arrive avec la nouvelle vue. La priorité d’une tâche s’applique à tous les participants d’un plan partagé. |
| Groupes de listes | Sans équivalent direct dans Planner. |
| Imprimer, envoyer une liste par e-mail | Pas actuellement prévu pour la solution de remplacement. |

Les utilisateurs qui travaillent hors ligne avec l’application To Do sous Windows ou Mac n’ont pas d’alternative équivalente dans Planner.

## Licences et accès à Planner

Planner est notamment inclus dans Microsoft 365 Business Basic, Business Standard et Business Premium, ainsi que dans Office 365 E1, E3 et E5. Si Planner manque dans une licence, Microsoft propose Planner Plan 1 ou Planner and Project Plan 3 sous forme d’abonnements distincts ; ils sont également nécessaires pour les fonctions Premium. Selon Microsoft, aucune nouvelle configuration n’est requise et il n’existe pas de commutateur spécifique permettant aux administrateurs de contrôler ou de reporter le changement.

Si Planner est désactivé dans l’attribution de licences (plan de service `PROJECTWORKMANAGEMENT`), les utilisateurs concernés doivent à nouveau y avoir accès si Planner doit remplacer To Do. Selon la FAQ Planner, supprimer la licence Planner ne masque de toute façon que la vignette : via `planner.cloud.microsoft`, les utilisateurs sans licence peuvent toujours créer et modifier des plans. Si vous avez jusqu’ici bloqué Planner via la licence, réévaluez cette décision au regard du changement.

## Inventorier l’existant avec Microsoft Graph

L’API To Do de Microsoft Graph permet de déterminer quels utilisateurs utilisent To Do et quelles listes sont partagées. Le script suivant écrit dans un fichier CSV toutes les listes de chaque utilisateur, avec leur statut de partage et le nombre de tâches ouvertes. Il nécessite les modules `Microsoft.Graph.Authentication` et `Microsoft.Graph.Users`, ainsi qu’une inscription d’application avec les autorisations d’application `Tasks.Read.All` et `User.Read.All` (avec consentement administrateur) et un certificat pour l’authentification. `Tasks.Read.All` accorde un accès en lecture aux tâches de tous les utilisateurs ; supprimez l’autorisation ou l’inscription d’application après l’inventaire.

```powershell
$verbindung = @{
    ClientId              = '<App-ID>'
    TenantId              = '<Tenant-ID>'
    CertificateThumbprint = '<Zertifikat-Fingerabdruck>'
}
Connect-MgGraph @verbindung -NoWelcome

$benutzer = Get-MgUser -All -Filter "accountEnabled eq true" -Property Id, UserPrincipalName

$bestand = foreach ($b in $benutzer) {
    try {
        $listen = Get-MgUserTodoList -UserId $b.Id -All -ErrorAction Stop
    }
    catch {
        # Aucune boîte aux lettres Exchange Online ou aucun accès : ignorer l’utilisateur
        continue
    }
    foreach ($l in $listen) {
        $aufgaben = @(Get-MgUserTodoTask -UserId $b.Id -TodoTaskListId $l.Id -All)
        [pscustomobject]@{
            Benutzer = $b.UserPrincipalName
            Liste    = $l.DisplayName
            Typ      = $l.WellknownListName
            Geteilt  = $l.IsShared
            Besitzer = $l.IsOwner
            Offen    = @($aufgaben | Where-Object Status -ne 'completed').Count
            Gesamt   = $aufgaben.Count
        }
    }
}

$bestand | Export-Csv -Path .\todo-bestand.csv -NoTypeInformation -Encoding UTF8
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `ClientId`, `TenantId` | Inscription d’application et tenant auxquels le script se connecte |
| `CertificateThumbprint` | Empreinte du certificat dans le magasin de certificats de l’utilisateur actuel ; authentification en tant qu’application sans compte utilisateur |
| `-NoWelcome` | Supprime le message d’accueil de `Connect-MgGraph` |
| `-All` | Lit toutes les pages d’un résultat au lieu de la première seulement |
| `-Filter "accountEnabled eq true"` | Comptes activés uniquement ; les comptes désactivés tels que les boîtes aux lettres partagées sont exclus |
| `-Property Id, UserPrincipalName` | Charge uniquement les deux propriétés nécessaires |
| `-UserId` | Utilisateur dont les listes ou tâches sont interrogées |
| `-ErrorAction Stop` | Rend une erreur interceptable afin que `catch` puisse ignorer l’utilisateur |
| `-TodoTaskListId` | Liste dont les tâches sont interrogées |
| `WellknownListName` | `defaultList` pour la liste standard « Tâches », `flaggedEmails` pour les messages marqués et `none` pour les listes personnalisées |
| `IsShared` | `True` lorsque la liste est partagée avec d’autres utilisateurs |
| `IsOwner` | `False` lorsque la liste appartient à un autre utilisateur et a seulement été partagée |
| `Status -ne 'completed'` | Compte toutes les tâches non terminées |
| `-NoTypeInformation` | Aucune ligne de type au début du fichier CSV |
| `-Encoding UTF8` | Les accents dans les noms de listes sont conservés |

</details>

Les comptes sans boîte aux lettres Exchange Online renvoient une erreur et sont ignorés. Dans les grands tenants, il est judicieux d’effectuer un test sur un sous-ensemble, par exemple avec `-Top 20` au lieu de `-All` dans `Get-MgUser`.

L’analyse du fichier CSV :

- **Listes partagées** (`Geteilt` = `True`) indique où la collaboration passe par To Do. Microsoft ne transfère pas ces appartenances ; les listes doivent être recréées comme plans Planner avec les personnes concernées.
- Les **tâches ouvertes** dans des listes personnelles (`Typ` = `none` ou `defaultList`) doivent être transférées manuellement par les utilisateurs vers Planner avant le 30 juin 2027 ou sauvegardées avec la fonction d’exportation annoncée.
- Les **messages marqués** (`Typ` = `flaggedEmails`) restent marqués dans l’e-mail de la boîte aux lettres.

## Comptes Microsoft personnels

Les applications To Do seront également abandonnées le 30 juin 2027 pour les comptes Microsoft personnels, y compris Microsoft 365 Personal et Family. Planner n’est pas disponible pour ces comptes. Microsoft recommande de choisir une autre gestion des tâches et de sauvegarder les données au moyen de la fonction d’exportation annoncée. Selon Microsoft, une exportation ne signifie pas que les données peuvent être importées dans Planner ou une autre application. Les utilisateurs ayant un compte professionnel et un compte personnel doivent vérifier séparément les deux comptes.

## Préparation pour les administrateurs

1. **Accès et licences :** Assurez-vous que tous les utilisateurs de To Do disposent d’une licence incluant Planner et que le plan de service `PROJECTWORKMANAGEMENT` est activé.

2. **Listes partagées et intégrations :** Inventoriez l’existant avec le script ci-dessus. Identifiez les flux Power Automate et applications tierces qui utilisent l’API To Do de Microsoft Graph. Microsoft retirera l’API séparément et ultérieurement ; selon Microsoft, les intégrations peuvent néanmoins nécessiter des adaptations.

3. **Conservation et lieu de stockage des données :** Vérifiez si des obligations de conservation existantes s’appliquent aux tâches To Do et si les stratégies et conservations légales Purview couvrent le nouveau plan privé. Évaluez si le stockage dans la macro-région Europe est compatible avec vos exigences. Consultez les responsables de la conformité avant de supprimer ou réorganiser des données de tâches.

4. **Informer les utilisateurs :** Indiquez aux utilisateurs qu’aucune migration n’a lieu, combien de temps les données existantes restent visibles dans Planner et quelles fonctions changent (utilisation hors ligne, rappels, pièces jointes, groupes de listes).

5. **Suivre le Centre de messages :** Microsoft y annoncera les détails du déploiement, la fonction d’exportation et les dates concernant le service et les API.

## Sources

1.  [Moving from Microsoft To Do to Microsoft Planner](https://support.microsoft.com/en-us/planner/to-do-to-planner): annonce avec calendrier, différences entre To Do et Planner, étapes de préparation et FAQ.

2.  [Review data storage and compliance in Microsoft To Do](https://support.microsoft.com/en-us/todo/review-data-storage-and-compliance-in-microsoft-to-do): stockage des tâches To Do en tant que tâches dans la boîte aux lettres Exchange Online.

3.  [Frequently asked questions for admins about Microsoft Planner](https://learn.microsoft.com/en-us/planner/faq-for-planner-admins): lieux de stockage des tâches (Exchange, Azure, SharePoint, Dataverse), plan de service `PROJECTWORKMANAGEMENT` et accès via `planner.cloud.microsoft` sans licence.

4.  [Microsoft Planner for admins](https://learn.microsoft.com/en-us/planner/planner-for-admins): abonnements incluant Planner ainsi que Planner Plan 1 et Planner and Project Plan 3.

5.  [Other Microsoft 365 services data locations](https://learn.microsoft.com/en-us/microsoft-365/enterprise/m365-dr-service-other?view=o365-worldwide#planner): Planner sans engagement permanent concernant le lieu de stockage des données, lieu de stockage « EUR1 » pour les tenants dont le pays est la Suisse.

6.  [Data Residency for Exchange Online](https://learn.microsoft.com/en-us/microsoft-365/enterprise/m365-dr-service-exo?view=o365-worldwide): engagement des Product Terms pour le contenu des boîtes aux lettres au repos en Suisse.

7.  [Security, privacy, and compliance in Microsoft Planner](https://learn.microsoft.com/en-us/planner/planner-security-privacy-compliance): stockage des données de plan dans Azure ; Planner ne participe pas à Advanced Data Residency. État en août 2025, avant l’introduction des stratégies de rétention pour Planner.

8.  [Microsoft Planner support in Microsoft Purview](https://learn.microsoft.com/en-us/planner/planner-in-purview): eDiscovery et conservation légale uniquement pour les tâches dans les plans basés sur des groupes, liste des types de plans non pris en charge.

9.  [MC1384412: Retention policies for Microsoft Planner content](https://mc.merill.net/message/MC1384412): miroir du message du Centre de messages relatif aux stratégies de rétention et de suppression pour Planner, déploiement de juillet à août 2026.

10.  [Learn about retention for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange): éléments de boîte aux lettres pris en charge ; tâches uniquement avec une date de fin.

11.  [Use the Microsoft To Do API](https://learn.microsoft.com/en-us/graph/api/resources/todo-overview?view=graph-rest-1.0): ressources de l’API To Do dans Microsoft Graph (listes, tâches, listes de contrôle, ressources liées).

12.  [List lists](https://learn.microsoft.com/en-us/graph/api/todo-list-lists?view=graph-rest-1.0): point de terminaison, autorisations et propriétés `isShared`, `isOwner` et `wellknownListName`.

13.  [List Todo tasks](https://learn.microsoft.com/en-us/graph/api/todotasklist-list-tasks?view=graph-rest-1.0): point de terminaison pour les tâches d’une liste et autorisation d’application `Tasks.Read.All`.

14.  [Microsoft Will Retire its To Do App in June 2027](https://www.thurrott.com/cloud/344071/microsoft-will-retire-its-to-do-app-on-in-june-2027): article comprenant une déclaration ultérieure de Microsoft sur les intégrations dans Outlook et Teams.
