---
title: "Exchange On-Premises : architecture et exploitation des serveurs"
blatt: "exchange-on-premises"
description: "Exchange Server dans votre propre centre de données : rôles Mailbox et Edge, pipeline de transport, Active Directory, bases de données ESE, DAG, accès client, sécurité, supervision, sauvegarde et récupération."
fakten:
  - label: Rôle du produit
    wert: Plateforme de messagerie et de travail collaboratif auto-hébergée
    href: https://learn.microsoft.com/en-us/exchange/exchange-server
  - label: Rôles serveur
    wert: Mailbox et Edge Transport en option
    href: https://learn.microsoft.com/en-us/exchange/architecture/architecture
  - label: Système d’exploitation
    wert: Windows Server conformément à la configuration requise pour Exchange
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
    wert: Server Recovery, restauration de base de données et Recovery Database
    href: https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery
werbung:
  - tools
  - newsletter
ctaThemen:
  - exchange-onprem-hybrid
  - smtp-mailflow
translationSourceHash: 8d30441ccb794fc2e8228dfe1fa084ee38a525dcb5e59222e182819a74596bbc
translationModel: gpt-5.6-terra
translatedAt: 2026-10-05T11:22:47.329Z
translationReview: required
---

# Exchange On-Premises : architecture et exploitation des serveurs

**Exchange On-Premises** signifie que l’organisation exploite des serveurs Exchange dans sa propre infrastructure. Elle contrôle les hôtes Windows, Active Directory, les certificats, les services de transport, les files d’attente, les bases de données de boîtes aux lettres et la récupération. Microsoft fournit le code produit, la documentation et les mises à jour ; la disponibilité et une maintenance sécurisée restent à la charge de l’exploitant ([Documentation Exchange Server](https://learn.microsoft.com/en-us/exchange/exchange-server), [Architecture Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/architecture)).

La différence pratique avec Exchange Online apparaît immédiatement lors d’une panne. Un administrateur On-Prem peut examiner une file de transport sur un serveur précis, vérifier l’état d’une copie de base de données et basculer de manière contrôlée vers une autre copie. Il doit toutefois aussi comprendre comment SMTP, Active Directory, ESE, Windows Failover Clustering, IIS et les services Exchange interagissent.

## Le serveur Mailbox est l’élément central

Les serveurs Exchange modernes utilisent le **serveur Mailbox** comme composant commun. Il contient les services Client Access qui acceptent et transmettent les connexions, les services de transport pour le flux de messages ainsi que l’Information Store avec les bases de données de boîtes aux lettres. Une installation peut démarrer modestement ; plusieurs serveurs et copies de bases de données étendent le même modèle de base pour la haute disponibilité ([Architecture Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/architecture)).

Cette consolidation ne signifie pas que toutes les fonctions présentent le même état. Un frontal HTTPS peut être accessible alors que la base de données sollicitée n’est pas montée. SMTP peut accepter des connexions alors qu’un message attend ensuite dans une file d’attente. Le diagnostic suit donc le chemin réel, et pas seulement l’état général du serveur.

Le **rôle Edge Transport** facultatif se situe généralement dans le réseau périmétrique et traite exclusivement le trafic SMTP. EdgeSync transfère des informations sélectionnées sur les destinataires et la configuration vers une instance AD LDS locale. Edge ne contient aucune base de données de boîtes aux lettres et ne remplace pas les serveurs Mailbox internes ([Serveurs Edge Transport](https://learn.microsoft.com/en-us/exchange/architecture/edge-transport-servers/edge-transport-servers)).

## Pile technologique et dépendances

Le composant serveur détermine la pile technologique. Exchange s’exécute sur des versions prises en charge de Windows Server et utilise Active Directory pour la configuration de l’organisation, des serveurs et des destinataires. IIS fournit les points de terminaison HTTP. PowerShell constitue l’interface d’administration. ESE stocke les données des boîtes aux lettres et des files d’attente dans des bases de données distinctes ([Configuration requise pour Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/system-requirements), [Active Directory dans Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)).

| Technologie | Rôle dans l’exploitation d’Exchange | Question importante pour l’administrateur |
|---|---|---|
| Windows Server | Processus, services, réseau, magasin de certificats et journaux d’événements | L’hôte est-il sain et correctement mis à jour ? |
| Active Directory | Organisation Exchange, serveurs, destinataires, RBAC et informations de routage | La modification correcte est-elle visible sur les contrôleurs de domaine utilisés ? |
| IIS et HTTPS | Outlook sur le web, EAC, EWS, ActiveSync, Autodiscover et frontaux MAPI/HTTP | Le nom, le certificat, l’authentification et la route vers le backend correspondent-ils ? |
| SMTP et TLS | Acceptation et transfert des messages | Quel connecteur a accepté la connexion et quel saut suivant a été sélectionné ? |
| ESE | Bases de données de boîtes aux lettres, file de transport et journaux de transactions | Quelle base de données et quelle séquence de journaux vont ensemble ? |
| PowerShell | Administration via des cmdlets et RBAC | Quel rôle, quelle étendue et quel contexte de serveur s’appliquent ? |

Pour les experts, la dépendance à Active Directory est particulièrement importante. Le programme d’installation d’Exchange étend le schéma et inscrit la configuration de l’organisation dans la partition Configuration. Les attributs des destinataires se trouvent dans la partition de domaine. Un délai de réplication ou un contrôleur de domaine inaccessible peut donc affecter différemment diverses fonctions.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-exchange-onprem.svg?v=20260813" title="Interaktive Infografik: Exchange-On-Premises-Pfad von Client und SMTP über Mailboxserver, Transport, Active Directory, ESE und DAG" loading="lazy">
  <a href="/images/kb-interaktiv-exchange-onprem.svg?v=20260813">Ouvrir directement le graphique interactif Exchange On-Premises</a>.
</iframe>

## Le pipeline de transport étape par étape

Avec ces fondations techniques, il est possible de lire plus précisément le cheminement des messages. Une connexion SMTP entrante atteint d’abord le Front End Transport Service. Il accepte le dialogue et le transmet au Transport Service ; il ne place pas lui-même le message dans une boîte aux lettres ([Flux de messagerie et pipeline de transport](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)).

Le **Transport Service** stocke le message dans sa base de données de file d’attente. Il le catégorise ensuite : les destinataires sont résolus, les règles et les agents de transport sont exécutés, et le routage détermine le saut suivant. Pour une boîte aux lettres locale, Mailbox Transport Delivery remet le message au Store. Un message envoyé depuis une boîte aux lettres retourne vers le transport via Mailbox Transport Submission.

Cet ordre explique les observations typiques. Un test SMTP réussi prouve uniquement l’acceptation par le frontal. Un événement `RECEIVE` dans le suivi des messages ne prouve pas encore la remise. Seuls les événements ultérieurs, la file d’attente et, le cas échéant, l’état du Store indiquent où le processus s’est arrêté ([Suivi des messages](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)).

Les agents de transport et les règles de flux de messagerie peuvent rejeter, rediriger, copier ou modifier des messages. Comme plusieurs instances de transport peuvent alors être créées, la recherche ne devrait pas reposer uniquement sur l’objet. Network Message ID, Internet Message ID, expéditeur, destinataire, heure et serveur forment ensemble une piste plus fiable.

## Routage, domaines et connecteurs

Après l’acceptation, Exchange doit savoir si un destinataire est local ou si le message doit être transféré. Les **Accepted Domains** décrivent cette relation. Un domaine autoritaire attend tous les destinataires valides dans sa propre organisation. Un domaine de relais interne permet de transférer les destinataires inconnus. Le relais externe remet entièrement le domaine à un autre serveur de messagerie ([Domaines acceptés dans Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)).

Les connecteurs de réception classifient les sessions entrantes selon la liaison locale, la plage d’adresses IP distantes, l’authentification et les autorisations. Les connecteurs d’envoi sélectionnent un chemin sortant à partir de l’espace d’adressage, du coût, des serveurs sources et du routage DNS ou via hôte intelligent. Plusieurs connecteurs correspondants sont évalués selon les règles de routage documentées ; le nom d’un connecteur ne détermine pas la sélection ([Connecteurs sur les serveurs Exchange](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors), [Routage du courrier dans Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing)).

Pour l’exploitation courante, un modèle simple suffit : le connecteur de réception explique **comment un message arrive** ; le domaine accepté et la résolution du destinataire expliquent **si Exchange est responsable** ; le connecteur d’envoi et le routage expliquent **où il est envoyé ensuite**. Les experts ajoutent les sites AD, les groupes de remise, l’appartenance à une DAG, la portée des connecteurs et les règles de transport.

## Base de données de boîtes aux lettres, journaux et point de contrôle

Lorsque le transport remet le message au Store, une autre partie du système commence. Exchange stocke les boîtes aux lettres dans des bases de données ESE. Les modifications sont d’abord écrites dans des journaux de transactions, puis transférées dans le fichier `.edb`. Le fichier de point de contrôle indique jusqu’à quelle position de journal les pages de la base de données ont été écrites ([Journaux de transactions et fichiers de point de contrôle](https://learn.microsoft.com/en-us/exchange/client-developer/backup-restore/transaction-logs-and-checkpoint-files-for-backup-and-restore-in-exchange)).

Cet ordre permet la récupération après incident, mais exige des fichiers associés. Un fichier `.edb` copié sans les journaux correspondants et sans état d’arrêt connu n’est pas automatiquement récupérable. De même, une sauvegarde ne doit pas supprimer de manière non contrôlée des fichiers journaux encore nécessaires à la récupération ou à la réplication.

La file de transport utilise également ESE, mais constitue une base de données distincte avec ses propres journaux. La base de données de boîtes aux lettres et la file d’attente sont donc supervisées et restaurées séparément. Une base de données de boîtes aux lettres saine ne résout pas un saut SMTP suivant bloqué ; une file vide ne répare pas une copie de boîte aux lettres endommagée ([Files d’attente et base de données de file d’attente](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues)).

## Database Availability Group et Active Manager

Un seul serveur Mailbox explique le fonctionnement normal. Pour la haute disponibilité, plusieurs serveurs sont regroupés dans une **Database Availability Group**, ou DAG. Chaque base de données de boîtes aux lettres possède exactement une copie active et peut avoir des copies passives sur d’autres membres de la DAG. Les modifications sont transférées par réplication de journaux et de blocs, puis rejouées sur les copies passives ([Groupes de disponibilité des bases de données](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups), [Copies de bases de données de boîtes aux lettres](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-copies)).

L’**Active Manager** du Microsoft Exchange Replication Service décide quelle copie est active. Best Copy and Server Selection évalue notamment l’état de copie et de relecture, les blocages d’activation et l’état de santé du serveur. Une Copy Queue à zéro est donc utile, mais ne prouve pas entièrement qu’une copie peut être activée immédiatement ([Active Manager](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/active-manager)).

La haute disponibilité du transport protège une autre partie du trajet. Shadow Redundancy conserve une copie supplémentaire tant que le message est en transit. Safety Net conserve les messages déjà traités pour une éventuelle retransmission après l’activation d’une base de données. DAG, Shadow Redundancy et Safety Net se complètent ; aucune de ces trois fonctions ne remplace une sauvegarde contre une suppression accidentelle ou une corruption non détectée durant une longue période ([Haute disponibilité du transport](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/transport-high-availability)).

## Accès client et Autodiscover

La base de données peut être saine sans qu’un utilisateur puisse pour autant ouvrir Outlook. Les services Client Access acceptent les connexions HTTPS et les transmettent au backend du serveur détenant la base de données active. Un équilibreur de charge nécessite donc plus qu’un port TCP ouvert : le nom, le certificat, le point de terminaison du protocole et la santé du backend doivent correspondre ([Architecture du protocole Client Access](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access)).

Autodiscover fournit au client les paramètres appropriés. Les clients internes au domaine peuvent utiliser les Service Connection Points dans Active Directory ; les clients externes et les autres clients suivent les procédures DNS et HTTPS. Les erreurs sont souvent causées par des SCP obsolètes, des réponses DNS contradictoires, des noms de certificat incorrects ou un frontal qui transfère vers le mauvais backend ([Service Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)).

MAPI over HTTP est le transport Outlook habituel. Outlook sur le web, EWS et ActiveSync utilisent également HTTPS, mais disposent de leurs propres répertoires virtuels, mécanismes d’authentification et caractéristiques applicatives. Un test OWA réussi ne prouve donc pas automatiquement qu’une session MAPI/HTTP est saine ([MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)).

## Active Directory et destinataires

Après le transport et l’accès client, l’annuaire reste la base commune. Exchange stocke la configuration de l’organisation et des serveurs ainsi que les attributs des destinataires dans Active Directory. Les cmdlets n’écrivent pas ces données dans une base de données Exchange privée, mais dans AD via la logique Exchange ([Active Directory dans Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)).

Un problème de destinataire est donc examiné selon trois questions : le bon objet existe-t-il ? Le type, l’adresse principale, les adresses proxy et les attributs cibles sont-ils corrects ? La modification a-t-elle atteint le contrôleur de domaine utilisé par le service Exchange concerné ? Ce n’est qu’ensuite qu’il est utile d’effectuer une recherche dans le transport.

Pour les experts, s’ajoutent les catalogues globaux, les sites AD, Recipient Update, les stratégies de carnet d’adresses et les attributs hybrides. Les modifications directes à l’aide d’outils AD génériques contournent la validation Exchange et peuvent créer des configurations syntaxiquement présentes mais fonctionnellement incohérentes.

## Sécurité et contrôle administratif

Exchange publie des services SMTP et HTTPS et traite des données d’annuaire et de boîtes aux lettres hautement privilégiées. La base repose sur des Security Updates appliquées en temps utile, des points de terminaison accessibles réduits au minimum, des certificats appropriés, des comptes d’administration sécurisés et des modifications traçables ([Security Updates pour Exchange Server](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates), [Certificats TLS dans Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/client-access/certificates)).

RBAC sépare les tâches par rôles, groupes de rôles et étendues. Les droits de boîte aux lettres tels que Full Access ou Send As restent distincts. Administrator Audit Logging enregistre les modifications de cmdlets, mais ne remplace pas les journaux du système d’exploitation, d’Active Directory et de sécurité ([Autorisations dans Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions), [Journalisation d’audit des administrateurs](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/admin-audit-logging/admin-audit-logging)).

Pour les experts, l’interface d’administration elle-même fait partie du modèle de protection. EAC, Exchange Management Shell, Remote PowerShell, WinRM, RDP et l’accès à l’hyperviseur disposent de droits et de protocoles différents. Un administrateur de serveur compromis peut effectuer des actions en dehors d’Exchange RBAC ; la séparation par niveaux et les comptes privilégiés distincts restent donc importants.

## Exploitation : du symptôme au serveur concret

Managed Availability exécute des probes, des moniteurs et des responders. Les Health Sets regroupent ces résultats par fonction et peuvent déclencher des actions de récupération automatique. Ils constituent un bon point de départ, mais pas une vérification complète de bout en bout ([Managed Availability](https://learn.microsoft.com/en-us/exchange/high-availability/managed-availability/managed-availability)).

Pour le flux de messagerie, le diagnostic local commence avec [`Get-Queue`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue) et [`Get-MessageTrackingLog`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog). Le nombre de files d’attente, le saut suivant, l’heure de nouvelle tentative et `LastError` doivent être considérés ensemble. Pour les bases de données, suivent [`Get-MailboxDatabaseCopyStatus`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-mailboxdatabasecopystatus) et [`Test-ReplicationHealth`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/test-replicationhealth). [`Get-ServerHealth`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-serverhealth) affiche les Health Sets et les moniteurs.

Ces cmdlets s’exécutent dans Exchange Management Shell sur des Windows Server pris en charge. Les tests réseau et DNS peuvent en revanche être effectués depuis les deux plateformes d’administration. [`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) vérifie un point de terminaison TCP sous Windows ; [`nc`](https://man.openbsd.org/nc) effectue le même test de port sous Unix. [`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) et [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) vérifient le DNS. Pour SMTP avec STARTTLS, [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) convient ; pour un dialogue SMTP contrôlé, [`swaks`](https://jetmore.org/john/code/swaks/).

L’ordre de diagnostic est le suivant : résoudre le nom public ou interne, vérifier la connexion au bon frontal, confirmer l’acceptation dans le journal de protocole, suivre les événements de suivi, vérifier la file d’attente et le saut suivant, puis n’examiner le Store et la base de données qu’en cas de remise locale.

## Sauvegarde et récupération

La haute disponibilité maintient le service disponible lors de pannes isolées ; la récupération restaure un état antérieur souhaité ou perdu. Exchange documente Server Recovery, la restauration de base de données et Recovery Database comme des procédures distinctes ([Sauvegarde, restauration et reprise après sinistre](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)).

Un inventaire récupérable comprend au minimum Active Directory, l’organisation Exchange et la configuration des serveurs, les certificats et les clés privées, les bases de données de boîtes aux lettres avec leurs journaux, la configuration des connecteurs et des règles, ainsi que les paramètres d’installation et de récupération documentés. La Recovery Database permet de monter une base de données restaurée de manière isolée et de transférer son contenu vers des boîtes aux lettres actives ([Restaurer des données à l’aide d’une base de données de récupération](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/restore-data-using-recovery-dbs)).

Les experts ne vérifient pas seulement si un travail de sauvegarde a réussi. Ils mesurent le temps nécessaire pour restaurer réellement Active Directory, un serveur défaillant, une base de données et le contenu de boîtes aux lettres individuelles. Ils vérifient alors les séquences de journaux requises, les dépendances DNS et de certificats, ainsi que le bon fonctionnement des chemins clients et SMTP après la restauration.

## Évolution technique et limites

Exchange 4.0 est apparu en 1996. Les premières versions utilisaient leur propre annuaire, MAPI et ESE ; SMTP et Active Directory sont devenus des composants centraux de la plateforme avec Exchange 2000. Exchange 2007 a introduit les rôles serveur et Exchange Management Shell. Exchange 2010 a remplacé les anciens modèles de cluster par la Database Availability Group ([Exchange Team : une brève histoire du temps](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388), [Exchange Server 2007 : refonte du transport](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)).

Les versions ultérieures ont à nouveau regroupé les fonctions Client Access et Mailbox dans un composant serveur commun. Exchange Server Subscription Edition a poursuivi la gamme de produits locale en 2025 dans le cadre du Modern Lifecycle. Les numéros de build, les chemins de mise à niveau pris en charge et les Security Updates doivent être vérifiés avant toute modification dans la documentation Microsoft en cours ([Notes de publication d’Exchange Server SE](https://learn.microsoft.com/en-us/exchange/release-notes), [Numéros de build et dates de publication d’Exchange Server](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates)).

Exchange On-Premises convient lorsque l’organisation a besoin de contrôler l’exploitation des bases de données, les chemins réseau et l’intégration locale, et qu’elle peut assurer l’exploitation 24 h/24 et 7 j/7 requise. En contrepartie, il implique des dépendances complexes, une maintenance de sécurité continue et la responsabilité de la récupération. Un seul serveur peut sembler simple ; un service Exchange robuste est toujours aussi un projet Active Directory, réseau, certificats, stockage et exploitation.

## Sources

- [Microsoft Learn – Documentation Exchange Server](https://learn.microsoft.com/en-us/exchange/exchange-server)
- [Microsoft Learn – Architecture Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/architecture)
- [Microsoft Learn – Configuration requise pour Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/system-requirements)
- [Microsoft Learn – Active Directory dans Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)
- [Microsoft Learn – Serveurs Edge Transport](https://learn.microsoft.com/en-us/exchange/architecture/edge-transport-servers/edge-transport-servers)
- [Microsoft Learn – Flux de messagerie et pipeline de transport](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)
- [Microsoft Learn – Suivi des messages](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)
- [Microsoft Learn – Domaines acceptés dans Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)
- [Microsoft Learn – Connecteurs sur les serveurs Exchange](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors)
- [Microsoft Learn – Routage du courrier dans Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing)
- [Microsoft Learn – Journaux de transactions et fichiers de point de contrôle](https://learn.microsoft.com/en-us/exchange/client-developer/backup-restore/transaction-logs-and-checkpoint-files-for-backup-and-restore-in-exchange)
- [Microsoft Learn – Files d’attente et base de données de file d’attente](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues)
- [Microsoft Learn – Groupes de disponibilité des bases de données](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups)
- [Microsoft Learn – Surveiller les groupes de disponibilité des bases de données](https://learn.microsoft.com/en-us/exchange/high-availability/manage-ha/monitor-dags)
- [Microsoft Learn – Copies de bases de données de boîtes aux lettres](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-copies)
- [Microsoft Learn – Active Manager](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/active-manager)
- [Microsoft Learn – Haute disponibilité du transport](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/transport-high-availability)
- [Microsoft Learn – Architecture du protocole Client Access](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access)
- [Microsoft Learn – Service Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)
- [Microsoft Learn – MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)
- [Microsoft Learn – Interfaces d’administration Exchange](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/admin-interface)
- [Microsoft Learn – Certificats TLS dans Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/client-access/certificates)
- [Microsoft Learn – Autorisations dans Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions)
- [Microsoft Learn – Journalisation d’audit des administrateurs](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/admin-audit-logging/admin-audit-logging)
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
- [Exchange Team – Une brève histoire du temps](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388)
- [Exchange Team – Exchange Server 2007 : refonte du transport](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)
- [Microsoft Learn – Notes de publication d’Exchange Server SE](https://learn.microsoft.com/en-us/exchange/release-notes)
- [Microsoft Learn – Numéros de build et dates de publication d’Exchange Server](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates)
