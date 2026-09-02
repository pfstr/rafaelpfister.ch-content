---
title: "Renouveler le certificat sur la Cisco SMA"
navTitle: "Certificat SMA"
description: "Les certificats ne peuvent être installés sur la Cisco SMA que via la CLI, et les versions récentes d’AsyncOS valident toute la chaîne lors de l’importation : sans Root-CA enregistrée, l’opération échoue. Cet article présente les moyens d’obtenir une nouvelle paire de clés, la méthode OpenSSL en détail, la gestion de l’erreur RC2-40-CBC d’OpenSSL 3 et l’importation de la Root-CA interne dans le magasin de certificats de confiance de l’appliance."
date: "2026-08-04"
kategorie: "Cisco ESA / SMA"
timeToRead: "11 min de lecture"
themen:
  - cisco-esa-sma
  - smtp-mailflow
hauptthema: "cisco-esa-sma"
slug: "renouveler-le-certificat-sur-la-cisco-sma"
translationId: "article-69d93a1e5e081848"
aiPrompt: |
  Du bist mein Assistent für die Zertifikatserneuerung auf einer Cisco SMA (Secure Email and Web Manager). Führe mich Schritt für Schritt durch den Ablauf aus diesem Artikel: 1. Wahl des Wegs zum Schlüsselpaar (OpenSSL-CSR in der eigenen Umgebung, PFX von der CA oder Umweg über eine ESA), 2. CN- und SAN-Liste für meine Hostnamen, 3. je nach Weg CSR-Erzeugung mit OpenSSL oder Konvertierung der PFX-Datei nach PEM inklusive Umgang mit dem Fehler RC2-40-CBC, 4. bei interner CA Import der Root-CA in die Custom-Liste der Appliance, 5. Installation über certconfig in der CLI, 6. Kontrolle. Frage mich zuerst nach den Hostnamen meiner Appliances und der Quarantäneseite, ob die ausstellende CA intern oder öffentlich ist und welche OpenSSL-Version ich installiert habe. Passe alle Befehle an meine Dateinamen an und erinnere mich vor dem Abschluss daran, die certconfig-Session nicht mit Ctrl+C zu beenden und die Änderung mit commit zu aktivieren.
translationOf: cisco-sma-zertifikat-erneuern
translationSourceHash: c99ce64a5e63875b84c7b6f14a7f2fb7e51290fedbdc93d99201cdc97a743508
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T09:07:11.228Z
translationReview: automatic
url: https://rafaelpfister.ch/fr/blog/renouveler-le-certificat-sur-la-cisco-sma
---

# Renouveler le certificat sur la Cisco SMA

La Cisco SMA (Security Management Appliance, désormais commercialisée sous le nom Cisco Secure Email and Web Manager) assure dans de nombreux environnements de messagerie la quarantaine centrale des spams et le reporting pour les Secure Email Gateways. Son certificat HTTPS couvre la GUI d’administration et la page de quarantaine, sur laquelle les utilisateurs finaux consultent et libèrent leurs e-mails retenus. Lorsqu’il expire, le flux de messagerie ne s’interrompt pas. L’expiration devient néanmoins immédiatement visible : chaque accès à la page de quarantaine aboutit à un avertissement de certificat dans le navigateur, et les utilisateurs à qui les formations de sensibilisation apprennent précisément à ne pas cliquer au-delà de tels avertissements sont alors censés les ignorer.

Lors d’un renouvellement dans le cadre d’un projet client, deux problèmes se sont présentés : OpenSSL 3 a d’abord répondu au fichier PFX de la CA interne par une erreur cryptique concernant `RC2-40-CBC`, puis l’appliance a refusé l’importation du certificat finalisé parce que la Root-CA émettrice lui était inconnue. Les deux obstacles et leurs solutions sont présentés plus bas.

## Ce que la SMA fait différemment de l’ESA

Sur l’ESA, tout le cycle de vie du certificat peut être géré via la GUI (`Network > Certificates`). La SMA ne le permet pas : le certificat serveur est installé exclusivement via la CLI, avec la commande `certconfig` dans une session SSH. La GUI de la SMA affiche uniquement les certificats ; seules les listes d’autorités de certification de confiance peuvent y être gérées, comme nous le verrons plus loin.

Deux autres particularités s’ajoutent à cela :

- La boîte de dialogue de collage n’accepte que le format PEM. Un fichier PFX (PKCS#12) doit être converti avant l’installation ; les versions actuelles d’AsyncOS proposent également un import PKCS#12 direct, mais le fichier doit d’abord être transféré sur l’appliance.
- Les anciennes versions d’AsyncOS (celles visées par la note technique Cisco) ne génèrent elles-mêmes ni clés ni CSR ; la paire de clés doit être créée en dehors. Les trois méthodes possibles sont décrites plus bas. Les versions actuelles peuvent générer directement sur l’appliance un certificat auto-signé avec CSR via `certconfig > CERTIFICATE > NEW`. Cela ne convient toutefois pas à un certificat partagé entre plusieurs appliances, car la clé privée ne quitte alors jamais l’appliance.

Un seul certificat peut au choix couvrir tous les services (TLS entrant et sortant, accès d’administration HTTPS, LDAPS) ou être enregistré séparément pour chaque service. Cela se configure dans la boîte de dialogue `certconfig` ; l’en-tête de la commande affiche à tout moment l’affectation active (`Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.`). Il n’existe pas de masque d’affectation séparé comme sur l’ESA, et rien ne peut être modifié à ce sujet dans la GUI. Dans la plupart des environnements, un certificat pour tous les usages constitue le choix pragmatique : la liste des noms couvre de toute façon les FQDN des appliances, et des paires de clés distinctes multiplient la charge de travail à chaque renouvellement.

Le fait que la boîte de dialogue demande le TLS entrant et sortant sur une appliance de quarantaine paraît d’abord surprenant, car la SMA ne se trouve sur aucun chemin MX. Elle utilise pourtant SMTP dans les deux sens. L’entrée (Receiving) est le côté de réception : les ESA remettent les messages mis en quarantaine à la SMA par SMTP, dans la quarantaine centrale des spams sur le port 6025 et dans les quarantaines centrales de stratégie, de virus et d’Outbreak sur le port 7025 ; ces dernières connexions sont chiffrées par TLS par défaut, et la SMA présente alors précisément ce certificat. La sortie (Delivery) est le côté d’envoi : lorsqu’un utilisateur libère un message de la quarantaine, la SMA le remet elle-même dans le flux de messagerie via ses routes SMTP, et l’appliance envoie également les notifications de quarantaine, les rapports planifiés et les alertes en tant que ses propres e-mails. Pour le renouvellement, HTTPS est donc essentiel en pratique ; les deux services SMTP sont simplement inclus avec le certificat destiné à tous les services.

## Définir les noms : CN et SAN

Quelle que soit la méthode choisie pour la paire de clés, il faut d’abord définir la liste des noms. Le Common Name doit correspondre au nom d’hôte sous lequel les utilisateurs accèdent à la page de quarantaine. La liste SAN doit en outre inclure les FQDN des appliances, afin que l’accès direct à la GUI d’administration fonctionne également sans avertissement. Pour un environnement avec deux appliances, la liste des noms se présente ainsi :

| Champ | Valeur |
|---|---|
| CN | `spam-quarantine.example.ch` |
| SAN | `spam-quarantine.example.ch` |
| SAN | `sma01.example.ch` |
| SAN | `sma02.example.ch` |

Deux remarques à ce sujet : les navigateurs n’évaluent depuis longtemps plus que les entrées SAN ; le CN seul ne suffit pas. Le nom d’hôte de la quarantaine doit donc également figurer comme SAN. Et les noms d’hôte courts sans partie de domaine (par exemple `SMA01`) ne sont émis que par une CA interne ; les CA publiques ne signent pas les noms internes.

## Trois méthodes pour obtenir une nouvelle paire de clés

Pour un certificat couvrant plusieurs appliances et le nom d’hôte de la quarantaine, la paire de clés doit être créée en dehors de l’appliance. Trois méthodes se sont imposées :

1. Générer la clé et le CSR avec OpenSSL au sein de votre propre environnement. La clé privée est créée là où elle est nécessaire et ne quitte jamais l’environnement. C’est la méthode recommandée ; les détails figurent dans la section suivante.
2. La CA génère la paire de clés et fournit un fichier PFX. Cela fonctionne, mais comporte deux inconvénients : la clé passe entre des mains externes (le mot de passe doit donc être transmis par un canal distinct, et non dans le même e-mail que le fichier), et selon l’outil de CA, un PFX chiffré avec RC2 peut être fourni, qu’OpenSSL 3 n’ouvre qu’avec des efforts supplémentaires ; voir plus bas.
3. Le détour par une ESA, documenté dans la note technique Cisco : créer sous `Network > Certificates` un certificat avec le CN de la SMA, télécharger le CSR et le faire signer par la CA, téléverser à nouveau le certificat signé sur l’ESA et exporter le tout en PFX. Ici aussi, une conversion en PEM est nécessaire à la fin.

## Les principales options d’openssl

Pour s’orienter, voici les sous-commandes et options de `openssl` utilisées dans cet article, traduites librement de la documentation OpenSSL :

<details class="options-details">
<summary>Aperçu des options</summary>

| Option | Signification |
|---|---|
| `req` | Sous-commande pour les demandes de certificat (CSR) : créer, afficher, vérifier |
| `-new` | Crée une nouvelle demande |
| `-newkey rsa:2048` | Crée également une nouvelle paire de clés RSA de 2048 bits |
| `-noenc` | Écrit la clé privée sans chiffrement (jusqu’à OpenSSL 3.0 : `-nodes`) |
| `-keyout datei` | Fichier cible de la clé privée |
| `-out datei` | Fichier cible de la sortie, ici CSR ou PEM |
| `-subj text` | Subject de la demande au format `/C=…/O=…/CN=…` |
| `-addext text` | Ajoute une extension à la demande, ici la liste SAN |
| `pkcs12` | Sous-commande pour les conteneurs PKCS#12 (PFX) : créer et extraire |
| `-in datei` | Fichier d’entrée |
| `-legacy` | Charge également le fournisseur Legacy, pour les anciens algorithmes tels que RC2 |
| `list` | Sous-commande pour afficher les capacités de l’installation |
| `-providers` | Liste les fournisseurs chargés |
| `-provider name` | Charge en plus le fournisseur indiqué pour cet appel |
| `s_client` | Sous-commande : client de test TLS pour les connexions à un serveur |
| `-connect host:port` | Hôte cible et port de la connexion TLS |
| `-servername name` | Définit la Server Name Indication (SNI) dans le handshake TLS |
| `x509` | Sous-commande pour afficher et traiter les certificats |
| `-noout` | Supprime l’affichage du certificat encodé |
| `-subject` | Affiche le Subject du certificat |
| `-enddate` | Affiche la date d’expiration (notAfter) |

</details>

La documentation OpenSSL présente les références complètes sous forme de page de manuel propre à chaque sous-commande : `openssl-req(1)`, `openssl-pkcs12(1)`, `openssl-s_client(1)` et `openssl-x509(1)`.

## Démarrer OpenSSL sous Windows

Toutes les étapes suivantes s’exécutent avec OpenSSL, sur un système situé au sein de l’environnement, par exemple un serveur d’administration. L’édition Light des builds Windows de Shining Light Productions suffit ; l’installeur pèse environ 6 Mo et peut être vérifié par rapport à la liste de sommes de contrôle publiée par slproweb.

L’installeur place tout sous `C:\Program Files\OpenSSL-Win64`, et l’exécutable se trouve dans `bin\openssl.exe`. Il ne s’ajoute pas au chemin de recherche : si l’on saisit `openssl` dans une invite de commandes fraîchement ouverte, on obtient un message d’erreur. Il existe trois possibilités :

- Ouvrir l’entrée `Win64 OpenSSL Command Prompt` dans le menu Démarrer. Elle lance `start.bat` depuis le répertoire d’installation, configure l’environnement et affiche le résultat de `openssl version -a`. Dans cette fenêtre, `openssl` fonctionne directement.
- Indiquer le chemin complet : `"C:\Program Files\OpenSSL-Win64\bin\openssl.exe" version`.
- Ajouter durablement `C:\Program Files\OpenSSL-Win64\bin` à la variable d’environnement `Path` ; `openssl` sera alors disponible dans chaque shell.

Aucune installation supplémentaire n’est nécessaire pour ceux qui utilisent déjà Git pour Windows : il inclut son propre OpenSSL (`C:\Program Files\Git\mingw64\bin\openssl.exe`), disponible immédiatement dans le chemin de recherche de Git Bash. Les versions récentes de Git fournissent OpenSSL 3.5 avec le fournisseur Legacy actif ; `-legacy` de la section sur la conversion PFX y fonctionne donc aussi. Vous pouvez le vérifier ainsi :

```bash
openssl list -providers -provider legacy
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `list` | Affiche les capacités de l’installation OpenSSL |
| `-providers` | Liste les fournisseurs chargés avec leur nom, leur version et leur statut |
| `-provider legacy` | Charge en plus le fournisseur `legacy` pour cet appel ; s’il apparaît dans la liste, il est disponible |

</details>

Git Bash a toutefois une particularité : elle considère les arguments commençant par `/` comme des chemins et les réécrit. `-subj "/C=CH/O=Example AG/CN=..."` devient `C:/Program Files/Git/C=CH/O=Example AG/CN=...`, et OpenSSL s’arrête :

```text
req: subject name is expected to be in the format /type0=value0/type1=value1/type2=...
where characters may be escaped by \. This name is not in that format:
'C:/Program Files/Git/C=CH/O=Example AG/CN=spam-quarantine.example.ch'
```

Un `MSYS_NO_PATHCONV=1` placé devant désactive cette réécriture pour un seul appel. Le problème ne se produit pas dans l’invite de commandes, PowerShell ni dans l’OpenSSL Command Prompt.

## Générer la clé et le CSR avec OpenSSL

Un seul appel génère la clé et le CSR avec la liste SAN complète :

```bash
openssl req -new -newkey rsa:2048 -noenc \
  -keyout spam-quarantine.example.ch.key \
  -out spam-quarantine.example.ch.csr \
  -subj "/C=CH/O=Example AG/CN=spam-quarantine.example.ch" \
  -addext "subjectAltName=DNS:spam-quarantine.example.ch,DNS:sma01.example.ch,DNS:sma02.example.ch"
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-new` | Crée une nouvelle demande de certificat (CSR) |
| `-newkey rsa:2048` | Crée également une nouvelle paire de clés RSA de 2048 bits |
| `-noenc` | Écrit la clé privée sans chiffrement dans le fichier |
| `-keyout …` | Fichier cible de la clé privée |
| `-out …` | Fichier cible du CSR |
| `-subj …` | Subject avec pays, organisation et Common Name |
| `-addext …` | Ajoute à la demande l’extension SAN avec tous les noms DNS |

</details>

Le fichier CSR est envoyé à la CA, tandis que la clé reste sur le serveur. La CA retourne le certificat signé avec l’intermédiaire, généralement directement au format PEM. Tout est alors prêt pour l’installation, et la conversion PFX est totalement inutile avec cette méthode.

Le fichier de clé n’est pas chiffré (`-noenc`), car `certconfig` l’attend précisément sous cette forme. Jusqu’à l’installation, il reste sur le serveur avec des autorisations restrictives, puis il est supprimé ou déplacé dans le gestionnaire de mots de passe.

## Convertir un PFX en PEM

Cette section et la suivante concernent les méthodes 2 et 3, qui produisent à la fin un fichier PFX. `certconfig` attend le certificat et la clé privée au format PEM, avec une clé non chiffrée. Un seul appel OpenSSL effectue ces deux opérations :

```bash
openssl pkcs12 -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `pkcs12` | Sous-commande permettant de créer et d’extraire des conteneurs PKCS#12 |
| `-in …` | Le fichier PFX d’entrée |
| `-out …` | Le fichier PEM de sortie avec certificat, clé et certificats de chaîne |
| `-noenc` | Écrit la clé privée sans passphrase (jusqu’à OpenSSL 3.0, l’option s’appelait `-nodes`) |

</details>

La demande du mot de passe d’importation s’effectue sans écho ; aucun astérisque ne s’affiche. Le fichier PEM créé contient le certificat, la clé et les certificats de chaîne fournis dans un seul fichier et doit donc être protégé en conséquence : supprimez-le après l’installation ou déplacez-le dans le gestionnaire de mots de passe.

## Lorsque OpenSSL 3 refuse le fichier PFX

Avec des fichiers PFX plus anciens, la conversion échoue sous OpenSSL 3.x avec le message suivant :

```text
Error outputting keys and certificates
error:0308010C:digital envelope routines:inner_evp_generic_fetch:unsupported:
crypto/evp/evp_fetch.c: Global default library context, Algorithm (RC2-40-CBC : 0)
```

La cause n’est pas un fichier défectueux, mais une décision de conception : OpenSSL 3 a déplacé les anciens algorithmes comme RC2, RC4 et DES dans un fournisseur Legacy séparé, qui n’est pas chargé par défaut. Or, de nombreuses exportations PFX d’anciens systèmes Windows et outils de CA chiffrent justement la partie certificat du conteneur avec RC2-40-CBC. OpenSSL 1.1 ouvrait ces fichiers sans difficulté, tandis qu’OpenSSL 3 les refuse.

La solution consiste en une seule option supplémentaire :

```bash
openssl pkcs12 -legacy -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-legacy` | Charge le fournisseur Legacy pour cet appel ; les anciens algorithmes tels que RC2-40-CBC sont alors à nouveau disponibles et la conversion réussit |

</details>

Cela nécessite une installation OpenSSL qui inclut le fournisseur Legacy ; c’est le cas des builds Windows courants.

Pour éliminer l’erreur durablement, vous pouvez agir à la source et exporter le fichier PFX avec un chiffrement moderne : les boîtes de dialogue d’exportation et outils de CA actuels proposent AES-256, ce qui supprime entièrement le détour par Legacy.

Comme alternative graphique, XCA (X Certificate and Key Management) fonctionne également : importer le fichier PFX via `Importieren > PKCS#12`, puis exporter le certificat au format PEM dans l’onglet `Zertifikate` et la clé séparément au format PEM non chiffré dans l’onglet `Private Schlüssel`. Les deux exportations sont nécessaires ; `certconfig` demande séparément le certificat et la clé. XCA inclut sa propre bibliothèque cryptographique et ouvre également les conteneurs utilisant des algorithmes Legacy.

Un mot encore sur la source de téléchargement : le projet OpenSSL ne publie pas lui-même de binaires Windows, mais renvoie vers des builds tiers comme Win64 OpenSSL de Shining Light Productions. Les portails de téléchargement avec leurs propres installateurs ne sont pas une adresse appropriée pour un outil cryptographique.

## Importer d’abord la Root-CA interne dans le magasin de confiance de l’appliance

Les versions actuelles d’AsyncOS valident la chaîne complète lors de la création d’un profil de certificat. Si le certificat provient d’une CA interne dont l’appliance ne connaît pas la racine, l’importation échoue avec le message suivant :

```text
Cannot import certificate: The certificate authority validation could not be
trusted for certificate because unable to get local issuer certificate
```

L’appliance maintient deux listes d’autorités de certification de confiance : la liste système fournie et une liste Custom destinée aux CA internes. La Root-CA interne doit être ajoutée à la liste Custom avant l’installation du certificat serveur. Seul le certificat public de la CA au format PEM est nécessaire (`-----BEGIN CERTIFICATE-----` à `-----END CERTIFICATE-----`), sans clé privée.

Voici comment importer la Root-CA sur l’appliance via l’interface web :

1. Ouvrez `Network > Certificates`.
2. Dans la section `Certificate Authorities`, cliquez sur `Edit Settings`.
3. Pour `Custom List`, sélectionnez l’option `Enable`.
4. Téléversez le fichier PEM avec `Choose File`.
5. Exécutez `Submit`, puis `Commit Changes`.
6. Sous `Network > Certificates > Manage Trusted Root Certificates`, vérifiez que la CA apparaît dans la liste des certificats personnalisés.

Si une liste Custom existe déjà, exportez-la au préalable et ajoutez la nouvelle CA au bundle PEM existant : l’importation remplace la liste, faute de quoi les CA précédemment enregistrées disparaissent. Pour une chaîne comportant un intermédiaire, importez d’abord la Root-CA, puis la CA intermédiaire. Lors de l’importation, AsyncOS vérifie notamment la date d’expiration, les doublons et le flag `CA:TRUE` défini, et refuse une intermédiaire tant que la racine correspondante est absente. Cette même importation peut aussi être effectuée via la CLI : `certconfig > CERTAUTHORITY > CUSTOM > IMPORT`, puis `commit`.

Deux précisions : pour les mises à jour via un proxy avec inspection TLS, la SMA utilise un magasin de confiance distinct (`updateconfig > TRUSTED_CERTIFICATES > ADD`) ; la liste Custom de CA ne s’y applique pas. Et la Root-CA sur la SMA n’élimine pas les avertissements du navigateur : les clients ont toujours besoin de cette racine via leur propre distribution de certificats, généralement par GPO, et l’appliance doit fournir le certificat serveur avec l’intermédiaire.

## Installation avec certconfig

Connectez-vous à la SMA par SSH et lancez `certconfig`. Dans les versions actuelles d’AsyncOS, la boîte de dialogue fonctionne avec des profils de certificats :

```text
sma01.example.ch> certconfig

Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.

Choose the operation you want to perform:
- CERTIFICATE - Import, Create a request, Edit or Remove Certificate Profiles
- CERTAUTHORITY - Manage System and Customized Authorities
- CRL - Manage Certificate Revocation Lists
[]> certificate
```

Derrière `CERTIFICATE` se trouvent les opérations `IMPORT` (fichier PKCS#12 préalablement transféré sur l’appliance), `PASTE` (coller un certificat dans la CLI), `NEW` (générer un certificat auto-signé avec CSR), `EDIT`, `EXPORT`, `DELETE` et `PRINT` (affiche l’affectation aux services). La méthode habituelle via SSH est `PASTE` : la boîte de dialogue demande un nom pour le profil, puis le certificat, la clé privée et, facultativement, le certificat intermédiaire de la CA, chacun sous forme de bloc PEM terminé par un `.` seul sur sa propre ligne. La question finale concernant la vérification FQDN du Common Name peut recevoir la réponse par défaut. L’intermédiaire doit être inclus dans le profil, faute de quoi la chaîne manque aux clients et l’avertissement peut persister selon le navigateur malgré un certificat valide.

Les anciennes versions d’AsyncOS (celles visées par la note technique Cisco) affichent à la place une boîte de dialogue `SETUP`. Elle commence par la question `Do you want to use one certificate/key for receiving, delivery, HTTPS management access, and LDAPS?` : un `y` attribue la même paire aux quatre services, un `n` parcourt les demandes de certificat, de clé et d’intermédiaire une fois par service. Le principe de collage est identique.

Deux points déterminent le succès ou l’échec : ne terminez pas la session avec Ctrl+C, car cela annule immédiatement toutes les modifications. Et exécutez `commit` à la fin ; ce n’est qu’alors que le certificat est actif. Avec deux appliances, répétez la procédure sur les deux : la configuration des certificats n’est pas synchronisée entre les SMA.

## Vérification

Le test le plus rapide s’effectue depuis l’extérieur contre la page de quarantaine. L’accès utilisateur final à la quarantaine de spams utilise HTTPS sur le port 83 par défaut, sauf configuration différente lors de l’activation :

```bash
openssl s_client -connect spam-quarantine.example.ch:83 \
  -servername spam-quarantine.example.ch </dev/null 2>/dev/null |
  openssl x509 -noout -subject -enddate
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `s_client` | Client de test TLS : établit la connexion et transmet le certificat présenté |
| `-connect …:83` | Hôte cible et port, ici le port HTTPS de la quarantaine de spams |
| `-servername …` | Définit la Server Name Indication (SNI), afin que le serveur fournisse le certificat approprié |
| `x509` | Traite le certificat transmis |
| `-noout` | Supprime l’affichage du certificat encodé |
| `-subject` | Affiche le Subject du certificat |
| `-enddate` | Affiche la date d’expiration (notAfter) |

</details>

La sortie doit afficher le nouveau Subject et la nouvelle date d’expiration. Sur l’appliance, `certconfig` avec l’opération `PRINT` liste les certificats actifs, et la vérification dans le navigateur de la GUI d’administration comme de la page de quarantaine confirme que la chaîne est correctement établie.

## Sources

1.  [How to Generate and Install a Certificate on an SMA](https://www.cisco.com/c/en/us/support/docs/security/content-security-management-appliance/118460-technote-sma-00.html): Note technique Cisco décrivant la procédure certconfig des anciennes versions d’AsyncOS, l’exigence PEM et les méthodes de génération de certificat via ESA ou OpenSSL.

2.  [Common Administrative Tasks, AsyncOS 16.0 pour Cisco Secure Email and Web Manager](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_01011.html): Chapitre du guide d’administration consacré à la gestion des listes d’autorités de certification (listes System et Custom), y compris les contrôles effectués lors de l’importation d’une CA.

3.  [Comprehensive Spam Quarantine Setup Guide on ESA and SMA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118692-configure-esa-00.html): Guide Cisco sur la quarantaine de spams, y compris l’accès utilisateur final par HTTPS sur le port 83.

4.  [openssl-req](https://docs.openssl.org/master/man1/openssl-req/): Référence pour la génération de clé et de CSR, y compris `-addext` pour la liste SAN.

5.  [openssl-pkcs12](https://docs.openssl.org/master/man1/openssl-pkcs12/): Référence des options de conversion, notamment `-noenc` (anciennement `-nodes`) et `-legacy`.

6.  [OpenSSL 3.0 Migration Guide](https://docs.openssl.org/3.0/man7/migration_guide/): Contexte sur le déplacement des anciens algorithmes vers le fournisseur Legacy.

7.  [XCA: X Certificate and Key Management](https://hohnstaedt.de/xca/): Outil open source pour l’importation et l’exportation de structures PKCS#12 et PEM.

8.  [Win32/Win64 OpenSSL](https://slproweb.com/products/Win32OpenSSL.html): Builds Windows de Shining Light Productions auxquels le projet OpenSSL renvoie, avec liste de sommes de contrôle publiée.

9.  [MSYS2: Filesystem Paths](https://www.msys2.org/docs/filesystem-paths/): Description de la réécriture automatique des chemins qui modifie l’argument `-subj` dans Git Bash, avec `MSYS_NO_PATHCONV`.

10.  [openssl-s_client](https://docs.openssl.org/master/man1/openssl-s_client/): Référence du client de test TLS, notamment `-connect` et `-servername`.

11.  [openssl-x509](https://docs.openssl.org/master/man1/openssl-x509/): Référence des options d’affichage, notamment `-noout`, `-subject` et `-enddate`.
