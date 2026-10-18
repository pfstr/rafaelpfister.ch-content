---
title: "Interruzioni di Exchange dopo gli aggiornamenti di Windows Server di ottobre 2026: MapiExceptionMdbOffline (ec=1142)"
navTitle: "Interruzione di Exchange 10/2026"
description: "Dopo gli aggiornamenti di Windows Server del 13 ottobre 2026, i server Exchange segnalano MapiExceptionMdbOffline (ec=1142) dopo circa 24 ore: OWA non funziona, le code crescono e le copie DAG non replicano più. Microsoft conferma il collegamento con gli aggiornamenti di Windows, ma non è ancora disponibile una correzione. Sintomi, KB interessati, comandi di verifica e procedura."
date: "2026-10-16"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "7 min di lettura"
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
slug: "interruzioni-di-exchange-dopo-gli-aggiornamenti-di-windows-server-di-ottobre-2026"
featured: "2026-10-16"
warnung: true
translationId: "article-8fdf2b2744bdbd41"
aiPrompt: |
  Du bist mein Assistent für Exchange Server On-Premises. Seit den Windows-Server-Updates vom Oktober 2026 (KB5126646, KB5126655, KB5126651, KB5126868) melden Exchange-Server MapiExceptionMdbOffline (hr=0x80004005, ec=1142), Event ID 1006, wachsende Warteschlangen und nicht replizierende DAG-Kopien. Hilf mir Schritt für Schritt: ermitteln, welche Oktober-Updates (Windows, .NET Framework, Exchange SU) auf jedem Exchange-Server installiert sind, den Zustand von Datenbanken, Kopien und Warteschlangen prüfen, den Betrieb mit kontrollierten Neustarts überbrücken und die Windows-Updates auf noch nicht betroffenen Servern zurückstellen. Frage zuerst nach Exchange-Version, Windows-Server-Version, Anzahl Server, ob eine DAG im Einsatz ist und wie Updates verteilt werden (WSUS, Intune, manuell).
translationOf: exchange-windows-updates-oktober-2026
url: https://rafaelpfister.ch/it/blog/interruzioni-di-exchange-dopo-gli-aggiornamenti-di-windows-server-di-ottobre-2026
translationSourceHash: 562adaae54d08dd594a344f91f79a05160af956ab320978cef34097a075cab52
translationModel: gpt-5.6-terra
translatedAt: 2026-10-16T08:18:46.684Z
translationReview: required
---

# Interruzioni di Exchange dopo gli aggiornamenti di Windows Server di ottobre 2026: MapiExceptionMdbOffline (ec=1142)

Dal Patch Tuesday del 13 ottobre 2026, i server Exchange smettono di funzionare dopo circa 24 ore di attività: i client perdono la connessione alla cassetta postale, OWA segnala `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)`, i messaggi rimangono nelle code e, nelle DAG, le copie dei database non replicano più. Il 15 ottobre 2026 Microsoft ha aggiunto una nota all'annuncio delle SU di Exchange di ottobre: la causa sono gli aggiornamenti di ottobre per Windows Server; l'installazione della SU di Exchange non è necessaria affinché si verifichi il problema. Non sono ancora disponibili una correzione né una soluzione alternativa ufficiale (al 16 ottobre 2026).

## Sintomi

I sintomi segnalati finora:

- OWA non funziona e restituisce `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)`; nel registro Application compare l'evento ID 1006.
- I client Outlook perdono la connessione alla cassetta postale.
- I messaggi non vengono recapitati né internamente né esternamente e le code di trasporto crescono.
- Nelle DAG, le copie dei database non replicano più; secondo le segnalazioni dei lettori su Born, lo switchover durante il passaggio alla modalità di manutenzione non riesce.
- L'Exchange Admin Center continua comunque a mostrare i database come montati (Frankys Web).

L'errore non si verifica subito dopo l'installazione, bensì, secondo le segnalazioni, dopo circa 22-30 ore di attività. Il riavvio del server ripristina il funzionamento, ma l'interruzione si ripete dopo all'incirca lo stesso periodo. Nei commenti su Born è stato osservato che il servizio di replica delle cassette postali termina durante o dopo le attività di manutenzione dei database e che successivamente iniziano gli errori MdbOffline; ciò non è confermato.

## Aggiornamenti interessati

Nell'integrazione all'annuncio di Exchange, Microsoft indica questi aggiornamenti cumulativi di Windows:

| Windows Server | Aggiornamento |
| --- | --- |
| Windows Server 2025 | KB5126646 |
| Windows Server 2022 | KB5126655 |
| Windows Server 2019 | KB5126651 |
| Windows Server 2016 | KB5126868 |

Nello stesso Patch Tuesday sono state pubblicate anche le SU di Exchange (Exchange SE: KB5128895, «SU11») e aggiornamenti cumulativi per .NET Framework. Su un Exchange SE interessato con Windows Server 2019 erano installati questi tre pacchetti:

- Security Update For Exchange Server Subscription Edition (KB5128895)
- 2026-10 Cumulative Update for .NET Framework 3.5, 4.7.2 and 4.8 for Windows Server 2019 for x64 (KB5128923)
- 2026-10 Cumulative Update for Windows Server 2019 for x64-based Systems (KB5126651)

Microsoft non menziona l'aggiornamento .NET nell'integrazione. Tuttavia, i lettori su Born segnalano interruzioni con la combinazione di aggiornamento Windows e .NET (in quel caso KB5128928 su Windows Server 2022) e un server con KB5126651 e la SU di Exchange, ma senza aggiornamento .NET, che era ancora in funzione dopo oltre 24 ore. Rimane quindi da chiarire se l'aggiornamento .NET sia coinvolto.

Riguardo alla SU di Exchange: nei forum e nei commenti si afferma che la situazione migliori se è installata Exchange SU11. Ciò non può essere confermato; le interruzioni si sono verificate anche su un server con SU11. Viceversa, diversi amministratori segnalano interruzioni senza una SU di Exchange installata, in linea con quanto dichiarato da Microsoft. La SU di Exchange risolve sei vulnerabilità, inclusa un'esecuzione di codice remoto critica tramite un'e-mail appositamente predisposta; dettagli nell'[articolo sulla SU di ottobre](/blog/exchange-security-updates-oktober-2026). Non installarla non risolve il problema.

## Verificare lo stato

Per prima cosa, identificate su ogni server Exchange quali aggiornamenti di ottobre sono installati:

```powershell
$kbs = "KB5126646", "KB5126655", "KB5126651", "KB5126868", "KB5128923"
Get-HotFix -Id $kbs -ErrorAction SilentlyContinue |
  Select-Object PSComputerName, HotFixID, InstalledOn
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `$kbs`, `-Id` | Elenco dei numeri KB cercati: le quattro CU Windows indicate nell'integrazione di Microsoft e l'aggiornamento .NET per Windows Server 2019. |
| `-ErrorAction SilentlyContinue` | Sopprime il messaggio di errore per i numeri KB non installati su questo server. |
| `Select-Object PSComputerName, HotFixID, InstalledOn` | Mostra server, numero KB e data di installazione; dalla data è possibile stimare quando attendersi la prossima interruzione. |

</details>

`Get-HotFix` non elenca in modo affidabile gli aggiornamenti .NET su tutti i sistemi; in caso di dubbio, la cronologia degli aggiornamenti nelle impostazioni mostra lo stato completo. Verificate la build di Exchange nella Exchange Management Shell con `Get-ExchangeServer | Format-Table Name, AdminDisplayVersion` e confrontatela con i [numeri di build di Exchange](/tools/exchange-builds).

Successivamente, verificate lo stato di database, copie e code nella Exchange Management Shell:

```powershell
Get-MailboxDatabaseCopyStatus -Server $env:COMPUTERNAME |
  Format-Table Name, Status, CopyQueueLength, ReplayQueueLength, ContentIndexState
Get-Queue | Where-Object MessageCount -gt 0 |
  Format-Table Identity, DeliveryType, Status, MessageCount, LastError
Get-WinEvent -FilterHashtable @{LogName='Application'; Id=1006; StartTime=(Get-Date).AddDays(-2)} |
  Select-Object TimeCreated, ProviderName, Message
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Get-MailboxDatabaseCopyStatus -Server $env:COMPUTERNAME` | Stato di tutte le copie dei database sul server locale; senza DAG viene visualizzata solo la copia attiva. |
| `CopyQueueLength`, `ReplayQueueLength` | Numero di file di log non ancora copiati o riprodotti; valori crescenti indicano che la replica è ferma. |
| `ContentIndexState` | Stato dell'indice di ricerca della copia. |
| `Get-Queue \| Where-Object MessageCount -gt 0` | Mostra solo le code che contengono messaggi. |
| `LastError` | Ultimo errore della coda; in questo problema, tipicamente errori di connessione al database delle cassette postali. |
| `-FilterHashtable @{LogName='Application'; Id=1006; …}` | Cerca nel registro Application l'evento ID 1006, come indicato da Microsoft. |
| `StartTime=(Get-Date).AddDays(-2)` | Limita la ricerca agli ultimi due giorni, cioè al periodo successivo al Patch Tuesday. |

</details>

Più componenti di Exchange registrano l'evento ID 1006; è determinante la voce con il messaggio MdbOffline nel testo.

## Procedura fino alla correzione

Microsoft dichiara di stare esaminando il problema insieme al team Windows e finora non indica né una soluzione alternativa né un rollback. Nel frattempo:

1. **Server Exchange non ancora aggiornati**: Rinviate gli aggiornamenti Windows di ottobre (e l'aggiornamento .NET dello stesso giorno) sui server Exchange, in WSUS, Intune o nella vostra gestione degli aggiornamenti. Si tratta di una valutazione: gli aggiornamenti Windows risolvono anch'essi vulnerabilità di sicurezza. Per questo motivo, i server Exchange raggiungibili da Internet devono essere monitorati particolarmente da vicino.

2. **Installare comunque la SU di Exchange**: Secondo Microsoft, la SU di Exchange KB5128895 (o i pacchetti ESU per 2016/2019) non è la causa e risolve una vulnerabilità critica sfruttabile via e-mail senza un account.

3. **Server già interessati**: Un riavvio ripristina il funzionamento per circa un giorno. Pianificate riavvii controllati a intervalli nettamente inferiori a 22 ore, ad esempio ogni notte, anziché attendere la prossima interruzione. Secondo i lettori su Born, anche il riavvio dei soli servizi Exchange aiuta temporaneamente; non è documentato per quanto tempo.

4. **DAG**: Riavviate i membri uno dopo l'altro e spostate prima i database attivi su un server appena riavviato. Dopo ogni riavvio, verificate con `Get-MailboxDatabaseCopyStatus` che le copie siano nuovamente `Healthy` e che le code vengano smaltite prima di riavviare il server successivo.

5. **Monitorare le code**: Tenete sotto controllo `Get-Queue` e l'evento ID 1006, idealmente con un avviso, affinché un'interruzione non emerga solo dalle segnalazioni degli utenti.

6. **Verificare il backup**: Un lettore su Born segnala un VSS Writer non riuscito nel backup dopo l'interruzione. Controllate i backup delle ultime notti.

Per disinstallare gli aggiornamenti Windows: dal 2021 Microsoft distribuisce l'aggiornamento cumulativo insieme al Servicing Stack Update come pacchetto combinato. `wusa /uninstall` non rimuove tale pacchetto; è invece necessario `DISM /Online /Remove-Package` con il nome del pacchetto da `DISM /Online /Get-Packages`. Microsoft non ha approvato la disinstallazione come soluzione alternativa e finora non sono disponibili segnalazioni di un rollback riuscito con successivo funzionamento stabile. Se intendete testarlo, fatelo prima su un singolo server.

Secondo Frankys Web, Microsoft non elenca ancora il problema come Known Issue negli aggiornamenti Windows (al 16 ottobre 2026). Tenete d'occhio l'annuncio Microsoft; Microsoft inserisce le integrazioni in fondo alla pagina sotto «Updates to this blog post». Questo articolo verrà aggiornato non appena sarà disponibile una correzione o una soluzione alternativa ufficiale.

## Fonti

1.  [Released: October 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-october-2026-exchange-server-security-updates/4563014): integrazione del 15 ottobre 2026 con gli aggiornamenti Windows interessati, il messaggio di errore, l'evento ID 1006 e l'indicazione che la SU di Exchange non è necessaria affinché si verifichi il problema.

2.  [Exchange Server SE: Problems with the October updates for Windows Server – Frankys Web](https://www.frankysweb.de/en/exchange-server-se-problems-with-the-october-update-kb5128895/): interruzione dopo circa 24 ore, database ancora mostrati come montati nell'EAC, riavvio come rimedio temporaneo, commenti sui riavvii DAG.

3.  [Achtung: Exchange Server SE Okt. 2026-SU führen zu Ausfällen – Borns IT- und Windows-Blog](https://borncity.com/blog/2026/10/15/achtung-exchange-server-okt-2026-su-fuehren-zu-ausfaellen/): segnalazioni dei lettori su tempistica, KB coinvolti (compreso .NET), servizio di replica, modalità di manutenzione ed errori VSS.

4.  [Oktober-Updates setzen Exchange-Server ausser Betrieb – IT Magazine](https://www.itmagazine.ch/artikel/88263/Oktober-Updates_setzen_Exchange-Server_ausser_Betrieb.html): sintesi svizzera della conferma di Microsoft.

5.  [Aggiornamenti di sicurezza Exchange di ottobre 2026 – rafaelpfister.ch](/blog/exchange-security-updates-oktober-2026): contenuto della SU di Exchange KB5128895 e dei pacchetti ESU, vulnerabilità e istruzioni di installazione.
