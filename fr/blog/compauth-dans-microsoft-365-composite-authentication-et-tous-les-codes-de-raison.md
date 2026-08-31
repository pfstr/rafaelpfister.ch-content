---
title: "compauth dans Microsoft 365 : Composite Authentication et tous les codes de raison"
navTitle: "Codes compauth"
description: "Microsoft 365 complète SPF, DKIM et DMARC par sa propre évaluation : compauth. Ce que vérifie Composite Authentication, la signification de pass, softpass, fail et none, et la cause de chaque code de raison, de 000 à 905."
date: "2026-08-26"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min de lecture"
themen:
  - microsoft-365-exchange
hauptthema: "microsoft-365-exchange"
produkte:
  - "exchange-online"
protokolle:
  - "mail-auth"
  - "troubleshooting"
related:
  - exchange-authmechanism-10-authas-internal
  - exchange-hybrid-header-intern-extern
  - dns-records-e-mail-stolpersteine
slug: "compauth-dans-microsoft-365-composite-authentication-et-tous-les-codes-de-raison"
translationId: "article-a9dceac9ee095bbd"
translationOf: microsoft-365-compauth-reason-codes
url: https://rafaelpfister.ch/fr/blog/compauth-dans-microsoft-365-composite-authentication-et-tous-les-codes-de-raison
translationSourceHash: a37557eaef3ea6605e72281d81c56154d6062ae726ef646baa906c2d7d9927a4
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:20:46.582Z
translationReview: automatic
---

# compauth dans Microsoft 365 : Composite Authentication et tous les codes de raison

Dans l’en-tête `Authentication-Results` d’un e-mail reçu dans Microsoft 365 figure, à côté des résultats standard pour SPF, DKIM et DMARC, un champ propre à Microsoft :

```text
Authentication-Results: spf=pass (sender IP is 192.0.2.10)
  smtp.mailfrom=example.com; dkim=pass (signature was verified)
  header.d=example.com; dmarc=pass action=none header.from=example.com;
  compauth=pass reason=100
```

`compauth` signifie Composite Authentication : Microsoft 365 combine les résultats de SPF, DKIM et DMARC avec d’autres signaux du message afin d’établir une évaluation globale de la crédibilité de l’adresse From visible. L’évaluation se base sur le domaine From, c’est-à-dire l’adresse que les destinataires voient dans leur client de messagerie. Microsoft comble ainsi la lacune qui apparaît lorsqu’un domaine expéditeur n’a pas publié d’enregistrements d’authentification ou ne les a publiés que de manière incomplète : même sans politique DMARC, il vérifie implicitement si l’e-mail correspond au domaine revendiqué.

## Les quatre résultats

- `compauth=pass` : Le message a réussi l’authentification explicite (DMARC) ou implicite.
- `compauth=softpass` : La vérification implicite a réussi avec un niveau de confiance moindre.
- `compauth=fail` : Le message a échoué à la vérification explicite ou implicite.
- `compauth=none` : Aucune vérification composite n’a eu lieu ou elle a été ignorée.

Un `compauth=fail` n’entraîne pas automatiquement une mise en quarantaine ou un classement dans le dossier de courrier indésirable. Il s’agit d’un signal d’entrée pour la décision de filtrage ; le traitement effectif dépend de `CAT` et d’autres champs de `X-Forefront-Antispam-Report`. À l’inverse, pour savoir pourquoi compauth a pris cette décision, il faut consulter le code `reason` qui suit directement le résultat.

## Vue d’ensemble des codes de raison

Le code à trois chiffres indique la règle ayant conduit au résultat. Le premier chiffre sert de groupe : 0xx et 6xx sont des échecs, 1xx et 7xx des vérifications réussies, 2xx correspond à softpass, tandis que 3xx, 4xx et 9xx signifient qu’aucune vérification n’a été effectuée ou qu’elle a été ignorée.

| Code | Signification |
|---|---|
| `000` | Échec explicite : échec DMARC avec une politique `p=quarantine` ou `p=reject`. |
| `001` | Échec implicite : le domaine ne publie aucun enregistrement d’authentification ou seulement des enregistrements faibles (SPF `~all`/`?all`, DMARC `p=none`). |
| `002` | L’organisation a explicitement interdit à cette paire expéditeur/domaine d’envoyer des e-mails usurpés (entrée gérée manuellement). |
| `010` | Échec DMARC avec `p=reject`/`p=quarantine`, et le domaine expéditeur est un Accepted Domain interne (usurpation de sa propre organisation). |
| `100` | SPF ou DKIM a réussi, et les domaines MAIL FROM et From sont alignés. |
| `101` | Le message est signé DKIM par le domaine From. |
| `102` | Les domaines MAIL FROM et From sont alignés, SPF a réussi. |
| `103` / `104` | Le domaine From correspond à l’enregistrement PTR (recherche inversée) de l’adresse IP de remise. |
| `108` | Échec DKIM dû à une modification du corps du message lors d’étapes légitimes précédentes, par exemple dans son propre environnement OnPrem. |
| `109` | Le domaine n’a pas d’enregistrement DMARC, mais la vérification aurait réussi. |
| `111` | Malgré une erreur temporaire ou permanente DMARC, le domaine SPF ou DKIM est aligné avec le domaine From. |
| `112` | Un délai d’attente DNS a empêché la récupération de l’enregistrement DMARC. |
| `115` | L’e-mail provient d’une organisation Microsoft 365 dans laquelle le domaine From est configuré comme Accepted Domain. |
| `116` | L’enregistrement MX du domaine From correspond à l’enregistrement PTR de l’adresse IP de remise. |
| `130` | Un ARC-Sealer configuré comme fiable a outrepassé l’échec DMARC. |
| `201` / `202` | Softpass : le domaine From correspond à l’enregistrement PTR ou à son sous-réseau. |
| `3xx` / `4xx` / `9xx` | Aucune vérification composite effectuée ou vérification ignorée. |
| `501` / `502` | DMARC non appliqué, car il s’agit d’un NDR valide. |
| `601` | Échec implicite : le domaine expéditeur est un Accepted Domain interne (auto-usurpation, souvent avec Direct Send). |
| `701`–`704` | DMARC non appliqué, car l’organisation reçoit manifestement des e-mails légitimes de cette infrastructure. |
| `905` | DMARC non appliqué en raison d’un routage complexe, par exemple des e-mails Internet passant par Exchange OnPrem ou un service tiers avant Microsoft 365. |

## Les cas les plus fréquents en pratique

**`compauth=fail reason=001`** est le cas standard pour les domaines sans authentification ou avec une authentification faible. La correction relève de l’expéditeur : publier SPF avec `-all`, une signature DKIM et une politique DMARC. Tant que ces enregistrements font défaut, la délivrabilité dépend des signaux de réputation.

**`compauth=fail reason=601`** apparaît lorsque des e-mails utilisant son propre domaine comme expéditeur arrivent de l’extérieur, typiquement avec Direct Send : des appareils multifonctions, applications ou prestataires remettent directement au MX sans connecteur authentifié. La correction passe par un Inbound Connector correctement configuré ou par l’ajout de la source à son propre SPF.

**`compauth=fail reason=000` ou `010`** signifie que DMARC a été appliqué normalement. Si `action=oreject` figure à côté, Microsoft 365 a traduit la politique Reject de l’expéditeur en une remise en quarantaine. Il n’y a rien à corriger, sauf si l’expéditeur est légitime et que son authentification est défectueuse.

**`reason=108`** et **`reason=130`** concernent les scénarios de transfert et de passerelle : une station intermédiaire a modifié l’e-mail ou un ARC-Sealer fiable a conservé les résultats de vérification d’origine. Quiconque exploite une passerelle avant Microsoft 365 devrait déclarer son ARC-Sealing comme fiable dans la configuration antispam ; sinon, les e-mails légitimes continueront d’échouer à DMARC.

## Lire compauth dans l’en-tête

En pratique, `compauth` est rarement seul : seule l’interaction avec les résultats SPF, DKIM et DMARC individuels, l’alignement des domaines impliqués et la chaîne `Received` donne une image complète. L’[analyseur d’en-têtes d’e-mail](/tools/header-analyzer) de ce site décode `compauth` et son code de raison directement dans le navigateur, puis affiche côte à côte les domaines concernés (From, Envelope-From, `d=`) pour l’évaluation de l’alignement ; l’en-tête collé ne quitte pas le navigateur.

## Sources

1.  [Microsoft Learn: Anti-spam message headers](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo): Référence officielle des champs Authentication-Results et du tableau complet des codes de raison compauth.

2.  [Microsoft Learn: Security Operations guide for email authentication](https://learn.microsoft.com/en-us/defender-office-365/email-auth-sec-ops-guide): Procédure à suivre en cas d’échecs d’authentification du point de vue SecOps.

3.  [Microsoft Learn: Configure trusted ARC sealers](https://learn.microsoft.com/en-us/defender-office-365/email-authentication-arc-configure): Configuration d’ARC-Sealers fiables pour les scénarios de passerelle et de transfert (code de raison 130).

4.  [Microsoft Learn: Spam confidence level (SCL)](https://learn.microsoft.com/en-us/defender-office-365/anti-spam-spam-confidence-level-scl-about): Distinction entre le signal compauth et la décision effective du filtre.
