---
title: "Facilitator dans Microsoft Teams : ce que l’agent de réunion peut faire et ne peut pas faire"
navTitle: "Teams Facilitator"
description: "Le Facilitator est un agent Copilot qui rédige des notes communes dans les réunions Teams planifiées, gère l’ordre du jour, surveille le temps et répond aux questions de manière visible pour tous. Cet article explique les fonctions, les prérequis en matière de licences et d’administration, les scénarios d’utilisation pertinents ainsi que les limites liées aux types de réunions, à la langue et à la conformité."
date: "2026-11-27"
kategorie: "Microsoft Teams"
timeToRead: "8 min de lecture"
themen:
  - microsoft-teams
  - microsoft-365-exchange
produkte:
  - "teams"
protokolle:
  - "lizenzierung"
  - "powershell"
hauptthema: "microsoft-teams"
slug: "facilitator-dans-microsoft-teams-ce-que-l-agent-de-reunion-peut-faire-et-ne-peut-pas-faire"
translationId: "article-33919f700e0a6587"
translationOf: teams-facilitator
url: https://rafaelpfister.ch/fr/blog/facilitator-dans-microsoft-teams-ce-que-l-agent-de-reunion-peut-faire-et-ne-peut-pas-faire
translationSourceHash: 796c74d7d8547585c4342621dfa84e867cf70a2cbc477de7f8e82cd88cbc2d0c
translationModel: gpt-5.6-terra
translatedAt: 2026-11-27T09:39:53.040Z
translationReview: automatic
---

Depuis que Microsoft déploie Facilitator dans les réunions Teams, une nouvelle entrée apparaît dans les options de réunion de nombreux tenants : «Allow Copilot and Facilitator». Son nom ne précise pas ce qui se cache derrière. Facilitator est un agent d’IA qui participe à la réunion et travaille de manière visible pour tous. Cela détermine à la fois ses possibilités d’utilisation et ses limites.

## Ce qu’est Facilitator

Facilitator est un agent de Microsoft 365 Copilot qui assume le rôle d’assistant de modération dans une réunion Teams planifiée. Pendant la réunion, il rédige des notes sur une page Loop commune que tous les participants internes peuvent lire et modifier. Il lit l’ordre du jour dans l’invitation, affiche un minuteur d’ordre du jour sur la scène de réunion et réagit aux @mentions dans le chat de réunion.

La différence par rapport aux autres fonctions d’IA dans Teams réside dans la visibilité :

| Fonction | Qui voit le résultat | Moment |
|---|---|---|
| Copilot dans les réunions Teams | uniquement la personne qui pose la question | pendant et après la réunion |
| Intelligent Recap | participants disposant d’une licence Teams Premium ou Copilot | après la réunion |
| Facilitator | tous les participants internes | pendant la réunion, notes également après |

Une question posée à Copilot reste privée. Une question posée à Facilitator et sa réponse apparaissent dans le chat de réunion pour tous. Facilitator travaille donc collectivement pour tous les participants.

## Vue d’ensemble pour les utilisateurs

Pour toutes les fonctions, les conditions suivantes s’appliquent : la réunion est une réunion Teams planifiée de votre propre organisation, sans participants externes, l’option «Autoriser Copilot et Facilitator» est activée dans les options de réunion et la langue parlée a été sélectionnée au démarrage. Seule la personne qui démarre Facilitator ou lui donne des instructions a besoin d’une licence Copilot. Tous les autres participants internes voient les résultats sans licence Copilot propre.

| Fonction | Ce que vous pouvez faire | Qui peut l’utiliser | Autres prérequis |
|---|---|---|---|
| Démarrer Facilitator | Activer l’agent avant ou pendant la réunion | personnes disposant d’une licence Microsoft 365 Copilot | Loop activé dans Teams, application «Facilitator» non bloquée |
| Notes en direct | Les décisions, questions ouvertes et tâches sont consignées en continu dans l’onglet «Notes» ; vous pouvez les corriger et les compléter directement | lecture et modification : tous les participants internes | la transcription est active (elle démarre automatiquement) ; seules les paroles dans la langue sélectionnée sont prises en compte, pas les messages du chat |
| Ordre du jour et suivi | La liste des points de l’ordre du jour de l’invitation apparaît dans le chat, les points traités sont cochés | affichage : tous les participants internes | ordre du jour dans l’invitation ou transmis via @Facilitator dans le chat ; invitation non chiffrée ; au moins une autre personne que l’organisateur participe à la réunion |
| Minuteur et rappels | Compte à rebours pour un point, indication à mi-parcours, avant la fin et en cas de dépassement | demander le minuteur : licence Copilot ; voir : tous les participants internes | aucun |
| Questions sur la réunion | Poser une question via @Facilitator dans le chat, par exemple «Qu’avons-nous décidé pour le point 2 ?» ; question et réponse visibles pour tous | poser des questions : licence Copilot ; lire les réponses : tous les participants internes | questions après la fin de la réunion uniquement avec l’option «Pendant et après la réunion» |
| Questions avec recherche web | Recherche rapide pendant la réunion, par exemple sur une version de produit | licence Copilot | recherche web pour Copilot non désactivée dans le tenant |
| Créer un document Word | Générer un document à partir des notes, par exemple comme projet de procès-verbal | licence Copilot | stockage dans le OneDrive de la personne qui effectue la demande |
| Créer une présentation PowerPoint | Générer une présentation à partir de la réunion | licence Copilot | Public Preview : participation à Teams Public Preview |
| Transférer les tâches dans Planner | Transférer les tâches recensées vers Planner avec «Accept to sync» | licence Copilot | Public Preview : participation à Teams Public Preview |
| Notes lors d’une réunion en présentiel (smartphone) | Documenter une discussion sur place avec l’application Teams sur smartphone | licence Copilot | application Teams sur smartphone |
| Notes dans une Teams Room | Afficher le chat, les notes et le minuteur sur l’écran de la salle | licence Copilot uniquement pour l’organisateur | Teams Rooms on Windows ; pour les réunions en présentiel non planifiées, Teams Rooms Pro (Public Preview) |

## Fonctions et statut

Microsoft distingue les fonctions généralement disponibles de celles en Public Preview dans la documentation d’administration (état : novembre 2026) :

| Fonction | Statut |
|---|---|
| Notes en direct dans l’onglet «Notes» (page Loop, modifiable en commun) | généralement disponible |
| Répondre aux questions à partir du contenu de la réunion et via recherche web | généralement disponible |
| Minuteur d’ordre du jour avec indications à mi-parcours et avant la fin de la réunion | généralement disponible |
| Créer un document Word à partir de la réunion | généralement disponible |
| @mention de Facilitator dans le chat de réunion | généralement disponible |
| Notes sur smartphone pour les réunions en présentiel | généralement disponible |
| Teams Rooms : chat, notes et minuteur dans la salle | généralement disponible |
| Recenser les tâches et les synchroniser avec Planner | Public Preview |
| Créer une présentation PowerPoint à partir de la réunion | Public Preview |
| Notes pour les réunions en présentiel non planifiées dans Teams Rooms | Public Preview |

Concernant l’ordre du jour : si l’invitation contient une liste de points, Facilitator la reprend, la publie dans le chat et coche les points dès qu’ils ont été abordés. Le suivi s’effectue à intervalles, et non immédiatement. Si la réunion dépasse l’heure de fin prévue, il le signale. Sur demande, il peut également lancer un compte à rebours pour un point précis.

Concernant les notes : Facilitator résume les décisions, les questions ouvertes et les tâches. Il se base exclusivement sur les propos retranscrits. Ce qui figure uniquement dans le chat de réunion n’est pas intégré aux notes. Celles-ci peuvent être corrigées et complétées manuellement à tout moment ; Facilitator continue d’écrire sur la même page.

## Prérequis

Plusieurs conditions doivent être remplies simultanément pour l’utiliser :

- **Licence :** Toute personne qui ajoute Facilitator à une réunion ou l’active pendant celle-ci a besoin d’une licence Microsoft 365 Copilot, en plus d’une licence Microsoft 365 et Teams appropriée. Les participants sans licence Copilot ne peuvent pas poser de questions à Facilitator, mais voient les notes, les questions des autres et ses réponses.
- **Loop :** Les expériences Loop dans Teams doivent être activées dans le tenant. Sans Loop, il n’y a pas de notes.
- **Transcription :** L’activation de Facilitator démarre automatiquement la transcription. Si l’option de réunion «Allow Copilot and Facilitator» est définie sur «Only during the meeting», Facilitator fonctionne sans transcription enregistrée ; il n’est alors pas possible de poser des questions après la fin de la réunion.
- **Autorisation de l’application :** L’application «Facilitator» est autorisée par défaut, à moins que toutes les applications ne soient bloquées dans le Teams Admin Center.

Dans l’invitation, activez Facilitator sous **Options** > **Copilot and other AI** avec le paramètre «Allow Copilot and Facilitator». Vous pouvez également l’activer pendant la réunion via le menu Copilot.

## Facilitator est grisé

Dans le menu «IA de réunion», Teams affiche trois commutateurs : Copilot, Facilitator et Récapitulatif intelligent. Si les trois sont grisés et que le message «Vous n’avez pas accès à l’IA dans cette réunion» apparaît, la cause est la licence ou la réunion elle-même. Si seul Facilitator est grisé, la cause est l’application ou Loop.

| Symptôme | Cause possible | Vérification |
|---|---|---|
| Les trois commutateurs sont gris | Le compte n’a ni licence Microsoft 365 Copilot ni licence Teams Premium | Microsoft 365 Admin Center > Utilisateurs > Licences. Une licence nouvellement attribuée ne prend souvent effet dans Teams qu’après quelques heures ou une nouvelle connexion |
| Les trois commutateurs sont gris | Réunion instantanée («Meet now»), réunion démarrée depuis un chat, réunion de canal ou appel | Créer la réunion dans le calendrier en tant que réunion Teams planifiée |
| Les trois commutateurs sont gris | Réunion d’une autre organisation, participation en tant que personne externe ou invité | Facilitator n’est pas disponible dans les réunions externes |
| Les trois commutateurs sont gris | L’option de réunion «Autoriser Copilot et Facilitator» est définie sur «Désactivé», par l’organisateur, un modèle de réunion ou une étiquette de confidentialité | Options de réunion > Copilot et autres IA |
| Les trois commutateurs sont gris | La stratégie de réunion Teams bloque Copilot ou la transcription | Teams Admin Center > Réunions > Stratégies de réunion |
| Seul Facilitator est gris | L’application «Facilitator» est bloquée, par exemple parce que toutes les applications sont interdites | Teams Admin Center > Applications Teams > Gérer les applications |
| Seul Facilitator est gris ou aucune note | Loop est désactivé pour les notes de réunion | `Get-SPOTenant` affiche `IsCollabMeetingNotesFluidEnabled` |

La licence est la cause la plus fréquente. Sans licence Microsoft 365 Copilot, le commutateur Facilitator reste grisé dans chaque réunion, indépendamment des options de réunion et des stratégies.

## À quoi sert Facilitator

Facilitator est particulièrement utile dans les réunions où une personne devrait autrement rédiger le procès-verbal et ne peut donc pas participer pleinement à la discussion. Exemples typiques :

- **Réunions de projet et de suivi avec un ordre du jour fixe :** le minuteur d’ordre du jour et les indications à mi-parcours permettent de respecter le calendrier, tandis que les notes consignent les décisions et les points ouverts sans procès-verbal séparé.
- **Ateliers avec décisions :** la page Loop commune est visible pendant la réunion. Les participants corrigent directement les formulations, au lieu de valider ultérieurement un procès-verbal par e-mail.
- **Questions pendant la réunion :** Facilitator répond dans le chat, de manière visible pour tous, à des questions comme «Qu’avons-nous décidé pour le point 2 ?» ou à une courte recherche web sur une version de produit.
- **Résumé ultérieur :** sur demande, Facilitator crée un document Word à partir des notes, qui est enregistré dans le OneDrive de la personne qui effectue la demande.
- **Réunions en présentiel :** l’application Teams sur smartphone ou une Teams Room permettent également de documenter les réunions sur place.

## Ce que Facilitator ne peut pas faire

Les limites sont clairement décrites dans la documentation Microsoft et concernent principalement les types de réunions, la langue et la conformité.

**Types de réunions non pris en charge.** Facilitator fonctionne uniquement dans les réunions Teams planifiées. Les réunions instantanées («Meet now»), les réunions démarrées depuis un chat, les appels Teams, les réunions de canal ainsi que les conversations 1:1 et de groupe ne sont pas pris en charge. La documentation ne mentionne aucune prise en charge des webinaires ni des assemblées publiques.

**Pas de participants externes.** Les notes générées par IA ne sont pas disponibles dans les réunions et les chats comportant des personnes externes. Les participants externes ne voient généralement pas les notes. Facilitator ne peut donc pas être utilisé comme outil de procès-verbal pour les réunions avec des clients ou des fournisseurs.

**Une langue par réunion.** La langue parlée est sélectionnée lors de l’activation. Les notes ne sont générées que pour les passages dans cette langue. Dans une réunion qui alterne entre l’allemand et le français, les parties en français sont absentes. Si la langue choisie diffère de celle qui est parlée, aucune note n’est créée. Le suisse allemand n’est pas répertorié comme langue distincte dans la liste des langues Copilot prises en charge.

**Pas d’enregistrement.** Facilitator active la transcription, mais pas l’enregistrement. Les personnes qui ont besoin de l’image et du son doivent démarrer l’enregistrement séparément.

**Pas d’ordre du jour à partir d’invitations protégées.** Si l’invitation est chiffrée à l’aide d’une étiquette de confidentialité, Facilitator ne peut pas lire l’ordre du jour. Il en va de même tant que seule la personne organisatrice est présente dans la réunion, car il ne lit pas sa copie de l’invitation.

**Ne remplace pas un procès-verbal formel.** Les notes sont un résumé généré par IA à partir de la transcription. Pour les séances du conseil d’administration, les entretiens RH ou d’autres réunions ayant une portée juridique, un procès-verbal vérifié et approuvé reste nécessaire.

## Stockage des données et conformité

Les notes sont stockées sous forme de fichier `.loop` dans le dossier «Meetings» du OneDrive de la personne qui a démarré Facilitator. Microsoft les traite comme des données de transcription. Trois limitations sont pertinentes pour les responsables de la conformité :

- **eDiscovery :** les notes de réunion générées par IA ne sont pas automatiquement recensées comme pièces jointes cloud dans Microsoft Purview eDiscovery.
- **Conservation :** les étiquettes de conservation ne sont pas prises en charge pour les pièces jointes cloud dans les notes générées par IA.
- **Étiquettes de confidentialité :** les notes ne reprennent pas l’étiquette de la réunion. Une étiquette peut être appliquée ultérieurement au composant Loop. Des paramètres tels que «Prevent copy and paste» et les filigranes ne s’appliquent pas aux réponses et aux notes de Facilitator.

Pour les organisations soumises à des obligations de conservation, cela signifie que les notes devant être considérées comme des documents commerciaux doivent être transférées vers un emplacement de stockage où la conservation fonctionne. Microsoft décrit les aspects Purview sur une page distincte (voir Sources).

## Gérer et désactiver Facilitator

Dans le Teams Admin Center, recherchez l’application sous **Teams apps** > **Manage apps** en recherchant «Facilitator». Vous pouvez l’y autoriser ou la bloquer pour l’ensemble du tenant. Pour certains groupes d’utilisateurs, gérez l’application via les stratégies d’autorisation des applications ou la gestion centrée sur les applications.

Les notes Loop dans les réunions, sur lesquelles Facilitator s’appuie, dépendent d’un paramètre SharePoint au niveau du tenant. Celui-ci s’applique à tous les utilisateurs et ne peut pas être défini par personne :

```powershell
Connect-SPOService -Url "https://contoso-admin.sharepoint.com"
Set-SPOTenant -IsCollabMeetingNotesFluidEnabled $false
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-Url` | Adresse du centre d’administration SharePoint du tenant (`<tenant>-admin.sharepoint.com`) |
| `-IsCollabMeetingNotesFluidEnabled` | `$false` désactive les notes de réunion communes et donc les notes IA de Facilitator pour tous les utilisateurs, `$true` les réactive |

</details>

Ce paramètre supprime également la fonction de notes pour les réunions sans Facilitator. Si seul l’agent doit être bloqué, le blocage de l’application dans le Teams Admin Center est la solution la plus ciblée. Pour les réunions sans transcription enregistrée, définissez l’option Copilot et Facilitator sur «Only during the meeting» dans la stratégie de réunion.

## Sources

1.  [Set up Facilitator in Microsoft Teams for collaborative AI-generated notes](https://learn.microsoft.com/en-us/microsoftteams/facilitator-teams): Documentation d’administration avec les exigences de licence, le statut des fonctionnalités, le stockage des données, les limitations de conformité et la gestion de l’application (état au 18.11.2026).

2.  [Facilitator in Microsoft Teams meetings](https://support.microsoft.com/en-us/teams/copilot/facilitator-in-microsoft-teams-meetings): Guide utilisateur sur les options de réunion, l’ordre du jour, le minuteur, les types de réunions pris en charge et le choix de la langue.

3.  [Facilitator in Microsoft Teams Rooms](https://learn.microsoft.com/en-us/microsoftteams/rooms/facilitator-teams-rooms): Réunions en présentiel dans Teams Rooms on Windows, licence de la personne organisatrice, attribution des intervenants dans la salle.

4.  [Keep meetings focused and productive with Facilitator in Teams](https://techcommunity.microsoft.com/blog/microsoft365insiderblog/keep-meetings-focused-and-productive-with-facilitator-in-teams/4456029): Blog Microsoft 365 Insider d’octobre 2025 sur la transcription, la base des notes et le fonctionnement sans transcription enregistrée.

5.  [Microsoft Purview: Teams Facilitator](https://learn.microsoft.com/en-us/purview/ai-teams-facilitator): Vue Purview des données Facilitator pour la conservation et eDiscovery.

6.  [Manage Loop components in OneDrive and SharePoint](https://learn.microsoft.com/microsoft-365/loop/loop-components-configuration): Paramètres de tenant `IsLoopEnabled` et `IsCollabMeetingNotesFluidEnabled` pour les composants Loop et les notes de réunion.
