---
title: "Renouvellement de la plateforme HIN 2026 : Access Gateway, Client et échéances jusqu’au 26 octobre"
navTitle: "Access Gateway 2026"
description: "Ouverture du pare-feu jusqu’au 31 août, Access Gateway version 4 dès septembre, points de terminaison SAML jusqu’au 12 octobre, jetons matériels et HIN Client 4.0 jusqu’au 26 octobre. La passerelle de messagerie n’est pas concernée et sera remplacée séparément."
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
translationSourceHash: cd34e6297afca9491bd965e40dbaf55b920296e62e57726754824474fcb27f0d
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:45:49.785Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/renouvellement-de-la-plateforme-hin-2026-access-gateway-client-et-echeances-jusqu-au-14
---

# Renouvellement de la plateforme HIN 2026 : Access Gateway, Client et échéances jusqu’au 26 octobre

En 2026, HIN renouvelle sa plateforme d’identité et d’accès. HIN a redéfini les échéances le 27 août 2026 : la première échéance a expiré le 31 août 2026, les anciens points de terminaison SAML ne sont plus utilisables depuis le 12 octobre 2026, et le grand changement suivra le 26 octobre 2026.

**Sont concernés le HIN Access Gateway (AGW), le HIN Client et les moyens d’authentification. La passerelle de messagerie HIN n’est pas concernée.** Elle sera également remplacée, mais dans le cadre d’un projet distinct avec son propre calendrier.

<div class="choice-row">
  <a class="choice" href="#die-fristen">
    <span class="choice__label">Votre situation</span>
    <span class="choice__title">Seul l’AGW est en service</span>
    <span class="choice__hint">Les échéances ci-dessous constituent l’intégralité des mesures à prendre. →</span>
  </a>
  <a class="choice" href="/stargate">
    <span class="choice__label">Votre situation</span>
    <span class="choice__title">Besoin supplémentaire de migration de la passerelle de messagerie</span>
    <span class="choice__hint">Le remplacement par « Stargate » est alors également prévu, avec un déploiement à grande échelle dès le T3 2026. Vérification gratuite de votre environnement. →</span>
  </a>
</div>

## Les échéances

| Date | Mesure | Concernés |
|---|---|---|
| 31.08.2026 | Ouverture du pare-feu pour `idp.id.hin.ch` et `cdn.hin.ch` (443/tcp) | Exploitants AGW |
| dès septembre 2026 | Installation automatique de l’AGW version 4 (hors installations en cluster) | Exploitants AGW |
| de septembre au 26.10.2026 | Mise à jour automatique vers HIN Client 4.0 par vagues | Tous les utilisateurs du Client |
| 12.10.2026 | Anciens points de terminaison SAML plus utilisables | Fédérations, connexions au DEP |
| 26.10.2026 | Les jetons matériels et les identités de test pour l’accès au DEP expirent | Utilisateurs de jetons, intégration |
| 26.10.2026 | L’application Authenticator actuelle utilisable pour la dernière fois, nouvelle configuration dès le 27.10.2026 | Utilisateurs de l’application |
| 26.10.2026 | HIN Client 4.0 requis | Tous les utilisateurs du Client |
| dès début novembre 2026 | Connexion via le navigateur dans HIN Client | Tous les utilisateurs du Client |

## Access Gateway n’est pas la passerelle de messagerie

Les deux portent le terme Gateway dans leur nom et sont régulièrement confondus. L’Access Gateway contrôle l’accès aux applications protégées par HIN et ne touche pas au trafic de messagerie. La passerelle de messagerie se trouve sur le trajet des e-mails et chiffre les messages.

## Access Gateway : pare-feu et version 4

Jusqu’au 31 août 2026, l’AGW devait pouvoir atteindre `idp.id.hin.ch` et `cdn.hin.ch` via 443/tcp. HIN recommande de définir la règle sur les noms d’hôte plutôt que sur les adresses IP, car celles-ci peuvent changer. Il s’agit d’une modification du pare-feu, et non d’un paramètre dans la passerelle ; elle relève donc souvent du responsable réseau plutôt que de l’administrateur de la passerelle.

Depuis septembre 2026, la version 4 est installée automatiquement ; les installations en cluster sont exclues, HIN contacte directement les clients concernés. Prérequis : AGW en version 3.1.50 ou supérieure et Kerberos activé comme méthode d’authentification. Pour la connexion à Active Directory, un compte LDAP avec droits de lecture est nécessaire.

Les systèmes qui ne remplissent pas les conditions préalables ne sont pas mis à jour, et l’expérience montre que cela ne se remarque que lorsque plus personne ne peut se connecter. Il vaut donc mieux vérifier la version dès maintenant plutôt que lors d’une panne.

## SAML : nouveaux points de terminaison, moins d’attributs

```text
Föderationsdienst
  broker.hin.ch/realms/HINBroker/protocol/saml/descriptor

EPD-Zugang
  idp.id.hin.ch/auth/realms/hinid/protocol/saml/descriptor
```

Avec ce changement, les formats d’attributs et les bindings changent. L’ensemble des attributs est réduit au GLN, au nom, à la date de naissance et au sexe.

C’est à ce stade que les intégrations échouent. Toute application qui exploite d’autres attributs pour les rôles ou la séparation des mandats ne les recevra plus via les nouveaux points de terminaison ; les anciens ne sont plus utilisables depuis le 12 octobre 2026. L’erreur ne se manifeste pas comme un échec de connexion, mais comme une autorisation manquante dans le système cible.

Les identités de test pour l’accès au DEP perdent leur validité le 26 octobre 2026 et doivent être demandées à nouveau. Pour l’intégration au DEP, HIN exige des identités vérifiées par identification vidéo ; aucun environnement de test n’est encore disponible. À partir du 26 octobre 2026, des assertions sans GLN peuvent en outre arriver.

Toute organisation qui exploite une fédération exploite presque toujours aussi sa propre infrastructure de messagerie. Pour ces organisations, le renouvellement de la plateforme coïncide avec le [remplacement de la passerelle de messagerie par « Stargate »](/stargate) la même année : les deux projets sont techniquement indépendants, mais se disputent les mêmes personnes et les mêmes fenêtres de maintenance.

## Jetons, application et HIN Client 4.0

Les jetons matériels ne sont plus distribués et expirent le 26 octobre 2026. Alternatives : HIN Client, code SMS ou application Authenticator. L’application Authenticator actuelle peut être utilisée jusqu’au 26 octobre 2026 ; la nouvelle application pourra être configurée comme deuxième facteur à partir du 27 octobre 2026 dans le portail en libre-service. HIN recommande de configurer auparavant la connexion avec mot de passe et code SMS comme accès d’urgence.

Le HIN Client est automatiquement mis à jour par vagues vers la version 4.0 de septembre au 26 octobre 2026. Si le Client n’affiche pas la version 4.0 le 26 octobre, il doit être installé manuellement via `download.hin.ch`. La connexion via le navigateur suivra à partir du début novembre 2026.

Le point critique concerne les exigences système : **la version 4.0 requiert Windows 11 avec TPM ou macOS 14**, et, sur les Mac Intel, une puce de sécurité supplémentaire ; sans TPM, la connexion reste possible avec un code SMS. Les appareils plus anciens doivent être mis à jour ou remplacés au préalable. Pour une partie des cabinets, l’échéance n’est donc pas une question de logiciel, mais d’approvisionnement. Ceux qui ne s’en rendent compte que peu avant le 26 octobre devront composer avec les délais de livraison et la réinstallation du logiciel du cabinet.

## Cinq questions pour faire le point

1. Quelle version d’AGW est utilisée, et Kerberos est-il actif ?
2. Le pare-feu autorise-t-il les connexions sortantes vers `idp.id.hin.ch` ?
3. Combien de postes de travail fonctionnent encore sous Windows 10 ou macOS 13 et versions antérieures ?
4. Combien de jetons matériels sont utilisés, et vers quelle solution les personnes concernées migrent-elles ?
5. Une application exploite-t-elle des attributs HIN qui disparaîtront à l’avenir ?

Les réponses aux questions 3 et 5 déterminent l’effort requis. Le reste se règle en quelques heures et est documenté par HIN.

## Le deuxième projet : « Stargate »

Indépendamment de cela, HIN remplace la passerelle de messagerie par la nouvelle HIN Gateway, appelée en interne projet « Stargate », techniquement une approche de maillage de données avec chiffrement de bout en bout et gestion décentralisée des clés. Il ne s’agit pas d’un remplacement de l’appliance, mais d’un changement d’architecture.

L’effort se situe donc à un tout autre niveau. Le renouvellement de la plateforme exige avant tout le respect des délais pour une règle de pare-feu, une version et un remplacement d’appareil, tandis qu’avec Stargate, c’est le circuit de messagerie de production lui-même qui est remis en question : l’ensemble de règles développé au fil du temps, le matériel de clés, le traitement des destinataires sans identité HIN et la question de la solution de repli si quelque chose ne se déroule pas comme prévu. La migration se déroulant dans des créneaux réservés de quatre heures et HIN recommandant un mois de préparation, un tel rendez-vous ne tolère aucun point en suspens.

<aside class="offer-box">
  <span class="offer-box__tag">Vérification gratuite</span>
  <p><strong>Vous n’avez pas besoin de savoir où vous en êtes. C’est précisément à cela que sert la vérification.</strong> J’examine votre environnement de passerelle existant et vous indique ce qui doit être fait avant la fenêtre de migration, que vous effectuiez ensuite la migration vous-même ou que vous vous fassiez accompagner.</p>
  <a class="offer-box__cta" href="/stargate">S’inscrire maintenant</a>
</aside>

## Historique des modifications

- **25 octobre 2026 :** échéances mises à jour conformément à la redéfinition par HIN du 27 août 2026 : ouverture du pare-feu (désormais également pour cdn.hin.ch) jusqu’au 31 août, anciens points de terminaison SAML jusqu’au 12 octobre, jetons matériels, application Authenticator actuelle et HIN Client 4.0 jusqu’au 26 octobre 2026. Ajouté : prérequis TPM pour HIN Client 4.0 et connexion via navigateur à partir du début novembre 2026.

## Sources

1.  [Renouvellement de la plateforme HIN : ces adaptations techniques sont requises pour les membres HIN](https://www.hin.ch/de/blog/2026/technische-anpassungen.cfm): échéances d’août à octobre 2026 (état au 27 août 2026), points de terminaison SAML, ensemble d’attributs réduit, ouvertures du pare-feu, application Authenticator et jetons matériels.

2.  [Le nouveau HIN Client est disponible : ce qui change pour les membres HIN](https://www.hin.ch/de/blog/2026/neuer-hin-client.cfm): version 4.0, déploiement jusqu’au 26 octobre 2026, prérequis de système d’exploitation et TPM, connexion via navigateur à partir de novembre 2026.

3.  [HIN Gateway : communication sécurisée au sein de la communauté HIN](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm): remplacement de la passerelle de messagerie, architecture, modèles d’exploitation, migration dans des créneaux réservés.

4.  [Configuration du HIN Access Gateway](https://cdn.hin.ch/agw/manual/DE/4-konfiguration-des-hin-access-gateway.html): rôle de l’AGW dans la gestion des accès.

5.  [Connexion à Active Directory](https://cdn.hin.ch/agw/manual/DE/5-anbindung-active-directory.html): Kerberos et compte LDAP avec droits de lecture.

6.  [HIN AG : « De la passerelle de messagerie au maillage de données »](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): contexte de « Stargate », nœuds décentralisés, calendrier.
