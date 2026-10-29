---
title: "NDR, DSN, Bounce : bien distinguer les notifications de non-remise"
navTitle: "NDR et bounces"
description: "NDR, DSN, bounce, reject, backscatter : les termes relatifs aux échecs de remise sont souvent employés comme synonymes, mais désignent des choses différentes. Ce que définissent les RFC, qui génère quel message, comment se compose une DSN et pourquoi la différence entre reject et bounce détermine le backscatter."
date: "2026-08-28"
kategorie: "SMTP et flux de messagerie"
timeToRead: "10 min de lecture"
themen:
  - smtp-mailflow
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "ndr-dsn-bounce-bien-distinguer-les-notifications-de-non-remise"
translationId: "article-5c5164049a129fa4"
translationOf: ndr-dsn-bounce-unterschiede
translationSourceHash: 1500e27283dc4c53456810f133d12d81e1f60d7e874b6f3450b0a50fb47d4cf2
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:19:36.357Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/ndr-dsn-bounce-bien-distinguer-les-notifications-de-non-remise
---

# NDR, DSN, Bounce : bien distinguer les notifications de non-remise

Un e-mail n'arrive pas, et le ticket mentionne tantôt un « bounce », un « NDR », « Mailer-Daemon » ou un « message d'erreur du serveur ». Dans le quotidien de l'administration, ces termes sont utilisés comme synonymes alors qu'ils désignent des choses différentes : un reject durant la session SMTP n'est pas un e-mail de retour, une notification de retard n'est pas un échec de remise, et un accusé de lecture n'a absolument rien à voir avec la non-remise. Distinguer clairement ces termes permet de trouver plus rapidement la cause, car chaque type de notification indique autre chose sur l'endroit du chemin de transport où se situe le problème et sur la personne susceptible de le résoudre.

## DSN : le terme générique des RFC

Le terme générique formel est Delivery Status Notification (DSN), défini dans les RFC 3461 à 3464. Une DSN est un e-mail généré automatiquement qui informe l'expéditeur de l'état de remise de son message. Point décisif : une DSN ne signale pas uniquement des échecs. Le champ `Action` de la partie lisible par machine connaît cinq valeurs :

| Action | Signification |
|---|---|
| `failed` | Remise définitivement échouée ; l'e-mail ne sera plus réessayé |
| `delayed` | Remise retardée ; le serveur continue les tentatives |
| `delivered` | Remis avec succès (confirmation de remise, uniquement sur demande explicite) |
| `relayed` | Transmis à un serveur qui ne génère pas lui-même de DSN |
| `expanded` | Transmis à une liste de diffusion et distribué |

La notification de non-remise n'est donc qu'un cas particulier : une DSN avec `Action: failed`. Microsoft appelle précisément ce cas particulier Non-Delivery Report (NDR). Le terme NDR vient de l'univers Exchange, mais il est désormais couramment utilisé au-delà des éditeurs. Pour être précis : chaque NDR est une DSN, mais toutes les DSN ne sont pas des NDR.

La notification de retard (`Action: delayed`) mérite une attention particulière, car elle est régulièrement interprétée à tort comme un échec de remise par le support. Un objet typique est « Delivery delayed » ou « Remise retardée ». L'e-mail se trouve alors encore dans la file d’attente du serveur expéditeur, qui continue à essayer, généralement pendant un à deux jours. Ce n'est qu'à l'expiration de la durée de vie de la file d’attente que le NDR définitif suit. Un utilisateur qui renvoie l'e-mail à la suite d'une notification de retard crée des doublons dès que le système cible est à nouveau joignable.

## Reject ou bounce : la distinction la plus importante

Avant d'aborder les autres termes, il faut expliquer l'aiguillage technique central, car c'est lui qui détermine quel serveur génère une notification.

**Reject (refus durant la session) :** Le serveur destinataire refuse l'e-mail dès la session SMTP, avec un code de réponse 5xx sur `RCPT TO` ou après `DATA`. Il n'accepte jamais l'e-mail et ne génère pas lui-même d'e-mail de retour. L'obligation d'informer l'expéditeur incombe au serveur de soumission : le MTA expéditeur voit la réponse 5xx et génère alors le NDR pour son utilisateur local. Dans ce cas, le NDR lu par l'utilisateur provient de son propre serveur, mais cite le message d'erreur du serveur distant.

**Bounce (acceptation avec retour ultérieur) :** Le serveur destinataire accepte l'e-mail avec `250 OK` et ne constate qu'ensuite qu'il ne peut pas le remettre, par exemple parce que la boîte aux lettres n'existe pas, que le quota est plein ou qu'un serveur en aval le refuse. Il porte alors la responsabilité du message et doit envoyer lui-même une DSN à l'expéditeur. Cet e-mail de retour ultérieur est le bounce au sens strict.

Pour la recherche d'erreurs, la différence est immédiatement exploitable : si le propre serveur est indiqué dans le NDR comme système générateur, l'e-mail a été refusé durant la session ou n'est jamais sorti. Si un serveur tiers est l'expéditeur de la notification, la partie distante a d'abord accepté l'e-mail, et le problème se situe derrière son point d'acceptation, invisible pour l'expéditeur.

Deux autres termes liés aux bounces viennent du marketing et ne figurent dans aucune RFC : hard bounce pour les erreurs définitives (5xx, `Action: failed`) et soft bounce pour les erreurs temporaires (4xx, `Action: delayed`). Cette distinction est essentielle pour les plateformes d'envoi, car les hard bounces devraient mener immédiatement au nettoyage des listes. Techniquement, ce sont les mêmes mécanismes que ceux décrits ci-dessus.

## Aperçu des termes

| Terme | Ce que c'est | Qui génère la notification | Standard |
|---|---|---|---|
| DSN | Terme générique : notification d'état de remise (failed, delayed, delivered, relayed, expanded) | Le MTA responsable de l'e-mail | RFC 3461 à 3464 |
| NDR | DSN avec `Action: failed` ; terme Microsoft pour la notification de non-remise | MTA expéditeur (après reject) ou MTA destinataire (après acceptation) | RFC 3464, documentation Microsoft |
| Reject | Refus 5xx pendant la session SMTP en cours ; pas d'e-mail distinct | Personne ; le MTA expéditeur en fait un NDR | RFC 5321 |
| Bounce | E-mail de retour après une acceptation déjà effectuée | MTA destinataire | RFC 5321, RFC 3464 |
| Hard/Soft Bounce | Classification marketing : définitif (5xx) contre temporaire (4xx) | comme bounce | aucune RFC |
| Notification de retard | DSN avec `Action: delayed` ; l'e-mail est encore dans la file d’attente | MTA expéditeur ou relais | RFC 3464 |
| Backscatter | NDR vers des adresses d'expéditeur falsifiées, généralement déclenchés par du spam | MTA destinataires mal configurés | aucune RFC, terme anti-abus |
| MDN / accusé de lecture | Notification d'affichage ou de suppression par le destinataire | Client de messagerie du destinataire | RFC 8098 |
| Réponse d'absence | Réponse automatique d'une boîte aux lettres atteinte | Serveur de boîte aux lettres ou de groupware | RFC 3834 |

## Structure d'une DSN

Les DSN conformes au standard utilisent le type MIME `multipart/report; report-type=delivery-status` avec trois parties : une explication lisible par l'humain, une partie lisible par machine de type `message/delivery-status` et, facultativement, le message original ou ses en-têtes. La partie lisible par machine est la plus précieuse pour le diagnostic, car ses champs sont normalisés :

```text
Reporting-MTA: dns; mail01.example.net
Received-From-MTA: dns; client.example.org

Final-Recipient: rfc822; max.muster@example.com
Action: failed
Status: 5.1.1
Remote-MTA: dns; mx.example.com
Diagnostic-Code: smtp; 550 5.1.1 <max.muster@example.com>:
    Recipient address rejected: User unknown
```

| Champ | Signification |
|---|---|
| `Reporting-MTA` | Le serveur qui a généré cette DSN ; premier indice de responsabilité |
| `Final-Recipient` | L'adresse destinataire à laquelle se rapporte l'état (un bloc par destinataire) |
| `Action` | L'une des cinq valeurs d'état (failed, delayed, delivered, relayed, expanded) |
| `Status` | Enhanced Status Code selon RFC 3463, p. ex. `5.1.1` |
| `Remote-MTA` | Le serveur distant avec lequel le MTA de rapport a communiqué |
| `Diagnostic-Code` | La réponse SMTP littérale du serveur distant ; souvent la ligne la plus parlante |

Une DSN est toujours envoyée avec un expéditeur d'enveloppe vide (`MAIL FROM:<>`). Ce n'est pas une négligence, mais une exigence de la RFC 5321 : l'expéditeur vide empêche qu'une DSN non distribuable entraîne une autre DSN et que deux serveurs s'envoient indéfiniment des messages d'erreur. Il en découle une règle de configuration : un système de messagerie ne doit pas refuser globalement les e-mails ayant un expéditeur d'enveloppe vide, sans quoi les notifications légitimes de non-remise n'atteindront jamais ses propres utilisateurs.

Exchange et Exchange Online respectent le standard pour le format, mais présentent le contenu dans leur propre représentation : l'utilisateur voit une page mise en forme avec une explication en texte clair, suivie de « Generating server » (correspond à `Reporting-MTA`) et des informations brutes. Pour le diagnostic, il est toujours utile de consulter cette partie technique inférieure.

## Lire les Enhanced Status Codes

Le champ `Status` et généralement aussi le `Diagnostic-Code` contiennent un code en trois parties selon RFC 3463 : classe.sujet.détail. La classe indique le caractère définitif, le sujet et le détail indiquent la cause :

| Plage de codes | Signification |
|---|---|
| `2.x.x` | Succès (uniquement dans les confirmations de remise) |
| `4.x.x` | Erreur temporaire ; le serveur réessaie |
| `5.x.x` | Erreur définitive ; aucune autre tentative |
| `x.1.x` | Problème d'adressage, p. ex. `5.1.1` destinataire inconnu, `5.1.10` domaine sans MX |
| `x.2.x` | Problème de boîte aux lettres, p. ex. `5.2.2` boîte aux lettres pleine, `5.2.3` message trop volumineux pour la boîte aux lettres |
| `x.3.x` | Problème du système cible, p. ex. `4.3.2` le système n'accepte rien actuellement |
| `x.4.x` | Réseau et routage, p. ex. `4.4.1` aucune réponse, `4.4.7` durée de vie de la file d’attente expirée |
| `x.5.x` | Erreur de protocole dans le dialogue SMTP |
| `x.7.x` | Politique et sécurité, p. ex. `5.7.1` relais refusé ou refus de politique, `5.7.26` authentification manquante (SPF/DKIM/DMARC) |

Le code de réponse SMTP classique à trois chiffres (par exemple `550`) et l'Enhanced Status Code figurent souvent ensemble sur une même ligne : `550 5.7.1 ...`. Le code à trois chiffres commande le comportement protocolaire du serveur expéditeur ; le code étendu fournit l'information de diagnostic. En cas de contradiction entre le code et le texte libre, le texte libre du serveur distant est souvent la source la plus précise, car de nombreux systèmes définissent des codes génériques et indiquent la véritable cause dans le commentaire, y compris des identifiants de référence pour le support de la partie distante.

À noter : les refus `5.7.x` dus à des filtres de réputation et de contenu donnent souvent volontairement peu d'informations. Se limiter au code dans ce cas conduit à chercher au mauvais endroit ; la liste de blocage ou le fournisseur de filtre mentionné dans le texte libre mène plus rapidement au but.

## Backscatter : le type de bounce nuisible

Le backscatter survient lorsqu'un serveur accepte d'abord du spam avec un expéditeur falsifié, puis envoie un NDR à l'adresse falsifiée. Le NDR atteint ainsi une personne non impliquée, dont l'adresse a été détournée par le spammeur. Lors de grandes vagues de spam, les personnes concernées reçoivent des milliers de NDR pour des e-mails qu'elles n'ont jamais envoyés, et les serveurs qui génèrent ces NDR en masse se retrouvent eux-mêmes sur des listes de blocage (par exemple la liste Backscatterer d'UCEPROTECT).

La solution découle directement de la distinction entre reject et bounce : tout ce qui peut être refusé doit l'être pendant la session SMTP, et non être renvoyé après acceptation. Concrètement, cela signifie validation des destinataires au point d'acceptation le plus externe (la passerelle Edge connaît les adresses valides, via comparaison avec l'annuaire ou Recipient Callout, au lieu de tout accepter et d'échouer en interne), refus du spam et des logiciels malveillants durant la session plutôt que NDR de quarantaine, et renoncement aux NDR pour les messages classifiés comme spam. Un reject ne génère pas de backscatter, car avec un expéditeur falsifié, la réponse 5xx arrive sur le serveur du spammeur, qui n'en crée pas de NDR vers la victime.

## Ce qui n'est pas une notification de non-remise

Trois types de notifications finissent régulièrement dans la même catégorie dans les tickets, mais n'en font pas partie :

**MDN (Message Disposition Notification, RFC 8098) :** L'accusé de lecture. Il n'est pas généré par le système de transport, mais par le client de messagerie du destinataire, et signale l'affichage ou la suppression du message, non sa remise. Le type MIME s'appelle donc `multipart/report; report-type=disposition-notification`. L'absence d'accusé de lecture ne dit rien sur la remise ; la plupart des clients demandent à l'utilisateur ou suppriment entièrement les MDN.

**Réponses d'absence et autorespondeurs (RFC 3834) :** Une réponse d'absence prouve le contraire d'un échec de remise, car elle suppose que l'e-mail a atteint la boîte aux lettres. Dans les descriptions de tickets (« je reçois une réponse automatique, mon e-mail arrive-t-il ? »), il vaut la peine de demander de quel message il s'agit exactement.

**Notifications de quarantaine :** Des messages tels que le résumé de quarantaine de Microsoft 365 ou d'une passerelle informent le destinataire d'e-mails retenus. Ils sont envoyés au destinataire, non à l'expéditeur, et ne suivent aucun standard DSN. Dans ce scénario, l'expéditeur ne reçoit souvent rien du tout, ce qui explique les cas où un e-mail « disparaît sans message d'erreur ».

## Liste de contrôle pour le diagnostic

Lorsqu'une notification est disponible, clarifiez les points suivants dans cet ordre :

1. De quel type s'agit-il : NDR (`Action: failed`), retard (`Action: delayed`), MDN, autorespondeur ou notification de quarantaine ? En cas de notification de retard : attendre, ne pas renvoyer.
2. Qui a généré la notification (`Reporting-MTA` ou « Generating server ») ? Le propre serveur indique un reject ou une erreur interne ; un serveur tiers indique une acceptation avec échec ultérieur du côté distant.
3. Que disent le statut et le code de diagnostic ? La classe 4 par rapport à la classe 5 distingue temporaire et définitif, le sujet (`x.1` adresse, `x.2` boîte aux lettres, `x.4` réseau, `x.7` politique) délimite la cause, et le texte libre du serveur distant fournit les détails.
4. Si toute notification manque alors que l'e-mail n'arrive pas : vérifier le suivi des messages sur son propre système et penser à la quarantaine ou au filtrage silencieux du côté distant.

Les articles consacrés au [Message Tracking et au diagnostic SMTP dans le générateur de commandes](https://rafaelpfister.ch/tools/command-builder) ainsi que le [Mail-Header-Analyzer](https://rafaelpfister.ch/tools/header-analyzer) montrent comment reproduire ensuite de manière ciblée les différents chemins de remise et analyser le chemin de transport d'un e-mail arrivé.

## Historique des modifications

- **25 octobre 2026 :** Correction : le lien vers le Mail-Header-Analyzer a été défini sur la bonne adresse. Le lien de source a été mis à jour vers la nouvelle adresse de Microsoft Learn.

## Sources

1.  [RFC 3461: SMTP Service Extension for Delivery Status Notifications](https://www.rfc-editor.org/rfc/rfc3461) : extension SMTP permettant aux expéditeurs de demander et contrôler les DSN.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463) : définition des codes d'état en trois parties (classe.sujet.détail).

3.  [RFC 3464: An Extensible Message Format for Delivery Status Notifications](https://www.rfc-editor.org/rfc/rfc3464) : structure de la DSN en tant que multipart/report, champs tels que Action, Status et Diagnostic-Code.

4.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321) : règles de base relatives aux codes de réponse, au transfert de responsabilité lors de l'acceptation et à l'expéditeur d'enveloppe vide pour les messages d'erreur.

5.  [RFC 8098: Message Disposition Notification](https://www.rfc-editor.org/rfc/rfc8098) : standard pour les accusés de lecture, afin de les distinguer des DSN.

6.  [RFC 3834: Recommendations for Automatic Responses to Electronic Mail](https://www.rfc-editor.org/rfc/rfc3834) : règles pour les autorespondeurs tels que les réponses d'absence.

7.  [Microsoft Learn: Email non-delivery reports and SMTP errors in Exchange Online](https://learn.microsoft.com/en-us/troubleshoot/exchange/email-delivery/ndr/non-delivery-reports-in-exchange-online) : structure des NDR et liste des codes du point de vue d'Exchange Online.

8.  [UCEPROTECT Backscatterer](https://www.backscatterer.org/) : liste de blocage pour les systèmes générant du backscatter ; explique les critères de référencement.
