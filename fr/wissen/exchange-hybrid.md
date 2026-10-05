---
title: "Exchange Hybrid : identité, coexistence et exploitation"
blatt: "exchange-hybrid"
description: "Exchange Hybrid expliqué clairement : prérequis, synchronisation d’annuaires, autorité sur les destinataires, Hybrid Configuration Wizard, OAuth, relations d’organisation, Autodiscover, déplacements de boîtes aux lettres et exploitation."
fakten:
  - label: Objectif
    wert: Coexistence d’Exchange Server et d’Exchange Online
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Espace de noms partagé
    wert: Les boîtes aux lettres des deux côtés peuvent utiliser les mêmes domaines SMTP
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Synchronisation d’annuaires
    wert: Microsoft Entra Connect Sync ou Cloud Sync selon le modèle pris en charge
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites
  - label: Outil de configuration
    wert: Hybrid Configuration Wizard
    href: https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard
  - label: Configuration locale
    wert: Objet HybridConfiguration dans Active Directory
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Configuration cloud
    wert: Connecteurs, relations d’organisation et approbation OAuth
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment/deploy-hybrid
  - label: Modèle de destinataire
    wert: Remote Mailbox locale, boîte aux lettres Exchange Online dans le cloud
    href: https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox
  - label: Fonctions de coexistence
    wert: Libre/occupé, MailTips, archivage, recherche et déplacements de boîtes aux lettres selon la configuration
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Migration de boîtes aux lettres
    wert: Mailbox Replication Service et points de terminaison de migration
    href: https://learn.microsoft.com/en-us/exchange/mailbox-migration/migrate-mailboxes-across-tenants
  - label: Transport de messagerie
    wert: SMTP/TLS basé sur des certificats entre les deux organisations
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites
  - label: Gestion des destinataires
    wert: Exchange Management Tools ou transfert Cloud-SoA pris en charge
    href: https://learn.microsoft.com/en-us/exchange/manage-hybrid-exchange-recipients-with-management-tools
  - label: Diagnostic
    wert: Journal HCW, état de synchronisation Entra, configuration OAuth, d’organisation et de transport
    href: https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard
werbung:
  - tools
  - newsletter
ctaThemen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
translationSourceHash: c35b1646509133dd8975f96d030b2990225af14469d14a61e4fd16a4e2737f08
translationModel: gpt-5.6-terra
translatedAt: 2026-10-05T11:18:50.073Z
translationReview: automatic
---

# Exchange Hybrid : identité, coexistence et exploitation

**Exchange Hybrid** relie une organisation Exchange locale à Exchange Online. Les utilisateurs peuvent disposer de boîtes aux lettres des deux côtés tout en utilisant les mêmes domaines SMTP, un carnet d’adresses commun et certaines fonctions interorganisationnelles. Hybrid est donc plus qu’une paire de connecteurs : il relie les données d’annuaire, les destinataires, l’authentification, Autodiscover, les fonctions de calendrier, la migration et le transport de messagerie ([Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

Commencez par clarifier trois questions : **Où se trouve la boîte aux lettres ?** **Où son objet destinataire est-il géré ?** Et **quel service exécute l’opération demandée ?** Lorsque ces trois réponses sont établies, les nombreux composants Hybrid forment une chaîne compréhensible.

## Ce que Hybrid réunit pour les utilisateurs

Sans Hybrid, l’organisation Exchange locale et Exchange Online sont deux systèmes séparés. Hybrid crée une expérience utilisateur commune. Les boîtes aux lettres peuvent utiliser le même domaine SMTP principal. Les informations du carnet d’adresses sont synchronisées. Les requêtes libre/occupé et les MailTips peuvent fonctionner entre les organisations. Les boîtes aux lettres peuvent être déplacées avec des Remote Moves pris en charge ([Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

Ces fonctions ne partagent toutefois pas un unique magasin de données commun. Une boîte aux lettres locale reste dans une base de données ESE locale ; une boîte aux lettres cloud reste dans Exchange Online. Active Directory et Entra ID conservent chacun des objets d’annuaire. Les relations d’organisation et OAuth autorisent certaines requêtes au-delà de cette frontière. Les connecteurs SMTP transportent les messages. L’apparente « unique organisation Exchange » résulte de connexions coordonnées.

Pour les administrateurs, il en découle une règle importante : un flux de messagerie vert ne prouve pas que la fonction libre/occupé fonctionne, et une requête libre/occupé réussie ne prouve pas qu’un Remote Move est possible. Chaque fonction a son propre chemin et ses propres preuves.

## Les composants dans un ordre logique

Un déploiement Hybrid commence par ses prérequis, et non par l’assistant. L’organisation Exchange locale doit être dans un état pris en charge. Les noms publics, certificats, DNS et l’accessibilité HTTPS et SMTP doivent être corrects. Un tenant Microsoft 365 avec Exchange Online et une synchronisation d’annuaire prise en charge relient ensuite les identités ([Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)).

C’est sur cette base que repose le **Hybrid Configuration Wizard**, HCW. Il lit la configuration souhaitée, écrit un objet `HybridConfiguration` dans l’Active Directory local et configure les paramètres appropriés localement et dans Exchange Online. Cela peut inclure des relations d’organisation, OAuth, des connecteurs Intra-Organization et des connecteurs de transport ([Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard), [Create a hybrid deployment](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/deploy-hybrid)).

| Composant | Fonction principale | Premier point à vérifier en cas de problème |
|---|---|---|
| Active Directory | Attributs locaux des utilisateurs et d’Exchange | Objet, type de destinataire, adresses proxy et heure de modification |
| Synchronisation Entra | Transfère les attributs d’identité et de destinataire pris en charge | Erreurs d’exportation, état de synchronisation et objet cloud |
| Exchange Online | Boîte aux lettres cloud et configuration cloud | Type de destinataire, licence, état de la boîte aux lettres et RBAC |
| Configuration HCW | Coordonne les deux organisations Exchange | Journal HCW, paramètres sélectionnés et objets modifiés ultérieurement |
| Relation d’organisation et OAuth | Fonctions interorganisationnelles | URI cible, Autodiscover, certificats et flux de jetons |
| Connecteurs SMTP | Messages entre les deux côtés | Nom du certificat, hôte source/cible, TLS et Message Trace |

Le tableau montre également pourquoi « exécuter à nouveau HCW » n’est pas une réparation universelle. L’assistant peut réajuster les objets Hybrid documentés. Il ne corrige ni une zone DNS erronée, ni un chemin de pare-feu bloqué, ni un objet destinataire incorrectement géré.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-exchange-hybrid.svg?v=20260813" title="Interaktive Infografik: Exchange-Hybrid-Verbindungen für Verzeichnissync, Empfänger, HCW, OAuth, Frei-Gebucht, Mailboxverschiebung und SMTP" loading="lazy">
  <a href="/images/kb-interaktiv-exchange-hybrid.svg?v=20260813">Ouvrir directement le graphique interactif Exchange Hybrid</a>.
</iframe>

## Pile technologique : protocoles et outils d’administration

Hybrid n’est pas un processus serveur Exchange supplémentaire, mais une connexion entre des systèmes existants. Active Directory et Entra ID conservent les identités et les attributs de destinataire. La synchronisation Entra transfère les valeurs prises en charge. HTTPS transporte Autodiscover, libre/occupé, les appels de service protégés par OAuth et les déplacements de boîtes aux lettres. SMTP avec TLS transporte les messages. PowerShell, Exchange Admin Center et HCW gèrent les objets concernés ([Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites), [Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)).

Cette répartition impose aussi l’ordre de recherche des incidents. Une erreur d’objet se recherche dans l’annuaire et la synchronisation, un problème de calendrier dans le chemin HTTPS/OAuth, un problème de messagerie dans SMTP et les connecteurs. La boîte à outils reste ainsi liée à la fonction concernée.

## Synchronisation d’annuaires et autorité sur les destinataires

Dès que les plateformes sont connectées, l’origine des données des destinataires devient la question d’exploitation la plus importante. Dans les environnements Hybrid classiques, un utilisateur est créé dans l’Active Directory local. Les outils Exchange écrivent les attributs liés à la messagerie. Entra Connect synchronise l’objet vers le cloud, où Exchange Online fournit l’objet cloud correspondant et, le cas échéant, une boîte aux lettres ([Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)).

Une **Remote Mailbox** est un objet local doté de la messagerie qui pointe vers une boîte aux lettres Exchange Online. Des attributs tels que `remoteRoutingAddress`, `proxyAddresses` et le type de destinataire aident l’organisation locale à diriger les messages et l’administration vers le côté cloud. [`Enable-RemoteMailbox`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox) crée ou active cette représentation locale ; la boîte aux lettres cloud n’est créée que par la synchronisation et l’attribution de licence.

La question habituelle de l’administrateur est donc : où dois-je modifier cette valeur ? La question de l’expert est : quel système fait autorité pour **cet attribut précis**, et quelle exécution de synchronisation le transfère ? Un portail cloud peut afficher une valeur synchronisée sans pouvoir la modifier durablement.

Microsoft prend en charge des scénarios où seuls les Exchange Management Tools restent utilisés pour les attributs locaux des destinataires. Certaines environnements disposent également d’une procédure pour transférer la gestion des attributs Exchange vers le cloud. Ce sont des modèles d’exploitation différents avec leurs prérequis ; l’arrêt du dernier serveur ne déplace pas à lui seul l’autorité sur les données ([Manage recipients with Exchange Management Tools](https://learn.microsoft.com/en-us/exchange/manage-hybrid-exchange-recipients-with-management-tools), [Decommission after Source of Authority transfer](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/decommission-last-exchange-server)).

## Autodiscover et le cheminement du client

Une fois les destinataires correctement configurés, un client doit trouver l’emplacement de la boîte aux lettres. Autodiscover répond à cette question. Les points de terminaison Exchange locaux peuvent rediriger un client vers Exchange Online pour une boîte aux lettres cloud ; les points de terminaison cloud fournissent les paramètres de la boîte aux lettres en ligne ([Autodiscover in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)).

Un problème Hybrid Autodiscover se manifeste donc souvent par un mauvais emplacement : l’utilisateur peut se connecter en principe, mais arrive sur le point de terminaison local, reçoit une redirection inattendue ou obtient des paramètres pour une boîte aux lettres qui n’existe plus. DNS, SCP, répertoires virtuels, certificats et attributs de destinataire sont vérifiés dans cet ordre.

Ce n’est qu’après l’arrivée du client au bon service de boîte aux lettres que les questions de protocole et d’autorisations ont un sens. Le diagnostic reste ainsi compréhensible : d’abord trouver, puis se connecter, puis autoriser.

## Libre/occupé et autres fonctions interorganisationnelles

Un carnet d’adresses commun ne suffit pas pour les requêtes de calendrier. Libre/occupé nécessite des relations d’organisation, un Autodiscover accessible et une configuration d’approbation ou OAuth fonctionnelle. Exchange interroge les informations de l’autre côté au lieu de copier entièrement les données de calendrier dans son propre système ([Sharing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/sharing/sharing)).

Le même schéma de base s’applique aux autres fonctions Hybrid : un composant local formule une demande, la partie distante l’authentifie, autorise l’opération et fournit un résultat limité. Lors de la recherche d’incidents, la boîte aux lettres source, la boîte aux lettres cible, le sens et le point de terminaison sont donc consignés. « Libre/occupé ne fonctionne pas » est trop vague sans ces informations.

Pour les experts, les jetons et les URI cibles deviennent pertinents. Le HCW configure les relations interorganisationnelles, mais des changements de certificat, des modifications manuelles ou des points de terminaison obsolètes peuvent perturber l’exploitation ultérieure. La configuration des deux côtés est toujours exportée ensemble.

## OAuth entre les organisations Exchange

Une fois identifiées les requêtes interorganisationnelles effectuées, il est possible de situer leur authentification. Exchange peut utiliser OAuth afin qu’une organisation atteste un appel de service auprès de l’autre. Cela concerne des fonctions Hybrid comme la disponibilité interorganisationnelle et certaines opérations d’archivage, de recherche ou de migration ; l’utilisation exacte dépend de la version et de la configuration ([Configure OAuth authentication](https://learn.microsoft.com/en-us/exchange/configure-oauth-authentication-between-exchange-and-exchange-online-organizations-exchange-2013-help)).

Le flux de jetons ne remplace pas SMTP-TLS. OAuth protège les appels d’application, tandis que le transport de messagerie Hybrid utilise ses propres connecteurs et vérifications de certificats. Cette séparation évite le raccourci qui prête à confusion dans de nombreuses explications : on détermine d’abord la fonction, puis son protocole, et seulement ensuite l’authentification.

Pour les experts, AuthConfig, AuthServer, PartnerApplication, Intra-Organization Connector et Organization Relationship font partie d’une même vue de vérification. Un objet individuel peut être présent syntaxiquement alors que le certificat, le realm ou l’URI cible ne correspondent plus à la partie distante.

## Hybrid Modern Authentication est un sujet client distinct

**Hybrid Modern Authentication**, HMA, n’est abordée qu’à présent car elle n’explique ni le transport de messagerie ni la synchronisation des destinataires. HMA permet aux ressources Exchange locales et Skype for Business prises en charge d’utiliser Microsoft Entra ID pour l’authentification moderne des clients. Le client obtient un jeton Entra et l’utilise auprès du service local ([Hybrid modern authentication overview](https://learn.microsoft.com/en-us/microsoft-365/enterprise/hybrid-modern-auth-overview)).

HMA étend ainsi l’accès client par une dépendance cloud. L’accessibilité Entra, les URL publiées, les Service Principal Names enregistrés et la configuration Exchange locale doivent correspondre. Un flux de messagerie Hybrid fonctionnel ne dit rien sur ce chemin de jetons.

Les experts traitent donc HMA dans un runbook distinct avec les versions prises en charge, les exclusions, les groupes de déploiement et un plan de repli. La fonction n’est pas ajoutée de manière accessoire à une option de routage.

## Déplacements de boîtes aux lettres

La coexistence est souvent mise en place afin de déplacer progressivement les boîtes aux lettres. Un Remote Move copie les données de boîte aux lettres via le Mailbox Replication Service, synchronise les modifications, puis bascule la boîte aux lettres de manière contrôlée vers la cible. Les attributs de destinataire et le routage sont conservés ou adaptés au cours du processus ([Move mailboxes between on-premises and Exchange Online](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes)).

Pour les administrateurs avancés, le processus se compose de la préparation, du démarrage, de la synchronisation, de la finalisation et du contrôle ultérieur. Avant la finalisation, le volume de données, les éléments défectueux, les délégations, les archives, l’accès client et le flux de messagerie sont vérifiés. Après la finalisation, Autodiscover, la licence, l’adresse cible et l’objet Remote Mailbox local doivent correspondre.

Les experts planifient les tailles de lots, le débit réseau, la limitation MRS, les limites d’éléments défectueux, les objets volumineux et la migration inverse. La seule valeur de progression technique ne constitue pas une réception ; l’accès utilisateur, les délégations, les clients mobiles et les fonctions interorganisationnelles en font également partie.

## Le flux de messagerie Hybrid reste un chemin distinct

Hybrid nécessite SMTP entre l’organisation locale et Exchange Online. Ce chemin de messagerie utilise des connecteurs, TLS et des certificats. Il est suffisamment important pour faire l’objet d’un article dédié, car la messagerie Internet, Centralized Mail Transport, les passerelles de messagerie et les domaines partagés forment plusieurs variantes ([Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)).

L’article [Flux de messagerie Hybrid](/kb/hybrid-mailfluss) part d’un message concret et suit chaque saut. C’est là que sont comparés Centralized Mail Transport, l’IP de sortie, l’emplacement du filtrage et les files d’attente supplémentaires. Cet article se concentre sur l’identité et la coexistence.

## Sécurité et exploitation

Hybrid étend les systèmes accessibles. Les points de terminaison HTTPS et SMTP publics, les certificats, la synchronisation Entra, les comptes privilégiés et les objets d’approbation interorganisationnels doivent être inventoriés ensemble. Le HCW nécessite des droits étendus des deux côtés ; son utilisation et ses journaux doivent être protégés et archivés de manière traçable ([Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)).

Au quotidien, chaque fonction Hybrid doit avoir un responsable et un test : synchronisation des destinataires, libre/occupé dans les deux sens, Remote Move, Autodiscover ainsi que SMTP dans les deux sens. Un test synthétique régulier détecte les certificats expirés ou les points de terminaison modifiés silencieusement plus tôt qu’un projet de migration.

En cas d’incident, une chronologie commune est utile. Les événements de synchronisation Entra, le journal HCW, les journaux d’événements Exchange, les tests OAuth, le suivi des messages et Message Trace ne sont pas collectés au hasard, mais associés à la fonction concernée. Cela raccourcit le diagnostic et évite qu’un test réussi d’une autre fonction soit interprété comme une preuve.

## Sauvegarde, reconstruction et retrait

Les données de boîte aux lettres sont protégées du côté où elles se trouvent : les bases de données locales avec une récupération locale, les boîtes aux lettres cloud avec les fonctions Exchange Online et Purview. La connexion doit en outre pouvoir être rétablie. Cela inclut l’objet HybridConfiguration local, les certificats et clés privées, la configuration des connecteurs et de l’organisation, les règles de synchronisation Entra ainsi que les décisions HCW documentées.

Une reconstruction commence par l’identité et la résolution de noms, suivies de l’accessibilité HTTPS et SMTP, de la configuration HCW et des tests fonctionnels. L’assistant peut recréer la configuration, mais sans certificats, DNS et objets destinataires appropriés, aucun système global fonctionnel ne peut être créé.

Lors du retrait, il faut d’abord déterminer quelles fonctions Hybrid sont encore utilisées. Microsoft distingue un serveur restant, les outils de gestion seuls et le transfert de la gestion des attributs Exchange vers le cloud. Ce n’est qu’après cette décision que les connecteurs, relations d’organisation, points de terminaison et serveurs sont supprimés de manière contrôlée ([Manage recipients with Exchange Management Tools](https://learn.microsoft.com/en-us/exchange/manage-hybrid-exchange-recipients-with-management-tools), [Decommission after Source of Authority transfer](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/decommission-last-exchange-server)).

## Évolution technique et limites

Hybrid est né avec Exchange Online comme moyen d’étendre de manière contrôlée des organisations locales vers le service cloud. Les générations précédentes reposaient davantage sur des Federation Trusts ; les versions Exchange plus récentes et les processus HCW utilisent OAuth et les connecteurs Intra-Organization pour de nombreuses fonctions interorganisationnelles ([Create a hybrid deployment](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/deploy-hybrid)).

Le modèle est puissant car il permet la migration et la coexistence durable. Il est exigeant car les deux organisations Exchange et leur connexion doivent être exploitées. Après une migration, ceux qui n’ont plus de boîtes aux lettres locales devraient donc décider consciemment quelle fonction de gestion ou de coexistence justifie encore Hybrid.

## Sources

- [Microsoft Learn – Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)
- [Microsoft Learn – Active Directory in Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)
- [Microsoft Learn – Recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/recipients-in-exchange-online)
- [Microsoft Learn – Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)
- [Microsoft Learn – Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)
- [Microsoft Learn – Create a hybrid deployment](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/deploy-hybrid)
- [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox)
- [Microsoft Learn – Manage recipients with Exchange Management Tools](https://learn.microsoft.com/en-us/exchange/manage-hybrid-exchange-recipients-with-management-tools)
- [Microsoft Learn – Decommission after Source of Authority transfer](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/decommission-last-exchange-server)
- [Microsoft Learn – Autodiscover service](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)
- [Microsoft Learn – Sharing in Exchange](https://learn.microsoft.com/en-us/exchange/sharing/sharing)
- [Microsoft Learn – Configure OAuth authentication](https://learn.microsoft.com/en-us/exchange/configure-oauth-authentication-between-exchange-and-exchange-online-organizations-exchange-2013-help)
- [Microsoft Learn – Hybrid modern authentication overview](https://learn.microsoft.com/en-us/microsoft-365/enterprise/hybrid-modern-auth-overview)
- [Microsoft Learn – Move mailboxes between on-premises and Exchange Online](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes)
- [Microsoft Learn – Cross-tenant mailbox migration](https://learn.microsoft.com/en-us/exchange/mailbox-migration/migrate-mailboxes-across-tenants)
