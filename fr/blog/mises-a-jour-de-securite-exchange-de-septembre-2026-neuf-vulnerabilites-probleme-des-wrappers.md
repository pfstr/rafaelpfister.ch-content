---
title: "Mises à jour de sécurité Exchange de septembre 2026 : neuf vulnérabilités, problème des wrappers corrigé, v2 publiée"
navTitle: "Exchange SU 09/2026"
description: "La mise à jour de sécurité de septembre corrige neuf vulnérabilités dans Exchange SE et 2019 (huit dans Exchange 2016), dont une faille d’usurpation avec un CVSS de 9.3, et résout le problème des wrappers dans les environnements hybrides. Le 2 octobre, une v2 a suivi avec une CVE supplémentaire ; s’y ajoutent trois problèmes connus avec solutions de contournement et un SettingOverride qui doit désormais être supprimé."
date: "2026-10-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "11 min de lecture"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
  - "powershell"
slug: "mises-a-jour-de-securite-exchange-de-septembre-2026-neuf-vulnerabilites-probleme-des-wrappers"
translationId: article-53db0c02bc33f9bd
translationOf: exchange-security-updates-september-2026
translationSourceHash: fb8f756123ccec8c1983887594f47b521485f9fcf463783b480bd5eaaac8a427
translationModel: gpt-5.6-terra
translatedAt: 2026-10-25T19:38:51.382Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/mises-a-jour-de-securite-exchange-de-septembre-2026-neuf-vulnerabilites-probleme-des-wrappers
---

# Mises à jour de sécurité Exchange de septembre 2026 : neuf vulnérabilités, problème des wrappers corrigé, v2 publiée

Le 8 septembre 2026, Microsoft a publié des mises à jour de sécurité (SU) pour Exchange Server. Elles corrigent neuf vulnérabilités dans Exchange SE et Exchange 2019, et huit dans Exchange 2016. Aucune n’était connue publiquement à l’avance, aucune n’est activement exploitée selon le Security Update Guide, et Microsoft les classe toutes comme *Important* avec « Exploitation Less Likely ». La valeur CVSS la plus élevée, 9.3, dépasse toutefois nettement celle du mois précédent. Ce mois-ci est particulier pour trois raisons : la SU corrige le problème, ouvert depuis juin, des *messages wrapper* dans les boîtes aux lettres partagées ; elle introduit trois problèmes connus nouveaux ou persistants ; et le 2 octobre, Microsoft a publié une **version 2** qui corrige une vulnérabilité supplémentaire.

## Versions d’Exchange pour lesquelles la mise à jour est disponible

Les SU du 8 septembre 2026 sont disponibles pour les versions suivantes :

- **Exchange Server Subscription Edition (SE) RTM** : KB5121608, build 15.2.2562.49 ; disponible publiquement.
- **Exchange Server 2019 CU15** : KB5121609, build 15.2.1748.51 ; uniquement via le **programme ESU Period 2**.
- **Exchange Server 2019 CU14** : KB5121610, build 15.2.1544.46 ; uniquement via ESU Period 2.
- **Exchange Server 2016 CU23** : KB5121611, build 15.1.2507.73 ; uniquement via ESU Period 2.

Exchange 2016 et 2019 ne sont plus pris en charge. Selon Microsoft, seules les organisations inscrites au programme ESU Period 2 reçoivent les SU de mai à octobre 2026. D’après les articles KB, cette éligibilité s’étend jusqu’en octobre 2026. À cela s’ajoute la pression d’Exchange Online : depuis la deuxième semaine de septembre, Exchange Online limite et bloque le flux de messagerie hybride des serveurs dont le niveau est antérieur à octobre 2025 ; détails dans l’[article sur le Transport Enforcement](/blog/exchange-online-transport-enforcement-hybrid-server). Exchange Online lui-même est déjà protégé selon l’annonce ; dans les environnements hybrides, chaque serveur Exchange doit néanmoins recevoir la SU, tout comme les machines équipées des Exchange Management Tools.

Vous pouvez comparer votre niveau actuel à l’aperçu des [numéros de build Exchange](/tools/exchange-builds).

## Vue d’ensemble des vulnérabilités

| CVE | Type | CVSS |
| --- | --- | --- |
| CVE-2026-69356 | Usurpation (Cross-Site-Scripting) | 9.3 |
| CVE-2026-69641 | Élévation de privilèges | 9.1 |
| CVE-2026-69355 | Exécution de code à distance | 8.8 |
| CVE-2026-55007 | Exécution de code à distance | 8.1 |
| CVE-2026-69380 | Élévation de privilèges | 8.1 |
| CVE-2026-69378 | Déni de service | 7.5 |
| CVE-2026-69361 | Usurpation (Server-Side Request Forgery) | 6.5 |
| CVE-2026-69375 | Altération | 6.5 |
| CVE-2026-69382 | Divulgation d’informations | 5.9 |

CVE-2026-55007 ne concerne pas Exchange 2016 ; le Security Update Guide ne liste pour cette vulnérabilité qu’Exchange SE et Exchange 2019 CU14/CU15. Un détail concernant la documentation : CVE-2026-69380 manque dans la liste des CVE des articles KB des quatre SU de septembre. Le Security Update Guide indique toutefois précisément les quatre builds de septembre comme correctif pour cette CVE (état au 7 octobre 2026).

**[CVE-2026-69356](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69356)** présente la valeur la plus élevée, avec un CVSS de 9.3. Selon Microsoft, un attaquant non authentifié peut envoyer une invitation de calendrier préparée contenant un lien de réunion malveillant ; si la destinataire ou le destinataire ouvre la réunion et sélectionne le lien pour la rejoindre, le Cross-Site-Scripting est déclenché. Une interaction utilisateur est donc nécessaire, mais aucun compte dans l’organisation.

**[CVE-2026-69380](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69380)** (élévation de privilèges, CVSS 8.1) ne requiert qu’un compte disposant de peu de droits et une boîte aux lettres attribuée. Selon la FAQ du Security Update Guide, un attaquant peut, en exploitant des faiblesses dans la vérification des requêtes et des jetons d’identité, se faire passer pour un autre utilisateur et prendre le contrôle des boîtes aux lettres de tous les utilisateurs Exchange : lire et envoyer des e-mails, ainsi que télécharger les pièces jointes. Un seul compte utilisateur compromis suffit comme point de départ.

**[CVE-2026-69641](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69641)** (élévation de privilèges, CVSS 9.1) aboutit au même résultat, la prise de contrôle de toutes les boîtes aux lettres, mais requiert l’appartenance à un groupe de rôles hautement privilégié.

Pour les deux vulnérabilités d’exécution de code à distance, le point de départ diffère : [CVE-2026-69355](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69355) (CVSS 8.8) exige un compte authentifié disposant de droits limités, tandis que [CVE-2026-55007](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-55007) (CVSS 8.1) peut être déclenchée sans authentification au moyen d’une pièce jointe Visio préparée, mais nécessite selon Microsoft une mémoire vive durablement insuffisante sur le système cible. Les quatre autres failles : CVE-2026-69378 (DoS par récursion non contrôlée, sans authentification), CVE-2026-69361 (SSRF, le serveur envoie des requêtes HTTP à des systèmes internes ou de bouclage), CVE-2026-69375 (un attaquant authentifié peut remplacer le contenu de fichiers) et CVE-2026-69382 (divulgation d’identifiants via un algorithme cryptographique faible, nécessite un cookie d’authentification déjà dérobé).

## Version 2 du 2 octobre : CVE-2026-96940 ajoutée

Le 2 octobre 2026, Microsoft a publié la « version 2 » des SU de septembre. Selon l’annonce, la seule différence avec la première édition est le correctif supplémentaire pour **[CVE-2026-96940](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940)**. Cette faille d’élévation de privilèges a un CVSS de 8.8 ; elle n’est ni connue publiquement ni exploitée, mais Microsoft est la seule à l’évaluer parmi les dix CVE comme **« Exploitation More Likely »**. Un attaquant authentifié peut ainsi accéder aux boîtes aux lettres d’autres utilisateurs de la même organisation et lire les e-mails, y compris leurs pièces jointes. Exchange Online est déjà corrigé côté serveur.

| Version | KB | Build v2 |
| --- | --- | --- |
| Exchange SE RTM | KB5129955 | 15.2.2562.53 |
| Exchange 2019 CU15 | KB5129956 | 15.2.1748.53 |
| Exchange 2019 CU14 | KB5129957 | 15.2.1544.48 |
| Exchange 2016 CU23 | KB5129958 | 15.1.2507.75 |

En pratique, cela signifie que le correctif de CVE-2026-96940 est uniquement inclus dans les builds v2. Les serveurs sur lesquels la SU du 8 septembre est déjà installée nécessitent également v2. Si vous appliquez le correctif seulement maintenant, vous pouvez installer directement v2, car les SU sont cumulatives. Les articles KB n’indiquent pas explicitement si les serveurs disposant de la première SU de septembre doivent impérativement installer v2 ; puisque la nouvelle CVE n’est corrigée qu’avec v2, c’est l’interprétation logique.

## Problème des wrappers corrigé : supprimer maintenant le SettingOverride

Le problème connu depuis la SU de juin, selon lequel des *messages wrapper* apparaissent dans la boîte de réception des boîtes aux lettres partagées dans les environnements hybrides, est corrigé avec la SU de septembre sur les quatre versions. L’[article d’août](/blog/exchange-security-updates-august-2026) indiquait encore que le SettingOverride documenté comme solution de contournement pouvait rester en place. Après l’installation de la SU de septembre, c’est l’inverse : dans l’article de support associé, Microsoft recommande de vérifier et de supprimer l’override.

```powershell
Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
```

<details class="options-details">
<summary>Explication des options</summary>

| Commande | Effet |
|---|---|
| `Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Vérifie si l’override de contournement est défini dans l’organisation. |
| `Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Supprime l’override dès que la SU de septembre est installée. |

</details>

Si la première commande indique que l’objet `DisableBlockSharedAndUserMailboxHeaders` est introuvable, Microsoft précise qu’aucune autre action n’est requise.

Pour Exchange SE, la SU corrige également une erreur lors des requêtes de disponibilité hybride via Microsoft Graph : les utilisateurs on-premises voyaient les périodes occupées des boîtes aux lettres Exchange Online décalées de leur propre décalage UTC, sans message d’erreur.

## Problèmes connus

**Les calendriers publiés (.ics) renvoient HTTP 500 aux applications de calendrier.** Le problème existe depuis la SU d’août (Exchange SE à partir du build 15.2.2562.46 ainsi qu’Exchange 2019 et 2016) et n’est pas corrigé par la SU de septembre ni par v2. Les abonnements aux calendriers publiés anonymement ne se mettent plus à jour ; la même URL fonctionne dans le navigateur. Selon Microsoft, la cause est la suivante : Exchange reconnaît les clients à partir du User-Agent ; les applications de calendrier sans identifiant de navigateur empruntent un chemin de code que la SU d’août a désactivé. Comme solution de contournement, Microsoft décrit une règle de réécriture d’URL sur le site « Exchange Back End » dans IIS, qui ajoute le paramètre `layout=premium` aux requêtes .ics sous `/owa/calendar/` ; le module IIS URL Rewrite doit être installé. Les étapes précises (via le gestionnaire IIS ou directement dans `applicationHost.config`) figurent dans l’[article de support KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672). Depuis le 13 octobre 2026, le problème est corrigé avec les SU d’octobre selon KB5126672 (KB5128895 pour Exchange SE, KB5128896 à KB5128898 pour 2019 et 2016) ; après leur installation, la règle de réécriture doit être supprimée.

**Disponibilité pour les boîtes aux lettres déléguées dans les environnements hybrides (Exchange SE uniquement).** Si la requête de disponibilité est configurée exclusivement via l’API Graph, les requêtes vers les boîtes aux lettres Exchange Online échouent via un accès on-premises délégué. Outlook affiche « Your server location could not be determined », OWA affiche « No information », et `(403) Forbidden` apparaît dans les journaux EWS. La solution de contournement documentée redirige les requêtes via EWS plutôt que Graph :

```powershell
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=False"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-Identity EnableRouteThroughMSGraphFeature` | L’override qui contrôle le routage des requêtes de disponibilité via Microsoft Graph. |
| `-Parameters "Enabled=False"` | Désactive le chemin Graph ; les requêtes repassent par EWS. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Dirige l’appel de diagnostic vers le service de topologie. |
| `-Component VariantConfiguration -Argument Refresh` | Recharge la Variant Configuration afin que l’override prenne effet sans attendre. |

</details>

Dans l’annonce de v2, Microsoft liste ce problème parmi les problèmes corrigés. Entre-temps, l’[article de support KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) indique que le problème est corrigé dans le service. Si vous avez défini la solution de contournement, supprimez-la selon KB5127092 :

```powershell
Get-SettingOverride -Identity EnableRouteThroughMSGraphFeature
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=True"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `Get-SettingOverride -Identity EnableRouteThroughMSGraphFeature` | Indique si l’override est défini et quelle est sa valeur. |
| `-Parameters "Enabled=True"` | Réactive le chemin Graph pour les requêtes de disponibilité. |
| `-Component VariantConfiguration -Argument Refresh` | Recharge la Variant Configuration afin que la modification prenne effet sans attendre. |

</details>

**Interblocage de ContentEngine en raison de fichiers WordBreaker coréens manquants (Exchange SE uniquement).** Avec la SU de septembre (build 15.2.2562.49 et build v2 15.2.2562.53), les fichiers de règles du WordBreaker coréen mis à jour ne sont pas installés. Conséquences : résultats de recherche manquants, distribution des e-mails retardée, clients Outlook ou MAPI qui se bloquent ou perdent la connexion. [KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098) cite comme affectés les environnements qui traitent des e-mails contenant du texte coréen. Cela ne dépend pas de la langue de vos utilisateurs : des messages en coréen atteignent également les organisations sans correspondance coréenne, par exemple sous forme de spam ou de newsletter. Les environnements suisses ne sont donc pas exclus d’emblée. Depuis le 13 octobre 2026, le problème est corrigé selon KB5130098 avec KB5128895 (build 15.2.2562.56) ou une mise à jour plus récente ; Microsoft indique qu’aucune action supplémentaire n’est ensuite nécessaire, même sur les serveurs auxquels la solution manuelle a été appliquée. La vérification et la solution de contournement de la section suivante ne concernent donc plus que les serveurs sur la SU de septembre ou v2.

## Solution de contournement pour l’interblocage WordBreaker

La solution de contournement de KB5130098 ajoute deux fichiers de règles provenant du package SQL Server 2025 Express RTM. Microsoft la soumet à des conditions : elle ne doit pas être appliquée uniquement en raison des symptômes, mais seulement sur les serveurs qui remplissent tous les critères de vérification et auxquels **les deux** fichiers manquent. L’[Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) ne vérifie pas ces fichiers ; la version v26.10.02 du 2 octobre 2026 ne contient pas de vérification correspondante (état au 9 octobre 2026). Le contrôle doit donc être effectué manuellement.

### 1. Vérifier le serveur

Le script suivant ne modifie rien sur le système. Il lit le chemin d’installation dans la variable d’environnement `ExchangeInstallPath` et affiche les valeurs que KB5130098 indique comme prérequis.

```powershell
$native  = Join-Path $env:ExchangeInstallPath 'Bin\Search\Ceres\Native'
$exsetup = Get-Item (Join-Path $env:ExchangeInstallPath 'Bin\ExSetup.exe')
$dll     = Get-Item (Join-Path $native 'korwbrkr.dll')
[pscustomobject]@{
    ExSetup         = $exsetup.VersionInfo.FileVersion
    KorwbrkrVersion = $dll.VersionInfo.FileVersion
    KorwbrkrBytes   = $dll.Length
    KorwbrkrSha256  = (Get-FileHash -LiteralPath $dll.FullName -Algorithm SHA256).Hash
    KoTokenRule     = Test-Path (Join-Path $native 'ko.token.rule.bin')
    KoComplexRule   = Test-Path (Join-Path $native 'ko.complex.rule.bin')
}
```

<details class="options-details">
<summary>Explication des options</summary>

| Ligne | Effet |
|---|---|
| `Join-Path $env:ExchangeInstallPath 'Bin\Search\Ceres\Native'` | Construit le chemin vers le répertoire où doivent se trouver les fichiers de règles. |
| `Get-Item ...\ExSetup.exe` | Lit le fichier dont la version indique le build Exchange installé. |
| `VersionInfo.FileVersion` | Version du fichier issue des ressources de l’EXE ou de la DLL. |
| `Get-FileHash -LiteralPath ... -Algorithm SHA256` | Calcule le hachage SHA256 de `korwbrkr.dll` installée. |
| `Test-Path ...` | Renvoie `True` lorsque le fichier de règles respectif est déjà présent. |

</details>

La solution de contournement n’est autorisée que si toutes les valeurs correspondent :

| Valeur | Attendu |
| --- | --- |
| `ExSetup` | 15.02.2562.049 ou 15.02.2562.053 |
| `KorwbrkrVersion` | 16.0.5194.1000 |
| `KorwbrkrBytes` | 326544 |
| `KorwbrkrSha256` | 1C6BD8E144BA677EBCC83323AE59DB3881918170F9B3A5189B44611558B92C61 |
| `KoTokenRule`, `KoComplexRule` | les deux `False` |

Si le build, la version ou le hachage de la DLL diffère, Microsoft renvoie vers le support. Si l’un des deux fichiers de règles est déjà présent, arrêtez-vous et n’écrasez rien.

### 2. Obtenir les fichiers de règles

Les fichiers proviennent de `SQLEXPR_x64_ENU.exe`, le package anglais x64 de SQL Server 2025 Express RTM (version 17.0.1000.7, 748772024 octets). Microsoft recommande d’effectuer cette étape sur une station de travail d’administration, et non sur le serveur Exchange. SQL Server n’est pas installé : le package est uniquement décompressé, et la MSI Full-Text est seulement extraite en mode administratif. Vérifiez d’abord la signature et le hachage du téléchargement :

```powershell
Get-AuthenticodeSignature .\SQLEXPR_x64_ENU.exe |
    Select-Object Status, SignerCertificate
Get-FileHash .\SQLEXPR_x64_ENU.exe -Algorithm SHA256
```

Le statut attendu est `Valid` avec un certificat Microsoft, ainsi que la valeur SHA256 `74AA90C11202A5524E769B9BC22531BAEF22D91E9B2D2E8C3CB99E89A65C5297`. Décompressez et extrayez ensuite :

```powershell
$work = 'C:\Temp\KoreanRules'
$p = Start-Process .\SQLEXPR_x64_ENU.exe -ArgumentList '/q', "/x:$work\Media" -Wait -PassThru
$p.ExitCode
$msi = "$work\Media\x64\Setup\SQL_FULLTEXT.MSI"
$msiArgs = '/a', $msi, "TARGETDIR=$work\Files", '/qn', '/L*V', "$work\extract.log"
$p = Start-Process msiexec.exe -ArgumentList $msiArgs -Wait -PassThru
$p.ExitCode
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `/q` | Décompresse sans boîtes de dialogue. |
| `/x:<Pfad>` | Mode d’extraction du package SQL : décompresse les médias d’installation dans le dossier sans lancer SQL Setup. |
| `-Wait -PassThru` | Attend la fin du processus et renvoie l’objet processus afin que le code de sortie soit lisible. |
| `msiexec /a <MSI>` | Installation administrative : dépose le contenu de la MSI sous forme de fichiers, sans rien installer ni enregistrer. |
| `TARGETDIR=<Pfad>` | Dossier cible pour les fichiers extraits. |
| `/qn` | Sans interface utilisateur. |
| `/L*V <Datei>` | Journal détaillé qui indique la cause en cas d’erreur. |

</details>

Les deux codes de sortie doivent être `0`. Si l’extraction échoue, sauvegardez `extract.log` et arrêtez-vous. Vérifiez ensuite la taille et le hachage des deux fichiers :

```powershell
Get-ChildItem "$work\Files" -Recurse -Include 'ko.token.rule.bin', 'ko.complex.rule.bin' |
    Select-Object FullName, Length,
        @{ n = 'SHA256'; e = { (Get-FileHash -LiteralPath $_.FullName -Algorithm SHA256).Hash } }
```

| Fichier | Octets | SHA256 |
| --- | --- | --- |
| `ko.token.rule.bin` | 56132 | 8F2BD853593913EB8F73DCD4FCAC4216F216A0FF76A4569DF071BE3C36773010 |
| `ko.complex.rule.bin` | 717792 | 0390D1E9A76EF33283025CF8F164430E311584B9535949C4EA1A74B6BB107B87 |

Les fichiers se trouvent dans l’arborescence extraite sous `...\MSSQL\Binn\ftcomponents\wordbreakers`. Transférez uniquement ces deux fichiers dans un dossier de transit sur le serveur Exchange, sans médias SQL et surtout pas la version `korwbrkr.dll` du package SQL.

### 3. Copier les fichiers

```powershell
$native  = Join-Path $env:ExchangeInstallPath 'Bin\Search\Ceres\Native'
$staging = 'C:\Temp\KoStaging'
foreach ($f in 'ko.token.rule.bin', 'ko.complex.rule.bin') {
    Copy-Item -LiteralPath (Join-Path $staging $f) -Destination $native
    Get-FileHash -LiteralPath (Join-Path $native $f) -Algorithm SHA256
}
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `Copy-Item -LiteralPath <Quelle>` | Copie le fichier depuis le dossier de transit ; le chemin est évalué sans caractères génériques. |
| `-Destination $native` | Répertoire cible à côté de `korwbrkr.dll` installée. |
| `Get-FileHash ... -Algorithm SHA256` | Vérifie à nouveau le hachage à l’emplacement cible avec le tableau ci-dessus. |

</details>

Utilisez `Copy-Item`, et non `Move-Item`. Un fichier copié reçoit les autorisations héritées du répertoire cible ; un fichier déplacé conserve les autorisations de son dossier d’origine. Les fichiers requièrent les droits de lecture normaux de `Native`. Ne remplacez aucun autre fichier dans ce répertoire.

### 4. Redémarrer le Search Host Controller

```powershell
Restart-Service -Name HostControllerService
```

Le redémarrage interrompt temporairement la recherche et le traitement de contenu sur le serveur ; Microsoft prévoit à cet effet une fenêtre de maintenance. Il n’est pas nécessaire de redémarrer le serveur lui-même. Ne forcez pas l’arrêt de processus Exchange : si le service ne démarre pas correctement, l’étape suivante est, selon Microsoft, de contacter le support.

### 5. Vérification

Le Host Controller démarre lui-même le processus `NodeRunner.exe` de ContentEngine ; ne le démarrez pas manuellement. Cette requête permet de vérifier s’il est en cours d’exécution :

```powershell
Get-CimInstance Win32_Process -Filter "Name = 'NodeRunner.exe'" |
    Where-Object CommandLine -like '*ContentEngineNode1*' |
    Select-Object ProcessId, CreationDate
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `Get-CimInstance Win32_Process -Filter "Name = 'NodeRunner.exe'"` | Lit tous les processus NodeRunner avec leur ligne de commande. |
| `Where-Object CommandLine -like '*ContentEngineNode1*'` | Filtre sur le processus de ContentEngine. |
| `Select-Object ProcessId, CreationDate` | Affiche l’ID du processus et l’heure de démarrage ; l’heure de démarrage doit être postérieure au redémarrage du service. |

</details>

Si aucun processus n’apparaît ou s’il s’arrête à répétition, ne traitez pas d’autres serveurs. Dans le cas contraire, vérifiez dans Outlook sur le Web avec une boîte aux lettres dont la copie de base de données active se trouve sur ce serveur que les nouveaux e-mails sont distribués et trouvés, y compris les e-mails contenant du texte coréen. Contrôlez également le service initialement perturbé (distribution, connexion Outlook). Surveillez séparément le retard dans l’index de recherche : le fait que de nouveaux e-mails soient trouvés ne prouve pas que les éléments plus anciens ont déjà été traités. Traitez les autres serveurs un par un, et seulement après une vérification réussie du précédent.

## Installation et suivi

Microsoft recommande la procédure connue : inventorier avec l’[Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), déterminer le chemin avec l’[Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) si le niveau est obsolète, installer la SU, redémarrer le serveur et vérifier que tous les services Exchange fonctionnent. Le Health Checker indique ensuite également si la SU est correctement installée. Le Security Update Guide indique qu’un redémarrage est requis pour les mises à jour.

Après l’installation, trois opérations de suivi sont nécessaires :

1. Supprimer le SettingOverride des wrappers `DisableBlockSharedAndUserMailboxHeaders` s’il est défini (voir ci-dessus).

2. Sur Exchange SE, contrôler avec le script de vérification de la [section WordBreaker](#workaround-für-den-wordbreaker-deadlock) si les deux fichiers de règles manquent, et vérifier si le problème de disponibilité survient. N’appliquez les solutions de contournement que si les conditions correspondantes sont remplies.

3. Pour les calendriers publiés avec des abonnés externes, configurer la règle de réécriture d’URL de KB5126672 si cela n’a pas encore été fait depuis la SU d’août et que les SU d’octobre ne sont pas encore installées.

Depuis juillet, il reste également à vérifier si la mitigation CVE-2026-42897 (M2.1.0) est toujours active ; la procédure pour la supprimer figure dans l’[article sur la SU de juillet](/blog/exchange-security-updates-juli-2026).

## Procédure recommandée

Depuis le 13 octobre 2026, les SU d’octobre remplacent les SU de septembre selon les articles KB (Exchange SE : KB5128895, build 15.2.2562.56). Comme les SU sont cumulatives, installez directement la SU d’octobre sur tous les serveurs Exchange et toutes les machines équipées des Management Tools ; détails dans l’[article sur la SU d’octobre](/blog/exchange-security-updates-oktober-2026). Là où cela n’est pas encore possible, un serveur avec la SU du 8 septembre doit disposer au minimum de v2 pour CVE-2026-96940. La faille d’usurpation avec un CVSS de 9.3 et la prise de contrôle de boîte aux lettres au moyen d’un compte utilisateur disposant de peu de droits (CVE-2026-69380) constituent des raisons suffisantes pour ne pas attendre le prochain Patch Tuesday. Supprimez ensuite l’override des wrappers, vérifiez les trois problèmes connus et exécutez le Health Checker. Le Health Checker ne signale pas les fichiers WordBreaker manquants ; le script de vérification de la section WordBreaker est nécessaire à cet effet. Pour Exchange 2016 et 2019, le programme ESU prend fin en octobre 2026 ; la migration vers Exchange SE ne peut plus être reportée.

## Sources

1.  [Released: September 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-september-2026-exchange-server-security-updates/4554411): Annonce officielle de publication avec les versions prises en charge, l’indication ESU, les problèmes connus, les problèmes corrigés et la procédure d’installation (consultée via le flux RSS du Exchange Team Blog).

2.  [Released: September 2026 V2 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-september-2026-v2-exchange-server-security-updates/ba-p/4561718): Annonce de la v2 du 2 octobre 2026 ; la seule différence est CVE-2026-96940.

3.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM: September 8, 2026 (KB5121608) – Microsoft Support](https://support.microsoft.com/help/5121608): Liste des CVE, problèmes corrigés et les trois problèmes connus pour Exchange SE.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU15: September 8, 2026 (KB5121609) – Microsoft Support](https://support.microsoft.com/help/5121609): Article KB pour Exchange 2019 CU15.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU14: September 8, 2026 (KB5121610) – Microsoft Support](https://support.microsoft.com/help/5121610): Article KB pour Exchange 2019 CU14.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: September 8, 2026 (KB5121611) – Microsoft Support](https://support.microsoft.com/help/5121611): Article KB pour Exchange 2016 CU23, sans CVE-2026-55007.

7.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Type, CVSS, gravité, évaluation de l’exploitation et FAQ concernant les neuf CVE de septembre ainsi que CVE-2026-96940, y compris les builds concernés pour chaque CVE.

8.  [Description of version 2 of the security update for Microsoft Exchange Server Subscription Edition RTM October 2, 2026 (KB5129955) – Microsoft Support](https://support.microsoft.com/help/5129955): Article KB relatif à la v2 pour Exchange SE ; les articles v2 pour 2019 et 2016 sont KB5129956 à KB5129958.

9.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Numéros de build des SU de septembre et de la v2 du 2 octobre 2026.

10. [Wrapper messages appear in shared mailbox in hybrid environments after installing the June 2026 Security Update – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Correctif dans la SU de septembre et instructions pour supprimer le SettingOverride.

11. [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Cause, solution de contournement par réécriture d’URL et correctif avec les SU d’octobre du 13 octobre 2026.

12. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Symptômes, solution de contournement SettingOverride pour Exchange SE, correction dans le service et commandes de réinitialisation.

13. [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Builds SE concernés, critères de vérification, tailles et hachages attendus des fichiers, obtention depuis SQL Server 2025 Express, redémarrage du service, contrôle et correctif avec KB5128895.

14. [Hybrid free/busy through Microsoft Graph incorrectly shifts busy times by requester timezone – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5125804): Erreur de fuseau horaire dans Exchange SE corrigée avec la SU de septembre.

15. [Neue Sicherheitsupdates für Exchange Server (September 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-september-2026/): Présentation en allemand des neuf CVE avec valeurs CVSS et builds.

16. [Exchange Server: Sicherheitsupdates 8. September 2026 – Borns Tech and Windows World](https://borncity.com/blog/?p=329371): Résumé en allemand avec indications sur la mise à jour de remplacement ultérieure.

17. [microsoft/CSS-Exchange – GitHub](https://github.com/microsoft/CSS-Exchange): Code source de l’Exchange Health Checker ; version v26.10.02 du 2 octobre 2026 sans vérification des fichiers WordBreaker (état au 9 octobre 2026).

18. [Description of the security update for Microsoft Exchange Server Subscription Edition RTM October 13, 2026 (KB5128895) – Microsoft Support](https://support.microsoft.com/help/5128895): SU d’octobre qui remplace la SU de septembre et corrige l’erreur .ics ainsi que l’interblocage WordBreaker.
