---
title: "Claude Desktop plante constamment : « GPU process gone » avec le code de sortie 101457950, cause et solution"
navTitle: "Plantage de Claude Desktop"
description: "L’application Claude Desktop sous Windows se ferme complètement avec « GPU process gone: exitCode 101457950 » (0x060C201E), souvent suivi de la boîte de dialogue de réparation de l’application Store. La chaîne complète des causes : Code Integrity bloque vk_swiftshader.dll, la chaîne de repli de Chromium s’épuise, l’arrêt automatique intégré ferme l’application. Avec une solution durable (passage à l’installation classique sans MSIX), un autodiagnostic via le journal des événements et une analyse jusqu’au minidump."
date: "2026-08-25"
kategorie: "Claude"
timeToRead: "10 min de lecture"
themen:
  - claude
slug: "le-crash-gpu-0x060c201e-dans-l-application-de-bureau-claude-enquete-jusqu-au-minidump"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
translationSourceHash: 769984b49b04b65b0b8f8a91ce3b6dd65e2eef1a4212bed32b83422f431a8559
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:23:53.072Z
translationReview: automatic
url: https://rafaelpfister.ch/fr/blog/le-crash-gpu-0x060c201e-dans-l-application-de-bureau-claude-enquete-jusqu-au-minidump
---

L’application Claude Desktop sous Windows se ferme sans message d’erreur, toutes les sessions Claude Code en cours disparaissent, et il arrive que l’application ne redémarre qu’après une « Réparation » dans les paramètres Windows. Cette ligne apparaît alors dans le journal de l’application :

```text
GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
```

101457950 est, en hexadécimal, `0x060C201E`. Si vous trouvez cette signature dans votre journal, vous êtes au bon endroit : cet article documente la chaîne complète des causes de ce plantage, les mesures immédiates permettant à l’application de redevenir stable et l’autodiagnostic qui permet de confirmer le constat sur votre propre système en deux minutes. Les installations MSIX (depuis le Microsoft Store ou via le programme d’installation MSIX) sont concernées sur tous les fabricants de GPU, des iGPU Intel à NVIDIA et AMD ; le matériel n’est, précisons-le d’emblée, pas en cause. L’installation classique sans MSIX n’est pas affectée, et c’est précisément la solution.

## La solution en bref : passer à l’installation classique

L’erreur réelle se trouve dans le paquet d’installation MSIX et seul Anthropic peut la corriger (toujours ouverte au 27.08.2026, issue [#81341](https://github.com/anthropics/claude-code/issues/81341); la version actuelle 1.37937.3 est également concernée). La même application existe toutefois aussi sous forme d’installation classique sans MSIX, qui n’est pas soumise au contrôle de signature AppX mettant fin au processus GPU. Ce passage est donc la seule mesure qui élimine entièrement le plantage ; il est confirmé tant dans l’issue [#81341](https://github.com/anthropics/claude-code/issues/81341) que sur le système étudié ici. Les fonctionnalités sont identiques et le flux de mises à jour fournit les mêmes versions pour les deux variantes.

**Étape 1 : télécharger et exécuter l’installeur classique.** Le téléchargement sur [claude.com/download](https://claude.com/download) fournit un installeur Squirrel qui installe l’application dans `%LOCALAPPDATA%\AnthropicClaude` (aucun droit administrateur nécessaire). En ligne de commande :

```powershell
curl.exe -L -o "$env:USERPROFILE\Downloads\Claude-Setup-x64.exe" `
  "https://storage.googleapis.com/osprey-downloads-c02f6a0d-347c-492b-a752-3e0651722e97/nest-win-x64/Claude-Setup-x64.exe"
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-L` | suit les redirections HTTP jusqu’au fichier proprement dit |
| `-o <pfad>` | fichier cible ; ici, le dossier Téléchargements |
| `<url>` | source officielle de l’installeur ; identique à la destination de la redirection de téléchargement de claude.ai |

</details>

Après le téléchargement, vérifiez la signature (`Get-AuthenticodeSignature`, attendu : `Valid`, émetteur « Anthropic, PBC ») puis lancez le fichier. L’installeur dépose d’abord une version de base plus ancienne ; le mécanisme de mise à jour la met à niveau, automatiquement au premier démarrage ou immédiatement avec :

```powershell
& "$env:LOCALAPPDATA\AnthropicClaude\Update.exe" `
  --update https://downloads.claude.ai/releases/win32/x64
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `--update <url>` | télécharge la dernière version depuis le flux de publications indiqué et l’installe dans un nouveau répertoire `app-<version>` |

</details>

**Étape 2 : reprendre la configuration.** La version MSIX conserve la connexion, la configuration des serveurs MCP et les paramètres dans son conteneur virtualisé ; l’application classique lit `%APPDATA%\Claude`. Copiez-les une seule fois (fermez auparavant l’application MSIX ; les deux variantes ne peuvent de toute façon pas fonctionner simultanément en raison d’un verrou partagé d’instance unique) :

```powershell
robocopy "$env:LOCALAPPDATA\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude" `
  "$env:APPDATA\Claude" /E /XD Cache "Code Cache" GPUCache claude-code Crashpad logs sentry
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `<quelle>` | dossier de configuration dans AppData virtualisé du paquet MSIX |
| `<ziel>` | dossier de configuration de l’installation classique |
| `/E` | copie tous les sous-répertoires, y compris les vides |
| `/XD <namen>` | ignore les répertoires indiqués ; ici, les caches et données d’exécution que la nouvelle application recrée elle-même |

</details>

Les historiques de conversation ne sont pas perdus : ils se trouvent dans le compte claude.ai ou, pour les sessions Claude Code, sous `%USERPROFILE%\.claude` et ne dépendent pas de l’installation de l’application.

**Étape 3 : supprimer le paquet MSIX.** Sinon, les anciens raccourcis continueront à lancer la variante qui plante :

```powershell
Get-AppxPackage Claude | Remove-AppxPackage
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `Claude` | argument positionnel de `Get-AppxPackage`: filtre les paquets AppX/MSIX installés selon le nom du paquet (caractères génériques autorisés) |
| `Remove-AppxPackage` | supprime pour le compte utilisateur actuel le paquet transmis via le pipeline |

</details>

L’entrée du menu Démarrer « Anthropic → Claude » correspond ensuite à l’installation classique ; un éventuel raccourci épinglé à la barre des tâches doit être recréé.

## Si vous devez conserver le paquet MSIX

Sans ce changement, il ne reste que des mesures qui réduisent la fréquence des plantages sans en supprimer la cause :

**Utilisez avec parcimonie le navigateur intégré.** Les pages dans la zone navigateur/aperçu de l’application déclenchent le plantage. Fermer cette zone après usage au lieu d’y laisser des onglets ouverts réduit sensiblement la fréquence des plantages ; cette corrélation est étayée à plusieurs reprises par des chiffres dans le fil communautaire.

**Désactivez WebGPU.** Un lancement avec `--disable-features=WebGPU` empêche le déclencheur le plus fréquent. Avec un paquet MSIX, le chemin d’installation change à chaque mise à jour ; utilisez donc un lanceur qui le résout à nouveau à chaque démarrage :

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `for /f "delims="` | traite la sortie de commande ligne par ligne ; `delims=` vide reprend la ligne entière (y compris les espaces dans le chemin) dans `%%i` |
| `-NoProfile` | lance PowerShell sans scripts de profil, pour un démarrage rapide et reproductible |
| `-Command` | exécute l’expression indiquée ; `(Get-AppxPackage Claude).InstallLocation` renvoie le chemin d’installation actuel du paquet |
| `start ""` | démarre le programme indépendamment de la fenêtre Batch ; les guillemets vides correspondent au titre de fenêtre (vide ici) |
| `--disable-features=WebGPU` | commutateur Chromium : désactive la fonctionnalité indiquée, ici l’API WebGPU |

</details>

Cela ne fonctionne que si l’application est effectivement lancée via ce lanceur.

La première version de cet article recommandait en premier lieu d’activer l’accélération matérielle avec `isHardwareAccelerationDisabled: false` dans `config.json`. Cette recommandation est obsolète : dans les versions actuelles (1.37937.x), le drapeau n’existe plus, l’application démarre par défaut avec l’accélération matérielle active et plante malgré tout (détails dans l’ajout ci-dessous).

Une « Réparation » ou une réinstallation du paquet MSIX ne résout d’ailleurs pas le problème ; elle ne traite que le symptôme consécutif (voir plus bas). Mettre à jour les pilotes graphiques est également vain.

## Autodiagnostic : confirmer le constat sur votre propre système

Deux vérifications suffisent. Premièrement, la signature du plantage dans le journal de l’application :

```powershell
Select-String -Path "$env:LOCALAPPDATA\Claude\Logs\main.log" `
  -Pattern 'GPU process gone'
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-Path` | fichier à parcourir, ici le journal principal de l’application |
| `-Pattern` | motif de recherche (expression régulière) ; affiche toutes les lignes contenant la signature du plantage |

</details>

Deuxièmement, et c’est la preuve proprement dite, le journal CodeIntegrity de Windows :

```powershell
Get-WinEvent -FilterHashtable @{
  LogName='Microsoft-Windows-CodeIntegrity/Operational'; Id=3033
} -MaxEvents 30 | Where-Object { $_.Message -match 'claude' } |
  Select-Object TimeCreated, Message
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-FilterHashtable` | filtre dès la récupération : `LogName` désigne le journal des événements, `Id` l’ID d’événement 3033 (blocage Code Integrity) |
| `-MaxEvents 30` | limite la requête aux 30 résultats les plus récents |
| `Where-Object { … -match 'claude' }` | ne conserve que les événements dont le texte du message contient le chemin de l’application |
| `Select-Object TimeCreated, Message` | réduit la sortie à l’horodatage et au message afin de les comparer aux heures de plantage |

</details>

Sur les systèmes affectés, vous y trouverez des entrées Event 3033 dont les horodatages coïncident à la seconde près avec les heures de plantage, avec ce message :

```text
Code Integrity determined that a process
(...\WindowsApps\Claude_..._x64__pzs8sxrjxfjjc\app\claude.exe)
attempted to load ...\app\vk_swiftshader.dll that did not meet
the Microsoft signing level requirements.
```

Sur le système étudié ici, sept plantages sur sept, sur trois semaines, coïncidaient à la seconde près avec un tel événement, y compris un plantage de contrôle provoqué délibérément.

## La chaîne complète des causes

Le plantage est le dernier maillon d’une chaîne de quatre éléments établie conjointement par deux analyses : la piste Code Integrity issue de l’issue communautaire [#81698](https://github.com/anthropics/claude-code/issues/81698) et une analyse personnelle de minidump ([#89250](https://github.com/anthropics/claude-code/issues/89250)).

**Maillon 1 : une page dans le navigateur intégré nécessite un rendu logiciel.** Un appel WebGPU est un déclencheur typique (`navigator.gpu.requestAdapter()`), identifiable dans le journal de la fenêtre par cet avertissement juste avant le plantage :

```text
[warn] The powerPreference option is currently ignored
       when calling requestAdapter() on Windows.
```

Si l’application fonctionne sans accélération matérielle, le chemin passe obligatoirement par l’implémentation Vulkan logicielle SwiftShader : le processus GPU tente de charger `vk_swiftshader.dll` fourni avec l’application.

**Maillon 2 : Windows Code Integrity bloque la propre DLL de l’application.** Le processus GPU s’exécute avec la stratégie de renforcement « MicrosoftSignedOnly » (vérifiable avec `Get-ProcessMitigation`). Pour qu’une application Store puisse charger ses propres DLL signées par le fabricant, le paquet MSIX doit contenir un catalogue de signatures `AppxMetadata\CodeIntegrity.cat`. Or ce fichier manque précisément dans le paquet livré, comme des membres de la communauté l’ont établi en inspectant le fichier MSIX. Conséquence : la vérification de signature échoue, Windows consigne l’événement 3033 et termine brutalement le processus GPU. Le code de sortie `0x060C201E` est une erreur d’intégrité AppX du chargeur Windows, et non un code Chromium ; c’est pourquoi il n’apparaît dans aucune source Chromium et que le processus GPU ne laisse pas non plus de dump de plantage : il n’y a aucune exception à exporter dans un dump.

**Maillon 3 : la chaîne de repli de Chromium s’épuise.** Lorsque le processus GPU plante, Chromium redescend d’un niveau de rendu : GL matériel, puis GL logiciel, puis simple compositeur d’affichage. Ce n’est que lorsqu’il ne reste plus aucun niveau que l’arrêt automatique intégré intervient. Dans le code source de la version intégrée (Chromium 148.0.7778.280 dans Electron 42.9.2), il est formulé littéralement ainsi :

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

**Maillon 4 : le processus principal se termine volontairement.** Cet `LOG(FATAL)` est le moment où « l’application plante ». Cela est démontré par un minidump du processus principal : `EXCEPTION_BREAKPOINT` (un `int3` intentionnel, pas une erreur de pilote), aucune DLL de pilote graphique dans le processus et, en clair dans la mémoire :

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

L’existence même de ce dump a été la partie la plus difficile de l’analyse : l’intégration Sentry de l’application consomme les dumps Crashpad au prochain démarrage de l’application, les envoie à la télémétrie du fabricant et les supprime localement. Le dossier Crashpad est donc toujours vide pour l’utilisateur. La solution consiste en un observateur indépendant de l’arborescence des processus de l’application (démarré via WMI afin que le plantage de l’application ne le termine pas aussi), qui examine la base de données Crashpad toutes les 200 millisecondes à la recherche de `*.dmp` et copie immédiatement les fichiers trouvés avant leur suppression. Le paquet Python `minidump` assure l’analyse, sans WinDbg.

## Pourquoi « désactiver l’accélération matérielle » aggrave tout

La chaîne explique aussi le constat le plus contre-intuitif. L’accélération matérielle désactivée a ici deux effets fatals simultanés. Premièrement, elle force le chemin SwiftShader, donc précisément la tentative de chargement de DLL bloquée par Code Integrity ; avec l’accélération matérielle active, `vk_swiftshader.dll` n’est en revanche presque jamais nécessaire. Deuxièmement, le processus GPU démarre alors déjà au bas de la chaîne de repli : un seul plantage suffit pour que le maillon 4 se déclenche. Cela explique également l’observation du fil communautaire selon laquelle un blocage Code Integrity reste parfois sans conséquence et termine parfois l’application : cela dépend du nombre de niveaux de repli qu’il reste au processus navigateur.

Particulièrement regrettable : l’application connaissait une désactivation automatique de l’accélération matérielle après des problèmes (`isHardwareAccelerationAutoDisabled`). Conçue comme mesure de stabilité, elle amenait les systèmes concernés précisément dans la configuration où le plantage suivant coûte toute l’application.

## Ajout du 27.08.2026 : l’accélération matérielle seule ne suffit pas

La première version de cet article recommandait l’accélération matérielle active comme mesure immédiate la plus efficace, et pendant deux jours l’application est effectivement restée exempte de plantage. Puis la mise à jour automatique vers 1.37937.3 est arrivée, accompagnée de trois plantages en un après-midi, chacun avec le désormais connu événement 3033 concernant `vk_swiftshader.dll`. Deux constats en résultent :

Premièrement, le catalogue de signatures manquant est également absent du paquet MSIX actuel ; le problème fondamental reste inchangé dans 1.37937.3.

Deuxièmement, l’accélération matérielle active ne protège que statistiquement : elle allonge la chaîne de repli, mais n’empêche pas Chromium de la parcourir jusqu’au niveau SwiftShader sous charge ou après une erreur du processus GPU matériel. Dès que cela se produit, Code Integrity bloque la DLL et la chaîne peut malgré tout s’épuiser. En outre, les drapeaux de configuration `isHardwareAccelerationDisabled`/`isHardwareAccelerationAutoDisabled` ont disparu de `config.json` dans 1.37937.x ; le réglage ne peut plus y être imposé.

La seule solution fiable est donc restée le passage à l’installation classique décrit plus haut. Depuis ce changement sur le système étudié ici : même version de l’application, utilisation identique incluant la zone navigateur, pas un seul événement 3033 et aucun plantage.

## Le symptôme consécutif : la boucle de réparation

L’échec de Code Integrity a un effet secondaire que de nombreuses personnes concernées prennent pour un problème distinct : Windows classe parfois le paquet de l’application après l’incident comme « Modified, NeedsRemediation ». L’application ne démarre alors plus du tout tant qu’elle n’est pas réinitialisée via Paramètres → Applications → Claude → Options avancées → « Réparer ». Si vous devez donc « réparer constamment » l’application, vous observez le même problème fondamental, un maillon plus loin : la réparation corrige l’état du paquet, non la cause ; le plantage suivant survient à la prochaine tentative de chargement de DLL bloquée.

## État des signalements

La cause liée au packaging est signalée dans [#81341](https://github.com/anthropics/claude-code/issues/81341), le fil collectif avec les preuves de la communauté est [#81698](https://github.com/anthropics/claude-code/issues/81698), l’analyse de minidump expliquant la chaîne de repli est [#89250](https://github.com/anthropics/claude-code/issues/89250), et un autre rapport détaillé incluant la boucle de réparation est [#80444](https://github.com/anthropics/claude-code/issues/80444). Le correctif proprement dit, un catalogue complet de signatures dans le paquet MSIX, relève d’Anthropic et reste absent de 1.37937.3. D’ici là : passez à l’installation classique ; si vous devez conserver le paquet MSIX, fermez rigoureusement la zone navigateur et désactivez WebGPU au besoin via le drapeau. Sur le système étudié ici, l’application ne plante plus depuis le passage à l’installation classique, sans un seul événement 3033 supplémentaire.

## Sources

1.  [GitHub Issue #81698: GPU process crash kills entire app](https://github.com/anthropics/claude-code/issues/81698): Le fil collectif réunissant les preuves de la communauté concernant la chaîne Code Integrity, les données sur plusieurs fabricants et la corrélation avec le panneau navigateur.

2.  [GitHub Issue #81341: CIG + vk_swiftshader.dll kills GPU process](https://github.com/anthropics/claude-code/issues/81341): La cause liée au packaging ; catalogue CodeIntegrity manquant dans le MSIX.

3.  [GitHub Issue #89250: analyse du minidump de l’arrêt de l’application](https://github.com/anthropics/claude-code/issues/89250): La seconde moitié de la chaîne décrite ici, avec la méthode de capture des dumps et des propositions de correction.

4.  [GitHub Issue #80444: plantage GPU avec analyse forensique et boucle de réparation](https://github.com/anthropics/claude-code/issues/80444): Rapport individuel détaillé avec chronologies, analyse du journal des événements et constat que chaque plantage fait passer le paquet à l’état « Modified ».

5.  [Claude Desktop : page de téléchargement officielle](https://claude.com/download): Source de l’installeur Windows classique (x64 et ARM64).

6.  [Code source Chromium : gpu_data_manager_impl_private.cc (tag 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): La fonction IntentionallyCrashBrowserForUnusableGpuProcess et la logique de repli.

7.  [Documentation Electron : child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): L’événement par lequel une application Electron peut observer les plantages du processus GPU et prendre ses propres contre-mesures.

8.  [Paquet Python minidump](https://pypi.org/project/minidump/): Outil d’analyse des dumps (enregistrement d’exception, liste des modules, chaînes en mémoire).

9.  [webgpureport.org](https://webgpureport.org/): Page de diagnostic WebGPU ; utilisée comme déclencheur minimal du plantage de contrôle et pour le test comparatif dans Chromium actuel.
