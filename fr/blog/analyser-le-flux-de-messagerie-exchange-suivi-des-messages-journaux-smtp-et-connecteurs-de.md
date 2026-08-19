---
title: "Analyser le flux de messagerie Exchange : suivi des messages, journaux SMTP et connecteurs de réception"
navTitle: "Analyser le flux de messagerie"
description: "Comment déterminer systématiquement, dans Exchange OnPrem, Hybrid et Exchange Online, où un message est resté bloqué : requêtes avec exemples de sortie, lecture correcte du journal SMTP et pièges qui mènent régulièrement sur de fausses pistes."
date: "2026-08-11"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "22 min de lecture"
themen:
  - exchange-onprem-hybrid
  - smtp-mailflow
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
  - "exchange-on-premises"
  - "exchange-hybrid"
  - "exchange-online"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
related:
  - typische-ursachen-fuer-mail-loops-und-deren-behebung
  - einliefernde-ip-adressen-aggregieren
  - mailflow-fehlersuche-kontrollierte-experimente
slug: "analyser-le-flux-de-messagerie-exchange-suivi-des-messages-journaux-smtp-et-connecteurs-de"
translationId: "article-ad93c41ab6cd20e6"
draft: false
translationOf: exchange-message-tracking-und-receive-connectoren-analysieren
url: https://rafaelpfister.ch/fr/blog/analyser-le-flux-de-messagerie-exchange-suivi-des-messages-journaux-smtp-et-connecteurs-de
translationSourceHash: 646cb713e4dd97300a2cd068ee8f04953f2e80a99aec63ed11eddc46e1981f13
translationModel: gpt-5.6-terra
translatedAt: 2026-08-12T05:15:30.413Z
translationReview: automatic
---

# Analyser le flux de messagerie Exchange : suivi des messages, journaux SMTP et connecteurs de réception

La question la plus fréquente dans l’exploitation de la messagerie est la suivante : un message n’est pas arrivé, où est-il passé ? Le suivi des messages répond à cette question de manière fiable, mais uniquement si vous savez ce qu’il ne vous indique **pas**. Cet article décrit une méthode dans l’ordre qui a fait ses preuves, montre la sortie typique de chaque requête et présente les pièges qui font régulièrement perdre des heures en suggérant des conclusions plausibles, mais erronées.

Tous les exemples utilisent des noms génériques : `SRV-MAIL01` et `SRV-MAIL02` comme serveurs de transport, `example.com` comme domaine. Si vous souhaitez composer les commandes pour votre environnement plutôt que de les saisir : le [générateur de commandes](https://rafaelpfister.ch/tools/command-builder) contient les commandes courantes de suivi des messages et de capture pour PowerShell et shell Unix, côte à côte et entièrement localement dans le navigateur.

## Le principe : localiser d’abord, expliquer ensuite

Le réflexe consiste à rechercher immédiatement la cause. Il est plus efficace de déterminer d’abord jusqu’où le message est réellement arrivé. Cela réduit considérablement l’espace de recherche en une seule étape, car vous savez alors si vous devez chercher dans votre propre système, auprès de la passerelle en amont ou chez la destination.

L’ordre est donc le suivant : trouver le message, lire le dernier événement, lire la raison de l’erreur, déterminer s’il s’agit d’un cas isolé ou d’un schéma récurrent, puis seulement reconstruire le chemin de soumission.

## Étape 1 : trouver le message

Commencez par le destinataire, car c’est presque toujours ce que vous connaissez. Il est important d’exécuter la requête sur **tous** les serveurs de transport, et pas sur un seul.

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object {
    Get-MessageTrackingLog -Server $_ `
        -Start (Get-Date).AddHours(-6) `
        -ResultSize Unlimited `
        -Recipients "empfaenger@example.com"
} | Sort-Object Timestamp |
    Format-Table Timestamp, ServerHostname, EventId, Source, ConnectorId, MessageId `
        -AutoSize -Wrap
```

Sortie typique pour un message correctement traité :

```text
Timestamp           ServerHostname EventId      Source  ConnectorId
---------           -------------- -------      ------  -----------
11.08.2026 08:27:15 SRV-MAIL02     HARECEIVE    SMTP
11.08.2026 08:27:15 SRV-MAIL01     RECEIVE      SMTP    SRV-MAIL01\Default SRV-MAIL01
11.08.2026 08:27:15 SRV-MAIL01     HAREDIRECT   SMTP
11.08.2026 08:27:15 SRV-MAIL01     RESOLVE      ROUTING
11.08.2026 08:27:15 SRV-MAIL01     AGENTINFO    AGENT
11.08.2026 08:27:16 SRV-MAIL01     SENDEXTERNAL SMTP    Outbound-to-O365
11.08.2026 08:27:53 SRV-MAIL02     HADISCARD    SMTP
```

Si la requête ne trouve rien, vérifiez si le destinataire a été développé via une liste de distribution. Dans ce cas, il vaut mieux chercher par expéditeur :

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object {
    Get-MessageTrackingLog -Server $_ `
        -Start (Get-Date).AddHours(-6) `
        -ResultSize Unlimited
} | Where-Object { $_.Sender -like "*@example.com" } |
    Sort-Object Timestamp |
    Format-Table Timestamp, EventId, Sender,
        @{n='To'; e={$_.Recipients -join ','}}, MessageSubject -AutoSize -Wrap
```

## Étape 2 : lire le dernier événement

Tout le diagnostic repose sur le **dernier** `EventId` du message. Il vous indique quel espace de recherche traiter ensuite.

| Dernier EventId | Signification | Étape suivante |
|---|---|---|
| `RECEIVE`, puis plus rien | le message est bloqué | vérifier les files d’attente |
| `SEND` ou `SENDEXTERNAL` | remis avec succès | poursuivre la recherche au saut suivant |
| `FAIL` | échec définitif | lire la raison dans `RecipientStatus` |
| `DEFER` | nouvelle tentative en cours | vérifier la file d’attente et le système cible |
| `DROP` ou `POISONMESSAGE` | rejeté | vérifier une règle de transport ou un agent |
| `DELIVER` | remis dans une boîte aux lettres locale | vérifier les règles de boîte aux lettres |
| `RESOLVE` | destinataire réécrit | lire l’adresse cible dans l’entrée |

`RESOLVE` est l’étape intermédiaire la plus révélatrice dans les environnements Hybrid, car elle rend visible la réécriture vers l’adresse de routage cloud :

```text
EventId : RESOLVE
Source  : ROUTING
Sender  : absender@example.com
To      : BENUTZER@example.mail.onmicrosoft.com
```

Si l’adresse `onmicrosoft.com` attendue apparaît, l’objet destinataire est correctement configuré et vous pouvez écarter cette piste. Si l’adresse d’origine apparaît toujours, l’adresse cible manque sur l’objet local et Exchange tente une remise locale.

Si le message est bloqué, la file d’attente indique généralement la raison en clair :

```powershell
Get-Queue -Server SRV-MAIL01 |
    Where-Object { $_.MessageCount -gt 0 } |
    Format-Table Identity, DeliveryType, Status, MessageCount, NextHopDomain, LastError `
        -AutoSize -Wrap
```

## Piège 1 : le suivi est lié au serveur et de nombreuses entrées sont des copies fantômes

Si la sortie présente des paires de `HARECEIVE` et `HADISCARD`, souvent avec l’ajout `ExplicitlyDiscarded`, ce serveur n’a **pas traité** le message. Il détenait seulement une copie fantôme dans le cadre de la redondance fantôme, tandis qu’un autre serveur assurait la remise effective. Dès que le serveur principal signale la réussite, le partenaire rejette sa copie.

Voici ce que cela donne lorsque vous n’interrogez que le mauvais serveur :

```text
Timestamp           EventId    SourceContext
---------           -------    -------------
11.08.2026 09:47:13 HARECEIVE  1a2aae6b-f3a3-4e04-9233-7de460b92223
11.08.2026 09:48:07 HADISCARD  ExplicitlyDiscarded;1a2aae6b-f3a3-4e04-9233-7de460b92223
```

Deux lignes, aucune erreur, aucune remise. En conclure que le message a disparu revient à chercher au mauvais endroit. Le traitement réel apparaît dans le suivi du serveur partenaire.

Concrètement, cela signifie deux choses. Premièrement, ces lignes ne signalent pas un problème, mais un fonctionnement normal. Deuxièmement, vous devez impérativement interroger tous les serveurs de transport.

## Piège 2 : `Format-Table` masque précisément les colonnes déterminantes

La raison de l’erreur figure dans `RecipientStatus`, et ce champ est long. Dans un tableau, il disparaît complètement ou est tronqué. Cela conduit précisément à voir `FAIL` sans en voir la raison, puis à commencer à deviner.

Dès que vous avez trouvé un cas d’erreur, passez donc à `Format-List` et développez les champs collectifs :

```powershell
Get-MessageTrackingLog -Server SRV-MAIL01 `
    -Start (Get-Date).AddHours(-6) `
    -ResultSize Unlimited `
    -Recipients "empfaenger@example.com" `
    -EventId FAIL |
  Format-List Timestamp, Sender,
    @{n='To';     e={$_.Recipients -join ','}},
    @{n='Status'; e={$_.RecipientStatus -join ' | '}},
    MessageSubject, MessageId, SourceContext
```

Voici la différence. D’abord la vue en tableau, qui n’explique rien :

```text
Timestamp           EventId ConnectorId
---------           ------- -----------
11.08.2026 09:47:13 FAIL    Outbound-to-O365
```

Puis le même message sous forme de liste :

```text
Timestamp      : 11.08.2026 09:47:13
Sender         : dienst@example-test.com
To             : BENUTZER@example.mail.onmicrosoft.com
Status         : [{LED=550 5.1.8 Access denied, bad outbound sender AS(42000001)
                 [XX1PEPF00000000.eurprd02.prod.outlook.com]};{MSG=};
                 {FQDN=10.0.0.40};{IP=10.0.0.40};{LRT=11.08.2026 09:47:13}]
MessageSubject : Statusmeldung Nachtlauf
MessageId      : <1897281176.1319@app01.intern.example.com>
```

Le diagnostic est alors établi sans avoir eu besoin de la moindre supposition : le système distant conteste l’expéditeur. `LED` contient la réponse SMTP complète, `FQDN` et `IP` désignent le système ayant répondu, et `LRT` indique l’heure de la dernière tentative.

## Étape 3 : cas isolé ou schéma récurrent ?

Avant d’approfondir un cas particulier, déterminez son étendue. Cette seule requête permet de savoir s’il s’agit d’une simple note de bas de page ou d’un incident :

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object {
    Get-MessageTrackingLog -Server $_ `
        -Start (Get-Date).AddHours(-8) `
        -EventId FAIL -ResultSize Unlimited
} | Where-Object { ($_.RecipientStatus -join '') -like "*5.1.8*" } |
    Group-Object { ($_.Sender -split '@')[-1] } |
    Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

Remplacez `5.1.8` par le code d’état que vous examinez. La sortie répond à la question en une ligne :

```text
Count Name
----- ----
    9 example-test.com
```

Un seul domaine expéditeur signifie un problème très circonscrit, pas un incident ; vous pouvez poursuivre sereinement vos recherches. S’il y avait vingt domaines différents, vous auriez une panne en cours et tout le reste devrait attendre. Faire cette distinction si tôt est, d’expérience, ce qui fait gagner le plus de temps.

## Piège 3 : `ConnectorId` ne révèle pas le véritable connecteur de réception

C’est le piège le plus coûteux, car la sortie paraît sérieuse. Le courrier soumis par un client ou un système tiers sur le port 25 atteint d’abord le **Front End Transport**. Celui-ci transmet le message au **Transport Service** sur le port 2525. Le suivi des messages n’est écrit qu’à cet endroit ; le Front End Transport n’écrit pas son propre suivi.

La conséquence apparaît dans cette ligne :

```text
EventId        : RECEIVE
ConnectorId    : SRV-MAIL01\Default SRV-MAIL01
ClientIp       : 10.0.1.11
ClientHostname : srv-mail01.intern.example.com
```

La `ConnectorId` indique le connecteur interne sur le port 2525, et la `ClientIp` est l’adresse du **serveur proxy**, non celle du soumetteur d’origine. Le suivi ne contient tout simplement pas le connecteur configuré sur le port 25 réellement atteint par un système. Quiconque se fie à cette indication cherche l’erreur sur un connecteur qui n’est pas impliqué.

Il existe deux moyens d’obtenir la réponse. Le premier consiste à reconstruire la situation à partir de la configuration :

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object { Get-ReceiveConnector -Server $_ } |
    Format-List Identity, Enabled,
        @{n='Bindings';       e={$_.Bindings -join ','}},
        @{n='RemoteIPRanges'; e={$_.RemoteIPRanges -join ','}},
        PermissionGroups, AuthMechanism
```

```text
Identity         : SRV-MAIL01\Default Frontend SRV-MAIL01
Bindings         : 10.0.1.11:25
RemoteIPRanges   : 0.0.0.0-255.255.255.255
PermissionGroups : AnonymousUsers, ExchangeServers, ExchangeLegacyServers
AuthMechanism    : Tls, Integrated, BasicAuth, BasicAuthRequireTLS, ExchangeServer

Identity         : SRV-MAIL01\smtp-noauth SRV-MAIL01
Bindings         : 10.0.1.13:25
RemoteIPRanges   : 10.0.20.22,10.0.21.11,10.0.21.12
PermissionGroups : AnonymousUsers, Custom
AuthMechanism    : Tls
```

Déterminez l’IP source du système soumettant le message et recherchez le connecteur dont `RemoteIPRanges` la contient. Si elle ne correspond à aucun des connecteurs restreints, il reste le connecteur frontal par défaut, qui accepte généralement tout l’espace d’adressage. Utilisez ici aussi `Format-List`, car `RemoteIPRanges` et `PermissionGroups` sont régulièrement tronqués dans les tableaux.

La deuxième voie est le journal SMTP, qui mérite sa propre section.

## Le journal SMTP : le seul endroit qui contient toute la vérité

Le journal du Front End Transport enregistre la session SMTP complète : quel connecteur a été contacté, quelle IP s’est connectée, et ce que le client et le serveur se sont dit. C’est la seule source qui résout le piège décrit ci-dessus.

### Activer la journalisation

Par défaut, la journalisation est **désactivée** sur la plupart des connecteurs. Vous l’activez par connecteur :

```powershell
Set-ReceiveConnector -Identity "SRV-MAIL01\Default Frontend SRV-MAIL01" `
    -ProtocolLoggingLevel Verbose
```

Pour les connexions sortantes, utilisez `Set-SendConnector`. N’oubliez pas de remettre la valeur sur `None` après l’analyse, car une journalisation détaillée consomme de l’espace disque et écrit des volumes considérables en cas de trafic élevé.

### Emplacement des fichiers

Exchange sépare les journaux par service et par direction. Il n’est pas nécessaire de coder les chemins en dur ; interrogez-les :

```powershell
Get-FrontendTransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath,
        ReceiveProtocolLogMaxAge, ReceiveProtocolLogMaxDirectorySize

Get-TransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath
```

Ils se trouvent généralement sous le chemin d’installation, dans `TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive` pour le Front End Transport et dans `TransportRoles\Logs\Hub\ProtocolLog\SmtpReceive` pour le Transport Service. **C’est le point essentiel :** les connexions clientes sur le port 25 se trouvent exclusivement dans le chemin `FrontEnd`, tandis que le chemin `Hub` ne contient que le trafic de transmission interne sur 2525.

Tenez compte de la rétention. `ReceiveProtocolLogMaxAge` est souvent réglé sur 30 jours, tandis que `ReceiveProtocolLogMaxDirectorySize` limite en plus l’espace utilisé. En cas de trafic élevé, la limite de taille entre en jeu bien avant la limite d’âge, et vos journaux ne remontent alors plus qu’à quelques jours.

### Comprendre le format

Les fichiers sont des CSV avec des lignes d’en-tête commençant par `#`. Les colonnes les plus importantes sont `date-time`, `connector-id`, `session-id`, `sequence-number`, `local-endpoint`, `remote-endpoint`, `event` et `data`.

La colonne déterminante est `event`, un caractère unique :

| Caractère | Signification |
|---|---|
| `+` | connexion établie |
| `-` | connexion terminée |
| `>` | le serveur envoie au client |
| `<` | le client envoie au serveur |
| `*` | information du serveur, pas de trafic SMTP |

Une session se reconnaît à sa `session-id` commune ; `sequence-number` indique l’ordre au sein de la session. Voici un extrait typique :

```text
2026-08-11T09:47:10.4Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,0,
  10.0.1.13:25,10.0.20.22:51244,+,,
2026-08-11T09:47:10.4Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,1,
  10.0.1.13:25,10.0.20.22:51244,>,"220 srv-mail01.intern.example.com Microsoft ESMTP",
2026-08-11T09:47:10.5Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,2,
  10.0.1.13:25,10.0.20.22:51244,<,EHLO app01.intern.example.com,
2026-08-11T09:47:10.6Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,6,
  10.0.1.13:25,10.0.20.22:51244,<,MAIL FROM:<dienst@example-test.com>,
2026-08-11T09:47:10.7Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,8,
  10.0.1.13:25,10.0.20.22:51244,>,"250 2.1.5 Recipient OK",
```

Tout ce qui manquait au suivi des messages apparaît ici : le **véritable** connecteur (`smtp-noauth`), la **véritable** IP source (`10.0.20.22`) et le nom sous lequel le système s’annonce dans `EHLO`.

### Rechercher de manière ciblée

Pour les cas isolés, un filtre textuel est nettement plus rapide qu’une analyse d’objets. Recherchez l’adresse de l’expéditeur ou le nom `EHLO` et obtenez l’identifiant de session :

```powershell
$pfad = (Get-FrontendTransportService SRV-MAIL01).ReceiveProtocolLogPath
Select-String -Path "$pfad\*.log" -Pattern "dienst@example-test.com" -SimpleMatch |
    Select-Object -First 5 Line
```

Avec la `session-id` trouvée, récupérez la session complète :

```powershell
Select-String -Path "$pfad\*.log" -Pattern "08DEF44EC454A414" -SimpleMatch |
    ForEach-Object { $_.Line } | Select-Object -First 40
```

Si vous voulez seulement savoir quels connecteurs voient effectivement du trafic, comptez les ouvertures de connexion. Sur des fichiers volumineux, c’est plusieurs ordres de grandeur plus rapide que d’analyser chaque ligne :

```powershell
Select-String -Path "$pfad\*.log" -Pattern ',\+,' |
    ForEach-Object { ($_.Line -split ',')[1] } |
    Group-Object | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

```text
Count Name
----- ----
51479 SRV-MAIL01\Default Frontend SRV-MAIL01
50756 SRV-MAIL01\smtp-auth SRV-MAIL01
19405 SRV-MAIL01\smtp-intern SRV-MAIL01
15789 SRV-MAIL01\smtp-noauth SRV-MAIL01
```

Cette répartition répond à une question à laquelle le suivi des messages ne peut pas répondre : quels chemins vos applications utilisent-elles réellement ? C’est le chiffre le plus important avant une modification de connecteur.

### Si rien n’a été journalisé

S’il n’existe aucune ligne à l’heure concernée, trois raisons sont habituelles : la journalisation était désactivée sur le connecteur concerné, la limite de rétention a déjà supprimé le fichier, ou vous regardez dans le mauvais chemin, c’est-à-dire le répertoire `Hub` au lieu de `FrontEnd`. Vérifiez dans cet ordre.

## Étape 4 : vérifier les autorisations

Lorsqu’une soumission est rejetée ou, à l’inverse, que davantage est autorisé que prévu, il faut examiner les autorisations du connecteur. Un piège technique vous attend ici : `Get-ADPermission` nécessite le **DistinguishedName**. Si vous transmettez l’identité habituelle au format `Server\Connectorname`, l’appel échoue dans une session distante avec le message trompeur indiquant que l’objet est introuvable.

```powershell
$dn = (Get-ReceiveConnector "SRV-MAIL01\Default Frontend SRV-MAIL01").DistinguishedName
Get-ADPermission -Identity $dn -User "NT AUTHORITY\ANONYMOUS LOGON" |
    Where-Object { $_.ExtendedRights -like "*ms-Exch-SMTP*" } |
    Format-Table User, @{n='Rights'; e={$_.ExtendedRights}} -AutoSize
```

```text
User                         Rights
----                         ------
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Submit
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Accept-Any-Sender
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Accept-Authoritative-Domain-Sender
```

L’analyse est plus simple qu’elle n’en a l’air si vous distinguez quatre droits :

| Droit | Signification |
|---|---|
| `ms-Exch-SMTP-Submit` | peut soumettre des messages |
| `ms-Exch-SMTP-Accept-Any-Sender` | peut utiliser des adresses d’expéditeur arbitraires |
| `ms-Exch-SMTP-Accept-Authoritative-Domain-Sender` | peut se présenter comme son propre domaine |
| `ms-Exch-SMTP-Accept-Any-Recipient` | **peut relayer vers des domaines externes** |

Les trois premiers constituent l’ensemble standard nécessaire pour la soumission anonyme et la réception de courrier Internet. Seul le quatrième droit transforme un connecteur entrant en relais. Sur un connecteur qui accepte tout l’espace d’adressage, il s’agit d’un relais ouvert. Sur un connecteur limité à des IP précises, c’est au contraire le moyen habituel et prévu pour permettre aux serveurs applicatifs d’envoyer vers l’extérieur.

Ne confondez pas `Accept-Any-Sender` avec `Accept-Any-Recipient`. Le premier est inoffensif et nécessaire ; le second est le paramètre pertinent pour la sécurité.

## Étape 5 : contre-vérification par votre propre soumission

Si l’analyse reste ambiguë, soumettez vous-même un message. Vous contrôlez ainsi entièrement l’expéditeur, le destinataire et le point de soumission :

```powershell
Send-MailMessage -SmtpServer 10.0.1.11 -Port 25 `
    -From 'test@example.com' `
    -To 'empfaenger@example.net' `
    -Subject 'Diagnose, bitte ignorieren' `
    -Body 'Testeinlieferung' `
    -Encoding UTF8
```

`Send-MailMessage` est officiellement déconseillé, mais reste l’outil le plus rapide à des fins de diagnostic et est présent sur chaque serveur Windows. En cas de réussite, aucune sortie n’est produite, ce qui peut être déroutant.

Si vous testez un trajet TLS sur le port 587 et que le système distant présente un certificat ne correspondant pas au nom utilisé, par exemple parce que vous contactez l’adresse IP, l’appel échoue avec une erreur de certificat. Pour le test, vous pouvez désactiver la vérification dans la session :

```powershell
[Net.ServicePointManager]::ServerCertificateValidationCallback = { $true }
```

Cela ne s’applique qu’à la session PowerShell en cours. Faites-le consciemment et jamais dans des scripts exécutés en production.

Si le message de test arrive et que vous souhaitez savoir ce qui lui est arrivé en chemin, l’[analyseur d’en-têtes de messagerie](https://rafaelpfister.ch/tools/header-analyzer) peut vous aider : il décompose les en-têtes, trace le parcours par sauts et affiche les résultats des vérifications d’authentification, entièrement localement dans le navigateur, sans que le message ne quitte votre appareil.

## Exchange Online : même question, outil différent

Dans le tenant, les règles sont différentes, et c’est là que les méthodes habituelles échouent. Attendez-vous aux différences suivantes :

| | Exchange OnPrem | Exchange Online |
|---|---|---|
| Requête | `Get-MessageTrackingLog` | `Get-MessageTraceV2` |
| Granularité | chaque événement de transport | une ligne par message et destinataire |
| Connecteur visible | oui (avec restrictions, voir ci-dessus) | **non** |
| Lié au serveur | oui, requête par serveur | sans objet |
| Journal SMTP | disponible | **non disponible** |
| Rétention | votre configuration | environ 10 jours via le cmdlet |
| Délai | presque immédiat | quelques minutes |

Les trois conséquences les plus importantes en pratique : il n’existe **aucune attribution de connecteur** ; utilisez plutôt `FromIP` et `ToIP`. Il n’existe **aucun journal SMTP**, la conversation SMTP ne peut pas être reconstruite. Et les données apparaissent **avec un délai** : un message tout juste envoyé n’apparaît pas immédiatement.

### La requête de base

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) `
    -EndDate (Get-Date) `
    -RecipientAddress "empfaenger@example.com" `
    -ResultSize 1000 |
  Sort-Object Received |
  Format-Table Received, SenderAddress, RecipientAddress, Status, FromIP, Size -AutoSize
```

```text
Received            SenderAddress          RecipientAddress          Status    FromIP
--------            -------------          ----------------          ------    ------
11.08.2026 08:27:16 emma@partner.example   empfaenger@example.com    Delivered 10.0.20.23
11.08.2026 09:05:24 dienst@example-test.com empfaenger@example.com   Failed    10.0.20.23
```

Les valeurs les plus importantes de `Status` : `Delivered`, `Failed`, `Pending`, `Quarantined`, `FilteredAsSpam` et `Expanded` pour les listes de distribution développées. `Pending` signifie que des tentatives de remise sont encore en cours, et non qu’un élément est défaillant.

### Les détails d’un message

Le statut seul ne dit rien de la raison. Pour cela, vous avez besoin de l’affichage détaillé, qui requiert l’identifiant du message issu de la requête de base :

```powershell
$n = Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) -EndDate (Get-Date) `
        -RecipientAddress "empfaenger@example.com" -ResultSize 100 |
     Where-Object { $_.Status -eq 'Failed' } | Select-Object -First 1

Get-MessageTraceDetailV2 -MessageTraceId $n.MessageTraceId `
    -RecipientAddress $n.RecipientAddress |
  Format-List Date, Event, Action, Detail
```

Vous y trouverez les étapes de traitement dans le service, telles que les applications de règles, les décisions de filtrage et la raison d’un rejet.

### Au-delà de dix jours

Le cmdlet remonte sur environ dix jours. Pour les périodes plus anciennes, il existe la recherche historique, qui s’exécute de manière asynchrone et fournit le résultat au format CSV, sur une période allant jusqu’à 90 jours :

```powershell
Start-HistoricalSearch -ReportTitle "Analyse Nachtlauf" `
    -StartDate (Get-Date).AddDays(-45) `
    -EndDate (Get-Date).AddDays(-30) `
    -ReportType MessageTrace `
    -SenderAddress "dienst@example-test.com" `
    -NotifyAddress "admin@example.com"

Get-HistoricalSearch | Format-Table JobId, ReportTitle, Status, SubmitDate -AutoSize
```

Prévoyez du temps : selon le volume, ces tâches peuvent durer plusieurs heures.

### Piège 4 : l’absence de résultats ne prouve pas l’absence de trafic

C’est le piège le plus subtil dans le tenant. `Get-MessageTraceV2` renvoie des pages de 5000 lignes au maximum par appel. En cas de trafic élevé, une page peut ne couvrir que quelques minutes, même si vous avez interrogé sept jours. Si vous filtrez ensuite localement, par exemple sur une IP source, vous ne filtrez qu’un infime extrait.

Vous le reconnaîtrez à l’avertissement indiquant qu’il existe d’autres résultats :

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

S’il apparaît, votre analyse est **incomplète**. Si aucun résultat ne revient, la conclusion correcte est : non trouvé dans l’extrait. Ce n’est pas : n’existe pas.

Il existe deux solutions propres. Soit vous réduisez la fenêtre temporelle jusqu’à ce qu’une page la couvre entièrement, ce que confirme l’absence d’avertissement. Soit vous parcourez toutes les pages à l’aide des indications de continuation figurant dans l’avertissement. Pour déterminer si quelque chose ne se produit **jamais**, une vérification de configuration est de toute façon préférable : si un système n’a pas de route vers une destination, il ne peut pas y remettre de message, indépendamment de toute fenêtre d’observation.

L’analyse complète de toutes les adresses de soumission est un sujet à part entière, avec ses propres pièges d’interprétation. Elle est présentée dans [Qui soumet réellement des messages dans votre tenant ? Agréger les adresses IP de soumission](https://rafaelpfister.ch/blog/einliefernde-ip-adressen-aggregieren).

## Une méthode qui a fait ses preuves

En résumé, cet ordre s’est révélé être le plus rapide. Recherchez le message sur tous les serveurs et déterminez le dernier événement. En cas d’échec, passez immédiatement à `Format-List` et lisez la réponse SMTP complète, plutôt que de tirer des conclusions à partir du type d’événement. Déterminez ensuite l’étendue, donc regroupez et comptez. Ce n’est que lorsque le cas est bien circonscrit que vous reconstruisez le chemin de soumission à partir de la configuration des connecteurs et du journal SMTP. Enfin, si nécessaire, vérifiez par une soumission personnelle.

Les pertes de temps les plus fréquentes sont toujours les mêmes : lire un tableau tronqué plutôt que le message d’erreur complet, prendre des copies fantômes pour des étapes de traitement, croire la `ConnectorId` du suivi, ou considérer un échantillon vide comme une preuve. En connaissant ces quatre points, vous atteindrez généralement le bon niveau d’analyse en quelques minutes.

## Sources

1.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking) : description des champs et liste complète des types d’événements dans le suivi des messages.

2.  [Protocol logging in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/protocol-logging) : emplacements, format et rétention des journaux SMTP, y compris Front End Transport.

3.  [Shadow redundancy in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/shadow-redundancy) : explique les événements liés aux copies fantômes et à leur rejet.

4.  [Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing) : interaction entre Front End Transport et Transport Service, fondement du comportement de proxy.

5.  [Receive connectors in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/receive-connectors) : liaisons, groupes d’autorisations et mécanismes d’authentification.

6.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2) : successeur de Get-MessageTrace, y compris la logique de pagination et la liste des champs.

7.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch) : suivi asynchrone des messages sur une période allant jusqu’à 90 jours.
