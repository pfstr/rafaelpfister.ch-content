---
slug: "entra-connect-sync-2-6-84-0-ce-qui-change-et-qui-devrait-effectuer-la-mise-a-jour-maintenant"
title: "Entra Connect Sync 2.6.84.0 : ce qui change et qui devrait effectuer la mise à jour maintenant"
navTitle: "Entra Connect 2.6.84"
description: "Cette version de sécurité apporte la prise en charge des passkeys et des modifications de l’authentification applicative, de PowerShell et de la synchronisation des hachages de mots de passe. La version précédente a été retirée ; la mise à jour nécessite donc une décision graduée."
date: "2026-07-17"
kategorie: "Microsoft Entra"
timeToRead: "11 min de lecture"
themen:
  - microsoft-entra
  - active-directory-entra
draft: false
translationOf: "entra-connect-2-6-84-0"
translationId: article-85bd27acb917e406
translationReview: required
translationSourceHash: da16eeec10c227af5ba6f33ae138e0148db5b34736874eeed6b2b60c0b469a81
translatedAt: 2026-09-04T08:49:49.972Z
url: https://rafaelpfister.ch/fr/blog/entra-connect-sync-2-6-84-0-ce-qui-change-et-qui-devrait-effectuer-la-mise-a-jour-maintenant
translationModel: gpt-5.6-terra
---

# Entra Connect Sync 2.6.84.0 : ce qui change et qui devrait effectuer la mise à jour maintenant

Microsoft a publié Entra Connect Sync 2.6.84.0 le 7 juillet 2026 en tant que version de sécurité et recommande une mise à niveau rapide. Dans le même temps, son prédécesseur direct, la version 2.6.79.0, a été retiré en raison d’un problème d’installation découvert ultérieurement. La conséquence n’est ni « installer immédiatement partout » ni « attendre et ignorer » : les systèmes concernés et ceux qui vont bientôt sortir du support devraient migrer rapidement, tandis que les autres peuvent d’abord tester la mise à jour de manière contrôlée.

## Pourquoi cette version mérite une prudence particulière

La branche 2.6 d’Entra Connect Sync a connu des débuts mouvementés. Un bref retour en arrière, car il est pertinent pour la décision de mise à jour :

- **2.6.1.0** (février 2026) corrigeait notamment un bug selon lequel la modification de la configuration du connecteur Entra ID dans Synchronization Service Manager supprimait les paramètres de l’Application-Based Authentication, entraînant l’échec de l’assistant et de la rotation des certificats. Pour toutes les versions 2.5, la recommandation remarquable était donc de ne tout simplement pas utiliser l’interface d’administration du produit.
- **2.6.3.0** (mars 2026) était un correctif urgent pour un problème où Auto-Upgrade pouvait arrêter inopinément le serveur Entra Connect. La solution de contournement à l’époque : Auto-Upgrade détecte les fichiers de configuration modifiés manuellement et ignore simplement ces serveurs.
- **2.6.79.0** (juin 2026) a été entièrement retirée après sa publication. L’installateur n’est plus disponible ; selon Microsoft, ceux qui ont installé cette version doivent la désinstaller et installer la version 2.6.84.0. Microsoft ne documente pas précisément la nature du problème.

À ce jour, la version 2.6.84.0 n’est disponible qu’au téléchargement via le centre d’administration Microsoft Entra (« Released for download »). Aucun déploiement Auto-Upgrade n’a encore été annoncé. C’est également un signal : Microsoft ne distribue pas encore cette version à grande échelle sur les installations existantes.

## Nouvelles fonctionnalités

### Authentification résistante au phishing dans l’assistant d’installation (Preview)

L’assistant d’installation prend désormais en charge l’authentification avec des passkeys et des clés de sécurité FIDO2 via le Windows Web Account Manager (WAM). Contexte : depuis 2024/2025, Microsoft impose progressivement la MFA pour les connexions aux interfaces d’administration Azure et Entra, et de nombreuses organisations ont restreint leurs comptes administrateurs, via Conditional Access, à des méthodes résistantes au phishing (FIDO2, passkeys, authentification basée sur des certificats). Or, ces comptes correctement sécurisés ne pouvaient jusqu’ici pas se connecter dans l’assistant Entra Connect, car la boîte de dialogue de connexion intégrée ne prenait pas en charge ces méthodes. En pratique, cela entraînait des solutions de contournement peu élégantes, par exemple des « comptes de configuration » dédiés soumis à des exigences d’authentification plus faibles, uniquement pour permettre à l’assistant de s’exécuter. Cette lacune est désormais comblée, même si ce n’est pour l’instant qu’en Preview.

### Prise en charge du Sovereign Cloud français

La version 2.6.84.0 apporte la prise en charge de l’environnement Sovereign Cloud français, notamment Pass-through Authentication, Seamless Single Sign-On, Password Writeback et la surveillance via Health Agent. Dans ce contexte, un bug a été corrigé : le nom du cloud Application Proxy n’était pas correctement résolu dans le France Cloud, et l’inscription PTA échouait avec « EnvironmentName attribute is invalid ».

## Changements de comportement en détail

La partie la plus intéressante de cette version n’est pas constituée des nouvelles fonctionnalités, mais des comportements modifiés. Plusieurs corrigent des choix de conception qui ont provoqué des surprises en pratique.

### Auto-Upgrade ne détruit plus les fichiers de configuration personnalisés

C’est le changement qui a l’historique le plus long. Jusqu’ici, Auto-Upgrade remplaçait entièrement le fichier `miiserver.exe.config` lors de la mise à jour. Les adaptations manuelles étaient perdues. Cela semble être un cas marginal, mais ce ne l’était pas : Microsoft avait lui-même demandé aux administrateurs dans des environnements FIPS de modifier précisément ce fichier afin que Password Hash Synchronization fonctionne avec le mode FIPS activé. Quiconque suivait les instructions officielles disposait donc d’un fichier de configuration « modifié ».

Les conséquences se sont manifestées lors de la mise à niveau vers les versions 2.5.190.0 et 2.6.1.0 comme problème connu : si l’installateur détecte un fichier `miiserver.exe.config` modifié, il n’y touche pas ; mais le nouveau binding d’assembly manque alors, et le service de synchronisation échoue après la mise à niveau avec `System.IO.FileLoadException: Could not load file or assembly 'System.Diagnostics.DiagnosticSource, Version=6.0.0.1'`. La solution de contournement documentée : ajouter manuellement un bindingRedirect dans la section `assemblyBinding` de `miiserver.exe.config` (sous `%programfiles%\Microsoft Azure AD Sync\Bin`) :

```xml
<dependentAssembly>
  <assemblyIdentity name="System.Diagnostics.DiagnosticSource" publicKeyToken="cc7b13ffcd2ddd51" culture="neutral" />
  <bindingRedirect oldVersion="0.0.0.0-8.0.0.0" newVersion="8.0.0.0" />
</dependentAssembly>
```

Redémarrez ensuite le service ADSync. Le correctif urgent 2.6.3.0 n’a atténué le problème que pour Auto-Upgrade : les serveurs concernés étaient simplement ignorés et restaient sur l’ancienne version. Avec la version 2.6.84.0 arrive la véritable solution : le processus de mise à niveau fusionne les personnalisations des clients avec la nouvelle configuration et valide le résultat avant de l’appliquer. Ceux qui effectuent une mise à niveau manuelle depuis une version concernée devraient néanmoins vérifier au préalable l’état de leur `miiserver.exe.config` et sauvegarder le fichier : le mécanisme de fusion est nouveau et n’a donc pas encore fait ses preuves en pratique.

### Application-Based Authentication : fin du repli silencieux et de la migration silencieuse

Pour rappel : depuis la version 2.5.76.0, l’Application-Based Authentication (ABA) est generally available et utilisée par défaut. Au lieu de l’ancien Directory Synchronization Account, un compte cloud avec mot de passe enregistré, le serveur de synchronisation s’authentifie comme application Entra ID avec un certificat, idéalement protégé par TPM. C’est une architecture nettement plus robuste : aucun mot de passe susceptible de fuiter, et une information d’identification liée à la machine.

La version 2.6.84.0 corrige deux comportements qui réduisaient ce gain de sécurité :

**Plus de repli silencieux.** Si la configuration ABA échouait dans l’assistant, l’installation revenait jusqu’ici silencieusement au compte hérité. Résultat : l’administrateur croyait disposer d’une authentification basée sur certificat, alors que le serveur fonctionnait en réalité avec l’ancien compte à mot de passe. Un schéma classique de fail-open. Désormais, l’assistant s’interrompt avec un message d’erreur clair (« Microsoft Entra Connect could not configure application-based authentication for this server. Setup cannot continue. »), afin que la cause réelle soit corrigée plutôt que masquée.

**Plus de migration automatique en arrière-plan.** Jusqu’ici, Entra Connect migrait automatiquement les serveurs existants du compte hérité vers ABA pendant l’exécution de la synchronisation. Bien intentionné du point de vue de la sécurité, mais constituant un risque important en exploitation : une méthode d’authentification change sans demande, sans fenêtre de changement et sans que personne ne le sache. Et si quelque chose échoue (problèmes TPM, conflits Conditional Access, pare-feu), la synchronisation s’arrête. Désormais, seules les nouvelles installations configurent ABA automatiquement ; les serveurs existants ne migrent que lorsqu’un administrateur lance l’assistant et sélectionne explicitement **Configure application-based authentication to Microsoft Entra ID**. La migration revient ainsi là où elle doit être : dans un changement planifié.

En complément, la gestion du TPM a été améliorée : l’installation teste désormais au préalable la capacité de signature d’un certificat et gère correctement la vérification de signature TPM. Sur des serveurs dotés d’un firmware TPM défectueux incapable de produire une signature valide, l’installation bascule de manière contrôlée vers un certificat logiciel. Là encore, il y a un historique : des échecs ABA liés au TPM se sont étendus sur plusieurs versions précédentes (2.5.79.0, 2.5.190.0), notamment à cause d’incompatibilités entre les implémentations TPM et le procédé de signature par défaut de la bibliothèque MSAL.

### Les cmdlets PowerShell exigent désormais une authentification administrateur explicite

Un changement que les exploitants de scripts doivent connaître : les cmdlets `Set-ADSyncAADCompanyFeature` et `Set-ADSyncAADPasswordSyncState`, qui modifient la configuration cloud, exigent désormais le paramètre `-AADUsername` pour une authentification administrateur interactive. L’assistant lui-même n’écrit plus les modifications cloud avec des informations d’identification de service enregistrées, mais via une connexion MSAL interactive. L’assistant de désinstallation demande également des informations d’identification administrateur pour nettoyer la configuration cloud ; si cette étape est ignorée, seul le nettoyage local est effectué.

Le contexte est le même fil conducteur que pour ABA : les actions contre le tenant doivent être associées à une identité d’administrateur réelle et traçable plutôt qu’à un compte de service anonyme. Cela correspond à une correction de bug de la même version : jusqu’ici, la journalisation d’audit d’administration enregistrait l’identité du compte de service lors de modifications des règles de synchronisation, au lieu de celle de l’administrateur ayant réellement effectué l’action ; une piste d’audit qui manquait son objectif. C’est l’ensemble de ces deux éléments qui rend enfin l’audit exploitable. Conséquence pratique : ceux qui appelaient jusqu’ici ces cmdlets sans supervision dans des scripts doivent restructurer ces processus : l’authentification interactive et l’automatisation ne font pas bon ménage.

### Suppression de l’auto-réparation PHS

Le changement le plus discret, mais conceptuellement intéressant : Password Hash Synchronization ne réactive plus elle-même son indicateur de fonctionnalité cloud en arrière-plan. Si cet indicateur est désactivé, un administrateur doit désormais le réactiver explicitement.

Auparavant, si PHS était désactivé au niveau du tenant, volontairement ou accidentellement, la fonctionnalité « se réparait » d’elle-même et se réactivait. Dans les environnements où PHS avait été désactivé intentionnellement, par exemple pour des raisons de conformité, parce qu’aucun hachage de mot de passe ne doit être transféré vers le cloud, ou pendant une phase de migration, il s’agissait d’une fonctionnalité qui passait outre une décision documentée de l’administrateur. Il était difficile de justifier qu’un mécanisme synchronisant des hachages de mots de passe se réactive de sa propre initiative.

Il ne faut toutefois pas taire le revers de la médaille : l’auto-réparation a aussi sauvé des environnements dans lesquels l’indicateur avait été désactivé par une erreur ou un script défaillant, sans que personne ne s’en aperçoive. Cette protection disparaît désormais. Ceux qui utilisent PHS en production, même uniquement comme solution de repli pour la connexion d’urgence, devraient à l’avenir surveiller activement le statut PHS, par exemple via Entra Connect Health ou en consultant les valeurs de heartbeat de la synchronisation.

### Composants mis à jour : SQL LocalDB 2022, MSAL, runtime VC++

Moins spectaculaire, mais nécessaire depuis longtemps, la modernisation des composants fournis :

- **SQL Server LocalDB 2019 → 2022.** La base de données interne d’Entra Connect reposait jusqu’ici sur SQL Server 2019 Express LocalDB, une version dont le support standard a pris fin en février 2025. Avec SQL Server 2022, l’installation repose à nouveau sur une version encore prise en charge.
- **MSAL 4.64.1 → 4.83.3.** Microsoft Authentication Library est le composant central pour toute l’obtention de jetons, notamment ABA, la connexion dans l’assistant et PowerShell. Le saut d’environ vingt versions mineures apporte les correctifs et améliorations accumulés de la bibliothèque.
- **Visual C++ Redistributable 2013 → 2015–2022 (14.42).** Ce qui est remarquable ici n’est pas tant la mise à jour que l’héritage : jusqu’à cette version, Entra Connect dépendait d’un environnement d’exécution dont le support avait expiré en avril 2024. La dépendance à VC++ 2013 est désormais entièrement supprimée.

Cela correspond à l’indication générale des notes de publication selon laquelle « multiple security vulnerabilities in bundled third-party dependencies » ont été corrigées. C’est probablement la principale raison de la classification comme version de sécurité : des composants intégrés obsolètes ne sont pas un problème cosmétique dans un produit qui fonctionne avec des droits proches de Domain Admin au cœur de l’infrastructure d’identité.

## Les autres corrections de bugs

Pour être complet, voici les autres corrections :

- **Recherche Metaverse dans Synchronization Service Manager** corrigée. Après l’avertissement de ne pas du tout utiliser l’interface dans les anciennes versions, elle semble désormais à nouveau maintenue.
- **Rapport de diagnostic PowerShell (HTML)** à nouveau rendu correctement ; pertinent pour tous ceux qui utilisent `Invoke-ADSyncDiagnostics` pour les cas de support.
- **Connecteur Generic SQL :** la création de profil échouait parce que des paramètres obligatoires n’étaient pas renseignés lors de la configuration. Cela concerne les environnements qui connectent des annuaires supplémentaires via le connecteur GSQL.
- **China Cloud :** le nom de l’instance n’était pas correctement résolu par l’API du point de terminaison Discovery, ce qui pouvait faire échouer la détection de l’instance cloud.
- **Journalisation d’audit d’administration** : lors de modifications des règles de synchronisation, elle enregistre désormais l’administrateur réel au lieu du compte de service (voir ci-dessus).

## Délais de support : qui doit agir malgré tout maintenant

Depuis mars 2023, Entra Connect Sync 2.x applique une politique de retrait stricte : chaque version sort du support douze mois après la publication de la version suivante. Les échéances actuelles :

| Version | Fin du support |
| --- | --- |
| 2.5.3.0 | **31 juillet 2026** |
| 2.5.76.0 | 1er septembre 2026 |
| 2.5.79.0 | 23 octobre 2026 |
| 2.5.190.0 | 2 février 2027 |
| 2.6.1.0 | 10 mars 2027 |
| 2.6.3.0 | 7 juillet 2027 |

Ceux qui utilisent encore la version 2.5.3.0 n’ont donc plus que deux semaines de support. La question n’est pas de savoir s’il faut effectuer la mise à jour, mais vers quelle version. Microsoft souligne en outre que les versions sorties du support peuvent cesser de fonctionner « unexpectedly » ; pour les versions 1.x retirées, la synchronisation est désormais effectivement désactivée côté serveur. Les prérequis minimaux restent .NET Framework 4.7.2 et TLS 1.2 ; l’installateur est disponible exclusivement dans le centre d’administration Entra (Entra ID → Entra Connect → Get started), et plus dans le Download Center.

## Recommandation selon la version de départ

Microsoft recommande de procéder à la mise à jour « as soon as possible ». Cette recommandation figurait toutefois mot pour mot au-dessus de la version 2.6.79.0, celle qui a ensuite été retirée. L’historique récent des versions, avec un installateur retiré, un correctif urgent dû à des serveurs arrêtés et des avertissements d’interface sur plusieurs versions, justifie une évaluation factuelle plutôt qu’un réflexe.

Mon appréciation pour les environnements typiques :

**Attendre quelques semaines est défendable** si vous utilisez une version encore prise en charge (2.5.190.0 ou ultérieure), qu’aucun des problèmes corrigés ne vous affecte de manière urgente et qu’aucune des nouvelles fonctionnalités n’est nécessaire. D’après les notes de publication, les vulnérabilités de sécurité corrigées se trouvent dans des composants tiers fournis ; un serveur Entra Connect devrait de toute façon être suffisamment cloisonné, sans accès Internet hormis vers les points de terminaison Microsoft, sans connexions interactives et avec un traitement Tier 0, pour que cette fenêtre soit justifiable. Si la version reste disponible sans rappel pendant quelques semaines et que Microsoft lance le déploiement Auto-Upgrade, ce sera un signal de qualité nettement plus fiable que toute annonce.

**Vous devriez agir rapidement** si l’un des points suivants s’applique :

- **Vous avez installé la version 2.6.79.0.** L’instruction est alors claire : désinstallez-la et installez la version 2.6.84.0, n’attendez pas.
- **Vous utilisez la version 2.5.3.0** (fin du support le 31 juillet 2026) ou une version encore plus ancienne, déjà expirée.
- **L’un des problèmes corrigés vous concerne concrètement**, par exemple la configuration ABA sur des serveurs TPM, le connecteur GSQL ou l’exigence d’audit selon laquelle les modifications de règles doivent être attribuées au bon administrateur.

Pour la mise à niveau elle-même, appliquez la procédure habituelle, particulièrement recommandée compte tenu de l’historique de cette version : exportez préalablement la configuration, l’assistant propose **View or export current configuration**, déployez d’abord la mise à jour sur un serveur en mode staging et vérifiez-y les cycles de synchronisation, l’assistant et la rotation des certificats, puis seulement ensuite sur le serveur actif. Ceux qui disposent d’un `miiserver.exe.config` personnalisé le sauvegardent avant la mise à jour et vérifient ensuite que le nouveau mécanisme de fusion a correctement repris les adaptations. Enfin, ceux qui exploitent des scripts avec `Set-ADSyncAADCompanyFeature` ou `Set-ADSyncAADPasswordSyncState` les testent avant le déploiement en production ; autrement, ils échoueront sur le nouveau paramètre obligatoire.

## Sources

1. [Microsoft Entra Connect: Version release history – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history): Notes de publication officielles de la version 2.6.84.0, y compris l’avis de retrait de la version 2.6.79.0, le tableau de retrait et le problème connu avec miiserver.exe.config modifié.
1. [Microsoft Entra Connect: Upgrade from a previous version to the latest – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-upgrade-previous-version): Procédure de mise à niveau, y compris la migration swing via un serveur en mode staging.
1. [Authenticate to Microsoft Entra ID by using application identity – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/authenticate-application-id): Fonctionnement de l’Application-Based Authentication, qui remplace le compte de service hérité.
1. [Microsoft Entra Connect: Phishing-resistant authentication – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-passwordless-authentication): La nouvelle authentification par passkey/FIDO2 dans l’assistant d’installation via le Windows Web Account Manager.
1. [Microsoft Entra Connect: Automatic upgrade – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-install-automatic-upgrade): Mécanisme et prérequis d’Auto-Upgrade, dont le déploiement pour la version 2.6.84.0 est toujours en attente.
1. [Auditing administrator events in Microsoft Entra Connect Sync – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/admin-audit-logging): La journalisation d’audit d’administration, dont l’attribution des identités pour les règles de synchronisation a été corrigée dans cette version.
1. [SQL Server 2019 – Microsoft Lifecycle](https://learn.microsoft.com/en-us/lifecycle/products/sql-server-2019): Informations de support concernant la base LocalDB fournie jusqu’ici, dont le support standard a pris fin en février 2025.
