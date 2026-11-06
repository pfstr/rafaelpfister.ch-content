---
title: "Exchange On-Premises : architecture et exploitation des serveurs"
blatt: "exchange-on-premises"
description: "Exchange dans son propre centre de données : rôles Mailbox et Edge, pipeline de transport, Active Directory, bases de données ESE, DAG, accès client, sécurité, supervision, sauvegarde et récupération."
fakten:
  - label: Rôle du produit
    wert: Plateforme de messagerie et de travail collaboratif autoexploitée
    href: https://learn.microsoft.com/en-us/exchange/exchange-server
  - label: Rôles serveur
    wert: Mailbox et Edge Transport en option
    href: https://learn.microsoft.com/en-us/exchange/architecture/architecture
  - label: Système d’exploitation
    wert: Windows Server conformément aux exigences système d’Exchange
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/system-requirements
  - label: Annuaire
    wert: Active Directory Domain Services
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory
  - label: Stockage des boîtes aux lettres
    wert: Base de données ESE, journaux de transactions et point de contrôle
    href: https://learn.microsoft.com/en-us/exchange/client-developer/backup-restore/transaction-logs-and-checkpoint-files-for-backup-and-restore-in-exchange
  - label: Haute disponibilité
    wert: Database Availability Group et copies de bases de données
    href: https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups
  - label: Transport
    wert: Frontend Transport, Transport Service et Mailbox Transport
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow
  - label: Résilience du transport
    wert: Shadow Redundancy et Safety Net
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/transport-high-availability
  - label: Accès client
    wert: HTTPS, MAPI/HTTP, Outlook sur le web, EWS et ActiveSync
    href: https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access
  - label: Administration
    wert: Exchange Admin Center et Exchange Management Shell
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/admin-interface
  - label: Supervision
    wert: Managed Availability, Health Sets, journaux d’événements et compteurs de performances
    href: https://learn.microsoft.com/en-us/exchange/high-availability/managed-availability/managed-availability
  - label: Récupération
    wert: Server Recovery, restauration de bases de données et Recovery Database
    href: https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery
werbung:
  - tools
  - newsletter
ctaThemen:
  - exchange-onprem-hybrid
  - smtp-mailflow
translationSourceHash: 45315e2f767bcdfb1b554eea6df80335a5ca86bbc102a345b83dd0e54cb0693b
translationModel: gpt-5.6-terra
translatedAt: 2026-11-06T03:33:21.370Z
translationReview: required
---

# Exchange On-Premises : architecture et exploitation des serveurs

**Exchange On-Premises** signifie que l’organisation exploite Exchange dans sa propre infrastructure. Elle contrôle les hôtes Windows, Active Directory, les certificats, les services de transport, les files d’attente, les bases de données de boîtes aux lettres et la récupération. Microsoft fournit le code produit, la documentation et les mises à jour ; la disponibilité et la maintenance sécurisée restent à la charge de l’exploitant ([Documentation Exchange Server](https://learn.microsoft.com/en-us/exchange/exchange-server), [Architecture Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/architecture)).

La différence pratique avec Exchange Online apparaît immédiatement en cas d’incident. Un administrateur On-Prem peut examiner une file d’attente de transport sur un serveur précis, contrôler l’état d’une copie de base de données et basculer de manière contrôlée vers une autre copie. Il doit toutefois aussi comprendre comment SMTP, Active Directory, ESE, Windows Failover Clustering, IIS et les services Exchange interagissent.

## Le serveur Mailbox est l’élément central

Les serveurs Exchange modernes utilisent le **serveur Mailbox** comme composant commun. Il contient les services Client Access, qui acceptent et transmettent les connexions, les services de transport pour le flux de messages ainsi que l’Information Store avec les bases de données de boîtes aux lettres. Une installation peut commencer à petite échelle ; plusieurs serveurs et copies de bases de données développent le même modèle de base pour assurer la haute disponibilité ([Architecture Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/architecture)).

Cette consolidation ne signifie pas que toutes les fonctions sont dans le même état. Un frontal HTTPS peut être accessible alors que la base de données concernée n’est pas montée. SMTP peut accepter des connexions tandis qu’un message attend ensuite dans une file d’attente. Le diagnostic suit donc le chemin réel et non seulement l’état global du serveur.

Le **rôle Edge Transport** optionnel se trouve généralement dans le réseau de périmètre et traite exclusivement le trafic SMTP. EdgeSync transfère certaines informations sur les destinataires et la configuration vers une instance AD LDS locale. Edge ne détient aucune base de données de boîtes aux lettres et ne remplace pas les serveurs Mailbox internes ([Serveurs Edge Transport](https://learn.microsoft.com/en-us/exchange/architecture/edge-transport-servers/edge-transport-servers)).

## Pile technologique et dépendances

Le composant serveur détermine la pile technologique. Exchange s’exécute sur des versions prises en charge de Windows Server et utilise Active Directory pour la configuration de l’organisation, des serveurs et des destinataires. IIS fournit les points de terminaison HTTP. PowerShell constitue l’interface d’administration. ESE stocke les données des boîtes aux lettres et des files d’attente dans des bases de données distinctes ([Configuration système requise pour Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/system-requirements), [Active Directory dans Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)).

| Technologie | Rôle dans l’exploitation d’Exchange | Question administrative importante |
|---|---|---|
| Windows Server | Processus, services, réseau, magasin de certificats et journaux d’événements | L’hôte est-il sain et correctement mis à jour ? |
| Active Directory | Organisation Exchange, serveurs, destinataires, RBAC et informations de routage | La bonne modification est-elle visible sur les contrôleurs de domaine utilisés ? |
| IIS et HTTPS | Outlook sur le web, EAC, EWS, ActiveSync, Autodiscover et frontaux MAPI/HTTP | Le nom, le certificat, l’authentification et la route vers le backend correspondent-ils ? |
| SMTP et TLS | Acceptation et transmission des messages | Quel connecteur a accepté la connexion et quel saut suivant a été choisi ? |
| ESE | Bases de données de boîtes aux lettres, file d’attente de transport et journaux de transactions | Quelle base de données et quelle séquence de journaux vont ensemble ? |
| PowerShell | Administration via des cmdlets et RBAC | Quel rôle, quelle étendue et quel contexte serveur s’appliquent ? |

Pour les experts, la dépendance à Active Directory est particulièrement importante. Le programme d’installation d’Exchange étend le schéma et écrit la configuration de l’organisation dans la partition Configuration. Les attributs des destinataires se trouvent dans la partition de domaine. Un retard de réplication ou un contrôleur de domaine inaccessible peut donc affecter diverses fonctions différemment.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-exchange-onprem.svg?v=20260813" title="Interaktive Infografik: Exchange-On-Premises-Pfad von Client und SMTP über Mailboxserver, Transport, Active Directory, ESE und DAG" loading="lazy">
  <a href="/images/kb-interaktiv-exchange-onprem.svg?v=20260813">Ouvrir directement le graphique interactif Exchange On-Premises</a>.
</iframe>

## Le pipeline de transport étape par étape

Avec cette base technique, le cheminement d’un message peut être lu plus précisément. Une connexion SMTP entrante atteint d’abord le Front End Transport Service. Il accepte le dialogue et le transmet au Transport Service ; il ne remet pas lui-même le message dans une boîte aux lettres ([Flux de messagerie et pipeline de transport](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)).

Le **Transport Service** stocke le message dans sa base de données de file d’attente. Il le catégorise ensuite : les destinataires sont résolus, les règles et les agents de transport sont exécutés, et le routage détermine le saut suivant. Pour une boîte aux lettres locale, Mailbox Transport Delivery remet le message au Store. Un message envoyé depuis une boîte aux lettres retourne vers le transport via Mailbox Transport Submission.

Cet ordre explique des observations typiques. Un test SMTP réussi prouve uniquement l’acceptation par le frontal. Un événement `RECEIVE` dans le suivi des messages ne prouve pas encore la remise. Seuls les événements suivants, la file d’attente et, le cas échéant, l’état du Store indiquent où le traitement s’est arrêté ([Suivi des messages](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)).

Les agents de transport et les règles de flux de messagerie peuvent rejeter, rediriger, copier ou modifier des messages. Comme plusieurs instances de transport peuvent alors être créées, la recherche ne devrait pas se limiter à l’objet. L’ID de message réseau, l’ID de message Internet, l’expéditeur, le destinataire, l’heure et le serveur constituent ensemble une piste plus fiable.

## Routage, domaines et connecteurs

Après l’acceptation, Exchange doit savoir si un destinataire est local ou si le message doit être transmis. Les **Accepted Domains** décrivent cette relation. Un domaine faisant autorité attend tous les destinataires valides dans sa propre organisation. Un domaine de relais interne autorise la transmission de destinataires inconnus. Un relais externe remet entièrement le domaine à un autre serveur de messagerie ([Domaines acceptés dans Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)).

Les connecteurs de réception classent les sessions entrantes selon la liaison locale, la plage d’adresses IP distante, l’authentification et les autorisations. Les connecteurs d’envoi sélectionnent une voie sortante à partir de l’espace d’adressage, du coût, des serveurs sources et du routage DNS ou par hôte actif. Plusieurs connecteurs correspondants sont évalués selon les règles de routage documentées ; le nom d’un connecteur ne détermine pas la sélection ([Connecteurs sur les serveurs Exchange](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors), [Routage du courrier dans Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing)).

Pour l’exploitation courante, un modèle simple suffit : le connecteur de réception explique **comment un message entre** ; le domaine accepté et la résolution du destinataire expliquent **si Exchange est responsable** ; le connecteur d’envoi et le routage expliquent **où il est transmis**. Les experts ajoutent les sites AD, les groupes de remise, l’appartenance à une DAG, l’étendue des connecteurs et les règles de transport.

## Base de données de boîtes aux lettres, journaux et point de contrôle

Lorsque le transport remet le message au Store, une autre partie du système entre en jeu. Exchange stocke les boîtes aux lettres dans des bases de données de boîtes aux lettres ESE. Les modifications sont d’abord écrites dans les journaux de transactions, puis intégrées au fichier `.edb`. Le fichier de point de contrôle indique jusqu’à quelle position de journal les pages de la base de données ont été écrites ([Journaux de transactions et fichiers de point de contrôle](https://learn.microsoft.com/en-us/exchange/client-developer/backup-restore/transaction-logs-and-checkpoint-files-for-backup-and-restore-in-exchange)).

Cet ordre permet la récupération après incident, mais exige des fichiers associés. Un fichier `.edb` copié sans les journaux correspondants et sans état d’arrêt connu n’est pas automatiquement récupérable. De même, une sauvegarde ne doit pas supprimer de manière incontrôlée des fichiers journaux encore nécessaires à la récupération ou à la réplication.

La file d’attente de transport utilise également ESE, mais il s’agit d’une base de données distincte avec ses propres journaux. La base de données de boîtes aux lettres et la file d’attente sont donc surveillées et restaurées séparément. Une base de données de boîtes aux lettres saine ne résout pas un saut suivant SMTP bloqué ; une file d’attente vide ne répare pas une copie de base de données de boîtes aux lettres endommagée ([Files d’attente et base de données de file d’attente](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues)).

## Database Availability Group et Active Manager

Un seul serveur Mailbox explique le fonctionnement normal. Pour la haute disponibilité, plusieurs serveurs sont réunis dans une **Database Availability Group**, DAG. Chaque base de données de boîtes aux lettres possède exactement une copie active et peut avoir des copies passives sur d’autres membres de la DAG. Les modifications sont transférées par réplication des journaux et des blocs, puis relues sur les copies passives ([Groupes de disponibilité de bases de données](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups), [Copies de bases de données de boîtes aux lettres](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-copies)).

L’**Active Manager** du Microsoft Exchange Replication Service décide quelle copie est active. Best Copy and Server Selection évalue notamment l’état de copie et de relecture, les blocages d’activation et l’état de santé du serveur. Une Copy Queue à zéro est donc utile, mais ne constitue pas une preuve complète qu’une copie puisse être activée immédiatement ([Active Manager](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/active-manager)).

La haute disponibilité du transport protège une autre partie du chemin. Shadow Redundancy conserve une copie supplémentaire tant que le message est en transit. Safety Net conserve des messages déjà traités pour une éventuelle nouvelle remise après l’activation d’une base de données. DAG, Shadow Redundancy et Safety Net se complètent ; aucune de ces trois fonctions ne remplace une sauvegarde contre une suppression accidentelle ou une corruption non détectée pendant longtemps ([Haute disponibilité du transport](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/transport-high-availability)).

## Accès client et Autodiscover

La base de données peut être saine et un utilisateur peut pourtant ne pas réussir à ouvrir Outlook. Les services Client Access acceptent les connexions HTTPS et les transmettent au backend du serveur hébergeant la base de données active. Un répartiteur de charge a donc besoin de plus qu’un port TCP ouvert : le nom, le certificat, le point de terminaison de protocole et l’état de santé du backend doivent correspondre ([Architecture des protocoles Client Access](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access)).

Autodiscover fournit au client les paramètres appropriés. Les clients internes au domaine peuvent utiliser les Service Connection Points dans Active Directory ; les clients externes et autres suivent les procédures DNS et HTTPS. Les erreurs proviennent souvent de SCP obsolètes, de réponses DNS contradictoires, de noms de certificats erronés ou d’un frontal qui transmet vers le mauvais backend ([Service Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)).

MAPI over HTTP est le transport Outlook typique. Outlook sur le web, EWS et ActiveSync utilisent également HTTPS, mais disposent de leurs propres répertoires virtuels, mécanismes d’authentification et propriétés applicatives. Un test OWA réussi ne prouve donc pas automatiquement qu’une session MAPI/HTTP est saine ([MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)).

## Active Directory et destinataires

Après le transport et l’accès client, l’annuaire reste la base commune. Exchange stocke la configuration de l’organisation et des serveurs ainsi que les attributs des destinataires dans Active Directory. Les cmdlets n’écrivent pas ces données dans une base de données Exchange privée, mais dans AD via la logique Exchange ([Active Directory dans Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)).

Un problème de destinataire est donc examiné selon trois questions : l’objet correct existe-t-il ? Son type, son adresse principale, ses adresses proxy et ses attributs cibles sont-ils corrects ? La modification a-t-elle atteint le contrôleur de domaine utilisé par le service Exchange concerné ? Ce n’est qu’ensuite que la recherche dans le transport est pertinente.

Pour les experts, s’ajoutent les catalogues globaux, les sites AD, Recipient Update, les Address Book Policies et les attributs hybrides. Les modifications directes avec des outils AD génériques contournent la validation d’Exchange et peuvent créer des configurations syntaxiquement présentes, mais fonctionnellement incohérentes.

## Sécurité et contrôle administratif

Exchange publie des services SMTP et HTTPS et traite des données d’annuaire et de boîtes aux lettres hautement privilégiées. La base repose sur des Security Updates installées rapidement, des points de terminaison accessibles au minimum nécessaire, des certificats appropriés, des comptes d’administration sécurisés et des modifications traçables ([Security Updates d’Exchange Server](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates), [Certificats TLS dans Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/client-access/certificates)).

RBAC sépare les tâches par rôles, groupes de rôles et étendues. Les droits de boîte aux lettres tels que Full Access ou Send As restent distincts. Administrator Audit Logging consigne les modifications des cmdlets, mais ne remplace pas les journaux du système d’exploitation, d’Active Directory et de sécurité ([Autorisations dans Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions), [Journalisation d’audit administrateur](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/admin-audit-logging/admin-audit-logging)).

Pour les experts, l’interface d’administration fait elle-même partie du modèle de protection. EAC, Exchange Management Shell, Remote PowerShell, WinRM, RDP et l’accès à l’hyperviseur disposent de droits et de protocoles différents. Un administrateur serveur compromis peut effectuer des actions en dehors du RBAC d’Exchange ; le tiering et des comptes privilégiés distincts restent donc importants.

## Exploitation : du symptôme au serveur concret

Managed Availability exécute des probes, des monitors et des responders. Les Health Sets regroupent ces résultats par fonction et peuvent déclencher des actions de récupération automatiques. Ils constituent un bon point de départ, mais pas un contrôle complet de bout en bout ([Managed Availability](https://learn.microsoft.com/en-us/exchange/high-availability/managed-availability/managed-availability)).

Pour le flux de messagerie, le diagnostic local commence par [`Get-Queue`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue) et [`Get-MessageTrackingLog`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog). Le nombre de files d’attente, le saut suivant, l’heure de nouvelle tentative et `LastError` vont de pair. Pour les bases de données, suivent [`Get-MailboxDatabaseCopyStatus`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-mailboxdatabasecopystatus) et [`Test-ReplicationHealth`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/test-replicationhealth). [`Get-ServerHealth`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-serverhealth) affiche les Health Sets et les monitors.

Ces cmdlets s’exécutent dans Exchange Management Shell sur des serveurs Windows pris en charge. Les tests réseau et DNS peuvent en revanche être réalisés depuis les deux plateformes d’administration. [`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) vérifie un point de terminaison TCP sous Windows ; [`nc`](https://man.openbsd.org/nc) effectue le même test de port sous Unix. [`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) et [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) vérifient DNS. Pour SMTP avec STARTTLS, [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) convient ; pour un dialogue SMTP contrôlé, utilisez [`swaks`](https://jetmore.org/john/code/swaks/).

L’ordre du diagnostic est le suivant : résoudre le nom public ou interne, vérifier la connexion au bon frontal, confirmer l’acceptation dans le journal de protocole, suivre les événements de suivi, vérifier la file d’attente et le saut suivant, puis examiner le Store et la base de données uniquement en cas de remise locale.

## Sauvegarde et récupération

La haute disponibilité maintient le service disponible lors de pannes individuelles ; la récupération restaure un état antérieur souhaité ou perdu. Exchange documente Server Recovery, la restauration de bases de données et Recovery Database comme des procédures distinctes ([Sauvegarde, restauration et reprise après sinistre](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)).

Un inventaire récupérable comprend au minimum Active Directory, l’organisation Exchange et la configuration des serveurs, les certificats et clés privées, les bases de données de boîtes aux lettres avec leurs journaux, la configuration des connecteurs et des règles ainsi que les paramètres d’installation et de récupération documentés. La Recovery Database permet de monter une base de données restaurée de manière isolée et de transférer son contenu vers des boîtes aux lettres actives ([Restaurer des données à l’aide d’une base de données de récupération](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/restore-data-using-recovery-dbs)).

Les experts ne vérifient pas seulement qu’un travail de sauvegarde a réussi. Ils mesurent le temps réellement nécessaire pour restaurer Active Directory, un serveur défaillant, une base de données et le contenu de boîtes aux lettres individuelles. Ils vérifient alors les séquences de journaux requises, les dépendances DNS et de certificats, ainsi que le bon fonctionnement des chemins client et SMTP après la restauration.

## Évolution technique et limites

Exchange 4.0 est apparu en 1996. Les premières versions utilisaient leur propre annuaire, MAPI et ESE ; SMTP et Active Directory sont devenus des composants centraux de la plateforme avec Exchange 2000. Exchange 2007 a introduit les rôles serveur et Exchange Management Shell. Exchange 2010 a remplacé les anciens modèles de cluster par la Database Availability Group ([Exchange Team : A brief history of time](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388), [Exchange Team : refonte du transport Exchange Server 2007](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)).

Les versions ultérieures ont de nouveau regroupé les fonctions Client Access et Mailbox dans un composant serveur commun. Exchange Server Subscription Edition a poursuivi la gamme de produits locale dans le Modern Lifecycle en 2025. Les numéros de build, les chemins de mise à niveau pris en charge et les Security Updates sont vérifiés avant chaque modification dans la documentation Microsoft actualisée ([Notes de publication d’Exchange Server SE](https://learn.microsoft.com/en-us/exchange/release-notes), [Numéros de build et dates de publication d’Exchange Server](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates)).

Exchange On-Premises convient lorsque l’organisation a besoin de contrôler l’exploitation des bases de données, les chemins réseau et l’intégration locale, et qu’elle peut assurer l’exploitation 24 h/24 et 7 j/7 nécessaire. En contrepartie, il implique des dépendances complexes, une maintenance de sécurité continue et la responsabilité de la récupération. Un serveur unique peut sembler simple ; un service Exchange robuste est toujours aussi un projet Active Directory, réseau, certificats, stockage et exploitation.

## Sources

- [Microsoft Learn – Documentation Exchange Server](https://learn.microsoft.com/en-us/exchange/exchange-server)
- [Microsoft Learn – Architecture Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/architecture)
- [Microsoft Learn – Configuration système requise pour Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/system-requirements)
- [Microsoft Learn – Active Directory dans Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)
- [Microsoft Learn – Serveurs Edge Transport](https://learn.microsoft.com/en-us/exchange/architecture/edge-transport-servers/edge-transport-servers)
- [Microsoft Learn – Flux de messagerie et pipeline de transport](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)
- [Microsoft Learn – Suivi des messages](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)
- [Microsoft Learn – Domaines acceptés dans Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)
- [Microsoft Learn – Connecteurs sur les serveurs Exchange](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors)
- [Microsoft Learn – Routage du courrier dans Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing)
- [Microsoft Learn – Journaux de transactions et fichiers de point de contrôle](https://learn.microsoft.com/en-us/exchange/client-developer/backup-restore/transaction-logs-and-checkpoint-files-for-backup-and-restore-in-exchange)
- [Microsoft Learn – Files d’attente et base de données de file d’attente](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues)
- [Microsoft Learn – Groupes de disponibilité de bases de données](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups)
- [Microsoft Learn – Superviser les groupes de disponibilité de bases de données](https://learn.microsoft.com/en-us/exchange/high-availability/manage-ha/monitor-dags)
- [Microsoft Learn – Copies de bases de données de boîtes aux lettres](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-copies)
- [Microsoft Learn – Active Manager](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/active-manager)
- [Microsoft Learn – Haute disponibilité du transport](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/transport-high-availability)
- [Microsoft Learn – Architecture des protocoles Client Access](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access)
- [Microsoft Learn – Service Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)
- [Microsoft Learn – MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)
- [Microsoft Learn – Interfaces d’administration Exchange](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/admin-interface)
- [Microsoft Learn – Certificats TLS dans Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/client-access/certificates)
- [Microsoft Learn – Autorisations dans Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions)
- [Microsoft Learn – Journalisation d’audit administrateur](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/admin-audit-logging/admin-audit-logging)
- [Microsoft Learn – Managed Availability](https://learn.microsoft.com/en-us/exchange/high-availability/managed-availability/managed-availability)
- [Microsoft Learn – Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue)
- [Microsoft Learn – Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog)
- [Microsoft Learn – Get-MailboxDatabaseCopyStatus](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-mailboxdatabasecopystatus)
- [Microsoft Learn – Test-ReplicationHealth](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/test-replicationhealth)
- [Microsoft Learn – Get-ServerHealth](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-serverhealth)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc(1)](https://man.openbsd.org/nc)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – manuel dig](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Swaks – outil de test SMTP](https://jetmore.org/john/code/swaks/)
- [Microsoft Learn – Sauvegarde, restauration et reprise après sinistre](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)
- [Microsoft Learn – Restaurer des données à l’aide d’une base de données de récupération](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/restore-data-using-recovery-dbs)
- [Exchange Team – A brief history of time](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388)
- [Exchange Team – Refonte du transport Exchange Server 2007](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)
- [Microsoft Learn – Notes de publication d’Exchange Server SE](https://learn.microsoft.com/en-us/exchange/release-notes)
- [Microsoft Learn – Numéros de build et dates de publication d’Exchange Server](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates)
