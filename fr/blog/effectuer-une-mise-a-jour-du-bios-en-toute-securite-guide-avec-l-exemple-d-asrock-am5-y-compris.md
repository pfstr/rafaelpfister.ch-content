---
title: "Effectuer une mise à jour du BIOS en toute sécurité : guide avec l’exemple d’ASRock AM5, préparation BitLocker incluse"
navTitle: "Mise à jour du BIOS"
description: "Le déroulement complet d’une mise à jour du BIOS à l’exemple d’une carte ASRock AM5 : déterminer la version, vérifier le téléchargement par hachage, suspendre correctement BitLocker, démarrer dans l’UEFI (même si F2 ne fonctionne pas), mettre à jour avec Instant Flash et configurer judicieusement les paramètres après la mise à jour."
date: "2026-08-26"
kategorie: "PC & matériel"
timeToRead: "8 min de lecture"
themen:
  - pc-hardware
produkte:
  - "windows-client"
protokolle:
  - "troubleshooting"
  - "releases"
slug: "effectuer-une-mise-a-jour-du-bios-en-toute-securite-guide-avec-l-exemple-d-asrock-am5-y-compris"
translationId: "article-82840b2d159b9367"
translationOf: bios-update-asrock-am5-sicher-durchfuehren
translationSourceHash: 555b16e753b2ac5dec357741b071ed6aa33de367a2197a8dbb10fef7c9f6a946
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:13:59.803Z
translationReview: automatic
url: https://rafaelpfister.ch/fr/blog/effectuer-une-mise-a-jour-du-bios-en-toute-securite-guide-avec-l-exemple-d-asrock-am5-y-compris
---

Une mise à jour du BIOS fait partie des opérations de maintenance rarement nécessaires et qui soulèvent donc à chaque fois les mêmes questions : quelle est la bonne version, comment la transférer en toute sécurité sur la carte mère, et que faut-il faire avant et après ? Ce guide documente l’ensemble de la procédure à l’exemple d’une ASRock A620I Lightning WiFi (socket AM5), avec la méthode Instant Flash du fabricant. Les étapes s’appliquent à toute carte mère moderne ; les points critiques (BitLocker, Fast Boot, réinitialisation des paramètres) sont indépendants du fabricant.

## Quand une mise à jour du BIOS est indiquée

Trois raisons justifient cette intervention. Premièrement, les correctifs de sécurité : les failles de firmware ne peuvent être corrigées que par une mise à jour du BIOS, et les journaux des modifications des fabricants ne les mentionnent généralement que brièvement. Deuxièmement, la compatibilité : la prise en charge des nouvelles générations de processeurs et l’amélioration de la compatibilité mémoire passent exclusivement par de nouvelles versions du firmware ; sur AM5, elles reposent sur le firmware de référence AGESA d’AMD, que les fabricants de cartes intègrent à leurs versions du BIOS. Troisièmement, la stabilité : si un système redémarre spontanément et que le journal des événements ne consigne à ce sujet que Kernel-Power 41 avec `BugcheckCode=0`, le plantage s’est produit au niveau matériel ou firmware, sans intervention de Windows ; les causes typiques sont des tensions instables et l’entraînement de la mémoire, précisément le niveau pris en charge par les versions AGESA. Des entrées telles que « Improve memory compatibility and system stability » ou une gestion EXPO revue dans les journaux des modifications indiquent qu’une mise à jour traite ces problèmes. En revanche, si un système est stable et n’est pas concerné par les failles corrigées, attendre est légitime ; une mise à jour du BIOS sans raison est un risque sans bénéfice.

## Étape 1 : déterminer l’état actuel

Avant de télécharger quoi que ce soit, vous avez besoin de deux informations : le modèle exact de la carte mère et la version du BIOS installée. PowerShell fournit les deux sans redémarrage :

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object Manufacturer, Model
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion, ReleaseDate
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Win32_ComputerSystem` | Argument positionnel ClassName : classe CIM contenant le fabricant et le modèle du système |
| `Win32_BIOS` | Classe CIM contenant les informations du firmware, notamment la version et la date |
| `Select-Object <eigenschaften>` | limite la sortie aux propriétés indiquées |

</details>

Notez la version. Vous en aurez besoin plus tard pour vérifier le succès de l’opération et, lors de la lecture des journaux des modifications, pour savoir quelles versions vous sautez.

## Étape 2 : télécharger le BIOS et vérifier la somme de contrôle

Téléchargez le BIOS exclusivement depuis la page produit du fabricant, jamais depuis des portails tiers. ASRock publie la somme de contrôle SHA256 pour chaque version ; après le téléchargement, comparez-la avant même que le fichier ne s’approche d’une clé USB :

```powershell
Get-FileHash .\A620I_Lightning_WiFi_4.43.zip -Algorithm SHA256
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `.\A620I_…_4.43.zip` | Argument positionnel Path : le fichier à vérifier |
| `-Algorithm SHA256` | Algorithme de hachage ; il doit correspondre au type de somme de contrôle publié par le fabricant |

</details>

Si la valeur ne correspond pas à celle indiquée par le fabricant, le téléchargement est endommagé ou manipulé : ne flashez pas. Après décompression, il reste un seul fichier ROM, dans l’exemple `A62IRW_4.43.ROM` de 32 Mo.

## Étape 3 : préparer la clé USB

Le mécanisme de flash dans l’UEFI (chez ASRock « Instant Flash », chez d’autres fabricants Q-Flash, EZ Flash ou M-Flash) lit directement la clé depuis le firmware. Cela signifie que seul FAT32 est reconnu de manière fiable, pas NTFS ni exFAT. Presque toutes les clés achetées prêtes à l’emploi sont déjà en FAT32 ; vous pouvez le vérifier ainsi :

```powershell
Get-CimInstance Win32_LogicalDisk -Filter "DriveType=2" |
  Select-Object DeviceID, FileSystem, VolumeName
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Win32_LogicalDisk` | Classe CIM des lecteurs logiques |
| `-Filter "DriveType=2"` | Filtre WQL sur les supports amovibles ; masque les disques durs et les lecteurs CD |
| `Select-Object DeviceID, FileSystem, VolumeName` | affiche la lettre du lecteur, le système de fichiers et le nom du volume |

</details>

Copiez le fichier ROM à la racine de la clé. Un reformatage n’est nécessaire que si le système de fichiers ne convient pas. La capacité de la clé n’a pas d’importance : le fichier est plus petit que toute capacité courante.

Une remarque sur le choix de la méthode : de nombreuses cartes proposent aussi un bouton BIOS Flashback, qui permet de flasher sans processeur ni système fonctionnel. C’est la solution de secours pour une carte qui ne démarre plus. Pour un système fonctionnel, Instant Flash dans l’UEFI est la solution appropriée et la plus simple. Les outils de flash sous Windows ne sont ni nécessaires ni recommandés sur les plateformes actuelles.

## Étape 4 : suspendre BitLocker, faute de quoi une demande de clé risque d’apparaître

C’est le point absent de nombreux guides. Si le disque système est chiffré avec BitLocker (souvent activé automatiquement sous Windows 11 avec un compte Microsoft), BitLocker associe la clé aux mesures du TPM. Une mise à jour du BIOS modifie ces mesures et, au démarrage suivant, Windows demande la clé de récupération à 48 chiffres. Sans elle à portée de main, le système devient inaccessible.

BitLocker dispose de son propre mécanisme pour ce scénario. Dans une PowerShell ouverte avec des droits d’administrateur :

```powershell
Suspend-BitLocker -MountPoint C: -RebootCount 2
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-MountPoint C:` | le volume concerné, ici le disque système |
| `-RebootCount 2` | nombre de redémarrages pendant lesquels la protection reste suspendue (0 à 15 ; 0 = jusqu’à la réactivation manuelle) |

</details>

La valeur 2 couvre les deux redémarrages à venir (un vers l’UEFI, un après le flash) ; la protection se réactive ensuite automatiquement et scelle la clé avec les nouvelles mesures. Vérifiez néanmoins au préalable que la clé de récupération est accessible, par exemple dans le compte Microsoft à l’adresse aka.ms/myrecoverykey ou avec `manage-bde -protectors -get C:`.

## Étape 5 : accéder à l’UEFI, même si F2 ne répond pas

La méthode classique, consistant à appuyer sur F2 ou Suppr à l’allumage, échoue souvent sur les systèmes modernes : lorsque Fast Boot est activé, le firmware n’initialise le clavier USB qu’après le POST, si bien que la touche pressée n’est pas prise en compte. Vous n’êtes toutefois pas dépendant de cette touche : Windows peut diriger le prochain redémarrage directement vers la configuration UEFI. Dans une PowerShell ouverte avec des droits d’administrateur :

```powershell
shutdown /r /fw /t 5
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `/r` | redémarre au lieu d’arrêter |
| `/fw` | définit la variable du firmware qui dirige le prochain démarrage directement vers la configuration UEFI ; uniquement avec une option d’arrêt telle que `/r`, nécessite des droits d’administrateur |
| `/t 5` | délai en secondes avant l’exécution |

</details>

Si la commande renvoie l’erreur 203 (« The system could not find the environment option that was entered »), les droits d’administrateur font presque toujours défaut : sans élévation, le processus ne peut pas définir la variable de firmware nécessaire, et le message d’erreur ne précise pas cette cause. Une seconde méthode, sans variable de firmware, passe par l’environnement de récupération : `shutdown /r /o`, puis Dépannage, Options avancées, Paramètres du microprogramme UEFI.

## Étape 6 : flasher avec Instant Flash

Dans l’UEFI, vous trouverez Instant Flash dans le menu Tool. L’outil répertorie tous les fichiers ROM présents sur la clé ; après sélection, il vérifie le fichier, le flashe puis redémarre automatiquement. Durant ces quelques minutes, une seule règle impérative s’applique à toute la procédure : ne coupez pas l’alimentation et n’éteignez pas l’ordinateur. Un flash interrompu est la seule étape de ce guide qui puisse réellement empêcher la carte de démarrer (et nécessite alors la procédure de secours Flashback mentionnée).

## Étape 7 : finaliser, car la mise à jour réinitialise tout

Après le flash, tous les paramètres du BIOS sont rétablis aux valeurs d’usine. C’est prévu et cela offre une opportunité de diagnostic : la RAM fonctionne désormais sans profil EXPO, à la vitesse JEDEC de base. Si vous avez effectué le flash pour résoudre des problèmes de stabilité, laissez volontairement la configuration ainsi pendant une à deux semaines. Si les plantages disparaissent, le profil mémoire était impliqué et vous pourrez tester à nouveau EXPO de manière ciblée avec le nouveau firmware. La différence au quotidien entre 4800 et 6000 MT/s est à peine perceptible en dehors des benchmarks ; un ordinateur stable vaut chaque point de benchmark.

Deux paramètres méritent de toute façon un passage dans l’UEFI : si vous aviez des redémarrages au repos, vous pouvez régler l’option « Power Supply Idle Control » sur « Typical Current Idle » sous Advanced, AMD CBS ; cela atténue une incompatibilité connue entre certaines alimentations et les états de veille profonde des processeurs Ryzen. Et si vous souhaitez à l’avenir accéder de nouveau à la configuration avec F2, vous pouvez désactiver Fast Boot.

Le contrôle de réussite de retour dans Windows :

```powershell
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion
Get-CimInstance Win32_PhysicalMemory |
  Select-Object PartNumber, ConfiguredClockSpeed
manage-bde -status C:
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Win32_BIOS` | Classe CIM contenant la version du firmware ; `SMBIOSBIOSVersion` doit maintenant afficher la nouvelle version |
| `Win32_PhysicalMemory` | Classe CIM des modules mémoire ; `ConfiguredClockSpeed` indique la fréquence réellement appliquée en MT/s |
| `-status` | manage-bde : affiche l’état du chiffrement et de la protection du volume |
| `C:` | Argument positionnel : le volume à vérifier |

</details>

La première ligne doit afficher la nouvelle version, la deuxième la fréquence mémoire attendue, et BitLocker doit de nouveau indiquer « Protection activée ». La mise à jour est alors terminée et documentée. Si le flash a été effectué pour des problèmes de stabilité, seule l’observation au cours des semaines suivantes permettra de déterminer s’ils sont résolus, le plus simplement en consultant les nouveaux événements Kernel-Power 41 dans le journal des événements système.

## Sources

1.  [ASRock A620I Lightning WiFi, téléchargements BIOS](https://pg.asrock.com/mb/AMD/A620I%20Lightning%20WiFi/index.asp#BIOS): liste des versions avec journaux des modifications, sommes de contrôle SHA256 et méthodes de mise à jour prises en charge par la carte utilisée en exemple.

2.  [Microsoft Learn: Suspend-BitLocker](https://learn.microsoft.com/en-us/powershell/module/bitlocker/suspend-bitlocker): référence sur la suspension de la protection BitLocker, y compris le paramètre RebootCount.

3.  [Microsoft Learn: Advanced troubleshooting for Event ID 41](https://learn.microsoft.com/en-us/troubleshoot/windows-client/performance/event-id-41-restart): explication de Kernel-Power 41 et de la signification de BugcheckCode 0.

4.  [Microsoft Learn: commande shutdown](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): documentation des paramètres /fw et /o pour redémarrer respectivement dans l’UEFI ou dans l’environnement de récupération.
