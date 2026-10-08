---
title: "Kiteworks : le fabricant recommande un arrêt le 26 septembre – ce que l’on sait jusqu’à présent"
navTitle: "Arrêt de Kiteworks"
description: "Kiteworks a demandé à ses clients d’arrêter tous les systèmes le samedi 26.09.2026, de 04:00 à 10:00. Rapport final : pendant l’arrêt, le fabricant a identifié et corrigé une faille critique sans CVE ; le 30.09, 125 avis ont suivi, dont CVE-2026-54154 (CVSS 10.0) dans l’Email Protection Gateway. Totemomail n’est pas concerné."
date: "2026-09-25"
kategorie: "Kiteworks / Totemomail"
timeToRead: "12 min de lecture"
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
translationSourceHash: 15d32c4610fafdeff76397f77ace28ebf6f4aab220c10dc03f5dd4713fab9e48
translationModel: gpt-5.6-terra
translatedAt: 2026-10-08T10:51:51.159Z
translationReview: required
url: https://rafaelpfister.ch/fr/blog/kiteworks-le-fabricant-recommande-un-arret-le-26-septembre-ce-que-l-on-sait-jusqu-a-present
---

# Kiteworks : le fabricant recommande un arrêt le 26 septembre – ce que l’on sait jusqu’à présent

Le 25 septembre 2026, Kiteworks a demandé à ses clients par e-mail d’arrêter tous les systèmes Kiteworks le samedi 26 septembre, de 04:00 à 10:00 (heure d’Europe centrale). Selon le courrier du CISO Frank Balonis, le fabricant disposait d’informations transmises par les autorités chargées de l’application de la loi, selon lesquelles une attaque contre des systèmes Kiteworks pourrait être imminente ce week-end-là. Le support client a justifié l’arrêt par la protection contre d’éventuelles attaques zero-day. heise online a confirmé l’authenticité du message par téléphone auprès du support.

<div class="update-hinweis">
<p class="update-hinweis__titel">Rapport final du 7 octobre 2026</p>
<p>Du point de vue du fabricant, l’incident est clos. La recommandation d’arrêt ne s’applique plus depuis le 27 septembre, et aucune attaque contre des systèmes Kiteworks ou des systèmes clients n’est connue à ce jour. Principales conclusions :</p>
<ul>
<li><strong>Faille critique identifiée pendant l’arrêt :</strong> Selon le communiqué de presse du 28 septembre, Kiteworks a découvert, lors de l’analyse menée avec les autorités fédérales, une vulnérabilité critique jusqu’alors inconnue dans une fonction activée chez moins de 1 % des clients. Le fabricant a développé et déployé un correctif pendant la fenêtre d’arrêt, et a en outre activé une couche de protection dans tous les environnements. La fonction concernée n’a pas été publiée ; elle ne possède à ce jour aucun numéro CVE.</li>
<li><strong>125 avis le 30 septembre :</strong> Deux jours plus tard, Kiteworks a publié sur GitHub 125 avis de sécurité pour Kiteworks Core (66), Email Protection Gateway (28), Secure Data Forms (28) et MFT Server (3) ; 12 sont critiques et 49 élevés. Tous sont corrigés dans les versions jusqu’à 9.5.1 incluse ; la plupart ont été signalés via le programme de bug bounty sur YesWeHack. Selon l’état actuel des connaissances, ils ne sont pas liés à la faille découverte pendant la fenêtre d’arrêt.</li>
<li><strong>CVE-2026-54154 (CVSS 10.0) :</strong> La faille la plus grave concerne l’Email Protection Gateway avant la version 9.4.1. Un attaquant non authentifié peut exécuter du code avec les droits root via des points de terminaison accessibles publiquement. 10 des 12 avis critiques concernent l’Email Protection Gateway.</li>
<li><strong>Aucune exploitation connue :</strong> Aucune des failles ne fait l’objet de rapports d’attaque ; au 7 octobre, le catalogue CISA KEV ne contient aucune entrée Kiteworks de 2026. Selon BleepingComputer, Shadowserver recense près de 400 instances Kiteworks accessibles depuis Internet.</li>
<li><strong>Totemomail :</strong> N’apparaît dans aucun des avis et, selon le fabricant, n’était pas concerné par l’arrêt.</li>
</ul>
<p><strong>Mesures à prendre :</strong> Les personnes qui exploitent elles-mêmes Kiteworks devraient mettre tous les composants à jour vers la version 9.5.1 ; l’Email Protection Gateway est prioritaire. Les détails figurent dans la section <a href="#abschlussbericht">Rapport final</a>.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Mise à jour du 28 septembre 2026 : Kiteworks lève la recommandation d’arrêt</p>
<p>Kiteworks a complété son communiqué de presse par une indication : depuis le 27 septembre, la recommandation d’arrêt ne s’applique plus à aucun client.</p>
<blockquote lang="en">
<p>As of September 27th, the shutdown recommendation is now lifted for all customers. If you have not already restarted, you may bring your Kiteworks system back online. Customers with self-hosted Advanced Forms should contact Customer Support for assistance. All systems Kiteworks hosts on customers’ behalf have been brought back up and are operating normally.</p>
</blockquote>
<p>Les personnes qui n’ont pas encore redémarré leurs systèmes peuvent désormais le faire. Celles qui exploitent elles-mêmes Advanced Forms doivent contacter le support Kiteworks avant le redémarrage. Les instances hébergées par Kiteworks sont de nouveau opérationnelles. Il n’existe toujours aucun numéro CVE, aucune nouvelle version au-delà de 9.5.1, aucun indicateur de compromission et aucune indication permettant de savoir si une attaque a été tentée ou ce qui a motivé l’alerte. La page des mises à jour de sécurité et les avis GitHub restent inchangés.</p>
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

Toutes les heures sont exprimées en heure d’été d’Europe centrale (CEST). Lorsqu’aucune heure n’est indiquée, aucune information horaire fiable n’est disponible.

<ol class="timeline">
<li class="timeline__item">
<p class="timeline__zeit">Ven. 25 septembre</p>
<p class="timeline__titel">Avis aux clients</p>
<p>Le CISO Frank Balonis informe les clients par e-mail d’informations des autorités chargées de l’application de la loi concernant une possible attaque ce week-end-là et recommande un arrêt de six heures. Selon l’avis, toutes les vulnérabilités connues sont corrigées dans la version 9.5.1.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Ven. 25 septembre</p>
<p class="timeline__titel">Premiers articles de presse</p>
<p>heise online rapporte que le support Kiteworks confirme l’authenticité du message et justifie l’arrêt par la protection contre d’éventuelles attaques zero-day. TechCrunch, BleepingComputer, Computer Weekly et d’autres suivent peu après.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Ven. 25 septembre, 17:41</p>
<p class="timeline__titel">Le BKA ne s’exprime pas</p>
<p>heise ajoute : le BKA refuse de faire une déclaration pour des raisons liées à la stratégie d’enquête. Le BSI ne répond pas et le FBI refuse de commenter auprès de TechCrunch.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Ven. 25 septembre</p>
<p class="timeline__titel">Déclaration et communiqué de presse</p>
<p>Kiteworks qualifie l’arrêt de mesure de précaution, sans compromission connue. Le communiqué de presse cite les « federal intelligence authorities » comme source et énumère les filiales non concernées, dont totemo. Le fabricant arrête lui-même les instances qu’il héberge.</p>
</li>
<li class="timeline__item timeline__item--fenster">
<p class="timeline__zeit">Sam. 26 septembre, 04:00 à 10:00</p>
<p class="timeline__titel">Fenêtre d’arrêt</p>
<p>La fenêtre est simultanée dans le monde entier : de 02:00 à 08:00 UTC, de 12:00 à 18:00 à Sydney et de vendredi 22:00 à samedi 04:00 à New York.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Sam. 26 septembre, 10:00</p>
<p class="timeline__titel">Fin de la fenêtre</p>
<p>La fenêtre indiquée dans l’e-mail aux clients prend fin. La levée formelle de la recommandation suit le 27 septembre.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Dim. 27 septembre</p>
<p class="timeline__titel">Recommandation levée</p>
<p>Kiteworks complète le communiqué de presse : la recommandation d’arrêt est levée pour tous les clients, les systèmes peuvent de nouveau fonctionner. Les instances hébergées sont de nouveau en service. Les clients exploitant eux-mêmes Advanced Forms doivent s’adresser au support.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Lun. 28 septembre</p>
<p class="timeline__titel">Faille critique identifiée et corrigée</p>
<p>Dans un autre communiqué de presse, Kiteworks indique qu’une vulnérabilité critique jusqu’alors inconnue a été identifiée lors du travail avec les autorités fédérales pendant l’arrêt. Elle concerne une fonction activée chez moins de 1 % des clients. Le correctif et une couche de protection supplémentaire sont déployés ; aucun indice de compromission n’existe. Le fabricant ne précise ni la fonction ni le numéro CVE.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Mer. 30 septembre, à partir de 18:38</p>
<p class="timeline__titel">125 avis de sécurité sur GitHub</p>
<p>Kiteworks publie 125 avis pour Core, Email Protection Gateway, Secure Data Forms et MFT Server, tous corrigés jusqu’à la version 9.5.1. La faille la plus grave est CVE-2026-54154 dans l’Email Protection Gateway avant 9.4.1 (CVSS 10.0, exécution de code avec les droits root sans authentification).</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Jeu. 1er octobre</p>
<p class="timeline__titel">Articles de presse et avis MS-ISAC</p>
<p>BleepingComputer, SecurityOnline et d’autres couvrent les avis ; le MS-ISAC (Center for Internet Security) publie son propre avis sur CVE-2026-54154. Aucune exploitation n’est connue.</p>
</li>
<li class="timeline__item timeline__item--offen">
<p class="timeline__zeit">État au mer. 7 octobre</p>
<p class="timeline__titel">Conclusion</p>
<p>Aucun rapport d’attaque réussie ou tentée, aucune entrée Kiteworks dans le catalogue CISA KEV. Restent inconnus la fonction concernée, un numéro CVE pour la faille découverte pendant la fenêtre d’arrêt et le contexte de l’alerte des autorités.</p>
</li>
</ol>

## Ce que l’on sait

La recommandation est mondiale ; l’e-mail mentionne la fenêtre pour tous les fuseaux horaires, d’AEST à PDT. Kiteworks recommande d’arrêter les systèmes avant même le début de la fenêtre, y compris lorsqu’ils ne sont pas accessibles depuis Internet.

Jusqu’au 28 septembre, presque tout le reste restait inconnu : il n’y avait aucun avis de sécurité public, aucun numéro CVE, aucun correctif et aucune indication sur les produits ou versions concernés. Le communiqué de presse cite les « federal intelligence authorities » comme source, vraisemblablement des autorités fédérales américaines ; lesquelles restent inconnues à ce jour. Dans les avis GitHub de Kiteworks, la dernière entrée jusque-là datait du 27 mai 2026 ; les avis du 30 septembre sont résumés dans la section [Rapport final](#abschlussbericht).

Auprès de TechCrunch, le CISO de Kiteworks Frank Balonis a fait la même déclaration dans les mêmes termes. Le BKA a refusé de s’exprimer auprès de heise pour des raisons liées à la stratégie d’enquête, et le BSI n’a pas répondu. Le FBI n’a pas souhaité s’exprimer auprès de TechCrunch, tandis qu’un porte-parole de la CISA n’a pas souhaité commenter publiquement. Selon TechCrunch, un client du secteur de la santé a immédiatement déconnecté son serveur, avec des perturbations notables de l’activité : les médecins n’ont temporairement pu joindre leurs patients qu’avec retard. Selon un chercheur en sécurité cité par TechCrunch, au moins 1 000 systèmes Kiteworks sont accessibles depuis Internet ; BornCity évoque plus de 1 000 organisations ayant reçu l’alerte.

Le communiqué de presse diffère de l’e-mail aux clients sur un point : il évoque une fenêtre d’arrêt de neuf heures, tandis que l’avis aux clients parle de six heures. Selon le communiqué de presse, la recommandation ne concerne que les installations exploitées par les clients eux-mêmes (on-premises, AWS, Azure). Selon le fabricant, ses filiales Zivver, DRACOON, totemo, ownCloud, WAMNET, Maytech, Bonfy.ai et 123FormBuilder ne sont pas concernées.

## L’avis aux clients

L’e-mail aux clients du 25 septembre contient, outre l’alerte, un calendrier par fuseau horaire et des instructions pour les clusters. En convertissant les heures en UTC, toutes les régions ont la même fenêtre, de 02:00 à 08:00 UTC.

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

Pour les clusters comprenant plusieurs serveurs, Kiteworks impose un ordre précis :

1.  **Activer le mode maintenance** dans System Setup > Maintenance Mode afin qu’aucun utilisateur ne puisse plus accéder au système.

2.  **Créer une sauvegarde :** un snapshot de chaque nœud ou une sauvegarde de la base de données Kiteworks (System Setup > Cluster Configuration > System Configuration). Une seule sauvegarde de base de données est conservée ; chaque nouvelle remplace la précédente.

3.  **Relever les rôles :** dans System Setup > Locations, la colonne Assigned Roles indique quels nœuds ont le rôle Application ; le nœud Application principal est marqué d’une étoile. Noter les nœuds et leurs adresses IP, nécessaires au redémarrage.

4.  **Arrêter dans cet ordre :** d’abord tous les nœuds sans rôle Application, puis les autres nœuds Application, enfin le nœud Application principal. Cela se fait via l’onglet Shut Down du nœud concerné ou via la console de l’hyperviseur (par exemple VMware ou AWS) si l’interface Kiteworks n’est plus accessible.

5.  **Redémarrer dans l’ordre inverse** via l’hyperviseur, car la console d’administration n’est accessible que lorsque suffisamment de nœuds sont en fonctionnement (annexe E de l’Administrator Guide) : d’abord le nœud Application principal, puis les autres nœuds Application, un par un et seulement une fois que le précédent fonctionne entièrement, afin que les serveurs de base de données puissent former un quorum. Ensuite les serveurs de stockage, puis les autres rôles (Repositories Gateway, Search, SFTP, Antivirus), et enfin les serveurs Web.

6.  **Désactiver le mode maintenance** dès que tous les nœuds sont verts dans le Cluster Health Dashboard, sur la page d’état de la console d’administration.

En réponse à une demande, le support Kiteworks a également confirmé qu’aucune filiale de Kiteworks n’est concernée.

## Causes possibles : théories

Cette section a été rédigée avant le 28 septembre ; l’évaluation selon les connaissances actuelles figure dans le [Rapport final](#abschlussbericht). Les explications suivantes sont des hypothèses déduites des éléments connus ; certaines sont aussi discutées dans les commentaires de l’article de heise. Aucune n’est confirmée.

Trois éléments circonscrivent les possibilités. Premièrement, l’alerte mentionne une fenêtre définie plutôt qu’un arrêt sans durée déterminée jusqu’à la publication d’un correctif. Deuxièmement, même les systèmes non accessibles depuis Internet doivent être déconnectés. Troisièmement, la fenêtre a lieu à la même heure dans le monde entier (02:00 à 08:00 UTC), et non durant la nuit locale de chaque région. Une faille classique exploitable via Internet n’expliquerait pas les deux premiers points : il suffit de déconnecter le système d’Internet, et ce jusqu’à ce que le correctif soit disponible.

### 1. Les autorités connaissent une heure prévue

Les autorités chargées de l’application de la loi apprennent parfois à l’avance la date d’une campagne planifiée, par exemple à partir de communications surveillées d’un groupe criminel ou d’une infrastructure saisie. Les exploitations massives de produits de transfert de fichiers se déroulent généralement dans une courte fenêtre coordonnée, souvent les week-ends ou jours fériés, lorsque les effectifs sont réduits. Kiteworks succède à Accellion, dont la File Transfer Appliance a été attaquée de cette manière en 2020 et 2021, attaques alors attribuées au groupe Clop : des données ont été exfiltrées via plusieurs failles, puis les organisations concernées ont fait l’objet d’extorsion.

La fenêtre étroitement définie durant le week-end plaide en faveur de cette hypothèse. L’objection soulevée par plusieurs commentateurs sur heise va toutefois à l’encontre : l’alerte a été envoyée à tous les clients, les attaquants doivent donc en avoir connaissance et peuvent simplement reporter l’attaque. Un report donnerait cependant au fabricant du temps pour créer un correctif.

### 2. Le fabricant ne connaît pas encore lui-même la faille

Il est également possible que Kiteworks ne dispose, en dehors de l’information des autorités, d’aucun détail technique : ni du composant concerné, ni d’un correctif ou d’une modification de configuration à recommander. L’arrêt serait alors la seule mesure efficace sans connaître la faille, et la fin fixe constituerait un compromis plus acceptable pour les clients. Dans les commentaires de heise, l’hypothèse est avancée que le fabricant pourrait laisser certains systèmes en ligne comme leurres pendant la fenêtre afin d’observer l’attaque. Il n’existe aucune preuve en ce sens.

L’absence d’avis comme de mesure de mitigation plaide en faveur de cette hypothèse. En revanche, Kiteworks déclare travailler avec Mandiant et, lors d’une alerte des autorités, des indicateurs sont généralement au moins disponibles.

### 3. Une porte dérobée déjà implantée avec déclenchement horaire

La recommandation d’arrêter également les systèmes internes correspond à un scénario dans lequel l’attaque ne vient pas de l’extérieur, mais a déjà été préparée sur les appliances : par exemple une porte dérobée issue d’une compromission antérieure, qui s’active à une heure précise ou contacte un serveur de contrôle. Un système arrêté ne peut rien exécuter à ce moment-là.

Le fait que l’accessibilité depuis Internet ne joue aucun rôle dans ce scénario plaide en sa faveur. En revanche, dans ce cas, un fabricant recommanderait plutôt une vérification de compromission et une réinstallation qu’un redémarrage après six heures.

### 4. Compromission du côté du fabricant

Une autre voie permettant d’atteindre les systèmes internes consiste en des connexions établies par l’appliance vers le fabricant, par exemple pour les mises à jour, la vérification de licence ou la maintenance à distance. Si un tel canal est compromis, un pare-feu ne protège pas du trafic entrant. Dans ce scénario, l’arrêt donnerait au fabricant une fenêtre pour assainir sa propre infrastructure, changer les clés ou certificats et n’autoriser de nouveau les connexions qu’ensuite.

L’heure uniforme à l’échelle mondiale, compatible avec une opération coordonnée chez le fabricant, plaide en faveur de cette hypothèse. En revanche, le fabricant recommanderait alors plutôt de bloquer les connexions sortantes que d’arrêter entièrement les systèmes.

### 5. Mesure d’accompagnement d’une opération des autorités

Enfin, il est possible que les autorités interviennent contre l’infrastructure des attaquants durant la même période et souhaitent empêcher que ceux-ci ne frappent encore rapidement en réaction. Cela expliquerait la courte fenêtre et le rôle des autorités chargées de l’application de la loi. Le refus du BKA de s’exprimer pour des raisons liées à la stratégie d’enquête suggère des investigations en cours, mais ne prouve pas cette hypothèse.

### Critique de la communication

Le scepticisme prédomine dans les commentaires de heise, et les objections sont objectivement compréhensibles : sans informations sur la faille, il est impossible d’évaluer si une déconnexion d’Internet par pare-feu aurait suffi. Une fenêtre sans correctif annoncé laisse ouverte la question de ce qui s’applique après 10:00. Et une alerte envoyée uniquement par e-mail aux clients ne touche pas tous les opérateurs, par exemple chez les partenaires, prestataires ou après des changements de personnel. Quelle que soit la théorie correcte : les personnes qui exploitent Kiteworks devraient vérifier les journaux après le redémarrage et surveiller les canaux du fabricant jusqu’à la publication d’un avis.

## Rapport final

Au 7 octobre 2026, le fabricant considère l’incident comme clos. Les événements postérieurs à la fenêtre d’arrêt peuvent être séparés en deux volets : la faille identifiée pendant l’arrêt et la publication groupée d’avis deux jours plus tard.

### La faille découverte pendant la fenêtre d’arrêt

Le 28 septembre, Kiteworks a publié un deuxième communiqué de presse. Selon celui-ci, le fabricant a travaillé tout le week-end avec les autorités fédérales ; une vulnérabilité critique jusqu’alors inconnue a alors été découverte, limitée à une fonction activée chez moins de 1 % des clients. Kiteworks a développé et déployé un correctif pendant la fenêtre d’arrêt et a en outre activé une couche de protection dans tous les environnements. La surveillance continue n’aurait révélé aucune activité suspecte, et aucun élément n’indique une compromission de systèmes Kiteworks ou de systèmes clients. Tous les autres produits Kiteworks ne seraient pas concernés.

La fonction concernée, un numéro CVE, les versions contenant le correctif et la question de savoir si les installations exploitées par les clients ont reçu automatiquement le correctif ne sont pas publiés. La levée du 27 septembre ne comportait qu’une exception : les clients exploitant eux-mêmes Advanced Forms devaient contacter le support avant le redémarrage. Kiteworks n’a pas confirmé si cette fonction était celle concernée.

Concernant les théories ci-dessus : le communiqué de presse décrit une faille découverte seulement pendant la fenêtre. Cela correspond à la théorie 2 (le fabricant ne connaissait pas la faille auparavant) en combinaison avec la théorie 1 (les autorités connaissaient une heure prévue). Il n’existe aucune confirmation pour les théories 3 à 5. Ce que les autorités savaient concrètement et si une attaque a été tentée restent inconnus.

### 125 avis de sécurité du 30 septembre

Le 30 septembre à partir de 18:38, Kiteworks a publié d’un seul coup 125 avis de sécurité sur GitHub. Leur répartition est la suivante :

| Produit | Avis | dont critiques |
|---|---|---|
| Kiteworks Core | 66 | 2 |
| Email Protection Gateway (EPG) | 28 | 10 |
| Secure Data Forms (SDF) | 28 | 0 |
| MFT Server | 3 | 0 |
| **Total** | **125** | **12** |

Par niveau de gravité, on compte 12 classifications critiques, 49 élevées, 52 moyennes et 12 faibles. Toutes les failles sont corrigées dans les versions jusqu’à 9.5.1 incluse ; les entrées les plus anciennes concernent la version 9.2.1. Il s’agit donc d’une divulgation a posteriori de correctifs déjà livrés, et non d’une nouvelle version. Les avis citent majoritairement comme signalants des participants au programme de bug bounty sur YesWeHack. Kiteworks n’établit aucun lien avec la faille découverte pendant la fenêtre d’arrêt ; une semaine après la fenêtre, l’état des avis reste conforme à l’affirmation du 25 septembre selon laquelle toutes les failles connues sont corrigées dans la version 9.5.1.

Les avis critiques :

| CVE | Produit | CVSS 3.1 | corrigé à partir de | Impact |
|---|---|---|---|---|
| CVE-2026-54154 | EPG | 10.0 | 9.4.1 | Exécution de code avec les droits root sans authentification |
| CVE-2026-85065 | EPG | 9.8 | 9.5.0 | Prise de contrôle de compte |
| CVE-2026-85066 | EPG | 9.8 | 9.5.0 | Prise de contrôle de compte |
| CVE-2026-102115 | Core | 9.8 | 9.5.0 | Prise de contrôle de compte via la réinitialisation de mot de passe |
| CVE-2026-102149 | EPG | 9.4 | 9.5.1 | Prise de contrôle de compte |
| CVE-2026-102147 | Core | 9.3 | 9.5.1 | Prise de contrôle de compte |
| CVE-2026-102106 | EPG | 9.1 | 9.5.0 | Contournement de fonctions de sécurité |
| CVE-2026-102095, CVE-2026-102102 à 102105 | EPG | 9.1 | 9.5.0 | Accès à des ressources réseau internes (SSRF) |

CVE-2026-54154 est la faille la plus grave : selon l’avis, une combinaison d’erreurs de validation des entrées dans des points de terminaison publiquement accessibles de l’Email Protection Gateway permet à un attaquant non authentifié d’exécuter du code et, grâce à d’autres faiblesses locales, d’obtenir les droits root sur l’appliance. Le MS-ISAC a publié son propre avis à ce sujet le 1er octobre. Pour les administrateurs de messagerie, le Gateway est la partie pertinente de cette publication : il se situe généralement directement dans le flux de messagerie et est accessible depuis Internet.

### Exploitation et diffusion

Aucun rapport d’exploitation ni exploit public n’est disponible pour aucune des failles. Au 7 octobre, le catalogue CISA KEV ne contient que les quatre entrées Accellion FTA de 2021. Selon BleepingComputer, Shadowserver recense près de 400 instances Kiteworks accessibles depuis Internet ; on ignore combien fonctionnent déjà sous 9.5.1. Totemomail n’apparaît dans aucun des avis.

## Après la fenêtre : ce que les opérateurs peuvent faire maintenant

La recommandation d’arrêt est levée et les failles connues sont corrigées dans la version 9.5.1. Pour les installations exploitées par les clients, les étapes suivantes sont pertinentes :

1.  **Vérifier la version :** tous les nœuds et composants (Core, Email Protection Gateway, Secure Data Forms, MFT Server) exécutent-ils la version 9.5.1 ? Un Email Protection Gateway antérieur à 9.4.1 est concerné par CVE-2026-54154 et devrait être mis à jour en priorité.

2.  **Vérifier l’état du cluster :** tous les nœuds doivent être verts dans le Cluster Health Dashboard et le mode maintenance doit être désactivé.

3.  **Analyser les journaux :** examiner les connexions, les actions d’administration et les téléchargements de fichiers inhabituels autour de la fenêtre d’arrêt, en particulier pour les systèmes qui n’ont pas été arrêtés ou l’ont été tardivement.

4.  **Restreindre l’accessibilité :** lorsque cela est possible, bloquer l’accès depuis Internet à l’interface d’administration et n’autoriser que les services nécessaires.

5.  **Advanced Forms :** toute personne exploitant elle-même le module et n’ayant pas encore contacté le support doit clarifier avec Kiteworks si le correctif de la fenêtre d’arrêt est arrivé sur sa propre installation.

6.  **Comparer les avis :** les avis GitHub peuvent être filtrés par produit (préfixe `[Core]`, `[EPG]`, `[SDF]`, `[MFT]`). Pour chaque composant utilisé, vérifier si la version installée est inférieure à la version de correction indiquée.

7.  **Surveiller les canaux :** les avis GitHub, le Newsroom et les e-mails aux clients de Kiteworks, au cas où le fabricant publierait finalement un avis avec numéro CVE pour la faille découverte pendant la fenêtre d’arrêt. Le [CVE Tracker](/cve) de cette page répertorie également les nouveaux CVE pour le Kiteworks Email Protection Gateway et Totemomail ; il est possible de s’y abonner à des alertes par e-mail.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Assistance pour la mise à jour</p>
<p>Si vous avez besoin d’aide pour mettre à jour une passerelle Kiteworks ou Totemomail, notamment pour rediriger le flux de messagerie pendant la fenêtre de maintenance ou analyser les journaux, veuillez utiliser le <a href="https://adeptio.ch/">formulaire de contact sur adeptio.ch</a>.</p>
</div>

## Sources

1.  [heise online : Attaque zero-day imminente : KiteWorks presse ses clients d’arrêter leurs serveurs](https://www.heise.de/news/Bevorstehender-Zero-Day-Angriff-KiteWorks-draengt-Kunden-zur-Serverabschaltung-11466114.html): premier article avec des extraits de l’e-mail aux clients et la fenêtre horaire ; mise à jour du 25.09 à 17:41 avec la réponse du BKA.

2.  [heise online (EN) : Imminent Zero-Day Attack: KiteWorks Urges Customers to Shut Down Servers](https://www.heise.de/en/news/Imminent-Zero-Day-Attack-KiteWorks-Urges-Customers-to-Shut-Down-Servers-11466375.html): version anglaise avec le texte original du CISO.

3.  [Kiteworks : Security Updates](https://www.kiteworks.com/company/security-updates/): ancienne page de mises à jour du fabricant, au 07.10.2026 sans entrée concernant l’alerte ou les avis du 30.09.2026.

4.  [Kiteworks : Security Advisories sur GitHub](https://github.com/kiteworks/security-advisories/security): liste d’avis du fabricant ; jusqu’au 28.09.2026, dernière entrée du 27.05.2026 ; le 30.09.2026, 125 nouveaux avis pour Core, EPG, SDF et MFT. Les chiffres de cet article ont été comptés via l’API GitHub.

5.  [Kiteworks : Newsroom](https://www.kiteworks.com/newsroom/): communications officielles, dont le communiqué de presse sur l’arrêt depuis le 25.09.2026.

6.  [Forum heise : commentaires sur l’article](https://www.heise.de/forum/heise-online/Kommentare/Server-am-Samstagmorgen-herunterfahren-Kiteworks-warnt-Admins-vor-Zero-Day/forum-591001/comment/): discussion des lecteurs avec les objections à la fenêtre fixe et à l’arrêt des systèmes internes, ainsi que la théorie du leurre.

7.  [CISA : Exploitation of Accellion File Transfer Appliance (AA21-055A)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-055a): avis relatif à l’exploitation d’Accellion FTA en 2020/2021 avec extorsion ultérieure ; Accellion est l’ancien nom de Kiteworks.

8.  [Kiteworks : Case Study Mandiant](https://www.kiteworks.com/case-study-mandiant/): indication du fabricant concernant sa collaboration avec Mandiant.

9.  [TechCrunch : Kiteworks urges customers to shut down their servers amid 'imminent' threat of cyberattack](https://techcrunch.com/2026/09/25/kiteworks-urges-customers-to-shut-down-their-servers-amid-imminent-threat-of-cyberattack/): déclaration du CISO, heure d’envoi de l’alerte, réactions du FBI et de la CISA (ajout), conséquences chez un client, nombre de systèmes accessibles depuis Internet.

10.  [Kiteworks : Precautionary Shutdown Advisory (communiqué de presse)](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): communication officielle du 25.09.2026 avec des informations sur les instances hébergées, la version 9.5.1 et les filiales non concernées ; complétée par la note du 27.09.2026 indiquant que la recommandation d’arrêt est levée.

11.  [BleepingComputer : Kiteworks urges 6-hour server shutdown over potential zero-day attacks](https://www.bleepingcomputer.com/news/security/kiteworks-urges-6-hour-server-shutdown-over-potential-zero-day-attacks/): fenêtre horaire par région et contexte des attaques antérieures contre des produits de transfert de fichiers.

12.  [Computer Weekly : Expecting cyber attack, Kiteworks tells users to turn off servers](https://www.computerweekly.com/news/366651301/Expecting-cyber-attack-Kiteworks-tells-users-to-turn-off-servers): évaluation de watchTowr concernant la recommandation d’arrêt inhabituelle.

13.  [BornCity : Kiteworks : plus de 1 000 organisations doivent arrêter leurs serveurs](https://borncity.com/news/kiteworks-mehr-als-1-000-organisationen-sollen-server-abschalten/): nombre d’organisations averties et secteurs dans l’espace germanophone.

14.  [The Record : Kiteworks urges customers to stop using platform after warning from federal intelligence agencies](https://therecord.media/kiteworks-urges-customers-to-stop-using-systems-incident): analyse des attaques de Clop contre Accellion en 2020/2021 et citation de watchTowr.

15.  [Cyber Daily : Kiteworks warns customers to enact a „precautionary shutdown“ in wake of attack intelligence](https://www.cyberdaily.au/security/14241-kiteworks-warns-customers-to-enact-a-precautionary-shutdown-in-wake-of-attack-intelligence): article du 28.09.2026 sur la levée de la recommandation et le fonctionnement des instances hébergées.

16.  [Kiteworks : Kiteworks Restores Systems After Credible Threat (communiqué de presse)](https://www.kiteworks.com/company/press-releases/kiteworks-restores-systems-credible-threat/): communication du 28.09.2026 sur la faille critique identifiée pendant l’arrêt, le correctif et la couche de protection supplémentaire.

17.  [The Hacker News : Kiteworks Fixes Critical Flaw Found During Nine-Hour Precautionary Shutdown](https://thehackernews.com/2026/09/kiteworks-fixes-critical-flaw-found.html): résumé du deuxième communiqué de presse avec des citations du CISO.

18.  [GitHub Advisory GHSA-5xhq-9wq3-rvj6 : CVE-2026-54154](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-5xhq-9wq3-rvj6): informations du fabricant sur l’exécution de code dans l’Email Protection Gateway avant 9.4.1, CVSS 10.0, signalement via YesWeHack.

19.  [BleepingComputer : Kiteworks patches max severity code injection vulnerability](https://www.bleepingcomputer.com/news/security/kiteworks-patches-max-severity-email-protection-gateway-code-injection-vulnerability/): article du 01.10.2026 sur CVE-2026-54154 et le nombre d’instances recensées par Shadowserver.

20.  [MS-ISAC Advisory 2026-107 : A Vulnerability in Kiteworks EPG Could Allow for Arbitrary Code Execution](https://www.cisecurity.org/advisory/a-vulnerability-in-kiteworks-epg-email-security-gateway-could-allow-for-arbitrary-code-execution_2026-107): avis du Center for Internet Security du 01.10.2026 avec des recommandations.

21.  [SecurityOnline : Kiteworks Patches 78 Vulnerabilities, Including Critical Account Takeover Flaw](https://securityonline.info/kiteworks-vulnerabilities/): analyse des failles de prise de contrôle de compte dans Core, dont CVE-2026-102115 ; le décompte diffère de la liste d’avis.

22.  [CISA : Known Exploited Vulnerabilities Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog): au 07.10.2026, uniquement les quatre entrées Accellion FTA de 2021, aucune entrée Kiteworks de 2026.
