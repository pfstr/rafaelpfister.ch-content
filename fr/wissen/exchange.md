---
title: "Microsoft Exchange : famille de produits et modèles d’exploitation"
blatt: "exchange"
description: "Microsoft Exchange en tant que famille de produits : termes et protocoles communs, différences entre Exchange Online et Exchange Server, ainsi que les rôles du fonctionnement hybride et du flux de messagerie hybride."
fakten:
  - label: Famille de produits
    wert: Exchange Online et Exchange Server
    href: https://learn.microsoft.com/en-us/exchange/
  - label: Fonctions principales
    wert: E-mail, calendrier, contacts, carnet d’adresses et stratégies
    href: https://learn.microsoft.com/en-us/exchange/recipients/recipients
  - label: Exploitation dans le cloud
    wert: Exchange Online au sein de Microsoft 365
    href: https://learn.microsoft.com/en-us/exchange/exchange-online
  - label: Exploitation sur site
    wert: Exchange sur Windows Server et Active Directory
    href: https://learn.microsoft.com/en-us/exchange/exchange-server
  - label: Coexistence
    wert: Exchange Hybrid relie l’organisation locale et Exchange Online
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Transport de messagerie
    wert: SMTP, connecteurs, règles, files d’attente et distribution
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow
  - label: Accès client
    wert: HTTPS, MAPI/HTTP, Outlook sur le web et clients mobiles
    href: https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access
  - label: Destinataires
    wert: Boîtes aux lettres, groupes, contacts, utilisateurs de messagerie et ressources
    href: https://learn.microsoft.com/en-us/exchange/recipients/recipients
  - label: Annuaires
    wert: Active Directory local, Microsoft Entra ID dans le cloud
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory
  - label: Administration
    wert: Centre d’administration Exchange, PowerShell et autorisations basées sur les rôles
    href: https://learn.microsoft.com/en-us/exchange/permissions/permissions
  - label: Diagnostic cloud
    wert: Message Trace, rapports et Microsoft 365 Service Health
    href: https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message
  - label: Diagnostic serveur
    wert: Files d’attente, suivi des messages, Health Sets et copies de bases de données
    href: https://learn.microsoft.com/en-us/exchange/server-health/server-health
werbung:
  - tools
  - newsletter
ctaThemen:
  - microsoft-365-exchange
  - exchange-onprem-hybrid
translationSourceHash: 120d6e74efed65ae14cfc80b5e66597bd4ea19799d6b0c4d9e22fce0152b8d30
translationModel: gpt-5.6-terra
translatedAt: 2026-11-06T03:29:42.394Z
translationReview: automatic
---

# Microsoft Exchange : famille de produits et modèles d’exploitation

Le nom **Microsoft Exchange** désigne aujourd’hui deux plateformes étroitement liées, mais exploitées différemment. Avec **Exchange Online**, Microsoft exploite les serveurs, les copies de bases de données et l’infrastructure de transport interne. Avec **Exchange Server**, les hôtes, Active Directory, les bases de données, les files d’attente, les certificats et la restauration relèvent de la responsabilité de l’organisation. **Exchange Hybrid** relie les deux organisations lorsque les boîtes aux lettres ou les fonctions sont réparties entre les deux environnements ([Microsoft Learn: Exchange](https://learn.microsoft.com/en-us/exchange/), [Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

Cette distinction est le point de départ de toute question ultérieure. Une boîte aux lettres peut se trouver dans le cloud ou dans le propre centre de données. L’expéditeur visible, le domaine SMTP et le carnet d’adresses peuvent néanmoins être utilisés conjointement. Ce n’est que lorsque l’emplacement de la boîte aux lettres, l’origine de ses attributs de destinataire et le chemin réel du message sont établis qu’il devient possible d’examiner utilement la distribution, les autorisations et les erreurs.

## Clarifier d’abord : où se trouve la boîte aux lettres ?

Exchange gère les e-mails, les calendriers, les contacts, les tâches, les objets du carnet d’adresses et les droits d’accès. Pour les utilisateurs, cela reste largement identique. Pour les administrateurs, l’emplacement de la boîte aux lettres change en revanche presque chaque outil et chaque responsabilité.

| Question | Exchange Online | Exchange local | Exchange Hybrid |
|---|---|---|---|
| Qui exploite les serveurs de boîtes aux lettres ? | Microsoft | l’organisation elle-même | les deux environnements pour leurs boîtes aux lettres respectives |
| Où les destinataires sont-ils gérés ? | Exchange Online et Entra ID | Exchange Server et Active Directory | généralement créés localement et synchronisés vers Entra ID ; le modèle exact doit être documenté |
| Où suit-on un message ? | Message Trace | Message Tracking Logs et files d’attente | des deux côtés, reliés par l’heure, l’expéditeur, le destinataire et les ID de message |
| Qui peut activer les copies de bases de données ? | Microsoft | l’administration Exchange interne | l’exploitant de l’environnement concerné |
| Qu’est-ce qui relie les deux environnements ? | non applicable | non applicable | synchronisation d’annuaires, relations d’organisation, OAuth, Autodiscover et connecteurs SMTP |

Le tableau ne donne qu’une vue d’ensemble. Les quatre approfondissements traitent séparément les modèles d’exploitation :

- [Exchange Online](/kb/exchange-online) explique les objets de locataire, EOP, les connecteurs, Message Trace, la conservation et l’exploitation d’un service cloud.
- [Exchange On-Premises](/kb/exchange-on-premises) suit le pipeline de transport, les bases de données ESE, les DAG, Active Directory et la récupération dans le propre centre de données.
- [Exchange Hybrid](/kb/exchange-hybrid) traite la synchronisation d’annuaires, la gestion des destinataires, OAuth, les relations d’organisation, Autodiscover et le Hybrid Configuration Wizard.
- [Flux de messagerie hybride](/kb/hybrid-mailfluss) suit les messages entre Internet, Exchange Online, l’organisation locale et une passerelle de messagerie facultative.

## Structure technique commune : ce qui reste identique dans toutes les variantes d’Exchange

Une fois le lieu d’exploitation clarifié, il vaut la peine d’examiner le modèle métier commun. Exchange connaît les **destinataires**, les **messages**, les **boîtes aux lettres**, les **règles de transport**, les **domaines** et les **connecteurs**. Ces termes apparaissent dans le cloud et sur les serveurs internes, même si les systèmes sous-jacents sont accessibles différemment ([Microsoft Learn: Recipients](https://learn.microsoft.com/en-us/exchange/recipients/recipients), [Mail flow and the transport pipeline](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)).

Un destinataire est d’abord un objet d’annuaire activé pour la messagerie. Il possède des adresses et un type, par exemple boîte aux lettres utilisateur, boîte aux lettres partagée, groupe, contact ou utilisateur de messagerie. L’objet répond à la question de savoir **qui** représente une adresse et **où** Exchange doit distribuer le message. La boîte aux lettres stocke ensuite les éléments réels. Un objet destinataire défectueux et une boîte aux lettres saine peuvent donc exister simultanément, ou inversement.

Le cheminement des messages suit également les mêmes grandes étapes sur les deux plateformes : Exchange accepte un message SMTP, résout les destinataires, applique les règles de transport et les fonctions de protection, choisit la destination suivante et le distribue soit dans une boîte aux lettres, soit vers un autre saut SMTP. L’implémentation exacte diffère. Sur les serveurs internes, l’administrateur peut voir les files d’attente et les journaux de suivi locaux ; dans Exchange Online, Message Trace, les rapports et Service Health sont disponibles à cette fin ([Exchange Server mail flow](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow), [Trace an email message in Exchange Online](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-exchange.svg?v=20260813" title="Interaktive Infografik: Exchange-Produktfamilie mit Transport, Postfächern, Exchange Online und Hybridverbindungen" loading="lazy">
  <a href="/images/kb-interaktiv-exchange.svg?v=20260813">Ouvrir directement la vue d’ensemble interactive d’Exchange</a>.
</iframe>

## De l’adresse au chemin du message

Le domaine SMTP commun conduit souvent à l’hypothèse erronée que tous les messages empruntent le même chemin. En réalité, une combinaison de DNS, de domaines acceptés, d’objet destinataire, de connecteurs et de règles détermine où un message est envoyé ensuite.

Un **domaine accepté** indique à Exchange comment traiter un domaine. Pour un domaine faisant autorité, Exchange attend tous les destinataires valides dans son propre annuaire. Pour un domaine de relais interne, les destinataires inconnus peuvent être transmis à un autre système. Ce paramètre n’est donc pas une simple ligne d’inventaire, mais fait partie de la décision de distribution ([Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains), [Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)).

Les **connecteurs** déterminent ensuite de quels systèmes Exchange accepte les messages et vers quels systèmes il les envoie. Dans Exchange Server, les connecteurs de réception et d’envoi fonctionnent avec des liaisons locales, des espaces d’adressage, des serveurs sources, des hôtes intelligents et des autorisations. Exchange Online utilise des connecteurs entrants et sortants pour les relations avec l’infrastructure interne ou les partenaires ([Connectors on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors), [Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)).

Ce n’est qu’à ce stade que les variantes hybrides deviennent importantes. Centralized Mail Transport, une passerelle en amont ou un domaine de destinataires partagé modifient les sauts supplémentaires et les responsabilités. Ils ont donc leur place dans l’article dédié [Flux de messagerie hybride](/kb/hybrid-mailfluss), et non parmi les sujets d’identité ou de client.

## De l’authentification à la boîte aux lettres

Le chemin du message n’explique pas encore comment Outlook trouve sa boîte aux lettres. Exchange utilise **Autodiscover** à cette fin. Un client commence par l’identité de l’utilisateur et en déduit le point de terminaison de service approprié. Localement, des points de connexion de service Active Directory et le DNS peuvent intervenir ; dans Exchange Online, les points de terminaison Microsoft 365 mènent au service cloud ([Microsoft Learn: Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)).

Après la découverte du point de terminaison, l’accès Outlook moderne passe par HTTPS, notamment avec MAPI over HTTP. Outlook sur le web, Exchange ActiveSync et diverses API utilisent également HTTPS, mais chacun avec ses propres protocoles d’application et autorisations. Une connexion réussie au portail Microsoft 365 ne prouve donc pas automatiquement qu’Autodiscover, le protocole Outlook ou l’accès à la boîte aux lettres concernée fonctionnent ([Client Access protocol architecture](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access), [MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)).

En fonctionnement hybride, une décision supplémentaire s’ajoute : la boîte aux lettres est-elle locale ou en ligne ? Autodiscover et les attributs de destinataire doivent diriger le client vers le bon environnement. Ce n’est qu’ensuite que les fonctions interorganisationnelles telles que les informations de disponibilité ou les déplacements de boîtes aux lettres entrent en jeu. Cet ordre est traité pas à pas dans l’article [Exchange Hybrid](/kb/exchange-hybrid).

## Administration et autorisations

Une fois les chemins de données et d’accès compris, vient la question de savoir qui peut les modifier. Exchange utilise un contrôle d’accès basé sur les rôles. Les rôles contiennent des cmdlets et des paramètres, les groupes de rôles ou les attributions de rôles les relient aux administrateurs, et les étendues limitent le périmètre. Exchange Online et Exchange Server disposent de concepts RBAC apparentés, mais de configurations distinctes ([Permissions in Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions), [Permissions in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)).

Au quotidien, cela signifie qu’un rôle d’administrateur Entra, un groupe de rôles Exchange et une autorisation de boîte aux lettres ne sont pas la même chose. **Full Access** permet d’ouvrir une boîte aux lettres, **Send As** d’envoyer en tant que destinataire et **Send on Behalf** d’envoyer de manière identifiable au nom de quelqu’un. Aucune de ces autorisations n’explique à elle seule si une application peut accéder via Microsoft Graph ou EWS ([Manage permissions for recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)).

La question d’expert n’est donc pas « L’utilisateur est-il administrateur ? », mais : quelle identité se connecte, quel rôle s’applique dans quelle organisation Exchange, quel objet est ciblé et quelle autorisation supplémentaire de boîte aux lettres ou d’application est vérifiée ?

## L’exploitation et le dépannage commencent du bon côté

Un diagnostic pertinent commence par trois indications : **utilisateur ou destinataire concerné, heure précise et emplacement de la boîte aux lettres**. Le chemin est ensuite suivi dans l’ordre : DNS ou Autodiscover, authentification, point de terminaison Exchange, résolution du destinataire, événement de transport et distribution dans la boîte aux lettres.

Pour Exchange Online, Message Trace et Microsoft 365 Service Health fournissent l’état du service visible par le client. Pour Exchange Server, s’ajoutent les files d’attente locales, Message Tracking Logs, Health Sets, Event Logs et copies de bases de données. En fonctionnement hybride, les éléments des deux côtés sont réunis sur une chronologie commune ([Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq), [Server health and performance](https://learn.microsoft.com/en-us/exchange/server-health/server-health)).

Les articles approfondis contiennent chacun des blocs de diagnostic Windows/Unix appropriés et la documentation officielle des outils utilisés. Ici, la règle d’exploitation la plus importante suffit : déterminer d’abord le lieu et le chemin, puis choisir l’outil.

## Stockage des données, disponibilité et restauration

La différence entre le cloud et l’exploitation interne est particulièrement évidente lors de la récupération. Exchange Server stocke les boîtes aux lettres dans des bases de données ESE avec des journaux de transactions. Les groupes de disponibilité de bases de données répliquent les copies de bases de données et permettent des activations sur d’autres serveurs. L’organisation reste néanmoins responsable de la stratégie de sauvegarde, de la capacité de restauration et de la dépendance à Active Directory ([Database availability groups](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups), [Backup, restore and disaster recovery](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)).

Exchange Online assure la redondance des bases de données et du transport dans le cadre du service. Les administrateurs de locataire travaillent à la place avec les éléments supprimés, Single Item Recovery, la conservation, les conservations légales et, le cas échéant, des exigences de sauvegarde externes. La résilience du service Microsoft et une règle de conservation métier répondent à des questions différentes : l’une protège le service en cours d’exécution, l’autre détermine quels contenus sont conservés après suppression ou à des fins de conformité ([Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency), [Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)).

En fonctionnement hybride, les deux modèles de récupération doivent être documentés côte à côte. La synchronisation d’annuaires, les certificats, la configuration OAuth et les connecteurs sont également nécessaires pour rétablir la connexion après une panne. Ces configurations ne contiennent certes pas de contenu de boîte aux lettres, mais elles déterminent si les deux organisations Exchange peuvent à nouveau collaborer.

## Évolution technique

Exchange 4.0 est apparu en 1996 comme successeur des premières plateformes de messagerie Microsoft. Les premières versions utilisaient un annuaire propre, MAPI et la famille de bases de données ESE ; les protocoles Internet ont progressivement gagné en importance. Avec Exchange 2000, Active Directory et SMTP sont devenus des composants centraux. Exchange 2007 a introduit des rôles serveur marqués et l’Exchange Management Shell, Exchange 2010 le Database Availability Group ([Exchange Team: A brief history of time](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388), [Exchange Team: Exchange Server 2007 transport redesign](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)).

En parallèle, Microsoft a développé ses offres Exchange hébergées en Exchange Online. Hybrid n’est donc pas né comme un produit unique, mais comme la connexion de deux organisations Exchange indépendantes. Exchange Server Subscription Edition poursuit cette ligne de développement locale dans le Modern Lifecycle depuis 2025. Les informations concrètes sur les builds et les mises à jour sont vérifiées avant toute modification dans la documentation Microsoft continuellement mise à jour ([Exchange Server SE release notes](https://learn.microsoft.com/en-us/exchange/release-notes), [Exchange Server Subscription Edition lifecycle](https://learn.microsoft.com/en-us/lifecycle/products/exchange-server-subscription-edition)).

## Sources

- [Microsoft Learn – Exchange](https://learn.microsoft.com/en-us/exchange/)
- [Microsoft Learn – Exchange Online](https://learn.microsoft.com/en-us/exchange/exchange-online)
- [Microsoft Learn – Exchange Server](https://learn.microsoft.com/en-us/exchange/exchange-server)
- [Microsoft Learn – Architecture d’Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/architecture)
- [Microsoft Learn – Déploiements hybrides Exchange](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)
- [Microsoft Learn – Destinataires](https://learn.microsoft.com/en-us/exchange/recipients/recipients)
- [Microsoft Learn – Flux de messagerie et pipeline de transport](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)
- [Microsoft Learn – Suivre un e-mail](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)
- [Microsoft Learn – FAQ sur Message Trace](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq)
- [Microsoft Learn – Domaines acceptés dans Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)
- [Microsoft Learn – Domaines acceptés dans Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)
- [Microsoft Learn – Connecteurs sur les serveurs Exchange](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors)
- [Microsoft Learn – Configurer des connecteurs pour acheminer le courrier](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)
- [Microsoft Learn – Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)
- [Microsoft Learn – Architecture des protocoles Client Access](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access)
- [Microsoft Learn – MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)
- [Microsoft Learn – Active Directory dans Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)
- [Microsoft Learn – Autorisations dans Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions)
- [Microsoft Learn – Autorisations dans Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)
- [Microsoft Learn – Gérer les autorisations des destinataires dans Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)
- [Microsoft Learn – État de santé et performances du serveur](https://learn.microsoft.com/en-us/exchange/server-health/server-health)
- [Microsoft Learn – Groupes de disponibilité de bases de données](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups)
- [Microsoft Learn – Sauvegarde, restauration et reprise après sinistre](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)
- [Microsoft Service Assurance – Résilience des données Exchange Online](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)
- [Microsoft Purview – Stratégies de conservation pour Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)
- [Exchange Team – Une brève histoire du temps](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388)
- [Exchange Team – Refonte du transport d’Exchange Server 2007](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)
- [Microsoft Learn – Notes de publication d’Exchange Server SE](https://learn.microsoft.com/en-us/exchange/release-notes)
- [Microsoft Lifecycle – Exchange Server Subscription Edition](https://learn.microsoft.com/en-us/lifecycle/products/exchange-server-subscription-edition)
