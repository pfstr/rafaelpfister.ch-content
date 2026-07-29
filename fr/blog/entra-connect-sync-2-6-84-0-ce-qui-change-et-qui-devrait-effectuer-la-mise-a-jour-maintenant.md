---
slug: "entra-connect-sync-2-6-84-0-ce-qui-change-et-qui-devrait-effectuer-la-mise-a-jour-maintenant"
title: "Entra Connect Sync 2.6.84.0 : ce qui change et qui devrait effectuer la mise à jour maintenant"
navTitle: "Entra Connect 2.6.84"
description: "Cette version de sécurité apporte la prise en charge des passkeys et des changements concernant l’authentification des applications, PowerShell et Password Hash Sync. La version précédente a été retirée ; la mise à jour nécessite donc une décision graduée."
date: "2026-07-17"
kategorie: "Microsoft Entra"
timeToRead: "11 min de lecture"
themen:
  - microsoft-entra
  - active-directory-entra
draft: false
translationOf: "entra-connect-2-6-84-0"
url: "https://rafaelpfister.ch/fr/blog/entra-connect-sync-2-6-84-0-ce-qui-change-et-qui-devrait-effectuer-la-mise-a-jour-maintenant"
translationId: article-85bd27acb917e406
translationReview: automatic
translationSourceHash: e4dc8f6498301c03d85afdba4b310d0af7ba497f7ee781448e2d02e5c62d26d9
translatedAt: 2026-07-29T12:29:38.933Z
---

# Entra Connect Sync 2.6.84.0 : ce qui change et qui devrait effectuer la mise à jour maintenant

Microsoft a publié Entra Connect Sync 2.6.84.0 le 7 juillet 2026 en tant que version de sécurité et recommande une mise à niveau rapide. Dans le même temps, la version précédente directe, 2.6.79.0, a été retirée en raison d’un problème d’installation découvert ultérieurement. La conséquence n’est ni « installer immédiatement partout » ni « attendre et ignorer » : les systèmes concernés et ceux qui sortiront bientôt du support devraient migrer rapidement, tandis que tous les autres peuvent d’abord tester la mise à jour de manière contrôlée.

## Pourquoi cette version mérite une prudence particulière

La branche 2.6 d’Entra Connect Sync a connu un démarrage difficile. Un bref retour en arrière, car il est pertinent pour la décision de mise à jour :

- **2.6.1.0** (février 2026) a notamment corrigé un bug selon lequel la modification de la configuration du connecteur Entra ID dans Synchronization Service Manager supprimait les paramètres de l’authentification basée sur les applications, entraînant l’échec de l’assistant et de la rotation des certificats. Pour toutes les versions 2.5, la recommandation remarquable était donc de ne tout simplement pas utiliser l’interface d’administration du produit.
- **2.6.3.0** (mars 2026) était un correctif pour un problème où Auto-Upgrade pouvait arrêter inopinément le serveur Entra Connect. La solution de contournement à l’époque : Auto-Upgrade détecte les fichiers de configuration modifiés manuellement et ignore simplement ces serveurs.
- **2.6.79.0** (juin 2026) a été entièrement retirée après sa publication. Le programme d’installation n’est plus disponible ; selon Microsoft, les personnes ayant installé cette version doivent la désinstaller et installer la version 2.6.84.0. Microsoft ne documente pas précisément la nature du problème.

À ce jour, la version 2.6.84.0 n’est disponible au téléchargement que via le Microsoft Entra Admin Center (« Released for download »). Aucun déploiement d’Auto-Upgrade n’a encore été annoncé. C’est également un signal : Microsoft ne distribue pas encore cette version à grande échelle sur les installations existantes.

## Nouvelles fonctionnalités

### Authentification résistante au phishing dans l’assistant d’installation (préversion)

L’assistant d’installation prend désormais en charge la connexion avec des passkeys et des clés de sécurité FIDO2 via Windows Web Account Manager (WAM). Contexte : depuis 2024/2025, Microsoft impose progressivement l’authentification multifacteur pour les connexions aux interfaces d’administration Azure et Entra, et de nombreuses organisations ont limité leurs comptes d’administrateur via Conditional Access à des méthodes résistantes au phishing (FIDO2, passkeys, authentification par certificat). Or, ces comptes correctement sécurisés ne pouvaient jusqu’à présent pas se connecter dans l’assistant Entra Connect, car la boîte de dialogue de connexion intégrée ne prenait pas en charge ces méthodes. En pratique, cela entraînait des solutions de contournement peu élégantes : par exemple, des « comptes de configuration » dédiés avec des exigences d’authentification plus faibles, uniquement pour permettre à l’assistant de s’exécuter. Cette lacune est désormais comblée, même si ce n’est pour l’instant qu’en préversion.

### Prise en charge du cloud souverain français

La version 2.6.84.0 apporte la prise en charge de l’environnement de cloud souverain français, notamment Pass-through Authentication, Seamless Single Sign-On, Password Writeback et la surveillance de l’agent Health. Dans le même temps, un bug a été corrigé : le nom du cloud Application Proxy n’était pas correctement résolu dans le cloud France et l’inscription PTA échouait avec « EnvironmentName attribute is invalid ».

## Changements de comportement en détail

La partie la plus intéressante de cette version ne réside pas dans les nouvelles fonctionnalités, mais dans les comportements modifiés. Plusieurs d’entre eux corrigent des choix de conception qui ont provoqué des surprises en pratique.

### Auto-Upgrade ne détruit plus les fichiers de configuration personnalisés

C’est le changement avec le plus long historique. Jusqu’à présent, Auto-Upgrade écrasait entièrement le fichier `miiserver.exe.config` lors de la mise à jour. Les adaptations manuelles étaient perdues. Cela peut sembler un cas marginal, mais ce ne l’était pas : Microsoft avait lui-même demandé aux administrateurs dans des environnements FIPS de modifier précisément ce fichier afin que Password Hash Synchronization fonctionne avec le mode FIPS activé. Toute personne ayant suivi les instructions officielles disposait donc d’un fichier de configuration « modifié ».

Les conséquences sont apparues lors de la mise à niveau vers 2.5.190.0 et 2.6.1.0 sous la forme d’un problème connu : si le programme d’installation détecte un fichier `miiserver.exe.config` modifié, il laisse le fichier intact ; mais la nouvelle liaison d’assembly manque alors, et le service de synchronisation s’arrête après la mise à niveau avec `System.IO.FileLoadException: Could not load file or assembly 'System.Diagnostics.DiagnosticSource, Version=6.0.0.1'`. La solution de contournement documentée : ajouter manuellement une redirection de liaison dans la section `assemblyBinding` de `miiserver.exe.config` (sous `%programfiles%\Microsoft Azure AD Sync\Bin`) :

```xml
<dependentAssembly>
  <assemblyIdentity name="System.Diagnostics.DiagnosticSource" publicKeyToken="cc7b13ffcd2ddd51" culture="neutral" />
  <bindingRedirect oldVersion="0.0.0.0-8.0.0.0" newVersion="8.0.0.0" />
</dependentAssembly>
```

Redémarrez ensuite le service ADSync. Le correctif 2.6.3.0 n’a atténué le problème que pour Auto-Upgrade : les serveurs concernés étaient simplement ignorés et restaient sur l’ancienne version. Avec la version 2.6.84.0 arrive la véritable solution : le processus de mise à niveau fusionne les personnalisations du client avec la nouvelle configuration et valide le résultat avant de l’appliquer. Lors d’une mise à niveau manuelle depuis une version concernée, il reste néanmoins recommandé de vérifier au préalable l’état de `miiserver.exe.config` et de sauvegarder le fichier : le mécanisme de fusion est nouveau et n’a donc pas encore fait ses preuves en pratique.

### Authentification basée sur les applications : fin du repli silencieux et de la conversion silencieuse

Pour rappel : depuis la version 2.5.76.0, l’authentification basée sur les applications (ABA) est généralement disponible et constitue la norme. Au lieu de l’ancien compte Directory Synchronization (un compte cloud avec mot de passe enregistré), le serveur Sync s’authentifie en tant qu’application Entra ID au moyen d’un certificat, idéalement protégé par TPM. Cette architecture est nettement plus robuste : aucun mot de passe susceptible de fuiter et une information d’identification liée à la machine.

La version 2.6.84.0 corrige deux comportements qui compromettaient ce gain de sécurité :

**Plus de repli silencieux.** Si la configuration ABA échouait dans l’assistant, l’installation revenait jusqu’à présent sans commentaire au compte hérité. Résultat : l’administrateur pensait disposer d’une connexion basée sur certificat, alors que le serveur utilisait en réalité l’ancien compte à mot de passe. Un schéma classique de fail-open. Désormais, l’assistant s’interrompt avec un message d’erreur clair (« Microsoft Entra Connect could not configure application-based authentication for this server. Setup cannot continue. »), afin que la cause réelle soit corrigée au lieu d’être masquée.

**Plus de conversion automatique en arrière-plan.** Jusqu’à présent, Entra Connect convertissait de lui-même les serveurs existants du compte hérité vers ABA pendant le fonctionnement de la synchronisation. Bien intentionné du point de vue de la sécurité, mais un cauchemar opérationnel : une méthode d’authentification change sans demande, sans fenêtre de changement et sans que personne ne le sache. Et si quelque chose se passe mal (problèmes TPM, conflits Conditional Access, pare-feu), la synchronisation s’arrête. Désormais : seules les nouvelles installations configurent ABA automatiquement ; les serveurs existants ne basculent que lorsqu’un administrateur démarre l’assistant et sélectionne explicitement **Configure application-based authentication to Microsoft Entra ID**. Le changement revient ainsi là où il doit être : dans une modification planifiée.

En complément, la gestion du TPM a été améliorée : l’installation teste désormais au préalable la capacité d’un certificat à signer et traite correctement la vérification des signatures TPM. Sur les serveurs dotés d’un firmware TPM défaillant, incapable de produire une signature valide, l’installation bascule de manière contrôlée vers un certificat logiciel. Cela aussi a un historique : les échecs ABA liés au TPM se sont prolongés sur plusieurs versions antérieures (2.5.79.0, 2.5.190.0), notamment en raison d’incompatibilités entre les implémentations TPM et le mécanisme de signature par défaut de la bibliothèque MSAL.

### Les cmdlets PowerShell exigent désormais une connexion administrateur explicite

Un changement que les exploitants de scripts doivent garder à l’esprit : les cmdlets `Set-ADSyncAADCompanyFeature` et `Set-ADSyncAADPasswordSyncState`, qui modifient la configuration cloud, exigent désormais le paramètre `-AADUsername` pour une authentification administrateur interactive. L’assistant lui-même n’écrit plus les modifications cloud avec les informations d’identification de service enregistrées, mais via une connexion MSAL interactive. Et l’assistant de désinstallation demande des informations d’identification administrateur pour nettoyer la configuration cloud ; si cette étape est ignorée, seul le nettoyage local est effectué.

Le contexte est le même fil conducteur que pour ABA : les actions sur le tenant doivent être associées à une identité d’administrateur réelle et traçable plutôt qu’à un compte de service anonyme. Cela correspond à un correctif de bug de cette même version : jusqu’à présent, la journalisation d’audit administrateur enregistrait l’identité du compte de service au lieu de celle de l’administrateur réellement intervenant lors des modifications de règles de synchronisation ; une piste d’audit qui manquait son objectif. Les deux éléments réunis donnent enfin une journalisation d’audit exploitable. Conséquence pratique : ceux qui appelaient jusqu’ici ces cmdlets sans surveillance dans des scripts doivent revoir ces processus : l’authentification interactive et l’automatisation ne font pas bon ménage.

### Suppression de l’auto-réparation PHS

Le changement le plus discret, mais conceptuellement intéressant : Password Hash Synchronization ne réactive plus automatiquement son indicateur de fonctionnalité cloud en arrière-plan. Si cet indicateur est désactivé, un administrateur doit désormais le réactiver explicitement.

Jusqu’à présent, lorsque PHS était désactivé au niveau du tenant, volontairement ou accidentellement, la fonctionnalité se « réparait » elle-même et se réactivait. Pour les environnements ayant délibérément désactivé PHS, par exemple pour des raisons de conformité parce qu’aucun hash de mot de passe ne doit être envoyé dans le cloud, ou pendant une phase de migration, il s’agissait d’une fonctionnalité qui annulait une décision documentée de l’administrateur. Le fait qu’un mécanisme synchronisant des hash de mots de passe se réactive de sa propre initiative était difficile à justifier.

L’envers de la médaille ne doit cependant pas être passé sous silence : l’auto-réparation a aussi sauvé des environnements où l’indicateur avait été désactivé par erreur ou par un script défaillant, sans que personne ne s’en aperçoive. Cette protection disparaît désormais. Toute personne utilisant PHS en production, ne serait-ce que comme solution de secours pour les connexions d’urgence, devrait à l’avenir surveiller activement l’état de PHS, par exemple via Entra Connect Health ou en consultant les valeurs de pulsation de la synchronisation.

### Composants mis à jour : SQL LocalDB 2022, MSAL, runtime VC++

Moins spectaculaire, mais nécessaire depuis longtemps : la modernisation des composants fournis :

- **SQL Server LocalDB 2019 → 2022.** La base de données interne d’Entra Connect reposait jusqu’à présent sur SQL Server 2019 Express LocalDB, une version dont le support standard a pris fin en février 2025. Avec SQL Server 2022, l’installation repose à nouveau sur une version encore prise en charge.
- **MSAL 4.64.1 → 4.83.3.** Microsoft Authentication Library est le composant central pour l’acquisition de tous les jetons (ABA, connexion à l’assistant, PowerShell). Ce saut d’une vingtaine de versions mineures apporte les correctifs et améliorations accumulés de la bibliothèque.
- **Visual C++ Redistributable 2013 → 2015–2022 (14.42).** Ici, ce qui est remarquable est moins la mise à jour que l’héritage technique : jusqu’à cette version, Entra Connect dépendait d’un environnement d’exécution dont le support a expiré en avril 2024. La dépendance à VC++ 2013 est maintenant entièrement supprimée.

À cela s’ajoute l’indication générale dans les notes de version selon laquelle « multiple security vulnerabilities in bundled third-party dependencies » ont été corrigées. C’est probablement la raison principale de la classification comme version de sécurité : des composants groupés obsolètes ne sont pas un problème cosmétique dans un produit qui s’exécute avec des droits proches de ceux d’un administrateur de domaine, au centre de l’infrastructure d’identité.

## Les autres correctifs de bugs

Pour être complet, voici les autres corrections :

- **Recherche dans le metaverse dans Synchronization Service Manager** corrigée. Après l’avertissement de ne pas utiliser du tout l’interface dans les versions antérieures, elle semble désormais à nouveau maintenue.
- **Rapport de diagnostic PowerShell (HTML)** à nouveau rendu correctement ; pertinent pour toutes les personnes qui utilisent `Invoke-ADSyncDiagnostics` pour des cas de support.
- **Connecteur Generic SQL :** la création de profils échouait parce que des paramètres obligatoires n’étaient pas renseignés lors de la configuration. Cela concerne les environnements qui connectent des annuaires supplémentaires via le connecteur GSQL.
- **Cloud Chine :** le nom d’instance n’était pas correctement résolu par l’API de point de terminaison Discovery, ce qui pouvait entraîner l’échec de la détection de l’instance cloud.
- **Journalisation d’audit administrateur** : lors de modifications de règles de synchronisation, elle enregistre désormais l’administrateur réel au lieu du compte de service (voir ci-dessus).

## Délais de support : qui doit tout de même agir maintenant

Depuis mars 2023, une politique de retrait stricte s’applique à Entra Connect Sync 2.x : chaque version sort du support douze mois après la publication de la version suivante. Délais actuels :

| Version | Fin du support |
| --- | --- |
| 2.5.3.0 | **31 juillet 2026** |
| 2.5.76.0 | 1er septembre 2026 |
| 2.5.79.0 | 23 octobre 2026 |
| 2.5.190.0 | 2 février 2027 |
| 2.6.1.0 | 10 mars 2027 |
| 2.6.3.0 | 7 juillet 2027 |

Les personnes qui utilisent encore la version 2.5.3.0 ne disposent donc plus que de deux semaines de support. La question n’est pas de savoir s’il faut effectuer la mise à jour, mais uniquement vers quelle version. Microsoft souligne par ailleurs que les versions sorties du support peuvent « unexpectedly » cesser de fonctionner ; pour les versions 1.x retirées, la synchronisation a effectivement été désactivée côté serveur entre-temps. Les prérequis minimaux restent .NET Framework 4.7.2 et TLS 1.2 ; le programme d’installation est disponible exclusivement dans le Centre d’administration Entra (Entra ID → Entra Connect → Get started), et non plus dans le Download Center.

## Recommandation selon la version de départ

Microsoft recommande d’effectuer la mise à jour « dès que possible ». Cette recommandation figurait toutefois dans les mêmes termes au-dessus de la version 2.6.79.0, celle qui a ensuite été retirée. L’historique récent des versions (programme d’installation retiré, correctif dû à des serveurs arrêtés, avertissements concernant l’interface sur plusieurs versions) justifie une évaluation factuelle plutôt qu’un réflexe.

Mon appréciation pour les environnements typiques :

**Attendre quelques semaines est défendable** si vous utilisez une version encore prise en charge (2.5.190.0 ou plus récente), qu’aucun des problèmes corrigés ne vous affecte de manière urgente et qu’aucune des nouvelles fonctionnalités n’est nécessaire. D’après les notes de version, les vulnérabilités de sécurité corrigées se trouvent dans des composants tiers fournis ; un serveur Entra Connect devrait de toute façon être suffisamment isolé — pas d’accès Internet hormis aux points de terminaison Microsoft, pas de connexions interactives, traitement de niveau 0 — pour que cette période soit acceptable. Si la version reste quelques semaines sans rappel et que Microsoft démarre le déploiement d’Auto-Upgrade, ce sera un signal de qualité nettement meilleur que n’importe quelle annonce.

**Vous devriez agir rapidement** si l’un des points suivants s’applique :

- **Vous avez installé la version 2.6.79.0.** L’instruction est alors claire : désinstallez-la et installez la version 2.6.84.0, n’attendez pas.
- **Vous utilisez la version 2.5.3.0** (fin de support le 31 juillet 2026) ou une version encore plus ancienne, déjà arrivée à expiration.
- **L’un des problèmes corrigés vous concerne concrètement**, par exemple la configuration ABA sur des serveurs TPM, le connecteur GSQL ou l’exigence d’audit selon laquelle les modifications de règles doivent être attribuées au bon administrateur.

Pour la mise à niveau elle-même, la procédure habituelle s’applique et est particulièrement recommandée avec cet historique de versions : exportez d’abord la configuration — l’assistant propose **View or export current configuration** —, appliquez d’abord la mise à jour sur un serveur en mode staging et vérifiez-y les cycles de synchronisation, l’assistant et la rotation des certificats, puis seulement ensuite sur le serveur actif. Toute personne disposant d’un fichier `miiserver.exe.config` personnalisé doit le sauvegarder avant la mise à jour et vérifier ensuite que le nouveau mécanisme de fusion a correctement repris les adaptations. Et les personnes exploitant des scripts avec `Set-ADSyncAADCompanyFeature` ou `Set-ADSyncAADPasswordSyncState` doivent les tester avant le déploiement en production ; sinon, ils échoueront en raison du nouveau paramètre obligatoire.

## Sources

1. [Microsoft Entra Connect: Version release history – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history): Notes de version officielles de la version 2.6.84.0, incluant l’avis de retrait de la version 2.6.79.0, le tableau de retrait et le problème connu avec miiserver.exe.config modifié.
1. [Microsoft Entra Connect: Upgrade from a previous version to the latest – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-upgrade-previous-version): Procédure de mise à niveau, y compris la migration par basculement via un serveur en mode staging.
1. [Authenticate to Microsoft Entra ID by using application identity – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/authenticate-application-id): Fonctionnement de l’authentification basée sur les applications, qui remplace le compte de service hérité.
1. [Microsoft Entra Connect: Phishing-resistant authentication – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-passwordless-authentication): Nouvelle connexion par passkey/FIDO2 dans l’assistant d’installation via Windows Web Account Manager.
1. [Microsoft Entra Connect: Automatic upgrade – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-install-automatic-upgrade): Mécanisme et prérequis d’Auto-Upgrade, dont le déploiement pour la version 2.6.84.0 est encore en attente.
1. [Auditing administrator events in Microsoft Entra Connect Sync – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/admin-audit-logging): Journalisation d’audit administrateur, dont l’attribution d’identité pour les règles de synchronisation a été corrigée dans cette version.
1. [SQL Server 2019 – Microsoft Lifecycle](https://learn.microsoft.com/en-us/lifecycle/products/sql-server-2019): Dates de support de la base LocalDB précédemment fournie, dont le support standard a pris fin en février 2025.
