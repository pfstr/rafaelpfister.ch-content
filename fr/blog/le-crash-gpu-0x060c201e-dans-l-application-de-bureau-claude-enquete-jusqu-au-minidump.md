---
title: "Claude Desktop plante sans cesse : « GPU process gone » avec le code de sortie 101457950, cause et solution"
navTitle: "Plantage de Claude Desktop"
description: "Sous Windows, l’application Claude Desktop se ferme complètement avec « GPU process gone: exitCode 101457950 » (0x060C201E), souvent suivi de la boîte de dialogue de réparation de l’application Store. La chaîne complète des causes : Code Integrity bloque vk_swiftshader.dll, la chaîne de repli de Chromium est épuisée, puis l’arrêt automatique intégré ferme l’application. Avec une solution permanente (passage à l’installation classique sans MSIX), un autodiagnostic via le journal des événements et une analyse jusqu’au minidump."
date: "2026-08-25"
kategorie: "Claude"
timeToRead: "10 min de lecture"
themen:
  - claude
slug: "le-crash-gpu-0x060c201e-dans-l-application-de-bureau-claude-enquete-jusqu-au-minidump"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
translationSourceHash: 9bd177698f756b588c97f11ea4c73ea48efcfb96c8dbe1435f5ff9fc11dffec0
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:29:16.718Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/le-crash-gpu-0x060c201e-dans-l-application-de-bureau-claude-enquete-jusqu-au-minidump
---

L’application Claude Desktop sous Windows se ferme sans message d’erreur, toutes les sessions Claude Code en cours disparaissent, et l’application ne redémarre parfois qu’après une « Réparation » dans les paramètres Windows. Cette ligne apparaît alors dans le journal de l’application :

```text
GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
```

101457950 s’écrit en hexadécimal `0x060C201E`. Si vous trouvez cette signature dans votre journal, vous êtes au bon endroit : cet article documente la chaîne complète des causes de ce plantage, les mesures immédiates qui permettent à l’application de redevenir stable et l’autodiagnostic permettant de confirmer le constat sur votre propre système en deux minutes. Les installations MSIX (depuis le Microsoft Store ou via l’installation MSIX) sont concernées chez tous les fabricants de GPU, des iGPU Intel à NVIDIA et AMD ; le matériel n’en est pas la cause. L’installation classique sans MSIX n’est pas concernée, et c’est précisément la solution.

## La solution en bref : passer à l’installation classique

L’erreur elle-même se situe dans le paquet d’installation MSIX et ne peut être corrigée que par Anthropic (l’issue [#81341](https://github.com/anthropics/claude-code/issues/81341) reste ouverte au 25.10.2026 ; début septembre 2026, des signalements dans l’issue ont également confirmé l’erreur dans la version 1.40609.1). La même application existe toutefois aussi sous forme d’installation classique sans MSIX, qui n’est pas soumise au contrôle de signature AppX mettant fin au processus GPU. Ce changement est donc la seule mesure qui élimine entièrement le plantage ; il est confirmé tant dans l’issue [#81341](https://github.com/anthropics/claude-code/issues/81341) que sur le système étudié ici. Les fonctionnalités sont identiques, et le flux de mises à jour fournit les mêmes versions pour les deux variantes.

**Étape 1 : télécharger et exécuter l’installateur classique.** Le téléchargement sur [claude.com/download](https://claude.com/download) fournit un installateur Squirrel qui installe l’application dans `%LOCALAPPDATA%\AnthropicClaude` (aucun droit administrateur requis). En ligne de commande :

```powershell
curl.exe -L -o "$env:USERPROFILE\Downloads\Claude-Setup-x64.exe" `
  "https://storage.googleapis.com/osprey-downloads-c02f6a0d-347c-492b-a752-3e0651722e97/nest-win-x64/Claude-Setup-x64.exe"
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-L` | suit les redirections HTTP jusqu’au fichier proprement dit |
| `-o <pfad>` | fichier cible ; ici, le dossier Téléchargements |
| `<url>` | source officielle de l’installateur ; identique à la cible de la redirection de téléchargement de claude.ai |

</details>

Après le téléchargement, vérifiez la signature (`Get-AuthenticodeSignature`, attendu : `Valid`, émetteur « Anthropic, PBC ») et lancez le fichier. L’installateur dépose d’abord une version de base plus ancienne ; le mécanisme de mise à jour la met à niveau, soit automatiquement au premier démarrage, soit immédiatement avec :

```powershell
& "$env:LOCALAPPDATA\AnthropicClaude\Update.exe" `
  --update https://downloads.claude.ai/releases/win32/x64
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `--update <url>` | télécharge la version la plus récente depuis le flux de versions indiqué et l’installe dans un nouveau répertoire `app-<version>` |

</details>

**Étape 2 : reprendre la configuration.** La version MSIX conserve l’identification, la configuration des serveurs MCP et les réglages dans son conteneur virtualisé ; l’application classique lit `%APPDATA%\Claude`. Copiez-les une fois (fermez auparavant l’application MSIX ; les deux variantes ne peuvent de toute façon pas s’exécuter simultanément en raison d’un verrou commun d’instance unique) :

```powershell
robocopy "$env:LOCALAPPDATA\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude" `
  "$env:APPDATA\Claude" /E /XD Cache "Code Cache" GPUCache claude-code Crashpad logs sentry
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `<quelle>` | dossier de configuration dans l’AppData virtualisé du paquet MSIX |
| `<ziel>` | dossier de configuration de l’installation classique |
| `/E` | copie tous les sous-répertoires, y compris les vides |
| `/XD <namen>` | ignore les répertoires indiqués ; ici les caches et données d’exécution que la nouvelle application recrée elle-même |

</details>

Les historiques de discussion ne sont pas perdus : ils sont stockés dans le compte claude.ai ou, pour les sessions Claude Code, sous `%USERPROFILE%\.claude`, et ne dépendent pas de l’installation de l’application.

**Étape 3 : supprimer le paquet MSIX.** Sans cela, l’ancienne variante qui plante continuera à démarrer depuis les anciens raccourcis :

```powershell
Get-AppxPackage Claude | Remove-AppxPackage
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Claude` | argument positionnel Name de `Get-AppxPackage`: filtre les paquets AppX/MSIX installés selon le nom de paquet (jokers autorisés) |
| `Remove-AppxPackage` | supprime pour le compte utilisateur actuel le paquet transmis via le pipeline |

</details>

L’entrée du menu Démarrer « Anthropic → Claude » correspond ensuite à l’installation classique ; tout épinglage dans la barre des tâches doit être recréé.

## Si vous devez conserver le paquet MSIX

Sans changement, il ne reste que des mesures qui réduisent la fréquence des plantages sans en éliminer la cause :

**Utilisez le navigateur intégré avec parcimonie.** Les pages affichées dans la zone de navigateur ou d’aperçu de l’application déclenchent le plantage. Fermer la zone après utilisation au lieu de laisser les onglets ouverts réduit nettement la fréquence des plantages ; ce lien est étayé à plusieurs reprises par des chiffres dans le fil communautaire.

**Désactivez WebGPU.** Un démarrage avec `--disable-features=WebGPU` empêche le déclencheur le plus fréquent. Dans le cas d’un paquet MSIX, le chemin d’installation change à chaque mise à jour ; utilisez donc un lanceur qui le résout à nouveau à chaque démarrage :

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `for /f "delims="` | traite la sortie de commande ligne par ligne ; `delims=` vide reprend la ligne entière, y compris les espaces dans le chemin, dans `%%i` |
| `-NoProfile` | lance PowerShell sans scripts de profil, pour un démarrage rapide et reproductible |
| `-Command` | exécute l’expression indiquée ; `(Get-AppxPackage Claude).InstallLocation` fournit le chemin d’installation actuel du paquet |
| `start ""` | lance le programme indépendamment de la fenêtre batch ; les guillemets vides correspondent au titre de fenêtre, ici vide |
| `--disable-features=WebGPU` | commutateur Chromium : désactive la fonctionnalité indiquée, ici l’API WebGPU |

</details>

Cela ne fonctionne que si l’application est elle aussi démarrée par ce lanceur.

La première version de cet article recommandait en priorité d’activer l’accélération matérielle via `isHardwareAccelerationDisabled: false` dans `config.json`. Cette recommandation est obsolète : dans les versions actuelles (1.37937.x), le drapeau n’existe plus, l’application démarre par défaut avec l’accélération matérielle active et elle plante malgré tout (voir les détails dans l’ajout ci-dessous).

Une « Réparation » ou une réinstallation du paquet MSIX ne résout d’ailleurs pas le problème ; cela ne traite que le symptôme consécutif (voir plus bas). Les mises à jour de pilotes graphiques sont également des efforts vains.

## Autodiagnostic : confirmer le constat sur son propre système

Deux vérifications suffisent. Premièrement, la signature du plantage dans le journal de l’application :

```powershell
Select-String -Path "$env:LOCALAPPDATA\Claude\Logs\main.log" `
  -Pattern 'GPU process gone'
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Path` | fichier à parcourir, ici le journal principal de l’application |
| `-Pattern` | motif de recherche, expression régulière ; affiche toutes les lignes contenant la signature du plantage |

</details>

Deuxièmement, et c’est la preuve décisive, le journal CodeIntegrity de Windows :

```powershell
Get-WinEvent -FilterHashtable @{
  LogName='Microsoft-Windows-CodeIntegrity/Operational'; Id=3033
} -MaxEvents 30 | Where-Object { $_.Message -match 'claude' } |
  Select-Object TimeCreated, Message
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-FilterHashtable` | filtre dès la récupération : `LogName` désigne le journal des événements, `Id` l’ID d’événement 3033, blocage Code Integrity |
| `-MaxEvents 30` | limite la requête aux 30 résultats les plus récents |
| `Where-Object { … -match 'claude' }` | conserve uniquement les événements dont le texte du message contient le chemin de l’application |
| `Select-Object TimeCreated, Message` | réduit la sortie à l’horodatage et au message pour comparaison avec les heures de plantage |

</details>

Sur les systèmes affectés, vous y trouverez des entrées Event 3033 dont l’horodatage coïncide à la seconde près avec les heures des plantages, avec ce message :

```text
Code Integrity determined that a process
(...\WindowsApps\Claude_..._x64__pzs8sxrjxfjjc\app\claude.exe)
attempted to load ...\app\vk_swiftshader.dll that did not meet
the Microsoft signing level requirements.
```

Sur le système étudié ici, sept plantages sur sept, sur trois semaines, ont coïncidé à la seconde près avec un tel événement, y compris un plantage de contrôle déclenché délibérément.

## La chaîne complète des causes

Le plantage est le dernier maillon d’une chaîne de quatre éléments mise en évidence par deux analyses : la piste Code Integrity issue de l’issue communautaire [#81698](https://github.com/anthropics/claude-code/issues/81698) et une analyse personnelle de minidump ([#89250](https://github.com/anthropics/claude-code/issues/89250)).

**Maillon 1 : une page du navigateur intégré nécessite un rendu logiciel.** Le déclencheur typique est un appel WebGPU (`navigator.gpu.requestAdapter()`), identifiable dans le journal de fenêtre par cet avertissement juste avant le plantage :

```text
[warn] The powerPreference option is currently ignored
       when calling requestAdapter() on Windows.
```

Lorsque l’application fonctionne sans accélération matérielle, le chemin passe nécessairement par l’implémentation logicielle Vulkan SwiftShader : le processus GPU tente de charger la bibliothèque `vk_swiftshader.dll` fournie avec l’application.

**Maillon 2 : Windows Code Integrity bloque la propre DLL de l’application.** Le processus GPU s’exécute avec la stratégie de durcissement « MicrosoftSignedOnly » (vérifiable avec `Get-ProcessMitigation`). Pour qu’une application Store puisse charger ses propres DLL signées par le fabricant, le paquet MSIX doit fournir un catalogue de signatures `AppxMetadata\CodeIntegrity.cat`. Ce fichier manque précisément dans le paquet distribué, comme des membres de la communauté l’ont démontré en inspectant le fichier MSIX. Conséquence : le contrôle de signature échoue, Windows consigne l’événement 3033 et met brutalement fin au processus GPU. Le code de sortie `0x060C201E` est une erreur d’intégrité AppX du chargeur Windows, et non un code Chromium ; c’est pourquoi il n’apparaît dans aucune source Chromium et pourquoi le processus GPU ne laisse aucun dump de crash : il n’y a pas d’exception à dumper.

**Maillon 3 : la chaîne de repli de Chromium est épuisée.** Lorsque le processus GPU plante, Chromium rétrograde d’un niveau de rendu : GL matériel, puis GL logiciel, puis compositeur d’affichage pur. L’arrêt automatique intégré n’intervient que lorsqu’il ne reste plus aucun niveau. Le code source de la version embarquée (Chromium 148.0.7778.280 dans Electron 42.9.2) l’indique littéralement ainsi :

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

**Maillon 4 : le processus principal se termine volontairement.** Cet `LOG(FATAL)` est le moment où « l’application plante ». Cela est démontré par un minidump du processus principal : `EXCEPTION_BREAKPOINT` (un `int3` intentionnel, et non une erreur de pilote), aucune DLL de pilote graphique dans le processus, et en clair en mémoire :

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

L’existence même de ce dump a été la partie la plus difficile de l’analyse : l’intégration Sentry de l’application consomme les dumps Crashpad au démarrage suivant de l’application, les envoie à la télémétrie du fabricant et les supprime localement. Le dossier Crashpad est donc toujours vide pour l’utilisateur. Un observateur indépendant de l’arborescence des processus de l’application apporte une solution (démarré via WMI afin que le plantage de l’application ne l’arrête pas lui aussi) : il parcourt la base de données Crashpad toutes les 200 millisecondes à la recherche de `*.dmp` et copie immédiatement ailleurs les fichiers trouvés avant leur suppression. L’analyse est effectuée par le paquet Python `minidump`, sans WinDbg.

## Pourquoi « désactiver l’accélération matérielle » aggrave tout

La chaîne explique aussi le constat le plus contre-intuitif. L’accélération matérielle désactivée a ici simultanément deux effets fatals. Premièrement, elle impose le chemin SwiftShader, donc précisément la tentative de chargement de DLL que Code Integrity bloque ; avec l’accélération matérielle active, `vk_swiftshader.dll` n’est en revanche presque jamais nécessaire. Deuxièmement, le processus GPU démarre alors déjà à l’extrémité inférieure de la chaîne de repli : un seul plantage suffit et le maillon 4 intervient. Cela explique aussi l’observation du fil communautaire selon laquelle un blocage Code Integrity peut parfois rester sans conséquence et parfois fermer l’application : cela dépend du nombre de niveaux de repli encore disponibles pour le processus de navigateur.

Particulièrement malheureux : l’application connaissait une désactivation automatique de l’accélération matérielle après des problèmes (`isHardwareAccelerationAutoDisabled`). Conçue comme mesure de stabilité, elle plaçait les systèmes affectés précisément dans la configuration où le plantage suivant coûte toute l’application.

## Ajout du 27.08.2026 : l’accélération matérielle seule ne suffit pas

La première version de cet article recommandait l’accélération matérielle active comme mesure immédiate la plus efficace, et pendant deux jours, l’application est effectivement restée exempte de plantage. Puis la mise à jour automatique vers 1.37937.3 est arrivée, avec elle trois plantages en un après-midi, chacun avec le connu événement 3033 relatif à `vk_swiftshader.dll`. Deux constats en découlent :

Premièrement, le catalogue de signatures manquant est aussi absent du paquet MSIX actuel ; le problème fondamental reste inchangé dans 1.37937.3.

Deuxièmement, l’accélération matérielle active ne protège que statistiquement : elle allonge la chaîne de repli, mais n’empêche pas Chromium de la parcourir jusqu’au niveau SwiftShader sous charge ou après une erreur du processus GPU matériel. Dès que cela se produit, Code Integrity bloque la DLL et la chaîne peut tout de même s’épuiser. De plus, les drapeaux de configuration `isHardwareAccelerationDisabled`/`isHardwareAccelerationAutoDisabled` ont disparu de `config.json` dans 1.37937.x ; le réglage ne peut plus y être fixé.

La seule solution fiable restait donc le passage à l’installation classique décrit plus haut. Depuis ce changement sur le système étudié ici : même version de l’application, utilisation identique y compris de la zone de navigateur, pas un seul événement 3033 ni plantage.

## Le symptôme consécutif : la boucle de réparation

L’échec de Code Integrity a un effet secondaire que beaucoup de personnes concernées considèrent comme un problème distinct : après l’incident, Windows classe parfois le paquet de l’application comme « Modified, NeedsRemediation ». L’application ne démarre alors plus du tout jusqu’à sa réinitialisation par Paramètres → Applications → Claude → Options avancées → « Réparer ». Quiconque doit donc « constamment réparer l’application » voit le même problème fondamental, un maillon plus loin : la réparation corrige l’état du paquet, pas la cause ; le plantage suivant survient lors de la prochaine tentative bloquée de chargement de DLL.

## État des signalements

La cause liée à l’empaquetage est signalée sous [#81341](https://github.com/anthropics/claude-code/issues/81341), le fil de discussion regroupant les éléments communautaires est [#81698](https://github.com/anthropics/claude-code/issues/81698), l’analyse de minidump expliquant la chaîne de repli est [#89250](https://github.com/anthropics/claude-code/issues/89250), et un autre rapport détaillé, incluant la boucle de réparation, est [#80444](https://github.com/anthropics/claude-code/issues/80444). Le correctif proprement dit, un catalogue complet de signatures dans le paquet MSIX, relève d’Anthropic. Au 25.10.2026, #81341 reste ouvert et des signalements du début septembre 2026 confirment également l’erreur dans la version 1.40609.1. #80444 a été fermé le 15 septembre 2026 sans indication de version corrigée, et #89250 automatiquement le 20 octobre 2026 pour inactivité ; aucune des deux issues ne mentionne de correctif. D’ici là : passez à l’installation classique ; toute personne devant conserver le paquet MSIX ferme rigoureusement la zone de navigateur et désactive si nécessaire WebGPU via un drapeau. Sur le système étudié ici, l’application ne plante plus depuis le passage à l’installation classique, sans un seul autre événement 3033.

## Historique des modifications

- **25 octobre 2026 :** Ajouté : l’issue #81341 reste ouverte au 25.10.2026 et l’erreur est aussi confirmée dans la version 1.40609.1. Les issues #80444 et #89250 ont été fermées sans mention de correctif.

## Sources

1.  [Issue GitHub #81698 : GPU process crash kills entire app](https://github.com/anthropics/claude-code/issues/81698) : le fil regroupant les éléments communautaires relatifs à la chaîne Code Integrity, aux données couvrant différents fabricants et à la corrélation avec le panneau de navigateur.

2.  [Issue GitHub #81341 : CIG + vk_swiftshader.dll kills GPU process](https://github.com/anthropics/claude-code/issues/81341) : la cause liée à l’empaquetage ; catalogue CodeIntegrity manquant dans le MSIX.

3.  [Issue GitHub #89250 : analyse du minidump de l’arrêt de l’application](https://github.com/anthropics/claude-code/issues/89250) : la seconde moitié de la chaîne décrite ici, avec la méthode de capture de dump et des propositions de correctifs.

4.  [Issue GitHub #80444 : crash GPU avec analyse forensique et boucle de réparation](https://github.com/anthropics/claude-code/issues/80444) : rapport individuel détaillé avec chronologies, analyse du journal des événements et constat que chaque plantage place le paquet dans l’état « Modified ».

5.  [Claude Desktop : page de téléchargement officielle](https://claude.com/download) : source de l’installateur Windows classique, x64 et ARM64.

6.  [Code source Chromium : gpu_data_manager_impl_private.cc (tag 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc) : la fonction IntentionallyCrashBrowserForUnusableGpuProcess et la logique de repli.

7.  [Documentation Electron : child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone) : l’événement permettant à une application Electron d’observer les plantages du processus GPU et de prendre ses propres contre-mesures.

8.  [Paquet Python minidump](https://pypi.org/project/minidump/) : outil d’analyse de dump, enregistrement d’exception, liste de modules et chaînes mémoire.

9.  [webgpureport.org](https://webgpureport.org/) : page de diagnostic WebGPU ; a servi de déclencheur minimal pour le plantage de contrôle et pour le test comparatif dans le Chromium actuel.
