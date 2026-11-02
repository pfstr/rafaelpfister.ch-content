---
title: "totemomail 9.5.1 : prise de contrôle de comptes dans le portail web et SSRF via les e-mails entrants corrigés"
navTitle: "totemomail 9.5.1"
description: "totemomail 9.5.1 corrige huit failles de sécurité et met totemomail au niveau auquel se réfèrent les avis Kiteworks publiés fin septembre concernant l’Email Protection Gateway. Deux failles peuvent être exploitées sans authentification : une erreur dans la réinitialisation du mot de passe du portail web permettant de prendre le contrôle de comptes administrateur, et des requêtes serveur vers des URL contenues dans les messages entrants. S’y ajoutent des correctifs de traversée de chemin dans l’Admin API et la Mail API, ainsi que des bibliothèques Tomcat et Batik mises à jour."
date: "2026-10-31"
kategorie: "Totemomail"
timeToRead: "5 min de lecture"
themen:
  - totemomail
  - sicherheitsluecken
produkte:
  - "totemomail"
  - "sicherheitsluecken"
protokolle:
  - "releases"
  - "haertung"
slug: "totemomail-9-5-1-prise-de-controle-de-comptes-dans-le-portail-web-et-ssrf-via-les-e-mails"
translationId: "article-2bd15f63632ab728"
draft: false
translationOf: totemomail-9-5-1-sicherheitsupdate
url: https://rafaelpfister.ch/fr/blog/totemomail-9-5-1-prise-de-controle-de-comptes-dans-le-portail-web-et-ssrf-via-les-e-mails
translationSourceHash: 5d441bdc7ec39d7fcef423e826bf14772f11bca2aa3dbdf4e1e3c83297131635
translationModel: gpt-5.6-terra
translatedAt: 2026-10-31T19:46:21.539Z
translationReview: automatic
---

# totemomail 9.5.1 : prise de contrôle de comptes dans le portail web et SSRF via les e-mails entrants corrigés

totemo a publié totemomail 9.5.1 (Build 101). Le correctif contient huit corrections de sécurité, une amélioration et trois corrections de bugs. Deux des failles peuvent être exploitées sans authentification : une erreur dans la réinitialisation du mot de passe du portail web, qui permet également de prendre le contrôle de comptes administrateur, et des requêtes serveur vers des URL contenues dans des messages entrants. Toutes deux concernent des composants accessibles depuis Internet dans la plupart des installations. Planifiez donc rapidement la mise à jour.

## De 9.2.1 directement à 9.5.1

Les notes de version présentent 9.5.1 directement après 9.2.1 ; les versions 9.3 et 9.4 n’apparaissent pas dans l’historique de totemomail. Une entrée de la version 9.5.1 renvoie néanmoins à un « security hardening introduced in 9.3.1 ». Les versions intermédiaires proviennent de Kiteworks Email Protection Gateway (EPG), qui repose sur la même base de code : les avis de sécurité de Kiteworks indiquent EPG 9.3.1, 9.4.0, 9.4.1, 9.5.0 et 9.5.1 comme versions correctives. Pour les installations totemomail en 9.2.1, 9.5.1 est la prochaine mise à jour ; les notes de version ne mentionnent aucun autre correctif entre les deux.

## Failles sans authentification

**Server-Side Request Forgery via le contenu des messages.** totemomail récupérait côté serveur des URL issues de messages traités sans suffisamment restreindre leur destination. Un expéditeur non authentifié pouvait concevoir un message de façon à ce que la passerelle envoie des requêtes à des services internes ou à des points de terminaison de métadonnées d’instances Cloud, puis en renvoie les réponses. Il suffit pour cela d’un e-mail traité par la passerelle ; avec une passerelle dans le flux des e-mails entrants, tout expéditeur externe peut le faire. Cela est particulièrement critique pour les installations dans Azure, AWS ou d’autres Clouds, où le point de terminaison de métadonnées `169.254.169.254` fournit des données de configuration et, selon la plateforme, des jetons d’accès de l’instance. Les notes de version ne donnent pas de numéro CVE pour cette faille. Kiteworks répertorie pour l’EPG cinq avis concernant l’accès à des ressources réseau internes (CVE-2026-102095, CVE-2026-102102 à CVE-2026-102105, tous avec un CVSS de 9.1, corrigés dans EPG 9.5.0) ; les textes succincts des avis ne permettent pas de déterminer lequel correspond à l’entrée totemomail.

**Réinitialisation de mot de passe dans le portail web (CVE-2026-85066).** Une erreur dans le processus de réinitialisation du mot de passe permettait de contourner les étapes de vérification et de définir un nouveau mot de passe pour un compte existant. Les notes de version incluent explicitement les comptes administrateur. Kiteworks évalue la faille dans l’avis associé GHSA-rwpq-5xfv-54pv à CVSS 9.8 (critique) et indique EPG 9.5.0 comme version corrective. Le portail web est destiné aux destinataires externes de messages chiffrés et est donc généralement accessible publiquement.

## Failles nécessitant des droits d’administrateur

Trois autres corrections nécessitent une authentification avec des droits administrateur. En combinaison avec la prise de contrôle de comptes dans le portail web, ces conditions perdent toutefois de leur importance.

- **Admin API, gestion de la file d’attente (CVE-2026-34843) :** des téléversements de fichiers insuffisamment protégés permettaient d’écrire des fichiers à des emplacements arbitraires du système de fichiers. Selon les notes de version, cela permettait l’exécution de code à distance. L’avis GHSA-mhc9-4pph-2r9x évalue la faille à CVSS 7.2 ; elle est corrigée dans l’EPG depuis la version 9.4.0.
- **Rule Designer :** lors de l’importation d’une configuration de règle, le pipeline de traitement pouvait charger et exécuter du code en dehors des composants de traitement des e-mails autorisés. Des droits de configuration des règles de messagerie sont requis.
- **Mail API :** une faille de traversée de chemin existait lors de l’extraction de fichiers ZIP téléversés. Les fichiers ne sont désormais extraits que dans le répertoire temporaire.

La quatrième correction concerne la propriété `security.login.requestingKeysAvailable`. Jusqu’à présent, elle masquait uniquement dans le portail web le lien permettant de demander des clés publiques ; la fonction elle-même restait accessible par une requête directe. À partir de la version 9.5.1, la propriété désactive cette fonction. Si vous avez défini cette propriété pour empêcher les demandes de clés, elle ne produit l’effet attendu qu’avec cette mise à jour.

## Ce que le registre Kiteworks répertorie en plus

Le 30 septembre 2026, Kiteworks a publié en bloc 125 avis de sécurité dans son registre d’avis sur GitHub, dont 28 concernant l’Email Protection Gateway. Les notes de version de totemomail 9.5.1 mentionnent nettement moins de points. Le registre ne distingue pas l’EPG de totemomail ; il ne permet donc pas de déterminer si chaque faille EPG affecte également totemomail. Les entrées EPG critiques sont les suivantes :

| CVE | CVSS | Impact selon l’avis | Corrigé à partir d’EPG |
|---|---|---|---|
| CVE-2026-54154 | 10.0 | Exécution de code avec droits root sans authentification | 9.4.1 |
| CVE-2026-85066 | 9.8 | Prise de contrôle de comptes utilisateurs (réinitialisation de mot de passe) | 9.5.0 |
| CVE-2026-85065 | 9.8 | Prise de contrôle de comptes utilisateurs | 9.5.0 |
| CVE-2026-102149 | 9.4 | Prise de contrôle de comptes utilisateurs | 9.5.1 |
| CVE-2026-102106 | 9.1 | Modification de la configuration d’administration | 9.5.0 |
| CVE-2026-102095, CVE-2026-102102 à 102105 | 9.1 | Accès à des ressources réseau internes | 9.5.0 |

S’y ajoutent 13 entrées classées comme élevées, dont la plupart concernent l’exécution de code avec des droits administrateur (CVSS 7.2). La version corrective la plus récente mentionnée est 9.5.1 ; Kiteworks a désigné 9.5.1 fin septembre comme la version actuelle intégrant toutes les vulnérabilités connues. En mettant totemomail à jour vers 9.5.1, vous atteignez ainsi le niveau auquel se réfèrent tous les avis EPG du registre.

## Bibliothèques mises à jour

Les bibliothèques fournies Apache Tomcat et Apache XML Graphics Batik ont été mises à jour. Les notes de version mentionnent les CVE-2026-50229, CVE-2026-53404, CVE-2026-53434, CVE-2026-55276, CVE-2026-55955, CVE-2026-55956 et CVE-2026-59083, sans les attribuer individuellement aux deux bibliothèques. En outre, woodstox-core-asl a été mis à jour en raison d’une vulnérabilité ; aucun numéro CVE n’est indiqué pour celle-ci.

## Amélioration et corrections de bugs

- **Migration EPG :** lors de la migration depuis une base de données Oracle, totemomail prend désormais en charge Oracle Wallet. Si aucun mot de passe de base de données n’est configuré, la migration se connecte via le Wallet.
- **RMS :** des messages pouvaient rester bloqués lors du traitement, car les actions RMS attendaient indéfiniment l’authentification. Le délai d’attente est désormais limité à une minute. Cela concerne les installations avec Microsoft Purview Information Protection (RMS).
- **TRE :** la fonction « Open Online » pour les Registered Envelopes fonctionne à nouveau. Elle ne fonctionnait plus depuis le renforcement de sécurité mentionné de la version 9.3.1.
- **Rapport de licence :** pour les licences Cloud, le rapport compte à nouveau correctement les utilisateurs internes avec des types de fonctionnalités prédéfinis. L’article sur la [limite de licence et le nettoyage via LDAP](/blog/totemomail-licensed-user-limit-ldap-cleanup) décrit l’incidence des comptes orphelins sur le décompte des licences.

## Installation

Le correctif peut être installé sur les systèmes à partir de totemomail 7 b985 sous Windows, Linux, Solaris et FreeBSD. Connectez-vous à la console d’administration, téléversez le fichier TPF fourni sous `[Instanzname]` > « Update », puis lancez la mise à jour. En cas de plusieurs instances, cette opération doit être effectuée séparément pour chaque instance. totemomail redémarre ensuite automatiquement dans la mesure du possible ; prévoyez une fenêtre de maintenance et créez au préalable une sauvegarde ou un snapshot.

En attendant l’installation de la mise à jour, deux mesures réduisent le risque : rendre la console d’administration et l’Admin API accessibles uniquement depuis le réseau d’administration, et limiter les connexions sortantes de la passerelle aux destinations réellement nécessaires. Pour les installations Cloud, le point de terminaison de métadonnées `169.254.169.254` fait partie des destinations que la passerelle ne doit pas pouvoir atteindre. Ces deux mesures ne remplacent pas le correctif ; seule la mise à jour corrige la faille de réinitialisation du mot de passe du portail web.

## Sources

1.  Notes de version totemomail 9.5.1 (Build 101), totemo ag : guide d’installation et notes de version au format PDF, disponibles pour les clients auprès du support du fabricant ; base de toutes les informations de cet article.

2.  [Kiteworks Security Advisories sur GitHub](https://github.com/kiteworks/security-advisories/security/advisories): registre officiel contenant les valeurs CVSS et les versions correctives pour Email Protection Gateway, Core, MFT et Advanced Forms.

3.  [GHSA-rwpq-5xfv-54pv : prise de contrôle de compte (CVE-2026-85066)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-rwpq-5xfv-54pv): avis sur l’erreur de réinitialisation du mot de passe, CVSS 9.8, corrigée dans EPG 9.5.0.

4.  [GHSA-5xhq-9wq3-rvj6 : exécution de code arbitraire (CVE-2026-54154)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-5xhq-9wq3-rvj6): faille EPG la plus critique avec un CVSS de 10.0, corrigée dans EPG 9.4.1.

5.  [P.K. Sharma : Kiteworks a publié 125 avis](https://www.pk-sharma.com/briefing/kiteworks-125-advisories-which-is-the-shutdown-flaw): analyse de la publication du 30 septembre 2026 avec un décompte par produit et degré de gravité, ainsi que les déclarations de Kiteworks concernant la version 9.5.1.

6.  [Comprendre le routage des e-mails entre totemomail et Exchange Online](/blog/totemomail-m365): explique comment totemomail traite les messages entrants ; pertinent pour déterminer quels e-mails peuvent déclencher la faille SSRF.

7.  [Limite de licence Totemomail atteinte : nettoyer les utilisateurs orphelins via LDAP](/blog/totemomail-licensed-user-limit-ldap-cleanup): contexte sur le décompte des licences concerné par la correction du rapport de licence.
