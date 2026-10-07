---
title: "Kerberos : tickets, clés et services de confiance"
blatt: "kerberos"
description: "Kerberos pour les administrateurs : KDC, flux AS/TGS/AP, principals et SPN, tickets, clés de session, keytabs et KVNO, GSS-API, Active Directory et PAC, relations d’approbation, délégation, chiffrement, DNS, temps, exploitation et diagnostic."
fakten:
  - label: Rôle
    wert: Authentification réseau par tickets
    href: https://datatracker.ietf.org/doc/html/rfc4120#section-1.1
  - label: Version du protocole
    wert: Kerberos V5 · RFC 4120
    href: https://datatracker.ietf.org/doc/html/rfc4120
  - label: Autorité de confiance
    wert: KDC avec Authentication Service et TGS
    href: https://datatracker.ietf.org/doc/html/rfc4120#section-1.2
  - label: Échanges
    wert: AS · TGS · AP client/serveur
    href: https://datatracker.ietf.org/doc/html/rfc4120#section-3
  - label: Port KDC
    wert: 88 via UDP et TCP
    href: https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=kerberos
  - label: Service de mot de passe
    wert: kpasswd · Port 464
    href: https://datatracker.ietf.org/doc/html/rfc3244
  - label: Nom de service
    wert: service/host@REALM
    href: https://web.mit.edu/kerberos/krb5-latest/doc/admin/princ_dns.html
  - label: Clé de service
    wert: Principal · KVNO · Enctype · Key
    href: https://web.mit.edu/kerberos/krb5-latest/doc/basic/keytab_def.html
  - label: Fenêtre temporelle
    wert: Le Clock Skew relève de la policy du Realm et du client
    href: https://datatracker.ietf.org/doc/html/rfc4120#section-3.2.3
  - label: API applicative
    wert: GSS-API / SSPI · souvent SPNEGO
    href: https://datatracker.ietf.org/doc/html/rfc4121
  - label: Active Directory
    wert: Les contrôleurs de domaine intègrent le KDC
    href: https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview
  - label: Autorisation AD
    wert: PAC en tant qu’Authorization Data de ticket
    href: https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-pac/166d8064-c863-41e1-9c23-edaaa5f36962
werbung:
  - newsletter
ctaThemen:
  - active-directory-entra
translationSourceHash: b5eccfbd21dfaa9b065424961e28940ca63617bba9dbb9f5ab68c1db128df31f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-07T11:07:30.364Z
translationReview: required
---

# Kerberos : tickets, clés et services de confiance

Kerberos permet à un utilisateur ou à un processus de s’authentifier auprès d’un service réseau sans envoyer son mot de passe à ce service. Pour cela, le client et le serveur font confiance à un Key Distribution Center (KDC), qui émet des tickets et des clés de session limités dans le temps. Lors d’une connexion basée sur un mot de passe, une clé à long terme est toutefois dérivée du mot de passe ; la qualité du mot de passe, la pré-authentification et la protection du client restent donc importantes pour la sécurité ([RFC 4120, sections 1.1 et 1.2](https://datatracker.ietf.org/doc/html/rfc4120#section-1.1), [RFC 3961, section 3](https://datatracker.ietf.org/doc/html/rfc3961#section-3)).

Pour les administrateurs de messagerie et d’annuaires, Kerberos est souvent utilisé en complément de [LDAP](/kb/ldap), et non à sa place. LDAP transporte des opérations d’annuaire ; Kerberos peut fournir l’identité d’une session LDAP via SASL. Les interfaces web utilisent HTTP Negotiate, les services Windows SSPI et les applications Unix généralement GSS-API. Une erreur peut donc provenir de DNS, de l’heure, de l’accessibilité du KDC, de l’affectation du Realm, du cache de tickets, du SPN, du compte de service, de la keytab, du type de chiffrement, du PAC ou de la stratégie de délégation, même si l’application ne signale que « Integrated Authentication failed ».

L’explication suit un principal depuis le premier contact avec le KDC, via le TGT et le ticket de service, jusqu’au service cible. Ce n’est qu’ensuite que les extensions Active Directory, la délégation, la dépendance au temps, le diagnostic et la récupération sont approfondis.

## Approche architecturale : un tiers de confiance

Kerberos distribue des clés symétriques par l’intermédiaire d’une instance de confiance. Le KDC a accès aux clés à long terme des principals de son Realm et réunit logiquement deux services : l’Authentication Service, AS, émet un Ticket-Granting Ticket, TGT ; le Ticket-Granting Service, TGS, échange ce TGT contre un ticket destiné à un service applicatif concret. L’échange client/serveur, AP, s’effectue ensuite entre le client et le service. Un service peut normalement vérifier un ticket de service avec sa propre clé, sans rappeler le KDC à chaque connexion ([RFC 4120, sections 1.2 et 3](https://datatracker.ietf.org/doc/html/rfc4120#section-1.2)).

Ce modèle réduit la transmission de mots de passe et les vérifications centrales en ligne, mais crée des domaines de panne clairs. Sans KDC accessible, aucun nouveau ticket ne peut être émis ; les tickets existants peuvent continuer à fonctionner jusqu’à l’expiration de leur validité. En revanche, une base de données KDC ou une clé de Realm compromise met en danger la chaîne de confiance de l’ensemble du Realm. Kerberos n’est donc pas une plateforme de signature de jetons sans état, mais un système distribué composé de l’état du KDC, de caches clients, de clés de service, d’horloges et de services de noms.

| Niveau | Composant technique | État concerné | Preuve administrative |
|---|---|---|---|
| Application | HTTP, SMB, LDAP, base de données, SMTP/IMAP avec SASL ou service propriétaire | Session et autorisation de l’application | procédé d’authentification effectivement choisi et nom cible |
| API d’intégration | GSS-API, Windows SSPI et souvent SPNEGO | Security Context, délégation et Channel Binding | mécanisme négocié, initiateur, accepteur et flags |
| Kerberos AP | `KRB_AP_REQ`, facultatif `KRB_AP_REP`, le cas échéant GSS Wrap/MIC | Ticket de service, Authenticator et clé de session | principal cible, périodes du ticket, Enctype et authentification mutuelle |
| Kerberos KDC | Échanges AS et TGS | Base de données des principals, clés de Realm, policies et ticket flags | résultat KDC, clé sélectionnée, KVNO et événement d’audit |
| Découverte et transport | DNS SRV, UDP/TCP 88, service de mot de passe 464 | Cache du résolveur, mapping de Realm, routage et taille des paquets | KDC réellement choisi, transport, temps de réponse et fallback |
| Persistance | AD DS ou base de données Kerberos, keytabs, caches de credentials et de replay | Clés à long terme, tickets, PAC, réplication et état de replay | état de réplication, contenu du cache, droits de fichiers et procédure de récupération |

Kerberos authentifie des principals et peut fournir l’intégrité ou la confidentialité à un contexte de sécurité GSS. Il ne chiffre pas automatiquement l’intégralité du flux de données utile d’une application et n’offre pas de Perfect Forward Secrecy pour les clés de session distribuées dans le cœur de Kerberos. Les applications peuvent utiliser Kerberos pour authentifier un canal protégé séparément ; [TLS](/kb/tls) reste donc une couche distincte, avec sa propre identité de serveur et sa propre vérification de certificat, pour HTTPS ou LDAPS ([RFC 4120, section 10](https://datatracker.ietf.org/doc/html/rfc4120#section-10), [RFC 4121, sections 2 et 4](https://datatracker.ietf.org/doc/html/rfc4121#section-2)).

## Principals, Realms et matériel de clés

Un principal Kerberos est un nom au sein d’un Realm. Les utilisateurs sont souvent représentés sous la forme `alice@EXAMPLE.CH`, les services sous la forme `HTTP/intranet.example.ch@EXAMPLE.CH`. La partie précédant `@` peut contenir plusieurs composants ; dans le modèle de nommage Kerberos, la casse est fondamentalement significative. Un nom de domaine DNS et un Realm Kerberos sont des espaces de noms différents, même si les Realms ressemblent habituellement à des domaines DNS en majuscules. Les clients ont donc besoin d’une correspondance vérifiable entre un nom d’hôte ou un domaine DNS et un Realm ([RFC 4120, sections 6.1 et 7.2.3](https://datatracker.ietf.org/doc/html/rfc4120#section-6.1), [MIT Kerberos: Mapping hostnames onto realms](https://web.mit.edu/kerberos/krb5-latest/doc/admin/realm_config.html#mapping-hostnames-onto-kerberos-realms)).

Les clés à long terme appartiennent aux principals. Les clés de session sont générées pour un échange limité dans le temps. Un ticket contient notamment le client, le serveur, le Realm, des champs temporels, des flags, une clé de session et des Authorization Data ; sa partie chiffrée est protégée par une clé du service cible. Le client reçoit la même clé de session dans une partie de réponse protégée pour lui. Il peut donc transporter le contenu du ticket, mais ne peut pas modifier lui-même la partie chiffrée pour le service ([RFC 4120, sections 5.3 et 5.4](https://datatracker.ietf.org/doc/html/rfc4120#section-5.3)).

Le Key Version Number, KVNO, distingue les générations d’une clé de principal. L’Encryption Type, Enctype, définit l’algorithme, la longueur de clé, le string-to-key et la somme de contrôle. Un service peut conserver plusieurs entrées de keytab pour le même principal, avec différents KVNO ou Enctypes, afin de traverser une rotation contrôlée. Si le KVNO et l’Enctype du ticket ne correspondent à aucune clé de service disponible, le service ne peut pas déchiffrer le ticket. La présence d’un SPN ne prouve donc pas encore qu’une clé adaptée se trouve sur le système cible ([RFC 4120, section 5.2.9](https://datatracker.ietf.org/doc/html/rfc4120#section-5.2.9), [MIT Kerberos: Keytabs](https://web.mit.edu/kerberos/krb5-latest/doc/basic/keytab_def.html)).

| Objet | Emplacement | Confidentialité | Limite opérationnelle |
|---|---|---|---|
| Clé de principal à long terme | Base de données KDC ; pour le service, également keytab ou magasin de clés du système d’exploitation | très critique | rotation, réplication, KVNO et Enctypes autorisés |
| TGT | Cache de credentials du client ; partie du ticket chiffrée pour `krbtgt/REALM` | proche d’un bearer token, plus clé de session | Lifetime, flags Forwardable/Renewable, isolation du cache |
| Ticket de service | Cache de credentials, puis service cible | déchiffrable uniquement par le service nommé | SPN, compte de service, hôte cible, PAC et Enctype du ticket |
| Authenticator | Nouveau pour chaque requête AP, protégé par la clé de session | protection contre le replay | heure du client, Subkey, séquence et cache de replay côté serveur |
| Keytab | Fichier ou autre magasin de keytab au niveau du service | à protéger comme un mot de passe non interactif | droits de fichiers, distribution, inventaire, rotation et suppression sécurisée |
| Cache de replay | État local de l’accepteur | état d’intégrité | à vérifier par instance de service, hôte et architecture de cluster |

Active Directory stocke les SPN dans l’attribut multivalué `servicePrincipalName` d’un compte utilisateur ou ordinateur. Le SPN relie le nom de service construit par le client au compte précis dont le KDC utilise la clé pour le ticket de service. Un alias, un nom de load balancer ou un compte de service modifié exige donc une affectation SPN consciente. Les SPN en double sont ambigus dans leur périmètre de recherche ; un SPN associé au mauvais compte conduit typiquement au fait que le service ne peut pas déchiffrer le ticket émis ([Microsoft: Service principal names](https://learn.microsoft.com/en-us/windows/win32/ad/service-principal-names), [Microsoft: setspn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn)).

Une keytab n’est pas un fichier d’export contenant une identité restaurable à volonté, mais une collection de véritables clés à long terme. Chaque entrée contient le principal, le KVNO, l’Enctype et la clé. Toute personne pouvant lire le fichier peut se faire passer pour ce principal. MIT recommande un stockage local restrictif et aucune transmission non protégée. Dans le cadre d’une interopérabilité avec Active Directory, [`ktpass`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/ktpass) peut relier principal, compte et keytab ; les paramètres choisis peuvent alors influencer le mot de passe, le salt, le KVNO ou l’Enctype et doivent faire partie d’un processus de rotation testé, et non d’une simple fiche d’installation ponctuelle ([MIT Kerberos: Application servers](https://web.mit.edu/kerberos/krb5-latest/doc/admin/appl_servers.html)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-kerberos.svg?v=20260813" title="Interaktive Infografik: Kerberos-Discovery, AS-, TGS- und AP-Fluss, Tickets, Schlüssel, Active-Directory-PAC und Delegationsgrenzen" loading="lazy">
  <a href="/images/kb-interaktiv-kerberos.svg?v=20260813">Ouvrir l’infographie sur le flux du protocole Kerberos et ses limites de confiance</a>
</iframe>

Une fois le principal, le Realm et les clés clarifiés, le flux de tickets peut être lu comme trois conversations successives : AS, TGS et enfin le service applicatif.

## Flux protocolaire : AS, TGS et AP

L’échange AS commence par `KRB_AS_REQ`. Un KDC peut répondre par `KDC_ERR_PREAUTH_REQUIRED` et indiquer les mécanismes de pré-authentification pris en charge. Avec le timestamp chiffré largement répandu, le client prouve la connaissance de sa clé à long terme avant que le KDC émette un TGT. `KRB_AS_REP` contient le TGT, chiffré pour le TGS, et une partie de réponse pour le client. PKINIT remplace cette première preuve par une cryptographie à clé publique et des certificats ; Kerberos FAST peut renforcer la pré-authentification dans un tunnel protégé ([RFC 4120, sections 3.1 et 5.2.7](https://datatracker.ietf.org/doc/html/rfc4120#section-3.1), [RFC 4556](https://datatracker.ietf.org/doc/html/rfc4556), [RFC 6113, section 5](https://datatracker.ietf.org/doc/html/rfc6113#section-5)).

Dans l’échange TGS, le client envoie `KRB_TGS_REQ` avec le TGT, un Authenticator et le service principal demandé. Le KDC vérifie la policy du Realm, les ticket flags, le principal cible et les clés prises en charge, puis fournit dans `KRB_TGS_REP` un ticket de service ainsi qu’une nouvelle clé de session client/service. Le mot de passe utilisateur n’est alors plus nécessaire. Un TGT déjà présent peut donc être utilisé pour de nombreux services, jusqu’à ce que la Lifetime, la policy ou l’état du cache nécessitent une nouvelle authentification initiale ([RFC 4120, section 3.3](https://datatracker.ietf.org/doc/html/rfc4120#section-3.3)).

Lors de l’échange AP, le client présente au service `KRB_AP_REQ` : le ticket de service et un Authenticator récent protégé par la clé de session. Le service déchiffre le ticket avec sa clé à long terme, vérifie le principal cible, les heures, les flags et l’état de replay, puis en obtient la clé de session. Si le client demande une authentification mutuelle, le service répond par `KRB_AP_REP`. Seule cette réponse prouve cryptographiquement au client que l’autre partie possède la clé de service ([RFC 4120, sections 3.2 et 5.5](https://datatracker.ietf.org/doc/html/rfc4120#section-3.2)).

| Échange | Requête | Réponse de succès | Entrées critiques | Limite d’erreur typique |
|---|---|---|---|---|
| AS | `KRB_AS_REQ` | `KRB_AS_REP` avec TGT | Principal client, Realm, Pre-Auth, Enctypes autorisés | compte inconnu, Pre-Auth, heure, clé client absente |
| TGS | `KRB_TGS_REQ` | `KRB_TGS_REP` avec ticket de service | TGT, Authenticator, SPN, flags, clés cible | SPN, referral de Realm, policy, délégation, Enctype |
| AP | `KRB_AP_REQ` | facultatif `KRB_AP_REP` | Ticket de service, Authenticator, clé de service, cache de replay | mauvais compte de service, keytab/KVNO, heure, replay |
| Erreur | une des requêtes | `KRB_ERROR` | Code d’erreur plus e-data facultatives et heure du serveur | conserver le code numérique et la phase du protocole concernée |

La pré-authentification ne protège pas contre toute attaque hors ligne, mais elle en modifie les conditions. Les comptes qui n’exigent pas de pré-authentification peuvent fournir une réponse AS dont la partie protégée par mot de passe peut être vérifiée hors ligne. Les tickets de service peuvent également être analysés hors ligne ; les mots de passe faibles des comptes de service et RC4 aggravent ce risque. La protection effective repose sur la pré-authentification, des clés de service fortes et aléatoires ou un gMSA, des Enctypes modernes, des droits limités et des données d’audit, et non sur la seule présence de Kerberos ([RFC 6113, section 1](https://datatracker.ietf.org/doc/html/rfc6113#section-1), [RFC 8429, section 5](https://datatracker.ietf.org/doc/html/rfc8429#section-5), [Microsoft: Detect and remediate RC4 usage](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos)).

## Tickets, temps, flags et transport

Un ticket possède `authtime`, facultativement `starttime`, `endtime` et, pour les tickets renouvelables, `renew-till`. Le TGT n’est pas un jeton d’accès universel : il est destiné au TGS et sert à obtenir d’autres tickets. Un ticket de service est destiné à un seul principal serveur. Lifetime et Renewal relèvent de la policy du KDC et peuvent être limités par Realm ou par compte. Des valeurs fixes telles que « dix heures » sont donc des valeurs par défaut de produit, et non une propriété du standard Kerberos ([RFC 4120, sections 2.3 et 5.3](https://datatracker.ietf.org/doc/html/rfc4120#section-2.3)).

Des flags tels que `forwardable`, `forwarded`, `proxiable`, `proxy`, `renewable`, `initial`, `pre-authent` et `ok-as-delegate` modifient l’utilisation possible d’un ticket. Ce ne sont pas de simples champs de diagnostic décoratifs. Un double hop peut échouer bien que le premier ticket de service soit valide, parce qu’un flag requis ou une policy KDC manque. À l’inverse, un TGT forwardable étend l’impact d’un service compromis. Les ticket flags font donc partie de toute preuve de délégation et d’incident ([RFC 4120, section 2](https://datatracker.ietf.org/doc/html/rfc4120#section-2)).

Les Authenticators et certains mécanismes de pré-authentification utilisent le temps pour vérifier la fraîcheur. RFC 4120 laisse l’écart toléré à la policy locale. Active Directory utilise cinq minutes par défaut ; cette tolérance ne remplace pas une synchronisation précise de l’heure. Le client, le KDC et le service cible sont déterminants : un client peut obtenir un ticket et néanmoins échouer auprès du service avec `KRB_AP_ERR_SKEW` si son horloge diffère ([RFC 4120, sections 3.2.3 et 7.5.1](https://datatracker.ietf.org/doc/html/rfc4120#section-3.2.3), [Microsoft: Kerberos troubleshooting guidance](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-authentication-troubleshooting-guidance), [Microsoft: Windows Time Service](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service)).

Kerberos utilise le port 88 via UDP et TCP. RFC 4120 exige la prise en charge de TCP et décrit UDP comme facultatif ; des réponses UDP trop volumineuses peuvent déclencher une nouvelle tentative TCP avec `KRB_ERR_RESPONSE_TOO_BIG`. Le PAC, les appartenances à des groupes et d’autres Authorization Data augmentent la taille des tickets. Un pare-feu qui n’autorise que de petits tests UDP ou bloque TCP 88 peut donc provoquer des pannes dépendantes de l’utilisateur ou du groupe ([RFC 4120, section 7.2.1](https://datatracker.ietf.org/doc/html/rfc4120#section-7.2.1), [Microsoft: Kerberos KDC configuration keys](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-protocol-registry-kdc-configuration-keys)).

Le ticket de service n’est que la preuve cryptographique. Pour que HTTP, LDAP ou SMB l’utilisent, GSS-API, SSPI et SPNEGO intègrent Kerberos dans le protocole applicatif concerné.

## Intégration dans les protocoles applicatifs

GSS-API fournit aux applications un Security Context indépendant du mécanisme ; RFC 4121 définit à cette fin le mécanisme Kerberos V5. Windows SSPI joue un rôle comparable. SPNEGO négocie entre les mécanismes GSS proposés et apparaît souvent dans HTTP sous la forme `Negotiate`. Le nom d’en-tête visible ne prouve pas automatiquement que Kerberos a été choisi à la place de NTLM. Le diagnostic doit capturer le mécanisme réellement négocié et le Target Name demandé ([RFC 4121](https://datatracker.ietf.org/doc/html/rfc4121), [RFC 4178](https://datatracker.ietf.org/doc/html/rfc4178), [Microsoft: Kerberos authentication overview](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview)).

SASL GSS-API intègre le même mécanisme Kerberos dans les protocoles applicatifs. C’est par exemple possible avec [LDAP](/kb/ldap), IMAP ou SMTP lorsque le client et le serveur proposent le mécanisme. Après un contexte GSS réussi, les SASL Security Layers peuvent fournir l’intégrité ou la confidentialité. Leur utilisation effective et leur interaction avec TLS relèvent de la configuration de l’application concernée ; `GSSAPI` dans la liste des capacités ne prouve à lui seul ni un SPN ni une Channel Protection ([RFC 4752](https://datatracker.ietf.org/doc/html/rfc4752)).

Le client compose le nom de service à partir de la Service Class et de l’hôte cible. Pour HTTP, `HTTP/fqdn` est typiquement pertinent, et pour LDAP `ldap/fqdn`. Un alias, un CNAME, une recherche inverse, un proxy, un nom de cluster et le nom d’hôte dans l’URL peuvent modifier l’identité construite. Le client doit demander le ticket pour le même principal dont le service accepteur possède la clé. Un load balancer ne résout pas cette liaison ; toutes les instances backend nécessitent une identité de service et une stratégie de clés cohérentes ([MIT Kerberos: Application servers – DNS](https://web.mit.edu/kerberos/krb5-latest/doc/admin/appl_servers.html#getting-dns-information-correct), [Microsoft: Service principal names](https://learn.microsoft.com/en-us/windows/win32/ad/service-principal-names)).

## Active Directory en tant que Realm Kerberos

Dans Active Directory Domain Services, le KDC est intégré au contrôleur de domaine. Les comptes utilisateur, ordinateur et service sont des principals Kerberos ; l’annuaire fournit les clés, les SPN, les groupes, les account flags et les policies. Le compte `krbtgt` représente le TGS du domaine. La disponibilité du KDC et la cohérence de ses données suivent donc DC Locator, DNS, la réplication AD, les sites et le modèle de récupération d’AD DS, et non un protocole de cluster Kerberos distinct ([Microsoft: Kerberos authentication overview](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview), [MS-KILE: Kerberos V5 Synopsis](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/b4af186e-b2ff-43f9-b18e-eedb366abf13), [DC Locator](/kb/ldap#active-directory-als-ldap-serverprofil)).

AD ajoute généralement aux tickets un Privilege Attribute Certificate, PAC, comme Authorization Data. Il peut notamment contenir des SID, des appartenances à des groupes, des informations de profil et de policy ainsi que des signatures. Le KDC génère et signe ces données ; le service les utilise pour l’autorisation Windows ou les fait éventuellement valider. L’authentification Kerberos de base et l’autorisation AD constituent donc des affirmations distinctes : un principal cryptographiquement valide ne possède pas automatiquement l’autorisation souhaitée ([MS-PAC](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-pac/166d8064-c863-41e1-9c23-edaaa5f36962), [MS-KILE: PAC Generation](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/c25d48df-67f0-4c5f-9e46-27a7d5710909)).

Le matériel de clés et les attributs SPN se répliquent avec Active Directory. Après une modification de compte, de mot de passe ou de SPN, différents DC peuvent donc brièvement utiliser des états différents. Si le client utilise un DC pour TGS et l’administrateur un autre DC pour le contrôle, une erreur paraît intermittente. Une preuve fiable indique le KDC émetteur, le DC cible de la requête d’annuaire, le KVNO, l’Enctype du ticket et l’état de réplication. Cela s’applique particulièrement aux rotations manuelles de keytab et aux services répartis sur plusieurs sites.

La sélection d’Enctype est l’intersection entre l’offre client, la policy KDC, les clés du compte cible et la prise en charge du service. Un logiciel compatible AES ne suffit pas si le compte ne possède pas les clés correspondantes ou si `msDS-SupportedEncryptionTypes` et la policy de domaine les excluent. RFC 8429 considère RC4 et 3DES comme obsolètes pour Kerberos ; Microsoft documente l’inventaire via les Security Events 4768 et 4769. Une migration commence par la mesure et la génération de clés, non par la désactivation globale d’un bit ([RFC 8429](https://datatracker.ietf.org/doc/html/rfc8429), [RFC 8009](https://datatracker.ietf.org/doc/html/rfc8009), [Microsoft: Detect and remediate RC4 usage](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos)).

Pour les services Windows, les Group Managed Service Accounts, gMSA, réduisent l’administration manuelle des mots de passe et des SPN. Ils ne remplacent toutefois pas la vérification de l’identité sous laquelle le processus s’exécute réellement, des hôtes autorisés à lire le Managed Password et des SPN enregistrés sur le compte. Pour les appliances ou les services Unix, une keytab reste souvent nécessaire ; sa rotation doit être synchronisée avec le compte AD ([Microsoft: Service Accounts](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-service-accounts)).

Au sein d’un domaine AD, ce chemin est direct. Lors d’un accès au-delà des limites de Realm ou de domaine, des tickets Cross-Realm et éventuellement plusieurs étapes intermédiaires s’ajoutent.

## Relations d’approbation et chemins Cross-Realm

Une relation d’approbation entre Realms ne fusionne pas les bases de données de principals. L’authentification Cross-Realm utilise des TGT pour `krbtgt/TARGET@SOURCE` et éventuellement plusieurs Realms intermédiaires. Le client suit un chemin de tickets jusqu’à atteindre le Realm du service cible. RFC 6806 ajoute les referrals et la canonisation des noms, tels qu’ils sont notamment employés dans les environnements AD. Le chemin visible dans le cache est donc plus parlant que l’affirmation générale « la relation d’approbation est au vert » ([RFC 4120, sections 1.1 et 3.3.3](https://datatracker.ietf.org/doc/html/rfc4120#section-3.3.3), [RFC 6806](https://datatracker.ietf.org/doc/html/rfc6806)).

La direction de la relation d’approbation, la transitivité, Selective Authentication, le filtrage SID, le Name Suffix Routing et les Enctypes disponibles limitent ce qu’un TGT Cross-Realm permet concrètement. Les SPN doivent être trouvables dans la forêt correcte et être univoques. Un résultat local de `setspn -Q` ne prouve pas une unicité à l’échelle de la forêt ; [`setspn`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn) propose à cet effet des options de forêt et de domaine. Le diagnostic documente chaque TGT de referral, et non seulement la dernière erreur de ticket de service ([Microsoft: Windows Authentication Concepts](https://learn.microsoft.com/en-us/windows-server/security/windows-authentication/windows-authentication-concepts), [Microsoft: setspn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn)).

Un ticket vers le frontend n’autorise pas encore automatiquement celui-ci à contacter un backend au nom de l’utilisateur. C’est précisément là que commence le problème du double hop.

## Délégation et double hop

Lors d’un échange AP normal, un frontend ne reçoit pas de clé utilisateur librement utilisable. S’il doit accéder à un backend au nom de l’utilisateur, il nécessite un modèle de délégation. La délégation par Forwarded-TGT, ou unconstrained delegation, transmet au service un TGT réutilisable et étend son impact bien au-delà d’un seul backend. Un frontend compromis peut l’utiliser pour obtenir des tickets vers d’autres services ; cette forme constitue donc une extension majeure de confiance ([RFC 4120, sections 2.5 et 2.6](https://datatracker.ietf.org/doc/html/rfc4120#section-2.5), [MS-SFU: Protocol Overview](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/1fb9caca-449f-4183-8f7a-1a5fc7e7290a)).

Les extensions Service-for-User de Microsoft répartissent le problème. Avec S4U2self, un service peut obtenir pour lui-même un ticket au nom d’un utilisateur, par exemple après une autre authentification frontend. Avec S4U2proxy, il demande selon la policy KDC un ticket vers un second service au nom de cet utilisateur. La constrained delegation classique stocke les cibles autorisées sur le compte frontend ; la resource-based constrained delegation enregistre les appelants autorisés sur le compte de ressource. Dans les deux cas, le SPN cible, les ticket flags, les paramètres de compte et le chemin de relation d’approbation font partie de la décision ([MS-SFU: Overview](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/36d103d2-61a6-42d5-a725-74de3205cdaf), [MS-SFU: Introduction](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/8ee85a47-7526-4184-a7c5-25a5e4155d7d)).

La délégation est une autorisation de propagation d’identité, et non une simple option de compatibilité. L’administrateur inventorie le principal frontend, les SPN backend, les catégories d’utilisateurs autorisées, la Protocol Transition, les limites de confiance et la portée cible techniquement la plus réduite. Un premier hop réussi ne prouve pas le second ; inversement, un test backend direct dans le contexte utilisateur contourne la limite de délégation et peut donner un faux résultat positif.

## Modèle d’exploitation, monitoring et récupération

Après le flux de tickets et la délégation, la question opérationnelle se pose : quel composant Kerberos doit être disponible, quelle preuve montre son état et que peut-on restaurer en urgence ? Le tableau attribue ces questions aux rôles concernés.

| Rôle | État à surveiller | Métrique ou preuve directrice | Angle mort fréquent |
|---|---|---|---|
| Client | Mapping de Realm, sélection KDC, horloge et cache de credentials | Latence AS/TGS, Lifetime du cache, KDC et Error Code | Le test utilise un autre nom DNS ou une autre session de connexion utilisateur |
| KDC | Clés de principal, policy, réplication, relation d’approbation et audit | 4768/4769/4771, taux d’erreur par code, Enctype et DC émetteur | Erreur globale sans SPN cible ni offre client |
| Service | Compte de service, SPN, keytab/key store, cache de replay et horloge | Succès AP, KVNO, Enctype du ticket, principal cible | Port ouvert, mais processus exécuté sous une autre identité |
| Frontend avec délégation | Policy S4U/forwarding et cibles backend | Premier et second hop séparés, chemin de délégation et ticket flags | Test backend direct contourne le double hop |
| Exploitation Realm/forêt | Base de données KDC, clés de Realm, réplication AD et récupération | État des clés répliquées, configuration sauvegardée, restauration testée | La keytab de service et la génération KDC divergent |

Les caches de tickets constituent un état opérationnel. Un processus utilisateur, un compte de service, un conteneur ou une Windows Logon Session peut voir un cache différent de celui du shell d’administration interactif. Supprimer et redemander un ticket est un test ciblé, mais pas une réparation de la cause sous-jacente liée au SPN, à la clé ou à la réplication. Avant le purge, le principal, le SPN cible, le KDC, le KVNO, l’Enctype, les flags et les champs temporels sont relevés ; sinon, la meilleure preuve de l’erreur disparaît.

Windows journalise les demandes de TGT sous 4768, les tickets de service sous 4769, les erreurs de pré-authentification sous 4771 et les autres erreurs AS sous 4772, lorsque les Advanced Audit Policies appropriées sont actives. Le volume est élevé sur les KDC. Le monitoring nécessite donc une agrégation par Result Code, client, service cible, DC et Enctype ainsi que des baselines, au lieu de traiter chaque requête TGS réussie comme une alerte ([Microsoft: Advanced Audit Policy Configuration](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/advanced-audit-policy-configuration)).

La récupération dépend de l’implémentation KDC. Avec AD DS, les données KDC et `krbtgt` relèvent du modèle de sauvegarde System State et de Forest Recovery ; un fichier de base de données quelconque ou une exportation LDIF ne constitue pas une sauvegarde Kerberos valide. Les exploitants de Realms MIT autonomes doivent sauvegarder ensemble la base de données KDC, le matériel Stash/Master Key, la configuration, les ACL et la réplication. Les keytabs de service doivent aussi être inventoriées et contrôlées quant à la cohérence du KVNO et des clés après une restauration ([Microsoft: Back up the System State data](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-backing-up-system-state), [MIT Kerberos: Backups of secure hosts](https://web.mit.edu/kerberos/krb5-latest/doc/admin/admin_commands/kdb5_util.html)).

Pour le dépannage, le chemin est vérifié à l’envers : nom cible et SPN, ticket de service existant, réponse TGS, TGT, découverte du Realm, DNS et heure.

## Outils de diagnostic

Le diagnostic commence depuis la même zone réseau, avec le même nom cible, le même Realm, le même contexte utilisateur ou de service et la même Logon Session que l’application. Les données de test utilisent des noms réservés. Les sorties de tickets et de keytabs peuvent révéler des principals et de l’infrastructure ; le matériel de clés lui-même ne doit jamais apparaître dans des tickets, des discussions ou des arguments de processus.

### Déterminer le KDC et le service de mot de passe via DNS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kerberos-Dienstsuche">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName -Type SRV "_kerberos._tcp.example.ch"
Resolve-DnsName -Type SRV "_kerberos._udp.example.ch"
Resolve-DnsName -Type SRV "_kpasswd._tcp.example.ch"</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig +short SRV _kerberos._tcp.example.ch
dig +short SRV _kerberos._udp.example.ch
dig +short SRV _kpasswd._tcp.example.ch</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) et [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) indiquent la priorité, le poids, le port et la cible. Il faut ensuite vérifier la résolution A/AAAA et l’accessibilité de chaque cible fournie. RFC 4120 définit la découverte DNS SRV, mais autorise une configuration locale de Realm ; un test SRV vide ne prouve donc une erreur que si le client concret utilise la découverte DNS ([RFC 4120, section 7.2.3](https://datatracker.ietf.org/doc/html/rfc4120#section-7.2.3), [RFC 2782](https://datatracker.ietf.org/doc/html/rfc2782)).

### Comparer l’heure et l’accessibilité TCP

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kerberos-Zeit- und Portprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">w32tm /query /status
w32tm /stripchart /computer:dc1.example.ch /samples:5 /dataonly
Test-NetConnection dc1.example.ch -Port 88</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">timedatectl status
chronyc tracking
nc -vz dc1.example.ch 88</code></pre>
  </div>
</div>

[`w32tm`](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings#w32tm-command-line-tool) et [`timedatectl`](https://man7.org/linux/man-pages/man1/timedatectl.1.html) indiquent la source et l’état de synchronisation ; [`chronyc`](https://chrony-project.org/doc/4.7/chronyc.html) complète avec les données d’offset et de suivi de Chrony. [`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) ou [`nc`](https://man.openbsd.org/nc) prouvent uniquement TCP 88. UDP, le protocole KDC, le Realm et la pré-authentification exigent un véritable test AS/TGS.

### Redemander et afficher les tickets de manière ciblée

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kerberos-Ticketprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">klist purge
klist get HTTP/intranet.example.ch
klist</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">kdestroy
KRB5_TRACE=/dev/stderr kinit alice@EXAMPLE.CH
kvno HTTP/intranet.example.ch@EXAMPLE.CH
klist -ef</code></pre>
  </div>
</div>

Le [`klist`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/klist) de Windows fonctionne dans le contexte de la Logon Session sélectionnée. Avec MIT Kerberos, [`kdestroy`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kdestroy.html) supprime, [`kinit`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kinit.html) et [`kvno`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kvno.html) obtiennent, tandis que [`klist`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/klist.html) affiche les credentials. [`KRB5_TRACE`](https://web.mit.edu/kerberos/krb5-current/doc/user/user_config/kerberos.html) rend visibles la sélection KDC et le chemin protocolaire. Avant `purge` ou `kdestroy`, un ticket d’erreur existant doit être documenté.

### Vérifier conjointement SPN et clé de service

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kerberos-SPN- und Keytabprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">setspn -Q HTTP/intranet.example.ch
setspn -X -F

Get-ADUser svc-web -Properties @(
  "servicePrincipalName"
  "msDS-SupportedEncryptionTypes"
)</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">klist -kte /etc/krb5.keytab
kvno -k /etc/krb5.keytab \
  HTTP/intranet.example.ch@EXAMPLE.CH</code></pre>
  </div>
</div>

[`setspn`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn) indique le compte cible AD et recherche avec `-X -F` les doublons à l’échelle de la forêt. [`Get-ADUser`](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser) lit les SPN et les Enctypes déclarés ; une valeur manquante a une sémantique de fallback propre au produit et ne doit pas être interprétée globalement comme « pas d’AES ». Le [`klist`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/klist.html) de MIT affiche le principal, le KVNO et l’Enctype de la keytab ; [`kvno -k`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kvno.html) demande un ticket et le valide par rapport à la keytab indiquée.

### Associer les erreurs à une phase protocolaire

| Code ou symptôme | Phase et limite fréquente | Preuve suivante |
|---|---|---|
| `KDC_ERR_C_PRINCIPAL_UNKNOWN` | AS : principal client inconnu dans le Realm sélectionné | Mapping du Realm, UPN/principal, KDC émetteur et réplication |
| `KDC_ERR_PREAUTH_FAILED` | AS : clé, mot de passe, certificat ou procédé de Pre-Auth rejeté | Heure du client, type de Pre-Auth, clé du compte et audit KDC 4771 |
| `KDC_ERR_S_PRINCIPAL_UNKNOWN` | TGS : SPN cible introuvable ou ne pouvant pas être résolu de façon univoque | SPN exact demandé, recherche dans toute la forêt et Realm cible |
| `KDC_ERR_ETYPE_NOSUPP` | AS/TGS : aucune intersection commune Enctype/clé | Offre du client, policy KDC, clés de compte, keytab et 4768/4769 |
| `KRB_AP_ERR_MODIFIED` | AP : le ticket ne correspond pas à la clé du service répondant | Compte SPN, identité du processus, keytab, KVNO, Enctype et nœud backend |
| `KRB_AP_ERR_SKEW` | AS/AP : heure hors de la tolérance | Heure du client, KDC et service, ainsi que leur source de temps |
| `KRB_AP_ERR_TKT_EXPIRED` | AP : ticket hors de la fenêtre de validité | Cache, `endtime`, Renewal, heure du client et nouvelle initialisation |
| `KDC_ERR_BADOPTION` | TGS/S4U : flag, délégation ou policy non autorisé | Flag Forwardable, compte frontend, SPN backend et configuration de délégation |
| Ticket Kerberos présent, application utilisant NTLM | Négociation de mécanisme ou Target Name | Résultat SPNEGO, URL/FQDN, policy de zone/client et SPN effectivement construit |

Les codes d’erreur sont normalisés dans RFC 4120 ; Windows ajoute des informations de statut et d’audit. L’automatisation doit conserver le code numérique, la phase, le KDC, le principal client et le principal cible. Le texte libre seul n’est ni stable ni univoque. Pour l’analyse de paquets, [Wireshark](https://www.wireshark.org/docs/dfref/k/kerberos.html) peut filtrer sur `kerberos` ; les parties chiffrées des tickets restent volontairement illisibles sans les clés appropriées ([RFC 4120, section 7.5.9](https://datatracker.ietf.org/doc/html/rfc4120#section-7.5.9), [Microsoft: Kerberos troubleshooting guidance](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-authentication-troubleshooting-guidance)).

## Histoire technique

Kerberos est né au début des années 1980 dans le MIT Project Athena. Le protocole s’appuie conceptuellement sur les travaux de tiers de confiance de Needham et Schroeder, ainsi que de Denning et Sacco. Les versions 1 à 4 ont été développées dans le cadre d’Athena ; la version 4 fut la première version largement déployée. Le nom renvoie à Kerberos, le gardien à plusieurs têtes de la mythologie grecque, et symbolise le rôle central de confiance du KDC ([RFC 4120, section 1](https://datatracker.ietf.org/doc/html/rfc4120#section-1), [MIT Kerberos Consortium: Documentation](https://kerberos.org/docs/index.html)).

Kerberos V5 a supprimé les limites de la version 4 concernant le naming, les durées de vie des tickets, la cryptographie, le Cross-Realm et l’extensibilité. RFC 1510 a standardisé V5 en 1993. RFC 4120 a remplacé cette spécification en 2005 avec des clarifications et une description ASN.1 complète. La famille de protocoles a ensuite été étendue de manière modulaire, notamment avec PKINIT, le framework de pré-authentification et FAST, GSS-API, les referrals ainsi que de nouveaux profils AES ([RFC 1510](https://datatracker.ietf.org/doc/html/rfc1510), [RFC 4120](https://datatracker.ietf.org/doc/html/rfc4120), [RFC 4556](https://datatracker.ietf.org/doc/html/rfc4556), [RFC 6113](https://datatracker.ietf.org/doc/html/rfc6113)).

Microsoft a fait de Kerberos V5 le protocole central d’authentification de domaine avec Windows 2000 et l’a associé aux principals AD, aux SPN, au PAC, à SSPI, aux Trust Referrals et aux extensions de délégation. Parallèlement, MIT Kerberos, Heimdal et d’autres implémentations sont restés interopérables grâce aux protocoles IETF et à GSS-API. La cryptographie a évolué de DES, puis RC4, vers les profils AES ; RFC 8429 a engagé en 2018 le remplacement de 3DES et RC4. Cette histoire technique explique pourquoi les anciens équipements, les vieux comptes de service et les Trust Keys révèlent encore aujourd’hui des limites d’Enctype, sans que l’article ne fixe un état fugitif des versions de produits ([MS-KILE](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/), [RFC 3962](https://datatracker.ietf.org/doc/html/rfc3962), [RFC 8009](https://datatracker.ietf.org/doc/html/rfc8009), [RFC 8429](https://datatracker.ietf.org/doc/html/rfc8429)).

## Sources

- [IETF RFC 3244 – Microsoft Windows 2000 Kerberos Change Password and Set Password Protocols](https://datatracker.ietf.org/doc/html/rfc3244)
- [MIT Kerberos – Mapping Hostnames onto Kerberos Realms](https://web.mit.edu/kerberos/krb5-latest/doc/admin/princ_dns.html)
- [IANA – Noms et ports de service Kerberos](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=kerberos)
- [RFC 4120 – The Kerberos Network Authentication Service (V5)](https://datatracker.ietf.org/doc/html/rfc4120) – Architecture, messages, tickets, flags, codes d’erreur, transport et modèle de sécurité.
- [RFC 3961, section 3](https://datatracker.ietf.org/doc/html/rfc3961)
- [RFC 4121 – Kerberos V5 GSS-API Mechanism](https://datatracker.ietf.org/doc/html/rfc4121) – Contexte GSS, jetons, intégrité et confidentialité.
- [MIT Kerberos: Mapping hostnames onto realms](https://web.mit.edu/kerberos/krb5-latest/doc/admin/realm_config.html)
- [MIT Kerberos – Keytabs](https://web.mit.edu/kerberos/krb5-latest/doc/basic/keytab_def.html) – Contenu des keytabs, KVNO, Enctype et clé.
- [Microsoft – Service principal names](https://learn.microsoft.com/en-us/windows/win32/ad/service-principal-names) – Unicité des SPN et liaison aux comptes de service.
- [Microsoft – setspn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn) – Requête SPN, enregistrement et recherche de doublons.
- [Microsoft – ktpass](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/ktpass) – Interopérabilité AD/keytab.
- [MIT Kerberos – Application servers](https://web.mit.edu/kerberos/krb5-latest/doc/admin/appl_servers.html) – Protection des keytabs, rotation, heure et DNS.
- [RFC 4556 – PKINIT](https://datatracker.ietf.org/doc/html/rfc4556) – Pré-authentification à clé publique.
- [RFC 6113 – Generalized Framework for Kerberos Pre-Authentication](https://datatracker.ietf.org/doc/html/rfc6113) – Framework de Pre-Auth et FAST.
- [RFC 8429 – Deprecate 3DES and RC4 in Kerberos](https://datatracker.ietf.org/doc/html/rfc8429) – IETF Best Current Practice pour les Enctypes obsolètes.
- [Microsoft – Detect and remediate RC4 usage](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos) – Inventaire des Enctypes, clés de compte et événements.
- [Microsoft – Kerberos authentication troubleshooting guidance](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-authentication-troubleshooting-guidance) – Codes d’erreur Windows, heure, SPN et double hop.
- [Microsoft – Windows Time Service](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service) – Service de temps et dépendance Kerberos.
- [Microsoft – Kerberos KDC registry and protocol settings](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-protocol-registry-kdc-configuration-keys) – Taille UDP, fallback TCP et diagnostic KDC.
- [RFC 4178 – SPNEGO](https://datatracker.ietf.org/doc/html/rfc4178) – Négociation des mécanismes GSS.
- [Microsoft – Kerberos authentication overview](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview) – Architecture Windows, KDC, SSPI et authentification mutuelle.
- [RFC 4752 – SASL GSS-API Mechanism](https://datatracker.ietf.org/doc/html/rfc4752) – Intégration dans LDAP, IMAP, SMTP et d’autres protocoles SASL.
- [MS-KILE – Kerberos V5 Synopsis](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/b4af186e-b2ff-43f9-b18e-eedb366abf13) – Échanges AS, TGS et AP sous Windows.
- [MS-PAC – Privilege Attribute Certificate](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-pac/166d8064-c863-41e1-9c23-edaaa5f36962) – Groupes, SID, profils, policies et signatures.
- [MS-KILE – PAC Generation](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/c25d48df-67f0-4c5f-9e46-27a7d5710909) – Génération des PAC Authorization Data.
- [RFC 8009 – AES with HMAC-SHA2 for Kerberos 5](https://datatracker.ietf.org/doc/html/rfc8009) – Enctypes AES-SHA2.
- [Microsoft – Service Accounts](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-service-accounts) – gMSA, gestion des mots de passe et des SPN.
- [RFC 6806 – Kerberos Principal Name Canonicalization and Cross-Realm Referrals](https://datatracker.ietf.org/doc/html/rfc6806) – Referrals et canonisation des noms.
- [Microsoft – Windows Authentication Concepts](https://learn.microsoft.com/en-us/windows-server/security/windows-authentication/windows-authentication-concepts) – Relations d’approbation, Protocol Transition et constrained delegation.
- [MS-SFU – Protocol Overview](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/1fb9caca-449f-4183-8f7a-1a5fc7e7290a) – Flux de délégation et risque des TGT transmis.
- [MS-SFU – Service for User Overview](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/36d103d2-61a6-42d5-a725-74de3205cdaf) – S4U2self et S4U2proxy.
- [MS-SFU – Introduction](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/8ee85a47-7526-4184-a7c5-25a5e4155d7d) – Protocol Transition et constrained delegation.
- [Microsoft – Advanced Audit Policy Configuration](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/advanced-audit-policy-configuration) – Événements 4768, 4769, 4771 et 4772.
- [Microsoft – Back up the System State data](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-backing-up-system-state) – Sauvegarde AD DS/KDC.
- [MIT Kerberos – kdb5_util](https://web.mit.edu/kerberos/krb5-latest/doc/admin/admin_commands/kdb5_util.html) – Base de données KDC et sauvegarde avec MIT Kerberos.
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – Requêtes DNS SRV sous Windows.
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html) – Requêtes DNS SRV sous systèmes Unix.
- [RFC 2782 – A DNS RR for specifying the location of services](https://datatracker.ietf.org/doc/html/rfc2782) – Priorité SRV, poids, port et cible.
- [Microsoft – w32tm](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings) – Source de temps Windows et diagnostic d’offset.
- [Linux man-pages – timedatectl(1)](https://man7.org/linux/man-pages/man1/timedatectl.1.html) – État de la synchronisation temporelle sous Linux.
- [Chrony – chronyc](https://chrony-project.org/doc/4.7/chronyc.html) – Diagnostic d’offset, de source et de suivi.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – Diagnostic des connexions TCP sous Windows.
- [OpenBSD – nc(1)](https://man.openbsd.org/nc) – Test de port TCP sous systèmes Unix.
- [Microsoft – klist](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/klist) – Caches de tickets Windows et test de ticket de service.
- [MIT Kerberos – kdestroy](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kdestroy.html) – Supprimer le cache de credentials.
- [MIT Kerberos – kinit](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kinit.html) – Obtention de TGT et options de Pre-Auth.
- [MIT Kerberos – kvno](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kvno.html) – Ticket de service et validation de keytab.
- [MIT Kerberos – klist](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/klist.html) – Affichage du cache de credentials et des keytabs.
- [MIT Kerberos – kerberos environment](https://web.mit.edu/kerberos/krb5-current/doc/user/user_config/kerberos.html) – `KRB5_TRACE` et variables de cache/keytab.
- [Microsoft Learn – Get-ADUser](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser) – Attributs SPN et Enctype d’un compte de service.
- [Wireshark – Kerberos display filter reference](https://www.wireshark.org/docs/dfref/k/kerberos.html) – Champs de protocole et filtres d’affichage.
- [MIT Kerberos Consortium – Documentation](https://kerberos.org/docs/index.html) – Origine dans le Project Athena et histoire des versions.
- [RFC 1510 – The Kerberos Network Authentication Service (V5)](https://datatracker.ietf.org/doc/html/rfc1510) – Spécification historique du noyau V5 de 1993.
- [MS-KILE – Kerberos Protocol Extensions](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/) – Kerberos Active Directory et extensions Microsoft.
- [RFC 3962 – AES Encryption for Kerberos 5](https://datatracker.ietf.org/doc/html/rfc3962) – Profils AES-SHA1.
