---
title: "Kiteworks : le fabricant recommande un arrêt le 26 septembre – ce que l’on sait à ce jour"
navTitle: "Arrêt de Kiteworks"
description: "Kiteworks demande à ses clients par e-mail d’arrêter tous les systèmes le samedi 26.09.2026, de 04:00 à 10:00. La raison est un avertissement des autorités chargées de l’application de la loi concernant une possible attaque. Depuis le 27.09, la recommandation est levée ; il n’y a ni CVE ni nouveau correctif. TotemoMail n’est pas concerné."
date: "2026-09-25"
kategorie: "Kiteworks / TotemoMail"
timeToRead: "9 min de lecture"
themen:
  - totemomail
  - sicherheitsluecken
produkte:
  - "totemomail"
protokolle:
  - "verschluesselung"
  - "haertung"
  - "smtp"
hauptthema: "totemomail"
slug: "kiteworks-le-fabricant-recommande-un-arret-le-26-septembre-ce-que-l-on-sait-jusqu-a-present"
featured: "2026-09-27"
translationId: "article-38fbaa0e9095957a"
aiPrompt: |
  Du bist mein Exchange- und Mailflow-Assistent. Kiteworks empfiehlt, alle Systeme am 26.09.2026 von 04:00 bis 10:00 Uhr herunterzufahren. Hilf mir zu ermitteln, welche Connectoren, Transportregeln und MX-Einträge in meiner Umgebung Mails über das Gateway leiten, wie ich den Mailflow für die Dauer der Abschaltung umleite oder kontrolliert anhalte und wie ich den ursprünglichen Zustand danach wiederherstelle. Frage zuerst nach meinem Aufbau (Exchange Online, Exchange Server oder Hybrid, Richtung des Mailflows, Position des Gateways).
translationOf: kiteworks-zero-day-abschaltung
translationSourceHash: 93bc9f973258d524a87baa5fe75957444b339bcac669281db049e3f1e5817813
translationModel: gpt-5.6-terra
translatedAt: 2026-09-28T09:57:08.625Z
translationReview: required
url: https://rafaelpfister.ch/fr/blog/kiteworks-le-fabricant-recommande-un-arret-le-26-septembre-ce-que-l-on-sait-jusqu-a-present
---

# Kiteworks : le fabricant recommande un arrêt le 26 septembre – ce que l’on sait à ce jour

Le 25 septembre 2026, Kiteworks a demandé à ses clients par e-mail d’arrêter tous les systèmes Kiteworks le samedi 26 septembre, de 04:00 à 10:00 (heure d’Europe centrale). Selon le courrier du CISO Frank Balonis, le fabricant dispose d’informations provenant des autorités chargées de l’application de la loi indiquant qu’une attaque contre des systèmes Kiteworks pourrait avoir lieu ce week-end. Le support client justifie l’arrêt par la protection contre d’éventuelles attaques zero-day. heise online a confirmé l’authenticité du message par téléphone auprès du support.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Assistance d’urgence pour la bascule du flux de messagerie</p>
<p>Si vous avez besoin d’aide pour rediriger le flux de messagerie avant l’arrêt, puis le rétablir ensuite, veuillez utiliser le <a href="https://adeptio.ch/">formulaire de contact sur adeptio.ch</a>. Je vous répondrai également à court terme.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Mise à jour du 28 septembre 2026 : Kiteworks lève la recommandation d’arrêt</p>
<p>Kiteworks a complété le communiqué de presse par une indication : depuis le 27 septembre, la recommandation d’arrêt ne s’applique plus à aucun client.</p>
<blockquote lang="en">
<p>As of September 27th, the shutdown recommendation is now lifted for all customers. If you have not already restarted, you may bring your Kiteworks system back online. Customers with self-hosted Advanced Forms should contact Customer Support for assistance. All systems Kiteworks hosts on customers’ behalf have been brought back up and are operating normally.</p>
</blockquote>
<p>Toute personne n’ayant pas encore redémarré ses systèmes peut désormais le faire. Les personnes exploitant elles-mêmes Advanced Forms doivent contacter le support de Kiteworks avant le redémarrage. Les instances hébergées par Kiteworks sont de nouveau opérationnelles. Il n’existe toujours pas de numéro CVE, pas de nouvelle version au-delà de 9.5.1, pas d’indicateur de compromission ni d’information indiquant si une attaque a été tentée ou ce qui motivait l’avertissement. La page des mises à jour de sécurité et les avis GitHub restent inchangés.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Mise à jour du 25 septembre 2026 : déclaration de Kiteworks</p>
<blockquote lang="en">
<p>Kiteworks received credible threat intelligence from law enforcement indicating that a threat actor may attempt to target some Kiteworks systems for customers. Out of an abundance of caution, we notified customers directly and recommended a precautionary shutdown window while we and our law enforcement partners work through the matter. We are not aware of any compromise of Kiteworks systems, and this advisory is preventative rather than a response to a confirmed breach. All known vulnerabilities are addressed in our current release, 9.5.1, and we continue to recommend customers run the latest version.</p>
<p>totemomail is not affected by this.</p>
</blockquote>
<p><strong>Totemomail n’est pas concerné.</strong> Il reste à déterminer si Kiteworks EPG (Email Protection Gateway) est concerné.</p>
</div>

## Chronologie

Toutes les heures sont indiquées en heure d’été d’Europe centrale (CEST). Lorsqu’aucune heure n’est indiquée, aucune information horaire fiable n’est disponible.

<ol class="timeline">
<li class="timeline__item">
<p class="timeline__zeit">Ven. 25 septembre</p>
<p class="timeline__titel">Avis aux clients</p>
<p>Le CISO Frank Balonis informe les clients par e-mail d’informations des autorités chargées de l’application de la loi concernant une possible attaque ce week-end et recommande un arrêt de six heures. Selon l’avis, toutes les vulnérabilités connues sont corrigées dans la version 9.5.1.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Ven. 25 septembre</p>
<p class="timeline__titel">Premiers articles de presse</p>
<p>heise online rapporte que le support de Kiteworks confirme l’authenticité du message et justifie l’arrêt par la protection contre d’éventuelles attaques zero-day. TechCrunch, BleepingComputer, Computer Weekly et d’autres suivent peu après.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Ven. 25 septembre, 17:41</p>
<p class="timeline__titel">Le BKA ne s’exprime pas</p>
<p>heise ajoute : le BKA refuse de s’exprimer pour des raisons tactiques liées à l’enquête. Le BSI ne répond pas, et le FBI refuse tout commentaire auprès de TechCrunch.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Ven. 25 septembre</p>
<p class="timeline__titel">Déclaration et communiqué de presse</p>
<p>Kiteworks qualifie l’arrêt de mesure de précaution, sans compromission connue. Le communiqué de presse cite des « autorités fédérales de renseignement » comme source et énumère les filiales non concernées, dont totemo. Le fabricant arrête lui-même les instances hébergées par Kiteworks.</p>
</li>
<li class="timeline__item timeline__item--fenster">
<p class="timeline__zeit">Sam. 26 septembre, de 04:00 à 10:00</p>
<p class="timeline__titel">Fenêtre d’arrêt</p>
<p>La fenêtre est simultanée dans le monde entier : de 02:00 à 08:00 UTC, de 12:00 à 18:00 à Sydney, et de vendredi 22:00 à samedi 04:00 à New York.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Sam. 26 septembre, 10:00</p>
<p class="timeline__titel">Fin de la fenêtre</p>
<p>La fenêtre indiquée dans l’e-mail client prend fin. La levée formelle de la recommandation suit le 27 septembre.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Dim. 27 septembre</p>
<p class="timeline__titel">Recommandation levée</p>
<p>Kiteworks complète le communiqué de presse : la recommandation d’arrêt est levée pour tous les clients, les systèmes peuvent de nouveau fonctionner. Les instances hébergées sont de nouveau en service. Les clients exploitant eux-mêmes Advanced Forms doivent contacter le support.</p>
</li>
<li class="timeline__item timeline__item--offen">
<p class="timeline__zeit">État au lundi 28 septembre</p>
<p class="timeline__titel">Toujours en suspens</p>
<p>Aucun avis public, aucun numéro CVE, aucune nouvelle version, aucun indicateur, aucune information sur la faille et aucun rapport d’attaque réussie ou tentée.</p>
</li>
</ol>

## Ce que l’on sait

La recommandation s’applique dans le monde entier ; l’e-mail indique la fenêtre pour tous les fuseaux horaires, de l’AEST au PDT. Kiteworks conseille d’arrêter les systèmes avant même le début de la fenêtre, y compris s’ils ne sont pas accessibles depuis Internet.

Presque tout le reste demeure inconnu : il n’existe aucun avis de sécurité public, aucun numéro CVE, aucun correctif et aucune indication sur les produits ou versions concernés. Le communiqué de presse cite des « autorités fédérales de renseignement » comme source, vraisemblablement des autorités fédérales américaines ; lesquelles ne sont pas connues. Au 28 septembre, aucune entrée n’apparaît dans Security Updates ni dans les avis GitHub de Kiteworks ; la dernière entrée GitHub date du 27 mai 2026. Sont publics la déclaration citée ci-dessus et le communiqué de presse du 25 septembre.

Auprès de TechCrunch, le CISO de Kiteworks, Frank Balonis, a fait la déclaration dans les mêmes termes. Le BKA a refusé de s’exprimer auprès de heise pour des raisons tactiques liées à l’enquête, tandis que le BSI n’a pas répondu. Le FBI n’a pas souhaité s’exprimer auprès de TechCrunch, et un porte-parole de la CISA n’a pas souhaité faire de déclaration publique. Selon TechCrunch, un client du secteur de la santé a immédiatement déconnecté son serveur, avec des restrictions opérationnelles perceptibles : les médecins n’ont temporairement pu joindre leurs patients qu’avec retard. Selon un chercheur en sécurité cité par TechCrunch, au moins 1 000 systèmes Kiteworks sont accessibles depuis Internet ; BornCity évoque plus de 1 000 organisations ayant reçu l’avertissement.

Le communiqué de presse diverge de l’e-mail client sur un point : il évoque une fenêtre d’arrêt de neuf heures, tandis que l’avis aux clients parle de six heures. Selon le communiqué de presse, la recommandation ne concerne que les installations exploitées par les clients eux-mêmes (on-premises, AWS, Azure). Selon le fabricant, les filiales Zivver, DRACOON, totemo, ownCloud, WAMNET, Maytech, Bonfy.ai et 123FormBuilder ne sont pas concernées.

## L’avis aux clients

Outre l’avertissement, l’e-mail client du 25 septembre contient un calendrier par fuseau horaire et des instructions pour les clusters. En convertissant les heures en UTC, toutes les régions obtiennent la même fenêtre, de 02:00 à 08:00 UTC.

| Fuseau horaire | Ville | Début | Fin |
|---|---|---|---|
| AEST (UTC+10) | Sydney | Sam. 12:00 | Sam. 18:00 |
| SGT (UTC+8) | Singapour | Sam. 10:00 | Sam. 16:00 |
| IDT (UTC+3) | Tel Aviv | Sam. 05:00 | Sam. 11:00 |
| CEST (UTC+2) | Amsterdam, Zurich | Sam. 04:00 | Sam. 10:00 |
| BST (UTC+1) | Londres | Sam. 03:00 | Sam. 09:00 |
| EDT (UTC−4) | New York | Ven. 22:00 | Sam. 04:00 |
| CDT (UTC−5) | Chicago | Ven. 21:00 | Sam. 03:00 |
| MDT (UTC−6) | Denver | Ven. 20:00 | Sam. 02:00 |
| PDT (UTC−7) | San Francisco | Ven. 19:00 | Sam. 01:00 |

Pour les clusters comportant plusieurs serveurs, Kiteworks impose un ordre précis :

1.  **Activer le mode maintenance** sous System Setup > Maintenance Mode, afin qu’aucun utilisateur ne puisse plus accéder au système.

2.  **Créer une sauvegarde :** un snapshot de chaque nœud ou une sauvegarde de la base de données Kiteworks (System Setup > Cluster Configuration > System Configuration). Une seule sauvegarde de base de données est conservée ; chaque nouvelle remplace la précédente.

3.  **Identifier les rôles :** sous System Setup > Locations, la colonne Assigned Roles indique quels nœuds ont le rôle Application ; le nœud Application principal est marqué d’un astérisque. Noter les nœuds et leurs adresses IP, car ils seront nécessaires au redémarrage.

4.  **Arrêter dans cet ordre :** d’abord tous les nœuds sans rôle Application, puis les autres nœuds Application, et enfin le nœud Application principal. Cela peut se faire via l’onglet Shut Down du nœud concerné ou via la console de l’hyperviseur (par exemple VMware ou AWS) si l’interface Kiteworks n’est plus accessible.

5.  **Redémarrer dans l’ordre inverse** via l’hyperviseur, car la console d’administration n’est accessible que lorsqu’un nombre suffisant de nœuds fonctionne (annexe E de l’Administrator Guide) : d’abord le nœud Application principal, puis les autres nœuds Application un par un et seulement lorsque le précédent est entièrement opérationnel, afin que les serveurs de base de données puissent former un quorum. Ensuite les serveurs de stockage, puis les autres rôles (Repositories Gateway, Search, SFTP, Antivirus), et enfin les serveurs web.

6.  **Désactiver le mode maintenance** dès que tous les nœuds sont au vert dans le Cluster Health Dashboard de la page d’état de la console d’administration.

Interrogé à ce sujet, le support de Kiteworks a également confirmé qu’aucune des filiales de Kiteworks n’est concernée.

## Causes possibles : théories

Tant que Kiteworks ne publie pas de détails, la cause reste inconnue. Les explications suivantes sont des hypothèses déduites des éléments connus ; certaines sont également discutées dans les commentaires de l’article de heise. Aucune n’est confirmée.

Trois éléments limitent les possibilités. Premièrement, l’avertissement mentionne une fenêtre fixe plutôt qu’un arrêt indéfini jusqu’à la publication d’un correctif. Deuxièmement, même les systèmes non accessibles depuis Internet doivent être déconnectés. Troisièmement, la fenêtre est fixée à la même heure dans le monde entier (de 02:00 à 08:00 UTC), plutôt que pendant la nuit locale dans chaque région. Une faille classique exploitable via Internet n’expliquerait pas les deux premiers points : il suffit de déconnecter le système d’Internet, et ce jusqu’à ce que le correctif soit disponible.

### 1. Les autorités connaissent une échéance planifiée

Les autorités chargées de l’application de la loi apprennent parfois à l’avance la date d’une campagne planifiée, par exemple à partir de communications surveillées d’un groupe criminel ou d’infrastructures saisies. Les exploitations massives de produits d’échange de fichiers se déroulent généralement dans une courte fenêtre coordonnée, souvent les week-ends ou jours fériés, lorsque les effectifs sont réduits. Kiteworks est le successeur d’Accellion, dont la File Transfer Appliance a été attaquée de cette manière en 2020 et 2021, alors attribuée au groupe Clop : des données ont été exfiltrées via plusieurs failles, puis les organisations touchées ont été victimes d’extorsion.

La fenêtre étroitement délimitée pendant le week-end plaide en faveur de cette hypothèse. L’objection soulevée par plusieurs commentateurs sur heise va à l’encontre : l’avertissement a été envoyé à tous les clients, les attaquants devraient donc le savoir et peuvent simplement reporter l’attaque. Un report donnerait toutefois au fabricant le temps de développer un correctif.

### 2. Le fabricant ne connaît pas encore la faille lui-même

Il est également possible que Kiteworks ne dispose, en dehors de l’information des autorités, d’aucun détail technique, et ne connaisse donc ni le composant concerné ni un correctif ou une modification de configuration à recommander. L’arrêt est alors la seule mesure efficace sans connaître la faille, et la fin fixe constitue un compromis que les clients sont plus susceptibles d’accepter. Dans les commentaires de heise, l’hypothèse est avancée que le fabricant pourrait laisser certains systèmes en ligne comme leurres pendant la fenêtre afin d’observer l’attaque. Il n’existe aucune preuve à cet égard.

L’absence d’avis ou de mesure d’atténuation plaide en faveur de cette hypothèse. À l’inverse, Kiteworks affirme collaborer avec Mandiant et, en cas d’avertissement des autorités, des indicateurs sont généralement au moins disponibles.

### 3. Une porte dérobée déjà implantée avec déclenchement temporel

La recommandation d’arrêter également les systèmes internes correspond à un scénario dans lequel l’attaque ne vient pas de l’extérieur, mais a déjà été préparée sur les appliances : par exemple, une porte dérobée issue d’une compromission antérieure, qui s’active à un moment précis ou contacte un serveur de contrôle. Un système arrêté ne peut rien exécuter à ce moment-là.

Le fait que l’accessibilité depuis Internet ne joue aucun rôle dans ce scénario plaide en faveur de cette hypothèse. À l’inverse, dans ce cas, un fabricant recommanderait plutôt une vérification de compromission et une réinstallation qu’un redémarrage après six heures.

### 4. Compromission chez le fabricant

Un autre moyen d’atteindre les systèmes internes consiste en des connexions établies de l’appliance vers le fabricant, par exemple pour les mises à jour, la vérification de licence ou la maintenance à distance. Si un tel canal est compromis, un pare-feu ne protège pas contre le trafic entrant. Dans ce scénario, l’arrêt donnerait au fabricant une fenêtre pour assainir sa propre infrastructure, remplacer des clés ou certificats et n’autoriser à nouveau les connexions qu’ensuite.

Le moment uniforme à l’échelle mondiale, compatible avec une opération coordonnée chez le fabricant, plaide en faveur de cette hypothèse. À l’inverse, le fabricant recommanderait alors plutôt de bloquer les connexions sortantes que d’arrêter entièrement les systèmes.

### 5. Mesure d’accompagnement d’une opération des autorités

Enfin, il est concevable que les autorités interviennent durant la même période contre l’infrastructure des attaquants et souhaitent empêcher que ceux-ci ne frappent rapidement en réaction. Cela expliquerait la courte fenêtre et le rôle des autorités chargées de l’application de la loi. Le refus du BKA de s’exprimer pour des raisons tactiques liées à l’enquête indique des investigations en cours, mais ne prouve pas cette variante.

### Critique de la communication

Les commentaires de heise sont majoritairement sceptiques, et les objections sont objectivement compréhensibles : sans information sur la faille, il est impossible d’évaluer si une coupure d’Internet par pare-feu aurait suffi. Une fenêtre temporelle sans correctif annoncé ne permet pas de savoir ce qui s’applique après 10:00. Et un avertissement envoyé uniquement par e-mail aux clients n’atteint pas tous les exploitants, par exemple chez les partenaires, prestataires ou après des changements de personnel. Quelle que soit la théorie retenue : les exploitants de Kiteworks devraient vérifier les journaux après le redémarrage et suivre les canaux du fabricant jusqu’à ce qu’un avis soit disponible.

## Après la fenêtre : ce que les exploitants peuvent faire maintenant

Kiteworks a levé la recommandation d’arrêt le 27 septembre, mais n’a publié aucun détail technique. Il est donc impossible d’évaluer si et comment le danger a été éliminé. Lors du redémarrage et après celui-ci, les étapes suivantes sont pertinentes :

1.  **Vérifier la version :** la version 9.5.1 est-elle exécutée sur tous les nœuds ? Selon le fabricant, toutes les vulnérabilités connues y sont corrigées.

2.  **Vérifier l’état du cluster :** dans le Cluster Health Dashboard, tous les nœuds devraient être au vert et le mode maintenance désactivé.

3.  **Analyser les journaux :** examiner les connexions, les actions d’administration et les téléchargements inhabituels de fichiers autour de la fenêtre d’arrêt, en particulier pour les systèmes qui n’ont pas été arrêtés ou l’ont été tardivement.

4.  **Restreindre l’accessibilité :** lorsque cela est possible, bloquer l’accès depuis Internet à l’interface d’administration et n’autoriser que les services nécessaires.

5.  **Advanced Forms :** toute personne exploitant elle-même le module doit clarifier le redémarrage au préalable avec le support de Kiteworks.

6.  **Surveiller les canaux :** Security Updates, les avis GitHub, le Newsroom et les e-mails clients de Kiteworks, jusqu’à ce qu’un avis contenant des détails techniques soit disponible.

## Sources

1.  [heise online : Attaque zero-day imminente : KiteWorks pousse ses clients à arrêter leurs serveurs](https://www.heise.de/news/Bevorstehender-Zero-Day-Angriff-KiteWorks-draengt-Kunden-zur-Serverabschaltung-11466114.html): premier article avec des extraits de l’e-mail client et la fenêtre temporelle ; mise à jour du 25.09 à 17:41 avec la réponse du BKA.

2.  [heise online (EN) : Imminent Zero-Day Attack: KiteWorks Urges Customers to Shut Down Servers](https://www.heise.de/en/news/Imminent-Zero-Day-Attack-KiteWorks-Urges-Customers-to-Shut-Down-Servers-11466375.html): version anglaise avec le texte original du CISO.

3.  [Kiteworks : Security Updates](https://www.kiteworks.com/company/security-updates/): canal officiel du fabricant, sans entrée concernant l’avertissement au 28.09.2026.

4.  [Kiteworks : Security Advisories sur GitHub](https://github.com/kiteworks/security-advisories/security): liste des avis du fabricant, dernière entrée du 27.05.2026 au 28.09.2026.

5.  [Kiteworks : Newsroom](https://www.kiteworks.com/newsroom/): communications officielles, avec depuis le 25.09.2026 le communiqué de presse sur l’arrêt.

6.  [Forum heise : commentaires sur l’article](https://www.heise.de/forum/heise-online/Kommentare/Server-am-Samstagmorgen-herunterfahren-Kiteworks-warnt-Admins-vor-Zero-Day/forum-591001/comment/): discussion des lecteurs sur les objections relatives à la fenêtre fixe et à l’arrêt des systèmes internes, ainsi que sur la thèse du leurre.

7.  [CISA : Exploitation of Accellion File Transfer Appliance (AA21-055A)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-055a): avis sur l’exploitation de l’Accellion FTA en 2020/2021 suivie d’extorsion ; Accellion est l’ancien nom de Kiteworks.

8.  [Kiteworks : Case Study Mandiant](https://www.kiteworks.com/case-study-mandiant/): indication du fabricant concernant sa collaboration avec Mandiant.

9.  [TechCrunch : Kiteworks urges customers to shut down their servers amid 'imminent' threat of cyberattack](https://techcrunch.com/2026/09/25/kiteworks-urges-customers-to-shut-down-their-servers-amid-imminent-threat-of-cyberattack/): déclaration du CISO, heure d’envoi de l’avertissement, réactions du FBI et de la CISA (ajout), conséquences chez un client, nombre de systèmes accessibles depuis Internet.

10.  [Kiteworks : Precautionary Shutdown Advisory (communiqué de presse)](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): communication officielle du 25.09.2026 avec des informations sur les instances hébergées, la version 9.5.1 et les filiales non concernées ; complétée par l’indication du 27.09.2026 selon laquelle la recommandation d’arrêt est levée.

11.  [BleepingComputer : Kiteworks urges 6-hour server shutdown over potential zero-day attacks](https://www.bleepingcomputer.com/news/security/kiteworks-urges-6-hour-server-shutdown-over-potential-zero-day-attacks/): fenêtre temporelle par région et contexte des précédentes attaques contre des produits d’échange de fichiers.

12.  [Computer Weekly : Expecting cyber attack, Kiteworks tells users to turn off servers](https://www.computerweekly.com/news/366651301/Expecting-cyber-attack-Kiteworks-tells-users-to-turn-off-servers): évaluation de watchTowr concernant l’inhabituelle recommandation d’arrêt.

13.  [BornCity : Kiteworks : plus de 1 000 organisations doivent arrêter leurs serveurs](https://borncity.com/news/kiteworks-mehr-als-1-000-organisationen-sollen-server-abschalten/): nombre d’organisations averties et secteurs dans l’espace germanophone.

14.  [The Record : Kiteworks urges customers to stop using platform after warning from federal intelligence agencies](https://therecord.media/kiteworks-urges-customers-to-stop-using-systems-incident): contexte des attaques d’Accellion par Clop en 2020/2021 et citation de watchTowr.

15.  [Cyber Daily : Kiteworks warns customers to enact a „precautionary shutdown“ in wake of attack intelligence](https://www.cyberdaily.au/security/14241-kiteworks-warns-customers-to-enact-a-precautionary-shutdown-in-wake-of-attack-intelligence): article du 28.09.2026 sur la levée de la recommandation et le fonctionnement des instances hébergées.
