---
title: "Downdetector sous le feu des critiques : ce que la semaine de pannes aux États-Unis révèle sur les signalements d’incidents"
navTitle: "Critiques de Downdetector"
description: "Entre le 13 et le 17 novembre 2026, Downdetector a enregistré aux États-Unis autant de pannes qu’il n’en avait plus vu depuis un an. Les incidents touchant la FAA, AWS, Chase, United et Apple étaient réels, mais avaient des causes distinctes et ont généralement été résolus en quelques heures. Les chiffres de Downdetector ont renforcé l’image d’une crise interconnectée. Ce que mesure la plateforme, d’où viennent les critiques et comment les administrateurs peuvent mieux interpréter les incidents."
date: "2026-11-18"
kategorie: "Exploitation"
timeToRead: "7 min de lecture"
themen: []
produkte: []
protokolle:
  - "troubleshooting"
slug: "downdetector-sous-le-feu-des-critiques-ce-que-la-semaine-de-pannes-aux-etats-unis-revele-sur"
translationId: article-0761d100437e6b25
translationOf: downdetector-kritik-ausfallwoche
url: https://rafaelpfister.ch/fr/blog/downdetector-sous-le-feu-des-critiques-ce-que-la-semaine-de-pannes-aux-etats-unis-revele-sur
translationSourceHash: 3f7c9f1263f57a9ba194011c81d634786e31250e328f8cc02bb45a26564c46d8
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T08:44:37.501Z
translationReview: automatic
---

# Downdetector sous le feu des critiques : ce que la semaine de pannes aux États-Unis révèle sur les signalements d’incidents

Du 13 au 17 novembre 2026, plusieurs grands systèmes sont tombés en panne aux États-Unis : le contrôle aérien de la FAA avec une interdiction nationale de décollage, certaines parties d’AWS dans la région us-east-1, la banque JPMorgan Chase, les opérations au sol de United Airlines et, mardi, la connexion à iCloud ainsi que d’autres services d’Apple. J’ai suivi cette série dans un [fil en direct](https://rafaelpfister.ch/liveticker/usa-ausfallserie-2026-11). À l’issue de la semaine, le constat est clair : les pannes avaient des causes distinctes, rien n’indique une cyberattaque et l’Europe n’a été touchée qu’à la marge. Dans l’opinion publique, l’image d’un Internet qui tombe simultanément en panne de toutes parts s’est pourtant imposée. Une grande partie de cette image provient des graphiques de Downdetector, cités dans presque tous les articles de presse. C’est là que s’inscrit la critique généralisée depuis lundi.

## Ce que mesure Downdetector

Downdetector appartient à Ookla et donc, depuis le 17 juin 2026, à Accenture. La plateforme collecte les signalements de problèmes de la part d’utilisateurs qui signalent un incident sur la page d’un service et les complète, selon sa propre description, par des signaux issus des médias sociaux et d’autres sources en ligne. Pour chaque service, Downdetector calcule une référence : le nombre moyen de signalements à la même heure de la journée au cours de l’année écoulée. Si le nombre actuel est nettement supérieur, il est considéré comme une panne. Downdetector ne publie ni la pondération des sources ni le facteur à partir duquel un pic est considéré comme une panne.

Il en découle trois caractéristiques souvent négligées dans la couverture médiatique :

1.  **Les signalements ne correspondent pas au nombre d’utilisateurs touchés.** Un chiffre tel que « 20’000 signalements » ne dit rien du nombre de clients qui n’avaient effectivement plus accès au service. Il compte les personnes qui ont constaté un problème et consulté Downdetector.

2.  **Les utilisateurs signalent le service qu’ils voient.** Lorsqu’un réseau de paiement, un fournisseur cloud ou un service d’identité tombe en panne, les signalements apparaissent chez les banques, applications et boutiques qui en dépendent. La cause n’est pas identifiable dans le graphique.

3.  **La référence ne reflète pas les semaines exceptionnelles.** Lorsque les médias parlent continuellement de pannes, davantage de personnes vérifient, au moindre retard, si un service est perturbé. Chaque nouveau titre augmente le nombre de signalements, même pour des services qui fonctionnent normalement.

## Les chiffres de la semaine

Selon Axios, Downdetector a reçu près de 6 millions de signalements de problèmes en provenance des États-Unis au cours des sept jours précédant le 16 novembre, soit le total hebdomadaire le plus élevé depuis un an. La même analyse montre toutefois que, sur 30 jours, il y a eu trois pannes majeures, contre cinq habituellement. Le nombre record de signalements contraste donc avec un nombre de pannes majeures réelles plutôt inférieur à la moyenne.

Plusieurs cas de la semaine montrent à quel point les signalements peuvent différer de la situation réelle :

- **Banques le 16 novembre :** les médias ont écrit, en se référant à Downdetector, que des utilisateurs signalaient des problèmes auprès de « pratiquement toutes les grandes banques américaines », notamment Chase, Wells Fargo et Capital One. Chase a elle-même indiqué que certaines transactions prenaient plus de temps ; les clients continuaient d’avoir accès à leurs comptes et à leurs soldes. Selon le New York Times, la cause était un bug logiciel dans le matériel propre à JPMorgan Chase, qui aurait pu affecter d’autres banques via des transactions partagées. Les autres banques n’ont pas confirmé leurs propres incidents.

- **Gmail les 16 et 17 novembre :** Downdetector a affiché plus de 2’800 signalements, principalement concernant des e-mails non distribués. Google a déclaré n’avoir aucun indice d’une panne généralisée ; le Workspace Status Dashboard ne comportait aucune entrée.

- **Charles Schwab le 17 novembre :** selon le Wall Street Journal, les signalements ont augmenté vers 9 h 50, heure locale, puis sont retombés une demi-heure plus tard. Aucun avis d’incident de Schwab n’existe à ce sujet.

- **Exchange Online le 12 novembre :** Reuters a fait état de plus de 97’000 signalements pour la même panne, contre près de 119’000 pour une chaîne locale, tous deux à des moments différents. Selon Downdetector, les signalements n’ont commencé à augmenter qu’environ 68 minutes après le début indiqué par Microsoft dans l’Admin Center.

Les pannes de la FAA, d’AWS, de Chase, de United et d’Apple étaient réelles et confirmées par les opérateurs. C’est leur mise en relation qui était exagérée : cinq incidents indépendants et une hausse générale des signalements ont donné naissance au récit d’une attaque coordonnée. La porte-parole de la Maison-Blanche a déclaré le 17 novembre qu’il n’y avait pas de crise dont les Américains devaient s’inquiéter. Liz Fong-Jones (Honeycomb) a déclaré à Axios que les incidents avaient très peu de choses en commun.

## D’où vient la critique

Les critiques viennent de plusieurs côtés. Des opérateurs tels que Google contredisent publiquement Downdetector lorsque celui-ci affiche une panne qu’ils ne constatent pas dans leurs propres données. Les médias spécialisés soulignent que Downdetector compte les signalements plutôt que les utilisateurs touchés, affiche les pannes avec retard et ne détecte pas du tout les défaillances de composants invisibles pour les clients finaux. Dans les discussions entre développeurs, notamment sur Hacker News, circule en outre l’hypothèse que le simple appel d’une page telle que « Chase est-il en panne ? » compte comme un signalement implicite et que les pics se propagent ainsi d’un service à des services similaires. Downdetector n’a pas confirmé cette hypothèse ; il est seulement établi que la plateforme analyse, en plus des signalements actifs, d’autres sources en ligne et ne divulgue pas leur pondération.

Les défenseurs de la plateforme rétorquent que Downdetector montre souvent les pannes plus tôt que les pages de statut des fournisseurs. Lors de la panne d’Apple le 17 novembre, la page de statut d’Apple indiquait encore une disponibilité normale alors que les signalements augmentaient déjà fortement. Le service est utile comme alerte précoce. En revanche, il ne constitue pas une mesure de l’ampleur d’une panne, et c’est précisément ainsi qu’il a été utilisé cette semaine : des titres tels que « Des milliers de personnes signalent des problèmes » côtoyaient, sans mise en contexte, des pannes confirmées.

## Mieux interpréter les incidents

Pour les administrateurs qui doivent décider, face à un incident signalé, s’ils doivent informer les utilisateurs ou prendre des mesures, une séquence simple se dégage :

1.  **Vérifier la page de statut et le portail d’administration du fournisseur.** Pour Microsoft 365, il s’agit de Service Health dans l’Admin Center, et non de la page de statut publique. AWS, Google Cloud, Cloudflare et Apple disposent de leurs propres pages de statut horodatées.

2.  **S’appuyer sur ses propres mesures.** Un simple test synthétique (connexion, envoi d’e-mail, consultation d’une page) depuis son propre réseau montre si sa propre organisation est touchée. Mon [moniteur de connectivité M365](https://rafaelpfister.ch/liveticker) vérifie toutes les cinq minutes l’accessibilité des points de terminaison Microsoft 365 depuis l’Europe.

3.  **Ne considérer Downdetector que comme un indice.** Un pic est une raison de vérifier. Le chiffre absolu et les pics simultanés chez plusieurs services similaires disent peu de chose sur la cause et l’ampleur.

4.  **Tenir compte de l’origine régionale.** La plupart des chiffres cités dans les médias se rapportent à la version américaine de Downdetector. Pour les utilisateurs en Suisse, ils ne sont pertinents que si le fournisseur confirme également l’incident pour les régions européennes.

## Le fil en direct

J’ai clôturé le fil en direct consacré à la série de pannes avec cet article. Il reste disponible en ligne comme chronique de la semaine, avec une conclusion sur les différentes causes et la précision que l’Europe n’a pas été touchée, sauf pour les vols transatlantiques et la panne mondiale d’iCloud.

## Sources

1.  [Axios : Tech outages spark fears over AI and cybersecurity](https://www.axios.com/2026/11/17/tech-outages-ai-fears-internet-breaking) : près de 6 millions de signalements Downdetector en sept jours, trois pannes majeures au lieu des cinq habituelles en 30 jours, évaluation de Liz Fong-Jones.

2.  [PPC Land : Downdetector](https://ppc.land/downdetector/) : fonctionnement, référence par heure de la journée, acquisition par Accenture, chiffres divergents concernant la panne d’Exchange Online du 12 novembre.

3.  [Futurism : Wait, Why Did Every Major Bank Just Go Down?](https://futurism.com/future-society/banks-go-down-outage) : signalements Downdetector auprès de plusieurs banques américaines, déclaration de Chase.

4.  [Hot Air : More Outages Today, Still No Evidence of Cyberattacks](https://hotair.com/john-s-2/2026/11/17/more-outages-today-still-no-evidence-of-cyberattacks-n3821101) : cite le New York Times et le Wall Street Journal au sujet d’Apple, de Charles Schwab et de la cause chez JPMorgan Chase.

5.  [Pasquale Pillitteri : Gmail down, Google denies a broad outage after 2,800 reports](https://pasqualepillitteri.it/en/news/34509/gmail-down-google-denies-broad-outage-2800-reports) : signalements Downdetector concernant Gmail et démenti de Google.

6.  [Forbes : Why Are So Many Outages Happening? Companies Deny Cyberattacks](https://www.forbes.com/sites/conormurray/2026/11/17/outages-plague-america-these-are-the-explanations/) : aperçu des explications des opérateurs.

7.  [Time : What to Know About the Apple Outage](https://time.com/article/2026/11/17/apple-outage-down-update-issue/) : déroulement de la panne d’Apple, page de statut en retard sur les signalements.

8.  [Hacker News : Discussion sur la méthodologie de Downdetector](https://news.ycombinator.com/item?id=50521819) : hypothèses sur les signalements implicites et leur propagation entre services similaires, non confirmées par Downdetector.

9.  [WMTV : Apple, banks, FAA and Microsoft outages raise questions](https://www.wmtv15news.com/2026/11/18/apple-banks-faa-microsoft-outages-raise-questions-cybersecurity-expert-weighs/) : évaluation d’un expert en sécurité, aucun indice d’un lien entre les incidents.
