---
title: "Connecter l’interface d’administration SEPPmail à Active Directory : configurer l’authentification LDAP à partir de la version 15.0.6"
navTitle: "Connexion LDAP admin"
description: "Depuis le firmware 15.0.6, les administrateurs de l’appliance SEPPmail peuvent s’authentifier auprès d’un serveur LDAP externe tel qu’Active Directory, y compris avec un mappage de groupes vers le groupe admin local. Configuration pas à pas sous User > Advanced Settings."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "7 min de lecture"
themen:
  - seppmail
slug: "connecter-l-interface-d-administration-seppmail-a-active-directory-configurer-l"
translationId: "article-21092a3dad6b84cb"
draft: false
translationOf: seppmail-admin-gui-ldap-authentifizierung
translationSourceHash: aad5af6607824c7af146d3214d622067cdb1dfe98b82358fbc7566a32256464a
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:22:57.926Z
translationReview: automatic
url: https://rafaelpfister.ch/fr/blog/connecter-l-interface-d-administration-seppmail-a-active-directory-configurer-l
---

# Connecter l’interface d’administration SEPPmail à Active Directory : configurer l’authentification LDAP à partir de la version 15.0.6

Jusqu’au firmware 15.0.5, l’interface d’administration de la passerelle de messagerie sécurisée SEPPmail ne connaissait que les comptes locaux. Pour travailler proprement, il fallait créer un utilisateur local distinct pour chaque administrateur et l’ajouter au groupe admin. Cela fonctionne, mais présente les inconvénients habituels des comptes locaux : mots de passe propres à chaque appliance, absence d’offboarding centralisé et impossibilité d’appliquer les politiques de mot de passe du service d’annuaire. Cela change avec la version corrective 15.0.6. L’interface d’administration peut, sur demande, authentifier les administrateurs auprès d’un serveur LDAP externe tel qu’Active Directory et mapper les groupes AD vers les groupes locaux de l’appliance.

Les autres modifications de la version sont résumées dans l’article consacré à [SEPPmail 15.0.6 et 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1). Il est ici uniquement question de la nouvelle authentification externe.

## Ce que permet cette fonctionnalité

Selon les Extended Release Notes, la version 15.0.6 ajoute une nouvelle section **External Authentication** sous **User > Advanced Settings**. Elle permet à l’interface d’administration de s’authentifier auprès d’un serveur LDAP externe et de mapper les groupes externes, tels que les groupes de sécurité AD, vers les groupes locaux de l’appliance.

Les utilisateurs authentifiés en externe apparaissent localement sur l’appliance et se comportent comme des utilisateurs locaux, à une différence près : leur mot de passe ne peut pas être modifié sur l’appliance, car il est stocké sur le serveur LDAP externe. La gestion des mots de passe est donc entièrement transférée vers l’annuaire.

Une distinction importante : l’appliance disposait déjà auparavant d’une authentification externe, mais uniquement pour l’interface web GINA, configurée par Managed Domain (section External authentication dans la configuration du domaine). La nouveauté de la version 15.0.6 est que l’accès à l’interface d’administration elle-même passe également par LDAP.

Je teste encore si la passerelle de messagerie HIN a elle aussi reçu la connexion LDAP et compléterai ensuite l’article. Les appliances HIN reposant sur le même firmware SEPPmail, je pars de ce principe.

## Prérequis

Avant la configuration, quatre éléments doivent être disponibles :

- **Firmware 15.0.6.1 :** La fonctionnalité arrive avec la version 15.0.6 ; en raison des deux erreurs RuleEngine de cette version, le correctif 15.0.6.1 est directement le bon choix.
- **Un annuaire compatible LDAP :** Active Directory, OpenLDAP ou équivalent. Si les utilisateurs ne se trouvent que dans Entra ID, qui ne parle pas lui-même LDAP, [Microsoft Entra Domain Services](/blog/microsoft-entra-domain-services-ldap-kerberos) fait le lien.
- **Un compte de liaison dans l’annuaire :** Un compte de service dédié, sans privilèges et disposant d’un accès en lecture, que l’appliance utilise pour effectuer la recherche LDAP. Pas un administrateur de domaine.
- **Un groupe AD pour les administrateurs de la passerelle :** Par exemple, un groupe de sécurité SEPPmail-Admins, qui sera ensuite mappé vers le groupe admin local. L’appartenance à ce groupe déterminera alors l’accès administratif complet.

TLS est activé par défaut dans les paramètres de connexion et doit le rester ; les identifiants des administrateurs ne doivent pas circuler en clair sur le réseau. L’appliance doit pouvoir atteindre le serveur LDAP sur le port configuré, généralement 636 pour LDAPS.

## Configuration sous User > Advanced Settings

La configuration se trouve dans l’interface d’administration sous **User > Advanced Settings**, dans la section **External Authentication**, et se compose de quatre blocs.

**1. Connection Settings :** La case à cocher *Authenticate users to external LDAP server (e.g. Active Directory)* active la fonctionnalité. Viennent ensuite l’adresse du serveur, le port, l’option *TLS required*, ainsi que le Bind DN et le Bind Password du compte de service.

**2. User Attributes :** Cette section définit comment l’appliance trouve les objets utilisateur : la LDAP Object Class, généralement person pour Active Directory, la Search Base, soit l’OU ou le conteneur sous lequel se trouvent les comptes administrateurs, et l’attribut d’e-mail, mail par défaut.

**3. Group Attributes :** Les informations équivalentes pour les objets de groupe, afin que l’appliance puisse résoudre les appartenances aux groupes.

**4. Mapping Settings :** La partie décisive. Sous *Remote Group*, le groupe provenant du serveur LDAP est sélectionné ; sous *Local Group*, un ou plusieurs groupes locaux auxquels il est mappé. Pour un accès administratif complet, il s’agit du groupe admin ; ses membres sont assimilés à l’utilisateur standard admin. Pour différencier les accès, mappez plutôt vers des groupes restreints tels que readonly admin ou vers des groupes fonctionnels de l’appliance.

Avant l’enregistrement, il est utile d’effectuer le **Login Test** intégré : avec le nom d’utilisateur et le mot de passe d’un compte de test, il est possible de vérifier la connexion, la recherche et l’authentification avant que la configuration ne devienne active.

## Exemples de configuration

Les valeurs suivantes doivent être adaptées à votre environnement (domaine d’exemple example.com). Les noms des champs correspondent à la section External Authentication de l’appliance.

### Active Directory

| Champ | Valeur |
|---|---|
| Server | dc01.example.com |
| Port | 636 |
| TLS required | activé |
| Bind DN | CN=svc-seppmail,OU=ServiceAccounts,DC=example,DC=com |
| Bind Password | Mot de passe du compte de service |
| User: LDAP Object Class | person |
| User: Search Base | OU=IT,DC=example,DC=com |
| User: E-Mail Attribute | mail |
| Group: LDAP Object Class | group |
| Group: Search Base | OU=Groups,DC=example,DC=com |
| Mapping: Remote Group | SEPPmail-Admins |
| Mapping: Local Group | admin |

Remarques concernant Active Directory : tout contrôleur de domaine accessible convient comme serveur ; dans les environnements comptant plusieurs sites, il est recommandé d’utiliser un contrôleur de domaine situé sur le même site ou un alias pointant vers plusieurs contrôleurs de domaine. Le port 636 correspond à LDAPS ; le certificat du contrôleur de domaine doit donc pouvoir être validé par l’appliance. La Search Base doit être définie de manière suffisamment restrictive pour contenir les comptes administrateurs, sans couvrir l’ensemble de l’annuaire. L’attribut mail doit être renseigné dans les comptes AD.

### OpenLDAP

| Champ | Valeur |
|---|---|
| Server | ldap01.example.com |
| Port | 636 |
| TLS required | activé |
| Bind DN | cn=seppmail,ou=services,dc=example,dc=com |
| Bind Password | Mot de passe du compte de service |
| User: LDAP Object Class | inetOrgPerson |
| User: Search Base | ou=people,dc=example,dc=com |
| User: E-Mail Attribute | mail |
| Group: LDAP Object Class | groupOfNames |
| Group: Search Base | ou=groups,dc=example,dc=com |
| Mapping: Remote Group | seppmail-admins |
| Mapping: Local Group | admin |

Remarques concernant OpenLDAP : dans les configurations typiques, les utilisateurs sont stockés en tant qu’inetOrgPerson sous ou=people. Pour les groupes, groupOfNames est le choix fiable, car l’appartenance y est représentée via l’attribut member avec le DN complet. Les groupes posixGroup, en revanche, ne listent leurs membres que sous forme de memberUid, c’est-à-dire le nom d’utilisateur plutôt que le DN ; la capacité de l’appliance à le résoudre n’est pas documentée et doit être vérifiée avec le Login Test avant toute bascule. Si le serveur fonctionne uniquement avec STARTTLS sur le port 389, ce port doit être indiqué dans le champ Server ; la connexion ne doit en aucun cas être non chiffrée.

## Consignes d’exploitation

Trois points méritent attention avant que la connexion LDAP ne devienne l’unique moyen d’accéder à l’appliance :

- **Conserver un accès local de secours.** Les mots de passe des utilisateurs externes sont stockés sur le serveur LDAP. Si l’annuaire est inaccessible, en raison d’un problème réseau, d’une maintenance AD ou parce que la passerelle doit justement résoudre un problème sur ce même réseau, un compte administrateur local avec un mot de passe conservé de façon sûre reste nécessaire. L’utilisateur standard admin ne doit donc pas être supprimé, mais maintenu comme accès de secours documenté.
- **La MFA reste pertinente.** La version 15.0.6 a également remanié la connexion MFA : le second facteur n’est plus ajouté au mot de passe, mais demandé dans un champ distinct. L’authentification externe ne remplace pas le second facteur.
- **Offboarding via l’annuaire.** C’est le principal avantage de l’intégration : lorsqu’un administrateur quitte l’entreprise, il suffit de désactiver son compte AD ou de le retirer du groupe mappé. La mise à jour auparavant nécessaire des comptes locaux sur chaque appliance disparaît. Les objets utilisateur authentifiés en externe, visibles localement, doivent néanmoins être périodiquement comparés avec l’annuaire.

## Conclusion

L’authentification LDAP pour l’interface d’administration comble une lacune qui existait depuis longtemps sur l’appliance : les accès administrateurs peuvent désormais être gérés de manière centralisée dans l’annuaire plutôt que par appareil. Associée au champ MFA distinct, la version 15.0.6 améliore ainsi considérablement la connexion à l’interface d’administration en une seule version. Toute personne mettant en place cette fonctionnalité devrait maintenir un mappage de groupes délibérément restrictif et conserver l’accès local de secours.

## Sources

1.  [SEPPmail Extended Release Notes 15.0](https://downloads.seppmail.com/extrelnotes/150/ERN15.0.html): entrée concernant l’authentification de l’interface d’administration avec description de la fonctionnalité, emplacement de configuration et comportement des utilisateurs authentifiés en externe.

2.  [Documentation SEPPmail – « User > Advanced Settings »](https://docs.seppmail.com/ch/07_mi_15_usr_04_advanced-settings.html): référence des champs de la section External Authentication (Connection, User Attributes, Group Attributes, Mapping, Login Test).

3.  [Documentation SEPPmail – « Groups »](https://docs.seppmail.com/ch/07_mi_16_groups.html): groupes prédéfinis de l’appliance ; les membres du groupe admin disposent d’un accès administratif illimité.

4.  [Documentation SEPPmail – « Revision History »](https://docs.seppmail.com/ch/20_revision-history.html): notes de version officielles de la version 15.0.6 avec l’entrée relative à l’authentification de l’interface d’administration auprès de serveurs LDAP externes.

5.  [SEPPmail 15.0.6 et 15.0.6.1 : correctifs de sécurité et nouvelles fonctions d’administration](/blog/seppmail-releases-15-0-6-und-15-0-6-1): aperçu de toutes les modifications des deux versions.
