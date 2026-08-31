---
title: "AuthMechanism 10 et AuthAs Internal : comment Exchange classe la remise dans l’en-tête"
navTitle: "AuthMechanism 10"
description: "L’en-tête X-MS-Exchange-Organization-AuthMechanism indique comment un serveur de remise s’est authentifié. La valeur 10 correspond à un Receive Connector avec Externally Secured et classe les e-mails externes comme internes, avec des conséquences sur les filtres antispam, les règles de flux de messagerie et la protection contre l’usurpation."
date: "2026-08-26"
featured: "2026-08-27"
kategorie: "Exchange OnPrem / Hybride"
timeToRead: "8 min de lecture"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
  - "exchange-hybrid"
protokolle:
  - "smtp"
  - "troubleshooting"
related:
  - exchange-hybrid-header-intern-extern
  - exchange-message-tracking-und-receive-connectoren-analysieren
  - typische-ursachen-fuer-mail-loops-und-deren-behebung
slug: "authmechanism-10-et-authas-internal-comment-exchange-classe-la-remise-dans-l-en-tete"
translationId: "article-0df383d5c49016da"
translationOf: exchange-authmechanism-10-authas-internal
url: https://rafaelpfister.ch/fr/blog/authmechanism-10-et-authas-internal-comment-exchange-classe-la-remise-dans-l-en-tete
translationSourceHash: 5a9335a90afc9bf7df78b908f71b679f64c29f3b9e96bd7f25bcc916123b82df
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:17:39.705Z
translationReview: automatic
---

# AuthMechanism 10 et AuthAs Internal : comment Exchange classe la remise dans l’en-tête

Lors de l’analyse de cas de spam, d’usurpation et de flux de messagerie dans des environnements Exchange, trois en-têtes apposés par Exchange à la réception sont déterminants :

```text
X-MS-Exchange-Organization-AuthAs: Internal
X-MS-Exchange-Organization-AuthSource: EX01.example.com
X-MS-Exchange-Organization-AuthMechanism: 10
```

`AuthAs` indique sous quelle identité l’expéditeur s’est présenté au transport. `AuthSource` désigne le serveur qui a effectué l’évaluation. `AuthMechanism` documente le mécanisme par lequel l’authentification a été établie. Ensemble, ils déterminent si Exchange traite un message comme interne ou externe, et cette classification a des conséquences importantes.

## Pourquoi cette classification est importante

`AuthAs` connaît en pratique deux valeurs : `Internal` et `Anonymous`. Un message classé comme `Internal` est traité différemment d’un e-mail externe :

- Les règles de flux de messagerie ayant la condition « expéditeur en dehors de l’organisation » ne s’appliquent pas.
- Le message peut être distribué à des groupes de distribution et à des boîtes aux lettres qui exigent des expéditeurs authentifiés (`RequireSenderAuthenticationEnabled`).
- Les contrôles antispam et anti-usurpation sont moins stricts ou ne sont pas effectués ; dans les environnements hybrides, l’avertissement externe n’est pas ajouté et Outlook n’affiche aucune indication « Externe ».
- Le nom d’affichage est résolu depuis le carnet d’adresses, et l’e-mail apparaît aux destinataires comme un message interne.

C’est précisément pourquoi la question « AuthAs Internal ou Anonymous ? » doit figurer au début de toute analyse d’en-tête : elle permet de déterminer pourquoi un e-mail d’usurpation manifeste a franchi le filtre antispam ou pourquoi une règle de flux de messagerie ne s’est jamais déclenchée.

## Les valeurs AuthMechanism

Microsoft ne documente pas entièrement et publiquement le codage de `AuthMechanism`. Deux valeurs sont pertinentes et bien documentées pour le dépannage :

| Valeur | Signification |
|---|---|
| `04` | Trafic Exchange authentifié : de boîte aux lettres à boîte aux lettres au sein de l’organisation, ainsi que trafic hybride via les connecteurs configurés par le Hybrid Configuration Wizard. |
| `10` | Receive Connector avec l’option d’authentification `ExternalAuthoritative` (« Protégé en externe » / « Externally secured ») : la connexion est considérée comme sécurisée en dehors d’Exchange, et tout ce qui est remis par son intermédiaire est traité comme interne. |

D’autres valeurs apparaissent dans les en-têtes, mais sans référence officielle. En pratique, la distinction suffit : `04` signifie une véritable authentification Exchange, `10` signifie une relation de confiance établie par la configuration du connecteur.

## Ce que signifie réellement Externally Secured

L’option `ExternalAuthoritative` sur un Receive Connector indique à Exchange : quelqu’un d’autre assure la sécurité de cette connexion, par exemple un pare-feu, un segment réseau dédié ou IPsec. Exchange ne vérifie alors plus rien et traite chaque remise via ce connecteur comme authentifiée et interne, y compris avec le droit d’utiliser n’importe quelle adresse d’expéditeur interne.

Cette option est prévue pour quelques scénarios seulement, par exemple un serveur d’applications entièrement digne de confiance dans son propre centre de données. Elle devient problématique lorsque le connecteur pointe vers une passerelle de messagerie en amont ou un filtre antispam dans la DMZ, par lequel transite également le courrier Internet. Après remise, chaque e-mail externe porte alors :

```text
X-MS-Exchange-Organization-AuthAs: Internal
X-MS-Exchange-Organization-AuthMechanism: 10
```

Conséquences : les e-mails externes sont considérés comme internes, les règles de flux de messagerie pour les expéditeurs externes ne s’appliquent pas, la protection contre l’usurpation pour votre propre domaine est inefficace, et toute personne pouvant atteindre la passerelle peut remettre des messages avec des adresses d’expéditeur internes à des destinataires qui exigent en principe des expéditeurs authentifiés.

## Trouver les connecteurs concernés

L’Exchange Management Shell permet d’afficher les Receive Connectors configurés avec `ExternalAuthoritative` :

```powershell
Get-ReceiveConnector | Where-Object {
  $_.AuthMechanism -match "ExternalAuthoritative"
} | Format-Table Identity, RemoteIPRanges, AuthMechanism, PermissionGroups
```

Pour chaque résultat, vérifiez quelles `RemoteIPRanges` sont renseignées et si les systèmes concernés ont réellement besoin de cette relation de confiance. Une passerelle qui doit uniquement relayer des e-mails n’en a pas besoin.

## L’alternative pour les scénarios de relais

Si un système doit uniquement relayer anonymement via Exchange (imprimante, application, supervision), un connecteur de relais anonyme est la solution plus propre : remise anonyme avec le droit de distribuer à n’importe quel destinataire, mais sans la classification Internal.

```powershell
New-ReceiveConnector -Name "Anonymous Relay" -TransportRole FrontendTransport `
  -RemoteIPRanges 192.0.2.10 -Bindings 0.0.0.0:25 -Usage Custom -PermissionGroups AnonymousUsers

Get-ReceiveConnector "EX01\Anonymous Relay" | Add-ADPermission `
  -User "NT AUTHORITY\ANONYMOUS LOGON" -ExtendedRights "ms-Exch-SMTP-Accept-Any-Recipient"
```

Les e-mails transmis via ce connecteur restent `AuthAs: Anonymous`, passent par les contrôles habituels et ne peuvent pas usurper des expéditeurs internes. `ExternalAuthoritative` doit rester réservé aux systèmes auxquels vous souhaitez délibérément accorder le droit d’utiliser des adresses d’expéditeur internes.

## Lire les en-têtes dans leur contexte

Pour déterminer le plus rapidement possible si un message concret a été classé comme interne ou externe et par quel chemin il est arrivé, consultez l’en-tête complet : `AuthAs`, `AuthMechanism` et `AuthSource`, ainsi que la chaîne `Received`. L’[analyseur d’en-têtes de messagerie](/tools/header-analyzer) sur ce site évalue directement ces champs dans le navigateur et met en évidence la classification hybride dans le chemin de distribution ; l’en-tête ne quitte pas le navigateur.

L’article [Interne ou externe ? Classer les e-mails Exchange hybrides dans l’en-tête](/blog/exchange-hybrid-header-intern-extern) explique comment la classification est préservée entre Exchange Online et OnPrem dans les environnements hybrides et comment identifier une attribution incorrecte.

## Sources

1.  [Microsoft Q&A: Exchange 2016 mail flow rule, which header is checked for "outside the organization"?](https://learn.microsoft.com/en-us/answers/questions/54418/exchange-2016-mail-flow-rule-which-header-is-check): Attribution d’AuthAs et d’AuthMechanism 10 à la configuration Externally Secured et à son effet sur les règles de flux de messagerie.

2.  [Microsoft Tech Community: Demystifying and troubleshooting hybrid mail flow: when is a message internal?](https://techcommunity.microsoft.com/blog/exchange/demystifying-and-troubleshooting-hybrid-mail-flow-when-is-a-message-internal/1420838): Description officielle de la classification Internal et de ses conséquences dans le flux de messagerie hybride.

3.  [msxfaq: X-MS-Exchange-Organization-AuthAs](https://www.msxfaq.de/cloud/exchangeonline/transport/x-ms-exchange-organization-authas.htm): Valeurs AuthAs, AuthSource et AuthMechanism observées dans différents scénarios de remise.

4.  [Microsoft Learn: Allow anonymous relay on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/allow-anonymous-relay): Configuration d’un connecteur de relais anonyme comme alternative à Externally Secured.
