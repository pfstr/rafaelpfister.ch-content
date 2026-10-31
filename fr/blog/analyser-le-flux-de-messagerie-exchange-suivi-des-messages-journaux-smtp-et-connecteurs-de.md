---
title: "Analyser le flux de messagerie Exchange : suivi des messages, journaux SMTP et connecteurs de réception"
navTitle: "Analyser le flux de messagerie"
description: "Comment déterminer systématiquement, dans Exchange OnPrem, Hybrid et Exchange Online, où un message est resté : requêtes avec exemples de sortie, lecture correcte du journal SMTP et points menant régulièrement à de fausses conclusions."
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
translationSourceHash: 867f3a9a27b8db05d2721cce6282149f4616c4170576d0ab3f39a3cbde3701e3
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:24:28.293Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/analyser-le-flux-de-messagerie-exchange-suivi-des-messages-journaux-smtp-et-connecteurs-de
---

# Analyser le flux de messagerie Exchange : suivi des messages, journaux SMTP et connecteurs de réception

La question la plus fréquente dans l’exploitation de la messagerie est la suivante : un message n’est pas arrivé, où est-il resté ? Le suivi des messages y répond de manière fiable, mais seulement si vous savez ce qu’il ne vous indique **pas**. Cet article décrit la procédure dans l’ordre qui a fait ses preuves, présente la sortie typique de chaque requête et identifie les sources d’erreurs qui font régulièrement perdre des heures parce qu’elles suggèrent des conclusions plausibles, mais erronées.

Tous les exemples utilisent des noms génériques : `SRV-MAIL01` et `SRV-MAIL02` comme serveurs de transport, `example.com` comme domaine. Si vous souhaitez composer les commandes pour votre environnement au lieu de les saisir : le [générateur de commandes](https://rafaelpfister.ch/tools/command-builder) contient les commandes courantes de suivi des messages et de capture pour PowerShell et le shell Unix côte à côte, entièrement localement dans le navigateur.

## Le principe : d’abord localiser, ensuite expliquer

Le réflexe est de chercher immédiatement la cause. Il est plus efficace de déterminer d’abord jusqu’où le message est réellement arrivé. Cela réduit considérablement l’espace de recherche en une étape, car vous savez ensuite si vous devez chercher dans votre propre système, chez la passerelle en amont ou chez la destination.

L’ordre est donc le suivant : trouver le message, lire le dernier événement, lire la raison de l’erreur, déterminer s’il s’agit d’un cas isolé ou d’un schéma, puis seulement reconstruire le chemin de soumission.

## Étape 1 : trouver le message

Commencez par le destinataire, car vous le connaissez presque toujours. Il est important d’exécuter la requête sur **tous** les serveurs de transport, et non sur un seul.

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
| `-Server` | Serveur de transport dont le journal de suivi est interrogé ; ici, les deux serveurs sont interrogés l’un après l’autre via le pipeline |
| `-Start` | Limite inférieure de la recherche, ici les six dernières heures |
| `-ResultSize Unlimited` | Supprime la limite par défaut de 1 000 entrées |
| `-Recipients` | Filtre les messages destinés à cette adresse de destinataire |
| `Sort-Object Timestamp` | Trie chronologiquement les résultats combinés des deux serveurs |
| `-AutoSize -Wrap` | Adapte la largeur des colonnes au contenu et renvoie à la ligne les valeurs longues au lieu de les tronquer |

</details>

Voici une sortie typique pour un message correctement traité :

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

Si la requête ne trouve rien, vérifiez si le destinataire a été développé via une liste de distribution. Dans ce cas, recherchez plutôt par expéditeur :

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
| `-Start` | Limite inférieure de la recherche |
| `-ResultSize Unlimited` | Supprime la limite par défaut de 1 000 entrées |
| `Where-Object` | Filtre côté client les expéditeurs du domaine propre, car `-Sender` n’accepte que les adresses exactes |
| `@{n=…; e=…}` | Colonne calculée : rassemble le champ de collection `Recipients` dans une chaîne séparée par des virgules |

</details>

## Étape 2 : lire le dernier événement

L’ensemble du diagnostic repose sur le **dernier** `EventId` du message. Il vous indique quel espace de recherche examiner ensuite.

| Dernier EventId | Signification | Étape suivante |
|---|---|---|
| `RECEIVE`, puis rien | Le message est bloqué | Vérifier les files d’attente |
| `SEND` ou `SENDEXTERNAL` | Transmis avec succès | Poursuivre la recherche au saut suivant |
| `FAIL` | Échec définitif | Lire la raison dans `RecipientStatus` |
| `DEFER` | Nouvelle tentative en cours | Vérifier la file d’attente et le système cible |
| `DROP` ou `POISONMESSAGE` | Rejeté | Vérifier la règle de transport ou l’agent |
| `DELIVER` | Distribué dans une boîte aux lettres locale | Vérifier les règles de boîte aux lettres |
| `RESOLVE` | Le destinataire a été réécrit | Lire l’adresse cible dans l’entrée |

`RESOLVE` est l’étape intermédiaire la plus révélatrice dans les environnements hybrides, car la réécriture vers l’adresse de routage cloud y devient visible :

```text
EventId : RESOLVE
Source  : ROUTING
Sender  : absender@example.com
To      : BENUTZER@example.mail.onmicrosoft.com
```

Si l’adresse `onmicrosoft.com` attendue s’y trouve, l’objet destinataire est correctement configuré et vous pouvez clore le sujet. Si l’adresse d’origine y figure toujours, l’adresse cible manque sur l’objet local et Exchange tente une distribution locale.

Si le message est bloqué, la file d’attente indique généralement la raison en clair :

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
| `Where-Object` | Masque les files d’attente vides, n’affiche que celles contenant des messages en attente |
| `-AutoSize -Wrap` | Empêche le tronquage de la longue colonne `LastError` |

</details>

## Source d’erreur 1 : le suivi est lié au serveur, et de nombreuses entrées sont des copies fantômes

Si vous voyez dans la sortie des paires de `HARECEIVE` et `HADISCARD`, souvent accompagnées de `ExplicitlyDiscarded`, ce serveur n’a **pas traité** le message. Il ne détenait qu’une copie fantôme dans le cadre de Shadow Redundancy, tandis qu’un autre serveur assurait la distribution effective. Dès que le serveur principal signale le succès, le partenaire supprime sa copie.

Voici ce que cela donne lorsque vous n’avez interrogé que le mauvais serveur :

```text
Timestamp           EventId    SourceContext
---------           -------    -------------
11.08.2026 09:47:13 HARECEIVE  1a2aae6b-f3a3-4e04-9233-7de460b92223
11.08.2026 09:48:07 HADISCARD  ExplicitlyDiscarded;1a2aae6b-f3a3-4e04-9233-7de460b92223
```

Deux lignes, aucune erreur, aucune distribution. En conclure que le message a disparu revient à chercher au mauvais endroit. Le traitement effectif figure dans le suivi du serveur partenaire.

En pratique, cela implique deux choses. Premièrement, ces lignes ne signalent pas un problème, mais un fonctionnement normal. Deuxièmement, vous devez impérativement interroger tous les serveurs de transport.

## Source d’erreur 2 : `Format-Table` tronque précisément les colonnes décisives

La raison de l’erreur se trouve dans `RecipientStatus`, et ce champ est long. Dans un tableau, il disparaît totalement ou est tronqué. C’est précisément ce qui conduit à voir le `FAIL` sans en voir la raison et à commencer à deviner.

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
| `-Start` | Limite inférieure de la recherche |
| `-ResultSize Unlimited` | Supprime la limite par défaut de 1 000 entrées |
| `-Recipients` | Filtre les messages destinés à cette adresse de destinataire |
| `-EventId FAIL` | Uniquement les entrées avec échec de distribution définitif |
| `Format-List` | Affiche chaque champ sur sa propre ligne dans toute sa longueur, sans rien tronquer |
| `@{n=…; e=…}` | Champs calculés : développent les champs de collection `Recipients` et `RecipientStatus` en chaînes lisibles |

</details>

Et voici la différence. D’abord la vue tabulaire, qui n’explique rien :

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

Le diagnostic est ainsi établi, sans avoir eu besoin de la moindre hypothèse : le système distant conteste l’expéditeur. `LED` contient la réponse SMTP complète, `FQDN` et `IP` désignent le système ayant répondu, et `LRT` le moment de la dernière tentative.

## Étape 3 : cas isolé ou schéma ?

Avant d’approfondir un cas particulier, déterminez l’étendue. Cette seule requête décide s’il s’agit d’une note marginale ou d’un incident :

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
| `-Start` | Limite inférieure, ici les huit dernières heures |
| `-EventId FAIL` | Uniquement les distributions ayant échoué définitivement |
| `-ResultSize Unlimited` | Supprime la limite par défaut de 1 000 entrées |
| `Where-Object` | Filtre le code d’état SMTP étudié dans le champ `RecipientStatus` |
| `Group-Object` | Regroupe par domaine expéditeur, la partie après `@` |
| `Sort-Object Count -Descending` | Domaine le plus fréquent en tête |

</details>

Remplacez `5.1.8` par le code d’état que vous examinez. La sortie répond à la question en une ligne :

```text
Count Name
----- ----
    9 example-test.com
```

Un seul domaine expéditeur signifie : problème très circonscrit, pas un incident ; vous pouvez poursuivre vos recherches sereinement. S’il y avait vingt domaines différents, vous auriez une panne en cours et tout le reste devrait attendre. Faire cette distinction aussi tôt permet, d’expérience, de gagner le plus de temps.

## Source d’erreur 3 : le `ConnectorId` ne désigne pas le véritable connecteur de réception

C’est la source d’erreur la plus coûteuse, car la sortie semble sérieuse. Le courrier soumis par un client ou un système tiers sur le port 25 arrive d’abord au **Front End Transport**. Celui-ci transmet le message au **Transport Service** sur le port 2525. Le suivi des messages n’est écrit qu’à ce stade ; le Front End Transport n’écrit pas son propre suivi.

Vous en voyez la conséquence sur cette ligne :

```text
EventId        : RECEIVE
ConnectorId    : SRV-MAIL01\Default SRV-MAIL01
ClientIp       : 10.0.1.11
ClientHostname : srv-mail01.intern.example.com
```

Le `ConnectorId` indique le connecteur interne sur le port 2525, et la `ClientIp` est l’adresse du **serveur proxy**, non celle de l’expéditeur d’origine. Le suivi n’indique tout simplement pas lequel des connecteurs configurés sur le port 25 un système a réellement atteint. Quiconque se fie à cette indication cherche l’erreur dans un connecteur qui n’a même pas été impliqué.

Il existe deux moyens d’obtenir la réponse. Le premier consiste à reconstruire à partir de la configuration :

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
| `-Server` | Serveur dont les connecteurs de réception sont listés |
| `Format-List` | Longueurs de champs complètes ; `RemoteIPRanges` et `PermissionGroups` seraient tronqués dans les tableaux |
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

Déterminez l’IP source du système soumettant le message et recherchez le connecteur dont `RemoteIPRanges` la contient. Si elle ne correspond à aucun connecteur restreint, il reste le connecteur frontal par défaut, qui accepte généralement tout l’espace d’adresses. Utilisez ici aussi `Format-List`, car `RemoteIPRanges` et `PermissionGroups` sont régulièrement tronqués dans les tableaux.

Le second moyen est le journal SMTP, qui mérite sa propre section.

## Le journal SMTP : l’unique source complète

Le journal du Front End Transport enregistre la session SMTP complète : quel connecteur a été contacté, quelle IP s’est connectée, ce que le client et le serveur se sont dit. C’est l’unique source qui résout le problème décrit ci-dessus avec `ConnectorId`.

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

Pour les connexions sortantes, procédez de manière analogue avec `Set-SendConnector`. N’oubliez pas de remettre la valeur à `None` après l’analyse, car une journalisation détaillée consomme de l’espace disque et génère des volumes de données considérables en cas de trafic élevé.

### Emplacement des fichiers

Exchange sépare les journaux par service et par direction. Il est inutile de coder les chemins en dur ; interrogez-les :

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
| `ReceiveProtocolLogPath`, `SendProtocolLogPath` | Chemins de stockage des journaux pour les connexions entrantes et sortantes |
| `ReceiveProtocolLogMaxAge` | Âge maximal des fichiers journaux ; les plus anciens sont supprimés |
| `ReceiveProtocolLogMaxDirectorySize` | Limite supérieure de l’espace disque utilisé par le répertoire des journaux |

</details>

Ils se trouvent généralement sous le chemin d’installation, dans `TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive` pour le Front End Transport et dans `TransportRoles\Logs\Hub\ProtocolLog\SmtpReceive` pour le Transport Service. **C’est le point essentiel :** les connexions clientes sur le port 25 se trouvent exclusivement dans le chemin `FrontEnd`, tandis que le chemin `Hub` ne contient que le trafic de transmission interne sur 2525.

Tenez compte de la conservation. `ReceiveProtocolLogMaxAge` est souvent réglé sur 30 jours, tandis que `ReceiveProtocolLogMaxDirectorySize` limite en plus l’espace utilisé. En cas de trafic élevé, la limite de taille intervient nettement avant la limite d’âge ; vos journaux ne remontent alors plus qu’à quelques jours.

### Comprendre le format

Les fichiers sont des CSV avec des en-têtes commençant par `#`. Les colonnes les plus importantes sont `date-time`, `connector-id`, `session-id`, `sequence-number`, `local-endpoint`, `remote-endpoint`, `event` et `data`.

La colonne déterminante est `event`, un seul caractère :

| Caractère | Signification |
|---|---|
| `+` | Connexion établie |
| `-` | Connexion terminée |
| `>` | Le serveur envoie au client |
| `<` | Le client envoie au serveur |
| `*` | Information du serveur, pas de trafic SMTP |

Vous reconnaissez une session à la valeur commune `session-id`; `sequence-number` indique l’ordre au sein de la session. Voici un extrait typique :

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

Vous y trouvez tout ce qui manquait au suivi des messages : le **véritable** connecteur (`smtp-noauth`), la **véritable** IP source (`10.0.20.22`) et le nom sous lequel le système se présente dans `EHLO`.

### Rechercher précisément

Pour les cas isolés, un filtre textuel est nettement plus rapide qu’une analyse d’objets. Recherchez l’adresse de l’expéditeur ou le nom `EHLO` et récupérez l’identifiant de session :

```powershell
$pfad = (Get-FrontendTransportService SRV-MAIL01).ReceiveProtocolLogPath
Select-String -Path "$pfad\*.log" -Pattern "dienst@example-test.com" -SimpleMatch |
    Select-Object -First 5 Line
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Path "$pfad\*.log"` | Recherche dans tous les fichiers journaux du chemin interrogé précédemment |
| `-Pattern` | Le terme de recherche, ici l’adresse de l’expéditeur |
| `-SimpleMatch` | Traite le modèle comme du texte plutôt que comme une expression régulière ; le point dans l’adresse ne nécessite donc pas d’échappement |
| `-First 5` | Limite la sortie aux cinq premiers résultats |

</details>

Avec le `session-id` trouvé, récupérez la session complète :

```powershell
Select-String -Path "$pfad\*.log" -Pattern "08DEF44EC454A414" -SimpleMatch |
    ForEach-Object { $_.Line } | Select-Object -First 40
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Pattern` | L’identifiant de session issu du premier résultat |
| `-SimpleMatch` | Recherche littérale sans évaluation par expression régulière |
| `-First 40` | Limite la sortie aux 40 premières lignes de la session |

</details>

Si vous voulez seulement savoir quels connecteurs voient du trafic, comptez les établissements de connexion. Avec de gros fichiers, cela est plusieurs ordres de grandeur plus rapide que d’analyser chaque ligne :

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
| `-Pattern ',\+,'` | Expression régulière pour l’événement `+` (établissement de connexion) entre deux virgules CSV ; le plus est échappé |
| `ForEach-Object { … -split ',' }` | Scinde la ligne correspondante aux virgules et récupère la deuxième colonne, `connector-id` |
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

Cette répartition répond à une question à laquelle le suivi des messages ne peut pas répondre : quels chemins vos applications utilisent-elles réellement ? Avant une modification de connecteur, c’est de loin le chiffre le plus important.

### Lorsqu’aucune donnée n’a été journalisée

S’il manque toute ligne au moment concerné, il existe trois raisons habituelles : la journalisation était désactivée sur le connecteur concerné, la limite de conservation a déjà supprimé le fichier, ou vous regardez dans le mauvais chemin, c’est-à-dire dans le répertoire `Hub` plutôt que `FrontEnd`. Vérifiez dans cet ordre.

## Étape 4 : vérifier les autorisations

Si une soumission est rejetée ou, inversement, si davantage est autorisé que prévu, examinez les autorisations du connecteur. Il existe ici une particularité technique : `Get-ADPermission` requiert le **DistinguishedName**. Si vous transmettez l’identité habituelle sous la forme `Server\Connectorname`, l’appel échoue dans une session distante avec le message trompeur indiquant que l’objet est introuvable.

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
| `Where-Object` | Filtre les Extended Rights pertinents pour SMTP |

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
| `ms-Exch-SMTP-Submit` | peut soumettre des messages |
| `ms-Exch-SMTP-Accept-Any-Sender` | peut utiliser des adresses d’expéditeur quelconques |
| `ms-Exch-SMTP-Accept-Authoritative-Domain-Sender` | peut se présenter comme le domaine propre |
| `ms-Exch-SMTP-Accept-Any-Recipient` | **peut relayer vers des domaines externes** |

Les trois premiers constituent l’ensemble standard requis pour la soumission anonyme et la réception de courrier Internet. Seul le quatrième droit transforme un connecteur entrant en relais. Sur un connecteur qui accepte tout l’espace d’adresses, c’est un relais ouvert. Sur un connecteur avec une restriction IP stricte, c’est en revanche le moyen habituel et souhaité de permettre à des serveurs d’applications d’envoyer vers l’extérieur.

Ne confondez pas `Accept-Any-Sender` avec `Accept-Any-Recipient`. Le premier est inoffensif et nécessaire, le second est le paramètre pertinent pour la sécurité.

## Étape 5 : test de contrôle par votre propre soumission

Si l’analyse reste ambiguë, soumettez vous-même un message. Vous contrôlez ainsi complètement l’expéditeur, le destinataire et le point de soumission :

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
| `-SmtpServer` | Hôte cible de la soumission, ici volontairement sous forme d’adresse IP afin d’atteindre un point de terminaison précis |
| `-Port 25` | Port cible ; 25 pour une soumission non authentifiée de serveur à serveur |
| `-From` | Expéditeur d’enveloppe et d’en-tête du message de test |
| `-To` | Adresse du destinataire |
| `-Subject` | Ligne d’objet |
| `-Body` | Corps du message |
| `-Encoding UTF8` | Encodage des caractères pour l’objet et le texte, évite les problèmes d’accents |

</details>

`Send-MailMessage` est officiellement obsolète, mais reste l’outil le plus rapide à des fins de diagnostic et est présent sur chaque serveur Windows. En cas de réussite, il n’y a aucune sortie, ce qui peut être déroutant.

Si vous testez un chemin TLS sur le port 587 et que le système distant présente un certificat ne correspondant pas au nom utilisé, par exemple parce que vous contactez l’adresse IP, l’appel échoue avec une erreur de certificat. Pour le test, vous pouvez suspendre la vérification dans la session :

```powershell
[Net.ServicePointManager]::ServerCertificateValidationCallback = { $true }
```

Cela ne s’applique qu’à la session PowerShell en cours. Faites-le consciemment et jamais dans des scripts exécutés en production.

Si le message de test arrive et que vous voulez savoir ce qui lui est arrivé en chemin, l’[analyseur d’en-têtes de courrier](https://rafaelpfister.ch/tools/header-analyzer) vous aide : il décompose les en-têtes, retrace le chemin à travers les sauts et affiche les résultats des vérifications d’authentification, entièrement localement dans le navigateur, sans que le message quitte votre appareil.

## Exchange Online : même question, autre outil

Dans le tenant, les règles diffèrent, et c’est là que les méthodes habituelles échouent. Attendez-vous aux différences suivantes :

| | Exchange OnPrem | Exchange Online |
|---|---|---|
| Requête | `Get-MessageTrackingLog` | `Get-MessageTraceV2` |
| Granularité | Chaque événement de transport | Une ligne par message et destinataire |
| Connecteur visible | Oui, avec restrictions, voir ci-dessus | **Non** |
| Dépendance au serveur | Oui, à interroger par serveur | Sans objet |
| Journal SMTP | Disponible | **Non disponible** |
| Conservation | Votre configuration | 90 jours via le cmdlet, au plus 10 jours par requête |
| Délai | Quasi immédiat | Quelques minutes |

Les trois conséquences les plus importantes en pratique : il n’y a **aucune association de connecteur**, vous vous appuyez sur `FromIP` et `ToIP`. Il n’y a **pas de journal SMTP**, la conversation SMTP ne peut pas être reconstruite. Et les données apparaissent **avec retard**, un message tout juste envoyé n’apparaît pas immédiatement.

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
| `-StartDate` | Limite inférieure de la requête, ici les quatre dernières heures |
| `-EndDate` | Limite supérieure ; le cmdlet exige les deux limites |
| `-RecipientAddress` | Filtre les messages destinés à cette adresse de destinataire |
| `-ResultSize 1000` | Nombre maximal de lignes sur cette page ; la limite est de 5 000 |

</details>

```text
Received            SenderAddress          RecipientAddress          Status    FromIP
--------            -------------          ----------------          ------    ------
11.08.2026 08:27:16 emma@partner.example   empfaenger@example.com    Delivered 10.0.20.23
11.08.2026 09:05:24 dienst@example-test.com empfaenger@example.com   Failed    10.0.20.23
```

Les valeurs les plus importantes de `Status` : `Delivered`, `Failed`, `Pending`, `Quarantined`, `FilteredAsSpam` et `Expanded` pour les listes de distribution développées. `Pending` signifie que des tentatives de distribution sont encore en cours, non que quelque chose est défectueux.

### Les détails d’un message

Le statut seul n’indique rien sur la raison. Pour cela, vous avez besoin de la vue détaillée, qui requiert l’identifiant du message issu de la requête de base :

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

Vous y trouverez les étapes de traitement dans le service, par exemple les applications de règles, les décisions de filtrage et la raison d’un rejet.

### Au-delà de dix jours

`Get-MessageTraceV2` fournit les données des 90 derniers jours, mais au maximum 10 jours par requête ; répartissez les périodes plus longues sur plusieurs requêtes. Il existe également la recherche historique, qui s’exécute de manière asynchrone et fournit le résultat en CSV, pour une plage allant jusqu’à 90 jours :

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
| `-ReportTitle` | Nom librement choisi pour la tâche, sous lequel le résultat pourra ensuite être retrouvé |
| `-StartDate`, `-EndDate` | Période examinée, jusqu’à 90 jours en arrière |
| `-ReportType MessageTrace` | Type de rapport ; `MessageTrace` fournit la vue d’ensemble des messages au format CSV |
| `-SenderAddress` | Filtre cette adresse d’expéditeur |
| `-NotifyAddress` | Destinataire de la notification de fin ; doit être une adresse d’un domaine accepté du tenant |

</details>

Prévoyez du temps : ces tâches peuvent prendre des heures selon leur étendue.

### Source d’erreur 4 : l’absence de résultats ne prouve pas l’absence de trafic

C’est la source d’erreur la plus subtile dans le tenant. `Get-MessageTraceV2` fournit les résultats par pages, au maximum 5 000 lignes par appel. En cas de trafic élevé, une page peut ne couvrir que quelques minutes, même si vous avez interrogé sept jours. Si vous filtrez ensuite localement, par exemple selon une IP source, vous ne filtrez qu’un extrait minuscule.

Vous le reconnaissez à l’avertissement indiquant l’existence de résultats supplémentaires :

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

S’il apparaît, votre analyse est **incomplète**. Si aucun résultat n’est renvoyé, la conclusion correcte est : non trouvé dans l’extrait. Elle n’est pas : n’existe pas.

Il existe deux solutions propres. Soit vous réduisez la fenêtre temporelle jusqu’à ce qu’une page la couvre entièrement, ce que confirme l’absence de l’avertissement. Soit vous parcourez toutes les pages à l’aide des indications de continuation figurant dans l’avertissement. Pour répondre à la question de savoir si quelque chose ne se produit **jamais**, une vérification de configuration est de toute façon supérieure : si un système ne possède aucune route vers une destination, il ne peut pas y distribuer, indépendamment de toute fenêtre d’observation.

L’analyse complète de toutes les adresses de soumission est un sujet à part entière, avec ses propres points délicats d’interprétation. Elle est présentée dans [Qui soumet réellement des messages dans votre tenant ? Agréger les adresses IP de soumission](https://rafaelpfister.ch/blog/einliefernde-ip-adressen-aggregieren).

## Une méthode qui a fait ses preuves

En résumé, cet ordre s’est révélé le plus rapide. Rechercher le message sur tous les serveurs et déterminer le dernier événement. En cas d’échec, passer immédiatement à `Format-List` et lire la réponse SMTP complète, plutôt que de tirer une conclusion du type d’événement. Déterminer ensuite l’étendue, donc regrouper et compter. Ce n’est que lorsque le cas est circonscrit que vous reconstruisez le chemin de soumission via la configuration des connecteurs et le journal SMTP. Enfin, si nécessaire, vérifiez avec votre propre soumission.

Les pertes de temps les plus fréquentes sont toujours les mêmes : lire un tableau tronqué au lieu du message d’erreur complet, prendre des copies fantômes pour des étapes de traitement, croire au `ConnectorId` du suivi, et considérer un échantillon vide comme une preuve. Celui qui connaît ces quatre pièges arrive généralement au bon niveau en quelques minutes.

## Historique des modifications

- **25 octobre 2026 :** Correction : Get-MessageTraceV2 fournit les données des 90 derniers jours, mais au maximum 10 jours par requête. Liens vers les sources mis à jour vers les nouvelles adresses Microsoft Learn.

## Sources

1.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking) : description des champs et liste complète des types d’événements dans le suivi des messages.

2.  [Protocol logging in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/protocol-logging) : emplacements, format et conservation des journaux SMTP, y compris Front End Transport.

3.  [Shadow redundancy in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/shadow-redundancy) : explique les événements liés aux copies fantômes et à leur suppression.

4.  [Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing) : interaction entre Front End Transport et Transport Service, base du comportement de proxy.

5.  [Receive connectors in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/receive-connectors) : liaisons, groupes d’autorisations et mécanismes d’authentification.

6.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2) : successeur de Get-MessageTrace, avec logique de pagination, liste des champs et plage de requête (90 jours, au maximum 10 jours par requête).

7.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/start-historicalsearch) : suivi asynchrone des messages sur une période allant jusqu’à 90 jours.
