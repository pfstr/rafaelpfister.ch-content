---
title: "Outlook New réinitialisé après une mise à jour Windows : comptes disparus, cause dans le journal"
navTitle: "Réinitialisation d’Outlook"
description: "Après un redémarrage consécutif à une mise à jour, le nouvel Outlook pour Windows démarre comme s’il venait d’être installé, avec tous les comptes et paramètres absents. La cause peut être retracée dans le journal Olk : une vérification activée côté serveur (Flight nh-userDataSentinel) ne trouve pas un fichier marqueur et supprime les données locales sans demander de confirmation. Cet article présente les entrées de journal, explique comment établir le constat sur vos propres appareils et indique quoi faire après la réinitialisation."
date: "2026-11-11"
kategorie: "Outlook"
timeToRead: "6 min de lecture"
themen:
  - outlook
  - windows-client
produkte:
  - "outlook"
  - "windows-client"
protokolle:
  - "troubleshooting"
slug: "outlook-new-reinitialise-apres-une-mise-a-jour-windows-comptes-disparus-cause-dans-le-journal"
translationId: "article-c1340c0bc1862647"
translationOf: outlook-new-reset-nach-update
url: https://rafaelpfister.ch/fr/blog/outlook-new-reinitialise-apres-une-mise-a-jour-windows-comptes-disparus-cause-dans-le-journal
translationSourceHash: 405d93eb98233f046e7ecb2823969a601db0041bb6ddaef8eeaa4c7d858e393c
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:27:32.181Z
translationReview: automatic
---

# Outlook New réinitialisé après une mise à jour Windows : comptes disparus, cause dans le journal

Après le redémarrage pour les mises à jour Windows de novembre 2026, le nouvel Outlook pour Windows s’est lancé sur l’un de mes appareils comme après une nouvelle installation. Le compte professionnel configuré comme compte principal avait disparu, tout comme les autres comptes, les signatures et les paramètres personnels. Aucun message d’erreur n’est apparu, Outlook a directement affiché l’assistant d’ajout d’un compte.

La cause figure dans le journal local d’Outlook : l’application a elle-même supprimé ses données locales parce qu’une vérification nouvellement activée a évalué l’état comme « réinitialisation requise ». Rien n’indique une intervention de Windows Update, d’un utilisateur ou d’un attaquant.

## Déroulement

| Heure (locale) | Événement | Source |
|---|---|---|
| 18:26 | Dernière entrée de la session Outlook en cours, sans arrêt ordonné | `Olk\logs\olk_*.log` |
| 18:27 et 18:28 | Deux redémarrages déclenchés par Windows Update (`MoNotificationUx.exe`, puis `TrustedInstaller.exe`) | Journal système, événement 1074 |
| 18:41:55 | Outlook démarre, la vérification `nh-userDataSentinel` est active et signale `ResetRequired` | nouveau `olk_*.log` |
| 18:41:56 | Outlook supprime ses données locales et démarre comme après une nouvelle installation | nouveau `olk_*.log` |

Les horodatages du journal Olk et des noms de fichiers journaux sont en UTC. Dans le tableau, ils sont convertis en heure locale (CET).

## Les entrées du journal

Les premières lignes de la nouvelle session montrent la vérification et son résultat :

```text
Checking flight enabled status  flightName nh-userDataSentinel  isEnabled true
Reset sentinel check completed.  installState ResetRequired
  settingsFound true  resetSentinelFlagFound true
Reset sentinel check triggered user data reset.
Prompted for clear local state  clearLocalStateName Silent
```

Les opérations de suppression suivent, chacune avec `succeeded true` :

| Chemin sous `%LOCALAPPDATA%\Microsoft\Olk` | Contenu |
|---|---|
| `EBWebView` | Profil WebView2 avec état de connexion, liste des comptes et données hors ligne |
| `UserSettings.json` | Paramètres de l’application |
| `Attachments` | Pièces jointes mises en cache localement |
| `pst_index_v2` | Index des fichiers PST intégrés |
| `cache\Notification`, `cache\*.db` | Cache des notifications et de l’arrière-plan |
| `updated.txt`, `xpdApi.log`, `Feedback` | Fichiers d’état et auxiliaires |

Quelques lignes plus loin figure `Last update date file missing, creating it. Barring clearing local state, this is likely a fresh install`. À partir de ce point, Outlook se comporte comme après une nouvelle installation.

## Pourquoi la vérification s’est déclenchée

`nh-userDataSentinel` est un Flight, c’est-à-dire une fonctionnalité que Microsoft active côté serveur pour certaines installations. Dans tous les anciens journaux de l’appareil, ce Flight affiche `Flight status not set; using default (false)`, pour la dernière fois à 13:23 UTC le même jour. Lors du démarrage à 17:41 UTC, il était actif pour la première fois.

La vérification recherche un fichier marqueur dans le dossier de package de l’application :

```text
%LOCALAPPDATA%\Packages\Microsoft.OutlookForWindows_8wekyb3d8bbwe\LocalState\OlkSentinel
```

Si Outlook trouve des paramètres utilisateur dans le dossier Olk (`settingsFound true`), mais aucun marqueur correspondant dans le dossier de package, il l’interprète comme l’indice que l’application a été réinitialisée ou réinstallée et supprime les données utilisateur. Sur l’appareil concerné, `OlkSentinel` a été créé à nouveau à 18:41:56, tandis que les autres fichiers de `LocalState` datent de septembre 2026. Le dossier de package lui-même n’a donc pas été réinitialisé ; le fichier marqueur n’existait simplement pas auparavant, car la vérification était nouvelle. Au premier démarrage avec le Flight actif, le résultat a donc été une réinitialisation, alors qu’elle n’était pas nécessaire.

D’après les journaux, le redémarrage déclenché par Windows Update n’est que le moment où cela s’est produit, pas la cause : le Flight est évalué au démarrage, et le premier démarrage après la mise à jour était le premier avec le Flight actif. La version de l’application (1.2026.1026.100) était identique avant et après le redémarrage.

## Établir le constat sur vos propres appareils

La requête suivante recherche dans tous les journaux Olk les entrées liées à la vérification :

```powershell
$logs = "$env:LOCALAPPDATA\Microsoft\Olk\logs"
Get-ChildItem $logs -Filter 'olk_*.log' |
  Select-String -Pattern 'Reset sentinel|Delete directory tree' |
  Select-Object Filename, LineNumber, Line
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Filter 'olk_*.log'` | uniquement les journaux de session de l’application, sans archives ZIP ni journaux Push |
| `-Pattern` | expression régulière ; correspond à la vérification et à chaque opération de suppression |
| `Select-Object Filename, LineNumber, Line` | affiche pour chaque correspondance le fichier journal, le numéro de ligne et le texte |

</details>

Outlook place les anciennes sessions dans des archives ZIP du même dossier. Si la réinitialisation remonte à quelques jours, l’entrée peut ne se trouver que dans l’archive et doit d’abord être extraite.

Les heures de création constituent un second indice : si `EBWebView` et `UserSettings.json` dans le dossier Olk, ainsi que `OlkSentinel` dans le dossier de package ont le même horodatage, la réinitialisation a eu lieu à ce moment-là.

```powershell
$olk = "$env:LOCALAPPDATA\Microsoft\Olk"
$pkg = "$env:LOCALAPPDATA\Packages\Microsoft.OutlookForWindows_8wekyb3d8bbwe"
Get-Item "$olk\EBWebView", "$olk\UserSettings.json", "$pkg\LocalState\OlkSentinel" |
  Select-Object FullName, CreationTime
```

## Ce qui est perdu et ce qui ne l’est pas

Seules les données locales de l’application ont été supprimées. Les boîtes aux lettres, calendriers et contacts sont stockés sur le serveur dans Microsoft 365 et Outlook.com et réapparaissent après une nouvelle connexion. La connexion à Windows (Web Account Manager) est conservée ; Outlook propose directement lors de l’ajout les comptes Microsoft enregistrés dans Windows.

Vous devez reconfigurer :

- tous les comptes, dans l’ordre souhaité (le premier compte ajouté est le compte principal, voir ci-dessous)
- les signatures enregistrées localement, si elles ne reviennent pas via la synchronisation cloud du compte
- les fichiers PST intégrés
- les paramètres de l’application, tels que l’affichage, les notifications et le comportement au démarrage

Pour le compte principal, l’ordre est déterminant, car le nouvel Outlook ne propose certaines fonctions que dans le compte principal, par exemple S/MIME (voir [Signature S/MIME dans le compte secondaire](/blog/outlook-smime-sekundaeres-konto-anhaenge-fehlen)). Ajoutez donc en premier le compte qui doit être le compte principal. Si Outlook a déjà repris automatiquement un autre compte après la réinitialisation, supprimez-le, ajoutez le compte principal souhaité, puis reconnectez les autres comptes.

## Pour les administrateurs

Il n’est pas possible d’annuler la réinitialisation après qu’elle s’est produite ; une sauvegarde du dossier `%LOCALAPPDATA%\Microsoft\Olk` avant le démarrage est la seule possibilité de restaurer l’état précédent. Dans les environnements utilisant des conteneurs de profil (FSLogix) et des sessions VDI non persistantes, vérifiez que le dossier Olk et le dossier de package `Microsoft.OutlookForWindows_8wekyb3d8bbwe` sont tous deux conservés dans le profil. Si un seul des deux dossiers est conservé, cela correspond exactement à l’état que la vérification interprète comme une nouvelle installation.

Les signalements d’utilisateurs selon lesquels le nouvel Outlook aurait « oublié tous les comptes » peuvent être analysés en quelques minutes avec la requête de journal ci-dessus. Si `Reset sentinel check triggered user data reset` y figure, il ne s’agit ni d’un problème de compte ni d’un incident de sécurité.

## Sources

1.  [Microsoft Learn: Deploy the new Outlook for Windows on VDI](https://learn.microsoft.com/en-us/microsoft-365-apps/outlook/get-started/virtualized-desktop-infrastructure): emplacements des données locales du nouvel Outlook et indications concernant les conteneurs de profil.

2.  [Microsoft Learn: Control installing and using the new Outlook](https://learn.microsoft.com/en-us/microsoft-365-apps/outlook/get-started/control-install): déploiement et gestion de l’application dans les environnements administrés.

3.  [Microsoft Q&A: Outlook (new) won't open, makes me attempt to sign in](https://learn.microsoft.com/en-us/answers/questions/5863503/outlook-new-wont-open-makes-me-attempt-to-sign-in): témoignages d’utilisateurs sur des comptes perdus après la réinitialisation des données locales.
