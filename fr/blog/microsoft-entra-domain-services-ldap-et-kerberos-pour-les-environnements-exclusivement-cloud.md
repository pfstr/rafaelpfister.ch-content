---
title: "Microsoft Entra Domain Services : LDAP et Kerberos pour les environnements cloud-only"
navTitle: "Entra Domain Services"
description: "Entra ID ne parle ni LDAP ni Kerberos. Microsoft Entra Domain Services fournit un domaine Active Directory géré qui synchronise les utilisateurs depuis Entra ID et propose des protocoles classiques. Fonctionnement, limites, coûts et cas pratique avec une passerelle e-mail."
date: "2026-07-30"
kategorie: "Active Directory / Entra"
timeToRead: "8 min de lecture"
themen:
  - active-directory-entra
slug: "microsoft-entra-domain-services-ldap-et-kerberos-pour-les-environnements-exclusivement-cloud"
translationId: "article-9c271900a94406b8"
draft: false
translationOf: microsoft-entra-domain-services-ldap-kerberos
translationSourceHash: 6360f60ed2e92d286f0e279f487b62a86fa9a987c2f574b0a53af0d31f0d736b
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:20:26.937Z
translationReview: automatic
url: https://rafaelpfister.ch/fr/blog/microsoft-entra-domain-services-ldap-et-kerberos-pour-les-environnements-exclusivement-cloud
---

# Microsoft Entra Domain Services : LDAP et Kerberos pour les environnements cloud-only

Quiconque a entièrement migré ses utilisateurs vers Microsoft Entra ID (anciennement Azure Active Directory) s’en rend compte au plus tard lors de la première appliance ou application legacy : Entra ID répond aux requêtes via Microsoft Graph et des protocoles d’authentification modernes tels qu’OAuth et SAML, mais pas via LDAP, Kerberos ou NTLM. Un bind LDAP vers Entra ID est tout simplement impossible. Pour tout ce qui attend un Active Directory classique, Microsoft propose son propre service : Microsoft Entra Domain Services, anciennement Azure AD Domain Services.

## Ce que fournit le service

Entra Domain Services est un domaine Active Directory géré. Microsoft exploite à cet effet deux contrôleurs de domaine Windows dans un VNet Azure, s’occupe du patching, de la réplication et des sauvegardes, et synchronise automatiquement les utilisateurs et les groupes d’Entra ID vers le domaine. La synchronisation ne fonctionne que dans un sens, d’Entra ID vers le domaine géré ; les modifications effectuées directement dans le domaine ne sont pas répercutées.

Vu de l’extérieur, le domaine se comporte comme un Active Directory ordinaire : il répond aux requêtes LDAP et LDAPS, prend en charge l’authentification Kerberos et NTLM, permet de joindre des VM au domaine et propose des stratégies de groupe limitées. Les applications et appareils n’ont pas besoin d’être adaptés pour cela ; ils voient un contrôleur de domaine.

## À quoi il sert

Le service cible les environnements qui sont en réalité cloud-only, mais qui exploitent certains composants ayant des exigences d’annuaire classiques :

- **Appliances et applications métier avec connexion LDAP :** appareils qui recherchent des utilisateurs via LDAP, évaluent les appartenances à des groupes ou vérifient les connexions par bind LDAP.
- **Migrations lift-and-shift :** charges de travail serveur qui doivent rester liées au domaine (Kerberos, NTLM, jonction au domaine), sans devoir exploiter ses propres contrôleurs de domaine dans Azure.
- **Environnements sans AD local :** là où il n’y a jamais eu d’Active Directory ou qu’il a été supprimé, le domaine géré remplace la mise en place de ses propres DC et la charge d’exploitation associée.

Important pour bien délimiter le sujet : ceux qui exploitent encore un Active Directory local avec synchronisation Entra Connect n’ont généralement pas besoin du service ; l’appliance interroge alors l’AD existant. Entra Domain Services comble la lacune lorsque Entra ID est la seule source d’utilisateurs.

## Architecture et configuration

Le domaine géré est fourni dans son propre VNet ou sous-réseau et reçoit deux adresses fixes de contrôleurs de domaine. Les charges de travail dans d’autres VNets y accèdent via le peering VNet ; les serveurs DNS des VNets concernés doivent pointer vers les contrôleurs de domaine afin que le nom de domaine et les objets puissent être résolus. L’accès est limité au moyen de Network Security Groups aux sources et ports réellement nécessaires.

Quelques particularités du domaine géré qui sont pertinentes lors de la connexion d’applications :

- Les utilisateurs synchronisés se trouvent dans l’OU **AADDC Users**, et le domaine porte, sans configuration propre, le suffixe **onmicrosoft.com**. La base de recherche et les DN de bind doivent refléter cette structure, par exemple CN=svc-ldap,OU=AADDC Users,DC=example,DC=onmicrosoft,DC=com.
- Il n’existe pas de Domain Administrator. L’administration passe par le groupe délégué AAD DC Administrators ; les extensions de schéma ne sont pas possibles.
- Pour les comptes de bind LDAP, un compte dédié et non privilégié suffit ; pour de simples requêtes d’annuaire dans Entra ID, le rôle Directory Readers est requis.

## Le problème des hash de mots de passe

Un point fait régulièrement perdre du temps lors des tests : les connexions Kerberos et NTLM ainsi que les binds LDAP nécessitent des hash de mots de passe dans le domaine géré. Pour les comptes cloud-only, Entra ID ne génère ces hash qu’au prochain changement de mot de passe après l’activation du service. Un utilisateur fraîchement synchronisé est donc visible dans l’annuaire, mais ne peut se connecter qu’après avoir changé son mot de passe une fois. Pour les comptes hybrides, les hash doivent également être synchronisés depuis l’AD local via Entra Connect.

## Secure LDAP étape par étape

Au sein du domaine, LDAP fonctionne par défaut sans chiffrement via le port 389. Pour les connexions et tout accès en dehors de réseaux strictement isolés, la connexion doit passer par Secure LDAP (LDAPS, port 636) ; de toute façon, le service ne propose l’accès depuis l’extérieur du VNet que sous forme chiffrée. La configuration se compose de quatre étapes.

**1. Obtenir un certificat.** Secure LDAP requiert son propre certificat, téléversé au format PFX avec la clé privée. Le Subject ou SAN doit couvrir le domaine géré avec un wildcard (par exemple *.example.onmicrosoft.com), car les requêtes peuvent arriver sur l’un ou l’autre des deux contrôleurs de domaine. Les options sont une CA publique, sa propre PKI ou un certificat autosigné créé spécifiquement. Avec un certificat autosigné, la chaîne doit être enregistrée comme fiable sur chaque système demandeur ; toutes les appliances ne le permettent pas. Lorsqu’on a le choix, sa propre PKI ou une CA publique est plus sereine.

**2. Activer Secure LDAP.** Dans le portail, sous Settings > Secure LDAP, activez la fonction et téléversez le PFX avec son mot de passe. L’accès via Internet peut y être autorisé en option ; le domaine géré reçoit alors une adresse IP publique.

**3. Réseau et DNS.** L’adresse IP externe est indiquée sous Properties. La règle NSG associée ouvre TCP/636 et doit être limitée aux adresses IP source réellement nécessaires, et non à Any. Pour la résolution de noms, une entrée DNS (par exemple ldaps.example.com) pointe vers cette IP ; elle doit correspondre au certificat. Les accès internes continuent de passer directement par les adresses des contrôleurs de domaine.

**4. Tester la connexion.** Avant de modifier l’application, il vaut la peine d’effectuer un test avec un navigateur LDAP, ldp.exe ou ldapsearch sur le port 636 : bind avec le compte de service, puis recherche sous l’OU AADDC Users. Ce n’est que lorsque le bind et la recherche fonctionnent correctement que l’application entre en jeu.

Pour configurer le service lui-même, le compte du portail doit disposer des rôles Application Administrator, Domain Services Contributor et Groups Administrator ; le déploiement du domaine géré dure un peu plus d’une heure. Les paramètres de sécurité permettent en outre d’imposer TLS 1.2 comme version minimale.

## Coûts

Entra Domain Services représente un coût d’exploitation permanent : le service est facturé à l’heure selon le SKU, auxquels s’ajoutent le VNet, le peering et les éventuelles VM de test. Pour un seul petit cas d’usage LDAP, c’est un prix conséquent ; l’alternative consistant à exploiter ses propres contrôleurs de domaine sous forme de VM paie toutefois cette économie par la responsabilité du patching, des sauvegardes et de la disponibilité.

## Cas pratique : passerelle e-mail avec connexion LDAP

Un exemple concret de la catégorie des appliances est le SEPPmail Secure E-Mail Gateway. Il utilise LDAP pour la création des utilisateurs et les requêtes d’autorisations, et depuis le firmware 15.0.6 également pour la [connexion à l’interface graphique d’administration](/blog/seppmail-admin-gui-ldap-authentifizierung). Une appliance dans le VNet Azure atteint le domaine géré via le peering VNet avec un compte de bind dédié (Directory Readers), sécurisé par des NSG. Au plus tard pour la connexion à l’interface graphique d’administration, dont l’option TLS est active par défaut, la connexion doit passer par Secure LDAP.

## Conclusion

Entra Domain Services n’est pas un remplacement d’Entra ID, mais un pont : le service traduit une base d’utilisateurs cloud en un domaine AD classique pour tout ce qui exige LDAP, Kerberos ou la jonction au domaine. Ceux qui ne doivent connecter qu’une seule application devraient mettre les coûts récurrents en balance avec une modernisation de l’application. Une fois le service en place, les appliances et applications legacy se comportent comme dans un environnement AD familier, y compris les particularités décrites concernant la structure des OU, les autorisations et les hash de mots de passe.

## Sources

1.  [Microsoft Learn – « Qu’est-ce que Microsoft Entra Domain Services ? »](https://learn.microsoft.com/de-de/entra/identity/domain-services/overview) : fonctionnalités du domaine géré, protocoles pris en charge et distinction par rapport à Entra ID et aux contrôleurs de domaine exploités en propre.

2.  [Microsoft Learn – « Synchronisation dans Entra Domain Services »](https://learn.microsoft.com/de-de/entra/identity/domain-services/synchronization) : synchronisation unidirectionnelle, structure des OU et comportement des hash de mots de passe pour les comptes cloud-only et hybrides.

3.  [Microsoft Learn – « Configurer Secure LDAP »](https://learn.microsoft.com/de-de/entra/identity/domain-services/tutorial-configure-ldaps) : LDAPS avec son propre certificat pour les accès LDAP chiffrés.

4.  [Connecter l’interface graphique d’administration SEPPmail à Active Directory](/blog/seppmail-admin-gui-ldap-authentifizierung) : configuration de la connexion LDAP à l’interface graphique d’administration à partir du firmware 15.0.6.
