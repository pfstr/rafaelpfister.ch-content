---
title: "S/MIME dans les banques : cadre juridique en Suisse et choix entre Silver et Gold"
navTitle: "S/MIME Silver/Gold"
description: "Quelles exigences légales s’appliquent lorsqu’une banque suisse signe et chiffre des e-mails adressés à ses clients : secret bancaire, circulaire FINMA 2023/1, LPD, ZertES, droit de la preuve et conservation. Et quel niveau de certificat SwissSign en découle."
date: "2026-11-16"
kategorie: "Chiffrement des e-mails"
timeToRead: "12 min de lecture"
themen:
  - e-mail-verschluesselung
produkte: []
protokolle:
  - "verschluesselung"
  - "tls"
hauptthema: "e-mail-verschluesselung"
startseiteAb: "2099-12-31"
slug: "s-mime-avec-les-banques-certificat-silver-ou-gold-de-swisssign"
translationId: "article-5c2fd8d42f088dcb"
translationOf: smime-silver-gold-schweiz
translationSourceHash: f4b52127b6883bb31850668cc18a27ebba43f75200bfdc605d199f427d602ddf
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T03:25:46.975Z
translationReview: required
url: https://rafaelpfister.ch/fr/blog/s-mime-avec-les-banques-certificat-silver-ou-gold-de-swisssign
---

# S/MIME dans les banques : cadre juridique en Suisse et choix entre Silver et Gold

En bref : aucune disposition suisse n’impose à une banque S/MIME ni un niveau de certificat déterminé. Le secret bancaire, la circulaire FINMA 2023/1 et la loi sur la protection des données exigent toutefois que les données clients soient protégées de manière adéquate lors de leur envoi, et que la banque puisse justifier et documenter sa solution. Une signature S/MIME ne remplace pas une signature manuscrite, pas même avec un certificat Gold. En tant que moyen de preuve et justificatif de l’origine, Gold est nettement supérieur au certificat Silver, car seul Gold contient le nom vérifié de la banque.

La question du certificat SwissSign dont une banque a besoin pour l’envoi d’e-mails revient régulièrement dans mes projets avec des établissements financiers. Derrière cette question technique se cache généralement une question juridique : que doit faire la banque pour remplir ses obligations lors de l’envoi de données clients par e-mail, et quels effets produit un e-mail signé ?

**Focus Suisse :** cet article traite de la situation juridique en Suisse. Lorsqu’elles diffèrent, les règles de l’UE font l’objet d’une indication spécifique. Il ne remplace pas un conseil juridique dans un cas particulier.

## Aperçu des bases juridiques

| Réglementation | Contenu | Importance pour l’envoi d’e-mails |
|---|---|---|
| LB art. 47 | secret bancaire, sanctionné pénalement même en cas de négligence | les données clients ne doivent pas être divulguées sans autorisation |
| Circ.-FINMA 2023/1, ch. 73 à 82 | protection des données critiques tout au long de leur cycle de vie | protection également lors de l’envoi, neutralité technologique |
| LPD art. 8 et 24, OPDo art. 1 à 6 | sécurité des données, annonce des violations | mesures appropriées, annonce au PFPDT en cas de risque élevé |
| ZertES et CO art. 14 al. 2bis | types de signatures électroniques, validité formelle | les signatures S/MIME ne remplacent pas une signature |
| CPC art. 157 et 177 | moyens de preuve et libre appréciation des preuves | les e-mails signés sont des titres dont la force probante est librement appréciée |
| CO art. 958f, Olico | conservation des documents commerciaux | les e-mails chiffrés doivent rester lisibles |

Aucun de ces textes ne mentionne S/MIME ni un niveau de certificat. Ils définissent des objectifs de protection et des conséquences juridiques ; la banque décide elle-même de la manière de les atteindre.

## Secret bancaire

L’art. 47 LB punit quiconque, en tant qu’organe, employé ou mandataire d’une banque, révèle un secret qui lui a été confié en cette qualité. L’infraction est également punissable lorsqu’elle est commise par négligence. Cette norme s’adresse aux personnes physiques ; elle repose toutefois sur la relation contractuelle entre la cliente ou le client, détenteur du secret, et la banque.

Pour l’envoi d’e-mails, cela signifie qu’un e-mail non chiffré contenant des données clients, envoyé à une mauvaise adresse ou lu en cours de route, peut constituer une révélation par négligence. La banque doit donc veiller, par des mesures organisationnelles et techniques, à ce que ses collaborateurs n’envoient des données clients que de manière protégée.

Le client, en tant que détenteur du secret, peut consentir à la révélation. C’est pourquoi de nombreuses banques font confirmer dans leurs conditions générales ou dans une convention séparée que le client connaît les risques de la communication non chiffrée par e-mail et la souhaite. Un tel consentement ne couvre que la communication avec ce client et uniquement dans la mesure convenue. Il ne remplace pas le chiffrement pour les documents concernant des tiers ou pour la communication avec des partenaires commerciaux.

## Circulaire FINMA 2023/1 : données critiques

La circulaire FINMA 2023/1 « Risques opérationnels et résilience : banques » est entrée en vigueur le 1er janvier 2024 et a remplacé la circulaire 2008/21. La FINMA a étendu l’ancien accent mis sur la confidentialité des données d’identification des clients (CID) aux données critiques en général. Chaque banque détermine elle-même, en fonction des risques, ce qui est critique ; les données clients en font généralement partie du point de vue de la confidentialité.

Les chiffres marginaux suivants sont particulièrement pertinents pour l’envoi d’e-mails :

| Ch. | Contenu | Lien avec l’envoi d’e-mails |
|---|---|---|
| 73 | identifier les données critiques et les catégoriser selon leur criticité | déterminer quelles données ne peuvent être envoyées que de manière protégée |
| 74, 75 | garantir la confidentialité, l’intégrité et la disponibilité pendant tout le cycle de vie | l’envoi fait partie du cycle de vie |
| 76 | protéger les données critiques contre les accès non autorisés | protection de l’e-mail pendant le transport et chez le destinataire |
| 79 | limiter les risques accrus liés au stockage à l’étranger ou à l’accès depuis l’étranger | concerne les passerelles et plateformes de messagerie situées à l’étranger |
| 81 | annoncer sans délai les incidents importants à la FINMA | par exemple un envoi massif de données clients à de mauvais destinataires |
| 82 | sélectionner et surveiller avec soin les prestataires qui traitent des données critiques | concerne les fournisseurs de passerelles de chiffrement et les autorités de certification disposant d’une PKI gérée |

La circulaire est technologiquement neutre et ne mentionne ni S/MIME ni les niveaux de certificat. Dans ses explications relatives à la consultation, la FINMA cite le chiffrement des données comme exemple de mesures particulières permettant de sécuriser les données critiques. La banque doit définir sa solution dans ses propres directives et y indiquer également le niveau de certificat qu’elle utilise et pourquoi.

## Protection des données : LPD et OPDo

L’art. 8 LPD oblige les responsables du traitement à garantir une sécurité des données adaptée au risque. L’ordonnance sur la protection des données précise cette exigence au moyen d’objectifs de protection tels que la confidentialité, l’intégrité et la traçabilité, sans imposer de procédé particulier. S/MIME, une passerelle avec remise via un portail web ou une plateforme d’échange satisfont pareillement à l’exigence s’ils sont utilisés de manière adaptée au risque.

Si une violation de la sécurité des données est susceptible d’entraîner un risque élevé pour la personnalité ou les droits fondamentaux des personnes concernées, la banque doit l’annoncer au PFPDT dès que possible conformément à l’art. 24 LPD. Cette obligation d’annonce s’ajoute à celle envers la FINMA. Si l’e-mail était chiffré, le risque est généralement faible en cas d’envoi erroné et aucune annonce n’est nécessaire.

## Effets juridiques d’une signature S/MIME

### Pas une signature au sens du CO

La ZertES distingue la signature électronique, la signature électronique avancée, la signature électronique réglementée et la signature électronique qualifiée, ainsi que, pour les entreprises, le sceau électronique réglementé. Selon l’art. 14 al. 2bis CO, seule la signature électronique qualifiée assortie d’un horodatage qualifié est assimilée à la signature manuscrite.

Les certificats S/MIME de SwissSign sont délivrés selon les règles du CA/Browser Forum et les politiques ETSI LCP (Silver) ou NCP (Gold). Ils ne sont pas des certificats qualifiés au sens de la ZertES. Une banque ne peut donc pas émettre valablement par e-mail signé S/MIME des déclarations pour lesquelles la loi ou un contrat exige la forme écrite, pas même avec Gold.

### Signature personnelle et signature par passerelle

Une signature avancée suppose notamment que la clé de signature soit sous le contrôle exclusif de la personne signataire. Lorsqu’une passerelle signe de manière centralisée tous les e-mails sortants avec des clés détenues sur la passerelle, cette condition n’est pas remplie. La signature prouve alors que l’e-mail provient du système de la banque et qu’il n’a pas été modifié, mais non qu’une personne donnée l’a validé. Elle correspond ainsi davantage, sur le fond, à un sceau de l’organisation, sans satisfaire aux exigences d’un sceau réglementé au sens de la ZertES.

### Valeur probante

Un e-mail est un titre au sens de l’art. 177 CPC, et le tribunal apprécie librement sa force probante conformément à l’art. 157 CPC. Une signature S/MIME valide montre que l’e-mail n’a pas été modifié depuis sa signature et qu’il a été signé avec la clé d’un certificat donné. Ce que cela indique sur l’expéditeur dépend de ce que l’autorité de certification a vérifié avant la délivrance. C’est là que réside la différence juridiquement pertinente entre Silver et Gold.

## Silver ou Gold d’un point de vue juridique

Depuis le 1er septembre 2023, les `S/MIME Baseline Requirements` du CA/Browser Forum s’appliquent aux certificats S/MIME publiquement reconnus. SwissSign associe ses produits à ces types de validation dans les profils de certificats (CPR S/MIME) :

| Produit | Type selon le CA/B Forum | Ce que SwissSign vérifie | Ce qui figure dans le certificat | Ce que la signature atteste |
|---|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | le contrôle de la boîte aux lettres | uniquement l’adresse e-mail | une personne ayant accès à cette boîte aux lettres |
| E-Mail ID Gold | `Sponsor-validated` | l’organisation et la personne | nom, organisation, identifiant de l’organisation, pays, adresse e-mail | une personne vérifiée de cette banque |
| Organization Validated | `Organization-validated` | l’organisation | nom de l’organisation, aucune personne | une boîte aux lettres de cette banque |

Sur le plan cryptographique, les niveaux sont équivalents. Ils ne jouent donc aucun rôle pour le chiffrement et, par conséquent, pour les obligations de protection découlant du secret bancaire, de la circulaire FINMA et de la LPD.

Pour la signature, la différence est importante. Toute personne pouvant recevoir des e-mails à une adresse peut obtenir un certificat Silver. Quiconque enregistre un domaine au nom similaire, par exemple `musterbank-ch.com` au lieu de `musterbank.ch`, peut ainsi envoyer des e-mails signés tout aussi valablement que les véritables e-mails de la banque. En cas de litige sur l’origine d’un e-mail, une signature Silver ne prouve donc pas grand-chose. En revanche, un certificat Gold ou Organization Validated contient le nom de la banque vérifié par SwissSign. Avec Gold s’ajoute le rattachement à une personne dont la banque, en tant qu’autorité d’enregistrement dans le cadre de la PKI gérée, confirme l’identité.

## Conservation : CO et Olico

Selon l’art. 958f CO, les livres et pièces comptables doivent être conservés pendant dix ans, et l’ordonnance concernant la tenue et la conservation des livres de comptes (Olico) exige que les documents conservés électroniquement puissent être rendus lisibles durant cette période. Pour les banques s’ajoutent des obligations prudentielles et contractuelles de conservation. Il en résulte deux exigences pour l’exploitation de S/MIME :

- Les e-mails chiffrés doivent pouvoir être déchiffrés durant toute la période de conservation. La banque doit donc conserver de manière sûre les clés privées des certificats expirés ou déchiffrer les e-mails avant leur archivage.
- Si une signature doit encore servir de preuve après plusieurs années, il doit rester possible de retracer que le certificat était valable au moment de la signature. Les résultats de vérification ou les informations de révocation au moment de la signature doivent donc également être archivés.

**Note UE :** dans l’UE également, selon l’art. 25 al. 2 eIDAS, seule la signature électronique qualifiée est assimilée à la signature manuscrite. Depuis le 17 janvier 2025, le règlement DORA, avec ses propres exigences en matière de gestion des risques liés aux TIC, s’applique en outre aux entreprises financières de l’UE.

## Exemple : une passerelle pour tous les collaborateurs

De nombreuses banques utilisent une passerelle centralisée de chiffrement, telle que PrivaSphere, SEPPmail ou Totemomail, afin que tous les collaborateurs puissent chiffrer et signer des e-mails sans devoir gérer eux-mêmes des certificats. Les clés sont détenues sur la passerelle, qui signe et chiffre au nom des différentes boîtes aux lettres. Pour cette architecture, le choix suivant découle des sections précédentes :

| Boîte aux lettres | Niveau | Motif |
|---|---|---|
| boîtes aux lettres personnelles | E-Mail ID Gold | identifie la banque comme expéditeur par son nom vérifié et son identifiant d’organisation |
| boîtes aux lettres fonctionnelles (service clientèle, trafic des paiements) | Organization Validated | identifie la banque sans citer une personne en particulier |
| aucune | E-Mail ID Silver | prouve uniquement l’accès à une boîte aux lettres ; les e-mails provenant de domaines au nom similaire sont signés tout aussi valablement |

Comme la passerelle signe et non chaque personne, les clés ne sont pas sous le contrôle exclusif des collaborateurs. La signature produit juridiquement l’effet d’un sceau de la banque et non celui d’une signature personnelle. Il est donc déterminant que l’organisation soit indiquée de manière vérifiée dans le certificat, ce que seuls Gold et Organization Validated permettent. Le niveau n’a aucune importance pour le chiffrement lui-même.

Avant l’introduction, les points suivants doivent être clarifiés avec le fournisseur de la passerelle et l’autorité de certification :

- **Délivrance et renouvellement :** déterminer si la passerelle obtient et renouvelle automatiquement les certificats Gold via la PKI gérée de l’autorité de certification. Avec Gold, la banque confirme, en tant qu’autorité d’enregistrement, l’identité de chaque personne ; un processus est donc nécessaire pour les arrivées et départs, les changements de nom et les révocations.
- **Examen des prestataires :** le fournisseur de la passerelle et la PKI gérée traitent des données critiques au sens du ch. 82 de la circulaire FINMA 2023/1. L’emplacement des clés et des données doit être documenté ; en cas de stockage à l’étranger ou d’accès depuis l’étranger, le ch. 79 s’applique également.
- **Archivage :** les clés privées des certificats expirés et les données de vérification au moment de la signature doivent rester disponibles pendant toute la période de conservation.
- **Directive :** les niveaux choisis et les procédures autorisées doivent figurer dans la directive interne, qui sert de justificatif envers la FINMA et l’audit.

## Recommandation

- **Définir une directive :** la banque définit dans une directive interne quelles données ne peuvent être envoyées que chiffrées, quelles procédures sont autorisées à cet effet et quels niveaux de certificat elle utilise. C’est la base permettant de démontrer le respect des exigences de la circulaire FINMA et de la LPD.
- **Gold pour les boîtes aux lettres personnelles, Organization Validated pour les boîtes aux lettres fonctionnelles :** seuls ces niveaux identifient la banque comme expéditeur dans le certificat et ont ainsi une valeur probante allant au-delà de l’accès à une boîte aux lettres. Silver ne convient pas à la communication avec les clients.
- **Clarifier les consentements :** quiconque communique avec des clients par e-mail non chiffré a besoin de leur consentement documenté. Il ne remplace pas le chiffrement envers les tiers et les partenaires commerciaux.
- **Ne pas transmettre par e-mail les déclarations soumises à une forme :** pour les déclarations exigeant la forme écrite, une signature électronique qualifiée ou une signature manuscrite est nécessaire.
- **Examiner les prestataires :** le fournisseur de la passerelle et la PKI gérée de l’autorité de certification relèvent du ch. 82 de la circulaire FINMA ; en cas d’implantation à l’étranger, également du ch. 79.
- **Archiver les clés et données de vérification :** les clés privées destinées au déchiffrement et les données de vérification des signatures doivent rester disponibles pendant la période de conservation.
- **Relier les processus d’annonce :** un envoi erroné de données clients peut déclencher simultanément une annonce à la FINMA (ch. 81) et au PFPDT (art. 24 LPD). Les deux obligations peuvent être intégrées dans un processus commun de gestion des incidents.

## Sources

1.  [Circulaire FINMA 2023/1 « Risques opérationnels et résilience : banques »](https://www.finma.ch/de/~/media/finma/dokumente/dokumentencenter/myfinma/rundschreiben/finma-rs-2023-01-20221207.pdf): ch. 73 à 82 sur les données critiques, en vigueur depuis le 1er janvier 2024.

2.  [datenrecht.ch : Circ.-FINMA 2023/1 avec explications](https://datenrecht.ch/wp-content/uploads/FINMA-RS-OpRisk-%E2%80%93-Text-revidiert-1.1.2024.pdf): texte révisé avec les explications de la FINMA issues de la consultation, notamment sur le chiffrement des données comme mesure de protection.

3.  [Loi sur les banques (LB)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): art. 47 sur le secret bancaire.

4.  [Swissbanking : secret bancaire et cloud, expertise 2019](https://www.swissbanking.ch/_Resources/Persistent/b/1/3/3/b13339fe5dcd344e715dbe0a899afe3c8c56ed0b/SBVg_Bankgeheimnis_und_Cloud_Gutachten_LLAG_2019_DE.pdf): avis de droit sur l’interprétation de l’art. 47 LB, notamment sur le consentement du client.

5.  [Loi fédérale sur la protection des données (LPD)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 8 sur la sécurité des données, art. 24 sur l’annonce des violations de la sécurité des données.

6.  [Ordonnance sur la protection des données (OPDo)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): art. 1 à 6 avec les principes et objectifs de protection de la sécurité des données.

7.  [Loi fédérale sur la signature électronique (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): types de signatures et de sceaux électroniques, exigences applicables aux certificats qualifiés.

8.  [Code des obligations (CO)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): art. 14 al. 2bis sur l’assimilation de la signature électronique qualifiée à la signature manuscrite, art. 958f sur la conservation.

9.  [Code de procédure civile (CPC)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): art. 157 sur la libre appréciation des preuves, art. 177 sur la notion de titre.

10.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): profils de certificats de SwissSign (état au 18.09.2025), attribution de Silver, Gold et Organization Validated aux types de validation et champs Subject.

11.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): règles contraignantes pour les certificats S/MIME publiquement reconnus, définition des types de validation.
