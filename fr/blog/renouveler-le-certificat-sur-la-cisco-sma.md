---
title: "Renouveler le certificat sur la Cisco SMA"
navTitle: "Certificat SMA"
description: "Les certificats ne peuvent être installés sur la Cisco SMA que via la CLI, et les versions récentes d’AsyncOS valident l’intégralité de la chaîne lors de l’importation : sans AC racine enregistrée, celle-ci échoue. Cet article présente les moyens d’obtenir une nouvelle paire de clés, la procédure OpenSSL en détail, la gestion de l’erreur RC2-40-CBC d’OpenSSL 3 et l’importation de l’AC racine interne dans le magasin de confiance de l’appliance."
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
url: https://rafaelpfister.ch/fr/blog/renouveler-le-certificat-sur-la-cisco-sma
translationSourceHash: 6dc8240e5839f04d73103bb79e45ad14bdc9a7a16e02e2c57f9a4f33be24b53c
translationModel: gpt-5.6-terra
translatedAt: 2026-08-05T06:05:56.491Z
translationReview: automatic
---

# Renouveler le certificat sur la Cisco SMA

La Cisco SMA (Security Management Appliance, désormais commercialisée sous le nom Cisco Secure Email and Web Manager) assure dans de nombreux environnements de messagerie la quarantaine centralisée des spams et le reporting pour les Secure Email Gateways. Son certificat HTTPS couvre l’interface d’administration et la page de quarantaine, où les utilisateurs finaux consultent et libèrent leurs e-mails retenus. Son expiration n’interrompt pas le flux de messagerie. Elle est néanmoins immédiatement visible : chaque ouverture de la page de quarantaine se termine par un avertissement de certificat dans le navigateur, et ce sont précisément les utilisateurs auxquels les formations de sensibilisation apprennent à ne pas poursuivre face à de tels avertissements qui sont alors censés les ignorer.

Lors d’un renouvellement dans le cadre d’un projet client, deux écueils se sont présentés : OpenSSL 3 a d’abord renvoyé une erreur cryptique concernant `RC2-40-CBC` pour le fichier PFX de l’AC interne, puis l’appliance a refusé l’importation du certificat final, car l’AC racine émettrice ne lui était pas connue. Les deux obstacles et leur solution sont détaillés plus bas.

## Ce que la SMA fait différemment de l’ESA

Sur l’ESA, l’ensemble du cycle de vie des certificats peut être géré via l’interface graphique (`Network > Certificates`). La SMA ne le permet pas : le certificat serveur est installé exclusivement via la CLI, avec la commande `certconfig` dans une session SSH. L’interface graphique de la SMA n’affiche que les certificats ; seules les listes d’autorités de certification approuvées peuvent y être gérées, comme expliqué plus loin.

Deux autres particularités s’ajoutent :

- La boîte de dialogue de collage n’accepte que le format PEM. Un fichier PFX (PKCS#12) doit être converti avant l’installation ; les versions récentes d’AsyncOS proposent également une importation PKCS#12 directe, mais le fichier doit d’abord être transféré sur l’appliance.
- Les anciennes versions d’AsyncOS (à l’époque de la Technote Cisco) ne génèrent elles-mêmes ni clés ni CSR ; la paire de clés doit être créée à l’extérieur. Les trois méthodes possibles sont présentées plus bas. Les versions actuelles peuvent générer directement sur l’appliance un certificat autosigné avec CSR à l’aide de `certconfig > CERTIFICATE > NEW`. Cela n’aide toutefois pas pour un certificat commun à plusieurs appliances, car la clé privée ne quitte alors jamais l’appliance.

Un seul certificat peut au choix servir tous les services (TLS entrant et sortant, accès d’administration HTTPS, LDAPS) ou être enregistré séparément pour chaque service. Cela se règle dans la boîte de dialogue `certconfig` ; l’en-tête de la commande affiche à tout moment l’affectation active (`Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.`). Il n’existe pas d’écran d’affectation distinct comme sur l’ESA, et rien ne peut être modifié à ce niveau via l’interface graphique. Dans la plupart des environnements, un certificat unique pour tout est le choix pragmatique : la liste des noms couvre de toute façon les FQDN des appliances, et des paires de clés distinctes multiplient les efforts à chaque renouvellement.

Le fait que la boîte de dialogue demande le TLS entrant et sortant sur une appliance de quarantaine peut sembler déroutant au premier abord, car la SMA ne se trouve sur aucun chemin MX. Elle communique néanmoins en SMTP dans les deux sens. Le trafic entrant (Receiving) est le côté réception : les ESA livrent les messages mis en quarantaine par SMTP à la SMA, vers la quarantaine centralisée des spams sur le port 6025 et vers les quarantaines centralisées de politiques, de virus et d’épidémies sur le port 7025 ; ces dernières connexions sont chiffrées par TLS par défaut, et la SMA y présente précisément ce certificat. Le trafic sortant (Delivery) est le côté envoi : lorsqu’un utilisateur libère un message de la quarantaine, la SMA le remet elle-même dans le flux de messagerie via ses routes SMTP ; l’appliance envoie aussi les notifications de quarantaine, les rapports planifiés et les alertes comme ses propres e-mails. Pour le renouvellement, HTTPS est donc critique en pratique ; les deux services SMTP sont simplement inclus dans le certificat destiné à tous les services.

## Définir les noms : CN et SAN

Quelle que soit la méthode retenue pour la paire de clés, il faut d’abord définir la liste des noms. Le Common Name doit être le nom d’hôte sous lequel les utilisateurs ouvrent la page de quarantaine. La liste SAN doit également contenir les FQDN des appliances afin que l’accès direct à l’interface d’administration fonctionne sans avertissement. Pour un environnement comportant deux appliances, la liste des noms se présente ainsi :

| Champ | Valeur |
|---|---|
| CN | `spam-quarantine.example.ch` |
| SAN | `spam-quarantine.example.ch` |
| SAN | `sma01.example.ch` |
| SAN | `sma02.example.ch` |

Deux remarques à ce sujet : les navigateurs ne tiennent depuis longtemps compte que des entrées SAN ; le CN seul ne suffit pas. Le nom d’hôte de la quarantaine doit donc également figurer comme SAN. Les noms d’hôte courts sans partie domaine (par exemple `SMA01`) ne sont délivrés que par une AC interne ; les AC publiques ne signent pas de noms internes.

## Trois méthodes pour obtenir une nouvelle paire de clés

Pour un certificat couvrant plusieurs appliances et le nom d’hôte de la quarantaine, la paire de clés doit être créée en dehors de l’appliance. Trois méthodes se sont établies :

1. Générer la clé et le CSR avec OpenSSL au sein de son propre environnement. La clé privée est créée là où elle est nécessaire et ne quitte jamais l’environnement. C’est la méthode recommandée, détaillée dans la section suivante.
2. L’AC génère la paire de clés et fournit un fichier PFX. Cela fonctionne, mais présente deux inconvénients : la clé passe entre des mains tierces (le mot de passe doit donc être transmis par un canal distinct et non dans le même e-mail que le fichier) et, selon l’outil de l’AC, un PFX chiffré en RC2 peut être fourni, qu’OpenSSL 3 n’ouvre qu’avec un effort supplémentaire ; voir plus bas.
3. Le détour via une ESA, documenté dans la Technote Cisco : créer un certificat avec le CN de la SMA sous `Network > Certificates`, télécharger le CSR et le faire signer par l’AC, téléverser le certificat signé sur l’ESA et exporter le tout sous forme de PFX. Ici aussi, une conversion en PEM est nécessaire à la fin.

## Démarrer OpenSSL sous Windows

Toutes les étapes suivantes s’effectuent avec OpenSSL, sur un système situé dans l’environnement, par exemple un serveur d’administration. L’édition Light des builds Windows de Shining Light Productions suffit ; l’installateur pèse environ 6 Mo et peut être vérifié avec la liste de sommes de contrôle publiée par slproweb.

L’installateur place tout sous `C:\Program Files\OpenSSL-Win64`, l’exécutable se trouve dans `bin\openssl.exe`. Il ne s’ajoute pas au chemin de recherche : si vous tapez `openssl` dans une invite de commandes fraîchement ouverte, vous obtenez un message d’erreur. Trois solutions sont possibles :

- Ouvrir l’entrée `Win64 OpenSSL Command Prompt` dans le menu Démarrer. Elle lance `start.bat` depuis le répertoire d’installation, configure l’environnement et affiche la sortie de `openssl version -a`. Dans cette fenêtre, `openssl` fonctionne directement.
- Indiquer le chemin complet : `"C:\Program Files\OpenSSL-Win64\bin\openssl.exe" version`.
- Ajouter durablement `C:\Program Files\OpenSSL-Win64\bin` à la variable d’environnement `Path` ; `openssl` sera alors disponible dans chaque shell.

Il est possible de se passer d’installation supplémentaire si Git pour Windows est déjà utilisé : il fournit son propre OpenSSL (`C:\Program Files\Git\mingw64\bin\openssl.exe`), immédiatement présent dans le chemin de recherche dans Git Bash. Les versions actuelles de Git incluent OpenSSL 3.5 avec le fournisseur Legacy actif ; `-legacy` de la section consacrée à la conversion PFX y fonctionne donc également. Vous pouvez le vérifier ainsi :

```bash
openssl list -providers -provider legacy
```

Git Bash présente toutefois un piège : il considère les arguments commençant par `/` comme des chemins et les réécrit. `-subj "/C=CH/O=Example AG/CN=..."` devient `C:/Program Files/Git/C=CH/O=Example AG/CN=...`, et OpenSSL s’interrompt :

```text
req: subject name is expected to be in the format /type0=value0/type1=value1/type2=...
where characters may be escaped by \. This name is not in that format:
'C:/Program Files/Git/C=CH/O=Example AG/CN=spam-quarantine.example.ch'
```

Un `MSYS_NO_PATHCONV=1` placé devant désactive la réécriture pour cet appel individuel. Le problème ne se produit pas dans l’invite de commandes, PowerShell ni dans OpenSSL Command Prompt.

## Générer la clé et le CSR avec OpenSSL

Une seule commande génère la clé et le CSR avec la liste SAN complète :

```bash
openssl req -new -newkey rsa:2048 -noenc -keyout spam-quarantine.example.ch.key -out spam-quarantine.example.ch.csr -subj "/C=CH/O=Example AG/CN=spam-quarantine.example.ch" -addext "subjectAltName=DNS:spam-quarantine.example.ch,DNS:sma01.example.ch,DNS:sma02.example.ch"
```

Le fichier CSR est envoyé à l’AC, la clé reste sur le serveur. En retour, vous recevez le certificat signé et l’intermédiaire, généralement directement au format PEM. Tout est alors prêt pour l’installation ; la conversion PFX est entièrement évitée avec cette méthode.

Le fichier de clé n’est pas chiffré (`-noenc`), car `certconfig` l’attend précisément ainsi. Jusqu’à l’installation, il reste sur le serveur avec des autorisations restrictives, puis il est supprimé ou déplacé vers le gestionnaire de mots de passe.

## Convertir un PFX en PEM

Cette section et la suivante concernent les méthodes 2 et 3, qui aboutissent à un fichier PFX. `certconfig` attend le certificat et la clé privée au format PEM, avec une clé non chiffrée. Une seule commande OpenSSL réalise les deux opérations :

```bash
openssl pkcs12 -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

`-noenc` (jusqu’à OpenSSL 3.0, l’option s’appelait `-nodes`) écrit la clé privée sans phrase secrète dans le fichier de sortie. La demande du mot de passe d’importation s’effectue sans écho ; aucun astérisque n’apparaît non plus. Le fichier PEM créé contient le certificat, la clé et les certificats de chaîne fournis dans un seul fichier ; il doit donc être protégé en conséquence : supprimez-le après l’installation ou placez-le dans le gestionnaire de mots de passe.

## Lorsque OpenSSL 3 refuse le fichier PFX

Avec des fichiers PFX plus anciens, la conversion sous OpenSSL 3.x s’interrompt avec ce message :

```text
Error outputting keys and certificates
error:0308010C:digital envelope routines:inner_evp_generic_fetch:unsupported:
crypto/evp/evp_fetch.c: Global default library context, Algorithm (RC2-40-CBC : 0)
```

La cause n’est pas un fichier défectueux, mais une décision de conception : OpenSSL 3 a déplacé les anciens algorithmes tels que RC2, RC4 et DES dans un fournisseur Legacy distinct, qui n’est pas chargé par défaut. Or, de nombreuses exportations PFX d’anciens systèmes Windows et outils d’AC chiffrent précisément la partie certificat du conteneur avec RC2-40-CBC. OpenSSL 1.1 ouvrait ces fichiers sans problème ; OpenSSL 3 les refuse.

La solution consiste en une seule option supplémentaire :

```bash
openssl pkcs12 -legacy -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

`-legacy` charge le fournisseur Legacy pour cet appel ; la conversion s’exécute alors correctement. Une installation OpenSSL incluant le fournisseur Legacy est nécessaire ; c’est le cas des builds Windows courants.

Pour éliminer durablement l’erreur, il faut agir à la source et exporter le fichier PFX avec un chiffrement moderne : les boîtes de dialogue d’exportation et outils d’AC actuels proposent AES-256, ce qui supprime entièrement le détour par Legacy.

Comme alternative graphique, XCA (X Certificate and Key Management) fonctionne : importez le fichier PFX via `Importieren > PKCS#12`, puis exportez le certificat au format PEM dans l’onglet `Zertifikate` et la clé séparément comme PEM non chiffré dans l’onglet `Private Schlüssel`. Les deux exportations sont nécessaires ; `certconfig` demande séparément le certificat et la clé. XCA intègre sa propre bibliothèque cryptographique et ouvre également les conteneurs utilisant des algorithmes Legacy.

Un mot encore sur la source de téléchargement : le projet OpenSSL ne publie pas lui-même de binaires Windows, mais renvoie vers des builds tiers tels que Win64 OpenSSL de Shining Light Productions. Les portails de téléchargement avec leurs propres installateurs ne sont pas une adresse appropriée pour un outil cryptographique.

## Importer d’abord l’AC racine interne dans le magasin de confiance de l’appliance

Les versions récentes d’AsyncOS valident l’intégralité de la chaîne lors de la création d’un profil de certificat. Si le certificat provient d’une AC interne dont l’appliance ne connaît pas la racine, l’importation s’interrompt avec ce message :

```text
Cannot import certificate: The certificate authority validation could not be
trusted for certificate because unable to get local issuer certificate
```

L’appliance gère deux listes d’autorités de certification approuvées : la liste système fournie et une liste Custom pour les AC propres. L’AC racine interne doit être ajoutée à la liste Custom avant l’installation du certificat serveur. Seul le certificat public de l’AC au format PEM est nécessaire (`-----BEGIN CERTIFICATE-----` à `-----END CERTIFICATE-----`), sans clé privée.

Voici comment transférer l’AC racine sur l’appliance via l’interface web :

1. Ouvrir `Network > Certificates`.
2. Dans la section `Certificate Authorities`, cliquer sur `Edit Settings`.
3. Pour `Custom List`, sélectionner l’option `Enable`.
4. Téléverser le fichier PEM via `Choose File`.
5. Exécuter `Submit`, puis `Commit Changes`.
6. Sous `Network > Certificates > Manage Trusted Root Certificates`, vérifier que l’AC apparaît dans la liste des certificats personnalisés.

Si une liste Custom existe déjà, exportez-la au préalable et ajoutez la nouvelle AC au bundle PEM existant : l’importation remplace la liste, sinon les AC précédemment enregistrées disparaissent. Dans le cas d’une chaîne comportant un intermédiaire, importez d’abord l’AC racine, puis l’AC intermédiaire. AsyncOS vérifie notamment la date d’expiration, les doublons et l’indicateur `CA:TRUE`, et refuse un intermédiaire tant que la racine correspondante est absente. La même importation est également possible via la CLI : `certconfig > CERTAUTHORITY > CUSTOM > IMPORT`, puis `commit`.

Deux distinctions importantes : pour les mises à jour via un proxy pratiquant l’inspection TLS, la SMA utilise un magasin de confiance distinct (`updateconfig > TRUSTED_CERTIFICATES > ADD`) ; la liste Custom d’AC ne s’y applique pas. Et l’AC racine sur la SMA ne supprime pas les avertissements du navigateur : les clients doivent toujours recevoir la racine par leur propre distribution de certificats, généralement par GPO, et l’appliance doit fournir le certificat serveur avec l’intermédiaire.

## Installation avec certconfig

Connectez-vous à la SMA par SSH et lancez `certconfig`. Sur les versions actuelles d’AsyncOS, la boîte de dialogue utilise des profils de certificats :

```text
sma01.example.ch> certconfig

Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.

Choose the operation you want to perform:
- CERTIFICATE - Import, Create a request, Edit or Remove Certificate Profiles
- CERTAUTHORITY - Manage System and Customized Authorities
- CRL - Manage Certificate Revocation Lists
[]> certificate
```

Derrière `CERTIFICATE` se trouvent les opérations `IMPORT` (fichier PKCS#12 préalablement transféré sur l’appliance), `PASTE` (coller le certificat dans la CLI), `NEW` (générer un certificat autosigné avec CSR), `EDIT`, `EXPORT`, `DELETE` et `PRINT` (affiche l’affectation aux services). La procédure SSH habituelle utilise `PASTE` : la boîte de dialogue demande un nom pour le profil, puis le certificat, la clé privée et, en option, le certificat intermédiaire de l’AC, chacun sous forme de bloc PEM terminé par un unique `.` sur sa propre ligne. La question finale sur la vérification du FQDN du Common Name peut recevoir la réponse par défaut. L’intermédiaire doit être inclus dans le profil ; sinon la chaîne manque aux clients et, selon le navigateur, l’avertissement peut persister malgré un certificat valide.

Les anciennes versions d’AsyncOS (à l’époque de la Technote Cisco) affichent à la place une boîte de dialogue `SETUP`. Elle commence par la question `Do you want to use one certificate/key for receiving, delivery, HTTPS management access, and LDAPS?` : un `y` affecte la même paire aux quatre services, un `n` parcourt la saisie du certificat, de la clé et de l’intermédiaire une fois par service. Le principe de collage est identique.

Deux points déterminent la réussite ou l’échec : ne terminez pas la session avec Ctrl+C, car cela annule immédiatement toutes les modifications. Et exécutez `commit` à la fin ; le certificat ne devient actif qu’à ce moment-là. Avec deux appliances, la procédure doit être répétée sur les deux ; la configuration des certificats n’est pas synchronisée entre les SMA.

## Vérification

Le test le plus rapide se fait de l’extérieur contre la page de quarantaine. L’accès utilisateur final à la quarantaine de spams utilise par défaut le port HTTPS 83, sauf si une autre configuration a été définie lors de l’activation :

```bash
openssl s_client -connect spam-quarantine.example.ch:83 -servername spam-quarantine.example.ch </dev/null 2>/dev/null | openssl x509 -noout -subject -enddate
```

La sortie doit afficher le nouveau sujet et la nouvelle date d’expiration. Sur l’appliance, `certconfig` avec l’opération `PRINT` liste les certificats actifs, et la vérification dans le navigateur contre l’interface d’administration et la page de quarantaine confirme que la chaîne est correctement établie.

## Sources

1.  [How to Generate and Install a Certificate on an SMA](https://www.cisco.com/c/en/us/support/docs/security/content-security-management-appliance/118460-technote-sma-00.html): Technote Cisco présentant la procédure certconfig des anciennes versions d’AsyncOS, l’exigence PEM et les méthodes de génération du certificat via ESA ou OpenSSL.

2.  [Common Administrative Tasks, AsyncOS 16.0 pour Cisco Secure Email and Web Manager](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_01011.html): Chapitre du guide d’administration consacré à la gestion des listes d’autorités de certification (listes système et Custom), y compris les vérifications lors de l’importation d’une AC.

3.  [Comprehensive Spam Quarantine Setup Guide on ESA and SMA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118692-configure-esa-00.html): Guide Cisco relatif à la quarantaine de spams, y compris l’accès utilisateur final par HTTPS sur le port 83.

4.  [openssl-req](https://docs.openssl.org/master/man1/openssl-req/): Référence pour la génération de clé et de CSR, y compris `-addext` pour la liste SAN.

5.  [openssl-pkcs12](https://docs.openssl.org/master/man1/openssl-pkcs12/): Référence des options de conversion, notamment `-noenc` (anciennement `-nodes`) et `-legacy`.

6.  [OpenSSL 3.0 Migration Guide](https://docs.openssl.org/3.0/man7/migration_guide/): Informations sur le déplacement des anciens algorithmes vers le fournisseur Legacy.

7.  [XCA: X Certificate and Key Management](https://hohnstaedt.de/xca/): Outil open source pour l’importation et l’exportation de structures PKCS#12 et PEM.

8.  [Win32/Win64 OpenSSL](https://slproweb.com/products/Win32OpenSSL.html): Builds Windows de Shining Light Productions, vers lesquels renvoie le projet OpenSSL, avec la liste de sommes de contrôle publiée.

9.  [MSYS2: Filesystem Paths](https://www.msys2.org/docs/filesystem-paths/): Description de la réécriture automatique des chemins qui modifie l’argument `-subj` dans Git Bash, ainsi que `MSYS_NO_PATHCONV`.
