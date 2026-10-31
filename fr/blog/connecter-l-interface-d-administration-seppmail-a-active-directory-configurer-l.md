---
title: "Connecter l’interface d’administration SEPPmail à Active Directory : configurer l’authentification LDAP à partir de la version 15.0.6"
navTitle: "Connexion LDAP admin"
description: "Depuis le firmware 15.0.6, les administrateurs de l’appliance SEPPmail peuvent s’authentifier auprès d’un serveur LDAP externe tel qu’Active Directory, y compris avec un mappage de groupes vers le groupe local admin. Configuration pas à pas sous User > Advanced Settings."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "7 min de lecture"
themen:
  - seppmail
slug: "connecter-l-interface-d-administration-seppmail-a-active-directory-configurer-l"
translationId: "article-21092a3dad6b84cb"
draft: false
translationOf: seppmail-admin-gui-ldap-authentifizierung
translationSourceHash: b9a1bdfe75f57458eb57c2e1dcb32fdfd25e1279eb720d4cd4754d49b5aac624
translationModel: gpt-5.6-terra
translatedAt: 2026-10-31T03:23:24.404Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/connecter-l-interface-d-administration-seppmail-a-active-directory-configurer-l
---

# Connecter l’interface d’administration SEPPmail à Active Directory : configurer l’authentification LDAP à partir de la version 15.0.6

Jusqu’au firmware 15.0.5, l’interface d’administration de la passerelle de messagerie sécurisée SEPPmail ne connaissait que les comptes locaux. Pour travailler proprement, il fallait créer un utilisateur local distinct pour chaque administrateur et l’ajouter au groupe admin. Cela fonctionne, mais présente les inconvénients habituels des comptes locaux : mots de passe propres à chaque appliance, absence de processus centralisé de départ et impossibilité d’appliquer les politiques de mots de passe du service d’annuaire. Cela change avec la version corrective 15.0.6. L’interface d’administration peut désormais, sur demande, authentifier les administrateurs auprès d’un serveur LDAP externe tel qu’Active Directory et mapper les groupes AD vers les groupes locaux de l’appliance.

Les autres modifications de cette version sont résumées dans l’article [SEPPmail 15.0.6 et 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1). Cet article traite uniquement de la nouvelle authentification externe.

## Ce que permet cette fonction

Selon les Extended Release Notes, la version 15.0.6 ajoute une nouvelle section **External Authentication** sous **User > Advanced Settings**. L’interface d’administration s’authentifie ainsi auprès d’un serveur LDAP externe, et les groupes externes (par exemple des groupes de sécurité AD) sont mappés vers les groupes locaux de l’appliance.

Les utilisateurs authentifiés de manière externe apparaissent localement sur l’appliance et se comportent comme des utilisateurs locaux, à une différence près : leur mot de passe ne peut pas être modifié sur l’appliance, car il est stocké dans le serveur LDAP externe. La gestion des mots de passe est donc entièrement transférée à l’annuaire.

À distinguer : l’appliance disposait déjà d’une authentification externe, mais uniquement pour l’interface web GINA, configurée pour chaque Managed Domain (section External authentication dans la configuration du domaine). La nouveauté de la version 15.0.6 est que l’accès à l’interface d’administration elle-même passe également par LDAP.

Je teste encore si la passerelle de messagerie HIN a également reçu la connexion LDAP et compléterai ensuite l’article. Les appliances HIN reposant sur le même firmware SEPPmail, je pars de ce principe.

## Prérequis

Trois éléments doivent être disponibles avant la configuration :

- **Firmware 15.0.7 ou version ultérieure :** La fonctionnalité est arrivée avec la version 15.0.6. Depuis le 2 septembre 2026, la version corrective 15.0.7 est disponible ; elle corrige notamment un contournement de l’obligation MFA (SEC-100) et une injection de commandes dans le traitement de la configuration (CVE-2026-84830). L’appliance devrait au minimum disposer de cette version ; la branche fonctionnelle 15.1 (actuellement le correctif 15.1.0.1 du 29 septembre 2026) inclut également cette fonction.
- **Un annuaire compatible LDAP :** Active Directory, OpenLDAP ou équivalent. Si les utilisateurs ne sont présents que dans Entra ID, qui ne parle pas lui-même LDAP, [Microsoft Entra Domain Services](/blog/microsoft-entra-domain-services-ldap-kerberos) fait le lien.
- **Un compte de liaison dans l’annuaire :** Un compte de service dédié, non privilégié et doté d’un accès en lecture, que l’appliance utilise pour effectuer la recherche LDAP. Pas un administrateur de domaine.
- **Un groupe AD pour les administrateurs de la passerelle :** Par exemple un groupe de sécurité SEPPmail-Admins, qui sera ensuite mappé vers le groupe local admin. L’appartenance à ce groupe détermine alors l’accès administratif complet.

TLS est activé par défaut dans les paramètres de connexion et devrait le rester ; les identifiants des administrateurs ne doivent pas circuler en clair sur le réseau. L’appliance doit pouvoir atteindre le serveur LDAP sur le port configuré (généralement 636 pour LDAPS).

## Configuration sous User > Advanced Settings

La configuration se trouve dans l’interface d’administration, sous **User > Advanced Settings**, dans la section **External Authentication**, et comporte quatre blocs.

**1. Connection Settings :** La case à cocher *Authenticate users to external LDAP server (e.g. Active Directory)* active la fonction. Viennent ensuite l’adresse du serveur, le port, l’option *TLS required*, ainsi que le Bind DN et le Bind Password du compte de service.

**2. User Attributes :** Cette section définit comment l’appliance trouve les objets utilisateur : la LDAP Object Class (généralement person pour Active Directory), la Search Base (l’OU ou le conteneur dans lequel se trouvent les comptes administrateur) et l’attribut e-mail (par défaut : mail).

**3. Group Attributes :** De manière analogue, les paramètres des objets de groupe permettent à l’appliance de résoudre les appartenances aux groupes.

**4. Mapping Settings :** La partie décisive. Sous *Remote Group*, sélectionnez le groupe du serveur LDAP ; sous *Local Group*, un ou plusieurs groupes locaux auxquels il est mappé. Pour un accès administratif complet, il s’agit du groupe admin ; ses membres disposent des mêmes droits que l’utilisateur standard admin. Pour différencier les droits, mappez plutôt vers des groupes restreints tels que readonly admin ou vers des groupes fonctionnels de l’appliance.

Avant l’enregistrement, il est utile d’utiliser le **Login Test** intégré : avec le nom d’utilisateur et le mot de passe d’un compte de test, vous pouvez vérifier que la connexion, la recherche et l’authentification fonctionnent avant d’activer la configuration.

## Exemples de configurations

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

Remarques concernant Active Directory : tout contrôleur de domaine accessible convient comme serveur ; dans les environnements comportant plusieurs sites, il est recommandé d’utiliser un DC situé sur le même site ou un alias pointant vers plusieurs DC. Le port 636 correspond à LDAPS ; le certificat du DC doit pouvoir être validé par l’appliance. La Search Base doit être suffisamment restreinte pour contenir les comptes administrateur, sans couvrir tout l’annuaire. L’attribut mail doit être renseigné dans les comptes AD.

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

Remarques concernant OpenLDAP : dans les configurations courantes, les utilisateurs sont stockés sous forme d’inetOrgPerson dans ou=people. Pour les groupes, groupOfNames est le choix le plus fiable, car l’appartenance y est représentée via l’attribut member avec le DN complet. Les groupes posixGroup, en revanche, ne référencent leurs membres que par memberUid (nom d’utilisateur au lieu du DN) ; la capacité de l’appliance à les résoudre n’est pas documentée et devrait être vérifiée avec le Login Test avant tout changement. Si le serveur fonctionne uniquement avec STARTTLS sur le port 389, ce port doit être indiqué dans le champ Server ; la connexion ne doit en aucun cas être non chiffrée.

## Consignes d’exploitation

Trois points méritent attention avant que la connexion LDAP ne devienne le seul moyen d’accéder à l’appliance :

- **Conserver un accès local d’urgence.** Les mots de passe des utilisateurs externes sont stockés dans le serveur LDAP. Si l’annuaire est inaccessible (problème réseau, maintenance AD ou nécessité pour la passerelle de résoudre justement un problème sur ce réseau), un compte administrateur local avec un mot de passe stocké de manière sûre reste nécessaire. L’utilisateur standard admin ne doit donc pas être supprimé, mais maintenu comme accès d’urgence documenté.
- **La MFA reste pertinente.** La version 15.0.6 a également revu la connexion MFA : le second facteur n’est plus ajouté au mot de passe, mais demandé dans un champ distinct. L’authentification externe ne remplace pas le second facteur. Une vulnérabilité permettant, dans certaines conditions, un accès avant la finalisation de toutes les étapes MFA n’est corrigée qu’à partir de la version 15.0.7.
- **Départs gérés via l’annuaire.** C’est le véritable avantage de l’intégration : lorsqu’un administrateur quitte l’entreprise, il suffit de désactiver le compte AD ou de le retirer du groupe mappé. La maintenance jusque-là nécessaire des comptes locaux sur chaque appliance disparaît. Les objets utilisateur authentifiés de manière externe et visibles localement devraient néanmoins être comparés périodiquement avec l’annuaire.

## Conclusion

L’authentification LDAP pour l’interface d’administration comble une lacune qui existait depuis longtemps sur l’appliance : les accès administrateur peuvent désormais être gérés de manière centralisée dans l’annuaire plutôt que pour chaque appareil. Associée au champ MFA distinct, la version 15.0.6 améliore donc considérablement la connexion à l’interface d’administration en une seule version. Lors de l’introduction de cette fonction, le mappage des groupes devrait être maintenu volontairement restrictif et l’accès local d’urgence conservé.

## Historique des modifications

- **25 octobre 2026 :** Version minimale recommandée relevée de 15.0.6.1 à 15.0.7, qui corrige un contournement de l’obligation MFA (SEC-100) et CVE-2026-84830. Ajout : la branche fonctionnelle 15.1 (actuellement 15.1.0.1) inclut également cette fonction.

## Sources

1.  [SEPPmail Extended Release Notes 15.0](https://downloads.seppmail.com/extrelnotes/150/ERN15.0.html): entrée sur l’authentification de l’interface d’administration avec description de la fonction, emplacement de configuration et comportement des utilisateurs authentifiés de manière externe ; correctifs de sécurité de la version 15.0.7 (SEC-100, CVE-2026-84830).

2.  [Documentation SEPPmail – « User > Advanced Settings »](https://docs.seppmail.com/ch/07_mi_15_usr_04_advanced-settings.html): référence des champs de la section External Authentication (Connection, User Attributes, Group Attributes, Mapping, Login Test).

3.  [Documentation SEPPmail – « Groups »](https://docs.seppmail.com/ch/07_mi_16_groups.html): groupes prédéfinis de l’appliance ; les membres du groupe admin disposent d’un accès administratif illimité.

4.  [Documentation SEPPmail – « Revision History »](https://docs.seppmail.com/ch/20_revision-history.html): notes de version officielles de la version 15.0.6 avec l’entrée concernant l’authentification de l’interface d’administration auprès de serveurs LDAP externes, ainsi que des versions ultérieures 15.0.7, 15.1.0 et 15.1.0.1.

5.  [SEPPmail 15.0.6 et 15.0.6.1 : correctifs de sécurité et nouvelles fonctions d’administration](/blog/seppmail-releases-15-0-6-und-15-0-6-1): aperçu de toutes les modifications des deux versions.
