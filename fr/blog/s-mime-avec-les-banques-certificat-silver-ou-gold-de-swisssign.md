---
title: "S/MIME dans les banques : cadre juridique en Suisse et choix entre Silver et Gold"
navTitle: "S/MIME Silver/Gold"
description: "Quelles exigences légales s’appliquent lorsqu’une banque suisse signe et chiffre des e-mails à ses clients : secret bancaire, Circulaire FINMA 2023/1, LPD, ZertES, droit de la preuve et conservation. Et quel niveau de certificat SwissSign en découle."
date: "2026-11-16"
kategorie: "Chiffrement des e-mails"
timeToRead: "11 min de lecture"
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
translationSourceHash: 60de430124ebe20a3001bd48fc55affcb921141039e7c85303da9c7216b01e89
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T15:20:27.065Z
translationReview: required
url: https://rafaelpfister.ch/fr/blog/s-mime-avec-les-banques-certificat-silver-ou-gold-de-swisssign
---

# S/MIME dans les banques : cadre juridique en Suisse et choix entre Silver et Gold

En bref : aucune disposition suisse n’impose à une banque d’utiliser S/MIME ou un niveau de certificat déterminé. Le secret bancaire, la Circulaire FINMA 2023/1 et la loi sur la protection des données exigent toutefois que les données des clients soient protégées de manière appropriée lors de leur envoi, et que la banque puisse justifier et documenter sa solution. Une signature S/MIME ne remplace pas une signature manuscrite, pas même avec un certificat Gold. En tant que moyen de preuve et preuve de l’origine, Gold est nettement supérieur au certificat Silver, car seul Gold contient le nom vérifié de la banque.

La question du certificat SwissSign dont une banque a besoin pour l’envoi d’e-mails revient régulièrement dans mes projets avec des institutions financières. Derrière la question technique se cache généralement une question juridique : que doit faire la banque pour remplir ses obligations lors de l’envoi de données clients par e-mail, et quel effet produit un e-mail signé ?

**Focus Suisse :** cet article traite du cadre juridique suisse. Lorsqu’elles diffèrent, les règles de l’UE font l’objet d’une indication signalée. Il ne remplace pas un conseil juridique pour un cas particulier.

## Les bases juridiques en bref

| Réglementation | Contenu | Importance pour l’envoi d’e-mails |
|---|---|---|
| LB art. 47 | Secret bancaire, passible de sanctions pénales même en cas de négligence | Les données clients ne doivent pas être divulguées sans autorisation |
| Circ.-FINMA 2023/1, ch. marg. 73 à 82 | Protection des données critiques pendant tout leur cycle de vie | Protection également lors de l’envoi, neutralité technologique |
| LPD art. 8 et 24, OPDo art. 1 à 6 | Sécurité des données, annonce des violations | Mesures appropriées, annonce au PFPDT en cas de risque élevé |
| ZertES et CO art. 14 al. 2bis | Types de signatures électroniques, validité de la forme | Les signatures S/MIME ne remplacent pas une signature |
| CPC art. 157 et 177 | Moyens de preuve et libre appréciation des preuves | Les e-mails signés sont des titres dont la force probante est librement appréciée |
| CO art. 958f, Olico | Conservation des documents commerciaux | Les e-mails chiffrés doivent rester lisibles |

Aucun de ces textes ne mentionne S/MIME ou un niveau de certificat. Ils définissent des objectifs de protection et des conséquences juridiques ; la banque choisit elle-même comment les atteindre.

## Secret bancaire

L’art. 47 LB sanctionne quiconque, en tant qu’organe, employé ou mandataire d’une banque, révèle un secret qui lui a été confié à ce titre. La commission par négligence est également punissable. La norme s’adresse aux personnes physiques ; elle repose toutefois sur la relation contractuelle entre la cliente ou le client, titulaire du secret, et la banque.

Pour l’envoi d’e-mails, cela signifie qu’un e-mail non chiffré contenant des données clients, envoyé à une mauvaise adresse ou lu durant son acheminement, peut constituer une divulgation par négligence. La banque doit donc veiller, par des mesures organisationnelles et techniques, à ce que ses collaborateurs n’envoient les données clients que de manière protégée.

En tant que titulaire du secret, le client peut consentir à la divulgation. C’est pourquoi de nombreuses banques font confirmer dans leurs conditions générales ou dans une convention séparée que le client connaît les risques de la communication non chiffrée par e-mail et la souhaite. Un tel consentement ne couvre que la communication avec ce client et uniquement dans la mesure convenue. Il ne remplace pas le chiffrement pour les documents concernant des tiers ou pour la communication avec des partenaires commerciaux.

## Circulaire FINMA 2023/1 : données critiques

La Circulaire FINMA 2023/1 « Risques opérationnels et résilience : banques » est entrée en vigueur le 1er janvier 2024 et a remplacé la Circulaire 2008/21. La FINMA a étendu l’ancien accent mis sur la confidentialité des données d’identification des clients (CID) aux données critiques en général. Chaque banque détermine elle-même, selon une approche fondée sur les risques, ce qui est critique ; les données clients en font en règle générale partie au regard de la confidentialité.

Pour l’envoi d’e-mails, les chiffres marginaux suivants sont particulièrement pertinents :

| Ch. marg. | Contenu | Lien avec l’envoi d’e-mails |
|---|---|---|
| 73 | Identifier les données critiques et les catégoriser selon leur criticité | Déterminer quelles données ne peuvent être envoyées que de manière protégée |
| 74, 75 | Garantir la confidentialité, l’intégrité et la disponibilité pendant tout le cycle de vie | L’envoi fait partie du cycle de vie |
| 76 | Protéger les données critiques contre les accès non autorisés | Protection de l’e-mail durant le transport et chez le destinataire |
| 79 | Limiter les risques accrus liés au stockage à l’étranger ou aux accès depuis l’étranger | Concerne les passerelles et plateformes de messagerie situées à l’étranger |
| 81 | Annoncer sans délai les incidents importants à la FINMA | Par exemple un envoi massif de données clients à de mauvais destinataires |
| 82 | Sélectionner et surveiller soigneusement les prestataires qui traitent des données critiques | Concerne les fournisseurs de passerelles de chiffrement et les autorités de certification avec PKI gérée |

La circulaire est technologiquement neutre et ne mentionne ni S/MIME ni niveaux de certificat. Dans ses explications relatives à la consultation, la FINMA cite le chiffrement des données comme exemple de mesures particulières permettant de sécuriser les données critiques. La banque doit définir sa solution dans ses propres directives et y indiquer également quel niveau de certificat elle utilise et pourquoi.

## Protection des données : LPD et OPDo

L’art. 8 LPD oblige les responsables du traitement à garantir une sécurité des données appropriée au risque. L’ordonnance sur la protection des données précise cette obligation au moyen d’objectifs de protection tels que la confidentialité, l’intégrité et la traçabilité, sans prescrire de procédé. S/MIME, une passerelle avec remise via un portail web ou une plateforme d’échange remplissent pareillement l’exigence s’ils sont utilisés de manière appropriée au risque.

Si une violation de la sécurité des données est susceptible d’entraîner un risque élevé pour la personnalité ou les droits fondamentaux des personnes concernées, la banque doit l’annoncer au PFPDT dès que possible conformément à l’art. 24 LPD. Cette obligation s’ajoute à l’obligation d’annonce envers la FINMA. Si l’e-mail était chiffré, le risque est généralement faible en cas de mauvais envoi, et une annonce n’est pas nécessaire.

## Les effets juridiques d’une signature S/MIME

### Pas une signature au sens du CO

La ZertES distingue la signature électronique, la signature électronique avancée, la signature électronique réglementée et la signature électronique qualifiée, ainsi que le sceau électronique réglementé pour les entreprises. Selon l’art. 14 al. 2bis CO, seule la signature électronique qualifiée assortie d’un horodatage qualifié est assimilée à la signature manuscrite.

Les certificats S/MIME de SwissSign sont délivrés conformément aux règles du CA/Browser Forum et aux politiques ETSI LCP (Silver) ou NCP (Gold). Ils ne sont pas des certificats qualifiés au sens de la ZertES. Une banque ne peut donc pas faire des déclarations valables quant à la forme par e-mail signé S/MIME lorsque la loi ou un contrat exige la forme écrite, même avec Gold.

### Signature personnelle et signature de passerelle

Une signature avancée suppose notamment que la clé de signature soit sous le contrôle exclusif de la personne signataire. Lorsqu’une passerelle signe de manière centralisée tous les e-mails sortants avec des clés stockées sur la passerelle, cette condition n’est pas remplie. La signature prouve alors que l’e-mail provient du système de la banque et n’a pas été modifié, mais pas qu’une personne déterminée l’a validé. Son contenu correspond ainsi davantage à un sceau de l’organisation, sans satisfaire aux exigences d’un sceau réglementé au sens de la ZertES.

### Force probante

Selon l’art. 177 CPC, un e-mail est un titre, et le tribunal apprécie librement sa force probante conformément à l’art. 157 CPC. Une signature S/MIME valide montre que l’e-mail n’a pas été modifié depuis sa signature et qu’il a été signé avec la clé d’un certificat déterminé. Ce que cela indique quant à l’expéditeur dépend de ce que l’autorité de certification a vérifié avant la délivrance. C’est là que réside la différence juridiquement pertinente entre Silver et Gold.

## Silver ou Gold d’un point de vue juridique

Depuis le 1er septembre 2023, les `S/MIME Baseline Requirements` du CA/Browser Forum s’appliquent aux certificats S/MIME publiquement reconnus. SwissSign associe ses produits à ces types de validation dans les profils de certificats (CPR S/MIME) :

| Produit | Type selon le CA/B Forum | Ce que SwissSign vérifie | Ce qui figure dans le certificat | Ce que la signature atteste |
|---|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Contrôle de la boîte aux lettres | Uniquement l’adresse e-mail | Une personne ayant accès à cette boîte aux lettres |
| E-Mail ID Gold | `Sponsor-validated` | Organisation et personne | Nom, organisation, identifiant d’organisation, pays, adresse e-mail | Une personne vérifiée de cette banque |
| Organization Validated | `Organization-validated` | Organisation | Nom de l’organisation, aucune personne | Une boîte aux lettres de cette banque |

Sur le plan cryptographique, les niveaux sont équivalents. Pour le chiffrement et donc pour les obligations de protection découlant du secret bancaire, de la Circulaire FINMA et de la LPD, le niveau ne joue donc aucun rôle.

Pour la signature, la différence est considérable. Toute personne pouvant recevoir des e-mails à une adresse peut obtenir un certificat Silver. Quiconque enregistre un domaine au nom similaire, par exemple `musterbank-ch.com` au lieu de `musterbank.ch`, peut ainsi envoyer des e-mails signés de manière tout aussi valable que les véritables e-mails de la banque. Dans un litige portant sur l’origine bancaire d’un e-mail, une signature Silver apporte donc peu de preuves. En revanche, un certificat Gold ou Organization Validated contient le nom de la banque vérifié par SwissSign. Avec Gold, s’ajoute le rattachement à une personne dont l’identité est confirmée par la banque, en tant qu’autorité d’enregistrement, dans le cadre de la PKI gérée.

## Conservation : CO et Olico

Selon l’art. 958f CO, les livres et pièces comptables doivent être conservés pendant dix ans, et l’ordonnance concernant la tenue et la conservation des livres de comptes (Olico) exige que les documents conservés sous forme électronique puissent être rendus lisibles pendant cette période. Pour les banques, s’y ajoutent des obligations prudentielles et contractuelles de conservation. Il en résulte deux exigences pour l’exploitation de S/MIME :

- Les e-mails chiffrés doivent pouvoir être déchiffrés pendant toute la durée de conservation. La banque doit donc conserver de manière sûre les clés privées des certificats expirés ou déchiffrer les e-mails avant leur archivage.
- Si une signature doit encore servir de preuve après plusieurs années, il doit rester possible d’établir que le certificat était valide au moment de la signature. Les résultats de vérification ou les informations de révocation au moment de la signature doivent donc être archivés également.

**Note UE :** dans l’UE également, conformément à l’art. 25, par. 2 eIDAS, seule la signature électronique qualifiée est assimilée à la signature manuscrite. Depuis le 17 janvier 2025, le règlement DORA s’applique en outre aux entreprises financières de l’UE, avec ses propres exigences de gestion des risques liés aux TIC.

## Recommandation

- **Définir une directive :** la banque définit dans une directive interne quelles données ne peuvent être envoyées que chiffrées, quels procédés sont autorisés à cet effet et quels niveaux de certificat elle utilise. C’est la base permettant de démontrer le respect des exigences de la Circulaire FINMA et de la LPD.
- **Gold pour les boîtes aux lettres personnelles, Organization Validated pour les boîtes fonctionnelles :** seuls ces niveaux identifient la banque comme expéditeur dans le certificat et ont donc une force probante allant au-delà de l’accès à une boîte aux lettres. Silver ne convient pas à la communication avec les clients.
- **Clarifier les consentements :** toute personne qui communique avec des clients par e-mail non chiffré a besoin de leur consentement documenté. Il ne remplace pas le chiffrement vis-à-vis des tiers et des partenaires commerciaux.
- **Ne pas envoyer par e-mail les déclarations soumises à une forme :** les déclarations exigeant la forme écrite nécessitent une signature électronique qualifiée ou une signature manuscrite.
- **Vérifier les prestataires :** les fournisseurs de passerelles et la PKI gérée de l’autorité de certification relèvent du ch. marg. 82 de la Circulaire FINMA et, en cas de localisation à l’étranger, également du ch. marg. 79.
- **Archiver les clés et les données de vérification :** les clés privées pour le déchiffrement et les données nécessaires à la vérification des signatures doivent rester disponibles pendant toute la durée de conservation.
- **Combiner les processus d’annonce :** un mauvais envoi de données clients peut déclencher simultanément une annonce à la FINMA (ch. marg. 81) et au PFPDT (art. 24 LPD). Ces deux obligations peuvent être intégrées à un processus commun de gestion des incidents.

## Sources

1.  [Circulaire FINMA 2023/1 « Risques opérationnels et résilience : banques »](https://www.finma.ch/de/~/media/finma/dokumente/dokumentencenter/myfinma/rundschreiben/finma-rs-2023-01-20221207.pdf): ch. marg. 73 à 82 sur les données critiques, en vigueur depuis le 1er janvier 2024.

2.  [datenrecht.ch : Circ.-FINMA 2023/1 avec explications](https://datenrecht.ch/wp-content/uploads/FINMA-RS-OpRisk-%E2%80%93-Text-revidiert-1.1.2024.pdf): texte révisé avec les explications de la FINMA issues de la consultation, notamment sur le chiffrement des données comme mesure de protection.

3.  [Loi sur les banques (LB)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): art. 47 sur le secret bancaire.

4.  [Swissbanking : Secret bancaire et cloud, expertise 2019](https://www.swissbanking.ch/_Resources/Persistent/b/1/3/3/b13339fe5dcd344e715dbe0a899afe3c8c56ed0b/SBVg_Bankgeheimnis_und_Cloud_Gutachten_LLAG_2019_DE.pdf): avis juridique sur l’interprétation de l’art. 47 LB, notamment sur le consentement du client.

5.  [Loi fédérale sur la protection des données (LPD)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 8 sur la sécurité des données, art. 24 sur l’annonce des violations de la sécurité des données.

6.  [Ordonnance sur la protection des données (OPDo)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): art. 1 à 6 avec les principes et objectifs de protection de la sécurité des données.

7.  [Loi fédérale sur la signature électronique (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): types de signatures et sceaux électroniques, exigences applicables aux certificats qualifiés.

8.  [Code des obligations (CO)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): art. 14 al. 2bis sur l’assimilation de la signature électronique qualifiée à la signature manuscrite, art. 958f sur la conservation.

9.  [Code de procédure civile (CPC)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): art. 157 sur la libre appréciation des preuves, art. 177 sur la notion de titre.

10.  [SwissSign CPR S/MIME : Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): profils de certificats de SwissSign (état au 18.09.2025), association de Silver, Gold et Organization Validated aux types de validation et aux champs Subject.

11.  [CA/Browser Forum : S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): règles contraignantes pour les certificats S/MIME publiquement reconnus, définition des types de validation.
