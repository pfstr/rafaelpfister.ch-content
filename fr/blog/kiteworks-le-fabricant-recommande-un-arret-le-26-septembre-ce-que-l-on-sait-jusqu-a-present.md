---
title: "Kiteworks : le fabricant recommande un arrêt le 26 septembre – ce que l’on sait jusqu’à présent"
navTitle: "Arrêt de Kiteworks"
description: "Kiteworks demande à ses clients par e-mail d’arrêter tous les systèmes le samedi 26.09.2026, de 04:00 à 10:00. La raison est un avertissement des autorités chargées de l’application de la loi concernant une possible attaque. Totemomail n’est pas concerné."
date: "2026-09-25"
kategorie: "Kiteworks / Totemomail"
timeToRead: "9 min de lecture"
themen:
  - totemomail
produkte:
  - "totemomail"
protokolle:
  - "verschluesselung"
  - "haertung"
  - "smtp"
hauptthema: "totemomail"
slug: "kiteworks-le-fabricant-recommande-un-arret-le-26-septembre-ce-que-l-on-sait-jusqu-a-present"
featured: "2026-09-27"
warnung: true
translationId: "article-38fbaa0e9095957a"
aiPrompt: |
  Du bist mein Exchange- und Mailflow-Assistent. Kiteworks empfiehlt, alle Systeme am 26.09.2026 von 04:00 bis 10:00 Uhr herunterzufahren. Hilf mir zu ermitteln, welche Connectoren, Transportregeln und MX-Einträge in meiner Umgebung Mails über das Gateway leiten, wie ich den Mailflow für die Dauer der Abschaltung umleite oder kontrolliert anhalte und wie ich den ursprünglichen Zustand danach wiederherstelle. Frage zuerst nach meinem Aufbau (Exchange Online, Exchange Server oder Hybrid, Richtung des Mailflows, Position des Gateways).
translationOf: kiteworks-zero-day-abschaltung
url: https://rafaelpfister.ch/fr/blog/kiteworks-le-fabricant-recommande-un-arret-le-26-septembre-ce-que-l-on-sait-jusqu-a-present
translationSourceHash: c7274a068cc60b422ffcdf30dbaef2d90eac1fe72cf3f768b454a71c676aa046
translationModel: gpt-5.6-terra
translatedAt: 2026-09-26T08:44:43.105Z
translationReview: automatic
---

# Kiteworks : le fabricant recommande un arrêt le 26 septembre – ce que l’on sait jusqu’à présent

Le 25 septembre 2026, Kiteworks a demandé à ses clients par e-mail d’arrêter tous les systèmes Kiteworks le samedi 26 septembre, de 04:00 à 10:00 (heure d’Europe centrale). Selon le courrier du CISO Frank Balonis, le fabricant dispose d’informations provenant des autorités chargées de l’application de la loi selon lesquelles une attaque contre des systèmes Kiteworks pourrait avoir lieu ce week-end. Le support client justifie l’arrêt par la protection contre de possibles attaques zero-day. heise online a confirmé l’authenticité du message par téléphone auprès du support.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Assistance d’urgence pour basculer le flux de messagerie</p>
<p>Si vous avez besoin d’aide pour rediriger le flux de messagerie avant l’arrêt, puis le rétablir ensuite, veuillez utiliser le <a href="https://adeptio.ch/">formulaire de contact sur adeptio.ch</a>. Je vous répondrai également à court terme.</p>
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
<p>heise online rapporte que le support de Kiteworks confirme l’authenticité du message et justifie l’arrêt par la protection contre de possibles attaques zero-day. Peu après, TechCrunch, BleepingComputer, Computer Weekly et d’autres suivent.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Ven. 25 septembre, 17:41</p>
<p class="timeline__titel">Le BKA ne s’exprime pas</p>
<p>heise ajoute : le BKA refuse de commenter pour des raisons tactiques liées à l’enquête. Le BSI ne répond pas, et le FBI refuse tout commentaire auprès de TechCrunch.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Ven. 25 septembre</p>
<p class="timeline__titel">Déclaration et communiqué de presse</p>
<p>Kiteworks qualifie l’arrêt de mesure de précaution sans compromission connue. Le communiqué de presse cite les « federal intelligence authorities » comme source et énumère les filiales non concernées, dont totemo. Le fabricant arrête lui-même les instances hébergées par Kiteworks.</p>
</li>
<li class="timeline__item timeline__item--fenster">
<p class="timeline__zeit">Sam. 26 septembre, de 04:00 à 10:00</p>
<p class="timeline__titel">Fenêtre d’arrêt</p>
<p>La fenêtre est simultanée dans le monde entier : de 02:00 à 08:00 UTC, de 12:00 à 18:00 à Sydney, et du vendredi 22:00 au samedi 04:00 à New York.</p>
</li>
<li class="timeline__item timeline__item--offen">
<p class="timeline__zeit">État au sam. 26 septembre</p>
<p class="timeline__titel">Toujours en suspens</p>
<p>Pas d’avis public, pas de numéro CVE, aucune information sur la faille et aucun rapport d’attaque ayant eu lieu.</p>
</li>
</ol>

## Ce que l’on sait

La recommandation s’applique dans le monde entier ; l’e-mail indique la fenêtre pour tous les fuseaux horaires, de l’AEST au PDT. Kiteworks conseille d’arrêter les systèmes avant le début de la fenêtre, y compris s’ils ne sont pas accessibles depuis Internet.

Presque tout le reste demeure incertain : il n’existe aucun avis de sécurité public, aucun numéro CVE, aucun correctif ni aucune indication sur les produits ou versions concernés. Le communiqué de presse cite les « federal intelligence authorities » comme source, vraisemblablement des autorités fédérales américaines ; on ignore lesquelles. Au 26 septembre, aucune entrée n’apparaît dans les Security Updates ni dans les avis GitHub de Kiteworks. Les seuls éléments publics sont la déclaration citée ci-dessus et le communiqué de presse du 25 septembre.

Auprès de TechCrunch, le CISO de Kiteworks Frank Balonis a fait la même déclaration dans les mêmes termes. Le BKA a refusé de commenter auprès de heise pour des raisons tactiques liées à l’enquête, tandis que le BSI n’a pas répondu. Le FBI n’a pas souhaité s’exprimer auprès de TechCrunch et la CISA n’avait pas répondu. Selon TechCrunch, un client du secteur de la santé a immédiatement déconnecté son serveur du réseau, avec des restrictions sensibles sur l’exploitation.

Le communiqué de presse diverge de l’e-mail aux clients sur un point : il évoque une fenêtre d’arrêt de neuf heures, tandis que l’avis aux clients parle de six heures. Selon le communiqué de presse, la recommandation ne concerne que les installations exploitées par les clients eux-mêmes (on-premises, AWS, Azure). Selon le fabricant, les filiales Zivver, DRACOON, totemo, ownCloud, WAMNET, Maytech, Bonfy.ai et 123FormBuilder ne sont pas concernées.

## L’avis aux clients

L’e-mail aux clients du 25 septembre contient, outre l’avertissement, un calendrier par fuseau horaire et des instructions pour les clusters. En convertissant les heures en UTC, on obtient pour toutes les régions la même fenêtre, de 02:00 à 08:00 UTC.

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

Pour les clusters comptant plusieurs serveurs, Kiteworks impose un ordre précis :

1.  **Activer le mode de maintenance** sous System Setup > Maintenance Mode, afin que les utilisateurs ne puissent plus y accéder.

2.  **Créer une sauvegarde :** un snapshot de chaque nœud ou une sauvegarde de la base de données Kiteworks (System Setup > Cluster Configuration > System Configuration). Une seule sauvegarde de base de données est conservée ; chaque nouvelle remplace la précédente.

3.  **Recenser les rôles :** sous System Setup > Locations, la colonne Assigned Roles indique quels nœuds ont le rôle Application ; le nœud Application principal est marqué d’un astérisque. Noter les nœuds et leurs adresses IP, car ils seront nécessaires au redémarrage.

4.  **Arrêter dans cet ordre :** d’abord tous les nœuds sans rôle Application, puis les autres nœuds Application, enfin le nœud Application principal. Cela peut se faire via l’onglet Shut Down du nœud concerné ou par la console de l’hyperviseur (par exemple VMware ou AWS) si l’interface Kiteworks n’est plus accessible.

5.  **Redémarrer dans l’ordre inverse** via l’hyperviseur, car la console d’administration n’est accessible que lorsqu’un nombre suffisant de nœuds est en fonctionnement (annexe E du Administrator Guide) : d’abord le nœud Application principal, puis les autres nœuds Application un par un et seulement lorsque le précédent est entièrement opérationnel, afin que les serveurs de bases de données puissent former un quorum. Ensuite, les serveurs de stockage, puis les autres rôles (Repositories Gateway, Search, SFTP, Antivirus), et enfin les serveurs Web.

6.  **Désactiver le mode de maintenance** dès que tous les nœuds sont verts dans le Cluster Health Dashboard, sur la page d’état de la console d’administration.

À la demande, le support de Kiteworks a en outre confirmé qu’aucune filiale de Kiteworks n’est concernée.

## Causes possibles : théories

Tant que Kiteworks ne publie pas de détails, la cause reste inconnue. Les explications suivantes sont des hypothèses déduites des éléments connus ; certaines sont également discutées dans les commentaires de l’article de heise. Aucune n’est confirmée.

Trois éléments restreignent les possibilités. Premièrement, l’avertissement indique une fenêtre fixe plutôt qu’un arrêt pour une durée indéterminée jusqu’à la publication d’un correctif. Deuxièmement, même les systèmes non accessibles depuis Internet doivent être déconnectés. Troisièmement, la fenêtre a lieu à la même heure dans le monde entier (de 02:00 à 08:00 UTC), plutôt que durant chaque nuit locale. Une faille classique exploitable via Internet n’expliquerait pas les deux premiers points : il suffirait de déconnecter le système d’Internet, et ce jusqu’à la disponibilité du correctif.

### 1. Les autorités connaissent une heure prévue

Les autorités chargées de l’application de la loi apprennent parfois à l’avance l’heure d’une campagne planifiée, par exemple grâce à des communications surveillées d’un groupe d’auteurs ou à une infrastructure saisie. Les exploitations massives de produits d’échange de fichiers se déroulent généralement dans une courte fenêtre coordonnée, souvent durant les week-ends ou les jours fériés, lorsque moins de personnel est en service. Kiteworks est le successeur d’Accellion, dont la File Transfer Appliance a été attaquée précisément de cette manière en 2020 et 2021 : des données ont été exfiltrées via plusieurs failles, puis les organisations concernées ont été victimes d’extorsion.

La fenêtre étroitement définie durant le week-end plaide en faveur de cette hypothèse. L’objection formulée par plusieurs commentateurs sur heise va à l’encontre : l’avertissement a été envoyé à tous les clients, les attaquants devraient donc en avoir connaissance et peuvent simplement reporter l’attaque. Un report donnerait toutefois au fabricant du temps pour préparer un correctif.

### 2. Le fabricant ne connaît pas encore lui-même la faille

Il est également possible que Kiteworks ne dispose d’aucun détail technique en dehors de l’information des autorités, et qu’il ne connaisse donc ni le composant concerné ni un correctif ou une modification de configuration à recommander. Dans ce cas, l’arrêt est la seule mesure efficace sans connaissance de la faille, et la fin fixée à l’avance constitue un compromis que les clients accepteront plus facilement. Dans les commentaires de heise, l’hypothèse est émise que le fabricant pourrait laisser certains systèmes en ligne comme leurres pendant la fenêtre afin d’observer l’attaque. Il n’existe aucune preuve en ce sens.

L’absence d’avis ou de mesure de mitigation plaide en faveur de cette hypothèse. À l’inverse, Kiteworks indique collaborer avec Mandiant et, en cas d’avertissement des autorités, des indicateurs sont normalement disponibles au moins.

### 3. Une porte dérobée déjà placée, avec déclencheur temporel

La recommandation d’arrêter également les systèmes internes correspond à un scénario où l’attaque ne vient pas de l’extérieur, mais est déjà préparée sur les appliances : par exemple une porte dérobée issue d’une compromission antérieure, qui s’active à une heure définie ou prend contact avec un serveur de commande. Un système éteint ne peut rien exécuter à ce moment-là.

Le fait que l’accessibilité depuis Internet ne joue aucun rôle dans ce scénario plaide en sa faveur. À l’inverse, dans ce cas, un fabricant recommanderait plus probablement une vérification de compromission et une réinstallation qu’un redémarrage après six heures.

### 4. Compromission du côté du fabricant

Une autre voie permettant d’atteindre des systèmes internes consiste en des connexions établies par l’appliance vers le fabricant, par exemple pour les mises à jour, la vérification de licence ou la maintenance à distance. Si un tel canal est compromis, un pare-feu ne protège pas contre le trafic entrant. Dans ce scénario, l’arrêt donnerait au fabricant une fenêtre pour assainir sa propre infrastructure, remplacer des clés ou des certificats et ne réautoriser les connexions qu’ensuite.

L’heure uniforme dans le monde entier, qui correspondrait à une action coordonnée du côté du fabricant, plaide en faveur de cette hypothèse. À l’inverse, le fabricant recommanderait alors plus probablement de bloquer les connexions sortantes plutôt que d’arrêter entièrement les systèmes.

### 5. Mesure d’accompagnement d’une action des autorités

Enfin, il est envisageable que les autorités interviennent contre l’infrastructure des attaquants durant la même période et souhaitent empêcher que ceux-ci frappent rapidement en réaction. Cela expliquerait la courte fenêtre et le rôle des autorités chargées de l’application de la loi. Le refus du BKA de commenter pour des raisons tactiques liées à l’enquête suggère des investigations en cours, mais ne prouve pas cette variante.

### Critique de la communication

Dans les commentaires de heise, le scepticisme prédomine, et les objections sont factuellement compréhensibles : sans indication sur la faille, il est impossible d’évaluer si une séparation d’Internet par pare-feu aurait suffi. Une fenêtre sans correctif annoncé ne précise pas ce qui s’applique après 10:00. Et un avertissement envoyé uniquement par e-mail aux clients n’atteint pas tous les exploitants, par exemple chez les partenaires, prestataires de services ou après des changements de personnel. Quelle que soit la théorie correcte : les personnes qui exploitent Kiteworks devraient vérifier les journaux après le redémarrage et surveiller les canaux du fabricant jusqu’à la publication d’un avis.

## Sources

1.  [heise online : Attaque zero-day imminente : KiteWorks pousse ses clients à arrêter leurs serveurs](https://www.heise.de/news/Bevorstehender-Zero-Day-Angriff-KiteWorks-draengt-Kunden-zur-Serverabschaltung-11466114.html): premier article avec des extraits de l’e-mail aux clients et la fenêtre horaire ; mise à jour du 25.09., 17:41, avec la réponse du BKA.

2.  [heise online (EN) : Imminent Zero-Day Attack: KiteWorks Urges Customers to Shut Down Servers](https://www.heise.de/en/news/Imminent-Zero-Day-Attack-KiteWorks-Urges-Customers-to-Shut-Down-Servers-11466375.html): version anglaise avec le texte original du CISO.

3.  [Kiteworks : Security Updates](https://www.kiteworks.com/company/security-updates/): canal officiel du fabricant, sans entrée concernant l’avertissement au 26.09.2026.

4.  [Kiteworks : Security Advisories sur GitHub](https://github.com/kiteworks/security-advisories/security): liste des avis du fabricant, dernière entrée du 27.05.2026.

5.  [Kiteworks : Newsroom](https://www.kiteworks.com/newsroom/): communications officielles, avec depuis le 25.09.2026 le communiqué de presse concernant l’arrêt.

6.  [Forum heise : commentaires sur l’article](https://www.heise.de/forum/heise-online/Kommentare/Server-am-Samstagmorgen-herunterfahren-Kiteworks-warnt-Admins-vor-Zero-Day/forum-591001/comment/): discussion des lecteurs avec les objections sur la fenêtre fixe et l’arrêt des systèmes internes, ainsi que la théorie du leurre.

7.  [CISA : Exploitation of Accellion File Transfer Appliance (AA21-055A)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-055a): avis sur l’exploitation de l’Accellion FTA en 2020/2021 suivie d’extorsion ; Accellion est l’ancien nom de Kiteworks.

8.  [Kiteworks : Case Study Mandiant](https://www.kiteworks.com/case-study-mandiant/): indication du fabricant concernant la collaboration avec Mandiant.

9.  [TechCrunch : Kiteworks urges customers to shut down their servers amid 'imminent' threat of cyberattack](https://techcrunch.com/2026/09/25/kiteworks-urges-customers-to-shut-down-their-servers-amid-imminent-threat-of-cyberattack/): déclaration du CISO, heure d’envoi de l’avertissement, réactions du FBI et de la CISA, conséquences chez un client.

10.  [Kiteworks : Precautionary Shutdown Advisory (communiqué de presse)](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): communication officielle du 25.09.2026 avec des informations sur les instances hébergées, la version 9.5.1 et les filiales non concernées.

11.  [BleepingComputer : Kiteworks urges 6-hour server shutdown over potential zero-day attacks](https://www.bleepingcomputer.com/news/security/kiteworks-urges-6-hour-server-shutdown-over-potential-zero-day-attacks/): fenêtre horaire par région et mise en contexte des attaques antérieures contre les produits d’échange de fichiers.

12.  [Computer Weekly : Expecting cyber attack, Kiteworks tells users to turn off servers](https://www.computerweekly.com/news/366651301/Expecting-cyber-attack-Kiteworks-tells-users-to-turn-off-servers): évaluation de watchTowr concernant la recommandation d’arrêt inhabituelle.
