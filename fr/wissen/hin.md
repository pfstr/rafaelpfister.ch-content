---
title: "HIN : espace de confiance, passerelle de messagerie et Stargate"
blatt: "hin"
description: "HIN pour les administrateurs de messagerie et de plateforme : modèle de confiance et d’identité, HIN Mail, messagerie classique et passerelle d’accès, HIN Client, PKI, chemins SMTP et portail, magasin de messagerie, architecture Stargate, migration, supervision, reprise et diagnostic."
fakten:
  - label: Plateforme
    wert: Espace de confiance pour le secteur suisse de la santé
    href: https://www.hin.ch/de/services/hin-mail/hin-mail.cfm
  - label: Exploitant
    wert: Health Info Net AG · fondée en 1996
    href: https://www.hin.ch/de/ueber-hin/unternehmen/geschichte.cfm
  - label: Modèle de messagerie
    wert: HIN vers HIN automatiquement · externe avec marquage confidentiel
    href: https://support.hin.ch/de/service/hin-mail-und-mobile.cfm
  - label: Edge classique
    wert: Messagerie et accès sous forme d’appliances virtuelles
    href: https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm
  - label: Architecture cible
    wert: Postfix → MXEngine → Postfix
    href: https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf
  - label: Stack principal
    wert: OPA/Rego · PostgreSQL · Vault · MinIO
    href: https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf
  - label: Transport mesh
    wert: WireGuard · IDAgent · Port 19818
    href: https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf
  - label: Ancre de confiance
    wert: Identité HIN · S/MIME · clés et CSR
    href: https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf
  - label: Accès Web
    wert: HIN Client · Access Gateway · SAML · OAuth 2.0
    href: https://download.hin.ch/oauth2/doku/de/
  - label: Magasin de messagerie
    wert: séparé du transport de passerelle · IMAP/POP/Webmail
    href: https://support.hin.ch/de/service/hin-gateway.cfm
  - label: Déploiement
    wert: Linux · Docker Compose · images VM
    href: https://health-info-net-ag.github.io/Stargate-deployment/de/
  - label: Observability
    wert: Prometheus · Promtail/Loki · Node Exporter
    href: https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf
werbung:
  - stargate
  - newsletter
ctaThemen:
  - hin-gateway
translationSourceHash: 4b3c48b30e1dfb939e082bba0637c401f51d48a6dc5ed8e71b90c9d8114b2c8c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-06T11:17:00.013Z
translationReview: automatic
---

# HIN : espace de confiance, passerelle de messagerie et Stargate

HIN n’est pas un simple programme de chiffrement, mais un espace de confiance sectoriel composé d’identités vérifiées, de services de plateforme centralisés et de composants d’accès ou de passerelle. **HIN Mail** protège les communications par e-mail, **HIN Access** assure l’accès aux applications Web protégées, et une **identité HIN** associe la personne ou l’organisation autorisée à du matériel cryptographique. Dans les institutions, les composants Mail et Access se situent à la frontière entre l’infrastructure propre et la plateforme HIN ([HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm), [Adhésion collective HIN avec passerelle](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)).

Pour les administrateurs de messagerie, il convient de distinguer quatre espaces d’état. Le serveur de messagerie local possède des boîtes aux lettres, des connecteurs et des files d’attente. La passerelle prend les décisions de transport, de confiance et de protection. La plateforme HIN fournit des services d’annuaire, d’identité, de clés, de messagerie et d’accès. Pour les destinataires en dehors de la communauté HIN, un chemin Web et d’authentification s’ajoute. Une perturbation dans un espace ne constitue pas automatiquement une perturbation dans tous les autres ; par exemple, un port SMTP accessible ne prouve ni la validité d’une identité HIN ni la réussite d’une livraison par portail ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm), [HIN Mail aux non-membres](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)).

Le nom du produit requiert également une contextualisation temporelle. La génération classique de passerelles comprend une passerelle de messagerie, MGW, et, selon le contrat, une passerelle d’accès, AGW. HIN présente comme successeur le nœud mesh compatible avec le courrier électronique développé dans le projet **Stargate**. Il demeure compatible SMTP vis-à-vis des systèmes de messagerie locaux, mais modifie l’architecture, la distribution des clés, le déploiement et le transport entre organisations. Les déclarations concernant la plateforme cible ne doivent donc pas être transférées sans vérification à un MGW existant, et inversement ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

L’explication suit un message de l’expéditeur au destinataire, via l’identité HIN et la passerelle. Elle traite ensuite des changements de plateforme, dépendances, diagnostics, matériel de clés et restauration.

## Approche architecturale : espace de confiance avec composants Edge

Le modèle collectif classique fournit HIN Mail et HIN Access sous forme d’appliances virtuelles. HIN documente le chiffrement et la signature S/MIME au niveau du domaine de messagerie ainsi qu’une piste d’audit ; l’Access Gateway fonctionne comme fournisseur local d’identité, associe les requêtes vérifiées à des eID HIN et peut intégrer les services d’annuaire et d’authentification existants ([Adhésion collective HIN avec passerelle](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)).

Cette construction est un **modèle de confiance Edge** : l’organisation contrôle le serveur de messagerie, le DNS, le pare-feu, la livraison interne et la ressource de passerelle locale ; HIN exploite les services de confiance et de plateforme de niveau supérieur. La conclusion SMTP positive transfère la responsabilité d’un message concret au saut suivant ; elle ne dit rien sur l’achèvement du chemin HIN, du portail ou du destinataire qui suit. Pour la nouvelle pile de passerelle, HIN attribue explicitement au client la responsabilité du DNS et de la réputation de messagerie ([HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

Stargate déplace cette frontière vers un nœud mesh lié à l’organisation. HIN décrit une architecture de microservices cloud-native avec API REST, identités décentralisées, gestion distribuée des clés et politiques programmables. Une instance dédiée est prévue par organisation ; HIN cite comme formes de déploiement des images virtuelles et des conteneurs ainsi que, dans la description du produit, des environnements OpenShift et Kubernetes. Il s’agit d’une architecture cible publiée, non d’une preuve que chaque organisation HIN existante est déjà exploitée ainsi ([HIN Gateway : architecture et déploiement](https://support.hin.ch/de/service/hin-gateway.cfm), [Description du produit HIN Gateway](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm)).

## Pile technologique et responsabilités

La pile publiquement documentée se compose de plusieurs générations et ne doit pas être mélangée en un monolithe unique :

| Niveau | Implémentation de la nouvelle passerelle | Domaine d’état et de panne | Preuve pour l’administrateur |
|---|---|---|---|
| Bord SMTP | **Postfix Relay** pour l’acceptation, les nouvelles tentatives, le routage DNS et la livraison | Le transport est séparé de la décision relative au contenu | Réponse SMTP finale, ID de file d’attente, saut suivant, MX et PTR |
| Traitement | **MXEngine** pour l’ingestion HTTP/SMTP, la transformation et la stratégie de livraison | Les erreurs de traitement restent distinctes des nouvelles tentatives Postfix | Message-ID, événement MXEngine, transformation, retour à Postfix |
| Politique | **Open Policy Agent avec Rego**, optionnellement synchronisé via Git | L’état des règles est un état de données et de versions | Révision de politique, entrées, résultat, validation et rollback |
| Identité et cryptographie | **S/MIME Keys Client**, **IDAgent**, émetteur et vérificateur | CSR, certificat, clé privée et identité de pair sont des objets distincts | Empreinte, titulaire, expiration, émetteur, pair et rotation |
| Persistance | **PostgreSQL** par service, **Vault** pour les secrets, **MinIO** pour les messages et pièces jointes | Base de données, magasin de secrets et stockage d’objets ont leurs propres limites de reprise | Volume, date de sauvegarde, test de restauration et contrôle de cohérence par service |
| Transport mesh | **WireGuard** via l’IDAgent | L’état du canal n’est pas identique à la livraison SMTP | Clé de pair, endpoint, handshake, port 19818 et événement SMTP en aval |
| Observabilité | **Promtail → Loki**, **Node Exporter**, métriques compatibles Prometheus et Version Collector | Transport des journaux, métriques hôte et état de santé du service peuvent échouer séparément | Liveness, ancienneté du scrape, réception des journaux, ressources hôte et base temporelle |
| Déploiement | Linux, **Docker Compose**, volumes Docker persistants ; images VM comme méthode d’installation | Hôte, conteneurs, images et volumes ont des cycles de vie différents | Image approuvée, configuration Compose, inventaire des volumes et test de redémarrage |

HIN documente explicitement le flux de messages comme `External SMTP → Postfix → MXEngine → Postfix → External SMTP`. Le transfert forcé vers MXEngine empêche le contournement des politiques ; Postfix reste responsable de la livraison et des nouvelles tentatives. OPA/Rego maintient les règles métier hors du code applicatif. PostgreSQL, Vault et MinIO stockent différentes classes d’état et ne doivent pas être traités dans la sauvegarde comme un système de fichiers unique ([HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)).

La documentation d’installation technique mentionne les distributions Linux des familles compatibles RHEL ainsi qu’Ubuntu et Debian, une exploitation Docker Compose sur un hôte unique et des images VM pour plusieurs plateformes. Cette déclaration de prise en charge dépend de la version et du déploiement ; la documentation HIN approuvée au moment de l’installation fait foi, et non une liste de distributions reprise statiquement ([HIN Stargate Deployment](https://health-info-net-ag.github.io/Stargate-deployment/de/)).

Le magasin de messagerie reste une autre frontière : selon HIN, Stargate agit comme Mail Transport Agent, tandis qu’IMAP reste assuré sur la plateforme Zimbra existante. HIN Access constitue en outre un chemin autonome d’authentification et d’autorisation via le client, l’AGW ou l’Access Control Service ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm), [Manuel HIN Client 3](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-hin.svg?v=20260813" title="Interaktive Infografik: HIN Vertrauensraum mit lokalem Mailserver, klassischem Mail und Access Gateway, HIN Identität, Mailplattform, Nichtmitglieder-Portal sowie Stargate-Zielarchitektur und Betriebssignalen" loading="lazy">
  <a href="/images/kb-interaktiv-hin.svg?v=20260813">Ouvrir directement le graphique interactif</a>.
</iframe>

## Identité et PKI avec autorisation

Une identité HIN est plus qu’une adresse e-mail. Lors de l’activation, le HIN Client génère une paire de clés ; le mot de passe déverrouille le matériel de clés local. Dans la nouvelle passerelle, le S/MIME Keys Client génère des clés cryptographiques et des Certificate Signing Requests, tandis que Vault stocke les clés privées, identifiants et configurations sensibles. Un certificat lie une clé publique à une identité nommée, mais ne remplace pas une autorisation liée à l’application ([Manuel HIN Client 3](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf), [HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280)).

Le cycle de vie doit figurer dans le runbook IAM : enregistrement, activation, changement d’appareil, changement de rôle ou de nom, blocage, nouvel enregistrement en cas de suspicion concernant une clé et départ. HIN exige une vérification d’identité après la commande et distingue les eID personnelles de l’ID d’organisation ou d’appareil dans le modèle collectif. Un transport de messagerie fonctionnel ne doit pas être considéré comme la preuve qu’une identité ancienne ou incorrectement attribuée ne possède plus d’accès ([Identité HIN](https://support.hin.ch/de/service/hin-identitaet.cfm), [Adhésion collective HIN avec passerelle](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)).

HIN Access et HIN Mail partagent l’espace de confiance, mais pas le même flux de protocole. L’Access Control Service peut demander un niveau d’authentification via la requête SAML `RequestedAuthnContext`. HIN documente des profils pour le mot de passe et la MFA. Pour les intégrations, HIN fournit également des flux OAuth 2.0 pour Authorization Code et Client Credentials. L’authentification, l’émission de jetons et l’autorisation par l’application cible doivent être journalisées séparément ([HIN Authentication Context](https://support.hin.ch/de/thema/hin-access/pwd.cfm), [Intégration OAuth2 HIN](https://download.hin.ch/oauth2/doku/de/), [SAML 2.0 Core](https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf), [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749)).

Ce n’est qu’une fois l’identité et la PKI clarifiées que le chemin de messagerie peut être évalué. La passerelle et la politique décident, selon l’expéditeur, le destinataire et la destination, quel mécanisme de protection est appliqué.

## Chemins de messagerie et décisions de protection

Les composants HIN impliqués étant identifiés, le chemin concret du message peut désormais être expliqué. Les cas suivants diffèrent selon l’emplacement de l’expéditeur et du destinataire ainsi que le service qui prend la décision de protection.

### Entre participants HIN

HIN décrit les messages entre adresses HIN comme transmis automatiquement conformément à la protection des données. La plateforme indique l’état d’intégrité dans l’objet avec `[HIN secured]` ou `[Not secured by HIN]`. Ce marquage est un signal pour l’utilisateur, mais pas une corrélation technique suffisante : pour un incident, il faut également l’expéditeur et le destinataire d’enveloppe, Internet `Message-ID`, la chaîne `Received`, l’événement de passerelle et la fenêtre temporelle ([HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm), [HIN Mail et Mobile](https://support.hin.ch/de/service/hin-mail-und-mobile.cfm), [RFC 5322](https://datatracker.ietf.org/doc/html/rfc5322)).

Un chemin organisationnel classique peut être décrit comme `Mailserver → SMTP → MGW → HIN → MGW → SMTP → Mailserver`. Chaque étape termine une session et peut posséder sa propre file d’attente. HIN décrit le modèle collectif comme un chiffrement et une signature S/MIME au niveau du domaine de messagerie ; la livraison locale avant et après cette frontière de passerelle reste un domaine distinct de protection et d’exploitation ([Adhésion collective HIN avec passerelle](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm), [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

### Aux personnes sans adhésion HIN

Pour les destinataires sans adresse HIN, l’expéditeur doit, selon la documentation HIN, marquer explicitement le message comme confidentiel, par exemple avec `(Vertraulich)` dans l’objet. Le destinataire ouvre le contenu protégé via un chemin Web et s’authentifie avec son numéro de téléphone mobile et un code SMS ; une réponse sécurisée est possible. Des états supplémentaires apparaissent ainsi : e-mail de notification, objet de portail, enregistrement du destinataire, second facteur, conservation et canal de réponse ([HIN Mail aux non-membres](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm), [HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)).

Une notification livrée n’est pas encore un message lu. Les tests synthétiques doivent donc couvrir l’ensemble du chemin jusqu’à la connexion, l’ouverture, la pièce jointe et la réponse. Les transferts depuis le portail peuvent quitter le chemin protégé ; HIN indique expressément que certains types de transfert sont envoyés sans chiffrement. Ces actions utilisateur doivent être intégrées à la formation, au modèle DLP et à l’analyse des incidents ([HIN Mail aux non-membres](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)).

### Appareils, applications et envoi de masse

La passerelle de messagerie classique peut servir d’interface SMTP pour les appareils internes ; HIN documente également cette capacité pour Stargate. La page de services HIN Mail indique que l’envoi système via la passerelle nécessite une licence séparée. Les scanners, systèmes d’information hospitaliers, applications de laboratoire et processus batch nécessitent donc un chemin propre et documenté pour l’expéditeur, le relais, les volumes et les erreurs, plutôt qu’une utilisation implicite du flux de messagerie utilisateur ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm), [HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)).

## Routage SMTP et limites d’acceptation

La passerelle doit savoir précisément, dans chaque direction, quels domaines sont locaux, autoritatifs, à relayer ou à refuser. Une responsabilité ambiguë entre Exchange, connecteur cloud, passerelle de messagerie sécurisée et edge HIN engendre soit des contournements, soit des boucles. [SMTP](/kb/smtp) prescrit des lignes de trace et décrit la détection des boucles ; le transfert effectif de responsabilité intervient seulement avec une réponse positive après l’intégralité du contenu du message ([RFC 5321, informations de trace](https://datatracker.ietf.org/doc/html/rfc5321#section-4.4), [RFC 5321, DATA](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4)).

Pour chaque route, la documentation d’exploitation doit au minimum inclure les valeurs suivantes :

- listener local, port et réseaux sources ou identités de pair attendus ;
- nom EHLO, domaines d’enveloppe et plages de relais autorisées ;
- ordre du filtre antispam/antimalware, de la passerelle HIN et du système de messagerie interne ;
- saut suivant, résolution DNS ou Smarthost et exigence [TLS](/kb/tls) ;
- comportement lorsque la plateforme HIN, le composant d’identité ou de clés est inaccessible ;
- ancienneté de la file d’attente, plan de nouvelles tentatives, nombre maximal de sauts et responsabilité des rebonds ;
- chemins d’exception pour les appareils, l’envoi système et la coexistence durant la migration.

Pour Exchange Online, il s’agit d’une question de connecteurs et non uniquement de DNS. Microsoft documente le routage vers des passerelles tierces ainsi que les conditions de connecteurs basées sur certificats ou adresses IP. La FAQ HIN confirme la prise en charge de principe des architectures hybrides et proches de Microsoft 365, mais renvoie à la documentation de migration spécifique au client pour la configuration concrète ([Microsoft : flux de messagerie avec connecteurs](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow), [Microsoft : flux de messagerie via un cloud tiers](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud), [HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

## Magasin de messagerie et accès client avec jeton

L’accès à la boîte aux lettres et le transport par passerelle sont des modèles d’exploitation différents. HIN publie pour les comptes HIN Mail personnels IMAP sur le port 993 avec TLS et Message Submission sur le port 587 avec STARTTLS ; le mot de passe est un jeton Mail généré. POP sur le port 995 est également documenté, mais télécharge généralement les messages localement et peut les supprimer côté serveur. Les rôles de protocoles correspondent à [IMAP](/kb/apache-james#protokolle-tls-und-ports), POP3 et Message Submission, et non au transport de passerelle à passerelle ([HIN Mail Token Service](https://support.hin.ch/de/service/hin-mail-und-mobile/token-service-mail-client.cfm), [Configuration POP HIN](https://support.hin.ch/de/service/hin-mail-und-mobile/mail-clients-einrichten-pop.cfm), [RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051), [RFC 1939](https://datatracker.ietf.org/doc/html/rfc1939), [RFC 6409](https://datatracker.ietf.org/doc/html/rfc6409)).

Le manuel HIN Client documente le remplacement de l’ancien proxy de messagerie local par un accès basé sur des jetons. HIN précise explicitement pour les serveurs de terminaux que l’envoi et la réception s’effectuent au moyen de jetons Mail lorsque le proxy est désactivé. Cela est pertinent pour l’historique technique : le client était à l’origine un intermédiaire de communication pour HTTP, SMTP, POP et IMAP ; les modèles d’exploitation ultérieurs dissocient davantage l’accès au client de messagerie du client d’identité HIN ([Manuel HIN Client 3](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf), [HIN Client sur serveurs de terminaux](https://support.hin.ch/de/thema/hin-client/hin-client-auf-terminalserver.cfm)).

Dans la FAQ Stargate, HIN décrit le Mail Storage Agent existant comme basé sur Zimbra et initialement non affecté par le nouveau transport de passerelle. Un flux de messagerie Stargate réussi ne prouve donc ni la disponibilité IMAP ni la cohérence de la boîte aux lettres. Inversement, le Webmail peut fonctionner alors que le connecteur d’organisation ou le transport mesh est perturbé ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

Le chemin classique du magasin de messagerie n’est pas la seule forme d’exploitation. Stargate déplace les fonctions et responsabilités et doit donc être compris comme un chemin distinct de messages et d’administration.

## Stargate comme architecture cible

Stargate doit préserver la compatibilité e-mail en périphérie tout en permettant un échange de données décentralisé plus général. HIN mentionne Self-Sovereign Identity, Data Mesh, microservices, API RESTful, composants open source et un nœud mesh compatible avec la messagerie. Pour le canal entre instances Stargate, un transport application à application basé sur [WireGuard](https://www.wireguard.com/protocol/) est annoncé ; HIN le distingue explicitement d’un tunnel VPN général ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

Pour les administrateurs, il en résulte une trace en trois parties :

1. Localement, SMTP avec réponse d’acceptation, file d’attente et connecteur reste la limite vérifiable.
2. Entre les nœuds mesh s’ajoutent les états d’identité, de découverte, de clés et de canal WireGuard.
3. À destination, un chemin SMTP local vers le système de messagerie destinataire est recréé.

Les aperçus système publiés attestent Postfix, MXEngine, OPA/Rego, PostgreSQL, Vault, MinIO, IDAgent, Promtail/Loki et les métriques compatibles Prometheus. Aucun langage de programmation ni arbre source complet des services produit n’y est indiqué ; cette information n’est donc pas déduite d’images de conteneurs ou de projets tiers portant le même nom ([HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)).

Les limites réseau de la nouvelle passerelle sont documentées publiquement de manière inhabituellement concrète :

| Port et direction | Rôle | Importance opérationnelle |
|---|---|---|
| TCP 25 entrant et sortant | Acceptation SMTP et livraison basée sur MX | Exposition Internet, réputation, file d’attente et routage du saut suivant |
| TCP 8084 entrant | Rappel HTTP du Sealer distant | selon HIN, intentionnellement sans couche TLS supplémentaire car la charge utile est elle-même chiffrée |
| TCP et UDP 19818 dans les deux directions | WireGuard entre IDAgents | Vérifier ensemble clé de pair, endpoint, NAT et pare-feu |
| TCP 443 et 4433 sortant | Registre, CA S/MIME, Sealer, Issuer, journalisation et Verifier | Dépendance à la plateforme malgré l’exploitation de la passerelle locale |
| TCP et UDP 53 sortant | Résolution MX, SPF, A/AAAA et PTR | Le routage et les décisions de sécurité dépendent de [DNS](/kb/dns) |

Les ports de diagnostic et de service exposés localement, notamment PostgreSQL, Vault, MinIO et les endpoints de métriques, ne doivent selon HIN pas être accessibles depuis l’Internet public. Les accès d’administration sur 22, 443, 8180 ou 8190 doivent appartenir à un réseau de gestion défini et non à une ouverture Internet globale ([HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf), [HIN Stargate Deployment](https://health-info-net-ag.github.io/Stargate-deployment/de/)).

HIN décrit la migration comme la mise en place parallèle de la nouvelle passerelle. Ce n’est qu’après configuration, tests et confirmation de l’état opérationnel que le client décide du basculement. Les adresses IP existantes peuvent en principe être réutilisées, mais la configuration doit être traduite. Un runbook de basculement sûr comprend donc au minimum un inventaire, une exportation, une route parallèle, une matrice de test, un critère de basculement, un chemin de retour, le traitement des files d’attente et un rollback clair ([HIN Gateway : migration](https://support.hin.ch/de/service/hin-gateway.cfm)).

## Haute disponibilité avec sauvegarde et reprise

HIN documente pour son propre environnement Stargate un cluster OpenShift redondant, mais ne prévoit pas automatiquement deux machines virtuelles redondantes chez le client. Ces affirmations ne doivent pas être réunies en une haute disponibilité générale de bout en bout. Un service de plateforme redondant ne protège pas contre un hyperviseur local unique, une règle de connecteur erronée, une clé expirée ou un pare-feu bloqué ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

Dans la nouvelle passerelle, les services persistent via des volumes Docker. Vault se scelle automatiquement après un redémarrage de conteneur et exige un processus d’unseal contrôlé. L’aperçu technique mentionne par défaut des sauvegardes quotidiennes des bases de données, clés et secrets Vault, fichiers de configuration ainsi que certificats ; la responsabilité et la conservation doivent être convenues avec le client ([HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)).

Un inventaire de reprise devrait contenir au minimum, pour chaque génération :

| Objet | Effet de la perte | Étape de reprise vérifiable |
|---|---|---|
| Hôte, définition Compose et conteneurs | Le nœud Edge ne démarre pas | Fournir une nouvelle cible à partir d’une image approuvée et générer un environnement d’exécution déterministe |
| Configuration client et politiques OPA/Rego | Route, politique ou domaine erroné | Restaurer l’état versionné, charger la politique et exécuter la matrice de test |
| Bases de données PostgreSQL | État de politique, métadonnées ou agent absent | Effectuer une restauration de base de données par service et une vérification référentielle |
| Clés Vault, secrets et certificats | Déchiffrement, identité de pair ou d’organisation absent | Effectuer une restauration approuvée, un unseal et un test fonctionnel cryptographique |
| Messages et pièces jointes MinIO | Message ou objet d’archive absent | Clarifier séparément avec HIN l’étendue et la conservation, puis tester la restauration d’objet |
| Connecteurs et DNS | Contournement, boucle ou impossibilité de livraison | Vérifier la route dans les deux directions avec une Message-ID sans ambiguïté |
| File d’attente ou preuve de transfert | Doublons ou perte de messages | Clarifier la responsabilité ouverte par message avant le basculement |
| Boîte aux lettres et jeton Mail | Accès client perturbé | Valider séparément via Webmail et IMAP/Submission |
| Journaux d’audit et d’exploitation | Incident non reconstructible | Tester la base temporelle, l’exportation, la conservation et l’entrée SIEM |

La liste publique des sauvegardes mentionne la base de données, Vault, la configuration et les certificats, mais pas explicitement les messages et pièces jointes MinIO. On ne peut en déduire ni leur sauvegarde ni leur exclusion intentionnelle ; ce point précis doit être consigné par écrit dans l’accord de conservation, sauvegarde et restauration avant la mise en production. Un snapshot VM générique ne prouve par ailleurs ni un état PostgreSQL cohérent ni un Vault restaurable ([HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf), [HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf)).

En cas de perturbation, le message est suivi depuis l’acceptation jusqu’au chemin de livraison choisi, en passant par la décision d’identité et de politique ; ce n’est qu’ensuite que des composants individuels sont redémarrés ou contournés.

## Supervision et triage des incidents

Une vision opérationnelle utile combine des signaux locaux et centralisés :

- disponibilité et ancienneté de file d’attente par saut SMTP suivant ;
- taux d’acceptation, de transmission et de rebond avec Message-ID corrélable ;
- erreurs HIN, portail, identité, clés et politique séparées ;
- expiration des certificats, état de l’enregistrement et des jetons ;
- accès aux boîtes aux lettres par Webmail et IMAP indépendamment de la passerelle ;
- résolution DNS et pare-feu par noms plutôt que par adresses IP HIN codées en dur ;
- messages de plateforme sur [HIN Status](https://status.hin.ch/) plus télémétrie locale.

La nouvelle pile fournit des métriques de services compatibles Prometheus, des métriques hôte via Node Exporter, un envoi centralisé des journaux via Promtail vers Loki et un Version Collector qui interroge les endpoints Liveness. Pour l’alerte, il convient au minimum de traiter séparément l’absence de scrape, l’absence de réception de journaux, l’ancienneté de la file Postfix, les erreurs MXEngine, l’état de scellement Vault, la capacité PostgreSQL et MinIO, l’état des pairs WireGuard ainsi que l’expiration des certificats ([HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)).

La documentation de pare-feu HIN recommande les noms DNS car les adresses IP peuvent changer, et cite notamment les chemins HTTPS et SMTP pour les services client. Une page d’état globale ne peut pas détecter une perturbation locale de DNS, NAT, MTU, connecteur ou clé. Le triage commence donc par le périmètre : un utilisateur, une identité, un domaine, une direction, une passerelle ou la plateforme ([Adaptations du pare-feu HIN](https://support.hin.ch/de/thema/hin-client/anpassung-firewall-hin-client.cfm), [HIN Status](https://status.hin.ch/)).

L’offre collective classique mentionne une piste d’audit pour le flux de messagerie ; la nouvelle passerelle ajoute des journaux centraux structurés. Pour une analyse complète des messages, ces preuves doivent être corrélées avec les journaux SMTP locaux et ceux du système de messagerie. La synchronisation temporelle et des fuseaux horaires uniformes sont des exigences opérationnelles, non des réglages cosmétiques ([Adhésion collective HIN avec passerelle](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm), [HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf)).

## Outils de diagnostic

Le diagnostic commence par le nom public ou interne, puis suit le chemin de messagerie réel. Ce n’est qu’une fois le DNS, la connexion et le certificat corrects que l’état de la passerelle, la file d’attente et les événements spécifiques à HIN sont évalués.

### DNS et endpoints HIN

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für HIN-DNS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName gateway.hin.ch -Type A
Resolve-DnsName gateway.hin.ch -Type AAAA
Resolve-DnsName smtp.mail.hin.ch -Type A
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig +short A gateway.hin.ch
dig +short AAAA gateway.hin.ch
dig +short A smtp.mail.hin.ch
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) et [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) indiquent si les noms documentés peuvent être résolus depuis la perspective du résolveur réellement utilisé. Cela est plus important qu’une valeur IP copiée, en particulier avec Split DNS et des proxys ; HIN recommande expressément les noms DNS plutôt que des adresses fixées à long terme ([Adaptations du pare-feu HIN](https://support.hin.ch/de/thema/hin-client/anpassung-firewall-hin-client.cfm)).

### Accessibilité TCP et TLS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für HIN-TCP- und TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection hin-gateway.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection hin-gateway.example.ch -Port 19818 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://hin-gateway.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz hin-gateway.example.ch 25
nc -vz hin-gateway.example.ch 19818
nc -vzu hin-gateway.example.ch 19818
openssl s_client -starttls smtp -connect hin-gateway.example.ch:25 \
  -servername hin-gateway.example.ch -showcerts
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) et [`nc`](https://man.openbsd.org/nc) attestent le chemin TCP. L’appel UDP de `nc` peut tout au plus suggérer l’accessibilité ; comme WireGuard rejette les paquets non autorisés sans réponse, seul le handshake de pair authentifié constitue une preuve solide pour le port 19818. Windows-[`curl.exe`](https://curl.se/docs/manpage.html) et [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) vérifient la périphérie SMTP/STARTTLS. Un handshake réussi ne prouve pas encore le traitement de la politique ni la livraison du message ([HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf), [RFC 8446](https://datatracker.ietf.org/doc/html/rfc8446)).

### Transaction SMTP contrôlée

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für einen autorisierten HIN-Gateway-SMTP-Test">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
curl.exe --verbose --url smtp://hin-gateway.intern.example:25 `
  --mail-from hin-test@example.ch `
  --mail-rcpt test-recipient@example.net `
  --upload-file .\hin-test.eml
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
swaks --server hin-gateway.intern.example --port 25 \
  --from hin-test@example.ch --to test-recipient@example.net \
  --data hin-test.eml
```

  </div>
</div>

[`curl`](https://curl.se/docs/manpage.html) et [`swaks`](https://jetmore.org/john/code/swaks/) ne doivent être utilisés que contre un listener et un destinataire de test expressément autorisés. Il convient d’enregistrer la réponse finale après `DATA`, l’ID de file d’attente locale, l’événement de passerelle, le chemin de protection choisi, le saut suivant et l’arrivée effective. Un `250` sur `RCPT TO` ne constitue pas encore une acceptation du contenu du message ([RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

### États locaux des sockets

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für lokale HIN-Gateway-Socketdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-NetTCPConnection -State Listen,Established |
  Where-Object LocalPort -In 25,443,587,993
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
ss -tanp '( sport = :25 or sport = :443 or sport = :587 or sport = :993 )'
```

  </div>
</div>

[`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) et [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) affichent les listeners locaux et les sessions TCP établies. Ils ne sont pertinents que là où l’administrateur a accès à l’hôte concerné ; un produit d’appliance ou de conteneur géré ne doit pas être modifié par des accès shell non documentés.

### Chemin des paquets au bon point de mesure

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für HIN-Paketerfassung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
pktmon filter remove
pktmon filter add HIN-SMTP -p 25
pktmon start --capture --pkt-size 0 --file-name hin.etl
pktmon stop
pktmon pcapng hin.etl -o hin.pcapng
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
tcpdump -ni any -s 0 -w hin.pcap \
  'tcp port 25 or tcp port 443 or tcp port 587 or tcp port 993'
```

  </div>
</div>

[`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) et [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) ne voient que le trafic au point de mesure sélectionné. Pour Stargate, une trace SMTP locale ne peut pas expliquer complètement le canal mesh ; des événements de passerelle et de plateforme sont également nécessaires. Les captures peuvent contenir des adresses, objets ou parties de protocoles non chiffrées et doivent être traitées comme des données d’exploitation sensibles.

## Histoire technique

La FMH et la Caisse des médecins ont fondé Health Info Net AG en 1996, lorsque l’e-mail est apparu dans le secteur de la santé et que l’envoi de données sensibles par e-mail Internet classique a été considéré comme insuffisamment protégé. HIN a donc débuté comme fournisseur de communications e-mail protégées pour le corps médical et s’est développé en un espace plus large de confiance et d’accès ([Histoire de l’entreprise HIN](https://www.hin.ch/de/ueber-hin/unternehmen/geschichte.cfm)).

L’architecture client illustre l’évolution technique. HIN Client 1 et 2 ont été remplacés par HIN Client 3. Pour des raisons de compatibilité, le client continuait à fonctionner comme proxy local pour les navigateurs et programmes de messagerie ; HIN documente ultérieurement Challenge/Response pour l’accès Web et, pour les comptes de messagerie, la transition vers des jetons indépendants via des ports standard. Cet historique explique pourquoi les anciennes instructions d’installation indiquent des ports proxy locaux, alors que les documents plus récents utilisent des endpoints IMAP, POP et Submission directs ([Manuel HIN Client 3](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf), [HIN Mail Token Service](https://support.hin.ch/de/service/hin-mail-und-mobile/token-service-mail-client.cfm)).

Le modèle collectif HIN classique regroupait des appliances Mail et Access sur le réseau client. La page de services actuelle documente pour cette génération le S/MIME au niveau du domaine de messagerie, une piste d’audit, un fournisseur d’identité local ainsi que l’intégration des services d’authentification et d’annuaire existants. Ces fonctions expliquent la séparation historique entre transport de messagerie et accès Web ([Adhésion collective HIN avec passerelle](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)).

À partir de 2025, HIN a introduit une nouvelle livraison aux non-membres ; parallèlement, l’infrastructure de plateforme et d’accès a été renouvelée. La documentation Gateway publiée en 2026 décrit Stargate comme le prochain changement de génération : d’une passerelle de chiffrement e-mail exclusivement à un nœud décentralisé cloud-native pour la messagerie et l’échange structuré de données de santé. Les documents techniques concrétisent ce changement avec Postfix, MXEngine, OPA/Rego, PostgreSQL, Vault, MinIO et une exploitation conteneurisée. Pour un plan de migration, il ne suffit donc pas de remplacer une VM : il faut recalibrer l’identité, les clés, le transport, l’observabilité et la reprise ([HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm), [HIN Access](https://support.hin.ch/de/thema/hin-access.cfm), [HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm), [HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf)).

## Sources

- [HIN – HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)
- [HIN – Adhésion collective avec passerelle](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)
- [Support HIN – HIN Gateway et Stargate](https://support.hin.ch/de/service/hin-gateway.cfm)
- [Support HIN – HIN Mail aux non-membres](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)
- [HIN – Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf)
- [RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [HIN – Description du produit HIN Gateway](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm)
- [HIN – Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)
- [HIN – Stargate Deployment](https://health-info-net-ag.github.io/Stargate-deployment/de/)
- [HIN – Manuel HIN Client 3](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf)
- [RFC 5280 – Internet X.509 PKI](https://datatracker.ietf.org/doc/html/rfc5280)
- [Support HIN – Identité HIN](https://support.hin.ch/de/service/hin-identitaet.cfm)
- [Support HIN – SAML Authentication Context](https://support.hin.ch/de/thema/hin-access/pwd.cfm)
- [HIN – Intégration OAuth2](https://download.hin.ch/oauth2/doku/de/)
- [OASIS – SAML 2.0 Core](https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf)
- [RFC 6749 – OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc6749)
- [Support HIN – HIN Mail et Mobile](https://support.hin.ch/de/service/hin-mail-und-mobile.cfm)
- [RFC 5322 – Internet Message Format](https://datatracker.ietf.org/doc/html/rfc5322)
- [Microsoft – Flux de messagerie avec connecteurs](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow)
- [Microsoft – Flux de messagerie avec un service cloud tiers](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)
- [Support HIN – HIN Mail Token Service](https://support.hin.ch/de/service/hin-mail-und-mobile/token-service-mail-client.cfm)
- [Support HIN – Configuration POP](https://support.hin.ch/de/service/hin-mail-und-mobile/mail-clients-einrichten-pop.cfm)
- [RFC 9051 – IMAP4rev2](https://datatracker.ietf.org/doc/html/rfc9051)
- [RFC 1939 – POP3](https://datatracker.ietf.org/doc/html/rfc1939)
- [RFC 6409 – Message Submission](https://datatracker.ietf.org/doc/html/rfc6409)
- [Support HIN – HIN Client sur serveurs de terminaux](https://support.hin.ch/de/thema/hin-client/hin-client-auf-terminalserver.cfm)
- [WireGuard – Protocol and Cryptography](https://www.wireguard.com/protocol/)
- [HIN Status](https://status.hin.ch/)
- [Support HIN – Adaptations du pare-feu pour HIN Client](https://support.hin.ch/de/thema/hin-client/anpassung-firewall-hin-client.cfm)
- [Microsoft – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [ISC BIND – Manuel dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – Manuel nc](https://man.openbsd.org/nc)
- [curl – Manuel de ligne de commande](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [RFC 8446 – TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446)
- [swaks – Swiss Army Knife for SMTP](https://jetmore.org/john/code/swaks/)
- [Microsoft – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection)
- [Linux man-pages – ss](https://man7.org/linux/man-pages/man8/ss.8.html)
- [Microsoft – Packet Monitor](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [tcpdump – tcpdump(1)](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [HIN – Histoire de l’entreprise](https://www.hin.ch/de/ueber-hin/unternehmen/geschichte.cfm)
- [Support HIN – HIN Access](https://support.hin.ch/de/thema/hin-access.cfm)
