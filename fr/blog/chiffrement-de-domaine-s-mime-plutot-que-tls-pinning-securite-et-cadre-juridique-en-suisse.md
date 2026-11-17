---
title: "Chiffrement de domaine S/MIME plutôt que TLS-Pinning : sécurité et cadre juridique en Suisse"
navTitle: "Domaine plutôt que pinning"
description: "Pourquoi le chiffrement de domaine S/MIME protège mieux le trafic de messagerie entre deux organisations que le TLS-Pinning, ce qu’un attaquant peut faire d’un message S/MIME intercepté, comment la LPD, l’OPDo et les secrets professionnels influencent ce choix, et pourquoi MTA-STS et DANE assurent la protection du transport."
date: "2026-11-16"
kategorie: "Chiffrement des e-mails"
timeToRead: "10 min de lecture"
themen:
  - e-mail-verschluesselung
produkte: []
protokolle:
  - "verschluesselung"
  - "tls"
  - "dns"
hauptthema: "e-mail-verschluesselung"
startseiteAb: "2099-12-31"
slug: "chiffrement-de-domaine-s-mime-plutot-que-tls-pinning-securite-et-cadre-juridique-en-suisse"
translationId: "article-92db7a4587660dec"
translationOf: domain-verschluesselung-statt-tls-pinning
url: https://rafaelpfister.ch/fr/blog/chiffrement-de-domaine-s-mime-plutot-que-tls-pinning-securite-et-cadre-juridique-en-suisse
translationSourceHash: 8eeb8d5552f8680381710c1a052edd3e62b69b34e1af9fd032c4bd481e8d65a1
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T15:57:48.705Z
translationReview: automatic
---

# Chiffrement de domaine S/MIME plutôt que TLS-Pinning : sécurité et cadre juridique en Suisse

En bref : le TLS-Pinning protège la connexion entre deux serveurs de messagerie, tandis que le chiffrement de domaine S/MIME protège le contenu du message tout au long de son parcours entre deux organisations. S’il faut choisir entre les deux, le chiffrement de domaine constitue le meilleur choix d’un point de vue technique et juridique. MTA-STS et DANE prennent en charge la protection du transport auparavant assurée par le pinning.

Dans le cadre d’un projet actuel, une liaison de messagerie existante entre deux organisations, jusqu’ici sécurisée par TLS imposé et pinning de certificat, doit être remplacée par un chiffrement de domaine S/MIME via un service de messagerie sécurisée. Pour une même liaison, le service ne permet pas d’utiliser simultanément les deux mécanismes : soit le pinning s’applique, soit le chiffrement de domaine. La question de savoir si cela entraîne une perte de sécurité est légitime. La réponse est sans équivoque dès lors que l’on examine ce que protègent respectivement les deux procédés.

**Focus Suisse :** l’analyse juridique suit le droit suisse, notamment la LPD, l’OPDo et le CP. Lorsque les règles de l’UE divergent, une remarque signalée est fournie. Cet article ne remplace pas un conseil juridique dans un cas particulier.

## Ce que protège le TLS-Pinning

Sans mesures supplémentaires, TLS entre serveurs de messagerie est opportuniste. Il en résulte trois possibilités d’attaque :

- **Dégradation :** un attaquant sur le chemin de transmission supprime l’offre STARTTLS de la session SMTP. Le serveur expéditeur transmet alors en clair.
- **Certificat quelconque :** les serveurs de messagerie expéditeurs ne vérifient pas par défaut le certificat du correspondant. Un serveur doté d’un certificat auto-signé est accepté.
- **MX falsifié :** quiconque manipule la réponse DNS redirige le courrier vers son propre serveur, qui propose un TLS correct.

Le pinning comble ces trois lacunes pour un partenaire donné. La passerelle expéditrice impose TLS et n’accepte qu’un certificat ou une clé prédéterminés. La protection prend toutefois fin avec la connexion. Sur le serveur de réception, dans les files d’attente, les filtres antispam et antivirus ainsi que les systèmes de journalisation et d’archivage, le message est disponible en clair. S’il traverse plusieurs serveurs, le pinning ne s’applique qu’au segment pour lequel il est configuré.

L’exploitation s’y ajoute. Le pin est une règle manuelle chez l’expéditeur. Si le destinataire renouvelle son certificat ou modifie son routage, par exemple avec un nouveau service de filtrage ou un MX de secours, la distribution s’arrête jusqu’à la mise à jour de la règle. En cas de perturbation, cette règle est donc souvent assouplie temporairement, sans être toujours réactivée par la suite.

## Ce que protège le chiffrement de domaine S/MIME

Avec le chiffrement de domaine, la passerelle de l’expéditeur chiffre le message à l’aide de la clé publique figurant dans le certificat de domaine du domaine destinataire. Seule la passerelle du destinataire le déchiffre. Tous les systèmes intermédiaires ne voient que le contenu chiffré, quel que soit le nombre de serveurs traversés par le message et l’endroit où il est stocké temporairement. En outre, la passerelle de l’expéditeur signe le message avec son propre certificat de domaine.

| Protection | TLS-Pinning | Chiffrement de domaine S/MIME |
|---|---|---|
| Texte du message et pièces jointes | uniquement pendant la transmission sur le segment épinglé | tout au long du parcours, y compris sur les serveurs intermédiaires et dans les files d’attente |
| Objet, expéditeur, destinataire, date | chiffrés pendant la transmission | lisibles, sauf si la protection des en-têtes est activée |
| Enveloppe, taille, moment | chiffrés pendant la transmission | lisibles |
| Authenticité du serveur de réception | oui, un mauvais serveur est refusé | non, le message est remis au serveur fourni par le routage MX |
| Perfect Forward Secrecy | oui, avec échange de clés ECDHE | non |
| Intégrité et origine après la distribution | non | oui, par signature |
| Dépendance au routage et à l’infrastructure | élevée, le pin s’applique à un serveur ou à une clé | faible, la protection dépend du certificat de domaine |

Le pinning possède ainsi trois propriétés que le chiffrement de domaine n’offre pas : il protège les métadonnées, empêche qu’un mauvais serveur accepte le courrier et assure la Perfect Forward Secrecy. Pour le contenu, le chiffrement de domaine est clairement supérieur.

## Ce qu’un attaquant peut faire d’un message S/MIME intercepté

Supposons qu’un attaquant introduise son propre serveur de messagerie sur le chemin de transmission, par des enregistrements MX falsifiés, un détournement BGP ou un relais compromis.

Il ne peut pas :

- **Lire le contenu :** le texte et les pièces jointes sont chiffrés avec la clé publique du domaine destinataire. Sans la clé privée de la passerelle destinataire, ils restent illisibles.
- **Modifier le contenu sans être détecté :** si le message est signé et que la passerelle destinataire exige une signature valide, toute modification est détectée.
- **Créer des messages au nom de l’expéditeur :** il ne dispose pas de la clé privée de signature du domaine expéditeur.

Il peut :

- **Exploiter les métadonnées :** l’expéditeur, le destinataire, l’heure, la taille et, en règle générale, l’objet sont lisibles. Un objet tel que « Rapport de sortie Max Muster » constitue déjà une communication de données personnelles.
- **Conserver des messages et les déchiffrer ultérieurement :** S/MIME n’offre pas de Perfect Forward Secrecy. Si la clé privée du domaine est compromise ultérieurement, tous les messages conservés deviennent lisibles. Il en va de même à long terme si RSA devient vulnérable aux ordinateurs quantiques.
- **Supprimer ou retarder des messages :** l’expéditeur ne le remarque pas nécessairement. La disponibilité est affectée, ce qui est pertinent en présence de délais.
- **Modifier des en-têtes non protégés :** il peut par exemple réécrire l’objet ou `Reply-To` afin de détourner les réponses. Le contenu chiffré reste inchangé.
- **Manipuler délibérément le texte chiffré :** pour les messages uniquement chiffrés, non signés et utilisant AES-CBC, il est possible de modifier le texte chiffré de manière à ce que certaines parties du texte en clair puissent être divulguées après déchiffrement. C’est le principe des attaques EFAIL de 2018. Une vérification obligatoire de la signature ou un chiffrement authentifié avec AES-GCM selon S/MIME 4.0 l’empêche.

Un texte en clair intercepté lorsque seul le transport est chiffré constitue une fuite complète de données. Un message S/MIME intercepté concerne essentiellement les métadonnées et la disponibilité.

## Qualification juridique

Aucune loi suisse ne prescrit S/MIME, le TLS-Pinning ou une autre procédure particulière. Les exigences sont technologiquement neutres et fixent des objectifs de protection. Le choix de la procédure doit néanmoins être évalué au regard de ces objectifs et être documenté.

| Réglementation | Contenu | Importance pour le choix de la procédure |
|---|---|---|
| LPD art. 8, OPDo art. 1 à 3 | sécurité adéquate des données, objectifs de protection, contrôle du transport | protection lors de la communication, mesurée selon le risque et l’état de la technique |
| LPD art. 61 let. c | amende en cas de violation des exigences minimales en matière de sécurité des données | responsabilité pénale personnelle de la personne responsable |
| LPD art. 9 | traitement par des sous-traitants | tout système ayant accès au texte en clair doit être pris en compte |
| LPD art. 16 et 17 | communication à l’étranger | traitement de textes en clair sur des systèmes étrangers |
| LPD art. 24 | annonce des violations de la sécurité des données | le risque pour les personnes concernées dépend du fait que les données divulguées soient lisibles ou non |
| CP art. 321, LPD art. 62, LPGA art. 33 | secret professionnel, obligation de discrétion professionnelle, obligation de garder le secret dans les assurances sociales | le contenu doit être protégé contre toute prise de connaissance non autorisée |
| CPC art. 177 | fichiers électroniques en tant que titres | la signature renforce la preuve de l’origine et de l’intégrité |

### Sécurité des données selon la LPD et l’OPDo

L’art. 8 LPD exige une sécurité des données appropriée au risque. L’OPDo concrétise les objectifs de protection que sont la confidentialité, la disponibilité, l’intégrité et la traçabilité et cite notamment, à l’art. 3, le contrôle du transport : lors de leur communication, les données personnelles ne doivent pas pouvoir être lues, copiées, modifiées ou supprimées sans autorisation. Des exigences plus élevées s’appliquent en conséquence aux données personnelles sensibles selon l’art. 5 let. c LPD, telles que les données de santé ou les données relatives à des mesures d’aide sociale.

Le TLS-Pinning remplit le contrôle du transport pour un segment de la transmission. Toutefois, le message est stocké sur plusieurs systèmes avant d’atteindre sa destination et y est lisible en clair. Avec le chiffrement de domaine, le contenu est protégé sur l’ensemble du trajet entre les deux organisations. La signature couvre en outre l’objectif de protection de l’intégrité, que TLS ne peut plus démontrer après la distribution.

### Responsabilité pénale personnelle

Selon l’art. 61 let. c LPD, quiconque enfreint intentionnellement les exigences minimales en matière de sécurité des données prévues à l’art. 8 al. 3 LPD est puni, sur plainte, d’une amende pouvant atteindre CHF 250'000. L’amende vise la personne physique responsable, et non l’entreprise. Un chiffrement du contenu conforme à l’état actuel de la technique, documenté dans une évaluation des risques, constitue pour ces personnes une preuve plus solide qu’une règle de transport applicable à des serveurs individuels.

**Remarque UE :** à l’art. 32, le RGPD exige également des mesures techniques et organisationnelles appropriées et mentionne expressément le chiffrement. Les amendes prévues à l’art. 83 RGPD s’adressent toutefois à l’entreprise et peuvent atteindre, en cas de violation de l’art. 32, 10 millions d’euros ou 2 % du chiffre d’affaires annuel mondial.

### Sous-traitants et communication à l’étranger

Chaque système situé sur le parcours qui traite le texte en clair doit être considéré juridiquement : services de filtrage, relais, passerelles cloud, services d’archivage. Lorsque des tiers exploitent ces systèmes, il s’agit en règle générale d’un traitement par des sous-traitants au sens de l’art. 9 LPD, avec les contrats et obligations de contrôle correspondants. Si les systèmes se trouvent à l’étranger, les règles relatives à la communication à l’étranger prévues aux art. 16 et 17 LPD s’ajoutent.

Avec le chiffrement de domaine, ces systèmes ne traitent que du texte chiffré. Le cercle des entités pouvant accéder au contenu est limité aux deux passerelles. Les métadonnées restent toutefois des données personnelles, et les obligations précitées continuent de s’y appliquer.

### Obligation d’annoncer les violations de la sécurité des données

Une violation de la sécurité des données susceptible d’entraîner un risque élevé pour les personnes concernées doit être annoncée au PFPDT selon l’art. 24 LPD. Si des messages sont divulgués via un serveur intermédiaire compromis, l’appréciation dépend du fait que les contenus soient lisibles. S’ils sont fortement chiffrés et que les clés ne sont pas concernées, le risque pour les personnes concernées est généralement faible, pour autant qu’aucune information sensible ne figure dans l’objet. Lorsque seul le transport est chiffré, les contenus sont disponibles en clair pour l’attaquant.

### Secrets professionnels et obligations de discrétion

Lorsque des données sont soumises à un secret professionnel selon l’art. 321 CP, par exemple pour les médecins, les avocats ou leurs auxiliaires, ou à une obligation légale de garder le secret telle que l’art. 33 LPGA dans le domaine des assurances sociales, le contenu doit être protégé contre toute prise de connaissance non autorisée. Cela inclut les exploitants de systèmes intermédiaires. L’art. 62 LPD réprime en outre la violation du secret professionnel par les personnes qui ne sont pas soumises à l’art. 321 CP. Seul un chiffrement du contenu satisfait à cette exigence.

### Valeur probante de la signature

Selon l’art. 177 CPC, les fichiers électroniques sont des titres dont le tribunal apprécie librement la force probante. Une signature S/MIME valide du domaine expéditeur prouve que le message provient de ce domaine et qu’il n’a pas été modifié après la signature. Cela renforce la preuve en cas de délais, d’engagements ou d’instructions. Elle ne remplace pas une signature : seules les signatures électroniques qualifiées selon ZertES satisfont aux exigences de forme du CO. Le TLS-Pinning ne permet de prouver ni l’origine ni l’intégrité après la distribution.

## MTA-STS et DANE en remplacement du TLS-Pinning

Lorsque le pinning disparaît, les propriétés qu’offre uniquement la protection du transport font défaut : TLS imposé, serveur de réception authentifié et donc protection des métadonnées contre l’écoute et le détournement. MTA-STS et DANE fournissent précisément ces propriétés, sous la forme d’un standard que le destinataire publie une fois pour tous les expéditeurs.

**MTA-STS (RFC 8461) :** le destinataire publie sous `https://mta-sts.<domain>/.well-known/mta-sts.txt` une politique contenant les noms d’hôte MX autorisés et le mode `enforce`. Un expéditeur prenant en charge MTA-STS récupère la politique, l’enregistre et ne remet désormais les messages que via TLS avec un certificat valide à l’un des MX mentionnés. Un MX falsifié ou un serveur ne disposant pas d’un certificat valide pour le nom MX est refusé.

**DANE (RFC 7672) :** le destinataire publie dans sa zone signée DNSSEC un enregistrement TLSA contenant le hash de son certificat ou de sa clé. L’expéditeur vérifie le certificat du MX par rapport à cet enregistrement. Techniquement, DANE est un pinning que le destinataire publie lui-même dans le DNS et sécurise avec DNSSEC.

| | TLS-Pinning | MTA-STS | DANE |
|---|---|---|---|
| Géré par | expéditeur, manuellement pour chaque partenaire | destinataire, une fois pour tous les expéditeurs | destinataire, une fois pour tous les expéditeurs |
| S’applique à | un partenaire configuré | tous les expéditeurs prenant en charge MTA-STS | tous les expéditeurs prenant en charge DANE |
| Renouvellement du certificat | le pin doit être mis à jour chez l’expéditeur | sans adaptation tant que le nouveau certificat est valide | sans adaptation avec la même clé, sinon publier à l’avance le nouvel enregistrement TLSA |
| Base de confiance | exactement ce certificat ou cette clé | autorités de certification publiques | DNSSEC |
| Faiblesse | charge opérationnelle, assouplissement en cas de perturbation | première récupération et cache expiré (Trust on First Use), délivrance erronée par une AC | requiert DNSSEC |
| Rapports d’erreur | aucun | TLS-RPT (RFC 8460) | TLS-RPT (RFC 8460) |

Pour la documentation requise par l’art. 8 LPD, MTA-STS et DANE présentent un autre avantage : TLS-RPT permet au destinataire de recevoir quotidiennement des rapports indiquant si les distributions ont été chiffrées ou ont échoué. Le pinning ne fournit pas ce retour. L’article [Enregistrements DNS pour les e-mails](https://rafaelpfister.ch/blog/dns-records-e-mail-stolpersteine) décrit la configuration des deux procédés dans la section consacrée à MTA-STS, TLS-RPT et DANE.

## Qui doit vérifier MTA-STS et DANE

La vérification doit toujours être effectuée par le système qui remet directement le message au MX du destinataire. Les enregistrements doivent être publiés par l’exploitant du MX.

Avec un service de messagerie sécurisée, le chemin sortant se présente généralement comme suit :

```text
Eigener Mailserver -> Secure-Mail-Dienst (verschlüsselt, signiert) -> MX des Partners
```

Le dernier segment était jusqu’ici protégé par le pinning. Le service doit ici vérifier MTA-STS et DANE du partenaire ; votre propre serveur de messagerie n’a aucune influence sur ce segment. Vous sécurisez vous-même le segment entre votre serveur de messagerie et le service, au moyen de TLS imposé et de la vérification du certificat sur le connecteur.

Côté réception, tout dépend du MX. Si le MX pointe vers le service, ce dernier doit fournir des certificats appropriés et vous publiez des enregistrements MTA-STS et TLSA correspondant à ses serveurs. Pour DANE, cela suppose que le service annonce les changements de clé suffisamment tôt. Si le MX pointe vers votre propre passerelle, par exemple Exchange Online, vous y configurez MTA-STS et DANE. Si les deux organisations utilisent le même service, celui-ci peut remettre le message au sein de sa propre plateforme ; il n’existe alors aucun segment SMTP public entre les deux organisations.

Si le service ne prend en charge ni MTA-STS ni DANE lors de l’envoi, le contenu reste protégé par S/MIME. La protection des métadonnées et l’authentification du serveur de réception sur le dernier segment restent alors ouvertes. Ce risque résiduel doit figurer dans l’évaluation des risques.

## Risques résiduels et mesures

| Risque résiduel | Mesure |
|---|---|
| les données personnelles dans l’objet sont lisibles | règle : aucune donnée personnelle dans l’objet, convenue par écrit avec le partenaire |
| absence de Perfect Forward Secrecy | stocker la clé privée du domaine de manière non exportable, idéalement dans un HSM ; journaliser les accès |
| manipulation de messages non signés | obligation de signature des deux côtés, AES-GCM si les deux passerelles le prennent en charge |
| remise en clair en cas d’erreur | retenir les messages en cas d’échec du chiffrement au lieu de les remettre en clair |
| métadonnées sur le chemin de transport | MTA-STS en mode `enforce` et, là où DNSSEC est disponible, DANE ; analyser TLS-RPT |
| certificats expirés | information mutuelle au moins 30 jours avant l’expiration, comparaison des empreintes via un second canal |

La décision d’opter pour le chiffrement de domaine ainsi que les risques restants doivent être intégrés au concept de sécurité des données ou à la documentation des activités de traitement.

## Questions à poser au prestataire de passerelle

1.  Lors de l’envoi, le service vérifie-t-il les politiques MTA-STS en mode `enforce` et les enregistrements DANE du domaine destinataire ?

2.  Depuis quels serveurs les messages sont-ils remis, et est-il possible de consulter les rapports TLS-RPT ou les journaux de distribution relatifs au chiffrement du transport ?

3.  En tant que MX, le service prend-il en charge MTA-STS et DANE pour les messages entrants, et comment les changements de clé sont-ils annoncés ?

4.  Le connecteur entre votre propre serveur de messagerie et le service peut-il être exploité avec TLS imposé et vérification du certificat ?

5.  La remise entre deux clients du service s’effectue-t-elle via SMTP public ou au sein de la plateforme ?

6.  Quels algorithmes sont utilisés pour le chiffrement et la signature, et AES-GCM est-il pris en charge ?

## Sources

1.  [Loi fédérale sur la protection des données (LPD)](https://www.fedlex.admin.ch/eli/cc/2022/491/de) : art. 8 sur la sécurité des données, art. 9 sur le traitement par des sous-traitants, art. 16 et 17 sur la communication à l’étranger, art. 24 sur l’obligation d’annoncer, art. 61 et 62 sur les dispositions pénales.

2.  [Ordonnance sur la protection des données (OPDo)](https://www.fedlex.admin.ch/eli/cc/2022/568/de) : art. 1 à 3 sur les principes, les objectifs de protection et les mesures techniques et organisationnelles, notamment le contrôle du transport.

3.  [Code pénal suisse (CP)](https://www.fedlex.admin.ch/eli/cc/54/757_781_799/de) : art. 321 sur la violation du secret professionnel.

4.  [Loi fédérale sur la partie générale du droit des assurances sociales (LPGA)](https://www.fedlex.admin.ch/eli/cc/2002/510/de) : art. 33 sur l’obligation de garder le secret des organes d’exécution.

5.  [Code de procédure civile (CPC)](https://www.fedlex.admin.ch/eli/cc/2010/262/de) : art. 177 sur la notion de titre, y compris les fichiers électroniques.

6.  [RFC 8461: SMTP MTA Strict Transport Security (MTA-STS)](https://www.rfc-editor.org/rfc/rfc8461) : spécification de la politique, des règles de récupération et du mode `enforce`.

7.  [RFC 7672: SMTP Security via Opportunistic DANE TLS](https://www.rfc-editor.org/rfc/rfc7672) : DANE pour SMTP, enregistrements TLSA et règles de vérification pour les serveurs expéditeurs.

8.  [RFC 8460: SMTP TLS Reporting](https://www.rfc-editor.org/rfc/rfc8460) : format et remise des rapports TLS-RPT.

9.  [RFC 8551: S/MIME Version 4.0 Message Specification](https://www.rfc-editor.org/rfc/rfc8551) : spécification S/MIME actuelle avec AuthEnvelopedData et AES-GCM.

10.  [EFAIL](https://efail.de/) : description des attaques publiées en 2018 contre des messages S/MIME et OpenPGP non signés en mode CBC.

11.  [Règlement général sur la protection des données (RGPD)](https://eur-lex.europa.eu/legal-content/DE/TXT/?uri=CELEX:32016R0679) : art. 32 sur la sécurité du traitement, art. 83 sur les amendes (remarque UE).
