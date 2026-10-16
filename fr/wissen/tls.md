---
title: "TLS : protocole, vérification des certificats et exploitation"
blatt: "tls"
description: "Transport Layer Security du point de vue de l’administrateur : Handshake et Record Layer, plan de clés TLS 1.3, validation des chemins X.509 et PKIX, SAN et SNI, magasins de confiance, intégration SMTP et LDAP, gestion des clés et diagnostic par couches."
fakten:
  - label: Nom complet
    wert: Transport Layer Security
    href: https://datatracker.ietf.org/doc/html/rfc9846
  - label: Objectifs de protection
    wert: Authentification · Confidentialité · Intégrité
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-1
  - label: Éléments du protocole
    wert: Handshake Protocol · Record Protocol · Alerts
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-1
  - label: Fondation
    wert: flux d’octets fiable et ordonné
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-1
  - label: Authentification
    wert: Serveur toujours · Client facultatif
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-1
  - label: Échange de clés
    wert: (EC)DHE · PSK · combiné
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-2
  - label: Protection des enregistrements
    wert: AEAD avec des Traffic Secrets distincts
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-5
  - label: Profil de certificat
    wert: X.509 v3 et validation de chemin PKIX
    href: https://datatracker.ietf.org/doc/html/rfc5280
  - label: Identité du service
    wert: ID de référence comparé à subjectAltName
    href: https://datatracker.ietf.org/doc/html/rfc9525
  - label: Modèles de démarrage
    wert: TLS implicite ou STARTTLS spécifique au protocole
    href: https://datatracker.ietf.org/doc/html/rfc8314
  - label: Reprise
    wert: Reprise de session basée sur PSK
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-2.2
  - label: Recommandation d’exploitation
    wert: BCP 195
    href: https://www.rfc-editor.org/info/bcp195
werbung:
  - tools
  - newsletter
ctaThemen:
  - e-mail-verschluesselung
  - smtp-mailflow
translationSourceHash: 50ea371b919edcc4811a303d26ec6c575ef8dc334ce6e681fdc3f74de9ca2058
translationModel: gpt-5.6-terra
translatedAt: 2026-10-14T09:42:00.491Z
translationReview: required
---

# TLS : protocole, vérification des certificats et exploitation

**Transport Layer Security** (TLS) protège une connexion individuelle entre deux points de terminaison. Le client et le serveur conviennent d’abord des procédés cryptographiques et des clés, puis SMTP, LDAP, HTTP ou un autre protocole applicatif transitent par ce canal protégé. TLS empêche l’écoute et les modifications non détectées sur cette connexion et authentifie au minimum le serveur. Il ne confirme toutefois ni l’expéditeur d’un e-mail ni ne garantit que le saut SMTP suivant utilise également TLS ([RFC 9846, Abschnitt 1](https://datatracker.ietf.org/doc/html/rfc9846#section-1), [RFC 3207, Abschnitt 6](https://datatracker.ietf.org/doc/html/rfc3207#section-6)).

En exploitation, plusieurs vérifications doivent donc réussir successivement : le bon hôte et le bon port doivent être accessibles, le client et le serveur doivent avoir des paramètres TLS communs, la chaîne de certificats doit mener à une ancre de confiance et le nom contacté doit figurer dans le certificat. Ce n’est qu’ensuite que débute le protocole applicatif proprement dit. Cet article suit précisément ce déroulement et indique à chaque étape ce qu’un administrateur doit observer, configurer et sécuriser.

## Position dans la pile de protocoles

Dans l’exploitation habituelle d’un serveur, TLS se situe entre [TCP](/kb/tcp) et le protocole applicatif. TCP fournit le flux de données ordonné ; TLS le protège ; [SMTP](/kb/smtp), [LDAP](/kb/ldap), IMAP ou HTTP déterminent quand TLS démarre et quel nom de serveur est attendu. Cette intégration explique pourquoi un port ouvert ne constitue pas encore un test TLS réussi et pourquoi un Handshake réussi ne prouve pas encore le bon fonctionnement d’un service de messagerie ou d’annuaire ([RFC 9846, Abschnitt 1](https://datatracker.ietf.org/doc/html/rfc9846#section-1)).

| Couche | Objet visible | Question d’administration typique |
|---|---|---|
| Sélection du nom et du service | Nom DNS, cible MX ou SRV, port | Quel point de terminaison a réellement été sélectionné ? |
| Transport | Connexion TCP et automate d’états | Les SYN, SYN-ACK et ACK arrivent-ils dans les deux sens ? |
| Handshake TLS | Version, Key Share, signature, certificats, Finished | Existe-t-il des paramètres communs et le pair est-il accepté ? |
| TLS Record Layer | Enregistrements chiffrés avec protection AEAD | Les enregistrements sont-ils transmis, rejetés ou interrompus par une alerte ? |
| Protocole applicatif | Dialogue SMTP, LDAP, IMAP ou HTTP | Quelles commandes et identités s’appliquent après le Handshake ? |

QUIC constitue une exception importante à la représentation classique de « TLS sur TCP » : il utilise le Handshake TLS 1.3, mais pas le TLS Record Layer ; QUIC protège ses propres paquets avec des clés dérivées de TLS ([RFC 9001](https://datatracker.ietf.org/doc/html/rfc9001)). Pour les protocoles de messagerie et d’annuaire considérés ici, le canal TLS basé sur TCP demeure le modèle pertinent.

## Architecture : Handshake, Record Layer et alertes

TLS accomplit deux tâches successives. Dans le **Handshake Protocol**, les pairs négocient des paramètres, s’authentifient et génèrent du matériel de clé commun. Ensuite, le **Record Protocol** découpe les données utiles en enregistrements protégés. Les **Alerts** signalent des erreurs de protocole ou une fermeture ordonnée ; en cas de panne réseau ou de processus brutale, la connexion TCP peut toutefois aussi se terminer sans alerte exploitable ([RFC 9846, Abschnitte 4 bis 6](https://datatracker.ietf.org/doc/html/rfc9846#section-4)).

Cette séparation est déterminante en exploitation. Une erreur de certificat relève de la phase d’authentification du Handshake. Un `bad_record_mac` ou un `decrypt_error` concerne le traitement cryptographique. Un délai d’attente sans ServerHello indique en revanche souvent une confusion de protocole, un filtrage, un mauvais port ou un serveur qui attend d’abord un dialogue en clair et une commande STARTTLS.

## Handshake TLS 1.3 étape par étape

Le Handshake TLS 1.3 complet peut se lire comme une conversation : le client fait une proposition, le serveur y sélectionne des éléments, prouve son identité et les deux parties confirment le protocole accompli jusque-là. Dès le **ServerHello**, les messages de Handshake suivants sont protégés. Avec les messages **Finished** des deux côtés, il est finalement établi que les deux parties possèdent la même transcription et les mêmes secrets ([RFC 9846, Abschnitt 2](https://datatracker.ietf.org/doc/html/rfc9846#section-2)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1056" src="/images/kb-interaktiv-tls-handshake.svg?v=20260813" title="Interaktive Infografik: TLS-1.3-Handshake, Zertifikatskette, Identitätsprüfung und Admin-Diagnosepunkte" loading="lazy">
  <a href="/images/kb-interaktiv-tls-handshake.svg?v=20260813">Ouvrir l’infographie sur le Handshake TLS et la vérification des certificats</a>
</iframe>

Le déroulement en détail :

1. Le **ClientHello** propose notamment les versions TLS prises en charge, les Cipher Suites, les algorithmes de signature et les groupes cryptographiques. Un `key_share` contient généralement déjà la partie publique éphémère du client. SNI peut transporter le nom de serveur souhaité et ALPN les protocoles applicatifs possibles.
2. Le **ServerHello** sélectionne la version, la Cipher Suite et le Key Share. La partie client et la partie serveur produisent le secret (EC)DHE commun. Si aucun groupe envoyé au préalable ne convient, le serveur peut demander un nouveau ClientHello avec **HelloRetryRequest**.
3. **EncryptedExtensions** confirme d’autres extensions négociées. Si le serveur demande un certificat client, **CertificateRequest** suit également.
4. **Certificate** transporte le certificat Leaf et normalement les certificats intermédiaires nécessaires au client. **CertificateVerify** signe la transcription du Handshake avec la clé privée correspondant au certificat Leaf. Le serveur prouve ainsi la possession de la clé ; le certificat seul ne la prouve pas.
5. **Finished** est un MAC sur la transcription effectuée jusqu’alors et confirme les secrets de Handshake dérivés. Le client valide les paramètres, le chemin de certificat et l’identité du service, puis envoie son Finished ; en cas d’authentification mutuelle demandée, il envoie en plus Certificate et CertificateVerify.
6. Des Traffic Secrets distincts sont dérivés pour les données applicatives. Les messages **NewSessionTicket** ultérieurs permettent une reprise basée sur PSK, sans que le serveur doive renvoyer le certificat initial à chaque Handshake ultérieur.

Le nombre observé d’allers-retours dépend donc de DNS, TCP, d’un Handshake TLS complet ou repris, de HelloRetryRequest et de l’application. Une capture de paquets doit mesurer ces phases séparément ; « TLS était lent » n’est pas un diagnostic sans cette répartition.

## Plan de clés et Record Layer

À partir du matériel (EC)DHE ou PSK convenu, TLS 1.3 ne génère pas une seule clé de session, mais plusieurs secrets distincts au moyen de **HKDF**. Le Handshake, les données applicatives, les exportations et les reprises ultérieures utilisent leur propre matériel ; de plus, le client et le serveur disposent de clés et de vecteurs d’initialisation distincts pour chaque direction. Cette séparation est importante pour l’analyse et la réponse aux incidents, car une clé de Record divulguée ne déchiffre pas automatiquement toutes les phases et toutes les directions ([RFC 9846, Abschnitt 7.1](https://datatracker.ietf.org/doc/html/rfc9846#section-7.1)).

Le Record Layer utilise exclusivement **Authenticated Encryption with Associated Data** (AEAD). Dans TLS 1.3, la Cipher Suite désigne l’algorithme AEAD et le hash pour HKDF ; le type de certificat, l’algorithme de signature et le groupe d’échange de clés sont négociés séparément. Cela diffère de TLS 1.2, où des noms tels que `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256` regroupent plusieurs décisions dans une seule suite.

| Propriété | TLS 1.2 | TLS 1.3 |
|---|---|---|
| Sémantique des Cipher Suites | Échange de clés, authentification, chiffrement et hash regroupés | Procédé AEAD et hash HKDF ; échange de clés et signature séparés |
| Forward Secrecy | dépend de la suite sélectionnée | négociation à clé publique uniquement avec (EC)DHE éphémère |
| Visibilité du Handshake | grandes parties en clair | chiffré après ServerHello |
| Reprise | Session ID ou ticket selon un modèle antérieur | modèle PSK unifié |
| 0-RTT | ne fait pas partie du protocole | facultatif, mais rejouable et inadapté aux opérations non idempotentes |

Le 0-RTT économise un aller-retour lors d’une reprise, mais présente des propriétés de rejeu plus faibles. Un serveur doit donc décider, selon l’application, quelles données précoces il accepte. Pour les opérations d’administration, de messagerie ou d’annuaire qui modifient un état, une faible latence ne doit pas être confondue avec la sécurité contre les rejouements ([RFC 9846, Abschnitte 2.3 und 8](https://datatracker.ietf.org/doc/html/rfc9846#section-2.3)).

## Certificat X.509 et clé privée

Un certificat X.509 est la description signée d’une clé publique. Il indique notamment l’émetteur, la période de validité, les usages autorisés et les noms pour lesquels la clé peut être utilisée. La clé privée correspondante ne fait pas partie du certificat et doit rester protégée chez l’exploitant. RFC 5280 définit le profil PKIX utilisé à cette fin, avec des champs tels que Key Usage, Extended Key Usage, Basic Constraints et Subject Alternative Name ([RFC 5280, Abschnitt 4](https://datatracker.ietf.org/doc/html/rfc5280#section-4)).

X.509 est encodé en binaire avec ASN.1 DER. Les lignes fréquemment visibles `BEGIN CERTIFICATE` et `END CERTIFICATE` constituent une enveloppe textuelle basée sur Base64, décrite par RFC 7468. Un conteneur **PKCS #12** peut en revanche transporter ensemble la clé privée, la chaîne de certificats et d’autres attributs ([RFC 7468](https://datatracker.ietf.org/doc/html/rfc7468), [RFC 7292](https://datatracker.ietf.org/doc/html/rfc7292)). Les extensions de fichiers telles que `.cer`, `.crt`, `.pem`, `.p12` ou `.pfx` sont des conventions, mais ne prouvent pas de manière fiable le contenu ou l’encodage.

Les extensions importantes ont des rôles différents :

| Extension | Question de vérification | Erreur d’exploitation fréquente |
|---|---|---|
| Subject Alternative Name | contient-elle l’identité DNS, IP, URI ou SRV attendue par le client ? | le nom de gestion ou l’alias du répartiteur de charge manque |
| Basic Constraints | le certificat peut-il agir comme CA et quelle peut être la profondeur du chemin ? | intermédiaire sans `CA=TRUE` ou mauvaise longueur de chemin |
| Key Usage | pour quelle opération cryptographique fondamentale la clé peut-elle servir ? | la CA ne peut pas signer ou le Leaf ne convient pas à l’usage de signature |
| Extended Key Usage | la clé est-elle autorisée pour l’authentification serveur ou client ? | un certificat S/MIME est utilisé par erreur comme certificat de serveur TLS |
| Authority / Subject Key Identifier | quelles variantes de clés aident à construire le chemin ? | mauvais intermédiaire avec le même Subject sélectionné |
| CRL Distribution Points / AIA | où le client trouve-t-il les informations de révocation ou d’émetteur ? | cible inaccessible depuis le réseau du serveur ou proxy absent |

## Chemin de certificat et frontière de confiance

Avec son **certificat Leaf**, un serveur envoie normalement les **certificats intermédiaires** nécessaires. Le client tente ainsi de construire un chemin vers une CA racine à laquelle il fait déjà confiance localement. C’est précisément ce dernier point qui est déterminant : une CA racine fournie par le serveur ne devient pas fiable pour cette seule raison. Le magasin de confiance du processus qui établit effectivement la connexion définit les ancres de confiance applicables ([RFC 5280, Abschnitt 6](https://datatracker.ietf.org/doc/html/rfc5280#section-6)).

La construction et la validation du chemin vont au-delà de trois simples coches. Le client doit notamment vérifier les signatures, les périodes de validité, Basic Constraints, Path Length, Name Constraints, Key Usage, Extended Key Usage, les extensions critiques, les politiques et, le cas échéant, la révocation. L’identité de service spécifique au protocole est ensuite vérifiée. Différents clients peuvent construire des chemins valides différents à partir du même certificat Leaf, car leurs magasins de confiance, le chargement AIA, les signatures croisées et les règles d’algorithmes diffèrent.

Le terme **Truststore** ne désigne donc pas un emplacement universel. Une application Windows peut utiliser Schannel et le magasin de certificats Windows, une application Java JSSE avec son propre fichier `cacerts` ou un fichier applicatif, un conteneur un bundle CA intégré et une appliance un magasin propriétaire. Même sur le même hôte, deux processus peuvent faire confiance différemment au même serveur. À chaque incident, il faut identifier le processus qui effectue réellement la validation et son Truststore.

La révocation constitue une décision de politique supplémentaire. Les CRL distribuent des listes signées de certificats révoqués ; OCSP répond aux requêtes de statut pour des certificats individuels. Le fait qu’un client vérifie, sa réaction à un délai d’attente et son utilisation d’OCSP Stapling dépendent de la bibliothèque et de l’application ([RFC 5280, Abschnitt 5](https://datatracker.ietf.org/doc/html/rfc5280#section-5), [RFC 6960](https://datatracker.ietf.org/doc/html/rfc6960)). « Le certificat est temporellement valide » ne signifie donc pas automatiquement « le certificat est accepté par ce client ».

## Identité du service : SAN, ID de référence et SNI

Une fois le chemin accepté, le client vérifie le nom du service. Le point de départ est l’**identité de référence**, généralement le nom d’hôte configuré ou la partie hôte d’une URL. Elle doit correspondre à une entrée du type adéquat dans le **Subject Alternative Name** du certificat Leaf. Pour les noms DNS, `dNSName` s’applique, et pour les adresses IP configurées directement, `iPAddress`. Le Common Name formulé librement n’est pas une solution de repli moderne. Les caractères génériques ne peuvent remplacer que l’étiquette gauche complète ([RFC 9525, Abschnitte 4 und 6](https://datatracker.ietf.org/doc/html/rfc9525#section-4)).

Il faut distinguer **Server Name Indication** (SNI) de cela. Le client envoie le nom de serveur souhaité dans le ClientHello afin qu’un listener partagé puisse sélectionner le certificat approprié et la configuration virtuelle adéquate. SNI indique au serveur ce que le client veut atteindre ; il ne remplace ni la comparaison SAN ni la validation du chemin ([RFC 6066, Abschnitt 3](https://datatracker.ietf.org/doc/html/rfc6066#section-3)). Un test sans SNI peut donc afficher le certificat par défaut, alors que le client de production en reçoit un autre.

**Application-Layer Protocol Negotiation** (ALPN) permet au client et au serveur de sélectionner un protocole de niveau supérieur au sein du Handshake TLS. Il est surtout visible pour HTTP et QUIC, mais ne constitue conceptuellement pas non plus une vérification de certificat ([RFC 7301](https://datatracker.ietf.org/doc/html/rfc7301)).

## TLS implicite et STARTTLS

Le protocole applicatif détermine le moment où le Handshake commence. Avec le **TLS implicite**, le ClientHello suit immédiatement l’établissement TCP. Avec **STARTTLS**, les deux côtés communiquent d’abord en clair et ne passent à TLS qu’après une commande spécifique au protocole. Envoyer directement un ClientHello à un port STARTTLS revient donc à utiliser TLS, mais au mauvais point du dialogue.

| Chemin de service | Port | Modèle de démarrage | Règle de protocole |
|---|---:|---|---|
| Relais SMTP entre MTA | 25 | ESMTP `STARTTLS` | après un Handshake réussi, SMTP est réinitialisé ; le client envoie à nouveau EHLO |
| Soumission de messages | 587 | STARTTLS | la soumission peut exiger TLS avant l’authentification |
| Soumission TLS implicite | 465 | TLS immédiat | décrit par RFC 8314 pour la soumission de messages |
| IMAP / IMAPS | 143 / 993 | STARTTLS / TLS immédiat | l’accès en clair est considéré comme obsolète pour les usages modernes |
| POP3 / POP3S | 110 / 995 | STLS / TLS immédiat | également des chemins distincts pour le clair et TLS implicite |
| LDAP / LDAPS | 389 / 636 | LDAP StartTLS Extended Operation / TLS immédiat | Bind et vérification d’identité suivent la politique de sécurité LDAP |
| HTTP / HTTPS | 80 / 443 | schémas URI distincts ; TLS immédiat sur 443 | Host/SNI, identité de certificat et version HTTP sont des décisions distinctes |

Les attributions de ports proviennent de la [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) ; la sémantique découle de la norme applicative correspondante. Le port 465 n’est donc pas indistinctement « SMTPS pour la distribution de serveur à serveur », mais le chemin TLS implicite enregistré pour la soumission.

SMTP STARTTLS définit un changement d’état particulièrement important. Avant la mise à niveau, la réponse EHLO et l’offre `STARTTLS` sont manipulables. Après le Handshake, les deux côtés doivent abandonner toute connaissance SMTP apprise auparavant, et le client envoie de nouveau EHLO ([RFC 3207, Abschnitt 4.2](https://datatracker.ietf.org/doc/html/rfc3207#section-4.2)). Pour LDAP, StartTLS est une Extended Operation au sein de la session LDAP ; ce n’est pas la même chose qu’un nouveau port TCP ([RFC 4511, Abschnitt 4.14](https://datatracker.ietf.org/doc/html/rfc4511#section-4.14)).

## Politique TLS pour SMTP

Pour SMTP sur le port 25, TLS est souvent opportuniste : le MTA expéditeur chiffre lorsque le pair propose STARTTLS et que la politique locale accepte le Handshake. Sans politique supplémentaire, un attaquant peut supprimer l’offre ou perturber le Handshake. Le MTA expéditeur doit alors décider s’il distribue en clair, réessaie ultérieurement ou abandonne définitivement. RFC 3207 précise ainsi qu’un saut SMTP protégé ne constitue pas une protection de bout en bout du message.

Deux mécanismes peuvent lier plus fortement le domaine destinataire et l’exigence TLS :

- **DANE for SMTP** publie des données TLSA sous DNSSEC et dérive l’identité à vérifier du chemin MX. Un résolveur validant et une chaîne DNSSEC intacte deviennent ainsi partie de la base de confiance ([RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672)).
- **MTA-STS** publie une politique via DNS et HTTPS. Elle indique les modèles MX autorisés et exige TLS validé par PKIX pendant la durée de vie de la politique ([RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461)). **SMTP TLS Reporting** rapporte les modèles agrégés de réussite et d’erreur de ces politiques ([RFC 8460](https://datatracker.ietf.org/doc/html/rfc8460)).

Ces modèles ne sont pas interchangeables. DANE ancre l’assertion dans DNSSEC et TLSA ; MTA-STS utilise Web-PKIX et une politique obtenue via HTTPS. Un administrateur doit donc surveiller, outre le certificat et le Cipher, [DNS](/kb/dns), la sélection MX, le cache de politiques et le canal de rapport.

## Authentification TLS mutuelle

**Mutual TLS** (mTLS) complète la preuve habituelle du serveur par un certificat client. Le serveur le demande avec CertificateRequest ; le client envoie sa chaîne et prouve avec CertificateVerify qu’il possède la clé privée. Le serveur vérifie ensuite le chemin, la validité, Extended Key Usage et son mappage local. Le certificat identifie d’abord un détenteur de clé. L’application décide toujours des droits de compte applicatif, LDAP ou de relais SMTP qui en découlent.

Dans les environnements hétérogènes, des problèmes de confiance asymétriques apparaissent fréquemment : le client fait confiance à la CA du serveur, mais le serveur ne fait pas confiance à la CA du client ; le serveur ne demande que certains émetteurs ; le certificat client ne possède que `serverAuth`; ou un répartiteur de charge termine TLS et ne transmet aucune identité client vérifiée au backend. Un Handshake réussi avec authentification du serveur ne prouve donc pas le bon fonctionnement de mTLS.

## Exploitation des clés, certificats et renouvellements

Un remplacement de certificat ne consiste pas seulement à remplacer un fichier. La clé, le certificat, la chaîne intermédiaire, la liaison au listener et l’état de déploiement doivent correspondre. Un cycle de vie robuste comprend au minimum :

1. **Inventaire :** relever le nom de service, le port, le modèle de démarrage, le processus terminateur, les SAN attendus, l’émetteur, le numéro de série, l’empreinte, `notBefore`/`notAfter`, le Key Store, le Truststore et l’automatisation responsable.
2. **Génération de clés :** générer une nouvelle paire de clés dans le contexte logiciel, TPM ou HSM prévu. Les clés privées restent confidentielles et nécessitent une protection, un contrôle d’accès et, selon le modèle de récupération, une stratégie documentée de sauvegarde ou de réémission ([NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)).
3. **Demande et émission :** générer une CSR avec les identités et paramètres corrects ; satisfaire à la politique de CA et aux justificatifs requis. ACME normalise la commande et le renouvellement automatisés, mais ne garantit pas encore une liaison de service correcte ([RFC 8555](https://datatracker.ietf.org/doc/html/rfc8555)).
4. **Installation :** placer le Leaf, le bundle intermédiaire approprié et la clé privée dans le format réellement lu par le processus. Vérifier les droits des fichiers, l’accès au mot de passe et la correspondance des paires de clés.
5. **Activation :** recharger le service ou basculer la liaison de manière atomique. Dans le cas d’un cluster, d’un répartiteur de charge et de plusieurs rôles de protocole, traiter séparément chaque terminateur TLS.
6. **Validation :** vérifier en externe et en interne avec le SNI, le port et le mode STARTTLS de production ; relever la chaîne fournie, le SAN, la validité, l’algorithme, la politique et le test applicatif.
7. **Remplacement :** ne retirer l’ancien certificat que lorsque tous les nœuds fournissent le nouveau et qu’aucune dépendance ne pointe plus vers l’empreinte ou l’ancien émetteur. En cas de compromission, le révoquer en outre et traiter la clé privée comme compromise.

Une sauvegarde doit correspondre à ce modèle. Une partie publique de certificat exportée ne rétablit pas un service s’il manque la clé privée, le mot de passe, la liaison au fournisseur, l’ACL ou l’objet HSM. Inversement, chaque copie non protégée de la clé privée augmente la surface d’attaque. La récupération des CA racines et émettrices, la récupération des clés serveur et la réémission rapide sont des procédures différentes et ne doivent pas être regroupées sous « sauvegarder les certificats ».

Les PKI Web publiques, les PKI Enterprise internes et les ancres de confiance autogérées ont des règles différentes d’émission et de durée de validité. Au lieu d’utiliser une valeur maximale statique comme objectif d’exploitation, la surveillance devrait mesurer le `notAfter` concret de chaque certificat fourni et prévoir suffisamment de temps pour l’émission, le déploiement, la validation et le retour en arrière.

## Pile d’implémentation et de technologies

La pile TLS réellement utilisée dépend du processus. Les services Windows peuvent utiliser Schannel, les applications Java JSSE, les services Unix OpenSSL ou GnuTLS et les appliances leur propre bibliothèque. Les Truststores, procédés pris en charge, outils de diagnostic et emplacements de configuration changent ainsi, bien que tous les pairs parlent « TLS ».

| Pile | Exécution et API | Source typique de confiance et de clés | Pertinence pour l’administration |
|---|---|---|---|
| [OpenSSL](https://docs.openssl.org/master/) ([Quellcode](https://github.com/openssl/openssl)) | bibliothèque native et CLI, principalement en C | fichier/répertoire CA, PEM, DER, PKCS #12, Provider | de nombreux services Unix et appliances ; `s_client`, `x509`, `verify` |
| [Schannel](https://learn.microsoft.com/windows-server/security/tls/tls-ssl-schannel-ssp-overview) | Windows Security Support Provider via SSPI | magasin de certificats Windows, clés CNG/CSP | les applications système et .NET peuvent utiliser la politique et les magasins de l’OS |
| [JSSE](https://docs.oracle.com/en/java/javase/24/security/java-secure-socket-extension-jsse-reference-guide.html) | API Java `SSLSocket`, `SSLEngine`, `SSLContext` et Provider | KeyStore, TrustStore, `cacerts`, PKCS #12 | une application JVM peut différer de la confiance du système d’exploitation |
| [GnuTLS](https://www.gnutls.org/documentation.html) | bibliothèque TLS native, API C et CLI | fichiers, PKCS #11, confiance système selon l’intégration | bibliothèque alternative dans les logiciels Unix et GNU |

Au-dessus se trouve la configuration de l’application. Un serveur de messagerie peut sélectionner le certificat et la politique par listener ou nom SNI ; un proxy inverse termine TLS devant le backend ; un produit Java fournit son propre JRE avec Truststore ; une image de conteneur fige un bundle CA au moment de sa construction. Affirmer que « le certificat est installé à l’échelle du système d’exploitation » n’est donc pas fiable sans preuve concernant le processus et la liaison.

La prise en charge des algorithmes est également l’intersection entre la version du protocole, la bibliothèque, le fournisseur cryptographique, le certificat, l’offre client et la politique locale. Un certificat ECDSA n’aide pas si le client ne prend pas en charge la signature ; une chaîne Cipher TLS 1.3 ne modifie aucune suite TLS 1.2 ; un paramètre Schannel global ne contrôle pas une exécution OpenSSL ou Java fournie séparément. [BCP 195](https://www.rfc-editor.org/info/bcp195) fournit la directive de sécurité IETF, mais l’activation concrète reste dépendante de l’implémentation et du cycle de vie.

## Diagnostic du port à l’application

Un diagnostic TLS ne commence donc pas par le certificat, mais par le chemin de connexion exact. Chaque test doit inclure le nom cible, l’adresse résolue, le port, le SNI, le mode STARTTLS, la bibliothèque cliente, le Truststore, le moment et la politique applicable. Seuls les tests avec les mêmes paramètres sont comparables.

### Vérifier séparément l’accessibilité TCP

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für TCP-Erreichbarkeit vor TLS">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) et [`nc`](https://man.openbsd.org/nc) vérifient ici uniquement la résolution de noms et l’établissement TCP. Un test réussi ne dit encore rien sur la version TLS, le certificat, SNI ou l’application.

### Tester SMTP STARTTLS avec le SNI de production

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-STARTTLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) est un client de diagnostic, non une preuve de la politique du MTA de production. `-starttls smtp` conduit le dialogue en clair jusqu’à la mise à niveau, `-servername` définit SNI, `-showcerts` affiche la liste de certificats envoyée par le serveur et `-verify_return_error` s’arrête en cas d’échec de validation. Pour un test d’identité complet, il convient d’indiquer en plus `-verify_hostname mail.example.ch` et le chemin CA effectivement utilisé. OpenSSL ne fait pas partie intégrante de Windows ; `openssl.exe` doit provenir d’une installation contrôlée.

### Distinguer LDAP StartTLS et TLS implicite

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für LDAP-TLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

Le premier appel attend TLS immédiatement sur 636, le second exécute la LDAP StartTLS Extended Operation sur 389. La réussite du test OpenSSL ne prouve pas encore que la bibliothèque LDAP de production utilise le même Truststore, la même politique de révocation ou la même identité de référence.

### Examiner localement le fichier de certificat et le chemin

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für lokale Zertifikatsprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`certutil`](https://learn.microsoft.com/windows-server/administration/windows-commands/certutil) utilise la logique de certificats Windows pour `-verify` et peut inclure un nom de serveur avec `-sslpolicy`. [`openssl x509`](https://docs.openssl.org/master/man1/openssl-x509/) affiche les champs d’un certificat ; [`openssl verify`](https://docs.openssl.org/master/man1/openssl-verify/) construit et valide un chemin par rapport aux Trust Anchors et intermédiaires explicitement indiqués. [`keytool`](https://docs.oracle.com/en/java/javase/25/docs/specs/man/keytool.html) est l’interface officielle appropriée de gestion et d’inspection pour les Key Stores et Truststores JVM ; un test avec OpenSSL ne remplace pas son point de vue.

### Associer un symptôme à une étape de vérification

| Observation | Couche probable | Vérification suivante |
|---|---|---|
| Délai d’attente TCP | DNS, route, pare-feu ou listener | adresse résolue, SYN/SYN-ACK, chemin retour, liaison du listener |
| Bannière en clair au lieu d’une réponse TLS | mauvais modèle de démarrage | port et `-starttls smtp` ou `-starttls ldap` |
| Alerte TLS immédiate après ClientHello | absence de paramètres communs ou politique | versions, groupes, algorithmes de signature, Cipher, SNI |
| Mauvais certificat | SNI ou liaison du listener | SNI envoyé, VHost par défaut, configuration du répartiteur de charge |
| `unable to get local issuer certificate` | intermédiaire ou Trust Anchor manquant | liste du serveur, AIA, Truststore client, construction du chemin |
| `hostname mismatch` | l’ID de référence ne correspond pas au SAN | nom réellement utilisé, type de SAN, alias et limite du wildcard |
| seule l’application Java échoue | politique JSSE ou Truststore propre | JVM, `cacerts`, Truststore de l’application, disabledAlgorithms |
| seuls certains nœuds de cluster fournissent l’ancien certificat | déploiement ou liaison | tester chaque point de terminaison et chaque IP avec le même SNI |
| SMTP revient parfois au clair | politique opportuniste ou downgrade | journaux MTA, offre STARTTLS, état MTA-STS/DANE, TLS-RPT |
| le Handshake mTLS réussit, mais l’accès reste interdit | autorisation après TLS | EKU client, mappage de certificat, transmission par proxy, rôle applicatif |

[`Wireshark`](https://www.wireshark.org/docs/wsug_html/index.html) peut rendre visibles ClientHello, ServerHello, les extensions et les alertes ; sans clés de session, les données applicatives et de nombreux messages de Handshake TLS 1.3 restent chiffrés. Une capture ne montre pas non plus automatiquement quel Truststore utilise l’application ou pourquoi sa politique rejette un chemin.

## Surveillance et critères d’exploitation

Un contrôle de port et une alerte avant la date d’expiration ne suffisent pas. La surveillance devrait vérifier le même chemin qu’un vrai client : sélectionner la cible, établir le Handshake, valider le certificat et les noms, puis atteindre le protocole applicatif. Les signaux suivants sont notamment utiles :

- sélection de la cible et SNI par service, listener et nœud de cluster ;
- protocole négocié, Cipher Suite, groupe et algorithme de signature ;
- empreinte du Leaf, numéro de série, SAN, émetteur ainsi que `notBefore` et `notAfter` ;
- chaîne intermédiaire effectivement envoyée et validation de chemin du point de vue des Truststores pertinents ;
- accessibilité OCSP/CRL et comportement réel de fail-open/fail-closed ;
- latence du Handshake, part de reprise, classes d’alertes et taux d’erreur ;
- offre STARTTLS, mises à niveau réussies et retours au clair par cible SMTP ;
- état MTA-STS, DANE et TLS-RPT là où ces mécanismes de politique sont utilisés ;
- cohérence de tous les terminateurs TLS après renouvellement et test applicatif réussi derrière la couche TLS.

Une surveillance uniquement interne ne voit pas la chaîne publique, le DNS externe ou les terminateurs en périphérie ; un contrôle uniquement externe ne voit ni les Truststores internes ni mTLS. Les plateformes de messagerie et d’annuaire nécessitent donc au minimum un chemin client externe, un chemin applicatif interne et une vérification directe de chaque terminateur.

## Histoire technique

L’histoire commence avec SSL de Netscape dans les années 1990. SSL 2.0 a ensuite été explicitement interdit ([RFC 6176](https://datatracker.ietf.org/doc/html/rfc6176)) ; SSL 3.0 n’est plus documenté que comme protocole historique et n’est plus autorisé depuis RFC 7568. L’IETF a ensuite développé **TLS 1.0** en 1999, **TLS 1.1** en 2006 et **TLS 1.2** en 2008. RFC 8996 a déclaré TLS 1.0 et 1.1 obsolètes en 2021. Pour les administrateurs, cette filiation explique pourquoi d’anciens noms de produits, syntaxes Cipher et options de compatibilité apparaissent encore aujourd’hui dans les configurations.

**TLS 1.3** est paru en 2018 avec RFC 8446. Il a supprimé l’échange de clés RSA statique, les anciens procédés non AEAD et d’autres fonctions héritées, chiffré plus tôt le Handshake et introduit le plan de clés basé sur HKDF ainsi que le modèle de reprise PSK. RFC 9846 a remplacé RFC 8446 en juillet 2026 comme spécification TLS 1.3 consolidée et a intégré les corrections et exigences documentées séparément jusque-là. La valeur de protocole `1.3` est restée identique ; seul le document normatif de référence a changé ([RFC 9846, Abschnitt 1.2](https://datatracker.ietf.org/doc/html/rfc9846#section-1.2)).

Parallèlement, le volet identité et exploitation a évolué : X.509-PKIX a été consolidé dans RFC 5280, SNI et d’autres extensions dans RFC 6066, ALPN dans RFC 7301, l’émission automatisée dans ACME/RFC 8555 et la vérification moderne d’identité de service dans RFC 9525. L’histoire de TLS n’est donc pas seulement une succession de versions, mais le développement conjoint de la cryptographie, des chemins de certificats, de l’intégration applicative et de l’automatisation opérationnelle.

## Sources

- [IETF RFC 8314 – Cleartext Considered Obsolete](https://datatracker.ietf.org/doc/html/rfc8314)
- [RFC 9846 – The Transport Layer Security (TLS) Protocol Version 1.3](https://datatracker.ietf.org/doc/html/rfc9846) – Handshake, Record Layer, plan de clés, reprise et modèle de sécurité.
- [RFC 3207 – SMTP Service Extension for Secure SMTP over TLS](https://datatracker.ietf.org/doc/html/rfc3207) – STARTTLS, changement d’état SMTP et politique opportuniste.
- [RFC 9001 – Using TLS to Secure QUIC](https://datatracker.ietf.org/doc/html/rfc9001) – distinction entre Handshake TLS et protection des paquets QUIC.
- [RFC 5280 – Internet X.509 Public Key Infrastructure Certificate and CRL Profile](https://datatracker.ietf.org/doc/html/rfc5280) – champs de certificat, extensions et validation de chemin.
- [RFC 7468 – Textual Encodings of PKIX, PKCS, and CMS Structures](https://datatracker.ietf.org/doc/html/rfc7468) – enveloppe textuelle PEM.
- [RFC 7292 – PKCS #12](https://datatracker.ietf.org/doc/html/rfc7292) – conteneur pour clés et certificats.
- [RFC 6960 – Online Certificate Status Protocol](https://datatracker.ietf.org/doc/html/rfc6960) – vérification du statut OCSP.
- [RFC 9525 – Service Identity in TLS](https://datatracker.ietf.org/doc/html/rfc9525) – identités de référence, subjectAltName, wildcards et vérification des noms.
- [RFC 6066, Abschnitt 3](https://datatracker.ietf.org/doc/html/rfc6066)
- [RFC 7301 – TLS Application-Layer Protocol Negotiation](https://datatracker.ietf.org/doc/html/rfc7301) – négociation ALPN.
- [IANA – Service Name and Transport Protocol Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) – ports et noms de service enregistrés.
- [RFC 4511, Abschnitt 4.14](https://datatracker.ietf.org/doc/html/rfc4511)
- [RFC 7672 – SMTP Security via DANE TLS](https://datatracker.ietf.org/doc/html/rfc7672) – authentification SMTP basée sur DNSSEC/TLSA.
- [RFC 8461 – SMTP MTA Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc8461) – politique MTA-STS et vérification MX.
- [RFC 8460 – SMTP TLS Reporting](https://datatracker.ietf.org/doc/html/rfc8460) – rapports agrégés d’erreurs TLS.
- [NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final) – gestion des clés et besoins de protection.
- [RFC 8555 – Automatic Certificate Management Environment](https://datatracker.ietf.org/doc/html/rfc8555) – émission et renouvellement automatisés.
- [OpenSSL](https://docs.openssl.org/master/)
- [Quellcode](https://github.com/openssl/openssl)
- [Microsoft Learn – Schannel SSP](https://learn.microsoft.com/windows-server/security/tls/tls-ssl-schannel-ssp-overview) – implémentation TLS Windows et positionnement SSPI.
- [Oracle – JSSE Reference Guide](https://docs.oracle.com/en/java/javase/24/security/java-secure-socket-extension-jsse-reference-guide.html) – API TLS Java, KeyManager et TrustManager.
- [GnuTLS Documentation](https://www.gnutls.org/documentation.html) – bibliothèque TLS native alternative et famille d’outils.
- [BCP 195 – Recommendations for Secure Use of TLS and DTLS](https://www.rfc-editor.org/info/bcp195) – directive de sécurité IETF évolutive pour la configuration TLS.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – accessibilité TCP sous Windows.
- [OpenBSD Manual – nc](https://man.openbsd.org/nc) – test de connexion TCP sous Unix.
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/) – client de diagnostic générique TLS et STARTTLS.
- [Microsoft Learn – certutil](https://learn.microsoft.com/windows-server/administration/windows-commands/certutil) – magasin de certificats Windows et moteur de chaînes.
- [OpenSSL – x509](https://docs.openssl.org/master/man1/openssl-x509/) – champs de certificat et options de sortie.
- [OpenSSL – verify](https://docs.openssl.org/master/man1/openssl-verify/) – construction et validation explicites de chemin.
- [Oracle – keytool](https://docs.oracle.com/en/java/javase/25/docs/specs/man/keytool.html) – gestion des Key Stores et Truststores Java.
- [Wireshark User’s Guide](https://www.wireshark.org/docs/wsug_html/index.html) – analyse de Handshake, Record et clés de session.
- [RFC 6176 – Prohibiting SSL Version 2.0](https://datatracker.ietf.org/doc/html/rfc6176) – abandon formel de SSL 2.0.
