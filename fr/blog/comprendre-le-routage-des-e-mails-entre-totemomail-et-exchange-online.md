---
title: "Comprendre le routage des e-mails entre totemomail et Exchange Online"
navTitle: "Apache James ↔ M365"
description: "Comment totemomail stocke et traite les messages, comment l’Apache James sous-jacent bascule entre les processeurs et ce qui importe pour une boucle de messagerie sécurisée avec Exchange Online."
date: "2026-06-17"
kategorie: "Totemomail"
timeToRead: "10 min de lecture"
themen:
  - totemomail
slug: "comprendre-le-routage-des-e-mails-entre-totemomail-et-exchange-online"
translationOf: "totemomail-m365"
translationId: article-60a86616507315fa
translationReview: automatic
translationSourceHash: d92dbf8dac2f4fe0cd6e28f70e9e5575c76b56acb4afa5c5444f9e10895b4abf
translatedAt: 2026-10-14T08:57:30.085Z
url: https://rafaelpfister.ch/fr/blog/comprendre-le-routage-des-e-mails-entre-totemomail-et-exchange-online
translationModel: gpt-5.6-terra
---

# Comprendre le routage des e-mails entre totemomail et Exchange Online

Dans une boucle de messagerie entre Exchange Online et totemomail, chaque système assume une tâche clairement délimitée. Exchange Online fournit les boîtes aux lettres. Totemomail, ou l’actuel Kiteworks Email Protection Gateway, s’occupe du chiffrement, des signatures, des politiques et des règles de routage particulières.

Pour obtenir un flux de messagerie fiable, il ne suffit pas de configurer deux connecteurs SMTP. Pour le dépannage, il faut également savoir clairement ce qui se passe dans la passerelle après l’acceptation d’un message : où se trouve-t-il ? Quelle règle est exécutée ensuite ? Et pourquoi un message peut-il attendre dans une file d’attente alors que le dialogue SMTP s’est déjà terminé avec succès ?

Cet article explique donc le modèle de traitement d’[Apache James](https://james.apache.org/), sur lequel repose totemomail. La configuration concrète du routage dépend de chaque environnement ; les processeurs, matchers, mailets et référentiels décrits constituent toutefois la base technique de toute installation.

Une règle de sécurité importante s’applique indépendamment des détails : si totemomail est la passerelle en amont, Exchange Online ne doit accepter le courrier Internet que depuis cette passerelle. Cela nécessite un connecteur partenaire restrictif. Un enregistrement MX seul ne bloque pas la voie de livraison directe. L’article [Un enregistrement MX n’est pas un pare-feu](/blog/ghost-sender-exchange-online-nebeneingang) montre comment cette entrée secondaire apparaît et comment la fermer.

## De l’entrée SMTP au traitement

La logique de traitement d’Apache James se compose de quatre éléments :

- **Matchers** vérifient des conditions et déterminent pour quels destinataires une règle s’applique.
- **Mailets** exécutent l’action proprement dite, par exemple modifier des en-têtes, chiffrer, livrer ou arrêter le traitement ultérieur.
- **Processors** regroupent les matchers et les mailets en étapes de traitement ordonnées.
- **Mail-Repositories** stockent les messages pendant leur traitement ou après une erreur.

Cette séparation est essentielle à l’analyse : le référentiel répond à la question de savoir **où** se trouve un message. Le processeur détermine **comment** il sera traité ensuite.

![Utilisation de James comme relais SMTP](../images/4CixEi383SY5WdvwMSGZ67odMU.png)

Le serveur SMTP accepte la connexion et lit le message jusqu’à la fin de la section `DATA`. James crée ensuite un objet `MailImpl`. Il contient le contenu MIME sous forme de `MimeMessage`, ainsi que les informations nécessaires au traitement : expéditeur, destinataires, état et autres attributs.

Avec un référentiel basé sur des fichiers, James stocke ces informations séparément :

- `FileStreamStore` contient le message RFC 822/MIME complet sous forme de flux d’octets.
- `FileObjectStore` contient l’objet `MailImpl` sérialisé avec l’état et les métadonnées.

Un message peut donc déjà avoir été entièrement accepté et enregistré, même si son traitement fonctionnel est encore en attente.

## Référentiels et files d’attente sous `/var/mail`

Les différents référentiels apparaissent dans le système de fichiers sous forme de répertoires. En fonctionnement normal, un message n’y reste que très brièvement. Si une file d’attente s’accumule, cela indique généralement une règle erronée, une destination inaccessible ou un service backend indisponible.

L’exemple suivant contient, en plus des files d’attente standard, des répertoires facultatifs pour une connexion HIN. HIN fournit l’espace de communication sécurisé du secteur suisse de la santé.

> Si vous avez besoin d’aide pour connecter HIN-Mailgateway ou pour migrer vers la nouvelle solution HIN-Stargate, vous trouverez les experts compétents chez [adeptio](https://adeptio.ch/).  
>   
> **adeptio** est partenaire officiel de [Health Info Net AG](https://www.hin.ch/de/index.cfm) et dispose à ce titre de contacts directs auprès du fabricant.  
> [➜ Réservez encore aujourd’hui un rendez-vous.](/kontakt)

```text
Root-Folder:
~/mailer/apps/james/var/mail

├── spool/
│   → Eingehende Mails (initiale Queue, noch nicht verarbeitet)
│
├── incoming/
│   → Mails, die als intern zuzustellen erkannt wurden (Standardfolder)
│
├── incomingHIN/
│   → Eingehende Mails für HIN-Netzwerk (Optional)
│
├── outgoing/
│   → Normale ausgehende Mails (Standardfolder)
│
├── outgoingHIN/
│   → Ausgehende Mails über HIN-Netzwerk (Optional)
│
├── outgoingNotifications/
│   → System- oder Zertifikatsbenachrichtigungen
│
├── error/
│   → Fehlgeschlagene Mails (z. B. Policy, Encryption, Routing)
│
├── DBUnavailable/
│   → Mails, die wegen Backend-/DB-Problemen nicht verarbeitet werden konnten
```

## Structure d’un message dans le système de fichiers

Chaque message enregistré se compose de deux fichiers.

### `FileStreamStore`: contenu du message

Le fichier `*.FileStreamStore` contient le message RFC 822/MIME complet. Avec `cat`, les en-têtes et le corps sont lisibles :

```text
From:
To:
Subject:
...
Body
```

Le format de message sous-jacent est décrit dans la [RFC 822](https://datatracker.ietf.org/doc/html/rfc822).

### `FileObjectStore`: état et métadonnées

Le fichier `*.FileObjectStore` est un objet Java sérialisé de type `org.apache.james.core.MailImpl`. Ses champs comprennent :

```text
attributes: HashMap
errorMessage: String
lastUpdated: Date
message: MimeMessage
name: String
state: String
recipients: Collection
remoteAddr
remoteHost
sender
```

La [documentation API de `MailImpl`](https://james.apache.org/server/3/apidocs/org/apache/james/core/MailImpl.html) décrit en détail le modèle d’objet.

## L’état sélectionne le processeur suivant

La structure des répertoires indique seulement le référentiel. L’état de traitement réel figure dans le champ `state` du `FileObjectStore`. Sa valeur renvoie à l’attribut `name` d’un processeur.

Après chaque mailet, le SpoolManager vérifie cet état :

1. Si l’état reste inchangé, la paire matcher-mailet suivante est exécutée dans le même processeur.
2. Si un mailet modifie l’état, James quitte le processeur actuel et passe au processeur du même nom.
3. L’état spécial `ghost` termine entièrement le traitement.

Les processeurs obligatoires `root` et `error` ont des tâches fixes. Les nouveaux messages démarrent dans `root` ; les erreurs internes et les mailets configurés en conséquence redirigent vers `error`. En revanche, l’ordre des éléments `<processor>` dans le fichier XML ne détermine **pas** l’ordre d’exécution.

## Structure des processeurs dans `totemomail_config.xml`

Avant toute modification, la configuration `totemomail_config.xml` actuelle doit être exportée et sauvegardée :

![Configuration / Open Current / Export to File](../images/kWKIN3vramf0IAuPjzioEGV4Znw.png)

Les différents processeurs et les mailets qu’ils contiennent sont visibles dans totemomail\_config.xml. Voici à nouveau un exemple pratique :

```xml
<?xml version="1.0" encoding="UTF-8"?>
<spoolmanager>
    <multiparamformat>XML</multiparamformat>

    <processor name="addExtSender">
    <processor name="decrypt">
    <processor name="error">
    <processor name="externalDelivery">
    <processor name="externalDeliveryToHIN">
    <processor name="incoming">
    <processor name="internalDelivery">
    <processor name="internalDeliveryToHIN">
    <processor name="outgoing">
    <processor name="outgoingCheckRecipientCertificate">
    <processor name="outgoingProcessAutoGeneratedMessages">
    <processor name="outgoingProcessEncryptionTriggers">
    <processor name="outgoingProcessEncryptionTriggersRemoval">
    <processor name="outgoingProcessExceptionTriggers">
    <processor name="processIncoming">
    <processor name="processOutgoing">
    <processor name="processOutgoingCertificateExchange">
    <processor name="processOutgoingDomainEncryptionPGP">
    <processor name="processOutgoingDomainEncryptionSMIME">
    <processor name="processOutgoingNotifications">
    <processor name="root">

</spoolmanager>
```

Bien que `root` apparaisse à la fin de cet extrait, chaque nouveau message y commence. Le nom est déterminant, et non la position dans le document.

Le processeur `root` contient lui-même une liste ordonnée de paires matcher-mailet :

```xml
   <processor name="root">
      <mailet class="SimpleLogger" match="All">
         <log-message>totemomail: New Mail</log-message>
         <showSenderEmailAddress>true</showSenderEmailAddress>
         <showRecipientsEmailAddress>true</showRecipientsEmailAddress>
         <showSubject>false</showSubject>
      </mailet>
      <mailet class="ToRepository" match="RelayLimit?Limit=20">
         <repositoryPath>file://var/mail/error</repositoryPath>
         <passThrough>false</passThrough>
         <notifySender>false</notifySender>
         <takeSenderInfoFrom>SMTP</takeSenderInfoFrom>
      </mailet>
      <mailet class="ToProcessor" match="HostIsLocal?includeSubdomains=no">
         <processor>incoming</processor>
      </mailet>
      <mailet class="ToProcessor" match="All">
         <processor>outgoing</processor>
      </mailet>
   </processor>
```

Le fichier XML configure les classes, mais ne les implémente pas. `SimpleLogger` est par exemple une classe fournie par totemomail ou Kiteworks, dont le code source n’est pas accessible dans l’appliance. L’aide de l’interface d’administration explique toutefois ses paramètres :

- `log-message` définit le texte du journal et est obligatoire.
- `showSenderEmailAddress` ajoute, sur demande, l’adresse de l’expéditeur.
- `showRecipientsEmailAddress` ajoute les adresses des destinataires.
- `showSubject` ajoute l’objet.

L’ordre **au sein** d’un processeur est impératif. Un matcher peut sélectionner aucun, tous ou seulement une partie des destinataires. En cas de sous-ensemble, James scinde le message : les destinataires correspondants passent par le mailet, tandis que les autres sont traités séparément. Si un mailet modifie ensuite l’état, le traitement passe immédiatement au processeur indiqué ; les règles restantes du processeur actuel sont ignorées.

Pour le dépannage, il en résulte une procédure fiable :

1. Déterminer le référentiel et les fichiers `FileStreamStore` / `FileObjectStore` associés.
2. Déterminer l’`state` actuel dans `FileObjectStore`.
3. Rechercher le processeur du même nom dans `totemomail_config.xml`.
4. Vérifier les matchers et les mailets dans leur ordre réel.
5. En cas de changement d’état, poursuivre dans le processeur cible.

Le flux de messagerie peut ainsi être suivi étape par étape, sans lire à tort le fichier XML de haut en bas comme un programme linéaire.

## Sources

1.  [Apache James – page du projet](https://james.apache.org/): MTA open source sur lequel totemomail ou Kiteworks EPG repose techniquement.
    
2.  [Apache James – « Spool Manager »](https://james.apache.org/server/head/spoolmanager.html): traitement des e-mails entrants, spool et files d’attente.
    
3.  [Apache James – « Spool Manager Configuration »](https://james.apache.org/server/head/spoolmanager_configuration.html): configuration des processeurs et ordre des règles.
    
4.  [Apache James – « Mailet API »](https://james.apache.org/server/head/mailet_api.html): concept de mailet et de matcher à la base des règles.
    
5.  [Apache James – « MailImpl » (documentation API)](https://james.apache.org/server/3/apidocs/org/apache/james/core/MailImpl.html): modèle d’objet Mail derrière FileStreamStore et FileObjectStore.
    
6.  [IETF – RFC 822](https://datatracker.ietf.org/doc/html/rfc822): format des messages texte Internet (en-têtes et corps).
    
7.  [Microsoft Learn – « Connectors for secure mail flow with a partner »](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-for-secure-mail-flow-with-a-partner): configuration des connecteurs pour le flux de messagerie sécurisé entre Exchange Online et la passerelle.
