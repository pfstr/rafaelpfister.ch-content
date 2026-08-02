---
title: "Renouvellement de la plateforme HIN 2026 : Access Gateway, Client et échéances jusqu’au 14 septembre"
navTitle: "Access Gateway 2026"
description: "Ouverture du pare-feu jusqu’au 14 août, Access Gateway version 4 dès le 17 août, points de terminaison SAML, jetons matériels et HIN Client jusqu’au 14 septembre. Le Mailgateway n’est pas concerné et sera remplacé séparément."
date: "2026-08-01"
kategorie: "Passerelle HIN"
timeToRead: "5 min de lecture"
themen:
  - hin-gateway
  - active-directory-entra
related:
  - hin-mailgateway-backup-disaster-recovery
  - hin-update-issue-version-15.0.5
slug: "renouvellement-de-la-plateforme-hin-2026-access-gateway-client-et-echeances-jusqu-au-14"
translationId: "article-106aa61d54408397"
translationOf: hin-plattformerneuerung-2026
url: https://rafaelpfister.ch/fr/blog/renouvellement-de-la-plateforme-hin-2026-access-gateway-client-et-echeances-jusqu-au-14
translationSourceHash: 1a174bd131b8bb29f9b1e1e793d4cf19b3f732e3c6fd779a25193f151ec8c109
translationModel: gpt-5.6-terra
translatedAt: 2026-08-02T06:15:58.214Z
translationReview: automatic
---

# Renouvellement de la plateforme HIN 2026 : Access Gateway, Client et échéances jusqu’au 14 septembre

En 2026, HIN renouvelle sa plateforme d’identité et d’accès. La première échéance arrive à expiration le 14 août 2026, suivie du grand changement le 14 septembre 2026.

**Sont concernés le HIN Access Gateway (AGW), le HIN Client et les moyens d’authentification. Le HIN Mailgateway n’est pas concerné.** Il sera également remplacé, mais dans le cadre d’un projet distinct avec son propre calendrier.

<div class="choice-row">
  <a class="choice" href="#die-fristen">
    <span class="choice__label">Votre situation</span>
    <span class="choice__title">Seul l’AGW est en service</span>
    <span class="choice__hint">Les échéances ci-dessous constituent l’ensemble des actions à entreprendre. →</span>
  </a>
  <a class="choice" href="/stargate">
    <span class="choice__label">Votre situation</span>
    <span class="choice__title">Besoin de migration supplémentaire pour le Mailgateway</span>
    <span class="choice__hint">Le remplacement par « Stargate » est alors également prévu, avec un déploiement à grande échelle à partir du T3 2026. Vérification gratuite de votre environnement. →</span>
  </a>
</div>

## Les échéances

| Date | Mesure | Concerne |
|---|---|---|
| 14.08.2026 | Autorisation dans le pare-feu pour `idp.id.hin.ch` (`185.154.38.46`, `193.168.215.45`) | Exploitants d’AGW |
| 17.08.2026 | Installation automatique de l’AGW version 4 | Exploitants d’AGW |
| à partir de la mi-août | Installation manuelle de HIN Client 4.0 recommandée | Tous les utilisateurs du Client |
| 14.09.2026 | Migration des points de terminaison SAML | Fédérations, connexions au DEP |
| 14.09.2026 | Expiration des jetons matériels et des identités de test | Utilisateurs de jetons, intégration |
| 14.09.2026 | Reconfiguration de l’application Authenticator | Utilisateurs de l’application |
| 14.09.2026 | Mise à jour forcée vers HIN Client 4.0 | Tous les utilisateurs du Client |

## Access Gateway n’est pas Mailgateway

Les deux portent le nom de Gateway et sont régulièrement confondus. L’Access Gateway contrôle l’accès aux applications protégées par HIN et n’intervient pas dans le trafic e-mail. Le Mailgateway se trouve sur le cheminement des e-mails et chiffre les messages.

## Access Gateway : pare-feu et version 4

D’ici au 14 août, l’AGW doit pouvoir atteindre `idp.id.hin.ch`. Il s’agit d’une modification du pare-feu, et non d’un réglage dans la passerelle ; elle relève donc souvent du responsable réseau plutôt que de l’administrateur de la passerelle.

À partir du 17 août, la version 4 sera installée automatiquement. Conditions requises : AGW en version 3.1.50 ou supérieure, et Kerberos activé comme méthode d’authentification. Pour la connexion à Active Directory, un compte LDAP disposant de droits de lecture est nécessaire.

Les personnes ne remplissant pas les conditions ne seront pas mises à jour, et l’expérience montre que cela ne devient apparent que lorsque plus personne ne peut se connecter. Il est donc préférable de vérifier dès maintenant la version plutôt qu’en septembre.

## SAML : nouveaux points de terminaison, moins d’attributs

```text
Föderationsdienst
  broker.hin.ch/realms/HINBroker/protocol/saml/descriptor

EPD-Zugang
  idp.id.hin.ch/auth/realms/hinid/protocol/saml/descriptor
```

Ce changement modifie les formats d’attributs et les bindings. L’ensemble des attributs est réduit au GLN, au nom, à la date de naissance et au sexe.

C’est à ce niveau que les intégrations se rompent. Toute application exploitant d’autres attributs pour les rôles ou la séparation des mandants ne les recevra plus après le 14 septembre. L’erreur ne se manifestera pas par un échec de connexion, mais par des autorisations manquantes dans le système cible.

Les identités de test expirent à la même date ; toute personne souhaitant tester le changement dans un environnement d’intégration devrait donc le faire auparavant.

Les organisations qui exploitent une fédération exploitent presque toujours aussi leur propre infrastructure e-mail. Pour elles, le renouvellement de la plateforme et le [remplacement du Mailgateway par « Stargate »](/stargate) tombent la même année : ils sont techniquement indépendants, mais concurrencent les mêmes personnes et les mêmes fenêtres de maintenance.

## Jetons, application et HIN Client 4.0

Les jetons matériels ne sont plus délivrés et expirent le 14 septembre. Alternatives : HIN Client, code SMS ou application Authenticator. L’application elle-même reste valable jusqu’au 14 septembre et devra ensuite être reconfigurée via le portail en libre-service.

Au plus tard le 14 septembre, HIN Client sera automatiquement mis à jour vers la version 4.0 ; une installation manuelle est possible à partir de la mi-août via `download.hin.ch`. La connexion s’effectue désormais via le navigateur.

Le point critique concerne les prérequis système : **la version 4.0 exige Windows 11 ou macOS 14.** Les appareils plus anciens doivent être mis à jour ou remplacés auparavant. Pour une partie des cabinets, l’échéance n’est donc pas une tâche logicielle, mais une tâche d’approvisionnement. Ceux qui ne s’en aperçoivent qu’en septembre devront composer avec les délais de livraison et la réinstallation du logiciel du cabinet.

## Cinq questions pour faire le point

1. Quelle version d’AGW est utilisée et Kerberos est-il actif ?
2. Le pare-feu autorise-t-il les connexions sortantes vers `idp.id.hin.ch` ?
3. Combien de postes de travail utilisent encore Windows 10 ou macOS 13 et versions antérieures ?
4. Combien de jetons matériels sont utilisés et vers quelle solution les personnes concernées vont-elles migrer ?
5. Une application exploite-t-elle des attributs HIN qui disparaîtront à l’avenir ?

Les réponses aux questions 3 et 5 déterminent l’effort nécessaire. Le reste peut être réalisé en quelques heures et est documenté par HIN.

## Le deuxième projet : « Stargate »

Indépendamment de cela, HIN remplace le Mailgateway par le nouveau HIN Gateway, connu en interne sous le nom de projet « Stargate », une approche Data Mesh sur le plan technique, avec chiffrement de bout en bout et gestion décentralisée des clés. Il ne s’agit pas d’un remplacement de l’appliance, mais d’un changement d’architecture.

L’effort se situe donc à un tout autre niveau. Le renouvellement de la plateforme requiert surtout le respect des délais concernant une règle de pare-feu, une version et un remplacement d’appareil, tandis qu’avec Stargate, c’est le cheminement productif des e-mails lui-même qui est remis en question : l’ensemble des règles élaboré au fil du temps, le matériel de clé, le traitement des destinataires sans identité HIN et la question de la solution de repli en cas de problème. Puisque la migration se déroule dans des créneaux réservés de quatre heures et que HIN recommande un mois de préparation, un tel rendez-vous ne laisse aucune place aux questions non résolues.

<aside class="offer-box">
  <span class="offer-box__tag">Vérification gratuite</span>
  <p><strong>Vous n’avez pas besoin de savoir où vous en êtes. C’est précisément à cela que sert cette vérification.</strong> J’examine votre environnement de passerelle existant et vous indique ce qui doit être fait avant la fenêtre de migration, que vous effectuiez ensuite vous-même la migration ou que vous choisissiez un accompagnement.</p>
  <a class="offer-box__cta" href="/stargate">S’inscrire maintenant</a>
</aside>

## Sources

1.  [Renouvellement de la plateforme HIN : ces adaptations techniques sont nécessaires pour les membres HIN](https://www.hin.ch/de/blog/2026/technische-anpassungen.cfm): échéances en août et septembre, points de terminaison SAML, ensemble d’attributs réduit, autorisations de pare-feu.

2.  [Le nouveau HIN Client est disponible : ce qui change pour les membres HIN](https://www.hin.ch/de/blog/2026/neuer-hin-client.cfm): version 4.0, prérequis du système d’exploitation, connexion basée sur le navigateur.

3.  [HIN Gateway : communication sécurisée au sein de la communauté HIN](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm): remplacement du Mailgateway, architecture, modèles d’exploitation, migration dans des créneaux réservés.

4.  [Configuration du HIN Access Gateway](https://cdn.hin.ch/agw/manual/DE/4-konfiguration-des-hin-access-gateway.html): rôle de l’AGW dans la gestion des accès.

5.  [Connexion à Active Directory](https://cdn.hin.ch/agw/manual/DE/5-anbindung-active-directory.html): Kerberos et le compte LDAP avec droits de lecture.

6.  [HIN AG : « Du Mailgateway au Data Mesh »](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): contexte de « Stargate », nœuds décentralisés, calendrier.
