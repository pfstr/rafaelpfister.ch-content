---
title: "Migration : architecture cible, coexistence et bascule contrôlée"
blatt: "migration"
description: "Migration technique pour les administrateurs de messagerie et d’infrastructure : architectures source et cible, dépendances, mappage des identités et des données, Rehost à Refactor, coexistence, synchronisation initiale et différentielle, pilote, bascule, limites de responsabilité, retour arrière, réception et mise hors service."
fakten:
  - label: Point de départ
    wert: transaction métier + architecture source + états + propriétaires
    href: https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html
  - label: Objectif
    wert: architecture cible mesurable avec interfaces, souveraineté des données, limites de confiance et de défaillance
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/final
  - label: Stratégies
    wert: Retain · Retire · Rehost/Relocate · Repurchase · Replatform · Refactor
    href: https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html
  - label: Modèles
    wert: Cutover · pilote/vagues · coexistence · remplacement progressif
    href: https://www.martinfowler.com/bliki/OriginalStranglerFigApplication.html
  - label: Identité
    wert: représenter séparément l’ID source, l’ID cible, la connexion, les adresses SMTP, les groupes, les rôles et le cycle de vie
    href: https://datatracker.ietf.org/doc/html/rfc7643
  - label: Chemin des données
    wert: Seed → synchronisation différentielle → Freeze/Final Delta → rapprochement
    href: https://learn.microsoft.com/en-us/exchange/mailbox-migration/perform-a-staged-migration/perform-a-staged-migration
  - label: Coexistence
    wert: carnet d’adresses · routage de messagerie · découverte · authentification · autorisations · calendrier · support
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Responsabilité des messages
    wert: 250 après End-of-Data signifie l’acceptation par le saut SMTP qui répond
    href: https://datatracker.ietf.org/doc/html/rfc5321
  - label: Bascule DNS
    wert: le TTL limite la durée de vie du cache ; vérifier séparément les caches négatifs et les remplacements locaux
    href: https://datatracker.ietf.org/doc/html/rfc1034
  - label: Limite de retour arrière
    wert: la définir explicitement avant la première écriture exclusive dans la cible
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/final
  - label: Réception
    wert: exhaustivité · intégrité · autorisation · transaction · performance · monitoring
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/final
  - label: Mise hors service
    wert: traiter de manière contrôlée le routage, les identités, les clés, les sauvegardes, les journaux, les contrats et les supports de données
    href: https://csrc.nist.gov/pubs/sp/800/88/r1/final
werbung:
  - stargate
  - newsletter
ctaThemen:
  - migration
  - messaging
  - exchange-hybrid
translationSourceHash: bdb2fea86e1f811e78f2f210afd26b082f808325322fb675a8ec446e47b38ac3
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:36:21.970Z
translationReview: automatic
---

# Migration : architecture cible, coexistence et bascule contrôlée

Une migration technique transfère des processus métier, des interfaces et des états d’une architecture source vers une architecture cible définie. Le changement de produit visible n’en est qu’une partie. Les identités, adresses, autorisations, files d’attente, données, clés, DNS, certificats, clients, automatisations, monitoring, sauvegardes et processus de support doivent continuer à honorer les mêmes engagements fonctionnels, ou être délibérément transférés vers un nouveau contrat.

Pour les systèmes de messagerie, l’engagement le plus important est la responsabilité contrôlée de chaque message. Pendant une migration, les anciennes et nouvelles plateformes peuvent simultanément connaître des adresses, accepter des connexions et modifier des données. Sans souveraineté explicite sur les données, des boucles, des carnets d’adresses split-brain, des distributions doubles, des autorisations perdues et un chemin de retour qui n’existe que dans la planification apparaissent. Une migration robuste commence donc par l’architecture et les états, non par la date de bascule.

L’explication part de l’état existant documenté et mène, via l’architecture cible, les dépendances et le transfert de données, à la coexistence. Le pilote, la bascule, le retour arrière et la réception ne suivent que lorsqu’il est clair quelles données et fonctions sont déplacées.

Une migration n’est pas une opération de copie, mais le passage contrôlé d’une architecture source opérationnelle à une architecture cible vérifiée. Les états et dépendances sont d’abord recensés, puis la coexistence, le mouvement des données, la bascule et le chemin de retour sont planifiés.

## De l’architecture source à l’architecture cible

Avant de choisir les produits, le service existant est documenté sous forme de graphe. Chaque nœud possède un rôle ; chaque arête possède un initiateur, une cible, un protocole, une identité, une classe de données, un comportement en cas d’erreur et un propriétaire. Une liste CMDB de noms de serveurs ne suffit pas si l’on ignore quel scanner appelle un saut SMTP, quel service développe les groupes ou quelle clé déchiffre les anciens messages.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-migration.svg?v=20260813" title="Interaktive Infografik: Migration von Ausgangs- zu Zielarchitektur mit Strategie, Zustandsklassen, Koexistenz, Seed und Delta, Pilotwellen, Cutover, Rollbackgrenze, Abnahme und Stilllegung" loading="lazy">
  <a href="/images/kb-interaktiv-migration.svg?v=20260813">Ouvrir directement le graphique interactif</a>.
</iframe>

La documentation source couvre au minimum :

| Domaine | État existant à recenser | Contrat caché typique |
|---|---|---|
| Routage des e-mails | MX, écouteurs de relais/soumission, connecteurs, smarthosts, domaines acceptés, IP source, politique TLS | Un partenaire ajoute une ancienne IP à sa liste d’autorisation ou attend un nom de certificat |
| Identité | Source d’annuaire, ancre d’objet, connexion, groupes, rôles, comptes de service, provisioning | Une application utilise l’adresse e-mail comme ID immuable |
| Données | Boîte aux lettres, archives, journaux, file d’attente, quarantaine, blob, index, configuration, audit | L’index de recherche est reconstructible, mais le journal est soumis à une obligation de conservation |
| Cryptographie | Certificats TLS, clés S/MIME/OpenPGP, HSM/KMS, phrases secrètes, récupération | Un ancien message chiffré nécessite une clé privée historique |
| Clients et API | Outlook/IMAP/SMTP/API, agents, mobile, webhooks, scripts | URL, chiffrement, port, en-tête ou champ API codé en dur |
| Exploitation | Monitoring, sauvegarde, restauration, correctifs, journalisation, support, licences | L’alerte vérifie uniquement l’état du processus de l’ancien système |
| Organisation | Propriétaire, astreinte, approbation, protection des données, conservation, contrat | Une équipe externe contrôle le DNS ou le pare-feu avec un long délai de préparation |

L’architecture cible décrit les mêmes champs comme **contrat cible**. Les frontières de confiance, la souveraineté des données, les domaines de défaillance, les objectifs de reprise, la mise à l’échelle et l’état final planifié de l’ancien système sont également définis. NIST SP 800-34 relie le développement de systèmes à la planification de continuité et exige que les stratégies et tests de reprise soient dérivés des exigences métier. Pour les migrations, il en résulte que l’architecture cible et l’exploitation de repli sont conçues ensemble ([NIST SP 800-34 Rev. 1](https://csrc.nist.gov/pubs/sp/800/34/r1/final)).

## Stratégie de migration : que modifie-t-on réellement ?

Les « 7 Rs » documentés par AWS distinguent Retire, Retain, Rehost, Relocate, Repurchase, Replatform et Refactor/Re-architect. Il s’agit d’une taxonomie de migration vers le cloud, mais elle peut s’appliquer comme question précise à toute migration de plateforme : la fonction est-elle conservée, supprimée, déplacée sans changement, remplacée par un produit, adaptée à une nouvelle plateforme ou reconstruite fonctionnellement ? AWS indique que le refactoring est la stratégie la plus complexe dans les grandes migrations et qu’une modernisation distincte peut réduire les risques ([AWS Prescriptive Guidance – Migration Strategies](https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html)).

| Stratégie | Qu’est-ce qui reste identique ? | Qu’est-ce qui change ? | Exemple de messagerie |
|---|---|---|---|
| Retain | Plateforme et responsabilité | uniquement exception/délai documenté | L’archive legacy reste en lecture jusqu’à la fin de la rétention |
| Retire | Rien de productif | Fonction et accès supprimés | Un point d’accès POP3 inutilisé est désactivé |
| Rehost / Relocate | Application et modèle de données en grande partie | Emplacement du calcul/de la virtualisation | La VM MTA est déplacée vers une autre infrastructure |
| Repurchase | Exigence métier | Produit, modèle de données, exploitation et contrat | Une passerelle appliance est remplacée par une passerelle SaaS |
| Replatform | Fonction métier et parties de l’architecture | Runtime, base de données ou modèle d’exploitation | Le service de messagerie utilise une nouvelle plateforme de base de données/conteneurs |
| Refactor | Uniquement l’intention métier | Interfaces, composants et états | Un pipeline de messagerie monolithique est reconstruit selon une approche événementielle |

La stratégie est choisie **par composant et classe de données**. Un projet peut replatformer le routage des e-mails, conserver les données d’archive, retirer un outil de reporting et repurchase l’interface utilisateur. Appeler globalement le tout « Lift and Shift » masque les risques réellement différents.

## Cutover, vagues, coexistence et remplacement progressif

Quatre modèles de transition sont techniquement différents et doivent être dérivés de capacités produit documentées :

- **Cutover unique :** tous les chemins d’écriture basculent dans une fenêtre. Approprié lorsque la coexistence n’est pas techniquement possible, que l’état est limité et que l’interruption est acceptable. Le chemin de retour doit être clair avant la première écriture dans la cible.
- **Pilote et vagues :** une population représentative bascule d’abord ; d’autres lots suivent après réception. Les lots de migration Microsoft sont un exemple concret de transfert progressif des boîtes aux lettres ([Microsoft – Staged Email Migration](https://learn.microsoft.com/en-us/exchange/mailbox-migration/perform-a-staged-migration/perform-a-staged-migration)).
- **Coexistence :** l’ancien et le nouveau partagent temporairement un espace de noms, un carnet d’adresses, un routage ou une population d’utilisateurs. Exchange Hybrid documente notamment le flux de messagerie sécurisé, l’espace de noms de domaine partagé, le carnet d’adresses et le partage de calendriers comme fonctions de coexistence ([Microsoft – Exchange Hybrid Deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).
- **Remplacement progressif :** une couche de routage ou d’intégration intercepte les fonctions et les déplace progressivement vers la cible. Martin Fowler a décrit ce modèle en 2004 comme « Strangler Fig Application » afin de réduire le risque d’une unique grande bascule de réécriture ([Martin Fowler – Original Strangler Fig Application](https://www.martinfowler.com/bliki/OriginalStranglerFigApplication.html)).

Aucun de ces modèles n’est sûr par principe. La coexistence réduit l’ampleur de la bascule, mais engendre des états, des règles de routage et des cas de support supplémentaires. Le Dual Write peut accroître la disponibilité, mais crée des problèmes de résolution de conflits et d’ordre. Une bascule unique est plus simple à comprendre, mais agrandit le rayon de dommage simultané. La décision mentionne donc la preuve produit, la durée admissible, la souveraineté des données et les critères de sortie.

Après le choix de la stratégie, il faut déterminer pour chaque classe de données quel système peut écrire durant chaque phase. Deux sources de vérité non coordonnées produisent des conflits qu’un outil de copie ne peut pas résoudre.

## Classes d’état et souveraineté des données

Avant toute synchronisation, il est déterminé quelles données constituent une **source faisant autorité**, une **copie répliquée**, une **projection dérivée** ou une **archive historique uniquement** :

| Classe d’état | Autorité pendant la coexistence | Question de conflit/reprise |
|---|---|---|
| Données de référence d’identité | exactement un annuaire directeur ou système RH | quelle ancre d’objet survit au renommage et au changement de tenant ? |
| Adresses SMTP et objets de routage | une source d’adresses approuvée | comment empêcher les doublons et les règles catch-all/alias ? |
| Contenu des boîtes aux lettres | pour chaque boîte migrée, source ou cible clairement définie | peut-on encore écrire dans la boîte source après le cutover ? |
| File d’attente | le MTA qui accepte porte la responsabilité | comment vider les anciennes files sans les réinjecter deux fois ? |
| Politique et configuration | état cible déclaratif ou maître désigné | quelles règles sont sémantiquement équivalentes, et non simplement copiées syntaxiquement ? |
| Clés et certificats | KMS/HSM ou propriétaire de clé défini | quelles clés privées historiques doivent rester lisibles ? |
| Audit, journal et rétention | source de conservation immuable | le temps, l’exhaustivité, le Legal Hold et l’export sont-ils démontrables ? |
| Index/cache | projection reconstructible | combien de temps dure la reconstruction, et à quel moment la recherche est-elle complète ? |

« Synchronisé bidirectionnellement » n’est pas un modèle de données sans règle de conflit. Pour chaque champ sont documentés la direction, la fréquence, la transformation, le stockage des erreurs et le comportement en cas de modification simultanée. Si l’outil ne peut pas fusionner les modifications sans perte, un côté est mis en lecture seule durant la phase concernée ou modifié uniquement via un chemin d’écriture centralisé.

## Identité, adresses et autorisations

Le nom d’utilisateur, l’adresse e-mail et l’ID technique d’objet sont des choses différentes. RFC 7643 définit notamment, pour SCIM, `id` comme un identifiant attribué par le fournisseur de services, immuable et unique au sein du tenant ; `externalId` peut porter une corrélation gérée par le client. Cela illustre une règle centrale de migration : une adresse modifiable n’est pas une ancre d’objet fiable ([RFC 7643](https://datatracker.ietf.org/doc/html/rfc7643)).

Un mappage d’identité contient au minimum l’ID source, l’ID cible, `externalId` ou une autre ancre de corrélation, la connexion/UPN, l’adresse SMTP principale, les alias, les groupes, les rôles, les attributs de responsable/organisation, les comptes de service, la méthode d’authentification et le statut de provisioning. RFC 7644 décrit les opérations SCIM de création, modification, recherche et traitement en masse ; la plateforme cible concrète peut néanmoins avoir des champs obligatoires, règles d’unicité et limites de débit différents ([RFC 7644](https://datatracker.ietf.org/doc/html/rfc7644)).

Les autorisations sont migrées comme des relations, non comme des listes de texte. La délégation, Send As, Send on Behalf, Folder ACL, Shared Mailbox, la réservation de ressources, Application Access et le rôle d’administrateur peuvent référencer différents types d’objets. Chaque relation nécessite une ancre cible résolue ; les références orphelines ou cycliques sont signalées avant la vague. Microsoft documente par exemple que la migration de boîtes aux lettres inter-tenant requiert des objets cible-`MailUser` et certains attributs, et ne transfère que des contenus/autorisations définis ([Microsoft – Cross-tenant Mailbox Migration](https://learn.microsoft.com/en-us/microsoft-365/migration/cross-tenant-mailbox-migration?view=o365-worldwide)).

## Chemin des données : Seed, Delta, Freeze et rapprochement

Les états volumineux sont généralement préchargés avant le cutover. Le processus robuste est le suivant :

1. **Seed :** inventaire initial complet avec nombre d’objets, volume en octets, liste d’erreurs et période.
2. **Synchronisation Delta :** transfert répété des changements depuis le Seed ; surveillance du retard et du taux d’erreur.
3. **Freeze ou redirection des écritures :** arrêt des chemins d’écriture source précisément définis ou redirection vers la cible.
4. **Final Delta :** rattrapage des changements restants et sécurisation d’un niveau d’eau reproductible.
5. **Cutover :** bascule du routage, de la découverte et de l’accès utilisateur vers la cible.
6. **Rapprochement :** comparaison des nombres, tailles, attributs clés, autorisations, échantillons et transactions métier.

La migration de messagerie progressive de Microsoft documente les lots, l’état de synchronisation et l’exigence d’attendre au moins une synchronisation supplémentaire après la modification du routage avant de supprimer un lot. Il s’agit d’une preuve spécifique au produit pour les phases Seed/Delta/finalisation, et non d’une instruction universelle pour d’autres systèmes ([Microsoft – Staged Email Migration](https://learn.microsoft.com/en-us/exchange/mailbox-migration/perform-a-staged-migration/perform-a-staged-migration)).

Les sommes de contrôle ne démontrent l’égalité des octets qu’en cas de sérialisation identique. L’export/import peut normaliser les messages, transformer les métadonnées ou structurer les conteneurs différemment. La réception combine donc les sommes techniques avec des invariants métier : nombres d’objets par classe, horodatage le plus ancien/le plus récent, taille totale, objets manquants/plus grands, relations d’autorisation, échantillon aléatoire et cas délibérément difficiles tels que les grands messages MIME, les récurrences de calendrier, Unicode, le chiffrement et le Legal Hold.

Une fois l’identité et la souveraineté des données définies, le trafic de messages en cours peut être conduit entre l’ancien et le système cible. La coexistence est alors une architecture de production temporaire avec ses propres cas de routage et d’erreur.

## Architecture de coexistence pour la messagerie

Un domaine SMTP commun sur deux plateformes nécessite une décision sans ambiguïté concernant les destinataires et le routage. Microsoft décrit pour l’exploitation hybride un espace de noms de domaine partagé, un flux de messagerie sécurisé, un carnet d’adresses commun et des fonctions de calendrier ; pour la coexistence avec des fournisseurs tiers, les décisions d’identité, de synchronisation et d’authentification sont explicitement mises en avant ([Microsoft – Exchange Hybrid Deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid), [Microsoft – Third-party Email Coexistence](https://learn.microsoft.com/en-us/exchange/mailbox-migration/plan-to-coexist-with-third-party-messaging-and-aad-ds)).

La matrice locale de coexistence répond aux questions suivantes :

- Quelle plateforme connaît chaque destinataire et avec quel type ?
- Qui est responsable des destinataires inconnus, et comment les boucles sont-elles évitées ?
- Quel saut signe DKIM, applique DLP/archive/disclaimer ou chiffre ?
- Comment l’ancien et le nouveau reconnaissent-ils l’autre comme saut interne, authentifié ou externe ?
- Où se trouvent Free/Busy, délégation, développement de groupes et Autodiscover/Service Discovery ?
- Quelle plateforme écrit quels attributs d’identité et d’adresse ?
- Comment les bounces, la quarantaine, les journaux et le Message Trace sont-ils corrélés entre les deux côtés ?
- Quel service de support possède une vue de bout en bout ?

Chaque transition est rendue observable avec une réponse [SMTP](/kb/smtp)-, l’ID de file d’attente et l’ID de trace/message. RFC 5321 définit que le serveur destinataire assume la responsabilité après la fin réussie de DATA. Un retour arrière après une réponse `250` ne doit pas oublier ce message accepté ni le réinjecter de manière non contrôlée ([RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

## DNS, certificats et contrats externes

Le DNS est un mécanisme de sélection distribué, pas un interrupteur global. RFC 1034 décrit la mise en cache comme un composant fondamental et laisse à la source de données le compromis TTL entre actualité et charge. Avant une transition, les TTL pertinents sont donc réduits suffisamment tôt ; après le cutover, les anciens chemins restent contrôlés au moins aussi longtemps que les résolveurs, caches négatifs, fichiers hosts locaux, proxys et passerelles externes peuvent utiliser d’anciennes données ([RFC 1034](https://datatracker.ietf.org/doc/html/rfc1034)).

Il convient de vérifier MX, A/AAAA, CNAME, SRV et TXT ainsi que les vues internes/externes. SPF doit autoriser les nouvelles IP d’envoi sans supprimer trop tôt les anciennes sources ; les sélecteurs DKIM et l’alignement DMARC sont vérifiés avec de vrais messages. Les certificats nécessitent des noms, une chaîne, la confiance, une clé privée, Extended Key Usage et la capacité d’exportation/HSM. Les clés de déchiffrement historiques S/MIME/OpenPGP suivent leur propre cycle de vie. NIST SP 800-57 traite des états, de la protection et de l’archivage des clés ; un changement de plateforme ne supprime pas ces exigences ([NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)).

Les contrats externes comprennent les listes d’autorisation de partenaires, les identifiants de smarthost, les URI de redirection API, les webhooks, les sources SIEM, les agents de sauvegarde, les sondes de monitoring, les pare-feux, NAT, le DNS inverse et les accès de support. Chaque contrat a un propriétaire technique et un test avant le cutover.

## Pilote, vagues et portes de décision

Un pilote est représentatif, et non seulement choisi parmi des volontaires bienveillants. Il comprend de grandes et petites boîtes aux lettres, des délégations, des boîtes aux lettres partagées/de ressources, des clients mobiles et anciens encore pris en charge, des partenaires externes, le chiffrement, des archives, des Power Users, différents sites et au moins un cas négatif contrôlé. La population pilote ne doit pas contenir simultanément une fonction unique irréversible.

Chaque vague comporte des portes :

- **Ready :** mappage complet, objets cibles existants, Seed dans l’objectif de retard, client/réseau/politique prêts, restauration et support assurés.
- **Go :** dernière vérification Delta et d’intégrité réussie, aucune classe de blocage ouverte, dépendances et créneau horaire confirmés.
- **Hold :** taux d’erreur, retard, file d’attente, réplication ou charge de support hors seuil ; aucun objet supplémentaire ne démarre.
- **Abort :** intégrité des données, sécurité, perte de messagerie incontrôlable ou autres critères rouges définis à l’avance.
- **Accept :** réception métier et période d’observation postérieure satisfaites ; l’objet appartient désormais clairement à la cible.

Les vagues sont découpées de sorte qu’une perturbation reste limitée : par exemple selon le site, le tenant, le domaine, la base de données, le type de client, le groupe fonctionnel ou la dépendance. Des ensembles aléatoires peuvent être représentatifs, mais sont difficiles à rétablir opérationnellement. Chaque vague reçoit une liste d’objets immuable et un enregistrement de résultat.

Le pilote et les vagues apportent les preuves pour le cutover réel. Le retour ne reste réaliste que tant que les changements d’état peuvent encore être entièrement retransférés vers l’ancien système.

## Cutover et limite de retour arrière

Le runbook de cutover ne contient pas seulement des étapes, mais aussi des **conditions préalables, un propriétaire, le constat attendu, la durée maximale, le critère d’arrêt et le chemin de retour**. Une séquence possible est :

1. Geler les changements, ouvrir le canal d’incident/de communication et sécuriser les métriques initiales.
2. Vérifier le Final Delta et les erreurs de mappage ; bloquer les chemins d’écriture source conformément au modèle.
3. Tester à nouveau les écouteurs cibles, certificats, identité, politique et dépendances en aval.
4. Basculer les contrôles internes de routage/découverte, puis les chemins DNS/partenaires externes.
5. Exécuter des transactions contrôlées inbound, outbound, Submission, Access, Policy et chiffrement.
6. Observer la file d’attente, les codes SMTP, l’authentification, la réplication, l’index, les erreurs client et le monitoring.
7. Décider Go/Hold/Abort et documenter l’heure.

Le **Rollback Point** se situe avant le moment où la cible assume la responsabilité d’écriture exclusive, non réversible sans perte. Après cela, une bascule aveugle vers la source n’est pas un retour arrière, mais une deuxième migration. Les nouveaux messages dans la cible, modifications de calendrier, autorisations, clés et états de file d’attente doivent alors être resynchronisés, exportés ou préservés par un roll-forward. Le modèle [Backup-und-DR-Modell](/kb/backup-dr) définit le point de cohérence, le RPO/RTO et l’ordre de restauration.

## Outils d’administration pour les preuves avant/après

Les exemples vérifient des contrats individuels. Ils sont exécutés depuis l’origine réellement concernée et avec des données de test approuvées. Les identifiants, contenus personnels et clés privées ne doivent pas être placés sans protection dans des tickets ou l’historique du shell.

### Comparer le DNS avant et après le cutover

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-DNS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName example.ch -Type MX -DnsOnly -Server 192.0.2.53
Resolve-DnsName autodiscover.example.ch -Type CNAME -DnsOnly -Server 192.0.2.53
Resolve-DnsName example.ch -Type TXT -DnsOnly -Server 192.0.2.53</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig @192.0.2.53 +noall +answer example.ch MX
dig @192.0.2.53 +noall +answer autodiscover.example.ch CNAME
dig @192.0.2.53 +noall +answer example.ch TXT</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) et [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) rendent explicites le résolveur et le type d’enregistrement. Les résolveurs internes et externes sont vérifiés séparément ; une réponse correcte d’un résolveur public ne prouve pas que le service de production utilise la même vue.

### Atteindre les nouveaux points d’accès depuis l’origine réelle

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-TCP-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection new-mx.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection new-api.example.ch -Port 443 -InformationLevel Detailed</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz -w 5 new-mx.example.ch 25
nc -vz -w 5 new-api.example.ch 443</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) et [`nc`](https://man.openbsd.org/nc) démontrent uniquement l’accessibilité DNS/TCP depuis l’origine choisie. Pare-feu, NAT, IP source, TLS, authentification et protocole métier sont ensuite vérifiés séparément.

### Vérifier l’identité TLS et la réponse de protocole

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-TLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">openssl.exe s_client -connect new-mx.example.ch:25 -starttls smtp -servername new-mx.example.ch -showcerts
curl.exe --verbose --connect-timeout 5 https://new-api.example.ch/health</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">openssl s_client -connect new-mx.example.ch:25 -starttls smtp -servername new-mx.example.ch -showcerts
curl --verbose --connect-timeout 5 https://new-api.example.ch/health</code></pre>
  </div>
</div>

[`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) et [`curl`](https://curl.se/docs/manpage.html) vérifient TLS et un dialogue contrôlé. Tous deux utilisent leur propre configuration de confiance/proxy/build ; le client de production doit en outre être testé avec sa politique réelle.

### Exporter et comparer le mappage d’identité

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-Identitätsprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-ADUser -Filter * -Properties ObjectGUID,UserPrincipalName,mail,proxyAddresses,Enabled |
  Select-Object ObjectGUID,SamAccountName,UserPrincipalName,mail,proxyAddresses,Enabled |
  Export-Csv .\\identity-source.csv -NoTypeInformation -Encoding UTF8</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ldapsearch -LLL -x -H ldaps://directory.example.ch \
  -b 'dc=example,dc=ch' '(objectClass=person)' \
  entryUUID uid userPrincipalName mail proxyAddresses</code></pre>
  </div>
</div>

[`Get-ADUser`](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser) et [`ldapsearch`](https://www.openldap.org/software/man.cgi?query=ldapsearch) fournissent des attributs d’identité issus des annuaires respectifs. L’export et la requête doivent tenir compte du périmètre, de la pagination, des objets supprimés/désactivés, des attributs à valeurs multiples et des droits d’accès. Le rapprochement cible utilise l’ancre d’objet convenue, et non seulement le nom d’affichage.

### Vérifier les artefacts d’export au niveau des octets

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-Hashprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-FileHash .\\export\\mailbox-001.pst -Algorithm SHA256
Get-FileHash .\\export\\mapping.csv -Algorithm SHA256</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sha256sum ./export/mailbox-001.mbox
sha256sum ./export/mapping.csv</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) et [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) détectent les modifications de fichiers identiques. Ils ne prouvent pas que les boîtes aux lettres source et cible sont sémantiquement identiques ; il faut pour cela des décomptes d’objets, attributs, listes d’erreurs et échantillons métier.

### Conserver les journaux dans la fenêtre temporelle du cutover

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-Logprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$since = [datetime]'2026-08-08T20:00:00Z'
Get-WinEvent -FilterHashtable @{LogName='System'; StartTime=$since} |
  Select-Object TimeCreated,Id,ProviderName,LevelDisplayName,Message</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">journalctl --since '2026-08-08 20:00:00 UTC' \
  --output short-iso-precise --no-pager</code></pre>
  </div>
</div>

[`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent) et [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) fournissent des événements système horodatés. S’y ajoutent les journaux produit, lots de migration, SMTP, annuaire, base de données et API avec ID de file/d’objet/de trace. Avant un redémarrage ou une rotation, les preuves volatiles sont sauvegardées.

## Réception et suivi

La réception ne vérifie pas seulement que « l’utilisateur peut se connecter », mais six dimensions :

1. **Exhaustivité :** objets, périodes, dossiers, attributs, règles, clés et autorisations attendus présents.
2. **Intégrité :** aucune modification non autorisée, aucun doublon, aucune relation perdue ou souveraineté des données incohérente.
3. **Autorisation :** utilisateurs, services et administrateurs peuvent exécuter exactement les opérations prévues, y compris les tests négatifs.
4. **Transaction :** inbound/outbound, Submission/Access, routage, politique, chiffrement, quarantaine, archive et API fonctionnent de bout en bout.
5. **Exploitation :** performance, file d’attente, réplication, index, sauvegarde, restauration, correctifs, support et capacité satisfont les valeurs cibles.
6. **Observabilité :** journaux, métriques, traces, suivi des messages, alerting et runbooks voient le chemin cible.

La phase de suivi s’étend sur les cycles pertinents de TTL, cache, retry, Delta, rétention et charge métier. Les anciennes plateformes ne restent actives que dans le rôle défini à l’avance : par exemple vider une file d’attente ou fournir une lecture historique, sans accepter discrètement de nouvelles données. Les redirecteurs temporaires et domaines de coexistence reçoivent une date d’expiration et du monitoring.

La mise hors service ne commence qu’après la réception métier et le suivi. Les journaux, clés, entrées DNS, comptes de service et intégrations cachées doivent être supprimés ou archivés de manière contrôlée.

## Mise hors service sans dépendances perdues

Le decommissioning est un changement d’état contrôlé à part entière. Avant l’arrêt, les derniers accès, files résiduelles, sessions actives, tâches, hits DNS/pare-feu, comptes de service, sauvegardes, obligations de restauration, Legal Holds, licences et contrats de support sont vérifiés. Ensuite, le routage et la découverte sont supprimés, les accès bloqués, les secrets et certificats révoqués ou archivés, le monitoring adapté et la CMDB mise à jour.

Les supports de données et ressources cloud sont supprimés ou rendus cryptographiquement inutilisables selon la classe de données et le contrat. NIST SP 800-88 décrit Clear, Purge et Destroy comme méthodes de sanitization et exige une décision adaptée au support et au risque ([NIST SP 800-88 Rev. 1](https://csrc.nist.gov/pubs/sp/800/88/r1/final)). Les sauvegardes et clés peuvent survivre plus longtemps que l’instance produit ; leur conservation, leur lisibilité et leur destruction définitive restent documentées.

## Évolution technique des migrations

Les premières migrations d’infrastructure étaient souvent un remplacement unique de matériel ou de logiciel avec une grande fenêtre de bascule. La virtualisation et les copies d’images ont facilité Rehost et Relocate ; les protocoles standardisés d’annuaire, d’API et de provisioning ont permis une coexistence plus fine. Les plateformes cloud et SaaS ont fait des frontières de tenant, d’identité et de contrat des objets de migration aussi importants que le serveur et la base de données. La taxonomie 7-R décrit ces différents degrés d’intervention ; le modèle Strangler Fig décrit le remplacement progressif de fonctions ([AWS – Migration Strategies](https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html), [Martin Fowler – Original Strangler Fig Application](https://www.martinfowler.com/bliki/OriginalStranglerFigApplication.html)).

Le progrès technique n’élimine pas le cutover. Il le déplace vers des unités plus petites : identités individuelles, domaines, API, classes de données ou décisions de routage. Cela réduit le rayon de dommage simultané, tandis que la coexistence et la souveraineté des données deviennent plus exigeantes. Une architecture de migration moderne n’optimise donc pas seulement la vitesse de copie, mais **la clarté des responsabilités, l’observabilité de chaque transition et un véritable chemin de retour ou de progression**.

## Sources

- [NIST – SP 800-34 Rev. 1, planification de continuité](https://csrc.nist.gov/pubs/sp/800/34/r1/final)
- [AWS Prescriptive Guidance – stratégies de migration](https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html)
- [Microsoft – migration de messagerie progressive](https://learn.microsoft.com/en-us/exchange/mailbox-migration/perform-a-staged-migration/perform-a-staged-migration)
- [Microsoft – déploiements Exchange Hybrid](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)
- [Martin Fowler – application Strangler Fig originale](https://www.martinfowler.com/bliki/OriginalStranglerFigApplication.html)
- [IETF RFC 7643 – schéma SCIM Core](https://datatracker.ietf.org/doc/html/rfc7643)
- [IETF RFC 7644 – protocole SCIM](https://datatracker.ietf.org/doc/html/rfc7644)
- [Microsoft – migration de boîte aux lettres inter-tenant](https://learn.microsoft.com/en-us/microsoft-365/migration/cross-tenant-mailbox-migration?view=o365-worldwide)
- [Microsoft – coexistence de messagerie avec des fournisseurs tiers](https://learn.microsoft.com/en-us/exchange/mailbox-migration/plan-to-coexist-with-third-party-messaging-and-aad-ds)
- [IETF RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [IETF RFC 1034 – noms de domaine : concepts et fonctionnalités](https://datatracker.ietf.org/doc/html/rfc1034)
- [NIST – SP 800-57 Part 1 Rev. 5, gestion des clés](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [curl – page de manuel en ligne de commande](https://curl.se/docs/manpage.html)
- [Microsoft Learn – Get-ADUser](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser)
- [OpenLDAP – ldapsearch](https://www.openldap.org/software/man.cgi?query=ldapsearch)
- [Microsoft Learn – Get-FileHash](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – utilitaires SHA-2](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Microsoft Learn – Get-WinEvent](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [NIST – SP 800-88 Rev. 1, assainissement des supports](https://csrc.nist.gov/pubs/sp/800/88/r1/final)
