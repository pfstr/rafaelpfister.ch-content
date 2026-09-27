---
title: "MailHeaderAnalyzer : analyser les en-têtes d’e-mails dans PowerShell, sans accès réseau"
navTitle: "MailHeaderAnalyzer (PowerShell)"
description: "Référence du module PowerShell MailHeaderAnalyzer : aperçu des paramètres, syntaxe, description, exemples et propriétés des paramètres de Get-MailHeaderAnalysis et ConvertTo-MailHeaderReport, ainsi que l’objet de sortie avec chaîne de remise, résultats d’authentification, classification Exchange Online et tous les codes de constat."
date: "2026-09-24"
kategorie: "SMTP et flux de messagerie"
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
translationSourceHash: 41c3cb7c83800c0cc30f797ad6b2d9ec480d6c32bc054d2e01bd037b3dcd6c9b
translationModel: gpt-5.6-terra
translatedAt: 2026-09-26T08:56:50.337Z
translationReview: automatic
url: https://rafaelpfister.ch/fr/blog/mailheaderanalyzer-analyser-les-en-tetes-d-e-mails-dans-powershell-sans-acces-reseau
---

# MailHeaderAnalyzer : analyser les en-têtes d’e-mails dans PowerShell, sans accès réseau

MailHeaderAnalyzer est un module PowerShell comprenant deux cmdlets. `Get-MailHeaderAnalysis` analyse l’en-tête d’un e-mail : chaîne de remise avec délais et indications TLS, résultats SPF, DKIM, DMARC et ARC, y compris le contrôle de provenance par rapport à l’authserv-id de votre passerelle, alignement DMARC, classification hybride d’Exchange Online, évaluations de Microsoft Defender, SpamAssassin et Rspamd, ainsi que des anomalies telles que des lignes `From` en double ou des caractères de contrôle Unicode. `ConvertTo-MailHeaderReport` génère à partir de cela un rapport pour les tickets. Le module fonctionne entièrement hors ligne : aucune requête DNS, aucune connexion HTTP. Il s’agit de la version en ligne de commande de [l’analyseur d’en-têtes de ce site](/tools/header-analyzer) et utilise la même logique d’analyse.

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
| `Get-MailHeaderAnalysis` | `-TrustedAuthServId` | `String[]` | Non | Non | authserv-id(s) de votre passerelle entrante ; seules les lignes de contrôle portant l’un de ces identifiants sont considérées comme établies |
| `ConvertTo-MailHeaderReport` | `-Analysis` | `MailHeaderAnalyzer.Analysis` | Oui | Oui, par valeur | L’objet résultat de `Get-MailHeaderAnalysis` |
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

Sur un système sans accès Internet, téléchargez le module sur un autre ordinateur avec `Save-Module -Name MailHeaderAnalyzer -Path C:\Temp` et copiez le dossier `MailHeaderAnalyzer` dans un répertoire de `$env:PSModulePath`, par exemple `%USERPROFILE%\Documents\WindowsPowerShell\Modules` (Windows PowerShell 5.1) ou `%USERPROFILE%\Documents\PowerShell\Modules` (PowerShell 7). `Update-Module -Name MailHeaderAnalyzer` récupère les mises à jour et `Get-Module -Name MailHeaderAnalyzer -ListAvailable` affiche la version installée.

## Get-MailHeaderAnalysis

Analyse l’en-tête d’un e-mail et renvoie un objet d’analyse.

### Syntaxe

#### Text (par défaut)

```powershell
Get-MailHeaderAnalysis
    [-Header] <String[]>
    [-TrustedAuthServId <String[]>]
    [<CommonParameters>]
```

#### Path

```powershell
Get-MailHeaderAnalysis
    -Path <String[]>
    [-TrustedAuthServId <String[]>]
    [<CommonParameters>]
```

#### Clipboard

```powershell
Get-MailHeaderAnalysis
    -FromClipboard
    [-TrustedAuthServId <String[]>]
    [<CommonParameters>]
```

### Description

La cmdlet décompose l’en-tête brut en champs, déplie les replis RFC 5322 et décode les valeurs RFC 2047 dans l’objet et les adresses. À partir des lignes `Received`, elle constitue la chaîne de remise dans l’ordre chronologique, calcule le délai à chaque station et lit la version TLS, le chiffrement et la classe de protocole selon RFC 3848. À partir de `Authentication-Results`, `Received-SPF`, `DKIM-Signature` et de la chaîne ARC, elle détermine les résultats d’authentification et attribue une provenance à chaque ligne de contrôle (RFC 8601, section 5) : établie si son authserv-id figure dans `-TrustedAuthServId`, sinon seulement plausible ou non établie ; voir [AuthTrust](#authtrust-herkunft-der-prüfergebnisse). S’y ajoutent l’alignement DMARC, la classification hybride d’Exchange Online, les évaluations des filtres antispam et une liste d’anomalies.

La cmdlet n’effectue aucune requête DNS et n’ouvre aucune connexion réseau. `Spf`, `Dkim`, `Dmarc` et `Arc` représentent donc toujours le jugement du serveur destinataire. Les signatures DKIM et ARC ne sont pas recalculées cryptographiquement ; pour la chaîne ARC, la cmdlet vérifie uniquement la structure (`ArcStructure`).

Les entrées sont lues avec tolérance : une ligne vide termine l’en-tête et le corps du message qui suit est ignoré. Les lignes sans nom de champ et sans espace initial, telles qu’elles résultent d’une copie depuis des boîtes de dialogue de clients, appartiennent au champ précédent. Une ligne de séparation mbox `From ...` avant le premier champ est ignorée et un Byte Order Mark est supprimé. Au maximum 200 lignes `Received` sont analysées, comptées depuis la remise.

### Exemples

#### Exemple 1

```powershell
Get-MailHeaderAnalysis -FromClipboard
```

Analyse l’en-tête présent dans le presse-papiers. Dans Outlook pour Windows, vous trouverez l’en-tête sous Fichier, Propriétés, En-têtes Internet ; dans Outlook sur le Web, sous « Afficher les détails du message » dans les options du message.

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

Analyse un message enregistré. Le fichier peut ne contenir que l’en-tête ou le message complet ; le corps du message est ignoré.

#### Exemple 3

```powershell
(Get-MailHeaderAnalysis -Path .\nachricht.eml).Hops
```

Affiche la chaîne de remise sous forme de tableau, avec le premier saut en premier.

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
| `Get-ChildItem -Path .\export\*.eml` | Fournit les objets fichier ; `-Path` utilise leur propriété `FullName` |
| `Select-Object … @{ Name; Expression }` | Colonne calculée qui regroupe tous les codes de constat dans une chaîne |
| `Export-Csv -NoTypeInformation -Encoding UTF8` | CSV sans ligne d’en-tête de type, UTF-8 pour les caractères accentués dans les lignes d’objet |

#### Exemple 6

```powershell
Get-ChildItem .\export\*.eml |
    Get-MailHeaderAnalysis -TrustedAuthServId 'mx.example.org' |
    Where-Object AuthTrust -ne 'Trusted' |
    Select-Object Source, AuthTrust, AuthServId, DeliveredBy
```

Répertorie les messages dont les résultats de contrôle ne proviennent pas de votre propre passerelle entrante `mx.example.org`. Lors d’analyses de phishing, il s’agit d’un premier filtre. Sans `-TrustedAuthServId`, il n’est possible de filtrer que sur `Unmatched` ; une falsification qui comporte en plus une ligne `Received` correspondante apparaît alors comme `Matched` et échappe au filtre.

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-TrustedAuthServId 'mx.example.org'` | authserv-id que votre passerelle écrit dans `Authentication-Results` ; comparaison exacte, sans sous-domaines |
| `Where-Object AuthTrust -ne 'Trusted'` | Conserve tous les messages dont la ligne de contrôle déterminante ne porte pas d’authserv-id fiable |
| `Select-Object Source, AuthTrust, AuthServId, DeliveredBy` | Fichier, niveau de provenance, authserv-id de la ligne de contrôle et station de remise |

</details>

#### Exemple 7

```powershell
$a = Get-MailHeaderAnalysis -Path .\nachricht.eml
$a.Hops[$a.SlowestHopIndex - 1] | Format-List Index, FromHost, ByHost, Delay
```

Affiche la station présentant le délai le plus important. `SlowestHopIndex` est basé sur 1, le tableau `Hops` est basé sur 0.

#### Exemple 8

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml | ConvertTo-Json -Depth 6 | Set-Content .\analyse.json
```

Écrit l’analyse complète au format JSON. `-Depth 6` est nécessaire, car `Hops`, `DkimSignatures` et `Exchange` sont des objets imbriqués ; la valeur par défaut de 2 ne les afficherait que sous forme de noms de type.

### Paramètres

#### -Header

L’en-tête sous forme de texte. Le paramètre accepte une chaîne unique avec l’en-tête entier ou plusieurs chaînes ; les entrées du pipeline sont collectées et assemblées à la fin en un en-tête, ce qui permet à `Get-Content datei | Get-MailHeaderAnalysis` de fonctionner sans `-Raw`. Pour analyser plusieurs en-têtes séparément, utilisez `-Path` avec plusieurs fichiers.

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

Chemin vers un fichier contenant l’en-tête ou un message `.eml` complet. Les chemins relatifs sont résolus par rapport au répertoire actuel. Le fichier est lu avec `[System.IO.File]::ReadAllText` : un Byte Order Mark est pris en compte, sinon UTF-8 est utilisé. Chaque fichier produit son propre objet résultat ; les fichiers absents génèrent une erreur non bloquante.

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

Lit l’en-tête du presse-papiers avec `Get-Clipboard -Raw`. Le paramètre n’est disponible que sous Windows ; sous Linux et macOS, la cmdlet s’arrête avec un message d’erreur, de même si le presse-papiers est vide.

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

#### -TrustedAuthServId

L’authserv-id ou les authserv-ids que votre passerelle entrante écrit dans `Authentication-Results`, par exemple `mx.example.org`. La cmdlet compare exactement, sans tenir compte de la casse et sans sous-domaines. Les lignes de contrôle portant l’un de ces identifiants reçoivent `AuthTrust = Trusted`, et seules celles-ci sont alors prises en compte pour `Spf`, `Dkim`, `Dmarc` et `Arc`. Il en va de même pour le `receiver=` d’une ligne `Received-SPF`.

Le résultat n’est fiable que dans la mesure où la passerelle l’est : elle doit supprimer les lignes `Authentication-Results` entrantes qui revendiquent son propre authserv-id (RFC 8601, section 5). Il n’est pas possible de déterminer depuis un en-tête si elle le fait. Si deux lignes portant un identifiant fiable se contredisent, la cmdlet signale `AuthTrustedConflict`. Microsoft 365 n’écrit pas d’authserv-id dans sa ligne de contrôle ; si Exchange Online reçoit votre courrier, omettez ce paramètre.

Pour une session entière, la valeur peut être définie par défaut, par exemple dans le profil PowerShell :

```powershell
$PSDefaultParameterValues['Get-MailHeaderAnalysis:TrustedAuthServId'] = 'mx.example.org'
```

| Propriété du paramètre | Valeur |
|---|---|
| Type | `String[]` |
| Valeur par défaut | Aucune |
| Caractères génériques pris en charge | Non |

| Jeu de paramètres (tous) | Valeur |
|---|---|
| Position | Nommée |
| Obligatoire | Non |
| Valeur depuis le pipeline | Non |
| Valeur depuis le pipeline par nom de propriété | Non |

### Entrées

`System.String` : lignes d’en-tête ou l’en-tête entier, vers `-Header`.

`System.IO.FileInfo` : objets fichier de `Get-ChildItem`, dont `FullName` est lié à `-Path`.

### Sorties

`MailHeaderAnalyzer.Analysis` : un objet par en-tête analysé. Les propriétés sont décrites dans la section [Objet de sortie](#ausgabeobjekt).

### Remarques

Le texte d’analyse (explications dans `Findings`, `CompAuthReasonMeaning` et les champs de signification) est en anglais afin de pouvoir être repris tel quel dans des tickets internationaux. La sortie de l’affichage par défaut peut être affichée intégralement avec `Format-List *`.

## ConvertTo-MailHeaderReport

Génère à partir d’un objet d’analyse un rapport en Markdown ou en texte.

### Syntaxe

```powershell
ConvertTo-MailHeaderReport
    [-Analysis] <Object>
    [-Format <String>]
    [<CommonParameters>]
```

### Description

Le rapport comprend l’objet, l’expéditeur, la date et le Message-ID, les résultats d’authentification avec indication de provenance et alignement, tous les constats, la chaîne de remise, la classification Exchange et les valeurs des filtres antispam. Les caractères de contrôle Unicode servant au sens d’écriture restent visibles dans le rapport sous la forme `<U+...>`, afin qu’ils ne soient pas transmis à un système de tickets via le rapport. La dernière ligne indique la version du module.

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

Produit le rapport en texte brut sans balisage Markdown, par exemple pour des e-mails ou des journaux de console.

#### Exemple 3

```powershell
Import-Module MailHeaderAnalyzer
Get-MailHeaderAnalysis -Path 'C:\Temp\bounce.eml' | ConvertTo-MailHeaderReport -Format Text
```

Utilisation dans Exchange Management Shell. Si `$env:PSModulePath` y est limité par des stratégies de groupe, chargez le module avec `Import-Module` et le chemin complet vers le fichier `.psd1`.

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

- `Markdown` : titres, listes à puces et chaîne de remise sous forme de tableau. Valeur par défaut.
- `Text` : titres en majuscules, lignes en retrait, chaîne de remise sous forme de liste numérotée.

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

`Get-MailHeaderAnalysis` renvoie un objet de type `MailHeaderAnalyzer.Analysis` pour chaque entrée. L’affichage par défaut montre le résumé de l’exemple 1 ; toutes les propriétés sont accessibles via `Select-Object`, `Format-List *` ou `ConvertTo-Json`.

### MailHeaderAnalyzer.Analysis

| Propriété | Type | Contenu |
|---|---|---|
| `Source` | String | Chemin de fichier, `Clipboard` ou `Text` |
| `Subject` | String | Objet, décodé RFC 2047 |
| `From`, `ReplyTo`, `ReturnPath` | Objet adresse | `Name`, `Address`, `Domain`, `Display`; `$null` si le champ est absent |
| `Date` | DateTime (UTC) | Valeur du champ `Date` |
| `MessageId` | String | `Message-ID` |
| `MailFromDomain` | String | Domaine de l’expéditeur de l’enveloppe issu de `smtp.mailfrom` du contrôle SPF, sinon de `Return-Path` |
| `Spf`, `Dkim`, `Dmarc`, `Arc`, `CompAuth` | String | Résultat selon la ligne `Authentication-Results` déterminante (`pass`, `fail`, `none`, `softfail` et autres) ; `$null` si non contrôlé. `Arc` est le jugement du destinataire sur la chaîne ARC |
| `CompAuthReason`, `CompAuthReasonMeaning` | String | Code de raison de l’authentification composite de Microsoft 365 et sa signification |
| `AuthTrust` | String | `Trusted`, `Matched`, `Unmatched`, `Absent` ou `None`, voir [AuthTrust](#authtrust-herkunft-der-prüfergebnisse) |
| `AuthServId` | String | authserv-id de la ligne de contrôle déterminante |
| `AuthenticationResults` | Objet[] | Toutes les lignes `Authentication-Results` avec `AuthServId`, `Methods`, `Trust`, `Raw` |
| `ReceivedSpf` | Objet | La ligne `Received-SPF` avec `Result` et `Properties` |
| `SpfAlignment`, `DkimAlignment` | String | `Strict`, `Relaxed`, `None` ou `$null` |
| `Hops` | Hop[] | Chaîne de remise dans l’ordre chronologique, voir [objet Hop](#mailheaderanalyzerhop) |
| `HopCount`, `TotalDuration`, `SlowestHopIndex`, `HasClockSkew` | Int, TimeSpan, Int, Bool | Indicateurs de la chaîne |
| `DeliveredBy` | String | Hôte `by` de la ligne `Received` la plus récente, donc la station de remise |
| `DkimSignatures` | Objet[] | Par signature : `Domain`, `Selector`, `Algorithm`, `Canonicalization`, `SignedHeaders`, `BodyLength`, `Timestamp`, `Expires`, `ReceiverResult`, `Tags` |
| `ArcChain` | Objet[] | Instances ARC avec `Instance`, `SealDomain`, `ChainValidation`, `Methods` |
| `ArcStructure`, `ArcStructureIssues` | String, String[] | Structure de la chaîne ARC : `Consistent`, `Inconsistent` ou `$null` sans en-tête ARC, ainsi que les écarts constatés. Contrôle structurel uniquement, sans vérification de signature ; voir [chaîne ARC](#arc-kette-aufbau-und-urteil) |
| `Exchange` | Objet | Classification hybride d’Exchange Online, voir [objet Exchange](#mailheaderanalyzerexchangeclassification) ; `$null` sans en-têtes correspondants |
| `Spam` | Objet | Évaluations des filtres antispam, voir [objet Spam](#mailheaderanalyzerspamassessment) ; `$null` sans en-têtes correspondants |
| `List` | Objet | `ListId`, `Unsubscribe`, `OneClick` (RFC 8058) ; `$null` sans en-têtes de liste |
| `Findings` | Finding[] | Anomalies avec `Severity`, `Code`, `Message`, voir [Findings](#findings) |
| `Fields` | Objet[] | Tous les champs avec `Name`, `Value` (déplié) et `Raw` |
| `HadBody` | Bool | Indique si un corps de message suivait l’en-tête |

### MailHeaderAnalyzer.Hop

Chaque entrée de `Hops` correspond à une ligne `Received`. L’ordre est chronologique, donc inverse de l’ordre dans l’en-tête.

| Propriété | Contenu |
|---|---|
| `Index` | Numéro séquentiel, 1 = soumission |
| `FromHost`, `Helo`, `ReverseDns`, `IPAddress`, `IsPrivateIP` | Informations sur le système émetteur provenant de la partie `from` ; l’IP provient des crochets dans le commentaire, le nom rDNS du commentaire précédent |
| `ByHost`, `Software` | Système récepteur et son logiciel (commentaire après `by`) |
| `Protocol`, `ProtocolClass` | Valeur `with` et classe selon RFC 3848 : `TlsAuthenticated` (ESMTPSA), `Tls` (ESMTPS ou indication TLS présente), `Authenticated` (ESMTPA), `Plain`, `Http`, `Mapi`, `Local` |
| `TlsVersion`, `TlsCipher` | À partir des notations de Microsoft (`version=TLS1_2, cipher=…`), Postfix (`using TLSv1.3 with cipher …`) et Exim |
| `Id`, `For`, `Via` | Autres composants de `Received` |
| `Date` | Horodatage après le point-virgule, UTC |
| `Delay` | TimeSpan jusqu’au saut précédent ; négatif en cas de décalage d’horloge |
| `Provider` | Fournisseur ou passerelle détecté à partir des noms d’hôte (Microsoft 365, Google Workspace, SEPPmail, HIN, Proofpoint, Mimecast et autres) |
| `Attested` | `$true` uniquement pour le dernier saut : seule cette ligne a été écrite par le système récepteur lui-même, toutes celles en dessous figuraient déjà dans le message |
| `Raw` | Ligne originale |

### AuthTrust : provenance des résultats de contrôle

Une ligne `Authentication-Results` peut être écrite dans un message par tout expéditeur, de même qu’une ligne `Received` correspondante. Il n’est donc pas possible d’établir à partir du seul en-tête qui a écrit une ligne de contrôle. Selon RFC 8601, section 5, seule la ligne dont l’authserv-id est connue comme propre par l’organisation destinataire fait foi, et la passerelle entrante doit supprimer les lignes entrantes portant cet identifiant. La cmdlet applique cette règle avec `-TrustedAuthServId`. Sans ce paramètre, elle compare l’authserv-id uniquement avec les hôtes `by` de la chaîne `Received` (même domaine ou sous-domaine, toujours à la frontière du point, sans sous-chaîne) ; il s’agit d’un contrôle de plausibilité.

| Valeur | Signification |
|---|---|
| `Trusted` | L’authserv-id figure dans `-TrustedAuthServId`. Dès qu’une telle ligne existe, seules les lignes de ce niveau sont prises en compte pour `Spf`, `Dkim`, `Dmarc` et `Arc`. Fiable à condition que la passerelle supprime les lignes étrangères portant cet identifiant |
| `Matched` | L’authserv-id apparaît comme hôte `by` dans la chaîne. Plausible, mais sans preuve : une falsification peut fournir la ligne `Received` correspondante. Sans `-TrustedAuthServId`, le constat `AuthPlausibleOnly` le signale. Les lignes de ce niveau comptent lorsqu’aucune ligne `Trusted` n’est présente |
| `Unmatched` | L’authserv-id n’est ni fiable ni présente dans la chaîne. Les résultats sont affichés, mais considérés comme une affirmation non établie ; le constat `AuthUnverified` le signale |
| `Absent` | La ligne ne comporte pas d’authserv-id. Microsoft 365 écrit sa ligne de contrôle sous cette forme ; elle commence directement par `spf=` |
| `None` | Aucune ligne de contrôle présente |

Si l’en-tête contient des lignes de contrôle de plusieurs provenances, le constat `AuthMixedOrigins` le signale. S’il n’y a pas de ligne de contrôle, mais qu’une ligne `Received-SPF` est présente, son résultat est repris comme `Spf` et marqué `ReceivedSpfOnly`.

Avec `-TrustedAuthServId`, deux constats s’ajoutent : `AuthNotTrusted` lorsqu’aucune ligne de contrôle ne porte d’identifiant fiable, et `AuthTrustedConflict` lorsque deux lignes de ce type signalent des résultats différents pour `spf`, `dmarc`, `arc` ou `compauth`. Le second cas signifie qu’au moins une ligne ne provient pas de la passerelle et que celle-ci ne l’a pas supprimée. Dans ce cas, la cmdlet reprend la ligne la plus haute ; il est impossible de déterminer à partir de l’en-tête laquelle des deux est authentique. DKIM est exclu de cette comparaison, car plusieurs signatures peuvent légitimement avoir des résultats différents.

### Chaîne ARC : structure et jugement

Pour ARC, la cmdlet fournit deux indications distinctes. `Arc` est le résultat `arc=` issu de la ligne de contrôle déterminante, donc le jugement du destinataire qui a vérifié les signatures de la chaîne. `ArcStructure` est le contrôle propre au module et ne concerne que la structure selon RFC 8617 : numéros d’instance continus de `i=1` à `i=n` (au plus 50), exactement un `ARC-Seal`, un `ARC-Message-Signature` et un `ARC-Authentication-Results` par instance, ainsi que `cv=none` pour l’instance 1 et `cv=pass` pour toutes les suivantes. Le module ne recalcule pas les signatures ; `Consistent` ne dit donc rien sur l’authenticité de la chaîne. Les écarts figurent dans `ArcStructureIssues` et dans le constat `ArcStructureInconsistent`.

Les indications dans `ARC-Authentication-Results` sont des affirmations de chaque relais. Le constat `DkimBrokenAfterForward` ne classe donc un échec DKIM comme conséquence d’un relais que si le destinataire signale lui-même `arc=pass` et qu’une instance antérieure a enregistré un DKIM-`pass` pour le même domaine.

### Alignement DMARC

`SpfAlignment` compare le domaine de l’expéditeur de l’enveloppe avec le domaine `From`, `DkimAlignment` compare le domaine `d=` de la signature vérifiée avec le domaine `From`. `Strict` signifie domaine identique, `Relaxed` le même domaine organisationnel, `None` aucune correspondance. Le domaine organisationnel est déterminé heuristiquement : les deux derniers labels, ou les trois derniers pour les terminaisons composées connues telles que `co.uk` ou `com.au`. Une liste complète de suffixes publics n’est pas incluse.

### MailHeaderAnalyzer.ExchangeClassification

Si l’en-tête contient les champs `X-MS-Exchange-Organization-*`, `X-OriginatorOrg` ou `X-MS-Exchange-CrossTenant-*`, la cmdlet renseigne la propriété `Exchange`. Les significations suivent l’article « Demystifying hybrid mail flow » de l’équipe Exchange ; le contexte est présenté dans l’article [En-têtes hybrides Exchange : interne ou externe ?](/blog/exchange-hybrid-header-intern-extern).

| Propriété | Contenu |
|---|---|
| `Directionality`, `DirectionalityMeaning` | `Originating` ou `Incoming` avec explication |
| `AuthAs`, `AuthAsMeaning` | `Internal` ou `Anonymous` avec conséquences pour le filtrage EOP |
| `AuthSource` | Serveur ayant effectué la classification |
| `AuthMechanism`, `AuthMechanismMeaning` | Code de mécanisme. Seule la valeur 10 (Externally Secured) est documentée publiquement ; pour tous les autres codes, le module précise explicitement que Microsoft ne les documente pas |
| `OriginatorOrg` | Domaine par défaut du tenant expéditeur, caractéristique de tenant infalsifiable lors de la réception depuis Microsoft 365 |
| `CrossTenantAuthAs`, `CrossTenantAuthSource`, `CrossTenantId` | Classification et ID de tenant à la frontière du tenant |
| `CrossTenantFromEntity`, `CrossTenantFromMeaning` | `Internet`, `Hosted` ou `HybridOnPrem` avec explication |
| `WrongTenantAttribution` | Valeur de `X-MS-Exchange-CrossTenant-OriginalAttributedTenantConnectingIp` lorsque le message a été attribué à un tenant tiers |
| `OrganizationHeadersPreserved`, `CrossPremisesHeadersFiltered` | Marqueurs indiquant respectivement les en-têtes d’organisation reçus et supprimés par le connecteur d’envoi |

### MailHeaderAnalyzer.SpamAssessment

La propriété `Spam` regroupe les évaluations des filtres connus. Les valeurs proviennent de systèmes tiers et sont décodées, mais non évaluées.

| Propriété | Source | Contenu |
|---|---|---|
| `Scl`, `SclMeaning` | `X-Forefront-Antispam-Report`, `X-Microsoft-Antispam` | Spam Confidence Level avec signification (-1 fiable, 0/1 pas de spam, 5/6 suspicion de spam, 9 très probablement spam) |
| `Bcl` | `X-Microsoft-Antispam` | Bulk Complaint Level de 0 à 9 |
| `Category`, `CategoryMeaning` | `CAT:` | Classification telle que `SPM`, `PHSH`, `HPHSH`, `MALW`, `SPOOF`, `DIMP`, `UIMP`, `BULK` |
| `SpamFilterVerdict`, `SpamFilterMeaning` | `SFV:` | Résultat du filtre tel que `NSPM`, `SPM`, `SKA` (liste d’autorisation), `SKI` (intraorganisationnel) |
| `IPVerdict`, `IPVerdictMeaning` | `IPV:` | `CAL` (IP sur liste d’autorisation de connexion) ou `NLI` (pas de réputation) |
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
| `DuplicateField` | Warning | Un champ que RFC 5322 limite à une occurrence (`From`, `Subject`, `Date`, `Message-ID` et autres) apparaît plusieurs fois. Les clients de messagerie et filtres peuvent choisir des occurrences différentes ; motif connu dans les falsifications |
| `BidiControls` | Warning | Caractères de contrôle Unicode du sens d’écriture dans un champ. Ils inversent le sens de lecture ; `fdp.exe` apparaît alors comme `exe.pdf`. Le module les affiche sous la forme `<U+202E>` |
| `HopOverflow` | Warning | Plus de 200 lignes `Received` ; les lignes excédentaires n’ont pas été analysées |
| `AuthPlausibleOnly` | Info | L’authserv-id apparaît dans la chaîne de remise, mais `-TrustedAuthServId` n’a pas été spécifié : plausible, mais sans preuve |
| `AuthNotTrusted` | Warning | `-TrustedAuthServId` a été spécifié, mais aucune ligne de contrôle ne porte l’un de ces identifiants |
| `AuthTrustedConflict` | Warning | Deux lignes de contrôle portant un identifiant fiable signalent des résultats différents pour la même méthode ; la passerelle ne supprime apparemment pas les lignes étrangères |
| `AuthUnverified` | Warning | Les résultats de contrôle portent un authserv-id qui n’est ni fiable ni présent dans la chaîne de remise |
| `AuthMixedOrigins` | Warning | Présence de lignes de contrôle de plusieurs provenances |
| `ReceivedSpfForeign` | Warning | Le `receiver=` de la ligne `Received-SPF` ne figure pas dans la chaîne |
| `ReceivedSpfOnly` | Info | Le résultat SPF provient uniquement de `Received-SPF`, et non d’une ligne de contrôle |
| `NoAuthResults` | Info | Aucun résultat de contrôle dans l’en-tête |
| `DmarcFail` | Fail | DMARC a échoué selon le serveur récepteur |
| `SpfNotPass` | Warning | Résultat SPF `fail`, `softfail`, `permerror` ou `temperror` |
| `DkimNotPass` | Warning | Résultat DKIM `fail`, `permerror` ou `temperror`, sans confirmation de la chaîne ARC par le destinataire |
| `DkimBrokenAfterForward` | Info | DKIM a échoué chez le destinataire, mais celui-ci signale `arc=pass`, et une instance ARC antérieure a enregistré un DKIM-`pass` pour le même domaine : typique des relais et listes de diffusion |
| `DkimWeakHash` | Warning | Signature avec `rsa-sha1` (RFC 8301 considère SHA-1 comme obsolète) |
| `DkimBodyLength` | Warning | La balise `l=` limite la longueur signée du corps du message ; le contenu ajouté n’est pas couvert |
| `DkimExpired` | Warning | L’instant `x=` est dans le passé |
| `DkimFromUnsigned` | Warning | Le champ `From` n’est pas contenu dans `h=`, bien que RFC 6376 l’exige |
| `ArcStructureInconsistent` | Warning | Les en-têtes ARC ne constituent pas une chaîne formellement complète (lacunes, en-têtes manquants ou en double, séquence `cv=` incorrecte) ; contrôle structurel uniquement |
| `ClockSkew` | Info | Un saut porte un horodatage antérieur à celui de son prédécesseur ; les délais ne sont que des valeurs approximatives |
| `ReplyToMismatch` | Info | Le domaine `Reply-To` diffère du domaine `From` ; courant pour les newsletters, mais aussi un motif de phishing |
| `SpfNotAligned` | Info | Le domaine de l’expéditeur de l’enveloppe et le domaine `From` appartiennent à des organisations différentes ; SPF ne contribue donc pas à DMARC |
| `ExchangeWrongTenant` | Warning | Le message a été attribué à un tenant tiers ; la cause classique est un connecteur entrant d’un autre tenant utilisant le même certificat ou les mêmes adresses IP |
| `ExchangeHeadersFiltered` | Warning | Le connecteur d’envoi a supprimé les en-têtes intersites (KB3212872) |
| `ExchangeExternallySecured` | Warning | AuthMechanism 10 : entrée par un connecteur de réception avec « Externally Secured », filtrage EOP ignoré |
| `SpamCategory` | Warning | Microsoft a attribué une catégorie autre que `NONE` |
| `SpamConfidence` | Warning | SCL 5 ou supérieur |

## Fonctionnement et limites

Le module lit ce qui figure dans l’en-tête et en déduit ce qui peut être établi sans requête externe. Il en résulte plusieurs limites :

- **Aucune vérification cryptographique.** Les signatures DKIM et ARC ne sont pas recalculées et les enregistrements DNS ne sont pas interrogés. `Spf`, `Dkim`, `Dmarc` et `Arc` représentent toujours le jugement du serveur destinataire ; `ArcStructure` vérifie uniquement la structure de la chaîne.
- **Provenance établissable uniquement avec la connaissance de la passerelle.** Sans `-TrustedAuthServId`, `AuthTrust` est un contrôle de plausibilité. Avec ce paramètre, l’affirmation dépend du fait que la passerelle supprime les lignes de contrôle étrangères portant son authserv-id ; le module ne peut pas le vérifier.
- **Seule la dernière ligne `Received` est établie.** Toutes les lignes en dessous ont été fournies par l’expéditeur, qui peut les façonner arbitrairement. `Attested` marque cette différence ; les délais des sauts antérieurs reposent sur les indications de ces lignes.
- **Domaines organisationnels déterminés heuristiquement.** Pour l’alignement Relaxed, le module utilise une courte liste de terminaisons composées, et non une liste complète de suffixes publics.
- **AuthMechanism seulement partiellement documenté.** À l’exception de la valeur 10, Microsoft n’a pas publié les codes ; le module n’invente pas de significations.
- **Jeux de caractères.** Les valeurs RFC 2047 sont décodées avec les encodages connus par .NET sur le système concerné. Les jeux de caractères inconnus restent inchangés.

## Protection des données

Un en-tête complet contient des noms d’hôtes internes, des adresses IP, des expéditeurs, des destinataires et des objets. L’article [Analyser les en-têtes d’e-mails sans téléverser le message](/blog/e-mail-header-analysieren-ohne-upload) explique pourquoi ces informations ne doivent pas être envoyées vers un outil en ligne. La même promesse vaut pour le module que pour la version navigateur : aucun accès réseau. La suite de tests du module contient un test qui échoue dès qu’une cmdlet réseau ou DNS apparaît dans le code source.

## Code source et versions

Le code source est disponible sous licence MIT sur [GitHub](https://github.com/pfstr/MailHeaderAnalyzer). La logique d’analyse est un portage de la bibliothèque utilisée également par l’analyseur d’en-têtes de ce site ; les deux partagent les cas de test. L’intégration continue vérifie chaque modification avec PSScriptAnalyzer et Pester sous Windows PowerShell 5.1, PowerShell 7 sous Windows, Ubuntu et macOS. Les publications dans la PowerShell Gallery sont effectuées automatiquement depuis des tags versionnés ; les modifications de chaque version figurent dans le [changelog](https://github.com/pfstr/MailHeaderAnalyzer/blob/main/CHANGELOG.md).

La version 0.2.0 du 26 septembre 2026 a renforcé le modèle de provenance à la suite d’une remarque de revue de @saltyslugga : paramètre `-TrustedAuthServId`, niveau `Trusted`, `Matched` uniquement comme plausibilité. La propriété `ArcValid` a été supprimée et remplacée par `ArcStructure` et `ArcStructureIssues` ; les scripts qui évaluent `ArcValid` doivent être adaptés.

Les bugs et demandes d’évolution sont acceptés comme [issue sur GitHub](https://github.com/pfstr/MailHeaderAnalyzer/issues). Veuillez anonymiser les en-têtes de test avant les rapports de bugs ; les cas de test fournis utilisent exclusivement des domaines d’exemple selon RFC 2606 et des adresses selon RFC 5737.

## Sources

1.  [MailHeaderAnalyzer dans la PowerShell Gallery](https://www.powershellgallery.com/packages/MailHeaderAnalyzer) : page du package avec commande d’installation et historique des versions.

2.  [pfstr/MailHeaderAnalyzer sur GitHub](https://github.com/pfstr/MailHeaderAnalyzer) : code source, suite de tests, changelog et issues.

3.  [Microsoft Learn : Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps) : modèle de la structure de cette référence (syntaxe, description, exemples, propriétés des paramètres).

4.  [RFC 8601 : Message Header Field for Indicating Message Authentication Status](https://www.rfc-editor.org/rfc/rfc8601) : structure de `Authentication-Results` et règle selon laquelle seule la ligne de l’organisation destinataire fait foi (section 5).

5.  [RFC 5321 : Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321) : structure des lignes `Received` et recommandation d’une limite supérieure comme protection contre les boucles (section 6.3).

6.  [RFC 3848 : ESMTP and LMTP Transmission Types Registration](https://www.rfc-editor.org/rfc/rfc3848) : les valeurs `with` `ESMTPS`, `ESMTPA` et `ESMTPSA`.

7.  [RFC 6376 : DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376) : balises de `DKIM-Signature`, obligation de signer `From`.

8.  [RFC 8301 : Cryptographic Algorithm and Key Usage Update to DKIM](https://www.rfc-editor.org/rfc/rfc8301) : classification de `rsa-sha1` comme obsolète.

9.  [RFC 7489 : Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489) : alignement Strict et Relaxed.

10.  [RFC 8617 : The Authenticated Received Chain (ARC) Protocol](https://www.rfc-editor.org/rfc/rfc8617) : structure de la chaîne à partir de `ARC-Seal`, `ARC-Message-Signature` et `ARC-Authentication-Results`, numéros d’instance et valeurs `cv=`.

11.  [Microsoft Learn : Anti-spam message headers in Microsoft 365](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo) : signification de SCL, BCL, CAT, SFV, IPV et des codes de raison `compauth`.

12.  [Exchange Team Blog : Demystifying hybrid mail flow](https://techcommunity.microsoft.com/blog/exchange/demystifying-hybrid-mail-flow-when-is-a-message-internal/1420838) : origine des significations de MessageDirectionality, AuthAs et AuthMechanism.

13.  [Analyseur d’en-têtes sur rafaelpfister.ch](/tools/header-analyzer) : version navigateur utilisant la même logique d’analyse.
