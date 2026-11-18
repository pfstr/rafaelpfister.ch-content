---
title: "Après la panne d’Exchange Online : demander un avoir SLA à Microsoft"
navTitle: "Avoir SLA"
description: "La panne d’Exchange Online des 12 et 13 novembre 2026 a duré environ 22 heures, ce qui place de nombreux tenants sous la disponibilité garantie de 99,9 %. Microsoft n’accorde pas l’avoir automatiquement : il doit être demandé avant le 31 décembre 2026. Comment calculer la disponibilité, quels justificatifs Microsoft exige et pourquoi les petites entreprises doivent passer par leur partenaire tandis que les grandes doivent s’adresser directement à Microsoft."
date: "2026-11-18"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min de lecture"
themen:
  - microsoft-365-exchange
produkte:
  - "exchange-online"
protokolle:
  - "lizenzierung"
  - "powershell"
slug: "apres-la-panne-d-exchange-online-demander-un-avoir-sla-a-microsoft"
translationId: "article-0e96d35615b0d8d9"
translationOf: microsoft-sla-gutschrift-einfordern
url: https://rafaelpfister.ch/fr/blog/apres-la-panne-d-exchange-online-demander-un-avoir-sla-a-microsoft
translationSourceHash: 537caf78f05fd364041d0a3155f020be11b80f4c936d732e61f0a01f443853c1
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T09:21:12.076Z
translationReview: automatic
---

# Après la panne d’Exchange Online : demander un avoir SLA à Microsoft

Du 12 novembre 2026 à 18 h 54 au 13 novembre 2026 à 17 h 10 (heure suisse), de nombreuses organisations n’ont pas pu accéder à leurs boîtes aux lettres Exchange Online, ou seulement de manière intermittente. Microsoft a répertorié la perturbation dans le Centre d’administration sous EX1550543 (Exchange Online) et MO1550821 (autres services Microsoft 365). J’ai documenté son déroulement dans le [live ticker sur la panne](https://rafaelpfister.ch/liveticker/exchange-online-ausfall-2026-11). Pour Exchange Online, Microsoft garantit une disponibilité mensuelle de 99,9 %. Une panne de plus de 22 heures est nettement inférieure à ce seuil. Vous ne recevez toutefois un avoir que si vous le demandez, et la procédure dépend de la manière dont votre organisation a acheté Microsoft 365.

## Ce que garantit le SLA

Le document de référence est le « Service Level Agreement for Microsoft Online Services », que Microsoft publie à nouveau chaque mois. Cet article se fonde sur la version du 17 novembre 2026. Pour Exchange Online, le SLA définit une indisponibilité comme toute période durant laquelle les utilisateurs ne peuvent pas envoyer ou recevoir d’e-mails avec Outlook sur le web. Il n’existe pas de fenêtres de maintenance planifiées pour Exchange Online.

La disponibilité est calculée en minutes-utilisateur :

```text
Verfügbarkeit (%) = (Benutzerminuten - Ausfallminuten) / Benutzerminuten x 100
```

Les minutes-utilisateur correspondent au nombre de minutes du mois multiplié par le nombre d’utilisateurs. Les minutes d’indisponibilité correspondent à la durée de chaque incident multipliée par le nombre d’utilisateurs concernés. Si la disponibilité passe sous un seuil, un avoir est accordé sur les frais mensuels du service concerné :

| Disponibilité mensuelle | Avoir |
|---|---|
| inférieure à 99,9 % | 25 % |
| inférieure à 99 % | 50 % |
| inférieure à 95 % | 100 % |

Pour Microsoft Teams, SharePoint Online et OneDrive, les mêmes seuils s’appliquent, avec leurs propres définitions d’indisponibilité. Pour Teams, cela ne compte que si les utilisateurs ne peuvent pas discuter ou démarrer des réunions. Durant EX1550543, les discussions et les réunions ont largement fonctionné selon Microsoft ; les calendriers et les invitations étaient perturbés. Une demande pour Teams a donc peu de chances d’aboutir.

Outre la disponibilité des boîtes aux lettres, Exchange Online comporte d’autres niveaux de service, par exemple le délai de remise au sein de Microsoft 365 et l’acceptation des e-mails entrants. Si un incident en touche plusieurs, vous devez, selon le SLA, en choisir un. Pour la panne de novembre, la disponibilité des boîtes aux lettres via Outlook sur le web est le critère le plus évident.

## Ce que représente la panne en pourcentage

Novembre 2026 compte 30 jours, soit 43'200 minutes par utilisateur. Cela donne les limites suivantes par utilisateur :

| Seuil | Durée d’indisponibilité admissible en novembre |
|---|---|
| 99,9 % | 43 minutes |
| 99 % | 7 heures 12 minutes |
| 95 % | 36 heures |

La perturbation a duré 1'336 minutes, de la première annonce jusqu’à sa résolution. Voici deux exemples de calcul pour une organisation comptant 200 boîtes aux lettres :

| Cas | Minutes d’indisponibilité | Disponibilité | Avoir |
|---|---|---|---|
| Les 200 utilisateurs sans accès pendant toute la perturbation | 200 x 1'336 = 267'200 | 96,91 % | 50 % |
| 120 utilisateurs sans accès pendant 10 heures | 120 x 600 = 72'000 | 99,17 % | 25 % |

Le nombre total de minutes-utilisateur est, dans les deux cas, de 200 x 43'200 = 8'640'000. Ce qui compte est le nombre d’utilisateurs réellement concernés et la durée pendant laquelle ils l’ont été. Microsoft le vérifie à l’aide de sa propre télémétrie, mais exige vos indications à ce sujet dans la demande. De nombreux tenants en Europe n’ont été touchés qu’à partir de la fin de soirée du 12 novembre, ou de manière intermittente. Calculez donc avec les périodes que vous pouvez justifier, et non avec la durée totale du message Microsoft.

## Ce qu’est l’avoir, et ce qu’il n’est pas

Le SLA règle plusieurs points qui devraient être clairs avant la demande :

1.  **Uniquement sur demande.** Microsoft ne vérifie ni n’accorde d’avoirs de sa propre initiative. Selon le SLA, un message d’état de Microsoft ne constitue expressément pas une confirmation qu’un niveau de service n’a pas été atteint.

2.  **Avoir, pas remboursement.** L’avoir est déduit d’une facture future. Il est limité aux frais mensuels du service concerné.

3.  **Recours exclusif.** Selon le SLA, les avoirs constituent le seul recours en cas de problèmes de disponibilité. Microsoft ne rembourse pas les pertes de chiffre d’affaires, les heures supplémentaires ni d’autres coûts indirects. Vous ne pouvez pas réduire vous-même les factures.

4.  **Uniquement le service concerné.** Si vous avez une suite telle que Microsoft 365 E3 ou Business Premium, l’avoir porte sur la part d’Exchange Online, et non sur le prix total de la licence. Microsoft détermine cette part lors de l’examen ; le SLA ne fournit aucune formule à cet effet.

5.  **Pas d’avoir pour les offres gratuites et d’essai.** Les versions d’essai, les préversions et les services gratuits sont exclus, de même que les licences données aux organisations à but non lucratif. Les clients particuliers disposant de Microsoft 365 Personal ou Family ne sont pas couverts par ce SLA.

6.  **Contrats Open en jours.** Les titulaires de licences via Open, Open Value ou Open Value Subscription reçoivent un éventuel avoir sous la forme de jours d’utilisation supplémentaires plutôt que d’un montant.

7.  **Délai.** Pour les services Microsoft 365, la demande accompagnée de toutes les informations doit parvenir avant la fin du mois suivant le mois de l’incident. Pour la panne de novembre, la date limite est le **31 décembre 2026**. Azure applique son propre délai de 60 jours.

## Les informations exigées par Microsoft

Le SLA mentionne cinq informations obligatoires. Si elles manquent, Microsoft rejette la demande :

1.  une description précise de l’incident,
2.  le moment et la durée de l’indisponibilité,
3.  les ressources concernées (pour Exchange Online : tenant et service),
4.  le nombre et les emplacements des utilisateurs concernés,
5.  une description des erreurs survenues.

L’identifiant de l’incident (EX1550543) et l’ID du tenant doivent toujours être inclus. Les deux peuvent être interrogés avec Microsoft Graph PowerShell :

```powershell
Connect-MgGraph -Scopes "ServiceHealth.Read.All", "Organization.Read.All"
(Get-MgOrganization).Id
Get-MgServiceAnnouncementIssue -ServiceHealthIssueId "EX1550543" |
    Select-Object Id, Title, StartDateTime, EndDateTime, Classification
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-Scopes "ServiceHealth.Read.All"` | autorisation de lire les messages d’intégrité des services |
| `-Scopes "Organization.Read.All"` | autorisation de lire les données de l’organisation, dont l’ID du tenant |
| `(Get-MgOrganization).Id` | affiche l’ID du tenant |
| `-ServiceHealthIssueId "EX1550543"` | récupère précisément cet incident |
| `Select-Object …` | affiche l’identifiant, le titre, le début, la fin et la classification de l’incident |

</details>

La classification (`Classification`) indique si Microsoft traite l’incident comme `incident` ou comme `advisory`. Selon la documentation partenaire de Microsoft, les avis de niveau « Advisory » ne donnent en règle générale pas droit à un avoir. EX1550543 était classé comme Incident.

Pour le nombre et l’emplacement des utilisateurs, une analyse des boîtes aux lettres sous licence par emplacement d’utilisation suffit :

```powershell
Connect-MgGraph -Scopes "User.Read.All"
$abfrage = @{
    All              = $true
    Filter           = 'assignedLicenses/$count ne 0'
    ConsistencyLevel = 'eventual'
    CountVariable    = 'anzahl'
    Property         = 'UsageLocation'
}
Get-MgUser @abfrage |
    Group-Object UsageLocation |
    Select-Object Name, Count
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `$abfrage = @{ … }` | collecte les paramètres pour `Get-MgUser` dans une table de hachage (splatting), afin que la ligne reste courte |
| `All` | lit tous les utilisateurs, et pas seulement la première page |
| `Filter 'assignedLicenses/$count ne 0'` | uniquement les utilisateurs disposant d’au moins une licence attribuée |
| `ConsistencyLevel eventual` | requis pour le filtre `$count` (requête avancée) |
| `CountVariable anzahl` | enregistre le nombre total dans la variable `$anzahl` |
| `Property UsageLocation` | charge également l’emplacement d’utilisation |
| `Group-Object UsageLocation` | compte les utilisateurs par pays |

</details>

Ce nombre constitue la limite supérieure. Vous démontrez le nombre d’utilisateurs qui n’ont réellement pas eu accès à l’aide de vos propres données : tickets du helpdesk horodatés, messages d’erreur d’Outlook et d’Outlook sur le web (captures d’écran), résultats des tests de disponibilité de votre monitoring ainsi que journaux de connexion et d’accès. Plus ces informations sont précises, moins le résultat dépend uniquement de la télémétrie de Microsoft.

## Petites entreprises : généralement via le partenaire

En Suisse, les petites et moyennes entreprises achètent généralement Microsoft 365 via un partenaire du Cloud Solution Provider Program (CSP) ou directement par carte de crédit dans le Centre d’administration. Vous voyez quelle voie s’applique dans le Centre d’administration Microsoft 365, sous **Facturation > Vos produits** : si un partenaire y est indiqué comme source d’approvisionnement, il s’agit d’un contrat CSP.

### Achat via un partenaire CSP

Dans ce cas, Microsoft n’accepte pas votre demande directement. Selon le Partner Center, seuls les fournisseurs directs CSP et les fournisseurs indirects (distributeurs) peuvent demander des avoirs, pas les revendeurs indirects. Si votre prestataire informatique est un revendeur indirect, la demande passe donc par lui vers son distributeur, puis vers Microsoft. Chaque niveau applique ses propres délais, souvent plus courts. Contactez-les donc maintenant, et non à la fin décembre.

Votre partenaire a besoin de votre part :

1.  l’ID du tenant,
2.  l’identifiant de l’incident (EX1550543),
3.  la période, le nombre et les emplacements des utilisateurs concernés, ainsi que les erreurs observées,
4.  un e-mail dans lequel vous demandez l’avoir SLA. Il doit provenir d’une adresse du domaine concerné ; Microsoft n’accepte pas les e-mails provenant d’une adresse privée.

Microsoft connaît l’incident lui-même. La demande doit démontrer que votre organisation a été touchée : selon Microsoft, le nom, l’ID du tenant et le numéro de ticket ne suffisent pas à eux seuls. Si l’avoir est accordé, Microsoft le crédite au partenaire, qui vous le transmet ensuite. Les modalités de cette transmission sont régies par votre contrat avec le partenaire. Demandez des précisions si rien n’apparaît sur la facture suivante.

### Achat direct auprès de Microsoft

Si vous avez payé Microsoft 365 directement auprès de Microsoft, vous soumettez vous-même la demande dans le Centre d’administration. Un rôle d’administrateur est nécessaire :

1.  Sous `admin.cloud.microsoft` , ouvrez **Aide et support** en bas à droite.
2.  Décrivez la demande, par exemple « Request an SLA service credit for a Microsoft 365 outage », puis choisissez **Contacter le support**.
3.  Choisissez l’e-mail comme méthode de contact. Le titre doit contenir « SLA service credit request » et l’identifiant EX1550543.
4.  Dans la description, indiquez les cinq informations obligatoires de la section précédente et joignez les justificatifs.

Une demande distincte est nécessaire pour chaque service et chaque incident. Selon le SLA, Microsoft traite généralement les demandes dans un délai de 45 jours.

### Est-ce intéressant pour les petites entreprises ?

Pour dix utilisateurs, l’avoir est faible : dans le cas d’une suite, seule la part d’Exchange est prise en compte, dont 25 ou 50 % pour un mois. L’effort est également faible si vous rassemblez maintenant les informations. La demande présente un second avantage : elle documente auprès du partenaire et de la direction dans quelle mesure votre organisation a été affectée. Cela sera utile ultérieurement pour déterminer si une solution de secours pour les e-mails est nécessaire.

## Grandes entreprises : directement auprès de Microsoft, avec leurs propres données de mesure

Les grandes organisations obtiennent généralement leurs licences Microsoft 365 via un Enterprise Agreement (EA) ou le Microsoft Customer Agreement for Enterprise (MCA-E). La demande est alors adressée directement à Microsoft, soit par le Centre d’administration comme décrit ci-dessus, soit par le contrat Unified Support. Si vous avez un Customer Success Account Manager (CSAM), informez-le de la demande ; il peut suivre son traitement. Dans le cas d’un EA via un Licensing Solution Partner (LSP), clarifiez également sur quelle facture l’avoir apparaîtra.

Avec plusieurs milliers d’utilisateurs, les montants sont significatifs et Microsoft vérifie donc les informations avec une attention correspondante. La différence avec un petit tenant réside dans l’établissement des preuves :

1.  **Déterminer précisément les utilisateurs concernés.** Les grands tenants sont répartis sur plusieurs bases de données et régions dans les centres de données de Microsoft ; toutes les boîtes aux lettres n’ont pas été perturbées durant la même période. Déclarer forfaitairement tous les utilisateurs comme concernés est contestable pour les grands tenants. Les données de mesure sont plus solides : tests synthétiques contre Outlook sur le web par emplacement, données de disponibilité de votre propre monitoring, analyses des tickets de helpdesk par emplacement et par période.

2.  **Indiquer les emplacements.** Le SLA exige les emplacements des utilisateurs concernés. Pour les organisations internationales, une répartition par pays est utile, car la perturbation a d’abord touché l’Amérique du Nord et l’Amérique du Sud, puis les autres régions plus tard.

3.  **Délimiter proprement l’hybride.** Le SLA exclut les retards dans votre propre centre de données, par exemple les e-mails dans la file d’attente des serveurs Exchange locaux vers le tenant. Démontrez que l’erreur se situait au sein de Microsoft 365. Les files d’attente des serveurs locaux constituent une bonne preuve, car elles montrent qu’Exchange Online n’a pas accepté les e-mails, ou seulement avec retard.

4.  **Choisir le niveau de service.** Si, chez vous, les délais de remise ont été principalement touchés et que l’accès aux boîtes aux lettres l’a été dans une moindre mesure, le niveau de service relatif au délai de remise peut constituer une meilleure base. Une seule demande est autorisée par incident et par service ; ce choix doit donc être fait avant la demande.

5.  **Impliquer les achats et la gestion des licences.** L’avoir apparaît sur la facture, et non dans le centre de coûts informatique. Convenez avec les achats qui soumet la demande, qui contrôle l’avoir et comment il est comptabilisé.

Les grandes organisations devraient en outre préparer la procédure pour la prochaine panne : un modèle incluant les cinq informations obligatoires, un monitoring qui mesure la disponibilité d’Outlook sur le web par emplacement, et une responsabilité clairement définie pour les demandes SLA.

## Modèle de demande

Le modèle suivant contient les informations obligatoires prévues par le SLA. Microsoft traite le plus rapidement les demandes rédigées en anglais :

```text
Subject: SLA service credit request: EX1550543, Exchange Online

Tenant ID: <Tenant-ID>
Service: Exchange Online
Incident ID: EX1550543 (related: MO1550821)
Service level: Exchange Online availability (Outlook on the web)

Description of the incident:
Users were unable to send or receive email using Outlook on the web,
Outlook for Windows and mobile clients.

Time and duration of downtime:
From 2026-11-12 <hh:mm> UTC to 2026-11-13 <hh:mm> UTC (<n> minutes).

Affected users:
<n> of <total> licensed users, locations: <Land: Anzahl, ...>

Errors observed:
<Fehlermeldungen, z. B. Verbindungsfehler in Outlook, HTTP 5xx in Outlook on the web>

Uptime calculation (November 2026):
(<total> x 43,200 - <Ausfallminuten>) / (<total> x 43,200) x 100 = <xx.xx> %

Attachments: helpdesk ticket export, monitoring data, screenshots.
```

## Sources

1.  [Service Level Agreement for Microsoft Online Services (aperçu des versions)](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services) : version du 17 novembre 2026, définition de l’indisponibilité pour Exchange Online, niveaux d’avoir, informations obligatoires, délais et exclusions.

2.  [Microsoft Learn: How and when to request a credit from Microsoft](https://learn.microsoft.com/en-us/partner-center/billing/request-credit) : procédure de demande pour les partenaires CSP, exclusion des revendeurs indirects, justificatifs nécessaires, classification des Advisories.

3.  [Microsoft Learn: Get-MgServiceAnnouncementIssue](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.devices.serviceannouncement/get-mgserviceannouncementissue) : cmdlet permettant de récupérer un incident d’intégrité de service via Microsoft Graph.

4.  [Microsoft 365 Status (status.cloud.microsoft)](https://status.cloud.microsoft/) : messages d’état publics relatifs à EX1550543 et MO1550821.

5.  [Windows Latest: Microsoft 365 outage, you may get money off your bill](https://www.windowslatest.com/2026/11/13/microsoft-365-outage-you-may-get-money-off-your-bill-if-you-use-it-for-work-but-youll-have-to-ask-microsoft-for-it-as-a-server-failure-takes-down-outlook-teams-and-exchange/) : procédure de demande dans le Centre d’administration pour la panne de novembre, remarque concernant les clients particuliers.

6.  [Office Watch: Microsoft 365 went down? How to claim your SLA credit](https://office-watch.com/?p=78178) : exclusion des licences privées, gratuites et destinées aux organisations à but non lucratif, une demande par service et par incident.

7.  [Live ticker : Exchange Online, panne mondiale d’accès aux boîtes aux lettres](https://rafaelpfister.ch/liveticker/exchange-online-ausfall-2026-11) : déroulement de la perturbation des 12 et 13 novembre 2026 avec la chronologie de Microsoft.
