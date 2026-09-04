---
title: "Combien de temps une session SMTP reste-t-elle ouverte ? ConnectionTimeout 00:10:00 dans Exchange et les systèmes pour lesquels c’est trop court"
navTitle: "Durée de session SMTP"
description: "Exchange met fin à chaque session SMTP entrante après dix minutes, même si elle transfère encore des données. Quels expéditeurs restent aussi longtemps sur une connexion, comment lire la durée réelle de la session dans le journal de protocole et quand adapter ConnectionTimeout et ConnectionInactivityTimeout sur un connecteur de relais."
date: "2026-09-03"
kategorie: "SMTP et flux de messagerie"
timeToRead: "10 min de lecture"
themen:
  - smtp-mailflow
  - exchange-onprem-hybrid
produkte:
  - "exchange-on-premises"
  - "exchange"
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "combien-de-temps-une-session-smtp-reste-t-elle-ouverte-connectiontimeout-00-10-00-dans-exchange"
translationId: "article-b40497933bbe0a88"
aiPrompt: |
  Du bist mein Exchange- und Mailflow-Assistent. Hilf mir, die SMTP-Session-Dauer auf einem Exchange-Receive-Connector zu beurteilen: 1. Frage mich, welche Systeme (Relays, Gateways, Applikationen, Scanner) über den Connector einliefern und ob sie Verbindungen über mehrere Nachrichten hinweg offen halten. 2. Lass dir die Ausgabe der Session-Auswertung aus dem Protokoll-Log geben (IP, Mails, Dauer, Timeout-Kennzeichen) und erkläre mir, welche Sessions am ConnectionTimeout abgebrochen wurden. 3. Empfiehl pro Connector konkrete Werte für ConnectionTimeout und ConnectionInactivityTimeout und begründe, warum der internetseitige Connector unverändert bleibt. 4. Nenne mir, was ich stattdessen auf der Client-Seite ändern kann, damit die Verbindung nach einer festen Anzahl Nachrichten neu aufgebaut wird.
translationOf: smtp-session-dauer-exchange-connectiontimeout
url: https://rafaelpfister.ch/fr/blog/combien-de-temps-une-session-smtp-reste-t-elle-ouverte-connectiontimeout-00-10-00-dans-exchange
translationSourceHash: a107c4edd960dabb30ba1b6f263a693808a5edf6815747d81f5d446c103a7e79
translationModel: gpt-5.6-terra
translatedAt: 2026-09-04T08:20:12.643Z
translationReview: automatic
---

# Combien de temps une session SMTP reste-t-elle ouverte ? ConnectionTimeout 00:10:00 dans Exchange et les systèmes pour lesquels c’est trop court

En bref : une session SMTP n’a pas de fin naturelle. La RFC 5321 limite uniquement le temps d’attente pour l’étape suivante, et un client peut continuer à remettre des messages sur une connexion ouverte aussi longtemps que le serveur la maintient ouverte. Exchange la maintient ouverte pendant dix minutes par défaut sur les connecteurs de réception, puis le serveur ferme la connexion, que des données soient en cours de transfert ou non. Pour le trafic Exchange-à-Exchange et pour la plupart des MTA, cela est sans importance, car ces expéditeurs se reconnectent d’eux-mêmes après quelques secondes. Pour les applications, les passerelles et les générateurs de charge qui utilisent une seule connexion pour tout un envoi, cette valeur est en revanche la cause d’interruptions qui apparaissent côté client comme des erreurs de connexion et dans le journal de protocole Exchange comme `421 4.4.1 Connection timed out`.

## Deux délais d’expiration aux significations différentes

Un connecteur de réception possède deux limites de temps souvent confondues :

| Paramètre | Signification | Valeur par défaut serveur de boîtes aux lettres | Valeur par défaut Edge Transport |
|---|---|---|---|
| `ConnectionInactivityTimeout` | durée maximale d’inactivité sans activité du client, après laquelle la connexion est fermée | 00:05:00 | 00:01:00 |
| `ConnectionTimeout` | durée totale maximale de la connexion, même lorsqu’elle transfère activement des données | 00:10:00 | 00:05:00 |

Les deux valeurs acceptent de 1 seconde à 1 jour (`1.00:00:00`), et `ConnectionTimeout` doit être supérieur à `ConnectionInactivityTimeout`. Les valeurs s’appliquent par connecteur, donc séparément pour le connecteur côté Internet `Default Frontend <Server>`, pour le connecteur du service de transport `Default <Server>` sur le port 2525 et pour chaque connecteur de relais créé manuellement.

Le délai d’inactivité n’est pas problématique : cinq minutes correspondent exactement au minimum que la RFC 5321 impose à un serveur comme délai d’attente de la commande suivante, et un client qui n’envoie rien pendant cinq minutes a en règle générale lui-même oublié la connexion. Le délai total est la particularité d’Exchange : il commence au moment de l’établissement de la connexion et continue de s’écouler pendant que le client remet message après message. Après dix minutes, Exchange ferme la connexion au point où se trouve le dialogue, au besoin au milieu d’un bloc `DATA`.

Côté émission, il n’existe pas d’équivalent : un connecteur d’envoi ne possède que `ConnectionInactivityTimeOut` (dix minutes par défaut) et limite les sessions via `SmtpMaxMessagesPerConnection`, à 20 messages par défaut. En tant que client, Exchange ferme donc lui-même chaque connexion après 20 messages au plus tard et en établit une nouvelle. C’est la raison pour laquelle le délai total ne se remarque jamais entre serveurs Exchange : les sessions ne durent que quelques secondes.

## Ce que prescrit la RFC 5321

La norme définit, à la section 4.5.3.2, les temps d’attente minimaux qu’un client doit respecter pour chaque étape du protocole avant d’abandonner la connexion :

| Étape | Délai minimal côté client |
|---|---|
| Attente de la bannière `220` | 5 minutes |
| Réponse à `MAIL` | 5 minutes |
| Réponse à `RCPT` | 5 minutes |
| Réponse à `DATA` (le `354`) | 2 minutes |
| Envoi d’un bloc de données | 3 minutes |
| Réponse au point final | 10 minutes |
| Serveur : attente de la commande suivante | au moins 5 minutes |

La RFC ne fixe aucune limite supérieure à la durée totale d’une session. Un client qui remet des messages pendant trente minutes sur la même connexion sans jamais rester silencieux plus de quelques secondes respecte la norme. La dernière valeur côté client est notable : dix minutes d’attente de la réponse après le point final, car c’est durant cette phase que le serveur accepte et prend en charge le message. Si le client abandonne trop tôt à ce stade, le message est déjà remis et sera livré une seconde fois lors de la tentative suivante. La même situation se produit en miroir lorsque le serveur ferme la connexion à ce moment-là en raison du délai total.

Si un serveur ferme la connexion avec `421`, le client doit traiter la transaction en cours conformément à la section 3.8 comme s’il avait reçu un `451`, c’est-à-dire comme une erreur temporaire avec nouvelle tentative. Un MTA avec file d’attente fait exactement cela. Une application sans file d’attente signale au contraire une exception et laisse le reste à l’appelant.

## Combien de temps les expéditeurs gardent réellement leurs sessions ouvertes

La durée de session est déterminée par le client, et les différences entre les types d’expéditeurs sont importantes :

| Expéditeur | Durée de session typique | Limitée par |
|---|---|---|
| Connecteur d’envoi Exchange | Secondes | `SmtpMaxMessagesPerConnection` = 20 |
| Postfix avec cache de connexions | 5 minutes maximum | `smtp_connection_reuse_time_limit` = 300s |
| Postfix sans cache de connexions | un message par connexion | comportement standard du client `smtp` |
| Application avec `.NET SmtpClient`, `JavaMail Transport`, Python `smtplib` | aussi longtemps que l’objet existe : pour une exécution par lots, toute l’exécution | uniquement par le code du programme |
| Notifications de quarantaine de passerelles de messagerie | une session par cycle de notification | comportement du produit, en partie avec keepalive `NOOP` |
| Appareils multifonctions, scan-to-mail | un message par connexion, plusieurs minutes pour les numérisations volumineuses sur des liaisons lentes | taille de fichier et bande passante |
| Générateurs de charge tels que `smtp-source -d` | jusqu’à la fin de l’exécution | paramètres d’appel |

Les deux premières lignes expliquent pourquoi la valeur passe inaperçue pendant des années dans les environnements classiques : les MTA établissent d’eux-mêmes des connexions courtes. Postfix, par exemple, utilise une connexion mise en cache pendant cinq minutes au maximum avant d’en ouvrir une nouvelle, et Exchange se déconnecte après 20 messages. Les deux restent donc sous toute valeur standard d’Exchange.

La ligne des applications est le cas problématique le plus fréquent. Un travail par lots qui envoie des factures, des fiches de paie ou des messages système crée typiquement un objet client, appelle sa méthode d’envoi dans une boucle et le ferme à la fin. `System.Net.Mail.SmtpClient` utilise depuis .NET Framework 4 la même connexion pour des appels `Send` successifs et n’envoie `QUIT` qu’au moment de `Dispose`; JavaMail se comporte de la même manière avec un `Transport` ouvert une seule fois. Si le travail dure plus de dix minutes, le `421` survient quelque part au milieu et le travail s’interrompt avec une exception, par exemple dans .NET avec le texte `Service not available, closing transmission channel. The server response was: 4.4.1 Connection timed out`. Le message concerné dépend de la durée d’exécution, ce qui donne l’impression que l’erreur est aléatoire : parfois l’interruption survient après 800 messages, parfois après 1200, selon la taille des messages et la charge du serveur.

La ligne des passerelles décrit un cas documenté : Symantec (aujourd’hui Broadcom) Messaging Gateway envoie les notifications de quarantaine antispam via une seule connexion et transmet `NOOP` entre les messages comme keepalive. Exchange répond à `NOOP` avec le délai de tarpit de cinq secondes, de sorte qu’en dix minutes, environ 120 notifications au maximum passent avant que la session ne se termine par `421 4.4.1` et que la passerelle doive se reconnecter.

La ligne des scanners relève d’un problème de taille plutôt que de volume : une numérisation de 60 Mo via une connexion de 2 Mbit/s nécessite environ quatre minutes de transfert pur ; avec 100 Mo, cela fait presque sept minutes. Sur un serveur Edge Transport avec un délai total de cinq minutes, cela suffit déjà à provoquer une interruption ; sur un serveur de boîtes aux lettres, il reste une marge, mais faible.

## Ce qui se passe lors de l’interruption

Lorsque le délai total expire, Exchange écrit la réponse `421 4.4.1 Connection timed out` dans le journal de protocole, l’envoie au client et ferme la connexion. Pour la transaction en cours : si le point final n’a pas encore été envoyé, le message n’est pas accepté et doit être entièrement répété. Si le point a été envoyé et que la connexion est fermée avant la réponse `250`, le client ne sait pas si Exchange a pris en charge le message ; un client correctement implémenté le répète, et le destinataire peut alors le recevoir en double. La probabilité est faible, mais elle n’est pas nulle avec des milliers de messages par exécution.

Il faut également tenir compte du chemin via proxy : le service de transport frontal accepte la connexion sur le port 25 et la transmet comme sa propre session SMTP au service de transport sur le port 2525, où le connecteur `Default <Server>` applique les mêmes valeurs par défaut. Une longue session apparaît donc dans les deux journaux, et l’adaptation doit couvrir les deux connecteurs.

## Lire la durée réelle de session depuis le journal de protocole

Avant de modifier une valeur, il vaut la peine d’examiner les sessions réelles. Cela nécessite une journalisation de protocole détaillée sur le connecteur concerné ; elle est déjà active sur `Default Frontend <Server>`, mais pas sur tous les autres connecteurs :

```powershell
Set-ReceiveConnector -Identity 'EX01\Relay Applikationen' -ProtocolLoggingLevel Verbose
```

Les journaux se trouvent sous `%ExchangeInstallPath%TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive` (frontal) et `%ExchangeInstallPath%TransportRoles\Logs\Hub\ProtocolLog\SmtpReceive` (service de transport), et sont nommés d’après l’heure UTC sous la forme `RECVyyyyMMddhh-nnnn.log`. Chaque ligne est un événement de protocole comportant les champs `date-time`, `connector-id`, `session-id`, `sequence-number`, `local-endpoint`, `remote-endpoint`, `event`, `data` et `context`. Toutes les lignes d’une session portent le même `session-id`, la durée de la session est donc la différence entre le premier et le dernier horodatage de cet ID.

Le script suivant évalue le fichier journal le plus récent de la journée pour un connecteur, regroupe les lignes par session et affiche les 15 sessions les plus longues avec le nombre de messages, la durée et l’indication qu’Exchange les a terminées avec `421 4.4.1` :

```powershell
$logPfad = Join-Path $env:ExchangeInstallPath 'TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive'
$connector = 'Relay Applikationen'
$tag = (Get-Date).ToUniversalTime().ToString('yyyyMMdd')
$datei = Get-ChildItem $logPfad -Filter "RECV$tag*.log" |
    Sort-Object Name -Descending |
    Select-Object -First 1

$sessions = @{}
Get-Content $datei.FullName |
    Where-Object { $_ -notlike '#*' -and $_ -like "*$connector*" } |
    ForEach-Object {
        $c = $_ -split ','
        $s = $c[2]
        if (-not $sessions[$s]) {
            $sessions[$s] = [pscustomobject]@{
                IP = ($c[5] -split ':')[0]; Start = $c[0]; Ende = $c[0]
                Zeilen = 0; Mails = 0; Timeout = $false
            }
        }
        $sessions[$s].Ende = $c[0]
        $sessions[$s].Zeilen++
        if ($c[7] -like 'MAIL FROM*') { $sessions[$s].Mails++ }
        if ($c[7] -like '421 4.4.1*') { $sessions[$s].Timeout = $true }
    }

$sessions.Values |
    Sort-Object Zeilen -Descending |
    Select-Object -First 15 IP, Mails, Zeilen, Timeout,
        @{ n = 'Dauer_s'
           e = { [math]::Round(([datetime]$_.Ende - [datetime]$_.Start).TotalSeconds, 1) } } |
    Format-Table -AutoSize
```

<details class="options-details">
<summary>Options expliquées</summary>

| Élément | Effet |
|---|---|
| `$logPfad` | répertoire des journaux du service de transport frontal ; pour le service de transport, utiliser `Hub` au lieu de `FrontEnd` |
| `$connector` | composant du nom du connecteur ; filtre via le champ `connector-id`, journalisé sous la forme `Server\Name` |
| `$tag` | date UTC, car les fichiers journaux sont nommés selon l’heure UTC |
| `-Filter "RECV$tag*.log"` | uniquement les journaux de réception du jour courant |
| `Sort-Object Name -Descending`, `Select-Object -First 1` | le fichier le plus récent (heure la plus élevée, numéro d’instance le plus élevé) |
| `$_ -notlike '#*'` | ignore les en-têtes `#Software`, `#Version`, `#Log-Type`, `#Date`, `#Fields` |
| `$_ -split ','` | décompose la ligne CSV ; les champs utilisés 0, 2, 5 et 7 se situent avant le premier texte libre et sont donc stables |
| `$c[2]` | `session-id`, la clé de regroupement |
| `($c[5] -split ':')[0]` | adresse IPv4 issue de `remote-endpoint` (pour les points de terminaison IPv6, il faut adapter le découpage) |
| `$c[0]` comme `Start` et `Ende` | premier et dernier horodatage de la session ; `Ende` est écrasé à chaque ligne |
| `$c[7] -like 'MAIL FROM*'` | compte les messages via la commande reçue `MAIL FROM` |
| `$c[7] -like '421 4.4.1*'` | marque les sessions qu’Exchange a terminées en raison du délai total |
| `Sort-Object Zeilen -Descending` | les sessions les plus actives en premier ; à défaut, trier par `Dauer_s` |
| `Dauer_s` | différence des horodatages ISO 8601 en secondes, arrondie à une décimale |

</details>

Dans la sortie, vous identifiez les systèmes concernés au fait que `Timeout` est défini sur `True` et que `Dauer_s` est proche de 600 : la session a vécu exactement aussi longtemps que le connecteur l’autorise. Les sessions comportant de nombreux messages et d’une durée nettement inférieure à 600 secondes ne sont pas problématiques, même si elles sont à ce moment les plus longues. Pour obtenir une vue d’ensemble des sources concernées, il suffit de regrouper les sessions marquées :

```powershell
$sessions.Values |
    Where-Object { $_.Timeout } |
    Group-Object IP |
    Select-Object Name, Count |
    Sort-Object Count -Descending
```

Deux limites de l’approche : une session qui franchit une limite d’heure est répartie sur deux fichiers journaux et apparaît raccourcie dans un fichier unique ; pour une évaluation quotidienne, lisez tous les fichiers de la journée. Et la valeur `Mails` compte les commandes `MAIL FROM`, donc des tentatives, non des messages acceptés.

## Adapter les valeurs : sur quel connecteur et dans quelle mesure

Les valeurs par défaut protègent le connecteur côté Internet, sur lequel des pairs arbitraires peuvent occuper des connexions. Elles restent inchangées à cet endroit ; un MTA externe légitime se reconnecte de toute façon. C’est le connecteur dédié par lequel les systèmes internes identifiés remettent les messages qui est adapté. En l’absence d’un tel connecteur, il peut être créé en le limitant aux IP des expéditeurs avec `RemoteIPRanges` ; c’est préférable à augmenter la valeur sur `Default Frontend`. L’état actuel de tous les connecteurs est fourni par :

```powershell
Get-ReceiveConnector |
    Format-Table Name, TransportRole, ConnectionTimeout, ConnectionInactivityTimeout, TarpitInterval -AutoSize
```

L’adaptation elle-même, ici avec une durée totale d’une heure et un délai d’inactivité inchangé :

```powershell
$werte = @{
    ConnectionTimeout           = '01:00:00'
    ConnectionInactivityTimeout = '00:05:00'
}
Set-ReceiveConnector -Identity 'EX01\Relay Applikationen' @werte
Set-ReceiveConnector -Identity 'EX01\Default EX01' @werte
```

<details class="options-details">
<summary>Options expliquées</summary>

| Paramètre | Effet |
|---|---|
| `ConnectionTimeout` | durée totale d’une connexion ; autorisée de 00:00:01 à 1.00:00:00, doit être supérieure à `ConnectionInactivityTimeout` |
| `ConnectionInactivityTimeout` | durée d’inactivité avant fermeture ; cinq minutes correspondent au minimum de la RFC et peuvent être conservées |
| `-Identity 'EX01\Relay Applikationen'` | le connecteur frontal des expéditeurs internes |
| `-Identity 'EX01\Default EX01'` | le connecteur du service de transport sur le port 2525 auquel le frontal transmet la session |
| `@werte` | splatting : transmet les deux paramètres de la table de hachage au cmdlet |

</details>

Pour la valeur : elle doit être supérieure à la session légitime la plus longue révélée par l’analyse, avec une marge pour les pics de charge. Une heure couvre la plupart des exécutions par lots ; pour une exécution nocturne de deux heures, il faut proportionnellement davantage, jusqu’au maximum d’un jour. La valeur ne devrait toutefois pas être arbitrairement élevée, même sur un connecteur interne, car `MaxInboundConnectionPerSource` (20 par défaut) et `MaxInboundConnection` (5000 par défaut) sont également comptabilisés : un client qui ouvre sans cesse de nouvelles connexions en plus d’une connexion bloquée atteint la limite par source d’autant plus tôt que les anciennes connexions restent ouvertes longtemps.

Pour les expéditeurs qui envoient `NOOP` entre les messages, `TarpitInterval` devrait être réglé sur `00:00:00` sur le même connecteur. Le délai de tarpit ne présente aucun intérêt pour les expéditeurs internes authentifiés ou restreints par IP et allonge artificiellement chaque session.

La modification côté Exchange corrige le symptôme. La solution plus robuste se situe côté client : il établit une nouvelle connexion après un nombre fixe de messages, comme Exchange le fait avec 20 et Postfix avec cinq minutes. Avec `.NET SmtpClient`, cela signifie créer et abandonner l’objet pour chaque bloc de, par exemple, 100 messages ; avec JavaMail, le `Transport` est fermé puis rouvert en conséquence. L’envoi fonctionne ainsi aussi vers des destinations dont les délais d’expiration ne peuvent pas être adaptés, notamment Exchange Online, dont les connecteurs entrants ne disposent pas de paramètres de délai d’expiration.

## Autres limites de temps sur le chemin

La valeur Exchange n’est pas la seule limite. Les pare-feu et les répartiteurs de charge possèdent leurs propres temporisateurs d’inactivité pour les connexions TCP : un profil FastL4 sur un F5 BIG-IP est réglé par défaut sur 300 secondes, un Azure Load Balancer sur quatre minutes. Ces temporisateurs mesurent l’inactivité, non la durée totale, et interviennent donc lors de pauses d’envoi, par exemple lorsqu’un travail par lots lit des données de la base de données entre deux blocs. La plus petite valeur sur l’ensemble du chemin est toujours déterminante. L’article [F5 BIG-IP comme proxy sortant pour l’envoi massif d’e-mails](https://rafaelpfister.ch/blog/f5-big-ip-outbound-smtp-massenversand) décrit comment dimensionner les délais d’expiration sur un répartiteur de charge pour des connexions SMTP persistantes.

## Sources

1.  [Microsoft Learn: Set-ReceiveConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-receiveconnector) : référence avec les valeurs par défaut et les plages de valeurs de `ConnectionTimeout`, `ConnectionInactivityTimeout`, `TarpitInterval`, `MaxInboundConnection` et `MaxInboundConnectionPerSource` pour les serveurs de boîtes aux lettres et Edge Transport.

2.  [Microsoft Learn: Set-SendConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-sendconnector) : `ConnectionInactivityTimeOut` et `SmtpMaxMessagesPerConnection` côté envoi.

3.  [Microsoft Learn: Protocol logging](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/protocol-logging) : emplacements de stockage, noms de fichiers et structure des champs des journaux de protocole SMTP pour le frontal et le service de transport.

4.  [Microsoft Learn: Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing) : le service de transport frontal comme proxy sans état devant le service de transport.

5.  [RFC 5321, section 4.5.3.2 Timeouts](https://www.rfc-editor.org/rfc/rfc5321.html#section-4.5.3.2) : temps d’attente minimaux par étape de protocole, justification des dix minutes après le point final et comportement en cas de `421` à la section 3.8.

6.  [Postfix: postconf(5)](https://www.postfix.org/postconf.5.html) : `smtp_connection_reuse_time_limit` (300s) et `smtpd_timeout` comme exemple d’un MTA qui garde de lui-même les sessions courtes.

7.  [Broadcom Knowledge Base: Quarantine notification process appears to be failing, logs may show 421 4.4.1 Connection timed out](https://knowledge.broadcom.com/external/article/154389/quarantine-notification-process-appears.html) : cas documenté d’une passerelle qui atteint le délai total d’Exchange avec un keepalive `NOOP` et le tarpit.

8.  [Microsoft Learn: SmtpClient Class](https://learn.microsoft.com/en-us/dotnet/api/system.net.mail.smtpclient) : réutilisation de connexion sur plusieurs appels `Send` et `QUIT` uniquement lors de `Dispose`.
