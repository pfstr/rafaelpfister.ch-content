---
title: "SMTP : transport, file d’attente et responsabilité de livraison"
blatt: "smtp"
description: "Simple Mail Transfer Protocol vu par l’administrateur : rôles de MUA, MSA, MTA et MDA, enveloppe et contenu du message, automate d’état SMTP, limite d’acceptation, routage MX, file d’attente et nouvelles tentatives, codes de réponse, extensions ESMTP, soumission, politique TLS et diagnostic par couche."
fakten:
  - label: Nom complet
    wert: Simple Mail Transfer Protocol
    href: https://datatracker.ietf.org/doc/html/rfc5321
  - label: Modèle de base
    wert: Command/Reply et Store-and-Forward
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-2.3.1
  - label: Objet de transport
    wert: Enveloppe plus contenu du message
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-2.3.1
  - label: Transport
    wert: flux de données fiable et ordonné ; sur Internet, TCP
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-1.1
  - label: Serveur à serveur
    wert: SMTP sur TCP 25
    href: https://datatracker.ietf.org/doc/html/rfc6409#section-1
  - label: Soumission
    wert: 587 avec STARTTLS · 465 avec TLS implicite
    href: https://datatracker.ietf.org/doc/html/rfc8314#section-3
  - label: Rôles principaux
    wert: MUA · MSA · MTA · MDA ou LMTP
    href: https://datatracker.ietf.org/doc/html/rfc5598#section-4
  - label: Fonctionnalités
    wert: EHLO négocie les extensions ESMTP à chaque session
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-2.2
  - label: Limite d’acceptation
    wert: 250 après End-of-Data transfère la responsabilité
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4
  - label: Classes d’erreurs
    wert: 4xx temporaire · 5xx permanent · x.y.z comme détail
    href: https://datatracker.ietf.org/doc/html/rfc3463#section-2
  - label: Dépendances
    wert: DNS · TCP · TLS · SASL · MIME · file d’attente persistante
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-2.3
  - label: Historique des normes
    wert: RFC 821 → RFC 2821 → RFC 5321
    href: https://datatracker.ietf.org/doc/html/rfc5321#appendix-C
werbung:
  - tools
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: bd51241025647a47c1c52f82747a0f7eee7d7b8e42d57f52057e7123d6bc3825
translationModel: gpt-5.6-terra
translatedAt: 2026-10-13T11:12:48.400Z
translationReview: required
---

# SMTP : transport, file d’attente et responsabilité de livraison

Le **Simple Mail Transfer Protocol** (SMTP) achemine un message d’un processus de messagerie au suivant. Le client se connecte à un serveur, interroge ses capacités, indique l’expéditeur et les destinataires de l’enveloppe, puis transfère seulement ensuite le contenu du message. Le serveur répond à chaque étape par un code numérique. De cette séquence apparemment simple naissent le routage, la mise en file d’attente, les nouvelles tentatives et la question décisive de savoir quel serveur assume actuellement la responsabilité du message ([RFC 5321, sections 1 et 2](https://datatracker.ietf.org/doc/html/rfc5321#section-1)).

Pour les administrateurs de messagerie, la propriété décisive réside dans le **transfert de responsabilité**. L’établissement réussi d’une connexion TCP signifie seulement qu’un écouteur est accessible. Un `RCPT TO` accepté réserve un destinataire au sein de la transaction en cours. Seule une conclusion positive après le contenu complet du message transfère au système SMTP destinataire la responsabilité de livrer le message ou de signaler ultérieurement une erreur à l’expéditeur de l’enveloppe ([RFC 5321, sections 4.1.1.4 et 6.1](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4)). L’ID de file d’attente, le code de réponse, le MTA distant, l’état du destinataire et la trace `Received` ne sont donc pas de simples détails de journalisation, mais la chaîne de preuve d’une livraison.

L’explication suit un message depuis le client émetteur, via la session SMTP, l’enveloppe et la file d’attente, jusqu’au serveur de messagerie suivant ou à la boîte aux lettres. DNS, les nouvelles tentatives, les codes d’état, la sécurité et le diagnostic sont approfondis là où ils influencent le cheminement.

## Rôles et responsabilités

SMTP ne connaît initialement que les rôles de **client** et de **serveur**. Dans une chaîne de livraison réelle, ces rôles changent à chaque saut : un agent de transfert de courrier reçoit un message en tant que serveur, le stocke dans sa file d’attente et devient lui-même client lors de la tentative de livraison suivante. Seul ce changement de rôle permet de lire correctement les journaux et les responsabilités ([RFC 5321, section 2.3.3](https://datatracker.ietf.org/doc/html/rfc5321#section-2.3.3), [RFC 5598, section 4](https://datatracker.ietf.org/doc/html/rfc5598#section-4), [RFC 6409, section 2.1](https://datatracker.ietf.org/doc/html/rfc6409#section-2.1)).

| Rôle | Tâche | Limite typique de confiance et d’état |
|---|---|---|
| **MUA** – Message User Agent | crée un message et affiche les messages reçus | identité utilisateur, brouillons locaux, carnet d’adresses et présentation ; aucun service de relais public |
| **MSA** – Message Submission Agent | accepte les nouveaux messages d’utilisateurs ou d’applications autorisés | identité authentifiée, autorisation d’expéditeur, politique de soumission et retour d’erreur immédiat |
| **MTA** – Message Transfer Agent | route et transfère les messages acceptés vers d’autres MTA | file d’attente persistante, planificateur de nouvelles tentatives, choix de destination DNS, politique TLS et responsabilité de livraison |
| **Passerelle ou relais** | effectue une conversion entre zones de sécurité, organisations ou systèmes de messagerie | politique, réécriture, inspection du contenu, conversion de protocole et propre file d’attente |
| **MDA** – Message Delivery Agent | remet à une boîte aux lettres, un fichier, un pipe ou un stockage propriétaire | résolution finale du destinataire, quota et cohérence de boîte aux lettres |
| **Serveur LMTP** | reçoit un message du MTA pour livraison locale | signale un état distinct pour chaque destinataire après le contenu et n’a pas lui-même besoin d’une file d’attente de relais à long terme |

Le MSA n’est pas simplement un MTA sur un autre port. La soumission de messages sépare l’injection contrôlée de nouveaux messages du transfert ouvert entre domaines de messagerie. Un MSA peut notamment contrôler les droits d’expéditeur, compléter les champs obligatoires manquants et adapter les messages aux règles locales ; un MTA en aval ne devrait en principe pas modifier arbitrairement le contenu de messages finalisés ([RFC 6409, sections 1, 6 et 8](https://datatracker.ietf.org/doc/html/rfc6409#section-1)). Cette limite est particulièrement pertinente pour les signatures DKIM : toute modification ultérieure d’en-têtes signés ou du corps peut invalider la signature.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-smtp-zustellpfad.svg?v=20260813" title="Interaktive Infografik: SMTP-Rollen, Sitzungsablauf, Envelope, Queue und Zustellverantwortung" loading="lazy">
  <a href="/images/kb-interaktiv-smtp-zustellpfad.svg?v=20260813">Ouvrir l’infographie sur le chemin de livraison SMTP et la limite d’acceptation</a>
</iframe>

Le graphique montre des limites logiques. Le MSA, le MTA sortant, la passerelle et le MTA entrant peuvent fonctionner comme des services distincts, être réunis sur un même système ou être masqués par un fournisseur cloud. Les normes ne prescrivent aucune architecture de processus. Sur le plan administratif, la séparation reste néanmoins nécessaire, car chaque transfert accepté peut créer une nouvelle file d’attente, un nouvel état de nouvelle tentative et un nouveau responsable.

## Enveloppe et contenu du message

Avant d’examiner le dialogue en détail, il faut distinguer deux zones de données : l’**enveloppe** pour le transport et le **contenu du message** pour les destinataires et les programmes de messagerie. L’enveloppe consiste en un chemin de retour issu de `MAIL FROM` et un ou plusieurs chemins de transfert issus de `RCPT TO`. Le contenu suit `DATA` et comprend une zone d’en-tête conforme à l’Internet Message Format ainsi que le corps, souvent avec une structure MIME ([RFC 5321, section 2.3.1](https://datatracker.ietf.org/doc/html/rfc5321#section-2.3.1), [RFC 5322, section 2.1](https://datatracker.ietf.org/doc/html/rfc5322#section-2.1)).

| Élément | Objet de protocole | Usage réel |
|---|---|---|
| `MAIL FROM:<bounce@example.net>` | chemin de retour de l’enveloppe | destination des rapports de non-livraison et entrée pour SPF ; peut être vide pour les DSN |
| `RCPT TO:<user@example.ch>` | chemin de transfert de l’enveloppe | routage effectif et décision de destinataire ; répétable par destinataire |
| `From: Author <author@example.net>` | en-tête de message | identité visible de l’auteur et domaine de référence DMARC ; aucune commande de routage SMTP |
| `To:` et `Cc:` | en-tête de message | indications de destination visibles pour les personnes et les MUA ; ne doivent pas correspondre à tous les destinataires de l’enveloppe |
| `Bcc:` | information de création | les destinataires Bcc sont livrés au moyen d’adresses d’enveloppe, sans divulguer leur adresse aux autres copies |
| `Message-ID:` | en-tête de message | identifiant de message de contenu ; différent de l’ID de file d’attente ou de l’ID d’enveloppe DSN |
| `Return-Path:` | en-tête de trace généré lors de la livraison finale | conserve le dernier chemin de retour utilisé ; ne doit pas être repris comme ligne d’expéditeur librement définie depuis un message entrant |

Les serveurs SMTP ne peuvent pas déduire la route finale de `To:`, `Cc:` ou `From:`. Inversement, les adresses d’enveloppe ne doivent pas apparaître dans l’en-tête du message. C’est pourquoi Bcc, les listes de diffusion, les redirections, SRS et les adresses de rebond distinctes fonctionnent. Cette séparation explique aussi pourquoi un message peut sembler visuellement provenir de `ceo@example.ch` dans le client, alors que son expéditeur d’enveloppe utilise un autre domaine. SPF considère le chemin SMTP ; DMARC établit en outre un lien avec le `From:` visible. Les détails de ces vérifications sont traités dans [l’authentification des e-mails](/kb/mail-auth) ([RFC 5321, annexe C](https://datatracker.ietf.org/doc/html/rfc5321#appendix-C), [RFC 5322, section 3.6](https://datatracker.ietf.org/doc/html/rfc5322#section-3.6)).

Un chemin de retour vide est transmis sous la forme `MAIL FROM:<>`. Il évite les boucles de rapports d’erreur et est obligatoire pour les Delivery Status Notifications. Un serveur qui refuse systématiquement les expéditeurs vides endommage donc le chemin de retour du système de messagerie ([RFC 5321, section 3.6.3](https://datatracker.ietf.org/doc/html/rfc5321#section-3.6.3)).

## Session et transaction SMTP

Une **session SMTP** s’exécute sur une [connexion TCP](/kb/tcp); plusieurs messages peuvent y suivre sous forme de **transactions de courrier** distinctes. Le serveur commence par un message d’accueil `220` ; le client s’identifie avec `EHLO` et reçoit une liste de capacités ESMTP propres à la session. Une transaction se compose ensuite de `MAIL`, d’au moins un `RCPT` accepté et de `DATA` ou d’un substitut négocié tel que `BDAT` ([RFC 5321, sections 3.1 et 4.1](https://datatracker.ietf.org/doc/html/rfc5321#section-3.1)).

1. **Connexion et message d’accueil :** le client se connecte ; le serveur répond avec `220`. S’il tarde ou refuse déjà à ce stade, aucune transaction de courrier n’existe encore.
2. **Négociation des capacités :** `EHLO client.example` place les deux côtés dans l’état initial et fournit des extensions telles que `SIZE`, `STARTTLS`, `PIPELINING` ou `SMTPUTF8`. Un client ne peut pas supposer qu’un autre nœud du cluster ou une session ultérieure offre les mêmes capacités.
3. **Mise à niveau TLS facultative :** après un `STARTTLS` réussi, l’état SMTP et les capacités précédemment apprises sont abandonnés. Le client envoie de nouveau `EHLO` dans le canal protégé ([RFC 3207, section 4](https://datatracker.ietf.org/doc/html/rfc3207#section-4)).
4. **Expéditeur de l’enveloppe :** `MAIL FROM` ouvre une transaction et vide les précédents tampons d’enveloppe et de données. Des paramètres ESMTP comme `SIZE`, `BODY` ou `SMTPUTF8` sont associés à cette commande.
5. **Destinataires :** chaque `RCPT TO` reçoit une réponse individuelle. Un serveur peut accepter deux destinataires et rejeter définitivement un troisième ; le client envoie alors le contenu pour le sous-ensemble accepté.
6. **Contenu :** après `DATA`, le serveur répond `354` lorsqu’il est prêt. Le client transfère les en-têtes et le corps sous forme de lignes terminées par CRLF, échappe les points en début de ligne et termine par une ligne composée uniquement de `.`.
7. **Conclusion :** la réponse suivant ce marqueur de fin de données décide si le serveur prend en charge le message dans son ensemble pour les destinataires précédemment acceptés. `RSET` abandonne la transaction en cours sans fermer la session ; `QUIT` la ferme proprement.

SMTP est orienté lockstep, mais ESMTP-`PIPELINING` peut envoyer des groupes de commandes autorisés sans attendre chaque réponse individuelle. Les réponses restent dans l’ordre de réception et doivent toujours être correctement associées à chaque commande ([RFC 2920](https://datatracker.ietf.org/doc/html/rfc2920)). Une capture de paquets montrant plusieurs lignes `MAIL`-/`RCPT` directement successives n’est donc pas automatiquement une violation du protocole.

Le dialogue est ainsi complet, mais la responsabilité de livraison n’est pas encore clarifiée. Elle ne change qu’avec une réponse bien précise du serveur destinataire.

## Limite d’acceptation : 250 après End-of-Data

La limite SMTP la plus importante se situe après le contenu du message. Un `250` en réponse à `MAIL FROM` n’accepte que le chemin de retour. Un `250` en réponse à `RCPT TO` accepte un destinataire pour la transaction en cours. Seul un `250` après le marqueur End-of-Data signifie que le serveur a assumé la responsabilité du message. Un système correct doit ensuite soit le livrer, soit produire un rapport de non-livraison si une erreur est détectée ultérieurement ; le client expéditeur peut retirer sa copie locale de la responsabilité de livraison active ([RFC 5321, sections 4.1.1.4 et 6.1](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4)).

Cette sémantique rend la dernière partie du dialogue critique. Si la connexion est interrompue après le point mais avant la réception complète d’une réponse, le client ne sait pas avec certitude si le serveur a stocké le message. Il doit pouvoir tenter à nouveau la livraison ; des doublons sont donc possibles. SMTP n’offre aucune garantie globale d’exécution exactement une fois. Les systèmes nécessitent par conséquent un suivi idempotent via l’ID de file d’attente, `Message-ID`, le destinataire, l’heure, la taille et, le cas échéant, une référence de transaction propriétaire, sans assimiler ces identifiants.

Les filtres de contenu, la DLP, l’analyse antimalware ou la signature DKIM devraient être intégrés à la chaîne d’acceptation de manière à ce que le serveur sache, avant la réponse positive finale, s’il peut assumer une responsabilité persistante. Un proxy en amont qui signale prématurément `250` puis perd le message en interne ne peut plus renvoyer l’erreur au client SMTP d’origine. C’est précisément ici qu’une architecture doit déterminer si l’acceptation est confirmée uniquement après une écriture durable, après réplication, après analyse ou déjà après la remise à un courtier interne. Il s’agit d’une propriété de produit, non d’une exigence de la norme SMTP, et elle doit être étayée par la documentation du fabricant ainsi que par des tests de défaillance et de restauration.

Après l’acceptation, le MTA doit décider vers où envoyer ensuite le message. Pour le courrier Internet, cette décision commence dans le DNS.

## Routage via DNS et MX

Pour une livraison Internet directe, le MTA expéditeur détermine la prochaine destination à partir du domaine du destinataire de l’enveloppe. Il interroge [DNS](/kb/dns) pour les enregistrements MX, trie les candidats par valeur de préférence croissante et résout les noms Exchange sélectionnés en adresses A ou AAAA. Plusieurs destinations de même préférence constituent des candidats équivalents ; un MX de secours à valeur plus élevée n’est pas une réplique de la file d’attente primaire, mais un destinataire SMTP indépendant avec sa propre politique et sa responsabilité de livraison ([RFC 5321, section 5](https://datatracker.ietf.org/doc/html/rfc5321#section-5)).

En l’absence de jeu d’enregistrements MX, SMTP traite historiquement le domaine comme un MX implicite pointant vers le nom de domaine lui-même et tente ses adresses. Un **Null MX** explicite avec la préférence `0` et Exchange `.` indique en revanche qu’un domaine n’accepte pas d’e-mails ([RFC 7505](https://datatracker.ietf.org/doc/html/rfc7505)). `NXDOMAIN`, `NODATA`, `SERVFAIL`, les échecs de validation DNSSEC et un MX accessible sans service SMTP fonctionnel sont des états distincts et doivent rester séparément visibles dans le motif de mise en file d’attente.

Les chemins de messagerie internes utilisent fréquemment des next hops statiques, des smarthosts, des connecteurs ou des tables de routage au lieu de la sélection MX publique. Le destinataire de l’enveloppe reste néanmoins l’adresse de livraison ; la politique de routage locale décide seulement à quel système suivant elle est confiée. Pour chaque diagnostic, trois noms doivent donc être consignés : domaine du destinataire, nom du next hop sélectionné et adresse IP effectivement connectée.

Même lorsqu’il connaît la destination, le MTA ne peut pas toujours livrer immédiatement. Les erreurs temporaires conduisent donc à la file d’attente plutôt qu’à la perte du message.

## File d’attente, nouvelles tentatives et DSN

Un MTA complet est un système store-and-forward. S’il ne peut pas joindre immédiatement le saut suivant ou si celui-ci répond temporairement, le MTA conserve le message et l’enveloppe dans une file d’attente persistante, planifie une nouvelle tentative et gère l’état par destinataire. RFC 5321 exige une stratégie de nouvelles tentatives, mais ne prescrit pas un algorithme unique pour tous les produits. Elle recommande des répétitions différées, une notification en cas de retard prolongé et une conservation de plusieurs jours avant l’abandon définitif ([RFC 5321, section 4.5.4](https://datatracker.ietf.org/doc/html/rfc5321#section-4.5.4)). Les valeurs produit telles que le backoff, la durée de vie maximale de la file d’attente, le parallélisme et la concurrence par destination doivent donc être lues dans la configuration MTA concernée.

Un enregistrement de file d’attente pertinent contient au minimum :

- le contenu de message immuable ou un blob référencé ;
- l’expéditeur de l’enveloppe et l’état de chaque destinataire de l’enveloppe ;
- l’heure d’acceptation, la prochaine tentative et la limite d’expiration ;
- le transport choisi, le next hop, la dernière adresse et la décision TLS/de politique ;
- le dernier code SMTP et code d’état étendu avec le texte distant ;
- l’ID de file d’attente local ainsi que des ID corrélables provenant de sauts en amont ou en aval ;
- les états active, deferred, hold, delivered, bounced et quarantined ;
- l’état de réplication, du système de fichiers ou de la base de données si l’acceptation doit être hautement disponible.

Les erreurs temporaires font normalement l’objet de nouvelles tentatives. Les erreurs permanentes avant sa propre acceptation sont renvoyées de manière synchrone au client SMTP expéditeur. Les erreurs permanentes après sa propre acceptation donnent lieu à une **Delivery Status Notification** (DSN) vers le chemin de retour. RFC 3464 définit à cet effet un message `multipart/report` lisible par machine, avec des champs par message et par destinataire tels que `Reporting-MTA`, `Final-Recipient`, `Action`, `Status` et, en option, `Remote-MTA` ou `Diagnostic-Code` ([RFC 3464](https://datatracker.ietf.org/doc/html/rfc3464)).

Un rebond n’est pas automatiquement digne de confiance. Les DSN peuvent être falsifiées, divulguer le contenu d’origine ou créer du backscatter lorsque l’expéditeur de l’enveloppe a été défini de manière abusive. Une passerelle devrait rejeter les messages non valides, si possible pendant la session SMTP initiale, plutôt que de les accepter puis de les renvoyer ultérieurement à une adresse potentiellement falsifiée.

## Codes de réponse et signification pour le diagnostic

Les réponses SMTP possèdent un code de base à trois chiffres. La première plage de chiffres décrit l’état immédiat ; le texte est destiné aux humains et peut être spécifique au produit ([RFC 5321, section 4.2](https://datatracker.ietf.org/doc/html/rfc5321#section-4.2)).

| Classe | Signification dans le dialogue | Conséquence pour le processus émetteur |
|---|---|---|
| `2xx` | achèvement positif | exécuter l’étape de protocole suivante ; après End-of-Data, enregistrer la responsabilité comme transférée |
| `3xx` | intermédiaire positif | envoyer des données supplémentaires, par exemple le contenu du message après `354` |
| `4xx` | achèvement négatif transitoire | conserver le message ou le destinataire concerné dans la file d’attente et réessayer conformément à la politique de nouvelle tentative |
| `5xx` | achèvement négatif permanent | ne pas répéter sans modification ; signaler de manière synchrone au client précédent ou produire une DSN après acceptation antérieure |

Les **Enhanced Status Codes** complètent le format `x.y.z`. `x` indique succès, temporaire ou permanent ; `y` désigne la catégorie générale de défaillance et `z` le détail. Ainsi, `5.1.1` relève de la classe d’adresse et désigne une boîte aux lettres cible inexistante, tandis que `4.4.x` classe les problèmes de réseau et de routage. Les significations enregistrées figurent dans le [registre IANA des Enhanced Status Codes](https://www.iana.org/assignments/smtp-enhanced-status-codes/smtp-enhanced-status-codes.xhtml) ; un texte de fabricant ne doit pas être interprété à l’encontre de la sémantique numérique ([RFC 3463](https://datatracker.ietf.org/doc/html/rfc3463), [RFC 5248](https://datatracker.ietf.org/doc/html/rfc5248)).

Le même code peut concerner quelque chose de différent selon la phase du protocole. Un `550` sur `MAIL FROM` rejette l’expéditeur ou son autorisation, sur `RCPT TO` un destinataire spécifique, et après le contenu la transaction complète. Un ticket sans hôte distant, heure, phase, codes de base, code étendu et texte de réponse complet est donc incomplet.

## ESMTP et extensions

Extended SMTP utilise `EHLO` pour négocier les capacités par connexion. L’IANA tient le [registre contraignant des SMTP Service Extensions](https://www.iana.org/assignments/smtp/smtp.xhtml). Un client ne peut utiliser une extension que si elle a été proposée dans la session actuelle ; après STARTTLS, il doit interroger à nouveau la liste.

| Mot-clé EHLO | Effet technique | Pertinence administrative |
|---|---|---|
| `SIZE` | le serveur peut indiquer une limite de taille fixe ; le client peut déclarer la taille attendue avec `MAIL FROM` | évite les transferts de données inutiles, mais ne garantit pas encore l’acceptation ([RFC 1870](https://datatracker.ietf.org/doc/html/rfc1870)) |
| `PIPELINING` | permet des groupes de commandes définis sans aller-retour par commande | augmente le débit sur les liaisons à forte latence ; les réponses doivent être corrélées exactement dans l’ordre ([RFC 2920](https://datatracker.ietf.org/doc/html/rfc2920)) |
| `8BITMIME` | autorise le transport de corps MIME 8 bits avec `BODY=8BITMIME` | n’autorise pas des données binaires arbitraires et ne modifie pas les règles d’en-tête ([RFC 6152](https://datatracker.ietf.org/doc/html/rfc6152)) |
| `CHUNKING` | introduit `BDAT` avec des nombres d’octets explicites comme alternative à `DATA` | aucune transparence des points ; condition préalable pour `BINARYMIME` ([RFC 3030](https://datatracker.ietf.org/doc/html/rfc3030)) |
| `SMTPUTF8` | autorise les boîtes aux lettres internationalisées et UTF-8 dans les en-têtes définis à cette fin | chaque saut suivant requis doit prendre en charge cette capacité, sinon une erreur d’interopérabilité survient ([RFC 6531](https://datatracker.ietf.org/doc/html/rfc6531)) |
| `DSN` | transporte les paramètres de notification de livraison sur `MAIL` et `RCPT` | distingue les rapports de succès, de retard et d’échec souhaités ([RFC 3461](https://datatracker.ietf.org/doc/html/rfc3461)) |
| `STARTTLS` | passe du dialogue en clair à un canal TLS | réinitialise l’état SMTP ; la sécurité dépend également de la politique lors du transfert MTA ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207)) |
| `AUTH` | intègre l’authentification SASL dans SMTP et liste les mécanismes | prévu pour la soumission ; les mécanismes proposés peuvent changer après STARTTLS ([RFC 4954](https://datatracker.ietf.org/doc/html/rfc4954)) |

La dérive des capacités entre nœuds de cluster est un problème opérationnel fréquent. Si seuls certains écouteurs proposent `SMTPUTF8`, la même valeur `SIZE` ou des mécanismes AUTH identiques, des échecs de livraison apparemment aléatoires surviennent. La supervision ne devrait donc pas contrôler uniquement une VIP, mais chaque chemin backend pertinent avec le même SNI, nom EHLO et réseau source.

Jusqu’ici, il s’agissait du transport entre serveurs. La soumission par des utilisateurs et applications constitue un service distinct avec d’autres ports, identités et risques d’abus.

## Soumission, relais et authentification

Le port **25** sert au transfert MTA à MTA. Le port **587** désigne la soumission de messages ; le port **465** est enregistré pour la soumission avec TLS actif dès le début sous le nom de service `submissions`. Le port 465 n’est pas un port MX alternatif : la livraison MX publique ne connaît aucune indication de port et utilise le port 25 ([RFC 6409, sections 1 et 3](https://datatracker.ietf.org/doc/html/rfc6409#section-3.1), [RFC 8314, section 7.3](https://datatracker.ietf.org/doc/html/rfc8314#section-7.3), [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)).

Un service de soumission associe authentification et autorisation. `AUTH` utilise SASL et peut transporter des mécanismes tels que PLAIN sur TLS ou des jetons Bearer basés sur OAuth. L’identité authentifiée, l’identité d’autorisation demandée et l’expéditeur d’enveloppe autorisé sont des grandeurs distinctes. Une connexion réussie n’autorise pas automatiquement l’utilisation de tout domaine expéditeur ou de tout expéditeur d’enveloppe ([RFC 4422, section 3.4](https://datatracker.ietf.org/doc/html/rfc4422#section-3.4), [RFC 4954](https://datatracker.ietf.org/doc/html/rfc4954), [RFC 7628](https://datatracker.ietf.org/doc/html/rfc7628)).

Un relais ouvert apparaît lorsqu’un MTA transmet des messages depuis des sources non autorisées vers des destinations non locales. Une configuration sûre décide donc séparément des domaines de destinataires locaux, des réseaux sources de confiance, de la soumission authentifiée et des connecteurs explicites. Une liste d’autorisation IP est une autorisation réseau et ne remplace pas une identité utilisateur ; avec NAT, une sortie cloud ou des relais partagés, elle peut autoriser un ensemble d’expéditeurs bien plus large que prévu.

Ce n’est qu’une fois le rôle et le port définis que la politique TLS appropriée peut être déterminée : la soumission doit être protégée et authentifiée, tandis que le transport MTA ouvert nécessite des politiques de domaine supplémentaires.

## Politique TLS pour SMTP

STARTTLS protège la connexion individuelle lorsque les deux parties effectuent la mise à niveau. Dans le transfert MTA ouvert, STARTTLS seul reste toutefois opportuniste : un attaquant actif peut supprimer l’offre ou manipuler les destinations DNS, et un expéditeur sans politique supplémentaire peut revenir au clair. [DANE pour SMTP](https://datatracker.ietf.org/doc/html/rfc7672) lie les données TLSA au choix de la destination MX via DNSSEC. [MTA-STS](https://datatracker.ietf.org/doc/html/rfc8461) publie via DNS et HTTPS une politique MX et TLS fondée sur PKIX. [TLS-RPT](https://datatracker.ietf.org/doc/html/rfc8460) signale les erreurs agrégées relatives à de telles politiques. L’article [TLS](/kb/tls) traite les détails cryptographiques, les chaînes de certificats et les procédures de test.

La soumission est différente : ici, le client et le service sont administrativement associés, et l’utilisateur ne doit pas revenir silencieusement au clair. RFC 8314 recommande le TLS implicite pour la soumission et l’accès au courrier et exige que le clair soit considéré comme obsolète. SMTP AUTH avec des mécanismes de mot de passe ou Bearer nécessite un canal suffisamment protégé ; un serveur ne devrait même pas proposer de mécanismes faibles avant STARTTLS ([RFC 8314, section 3](https://datatracker.ietf.org/doc/html/rfc8314#section-3), [RFC 4954, section 14](https://datatracker.ietf.org/doc/html/rfc4954#section-14)).

## Livraison locale avec LMTP

Le **Local Mail Transfer Protocol** (LMTP) utilise un dialogue semblable à SMTP pour la remise à un système sans propre file d’attente de courrier à long terme, par exemple un service de boîtes aux lettres. La différence essentielle apparaît après le contenu du message : LMTP fournit une réponse finale distincte pour chaque destinataire préalablement accepté. Le stockage de courrier peut ainsi livrer avec succès à un destinataire et rejeter un autre à cause d’un quota ou d’une erreur locale, sans que le MTA en amont doive traiter toute la transaction de manière uniforme ([RFC 2033, section 4.2](https://datatracker.ietf.org/doc/html/rfc2033#section-4.2)).

LMTP ne déplace pas la responsabilité hors du système. Si le stockage de courrier rejette temporairement un destinataire, le MTA en amont doit conserver ce destinataire dans sa file d’attente. Avec les sockets Unix, le chemin réseau IP disparaît certes, mais la nécessité de surveiller les autorisations des sockets, la contre-pression, les délais d’attente, les réponses par destinataire et la limite de persistance demeure.

## Pile d’implémentation et de technologies

SMTP définit le protocole sur le fil, non l’architecture logicielle interne. En pratique, un MTA associe plusieurs couches techniques :

| Couche | Composants typiques | État opérationnel critique |
|---|---|---|
| Sélection de service | résolveur DNS, MX/A/AAAA, tables de routage statiques | cache, état DNSSEC, sélection du next hop et IP connectée |
| Transport réseau | pile TCP, écouteur, équilibreur de charge, proxy | limites de connexion, délais d’attente, préservation de l’IP source et contre-pression |
| Protection de canal | bibliothèque TLS, moteur de certificats et de politiques | SNI, magasin de confiance, décision DANE/MTA-STS et repli |
| Frontend de protocole | analyseur SMTP, automate d’état, AUTH/SASL, limites de débit | état de session, capacités proposées et limites contre les abus |
| Politique et contenu | résolution d’adresses, antispam, antimalware, DLP, signature | ordre déterministe, délais d’attente et comportement lors de la défaillance d’un scanner |
| Persistance | fichiers de file d’attente/spool, base de données ou blob store | écriture durable avant `250`, espace libre, réplication et récupération |
| Planificateur et routage | minuteries de nouvelle tentative, concurrence par destination, tables de transport | équité, backoff, âge de la file d’attente et domaines bloqués |
| Adaptateur de livraison | SMTP, LMTP, boîte aux lettres locale, fichier, pipe ou API | état par destinataire, idempotence et limite de remise |
| Observabilité | journaux structurés, ID de file d’attente, métriques, traces et DSN | corrélation entre sauts sans perte de contenu sensible |

Les implémentations organisent ces couches différemment. [Postfix](https://www.postfix.org/OVERVIEW.html) répartit la réception, le nettoyage, le gestionnaire de file d’attente et les agents de livraison en processus coopérants et distingue `incoming`, `active` et `deferred` comme états de file d’attente. [Exim](https://www.exim.org/exim-html-current/doc/html/spec_html/ch-how_exim_receives_and_delivers_mail.html) stocke l’enveloppe/les en-têtes et le corps dans le spool, puis laisse les queue runners traiter les tentatives de livraison arrivées à échéance. [Apache James](/kb/apache-james) s’exécute sur la JVM et associe serveur de protocole, file d’attente et pipeline Mailet avec des backends de stockage interchangeables. Tous trois parlent SMTP, mais leurs modèles de processus, de file d’attente, d’extensions et de récupération ne sont pas interchangeables.

Le langage de programmation et le runtime influencent le packaging, le processus de correctifs, le profil mémoire et l’extensibilité, non la sémantique du dialogue SMTP. Postfix est construit avec une chaîne d’outils C native pour les systèmes de type Unix ([Postfix INSTALL](https://www.postfix.org/INSTALL.html)) ; la [documentation Exim](https://www.exim.org/exim-html-2.00/doc/html/oview.html) cite ANSI C comme langage d’implémentation. Apache James se décrit en revanche comme une plateforme Java/Scala sur la JVM ([Apache James](https://james.apache.org/)). Un service cloud peut représenter les mêmes rôles RFC via des microservices propriétaires, des files d’attente distribuées et des API. Pour la validation opérationnelle, il faut donc documenter non seulement les capacités EHLO, mais aussi l’engagement de persistance, le modèle de nouvelles tentatives, l’export de file d’attente, la sauvegarde, la gestion des clés, les limites de mise à l’échelle et les domaines de défaillance.

Du déroulement de livraison découle une méthode de recherche d’erreurs fixe : déterminer la destination, vérifier la connexion et le dialogue SMTP, évaluer la réponse, puis suivre la file d’attente et le saut suivant.

## Diagnostic du DNS à la file d’attente

Un diagnostic SMTP fiable suit le chemin de livraison et sépare les niveaux d’observation. Les tests doivent être réalisés avec des domaines de test et des destinataires autorisés ; des serveurs tiers ne doivent pas être utilisés abusivement pour des tests de relais, de validation d’adresses ou de charge.

### Déterminer la cible MX et les adresses

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-DNS-Prüfung">
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

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) et [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) affichent les données DNS publiées. La sortie doit être vérifiée quant à la préférence, au Null MX, aux chaînes CNAME, à IPv4/IPv6 et au résolveur effectivement interrogé. Une réponse DNS ne prouve ni le fonctionnement de l’écouteur ni celui de TLS.

### Vérifier l’accessibilité TCP du saut sélectionné

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-TCP-Prüfung">
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

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) et [`nc`](https://man.openbsd.org/nc) vérifient ici uniquement la résolution DNS et l’établissement TCP. Une réussite sur le port 25 ne dit pas si le message d’accueil SMTP arrive à temps ; une réussite sur 587 ou 465 ne dit rien de la politique TLS et AUTH.

### Vérifier STARTTLS, EHLO et le certificat

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

[`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) exécute le dialogue en clair requis avec `-starttls smtp`, définit le nom SMTP-EHLO avec `-name` et SNI avec `-servername`. `-verify_hostname` et `-verify_return_error` transforment le simple handshake en test explicite de nom PKIX. Le résultat décrit OpenSSL avec son magasin de confiance, et non automatiquement la politique du MTA de production.

### Tester de manière contrôlée l’enveloppe et la phase de réponse

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für kontrollierten SMTP-Dialog">
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

[`swaks`](https://jetmore.org/john/code/swaks/) journalise les commandes et réponses SMTP et peut terminer délibérément le test après `RCPT`, sans envoyer de données de message. L’outil ne fait pas partie du système sous Windows ; `swaks.exe` ou une installation Perl doit provenir d’une source contrôlée. Un test RCPT ne doit être effectué que contre des systèmes et adresses propres ou explicitement autorisés, car il déclenche des politiques, des annuaires et des limites de débit.

### Examiner les écouteurs et les paquets sur le serveur

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-Listener- und Paketprüfung">
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

[`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) et [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) associent les écouteurs et états TCP à des processus locaux. [`pktmon`](https://learn.microsoft.com/windows-server/networking/technologies/pktmon/pktmon) et [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) capturent des paquets ; les captures de production peuvent contenir des adresses d’enveloppe, des contenus non chiffrés et des données d’authentification, et doivent être protégées, limitées et supprimées en conséquence. [`Wireshark`](https://www.wireshark.org/docs/wsug_html/index.html) convient à l’analyse de TCP, SMTP et des handshakes TLS, mais ne remplace ni les journaux de file d’attente ni les journaux de politique.

### Associer les erreurs à une limite

| Observation | Limite probable | Prochaine vérification fiable |
|---|---|---|
| aucun MX ou Null MX | publication DNS du domaine destinataire | réponse autoritative, état DNSSEC, TTL du cache négatif |
| délai TCP vers un MX parmi plusieurs | chemin réseau ou MX individuel | chaque adresse MX et famille IP séparément, SYN/SYN-ACK et chemin de retour |
| connexion établie, aucun `220` | écouteur, proxy, tarpit ou surcharge | latence du message d’accueil, limites de connexion, journaux frontend et backend |
| `STARTTLS` absent seulement sur certaines connexions | dérive de capacités, proxy ou politique selon le réseau source | chaque nœud de cluster, EHLO avant/après TLS, chemin de l’équilibreur de charge |
| `4xx` lors de `MAIL FROM` | politique d’expéditeur, de débit ou de ressources | code complet, IP source, expéditeur d’enveloppe, journal de file d’attente/politique |
| `5xx` pour un `RCPT TO` | décision de destinataire ou de relais | destinataire d’enveloppe concerné, domaines acceptés, périphérie d’annuaire |
| `4xx` après End-of-Data | scanner, stockage, file d’attente ou politique de contenu | chaîne d’acceptation locale, espace du spool, latence du scanner, aucune prise en charge en file d’attente |
| `250` après End-of-Data, DSN ultérieure | échec de livraison en aval après sa propre acceptation | MTA rapporteur, MTA distant, ID de file d’attente, état DSN et destinataire d’origine |
| la file d’attente augmente pour de nombreux domaines | planificateur local, DNS, réseau ou ressources | distribution d’âge, débits de livraison, états active/deferred, dépendances globales |
| la file d’attente augmente pour un seul domaine | MX distant, politique TLS ou throttling de destination | dernière erreur par MX/IP, DANE/MTA-STS, concurrence et heure de nouvelle tentative |
| boîte aux lettres absente malgré un saut SMTP réussi | MDA/LMTP ou saut de routage interne | dernière ligne `Received` fiable, réponse LMTP par destinataire, journaux du stockage |
| doublons sporadiques après délai d’attente | réponse ambiguë à la limite d’acceptation | instant de l’interruption par rapport au `250` final, corrélation ID de file d’attente et Message-ID |

## Observabilité et modèle opérationnel

Un moniteur SMTP ne devrait pas seulement envoyer un e-mail de test. Il doit rendre visibles les différents états du système :

- latence distincte du message d’accueil, EHLO, STARTTLS, AUTH et End-of-Data ;
- connexions, sessions, transactions actives et refus par phase ;
- capacités ESMTP proposées par écouteur et backend ;
- taux d’acceptation, nombre de destinataires, taille de message et latence d’écriture durable ;
- volume de file d’attente par âge, destination, transport, état et dernière erreur ;
- taux de nouvelles tentatives, message le plus ancien, livraisons arrivant à expiration et génération de DSN ;
- cible DNS, IP connectée, IPv4/IPv6, mode TLS, chiffrement et résultat de politique ;
- codes de réponse comme champs structurés plutôt que seulement comme texte libre ;
- corrélation de l’ID edge, de l’ID interne de file d’attente, de l’ID de file d’attente distante, de `Message-ID` et du destinataire ;
- état LMTP/boîte aux lettres par destinataire ainsi que chemins de quarantaine et de libération ;
- capacité du spool, latence d’E/S, état de la base de données ou du courtier et retard de réplication ;
- limites de protection pour les journaux et captures, car l’enveloppe, les en-têtes et les contenus contiennent des données personnelles ou confidentielles.

Les sauvegardes de file d’attente seules ne suffisent pas. La restauration doit prendre en compte la configuration, les tables de routage, les certificats et clés, les métadonnées de file d’attente, les blobs de messages, la quarantaine, l’état de signature et l’idempotence. Après une restauration, une copie déjà livrée à un destinataire ne doit pas être renvoyée sans contrôle ; en même temps, les destinataires encore ouverts d’un message partiellement livré ne doivent pas être perdus. Les RPO et RTO doivent donc être définis par composant d’état et testés avec une véritable file d’attente comportant plusieurs destinataires.

## Histoire technique

SMTP est issu de procédures de messagerie ARPANET antérieures. La [RFC 821 de Jon Postel](https://datatracker.ietf.org/doc/html/rfc821) a normalisé en 1982 le modèle encore reconnaissable aujourd’hui, composé de SMTP émetteur, SMTP récepteur, `MAIL`, `RCPT`, `DATA` et réponses à trois chiffres. Le format de message a été décrit séparément la même année dans la RFC 822. Cette séparation entre protocole de transfert et format de message a été conservée.

Avec MIME, SMTP a dépassé le simple texte 7 bits. Le framework ESMTP a introduit `EHLO`, des capacités enregistrées et des paramètres sur `MAIL` et `RCPT` ; sa version de 1995 figurait dans la [RFC 1869](https://datatracker.ietf.org/doc/html/rfc1869). Des extensions fondées sur celui-ci, telles que `SIZE`, `8BITMIME`, `PIPELINING`, `AUTH`, `STARTTLS`, `CHUNKING`, DSN et, plus tard, `SMTPUTF8`, ont développé le protocole de façon incrémentale sans créer une pile parallèle « SMTP 2 ».

La [RFC 2821](https://datatracker.ietf.org/doc/html/rfc2821) a consolidé SMTP en 2001 et remplacé RFC 821 ainsi que plusieurs documents complémentaires. La [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321) a suivi en 2008 comme nouvelle consolidation ; l’Internet Message Format correspondant figure dans la [RFC 5322](https://datatracker.ietf.org/doc/html/rfc5322). La soumission de messages a d’abord été spécifiée séparément, puis consolidée dans RFC 6409. L’évolution technique s’est donc poursuivie par des rôles plus clairs, des registres d’extensions, des adresses internationalisées, des rapports de livraison et des politiques de transport, tandis que le dialogue fondamental à états est resté compatible.

Les architectures logicielles ont également évolué de façon divergente. La compatibilité historique avec Sendmail a façonné les interfaces locales en ligne de commande ; Postfix a misé sur des processus coopérants séparés par privilèges et des files d’attente explicites, Exim sur une approche très configurable de routeur/transport et de spool, et les systèmes JVM comme Apache James ont rendu le traitement extensible via des composants et des Mailets. Les plateformes cloud répartissent les mêmes rôles logiques entre frontends, courtiers, services de politique et files d’attente régionales. Pour les administrateurs, la sémantique RFC demeure donc le dénominateur commun, tandis que la persistance, la mise à l’échelle et la récupération doivent toujours être étayées de manière spécifique au produit.

## Sources

- [RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321) – modèle SMTP, routage, commandes, réponses, file d’attente, nouvelles tentatives, trace et responsabilité de livraison.
- [RFC 5598, section 4](https://datatracker.ietf.org/doc/html/rfc5598)
- [RFC 6409 – Message Submission for Mail](https://datatracker.ietf.org/doc/html/rfc6409) – séparation entre soumission et relais, rôles, port 587 et politique MSA.
- [RFC 5322 – Internet Message Format](https://datatracker.ietf.org/doc/html/rfc5322) – en-têtes, corps, champs d’adresse, Message-ID et champs de trace.
- [RFC 3207 – SMTP Service Extension for Secure SMTP over TLS](https://datatracker.ietf.org/doc/html/rfc3207) – STARTTLS et réinitialisation de l’état SMTP.
- [RFC 2920 – SMTP Command Pipelining](https://datatracker.ietf.org/doc/html/rfc2920) – groupes de commandes ordonnés sans aller-retour par commande.
- [RFC 7505 – A Null MX No Service Resource Record](https://datatracker.ietf.org/doc/html/rfc7505) – désignation explicite des domaines qui ne peuvent pas recevoir d’e-mails.
- [RFC 3464 – Delivery Status Notification Format](https://datatracker.ietf.org/doc/html/rfc3464) – structure DSN lisible par machine et état des destinataires.
- [IANA – SMTP Enhanced Status Codes](https://www.iana.org/assignments/smtp-enhanced-status-codes/smtp-enhanced-status-codes.xhtml) – significations enregistrées des codes détaillés.
- [RFC 3463 – Enhanced Mail System Status Codes](https://datatracker.ietf.org/doc/html/rfc3463) – structure et classes des codes d’état étendus.
- [RFC 5248 – SMTP Enhanced Status Code Registry](https://datatracker.ietf.org/doc/html/rfc5248) – modèle d’enregistrement des Enhanced Codes.
- [IANA – Simple Mail Transfer Protocol Parameters](https://www.iana.org/assignments/smtp/smtp.xhtml) – commandes SMTP, extensions et mots-clés EHLO.
- [RFC 1870 – SMTP SIZE Extension](https://datatracker.ietf.org/doc/html/rfc1870) – négociation de taille et refus avant le contenu.
- [RFC 6152 – SMTP Service Extension for 8-bit MIME Transport](https://datatracker.ietf.org/doc/html/rfc6152) – 8BITMIME et paramètre BODY.
- [RFC 3030 – CHUNKING and BINARYMIME](https://datatracker.ietf.org/doc/html/rfc3030) – transfert BDAT et messages MIME binaires.
- [RFC 6531 – SMTP Extension for Internationalized Email](https://datatracker.ietf.org/doc/html/rfc6531) – SMTPUTF8 et boîtes aux lettres internationales.
- [RFC 3461 – SMTP Service Extension for Delivery Status Notifications](https://datatracker.ietf.org/doc/html/rfc3461) – paramètres d’enveloppe pour les demandes DSN.
- [RFC 4954 – SMTP Service Extension for Authentication](https://datatracker.ietf.org/doc/html/rfc4954) – commande AUTH et profil SASL pour SMTP.
- [RFC 8314, section 7.3](https://datatracker.ietf.org/doc/html/rfc8314)
- [IANA – Service Name and Transport Protocol Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) – attribution des ports SMTP, Submission et Submissions.
- [RFC 4422, section 3.4](https://datatracker.ietf.org/doc/html/rfc4422)
- [RFC 7628 – SASL Mechanisms for OAuth](https://datatracker.ietf.org/doc/html/rfc7628) – mécanismes SASL basés sur OAuth pour SMTP et d’autres protocoles.
- [RFC 7672 – SMTP Security via DANE TLS](https://datatracker.ietf.org/doc/html/rfc7672) – politique de transport SMTP liée à DNSSEC et TLSA.
- [RFC 8461 – SMTP MTA Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc8461) – politique MX et TLS fondée sur PKIX.
- [RFC 8460 – SMTP TLS Reporting](https://datatracker.ietf.org/doc/html/rfc8460) – rapports agrégés sur les erreurs de TLS de transport.
- [RFC 2033, section 4.2](https://datatracker.ietf.org/doc/html/rfc2033)
- [Postfix – Architecture Overview](https://www.postfix.org/OVERVIEW.html) – rôles de processus, gestionnaire de file d’attente et agents de livraison.
- [Exim – How Exim receives and delivers mail](https://www.exim.org/exim-html-current/doc/html/spec_html/ch-how_exim_receives_and_delivers_mail.html) – modèle de spool, routage, transport et queue runners.
- [Postfix – Installation From Source Code](https://www.postfix.org/INSTALL.html) – exigences de build natif, de plateforme et de chaîne d’outils.
- [Exim – aperçu technique](https://www.exim.org/exim-html-2.00/doc/html/oview.html) – ANSI C comme langage d’implémentation et propriétés techniques du produit.
- [Apache James Project](https://james.apache.org/) – architecture JVM, Java/Scala et composants.
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – requêtes DNS sous Windows.
- [BIND 9 – manuel dig](https://bind9.readthedocs.io/en/latest/manpages.html) – requêtes DNS sous les systèmes Unix.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – accessibilité TCP sous Windows.
- [OpenBSD – nc(1)](https://man.openbsd.org/nc) – connexions TCP avec Netcat.
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/) – diagnostic STARTTLS, SNI et certificat.
- [Swaks – Swiss Army Knife for SMTP](https://jetmore.org/john/code/swaks/) – test de transaction SMTP et d’extensions.
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) – écouteurs et connexions TCP locaux.
- [Linux man-pages – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html) – diagnostic de l’état des sockets et TCP.
- [Microsoft Learn – Packet Monitor](https://learn.microsoft.com/windows-server/networking/technologies/pktmon/pktmon) – capture de paquets Windows et conversion PCAPNG.
- [manuel tcpdump(1)](https://www.tcpdump.org/manpages/tcpdump.1.html) – diagnostic par paquets sous les systèmes Unix.
- [Guide de l’utilisateur Wireshark](https://www.wireshark.org/docs/wsug_html/index.html) – analyse interactive de traces réseau et de protocole.
- [RFC 821 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc821) – spécification SMTP historique de 1982.
- [RFC 1869 – SMTP Service Extensions](https://datatracker.ietf.org/doc/html/rfc1869) – framework ESMTP historique et modèle EHLO.
- [RFC 2821 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc2821) – consolidation SMTP de 2001.
- [RFC 3848 – ESMTP and LMTP Transmission Types](https://datatracker.ietf.org/doc/html/rfc3848) – valeurs `with` telles que ESMTPSA dans les en-têtes Received.
