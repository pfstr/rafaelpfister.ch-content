---
title: "Interne ou externe ? Classer les e-mails hybrides Exchange à l’aide des en-têtes : AuthAs, MessageDirectionality et X-originatorOrg"
navTitle: "Lire les en-têtes hybrides"
description: "Dans les environnements hybrides Exchange, la classification des en-têtes détermine si un e-mail est traité comme interne. Découvrez quels en-têtes déterminent cette classification, comment l’attribution au tenant fonctionne via le certificat et le connecteur, et comment reconnaître un message mal routé."
date: "2026-08-26"
kategorie: "Exchange OnPrem / hybride"
timeToRead: "10 min de lecture"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange-hybrid"
  - "hybrid-mailfluss"
  - "exchange-online"
protokolle:
  - "smtp"
  - "troubleshooting"
related:
  - exchange-authmechanism-10-authas-internal
  - microsoft-365-compauth-reason-codes
  - einliefernde-ip-adressen-aggregieren
slug: "interne-ou-externe-classer-les-e-mails-hybrides-exchange-a-l-aide-des-en-tetes-authas"
translationId: "article-c8d7859be8dbfe63"
translationOf: exchange-hybrid-header-intern-extern
url: https://rafaelpfister.ch/fr/blog/interne-ou-externe-classer-les-e-mails-hybrides-exchange-a-l-aide-des-en-tetes-authas
translationSourceHash: 5a0eccedd4b1a194461602319f5f1a8f59de204c1710e261c2358591bb720dfb
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:19:02.247Z
translationReview: automatic
---

# Interne ou externe ? Classer les e-mails hybrides Exchange à l’aide des en-têtes : AuthAs, MessageDirectionality et X-originatorOrg

Dans un environnement hybride, les e-mails entre Exchange OnPrem et Exchange Online doivent être traités comme du courrier interne : pas de filtre antispam intermédiaire, pas d’indication « Externe », remise aux listes de distribution protégées, noms d’affichage résolus. Le bon fonctionnement ne dépend pas du domaine de l’expéditeur, mais d’une poignée d’en-têtes qui doivent être conservés lors du passage entre les deux environnements. Savoir les lire permet de répondre directement à partir des en-têtes aux questions hybrides les plus courantes : l’e-mail est-il passé par le connecteur hybride ? Pourquoi a-t-il été classé comme externe ? Et à quel tenant a-t-il été attribué ?

## Les en-têtes concernés

```text
X-MS-Exchange-Organization-AuthAs: Internal
X-MS-Exchange-Organization-AuthSource: EX01.example.com
X-MS-Exchange-Organization-MessageDirectionality: Originating
X-OriginatorOrg: example.com
```

**`AuthAs`** porte la classification : `Internal` ou `Anonymous`. Elle résulte des autres signaux et constitue l’indicateur le plus direct du traitement appliqué par Exchange au message.

**`AuthSource`** indique le FQDN du serveur ayant effectué la classification : un serveur OnPrem interne, un serveur de boîtes aux lettres dans Exchange Online ou un serveur frontal EOP. Cela permet de voir de quel côté l’évaluation a eu lieu.

**`MessageDirectionality`** distingue `Originating` (le message a été créé au sein de l’organisation, dans Exchange Online ou via un connecteur entrant authentifié) de `Incoming` (le message est arrivé depuis l’extérieur).

**`X-OriginatorOrg`** identifie l’organisation expéditrice du point de vue d’Exchange Online : le domaine accepté par défaut ou correspondant du tenant expéditeur. L’en-tête est défini lors de l’envoi depuis Exchange Online via l’extension SMTP XOORG et est lié à la combinaison du certificat TLS EOP, de la configuration du connecteur et du domaine accepté. Il ne peut donc pas être falsifié par simple ajout : un `X-OriginatorOrg` soumis depuis l’extérieur sans la relation de confiance associée n’est pas reconnu comme tel.

S’y ajoutent les en-têtes `X-MS-Exchange-CrossTenant-*`, qu’Exchange Online appose lors du passage entre tenants, notamment `X-MS-Exchange-CrossTenant-AuthAs`. Ils reflètent la classification du point de vue du tenant destinataire.

## Fonctionnement technique de la relation de confiance

La classification Internal au-delà de la frontière organisationnelle repose sur deux composants configurés par le Hybrid Configuration Wizard :

Premièrement, le **connecteur entrant** dans Exchange Online de type OnPremises, qui identifie la source d’envoi via le certificat TLS (`TlsSenderCertificateName`) ou l’adresse IP. Cette attribution permet également à Exchange Online de déterminer à quel tenant une soumission est imputée (attribution).

Deuxièmement, l’indicateur **`CloudServicesMailEnabled`** sur les connecteurs des deux côtés. Il garantit que les en-têtes `X-MS-Exchange-Organization-*` (en-têtes inter-environnements) sont conservés lors du passage, au lieu d’être supprimés comme pour le courrier externe. Si cet indicateur manque ou si l’e-mail emprunte un chemin sans cette configuration, les en-têtes sont perdus et l’e-mail arrive comme `Anonymous`.

Il en découle la règle de diagnostic la plus importante : un e-mail hybride n’est interne que s’il a effectivement suivi le chemin configuré par le HCW.

## Cas 1 : l’e-mail arrive comme Anonymous alors qu’il devrait être interne

C’est le problème le plus courant : les e-mails provenant de boîtes aux lettres OnPrem apparaissent comme externes dans Exchange Online, avec analyse antispam, marquage « Externe » ou rejet par les listes de distribution protégées. Les causes, par fréquence décroissante :

- **Mauvais routage :** l’e-mail n’est pas passé par le connecteur hybride, mais par le MX (donc via EOP comme du courrier Internet) ou par une passerelle en amont qui supprime les en-têtes inter-environnements ou termine la connexion TLS. Cela est visible dans la chaîne `Received` : au lieu du passage direct d’OnPrem à `*.mail.protection.outlook.com` via le connecteur, des étapes intermédiaires apparaissent.
- **Renouvellement du certificat :** le certificat OnPrem a été renouvelé, mais le `TlsSenderCertificateName` du connecteur entrant n’a pas été mis à jour. L’identification par certificat ne fonctionne alors plus.
- **Configuration du connecteur modifiée :** `CloudServicesMailEnabled` a été désactivé lors du dépannage ou un connecteur créé manuellement remplace le connecteur HCW sans les paramètres requis.

Vérification côté Exchange Online :

```powershell
Get-InboundConnector | Format-List Name, ConnectorType,
  TlsSenderCertificateName, SenderIPAddresses, CloudServicesMailEnabled
```

Dans le suivi des messages, le champ `ConnectorName` indique si le message a effectivement été soumis via le connecteur attendu.

## Cas 2 : attribution au mauvais tenant

Exchange Online attribue chaque message entrant à un tenant ; l’en-tête `X-EOPTenantAttributedMessage` contient le GUID du tenant attribué. Si deux tenants utilisent le même `TlsSenderCertificateName` ou les mêmes `SenderIPAddresses` dans leurs connecteurs entrants, par exemple chez un prestataire de passerelle partagé ou après une migration, un message peut être imputé au mauvais tenant. Il n’apparaît alors pas dans le suivi des messages du tenant concerné et est soumis à des règles de transport tierces.

La GUID de votre tenant est fournie par `Get-OrganizationConfig | Select-Object GUID`; si elle ne correspond pas à l’en-tête, les identifiants des connecteurs doivent être séparés : un certificat distinct ou des plages IP distinctes par tenant.

## Cas 3 : un e-mail classé externe est tout de même traité comme interne

Le cas inverse se produit OnPrem : un connecteur de réception avec l’option `ExternalAuthoritative` (« Externally secured ») classe comme interne tout ce qui lui est soumis, ce qui se reconnaît à `AuthAs: Internal` associé à `AuthMechanism: 10`. Si un tel connecteur pointe vers une passerelle qui transporte également du courrier Internet, le courrier externe est considéré comme interne, avec toutes les conséquences pour les filtres antispam et la protection contre l’usurpation. Les détails et contre-mesures figurent dans l’article [AuthMechanism 10 et AuthAs Internal](/blog/exchange-authmechanism-10-authas-internal).

## Lire l’en-tête dans son ensemble

Pour classer un message concret, tous les signaux doivent être considérés ensemble : la chaîne `Received` pour le chemin réellement emprunté, `AuthAs`/`AuthSource`/`MessageDirectionality` pour la classification, `X-OriginatorOrg` et les en-têtes CrossTenant pour l’organisation d’origine. L’[analyseur d’en-têtes e-mail](/tools/header-analyzer) de ce site analyse ces champs directement dans le navigateur et met en évidence le passage entre tenants ainsi que la classification hybride dans le chemin de remise ; l’en-tête ne quitte pas le navigateur.

## Sources

1.  [Microsoft Tech Community: Demystifying and troubleshooting hybrid mail flow: when is a message internal?](https://techcommunity.microsoft.com/blog/exchange/demystifying-and-troubleshooting-hybrid-mail-flow-when-is-a-message-internal/1420838): Description officielle de la classification Internal, des en-têtes concernés et des prérequis des connecteurs.

2.  [Microsoft Tech Community: Advanced Office 365 Routing: Locking Down Exchange On-Premises when MX points to Office 365](https://techcommunity.microsoft.com/blog/exchange/advanced-office-365-routing-locking-down-exchange-on-premises-when-mx-points-to-/609238): Fonctionnement de XOORG et de X-OriginatorOrg pour le routage entre Exchange Online et OnPrem.

3.  [Microsoft Learn (archive) : Use headers to determine which Exchange Online tenant a message was attributed to](https://learn.microsoft.com/en-us/archive/blogs/eopfieldnotes/use-headers-to-determine-which-exchange-online-tenant-a-message-was-attributed-to): X-EOPTenantAttributedMessage et la procédure en cas d’attribution au mauvais tenant.

4.  [Microsoft Learn: Configure mail flow using connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow): Référence concernant les types de connecteurs entrants, TlsSenderCertificateName et l’attribution.
