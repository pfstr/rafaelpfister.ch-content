---
title: "Microsoft Entra Connect Sync 2.6.91.0 et 2.6.92.0 : correctifs de sécurité, migration vers Cloud Sync et mise à niveau obligatoire d’ici avril 2027"
navTitle: "Microsoft Entra Connect Sync 2.6.92.0"
description: "La version 2.6.91.0 apporte des correctifs de sécurité, une migration guidée vers Cloud Sync et résout les problèmes connus de la version 2.6.84.0 ; le correctif 2.6.92.0 corrige la configuration de l’authentification directe. D’ici le 7 avril 2027, tous les serveurs devront exécuter au moins la version 2.6.84.0 avec l’authentification basée sur les applications."
date: "2026-10-26"
kategorie: "Microsoft Entra"
timeToRead: "8 min de lecture"
themen:
  - microsoft-entra
  - active-directory-entra
produkte:
  - "entra"
  - "entra-connect-releases"
protokolle:
  - "releases"
slug: "microsoft-entra-connect-sync-2-6-91-0-et-2-6-92-0-correctifs-de-securite-migration-vers-cloud"
translationId: article-59a51ef69f2e3538
translationOf: entra-connect-2-6-92-0
url: https://rafaelpfister.ch/fr/blog/microsoft-entra-connect-sync-2-6-91-0-et-2-6-92-0-correctifs-de-securite-migration-vers-cloud
translationSourceHash: ff045b1bab72cea3839167d59c0a71010e80a3a20b1644acef89c79921f6f219
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T08:24:33.784Z
translationReview: required
---

# Microsoft Entra Connect Sync 2.6.91.0 et 2.6.92.0 : correctifs de sécurité, migration vers Cloud Sync et mise à niveau obligatoire d’ici avril 2027

Microsoft a publié Entra Connect Sync 2.6.91.0 le 16 septembre 2026 et le correctif 2.6.92.0 le 23 septembre 2026, tous deux uniquement téléchargeables via le Centre d’administration Microsoft Entra. La version 2.6.91.0 corrige des vulnérabilités de sécurité dans des composants tiers inclus, rend la connexion résistante au phishing généralement disponible dans l’assistant d’installation, propose un processus de migration guidé vers Microsoft Entra Cloud Sync et corrige les trois problèmes connus de la version 2.6.84.0. La version 2.6.92.0 corrige une erreur de la version 2.6.91.0 lors de la configuration de l’authentification directe. Microsoft fixe également une échéance : d’ici le 7 avril 2027, tous les serveurs devront exécuter au moins la version 2.6.84.0 et utiliser l’authentification basée sur les applications, sans quoi la synchronisation s’arrêtera. Cet article s’adresse aux administrateurs qui utilisent Entra Connect Sync. La version précédente est décrite dans l’article sur [Entra Connect Sync 2.6.84.0](/blog/entra-connect-2-6-84-0).

## Aperçu des versions

| Version | Date | Type | Points essentiels |
|---|---|---|---|
| 2.6.91.0 | 16.09.2026 | Version fonctionnelle et de sécurité (téléchargement) | Correctifs de sécurité dans des composants tiers, assistant de migration vers Cloud Sync, connexion par passkey dans l’assistant en disponibilité générale, nouvelles autorisations Microsoft Graph, correctifs pour les problèmes connus de la version 2.6.84.0 |
| 2.6.92.0 | 23.09.2026 | Correctif avec corrections de sécurité (téléchargement) | Corrige l’échec de l’enregistrement de l’Authentication Agent lors de l’activation de l’authentification directe |

Microsoft classe la version 2.6.92.0 comme une version contenant des correctifs de sécurité dans les notes de publication et recommande une mise à jour « dès que possible ». Aucun déploiement de mise à niveau automatique n’est annoncé pour ces deux versions (état au 26 octobre 2026). La version 2.6.84.0 reste elle aussi uniquement disponible au téléchargement.

## Mise à niveau obligatoire d’ici le 7 avril 2027

L’historique des versions et le guide de mise à niveau comportent désormais un avertissement identique : Entra Connect Sync doit être mis à jour vers la version 2.6.84.0 ou ultérieure et basculé vers l’authentification basée sur les applications (ABA) d’ici le 7 avril 2027. Microsoft abandonne l’authentification héritée via le compte de synchronisation d’annuaire ; les serveurs qui ne remplissent pas ces conditions ne se synchroniseront plus après cette date. Si la synchronisation s’arrête, Microsoft indique qu’une mise à niveau vers la version actuelle suivie de la configuration d’ABA rétablit le fonctionnement.

Pour la planification, un changement de comportement introduit dans la version 2.6.84.0 est déterminant : depuis cette version, Entra Connect ne convertit plus automatiquement les serveurs existants vers ABA. Seules les nouvelles installations configurent ABA automatiquement. Une simple mise à niveau ne suffit donc pas à respecter l’échéance. Vous devez convertir les serveurs existants dans l’assistant via **Additional tasks** > **Configure application-based authentication to Microsoft Entra ID**.

L’assistant indique la méthode actuellement utilisée par un serveur sous **View or export current configuration**. Microsoft mentionne également ce cmdlet :

```powershell
Get-ADSyncEntraConnectorCredential
```

La valeur renvoyée `ConnectorIdentityType` est `ServiceAccount` (compte hérité) ou `Application` (ABA). Si `ServiceAccount` y figure, la conversion est requise avant le 7 avril 2027. Après une conversion réussie, Microsoft recommande de supprimer l’ancien compte de synchronisation d’annuaire d’Entra ID.

## Version 2.6.91.0

### Correctifs de sécurité

Les notes de publication indiquent des vulnérabilités corrigées dans des composants tiers inclus, des autorisations renforcées sur les fichiers temporaires locaux et des téléchargements sécurisés de photos de profil dans le connecteur SharePoint. Microsoft ne mentionne ni numéros CVE, ni scores CVSS, ni composants concernés. Il n’est donc pas possible d’évaluer le niveau de gravité sur la base des sources publiques.

### Nouvelles autorisations Microsoft Graph et Conditional Access

Entra Connect obtient des autorisations Microsoft Graph supplémentaires. Microsoft précise explicitement que les organisations utilisant des stratégies Conditional Access ciblant des applications devraient vérifier les stratégies qui ciblent `Microsoft.Azure.SyncFabric` ou le Microsoft 365 Reporting Service. Vérifiez ces stratégies avant la mise à niveau afin que les nouveaux accès ne soient pas bloqués.

### Migration guidée vers Cloud Sync

Une procédure guidée est désormais disponible pour passer d’Entra Connect Sync à Microsoft Entra Cloud Sync. Elle comprend une évaluation de la configuration existante, l’installation du Provisioning Agent, une activation progressive et une validation. Cette fonctionnalité est disponible uniquement dans le cloud public Azure. Le guide de mise à niveau de Microsoft recommande de toute façon de vérifier avant une mise à niveau si Cloud Sync convient à votre environnement.

### Connexion résistante au phishing dans l’assistant généralement disponible

La connexion via Windows Web Account Manager, introduite en préversion dans la version 2.6.84.0, est désormais generally available et activée par défaut. La boîte de dialogue prend en charge les passkeys, les clés de sécurité FIDO2 et les mots de passe, réutilise une session de connexion existante pour d’autres services Entra et maintient la rotation des clés Kerberos pour le single sign-on transparent.

### Cmdlets PowerShell modifiés

La version 2.6.84.0 avait rendu le paramètre `-AADUsername` obligatoire pour `Set-ADSyncAADCompanyFeature` et `Set-ADSyncAADPasswordSyncState`. Avec la version 2.6.91.0, ce paramètre est à nouveau facultatif : s’il manque, Entra Connect déduit une indication de connexion de la configuration du connecteur et ouvre une connexion interactive. Cela s’applique à `Set-ADSyncAADCompanyFeature`, `Set-ADSyncAADPasswordSyncState`, `Enable-ADSyncExportDeletionThreshold`, `Set-ADSyncScheduler` et `Set-ADSyncDirSyncConfiguration`. La connexion interactive reste donc intégrée à ces cmdlets ; vous devriez tester après la mise à niveau les scripts qui les appellent sans supervision.

Les personnes configurant le single sign-on transparent avec le module PowerShell autonome doivent importer `ADSync.psd1` avant `AzureADSSO.psd1`.

### Autres modifications

- La boîte de dialogue **Select Containers** du Synchronization Service Manager est désormais en lecture seule. Les modifications de la sélection des conteneurs s’effectuent via **Customize synchronization options** dans l’assistant.
- Prise en charge d’un environnement Sovereign Cloud supplémentaire, y compris l’authentification directe, le single sign-on transparent, Password Writeback et la supervision Health Agent. Les notes de publication ne précisent pas de quel environnement il s’agit.
- La version incluse de SQL Server 2022 LocalDB a été mise à jour de 16.0.4250.1 à 16.0.4262.2.

### Corrections de bugs

- L’installation ou la mise à niveau avec une base de données ADSync existante pouvait échouer avec l’erreur `0xE0474352` (problème connu de la version 2.6.84.0).
- Une mise à niveau ne mettait pas à jour une version installée de Visual C++ 2015–2022 Redistributable antérieure à 14.41.
- L’assistant pouvait se fermer au démarrage si la stratégie Windows PowerShell Transcription contenait un répertoire de sortie non valide.
- La rotation des certificats de l’authentification basée sur les applications pouvait supprimer la dernière clé d’application utilisable en cas de délais de réplication dans l’annuaire.
- Le Synchronization Service Manager pouvait se fermer lors de l’énumération des partitions d’annuaire dans **Connector Properties** (problème connu de la version 2.6.84.0).
- **Change Credentials** ne s’ouvrait pas pour les lignes de connecteur en dehors de la zone visible lorsqu’un grand nombre de connecteurs Active Directory étaient configurés.
- En développant un domaine précédemment désélectionné sur la page **Domain and OU filtering**, l’assistant pouvait sélectionner à nouveau le domaine et activer la synchronisation pour l’ensemble du domaine à la fin (problème connu de la version 2.6.84.0).

Le dernier point mérite attention : si vous utilisez la version 2.6.84.0 et avez repassé l’assistant après la mise à niveau, vous devriez contrôler le filtrage des domaines et des unités d’organisation. Un domaine activé involontairement entraîne des objets supplémentaires dans Entra ID.

## Version 2.6.92.0

Le correctif résout une erreur de la version 2.6.91.0 : l’activation de l’authentification directe via l’assistant pouvait échouer lors de l’enregistrement du Microsoft Entra Connect Authentication Agent installé localement. Les notes de publication ne mentionnent aucun autre changement. Microsoft identifie la version 2.6.92.0 comme une version contenant des correctifs de sécurité ; les vulnérabilités individuelles ne sont pas détaillées.

## Problèmes connus

| Problème | Version concernée | Statut |
|---|---|---|
| L’installation ou la mise à niveau avec une base de données ADSync existante échoue avec `0xE0474352` | 2.6.84.0 | Corrigé dans la version 2.6.91.0 |
| Le Synchronization Service Manager se ferme dans **Connector Properties** | 2.6.84.0 | Corrigé dans la version 2.6.91.0 |
| L’assistant sélectionne à nouveau un domaine désélectionné | 2.6.84.0 | Corrigé dans la version 2.6.91.0 |
| L’activation de PTA dans l’assistant échoue lors de l’enregistrement de l’agent | 2.6.91.0 | Corrigé dans la version 2.6.92.0 |
| La synchronisation échoue après la mise à niveau (`System.IO.FileLoadException`) si `miiserver.exe.config` a été modifié auparavant | 2.5.190.0, 2.6.1.0 | Toujours répertorié dans l’historique des versions ; solution de contournement via un bindingRedirect manuel, voir [article sur la version 2.6.84.0](/blog/entra-connect-2-6-84-0) |

La version retirée 2.6.79.0 reste indisponible. Microsoft renvoie désormais les personnes concernées vers la version 2.6.92.0 : désinstaller, puis installer la version actuelle.

## Délais de support

Chaque version 2.x n’est plus prise en charge douze mois après la publication de la version suivante. Les deux nouvelles versions entraînent les échéances suivantes :

| Version | Publiée le | Fin de support |
|---|---|---|
| 2.5.79.0 | 1er septembre 2025 | 23 octobre 2026 (expiré) |
| 2.5.190.0 | 19 novembre 2025 | 2 février 2027 |
| 2.6.1.0 | 2 février 2026 | 10 mars 2027 |
| 2.6.3.0 | 10 mars 2026 | 7 juillet 2027 |
| 2.6.84.0 | 7 juillet 2026 | 16 septembre 2027 |
| 2.6.91.0 | 16 septembre 2026 | 23 septembre 2027 |
| 2.6.92.0 | 23 septembre 2026 | encore inconnu |

Pour les versions 2.5.190.0 à 2.6.3.0, l’échéance du 7 avril 2027 s’applique en outre : elles sont inférieures à la version minimale requise 2.6.84.0 et ne se synchroniseront plus après cette date, bien que la fin de support normale de la version 2.6.3.0 ne soit prévue qu’en juillet 2027. La version 2.5.79.0 n’est plus prise en charge depuis le 23 octobre 2026 ; selon Microsoft, les versions qui ne sont plus prises en charge peuvent cesser de fonctionner de manière inattendue.

## Procédure recommandée

La version cible est 2.6.92.0. Elle comprend les correctifs de sécurité des deux versions, résout tous les problèmes connus des versions 2.6.84.0 et 2.6.91.0 et satisfait à la version minimale pour l’échéance du 7 avril 2027. Le programme d’installation est uniquement disponible dans le Centre d’administration Microsoft Entra, sous **Microsoft Entra Connect | Get started**, onglet **Manage** ; les prérequis sont .NET Framework 4.7.2 et TLS 1.2. Avant la mise à niveau, vérifiez les stratégies Conditional Access ciblant des applications pour `Microsoft.Azure.SyncFabric` et le Microsoft 365 Reporting Service, ainsi que les scripts appelant les cmdlets de configuration cloud mentionnés. Pour des mises à niveau à faible risque, Microsoft recommande la Swing Migration : mettez d’abord à jour le serveur de préproduction, vérifiez la configuration après Full Import et Full Synchronization, échangez ensuite les rôles, puis mettez à jour l’ancien serveur actif. Selon Microsoft, les organisations qui n’ont pas effectué de mise à jour depuis 12 à 18 mois devraient suivre cette approche. Après la mise à niveau, convertissez les serveurs existants qui signalent encore `ServiceAccount` vers l’authentification basée sur les applications, et contrôlez le filtrage des domaines et des unités d’organisation sur les serveurs qui exécutaient auparavant la version 2.6.84.0.

## Sources

1.  [Microsoft Entra Connect: Version release history](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history): Notes de publication des versions 2.6.91.0 et 2.6.92.0, problèmes connus de la version 2.6.84.0, tableau de retrait et échéance du 7 avril 2027.

2.  [Microsoft Entra Connect: Upgrade from a previous version](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-upgrade-previous-version): Méthodes de mise à niveau, Swing Migration via un serveur de préproduction et avertissement de mise à niveau obligatoire.

3.  [Authenticate to Microsoft Entra ID by using application identity](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/authenticate-application-id): Conversion manuelle des serveurs existants vers ABA, `Get-ADSyncEntraConnectorCredential` et suppression du compte hérité.
