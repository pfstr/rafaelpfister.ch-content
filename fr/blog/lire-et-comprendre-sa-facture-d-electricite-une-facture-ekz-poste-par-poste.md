---
title: "Lire et comprendre sa facture d’électricité : une facture EKZ décryptée poste par poste"
navTitle: "Comprendre sa facture d’électricité"
description: "Énergie, utilisation du réseau, mesure, redevances : ce qui figure réellement sur une facture d’électricité suisse, qui fixe les différents prix et quels postes peuvent être modifiés, avec un exemple de facture interactive sur le modèle d’EKZ."
date: "2026-08-20"
kategorie: "Électricité et énergie"
timeToRead: "9 min de lecture"
themen:
  - stromtarife-leg
hauptthema: "stromtarife-leg"
protokolle:
  - "strom"
related:
  - lokale-elektrizitaetsgemeinschaft-leg-erklaert
  - lohnt-sich-leg-beitritt
  - leg-preisrechner
translationId: "article-76c220e720fdffbe"
slug: "lire-et-comprendre-sa-facture-d-electricite-une-facture-ekz-poste-par-poste"
aiPrompt: "Ich füge dir gleich die Positionen meiner Schweizer Stromrechnung ein. Erkläre mir jede Position einzeln: was sie bedeutet, wer den Preis festlegt (Energielieferant, Netzbetreiber oder Bund/Gemeinde) und ob ich sie beeinflussen kann. Rechne zum Schluss aus, wie sich meine Gesamtkosten pro kWh zusammensetzen, und nenne die zwei grössten Hebel zum Sparen. Meine Rechnung:"
translationOf: stromrechnung-verstehen-ekz
translationSourceHash: d81a9bfcf0e980271b4b1f54234b918f4658a44fa002a3f7572dfd80df8ba9b1
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T09:53:59.983Z
translationReview: required
url: https://rafaelpfister.ch/fr/blog/lire-et-comprendre-sa-facture-d-electricite-une-facture-ekz-poste-par-poste
---

# Lire et comprendre sa facture d’électricité : une facture EKZ décryptée poste par poste

La facture d’électricité fait partie de ces documents que l’on paie sans les lire. Pourtant, elle est structurée de manière transparente : chaque poste a un objectif clair, un émetteur clairement identifié et une réponse précise à la question de savoir s’il est possible de le modifier. Une fois les quatre blocs compris, vous saurez lire n’importe quelle facture d’électricité suisse, car leur structure est imposée par la loi et identique chez tous les gestionnaires de réseau.

Cet article examine poste par poste une facture d’EKZ (Elektrizitätswerke des Kantons Zürich), notre gestionnaire de réseau. L’exemple de facture interactif ci-dessous suit la structure de notre propre facture trimestrielle ; les chiffres correspondent à un ménage type consommant 1'800 kWh par trimestre, calculés avec les tarifs EKZ réels de 2026.

## Qui participe réellement à la facture

Une facture d’électricité comporte trois émetteurs, même si un seul l’envoie :

1. **Le fournisseur d’énergie** vend les kilowattheures. Dans l’approvisionnement de base, il s’agit du fournisseur local ; les prix sont contrôlés par le Surveillant des prix ou par l’ElCom. C’est le seul bloc où le produit peut être choisi.
2. **Le gestionnaire de réseau** transporte l’électricité. Le réseau est un monopole réglementé : il ne peut pas être changé, et l’ElCom contrôle les tarifs. Il existe toutefois ici des tarifs à choix et, depuis 2026, la réduction LEG.
3. **La Confédération, le canton et la commune** ajoutent des redevances : supplément réseau, réserve d’électricité, taxes communales. Ni le fournisseur ni le gestionnaire de réseau ne peuvent y changer quoi que ce soit.

Avec cette grille de lecture, chaque poste ci-dessous peut être classé. Passez la souris sur les lignes de l’exemple de facture (ou touchez-les) pour afficher l’explication :

<div class="sr-embed">
<div class="sr-grid">
<div class="sr-paper" role="group" aria-label="Interaktive Beispielrechnung">
<div class="sr-head">
<div class="sr-brand">Musterwerk AG</div>
<div class="sr-meta">Votre facture du 01.03.2026 au 31.05.2026<br>Ménage type, maison individuelle, tarif réseau EKZ Netz 400F, 1'800 kWh</div>
</div>
<div class="sr-block">
<div class="sr-blocktitle">Fourniture d’énergie</div>
<div class="sr-row" tabindex="0" data-sr="energie-winter"><span class="sr-label">Tarif énergétique janv.–mars</span><span class="sr-calc">600 kWh × 13.30 Rp.</span><span class="sr-amount">79.80</span></div>
<div class="sr-row" tabindex="0" data-sr="energie-sommer"><span class="sr-label">Tarif énergétique avr.–juin</span><span class="sr-calc">1'200 kWh × 9.00 Rp.</span><span class="sr-amount">108.00</span></div>
<div class="sr-row" tabindex="0" data-sr="grundtarif"><span class="sr-label">Tarif de base</span><span class="sr-calc">3 mois × CHF 3.00</span><span class="sr-amount">9.00</span></div>
</div>
<div class="sr-block">
<div class="sr-blocktitle">Utilisation du réseau</div>
<div class="sr-row" tabindex="0" data-sr="netz"><span class="sr-label">Réseau 400F</span><span class="sr-calc">1'800 kWh × 7.50 Rp.</span><span class="sr-amount">135.00</span></div>
<div class="sr-row" tabindex="0" data-sr="sdl"><span class="sr-label">Services-système (SDL)</span><span class="sr-calc">1'800 kWh × 0.27 Rp.</span><span class="sr-amount">4.86</span></div>
</div>
<div class="sr-block">
<div class="sr-blocktitle">Mesure</div>
<div class="sr-row" tabindex="0" data-sr="messung"><span class="sr-label">Tarif de mesure</span><span class="sr-calc">3 mois × CHF 5.00</span><span class="sr-amount">15.00</span></div>
</div>
<div class="sr-block">
<div class="sr-blocktitle">Suppléments et redevances</div>
<div class="sr-row" tabindex="0" data-sr="bundesabgaben"><span class="sr-label">Redevances fédérales</span><span class="sr-calc">1'800 kWh × 2.30 Rp.</span><span class="sr-amount">41.40</span></div>
<div class="sr-row" tabindex="0" data-sr="stromreserve"><span class="sr-label">Réserve d’électricité</span><span class="sr-calc">1'800 kWh × 0.41 Rp.</span><span class="sr-amount">7.38</span></div>
<div class="sr-row" tabindex="0" data-sr="solidarisiert"><span class="sr-label">Coûts mutualisés</span><span class="sr-calc">1'800 kWh × 0.05 Rp.</span><span class="sr-amount">0.90</span></div>
<div class="sr-row" tabindex="0" data-sr="effizienz"><span class="sr-label">Promotion de l’efficacité énergétique</span><span class="sr-calc">1'800 kWh × 0.16 Rp.</span><span class="sr-amount">2.88</span></div>
</div>
<div class="sr-block sr-sums">
<div class="sr-row sr-net" tabindex="0" data-sr="netto"><span class="sr-label">Montant net (hors TVA)</span><span class="sr-calc"></span><span class="sr-amount">404.22</span></div>
<div class="sr-row" tabindex="0" data-sr="mwst"><span class="sr-label">TVA de 8.1 %</span><span class="sr-calc"></span><span class="sr-amount">32.74</span></div>
<div class="sr-row sr-total" tabindex="0" data-sr="total"><span class="sr-label">Montant de la facture</span><span class="sr-calc"></span><span class="sr-amount">CHF 436.95</span></div>
</div>
</div>
<aside class="sr-panel" aria-live="polite">
<div class="sr-panel-inner" id="sr-panel-target">
<p class="sr-panel-hint">Survolez un poste ou touchez-le pour voir ce qu’il recouvre.</p>
</div>
</aside>
</div>
<div hidden id="sr-explanations">
<div data-exp="energie-winter"><strong>Énergie, trimestre d’hiver</strong><p>L’électricité proprement dite. Depuis 2026, EKZ facture un tarif unique par trimestre au lieu des tarifs heures pleines et heures creuses : 13.30 Rp./kWh durant le semestre d’hiver (janvier à mars, octobre à décembre). Émetteur : fournisseur d’énergie. C’est le seul bloc dans lequel un autre produit peut être choisi.</p></div>
<div data-exp="energie-sommer"><strong>Énergie, trimestre d’été</strong><p>La même énergie, à un autre prix : 9.00 Rp./kWh d’avril à septembre. L’été est moins cher, car l’hydroélectricité et l’électricité solaire sont alors abondantes. Les prix saisonniers rendent visible ce qui s’applique depuis longtemps sur le marché de l’électricité : l’électricité a une valeur différente selon la saison.</p></div>
<div data-exp="grundtarif"><strong>Tarif de base de l’énergie</strong><p>Montant mensuel fixe pour la fourniture d’énergie (CHF 3.00 par mois), indépendant de la consommation. Il couvre la facturation et la distribution.</p></div>
<div data-exp="netz"><strong>Utilisation du réseau, prix de l’énergie</strong><p>Le transport : construction, exploitation et entretien du réseau électrique. Il s’agit d’un monopole réglementé, et il n’est pas possible de changer de gestionnaire de réseau. Le prix dépend du produit réseau choisi : tarif standard 400ST 7.95 Rp./kWh, 400F avec pilotage favorable au réseau 7.50 Rp./kWh, tarif pour pompe à chaleur 400WP 6.45 Rp./kWh (tous hors TVA). C’est précisément sur ce poste que s’applique la réduction LEG de 20 ou 40 pour cent.</p></div>
<div data-exp="sdl"><strong>Services-système</strong><p>La contribution versée à Swissgrid pour la stabilité du réseau de transport : maintien de la fréquence, puissance de réglage, énergie réactive. 0.27 Rp./kWh, un montant similaire chez tous les gestionnaires de réseau.</p></div>
<div data-exp="messung"><strong>Mesure</strong><p>Exploitation du compteur et mise à disposition des données de mesure, indiquées séparément depuis 2026 (auparavant incluses dans l’utilisation du réseau). CHF 5.00 par mois. Le compteur intelligent payé ici est d’ailleurs la condition technique requise pour participer à une LEG.</p></div>
<div data-exp="bundesabgaben"><strong>Redevances fédérales (supplément réseau)</strong><p>Le supplément réseau légal selon l’art. 35 de la loi sur l’énergie : 2.30 Rp./kWh pour encourager les énergies renouvelables et l’assainissement écologique de la force hydraulique. Fixé par la Confédération, il est identique pour chaque consommateur final.</p></div>
<div data-exp="stromreserve"><strong>Réserve d’électricité</strong><p>Tarif Swissgrid destiné à financer la réserve hivernale : réserve hydroélectrique, centrales électriques de réserve, groupes électrogènes de secours. Une conséquence de la crise énergétique de 2022. 0.41 Rp./kWh.</p></div>
<div data-exp="solidarisiert"><strong>Coûts mutualisés</strong><p>Coûts répartis dans toute la Suisse pour les renforcements du réseau (par exemple pour le raccordement d’installations solaires) et les mesures de soutien. Le poste le plus faible : 0.05 Rp./kWh.</p></div>
<div data-exp="effizienz"><strong>Promotion de l’efficacité énergétique</strong><p>Redevance cantonale ou communale pour le conseil énergétique et les programmes de promotion, 0.16 Rp./kWh. Selon la commune, des redevances de concession peuvent également figurer ici.</p></div>
<div data-exp="netto"><strong>Montant net</strong><p>Somme de tous les postes avant TVA. Pour ce ménage type : environ 22.5 Rp. par kWh consommé, dont seulement environ 10.4 Rp. correspondent réellement à l’énergie.</p></div>
<div data-exp="mwst"><strong>TVA</strong><p>8.1 pour cent sur le montant net, sur tous les postes, y compris les redevances publiques. Autrement dit : la TVA est également prélevée sur les redevances.</p></div>
<div data-exp="total"><strong>Montant de la facture</strong><p>Le montant final est arrondi à 5 centimes ; il diffère donc de quelques centimes du total exact. EKZ indique séparément la différence d’arrondi.</p></div>
</div>
</div>

<style>
</style>

<script>
</script>

## Les quatre blocs en détail

Pour toutes celles et ceux qui préfèrent lire un texte suivi (et pour les moteurs de recherche), voici les mêmes postes expliqués en détail.

### Fourniture d’énergie : le seul bloc avec un choix de produit

La fourniture d’énergie, c’est l’électricité elle-même. Dans l’approvisionnement de base, auquel appartient la très grande majorité des ménages, le fournisseur local fixe le tarif chaque année et l’ElCom le contrôle. Chez EKZ, le produit standard s’appelle « EKZ Energie Erneuerbar » et coûte en 2026 13.30 Rp./kWh durant le semestre d’hiver, et 9.00 Rp./kWh durant le semestre d’été (hors TVA).

Ce qui a disparu en 2026 est remarquable : les tarifs heures pleines et heures creuses. Au lieu de « cher le jour, bon marché la nuit », un tarif unique s’applique désormais et change chaque trimestre. Le conseil classique de faire tourner le lave-linge la nuit est donc devenu obsolète du point de vue tarifaire ; c’est la saison qui compte, pas l’heure. Pour aller plus loin, il est possible de passer au tarif dynamique à choix, dont le prix suit chaque heure le prix de la bourse.

S’y ajoute un tarif de base fixe de CHF 3.00 par mois.

### Utilisation du réseau : le monopole réglementé

L’utilisation du réseau finance la construction, l’exploitation et l’entretien des lignes, postes de transformation et sous-stations. Il n’est pas possible de changer de gestionnaire de réseau ; en contrepartie, les tarifs sont réglementés et contrôlables par l’ElCom.

Il existe toutefois des possibilités de choix au sein du monopole, qui peuvent en valoir la peine. EKZ propose en 2026 trois produits réseau pour les ménages :

| Produit réseau | Prix de l’énergie (hors TVA) | Condition |
| --- | --- | --- |
| EKZ Netz 400ST (standard) | 7.95 Rp./kWh | aucune |
| EKZ Netz 400F | 7.50 Rp./kWh | EKZ peut piloter les charges flexibles (boiler, pompe à chaleur) de manière favorable au réseau |
| EKZ Netz 400WP | 6.45 Rp./kWh | applications de chauffage avec pilotage |

Autoriser le pilotage de son boiler permet donc d’économiser un peu moins d’un demi-rappen par kilowattheure par rapport au tarif standard. Et depuis 2026, un deuxième levier existe à ce poste : toute personne qui rejoint une communauté électrique locale (LEG) bénéficie d’une réduction légale de 20 ou 40 pour cent sur le prix de l’énergie liée à l’utilisation du réseau pour l’électricité échangée localement. Ce qu’est une LEG est expliqué dans [l’article dédié](/blog/lokale-elektrizitaetsgemeinschaft-leg-erklaert) ; pour savoir si cela en vaut la peine, consultez [l’article suivant](/blog/lohnt-sich-leg-beitritt).

Le poste « services-système » (0.27 Rp./kWh) est versé à Swissgrid pour la stabilité de l’ensemble du système : maintien de la fréquence, énergie de réglage, capacité de démarrage autonome.

### Mesure : désormais visible

Depuis 2026, EKZ indique les coûts de mesure séparément : CHF 5.00 par mois pour le compteur, la transmission des données et la mise à disposition des valeurs mesurées. Auparavant, ils étaient inclus de manière invisible dans l’utilisation du réseau. Le compteur intelligent payé ici mesure au quart d’heure près et constitue la base technique des nouveautés actuelles sur le marché de l’électricité : tarifs dynamiques, facturation LEG, déplacement de charge.

### Suppléments et redevances : le bloc étatique

Quatre postes sur lesquels ni le fournisseur ni le gestionnaire de réseau n’ont d’influence :

- **Redevances fédérales** (2.30 Rp./kWh) : le supplément réseau prévu par la loi sur l’énergie finance la promotion des énergies renouvelables et l’assainissement de la force hydraulique.
- **Réserve d’électricité** (0.41 Rp./kWh) : la prime d’assurance du pays contre les situations de pénurie d’électricité, introduite après l’hiver 2022/23. Elle finance la réserve hydroélectrique et les centrales électriques de réserve.
- **Coûts mutualisés** (0.05 Rp./kWh) : renforcements du réseau répartis dans toute la Suisse, par exemple pour les raccordements d’installations solaires.
- **Promotion de l’efficacité énergétique** (0.16 Rp./kWh) : programmes cantonaux et communaux de promotion ainsi que conseil énergétique. Selon le lieu de résidence, des redevances communales de concession s’y ajoutent.

Tout en bas figure enfin la TVA : 8.1 pour cent sur tout, y compris les redevances.

## Ce qui reste par kilowattheure

Si l’on ramène l’exemple de facture à un seul kilowattheure (trimestre d’été, tarif 400F), on obtient approximativement la répartition suivante : 9.0 Rp. pour l’énergie, 7.8 Rp. pour le réseau et les services-système, 2.9 Rp. de redevances, auxquels s’ajoutent au prorata les tarifs de base et de mesure ainsi que la TVA. Résultat : sur environ 21 à 22 centimes par kWh, l’électricité proprement dite ne représente même pas la moitié. Discuter des prix de l’électricité, c’est pour moitié discuter du réseau et de l’État.

C’est précisément pourquoi il vaut la peine d’examiner les postes : le principal levier individuel d’un ménage reste sa consommation, puis viennent le choix du produit réseau, le produit énergétique et désormais la LEG. Pour cette dernière, nous avons créé un [calculateur de prix](/tools/leg-rechner), qui prolonge le calcul présenté dans cet article.

*Tous les tarifs : EKZ 2026, hors TVA, source : recueil tarifaire EKZ 2026. Les autres zones de réseau appliquent d’autres prix, mais présentent la même structure de facture.*
