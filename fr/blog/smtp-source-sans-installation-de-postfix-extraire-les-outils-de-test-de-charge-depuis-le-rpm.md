---
title: "smtp-source sans installation de Postfix : extraire les outils de test de charge depuis le RPM"
navTitle: "Extraire smtp-source"
description: "smtp-source et smtp-sink font partie de Postfix, mais fonctionnent aussi sans serveur de messagerie installé. Comment extraire les deux outils du paquet sur RHEL, pourquoi leur exécution depuis /tmp peut échouer en raison de l’option de montage noexec et quelles bibliothèques doivent être incluses."
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
url: https://rafaelpfister.ch/fr/blog/smtp-source-sans-installation-de-postfix-extraire-les-outils-de-test-de-charge-depuis-le-rpm
translationSourceHash: 2b4bda3ea22f49c9d5269ec15b0c1dbfd779ccc6d03ad5b234aba738e5bb119f
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:23:40.967Z
translationReview: automatic
---

# smtp-source sans installation de Postfix : extraire les outils de test de charge depuis le RPM

Pour les tests de charge SMTP, `smtp-source` est un bon choix : l’outil ouvre des sessions parallèles, les maintient ouvertes sur plusieurs messages et reproduit ainsi le comportement de connexion d’un expéditeur de masse de manière bien plus réaliste que les outils de test qui établissent une nouvelle connexion pour chaque e-mail. Son pendant, `smtp-sink`, accepte les e-mails et les détruit sans rien distribuer. Les deux font partie de Postfix.

C’est précisément là que réside le problème : Postfix n’est souvent pas installé sur le système depuis lequel vous souhaitez effectuer les tests. Son installation n’est pas non plus souhaitable sur une appliance de passerelle de messagerie, car un Postfix supplémentaire apporte sa propre configuration sous `/etc/postfix` ainsi qu’un service système qui, dans le pire des cas, occupe le port 25 et bloque ainsi le véritable système de messagerie. Il faut aussi se demander ce que le support du fabricant pense des paquets installés ultérieurement sur son appliance.

Les deux outils peuvent toutefois être utilisés sans installation : téléchargez le RPM, extrayez les binaires et les bibliothèques, et le tour est joué. Cette procédure présente deux particularités, illustrées dans cet article sur un système RHEL 8. Vous n’avez pas besoin de droits root, seulement d’un accès aux sources de paquets.

## smtp-source est-il déjà présent ?

Commencez par vérifier si l’outil ne se trouve pas déjà sur le système. Selon la distribution, `smtp-source` se trouve hors du PATH habituel :

```bash
command -v smtp-source || \
  ls /usr/sbin/smtp-source /usr/lib/postfix/sbin/smtp-source 2>/dev/null
```

Si la sortie reste vide, le paquet correspondant est également absent. Sur les systèmes RPM, vous pouvez le confirmer et vérifier simultanément si les dépôts proposent Postfix :

```bash
rpm -qa | grep -i postfix
```

```bash
yum list available postfix
```

Sur le système de test, aucun Postfix n’était installé, mais le dépôt BaseOS proposait `postfix-3.5.8-8.el8_10` . La voie est donc libre : le paquet peut être téléchargé sans être installé.

## Télécharger uniquement le RPM

`yum download` (issu du paquet de plugins `dnf-plugins-core`, généralement présent sur RHEL 8) télécharge un paquet dans le répertoire courant sans l’installer. Cela fonctionne sans droits root tant que le répertoire cible est accessible en écriture :

```bash
cd /tmp && yum download postfix
```

Si yum affiche `No such command: download`, le plugin est absent. Avec des droits root, vous obtenez le même résultat via la commande d’installation avec `--downloadonly` :

```bash
sudo yum install --downloadonly --downloaddir=/tmp postfix
```

Sans l’un ni l’autre, il reste le détour par un second système de même version RHEL : téléchargez le RPM sur celui-ci et copiez-le vers le système cible avec `scp`.

## Extraire les binaires et les bibliothèques

`rpm2cpio` transforme le RPM en un flux d’archive cpio, dont `cpio` extrait sélectivement certains chemins. Outre les deux binaires, vous avez également besoin des bibliothèques Postfix, car sur RHEL les outils sont liés dynamiquement à `libpostfix-*.so` :

```bash
cd /tmp && rpm2cpio postfix-*.rpm | \
  cpio -idmv ./usr/sbin/smtp-source ./usr/sbin/smtp-sink \
  './usr/lib64/postfix/*'
```

Les fichiers se trouvent ensuite sous `/tmp/usr/`. Les indications de chemin commencent par `./`, car cpio attend les chemins exactement tels qu’ils figurent dans l’archive.

## Problème 1 : /tmp est monté avec noexec

Le lancement direct depuis /tmp, qui semble évident, échoue sur les systèmes durcis :

```text
bash: /tmp/usr/sbin/smtp-sink: Permission denied
[1]+  Exit 126
```

Un code de sortie 126 malgré un bit d’exécution correctement défini est le symptôme typique d’un système de fichiers monté avec l’option `noexec`. Le noyau refuse alors toute exécution de programme depuis ce système de fichiers, indépendamment des droits sur les fichiers. Vous pouvez le vérifier directement :

```bash
mount | grep ' /tmp '
```

La solution : copiez les binaires et les bibliothèques dans un répertoire dont le système de fichiers autorise l’exécution, par exemple votre répertoire personnel :

```bash
mkdir -p ~/bin && \
  cp /tmp/usr/sbin/smtp-source /tmp/usr/sbin/smtp-sink \
     /tmp/usr/lib64/postfix/libpostfix-*.so ~/bin/ && \
  chmod +x ~/bin/smtp-source ~/bin/smtp-sink
```

Notez que `noexec` affecte également le chargement des bibliothèques partagées. Il ne suffit donc pas de déplacer uniquement les binaires et de laisser les bibliothèques dans /tmp.

## Problème 2 : le chemin des bibliothèques

Sans indication supplémentaire, l’éditeur de liens dynamique recherche les bibliothèques Postfix sous `/usr/lib64/postfix`, où elles ne se trouvent pas faute d’installation :

```text
smtp-sink: error while loading shared libraries: libpostfix-global.so:
cannot open shared object file: No such file or directory
```

`LD_LIBRARY_PATH` ajoute le répertoire personnel au chemin de recherche de l’éditeur de liens. La variable précède chaque appel :

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-source ...
```

Avec `ldd ~/bin/smtp-source`, vous pouvez vérifier à l’avance si toutes les dépendances sont résolubles. Hormis les bibliothèques Postfix, les outils ne dépendent que des bibliothèques standard du système.

## Test de fonctionnement en boucle locale

Vous pouvez vérifier que tout fonctionne sans envoyer un seul véritable e-mail : `smtp-sink` écoute en tant que destinataire jetable sur un port élevé, tandis que `smtp-source` envoie les messages. Tout le trafic reste sur localhost.

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-sink -v 127.0.0.1:2525 100 &
```

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-source -s 2 -m 10 -l 5120 \
  -f test@example.com -t test@example.com 127.0.0.1:2525
```

| Option | Effet |
|---|---|
| `-v` (smtp-sink) | Journalise chaque étape du dialogue des connexions acceptées |
| `127.0.0.1:2525` | smtp-sink écoute uniquement sur localhost, port 2525 |
| `100` | Backlog : longueur maximale de la file d’attente des connexions en attente selon listen(2) |
| `-s 2` | Deux sessions SMTP parallèles |
| `-m 10` | Dix messages au total, répartis entre les sessions |
| `-l 5120` | Taille du message en octets (sans en-têtes), ici 5 Ko |
| `-f` / `-t` | Adresses de l’expéditeur et du destinataire |

En cas de succès, `smtp-source` ne produit aucune sortie, tandis que smtp-sink affiche le dialogue SMTP complet, de `HELO` à `QUIT`, pour chaque message. Arrêtez ensuite le processus en arrière-plan et supprimez les restes dans /tmp :

```bash
kill %1
```

```bash
rm -rf /tmp/usr /tmp/postfix-*.rpm
```

## Conseils pour le véritable test de charge

Pour obtenir des mesures de débit fiables, le générateur de charge doit se trouver sur une machine distincte dans le même segment réseau, et non sur l’objet testé lui-même. Si `smtp-source` s’exécute sur la passerelle examinée, le générateur et le système de messagerie se disputent le CPU et les E/S, et la mesure reflète cette concurrence au lieu de la capacité réelle. Localement sur le système cible, l’outil extrait convient surtout aux tests fonctionnels des règles et aux premières vérifications de plausibilité.

Dès que le test cible le véritable port 25, il s’agit de vrais e-mails qui traversent les règles de la passerelle et sont distribués selon la configuration. Utilisez donc des adresses de destinataires contrôlées : une boîte aux lettres de test dédiée, une règle qui détruit les expéditeurs de test ou un domaine de rejet prévu à cet effet par le fournisseur. Les adresses de production n’ont pas leur place dans un test de charge.

La procédure décrite convient, au-delà de ces deux outils SMTP, à tout programme en ligne de commande fourni par un paquet dont l’installation sur le système cible n’est pas envisageable. La combinaison de `yum download`, `rpm2cpio` et d’un répertoire exécutable dans le répertoire personnel est identique sur tous les systèmes RPM.

## Sources

1.  [Postfix: smtp-source(1)](https://www.postfix.org/smtp-source.1.html): page de manuel contenant tous les paramètres du générateur de charge, y compris la gestion des sessions et des messages.

2.  [Postfix: smtp-sink(1)](https://www.postfix.org/smtp-sink.1.html): page de manuel du récepteur de test, notamment avec des options pour les délais artificiels et les réponses d’erreur.

3.  [Red Hat: How to download a package without installing it](https://access.redhat.com/solutions/10154): documente `yum download` et l’alternative `--downloadonly`.

4.  [man7.org: mount(8)](https://man7.org/linux/man-pages/man8/mount.8.html): description de l’option de montage `noexec` et de son effet sur l’exécution des programmes.
