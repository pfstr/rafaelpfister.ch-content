---
title: "Après la panne d’Exchange Online : demander un avoir SLA à Microsoft"
navTitle: "Avoir SLA"
description: "La panne d’Exchange Online des 12 et 13 novembre 2026 a duré environ 22 heures et a donc placé de nombreux tenants sous la disponibilité garantie de 99,9 %. Microsoft ne crédite pas automatiquement l’avoir : il doit être demandé avant le 31 décembre 2026. Comment calculer la disponibilité, quels justificatifs Microsoft exige et pourquoi les petites entreprises passent par leur partenaire tandis que les grandes s’adressent directement à Microsoft."
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
translationSourceHash: e2a540fa8b0a15956e2a9f4af6b604f517d4f077ec2d566bd2dd187a1d423216
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T11:21:03.262Z
translationReview: automatic
url: https://rafaelpfister.ch/fr/blog/apres-la-panne-d-exchange-online-demander-un-avoir-sla-a-microsoft
---

# Après la panne d’Exchange Online : demander un avoir SLA à Microsoft

<div class="notfall-hinweis notfall-hinweis--gross">
<p class="notfall-hinweis__titel">Assistance gratuite pour la demande SLA</p>
<p>Nous vous aidons gratuitement à demander à Microsoft ou à votre partenaire l’avoir lié à la panne d’Exchange Online des 12 et 13 novembre 2026 : calcul de la disponibilité, préparation des justificatifs, dépôt de la demande. Le délai court jusqu’au 31 décembre 2026. Contactez-nous via le <a href="https://rafaelpfister.ch/kontakt">formulaire de contact</a>.</p>
</div>

Du 12 novembre 2026 à 18 h 54 au 13 novembre 2026 à 17 h 10 (heure suisse), de nombreuses organisations n’ont eu aucun accès ou seulement un accès intermittent à leurs boîtes aux lettres Exchange Online. Microsoft a répertorié l’incident dans l’Admin Center sous EX1550543 (Exchange Online) et MO1550821 (autres services Microsoft 365). J’ai documenté son déroulement dans le [suivi en direct de la panne](https://rafaelpfister.ch/liveticker/exchange-online-ausfall-2026-11). Pour Exchange Online, Microsoft garantit une disponibilité mensuelle de 99,9 %. Une panne de plus de 22 heures est nettement inférieure à ce niveau. Vous ne recevrez toutefois un avoir que si vous en faites la demande, et la procédure dépend de la manière dont votre organisation a acheté Microsoft 365.

## Ce que garantit le SLA

Le document déterminant est le « Service Level Agreement for Microsoft Online Services », que Microsoft publie à nouveau chaque mois. Cet article s’appuie sur la version du 17 novembre 2026. Pour Exchange Online, le SLA définit une indisponibilité comme toute période durant laquelle les utilisateurs ne peuvent pas envoyer ni recevoir d’e-mails avec Outlook sur le web. Il n’existe pas de fenêtres de maintenance planifiées pour Exchange Online.

La disponibilité est calculée en minutes-utilisateurs :

```text
Verfügbarkeit (%) = (Benutzerminuten - Ausfallminuten) / Benutzerminuten x 100
```

Les minutes-utilisateurs correspondent au nombre de minutes du mois multiplié par le nombre d’utilisateurs. Les minutes d’indisponibilité correspondent à la durée de chaque incident multipliée par le nombre d’utilisateurs concernés. Si la disponibilité passe sous un seuil, un avoir est accordé sur les frais mensuels du service concerné :

| Disponibilité mensuelle | Avoir |
|---|---|
| moins de 99,9 % | 25 % |
| moins de 99 % | 50 % |
| moins de 95 % | 100 % |

Les mêmes seuils s’appliquent à Microsoft Teams, SharePoint Online et OneDrive, avec leurs propres définitions de l’indisponibilité. Pour Teams, seuls les cas où les utilisateurs ne peuvent pas discuter ou démarrer des réunions comptent. Pendant EX1550543, les conversations et les réunions ont, selon Microsoft, majoritairement fonctionné ; les calendriers et les invitations étaient perturbés. Une demande pour Teams a donc peu de chances d’aboutir.

Outre la disponibilité des boîtes aux lettres, Exchange Online propose d’autres niveaux de service, par exemple le délai de remise au sein de Microsoft 365 et l’acceptation des e-mails entrants. Si un incident en touche plusieurs, vous devez, selon le SLA, en choisir un seul. Pour la panne de novembre, la disponibilité des boîtes aux lettres via Outlook sur le web est la mesure la plus pertinente.

## Ce que représente la panne en pourcentage

Le mois de novembre 2026 compte 30 jours, soit 43'200 minutes par utilisateur. Il en résulte les seuils suivants par utilisateur :

| Seuil | Durée d’indisponibilité autorisée en novembre |
|---|---|
| 99,9 % | 43 minutes |
| 99 % | 7 heures 12 minutes |
| 95 % | 36 heures |

L’incident a duré 1'336 minutes entre le premier signalement et sa résolution. Deux exemples de calcul pour une organisation disposant de 200 boîtes aux lettres :

| Cas | Minutes d’indisponibilité | Disponibilité | Avoir |
|---|---|---|---|
| Les 200 utilisateurs sans accès pendant toute la panne | 200 x 1'336 = 267'200 | 96,91 % | 50 % |
| 120 utilisateurs sans accès pendant 10 heures | 120 x 600 = 72'000 | 99,17 % | 25 % |

Le nombre total de minutes-utilisateurs s’élève dans les deux cas à 200 x 43'200 = 8'640'000. L’élément déterminant est le nombre d’utilisateurs réellement affectés et la durée pendant laquelle ils l’ont été. Microsoft le vérifie au moyen de sa propre télémétrie, mais exige que vous indiquiez ces données dans la demande. De nombreux tenants en Europe n’ont été touchés que tard dans la soirée du 12 novembre ou de manière intermittente. Calculez donc avec les périodes que vous pouvez justifier, et non avec la durée totale du message de Microsoft.

## Ce qu’est l’avoir, et ce qu’il n’est pas

Le SLA règle plusieurs points qui doivent être clairs avant la demande :

1.  **Uniquement sur demande.** Microsoft n’examine ni n’accorde les avoirs de sa propre initiative. Selon le SLA, un message d’état de Microsoft ne vaut explicitement pas confirmation qu’un niveau de service n’a pas été atteint.

2.  **Avoir, pas remboursement.** L’avoir est déduit d’une facture ultérieure. Il est limité aux frais mensuels du service concerné.

3.  **Recours exclusif.** Selon le SLA, les avoirs constituent le seul recours en cas de problèmes de disponibilité. Microsoft ne rembourse pas les pertes de chiffre d’affaires, les heures supplémentaires ni d’autres coûts consécutifs. Vous ne pouvez pas réduire les factures de votre propre chef.

4.  **Uniquement le service concerné.** Si vous disposez d’une suite telle que Microsoft 365 E3 ou Business Premium, l’avoir porte sur la part d’Exchange Online, et non sur l’ensemble du prix de la licence. Microsoft détermine cette part lors de l’examen ; le SLA ne fournit aucune formule à cet effet.

5.  **Pas d’avoir pour les offres gratuites et d’essai.** Les versions d’essai, les aperçus et les services gratuits sont exclus, de même que les licences offertes aux organisations à but non lucratif. Les clients particuliers avec Microsoft 365 Personal ou Family ne relèvent pas de ce SLA.

6.  **Contrats Open en jours.** Les personnes ayant acquis leurs licences via Open, Open Value ou Open Value Subscription reçoivent un éventuel avoir sous forme de jours d’utilisation supplémentaires plutôt que d’une somme d’argent.

7.  **Délai.** Pour les services Microsoft 365, la demande avec toutes les informations doit parvenir avant la fin du mois suivant celui de l’incident. Pour la panne de novembre, la date limite est le **31 décembre 2026**. Azure est soumis à un délai distinct de 60 jours.

## Les informations exigées par Microsoft

Le SLA énumère cinq informations obligatoires. Si elles manquent, Microsoft rejette la demande :

1.  une description précise de l’incident,
2.  le moment et la durée de l’indisponibilité,
3.  les ressources concernées (pour Exchange Online : tenant et service),
4.  le nombre et les emplacements des utilisateurs concernés,
5.  une description des erreurs survenues.

L’identifiant de l’incident (EX1550543) et l’ID du tenant doivent toujours être inclus. Les deux peuvent être obtenus avec Microsoft Graph PowerShell :

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
| `-Scopes "ServiceHealth.Read.All"` | Autorisation de lire les messages d’intégrité du service |
| `-Scopes "Organization.Read.All"` | Autorisation de lire les données de l’organisation, y compris l’ID du tenant |
| `(Get-MgOrganization).Id` | affiche l’ID du tenant |
| `-ServiceHealthIssueId "EX1550543"` | récupère précisément cet incident |
| `Select-Object …` | affiche l’identifiant, le titre, le début, la fin et la classification de l’incident |

</details>

La classification (`Classification`) indique si Microsoft considère l’incident comme un `incident` ou comme un `advisory`. Selon la documentation partenaire de Microsoft, les avis de niveau « Advisory » ne donnent généralement pas droit à un avoir. EX1550543 était classé comme Incident.

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
| `$abfrage = @{ … }` | collecte les paramètres de `Get-MgUser` dans une table de hachage (splatting) afin de garder la ligne courte |
| `All` | lit tous les utilisateurs, et pas uniquement la première page |
| `Filter 'assignedLicenses/$count ne 0'` | uniquement les utilisateurs ayant au moins une licence attribuée |
| `ConsistencyLevel eventual` | requis pour le filtre `$count` (requête avancée) |
| `CountVariable anzahl` | enregistre le nombre total dans la variable `$anzahl` |
| `Property UsageLocation` | charge également l’emplacement d’utilisation |
| `Group-Object UsageLocation` | compte les utilisateurs par pays |

</details>

Ce nombre constitue la limite supérieure. Vous démontrez combien d’utilisateurs n’avaient réellement pas accès à l’aide de vos propres données : tickets du service d’assistance horodatés, messages d’erreur d’Outlook et d’Outlook sur le web (captures d’écran), résultats de tests de disponibilité issus de votre monitoring ainsi que journaux de connexion et d’accès. Plus ces informations sont précises, moins le résultat dépend uniquement de la télémétrie de Microsoft.

## Petites entreprises : généralement via le partenaire

En Suisse, les petites et moyennes entreprises achètent généralement Microsoft 365 auprès d’un partenaire du Cloud Solution Provider Program (CSP) ou directement par carte de crédit dans l’Admin Center. Vous pouvez déterminer la procédure applicable dans le Microsoft 365 Admin Center sous **Facturation > Vos produits** : si un partenaire est indiqué comme source d’approvisionnement, le contrat est un contrat CSP.

### Acheté auprès d’un partenaire CSP

Dans ce cas, Microsoft n’accepte pas votre demande directement. Selon le Partner Center, seuls les fournisseurs directs CSP et les fournisseurs indirects (distributeurs) peuvent demander des avoirs, à l’exclusion des revendeurs indirects. Si votre prestataire informatique est un revendeur indirect, la demande passe donc par lui vers son distributeur, puis vers Microsoft. Chaque niveau a ses propres délais, souvent plus courts. Contactez-le donc maintenant et non seulement fin décembre.

Votre partenaire a besoin de votre part :

1.  de l’ID du tenant,
2.  de l’identifiant de l’incident (EX1550543),
3.  de la période, du nombre et des emplacements des utilisateurs concernés ainsi que des erreurs observées,
4.  d’un e-mail dans lequel vous demandez l’avoir SLA. Il doit provenir d’une adresse du domaine concerné ; Microsoft n’accepte pas un e-mail provenant d’une adresse privée.

Microsoft connaît l’incident lui-même. La demande doit démontrer que votre organisation a été concernée : selon Microsoft, le nom, l’ID du tenant et le numéro de ticket ne suffisent pas à eux seuls. Si l’avoir est accordé, Microsoft le crédite au partenaire, qui vous le transmet ensuite. Les modalités sont régies par votre contrat avec le partenaire. Renseignez-vous si rien n’apparaît sur la facture suivante.

### Acheté directement auprès de Microsoft

Si vous avez payé Microsoft 365 directement à Microsoft, vous déposez vous-même la demande dans l’Admin Center. Un rôle d’administrateur est nécessaire :

1.  Sous `admin.cloud.microsoft` ouvrez **Aide et support** en bas à droite.
2.  Décrivez votre demande, par exemple « Request an SLA service credit for a Microsoft 365 outage », puis sélectionnez **Contacter le support**.
3.  Choisissez l’e-mail comme moyen de contact. Le titre doit inclure « SLA service credit request » et l’identifiant EX1550543.
4.  Dans la description, indiquez les cinq informations obligatoires de la section précédente et joignez les justificatifs.

Une demande distincte est nécessaire pour chaque service et chaque incident. Selon le SLA, Microsoft traite généralement les demandes dans un délai de 45 jours.

### Cela en vaut-il la peine pour les petites entreprises ?

Pour dix utilisateurs, l’avoir est faible : dans une suite, seule la part Exchange compte, dont 25 ou 50 % pour un mois. L’effort est également faible si vous rassemblez les informations dès maintenant. La demande présente un second avantage : elle documente auprès du partenaire et de la direction l’ampleur des conséquences pour votre organisation. Cela aide ensuite à déterminer si une solution de secours pour l’e-mail est nécessaire.

## Grandes entreprises : directement auprès de Microsoft, avec leurs propres données de mesure

Les grandes organisations attribuent généralement les licences Microsoft 365 via un Enterprise Agreement (EA) ou le Microsoft Customer Agreement for Enterprise (MCA-E). La demande est alors adressée directement à Microsoft, soit via l’Admin Center comme décrit ci-dessus, soit via le contrat Unified Support. Si vous avez un Customer Success Account Manager (CSAM), informez-le de la demande ; il peut en suivre le traitement. En cas d’EA conclu par l’intermédiaire d’un Licensing Solution Partner (LSP), vérifiez également sur quelle facture l’avoir apparaîtra.

Avec plusieurs milliers d’utilisateurs, les montants sont significatifs et Microsoft vérifie donc les informations avec une attention correspondante. La différence avec un petit tenant réside dans l’administration de la preuve :

1.  **Déterminer précisément les utilisateurs concernés.** Les grands tenants sont répartis sur plusieurs bases de données et régions des centres de données de Microsoft ; toutes les boîtes aux lettres n’ont pas été perturbées pendant la même durée. Déclarer forfaitairement tous les utilisateurs comme concernés est contestable pour les grands tenants. Des données de mesure sont plus solides : tests synthétiques contre Outlook sur le web par emplacement, données de disponibilité de votre propre monitoring, analyses des tickets du service d’assistance par emplacement et par heure.

2.  **Indiquer les emplacements.** Le SLA exige les emplacements des utilisateurs concernés. Pour les organisations internationales, il est utile de répartir les données par pays, car la panne a d’abord touché l’Amérique du Nord et l’Amérique du Sud, puis d’autres régions.

3.  **Délimiter proprement l’hybride.** Le SLA exclut les retards dans votre propre centre de données, par exemple les e-mails dans la file d’attente de serveurs Exchange locaux à destination du tenant. Démontrez que l’erreur se situait au sein de Microsoft 365. Les files d’attente des serveurs locaux en sont une bonne preuve, car elles montrent qu’Exchange Online n’acceptait pas les e-mails ou seulement avec retard.

4.  **Choisir le niveau de service.** Si ce sont principalement les délais de remise qui ont été affectés chez vous, et moins l’accès aux boîtes aux lettres, le niveau de service relatif au délai de remise peut constituer une meilleure base. Une seule demande est autorisée par incident et par service ; le choix doit donc être fait avant la demande.

5.  **Impliquer les achats et la gestion des licences.** L’avoir apparaît sur la facture, et non dans le centre de coûts informatique. Convenez avec les achats de qui dépose la demande, qui contrôle l’avoir et comment il est comptabilisé.

Les grandes organisations devraient également préparer la procédure pour la prochaine panne : un modèle comprenant les cinq informations obligatoires, un monitoring qui mesure la disponibilité d’Outlook sur le web par emplacement et une responsabilité clairement définie pour les demandes SLA.

## Modèle de demande

Le modèle suivant contient les informations obligatoires du SLA. Microsoft traite les demandes en anglais le plus rapidement :

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

1.  [Service Level Agreement for Microsoft Online Services (aperçu des versions)](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services): version du 17 novembre 2026, définition de l’indisponibilité pour Exchange Online, niveaux d’avoir, informations obligatoires, délais et exclusions.

2.  [Microsoft Learn: How and when to request a credit from Microsoft](https://learn.microsoft.com/en-us/partner-center/billing/request-credit): procédure de demande pour les partenaires CSP, exclusion des revendeurs indirects, justificatifs requis, classification des Advisories.

3.  [Microsoft Learn: Get-MgServiceAnnouncementIssue](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.devices.serviceannouncement/get-mgserviceannouncementissue): cmdlet permettant de récupérer un incident d’intégrité du service via Microsoft Graph.

4.  [Microsoft 365 Status (status.cloud.microsoft)](https://status.cloud.microsoft/): messages d’état publics relatifs à EX1550543 et MO1550821.

5.  [Windows Latest: Microsoft 365 outage, you may get money off your bill](https://www.windowslatest.com/2026/11/13/microsoft-365-outage-you-may-get-money-off-your-bill-if-you-use-it-for-work-but-youll-have-to-ask-microsoft-for-it-as-a-server-failure-takes-down-outlook-teams-and-exchange/): procédure de demande dans l’Admin Center pour la panne de novembre, indication concernant les clients particuliers.

6.  [Office Watch: Microsoft 365 went down? How to claim your SLA credit](https://office-watch.com/?p=78178): exclusion des licences privées, gratuites et à but non lucratif, une demande par service et incident.

7.  [Suivi en direct : Exchange Online, panne mondiale de l’accès aux boîtes aux lettres](https://rafaelpfister.ch/liveticker/exchange-online-ausfall-2026-11): déroulement de l’incident des 12 et 13 novembre 2026 avec la chronologie de Microsoft.
