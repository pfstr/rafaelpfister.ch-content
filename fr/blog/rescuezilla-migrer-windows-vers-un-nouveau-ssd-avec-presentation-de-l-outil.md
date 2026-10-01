---
title: "Rescuezilla : migrer Windows vers un nouveau SSD, avec présentation de l’outil"
navTitle: "Migration Rescuezilla"
description: "Rescuezilla est une solution d’imagerie gratuite, compatible avec Clonezilla et dotée d’une interface graphique. Cet article présente l’outil et montre la migration complète d’une installation Windows vers un nouveau SSD : préparation dans Windows, clé de démarrage, sauvegarde, vérification, restauration et étapes finales."
date: "2026-09-29"
kategorie: "PC & matériel"
timeToRead: "10 min de lecture"
themen:
  - pc-hardware
produkte:
  - "windows-client"
protokolle:
  - "backup-dr"
  - "migration"
slug: "rescuezilla-migrer-windows-vers-un-nouveau-ssd-avec-presentation-de-l-outil"
translationId: "article-d9438c5774d0167c"
translationOf: rescuezilla-windows-migration
url: https://rafaelpfister.ch/fr/blog/rescuezilla-migrer-windows-vers-un-nouveau-ssd-avec-presentation-de-l-outil
translationSourceHash: 33320c51c0a77d69bc5804e2c7e69d14c63731a23df6ef55969b37b043a49cd5
translationModel: gpt-5.6-terra
translatedAt: 2026-09-30T09:50:07.704Z
translationReview: automatic
---

Pour transférer une installation Windows vers un nouveau SSD ou un nouvel ordinateur sans réinstaller Windows, il faut un outil capable de sauvegarder et de restaurer l’intégralité du disque avec toutes ses partitions. Rescuezilla est un tel outil : gratuit, open source et doté d’une interface graphique utilisable sans connaissances de Linux. Cet article présente l’outil et décrit la migration vers un SSD de même taille ou plus grand. Un guide distinct est disponible pour un SSD cible plus petit : [Migrer Windows vers un SSD plus petit avec Rescuezilla](/blog/windows-kleinere-ssd-rescuezilla).

## Qu’est-ce que Rescuezilla ?

Rescuezilla est un système live basé sur Ubuntu, qui démarre depuis une clé USB. Il fonctionne indépendamment du système d’exploitation installé et sauvegarde donc aussi les volumes que Windows verrouille pendant son fonctionnement. Le projet est né en 2019 sous la forme d’un fork de Redo Backup and Recovery, qui n’était alors plus maintenu depuis sept ans. Depuis la version 2.0 (2020), Rescuezilla écrit des images au format Clonezilla : une sauvegarde créée avec Rescuezilla peut être restaurée avec Clonezilla, et inversement. La licence est GPL-3.0. La version actuelle est la 2.6.2 de mai 2026, basée sur Ubuntu 26.04 LTS avec Partclone 0.3.47.

Partclone effectue le travail proprement dit. Il connaît les systèmes de fichiers courants (NTFS, FAT, ext4 et autres) et ne lit que les blocs occupés. Une partition de 1 To contenant 350 Go de données produit ainsi une image d’environ 215 Go (compressée avec gzip), et non de 1 To. Les partitions sans système de fichiers reconnu, comme la partition réservée Microsoft, sont sauvegardées bloc par bloc par Rescuezilla avec `dd`; ces fichiers portent l’extension `.dd-ptcl-img` dans l’image.

<details class="options-details">
<summary>Fonctions en un coup d’œil</summary>

| Fonction | Objectif |
|---|---|
| Backup | sauvegarde les partitions sélectionnées d’un disque, y compris la table de partitions, sous forme d’image sur un lecteur local ou un partage réseau (SMB, SSH) |
| Restore | restaure une image sur un disque, avec ou sans écrasement de la table de partitions |
| Verify Image | vérifie qu’une image existante est complète et lisible |
| Clone | copie directement un disque vers un second, sans stockage intermédiaire |
| Image Explorer (beta) | monte une image en lecture seule pour récupérer des fichiers individuels |
| Images de VM | lit, en plus des images Clonezilla, les images VDI, VMDK, VHDX, QCOW2 et Raw |
| Outils supplémentaires | GParted, gestionnaire de fichiers, navigateur web et outils de récupération de fichiers supprimés sur le bureau live |
| CLI | ligne de commande expérimentale (depuis la version 2.5) pour Backup, Verify, Restore et Clone |

</details>

Une limitation est importante pour les migrations : Rescuezilla ne réduit pas les partitions. Le disque cible doit atteindre au minimum la fin de la dernière partition sauvegardée. S’il est plus petit, une préparation avec GParted est nécessaire, comme décrit dans le guide lié ci-dessus.

## Image ou clonage

Rescuezilla offre deux méthodes de migration. Avec le **clonage**, les disques source et cible sont connectés simultanément, et Rescuezilla copie directement. Cela économise du temps et un troisième support, mais suppose que les deux disques soient connectés simultanément à l’ordinateur ou à un adaptateur. Avec l’**imagerie**, une image est d’abord créée sur un lecteur externe, puis restaurée sur le nouveau disque. Cela prend plus de temps, mais présente un avantage : l’image demeure une sauvegarde complète de l’ancien état, même si quelque chose se passe mal lors de la restauration. Pour une migration, l’image est donc le choix le plus sûr, et c’est cette méthode que décrit l’article.

## Prérequis

1.  **Clé USB** pour Rescuezilla. Son contenu sera effacé lors de l’écriture de l’image de démarrage.

2.  **Lecteur externe** pour l’image, avec un espace libre correspondant approximativement à la taille des données occupées. exFAT et NTFS fonctionnent tous deux.

3.  **Disque cible**, au moins aussi grand que le disque source. S’il est plus petit, commencez par suivre le guide pour les SSD plus petits.

4.  **Clé de récupération BitLocker**, si C: est chiffré. Elle est accessible sur aka.ms/myrecoverykey ou dans le portail Entra pour les appareils gérés.

## Étape 1 : préparer Windows

Avant la sauvegarde, vérifiez trois points dans une PowerShell exécutée avec des droits d’administrateur.

**BitLocker.** Partclone ne peut pas lire un volume BitLocker comme un volume NTFS. Rescuezilla le sauvegarde alors bloc par bloc, comme toute partition sans système de fichiers reconnu ; l’image a alors la taille de toute la partition. Pour une migration, il est recommandé de désactiver BitLocker au préalable et de le réactiver après le transfert :

```powershell
manage-bde -status C:
manage-bde -off C:
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-status` | affiche le niveau de chiffrement et l’état de protection du volume |
| `-off` | déchiffre entièrement le volume ; continue en arrière-plan |
| `C:` | argument positionnel : le volume concerné |

</details>

Attendez que `manage-bde -status C:` affiche la valeur `Fully Decrypted`. Sur les appareils gérés via Intune, une stratégie peut réactiver BitLocker ; vérifiez donc à nouveau l’état juste avant le redémarrage.

**Mise en veille prolongée et démarrage rapide.** Lorsque le démarrage rapide est activé, Windows ne s’arrête pas complètement et NTFS est considéré comme toujours utilisé. Partclone s’interrompt alors. Une seule commande désactive les deux :

```powershell
powercfg /h off
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `/h off` | forme abrégée de `/hibernate off`: désactive la mise en veille prolongée et le démarrage rapide, `hiberfil.sys` est supprimé |

</details>

**État du système de fichiers.** Si le Dirty Bit est défini, Partclone s’interrompt avec le message indiquant que le volume est « scheduled for a check or it was shutdown uncleanly ». Vérifiez-le au préalable :

```powershell
fsutil dirty query C:
```

Si la commande indique `is Dirty`, planifiez une vérification pour le prochain démarrage avec `chkdsk C: /f`, redémarrez puis vérifiez à nouveau.

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `dirty query C:` | fsutil : vérifie si le Dirty Bit du volume est défini |
| `/f` | chkdsk : corrige les erreurs ; sur le volume système, la vérification est planifiée au prochain démarrage |

</details>

## Étape 2 : créer et démarrer la clé USB amorçable

Téléchargez l’image ISO depuis la page GitHub des releases du projet. La variante standard porte le nom de code Ubuntu dans son nom de fichier, pour la version 2.6.2 `rescuezilla-2.6.2-64bit.resolute.iso`. Écrivez-la sur la clé USB avec un utilitaire d’écriture d’images tel que balenaEtcher ; la page du projet recommande ce programme pour Windows, macOS et Linux.

Windows permet lui-même de lancer le démarrage depuis la clé :

```powershell
shutdown /r /o /t 0
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `/r` | redémarre au lieu d’éteindre |
| `/o` | démarre dans les options de démarrage avancées ; uniquement avec `/r` |
| `/t 0` | délai en secondes avant l’exécution |

</details>

Dans les options de démarrage avancées, sélectionnez « Utiliser un périphérique » puis la clé USB. Vous pouvez également ouvrir le menu de démarrage du firmware à la mise sous tension (selon le fabricant : F8, F11 ou F12). Rescuezilla démarre avec Secure Boot activé ; il n’est pas nécessaire de le désactiver. Si l’écran reste noir après la sélection de la langue, l’entrée « Graphical Fallback Mode » du menu de démarrage de la clé peut aider.

## Étape 3 : créer la sauvegarde

Rescuezilla démarre automatiquement sur le bureau. L’assistant guide la sauvegarde en étapes numérotées :

1.  Sélectionnez **Backup**.

2.  *Step 1: Select Drive To Backup* : sélectionnez le disque système. Le modèle et la taille aident à les distinguer ; les noms de périphériques (`nvme0n1`, `sda`) peuvent changer entre deux démarrages.

3.  *Step 2: Select Partitions to Save* : laissez toutes les partitions cochées. Une restauration amorçable nécessite la partition système EFI, la partition réservée Microsoft, C: et la partition de récupération.

4.  *Step 3: Select Destination Drive* : le lecteur externe (Local) ou un partage réseau (Network).

5.  *Step 4: Select Destination Folder* : le dossier de destination sur le lecteur. Rescuezilla y crée un sous-dossier horodaté.

6.  *Step 5: Name Your Backup* : ajoutez éventuellement une description ; elle apparaîtra plus tard dans la liste des images.

7.  *Step 6: Customize Compression Settings* : le réglage gzip par défaut convient dans la plupart des cas.

8.  *Step 7: Confirm Backup Configuration* : vérifiez les informations et démarrez.

Pour 350 Go de données sur un SSD NVMe, la sauvegarde vers un SSD USB externe a duré environ 20 minutes. À la fin, Rescuezilla indique pour chaque partition si elle a été sauvegardée avec succès. Si une erreur est indiquée pour une partition, l’image est incomplète, même si le dossier existe.

## Étape 4 : vérifier l’image

Vérifiez l’image avant la restauration. **Verify Image** dans le menu principal lit toutes les parties et vérifie qu’elles sont complètes et lisibles. Dans la liste des images, la colonne **Partitions** indique les partitions sauvegardées avec leur taille ; un triangle d’avertissement jaune signale les images dont des partitions sont manquantes.

Le dossier de l’image peut également être consulté sous Windows. Les fichiers les plus importants :

| Fichier | Contenu |
|---|---|
| `nvme0n1-pt.parted` | table de partitions au format texte, avec le début et la fin de chaque partition en secteurs |
| `nvme0n1-gpt-1st`, `nvme0n1-gpt-2nd` | copies binaires de la GPT principale et de sauvegarde |
| `nvme0n1p3.ntfs-ptcl-img.gz.aa`, `.ab`, … | image Partclone de C:, compressée avec gzip et découpée en parties de 4 Go |
| `nvme0n1p2.dd-ptcl-img.gz.aa` | copie bloc par bloc d’une partition sans système de fichiers reconnu |
| `clonezilla-img` | journal de la sauvegarde avec le résultat pour chaque partition |
| `Info-*.txt` | informations matérielles et SMART du système source au moment de la sauvegarde |

`nvme0n1-pt.parted` permet de déterminer si l’image tient sur le disque cible : le secteur final de la dernière partition plus un, multiplié par 512 octets, doit être inférieur à la taille du disque cible en octets.

## Étape 5 : restaurer sur le nouveau SSD

Installez le nouveau SSD puis démarrez à nouveau Rescuezilla.

1.  Sélectionnez **Restore**.

2.  *Step 1: Select Image Location* : le lecteur externe contenant l’image.

3.  *Step 2: Select Backup Image* : sélectionnez l’image à l’aide de la date et de la taille des partitions.

4.  *Step 3: Select Drive To Restore* : le nouveau SSD. Vérifiez deux fois le modèle et la taille : toutes les données présentes sur ce lecteur seront écrasées.

5.  *Step 4: Select Partitions to Restore* : laissez toutes les partitions cochées, ainsi que **Overwrite partition table**.

6.  *Step 5: Confirm Restore Configuration* : vérifiez et démarrez.

La restauration dure environ aussi longtemps que la sauvegarde. Éteignez ensuite l’ordinateur et retirez la clé.

## Étape 6 : démarrer depuis le nouveau lecteur

Le plus simple est de déconnecter l’ancien disque avant le premier démarrage. Si les deux disques sont connectés, ils portent les mêmes ID de partition et de volume, et le firmware peut démarrer sur l’ancien. Si l’ancien disque doit continuer à être utilisé, formatez-le uniquement après avoir vérifié le nouveau système.

Si le nouveau SSD est plus grand que l’ancien, l’espace supplémentaire se trouve sous forme de zone non allouée à la fin. Il n’est toutefois pas possible d’étendre directement C: dans la Gestion des disques, car la partition de récupération se trouve entre C: et l’espace libre. Avec GParted sur la clé Rescuezilla, déplacez d’abord la partition de récupération à la fin (Resize/Move, **Free space following** à 0), puis agrandissez C: sur tout l’espace libre. Windows RE retrouve ensuite sa partition, car celle-ci conserve son numéro de partition ; `reagentc /info` affiche l’état.

Si l’ordinateur change également lors de la migration, Windows démarre généralement sans ajustements et installe ensuite les pilotes manquants. L’activation s’effectue via la licence numérique ; en cas de changement de carte mère, elle peut nécessiter une connexion au compte Microsoft via l’utilitaire de résolution des problèmes d’activation.

## Étapes finales

Sur le nouveau lecteur, réactivez ce qui a été désactivé pour la migration. La mise en veille prolongée est activée par `powercfg /h on`. Activez BitLocker via Paramètres → Confidentialité et sécurité → Chiffrement de l’appareil ou avec `manage-bde -on C:`; vérifiez ensuite avec `manage-bde -protectors -get C:` qu’une clé de récupération est présente et sauvegardée.

Conservez l’image sur le lecteur externe jusqu’à ce que Windows ait fonctionné quelques jours sans anomalie sur le nouveau lecteur et que toutes les données aient été vérifiées. Elle peut ensuite être supprimée ou archivée comme sauvegarde de l’état de livraison.

## Sources

1.  [Rescuezilla](https://rescuezilla.com/): page du projet avec aperçu des fonctions, téléchargement et FAQ.

2.  [Rescuezilla sur GitHub](https://github.com/rescuezilla/rescuezilla): code source, historique du projet et liste des fonctions dans le README.

3.  [Rescuezilla Releases](https://github.com/rescuezilla/rescuezilla/releases/latest): version actuelle avec notes de publication, liste de compatibilité et variantes ISO.

4.  [Rescuezilla Changelog](https://raw.githubusercontent.com/rescuezilla/rescuezilla/master/CHANGELOG.md): introduction de CLI, Verify Image et Clone, ainsi que mise à jour du shim Secure Boot.

5.  [Rescuezilla Wiki: Restoring to a smaller disk](https://github.com/rescuezilla/rescuezilla/wiki/HOWTO:-Restoring-to-a-smaller-disk.-Eg,-1000GB-HDD-to-500GB-SSD): procédure officielle pour les disques cibles plus petits que l’original.

6.  [Partclone](https://partclone.org/): outil utilisé par Rescuezilla et Clonezilla pour les sauvegardes tenant compte du système de fichiers.

7.  [GParted Manual](https://gparted.org/display-doc.php?name=help-manual): déplacement et agrandissement des partitions.

8.  [Microsoft Learn: manage-bde status](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-status): consultation de l’état de BitLocker.

9.  [Microsoft Learn: manage-bde off](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-off): déchiffrement d’un volume.

10.  [Microsoft Learn: Powercfg command-line options](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/powercfg-command-line-options): option `/hibernate` et son influence sur le démarrage rapide.

11.  [Microsoft Learn: fsutil dirty](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/fsutil-dirty): interrogation du Dirty Bit.

12.  [Microsoft Learn: chkdsk](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/chkdsk): vérification et réparation des volumes NTFS.

13.  [Microsoft Learn: shutdown](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): paramètre `/o` pour démarrer dans les options de démarrage avancées.

14.  [Microsoft Support: Reactivating Windows after a hardware change](https://support.microsoft.com/en-us/windows/reactivating-windows-after-a-hardware-change-2c0e962a-f04c-145b-6ead-fb3fc72b6665): activation par licence numérique après un changement de carte mère.
