---
title: "Qui remet réellement des e-mails à votre tenant ? Agréger les adresses IP d’envoi"
navTitle: "IP d’envoi"
description: "Une seule analyse montre quels systèmes remettent réellement des e-mails à votre tenant : connecteurs oubliés, applications envoyant directement et prestataires que personne n’a documentés, y compris les erreurs d’analyse typiques liées à la pagination et à l’interprétation."
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
translationSourceHash: 9209720819061360cb72bfa01ab6261e6af80e547a398c25f6802edfbe49bb6c
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T10:05:09.899Z
translationReview: automatic
url: https://rafaelpfister.ch/fr/blog/qui-remet-reellement-des-e-mails-a-votre-tenant-agreger-les-adresses-ip-d-envoi
---

# Qui remet réellement des e-mails à votre tenant ? Agréger les adresses IP d’envoi

Pratiquement aucun environnement de messagerie ne sait encore précisément qui lui remet des e-mails. Au fil des années s’accumulent des connecteurs issus de migrations, des applications qui envoient directement, des prestataires dont le contrat a expiré depuis longtemps et des environnements de test qui n’ont jamais été démantelés. Tant que les e-mails circulent, personne ne le remarque.

Une seule analyse permet d’y voir clair : le regroupement de tous les messages entrants par leur adresse IP source. Elle se réalise en deux minutes et la liste des résultats est régulièrement surprenante. Cet article présente la requête, explique comment obtenir des résultats **exhaustifs** et, surtout, comment interpréter correctement les chiffres. Car l’interprétation est la partie la plus difficile.

## Pourquoi cela en vaut la peine

La liste répond à quatre questions qu’il est autrement laborieux de clarifier individuellement. Quels systèmes envoient réellement des messages à votre tenant ? Tout passe-t-il par les chemins que vous avez documentés, ou existe-t-il une seconde entrée ? Un connecteur que vous souhaitez supprimer est-il encore utilisé ? Et : une application envoie-t-elle directement au service en contournant votre passerelle, et donc votre filtrage ?

La dernière question en particulier est importante pour la sécurité. Quiconque remet directement des e-mails contourne non seulement le filtrage, mais souvent aussi la journalisation sur laquelle vous souhaitez pouvoir compter en cas d’incident.

## La requête

Dans le tenant, regroupez le suivi des messages par `FromIP` :

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
| `-StartDate (Get-Date).AddHours(-2)` | Début de la période de requête, ici il y a deux heures |
| `-EndDate (Get-Date)` | Fin de la période de requête, l’instant actuel |
| `-ResultSize 5000` | nombre maximal de lignes par appel ; 5000 est également la valeur maximale |
| `Group-Object FromIP` | regroupe les messages par adresse IP d’envoi |
| `Sort-Object Count -Descending` | trie les groupes par nombre de messages décroissant |
| `Format-Table Count, Name -AutoSize` | sortie à deux colonnes (nombre, adresse IP) avec largeur de colonne automatique |

</details>

Exemple de sortie :

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

Avant d’en tirer des conclusions, deux conditions doivent être remplies : la liste doit être exhaustive et vous devez savoir ce que signifient les entrées.

## Source d’erreur 1 : la liste est presque toujours incomplète

`Get-MessageTraceV2` renvoie les résultats par pages, avec un maximum de 5000 lignes par appel. En cas de volume élevé, une page ne couvre qu’une fraction de votre période. Vous regroupez alors un extrait et prenez le résultat pour l’ensemble.

Cela se reconnaît à cet avertissement :

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

**Si cet avertissement apparaît, votre analyse ne vaut rien.** En particulier, une entrée absente ne doit alors pas être interprétée comme une absence. Une adresse avec trois messages par jour n’apparaîtra de toute façon pas dans un extrait.

Il existe deux solutions. La plus simple : réduisez la période jusqu’à ce que l’avertissement disparaisse. Avec 5000 messages par heure, cela représente 55 minutes et non sept jours. Pour répondre à la question « quels systèmes envoient des messages ? », une courte période complète suffit généralement amplement.

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
    $naechster = if ($seite.Count -eq 5000) { $seite[-1].RecipientAddress } else { $null }
    Write-Host "Gesammelt: $($alle.Count)"
} while ($naechster)

$alle | Group-Object FromIP | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-StartDate` / `-EndDate` | Période de requête, ici les dernières 24 heures |
| `-StartingRecipientAddress` | Point de reprise de la pagination : l’adresse du destinataire à partir de laquelle commence la page suivante |
| `-ResultSize 5000` | Taille de page ; une page complète indique que d’autres résultats suivent |
| `Group-Object FromIP` | regroupe l’ensemble des résultats par adresse IP d’envoi |
| `Sort-Object Count -Descending` | trie les groupes par nombre de messages décroissant |
| `Format-Table Count, Name -AutoSize` | sortie du nombre par adresse avec largeur de colonne automatique |

</details>

La boucle récupère d’autres pages tant qu’une page contient exactement 5000 lignes, en reprenant à chaque fois à la dernière adresse de destinataire de la page précédente ; le regroupement n’intervient qu’une fois l’ensemble des résultats collecté.

Pour 24 heures dans un environnement de taille moyenne, comptez quelques minutes d’exécution. Pour un inventaire ponctuel, c’est du temps bien investi.

## Source d’erreur 2 : les chiffres ne signifient pas ce qu’ils semblent indiquer

La liste de résultats contient quatre types d’entrées fondamentalement différents ; les confondre conduit à de mauvaises conclusions.

**`255.255.255.255` ne représente pas un système.** Cette valeur apparaît lorsqu’il n’y a pas eu de connexion SMTP entrante depuis l’extérieur pour le message. Cela concerne les messages générés au sein même du service : rapports de journalisation, rapports de non-remise, réponses automatiques d’absence, messages entre des boîtes aux lettres du même tenant. Dans presque tous les environnements, c’est le poste le plus important, et il est parfaitement normal.

**Les adresses privées définies dans la RFC 1918** proviennent de votre propre réseau. Dans les environnements hybrides, vous verrez ici les serveurs de transport locaux, car leur adresse interne est conservée lors de la remise au service. Ce sont les grands chiffres de la liste et, en règle générale, le chemin principal attendu.

**Les adresses de services de sécurité et de filtrage** s’identifient par leur opérateur, non par leur valeur numérique. Les proxys cloud, les passerelles de messagerie en amont et les services de sécurité web apparaissent avec de nombreuses adresses voisines et des volumes moyens. Ils sont généralement légitimes, mais devraient figurer dans le manuel d’exploitation.

**Les adresses publiques isolées avec de faibles volumes** sont les plus intéressantes. C’est précisément là que se cachent les applications oubliées, les anciens prestataires et les systèmes dont plus personne ne se souvient.

## La résolution : des adresses aux noms

Pour tout ce que vous ne pouvez pas attribuer immédiatement, la résolution inverse est utile. Elle n’est pas toujours configurée ni toujours fiable, mais dans la majorité des cas, elle fournit l’indice décisif :

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
| `[pscustomobject]@{ … }` | crée pour chaque adresse un objet avec l’IP et le nom résolu pour l’affichage en tableau |
| `Format-Table -AutoSize` | sortie avec largeur de colonne automatique |

</details>

```text
IP            Name
--            ----
198.51.100.77 mail-out-03.newsletter-provider.example
203.0.113.9   (kein PTR)
```

L’absence d’un PTR n’est pas en soi le signe d’un problème, mais c’est une bonne raison d’examiner l’adresse de plus près. Pour ces adresses, examinez les messages correspondants :

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date) -ResultSize 5000 |
  Where-Object { $_.FromIP -eq '203.0.113.9' } |
  Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-StartDate` / `-EndDate` / `-ResultSize` | Période de requête et taille de page comme dans la requête principale |
| `Where-Object { $_.FromIP -eq '203.0.113.9' }` | filtre côté client sur l’adresse source concernée |
| `Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize` | affiche pour chaque message l’heure de réception, l’expéditeur, le destinataire, l’objet et l’état de remise |

</details>

L’expéditeur et l’objet vous indiquent généralement immédiatement quelle application se trouve derrière.

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
| `Get-InboundConnector` | répertorie tous les connecteurs entrants du tenant ; ici volontairement sans paramètres restrictifs |
| `Format-List <Eigenschaften>` | sortie sous forme de liste des propriétés indiquées, une par ligne |
| `@{n='…'; e={…}}` | propriété calculée avec un nom (`n`) et une expression (`e`) |
| `-join ', '` | transforme le tableau d’adresses ou de domaines en une ligne lisible séparée par des virgules |

</details>

Trois situations sont révélatrices.

**Une adresse remet des e-mails, mais n’est mentionnée dans aucun connecteur.** Le message arrive alors comme un e-mail Internet ordinaire. C’est autorisé, mais cela signifie que cette application ne bénéficie d’aucun traitement particulier et que ses messages sont soumis au filtrage complet. Si quelqu’un affirme qu’un connecteur a été configuré pour ce système, ce n’est manifestement plus le cas.

**Un connecteur mentionne des adresses dont aucun message ne provient.** C’est un candidat à la suppression. Avant de le supprimer, vérifiez s’il s’agit de systèmes saisonniers ou rares, puis désactivez-le d’abord plutôt que de le retirer immédiatement.

**Un connecteur définit `TreatMessagesAsInternal` ou `CloudServicesMailEnabled` sur vrai.** Cela mérite un examen attentif. Ces deux paramètres font que les messages transitant par ce chemin sont traités comme internes à l’organisation. Si des e-mails provenant d’Internet arrivent par ce biais, ils contournent donc des contrôles conçus pour les messages externes, notamment la protection contre les expéditeurs usurpés de votre propre domaine. Pour un connecteur hybride pur, c’est correct ; pour un connecteur par lequel des systèmes quelconques remettent des e-mails, c’est un constat à traiter.

## Ce que vous trouverez généralement

Dans la pratique, sans prétention d’exhaustivité : un connecteur de test issu d’une migration, actif depuis des années. Une application métier qui envoie directement au service alors que tout le monde pense qu’elle passe par la passerelle. Un prestataire de newsletters dont le contrat a expiré mais qui peut toujours remettre des messages. Et régulièrement un connecteur aux conditions très larges, créé un jour pour résoudre un problème urgent.

Aucune de ces découvertes n’est dramatique prise isolément. Ensemble, elles dessinent l’image d’un environnement dont plus personne n’a une vue complète, et c’est précisément le véritable risque.

## Limites de la méthode

Vous devez connaître trois limites.

Le suivi des messages via la cmdlet ne remonte qu’à environ dix jours. Pour des périodes plus longues, vous avez besoin de la recherche historique, qui s’exécute de manière asynchrone et couvre jusqu’à 90 jours. Les systèmes rares qui envoient des messages mensuellement vous échapperaient sinon.

`FromIP` ne signifie pas la même chose partout. Pour les e-mails provenant d’Internet, il s’agit de l’adresse du serveur d’envoi. Pour les e-mails hybrides, il s’agit de l’adresse de votre serveur de transport local, et non de celle de l’expéditeur d’origine. L’analyse vous montre donc la **dernière étape avant le service**, pas l’origine.

Et l’attribution à un connecteur n’est pas directement visible dans le tenant. Vous la déduisez de l’adresse, du certificat et du domaine expéditeur. Pour une déclaration fiable sur l’utilisation d’un connecteur individuel, le rapport sur les connecteurs dans le Centre d’administration Exchange, sous Rapports et Flux de messagerie, est une meilleure source, car il agrège les données côté serveur sur des périodes plus longues.

## Comme contrôle récurrent

Cette analyse se prête bien à une routine trimestrielle. Conservez le résultat et comparez-le lors du passage suivant. Les nouvelles adresses dans la liste correspondent soit à des changements documentés, soit à quelque chose que vous devez connaître.

Si vous vérifiez déjà la configuration de messagerie de vos domaines à cette occasion : le [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check) affiche SPF, DKIM, DMARC et les autres normes de messagerie pour n’importe quel domaine directement dans le navigateur, y compris les domaines secondaires et marketing qui, d’expérience, sont oubliés lors de tels inventaires. Et pour les requêtes elles-mêmes, le [Générateur de commandes](https://rafaelpfister.ch/tools/command-builder) fournit des blocs prêts à l’emploi pour PowerShell et les shells Unix.

La manière de suivre des messages suspects individuels est expliquée dans [Analyser le flux de messagerie Exchange : suivi des messages, protocoles SMTP et connecteurs de réception](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren).

## Sources

1.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2) : liste des champs, y compris FromIP et ToIP, ainsi que la pagination avec StartingRecipientAddress.

2.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch) : suivi asynchrone des messages sur une période allant jusqu’à 90 jours pour les périodes plus anciennes.

3.  [Get-InboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchange/get-inboundconnector) : paramètres des connecteurs entrants, notamment SenderIPAddresses et TreatMessagesAsInternal.

4.  [Configurer le flux de messagerie à l’aide de connecteurs dans Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow) : interaction entre les types de connecteurs et conditions d’application de chacun.

5.  [RFC 1918 : Allocation d’adresses pour les réseaux Internet privés](https://www.rfc-editor.org/rfc/rfc1918) : définit les plages d’adresses privées que vous devez distinguer des adresses publiques dans l’analyse.
