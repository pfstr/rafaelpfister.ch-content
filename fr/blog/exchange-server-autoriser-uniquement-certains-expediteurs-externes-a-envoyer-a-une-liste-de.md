---
title: "Exchange Server : autoriser uniquement certains expéditeurs externes à envoyer à une liste de distribution"
navTitle: "Liste de distribution : expéditeurs externes"
description: "Une liste de distribution dans Exchange Server peut être configurée pour n’être accessible qu’à certains expéditeurs externes. Deux méthodes avec PowerShell : restriction de remise via des contacts de messagerie ou règle de transport. Découvrez également les limites de la vérification de l’expéditeur et comment DMARC les comble."
date: "2026-11-05"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "7 min de lecture"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
  - "exchange-on-premises"
protokolle:
  - "smtp"
  - "mail-auth"
  - "powershell"
related:
  - exchange-authmechanism-10-authas-internal
  - exchange-message-tracking-und-receive-connectoren-analysieren
slug: "exchange-server-autoriser-uniquement-certains-expediteurs-externes-a-envoyer-a-une-liste-de"
translationId: "article-b3dc70052f5e8729"
translationOf: exchange-verteiler-externe-absender
url: https://rafaelpfister.ch/fr/blog/exchange-server-autoriser-uniquement-certains-expediteurs-externes-a-envoyer-a-une-liste-de
translationSourceHash: cff4b579945841f2469f23b5e44d834f6be7528cf9817f9d7ade0fd613f1c582
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:22:04.690Z
translationReview: automatic
---

# Exchange Server : autoriser uniquement certains expéditeurs externes à envoyer à une liste de distribution

Par défaut, une liste de distribution dans Exchange Server n’accepte que les messages d’expéditeurs authentifiés, c’est-à-dire internes. Si un partenaire, un fournisseur ou un client doit également pouvoir joindre la liste sans pour autant l’ouvrir à l’ensemble d’Internet, deux possibilités existent : une restriction de remise directement sur le groupe ou une règle de transport. Les deux fonctionnent dans Exchange Server 2016, 2019 et la Subscription Edition.

## Situation de départ : RequireSenderAuthenticationEnabled

La propriété `RequireSenderAuthenticationEnabled` détermine si un groupe accepte des messages externes. Si elle est définie sur `$true` (valeur par défaut pour les nouveaux groupes de distribution), Exchange rejette tout message provenant d’un expéditeur non authentifié. Les deux variantes suivantes supposent donc que cette vérification soit désactivée :

```powershell
Set-DistributionGroup -Identity "Projektteam" -RequireSenderAuthenticationEnabled $false
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Identity` | Le groupe de distribution (nom, alias ou adresse e-mail). |
| `-RequireSenderAuthenticationEnabled $false` | Le groupe accepte également les messages d’expéditeurs externes non authentifiés. |

</details>

Sans autre restriction, tout expéditeur externe peut écrire à la liste après cette étape. La restriction à certains expéditeurs intervient à l’étape suivante, idéalement pendant la même fenêtre de maintenance.

## Variante 1 : restriction de remise avec des contacts de messagerie

La liste des expéditeurs autorisés (`AcceptMessagesOnlyFromSendersOrMembers`) n’accepte que des objets de votre propre annuaire : boîtes aux lettres, utilisateurs de messagerie, contacts de messagerie et groupes. Il n’est pas possible d’y ajouter une simple adresse SMTP. Créez donc un contact de messagerie pour chaque expéditeur externe. Pour éviter que le contact apparaisse dans la liste d’adresses globale, masquez-le :

```powershell
$kontakt = @{
    Name                 = "Extern Max Muster"
    ExternalEmailAddress = "max.muster@partner.example"
    OrganizationalUnit   = "example.local/Kontakte/Extern"
}
New-MailContact @kontakt
Set-MailContact -Identity "Extern Max Muster" -HiddenFromAddressListsEnabled $true
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Name` | Nom d’affichage du contact dans Active Directory. Un préfixe uniforme tel que « Externe » facilite la maintenance ultérieure. |
| `ExternalEmailAddress` | L’adresse externe utilisée par l’expéditeur. |
| `OrganizationalUnit` | OU cible dans Active Directory. Sans indication, le contact est placé dans le conteneur par défaut `Users`. |
| `-HiddenFromAddressListsEnabled $true` | Masque le contact dans la liste d’adresses globale et dans toutes les listes d’adresses. |

</details>

Ajoutez ensuite le contact comme expéditeur autorisé. La table de hachage avec `Add` complète la liste existante au lieu de l’écraser :

```powershell
$erlaubt = @{ Add = "Extern Max Muster", "Extern Anna Beispiel" }
Set-DistributionGroup -Identity "Projektteam" -AcceptMessagesOnlyFromSendersOrMembers $erlaubt
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-AcceptMessagesOnlyFromSendersOrMembers` | Liste des expéditeurs autorisés à joindre le groupe. Les expéditeurs individuels et les groupes sont possibles ; pour les groupes, tous les membres peuvent envoyer. |
| `@{ Add = ... }` | Ajoute des entrées sans supprimer celles qui existent. `@{ Remove = ... }` supprime des entrées ; une valeur sans table de hachage remplace toute la liste. |

</details>

Dès que cette liste contient au moins une entrée, Exchange rejette **tous les autres expéditeurs**, y compris les internes. Si les collaborateurs doivent continuer à écrire à la liste, ajoutez également un groupe approprié, par exemple la liste de distribution elle-même (tous les membres peuvent alors envoyer) ou un groupe contenant tous les collaborateurs. Les expéditeurs rejetés reçoivent un rapport de non-remise avec le statut `5.7.1`.

Vérifiez l’état actuel comme suit :

```powershell
Get-DistributionGroup -Identity "Projektteam" |
    Format-List Name, RequireSenderAuthenticationEnabled,
        AcceptMessagesOnlyFrom, AcceptMessagesOnlyFromDLMembers
```

`AcceptMessagesOnlyFromSendersOrMembers` est un paramètre regroupé : Exchange répartit les entrées entre les propriétés `AcceptMessagesOnlyFrom` (expéditeurs individuels) et `AcceptMessagesOnlyFromDLMembers` (groupes). C’est pourquoi la requête affiche ces deux propriétés. Le même paramètre se trouve dans le Centre d’administration Exchange, dans les propriétés du groupe sous Gestion de la remise (Delivery management).

Cette variante convient lorsqu’il n’y a que quelques expéditeurs externes à ajouter et que la liste des expéditeurs autorisés doit être visible directement sur l’objet du groupe. Son inconvénient est l’effort de maintenance : chaque nouvel expéditeur nécessite un contact dans Active Directory.

## Variante 2 : règle de transport sans contacts

Si tous les utilisateurs internes doivent continuer à pouvoir écrire à la liste et que seuls les expéditeurs externes doivent être limités, une règle de transport est plus simple. Elle ne nécessite aucun contact et peut également s’appliquer à des domaines entiers. `RequireSenderAuthenticationEnabled` doit ici aussi être défini sur `$false`.

```powershell
$regel = @{
    Name                              = "Projektteam: nur freigegebene externe Absender"
    SentTo                            = "projektteam@example.com"
    FromScope                         = "NotInOrganization"
    ExceptIfFromAddressMatchesPatterns = '^max\.muster@partner\.example$',
                                         '@lieferant\.example$'
    RejectMessageReasonText           = "Sie sind nicht berechtigt, an diese Adresse zu senden."
    RejectMessageEnhancedStatusCode   = "5.7.1"
}
New-TransportRule @regel
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Name` | Nom de la règle dans la liste des règles de transport. |
| `SentTo` | Condition : le message est adressé à ce groupe. |
| `FromScope NotInOrganization` | Condition : l’expéditeur n’appartient pas à votre organisation. Les messages internes ne sont donc pas concernés. |
| `ExceptIfFromAddressMatchesPatterns` | Exception : les adresses des expéditeurs qui correspondent à l’une des expressions régulières. `^…$` ancre une adresse individuelle, `@domain$` autorise un domaine entier. Échappez les points avec `\.`. |
| `RejectMessageReasonText` | Texte du rapport de non-remise envoyé à l’expéditeur rejeté. |
| `RejectMessageEnhancedStatusCode` | Code d’état étendu dans le rapport de non-remise ; `5.7.1` indique une autorisation manquante. |

</details>

Sans l’ancrage avec `^` et `$`, un modèle tel que `max\.muster@partner\.example` correspondrait également à `max.muster@partner.example.net`. Ajoutez de nouveaux expéditeurs avec `Set-TransportRule` et la liste complète étendue de modèles ; le paramètre remplace la liste existante.

## Limite des deux variantes : l’adresse de l’expéditeur peut être falsifiée

Les deux variantes vérifient uniquement l’adresse de l’expéditeur. Exchange ne détermine pas si le message provient réellement du partenaire autorisé. Toute personne connaissant l’adresse d’un expéditeur autorisé peut la falsifier et ainsi joindre la liste. Pour une liste dont les participants sont internes, cela constitue une porte d’entrée pour le phishing, qui donne en outre l’apparence d’une adresse connue.

SPF, DKIM et DMARC comblent cette faille. Si le domaine du partenaire applique une politique DMARC avec `p=reject` ou `p=quarantine` et que votre passerelle de messagerie applique cette politique, les messages falsifiés de ce domaine n’atteignent même pas le Exchange Server. Les domaines sans politique DMARC ou avec `p=none` ne bénéficient pas de cette protection.

Si votre passerelle inscrit le résultat de la vérification dans l’en-tête `Authentication-Results`, une seconde règle de transport peut également rejeter les expéditeurs autorisés si DMARC échoue :

```powershell
$regel = @{
    Name                            = "Projektteam: freigegebene Absender nur mit DMARC pass"
    SentTo                          = "projektteam@example.com"
    FromScope                       = "NotInOrganization"
    FromAddressMatchesPatterns      = '^max\.muster@partner\.example$',
                                      '@lieferant\.example$'
    ExceptIfHeaderMatchesMessageHeader = "Authentication-Results"
    ExceptIfHeaderMatchesPatterns   = 'dmarc=pass'
    RejectMessageReasonText         = "Absender konnte nicht verifiziert werden."
    RejectMessageEnhancedStatusCode = "5.7.1"
}
New-TransportRule @regel
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `FromAddressMatchesPatterns` | Condition : la règle ne s’applique qu’aux expéditeurs autorisés, avec les mêmes modèles que dans la première règle. |
| `ExceptIfHeaderMatchesMessageHeader` | Nom de l’en-tête vérifié par l’exception. |
| `ExceptIfHeaderMatchesPatterns` | Exception : si l’en-tête contient `dmarc=pass`, le message est remis. Tous les autres messages des expéditeurs autorisés sont rejetés. |

</details>

Cette règle ne fonctionne de manière fiable que si la passerelle supprime ou remplace les en-têtes `Authentication-Results` entrants depuis Internet. Sinon, un expéditeur peut fournir lui-même l’en-tête avec `dmarc=pass`. La documentation de votre passerelle indique quels en-têtes elle définit et si elle supprime les en-têtes tiers ; Exchange Server lui-même n’évalue pas DMARC.

## Quelle variante choisir

La restriction de remise avec des contacts convient aux listes ayant un cercle restreint et fixe d’expéditeurs externes, et permet également de limiter les expéditeurs internes si souhaité. La règle de transport convient si tous les utilisateurs internes doivent pouvoir envoyer, si des domaines partenaires entiers doivent être autorisés ou si aucun contact ne doit être créé dans Active Directory. Quelle que soit la variante, vérifiez la politique DMARC de chaque domaine autorisé et documentez qui maintient la liste des expéditeurs autorisés.

## Sources

1.  [Microsoft Learn: Set-DistributionGroup](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-distributiongroup?view=exchange-ps): paramètres AcceptMessagesOnlyFromSendersOrMembers et RequireSenderAuthenticationEnabled, types d’objets autorisés.

2.  [Microsoft Learn: Recipient resolution in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/recipient-resolution): comment le catégoriseur évalue les restrictions de remise et l’authentification de l’expéditeur.

3.  [Microsoft Learn: New-MailContact](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-mailcontact?view=exchange-ps): création de contacts de messagerie pour des adresses externes.

4.  [Microsoft Learn: New-TransportRule](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-transportrule?view=exchange-ps): conditions, exceptions et actions des règles de transport, notamment FromScope et HeaderMatchesPatterns.

5.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489): spécification des politiques DMARC p=none, quarantine et reject.

6.  [RFC 8601: Message Header Field for Indicating Message Authentication Status](https://datatracker.ietf.org/doc/html/rfc8601): structure de l’en-tête Authentication-Results et indication que les systèmes destinataires doivent supprimer les instances tierces.
