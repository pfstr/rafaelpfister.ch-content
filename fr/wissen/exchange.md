---
title: "Microsoft Exchange : famille de produits et modèles de fonctionnement"
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
  - label: Exploitation en interne
    wert: Exchange Server sur Windows Server et Active Directory
    href: https://learn.microsoft.com/en-us/exchange/exchange-server
  - label: Coexistence
    wert: Exchange Hybrid relie l’organisation locale et Exchange Online
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Transport de messagerie
    wert: SMTP, connecteurs, règles, files d’attente et distribution
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow
  - label: Accès client
    wert: HTTPS, MAPI/HTTP, Outlook sur le Web et clients mobiles
    href: https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access
  - label: Destinataires
    wert: Boîtes aux lettres, groupes, contacts, utilisateurs à extension messagerie et ressources
    href: https://learn.microsoft.com/en-us/exchange/recipients/recipients
  - label: Annuaires
    wert: Active Directory local, Microsoft Entra ID dans le cloud
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory
  - label: Administration
    wert: Centre d’administration Exchange, PowerShell et droits basés sur les rôles
    href: https://learn.microsoft.com/en-us/exchange/permissions/permissions
  - label: Diagnostic cloud
    wert: Suivi des messages, rapports et Microsoft 365 Service Health
    href: https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message
  - label: Diagnostic serveur
    wert: Files d’attente, suivi des messages, ensembles d’intégrité et copies de bases de données
    href: https://learn.microsoft.com/en-us/exchange/server-health/server-health
werbung:
  - tools
  - newsletter
ctaThemen:
  - microsoft-365-exchange
  - exchange-onprem-hybrid
translationSourceHash: f76fa79338367d134f158edf1b05fbee051e68cffacc657f421ca0bbdd03dc2f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-05T11:14:43.611Z
translationReview: automatic
---

# Microsoft Exchange : famille de produits et modèles de fonctionnement

Le nom **Microsoft Exchange** désigne aujourd’hui deux plateformes étroitement liées, mais exploitées différemment. Avec **Exchange Online**, Microsoft exploite les serveurs, les copies de bases de données et l’infrastructure de transport interne. Avec **Exchange Server**, les hôtes, Active Directory, les bases de données, les files d’attente, les certificats et la récupération relèvent de votre propre responsabilité. **Exchange Hybrid** relie les deux organisations lorsque les boîtes aux lettres ou les fonctions sont réparties entre elles ([Microsoft Learn: Exchange](https://learn.microsoft.com/en-us/exchange/), [Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

Cette distinction est le point de départ de toute autre question. Une boîte aux lettres peut se trouver dans le cloud ou dans votre propre centre de données. L’expéditeur visible, le domaine SMTP et le carnet d’adresses peuvent néanmoins être utilisés conjointement. Ce n’est que lorsque l’emplacement de la boîte aux lettres, l’origine de ses attributs de destinataire et le chemin réel du message sont établis que la distribution, les autorisations et les erreurs peuvent être analysées de manière pertinente.

## À clarifier d’abord : où se trouve la boîte aux lettres ?

Exchange gère les e-mails, les calendriers, les contacts, les tâches, les objets du carnet d’adresses et les droits d’accès. Pour les utilisateurs, cela reste en grande partie identique. Pour les administrateurs, l’emplacement de la boîte aux lettres modifie en revanche presque chaque outil et chaque responsabilité.

| Question | Exchange Online | Exchange local | Exchange Hybrid |
|---|---|---|---|
| Qui exploite les serveurs de boîtes aux lettres ? | Microsoft | votre propre organisation | les deux côtés pour leurs boîtes aux lettres respectives |
| Où les destinataires sont-ils gérés ? | Exchange Online et Entra ID | Exchange Server et Active Directory | généralement créés localement et synchronisés vers Entra ID ; le modèle exact doit être documenté |
| Où un message est-il suivi ? | Suivi des messages | Journaux de suivi des messages et files d’attente | des deux côtés, reliés par l’heure, l’expéditeur, le destinataire et les ID de message |
| Qui peut activer des copies de bases de données ? | Microsoft | votre propre administration Exchange | l’exploitant du côté concerné |
| Qu’est-ce qui relie les deux côtés ? | non applicable | non applicable | synchronisation d’annuaire, relations organisationnelles, OAuth, Autodiscover et connecteurs SMTP |

Le tableau n’est qu’une vue d’ensemble. Les quatre approfondissements traitent séparément les modèles de fonctionnement :

- [Exchange Online](/kb/exchange-online) explique les objets de locataire, EOP, les connecteurs, le suivi des messages, la conservation et l’exploitation d’un service cloud.
- [Exchange On-Premises](/kb/exchange-on-premises) suit le pipeline de transport, les bases de données ESE, les DAG, Active Directory et la récupération dans votre propre centre de données.
- [Exchange Hybrid](/kb/exchange-hybrid) traite de la synchronisation d’annuaire, de la gestion des destinataires, d’OAuth, des relations organisationnelles, d’Autodiscover et de l’Hybrid Configuration Wizard.
- [Flux de messagerie hybride](/kb/hybrid-mailfluss) suit les messages entre Internet, Exchange Online, l’organisation locale et une passerelle de messagerie facultative.

## Structure technique commune : ce qui reste identique dans toutes les variantes d’Exchange

Une fois le lieu d’exploitation clarifié, il est utile d’examiner le modèle fonctionnel commun. Exchange connaît les **destinataires**, les **messages**, les **boîtes aux lettres**, les **règles de transport**, les **domaines** et les **connecteurs**. Ces termes apparaissent dans le cloud et sur vos propres serveurs, même si les systèmes sous-jacents sont accessibles différemment ([Microsoft Learn: Recipients](https://learn.microsoft.com/en-us/exchange/recipients/recipients), [Mail flow and the transport pipeline](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)).

Un destinataire est d’abord un objet d’annuaire à extension messagerie. Il possède des adresses et un type, par exemple une boîte aux lettres utilisateur, une boîte aux lettres partagée, un groupe, un contact ou un utilisateur à extension messagerie. L’objet répond à la question de savoir **qui** une adresse représente et **où** Exchange doit distribuer le message. La boîte aux lettres stocke ensuite les éléments proprement dits. Ainsi, un objet de destinataire défectueux et une boîte aux lettres saine peuvent coexister, ou inversement.

Le chemin des messages suit également les mêmes grandes étapes sur les deux plateformes : Exchange accepte un message SMTP, résout les destinataires, applique les règles de transport et les fonctions de protection, sélectionne la destination suivante et distribue le message soit dans une boîte aux lettres, soit vers un autre saut SMTP. L’implémentation exacte diffère. Sur vos propres serveurs, l’administrateur peut consulter les files d’attente et les journaux de suivi locaux ; dans Exchange Online, le suivi des messages, les rapports et Service Health sont disponibles à cette fin ([Exchange Server mail flow](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow), [Trace an email message in Exchange Online](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-exchange.svg?v=20260813" title="Interaktive Infografik: Exchange-Produktfamilie mit Transport, Postfächern, Exchange Online und Hybridverbindungen" loading="lazy">
  <a href="/images/kb-interaktiv-exchange.svg?v=20260813">Ouvrir directement la vue d’ensemble interactive d’Exchange</a>.
</iframe>

## De l’adresse au chemin du message

Le domaine SMTP commun conduit souvent à l’hypothèse erronée que tous les messages empruntent le même chemin. En réalité, une combinaison de DNS, de domaines acceptés, d’objets de destinataire, de connecteurs et de règles détermine où un message va ensuite.

Un **domaine accepté** indique à Exchange comment traiter un domaine. Pour un domaine faisant autorité, Exchange attend tous les destinataires valides dans son propre annuaire. Pour un domaine de relais interne, les destinataires inconnus peuvent être transférés vers un autre système. Ce paramètre n’est donc pas une simple ligne d’inventaire, mais fait partie de la décision de distribution ([Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains), [Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)).

Les **connecteurs** déterminent ensuite de quels systèmes Exchange accepte les messages et vers quels systèmes il les envoie. Dans Exchange Server, les connecteurs de réception et d’envoi utilisent des liaisons locales, des espaces d’adressage, des serveurs sources, des hôtes intelligents et des autorisations. Exchange Online utilise des connecteurs entrants et sortants pour les relations avec votre propre infrastructure ou avec des partenaires ([Connectors on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors), [Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)).

C’est seulement à ce stade que les variantes hybrides deviennent importantes. Le transport de messagerie centralisé, une passerelle en amont ou un domaine de destinataires partagé modifient les sauts supplémentaires et les responsabilités. Ils ont donc leur place dans l’article dédié [Flux de messagerie hybride](/kb/hybrid-mailfluss), et non parmi les sujets d’identité ou de client.

## De la connexion à la boîte aux lettres

Le chemin du message n’explique pas encore comment Outlook trouve sa boîte aux lettres. Pour cela, Exchange utilise **Autodiscover**. Un client commence par l’identité de l’utilisateur et en déduit le point de terminaison de service approprié. Localement, les points de connexion de service Active Directory et le DNS peuvent être impliqués ; dans Exchange Online, les points de terminaison Microsoft 365 conduisent au service cloud ([Microsoft Learn: Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)).

Après la détermination du point de terminaison, l’accès moderne d’Outlook s’effectue via HTTPS, notamment avec MAPI over HTTP. Outlook sur le Web, Exchange ActiveSync et diverses API utilisent également HTTPS, mais chacun avec ses propres protocoles d’application et autorisations. Une connexion réussie au portail Microsoft 365 ne prouve donc pas automatiquement que le fonctionnement d’Autodiscover, du protocole Outlook ou de l’accès à la boîte aux lettres concernée ([Client Access protocol architecture](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access), [MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)).

En fonctionnement hybride, une décision supplémentaire s’ajoute : la boîte aux lettres est-elle locale ou en ligne ? Autodiscover et les attributs de destinataire doivent guider le client vers le bon côté. Ce n’est qu’ensuite que les fonctions interorganisationnelles telles que les informations de disponibilité ou les déplacements de boîtes aux lettres entrent en jeu. Cet ordre est traité étape par étape dans l’article [Exchange Hybrid](/kb/exchange-hybrid).

## Administration et autorisations

Une fois les chemins des données et des accès compris, la question est de savoir qui peut les modifier. Exchange utilise un contrôle d’accès basé sur les rôles. Les rôles contiennent des cmdlets et des paramètres, les groupes de rôles ou les attributions de rôles les associent à des administrateurs, et les étendues limitent leur champ d’application. Exchange Online et Exchange Server possèdent des concepts RBAC apparentés, mais des configurations distinctes ([Permissions in Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions), [Permissions in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)).

Au quotidien, cela signifie qu’un rôle d’administrateur Entra, un groupe de rôles Exchange et une autorisation de boîte aux lettres ne sont pas la même chose. **Full Access** permet d’ouvrir une boîte aux lettres, **Send As** d’envoyer en tant que destinataire et **Send on Behalf** d’envoyer de manière identifiable au nom de quelqu’un. Aucune de ces autorisations n’explique à elle seule si une application peut accéder via Microsoft Graph ou EWS ([Manage permissions for recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)).

La question d’expert n’est donc pas « L’utilisateur est-il administrateur ? », mais : quelle identité se connecte, quel rôle s’applique dans quelle organisation Exchange, quel objet est ciblé et quelle autorisation supplémentaire de boîte aux lettres ou d’application est vérifiée ?

## L’exploitation et le dépannage commencent du bon côté

Un diagnostic pertinent commence par trois informations : **utilisateur ou destinataire concerné, heure exacte et emplacement de la boîte aux lettres**. Le chemin est ensuite suivi dans l’ordre suivant : DNS ou Autodiscover, connexion, point de terminaison Exchange, résolution du destinataire, événement de transport et distribution dans la boîte aux lettres.

Pour Exchange Online, le suivi des messages et Microsoft 365 Service Health fournissent l’état du service visible par le client. Pour Exchange Server, s’y ajoutent les files d’attente locales, les journaux de suivi des messages, les ensembles d’intégrité, les journaux d’événements et les copies de bases de données. En fonctionnement hybride, les éléments probants des deux côtés sont réunis sur une chronologie commune ([Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq), [Server health and performance](https://learn.microsoft.com/en-us/exchange/server-health/server-health)).

Les articles approfondis contiennent chacun des blocs de diagnostic Windows/Unix appropriés et la documentation officielle des outils utilisés. Ici, la règle d’exploitation principale suffit : déterminer d’abord le lieu et le chemin, puis choisir l’outil.

## Stockage des données, disponibilité et récupération

La différence entre le cloud et l’exploitation en interne est particulièrement visible lors de la récupération. Exchange Server stocke les boîtes aux lettres dans des bases de données ESE avec des journaux de transactions. Les Database Availability Groups répliquent les copies de bases de données et permettent des activations sur d’autres serveurs. L’organisation reste toutefois responsable de la stratégie de sauvegarde, de la récupérabilité et de la dépendance à Active Directory ([Database availability groups](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups), [Backup, restore and disaster recovery](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)).

Exchange Online exploite la redondance des bases de données et du transport comme une composante du service. Les administrateurs du locataire travaillent plutôt avec les éléments supprimés, Single Item Recovery, la conservation, les conservations légales et, le cas échéant, des exigences de sauvegarde externes. La résilience du service Microsoft et une règle de conservation métier répondent à des questions différentes : l’une protège le service en cours d’exécution, l’autre détermine quels contenus sont conservés après suppression ou à des fins de conformité ([Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency), [Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)).

En fonctionnement hybride, les deux modèles de récupération doivent être documentés côte à côte. La synchronisation d’annuaire, les certificats, la configuration OAuth et les connecteurs sont également nécessaires pour rétablir la connexion après une panne. Ces configurations ne contiennent certes pas de contenu de boîtes aux lettres, mais elles déterminent si les deux organisations Exchange peuvent à nouveau fonctionner ensemble.

## Évolution technique

Exchange 4.0 est apparu en 1996 comme successeur des premières plateformes de messagerie Microsoft. Les premières versions utilisaient leur propre annuaire, MAPI et la famille de bases de données ESE ; les protocoles Internet ont progressivement gagné en importance. Avec Exchange 2000, Active Directory et SMTP sont devenus des composants centraux. Exchange 2007 a introduit des rôles de serveur marqués et l’Exchange Management Shell, Exchange 2010 le Database Availability Group ([Exchange Team: A brief history of time](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388), [Exchange Team: Exchange Server 2007 transport redesign](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)).

Parallèlement, Microsoft a développé ses offres Exchange hébergées vers Exchange Online. Ainsi, Hybrid n’est pas né comme un produit unique, mais comme la connexion de deux organisations Exchange indépendantes. Exchange Server Subscription Edition poursuit cette ligne de développement locale dans le Modern Lifecycle depuis 2025. Les informations concrètes sur les builds et les mises à jour sont vérifiées avant toute modification dans la documentation Microsoft continuellement mise à jour ([Exchange Server SE release notes](https://learn.microsoft.com/en-us/exchange/release-notes), [Exchange Server Subscription Edition lifecycle](https://learn.microsoft.com/en-us/lifecycle/products/exchange-server-subscription-edition)).

## Sources

- [Microsoft Learn – Exchange](https://learn.microsoft.com/en-us/exchange/)
- [Microsoft Learn – Exchange Online](https://learn.microsoft.com/en-us/exchange/exchange-online)
- [Microsoft Learn – Exchange Server](https://learn.microsoft.com/en-us/exchange/exchange-server)
- [Microsoft Learn – Exchange Server architecture](https://learn.microsoft.com/en-us/exchange/architecture/architecture)
- [Microsoft Learn – Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)
- [Microsoft Learn – Recipients](https://learn.microsoft.com/en-us/exchange/recipients/recipients)
- [Microsoft Learn – Mail flow and the transport pipeline](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)
- [Microsoft Learn – Trace an email message](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)
- [Microsoft Learn – Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq)
- [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)
- [Microsoft Learn – Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)
- [Microsoft Learn – Connectors on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors)
- [Microsoft Learn – Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)
- [Microsoft Learn – Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)
- [Microsoft Learn – Client Access protocol architecture](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access)
- [Microsoft Learn – MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)
- [Microsoft Learn – Active Directory in Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)
- [Microsoft Learn – Permissions in Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions)
- [Microsoft Learn – Permissions in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)
- [Microsoft Learn – Manage permissions for recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)
- [Microsoft Learn – Server health and performance](https://learn.microsoft.com/en-us/exchange/server-health/server-health)
- [Microsoft Learn – Database availability groups](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups)
- [Microsoft Learn – Backup, restore and disaster recovery](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)
- [Microsoft Service Assurance – Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)
- [Microsoft Purview – Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)
- [Exchange Team – A brief history of time](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388)
- [Exchange Team – Exchange Server 2007 transport redesign](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)
- [Microsoft Learn – Exchange Server SE release notes](https://learn.microsoft.com/en-us/exchange/release-notes)
- [Microsoft Lifecycle – Exchange Server Subscription Edition](https://learn.microsoft.com/en-us/lifecycle/products/exchange-server-subscription-edition)
