---
title: "Microsoft Entra Domain Services : LDAP et Kerberos pour les environnements exclusivement cloud"
navTitle: "Entra Domain Services"
description: "Entra ID ne parle ni LDAP ni Kerberos. Microsoft Entra Domain Services fournit un domaine Active Directory géré, synchronise les utilisateurs depuis Entra ID et propose des protocoles classiques. Fonctionnement, limites, coûts et cas pratique avec une passerelle e-mail."
date: "2026-07-30"
kategorie: "Active Directory / Entra"
timeToRead: "8 min de lecture"
themen:
  - active-directory-entra
slug: "microsoft-entra-domain-services-ldap-et-kerberos-pour-les-environnements-exclusivement-cloud"
translationId: "article-9c271900a94406b8"
draft: false
translationOf: microsoft-entra-domain-services-ldap-kerberos
url: https://rafaelpfister.ch/fr/blog/microsoft-entra-domain-services-ldap-et-kerberos-pour-les-environnements-exclusivement-cloud
translationSourceHash: 00f01b9fa1426d692146e27b2e15e6926e04ea3cccd4855bd0b18c8c10e36e0d
translationModel: gpt-5.6-terra
translatedAt: 2026-07-30T12:21:46.295Z
translationReview: automatic
---

# Microsoft Entra Domain Services : LDAP et Kerberos pour les environnements exclusivement cloud

Quiconque a entièrement migré ses utilisateurs vers Microsoft Entra ID (anciennement Azure Active Directory) s’en rend compte au plus tard avec la première appliance ou application legacy : Entra ID répond aux requêtes via Microsoft Graph et des protocoles d’authentification modernes tels qu’OAuth et SAML, mais pas via LDAP, Kerberos ou NTLM. Un LDAP bind vers Entra ID n’est tout simplement pas possible. Pour tout ce qui attend un Active Directory classique, Microsoft propose donc son propre service : Microsoft Entra Domain Services, anciennement Azure AD Domain Services.

## Ce que fournit le service

Entra Domain Services est un domaine Active Directory géré. Microsoft exploite à cet effet deux contrôleurs de domaine Windows dans un VNet Azure, assure le patching, la réplication et les sauvegardes, et synchronise automatiquement les utilisateurs et les groupes d’Entra ID vers le domaine. La synchronisation ne s’effectue que dans un sens, d’Entra ID vers le domaine géré ; les modifications apportées directement dans le domaine ne sont pas répercutées.

Vu de l’extérieur, le domaine se comporte comme un Active Directory ordinaire : il répond aux requêtes LDAP et LDAPS, prend en charge l’authentification Kerberos et NTLM, permet de joindre des VM au domaine et offre des stratégies de groupe limitées. Les applications et les appareils ne doivent pas être adaptés ; ils voient un contrôleur de domaine.

## À quoi il sert

Le service vise les environnements qui sont en réalité exclusivement cloud, mais qui exploitent certains composants ayant des exigences classiques en matière d’annuaire :

- **Appliances et applications métier avec intégration LDAP :** appareils qui recherchent des utilisateurs via LDAP, évaluent les appartenances à des groupes ou vérifient les connexions via LDAP bind.
- **Migrations lift-and-shift :** charges de travail serveur qui doivent rester liées au domaine (Kerberos, NTLM, jointure de domaine), sans devoir exploiter leurs propres contrôleurs de domaine dans Azure.
- **Environnements sans AD local :** lorsqu’il n’y a jamais eu d’Active Directory ou qu’il a été démantelé, le domaine géré remplace la mise en place de ses propres DC et leur charge d’exploitation.

Distinction importante : si vous exploitez encore un Active Directory local avec synchronisation Entra Connect, vous n’avez généralement pas besoin de ce service ; l’appliance interroge alors l’AD existant. Entra Domain Services comble le vide lorsque Entra ID est l’unique source d’utilisateurs.

## Architecture et mise en place

Le domaine géré est déployé dans son propre VNet ou sous-réseau et reçoit deux adresses fixes de contrôleurs de domaine. Les charges de travail dans d’autres VNet y accèdent via l’appairage de VNet ; les serveurs DNS des VNet concernés doivent pointer vers les contrôleurs de domaine afin que le nom de domaine et les objets puissent être résolus. L’accès est limité via des groupes de sécurité réseau aux sources et ports réellement nécessaires.

Quelques particularités du domaine géré sont pertinentes lors de la connexion d’applications :

- Les utilisateurs synchronisés se trouvent dans l’OU **AADDC Users** et, sans configuration spécifique, le domaine porte le suffixe **onmicrosoft.com**. Les Search Base et Bind DN doivent refléter cette structure, par exemple CN=svc-ldap,OU=AADDC Users,DC=example,DC=onmicrosoft,DC=com.
- Il n’existe pas de Domain Administrator. L’administration s’effectue via le groupe délégué AAD DC Administrators ; les extensions de schéma ne sont pas possibles.
- Pour les comptes LDAP bind, un compte dédié non privilégié suffit ; pour les requêtes d’annuaire pures dans Entra ID, le rôle Directory Readers est requis.

## Le piège des hash de mots de passe

Un point fait régulièrement perdre du temps lors des tests : les connexions Kerberos et NTLM ainsi que les LDAP bind nécessitent des hash de mots de passe dans le domaine géré. Pour les comptes exclusivement cloud, Entra ID ne génère ces hash qu’à la prochaine modification du mot de passe après l’activation du service. Un utilisateur fraîchement synchronisé est donc visible dans l’annuaire, mais ne peut se connecter qu’après avoir changé une fois son mot de passe. Pour les comptes hybrides, les hash doivent également être synchronisés depuis l’AD local via Entra Connect.

## Secure LDAP pas à pas

Au sein du domaine, LDAP fonctionne par défaut sans chiffrement sur le port 389. Pour les connexions et tout accès en dehors de réseaux strictement isolés, la connexion doit passer par Secure LDAP (LDAPS, port 636) ; le service ne propose de toute façon l’accès depuis l’extérieur du VNet que de manière chiffrée. La configuration se compose de quatre étapes.

**1. Obtenir un certificat.** Secure LDAP requiert un certificat dédié, chargé au format PFX avec sa clé privée. Le Subject ou le SAN doit couvrir le domaine géré à l’aide d’un caractère générique (par exemple *.example.onmicrosoft.com), car les requêtes peuvent aboutir sur l’un ou l’autre des deux contrôleurs de domaine. Les sources possibles sont une CA publique, votre propre PKI ou un certificat autosigné créé spécifiquement. Avec un certificat autosigné, la chaîne doit être enregistrée comme fiable sur chaque système effectuant des requêtes ; toutes les appliances ne le permettent pas. Lorsqu’on a le choix, sa propre PKI ou une CA publique offre davantage de sérénité.

**2. Activer Secure LDAP.** Dans le portail, sous Settings > Secure LDAP, activez la fonctionnalité et chargez le PFX avec son mot de passe. Il est possible d’y autoriser en option l’accès via Internet ; le domaine géré reçoit alors une adresse IP publique.

**3. Réseau et DNS.** L’adresse IP externe se trouve sous Properties. La règle NSG associée ouvre TCP/636 et devrait être limitée aux adresses IP sources réellement nécessaires, et non à Any. Pour la résolution de noms, un enregistrement DNS (par exemple ldaps.example.com) pointe vers cette adresse IP ; il doit correspondre au certificat. Les accès internes continuent de se faire directement vers les adresses des contrôleurs de domaine.

**4. Tester la connexion.** Avant de basculer l’application, il est utile de tester avec un navigateur LDAP, ldp.exe ou ldapsearch sur le port 636 : bind avec le compte de service, puis recherche sous l’OU AADDC Users. Ce n’est que lorsque le bind et la recherche fonctionnent correctement que l’application peut être configurée.

Pour configurer le service lui-même, le compte du portail nécessite les rôles Application Administrator, Domain Services Contributor et Groups Administrator ; le déploiement du domaine géré dure un peu plus d’une heure. Dans les paramètres de sécurité, il est également possible d’imposer TLS 1.2 comme version minimale.

## Coûts

Entra Domain Services représente un coût d’exploitation permanent : le service est facturé à l’heure selon la SKU, auxquels s’ajoutent le VNet, l’appairage et d’éventuelles VM de test. Pour un seul petit cas d’usage LDAP, le prix est conséquent ; l’alternative consistant à exploiter ses propres contrôleurs de domaine sous forme de VM achète toutefois cette économie au prix de la responsabilité du patching, des sauvegardes et de la disponibilité.

## Cas pratique : passerelle e-mail avec intégration LDAP

Un exemple concret de la catégorie des appliances est la SEPPmail Secure E-Mail Gateway. Elle utilise LDAP pour la création d’utilisateurs et les requêtes d’autorisations, et depuis le firmware 15.0.6 également pour la [connexion à l’interface d’administration](/blog/seppmail-admin-gui-ldap-authentifizierung). Une appliance dans le VNet Azure atteint le domaine géré via l’appairage de VNet avec un compte bind dédié (Directory Readers), sécurisé par des NSG. Au plus tard pour la connexion à l’interface d’administration, dont l’option TLS est activée par défaut, la connexion doit passer par Secure LDAP.

## Conclusion

Entra Domain Services n’est pas un remplacement d’Entra ID, mais un pont : le service traduit une base d’utilisateurs cloud en un domaine AD classique pour tout ce qui exige LDAP, Kerberos ou une jointure de domaine. Si une seule application doit être connectée, il convient de comparer les coûts récurrents avec une modernisation de l’application. Une fois le service en place, les appliances et applications legacy se comportent comme dans un environnement AD familier, y compris les particularités décrites concernant la structure des OU, les autorisations et les hash de mots de passe.

## Sources

1.  [Microsoft Learn – « Qu’est-ce que Microsoft Entra Domain Services ? »](https://learn.microsoft.com/de-de/entra/identity/domain-services/overview) : étendue fonctionnelle du domaine géré, protocoles pris en charge et distinction par rapport à Entra ID et aux contrôleurs de domaine exploités en propre.

2.  [Microsoft Learn – « Synchronisation dans Entra Domain Services »](https://learn.microsoft.com/de-de/entra/identity/domain-services/synchronization) : synchronisation à sens unique, structure des OU et comportement des hash de mots de passe pour les comptes exclusivement cloud et hybrides.

3.  [Microsoft Learn – « Configurer Secure LDAP »](https://learn.microsoft.com/de-de/entra/identity/domain-services/tutorial-configure-ldaps) : LDAPS avec son propre certificat pour des accès LDAP chiffrés.

4.  [Connecter l’interface d’administration SEPPmail à Active Directory](/blog/seppmail-admin-gui-ldap-authentifizierung) : configuration de la connexion LDAP à l’interface d’administration à partir du firmware 15.0.6.
