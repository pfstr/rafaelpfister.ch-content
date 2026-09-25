---
title: "MailHeaderAnalyzer : analyser les en-têtes d’e-mails dans PowerShell sans accès réseau"
navTitle: "MailHeaderAnalyzer (PowerShell)"
description: "Référence du module PowerShell MailHeaderAnalyzer : aperçu des paramètres, syntaxe, description, exemples et propriétés des paramètres de Get-MailHeaderAnalysis et ConvertTo-MailHeaderReport, ainsi que l’objet de sortie avec la chaîne de distribution, les résultats d’authentification, la classification Exchange Online et tous les codes de constat."
date: "2026-09-24"
kategorie: "SMTP & flux de messagerie"
timeToRead: "16 min de lecture"
themen:
  - smtp-mailflow
  - microsoft-365-exchange
hauptthema: "smtp-mailflow"
produkte:
  - "exchange-online"
  - "uebergreifend"
protokolle:
  - "powershell"
  - "mail-auth"
  - "smtp"
  - "troubleshooting"
related:
  - e-mail-header-analysieren-ohne-upload
  - microsoft-365-compauth-reason-codes
  - exchange-hybrid-header-intern-extern
slug: "mailheaderanalyzer-analyser-les-en-tetes-d-e-mails-dans-powershell-sans-acces-reseau"
translationId: "article-041d7b2f9615f670"
translationOf: mailheaderanalyzer-powershell-modul
url: https://rafaelpfister.ch/fr/blog/mailheaderanalyzer-analyser-les-en-tetes-d-e-mails-dans-powershell-sans-acces-reseau
translationSourceHash: 7c01d42e83d3aa175486c74e2f88855bc7d9dee519026ce748edb239fdd355cc
translationModel: gpt-5.6-terra
translatedAt: 2026-09-24T18:51:21.192Z
translationReview: automatic
---

# MailHeaderAnalyzer : analyser les en-têtes d’e-mails dans PowerShell sans accès réseau

MailHeaderAnalyzer est un module PowerShell doté de deux cmdlets. `Get-MailHeaderAnalysis` analyse l’en-tête d’un e-mail : chaîne de distribution avec délais et informations TLS, résultats SPF, DKIM, DMARC et ARC, y compris la vérification que ces résultats proviennent du serveur destinataire, alignement DMARC, classification hybride d’Exchange Online, évaluations de Microsoft Defender, SpamAssassin et Rspamd, ainsi que des anomalies telles que des lignes `From` en double ou des caractères de contrôle Unicode. `ConvertTo-MailHeaderReport` génère à partir de cela un rapport pour les tickets. Le module fonctionne entièrement hors ligne : aucune requête DNS, aucune connexion HTTP. C’est la version en ligne de commande de l’[analyseur d’en-têtes de ce site web](/tools/header-analyzer) et il utilise la même logique d’analyse.

| | |
|---|---|
| Module | [MailHeaderAnalyzer dans la PowerShell Gallery](https://www.powershellgallery.com/packages/MailHeaderAnalyzer) |
| Code source | [pfstr/MailHeaderAnalyzer sur GitHub](https://github.com/pfstr/MailHeaderAnalyzer), licence MIT |
| S’applique à | Windows PowerShell 5.1, PowerShell 7.x ; Windows, Linux, macOS ; Exchange Management Shell |
| Cmdlets | `Get-MailHeaderAnalysis`, `ConvertTo-MailHeaderReport` |

## Aperçu des paramètres

| Cmdlet | Paramètre | Type | Obligatoire | Pipeline | Effet |
|---|---|---|---|---|---|
| `Get-MailHeaderAnalysis` | `-Header` | `String[]` | Oui (jeu de paramètres Text) | Oui, par valeur | L’en-tête sous forme de texte. Les lignes du pipeline sont assemblées en un en-tête |
| `Get-MailHeaderAnalysis` | `-Path` | `String[]` | Oui (jeu de paramètres Path) | Oui, par nom de propriété | Fichier contenant l’en-tête ou le message `.eml` complet ; accepte les objets de `Get-ChildItem` |
| `Get-MailHeaderAnalysis` | `-FromClipboard` | `Switch` | Oui (jeu de paramètres Clipboard) | Non | Lit l’en-tête depuis le presse-papiers (Windows uniquement) |
| `ConvertTo-MailHeaderReport` | `-Analysis` | `MailHeaderAnalyzer.Analysis` | Oui | Oui, par valeur | L’objet de résultat de `Get-MailHeaderAnalysis` |
| `ConvertTo-MailHeaderReport` | `-Format` | `String` | Non | Non | `Markdown` (par défaut) ou `Text` |

Les deux cmdlets prennent en charge les paramètres communs `-Verbose`, `-ErrorAction`, `-ErrorVariable`, `-OutVariable` et les autres décrits dans [about_CommonParameters](https://learn.microsoft.com/de-de/powershell/module/microsoft.powershell.core/about/about_commonparameters).

## Installation

```powershell
Install-Module -Name MailHeaderAnalyzer -Scope CurrentUser
```

| Option | Effet |
|---|---|
| `-Name MailHeaderAnalyzer` | Nom du module dans la PowerShell Gallery |
| `-Scope CurrentUser` | Installe dans le répertoire de modules de l’utilisateur, sans droits d’administrateur |

Sur un système sans accès Internet, téléchargez le module sur un autre ordinateur avec `Save-Module -Name MailHeaderAnalyzer -Path C:\Temp` et copiez le dossier `MailHeaderAnalyzer` dans un répertoire figurant dans `$env:PSModulePath`, par exemple `%USERPROFILE%\Documents\WindowsPowerShell\Modules` (Windows PowerShell 5.1) ou `%USERPROFILE%\Documents\PowerShell\Modules` (PowerShell 7). `Update-Module -Name MailHeaderAnalyzer` récupère les mises à jour, et `Get-Module -Name MailHeaderAnalyzer -ListAvailable` affiche la version installée.

## Get-MailHeaderAnalysis

Analyse l’en-tête d’un e-mail et renvoie un objet d’analyse.

### Syntaxe

#### Text (par défaut)

```powershell
Get-MailHeaderAnalysis
    [-Header] <String[]>
    [<CommonParameters>]
```

#### Path

```powershell
Get-MailHeaderAnalysis
    -Path <String[]>
    [<CommonParameters>]
```

#### Clipboard

```powershell
Get-MailHeaderAnalysis
    -FromClipboard
    [<CommonParameters>]
```

### Description

La cmdlet décompose l’en-tête brut en champs, déplie les replis RFC 5322 et décode les valeurs RFC 2047 dans l’objet et les adresses. À partir des lignes `Received`, elle constitue la chaîne de distribution dans l’ordre chronologique, calcule le délai de chaque relais et lit la version TLS, le chiffrement et la classe de protocole selon RFC 3848. À partir de `Authentication-Results`, `Received-SPF`, `DKIM-Signature` et de la chaîne ARC, elle détermine les résultats d’authentification et vérifie si la ligne de vérification provient effectivement d’une station de la chaîne de distribution (RFC 8601, section 5). S’y ajoutent l’alignement DMARC, la classification hybride d’Exchange Online, les évaluations des filtres antispam et une liste d’anomalies.

La cmdlet n’effectue aucune requête DNS et n’ouvre aucune connexion réseau. `Spf`, `Dkim` et `Dmarc` correspondent donc toujours au verdict du serveur destinataire, complété par la vérification que ce verdict provient bien de lui. Les signatures DKIM ne sont pas recalculées cryptographiquement.

Les entrées sont lues de manière tolérante : une ligne vide termine l’en-tête, et tout corps de message qui suit est ignoré. Les lignes sans nom de champ et sans espace initial, comme celles produites lors d’une copie depuis les boîtes de dialogue d’un client, appartiennent au champ précédent. Une ligne de séparation mbox `From ...` avant le premier champ est ignorée et une marque d’ordre des octets est supprimée. Au maximum 200 lignes `Received` sont analysées, comptées depuis la remise.

### Exemples

#### Exemple 1

```powershell
Get-MailHeaderAnalysis -FromClipboard
```

Analyse l’en-tête présent dans le presse-papiers. Dans Outlook pour Windows, vous trouverez l’en-tête sous Fichier, Propriétés, En-têtes Internet ; dans Outlook sur le web, dans les options du message sous « Afficher les détails du message ».

```text
Subject     : Service-Report für März
From        : Beispiel Newsletter <news@example.org>
Date        : 2026-08-03 09:14:27Z
SPF         : pass
DKIM        : pass
DMARC       : pass
ARC         : -
CompAuth    : pass (reason 100)
AuthTrust   : Absent (no authserv-id, Microsoft 365 style)
Hops        : 3 (total 00:00:42)
DeliveredBy : zr0p278mb0570.chep278.prod.outlook.com
Findings    : none
```

#### Exemple 2

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml
```

Analyse un message enregistré. Le fichier peut contenir uniquement l’en-tête ou le message complet ; le corps du message est ignoré.

#### Exemple 3

```powershell
(Get-MailHeaderAnalysis -Path .\nachricht.eml).Hops
```

Affiche la chaîne de distribution sous forme de tableau, premier relais en premier.

```text
#   From                 IP              By                                     Protocol   TLS      Time (UTC)           Delay
-   ----                 --              --                                     --------   ---      ----------           -----
1   client.example.net   198.51.100.34   mail.example.org                       ESMTPSA    -        2026-08-03 09:14:28  -
2   mail.example.org     203.0.113.25    mx.eur02.prod.protection.outlook.com   Microsoft… TLS 1.3  2026-08-03 09:15:09  41 s
3   AM0EUR02FT056.eop…   -               ZR0P278MB0570.CHEP278.PROD.OUTLOOK.COM Microsoft… TLS 1.2  2026-08-03 09:15:10  1 s
```

#### Exemple 4

```powershell
Get-Content -Path .\header.txt | Get-MailHeaderAnalysis | Select-Object -ExpandProperty Findings
```

Lit l’en-tête ligne par ligne depuis un fichier texte et n’affiche que les anomalies. `-Raw` avec `Get-Content` n’est pas nécessaire, la cmdlet assemble elle-même les lignes.

#### Exemple 5

```powershell
Get-ChildItem -Path .\export\*.eml |
    Get-MailHeaderAnalysis |
    Select-Object Source, Subject, Spf, Dkim, Dmarc, AuthTrust, HopCount,
        @{ Name = 'Findings'; Expression = { ($_.Findings.Code | Sort-Object -Unique) -join ',' } } |
    Export-Csv -Path .\auswertung.csv -NoTypeInformation -Encoding UTF8
```

Analyse tous les messages d’un dossier et écrit un fichier CSV contenant une ligne par message. La colonne calculée regroupe les codes de constat.

| Option | Effet |
|---|---|
| `Get-ChildItem -Path .\export\*.eml` | Fournit les objets fichier ; `-Path` reprend leur propriété `FullName` |
| `Select-Object … @{ Name; Expression }` | Colonne calculée qui regroupe tous les codes de constat dans une chaîne |
| `Export-Csv -NoTypeInformation -Encoding UTF8` | CSV sans ligne d’en-tête de type, UTF-8 pour les caractères accentués dans les lignes d’objet |

#### Exemple 6

```powershell
Get-ChildItem .\export\*.eml |
    Get-MailHeaderAnalysis |
    Where-Object AuthTrust -eq 'Unmatched' |
    Select-Object Source, AuthServId, DeliveredBy
```

Liste les messages dont la ligne `Authentication-Results` ne provient pas du système de remise. Pour les analyses de phishing, c’est un premier filtre rapide.

#### Exemple 7

```powershell
$a = Get-MailHeaderAnalysis -Path .\nachricht.eml
$a.Hops[$a.SlowestHopIndex - 1] | Format-List Index, FromHost, ByHost, Delay
```

Affiche la station ayant le plus grand délai. `SlowestHopIndex` est basé sur 1, le tableau `Hops` sur 0.

#### Exemple 8

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml | ConvertTo-Json -Depth 6 | Set-Content .\analyse.json
```

Écrit l’analyse complète au format JSON. `-Depth 6` est nécessaire, car `Hops`, `DkimSignatures` et `Exchange` sont des objets imbriqués ; la valeur par défaut de 2 ne les afficherait que sous forme de noms de type.

### Paramètres

#### -Header

L’en-tête sous forme de texte. Le paramètre accepte une seule chaîne contenant tout l’en-tête ou plusieurs chaînes ; les entrées du pipeline sont collectées puis assemblées en un en-tête à la fin, c’est pourquoi `Get-Content datei | Get-MailHeaderAnalysis` fonctionne sans `-Raw`. Pour analyser séparément plusieurs en-têtes, utilisez `-Path` avec plusieurs fichiers.

Alias : `Text`, `Raw`, `InputObject`

| Propriété du paramètre | Valeur |
|---|---|
| Type | `String[]` |
| Valeur par défaut | Aucune |
| Caractères génériques pris en charge | Non |

| Jeu de paramètres Text | Valeur |
|---|---|
| Position | 0 |
| Obligatoire | Oui |
| Valeur depuis le pipeline | Oui |
| Valeur depuis le pipeline par nom de propriété | Non |

#### -Path

Chemin vers un fichier contenant l’en-tête ou un message `.eml` complet. Les chemins relatifs sont résolus par rapport au répertoire actuel. Le fichier est lu avec `[System.IO.File]::ReadAllText` : une marque d’ordre des octets est prise en compte ; sans BOM, UTF-8 est utilisé. Chaque fichier produit un objet de résultat distinct ; les fichiers manquants génèrent une erreur non bloquante.

Alias : `FullName`, `PSPath`, `LiteralPath`

| Propriété du paramètre | Valeur |
|---|---|
| Type | `String[]` |
| Valeur par défaut | Aucune |
| Caractères génériques pris en charge | Non |

| Jeu de paramètres Path | Valeur |
|---|---|
| Position | Nommée |
| Obligatoire | Oui |
| Valeur depuis le pipeline | Non |
| Valeur depuis le pipeline par nom de propriété | Oui |

#### -FromClipboard

Lit l’en-tête depuis le presse-papiers avec `Get-Clipboard -Raw`. Le paramètre n’est disponible que sous Windows ; sous Linux et macOS, la cmdlet s’arrête avec un message d’erreur, de même si le presse-papiers est vide.

| Propriété du paramètre | Valeur |
|---|---|
| Type | `SwitchParameter` |
| Valeur par défaut | `False` |
| Caractères génériques pris en charge | Non |

| Jeu de paramètres Clipboard | Valeur |
|---|---|
| Position | Nommée |
| Obligatoire | Oui |
| Valeur depuis le pipeline | Non |
| Valeur depuis le pipeline par nom de propriété | Non |

### Entrées

`System.String` : lignes d’en-tête ou l’en-tête complet, vers `-Header`.

`System.IO.FileInfo` : objets fichier de `Get-ChildItem`, dont la propriété `FullName` est liée à `-Path`.

### Sorties

`MailHeaderAnalyzer.Analysis` : un objet par en-tête analysé. Les propriétés sont décrites dans la section [Objet de sortie](#ausgabeobjekt).

### Remarques

Le texte d’analyse (explications dans `Findings`, `CompAuthReasonMeaning` et les champs de signification) est en anglais afin de pouvoir être repris sans modification dans des tickets internationaux. La sortie de l’affichage standard peut être affichée intégralement avec `Format-List *`.

## ConvertTo-MailHeaderReport

Génère un rapport au format Markdown ou texte à partir d’un objet d’analyse.

### Syntaxe

```powershell
ConvertTo-MailHeaderReport
    [-Analysis] <Object>
    [-Format <String>]
    [<CommonParameters>]
```

### Description

Le rapport comprend l’objet, l’expéditeur, la date et le Message-ID, les résultats d’authentification avec indication de provenance et alignement, tous les constats, la chaîne de distribution, la classification Exchange et les valeurs des filtres antispam. Les caractères de contrôle Unicode de direction d’écriture restent visibles dans le rapport sous la forme `<U+...>`, afin qu’ils ne soient pas transmis à un système de tickets par le rapport. La dernière ligne indique la version du module.

### Exemples

#### Exemple 1

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml | ConvertTo-MailHeaderReport | Set-Clipboard
```

Génère un rapport Markdown et le place dans le presse-papiers.

```markdown
# Email header analysis

- Subject: Service-Report für März
- From: Beispiel Newsletter <news@example.org>
- Date: 2026-08-03T09:14:27Z
- Message-ID: <20260803091427.4Wq2Wx5RbTz@mail.example.org>

## Authentication

- spf: pass
- dkim: pass
- dmarc: pass
- compauth: pass (reason=100: passed authentication)
- Results without authserv-id (Microsoft 365 style)
- DMARC alignment: SPF Strict, DKIM Strict

## Delivery chain (total: 42 s)

| # | From | By | Protocol | TLS | Time (UTC) | Delay |
|---|---|---|---|---|---|---|
| 1 | client.example.net (198.51.100.34) | mail.example.org | ESMTPSA | - | 2026-08-03T09:14:28Z | - |
| 2 | mail.example.org (203.0.113.25) | mx.eur02.prod.protection.outlook.com | Microsoft SMTP Server | TLS 1.3 | 2026-08-03T09:15:09Z | 41 s |
| 3 | AM0EUR02FT056.eop-EUR02.prod.protection.outlook.com | ZR0P278MB0570.CHEP278.PROD.OUTLOOK.COM | Microsoft SMTP Server | TLS 1.2 | 2026-08-03T09:15:10Z | 1 s |
```

#### Exemple 2

```powershell
Get-MailHeaderAnalysis -FromClipboard | ConvertTo-MailHeaderReport -Format Text
```

Produit le rapport en texte sans balisage Markdown, par exemple pour des e-mails ou des journaux de console.

#### Exemple 3

```powershell
Import-Module MailHeaderAnalyzer
Get-MailHeaderAnalysis -Path 'C:\Temp\bounce.eml' | ConvertTo-MailHeaderReport -Format Text
```

Utilisation dans Exchange Management Shell. Si `$env:PSModulePath` y est restreint par des stratégies de groupe, chargez le module avec `Import-Module` et le chemin complet vers le fichier `.psd1`.

### Paramètres

#### -Analysis

L’objet d’analyse de `Get-MailHeaderAnalysis`. La cmdlet rejette les autres types d’objets avec une erreur de liaison.

| Propriété du paramètre | Valeur |
|---|---|
| Type | `MailHeaderAnalyzer.Analysis` |
| Valeur par défaut | Aucune |
| Caractères génériques pris en charge | Non |

| Jeu de paramètres (tous) | Valeur |
|---|---|
| Position | 0 |
| Obligatoire | Oui |
| Valeur depuis le pipeline | Oui |
| Valeur depuis le pipeline par nom de propriété | Non |

#### -Format

Le format de sortie. Valeurs valides :

- `Markdown` : titres, listes et chaîne de distribution sous forme de tableau. Valeur par défaut.
- `Text` : titres en majuscules, lignes indentées, chaîne de distribution sous forme de liste numérotée.

| Propriété du paramètre | Valeur |
|---|---|
| Type | `String` |
| Valeurs autorisées | `Markdown`, `Text` |
| Valeur par défaut | `Markdown` |
| Caractères génériques pris en charge | Non |

| Jeu de paramètres (tous) | Valeur |
|---|---|
| Position | Nommée |
| Obligatoire | Non |
| Valeur depuis le pipeline | Non |
| Valeur depuis le pipeline par nom de propriété | Non |

### Entrées

`MailHeaderAnalyzer.Analysis` : le résultat de `Get-MailHeaderAnalysis`.

### Sorties

`System.String` : le rapport, une chaîne par objet d’analyse.

## Objet de sortie

`Get-MailHeaderAnalysis` renvoie un objet de type `MailHeaderAnalyzer.Analysis` par entrée. L’affichage standard montre le résumé de l’exemple 1 ; toutes les propriétés sont accessibles via `Select-Object`, `Format-List *` ou `ConvertTo-Json`.

### MailHeaderAnalyzer.Analysis

| Propriété | Type | Contenu |
|---|---|---|
| `Source` | String | Chemin de fichier, `Clipboard` ou `Text` |
| `Subject` | String | Objet, décodé selon RFC 2047 |
| `From`, `ReplyTo`, `ReturnPath` | Objet d’adresse | `Name`, `Address`, `Domain`, `Display`; `$null` si le champ est absent |
| `Date` | DateTime (UTC) | Valeur du champ `Date` |
| `MessageId` | String | `Message-ID` |
| `MailFromDomain` | String | Domaine de l’expéditeur d’enveloppe issu de `smtp.mailfrom` de la vérification SPF, sinon de `Return-Path` |
| `Spf`, `Dkim`, `Dmarc`, `Arc`, `CompAuth` | String | Résultat selon la ligne `Authentication-Results` faisant autorité (`pass`, `fail`, `none`, `softfail` et autres) ; `$null` si non vérifié |
| `CompAuthReason`, `CompAuthReasonMeaning` | String | Code de raison de l’authentification composite de Microsoft 365 et sa signification |
| `AuthTrust` | String | `Matched`, `Unmatched`, `Absent` ou `None`, voir [AuthTrust](#authtrust-herkunft-der-prüfergebnisse) |
| `AuthServId` | String | authserv-id de la ligne de vérification faisant autorité |
| `AuthenticationResults` | Objet[] | Toutes les lignes `Authentication-Results` avec `AuthServId`, `Methods`, `Trust`, `Raw` |
| `ReceivedSpf` | Objet | La ligne `Received-SPF` avec `Result` et `Properties` |
| `SpfAlignment`, `DkimAlignment` | String | `Strict`, `Relaxed`, `None` ou `$null` |
| `Hops` | Hop[] | Chaîne de distribution dans l’ordre chronologique, voir [objet Hop](#mailheaderanalyzerhop) |
| `HopCount`, `TotalDuration`, `SlowestHopIndex`, `HasClockSkew` | Int, TimeSpan, Int, Bool | Indicateurs de la chaîne |
| `DeliveredBy` | String | Hôte `by` de la ligne `Received` la plus récente, c’est-à-dire la station de remise |
| `DkimSignatures` | Objet[] | Par signature : `Domain`, `Selector`, `Algorithm`, `Canonicalization`, `SignedHeaders`, `BodyLength`, `Timestamp`, `Expires`, `ReceiverResult`, `Tags` |
| `ArcChain`, `ArcValid` | Objet[], Bool | Instances ARC avec `Instance`, `SealDomain`, `ChainValidation`, `Methods`; `ArcValid` vaut `$null` sans chaîne ARC |
| `Exchange` | Objet | Classification hybride d’Exchange Online, voir [objet Exchange](#mailheaderanalyzerexchangeclassification); `$null` sans en-têtes correspondants |
| `Spam` | Objet | Évaluations des filtres antispam, voir [objet Spam](#mailheaderanalyzerspamassessment); `$null` sans en-têtes correspondants |
| `List` | Objet | `ListId`, `Unsubscribe`, `OneClick` (RFC 8058) ; `$null` sans en-têtes de liste |
| `Findings` | Finding[] | Anomalies avec `Severity`, `Code`, `Message`, voir [Findings](#findings) |
| `Fields` | Objet[] | Tous les champs avec `Name`, `Value` (déplié) et `Raw` |
| `HadBody` | Bool | Indique si un corps de message suivait l’en-tête |

### MailHeaderAnalyzer.Hop

Chaque entrée de `Hops` correspond à une ligne `Received`. L’ordre est chronologique, donc inverse à l’ordre dans l’en-tête.

| Propriété | Contenu |
|---|---|
| `Index` | Numéro séquentiel, 1 = dépôt |
| `FromHost`, `Helo`, `ReverseDns`, `IPAddress`, `IsPrivateIP` | Informations sur le système émetteur issues de la partie `from` ; l’IP provient des crochets dans le commentaire, le nom rDNS du commentaire qui le précède |
| `ByHost`, `Software` | Système destinataire et son logiciel (commentaire après `by`) |
| `Protocol`, `ProtocolClass` | Valeur `with` et classe selon RFC 3848 : `TlsAuthenticated` (ESMTPSA), `Tls` (ESMTPS ou indication TLS présente), `Authenticated` (ESMTPA), `Plain`, `Http`, `Mapi`, `Local` |
| `TlsVersion`, `TlsCipher` | À partir des notations de Microsoft (`version=TLS1_2, cipher=…`), Postfix (`using TLSv1.3 with cipher …`) et Exim |
| `Id`, `For`, `Via` | Autres composants de `Received` |
| `Date` | Horodatage après le point-virgule, UTC |
| `Delay` | TimeSpan jusqu’au relais précédent ; négatif en cas de décalage d’horloge |
| `Provider` | Fournisseur ou passerelle détecté à partir des noms d’hôte (Microsoft 365, Google Workspace, SEPPmail, HIN, Proofpoint, Mimecast et autres) |
| `Attested` | `$true` uniquement pour le dernier relais : seule cette ligne a été écrite par le système destinataire lui-même, toutes celles situées en dessous étaient déjà présentes dans le message |
| `Raw` | La ligne originale |

### AuthTrust : origine des résultats de vérification

Une ligne `Authentication-Results` peut être ajoutée à un message par n’importe quel expéditeur. Selon RFC 8601, section 5, seule la ligne de l’organisation destinataire fait autorité, et son authserv-id doit pouvoir être attribué à une station de la chaîne de distribution. La cmdlet compare l’authserv-id de chaque ligne aux hôtes `by` de la chaîne `Received` (même domaine ou sous-domaine, toujours à la limite du point, jamais par sous-chaîne).

| Valeur | Signification |
|---|---|
| `Matched` | L’authserv-id apparaît comme hôte `by` dans la chaîne. Seules ces lignes sont prises en compte dans `Spf`, `Dkim`, `Dmarc` dès lors qu’il en existe au moins une |
| `Unmatched` | L’authserv-id n’apparaît pas dans la chaîne. Les résultats sont affichés, mais considérés comme une affirmation non étayée ; le constat `AuthUnverified` le signale |
| `Absent` | La ligne ne contient pas d’authserv-id. Microsoft 365 écrit sa ligne de vérification sous cette forme, elle commence directement par `spf=` |
| `None` | Aucune ligne de vérification présente |

Si l’en-tête contient des lignes de vérification provenant de plusieurs sources, le constat `AuthMixedOrigins` le signale. En l’absence de ligne de vérification mais en présence d’une ligne `Received-SPF`, son résultat est repris comme `Spf` et marqué `ReceivedSpfOnly`.

### Alignement DMARC

`SpfAlignment` compare le domaine de l’expéditeur d’enveloppe au domaine `From`, tandis que `DkimAlignment` compare le domaine `d=` de la signature vérifiée au domaine `From`. `Strict` signifie un domaine identique, `Relaxed` le même domaine organisationnel et `None` aucune correspondance. Le domaine organisationnel est déterminé de manière heuristique : les deux derniers labels, ou les trois derniers pour les terminaisons composées connues telles que `co.uk` ou `com.au`. Une liste complète des suffixes publics n’est pas incluse.

### MailHeaderAnalyzer.ExchangeClassification

Si l’en-tête contient les champs `X-MS-Exchange-Organization-*`, `X-OriginatorOrg` ou `X-MS-Exchange-CrossTenant-*`, la cmdlet renseigne la propriété `Exchange`. Les significations suivent l’article « Demystifying hybrid mail flow » de l’équipe Exchange ; le contexte est expliqué dans l’article [En-têtes hybrides Exchange : interne ou externe ?](/blog/exchange-hybrid-header-intern-extern).

| Propriété | Contenu |
|---|---|
| `Directionality`, `DirectionalityMeaning` | `Originating` ou `Incoming` avec explication |
| `AuthAs`, `AuthAsMeaning` | `Internal` ou `Anonymous` avec les conséquences pour le filtrage EOP |
| `AuthSource` | Serveur ayant effectué la classification |
| `AuthMechanism`, `AuthMechanismMeaning` | Code de mécanisme. Seule la valeur 10 (Externally Secured) est documentée publiquement ; pour tous les autres codes, le module précise expressément que Microsoft ne les documente pas |
| `OriginatorOrg` | Domaine par défaut du tenant expéditeur, caractéristique de tenant infalsifiable à la réception depuis Microsoft 365 |
| `CrossTenantAuthAs`, `CrossTenantAuthSource`, `CrossTenantId` | Classification et ID de tenant à la frontière du tenant |
| `CrossTenantFromEntity`, `CrossTenantFromMeaning` | `Internet`, `Hosted` ou `HybridOnPrem` avec explication |
| `WrongTenantAttribution` | Valeur de `X-MS-Exchange-CrossTenant-OriginalAttributedTenantConnectingIp` lorsque le message a été attribué à un tenant tiers |
| `OrganizationHeadersPreserved`, `CrossPremisesHeadersFiltered` | Marqueurs des en-têtes d’organisation reçus ou supprimés du connecteur d’envoi |

### MailHeaderAnalyzer.SpamAssessment

La propriété `Spam` regroupe les évaluations des filtres connus. Les valeurs proviennent de systèmes tiers et sont décodées, mais non évaluées.

| Propriété | Source | Contenu |
|---|---|---|
| `Scl`, `SclMeaning` | `X-Forefront-Antispam-Report`, `X-Microsoft-Antispam` | Spam Confidence Level avec signification (-1 fiable, 0/1 non spam, 5/6 suspicion de spam, 9 très probablement du spam) |
| `Bcl` | `X-Microsoft-Antispam` | Bulk Complaint Level de 0 à 9 |
| `Category`, `CategoryMeaning` | `CAT:` | Classification telle que `SPM`, `PHSH`, `HPHSH`, `MALW`, `SPOOF`, `DIMP`, `UIMP`, `BULK` |
| `SpamFilterVerdict`, `SpamFilterMeaning` | `SFV:` | Résultat du filtre tel que `NSPM`, `SPM`, `SKA` (liste d’autorisation), `SKI` (intra-organisationnel) |
| `IPVerdict`, `IPVerdictMeaning` | `IPV:` | `CAL` (IP dans la liste d’autorisation de connexion) ou `NLI` (aucune réputation) |
| `Direction`, `DirectionMeaning` | `DIR:` | `INB`, `OUT`, `INT` |
| `ConnectingIP`, `Country` | `CIP:`, `CTRY:` | IP émettrice et pays d’origine |
| `Forefront` | `X-Forefront-Antispam-Report` | Toutes les paires clé-valeur du champ |
| `SpamAssassinScore`, `SpamAssassinTests` | `X-Spam-Status` | Score et tests déclenchés |
| `RspamdSymbols` | `X-Spamd-Result`, `X-Spam-Report` | Symboles avec score |

La liste complète des codes de raison `compauth` figure dans l’article [Microsoft 365 compauth : codes de raison](/blog/microsoft-365-compauth-reason-codes).

### Findings

La cmdlet fournit les anomalies sous forme d’objets dans `Findings`, chacun avec `Severity` (`Info`, `Warning`, `Fail`), un `Code` stable pour les filtres et scripts, ainsi qu’une explication dans `Message`.

| Code | Gravité | Signification |
|---|---|---|
| `DuplicateField` | Warning | Un champ que RFC 5322 limite à une seule instance (`From`, `Subject`, `Date`, `Message-ID` et d’autres) est présent plusieurs fois. Les clients de messagerie et filtres peuvent sélectionner des instances différentes ; c’est un motif connu de falsification |
| `BidiControls` | Warning | Caractères de contrôle Unicode de direction d’écriture dans un champ. Ils inversent le sens de lecture : `fdp.exe` apparaît alors comme `exe.pdf`. Le module les affiche comme `<U+202E>` |
| `HopOverflow` | Warning | Plus de 200 lignes `Received` ; les lignes excédentaires n’ont pas été analysées |
| `AuthUnverified` | Warning | Les résultats de vérification comportent un authserv-id absent de la chaîne de distribution |
| `AuthMixedOrigins` | Warning | Lignes de vérification provenant de plusieurs sources présentes |
| `ReceivedSpfForeign` | Warning | Le `receiver=` de la ligne `Received-SPF` n’apparaît pas dans la chaîne |
| `ReceivedSpfOnly` | Info | Le résultat SPF provient uniquement de `Received-SPF`, et non d’une ligne de vérification |
| `NoAuthResults` | Info | Aucun résultat de vérification dans l’en-tête |
| `DmarcFail` | Fail | DMARC a échoué selon le serveur destinataire |
| `SpfNotPass` | Warning | Résultat SPF `fail`, `softfail`, `permerror` ou `temperror` |
| `DkimNotPass` | Warning | Résultat DKIM `fail`, `permerror` ou `temperror` sans témoin ARC |
| `DkimBrokenAfterForward` | Info | DKIM a échoué chez le destinataire, mais un sceau ARC du même domaine atteste une signature antérieurement valide : typique des redirections et listes de diffusion |
| `DkimWeakHash` | Warning | Signature avec `rsa-sha1` (RFC 8301 considère SHA-1 comme obsolète) |
| `DkimBodyLength` | Warning | La balise `l=` limite la longueur signée du corps du message ; le contenu ajouté n’est pas couvert |
| `DkimExpired` | Warning | L’instant `x=` est situé dans le passé |
| `DkimFromUnsigned` | Warning | Le champ `From` n’est pas contenu dans `h=`, bien que RFC 6376 l’exige |
| `ClockSkew` | Info | Un relais porte un horodatage antérieur à celui de son prédécesseur ; les délais ne sont que des approximations |
| `ReplyToMismatch` | Info | Le domaine `Reply-To` diffère du domaine `From` ; courant pour les newsletters, mais également un motif de phishing |
| `SpfNotAligned` | Info | Le domaine de l’expéditeur d’enveloppe et le domaine `From` appartiennent à des organisations différentes ; SPF ne contribue alors pas à DMARC |
| `ExchangeWrongTenant` | Warning | Le message a été attribué à un tenant tiers ; une cause classique est un connecteur entrant d’un autre tenant utilisant le même certificat ou les mêmes adresses IP |
| `ExchangeHeadersFiltered` | Warning | Le connecteur d’envoi a supprimé les en-têtes cross-premises (KB3212872) |
| `ExchangeExternallySecured` | Warning | AuthMechanism 10 : réception via un connecteur de réception avec « Externally Secured », filtrage EOP ignoré |
| `SpamCategory` | Warning | Microsoft a attribué une catégorie autre que `NONE` |
| `SpamConfidence` | Warning | SCL 5 ou supérieur |

## Fonctionnement et limites

Le module lit ce qui figure dans l’en-tête et en déduit ce qui peut être établi sans requête externe. Cela implique certaines limites :

- **Aucune vérification cryptographique.** Les signatures DKIM ne sont pas recalculées et les enregistrements DNS ne sont pas interrogés. `Spf`, `Dkim` et `Dmarc` correspondent toujours au verdict du serveur destinataire, complété par la vérification que ce verdict provient bien de lui.
- **Seule la dernière ligne `Received` est établie.** Toutes les lignes situées en dessous ont été fournies par l’expéditeur, qui peut les façonner à sa guise. `Attested` marque cette différence ; les délais des relais antérieurs reposent sur les indications de ces lignes.
- **Domaines organisationnels déterminés heuristiquement.** Pour l’alignement Relaxed, le module utilise une courte liste de terminaisons composées, et non une liste complète des suffixes publics.
- **AuthMechanism seulement partiellement documenté.** Hormis la valeur 10, Microsoft n’a pas publié les codes ; le module n’invente aucune signification.
- **Jeux de caractères.** Les valeurs RFC 2047 sont décodées avec les encodages que .NET connaît sur le système concerné. Les jeux de caractères inconnus restent inchangés.

## Protection des données

Un en-tête complet contient des noms d’hôte internes, des adresses IP, des expéditeurs, des destinataires et l’objet. L’article [Analyser des en-têtes d’e-mails sans téléverser le message](/blog/e-mail-header-analysieren-ohne-upload) explique pourquoi ces informations n’ont pas leur place dans un outil en ligne. La même promesse s’applique au module qu’à la version navigateur : aucun accès réseau. La suite de tests du module contient un test qui échoue dès qu’une cmdlet réseau ou DNS apparaît dans le code source.

## Code source et versions

Le code source est disponible sous licence MIT sur [GitHub](https://github.com/pfstr/MailHeaderAnalyzer). La logique d’analyse est un portage de la bibliothèque également utilisée par l’analyseur d’en-têtes de ce site web ; les deux partagent les cas de test. L’intégration continue vérifie chaque modification avec PSScriptAnalyzer et Pester sous Windows PowerShell 5.1, PowerShell 7 sur Windows, Ubuntu et macOS. Les publications dans la PowerShell Gallery sont effectuées automatiquement à partir de balises versionnées ; les modifications de chaque version sont indiquées dans le [changelog](https://github.com/pfstr/MailHeaderAnalyzer/blob/main/CHANGELOG.md).

Vous pouvez soumettre les erreurs et demandes d’amélioration comme [issue sur GitHub](https://github.com/pfstr/MailHeaderAnalyzer/issues). Veuillez anonymiser les en-têtes de test avant de signaler une erreur ; les cas de test inclus utilisent exclusivement des domaines d’exemple conformes à RFC 2606 et des adresses conformes à RFC 5737.

## Sources

1.  [MailHeaderAnalyzer dans la PowerShell Gallery](https://www.powershellgallery.com/packages/MailHeaderAnalyzer) : page du package avec commande d’installation et historique des versions.

2.  [pfstr/MailHeaderAnalyzer sur GitHub](https://github.com/pfstr/MailHeaderAnalyzer) : code source, suite de tests, changelog et issues.

3.  [Microsoft Learn : Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps) : modèle de la structure de cette référence (syntaxe, description, exemples, propriétés des paramètres).

4.  [RFC 8601 : Message Header Field for Indicating Message Authentication Status](https://www.rfc-editor.org/rfc/rfc8601) : structure de `Authentication-Results` et règle selon laquelle seule la ligne de l’organisation destinataire fait autorité (section 5).

5.  [RFC 5321 : Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321) : structure des lignes `Received` et recommandation d’une limite supérieure comme protection contre les boucles (section 6.3).

6.  [RFC 3848 : ESMTP and LMTP Transmission Types Registration](https://www.rfc-editor.org/rfc/rfc3848) : valeurs `with` `ESMTPS`, `ESMTPA` et `ESMTPSA`.

7.  [RFC 6376 : DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376) : balises de `DKIM-Signature`, obligation de signer `From`.

8.  [RFC 8301 : Cryptographic Algorithm and Key Usage Update to DKIM](https://www.rfc-editor.org/rfc/rfc8301) : classification de `rsa-sha1` comme obsolète.

9.  [RFC 7489 : Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489) : alignement Strict et Relaxed.

10.  [RFC 8617 : The Authenticated Received Chain (ARC) Protocol](https://www.rfc-editor.org/rfc/rfc8617) : `ARC-Seal`, `ARC-Authentication-Results` et vérification de chaîne `cv=`.

11.  [Microsoft Learn : Anti-spam message headers in Microsoft 365](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo) : signification de SCL, BCL, CAT, SFV, IPV et des codes de raison `compauth`.

12.  [Blog de l’équipe Exchange : Demystifying hybrid mail flow](https://techcommunity.microsoft.com/blog/exchange/demystifying-hybrid-mail-flow-when-is-a-message-internal/1420838) : origine des significations de MessageDirectionality, AuthAs et AuthMechanism.

13.  [Analyseur d’en-têtes sur rafaelpfister.ch](/tools/header-analyzer) : version navigateur utilisant la même logique d’analyse.
