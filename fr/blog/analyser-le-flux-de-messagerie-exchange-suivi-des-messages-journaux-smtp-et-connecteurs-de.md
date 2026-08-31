---
title: "Analyser le flux de messagerie Exchange : suivi des messages, journaux SMTP et connecteurs de réception"
navTitle: "Analyser le flux de messagerie"
description: "Comment déterminer systématiquement, dans Exchange OnPrem, Hybrid et Exchange Online, où un message s’est arrêté : requêtes avec exemples de sortie, lecture correcte du journal SMTP et points qui conduisent régulièrement à de fausses conclusions."
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
translationSourceHash: da923f7fa45ee5c38ea52e96d56781f7c3806556245a5f071242e7f02473a71c
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T10:09:45.097Z
translationReview: required
url: https://rafaelpfister.ch/fr/blog/analyser-le-flux-de-messagerie-exchange-suivi-des-messages-journaux-smtp-et-connecteurs-de
---

# Analyser le flux de messagerie Exchange : suivi des messages, journaux SMTP et connecteurs de réception

La question la plus fréquente en exploitation de messagerie est la suivante : un message n’est pas arrivé, où est-il passé ? Le suivi des messages y répond de manière fiable, mais uniquement si vous savez ce qu’il ne vous dit **pas**. Cet article décrit la procédure dans l’ordre qui a fait ses preuves, présente pour chaque requête la sortie typique et nomme les sources d’erreur qui coûtent régulièrement des heures, car elles suggèrent des conclusions plausibles mais erronées.

Tous les exemples utilisent des noms génériques : `SRV-MAIL01` et `SRV-MAIL02` comme serveurs de transport, `example.com` comme domaine. Si vous souhaitez composer les commandes pour votre environnement plutôt que les saisir : le [générateur de commandes](https://rafaelpfister.ch/tools/command-builder) contient les commandes courantes de suivi des messages et de capture pour PowerShell et shell Unix côte à côte, entièrement en local dans le navigateur.

## Le principe : d’abord localiser, puis expliquer

Le réflexe est de chercher immédiatement la cause. Il est plus efficace de déterminer d’abord jusqu’où le message est réellement parvenu. Cela réduit drastiquement l’espace de recherche en une étape, car vous savez ensuite si vous devez chercher dans votre propre système, sur la passerelle en amont ou chez la destination.

L’ordre est donc le suivant : trouver le message, lire le dernier événement, lire le motif de l’erreur, déterminer s’il s’agit d’un cas isolé ou d’un modèle, puis seulement reconstituer le chemin de soumission.

## Étape 1 : trouver le message

Commencez par le destinataire, car c’est presque toujours ce que vous connaissez. Il est important d’exécuter la requête sur **tous** les serveurs de transport, pas sur un seul.

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

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Server` | Serveur de transport dont le journal de suivi est interrogé ; ici, les deux serveurs successivement via le pipeline |
| `-Start` | Limite temporelle inférieure de la recherche, ici les six dernières heures |
| `-ResultSize Unlimited` | Supprime la limite par défaut de 1 000 entrées |
| `-Recipients` | Filtre les messages destinés à cette adresse de destinataire |
| `Sort-Object Timestamp` | Trie chronologiquement les résultats fusionnés des deux serveurs |
| `-AutoSize -Wrap` | Adapte la largeur des colonnes au contenu et renvoie les valeurs longues à la ligne au lieu de les tronquer |

</details>

Voici une sortie typique pour un message qui a été traité correctement :

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

Si la requête ne trouve rien, vérifiez si le destinataire a été développé via une liste de distribution. Dans ce cas, il vaut mieux rechercher par expéditeur :

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

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Server` | Serveur de transport dont le journal de suivi est interrogé |
| `-Start` | Limite temporelle inférieure de la recherche |
| `-ResultSize Unlimited` | Supprime la limite par défaut de 1 000 entrées |
| `Where-Object` | Filtre côté client les expéditeurs du domaine interne, car `-Sender` n’accepte que les adresses exactes |
| `@{n=…; e=…}` | Colonne calculée : rassemble le champ de collection `Recipients` dans une chaîne séparée par des virgules |

</details>

## Étape 2 : lire le dernier événement

L’ensemble du diagnostic repose sur le **dernier** `EventId` du message. Il vous indique quel espace de recherche doit être exploré ensuite.

| Dernier EventId | Signification | Étape suivante |
|---|---|---|
| `RECEIVE`, puis rien | Le message est bloqué | Vérifier les files d’attente |
| `SEND` ou `SENDEXTERNAL` | Transmis avec succès | Poursuivre la recherche au saut suivant |
| `FAIL` | Échec définitif | Lire la raison dans `RecipientStatus` |
| `DEFER` | Nouvelle tentative en cours | Vérifier la file d’attente et le système cible |
| `DROP` ou `POISONMESSAGE` | Rejeté | Vérifier la règle de transport ou l’agent |
| `DELIVER` | Distribué dans une boîte aux lettres locale | Vérifier les règles de boîte aux lettres |
| `RESOLVE` | Le destinataire a été réécrit | Lire l’adresse cible dans l’entrée |

`RESOLVE` est l’étape intermédiaire la plus révélatrice dans les environnements hybrides, car la réécriture vers l’adresse de routage Cloud y devient visible :

```text
EventId : RESOLVE
Source  : ROUTING
Sender  : absender@example.com
To      : BENUTZER@example.mail.onmicrosoft.com
```

Si l’adresse `onmicrosoft.com` attendue y figure, l’objet destinataire est correctement configuré et vous pouvez clore le sujet. Si l’adresse d’origine y figure toujours, il manque l’adresse cible sur l’objet local et Exchange tente une remise locale.

Si le message est bloqué, la file d’attente affiche généralement le motif en clair :

```powershell
Get-Queue -Server SRV-MAIL01 |
    Where-Object { $_.MessageCount -gt 0 } |
    Format-Table Identity, DeliveryType, Status, MessageCount, NextHopDomain, LastError `
        -AutoSize -Wrap
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Server` | Serveur dont les files d’attente de transport sont interrogées |
| `Where-Object` | Masque les files d’attente vides et n’affiche que celles contenant des messages en attente |
| `-AutoSize -Wrap` | Empêche le tronquage de la longue colonne `LastError` |

</details>

## Source d’erreur 1 : le suivi est lié au serveur, et de nombreuses entrées sont des copies fantômes

Si vous voyez dans la sortie des paires de `HARECEIVE` et `HADISCARD`, souvent avec le complément `ExplicitlyDiscarded`, ce serveur n’a **pas traité** le message. Il ne détenait qu’une copie fantôme dans le cadre de Shadow Redundancy, tandis qu’un autre serveur effectuait la remise réelle. Dès que le serveur principal signale le succès, le partenaire abandonne sa copie.

Voici à quoi cela ressemble lorsque vous n’avez interrogé que le mauvais serveur :

```text
Timestamp           EventId    SourceContext
---------           -------    -------------
11.08.2026 09:47:13 HARECEIVE  1a2aae6b-f3a3-4e04-9233-7de460b92223
11.08.2026 09:48:07 HADISCARD  ExplicitlyDiscarded;1a2aae6b-f3a3-4e04-9233-7de460b92223
```

Deux lignes, aucune erreur, aucune remise. Quiconque en conclut que le message a disparu cherche au mauvais endroit. Le traitement réel figure dans le suivi du serveur partenaire.

En pratique, cela signifie deux choses. Premièrement, ces lignes ne signalent pas un problème, mais un fonctionnement normal. Deuxièmement, vous devez impérativement interroger tous les serveurs de transport.

## Source d’erreur 2 : `Format-Table` tronque précisément les colonnes décisives

Le motif de l’erreur se trouve dans `RecipientStatus`, et ce champ est long. Dans un tableau, il disparaît complètement ou est tronqué. C’est précisément ce qui conduit à voir le `FAIL` sans en voir la raison, puis à commencer à deviner.

Dès que vous avez trouvé un cas d’erreur, passez donc à `Format-List` et développez les champs de collection :

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

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Server` | Serveur de transport dont le journal de suivi est interrogé |
| `-Start` | Limite temporelle inférieure de la recherche |
| `-ResultSize Unlimited` | Supprime la limite par défaut de 1 000 entrées |
| `-Recipients` | Filtre les messages destinés à cette adresse de destinataire |
| `-EventId FAIL` | Uniquement les entrées avec erreur définitive de remise |
| `Format-List` | Affiche chaque champ sur sa propre ligne dans toute sa longueur, sans rien tronquer |
| `@{n=…; e=…}` | Champs calculés : transforment les champs de collection `Recipients` et `RecipientStatus` en chaînes lisibles |

</details>

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

Le diagnostic est ainsi établi sans avoir eu besoin d’une seule hypothèse : le système distant conteste l’expéditeur. `LED` contient la réponse SMTP complète, `FQDN` et `IP` désignent le système ayant répondu, et `LRT` l’heure de la dernière tentative.

## Étape 3 : cas isolé ou modèle ?

Avant d’approfondir un cas individuel, déterminez l’ampleur. Cette seule requête permet de savoir si vous avez affaire à une note marginale ou à un incident :

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

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Start` | Limite temporelle inférieure, ici les huit dernières heures |
| `-EventId FAIL` | Uniquement les remises ayant définitivement échoué |
| `-ResultSize Unlimited` | Supprime la limite par défaut de 1 000 entrées |
| `Where-Object` | Filtre sur le code d’état SMTP étudié dans le champ `RecipientStatus` |
| `Group-Object` | Regroupe par domaine expéditeur (la partie après le `@`) |
| `Sort-Object Count -Descending` | Domaine le plus fréquent en tête |

</details>

Remplacez `5.1.8` par le code d’état que vous examinez. La sortie répond à la question en une ligne :

```text
Count Name
----- ----
    9 example-test.com
```

Un seul domaine expéditeur signifie : problème très circonscrit, pas d’incident, vous pouvez poursuivre sereinement vos recherches. Si vingt domaines différents y figuraient, vous auriez une panne en cours et tout le reste devrait attendre. Faire cette distinction aussi tôt permet, d’expérience, de gagner le plus de temps.

## Source d’erreur 3 : le `ConnectorId` ne désigne pas le véritable connecteur de réception

C’est la source d’erreur la plus coûteuse, car la sortie semble sérieuse. Le courrier qu’un client ou un système tiers soumet sur le port 25 atteint d’abord le **Front End Transport**. Celui-ci transmet le message au **Transport Service** sur le port 2525. Le suivi des messages n’est écrit qu’à cet endroit ; le Front End Transport n’écrit aucun suivi propre.

Vous en voyez la conséquence sur cette ligne :

```text
EventId        : RECEIVE
ConnectorId    : SRV-MAIL01\Default SRV-MAIL01
ClientIp       : 10.0.1.11
ClientHostname : srv-mail01.intern.example.com
```

Le `ConnectorId` désigne le connecteur interne sur le port 2525, et le `ClientIp` est l’adresse du **serveur proxy**, non celle de l’émetteur d’origine. Le suivi n’indique tout simplement pas lequel des connecteurs configurés sur le port 25 un système a réellement atteint. Quiconque se fie à cette indication cherche l’erreur sur un connecteur qui n’est pas impliqué.

Il existe deux moyens d’obtenir la réponse. Le premier consiste à reconstruire le parcours à partir de la configuration :

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object { Get-ReceiveConnector -Server $_ } |
    Format-List Identity, Enabled,
        @{n='Bindings';       e={$_.Bindings -join ','}},
        @{n='RemoteIPRanges'; e={$_.RemoteIPRanges -join ','}},
        PermissionGroups, AuthMechanism
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Server` | Serveur dont les connecteurs de réception sont répertoriés |
| `Format-List` | Longueurs complètes des champs ; `RemoteIPRanges` et `PermissionGroups` seraient tronqués dans des tableaux |
| `@{n=…; e=…}` | Champs calculés : regroupent les champs de collection `Bindings` et `RemoteIPRanges` en chaînes séparées par des virgules |

</details>

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

Déterminez l’IP source du système qui soumet le message et recherchez le connecteur dont `RemoteIPRanges` la contient. Si elle ne relève d’aucun connecteur restreint, il reste le connecteur frontend par défaut, qui accepte habituellement l’ensemble de l’espace d’adressage. Utilisez ici aussi `Format-List`, car `RemoteIPRanges` et `PermissionGroups` sont régulièrement tronqués dans les tableaux.

La deuxième voie est le journal SMTP, qui mérite sa propre section.

## Le journal SMTP : la seule source complète

Le journal du Front End Transport enregistre la session SMTP complète : quel connecteur a été contacté, quelle IP s’est connectée, ce que le client et le serveur se sont dit. C’est la seule source qui résout le problème décrit ci-dessus avec le `ConnectorId`.

### Activer la journalisation

Par défaut, la journalisation est **désactivée** sur la plupart des connecteurs. Vous l’activez par connecteur :

```powershell
Set-ReceiveConnector -Identity "SRV-MAIL01\Default Frontend SRV-MAIL01" `
    -ProtocolLoggingLevel Verbose
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Identity` | Le connecteur à modifier sous la forme `Server\Connectorname` |
| `-ProtocolLoggingLevel Verbose` | Active la journalisation SMTP ; `None` la désactive à nouveau |

</details>

Pour les connexions sortantes, procédez de même avec `Set-SendConnector`. N’oubliez pas de rétablir la valeur sur `None` après l’analyse, car une journalisation détaillée consomme de l’espace disque et écrit des volumes considérables en cas de trafic important.

### Où se trouvent les fichiers

Exchange sépare les journaux selon le service et le sens. Il est inutile de coder les chemins en dur, interrogez-les :

```powershell
Get-FrontendTransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath,
        ReceiveProtocolLogMaxAge, ReceiveProtocolLogMaxDirectorySize

Get-TransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `SRV-MAIL01` | Paramètre positionnel `-Identity`: le serveur à interroger |
| `ReceiveProtocolLogPath`, `SendProtocolLogPath` | Chemins de stockage des journaux pour les connexions entrantes et sortantes respectivement |
| `ReceiveProtocolLogMaxAge` | Âge maximal des fichiers journaux ; les fichiers plus anciens sont supprimés |
| `ReceiveProtocolLogMaxDirectorySize` | Limite supérieure de l’espace utilisé par le répertoire des journaux |

</details>

Ils se trouvent généralement sous le chemin d’installation, dans `TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive` pour le Front End Transport et dans `TransportRoles\Logs\Hub\ProtocolLog\SmtpReceive` pour le Transport Service. **C’est le cœur du sujet :** les connexions clientes sur le port 25 se trouvent exclusivement dans le chemin `FrontEnd`, tandis que le chemin `Hub` ne contient que le trafic de transmission interne sur 2525.

Tenez compte de la rétention. `ReceiveProtocolLogMaxAge` est souvent défini sur 30 jours, tandis que `ReceiveProtocolLogMaxDirectorySize` limite aussi l’espace consommé. En cas de trafic important, la limite de taille intervient bien avant la limite d’âge, et vos journaux n’ont alors plus que quelques jours.

### Comprendre le format

Les fichiers sont des CSV avec des lignes d’en-tête commençant par `#`. Les principales colonnes sont `date-time`, `connector-id`, `session-id`, `sequence-number`, `local-endpoint`, `remote-endpoint`, `event` et `data`.

La colonne décisive est `event`, un seul caractère :

| Caractère | Signification |
|---|---|
| `+` | Connexion établie |
| `-` | Connexion terminée |
| `>` | Le serveur envoie au client |
| `<` | Le client envoie au serveur |
| `*` | Information du serveur, pas de trafic SMTP |

Vous reconnaissez une session à son `session-id` commun ; `sequence-number` indique l’ordre au sein de la session. Voici un extrait typique :

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

Vous y trouvez tout ce qui manquait dans le suivi des messages : le **véritable** connecteur (`smtp-noauth`), la **véritable** IP source (`10.0.20.22`) et le nom sous lequel le système se présente dans `EHLO`.

### Rechercher de manière ciblée

Pour les cas individuels, un filtre de texte est nettement plus rapide qu’une évaluation d’objets. Recherchez l’adresse de l’expéditeur ou le nom `EHLO` et obtenez l’identifiant de session :

```powershell
$pfad = (Get-FrontendTransportService SRV-MAIL01).ReceiveProtocolLogPath
Select-String -Path "$pfad\*.log" -Pattern "dienst@example-test.com" -SimpleMatch |
    Select-Object -First 5 Line
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Path "$pfad\*.log"` | Recherche dans tous les fichiers journaux du chemin précédemment interrogé |
| `-Pattern` | Le terme recherché, ici l’adresse de l’expéditeur |
| `-SimpleMatch` | Traite le motif comme du texte plutôt que comme une expression régulière ; le point dans l’adresse n’a ainsi pas besoin d’être échappé |
| `-First 5` | Limite la sortie aux cinq premiers résultats |

</details>

Avec l’`session-id` trouvé, récupérez la session complète :

```powershell
Select-String -Path "$pfad\*.log" -Pattern "08DEF44EC454A414" -SimpleMatch |
    ForEach-Object { $_.Line } | Select-Object -First 40
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Pattern` | L’identifiant de session issu du premier résultat |
| `-SimpleMatch` | Recherche littérale sans évaluation regex |
| `-First 40` | Limite la sortie aux 40 premières lignes de la session |

</details>

Si vous voulez seulement savoir quels connecteurs voient effectivement du trafic, comptez les établissements de connexion. C’est, dans les fichiers volumineux, bien plus rapide que d’analyser chaque ligne :

```powershell
Select-String -Path "$pfad\*.log" -Pattern ',\+,' |
    ForEach-Object { ($_.Line -split ',')[1] } |
    Group-Object | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Pattern ',\+,'` | Expression régulière pour l’événement `+` (établissement de connexion) entre deux virgules CSV ; le signe plus est échappé |
| `ForEach-Object { … -split ',' }` | Découpe la ligne trouvée aux virgules et accède à la deuxième colonne, `connector-id` |
| `Group-Object` | Compte les établissements de connexion par connecteur |
| `Sort-Object Count -Descending` | Connecteur le plus utilisé en tête |

</details>

```text
Count Name
----- ----
51479 SRV-MAIL01\Default Frontend SRV-MAIL01
50756 SRV-MAIL01\smtp-auth SRV-MAIL01
19405 SRV-MAIL01\smtp-intern SRV-MAIL01
15789 SRV-MAIL01\smtp-noauth SRV-MAIL01
```

Cette répartition répond à une question à laquelle le suivi des messages ne peut pas répondre : quels chemins vos applications utilisent-elles réellement ? Avant une modification de connecteur, c’est le chiffre le plus important qui soit.

### Si rien n’a été journalisé

S’il manque toute ligne au moment concerné, trois raisons sont habituelles : la journalisation était désactivée sur le connecteur concerné, la limite de rétention a déjà supprimé le fichier, ou vous regardez dans le mauvais chemin, c’est-à-dire dans le répertoire `Hub` au lieu de `FrontEnd`. Vérifiez dans cet ordre.

## Étape 4 : vérifier les autorisations

Si une soumission est rejetée ou si, à l’inverse, davantage est autorisé que prévu, il faut examiner les autorisations du connecteur. Il existe ici une particularité technique : `Get-ADPermission` requiert le **DistinguishedName**. Si vous transmettez l’identité habituelle sous la forme `Server\Connectorname`, l’appel échoue dans une session distante avec le message trompeur indiquant que l’objet est introuvable.

```powershell
$dn = (Get-ReceiveConnector "SRV-MAIL01\Default Frontend SRV-MAIL01").DistinguishedName
Get-ADPermission -Identity $dn -User "NT AUTHORITY\ANONYMOUS LOGON" |
    Where-Object { $_.ExtendedRights -like "*ms-Exch-SMTP*" } |
    Format-Table User, @{n='Rights'; e={$_.ExtendedRights}} -AutoSize
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Identity $dn` | L’objet à vérifier sous forme de DistinguishedName ; la forme `Server\Connectorname` échoue dans les sessions distantes |
| `-User` | Limite la sortie aux autorisations de ce principal de sécurité, ici l’accès anonyme |
| `Where-Object` | Filtre sur les Extended Rights pertinents pour SMTP |

</details>

```text
User                         Rights
----                         ------
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Submit
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Accept-Any-Sender
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Accept-Authoritative-Domain-Sender
```

L’interprétation est plus simple qu’elle n’en a l’air si vous distinguez quatre droits :

| Droit | Signification |
|---|---|
| `ms-Exch-SMTP-Submit` | Peut soumettre des messages |
| `ms-Exch-SMTP-Accept-Any-Sender` | Peut utiliser des adresses d’expéditeur arbitraires |
| `ms-Exch-SMTP-Accept-Authoritative-Domain-Sender` | Peut se présenter comme le domaine interne |
| `ms-Exch-SMTP-Accept-Any-Recipient` | **peut relayer vers des domaines externes** |

Les trois premiers constituent l’ensemble standard nécessaire pour la soumission anonyme et la réception de courrier Internet. Seul le quatrième droit transforme un connecteur entrant en relais. Sur un connecteur qui accepte l’ensemble de l’espace d’adressage, il s’agit d’un relais ouvert. Sur un connecteur avec une restriction IP stricte, c’est en revanche le moyen habituel et voulu pour permettre aux serveurs d’applications d’envoyer vers l’extérieur.

Ne confondez pas `Accept-Any-Sender` avec `Accept-Any-Recipient`. Le premier est inoffensif et nécessaire, le second est le paramètre pertinent pour la sécurité.

## Étape 5 : test de contrôle par soumission propre

Si l’analyse reste ambiguë, soumettez vous-même un message. Vous contrôlez ainsi intégralement l’expéditeur, le destinataire et le point de soumission :

```powershell
Send-MailMessage -SmtpServer 10.0.1.11 -Port 25 `
    -From 'test@example.com' `
    -To 'empfaenger@example.net' `
    -Subject 'Diagnose, bitte ignorieren' `
    -Body 'Testeinlieferung' `
    -Encoding UTF8
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-SmtpServer` | Hôte cible de la soumission, ici volontairement sous forme d’adresse IP pour atteindre un point de terminaison précis |
| `-Port 25` | Port cible ; 25 pour une soumission non authentifiée de serveur à serveur |
| `-From` | Expéditeur de l’enveloppe et de l’en-tête du message de test |
| `-To` | Adresse du destinataire |
| `-Subject` | Ligne d’objet |
| `-Body` | Corps du message |
| `-Encoding UTF8` | Encodage des caractères pour l’objet et le texte, évite les problèmes d’accents |

</details>

`Send-MailMessage` est officiellement obsolète, mais reste l’outil le plus rapide à des fins de diagnostic et est présent sur chaque serveur Windows. En cas de succès, aucune sortie n’est affichée, ce qui peut surprendre.

Si vous testez un chemin TLS sur le port 587 et que le système distant présente un certificat ne correspondant pas au nom utilisé, par exemple parce que vous contactez l’adresse IP, l’appel échoue avec une erreur de certificat. Pour le test, vous pouvez suspendre la vérification dans la session :

```powershell
[Net.ServicePointManager]::ServerCertificateValidationCallback = { $true }
```

Cela ne s’applique qu’à la session PowerShell en cours. Définissez-le consciemment et jamais dans des scripts exécutés en production.

Si le message de test arrive et que vous souhaitez savoir ce qui lui est arrivé en chemin, l’[analyseur d’en-têtes de messagerie](https://rafaelpfister.ch/tools/header-analyzer) vous aide : il décompose les en-têtes, trace le chemin à travers les sauts et affiche les résultats des vérifications d’authentification, entièrement en local dans le navigateur, sans que le message ne quitte votre appareil.

## Exchange Online : la même question, un autre outil

Dans le tenant, les règles sont différentes, et c’est là que les méthodes habituelles échouent. Attendez-vous aux différences suivantes :

| | Exchange OnPrem | Exchange Online |
|---|---|---|
| Requête | `Get-MessageTrackingLog` | `Get-MessageTraceV2` |
| Granularité | Chaque événement de transport | Une ligne par message et destinataire |
| Connecteur visible | Oui (avec restriction, voir ci-dessus) | **Non** |
| Liaison au serveur | Oui, requête par serveur | Sans objet |
| Journal SMTP | Disponible | **Non disponible** |
| Rétention | Votre configuration | Environ 10 jours via le cmdlet |
| Délai | Presque immédiat | Quelques minutes |

Les trois conséquences les plus importantes en pratique : il n’existe **aucune attribution de connecteur**, vous vous appuyez sur `FromIP` et `ToIP`. Il n’existe **aucun journal SMTP**, la conversation SMTP ne peut pas être reconstituée. Et les données apparaissent **avec retard**, un message tout juste envoyé n’apparaît pas immédiatement.

### La requête de base

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) `
    -EndDate (Get-Date) `
    -RecipientAddress "empfaenger@example.com" `
    -ResultSize 1000 |
  Sort-Object Received |
  Format-Table Received, SenderAddress, RecipientAddress, Status, FromIP, Size -AutoSize
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-StartDate` | Limite temporelle inférieure de la requête, ici les quatre dernières heures |
| `-EndDate` | Limite temporelle supérieure ; le cmdlet exige les deux limites |
| `-RecipientAddress` | Filtre les messages destinés à cette adresse de destinataire |
| `-ResultSize 1000` | Nombre maximal de lignes de cette page ; la limite supérieure est de 5 000 |

</details>

```text
Received            SenderAddress          RecipientAddress          Status    FromIP
--------            -------------          ----------------          ------    ------
11.08.2026 08:27:16 emma@partner.example   empfaenger@example.com    Delivered 10.0.20.23
11.08.2026 09:05:24 dienst@example-test.com empfaenger@example.com   Failed    10.0.20.23
```

Les principales valeurs de `Status` : `Delivered`, `Failed`, `Pending`, `Quarantined`, `FilteredAsSpam` et `Expanded` pour les listes de distribution développées. `Pending` signifie que les tentatives de remise sont encore en cours, non que quelque chose est défaillant.

### Les détails d’un message

Le statut seul ne dit rien de la raison. Pour cela, vous avez besoin de la vue détaillée, qui nécessite l’identifiant du message issu de la requête de base :

```powershell
$n = Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) -EndDate (Get-Date) `
        -RecipientAddress "empfaenger@example.com" -ResultSize 100 |
     Where-Object { $_.Status -eq 'Failed' } | Select-Object -First 1

Get-MessageTraceDetailV2 -MessageTraceId $n.MessageTraceId `
    -RecipientAddress $n.RecipientAddress |
  Format-List Date, Event, Action, Detail
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-MessageTraceId` | Identifiant unique du message issu de la requête de base ; obligatoire |
| `-RecipientAddress` | Destinataire dont les étapes de traitement sont affichées ; également obligatoire, car un message peut avoir plusieurs destinataires |

</details>

Vous y trouverez les étapes de traitement dans le service, par exemple les applications de règles, les décisions de filtrage et le motif d’un rejet.

### Au-delà de dix jours

Le cmdlet remonte sur environ dix jours. Pour les périodes plus anciennes, il existe la recherche historique, qui fonctionne de façon asynchrone et fournit le résultat sous forme de CSV, sur une plage allant jusqu’à 90 jours :

```powershell
Start-HistoricalSearch -ReportTitle "Analyse Nachtlauf" `
    -StartDate (Get-Date).AddDays(-45) `
    -EndDate (Get-Date).AddDays(-30) `
    -ReportType MessageTrace `
    -SenderAddress "dienst@example-test.com" `
    -NotifyAddress "admin@example.com"

Get-HistoricalSearch | Format-Table JobId, ReportTitle, Status, SubmitDate -AutoSize
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-ReportTitle` | Nom librement choisi de la tâche, sous lequel le résultat pourra être retrouvé ultérieurement |
| `-StartDate`, `-EndDate` | Période étudiée, jusqu’à 90 jours en arrière |
| `-ReportType MessageTrace` | Type de rapport ; `MessageTrace` fournit la vue d’ensemble des messages au format CSV |
| `-SenderAddress` | Filtre sur cette adresse d’expéditeur |
| `-NotifyAddress` | Destinataire de la notification de fin ; doit être une adresse d’un domaine accepté du tenant |

</details>

Prévoyez du temps : selon leur ampleur, ces tâches peuvent s’exécuter pendant des heures.

### Source d’erreur 4 : l’absence de résultats ne prouve pas l’absence de trafic

C’est la source d’erreur la plus subtile dans le tenant. `Get-MessageTraceV2` renvoie les résultats par pages, avec un maximum de 5 000 lignes par appel. En cas de trafic important, une page peut ne couvrir que quelques minutes, même si vous avez interrogé sept jours. Si vous filtrez ensuite localement, par exemple sur une IP source, vous filtrez sur un extrait minuscule.

Vous le reconnaissez à l’avertissement indiquant qu’il existe d’autres résultats :

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

S’il apparaît, votre analyse est **incomplète**. Si aucun résultat n’est renvoyé, la conclusion correcte est : non trouvé dans l’extrait. Ce n’est pas : n’existe pas.

Il existe deux solutions propres. Soit vous réduisez la fenêtre temporelle jusqu’à ce qu’une page la couvre entièrement, ce que confirme l’absence de l’avertissement. Soit vous parcourez toutes les pages à l’aide des informations de continuation de l’avertissement. Pour déterminer si quelque chose n’arrive **jamais**, une vérification de configuration est de toute façon préférable : si un système ne possède aucune route vers une destination, il ne peut pas y remettre de messages, indépendamment de toute fenêtre d’observation.

L’analyse complète de toutes les adresses de soumission est un sujet à part entière, avec ses propres points délicats d’interprétation. Elle est présentée dans [Qui soumet réellement du courrier dans votre tenant ? Agréger les adresses IP de soumission](https://rafaelpfister.ch/blog/einliefernde-ip-adressen-aggregieren).

## Une procédure qui a fait ses preuves

En résumé, cet ordre s’est avéré le plus rapide. Rechercher le message sur tous les serveurs et déterminer le dernier événement. En cas d’échec, passer immédiatement à `Format-List` et lire la réponse SMTP complète, plutôt que de déduire quoi que ce soit du type d’événement. Ensuite, déterminer l’ampleur, donc regrouper et compter. Ce n’est que lorsque le cas est bien circonscrit qu’il faut reconstituer le chemin de soumission à partir de la configuration des connecteurs et du journal SMTP. Enfin, si nécessaire, effectuer une contre-vérification par une soumission propre.

Les pertes de temps les plus fréquentes sont toujours les mêmes : lire un tableau tronqué au lieu du message d’erreur complet, prendre des copies fantômes pour des étapes de traitement, croire au `ConnectorId` du suivi et considérer un échantillon vide comme une preuve. Ceux qui connaissent ces quatre points arrivent généralement au bon niveau en quelques minutes.

## Sources

1.  [Suivi des messages dans Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking) : description des champs et liste complète des types d’événements dans le suivi des messages.

2.  [Journalisation des protocoles dans Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/protocol-logging) : emplacements de stockage, format et rétention des journaux SMTP, y compris Front End Transport.

3.  [Redondance fantôme dans Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/shadow-redundancy) : explique les événements liés aux copies fantômes et à leur abandon.

4.  [Routage du courrier dans Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing) : interaction entre Front End Transport et Transport Service, fondement du comportement de proxy.

5.  [Connecteurs de réception dans Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/receive-connectors) : liaisons, groupes d’autorisations et mécanismes d’authentification.

6.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2) : successeur de Get-MessageTrace, y compris la logique de pagination et la liste des champs.

7.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch) : suivi asynchrone des messages sur une période allant jusqu’à 90 jours.
