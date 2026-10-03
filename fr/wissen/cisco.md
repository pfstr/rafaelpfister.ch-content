---
title: "Cisco Secure Email : Gateway, AsyncOS et SMA"
blatt: "cisco"
description: "Cisco Secure Email pour les administrateurs de messagerie : pipeline de messagerie SEG/ESA, listeners, HAT et RAT, file d’attente de travail et livraison, politiques de messagerie, AsyncOS, services SMA, limites de cluster, pile technologique, supervision, récupération et diagnostic."
fakten:
  - label: Rôles des produits
    wert: Secure Email Gateway (SEG/ESA) · Secure Email and Web Manager (SMA)
    href: https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/datasheet-c78-742868.html
  - label: Rôle système
    wert: Passerelle de messagerie SMTP avant ou entre des systèmes de messagerie
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html
  - label: Système d’exploitation
    wert: Cisco AsyncOS
    href: https://www.cisco.com/c/en/us/support/security/email-security-appliance/series.html
  - label: Pipeline
    wert: Réception → file d’attente de travail → livraison
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html
  - label: Acceptation
    wert: Listener · HAT · Sender Groups · RAT · LDAP
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html
  - label: Traitement
    wert: Message Filters · Mail Policies · Content Filters · moteurs d’analyse
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html
  - label: Services centralisés
    wert: Tracking · reporting · quarantaines antispam et de politique sur SMA
    href: https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0100.html
  - label: Groupe de configuration
    wert: peer-to-peer ; aucune HA de file d’attente ou de trafic
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0101000.html
  - label: Formats de déploiement
    wert: Appliance virtuelle · cloud public · Cisco Cloud Gateway
    href: https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/datasheet-c78-742868.html
  - label: Administration
    wert: Interface Web · CLI via SSH · API REST
    href: https://docs.ces.cisco.com/docs/api
  - label: États principaux
    wert: Configuration · file d’attente · quarantaines · tracking/reporting · clés et certificats
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0100010.html
  - label: Origine
    wert: Technologie IronPort ; acquise par Cisco en 2007
    href: https://newsroom.cisco.com/c/r/newsroom/en/us/a/y2007/m01/cisco-announces-agreement-to-acquire-ironport.html
werbung:
  - tools
  - newsletter
ctaThemen:
  - cisco-esa-sma
translationSourceHash: c796a2c50226bbdcf5a0d7a7562913a5126b28cdbd05a5b72065afcf531bfd42
translationModel: gpt-5.6-terra
translatedAt: 2026-10-03T09:41:42.032Z
translationReview: automatic
---

# Cisco Secure Email : Gateway, AsyncOS et SMA

**Cisco Secure Email Gateway**, abrégé SEG et historiquement nommé **Email Security Appliance** ou ESA, est une passerelle de messagerie [SMTP](/kb/smtp). Elle termine les sessions SMTP entrantes, décide de l’acceptation, traite les messages dans une file d’attente de travail interne et ouvre une nouvelle session SMTP pour la livraison. La limite de responsabilité technique ne se situe donc pas au niveau du handshake TCP ou TLS réussi, mais à la réponse SMTP positive après le contenu du message : à partir de cet instant, la passerelle doit livrer le message ou générer une erreur conforme à la norme ([Cisco: Email Pipeline](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html), [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

Le deuxième rôle classique est le **Cisco Secure Email and Web Manager**, SMA. Il ne se trouve normalement pas dans le chemin de messagerie de production comme MTA régulier. Il prend en charge les données centralisées de tracking et de reporting ainsi que, selon la conception, les quarantaines de spam, de politiques, de virus et d’épidémies de plusieurs passerelles. Une panne du SMA peut donc laisser la livraison intacte sur les nœuds SEG tout en interrompant la recherche, la quarantaine des utilisateurs finaux ou le traitement des messages retenus ([Cisco: SMA Message Tracking](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0100.html), [Cisco: Centralized Quarantines](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0110.html)).

Les deux rôles s’exécutent sur **AsyncOS**, une plateforme logicielle maintenue par Cisco en tant qu’unité d’appliance. Les administrateurs ne gèrent pas les paquets sous-jacents comme sur un serveur Linux généraliste ; l’interface technique fiable se compose de la configuration AsyncOS, de la CLI, de l’interface Web, de l’API REST, des abonnements aux journaux, des MIB, des canaux de mise à jour et des intégrations documentées. Les aperçus open source de Cisco attestent de nombreux composants intégrés, mais pas d’une nomenclature publiquement maintenable du pipeline de messagerie propriétaire. Les bibliothèques individuelles ne doivent donc pas être assimilées à l’architecture globale ([Cisco: Open Source Used in AsyncOS](https://www.cisco.com/c/dam/en_us/about/doing_business/open_source/docs/CiscoEmailSecurityAppliance-AsyncOS1253-1688323737.pdf), [Cisco: AsyncOS API](https://docs.ces.cisco.com/docs/api)).

L’explication accompagne un message à travers Cisco Secure Email : du listener, via HAT, RAT et la file d’attente de travail, jusqu’à la livraison. Viennent ensuite SMA, le fonctionnement en cluster, les dépendances, le diagnostic et la récupération.

## Rôles des produits et limites de confiance

Une conception on-premises typique place au moins deux nœuds SEG dans la DMZ et un SMA dans un réseau de gestion interne. Les DNS MX ou un service en amont répartissent les connexions entrantes vers les passerelles ; en sortie, les connecteurs de smarthost du système de messagerie déterminent le chemin via la passerelle. Plusieurs nœuds SEG ne sont hautement disponibles que si DNS, load balancer ou MTA émetteur peuvent utiliser des cibles alternatives. Le cluster de configuration AsyncOS seul n’assure pas ce routage du trafic ([Cisco: Centralized Management Using Clusters](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0101000.html), [RFC 5321, Address Resolution](https://datatracker.ietf.org/doc/html/rfc5321)).

Cisco documente des appliances virtuelles, des déploiements dans le cloud public et un Secure Email Cloud Gateway exploité. Ces variantes partagent des termes de produit, mais déplacent les responsabilités : pour l’appliance virtuelle, le client est responsable de l’hyperviseur, du réseau, de la capacité et de la restauration ; pour le Cloud Gateway, Cisco fournit l’infrastructure de passerelle. **Secure Email Threat Defense** est quant à lui une plateforme cloud native d’analyse et de protection qui peut être intégrée par passerelle, journalisation ou API Microsoft. Ce n’est ni un synonyme de la file d’attente de travail locale ni un terme de remplacement pour le SMA ([Cisco: Secure Email Data Sheet](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/datasheet-c78-742868.html), [Cisco: Email Threat Defense Data Sheet](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/secure-email-threat-defense-ds.html)).

| Rôle | Dans le chemin SMTP | État persistant | Effet d’une panne |
|---|---:|---|---|
| Secure Email Gateway | oui | File d’attente, quarantaines locales, configuration, certificats, journaux | Acceptation ou livraison perturbée sur ce nœud |
| Secure Email and Web Manager | normalement non | Tracking, reporting, quarantaines centralisées, listes Safe-/Blocklists, configuration propre | Visibilité et services de quarantaine centralisés affectés |
| Email Threat Defense | dépend de l’intégration | Télémétrie, investigation et politiques côté cloud | Analyse ou remédiation supplémentaire affectée |
| Système de messagerie | avant ou après la passerelle | Boîtes aux lettres, files de transport, connecteurs | Accès utilisateur ou livraison de bout en bout affectés |

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-cisco.svg?v=20260813" title="Interaktive Infografik: Cisco Secure Email mit SEG-Mailpipeline, Listener, HAT und RAT, Work Queue, Delivery, SMA-Diensten, Konfigurationscluster und Admin-Kontrollpunkten" loading="lazy">
  <a href="/images/kb-interaktiv-cisco.svg?v=20260813">Ouvrir directement le graphique interactif</a>.
</iframe>

## Réception : listener, HAT et RAT

Un **listener** lie SMTP à une interface IP et constitue la première limite de politique. Les listeners publics acceptent généralement le trafic Internet pour les domaines locaux ; les listeners privés reçoivent les messages sortants depuis des réseaux contrôlés. Ces rôles relèvent de la configuration et ne constituent pas une propriété de confiance intrinsèque du port. Un listener privé avec une autorisation de relais trop large est un open relay, même s’il porte un nom interne.

La **Host Access Table**, HAT, attribue les hôtes qui se connectent à des Sender Groups. Leurs Mail Flow Policies déterminent notamment si une connexion est acceptée, rejetée, limitée ou traitée sans certaines analyses. La **Recipient Access Table**, RAT, définit les domaines de destinataires locaux pour les messages entrants. En option, [LDAP](/kb/ldap) vérifie des destinataires précis durant la session SMTP ou ultérieurement dans la file d’attente de travail ; SMTP Call-Ahead peut également interroger le serveur en aval. Cisco distingue ainsi quatre identités qui ne doivent pas être confondues lors d’un incident : IP source, expéditeur d’enveloppe, destinataire d’enveloppe et identités des en-têtes ([Cisco: Email Pipeline, Incoming](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html)).

Une documentation de listener complète contient au minimum, pour chaque direction, l’IP et le port de liaison, les réseaux sources autorisés, les noms EHLO attendus, le mode TLS, l’exigence de certificat client, l’ordre HAT, les domaines RAT, la vérification des destinataires, la taille maximale des messages, les limites de débit et le profil de bounce. L’ordre est particulièrement critique : un rejet HAT précoce ne génère pas d’enregistrement Message-ID comme un message accepté ultérieurement ; le helpdesk ne peut donc pas le trouver avec la même recherche.

Après l’acceptation SMTP commence le traitement effectif du contenu et des politiques. Son ordre est important, car un résultat précoce peut influencer les contrôles ultérieurs, les groupes de destinataires ou les chemins de livraison.

## File d’attente de travail : ordre, splintering et politiques

Après son acceptation, le message entre dans la **Work Queue**. Cisco y documente le routage et le masquage, les Message Filters, les Safe-/Blocklists, l’anti-spam, l’antivirus, Graymail, la réputation et l’analyse des fichiers, les Content Filters, les Outbreak Filters et les quarantaines. L’ordre fait partie du modèle de sécurité. Une modification de politique ne s’applique généralement pas rétroactivement aux messages déjà stockés ; l’activation ultérieure d’un scanner ne corrige donc pas automatiquement un contournement antérieur ([Cisco: Email Pipeline, Work Queue](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html)).

Les **Message Filters** fonctionnent avant la Mail Policy liée au destinataire et peuvent modifier, archiver, mettre en quarantaine, rejeter ou supprimer des messages selon l’enveloppe, les en-têtes, le contenu, les pièces jointes ou les données de connexion. AsyncOS peut ensuite **splinter** un message avec plusieurs destinataires : des Message IDs distincts et, par conséquent, différents états finaux sont créés pour les différentes politiques de destinataires. Une valeur Inject ou ICID unique d’origine peut ainsi se ramifier vers plusieurs MID et résultats de livraison. Le tracking doit afficher cet arbre, pas uniquement rechercher par objet.

Les **Mail Policies** contrôlent les analyses et filtres de contenu liés au destinataire ou à l’expéditeur. Selon Cisco, DLP est limité au traitement sortant. Les licences, l’état de mise à jour des moteurs et la connectivité cloud déterminent en outre quels contrôles sont réellement effectués. Pour chaque politique, un test fiable requiert un cas positif inoffensif, un cas négatif ciblé et l’état final attendu : livraison, modification, quarantaine, suppression ou bounce.

## Livraison : SMTP Routes, Destination Controls et file d’attente

Dans la phase de livraison, AsyncOS sélectionne la route, l’interface source et la destination. Les **SMTP Routes** remplacent la résolution MX normale pour les domaines configurés ; les **Destination Controls** limitent les connexions parallèles et les destinataires par destination. Les Virtual Gateways peuvent fournir différentes adresses IP source, noms d’hôte et files de livraison. Ces paramètres influencent la réputation, SPF, l’allowlisting des pairs et l’emplacement où un message attend ([Cisco: Email Pipeline, Delivery](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html), [RFC 7208](https://datatracker.ietf.org/doc/html/rfc7208)).

Le TLS sortant est hop-by-hop. AsyncOS peut utiliser [STARTTLS](/kb/tls) avec les pairs ; le succès protège ce tronçon de transport, mais ne dit rien des hops précédents ou suivants. Pour les politiques forcées, les modèles de destination, la vérification du certificat, le lien avec le nom et le comportement en cas d’erreur doivent être documentés. Le TLS opportuniste peut revenir au clair en cas d’échec du handshake ; une politique obligatoire doit au contraire mettre le message en file d’attente ou échouer ([Cisco: Verify and Troubleshoot TLS Certificates](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118844-technote-esa-00.html), [RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207)).

L’âge de la file d’attente est plus important que sa longueur seule. Un volume élevé peut être sain lorsque le débit est important ; quelques messages très anciens indiquent un blocage persistant de destination, DNS, TLS ou politique. Pour chaque route, l’image de l’incident doit inclure le message le plus ancien, le motif de nouvelle tentative, le prochain essai, la réponse de destination et le pair responsable.

Dès que l’ESA a transmis ou mis un message en quarantaine, une partie de la visibilité administrateur se déplace vers le SMA. Celui-ci ne remplace toutefois pas les données locales de file d’attente et de système de l’ESA.

## SMA : tracking, reporting et quarantaines

Le SMA collecte des données de tracking et de reporting de plusieurs nœuds SEG. Le Message Tracking peut afficher des états finaux tels que `Delivered`, `Dropped`, `Bounced`, `Quarantined`, `Queued`, `Processing` et `Splintered`. Il s’agit cependant d’un index dérivé : si des données d’export manquent, qu’un service est retardé ou que le message est hors rétention, un résultat vide ne prouve pas que le message n’a jamais été traité. La preuve primaire reste les Mail Logs correspondants et la chaîne MID sur le SEG ([Cisco: Tracking Messages](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0100.html)).

La quarantaine antispam et les quarantaines de politiques, virus et épidémies sont des services distincts avec des utilisateurs, chemins de libération et rétentions différents. Les quarantaines centralisées stockent les messages sur le SMA derrière le pare-feu et peuvent être incluses dans sa sauvegarde standard. À 75, 85 et 95 % d’occupation, AsyncOS génère des alertes de seuil documentées. Lorsqu’un service de quarantaine centralisé devient inaccessible, l’exploitation doit disposer d’une décision testée à l’avance : mettre temporairement en file d’attente, modifier le traitement local ou désactiver de manière contrôlée la politique associée ([Cisco: Centralized Quarantines](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0110.html), [Cisco: Centralizing Services](https://www.cisco.com/c/en/us/td/docs/security/esa/esa13-7-0/user_guide/b_ESA_Admin_Guide_13-7/b_ESA_Admin_Guide_12_1_chapter_0101011.html)).

## Un cluster de configuration n’est pas de la HA de messagerie

AsyncOS peut relier plusieurs passerelles dans un **cluster de configuration** peer-to-peer. Les paramètres peuvent être maintenus au niveau du cluster, du groupe ou de la machine ; il n’existe pas de nœud de cluster primaire. Les membres doivent utiliser une version AsyncOS compatible et communiquent par SSH ou Cluster Communication Service. Le cluster réplique la configuration, pas les sessions SMTP actives, le contenu des files d’attente, les quarantaines locales ou la progression des livraisons ([Cisco: Centralized Management Using Clusters](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0101000.html)).

Il existe donc trois mécanismes distincts :

- **Répartition du trafic :** plusieurs cibles MX, load balancers ou basculement de smarthost ;
- **Cohérence de configuration :** cluster AsyncOS avec des overrides clairs au niveau cluster, groupe et machine ;
- **Disponibilité des données :** état de file d’attente par SEG ainsi que données de tracking et de quarantaine sur le SMA.

La perte d’un nœud après une acceptation SMTP positive peut affecter des messages présents uniquement dans sa file d’attente locale. L’expéditeur ne doit pas simplement les renvoyer tant que l’état de livraison initial n’est pas clair ; des doublons seraient sinon créés. Un test de récupération doit donc non seulement charger la configuration, mais aussi suivre des messages de test acceptés lors d’une panne contrôlée de nœud.

## Pile technologique et surfaces d’administration

AsyncOS est une plateforme d’appliance fermée. Cisco publie des avis open source pour les composants fournis, mais aucun plan complet du code source ou des langages des services propriétaires. Des affirmations telles que « écrit en Python » ou « basé sur FreeBSD » ne constituent pas des informations d’exploitation fiables sans preuve du fabricant spécifique à la version. Pour les administrateurs, la pile vérifiable suivante est plus pertinente ([Cisco: Open Source Used in AsyncOS](https://www.cisco.com/c/dam/en_us/about/doing_business/open_source/docs/CiscoEmailSecurityAppliance-AsyncOS1253-1688323737.pdf)) :

| Couche | Technologie vérifiable | Pertinence opérationnelle |
|---|---|---|
| Transport de messagerie | Listener SMTP, réception, Work Queue, Delivery Queue | Limite d’acceptation, ordre des politiques, nouvelle tentative et bounce |
| Politiques et analyse | HAT/RAT, Message Filters, Mail Policies, Content Filters, moteurs d’analyse | Ordre, licences, mises à jour des moteurs, splintering |
| Données et recherche | Files d’attente/quarantaines locales ; tracking, reporting et quarantaines centralisées SMA | Capacité, rétention, sauvegarde et protection des données |
| Administration | GUI HTTPS, CLI interactive via SSH, configuration XML | Changement, commit, export, restauration et audit |
| Automatisation | API RESTful AsyncOS avec Swagger | Reporting, tracking et accès aux quarantaines ; pas de configuration complète non vérifiée |
| Télémétrie | Mail Logs, autres Log Subscriptions, Syslog, alertes, SNMP/MIB, API | Corrélation via ICID/MID/DCID et état des ressources |
| Plateforme | Appliance matérielle, virtuelle et cloud | Responsabilité du calcul, du stockage, du réseau et du cycle de vie |

Les modifications CLI suivent un modèle transactionnel : les commandes modifient d’abord une configuration en cours, `commit` l’active, `clearchanges` l’annule. Un runbook doit indiquer le dialogue complet et le mode de configuration ; de simples fragments de copier-coller sont dangereux en raison des différences entre versions et clusters. L’API REST fournit un accès authentifié de manière sûre aux rapports, compteurs, données de tracking et de quarantaine ; son interface Swagger locale documente l’étendue réellement installée de l’API ([Cisco: AsyncOS API](https://docs.ces.cisco.com/docs/api), [Cisco: SEG Support Documentation](https://www.cisco.com/c/en/us/support/security/email-security-appliance/series.html)).

## Dépendances réseau, identité et temps

Une fois le pipeline de messagerie établi, ses connexions externes peuvent être vérifiées. Chaque ligne indique quel système initie une connexion, à quoi elle sert et comment une erreur se manifeste.

| Connexion | Port habituel | Initiateur | Objectif et symptômes d’erreur |
|---|---:|---|---|
| SMTP | TCP 25 | MTA externe, système de messagerie interne ou SEG | Acceptation et transfert ; timeout, 4xx/5xx, croissance de la file d’attente |
| HTTPS | TCP 443 ou configuré | Administrateur, utilisateur final ou client API | GUI, API, quarantaine ; vérifier séparément certificat, SSO et rôles |
| SSH | TCP 22 ou configuré | Administrateur ou membre SEG | CLI et communication de cluster facultative |
| CCS | TCP 2222 par défaut, configurable | Membre SEG | Cluster de configuration ; aucun flux de messagerie |
| DNS | UDP/TCP 53 | SEG/SMA | MX, A/AAAA, PTR, réputation et mises à jour |
| LDAP/LDAPS | TCP 389/636 | SEG/SMA | Destinataires, routage, groupes et authentification administrateur |
| Syslog | UDP/TCP 514 ou TLS 6514 selon la conception | SEG/SMA | Transport de journaux externe ; définir le modèle de perte et de backpressure |
| SNMP | UDP 161/162 | Supervision ou appliance | Interrogation d’état et traps ; préférer SNMPv3 |

Les numéros de port seuls ne prouvent aucune fonction active. Les attributions proviennent du [IANA Service Name and Port Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml); Cisco documente CCS et sa configurabilité dans le chapitre consacré aux clusters. Les pare-feu doivent inclure la source, la destination, le sens, le protocole, les exigences TLS ou d’authentification et l’objectif métier.

[LDAP](/kb/ldap) peut alimenter l’acceptation des destinataires, le routage, l’appartenance aux groupes et l’authentification administrateur. Ces requêtes ont des schémas, timeouts et conséquences d’erreur différents. Si Recipient Acceptance échoue, le système peut, selon la configuration, générer un bounce différé ou supprimer le message ; une erreur d’authentification de la GUI ne prouve donc pas un défaut de vérification SMTP des destinataires. Les comptes de service, Base DN, filtres, comportement de referral, chaîne de certificats et ordre de basculement doivent être documentés par requête ([Cisco: Email Pipeline, LDAP Recipient Acceptance](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html), [RFC 4511](https://datatracker.ietf.org/doc/html/rfc4511)).

Le DNS et une heure correcte sont des dépendances système. La résolution MX et d’hôtes contrôle la livraison et l’accessibilité du cluster ; PTR et réputation influencent la classification. NTP rend les heures des journaux, Received et tracking corrélables. Cisco exige, pour les clusters, des noms d’hôte résolubles ou des adresses IP utilisées de manière cohérente et décrit l’heure système ainsi que NTP comme faisant partie de la configuration de base ([Cisco: Setup and Installation](https://www.cisco.com/c/en/us/td/docs/security/esa/esa13-7-0/user_guide/b_ESA_Admin_Guide_13-7/b_ESA_Admin_Guide_12_1_chapter_010.html)).

En cas d’incident, le même chemin est suivi à l’envers : état de livraison, décision de la file d’attente de travail, politique de réception, listener et dépendances réseau.

## Supervision et triage des incidents

La question centrale est : **La passerelle a-t-elle accepté le message, l’a-t-elle traité et à quel hop l’a-t-elle remis ?** À cette fin, les identifiants de connexion, de message et de livraison des Mail Logs sont chaînés. Le Message Tracking sur le SMA accélère la recherche, mais ne remplace pas les journaux bruts. Les signaux techniques utiles sont :

- taux d’acceptation, réponses 4xx/5xx et connexions rejetées par listener et Sender Group ;
- files d’attente de travail et de livraison, âge du message le plus ancien ainsi que réponses de destination récurrentes ;
- durée de traitement et de splintering, erreurs des moteurs d’analyse et ancienneté des mises à jour ;
- occupation des quarantaines locales et centralisées, événements de libération et de suppression ;
- valeur Resource Conservation, CPU, mémoire, occupation disque et alertes critiques ;
- disponibilité et latence de DNS, LDAP, SMA, services de mise à jour et cloud ;
- cohérence du cluster et Machine Overrides involontaires ;
- expiration et utilisation de chaque certificat TLS ainsi que modifications du truststore.

En **Resource Conservation Mode**, AsyncOS limite progressivement l’acceptation afin que la livraison puisse résorber le retard ; en cas de pénurie extrême de ressources, aucun nouveau message n’est accepté. Le symptôme est donc souvent une baisse du débit entrant alors que la cause réelle est une route de destination lente ou une ressource saturée. Cisco fournit l’état et les alertes dans la GUI/CLI ; SNMPv3 et `ASYNCOS-MAIL-MIB` permettent une supervision externe ([Cisco: Resource Conservation](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/117834-qanda-esa-00.html), [Cisco: SNMP Monitoring](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/117831-qanda-esa-00.html)).

## Sauvegarde, récupération et mise à niveau

Un fichier de configuration XML exporté est nécessaire, mais ne constitue pas une sauvegarde système complète. Cisco documente `saveconfig`, `mailconfig` et `loadconfig`; les phrases de passe masquées ne peuvent pas être rechargées. Les certificats et clés, l’état du cluster, les Feature Keys, les files d’attente locales, les quarantaines locales, les données SMA ainsi que les dépendances externes nécessitent leurs propres preuves ([Cisco: System Administration](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0100010.html), [Cisco: Automated Configuration Backup](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118403-technote-esa-00.html)).

| Objet de récupération | Sauvegarde ou reconstruction | Test de réception |
|---|---|---|
| Configuration SEG | Export non masqué, stocké de manière protégée, avec phrases de passe documentées | Charger sur une instance de remplacement, effectuer un diff et tester listener/politique |
| Certificats et clés privées | Sauvegarde chiffrée des clés, chaîne CA et matrice de rôles | Handshake HTTPS et SMTP-TLS avec vérification du nom |
| File d’attente locale | Normalement non reconstructible depuis une sauvegarde de configuration | Panne de nœud avec e-mail de test accepté et contrôle des doublons |
| Données SMA | Sauvegarde SMA pour tracking, reporting, quarantaines et listes | Vérifier la recherche, la libération d’un e-mail de test et la rétention |
| Cluster | Export à chaque niveau avec Machine Overrides documentés | Reconnecter le membre et contrôler la cohérence |
| Services externes | Configuration DNS, LDAP, Syslog, NTP, mise à jour et cloud | Vérification synthétique de bout en bout |

Les mises à niveau sont des migrations d’appliance. Au préalable, il faut vérifier le chemin cible, les états intermédiaires compatibles, les exigences d’hyperviseur ou de cloud, les modifications de fonctionnalités, l’ordre de cluster, l’espace libre, l’indisponibilité et la limite de rollback. Les catégories de version GD et MD de Cisco ne constituent pas une recommandation automatique pour chaque environnement ; les Security Advisories, la matrice de support et l’étendue de politiques propre à l’environnement testée sont déterminantes. La page de support et les explications de cycle de vie doivent faire partie du processus de patch, et non figurer comme numéro de version statique dans l’article ([Cisco: SEG Release Notes](https://www.cisco.com/c/en/us/support/security/email-security-appliance/products-release-notes-list.html), [Cisco: Software Lifecycle Support Statement](https://www.cisco.com/c/dam/en/us/td/docs/security/esa/lifecycle_support_statement/Secure_Email_Gateway_Software_Lifecycle_Support_Statement.pdf)).

## Outils de diagnostic

La recherche d’erreur suit le chemin du message de l’extérieur vers l’intérieur. Les noms et l’accessibilité sont vérifiés d’abord, puis l’acceptation SMTP, les événements du pipeline, la file d’attente et, le cas échéant, l’évaluation SMA.

### DNS, MX et résolution de destination

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cisco-Secure-Email-DNS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.ch
Resolve-DnsName seg1.example.ch -Type A,AAAA
Resolve-DnsName 192.0.2.25 -Type PTR
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig +short MX example.ch
dig +short A seg1.example.ch
dig +short AAAA seg1.example.ch
dig +short -x 192.0.2.25
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) et [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) affichent la résolution MX, directe et inverse. La requête doit être répétée depuis la perspective des résolveurs internes et externes ; les SMTP Routes AsyncOS peuvent remplacer le résultat MX visible.

### TCP et SMTP-TLS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cisco-Secure-Email-TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection seg1.example.ch -Port 25 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://seg1.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz seg1.example.ch 25
openssl s_client -starttls smtp -connect seg1.example.ch:25 \
  -servername seg1.example.ch -showcerts
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) et [`nc`](https://man.openbsd.org/nc) attestent uniquement le chemin TCP. [`curl`](https://curl.se/docs/manpage.html) et [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) demandent STARTTLS et affichent le handshake ainsi que la chaîne de certificats ; seule la vérification attendue du nom et de la confiance atteste la politique TLS configurée.

### Message de test SMTP autorisé

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cisco-Secure-Email-SMTP-Test">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
curl.exe --verbose --ssl-reqd --url smtp://seg1.example.ch:25 `
  --mail-from test-sender@example.ch `
  --mail-rcpt test-recipient@example.net `
  --upload-file .\seg-test.eml
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
swaks --server seg1.example.ch --port 25 --tls \
  --from test-sender@example.ch --to test-recipient@example.net \
  --data seg-test.eml
```

  </div>
</div>

[`curl`](https://curl.se/docs/manpage.html) et [`swaks`](https://jetmore.org/john/code/swaks/) envoient un message de test contrôlé. L’expéditeur, le destinataire et la destination doivent être autorisés. Il faut consigner la réponse SMTP finale, l’ICID/MID, les MID splinter, la politique, l’état de quarantaine ou de livraison et l’arrivée effective.

### Interface Web et API REST

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cisco-AsyncOS-API-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Invoke-WebRequest -Method Head -Uri https://sma.example.ch/
Invoke-WebRequest -Method Head -Uri https://seg1.example.ch/swagger
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
curl --head --verbose https://sma.example.ch/
curl --head --verbose https://seg1.example.ch/swagger
```

  </div>
</div>

[`Invoke-WebRequest`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest) et [`curl`](https://curl.se/docs/manpage.html) vérifient l’accessibilité HTTP et TLS. Un code d’état ne prouve ni la connexion, ni l’autorisation de rôle, ni l’importation de tracking ou le fonctionnement de la quarantaine. La page Swagger décrit uniquement l’API de l’instance interrogée ; les tests API de production utilisent un compte minimal en lecture seule et ne stockent aucun token dans l’historique du shell.

### Chemin des paquets sur un point de mesure autorisé

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cisco-Secure-Email-Paketerfassung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
pktmon filter remove
pktmon filter add SEG-SMTP -p 25
pktmon start --capture --pkt-size 0 --file-name seg.etl
pktmon stop
pktmon pcapng seg.etl -o seg.pcapng
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
tcpdump -ni any -s 0 -w seg.pcap 'tcp port 25 or tcp port 443'
```

  </div>
</div>

[`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) et [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) ne voient que le trafic au point de mesure sélectionné. Un client administrateur n’observe pas automatiquement le chemin entre le load balancer, le SEG, le SMA et le MTA de destination. Les données de paquets peuvent contenir du contenu SMTP avant STARTTLS et des métadonnées personnelles ; elles doivent être protégées en conséquence.

## Histoire technique

IronPort Systems a développé des passerelles de messagerie spécialisées et la gamme de produits AsyncOS. Cisco a annoncé l’acquisition de l’entreprise en janvier 2007 et a intégré ses technologies de sécurité e-mail et Web à son propre portefeuille de sécurité ([Cisco: Agreement to Acquire IronPort](https://newsroom.cisco.com/c/r/newsroom/en/us/a/y2007/m01/cisco-announces-agreement-to-acquire-ironport.html)). Les noms de produits ont ensuite évolué de Cisco IronPort Email Security Appliance à Cisco Email Security Appliance, puis à **Cisco Secure Email Gateway** ; des termes historiques tels que ESA, C-Series et M-Series restent visibles dans les runbooks, les messages de journal, les licences et les chemins de documentation.

L’idée architecturale est restée reconnaissable malgré ces renommages : une passerelle spécialisée avec le pipeline à trois étapes Receipt, Work Queue et Delivery, ainsi qu’un système de gestion séparé pour les données agrégées et les quarantaines. Des appliances virtuelles et de cloud public, Cloud Gateway, des API REST et des services d’analyse basés sur le cloud ont été ajoutés ultérieurement. Email Threat Defense étend le portefeuille avec des modèles API, journalisation et passerelle ; il ne modifie pas rétroactivement les limites d’état d’une installation ESA/SMA existante ([Cisco: Secure Email Data Sheet](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/datasheet-c78-742868.html), [Cisco: Email Threat Defense](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/secure-email-threat-defense-ds.html)).

Le nom d’une appliance installée ne suffit donc pas comme information de cycle de vie. Le modèle matériel, la plateforme virtuelle, la branche AsyncOS, les licences activées, les mises à jour de moteurs et de règles ainsi que les services cloud dépendants ont leurs propres cycles de vie. Les pages de support, de versions et de fin de vie de Cisco sont des sources opérationnelles dynamiques ; un article statique devrait y renvoyer, mais ne pas figer une prétendue version durablement actuelle ([Cisco: SEG End-of-Life Notices](https://www.cisco.com/c/en/us/products/security/email-security-appliance/eos-eol-notice-listing.html), [Cisco: SEG Support](https://www.cisco.com/c/en/us/support/security/email-security-appliance/series.html)).

## Sources

- [Cisco – Guide utilisateur AsyncOS : compréhension du pipeline e-mail](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html)
- [IETF RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [Cisco – Guide utilisateur SMA : suivi des messages](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0100.html)
- [Cisco – Guide utilisateur SMA : quarantaines centralisées](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0110.html)
- [Cisco – Open Source utilisé dans Email Security Appliance AsyncOS](https://www.cisco.com/c/dam/en_us/about/doing_business/open_source/docs/CiscoEmailSecurityAppliance-AsyncOS1253-1688323737.pdf)
- [Cisco – API AsyncOS](https://docs.ces.cisco.com/docs/api)
- [Cisco – Guide utilisateur AsyncOS : gestion centralisée avec des clusters](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0101000.html)
- [Cisco – Fiche technique Secure Email Gateway et Secure Email and Web Manager](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/datasheet-c78-742868.html)
- [Cisco – Fiche technique Secure Email Threat Defense](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/secure-email-threat-defense-ds.html)
- [IETF RFC 7208 – Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Cisco – Vérifier et dépanner les certificats TLS sur ESA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118844-technote-esa-00.html)
- [IETF RFC 3207 – Extension de service SMTP pour SMTP sécurisé via TLS](https://datatracker.ietf.org/doc/html/rfc3207)
- [Cisco – Centralisation des services sur un Secure Email and Web Manager](https://www.cisco.com/c/en/us/td/docs/security/esa/esa13-7-0/user_guide/b_ESA_Admin_Guide_13-7/b_ESA_Admin_Guide_12_1_chapter_0101011.html)
- [Cisco – Documentation de support Secure Email Gateway](https://www.cisco.com/c/en/us/support/security/email-security-appliance/series.html)
- [IANA – Registre des noms de services et numéros de port de protocoles de transport](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)
- [IETF RFC 4511 – Lightweight Directory Access Protocol](https://datatracker.ietf.org/doc/html/rfc4511)
- [Cisco – Configuration et installation](https://www.cisco.com/c/en/us/td/docs/security/esa/esa13-7-0/user_guide/b_ESA_Admin_Guide_13-7/b_ESA_Admin_Guide_12_1_chapter_010.html)
- [Cisco – Mode Resource Conservation](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/117834-qanda-esa-00.html)
- [Cisco – Supervision SNMP sur ESA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/117831-qanda-esa-00.html)
- [Cisco – Guide utilisateur AsyncOS : administration système](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0100010.html)
- [Cisco – Automatiser la sauvegarde de configuration d’un ESA en cluster](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118403-technote-esa-00.html)
- [Cisco – Notes de version Secure Email Gateway](https://www.cisco.com/c/en/us/support/security/email-security-appliance/products-release-notes-list.html)
- [Cisco – Déclaration de support du cycle de vie logiciel Secure Email Gateway](https://www.cisco.com/c/dam/en/us/td/docs/security/esa/lifecycle_support_statement/Secure_Email_Gateway_Software_Lifecycle_Support_Statement.pdf)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [ISC BIND 9 – page de manuel dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – page de manuel nc](https://man.openbsd.org/nc)
- [curl – page de manuel en ligne de commande](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [swaks – outil de test SMTP](https://jetmore.org/john/code/swaks/)
- [Microsoft Learn – Invoke-WebRequest](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest)
- [Microsoft Learn – pktmon](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [tcpdump – page de manuel](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [Cisco – Accord d’acquisition d’IronPort](https://newsroom.cisco.com/c/r/newsroom/en/us/a/y2007/m01/cisco-announces-agreement-to-acquire-ironport.html)
- [Cisco – Avis de fin de vie de Secure Email Gateway](https://www.cisco.com/c/en/us/products/security/email-security-appliance/eos-eol-notice-listing.html)
