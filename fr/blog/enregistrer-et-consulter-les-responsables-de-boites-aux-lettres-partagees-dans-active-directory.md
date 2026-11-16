---
title: "Enregistrer et consulter les responsables de boîtes aux lettres partagées dans Active Directory"
navTitle: "Responsables de boîtes aux lettres partagées"
description: "En cas de message bloqué provenant d’une boîte aux lettres partagée, les solutions DLP doivent savoir qui informer. Quels attributs AD conviennent, lesquels ne représentent que des droits d’accès, pourquoi les autorisations Exchange Online ne remontent pas dans l’AD local et comment un fournisseur consulte automatiquement l’attribution via LDAP ou Microsoft Graph."
date: "2026-11-16"
kategorie: "Active Directory / Entra"
timeToRead: "8 min de lecture"
themen:
  - active-directory-entra
  - microsoft-365-exchange
produkte:
  - "entra"
  - "exchange-online"
  - "exchange"
protokolle:
  - "ldap"
  - "powershell"
slug: "enregistrer-et-consulter-les-responsables-de-boites-aux-lettres-partagees-dans-active-directory"
translationId: "article-ed41bd7fefc08361"
translationOf: shared-mailbox-verantwortliche-ad
url: https://rafaelpfister.ch/fr/blog/enregistrer-et-consulter-les-responsables-de-boites-aux-lettres-partagees-dans-active-directory
translationSourceHash: 54b0d22f6d1fd74974494b96e5566af25ea846062f8f9ce6a255607e40f937e6
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T08:51:55.973Z
translationReview: automatic
---

# Enregistrer et consulter les responsables de boîtes aux lettres partagées dans Active Directory

Dans le cadre d’un projet de prévention de la perte de données, la demande suivante m’est parvenue : dès qu’un message provenant d’une boîte aux lettres partagée (Shared Mailbox) est bloqué, les personnes responsables doivent être informées afin de pouvoir réagir. Le fournisseur de la solution DLP doit lire automatiquement cette attribution dans Active Directory. La question est donc la suivante : où l’AD indique-t-il qui est responsable d’une boîte aux lettres partagée ?

En bref : l’AD ne connaît que les droits d’accès, pas les responsabilités. Certains attributs indiquent qui peut accéder à une boîte aux lettres ou envoyer des messages en son nom, et même ceux-ci ne sont qu’incomplets dès lors que la boîte aux lettres se trouve dans Exchange Online. Pour une automatisation fiable, la personne responsable doit être explicitement renseignée dans un attribut. L’attribut standard `manager` convient à cet effet, est synchronisé par Entra Connect et peut être interrogé via LDAP et Microsoft Graph.

## L’accès n’est pas la responsabilité

Dans l’AD, une boîte aux lettres partagée est un objet utilisateur désactivé doté d’attributs Exchange. Les personnes qui l’utilisent sont déterminées par les autorisations : accès complet (FullAccess), envoyer en tant que (Send As) et envoyer de la part de (Send on Behalf). Ces listes contiennent souvent dix ou vingt personnes, ainsi que des suppléants, des apprentis et des comptes de service. Une alerte DLP envoyée à toutes les personnes autorisées atteint de nombreuses personnes dont aucune ne se sent responsable.

La question de la responsabilité est organisationnelle : une personne (ou un petit groupe) décide si un message bloqué était légitime et gère les autorisations. Cette information n’existe dans l’AD que si quelqu’un la renseigne.

## Attributs AD envisageables

| Attribut | Contenu | Pertinence pour les responsables |
|---|---|---|
| `manager` | DN d’une personne, à valeur unique, faisant partie des classes de personnes et donc disponible sur l’objet utilisateur de la boîte aux lettres | Adapté. Est synchronisé par Entra Connect vers Entra ID et peut être lu via Graph |
| `managedBy` | DN d’un gestionnaire, à valeur unique | Non disponible sur l’objet utilisateur. Selon le schéma, défini entre autres pour les groupes, contacts, ordinateurs et unités d’organisation, mais pas pour la classe User |
| `msExchDelegateListLink` | DN des personnes disposant d’un accès complet avec Automapping, à valeurs multiples ; référence inverse `msExchDelegateListLinkBL` sur l’utilisateur | Accès uniquement et incomplet : accès complet sans Automapping, droits via des groupes et droits attribués dans Exchange Online absents |
| `publicDelegates` | DN des personnes pouvant envoyer de la part de, à valeurs multiples | Accès uniquement. Est réécrit par Entra Connect dans les environnements hybrides |
| Liste d’autorisations (`nTSecurityDescriptor`) | Droit étendu « Send As » sur l’objet utilisateur | Uniquement l’envoi en tant que attribué localement. Les droits attribués dans Exchange Online n’y figurent pas |
| `msExchMailboxSecurityDescriptor` | Autorisations de boîte aux lettres pour les boîtes locales, binaire | Accès uniquement, uniquement pour les boîtes aux lettres sur des serveurs Exchange locaux, difficile à évaluer pour le fournisseur |
| `extensionAttribute1` à `extensionAttribute15` | Champs de texte libres | Possible, mais uniquement sous forme de texte ; aucune référence à un objet, aucun nettoyage automatique en cas de renommage ou de départ |

En pratique, l’indication selon laquelle l’attribution se trouve « directement dans l’AD » fait généralement référence à `msExchDelegateListLink` : dans ADUC ou ADSI Edit, on y voit effectivement des personnes. Cet attribut n’est toutefois créé que comme effet secondaire de l’Automapping Outlook et ne convient pas comme source de données pour les responsabilités.

## L’emplacement de la boîte aux lettres détermine la source de données

Si les boîtes aux lettres se trouvent dans Exchange Online et que les utilisateurs sont synchronisés depuis l’AD local (hybride), la règle suivante s’applique : les autorisations attribuées dans Exchange Online restent dans Exchange Online. Entra Connect ne réécrit pour Exchange Hybrid qu’une petite liste d’attributs (dont `publicDelegates` et `proxyAddresses`), mais ni `msExchDelegateListLink` ni les droits Send As. Dans ce cas, un AD local affiche l’état antérieur à la migration, complété par ce qui a été maintenu localement depuis.

Pour Exchange Online, seul Exchange Online lui-même fournit l’état réel des autorisations (`Get-EXOMailboxPermission`, `Get-EXORecipientPermission`). Le fournisseur aurait besoin pour cela d’un accès à Exchange Online PowerShell, ce qui est disproportionné pour une simple attribution. Un attribut maintenu sur l’objet utilisateur est plus simple et fonctionne indépendamment de l’emplacement de la boîte aux lettres.

## Recommandation : `manager` comme attribut du responsable

Pour chaque boîte aux lettres partagée, la personne responsable est renseignée dans l’attribut `manager`. Raisons :

- L’attribut fait partie du schéma standard ; aucune extension du schéma n’est nécessaire.
- Il contient une référence (DN) plutôt que du texte. Si la personne est renommée ou déplacée, la référence reste valide.
- Entra Connect synchronise `manager` vers Entra ID. Le fournisseur peut lire au choix via LDAP dans l’AD local ou via Microsoft Graph.
- Grâce à la référence inverse `directReports`, chaque personne responsable (et chaque rapport) voit de quelles boîtes aux lettres elle est responsable.

La limite : `manager` est à valeur unique. Lorsqu’un suppléant est indispensable, deux possibilités existent. Possibilité 1 : un groupe par boîte aux lettres, dont les membres sont les responsables ; son nom suit une convention (par exemple `SM-Verantwortlich-buchhaltung`) ou figure dans un `extensionAttribute`. Possibilité 2 : le suppléant n’est pas déduit par le fournisseur depuis l’AD, mais depuis le `manager` de la personne responsable (escalade vers le supérieur hiérarchique). La possibilité 2 ne nécessite aucun objet supplémentaire et suffit dans la plupart des organisations.

L’attribut n’est défini que dans le système source : dans l’AD local pour les objets synchronisés, dans Entra ID pour les objets exclusivement cloud.

## Établir l’inventaire : qui a aujourd’hui accès ?

Pour le remplissage initial, une liste des personnes actuellement autorisées est utile. La personne responsable en fait généralement partie ; le service métier confirme le choix. Pour les boîtes aux lettres dans Exchange Online :

```powershell
Connect-ExchangeOnline

$postfaecher = Get-EXOMailbox -RecipientTypeDetails SharedMailbox -ResultSize Unlimited

$bestand = foreach ($pf in $postfaecher) {
    $voll = Get-EXOMailboxPermission -Identity $pf.UserPrincipalName |
        Where-Object {
            $_.AccessRights -contains 'FullAccess' -and
            -not $_.IsInherited -and
            $_.User -notlike 'NT AUTHORITY\*'
        }
    $sendas = Get-EXORecipientPermission -Identity $pf.UserPrincipalName |
        Where-Object {
            $_.AccessRights -contains 'SendAs' -and
            $_.Trustee -notlike 'NT AUTHORITY\*'
        }
    [pscustomobject]@{
        Postfach  = $pf.PrimarySmtpAddress
        Vollzugriff = ($voll.User -join '; ')
        SendenAls = ($sendas.Trustee -join '; ')
    }
}

$bestand | Export-Csv -Path .\shared-mailbox-bestand.csv -NoTypeInformation -Encoding UTF8
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-RecipientTypeDetails SharedMailbox` | Uniquement les boîtes aux lettres partagées, pas les boîtes utilisateur, de salle ou d’équipement |
| `-ResultSize Unlimited` | Toutes les boîtes aux lettres au lieu de la limite standard de 1000 |
| `-Identity` | Boîte aux lettres dont les autorisations sont interrogées |
| `AccessRights -contains 'FullAccess'` | Filtre sur l’accès complet ; les autres droits de boîte aux lettres sont exclus |
| `-not $_.IsInherited` | Supprime les entrées héritées (administrateurs, comptes système) |
| `User -notlike 'NT AUTHORITY\*'` | Supprime l’entrée `NT AUTHORITY\SELF` de la boîte aux lettres elle-même |
| `-NoTypeInformation` | Aucune ligne de type au début du fichier CSV |
| `-Encoding UTF8` | Les caractères accentués dans les noms sont conservés |

</details>

Si un groupe est autorisé, son nom apparaît dans la liste ; ses membres peuvent être résolus avec `Get-DistributionGroupMember` ou `Get-ADGroupMember`.

## Définir les responsables

L’attribution confirmée est préparée sous forme de CSV avec les colonnes `Postfach` et `Verantwortlich` (adresses e-mail dans les deux cas), puis définie dans l’AD local :

```powershell
Import-Module ActiveDirectory

$zuordnung = Import-Csv -Path .\shared-mailbox-verantwortliche.csv

foreach ($eintrag in $zuordnung) {
    $postfach = Get-ADUser -Filter "mail -eq '$($eintrag.Postfach)'"
    $person = Get-ADUser -Filter "mail -eq '$($eintrag.Verantwortlich)'"
    Set-ADUser -Identity $postfach -Manager $person
}
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-Filter "mail -eq '…'"` | Recherche l’objet à l’aide de l’adresse e-mail principale |
| `-Identity` | Objet utilisateur de la boîte aux lettres partagée |
| `-Manager` | Écrit le DN de la personne responsable dans l’attribut `manager` |

</details>

Pour les objets exclusivement cloud sans AD local, `Set-MgUserManagerByRef` du SDK PowerShell Microsoft Graph définit la même valeur dans Entra ID.

## Consultation par le fournisseur : LDAP

Les boîtes aux lettres partagées peuvent être identifiées dans l’AD via `msExchRecipientTypeDetails` : `4` désigne une boîte aux lettres partagée sur un serveur Exchange local, `34359738368` une boîte aux lettres partagée dans Exchange Online (RemoteSharedMailbox). La requête que le fournisseur peut intégrer à sa solution :

```text
Basis:     DC=example,DC=com (Unterbaum)
Filter:    (&(objectClass=user)(|(msExchRecipientTypeDetails=4)(msExchRecipientTypeDetails=34359738368)))
Attribute: mail, proxyAddresses, manager
```

La valeur de `manager` est un DN. L’adresse e-mail de la personne responsable est obtenue par une seconde requête sur ce DN avec l’attribut `mail`. Un compte de lecture sans droits supplémentaires suffit, car `manager` et `mail` sont lisibles par les utilisateurs authentifiés. La connexion doit utiliser LDAPS (port 636).

Pour vérifier l’attribution dans PowerShell :

```powershell
$filter = '(|(msExchRecipientTypeDetails=4)(msExchRecipientTypeDetails=34359738368))'

$verantwortlich = @{
    Name       = 'Verantwortlich'
    Expression = {
        if ($_.manager) {
            (Get-ADUser -Identity $_.manager -Properties mail).mail
        }
    }
}

Get-ADUser -LDAPFilter $filter -Properties mail, manager |
    Select-Object mail, $verantwortlich |
    Sort-Object mail
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-LDAPFilter` | Filtre LDAP sur les deux types de destinataires pour les boîtes aux lettres partagées |
| `-Properties mail, manager` | Charge les deux attributs que `Get-ADUser` ne fournit pas par défaut |
| `Expression = { … }` | Résout le DN dans `manager` en adresse e-mail de la personne responsable |

</details>

Pour l’attribution des messages, le fournisseur doit prendre en compte toutes les adresses de `proxyAddresses`, et non seulement `mail` : un message peut être envoyé avec une adresse secondaire de la boîte aux lettres.

## Consultation par le fournisseur : Microsoft Graph

Si le fournisseur n’a pas d’accès LDAP au réseau interne, Microsoft Graph est une bonne option. Les boîtes aux lettres partagées existent dans Entra ID sous forme d’objets utilisateur (désactivés), et `manager` y est disponible grâce à la synchronisation :

```http
GET https://graph.microsoft.com/v1.0/users/buchhaltung@example.com/manager?$select=displayName,mail
```

L’inscription d’application du fournisseur nécessite pour cela l’autorisation d’application `User.Read.All`. Graph ne fournit pas directement la liste de toutes les boîtes aux lettres partagées, car Entra ID ne connaît pas le type de boîte aux lettres. Le fournisseur interroge donc de manière ciblée l’adresse de l’expéditeur du message bloqué, ce qui suffit de toute façon pour le cas d’utilisation DLP.

## Intégration dans la notification DLP

La manière dont la notification est déclenchée techniquement dépend de la solution DLP. Avec Microsoft Purview DLP, il convient de noter que les notifications peuvent être envoyées en plus à des personnes renseignées de manière fixe, mais uniquement à des destinataires individuels, et non à des groupes ou listes de distribution. Une attribution dynamique « boîte aux lettres vers personne responsable » n’est généralement pas prévue. Pour un petit nombre de boîtes aux lettres, une règle par boîte est envisageable ; pour un grand nombre, il faut un processus en aval qui récupère les incidents, résout l’adresse de l’expéditeur via LDAP ou Graph et informe la personne responsable.

La même logique s’applique à une solution externe : déterminer l’adresse de l’expéditeur du message bloqué, trouver la boîte aux lettres via `mail` ou `proxyAddresses`, résoudre `manager`, notifier. Si `manager` est absent, la solution doit signaler l’incident à une adresse collective définie (service desk ou protection des données) plutôt que de rejeter l’alerte.

## Maintenance en exploitation

Pour maintenir l’attribution à jour, deux règles suffisent en pratique :

- Les nouvelles boîtes aux lettres partagées ne sont créées qu’avec une personne responsable renseignée ; le formulaire de demande contient ce champ.
- Un rapport mensuel répertorie les boîtes aux lettres sans `manager` et celles dont la personne responsable est désactivée.

```powershell
$filter = '(|(msExchRecipientTypeDetails=4)(msExchRecipientTypeDetails=34359738368))'

Get-ADUser -LDAPFilter $filter -Properties mail, manager |
    Where-Object {
        -not $_.manager -or
        -not (Get-ADUser -Identity $_.manager).Enabled
    } |
    Select-Object mail, manager
```

Lors du départ d’une personne, `directReports` sur son objet indique quelles boîtes aux lettres nécessitent une nouvelle personne responsable. Cette vérification peut être intégrée au processus de départ.

## Sources

1.  [Attribut Manager (schéma Active Directory)](https://learn.microsoft.com/en-us/windows/win32/adschema/a-manager): attribut DN à valeur unique des classes de personnes, référence inverse `directReports`.

2.  [Attribut Managed-By (schéma Active Directory)](https://learn.microsoft.com/en-us/windows/win32/adschema/a-managedby): liste des classes pour lesquelles `managedBy` est défini ; la classe User n’en fait pas partie.

3.  [Microsoft Entra Connect Sync : attributs synchronisés vers Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-sync-attributes-synchronized): attributs synchronisés, y compris `manager`, ainsi que la liste des attributs de réécriture Exchange Hybrid.

4.  [Get-EXOMailboxPermission](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-exomailboxpermission): interrogation des autorisations de boîte aux lettres dans Exchange Online.

5.  [Get-EXORecipientPermission](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-exorecipientpermission): interrogation des autorisations Send As dans Exchange Online.

6.  [Adding or removing mailbox permission in EAC doesn't address the msExchDelegateListLink attribute](https://support.microsoft.com/en-us/help/4502159): article du support Microsoft sur le lien entre l’accès complet et `msExchDelegateListLink` dans Exchange 2016 et 2019.

7.  [List manager (Microsoft Graph)](https://learn.microsoft.com/en-us/graph/api/user-list-manager): point de terminaison Graph et autorisations nécessaires pour consulter le manager.

8.  [Send email notifications and show policy tips for DLP policies](https://learn.microsoft.com/en-us/purview/dlp-use-notifications-and-policy-tips): destinataires des notifications DLP dans Microsoft Purview ; uniquement des destinataires individuels, pas de groupes.
