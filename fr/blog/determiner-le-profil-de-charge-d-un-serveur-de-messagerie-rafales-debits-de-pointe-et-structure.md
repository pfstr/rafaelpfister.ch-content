---
title: "Déterminer le profil de charge d’un serveur de messagerie : rafales, débits de pointe et structure des destinataires à partir du suivi des messages"
navTitle: "Déterminer le profil de charge"
description: "Combien d’e-mails par minute votre serveur de messagerie traite-t-il réellement, et quels sont les pics ? Comment utiliser PowerShell pour déterminer le véritable profil de charge à partir du suivi des messages Exchange : débits par minute et par heure, durée des rafales, structure des destinataires, tailles des messages et erreurs d’analyse typiques."
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
translationSourceHash: b0fa7236ccc56203c5c0e7745b05de74b4b3890d470d3354a6299a295eb9b154
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:37:06.723Z
translationReview: automatic
url: https://rafaelpfister.ch/fr/blog/determiner-le-profil-de-charge-d-un-serveur-de-messagerie-rafales-debits-de-pointe-et-structure
---

# Déterminer le profil de charge d’un serveur de messagerie : rafales, débits de pointe et structure des destinataires à partir du suivi des messages

Qu’il s’agisse de remplacer une passerelle, de dimensionner un serveur ou de planifier une fenêtre de maintenance : tôt ou tard, tout administrateur de messagerie doit répondre à la question de savoir quel volume son système traite réellement. L’intuition est régulièrement trompeuse, car le trafic de messagerie est rarement régulier. Un système qui reçoit en moyenne 20 e-mails par minute sur une journée peut devoir en traiter 400 par minute pendant une heure lors d’un traitement de facturation. Ne connaître que la moyenne conduit à dimensionner à côté du véritable problème.

Un profil de charge exploitable se compose de quatre indicateurs : le débit moyen (par minute, heure, jour), les rafales (niveau du pic, durée, moment où elles surviennent), la structure des destinataires (nombre de destinataires distincts, domaines de destination) et la taille des messages. Les quatre figurent dans le suivi des messages et, sur Exchange, peuvent être calculés avec quelques lignes de PowerShell.

## La source de données : suivi des messages

Exchange consigne chaque message dans le journal de suivi des messages. Avant d’effectuer l’analyse, vérifiez jusqu’à quelle date les données remontent ; la valeur par défaut est de 30 jours, mais une limite de taille restreinte peut réduire sensiblement la durée de conservation réelle :

```powershell
Get-TransportService |
    Select-Object Name, MessageTrackingLogMaxAge,
        MessageTrackingLogMaxDirectorySize, MessageTrackingLogPath
```

Pour un profil de charge, la période doit couvrir au moins un cycle complet de traitement par lots de l’entreprise : cycles mensuels de facturation, décomptes de salaire, newsletters. Une semaine est le minimum, un mois est préférable.

## Collecter les données brutes : un événement par message

La décision préalable la plus importante est la suivante : quel événement compte comme « un e-mail » ? Le suivi des messages écrit plusieurs entrées par message (RECEIVE lors de l’acceptation, SEND lors de la transmission au saut suivant, DELIVER lors de la remise dans la boîte aux lettres, ainsi qu’AGENTINFO, HAREDIRECT et d’autres). Compter simplement toutes les lignes surestime le volume de plusieurs fois. Pour la charge d’entrée, comptez RECEIVE ; pour la charge de sortie vers le smarthost ou Internet, comptez SEND.

```powershell
$start = (Get-Date).AddDays(-7)
$events = Get-TransportService | ForEach-Object {
    Get-MessageTrackingLog -Server $_.Name -ResultSize Unlimited `
        -Start $start -EventId RECEIVE
}
"{0} Nachrichten seit {1:yyyy-MM-dd}" -f $events.Count, $start
```

La requête s’exécute volontairement sur tous les serveurs de transport, car chaque serveur ne consigne que sa propre part. Interroger un seul serveur ne montre, dans un cluster, qu’une fraction de la charge.

## Débits par minute et par heure : c’est ici que les rafales apparaissent

L’agrégation consiste en un Group-Object sur l’horodatage arrondi. Les minutes les plus chargées sont directement vos candidates aux rafales :

```powershell
$proMinute = $events |
    Group-Object { $_.Timestamp.ToString("yyyy-MM-dd HH:mm") } |
    Sort-Object Count -Descending

$proMinute | Select-Object -First 10 Name, Count
```

Il en va de même par heure et pour le profil journalier (à quelle heure la charge est-elle généralement la plus élevée) :

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

Une rafale n’est caractérisée que si vous connaissez sa durée en plus du pic. Un pic de 400/min qui dure deux minutes n’impose pas les mêmes exigences que le même pic pendant une heure. Comptez les minutes au-dessus d’un seuil :

```powershell
$schwelle = 100
$burstMinuten = $proMinute | Where-Object Count -ge $schwelle
"{0} Minuten mit >= {1}/min, Peak: {2}/min" -f $burstMinuten.Count,
    $schwelle, ($proMinute | Select-Object -First 1).Count
```

Si les minutes de rafale sont consécutives (directement visibles dans la sortie de `$burstMinuten | Sort-Object Name`), il s’agit d’un traitement par lots. Notez l’heure de début, la durée et le schéma de répétition, car c’est précisément cette fenêtre que l’infrastructure doit pouvoir supporter.

## Structure des destinataires : combien de destinations, quels domaines

Pour les passerelles, la diversité des destinataires est souvent plus importante que le seul débit, car chaque destinataire entraîne des recherches (routage, stratégies, règles de chiffrement). Un e-mail adressé à une liste de distribution de 5'000 membres sollicite le système différemment de 5'000 e-mails individuels. Le champ `RecipientCount` et la liste des destinataires fournissent les deux perspectives :

```powershell
"Nachrichten: {0}, Empfänger-Zustellungen: {1}" -f $events.Count,
    ($events | Measure-Object RecipientCount -Sum).Sum

$alleEmpfaenger = $events | ForEach-Object { $_.Recipients } |
    ForEach-Object { $_.ToLower() }
"Eindeutige Empfänger: {0}" -f ($alleEmpfaenger | Sort-Object -Unique).Count
```

La répartition par domaine montre vers où le trafic s’écoule. Si Gmail et Microsoft dominent, ce sont leurs limites de débit et la réputation de votre propre IP qui déterminent le débit atteignable, et non votre matériel :

```powershell
$alleEmpfaenger |
    ForEach-Object { ($_ -split "@")[1] } |
    Group-Object |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

Et dans l’autre sens : quels expéditeurs (applications, boîtes aux lettres fonctionnelles) génèrent réellement la charge ? Cela répond également à la question des systèmes à prendre en compte lors d’une migration :

```powershell
$events |
    Group-Object Sender |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

## Taille des messages : octets par seconde plutôt qu’e-mails par seconde

Les indications de débit des passerelles se rapportent souvent au volume de données, et non au nombre de messages. Deux systèmes ayant le même débit d’e-mails diffèrent d’un facteur 100 si l’un envoie des notifications de 50 KB et l’autre des PDF de factures de 5 MB. Le champ `TotalBytes` fournit la répartition :

```powershell
$events | Measure-Object TotalBytes -Average -Maximum -Sum |
    Select-Object @{n = "MittelKB"; e = { [math]::Round($_.Average / 1KB) } },
        @{n = "MaxMB"; e = { [math]::Round($_.Maximum / 1MB, 1) } },
        @{n = "TotalGB"; e = { [math]::Round($_.Sum / 1GB, 1) } }
```

Multipliez le débit de rafale par la taille moyenne durant la fenêtre de rafale, et vous obtenez l’exigence de bande passante que doit supporter une nouvelle passerelle ou une liaison WAN.

## Débits en direct sans suivi : jeter un œil aux files d’attente

Pour obtenir une vue instantanée (le serveur traite-t-il actuellement un volume important, quelque chose s’accumule-t-il ?), vous n’avez pas besoin du suivi : les files d’attente l’indiquent directement. `IncomingRate` et `OutgoingRate` correspondent à des e-mails par minute, lissés sur les dernières minutes :

```powershell
Get-TransportService |
    ForEach-Object { Get-Queue -Server $_.Name } |
    Sort-Object MessageCount -Descending |
    Select-Object Identity, Status, MessageCount, IncomingRate,
        OutgoingRate, NextHopDomain, LastError |
    Format-Table -AutoSize
```

Interprétation : une file d’attente `Submission` avec un débit élevé et une profondeur de 0 signifie que le serveur traite la charge sans accumulation. `MessageCount` élevé avec `OutgoingRate` proche de zéro signifie un engorgement. `Status Retry` avec un message 4xx dans `LastError` signifie que le système distant limite le débit. En revanche, les files d’attente `Shadow` contenant des messages sont normales : il s’agit de copies redondantes destinées au serveur partenaire, non d’un engorgement.

Pour une courbe continue pendant une fenêtre de charge, le compteur de performances des files d’attente de transport convient, ici toutes les cinq secondes pendant une minute :

```powershell
Get-Counter "\MSExchangeTransport Queues(_total)\Messages Completed Delivery Per Second" `
    -SampleInterval 5 -MaxSamples 12
```

## Autres systèmes : le même principe avec CSV

Au lieu d’objets PowerShell, les passerelles et appliances fournissent généralement une exportation CSV du suivi. La méthode reste identique (choisir un événement par e-mail, regrouper par fenêtres temporelles), seul l’outil change, par exemple Python :

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

## Les cinq erreurs d’analyse typiques

**Événements multiples par e-mail.** La source d’erreur la plus fréquente : compter les lignes au lieu des messages. Vérifiez avec `$events | Group-Object EventId` ce que contient réellement votre jeu de données et filtrez exactement un événement par message.

**Exportations tronquées.** De nombreuses fonctions d’exportation fournissent au maximum 10'000 ou 50'000 lignes, puis tronquent silencieusement les données, volontiers au milieu de la plus grande rafale. Un nombre de lignes suspectement rond est un signal d’alarme. Vérifiez toujours que la période des données correspond à la période demandée.

**Boucles de passerelle.** Si le flux de messagerie passe par une station intermédiaire (passerelle de chiffrement, appliance d’hygiène) puis revient, le même e-mail apparaît plusieurs fois dans le suivi. Dédupliquez à l’aide de l’ID de message ou filtrez sur un point univoque de la chaîne.

**Fuseaux horaires.** `Get-MessageTrackingLog` fournit des horodatages en heure locale du serveur, tandis que les exportations CSV d’appliances sont souvent en UTC. Une rafale qui semble se produire à 13 heures peut en réalité être le traitement par lots de 15 heures. Clarifiez la base temporelle avant toute interprétation.

**Fenêtres trop courtes.** Un profil de charge établi à partir de deux jours calmes ne vaut rien si le cycle mensuel de facturation manque. La fenêtre d’analyse doit contenir les cycles de traitement par lots connus ; demandez aux responsables des applications leurs calendriers d’envoi avant de définir la fenêtre.

## Ce que vous faites du profil

À la fin, vous disposez de quatre chiffres sur une page : débit moyen, rafale (pic, durée, moment, schéma de répétition), structure des destinataires (destinataires uniques par traitement, principaux domaines) et répartition des tailles. Ils permettent de dimensionner les passerelles, de placer les fenêtres de maintenance pendant les heures nocturnes de charge réellement nulle et de formuler des critères de réception, par exemple : le nouveau système doit pouvoir traiter sans erreur le double du pic mesuré. L’article [Test de charge SMTP avec Apache JMeter en pratique](/blog/jmeter-smtp-lasttest-html-report) montre comment transformer un tel profil en un test de charge reproductible.

## Sources

1.  [Microsoft Learn: Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): Référence de la requête de suivi, y compris tous les champs tels que EventId, RecipientCount et TotalBytes.

2.  [Microsoft Learn: Suivi des messages](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): Structure des journaux de suivi, types d’événements et configuration de la conservation et de la taille du répertoire.

3.  [Microsoft Learn: Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue): Référence de la requête sur les files d’attente, y compris les champs IncomingRate, OutgoingRate et Velocity.

4.  [Microsoft Learn: Files d’attente et messages dans les files d’attente](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues): Types de files d’attente, Shadow Redundancy et signification des valeurs d’état.
