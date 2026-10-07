---
title: "Flux de messagerie hybride : routage entre Exchange Online et l’environnement local"
blatt: "hybrid-mailfluss"
description: "Flux de messagerie hybride pas à pas : distribution directe et centralisée, connecteurs, certificats TLS, domaines distants, domaines SMTP partagés, passerelles de messagerie, Message Trace et Message Tracking."
fakten:
  - label: Tâche
    wert: Routage SMTP entre Exchange Online et l’organisation Exchange locale
    href: https://learn.microsoft.com/en-us/exchange/transport-routing
  - label: Configuration
    wert: Hybrid Configuration Wizard crée et maintient la configuration de transport
    href: https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard
  - label: Transport
    wert: SMTP sur TCP 25 avec TLS
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites
  - label: Vérification du pair
    wert: Nom du certificat et conditions du connecteur
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment/hybrid-mail-flow
  - label: Connecteurs cloud
    wert: Connecteurs entrants et sortants
    href: https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail
  - label: Connecteurs locaux
    wert: Connecteurs de réception et d’envoi
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors
  - label: Routage des destinataires
    wert: Boîte aux lettres distante, Target Address et domaine de coexistence
    href: https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox
  - label: Routage standard
    wert: Les organisations cloud et locales peuvent chacune envoyer directement des e-mails Internet
    href: https://learn.microsoft.com/en-us/exchange/transport-routing
  - label: Centralized Mail Transport
    wert: Les e-mails Internet d’Exchange Online transitent par l’organisation locale
    href: https://learn.microsoft.com/en-us/exchange/transport-routing
  - label: Passerelles externes
    wert: Chaîne supplémentaire de connecteurs et de filtrage avant ou après Microsoft 365
    href: https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud
  - label: Diagnostic cloud
    wert: Message Trace et validation des connecteurs
    href: https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message
  - label: Diagnostic local
    wert: Message Tracking, files d’attente et journaux de protocole
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking
werbung:
  - tools
  - newsletter
ctaThemen:
  - exchange-onprem-hybrid
  - smtp-mailflow
translationSourceHash: 442ec532c64e034403d3449ffe1b46d4b6de05a2337d3c3210da2d83cb5f618f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-07T11:02:42.718Z
translationReview: automatic
---

# Flux de messagerie hybride : routage entre Exchange Online et l’environnement local

Le **flux de messagerie hybride** est le chemin SMTP entre une organisation Exchange locale et Exchange Online. Il permet aux boîtes aux lettres des deux côtés d’utiliser le même domaine SMTP tout en garantissant que les messages arrivent au bon endroit. Le Hybrid Configuration Wizard configure à cette fin les connecteurs et les paramètres TLS ([Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing), [Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)).

Le flux de messagerie n’est qu’une partie d’Exchange Hybrid. La synchronisation d’annuaire, les informations de disponibilité, OAuth et les déplacements de boîtes aux lettres utilisent d’autres chemins. Cet article se limite donc délibérément à une seule question : **quels sauts SMTP un message concret parcourt-il, et quelle décision est prise à chaque saut ?**

La **pile de protocoles** reste claire : DNS désigne les destinations accessibles publiquement, SMTP sur TCP 25 transporte le message, TLS protège et identifie la connexion, et les connecteurs Exchange déterminent quel pair est utilisé pour quel domaine. Les objets destinataires fournissent l’adresse de routage ; Message Trace et les journaux de suivi locaux indiquent ensuite ce que chaque organisation a fait du message ([Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing), [Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)).

## Le modèle de base : deux organisations Exchange, un espace d’adressage

Un environnement hybride comporte au moins deux organisations de transport. L’organisation Exchange locale connaît les boîtes aux lettres locales et les objets de boîtes aux lettres distantes. Exchange Online connaît les boîtes aux lettres cloud et les représentations synchronisées des destinataires locaux. Les deux côtés peuvent utiliser le même domaine principal, tel que `example.com` ([Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

Pour qu’un message ne se termine pas au mauvais endroit, chaque côté a besoin d’une indication sur l’emplacement réel du destinataire. Pour une boîte aux lettres cloud, l’objet de boîte aux lettres distante local contient une adresse de routage distante dans le domaine de coexistence, généralement `tenant.mail.onmicrosoft.com`. Inversement, Exchange Online connaît les destinataires locaux synchronisés en tant qu’objets à extension messagerie ([Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox)).

Le déroulement normal est donc simple :

1. La première organisation Exchange accepte le message.
2. Elle résout le destinataire dans son annuaire.
3. L’objet destinataire indique si la boîte aux lettres est locale ou se trouve de l’autre côté.
4. Le connecteur hybride approprié envoie le message via SMTP/TLS à l’autre organisation.
5. Le destinataire y est à nouveau résolu et le message est distribué.

Les experts vérifient en outre si des règles de transport modifient l’itinéraire, si une passerelle est insérée et quelle priorité de domaine ou de connecteur explique le saut suivant sélectionné.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-hybrid-mailfluss.svg?v=20260813" title="Interaktive Infografik: direkter und zentraler Hybrid-Mailfluss zwischen Internet, Exchange Online, Exchange On-Premises und Mail-Gateway" loading="lazy">
  <a href="/images/kb-interaktiv-hybrid-mailfluss.svg?v=20260813">Ouvrir directement le graphique interactif du flux de messagerie hybride</a>.
</iframe>

## L’itinéraire standard sans transport Internet centralisé

Dans le modèle décentralisé habituel, chaque côté envoie lui-même ses e-mails Internet. Une boîte aux lettres locale utilise l’organisation de transport Exchange locale pour les e-mails Internet sortants. Une boîte aux lettres cloud envoie via Exchange Online Protection. Seuls les messages entre boîtes aux lettres locales et cloud passent par les connecteurs hybrides ([Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing)).

Les e-mails Internet entrants suivent l’enregistrement MX publié. Si le MX pointe vers Exchange Online, EOP accepte le message en premier. Pour une boîte aux lettres cloud, Exchange Online distribue localement ; pour un destinataire local synchronisé, il utilise le connecteur sortant hybride. Si le MX pointe au contraire vers l’environnement local ou une passerelle en amont, la première décision concernant le destinataire est prise là-bas.

Ce modèle maintient les chemins Internet courts, mais entraîne plusieurs IP de sortie et emplacements de filtrage possibles. SPF, DKIM, DMARC, les listes d’autorisation et les règles partenaires doivent prendre en compte les deux chemins de sortie. Il ne s’agit pas d’une erreur du modèle hybride, mais d’une conséquence de la distribution du courrier.

## Situer délibérément Centralized Mail Transport

**Centralized Mail Transport**, CMT, modifie précisément ce chemin sortant. Les messages envoyés depuis des boîtes aux lettres Exchange Online vers Internet sont d’abord envoyés à l’organisation Exchange locale. Ils ne quittent l’organisation qu’à partir de là. Le côté local peut ainsi continuer à utiliser des règles de transport centralisées, des appliances ou des IP de sortie fixes ([Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing)).

L’avantage est un contrôle de sortie commun. Le prix à payer est l’ajout de sauts et de dépendances. Si le transport local ou sa connectivité Internet tombe en panne, les e-mails cloud sortants sont désormais également affectés. La latence, l’emplacement de la file d’attente, l’IP de sortie et le lieu du dernier filtrage changent.

Pour les administrateurs avancés, la décision n’est donc pas « CMT activé ou désactivé », mais : quelle politique concrète exige le saut local, quelle capacité doit-il prendre en charge et comment le routage est-il assuré en cas de panne ? Les experts documentent également la protection contre les boucles, les exigences TLS, la priorité des connecteurs et la preuve que chaque message prévu emprunte effectivement le chemin centralisé.

Le sujet de CMT est ainsi clos. L’authentification des clients Outlook ou Hybrid Modern Authentication n’a pas sa place ici, car elle ne sélectionne aucun saut SMTP.

## Comment les connecteurs identifient le pair

Une fois l’itinéraire choisi, chaque côté doit pouvoir faire confiance au pair. Le Hybrid Configuration Wizard crée des connecteurs d’envoi/réception locaux ainsi que les connecteurs entrants/sortants correspondants dans Exchange Online. Le transport s’effectue via SMTP sur TCP 25 et utilise TLS ([Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites), [Hybrid mail flow](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/hybrid-mail-flow)).

Le connecteur d’envoi local détermine la destination et les exigences TLS pour le domaine de coexistence. Le connecteur sortant cloud décrit l’organisation locale comme destination. Dans le sens inverse, le connecteur de réception ou entrant accepte le trafic selon des conditions documentées, notamment l’identité du certificat et l’origine.

Le certificat remplit ici une fonction concrète : il identifie le point de terminaison SMTP lors de la négociation TLS. Le Subject ou le Subject Alternative Name, les paramètres du connecteur, la chaîne de certificats présentée et le nom d’hôte réel doivent correspondre. Un certificat valide présent dans le magasin de certificats ne suffit pas si le service de transport en présente un autre.

Les experts vérifient donc les deux sens séparément. Le sens A peut fonctionner, tandis que le sens B échoue à cause d’un autre connecteur, d’une autre destination DNS ou d’un autre nom de certificat.

## Routage des destinataires et domaines partagés

Un connecteur fonctionnel n’indique pas encore quels messages l’utilisent. Cette décision commence avec l’objet destinataire. Une boîte aux lettres distante locale pointe vers le cloud. Un objet de boîte aux lettres locale synchronisé dans Exchange Online pointe en retour vers l’organisation locale.

Les Accepted Domains déterminent également si une organisation est responsable de tous les destinataires d’un domaine ou si elle peut relayer des destinataires inconnus. Dans les domaines partagés, une configuration de relais interne n’est sûre que si le saut suivant connaît correctement les destinataires inconnus ou les refuse ([Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains), [Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)).

Un objet de boîte aux lettres distante obsolète peut donc entraîner un mauvais routage malgré un TLS sain. Inversement, une Target Address correcte ne sert à rien si le connecteur cloud est désactivé. Le diagnostic relie l’objet et le transport au lieu de ne considérer qu’un seul côté.

Pour les experts, les redirections, les contacts de messagerie, l’expansion des groupes de distribution et les règles de transport deviennent importants. Ils peuvent modifier l’adresse du destinataire initiale ou générer des destinataires supplémentaires. Chaque message qui en résulte reçoit sa propre décision de routage.

## Passerelles de messagerie avant ou après Exchange Online

De nombreuses organisations complètent l’hybride par une Secure Email Gateway ou une plateforme de filtrage cloud. Cela ajoute au moins un saut SMTP. Le chemin doit être dessiné séparément pour les messages entrants et sortants ([Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)).

Si la passerelle se trouve avant Exchange Online, le MX pointe vers la passerelle. EOP voit alors d’abord son IP source. Enhanced Filtering for Connectors peut intégrer les informations d’expéditeur d’origine dans l’évaluation de filtrage Microsoft si le connecteur et les IP ignorées sont correctement configurés ([Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)).

Pour les messages sortants, il doit être établi si Exchange Online envoie directement, via la passerelle ou, avec CMT, d’abord localement puis via la passerelle. Plusieurs chemins autorisés peuvent contourner les politiques et générer des signatures DKIM, des IP de sortie et des journaux différents.

Le contrôle des experts est un graphe de chemins autorisés : chaque flèche nomme l’initiateur, la destination, le port, la vérification TLS, les domaines autorisés, la tâche de filtrage, le propriétaire de la file d’attente et la source des journaux. Un nom de passerelle sans ces informations ne constitue pas encore une architecture.

## Suivre un message de bout en bout

Le dépannage commence par un message de test dont l’expéditeur, le destinataire et l’heure sont connus. Le chemin public est d’abord vérifié, puis les événements de chaque côté Exchange concerné.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für DNS- und SMTP-Test">
    <button type="button" role="tab" data-os-tab="windows" aria-selected="true">Windows</button>
    <button type="button" role="tab" data-os-tab="unix" aria-selected="false">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.com
Test-NetConnection mail.example.com -Port 25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig MX example.com
nc -vz mail.example.com 25
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) et [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) indiquent la destination MX publiée. [`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) et [`nc`](https://man.openbsd.org/nc) vérifient si TCP 25 est accessible depuis le point de mesure. Cela ne prouve pas encore que la vérification TLS ou du connecteur ait réussi.

Vient ensuite la négociation SMTP. OpenSSL peut être utilisé sur les deux plateformes d’administration.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-STARTTLS-Test">
    <button type="button" role="tab" data-os-tab="windows" aria-selected="true">Windows</button>
    <button type="button" role="tab" data-os-tab="unix" aria-selected="false">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
openssl s_client -starttls smtp -connect mail.example.com:25 `
  -servername mail.example.com -showcerts
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
openssl s_client -starttls smtp -connect mail.example.com:25 \
  -servername mail.example.com -showcerts
```

  </div>
</div>

[`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) affiche la chaîne de certificats, les noms et la négociation TLS. Pour un dialogue de test complet et autorisé, [`swaks`](https://jetmore.org/john/code/swaks/) convient. Les messages de production ne sont pas testés avec des expéditeurs inventés ; l’identité de test et l’itinéraire attendu sont définis à l’avance.

Dans Exchange Online, [`Get-MessageTraceV2`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2) fournit les événements cloud. Localement, [`Get-MessageTrackingLog`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog) montre le traitement sur les serveurs Exchange, et [`Get-Queue`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue) affiche les sauts suivants en attente. Les horodatages sont ramenés dans un fuseau horaire commun ; Internet Message ID et Network Message ID aident à relier les différentes parties ([Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq), [Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)).

## Problèmes typiques sans digressions

Une **erreur TLS** est d’abord traitée comme un problème de transport : à quel hôte la connexion a-t-elle été établie, quel certificat a-t-il présenté et quelle condition de connecteur le pair attendait-il ? La synchronisation des destinataires ne devient pertinente que si le message est mal routé après une acceptation réussie.

Un **NDR pour destinataire inconnu** conduit au contraire d’abord vers l’objet destinataire et le type d’Accepted Domain. Ce n’est que lorsque l’objet est correct que l’on vérifie si l’itinéraire choisi le transporte vers le bon côté.

Une **file d’attente en attente** nécessite le saut suivant, l’heure de nouvelle tentative et `LastError`. Le port ouvert de l’hôte de destination ne sert que de test suivant. Une **boucle** se manifeste par des en-têtes `Received` répétés, des sauts ou des événements de suivi, et survient généralement lorsque les deux côtés se retransmettent des destinataires inconnus ([RFC 5321: Trace information and loop detection](https://datatracker.ietf.org/doc/html/rfc5321#section-6.3)).

Une **connexion qui ne fonctionne que dans un sens** n’est pas une contradiction. Les sens opposés utilisent des expéditeurs, des connecteurs et des vérifications de certificats différents. Ils sont enregistrés et testés séparément.

## Sécurité, exploitation et modifications

Le SMTP hybride ouvre un chemin de transport délibérément autorisé. Ce chemin devrait être limité aux systèmes source et destination, aux certificats et aux domaines documentés. Les relais ouverts, les plages IP trop larges ou les connecteurs qui considèrent chaque message comme fiable contredisent ce modèle.

Les changements de certificat sont planifiés comme des changements de routage. Avant l’expiration, le nouveau certificat, l’attribution de service, la chaîne présentée et l’attente du connecteur sont vérifiés des deux côtés. Des messages de test dans les deux sens et un plan de retour arrière contrôlé suivent ensuite.

Pour l’exploitation courante, les connecteurs hybrides, l’expiration des certificats, la croissance des files d’attente, les erreurs Message Trace, les destinations DNS et l’état de santé des passerelles sont surveillés au minimum. Avec Centralized Mail Transport, la capacité du chemin de sortie local s’ajoute.

## Sauvegarde et reconstruction du chemin de messagerie

Pendant une interruption, les messages SMTP se trouvent dans les files d’attente des systèmes respectivement responsables. Une sauvegarde de configuration ne restaure pas ces messages en attente. La configuration et l’état du transport sont donc considérés séparément.

La reconstruction comprend les paramètres de connecteur des deux côtés, les domaines acceptés et distants, les règles de transport, les entrées DNS publiques, les certificats avec clés privées, la configuration de la passerelle et la sélection HCW. Les secrets sont stockés de manière protégée ; des exports lisibles documentent la structure et les dépendances.

Après une reconstruction, on ne teste pas seulement un port. Un message marqué emprunte le chemin attendu dans chaque sens. Message Trace, les journaux de suivi locaux, les journaux de passerelle et la boîte aux lettres de destination confirment chaque saut. Seule cette réception de bout en bout montre que le routage et le filtrage sont à nouveau corrects.

## Évolution technique et compromis

Le Hybrid Configuration Wizard a automatisé, au fil de plusieurs générations d’Exchange, la connexion à Exchange Online. Le transport est resté SMTP/TLS, tandis que les connecteurs cloud, les certificats pris en charge et les options de routage ont évolué ([Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)).

Le transport Internet direct maintient les chemins courts et utilise chaque plateforme là où se trouve la boîte aux lettres. Centralized Mail Transport centralise le contrôle, mais rend les e-mails cloud dépendants de la sortie locale. Une passerelle tierce ajoute un filtrage ou un chiffrement spécialisé, mais augmente le nombre de sauts et de sources de journaux. Le bon choix découle d’une exigence démontrable, et non du souhait que tout passe par le même bloc dans le diagramme.

## Sources

- [Microsoft Learn – Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing)
- [Microsoft Learn – Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)
- [Microsoft Learn – Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)
- [Microsoft Learn – Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)
- [Microsoft Learn – Hybrid mail flow](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/hybrid-mail-flow)
- [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox)
- [Microsoft Learn – Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)
- [Microsoft Learn – Connectors on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors)
- [Microsoft Learn – Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)
- [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)
- [Microsoft Learn – Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)
- [Microsoft Learn – Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)
- [Microsoft Learn – Trace an email message](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)
- [Microsoft Learn – Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq)
- [Microsoft Learn – Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)
- [Microsoft Learn – Queues in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues)
- [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2)
- [Microsoft Learn – Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog)
- [Microsoft Learn – Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc(1)](https://man.openbsd.org/nc)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Swaks – SMTP test tool](https://jetmore.org/john/code/swaks/)
- [RFC 5321 – Trace information and loop detection](https://datatracker.ietf.org/doc/html/rfc5321#section-6.3)
