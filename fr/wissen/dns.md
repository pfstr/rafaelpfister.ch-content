---
title: "DNS : résolution, délégation et exploitation"
blatt: "dns"
description: "Le Domain Name System vu par un administrateur : espace de noms, zones et délégation, résolution récursive, RRsets, cache et TTL, réponses négatives, UDP et TCP, EDNS, DNSSEC, réplication de zones ainsi que diagnostic des infrastructures de messagerie."
fakten:
  - label: Nom complet
    wert: Domain Name System
    href: https://datatracker.ietf.org/doc/html/rfc1034
  - label: Modèle de base
    wert: espace de noms hiérarchique distribué et ensemble de données
    href: https://datatracker.ietf.org/doc/html/rfc9499#section-2
  - label: Normes fondamentales
    wert: RFC 1034 · RFC 1035 · STD 13
    href: https://www.rfc-editor.org/info/std13
  - label: Clé de requête
    wert: QNAME · QTYPE · QCLASS
    href: https://datatracker.ietf.org/doc/html/rfc1035#section-4.1.2
  - label: Unité de données
    wert: Resource Record Set (RRset)
    href: https://datatracker.ietf.org/doc/html/rfc9499#section-5
  - label: Rôles des serveurs
    wert: autoritatif · récursif · redirecteur
    href: https://datatracker.ietf.org/doc/html/rfc9499#section-6
  - label: Transport
    wert: UDP et TCP · Port 53
    href: https://datatracker.ietf.org/doc/html/rfc7766
  - label: Extensions
    wert: EDNS(0) via OPT
    href: https://datatracker.ietf.org/doc/html/rfc6891
  - label: Cohérence
    wert: caches positifs et négatifs gérés par TTL
    href: https://datatracker.ietf.org/doc/html/rfc2308
  - label: Intégrité
    wert: "DNSSEC: DNSKEY · DS · RRSIG · NSEC"
    href: https://datatracker.ietf.org/doc/html/rfc4034
  - label: Synchronisation des zones
    wert: NOTIFY · AXFR · IXFR
    href: https://datatracker.ietf.org/doc/html/rfc1996
  - label: Lien avec la messagerie
    wert: MX · PTR · TXT et noms de destination dérivés
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-5
werbung:
  - tools
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: f68d5b35d0c022538eb216baafcdf1c277fffbe2c2db0ed4a3b519c01ba63062
translationModel: gpt-5.6-terra
translatedAt: 2026-10-04T10:35:31.538Z
translationReview: required
---

# DNS : résolution, délégation et exploitation

DNS répond à la question de savoir quelle information est publiée pour un nom donné et qui en est responsable. Le système est à la fois un espace de noms hiérarchique, une base de données distribuée et un protocole de requête binaire. Il fournit non seulement des adresses IP, mais aussi des serveurs de noms, des destinations de messagerie, des points de terminaison de services, des clés et des politiques. Considérer DNS uniquement comme une « résolution de noms » revient donc à ignorer précisément les enregistrements dont dépendent la messagerie et l’identité ([RFC 9499](https://datatracker.ietf.org/doc/html/rfc9499)).

Le chemin suivant commence au niveau du stub resolver d’une application, suit le cache et les délégations jusqu’au serveur autoritatif, puis ramène la réponse. Ce déroulement permet de situer TTL, Glue, repli TCP, DNSSEC, exploitation des zones et scénarios d’erreur typiques.

Pour les administrateurs de messagerie, DNS est un système de pilotage en amont. Un MTA y détermine la prochaine destination de messagerie, l’authentification de l’expéditeur y lit les politiques et les clés, les procédures de certificats peuvent intégrer des données sécurisées par DNSSEC, et les clients d’annuaire ou Kerberos recherchent des services via des enregistrements SRV. DNS ne vérifie toutefois pas si le service trouvé est sain. Une réponse MX syntaxiquement correcte peut pointer vers un écouteur SMTP inaccessible ; une recherche A réussie ne dit rien sur TLS, l’authentification ou l’état de l’application.

## Architecture et rôles

La responsabilité des noms est organisée sous forme d’arbre. À la racine sans nom `.` commencent les domaines de premier niveau tels que `ch.`, suivis de domaines délégués et d’autres labels. Un point final rend un nom complet et empêche l’ajout de suffixes de recherche locaux. Cette petite différence d’écriture est pertinente en exploitation : `mail.example.ch` peut être complété par un domaine de recherche sur un client, `mail.example.ch.` ne le peut pas.

Un **domaine** est une partie de l’espace de noms. Une **zone**, en revanche, est un ensemble de données administrativement cohérent pour lequel un serveur autoritatif assume une responsabilité locale. Une délégation extrait une zone enfant de la zone parente. Le parent publie à cet effet un RRset NS et, lorsqu’un nom de serveur de noms se trouve dans la zone enfant déléguée, les données A ou AAAA nécessaires pour l’atteindre en tant que **Glue**. La distinction fondamentale entre espace de noms, zones et délégation provient de [RFC 1034](https://datatracker.ietf.org/doc/html/rfc1034#section-4.2); les termes actuels sont résumés dans [RFC 9499, section 7](https://datatracker.ietf.org/doc/html/rfc9499#section-7).

Quatre rôles logiques expliquent le chemin de résolution :

| Rôle | Connaissances et tâche | Limite d’exploitation importante |
|---|---|---|
| Stub resolver | reçoit la requête de l’application et la transmet à un resolver configuré | les suffixes de recherche, le fichier hosts local et le cache client peuvent influencer le résultat avant le DNS proprement dit |
| Resolver récursif | fournit une réponse finale depuis le cache ou via des requêtes itératives | constitue une frontière de confiance pour le cache, le filtrage, la journalisation et la validation DNSSEC |
| Redirecteur | prend en charge les requêtes récursives d’un autre resolver | déplace la résolution et l’observabilité vers un autre opérateur |
| Serveur autoritatif | répond à partir de zones chargées localement et positionne le bit AA dans les réponses autoritatives | ne connaît pas l’état de santé des services et ne doit pas proposer de récursion ouverte pour des noms tiers |

Un produit peut implémenter plusieurs rôles, mais il convient tout de même de les considérer séparément en exploitation. Une défaillance du service autoritatif affecte la publication des zones propres ; une défaillance du service récursif affecte la résolution de noms des propres clients. Des processus, adresses ou domaines de défaillance communs rendent cette distinction plus difficile.

## Résolution de noms étape par étape

Une application n’interroge normalement pas elle-même les serveurs racine et autoritatifs. Son stub resolver transmet une requête récursive à un resolver. Si celui-ci ne possède pas d’entrée de cache exploitable, il suit les délégations depuis la racine, via le domaine de premier niveau, jusqu’à la zone responsable. Chaque referral indique le RRset NS suivant et, si nécessaire, les adresses Glue. Le resolver compose la réponse finale à partir de ces éléments, valide DNSSEC le cas échéant et met le résultat en cache ([RFC 1034, section 4.3](https://datatracker.ietf.org/doc/html/rfc1034#section-4.3)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1016" src="/images/kb-interaktiv-dns-aufloesung.svg?v=20260813" title="Interaktive Infografik: DNS-Auflösung von Stub Resolver über Cache, Root und Delegationen bis zur autoritativen Antwort" loading="lazy">
  <a href="/images/kb-interaktiv-dns-aufloesung.svg?v=20260813">Ouvrir l’infographie sur la résolution DNS</a>
</iframe>

Le parcours linéaire représenté est un modèle de démarrage à froid. Dans le cache chaud d’un resolver, les délégations racine et TLD sont généralement déjà présentes, de sorte qu’une partie seulement des étapes est nécessaire. La minimisation QNAME, le forwarding, les zones locales ou les caches DNSSEC agressifs peuvent également modifier le chemin visible des paquets. L’essentiel reste d’attribuer chaque observation à un rôle : une réponse de cache non autoritative ne prouve pas ce que le serveur autoritatif responsable délivre à cet instant.

## Structure technique d’un message DNS

Sur le réseau, un message DNS se compose d’un en-tête, de sections Question, Answer, Authority et Additional. La question désigne QNAME, QTYPE et QCLASS. Les réponses et renvois apparaissent sous forme de Resource Records dans les autres sections. Les flags indiquent notamment l’autorité, la demande de récursion et la troncature ; le Response Code décrit le résultat. L’analyse ne porte donc pas uniquement sur le texte de la réponse, mais aussi sur le serveur dont elle provient et les flags qui l’accompagnent ([RFC 1035, section 4.1](https://datatracker.ietf.org/doc/html/rfc1035#section-4.1)).

Pour un diagnostic, les éléments suivants sont particulièrement pertinents :

| Signal | Signification | Question typique de l’administrateur |
|---|---|---|
| `AA` | la réponse est autoritative pour le nom traité | La zone responsable a-t-elle été interrogée directement, ou seulement un cache ? |
| `TC` | la réponse a été tronquée pour le transport utilisé | La répétition via TCP fonctionne-t-elle et le pare-feu autorise-t-il TCP/53 ? |
| `RD` / `RA` | récursion demandée / proposée par le serveur | Un serveur autoritatif a-t-il été utilisé par erreur comme resolver ? |
| `AD` | le validateur répondant considère les données comme authentifiées | Le resolver est-il digne de confiance et le transport vers lui est-il protégé ? |
| `CD` | le client demande à ne pas rejeter les erreurs de validation au niveau du resolver | DNSSEC est-il en cours de validation ou examine-t-on seulement les données brutes ? |
| `RCODE` | statut de résultat tel que NOERROR, NXDOMAIN, SERVFAIL ou REFUSED | Le nom est-il erroné, le type absent, la résolution perturbée ou la requête rejetée par une politique ? |

EDNS(0) ajoute, au moyen d’un pseudo-**enregistrement OPT**, des flags, options et une charge utile UDP annoncée plus importante, sans remplacer le format de base ([RFC 6891](https://datatracker.ietf.org/doc/html/rfc6891)). Le bit DNSSEC DO se trouve dans ce champ de flags étendu, et non dans l’en-tête DNS d’origine.

## Resource Records et RRsets

Le contenu métier se trouve dans les Resource Records. Le nom propriétaire, le type et la classe déterminent à quoi appartiennent les données ; TTL et RDATA fournissent la durée de cache et la valeur spécifique au type. Tous les enregistrements ayant le même propriétaire, le même type et la même classe forment un RRset et partagent un TTL. Plusieurs valeurs MX, A ou AAAA constituent donc un ensemble mis en cache conjointement, et non des objets individuels pouvant être gérés indépendamment ([RFC 2181, section 5](https://datatracker.ietf.org/doc/html/rfc2181#section-5)).

| Type | Fonction | Limite importante |
|---|---|---|
| `SOA` | métadonnées de zone, Serial, Refresh/Retry/Expire et paramètre de cache négatif | un RR SOA par zone à l’apex ; la modification du Serial pilote la synchronisation avec les secondaires |
| `NS` | serveurs autoritatifs d’une zone ou d’une délégation | la cible d’un NS ne doit pas être un alias |
| `A` / [`AAAA`](https://datatracker.ietf.org/doc/html/rfc3596#section-2.1) | adresse IPv4 ou IPv6 d’un nom | ne dit rien sur le port de service ou l’accessibilité |
| `CNAME` | alias d’un nom vers un nom canonique | ne doit en principe pas coexister avec d’autres données chez le même propriétaire |
| `MX` | Mail Exchanger avec préférence | la cible doit pouvoir être résolue en A/AAAA et ne doit pas être un CNAME |
| `PTR` | résolution inverse, généralement sous `in-addr.arpa.` ou `ip6.arpa.` | la zone inverse appartient habituellement au détenteur de l’adresse, et non à l’opérateur de la zone directe |
| `TXT` | une ou plusieurs chaînes de caractères sans sémantique inter-protocole | l’interprétation n’apparaît qu’avec SPF, DKIM, DMARC ou un autre mécanisme |
| [`SRV`](https://datatracker.ietf.org/doc/html/rfc2782) | service, transport, priorité, poids, port et cible | le client doit implémenter la sémantique SRV du service concerné |
| [`CAA`](https://datatracker.ietf.org/doc/html/rfc8659) | politique d’autorités de certification pour les noms de domaine | n’est ni un chiffrement du transport ni un certificat de serveur |
| `DS`, `DNSKEY`, `RRSIG`, `NSEC` | chaîne de confiance DNSSEC, clés, signatures et non-existence authentifiée | protège l’intégrité des données DNS, pas leur confidentialité |

Le texte d’un fichier de zone n’est que la **forme de présentation**. Sur le réseau, les noms sont transmis par labels, les nombres en binaire et certains noms peuvent être compressés. Lorsqu’on copie une chaîne depuis une interface, il faut donc distinguer si celle-ci a déjà assemblé des guillemets, séquences d’échappement ou plusieurs chaînes TXT en une charge utile logique.

## Délégation, Glue et autorité

Par une délégation, une zone transmet la responsabilité d’un sous-arbre. Le parent publie le RRset NS de la zone enfant ; la zone enfant indique elle-même à nouveau ses serveurs autoritatifs. Si les deux côtés divergent, les resolvers peuvent emprunter des chemins différents. Les adresses Glue du parent ne résolvent que le problème de l’œuf et de la poule de l’accessibilité. Les valeurs A/AAAA autoritatives au nom du serveur de noms restent des données indépendantes, avec leur propre TTL et leur propre maintenance.

Le **Glue in-bailiwick** est particulièrement critique : si `example.ch.` est délégué à `ns1.example.ch.`, le resolver a besoin de l’adresse de `ns1.example.ch.` avant de pouvoir interroger la zone enfant. Sans Glue, une boucle de résolution se produirait. Si la délégation pointe au contraire vers `ns1.provider.net.`, son adresse peut être déterminée via une autre chaîne de délégation.

Lors d’un changement de serveur de noms, il faut donc vérifier au minimum quatre états : la nouvelle zone est chargée sur tous les serveurs, le RRset NS enfant est adapté, la délégation parent est adaptée et le Glue nécessaire est actualisé. Ce n’est qu’ensuite que les anciens serveurs doivent être retirés de l’exploitation ou de l’accessibilité.

## Cache, TTL et réponses négatives

Après la résolution, une réponse continue d’exister dans les caches. Son TTL est la durée maximale d’utilisation à partir du moment où chaque resolver l’a apprise. Il n’existe donc pas de décompte commun à l’échelle mondiale. Les stub resolvers, redirecteurs, resolvers récursifs et applications peuvent abandonner le même ancien RRset à des moments différents. Une réduction du TTL avant une migration n’aide que pour les réponses rechargées ensuite ; les données déjà mises en cache ne peuvent pas être rappelées.

La non-existence est également mise en cache. **NXDOMAIN** signifie que le nom demandé n’existe pas ; **NODATA** est une réponse NOERROR dans laquelle le nom existe, mais sans RRset du type interrogé. Le serveur autoritatif place son RR SOA dans la section Authority pour ces deux réponses. La durée du cache négatif est le minimum entre le TTL du SOA et SOA.MINIMUM ([RFC 2308, sections 3 à 5](https://datatracker.ietf.org/doc/html/rfc2308#section-3)). Cela explique pourquoi un sélecteur DKIM ou un nom d’hôte nouvellement créé peut continuer à sembler absent après une tentative précédente ayant échoué.

`SERVFAIL` doit être distingué de cela : le resolver n’a pas pu produire de réponse exploitable. Parmi les causes figurent des timeouts vers les serveurs autoritatifs, une délégation défectueuse, une erreur de validation DNSSEC ou une limitation interne de ressources. `REFUSED` signifie en revanche que le serveur interrogé n’exécute pas l’opération en raison de sa politique. Un outil de diagnostic doit donc afficher le RCODE, le bit AA, le serveur répondant et les sections ; une sortie indiquant seulement « pas d’adresse » masque des différences essentielles.

## Transport : UDP, TCP et chemins de resolver chiffrés

Pour cet échange, le chemin réseau doit autoriser UDP et TCP sur le port 53. TCP n’est pas limité aux transferts de zone : un resolver peut l’utiliser directement et doit pouvoir s’y replier après une réponse UDP tronquée. Le blocage de TCP n’apparaît donc souvent qu’avec des RRsets volumineux, riches en DNSSEC ou comportant de nombreuses réponses. Les petites requêtes A restent vertes et donnent une image trompeuse ([RFC 7766](https://datatracker.ietf.org/doc/html/rfc7766)).

Sans EDNS, la charge utile DNS sur UDP est limitée à 512 octets. EDNS permet au demandeur d’annoncer une charge utile recevable plus importante. Une valeur trop élevée peut toutefois imposer une fragmentation IP ; si un chemin perd ou bloque des fragments, on obtient le scénario typique où les petites réponses fonctionnent et les grandes expirent. La spécification EDNS recommande de tenir compte de la capacité de réception réelle et du chemin, et de se replier vers des valeurs plus petites ou TCP en cas de problème ([RFC 6891, section 6.2](https://datatracker.ietf.org/doc/html/rfc6891#section-6.2)).

[DNS over TLS](https://datatracker.ietf.org/doc/html/rfc7858), [DNS over HTTPS](https://datatracker.ietf.org/doc/html/rfc8484) ou [DNS over QUIC](https://datatracker.ietf.org/doc/html/rfc9250) chiffrent un chemin de transport DNS. Ces mécanismes ne modifient ni le contenu des zones ni la délégation et ne remplacent pas DNSSEC : le chiffrement du transport protège la connexion à un resolver, DNSSEC authentifie les données le long de la chaîne de délégation. Le DNS classique sur le port 53 n’est pas chiffré ; la terminologie commune des types de transport est définie dans [RFC 9499, section 6](https://datatracker.ietf.org/doc/html/rfc9499#section-6).

## DNSSEC et chaîne de confiance

DNSSEC complète la résolution de noms par une origine et une intégrité vérifiables. Une zone signe les RRsets avec RRSIG et publie les clés publiques sous forme de DNSKEY. Le parent relie la zone enfant à la chaîne de confiance supérieure par DS ; NSEC ou NSEC3 peut aussi prouver la non-existence. Un resolver validant commence à son Trust Anchor et vérifie cette chaîne jusqu’à la réponse. Le contenu reste public : DNSSEC ne chiffre aucune requête ([RFC 4033](https://datatracker.ietf.org/doc/html/rfc4033), [RFC 4034](https://datatracker.ietf.org/doc/html/rfc4034)).

Pour l’exploitation, les fichiers de clés ne sont pas les seuls éléments pertinents, plusieurs états liés dans le temps doivent être pris en compte :

- Les RRSIG ont un début et une expiration ; une heure système erronée ou un échec de re-signature peut rendre toute une zone **bogus**.
- Un DS dans le parent doit correspondre à une DNSKEY exploitable de la zone enfant. Un DS orphelin est pire pour les resolvers validants qu’une délégation délibérément non signée.
- Lors des rollovers, publication, signature, modification du parent, TTL et durées de cache doivent être planifiés comme une machine à états.
- Un resolver validant fournit souvent SERVFAIL pour des données bogus. Un test sans validation peut afficher simultanément une réponse apparemment normale.

Le flag AD seul n’est fiable qu’à la hauteur du resolver et du chemin qui mène à lui. Pour une vérification indépendante, un administrateur doit examiner la chaîne avec un outil validant et localiser la transition défectueuse entre DS, DNSKEY et RRSIG.

## Exploitation autoritative et flux de données

Derrière la réponse autoritative se trouve son propre chemin de distribution. Dans un modèle classique, une source primaire gère la zone, DNS NOTIFY informe les secondaires d’un nouveau SOA Serial et AXFR ou IXFR transfèrent respectivement les données complètes ou incrémentielles. Un écouteur sain ne prouve donc pas encore que le serveur a chargé la nouvelle zone. Serial, statut de transfert et réponse de chaque nœud autoritatif doivent être considérés ensemble ([RFC 1996](https://datatracker.ietf.org/doc/html/rfc1996), [RFC 5936](https://datatracker.ietf.org/doc/html/rfc5936), [RFC 1995](https://datatracker.ietf.org/doc/html/rfc1995)).

La zone peut être générée à partir de fichiers texte, d’une base de données, d’une API, d’Active Directory ou d’un pipeline Git/CI. Ce choix d’implémentation ne modifie pas le protocole DNS sur le réseau, mais détermine les limites transactionnelles, l’auditabilité et la récupération. RFC 2136 définit les mises à jour dynamiques atomiques avec des prérequis ; un appel d’API de fournisseur constitue en revanche un protocole de contrôle distinct et doit documenter ses propres règles de cohérence et de gestion des erreurs ([RFC 2136](https://datatracker.ietf.org/doc/html/rfc2136)).

Une sauvegarde DNS n’est exploitable que si elle permet de restaurer la zone réellement servie. Selon la plateforme, cela inclut :

- la zone ou la base de données source, y compris le SOA Serial et le journal dynamique ;
- la configuration du serveur, les vues, ACL, redirecteurs et associations de catalogue ;
- les secrets TSIG, les clés privées DNSSEC et les états des rollovers automatiques ;
- les données parent hors de sa propre zone, en particulier délégation, Glue et DS ;
- une méthode testée pour réapprovisionner les secondaires et vérifier sémantiquement les données de zone.

Les secondaires sont des copies de disponibilité, mais pas automatiquement une sauvegarde historique. Une modification erronée ou malveillante peut être rapidement répliquée sur tous les serveurs autoritatifs via NOTIFY et transfert de zone.

## Implémentations et pile technologique

DNS ne désigne pas un daemon unique. La pile technologique commune se compose de noms et RRsets, d’un format binaire de requête/réponse, d’UDP et TCP, de logique de cache et, en option, de DNSSEC. Les serveurs autoritatifs, resolvers récursifs, redirecteurs et DNS managés implémentent ces composants différemment. Pour l’exploitation, le rôle d’un produit compte donc d’abord, puis son langage ou son packaging :

| Implémentation | Rôle principal | Priorité technique |
|---|---|---|
| [BIND 9](https://bind9.readthedocs.io/en/latest/) | autoritatif et/ou récursif | serveur de noms universel, fichiers de zone, mises à jour dynamiques, DNSSEC et outils de diagnostic |
| [Unbound](https://unbound.docs.nlnetlabs.nl/en/latest/manpages/unbound.html) | resolver récursif avec cache | pipeline modulaire de résolution, validation et cache ; pas de rôle autoritatif principal |
| [Knot DNS](https://www.knot-dns.cz/) | autoritatif | autoritatif uniquement, traitement parallèle, transfert de zone, DDNS et DNSSEC |
| [Windows Server DNS](https://learn.microsoft.com/windows-server/networking/dns/dns-overview) | autoritatif et récursif | zones facultativement intégrées à AD, mises à jour dynamiques sécurisées, politiques, cache et forwarding |

La séparation des rôles est plus importante que le nom du produit. Pour une plateforme autoritative publique, la fourniture des zones, les secondaires, la signature DNSSEC et la résilience DDoS comptent ; pour un resolver d’entreprise, ce sont le cache, le forwarding, les espaces de noms internes, les politiques, la confidentialité et la validation.

## DNS dans l’exploitation de messagerie et d’identité

En messagerie, une réponse DNS devient directement un chemin de livraison. Un MTA émetteur interroge les RRsets MX du domaine destinataire, préfère la valeur la plus faible et traite les préférences identiques comme équivalentes. Ce n’est que lorsqu’aucun MX n’existe que le domaine lui-même constitue une destination implicite. Dès que des enregistrements MX sont présents, une valeur A/AAAA à l’apex du domaine ne les remplace pas. Chaque cible MX nécessite ses propres adresses et ne doit pas être un alias ; un domaine qui n’accepte pas de messagerie publie le Null MX `0 .` ([RFC 5321, section 5](https://datatracker.ietf.org/doc/html/rfc5321#section-5), [RFC 2181, section 10.3](https://datatracker.ietf.org/doc/html/rfc2181#section-10.3), [RFC 7505](https://datatracker.ietf.org/doc/html/rfc7505)).

Les enregistrements PTR sont résolus via l’arbre d’adresses inversé. Les zones directe et inverse ont souvent des propriétaires différents ; les modifications doivent donc être coordonnées entre l’opérateur du domaine et celui de l’adresse IP. Un PTR est un nom, pas une preuve cryptographique d’identité. Les plateformes de messagerie destinataires peuvent utiliser une résolution directe/inverse cohérente comme signal, mais leur politique concrète de réputation ou d’acceptation n’est pas une propriété DNS.

SPF, DKIM et DMARC utilisent DNS comme canal de publication, mais définissent leurs propres règles d’évaluation. SPF lit une seule charge utile TXT logique et limite les termes qui provoquent des requêtes DNS ; DKIM adresse les clés via des sélecteurs ; DMARC se trouve sous `_dmarc`. Ces mécanismes sont traités fonctionnellement dans [SPF, DKIM et DMARC](/kb/mail-auth). L’administrateur DNS doit surtout maîtriser correctement pour eux le nom propriétaire, la répartition des chaînes TXT, la taille de réponse, le TTL, la délégation et les durées de cache négatif.

[LDAP](/kb/ldap) et [Kerberos](/kb/kerberos) utilisent également souvent des enregistrements SRV pour découvrir les services. Un enregistrement SRV contient, outre la cible et le port, une priorité et un poids. Ces valeurs ne constituent pas une configuration universelle d’équilibrage de charge ; seuls les clients qui implémentent le mécanisme SRV correspondant les interprètent.

## Diagnostic

Le diagnostic ne commence donc jamais par « DNS ne fonctionne pas », mais par le nom, le type, la classe, le serveur interrogé, le transport et le moment. Les resolvers d’entreprise, les resolvers publics et les serveurs autoritatifs peuvent fournir temporairement ou en raison du Split DNS et des politiques des réponses différentes. Cette divergence n’est pas du bruit de mesure, mais l’indice le plus important de l’endroit où le chemin de résolution diverge.

### Interroger les RRsets de manière ciblée

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für gezielte DNS-Abfragen">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$server = "192.0.2.53"
Resolve-DnsName example.ch. -Type SOA -Server $server -DnsOnly
Resolve-DnsName example.ch. -Type MX -Server $server -DnsOnly
Resolve-DnsName _dmarc.example.ch. -Type TXT -Server $server -DnsOnly
Resolve-DnsName 25.113.0.203.in-addr.arpa. -Type PTR -Server $server -DnsOnly</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">server=192.0.2.53
dig @"$server" example.ch. SOA +noall +answer +authority
dig @"$server" example.ch. MX +noall +answer +authority
dig @"$server" _dmarc.example.ch. TXT +noall +answer +authority
dig @"$server" -x 203.0.113.25 +noall +answer +authority</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) peut forcer un serveur spécifique, un type d’enregistrement, le DNS pur et TCP. [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) affiche en outre les flags, sections, RCODE, le serveur répondant et le temps de requête. Sans `-Server` ou `@server` explicite, c’est le resolver configuré qui est testé, et pas nécessairement la source autoritative.

### Vérifier séparément TCP et DNSSEC

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für DNS-Transport- und DNSSEC-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$server = "192.0.2.53"
Resolve-DnsName example.ch. -Type MX -Server $server -DnsOnly -TcpOnly
Resolve-DnsName example.ch. -Type DNSKEY -Server $server -DnsOnly -DnssecOk</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">server=192.0.2.53
dig @"$server" example.ch. MX +tcp
dig @"$server" example.ch. DNSKEY +dnssec
delv @"$server" example.ch. MX</code></pre>
  </div>
</div>

[`delv`](https://bind9.readthedocs.io/en/latest/manpages.html#delv-dns-lookup-and-validation-utility) utilise la logique de resolver et de validateur BIND pour vérifier une chaîne DNSSEC. Une requête réussie avec `+dnssec` prouve en revanche uniquement que des données DNSSEC ont été demandées et fournies ; elle ne valide pas automatiquement la chaîne. Le test TCP séparé détecte les pare-feu qui autorisent UDP/53 mais bloquent TCP/53.

### Vider de manière ciblée les caches locaux

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem zum Leeren des lokalen DNS-Caches">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Clear-DnsClientCache</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sudo resolvectl flush-caches
resolvectl statistics</code></pre>
  </div>
</div>

[`Clear-DnsClientCache`](https://learn.microsoft.com/powershell/module/dnsclient/clear-dnsclientcache) vide le cache du client DNS Windows. [`resolvectl`](https://man7.org/linux/man-pages/man1/resolvectl.1.html) contrôle le cache de `systemd-resolved`; sur les systèmes équipés de `nscd`, `dnsmasq`, d’un Unbound local ou d’un cache applicatif, un autre cache est responsable. Un vidage du client ne modifie jamais le cache d’un resolver en amont.

### Attribuer le scénario d’erreur au point de transition

| Observation | Niveau probable | Vérification suivante |
|---|---|---|
| un resolver fournit une ancienne valeur, les serveurs autoritatifs la nouvelle | cache positif | TTL restant, chaîne de redirecteurs, cache applicatif |
| NXDOMAIN persiste après création d’un enregistrement | cache négatif ou mauvaise zone | SOA dans la réponse négative, délégation parent, nom propriétaire |
| NOERROR sans Answer | le nom existe, le type demandé est absent | chaîne CNAME, QTYPE exact, SOA NODATA |
| seules les réponses volumineuses ou signées expirent | EDNS, fragmentation ou repli TCP | `TC`, taille EDNS plus petite, test TCP explicite, pare-feu |
| un resolver validant fournit SERVFAIL, un resolver non validant une réponse | DNSSEC bogus | correspondance DS/DNSKEY, temps RRSIG, algorithme, heure système |
| les serveurs autoritatifs fournissent des Serials différents | réplication | NOTIFY, IXFR/AXFR, ACL de transfert, journal, source primaire |
| les clients publics et internes voient des destinations différentes | Split DNS ou politique de resolver | resolver interrogé, attribution de vue, sous-réseau client, redirecteur |
| MX existe, la livraison échoue avant SMTP | nom de destination dérivé ou transport | préférence MX, A/AAAA de la cible MX, TCP/25, interdiction CNAME |

## Monitoring et critères d’exploitation

Le monitoring doit reproduire le même chemin. Une seule recherche A contre le resolver local par défaut ne détecte ni une délégation défectueuse, ni TCP bloqué, des signatures expirées ou un secondaire avec un Serial ancien. Pour la messagerie et l’identité, les signaux suivants doivent donc au minimum être relevés séparément :

- réponses autoritatives de chaque NS publié via UDP et TCP, y compris bit AA et SOA Serial ;
- délégation parent, RRset NS enfant, Glue et, pour les zones signées, transition DS/DNSKEY ;
- latence récursive, taux de cache hit, taux de timeout, SERVFAIL, REFUSED et NXDOMAIN ;
- taille de réponse, troncature, erreurs EDNS et repli TCP ;
- dates d’expiration des RRSIG, état du rollover de clés et files d’attente de signature ;
- succès de NOTIFY, AXFR et IXFR ainsi que l’ancienneté de la zone sur chaque secondaire ;
- RRsets fonctionnels tels que MX, adresses A/AAAA associées, PTR et noms TXT nécessaires à [l’authentification de la messagerie](/kb/mail-auth).

Un test synthétique doit vérifier à la fois le chemin normal du client et la source autoritative. C’est la seule façon de distinguer si une perturbation se trouve dans le jeu de données publié, la délégation, le cache d’un resolver ou l’application.

## Sécurité et domaines de défaillance

Les deux rôles de serveur requièrent des mesures de protection différentes. La récursion ne doit être accessible qu’aux clients de confiance ; un resolver ouvert peut être utilisé pour des attaques par réflexion et amplification. Les serveurs autoritatifs doivent en revanche rester accessibles dans le monde entier, mais ne doivent pas résoudre récursivement des noms tiers arbitraires. Mélanger les rôles augmente la surface d’attaque et rend les causes de charge plus difficiles à identifier ([RFC 5358](https://datatracker.ietf.org/doc/html/rfc5358)).

DNSSEC protège les RRsets publiés contre des modifications non détectées, mais ni le processus serveur ni la disponibilité. **TSIG** authentifie des messages DNS individuels au moyen d’une clé partagée et est notamment utilisé pour les mises à jour et les transferts de zone ; il ne constitue pas une signature publique des données de zone ([RFC 8945](https://datatracker.ietf.org/doc/html/rfc8945)). Les ACL de transfert, secrets TSIG et clés privées DNSSEC sont des objets de protection distincts.

Le Split DNS et les réponses de politique peuvent être nécessaires, mais créent plusieurs vérités pour le même QNAME/QTYPE. Il faut documenter au minimum le critère d’attribution, la zone source, le chemin de forwarding, le comportement DNSSEC et le monitoring par vue. Sinon, une divergence voulue sera interprétée lors de la prochaine perturbation comme une erreur de cache ou un problème de réplication.

## Histoire technique

Avant DNS, Internet distribuait des tables d’hôtes gérées de manière centralisée. Avec l’augmentation du nombre de réseaux, d’hôtes et d’opérateurs indépendants, cette méthode est devenue un goulot d’étranglement. En 1983, Paul Mockapetris a décrit dans RFC 882 et RFC 883 un service de noms hiérarchique et délégable. RFC 1034 et RFC 1035 ont remplacé cette version en 1987 et constituent encore, sous STD 13, le cœur du système.

La première implémentation de serveur fonctionnelle, **Jeeves**, a tourné en 1983/84 sur des systèmes DEC-TOPS-20. Peu après, l’University of California, Berkeley, a créé sous financement DARPA le Berkeley Internet Name Domain Package **BIND** pour Unix. BIND 8 est apparu en 1997, BIND 9 en septembre 2000 comme une réécriture majeure. ISC documente cette histoire dans [A Brief History of the DNS and BIND](https://bind9.readthedocs.io/en/latest/history.html).

Le protocole s’est développé progressivement sans remplacer le cœur hiérarchique : NOTIFY et IXFR ont accéléré la synchronisation des zones dans les années 1990, EDNS a étendu le modèle de message en 1999 et a été consolidé plus tard dans RFC 6891, DNSSEC a reçu en 2005 les mécanismes DNSKEY/DS/RRSIG aujourd’hui fondamentaux, et des transports de resolver chiffrés sont venus s’ajouter avec DoT, DoH et DoQ. DNS n’est donc pas un protocole figé de 1987, mais un système extensible dont la rétrocompatibilité et les longs états de cache façonnent chaque modification en exploitation.

## Sources

- [RFC Editor – STD 13: Domain Name System](https://www.rfc-editor.org/info/std13)
- [RFC 9499 – DNS Terminology](https://datatracker.ietf.org/doc/html/rfc9499) – termes actuels relatifs aux rôles, zones, cache, DNSSEC et transports.
- [RFC 1034 – Domain Names: Concepts and Facilities](https://datatracker.ietf.org/doc/html/rfc1034) – espace de noms, zones, délégation, resolvers et rôles de serveur.
- [RFC 1035 – Domain Names: Implementation and Specification](https://datatracker.ietf.org/doc/html/rfc1035) – format sur le réseau, Resource Records, sections de messages et fichiers maîtres.
- [RFC 6891 – Extension Mechanisms for DNS (EDNS(0))](https://datatracker.ietf.org/doc/html/rfc6891) – OPT, flags étendus et charge utile UDP.
- [RFC 2181, section 5](https://datatracker.ietf.org/doc/html/rfc2181)
- [`AAAA`](https://datatracker.ietf.org/doc/html/rfc3596)
- [RFC 2782 – A DNS RR for specifying the location of services](https://datatracker.ietf.org/doc/html/rfc2782) – structure et sélection des enregistrements SRV.
- [RFC 8659 – DNS Certification Authority Authorization](https://datatracker.ietf.org/doc/html/rfc8659) – enregistrements CAA et évaluation par les autorités de certification.
- [RFC 2308 – Negative Caching of DNS Queries](https://datatracker.ietf.org/doc/html/rfc2308) – NXDOMAIN, NODATA, SOA et durée du cache négatif.
- [RFC 7766 – DNS Transport over TCP](https://datatracker.ietf.org/doc/html/rfc7766) – prise en charge TCP obligatoire et comportement des connexions.
- [RFC 7858 – DNS over TLS](https://datatracker.ietf.org/doc/html/rfc7858) – DNS sur TLS.
- [RFC 8484 – DNS Queries over HTTPS](https://datatracker.ietf.org/doc/html/rfc8484) – requêtes DNS sur HTTPS.
- [RFC 9250 – DNS over Dedicated QUIC Connections](https://datatracker.ietf.org/doc/html/rfc9250) – DNS sur QUIC.
- [RFC 4033 – DNS Security Introduction and Requirements](https://datatracker.ietf.org/doc/html/rfc4033) – objectifs de protection DNSSEC, validation et limites.
- [RFC 4034 – Resource Records for DNSSEC](https://datatracker.ietf.org/doc/html/rfc4034) – DNSKEY, DS, RRSIG et NSEC.
- [RFC 1996 – DNS NOTIFY](https://datatracker.ietf.org/doc/html/rfc1996) – notification des serveurs secondaires.
- [RFC 5936 – DNS Zone Transfer Protocol (AXFR)](https://datatracker.ietf.org/doc/html/rfc5936) – transferts complets de zone via TCP.
- [RFC 1995 – Incremental Zone Transfer (IXFR)](https://datatracker.ietf.org/doc/html/rfc1995) – synchronisation incrémentielle des zones.
- [RFC 2136 – Dynamic Updates in DNS](https://datatracker.ietf.org/doc/html/rfc2136) – modifications atomiques avec prérequis.
- [BIND 9](https://bind9.readthedocs.io/en/latest/)
- [Unbound Documentation – unbound(8)](https://unbound.docs.nlnetlabs.nl/en/latest/manpages/unbound.html) – cache récursif et validation DNSSEC.
- [Knot DNS](https://www.knot-dns.cz/) – implémentation autoritative uniquement et fonctions d’exploitation.
- [Microsoft Learn – DNS in Windows Server](https://learn.microsoft.com/windows-server/networking/dns/dns-overview) – rôles DNS Windows, intégration AD, cache et forwarding.
- [RFC 5321, section 5](https://datatracker.ietf.org/doc/html/rfc5321)
- [RFC 7505 – Null MX](https://datatracker.ietf.org/doc/html/rfc7505) – signalisation explicite des domaines ne recevant pas de messagerie.
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – requêtes DNS sous Windows.
- [ISC BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html) – options de requête, sélection de serveur et interprétation de sortie.
- [`Clear-DnsClientCache`](https://learn.microsoft.com/powershell/module/dnsclient/clear-dnsclientcache)
- [`resolvectl`](https://man7.org/linux/man-pages/man1/resolvectl.1.html)
- [RFC 5358 – Preventing Use of Recursive Nameservers in Reflector Attacks](https://datatracker.ietf.org/doc/html/rfc5358) – limitation de récursion et protection contre les abus.
- [RFC 8945 – Secret Key Transaction Authentication for DNS (TSIG)](https://datatracker.ietf.org/doc/html/rfc8945) – authentification de messages pour mises à jour et transferts.
- [ISC – A Brief History of the DNS and BIND](https://bind9.readthedocs.io/en/latest/history.html) – Jeeves, Berkeley, BIND 8 et BIND 9.
