---
title: "S/MIME pour les banques : certificat Silver ou Gold de SwissSign ?"
navTitle: "S/MIME Silver/Gold"
description: "Une banque qui signe et chiffre les e-mails de ses clients avec S/MIME a besoin de certificats Gold pour les boîtes aux lettres personnelles et de certificats validés par l’organisation pour les boîtes aux lettres fonctionnelles. Silver chiffre tout aussi fortement, mais n’identifie pas la banque comme expéditeur. Ce que SwissSign vérifie, ce qui figure dans le certificat et quelles règles s’appliquent en Suisse."
date: "2026-11-16"
kategorie: "Chiffrement des e-mails"
timeToRead: "9 min de lecture"
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
translationSourceHash: f95529dd7c72d4d783714affbcf18d96e37adf66a52379b94212034dda610b74
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T14:47:58.623Z
translationReview: required
url: https://rafaelpfister.ch/fr/blog/s-mime-avec-les-banques-certificat-silver-ou-gold-de-swisssign
---

# S/MIME pour les banques : certificat Silver ou Gold de SwissSign ?

En bref : une banque qui signe et chiffre avec S/MIME des e-mails destinés à ses clientes, clients et partenaires commerciaux devrait équiper les boîtes aux lettres personnelles de certificats Gold et les boîtes aux lettres fonctionnelles de certificats validés par l’organisation de SwissSign. Silver chiffre tout aussi fortement, mais le certificat ne contient que l’adresse e-mail et non le nom de la banque. Ni la loi ni la FINMA n’imposent un niveau de certificat.

La question du niveau de certificat requis pour l’envoi d’e-mails revient régulièrement dans mes projets avec des établissements financiers. Elle se pose généralement au début d’un projet de passerelle ou lorsque des clients professionnels exigent que les documents confidentiels ne soient plus envoyés que chiffrés. Pour y répondre, il faut distinguer deux fonctions de S/MIME : le chiffrement, qui protège le contenu, et la signature, qui prouve au destinataire qui a envoyé l’e-mail.

**Focus Suisse :** cet article traite de la situation juridique en Suisse. Lorsque les règles de l’UE diffèrent, une remarque l’indique.

## Ce que SwissSign vérifie pour Silver et Gold

Depuis le 1er septembre 2023, les `S/MIME Baseline Requirements` du CA/Browser Forum s’appliquent aux certificats S/MIME publiquement reconnus. Ils distinguent quatre types de validation, selon ce que l’autorité de certification vérifie avant la délivrance. Dans les profils de certificats publiés (CPR S/MIME), SwissSign attribue ses produits à ces types :

| Produit | Type selon le CA/B Forum | Ce qui est vérifié | Ce qui figure dans le certificat |
|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Contrôle de la boîte aux lettres | uniquement l’adresse e-mail |
| E-Mail ID Gold | `Sponsor-validated` | Organisation et personne | Prénom et nom (ou pseudonyme), organisation, identifiant d’organisation, pays, adresse e-mail |
| Organization Validated | `Organization-validated` | Organisation | Nom de l’organisation, sans personne |

Pour Silver, SwissSign envoie un code de confirmation à l’adresse demandée. Toute personne pouvant recevoir l’e-mail obtient le certificat. L’identité de la personne et l’organisation ne sont pas vérifiées, et l’adresse figure également dans le certificat comme `Common Name`.

Chez SwissSign, Gold est proposé exclusivement comme `Sponsor-validated`. SwissSign vérifie l’organisation, à laquelle la personne est rattachée. Son identité est vérifiée soit par SwissSign, soit par l’organisation elle-même en tant qu’autorité d’enregistrement, généralement dans le cadre d’une Managed PKI. Pour les boîtes aux lettres fonctionnelles telles que `kundendienst@` ou `zahlungsverkehr@`, le profil `Organization-validated` est prévu ; la banque y figure comme organisation dans le certificat, mais aucune personne.

Sur le plan cryptographique, les niveaux ne diffèrent pas. Tous les profils utilisent des clés RSA d’au moins 2048 bits, l’usage de clé `digitalSignature` et `keyEncipherment` ainsi que l’usage étendu `id-kp-emailProtection`.

## Ce que la différence implique pour une banque en tant qu’expéditeur

| | Silver | Gold | Organization Validated |
|---|---|---|---|
| Chiffrer et déchiffrer les e-mails | oui | oui | oui |
| La signature affiche dans le logiciel de messagerie | Adresse e-mail | Nom et banque | Banque |
| Ce qu’atteste la signature | L’expéditeur a accès à la boîte aux lettres | Personne vérifiée de cette banque | Boîte aux lettres de cette banque |
| Usage dans une banque | inadapté à la communication avec les clients | boîtes aux lettres personnelles | boîtes aux lettres fonctionnelles |

La différence concerne la signature. Une banque signe ses e-mails sortants afin que les destinataires puissent vérifier que le message provient réellement d’elle et n’a pas été modifié en cours de route. Avec Silver, cela n’est possible que de manière limitée :

- Toute personne qui enregistre un domaine au nom similaire, par exemple `musterbank-ch.com` au lieu de `musterbank.ch`, obtient un certificat Silver pour les adresses de ce domaine sans vérification supplémentaire. Ses e-mails de phishing sont alors signés de manière tout aussi valide que les véritables e-mails de la banque, et le certificat ne contient dans les deux cas qu’une adresse e-mail.
- Toute personne qui prend le contrôle d’une boîte aux lettres de la banque peut également demander un certificat Silver pour cette adresse.

Un certificat Gold ou Organization Validated contient le nom vérifié de la banque dans le champ `Organization`. Seule une entité ayant passé avec succès la vérification de l’organisation par SwissSign obtient cette mention. Pour les destinataires qui consultent la signature, c’est la preuve pertinente.

Pour le chiffrement, le niveau ne joue aucun rôle. Un e-mail chiffré avec Silver est tout aussi bien protégé qu’un e-mail chiffré avec Gold.

## Le chiffrement requiert les certificats des destinataires

Pour chiffrer un e-mail avec S/MIME, la banque a besoin de la clé publique du destinataire. Le propre certificat de la banque n’est pas utilisé à cette fin ; il sert à la signature et à la réception de réponses chiffrées.

La plupart des clients particuliers ne disposent pas d’un certificat S/MIME. Les banques leur transmettent donc généralement les e-mails confidentiels via une passerelle de chiffrement, telle que SEPPmail (GINA) ou Kiteworks/Totemomail, qui met le message à disposition dans un portail web ou sous forme de pièce jointe chiffrée. Le chiffrement S/MIME est surtout utilisé avec les clients professionnels et partenaires qui exploitent eux-mêmes des certificats ou une passerelle. Lorsque les deux parties utilisent une passerelle, des certificats de domaine ou du TLS forcé entre les serveurs de messagerie suffisent souvent.

## Certificats en exploitation : Managed PKI et passerelle

Pour plus d’une poignée de boîtes aux lettres, une banque obtient ses certificats via la Managed PKI de SwissSign. La banque agit alors comme autorité d’enregistrement et confirme elle-même l’identité des collaborateurs ; cela correspond au type `Sponsor-validated`. Des passerelles telles que SEPPmail et Kiteworks/Totemomail peuvent, via une interface avec la Managed PKI, demander et renouveler automatiquement les certificats et les attribuer aux boîtes aux lettres.

Trois points connus dans les projets provoquent des problèmes en exploitation :

- Les demandes Gold pour les boîtes aux lettres fonctionnelles échouent parce que la passerelle reprend le prénom et le nom depuis l’annuaire et que SwissSign rejette les termes génériques tels que « Support » ou « Helpdesk » dans le champ `GivenName`. Pour ces boîtes aux lettres, le profil `Organization-validated` peut être configuré dans la passerelle.
- Les renouvellements automatiques peuvent échouer pendant des semaines sans que cela ne soit remarqué, jusqu’à ce que les premiers certificats expirent et que les clients ne puissent plus envoyer d’e-mails chiffrés. Les dates d’expiration des certificats émis et les messages d’erreur des opérations de renouvellement devraient donc être surveillés.
- La passerelle signe ses demandes à la Managed PKI avec son propre certificat de l’autorité d’enregistrement. Lorsque celui-ci expire, toute l’automatisation s’arrête.

## Exigences légales en Suisse

Aucune loi suisse n’impose S/MIME, et aucune ne distingue les niveaux de certificat. Les obligations d’une banque concernent la protection des données des clients ; elle décide elle-même de la manière de les mettre en œuvre.

| Réglementation | Contenu | Importance pour l’envoi d’e-mails de la banque |
|---|---|---|
| LB art. 47 | Secret bancaire | Les données des clients ne doivent pas être divulguées sans autorisation ; l’envoi non chiffré par Internet est un risque que la banque doit évaluer |
| FINMA-Circ. 2023/1 | Protection des données critiques | Protection également lors de la transmission, neutre sur le plan technologique, sans exigence relative aux certificats |
| LPD art. 8 et OPDo art. 1 à 6 | Sécurité des données adaptée au risque | Le chiffrement est un moyen courant, mais aucune procédure précise n’est prescrite |
| ZertES et CO art. 14 al. 2bis | Seule la signature électronique qualifiée est assimilée à la signature manuscrite | Les signatures S/MIME ne remplacent pas une signature |

### Secret bancaire et FINMA

Le secret bancaire selon l’art. 47 LB oblige la banque, ses organes et ses collaborateurs à ne pas divulguer sans autorisation les données des clients. La circulaire FINMA 2023/1 « Risques opérationnels et résilience : banques » exige que les banques protègent leurs données critiques durant tout leur cycle de vie, y compris lors de leur transmission. La banque définit elle-même la manière dont cela est effectué dans ses propres directives. Le choix du niveau de certificat fait donc partie de la politique interne et devrait y être documenté.

### Protection des données : LPD et OPDo

L’art. 8 LPD oblige les responsables à garantir une sécurité des données adaptée au risque. L’ordonnance sur la protection des données précise cette obligation par des objectifs de protection tels que la confidentialité et l’intégrité, mais ne prescrit aucune procédure. S/MIME, une passerelle avec livraison par webmail ou une plateforme d’échange répondent tous de la même manière à cette exigence, à condition d’être utilisés de façon adaptée au risque.

### Signature électronique : ZertES et CO

La ZertES distingue la signature électronique, la signature électronique avancée, la signature électronique réglementée et la signature électronique qualifiée. Selon l’art. 14 al. 2bis CO, seule la signature électronique qualifiée avec horodatage qualifié est assimilée à la signature manuscrite. Les certificats S/MIME de SwissSign sont émis selon les règles du CA/Browser Forum et les politiques ETSI LCP (Silver) ou NCP (Gold) et ne sont pas des certificats qualifiés au sens de la ZertES. Les contrats ou déclarations pour lesquels la forme écrite est prescrite ne peuvent pas être valablement établis par un e-mail signé S/MIME, même avec Gold.

**Remarque UE :** dans l’UE également, selon l’art. 25, par. 2, eIDAS, seule la signature électronique qualifiée est assimilée à la signature manuscrite.

## Recommandation

- **Boîtes aux lettres personnelles** des conseillers clientèle et autres collaborateurs en contact avec la clientèle : Gold (`Sponsor-validated`) via la Managed PKI.
- **Boîtes aux lettres fonctionnelles :** `Organization-validated`, afin que la banque figure comme expéditeur dans le certificat.
- Ne pas utiliser **Silver** pour la communication avec les clients, car la signature n’identifie pas la banque comme expéditeur.
- **Chiffrement pour les clients particuliers** via la livraison push ou par portail de la passerelle ; S/MIME pour les destinataires disposant de leur propre certificat ou passerelle.
- **La signature et la vérification de l’expéditeur** se complètent : la signature S/MIME ne remplace pas SPF, DKIM et DMARC, mais elle affiche l’organisation vérifiée au destinataire dans son logiciel de messagerie.
- **Fiche d’information pour les clients :** décrivez quels certificats la banque utilise, comment les clients peuvent vérifier une signature et quels niveaux de certificat la banque accepte de la part de clients souhaitant eux-mêmes envoyer des messages chiffrés. En l’absence d’une telle fiche, la question est adressée au conseil clientèle, qui ne peut généralement pas y répondre.

## Sources

1.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): profils de certificats de SwissSign (état au 18.09.2025), attribution de Silver, Gold et Organization Validated aux types de validation, champs Subject et OID de politiques.

2.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): règles contraignantes pour les certificats S/MIME publiquement reconnus, définition des quatre types de validation.

3.  [SEPPmail: Managed PKI SwissSign](https://docs.seppmail.com/en/07_mi_10_mpki_11_swisssign.html): connexion de la passerelle SEPPmail à la Managed PKI de SwissSign.

4.  [Loi sur les banques (LB)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): art. 47 relatif au secret bancaire.

5.  [FINMA : Circulaires](https://www.finma.ch/de/dokumentation/rundschreiben/): aperçu comprenant la circulaire 2023/1 « Risques opérationnels et résilience : banques », en vigueur depuis le 1er janvier 2024.

6.  [Loi fédérale sur la protection des données (LPD)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 8 relatif à la sécurité des données.

7.  [Ordonnance sur la protection des données (OPDo)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): art. 1 à 6 contenant les principes et objectifs de protection de la sécurité des données.

8.  [Loi fédérale sur la signature électronique (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): types de signatures électroniques et exigences applicables aux certificats qualifiés.

9.  [Code des obligations (CO)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): art. 14 al. 2bis relatif à l’assimilation de la signature électronique qualifiée à la signature manuscrite.
