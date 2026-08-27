---
title: "Claude Desktop plante constamment : « GPU process gone » avec le code de sortie 101457950, cause et solution"
navTitle: "Plantage de Claude Desktop"
description: "L’application Claude Desktop sous Windows se ferme entièrement avec « GPU process gone: exitCode 101457950 » (0x060C201E), souvent suivi de la boîte de dialogue de réparation de l’application Store. La chaîne de causes complète : Code Integrity bloque vk_swiftshader.dll, la chaîne de repli de Chromium s’épuise, l’arrêt automatique intégré ferme l’application. Avec solution immédiate, autodiagnostic via le journal d’événements et analyse jusqu’au minidump."
date: "2026-08-25"
kategorie: "Claude"
timeToRead: "9 min de lecture"
themen:
  - claude
slug: "le-crash-gpu-0x060c201e-dans-l-application-de-bureau-claude-enquete-jusqu-au-minidump"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
translationSourceHash: 61bcad89e160ee37f5abd04905ed9e425236f770f9cfcc4448716acbd3569939
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:34:25.240Z
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

101457950 s’écrit en hexadécimal `0x060C201E`. Si vous trouvez cette signature dans votre journal, vous êtes au bon endroit : cet article documente la chaîne complète des causes de ce plantage, les mesures immédiates qui rétablissent la stabilité de l’application et l’autodiagnostic qui permet de confirmer le constat sur votre propre système en deux minutes. Les installations issues du Microsoft Store (MSIX) sont concernées sur tous les fabricants de GPU, des GPU intégrés Intel à NVIDIA et AMD ; le matériel n’est, pour l’anticiper, pas la cause.

## La solution en bref

L’erreur réelle se trouve dans le package d’installation de l’application et ne peut être corrigée que par Anthropic (toujours ouvert au 25.08.2026, issue [#81341](https://github.com/anthropics/claude-code/issues/81341)). D’ici là, trois mesures rendent l’application stable, par ordre d’efficacité :

**1. Activer l’accélération matérielle.** Vérifiez les deux valeurs dans le fichier `%LOCALAPPDATA%\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude\config.json` et définissez-les sur `false` si nécessaire (fermez l’application au préalable, puis redémarrez-la) :

```json
"isHardwareAccelerationDisabled": false,
"isHardwareAccelerationAutoDisabled": false
```

Cela semble paradoxal, car désactiver l’accélération matérielle est habituellement le choix le plus stable. Pour ce bug, c’est l’inverse, et la chaîne de causes ci-dessous explique pourquoi : ce réglage détermine si le plantage d’un processus GPU ne coûte qu’un niveau de repli ou toute l’application.

**2. Utiliser avec parcimonie le navigateur intégré.** Les pages dans la zone navigateur/aperçu de l’application déclenchent le plantage. Fermer cette zone après usage, au lieu de laisser des onglets ouverts, réduit considérablement la fréquence des plantages ; cette corrélation est étayée à plusieurs reprises par des chiffres dans le fil communautaire.

**3. Facultatif : désactiver WebGPU.** Un démarrage avec `--disable-features=WebGPU` élimine complètement le déclencheur le plus fréquent. Pour une application Store, le chemin d’installation change à chaque mise à jour ; voici donc un lanceur qui le résout à nouveau à chaque démarrage :

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

La limite : cela ne fonctionne que si l’application est également lancée via ce lanceur. La mesure 1 s’applique à chaque démarrage.

Une « Réparation » ou une réinstallation de l’application ne résout d’ailleurs pas le problème ; elle ne traite que le symptôme consécutif (voir plus bas). Les mises à jour de pilotes graphiques sont également vaines.

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

Sur les systèmes concernés, vous y trouverez des entrées Event 3033 dont les horodatages correspondent à la seconde près aux heures de plantage, avec ce message :

```text
Code Integrity determined that a process
(...\WindowsApps\Claude_..._x64__pzs8sxrjxfjjc\app\claude.exe)
attempted to load ...\app\vk_swiftshader.dll that did not meet
the Microsoft signing level requirements.
```

Sur le système étudié ici, sept plantages sur sept, sur trois semaines, correspondaient à la seconde près à un tel événement, y compris un plantage de contrôle déclenché délibérément.

## La chaîne complète des causes

Le plantage est le dernier maillon d’une chaîne de quatre éléments établie par deux analyses : la piste Code Integrity issue de l’issue communautaire [#81698](https://github.com/anthropics/claude-code/issues/81698) et une analyse personnelle de minidump ([#89250](https://github.com/anthropics/claude-code/issues/89250)).

**Maillon 1 : une page dans le navigateur intégré nécessite un rendu logiciel.** Le déclencheur typique est un appel WebGPU (`navigator.gpu.requestAdapter()`), reconnaissable dans le journal de la fenêtre à cet avertissement juste avant le plantage :

```text
[warn] The powerPreference option is currently ignored
       when calling requestAdapter() on Windows.
```

Lorsque l’application s’exécute sans accélération matérielle, le chemin passe obligatoirement par l’implémentation logicielle Vulkan SwiftShader : le processus GPU tente de charger la `vk_swiftshader.dll` fournie.

**Maillon 2 : Windows Code Integrity bloque la propre DLL de l’application.** Le processus GPU s’exécute avec la stratégie de renforcement « MicrosoftSignedOnly » (vérifiable avec `Get-ProcessMitigation`). Pour qu’une application Store puisse charger ses propres DLL signées par le fabricant, le package MSIX doit fournir un catalogue de signatures `AppxMetadata\CodeIntegrity.cat`. Or, ce fichier manque précisément dans le package distribué, comme l’ont démontré des membres de la communauté en inspectant le fichier MSIX. Conséquence : la vérification de signature échoue, Windows enregistre l’Event 3033 et arrête brutalement le processus GPU. Le code de sortie `0x060C201E` est une erreur d’intégrité AppX du chargeur Windows, et non un code Chromium ; c’est pourquoi il n’apparaît dans aucune source Chromium, et pourquoi le processus GPU ne laisse aucun crash dump : il n’y a aucune exception dont un dump pourrait être créé.

**Maillon 3 : la chaîne de repli de Chromium s’épuise.** Lorsque le processus GPU plante, Chromium recule d’un niveau de rendu : GL matériel, puis GL logiciel, puis compositeur d’affichage pur. L’arrêt automatique intégré n’intervient que lorsqu’il ne reste plus aucun niveau. Dans le code source de la version intégrée (Chromium 148.0.7778.280 dans Electron 42.9.2), il est écrit textuellement ainsi :

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

**Maillon 4 : le processus principal s’arrête délibérément.** Cet `LOG(FATAL)` est le moment où « l’application plante ». Il est attesté par un minidump du processus principal : `EXCEPTION_BREAKPOINT` (un `int3` intentionnel, pas une erreur de pilote), aucune DLL de pilote graphique dans le processus, et en clair dans la mémoire :

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

Le fait que ce dump existe a été la partie la plus difficile de l’analyse : l’intégration Sentry de l’application consomme les dumps Crashpad au démarrage suivant de l’application, les envoie à la télémétrie du fabricant et les supprime localement. Le dossier Crashpad est donc toujours vide pour l’utilisateur. Un observateur indépendant de l’arbre de processus de l’application apporte une solution (démarré via WMI afin que le plantage de l’application ne l’arrête pas aussi) : il examine la base de données Crashpad toutes les 200 millisecondes à la recherche de `*.dmp` et copie immédiatement les résultats ailleurs avant leur suppression. Le package Python `minidump` prend en charge l’analyse, sans WinDbg.

## Pourquoi « désactiver l’accélération matérielle » aggrave tout

La chaîne explique aussi le constat le plus contre-intuitif. Désactiver l’accélération matérielle a ici simultanément deux effets fatals. Premièrement, cela force le chemin SwiftShader, donc précisément la tentative de chargement de DLL que Code Integrity bloque ; avec l’accélération matérielle active, `vk_swiftshader.dll` n’est en revanche presque jamais nécessaire. Deuxièmement, le processus GPU démarre alors déjà au bas de la chaîne de repli : un seul plantage suffit et le maillon 4 intervient. Cela explique aussi l’observation du fil communautaire selon laquelle un blocage Code Integrity reste parfois sans conséquence et ferme parfois l’application : cela dépend du nombre de niveaux de repli restant au processus du navigateur.

Particulièrement malheureux : l’application dispose d’une désactivation automatique de l’accélération matérielle après des problèmes (`isHardwareAccelerationAutoDisabled`). Conçue comme mesure de stabilité, elle amène les systèmes concernés précisément dans la configuration où le plantage suivant coûte toute l’application.

## Le symptôme consécutif : la boucle de réparation

L’échec de Code Integrity a un effet secondaire que de nombreuses personnes concernées prennent pour un problème distinct : après l’incident, Windows classe parfois le package de l’application comme « Modified, NeedsRemediation ». L’application ne démarre alors plus du tout jusqu’à ce qu’elle soit réinitialisée via Paramètres → Applications → Claude → Options avancées → « Réparer ». Les personnes qui doivent donc « réparer l’application constamment » observent le même problème fondamental, simplement un maillon plus loin : la réparation corrige l’état du package, pas la cause ; le plantage suivant survient lors de la prochaine tentative de chargement de DLL bloquée.

## État des signalements

La cause liée au packaging a été signalée sous [#81341](https://github.com/anthropics/claude-code/issues/81341), le fil récapitulatif contenant les preuves de la communauté est [#81698](https://github.com/anthropics/claude-code/issues/81698), et l’analyse de minidump expliquant la chaîne de repli est [#89250](https://github.com/anthropics/claude-code/issues/89250). La véritable correction, un catalogue de signatures complet dans le package MSIX, relève d’Anthropic. D’ici là : activez l’accélération matérielle, fermez avec discipline la zone navigateur et, si nécessaire, désactivez WebGPU via le flag. Sur le système étudié ici, l’application ne plante plus depuis la mise en œuvre de la mesure 1.

## Sources

1.  [GitHub Issue #81698: GPU process crash kills entire app](https://github.com/anthropics/claude-code/issues/81698): Le fil récapitulatif avec les preuves de la communauté sur la chaîne Code Integrity, les données couvrant plusieurs fabricants et la corrélation avec le panneau navigateur.

2.  [GitHub Issue #81341: CIG + vk_swiftshader.dll kills GPU process](https://github.com/anthropics/claude-code/issues/81341): La cause liée au packaging ; catalogue CodeIntegrity manquant dans le MSIX.

3.  [GitHub Issue #89250: Analyse du minidump de l’arrêt de l’application](https://github.com/anthropics/claude-code/issues/89250): La seconde moitié de la chaîne décrite ici, avec la méthode de capture des dumps et des propositions de correction.

4.  [Code source Chromium : gpu_data_manager_impl_private.cc (tag 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): La fonction IntentionallyCrashBrowserForUnusableGpuProcess et la logique de repli.

5.  [Documentation Electron : child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): L’événement permettant à une application Electron d’observer les plantages du processus GPU et de prendre ses propres contre-mesures.

6.  [Package Python minidump](https://pypi.org/project/minidump/): Outil d’analyse des dumps (enregistrement d’exception, liste de modules, chaînes mémoire).

7.  [webgpureport.org](https://webgpureport.org/): Page de diagnostic WebGPU ; a servi de déclencheur minimal pour le plantage de contrôle et le test comparatif dans le Chromium actuel.
