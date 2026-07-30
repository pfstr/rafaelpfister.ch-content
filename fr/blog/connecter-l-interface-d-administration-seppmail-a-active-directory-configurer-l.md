---
title: "Connecter l’interface d’administration SEPPmail à Active Directory : configurer l’authentification LDAP à partir de 15.0.6"
navTitle: "Connexion LDAP admin"
description: "Depuis le firmware 15.0.6, les administrateurs de l’appliance SEPPmail peuvent s’authentifier auprès d’un serveur LDAP externe tel qu’Active Directory, y compris avec le mappage de groupes vers le groupe local admin. Configuration pas à pas sous User > Advanced Settings."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "7 min de lecture"
themen:
  - seppmail
slug: "connecter-l-interface-d-administration-seppmail-a-active-directory-configurer-l"
translationId: "article-21092a3dad6b84cb"
draft: false
translationOf: seppmail-admin-gui-ldap-authentifizierung
url: https://rafaelpfister.ch/fr/blog/connecter-l-interface-d-administration-seppmail-a-active-directory-configurer-l
translationSourceHash: bb8386d1f880934d4811eb317bcd51d47900fdd493dad90b1d7752bfc25ba55c
translationModel: gpt-5.6-terra
translatedAt: 2026-07-30T12:24:06.834Z
translationReview: automatic
---

# Connecter l’interface d’administration SEPPmail à Active Directory : configurer l’authentification LDAP à partir de 15.0.6

Jusqu’au firmware 15.0.5, l’interface d’administration de SEPPmail Secure E-Mail Gateway ne connaissait que les comptes locaux. Pour travailler proprement, il fallait créer un utilisateur local distinct pour chaque administrateur et l’ajouter au groupe admin. Cela fonctionne, mais présente les inconvénients habituels des comptes locaux : mots de passe propres à chaque appliance, absence d’offboarding centralisé et non-application des politiques de mot de passe du service d’annuaire. Le patch release 15.0.6 change la donne. L’interface d’administration authentifie désormais, à la demande, les administrateurs auprès d’un serveur LDAP externe tel qu’Active Directory et mappe les groupes AD vers les groupes locaux de l’appliance.

Les autres changements de cette version sont résumés dans l’article consacré à [SEPPmail 15.0.6 et 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1). Cet article traite uniquement de la nouvelle authentification externe.

## Ce que permet cette fonctionnalité

Selon les Extended Release Notes, la version 15.0.6 ajoute une nouvelle section **External Authentication** sous **User > Advanced Settings**. Elle permet à l’interface d’administration de s’authentifier auprès d’un serveur LDAP externe et de mapper des groupes externes, par exemple des groupes de sécurité AD, vers les groupes locaux de l’appliance.

Les utilisateurs authentifiés de manière externe apparaissent localement sur l’appliance et se comportent comme des utilisateurs locaux, à une différence près : leur mot de passe ne peut pas être modifié sur l’appliance, car il est stocké sur le serveur LDAP externe. La gestion des mots de passe est donc entièrement transférée vers l’annuaire.

Une distinction importante : l’appliance connaissait déjà une authentification externe, mais uniquement pour l’interface web GINA, configurée par Managed Domain (section External authentication dans la configuration du domaine). La nouveauté de la version 15.0.6 est que l’accès à l’interface d’administration elle-même passe également par LDAP.

Je teste encore si la passerelle de messagerie HIN a également reçu l’authentification LDAP et compléterai ensuite l’article. Les appliances HIN reposant sur le même firmware SEPPmail, je pars du principe que c’est le cas.

## Prérequis

Avant la configuration, les éléments suivants doivent être disponibles :

- **Firmware 15.0.6.1 :** La fonctionnalité arrive avec la version 15.0.6 ; en raison des deux erreurs RuleEngine de cette version, le hotfix 15.0.6.1 est directement le bon choix.
- **Un annuaire compatible LDAP :** Active Directory, OpenLDAP ou équivalent. Si les utilisateurs se trouvent uniquement dans Entra ID, qui ne parle pas lui-même LDAP, [Microsoft Entra Domain Services](/blog/microsoft-entra-domain-services-ldap-kerberos) fait le lien.
- **Un compte de liaison dans l’annuaire :** Un compte de service dédié, non privilégié et disposant d’un accès en lecture, utilisé par l’appliance pour effectuer la recherche LDAP. Pas un administrateur de domaine.
- **Un groupe AD pour les administrateurs de la passerelle :** Par exemple un groupe de sécurité SEPPmail-Admins, qui sera ensuite mappé vers le groupe local admin. L’appartenance à ce groupe déterminera alors l’accès administratif complet.

TLS est activé par défaut dans les paramètres de connexion et doit le rester ; les identifiants des administrateurs ne doivent pas circuler en clair sur le réseau. L’appliance doit pouvoir joindre le serveur LDAP sur le port configuré, généralement 636 pour LDAPS.

## Configuration sous User > Advanced Settings

La configuration se trouve dans l’interface d’administration, sous **User > Advanced Settings**, dans la section **External Authentication**, et se compose de quatre blocs.

**1. Connection Settings :** La case à cocher *Authenticate users to external LDAP server (e.g. Active Directory)* active la fonctionnalité. Viennent ensuite l’adresse du serveur, le port, l’option *TLS required*, ainsi que le Bind DN et le Bind Password du compte de service.

**2. User Attributes :** Cette section définit la manière dont l’appliance trouve les objets utilisateurs : la LDAP Object Class (généralement person dans Active Directory), la Search Base (l’OU ou le conteneur où se trouvent les comptes administrateurs) et l’attribut e-mail (par défaut : mail).

**3. Group Attributes :** De même, les paramètres des objets de groupe permettent à l’appliance de résoudre les appartenances aux groupes.

**4. Mapping Settings :** La partie décisive. Sous *Remote Group*, le groupe du serveur LDAP est sélectionné ; sous *Local Group*, un ou plusieurs groupes locaux vers lesquels il est mappé. Pour disposer d’un accès administratif complet, il s’agit du groupe admin ; ses membres disposent des mêmes droits que l’utilisateur standard admin. Pour différencier les droits, mappez plutôt vers des groupes restreints tels que readonly admin ou vers des groupes fonctionnels de l’appliance.

Avant l’enregistrement, il est utile d’exécuter le **Login Test** intégré : avec le nom d’utilisateur et le mot de passe d’un compte de test, il permet de vérifier la connexion, la recherche et l’authentification avant l’activation de la configuration.

## Exemples de configuration

Les valeurs suivantes doivent être adaptées à votre environnement (domaine d’exemple example.com). Les noms de champs correspondent à la section External Authentication de l’appliance.

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

Remarques concernant Active Directory : tout contrôleur de domaine accessible convient comme serveur ; dans les environnements avec plusieurs sites, il est recommandé d’utiliser un contrôleur de domaine situé sur le même site ou un alias pointant vers plusieurs contrôleurs de domaine. Le port 636 correspond à LDAPS ; le certificat du contrôleur de domaine doit donc pouvoir être validé par l’appliance. La Search Base doit être suffisamment restreinte pour inclure les comptes administrateurs, sans englober tout l’annuaire. L’attribut mail doit être renseigné dans les comptes AD.

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

Remarques concernant OpenLDAP : dans les configurations typiques, les utilisateurs sont stockés sous forme d’inetOrgPerson dans ou=people. Pour les groupes, groupOfNames est le choix fiable, car l’appartenance y est représentée par l’attribut member avec le DN complet. À l’inverse, les groupes posixGroup ne répertorient leurs membres que sous forme de memberUid (nom d’utilisateur plutôt que DN) ; la documentation n’indique pas si l’appliance peut le résoudre. Il convient donc de le vérifier avec le Login Test avant le changement. Si le serveur fonctionne uniquement avec STARTTLS sur le port 389, ce port doit être renseigné dans le champ Server ; la connexion ne doit en aucun cas s’effectuer sans chiffrement.

## Consignes d’exploitation

Trois points méritent attention avant de faire de l’authentification LDAP le seul moyen d’accès à l’appliance :

- **Conserver un accès local de secours.** Les mots de passe des utilisateurs externes sont stockés sur le serveur LDAP. Si l’annuaire est inaccessible (problème réseau, maintenance AD ou nécessité pour la passerelle de résoudre un problème sur ce même réseau), un compte administrateur local avec un mot de passe conservé en lieu sûr reste indispensable. L’utilisateur standard admin ne doit donc pas être supprimé, mais maintenu comme accès de secours documenté.
- **La MFA reste pertinente.** La version 15.0.6 a également remanié la connexion MFA : le deuxième facteur n’est plus ajouté au mot de passe, mais demandé dans un champ distinct. L’authentification externe ne remplace pas le deuxième facteur.
- **Offboarding via l’annuaire.** C’est le principal avantage de l’intégration : lorsqu’un administrateur quitte l’entreprise, il suffit de désactiver son compte AD ou de le retirer du groupe mappé. Il n’est plus nécessaire de mettre à jour les comptes locaux sur chaque appliance. Les objets utilisateurs localement visibles et authentifiés de manière externe devraient néanmoins être régulièrement comparés avec l’annuaire.

## Conclusion

L’authentification LDAP pour l’interface d’administration comble une lacune qui existait depuis longtemps sur l’appliance : les accès administrateurs peuvent désormais être gérés de manière centralisée dans l’annuaire plutôt que sur chaque appareil. Avec le champ MFA distinct, la version 15.0.6 rend ainsi la connexion à l’interface d’administration nettement plus mature en une seule version. Toute personne mettant en œuvre cette fonctionnalité devrait maintenir le mappage de groupes de manière délibérément restrictive et ne pas sacrifier l’accès local de secours.

## Sources

1.  [SEPPmail Extended Release Notes 15.0](https://downloads.seppmail.com/extrelnotes/150/ERN15.0.html): entrée sur l’authentification de l’interface d’administration avec description de la fonctionnalité, emplacement de configuration et comportement des utilisateurs authentifiés de manière externe.

2.  [Documentation SEPPmail – « User > Advanced Settings »](https://docs.seppmail.com/ch/07_mi_15_usr_04_advanced-settings.html): référence des champs de la section External Authentication (Connection, User Attributes, Group Attributes, Mapping, Login Test).

3.  [Documentation SEPPmail – « Groups »](https://docs.seppmail.com/ch/07_mi_16_groups.html): groupes prédéfinis de l’appliance ; les membres du groupe admin disposent d’un accès administratif sans restriction.

4.  [Documentation SEPPmail – « Revision History »](https://docs.seppmail.com/ch/20_revision-history.html): notes de version officielles de la version 15.0.6 avec l’entrée relative à l’authentification de l’interface d’administration auprès de serveurs LDAP externes.

5.  [SEPPmail 15.0.6 et 15.0.6.1 : correctifs de sécurité et nouvelles fonctions d’administration](/blog/seppmail-releases-15-0-6-und-15-0-6-1): aperçu de toutes les modifications des deux versions.
