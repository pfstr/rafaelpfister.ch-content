---
title: "Exchange Online : architecture, flux de messagerie et exploitation"
blatt: "exchange-online"
description: "Exchange Online pour les administrateurs de messagerie : modèle de tenant et de destinataires, transport EOP, connecteurs, accès client, PowerShell et Graph, Message Trace, conservation, sécurité et restauration."
fakten:
  - label: Rôle du produit
    wert: Service cloud de messagerie, de calendrier et d’annuaire
    href: https://learn.microsoft.com/en-us/exchange/exchange-online
  - label: Plateforme
    wert: Microsoft 365
    href: https://learn.microsoft.com/en-us/office365/servicedescriptions/exchange-online-service-description/exchange-online-service-description
  - label: Réception du courrier
    wert: Exchange Online Protection et SMTP
    href: https://learn.microsoft.com/en-us/defender-office-365/eop-about
  - label: Destinataires
    wert: Boîtes aux lettres, groupes, contacts, utilisateurs de messagerie et ressources
    href: https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/recipients-in-exchange-online
  - label: Domaines
    wert: Authoritative ou Internal Relay
    href: https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains
  - label: Routage
    wert: Connecteurs entrants et sortants, règles et MX
    href: https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail
  - label: Accès client
    wert: Outlook, Outlook sur le web, ActiveSync et cas particuliers IMAP/POP
    href: https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/clients-and-mobile-in-exchange-online
  - label: Identité
    wert: Microsoft Entra ID et authentification moderne
    href: https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/enable-or-disable-modern-auth-in-exchange-online
  - label: Administration
    wert: Centre d’administration Exchange et Exchange Online PowerShell
    href: https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell
  - label: API
    wert: Microsoft Graph pour les fonctions de messagerie, de calendrier et d’administration
    href: https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview
  - label: Diagnostic
    wert: Message Trace, rapports et Service Health
    href: https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message
  - label: Conservation
    wert: Recoverable Items, Retention et Holds
    href: https://learn.microsoft.com/en-us/purview/retention-policies-exchange
werbung:
  - tools
  - newsletter
ctaThemen:
  - microsoft-365-exchange
  - smtp-mailflow
translationSourceHash: 5965bf4a9447505ffbe8b9a5d00c6f1abf629dfbde070d9c7700d9fdc9e4b603
translationModel: gpt-5.6-terra
translatedAt: 2026-10-05T11:27:43.659Z
translationReview: automatic
---

# Exchange Online : architecture, flux de messagerie et exploitation

**Exchange Online** est le service Exchange exploité par Microsoft dans Microsoft 365. Il fournit des boîtes aux lettres, calendriers, contacts, groupes, le transport SMTP et des fonctions d’administration. L’administrateur du tenant décide des destinataires, domaines, connecteurs, règles, autorisations et de la conservation. Microsoft exploite en revanche les serveurs de boîtes aux lettres, les copies de bases de données, les files d’attente internes, les correctifs et les opérations de basculement ([Exchange Online service description](https://learn.microsoft.com/en-us/office365/servicedescriptions/exchange-online-service-description/exchange-online-service-description), [Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)).

Ainsi, Exchange Online ressemble fonctionnellement à son propre système Exchange, mais pas sur le plan de l’exploitation. Un administrateur local peut examiner un fichier de file d’attente ou activer une copie de base de données. Dans Exchange Online, il voit à la place les événements, états et objets de configuration fournis par le service. La capacité la plus importante consiste donc à associer une plainte d’utilisateur à un chemin clair : identité, accès client, objet destinataire, transport, filtrage, remise ou conservation.

## Du tenant à la boîte aux lettres

Le tenant constitue le cadre organisationnel. Exchange Online y gère des destinataires à capacité de messagerie : boîtes aux lettres utilisateur et partagées, boîtes aux lettres de salle et d’équipement, listes de distribution, groupes Microsoft 365, contacts et utilisateurs de messagerie. Le type de destinataire détermine si des données sont stockées, comment la remise s’effectue et quelles autorisations sont disponibles ([Recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/recipients-in-exchange-online)).

Un compte utilisateur dans Microsoft Entra ID et une boîte aux lettres Exchange sont liés, mais ne constituent pas le même objet. L’attribution de licences peut déclencher le provisionnement d’une boîte aux lettres. Exchange ajoute alors des attributs et services liés à la messagerie. Lorsqu’un administrateur retire une licence ou supprime un compte, différents délais de conservation et de suppression s’appliquent. Pour l’exploitation et le départ des collaborateurs, le cycle de vie de l’identité, celui de la boîte aux lettres et la conservation de conformité doivent donc être planifiés conjointement ([Delete or restore user mailboxes](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/delete-or-restore-mailboxes), [Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)).

Pour les experts, l’origine des attributs devient importante. Dans un tenant entièrement cloud, les propriétés Exchange sont gérées en ligne. Avec des identités synchronisées, l’environnement local peut rester la source faisant autorité pour certains attributs de destinataires. Une valeur apparaît alors dans Exchange Online, mais doit être modifiée localement puis synchronisée à nouveau. Ce modèle appartient à l’article [Exchange Hybrid](/kb/exchange-hybrid), car il n’existe pas sans synchronisation d’annuaire.

## Comment un message entrant atteint la boîte aux lettres

Une fois le destinataire compris, le chemin du courrier peut être suivi. Le MX public d’un domaine pointe normalement vers Exchange Online Protection, EOP. EOP accepte la connexion SMTP, évalue l’expéditeur et le message, applique des règles de protection et de transport, puis transmet un message autorisé à Exchange Online. Pour les destinataires locaux, la remise dans la boîte aux lettres suit ensuite ([Exchange Online Protection overview](https://learn.microsoft.com/en-us/defender-office-365/eop-about), [Mail flow in EOP](https://learn.microsoft.com/en-us/defender-office-365/eop-mail-flow)).

Le domaine accepté (**Accepted Domain**) définit la façon dont Exchange Online traite le domaine destinataire. Avec `Authoritative`, le service attend tous les destinataires valides dans sa propre organisation et rejette les adresses inconnues. `Internal Relay` permet de transférer des destinataires inconnus vers un autre système. Ce paramètre n’est pertinent que si le prochain saut et la résolution des destinataires sont planifiés de manière fiable ; sinon, des non-remises ou des boucles surviennent ([Manage accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)).

Un message interne ne reste pas automatiquement « sur le même serveur ». Exchange Online résout l’expéditeur et le destinataire, vérifie les règles et stratégies de protection, puis enregistre des événements de transport. Cette chaîne d’événements est déterminante pour l’administrateur : `Delivered` signifie que le service a remis le message à sa destination ; `Filtered`, `Failed`, `Pending` ou `Expanded` décrivent d’autres étapes. Message Trace rend ces étapes visibles, mais ne remplace pas la vérification de la boîte aux lettres de destination ou d’une règle en aval ([Trace an email message](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message), [Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq)).

## Messages sortants et connecteurs

Pour les messages sortants, il faut d’abord déterminer si Exchange Online envoie directement au système de destination ou utilise un connecteur sortant configuré. Un connecteur peut diriger les messages vers sa propre infrastructure, vers un partenaire ou vers une passerelle de messagerie. La sélection repose notamment sur le domaine destinataire, les conditions du connecteur et les règles de transport ([Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)).

Les connecteurs entrants décrivent inversement les conditions dans lesquelles Exchange Online fait confiance à un système expéditeur. Les critères typiques sont l’adresse IP source ou un certificat TLS. Ces informations sont pertinentes pour la sécurité : une plage IP trop large ou un certificat vérifié de façon imprécise peut faire paraître du trafic tiers comme du trafic interne ou partenaire.

Lorsqu’une passerelle de messagerie externe est placée devant EOP, Microsoft voit d’abord l’adresse IP de la passerelle. **Enhanced Filtering for Connectors** peut intégrer les informations sur le saut d’origine dans l’évaluation du filtrage. Cette fonction n’est pas un « commutateur de filtre antispam » général, mais doit correspondre au chemin réel, aux connecteurs et aux adresses IP ignorées ([Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)).

La question des experts est ici la suivante : quelle contrepartie a réellement accepté un message, quelle identité a été vérifiée pour le connecteur et à quel saut a eu lieu le dernier filtrage de contenu ? Ces trois réponses doivent figurer dans chaque diagramme de flux de messagerie.

## Structure technique du point de vue de l’administrateur

Exchange Online ne publie pas de liste de serveurs qu’un administrateur de tenant gère comme une batterie locale. Le service possède néanmoins des composants techniques clairement identifiables. Ils sont visibles via des protocoles et des interfaces d’administration.

| Composant | Tâche | Ce que voit l’administrateur du tenant |
|---|---|---|
| Exchange Online Protection | Acceptation SMTP, antimalware, antispam et traitement du transport | Quarantaine, stratégies, rapports et Message Trace |
| Transport Exchange | Résolution des destinataires, règles, routage et remise | Connecteurs, domaines acceptés, règles et événements |
| Service de boîtes aux lettres | Stockage des e-mails, calendriers, contacts et dossiers | Objets de boîte aux lettres, quotas, autorisations et accès client |
| Microsoft Entra ID | Identités des utilisateurs, groupes, applications et connexions | Comptes, rôles, Conditional Access et inscriptions d’applications |
| Exchange Online PowerShell | Administration spécifique à Exchange | Cmdlets, RBAC et modifications auditables |
| Microsoft Graph | API REST pour les applications et l’automatisation | Autorisations OAuth, ressources et limitation du débit |

La pile technologique périphérique se compose donc principalement de SMTP et TLS pour le transport de messagerie, ainsi que de HTTPS, OAuth, PowerShell et REST pour l’accès client et administratif. Les détails internes d’implémentation ne sont pertinents pour le client que dans la mesure où Microsoft les documente comme comportement de service, limite ou interface de diagnostic ([About the Exchange Online PowerShell module](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell-v2), [Microsoft Graph mail API](https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1056" src="/images/kb-interaktiv-exchange-online.svg?v=20260813" title="Interaktive Infografik: Exchange-Online-Pfad von DNS und EOP über Transport und Postfach bis Entra, PowerShell, Graph und Betriebsnachweis" loading="lazy">
  <a href="/images/kb-interaktiv-exchange-online.svg?v=20260813">Ouvrir directement le graphique interactif d’Exchange Online</a>.
</iframe>

## Accès client et authentification moderne

Le transport de messagerie se termine dans la boîte aux lettres ; les utilisateurs y accèdent ensuite via des protocoles client. Outlook, Outlook sur le web, les clients mobiles et les applications utilisent des points de terminaison basés sur HTTPS. Autodiscover aide les clients à trouver le service approprié. L’authentification s’effectue via Microsoft Entra ID, tandis qu’Exchange vérifie l’autorisation au niveau de la boîte aux lettres ([Clients and mobile in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/clients-and-mobile-in-exchange-online), [Modern authentication in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/enable-or-disable-modern-auth-in-exchange-online)).

Cela distingue deux erreurs souvent confondues. Si l’authentification échoue dans Entra, le client n’atteint souvent pas Exchange du tout. Si le jeton est valide, Exchange peut néanmoins refuser l’accès en raison d’un rôle manquant, d’une autorisation de boîte aux lettres, d’une stratégie client ou d’une boîte aux lettres cible incorrecte. Les journaux de connexion et le diagnostic Exchange doivent donc être considérés ensemble dans le temps.

Les applications accèdent de préférence via Microsoft Graph ou des interfaces Exchange prises en charge. Une autorisation d’application Graph peut être étendue ; Exchange RBAC for Applications peut restreindre davantage l’étendue des boîtes aux lettres accessibles. Un jeton OAuth valide n’est donc que la première étape. Le service de ressources vérifie ensuite quelle action est autorisée sur quelle boîte aux lettres ([Role Based Access Control for Applications](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac)).

## Suivre les autorisations et les modifications

Exchange Online possède ses propres rôles d’administration. Les rôles Entra peuvent permettre l’accès à l’administration Exchange, mais les cmdlets Exchange effectives et leur étendue sont déterminées par Exchange-RBAC ([Permissions in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)).

Il existe également des autorisations de boîte aux lettres telles que Full Access, Send As et Send on Behalf. Elles contrôlent des actions différentes et ne doivent pas être inventoriées comme un droit de délégation commun. Pour les applications, s’ajoutent les rôles OAuth et les rôles d’application Exchange ([Manage permissions for recipients](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)).

Pour les experts, l’origine d’une modification est aussi importante que l’état final. Les journaux d’audit, les journaux de connexion Entra et les exportations de configuration indiquent qui a modifié une règle, un connecteur ou une autorisation. Une exportation nocturne des objets centraux du flux de messagerie facilite les comparaisons, mais ne remplace pas une source d’audit protégée.

## Diagnostic : d’abord DNS, puis événements de transport

Une analyse du flux de messagerie commence en dehors du tenant. Le MX indique quel système accepte le courrier Internet. Ensuite, Message Trace permet de vérifier si Exchange Online a vu le message précis et comment il l’a traité.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für MX- und Autodiscover-Abfrage">
    <button type="button" role="tab" data-os-tab="windows" aria-selected="true">Windows</button>
    <button type="button" role="tab" data-os-tab="unix" aria-selected="false">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.com
Resolve-DnsName autodiscover.example.com
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig MX example.com
dig autodiscover.example.com
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) et [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) montrent la publication et la résolution. Ils n’indiquent pas encore si EOP a accepté le message ou si une boîte aux lettres l’a reçu.

Pour l’étape suivante, une période restreinte est choisie avec l’expéditeur et le destinataire. Le même Exchange Online PowerShell fonctionne sous Windows et avec `pwsh` sur les systèmes Unix pris en charge.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Exchange Online PowerShell">
    <button type="button" role="tab" data-os-tab="windows" aria-selected="true">Windows</button>
    <button type="button" role="tab" data-os-tab="unix" aria-selected="false">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Connect-ExchangeOnline
Get-MessageTraceV2 -SenderAddress sender@example.net `
  -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date)
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```powershell
pwsh
Connect-ExchangeOnline
Get-MessageTraceV2 -SenderAddress sender@example.net `
  -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date)
```

  </div>
</div>

[`Connect-ExchangeOnline`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/connect-exchangeonline) établit la session d’administration authentifiée. [`Get-MessageTraceV2`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2) recherche les événements de transport ; [`Get-Date`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-date) limite la fenêtre temporelle. Pour les tendances, des rapports sont disponibles, et pour les incidents Microsoft, Service Health. Un seul indicateur vert ne répond pas aux trois questions ([Exchange Online monitoring](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-exchange-monitoring?view=o365-worldwide)).

## Conservation, suppression et restauration

Microsoft protège le service en fonctionnement avec plusieurs copies de bases de données, Shadow Redundancy et Safety Net. Ces mécanismes servent à la disponibilité et à l’intégrité des données du service. Ils ne constituent pas l’interface utilisateur permettant de restaurer un message supprimé accidentellement ([Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)).

Pour les cas utilisateur et de conformité, d’autres fonctions s’appliquent : Deleted Item Retention, Recoverable Items, Single Item Recovery, Retention Policies et Holds. Leurs effets se chevauchent, mais leurs objectifs diffèrent. Une règle de conservation peut protéger des contenus contre une suppression définitive ; elle ne fournit pas automatiquement une sauvegarde séparée, indépendante du tenant, avec un moment de restauration librement sélectionnable ([Recoverable Items folder](https://learn.microsoft.com/en-us/exchange/security-and-compliance/recoverable-items-folder/recoverable-items-folder), [Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)).

Un concept de récupération fiable précise donc quels événements sont couverts par la résilience du service Microsoft, quels contenus peuvent être récupérés via la conservation Exchange ou Purview et pour quelles exigences une copie indépendante est nécessaire. Les tests de restauration doivent utiliser des cas concrets : message individuel, dossier, boîte aux lettres après suppression d’utilisateur, élément conservé pour des raisons légales et incident à l’échelle du tenant.

## Sécurité et limites typiques

Exchange Online relie plusieurs domaines de sécurité : courrier Internet, EOP, configuration du tenant, connexion Entra, droits de boîte aux lettres et applications. L’efficacité de la protection dépend de la correspondance entre le chemin réel des messages et des connexions, et la configuration.

Pour le flux de messagerie, cela signifie que MX, identité du connecteur, Enhanced Filtering, SPF/DKIM/DMARC et règles de transport doivent être vérifiés comme une chaîne. Pour l’accès client, l’authentification moderne, Conditional Access, Exchange-RBAC et les autorisations de boîte aux lettres sont des contrôles distincts. Pour les applications, s’ajoutent le consentement OAuth et l’étendue autorisée des boîtes aux lettres.

La question administrative plus approfondie reste la même dans chaque cas : quel système a pris la décision, quelles données d’entrée a-t-il vues et où le résultat est-il journalisé ? Sans ces trois informations, même une stratégie formellement correcte reste difficile à vérifier.

## Évolution technique et compromis délibérés

Exchange Online est issu des offres Exchange hébergées de Microsoft et a repris de nombreux concepts du produit serveur : destinataires, bases de données de boîtes aux lettres, transport, DAG, Shadow Redundancy et Safety Net. Le service automatise l’exploitation de cette infrastructure et fournit aux administrateurs de tenant un niveau d’administration plus élevé ([Exchange Team: 20 years ago](https://techcommunity.microsoft.com/blog/exchange/20-years-ago-in-a-galaxy-far-away8230/604456), [Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)).

Le gain réside dans l’exploitation externalisée de la plateforme, l’intégration mondiale des services et des interfaces d’administration standardisées. Le prix est un accès moindre aux serveurs individuels, aux files d’attente et aux copies de bases de données, ainsi qu’une dépendance accrue aux fonctions de diagnostic, d’exportation et de récupération publiées. La tâche des experts n’est donc pas de deviner la topologie interne invisible, mais d’utiliser pleinement les contrôles du tenant et les signaux du service documentés.

## Sources

- [Microsoft Learn – Exchange Online](https://learn.microsoft.com/en-us/exchange/exchange-online)
- [Microsoft – Exchange Online service description](https://learn.microsoft.com/en-us/office365/servicedescriptions/exchange-online-service-description/exchange-online-service-description)
- [Microsoft Defender – Exchange Online Protection overview](https://learn.microsoft.com/en-us/defender-office-365/eop-about)
- [Microsoft Defender – Mail flow in EOP](https://learn.microsoft.com/en-us/defender-office-365/eop-mail-flow)
- [Microsoft Learn – Recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/recipients-in-exchange-online)
- [Microsoft Learn – Delete or restore user mailboxes](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/delete-or-restore-mailboxes)
- [Microsoft Learn – Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)
- [Microsoft Learn – Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)
- [Microsoft Learn – Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)
- [Microsoft Learn – Trace an email message](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)
- [Microsoft Learn – Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq)
- [Microsoft Learn – Clients and mobile in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/clients-and-mobile-in-exchange-online)
- [Microsoft Learn – Modern authentication in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/enable-or-disable-modern-auth-in-exchange-online)
- [Microsoft Learn – Exchange Online PowerShell](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell)
- [Microsoft Learn – About the Exchange Online PowerShell module](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell-v2)
- [Microsoft Graph – Mail API overview](https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview)
- [Microsoft Learn – RBAC for Applications](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac)
- [Microsoft Learn – Permissions in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)
- [Microsoft Learn – Manage permissions for recipients](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)
- [Microsoft Service Assurance – Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)
- [Microsoft Learn – Recoverable Items folder](https://learn.microsoft.com/en-us/exchange/security-and-compliance/recoverable-items-folder/recoverable-items-folder)
- [Microsoft Purview – Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)
- [Microsoft Learn – Exchange Online monitoring](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-exchange-monitoring?view=o365-worldwide)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility)
- [Microsoft Learn – Connect-ExchangeOnline](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/connect-exchangeonline)
- [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2)
- [Microsoft Learn – Get-Date](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-date)
- [Exchange Team – 20 years ago](https://techcommunity.microsoft.com/blog/exchange/20-years-ago-in-a-galaxy-far-away8230/604456)
