---
title: "Le prestataire de caisses de pension attaqué n’a aucun enregistrement DMARC"
navTitle: "DMARC caisses de pension"
description: "Le prestataire de caisses de pension PK Softech, attaqué fin septembre 2026, n’a toujours pas d’enregistrement DMARC. Un contrôle passif de 66 institutions de prévoyance et de 30 prestataires montre qu’il ne s’agit pas d’un cas isolé : les éditeurs de logiciels d’administration sont particulièrement insuffisamment protégés."
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
aiPrompt: |
  Du bist mein Berater für E-Mail-Sicherheit. Prüfe für meine Domain, ob SPF, DKIM, DMARC und MTA-STS korrekt gesetzt sind, erkläre mir den aktuellen Stand in einfachen Worten und zeige mir Schritt für Schritt, wie ich DMARC gefahrlos von p=none über p=quarantine auf p=reject anhebe.
translationOf: dmarc-pensionskassen-schweiz
url: https://rafaelpfister.ch/fr/blog/le-prestataire-de-caisses-de-pension-attaque-n-a-aucun-enregistrement-dmarc
translationSourceHash: 64ec035a358c39b09487102da66cf7aca02df2c567e29e343895906a4490d205
translationModel: gpt-5.6-terra
translatedAt: 2026-10-18T14:53:06.918Z
translationReview: automatic
---

Fin septembre 2026, PK Softech AG a été attaquée. Il s’agit d’un éditeur de logiciels dont plusieurs caisses de pension suisses utilisent la solution de prévoyance. Des données ont été exfiltrées par cette voie. Selon les informations publiées, les entités touchées comprennent notamment la caisse fédérale de pensions Publica, la caisse de pension de Migros et la caisse de pension de la ville de Zurich, ainsi que d’autres institutions. Selon la caisse concernée, il s’agit du nom, du numéro AVS, de l’état civil, de l’adresse et de l’IBAN, ainsi que de données salariales et de prévoyance.

Cette exfiltration est déjà grave en elle-même. Elle concerne des données d’identification et financières d’une grande partie de la population active et, via Publica, d’importantes parties de l’administration fédérale. Des données de cette quantité et de cette qualité ne constituent pas seulement un problème pour chaque personne concernée. Elles permettent l’usurpation d’identité, la fraude ciblée et l’extorsion à grande échelle, et représentent ainsi un danger pour l’ordre public et la sécurité nationale. Les avoirs de prévoyance eux-mêmes n’ont pas été touchés, mais la police de la cybercriminalité avait déjà mis en garde contre des appels frauduleux utilisant les données dérobées.

S’y ajoute un constat difficile à expliquer pour un prestataire de cette catégorie : **PK Softech AG n’a toujours pas d’enregistrement DMARC.** Son domaine possède un enregistrement SPF strict, mais aucune politique DMARC (vérifié le 18.10.2026). Il est donc possible d’usurper l’adresse d’expéditeur du prestataire attaqué sans qu’une plateforme de messagerie destinataire ne la rejette de manière fiable. Après une exfiltration de données, c’est la situation la plus défavorable imaginable. Quiconque connaît le nom, l’adresse et les coordonnées bancaires d’une personne assurée et peut en plus usurper un e-mail au nom du prestataire ou de la caisse peut rédiger un message presque impossible à distinguer d’un vrai, par exemple une demande de mise à jour de l’IBAN pour le prochain versement de rente. C’est précisément contre cette usurpation que DMARC protège.

PK Softech n’est pas un cas isolé, mais l’exemple le plus frappant d’une tendance. J’ai examiné le niveau d’authentification des e-mails des institutions de prévoyance suisses et de leurs prestataires habituels.

## Méthode de vérification

L’évaluation porte sur 66 institutions de prévoyance du deuxième pilier, des grandes caisses de droit public aux fondations collectives des assureurs, en passant par de plus petites institutions communes. Seuls les enregistrements DNS publics ont été vérifiés : SPF, DKIM, DMARC, MTA-STS, TLS-RPT et les enregistrements MX. Ce sont les mêmes informations que chaque système de messagerie destinataire consulte pour chaque message entrant.

Le contrôle était purement passif. Seul le DNS public a été interrogé ; aucune connexion aux systèmes des institutions n’a été établie, aucune tentative de connexion n’a été faite et aucune donnée personnelle n’a été collectée. L’évaluation reflète uniquement ce qui est de toute façon publiquement visible pour tous. Les fondations collectives de groupes sont regroupées sous le domaine principal de leur entité porteuse ; certaines fondations affiliées peuvent différer.

## Résultat

| Caractéristique | Part des 66 institutions |
|---|---|
| SPF présent | 88 % |
| SPF strict (`-all`) | 59 % |
| DMARC présent | 59 % |
| DMARC `p=reject` (protection complète) | 20 % |
| DMARC `p=quarantine` | 26 % |
| DMARC `p=none` (observation uniquement) | 14 % |
| aucun DMARC | 41 % |
| DKIM identifiable | 55 % |
| MTA-STS | 2 % |
| TLS-RPT | 5 % |

Trois points ressortent.

**Seuls 20 % sont efficacement protégés par DMARC `p=reject`.** Seule cette politique ordonne à une plateforme de messagerie destinataire de rejeter effectivement les messages falsifiés. Dans tous les autres cas, une falsification finit en cas de doute dans la boîte de réception de la personne assurée.

**41 % n’ont pas de DMARC.** Leur domaine d’expéditeur peut facilement être utilisé pour des e-mails frauduleux, et l’institution ne l’apprend même pas, car sans DMARC, la fonction de rapport fait également défaut. Il est frappant que cela concerne aussi des caisses qui exploitent leur messagerie sur Microsoft 365, où DMARC peut être configuré avec peu d’efforts.

**DMARC `p=none` n’est pas une protection.** Les 14 % de ce groupe ont défini une politique DMARC, mais l’ont laissée au niveau de l’observation. Cela fournit des rapports, mais ne rejette aucun message falsifié. En pratique, il s’agit d’une étape intermédiaire qui reste souvent en place pendant des mois ou des années.

Un constat général s’y ajoute : avec 2 %, MTA-STS est pratiquement inexistant. Sans MTA-STS, le chiffrement du transport des e-mails entrants n’est pas imposé, ce qui favorise un attaquant ayant accès au chemin de transport.

## Tous les jeux de données à consulter

L’infographie suivante présente l’état vérifié des 66 institutions de prévoyance et 30 prestataires. Les filtres permettent de restreindre l’affichage selon le statut DMARC, la catégorie et le fournisseur de messagerie, et chaque entrée peut être dépliée pour afficher les résultats DNS complets.

<div data-dmarc-infographic></div>

## Comparaison : les plus grandes entreprises suisses

La position des institutions de prévoyance par rapport au reste de l’économie est présentée par l’[indice suisse de sécurité des e-mails](/open-data/mail-security-index). Il vérifie quotidiennement, selon la même méthode passive, plus de 500 des plus grandes entreprises suisses, avec une carte interactive par canton, des comparaisons par secteur et fournisseur de messagerie, ainsi qu’une page détaillée pour chaque entreprise. Parmi elles figurent également les banques, assureurs et prestataires IT avec lesquels collaborent les institutions de prévoyance.

L’écart est net : parmi les 518 entreprises de l’indice, 38 % utilisent DMARC avec `p=reject` au 18.10.2026, contre 20 % des institutions de prévoyance. 5 % des entreprises n’ont absolument aucun DMARC, contre 41 % des institutions de prévoyance.

## Les fournisseurs constituent le sujet le plus important

L’incident de fin septembre n’a pas touché une caisse, mais un prestataire commun. J’ai donc vérifié 30 fournisseurs typiques selon la même méthode passive : éditeurs de logiciels d’administration, entreprises assurant la gestion externalisée, l’actuariat et le conseil, banques dépositaires et prestataires IT.

En moyenne, les prestataires se situent un peu mieux que les caisses : 40 % utilisent DMARC avec `p=reject`, contre 20 % des institutions. Mais la répartition entre les catégories est déterminante.

| Catégorie | avec DMARC `p=reject` |
|---|---|
| Actuariat et conseil | bon |
| Banques dépositaires | bon |
| Prestataires IT | mitigé |
| Gestion externalisée | faible |
| Logiciels d’administration | le plus faible |

Les éditeurs de logiciels d’administration constituent le groupe le plus faible. Aucun des fournisseurs de logiciels examinés n’utilise DMARC avec pleine efficacité, et le prestataire attaqué PK Softech n’a même aucun enregistrement DMARC. C’est précisément la catégorie qui traite des données de prévoyance et qui est profondément connectée aux caisses par des interfaces, et c’est la catégorie concernée par l’incident actuel.

Il en découle le véritable enjeu de diligence : une caisse avec `p=reject` est bien protégée tant que l’attaquant veut usurper son domaine. S’il peut à la place usurper le domaine d’un prestataire insuffisamment protégé, par lequel s’effectue déjà la communication avec les personnes assurées, l’e-mail est tout aussi crédible. La protection de son propre domaine n’est donc que la première étape. La deuxième consiste à imposer le même niveau aux principaux prestataires, idéalement par contrat.

## Ce que cela signifie pour le conseil de fondation

La responsabilité de la sécurité de l’information incombe à l’organe suprême de l’institution de prévoyance. Une exfiltration de données déclenche en outre des obligations concrètes au titre de la loi révisée sur la protection des données : une violation de la sécurité des données doit être signalée au PFPDT si elle est susceptible d’entraîner un risque élevé pour les personnes concernées, et celles-ci doivent être informées lorsque cela est nécessaire à leur protection.

Dans cette situation, l’authentification des e-mails est la mesure individuelle la plus efficace pour l’effort le plus faible. SPF, DKIM et DMARC ne nécessitent ni licence ni nouveau logiciel. Ils reposent sur des enregistrements DNS et une configuration correcte de la plateforme de messagerie existante. La protection agit immédiatement pour chaque personne assurée, sans qu’elle ait elle-même quoi que ce soit à faire.

## Ce qu’il faut faire

Pour les institutions sans protection efficace, un chemin clair mène à l’objectif :

1. **Vérifier SPF et le définir de manière stricte.** L’enregistrement SPF liste les systèmes d’envoi autorisés et se termine par `-all`, afin que tous les autres expéditeurs soient considérés comme non autorisés.
2. **Activer DKIM.** Chaque message sortant est signé cryptographiquement. Dans Microsoft 365 et les passerelles de messagerie courantes, c’est un paramètre, pas un projet.
3. **Introduire DMARC et renforcer progressivement la politique.** Commencer par `p=none` et un reporting actif afin de recenser le trafic légitime. Puis `p=quarantine`, et enfin `p=reject`. Selon l’environnement, l’ensemble du processus prend quelques semaines, pas des mois.
4. **Évaluer les rapports.** Les rapports DMARC montrent qui envoie des e-mails au nom du domaine et révèlent aussi bien des systèmes légitimes oubliés que des abus.
5. **Ajouter MTA-STS et TLS-RPT.** Tous deux imposent et surveillent le chiffrement du transport des e-mails entrants.
6. **Obliger les prestataires à respecter ces exigences.** Les fournisseurs de logiciels, les prestataires de gestion externalisée et les partenaires IT devraient démontrer le même niveau de SPF, DKIM et DMARC. Cela doit figurer dans les contrats et dans l’évaluation régulière des fournisseurs.

## Conclusion

Les attaques récentes visaient un prestataire, non les caisses elles-mêmes. Mais les dommages consécutifs surviennent là où un domaine d’expéditeur n’est pas protégé, aussi bien chez la caisse que chez ses fournisseurs. Quatre institutions de prévoyance examinées sur cinq ne sont pas entièrement protégées contre les expéditeurs falsifiés, deux sur cinq n’ont aucun DMARC, et les éditeurs de logiciels d’administration obtiennent précisément les pires résultats. Cette lacune est connue, peu coûteuse à combler et son effet est immédiat. Après une exfiltration de données concernant le nom, l’adresse et les coordonnées bancaires, elle n’est pas un détail technique, mais fait partie du devoir de diligence de l’organe suprême, tout au long de la chaîne d’approvisionnement.

## Sources

1.  [SRF : Caisse fédérale de pensions, comment des données sensibles ont-elles pu se retrouver chez une société de logiciels externe ?](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen): Contexte de l’incident, types de données concernés et rôle du fournisseur de logiciels.

2.  [cash.ch (AWP) : attaque contre un fournisseur de Publica, des données salariales pourraient être concernées](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247): Publica désigne PK Softech AG auprès d’AWP comme l’entreprise attaquée.

3.  [Netzwoche : la caisse fédérale de pensions Publica signale une exfiltration de données](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss): Informations sur l’attaque contre le fournisseur de logiciels et l’exfiltration de données.

4.  [Mimikama : la police suisse met en garde contre d’éventuels appels d’investissement après une exfiltration de données](https://www.mimikama.org/?p=436628): Mise en contexte de l’avertissement concernant des appels frauduleux utilisant les données exfiltrées.

5.  [PFPDT : annonce des violations de la sécurité des données](https://www.edoeb.admin.ch/de/meldung-von-verletzungen-der-datensicherheit): Obligation d’annonce selon la LPD révisée et procédure en cas de violation de la sécurité des données.
