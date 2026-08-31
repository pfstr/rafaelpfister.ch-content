---
title: "smtp-source sans installation de Postfix : extraire les outils de test de charge du RPM"
navTitle: "Extraire smtp-source"
description: "smtp-source et smtp-sink font partie de Postfix, mais fonctionnent aussi sans serveur de messagerie installé. Découvrez comment extraire ces deux outils du paquet sur RHEL, pourquoi une exécution depuis /tmp peut échouer à cause de l’option de montage noexec et quelles bibliothèques doivent être incluses."
date: "2026-08-27"
kategorie: "SMTP et flux de messagerie"
timeToRead: "7 min de lecture"
themen:
  - smtp-mailflow
  - smtp-lasttests
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "testing"
  - "troubleshooting"
slug: "smtp-source-sans-installation-de-postfix-extraire-les-outils-de-test-de-charge-depuis-le-rpm"
translationId: "article-d0a27da11509d24b"
translationOf: smtp-source-ohne-postfix-installation
translationSourceHash: fd4ad6beb5036817db9b7758653a2b7d015a6adb15d7b4a0b47f94161e34b4e6
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:52:20.514Z
translationReview: automatic
url: https://rafaelpfister.ch/fr/blog/smtp-source-sans-installation-de-postfix-extraire-les-outils-de-test-de-charge-depuis-le-rpm
---

# smtp-source sans installation de Postfix : extraire les outils de test de charge du RPM

Pour les tests de charge SMTP, `smtp-source` est un bon choix : l’outil ouvre des sessions parallèles, les maintient ouvertes pour plusieurs messages et reproduit ainsi le comportement de connexion d’un expéditeur de masse bien plus réalistement que les outils de test qui établissent une nouvelle connexion pour chaque e-mail. Son pendant, `smtp-sink`, accepte les e-mails et les rejette sans rien distribuer. Les deux font partie de Postfix.

C’est précisément là que se trouve le problème : le système depuis lequel vous souhaitez effectuer les tests n’a souvent pas Postfix installé. Sur une appliance de passerelle de messagerie, une installation n’est pas non plus souhaitable, car un Postfix supplémentaire apporte sa propre configuration sous `/etc/postfix` ainsi qu’un service système qui, dans le pire des cas, occupe le port 25 et bloque ainsi le système de messagerie proprement dit. S’ajoute à cela la question de savoir ce que le support du fabricant pense des paquets installés ultérieurement sur son appliance.

Ces deux outils peuvent toutefois être utilisés sans installation : téléchargez le RPM, extrayez les binaires et les bibliothèques, et c’est tout. Ce chemin présente deux particularités que cet article illustre sur un système RHEL 8. Vous n’avez pas besoin de droits root, seulement d’un accès aux sources de paquets.

## smtp-source est-il déjà présent ?

Commencez par vérifier si l’outil ne se trouve pas déjà sur le système. Selon la distribution, `smtp-source` se trouve hors du PATH habituel :

```bash
command -v smtp-source || \
  ls /usr/sbin/smtp-source /usr/lib/postfix/sbin/smtp-source 2>/dev/null
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `command -v smtp-source` | Affiche le chemin si le programme se trouve dans le PATH ; sinon, n’affiche rien |
| `/usr/sbin/... /usr/lib/postfix/sbin/...` | Les emplacements habituels hors du PATH (RHEL ou Debian/Ubuntu) |
| `2>/dev/null` | Supprime les messages d’erreur de `ls` pour les chemins inexistants |

</details>

Si la sortie reste vide, le paquet correspondant est également absent. Sur les systèmes RPM, confirmez-le et vérifiez en même temps si les référentiels proposent Postfix :

```bash
rpm -qa | grep -i postfix
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-q` | Mode d’interrogation de rpm |
| `-a` | Liste tous les paquets installés |
| `grep -i postfix` | Filtre la liste sans tenir compte de la casse |

</details>

```bash
yum list available postfix
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `list available` | Affiche uniquement les paquets disponibles dans les référentiels mais non installés |
| `postfix` | Limite la sortie au paquet recherché |

</details>

Sur le système de test, aucun Postfix n’était installé, mais le référentiel BaseOS proposait `postfix-3.5.8-8.el8_10`. La voie est donc libre : le paquet peut être téléchargé sans être installé.

## Télécharger uniquement le RPM

`yum download` (issu du paquet de plugin `dnf-plugins-core`, généralement présent sur RHEL 8) télécharge un paquet dans le répertoire actuel sans l’installer. Cela fonctionne sans droits root tant que le répertoire cible est accessible en écriture :

```bash
cd /tmp && yum download postfix
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `cd /tmp` | Se place dans un répertoire accessible en écriture ; `yum download` y dépose le RPM |
| `download` | Sous-commande de `dnf-plugins-core` : télécharge le paquet sans l’installer |
| `postfix` | Nom du paquet à télécharger |

</details>

Si yum affiche `No such command: download`, le plugin manque. Avec des droits root, vous obtenez le même résultat via la commande d’installation avec `--downloadonly` :

```bash
sudo yum install --downloadonly --downloaddir=/tmp postfix
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `--downloadonly` | S’arrête après le téléchargement, rien n’est installé |
| `--downloaddir=/tmp` | Répertoire cible du RPM téléchargé |
| `postfix` | Nom du paquet |

</details>

Sans aucune de ces deux possibilités, il reste le détour par un deuxième système de même version RHEL : téléchargez le RPM sur celui-ci et copiez-le vers le système cible avec `scp`.

## Extraire les binaires et les bibliothèques

`rpm2cpio` convertit le RPM en un flux d’archive cpio, dont `cpio` extrait de manière ciblée certains chemins. Outre les deux binaires, vous avez également besoin des bibliothèques Postfix, car sur RHEL les outils sont liés dynamiquement à `libpostfix-*.so` :

```bash
cd /tmp && rpm2cpio postfix-*.rpm | \
  cpio -idmv ./usr/sbin/smtp-source ./usr/sbin/smtp-sink \
  './usr/lib64/postfix/*'
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `rpm2cpio postfix-*.rpm` | Convertit le RPM en un flux d’archive cpio sur stdout |
| `-i` | Mode d’extraction cpio (copy-in) |
| `-d` | Crée les répertoires manquants lors de l’extraction |
| `-m` | Conserve les dates de modification des fichiers |
| `-v` | Liste chaque fichier extrait |
| `./usr/sbin/smtp-source ./usr/sbin/smtp-sink` | Les deux binaires, chemins exactement tels que dans l’archive (avec le `./` initial) |
| `'./usr/lib64/postfix/*'` | Les bibliothèques Postfix ; le motif est entre guillemets afin que cpio l’évalue et non le shell |

</details>

Les fichiers se trouvent ensuite sous `/tmp/usr/`.

## Problème 1 : /tmp est monté avec noexec

Le démarrage évident directement depuis /tmp échoue sur les systèmes durcis :

```text
bash: /tmp/usr/sbin/smtp-sink: Permission denied
[1]+  Exit 126
```

Un code de sortie 126 malgré un bit d’exécution correctement défini est le symptôme typique d’un système de fichiers monté avec l’option `noexec`. Le noyau refuse alors toute exécution de programme depuis ce système de fichiers, indépendamment des droits de fichier. Vous pouvez le vérifier directement :

```bash
mount | grep ' /tmp '
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `mount` | Sans argument, liste tous les systèmes de fichiers montés avec leurs options de montage |
| `' /tmp '` | Motif de recherche avec un espace avant et après, afin de correspondre uniquement au point de montage `/tmp` et non, par exemple, à `/var/tmp` |

</details>

La solution : copiez les binaires et les bibliothèques dans un répertoire dont le système de fichiers autorise l’exécution, par exemple votre propre répertoire personnel :

```bash
mkdir -p ~/bin && \
  cp /tmp/usr/sbin/smtp-source /tmp/usr/sbin/smtp-sink \
     /tmp/usr/lib64/postfix/libpostfix-*.so ~/bin/ && \
  chmod +x ~/bin/smtp-source ~/bin/smtp-sink
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `mkdir -p ~/bin` | Crée le répertoire cible ; sans erreur s’il existe déjà |
| `cp ... ~/bin/` | Copie les deux binaires et les bibliothèques `libpostfix-*.so` dans le répertoire exécutable |
| `chmod +x` | Définit le bit d’exécution sur les deux binaires |

</details>

Notez que `noexec` affecte également le chargement des bibliothèques partagées. Il ne suffit donc pas de déplacer uniquement les binaires et de laisser les bibliothèques dans /tmp.

## Problème 2 : le chemin des bibliothèques

Sans indication supplémentaire, l’éditeur de liens dynamique cherche les bibliothèques Postfix sous `/usr/lib64/postfix`, où elles ne se trouvent pas faute d’installation :

```text
smtp-sink: error while loading shared libraries: libpostfix-global.so:
cannot open shared object file: No such file or directory
```

`LD_LIBRARY_PATH` ajoute le répertoire personnel au chemin de recherche de l’éditeur de liens. La variable est placée devant chaque appel :

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-source ...
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `LD_LIBRARY_PATH=~/bin` | Ajoute `~/bin` au chemin de recherche de l’éditeur de liens dynamique pour cet unique appel |
| `~/bin/smtp-source` | Appel via le chemin complet, car `~/bin` ne doit pas nécessairement être dans le PATH |

</details>

Avec `ldd ~/bin/smtp-source`, vous pouvez vérifier à l’avance si toutes les dépendances peuvent être résolues. En dehors des bibliothèques Postfix, les outils ne dépendent que des bibliothèques standard du système.

## Test de fonctionnement en loopback

Vous pouvez vérifier que tout fonctionne sans envoyer un seul véritable e-mail : `smtp-sink` écoute comme destinataire jetable sur un port élevé, tandis que `smtp-source` envoie les messages. Tout le trafic reste sur localhost.

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-sink -v 127.0.0.1:2525 100 &
```

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-source -s 2 -m 10 -l 5120 \
  -f test@example.com -t test@example.com 127.0.0.1:2525
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-v` (smtp-sink) | Journalise chaque étape du dialogue des connexions acceptées |
| `127.0.0.1:2525` | smtp-sink écoute uniquement sur localhost, port 2525 |
| `100` | Backlog : longueur maximale de la file d’attente des connexions en attente selon listen(2) |
| `-s 2` | Deux sessions SMTP parallèles |
| `-m 10` | Dix messages au total, répartis entre les sessions |
| `-l 5120` | Taille du message en octets (sans en-tête), ici 5 Ko |
| `-f` / `-t` | Adresse de l’expéditeur et du destinataire |

</details>

En cas de réussite, `smtp-source` ne produit aucune sortie, tandis que smtp-sink affiche pour chaque message le dialogue SMTP complet, de `HELO` à `QUIT`. Ensuite, arrêtez le processus d’arrière-plan et supprimez les restes dans /tmp :

```bash
kill %1
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `%1` | Désignation de tâche du shell : arrête la première tâche en arrière-plan, ici smtp-sink |

</details>

```bash
rm -rf /tmp/usr /tmp/postfix-*.rpm
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-r` | Supprime l’arborescence de répertoires récursivement |
| `-f` | Aucune confirmation, aucune erreur pour les chemins inexistants |
| `/tmp/usr /tmp/postfix-*.rpm` | L’arborescence extraite et le RPM téléchargé |

</details>

## Conseils pour le véritable test de charge

Pour des mesures de débit fiables, le générateur de charge doit se trouver sur une machine distincte du même segment réseau, et non sur l’objet à tester lui-même. Si `smtp-source` s’exécute sur la passerelle testée, le générateur et le système de messagerie se disputent le CPU et les E/S, et la mesure reflète cette concurrence plutôt que la capacité réelle. Localement sur le système cible, l’outil extrait convient surtout aux tests fonctionnels du jeu de règles et aux premières vérifications de plausibilité.

Dès que le test vise le véritable port 25, il s’agit de vrais e-mails qui traversent le jeu de règles de la passerelle et sont distribués selon la configuration. Utilisez donc des adresses de destinataires dont l’aboutissement est contrôlé : une boîte aux lettres de test dédiée, une règle qui rejette les expéditeurs de test ou un domaine de rejet prévu à cet effet par le fournisseur. Les adresses de production n’ont pas leur place dans un test de charge.

La procédure décrite convient, au-delà de ces deux outils SMTP, à tout programme en ligne de commande fourni dans un paquet dont l’installation sur le système cible n’est pas envisageable. La combinaison de `yum download`, `rpm2cpio` et d’un répertoire exécutable dans le répertoire personnel est identique sur tous les systèmes RPM.

## Sources

1.  [Postfix: smtp-source(1)](https://www.postfix.org/smtp-source.1.html): page de manuel avec tous les paramètres du générateur de charge, y compris le contrôle des sessions et des messages.

2.  [Postfix: smtp-sink(1)](https://www.postfix.org/smtp-sink.1.html): page de manuel du destinataire de test, notamment avec des options pour les délais artificiels et les réponses d’erreur.

3.  [Red Hat: How to download a package without installing it](https://access.redhat.com/solutions/10154): documente `yum download` et l’alternative `--downloadonly`.

4.  [man7.org: mount(8)](https://man7.org/linux/man-pages/man8/mount.8.html): description de l’option de montage `noexec` et de son effet sur l’exécution des programmes.
