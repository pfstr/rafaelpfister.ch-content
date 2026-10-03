---
title: "Sauvegarde et reprise après sinistre : états, objectifs et redémarrage"
blatt: "backup-dr"
description: "Sauvegarde et reprise après sinistre pour les administrateurs de messagerie : distinction avec les snapshots, la réplication et la haute disponibilité, RPO, RTO et MTD, sauvegarde cohérente des boîtes aux lettres, files d’attente, configurations, clés et index, reprise isolée, ordre de redémarrage, tests de restauration et diagnostic."
fakten:
  - label: Objectif
    wert: ramener les services et les données à un état connu et fiable
    href: https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf
  - label: Base de planification
    wert: analyse d’impact métier et inventaire des dépendances
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final
  - label: MTD
    wert: interruption maximale tolérable du processus métier
    href: https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf
  - label: RTO
    wert: indisponibilité maximale d’une ressource avant un impact inacceptable
    href: https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf
  - label: RPO
    wert: point dans le temps jusqu’auquel les données doivent être restaurées après l’événement
    href: https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf
  - label: Sauvegarde
    wert: copie horodatée et restaurable ; complète la réplication
    href: https://learn.microsoft.com/en-us/azure/reliability/concept-redundancy-replication-backup
  - label: Cohérence
    wert: l’application, le logiciel de sauvegarde et le stockage doivent coordonner le point de sauvegarde
    href: https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service
  - label: Couches de données
    wert: boîte aux lettres/blob · métadonnées · file d’attente · index · configuration · clés
    href: https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf
  - label: Cyber-reprise
    wert: copie isolée et restauration vers un état fiable
    href: https://cas8.docs.cisecurity.org/en/latest/source/Controls11/
  - label: Immutabilité
    wert: la rétention empêche la suppression ou l’écrasement de certaines versions d’objets
    href: https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html
  - label: Règle relative aux clés
    wert: planifier la reprise selon le type de clé et son usage
    href: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r5.pdf
  - label: Critère d’acceptation
    wert: restauration réussie et effectuée à temps d’un service métier
    href: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-184.pdf
werbung:
  - tools
  - newsletter
ctaThemen:
  - backup
  - disaster-recovery
  - messaging
translationSourceHash: dec2b9d3915258979e801cb9d3ea573f054ae89136cd0631af06b681f36c3ca9
translationModel: gpt-5.6-terra
translatedAt: 2026-10-03T09:36:07.908Z
translationReview: required
---

# Sauvegarde et reprise après sinistre : états, objectifs et redémarrage

Une tâche de sauvegarde terminée avec succès prouve d’abord uniquement qu’un outil a écrit des données. Elle ne dit pas encore si le point de sauvegarde est complet, cohérent au niveau de l’application, protégé contre la même panne et s’il permet de rétablir un service exploitable dans le délai convenu. La **sauvegarde** désigne la copie restaurable d’un état antérieur ; la **reprise après sinistre** englobe en outre les personnes, les priorités, l’infrastructure cible, les dépendances, la validation et le retour contrôlé en production. Le NIST distingue donc la sauvegarde, la stratégie de restauration, les procédures de reprise, les tests et la maintenance continue du plan ([NIST SP 800-34 Rev. 1](https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final)).

Pour les plateformes de messagerie, « la base de données » n’est pas un objet de protection complet. Un état exploitable peut être réparti entre le stockage des boîtes aux lettres ou des blobs, les métadonnées relationnelles, les files d’attente de transport, les index de recherche, les configurations, les règles de routage, les références d’annuaire, les certificats, les clés privées, le DNS, les licences et l’automatisation. Certains éléments font autorité, d’autres ne sont que des projections et d’autres encore sont des états transitoires. Le plan de reprise doit préciser pour chaque élément s’il est **restauré, reconstruit, réémis ou délibérément abandonné**. Outre les données utilisateur, le NIST exige l’état du système, les logiciels, l’inventaire, les licences et la documentation de sécurité ; PostgreSQL indique par exemple explicitement que l’archivage WAL ne sauvegarde pas ses fichiers de configuration ([NIST SP 800-34 Rev. 1, CP-9](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf), [PostgreSQL: archivage continu et PITR](https://www.postgresql.org/docs/17/continuous-archiving.html)).

Le critère d’acceptation opérationnel n’est donc pas « sauvegarde montée », mais par exemple : un expéditeur externe peut remettre un message, la bonne politique est appliquée, le message apparaît dans la boîte aux lettres prévue, peut être recherché et recevoir une réponse, tandis que la supervision et l’audit enregistrent l’opération. Le NIST cite les restaurations réussies et réalisées à temps, les objectifs de reprise atteints et les utilisateurs ou systèmes de nouveau disponibles comme résultats de reprise mesurables ([NIST SP 800-184](https://www.nist.gov/publications/guide-cybersecurity-event-recovery), [NIST SP 800-184, métriques de reprise](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-184.pdf)).

L’explication commence par le processus métier qui doit fonctionner à nouveau après une panne. De là découlent le RTO et le RPO, puis la chaîne de sauvegarde appropriée ; le résultat final n’est pas la tâche de sauvegarde, mais un test de restauration mesuré.

## La sauvegarde n’est pas la haute disponibilité

La sauvegarde, le snapshot, la réplication et la haute disponibilité couvrent différentes catégories d’erreurs. Microsoft décrit explicitement la sauvegarde et la réplication comme complémentaires : la réplication maintient une copie à jour pour l’exploitation courante, mais reproduit également les suppressions logiques ou les corruptions ; une sauvegarde horodatée permet de revenir à un état antérieur ([Microsoft Azure Reliability: Redundancy, Replication and Backup](https://learn.microsoft.com/en-us/azure/reliability/concept-redundancy-replication-backup)).

| Mécanisme | Bénéfice principal | Ce qu’il ne prouve pas à lui seul |
|---|---|---|
| Sauvegarde | état historique, restaurable et conservé | temps de basculement court ou plateforme cible immédiatement opérationnelle |
| Snapshot de stockage | état rapide à un instant donné d’un volume | cohérence applicative, domaine de défaillance distinct ou conservation à long terme |
| Réplication | état actuel des données vers une seconde cible | protection contre une suppression, un chiffrement ou une corruption silencieuse répliqués |
| Haute disponibilité | continuité de service lors de défaillances de composants définis | retour historique en arrière ou reconstruction après compromission administrative |
| Archive | conservation de données sélectionnées pendant de longues périodes | reconstruction complète du service et de ses dépendances |
| Reprise après sinistre | redémarrage coordonné après un événement de site, de plateforme ou de sécurité | restauration des données sans sauvegardes adéquates et testées |

Un snapshot peut constituer un élément de la sauvegarde. Toutefois, il ne devient une source de reprise fiable qu’avec la cohérence, l’exportation ou la réplication vers un stockage exploité indépendamment, la rétention, un catalogue et une procédure de restauration. L’API de snapshots de Kubernetes, par exemple, ne garantit pas elle-même la cohérence applicative ; les applications doivent être préparées de manière appropriée avant le snapshot. Sous Windows, VSS n’assure cette coordination que si le demandeur, le writer et le provider interagissent correctement ([Kubernetes: snapshot de volume et cohérence applicative](https://kubernetes.io/blog/2020/12/10/kubernetes-1.20-volume-snapshot-moves-to-ga/), [Microsoft: Volume Shadow Copy Service](https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service)).

## MTD, RTO et RPO sont liés aux fonctions métier et système

La **durée d’interruption maximale tolérable (MTD)** est la plus longue interruption que le processus métier peut tolérer dans son ensemble. L’**objectif de temps de reprise (RTO)** décrit combien de temps une ressource système donnée peut être indisponible avant que d’autres ressources, le processus pris en charge ou sa MTD ne soient affectés de manière inacceptable. L’**objectif de point de reprise (RPO)** désigne le moment antérieur à l’événement jusqu’auquel les données doivent être restaurées. Le RTO doit généralement être inférieur à la MTD, car après le redémarrage technique, les données doivent encore être retraitées et le service validé sur le plan métier ([NIST SP 800-34 Rev. 1, section 3.2](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf)).

Un unique « RTO de messagerie » masque des différences importantes. Une plateforme peut accepter SMTP alors que l’accès utilisateur ou la recherche ne sont pas encore disponibles. Une passerelle peut mettre les messages en mémoire tampon alors que le service de boîtes aux lettres en aval est indisponible. À l’inverse, une interface web peut être accessible alors que les clés, les recherches dans l’annuaire ou les connecteurs sortants manquent. Les objectifs doivent donc être définis par **fonction métier et dépendance**.

| Fonction | État à mesurer | Question RPO typique | Le RTO ne prend fin que lorsque |
|---|---|---|---|
| réception externe | MX, TLS, écouteur SMTP, politique et file d’attente | Quels messages acceptés peuvent manquer ? | la réception est contrôlée et le traitement de la file d’attente est vérifiable |
| livraison sortante | routage, DNS, politique TLS, nouvelle tentative et DSN | Quelles entrées de file d’attente peuvent être perdues ? | la livraison réussit ou le retard est signalé conformément aux normes |
| accès aux boîtes aux lettres | identité, métadonnées, blob et protocole | Quel dernier état de boîte aux lettres est requis ? | la connexion ainsi que la lecture, l’écriture et les opérations sur les dossiers fonctionnent |
| recherche | index et projections | L’index doit-il être sauvegardé ou régénéré ? | le périmètre de données défini est de nouveau retrouvable |
| chiffrement | politique, certificats, clés et relation de confiance | Quels anciens contenus doivent rester déchiffrables ? | un message de test défini peut être chiffré et déchiffré |
| administration | plan de contrôle, rôles, audit et supervision | Quelle modification de configuration peut manquer ? | une modification autorisée, l’alerte et l’audit sont traçables |

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-backup-dr.svg?v=20260813" title="Interaktive Infografik: Backup- und Disaster-Recovery-Kette für Messaging-Plattformen von produktiven Zuständen über konsistente, isolierte Sicherungen bis zur geprüften Wiederherstellung" loading="lazy">
  <a href="/images/kb-interaktiv-backup-dr.svg?v=20260813">Ouvrir directement le graphique interactif</a>.
</iframe>

Une fois les objectifs définis, l’état réel de la plateforme doit être inventorié. Une base de données de boîtes aux lettres seule ne restaure ni le routage, ni les identités, ni les clés, ni les index de recherche.

## Inventaire des états d’une plateforme de messagerie

Une politique de sauvegarde commence par un inventaire des états et des dépendances, et non par le catalogue de produits du fournisseur de sauvegarde. Pour chaque état sont documentés la **source faisant autorité, le mécanisme de cohérence, le RPO, la rétention, le domaine de protection, la méthode de restauration, l’ordre et l’étape de vérification**. L’analyse d’impact métier du NIST identifie les processus critiques, les ressources et leur priorité de reprise ; NIST SP 800-184 ajoute des scénarios réalistes et les dépendances découvertes pendant la restauration ([NIST SP 800-34 Rev. 1](https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final), [NIST SP 800-184](https://www.nist.gov/publications/guide-cybersecurity-event-recovery)).

| État | Caractère | Décision de reprise | Étape de vérification métier |
|---|---|---|---|
| Boîtes aux lettres et blobs de messages | données utilisateur faisant autorité | restaurer de manière cohérente ou reconstruire à partir d’une source immuable | lire un message connu avec ses pièces jointes MIME |
| Base de métadonnées | transactions, associations, ACL, UID | utiliser une sauvegarde de base plus relecture des journaux ou une restauration propre à l’application | dossiers, droits et références de messages concordent |
| File d’attente de transport | état de livraison transitoire mais pertinent pour le métier | sauvegarder, reprendre de façon ordonnée ou remettre délibérément | aucune lacune silencieuse et gestion contrôlée des doublons |
| Index de recherche et projections | généralement dérivés et éventuellement cohérents | sauvegarder uniquement si la reconstruction viole le RTO ; sinon réindexer | l’échantillon défini est entièrement retrouvable |
| Configuration et politique | déclarative, exportée ou basée sur une base de données | sauvegarder un export versionné avec l’état du schéma/produit | routage, filtres, limites et séparation des locataires sont effectifs |
| Identités et références d’annuaire | souvent gérées par une source externe | restaurer l’annuaire séparément ; conserver les liaisons, ID et claims | connexion des services et des utilisateurs fonctionnelle |
| Certificats, clés et secrets | très sensibles, parfois non exportables | sauvegarder, réémettre ou reconstruire via HSM/KMS selon le type de clé | TLS, signature, déchiffrement et rotation vérifiés |
| DNS, temps, réseau et équilibreurs de charge | couche externe de pilotage et de nommage | documenter comme code/export et auprès du fournisseur | noms, ports, noms de certificats et heure corrects |
| Logiciels, images, IaC et licences | base d’exécution reproductible | conserver des artefacts fiables, versions et dépendances | une version identique ou compatible approuvée démarre |
| Journaux, audit, catalogue de sauvegarde et runbooks | preuve et pilotage | maintenir disponibles hors du domaine d’administration concerné | incident, point de restauration et validations traçables |

La reprise des files d’attente constitue un cas particulier. SMTP exige que la responsabilité acceptée soit exécutée de manière fiable, mais autorise, lors d’interruptions de connexion, des situations dans lesquelles l’expéditeur et le destinataire évaluent différemment l’achèvement. Un état de file d’attente restauré peut donc remettre des messages. Les runbooks doivent prévoir une stratégie de gestion des doublons, des ID de file d’attente, des fenêtres temporelles et une communication aux destinataires ; la simple copie d’un répertoire spool n’est pas une restauration conforme aux normes ([RFC 5321, mise en file d’attente et messages en double](https://datatracker.ietf.org/doc/html/rfc5321)).

## La cohérence naît au niveau de l’application

Un point de sauvegarde **cohérent après incident** contient l’état qu’un système verrait après une coupure d’alimentation brutale. Le système de fichiers et les blocs individuels peuvent être cohérents en eux-mêmes, tandis que les bases de données, blobs et files d’attente associés représentent des instants différents. Un point de sauvegarde **cohérent au niveau de l’application** coordonne les tampons d’écriture, les journaux de transactions, les checkpoints et, le cas échéant, plusieurs volumes afin que l’application dispose d’un chemin de reprise défini.

VSS illustre explicitement cette architecture : le demandeur de sauvegarde demande la sauvegarde, le writer spécifique à l’application fournit un jeu de données cohérent et le provider crée la Shadow Copy. Exchange propose pour cela son propre VSS Writer ; une sauvegarde consciente d’Exchange est donc plus qu’un snapshot de ses fichiers de base de données ([Microsoft: Volume Shadow Copy Service](https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service), [Microsoft: Windows Server Backup pour Exchange](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/windows-server-backup)).

PostgreSQL utilise un modèle de reprise différent mais comparable. Une sauvegarde de base fournit le point de départ, puis une suite ininterrompue de segments Write-Ahead Log archivés la prolonge jusqu’au moment souhaité. Un `pg_dump` est une exportation logique et ne remplace pas la chaîne de sauvegarde de base/WAL requise pour le PITR. Les fichiers de configuration tels que `postgresql.conf` et `pg_hba.conf` se trouvent également hors de cette reprise WAL et nécessitent une voie de sauvegarde distincte ([PostgreSQL: archivage continu et PITR](https://www.postgresql.org/docs/17/continuous-archiving.html)).

Pour les produits distribués, la documentation du produit doit indiquer si les backends sont sauvegardés indépendamment, en tant que groupe de cohérence ou via des fonctions d’exportation propres à l’application. Un snapshot de stockage simultané de plusieurs volumes ne constitue pas automatiquement une coupe cohérente entre la base de données, le stockage d’objets, la file d’attente et l’index de recherche. L’administrateur doit connaître la **source de vérité** et le chemin de reconstruction admissible pour chaque projection.

### Inventorier la capacité et les artefacts de sauvegarde

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kapazitäts- und Sicherungsinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-Volume | Sort-Object DriveLetter |
  Select-Object DriveLetter, FileSystemLabel, Size, SizeRemaining
Get-ChildItem \\backup.example.ch\mail -File -Recurse |
  Select-Object FullName, Length, LastWriteTime
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
df -hT
find /backup/mail -type f -printf '%TY-%Tm-%TdT%TH:%TM:%TS %s %p\n'
```

  </div>
</div>

[`Get-Volume`](https://learn.microsoft.com/powershell/module/storage/get-volume) et [`df`](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html) affichent la capacité occupée et disponible. [`Get-ChildItem`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-childitem) et [`find`](https://www.gnu.org/software/findutils/find) inventorient les artefacts et leurs horodatages. Aucun des deux ne prouve la cohérence applicative ni la restaurabilité ; pour cela, il faut des preuves issues du catalogue, des journaux et de la restauration.

## Architecture de protection : séparée, isolée et vérifiable

Une chaîne de sauvegarde fiable comporte au moins quatre rôles distincts :

1. **Capture :** l’application ou la fonction d’exportation produit un état défini.
2. **Catalogue et manifeste :** ID de sauvegarde, source, instant, version logicielle, journaux requis, références de clés et sommes de contrôle rendent le jeu retrouvable et vérifiable.
3. **Stockage de reprise :** des copies versionnées se trouvent hors du domaine de défaillance primaire et, si possible, hors du domaine d’administration primaire.
4. **Plan de contrôle de reprise :** identités séparées, runbooks, infrastructure cible et validations permettent la restauration lorsque la production n’est pas fiable.

CIS Control 11 exige des données de reprise protégées de manière équivalente, une instance isolée, par exemple hors ligne, dans le cloud ou hors site, ainsi que des tests de restauration réguliers. Pour les scénarios de ransomware, CISA recommande des sauvegardes hors ligne ou autrement isolées, chiffrées et régulièrement testées, ainsi que des images propres et un environnement de reprise séparé ([CIS Control 11: Data Recovery](https://cas8.docs.cisecurity.org/en/latest/source/Controls11/), [CISA: StopRansomware Guide](https://www.cisa.gov/stopransomware/ransomware-guide)).

L’**immutabilité** et l’**isolation** ne sont pas la même chose. S3 Object Lock peut protéger certaines versions d’objets contre la suppression et l’écrasement selon le modèle WORM durant une rétention ou un Legal Hold. Les modes Governance et Compliance offrent différentes possibilités de contournement. Cela protège les versions stockées, mais ne prouve ni l’existence d’identifiants séparés, ni un point de restauration propre, ni une chaîne applicative complète, ni l’accessibilité des clés de déchiffrement ([Amazon S3: Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)).

### Contrôler les sommes de contrôle et le manifeste

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Prüfsummen von Sicherungsartefakten">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-FileHash .\mail-backup-2026-08-08.tar.zst -Algorithm SHA256
Get-FileHash .\mail-backup-2026-08-08.manifest.json -Algorithm SHA256
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
sha256sum mail-backup-2026-08-08.tar.zst
sha256sum mail-backup-2026-08-08.manifest.json
```

  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) et [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) détectent les modifications d’un artefact lorsque le hachage attendu provient d’une source fiable. Une somme de contrôle ne remplace ni l’authentification du manifeste ni un essai de restauration. Le NIST cite les hachages cryptographiques et les signatures numériques comme mécanismes de protection de l’intégrité des informations de sauvegarde ([NIST SP 800-34 Rev. 1, CP-9](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf)).

## Les clés et secrets nécessitent leur propre plan de reprise

« Sauvegarder toutes les clés privées » est aussi erroné que « les certificats peuvent être réémis ». L’usage prévu est déterminant :

- Une **clé de serveur TLS** perdue peut généralement être remplacée par une nouvelle paire de clés et un nouveau certificat ; le basculement doit néanmoins respecter le RTO et les dépendances associées de confiance ou de pinning doivent être vérifiées.
- Une **clé de déchiffrement** des données S/MIME, OpenPGP ou de sauvegarde stockées doit rester disponible aussi longtemps que le texte chiffré protégé doit demeurer lisible.
- Selon le NIST, la sauvegarde d’une **clé privée de signature** n’est généralement pas souhaitable, car sa réutilisation peut affecter la valeur probante de la signature ; les exceptions justifiées exigent une reprise particulièrement sûre et un remplacement rapide.
- Une **clé HSM/KMS non exportable** nécessite le chemin de redondance, de sauvegarde ou de reprovisionnement prévu par le système. Exporter un certificat sans clé privée n’est pas une sauvegarde de clé.
- La **clé qui chiffre la sauvegarde** ne doit pas se trouver exclusivement dans la sauvegarde chiffrée ou dans le domaine de production compromis.

Le NIST exige une décision par type de clé, les métadonnées associées, une politique de récupération de clés ainsi que des contrôles de confidentialité, d’intégrité, de disponibilité et d’audit pour le matériel de reprise. Lorsqu’une clé de déchiffrement est perdue, le texte chiffré ne peut plus être ramené en clair ([NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final), [NIST SP 800-57 Part 1 Rev. 5, récupération de clés](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r5.pdf)).

### Vérifier l’heure et la résolution de noms avant la restauration

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Zeit- und DNS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
w32tm /query /status
Resolve-DnsName -Type MX example.ch
Resolve-DnsName backup.example.ch
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
timedatectl status
dig +short MX example.ch
dig +short A backup.example.ch
```

  </div>
</div>

[`w32tm`](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings) et [`timedatectl`](https://www.freedesktop.org/software/systemd/man/latest/timedatectl.html) contrôlent la base de temps pour les certificats, Kerberos, les journaux et les points de reprise. [`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) et [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) indiquent si les noms MX, de service et de dépôt sont résolus comme prévu dans la zone de reprise. La source [DNS](/kb/dns) et son autorisation de modification font elles-mêmes partie de l’inventaire des dépendances.

Une sauvegarde cohérente ne représente que la moitié du plan. Lors de la restauration, l’identité, le DNS, la base de données, la file d’attente, les clés et les applications doivent revenir dans un ordre justifié.

## Le redémarrage suit le graphe des dépendances

Un ordre fixe de produits serait inventé. L’ordre fiable découle de la BIA, de l’inventaire des ressources et des dépendances réelles. Le NIST exige une liste priorisée des ressources système et des scénarios de test réalistes ; NIST SP 800-184 impose de réintégrer à la documentation les dépendances nouvellement identifiées pendant la restauration ([NIST SP 800-34 Rev. 1, priorités de reprise](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf), [NIST SP 800-184, exécution de la reprise](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-184.pdf)). Pour une plateforme de messagerie typique, il en résulte souvent la chaîne suivante, à valider localement :

1. **Délimiter l’événement :** distinguer la panne de la compromission, préserver les preuves, définir un point de reprise connu comme propre et obtenir la validation.
2. **Établir le plan de contrôle de reprise :** fournir des identités d’administration séparées, MFA, les runbooks, le catalogue de sauvegarde et l’accès au déchiffrement.
3. **Valider les services de base :** vérifier dans la zone cible le réseau, le routage, [DNS](/kb/dns), le temps, [LDAP](/kb/ldap) ou [Kerberos](/kb/kerberos), la PKI/KMS et les équilibreurs de charge.
4. **Restaurer la persistance :** reconstruire dans une coupe cohérente le stockage d’objets/de boîtes aux lettres, les bases de données et les journaux de transactions requis.
5. **Démarrer l’application et la politique :** déployer les images approuvées, la configuration, les secrets, les connecteurs et les rôles ; ne pas encore autoriser un flux de messagerie externe non contrôlé.
6. **Activer de manière contrôlée la file d’attente et le routage :** évaluer l’ancienneté, les destinataires, l’état des nouvelles tentatives et les doublons possibles ; autoriser séparément l’entrée et la sortie.
7. **Recréer les projections :** générer les index de recherche, les caches et le reporting à partir des sources faisant autorité et surveiller le retard de reconstruction.
8. **Accepter la transaction métier :** tester la livraison, l’accès à la boîte aux lettres, la recherche, [TLS](/kb/tls), le chiffrement, la supervision et l’audit selon des critères définis.

Pour un incident de sécurité, « le système démarre » ne suffit explicitement pas. Le NIST décrit la reconstitution dans un état connu et sûr avec des paramètres sécurisés, des correctifs, une configuration, des logiciels fiables, une sauvegarde connue comme propre et des tests complets. CIS formule le même objectif comme une restauration vers un « pre-incident and trusted state » ([NIST SP 800-34 Rev. 1, CP-10](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf), [CIS Control 11: Data Recovery](https://cas8.docs.cisecurity.org/en/latest/source/Controls11/)).

### Atteindre les points de terminaison de reprise et TLS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Recovery-Endpunkt- und TLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection backup.example.ch -Port 443 -InformationLevel Detailed
curl.exe --verbose https://backup.example.ch/health
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz backup.example.ch 443
openssl s_client -connect backup.example.ch:443 \
  -servername backup.example.ch -verify_return_error
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) et [`nc`](https://man.openbsd.org/nc) vérifient le chemin TCP. [`curl`](https://curl.se/docs/manpage.html) et [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) montrent respectivement le comportement HTTP et TLS. Un point de terminaison de santé accessible ne prouve que le plan de contrôle, pas la lisibilité de tous les jeux de sauvegarde.

Le redémarrage n’est achevé que lorsqu’un utilisateur ou un correspondant peut effectivement utiliser le service. Un support de sauvegarde lu avec succès n’en constitue pas une preuve suffisante.

## Les tests de restauration mesurent le service, pas le support

Un test complet s’exécute dans un environnement cible isolé avec un point de départ documenté, une mesure du temps et des critères d’acceptation. Il vérifie au minimum :

- que le catalogue, les identifiants, les clés de déchiffrement et les artefacts sont accessibles sans la production ;
- qu’un système cible compatible peut être fourni à partir d’images fiables ;
- que la sauvegarde de base, les journaux de transactions, le stockage de blobs et la configuration aboutissent au même état métier ;
- que les files d’attente sont traitées de manière contrôlée et que les doublons sont détectés ;
- que l’identité, le DNS, TLS, le flux de messagerie, l’accès aux boîtes aux lettres, la recherche et la supervision fonctionnent ;
- que la perte de données mesurée et le temps de redémarrage mesuré respectent le RPO et le RTO ;
- que le service peut être approuvé comme fiable après des incidents de sécurité.

CIS Control 11.5 évalue un échantillon de sauvegardes restaurées puis effectivement fonctionnelles. NIST SP 800-184 mesure les restaurations réussies et réalisées à temps et exige des scénarios réalistes, un débriefing et l’amélioration du plan ([CIS Control 11: Test Data Recovery](https://cas8.docs.cisecurity.org/en/latest/source/Controls11/), [NIST SP 800-184](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-184.pdf)). La fréquence des tests dépend du risque, du rythme de changement et des exigences ; un test complet annuel peut être complété par des échantillons automatisés plus fréquents et des restaurations par composant, mais ne doit pas être remplacé par des statistiques de tâches réussies.

### Écouteurs et état du stockage après la restauration

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Listener- und Speicherprüfung nach Restore">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-NetTCPConnection -State Listen |
  Sort-Object LocalPort |
  Select-Object LocalAddress, LocalPort, OwningProcess
Get-Volume | Select-Object DriveLetter, FileSystemLabel, SizeRemaining
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
ss -lntup
df -hT
```

  </div>
</div>

[`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) et [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) affichent les écouteurs locaux et les processus associés. [`Get-Volume`](https://learn.microsoft.com/powershell/module/storage/get-volume) et [`df`](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html) affichent l’espace libre. La vérification doit ensuite se poursuivre au niveau du protocole : un écouteur sur le port 25 n’est pas encore une transaction [SMTP](/kb/smtp) fonctionnelle.

## Scénarios de panne et périmètre de reprise approprié

Une panne détermine l’ampleur nécessaire de la restauration. Le tableau associe donc l’événement observé au périmètre de reprise pertinent le plus restreint et à l’hypothèse erronée la plus fréquente.

| Événement | Risque principal | Périmètre de reprise approprié | Hypothèse erronée fréquente |
|---|---|---|---|
| suppression accidentelle | dommage logique limité | restaurer de manière ciblée l’objet, la boîte aux lettres, la politique ou un point dans le temps | restaurer toute la plateforme et perdre des données correctes plus récentes |
| nœud ou disque individuel | panne d’infrastructure locale | basculement HA, réplique ou restauration par composant | confondre le basculement avec une sauvegarde historique |
| perte d’un site ou d’un fournisseur | domaine de défaillance physique ou administrative commun | zone/région/site alternatif avec copies externes et basculement DNS/réseau | considérer une copie de données sans capacité cible accessible comme une reprise après sinistre |
| ransomware ou compromission d’administrateur | données, identités, logiciels et sauvegardes non fiables | plan de contrôle de reprise isolé, build propre, point de restauration connu | continuer à utiliser l’identité compromise pour déverrouiller toutes les sauvegardes |
| perte de clé | texte chiffré définitivement illisible ou identité inutilisable | reprise spécifique au type de clé, réémission ou procédure HSM/KMS | confondre certificat public et clé privée |
| modification de configuration erronée | données correctes, comportement incorrect | annuler une configuration versionnée et valider de manière ciblée | choisir en premier lieu une restauration de base de données ou de boîte aux lettres |

En cas de compromission, le point connu comme propre ne doit pas nécessairement être le point de sauvegarde le plus récent. Des sauvegardes plus récentes peuvent contenir l’état de l’attaquant ; des sauvegardes plus anciennes peuvent entraîner des vulnérabilités connues ou des versions logicielles incompatibles. La reprise associe donc l’investigation forensique, le niveau de correctifs, la baseline de configuration, la rotation des clés et la restauration des données métier. CISA recommande notamment des « golden images » propres, des définitions d’infrastructure conservées hors ligne et une zone réseau de reprise afin que les systèmes ne soient pas réinfectés pendant leur reconstruction ([CISA: StopRansomware Guide](https://www.cisa.gov/stopransomware/ransomware-guide)).

## Évolution technique

La bande magnétique a été introduite au début des années 1950 comme support de stockage rapide pour les ordinateurs et demeure un support de sauvegarde en raison de son coût, de sa capacité et de sa séparation physique ([IBM: Magnetic Tape](https://www.ibm.com/history/magnetic-tape)). Les architectures de sauvegarde ultérieures ont de plus en plus séparé le point de sauvegarde logique du support cible : les bases de données ont combiné sauvegardes de base, journaux de transactions et reprise à un instant donné ; les systèmes de stockage ont permis des snapshots rapides ; la déduplication et le stockage d’objets ont modifié le transfert et la rétention.

Pour les applications Windows en cours d’exécution, Microsoft a introduit avec VSS un modèle coordonné composé d’un demandeur, d’un writer et d’un provider ; cette technologie est apparue avec Windows XP et Windows Server 2003. Dans les plateformes distribuées et conteneurisées, les API de snapshots et d’orchestration ont été standardisées sans résoudre automatiquement la cohérence applicative ([Microsoft: Volume Shadow Copy Service](https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service), [Kubernetes: Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)).

La cyber-reprise a de nouveau déplacé l’accent. Le versioning et les copies hors site ne suffisent pas lorsque des identités hautement privilégiées peuvent supprimer toutes les cibles ou lorsque des images compromises sont réintroduites. Des instances de reprise isolées, des identités séparées, des versions d’objets immuables, une infrastructure déclarative et des zones de redémarrage propres complètent les sauvegardes classiques complètes, incrémentielles et basées sur les journaux. Amazon S3 Object Lock a été introduit en 2018 comme protection WORM des versions d’objets ; cette fonction illustre cette transition, mais ne remplace toujours ni la cohérence applicative ni les tests de restauration ([AWS: introduction de S3 Object Lock](https://aws.amazon.com/about-aws/whats-new/2018/11/s3-object-lock/), [Amazon S3: Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)).

## Checklist administrateur

La planification n’est fiable que lorsque les objectifs, les copies, les accès et les tests sont documentés conjointement. La checklist résume ces dépendances pour la revue et l’exercice de restauration.

- [ ] Les processus métier, la MTD ainsi que le RTO et le RPO pour chaque fonction système sont approuvés.
- [ ] Tous les états faisant autorité et dérivés de la plateforme de messagerie sont inventoriés.
- [ ] La cohérence applicative, la chaîne de journaux et les groupes de cohérence sont documentés pour chaque produit.
- [ ] La restauration des files d’attente, les doublons possibles et la remise en service des flux entrants et sortants sont régis.
- [ ] La configuration, les politiques, le DNS, les certificats, les clés, les secrets, les licences et les runbooks sont dans le périmètre.
- [ ] Au moins une copie de reprise est séparée de la production et des comptes administratifs principaux.
- [ ] L’immutabilité, l’isolation, le chiffrement et l’accès aux clés sont évalués séparément.
- [ ] Le plan de contrôle de reprise et la capacité cible fonctionnent sans la production compromise.
- [ ] L’ordre de redémarrage suit un graphe de dépendances maintenu.
- [ ] Les tests restaurent une transaction métier complète de messagerie et mesurent RPO/RTO.
- [ ] Les résultats, nouvelles dépendances et écarts sont réintégrés dans le runbook et l’architecture.

## Sources

- [NIST – SP 800-34 Rev. 1, Guide de planification de continuité](https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final)
- [NIST – SP 800-34 Rev. 1, PDF](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf)
- [PostgreSQL – Archivage continu et reprise à un instant donné](https://www.postgresql.org/docs/17/continuous-archiving.html)
- [NIST – SP 800-184, Guide de reprise après un événement de cybersécurité](https://www.nist.gov/publications/guide-cybersecurity-event-recovery)
- [NIST – SP 800-184, PDF](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-184.pdf)
- [Microsoft Azure Reliability – Redondance, réplication et sauvegarde](https://learn.microsoft.com/en-us/azure/reliability/concept-redundancy-replication-backup)
- [Kubernetes – Snapshot de volume et cohérence applicative](https://kubernetes.io/blog/2020/12/10/kubernetes-1.20-volume-snapshot-moves-to-ga/)
- [Microsoft Learn – Volume Shadow Copy Service](https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service)
- [IETF RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [Microsoft Learn – Windows Server Backup pour Exchange](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/windows-server-backup)
- [Microsoft Learn – Get-Volume](https://learn.microsoft.com/powershell/module/storage/get-volume)
- [GNU Coreutils – df](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html)
- [Microsoft Learn – Get-ChildItem](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-childitem)
- [GNU Findutils – find](https://www.gnu.org/software/findutils/find)
- [CIS – Control 11: reprise des données](https://cas8.docs.cisecurity.org/en/latest/source/Controls11/)
- [CISA – Guide StopRansomware](https://www.cisa.gov/stopransomware/ransomware-guide)
- [Amazon S3 – Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)
- [Microsoft Learn – Get-FileHash](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – sha256sum](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [NIST – SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)
- [NIST – SP 800-57 Part 1 Rev. 5, PDF](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r5.pdf)
- [Microsoft Learn – w32tm](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings)
- [systemd – timedatectl](https://www.freedesktop.org/software/systemd/man/latest/timedatectl.html)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [ISC BIND 9 – page de manuel dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – page de manuel nc](https://man.openbsd.org/nc)
- [curl – page de manuel de la ligne de commande](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection)
- [Linux man-pages – ss](https://man7.org/linux/man-pages/man8/ss.8.html)
- [IBM – Bande magnétique](https://www.ibm.com/history/magnetic-tape)
- [Kubernetes – Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
- [AWS – Introduction de S3 Object Lock](https://aws.amazon.com/about-aws/whats-new/2018/11/s3-object-lock/)
