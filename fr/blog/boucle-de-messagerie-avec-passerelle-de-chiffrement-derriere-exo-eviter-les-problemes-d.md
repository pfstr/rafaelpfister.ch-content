---
title: "Boucle de messagerie avec passerelle de chiffrement derrière EXO : éviter les problèmes d’usurpation"
navTitle: "Boucle de passerelle"
description: "Si la passerelle de chiffrement (HIN dans cet exemple) se trouve derrière Exchange Online, chaque message entrant arrive une seconde fois dans EOP : avec un domaine d’expéditeur tiers, une signature DKIM non valide et l’adresse IP de la passerelle comme source. Il en résulte un verdict d’usurpation et un placement dans les courriers indésirables. Quatre paramètres évitent cela sans désactiver le filtrage via SCL -1 : enregistrement PTR, Enhanced Filtering désactivé, exception d’usurpation pour l’infrastructure de la passerelle et CloudServicesMailEnabled sur les deux connecteurs."
date: "2026-09-23"
kategorie: "Flux de messagerie et SMTP"
timeToRead: "9 min de lecture"
themen:
  - smtp-mailflow
  - microsoft-365-exchange
  - hin-gateway
  - e-mail-verschluesselung
produkte:
  - "exchange-online"
  - "hybrid-mailfluss"
  - "hin"
protokolle:
  - "mail-auth"
  - "smtp"
  - "powershell"
slug: "boucle-de-messagerie-avec-passerelle-de-chiffrement-derriere-exo-eviter-les-problemes-d"
translationId: "article-b773c8d303aee87a"
aiPrompt: |
  Du bist mein Exchange-Online-Assistent. Ich betreibe ein Verschlüsselungsgateway hinter Exchange Online (Mail-Schlaufe: EXO, Gateway, EXO). Prüfe mit mir die vier Einstellungen: PTR-Record der Gateway-IP, Enhanced Filtering auf dem Inbound-Connector, Spoof-Ausnahme in der Tenant Allow/Block List für die Gateway-Infrastruktur und CloudServicesMailEnabled auf beiden Connectoren. Erkläre mir zu jeder Einstellung, warum sie nötig ist, und hilf mir, das Ergebnis anhand der Authentication-Results-Header einer Testnachricht zu verifizieren.
translationOf: verschluesselungsgateway-hinter-exchange-online
url: https://rafaelpfister.ch/fr/blog/boucle-de-messagerie-avec-passerelle-de-chiffrement-derriere-exo-eviter-les-problemes-d
translationSourceHash: ac3ae7b36a2682c2913479a6d9d0356d1abd96c68459bcfb4912362884e0a708
translationModel: gpt-5.6-terra
translatedAt: 2026-09-24T08:34:44.633Z
translationReview: automatic
---

# Boucle de messagerie avec passerelle de chiffrement derrière EXO : éviter les problèmes d’usurpation

De nombreuses organisations du secteur suisse de la santé exploitent une passerelle HIN, d’autres une passerelle SEPPmail ou TotemoMail pour S/MIME et PGP. Si l’enregistrement MX pointe vers Microsoft et que la passerelle se trouve derrière Exchange Online, chaque message entrant passe deux fois par le filtrage. Lors du second passage, EOP voit un message avec un domaine d’expéditeur tiers, une signature DKIM non valide et l’adresse IP de la passerelle comme source. Du point de vue du filtre, il s’agit du schéma d’une usurpation d’expéditeur, et les messages légitimes finissent dans le dossier des courriers indésirables. J’ai configuré cette architecture à plusieurs reprises et décris ici les quatre paramètres qui permettent à la boucle de fonctionner sans verdict d’usurpation, ainsi que la raison pour laquelle chacun est nécessaire. L’exemple utilise la passerelle HIN ; le mécanisme est identique pour toute autre passerelle placée à la même position.

## L’architecture

```text
Absender > MX > Exchange Online (1. Durchlauf)
                    > HIN-Gateway: Entschlüsselung, Signaturprüfung
                            > Exchange Online (2. Durchlauf) > Postfach
```

Une règle de transport achemine les messages entrants vers la passerelle via un connecteur sortant. La passerelle déchiffre, vérifie les signatures et remet à nouveau le message à Exchange Online via un connecteur entrant. Un champ d’en-tête défini par la passerelle empêche la règle de s’appliquer à nouveau.

Cette architecture a de bonnes raisons d’être : Microsoft filtre d’abord, la passerelle ne reçoit que des messages déjà contrôlés et l’exploitation ne doit pas exposer son propre MX sur Internet. Le prix à payer est le second passage, qui ne peut pas être désactivé, seulement configuré correctement.

## Pourquoi SCL -1 est la mauvaise réponse

La solution répandue consiste en une règle de transport sur le chemin de retour qui définit le Spam Confidence Level sur `-1`. Exchange Online ignore alors entièrement l’analyse du contenu lors du second passage. Cela fonctionne immédiatement, mais présente deux inconvénients : le second passage ne sert ensuite plus à rien et la règle repose sur une condition (connecteur ou IP) qui change à chaque modification. Microsoft décrit par ailleurs SCL -1 comme une entrée pour le filtrage, et non comme une décision finale ; la valeur apposée peut différer. Les quatre paramètres ci-dessous résolvent le problème là où il survient : lors de l’évaluation de l’infrastructure de remise.

## Les quatre paramètres

1. Publier un enregistrement PTR dans le DNS public pour l’adresse IP publique de la passerelle HIN. Sans cet enregistrement, l’ensemble du chemin ne fonctionne pas.
2. Désactiver entièrement Enhanced Filtering sur le connecteur entrant par lequel HIN remet les messages à Exchange Online.
3. Créer une exception d’usurpation pour l’infrastructure HIN de remise dans la Tenant Allow/Block List.
4. Activer `CloudServicesMailEnabled` sur le connecteur sortant vers HIN et sur le connecteur entrant depuis HIN.

Pour le point 2, vérifiez d’abord quels connecteurs sont concernés :

```powershell
Get-InboundConnector |
    Select-Object Name, Enabled, ConnectorType, EFSkipLastIP, EFSkipIPs, EFUsers, EFTestMode |
    Format-List
```

Désactivez-le ensuite sur le connecteur HIN :

```powershell
$efAus = @{
    Identity     = "<Inbound-Connector HIN>"
    EFSkipLastIP = $false
    EFSkipIPs    = $null
    EFUsers      = $null
}
Set-InboundConnector @efAus
```

<details class="options-details">
<summary>Explication des options</summary>

| Paramètre | Effet |
|---|---|
| `EFSkipLastIP = $false` | Le dernier saut n’est plus automatiquement ignoré. Si aucune adresse IP ne figure simultanément dans `EFSkipIPs`, Enhanced Filtering est désactivé sur le connecteur. |
| `EFSkipIPs = $null` | Vide la liste des adresses IP à ignorer. |
| `EFUsers = $null` | Supprime la restriction à certains destinataires ; sans Enhanced Filtering actif, la valeur n’a aucune importance, mais reste ainsi sans ambiguïté. |
| `EFTestMode` | Uniquement dans la requête : indique si le connecteur est en mode test. Microsoft qualifie le paramètre d’interne, mais il est lisible. |

</details>

Point 3, l’exception d’usurpation :

```powershell
$spoof = @{
    Identity              = "Default"
    Action                = "Allow"
    SpoofedUser           = "*"
    SendingInfrastructure = "gateway.example.com"
    SpoofType             = "External"
}
New-TenantAllowBlockListSpoofItems @spoof
```

<details class="options-details">
<summary>Explication des options</summary>

| Paramètre | Effet |
|---|---|
| `Identity = "Default"` | La liste elle-même ; il n’en existe qu’une seule. |
| `Action = "Allow"` | Autorise la combinaison. `Block` la classe au contraire comme phishing. |
| `SpoofedUser = "*"` | L’adresse visible dans le champ From. Le caractère générique représente n’importe quel expéditeur. Un caractère générique est autorisé d’un côté de la paire, mais pas des deux. |
| `SendingInfrastructure` | La source : le domaine de l’enregistrement PTR de l’IP de remise (point 1). Sans enregistrement PTR, la liste n’accepte que `<IP>/24`. |
| `SpoofType = "External"` | S’applique aux domaines d’expéditeur tiers. `Internal` couvre les domaines acceptés propres et nécessite une seconde entrée. |

</details>

Point 4, les en-têtes cross-premises :

```powershell
Set-OutboundConnector -Identity "<Outbound-Connector zu HIN>" -CloudServicesMailEnabled $true
$crossPremises = @{
    Identity                 = "<Inbound-Connector HIN>"
    TreatMessagesAsInternal  = $false
    CloudServicesMailEnabled = $true
}
Set-InboundConnector @crossPremises
```

J’explique plus bas, au point 4, pourquoi `TreatMessagesAsInternal` figure dans la même commande pour le connecteur entrant.

## 1 : Enregistrement PTR

EOP identifie l’infrastructure de remise au moyen de la recherche inverse de l’adresse IP source. La valeur PTR apparaît dans l’en-tête `Authentication-Results` comme sending infrastructure, et c’est précisément cette valeur à laquelle se rapporte l’exception d’usurpation du point 3. En l’absence d’enregistrement PTR, Exchange Online évalue chaque message sur le chemin de retour avec `PTR:InfoDomainNonexistent`, et l’exception doit alors s’appliquer à un `/24` entier plutôt qu’à un nom. L’absence d’entrée DNS inverse constitue en outre depuis longtemps un signal négatif établi dans tout filtre anti-spam. L’entrée devrait à son tour se résoudre vers la même adresse IP.

Avec HIN, l’adresse IP de remise est celle de la passerelle de messagerie HIN par laquelle les messages reviennent vers Exchange Online. L’entité qui gère l’enregistrement PTR de cette IP dépend du modèle d’exploitation ; la responsabilité doit être clarifiée avant la bascule.

## 2 : Désactiver Enhanced Filtering

Enhanced Filtering for Connectors (Skip Listing) est conçu pour l’architecture inverse : une passerelle devant Microsoft 365, avec le MX pointant vers la passerelle. EOP y ignore le dernier saut et évalue l’origine réelle du message.

Dans une configuration en boucle, cela ne fonctionne pas. Microsoft précise dans sa documentation qu’Enhanced Filtering n’est pas prévu pour les services qui traitent les messages après Microsoft 365, et considère le routage non linéaire (Internet, Microsoft 365, système externe, Microsoft 365) comme non pris en charge. La documentation indique précisément le symptôme observé dans la pratique : Microsoft 365 vérifie à nouveau le courrier retourné, attribue une valeur `compauth`, et le message peut être classé comme spam.

Si Enhanced Filtering reste actif, EOP ignore l’IP HIN et évalue l’IP précédente. Dans la boucle, il s’agit soit d’une IP appartenant à Microsoft issue du premier passage, soit de l’expéditeur externe d’origine. Dans les deux cas, c’est incorrect :

* L’évaluation porte sur une IP qui ne permet plus de tirer des conclusions sur le message lors du second passage.
* L’exception d’usurpation du point 3 ne s’applique jamais, car elle est liée à l’infrastructure HIN qu’EOP vient justement d’ignorer.

Enhanced Filtering doit donc être désactivé sur ce connecteur : ce n’est qu’alors que HIN est visible et identifiable comme infrastructure de remise.

## 3 : Exception d’usurpation

Une entrée d’usurpation est toujours une paire composée de Spoofed user (l’adresse From ou son domaine) et de Sending infrastructure (la source). Seule cette combinaison est autorisée.

`SpoofedUser = "*"` associé à l’infrastructure HIN signifie donc : toute adresse From peut remettre via HIN sans que le verdict d’usurpation ne s’applique. Toute autre source utilisant les mêmes expéditeurs continue d’être vérifiée. L’exception ne contourne pas le filtre anti-spam : les vérifications du spam, du contenu et des menaces continuent de s’exécuter sans modification lors du second passage. Un message peut donc toujours être écarté en raison de son contenu, mais il n’est simplement plus considéré comme une falsification.

La commande ci-dessus contient délibérément uniquement `SpoofType = "External"`. Les propres messages, c’est-à-dire les messages dont l’expéditeur appartient aux domaines acceptés propres, ne devraient pas revenir via la passerelle. Ils vont des systèmes internes ou d’Exchange OnPrem directement vers Exchange Online, sans passer par HIN. L’analyse Spoof Intelligence indique si c’est effectivement le cas dans votre environnement :

```powershell
Get-SpoofIntelligenceInsight |
    Select-Object SpoofedUser, SendingInfrastructure, SpoofType, MessageCount, Action |
    Sort-Object MessageCount -Descending |
    Format-Table -AutoSize |
    Out-String -Width 200
```

Si vos propres domaines y apparaissent avec l’infrastructure HIN et `SpoofType Internal`, une seconde entrée avec `SpoofType = "Internal"` est nécessaire, ou mieux encore : corriger la cause pour laquelle le courrier interne passe par la passerelle.

Les entrées d’usurpation n’expirent pas d’elles-mêmes. Si la passerelle est supprimée ou si son infrastructure change, l’entrée doit être supprimée.

## 4 : CloudServicesMailEnabled

Ce point est le seul que Microsoft ne documente pas explicitement pour ce scénario. Il s’agit de ma déduction à partir du comportement documenté du paramètre : il préserve les en-têtes cross-premises à travers la boucle.

Le paramètre contrôle le traitement des en-têtes internes `X-MS-Exchange-Organization-*`. Sur le connecteur sortant, ils sont convertis en `X-MS-Exchange-CrossPremises-*` et survivent ainsi au trajet via HIN. Sur le connecteur entrant, ils sont à nouveau réécrits en `X-MS-Exchange-Organization-*` et remplacent les en-têtes du même nom déjà présents dans le message. Si le paramètre est défini sur `$false`, le connecteur supprime ces en-têtes.

En pratique, cela signifie que ce qu’Exchange Online a déterminé lors du premier passage, notamment l’état d’authentification et le marquage interne, survit à la boucle au lieu d’être supprimé lors de la nouvelle entrée. Dans les en-têtes du message remis figure alors `X-CrossPremisesHeadersPromoted`, et les résultats de vérification d’origine restent dans `Authentication-Results-Original`. Cela n’empêche pas à lui seul un verdict d’usurpation (les points 1 à 3 s’en chargent), mais préserve le verdict du premier passage.

Trois éléments sont à prendre en compte :

* Si les connecteurs ont été créés par le Hybrid Configuration Wizard (`ConnectorSource: HybridWizard`), une exécution ultérieure de HCW écrase le paramètre. Les connecteurs HIN devraient donc être des connecteurs distincts, créés manuellement.
* Sur le connecteur entrant, `CloudServicesMailEnabled $true` et `TreatMessagesAsInternal $true` s’excluent mutuellement. Si `TreatMessagesAsInternal` est déjà défini sur `$true`, Exchange Online rejette la commande. Les deux paramètres doivent donc figurer dans le même appel `Set-InboundConnector`, comme indiqué ci-dessus.
* Microsoft recommande de ne définir le paramètre que sur instruction du support ou d’une documentation produit. Cette documentation n’existe pas pour la boucle ; la décision vous appartient et doit être documentée.

## Sécuriser la remise

Puisqu’après le point 4 Exchange Online reprend les en-têtes du chemin de retour et qu’après le point 3 il accepte toute adresse From provenant de l’infrastructure HIN, le connecteur entrant doit être limité de manière à ce que seul HIN puisse remettre des messages par son intermédiaire. Pour un connecteur de type `Partner`, il s’agit de `RestrictDomainsToCertificate` avec `TlsSenderCertificateName` ou de `RestrictDomainsToIPAddresses` avec `SenderIPAddresses`, dans chaque cas avec `RequireTls`. Sans cette restriction, toute source atteignant le connecteur pourrait remettre des messages avec des expéditeurs arbitraires et des en-têtes d’organisation repris.

```powershell
$absicherung = @{
    Identity                     = "<Inbound-Connector HIN>"
    RequireTls                   = $true
    RestrictDomainsToCertificate = $true
    TlsSenderCertificateName     = "gateway.example.com"
}
Set-InboundConnector @absicherung
```

## Vérification

Après la bascule, envoyez un message de test depuis un domaine externe avec une stratégie DMARC via la passerelle, puis vérifiez les en-têtes du message remis. Dans l’en-tête `Authentication-Results` du second passage, le domaine HIN doit apparaître comme sending infrastructure et `compauth` doit être défini sur `pass` avec un motif issu de la catégorie des entrées Tenant Allow, et non plus sur `fail reason=001`. `X-CrossPremisesHeadersPromoted` et `Authentication-Results-Original` indiquent que le point 4 fonctionne. L’[analyseur d’en-têtes de messagerie](/tools/header-analyzer) de ce site représente les deux passages sous forme de diagramme de flux et marque les sauts entre Exchange Online et la passerelle.

## Sources

1.  [Enhanced filtering for connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): délimitation entre routage linéaire et non linéaire, indication concernant les services derrière Microsoft 365, conséquence `compauth` et classification comme spam, SCL -1 comme entrée plutôt que comme décision, paramètres PowerShell `EFSkipLastIP`, `EFSkipIPs`, `EFUsers`.

2.  [Set-InboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-inboundconnector): comportement de `CloudServicesMailEnabled` (conversion et promotion des en-têtes cross-premises, suppression avec `$false`), exclusion avec `TreatMessagesAsInternal`, `RestrictDomainsToCertificate` et `RestrictDomainsToIPAddresses` pour les connecteurs partenaires.

3.  [Set-OutboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-outboundconnector): `CloudServicesMailEnabled` sur le connecteur sortant.

4.  [Allow or block email using the Tenant Allow/Block List](https://learn.microsoft.com/en-us/defender-office-365/tenant-allow-block-list-email-spoof-configure): syntaxe des entrées d’usurpation, règles des caractères génériques, détermination de l’infrastructure d’envoi via l’enregistrement PTR ou `/24`, couverture des verdicts d’usurpation liés à DMARC.

5.  [New-TenantAllowBlockListSpoofItems](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-tenantallowblocklistspoofitems): paramètres `SpoofedUser`, `SendingInfrastructure`, `SpoofType`, `Action`.

6.  [Get-SpoofIntelligenceInsight](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-spoofintelligenceinsight): analyse des paires d’usurpation détectées au cours des sept derniers jours.
