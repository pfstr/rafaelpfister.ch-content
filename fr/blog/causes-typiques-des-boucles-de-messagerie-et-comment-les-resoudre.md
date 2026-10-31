---
title: "Causes typiques des boucles de messagerie et comment les résoudre"
navTitle: "Résoudre les boucles de messagerie"
description: "Comment identifier et résoudre systématiquement les boucles SMTP dans Exchange Online, les environnements hybrides et les passerelles de messagerie en amont à l’aide de NDR, d’en-têtes, de Message Trace, d’objets destinataires et de connecteurs."
date: "2026-08-07"
kategorie: "Exchange OnPrem / hybride"
timeToRead: "12 min de lecture"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
related:
  - exchange-mailboxen-ohne-remote-move-cloud-neu-erstellen
  - totemomail-m365
  - ghost-sender-exchange-online-nebeneingang
slug: "causes-typiques-des-boucles-de-messagerie-et-comment-les-resoudre"
translationId: "article-4c91e7b2a8605fd3"
draft: false
translationOf: typische-ursachen-fuer-mail-loops-und-deren-behebung
translationSourceHash: 6bd8a9a875aa7795711db421084241c47d7b2c1f24abb9ba64bc1b18128d86b7
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:36:30.773Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/causes-typiques-des-boucles-de-messagerie-et-comment-les-resoudre
---

# Causes typiques des boucles de messagerie et comment les résoudre

Une boucle de messagerie se produit lorsque au moins deux systèmes de transport se transmettent sans cesse le même message. Aucun des systèmes ne se considère comme la destination finale, mais tous deux connaissent un prochain saut apparemment approprié. La boucle ne s’arrête que lorsqu’un serveur constate que le nombre autorisé de stations de transport a été dépassé et génère un NDR.

Dans Exchange, deux messages sont particulièrement révélateurs :

- `554 5.4.6 Hop count exceeded - possible mail loop` est généralement généré par l’Exchange local.
- `554 5.4.14 Hop count exceeded - possible mail loop ATTR34` est généré par Exchange Online.

La limite de sauts n’est pas la cause, mais la protection contre une répétition sans fin. L’augmenter ne résout donc rien. Il faut identifier le point où le message est renvoyé vers un système déjà traversé, contrairement à l’architecture cible.

## Reconnaître le schéma de boucle dans l’en-tête

Le NDR et les en-têtes complets des messages d’origine doivent être sauvegardés avant toute modification. Les lignes `Received` se lisent de bas en haut : la ligne la plus basse correspond au premier saut documenté, celle du haut au plus récent.

Une boucle se manifeste généralement par une séquence récurrente :

```text
Internet
  → Exchange Online Protection
  → Mailgateway
  → Exchange Online Protection
  → Mailgateway
  → Exchange Online Protection
  → ...
```

Chaque nom d’hôte Microsoft apparaissant plusieurs fois n’indique pas forcément une boucle. Exchange Online traite les messages en interne via plusieurs rôles de transport. Le retour répété entre les mêmes frontières administratives, par exemple entre Exchange Online et une passerelle locale, est révélateur. Les horodatages, l’adresse IP émettrice, l’hôte destinataire et `Message-ID` permettent d’identifier clairement le cycle.

Pour la première analyse, il faut répondre à ces questions :

1. Quel système a généré le NDR ?
2. Quels deux ou trois sauts se répètent ?
3. Quel système aurait dû remettre définitivement le message ?
4. Sur quelle décision liée au domaine, au destinataire, au connecteur ou à une règle a-t-il été transféré ?
5. Quelle modification a récemment influencé le flux de messagerie ?

## Diagnostic dans Exchange Online

Avec `Get-MessageTraceV2`, il est possible d’examiner le traitement des 90 derniers jours ; chaque requête est limitée à dix jours. Une fenêtre temporelle étroite et l’adresse exacte du destinataire fournissent les résultats les plus utiles :

```powershell
$start = (Get-Date).AddHours(-2)
$end = Get-Date
$recipient = "user01@contoso.com"

$trace = Get-MessageTraceV2 `
    -RecipientAddress $recipient `
    -StartDate $start `
    -EndDate $end `
    -ResultSize 5000

$trace |
    Select-Object Received,SenderAddress,RecipientAddress,Subject,
        Status,FromIP,ToIP,MessageTraceId,MessageId |
    Sort-Object Received
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-RecipientAddress` | Filtre la trace sur l’adresse de destinataire indiquée |
| `-StartDate` / `-EndDate` | Fenêtre temporelle de la requête ; chaque requête est limitée à dix jours |
| `-ResultSize 5000` | Nombre maximal d’entrées renvoyées |
| `Select-Object …` | Limite la sortie aux champs pertinents pour l’analyse de la boucle |
| `Sort-Object Received` | Trie chronologiquement les résultats selon l’heure de réception |

</details>

Les détails d’un résultat affichent les différents événements de transport :

```powershell
$trace | ForEach-Object {
    Get-MessageTraceDetailV2 `
        -MessageTraceId $_.MessageTraceId `
        -RecipientAddress $_.RecipientAddress
} | Format-Table Date,Event,Action,Detail -AutoSize
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-MessageTraceId` | ID de trace unique provenant du résultat de `Get-MessageTraceV2` |
| `-RecipientAddress` | Adresse de destinataire du résultat ; requise avec l’ID de trace pour la requête détaillée |
| `Format-Table … -AutoSize` | Adapte la largeur des colonnes au contenu afin que les détails des événements restent lisibles |

</details>

Ensuite, le domaine, le destinataire et les connecteurs sont relevés ensemble :

```powershell
Get-AcceptedDomain |
    Format-Table Name,DomainName,DomainType,MatchSubDomains -AutoSize

Get-EXORecipient -Identity $recipient |
    Format-List DisplayName,RecipientTypeDetails,PrimarySmtpAddress,
        ExternalEmailAddress,EmailAddresses

Get-OutboundConnector -IncludeTestModeConnectors |
    Format-List Name,Enabled,ConnectorType,RecipientDomains,SmartHosts,
        UseMXRecord,RouteAllMessagesViaOnPremises,TlsSettings

Get-InboundConnector |
    Format-List Name,Enabled,ConnectorType,SenderDomains,SenderIPAddresses,
        TlsSenderCertificateName,RequireTls,RestrictDomainsToIPAddresses,
        RestrictDomainsToCertificate
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Identity $recipient` | Sélectionne l’objet destinataire par adresse, alias ou nom |
| `-IncludeTestModeConnectors` | Inclut également les connecteurs en mode test dans la sortie |
| `Format-Table … -AutoSize` | Affichage tabulaire avec largeur de colonnes adaptée au contenu |
| `Format-List …` | Affichage en liste des propriétés indiquées, adapté aux valeurs longues telles que les listes d’adresses |

</details>

L’important n’est pas qu’un objet isolé semble plausible. Le type de domaine, le type réel de destinataire et le connecteur applicable doivent tous décrire la même destination.

## Diagnostic dans Exchange local

Dans un environnement hybride, le même destinataire est également vérifié localement. Les requêtes distinguent une véritable boîte aux lettres locale, une RemoteMailbox et un MailUser :

```powershell
Get-Recipient -Identity $recipient |
    Format-List DisplayName,RecipientType,RecipientTypeDetails,
        PrimarySmtpAddress,EmailAddresses

Get-Mailbox -Identity $recipient -ErrorAction SilentlyContinue |
    Format-List RecipientTypeDetails,ServerName,Database,PrimarySmtpAddress

Get-RemoteMailbox -Identity $recipient -ErrorAction SilentlyContinue |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,RemoteRoutingAddress

Get-MailUser -Identity $recipient -ErrorAction SilentlyContinue |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,ExternalEmailAddress
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Identity $recipient` | Sélectionne l’objet par adresse, alias ou nom |
| `-ErrorAction SilentlyContinue` | Supprime le message d’erreur si l’objet n’existe pas dans le type concerné ; la requête ne renvoie alors simplement aucun résultat |

</details>

Pour le chemin de transport, les connecteurs d’envoi et de réception ainsi que les journaux de suivi sont nécessaires :

```powershell
Get-SendConnector |
    Format-List Name,Enabled,AddressSpaces,DNSRoutingEnabled,SmartHosts,
        SourceTransportServers,CloudServicesMailEnabled,TlsDomain

Get-ReceiveConnector |
    Format-List Identity,Enabled,Bindings,RemoteIPRanges,PermissionGroups

$servers = Get-ExchangeServer |
    Where-Object { $_.IsMailboxServer -or $_.IsHubTransportServer }

$servers |
    Get-MessageTrackingLog `
        -Start $start `
        -End $end `
        -Recipients $recipient `
        -ResultSize Unlimited |
    Select-Object Timestamp,ServerHostname,ClientHostname,Source,EventId,
        ConnectorId,Sender,Recipients,MessageId,NetworkMessageId |
    Sort-Object Timestamp
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Where-Object { … }` | Limite la liste des serveurs aux serveurs Mailbox et Hub Transport, c’est-à-dire aux rôles disposant de journaux de suivi |
| `-Start` / `-End` | Fenêtre temporelle de la recherche dans les journaux |
| `-Recipients $recipient` | Filtre les événements de suivi correspondant à cette adresse de destinataire |
| `-ResultSize Unlimited` | Supprime la limite par défaut de 1000 entrées renvoyées |
| `Select-Object …` | Limite la sortie aux champs pertinents pour l’analyse du chemin |
| `Sort-Object Timestamp` | Trie chronologiquement les événements de tous les serveurs |

</details>

Un `SEND` vers Exchange Online, suivi d’un nouveau `RECEIVE` du même message depuis Exchange Online, rend le renvoi visible. Avec `MessageId` et `NetworkMessageId`, il est possible d’éviter de confondre différents messages de test.

## Les causes les plus fréquentes en un coup d’œil

| Schéma | Cause typique | Résolution |
| --- | --- | --- |
| Les destinataires inconnus oscillent entre deux systèmes | Le domaine accepté est défini sur `InternalRelay`, mais les deux côtés transfèrent les destinataires inconnus | Définir une responsabilité claire ; en cas de remise complète dans EXO, utiliser `Authoritative` ou, pour un domaine fractionné, définir un seul saut final |
| EXO envoie vers Exchange local, qui renvoie immédiatement vers EXO | Le connecteur hybride ou Centralized Mail Transport ne correspond plus à l’emplacement de la boîte aux lettres | Vérifier la configuration HCW et `RouteAllMessagesViaOnPremises` ; désactiver le routage central obsolète ou corriger la résolution locale des destinataires |
| Le message oscille entre EXO et une passerelle de sécurité, de signature ou de chiffrement | Les messages renvoyés satisfont à nouveau la règle de sortie | Utiliser comme exception l’en-tête défini par la passerelle ou le mécanisme documenté de prévention des boucles ; authentifier clairement les connecteurs entrants et sortants |
| Un seul destinataire est concerné | `targetAddress` obsolète ou incorrect, type RemoteMailbox erroné ou adresses proxy contradictoires | Déterminer la source d’autorité, corriger l’objet destinataire à cet endroit et synchroniser |
| Seuls les messages transférés bouclent | Une règle de transport, un transfert de boîte aux lettres ou une règle de boîte de réception réadresse le chemin d’origine | Désactiver la règle, corriger la cible et définir une exception robuste |
| Seul un sous-domaine ou une application est concerné | Le domaine parent ne couvre pas correctement le sous-domaine dans le chemin de connecteur attendu | Configurer explicitement le sous-domaine comme domaine accepté et dans le connecteur d’envoi approprié |
| Tous les messages bouclent après une modification de passerelle ou DNS | Le Smart Host ou l’enregistrement MX pointe vers l’entrée du système émetteur | Corriger le prochain saut et vérifier séparément les cibles DNS, NAT et d’équilibrage de charge |

## Cause 1 : type incorrect du domaine accepté

Un domaine authoritative signifie que tous les destinataires valides de ce domaine sont connus dans l’organisation Exchange ; les destinataires inconnus sont rejetés. Un domaine Internal Relay signifie qu’une partie des destinataires se trouve dans un autre système et doit être transférée via un connecteur d’envoi ou sortant.

La configuration problématique survient lorsque Exchange Online envoie des destinataires inconnus vers un système local et que celui-ci ne traite pas non plus ce même domaine de façon définitive, mais le renvoie à Exchange Online via MX ou Smart Host.

```powershell
Get-AcceptedDomain -Identity contoso.com |
    Format-List DomainName,DomainType,MatchSubDomains
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Identity contoso.com` | Sélectionne le domaine accepté à vérifier |
| `Format-List …` | Affiche le nom de domaine, le type de domaine et la couverture des sous-domaines sous forme de liste |

</details>

Si tous les destinataires se trouvent dans Exchange Online après la fin d’une migration, `Authoritative` est généralement l’état cible adéquat :

```powershell
# À exécuter uniquement après une vérification complète des destinataires et du routage.
Set-AcceptedDomain -Identity contoso.com -DomainType Authoritative
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Identity contoso.com` | Le domaine accepté à modifier |
| `-DomainType Authoritative` | Définit le domaine comme authoritative : les destinataires inconnus sont rejetés au lieu d’être transférés |

</details>

Pour un véritable domaine fractionné, `InternalRelay` peut être correct. Il faut alors disposer d’un connecteur clair vers le système qui connaît les destinataires restants. Cette destination ne doit pas renvoyer les adresses inconnues vers le point de départ.

## Cause 2 : connecteurs hybrides qui se chevauchent et Centralized Mail Transport

Centralized Mail Transport achemine délibérément les messages sortants d’Exchange Online via Exchange local. Cela est utile pour certaines exigences de conformité, mais crée des chemins de transport supplémentaires. Si l’option reste active après une migration alors que le système local renvoie les messages vers Exchange Online via son propre MX, un cycle peut se produire.

```powershell
Get-OutboundConnector -IncludeTestModeConnectors |
    Format-Table Name,Enabled,ConnectorType,RouteAllMessagesViaOnPremises,
        RecipientDomains,SmartHosts,UseMXRecord -AutoSize
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-IncludeTestModeConnectors` | Inclut également les connecteurs en mode test dans la sortie |
| `Format-Table … -AutoSize` | Affichage tabulaire des propriétés de routage avec largeur de colonnes adaptée au contenu |

</details>

Les connecteurs multiples dont la portée se chevauche doivent également être vérifiés. Microsoft recommande un connecteur local dédié pour le flux de messagerie hybride ; une réparation à l’aide du Hybrid Configuration Wizard est souvent plus sûre que des modifications isolées.

S’il est établi que Centralized Mail Transport n’est plus nécessaire, le paramètre peut être désactivé de manière ciblée :

```powershell
# Uniquement après vérification des exigences de conformité et de passerelle.
Set-OutboundConnector `
    -Identity "Outbound to On-Premises" `
    -RouteAllMessagesViaOnPremises:$false
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Identity "Outbound to On-Premises"` | Le connecteur sortant à modifier |
| `-RouteAllMessagesViaOnPremises:$false` | Désactive Centralized Mail Transport : les messages sortants d’Exchange Online ne transitent plus par Exchange local |

</details>

## Cause 3 : une passerelle traite à nouveau ses messages renvoyés

Dans un scénario entrant et sortant, Exchange Online envoie un message à un service supplémentaire pour signature, chiffrement ou archivage. Celui-ci le renvoie ensuite à Exchange Online. La règle de sortie doit reconnaître le message renvoyé ; sinon, il est à nouveau envoyé au service.

La vérification commence par toutes les règles qui sélectionnent des connecteurs, redirigent des destinataires ou évaluent des en-têtes :

```powershell
Get-TransportRule |
    Sort-Object Priority |
    Format-List Name,State,Mode,Priority,FromScope,SentToScope,
        RedirectMessageTo,RouteMessageOutboundConnector,
        SetHeaderName,SetHeaderValue,ExceptIfHeaderContainsMessageHeader,
        ExceptIfHeaderContainsWords
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Sort-Object Priority` | Trie les règles selon leur ordre d’évaluation |
| `Format-List …` | Affiche les propriétés qui sélectionnent des connecteurs, redirigent des destinataires ou définissent des en-têtes, ou les évaluent comme exceptions |

</details>

L’exception concrète doit suivre la documentation du fabricant de la passerelle. Il est courant d’utiliser un en-tête défini par le service, qui ne peut pas être falsifié de manière fiable depuis Internet. Les connecteurs entrants doivent en outre identifier le service par certificat ou adresse IP émettrice fixe. Une exception générale pour tous les messages paraissant « internes » est trop large.

## Cause 4 : l’objet destinataire et la boîte aux lettres réelle ne se trouvent pas au même endroit

Un objet peut apparaître dans Exchange Online comme `MailUser`, alors que la boîte aux lettres active se trouve localement. Dans un environnement hybride synchronisé, il ne s’agit pas automatiquement d’un doublon. De même, une `ExternalEmailAddress` correspondant à l’adresse SMTP principale ne prouve pas à elle seule une mauvaise configuration.

La combinaison de toutes les requêtes est déterminante :

- `Get-Mailbox` local renvoie un résultat : la boîte aux lettres active se trouve localement.
- `Get-RemoteMailbox` local renvoie un résultat : la destination gérée se trouve dans Exchange Online.
- `Get-EXOMailbox` renvoie un résultat : une véritable boîte aux lettres existe dans le cloud.
- `Get-EXORecipient` ne renvoie qu’un MailUser : l’objet est une destination de routage, pas une boîte aux lettres cloud.

Les objets obsolètes après une migration, les domaines de routage à distance incorrects ou les valeurs `targetAddress` définies manuellement dont le domaine renvoie par le même chemin de transport posent problème. Les modifications doivent être effectuées à la source d’autorité : dans les environnements synchronisés, avec les outils de gestion Exchange locaux et non en modifiant directement des attributs isolés dans Exchange Online.

## Cause 5 : les transferts et règles de transport forment un cercle

Une règle peut rediriger l’adresse A vers B, tandis que B est renvoyée vers A par une seconde règle, un transfert de boîte aux lettres ou un système externe. Ces boucles ne concernent souvent que certains destinataires ou types de messages.

```powershell
Get-TransportRule |
    Sort-Object Priority |
    Select-Object Name,State,Mode,Priority,RedirectMessageTo,
        BlindCopyTo,AddToRecipients,RouteMessageOutboundConnector

Get-Mailbox -ResultSize Unlimited |
    Select-Object DisplayName,PrimarySmtpAddress,
        ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward

Get-InboxRule -Mailbox user01@contoso.com |
    Select-Object Name,Enabled,Priority,ForwardTo,RedirectTo,ForwardAsAttachmentTo
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Sort-Object Priority` | Trie les règles de transport selon leur ordre d’évaluation |
| `-ResultSize Unlimited` | Supprime la limite par défaut de 1000 boîtes aux lettres renvoyées |
| `-Mailbox user01@contoso.com` | Boîte aux lettres dont les règles de boîte de réception sont interrogées |
| `Select-Object …` | Limite la sortie aux destinations de transfert et de redirection |

</details>

La résolution ne consiste pas seulement à désactiver temporairement une règle. Toute la chaîne doit être supprimée, et les règles destinées aux services externes nécessitent une exception qui reconnaît de manière fiable les messages déjà traités.

## Cause 6 : MX, Smart Host ou sous-domaine pointe en retour

Une passerelle peut nécessiter un prochain saut interne différent de celui utilisé par les expéditeurs externes. Si elle utilise simplement le MX public pour le transfert, celui-ci peut à son tour pointer vers la passerelle elle-même. Le même problème survient lorsqu’un Smart Host revient vers son propre écouteur via NAT ou équilibrage de charge.

```powershell
Resolve-DnsName -Type MX contoso.com
Resolve-DnsName -Type MX app.contoso.com

Get-SendConnector |
    Format-List Name,AddressSpaces,DNSRoutingEnabled,SmartHosts
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Type MX` | Interroge les enregistrements MX au lieu des enregistrements A par défaut |
| `contoso.com` / `app.contoso.com` | Domaine à interroger comme argument positionnel (paramètre `-Name`) |
| `Format-List …` | Affiche, pour chaque connecteur d’envoi, les espaces d’adresses, le mode de routage et les Smart Hosts |

</details>

Les sous-domaines méritent une vérification spécifique. Dans un article de support désormais retiré, Microsoft a documenté des cas où un sous-domaine d’application doit être créé explicitement comme domaine Internal Relay et synchronisé avec les systèmes Edge :

```powershell
New-AcceptedDomain `
    -Name "app.contoso.com" `
    -DomainName app.contoso.com `
    -DomainType InternalRelay

Start-EdgeSynchronization
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Name "app.contoso.com"` | Nom d’affichage du nouvel objet de domaine accepté |
| `-DomainName app.contoso.com` | Domaine SMTP pour lequel Exchange accepte les messages |
| `-DomainType InternalRelay` | Une partie des destinataires se trouve hors de l’organisation ; les destinataires inconnus sont transférés via un connecteur d’envoi au lieu d’être rejetés |

</details>

Ces commandes ne constituent pas une correction universelle. Elles ne conviennent que si `app.contoso.com` est effectivement remis en dehors de l’organisation Exchange et que le connecteur d’envoi possède un prochain saut sans ambiguïté.

## Procédure sûre en cas de boucle active

Pendant l’incident, il convient d’abord d’arrêter la multiplication des messages. Selon l’architecture, la règle de transport déclenchante ou le connecteur spécifique est désactivé de manière contrôlée, ou la passerelle retient la file d’attente concernée. La configuration et des exemples de messages sont exportés au préalable.

Il faut ensuite effectuer un test avec un seul expéditeur, un seul destinataire et une ligne d’objet clairement identifiable. Le message est suivi intégralement au moyen des en-têtes, de Message Trace et des journaux de suivi locaux. Le flux de messagerie n’est réactivé progressivement que lorsqu’il se termine à la destination prévue.

Sont déconseillés :

- augmenter les limites de sauts
- modifier plusieurs connecteurs simultanément
- basculer des domaines acceptés au hasard entre `Authoritative` et `InternalRelay`
- réinjecter à plusieurs reprises une file d’attente problématique sans vérification
- corriger directement dans AD ou Exchange Online des attributs Exchange synchronisés
- désactiver les vérifications TLS, IP ou de certificat comme prétendue solution rapide

## Contrôle final

Après la correction, la documentation doit comporter exactement une déclaration pour chaque domaine pertinent : quel système connaît le destinataire, quel connecteur est applicable et quel hôte est le prochain saut final ?

La réception technique comprend au minimum :

- un message de test externe et interne
- un destinataire inconnu du même domaine
- un destinataire de chaque côté d’un véritable domaine fractionné
- un message sortant avec une passerelle ou Centralized Mail Transport activé
- des en-têtes sans séquence de sauts récurrente
- Message Trace avec `Delivered` ou le transfert attendu
- un suivi local sans nouveau `RECEIVE` après un `SEND` vers la même destination
- la validation de tous les connecteurs encore nécessaires

Une boucle de messagerie n’est réellement résolue que lorsque non seulement le message de test arrive, mais aussi les destinataires inconnus et les chemins alternatifs de flux de messagerie se terminent comme prévu. C’est précisément là que surviennent la plupart des récidives.

## Historique des modifications

- **25 octobre 2026 :** Microsoft a retiré l’article de support sur le scénario de sous-domaine et EdgeSync ; la source renvoie désormais vers la version archivée du 19 avril 2026.

## Sources

1. [Microsoft Learn – Fix NDR error 5.4.6 or 5.4.14 in Exchange Online](https://learn.microsoft.com/en-us/troubleshoot/exchange/email-delivery/ndr/fix-error-code-5-4-6-through-5-4-20-in-exchange-online): signification des NDR Exchange et causes typiques dans les domaines acceptés et les connecteurs hybrides.

2. [Microsoft Learn – Manage accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains): différences entre `Authoritative` et `InternalRelay`.

3. [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains): responsabilité, domaines de relais et recherche de destinataires dans Exchange local.

4. [Microsoft Learn – Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing): chemins de transport attendus avec et sans Centralized Mail Transport.

5. [Microsoft Learn – Validate connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/validate-connectors): validation des connecteurs et indications concernant plusieurs connecteurs applicables simultanément.

6. [Microsoft Learn – Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud): modèles de flux de messagerie pris en charge avec des services tiers en amont.

7. [Microsoft Learn – Mail flow rules in Exchange Online](https://learn.microsoft.com/en-us/exchange/security-and-compliance/mail-flow-rules/mail-flow-rules): traitement, priorité, actions et exceptions des règles de transport.

8. [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps): recherche de messages dans le transport Exchange Online.

9. [Microsoft Learn – Search message tracking logs](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/search-message-tracking-logs): suivi local des messages sur tous les serveurs Exchange.

10. [Microsoft Learn – Hop count exceeded for an on-premises application subdomain (version archivée du 19 avril 2026)](https://web.archive.org/web/20260419203247/https://learn.microsoft.com/en-us/troubleshoot/exchange/mailflow/hop-count-exceeded-possible-mail-loop): scénario documenté de sous-domaine/EdgeSync avec domaine Internal Relay explicite ; Microsoft a depuis retiré l’article.
