---
title: "Le prestataire de caisses de pension attaqué n’a tout simplement aucun enregistrement DMARC"
navTitle: "DMARC caisses de pension"
description: "Le prestataire de caisses de pension PK Softech, attaqué fin septembre 2026, n’a toujours aucun enregistrement DMARC. Un contrôle passif de 66 institutions de prévoyance et de 30 prestataires montre qu’il ne s’agit pas d’un cas isolé : les éditeurs de logiciels de gestion sont particulièrement non protégés."
date: "2026-10-18"
kategorie: "SMTP et flux de messagerie"
timeToRead: "9 min de lecture"
themen:
  - smtp-mailflow
produkte: []
protokolle:
  - "mail-auth"
  - "dns"
slug: "le-prestataire-de-caisses-de-pension-attaque-n-a-aucun-enregistrement-dmarc"
translationId: "article-a34017b39fecf271"
zielgruppe: "entscheider"
translationOf: dmarc-pensionskassen-schweiz
translationSourceHash: 357621c8e85052ed11d792c9e6f911df15e6e32a0ad4cf0c5d92ba1fdbd9f906
translationModel: gpt-5.6-terra
translatedAt: 2026-10-25T19:33:37.109Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/le-prestataire-de-caisses-de-pension-attaque-n-a-aucun-enregistrement-dmarc
---

Fin septembre 2026, PK Softech AG a été attaquée, un éditeur de logiciels dont plusieurs caisses de pension suisses utilisent la solution de prévoyance. Des données ont ainsi été exfiltrées. Selon les informations publiées, les entités concernées comprennent notamment la caisse de pension de la Confédération Publica, la caisse de pension de Migros et la caisse de pension de la ville de Zurich, ainsi que d’autres institutions. Selon la caisse, il s’agit de nom, numéro AVS, état civil, adresse et IBAN, ainsi que de données salariales et de prévoyance.

Cette exfiltration est grave en soi. Elle concerne des données d’identification et financières d’une grande partie de la population active et, via Publica, de larges pans de l’administration fédérale. Des données de cette quantité et de cette qualité ne posent pas seulement un problème à chaque personne concernée. Elles permettent l’usurpation d’identité, la tromperie ciblée et l’extorsion à grande échelle et constituent donc un danger pour l’ordre public et la sécurité nationale. Les avoirs de prévoyance eux-mêmes n’ont pas été touchés, mais la plateforme d’information Cybercrimepolice.ch avait déjà mis en garde contre des appels frauduleux utilisant les données dérobées.

À cela s’ajoute un constat difficile à expliquer pour un prestataire de cette catégorie : **PK Softech AG n’a toujours aucun enregistrement DMARC.** Son domaine dispose d’un enregistrement SPF strict, mais d’aucune politique DMARC (vérifié le 18.10.2026). Il est donc possible d’usurper l’adresse d’expéditeur du prestataire attaqué sans qu’une plateforme de messagerie destinataire ne le rejette de manière fiable. Après une exfiltration de données, c’est la situation initiale la plus défavorable qui soit. Quiconque connaît le nom, l’adresse et les coordonnées bancaires d’une personne assurée et peut en outre falsifier un e-mail au nom du prestataire ou de la caisse peut rédiger un message qui se distingue à peine d’un vrai, par exemple une demande de mettre à jour l’IBAN pour le prochain versement de rente. C’est précisément contre cette falsification que DMARC protège.

PK Softech n’est pas un cas isolé, mais l’exemple le plus flagrant d’un schéma récurrent. J’ai vérifié dans quelle mesure les institutions de prévoyance suisses et leurs prestataires typiques sont préparés en matière d’authentification des e-mails.

## Méthode de contrôle

L’évaluation couvre 66 institutions de prévoyance du deuxième pilier, des grandes caisses de droit public aux fondations collectives des assureurs, en passant par de plus petites institutions communes. Seuls les enregistrements DNS publics ont été contrôlés : SPF, DKIM, DMARC, les enregistrements MX ainsi que DNSSEC et DANE (enregistrements TLSA des serveurs de messagerie, vérifiés le 19.10.2026). Ce sont les mêmes informations que tout système de messagerie destinataire consulte pour chaque message entrant.

Le contrôle a été purement passif. Seul le DNS public a été interrogé, aucune connexion n’a été établie avec les systèmes des institutions, aucune tentative de connexion n’a été effectuée et aucune donnée personnelle n’a été collectée. L’évaluation ne reflète que ce qui est de toute façon visible par tous à partir de sources publiques. Les fondations collectives de groupe sont regroupées sous le domaine principal de leur organisme responsable ; certaines fondations affiliées peuvent différer.

## Le résultat

| Caractéristique | Part des 66 institutions |
|---|---|
| SPF présent | 88 % |
| SPF strict (`-all`) | 59 % |
| DMARC présent | 59 % |
| DMARC `p=reject` (protection complète) | 20 % |
| DMARC `p=quarantine` | 26 % |
| DMARC `p=none` (observation uniquement) | 14 % |
| pas de DMARC | 41 % |
| DKIM identifiable | 55 % |
| DNSSEC | 38 % |
| DANE | 9 % |

Trois points ressortent.

**Seuls 20 % sont efficacement protégés par DMARC `p=reject`.** Seule cette politique indique à une plateforme de messagerie destinataire de rejeter effectivement les messages falsifiés. Dans tous les autres cas, un faux message finit, en cas de doute, dans la boîte de réception de la personne assurée.

**41 % n’ont pas de DMARC.** Leur domaine d’expéditeur peut aisément être utilisé pour des e-mails frauduleux, et l’institution ne le saura même pas, car sans DMARC, la fonction de reporting fait également défaut. Il est frappant que cela inclue aussi des caisses qui exploitent leur messagerie avec Microsoft 365, où DMARC peut être mis en place avec peu d’efforts.

**DMARC `p=none` n’est pas une protection.** Les 14 % de ce groupe ont défini une politique DMARC, mais l’ont laissée au niveau d’observation. Cela fournit des rapports, mais ne rejette pas le moindre message falsifié. En pratique, il s’agit d’une étape intermédiaire qui reste souvent en place pendant des mois ou des années.

Le chiffrement du transport présente une image similaire : seules 6 des 66 institutions (9 %) sécurisent leurs serveurs de messagerie avec DANE. Un serveur de messagerie expéditeur vérifie ainsi, à l’aide d’un enregistrement signé par DNSSEC, qu’il est connecté de manière chiffrée au bon destinataire. DNSSEC est une condition préalable, et 38 % des institutions l’ont déjà activé. Pour celles-ci, DANE est l’étape suivante logique.

## Toutes les données à explorer

L’infographie suivante montre l’état vérifié des 66 institutions de prévoyance et 30 prestataires. Les filtres permettent de limiter l’affichage selon le statut DMARC, la catégorie et le fournisseur de messagerie, et chaque entrée peut être développée pour consulter les résultats DNS complets.

<div data-dmarc-infographic></div>

## Comparaison : les plus grandes entreprises suisses

La position des institutions de prévoyance par rapport au reste de l’économie est illustrée par l’[indice suisse de sécurité des e-mails](/open-data/mail-security-index). Il contrôle quotidiennement, avec la même méthode passive, plus de 1000 des plus grandes entreprises suisses ainsi que la Confédération, les cantons et les communes, avec une carte interactive par canton, des comparaisons par secteur et fournisseur de messagerie, et une page détaillée pour chaque organisation. Toutes les institutions de prévoyance et tous les prestataires contrôlés ici y figurent avec leur propre profil, regroupés dans le secteur [institutions de prévoyance](/open-data/mail-security-index/branche/vorsorgeeinrichtungen) et mis à jour quotidiennement. Dans l’infographie ci-dessus, le lien de l’entrée développée mène directement au profil correspondant.

L’écart est net : parmi les 518 entreprises de l’indice, 38 % utilisaient le 18.10.2026 DMARC avec `p=reject`, contre 20 % pour les institutions de prévoyance. 5 % des entreprises n’avaient aucun DMARC, contre 41 % des institutions de prévoyance.

## Les fournisseurs sont le sujet le plus important

L’incident de fin septembre n’a pas touché une caisse, mais un prestataire commun. J’ai donc contrôlé, avec la même méthode passive, 30 fournisseurs typiques : éditeurs de logiciels de gestion, entreprises assurant la direction externalisée, actuariat et conseil, banques dépositaires et prestataires informatiques.

En moyenne, les prestataires sont légèrement mieux lotis que les caisses : 40 % utilisent DMARC avec `p=reject`, contre 20 % des institutions. Mais la répartition entre les catégories est déterminante.

| Catégorie | avec DMARC `p=reject` |
|---|---|
| Actuariat et conseil | bon |
| Banques dépositaires | bon |
| Prestataires informatiques | mitigé |
| Direction externalisée | faible |
| Logiciels de gestion | le plus faible |

Les éditeurs de logiciels de gestion constituent le groupe le plus faible. Aucun des fournisseurs de logiciels contrôlés n’utilise DMARC avec une efficacité complète, et le fournisseur attaqué PK Softech n’a aucun enregistrement DMARC. Or, c’est précisément cette catégorie qui traite les données de prévoyance et est profondément reliée aux caisses via des interfaces, et c’est aussi la catégorie concernée par l’incident actuel.

Il en découle le véritable point de vigilance : une caisse avec `p=reject` est bien protégée tant que l’attaquant cherche à usurper son domaine. S’il peut à la place falsifier le domaine d’un prestataire mal protégé par lequel les communications avec les assurés passent déjà, l’e-mail est tout aussi crédible. La sécurisation de son propre domaine n’est donc que la première étape. La deuxième consiste à imposer le même niveau aux prestataires les plus importants, idéalement par contrat.

## Ce que cela signifie pour le conseil de fondation

La responsabilité de la sécurité de l’information incombe à l’organe suprême de l’institution de prévoyance. Une exfiltration de données entraîne en outre des obligations concrètes en vertu de la loi révisée sur la protection des données : une violation de la sécurité des données doit être annoncée au PFPDT lorsqu’elle est susceptible d’entraîner un risque élevé pour les personnes concernées, et les personnes concernées doivent être informées lorsque cela est nécessaire à leur protection.

Dans cette situation, l’authentification des e-mails est la mesure individuelle la plus efficace pour le moindre effort. SPF, DKIM et DMARC ne nécessitent ni licence ni nouveau logiciel. Ils reposent sur des enregistrements DNS et une configuration propre de la plateforme de messagerie existante. La protection agit immédiatement pour chaque personne assurée, sans qu’elle ait elle-même quoi que ce soit à faire.

## Ce qu’il faut faire

Pour les institutions sans protection efficace, une voie claire mène à l’objectif :

1. **Vérifier SPF et le définir de manière stricte.** L’enregistrement SPF liste les systèmes d’envoi autorisés et se termine par `-all`, afin que tous les autres expéditeurs soient considérés comme non autorisés.
2. **Activer DKIM.** Chaque message sortant est signé cryptographiquement. Dans Microsoft 365 et les passerelles de messagerie courantes, il s’agit d’un réglage, pas d’un projet.
3. **Introduire DMARC et le renforcer progressivement.** Commencer avec `p=none` et un reporting actif afin de recenser le trafic de messagerie légitime. Puis `p=quarantine`, et enfin `p=reject`. L’ensemble du processus prend quelques semaines selon l’environnement, pas des mois.
4. **Analyser les rapports.** Les rapports DMARC montrent qui envoie des e-mails au nom du domaine et révèlent aussi bien les systèmes légitimes oubliés que les abus.
5. **Au moins examiner DANE et le chiffrement des e-mails.** DANE sécurise le chiffrement du transport entre serveurs de messagerie via DNSSEC. Il est possible de déterminer avec peu d’efforts si sa propre plateforme de messagerie et son fournisseur DNS le prennent en charge. Pour les messages contenant des données salariales et de prévoyance, il faut en outre définir comment ils sont envoyés chiffrés de bout en bout, par exemple au moyen d’une passerelle de messagerie avec S/MIME ou d’une transmission sécurisée via un portail web.
6. **Obliger les prestataires à respecter ces exigences.** Les fournisseurs de logiciels, les directions externalisées et les partenaires informatiques devraient démontrer le même niveau pour SPF, DKIM et DMARC. Cela doit figurer dans les contrats et dans l’évaluation régulière des fournisseurs.

## Conclusion

Les attaques récentes visaient un prestataire, et non les caisses elles-mêmes. Mais les dommages consécutifs surviennent là où un domaine d’expéditeur n’est pas protégé, tant auprès de la caisse que de ses fournisseurs. Quatre institutions de prévoyance contrôlées sur cinq ne sont pas entièrement protégées contre les expéditeurs falsifiés, deux sur cinq n’ont aucun DMARC, et les éditeurs de logiciels de gestion obtiennent précisément les plus mauvais résultats. Cette lacune est connue, peu coûteuse à combler et produit un effet immédiat. Après une exfiltration de données concernant le nom, l’adresse et les coordonnées bancaires, il ne s’agit pas d’un détail technique, mais d’un élément du devoir de diligence de l’organe suprême, et ce tout au long de la chaîne d’approvisionnement.

## Sources

1.  [SRF : Caisse de pension de la Confédération, comment des données sensibles ont-elles pu se retrouver chez une société de logiciels externe ?](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen) : contexte de l’incident, types de données concernés et rôle du fournisseur de logiciels.

2.  [cash.ch (AWP) : attaque contre un fournisseur de Publica, des données salariales pourraient être concernées](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247) : Publica désigne PK Softech AG auprès d’AWP comme l’entreprise attaquée.

3.  [Netzwoche : la caisse de pension de la Confédération Publica signale une exfiltration de données](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss) : information sur l’attaque contre le fournisseur de logiciels et l’exfiltration de données.

4.  [Mimikama : la police suisse met en garde contre de possibles appels d’investissement après une exfiltration de données](https://www.mimikama.org/?p=436628) : mise en contexte de l’avertissement concernant des appels frauduleux utilisant les données exfiltrées.

5.  [PFPDT : DataBreach](https://www.edoeb.admin.ch/de/databreach) : obligation d’annonce selon l’art. 24 LPD, portail d’annonce et guide sur la marche à suivre en cas de violation de la sécurité des données.
