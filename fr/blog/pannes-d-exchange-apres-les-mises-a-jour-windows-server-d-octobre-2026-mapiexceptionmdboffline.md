---
title: "Pannes d’Exchange après les mises à jour Windows Server d’octobre 2026 : MapiExceptionMdbOffline (ec=1142)"
navTitle: "Panne Exchange 10/2026"
description: "Après les mises à jour Windows Server du 13 octobre 2026, les serveurs Exchange signalent MapiExceptionMdbOffline (ec=1142) après environ 24 heures : OWA échoue, les files d’attente augmentent et les copies DAG ne se répliquent plus. La cause est une nouvelle limite dans le RPC Endpoint Mapper ; depuis le 17 octobre 2026, Microsoft recommande la valeur de registre EpmapSavedContextCap, mais aucun correctif n’est encore disponible. Symptômes, KB concernées, commandes de vérification et procédure."
date: "2026-10-16"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "10 min de lecture"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
  - "exchange-on-premises"
protokolle:
  - "troubleshooting"
  - "releases"
  - "powershell"
slug: "pannes-d-exchange-apres-les-mises-a-jour-windows-server-d-octobre-2026-mapiexceptionmdboffline"
featured: "2026-10-16"
warnung: true
translationId: "article-8fdf2b2744bdbd41"
aiPrompt: |
  Du bist mein Assistent für Exchange Server On-Premises. Seit den Windows-Server-Updates vom Oktober 2026 (KB5126646, KB5126655, KB5126651, KB5126868) melden Exchange-Server MapiExceptionMdbOffline (hr=0x80004005, ec=1142), Event ID 1006, wachsende Warteschlangen und nicht replizierende DAG-Kopien. Hilf mir Schritt für Schritt: ermitteln, welche Oktober-Updates (Windows, .NET Framework, Exchange SU) auf jedem Exchange-Server installiert sind, den Zustand von Datenbanken, Kopien und Warteschlangen prüfen, den Betrieb mit kontrollierten Neustarts überbrücken und die Windows-Updates auf noch nicht betroffenen Servern zurückstellen. Frage zuerst nach Exchange-Version, Windows-Server-Version, Anzahl Server, ob eine DAG im Einsatz ist und wie Updates verteilt werden (WSUS, Intune, manuell).
translationOf: exchange-windows-updates-oktober-2026
translationSourceHash: 47c39c05826230354e6b431330725587440880dac2b6a33c00f5d34da2afe2d7
translationModel: gpt-5.6-terra
translatedAt: 2026-10-19T07:49:39.046Z
translationReview: automatic
url: https://rafaelpfister.ch/fr/blog/pannes-d-exchange-apres-les-mises-a-jour-windows-server-d-octobre-2026-mapiexceptionmdboffline
---

# Pannes d’Exchange après les mises à jour Windows Server d’octobre 2026 : MapiExceptionMdbOffline (ec=1142)

Depuis le Patch Tuesday du 13 octobre 2026, des serveurs Exchange tombent en panne après environ 24 heures de fonctionnement : les clients perdent la connexion à leur boîte aux lettres, OWA affiche `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)`, les e-mails restent dans les files d’attente et, dans les DAG, les copies de bases de données ne se répliquent plus. Microsoft a complété l’annonce des SU Exchange d’octobre le 15 octobre 2026 par une remarque : les mises à jour Windows Server d’octobre sont à l’origine du problème ; l’installation du SU Exchange n’est pas nécessaire pour qu’il survienne. Depuis le 17 octobre 2026, la cause et une mesure officielle via le registre sont connues ; aucun correctif n’est encore disponible. Les détails figurent dans la section suivante.

## Mise à jour du 19 octobre 2026 : cause et mesure via le registre

L’équipe Exchange a publié son propre article sur le problème le 17 octobre 2026. Dans les notes de publication des mises à jour Windows (par ex. KB5126646), le problème est désormais indiqué comme un problème connu avec le statut « non résolu ».

### Cause

Les mises à jour d’octobre limitent la mémoire utilisée par le RPC Endpoint Mapper pour les appels de recherche d’endpoint. Par défaut, seuls 512 contextes enregistrés sont désormais autorisés ; selon Microsoft, cela ne suffit pas pour des charges telles qu’Exchange. Exchange accumule ces contextes pendant son fonctionnement jusqu’à ce que la limite soit atteinte. Les connexions RPC à la base de données de boîtes aux lettres échouent alors et OWA affiche `MapiExceptionMdbOffline`. Un lecteur de Born a mis en évidence la nouvelle valeur de configuration `EpmapSavedContextCap` dans la `RpcEpMap.dll` des mises à jour d’octobre. Frankys Web attribue cette limite à la vulnérabilité de déni de service CVE-2026-96300 du RPC Endpoint Mapper, corrigée lors du même Patch Tuesday.

Cela explique plusieurs observations des premiers jours :

- La panne ne survient qu’après plusieurs heures, et le moment dépend de la charge. Selon Microsoft, certains serveurs ne sont jamais affectés.
- Le redémarrage du serveur réinitialise le compteur et n’aide donc que jusqu’à ce que la limite soit à nouveau atteinte. Selon un lecteur de Born, redémarrer uniquement les services Exchange ne libère pas la mémoire occupée.
- La limite se trouve dans les mises à jour cumulatives Windows. Le SU Exchange et la mise à jour .NET ne sont pas impliqués.
- Le lien présumé avec les tâches de maintenance de bases de données et le service de réplication n’est donc pas la cause.

### Mesure corrective

Microsoft recommande d’augmenter la limite via le registre, puis de redémarrer le serveur. Dans une invite de commandes avec des droits d’administrateur :

```cmd
reg add "HKLM\SOFTWARE\Microsoft\Rpc" /v EpmapSavedContextCap /t REG_DWORD /d 65536 /f
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `"HKLM\SOFTWARE\Microsoft\Rpc"` | Clé de registre de la configuration RPC. |
| `/v EpmapSavedContextCap` | Nom de la valeur qui définit la limite des contextes Endpoint Mapper enregistrés (en l’absence d’entrée, la valeur est 512). |
| `/t REG_DWORD` | Type de données : nombre 32 bits. |
| `/d 65536` | Nouvelle limite, comme recommandé par Microsoft. |
| `/f` | Écrase une valeur existante sans demander de confirmation. |

</details>

L’Endpoint Mapper ne lit cette valeur qu’au démarrage ; sans redémarrage du serveur, la modification ne prend pas effet. Si le problème se reproduit, Microsoft recommande d’augmenter davantage la valeur. Selon Microsoft, cette mesure a résolu le problème dans les cas clients traités jusqu’à présent. D’après Born, Nino Bilic (Microsoft) décrit cette entrée comme une solution temporaire qui ne présente pas de risque de sécurité, mais qui ne constitue pas la solution définitive. Un lecteur de Born rapporte un serveur fonctionnant sans interruption depuis plus de 26 heures avec la valeur 65536. Certains administrateurs utilisent des valeurs plus élevées, comme 16777215 ; aucun cas documenté où 65536 n’aurait pas suffi après un redémarrage n’a été rapporté jusqu’à présent.

Cette requête PowerShell permet de vérifier si la valeur est définie :

```powershell
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Rpc" -Name EpmapSavedContextCap
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-Path "HKLM:\SOFTWARE\Microsoft\Rpc"` | Clé de registre dans laquelle se trouve la valeur. |
| `-Name EpmapSavedContextCap` | Lit uniquement cette valeur ; si elle est absente, PowerShell signale une erreur et la limite par défaut de 512 s’applique. |

</details>

Définissez la valeur sur tous les serveurs Exchange, y compris ceux qui ne sont pas encore tombés en panne. Sur les serveurs qui n’ont pas encore reçu les mises à jour d’octobre, vous pouvez définir la valeur avant l’installation ; le redémarrage suivant l’installation l’activera. Dans un DAG, redémarrez les membres l’un après l’autre comme décrit ci-dessous. Frankys Web fournit également un script PowerShell qui définit la valeur, sauvegarde l’état précédent et peut être annulé avec `-Remove`.

Selon Microsoft, les mises à jour Windows ne doivent explicitement pas être désinstallées. Un lecteur de Frankys Web rapporte en outre des problèmes de connexion RDP (CredSSP) après la désinstallation de KB5126868. Si la mesure via le registre ne suffit pas, Microsoft propose une autre mesure via le Support for business. Dès qu’un correctif sera disponible, vous devriez supprimer à nouveau la valeur de registre, car la limite plus élevée affaiblit la protection contre CVE-2026-96300.

### Points en suspens

- **Correctif** : Microsoft ne donne aucune échéance. Un lecteur de Born suppose que Microsoft pourrait supprimer la limite via un Known Issue Rollback (Feature-ID 63096240) ; cela n’est pas confirmé.
- **Valeur** : Il n’est pas certain que 65536 suffise également dans les grands environnements jusqu’au prochain Patch Tuesday. Continuez donc de surveiller l’Event ID 1006.
- **Autres rôles de serveur** : Un lecteur de Born observe également des serveurs sans Exchange affectés. Microsoft ne s’exprime qu’au sujet d’Exchange.
- **Sauvegarde** : Il n’est pas établi si l’erreur VSS Writer signalée est une conséquence de la panne.

## Symptômes

Les symptômes rapportés jusqu’à présent :

- OWA échoue avec `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)` ; l’Application Log contient l’Event ID 1006.
- Les clients Outlook perdent la connexion à leur boîte aux lettres.
- Les e-mails ne sont distribués ni en interne ni en externe, les files d’attente de transport augmentent.
- Dans les DAG, les copies de bases de données ne se répliquent plus ; selon les témoignages de lecteurs chez Born, les basculements lors du passage en mode maintenance échouent.
- L’Exchange Admin Center affiche néanmoins les bases de données comme montées (Frankys Web).

L’erreur ne survient pas immédiatement après l’installation, mais, selon les rapports, après environ 22 à 30 heures de fonctionnement. Un redémarrage du serveur rétablit le service, mais la panne se répète après approximativement le même délai. Dans les commentaires chez Born, il a été observé que le service de réplication des boîtes aux lettres s’arrête pendant ou après les tâches de maintenance des bases de données et que les erreurs MdbOffline commencent ensuite ; compte tenu de la cause désormais connue, il s’agit d’un effet secondaire et non du déclencheur.

## Mises à jour concernées

Dans l’ajout à l’annonce Exchange, Microsoft cite les mises à jour cumulatives Windows suivantes :

| Windows Server | Mise à jour |
| --- | --- |
| Windows Server 2025 | KB5126646 |
| Windows Server 2022 | KB5126655 |
| Windows Server 2019 | KB5126651 |
| Windows Server 2016 | KB5126868 |

Lors du même Patch Tuesday, les SU Exchange (Exchange SE : KB5128895, « SU11 ») et des mises à jour cumulatives pour le .NET Framework ont également été publiés. Sur un Exchange SE affecté sous Windows Server 2019, ces trois paquets étaient installés :

- Security Update For Exchange Server Subscription Edition (KB5128895)
- 2026-10 Cumulative Update for .NET Framework 3.5, 4.7.2 and 4.8 for Windows Server 2019 for x64 (KB5128923)
- 2026-10 Cumulative Update for Windows Server 2019 for x64-based Systems (KB5126651)

Microsoft ne mentionne pas la mise à jour .NET dans l’ajout. Des lecteurs de Born rapportent toutefois des pannes avec la combinaison de mises à jour Windows et .NET (KB5128928 sous Windows Server 2022) ainsi qu’un serveur avec KB5126651 et le SU Exchange, mais sans mise à jour .NET, qui fonctionnait encore après plus de 24 heures. Il reste donc incertain que la mise à jour .NET soit impliquée.

Concernant le SU Exchange : des forums et commentaires indiquent que la situation s’améliore lorsque le SU11 Exchange est installé. Cela ne peut pas être confirmé ; les pannes sont également survenues sur un serveur avec SU11. À l’inverse, plusieurs administrateurs rapportent des pannes sans SU Exchange installé, ce qui correspond aux déclarations de Microsoft. Le SU Exchange corrige six vulnérabilités, dont une exécution de code à distance critique via un e-mail spécialement conçu ; détails dans l’[article sur le SU d’octobre](/blog/exchange-security-updates-oktober-2026). Le retenir ne résout pas le problème.

## Vérifier l’état

Commencez par déterminer sur chaque serveur Exchange lesquelles des mises à jour d’octobre sont installées :

```powershell
$kbs = "KB5126646", "KB5126655", "KB5126651", "KB5126868", "KB5128923"
Get-HotFix -Id $kbs -ErrorAction SilentlyContinue |
  Select-Object PSComputerName, HotFixID, InstalledOn
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `$kbs`, `-Id` | Liste des numéros KB recherchés : les quatre CU Windows de l’ajout Microsoft et la mise à jour .NET pour Windows Server 2019. |
| `-ErrorAction SilentlyContinue` | Supprime le message d’erreur pour les numéros KB qui ne sont pas installés sur ce serveur. |
| `Select-Object PSComputerName, HotFixID, InstalledOn` | Affiche le serveur, le numéro KB et la date d’installation ; cette date permet d’estimer quand la prochaine panne est à prévoir. |

</details>

`Get-HotFix` ne répertorie pas de manière fiable les mises à jour .NET sur tous les systèmes ; en cas de doute, l’historique des mises à jour dans les paramètres affiche l’état complet. Vérifiez la build Exchange dans l’Exchange Management Shell avec `Get-ExchangeServer | Format-Table Name, AdminDisplayVersion` et comparez-la aux [numéros de build Exchange](/tools/exchange-builds).

Vérifiez ensuite l’état des bases de données, des copies et des files d’attente dans l’Exchange Management Shell :

```powershell
Get-MailboxDatabaseCopyStatus -Server $env:COMPUTERNAME |
  Format-Table Name, Status, CopyQueueLength, ReplayQueueLength, ContentIndexState
Get-Queue | Where-Object MessageCount -gt 0 |
  Format-Table Identity, DeliveryType, Status, MessageCount, LastError
Get-WinEvent -FilterHashtable @{LogName='Application'; Id=1006; StartTime=(Get-Date).AddDays(-2)} |
  Select-Object TimeCreated, ProviderName, Message
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `Get-MailboxDatabaseCopyStatus -Server $env:COMPUTERNAME` | État de toutes les copies de bases de données sur le serveur local ; sans DAG, seule la copie active apparaît. |
| `CopyQueueLength`, `ReplayQueueLength` | Nombre de fichiers journaux pas encore copiés ou pas encore rejoués ; des valeurs en hausse indiquent que la réplication est arrêtée. |
| `ContentIndexState` | État de l’index de recherche de la copie. |
| `Get-Queue \| Where-Object MessageCount -gt 0` | Affiche uniquement les files d’attente contenant des e-mails. |
| `LastError` | Dernière erreur de la file d’attente ; avec ce problème, il s’agit typiquement d’erreurs de connexion à la base de données de boîtes aux lettres. |
| `-FilterHashtable @{LogName='Application'; Id=1006; …}` | Recherche l’Event ID 1006 dans l’Application Log, comme indiqué par Microsoft. |
| `StartTime=(Get-Date).AddDays(-2)` | Limite la recherche aux deux derniers jours, soit à la période depuis le Patch Tuesday. |

</details>

Plusieurs composants Exchange consignent l’Event ID 1006 ; l’élément déterminant est l’entrée contenant le message MdbOffline dans le texte.

## Procédure jusqu’au correctif

Les points suivants datent du 16 octobre 2026, lorsqu’aucune mesure officielle n’était encore disponible. Depuis le 17 octobre 2026, la mesure via le registre de la mise à jour ci-dessus est prioritaire ; les points 3 à 6 restent valables comme solution de repli.

1. **Serveurs Exchange pas encore mis à jour** : à l’origine, la recommandation était de reporter les mises à jour Windows d’octobre. Avec la mesure via le registre, ce n’est plus nécessaire : définissez la valeur et installez les mises à jour qui corrigent également des vulnérabilités de sécurité.

2. **Installer malgré tout le SU Exchange** : selon Microsoft, le SU Exchange KB5128895 (ou les paquets ESU pour 2016/2019) n’est pas le déclencheur et corrige une faille critique exploitable par e-mail sans compte.

3. **Serveurs déjà affectés** : un redémarrage rétablit le fonctionnement pendant environ une journée. Prévoyez des redémarrages contrôlés à intervalles nettement inférieurs à 22 heures, par exemple chaque nuit, au lieu d’attendre la prochaine panne. Selon des lecteurs de Born, redémarrer uniquement les services Exchange aide également temporairement ; la durée n’est pas établie.

4. **DAG** : redémarrez les membres l’un après l’autre et déplacez au préalable les bases de données actives vers un serveur fraîchement redémarré. Après chaque redémarrage, vérifiez avec `Get-MailboxDatabaseCopyStatus` que les copies sont à nouveau `Healthy` et que les files d’attente diminuent avant de redémarrer le serveur suivant.

5. **Surveiller les files d’attente** : surveillez `Get-Queue` et l’Event ID 1006, idéalement avec une alerte, afin qu’une panne ne soit pas détectée seulement par les signalements des utilisateurs.

6. **Vérifier les sauvegardes** : un lecteur de Born rapporte l’échec d’un VSS Writer dans la sauvegarde après la panne. Contrôlez les sauvegardes des dernières nuits.

Concernant la désinstallation des mises à jour Windows : depuis 2021, Microsoft fournit la mise à jour cumulative et la Servicing Stack Update sous forme de paquet combiné. `wusa /uninstall` ne supprime pas un tel paquet ; il faut pour cela utiliser `DISM /Online /Remove-Package` avec le nom du paquet issu de `DISM /Online /Get-Packages`. Depuis le 17 octobre 2026, Microsoft déconseille explicitement la désinstallation ; avec la mesure via le registre, il n’y a plus non plus de raison de le faire.

Microsoft répertorie désormais le problème comme un problème connu dans les notes de publication des mises à jour Windows (état au 19 octobre 2026, non résolu). Cet article sera mis à jour dès qu’un correctif sera disponible.

## Sources

1.  [Released: October 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-october-2026-exchange-server-security-updates/4563014): ajout du 15 octobre 2026 avec les mises à jour Windows concernées, le message d’erreur, l’Event ID 1006 et l’indication que le SU Exchange n’est pas nécessaire au problème.

2.  [Exchange Server SE: Problems with the October updates for Windows Server – Frankys Web](https://www.frankysweb.de/en/exchange-server-se-problems-with-the-october-update-kb5128895/): panne après environ 24 heures, bases de données toujours affichées comme montées dans l’EAC, redémarrage comme solution temporaire, commentaires concernant les redémarrages DAG.

3.  [Achtung: Exchange Server SE Okt. 2026-SU führen zu Ausfällen – Borns IT- und Windows-Blog](https://borncity.com/blog/2026/10/15/achtung-exchange-server-okt-2026-su-fuehren-zu-ausfaellen/): témoignages de lecteurs concernant le moment, les KB impliquées (y compris .NET), le service de réplication, le mode maintenance et les erreurs VSS.

4.  [Oktober-Updates setzen Exchange-Server ausser Betrieb – IT Magazine](https://www.itmagazine.ch/artikel/88263/Oktober-Updates_setzen_Exchange-Server_ausser_Betrieb.html): résumé suisse de la confirmation de Microsoft.

5.  [Exchange servers may become unstable after installation of October 2026 Windows updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/exchange-servers-may-become-unstable-after-installation-of-october-2026-windows-/4565382): article de l’équipe Exchange du 17 octobre 2026 avec la valeur de registre EpmapSavedContextCap, la recommandation de ne pas désinstaller les mises à jour Windows et la remarque concernant les effets dépendants de la charge.

6.  [October 13, 2026: KB5126646 (OS Build 26100.33650) – Microsoft Support](https://support.microsoft.com/en-us/servicing/os/windows-server/2026/10/kb5126646-windows-server-2025-security-update): problème connu avec la limite de 512 appels de recherche d’endpoint, statut non résolu, autre mesure disponible via Microsoft Support for business.

7.  [Microsoft analysiert Exchange-Server-Probleme durch Windows-Oktober-2026-Updates – Borns IT- und Windows-Blog](https://borncity.com/blog/2026/10/17/microsoft-analysiert-exchange-server-probleme-durch-windows-oktober-2026-updates/): analyse par un lecteur de RpcEpMap.dll (EpmapSavedContextCap, Feature-ID), déclaration de Nino Bilic, expériences avec 65536 et des valeurs plus élevées.

8.  [CVE-2026-96300 – OpenCVE](https://app.opencve.io/cve/CVE-2026-96300): vulnérabilité de déni de service du service RPC Endpoint Mapper, CWE-770, publiée le 13 octobre 2026.

9.  [Exchange-Sicherheitsupdates vom Oktober 2026 – rafaelpfister.ch](/blog/exchange-security-updates-oktober-2026): contenu du SU Exchange KB5128895 et des paquets ESU, vulnérabilités et instructions d’installation.
