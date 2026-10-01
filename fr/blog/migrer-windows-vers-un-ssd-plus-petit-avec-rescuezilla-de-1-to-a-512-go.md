---
title: "Migrer Windows vers un SSD plus petit avec Rescuezilla : de 1 To à 512 Go"
navTitle: "Migration vers un SSD plus petit"
description: "Rescuezilla ne restaure une image que sur un disque qui s’étend au moins jusqu’à la fin de la dernière partition. Ce guide montre, à l’exemple d’un NVMe de 1 To, comment réduire C: avec GParted, déplacer la partition de récupération, supprimer le dirty flag et restaurer la nouvelle image sur un SSD de 512 Go."
date: "2026-09-29"
kategorie: "PC et matériel"
timeToRead: "9 min de lecture"
themen:
  - pc-hardware
produkte:
  - "windows-client"
protokolle:
  - "backup-dr"
  - "migration"
slug: "migrer-windows-vers-un-ssd-plus-petit-avec-rescuezilla-de-1-to-a-512-go"
translationId: "article-6947213279383f27"
translationOf: windows-kleinere-ssd-rescuezilla
url: https://rafaelpfister.ch/fr/blog/migrer-windows-vers-un-ssd-plus-petit-avec-rescuezilla-de-1-to-a-512-go
translationSourceHash: ad4485736511552671c4e469ddaa672e9439cffb3e74d43bd8dd7333eaf662d3
translationModel: gpt-5.6-terra
translatedAt: 2026-09-30T09:54:13.470Z
translationReview: automatic
---

Rescuezilla est un système live gratuit permettant de sauvegarder et de restaurer des disques entiers, compatible avec le format d’image de Clonezilla (présentation de l’outil et procédure de base : [Rescuezilla : migrer Windows vers un nouveau SSD](/blog/rescuezilla-windows-migration)). Tant que le disque cible est de taille égale ou supérieure, une sauvegarde et une restauration suffisent. S’il est plus petit, la restauration échoue, même si les données utilisées y tiendraient. Cet article documente la migration d’un système Windows 11 d’un NVMe de 1 To (C: de 930 Go, dont environ 350 Go utilisés) vers un NVMe de 512 Go, y compris les messages d’erreur rencontrés en cours de route.

## Pourquoi la restauration échoue sur un disque plus petit

Rescuezilla sauvegarde chaque partition séparément avec Partclone et enregistre également la table de partitions. Lors de la restauration, il écrit cette table sans la modifier sur le disque cible. Rescuezilla ne peut pas réduire une partition. Avant de commencer, il vérifie donc que la dernière partition se trouve entièrement sur le disque cible ; dans le cas contraire, il s’interrompt avec ce message :

```text
The source partition table's final partition (/dev/nvme0n1p4:
1000203091968 bytes) must refer to a region completely within
the destination disk (512110190592 bytes).
```

Une installation Windows typique comporte quatre partitions : la partition système EFI (200 Mo), la partition réservée Microsoft (16 Mo), C: et la partition de récupération avec Windows RE (ici 904 Mio). Windows place la partition de récupération à la fin du disque. Il ne suffit donc pas de réduire C: : la partition de récupération doit aussi être déplacée vers l’avant, directement derrière C:.

La procédure décrite par le wiki de Rescuezilla pour ce cas est la suivante : réduire le disque source avec GParted, créer une nouvelle image, puis restaurer cette image. Une image créée auparavant du disque non modifié reste conservée comme sauvegarde jusqu’à ce que le nouveau système fonctionne.

## Étape 1 : calculer la taille cible

Un SSD vendu comme ayant 512 Go possède 512'110'190'592 octets, soit 476,9 Gio (Rescuezilla affiche également cette valeur). Il faut en déduire EFI, MSR et la partition de récupération, soit un peu plus de 1,1 Gio au total. Il reste donc un peu moins de 475 Gio pour C:. Avec une petite marge, **470 Gio = 481'280 Mio** est une valeur cible judicieuse. Les quelque 6 Go qui resteront libres à la fin du nouveau SSD sont négligeables.

Les données utilisées doivent être inférieures à cette valeur. Cette commande, exécutée dans PowerShell avec des droits d’administrateur, indique jusqu’à quelle taille Windows pourrait réduire un volume :

```powershell
$s = Get-PartitionSupportedSize -DriveLetter C
"{0:N1} GB" -f ($s.SizeMin / 1GB)
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-DriveLetter C` | Volume dont les tailles possibles sont interrogées |
| `SizeMin` | Taille minimale à laquelle Windows pourrait lui-même réduire le volume |
| `"{0:N1} GB" -f` | Formate la valeur en octets en gigaoctets avec une décimale |

</details>

Si `SizeMin` est nettement supérieur à la quantité de données utilisée, des fichiers non déplaçables (fichier d’échange, points de restauration, MFT) empêchent la réduction avec les outils intégrés à Windows. GParted déplace ces données lors de la réduction ; cette limite ne s’y applique pas.

## Étape 2 : désactiver BitLocker et l’hibernation

GParted ne peut ni lire ni réduire un volume chiffré avec BitLocker, et Partclone ne peut pas le sauvegarder comme NTFS. Vérifiez l’état dans PowerShell avec des droits d’administrateur :

```powershell
manage-bde -status C:
```

Si `Fully Decrypted` n’y figure pas, désactivez BitLocker et attendez que le déchiffrement soit terminé :

```powershell
manage-bde -off C:
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-status` | Affiche le niveau de chiffrement, la méthode et l’état de protection du volume |
| `-off` | Déchiffre entièrement le volume et supprime les protecteurs de clé |
| `C:` | Argument positionnel : le volume concerné |

</details>

Le déchiffrement s’effectue en arrière-plan et peut prendre une heure ou plus selon la quantité de données. Sur les appareils gérés par Intune, une stratégie peut réactiver BitLocker peu après. Vérifiez donc à nouveau l’état juste avant de démarrer GParted.

L’hibernation doit également être désactivée. Lorsque le démarrage rapide est actif, Windows ne s’arrête pas complètement, mais enregistre l’état du noyau dans `hiberfil.sys`. NTFS est alors considéré comme encore utilisé, et GParted refuse toute modification. Une commande désactive simultanément l’hibernation et le démarrage rapide, et supprime `hiberfil.sys` :

```powershell
powercfg /h off
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `/h off` | Forme courte de `/hibernate off`: désactive l’hibernation et le démarrage rapide, `hiberfil.sys` est supprimé |

</details>

## Étape 3 : démarrer Rescuezilla

Windows peut diriger le prochain démarrage directement vers le menu de démarrage avec les options avancées. Sélectionnez-y « Utiliser un périphérique », puis la clé USB contenant Rescuezilla :

```powershell
shutdown /r /o /t 0
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `/r` | Redémarre au lieu d’arrêter l’ordinateur |
| `/o` | Démarre dans les options de démarrage avancées (Windows RE) ; uniquement avec `/r` |
| `/t 0` | Délai en secondes avant l’exécution |

</details>

Sinon, le menu de démarrage du firmware s’ouvre lors de la mise sous tension, avec F11 sur les cartes ASRock. Si « Utiliser un périphérique » est absent, `shutdown /r /fw /t 0` mène directement à la configuration UEFI, où il est possible de sélectionner le disque de démarrage pour le prochain démarrage.

## Étape 4 : réduire C: et déplacer la partition de récupération

Sur le bureau de Rescuezilla, lancez **Partition Editor** (GParted), et non Rescuezilla lui-même.

1.  Sélectionnez le disque source en haut à droite. Vérifiez sa taille (ici 931.51 Gio) afin de ne pas modifier par erreur le disque de sauvegarde externe ou un second disque interne.

2.  Faites un clic droit sur la partition NTFS contenant C: → **Resize/Move**. Saisissez la valeur cible dans le champ **New size (MiB)**, ici `481280`, et validez avec la touche Tab. **Free space preceding** reste inchangé. Confirmez avec **Resize/Move**.

3.  Sous C: apparaît maintenant une ligne `unallocated`, puis la partition de récupération (NTFS, environ 900 Mio, indicateurs `hidden, diag`). Faites un clic droit sur la partition de récupération → **Resize/Move**.

4.  Dans le champ **Free space preceding (MiB)**, effacez la valeur, saisissez `0` et validez avec Tab. **New size** reste identique ; **Free space following** passe à l’ensemble de l’espace libre. Si `1` apparaît après avoir appuyé sur Tab au lieu de `0`, cela correspond à l’alignement sur des Mio entiers et est normal. Confirmez avec **Resize/Move**.

5.  Confirmez avec OK l’avertissement indiquant que le déplacement pourrait empêcher le démarrage. Il concerne les partitions contenant un chargeur de démarrage ; Windows démarre depuis la partition système EFI, qui reste inchangée.

6.  L’ordre est désormais : EFI, MSR, C:, partition de récupération, `unallocated`. Jusqu’ici, les opérations sont seulement mises en attente. Ce n’est qu’un clic sur la coche verte (**Apply All Operations**) qui applique les modifications.

Windows RE retrouve sa partition après le déplacement, car elle conserve son numéro de partition. `reagentc /info` affiche toujours ensuite `Enabled` avec le chemin `harddisk0\partition4\Recovery\WindowsRE`.

Le guide de Rescuezilla ne réduit que la dernière partition. Cela suffit lorsque C: est la dernière partition. Dans une installation standard de Windows 10 ou 11, la partition de récupération se trouve après elle ; son déplacement est alors obligatoire.

## Étape 5 : supprimer le dirty flag

La sauvegarde suivante échoue sur C: après la réduction avec le message suivant :

```text
ntfsclone-ng.c: NTFS Volume '/dev/nvme0n1p3' is scheduled for a check
or it was shutdown uncleanly. Please boot Windows or fix it by fsck.
```

La cause est intentionnelle : `ntfsresize`, utilisé par GParted pour NTFS, marque le système de fichiers pour vérification avant chaque modification de taille et laisse ce marquage en place. Selon la page de manuel, Windows doit exécuter `chkdsk` au prochain démarrage. Dans le cas décrit, le flag restait toutefois actif après un démarrage normal de Windows, et `Get-Volume C` indiquait `Full Repair Needed`. Vérifiez l’état dans PowerShell avec des droits d’administrateur :

```powershell
fsutil dirty query C:
```

Si la commande indique `Volume - C: is Dirty`, faites exécuter la vérification au prochain démarrage :

```powershell
chkdsk C: /f
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `dirty query C:` | fsutil : vérifie si le dirty bit du volume est défini |
| `C:` | chkdsk : volume à vérifier |
| `/f` | Corrige les erreurs trouvées ; sur le volume système, la vérification est planifiée pour le prochain démarrage |

</details>

Répondez J à la question demandant si la vérification doit être exécutée au prochain redémarrage, puis redémarrez Windows normalement. Ensuite, `fsutil dirty query C:` doit afficher le message `is NOT Dirty`, et `Get-Volume C` doit à nouveau indiquer `Healthy`. Ne redémarrez Rescuezilla qu’à ce moment-là.

## Étape 6 : créer et vérifier la nouvelle image

Créez maintenant avec Rescuezilla une nouvelle image du disque réduit. La taille de l’image reste pratiquement identique à celle de la première sauvegarde (ici environ 214 Go), car Partclone ne sauvegarde que les blocs utilisés. Rescuezilla place chaque sauvegarde dans son propre dossier horodaté, par exemple `2026-09-29-1343-img-rescuezilla`.

Tous les images apparaissent ensuite côte à côte dans la liste de sélection des images. La colonne **Partitions** indique les tailles : l’image créée avant la réduction contient `ntfs 930.4GB`, la nouvelle contient `ntfs 470GB`. Une tentative de sauvegarde interrompue apparaît avec un triangle d’avertissement jaune ; elle contient l’ancienne table de partitions et déclenche à la restauration exactement le message d’erreur de la première section. Supprimez de tels dossiers afin de ne pas les sélectionner par erreur.

Vous pouvez aussi vérifier au préalable si une image tient sur le disque cible dans le fichier `<disk>-pt.parted` du dossier d’image. Il contient la table de partitions sauvegardée en secteurs de 512 octets :

```text
Number  Start       End         Size        File system  Name
 1      2048s       411647s     409600s     fat32        EFI system partition
 2      411648s     444415s     32768s                   Microsoft reserved partition
 3      444416s     986105855s  985661440s  ntfs         Basic data partition
 4      986105856s  987957247s  1851392s    ntfs
```

La fin de la dernière partition (987'957'247 + 1 secteurs × 512 octets = 505,8 Go) est inférieure aux 512,1 Go du disque cible. L’image convient.

## Étape 7 : restaurer sur le nouveau SSD

1.  Dans Rescuezilla, sélectionnez **Restore**, le disque contenant les images, puis la **nouvelle** image (sans triangle d’avertissement, avec la partition C: réduite).

2.  Sélectionnez le nouveau SSD comme cible. Vérifiez deux fois le modèle et la taille : la restauration écrase entièrement le disque cible.

3.  Laissez toutes les partitions cochées, ainsi que **Overwrite partition table**, puis démarrez la restauration.

4.  Démarrez ensuite depuis le nouveau SSD. Le plus simple est de débrancher auparavant l’ancien disque ; sinon, sélectionnez le nouveau SSD dans le menu de démarrage du firmware.

## Opérations finales

Sur le nouveau disque, réactivez ce qui a été désactivé pour la migration. `powercfg /h on` réactive l’hibernation. Activez BitLocker via Paramètres → Confidentialité et sécurité → Chiffrement de l’appareil, ou avec `manage-bde -on C:` ; vérifiez ensuite avec `manage-bde -protectors -get C:` que la clé de récupération est sauvegardée. Si Secure Boot a été désactivé dans l’UEFI pour démarrer Rescuezilla, réactivez-le ; tant qu’il est désactivé, BitLocker consigne l’événement 810 à chaque démarrage.

Vous pouvez supprimer l’image du disque de 1 To non modifié dès que Windows démarre correctement depuis le nouveau SSD et que les données sont complètes.

## Sources

1.  [Wiki Rescuezilla : restauration sur un disque plus petit](https://github.com/rescuezilla/rescuezilla/wiki/HOWTO:-Restoring-to-a-smaller-disk.-Eg,-1000GB-HDD-to-500GB-SSD): procédure officielle avec GParted, raison de la restriction lors de la restauration.

2.  [Rescuezilla](https://rescuezilla.com/): page du projet avec le téléchargement du système live.

3.  [Manuel GParted](https://gparted.org/display-doc.php?name=help-manual): utilisation de Resize/Move, alignement sur les Mio et exécution des opérations mises en attente.

4.  [ntfsresize(8), page de manuel Ubuntu](https://manpages.ubuntu.com/manpages/noble/man8/ntfsresize.8.html): outil utilisé pour la réduction NTFS dans GParted ; décrit le marquage de vérification défini intentionnellement.

5.  [Microsoft Learn : chkdsk](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/chkdsk): paramètre `/f` et planification de la vérification sur le volume système.

6.  [Microsoft Learn : fsutil dirty](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/fsutil-dirty): interrogation et signification du dirty bit.

7.  [Microsoft Learn : manage-bde status](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-status): interrogation de l’état de BitLocker.

8.  [Microsoft Learn : manage-bde off](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-off): déchiffrement d’un volume.

9.  [Microsoft Learn : options de ligne de commande Powercfg](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/powercfg-command-line-options): option `/hibernate` et son influence sur le démarrage rapide.

10.  [Microsoft Learn : Get-PartitionSupportedSize](https://learn.microsoft.com/en-us/powershell/module/storage/get-partitionsupportedsize): taille minimale et maximale de partition du point de vue de Windows.

11.  [Microsoft Learn : options de ligne de commande REAgentC](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/reagentc-command-line-options): vérification de l’état et de l’emplacement de Windows RE.

12.  [Microsoft Learn : shutdown](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): paramètres `/o` et `/fw` pour démarrer respectivement dans les options avancées ou dans l’UEFI.
