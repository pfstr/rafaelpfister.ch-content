---
title: "Déterminer le profil de charge d’un serveur de messagerie : rafales, débits de pointe et structure des destinataires à partir du Message Tracking"
navTitle: "Déterminer le profil de charge"
description: "Combien d’e-mails par minute votre serveur de messagerie traite-t-il réellement, et quels sont les pics ? Comment déterminer le véritable profil de charge à partir de l’Exchange Message Tracking avec PowerShell : débits par minute et par heure, durée des rafales, structure des destinataires, tailles des messages. Avec les pièges d’analyse typiques."
date: "2026-08-25"
kategorie: "SMTP et flux de messagerie"
timeToRead: "9 min de lecture"
themen:
  - smtp-mailflow
  - exchange-onprem-hybrid
produkte:
  - "exchange-on-premises"
  - "uebergreifend"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
slug: "determiner-le-profil-de-charge-d-un-serveur-de-messagerie-rafales-debits-de-pointe-et-structure"
translationId: "article-1ff17a188d73e289"
aiPrompt: |
  Du bist mein Mailflow-Assistent. Hilf mir Schritt für Schritt, das Lastprofil meines Mailservers zu ermitteln: 1. Die richtige Datenquelle wählen (Message Tracking, Gateway-Logs) und das passende Event pro Nachricht bestimmen. 2. Raten pro Minute, Stunde und Tag berechnen und Bursts mit Dauer und Peak charakterisieren. 3. Empfängerstruktur, Domain-Verteilung und Nachrichtengrössen auswerten. Weise mich auf Doppelzählungen, Export-Limits und Zeitzonen-Fallen hin.
translationOf: mailserver-lastprofil-ermitteln
url: https://rafaelpfister.ch/fr/blog/determiner-le-profil-de-charge-d-un-serveur-de-messagerie-rafales-debits-de-pointe-et-structure
translationSourceHash: 16095cf53ce6f67abe31387ce2f02958eacc3898d3a42b61ad8c7b885ab7ce5d
translationModel: gpt-5.6-terra
translatedAt: 2026-08-26T04:09:40.290Z
translationReview: automatic
---

# Déterminer le profil de charge d’un serveur de messagerie : rafales, débits de pointe et structure des destinataires à partir du Message Tracking

Qu’il s’agisse de remplacer une passerelle, de dimensionner un serveur ou de planifier une fenêtre de maintenance : tôt ou tard, chaque administrateur de messagerie doit répondre à la question de savoir quel volume son système traite réellement. L’intuition est régulièrement trompeuse, car le trafic de messagerie est rarement uniforme. Un système qui reçoit en moyenne 20 e-mails par minute sur la journée peut devoir en traiter 400 par minute pendant une heure lors d’un traitement de facturation. Se limiter à la moyenne revient à dimensionner à côté du véritable problème.

Un profil de charge exploitable repose sur quatre indicateurs : le débit moyen (par minute, heure, jour), les rafales (quelle est la pointe, combien de temps dure-t-elle, quand survient-elle), la structure des destinataires (combien de destinataires distincts, quels domaines de destination) et la taille des messages. Les quatre figurent dans le Message Tracking et, sur Exchange, peuvent être calculés en quelques lignes de PowerShell.

## La source de données : Message Tracking

Exchange consigne chaque message dans le Message Tracking Log. Avant de procéder à l’analyse, vérifiez jusqu’à quelle date remontent les données ; la valeur par défaut est de 30 jours, mais une limite de taille trop restreinte peut réduire considérablement la conservation réelle :

```powershell
Get-TransportService |
    Select-Object Name, MessageTrackingLogMaxAge,
        MessageTrackingLogMaxDirectorySize, MessageTrackingLogPath
```

Pour un profil de charge, la période devrait couvrir au moins un cycle complet de traitement par lots de l’entreprise : traitements mensuels de facturation, décomptes de salaire, newsletters. Une semaine est le minimum, un mois est préférable.

## Collecter les données brutes : un événement par message

La décision préliminaire la plus importante : quel événement compte comme « un e-mail » ? Le Message Tracking écrit plusieurs entrées par message (RECEIVE lors de la réception, SEND lors de la transmission au saut suivant, DELIVER lors de la remise en boîte aux lettres, ainsi que AGENTINFO, HAREDIRECT et d’autres). Compter simplement toutes les lignes surestime le volume d’un multiple. Pour la charge d’entrée, comptez RECEIVE ; pour la charge de sortie vers un smarthost ou Internet, comptez SEND.

```powershell
$start = (Get-Date).AddDays(-7)
$events = Get-TransportService | ForEach-Object {
    Get-MessageTrackingLog -Server $_.Name -ResultSize Unlimited `
        -Start $start -EventId RECEIVE
}
"{0} Nachrichten seit {1:yyyy-MM-dd}" -f $events.Count, $start
```

La requête s’exécute délibérément sur tous les serveurs de transport, car chaque serveur ne consigne que sa propre part. Interroger un seul serveur ne montre qu’une fraction de la charge dans un cluster.

## Débits par minute et par heure : c’est ici que les rafales apparaissent

L’agrégation consiste en un Group-Object sur l’horodatage arrondi. Les minutes les plus chargées sont directement vos candidates aux rafales :

```powershell
$proMinute = $events |
    Group-Object { $_.Timestamp.ToString("yyyy-MM-dd HH:mm") } |
    Sort-Object Count -Descending

$proMinute | Select-Object -First 10 Name, Count
```

Même principe par heure et sous forme de profil journalier (à quelle heure la charge est-elle habituellement la plus forte) :

```powershell
$events |
    Group-Object { $_.Timestamp.ToString("yyyy-MM-dd HH") } |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count

$events |
    Group-Object { $_.Timestamp.ToString("HH") } |
    Sort-Object Name |
    Format-Table Name, Count
```

Une rafale n’est caractérisée que lorsque vous connaissez sa durée en plus de son pic. Un pic de 400/min qui dure deux minutes représente une exigence différente du même pic durant une heure. Comptez les minutes au-dessus d’un seuil :

```powershell
$schwelle = 100
$burstMinuten = $proMinute | Where-Object Count -ge $schwelle
"{0} Minuten mit >= {1}/min, Peak: {2}/min" -f $burstMinuten.Count,
    $schwelle, ($proMinute | Select-Object -First 1).Count
```

Si les minutes de rafale sont consécutives (directement visible dans la sortie de `$burstMinuten | Sort-Object Name`), il s’agit d’un traitement par lots. Notez l’heure de début, la durée et le schéma de répétition, car c’est précisément cette fenêtre que l’infrastructure doit supporter.

## Structure des destinataires : combien de destinations, quels domaines

Pour les passerelles, la diversité des destinataires est souvent plus importante que le simple débit, car chaque destinataire entraîne des recherches (routage, politiques, règles de chiffrement). Un e-mail adressé à une liste de diffusion de 5'000 membres ne sollicite pas le système comme 5'000 e-mails individuels. Le champ `RecipientCount` et la liste des destinataires fournissent les deux perspectives :

```powershell
"Nachrichten: {0}, Empfänger-Zustellungen: {1}" -f $events.Count,
    ($events | Measure-Object RecipientCount -Sum).Sum

$alleEmpfaenger = $events | ForEach-Object { $_.Recipients } |
    ForEach-Object { $_.ToLower() }
"Eindeutige Empfänger: {0}" -f ($alleEmpfaenger | Sort-Object -Unique).Count
```

La répartition par domaine montre où circule le trafic. Si Gmail et Microsoft dominent, leurs limites de débit et votre propre réputation IP déterminent le débit atteignable, et non votre matériel :

```powershell
$alleEmpfaenger |
    ForEach-Object { ($_ -split "@")[1] } |
    Group-Object |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

Et dans l’autre sens : quels expéditeurs (applications, boîtes aux lettres fonctionnelles) génèrent réellement la charge ? Cela répond accessoirement à la question des systèmes à prendre en compte lors d’une migration :

```powershell
$events |
    Group-Object Sender |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

## Taille des messages : octets par seconde plutôt qu’e-mails par seconde

Les indications de débit des passerelles se rapportent souvent au volume de données, et non au nombre de messages. Deux systèmes avec le même débit d’e-mails diffèrent d’un facteur 100 si l’un envoie des notifications de 50 KB et l’autre des PDF de factures de 5 MB. Le champ `TotalBytes` fournit la répartition :

```powershell
$events | Measure-Object TotalBytes -Average -Maximum -Sum |
    Select-Object @{n = "MittelKB"; e = { [math]::Round($_.Average / 1KB) } },
        @{n = "MaxMB"; e = { [math]::Round($_.Maximum / 1MB, 1) } },
        @{n = "TotalGB"; e = { [math]::Round($_.Sum / 1GB, 1) } }
```

Multipliez le débit de rafale par la taille moyenne durant la fenêtre de rafale, et vous obtenez l’exigence de bande passante que doit supporter une nouvelle passerelle ou une liaison WAN.

## Débits en direct sans tracking : un regard sur les files d’attente

Pour une vue instantanée (le serveur traite-t-il actuellement beaucoup de messages, quelque chose s’accumule-t-il ?), vous n’avez pas besoin de tracking : les files d’attente l’indiquent directement. `IncomingRate` et `OutgoingRate` correspondent au nombre d’e-mails par minute, lissé sur les dernières minutes :

```powershell
Get-TransportService |
    ForEach-Object { Get-Queue -Server $_.Name } |
    Sort-Object MessageCount -Descending |
    Select-Object Identity, Status, MessageCount, IncomingRate,
        OutgoingRate, NextHopDomain, LastError |
    Format-Table -AutoSize
```

Interprétation : une file d’attente `Submission` avec un débit élevé et une profondeur de 0 signifie que le serveur traite la charge sans accumulation. `MessageCount` élevé avec `OutgoingRate` proche de zéro signifie un engorgement. `Status Retry` avec un message 4xx dans `LastError` signifie que le système distant limite le débit. Les files d’attente `Shadow` avec des messages en attente sont en revanche normales : ce sont des copies de redondance pour le serveur partenaire, pas un engorgement.

Pour obtenir une courbe continue durant une fenêtre de charge, le compteur de performance des files d’attente de transport convient bien, ici toutes les cinq secondes pendant une minute :

```powershell
Get-Counter "\MSExchangeTransport Queues(_total)\Messages Completed Delivery Per Second" `
    -SampleInterval 5 -MaxSamples 12
```

## Autres systèmes : le même principe avec CSV

Les passerelles et appliances fournissent généralement une exportation CSV du tracking plutôt que des objets PowerShell. La procédure reste identique (choisir un événement par e-mail, regrouper par plages horaires), seul l’outil change, par exemple pour Python :

```python
import csv, collections, datetime

per_min = collections.Counter()
with open("tracking-export.csv", encoding="utf-8") as f:
    reader = csv.reader(f)
    next(reader)
    for row in reader:
        if "response '2" not in row[6]:   # nur finale Zustellungen
            continue
        d = datetime.datetime.strptime(row[0][:16], "%Y-%m-%d %H:%M")
        per_min[d.strftime("%Y-%m-%d %H:%M")] += 1

print(per_min.most_common(10))
```

## Les cinq pièges classiques de l’analyse

**Événements multiples par e-mail.** La source d’erreur la plus fréquente : compter les lignes plutôt que les messages. Vérifiez avec `$events | Group-Object EventId` ce qui se trouve réellement dans votre jeu de données, puis filtrez sur exactement un événement par message.

**Exportations tronquées.** De nombreuses fonctions d’exportation ne renvoient qu’un maximum de 10'000 ou 50'000 lignes, puis tronquent silencieusement les données, volontiers au milieu de la plus grande rafale. Un nombre de lignes suspectement rond est un signal d’alarme. Vérifiez toujours que la période couverte par les données correspond à la période demandée.

**Boucles de passerelle.** Si le flux de messagerie passe par une station intermédiaire (passerelle de chiffrement, appliance d’hygiène) puis revient, le même e-mail apparaît plusieurs fois dans le tracking. Dédupliquez à l’aide du Message-ID ou filtrez sur un point unique de la chaîne.

**Fuseaux horaires.** `Get-MessageTrackingLog` fournit les horodatages dans l’heure locale du serveur, tandis que les exportations CSV des appliances sont souvent en UTC. Une rafale qui semble se produire à 13 heures peut en réalité être le traitement par lots de 15 heures. Clarifiez la base horaire avant toute interprétation.

**Fenêtres trop courtes.** Un profil de charge établi sur deux jours calmes ne vaut rien si le traitement mensuel de facturation manque. La fenêtre d’analyse doit inclure les cycles de traitement par lots connus ; demandez aux responsables applicatifs leurs calendriers d’envoi avant de définir la fenêtre.

## Ce que vous faites du profil

Au final, vous disposez de quatre chiffres sur une page : débit moyen, rafale (pic, durée, moment, schéma de répétition), structure des destinataires (destinataires uniques par traitement, principaux domaines) et répartition des tailles. Cela permet de dimensionner les passerelles, de placer les fenêtres de maintenance dans les heures nocturnes où la charge est réellement nulle et de formuler des critères de réception, par exemple : le nouveau système doit traiter sans erreur le double du pic mesuré. L’article [Test de charge SMTP avec Apache JMeter en pratique](/blog/jmeter-smtp-lasttest-html-report) montre comment transformer un tel profil en test de charge reproductible.

## Sources

1.  [Microsoft Learn: Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): référence de la requête de tracking, y compris tous les champs tels que EventId, RecipientCount et TotalBytes.

2.  [Microsoft Learn: Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): structure des journaux de tracking, types d’événements et configuration de la conservation et de la taille du répertoire.

3.  [Microsoft Learn: Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue): référence de la requête de file d’attente, y compris les champs IncomingRate, OutgoingRate et Velocity.

4.  [Microsoft Learn: Queues and messages in queues](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues): types de files d’attente, Shadow Redundancy et signification des valeurs d’état.
