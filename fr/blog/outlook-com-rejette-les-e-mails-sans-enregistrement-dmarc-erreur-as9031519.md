---
title: "Outlook.com rejette les e-mails sans enregistrement DMARC : erreur AS9031519"
navTitle: "Outlook.com : AS9031519"
description: "Depuis le 5 novembre 2026, Outlook.com, Hotmail et Live rejettent avec le code 550 5.7.1 et la mention AS9031519 les messages provenant de domaines sans enregistrement DMARC, même à faible volume. Ce que Microsoft documente, ce que les opérateurs ont mesuré, comment vérifier son propre domaine et ce que révèle l’indice de sécurité des e-mails des communes suisses."
date: "2026-11-10"
kategorie: "SMTP et flux de messagerie"
timeToRead: "8 min de lecture"
themen:
  - smtp-mailflow
  - microsoft-365-exchange
hauptthema: "smtp-mailflow"
produkte:
  - "exchange-online"
protokolle:
  - "mail-auth"
  - "dns"
  - "smtp"
  - "troubleshooting"
related:
  - dns-records-e-mail-stolpersteine
  - microsoft-365-compauth-reason-codes
slug: "outlook-com-rejette-les-e-mails-sans-enregistrement-dmarc-erreur-as9031519"
translationId: "article-84656f0bf3da2006"
translationOf: outlook-com-dmarc-ablehnung-as9031519
url: https://rafaelpfister.ch/fr/blog/outlook-com-rejette-les-e-mails-sans-enregistrement-dmarc-erreur-as9031519
translationSourceHash: afec1593a28f2ec6ab14b2af5b064bae9b03ef23d0d091f8b659713eff887222
translationModel: gpt-5.6-terra
translatedAt: 2026-11-11T03:19:53.248Z
translationReview: required
---

# Outlook.com rejette les e-mails sans enregistrement DMARC : erreur AS9031519

Depuis le 5 novembre 2026, les messages envoyés vers des boîtes aux lettres Outlook.com, Hotmail, Live et MSN reviennent avec une nouvelle erreur lorsque le domaine expéditeur n’a pas publié d’enregistrement DMARC :

```text
550 5.7.1 Message rejected: Message lacks SPF/DKIM/DMARC Auth Records
(AS9031519)
```

État des informations : 10 novembre 2026.

## Ce que Microsoft documente

Les politiques Postmaster d’Outlook.com ne contiennent toujours que la règle du 5 mai 2025 : les domaines qui envoient plus de 5 000 messages par jour à des comptes Outlook.com doivent passer SPF et DKIM et publier un enregistrement DMARC avec au minimum `p=none`, aligné via SPF ou DKIM. Les messages qui ne remplissent pas ces conditions arrivent d’abord dans le dossier Courrier indésirable et peuvent ensuite être rejetés avec l’erreur suivante :

```text
550; 5.7.515 Access denied, sending domain [SendingDomain] does not meet
the required authentication level.
```

Pour tous les autres expéditeurs, cette même page ne formule qu’une recommandation. La mention `AS9031519` et le texte d’erreur `Message lacks SPF/DKIM/DMARC Auth Records` ne figurent ni dans les politiques Postmaster ni dans l’article de support relatif à 5.7.515.

Microsoft ne s’est exprimé sur la situation actuelle que dans les forums Microsoft Q&A. Le 9 novembre, un modérateur identifié comme « Microsoft External Staff » a écrit qu’une perturbation en cours touchait le service Exchange lors de la remise à Outlook.com, Hotmail.com et Live.com. Elle est classée comme une dégradation de service et Microsoft en examine la cause. La publication ne mentionne ni numéro d’incident, ni cause, ni date de résolution. Aucun article de blog, ajout aux politiques Postmaster ni message public de santé du service n’est disponible.

## Ce que les opérateurs ont mesuré

Les données les plus fiables proviennent de fournisseurs de messagerie ayant analysé leurs journaux et publié les résultats sur la liste de diffusion mailop ainsi que dans leurs propres articles de blog (MailRoute, Mythic Beasts, MXroute, Fastmail, one.com, EasyDMARC). Les observations concordent :

- **Déclencheur :** le domaine expéditeur ne possède pas d’enregistrement DMARC. SPF et DKIM étaient valides et alignés dans les cas documentés. Le texte de l’erreur mentionne les trois mécanismes, mais le rejet est dû à l’absence d’enregistrement DMARC.
- **Volume :** même des domaines envoyant quelques messages par semaine sont touchés, bien en dessous du seuil de 5 000 par jour.
- **Destinataires :** seules les boîtes grand public sont concernées (outlook.com, hotmail.com, live.com, msn.com et des variantes nationales telles que hotmail.de ou outlook.be). Selon les rapports disponibles, les tenants Exchange Online destinataires ne rejettent pas avec cette erreur.
- **Irrégulier :** seule une partie des messages est rejetée. Un nouvel envoi du même message peut passer.
- **Correction :** un enregistrement avec `v=DMARC1; p=none` a suffi dans les cas documentés. Après publication, Outlook.com a recommencé à accepter les messages après quelques minutes à environ une heure et demie.

Deux analyses précisent davantage le déclencheur : chez one.com, seuls les domaines dont la requête pour `_dmarc.<domain>` se termine par `NXDOMAIN` ont été rejetés. Les domaines pour lesquels le nom existe mais sans enregistrement TXT (`NOERROR` sans réponse) n’y étaient pas concernés. Dans un journal, MXroute a trouvé 367 domaines expéditeurs rejetés, dont 286 avec `NXDOMAIN` sur `_dmarc`. Cette délimitation repose sur les données d’opérateurs individuels et n’est pas définitivement établie.

Depuis midi le 9 novembre, certaines personnes concernées signalent que les rejets ont cessé. Microsoft ne l’a pas confirmé.

## Cas particulier : Microsoft 365 comme expéditeur

Les tenants Microsoft 365 sont touchés de manière disproportionnée en tant qu’expéditeurs. Microsoft 365 ne crée pas d’enregistrement DMARC pour un domaine propre et, tant que DKIM n’est pas activé pour le domaine, Exchange Online signe avec le domaine `onmicrosoft.com` du tenant. Le rapport de non-remise enveloppe en outre le rejet dans un second code d’état faisant référence à SPF :

```text
550 5.7.23 The message was rejected because of Sender Policy Framework
violation -> 550 5.7.1 Message rejected: Message lacks SPF/DKIM/DMARC
Auth Records (AS9031519)
```

Dans ces cas, l’enregistrement SPF est généralement correct. L’erreur interne après la flèche est déterminante.

## Vérifier son propre domaine

La requête porte sur l’enregistrement TXT sous `_dmarc.<domain>` :

```bash
dig +short TXT _dmarc.example.com
```

```powershell
Resolve-DnsName -Type TXT -Name _dmarc.example.com
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `+short` | `dig` affiche uniquement le contenu de l’enregistrement |
| `TXT` ou `-Type TXT` | Type d’enregistrement de la requête |
| `_dmarc.example.com` ou `-Name` | Nom interrogé : le label `_dmarc` avant le domaine expéditeur |

</details>

Une sortie vide avec `dig +short` ou l’erreur `DNS name does not exist` avec `Resolve-DnsName` signifie qu’aucun enregistrement DMARC n’existe. Pour les sous-domaines : si `news.example.com` envoie des messages et ne possède pas lui-même d’enregistrement, l’enregistrement du domaine organisationnel `example.com` s’applique selon la RFC 7489.

Le [contrôle DNS des e-mails](https://rafaelpfister.ch/tools/mail-check) de cette page vérifie DMARC avec SPF, DKIM et l’alignement en une seule opération.

## Mesure immédiate et suite de la démarche

Si l’enregistrement manque, cette entrée rétablit la remise vers Outlook.com :

```text
_dmarc.example.com.  3600  IN  TXT  "v=DMARC1; p=none; rua=mailto:dmarc-reports@example.com"
```

`p=none` ne modifie pas le traitement de vos e-mails chez les autres destinataires et peut donc être publié sans préparation préalable. La balise `rua` n’est pas nécessaire pour l’acceptation par Outlook.com. Sans `rua`, vous ne recevez toutefois aucun rapport agrégé, et sans ces rapports, il manque la base nécessaire à toute étape ultérieure. L’adresse doit aboutir à une boîte aux lettres ou à un service d’analyse qui traite les rapports XML.

`p=none` ne protège pas le domaine contre les abus. L’enregistrement ne remplit que la condition d’acceptation. Pour une protection efficace, suivez ces étapes :

1. Recenser tous les systèmes qui envoient avec le domaine (serveur de messagerie, service de newsletter, ERP, boutique en ligne, scanner).
2. Maintenir SPF complet et valide pour ces systèmes, avec au maximum dix recherches DNS.
3. Activer DKIM avec le domaine propre, dans Microsoft 365 pour chaque domaine dans le portail Defender.
4. Analyser les rapports agrégés pendant quelques semaines et corriger les sources non alignées.
5. Relever la politique à `p=quarantine`, puis à `p=reject`.

Le [guide pour administrateurs DNS](https://rafaelpfister.ch/blog/dns-records-e-mail-stolpersteine) décrit les erreurs typiques lors de la publication de ces enregistrements (enregistrements SPF en double, clés DKIM trop longues, nom de zone ajouté automatiquement).

## L’effet en Suisse : 152 communes en cinq jours

L’[indice de sécurité des e-mails Suisse](https://rafaelpfister.ch/open-data/mail-security-index) vérifie chaque nuit les enregistrements DNS de 3 364 organisations suisses, dont 2 078 communes. Le [journal des modifications](https://rafaelpfister.ch/open-data/mail-security-index/aenderungen) montre depuis le 6 novembre une vague de nouveaux enregistrements DMARC publiés. La date correspond à chaque fois à la synchronisation nocturne.

| Synchronisation | Organisations nouvellement dotées d’un enregistrement DMARC | dont communes |
|---|---|---|
| 19 octobre au 5 novembre | au maximum 5 par jour | au maximum 4 par jour |
| 6 novembre | 4 | 4 |
| 7 novembre | 22 | 21 |
| 8 novembre | 1 | 1 |
| 9 novembre | 2 | 2 |
| 10 novembre | 140 | 124 |

En cinq jours, 169 organisations ont publié pour la première fois un enregistrement DMARC, dont 152 communes. Le nombre d’organisations sans DMARC dans l’indice est passé de 817 à 650. Le moment coïncide avec les rejets chez Outlook.com. Je n’ai pas demandé les raisons aux communes elles-mêmes.

Les enregistrements présentent les caractéristiques suivantes :

- **Politique :** 139 des 152 communes ont défini `p=none`, cinq `p=quarantine`, trois `p=reject`. Pour cinq communes, l’enregistrement se limite à `v=DMARC1` sans balise `p=` ; il est donc invalide.
- **Rapports :** 114 enregistrements contiennent exclusivement `v=DMARC1; p=none`. Au total, 124 des 152 communes n’ont pas renseigné d’adresse `rua` et ne reçoivent donc aucun rapport.
- **Conversions centralisées :** les changements apparaissent de manière groupée. Le 10 novembre, 32 communes genevoises ont été ajoutées, dont les serveurs de messagerie fonctionnent tous sous `acg-geneve.ch`, ainsi que 26 communes zurichoises et 19 communes fribourgeoises. Le 7 novembre, il s’agissait de huit communes nidwaldiennes et de sept communes tessinoises. Le regroupement par serveur de messagerie et par canton indique que des prestataires informatiques ont configuré les enregistrements pour toutes les communes qu’ils gèrent en une seule opération.

Ces enregistrements suffisent pour la remise vers Outlook.com. Ils ne protègent pas encore les domaines des communes contre les expéditeurs falsifiés : sans `rua`, les données manquent pour pouvoir relever ultérieurement la politique. Parmi les 2 078 communes de l’indice, 537 n’ont toujours pas d’enregistrement DMARC.

## Mise en perspective

On ne sait pas si Microsoft retirera ce rejet ou le confirmera comme règle. Cela joue un rôle mineur pour la planification : Google et Yahoo exigent DMARC des expéditeurs de masse depuis février 2024, Microsoft depuis mai 2025, et les politiques Postmaster d’Outlook.com le recommandent à tous les expéditeurs. Ces derniers jours, un domaine sans enregistrement DMARC n’était plus remis de manière fiable chez l’un des plus grands fournisseurs de messagerie. L’enregistrement avec `p=none` et `rua` se publie en quelques minutes et n’a aucun effet secondaire sur la remise.

## Sources

1.  [Outlook.com Postmaster: Policies, Practices, and Guidelines](https://substrate.office.com/ip-domain-management-snds/postmaster/Policies) : exigences officielles pour les expéditeurs ; mentionne le seuil de 5 000 messages par jour, l’erreur 5.7.515 et la recommandation à tous les expéditeurs.

2.  [Microsoft Support: Fix NDR error 550 5.7.515 in Outlook.com](https://support.microsoft.com/en-us/outlook/fix-ndr-error-550-5-7-515-in-outlook-com) : article de support relatif à l’erreur documentée pour les expéditeurs à fort volume.

3.  [Microsoft Q&A: SPF Validation Error](https://learn.microsoft.com/en-us/answers/questions/6063210/spf-validation-error) : fil de forum contenant la réponse du modérateur du 9 novembre 2026 sur la perturbation en cours ; contribution de forum, pas un message officiel de santé du service.

4.  [MailRoute: Outlook.com Is Quietly Rejecting Email From Domains Without DMARC](https://mailroute.net/blog/outlook-rejecting-email-without-dmarc) : analyse des journaux sortants d’un fournisseur de messagerie depuis le 5 novembre 2026.

5.  [Mythic Beasts: Microsoft are rejecting your mail](https://www.mythic-beasts.com/blog/2026/11/09/microsoft-are-rejecting-your-mail/) : rapport d’un hébergeur faisant état de plus de 500 rejets le premier jour et de leur résolution via `p=none`.

6.  [mailop: Microsoft requiring DMARC now or just us?](https://www.mail-archive.com/mailop@mailop.org/msg27569.html) : fil de la liste de diffusion des opérateurs avec les mesures de MXroute, Fastmail et one.com, notamment la délimitation à `NXDOMAIN`.

7.  [EasyDMARC: Microsoft Rejecting Emails Without DMARC](https://easydmarc.com/blog/microsoft-rejecting-emails-without-dmarc/) : test reproduit avec SPF et DKIM valides et alignés, sans enregistrement DMARC.

8.  [heise online: Outlook.com: Die Mail ist echt und sie kommt trotzdem nicht an](https://www.heise.de/news/Outlook-com-Die-Mail-ist-echt-und-sie-kommt-trotzdem-nicht-an-11517776.html) : résumé en allemand des rapports du 9 novembre 2026.

9.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489) : spécification, notamment sur la détermination de l’enregistrement via le domaine organisationnel.

10.  [Indice de sécurité des e-mails Suisse : journal des modifications](https://rafaelpfister.ch/open-data/mail-security-index/aenderungen) : modifications quotidiennes des enregistrements DNS de 3 364 organisations suisses ; base des chiffres concernant les communes.
