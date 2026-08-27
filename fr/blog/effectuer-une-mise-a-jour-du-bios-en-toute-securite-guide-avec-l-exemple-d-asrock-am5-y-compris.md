---
title: "Effectuer une mise à jour du BIOS en toute sécurité : guide avec l’exemple d’ASRock AM5, y compris la préparation de BitLocker"
navTitle: "Mise à jour du BIOS"
description: "La procédure complète d’une mise à jour du BIOS à partir d’une carte ASRock AM5 : déterminer la version, vérifier le téléchargement par hachage, suspendre correctement BitLocker, démarrer dans l’UEFI (même si F2 ne fonctionne pas), mettre à jour via Instant Flash et configurer judicieusement les paramètres après la mise à jour."
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
url: https://rafaelpfister.ch/fr/blog/effectuer-une-mise-a-jour-du-bios-en-toute-securite-guide-avec-l-exemple-d-asrock-am5-y-compris
translationSourceHash: 60fff28a10b0f91f3d59996b00afe614f2230b9831514dcf01a1f496b99f4fbd
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:25:42.694Z
translationReview: automatic
---

Une mise à jour du BIOS fait partie des opérations de maintenance qui sont rarement nécessaires et soulèvent donc à chaque fois les mêmes questions : quelle version choisir, comment la transférer sur la carte en toute sécurité, et quels sont les points à respecter avant et après ? Ce guide documente l’ensemble de la procédure à partir d’une ASRock A620I Lightning WiFi (socket AM5), avec la méthode Instant Flash du fabricant. Les étapes s’appliquent à toute carte mère moderne ; les points critiques (BitLocker, Fast Boot, réinitialisation des paramètres) sont indépendants du fabricant.

## Quand une mise à jour du BIOS est indiquée

Trois raisons justifient cette intervention. Premièrement, les correctifs de sécurité : les failles du firmware ne peuvent être corrigées que par une mise à jour du BIOS, et les journaux des modifications des fabricants ne les mentionnent généralement que brièvement. Deuxièmement, la compatibilité : la prise en charge des nouvelles générations de CPU et l’amélioration de la compatibilité mémoire passent exclusivement par de nouvelles versions du firmware, sur AM5 via le firmware de référence AGESA d’AMD, que les fabricants de cartes intègrent à leurs versions du BIOS. Troisièmement, la stabilité : si un système redémarre spontanément et que le journal des événements n’enregistre que Kernel-Power 41 avec `BugcheckCode=0`, le plantage s’est produit au niveau matériel ou firmware, sans intervention de Windows ; les causes typiques sont des tensions instables et l’entraînement mémoire, précisément la couche que maintiennent les versions d’AGESA. Des entrées telles que « Improve memory compatibility and system stability » ou une gestion EXPO révisée dans les journaux des modifications indiquent qu’une mise à jour traite ce type de problèmes. Si, en revanche, un système est stable et n’est pas concerné par les failles corrigées, attendre est légitime ; une mise à jour du BIOS sans raison représente un risque sans contrepartie.

## Étape 1 : déterminer l’état actuel

Avant de télécharger quoi que ce soit, vous avez besoin de deux informations : le modèle exact de la carte et la version du BIOS installée. PowerShell fournit les deux sans redémarrage :

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object Manufacturer, Model
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion, ReleaseDate
```

Notez la version. Vous en aurez besoin plus tard pour contrôler le résultat, et lors de la lecture des journaux des modifications, vous voudrez savoir quelles versions vous sautez.

## Étape 2 : télécharger le BIOS et vérifier la somme de contrôle

Téléchargez le BIOS exclusivement depuis la page produit du fabricant, jamais depuis des portails tiers. ASRock publie la somme de contrôle SHA256 pour chaque version ; après le téléchargement, comparez-la avant même que le fichier n’approche d’une clé USB :

```powershell
Get-FileHash .\A620I_Lightning_WiFi_4.43.zip -Algorithm SHA256
```

Si la valeur ne correspond pas à celle indiquée par le fabricant, le téléchargement est endommagé ou manipulé : ne flashez pas. Après décompression, il reste un seul fichier ROM, dans l’exemple `A62IRW_4.43.ROM` de 32 Mo.

## Étape 3 : préparer la clé USB

Le mécanisme de flash dans l’UEFI (chez ASRock, « Instant Flash », chez d’autres fabricants Q-Flash, EZ Flash ou M-Flash) lit directement la clé depuis le firmware. Cela signifie que seul FAT32 est reconnu de manière fiable, pas NTFS ni exFAT. Presque toutes les clés achetées prêtes à l’emploi sont déjà en FAT32 ; vous pouvez le vérifier ainsi :

```powershell
Get-CimInstance Win32_LogicalDisk -Filter "DriveType=2" |
  Select-Object DeviceID, FileSystem, VolumeName
```

Copiez le fichier ROM dans le répertoire racine de la clé. Un reformatage n’est nécessaire que si le système de fichiers ne convient pas. La taille de la clé n’a pas d’importance, le fichier est inférieur à toute capacité courante.

Une remarque concernant le choix de la méthode : de nombreuses cartes disposent également d’un bouton BIOS Flashback, qui permet de flasher sans CPU ni système fonctionnel. C’est la solution de secours pour une carte qui ne démarre plus. Pour un système opérationnel, Instant Flash dans l’UEFI est la méthode appropriée et la plus simple. Les outils de flash sous Windows ne sont ni nécessaires ni recommandés sur les plateformes actuelles.

## Étape 4 : suspendre BitLocker, sinon une demande de clé risque d’apparaître

C’est le point absent de nombreux guides. Si le disque système est chiffré avec BitLocker (souvent activé automatiquement sous Windows 11 avec un compte Microsoft), BitLocker lie la clé aux valeurs mesurées par le TPM. Une mise à jour du BIOS modifie ces valeurs et, au démarrage suivant, Windows demande la clé de récupération à 48 chiffres. Sans l’avoir à portée de main, le système devient inaccessible.

BitLocker intègre son propre mécanisme pour ce scénario. Dans une PowerShell exécutée avec des droits d’administrateur :

```powershell
Suspend-BitLocker -MountPoint C: -RebootCount 2
```

Le paramètre `RebootCount 2` couvre les deux redémarrages à venir (une fois vers l’UEFI, une fois après le flash) ; la protection se réactive ensuite d’elle-même et scelle la clé avec les nouvelles valeurs. Vérifiez néanmoins au préalable que la clé de récupération est accessible, par exemple dans le compte Microsoft à l’adresse aka.ms/myrecoverykey ou via `manage-bde -protectors -get C:`.

## Étape 5 : accéder à l’UEFI, même si F2 ne répond pas

La méthode classique, consistant à appuyer sur F2 ou Suppr à l’allumage, échoue souvent sur les systèmes modernes : lorsque Fast Boot est activé, le firmware n’initialise le clavier USB qu’après le POST, la pression de touche n’est donc pas prise en compte. Vous n’êtes toutefois pas dépendant de cette touche : Windows peut diriger directement le prochain redémarrage vers la configuration UEFI. Dans une PowerShell exécutée avec des droits d’administrateur :

```powershell
shutdown /r /fw /t 5
```

Si la commande signale l’erreur 203 (« The system could not find the environment option that was entered »), les droits d’administrateur font presque toujours défaut : sans élévation, le processus ne peut pas définir la variable firmware nécessaire, et le message d’erreur ne mentionne pas cette cause. Une deuxième méthode, sans variable firmware, passe par l’environnement de récupération : `shutdown /r /o`, puis Dépannage, Options avancées, Paramètres du microprogramme UEFI.

## Étape 6 : flasher avec Instant Flash

Dans l’UEFI, vous trouverez Instant Flash dans le menu Tool. L’outil répertorie tous les fichiers ROM présents sur la clé ; après sélection, il vérifie le fichier, flashe puis redémarre automatiquement. Durant ces quelques minutes, la seule règle impérative de toute la procédure s’applique : ne coupez pas l’alimentation et n’éteignez pas l’ordinateur. Un flash interrompu est la seule étape de ce guide qui puisse réellement empêcher la carte de démarrer (et nécessite alors la solution de secours Flashback mentionnée plus haut).

## Étape 7 : effectuer les réglages après coup, car la mise à jour réinitialise tout

Après le flash, tous les paramètres du BIOS reviennent aux réglages d’usine. C’est prévu ainsi et offre une opportunité de diagnostic : la RAM fonctionne désormais sans profil EXPO, à la vitesse de base JEDEC. Si vous avez flashé en raison de problèmes de stabilité, laissez délibérément cette configuration pendant une à deux semaines. Si les plantages disparaissent, le profil mémoire était impliqué et vous pourrez tester à nouveau EXPO de manière ciblée avec le nouveau firmware. La différence au quotidien entre 4800 et 6000 MT/s est à peine perceptible en dehors des benchmarks ; un ordinateur stable vaut tous les points de benchmark.

Deux paramètres méritent dans tous les cas une visite dans l’UEFI : si vous aviez des redémarrages au repos, vous pouvez régler l’option « Power Supply Idle Control » sur « Typical Current Idle » dans Advanced, AMD CBS ; cela atténue une incompatibilité connue entre certaines alimentations et les états de veille profonds des CPU Ryzen. Et si vous souhaitez à l’avenir accéder à nouveau à la configuration avec F2, vous pouvez désactiver Fast Boot.

Le contrôle du résultat de retour dans Windows :

```powershell
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion
Get-CimInstance Win32_PhysicalMemory |
  Select-Object PartNumber, ConfiguredClockSpeed
manage-bde -status C:
```

La première ligne doit afficher la nouvelle version, la deuxième la fréquence mémoire attendue, et BitLocker doit à nouveau indiquer « Protection activée ». La mise à jour est ainsi terminée et documentée. Si le flash a été effectué en raison de problèmes de stabilité, seule l’observation au cours des semaines suivantes montrera s’ils sont résolus, le plus simplement en vérifiant les nouvelles entrées Kernel-Power 41 dans le journal des événements système.

## Sources

1.  [ASRock A620I Lightning WiFi, téléchargements du BIOS](https://pg.asrock.com/mb/AMD/A620I%20Lightning%20WiFi/index.asp#BIOS): liste des versions avec journaux des modifications, sommes de contrôle SHA256 et méthodes de mise à jour prises en charge par la carte utilisée en exemple.

2.  [Microsoft Learn : Suspend-BitLocker](https://learn.microsoft.com/en-us/powershell/module/bitlocker/suspend-bitlocker): référence pour suspendre la protection BitLocker, y compris le paramètre RebootCount.

3.  [Microsoft Learn : dépannage avancé pour l’ID d’événement 41](https://learn.microsoft.com/en-us/troubleshoot/windows-client/performance/event-id-41-restart): explication de Kernel-Power 41 et de la signification de BugcheckCode 0.

4.  [Microsoft Learn : commande shutdown](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): documentation des paramètres /fw et /o pour redémarrer respectivement dans l’UEFI ou dans l’environnement de récupération.
