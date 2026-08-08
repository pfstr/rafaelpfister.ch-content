---
title: "Causes typiques des boucles de messagerie et comment les résoudre"
navTitle: "Résoudre les boucles de messagerie"
description: "Comment identifier et résoudre systématiquement les boucles SMTP dans Exchange Online, les environnements hybrides et les passerelles de messagerie en amont à l’aide des NDR, en-têtes, traces de messages, objets destinataires et connecteurs."
date: "2026-08-07"
kategorie: "Exchange OnPrem / Hybrid"
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
url: https://rafaelpfister.ch/fr/blog/causes-typiques-des-boucles-de-messagerie-et-comment-les-resoudre
translationSourceHash: 5353684681217adafc789a3b28ec218fa927e18d801c82c437ae281e1e1017bd
translationModel: gpt-5.6-terra
translatedAt: 2026-08-08T13:50:00.318Z
translationReview: automatic
---

# Causes typiques des boucles de messagerie et comment les résoudre

Une boucle de messagerie survient lorsqu’au moins deux systèmes de transport se transmettent sans cesse le même message. Aucun des systèmes ne se reconnaît comme destination finale, mais tous deux connaissent un prochain saut apparemment approprié. La boucle ne prend fin que lorsqu’un serveur constate que le nombre autorisé d’étapes de transport a été dépassé et génère un NDR.

Avec Exchange, deux messages sont particulièrement révélateurs :

- `554 5.4.6 Hop count exceeded - possible mail loop` est généralement généré par l’Exchange local.
- `554 5.4.14 Hop count exceeded - possible mail loop ATTR34` est généré par Exchange Online.

La limite de sauts n’est pas la cause, mais une protection contre une répétition infinie. L’augmenter ne résout donc rien. Il faut identifier le point où le message est renvoyé vers un système déjà traversé, contrairement à l’architecture cible.

## Identifier le schéma de boucle dans l’en-tête

Le NDR et les en-têtes complets du message d’origine doivent être sauvegardés avant toute modification. Les lignes `Received` se lisent de bas en haut : la ligne la plus basse correspond au premier saut documenté, celle du haut au plus récent.

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

La présence répétée d’un nom d’hôte Microsoft ne constitue pas à elle seule une boucle. Exchange Online traite les messages en interne via plusieurs rôles de transport. Ce qui est suspect, c’est le retour répété entre les mêmes frontières administratives, par exemple entre Exchange Online et une passerelle locale. Les horodatages, l’adresse IP émettrice, l’hôte destinataire et `Message-ID` permettent d’identifier clairement le cycle.

Pour la première analyse, répondez à ces questions :

1. Quel système a généré le NDR ?
2. Quels sont les deux ou trois sauts qui se répètent ?
3. Quel système aurait dû livrer le message définitivement ?
4. Sur la base de quelle décision de domaine, de destinataire, de connecteur ou de règle le message a-t-il été transféré ?
5. Quelle modification a récemment influencé le flux de messagerie ?

## Diagnostic dans Exchange Online

Avec `Get-MessageTraceV2`, il est possible d’examiner le traitement des 90 derniers jours ; chaque requête est limitée à dix jours. Une plage temporelle restreinte et l’adresse précise du destinataire donnent les résultats les plus utiles :

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

Les détails d’un résultat affichent les différents événements de transport :

```powershell
$trace | ForEach-Object {
    Get-MessageTraceDetailV2 `
        -MessageTraceId $_.MessageTraceId `
        -RecipientAddress $_.RecipientAddress
} | Format-Table Date,Event,Action,Detail -AutoSize
```

Ensuite, le domaine, le destinataire et les connecteurs sont relevés conjointement :

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

L’essentiel n’est pas qu’un objet isolé semble plausible. Le type de domaine, le type réel de destinataire et le connecteur applicable doivent tous décrire la même destination.

## Diagnostic dans Exchange local

Dans un environnement hybride, le même destinataire est également vérifié localement. Les requêtes font la distinction entre une véritable boîte aux lettres locale, une RemoteMailbox et un MailUser :

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

Un `SEND` vers Exchange Online, suivi d’un nouveau `RECEIVE` du même message depuis Exchange Online, rend le renvoi visible. `MessageId` et `NetworkMessageId` permettent d’éviter de confondre différents messages de test.

## Les causes les plus fréquentes en bref

| Schéma | Cause typique | Résolution |
| --- | --- | --- |
| Des destinataires inconnus oscillent entre deux systèmes | Le domaine accepté est défini sur `InternalRelay`, mais les deux côtés transfèrent les destinataires inconnus | Définir une responsabilité claire ; pour une livraison entièrement dans EXO, utiliser `Authoritative`, ou, pour un domaine fractionné, définir un unique saut final |
| EXO envoie vers l’Exchange local, qui renvoie immédiatement vers EXO | Le connecteur hybride ou Centralized Mail Transport ne correspond plus à l’emplacement de la boîte aux lettres | Vérifier la configuration HCW et `RouteAllMessagesViaOnPremises` ; désactiver le routage central obsolète ou corriger la résolution locale des destinataires |
| Le message oscille entre EXO et une passerelle de sécurité, de signature ou de chiffrement | Les messages de retour satisfont à nouveau la règle de sortie | Utiliser comme exception l’en-tête défini par la passerelle ou le mécanisme documenté de prévention des boucles ; authentifier clairement les connecteurs entrants et sortants |
| Un seul destinataire est concerné | `targetAddress` obsolète ou erroné, type RemoteMailbox incorrect ou adresses proxy contradictoires | Déterminer la source d’autorité, corriger l’objet destinataire à cet endroit et le synchroniser |
| Seuls les messages transférés sont concernés | Une règle de transport, un transfert de boîte aux lettres ou une règle de boîte de réception cible à nouveau le chemin d’origine | Désactiver la règle, corriger la cible et définir une exception robuste |
| Seul un sous-domaine ou une application est concerné | Le domaine parent ne couvre pas correctement le sous-domaine dans le chemin de connecteur attendu | Configurer explicitement le sous-domaine comme domaine accepté et dans le connecteur d’envoi approprié |
| Tous les messages bouclent après une modification de passerelle ou de DNS | L’hôte intelligent ou le MX pointe vers l’entrée du système émetteur | Corriger le prochain saut et vérifier séparément les cibles DNS, NAT et d’équilibrage de charge |

## Cause 1 : type incorrect du domaine accepté

Un domaine authoritative signifie que tous les destinataires valides de ce domaine sont connus de l’organisation Exchange ; les destinataires inconnus sont rejetés. Un domaine Internal Relay signifie qu’une partie des destinataires se trouve dans un autre système et doit être transférée via un connecteur d’envoi ou sortant.

La configuration problématique survient lorsque Exchange Online envoie des destinataires inconnus à un système local et que celui-ci ne traite pas non plus ce domaine de manière définitive, mais le renvoie à Exchange Online par MX ou hôte intelligent.

```powershell
Get-AcceptedDomain -Identity contoso.com |
    Format-List DomainName,DomainType,MatchSubDomains
```

Lorsque tous les destinataires se trouvent dans Exchange Online après une migration terminée, `Authoritative` est généralement l’état cible approprié :

```powershell
# Exécuter uniquement après une vérification complète des destinataires et du routage.
Set-AcceptedDomain -Identity contoso.com -DomainType Authoritative
```

Pour un véritable domaine fractionné, `InternalRelay` peut être correct. Il faut toutefois disposer d’un connecteur clair vers le système qui connaît les destinataires restants. Cette destination ne doit pas renvoyer les adresses inconnues vers le point de départ.

## Cause 2 : connecteurs hybrides qui se chevauchent et Centralized Mail Transport

Centralized Mail Transport achemine délibérément les messages sortants d’Exchange Online via l’Exchange local. Cela est utile pour certaines exigences de conformité, mais crée des chemins de transport supplémentaires. Si l’option reste active après une migration alors que le système local renvoie les messages à Exchange Online via son propre MX, un cercle peut se former.

```powershell
Get-OutboundConnector -IncludeTestModeConnectors |
    Format-Table Name,Enabled,ConnectorType,RouteAllMessagesViaOnPremises,
        RecipientDomains,SmartHosts,UseMXRecord -AutoSize
```

Plusieurs connecteurs aux portées qui se chevauchent sont également suspects. Microsoft recommande un connecteur local dédié pour le flux de messagerie hybride ; une réparation à l’aide du Hybrid Configuration Wizard est souvent plus sûre que des modifications isolées.

S’il est établi que Centralized Mail Transport n’est plus nécessaire, le paramètre peut être désactivé de manière ciblée :

```powershell
# Uniquement après vérification des exigences de conformité et de passerelle.
Set-OutboundConnector `
    -Identity "Outbound to On-Premises" `
    -RouteAllMessagesViaOnPremises:$false
```

## Cause 3 : une passerelle traite à nouveau ses messages de retour

Dans un scénario In-and-out, Exchange Online envoie un message à un service supplémentaire pour signature, chiffrement ou archivage. Ce dernier le renvoie ensuite à Exchange Online. La règle de sortie doit reconnaître le message de retour ; sinon, il est de nouveau envoyé au service.

La vérification commence par toutes les règles qui sélectionnent des connecteurs, redirigent des destinataires ou évaluent des en-têtes :

```powershell
Get-TransportRule |
    Sort-Object Priority |
    Format-List Name,State,Mode,Priority,FromScope,SentToScope,
        RedirectMessageTo,RouteMessageOutboundConnector,
        SetHeaderName,SetHeaderValue,ExceptIfHeaderContainsMessageHeader,
        ExceptIfHeaderContainsWords
```

L’exception concrète doit suivre la documentation du fabricant de la passerelle. Il est courant d’utiliser un en-tête défini par le service, qui ne peut pas être falsifié de manière fiable depuis Internet. Les connecteurs entrants doivent également identifier le service par certificat ou adresse IP expéditrice fixe. Une exception générale pour tous les messages semblant « internes » est trop large.

## Cause 4 : l’objet destinataire et la boîte aux lettres réelle ne se trouvent pas au même endroit

Un objet peut apparaître dans Exchange Online comme `MailUser`, alors que la boîte aux lettres active se trouve localement. Dans un environnement hybride synchronisé, cela n’est pas automatiquement un doublon. De même, une `ExternalEmailAddress`, correspondant à l’adresse SMTP principale, ne prouve pas à elle seule une mauvaise configuration.

La combinaison de toutes les requêtes est déterminante :

- `Get-Mailbox` local renvoie un résultat : la boîte aux lettres active se trouve localement.
- `Get-RemoteMailbox` local renvoie un résultat : la cible gérée se trouve dans Exchange Online.
- `Get-EXOMailbox` renvoie un résultat : une véritable boîte aux lettres existe dans le cloud.
- `Get-EXORecipient` ne renvoie qu’un MailUser : l’objet est une cible de routage, pas une boîte aux lettres cloud.

Les objets obsolètes après une migration, des domaines de routage distant incorrects ou des valeurs `targetAddress` définies manuellement, dont le domaine ramène par le même chemin de transport, posent problème. Les modifications sont effectuées à la source d’autorité : dans les environnements synchronisés, avec les outils de gestion Exchange locaux, et non en modifiant directement des attributs isolés dans Exchange Online.

## Cause 5 : les transferts et règles de transport forment un cercle

Une règle peut rediriger de l’adresse A vers B, tandis que B renvoie vers A par une seconde règle, un transfert de boîte aux lettres ou un système externe. Ces boucles ne concernent souvent que certains destinataires ou types de messages.

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

La résolution ne consiste pas seulement à désactiver temporairement une règle. La chaîne complète doit être supprimée, et les règles destinées à des services externes nécessitent une exception qui reconnaît de manière fiable les messages déjà traités.

## Cause 6 : le MX, l’hôte intelligent ou le sous-domaine pointe en retour

Une passerelle peut nécessiter en interne un prochain saut différent de celui des expéditeurs externes. Si elle utilise simplement le MX public pour le transfert, celui-ci peut à son tour pointer vers la passerelle elle-même. Le même problème survient lorsqu’un hôte intelligent renvoie vers son propre écouteur via NAT ou équilibrage de charge.

```powershell
Resolve-DnsName -Type MX contoso.com
Resolve-DnsName -Type MX app.contoso.com

Get-SendConnector |
    Format-List Name,AddressSpaces,DNSRoutingEnabled,SmartHosts
```

Les sous-domaines méritent une vérification spécifique. Microsoft documente des cas dans lesquels un sous-domaine d’application doit être créé explicitement comme domaine Internal Relay et synchronisé avec les systèmes Edge :

```powershell
New-AcceptedDomain `
    -Name "app.contoso.com" `
    -DomainName app.contoso.com `
    -DomainType InternalRelay

Start-EdgeSynchronization
```

Ces commandes ne constituent pas une correction universelle. Elles ne conviennent que si `app.contoso.com` est effectivement livré en dehors de l’organisation Exchange et que le connecteur d’envoi possède un prochain saut clairement défini.

## Procédure sûre en cas de boucle active

Pendant l’incident, il faut d’abord arrêter la multiplication des messages. Selon l’architecture, la règle de transport déclenchante ou le connecteur spécifique est désactivé de manière contrôlée, ou la passerelle retient la file d’attente concernée. La configuration et des exemples de messages sont exportés au préalable.

Ensuite, effectuez un test avec exactement un expéditeur, un destinataire et un objet facilement identifiable. Le message est suivi sans interruption à l’aide des en-têtes, de Message Trace et des journaux de suivi locaux. Le flux de messagerie n’est rouvert progressivement que lorsqu’il se termine à la destination prévue.

À éviter :

- augmenter les limites de sauts
- modifier plusieurs connecteurs simultanément
- basculer par hypothèse les domaines acceptés entre `Authoritative` et `InternalRelay`
- réinjecter à plusieurs reprises une file d’attente problématique sans vérification
- corriger directement dans AD ou Exchange Online des attributs Exchange synchronisés
- désactiver les vérifications TLS, IP ou de certificat comme prétendue correction rapide

## Contrôle final

Après la correction, la documentation doit contenir exactement une information pour chaque domaine pertinent : quel système connaît le destinataire, quel connecteur s’applique et quel hôte est le prochain saut final ?

La validation technique comprend au minimum :

- un message de test externe et interne
- un destinataire inconnu du même domaine
- un destinataire de chaque côté d’un véritable domaine fractionné
- un message sortant avec passerelle ou Centralized Mail Transport activé
- des en-têtes sans séquence de sauts récurrente
- Message Trace avec `Delivered` ou la remise attendue
- un suivi local sans nouveau `RECEIVE` après un `SEND` vers la même destination
- la validation des connecteurs pour tous les connecteurs encore nécessaires

Une boucle de messagerie résolue n’est réellement terminée que lorsque non seulement le message de test arrive, mais aussi lorsque les destinataires inconnus et les chemins alternatifs de flux de messagerie se terminent de manière définie. C’est précisément là que surviennent la plupart des récidives.

## Sources

1. [Microsoft Learn – Fix NDR error 5.4.6 or 5.4.14 in Exchange Online](https://learn.microsoft.com/en-us/troubleshoot/exchange/email-delivery/ndr/fix-error-code-5-4-6-through-5-4-20-in-exchange-online): signification des NDR Exchange et causes typiques dans les domaines acceptés et les connecteurs hybrides.

2. [Microsoft Learn – Manage accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains): différences entre `Authoritative` et `InternalRelay`.

3. [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains): responsabilité, domaines de relais et recherche de destinataires dans Exchange local.

4. [Microsoft Learn – Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing): chemins de transport attendus avec et sans Centralized Mail Transport.

5. [Microsoft Learn – Validate connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/validate-connectors): validation des connecteurs et indications concernant plusieurs connecteurs applicables simultanément.

6. [Microsoft Learn – Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud): schémas de flux de messagerie pris en charge avec des services tiers en amont.

7. [Microsoft Learn – Mail flow rules in Exchange Online](https://learn.microsoft.com/en-us/exchange/security-and-compliance/mail-flow-rules/mail-flow-rules): traitement, priorité, actions et exceptions des règles de transport.

8. [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps): recherche de messages dans le transport Exchange Online.

9. [Microsoft Learn – Search message tracking logs](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/search-message-tracking-logs): suivi local des messages sur tous les serveurs Exchange.

10. [Microsoft Learn – Hop count exceeded for an on-premises application subdomain](https://learn.microsoft.com/en-us/troubleshoot/exchange/mailflow/hop-count-exceeded-possible-mail-loop): scénario documenté de sous-domaine/EdgeSync avec domaine Internal Relay explicite.
