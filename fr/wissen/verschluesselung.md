---
title: "Chiffrement des e-mails : méthodes, clés et modèles d’exploitation"
blatt: "verschluesselung"
description: "Chiffrement des e-mails pour les administrateurs de messagerie : SMTP-TLS, MTA-STS, DANE et REQUIRETLS, S/MIME et OpenPGP, modèles client, passerelle et portail, cycle de vie des clés et certificats, métadonnées, surveillance, récupération et diagnostic."
fakten:
  - label: Question fondamentale
    wert: Quelles données sont protégées entre quelles limites de confiance ?
    href: https://www.nist.gov/publications/trustworthy-email-0
  - label: Transport
    wert: SMTP STARTTLS protège exactement un saut
    href: https://datatracker.ietf.org/doc/html/rfc3207
  - label: Politique de transport
    wert: DANE · MTA-STS · REQUIRETLS
    href: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-177r1.pdf
  - label: S/MIME
    wert: MIME + CMS + X.509/PKIX
    href: https://datatracker.ietf.org/doc/html/rfc8551
  - label: OpenPGP
    wert: Format de paquet, clés, signatures et chiffrement
    href: https://datatracker.ietf.org/doc/html/rfc9580
  - label: Signature
    wert: Authenticité et intégrité ; aucune confidentialité
    href: https://datatracker.ietf.org/doc/html/rfc8551
  - label: Modèle client
    wert: clés privées au point terminal ; le serveur de messagerie stocke le texte chiffré
    href: https://www.nccoe.nist.gov/publication/1800-6/VolC/
  - label: Modèle de passerelle
    wert: texte en clair dans la zone contrôlée et sur la passerelle
    href: https://www.seppmail.com/products/
  - label: Modèle de portail
    wert: notification par e-mail ; contenu dans un stockage web avec une identité propre
    href: https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm
  - label: Gestion des clés
    wert: inventaire · protection · distribution · rotation · révocation · récupération
    href: https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final
  - label: Métadonnées visibles
    wert: l’enveloppe SMTP et les en-têtes externes restent visibles selon le modèle
    href: https://datatracker.ietf.org/doc/html/rfc8551
  - label: Règle de récupération
    wert: sauvegarder les clés de chiffrement si nécessaire ; ne pas reproduire les clés de signature
    href: https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-57pt1r5.pdf
werbung:
  - tools
  - newsletter
ctaThemen:
  - e-mail-verschluesselung
  - seppmail
translationSourceHash: 915d86b5c19b97fc5564faf33053144b692f76fd3d6c732fd955f466d25c4983
translationModel: gpt-5.6-terra
translatedAt: 2026-10-15T09:16:35.442Z
translationReview: required
---

# Chiffrement des e-mails : méthodes, clés et modèles d’exploitation

La question « Cet e-mail est-il chiffré ? » est techniquement incomplète. Un message peut être lisible sur le client, stocké comme texte chiffré sur le serveur de messagerie, déchiffré sur une passerelle, protégé entre deux MTA par [TLS](/kb/tls) et à nouveau stocké sans chiffrement chez le destinataire. Une réponse fiable doit donc indiquer **l’objet protégé, les points terminaux, les détenteurs des clés et les emplacements où le texte est en clair**. Pour cette raison précise, NIST traite les mécanismes de transport, de domaine et de contenu comme des contrôles distincts mais interdépendants ([NIST SP 800-177 Rev. 1](https://www.nist.gov/publications/trustworthy-email-0)).

Le **chiffrement de transport** protège une connexion. SMTP STARTTLS est négocié à nouveau pour chaque saut et s’arrête au MTA concerné. Le **chiffrement de contenu** protège une entité MIME avec S/MIME ou OpenPGP et peut survivre à plusieurs sauts de transport et de stockage. Son effet réellement de bout en bout ne dépend pas uniquement du format : si une passerelle d’entreprise déchiffre par délégation, les points terminaux cryptographiques se situent aux passerelles et non sur les clients utilisateurs ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207), [RFC 8551](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580)).

Une **signature numérique** constitue encore une autre propriété de sécurité. Elle lie des données à une clé de signature privée et permet de vérifier l’intégrité et l’identité ; elle ne rend pas le contenu secret. Inversement, un message déchiffré avec succès ne dit rien sur la personne qui l’a créé. En S/MIME, les éléments CMS `SignedData` et `EnvelopedData` ou `AuthEnvelopedData` sont distingués à cette fin. OpenPGP sépare également les paquets de signature, de clé de session et de données chiffrées ([RFC 5652](https://datatracker.ietf.org/doc/html/rfc5652), [RFC 8551, S/MIME Content Types](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 9580, Packet Syntax](https://datatracker.ietf.org/doc/html/rfc9580)).

L’explication suit un message et examine à chaque saut où le texte en clair est présent et qui contrôle les clés. Le chiffrement de transport, les méthodes de bout en bout, les passerelles, les portails, les métadonnées, le diagnostic et la récupération sont comparés sur cette base.

Le chiffrement des e-mails peut protéger le saut de transport, le contenu du message ou un accès par portail. Ces modèles ont des détenteurs de clés et des emplacements de texte en clair différents. Cet article suit donc d’abord le chemin de protection, puis les produits et méthodes individuels.

## Approche architecturale : modèles de protection et limites du texte en clair

Le chiffrement des e-mails se distingue surtout par les deux points qui chiffrent et déchiffrent. Le tableau commence donc par le cheminement normal d’un message et montre ensuite où le texte en clair reste techniquement visible.

| Modèle | Points terminaux cryptographiques | Le texte en clair est présent au minimum | État d’exploitation principal |
|---|---|---|---|
| SMTP-TLS opportuniste | deux MTA adjacents | sur chaque MTA impliqué | certificats, capacités TLS, file d’attente et nouvelle tentative |
| TLS de transport imposé | deux MTA plus une politique publiée ou locale | sur chaque MTA impliqué | MTA-STS/DANE/REQUIRETLS, DNS/HTTPS et rapports |
| S/MIME/OpenPGP client à client | MUA ou module cryptographique des utilisateurs | avant le chiffrement et après le déchiffrement au point terminal | clés privées, certificats/clés, compatibilité client |
| passerelle à passerelle | passerelles de messagerie des organisations | système de messagerie interne, traitement de la passerelle, zone de destination | politique, annuaire, magasin de clés de passerelle, file d’attente et mappage |
| portail web | expéditeur/passerelle et application web du portail | passerelle/portail, navigateur après connexion | objet message, compte destinataire, OTP, lien, conservation |

Aucun de ces modèles n’est systématiquement « plus fort ». Le modèle client à client réduit les emplacements de texte en clair dans l’infrastructure, mais complique l’analyse des malwares, la journalisation, l’eDiscovery et la récupération. L’exemple pratique du NCCoE fait donc exécuter la vérification S/MIME sur le client utilisateur avec ses ancres de confiance, après l’achèvement des contrôles de transport et de domaine au niveau du MTA ([NCCoE: DNS-Based Email Security](https://www.nccoe.nist.gov/publication/1800-6/VolC/)). Les passerelles centralisent la politique et les clés, mais deviennent elles-mêmes des emplacements de texte en clair et de clés hautement privilégiés. Les portails atteignent les destinataires sans cryptographie de messagerie, mais créent un mécanisme supplémentaire pour le web, l’identité et le stockage. Le choix découle de la classification des données, du modèle de menace, de la conservation, des exigences légales et des capacités des deux parties communicantes — et non du nom du produit.

<iframe class="kb-infographic kb-infographic--compact" style="aspect-ratio: 1400 / 1076; max-width: 720px; margin-inline: auto" src="/images/kb-interaktiv-verschluesselung.svg?v=20260813" title="Interaktive Infografik: E-Mail-Verschlüsselung mit Transport-, Client-, Gateway- und Portalmodell, Klartextgrenzen, Schlüsselzustand, Metadaten und Admin-Kontrollpunkten" loading="lazy">
  <a href="/images/kb-interaktiv-verschluesselung.svg?v=20260813">Ouvrir directement le graphique interactif</a>.
</iframe>

## SMTP-TLS : protection par saut de transport

[SMTP](/kb/smtp) a été conçu à l’origine sans confidentialité obligatoire. STARTTLS étend une session SMTP existante : après une négociation TLS réussie, les capacités sont à nouveau négociées par EHLO. En mode opportuniste, le MTA expéditeur peut revenir au texte en clair en l’absence de STARTTLS ou en cas d’erreur de certificat. Cela améliore la protection contre l’écoute passive, mais n’offre, sans politique supplémentaire, aucune garantie fiable contre une rétrogradation ou un attaquant actif ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207)).

**MTA-STS** publie d’abord une indication DNS TXT et récupère la politique effective via HTTPS. En mode `enforce`, les noms MX indiqués doivent être validés par PKIX ; les erreurs entraînent un retard de livraison plutôt qu’un texte en clair. **DANE pour SMTP** associe des informations de certificat ou de clé à des enregistrements TLSA validés par DNSSEC. **REQUIRETLS** est un marquage SMTP et de message par lequel un expéditeur exige TLS pour la transmission ultérieure. Ces mécanismes ont des dépendances et des portées différentes et ne doivent pas être regroupés sous une seule case à cocher « TLS requis » ([RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461), [RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672), [RFC 8689](https://datatracker.ietf.org/doc/html/rfc8689)).

SMTP TLS Reporting, **TLS-RPT**, fournit des rapports agrégés sur les vérifications réussies et échouées des politiques TLS. Il ne prouve aucune livraison individuelle et ne contient aucun message ; il aide à détecter les erreurs de certificat, de DNS et de politique avant que les files d’attente ne s’allongent durablement. NIST recommande aux domaines destinataires de signaler DANE, MTA-STS ou les deux, et décrit les différentes dépendances à DNSSEC, HTTPS et PKIX ([RFC 8460](https://datatracker.ietf.org/doc/html/rfc8460), [NIST SP 800-177 Rev. 1, section 5.2](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-177r1.pdf)).

SMTP-TLS s’arrête au serveur de messagerie suivant. Si le contenu du message doit rester protégé sur plusieurs sauts, des méthodes liées au message telles que S/MIME ou OpenPGP sont utilisées.

## S/MIME : CMS, X.509 et MIME

S/MIME n’est pas une bibliothèque de chiffrement unique, mais l’association de plusieurs normes. MIME structure les parties d’un message ; CMS définit des conteneurs signés et chiffrés ; X.509/PKIX lie les clés publiques à des identités et à des ancres de confiance par des certificats. Pour chaque destinataire, l’expéditeur chiffre la clé de contenu symétrique avec sa clé publique ou une méthode de gestion de clés convenue. La lecture exige la clé privée correspondante ([RFC 8551](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 5652](https://datatracker.ietf.org/doc/html/rfc5652), [RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280)).

L’administrateur doit distinguer deux rôles de clés. Une **clé de signature** ne doit être disponible que pour le signataire ; une copie centrale permettrait techniquement à des tiers de générer de nouvelles signatures sous son identité. Une **clé de déchiffrement** peut être sécurisée ou placée sous séquestre selon la politique de l’organisation, afin que les anciens messages professionnels restent lisibles après la perte d’un appareil, un changement de rôle ou le renouvellement d’un certificat. RFC 8551 indique explicitement qu’une personne peut posséder plusieurs paires de clés actives à des fins différentes ; NIST distingue la protection et la récupération selon le type de clé ([RFC 8551, Multiple Key Pairs](https://datatracker.ietf.org/doc/html/rfc8551), [NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)).

Avant l’envoi, l’expéditeur a besoin d’un certificat de chiffrement approprié et valide pour **chaque** destinataire. La distribution peut passer par un annuaire, des messages signés, une plateforme de certificats ou une procédure manuelle. Les groupes de distribution sont particulièrement exigeants, car l’expansion, la visibilité et la disponibilité des certificats de tous les membres doivent être cohérentes. Microsoft documente pour Exchange les attributs `userSMIMECertificate` et `userCertificate` ainsi que la collection de certificats pour la vérification de confiance ([Microsoft: S/MIME in Exchange Server](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/smime/smime), [Microsoft: S/MIME in Exchange Online](https://learn.microsoft.com/en-us/exchange/security-and-compliance/smime-exo/smime-exo)).

## OpenPGP : le format n’est pas un modèle de confiance

RFC 9580 définit les paquets, clés, empreintes, signatures, clés de session et données chiffrées OpenPGP. Selon la norme, un trousseau de clés n’est qu’une collection de clés ; la manière dont une organisation trouve les clés, vérifie les propriétaires et décide de la confiance est extérieure à ce format de données. Le **Web of Trust**, historiquement répandu, est donc une forme d’exploitation possible, mais n’est pas une définition synonyme d’OpenPGP. D’autres environnements utilisent des empreintes vérifiées manuellement, des clés signées par l’organisation, WKD, des services d’annuaire ou des passerelles gérées de manière centralisée ([RFC 9580, Keyrings et Fingerprints](https://datatracker.ietf.org/doc/html/rfc9580)).

Pour les e-mails, PGP/MIME encapsule les signatures et le texte chiffré dans des structures MIME. Inline-PGP, en revanche, ne protège que certaines parties de texte et entraîne facilement des problèmes d’interopérabilité avec HTML, les pièces jointes, les jeux de caractères et la canonisation des lignes. RFC 3156 normalise `multipart/encrypted` et `multipart/signed` pour OpenPGP. Il convient de tester avec les véritables homologues si un client prend entièrement en charge les clés RFC 9580 modernes, AEAD, les courbes utilisées et PGP/MIME ([RFC 3156](https://datatracker.ietf.org/doc/html/rfc3156), [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580)).

Un Key ID court ne constitue pas une preuve d’identité suffisante. RFC 9580 avertit que les Key IDs ne doivent pas être considérés comme uniques ; l’empreinte complète est utilisée pour les vérifications. Les certificats de révocation, l’expiration, les sous-clés, les nouvelles empreintes et la distribution sécurisée d’une rotation font donc partie du même cycle de vie que la première émission de clé.

S/MIME et OpenPGP peuvent être appliqués sur le terminal ou de manière centralisée sur la passerelle. La passerelle simplifie la politique et l’exploitation des clés, mais accède en contrepartie au texte en clair.

## Modèle de passerelle : politique centrale et pouvoir central sur le texte en clair

Une passerelle de messagerie accepte les messages internes normaux, classe l’expéditeur, les destinataires et les données, recherche les clés S/MIME ou OpenPGP appropriées et choisit le chiffrement, la signature, TLS ou la livraison par portail. À l’entrée, elle déchiffre éventuellement avant la remise au système de messagerie interne. Des produits tels que [SEPPmail](/kb/seppmail), [Totemomail beziehungsweise Kiteworks EPG](/kb/totemomail) et [HIN](/kb/hin) utilisent des plateformes différentes, mais reposent sur le même transfert de confiance fondamental : l’organisation délègue la cryptographie du terminal utilisateur à un composant d’infrastructure ([SEPPmail: Produkte](https://www.seppmail.com/products/), [HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)).

Cela ne résout pas à lui seul la distribution des clés. La passerelle a toujours besoin d’une association établie entre l’identité du destinataire et une clé publique, d’une priorité entre plusieurs méthodes, de règles pour les clés expirées ou révoquées et d’un mécanisme de repli défini. L’échange automatique de clés entre passerelles peut simplifier ces étapes, mais constitue lui-même un protocole de confiance impliquant l’identité des homologues, le cache, l’expiration et les états d’erreur.

L’ordre de la politique détermine quels contrôles voient le texte en clair. Si une passerelle entrante déchiffre, l’anti-malware, la DLP et l’archivage peuvent ensuite fonctionner. Si une passerelle sortante chiffre trop tôt, les scanners placés en aval sont aveugles. Si une passerelle modifie une entité MIME déjà signée, la signature du contenu ou [DKIM](/kb/mail-auth) peut devenir invalide. RFC 6376 fait des en-têtes canonisés et du corps l’objet de la signature ; signature, chiffrement, avertissements, réécriture et DKIM doivent donc être ordonnés dans un pipeline documenté ([RFC 6376](https://datatracker.ietf.org/doc/html/rfc6376)).

Si aucune clé appropriée n’est disponible chez le destinataire externe, un portail peut stocker le message et contrôler l’accès via une connexion web. Le problème passe alors de la cryptographie de messagerie à l’identité web et à l’exploitation de la plateforme.

## Livraison par portail : application web au lieu de clé destinataire

Lors d’une livraison par portail, le destinataire reçoit un e-mail de notification normal et accède au contenu protégé via une application web. Le point terminal cryptographique est le portail, et non le client de messagerie du destinataire. La tâche technique se déplace ainsi de la distribution de clés publiques vers l’identité web, la gestion de session, le stockage des messages et pièces jointes, le lien de notification, la conservation et le canal de réponse sécurisé.

HIN documente, à titre d’exemple concret, l’inscription, le numéro de téléphone mobile et le code SMS, ainsi que l’option de mémoriser un appareil pour les accès ultérieurs. Cet exemple atteste le modèle d’exploitation, mais non une implémentation de portail universellement valable. D’autres produits utilisent un mot de passe, un code à usage unique sur un canal séparé, une identité fédérée ou une connexion client existante ([HIN: Mail an Nichtmitglieder](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)).

Pour les administrateurs, un portail exige au minimum les preuves suivantes :

- Quelles données figurent dans la notification non protégée et dans le jeton URL ?
- Comment l’inscription initiale, la liaison d’identité, les tentatives échouées, la mémorisation d’appareil et la récupération de compte sont-elles sécurisées ?
- Où sont stockés les messages, les pièces jointes, les clés et les données d’audit, et quand sont-ils supprimés ?
- Un message reste-t-il récupérable après expiration, suppression d’utilisateur ou changement de tenant ?
- Comment la réponse sécurisée, le transfert, le téléchargement et l’accusé de lecture sont-ils journalisés ?
- Que se passe-t-il si le portail est inaccessible, si le service SMS est indisponible ou si le compte du destinataire est compromis ?

Un statut HTTP `200` prouve seulement qu’une interface web répond. Un parcours synthétique du portail doit couvrir la notification, l’inscription ou la connexion, le second facteur, l’ouverture, la pièce jointe, la réponse et l’enregistrement d’audit attendu.

## Métadonnées, en-têtes et archivage

Le chiffrement de contenu ne protège pas automatiquement l’enveloppe SMTP. Les MTA de livraison ont besoin au minimum des adresses de destinataires ; les systèmes génèrent des lignes `Received` et traitent les informations de taille et de routage. Même avec S/MIME, les en-têtes externes restent normalement visibles. RFC 8551 autorise l’encapsulation d’un message complet comme `message/rfc822` dans l’entité MIME protégée, mais exige une représentation distincte des en-têtes internes et externes. La protection effective de l’objet, de l’affichage de l’expéditeur et de la liste des destinataires est une fonction du client et de la passerelle, et non une conséquence certaine de l’icône de cadenas ([RFC 5322](https://datatracker.ietf.org/doc/html/rfc5322), [RFC 8551, Header Protection](https://datatracker.ietf.org/doc/html/rfc8551)).

L’archivage doit définir **quelle représentation** est conservée : texte en clair d’origine, entité originale signée, original chiffré, copie de travail déchiffrée ou objet de portail. Seul le texte en clair facilite l’eDiscovery, mais perd la preuve cryptographique de l’original. Seul le texte chiffré peut devenir illisible après la perte d’une clé. La validation d’une signature à une date ultérieure nécessite les octets originaux, la chaîne de certificats, les informations de confiance et de révocation de l’époque ainsi qu’une référence temporelle ; un certificat aujourd’hui expiré ne rend pas automatiquement sans valeur une signature alors valide.

## Cycle de vie des clés et certificats

Un service cryptographique en production requiert un inventaire qui va au-delà des dates d’expiration. NIST SP 800-57 considère la génération, l’activation, la protection, l’inventaire, la distribution, l’utilisation, la rotation, la révocation, l’archivage, la récupération et la destruction comme des tâches de gestion interdépendantes ([NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)).

| Phase | Question d’administration | Preuve |
|---|---|---|
| Identité | Qui ou quel rôle est lié à la clé ? | processus d’enrôlement vérifié, Subject/SAN/User ID, approbation |
| Génération | Où la clé privée est-elle créée ? | client, HSM, passerelle ou CA ; algorithme et exportabilité |
| Distribution | Comment l’expéditeur obtient-il la bonne clé publique ? | annuaire, e-mail signé, WKD, service de certificats, vérification d’empreinte |
| Utilisation | La clé peut-elle signer, déchiffrer ou les deux ? | Key Usage/EKU, indicateurs OpenPGP, politique et client réel |
| Rotation | Comment l’ancienne et la nouvelle clé se chevauchent-elles ? | publication avant le basculement, test sur toutes les plateformes cibles |
| Révocation | Comment les vérificateurs et expéditeurs apprennent-ils la compromission ? | CRL/OCSP ou révocation OpenPGP, politique de cache et d’erreur |
| Récupération | Qui peut déchiffrer les contenus historiques ? | procédure de séquestre, validation à quatre yeux, audit, test de restauration |
| Destruction | Quand la capacité de déchiffrement prend-elle fin délibérément ? | preuve de conservation et de suppression des clés et sauvegardes |

La validation de chemin S/MIME comprend les ancres de confiance, les certificats intermédiaires, la période de validité, le nom ou l’identité e-mail, Key Usage/Extended Key Usage, l’algorithme et le statut de révocation. OCSP et les CRL dépendent à leur tour du réseau et du cache. Avec un soft fail, une vérification reste possible malgré une information de révocation inaccessible ; avec un hard fail, un problème d’infrastructure peut bloquer tous les messages signés ou chiffrés. Cette décision doit figurer dans la politique et dans la surveillance ([RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280), [RFC 6960](https://datatracker.ietf.org/doc/html/rfc6960)).

Le diagnostic commence par le modèle de protection choisi : saut TLS, signature et chiffrement du contenu ou accès par portail. Ce n’est qu’ensuite que le certificat, la clé, la politique et le chemin de livraison sont vérifiés.

## Surveillance et analyse des incidents

La surveillance doit être séparée selon le modèle de protection. Pour TLS de transport, il faut suivre les protocoles proposés et négociés, les erreurs de politique, la validation des certificats, TLS-RPT et l’âge de la file d’attente. Pour S/MIME/OpenPGP, il faut suivre les clés de destinataire manquantes, les chemins ou signatures invalides, les mécanismes de repli, la rotation et les erreurs de déchiffrement. Pour les passerelles s’ajoutent la décision de politique, le Message-ID, l’empreinte de clé, le traitement du texte en clair et le saut suivant. Pour les portails, l’inscription, MFA/OTP, le stockage, l’expiration du lien, la connexion, l’ouverture du message et le chemin de réponse sont des signaux propres.

Un protocole d’incident pertinent répond aux questions suivantes :

1. Quel message et quel chemin de destinataire sont concernés ?
2. Quel modèle de protection et quelle politique ont été sélectionnés ?
3. Quelle clé publique ou quel certificat a été utilisé, avec quelle empreinte et quel statut ?
4. Où le message était-il lisible, chiffré, stocké ou en file d’attente pour la dernière fois ?
5. Le résultat était-il une livraison, un dépôt sur portail, une quarantaine, un repli, une nouvelle tentative ou une erreur permanente ?
6. Le résultat peut-il être reproduit avec une identité de test contrôlée, sans divulguer de données de production ?

## Sauvegarde et récupération

Les données chiffrées et les clés doivent être planifiées ensemble, mais avec des mesures de protection distinctes. Une sauvegarde de messages chiffrés sans les clés privées de déchiffrement nécessaires à long terme peut être correctement sécurisée tout en restant irrécupérable sur le plan métier. Inversement, une sauvegarde de séquestre de clés insuffisamment protégée transforme de nombreux secrets individuels en un point d’attaque central. NIST traite l’utilisation autorisée de clés privées récupérées en fonction du type de clé et de la période cryptographique ([NIST SP 800-57 Part 1 Rev. 5, PDF](https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-57pt1r5.pdf)).

L’étendue minimale de récupération comprend :

- les messages originaux chiffrés et, le cas échéant, les entités MIME signées à valeur probante ;
- les clés privées de **déchiffrement** avec métadonnées, phrases secrètes, rôles et procédures d’autorisation ;
- les clés publiques, chaînes de certificats, ancres de confiance, données de révocation et de validation historiques ;
- la politique de passerelle et de portail, le routage, l’association de clés, les données utilisateur et les objets de message ;
- les sources d’audit et de temps afin que les décisions antérieures restent traçables ;
- un test de restauration avec anciennes et nouvelles clés, plusieurs destinataires, pièces jointes et vérification de signature.

Les clés privées de signature ne devraient pas être copiées comme mécanisme général de récupération. Si une organisation doit signer de manière automatisée ou par délégation, une identité organisationnelle explicite dotée de sa propre politique est préférable à la copie non remarquée d’une clé personnelle. La capacité de récupération et la non-répudiation entrent sinon en conflit.

## Outils de diagnostic

La recherche d’erreur suit le modèle de protection choisi. Pour le chiffrement de transport, DNS et TLS sont vérifiés ; pour les méthodes de bout en bout, suivent les certificats, les clés et le traitement par le client ; pour les passerelles, leur état de politique et de file d’attente s’ajoute.

### MTA-STS, TLS-RPT et DANE dans le DNS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für E-Mail-Transportpolicy-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type TXT _mta-sts.example.ch
Resolve-DnsName -Type TXT _smtp._tls.example.ch
Resolve-DnsName -Type TLSA _25._tcp.mx.example.ch
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig +short TXT _mta-sts.example.ch
dig +short TXT _smtp._tls.example.ch
dig +dnssec TLSA _25._tcp.mx.example.ch
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) et [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) affichent les politiques de transport publiées. Pour DANE, le statut de validation DNSSEC doit également être démontré ; un texte TLSA simplement visible n’est pas encore un enregistrement authentifié.

### STARTTLS et chaîne de certificats

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection mx.example.ch -Port 25 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://mx.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz mx.example.ch 25
openssl s_client -starttls smtp -connect mx.example.ch:25 \
  -servername mx.example.ch -showcerts
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) et [`nc`](https://man.openbsd.org/nc) vérifient le chemin TCP. [`curl`](https://curl.se/docs/manpage.html) et [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) affichent STARTTLS, la négociation et la chaîne. La question de savoir si un MTA de production revient en arrière ou met en file d’attente en cas d’erreurs ne ressort que de sa politique et de ses journaux.

### Certificat S/MIME et usage prévu

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für S-MIME-Zertifikatsprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
certutil.exe -dump .\recipient-smime.cer
certutil.exe -verify -urlfetch .\recipient-smime.cer
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
openssl x509 -in recipient-smime.pem -noout \
  -subject -issuer -serial -fingerprint -dates -purpose -ext keyUsage -ext extendedKeyUsage
openssl verify -show_chain -CAfile trust-chain.pem recipient-smime.pem
```

  </div>
</div>

[`certutil`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/certutil) et [`openssl x509`](https://docs.openssl.org/master/man1/openssl-x509/) affichent l’identité, l’empreinte, la validité et l’usage prévu ; [`openssl verify`](https://docs.openssl.org/master/man1/openssl-verify/) vérifie un magasin de confiance indiqué. Le test doit être répété avec les mêmes ancres de confiance et règles de révocation que le client ou la passerelle de production.

### Empreinte OpenPGP et structure de clé

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für OpenPGP-Schlüsselprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
gpg.exe --show-keys --with-fingerprint --with-subkey-fingerprint .\recipient.asc
gpg.exe --list-packets .\message.pgp
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
gpg --show-keys --with-fingerprint --with-subkey-fingerprint recipient.asc
gpg --list-packets message.pgp
```

  </div>
</div>

[`gpg`](https://www.gnupg.org/documentation/manuals/gnupg24/gpg.1.html) affiche les empreintes complètes, les sous-clés, les capacités et la structure des paquets. `--list-packets` analyse uniquement le format ; il ne confirme ni le propriétaire ni la confiance. Les empreintes doivent être vérifiées par un second canal authentifié ou une procédure organisationnelle définie.

### Chemin de portail et capture de paquets

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Portal- und Paketpfaddiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Invoke-WebRequest -Method Head -Uri https://securemail.example.ch/
pktmon filter remove
pktmon filter add SecureMail -p 25
pktmon start --capture --pkt-size 0 --file-name securemail.etl
pktmon stop
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
curl --head --verbose https://securemail.example.ch/
tcpdump -ni any -s 0 -w securemail.pcap 'tcp port 25 or tcp port 443'
```

  </div>
</div>

[`Invoke-WebRequest`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest) et [`curl`](https://curl.se/docs/manpage.html) vérifient uniquement l’accessibilité HTTP/TLS. [`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) et [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) capturent uniquement au point de mesure sélectionné. Les données de paquets peuvent contenir du texte SMTP en clair, des adresses et des métadonnées de portail, et doivent être protégées en conséquence.

## Histoire technique

PEM, Privacy Enhanced Mail, a normalisé au début des années 1990 les e-mails Internet protégés par cryptographie, mais n’a pas atteint une large interopérabilité. S/MIME a ensuite associé la structure CMS issue de PKCS à MIME et X.509. Les normes S/MIME ont évolué sur plusieurs générations jusqu’à la version 4.0 publiée en 2019 ; CMS lui-même est normalisé dans RFC 5652 ([RFC 1421](https://datatracker.ietf.org/doc/html/rfc1421), [RFC 8551](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 5652](https://datatracker.ietf.org/doc/html/rfc5652)).

OpenPGP est issu du format de paquets PGP et a d’abord été normalisé dans RFC 2440, puis dans RFC 4880. RFC 9580 a remplacé RFC 4880 en 2024 et a notamment modernisé les versions de clés, les empreintes et les méthodes de chiffrement authentifié. PGP/MIME selon RFC 3156 demeure l’encapsulation d’e-mail ; l’évolution du format n’élimine toutefois pas la nécessité organisationnelle de régler l’identité des clés et la récupération ([RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580), [RFC 3156](https://datatracker.ietf.org/doc/html/rfc3156)).

SMTP STARTTLS a été normalisé en 1999 dans RFC 2487 et mis à jour en 2002 par RFC 3207. DANE pour SMTP, MTA-STS, TLS-RPT et REQUIRETLS ont ensuite complété les couches de politique et de rapport contre la rétrogradation et les erreurs de configuration. Parallèlement, les entreprises ont établi des passerelles et des portails web, car les destinataires externes ne disposent pas de manière fiable de clés S/MIME ou OpenPGP compatibles. Le chiffrement des e-mails est ainsi devenu moins une fonctionnalité client isolée qu’un système distribué composé de SMTP, DNS, HTTPS, PKI, annuaire, politique, identité web et récupération ([RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672), [RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461), [RFC 8460](https://datatracker.ietf.org/doc/html/rfc8460), [RFC 8689](https://datatracker.ietf.org/doc/html/rfc8689)).

## Sources

- [NIST – SP 800-177 Rev. 1, Trustworthy Email](https://www.nist.gov/publications/trustworthy-email-0)
- [IETF RFC 3207 – SMTP Service Extension for Secure SMTP over TLS](https://datatracker.ietf.org/doc/html/rfc3207)
- [IETF RFC 8551 – S/MIME 4.0 Message Specification](https://datatracker.ietf.org/doc/html/rfc8551)
- [IETF RFC 9580 – OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580)
- [IETF RFC 5652 – Cryptographic Message Syntax](https://datatracker.ietf.org/doc/html/rfc5652)
- [NCCoE – DNS-Based Email Security Practice Guide](https://www.nccoe.nist.gov/publication/1800-6/VolC/)
- [IETF RFC 8461 – SMTP MTA Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc8461)
- [IETF RFC 7672 – SMTP Security via DANE](https://datatracker.ietf.org/doc/html/rfc7672)
- [IETF RFC 8689 – SMTP REQUIRETLS](https://datatracker.ietf.org/doc/html/rfc8689)
- [IETF RFC 8460 – SMTP TLS Reporting](https://datatracker.ietf.org/doc/html/rfc8460)
- [NIST – SP 800-177 Rev. 1, PDF](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-177r1.pdf)
- [IETF RFC 5280 – Internet X.509 PKI Certificate and CRL Profile](https://datatracker.ietf.org/doc/html/rfc5280)
- [NIST – SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)
- [Microsoft Learn – S/MIME in Exchange Server](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/smime/smime)
- [Microsoft Learn – S/MIME in Exchange Online](https://learn.microsoft.com/en-us/exchange/security-and-compliance/smime-exo/smime-exo)
- [IETF RFC 3156 – MIME Security with OpenPGP](https://datatracker.ietf.org/doc/html/rfc3156)
- [SEPPmail – Produits](https://www.seppmail.com/products/)
- [HIN – HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)
- [IETF RFC 6376 – DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [HIN Support – Mail aux non-membres](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)
- [IETF RFC 5322 – Internet Message Format](https://datatracker.ietf.org/doc/html/rfc5322)
- [IETF RFC 6960 – Online Certificate Status Protocol](https://datatracker.ietf.org/doc/html/rfc6960)
- [NIST – SP 800-57 Part 1 Rev. 5, PDF](https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-57pt1r5.pdf)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [ISC BIND 9 – page de manuel dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – page de manuel nc](https://man.openbsd.org/nc)
- [curl – page de manuel de ligne de commande](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Microsoft Learn – certutil](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/certutil)
- [OpenSSL – x509](https://docs.openssl.org/master/man1/openssl-x509/)
- [OpenSSL – verify](https://docs.openssl.org/master/man1/openssl-verify/)
- [GnuPG – manuel gpg](https://www.gnupg.org/documentation/manuals/gnupg24/gpg.1.html)
- [Microsoft Learn – Invoke-WebRequest](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest)
- [Microsoft Learn – pktmon](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [tcpdump – page de manuel](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [IETF RFC 1421 – Privacy Enhancement for Internet Electronic Mail](https://datatracker.ietf.org/doc/html/rfc1421)
