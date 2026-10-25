---
title: "Le prestataire de caisses de pension attaqué n’a même pas d’enregistrement DMARC"
navTitle: "DMARC caisses de pension"
description: "Le prestataire de caisses de pension PK Softech, attaqué fin septembre 2026, n’a toujours pas d’enregistrement DMARC. Un examen passif de 66 institutions de prévoyance et de 30 prestataires montre qu’il ne s’agit pas d’un cas isolé : les fabricants de logiciels de gestion sont particulièrement insuffisamment protégés."
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
translationSourceHash: 57506056bfd8dcc96818df43dc71d22a158978bcd7432c91c56eb92698e52fe1
translationModel: gpt-5.6-terra
translatedAt: 2026-10-20T08:41:48.971Z
translationReview: automatic
url: https://rafaelpfister.ch/fr/blog/le-prestataire-de-caisses-de-pension-attaque-n-a-aucun-enregistrement-dmarc
---

Fin septembre 2026, PK Softech AG a été attaquée, un éditeur de logiciels dont plusieurs caisses de pension suisses utilisent la solution de prévoyance. Des données ont été exfiltrées par cette voie. Selon les informations publiées, les entités touchées comprennent notamment la caisse de pension de la Confédération Publica, la caisse de pension de Migros et la caisse de pension de la ville de Zurich, ainsi que d’autres institutions. Selon la caisse concernée, il s’agit du nom, du numéro AVS, de l’état civil, de l’adresse et de l’IBAN, ainsi que de données salariales et de prévoyance.

Cette fuite est grave en soi. Elle concerne des données d’identification et financières d’une grande partie de la population active et, par l’intermédiaire de Publica, de larges pans de l’administration fédérale. Des données d’un tel volume et d’une telle qualité ne constituent pas seulement un problème pour chaque personne concernée. Elles permettent l’usurpation d’identité, la tromperie ciblée et l’extorsion à grande échelle, et représentent ainsi une menace pour l’ordre public et la sécurité nationale. Les avoirs de prévoyance eux-mêmes n’ont pas été affectés, mais la plateforme d’information Cybercrimepolice.ch avait déjà mis en garde contre des appels frauduleux utilisant les données dérobées.

À cela s’ajoute une constatation difficile à expliquer pour un prestataire de cette catégorie : **PK Softech AG n’a toujours pas d’enregistrement DMARC.** Son domaine dispose d’un enregistrement SPF strict, mais d’aucune politique DMARC (vérifié le 18.10.2026). Il est donc possible d’usurper l’adresse d’expéditeur du prestataire attaqué sans qu’une plateforme de messagerie destinataire ne le refuse de manière fiable. Après une fuite de données, c’est la pire situation imaginable. Quiconque connaît le nom, l’adresse et les coordonnées bancaires d’une personne assurée et peut en plus falsifier un e-mail au nom du prestataire ou de la caisse peut rédiger un message presque impossible à distinguer d’un message authentique, par exemple une demande de mise à jour de l’IBAN pour le prochain versement de rente. C’est précisément contre cette falsification que DMARC protège.

PK Softech n’est pas un cas isolé, mais l’exemple le plus frappant d’une tendance. J’ai examiné dans quelle mesure les institutions de prévoyance suisses et leurs prestataires habituels sont correctement équipés en matière d’authentification des e-mails.

## Méthode de vérification

L’analyse porte sur 66 institutions de prévoyance du deuxième pilier, des grandes caisses de droit public aux fondations collectives des assureurs, en passant par de plus petites institutions communes. Seuls les enregistrements DNS publics ont été examinés : SPF, DKIM, DMARC, les enregistrements MX ainsi que DNSSEC et DANE (enregistrements TLSA des serveurs de messagerie, vérifiés le 19.10.2026). Il s’agit des mêmes informations que tout système de messagerie destinataire interroge pour chaque message entrant.

La vérification était purement passive. Seul le DNS public a été interrogé, sans connexion aux systèmes des institutions, sans tentative d’authentification et sans collecte de données personnelles. L’analyse ne reflète que ce qui est de toute façon publiquement accessible à tous. Les fondations collectives de groupes sont regroupées sous le domaine principal de leur organisme porteur ; certaines fondations affiliées peuvent différer.

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

**Seuls 20 % sont efficacement protégés avec DMARC `p=reject`.** Seule cette politique demande à une plateforme de messagerie destinataire de rejeter effectivement les messages falsifiés. Dans tous les autres cas, un message falsifié risque d’atterrir dans la boîte de réception de la personne assurée.

**41 % n’ont pas de DMARC.** Leur domaine expéditeur peut sans difficulté être utilisé pour des e-mails frauduleux, et l’institution ne l’apprend même pas, car sans DMARC, la fonction de reporting fait également défaut. Il est frappant que cela inclue aussi des caisses qui exploitent leur messagerie avec Microsoft 365, où DMARC peut être configuré avec peu d’efforts.

**DMARC `p=none` n’est pas une protection.** Les 14 % de ce groupe ont défini une politique DMARC, mais l’ont laissée au niveau d’observation. Cela fournit des rapports, mais ne rejette pas un seul message falsifié. En pratique, il s’agit d’une étape intermédiaire qui reste souvent en place pendant des mois ou des années.

Le chiffrement du transport présente un tableau similaire : seules 6 des 66 institutions (9 %) protègent leurs serveurs de messagerie avec DANE. Un serveur de messagerie expéditeur vérifie ainsi, à l’aide d’un enregistrement signé par DNSSEC, qu’il est connecté de manière chiffrée au bon destinataire. Cela nécessite DNSSEC, que 38 % des institutions ont déjà activé. Pour elles, DANE constitue l’étape suivante logique.

## Tous les jeux de données à explorer

L’infographie suivante présente l’état vérifié des 66 institutions de prévoyance et des 30 prestataires. Les filtres permettent de sélectionner selon le statut DMARC, la catégorie et le fournisseur de messagerie, et chaque entrée peut être déployée pour consulter les résultats DNS complets.

<div data-dmarc-infographic></div>

## Comparaison : les plus grandes entreprises suisses

Le [Mail-Sicherheitsindex Schweiz](/open-data/mail-security-index) montre comment les institutions de prévoyance se situent par rapport au reste de l’économie. Il examine quotidiennement, selon la même méthode passive, plus de 1000 des plus grandes entreprises suisses ainsi que la Confédération, les cantons et les communes, avec une carte interactive par canton, des comparaisons par secteur et fournisseur de messagerie, et une page détaillée pour chaque organisation. Toutes les institutions de prévoyance et tous les prestataires examinés ici y figurent avec leur propre profil, regroupés dans le secteur [Institutions de prévoyance](/open-data/mail-security-index/branche/vorsorgeeinrichtungen) et mis à jour quotidiennement. Dans l’infographie ci-dessus, le lien de l’entrée déployée mène directement au profil correspondant.

L’écart est net : parmi les 518 entreprises de l’indice, 38 % utilisent le 18.10.2026 DMARC avec `p=reject`, contre 20 % des institutions de prévoyance. 5 % des entreprises n’ont pas du tout de DMARC, contre 41 % des institutions de prévoyance.

## Les fournisseurs constituent le problème majeur

L’incident de fin septembre n’a pas touché une caisse, mais un prestataire commun. J’ai donc examiné, selon la même méthode passive, 30 fournisseurs typiques : éditeurs de logiciels de gestion, entreprises assurant la gestion externalisée, actuariat et conseil, banques dépositaires et prestataires informatiques.

En moyenne, les prestataires sont légèrement mieux positionnés que les caisses : 40 % utilisent DMARC avec `p=reject`, contre 20 % des institutions. La répartition entre les catégories est toutefois déterminante.

| Catégorie | avec DMARC `p=reject` |
|---|---|
| Actuariat et conseil | bon |
| Banques dépositaires | bon |
| Prestataires informatiques | mitigé |
| Gestion externalisée | faible |
| Logiciels de gestion | le plus faible |

Les éditeurs de logiciels de gestion constituent le groupe le plus faible. Aucun des fournisseurs de logiciels examinés n’utilise DMARC avec une protection complète, et le prestataire attaqué PK Softech n’a même pas d’enregistrement DMARC. C’est précisément la catégorie qui traite des données de prévoyance et est étroitement connectée aux caisses par des interfaces, et c’est la catégorie concernée par l’incident actuel.

C’est là que réside le véritable enjeu de diligence : une caisse disposant de `p=reject` est bien protégée tant que l’attaquant veut falsifier son domaine. S’il peut en revanche falsifier le domaine d’un prestataire mal protégé, par lequel la communication avec les personnes assurées passe de toute façon, l’e-mail est tout aussi crédible. La protection de son propre domaine n’est donc que la première étape. La seconde consiste à exiger des principaux prestataires le même niveau de protection, idéalement par contrat.

## Ce que cela signifie pour le conseil de fondation

La responsabilité de la sécurité de l’information incombe à l’organe suprême de l’institution de prévoyance. Une fuite de données entraîne en outre des obligations concrètes en vertu de la loi révisée sur la protection des données : une violation de la sécurité des données doit être signalée au PFPDT lorsqu’elle est susceptible d’entraîner un risque élevé pour les personnes concernées, et les personnes concernées doivent être informées lorsque cela est nécessaire à leur protection.

Dans cette situation, l’authentification des e-mails est la mesure individuelle la plus efficace avec le moindre effort. SPF, DKIM et DMARC ne nécessitent ni licence ni nouveau logiciel. Ils reposent sur des enregistrements DNS et une configuration correcte de la plateforme de messagerie existante. La protection agit immédiatement pour chaque personne assurée, sans qu’elle ait elle-même quoi que ce soit à faire.

## Ce qu’il faut faire

Pour les institutions sans protection efficace, un chemin clair mène à l’objectif :

1. **Vérifier SPF et le définir de manière stricte.** L’enregistrement SPF liste les systèmes d’envoi autorisés et se termine par `-all`, afin que tous les autres expéditeurs soient considérés comme non autorisés.
2. **Activer DKIM.** Chaque message sortant est signé cryptographiquement. Dans Microsoft 365 et les passerelles de messagerie courantes, il s’agit d’un paramètre, pas d’un projet.
3. **Introduire DMARC et renforcer progressivement la politique.** Commencer avec `p=none` et un reporting actif afin d’identifier le trafic de messagerie légitime. Ensuite `p=quarantine`, puis enfin `p=reject`. Selon l’environnement, l’ensemble du processus prend quelques semaines, et non des mois.
4. **Analyser les rapports.** Les rapports DMARC montrent qui envoie des e-mails au nom du domaine et révèlent tant les systèmes légitimes oubliés que les abus.
5. **Au minimum examiner DANE et le chiffrement des e-mails.** DANE protège le chiffrement du transport entre serveurs de messagerie via DNSSEC. Il est possible de déterminer avec peu d’efforts si sa propre plateforme de messagerie et son fournisseur DNS le prennent en charge. Pour les messages contenant des données salariales et de prévoyance, il faut en outre définir comment ils sont envoyés de bout en bout de manière chiffrée, par exemple par une passerelle de messagerie avec S/MIME ou une livraison sécurisée via un portail web.
6. **Responsabiliser les prestataires.** Les fournisseurs de logiciels, les gestionnaires externalisés et les partenaires informatiques doivent démontrer le même niveau de protection en matière de SPF, DKIM et DMARC. Cela doit figurer dans les contrats et dans l’évaluation régulière des fournisseurs.

## Conclusion

Les attaques récentes visaient un prestataire, et non les caisses elles-mêmes. Les dommages consécutifs surviennent toutefois là où un domaine expéditeur n’est pas protégé, que ce soit auprès de la caisse ou de ses fournisseurs. Quatre institutions de prévoyance examinées sur cinq ne sont pas entièrement protégées contre les expéditeurs falsifiés, deux sur cinq n’ont pas du tout de DMARC, et les éditeurs de logiciels de gestion obtiennent précisément les plus mauvais résultats. La lacune est connue, peu coûteuse à combler et son effet est immédiat. Après une fuite de données touchant le nom, l’adresse et les coordonnées bancaires, il ne s’agit pas d’un détail technique, mais d’une partie du devoir de diligence de l’organe suprême, sur l’ensemble de la chaîne d’approvisionnement.

## Sources

1.  [SRF : Caisse de pension de la Confédération, comment des données sensibles ont-elles pu se retrouver chez une entreprise externe de logiciels](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen) : contexte de l’incident, types de données concernés et rôle du fournisseur de logiciels.

2.  [cash.ch (AWP) : attaque contre un fournisseur de Publica, des données salariales pourraient être concernées](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247) : Publica identifie auprès d’AWP PK Softech AG comme l’entreprise attaquée.

3.  [Netzwoche : la caisse de pension de la Confédération Publica signale une fuite de données](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss) : information sur l’attaque contre le fournisseur de logiciels et la fuite de données.

4.  [Mimikama : la police suisse met en garde contre de possibles appels liés à des investissements après une fuite de données](https://www.mimikama.org/?p=436628) : mise en contexte de l’avertissement concernant les appels frauduleux utilisant les données exfiltrées.

5.  [PFPDT : annonce des violations de la sécurité des données](https://www.edoeb.admin.ch/de/meldung-von-verletzungen-der-datensicherheit) : obligation d’annonce selon la LPD révisée et procédure en cas de violation de la sécurité des données.
