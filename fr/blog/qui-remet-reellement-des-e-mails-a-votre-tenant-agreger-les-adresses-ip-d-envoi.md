---
title: "Qui remet réellement des e-mails à votre tenant ? Agréger les adresses IP d’envoi"
navTitle: "IP d’envoi"
description: "Une seule analyse montre quels systèmes remettent réellement des e-mails à votre tenant : connecteurs oubliés, applications qui envoient directement et prestataires que personne n’a documentés. Y compris les pièges liés à la pagination et à l’interprétation."
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
url: https://rafaelpfister.ch/fr/blog/qui-remet-reellement-des-e-mails-a-votre-tenant-agreger-les-adresses-ip-d-envoi
translationSourceHash: 9dc48329a06945f705380eb3db428efb548f0c36a1fe3c4f2fb7de1185fee879
translationModel: gpt-5.6-terra
translatedAt: 2026-08-12T05:12:21.208Z
translationReview: required
---

# Qui remet réellement des e-mails à votre tenant ? Agréger les adresses IP d’envoi

Presque aucun environnement de messagerie ne sait encore parfaitement qui lui remet des e-mails. Au fil des ans s’accumulent des connecteurs issus de migrations, des applications qui envoient directement, des prestataires dont le contrat a expiré depuis longtemps et des environnements de test qui n’ont jamais été démantelés. Tant que les e-mails circulent, personne ne s’en aperçoit.

Une seule analyse permet d’y voir clair : le regroupement de tous les messages entrants par adresse IP source. Elle se réalise en deux minutes, et la liste des résultats réserve régulièrement des surprises. Cet article présente la requête, explique comment l’obtenir de manière **complète** et, surtout, comment interpréter correctement les chiffres. Car l’interprétation est la partie la plus difficile.

## Pourquoi cela en vaut la peine

La liste répond à quatre questions qu’il serait sinon fastidieux de clarifier séparément. Quels systèmes envoient réellement des e-mails à votre tenant ? Tout passe-t-il par les voies que vous avez documentées, ou existe-t-il une deuxième entrée ? Un connecteur que vous souhaitez supprimer est-il encore utilisé ? Et : une application envoie-t-elle directement au service en contournant votre passerelle, donc en évitant votre filtrage ?

La dernière question en particulier est pertinente pour la sécurité. Quiconque remet directement des e-mails contourne non seulement le filtrage, mais souvent aussi la journalisation sur laquelle vous souhaitez pouvoir compter en cas d’incident.

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

## Piège 1 : la liste est presque toujours incomplète

`Get-MessageTraceV2` renvoie les résultats par pages, avec un maximum de 5 000 lignes par appel. En cas de volume élevé, une page ne couvre qu’une fraction de votre fenêtre temporelle. Vous regroupez alors un extrait et prenez le résultat pour l’ensemble.

Cela se reconnaît à cet avertissement :

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

**Si cet avertissement apparaît, votre analyse ne vaut rien.** En particulier, l’absence d’une entrée ne doit alors pas être interprétée comme une absence réelle. Une adresse avec trois messages par jour n’apparaîtra de toute façon pas dans un extrait.

Il existe deux solutions. La plus simple : réduisez la fenêtre temporelle jusqu’à ce que l’avertissement disparaisse. Avec 5 000 messages par heure, cela signifie 55 minutes et non sept jours. Pour répondre à la question « quels systèmes envoient réellement des e-mails », une courte fenêtre complète suffit généralement amplement.

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

Pour 24 heures dans un environnement de taille moyenne, comptez quelques minutes d’exécution. Pour un inventaire ponctuel, c’est un investissement judicieux.

## Piège 2 : les chiffres ne signifient pas ce qu’ils semblent signifier

La liste des résultats contient quatre types d’entrées fondamentalement différents, et les mettre dans le même panier conduit à de mauvaises conclusions.

**`255.255.255.255` ne représente pas un système.** Cette valeur apparaît lorsqu’il n’y a pas eu de connexion SMTP entrante depuis l’extérieur pour le message. Cela concerne les messages générés dans le service lui-même : rapports de journalisation, rapports de non-remise, réponses automatiques d’absence, messages entre des boîtes aux lettres du même tenant. Dans presque tous les environnements, c’est le poste le plus important, et il est parfaitement normal. Ne vous inquiétez pas.

**Les adresses privées de la RFC 1918** proviennent de votre propre réseau. Dans les environnements hybrides, vous y voyez les serveurs de transport locaux, car leur adresse interne est conservée lors de la remise au service. Ce sont les chiffres élevés de la liste et, dans la grande majorité des cas, la voie principale attendue.

**Les adresses des services de sécurité et de filtrage** s’identifient par leur opérateur, non par leur valeur numérique. Les proxys cloud, passerelles de messagerie en amont et services de sécurité web apparaissent avec de nombreuses adresses voisines et des volumes moyens. Ils en font généralement partie, mais devraient figurer dans le manuel d’exploitation.

**Les adresses publiques isolées avec de faibles volumes** sont les plus intéressantes. C’est précisément là que se cachent les applications oubliées, les anciens prestataires et les systèmes dont plus personne ne se souvient.

## La résolution : des adresses aux noms

Pour tout ce que vous ne pouvez pas attribuer immédiatement, la résolution inverse est utile. Elle n’est pas toujours configurée et n’est pas toujours fiable, mais dans la majorité des cas elle fournit l’indice décisif :

```powershell
$unbekannt = '198.51.100.77','203.0.113.9'

$unbekannt | ForEach-Object {
    $name = try { (Resolve-DnsName $_ -Type PTR -ErrorAction Stop).NameHost } catch { '(kein PTR)' }
    [pscustomobject]@{ IP = $_; Name = $name }
} | Format-Table -AutoSize
```

```text
IP            Name
--            ----
198.51.100.77 mail-out-03.newsletter-provider.example
203.0.113.9   (kein PTR)
```

L’absence de PTR n’est pas la preuve de quelque chose de malveillant, mais c’est une bonne raison d’examiner la situation de plus près. Pour de telles adresses, examinez les messages associés :

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date) -ResultSize 5000 |
  Where-Object { $_.FromIP -eq '203.0.113.9' } |
  Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize
```

L’expéditeur et l’objet vous indiquent généralement immédiatement quelle application se trouve derrière.

## Le rapprochement : quelle adresse appartient à quel connecteur ?

Voici maintenant le véritable gain de connaissance. Comparez votre liste de résultats aux connecteurs configurés :

```powershell
Get-InboundConnector |
    Format-List Name, Enabled, ConnectorType, ConnectorSource,
        @{n='SenderIPAddresses'; e={$_.SenderIPAddresses -join ', '}},
        @{n='SenderDomains';     e={$_.SenderDomains -join ', '}},
        RequireTls, TlsSenderCertificateName,
        RestrictDomainsToCertificate, CloudServicesMailEnabled, TreatMessagesAsInternal
```

Trois configurations sont révélatrices.

**Une adresse remet des e-mails, mais n’est mentionnée dans aucun connecteur.** L’e-mail arrive alors comme un e-mail Internet ordinaire. C’est autorisé, mais cela signifie que cette application ne bénéficie d’aucun traitement particulier et que ses messages sont soumis au filtrage complet. Si quelqu’un affirme qu’un connecteur est configuré pour ce système, ce n’est manifestement plus le cas.

**Un connecteur mentionne des adresses dont rien ne provient.** C’est un candidat à la suppression. Avant de le supprimer, vérifiez s’il s’agit de systèmes saisonniers ou rarement utilisés, et désactivez-le d’abord plutôt que de le retirer immédiatement.

**Un connecteur définit `TreatMessagesAsInternal` ou `CloudServicesMailEnabled` sur vrai.** Cela mérite un examen attentif. Ces deux paramètres font que les messages arrivant par cette voie sont traités comme internes à l’organisation. Si des e-mails provenant d’Internet arrivent par là, ils contournent ainsi les contrôles prévus pour les messages externes, notamment la protection contre les expéditeurs falsifiés utilisant votre propre domaine. C’est correct pour un connecteur hybride pur ; pour un connecteur par lequel des systèmes quelconques remettent des e-mails, c’est un constat à prendre en compte.

## Ce que vous trouvez généralement

Dans la pratique, sans prétention d’exhaustivité : un connecteur de test issu d’une migration, actif depuis des années. Une application métier qui envoie directement au service alors que tout le monde pense qu’elle passe par la passerelle. Un prestataire de newsletters dont le contrat a expiré, mais qui peut toujours livrer des e-mails. Et régulièrement un connecteur aux conditions très ouvertes, créé un jour pour résoudre un problème urgent.

Aucune de ces découvertes n’est dramatique en soi. Ensemble, elles dessinent l’image d’un environnement dont plus personne n’a une vue complète, et c’est précisément là que réside le véritable risque.

## Limites de la méthode

Vous devez connaître trois limitations.

Le suivi des messages via le cmdlet ne remonte qu’à une dizaine de jours environ. Pour des périodes plus longues, vous avez besoin de la recherche historique, qui fonctionne de manière asynchrone et couvre jusqu’à 90 jours. Sinon, les systèmes rares qui envoient une fois par mois vous échapperont.

`FromIP` ne signifie pas la même chose partout. Pour les e-mails venant d’Internet, il s’agit de l’adresse du serveur qui remet le message. Pour les e-mails hybrides, il s’agit de l’adresse de votre serveur de transport local, et non de celle de l’expéditeur d’origine. L’analyse vous montre donc la **dernière étape avant le service**, et non l’origine.

Et l’association à un connecteur n’est pas directement visible dans le tenant. Vous la déduisez de l’adresse, du certificat et du domaine de l’expéditeur. Pour une déclaration fiable sur l’utilisation d’un connecteur donné, le rapport sur les connecteurs dans le Centre d’administration Exchange, sous Rapports et Flux de messagerie, est une meilleure source, car il agrège côté serveur sur des périodes plus longues.

## Comme contrôle récurrent

Cette analyse convient bien comme routine trimestrielle. Archivez le résultat et comparez-le lors du passage suivant. Les nouvelles adresses dans la liste correspondent soit à des modifications documentées, soit à quelque chose que vous souhaitez connaître.

Si vous vérifiez de toute façon la configuration de messagerie de vos domaines : le [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check) affiche SPF, DKIM, DMARC et les autres standards de messagerie pour n’importe quel domaine directement dans le navigateur, y compris pour les domaines secondaires et marketing qui sont, selon l’expérience, oubliés lors de tels inventaires. Et pour les requêtes elles-mêmes, le [Générateur de commandes](https://rafaelpfister.ch/tools/command-builder) fournit des blocs prêts à l’emploi pour PowerShell et les shells Unix.

La manière de suivre des messages suspects individuels est expliquée dans [Analyser le flux de messagerie Exchange : suivi des messages, journaux SMTP et connecteurs de réception](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren).

## Sources

1.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2) : liste des champs, y compris FromIP et ToIP, ainsi que la logique de pagination avec StartingRecipientAddress.

2.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch) : suivi asynchrone des messages sur une période allant jusqu’à 90 jours pour les périodes plus anciennes.

3.  [Get-InboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchange/get-inboundconnector) : paramètres des connecteurs entrants, notamment SenderIPAddresses et TreatMessagesAsInternal.

4.  [Configurer le flux de messagerie à l’aide de connecteurs dans Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow) : interaction entre les types de connecteurs et conditions dans lesquelles chacun s’applique.

5.  [RFC 1918: Address Allocation for Private Internets](https://www.rfc-editor.org/rfc/rfc1918) : définit les plages d’adresses privées que vous devez distinguer des adresses publiques dans l’analyse.
