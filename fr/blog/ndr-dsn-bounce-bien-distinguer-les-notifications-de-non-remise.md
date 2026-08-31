---
title: "NDR, DSN, Bounce : bien distinguer les notifications de non-remise"
navTitle: "NDR et bounces"
description: "NDR, DSN, bounce, reject, backscatter : les termes relatifs aux échecs de remise sont souvent utilisés comme synonymes, mais désignent des choses différentes. Ce que définissent les RFC, qui génère quelle notification, comment est structurée une DSN et pourquoi la différence entre reject et bounce détermine le backscatter."
date: "2026-08-28"
kategorie: "SMTP et flux de messagerie"
timeToRead: "10 min de lecture"
themen:
  - smtp-mailflow
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "ndr-dsn-bounce-bien-distinguer-les-notifications-de-non-remise"
translationId: "article-5c5164049a129fa4"
aiPrompt: |
  Du bist mein Mailflow-Assistent. Ich füge dir gleich eine Unzustellbarkeitsmeldung (NDR/DSN) ein. Analysiere sie Schritt für Schritt: 1. Welcher Server hat die Meldung erzeugt (Reporting-MTA bzw. Generating server)? 2. Wurde die Mail in der SMTP-Session abgewiesen oder nach Annahme zurückgeschickt? 3. Was bedeuten SMTP-Antwortcode und Enhanced Status Code (RFC 3463) konkret? 4. Liegt die Ursache beim Absender, beim Empfänger oder auf dem Transportweg? 5. Welche nächsten Diagnose-Schritte empfiehlst du?
translationOf: ndr-dsn-bounce-unterschiede
url: https://rafaelpfister.ch/fr/blog/ndr-dsn-bounce-bien-distinguer-les-notifications-de-non-remise
translationSourceHash: e526de6f4a454b4f4975eac3e8a406ab5b30314c624bf12c69f87bec99fdd0e7
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:32:19.171Z
translationReview: automatic
---

# NDR, DSN, Bounce : bien distinguer les notifications de non-remise

Un e-mail n’arrive pas, et le ticket mentionne au choix « Bounce », « NDR », « Mailer-Daemon » ou « message d’erreur du serveur ». Au quotidien, les administrateurs utilisent ces termes comme des synonymes, alors qu’ils désignent des choses différentes : un reject pendant la session SMTP n’est pas un e-mail de retour, une notification de retard n’est pas une erreur de remise, et un accusé de lecture n’a absolument rien à voir avec la non-remise. Distinguer clairement ces termes permet d’identifier plus rapidement la cause, car chaque type de notification indique quelque chose de différent sur l’endroit du chemin de transport où se situe le problème et sur la personne qui peut le résoudre.

## DSN : le terme générique des RFC

Le terme formel générique est Delivery Status Notification (DSN), défini dans les RFC 3461 à 3464. Une DSN est un e-mail généré automatiquement qui informe l’expéditeur du statut de remise de son message. Point essentiel : une DSN ne signale pas uniquement les échecs. Le champ `Action` de la partie lisible par machine connaît cinq valeurs :

| Action | Signification |
|---|---|
| `failed` | La remise a définitivement échoué ; l’e-mail ne sera plus réessayé |
| `delayed` | La remise est retardée ; le serveur continue d’essayer |
| `delivered` | Remis avec succès (confirmation de remise, uniquement sur demande explicite) |
| `relayed` | Transmis à un serveur qui ne génère pas lui-même de DSN |
| `expanded` | Transmis à une liste de diffusion et réparti |

La notification de non-remise n’est donc qu’un cas particulier : une DSN avec `Action: failed`. Microsoft appelle précisément ce cas particulier un Non-Delivery Report (NDR). Le terme NDR vient de l’univers Exchange, mais est désormais utilisé au-delà des éditeurs. Pour être précis : chaque NDR est une DSN, mais chaque DSN n’est pas un NDR.

La notification de retard (`Action: delayed`) mérite une attention particulière, car elle est régulièrement interprétée à tort comme une erreur de remise dans le support. Un objet typique est « Delivery delayed » ou « Remise retardée ». L’e-mail se trouve alors encore dans la file d’attente du serveur expéditeur, qui continue à essayer, généralement pendant un à deux jours. Le NDR définitif n’est envoyé qu’à l’expiration de la durée de vie dans la file d’attente. Un utilisateur qui renvoie l’e-mail après une notification de retard crée des doublons dès que le système de destination est à nouveau joignable.

## Reject ou bounce : la distinction la plus importante

Avant d’aborder les autres termes, il convient d’expliquer l’aiguillage technique central, car c’est lui qui détermine quel serveur génère une notification.

**Reject (refus pendant la session) :** Le serveur destinataire refuse l’e-mail dès la session SMTP, avec un code de réponse 5xx à `RCPT TO` ou après `DATA`. Il n’accepte jamais l’e-mail et ne génère lui-même aucun e-mail de retour. L’obligation d’informer l’expéditeur incombe au serveur qui soumet le message : le MTA expéditeur voit la réponse 5xx et génère ensuite le NDR pour son utilisateur local. Dans ce cas, le NDR lu par l’utilisateur provient de son propre serveur, mais cite le message d’erreur du serveur distant.

**Bounce (acceptation suivie d’un retour ultérieur) :** Le serveur destinataire accepte l’e-mail avec `250 OK`, puis constate seulement ensuite qu’il ne peut pas le remettre, par exemple parce que la boîte aux lettres n’existe pas, que le quota est plein ou qu’un serveur en aval le refuse. Il est alors responsable du message et doit lui-même envoyer une DSN à l’expéditeur. Cet e-mail de retour ultérieur est le bounce au sens strict.

Pour le dépannage, cette différence est directement exploitable : si le NDR indique le serveur local comme système générateur, l’e-mail a été refusé durant la session ou n’est jamais sorti. Si un serveur tiers est l’expéditeur de la notification, le côté distant a d’abord accepté l’e-mail, et le problème se situe après son point d’acceptation, invisible pour l’expéditeur.

Deux autres termes liés aux bounces viennent du marketing et ne figurent dans aucune RFC : hard bounce pour les erreurs définitives (5xx, `Action: failed`) et soft bounce pour les erreurs temporaires (4xx, `Action: delayed`). Pour les plateformes d’envoi de masse, cette distinction est essentielle, car les hard bounces devraient entraîner un nettoyage immédiat des listes. Techniquement, il s’agit des mêmes mécanismes que ci-dessus.

## Vue d’ensemble des termes

| Terme | Ce que c’est | Qui génère la notification | Standard |
|---|---|---|---|
| DSN | Terme générique : notification de statut de remise (failed, delayed, delivered, relayed, expanded) | Le MTA qui est responsable de l’e-mail | RFC 3461 à 3464 |
| NDR | DSN avec `Action: failed` ; terme Microsoft pour la notification de non-remise | MTA expéditeur (après reject) ou MTA destinataire (après acceptation) | RFC 3464, documentation Microsoft |
| Reject | Refus 5xx pendant la session SMTP en cours ; pas d’e-mail distinct | Personne ; le MTA expéditeur en fait un NDR | RFC 5321 |
| Bounce | E-mail de retour après une acceptation déjà effectuée | MTA destinataire | RFC 5321, RFC 3464 |
| Hard/Soft Bounce | Classification marketing : définitif (5xx) contre temporaire (4xx) | comme pour le bounce | aucune RFC |
| Notification de retard | DSN avec `Action: delayed` ; l’e-mail est encore dans la file d’attente | MTA expéditeur ou relais | RFC 3464 |
| Backscatter | NDR envoyés à des adresses d’expéditeur falsifiées, généralement déclenchés par du spam | MTA destinataires mal configurés | aucune RFC, terme anti-abus |
| MDN / accusé de lecture | Notification concernant l’affichage ou la suppression par le destinataire | Client de messagerie du destinataire | RFC 8098 |
| Réponse d’absence | Réponse automatique d’une boîte aux lettres atteinte | Serveur de boîte aux lettres ou de groupware | RFC 3834 |

## Structure d’une DSN

Les DSN conformes au standard utilisent le type MIME `multipart/report; report-type=delivery-status` avec trois parties : une explication lisible par l’humain, une partie lisible par machine de type `message/delivery-status` et, facultativement, le message d’origine ou ses en-têtes. La partie lisible par machine est la plus précieuse pour le diagnostic, car ses champs sont normalisés :

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
| `Final-Recipient` | L’adresse destinataire à laquelle se rapporte le statut (un bloc par destinataire) |
| `Action` | L’une des cinq valeurs de statut (failed, delayed, delivered, relayed, expanded) |
| `Status` | Enhanced Status Code selon RFC 3463, p. ex. `5.1.1` |
| `Remote-MTA` | Le serveur distant avec lequel le MTA générateur a communiqué |
| `Diagnostic-Code` | La réponse SMTP littérale du serveur distant ; souvent la ligne la plus révélatrice |

Une DSN est toujours envoyée avec un expéditeur d’enveloppe vide (`MAIL FROM:<>`). Ce n’est pas une négligence, mais une prescription de la RFC 5321 : l’expéditeur vide empêche qu’une nouvelle DSN soit générée pour une DSN non distribuable et que deux serveurs s’échangent indéfiniment des messages d’erreur. Il en découle une règle de configuration : un système de messagerie ne doit pas refuser globalement les e-mails dont l’expéditeur d’enveloppe est vide, faute de quoi les notifications de non-remise légitimes n’atteignent jamais ses utilisateurs.

Exchange et Exchange Online respectent le standard pour le format, mais présentent le contenu à leur manière : l’utilisateur voit une page préparée avec une explication en texte clair, suivie de « Generating server » (correspond à `Reporting-MTA`) et des informations brutes. Pour le diagnostic, il vaut toujours la peine de consulter cette partie technique inférieure.

## Lire les Enhanced Status Codes

Le champ `Status` et, le plus souvent, le champ `Diagnostic-Code` contiennent un code en trois parties selon la RFC 3463 : classe.sujet.détail. La classe indique le caractère définitif, le sujet et le détail indiquent la cause :

| Plage de codes | Signification |
|---|---|
| `2.x.x` | Succès (uniquement dans les confirmations de remise) |
| `4.x.x` | Erreur temporaire ; le serveur réessaie |
| `5.x.x` | Erreur définitive ; aucune autre tentative |
| `x.1.x` | Problème d’adressage, p. ex. `5.1.1` destinataire inconnu, `5.1.10` domaine sans MX |
| `x.2.x` | Problème de boîte aux lettres, p. ex. `5.2.2` boîte aux lettres pleine, `5.2.3` message trop volumineux pour la boîte aux lettres |
| `x.3.x` | Problème du système de destination, p. ex. `4.3.2` le système n’accepte actuellement rien |
| `x.4.x` | Réseau et routage, p. ex. `4.4.1` aucune réponse, `4.4.7` durée de vie dans la file d’attente expirée |
| `x.5.x` | Erreur de protocole dans le dialogue SMTP |
| `x.7.x` | Politique et sécurité, p. ex. `5.7.1` relais refusé ou rejet par politique, `5.7.26` authentification manquante (SPF/DKIM/DMARC) |

Le code de réponse SMTP classique à trois chiffres (par exemple `550`) et l’Enhanced Status Code figurent souvent ensemble sur une ligne : `550 5.7.1 ...`. Le code à trois chiffres contrôle le comportement protocolaire du serveur expéditeur, tandis que le code étendu fournit l’information de diagnostic. En cas de contradiction entre le code et le texte libre, le texte libre du serveur distant constitue souvent la source la plus précise, car de nombreux systèmes définissent des codes génériques et indiquent la cause réelle dans le commentaire, y compris des identifiants de référence pour le support du côté distant.

À noter : les rejets `5.7.x` par les filtres de réputation et de contenu restent souvent volontairement peu explicites. Ne regarder que le code dans ce cas mène au mauvais endroit ; la liste de blocage ou le fabricant du filtre indiqué dans le texte libre permet d’atteindre plus vite la solution.

## Backscatter : la forme nuisible de bounce

Le backscatter se produit lorsqu’un serveur accepte d’abord du spam avec un expéditeur falsifié, puis envoie un NDR à l’adresse falsifiée. Le NDR atteint alors une personne non impliquée, dont l’adresse a été utilisée abusivement par le spammeur. Lors de grandes vagues de spam, les victimes reçoivent des milliers de NDR pour des e-mails qu’elles n’ont jamais envoyés, et les serveurs qui génèrent ces NDR en masse finissent eux-mêmes sur des listes de blocage (par exemple la liste Backscatterer de UCEPROTECT).

La solution découle directement de la distinction reject-bounce : tout ce qui peut être refusé doit l’être pendant la session SMTP, et non être renvoyé après acceptation. Concrètement, cela signifie la validation des destinataires au point d’acceptation le plus externe (la passerelle Edge connaît les adresses valides, par synchronisation avec l’annuaire ou Recipient Callout, au lieu de tout accepter puis d’échouer en interne), le refus du spam et des logiciels malveillants pendant la session plutôt que par des NDR de quarantaine, ainsi que l’absence de NDR pour les messages classifiés comme spam. Un reject ne génère pas de backscatter, car avec un expéditeur falsifié, la réponse 5xx est reçue par le serveur du spammeur, qui n’en crée pas de NDR à destination de la victime.

## Ce qui n’est pas une notification de non-remise

Trois types de notifications arrivent régulièrement ensemble dans les tickets, mais n’en font pas partie :

**MDN (Message Disposition Notification, RFC 8098) :** L’accusé de lecture. Il n’est pas généré par le système de transport, mais par le client de messagerie du destinataire, et signale l’affichage ou la suppression du message, non sa remise. Le type MIME est donc `multipart/report; report-type=disposition-notification`. L’absence d’accusé de lecture ne dit rien sur la remise ; la plupart des clients demandent l’autorisation à l’utilisateur ou suppriment entièrement les MDN.

**Réponses d’absence et répondeurs automatiques (RFC 3834) :** Une réponse d’absence prouve l’inverse d’un échec de remise, car elle suppose que l’e-mail a atteint la boîte aux lettres. Dans les descriptions de tickets (« je reçois une réponse automatique, mon e-mail arrive-t-il ? »), il est utile de demander quelle notification est exactement concernée.

**Notifications de quarantaine :** Des notifications telles que le récapitulatif de quarantaine de Microsoft 365 ou d’une passerelle informent le destinataire des e-mails retenus. Elles sont adressées au destinataire, non à l’expéditeur, et ne suivent aucun standard DSN. Dans ce scénario, l’expéditeur ne reçoit souvent rien du tout, ce qui explique les cas où un e-mail « disparaît sans message d’erreur ».

## Liste de contrôle pour le diagnostic

Lorsqu’une notification est disponible, procédez dans cet ordre :

1. De quel type s’agit-il : NDR (`Action: failed`), retard (`Action: delayed`), MDN, répondeur automatique ou notification de quarantaine ? En cas de notification de retard : attendre, ne pas renvoyer.
2. Qui a généré la notification (`Reporting-MTA` ou « Generating server ») ? Le serveur local indique un reject ou une erreur interne ; un serveur tiers indique une acceptation suivie d’un échec ultérieur du côté distant.
3. Que disent le statut et le code de diagnostic ? La classe 4 par rapport à la classe 5 distingue le temporaire du définitif, le sujet (`x.1` adresse, `x.2` boîte aux lettres, `x.4` réseau, `x.7` politique) restreint la cause, et le texte libre du serveur distant fournit les détails.
4. Aucune notification ne manque alors que l’e-mail n’arrive pas : vérifiez le suivi des messages sur votre propre système et pensez à une quarantaine ou à un filtrage silencieux du côté distant.

Les articles sur le [suivi des messages et le diagnostic SMTP dans le générateur de commandes](https://rafaelpfister.ch/tools/command-builder) ainsi que sur l’[analyseur d’en-têtes d’e-mail](https://rafaelpfister.ch/tools/mail-header-analyzer) montrent comment reproduire ensuite de manière ciblée les différents chemins de remise et analyser le chemin de transport d’un e-mail arrivé.

## Sources

1.  [RFC 3461: SMTP Service Extension for Delivery Status Notifications](https://www.rfc-editor.org/rfc/rfc3461): Extension SMTP permettant aux expéditeurs de demander et de contrôler les DSN.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): Définition des codes de statut en trois parties (classe.sujet.détail).

3.  [RFC 3464: An Extensible Message Format for Delivery Status Notifications](https://www.rfc-editor.org/rfc/rfc3464): Structure de la DSN en tant que multipart/report, champs tels que Action, Status et Diagnostic-Code.

4.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Règles de base concernant les codes de réponse, le transfert de responsabilité lors de l’acceptation et l’expéditeur d’enveloppe vide pour les messages d’erreur.

5.  [RFC 8098: Message Disposition Notification](https://www.rfc-editor.org/rfc/rfc8098): Standard pour les accusés de lecture, afin de les distinguer des DSN.

6.  [RFC 3834: Recommendations for Automatic Responses to Electronic Mail](https://www.rfc-editor.org/rfc/rfc3834): Règles pour les répondeurs automatiques tels que les réponses d’absence.

7.  [Microsoft Learn: Email non-delivery reports and SMTP errors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/non-delivery-reports-in-exchange-online/non-delivery-reports-in-exchange-online): Structure des NDR et liste des codes du point de vue d’Exchange Online.

8.  [UCEPROTECT Backscatterer](https://www.backscatterer.org/): Liste de blocage pour les systèmes qui génèrent du backscatter ; explique les critères d’inscription.
