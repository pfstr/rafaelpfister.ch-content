---
title: "Pannes d’Exchange après les mises à jour Windows Server d’octobre 2026 : MapiExceptionMdbOffline (ec=1142)"
navTitle: "Panne Exchange 10/2026"
description: "Après les mises à jour Windows Server du 13 octobre 2026, les serveurs Exchange signalent MapiExceptionMdbOffline (ec=1142) après environ 24 heures : OWA échoue, les files d’attente grossissent et les copies DAG ne se répliquent plus. Microsoft confirme le lien avec les mises à jour Windows, mais aucun correctif n’est encore disponible. Symptômes, KB concernés, commandes de vérification et procédure."
date: "2026-10-16"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "7 min de lecture"
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
url: https://rafaelpfister.ch/fr/blog/pannes-d-exchange-apres-les-mises-a-jour-windows-server-d-octobre-2026-mapiexceptionmdboffline
translationSourceHash: 562adaae54d08dd594a344f91f79a05160af956ab320978cef34097a075cab52
translationModel: gpt-5.6-terra
translatedAt: 2026-10-16T08:18:24.493Z
translationReview: automatic
---

# Pannes d’Exchange après les mises à jour Windows Server d’octobre 2026 : MapiExceptionMdbOffline (ec=1142)

Depuis le Patch Tuesday du 13 octobre 2026, les serveurs Exchange tombent en panne après environ 24 heures de fonctionnement : les clients perdent la connexion à leur boîte aux lettres, OWA affiche `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)`, les e-mails restent dans les files d’attente et, dans les DAG, les copies de bases de données ne se répliquent plus. Le 15 octobre 2026, Microsoft a ajouté une note à l’annonce des SU Exchange d’octobre : les mises à jour d’octobre pour Windows Server sont à l’origine du problème ; l’installation du SU Exchange n’est pas nécessaire pour que le problème survienne. Aucun correctif ni contournement officiel n’est encore disponible (état au 16 octobre 2026).

## Symptômes

Les symptômes rapportés jusqu’à présent :

- OWA échoue avec `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)` ; l’Application Log contient l’Event ID 1006.
- Les clients Outlook perdent la connexion à la boîte aux lettres.
- Les e-mails ne sont livrés ni en interne ni en externe, et les files d’attente de transport grossissent.
- Dans les DAG, les copies de bases de données ne se répliquent plus ; selon des témoignages de lecteurs sur Born, les basculements lors du passage en mode maintenance échouent.
- Le Centre d’administration Exchange affiche néanmoins les bases de données comme montées (Frankys Web).

L’erreur ne survient pas directement après l’installation, mais, selon les rapports, après environ 22 à 30 heures de fonctionnement. Le redémarrage du serveur rétablit le fonctionnement, mais la panne se reproduit après approximativement le même délai. Dans les commentaires sur Born, il est observé que le service de réplication des boîtes aux lettres s’arrête pendant ou après les tâches de maintenance des bases de données, puis que les erreurs MdbOffline apparaissent ; cela n’est toutefois pas confirmé.

## Mises à jour concernées

Dans l’ajout à l’annonce Exchange, Microsoft cite les mises à jour cumulatives Windows suivantes :

| Windows Server | Mise à jour |
| --- | --- |
| Windows Server 2025 | KB5126646 |
| Windows Server 2022 | KB5126655 |
| Windows Server 2019 | KB5126651 |
| Windows Server 2016 | KB5126868 |

Les SU Exchange (Exchange SE : KB5128895, « SU11 ») ainsi que des mises à jour cumulatives pour le .NET Framework ont également été publiés le même Patch Tuesday. Sur un Exchange SE affecté sous Windows Server 2019, les trois paquets suivants étaient installés :

- Security Update For Exchange Server Subscription Edition (KB5128895)
- 2026-10 Cumulative Update for .NET Framework 3.5, 4.7.2 and 4.8 for Windows Server 2019 for x64 (KB5128923)
- 2026-10 Cumulative Update for Windows Server 2019 for x64-based Systems (KB5126651)

Microsoft ne mentionne pas la mise à jour .NET dans l’ajout. Des lecteurs sur Born signalent toutefois des pannes avec la combinaison des mises à jour Windows et .NET (KB5128928 sous Windows Server 2022) ainsi qu’un serveur avec KB5126651 et le SU Exchange, mais sans mise à jour .NET, qui fonctionnait toujours après plus de 24 heures. La participation de la mise à jour .NET reste donc incertaine.

Concernant le SU Exchange : des forums et commentaires indiquent que la situation s’améliorerait avec l’installation du SU11 Exchange. Cela ne peut pas être confirmé ; les pannes sont également survenues sur un serveur avec SU11. Inversement, plusieurs administrateurs signalent des pannes sans SU Exchange installé, ce qui correspond à la déclaration de Microsoft. Le SU Exchange corrige six vulnérabilités, dont une exécution de code à distance critique via un e-mail spécialement conçu ; détails dans l’[article sur le SU d’octobre](/blog/exchange-security-updates-oktober-2026). Le différer ne résout pas le problème.

## Vérifier l’état

Commencez par déterminer, sur chaque serveur Exchange, lesquelles des mises à jour d’octobre sont installées :

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
| `-ErrorAction SilentlyContinue` | Supprime le message d’erreur pour les numéros KB non installés sur ce serveur. |
| `Select-Object PSComputerName, HotFixID, InstalledOn` | Affiche le serveur, le numéro KB et la date d’installation ; la date permet d’estimer quand la prochaine panne est à prévoir. |

</details>

`Get-HotFix` ne répertorie pas les mises à jour .NET de manière fiable sur tous les systèmes ; en cas de doute, l’historique des mises à jour dans les paramètres affiche l’état complet. Vérifiez la version d’Exchange dans l’Exchange Management Shell avec `Get-ExchangeServer | Format-Table Name, AdminDisplayVersion` et comparez-la aux [numéros de build Exchange](/tools/exchange-builds).

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
| `CopyQueueLength`, `ReplayQueueLength` | Nombre de fichiers journaux qui n’ont pas encore été copiés ou rejoués ; des valeurs croissantes indiquent que la réplication est arrêtée. |
| `ContentIndexState` | État de l’index de recherche de la copie. |
| `Get-Queue \| Where-Object MessageCount -gt 0` | Affiche uniquement les files d’attente contenant des e-mails. |
| `LastError` | Dernière erreur de la file d’attente ; pour ce problème, généralement une erreur de connexion à la base de données de boîtes aux lettres. |
| `-FilterHashtable @{LogName='Application'; Id=1006; …}` | Recherche l’Event ID 1006 dans l’Application Log, comme indiqué par Microsoft. |
| `StartTime=(Get-Date).AddDays(-2)` | Limite la recherche aux deux derniers jours, donc à la période depuis le Patch Tuesday. |

</details>

Plusieurs composants Exchange consignent l’Event ID 1006 ; l’entrée déterminante est celle contenant le message MdbOffline dans le texte.

## Procédure en attendant le correctif

Microsoft indique examiner le problème avec l’équipe Windows et ne mentionne jusqu’ici ni contournement ni rollback. Dans l’intervalle :

1. **Serveurs Exchange pas encore mis à jour** : Suspendez les mises à jour Windows d’octobre (ainsi que la mise à jour .NET du même jour) sur les serveurs Exchange, dans WSUS, Intune ou votre propre gestion des mises à jour. C’est un compromis : les mises à jour Windows corrigent elles aussi des vulnérabilités de sécurité. Les serveurs Exchange accessibles depuis Internet doivent donc faire l’objet d’une surveillance particulièrement étroite.

2. **Installer tout de même le SU Exchange** : Selon Microsoft, le SU Exchange KB5128895 (ou les paquets ESU pour 2016/2019) n’est pas le déclencheur et corrige une faille critique exploitable sans compte par e-mail.

3. **Serveurs déjà affectés** : Un redémarrage rétablit le fonctionnement pendant environ une journée. Planifiez des redémarrages contrôlés à des intervalles nettement inférieurs à 22 heures, par exemple chaque nuit, au lieu d’attendre la prochaine panne. Selon des lecteurs sur Born, le redémarrage des seuls services Exchange aide aussi temporairement ; sa durée d’efficacité n’est pas établie.

4. **DAG** : Redémarrez les membres l’un après l’autre et déplacez au préalable les bases de données actives vers un serveur fraîchement redémarré. Après chaque redémarrage, vérifiez avec `Get-MailboxDatabaseCopyStatus` que les copies sont à nouveau `Healthy` et que les files d’attente se résorbent avant de redémarrer le serveur suivant.

5. **Surveiller les files d’attente** : Gardez `Get-Queue` et l’Event ID 1006 à l’œil, idéalement avec une alerte, afin qu’une panne ne soit pas détectée seulement via des signalements d’utilisateurs.

6. **Vérifier les sauvegardes** : Un lecteur sur Born signale un VSS Writer défaillant dans la sauvegarde après la panne. Contrôlez les sauvegardes des dernières nuits.

Pour désinstaller les mises à jour Windows : depuis 2021, Microsoft fournit la mise à jour cumulative avec la Servicing Stack Update sous la forme d’un paquet combiné. `wusa /uninstall` ne supprime pas un tel paquet ; il faut utiliser `DISM /Online /Remove-Package` avec le nom du paquet provenant de `DISM /Online /Get-Packages`. Microsoft n’a pas validé la désinstallation comme contournement et aucun rapport de rollback réussi suivi d’un fonctionnement stable n’est encore disponible. Le cas échéant, testez d’abord cette procédure sur un seul serveur.

Selon Frankys Web, Microsoft ne répertorie pas encore le problème comme Known Issue dans les mises à jour Windows (état au 16 octobre 2026). Gardez l’annonce Microsoft à l’œil ; Microsoft y ajoute les compléments à la fin, sous « Updates to this blog post ». Cet article sera mis à jour dès qu’un correctif ou un contournement officiel sera disponible.

## Sources

1.  [Released: October 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-october-2026-exchange-server-security-updates/4563014): Ajout du 15 octobre 2026 avec les mises à jour Windows concernées, le message d’erreur, l’Event ID 1006 et l’indication que le SU Exchange n’est pas nécessaire pour que le problème survienne.

2.  [Exchange Server SE: Problems with the October updates for Windows Server – Frankys Web](https://www.frankysweb.de/en/exchange-server-se-problems-with-the-october-update-kb5128895/): Panne après environ 24 heures, bases de données toujours affichées comme montées dans l’EAC, redémarrage comme solution temporaire, commentaires sur les redémarrages DAG.

3.  [Achtung: Exchange Server SE Okt. 2026-SU führen zu Ausfällen – Borns IT- und Windows-Blog](https://borncity.com/blog/2026/10/15/achtung-exchange-server-okt-2026-su-fuehren-zu-ausfaellen/): Témoignages de lecteurs concernant le moment de la panne, les KB impliqués (y compris .NET), le service de réplication, le mode maintenance et les erreurs VSS.

4.  [Oktober-Updates setzen Exchange-Server ausser Betrieb – IT Magazine](https://www.itmagazine.ch/artikel/88263/Oktober-Updates_setzen_Exchange-Server_ausser_Betrieb.html): Résumé suisse de la confirmation de Microsoft.

5.  [Exchange-Sicherheitsupdates vom Oktober 2026 – rafaelpfister.ch](/blog/exchange-security-updates-oktober-2026): Contenu du SU Exchange KB5128895 et des paquets ESU, vulnérabilités et indications d’installation.
