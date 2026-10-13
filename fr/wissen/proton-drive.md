---
title: "Proton Drive : modèle de données cryptographique et fonctionnement de la synchronisation"
blatt: "proton-drive"
description: "Proton Drive pour les administrateurs du stockage, de la sécurité et des plateformes : stockage cloud centralisé avec chiffrement de bout en bout côté client, volumes, partages, nœuds, révisions et blocs, hiérarchie de clés et signatures OpenPGP, Drive SDK, synchronisation de bureau et fichiers à la demande, automatisation CLI, partage, conflits, versions, journalisation, récupération et historique technique."
fakten:
  - label: Rôle système
    wert: stockage de fichiers cloud centralisé avec chiffrement de bout en bout côté client, synchronisation, partage et versionnage
    href: https://proton.me/support/drive
  - label: Modèle de confiance
    wert: le serveur stocke les contenus et noms chiffrés ; le texte en clair et le matériel de clés déverrouillé sont générés dans le client autorisé
    href: https://proton.me/blog/protondrive-security
  - label: Modèle de données
    wert: un volume contient des partages ; un partage pointe vers une racine d’arborescence ; les fichiers et dossiers sont des nœuds avec liens, clés et révisions
    href: https://proton.me/blog/protondrive-security
  - label: Transfert de fichiers
    wert: les révisions se composent de blocs chiffrés ; le client chiffre, transfère, vérifie et recompose l’objet
    href: https://proton.me/blog/drive-cryptography-update
  - label: Cryptographie
    wert: hiérarchie de clés basée sur OpenPGP ; les nouveaux fichiers utilisent v6 PKESK, v2 SEIPD et AES-GCM, les anciennes révisions restent liées à leur schéma
    href: https://proton.me/blog/drive-cryptography-update
  - label: Métadonnées
    wert: le contenu ainsi que les noms de fichiers et de dossiers sont E2EE ; selon Proton, certains champs tels que la taille des fichiers restent lisibles par le serveur
    href: https://proton.me/support/drive
  - label: Intégrité
    wert: les informations signées sur les clés, l’arborescence et les blocs, ainsi que les signatures OpenPGP, détectent les manipulations
    href: https://proton.me/support/drive-signature-management
  - label: Noyau client
    wert: Drive SDK unifie les opérations sur les fichiers et la cryptographie pour les clients web, de bureau, mobiles et CLI
    href: https://github.com/ProtonDriveApps/sdk
  - label: Synchronisation de bureau
    wert: Windows et macOS intègrent des espaces réservés à la demande, des états locaux hors ligne et la synchronisation en arrière-plan dans Explorer ou Finder
    href: https://proton.me/support/proton-drive-windows-on-demand-sync
  - label: CLI
    wert: programme autonome officiel pour Windows, macOS et Linux ; opérations sur les fichiers et le partage, mais pas de moteur de synchronisation de bureau permanent
    href: https://proton.me/support/drive-cli
  - label: Noyau de récupération
    wert: l’accès au compte ne suffit pas après une réinitialisation de mot de passe ; l’ancienne clé est déverrouillée à l’aide du mot de passe précédent, de la Recovery Phrase ou du Recovery File
    href: https://proton.me/support/drive-data-recovery
  - label: Règle de sauvegarde
    wert: la synchronisation et l’historique des versions ne constituent pas une sauvegarde indépendante ; les données critiques nécessitent une exportation séparée, des sommes de contrôle, une conservation et un test de restauration
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/final
werbung:
  - newsletter
ctaThemen:
  - proton-drive
translationSourceHash: 6063c7d85190de63a8973bc7cdb304fff10f81fd617a2044cf40f8371849717c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-11T10:44:14.447Z
translationReview: automatic
---

# Proton Drive : modèle de données cryptographique et fonctionnement de la synchronisation

Proton Drive est un stockage de fichiers cloud exploité de manière centralisée, dont les clients chiffrent les contenus et les métadonnées essentielles avant le téléversement. Le service associe accès web, synchronisation de bureau, applications mobiles, partages, versions et ligne de commande à un modèle de données chiffré de bout en bout. La [vue d’ensemble de l’assistance](https://proton.me/support/drive) indique des accès web, Windows, macOS, Android et iOS/iPadOS ; la [Drive CLI](https://proton.me/support/drive-cli) officielle ajoute des opérations scriptables sous Windows, macOS et Linux.

Pour les administrateurs, Proton Drive n’est ni un partage de fichiers classique ni un simple portail de téléversement. Un client autorisé gère la session du compte et les clés, résout une arborescence chiffrée, surveille les modifications locales, crée des révisions, chiffre des blocs et synchronise les événements côté serveur vers le système de fichiers. Les serveurs restent responsables de la disponibilité, de l’autorisation, du quota de stockage, des versions et des partages, sans pouvoir traiter en clair le contenu des fichiers ou les noms chiffrés.

L’explication suit un fichier depuis le client, en passant par les clés, les blocs chiffrés et l’API, jusqu’au stockage côté serveur. La synchronisation, les partages, le versionnage, le diagnostic et la sauvegarde indépendante reposent sur ce chemin de données.

Proton Drive stocke les données de manière centralisée, mais délègue le chiffrement et la vérification des signatures à des clients de confiance. Cet article suit un fichier depuis la modification locale, à travers la hiérarchie de clés, les blocs et les révisions, jusqu’à la synchronisation, au partage et à la restauration.

## Architecture : service centralisé, client de confiance

Proton a choisi un stockage cloud centralisé plutôt qu’un réseau pair à pair. Les serveurs fournissent l’API, le stockage d’objets, les événements, les appartenances, les quotas, les versions et la disponibilité ; les clients prennent en charge le traitement en clair et la cryptographie. Proton explique les raisons de ce modèle dans sa [comparaison entre stockage centralisé et décentralisé](https://proton.me/blog/centalized-vs-p2p-protondrive).

Le chemin technique comporte six niveaux :

1. **Identité et session :** le client authentifie un compte Proton, charge des clés d’adresse chiffrées et les déverrouille localement.
2. **Volume et partage :** un volume délimite l’espace de stockage. Les partages définissent les racines d’arborescence, les membres et les droits.
3. **Nœud et lien :** un fichier ou un dossier est un nœud ; le lien le place dans la hiérarchie chiffrée.
4. **Révision et bloc :** un fichier peut posséder plusieurs révisions. Le contenu est découpé en blocs et chiffré côté client.
5. **API et stockage d’objets :** le client transmet les métadonnées chiffrées, les paquets de clés, les manifestes et les blocs via HTTPS.
6. **Système de fichiers local :** les clients de bureau projettent l’arborescence cloud sous forme d’espace réservé ou de fichier local, surveillent les modifications et résolvent les conflits.

Le [modèle de sécurité de Proton Drive](https://proton.me/blog/protondrive-security) documente le volume, le partage, le nœud, le lien et les clés hiérarchiques. La [mise à jour cryptographique de 2026](https://proton.me/blog/drive-cryptography-update) décrit en outre le nœud, la révision, le bloc et le paquet de clés de contenu du moteur client partagé.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-proton-drive.svg?v=20260813" title="Interaktive Infografik: Proton Drive von Konto und Share über Node-, Revisions- und Blockverschlüsselung bis zu API, SDK, Desktop-Sync, CLI, Sharing, Versionierung, Recovery und unabhängiger Sicherung" loading="lazy">
  <a href="/images/kb-interaktiv-proton-drive.svg?v=20260813">Ouvrir directement le graphique interactif</a>.
</iframe>

## Modèle de données : volume, partage, nœud, lien et révision

Un **volume** est un espace de stockage attribué. Son partage par défaut pointe vers la racine de l’arborescence privée. D’autres **partages** peuvent pointer vers des fichiers ou des dossiers et associer des membres à des rôles. Un partage constitue donc à la fois un point d’accès et une frontière cryptographique.

Un **nœud** décrit un fichier ou un dossier et contient des attributs techniques ainsi que du matériel cryptographique. Un **lien** positionne le nœud dans l’arborescence et le relie à son nœud parent. Pour les fichiers, les **révisions** gèrent les états successifs du contenu ; une révision référence des **blocs** chiffrés. Cette séparation terminologique explique des phénomènes opérationnels importants :

- Renommer modifie la désignation chiffrée dans l’arborescence, mais pas nécessairement le contenu du fichier.
- Déplacer modifie la relation parent-enfant et donc les informations de lien protégées cryptographiquement.
- Remplacer crée une révision du même objet fichier, tandis que « conserver les deux » crée un second nœud.
- Partager délègue l’accès à une racine de partage au lieu de déchiffrer côté serveur tous les contenus qu’elle contient.
- Supprimer déplace d’abord une entrée vers la corbeille Proton Drive ; les corbeilles locales constituent un état distinct.

La [gestion des versions](https://proton.me/support/version-history) officielle décrit Replace, Keep both, Skip et la restauration des révisions antérieures. La [documentation sur la suppression et la restauration](https://proton.me/support/proton-drive-delete-restore-synced-files) explique la différence entre la corbeille cloud et la corbeille locale.

Le volume, le partage, le nœud et la révision décrivent quel objet est stocké. La hiérarchie de clés explique ensuite qui peut déchiffrer et signer les noms, les métadonnées et le contenu.

## Hiérarchie de clés et OpenPGP

Les clés et les phrases secrètes sont générées sur le client et ne sont transmises au serveur que sous forme chiffrée. Le modèle de sécurité utilise plusieurs niveaux :

1. Une adresse Proton possède une paire de clés asymétriques.
2. Pour un partage, une phrase secrète de partage aléatoire et une clé de partage sont générées.
3. La phrase secrète de partage est chiffrée et signée, pour chaque membre autorisé, avec sa clé d’adresse.
4. Chaque nœud de fichier ou de dossier possède des clés de nœud et une phrase secrète de nœud.
5. La phrase secrète du nœud est protégée par la clé du dossier parent ou de la racine du partage.
6. Le nom et les autres charges utiles sont chiffrés dans ce contexte hiérarchique.
7. Le contenu des fichiers utilise une clé de contenu symétrique dont le paquet de clés est protégé par la clé du nœud.

OpenPGP sert ici de format de messages et de paquets de clés. La cryptographie asymétrique protège les clés de session symétriques ; le chiffrement symétrique traite efficacement les données elles-mêmes. La base normative d’OpenPGP est [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580).

Pour les fichiers nouvellement créés depuis la migration cryptographique, Proton décrit des paquets v6-PKESK, v2-SEIPD et AES-GCM. Le schéma d’un fichier reste lié à ses révisions : lors d’un remplacement, un client doit utiliser le schéma de paquet correspondant à l’objet fichier existant, faute de quoi la révision est rejetée ([mise à jour cryptographique](https://proton.me/blog/drive-cryptography-update)). C’est une raison d’interopérabilité pour mettre à jour les clients de manière contrôlée.

## Chiffré, signé et lisible par le serveur

« Chiffré de bout en bout » ne signifie pas que le serveur ne voit aucune donnée d’exploitation. Dans la [vue d’ensemble de l’assistance Drive](https://proton.me/support/drive), Proton désigne le contenu des fichiers ainsi que les noms de fichiers et de dossiers comme E2EE, tout en indiquant que certains champs, tels que la taille des fichiers, restent lisibles par le serveur afin de permettre des fonctions comme le tri.

Pour un modèle de menace, les classes de données sont séparées :

| Classe de données | Chemin de protection | Pertinence pour l’administration |
|---|---|---|
| Contenu des fichiers | blocs chiffrés côté client | le serveur ne peut pas indexer ni réparer le contenu |
| Nom de fichier/dossier | charge utile chiffrée dans l’arborescence | la recherche par nom côté serveur est limitée |
| Paquets de clés | chiffrés et signés | la récupération dépend des clés privées |
| Signatures/manifestes | preuve d’intégrité et d’auteur | les avertissements ne doivent pas être ignorés systématiquement |
| Taille des fichiers | champ lisible par le serveur | les schémas de volume et de trafic restent visibles |
| Compte, session et relation de partage | métadonnées du plan de contrôle | l’autorisation et l’audit restent côté serveur |
| IP, moment, version du client | métadonnées opérationnelles/de sécurité | font partie du modèle de menace du fournisseur et du terminal |

Les signatures numériques confirment l’émetteur du téléversement et l’intégrité des charges utiles. La [gestion des signatures](https://proton.me/support/drive-signature-management) explique la vérification des signatures OpenPGP et les avertissements. Une signature ne prouve pas que le contenu est inoffensif ou techniquement correct ; elle atteste son origine ou son absence de modification dans le contexte de clé vérifié.

## Blocs, révisions et limite de validation

Les fichiers sont découpés en blocs côté client. Le client génère ou reprend la clé de contenu, chiffre les blocs, les transfère et publie la révision avec les informations cryptographiques associées. Le serveur peut stocker et distribuer les blocs sans connaître le texte en clair.

La progression du téléversement ne constitue pas encore une validation fonctionnelle. Un administrateur distingue :

- fichier local entièrement lu,
- tous les blocs chiffrés,
- tous les blocs transférés avec succès,
- révision acceptée côté serveur,
- curseur d’événements avancé,
- fichier résoluble sur un second client,
- signature et contenu vérifiés lors du téléchargement.

Lors de téléchargements volumineux, le client déchiffre et vérifie les données localement. Pour des données importantes, le client web utilise un chemin de streaming avec Service Worker afin de ne pas devoir conserver l’intégralité de l’archive dans la mémoire du navigateur ; Proton décrit ce processus dans [téléchargement web et streaming](https://proton.me/blog/proton-drive-web-encryption-technique).

## Drive SDK et architecture des clients

L’écosystème client est issu à l’origine d’implémentations spécifiques aux plateformes. Le **Drive SDK** commun unifie les chemins de compte, d’API, de cache, de nœuds, de partages, de téléversement, de téléchargement et de cryptographie. Le code source se trouve dans le [référentiel officiel Proton Drive SDK](https://github.com/ProtonDriveApps/sdk) ; le contrat de paquet JavaScript/TypeScript est publié sous [`@protontech/drive-sdk`](https://www.npmjs.com/package/@protontech/drive-sdk).

L’organisation GitHub officielle présente les principales piles technologiques ([ProtonDriveApps](https://github.com/ProtonDriveApps)) :

| Client/composant | Pile principale | Rôle opérationnel |
|---|---|---|
| Web | TypeScript/JavaScript et cryptographie du navigateur | accès interactif, streaming, partage |
| Drive SDK / CLI | TypeScript ; CLI empaquetée avec Bun en programme autonome | opérations communes sur les fichiers et automatisation |
| [Windows](https://github.com/ProtonDriveApps/windows-drive) | C#/.NET et composants SDK/Crypto communs | synchronisation en arrière-plan, Explorer et Cloud Files |
| [macOS](https://github.com/ProtonDriveApps/mac-drive) | Swift et composants SDK communs | synchronisation en arrière-plan, Finder/File Provider |
| Android | Kotlin | fichiers mobiles, état hors ligne et photos |
| iOS/iPadOS | Swift | fichiers mobiles, état hors ligne et photos |

L’approche SDK réduit les divergences entre les implémentations de cryptographie et de fichiers, sans supprimer les limites des plateformes. Les règles de nommage Windows, le comportement de macOS File Provider, les limites d’exécution en arrière-plan sur mobile et l’intégration de bureau Linux restent des tâches propres à chaque client.

L’API cryptographique n’est pas encore un service de synchronisation de fichiers. Le client de bureau surveille les modifications locales et distantes, les rapproche et décide d’un téléversement, d’un téléchargement ou d’un conflit.

## Synchronisation de bureau comme machine à états

La synchronisation de bureau ne maintient pas une simple copie d’octets, mais réconcilie deux espaces de noms variables. Un cycle abstrait est le suivant :

1. Charger l’état du système de fichiers local et le curseur d’événements cloud.
2. Déterminer les modifications locales, les événements côté serveur et les transferts en attente.
3. Normaliser les noms et les règles de plateforme ou signaler les conflits.
4. Résoudre la hiérarchie de clés et les nœuds requis.
5. Planifier le téléversement, le téléchargement, le déplacement, la suppression ou l’hydratation.
6. Transférer le contenu par blocs et valider la révision.
7. Mettre à jour de façon atomique les espaces réservés locaux, les attributs, le cache et le curseur.
8. Réessayer les erreurs avec une référence d’objet stable ou les faire remonter de manière visible.

L’état de synchronisation « synchronisé » n’est pertinent que pour le client et le curseur considérés. Un second client, un partage ou une sauvegarde distincte fournit une preuve de bout en bout plus solide.

## Fichiers à la demande et états locaux

Windows et macOS affichent l’arborescence cloud dans le gestionnaire de fichiers sans stocker en permanence tous les contenus localement. La [documentation Windows On-Demand](https://proton.me/support/proton-drive-windows-on-demand-sync) distingue **Available when online**, **Available on this device** et **Always available on this device**. Le [guide macOS](https://proton.me/support/drive-macos-guide) distingue Online Only, temporairement local et Make Available Offline.

Un nom de fichier visible ne prouve donc pas l’existence d’une copie locale. Pour une sauvegarde, un antivirus, un indexeur, un système de build ou un flux de pièces jointes, il faut déterminer si le fichier :

- n’existe que comme espace réservé,
- est hydraté à l’ouverture,
- est temporairement mis en cache localement,
- a été explicitement épinglé pour le mode hors ligne,
- a été modifié localement mais n’est pas encore téléversé,
- ne se trouve que sur cet appareil en raison d’une erreur.

L’imbrication de deux clients cloud à la demande dans la même arborescence est risquée. L’[aide Windows sur les erreurs](https://proton.me/support/proton-drive-windows-common-issues) cite explicitement les fichiers d’un autre stockage cloud comme problématiques lorsqu’ils ne sont disponibles qu’en ligne. Une tâche de sauvegarde épingle localement le jeu de données source ou utilise la CLI officielle pour des transferts explicites.

## Conflits, noms et fonctionnement multi-appareils

Des conflits apparaissent lorsque le client et le cloud modifient indépendamment le même nœud logique ou lorsque la plateforme cible ne peut pas représenter un nom. Les classes typiques sont :

- modification parallèle du contenu de la même révision,
- suppression contre modification,
- renommage contre déplacement,
- majuscules/minuscules sur des plateformes ayant une autre sémantique de casse,
- caractères interdits ou noms réservés sous Windows,
- téléversements homonymes avec Replace, Keep both ou Skip,
- modification hors ligne après un long retard d’événements,
- fichier local verrouillé par une application pendant le transfert.

Proton documente qu’un conflit peut créer un fichier renommé ou une nouvelle version ([Shared with me sous Windows](https://proton.me/support/shared-with-me-windows)). L’aide Windows sur les erreurs décrit les fichiers ouverts et les noms interdits ([problèmes de synchronisation fréquents](https://proton.me/support/proton-drive-windows-common-issues)). Les administrateurs ne recherchent pas seulement des « erreurs », mais aussi des copies de conflit, des téléversements en attente et des fichiers qui ne sont jamais arrivés dans le cloud à cause des règles de nommage.

## Inventaire client, binaire et journaux

Un terminal géré inventorie la plateforme, le paquet client, la signature, le démarrage automatique, le compte, la racine de synchronisation, la disponibilité locale, l’espace libre et le chemin des journaux. Les numéros de version font partie de l’inventaire d’exécution et non des faits statiques de l’article.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Proton-Drive-Clientinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Process ProtonDrive -ErrorAction SilentlyContinue
Get-AuthenticodeSignature "$env:LOCALAPPDATA\Proton\Drive\ProtonDrive.exe"
Get-ChildItem "$env:LOCALAPPDATA\Proton\ProtonDrive\Logs"
proton-drive version</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">proton-drive version
file ./proton-drive
sha256sum ./proton-drive
ldd ./proton-drive</code></pre>
  </div>
</div>

[`Get-Process`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-process) vérifie le processus Windows ; [`Get-AuthenticodeSignature`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/get-authenticodesignature) la signature de code ; [`Get-ChildItem`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-childitem) inventorie les journaux. Sous Unix, [`file`](https://man7.org/linux/man-pages/man1/file.1.html) identifie le binaire, [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) son hachage et [`ldd`](https://man7.org/linux/man-pages/man1/ldd.1.html) les dépendances dynamiques. L’accès aux journaux pris en charge pour Windows et macOS est décrit dans [Proton Drive Logs](https://proton.me/support/proton-drive-logs).

## CLI : opération explicite plutôt que synchronisation en arrière-plan

La Proton Drive CLI est un client basé sur le SDK pour Windows, macOS et Linux. La connexion s’effectue via un flux de navigateur ; selon l’[introduction à la CLI](https://proton.me/blog/proton-drive-cli), les sessions sont stockées dans Windows Credential Manager, macOS Keychain ou libsecret. La CLI liste, téléverse et télécharge, gère les fichiers, la corbeille, les partages et les invitations. Elle s’exécute comme opération individuelle puis se termine.

La limite opérationnelle essentielle est explicitement documentée : la CLI ne remplace pas un moteur de synchronisation de bureau. Un téléversement planifié ne détecte pas automatiquement les suppressions, renommages, conflits bidirectionnels ni le succès sur un second appareil. Quiconque utilise la CLI comme transport de sauvegarde implémente lui-même l’inventaire source, la direction, la règle de suppression, les répétitions, le code de sortie, la vérification de destination et la conservation.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Anmeldung und Proton-Drive-Inventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">proton-drive auth login
proton-drive filesystem list /my-files
proton-drive filesystem list /my-files --json | ConvertFrom-Json
proton-drive sharing status /my-files/Reports</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">proton-drive auth login
proton-drive filesystem list /my-files
proton-drive filesystem list /my-files --json | jq .
proton-drive sharing status /my-files/Reports</code></pre>
  </div>
</div>

Toutes les commandes et aides CLI sont documentées dans [Using Proton Drive CLI](https://proton.me/support/drive-cli). [`ConvertFrom-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json) et [`jq`](https://jqlang.org/manual/) traitent les sorties lisibles par machine. Les automatisations s’appuient sur les champs JSON et le code de sortie, et non sur des textes d’écran localisés.

## Téléversement, stratégie de conflit et vérification

Un téléversement reproductible indique la source locale, la racine de destination et la stratégie de conflit. Les caractères génériques sont développés par le shell appelant et peuvent fournir des ensembles différents sous Windows et Unix. Les tâches critiques créent donc à l’avance un manifeste contenant le chemin, la taille et le hachage.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Proton-Drive-Upload und Manifest">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-ChildItem .\reports -File -Recurse |
  Get-FileHash -Algorithm SHA256 |
  Export-Csv .\reports-manifest.csv -NoTypeInformation
proton-drive filesystem upload .\reports\* /my-files/Reports --conflict-strategy skip
proton-drive filesystem list /my-files/Reports --json | Out-File .\drive-result.json</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">find ./reports -type f -print0 | sort -z | xargs -0 sha256sum > reports-manifest.sha256
proton-drive filesystem upload ./reports/* /my-files/Reports --conflict-strategy skip
proton-drive filesystem list /my-files/Reports --json > drive-result.json
jq . drive-result.json</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash), [`Export-Csv`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/export-csv), [`find`](https://man7.org/linux/man-pages/man1/find.1.html), [`sort`](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html) et [`xargs`](https://man7.org/linux/man-pages/man1/xargs.1.html) créent le jeu attendu local. La liste du fournisseur ne prouve toutefois pas encore l’égalité des contenus. Pour les données critiques, un téléchargement vers une destination distincte et une comparaison de hachages suivent.

## Téléchargement et réception de bout en bout

Un test de téléchargement utilise un nouveau chemin de destination afin qu’aucun fichier local existant ne masque le résultat. Il est ensuite contrôlé par rapport au manifeste source. En présence de plusieurs révisions, le test doit préciser quelle révision est attendue.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Proton-Drive-Downloadprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">New-Item -ItemType Directory .\restore-test
proton-drive filesystem download /my-files/Reports .\restore-test
Get-ChildItem .\restore-test -File -Recurse |
  Get-FileHash -Algorithm SHA256 |
  Export-Csv .\restore-manifest.csv -NoTypeInformation</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">mkdir -p ./restore-test
proton-drive filesystem download /my-files/Reports ./restore-test
find ./restore-test -type f -print0 | sort -z | xargs -0 sha256sum > restore-manifest.sha256
diff -u reports-manifest.sha256 restore-manifest.sha256</code></pre>
  </div>
</div>

[`New-Item`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/new-item) et [`mkdir`](https://www.gnu.org/software/coreutils/manual/html_node/mkdir-invocation.html) créent la destination isolée ; [`diff`](https://www.gnu.org/software/diffutils/manual/html_node/Invoking-diff.html) compare les manifestes. Les chemins absolus ou préfixés différemment sont normalisés avant la comparaison.

## Partage avec des comptes et via des liens publics

Proton Drive connaît au moins deux modèles de partage :

- **Adhésion/invitation :** une personne devient membre d’un partage avec le rôle Viewer ou Editor. La phrase secrète du partage est empaquetée pour la clé d’adresse du membre.
- **Lien public :** un lien permet l’accès sans compte Proton. Un mot de passe protège la phrase secrète du partage ; le lien et le mot de passe constituent ensemble le secret d’accès.

La [gestion des accès](https://proton.me/support/drive-manage-access-shared-files) documente les rôles, la délégation Editor, la révocation et les paramètres séparés pour l’invitation par e-mail et le Public Link. [Mot de passe et date d’expiration](https://proton.me/support/password-protect-files-proton-drive) limitent l’accès par lien. Si un mot de passe aléatoire est transmis comme fragment d’URL, le navigateur et le destinataire voient le lien complet ; le fragment n’est certes pas envoyé au serveur comme chemin HTTP, mais peut fuiter via le presse-papiers, l’historique du navigateur, des captures d’écran ou la télémétrie des terminaux.

Les administrateurs inventorient le propriétaire, la racine de partage, les membres, le rôle, le droit de partage, l’état du lien public, la date d’expiration et la dernière vérification. Selon les paramètres, un Editor peut lui-même partager et modifier les droits. Lors d’un départ, on ne révoque donc pas seulement le compte, mais on vérifie tous les partages et liens externes.

## Versions, corbeille et sauvegarde

L’historique des versions stocke les révisions antérieures du même objet fichier. Une révision restaurée ne supprime pas automatiquement les versions plus récentes. La corbeille conserve les nœuds supprimés jusqu’à leur suppression définitive. Ces deux fonctions sont précieuses, mais elles restent dans le même domaine de défaillance administratif et cryptographique.

Elles ne protègent pas de manière fiable contre :

- la perte ou l’impossibilité de récupérer la clé du compte,
- des droits erronés ou une suppression définitive malveillante,
- des événements de quota, de contrat ou de fournisseur,
- des erreurs survenues avant la première révision cloud réussie,
- des omissions de synchronisation non détectées,
- la compromission de l’identité ou des terminaux à l’échelle de l’organisation.

Une sauvegarde indépendante lit un ensemble cloud défini à l’aide d’un client autorisé, l’écrit dans une destination gérée séparément, l’accompagne d’un manifeste, d’une politique de conservation et d’un historique immuable, puis effectue des tests de restauration. NIST SP 800-34 traite la sauvegarde, le stockage alternatif et les tests de récupération comme des contrôles liés de planification de continuité ([NIST SP 800-34 Rev. 1](https://csrc.nist.gov/pubs/sp/800/34/r1/final)). Les principes de base sont présentés dans [Backup and Disaster Recovery](/kb/backup-dr) et [Storage](/kb/storage).

## Identité, session et confiance dans les terminaux

Le chiffrement de bout en bout déplace la confiance du fournisseur vers le client, le compte et le terminal. Quiconque contrôle l’appareil déverrouillé, la session du navigateur ou le compte peut déchiffrer et signer des modifications au nom de l’utilisateur. C’est pourquoi MFA, clés de sécurité matérielles, protection des appareils, révocation des sessions, origine du client et protection du cache local doivent être considérés ensemble.

Le [modèle de menace](https://proton.me/blog/proton-drive-threat-model) cite notamment les appareils compromis, le phishing et l’authenticité des clients officiels parmi ses limites. L’ouverture du code source permet son examen, mais ne remplace pas la vérification du binaire réellement exécuté et de son chemin de mise à jour.

L’analyse des logiciels malveillants est également limitée : la [détection facultative de fichiers malveillants](https://proton.me/support/proton-drive-malware-protection) compare à la demande un hachage à des hachages de logiciels malveillants connus. Elle ne déchiffre pas l’ensemble du contenu côté serveur, ne détecte pas les contenus inconnus et ne remplace pas un capteur de terminal.

## Réinitialisation de mot de passe et récupération des données

La récupération du compte et la récupération des données sont deux procédures différentes. Une réinitialisation de mot de passe peut restaurer l’accès au compte, mais rendre dans un premier temps les anciennes clés privées inaccessibles. Proton Drive affiche alors les anciennes données comme verrouillées.

La [récupération des données Drive](https://proton.me/support/drive-data-recovery) officielle cite le mot de passe précédent, la Recovery Phrase ou le Recovery File comme moyens de déverrouillage. Les nouveaux téléversements après réinitialisation peuvent être créés sous de nouvelles clés, tandis que les anciennes données restent verrouillées ; les photos et albums peuvent nécessiter une étape de fusion supplémentaire.

Un runbook organisationnel documente donc :

- qui est autorisé à déclencher une réinitialisation de mot de passe,
- quelles méthodes de récupération sont configurées et testées au préalable,
- où la Recovery Phrase ou le Recovery File sont conservés hors ligne et protégés par contrôle d’accès,
- comment identifier les anciens et nouveaux états de clés,
- comment vérifier les partages et les membres après récupération,
- quand un Drive verrouillé ne doit en aucun cas être supprimé.

« Unlock later » préserve la possibilité d’une récupération ultérieure. « Delete locked drive » est une suppression irréversible des données et ne doit pas figurer dans une procédure standard non confirmée.

Lorsque l’identité, les clés et l’état local sont plausibles, le chemin réseau vient ensuite. Le proxy, TLS, les points de terminaison API et les transferts bloqués doivent être vérifiés séparément du modèle de données cryptographique.

## Chemin réseau et diagnostic du transport

HTTPS protège le transport entre le client et l’API Proton ou les points de terminaison de stockage. Il ne remplace pas le chiffrement de bout en bout, mais protège les métadonnées de session et d’exploitation, l’identité du point de terminaison et la disponibilité. Un proxy, un filtre DNS, une inspection TLS, un VPN, une MTU ou des connexions instables peuvent affecter différemment les requêtes du plan de contrôle et les transferts de blocs.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Proton-Drive-Netzwerkdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName drive.proton.me
Test-NetConnection drive.proton.me -Port 443
curl.exe -sS -D - -o NUL https://drive.proton.me/
Get-NetTCPConnection -State Established | Where-Object RemotePort -eq 443</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">getent ahosts drive.proton.me
curl -sS -D - -o /dev/null https://drive.proton.me/
ss -ntp state established '( dport = :443 )'
openssl s_client -connect drive.proton.me:443 -servername drive.proton.me -brief</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) et [`getent`](https://man7.org/linux/man-pages/man1/getent.1.html) vérifient la résolution de noms ; [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) TCP 443 ; [`curl`](https://curl.se/docs/manpage.html) HTTP et les redirections ; [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) ou [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) les connexions actives ; [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) TLS et SNI. Le test sur le domaine web ne prouve pas que chaque point de terminaison API/de blocs est accessible ; les journaux du client fournissent la cible et la classe de requête ayant réellement échoué.

## Observabilité et chaîne de diagnostic

Un diagnostic pertinent commence par l’objet fonctionnel :

1. Déterminer le compte, le partage, le chemin logique et le client concerné.
2. Distinguer l’état local : espace réservé, hydraté, épinglé, modifié, verrouillé ou défaillant.
3. Vérifier la vue d’activité et les journaux pour les références de nœud, révision ou transfert.
4. Contrôler l’espace local libre et le quota cloud.
5. Vérifier la compatibilité des noms et des chemins.
6. Vérifier DNS, TCP, TLS et proxy/VPN avec le véritable point de terminaison.
7. Traiter séparément les avertissements de signature, de déchiffrement et de clé des erreurs de transport.
8. Lister ou télécharger le fichier avec la CLI officielle ou un second client.
9. Comparer le hachage téléchargé et la révision attendue.
10. N’envisager qu’ensuite la pause/reprise, la réauthentification, la mise à jour ou une intervention sur le cache.

Les [erreurs de synchronisation Windows](https://proton.me/support/proton-drive-windows-common-issues) mentionnent la progression, les fichiers ouverts, d’autres clients cloud, la pause/reprise, les règles de nommage et le réseau. En exploitation d’administration, ces aides destinées aux utilisateurs sont complétées par un inventaire d’objets et une vérification de bout en bout. La méthodologie générale est présentée dans [Troubleshooting](/kb/troubleshooting).

## Modèle de performance et de capacité

Les coûts ne résultent pas uniquement des octets transférés. De nombreux petits fichiers génèrent du travail de clés, métadonnées, API et signatures pour chaque nœud. Les arborescences profondes exigent une résolution répétée de la hiérarchie. Un grand contenu de fichier peut en revanche être traité plus efficacement dans des blocs parallèles. Proton indique dans l’aide Windows sur les erreurs qu’un grand nombre de petits fichiers ou des fichiers fortement imbriqués peuvent se synchroniser plus lentement qu’un seul fichier de même taille.

La planification des capacités prend en compte :

- le quota cloud, y compris les versions et la corbeille,
- les états locaux hors ligne et de cache,
- l’espace temporaire pour le téléversement, le téléchargement et les copies de conflit,
- le CPU pour la cryptographie et le hachage,
- le nombre de fichiers et la profondeur de l’arborescence,
- le retard d’événements après des périodes hors ligne,
- le comportement de Fair Use/de limitation lors des transferts automatisés.

L’introduction à la CLI avertit de ne pas téléverser entièrement et de manière répétée des données inchangées, et renvoie à Fair Use ([Proton Drive CLI](https://proton.me/blog/proton-drive-cli)). Une tâche planifiée maintient donc son propre manifeste source ou vérifie l’état cible, au lieu de traiter chaque exécution comme un téléversement complet.

## Exploitation avec Windows, macOS et Linux

Windows et macOS proposent une intégration au gestionnaire de fichiers et une synchronisation en arrière-plan. Les applications mobiles gèrent le téléversement, le téléchargement, les objets hors ligne et les photos, mais ne constituent pas un remplacement identique d’un service de système de fichiers de bureau. La CLI officielle rend les opérations sur les fichiers et les partages scriptables sur les trois systèmes d’exploitation de bureau.

En juin 2026, Proton a publié la CLI et désigné une application Linux de bureau complète avec synchronisation en arrière-plan comme un projet distinct encore en développement ([annonce de la CLI](https://proton.me/blog/proton-drive-cli)). Cette distinction historique reste importante : même si l’offre de plateformes évolue, **opération CLI**, **montage** et **moteur de synchronisation bidirectionnel** ne sont pas assimilés. Pour chaque déploiement, la [page des notes de version](https://proton.me/support/drive-release-notes) est vérifiée comme source dépendante du temps.

Dans les tâches serveur, l’authentification navigateur s’exécute de manière contrôlée et le jeton de session résultant est enregistré dans le magasin de secrets du système d’exploitation d’un compte de service dédié. Une tâche sans interface graphique ne reçoit pas le répertoire personnel d’un administrateur. Le code de sortie, le résultat JSON, le manifeste source et la vérification de destination sont conservés ; les secrets et les liens de partage sont retirés des journaux.

Les versions et la corbeille protègent contre certaines erreurs de manipulation, mais ne remplacent pas une récupération indépendante. Le runbook doit traiter ensemble le compte, l’accès aux clés, les données locales et une exportation vérifiée.

## Runbook de sauvegarde et de récupération

Proton Drive peut être la destination d’une copie de sauvegarde, mais ne remplace pas une conception de récupération. Un runbook robuste comprend :

1. Définir l’ensemble source et la source de vérité.
2. Hydrater entièrement les données locales à la demande avant une sauvegarde basée sur des fichiers, ou les exporter directement avec la CLI.
3. Créer un manifeste de fichiers avec chemin, taille et hachage.
4. Exécuter le téléversement avec une règle explicite de conflit et de suppression.
5. Archiver le code de sortie CLI et la liste de destination.
6. Effectuer un échantillonnage ou un téléchargement complet vers un chemin isolé.
7. Comparer les hachages et les révisions attendues.
8. Documenter les partages, le versionnage, la corbeille et la conservation.
9. Tester séparément la récupération du compte Proton et des données.
10. Conserver une seconde copie indépendante en dehors du même domaine de compte et de fournisseur.

Le RPO se rapporte à la dernière révision dont la validation et la vérification dans Proton sont démontrées, et non au moment de l’écriture locale du fichier. Le RTO comprend l’accès au compte, le déverrouillage des clés, le listing, le téléchargement, le déchiffrement, la bande passante et la restauration de destination.

## Historique technique

Proton a développé Drive comme complément à son écosystème Proton chiffré. Le modèle de sécurité publié en 2020 a établi un stockage centralisé avec une hiérarchie de clés OpenPGP côté client, des noms chiffrés, des arborescences de nœuds, des partages, des blocs signés et des partages par lien. Les clients web et mobiles ont précédé les clients de synchronisation de bureau ; l’historique des versions a été introduit en 2023 comme chemin de révision distinct ([introduction de l’historique des versions](https://proton.me/blog/drive-version-history)).

Windows et macOS ont ajouté la synchronisation en arrière-plan et les fichiers à la demande avec une intégration Explorer/Finder spécifique à chaque plateforme. Les implémentations séparées ont progressivement conduit au Drive SDK commun. Début 2026, Proton a décrit le déploiement du SDK sur le web, Windows, macOS, Android et iOS ([développement du SDK](https://proton.me/blog/drive-sdk-january-2026)).

En juin 2026, deux évolutions techniques ont suivi : une migration cryptographique commune vers v6 PKESK, v2 SEIPD et AES-GCM pour les nouveaux objets fichiers, ainsi que la CLI basée sur le SDK pour Windows, macOS et Linux. Cela a permis un accès officiellement scriptable sans recréer la logique client cryptographique. L’architecture reste néanmoins centrée sur le client : les serveurs stockent et coordonnent, les clients autorisés déchiffrent, signent et réconcilient.

## Liste de contrôle d’administration

Avec un stockage chiffré côté client, le modèle de données, les clés, la synchronisation et la sauvegarde indépendante doivent être compris conjointement. Cette liste de contrôle condense ces éléments pour l’autorisation d’exploitation.

- **Modèle de données :** distinguer volume, partage, nœud, lien, révision et bloc.
- **Cryptographie :** documenter le contexte des clés d’adresse, de partage, de nœud et de contenu, ainsi que le schéma de paquet.
- **Métadonnées :** séparer les champs E2EE et les champs opérationnels lisibles par le serveur dans le modèle de menace.
- **Clients :** inventorier la plateforme, la technologie, le SDK, l’intégration du système de fichiers, la signature binaire et le chemin des journaux.
- **Synchronisation :** surveiller l’état local, le curseur d’événements, les copies de conflit, les noms et les transferts en attente.
- **CLI :** ne pas traiter une opération explicite sur un fichier comme un moteur de synchronisation ou de sauvegarde permanent.
- **Partage :** vérifier le propriétaire, les membres, les rôles, la délégation Editor, le Public Link, le mot de passe et l’expiration.
- **Récupération :** tester séparément le mot de passe du compte, le déverrouillage des données, la Recovery Phrase/Recovery File et l’accès aux partages.
- **Sauvegarde :** prévoir une copie indépendante, un manifeste, un versionnage et un téléchargement de restauration.
- **Réception :** déchiffrer et ouvrir le fichier sur un second client ou depuis un téléchargement isolé, après vérification de la signature et du hachage.

## Sources

- [NIST SP 800-34 Rev. 1 – Guide de planification de continuité](https://csrc.nist.gov/pubs/sp/800/34/r1/final)
- [Proton Drive – Vue d’ensemble de l’assistance](https://proton.me/support/drive)
- [Proton Drive – CLI](https://proton.me/support/drive-cli)
- [Proton – Stockage centralisé et décentralisé](https://proton.me/blog/centalized-vs-p2p-protondrive)
- [Proton Drive – Modèle de sécurité](https://proton.me/blog/protondrive-security)
- [Proton Drive – Mise à jour cryptographique](https://proton.me/blog/drive-cryptography-update)
- [Proton Drive – Gestion des versions](https://proton.me/support/version-history)
- [Proton Drive – Supprimer et restaurer des fichiers synchronisés](https://proton.me/support/proton-drive-delete-restore-synced-files)
- [RFC 9580 – OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580)
- [Proton Drive – Gestion des signatures](https://proton.me/support/drive-signature-management)
- [Proton Drive – Téléchargement web et streaming](https://proton.me/blog/proton-drive-web-encryption-technique)
- [Proton Drive – Référentiel SDK](https://github.com/ProtonDriveApps/sdk)
- [npm – @protontech/drive-sdk](https://www.npmjs.com/package/@protontech/drive-sdk)
- [GitHub – ProtonDriveApps](https://github.com/ProtonDriveApps)
- [Windows](https://github.com/ProtonDriveApps/windows-drive)
- [macOS](https://github.com/ProtonDriveApps/mac-drive)
- [Proton Drive – Synchronisation Windows On-Demand](https://proton.me/support/proton-drive-windows-on-demand-sync)
- [Proton Drive – Guide macOS](https://proton.me/support/drive-macos-guide)
- [Proton Drive – Problèmes de synchronisation Windows fréquents](https://proton.me/support/proton-drive-windows-common-issues)
- [Proton Drive – Shared with me sous Windows](https://proton.me/support/shared-with-me-windows)
- [Microsoft – Get-Process](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-process)
- [Microsoft – Get-AuthenticodeSignature](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/get-authenticodesignature)
- [Microsoft – Get-ChildItem](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-childitem)
- [Pages de manuel Linux – file(1)](https://man7.org/linux/man-pages/man1/file.1.html)
- [GNU Coreutils – Utilitaires sha2](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Pages de manuel Linux – ldd(1)](https://man7.org/linux/man-pages/man1/ldd.1.html)
- [Proton Drive – Journaux](https://proton.me/support/proton-drive-logs)
- [Proton – Introduction à la Drive CLI](https://proton.me/blog/proton-drive-cli)
- [Microsoft – ConvertFrom-Json](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json)
- [jq – Manuel](https://jqlang.org/manual/)
- [Microsoft – Get-FileHash](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash)
- [Microsoft – Export-Csv](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/export-csv)
- [Pages de manuel Linux – find(1)](https://man7.org/linux/man-pages/man1/find.1.html)
- [GNU Coreutils – sort](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html)
- [Pages de manuel Linux – xargs(1)](https://man7.org/linux/man-pages/man1/xargs.1.html)
- [Microsoft – New-Item](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/new-item)
- [GNU Coreutils – mkdir](https://www.gnu.org/software/coreutils/manual/html_node/mkdir-invocation.html)
- [GNU Diffutils – diff](https://www.gnu.org/software/diffutils/manual/html_node/Invoking-diff.html)
- [Proton Drive – Gestion des accès](https://proton.me/support/drive-manage-access-shared-files)
- [Proton Drive – Protection par mot de passe et date d’expiration](https://proton.me/support/password-protect-files-proton-drive)
- [Proton Drive – Modèle de menace](https://proton.me/blog/proton-drive-threat-model)
- [Proton Drive – Détection de fichiers malveillants](https://proton.me/support/proton-drive-malware-protection)
- [Proton Drive – Récupération des données](https://proton.me/support/drive-data-recovery)
- [Microsoft – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [Pages de manuel Linux – getent(1)](https://man7.org/linux/man-pages/man1/getent.1.html)
- [Microsoft – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [curl – Manuel](https://curl.se/docs/manpage.html)
- [Microsoft – Get-NetTCPConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection)
- [Pages de manuel Linux – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Proton Drive – Notes de version](https://proton.me/support/drive-release-notes)
- [Proton Drive – Introduction à l’historique des versions](https://proton.me/blog/drive-version-history)
- [Proton Drive – Développement du SDK 2026](https://proton.me/blog/drive-sdk-january-2026)
