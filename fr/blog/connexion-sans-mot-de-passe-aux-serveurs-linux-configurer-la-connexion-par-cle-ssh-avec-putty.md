---
title: "Connexion sans mot de passe aux serveurs Linux : configurer la connexion par clé SSH avec PuTTY, Pageant et autres"
navTitle: "PuTTY sans mot de passe"
description: "Les administrateurs qui accèdent quotidiennement à des serveurs Linux saisissent à chaque fois leur nom d’utilisateur et leur mot de passe. Une paire de clés SSH réduit cela à un double-clic : générer la clé avec PuTTYgen, enregistrer la clé publique sur le serveur, charger Pageant. La même clé fonctionne dans WinSCP, MobaXterm et OpenSSH, et ceux qui le souhaitent arrivent directement dans le shell d’un compte de service."
date: "2026-08-28"
kategorie: "Linux et SSH"
timeToRead: "9 min de lecture"
themen:
  - totemomail
  - windows-client
produkte:
  - "totemomail"
  - "uebergreifend"
protokolle:
  - "ssh"
  - "haertung"
slug: "connexion-sans-mot-de-passe-aux-serveurs-linux-configurer-la-connexion-par-cle-ssh-avec-putty"
translationId: "article-9f94fa6eb8b95bcf"
aiPrompt: |
  Du bist mein Linux- und SSH-Assistent. Hilf mir Schritt für Schritt, einen passwortlosen SSH-Login von Windows auf meine Linux-Server einzurichten: 1. Ein Ed25519-Schlüsselpaar mit PuTTYgen erzeugen und den Public Key in authorized_keys eintragen. 2. Die PuTTY-Session mit Schlüsseldatei und Auto-login username vervollständigen und Pageant mit Autostart einrichten. 3. Optional ein Remote command hinterlegen, das mich direkt in die Shell eines Service-Accounts bringt, samt minimaler NOPASSWD-Regel unter /etc/sudoers.d. Weise mich auf typische Fehler hin: Key im falschen Home-Verzeichnis, mehrzeiliger Public Key, falsche Berechtigungen, sudoers-Befehl stimmt nicht exakt mit dem Remote command überein.
translationOf: putty-ssh-login-service-account-shell
url: https://rafaelpfister.ch/fr/blog/connexion-sans-mot-de-passe-aux-serveurs-linux-configurer-la-connexion-par-cle-ssh-avec-putty
translationSourceHash: e95b27e9a86f59dfb0808afee63664493f5961b983f807f31ef9ee7a36f6fb3e
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:36:10.974Z
translationReview: automatic
---

# Connexion sans mot de passe aux serveurs Linux : configurer la connexion par clé SSH avec PuTTY, Pageant et autres

Les administrateurs qui accèdent quotidiennement à des serveurs Linux répètent les mêmes saisies à chaque connexion par mot de passe : nom d’utilisateur, mot de passe, puis, pour les comptes de service, une commande sudo. Avec une paire de clés SSH, tout cela disparaît. Une fois la configuration effectuée, un double-clic sur la session enregistrée ouvre un shell prêt à l’emploi, et la même clé fonctionne dans PuTTY, WinSCP, MobaXterm et le client OpenSSH de Windows.

Ce guide configure entièrement la connexion sans mot de passe depuis Windows : générer une paire de clés, enregistrer la clé publique sur le serveur, compléter la session PuTTY, configurer Pageant comme agent de clés. Il aborde également le dépannage du problème le plus fréquent (le serveur continue de demander le mot de passe malgré la clé) et, en extension, l’accès direct au shell d’un compte de service tel que `totemo`.

## Pourquoi utiliser une clé plutôt qu’un mot de passe

Le gain de confort est l’effet le plus visible, mais pas le plus important. Une clé Ed25519 est pratiquement insensible aux attaques par force brute, tandis qu’un mot de passe n’est sûr qu’à la hauteur de sa longueur et de la discipline consistant à ne jamais le réutiliser. Sur les serveurs dont les utilisateurs sont entièrement passés aux clés, l’authentification par mot de passe peut être désactivée complètement dans la configuration de sshd (`PasswordAuthentication no`), ce qui rend vaines les tentatives de connexion automatisées depuis Internet. Ne désactivez l’authentification par mot de passe qu’une fois le login par clé confirmé comme fonctionnel et après avoir prévu un second accès (console, deuxième clé).

Le principe : la clé privée reste sur votre ordinateur Windows, la clé publique est enregistrée sur le serveur. Lors de l’établissement de la connexion, le serveur vérifie que l’autre partie possède la clé privée correspondante, sans que celle-ci ne quitte jamais l’ordinateur.

## Étape 1 : générer une paire de clés avec PuTTYgen

1. Démarrez **PuTTYgen** (inclus dans le paquet PuTTY), sélectionnez le type **Ed25519**, cliquez sur **Generate** et déplacez la souris dans la zone.
2. Saisissez une **passphrase** dans les deux champs et enregistrez la clé privée avec **Save private key** dans un fichier `.ppk`.
3. Copiez intégralement le champ de texte en haut (la ligne commençant par `ssh-ed25519 AAAA...`). Il s’agit de la clé publique au format attendu par le serveur.

Enregistrez la clé privée avec une passphrase. Sans passphrase, chaque copie du fichier constitue un accès immédiat au serveur ; avec une passphrase, le fichier seul est inutilisable. Pageant (étape 4) élimine presque entièrement cet inconvénient de confort. Une clé sans passphrase ne convient qu’à l’automatisation non supervisée, pas à la connexion interactive.

## Étape 2 : enregistrer la clé publique sur le serveur

Sur le serveur, connecté avec l’utilisateur avec lequel vous vous connecterez à l’avenir :

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo 'ssh-ed25519 AAAA... kommentar' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod go-w ~
```

<details class="options-details">
<summary>Options expliquées</summary>

| Commande / option | Effet |
|---|---|
| `mkdir -p ~/.ssh` | Crée le répertoire SSH ; `-p` supprime l’erreur s’il existe déjà |
| `chmod 700 ~/.ssh` | Seul le propriétaire peut lire, écrire et accéder au répertoire |
| `echo '...' >> ~/.ssh/authorized_keys` | Ajoute la clé publique comme nouvelle ligne au fichier (`>>` au lieu de `>`, sinon vous écrasez les clés existantes) |
| `chmod 600 ~/.ssh/authorized_keys` | Seul le propriétaire peut lire et écrire le fichier de clés |
| `chmod go-w ~` | Retire au groupe et aux autres le droit d’écriture sur le répertoire personnel |

</details>

La dernière ligne paraît anodine, mais elle est indispensable : un répertoire personnel accessible en écriture par le groupe ou par tous conduit le démon SSH à ignorer silencieusement la clé, et le serveur continue de demander le mot de passe sans en indiquer la raison.

## Étape 3 : compléter la session PuTTY

1. Ouvrez PuTTY, sélectionnez la session enregistrée et chargez-la avec **Load**.
2. **Connection → SSH → Auth → Credentials** : sélectionnez le fichier `.ppk` dans **Private key file for authentication**.
3. **Connection → Data** : saisissez le nom d’utilisateur dans **Auto-login username**, sinon PuTTY le demandera encore à chaque connexion.
4. Revenez à la catégorie **Session**, sélectionnez à nouveau le nom de la session et cliquez sur **Save**.

L’erreur de manipulation la plus fréquente consiste à oublier **Load** avant la modification ou **Save** après. Sans Load, vous ne modifiez que les paramètres par défaut ; sans Save, la modification est perdue au prochain démarrage de PuTTY.

## Étape 4 : Pageant, une passphrase par session Windows

Pageant est l’agent de clés de PuTTY. Il conserve la clé privée déchiffrée en mémoire, de sorte que la passphrase n’est requise qu’une fois par session Windows :

1. Démarrez Pageant (son icône apparaît dans la zone de notification).
2. Faites un clic droit sur l’icône, sélectionnez **Add Key**, choisissez le fichier `.ppk` et saisissez la passphrase.
3. Dès lors, toutes les connexions s’effectuent sans demande jusqu’au prochain redémarrage.

Pour démarrer automatiquement Pageant, placez un raccourci dans le dossier de démarrage automatique (`Win+R`, puis `shell:startup`) et passez la clé comme argument :

```text
"C:\Program Files\PuTTY\pageant.exe" "C:\Pfad\zum\schluessel.ppk"
```

Windows demandera alors la passphrase une fois après la connexion ; le reste de la journée de travail se déroulera sans demande.

## Lorsque le serveur continue de demander le mot de passe

Le dépannage commence dans l’**Event Log** de PuTTY (clic droit sur la barre de titre de la session de terminal). Il indique si la clé a été proposée :

| Constat dans l’Event Log | Cause et solution |
|---|---|
| Aucune entrée concernant une clé publique | Le fichier `.ppk` n’est pas enregistré dans la session sauvegardée, ou la mauvaise session a été modifiée. Chargez la session, définissez la clé, enregistrez-la. |
| `Server refused our key` | Le serveur ne trouve pas ou n’accepte pas la clé : mauvais répertoire personnel, mauvais format ou mauvaises permissions (voir ci-dessous). |
| `Access granted`, puis demande de mot de passe | Le login par clé a fonctionné ; la demande vient d’un programme lancé ensuite, généralement sudo. Voir l’extension ci-dessous. |

Les trois causes les plus fréquentes de `Server refused our key` :

- **Clé dans le mauvais répertoire personnel.** La clé publique doit se trouver dans le fichier `authorized_keys` de l’utilisateur avec lequel la connexion est établie. Si vous êtes déjà passé à un autre compte avec `sudo -u` ou `su` lors de l’ajout, le fichier est placé dans son répertoire personnel plutôt que dans le vôtre. `whoami` avant l’ajout indique dans quel répertoire personnel la clé sera placée.
- **Mauvais format.** La clé publique doit figurer sur une seule ligne dans `authorized_keys`, au format affiché dans le champ de texte supérieur de PuTTYgen. Le fichier issu de **Save public key** a un autre format sur plusieurs lignes (`---- BEGIN SSH2 PUBLIC KEY ----`) et ne fonctionne pas dans `authorized_keys`.
- **Permissions.** `~/.ssh` à `700`, `authorized_keys` à `600`, répertoire personnel non accessible en écriture par le groupe ou par tous.

Si le constat reste incertain, consultez côté serveur `/var/log/secure` ou `journalctl -u sshd`, où le démon SSH indique pourquoi il a rejeté la clé.

## La même clé dans d’autres outils

La configuration sur le serveur est indépendante de l’outil ; la clé peut être réutilisée partout :

| Outil | Configuration |
|---|---|
| **WinSCP** | Utilise directement les fichiers `.ppk` (Connexion → Avancé → SSH → Authentification) et utilise automatiquement Pageant si la clé y est chargée |
| **MobaXterm** | Fichier `.ppk` dans Session settings → SSH → Advanced → Use private key ; comprend également le format OpenSSH |
| **FileZilla** | Indiquez le fichier `.ppk` dans Paramètres → SFTP ou laissez Pageant actif |
| **OpenSSH (Windows Terminal, `ssh`)** | Nécessite le format OpenSSH : exportez-le dans PuTTYgen via **Conversions → Export OpenSSH key** et placez-le dans `~/.ssh/` |

Pour le client OpenSSH, une entrée dans `~/.ssh/config` sur l’ordinateur Windows permet une connexion confortable :

```text
Host mailgw
    HostName server.example.com
    User mmuster
    IdentityFile ~/.ssh/id_ed25519
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Host mailgw` | Alias librement choisi ; `ssh mailgw` suffit ensuite comme commande |
| `HostName` | Nom réel du serveur ou adresse IP |
| `User` | Nom d’utilisateur, correspond à Auto-login username dans PuTTY |
| `IdentityFile` | Chemin vers la clé privée au format OpenSSH |

</details>

## Extension : arriver directement dans le shell du compte de service

De nombreux serveurs Linux dans les environnements d’applications et de messagerie ne sont pas administrés avec le compte personnel, mais via un compte de service : Totemomail s’exécute sous `totemo`, et d’autres passerelles et applications disposent de leurs propres comptes fonctionnels. Après la connexion, la même commande suit donc systématiquement :

```bash
sudo -u totemo /bin/bash -l
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `sudo` | Exécute la commande suivante avec d’autres droits et journalise l’appel |
| `-u totemo` | L’utilisateur cible est `totemo` au lieu de l’utilisateur par défaut `root` |
| `/bin/bash` | La commande à exécuter : un nouveau shell Bash |
| `-l` | Démarre Bash comme shell de connexion ; il charge le profil de l’utilisateur cible (`.bash_profile`, variables d’environnement, chemins) |

</details>

Le paramètre `-l` est décisif pour les comptes de service : sans shell de connexion, les variables d’environnement du profil du compte fonctionnel, telles que les chemins vers les répertoires de l’application ou les installations Java, sont absentes, et les commandes propres à l’application échouent avec des messages d’erreur trompeurs.

La connexion SSH directe au compte de service serait encore plus courte, mais elle n’est généralement pas possible pour de bonnes raisons : les comptes fonctionnels n’ont souvent pas de mot de passe ou disposent d’un shell de connexion verrouillé, et un accès direct partagé par plusieurs personnes supprimerait la traçabilité personnalisée. En passant par sudo, il reste possible de savoir quelle personne a basculé dans le shell du compte de service et à quel moment. L’automatisation suivante ne change rien à cela, elle évite uniquement la saisie manuelle.

### Remote command dans la session PuTTY

PuTTY permet d’enregistrer une commande par session sauvegardée, qui est exécutée après la connexion à la place du shell normal :

1. Chargez la session avec **Load**.
2. Dans l’arborescence de gauche, accédez à **Connection → SSH** (le nœud principal, pas un sous-menu).
3. Saisissez dans le champ **Remote command** : `sudo -u totemo /bin/bash -l`
4. Enregistrez dans **Session**.

Vous devez connaître trois particularités de la commande distante :

- Un `exit` dans le shell du compte de service met entièrement fin à la connexion, au lieu de revenir à votre shell personnel. La commande remplace le shell de connexion, elle ne l’imbrique pas.
- Si vous travaillez occasionnellement sur le serveur avec votre compte personnel, enregistrez une seconde session sans commande distante (chargez la session, videz le champ, enregistrez sous un nouveau nom).
- Les outils de transfert de fichiers tels que WinSCP ou `pscp` ne sont pas concernés. Ils établissent leurs propres connexions et ignorent la commande distante de la session PuTTY.

Si la connexion se ferme immédiatement après son établissement ou si sudo signale l’absence de terminal : vérifiez dans **Connection → SSH → TTY** que **Don't allocate a pseudo-terminal** n’est pas coché. Par défaut, il ne l’est pas. Important pour l’étape 2 ci-dessus : avec une commande distante active, vous êtes déjà le compte de service lors de l’ajout de la clé publique ; la clé doit toutefois être placée dans le répertoire personnel de l’utilisateur personnel.

Dans le client OpenSSH, deux lignes de l’entrée `Host` produisent le même effet : `RequestTTY yes` et `RemoteCommand sudo -u totemo /bin/bash -l`.

### sudo sans demande de mot de passe

Après la clé et la commande distante, il reste une seule saisie : la demande de mot de passe de sudo. Elle ne disparaît qu’avec la configuration sudoers sur le serveur, ce qui exige les droits root. Sur un serveur d’entreprise géré, il s’agit d’une demande à adresser à l’administrateur du serveur, et non d’un paramètre PuTTY.

La règle doit figurer dans un fichier séparé sous `/etc/sudoers.d/` et être modifiée avec `visudo` :

```bash
visudo -f /etc/sudoers.d/totemo-shell
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `visudo` | Ouvre le fichier sudoers dans l’éditeur et vérifie la syntaxe avant l’enregistrement |
| `-f /etc/sudoers.d/totemo-shell` | Modifie le fichier indiqué plutôt que le fichier central `/etc/sudoers` |

</details>

Contenu du fichier, avec le nom d’utilisateur personnel (ici `mmuster` à titre d’exemple) :

```text
mmuster ALL=(totemo) NOPASSWD: /bin/bash -l
```

<details class="options-details">
<summary>Options expliquées</summary>

| Élément | Effet |
|---|---|
| `mmuster` | La règle ne s’applique qu’à cet utilisateur |
| `ALL=` | Sur tous les hôtes (pertinent pour les fichiers sudoers distribués de manière centralisée) |
| `(totemo)` | Uniquement pour les commandes exécutées en tant qu’utilisateur cible `totemo`, pas en tant que root |
| `NOPASSWD:` | Aucune demande de mot de passe pour les commandes suivantes |
| `/bin/bash -l` | Exactement cette commande avec exactement cet argument, rien d’autre |

</details>

Deux points déterminent si la règle s’applique et si elle est acceptable :

- **Correspondance exacte.** La commande de la règle doit correspondre à la commande distante, argument compris. Si PuTTY contient `sudo -u totemo /bin/bash -l`, la règle doit autoriser `/bin/bash -l`. Une règle pour `/bin/bash` sans `-l` ne couvre pas l’appel avec `-l`, et sudo continue de demander le mot de passe.
- **Portée minimale.** La règle autorise une seule commande pour un seul utilisateur cible. Elle n’accorde ni droits root ni accès à des commandes arbitraires. Sous cette forme, il s’agit aussi d’une demande courante et justifiable sur les serveurs gérés. La journalisation sudo reste intégralement conservée ; chaque basculement dans le shell du compte de service figure toujours dans le journal.

`visudo` n’est pas facultatif : il vérifie la syntaxe avant l’enregistrement. Une faute de frappe écrite directement dans le fichier peut rendre sudo inutilisable pour tous les utilisateurs du système. Pour la même raison, un fichier séparé sous `/etc/sudoers.d/` est préférable à la modification du fichier central `/etc/sudoers` ; il survit aux mises à jour de paquets et peut être supprimé sans risque.

## Le résultat

Après la configuration, la connexion se présente ainsi : double-cliquez sur la session PuTTY et le shell est prêt. Aucun nom d’utilisateur, aucun mot de passe, et avec l’extension, aucune demande sudo non plus. La sécurité ne s’est pas dégradée ; elle s’est même améliorée sur un point :

| Aspect | Avant | Après |
|---|---|---|
| Authentification | Mot de passe à chaque connexion | Clé Ed25519 avec passphrase, conservée dans Pageant |
| Identité de connexion | Compte personnel | Compte personnel inchangé |
| Journalisation sudo | Chaque basculement dans le journal | Chaque basculement dans le journal, inchangé |
| Portée de NOPASSWD | Aucune | Une commande, un utilisateur cible, pas de root |

## Sources

1.  [PuTTY User Manual, chapitre 4 : Configuring PuTTY](https://the.earth.li/~sgtatham/putty/0.83/htmldoc/Chapter4.html): documentation des paramètres de session, notamment Remote command (panneau SSH), Auto-login username (panneau Data) et pseudo-terminal (panneau TTY).

2.  [PuTTY User Manual, chapitre 8 : Using public keys for SSH authentication](https://the.earth.li/~sgtatham/putty/0.83/htmldoc/Chapter8.html): PuTTYgen, types de clés, passphrases, export OpenSSH et ajout de la clé publique sur le serveur.

3.  [PuTTY User Manual, chapitre 9 : Using Pageant for authentication](https://the.earth.li/~sgtatham/putty/0.83/htmldoc/Chapter9.html): fonctionnement de l’agent, chargement des clés et démarrage avec une clé comme argument de ligne de commande.

4.  [Manuel ssh_config(5)](https://man.openbsd.org/ssh_config.5): configuration cliente du client OpenSSH, notamment les alias d’hôte, IdentityFile, RequestTTY et RemoteCommand.

5.  [Manuel sudoers(5)](https://www.sudo.ws/docs/man/sudoers.man/): syntaxe des règles sudoers, spécification Runas et balise NOPASSWD.

6.  [Manuel sshd(8), section AUTHORIZED_KEYS FILE FORMAT](https://man.openbsd.org/sshd.8): format du fichier authorized_keys et exigences concernant les permissions des fichiers.
