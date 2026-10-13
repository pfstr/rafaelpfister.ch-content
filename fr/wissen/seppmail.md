---
title: "SEPPmail : passerelle de messagerie, clés et livraison GINA"
blatt: "seppmail"
description: "SEPPmail pour les administrateurs de messagerie : architecture SMTP inline, pipeline Postfix et ruleset, S/MIME, OpenPGP et GINA, état des clés et certificats, intégration d’annuaire, cluster, supervision, sauvegarde et restauration."
fakten:
  - label: Produit
    wert: SEPPmail Secure E-Mail Gateway
    href: https://www.seppmail.com/products/
  - label: Rôle système
    wert: Passerelle SMTP sur le chemin des messages
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Cœur de transport
    wert: Postfix MTA
    href: https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html
  - label: Base système
    wert: Stack d’appliance basé sur OpenBSD
    href: https://docs.seppmail.com/ch/20_revision-history.html
  - label: Cryptographie des messages
    wert: S/MIME · OpenPGP · clés de domaine
    href: https://docs.seppmail.com/ch/07_mi_07_mp_01_ep_01_addedit-encryption-policy.html
  - label: Sans clé de destinataire
    wert: Livraison Web GINA via HTTPS
    href: https://docs.seppmail.com/en/07_mi_17_gd__gina-domains.html
  - label: Modèle de règles
    wert: ruleset généré · politiques · commandes personnalisées
    href: https://docs.seppmail.com/ch/08_cr_01_gi_04_macros.html
  - label: Annuaire
    wert: LDAP/LDAPS pour les requêtes de clés, d’identité et de règles
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Interfaces Web
    wert: GINA 443 · Admin 8443 · REST 8445
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Supervision
    wert: Postmaster · SNMP · NRPE · Syslog
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Transport de cluster
    wert: SSH entre appliances · NTP requis
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Origine
    wert: Développement produit suisse depuis 2001
    href: https://www.seppmail.com/de/die-seppmail-story/
werbung:
  - tools
  - newsletter
ctaThemen:
  - seppmail
translationSourceHash: f01b161a8f003ca453398dd3d393ca671b7346ad63164cd311dabb11055526fe
translationModel: gpt-5.6-terra
translatedAt: 2026-10-12T11:47:33.616Z
translationReview: automatic
---

# SEPPmail : passerelle de messagerie, clés et livraison GINA

SEPPmail est une passerelle de messagerie spécialisée qui reçoit les messages dans le chemin SMTP, les traite selon des règles centralisées et les transmet à un système de messagerie interne, un smarthost ou le prochain correspondant Internet. Les modes de traitement documentés incluent la signature ou le chiffrement S/MIME et OpenPGP, le chiffrement de domaine basé sur S/MIME, les exigences TLS liées à la connexion et la livraison Web GINA pour les destinataires ne disposant pas de clé publique utilisable ([SEPPmail: Products](https://www.seppmail.com/products/), [SEPPmail: Mail System](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html), [SEPPmail: Encryption Policy](https://docs.seppmail.com/ch/07_mi_07_mp_01_ep_01_addedit-encryption-policy.html)).

Le traitement centralisé simplifie le client : les utilisateurs peuvent envoyer depuis leur système de messagerie habituel, tandis que la passerelle recherche les clés, évalue les politiques et exécute les opérations cryptographiques. L’appliance devient ainsi aussi une frontière de confiance et de disponibilité. Elle voit le contenu du message avant le chiffrement sortant ou après le déchiffrement entrant, conserve les éléments de clé privée et peut devenir le nouveau point de responsabilité du message après une acceptation SMTP réussie. Un administrateur doit donc non seulement démontrer que le « chiffrement est actif », mais aussi pouvoir retracer l’ensemble du chemin composé du routage, de la règle, du choix de clé, du mode de livraison, de la file d’attente, de la journalisation et de la restauration ([RFC 5321: SMTP responsibility](https://datatracker.ietf.org/doc/html/rfc5321#section-6.1), [RFC 8551: S/MIME](https://datatracker.ietf.org/doc/html/rfc8551), [SEPPmail: Frontend/Backend Cluster](https://docs.seppmail.com/en/04_com_09_cl_05_frontend-backend-cluster.html)).

L’explication suit un message depuis son acceptation SMTP, le ruleset et la méthode de chiffrement choisie jusqu’à la livraison ou au portail GINA. Les clés, le cluster, le diagnostic et la restauration sont ensuite expliqués à partir de ce chemin.

SEPPmail se situe sur le chemin du message comme passerelle à état. Il accepte SMTP, détermine la politique et la capacité du destinataire, sélectionne S/MIME, OpenPGP, TLS ou GINA, puis transmet le message. L’architecture, l’exploitation et la restauration suivent ce processus.

## Approche architecturale : passerelle inline à état

Dans la configuration la plus simple, le système de messagerie interne envoie les messages sortants par [SMTP](/kb/smtp) à SEPPmail ; pour les messages entrants, l’[enregistrement MX](/kb/dns#dns-im-mail--und-identitätsbetrieb) public pointe directement ou via un smarthost en amont vers la passerelle. Un serveur de transfert interne est défini pour chaque domaine géré. En sortie, l’appliance peut effectuer elle-même la livraison via DNS ou utiliser un smarthost SMTP défini. La matrice de pare-feu documentée par le fabricant représente ces directions séparément et mentionne TCP 25 entre le serveur de messagerie, l’appliance, Internet ou le smarthost ([SEPPmail: Setting Up The Firewall/Router](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html), [SEPPmail: Mail System](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

La passerelle n’est pas un filtre de paquets transparent. Elle termine une session SMTP et en ouvre une nouvelle pour la transmission. Après `250` à la fin de la phase DATA, la responsabilité de livraison revient au système SMTP qui accepte le message ; l’établissement réussi d’une connexion ou `STARTTLS` seul ne prouve pas ce transfert ([RFC 5321, section 4.1.1.4](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4), [RFC 3207: SMTP STARTTLS](https://datatracker.ietf.org/doc/html/rfc3207)). Du point de vue de l’exploitation, il en résulte au moins cinq objets de contrôle distincts : la session SMTP entrante, le traitement local, le prochain saut SMTP, l’état des clés et des politiques ainsi que, le cas échéant, l’état Web GINA.

Pour Exchange Online, SEPPmail décrit un modèle explicite de connecteurs et de règles de flux de messagerie. Dans les environnements multilocataires, le guide du fabricant recommande des connecteurs basés sur des certificats ; plusieurs hôtes de passerelle doivent en outre être pris en compte dans les connecteurs Microsoft et, le cas échéant, dans la configuration antispam. Microsoft documente également les connecteurs comme une relation définie de confiance et de routage, et non comme la désactivation générale de tous les autres chemins de livraison ([SEPPmail: Setup with Exchange Online](https://docs.seppmail.com/en/04_com_powershell_setup-exo.html), [Microsoft: Configure mail flow using connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow)).

## Stack technologique et états persistants

L’historique des révisions accessible publiquement documente un stack d’appliance basé sur OpenBSD, avec Postfix comme MTA ainsi que, entre autres, Apache HTTP Server, OpenSSL ou LibreSSL, GnuPG, OpenLDAP, Dovecot, Fetchmail, ClamAV, OpenSSH et Perl. Ces composants remplissent différents rôles ; les versions mentionnées dans une note de version donnée ne sont pas des caractéristiques intemporelles du produit. Pour l’exploitation, l’essentiel est de savoir quelle interface est prise en charge par le fabricant : les paramètres SMTP et TLS sont par exemple gérés via « Mail System », tandis que les « Extended Postfix MTA settings » sont explicitement décrits comme risqués et non synchronisés dans le cluster ([SEPPmail: Revision History](https://docs.seppmail.com/ch/20_revision-history.html), [SEPPmail: Mail System – Extended Postfix settings](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

Le flux de messagerie visible n’est qu’une partie de l’état. Pour un inventaire fiable, les objets suivants doivent au minimum être répertoriés séparément :

| Espace d’état | Exemples | Pourquoi il est vérifié séparément |
|---|---|---|
| SMTP et routage | Managed Domains, Forwarding Server, Smarthost, réseaux de relais, cibles TLS | détermine l’acceptation et le prochain saut |
| Traitement | ruleset généré, Encryption Policies, Custom Commands, déclencheurs d’en-tête et d’objet | détermine la signature, le chiffrement, GINA, le rebond ou le chemin en clair |
| Cryptographie | clés privées S/MIME/OpenPGP, clés publiques de destinataires, confiance CA, OCSP/CRL, clés de domaine | détermine si et pour qui une opération est possible et vérifiable |
| GINA | domaines, mise en page, comptes, mots de passe, historique de livraison, messages et clés stockés | constitue un chemin supplémentaire HTTPS, d’identité et de conservation |
| Plateforme | configuration de l’appliance, nom système, réseau, certificats, licences, rôle de cluster | détermine l’accessibilité, l’administration et le redémarrage |
| Observabilité | journaux de messagerie, système et audit, messages Postmaster, SNMP/NRPE/Syslog | fournit des preuves et des alertes, mais n’est pas entièrement inclus dans la sauvegarde |

Cette répartition suit la structure documentée des menus et composants, la gestion des comptes GINA, la description du cluster et le Quick Setup Guide. Il s’agit d’un modèle d’inventaire pour les administrateurs, et non d’une affirmation sur une structure de base de données interne non documentée ([SEPPmail: Mail System](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html), [SEPPmail: GINA User Details](https://docs.seppmail.com/en/07_mi_18_ga_01_user-x.html), [SEPPmail: Frontend/Backend Cluster](https://docs.seppmail.com/en/04_com_09_cl_05_frontend-backend-cluster.html), [SEPPmail: Quick Setup Guide](https://downloads.seppmail.com/wp-content/uploads/docu/Quick-Setup-Guide-en.pdf)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-seppmail.svg?v=20260813" title="Interaktive Infografik: SEPPmail-SMTP-Pfad, Ruleset, Verschlüsselungsverfahren, GINA, Schlüsselzustand, Cluster, Monitoring und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-seppmail.svg?v=20260813">Ouvrir le graphique interactif directement</a>.
</iframe>

## Acceptation SMTP, routage et limite de relais

Un Managed Domain associe un espace de noms SMTP accepté au Forwarding Server interne et à d’autres paramètres spécifiques au domaine. La passerelle peut éventuellement vérifier l’adresse du destinataire par une recherche SMTP auprès de ce serveur. Le chemin sortant est déterminé soit par le MTA intégré via DNS, soit par un serveur SMTP configuré ; des paramètres TLS et de routage basé sur l’expéditeur spécifiques au domaine peuvent remplacer ce chemin par défaut ([SEPPmail: Mail System – Managed Domains and Outgoing Server](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

« Relaying allowed » définit depuis quelles adresses IP ou quels réseaux l’appliance accepte les messages destinés à des destinataires externes. La documentation du fabricant met expressément en garde contre l’ajout global du serveur sortant, car cela peut créer un relais ouvert. L’ouverture de port, l’autorisation de relais, le domaine d’expéditeur géré et, le cas échéant, l’authentification SMTP sont donc des contrôles distincts. Un test TCP positif ne dit rien de la décision de relais après `MAIL FROM` et `RCPT TO` ([SEPPmail: Mail System – Relaying](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html), [RFC 5321: SMTP commands](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1)).

Avec plusieurs passerelles, l’équilibreur de charge en amont ou le service émetteur doit pouvoir joindre chaque hôte de production. Dans le même temps, les systèmes en aval doivent accepter toutes les identités de passerelle attendues. Dans Exchange Online, cela concerne les certificats de connecteur ou les plages IP ; dans un environnement de cluster, cela concerne aussi les IP virtuelles et la particularité documentée selon laquelle la réception se fait avec un alias IP, mais l’envoi via l’adresse physique du système émetteur ([SEPPmail: Firewall and IP aliases](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html), [SEPPmail: Exchange Online clustering and multi-host](https://docs.seppmail.com/en/04_com_powershell_setup-exo.html)).

Après l’acceptation SMTP, le ruleset décide quel chemin de protection et de livraison s’applique. Cette décision doit rester traçable ultérieurement à partir du message, de l’identité, des données d’annuaire et de la version de la règle.

## Ruleset, politiques et preuve de décision

SEPPmail génère des règles de traitement exécutables à partir des paramètres du Ruleset Generator. Des Encryption Policies supplémentaires peuvent imposer, pour des Managed Domains sélectionnés et des indicateurs optionnels, des procédés tels que S/MIME-only, OpenPGP-only, GINA-only ou une sélection hiérarchique. Après une modification de politique, le ruleset doit selon le fabricant être régénéré avant que la modification ne soit active. « Enregistré dans l’interface graphique » et « effectif dans le chemin actif des messages » sont donc deux états différents ([SEPPmail: Add/Edit Encryption Policy](https://docs.seppmail.com/ch/07_mi_07_mp_01_ep_01_addedit-encryption-policy.html)).

Les Custom Commands étendent ce pipeline avec des conditions et des actions. Le fabricant documente notamment les requêtes LDAP, les indicateurs, la création d’utilisateurs et de clés, les opérations sur les en-têtes, la signature, le chiffrement, la livraison et le rejet. Les macros sont développées lors de la génération du ruleset ; une erreur de syntaxe peut empêcher cette génération ([SEPPmail: Macros](https://docs.seppmail.com/ch/08_cr_01_gi_04_macros.html), [SEPPmail: LDAP-based user creation](https://docs.seppmail.com/en/09_ht_user_creation-with-ldap.html)).

Les déclencheurs d’en-tête ou d’objet sont des entrées dans une décision de sécurité. Ils ne doivent être évalués que depuis une direction de confiance ou être nettoyés avant l’évaluation. Les exemples SEPPmail de signature interclients avertissent expressément de ne pas présupposer l’identité de l’expéditeur sans vérification. Pour chaque déclencheur, la source, le connecteur autorisé, l’authentification, la normalisation, la priorité et le comportement de refus par défaut doivent donc figurer dans la documentation ([SEPPmail: Signing emails between clients](https://docs.seppmail.com/en/09_ht_tenant_signing-e-mails-between-tenants.html), [SEPPmail: Internal X-header processing](https://docs.seppmail.com/en/08_cr_01_gi_06_x-header.html)).

Une preuve de décision reproductible contient au minimum : l’expéditeur et le destinataire de l’enveloppe, les en-têtes `From:` et, le cas échéant, `Sender:` visibles, la direction et le Managed Domain, la politique ou règle déclenchante, les clés trouvées, le procédé choisi, l’action finale et l’ID de file d’attente. Cela est particulièrement important, car l’identité de signature S/MIME est habituellement vérifiée par rapport à l’expéditeur visible, tandis que le routage SMTP travaille d’abord avec l’enveloppe ([SEPPmail: Sender identity selection](https://docs.seppmail.com/en/09_ht_user_creation-based-on-other-criteria-than-from-header.html), [RFC 5322: Originator fields](https://datatracker.ietf.org/doc/html/rfc5322#section-3.6.2)).

## S/MIME, OpenPGP et TLS comme niveaux de protection distincts

[S/MIME](https://datatracker.ietf.org/doc/html/rfc8551) encapsule des contenus MIME signés ou chiffrés avec Cryptographic Message Syntax. Les identités et les clés publiques sont généralement liées par des certificats X.509 ; la vérification de chemin, la période de validité, l’utilisation des clés et les informations de révocation font donc partie de la décision ([RFC 5652: CMS](https://datatracker.ietf.org/doc/html/rfc5652), [RFC 5280: PKIX certificate validation](https://datatracker.ietf.org/doc/html/rfc5280), [RFC 6960: OCSP](https://datatracker.ietf.org/doc/html/rfc6960)). SEPPmail peut gérer des certificats utilisateur et de domaine et les obtenir par des interfaces MPKI ; la documentation du pare-feu indique HTTPS à cet effet, ainsi que HTTP ou HTTPS pour OCSP/CRL ([SEPPmail: Firewall – MPKI and revocation](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html)).

[OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580) utilise un autre modèle de paquets, de clés et de confiance. La présence d’une clé publique ne prouve pas automatiquement qu’elle appartient au destinataire prévu ou qu’elle doit encore être utilisée. L’empreinte, l’origine, l’ID utilisateur, l’expiration, la révocation, l’algorithme et la capacité de déchiffrement privée doivent être traités comme un cycle de vie. L’Encryption Policy distingue PGP/MIME et Inline PGP et peut configurer séparément la signature et le chiffrement ([SEPPmail: Encryption Policy – OpenPGP options](https://docs.seppmail.com/ch/07_mi_07_mp_01_ep_01_addedit-encryption-policy.html), [RFC 9580: Key structures](https://datatracker.ietf.org/doc/html/rfc9580#section-5.5)).

[SMTP-TLS](/kb/tls#tls-policy-bei-smtp) protège une connexion entre deux sauts. Sans politique contraignante, STARTTLS est opportuniste : s’il n’est pas disponible, le MTA émetteur peut livrer sans chiffrement selon sa configuration. SEPPmail peut configurer TLS par cible avec des niveaux, un serveur fixe, une empreinte de certificat serveur ou d’autres exigences. Ce n’est ni S/MIME ni OpenPGP et cela ne protège pas le message contre les MTA impliqués ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207), [RFC 7672: SMTP DANE](https://datatracker.ietf.org/doc/html/rfc7672), [SEPPmail: Mail System – TLS Settings](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

Dans le cadre d’un chiffrement par passerelle, le texte en clair et les clés privées de l’expéditeur se trouvent à l’intérieur de la frontière de confiance SEPPmail. Cela permet des politiques centralisées, mais implique aussi que les rôles administratifs, les sauvegardes, les connexions de cluster, les exportations de débogage, les journaux et les traitements de contenu en aval doivent être protégés comme s’ils pouvaient toucher des données de communication confidentielles ou des clés. La présence effective de contenu dans un export concret doit être vérifiée à l’aide de la documentation du fabricant et d’un test contrôlé.

Lorsqu’aucune clé de destinataire appropriée ou aucun chemin de protection direct n’est disponible, GINA déplace l’accès vers un portail Web. L’identité Web, la session, le stockage et le courriel de notification s’ajoutent alors au chemin SMTP.

## GINA comme service Web avec identité et stockage

GINA est le chemin de repli documenté lorsqu’un message n’est pas livré avec une clé de destinataire appropriée ou qu’une politique impose explicitement GINA. La communication avec le portail s’effectue via HTTPS sur TCP 443. Les GINA Domains contrôlent notamment le nom du portail, la mise en page, le comportement des comptes et mots de passe, la transmission initiale du mot de passe ainsi que les paramètres de conservation ([SEPPmail: GINA Domains](https://docs.seppmail.com/en/07_mi_17_gd__gina-domains.html), [SEPPmail: Firewall – GINA](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html)).

Pour les administrateurs, GINA n’est pas un simple format de présentation, mais un état de service distinct. Un compte GINA possède un statut, des paramètres de mot de passe et, le cas échéant, d’authentification externe, des journaux utilisateur, un historique de messagerie et une clé. La documentation du fabricant avertit que la suppression du compte supprime aussi irrévocablement sa clé et peut rendre les messages GINA restants illisibles ([SEPPmail: GINA User Details](https://docs.seppmail.com/en/07_mi_18_ga_01_user-x.html)).

L’exploitation de GINA comprend donc au minimum :

- un nom d’hôte public stable, conforme au certificat, et son cycle de vie [DNS](/kb/dns) ;
- un certificat HTTPS, une chaîne complète, le renouvellement et l’accessibilité externe ;
- une transmission ou dérivation séparée du secret initial ;
- le verrouillage de compte, la réinitialisation, l’identité externe et le processus de support ;
- des règles de conservation et de suppression des messages, métadonnées, journaux et clés ;
- un test de restauration qui ouvre effectivement un message GINA créé avant la sauvegarde après la restauration.

Le Quick Setup Guide indique qu’une modification du nom d’hôte peut rendre les anciens messages GINA inaccessibles. Il documente en outre qu’une restauration sur un nouveau système nécessite le mot de passe défini lors de la sauvegarde ([SEPPmail: Quick Setup Guide](https://downloads.seppmail.com/wp-content/uploads/docu/Quick-Setup-Guide-en.pdf)). Le nom d’hôte, le certificat, le domaine GINA, le compte et la clé doivent donc être considérés conjointement.

## Annuaire et cycle de vie des clés et certificats

LDAP/LDAPS peut fournir des clés publiques, évaluer des conditions d’utilisateur ou de groupe et limiter la création automatique de comptes ou de clés. L’appliance peut également proposer ses propres points de terminaison LDAP pour la distribution de clés publiques collectées ou internes. Les directions et ports requis diffèrent selon le rôle ; la matrice officielle du pare-feu mentionne 389/636 pour les requêtes d’annuaire externes ou internes ainsi que des ports distincts pour les points de terminaison Keyserver intégrés ([SEPPmail: Firewall – LDAP and Key server](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html), [SEPPmail: LDAP-based user creation](https://docs.seppmail.com/en/09_ht_user_creation-with-ldap.html)).

Une requête LDAP fait partie du chemin du message dès lors qu’une règle ne peut pas décider sans son résultat. La base de recherche, le filtre, la classe d’objet, l’identité de liaison, la confiance TLS, le délai d’attente, le comportement du cache et le comportement en cas d’indisponibilité doivent donc faire partie du modèle d’exploitation. Un résultat de recherche vide, un délai d’attente et un certificat non fiable ne doivent pas être traités comme le même résultat. Les exigences générales de protocole et TLS sont décrites dans l’[article LDAP](/kb/ldap#bind-sasl-und-tls-vertrauensgrenzen).

Pour les clés et certificats, les cycles de vie suivants, mesurables séparément, sont au minimum nécessaires :

| Objet | Création ou obtention | Contrôle continu | Preuve de restauration |
|---|---|---|---|
| Clé utilisateur S/MIME | locale, importation ou MPKI | expiration, chemin CA, statut de révocation, liaison à l’adresse, algorithme | déchiffrer un ancien message avec une clé privée historique |
| Clé OpenPGP | locale, importation ou serveur de clés | empreinte, ID utilisateur, expiration/révocation, capacités | déchiffrer un ancien message avec la Secret Key correspondante |
| Clé de domaine | Managed Domain Service ou échange manuel | état publié du correspondant, rotation, chemin de messagerie complet | tester un ancien et un nouveau message de domaine au-delà de la limite de rotation |
| Certificat SMTP/HTTPS | importation ou CA locale/publique | SAN, chaîne, expiration, correspondance de clé privée, politique pair | démarrer le service avec la clé restaurée et la chaîne attendue |
| Confiance CA | importation Root/Intermediate | objectif, portée, suppression des anciennes CA, OCSP/CRL | tester des chaînes connues valides et révoquées |

Les caractéristiques de vérification suivent les normes respectives ainsi que les contrôles d’importation, de politique et de signature documentés. En cas de signature S/MIME non valide, SEPPmail cite notamment la confiance CA, une chaîne complète, la liaison à l’expéditeur et une modification ultérieure du contenu comme causes ([SEPPmail: Reasons for an invalid signature](https://docs.seppmail.com/en/09_ht_signature_reasons-for-an-invalid-signature.html), [RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280), [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580)).

Comme la passerelle, les clés, l’annuaire, GINA et la file d’attente agissent ensemble, la haute disponibilité doit couvrir davantage qu’un écouteur virtuel. La répartition frontend/backend détermine quels états doivent être disponibles conjointement.

## Cluster, séparation frontend/backend et domaines de défaillance

Selon la matrice de pare-feu, la communication générale de cluster utilise SSH entre les appliances ; la synchronisation horaire par NTP est obligatoire dans le cluster. La matrice mentionne également les IP virtuelles et indique qu’en cas d’alias IP, l’envoi est effectué via l’adresse physique du système actif. Les listes d’autorisation du pare-feu, du smarthost et des connecteurs doivent donc représenter le chemin d’expéditeur réel, et non uniquement l’adresse virtuelle de réception ([SEPPmail: Firewall – Cluster and NTP](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html)).

Un cluster frontend/backend spécial sépare le traitement de messagerie et les données de configuration persistantes. Selon la description du fabricant, le frontend ne contient aucune base de données de configuration ; la configuration et les éléments de clé nécessaires sont fournis par le backend pour une action, puis supprimés du frontend. Lorsque Large File Transfer est activé, un état de fichier supplémentaire reste toutefois nécessaire sur le frontend ([SEPPmail: Frontend/Backend Cluster](https://docs.seppmail.com/en/04_com_09_cl_05_frontend-backend-cluster.html)).

Cette architecture réduit la conservation durable de clés dans la DMZ, mais n’élimine pas toutes les dépendances. Le backend, le chemin SSH, la résolution de noms, l’heure, les certificats et, le cas échéant, le stockage LFT demeurent des domaines de défaillance. « Le frontend accepte SMTP » ne prouve ni que le backend est accessible ni qu’une opération cryptographique peut être exécutée. Inversement, un backend fonctionnel ne prouve pas que l’IP virtuelle, la route externe et le prochain saut SMTP fonctionnent.

Tous les paramètres ne s’appliquent pas à l’échelle du cluster. La documentation Mail System décrit plusieurs paramètres comme spécifiques à la machine, notamment les Extended Postfix settings et certaines cibles TLS, qui doivent être gérés localement. Avant les tests de basculement, une matrice cible-réel par nœud est donc utile : nom système, IP, certificat, identité sortante, routes TLS, Extended Settings, services locaux, cible de supervision et réplication de cluster attendue ([SEPPmail: Mail System – cluster notes](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

## Supervision, journalisation, sauvegarde et restauration

Selon le fabricant, l’adresse Postmaster reçoit des messages importants du watchdog et des rapports quotidiens ; sans adresse configurée et réellement accessible, ce canal d’alerte manque. SEPPmail documente en complément SNMP via UDP 161, NRPE via TCP 5666 ainsi que Syslog via UDP 514 ou TCP 6514 protégé par TLS. Les valeurs et événements surveillés doivent être définis pour chaque installation ; une interface de supervision ouverte ne constitue pas à elle seule un contrôle de santé ([SEPPmail: Mail System – Postmaster](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html), [SEPPmail: Firewall – Monitoring and Syslog](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html)).

Un tableau de bord significatif distingue au minimum l’acceptation et les rejets SMTP, l’âge de la file d’attente par prochain saut, les erreurs de ruleset, les succès et erreurs cryptographiques, l’expiration des certificats, l’accessibilité OCSP/CRL et MPKI, la latence LDAP, les erreurs de connexion et de livraison GINA, l’état du disque, la communication de cluster et l’âge des sauvegardes. Les seuils doivent être liés au débit normal ainsi qu’aux RPO/RTO convenus. Une file d’attente élevée peut indiquer un correspondant lent ; une file d’attente vide peut aussi signifier qu’aucun message n’est plus accepté.

Le Quick Setup Guide décrit une sauvegarde protégée par mot de passe qui sauvegarde les données de l’appliance, y compris les utilisateurs enregistrés et les certificats, mais exclut les journaux. Le mot de passe est requis pour restaurer sur un nouveau système. Le guide décrit aussi une livraison nocturne au « Backup Operator » ([SEPPmail: Quick Setup Guide](https://downloads.seppmail.com/wp-content/uploads/docu/Quick-Setup-Guide-en.pdf)). Il en découle trois contrôles distincts :

1. Le fichier de sauvegarde doit parvenir hors de l’appliance et son âge doit être surveillé.
2. Le mot de passe de sauvegarde doit être stocké séparément, protégé contre les accès et disponible en cas d’urgence.
3. Les journaux ont besoin de leur propre chemin d’exportation et de conservation s’ils doivent être conservés à des fins de forensic ou de preuve.

Une restauration n’est testée que lorsque le système cible démarre, que le réseau et les licences correspondent, que les règles sont actives, que SMTP fonctionne dans les deux sens, que les contenus S/MIME/OpenPGP et GINA historiques peuvent être déchiffrés et qu’un cas de test documenté exécute la politique attendue. La réplication de cluster ne remplace pas ce test : elle traite de la disponibilité, pas d’une copie indépendante contre une mauvaise configuration, une suppression ou des clés compromises.

Le diagnostic suit un message depuis l’acceptation SMTP, la correspondance de règle et la décision de clé jusqu’à la sortie choisie ou au portail GINA.

## Outils de diagnostic

La recherche d’erreurs suit le message depuis le nom public, via SMTP et TLS, jusqu’au ruleset et au chemin de chiffrement sélectionné. Le transport est ainsi d’abord prouvé avant de modifier les fonctions de clés ou de portail.

### DNS, MX et noms publics

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SEPPmail-DNS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.ch
Resolve-DnsName securemail.example.ch -Type A,AAAA
Resolve-DnsName _25._tcp.securemail.example.ch -Type TLSA
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig +short MX example.ch
dig +short A securemail.example.ch
dig +short AAAA securemail.example.ch
dig +short TLSA _25._tcp.securemail.example.ch
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) et [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) vérifient le MX, l’adresse de la passerelle et, le cas échéant, DANE-TLSA. Les vues des résolveurs externes et internes doivent être testées séparément. Le nom d’hôte GINA doit en outre correspondre au certificat ; le MX seul ne dit rien du Forwarding Server en aval ni du connecteur Exchange Online ([RFC 5321: Address resolution and mail routing](https://datatracker.ietf.org/doc/html/rfc5321#section-5), [RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672)).

### Accessibilité TCP, SMTP et STARTTLS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SEPPmail-SMTP- und TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection securemail.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection securemail.example.ch -Port 443 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://securemail.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz securemail.example.ch 25
nc -vz securemail.example.ch 443
openssl s_client -starttls smtp -connect securemail.example.ch:25 \
  -servername securemail.example.ch -showcerts
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) et [`nc`](https://man.openbsd.org/nc) prouvent uniquement l’accessibilité TCP. [`curl.exe`](https://curl.se/docs/manpage.html) sous Windows peut demander STARTTLS ; [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) affiche, sur les systèmes Unix, les capacités SMTP, le handshake et la chaîne de certificats. [`swaks`](https://jetmore.org/john/code/swaks/) convient pour une transaction SMTP contrôlée avec enveloppe et DATA ; il ne doit être utilisé qu’avec des destinataires de test autorisés. Un test de livraison n’est complet qu’avec l’ID de file d’attente, l’événement de journalisation et l’arrivée au prochain saut.

### HTTPS GINA et nom de certificat

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SEPPmail-GINA-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Invoke-WebRequest -Method Head -Uri https://securemail.example.ch/
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
curl --head --verbose https://securemail.example.ch/
openssl s_client -connect securemail.example.ch:443 \
  -servername securemail.example.ch -showcerts
```

  </div>
</div>

[`Invoke-WebRequest`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest) et [`curl`](https://curl.se/docs/manpage.html) vérifient l’accessibilité HTTP et le comportement de réponse ; [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) rend visibles la session TLS négociée et la chaîne présentée. Un `200`, `301` ou `302` ne prouve encore ni une connexion GINA réussie ni l’accès à un message historique. Cela nécessite un flux utilisateur synthétique avec inscription, connexion, lecture, réponse et réinitialisation conformément au domaine GINA configuré.

### Chemin de paquets sur un point de mesure autorisé

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SEPPmail-Paketerfassung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
pktmon filter remove
pktmon filter add SEPPmail-SMTP -p 25
pktmon start --capture --pkt-size 0 --file-name seppmail.etl
pktmon stop
pktmon pcapng seppmail.etl -o seppmail.pcapng
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
tcpdump -ni any -s 0 -w seppmail.pcap 'tcp port 25 or tcp port 443'
```

  </div>
</div>

[`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) et [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) ne capturent que le trafic qui passe par le point de mesure choisi. Sur un client d’administration, il ne s’agit pas automatiquement du trafic de messagerie entre le système de messagerie et la passerelle ; un hôte autorisé, un port miroir ou un autre point d’observation approprié est nécessaire. SMTP avant STARTTLS peut contenir des contenus à caractère personnel, HTTPS et STARTTLS fournissent au moins des métadonnées. La capture, l’accès, la conservation et la suppression doivent donc faire partie du runbook d’incident.

## Histoire technique

SEPPmail décrit l’origine de l’entreprise en 2001 : son fondateur Stefan Klein a d’abord développé une solution de communication chiffrée entre un cabinet d’avocats zurichois et ses clients. Cette exigence a donné naissance à l’entreprise de produits ultérieure ([SEPPmail: Die SEPPmail-Story](https://www.seppmail.com/de/die-seppmail-story/)).

L’histoire du produit documentée publiquement montre l’évolution d’une passerelle centrale de chiffrement vers une appliance modulaire : des notes de version historiques mentionnent déjà en 2007 une CA/RA interne, le chiffrement de domaine S/MIME, des fonctions de webmail, de l’antispam/de l’antivirus et un utilisateur de sauvegarde restreint. Les générations ultérieures ont notamment ajouté des politiques multilocataires, des interfaces REST, des connecteurs MPKI, Large File Transfer, DKIM/ARC, des méthodes S/MIME modernes et des rôles de cluster spécialisés ([SEPPmail: Revision History](https://docs.seppmail.com/ch/20_revision-history.html)).

L’infrastructure sous-jacente a également évolué. L’historique des révisions mentionne au fil des ans OpenBSD, Postfix, Apache, Perl, GnuPG, OpenLDAP, OpenSSL/LibreSSL, Dovecot, Fetchmail et ClamAV. Cette origine est pertinente pour les administrateurs, car les termes de journalisation, les paramètres MTA, les formats de certificats et le comportement réseau proviennent de ces composants. Les interventions directes sous l’interface du fabricant ne sont toutefois pas automatiquement prises en charge ; la documentation avertit en particulier des modifications Extended Postfix et de leur absence de synchronisation de cluster ([SEPPmail: Mail System](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

## Sources

- [SEPPmail – Products](https://www.seppmail.com/products/)
- [SEPPmail – Mail System](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)
- [SEPPmail – Add/Edit Encryption Policy](https://docs.seppmail.com/ch/07_mi_07_mp_01_ep_01_addedit-encryption-policy.html)
- [RFC 5321: SMTP responsibility](https://datatracker.ietf.org/doc/html/rfc5321)
- [IETF RFC 8551 – S/MIME 4.0 Message Specification](https://datatracker.ietf.org/doc/html/rfc8551)
- [SEPPmail – Frontend/Backend Cluster](https://docs.seppmail.com/en/04_com_09_cl_05_frontend-backend-cluster.html)
- [SEPPmail – Setting Up The Firewall/Router](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html)
- [IETF RFC 3207 – SMTP Service Extension for Secure SMTP over TLS](https://datatracker.ietf.org/doc/html/rfc3207)
- [SEPPmail – Setup with Exchange Online](https://docs.seppmail.com/en/04_com_powershell_setup-exo.html)
- [Microsoft – Configure mail flow using connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow)
- [SEPPmail – Revision History](https://docs.seppmail.com/ch/20_revision-history.html)
- [SEPPmail – GINA User Details](https://docs.seppmail.com/en/07_mi_18_ga_01_user-x.html)
- [SEPPmail – Quick Setup Guide](https://downloads.seppmail.com/wp-content/uploads/docu/Quick-Setup-Guide-en.pdf)
- [SEPPmail – Macros](https://docs.seppmail.com/ch/08_cr_01_gi_04_macros.html)
- [SEPPmail – LDAP-based user creation](https://docs.seppmail.com/en/09_ht_user_creation-with-ldap.html)
- [SEPPmail – Signing Emails Between Clients](https://docs.seppmail.com/en/09_ht_tenant_signing-e-mails-between-tenants.html)
- [SEPPmail – Internal X-header processing](https://docs.seppmail.com/en/08_cr_01_gi_06_x-header.html)
- [SEPPmail – Sender identity selection](https://docs.seppmail.com/en/09_ht_user_creation-based-on-other-criteria-than-from-header.html)
- [RFC 5322: Originator fields](https://datatracker.ietf.org/doc/html/rfc5322)
- [IETF RFC 5652 – Cryptographic Message Syntax](https://datatracker.ietf.org/doc/html/rfc5652)
- [IETF RFC 5280 – Internet X.509 PKI Certificate and CRL Profile](https://datatracker.ietf.org/doc/html/rfc5280)
- [IETF RFC 6960 – Online Certificate Status Protocol](https://datatracker.ietf.org/doc/html/rfc6960)
- [IETF RFC 9580 – OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580)
- [IETF RFC 7672 – SMTP Security via DANE](https://datatracker.ietf.org/doc/html/rfc7672)
- [SEPPmail – GINA Domains](https://docs.seppmail.com/en/07_mi_17_gd__gina-domains.html)
- [SEPPmail – Reasons For An Invalid Signature](https://docs.seppmail.com/en/09_ht_signature_reasons-for-an-invalid-signature.html)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [ISC BIND 9 – dig manpage](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc manpage](https://man.openbsd.org/nc)
- [curl – command line manpage](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [swaks – SMTP test tool](https://jetmore.org/john/code/swaks/)
- [Microsoft Learn – Invoke-WebRequest](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest)
- [Microsoft Learn – pktmon](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [tcpdump – manual page](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [SEPPmail – Die SEPPmail-Story](https://www.seppmail.com/de/die-seppmail-story/)
