---
title: "Déterminer le profil de charge d’un serveur de messagerie : rafales, débits de pointe et structure des destinataires à partir du suivi des messages"
navTitle: "Déterminer le profil de charge"
description: "Combien d’e-mails par minute votre serveur de messagerie traite-t-il réellement, et quels sont les pics ? Comment déterminer le véritable profil de charge à partir du suivi des messages Exchange avec PowerShell : débits par minute et par heure, durée des rafales, structure des destinataires, tailles des messages et erreurs d’analyse courantes."
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
translationSourceHash: 298fabdf5f8f248539ea8a119681be130cd76f5c8ebc35db5d0c61e1126251b0
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:28:15.490Z
translationReview: automatic
url: https://rafaelpfister.ch/fr/blog/determiner-le-profil-de-charge-d-un-serveur-de-messagerie-rafales-debits-de-pointe-et-structure
---

# Déterminer le profil de charge d’un serveur de messagerie : rafales, débits de pointe et structure des destinataires à partir du suivi des messages

Qu’il s’agisse de remplacer une passerelle, de dimensionner un serveur ou de planifier une fenêtre de maintenance : tôt ou tard, tout administrateur de messagerie doit répondre à la question de savoir ce que son système traite réellement. L’intuition est régulièrement trompeuse, car le trafic de messagerie est rarement uniforme. Un système qui traite en moyenne 20 e-mails par minute sur la journée peut devoir en traiter 400 par minute pendant une heure lors d’un cycle de facturation. Se fier uniquement à la moyenne revient à dimensionner à côté du véritable problème.

Un profil de charge exploitable comprend quatre indicateurs : le débit moyen (par minute, heure, jour), les rafales (niveau du pic, durée et moment où elles se produisent), la structure des destinataires (nombre de destinataires distincts, domaines de destination) et la taille des messages. Ces quatre informations figurent dans le suivi des messages et, avec Exchange, quelques lignes de PowerShell suffisent à les calculer.

## La source de données : le suivi des messages

Exchange consigne chaque message dans le journal de suivi des messages. Avant de procéder à l’analyse, vérifiez jusqu’où remontent les données ; la valeur par défaut est de 30 jours, mais une limite de taille restreinte peut considérablement réduire la rétention réelle :

```powershell
Get-TransportService |
    Select-Object Name, MessageTrackingLogMaxAge,
        MessageTrackingLogMaxDirectorySize, MessageTrackingLogPath
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Get-TransportService` | Répertorie tous les serveurs de transport de l’organisation ; sans paramètre, tous les serveurs |
| `Select-Object Name, MessageTrackingLog…` | Limite la sortie aux propriétés indiquées : durée de rétention, limite de taille du répertoire des journaux et chemin des journaux |

</details>

Pour un profil de charge, la période doit couvrir au minimum un cycle complet de traitement par lots de l’entreprise : cycles de facturation mensuels, décomptes de salaire, newsletters. Une semaine est le minimum, un mois est préférable.

## Collecter les données brutes : un événement par message

La décision préalable la plus importante est la suivante : quel événement compte comme « un e-mail » ? Le suivi des messages écrit plusieurs entrées par message (RECEIVE lors de l’acceptation, SEND lors de la transmission au saut suivant, DELIVER lors de la livraison dans la boîte aux lettres, ainsi que AGENTINFO, HAREDIRECT et d’autres). Compter simplement toutes les lignes surestime le volume d’un facteur multiple. Pour la charge d’entrée, comptez RECEIVE ; pour la charge sortante vers un smarthost ou Internet, comptez SEND.

```powershell
$start = (Get-Date).AddDays(-7)
$events = Get-TransportService | ForEach-Object {
    Get-MessageTrackingLog -Server $_.Name -ResultSize Unlimited `
        -Start $start -EventId RECEIVE
}
"{0} Nachrichten seit {1:yyyy-MM-dd}" -f $events.Count, $start
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Server $_.Name` | Interroge, via le pipeline, le journal de suivi du serveur de transport concerné |
| `-ResultSize Unlimited` | Supprime la limite par défaut de 1'000 entrées renvoyées |
| `-Start $start` | Limite temporelle inférieure de la requête ; ici, les sept derniers jours |
| `-EventId RECEIVE` | Filtre sur exactement un événement par message, ici l’acceptation par le service de transport |
| `-f` | Opérateur de formatage : insère les valeurs de droite dans les espaces réservés `{0}` et `{1}` de la chaîne |

</details>

La requête s’exécute délibérément sur tous les serveurs de transport, car chaque serveur ne consigne que sa propre part. Interroger un seul serveur dans un cluster ne montre qu’une fraction de la charge.

## Débits par minute et par heure : c’est ici que les rafales apparaissent

L’agrégation repose sur un Group-Object appliqué à l’horodatage arrondi. Les minutes les plus chargées sont directement vos candidates aux rafales :

```powershell
$proMinute = $events |
    Group-Object { $_.Timestamp.ToString("yyyy-MM-dd HH:mm") } |
    Sort-Object Count -Descending

$proMinute | Select-Object -First 10 Name, Count
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Group-Object { … }` | Regroupe selon la valeur renvoyée par le bloc de script, ici l’horodatage tronqué à la minute |
| `Sort-Object Count -Descending` | Trie les groupes par nombre décroissant ; les minutes les plus chargées apparaissent en tête |
| `Select-Object -First 10 Name, Count` | Affiche uniquement les dix groupes les plus importants, réduits à la minute et au nombre |

</details>

La même chose par heure et sous forme de profil journalier (quelle heure est généralement la plus chargée) :

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

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Group-Object { … ToString("yyyy-MM-dd HH") }` | Regroupe par heures pleines d’un jour donné |
| `Group-Object { … ToString("HH") }` | Regroupe uniquement par heure et agrège ainsi tous les jours : le profil journalier |
| `Sort-Object Count -Descending` | Heures les plus chargées en tête |
| `Sort-Object Name` | Trie le profil journalier chronologiquement par heure plutôt que par nombre |
| `Format-Table Name, Count` | Affichage tabulaire des deux colonnes |

</details>

Une rafale n’est caractérisée que lorsque vous connaissez sa durée en plus de son pic. Un pic de 400/min qui dure deux minutes représente une exigence différente du même pic pendant une heure. Comptez les minutes dépassant un seuil :

```powershell
$schwelle = 100
$burstMinuten = $proMinute | Where-Object Count -ge $schwelle
"{0} Minuten mit >= {1}/min, Peak: {2}/min" -f $burstMinuten.Count,
    $schwelle, ($proMinute | Select-Object -First 1).Count
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Where-Object Count -ge $schwelle` | Filtre les minutes comptant au moins autant de messages que le seuil (syntaxe simplifiée sans bloc de script) |
| `Select-Object -First 1` | Premier groupe de la liste triée par ordre décroissant, donc la minute la plus chargée |
| `-f` | Opérateur de formatage : insère le nombre, le seuil et le pic dans les espaces réservés `{0}` à `{2}` |

</details>

Si les minutes de rafale sont contiguës (directement visibles dans la sortie de `$burstMinuten | Sort-Object Name`), il s’agit d’un traitement par lots. Notez l’heure de début, la durée et le schéma de répétition, car c’est précisément cette fenêtre que l’infrastructure doit supporter.

## Structure des destinataires : combien de cibles, quels domaines

Pour les passerelles, la diversité des destinataires est souvent plus importante que le débit brut, car chaque destinataire entraîne des recherches (routage, politiques, règles de chiffrement). Un e-mail envoyé à une liste de distribution de 5'000 membres sollicite le système différemment de 5'000 e-mails individuels. Le champ `RecipientCount` et la liste des destinataires fournissent les deux perspectives :

```powershell
"Nachrichten: {0}, Empfänger-Zustellungen: {1}" -f $events.Count,
    ($events | Measure-Object RecipientCount -Sum).Sum

$alleEmpfaenger = $events | ForEach-Object { $_.Recipients } |
    ForEach-Object { $_.ToLower() }
"Eindeutige Empfänger: {0}" -f ($alleEmpfaenger | Sort-Object -Unique).Count
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Measure-Object RecipientCount -Sum` | Additionne le champ `RecipientCount` sur tous les événements : le nombre de livraisons aux destinataires |
| `ForEach-Object { $_.Recipients }` | Déplie la liste des destinataires de chaque événement en adresses individuelles |
| `ForEach-Object { $_.ToLower() }` | Normalise les adresses en minuscules afin que les doublons soient reconnus comme tels |
| `Sort-Object -Unique` | Trie et supprime les doublons ; `Count` fournit ensuite le nombre d’adresses uniques |

</details>

La répartition par domaine montre où circule le trafic. Si Gmail et Microsoft dominent, ce sont leurs limites de débit et la réputation de votre propre adresse IP qui déterminent le débit atteignable, et non votre matériel :

```powershell
$alleEmpfaenger |
    ForEach-Object { ($_ -split "@")[1] } |
    Group-Object |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `($_ -split "@")[1]` | Scinde l’adresse au niveau de `@` et conserve la partie domaine |
| `Group-Object` | Regroupe sans argument selon la valeur elle-même, ici le domaine |
| `Sort-Object Count -Descending` | Domaines les plus fréquents en tête |
| `Select-Object -First 10 Name, Count` | Limite la sortie aux 10 premiers |

</details>

Et dans l’autre sens : quels expéditeurs (applications, boîtes aux lettres fonctionnelles) génèrent réellement la charge ? Cela répond également à la question de savoir quels systèmes doivent être pris en compte lors d’une migration :

```powershell
$events |
    Group-Object Sender |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Group-Object Sender` | Regroupe selon le champ `Sender` (paramètre positionnel `-Property`) |
| `Sort-Object Count -Descending` | Expéditeurs ayant le plus de messages en tête |
| `Select-Object -First 10 Name, Count` | Limite la sortie aux 10 premiers |

</details>

## Taille des messages : octets par seconde plutôt qu’e-mails par seconde

Les indications de débit des passerelles se rapportent souvent au volume de données, et non au nombre de messages. Deux systèmes présentant le même débit d’e-mails diffèrent d’un facteur 100 si l’un envoie des notifications de 50 Ko et l’autre des PDF de factures de 5 Mo. Le champ `TotalBytes` fournit la répartition :

```powershell
$events | Measure-Object TotalBytes -Average -Maximum -Sum |
    Select-Object @{n = "MittelKB"; e = { [math]::Round($_.Average / 1KB) } },
        @{n = "MaxMB"; e = { [math]::Round($_.Maximum / 1MB, 1) } },
        @{n = "TotalGB"; e = { [math]::Round($_.Sum / 1GB, 1) } }
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Measure-Object TotalBytes -Average -Maximum -Sum` | Calcule en une seule opération la moyenne, le maximum et la somme du champ `TotalBytes` |
| `@{n = "…"; e = { … }}` | Propriété calculée : `n` nomme la colonne, `e` fournit la valeur via un bloc de script, ici la conversion en Ko, Mo et Go |

</details>

Multipliez le débit de rafale par la taille moyenne dans la fenêtre de rafale, et vous obtenez l’exigence de bande passante que doit supporter une nouvelle passerelle ou une liaison WAN.

## Débits en direct sans suivi : un regard sur les files d’attente

Pour une vue instantanée (le serveur traite-t-il beaucoup en ce moment, quelque chose s’accumule-t-il ?), vous n’avez pas besoin de suivi : les files d’attente l’indiquent directement. `IncomingRate` et `OutgoingRate` correspondent à des e-mails par minute, lissés sur les dernières minutes :

```powershell
Get-TransportService |
    ForEach-Object { Get-Queue -Server $_.Name } |
    Sort-Object MessageCount -Descending |
    Select-Object Identity, Status, MessageCount, IncomingRate,
        OutgoingRate, NextHopDomain, LastError |
    Format-Table -AutoSize
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Get-Queue -Server $_.Name` | Répertorie, via le pipeline, les files d’attente de transport du serveur concerné |
| `Sort-Object MessageCount -Descending` | Files d’attente les plus remplies en tête |
| `Select-Object Identity, Status, …` | Limite la sortie aux champs pertinents pour l’évaluation de la charge |
| `Format-Table -AutoSize` | Adapte la largeur des colonnes au contenu au lieu de les tronquer |

</details>

Interprétation : une file d’attente `Submission` avec un débit élevé et une profondeur de 0 signifie que le serveur traite la charge sans accumulation. Une valeur `MessageCount` élevée alors que `OutgoingRate` est proche de zéro indique un engorgement. `Status Retry` avec un message 4xx dans `LastError` signifie que le destinataire distant limite le débit. En revanche, des files d’attente `Shadow` contenant des messages sont normales : il s’agit de copies de redondance pour le serveur partenaire, pas d’un engorgement.

Pour une courbe continue pendant une fenêtre de charge, le compteur de performances des files d’attente de transport convient bien ; ici, toutes les cinq secondes pendant une minute :

```powershell
Get-Counter "\MSExchangeTransport Queues(_total)\Messages Completed Delivery Per Second" `
    -SampleInterval 5 -MaxSamples 12
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `"\MSExchangeTransport Queues(_total)\…"` | Chemin du compteur de performances (paramètre positionnel `-Counter`) ; l’instance `_total` effectue la somme sur toutes les files d’attente |
| `-SampleInterval 5` | Intervalle entre deux mesures, en secondes |
| `-MaxSamples 12` | Nombre de mesures ; 12 mesures toutes les 5 secondes correspondent à une minute |

</details>

## Autres systèmes : le même principe avec CSV

Les passerelles et appliances fournissent généralement un export CSV du suivi plutôt que des objets PowerShell. La procédure reste identique (choisir un événement par e-mail, regrouper par fenêtres temporelles), seul l’outil change, par exemple pour Python :

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

## Les cinq erreurs d’analyse les plus courantes

**Événements multiples par e-mail.** C’est la source d’erreur la plus fréquente : compter les lignes plutôt que les messages. Vérifiez avec `$events | Group-Object EventId` ce que contient réellement votre jeu de données, puis filtrez sur exactement un événement par message.

**Exports tronqués.** De nombreuses fonctions d’export renvoient au maximum 10'000 ou 50'000 lignes, puis tronquent silencieusement les données, souvent en plein milieu de la plus grande rafale. Un nombre de lignes étrangement rond est un signal d’alarme. Vérifiez toujours que la période des données correspond à la période demandée.

**Boucles de passerelle.** Si le flux de messagerie passe par une station intermédiaire (passerelle de chiffrement, appliance d’hygiène) puis revient, le même e-mail apparaît plusieurs fois dans le suivi. Dédupliquez à l’aide de l’ID de message ou filtrez sur un point unique de la chaîne.

**Fuseaux horaires.** `Get-MessageTrackingLog` fournit des horodatages en heure locale du serveur, tandis que les exports CSV d’appliances sont souvent en UTC. Une rafale qui semble se produire à 13 heures peut en réalité être le traitement par lots de 15 heures. Clarifiez la référence temporelle avant d’interpréter les données.

**Fenêtres trop courtes.** Un profil de charge établi à partir de deux jours calmes ne vaut rien si le cycle mensuel de facturation en est absent. La fenêtre d’analyse doit contenir les cycles de traitement par lots connus ; demandez aux responsables des applications leurs calendriers d’envoi avant de définir la fenêtre.

## Ce que vous faites du profil

Au final, quatre chiffres tiennent sur une page : débit moyen, rafale (pic, durée, moment, schéma de répétition), structure des destinataires (destinataires uniques par traitement, principaux domaines) et répartition des tailles. Ils permettent de dimensionner les passerelles, de placer les fenêtres de maintenance dans les heures nocturnes de charge réellement nulle et de formuler des critères de réception, par exemple : le nouveau système doit traiter sans erreur le double du pic mesuré. L’article [Test de charge SMTP avec Apache JMeter en pratique](/blog/jmeter-smtp-lasttest-html-report) montre comment transformer un tel profil en un test de charge reproductible.

## Sources

1.  [Microsoft Learn: Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): Référence de la requête de suivi, y compris tous les champs tels que EventId, RecipientCount et TotalBytes.

2.  [Microsoft Learn: Suivi des messages](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): Structure des journaux de suivi, types d’événements et configuration de la rétention et de la taille du répertoire.

3.  [Microsoft Learn: Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue): Référence de la requête de file d’attente, y compris les champs IncomingRate, OutgoingRate et Velocity.

4.  [Microsoft Learn: Files d’attente et messages dans les files d’attente](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues): Types de files d’attente, Shadow Redundancy et signification des valeurs d’état.
