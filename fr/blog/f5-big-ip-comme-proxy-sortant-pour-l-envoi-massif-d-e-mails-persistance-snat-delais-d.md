---
title: "F5 BIG-IP comme proxy sortant pour l’envoi massif d’e-mails : persistance, SNAT, délais d’attente et résolution DNS"
navTitle: "F5 envoi massif"
description: "Un envoi massif de 1 000 e-mails par minute transite par une BIG-IP comme proxy sortant vers le relais du fournisseur. Cet article explique pourquoi les sessions persistantes n’apportent rien ici, comment résoudre correctement le nom d’hôte du fournisseur via un nœud FQDN et quels réglages de SNAT, délais d’attente et limites de connexion déterminent réellement le débit."
date: "2026-08-26"
kategorie: "Répartiteur de charge"
timeToRead: "9 min de lecture"
themen:
  - loadbalancer
  - smtp-mailflow
produkte:
  - "loadbalancer"
protokolle:
  - "smtp"
  - "tcp"
  - "dns"
hauptthema: "loadbalancer"
related:
  - massenmailing-provider-wechsel-checkliste
  - mailserver-lastprofil-ermitteln
slug: "f5-big-ip-comme-proxy-sortant-pour-l-envoi-massif-d-e-mails-persistance-snat-delais-d"
featured: true
translationId: "article-ee5e63e82ffd2604"
translationOf: f5-big-ip-outbound-smtp-massenversand
translationSourceHash: 5c62d1467d3a000a8c4a3255d6f925e294d76d1b65054e91b863a5166e1a22f8
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:22:48.079Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/f5-big-ip-comme-proxy-sortant-pour-l-envoi-massif-d-e-mails-persistance-snat-delais-d
---

Un traitement de factures ou l’envoi d’une newsletter à raison d’environ 1 000 e-mails par minute quitte le réseau d’entreprise, avec une F5 BIG-IP faisant office de proxy sortant vers le point de dépôt du fournisseur. La BIG-IP ne répartit pas la charge entre plusieurs destinations, elle relaie simplement le trafic. C’est précisément cette configuration qui détermine quels réglages sont pertinents et quelles optimisations supposées restent sans effet.

## L’architecture en une phrase

Les systèmes d’envoi utilisent comme smarthost une adresse interne de serveur virtuel sur la BIG-IP ; la BIG-IP traduit les adresses source via SNAT vers une IP publique fixe et transmet chaque connexion au nom d’hôte du fournisseur. Il n’y a pas d’équilibrage de charge au sens strict sur la BIG-IP, car le pool ne contient qu’un seul membre. Cela peut sembler être une configuration triviale, mais les choix de détail (persistance, délais d’attente, type de SNAT, résolution DNS) déterminent si l’envoi fonctionne de manière stable ou s’il présente des interruptions inexpliquées sous charge.

## Les sessions persistantes sont-elles préférables ? Non, pour deux raisons

La question de la persistance de session vient du monde HTTP, où un utilisateur ayant un panier ou une session de connexion doit toujours aboutir sur le même backend. Transposé à SMTP, ce concept n’a pas de sens.

Premièrement, SMTP est sans état d’une connexion à l’autre : chaque connexion traite une ou plusieurs transactions complètes (MAIL FROM, RCPT TO, DATA) et se termine par QUIT. Il n’existe aucun état qui devrait résider sur le même système cible entre les connexions. Le système qui accepte la connexion suivante du côté du fournisseur est sans importance pour la remise.

Deuxièmement, cette BIG-IP n’a tout simplement rien à rendre persistant : le pool contient exactement un membre, l’unique adresse IP du fournisseur. Un profil de persistance ne ferait que consommer de la mémoire pour une table de persistance et nécessiterait une recherche à chaque connexion, laquelle donnerait toujours le même résultat. Le réglage correct est donc : Default Persistence Profile sur None. Même si le fournisseur publiait ultérieurement plusieurs adresses IP derrière le nom d’hôte, la persistance serait contre-productive, car elle empêcherait la répartition entre ces adresses et chargerait certaines destinations de façon déséquilibrée.

Pour le débit lors d’un envoi massif, c’est le profil de connexion de l’expéditeur qui est déterminant : peu de connexions durables avec de nombreux messages par connexion plutôt qu’une nouvelle connexion par e-mail ; voir plus bas.

## Serveur virtuel : FastL4 plutôt que Full Proxy

Pour un simple relais SMTP, un serveur virtuel Performance (couche 4) avec un profil FastL4 est le bon choix. La BIG-IP traite alors la connexion en grande partie dans le matériel ou sur le chemin accéléré, sans terminer complètement la connexion TCP. Un serveur virtuel standard en mode Full Proxy n’apporte une valeur ajoutée que si vous voulez réellement intervenir dans le flux de données sur la BIG-IP, par exemple avec un profil de sécurité SMTP ou des iRules au niveau du protocole. Pour un proxy sortant vers votre propre fournisseur contractuel, cela est inutile et ne fait qu’ajouter des sources d’erreur.

Important dans les deux cas : n’activez aucun profil qui écrit dans la connexion. Les systèmes d’envoi négocient STARTTLS directement avec le relais du fournisseur ; toute instance qui modifie ou filtre des octets compromet l’établissement de TLS.

## Résolution DNS : le nom d’hôte du fournisseur doit figurer dans le pool comme nœud FQDN

Le fournisseur a fourni un nom d’hôte, pas une adresse IP. Le réflexe naturel consistant à résoudre l’IP une fois puis à l’enregistrer statiquement comme nœud est la pire option : si le fournisseur change d’adresse (maintenance, migration, cas de reprise après sinistre), l’envoi s’arrête jusqu’à ce que quelqu’un adapte la configuration BIG-IP. C’est précisément à cela que servent les nœuds FQDN.

Un nœud FQDN enregistre le nom d’hôte au lieu de l’adresse. La BIG-IP résout elle-même le nom, crée un nœud dit éphémère pour chaque adresse retournée et les met automatiquement à jour lorsque la réponse DNS change. Par défaut, elle interroge à nouveau le nom toutes les 3 600 secondes (une heure) ; avec `interval ttl` elle se base à la place sur le TTL de la réponse DNS, ou un autre intervalle d’interrogation fixe peut être défini. Lorsque Autopopulate est activé, le pool reprend également automatiquement plusieurs enregistrements A comme membres : si le fournisseur étend ultérieurement son point de dépôt à plusieurs adresses, la BIG-IP suit sans modification de configuration.

Deux prérequis sont souvent oubliés. Premièrement, la BIG-IP doit disposer de serveurs DNS fonctionnels dans la configuration système (System, Configuration, Device, DNS) ; les nœuds FQDN utilisent les résolveurs système, et non un cache DNS d’un profil de listener. Deuxièmement, ces résolveurs doivent être effectivement accessibles depuis le contexte de gestion ou TMM, sinon le nœud reste au statut unresolved et le pool est vide.

La configuration dans tmsh se présente ainsi (les adresses et noms sont des exemples) :

```bash
tmsh create ltm node relay-provider fqdn { \
  name mail-relay.provider.example autopopulate enabled }

tmsh create ltm pool pool_provider_smtp \
  members add { relay-provider:25 } monitor tcp

tmsh create ltm snatpool snat_mailout \
  members add { 198.51.100.10 }

tmsh create ltm virtual vs_mailout_smtp \
  destination 10.0.5.10:25 ip-protocol tcp \
  profiles add { fastL4 } pool pool_provider_smtp \
  source-address-translation { type snat pool snat_mailout }
```

Les systèmes d’envoi configurent ensuite 10.0.5.10 comme smarthost. L’utilisation du port 25 ou 587 est imposée par le fournisseur ; la configuration BIG-IP est identique dans les deux cas, seul le port change.

## SNAT : adresse fixe plutôt qu’Automap

Pour le trafic e-mail sortant, l’adresse source doit être maîtrisée. SNAT Automap utilise la Floating Self-IP du VLAN sortant, laquelle peut changer discrètement lors de modifications réseau ou de restructurations de basculement. Les fournisseurs associent toutefois souvent le dépôt à une liste d’autorisation d’IP, et même sans liste d’autorisation formelle, la réputation dépend de l’adresse source. Un pool SNAT dédié avec une adresse attribuée de manière fixe transforme l’IP source en objet de configuration documenté et stable.

Concernant la capacité : une seule adresse SNAT offre environ 64 000 traductions simultanées vers une destination unique (une IP, un port), car chaque connexion reçoit son propre port source éphémère. Avec le profil de charge décrit ici, composé de quelques dizaines de connexions simultanées, cela est largement suffisant. L’épuisement des ports ne devient un sujet que lorsqu’un expéditeur mal configuré ouvre une nouvelle connexion par e-mail sans les fermer correctement ; les traductions s’accumulent alors dans un état comparable à TIME-WAIT. Ce comportement se corrige côté expéditeur, et non avec une deuxième adresse SNAT.

## Délais d’attente : la cause la plus fréquente des interruptions de connexion sous charge

Un expéditeur en masse maintient les connexions ouvertes et y envoie message après message. Des pauses peuvent survenir entre deux messages : l’expéditeur génère le bloc suivant, le relais retarde l’acceptation (tarpitting, restes de greylisting, files d’attente internes). Le délai d’inactivité du profil FastL4 est par défaut de 300 secondes. Si une pause dépasse cette durée, la BIG-IP supprime la connexion et l’expéditeur écrit dans une connexion qui n’existe plus.

Deux réglages atténuent ce problème. Premièrement, définissez le délai d’inactivité à une valeur supérieure aux pauses réalistes ; pour les envois massifs, 600 secondes constituent une valeur initiale raisonnable. La valeur ne doit pas être arbitrairement élevée, sinon des connexions orphelines s’accumulent dans la table des connexions. Deuxièmement, laissez Reset on Timeout activé dans le profil : la BIG-IP confirme alors la suppression par un reset TCP, et le MTA émetteur détecte immédiatement que la connexion est perdue, au lieu d’attendre un délai d’expiration et de ne replanifier le message qu’après plusieurs minutes.

Vous n’avez aucune influence sur les délais d’attente de l’autre côté, mais ils font partie de l’équation : si le relais du fournisseur ferme les connexions après 120 secondes d’inactivité, un délai d’attente généreux sur la BIG-IP ne sert à rien. La valeur déterminante est le plus petit délai d’attente sur l’ensemble du chemin ; en cas de doute, renseignez-vous auprès du fournisseur et utilisez cette valeur comme base de planification.

## Stratégie de connexion : peu de connexions, beaucoup de messages

Sans consignes de dépôt du fournisseur, un bref calcul est utile. 1 000 e-mails par minute représentent environ 17 par seconde. Une transaction SMTP sur une connexion déjà établie dure nettement moins d’une demi-seconde avec une latence normale. Avec 10 à 20 connexions parallèles et, par exemple, 100 messages par connexion avant que l’expéditeur ne les renouvelle, le débit cible est atteint sans difficulté. Le côté fournisseur offre généralement une capacité de connexion nettement plus importante, mais elle est partagée avec tous les autres clients. Quelques connexions durables avec de nombreuses transactions sont donc non seulement efficaces (l’établissement TCP et TLS est évité pour chaque message), mais aussi la manière la plus respectueuse d’utiliser une infrastructure tierce.

Les leviers correspondants se trouvent dans le système d’envoi, pas sur la BIG-IP : nombre maximal de messages par connexion, nombre maximal de connexions parallèles vers le smarthost, réutilisation des connexions établies. Sur la BIG-IP, l’ensemble peut être protégé avec une Connection Limit sur le membre du pool, par exemple 200 connexions simultanées : en fonctionnement normal, cette valeur n’est jamais atteinte, mais un expéditeur mal configuré qui ouvrirait soudain une connexion par e-mail ne saturerait pas sans limite le relais du fournisseur. Cette limite est un filet de sécurité, pas un outil de pilotage.

La mesure permet de vérifier si le profil de connexion défini est effectivement appliqué en pratique : les connexions par minute et les messages par connexion peuvent être analysés à partir du Message Tracking ou des logs du connecteur, comme décrit dans l’article [Déterminer le profil de charge d’un serveur de messagerie](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln). Pour un test de charge avec un profil Bulk réaliste (peu de sessions, nombreux messages par session), smtp-source du paquet Postfix est mieux adapté que les outils de charge orientés HTTP, car il génère précisément ce profil de connexion.

## Monitoring : ne surchargez pas le fournisseur avec des contrôles d’état

Un moniteur sur le membre du pool est utile afin que la BIG-IP détecte une panne du côté fournisseur et la signale correctement. La règle est la suivante : chaque contrôle d’état est une véritable connexion au fournisseur et compte pour les mêmes limites que le trafic utile. Un simple moniteur TCP à intervalle modéré (30 secondes ou plus) suffit amplement. Un moniteur SMTP complet, qui vérifie jusqu’à la bannière ou EHLO, apporte peu d’informations supplémentaires, mais génère des entrées de journal côté fournisseur et, dans le pire des cas, des questions sur la raison pour laquelle une connexion sans e-mail arrive toutes les 5 secondes.

## Liste de contrôle

| Réglage | Recommandation |
|---|---|
| Profil de persistance | None ; les sessions persistantes n’apportent rien avec SMTP, et encore moins avec un pool à un seul membre |
| Type de serveur virtuel | Performance (couche 4) avec profil FastL4, sans intervention dans le flux de données |
| Nœud cible | Nœud FQDN avec Autopopulate plutôt qu’IP statique ; serveurs DNS configurés sur la BIG-IP |
| SNAT | pool SNAT dédié avec adresse fixe connue du fournisseur ; pas d’Automap |
| Délai d’inactivité | supérieur aux pauses réelles d’envoi, valeur initiale 600 s ; Reset on Timeout activé |
| Connection Limit | comme filet de sécurité sur le membre du pool, p. ex. 200 |
| Moniteur | TCP, intervalle de 30 s ou plus ; pas de moniteur SMTP agressif |
| Configuration de l’expéditeur | peu de connexions parallèles, beaucoup de messages par connexion ; réutilisation activée |

La réponse courte à la question de départ est donc : non, les sessions persistantes ne sont pas meilleures ; dans cette configuration, elles sont inefficaces voire nuisibles. La qualité de la solution dépend de la résolution DNS du nom d’hôte du fournisseur, d’une adresse SNAT stable, de délais d’attente adaptés au profil de charge et du fait que les systèmes d’envoi déposent leurs 1 000 e-mails par minute via quelques connexions établies plutôt que via mille connexions distinctes.

## Historique des modifications

- **25 octobre 2026 :** Correction : l’intervalle par défaut de résolution de nom pour les nœuds FQDN est de 3 600 secondes, et non le TTL DNS ; la BIG-IP ne se base sur le TTL qu’avec le réglage interval ttl.

## Sources

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321) : la section 4.5.4 et le modèle transactionnel montrent que plusieurs transactions e-mail sur une connexion constituent le cas normal prévu.

2.  [K7820: Overview of SNAT features](https://my.f5.com/manage/s/article/K7820) : article de référence F5 sur SNAT, les pools SNAT et la traduction de ports par destination.

3.  [Référence tmsh : ltm node](https://clouddocs.f5.com/cli/tmsh-reference/latest/modules/ltm/ltm_node.html) : documente les options FQDN (name, autopopulate, interval) pour les nœuds et donc les membres du pool, y compris l’intervalle par défaut de 3 600 secondes et la valeur `ttl`.

4.  [smtp-source(1), Postfix](https://www.postfix.org/smtp-source.1.html) : générateur de charge qui reproduit le profil de connexion d’un expéditeur Bulk (peu de sessions, nombreux messages).

5.  [Déterminer le profil de charge d’un serveur de messagerie](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln) : guide expliquant comment analyser les connexions par minute et les messages par connexion à partir du Message Tracking.

6.  [F5 Bug ID 746657: tmsh help for FQDN node or pool member shows incorrect default for fqdn interval](https://cdn.f5.com/product/bugtracker/ID746657.html) : confirme 3 600 secondes comme valeur par défaut effective de l’intervalle d’interrogation FQDN ; l’ancienne indication TTL dans l’aide était erronée.
