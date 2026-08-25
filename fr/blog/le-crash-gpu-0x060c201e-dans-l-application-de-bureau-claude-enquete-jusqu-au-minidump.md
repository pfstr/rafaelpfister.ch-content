---
title: "Le crash GPU 0x060C201E dans l’application de bureau Claude : enquête jusqu’au minidump"
navTitle: "Crash GPU 0x060C201E"
description: "L’application de bureau Claude se ferme de manière reproductible avec « GPU process gone ». Tout semble d’abord indiquer un bug du pilote AMD, puis des expériences personnelles réfutent cette hypothèse, et finalement un minidump intercepté révèle la véritable cause : l’arrêt automatique intégré de Chromium, « GPU process isn't usable. Goodbye. »."
date: "2026-08-25"
kategorie: "Claude"
timeToRead: "12 min de lecture"
themen:
  - claude
slug: "le-crash-gpu-0x060c201e-dans-l-application-de-bureau-claude-enquete-jusqu-au-minidump"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
url: https://rafaelpfister.ch/fr/blog/le-crash-gpu-0x060c201e-dans-l-application-de-bureau-claude-enquete-jusqu-au-minidump
translationSourceHash: 6bd2b58fe661a5639010e16b417412ca9e85f687bae94531890c8fefaef4050d
translationModel: gpt-5.6-terra
translatedAt: 2026-08-25T04:04:59.328Z
translationReview: automatic
---

Depuis la fin juillet, mon application de bureau Claude se ferme plusieurs fois par jour sous Windows. Pas de boîte de dialogue, pas de fenêtre d’erreur : l’application a simplement disparu, avec toutes les sessions Claude Code en cours. Plus de 25 fois à ce jour. Il est temps de ne plus redémarrer, mais de voir où l’erreur se produit réellement. Pour l’essentiel : le principal suspect de la première heure s’avère innocent, et la véritable cause apparaît noir sur blanc à la fin dans un minidump que l’application ne voulait en fait pas du tout livrer.

## La piste dans le journal

L’application enregistre ses journaux sous `%LOCALAPPDATA%\Claude\Logs`, les générations plus anciennes et la configuration se trouvent dans le chemin de Store virtualisé `%LOCALAPPDATA%\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude`. Dans `main.log`, on trouve exactement la même chose avant chaque crash :

```text
16:01:38 [info] GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
16:03:34 [info] Starting app { appVersion: '1.34493.1', ... }
```

101457950 correspond en hexadécimal à `0x060C201E`. Retenez ce nombre : c’est la signature du bug. Le journal de la fenêtre en fournit le déclencheur : juste avant chaque crash, une page dans le navigateur intégré de l’application demande un adaptateur WebGPU.

```text
16:01:38 [warn] The powerPreference option is currently ignored
                when calling requestAdapter() on Windows.
16:01:38 [warn] A valid external Instance reference no longer exists.
14:59:15 [warn] WebGL: CONTEXT_LOST_WEBGL: loseContext: context lost
```

Donc : `navigator.gpu.requestAdapter()` passe, dans le processus GPU de Chromium, à l’énumération des adaptateurs de Dawn, le processus GPU plante et, au lieu de le redémarrer, l’application entière se ferme.

## Suspect n° 1 : le pilote graphique

La machine dispose d’une Radeon RX 7900 XT avec Adrenalin 32.0.31035.1003, et l’application embarque Electron 42.9.2 avec Chromium 148. L’explication commode est sur la table : ancien code Dawn rencontre un pilote RDNA3, le pilote plante, affaire classée. Commode, plausible et, comme on le verra : fausse. Mais procédons dans l’ordre, car on ne peut réfuter qu’avec des expériences.

Deux éléments se sont d’emblée révélés être de fausses pistes. L’iGPU désactivé dans le Gestionnaire de périphériques (statut « Error ») correspond simplement au code 22, une désactivation volontaire. Et l’application avait depuis longtemps désactivé l’accélération matérielle (`isHardwareAccelerationDisabled: true` dans config.json), ce qui n’a nullement impressionné les crashes. La raison pour laquelle ce réglage aggrave même le problème ne devient visible qu’à la toute fin.

## Expérience 1 : contre-épreuve dans Chromium actuel

Même charge, même machine, navigateur actuel : webgpureport.org initialise entièrement WebGPU dans Chromium 151, adaptateur `amd / rdna-3`, création du périphérique comprise, sans aucune anomalie. Le pilote actuel avec Dawn actuel est donc sain.

## Expérience 2 : Electron 42.9.2 standard, chemin matériel

Si Electron 42 ne s’entend pas avec ce pilote, il doit être possible de le reproduire en 20 lignes. Donc : exactement la même version d’Electron que dans l’application, sous forme de simple projet de test, une fenêtre, une page, un `requestAdapter()` :

```js
const { app, BrowserWindow, crashReporter } = require('electron');
crashReporter.start({ submitURL: '', uploadToServer: false });
app.on('child-process-gone', (e, d) =>
  console.log('GONE: ' + JSON.stringify(d)));
app.whenReady().then(() => {
  const win = new BrowserWindow({ show: false });
  win.loadFile('index.html'); // ruft requestAdapter() auf
});
```

Résultat avec accélération matérielle : `adapter ok (amd/rdna-3), device ok`. Aucun crash. Le chemin D3D12 d’Electron 42 sur ce pilote fonctionne parfaitement. L’hypothèse selon laquelle « l’ancien code Dawn ne supporte pas le pilote RDNA3 » est ainsi réfutée.

## Expérience 3 : Electron 42.9.2 standard, chemin logiciel comme dans l’application

L’application fonctionne toutefois sans accélération matérielle. Donc la même expérience avec `app.disableHardwareAcceleration()`, en ajoutant un contexte WebGL actif (qui passe par SwiftShader en mode logiciel) et `powerPreference: 'high-performance'` lors de la demande d’adaptateur, afin de reproduire exactement le déroulement des journaux de l’application :

```text
[renderer] webgl context: WebKit WebGL
[renderer] The powerPreference option is currently ignored
           when calling requestAdapter() on Windows.
[renderer] No available adapters.
[renderer] RESULT: adapter=null
TIMEOUT: no crash after 25s
```

Même avertissement powerPreference que dans le journal de l’application, même chemin de code jusqu’à l’énumération des adaptateurs, puis la réponse correcte : aucun adaptateur disponible, rejet propre, processus vivant. Electron 42.9.2 standard ne plante tout simplement pas sur cette machine, quel que soit le chemin.

## Expérience 4 : autre matériel, même signature

Avant de continuer à spéculer, il vaut la peine de consulter le suivi des issues, et le constat y est clair : le crash identique avec le même code de sortie 0x060C201E a été signalé plusieurs fois, notamment sur un GPU portable NVIDIA RTX 5080. Dans leur journal d’événements système : aucun événement TDR, aucune réinitialisation du pilote. Le pilote, quel que soit son fabricant, n’est pas la cause. La cause du crash se situe dans le processus GPU de l’application elle-même ou, comme on va le voir, dans la réaction de l’application à son crash.

## Expérience 5 : récupérer le minidump que l’application supprime

Jusqu’ici, il manquait la pièce décisive : un dump de crash. Le dossier Crashpad de l’application était vide après chaque crash, ce qui semblait d’abord indiquer que le signalement des crashes était désactivé. La liste des processus raconte autre chose : un processus `crashpad-handler` est en cours d’exécution, sa ligne de commande pointe vers la base de données du profil Roaming et vers une URL d’envoi fictive. C’est le schéma habituel de l’intégration Sentry dans les applications Electron : Crashpad écrit le dump localement, la bibliothèque Sentry le consomme au démarrage suivant de l’application, l’envoie à la télémétrie du fabricant et le supprime localement. Les dumps existent donc, mais pas pour l’utilisateur.

La solution est peu spectaculaire : un observateur indépendant de l’arbre de processus de l’application (lancé via WMI afin que le crash de l’application ne l’emporte pas), qui parcourt toutes les 200 millisecondes la base de données Crashpad à la recherche de `*.dmp` et copie immédiatement ailleurs toute découverte. Ensuite, déclencher le crash délibérément : ouvrir webgpureport.org dans le navigateur intégré de l’application. Quelques secondes plus tard, un minidump de 35 Mo se trouve dans le dossier de sauvegarde, et Sentry tente en vain de le supprimer au démarrage suivant de l’application.

## Le minidump : aucun pilote à l’horizon

L’analyse avec le paquet Python `minidump` fournit trois résultats qui changent complètement la perspective :

```text
Exception: EXCEPTION_BREAKPOINT (0x80000003)
Adresse:   Claude.exe+0x5e8a6c9
Prozess:   PID 27660
```

Premièrement : le processus dumpé n’est pas le processus GPU, mais le **processus principal** de l’application (le PID apparaît dans les journaux de l’application sous `electron_main`). Deuxièmement : l’exception est un breakpoint, donc un `int3` exécuté intentionnellement. C’est ainsi que Chromium s’arrête lui-même lorsqu’un `CHECK()` ou `LOG(FATAL)` se déclenche ; une erreur de pilote apparaîtrait comme une violation d’accès. Troisièmement : aucune DLL de pilote graphique n’est chargée dans la liste des modules du processus.

Et dans la mémoire du dump, le message de journal fatal apparaît en clair :

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

## La résolution : l’arrêt automatique intégré de Chromium

Cette ligne n’est pas un dysfonctionnement, elle est intentionnelle. Dans le code source Chromium de la version exacte embarquée (148.0.7778.280), on trouve dans `gpu_data_manager_impl_private.cc` :

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

Elle est appelée par `FallBackToNextGpuMode()` : si le processus GPU plante, Chromium recule d’un niveau (GL matériel → GL logiciel → compositeur d’affichage seul). Si la liste des modes de repli est vide, Chromium arrête volontairement le processus navigateur, car sans processus GPU fonctionnel, il ne peut même plus coordonner le rendu logiciel.

Cela explique également pourquoi l’application est bien plus durement touchée qu’un navigateur normal : elle démarre avec l’accélération matérielle désactivée, donc déjà tout en bas de la chaîne de repli. Si une page du navigateur intégré demande alors WebGPU et que le processus GPU logiciel plante, Chromium n’a plus aucun niveau vers lequel se replier. L’étape suivante est « Goodbye ». Dans un Chrome normal avec accélération matérielle active, le même crash coûte un niveau de repli, et le navigateur continue de fonctionner.

Particulièrement malheureux : la configuration de l’application connaît un champ `isHardwareAccelerationAutoDisabled`, l’application désactive donc apparemment elle-même l’accélération matérielle après des problèmes. Conçue comme mesure anti-crash, cette action raccourcit précisément la chaîne de repli et rend l’arrêt automatique fatal plus probable au lieu de plus rare. Un mécanisme de protection et un coupe-circuit qui s’arment mutuellement.

## Ce qui reste du code de sortie

Reste le processus enfant GPU lui-même, qui déclenche à chaque fois la séquence. Il ne laisse aucun rapport de crash propre, bien que le gestionnaire Crashpad fonctionne manifestement (il a dumpé le processus principal quelques secondes plus tard). Cela indique que le processus GPU ne provoque pas d’exception normale, mais est arrêté brutalement, à la manière de `TerminateProcess`, et que le code de sortie non documenté 0x060C201E provient précisément de là. Son dernier kilomètre se situe donc chez Anthropic : leur télémétrie Sentry reçoit les dumps supprimés localement, y compris la question de savoir si le signalement des crashes couvre le processus GPU.

## Contournement et état des signalements

Puisque le déclencheur est constitué par les demandes WebGPU des pages dans le navigateur intégré, désactiver WebGPU via un flag Chromium aide jusqu’à la correction. Dans une installation Store, le chemin d’installation change à chaque mise à jour ; un petit lanceur le résout donc à nouveau à chaque démarrage :

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

Depuis ce changement : plus un seul crash. L’analyse complète a été signalée : les expériences de laboratoire et les références aux doublons dans la première issue, l’évaluation du minidump avec la chaîne causale dans la seconde. Les trois corrections pertinentes découlent directement du constat : clarifier la cause du crash dans le processus GPU logiciel (les dumps correspondants sont disponibles dans la télémétrie du fabricant), désactiver WebGPU de manière ciblée lors de crashes GPU répétés au lieu de laisser la chaîne de repli s’épuiser, et repenser la désactivation automatique de l’accélération matérielle, car elle raccourcit la chaîne.

## Addendum : le contournement va trop peu loin, la solution est plus profonde

Le soir même, nouveau crash, signature identique. La raison est simple : le lanceur avec `--disable-features=WebGPU` ne fonctionne que si l’application est effectivement lancée par son intermédiaire. Lors du démarrage habituel depuis le menu Démarrer, l’application s’exécute sans le flag et, pour une application Store, il n’existe aucun moyen propre d’ajouter durablement des flags de ligne de commande à un démarrage normal.

La solution durable figure pourtant depuis longtemps dans la chaîne causale de cet article : l’arrêt automatique fatal suppose que la chaîne de repli soit vide, et elle ne l’est immédiatement que parce que l’application démarre avec l’accélération matérielle désactivée. Il faut donc réactiver l’accélération matérielle dans la `config.json` de l’application :

```json
"isHardwareAccelerationDisabled": false
```

Cela prend effet au démarrage suivant de l’application et résout simultanément les deux aspects du problème. Premièrement, `requestAdapter()` emprunte alors le chemin matériel, dont la stabilité sur cette machine est démontrée (expérience 2 : adaptateur et périphérique sans erreur). Deuxièmement, Chromium dispose à nouveau de niveaux de repli en réserve : si le processus GPU devait à nouveau planter, le navigateur bascule vers le rendu logiciel et continue de fonctionner, au lieu de se fermer. La désactivation initiale de l’accélération matérielle, probablement définie à un moment donné comme mesure de stabilité, était en réalité la condition préalable au crash.

Conclusion de l’enquête : l’explication la plus évidente (« c’était le pilote ») aurait conduit à une odyssée infructueuse des pilotes. Deux heures de laboratoire avec la véritable version du moteur l’ont réfutée, et la cause n’a été trouvée que dans le minidump que l’application élimine habituellement. Lorsqu’un processus GPU plante, quatre vérifications doivent donc être effectuées en premier, avant d’accuser un fabricant : la contre-épreuve dans le navigateur actuel, la contre-épreuve dans la version pure du moteur, vérifier si d’autres matériels présentent la même signature, et dumper le processus qui décide réellement de l’arrêt.

## Sources

1.  [Cause racine : « GPU process isn't usable. Goodbye. » de Chromium (issue GitHub #89250)](https://github.com/anthropics/claude-code/issues/89250): L’analyse du minidump de cet article sous forme de rapport de bug, y compris la méthode de capture et des propositions de correction.

2.  [Premier rapport personnel avec résultats de laboratoire (issue GitHub #89226)](https://github.com/anthropics/claude-code/issues/89226): Les expériences 1 à 3 et la correction de l’hypothèse AMD, avec des références aux doublons.

3.  [Le crash du processus GPU ferme toute l’application (issue GitHub #81698)](https://github.com/anthropics/claude-code/issues/81698): Le même crash avec le même code de sortie sur NVIDIA RTX 5080, sans événements TDR ; cela disculpe les pilotes graphiques.

4.  [Code source Chromium : gpu_data_manager_impl_private.cc (tag 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): La fonction IntentionallyCrashBrowserForUnusableGpuProcess et la logique de repli.

5.  [Documentation Electron : child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): L’événement permettant à une application Electron d’observer les crashes du processus GPU et de prendre ses propres mesures.

6.  [Paquet Python minidump](https://pypi.org/project/minidump/): Outil d’analyse des dumps (enregistrement d’exception, liste des modules, chaînes mémoire), sans WinDbg.

7.  [webgpureport.org](https://webgpureport.org/): Page de diagnostic WebGPU ; elle a servi de déclencheur minimal et de contre-épreuve dans Chromium actuel.
