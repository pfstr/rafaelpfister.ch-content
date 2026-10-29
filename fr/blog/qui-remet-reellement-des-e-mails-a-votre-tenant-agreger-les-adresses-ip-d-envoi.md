---
title: "Qui remet réellement des e-mails à votre tenant ? Agréger les adresses IP d’envoi"
navTitle: "IP d’envoi"
description: "Une seule analyse montre quels systèmes remettent réellement des e-mails à votre tenant : connecteurs oubliés, applications qui envoient directement et prestataires que personne n’a documentés, y compris les erreurs d’analyse typiques liées à la logique de pagination et à l’interprétation."
date: "2026-08-11"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "12 min de lecture"
themen:
  - microsoft-365-exchange
  - smtp-mailflow
  - exchange-onprem-hybrid
hauptthema: "microsoft-365-exchange"
produkte:
  - "exchange-online"
  - "exchange-hybrid"
protokolle:
  - "smtp"
  - "powershell"
  - "haertung"
related:
  - exchange-message-tracking-und-receive-connectoren-analysieren
  - ghost-sender-exchange-online-nebeneingang
  - mailflow-fehlersuche-kontrollierte-experimente
slug: "qui-remet-reellement-des-e-mails-a-votre-tenant-agreger-les-adresses-ip-d-envoi"
translationId: "article-5879cc0eb17ed951"
draft: false
translationOf: einliefernde-ip-adressen-aggregieren
translationSourceHash: 291ed92e756d5fa011212a5cf9d3ce9061df96d73857f508c838183965ba25b4
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:38:15.659Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/qui-remet-reellement-des-e-mails-a-votre-tenant-agreger-les-adresses-ip-d-envoi
---

# Qui remet réellement des e-mails à votre tenant ? Agréger les adresses IP d’envoi

Presque aucun environnement de messagerie ne sait encore complètement qui lui remet des e-mails. Au fil des années s’accumulent des connecteurs issus de migrations, des applications qui envoient directement, des prestataires dont le contrat a expiré depuis longtemps et des configurations de test qui n’ont jamais été démontées. Tant que le courrier circule, personne ne s’en aperçoit.

Une seule analyse apporte de la clarté : le regroupement de tous les messages entrants selon leur adresse IP source. Elle se réalise en deux minutes et la liste des résultats est régulièrement surprenante. Cet article présente la requête, explique comment l’obtenir **dans son intégralité** et, surtout, comment interpréter correctement les chiffres. Car l’interprétation est la partie la plus difficile.

## Pourquoi cela en vaut la peine

La liste répond à quatre questions autrement fastidieuses à clarifier séparément. Quels systèmes envoient réellement vers votre tenant ? Tout passe-t-il par les voies que vous avez documentées ou existe-t-il une seconde entrée ? Un connecteur que vous souhaitez supprimer est-il encore utilisé ? Et : une application envoie-t-elle directement au service en contournant votre passerelle, donc en contournant votre filtrage ?

La dernière question, en particulier, est importante pour la sécurité. Quiconque remet directement des e-mails contourne non seulement le filtrage, mais souvent aussi la journalisation sur laquelle vous souhaitez pouvoir compter en cas d’incident.

## La requête

Dans le tenant, regroupez le suivi des messages selon `FromIP` :

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-2) `
    -EndDate (Get-Date) `
    -ResultSize 5000 |
  Group-Object FromIP |
  Sort-Object Count -Descending |
  Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-StartDate (Get-Date).AddHours(-2)` | début de la fenêtre de requête, ici il y a deux heures |
| `-EndDate (Get-Date)` | fin de la fenêtre de requête, le moment actuel |
| `-ResultSize 5000` | nombre maximal de lignes par appel ; 5 000 est également la valeur maximale |
| `Group-Object FromIP` | regroupe les messages selon l’adresse IP de remise |
| `Sort-Object Count -Descending` | trie les groupes par ordre décroissant du nombre de messages |
| `Format-Table Count, Name -AutoSize` | sortie sur deux colonnes (nombre, adresse IP) avec largeur de colonne automatique |

</details>

Une sortie typique :

```text
Count Name
----- ----
 1771 255.255.255.255
 1649 10.0.20.23
  260 10.0.20.21
   49 2603:10a6:150:1f3::17
   46 165.225.94.87
   36 136.226.192.164
   35 147.161.246.105
   12 198.51.100.77
    3 203.0.113.9
```

Avant d’en tirer des conclusions, deux conditions doivent être réunies : la liste doit être complète et vous devez savoir ce que signifient les entrées.

## Source d’erreur 1 : la liste est presque toujours incomplète

`Get-MessageTraceV2` retourne les résultats par pages, avec un maximum de 5 000 lignes par appel. En cas de volume élevé, une page ne couvre qu’une fraction de votre fenêtre temporelle. Vous regroupez alors un extrait et prenez le résultat pour l’ensemble.

Cela se reconnaît à cet avertissement :

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

**Si cet avertissement apparaît, votre analyse ne vaut rien.** En particulier, une entrée absente ne doit alors pas être interprétée comme une absence. Une adresse avec trois messages par jour n’apparaîtra de toute façon pas dans un extrait.

Il existe deux solutions. La plus simple : réduisez la fenêtre temporelle jusqu’à ce que l’avertissement disparaisse. Avec 5 000 messages par heure, cela représente 55 minutes et non sept jours. Pour répondre à la question « quels systèmes envoient réellement », une courte fenêtre complète suffit généralement amplement.

La méthode approfondie parcourt toutes les pages et collecte les résultats :

```powershell
$start = (Get-Date).AddHours(-24)
$ende  = Get-Date
$alle  = @()
$naechster = $null

do {
    $seite = if ($naechster) {
        Get-MessageTraceV2 -StartDate $start -EndDate $ende `
            -StartingRecipientAddress $naechster -ResultSize 5000
    } else {
        Get-MessageTraceV2 -StartDate $start -EndDate $ende -ResultSize 5000
    }

    $alle += $seite
    if ($seite.Count -eq 5000) {
        $naechster = $seite[-1].RecipientAddress
        $ende      = $seite[-1].Received
    } else {
        $naechster = $null
    }
    Write-Host "Gesammelt: $($alle.Count)"
} while ($naechster)

$alle | Group-Object FromIP | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-StartDate` / `-EndDate` | fenêtre de requête, ici les dernières 24 heures |
| `-StartingRecipientAddress` | point de reprise de la pagination : l’adresse du destinataire de la dernière entrée de la page précédente ; avec `-EndDate` = son heure de réception (`Received`), la page suivante commence à cet endroit |
| `-ResultSize 5000` | taille de page ; une page complète indique que d’autres résultats suivent |
| `Group-Object FromIP` | regroupe l’ensemble des données selon l’adresse IP de remise |
| `Sort-Object Count -Descending` | trie les groupes par ordre décroissant du nombre de messages |
| `Format-Table Count, Name -AutoSize` | affiche le nombre par adresse avec largeur de colonne automatique |

</details>

La boucle récupère des pages supplémentaires tant qu’une page retourne exactement 5 000 lignes, puis reprend à chaque fois avec l’adresse du destinataire et l’heure de réception de la dernière entrée de la page précédente, comme le prévoit la documentation de `Get-MessageTraceV2` ; l’ensemble complet n’est regroupé qu’ensuite.

Pour 24 heures dans un environnement de taille moyenne, comptez quelques minutes d’exécution. Pour un inventaire ponctuel, c’est un investissement judicieux.

## Source d’erreur 2 : les chiffres ne signifient pas ce qu’ils semblent indiquer

La liste des résultats contient quatre types d’entrées fondamentalement différents, et les mettre dans le même sac mène à de fausses conclusions.

**`255.255.255.255` ne représente pas un système.** Cette valeur apparaît lorsqu’il n’y a pas eu de connexion SMTP entrante depuis l’extérieur pour le message. Cela concerne les messages générés dans le service lui-même : rapports de journalisation, notifications de non-remise, réponses automatiques d’absence, messages entre boîtes aux lettres du même tenant. Dans presque tous les environnements, c’est le poste le plus important, et il est tout à fait normal.

**Les adresses privées définies par la RFC 1918** proviennent de votre propre réseau. Dans les environnements hybrides, vous voyez ici les serveurs de transport locaux, car leur adresse interne est conservée lors de la remise au service. Ce sont les grands chiffres de la liste et, dans la grande majorité des cas, le chemin principal attendu.

**Les adresses des services de sécurité et de filtrage** se reconnaissent à leur opérateur, non à leur valeur numérique. Les proxys cloud, les passerelles de messagerie en amont et les services de sécurité web apparaissent avec de nombreuses adresses voisines et des chiffres moyens. Ils en font généralement partie, mais devraient figurer dans le manuel d’exploitation.

**Les adresses publiques isolées avec de petits chiffres** sont les plus intéressantes. C’est précisément là que se cachent les applications oubliées, les anciens prestataires et les systèmes dont plus personne ne se souvient.

## La résolution : des adresses aux noms

Pour tout ce que vous ne pouvez pas attribuer immédiatement, la résolution inverse est utile. Elle n’est pas toujours configurée ni toujours fiable, mais elle fournit dans la majorité des cas l’indice décisif :

```powershell
$unbekannt = '198.51.100.77','203.0.113.9'

$unbekannt | ForEach-Object {
    $name = try { (Resolve-DnsName $_ -Type PTR -ErrorAction Stop).NameHost } catch { '(kein PTR)' }
    [pscustomobject]@{ IP = $_; Name = $name }
} | Format-Table -AutoSize
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `Resolve-DnsName $_ -Type PTR` | interroge l’enregistrement inverse (PTR) de chaque adresse IP |
| `-ErrorAction Stop` | transforme une entrée absente en erreur interceptable pour le bloc `try`/`catch` |
| `[pscustomobject]@{ … }` | crée pour chaque adresse un objet avec l’IP et le nom résolu pour l’affichage sous forme de tableau |
| `Format-Table -AutoSize` | sortie avec largeur de colonne automatique |

</details>

```text
IP            Name
--            ----
198.51.100.77 mail-out-03.newsletter-provider.example
203.0.113.9   (kein PTR)
```

L’absence d’un PTR n’est pas en soi le signe d’un problème, mais c’est une bonne raison de regarder de plus près. Pour ces adresses, examinez les messages correspondants :

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date) -ResultSize 5000 |
  Where-Object { $_.FromIP -eq '203.0.113.9' } |
  Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-StartDate` / `-EndDate` / `-ResultSize` | fenêtre de requête et taille de page comme dans la requête principale |
| `Where-Object { $_.FromIP -eq '203.0.113.9' }` | filtre côté client sur l’adresse source concernée |
| `Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize` | affiche, pour chaque message, l’heure de réception, l’expéditeur, le destinataire, l’objet et le statut de remise |

</details>

L’expéditeur et l’objet vous indiqueront généralement immédiatement quelle application se trouve derrière.

## Le rapprochement : quelle adresse appartient à quel connecteur ?

Comparez votre liste de résultats aux connecteurs configurés :

```powershell
Get-InboundConnector |
    Format-List Name, Enabled, ConnectorType, ConnectorSource,
        @{n='SenderIPAddresses'; e={$_.SenderIPAddresses -join ', '}},
        @{n='SenderDomains';     e={$_.SenderDomains -join ', '}},
        RequireTls, TlsSenderCertificateName,
        RestrictDomainsToCertificate, CloudServicesMailEnabled, TreatMessagesAsInternal
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `Get-InboundConnector` | répertorie tous les connecteurs entrants du tenant ; ici délibérément sans paramètres restrictifs |
| `Format-List <Eigenschaften>` | sortie sous forme de liste des propriétés indiquées, une par ligne |
| `@{n='…'; e={…}}` | propriété calculée avec nom (`n`) et expression (`e`) |
| `-join ', '` | transforme le tableau d’adresses ou de domaines en une ligne lisible séparée par des virgules |

</details>

Trois configurations sont révélatrices.

**Une adresse remet des e-mails, mais n’est mentionnée dans aucun connecteur.** Le courrier arrive alors comme un e-mail Internet ordinaire. C’est autorisé, mais cela signifie que cette application ne bénéficie d’aucun traitement particulier et que ses messages sont soumis à l’intégralité du filtrage. Si quelqu’un affirme qu’un connecteur est configuré pour ce système, ce n’est manifestement plus le cas.

**Un connecteur mentionne des adresses dont aucun message n’arrive.** C’est le candidat à la suppression. Avant de supprimer, vérifiez s’il s’agit de systèmes saisonniers ou rares, et désactivez d’abord plutôt que de supprimer immédiatement.

**Un connecteur définit `TreatMessagesAsInternal` ou `CloudServicesMailEnabled` sur vrai.** Cela mérite un examen attentif. Ces deux paramètres font que les messages arrivant par cette voie sont traités comme internes à l’organisation. Si des e-mails provenant d’Internet arrivent ainsi, ils contournent les vérifications prévues pour les messages externes, notamment la protection contre les expéditeurs usurpés de votre propre domaine. C’est correct pour un connecteur hybride pur ; pour un connecteur via lequel des systèmes quelconques remettent des e-mails, c’est un constat.

## Ce que vous trouvez généralement

Dans la pratique, sans prétendre à l’exhaustivité : un connecteur de test issu d’une migration, actif depuis des années. Une application métier qui envoie directement au service, bien que tout le monde pense qu’elle passe par la passerelle. Un prestataire de newsletter dont le contrat a expiré, mais qui peut toujours effectuer des remises. Et régulièrement un connecteur aux conditions très ouvertes, créé un jour pour résoudre un problème urgent.

Aucune de ces découvertes n’est dramatique en soi. Ensemble, elles dressent le tableau d’un environnement dont plus personne n’a une vue complète, et c’est précisément là le véritable risque.

## Limites de la méthode

Vous devez connaître trois limitations.

`Get-MessageTraceV2` couvre les 90 derniers jours, mais retourne au maximum dix jours par requête. Pour des périodes plus longues, interrogez donc plusieurs fenêtres temporelles ou utilisez la recherche historique, qui s’exécute de manière asynchrone et couvre également jusqu’à 90 jours. Sinon, les systèmes qui envoient mensuellement vous échapperont.

`FromIP` ne signifie pas la même chose partout. Pour les e-mails provenant d’Internet, c’est l’adresse du serveur de remise. Pour les e-mails hybrides, c’est l’adresse de votre serveur de transport local, et non celle de l’expéditeur d’origine. L’analyse vous montre donc la **dernière étape avant le service**, pas l’origine.

Et l’attribution à un connecteur n’est pas directement visible dans le tenant. Vous la déduisez de l’adresse, du certificat et du domaine expéditeur. Pour une affirmation fiable sur l’utilisation d’un connecteur individuel, le rapport sur les connecteurs dans le Centre d’administration Exchange, sous Rapports et Flux de messagerie, est une meilleure source, car il agrège côté serveur sur des périodes plus longues.

## En tant que contrôle récurrent

Cette analyse convient bien à une routine trimestrielle. Archivez le résultat et comparez-le lors du passage suivant. Les nouvelles adresses de la liste correspondent soit à des changements documentés, soit à quelque chose que vous voudrez connaître.

Si vous vérifiez de toute façon la configuration de messagerie de vos domaines : le [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check) affiche SPF, DKIM, DMARC et les autres standards de messagerie pour n’importe quel domaine directement dans le navigateur, y compris les domaines secondaires et marketing qui, lors de tels inventaires, sont notoirement oubliés. Et pour les requêtes elles-mêmes, le [générateur de commandes](https://rafaelpfister.ch/tools/command-builder) fournit des éléments prêts à l’emploi pour PowerShell et le shell Unix.

La procédure pour suivre des messages individuels suspects est décrite dans [Analyser le flux de messagerie Exchange : suivi des messages, journaux SMTP et connecteurs de réception](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren).

## Historique des modifications

- **25 octobre 2026 :** Correction : la logique de pagination de Get-MessageTraceV2 reprend avec l’adresse du destinataire et l’heure de réception (EndDate) de la dernière entrée. Correction : la cmdlet couvre les 90 derniers jours, mais retourne au maximum dix jours par requête.

## Sources

1.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2) : liste des champs, y compris FromIP et ToIP, période de requête (90 jours, au maximum dix jours par requête) ainsi que la reprise avec StartingRecipientAddress et EndDate.

2.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch) : suivi asynchrone des messages sur une période allant jusqu’à 90 jours pour les périodes plus anciennes.

3.  [Get-InboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchange/get-inboundconnector) : paramètres des connecteurs entrants, notamment SenderIPAddresses et TreatMessagesAsInternal.

4.  [Configure mail flow using connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow) : interaction entre les types de connecteurs et conditions d’application de chacun.

5.  [RFC 1918: Address Allocation for Private Internets](https://www.rfc-editor.org/rfc/rfc1918) : définit les plages d’adresses privées que vous devez distinguer des adresses publiques dans l’analyse.
