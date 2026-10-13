---
title: "Rclone : remotes, comparaison de données et fonctionnement VFS"
blatt: "rclone"
description: "Rclone pour les administrateurs du stockage, des plateformes et de la messagerie : architecture Go et modèle de backend unifié, modèle de remote et de configuration, Copy/Sync/Move/Bisync, comparaison et sommes de contrôle, Crypt, montage VFS et cache, Retry/Pacer, filtres, API RC, journaux, sécurité, récupération et histoire technique."
fakten:
  - label: Rôle système
    wert: Outil CLI et API pour la copie, la synchronisation, la vérification, le montage et la mise à disposition entre stockages de fichiers et d’objets
    href: https://rclone.org/
  - label: Pile technologique
    wert: Projet open source en Go ; les backends et les commandes sont intégrés dans un binaire rclone portable
    href: https://github.com/rclone/rclone
  - label: Modèle d’adressage
    wert: remote nommé suivi de deux-points et d’un chemin ; les chemins locaux restent sans préfixe remote
    href: https://rclone.org/docs/
  - label: Contrat de backend
    wert: interface Fs/Object commune avec capacités optionnelles pour Hash, ModTime, Copy, Move, Polling, Metadata et ListR
    href: https://rclone.org/overview/
  - label: Direction
    wert: copy ajoute sans nettoyer la cible · sync fait de la cible le reflet de la source · move supprime la source seulement après une copie réussie
    href: https://rclone.org/commands/rclone_sync/
  - label: Comparaison
    wert: taille plus ModTime ou hash commun ; les flags et capacités du backend modifient l’empreinte
    href: https://rclone.org/overview/
  - label: Preuve d’intégrité
    wert: vérification du transfert, rclone check, cryptcheck ou comparaison par téléchargement ; le code de sortie et le journal final font partie du résultat
    href: https://rclone.org/commands/rclone_check/
  - label: Chemin de montage
    wert: VFS via FUSE/WinFsp ; Directory Cache, Buffer et File Cache se trouvent entre l’application et le remote
    href: https://rclone.org/commands/rclone_mount/
  - label: Chiffrement
    wert: Crypt chiffre les contenus avec NaCl SecretBox et aussi les noms par défaut ; la configuration, ou le mot de passe et le salt, sont des éléments de récupération
    href: https://rclone.org/crypt/
  - label: Pilotage
    wert: API Remote Control et endpoint OpenMetrics ; l’accessibilité équivaut pratiquement aux droits shell de l’utilisateur rclone
    href: https://rclone.org/rc/
  - label: Règle d’erreur
    wert: Retry de bas niveau, Pacer et retry de commande ; succès uniquement après l’état final et le code de sortie, pas après un message d’erreur isolé
    href: https://rclone.org/docs/#retries
  - label: Élément de récupération
    wert: Binaire, configuration, identifiants, filtres, définition du job, fichiers de travail Bisync, cache VFS, versions de destination et rapport de vérification
    href: https://rclone.org/docs/#configuration-encryption
werbung:
  - newsletter
ctaThemen:
  - rclone
translationSourceHash: bb7131a94d721cf6984869b9d907a4d2089758b80e651c8b03bd86f72841d42e
translationModel: gpt-5.6-terra
translatedAt: 2026-10-12T11:30:25.873Z
translationReview: automatic
---

# Rclone : remotes, comparaison de données et fonctionnement VFS

Rclone est un outil open source de déplacement de données pour les systèmes de fichiers locaux, les protocoles réseau, les lecteurs cloud et le stockage d’objets. La [page du projet Rclone](https://rclone.org/) le décrit comme un programme en ligne de commande destiné à gérer des fichiers sur des stockages cloud. Il offre à ces systèmes une interface commune pour le listing, la copie, la synchronisation, la vérification, le montage, le chiffrement et l’accès aux services. Cette uniformité est utile, mais ne doit pas être confondue avec une sémantique de stockage uniforme : les hashes, heures de modification, majuscules/minuscules, répertoires vides, métadonnées, copies côté serveur et la cohérence diffèrent selon le backend.

Pour les administrateurs, rclone n’est pas un « explorateur cloud », mais un pipeline de données. Sa correction dépend de la source et de la cible, de la direction, du critère de comparaison, de l’ensemble des filtres, de la règle de suppression, de l’état des retries, des identifiants, des capacités du backend et de la vérification finale. Un processus peut avoir transféré 100 pour cent des octets tout en étant fonctionnellement incorrect s’il utilise le mauvais espace de noms, une vue de listing obsolète ou un ensemble de filtres incomplet.

L’explication suit une commande rclone, de la source et du remote jusqu’au backend, en passant par la comparaison, les checkers et les transferts. Les filtres, Crypt, VFS, l’automatisation, la sécurité et la vérification sont ensuite expliqués le long du même cheminement de job.

Rclone traduit un modèle uniforme de fichiers et de répertoires dans les API de services de stockage très différents. L’article commence donc par les remotes et les capacités des backends, traite ensuite de la sémantique de copie et de la vérification, et se termine par le montage, la sécurité et la récupération.

## Architecture : commande, modèle de système de fichiers et backend

Le programme écrit en Go relie plusieurs couches :

1. **Commande et définition du job :** `copy`, `sync`, `move`, `check`, `bisync`, `mount`, `serve` ou un appel RC définissent la direction et l’opération.
2. **Filtres et comparaison :** les règles de chemin, la taille, l’heure de modification, le hash, les métadonnées et les flags déterminent l’ensemble d’objets considéré et l’égalité.
3. **Moteur de transfert :** les listers, checkers et workers de transfert planifient les opérations, limitent le parallélisme et collectent les erreurs.
4. **Modèle d’interfaces Fs/Object :** Rclone modélise les répertoires, objets et fonctionnalités optionnelles des backends au moyen d’interfaces Go communes.
5. **Adaptateurs de backend :** S3, SFTP, WebDAV, Drive, OneDrive, systèmes de fichiers locaux et autres services traduisent le contrat dans leurs API.
6. **Transport et fournisseur :** HTTP/TLS, SSH/SFTP ou E/S locale rencontrent les limites de débit, la cohérence, le versioning et les politiques du fournisseur.

La vue d’ensemble officielle des backends indique explicitement quels systèmes prennent en charge les hashes, ModTime, les noms insensibles à la casse, les doublons, les types MIME et les métadonnées ([Overview of cloud storage systems](https://rclone.org/overview/)). L’arborescence source sépare les backends, les commandes et le modèle d’interfaces `fs` commun ; les builds officiels regroupent les adaptateurs sélectionnés dans un binaire ([rclone source repository](https://github.com/rclone/rclone), [Install from source](https://github.com/rclone/rclone/blob/master/docs/content/install.md#source)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-rclone.svg?v=20260813" title="Interaktive Infografik: rclone-Job von Quelle, Remote und Filter über Listing, Vergleich, Checker, Transfer, Backend und Provider bis zu VFS, Crypt, Logs, Exitcode, Verifikation und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-rclone.svg?v=20260813">Ouvrir le graphique interactif directement</a>.
</iframe>

## Syntaxe des remotes et espace de noms

Un remote configuré est adressé sous la forme `name:pfad`. Les deux-points séparent le nom du remote du chemin ; un chemin local sans préfixe de remote utilise le backend Local. Des chaînes de connexion telles que `:sftp,host=example.net:/data` peuvent instancier un backend sans nom de remote enregistré. Cette flexibilité augmente aussi le risque qu’une faute de frappe désigne un chemin local plutôt que distant ([Rclone documentation – remotes](https://rclone.org/docs/)).

Avec les commandes de répertoires, Rclone transfère le **contenu** du chemin source, et non automatiquement le nom du répertoire source. La sémantique correspond au slash final familier de `rsync`. La source, la cible et l’arborescence cible attendue sont donc consignées dans le runbook sous forme d’exemples concrets ([rclone copy](https://rclone.org/commands/rclone_copy/)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für rclone-Inventar und Konfigurationspfad">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">rclone version
rclone config file
rclone listremotes --long
rclone backend features archive:
rclone about archive:</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">rclone version
rclone config file
rclone listremotes --long
rclone backend features archive:
rclone about archive:</code></pre>
  </div>
</div>

[`rclone version`](https://rclone.org/commands/rclone_version/) identifie le binaire, le système d’exploitation et le build ; [`rclone config file`](https://rclone.org/commands/rclone_config_file/) indique la configuration effectivement chargée ; [`rclone listremotes`](https://rclone.org/commands/rclone_listremotes/) inventorie les remotes ; [`rclone backend features`](https://rclone.org/commands/rclone_backend/) fournit les capacités du backend résolu ; [`rclone about`](https://rclone.org/commands/rclone_about/) interroge les informations de capacité, lorsque le service les fournit.

## Capacités des backends et différences apparentes

L’interface commune ne peut utiliser que les capacités fournies par le backend ou un wrapper. Les différences importantes sont :

| Capacité | Signification | Effet en cas d’absence ou de divergence |
|---|---|---|
| hash commun | empreinte de contenu côté fournisseur | la comparaison retombe sur taille/ModTime ou téléchargement |
| ModTime inscriptible | l’heure source peut être définie sans réupload | un contenu identique peut être transféré à nouveau |
| Copy/Move côté serveur | les données restent chez le fournisseur | sinon téléchargement/upload ou Copy plus Delete |
| ListR/Fast List | listing récursif avec moins d’appels | davantage de requêtes API ou de mémoire client |
| Change Polling | signale les modifications externes | VFS ne voit les changements qu’après expiration du cache |
| répertoires vides | véritable objet répertoire/marqueur | les dossiers vides disparaissent dans les modèles de buckets |
| sensibilité à la casse | `A` et `a` peuvent être distingués | synchronisation infinie ou collision de noms à la cible |
| noms dupliqués | plusieurs objets portant le même nom | le chemin n’est pas un identifiant unique |
| Metadata/MIME | attributs supplémentaires lisibles/inscriptibles | les ACL ou métadonnées métier ne suivent pas automatiquement |

Les niveaux de support évaluent les backends selon les tests et les capacités, et non selon une promesse fonctionnelle générale ([Backend Support Tiers](https://rclone.org/tiers/)). La documentation du fournisseur et du backend fait donc partie de la définition du job. Un service compatible S3 peut avoir des limites différentes pour les multipart, les hashes ou les copies côté serveur qu’un autre fournisseur S3.

## Configuration, identifiants et ordre de résolution

`rclone.conf` stocke les définitions de remotes et souvent des tokens OAuth ou identifiants. Les champs de mot de passe « obscured » ne constituent pas un chiffrement sûr. Toute la configuration peut être chiffrée avec un mot de passe ; pour les jobs non surveillés, une commande de mot de passe externe peut fournir le secret depuis un coffre de secrets ([Configuration encryption](https://rclone.org/docs/#configuration-encryption), [rclone config encryption](https://rclone.org/commands/rclone_config_encryption/)).

Rclone peut obtenir les options de backend depuis la ligne de commande, des variables d’environnement spécifiques au remote, des variables globales du backend et le fichier de configuration. L’ordre de priorité doit être documenté pour les services et conteneurs, car une variable héritée peut remplacer le `rclone.conf` attendu ([Environment variables](https://rclone.org/docs/#environment-variables)). Les journaux de débogage indiquent les options résolues, mais peuvent contenir des endpoints ou chemins sensibles.

Les administrateurs protègent au minimum :

- `rclone.conf` et ses droits de fichier ;
- le mot de passe de configuration ou la Password Command ;
- les refresh tokens OAuth, Client Secrets, Access Keys et clés privées SSH ;
- le mot de passe Crypt et le salt facultatif ;
- les autorisations de compte de service, bucket et dossier ;
- les filtres et définitions de jobs dans le cadre du contrôle des changements.

Une sauvegarde de configuration sans le secret de déchiffrement n’est pas une récupération testée. Inversement, une configuration chiffrée avec un mot de passe faible ne doit pas être déposée publiquement.

Une fois le remote, l’espace de noms et les identifiants clarifiés, la commande choisie détermine l’effet sur les données. En particulier, `sync` n’est pas une copie inoffensive, mais impose un côté comme vérité souhaitée.

## Copy, Sync et Move : la direction est une politique

[`rclone copy`](https://rclone.org/commands/rclone_copy/) copie les fichiers nouveaux ou différents vers la cible et n’y supprime aucun objet supplémentaire. [`rclone sync`](https://rclone.org/commands/rclone_sync/) fait de la cible l’image de la source et peut pour cela supprimer des objets de destination. [`rclone move`](https://rclone.org/commands/rclone_move/) copie ou déplace, puis ne supprime la source qu’après une opération de fichier réussie.

Ces commandes ne sont pas interchangeables :

| Opération | Écrasement à la cible | Supprime les objets supplémentaires de la cible | Supprime la source | Utilisation typique |
|---|---:|---:|---:|---|
| `copy` | pour les objets évalués comme différents | non | non | import, seed, réplication additive |
| `sync` | oui | oui | non | miroir unidirectionnel autoritatif |
| `move` | oui | non | après transfert réussi | ingestion, transfert vers archive |
| `copyto`/`moveto` | objet unique avec nom de cible explicite | non | uniquement `moveto` | opération atomique sur un fichier unique |
| `bisync` | des deux côtés | selon le delta déterminé | possible des deux côtés | réplication bidirectionnelle contrôlée |

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für sicheren rclone-Probelauf">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">rclone sync 'D:\Export' 'archive:current' `
  --dry-run --check-first --backup-dir 'archive:history/prechange' `
  --combined '.\sync-plan.txt' --log-level INFO --log-file '.\sync-dry-run.log'
if ($LASTEXITCODE -ne 0) { throw "rclone dry-run failed: $LASTEXITCODE" }
Get-Content '.\sync-plan.txt' | Group-Object { $_[0] } | Select-Object Name,Count</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">rclone sync /srv/export archive:current \
  --dry-run --check-first --backup-dir archive:history/prechange \
  --combined ./sync-plan.txt --log-level INFO --log-file ./sync-dry-run.log
test "$?" -eq 0 || exit "$?"
cut -c1 ./sync-plan.txt | sort | uniq -c</code></pre>
  </div>
</div>

`--dry-run` empêche les modifications, mais ne prouve que les listings et options visibles au moment du test. `--check-first` sépare la phase de comparaison de la phase de transfert. `--backup-dir` déplace les objets de destination écrasés ou supprimés vers un chemin distinct, si le backend prend en charge l’opération côté serveur nécessaire ([Global options – backup-dir and retries](https://rclone.org/docs/)). Le rapport Combined classe les chemins identiques, manquants, différents et erronés.

## Que signifie « identique » ?

Par défaut, Rclone ignore les fichiers selon la taille et l’heure de modification, ou utilise un hash commun lorsque la commande et le backend le permettent. L’empreinte exacte dépend des capacités et des flags. `--checksum`, `--size-only`, `--ignore-times`, `--update`, `--ignore-size`, les options de métadonnées et la précision de ModTime modifient la décision ([Overview – Hash and ModTime](https://rclone.org/overview/)).

Un ETag côté fournisseur n’est pas universellement un MD5 du contenu. Les uploads multipart, le chiffrement ou l’implémentation du fournisseur peuvent modifier sa signification. Rclone utilise la capacité de hash explicitement déclarée par le backend, et non une hypothèse générale concernant les ETags.

Entre Linux sensible à la casse et Windows généralement insensible à la casse, `report.pdf` et `Report.pdf` ne peuvent pas être reproduits sans perte. Les lecteurs cloud peuvent autoriser les noms dupliqués ou leurs propres ID d’objet. Un contrôle préalable des collisions de noms et des caractères non autorisés fait donc partie de chaque migration.

## Moteur de transfert, parallélisme et Pacer

Le moteur de transfert liste la source et la cible, compare les entrées, planifie les vérifications et transferts, puis exécute les opérations de backend en parallèle. `--checkers` contrôle les vérifications parallèles, `--transfers` les transferts. `--fast-list` peut réduire les appels API, mais requiert davantage de mémoire parce que les grands listings sont collectés ([Rclone documentation – performance options](https://rclone.org/docs/)).

Le parallélisme n’est pas une simple option de vitesse. Davantage de workers augmentent les fichiers ouverts, la mémoire, les requêtes API, les états multipart et la pression sur le fournisseur. Avec de petits objets, la latence de listing et de requêtes domine ; avec de grands objets, ce sont la bande passante et la configuration multipart. Les copies cloud à cloud peuvent rester côté serveur ou transiter par l’hôte rclone ; cela est vérifié dans le journal et les fonctionnalités du backend.

Les backends implémentent un pacing pour les limites de débit API. Les retries de bas niveau traitent les requêtes individuelles ; les retries de haut niveau répètent une opération globale échouée. Seul l’état final décide si les erreurs précédentes ont été compensées avec succès ([Retries](https://rclone.org/docs/#retries)). Un ordonnanceur évalue le code de sortie du processus, et non la présence du mot `ERROR` quelque part dans le journal.

## Les filtres définissent l’ensemble de données

Les règles Include, Exclude, Size, Age et Files-from déterminent les chemins visibles. Les filtres se rapportent à la racine indiquée ; un glob shell dans le chemin source ne remplace pas une règle de filtre rclone. L’ordre de Include et Exclude peut être pertinent ([Rclone Filtering](https://rclone.org/filtering/)).

Si un filtre change, l’ensemble de données métier change. Avec `sync`, des objets jusque-là visibles peuvent apparaître comme supprimés ; `--delete-excluded` est particulièrement dangereux. Les fichiers de filtres sont versionnés, hachés et validés avec le rapport de dry-run et la définition du job. Un test sûr utilise `rclone ls` ou `lsf` avec les mêmes règles avant toute commande en écriture.

Un code de sortie sans erreur de transfert prouve uniquement l’exécution, et non l’égalité complète des deux ensembles. `check` compare sources et cibles en fonction des capacités de chaque backend.

## Vérification : Check plutôt que compteur de transfert

[`rclone check`](https://rclone.org/commands/rclone_check/) ne modifie aucune donnée. Il compare la taille et les hashes communs, peut produire des rapports pour les correspondances, différences et chemins manquants, et lire les contenus avec `--download` lorsqu’aucun hash remote commun n’existe. Pour Crypt, il existe [`rclone cryptcheck`](https://rclone.org/commands/rclone_cryptcheck/), qui compare de façon appropriée une source en clair et une cible chiffrée.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für rclone-Integritätsprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">rclone check 'D:\Export' 'archive:current' `
  --combined '.\check-report.txt' --checkers 8 `
  --log-level INFO --log-file '.\check.log'
$code = $LASTEXITCODE
Get-Content '.\check-report.txt' | Group-Object { $_[0] } | Select-Object Name,Count
if ($code -ne 0) { throw "Integrity check failed: $code" }</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">rclone check /srv/export archive:current \
  --combined ./check-report.txt --checkers 8 \
  --log-level INFO --log-file ./check.log
code=$?
cut -c1 ./check-report.txt | sort | uniq -c
test "$code" -eq 0 || exit "$code"</code></pre>
  </div>
</div>

Une comparaison de hash prouve l’égalité pour l’algorithme et l’espace de noms utilisés. Elle ne prouve pas automatiquement les ACL, la rétention, les versions d’objets du fournisseur, les métadonnées de répertoires ou la lisibilité métier. Pour les archives critiques, un test de restauration complète le contrôle technique.

## Crypt : chiffrement des données et des noms

Crypt est un backend wrapper au-dessus d’un autre remote. Le contenu des fichiers est chiffré avec NaCl SecretBox basé sur XSalsa20 et Poly1305 ; les noms de fichiers et de répertoires sont également transformés par défaut ([Crypt backend](https://rclone.org/crypt/)). Le fournisseur voit le texte chiffré, les motifs de taille, les heures d’accès et le nombre d’objets ; Crypt protège donc le contenu et les noms, mais n’assure pas un anonymat complet du trafic ou des métadonnées.

Le mot de passe et le salt facultatif ne sont stockés que sous forme « obscured » dans la configuration, sauf si toute la configuration est chiffrée. La perte des deux informations de récupération rend les données illisibles. Avant une rotation ou une modification du chiffrement des noms, un second remote en lecture est testé sur les mêmes données.

Crypt modifie les tailles et possibilités de hash. Un `check` normal entre texte clair et texte chiffré ne peut donc pas simplement attendre le même hash du fournisseur. `cryptcheck` ou un chemin de téléchargement contrôlé constitue le bon niveau de vérification.

Les commandes de copie travaillent explicitement sur des objets. Un montage VFS doit en revanche reproduire des opérations de système de fichiers en cours et nécessite à cette fin un cache, un mode d’écriture et des hypothèses de cohérence claires.

## Montage VFS : le stockage d’objets n’est pas un système de fichiers POSIX

[`rclone mount`](https://rclone.org/commands/rclone_mount/) présente un remote comme système de fichiers via FUSE ou WinFsp. La couche VFS traduit les opérations sur les fichiers en opérations sur les objets. Comme les objets cloud ne peuvent généralement pas être modifiés au milieu ni étendus arbitrairement, Rclone met les objets en mémoire tampon ou les remplace.

Quatre modes de cache représentent des contrats différents :

| Mode | Cache local de fichiers | Comportement d’écriture | Limite opérationnelle |
|---|---|---|---|
| `off` | aucun | direct et limité | pas de Seek/Read-Write comme pour un fichier normal ; upload échoué plus difficile à répéter |
| `minimal` | uniquement pour les cas Read-Write nécessaires | partiellement mis en mémoire tampon | toutes les sémantiques Open/Seek ne sont pas possibles |
| `writes` | fichiers en écriture locaux | upload après Close/Writeback | le cache local n’est pas encore un commit remote |
| `full` | lectures et écritures locales, partiellement sparse | compatibilité élevée avec le système de fichiers | surveiller les besoins disque, l’éviction et la cohérence du cache |

Le Directory Cache conserve les noms et métadonnées en mémoire. Les modifications externes n’apparaissent qu’avec le polling ou après expiration ou flush. Le buffer est défini par fichier ouvert ; le besoin maximal de mémoire augmente avec `buffer-size × open files`. Deux processus ne doivent pas utiliser le même cache VFS qui se chevauche avec File Caching actif, car cela peut corrompre les données ([VFS file caching](https://rclone.org/commands/rclone_mount/#vfs-file-caching)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für kontrollierten rclone-Mount">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">rclone mount crypt-archive: X: `
  --vfs-cache-mode writes --cache-dir 'D:\rclone-cache\archive' `
  --vfs-cache-max-size 100G --vfs-cache-min-free-space 20G `
  --log-file 'D:\Logs\rclone-mount.log' --log-level INFO</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">rclone mount crypt-archive: /mnt/archive \
  --vfs-cache-mode writes --cache-dir /var/cache/rclone/archive \
  --vfs-cache-max-size 100G --vfs-cache-min-free-space 20G \
  --log-file /var/log/rclone-mount.log --log-level INFO</code></pre>
  </div>
</div>

Sous Windows, [WinFsp](https://winfsp.dev/doc/WinFsp-Tutorial/) fournit le pont de système de fichiers ; sous Linux et les autres systèmes Unix, Rclone utilise FUSE. Un service attend la disponibilité du réseau, de la configuration et du montage, possède un chemin de démontage contrôlé et surveille le volume de cache local. Les montages en conteneur nécessitent des droits supplémentaires de périphérique et de propagation de montage, et étendent la surface de privilèges ([Rclone mount in Docker](https://rclone.org/install/#docker)).

## Bisync : deux sources de vérité

[`rclone bisync`](https://rclone.org/bisync/) détecte les modifications des deux côtés à partir de listings enregistrés et transfère les deltas dans les deux directions. Cela crée un état de job persistant en dehors des données utilisateur. Un répertoire de travail perdu ou incohérent peut nécessiter une phase de resync.

Bisync possède des contrôles de sécurité pour l’accessibilité et un nombre inhabituellement élevé de suppressions. Les changements de filtres modifient l’ensemble de données visible et imposent consciemment un resync. Si les applications modifient le contenu sans changer la taille ni ModTime, la détection standard peut échouer. La synchronisation bidirectionnelle n’est donc utilisée que lorsque la politique de conflits, le comportement des horloges et timestamps, les contrôles d’accès, Max-Delete, la sauvegarde des fichiers de travail et la résolution humaine sont documentés.

Bisync n’est pas un système de fichiers multi-maître avec verrouillage distribué. Les modifications simultanées, renommages et phases hors ligne restent des conflits métier.

## API Remote Control, métriques et modes Serve

Avec `--rc` ou [`rclone rcd`](https://rclone.org/commands/rclone_rcd/), Rclone ouvre une API HTTP. Elle peut démarrer des jobs, configurer des remotes, lire ou écrire des fichiers locaux et, dans certaines circonstances, exécuter des commandes OS dans le contexte de l’utilisateur rclone. La documentation considère donc l’accès à RC comme équivalent à un accès shell ([Remote Control / API](https://rclone.org/rc/)).

RC écoute localement par défaut. En cas de publication réseau, une authentification forte, TLS ou un reverse proxy protégé, une liste d’autorisation de pare-feu et des privilèges de processus minimaux sont nécessaires. `--rc-no-auth` n’a pas sa place sur une interface accessible. L’endpoint OpenMetrics peut exposer des métriques de transfert et de job ; les noms de remotes et chemins peuvent aussi constituer des métadonnées sensibles.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für rclone-RC-Status und Metriken">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">rclone rc core/version
rclone rc core/stats
$metrics = Invoke-WebRequest 'http://127.0.0.1:5572/metrics'
$metrics.StatusCode
$metrics.Content | Select-String 'rclone_' | Select-Object -First 20</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">rclone rc core/version
rclone rc core/stats
curl --silent --show-error --fail http://127.0.0.1:5572/metrics | grep '^rclone_' | head -n 20</code></pre>
  </div>
</div>

[`Invoke-WebRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest) et [`curl`](https://curl.se/docs/manpage.html) vérifient l’endpoint HTTP local. [`rclone rc`](https://rclone.org/commands/rclone_rc/) utilise la même API et reprend les paramètres d’authentification de la configuration du job.

Rclone peut également proposer des remotes via les modes Serve HTTP, WebDAV, SFTP, FTP, NFS, DLNA ou compatibles S3. Le rôle du système passe alors de client à serveur. L’adresse d’écoute, l’authentification, la sécurité du protocole, le cache VFS et les droits des données sont alors durcis comme pour un service autonome ([Rclone serve](https://rclone.org/commands/rclone_serve/)).

## Journalisation, statistiques et codes de sortie

Les journaux distinguent DEBUG, INFO, NOTICE et ERROR. La journalisation JSON est disponible pour le traitement automatisé. Les statistiques affichent les checks, transferts, erreurs, octets, vitesse et ETA. Un compteur `Transferred` de courte durée n’est pas une preuve d’audit durable ; un job enregistre l’heure de début, les racines source/cible, le hash des filtres, le build Rclone, les options résolues non sensibles, les statistiques finales et le code de sortie.

Les codes de sortie documentés distinguent le succès, les erreurs de syntaxe, les chemins manquants, les erreurs temporaires, moins graves et fatales, ainsi que les limites de transfert ou de temps atteintes ([List of exit codes](https://rclone.org/docs/#list-of-exit-codes)). `--error-on-no-transfer` peut signaler séparément une exécution réussie sans transfert. Les ordonnanceurs et la supervision traitent les codes selon le type de job ; un job incrémentiel vide attendu s’évalue différemment d’un export quotidien absent.

Un retry de haut niveau peut corriger avec succès les premiers messages d’erreur. Inversement, un processus peut transférer beaucoup puis se terminer non zéro en raison de quelques objets non résolus. Les alertes portent sur l’état final, le nombre d’erreurs, l’épuisement des retries, la durée du job, l’absence de heartbeat et le résultat de vérification.

## Sécurité du chemin de données

Rclone hérite des droits de son compte de processus et des identifiants du remote. Un `sync` avec droits de suppression est un agent de suppression hautement privilégié. Le moindre privilège sépare la source en lecture, la cible en écriture, les droits de versioning/rétention et les actions administratives de bucket.

La vérification des certificats TLS reste active. `--no-check-certificate` désactive la vérification de la chaîne et du nom d’hôte et, selon la documentation, est réservé aux tests ([TLS options](https://rclone.org/docs/#no-check-certificate)). Les backends SFTP vérifient les clés hôtes au moyen d’une politique Known Hosts ou d’empreintes maintenue ; les endpoints API ne sont pas redirigés via des proxys non contrôlés.

Les filtres, `--backup-dir`, le versioning des objets et Object Lock limitent des risques différents. Un répertoire de sauvegarde dans le même compte ne protège pas contre une compromission du compte ou une suppression par le fournisseur. [Storage](/kb/storage), [Chiffrement](/kb/verschluesselung) et [Durcissement](/kb/haertung) traitent des surfaces de contrôle sous-jacentes.

Le diagnostic sépare le chemin local, le filtre Rclone, l’API backend, l’authentification, la limite de débit et le chemin réseau. C’est la seule façon de savoir si un fichier n’a jamais été sélectionné, n’a pas été transféré ou a été présenté différemment par la cible.

## Diagnostic réseau et backend

Un message d’erreur rclone peut provenir de la résolution de noms, de TCP, TLS, du proxy, de l’authentification du fournisseur, d’une limite de débit API, de la traduction du backend, du système de fichiers local ou de la logique de comparaison. Le diagnostic suit le chemin :

1. Confirmer le binaire, le chemin de configuration et le nom de remote résolu.
2. Vérifier le DNS, le port et TLS de l’endpoint concret.
3. Utiliser `rclone lsd` ou `lsjson` comme test minimal de backend en lecture.
4. Capturer la phase de requête/retry et les capacités du backend avec `-vv`, sans publier de secrets.
5. Vérifier la source/cible, les filtres et la décision d’empreinte.
6. Corréler l’état final, le code de sortie et le journal d’audit du fournisseur.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für rclone-Netzwerk- und Backenddiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName $env:RCLONE_ENDPOINT
Test-NetConnection $env:RCLONE_ENDPOINT -Port 443
rclone lsd archive: -vv --dump headers --log-file '.\rclone-debug.log'
rclone lsjson archive:health --max-depth 1 --metadata</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig +short "$RCLONE_ENDPOINT"
nc -vz "$RCLONE_ENDPOINT" 443
openssl s_client -connect "$RCLONE_ENDPOINT:443" -servername "$RCLONE_ENDPOINT" -brief &lt;/dev/null
rclone lsd archive: -vv --dump headers --log-file ./rclone-debug.log
rclone lsjson archive:health --max-depth 1 --metadata</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname), [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection), [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility), [`nc`](https://man.openbsd.org/nc), [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/), [`rclone lsd`](https://rclone.org/commands/rclone_lsd/) et [`rclone lsjson`](https://rclone.org/commands/rclone_lsjson/) séparent le transport, l’authentification, le listing et la visibilité des objets. Les dumps d’en-têtes sont contrôlés avant transmission afin d’y détecter tokens, signatures, cookies et chemins. [DNS](/kb/dns), [TCP](/kb/tcp), [TLS](/kb/tls) et [Troubleshooting](/kb/troubleshooting) approfondissent les différentes phases.

## Problèmes typiques

Rclone signale souvent l’étape à laquelle un job échoue visiblement, et non automatiquement la cause initiale. Le tableau associe donc symptôme, déclencheur probable et preuve qui confirme ou infirme l’hypothèse.

| Symptôme | Cause probable | Preuve | Fausse solution rapide |
|---|---|---|---|
| Les fichiers sont recopiés à chaque exécution | ModTime/hash non communs ou précision différente | fonctionnalités du backend, `-vv`, rapport Check | définir aveuglément `--size-only` |
| Les objets de destination disparaissent | direction de Sync, filtre ou option Delete | rapport Dry-Run/Combined, définition du job | redémarrer simplement le job |
| Le montage affiche d’anciens noms | Directory Cache ou absence de polling | statistiques VFS, RC `vfs/forget`, listing du fournisseur | arrêter brutalement le processus |
| L’écriture sur le montage manque sur le remote | fichier ouvert, Writeback en attente, cache plein | cache VFS et journal | supprimer le cache local |
| Vérification de hash impossible | aucun hash commun, Crypt ou sémantique multipart | aperçu des backends, `hashsum`, `check --download` | supposer qu’un ETag est un MD5 |
| RC répond de l’extérieur | adresse d’écoute trop large/authentification absente | liste de sockets, pare-feu, flags RC | garder seulement l’URL secrète |
| Bisync s’interrompt après un changement de filtre | l’état de travail protège contre des suppressions erronées | journal Bisync et hash du filtre | supprimer les fichiers de travail et démarrer normalement |
| Code de sortie non nul malgré de nombreux fichiers transférés | erreurs résiduelles après les retries | journal final, rapport d’erreur | considérer le compteur de transfert comme un succès |

## Sauvegarde, récupération et réconciliation

Rclone fait souvent partie d’une sauvegarde, mais n’est pas lui-même un modèle de sauvegarde immuable. `sync` réplique les suppressions et les dommages dus au chiffrement. `copy` conserve les objets supplémentaires de la cible, mais peut écraser des versions existantes. La protection résulte du versioning de destination, de la rétention/Object Lock, d’identifiants séparés, de `--backup-dir`, de snapshots immuables et de tests de restauration.

L’ensemble de récupération comprend :

- un binaire rclone approuvé ou un artefact de release vérifiable ;
- `rclone.conf` chiffré et secret de déchiffrement séparé ;
- les identifiants du fournisseur et leur procédure de récupération ;
- le mot de passe Crypt et le salt ;
- les définitions de job, de filtre et d’environnement ;
- les fichiers de travail Bisync ;
- les objets de cache VFS non téléversés et leur correspondance ;
- les rapports Dry-Run, Combined, Check et de codes de sortie ;
- les procédures de versioning, rétention et restauration de la cible.

Une restauration est effectuée vers un chemin séparé, puis vérifiée avec le hash, le nombre, la taille, un échantillon de métadonnées et un test applicatif. La réconciliation compare la source de vérité, les versions de destination, le writeback VFS et l’état de l’ordonnanceur. [Backup et Disaster Recovery](/kb/backup-dr) décrit de manière transversale le RPO/RTO et la réception.

## Histoire technique

Nick Craig-Wood a commencé le projet avec un premier commit en novembre 2012. Le fondateur a documenté cette origine à l’occasion du quatorzième anniversaire et a lié l’état initial des sources ([Happy 14th Birthday rclone](https://forum.rclone.org/t/happy-14th-birthday-rclone/53005), [initial commit](https://github.com/rclone/rclone/commit/e9ae4f89a44b1cac8e83ede1572f15dace692c67)).

Rclone a repris la syntaxe de direction familière de `rsync`, mais a remplacé ses hypothèses de delta par blocs par des opérations d’objets et de fournisseurs. Les stockages cloud n’offrent généralement pas d’interface côté serveur pour les sommes de contrôle glissantes rsync ; Rclone compare des objets entiers au moyen des empreintes disponibles et les transfère en conséquence.

Avec le nombre croissant de backends, le modèle central d’interfaces Fs/Object et ses fonctionnalités optionnelles se sont développés. Crypt a ajouté le chiffrement côté client ; Mount et VFS ont rendu le stockage d’objets accessible aux applications basées sur des fichiers ; l’API Remote Control a été fortement étendue en 2018 ; Bisync a ajouté une comparaison bidirectionnelle persistante. Le [Changelog](https://rclone.org/changelog/) documente cette évolution ainsi que les corrections pertinentes pour la sécurité. Il reste donc, pour l’exploitation, le complément temporel de cet article d’architecture.

La ligne technique reste cohérente : un binaire Go portable, de nombreux backends intégrés, des commandes communes et des différences de fournisseurs explicitement visibles. Ce sont précisément ces différences qui rendent Rclone puissant pour les administrateurs tout en exigeant une révision.

## Checklist administrateur

Avant un job de production, au minimum les points suivants sont documentés :

- L’origine du binaire, le chemin de configuration et la résolution du remote sont univoques.
- La source, la cible, la direction et l’arborescence cible attendue figurent dans le runbook.
- Les capacités du backend pour Hash, ModTime, Case, Metadata, Copy/Move et Listing ont été vérifiées.
- Le fichier de filtre et les paramètres du job sont versionnés ; le Dry Run et le rapport Combined ont été approuvés.
- Les politiques Delete, Max-Delete, Backup-Dir, versioning et rétention sont cohérentes.
- Les identifiants ne possèdent que les droits nécessaires ; la configuration, le mot de passe Crypt et le salt sont récupérables.
- Les transferts, checkers, Pacer et la bande passante respectent l’hôte et le fournisseur.
- Le cache VFS dispose de son propre volume, d’une alarme de capacité, d’un arrêt ordonné et d’une vérification du writeback.
- RC/Serve n’écoute que de manière contrôlée et est protégé comme un accès shell.
- Le succès est prouvé par l’état final, le code de sortie, le rapport Check et le test de restauration.

## Sources

- [Rclone](https://rclone.org/)
- [Overview of cloud storage systems](https://rclone.org/overview/)
- [rclone source repository](https://github.com/rclone/rclone)
- [Install from source](https://github.com/rclone/rclone/blob/master/docs/content/install.md)
- [Rclone documentation – remotes](https://rclone.org/docs/)
- [rclone copy](https://rclone.org/commands/rclone_copy/)
- [rclone version](https://rclone.org/commands/rclone_version/)
- [rclone config file](https://rclone.org/commands/rclone_config_file/)
- [rclone listremotes](https://rclone.org/commands/rclone_listremotes/)
- [rclone backend features](https://rclone.org/commands/rclone_backend/)
- [rclone about](https://rclone.org/commands/rclone_about/)
- [Backend Support Tiers](https://rclone.org/tiers/)
- [rclone config encryption](https://rclone.org/commands/rclone_config_encryption/)
- [rclone sync](https://rclone.org/commands/rclone_sync/)
- [rclone move](https://rclone.org/commands/rclone_move/)
- [Rclone Filtering](https://rclone.org/filtering/)
- [rclone check](https://rclone.org/commands/rclone_check/)
- [rclone cryptcheck](https://rclone.org/commands/rclone_cryptcheck/)
- [Crypt backend](https://rclone.org/crypt/)
- [rclone mount](https://rclone.org/commands/rclone_mount/)
- [WinFsp documentation](https://winfsp.dev/doc/WinFsp-Tutorial/)
- [Rclone mount in Docker](https://rclone.org/install/)
- [rclone bisync](https://rclone.org/bisync/)
- [rclone rcd](https://rclone.org/commands/rclone_rcd/)
- [Remote Control / API](https://rclone.org/rc/)
- [Invoke-WebRequest](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest)
- [curl manual](https://curl.se/docs/manpage.html)
- [rclone rc](https://rclone.org/commands/rclone_rc/)
- [Rclone serve](https://rclone.org/commands/rclone_serve/)
- [Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [dig manual](https://bind9.readthedocs.io/en/latest/manpages.html)
- [nc(1)](https://man.openbsd.org/nc)
- [openssl-s_client(1)](https://docs.openssl.org/master/man1/openssl-s_client/)
- [rclone lsd](https://rclone.org/commands/rclone_lsd/)
- [rclone lsjson](https://rclone.org/commands/rclone_lsjson/)
- [Happy 14th Birthday rclone](https://forum.rclone.org/t/happy-14th-birthday-rclone/53005)
- [Initial rclone commit](https://github.com/rclone/rclone/commit/e9ae4f89a44b1cac8e83ede1572f15dace692c67)
- [Rclone Changelog](https://rclone.org/changelog/)
