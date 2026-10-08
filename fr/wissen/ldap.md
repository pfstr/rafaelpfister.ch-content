---
title: "LDAP : protocole, modèle de données et exploitation d’annuaire"
blatt: "ldap"
description: "LDAP pour les administrateurs : pile de protocoles et format filaire BER, DIT, Distinguished Names, schéma, Bind et SASL, recherche et contrôles, TLS, Active Directory, Global Catalog, limites de réplication, mise à l’échelle et diagnostic."
fakten:
  - label: Nom
    wert: Lightweight Directory Access Protocol
    href: https://datatracker.ietf.org/doc/html/rfc4510
  - label: Version du protocole
    wert: LDAPv3 · RFC 4510 à 4519
    href: https://datatracker.ietf.org/doc/html/rfc4510#section-1
  - label: Format filaire
    wert: Structures ASN.1, encodées en BER
    href: https://datatracker.ietf.org/doc/html/rfc4511#section-5.1
  - label: Transport
    wert: TCP ; TLS et SASL facultatifs par-dessus
    href: https://datatracker.ietf.org/doc/html/rfc4511#section-5
  - label: Ports
    wert: 389 LDAP · 636 LDAP sur TLS
    href: https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=ldap
  - label: Modèle de données
    wert: DIT composé d’entrées nommées et d’attributs
    href: https://datatracker.ietf.org/doc/html/rfc4512#section-2
  - label: Noms
    wert: DN composé de RDN ordonnés
    href: https://datatracker.ietf.org/doc/html/rfc4514
  - label: Filtres
    wert: Syntaxe préfixée selon RFC 4515
    href: https://datatracker.ietf.org/doc/html/rfc4515
  - label: Bind
    wert: anonymous, simple ou SASL
    href: https://datatracker.ietf.org/doc/html/rfc4513#section-5
  - label: StartTLS
    wert: Extended Operation sur une session existante
    href: https://datatracker.ietf.org/doc/html/rfc4511#section-4.14
  - label: Pagination
    wert: Simple Paged Results Control
    href: https://datatracker.ietf.org/doc/html/rfc2696
  - label: AD Global Catalog
    wert: 3268 LDAP · 3269 LDAP sur TLS
    href: https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/1010e59a-cf64-410c-a05c-a3a6c715261a
werbung:
  - newsletter
ctaThemen:
  - active-directory-entra
  - seppmail
translationSourceHash: 7b219213aa84d6ce78de262f2cd3a5ba23a220cc1669e134bceb3a325b06de33
translationModel: gpt-5.6-terra
translatedAt: 2026-10-08T11:21:12.156Z
translationReview: required
---

# LDAP : protocole, modèle de données et exploitation d’annuaire

LDAP est le langage commun permettant aux applications d’accéder aux services d’annuaire. Un client peut ainsi rechercher des entrées nommées, lire ou modifier des attributs et s’authentifier auprès de l’annuaire. Le protocole définit les messages, les opérations et les codes d’erreur. La manière dont un serveur stocke ses données, les réplique ou les protège contre les défaillances relève en revanche de chaque implémentation. Active Directory Domain Services, OpenLDAP et 389 Directory Server parlent donc LDAP sans être en interne la même plateforme ([RFC 4510, sections 1 et 2](https://datatracker.ietf.org/doc/html/rfc4510#section-1), [RFC 4511, section 3](https://datatracker.ietf.org/doc/html/rfc4511#section-3)).

Cette séparation est déterminante dans l’exploitation de la messagerie. Avant l’acceptation SMTP, une passerelle peut vérifier l’existence d’un destinataire, résoudre des groupes pour une politique ou authentifier un administrateur. Si cette requête échoue ou renvoie des données obsolètes, il ne s’agit pas simplement d’un « LDAP perturbé » : selon l’intégration, les messages sont refusés, les règles sont appliquées de manière erronée ou les connexions sont bloquées. L’administrateur doit donc savoir à quel endroit le chemin, du nom DNS à l’attribut lu, est interrompu.

L’explication suit ce chemin. Le client trouve d’abord un serveur et établit une session protégée. Il s’authentifie ensuite, effectue une recherche et interprète les réponses. Ce n’est qu’une fois ce déroulement normal compris que le schéma, les particularités d’Active Directory, la réplication, la mise à l’échelle et la restauration peuvent être correctement situés.

## Pile de protocoles et modèle de session

Avant qu’une application puisse effectuer une recherche, elle a besoin d’une cible de service concrète. Dans les environnements Active Directory, les enregistrements DNS SRV fournissent des contrôleurs de domaine ou Global Catalogs possibles ; d’autres produits utilisent des FQDN statiques, leur propre découverte de service ou un répartiteur de charge. Ce choix détermine non seulement l’adresse IP, mais aussi le site, le rôle du serveur et le nom auquel le certificat est vérifié. Un test de port vers un serveur quelconque accessible ne permet donc pas encore de savoir si l’application atteint sa cible prévue.

Sur la cible sélectionnée, LDAP établit une connexion [TCP](/kb/tcp). Le port 389 démarre en LDAP et peut passer à une session protégée avec l’Extended Operation StartTLS. Le port 636 est enregistré auprès de l’IANA sous le nom `ldaps` et est notamment utilisé par Active Directory pour TLS démarrant immédiatement. Dans les deux cas, le client doit vérifier la chaîne de certificats et le nom du serveur ; « chiffré » et « connecté au bon serveur » sont deux preuves distinctes ([RFC 4511, sections 4.14 et 5](https://datatracker.ietf.org/doc/html/rfc4511#section-4.14), [IANA Service Name Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=ldap), [MS-ADTS, Using SSL/TLS](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/8e73932f-70cf-46d6-88b1-8d9f86235e81)).

Au sein de cette connexion, LDAP ne transmet pas de lignes de commande lisibles comme [SMTP](/kb/smtp). Les messages sont décrits comme des structures ASN.1 et encodés avec les Basic Encoding Rules, BER. Chaque `LDAPMessage` porte un `messageID`, exactement une opération et éventuellement des contrôles. L’ID de message permet à une connexion persistante de distinguer plusieurs opérations en cours ; leurs réponses ne doivent pas nécessairement arriver dans l’ordre des requêtes. Une négociation TCP réussie ne dit donc rien du décodage BER, du Bind ou d’une recherche entièrement terminée ([RFC 4511, sections 3.1, 4.1.1 et 5.1](https://datatracker.ietf.org/doc/html/rfc4511#section-4.1.1)).

| Couche | Contenu normalisé | Observation pertinente pour l’administrateur |
|---|---|---|
| Application | Bind, Search, Compare, Modify, Add, Delete, ModifyDN, Extended Operations et Controls | Result Code, `diagnosticMessage`, Entries, References et Controls |
| Encodage | Types de données ASN.1 en BER | Erreurs de décodeur, taille maximale des requêtes, ID de message et OID |
| Sécurité | TLS ainsi que mécanismes SASL et leur Security Layer | Nom de certificat, chaîne de confiance, méthode de Bind, Signing, Channel Binding |
| Transport | connexion TCP persistante | Cible DNS, port, latence de connexion, réinitialisations, délai d’inactivité et état du pool |
| Interne au serveur | DIT, schéma, ACL, index, stockage et réplication | non normalisé par LDAP ; spécifique au produit et à la topologie |

Une limite importante s’applique aux modifications : une opération LDAP individuelle est atomique dans son périmètre, mais plusieurs entrées ne forment pas une transaction commune du protocole de base. RFC 5805 décrit une extension transactionnelle expérimentale, dont le client doit détecter la prise en charge au Root DSE. Même dans ce cas, il reste à vérifier comment les répliques voient la modification. Les processus de provisionnement ont donc besoin de leurs propres règles de répétition, d’erreurs partielles et de rapprochement, plutôt que d’une transaction de base de données tacitement supposée ([RFC 4511, section 3](https://datatracker.ietf.org/doc/html/rfc4511#section-3), [RFC 5805, sections 1 et 3](https://datatracker.ietf.org/doc/html/rfc5805#section-1)).

## Modèle de données : DIT, Entry, attribut et schéma

Après l’établissement de la session, le client doit pouvoir nommer où et quoi il recherche. LDAP organise pour cela les données d’annuaire sous forme de Directory Information Tree, ou DIT. Chaque Entry possède un Distinguished Name unique et des attributs. Le schéma décrit quels attributs existent, comment leurs valeurs sont comparées et quelles Object Classes ils exigent ou autorisent. Sans ce modèle, la base de recherche, les filtres et les résultats ne sont que des chaînes de caractères sans signification fiable ([RFC 4512, sections 2 et 3](https://datatracker.ietf.org/doc/html/rfc4512#section-2), [RFC 4511, section 4.1.7](https://datatracker.ietf.org/doc/html/rfc4511#section-4.1.7)).

Le Distinguished Name forme le chemin d’une entrée dans l’arborescence. Avec `cn=Mail Gateway,ou=Services,dc=example,dc=ch`, `cn=Mail Gateway` désigne le Relative Distinguished Name local ; les RDN suivants mènent via le conteneur jusqu’à la racine de nommage. Comme les RDN peuvent avoir plusieurs valeurs et que des caractères tels que la virgule, le plus ou la barre oblique inverse doivent être échappés, un logiciel ne doit pas traiter un DN en le découpant simplement sur les virgules. Il a besoin d’un analyseur conforme à RFC 4514 ([RFC 4512, section 2.3](https://datatracker.ietf.org/doc/html/rfc4512#section-2.3), [RFC 4514, sections 2 et 3](https://datatracker.ietf.org/doc/html/rfc4514#section-2)).

```text
dn: cn=Mail Gateway,ou=Services,dc=example,dc=ch
objectClass: top
objectClass: person
objectClass: organizationalPerson
cn: Mail Gateway
sn: Gateway
mail: mail-gateway@example.ch
```

Pour les exportations et importations, LDIF offre une représentation textuelle standardisée. LDIF représente des Entries ou des enregistrements de modification, mais n’est pas le format filaire d’une session LDAP en cours. Le repliement des lignes, les valeurs Base64 et les Change Records suivent leurs propres règles. Surtout, une exportation ne contient que ce que le serveur et les autorisations rendent visible ; des attributs opérationnels, ACL ou états de backend peuvent manquer. Un dump LDIF est donc un extrait de données, mais pas automatiquement une sauvegarde serveur restaurable ([RFC 2849, sections 2 et 4](https://datatracker.ietf.org/doc/html/rfc2849#section-2)).

La signification d’une valeur d’attribut provient uniquement du schéma. Une Matching Rule telle que `caseIgnoreMatch`, `integerMatch` ou une comparaison de DN détermine si deux valeurs sont égales et quels filtres fonctionnent sur elles. Sur le réseau, les valeurs apparaissent d’abord sous forme d’Octet Strings ; la syntaxe et le type d’attribut fournissent leur interprétation. Les éléments de schéma personnalisés ont donc besoin d’OID durablement uniques, de syntaxes et Matching Rules définies, ainsi que d’un déploiement tenant compte conjointement des serveurs et de tous les clients dépendants ([RFC 4512, section 4](https://datatracker.ietf.org/doc/html/rfc4512#section-4), [RFC 4517](https://datatracker.ietf.org/doc/html/rfc4517), [RFC 4520](https://datatracker.ietf.org/doc/html/rfc4520)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-ldap.svg?v=20260813" title="Interaktive Infografik: LDAP-Protokollstack, Nachrichtenschicht, DIT, Serverarchitektur und Admin-Diagnosepunkte" loading="lazy">
  <a href="/images/kb-interaktiv-ldap.svg?v=20260813">Ouvrir l’infographie sur le protocole LDAP et l’architecture d’annuaire</a>
</iframe>

## Opérations et changements d’état

Avec le transport, les noms et le schéma, l’ossature est en place ; le véritable dialogue de protocole commence alors. Le premier changement d’état décisif est généralement `Bind`. Il détermine sous quelle identité et avec quels droits en découlant les opérations suivantes sont exécutées. Un nouveau Bind remplace cet état. Tant que le serveur traite un Bind, le client ne peut pas démarrer d’autres opérations sur la même connexion ([RFC 4511, sections 3.1 et 4.2](https://datatracker.ietf.org/doc/html/rfc4511#section-4.2)).

Après un Bind réussi, le client peut lire ou écrire. Une recherche ne renvoie pas une seule grande réponse, mais zéro ou plusieurs messages `SearchResultEntry`, éventuellement des References et, à la fin, exactement un `SearchResultDone`. Seul ce résultat final indique si la séquence était complète. `Modify`, `Add`, `Delete` et `ModifyDN` modifient des entrées ; `Compare` vérifie une valeur d’attribut selon sa Matching Rule, sans renvoyer de résultat de recherche normal ([RFC 4511, sections 4.5 à 4.9](https://datatracker.ietf.org/doc/html/rfc4511#section-4.5)).

La fin de session possède également une sémantique claire. `Unbind` est une demande unilatérale de fermeture et ne possède pas de réponse. `Abandon` demande au serveur d’interrompre une opération donnée, sans toutefois garantir cette interruption. Si TCP est au contraire interrompu, toutes les opérations en cours disparaissent. Lors d’une opération d’écriture, le client ne peut alors pas savoir avec certitude si la modification a pris effet avant ou après la perte de connexion ; une nouvelle tentative nécessite donc d’abord un rapprochement d’état au lieu d’une répétition aveugle ([RFC 4511, sections 4.3 et 4.11](https://datatracker.ietf.org/doc/html/rfc4511#section-4.3)).

| Opération | Utilisation typique | Limite qu’un client doit traiter |
|---|---|---|
| Bind | compte de service, vérification d’utilisateur ou authentification SASL | une connexion TCP/TLS réussie n’est pas encore un Bind réussi |
| Search | destinataires, groupes, adresses, politiques et lecture du Root DSE | plusieurs Entries, References, limites, Controls et résultat final |
| Compare | vérifier côté serveur une valeur d’attribut connue | le résultat est `compareTrue` ou `compareFalse`, et non un résultat Search |
| Modify/Add/Delete/ModifyDN | provisionnement et cycle de vie | atomique par opération, mais sans transaction du protocole de base sur plusieurs Entries |
| Extended Operation | StartTLS, Password Modify ou fonctions spécifiques au fournisseur | vérifier l’OID et la prise en charge par le serveur cible |
| Controls | pagination, tri, assertion, synchronisation ou fonction du fournisseur | un Control critique inconnu doit conduire à une erreur |

Les Controls et Extended Operations complètent ce déroulement sans introduire une nouvelle version de LDAP. Chaque Control possède un OID, une Criticality et éventuellement une valeur encodée BER. Si un client marque comme critique un Control inconnu ou inexécutable, l’opération doit échouer avec `unavailableCriticalExtension` ; sinon, le serveur peut l’ignorer. Avant la pagination, la synchronisation ou une fonction du fournisseur, un client propre lit donc notamment au Root DSE `supportedControl`, `supportedExtension`, `supportedFeatures`, `supportedLDAPVersion` et `supportedSASLMechanisms` ([RFC 4511, section 4.1.11](https://datatracker.ietf.org/doc/html/rfc4511#section-4.1.11), [RFC 4512, section 5.1](https://datatracker.ietf.org/doc/html/rfc4512#section-5.1)).

## Bind, SASL et limites de confiance TLS

Le Bind détermine à qui le serveur attribue la recherche suivante. Avec Simple Bind, trois cas doivent être distingués : un DN vide et un mot de passe vide donnent un accès anonyme. Un DN non vide avec un mot de passe vide est un *unauthenticated Bind* et ne confirme expressément pas l’identité indiquée, même si le serveur peut renvoyer `success`. Seul un DN non vide avec un mot de passe non vide constitue l’authentification normale par nom et mot de passe. Les clients devraient donc refuser les mots de passe vides avant même la requête ; les serveurs ne devraient pas autoriser par inadvertance les unauthenticated Binds ([RFC 4513, sections 5.1.1 à 5.1.3 et 6.3.1](https://datatracker.ietf.org/doc/html/rfc4513#section-5.1)).

Avec cette authentification par mot de passe, le serveur connaît le secret présenté. Le transport doit donc non seulement être chiffré, mais aussi authentifié. Cela comprend une chaîne de certificats valide et la vérification que le nom DNS configuré figure dans le certificat. Quiconque accepte chaque certificat ou utilise une adresse IP peut établir un canal chiffré avec le mauvais interlocuteur. La même vérification s’applique à StartTLS et à LDAP avec TLS immédiat ([RFC 4513, sections 3.1 et 5.1.3](https://datatracker.ietf.org/doc/html/rfc4513#section-3.1), [RFC 9525, sections 2 et 4](https://datatracker.ietf.org/doc/html/rfc9525#section-4), [TLS](/kb/tls)).

SASL permet d’utiliser différents mécanismes d’authentification au lieu d’un simple Bind par mot de passe et peut en outre négocier une protection des messages LDAP suivants. Dans Active Directory, on rencontre notamment Negotiate, Kerberos et NTLM. LDAP Signing y protège l’intégrité de certaines sessions SASL ; Channel Binding lie l’authentification à la connexion TLS sous-jacente. TLS, Signing et Channel Binding résolvent ainsi des problèmes apparentés, mais non identiques. Un test doit reproduire le type de Bind réellement utilisé par le produit ([RFC 4513, section 5.2](https://datatracker.ietf.org/doc/html/rfc4513#section-5.2), [Microsoft: LDAP signing](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing), [MS-ADTS, Channel Binding](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/2c3ea153-eff9-46a7-8614-19b677efa4e0)).

Pour introduire des politiques AD strictes, il ne suffit donc pas de considérer la version de Windows. Les nouveaux déploiements AD DS sous Windows Server 2025 exigent LDAP Signing par défaut, tandis que les mises à niveau reprennent les paramètres existants. Microsoft cite les événements Directory Service 2886 à 2889 pour Signing ainsi que 3039 à 3041 pour Channel Binding. Ces données d’audit montrent quels clients, ports et méthodes de Bind seraient réellement concernés ; ce n’est qu’ensuite que l’application peut être planifiée sur une base de données fiable ([Microsoft: LDAP signing, Default Security Behavior et Event Monitoring](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing), [Microsoft: LDAP session security after ADV190023](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/ldap-session-security-settings-requirements-adv190023)).

## Search : base, scope, filtre et projection d’attributs

Après un Bind sécurisé vient l’opération dont dépendent la plupart des intégrations : Search. La requête désigne un Base DN, le Scope, le traitement des alias, ses propres limites de taille et de temps, un filtre et les attributs demandés. `baseObject` lit uniquement l’entrée de base, `singleLevel` ses enfants directs et `wholeSubtree` l’ensemble du sous-arbre, base comprise. Le serveur peut imposer des limites plus strictes. Aucun résultat avec `success` est une réponse valide ; `noSuchObject` signifie au contraire que la base de recherche est absente ou non visible pour cette identité ([RFC 4511, sections 4.5.1 et 4.5.2](https://datatracker.ietf.org/doc/html/rfc4511#section-4.5.1)).

Le filtre ne décrit pas une logique SQL libre, mais une arborescence en notation préfixée. `(&(objectClass=person)(mail=*@example.ch))` combine par exemple une expression Equality et une expression Substring. `|` signifie OR, `!` NOT, `=*` Presence et `:=` un Extensible Match. La Matching Rule de l’attribut concerné détermine si une comparaison utilise la casse, l’ordre numérique ou la sémantique DN ([RFC 4511, section 4.5.1](https://datatracker.ietf.org/doc/html/rfc4511#section-4.5.1), [RFC 4515](https://datatracker.ietf.org/doc/html/rfc4515), [RFC 4517](https://datatracker.ietf.org/doc/html/rfc4517)).

La création de filtres devient ainsi une tâche de sécurité. Les valeurs issues d’entrées utilisateur doivent être encodées selon RFC 4515 ; en particulier, `*`, les parenthèses, la barre oblique inverse, NUL et les octets UTF-8 non valides ne doivent pas parvenir bruts dans l’expression. Une concaténation de chaînes peut sinon modifier la structure du filtre et permettre une injection LDAP. L’échappement de DN selon RFC 4514 suit d’autres règles et ne remplace pas l’encodage de filtre ([RFC 4515, section 3](https://datatracker.ietf.org/doc/html/rfc4515#section-3), [RFC 4514, section 3](https://datatracker.ietf.org/doc/html/rfc4514#section-3)).

Outre le filtre, la liste d’attributs détermine la quantité de données renvoyées par le serveur. Une liste vide demande tous les attributs utilisateur normaux, `1.1` ne demande aucun attribut, `*` tous les attributs utilisateur et `+` demande, selon RFC 3673, tous les attributs opérationnels. Les ACL peuvent encore masquer certaines valeurs. Les clients de production ne devraient demander que les attributs nécessaires : de grandes valeurs multivaluées sollicitent le réseau, le décodeur et la mémoire, et peuvent déclencher leurs propres limites serveur ([RFC 4511, section 4.5.1.8](https://datatracker.ietf.org/doc/html/rfc4511#section-4.5.1.8), [RFC 3673](https://datatracker.ietf.org/doc/html/rfc3673)).

### Pagination, tri et résultats changeants

Les ensembles de résultats plus importants sont généralement transmis avec le Simple Paged Results Control. Le serveur joint à chaque page un cookie opaque que le client renvoie avec la même requête. Ce cookie n’est ni un offset ni un curseur durable. Si le contenu de l’annuaire change pendant la séquence, des entrées peuvent manquer ou apparaître deux fois. La pagination limite donc la quantité de données par réponse, mais ne produit pas de snapshot cohérent ([RFC 2696, sections 2 et 3](https://datatracker.ietf.org/doc/html/rfc2696#section-2)).

Active Directory rend cette distinction visible au quotidien : la LDAP Policy `MaxPageSize` limite par défaut les résultats non paginés à 1000 objets. Une importation qui reçoit exactement 1000 entrées n’a donc pas démontré son exhaustivité. Le client doit traiter correctement les pages et les cookies et détecter une interruption. D’autres politiques limitent la durée des requêtes, le Receive Buffer et les Result Sets simultanément conservés. Pour l’exploitation, il faut donc consigner la taille de page, le nombre de pages, la progression du dernier cookie, le délai d’expiration et la reprise ([MS-ADTS, LDAP Policies](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/3f0137a1-63df-400c-bf97-e1040f055a99), [Microsoft: Paging Search Results](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ldap/paging-search-results)).

## Active Directory comme profil de serveur LDAP

Les règles précédentes s’appliquent à LDAP en général. Active Directory Domain Services est une implémentation de serveur concrète avec des rôles et conventions supplémentaires. Ses données sont réparties sur des Naming Contexts ; un contrôleur de domaine détient au minimum le schéma, la configuration et le Domain Naming Context de son propre domaine. Le Root DSE possède le DN vide et indique notamment `defaultNamingContext`, `configurationNamingContext`, `schemaNamingContext`, tous les `namingContexts`, le nom du serveur et les mécanismes pris en charge. Après TCP et TLS, cette entrée est le premier test qui renseigne réellement sur le service d’annuaire atteint ([RFC 4512, section 5.1](https://datatracker.ietf.org/doc/html/rfc4512#section-5.1), [Microsoft RootDSE](https://learn.microsoft.com/en-us/windows/win32/adschema/rootdse), [MS-ADTS, rootDSE Attributes](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/96f7b086-1ca3-4764-9a08-33f8f7a543db)).

Le choix entre un contrôleur de domaine et un Global Catalog modifie le résultat de recherche. Un DC sert LDAP sur 389 ou 636 et connaît le Domain Naming Context complet de son domaine. Le Global Catalog utilise en plus 3268 ou 3269 et conserve une réplique partielle de tous les domaines de la forêt. Il peut trouver des objets à l’échelle de la forêt, mais ne fournit pour les domaines distants que les attributs du Partial Attribute Set. Un résultat positif ne prouve donc pas encore que l’attribut requis par l’application soit disponible ([MS-ADTS, Ports](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/1010e59a-cf64-410c-a05c-a3a6c715261a), [Microsoft: Searching the Global Catalog](https://learn.microsoft.com/en-us/windows/win32/ad/searching-global-catalog-contents), [Microsoft: Attributes included in the Global Catalog](https://learn.microsoft.com/en-us/windows/win32/ad/attributes-included-in-the-global-catalog)).

Les filtres peuvent eux aussi devenir spécifiques à AD. La Matching Rule `1.2.840.113556.1.4.1941`, `LDAP_MATCHING_RULE_TRANSITIVE_EVAL`, suit par exemple les attributs liés et peut évaluer des groupes imbriqués. Sa prise en charge n’apparaît pas simplement dans `supportedControl`. En outre, `memberOf` ne contient pas le Primary Group. Une décision d’autorisation basée sur les appartenances aux groupes doit donc explicitement tenir compte de l’imbrication des groupes, du Primary Group, du périmètre des groupes, de la visibilité ACL et de l’état de réplication ([MS-ADTS, LDAP Matching Rules](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/4e638665-f466-4597-93c4-12f2ebfabab5), [Microsoft: Primary group membership](https://learn.microsoft.com/en-us/troubleshoot/entra/entra-id/user-prov-sync/exclude-user-primary-group)).

Pour les clients Windows, le DC Locator décide, conjointement avec les enregistrements [DNS](/kb/dns)-SRV, quel serveur répond à ces requêtes. Les enregistrements liés au site et au rôle fournissent des candidats avec priorité et poids. Une adresse IP saisie statiquement contourne cette sélection et complique la vérification du certificat. Un simple répartiteur de charge TCP distribue certes les connexions, mais sans logique supplémentaire, il ne connaît ni les DC inscriptibles ni les Global Catalogs, Naming Contexts ou l’état de la réplication ([Microsoft: DC Locator](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/dc-locator), [Microsoft: Verify LDAP SRV records](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/verify-srv-dns-records-have-been-created)).

Enfin, l’accessibilité LDAP ne doit pas être confondue avec une réplication saine. AD réplique les modifications d’annuaire via le Directory Replication Service Remote Protocol ; le `syncrepl` d’OpenLDAP utilise au contraire LDAP Content Synchronization avec Provider, Consumer et cookies. Une recherche de test peut montrer qu’un serveur donné répond. Pour savoir si tous les serveurs possèdent les mêmes modifications et rattrapent leur retard après une panne, il faut utiliser les outils de la plateforme concernée ([MS-DRSR, Relationship to Other Protocols](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-drsr/0597aff6-0177-4d52-99f2-14a5441bc3c1), [RFC 4533](https://datatracker.ietf.org/doc/html/rfc4533), [OpenLDAP Administrator's Guide: Replication](https://www.openldap.org/doc/admin25/replication.html)).

## Modèles d’intégration et d’exploitation

Pour l’exploitation, le fait qu’un produit « prend en charge LDAP » importe désormais moins que la manière dont il utilise LDAP. Avec une **recherche à l’exécution**, un message ou une session attend directement Search et la réponse du serveur. Avec le **Credential Check**, un compte technique recherche d’abord le DN de l’utilisateur, puis effectue un second Bind avec le mot de passe saisi. Un **import ou cache**, en revanche, lit de nombreuses entrées et travaille avec une copie locale jusqu’à l’exécution suivante. Ces modèles ont des conséquences différentes pour la latence, le traitement des mots de passe, le basculement et l’ancienneté des données ; la documentation du produit doit préciser le comportement concret ([RFC 4511, sections 4.2 et 4.5](https://datatracker.ietf.org/doc/html/rfc4511#section-4.2), [RFC 2696](https://datatracker.ietf.org/doc/html/rfc2696)).

| Modèle | Chemin critique | Preuve opérationnelle immédiate |
|---|---|---|
| recherche à l’exécution | DNS, connexion, TLS, pool, Bind, Search et réponse du serveur par opération | p50/p95/p99 par opération, saturation du pool, Result Codes, cible de secours |
| Credential Check | recherche d’utilisateur plus second Bind avec mot de passe utilisateur | résolution DN, blocage des mots de passe vides, vérification du nom TLS, comportement de verrouillage |
| import périodique | énumération complète et paginée, puis validation dans le cache local | progression page/cookie, nombre d’objets, modèle de suppression, dernier commit réussi |
| Change Sync | curseur de synchronisation spécifique au fournisseur ou LDAP | persistance du curseur, replay, resynchronisation et objets supprimés |

Quel que soit le modèle, le client nécessite des délais d’expiration distincts pour la connexion, le Bind, l’opération et l’inactivité. Un Connection Pool économise l’établissement TCP, TLS et Bind, mais transporte l’état d’authentification de la connexion. Les sessions mortes doivent être détectées, et une connexion ne doit pas changer par inadvertance d’utilisateur ou de locataire. Le basculement nécessite un ordre de cibles compréhensible, des répétitions limitées et un chemin de retour vers la cible privilégiée. Sinon, des tentatives parallèles multiplient la charge précisément pendant une panne de l’annuaire ([RFC 4511, sections 3.1, 4.2 et 5.3](https://datatracker.ietf.org/doc/html/rfc4511#section-3.1)).

Côté serveur, Base DN, Scope, filtre et liste d’attributs déterminent le travail. Une condition d’égalité sélective sur un attribut indexé est différente d’un Substring initial ou d’une grande expression OR. LDAP ne publie pas de plan d’exécution et ne prescrit aucune technique d’indexation. L’administrateur doit donc corréler les filtres réels du produit avec le volume de résultats, la latence p95/p99 et les métriques serveur. Une recherche rapide d’un seul compte de test ne prouve pas qu’une vérification de destinataires évolue à la charge de pointe ([OpenLDAP Administrator's Guide: Performance Tuning](https://www.openldap.org/doc/admin26/tuning.html), [MS-ADTS: LDAP Policies](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/3f0137a1-63df-400c-bf97-e1040f055a99)).

La supervision devrait décomposer le déroulement selon les mêmes étapes que le diagnostic : sélection DNS, établissement TCP et TLS, Bind, latence Search, Result Code, nombre de résultats et progression de pagination. S’y ajoutent l’occupation du pool, le taux de répétition et l’état de l’import ou de la synchronisation. Un seul Bind synthétique peut confirmer l’accessibilité, mais ne détecte ni les attributs manquants, ni une importation incomplète, ni un partenaire de réplication en retard.

Pour la sauvegarde et la restauration, le contenu d’annuaire visible ne suffit pas non plus. Il faut sauvegarder le schéma, les ACL, la configuration du backend et du serveur, les clés et certificats, les identités de réplication ainsi que la procédure de réintégration d’un nœud restauré dans la topologie. Active Directory utilise à cette fin System State et ses propres étapes de Forest Recovery ; OpenLDAP dépend de son backend. Le manuel distingue par exemple une sauvegarde LMDB de `slapcat` et signale des états LDIF sémantiquement incohérents lors de modifications en plusieurs parties. LDAP lui-même ne définit aucun mécanisme de sauvegarde ([Microsoft: Back up the System State data](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-backing-up-system-state), [OpenLDAP Administrator's Guide: Directory Backups](https://www.openldap.org/doc/admin26/maintenance.html)).

Un test de restauration n’est terminé que lorsqu’un client trouve le service restauré via le nom DNS prévu, que TLS et Bind réussissent, que le Root DSE et le schéma correspondent, que des recherches réelles renvoient des attributs complets et que la réplication redémarre de manière contrôlée. La restauration ramène ainsi au début de l’article : c’est l’ensemble du chemin qui compte, pas seulement une base de données démarrée.

## Outils de diagnostic

Le diagnostic suit le même chemin qu’une requête de production. Il commence sur le réseau de l’application concernée et utilise son nom DNS, son magasin de confiance, sa méthode de Bind, son Base DN, son filtre et sa liste d’attributs. Un test depuis l’ordinateur portable de l’administrateur peut sinon réussir alors que la passerelle utilise toujours un autre DC, une autre CA ou un autre Scope. Les exemples utilisent des noms réservés et ne lisent que des métadonnées ; les mots de passe de Bind n’ont leur place ni dans l’historique du shell ni dans les arguments de processus. `ldapsearch -W` les demande de manière interactive.

### Déterminer les cibles de service via DNS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für LDAP-Dienstsuche">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName _ldap._tcp.dc._msdcs.example.ch -Type SRV -DnsOnly
Resolve-DnsName _ldap._tcp.gc._msdcs.example.ch -Type SRV -DnsOnly</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig +noall +answer SRV _ldap._tcp.dc._msdcs.example.ch
dig +noall +answer SRV _ldap._tcp.gc._msdcs.example.ch</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) et [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) indiquent les cibles, ports, priorités et poids. Il faut ensuite vérifier la résolution A/AAAA, l’affinité au site et l’accessibilité de chaque cible réellement sélectionnable. Un seul DC accessible ne corrige pas un jeu SRV erroné ([Microsoft: DC Locator](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/dc-locator)).

### Vérifier TCP et TLS implicite sur le port 636

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für LDAP-TLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection dc1.example.ch -Port 636 -InformationLevel Detailed

$tcp = [Net.Sockets.TcpClient]::new("dc1.example.ch", 636)
$tls = [Net.Security.SslStream]::new($tcp.GetStream(), $false)
$tls.AuthenticateAsClient("dc1.example.ch")
$tls.SslProtocol
$tls.RemoteCertificate.Subject
$tls.RemoteCertificate.GetExpirationDateString()
$tls.Dispose(); $tcp.Dispose()</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">openssl s_client \
  -connect dc1.example.ch:636 \
  -servername dc1.example.ch \
  -verify_hostname dc1.example.ch \
  -verify_return_error -brief</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) établit d’abord uniquement la connexion TCP. Le [.NET `SslStream`](https://learn.microsoft.com/dotnet/api/system.net.security.sslstream) ou [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) qui suit vérifie TLS avec le nom DNS configuré. `s_client -showcerts` affiche seulement les certificats envoyés par le serveur et ne constitue pas à lui seul une preuve réussie de chaîne ou de nom d’hôte. StartTLS sur 389 peut être vérifié séparément sous Unix avec `openssl s_client -starttls ldap` ([OpenSSL `s_client`](https://docs.openssl.org/master/man1/openssl-s_client/), [RFC 4511, section 4.14](https://datatracker.ietf.org/doc/html/rfc4511#section-4.14)).

### Lire le Root DSE et les capacités

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für LDAP-Root-DSE-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-ADRootDSE -Server dc1.example.ch -Properties @(
  "defaultNamingContext"
  "namingContexts"
  "supportedLDAPVersion"
  "supportedControl"
  "supportedSASLMechanisms"
)</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ldapsearch -LLL -x -ZZ -H ldap://dc1.example.ch \
  -s base -b "" \
  defaultNamingContext namingContexts supportedLDAPVersion \
  supportedControl supportedSASLMechanisms</code></pre>
  </div>
</div>

[`Get-ADRootDSE`](https://learn.microsoft.com/powershell/module/activedirectory/get-adrootdse?view=windowsserver2025-ps) utilise ici le module ActiveDirectory et, par défaut, l’identité Windows connectée. [`ldapsearch`](https://www.openldap.org/software/man.cgi?query=ldapsearch&sektion=1&apropos=0&manpath=OpenLDAP+2.6-Release) impose avec `-ZZ` un StartTLS réussi et lit anonymement uniquement les attributs Root DSE publiés par le serveur. Un OID absent prouve que cette cible précise ne publie pas la fonction ; cela ne dit rien des autres nœuds du cluster.

### Reproduire une recherche réelle avec Scope, filtre et pagination

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für LDAP-Suchprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$params = @{
  Server         = "dc1.example.ch"
  SearchBase     = "OU=People,DC=example,DC=ch"
  SearchScope    = "Subtree"
  LDAPFilter     = "(&(objectClass=user)(mail=admin@example.ch))"
  Properties     = @("mail", "proxyAddresses", "memberOf")
  ResultPageSize = 500
}

Get-ADUser @params</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ldapsearch -LLL -x -ZZ -H ldap://dc1.example.ch \
  -D "CN=svc-lookup,OU=Services,DC=example,DC=ch" -W \
  -b "OU=People,DC=example,DC=ch" -s sub \
  -E pr=500/noprompt \
  "(&(objectClass=user)(mail=admin@example.ch))" \
  mail proxyAddresses memberOf</code></pre>
  </div>
</div>

[`Get-ADUser`](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser?view=windowsserver2025-ps) accepte avec `-LDAPFilter` la syntaxe de filtre proche des RFC et effectue la pagination via `-ResultPageSize`. [`ldapsearch`](https://www.openldap.org/software/man.cgi?query=ldapsearch&sektion=1&apropos=0&manpath=OpenLDAP+2.6-Release) utilise `-E pr=500/noprompt` pour le Paged Results Control et `-W` pour une demande de mot de passe interactive. En plus du résultat, le test doit documenter le résultat final, le nombre de pages, les attributs renvoyés et la durée d’exécution.

### Associer les erreurs à une limite

| Observation | Signification protocolaire | Preuve fiable suivante |
|---|---|---|
| Timeout avant TLS | résolution de cible, routage, pare-feu, écouteur ou pool épuisé | SRV/A/AAAA, négociation TCP, écouteur serveur et latence de connexion |
| erreur de certificat | chaîne, validité, nom ou confiance du client incorrects | chaîne envoyée, Trust Anchor, SAN par rapport au FQDN configuré exact |
| `strongAuthRequired` / `confidentialityRequired` | le serveur exige une méthode de Bind ou de protection plus forte | port, succès StartTLS, mécanisme SASL, politique Signing/CBT |
| `invalidCredentials` | identité Bind présentée ou identifiants refusés | type et DN de Bind exacts ; aucune journalisation de mot de passe |
| `invalidDNSyntax` | DN syntaxiquement invalide | encodage RFC 4514 et DN effectif issu du Search Result |
| `noSuchObject` avec `matchedDN` | Base DN absent ou invisible à partir d’un ancêtre | Root DSE, Naming Context, visibilité ACL et `matchedDN` |
| `sizeLimitExceeded` | limite client ou serveur avant le résultat complet | Paging Control, cookies de page, LDAP Policy et comptage total |
| `adminLimitExceeded` / `busy` / `unavailable` | ressource serveur ou limite administrative | métriques serveur, Query Policy, coût du filtre, taux de répétition et nœud cible |
| aucun résultat avec `success` | recherche valide sans correspondance visible | comparer base, Scope, filtre, ACL, nœud cible et état de réplication |

Les Result Codes numériques font partie du protocole LDAP ; `diagnosticMessage` et les sous-codes AD supplémentaires sont en revanche un contexte spécifique à l’implémentation. L’automatisation devrait donc d’abord évaluer le Result Code et consigner le texte en complément. Pour `busy` et `unavailable`, chaque client a besoin d’un budget de répétition limité avec backoff. Des répétitions illimitées transforment un problème d’annuaire isolé en pic de charge dans tous les systèmes dépendants ([RFC 4511, section 4.1.9 et annexe A](https://datatracker.ietf.org/doc/html/rfc4511#appendix-A)).

## Histoire technique

LDAP n’est pas né comme une base de données d’annuaire indépendante. À la fin des années 1980, X.500 avait défini un modèle d’annuaire complet et le Directory Access Protocol. En 1993, RFC 1487 décrivait un accès plus léger à ce modèle ; RFC 1777 suivit en 1995 sous le nom LDAP Version 2. Le terme « Lightweight » se rapportait à l’accès au protocole simplifié par rapport à DAP, non à de petits annuaires ou à une faible importance opérationnelle. Le développement initial est étroitement associé à Tim Howes et à l’University of Michigan ([RFC 1487](https://datatracker.ietf.org/doc/html/rfc1487), [RFC 1777](https://datatracker.ietf.org/doc/html/rfc1777)).

LDAPv3 a été publié en 1997 avec RFC 2251 et des documents associés. Les opérations extensibles, Controls, SASL, l’internationalisation et le modèle de données révisé en ont fait la base des implémentations actuelles. Le travail LDAPbis a réorganisé cet état en 2006 : RFC 4510 sert de feuille de route, RFC 4511 décrit le protocole, RFC 4512 le modèle d’information et RFC 4513 la sécurité ; RFC 4514 à 4519 complètent les représentations, URL, syntaxes et schéma ([RFC 2251](https://datatracker.ietf.org/doc/html/rfc2251), [RFC 4510, section 3](https://datatracker.ietf.org/doc/html/rfc4510#section-3)).

Parallèlement, des serveurs très différents ont évolué. OpenLDAP est issu en 1998 de l’implémentation de l’University of Michigan et a poursuivi `slapd`, les bibliothèques et les outils comme projet open source. Avec Windows 2000, Active Directory a largement introduit un profil LDAPv3 avec son propre schéma, Naming Contexts, Controls, Matching Rules et protocole de réplication distinct ([OpenLDAP Release Road Map](https://www.openldap.org/software/roadmap.html), [OpenLDAP Administrator's Guide – Preface](https://www.openldap.org/doc/admin26/preface.html), [MS-ADTS](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/)).

Cette histoire explique la règle d’exploitation la plus importante : LDAP unifie l’accès, non l’architecture interne. Quiconque déplace un client d’OpenLDAP vers AD DS ou entre deux appliances doit donc vérifier davantage que l’hôte, le port et le Bind DN. Le schéma, les Controls, les limites, la résolution des groupes, la réplication et la restauration restent des propriétés du produit.

## Sources

- [RFC 4510, sections 1 et 2](https://datatracker.ietf.org/doc/html/rfc4510)
- [RFC 4511 – LDAP: The Protocol](https://datatracker.ietf.org/doc/html/rfc4511) – couche de messages, opérations, BER, TCP, StartTLS et Result Codes.
- [IANA – Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=ldap) – `ldap` 389 et `ldaps` 636.
- [MS-ADTS – Using SSL/TLS](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/8e73932f-70cf-46d6-88b1-8d9f86235e81) – TLS implicite et StartTLS dans Active Directory.
- [RFC 5805, sections 1 et 3](https://datatracker.ietf.org/doc/html/rfc5805)
- [RFC 4512 – LDAP Directory Information Models](https://datatracker.ietf.org/doc/html/rfc4512) – DIT, Entries, attributs, schéma, Root DSE et Subschema.
- [RFC 4514, sections 2 et 3](https://datatracker.ietf.org/doc/html/rfc4514)
- [RFC 2849, sections 2 et 4](https://datatracker.ietf.org/doc/html/rfc2849)
- [RFC 4517 – LDAP Syntaxes and Matching Rules](https://datatracker.ietf.org/doc/html/rfc4517) – syntaxes standard et règles de comparaison.
- [RFC 4520 – IANA Considerations for LDAP](https://datatracker.ietf.org/doc/html/rfc4520) – enregistrement des OID et paramètres de protocole.
- [RFC 4513 – LDAP Authentication Methods and Security Mechanisms](https://datatracker.ietf.org/doc/html/rfc4513) – méthodes de Bind, SASL, TLS et limites de sécurité.
- [RFC 9525, sections 2 et 4](https://datatracker.ietf.org/doc/html/rfc9525)
- [Microsoft – LDAP signing for AD DS](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing) – Signing, Channel Binding, paramètres par défaut et événements.
- [MS-ADTS – Channel Binding](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/2c3ea153-eff9-46a7-8614-19b677efa4e0) – LDAP Channel Binding dans Active Directory.
- [Microsoft – LDAP session security after ADV190023](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/ldap-session-security-settings-requirements-adv190023) – exigences de sécurité selon le modèle de Bind et TLS.
- [RFC 4515 – String Representation of Search Filters](https://datatracker.ietf.org/doc/html/rfc4515) – grammaire des filtres et encodage des valeurs.
- [RFC 3673 – All Operational Attributes](https://datatracker.ietf.org/doc/html/rfc3673) – `+` comme sélecteur d’attributs pour les attributs opérationnels.
- [RFC 2696 – Simple Paged Results Control](https://datatracker.ietf.org/doc/html/rfc2696) – pages, cookies et limites de cohérence.
- [MS-ADTS – LDAP Policies](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/3f0137a1-63df-400c-bf97-e1040f055a99) – limites administratives de recherche et de ressources.
- [Microsoft – Paging Search Results](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ldap/paging-search-results) – recherche paginée dans Active Directory.
- [Microsoft – RootDSE](https://learn.microsoft.com/en-us/windows/win32/adschema/rootdse) – Naming Contexts et capacités du serveur.
- [MS-ADTS – rootDSE Attributes](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/96f7b086-1ca3-4764-9a08-33f8f7a543db) – attributs Root DSE spécifiques à AD.
- [MS-ADTS – Ports](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/1010e59a-cf64-410c-a05c-a3a6c715261a) – ports LDAP, LDAPS et Global Catalog.
- [Microsoft – Searching the Global Catalog](https://learn.microsoft.com/en-us/windows/win32/ad/searching-global-catalog-contents) – recherche à l’échelle de la forêt et réplique partielle.
- [Microsoft – Attributes included in the Global Catalog](https://learn.microsoft.com/en-us/windows/win32/ad/attributes-included-in-the-global-catalog) – Partial Attribute Set du Global Catalog.
- [MS-ADTS – LDAP Matching Rules](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/4e638665-f466-4597-93c4-12f2ebfabab5) – OID Extensible Match spécifiques à AD.
- [Microsoft – Primary group membership](https://learn.microsoft.com/en-us/troubleshoot/entra/entra-id/user-prov-sync/exclude-user-primary-group) – distinction entre `memberOf` et Primary Group.
- [Microsoft – DC Locator](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/dc-locator) – sélection DNS SRV, LDAP Ping et affinité au site.
- [Microsoft – Verify LDAP SRV records](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/verify-srv-dns-records-have-been-created) – enregistrement SRV des contrôleurs de domaine.
- [MS-DRSR – Relationship to Other Protocols](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-drsr/0597aff6-0177-4d52-99f2-14a5441bc3c1) – distinction entre la réplication AD et LDAP.
- [RFC 4533 – LDAP Content Synchronization Operation](https://datatracker.ietf.org/doc/html/rfc4533) – LDAP Sync Controls, cookies et modèle d’état.
- [OpenLDAP Administrator's Guide – Replication](https://www.openldap.org/doc/admin25/replication.html) – `syncrepl`, cookies et modèle Provider/Consumer.
- [OpenLDAP Administrator's Guide – Performance Tuning](https://www.openldap.org/doc/admin26/tuning.html) – indexation et coûts de recherche internes au serveur.
- [Microsoft – Back up the System State data](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-backing-up-system-state) – sauvegarde System State pour AD DS.
- [OpenLDAP Administrator's Guide – Directory Backups](https://www.openldap.org/doc/admin26/maintenance.html) – sauvegarde LMDB, `slapcat` et limites de cohérence.
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – requêtes DNS et SRV sous Windows.
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html) – requêtes DNS et SRV sous systèmes Unix.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – diagnostic de connexion TCP sous Windows.
- [Microsoft Learn – SslStream](https://learn.microsoft.com/dotnet/api/system.net.security.sslstream) – négociation TLS et vérification des certificats avec .NET.
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/) – négociation TLS, vérification de nom et LDAP StartTLS.
- [Microsoft Learn – Get-ADRootDSE](https://learn.microsoft.com/powershell/module/activedirectory/get-adrootdse?view=windowsserver2025-ps) – diagnostic Root DSE sous Windows.
- [OpenLDAP – ldapsearch(1)](https://www.openldap.org/software/man.cgi?query=ldapsearch&sektion=1&apropos=0&manpath=OpenLDAP+2.6-Release) – options Search, StartTLS, SASL et Control.
- [Microsoft Learn – Get-ADUser](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser?view=windowsserver2025-ps) – LDAPFilter, SearchBase, Scope et pagination.
- [RFC 1487 – X.500 Lightweight Directory Access Protocol](https://datatracker.ietf.org/doc/html/rfc1487) – première spécification LDAP de 1993.
- [RFC 1777 – Lightweight Directory Access Protocol](https://datatracker.ietf.org/doc/html/rfc1777) – LDAPv2 et modèle de protocole historique.
- [RFC 2251 – Lightweight Directory Access Protocol v3](https://datatracker.ietf.org/doc/html/rfc2251) – première spécification centrale LDAPv3.
- [OpenLDAP – Release Road Map](https://www.openldap.org/software/roadmap.html) – version 1.0 en août 1998.
- [OpenLDAP Administrator's Guide – Preface](https://www.openldap.org/doc/admin26/preface.html) – LDAP de l’University of Michigan comme fondement du projet.
- [MS-ADTS – Active Directory Technical Specification](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/) – profil de serveur LDAP d’AD DS et AD LDS.
