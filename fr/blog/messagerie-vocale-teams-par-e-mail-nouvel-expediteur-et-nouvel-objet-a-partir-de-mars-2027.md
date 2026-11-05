---
title: "Messagerie vocale Teams par e-mail : nouvel expéditeur et nouvel objet à partir de mars 2027 (MC1511072)"
navTitle: "Expéditeur de la messagerie vocale"
description: "Microsoft modifie la livraison des messages vocaux Teams par e-mail : l’expéditeur sera désormais « Microsoft Teams » depuis le domaine de votre tenant, et les informations sur l’appelant seront déplacées dans l’objet. Cet article indique qui est concerné, comment vérifier les règles de transport, les listes d’autorisation et les automatisations, ainsi que les adaptations à effectuer avant mars 2027."
date: "2026-11-02"
kategorie: "Microsoft Teams"
timeToRead: "4 min de lecture"
themen:
  - microsoft-teams
  - microsoft-365-exchange
produkte:
  - "teams"
  - "exchange-online"
protokolle:
  - "smtp"
  - "mail-auth"
  - "powershell"
hauptthema: "microsoft-teams"
slug: "messagerie-vocale-teams-par-e-mail-nouvel-expediteur-et-nouvel-objet-a-partir-de-mars-2027"
translationId: "article-b75c975f086dd0d5"
translationOf: teams-voicemail-mail-absender
url: https://rafaelpfister.ch/fr/blog/messagerie-vocale-teams-par-e-mail-nouvel-expediteur-et-nouvel-objet-a-partir-de-mars-2027
translationSourceHash: 32089440ce20d39865ec421317bed6588c48eace59eb8f114015c626da412eba
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:36:41.424Z
translationReview: automatic
---

Avec le message du Centre de messages MC1511072 du 16 octobre 2026, Microsoft annonce une nouvelle méthode de livraison des e-mails de messagerie vocale de Teams. Selon le calendrier actuel, le déploiement commencera début mars 2027 et devrait être achevé fin juin 2027 ; Microsoft se réserve le droit de reporter ces dates et prévoit de communiquer ultérieurement les échéances définitives ainsi que les éventuelles actions requises des clients. L’objectif est de prendre en charge les domaines d’expéditeur personnalisés (Bring Your Own Domain), afin que les e-mails de messagerie vocale respectent les stratégies SPF, DKIM et DMARC de l’organisation.

## Ce qui change

Le champ « De » ne contiendra plus ni le nom, ni le numéro de téléphone, ni l’adresse e-mail de l’appelant. Ces informations figureront désormais dans l’objet et resteront présentes dans le corps du message. L’affichage dans Teams reste inchangé.

| Caractéristique | Aujourd’hui | Après le déploiement |
|---|---|---|
| Expéditeur (messagerie vocale personnelle) | Nom ou numéro de l’appelant | `"Microsoft Teams" no-reply@<Standard-Tenant-Domain>` ou domaine personnalisé, s’il est configuré |
| Domaine de l’expéditeur | `skype.voicemail.microsoft.com` pour les appels PSTN, domaine du tenant pour les appels Teams à Teams | Domaine par défaut du tenant ou domaine personnalisé |
| Objet (messagerie vocale personnelle) | `Voice mail (22 seconds)` | `Identified: John Smith (+19371817273) - Voicemail (00:22)` ou `Unidentified: +19371817273 - Voicemail (00:22)` |
| File d’attente d’appels (messagerie vocale partagée) | `Shared Voicemail (<CQ-Name>)` | `John Smith (+37251223123) - Shared voicemail in <CQ-Name> (00:05)` |
| Standard automatique (messagerie vocale partagée) | `Shared Voicemail (<AA-Name>)` | `John Smith - Shared voicemail in <AA-Name> (00:11)` |

Pour les utilisateurs, cela a deux conséquences visibles : toutes les messageries vocales apparaîtront dans la boîte de réception avec le même expéditeur, « Microsoft Teams », et, comme l’adresse est une adresse sans réponse, la fonction « Répondre » ne permettra plus de joindre l’appelant.

## Qui est concerné

Toutes les organisations utilisant Teams Calling avec la messagerie vocale sont concernées, c’est-à-dire Teams Phone avec Calling Plan, Operator Connect ou Direct Routing, ainsi que les files d’attente d’appels et les standards automatiques avec messagerie vocale partagée. Le message ne mentionne aucune restriction de licence ou de cloud ; le déploiement est annoncé à l’échelle mondiale.

Des mesures sont nécessaires lorsque des éléments dépendent de l’expéditeur ou de l’objet actuels :

- les règles de transport Exchange qui vérifient `skype.voicemail.microsoft.com`, l’appelant en tant qu’expéditeur ou l’objet `Voice mail` ou `Shared Voicemail`
- les entrées dans les listes d’autorisation : Tenant Allow/Block List, autorisations d’usurpation, `AllowedSenderDomains` dans les stratégies anti-spam
- les règles de boîte de réception des utilisateurs et des boîtes aux lettres de groupe des files d’attente d’appels, qui classent ou transfèrent les messages vocaux selon l’expéditeur ou l’objet
- les flux Power Automate, les systèmes de tickets et les intégrations CRM qui lisent le numéro de téléphone de l’appelant dans l’expéditeur
- les passerelles en amont ou en aval (par exemple dans le cadre d’un flux de messagerie centralisé en environnement hybride) qui contiennent des règles ou des exceptions pour le domaine de messagerie vocale

## Comment vérifier si vous êtes concerné

Les requêtes suivantes s’exécutent dans une session PowerShell Exchange Online (`Connect-ExchangeOnline`).

### Rechercher les règles de transport

```powershell
$muster = 'skype\.voicemail|voice ?mail|shared voicemail'
Get-TransportRule |
  Where-Object { ($_ | Format-List * | Out-String) -match $muster } |
  Select-Object Name, State, Priority
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `$muster` | Expression régulière pour le domaine d’expéditeur actuel et les textes d’objet actuels |
| `Format-List *` | Affiche toutes les propriétés de la règle afin de rechercher également dans les conditions et les exceptions |
| `Out-String` | Convertit la sortie en texte auquel `-match` est appliqué |
| `Select-Object Name, State, Priority` | Affiche uniquement le nom, l’état et l’ordre des règles trouvées |

</details>

Pour chaque règle trouvée, vérifiez les conditions telles que `FromAddressContainsWords`, `SenderDomainIs` ou `SubjectContainsWords`.

### Listes d’autorisation et stratégies anti-spam

```powershell
Get-TenantAllowBlockListItems -ListType Sender |
  Where-Object Value -match 'voicemail'
Get-TenantAllowBlockListSpoofItems |
  Where-Object SpoofedUser -match 'voicemail'
Get-HostedContentFilterPolicy |
  Select-Object Name, AllowedSenderDomains, AllowedSenders
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-ListType Sender` | Lit les entrées d’expéditeur de la Tenant Allow/Block List |
| `Where-Object Value -match 'voicemail'` | Filtre les entrées dont la valeur contient le domaine de messagerie vocale |
| `Get-TenantAllowBlockListSpoofItems` | Répertorie les autorisations et blocages d’usurpation |
| `AllowedSenderDomains, AllowedSenders` | Autorisations directement définies dans les stratégies anti-spam |

</details>

### Déterminer le volume réel d’e-mails

Le suivi des messages indique si et où les messages vocaux sont actuellement livrés :

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddDays(-10) -EndDate (Get-Date) -ResultSize 5000 |
  Where-Object { $_.SenderAddress -like '*@skype.voicemail.microsoft.com' -or $_.Subject -match 'voice ?mail' } |
  Group-Object RecipientAddress | Sort-Object Count -Descending |
  Select-Object Count, Name
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-StartDate` / `-EndDate` | Période de recherche ; Exchange Online fournit au maximum les 90 derniers jours, et au plus 10 jours par requête |
| `-ResultSize 5000` | Nombre maximal de résultats par requête |
| `Where-Object …` | Conserve les e-mails provenant du domaine PSTN de messagerie vocale ou ayant un objet de messagerie vocale |
| `Group-Object RecipientAddress` | Compte les messages vocaux par destinataire |

</details>

La liste des destinataires indique quelles boîtes aux lettres et boîtes aux lettres de groupe reçoivent des messages vocaux. Il est alors utile de vérifier les règles de boîte de réception (`Get-InboxRule -Mailbox <Adresse>`) et les flux connectés.

## Que faire

1.  Modifier les règles pour utiliser l’objet plutôt que l’expéditeur : alignez les nouvelles conditions sur les futurs modèles d’objet `- Voicemail (` et `- Shared voicemail in `. Jusqu’à ce que le déploiement atteigne votre tenant, conservez les anciennes et les nouvelles conditions en parallèle.

2.  Supprimer les autorisations pour `skype.voicemail.microsoft.com` après le déploiement. Les e-mails provenant du domaine de votre tenant ne nécessitent aucune exception ; une autorisation pour votre propre domaine représenterait un risque de sécurité, car elle faciliterait l’usurpation.

3.  Adapter les automatisations qui lisent le numéro de l’appelant dans l’expéditeur. Le numéro figurera désormais entre parenthèses dans l’objet ou, pour les appelants inconnus, directement après `Unidentified:`.

4.  Préparer un domaine d’expéditeur personnalisé : Microsoft annoncera les instructions de configuration pour l’envoi sous un domaine personnalisé dans le message suivant. Si DMARC est défini sur `p=reject` pour ce domaine, SPF et DKIM doivent couvrir l’envoi depuis Exchange Online ; sinon, les messages vocaux seront placés en quarantaine.

5.  Informer les utilisateurs et le service desk : expéditeur « Microsoft Teams », informations sur l’appelant dans l’objet, impossibilité de répondre à l’appelant par e-mail.

## Sources

1.  [MC1511072: Microsoft Teams: Upcoming change to voicemail message delivery by email](https://mc.merill.net/message/MC1511072): texte archivé du message du Centre de messages avec le calendrier et la comparaison entre l’expéditeur et l’objet actuels et futurs.

2.  [Microsoft Learn: Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2): paramètres et limites du suivi des messages dans Exchange Online.

3.  [Microsoft Learn: Get-TenantAllowBlockListItems](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-tenantallowblocklistitems): lecture de la Tenant Allow/Block List.

4.  [Microsoft Learn: Set up voicemail for Teams](https://learn.microsoft.com/en-us/microsoftteams/set-up-phone-system-voicemail): principes de base de la messagerie vocale Teams et de sa livraison par e-mail.
