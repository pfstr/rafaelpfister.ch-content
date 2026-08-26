---
title: "Claude Desktop plante constamment : « GPU process gone » avec le code de sortie 101457950, cause et solution"
navTitle: "Plantage de Claude Desktop"
description: "Sous Windows, l’application Claude Desktop se ferme complètement avec « GPU process gone: exitCode 101457950 » (0x060C201E), souvent suivi de la boîte de dialogue de réparation de l’application Store. La chaîne de causes complète : Code Integrity bloque vk_swiftshader.dll, la chaîne de repli de Chromium s’épuise, puis l’arrêt automatique intégré ferme l’application. Avec solution immédiate, autodiagnostic via le journal d’événements et analyse jusqu’au minidump."
date: "2026-08-25"
kategorie: "Claude"
timeToRead: "9 min de lecture"
themen:
  - claude
slug: "le-crash-gpu-0x060c201e-dans-l-application-de-bureau-claude-enquete-jusqu-au-minidump"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
translationSourceHash: 2cf7e9455d4d9b5c148e7b55fd0433206810dc26e53bacb85e1d2dc82a0444c6
translationModel: gpt-5.6-terra
translatedAt: 2026-08-26T04:07:18.159Z
translationReview: automatic
url: https://rafaelpfister.ch/fr/blog/le-crash-gpu-0x060c201e-dans-l-application-de-bureau-claude-enquete-jusqu-au-minidump
---

Sous Windows, l’application Claude Desktop se ferme sans message d’erreur, toutes les sessions Claude Code en cours disparaissent et il arrive que l’application ne redémarre ensuite qu’après une « Réparation » via les paramètres Windows. Cette ligne apparaît alors dans le journal de l’application :

```text
GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
```

101457950 s’écrit en hexadécimal `0x060C201E`. Si vous trouvez cette signature dans votre journal, vous êtes au bon endroit : cet article documente la chaîne de causes complète de ce plantage, les mesures immédiates permettant de stabiliser l’application, ainsi que l’autodiagnostic qui permet de confirmer le constat sur votre propre système en deux minutes. Les installations provenant du Microsoft Store (MSIX) sont concernées sur tous les fabricants de GPU, des iGPU Intel à NVIDIA et AMD ; le matériel n’est, précisons-le d’emblée, pas la cause.

## La solution en bref

L’erreur réelle se trouve dans le paquet d’installation de l’application et ne peut être corrigée que par Anthropic (toujours ouvert au 25.08.2026, ticket [#81341](https://github.com/anthropics/claude-code/issues/81341)). En attendant, trois mesures rendent l’application stable, par ordre d’efficacité :

**1. Activer l’accélération matérielle.** Vérifiez ces deux valeurs dans le fichier `%LOCALAPPDATA%\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude\config.json` et définissez-les sur `false` si nécessaire (quittez l’application avant, puis redémarrez-la) :

```json
"isHardwareAccelerationDisabled": false,
"isHardwareAccelerationAutoDisabled": false
```

Cela semble paradoxal, car désactiver l’accélération matérielle est généralement le choix le plus stable. Pour ce bug, c’est l’inverse, et la chaîne de causes expliquée plus bas montre pourquoi : ce paramètre détermine si un plantage du processus GPU ne coûte qu’un niveau de repli ou toute l’application.

**2. Utiliser avec parcimonie le navigateur intégré.** Les pages dans la zone navigateur/aperçu de l’application déclenchent le plantage. Fermer la zone après utilisation au lieu de laisser des onglets ouverts réduit drastiquement la fréquence des plantages ; ce lien est étayé à plusieurs reprises par des chiffres dans le fil communautaire.

**3. Facultatif : désactiver WebGPU.** Un démarrage avec `--disable-features=WebGPU` empêche totalement le déclencheur le plus fréquent. Avec une application Store, le chemin d’installation change à chaque mise à jour ; il faut donc un lanceur qui le résout à nouveau à chaque démarrage :

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

Inconvénient : cela ne fonctionne que si l’application est lancée via ce lanceur. La mesure 1 s’applique à chaque démarrage.

Une « Réparation » ou une réinstallation de l’application ne résout d’ailleurs pas le problème ; elle ne traite que le symptôme secondaire (voir plus bas). Les mises à jour des pilotes graphiques sont également vaines.

## Autodiagnostic : confirmer le constat sur son propre système

Deux vérifications suffisent. Premièrement, la signature du plantage dans le journal de l’application :

```powershell
Select-String -Path "$env:LOCALAPPDATA\Claude\Logs\main.log" `
  -Pattern 'GPU process gone'
```

Deuxièmement, et c’est la véritable preuve, le journal CodeIntegrity de Windows :

```powershell
Get-WinEvent -FilterHashtable @{
  LogName='Microsoft-Windows-CodeIntegrity/Operational'; Id=3033
} -MaxEvents 30 | Where-Object { $_.Message -match 'claude' } |
  Select-Object TimeCreated, Message
```

Sur les systèmes concernés, vous y trouverez des entrées Event 3033 dont les horodatages correspondent à la seconde près aux heures des plantages, avec ce message :

```text
Code Integrity determined that a process
(...\WindowsApps\Claude_..._x64__pzs8sxrjxfjjc\app\claude.exe)
attempted to load ...\app\vk_swiftshader.dll that did not meet
the Microsoft signing level requirements.
```

Sur le système examiné ici, sept plantages sur sept, répartis sur trois semaines, correspondaient à la seconde près à un tel événement, y compris un plantage de contrôle déclenché délibérément.

## La chaîne de causes complète

Le plantage est le dernier maillon d’une chaîne de quatre éléments révélée par deux analyses : la piste Code Integrity issue du ticket communautaire [#81698](https://github.com/anthropics/claude-code/issues/81698) et notre propre analyse de minidump ([#89250](https://github.com/anthropics/claude-code/issues/89250)).

**Maillon 1 : une page dans le navigateur intégré nécessite un rendu logiciel.** Un appel WebGPU (`navigator.gpu.requestAdapter()`) est un déclencheur typique, reconnaissable dans le journal de la fenêtre à cet avertissement juste avant le plantage :

```text
[warn] The powerPreference option is currently ignored
       when calling requestAdapter() on Windows.
```

Lorsque l’application s’exécute sans accélération matérielle, le chemin passe obligatoirement par l’implémentation logicielle Vulkan SwiftShader : le processus GPU tente de charger la `vk_swiftshader.dll` fournie.

**Maillon 2 : Windows Code Integrity bloque la DLL de l’application elle-même.** Le processus GPU s’exécute avec la stratégie de renforcement « MicrosoftSignedOnly » (vérifiable via `Get-ProcessMitigation`). Pour qu’une application Store puisse charger ses propres DLL signées par le fabricant, le paquet MSIX doit inclure un catalogue de signatures `AppxMetadata\CodeIntegrity.cat`. Or ce fichier manque précisément dans le paquet distribué, comme des membres de la communauté l’ont démontré en inspectant le fichier MSIX. Conséquence : la vérification de signature échoue, Windows enregistre l’Event 3033 et termine brutalement le processus GPU. Le code de sortie `0x060C201E` est une erreur d’intégrité AppX du chargeur Windows, et non un code Chromium ; c’est pourquoi il ne figure dans aucune source Chromium, et pourquoi le processus GPU ne laisse pas non plus de dump de crash : il n’y a aucune exception à capturer dans un dump.

**Maillon 3 : la chaîne de repli de Chromium s’épuise.** Lorsqu’un processus GPU plante, Chromium recule d’un niveau de rendu : GL matériel, puis GL logiciel, puis simple compositeur d’affichage. Ce n’est que lorsqu’il ne reste plus aucun niveau que l’arrêt automatique intégré intervient. Dans le code source de la version embarquée (Chromium 148.0.7778.280 dans Electron 42.9.2), il est écrit littéralement ainsi :

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

**Maillon 4 : le processus principal se termine volontairement.** Cet `LOG(FATAL)` est le moment où « l’application plante ». Cela est prouvé par un minidump du processus principal : `EXCEPTION_BREAKPOINT` (un `int3` intentionnel, pas une erreur de pilote), aucune DLL de pilote graphique dans le processus, et en clair dans la mémoire :

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

Le fait que ce dump existe a été la partie la plus difficile de l’analyse : l’intégration Sentry de l’application consomme les dumps Crashpad au démarrage suivant de l’application, les envoie à la télémétrie du fabricant et les supprime localement. Le dossier Crashpad est donc toujours vide pour l’utilisateur. La solution consiste en un observateur indépendant de l’arbre des processus de l’application (lancé via WMI afin que le plantage de l’application ne le termine pas avec elle), qui recherche toutes les 200 millisecondes `*.dmp` dans la base de données Crashpad et copie immédiatement les éléments trouvés avant leur suppression. L’évaluation est assurée par le paquet Python `minidump`, sans WinDbg.

## Pourquoi « désactiver l’accélération matérielle » aggrave tout

La chaîne explique aussi le constat le plus contre-intuitif. Ici, l’accélération matérielle désactivée a simultanément deux effets fatals. Premièrement, elle impose le chemin SwiftShader, c’est-à-dire précisément la tentative de chargement de DLL que Code Integrity bloque ; avec l’accélération matérielle active, `vk_swiftshader.dll` n’est en revanche presque jamais nécessaire. Deuxièmement, le processus GPU démarre alors déjà à l’extrémité inférieure de la chaîne de repli : un seul plantage suffit et le maillon 4 entre en jeu. Cela explique aussi l’observation du fil communautaire selon laquelle un blocage Code Integrity reste parfois sans conséquence et termine parfois l’application : tout dépend du nombre de niveaux de repli restant au processus navigateur.

Particulièrement malheureux : l’application connaît une désactivation automatique de l’accélération matérielle après des problèmes (`isHardwareAccelerationAutoDisabled`). Conçue comme mesure de stabilité, elle place les systèmes concernés précisément dans la configuration où le plantage suivant coûte toute l’application.

## Le symptôme secondaire : la boucle de réparation

L’échec de Code Integrity a un effet secondaire que de nombreuses personnes concernées considèrent comme un problème distinct : après l’incident, Windows classe parfois le paquet de l’application comme « Modified, NeedsRemediation ». L’application ne démarre alors plus du tout jusqu’à ce qu’elle soit réinitialisée via Paramètres → Applications → Claude → Options avancées → « Réparer ». Ceux qui doivent donc « constamment réparer » l’application constatent le même problème de fond, simplement un maillon plus loin : la réparation corrige l’état du paquet, pas la cause ; le plantage suivant survient lors de la prochaine tentative bloquée de chargement de DLL.

## État des signalements

La cause liée au empaquetage est signalée dans [#81341](https://github.com/anthropics/claude-code/issues/81341), le fil collectif contenant les preuves de la communauté est [#81698](https://github.com/anthropics/claude-code/issues/81698), et l’analyse de minidump avec l’explication de la chaîne de repli est [#89250](https://github.com/anthropics/claude-code/issues/89250). Le véritable correctif, un catalogue de signatures complet dans le paquet MSIX, relève d’Anthropic. D’ici là : activez l’accélération matérielle, fermez rigoureusement la zone navigateur et, si nécessaire, désactivez WebGPU via un indicateur. Sur le système examiné ici, l’application ne plante plus depuis la mise en œuvre de la mesure 1.

## Sources

1.  [Ticket GitHub #81698: GPU process crash kills entire app](https://github.com/anthropics/claude-code/issues/81698): Le fil collectif avec les preuves de la communauté concernant la chaîne Code Integrity, les données couvrant plusieurs fabricants et la corrélation avec le panneau navigateur.

2.  [Ticket GitHub #81341: CIG + vk_swiftshader.dll kills GPU process](https://github.com/anthropics/claude-code/issues/81341): La cause liée à l’empaquetage ; catalogue CodeIntegrity manquant dans le MSIX.

3.  [Ticket GitHub #89250: Analyse du minidump de l’arrêt de l’application](https://github.com/anthropics/claude-code/issues/89250): La seconde moitié de la chaîne décrite ici, avec la méthode de capture de dump et des propositions de correctifs.

4.  [Code source Chromium : gpu_data_manager_impl_private.cc (balise 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): La fonction IntentionallyCrashBrowserForUnusableGpuProcess et la logique de repli.

5.  [Documentation Electron : child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): L’événement permettant à une application Electron d’observer les plantages du processus GPU et de prendre ses propres contre-mesures.

6.  [Paquet Python minidump](https://pypi.org/project/minidump/): Outil d’analyse de dump (enregistrement d’exception, liste des modules, chaînes mémoire).

7.  [webgpureport.org](https://webgpureport.org/): Page de diagnostic WebGPU ; a servi de déclencheur minimal pour le plantage de contrôle et pour le test comparatif dans le Chromium actuel.
