---
title: "Le prestataire de caisses de pension attaqué n’a aucun enregistrement DMARC"
navTitle: "DMARC caisses de pension"
description: "Le prestataire de caisses de pension PK Softech, attaqué fin septembre 2026, n’a toujours aucun enregistrement DMARC. Un examen passif de 66 institutions de prévoyance et de 30 prestataires montre qu’il ne s’agit pas d’un cas isolé : les fabricants de logiciels de gestion, en particulier, ne sont pas protégés."
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
translationSourceHash: a673909df1426a76c58ced5aaa46d839e480ae867b2ffad468c903d65919e0c3
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T09:14:57.488Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/le-prestataire-de-caisses-de-pension-attaque-n-a-aucun-enregistrement-dmarc
---

Fin septembre 2026, PK Softech AG a été attaquée, un fabricant de logiciels dont plusieurs caisses de pension suisses utilisent la solution de prévoyance. Des données ont été exfiltrées par ce biais. Selon les informations publiées, les entités concernées comprennent notamment la caisse de pension de la Confédération Publica, la caisse de pension de Migros et la caisse de pension de la ville de Zurich, ainsi que d’autres institutions. Selon la caisse, il s’agit de nom, numéro AVS, état civil, adresse et IBAN, ainsi que de données salariales et de prévoyance.

Cette exfiltration est grave en soi. Elle concerne des données d’identification et financières d’une grande partie de la population active et, via Publica, de larges pans de l’administration fédérale. Des données d’une telle quantité et qualité ne posent pas seulement problème aux personnes concernées. Elles permettent l’usurpation d’identité, la tromperie ciblée et l’extorsion à grande échelle, et constituent ainsi une menace pour l’ordre public et la sécurité nationale. Les avoirs de prévoyance eux-mêmes n’ont pas été touchés, mais la plateforme d’information Cybercrimepolice.ch a déjà mis en garde contre des appels frauduleux utilisant les données dérobées.

S’y ajoute un constat difficile à expliquer pour un prestataire de cette catégorie : **PK Softech AG n’a toujours aucun enregistrement DMARC.** Son domaine dispose d’un enregistrement SPF strict, mais d’aucune politique DMARC (vérifié le 18.10.2026). L’adresse d’expéditeur du prestataire attaqué peut donc être usurpée sans qu’une plateforme de messagerie destinataire la rejette de manière fiable. Après une exfiltration de données, c’est la pire situation imaginable. Quiconque connaît le nom, l’adresse et les coordonnées bancaires d’une personne assurée et peut en plus falsifier un e-mail au nom du prestataire ou de la caisse peut rédiger un message presque impossible à distinguer d’un vrai, par exemple une demande de mise à jour de l’IBAN pour le prochain versement de rente. C’est précisément contre cette falsification que DMARC protège.

PK Softech n’est pas un cas isolé, mais l’exemple le plus manifeste d’un schéma récurrent. J’ai vérifié dans quelle mesure les institutions de prévoyance suisses et leurs prestataires habituels sont préparés en matière d’authentification des e-mails.

## Méthode de vérification

L’évaluation porte sur 66 institutions de prévoyance du deuxième pilier, des grandes caisses de droit public aux fondations collectives des assureurs, en passant par de plus petites institutions communes. Seuls les enregistrements DNS publics ont été vérifiés : SPF, DKIM, DMARC, les enregistrements MX, ainsi que DNSSEC et DANE (enregistrements TLSA des serveurs de messagerie, vérifiés le 19.10.2026). Ce sont les mêmes informations que tout système de messagerie destinataire consulte pour chaque message entrant.

La vérification a été purement passive. Seul le DNS public a été interrogé, sans connexion aux systèmes des institutions, sans tentative de connexion et sans collecte de données personnelles. L’évaluation ne reflète que ce qui est de toute façon accessible à tous depuis des sources publiques. Les fondations collectives de groupe sont regroupées sous le domaine principal de leur organisme porteur ; certaines fondations affiliées peuvent différer.

## Le résultat

| Caractéristique | Part des 66 institutions |
|---|---|
| SPF présent | 88 % |
| SPF strict (`-all`) | 59 % |
| DMARC présent | 59 % |
| DMARC `p=reject` (protection complète) | 20 % |
| DMARC `p=quarantine` | 26 % |
| DMARC `p=none` (observation uniquement) | 14 % |
| aucun DMARC | 41 % |
| DKIM détectable | 55 % |
| DNSSEC | 38 % |
| DANE | 9 % |

Trois éléments ressortent.

**Seuls 20 % sont efficacement protégés par DMARC `p=reject`.** Seule cette politique demande à une plateforme de messagerie destinataire de rejeter effectivement les messages falsifiés. Dans tous les autres cas, un faux message aboutit, en cas de doute, dans la boîte de réception de la personne assurée.

**41 % n’ont aucun DMARC.** Leur domaine d’expéditeur peut facilement être utilisé pour des e-mails frauduleux, et l’institution ne l’apprend même pas, car sans DMARC, la fonction de rapport fait également défaut. Il est frappant que cela inclue aussi des caisses qui exploitent leur messagerie avec Microsoft 365, où DMARC peut être mis en place avec peu d’efforts.

**DMARC `p=none` n’est pas une protection.** Les 14 % de ce groupe ont défini une politique DMARC, mais l’ont laissée au niveau d’observation. Cela fournit des rapports, mais ne rejette pas un seul message falsifié. En pratique, il s’agit d’une étape intermédiaire qui reste souvent en place pendant des mois ou des années.

Le chiffrement du transport présente une situation similaire : seules 6 des 66 institutions (9 %) sécurisent leurs serveurs de messagerie avec DANE. Un serveur de messagerie expéditeur vérifie ainsi, à l’aide d’un enregistrement signé par DNSSEC, qu’il est connecté de manière chiffrée au bon destinataire. DNSSEC est une condition préalable et 38 % des institutions l’ont déjà activé. Pour elles, DANE constitue l’étape suivante logique.

## Tous les jeux de données à consulter

L’infographie suivante présente l’état vérifié des 66 institutions de prévoyance et des 30 prestataires. Les filtres permettent de restreindre l’affichage selon le statut DMARC, la catégorie et le fournisseur de messagerie, et chaque entrée peut être déployée pour consulter les résultats DNS complets.

<div data-dmarc-infographic></div>

## Comparaison : les plus grandes entreprises suisses

La position des institutions de prévoyance par rapport au reste de l’économie est présentée par l’[indice suisse de sécurité des e-mails](/open-data/mail-security-index). Il vérifie quotidiennement, selon la même méthode passive, plus de 1000 des plus grandes entreprises suisses ainsi que la Confédération, les cantons et les communes, avec une carte interactive par canton, des comparaisons par secteur et fournisseur de messagerie, et une page détaillée pour chaque organisation. Toutes les institutions de prévoyance et tous les prestataires examinés ici y figurent avec leur propre profil, regroupés dans le secteur [institutions de prévoyance](/open-data/mail-security-index/branche/vorsorgeeinrichtungen) et mis à jour quotidiennement. Dans l’infographie ci-dessus, le lien de l’entrée déployée mène directement au profil correspondant.

L’écart est net : parmi les 518 entreprises de l’indice, 38 % utilisent, le 18.10.2026, DMARC avec `p=reject`, contre 20 % des institutions de prévoyance. 5 % des entreprises n’ont aucun DMARC, contre 41 % des institutions de prévoyance.

## Les fournisseurs constituent le problème majeur

L’incident de fin septembre n’a pas touché une caisse, mais un prestataire commun. J’ai donc vérifié, selon la même méthode passive, 30 fournisseurs typiques : fabricants de logiciels de gestion, entreprises de gestion externalisée, d’actuariat et de conseil, banques dépositaires et prestataires informatiques.

En moyenne, les prestataires s’en sortent un peu mieux que les caisses : 40 % utilisent DMARC avec `p=reject`, contre 20 % des institutions. La répartition entre les catégories est toutefois déterminante.

| Catégorie | avec DMARC `p=reject` |
|---|---|
| Actuariat et conseil | bon |
| Banques dépositaires | bon |
| Prestataires informatiques | mitigé |
| Gestion externalisée | faible |
| Logiciels de gestion | le plus faible |

Les fabricants de logiciels de gestion constituent le groupe le plus faible. Aucun des fournisseurs de logiciels examinés n’utilise DMARC avec une protection complète, et le prestataire attaqué PK Softech n’a aucun enregistrement DMARC. Or, il s’agit précisément de la catégorie qui traite les données de prévoyance et est profondément connectée aux caisses par des interfaces, ainsi que de la catégorie concernée par l’incident actuel.

C’est là que réside l’enjeu réel en matière de diligence : une caisse avec `p=reject` est bien protégée tant que l’attaquant veut usurper son domaine. S’il peut en revanche usurper le domaine d’un prestataire insuffisamment protégé, par lequel la communication avec les assurés a de toute façon lieu, l’e-mail est tout aussi crédible. La sécurisation de son propre domaine n’est donc que la première étape. La seconde consiste à exiger des principaux prestataires le même niveau de protection, idéalement par contrat.

## Ce que cela signifie pour le conseil de fondation

La responsabilité de la sécurité de l’information incombe à l’organe suprême de l’institution de prévoyance. Une exfiltration de données entraîne en outre des obligations concrètes au titre de la loi révisée sur la protection des données : une violation de la sécurité des données doit être signalée au PFPDT si elle est susceptible d’entraîner un risque élevé pour les personnes concernées, et celles-ci doivent être informées lorsque cela est nécessaire à leur protection.

Dans cette situation, l’authentification des e-mails est la mesure individuelle la plus efficace avec l’effort le plus faible. SPF, DKIM et DMARC ne nécessitent ni licence ni nouveau logiciel. Ils reposent sur des enregistrements DNS et une configuration propre de la plateforme de messagerie existante. La protection agit immédiatement pour chaque personne assurée, sans que celle-ci ait elle-même quoi que ce soit à faire.

## Ce qu’il faut faire

Pour les institutions sans protection efficace, une voie claire mène à l’objectif :

1. **Vérifier SPF et le définir strictement.** L’enregistrement SPF répertorie les systèmes d’envoi autorisés et se termine par `-all`, afin que tous les autres expéditeurs soient considérés comme non autorisés.
2. **Activer DKIM.** Chaque message sortant est signé cryptographiquement. Dans Microsoft 365 et les passerelles de messagerie courantes, il s’agit d’un réglage, pas d’un projet.
3. **Introduire DMARC et renforcer progressivement la politique.** Commencer par `p=none` et un reporting actif afin d’identifier le trafic de messagerie légitime. Passer ensuite à `p=quarantine`, puis enfin à `p=reject`. Selon l’environnement, l’ensemble du processus prend quelques semaines, pas des mois.
4. **Évaluer les rapports.** Les rapports DMARC indiquent qui envoie des e-mails au nom du domaine et révèlent tant les systèmes légitimes oubliés que les abus.
5. **Au minimum examiner DANE et le chiffrement des e-mails.** DANE sécurise le chiffrement du transport entre serveurs de messagerie via DNSSEC. Il est facile de déterminer si sa propre plateforme de messagerie et le fournisseur DNS le prennent en charge. Pour les messages contenant des données salariales et de prévoyance, il convient en outre de définir comment ils sont envoyés avec un chiffrement de bout en bout, par exemple via une passerelle de messagerie avec S/MIME ou une livraison sécurisée par portail web.
6. **Responsabiliser les prestataires.** Les fournisseurs de logiciels, les gestionnaires externalisés et les partenaires informatiques devraient démontrer le même niveau de protection en matière de SPF, DKIM et DMARC. Cela doit figurer dans les contrats et dans l’évaluation régulière des fournisseurs.

## Conclusion

Les attaques récentes visaient un prestataire, et non les caisses elles-mêmes. Les dommages consécutifs surviennent toutefois là où un domaine d’expéditeur n’est pas protégé, que ce soit celui de la caisse ou de ses fournisseurs. Quatre institutions de prévoyance examinées sur cinq ne sont pas entièrement protégées contre les expéditeurs falsifiés, deux sur cinq n’ont aucun DMARC, et les fabricants de logiciels de gestion obtiennent précisément les plus mauvais résultats. La lacune est connue, peu coûteuse à combler et produit un effet immédiat. Après une exfiltration de données concernant le nom, l’adresse et les coordonnées bancaires, il ne s’agit pas d’un détail technique, mais d’une composante du devoir de diligence de l’organe suprême, tout au long de la chaîne d’approvisionnement.

## Historique des modifications

- **25 octobre 2026 :** lien vers les Sources mis à jour avec la nouvelle adresse du PFPDT.

## Sources

1.  [SRF : Caisse de pension de la Confédération, comment des données sensibles ont-elles pu se retrouver chez une entreprise de logiciels externe ?](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen) : contexte de l’incident, types de données concernées et rôle du fournisseur de logiciels.

2.  [cash.ch (AWP) : Attaque contre un fournisseur de Publica, des données salariales pourraient être concernées](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247) : Publica désigne PK Softech AG comme l’entreprise attaquée auprès d’AWP.

3.  [Netzwoche : La caisse de pension de la Confédération Publica signale une exfiltration de données](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss) : information sur l’attaque contre le fournisseur de logiciels et l’exfiltration de données.

4.  [Mimikama : La police suisse met en garde contre de possibles appels d’investissement après une exfiltration de données](https://www.mimikama.org/?p=436628) : mise en contexte de l’avertissement concernant des appels frauduleux utilisant les données exfiltrées.

5.  [PFPDT : DataBreach](https://www.edoeb.admin.ch/de/databreach) : obligation de notification selon l’art. 24 LPD, portail de notification et guide relatif à la procédure à suivre en cas de violation de la sécurité des données.
