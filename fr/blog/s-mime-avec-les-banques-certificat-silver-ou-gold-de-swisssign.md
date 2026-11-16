---
title: "S/MIME avec les banques : certificat Silver ou Gold de SwissSign ?"
navTitle: "S/MIME Silver/Gold"
description: "Pour les e-mails chiffrés avec une banque, un certificat Silver de SwissSign suffit techniquement. Gold confirme en plus le nom et l’organisation de l’expéditeur. Découvrez ce que vérifie chaque niveau, ce qui figure dans le certificat et quelles règles juridiques s’appliquent en Suisse."
date: "2026-11-16"
kategorie: "Chiffrement des e-mails"
timeToRead: "8 min de lecture"
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
url: https://rafaelpfister.ch/fr/blog/s-mime-avec-les-banques-certificat-silver-ou-gold-de-swisssign
translationSourceHash: cbcb890de323f2b9db40a5682192296524250e7bc8c23a67c6f643bcba605a14
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T14:03:41.565Z
translationReview: required
---

# S/MIME avec les banques : certificat Silver ou Gold de SwissSign ?

En bref : ni une loi suisse ni la FINMA n’impose un niveau de certificat précis pour S/MIME. Pour le chiffrement, un certificat Silver suffit entièrement sur le plan technique. Gold confirme en plus le nom et l’organisation de l’expéditeur et constitue le choix approprié lorsque la banque ne donne aucune consigne.

La question de savoir si un certificat Silver ou Gold de SwissSign est nécessaire pour les échanges d’e-mails avec une banque revient régulièrement dans mes projets. La banque ne fournit généralement aucune fiche d’information à ce sujet, et le ou la conseillère clientèle ne peut pas répondre à la question. La réponse dépend de l’usage du certificat : pour le chiffrement ou comme preuve de l’identité de l’expéditeur d’un e-mail.

**Focus Suisse :** cet article traite de la situation juridique en Suisse. Lorsqu’elles diffèrent, les règles de l’UE font l’objet d’une indication spécifique.

## Ce que SwissSign vérifie pour Silver et Gold

Depuis le 1er septembre 2023, les `S/MIME Baseline Requirements` du CA/Browser Forum s’appliquent aux certificats S/MIME publiquement reconnus. Elles distinguent quatre types de validation selon ce que l’autorité de certification vérifie avant l’émission. Dans les profils de certificats publiés (CPR S/MIME), SwissSign attribue ses produits à ces types :

| Produit | Type selon le CA/B Forum | Ce qui est vérifié | Ce qui figure dans le certificat |
|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Contrôle de la boîte aux lettres | uniquement l’adresse e-mail |
| E-Mail ID Gold | `Sponsor-validated` | Organisation et personne | Prénom et nom (ou pseudonyme), organisation, identifiant de l’organisation, pays, adresse e-mail |
| Organization Validated | `Organization-validated` | Organisation | Nom de l’organisation, sans personne |

Pour Silver, SwissSign envoie un code de confirmation à l’adresse demandée. Quiconque peut recevoir l’e-mail obtient le certificat. L’identité de la personne n’est pas vérifiée et l’adresse figure également dans le certificat comme `Common Name`.

Chez SwissSign, Gold est proposé exclusivement comme `Sponsor-validated`. L’organisation est vérifiée et la personne y est rattachée : son identité est vérifiée soit par SwissSign, soit par l’organisation elle-même en tant qu’autorité d’enregistrement, par exemple dans le cadre d’une PKI gérée. Un profil pour les particuliers sans organisation (`Individual-validated`) ne figure pas dans les profils de certificats.

Pour les boîtes aux lettres fonctionnelles telles que `support@` ou `buchhaltung@`, le profil `Organization-validated` est prévu. Les demandes Gold pour de telles boîtes peuvent échouer, car SwissSign refuse les termes génériques tels que « Support » ou « Helpdesk » dans le champ `GivenName`.

Les niveaux ne diffèrent pas sur le plan cryptographique. Tous les profils utilisent des clés RSA d’au moins 2048 bits, les usages de clé `digitalSignature` et `keyEncipherment` ainsi que l’usage étendu `id-kp-emailProtection`. Un e-mail chiffré avec Silver est tout aussi bien protégé qu’un e-mail chiffré avec Gold.

## Ce que cette différence signifie dans la pratique

| | Silver | Gold |
|---|---|---|
| Chiffrer et déchiffrer des e-mails | oui | oui |
| Affichage de la signature dans Outlook | Adresse e-mail | Nom et organisation |
| Ce que dit la signature | L’expéditeur a accès à la boîte aux lettres | L’expéditeur est une personne vérifiée de cette organisation |
| Effort lors de la demande | Confirmation par e-mail | Justificatif de l’organisation et vérification de l’identité |

La différence apparaît en cas de compromission d’un compte. Quiconque contrôle une boîte aux lettres peut demander un certificat Silver pour cette adresse et envoyer ainsi des e-mails valablement signés. Un certificat Gold exige en revanche que l’organisation et la personne soient vérifiées. Pour une banque qui reçoit par exemple des ordres de paiement ou une correspondance contractuelle par e-mail, une signature Gold a donc davantage de valeur probante.

Cela ne joue aucun rôle pour le chiffrement. La banque a seulement besoin de la clé publique du destinataire, et elle est équivalente dans les deux niveaux.

## Exigences légales en Suisse

Aucune loi suisse n’impose S/MIME, et aucune ne distingue les niveaux de certificat. Il existe des obligations générales de protection des données, dont la mise en œuvre technique reste ouverte.

| Réglementation | Contenu | Lien avec S/MIME |
|---|---|---|
| LPD art. 8 et OPDo art. 1 à 6 | sécurité des données adaptée au risque | Le chiffrement est un moyen courant pour les données personnelles sensibles ; aucune méthode précise n’est prescrite |
| LB art. 47 | secret bancaire | oblige la banque, non sa clientèle |
| Circ.-FINMA 2023/1 | protection des données critiques dans les banques | neutre sur le plan technologique, aucune exigence relative aux certificats |
| SCSE et CO art. 14, al. 2bis | seule la signature électronique qualifiée est assimilée à la signature manuscrite | Silver et Gold ne sont pas des signatures qualifiées |
| OCEI-PCPP | dépôts électroniques auprès des tribunaux et offices des poursuites | signature qualifiée et plateforme de transmission reconnue nécessaires ; S/MIME ne suffit pas |

### Protection des données : LPD et OPDo

L’art. 8 LPD oblige les responsables du traitement et les sous-traitants à garantir une sécurité des données adaptée au risque. L’ordonnance sur la protection des données précise cette obligation par des objectifs de protection tels que la confidentialité et l’intégrité, mais ne mentionne aucune procédure particulière. Le recours à S/MIME, à une passerelle avec remise par webmail ou à une plateforme d’échange doit être évalué au cas par cas.

### Secret bancaire et FINMA

Le secret bancaire prévu à l’art. 47 LB et la circulaire FINMA 2023/1 « Risques opérationnels et résilience : banques » s’adressent à la banque. La circulaire exige que les banques protègent leurs données critiques tout au long de leur cycle de vie, y compris lors de leur transmission. Les banques définissent elles-mêmes, dans leurs propres directives, la manière de le mettre en œuvre. L’acceptation de Silver dépend donc de la politique interne de chaque banque.

### Signature électronique : SCSE et CO

La SCSE distingue la signature électronique, la signature électronique avancée, la signature électronique réglementée et la signature électronique qualifiée. Selon l’art. 14, al. 2bis CO, seule la signature électronique qualifiée accompagnée d’un horodatage qualifié est assimilée à la signature manuscrite. Les certificats S/MIME de SwissSign sont émis selon les règles du CA/Browser Forum et les politiques ETSI LCP (Silver) et NCP (Gold), et ne sont pas des certificats qualifiés au sens de la SCSE.

Un contrat pour lequel la forme écrite est prescrite ne peut donc pas être conclu valablement au moyen d’un e-mail signé S/MIME, même avec Gold. En cas de litige, une signature Gold peut néanmoins servir de preuve ; le tribunal en apprécie librement la valeur.

### Dépôts auprès des tribunaux et autorités

Pour les dépôts électroniques dans les procédures civiles, pénales, de poursuite pour dettes et de faillite, l’OCEI-PCPP exige une signature électronique qualifiée et une transmission par une plateforme de transmission reconnue. Un e-mail S/MIME ne répond pas à ces exigences.

**Note UE :** dans l’UE également, selon l’art. 25, par. 2 eIDAS, seule la signature électronique qualifiée est assimilée à la signature manuscrite.

## Lorsque la banque ne donne aucune consigne

En l’absence de fiche d’information, trois questions permettent de clarifier la situation avec la banque :

1.  La banque exige-t-elle, pour les e-mails signés, un certificat avec identité vérifiée, ou une simple validation de l’adresse e-mail suffit-elle ?

2.  La banque utilise-t-elle une passerelle de chiffrement (par ex. SEPPmail ou Kiteworks/Totemomail) et un certificat de domaine, ou les certificats personnels de ses collaborateurs ?

3.  Comment la banque met-elle ses propres clés publiques à disposition ?

Voici une courte demande à adresser au conseil clientèle, qui peut être transmise à l’informatique :

> Pour nos échanges d’e-mails chiffrés, je vais acquérir un certificat S/MIME. Merci de m’indiquer si vous exigez un certificat avec identité vérifiée (p. ex. SwissSign E-Mail ID Gold avec nom et organisation) ou si un certificat avec simple validation de l’adresse e-mail (Silver) suffit. Je vous prie également de me communiquer votre certificat public ou de m’indiquer si vous utilisez une passerelle avec certificat de domaine.

De nombreuses banques envoient de toute façon les e-mails confidentiels via une passerelle avec remise par webmail. Le destinataire se connecte alors au portail et n’a pas besoin de son propre certificat. Lorsque les deux parties utilisent une passerelle, des certificats de domaine ou TLS imposé entre les serveurs de messagerie suffisent souvent, et les certificats personnels ne sont pas nécessaires.

Si la question reste ouverte, Gold est le choix approprié pour la correspondance d’entreprise. Gold couvre le chiffrement et la signature, et un passage ultérieur de Silver à Gold implique une nouvelle demande ainsi qu’une nouvelle distribution de la clé publique à tous les correspondants. Silver suffit si la banque confirme qu’elle chiffre uniquement les messages et n’évalue pas les signatures. Pour les boîtes aux lettres fonctionnelles, un certificat `Organization-validated` est prévu.

## Sources

1.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): profils de certificats de SwissSign (état au 18.09.2025), attribution de Silver, Gold et Organization Validated aux types de validation, champs Subject et OID de politiques.

2.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): règles contraignantes pour les certificats S/MIME publiquement reconnus, définition des quatre types de validation.

3.  [Loi fédérale sur la protection des données (LPD)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 8 sur la sécurité des données.

4.  [Ordonnance sur la protection des données (OPDo)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): art. 1 à 6 avec les principes et objectifs de protection de la sécurité des données.

5.  [Loi sur les banques (LB)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): art. 47 sur le secret bancaire.

6.  [FINMA : circulaires](https://www.finma.ch/de/dokumentation/rundschreiben/): aperçu comprenant la circulaire 2023/1 « Risques opérationnels et résilience : banques », en vigueur depuis le 1er janvier 2024.

7.  [Loi fédérale sur la signature électronique (SCSE)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): types de signature électronique et exigences applicables aux certificats qualifiés.

8.  [Code des obligations (CO)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): art. 14, al. 2bis sur l’assimilation de la signature électronique qualifiée à la signature manuscrite.
