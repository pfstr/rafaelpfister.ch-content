---
title: "Planifier des tests de charge e-mail : comparaison des outils pour des rafales de 10'000 e-mails sous Linux et Windows"
navTitle: "Tests de charge e-mail"
description: "Quiconque migre une passerelle ou dimensionne un environnement de messagerie a besoin de chiffres fiables plutôt que d’intuitions. Quels outils génèrent des rafales de plusieurs dizaines de milliers d’e-mails, à quoi ressemble un plan de test propre et comment exploiter les résultats à partir des logs."
date: "2026-08-24"
kategorie: "SMTP et flux de messagerie"
timeToRead: "12 min de lecture"
themen:
  - smtp-mailflow
  - testing
produkte:
  - "uebergreifend"
protokolle:
  - "testing"
  - "smtp"
  - "tcp"
  - "tls"
  - "troubleshooting"
slug: "planifier-des-tests-de-charge-e-mail-comparaison-des-outils-pour-des-rafales-de-10-000-e-mails"
translationId: "article-14a98de0cef45565"
aiPrompt: |
  Du bist mein Assistent für Mail-Lasttests. Hilf mir Schritt für Schritt, einen Lasttest gegen mein eigenes Test-Mailgateway zu planen: Zieldefinition (Durchsatz, Latenz, Queue-Verhalten), Wahl des Lastgenerators (smtp-source, Postal, JMeter oder Skript), Aufbau einer Mail-Senke, Testablauf (Baseline, Burst, Soak) und Auswertung der Logs mit Perzentilen. Frage zuerst nach Plattform, Zielsystem und erwartetem Mailvolumen.
translationOf: mail-lasttest-tools-linux-windows-vergleich
url: https://rafaelpfister.ch/fr/blog/planifier-des-tests-de-charge-e-mail-comparaison-des-outils-pour-des-rafales-de-10-000-e-mails
translationSourceHash: c9b76f3c9887117756e07c71a3dc30d1ee99aeb8a322c50dee994a07df46cb97
translationModel: gpt-5.6-terra
translatedAt: 2026-08-25T04:11:07.482Z
translationReview: automatic
---

# Planifier des tests de charge e-mail : comparaison des outils pour des rafales de 10'000 e-mails sous Linux et Windows

La capacité d’une nouvelle passerelle de messagerie à supporter la charge de pointe d’une nuit de traitement de factures ne se vérifie pas dans une fiche technique, mais lors d’un test. Quiconque remplace une appliance, dimensionne un environnement Exchange ou prévoit l’envoi d’une newsletter via sa propre infrastructure a besoin au préalable de chiffres fiables : combien de messages par seconde le système accepte-t-il, comment la file d’attente se comporte-t-elle sous pression et à partir de quel point les reports commencent-ils ? Cet article compare les générateurs de charge courants sous Linux et Windows et montre comment planifier, réaliser et évaluer un test avec des rafales de plusieurs dizaines de milliers d’e-mails.

La règle la plus importante d’emblée : les tests de charge doivent être effectués exclusivement dans sa propre infrastructure ou dans un environnement de test explicitement autorisé à cet effet. Une rafale vers des systèmes tiers constitue une attaque, et un test avec des adresses d’expéditeur fictives vers des cibles de production génère du backscatter, ce qui mène à des listes de blocage. Une architecture propre se compose d’un générateur de charge, du système à tester et d’un puits contrôlé qui accepte les e-mails à la fin puis les rejette.

## Ce qu’un test de charge e-mail doit mesurer

Avant même de parler d’un outil, il convient de se demander quelle grandeur est réellement intéressante. En pratique, il y en a quatre différentes, et elles requièrent des configurations de test distinctes :

1. **Taux d’acceptation :** Combien de messages par seconde le premier saut accepte-t-il via SMTP ? C’est la mesure de débit classique et la valeur que les générateurs de charge fournissent directement.
2. **Latence de session :** Combien de temps dure une transaction SMTP individuelle, de l’établissement de la connexion jusqu’au `250` après `DATA` ? Sous charge, cette valeur augmente souvent bien avant que le taux d’acceptation ne s’effondre.
3. **Latence de bout en bout :** Combien de temps met un message entre son dépôt et sa livraison au puits, à travers toutes les étapes intermédiaires ? C’est la grandeur que les utilisateurs perçoivent.
4. **Comportement de la file d’attente :** Jusqu’à quelle profondeur la file d’attente croît-elle pendant la rafale, et à quelle vitesse se vide-t-elle ensuite ? Une passerelle qui accepte 50'000 e-mails puis les traite pendant trois heures réussit le test d’acceptation et échoue malgré tout.

Un test qui ne mesure que le taux d’acceptation dit peu de choses d’un environnement à plusieurs niveaux avec passerelle, couche de chiffrement et serveur cible. C’est précisément dans ce type d’environnement que la vue de bout en bout est décisive.

## Outils sous Linux

**smtp-source et smtp-sink** du paquet Postfix sont la référence pour la charge SMTP brute et sont disponibles sur pratiquement tout système où Postfix est installé. `smtp-source` génère des messages avec une taille, un parallélisme et un nombre configurables, tandis que `smtp-sink` est son pendant : un serveur SMTP qui accepte tout puis rejette tout. Une rafale de 10'000 e-mails avec 50 sessions parallèles et des messages de 5 Ko tient en une ligne :

```bash
time smtp-source -s 50 -m 10000 -l 5120 -c -f last@test.example -t senke@test.example gateway.test.example:25
```

L’option `-c` compte en direct les messages envoyés, tandis que `time` fournit la durée totale et donc le débit. Limites importantes : `smtp-source` ne mesure pas les percentiles de latence et les messages synthétiques sont uniformes. Pour répondre à la question « quelle est la capacité maximale d’acceptation du système ? », il reste néanmoins le premier choix, car même sur du matériel modeste, il génère des dizaines de milliers de messages par minute et le générateur ne devient pratiquement jamais le goulot d’étranglement.

**Postal** est l’outil classique de benchmark de serveur de messagerie dédié sous Linux. Il fait varier automatiquement l’expéditeur, le destinataire et la taille des messages, maintient un débit cible sur de longues périodes et écrit des statistiques par minute. Il convient donc mieux que `smtp-source` aux tests d’endurance, c’est-à-dire à une charge continue pendant plusieurs heures. Le `bhm` (Black Hole Mailer) associé assure le rôle de puits. Postal est ancien, mais conçu précisément pour cela et disponible dans les dépôts de paquets de la plupart des distributions.

**swaks** n’est pas un générateur de charge, mais il devrait figurer dans tout plan de test. Il exécute une transaction SMTP individuelle avec un contrôle total sur chaque étape : authentification, STARTTLS, en-têtes arbitraires, pièces jointes. Avant chaque test de charge, un passage avec swaks doit servir de test fonctionnel afin que la rafale n’échoue pas à cause d’un mauvais destinataire ou d’un problème TLS, ce qui rendrait la mesure inutile. Dans une boucle avec `xargs -P`, swaks peut aussi être détourné en petit générateur de charge, mais pour des dizaines de milliers d’e-mails, la surcharge liée aux processus est trop élevée.

**Les scripts maison** en Python (smtplib, aiosmtplib) ou Go sont la solution lorsque le test requiert des mélanges de messages réalistes : tailles différentes, vraies pièces jointes, nombre variable de destinataires par transaction, cas d’erreur ciblés. L’effort est plus important, mais le script mesure exactement ce que l’environnement rencontrera par la suite et peut enregistrer des horodatages par message pour l’analyse de latence.

## Outils sous Windows

**Apache JMeter** est la première recommandation sous Windows. Le SMTP Sampler intégré gère l’authentification, STARTTLS, les pièces jointes et les fichiers EML comme source de messages, et le mécanisme JMeter apporte ce qui manque aux outils Postfix : groupes de threads pour des profils de charge progressifs, percentiles des temps de réponse, taux d’erreur et rapports. Pour les rafales dépassant quelques milliers d’e-mails par minute, la règle habituelle de JMeter s’applique : utiliser l’interface graphique uniquement pour créer le plan de test, et exécuter la mesure elle-même en mode CLI, faute de quoi l’on mesure l’interface.

**PowerShell avec MailKit** est la voie des outils intégrés. Le `Send-MailMessage` autrefois courant est lui-même indiqué par Microsoft comme obsolète, qui recommande une migration ; MailKit peut être chargé via NuGet et parallélisé avec des runspaces depuis PowerShell 7. Cela permet de façon réaliste quelques centaines à quelques milliers d’e-mails par minute, suffisant pour les tests fonctionnels et de régression, mais insuffisant pour mesurer la charge maximale. Avantage : le script fonctionne sans installation supplémentaire sur tout poste de travail d’administration et peut écrire directement les résultats en CSV pour l’évaluation.

**Microsoft Exchange Load Generator (LoadGen)** a été pendant des années l’outil officiel pour soumettre des environnements Exchange à une charge simulant des profils utilisateurs (Outlook, ActiveSync, OWA). Microsoft ne l’a plus maintenu après Exchange 2013 et a retiré le téléchargement. Pour la charge SMTP pure, LoadGen était de toute façon le mauvais outil ; quiconque souhaite aujourd’hui simuler une charge de boîte aux lettres Exchange se retrouve sans outil officiel et a tout intérêt à tester directement le chemin SMTP.

**WSL** mérite son propre point : quiconque travaille sur une machine Windows mais a besoin d’outils Linux installe `smtp-source` et Postal dans une distribution WSL et dispose ainsi de toute la boîte à outils Linux sans machine virtuelle de test distincte. Pour les charges discutées ici, le chemin réseau WSL ne représente pas un goulot d’étranglement pertinent.

## Comparaison

| Outil | Plateforme | Point fort | Limite |
|---|---|---|---|
| smtp-source / smtp-sink | Linux (Postfix) | Charge brute maximale avec un effort minimal, générateur et puits d’un seul tenant | Pas de percentiles de latence, messages uniformes |
| Postal / bhm | Linux | Charge continue à débit cible, messages variés, statistiques par minute | Outillage vieillissant, évaluation à construire soi-même |
| swaks | Linux, Windows (Perl) | Test individuel entièrement contrôlable, idéal comme contrôle fonctionnel avant la rafale | Pas un générateur de charge |
| JMeter (SMTP Sampler) | Windows, Linux (Java) | Profils de charge, percentiles, rapports, sources de messages EML | Surcharge Java, piège de l’interface graphique à haut débit |
| PowerShell + MailKit | Windows | Sans installation supplémentaire sur chaque poste d’administration, sortie CSV | Débit limité, parallélisation à construire soi-même |
| Script maison (Python/Go) | les deux | Mélange de messages réaliste, points de mesure propres | Effort de développement, générateur à valider soi-même |

## Le puits : où envoyer les e-mails

La moitié sous-estimée de la configuration de test est la cible. Trois variantes ont fait leurs preuves :

- **smtp-sink** ou `bhm` comme trou noir : accepte tout, rejette tout, mesure la chaîne de transport pure. `smtp-sink` peut, sur demande, générer artificiellement des retards de réponse et des codes d’erreur, permettant ainsi de tester le comportement du système testé face à une cible lente ou récalcitrante.
- **Postfix avec transport discard** comme puits plus réaliste, lorsque la cible elle-même doit être un serveur SMTP complet avec mise en file d’attente.
- **Quelques véritables boîtes aux lettres de contrôle** en complément du puits, pour vérifier par échantillonnage que les messages arrivent intacts sur le fond, y compris avec une couche de chiffrement ou de signature.

Les outils avec interface Web tels que Mailpit sont destinés au développement et deviennent rapidement eux-mêmes le goulot d’étranglement avec des dizaines de milliers d’e-mails. Ils ne conviennent pas comme puits pour un test de charge ; la mesure évaluerait l’outil d’analyse au lieu du système testé.

## Planifier le test

Un test fiable se déroule en trois étapes, chacune avec sa propre question :

1. **Référence :** Une charge modérée et connue (environ 10 % de la pointe attendue) pendant quelques minutes. Elle fournit les valeurs de référence pour la latence et l’utilisation des ressources, et révèle les erreurs de configuration avant qu’elles ne se perdent dans la mesure de rafale.
2. **Rafale :** La mesure de charge de pointe proprement dite, par exemple 10'000 à 50'000 e-mails aussi rapidement que possible ou à un débit cible défini. Plusieurs passages avec un parallélisme croissant montrent où le taux d’acceptation plafonne et la latence bascule.
3. **Endurance :** La charge quotidienne attendue pendant plusieurs heures. Ce n’est qu’ici que se révèlent les fuites de mémoire, les partitions de spool saturées, la rotation des logs sous charge et les limites de connexion qu’une courte rafale n’atteint jamais.

Pour le mélange de messages, la règle est : aussi réaliste que nécessaire. Une mesure avec uniquement des e-mails texte de 5 Ko surestime de plusieurs fois le débit d’un environnement dont le quotidien comporte des pièces jointes PDF. Un mélange issu de son propre parc est pertinent, par exemple 70 % de petits messages, 25 % avec une pièce jointe typique et 5 % de gros messages. TLS doit également faire partie du test si la production utilise TLS : le handshake coûte nettement plus par connexion que le transfert des messages lui-même, et des générateurs qui ouvrent une nouvelle connexion pour chaque e-mail mesurent sinon principalement la terminaison TLS.

Pour l’évaluation ultérieure, chaque message de test reçoit un marqueur unique, le plus simplement un en-tête propre tel que `X-Loadtest-Id` avec numéro d’exécution et horodatage, ainsi qu’une convention d’objet reconnaissable. Les exécutions de test peuvent ainsi être clairement séparées dans les logs les unes des autres et du trafic restant, et les e-mails de test peuvent être supprimés de façon ciblée des quarantaines et des journaux après l’exécution.

Trois points sont régulièrement oubliés dans la planification : premièrement, les limites de débit et le tarpitting sur le chemin de test ; une passerelle qui ralentit après 100 e-mails par minute et par IP source ne teste sinon que sa propre limitation (les exclure délibérément pour la mesure de charge maximale, les laisser volontairement actifs pour le contrôle de réalisme). Deuxièmement, le DNS : si le système testé résout des domaines destinataires ou effectue des requêtes DNSBL pour chaque message, un résolveur doit faire partie de l’environnement de test, faute de quoi le test mesure le DNS amont. Troisièmement, le générateur lui-même : avant le premier passage vers le système cible, il faut exécuter le générateur directement contre le puits afin de prouver qu’il peut réellement produire le débit cible.

## Évaluer les résultats

Les valeurs mesurées par le générateur de charge ne constituent que la moitié de la vérité, car elles montrent le point de vue de l’émetteur. L’autre moitié se trouve dans les logs du système testé.

Sous Postfix, le log de messagerie fournit par message les champs `delay` et `delays`, ce dernier étant ventilé entre temps dans la file d’attente, établissement de connexion et transfert. Une évaluation sur une exécution de test se fait avec les outils intégrés :

```bash
grep "status=sent" /var/log/mail.log |
  grep -o "delay=[0-9.]*" | cut -d= -f2 | sort -n |
  awk '{a[NR]=$1} END {print "n="NR, "p50="a[int(NR*0.5)], "p95="a[int(NR*0.95)], "p99="a[int(NR*0.99)], "max="a[NR]}'
```

Côté Exchange, le Message Tracking Log est la source centrale. Pour une exécution de test avec convention d’objet :

```powershell
$p = @{
    Start          = "24.08.2026 14:00"
    End            = "24.08.2026 15:00"
    MessageSubject = "LOADTEST"
    ResultSize     = "Unlimited"
}
Get-MessageTrackingLog @p | Group-Object EventId |
    Sort-Object Count -Descending | Format-Table Name, Count
```

La différence entre les horodatages des événements RECEIVE et DELIVER du même MessageId donne la latence de bout en bout par message ; une fois exportée en CSV, elle permet de calculer la distribution des percentiles.

Trois principes comptent pour l’interprétation. Premièrement : les percentiles plutôt que les moyennes. Une moyenne de deux secondes peut signifier que tout prend deux secondes, ou que 95 % passent en une demi-seconde et que le reste reste bloqué dans la file d’attente ; p50, p95 et p99 distinguent ces cas. Deuxièmement : analyser les codes de réponse SMTP. La distribution des réponses 4xx dans le temps montre quand le système commence à ralentir, et les codes concernés (limite de connexions, protection de file d’attente, greylisting) indiquent quel mécanisme intervient en premier. Troisièmement : représenter la profondeur de la file d’attente dans le temps, sous Postfix via `qshape` ou `postqueue -j`, sous Exchange via `Get-Queue` à intervalles d’une minute. C’est l’aire sous cette courbe, et non le taux d’acceptation, qui détermine si l’environnement absorbe une rafale ou s’il ne fait que la stocker.

Parallèlement aux logs de messagerie, les métriques système du système testé doivent être intégrées à l’évaluation : CPU, temps d’attente d’E/S sur la partition de spool, descripteurs de fichiers, compteurs de connexions. Dans les environnements à plusieurs niveaux, le constat le plus fréquent est que le processus de messagerie n’est pas limitant, mais qu’une étape d’inspection de contenu (antivirus, module de chiffrement, DLP) avec un nombre fixe de workers l’est. De tels constats constituent la véritable valeur du test : ils identifient le levier de réglage avant que la production ne le révèle.

## Conclusion

Pour une mesure rapide de la charge maximale sous Linux, `smtp-source` avec `smtp-sink` est incontournable ; Postal complète le cas de la charge continue. Sous Windows, JMeter fournit la mesure la plus complète, PowerShell avec MailKit couvre les tests fonctionnels et de régression, et WSL apporte au besoin les outils Linux au poste de travail d’administration. Plus important que l’outil, il y a le plan : mesure séparée de l’acceptation, de la latence et du comportement de la file d’attente, mélange de messages réaliste, exécution de test marquée et évaluation intégrant les percentiles et les logs du système cible plutôt que le seul compteur du générateur.

## Sources

1.  [smtp-source(1), manuel Postfix](https://www.postfix.org/smtp-source.1.html): Référence du générateur de charge avec toutes les options de parallélisme, de taille des messages et de TLS.

2.  [smtp-sink(1), manuel Postfix](https://www.postfix.org/smtp-sink.1.html): Référence du puits de messagerie, y compris les retards artificiels et les réponses d’erreur.

3.  [Documentation Postal, Russell Coker](https://doc.coker.com.au/projects/postal/): Description du benchmark de serveur de messagerie avec débit cible, variation des messages et puits bhm.

4.  [swaks, John Jetmore](https://www.jetmore.org/john/code/swaks/): Le testeur fonctionnel SMTP pour le contrôle préalable de chaque chemin de test.

5.  [Apache JMeter Component Reference: SMTP Sampler](https://jmeter.apache.org/usermanual/component_reference.html): Fonctionnalités du SMTP Sampler, y compris authentification, TLS et sources EML.

6.  [Send-MailMessage, Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/send-mailmessage): Indication officielle de Microsoft selon laquelle le cmdlet est obsolète, avec renvoi vers des alternatives telles que MailKit.

7.  [MailKit, Jeffrey Stedfast](https://github.com/jstedfast/MailKit): La bibliothèque de messagerie .NET pour les scripts d’envoi personnalisés sous PowerShell 7.

8.  [Get-MessageTrackingLog, Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): Référence pour l’évaluation du Exchange Message Tracking Log après une exécution de test.

9.  [qshape(1), manuel Postfix](https://www.postfix.org/qshape.1.html): Outil d’analyse de la distribution de la file d’attente pendant et après la rafale.
