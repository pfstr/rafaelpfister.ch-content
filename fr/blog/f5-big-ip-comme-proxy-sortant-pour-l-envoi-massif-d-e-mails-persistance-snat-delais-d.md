---
title: "F5 BIG-IP comme proxy sortant pour l’envoi massif d’e-mails : persistance, SNAT, délais d’expiration et résolution DNS"
navTitle: "F5 envoi massif"
description: "Un envoi massif de 1 000 e-mails par minute passe par une BIG-IP en tant que proxy sortant vers le relais du fournisseur. Cet article explique pourquoi les sessions persistantes n’apportent rien ici, comment résoudre correctement le nom d’hôte du fournisseur avec un nœud FQDN et quels réglages de SNAT, de délais d’expiration et de limites de connexion déterminent réellement le débit."
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
aiPrompt: |
  Du bist mein Netzwerk- und Mailflow-Assistent. Wir versenden Massenmails über eine F5 BIG-IP als ausgehenden Proxy zu einem Provider-Relay. Hilf mir, die BIG-IP-Konfiguration nach diesem Artikel zu prüfen: 1. Frage mich nach Versandrate, Anzahl paralleler Verbindungen und Nachrichten pro Verbindung. 2. Frage nach Virtual-Server-Typ, Persistenzprofil, Idle-Timeout und SNAT-Konfiguration. 3. Prüfe, ob der Provider-Hostname als FQDN-Node mit Autopopulate hinterlegt ist und ob DNS-Server auf der BIG-IP konfiguriert sind. 4. Nenne mir konkrete Abweichungen von den Empfehlungen aus dem Artikel und begründe jede Änderung.
translationOf: f5-big-ip-outbound-smtp-massenversand
url: https://rafaelpfister.ch/fr/blog/f5-big-ip-comme-proxy-sortant-pour-l-envoi-massif-d-e-mails-persistance-snat-delais-d
translationSourceHash: 218c4d189dd18000d6db2ead4b2106f8be858169c9d7b234e4f9320ac802fd46
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:28:29.322Z
translationReview: required
---

Un traitement de factures ou l’envoi d’une newsletter d’environ 1 000 e-mails par minute quitte le réseau de l’entreprise, avec une F5 BIG-IP placée entre les deux comme proxy sortant vers le point de dépôt du fournisseur. La BIG-IP ne répartit pas le trafic entre plusieurs destinations, elle le transmet. C’est précisément cette configuration qui détermine quels réglages sont pertinents et quelles optimisations supposées restent sans effet.

## L’architecture en une phrase

Les systèmes d’envoi utilisent comme smarthost une adresse interne de serveur virtuel sur la BIG-IP ; la BIG-IP traduit les adresses source via SNAT vers une IP publique fixe et transmet chaque connexion au nom d’hôte du fournisseur. Il n’y a pas de répartition de charge à proprement parler sur la BIG-IP, car le pool ne comporte qu’un seul membre. Cela semble être une configuration triviale, mais les décisions de détail (persistance, délais d’expiration, type de SNAT, résolution DNS) déterminent si l’envoi fonctionne de manière stable ou s’il présente des coupures inexplicables sous charge.

## Les sessions persistantes sont-elles préférables ? Non, pour deux raisons

La question de la persistance de session vient du monde HTTP, où un utilisateur avec un panier ou une session de connexion doit toujours aboutir sur le même backend. Transposé à SMTP, ce concept n’a pas de sens.

Premièrement, SMTP est achevé sans état par connexion : chaque connexion traite une ou plusieurs transactions complètes (MAIL FROM, RCPT TO, DATA) et se termine par QUIT. Aucun état ne doit résider sur le même système cible au-delà des connexions. Le système côté fournisseur qui accepte la connexion suivante n’a aucune importance pour la livraison.

Deuxièmement, il n’y a tout simplement rien à rendre persistant sur cette BIG-IP : le pool contient exactement un membre, l’unique adresse IP du fournisseur. Un profil de persistance ne ferait que consommer de la mémoire pour une table de persistance et nécessiterait une recherche à chaque connexion, donnant toujours le même résultat. Le réglage correct est donc : Default Persistence Profile sur None. Même si le fournisseur publiait ultérieurement plusieurs adresses IP derrière le nom d’hôte, la persistance serait contre-productive, car elle empêcherait la répartition sur ces adresses et chargerait certains objectifs de manière déséquilibrée.

Pour le débit lors d’un envoi massif, le facteur décisif est le profil de connexion de l’expéditeur : peu de connexions durables contenant de nombreux messages par connexion plutôt qu’une nouvelle connexion par e-mail ; davantage à ce sujet ci-dessous.

## Serveur virtuel : FastL4 plutôt que Full Proxy

Pour la simple transmission de SMTP, un serveur virtuel Performance (couche 4) avec un profil FastL4 est le bon choix. La BIG-IP traite alors la connexion en grande partie dans le matériel ou dans le chemin accéléré, sans terminer complètement la connexion TCP. Un serveur virtuel standard en mode Full Proxy n’apporte une valeur ajoutée que si vous souhaitez réellement intervenir dans le flux de données sur la BIG-IP, par exemple avec un profil de sécurité SMTP ou des iRules au niveau protocolaire. Pour un proxy sortant vers votre propre fournisseur sous contrat, cela est inutile et ne fait qu’ajouter des sources d’erreur.

Important dans les deux cas : n’activez aucun profil qui écrit dans la connexion. Les systèmes d’envoi négocient STARTTLS directement avec le relais du fournisseur ; toute instance qui modifie ou filtre les octets met en danger l’établissement de TLS.

## Résolution DNS : le nom d’hôte du fournisseur doit figurer comme nœud FQDN dans le pool

Le fournisseur a livré un nom d’hôte, et non une adresse IP. Le réflexe évident consistant à résoudre l’IP une fois et à l’enregistrer comme nœud statique est la pire option : si le fournisseur change d’adresse (maintenance, migration, cas de reprise après sinistre), l’envoi s’arrête jusqu’à ce que quelqu’un adapte la configuration BIG-IP. C’est exactement pour cela qu’il existe des nœuds FQDN.

Un nœud FQDN enregistre le nom d’hôte au lieu de l’adresse. La BIG-IP résout elle-même le nom, crée un nœud dit éphémère pour chaque adresse renvoyée et les met automatiquement à jour lorsque la réponse DNS change. Par défaut, elle interroge à nouveau le nom à l’expiration du TTL DNS ; il est également possible de définir un intervalle d’interrogation fixe. Lorsque Autopopulate est activé, le pool reprend aussi automatiquement plusieurs enregistrements A comme membres : si le fournisseur étend ultérieurement son point de dépôt à plusieurs adresses, la BIG-IP suit sans modification de configuration.

Deux prérequis sont souvent oubliés. Premièrement, la BIG-IP nécessite pour cela des serveurs DNS fonctionnels dans la configuration système (System, Configuration, Device, DNS) ; les nœuds FQDN utilisent les résolveurs système, et non un cache DNS d’un profil de listener. Deuxièmement, ces résolveurs doivent être effectivement accessibles depuis le contexte de gestion ou TMM, sans quoi le nœud reste dans l’état unresolved et le pool est vide.

La configuration dans tmsh se présente ainsi (les adresses et les noms sont des exemples) :

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

Les systèmes d’envoi configurent ensuite 10.0.5.10 comme smarthost. Le fournisseur détermine si vous utilisez le port 25 ou 587 ; la configuration BIG-IP est identique dans les deux cas, seul le port change.

## SNAT : adresse fixe plutôt qu’Automap

Pour le trafic e-mail sortant, l’adresse source doit rester sous contrôle. SNAT Automap utilise l’adresse IP flottante du VLAN sortant, qui peut changer sans que cela soit remarqué lors de modifications réseau ou de transformations liées au basculement. Les fournisseurs associent toutefois fréquemment le dépôt à une liste d’autorisation d’IP et, même sans liste d’autorisation formelle, la réputation dépend de l’adresse source. Un pool SNAT dédié avec une adresse attribuée de manière fixe fait de l’IP source un objet de configuration documenté et stable.

Concernant la capacité : une seule adresse SNAT offre environ 64 000 traductions simultanées vers une cible unique (une IP, un port), car chaque connexion reçoit son propre port source éphémère. Avec le profil de charge décrit ici, comportant quelques dizaines de connexions simultanées, cela est largement suffisant de plusieurs ordres de grandeur. L’épuisement des ports ne devient un sujet que lorsqu’un expéditeur mal configuré ouvre une nouvelle connexion par e-mail et ne la ferme pas correctement ; les traductions s’accumulent alors dans un état semblable à TIME-WAIT. Ce comportement doit être corrigé sur l’expéditeur, non avec une seconde adresse SNAT.

## Délais d’expiration : la cause la plus fréquente des coupures de connexion sous charge

Un expéditeur bulk maintient les connexions ouvertes et y transmet message après message. Des pauses peuvent survenir entre deux messages : l’expéditeur génère le bloc suivant, le relais retarde l’acceptation (tarpitting, résidus de greylisting, files d’attente internes). Le délai d’inactivité du profil FastL4 est de 300 secondes par défaut. Si une pause dépasse cette durée, la BIG-IP ferme la connexion et l’expéditeur écrit dans une connexion qui n’existe plus.

Deux réglages atténuent ce problème. Premièrement, définissez le délai d’inactivité à une valeur supérieure aux pauses réalistes ; pour l’envoi massif, 600 secondes constituent une valeur de départ raisonnable. La valeur ne doit toutefois pas être arbitrairement élevée, sinon des connexions orphelines s’accumulent dans la table des connexions. Deuxièmement, laissez Reset on Timeout activé dans le profil : la BIG-IP confirme alors la fermeture par un TCP Reset, et le MTA émetteur détecte immédiatement que la connexion est interrompue, au lieu d’attendre un timeout et de ne replanifier le message qu’après plusieurs minutes.

Vous n’avez aucune influence sur les délais d’expiration de l’autre côté, mais ils doivent être pris en compte : si le relais du fournisseur ferme les connexions après 120 secondes d’inactivité, un délai BIG-IP généreux ne sert à rien. La valeur déterminante est le délai le plus faible sur l’ensemble du chemin ; en cas de doute, renseignez-vous auprès du fournisseur et utilisez cette valeur comme base de planification.

## Stratégie de connexion : peu de connexions, beaucoup de messages

En l’absence de consignes de dépôt du fournisseur, un calcul rapide s’impose. 1 000 e-mails par minute représentent environ 17 par seconde. Une transaction SMTP sur une connexion déjà établie dure nettement moins d’une demi-seconde avec une latence normale. Avec 10 à 20 connexions parallèles et, par exemple, 100 messages par connexion avant que l’expéditeur ne les renouvelle, le débit cible est atteint confortablement. Côté fournisseur, une capacité de connexion nettement plus importante est généralement disponible, mais elle est partagée avec tous les autres clients. Quelques connexions durables avec de nombreuses transactions sont donc non seulement efficaces (l’établissement TCP et TLS est supprimé pour chaque message), mais aussi la manière la plus respectueuse d’utiliser une infrastructure tierce.

Les leviers correspondants se trouvent dans le système d’envoi, pas sur la BIG-IP : nombre maximal de messages par connexion, nombre maximal de connexions parallèles au smarthost, réutilisation des connexions établies. Sur la BIG-IP, l’ensemble peut être sécurisé avec une limite de connexions sur le membre du pool, par exemple 200 connexions simultanées : en fonctionnement normal, cette valeur n’est jamais atteinte, mais un expéditeur mal configuré qui ouvre soudain une connexion par e-mail ne peut ainsi pas inonder sans frein le relais du fournisseur. La limite est un filet de sécurité, pas un instrument de pilotage.

La mesure montre si le profil de connexion configuré est réellement atteint en pratique : les connexions par minute et les messages par connexion peuvent être évalués à partir du Message Tracking ou des journaux du connecteur, comme décrit dans l’article [Déterminer le profil de charge d’un serveur de messagerie](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln). Pour un test de charge avec un profil bulk réaliste (peu de sessions, beaucoup de messages par session), smtp-source du paquet Postfix est mieux adapté que les outils de charge orientés HTTP, car il génère précisément ce profil de connexion.

## Monitoring : ne surchargez pas le fournisseur avec des contrôles de santé

Un moniteur sur le membre du pool est utile afin que la BIG-IP détecte une défaillance côté fournisseur et la signale correctement. La règle est la suivante : chaque contrôle de santé constitue une véritable connexion au fournisseur et compte contre les mêmes limites que le trafic utile. Un simple moniteur TCP avec un intervalle modéré (30 secondes ou davantage) suffit amplement. Un moniteur SMTP complet, qui vérifie jusqu’à la bannière ou EHLO, n’apporte guère d’informations supplémentaires, mais génère des entrées de journal côté fournisseur et, dans le pire des cas, des questions sur l’origine d’une connexion sans e-mail toutes les 5 secondes.

## Liste de contrôle

| Réglage | Recommandation |
|---|---|
| Profil de persistance | None ; les sessions persistantes n’apportent rien avec SMTP, et encore moins avec un pool à un seul membre |
| Type de serveur virtuel | Performance (couche 4) avec profil FastL4, sans intervention dans le flux de données |
| Nœud cible | Nœud FQDN avec Autopopulate plutôt qu’IP statique ; serveurs DNS configurés sur la BIG-IP |
| SNAT | pool SNAT dédié avec adresse fixe connue du fournisseur ; pas d’Automap |
| Délai d’inactivité | supérieur aux pauses réelles d’envoi, valeur initiale de 600 s ; Reset on Timeout activé |
| Limite de connexions | comme filet de sécurité sur le membre du pool, par ex. 200 |
| Moniteur | TCP, intervalle de 30 s ou davantage ; pas de moniteur SMTP agressif |
| Configuration de l’expéditeur | peu de connexions parallèles, beaucoup de messages par connexion ; réutilisation activée |

La réponse courte à la question initiale est donc : non, les sessions persistantes ne sont pas meilleures ; elles sont inefficaces, voire nuisibles, dans cette configuration. La qualité de la solution dépend de la résolution DNS du nom d’hôte du fournisseur, d’une adresse SNAT stable, de délais d’expiration adaptés au profil de charge et du fait que les systèmes d’envoi déposent leurs 1 000 e-mails par minute via quelques connexions établies plutôt que par mille connexions individuelles.

## Sources

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321) : la section 4.5.4 et le modèle transactionnel montrent que plusieurs transactions e-mail sur une même connexion constituent le cas normal prévu.

2.  [K7820: Overview of SNAT features](https://my.f5.com/manage/s/article/K7820) : article de référence F5 sur SNAT, les pools SNAT et la traduction de ports par destination.

3.  [Référence tmsh : ltm node](https://clouddocs.f5.com/cli/tmsh-reference/latest/modules/ltm/ltm_node.html) : documente les options FQDN (name, autopopulate, interval) pour les nœuds, et donc pour les membres de pool.

4.  [smtp-source(1), Postfix](https://www.postfix.org/smtp-source.1.html) : générateur de charge qui reproduit le profil de connexion d’un expéditeur bulk (peu de sessions, beaucoup de messages).

5.  [Déterminer le profil de charge d’un serveur de messagerie](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln) : guide expliquant comment évaluer les connexions par minute et les messages par connexion à partir du Message Tracking.
