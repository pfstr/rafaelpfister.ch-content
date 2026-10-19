---
title: "Interruzioni di Exchange dopo gli aggiornamenti di Windows Server di ottobre 2026: MapiExceptionMdbOffline (ec=1142)"
navTitle: "Interruzione di Exchange 10/2026"
description: "Dopo gli aggiornamenti di Windows Server del 13 ottobre 2026, i server Exchange segnalano MapiExceptionMdbOffline (ec=1142) dopo circa 24 ore: OWA non funziona, le code crescono, le copie DAG non replicano più. La causa è un nuovo limite nel RPC Endpoint Mapper; dal 17 ottobre 2026 Microsoft consiglia il valore del Registro EpmapSavedContextCap, ma manca ancora una correzione. Sintomi, KB interessati, comandi di verifica e procedura."
date: "2026-10-16"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "10 min di lettura"
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
translationOf: exchange-windows-updates-oktober-2026
translationSourceHash: 47c39c05826230354e6b431330725587440880dac2b6a33c00f5d34da2afe2d7
translationModel: gpt-5.6-terra
translatedAt: 2026-10-19T07:50:11.831Z
translationReview: required
url: https://rafaelpfister.ch/it/blog/interruzioni-di-exchange-dopo-gli-aggiornamenti-di-windows-server-di-ottobre-2026
---

# Interruzioni di Exchange dopo gli aggiornamenti di Windows Server di ottobre 2026: MapiExceptionMdbOffline (ec=1142)

Dal Patch Tuesday del 13 ottobre 2026, i server Exchange smettono di funzionare dopo circa 24 ore di attività: i client perdono la connessione alla cassetta postale, OWA segnala `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)`, le e-mail rimangono nelle code e, nei DAG, le copie dei database non replicano più. Il 15 ottobre 2026 Microsoft ha integrato l'annuncio delle SU di Exchange di ottobre con un avviso: il problema è innescato dagli aggiornamenti di ottobre per Windows Server; per il problema non è necessaria l'installazione della SU di Exchange. Dal 17 ottobre 2026 sono note la causa e una contromisura ufficiale tramite Registro; non è ancora disponibile una correzione. I dettagli sono nella sezione seguente.

## Aggiornamento del 19 ottobre 2026: causa e contromisura nel Registro

Il team Exchange ha pubblicato un proprio articolo sul problema il 17 ottobre 2026. Nelle note di rilascio degli aggiornamenti Windows (ad es. KB5126646), il problema è ora indicato come Known Issue con stato «non risolto».

### Causa

Gli aggiornamenti di ottobre limitano la memoria utilizzata da RPC Endpoint Mapper per le chiamate di ricerca degli endpoint. Per impostazione predefinita sono ora consentiti solo 512 contesti salvati; secondo Microsoft, ciò non è sufficiente per carichi quali Exchange. Durante il funzionamento, Exchange accumula tali contesti fino al raggiungimento del limite. Successivamente le connessioni RPC al database delle cassette postali non riescono e OWA segnala `MapiExceptionMdbOffline`. Un lettore di Born ha dimostrato il nuovo valore di configurazione `EpmapSavedContextCap` nella `RpcEpMap.dll` degli aggiornamenti di ottobre. Frankys Web attribuisce il limite alla vulnerabilità Denial of Service CVE-2026-96300 nel RPC Endpoint Mapper, corretta nello stesso Patch Tuesday.

Ciò spiega diverse osservazioni dei primi giorni:

- L'interruzione si verifica solo dopo alcune ore e il momento dipende dal carico. Secondo Microsoft, alcuni server non sono mai interessati.
- Il riavvio del server azzera il contatore e quindi aiuta solo finché il limite non viene nuovamente raggiunto. Secondo un lettore di Born, il riavvio dei soli servizi Exchange non libera la memoria occupata.
- Il limite è contenuto negli aggiornamenti cumulativi di Windows. La SU di Exchange e l'aggiornamento .NET non sono coinvolti.
- Il sospetto collegamento con le attività di manutenzione del database e il servizio di replica non è quindi la causa.

### Contromisura

Microsoft consiglia di aumentare il limite tramite Registro e di riavviare quindi il server. In un prompt dei comandi con diritti di amministratore:

```cmd
reg add "HKLM\SOFTWARE\Microsoft\Rpc" /v EpmapSavedContextCap /t REG_DWORD /d 65536 /f
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `"HKLM\SOFTWARE\Microsoft\Rpc"` | Chiave del Registro della configurazione RPC. |
| `/v EpmapSavedContextCap` | Nome del valore che stabilisce il limite per i contesti salvati di Endpoint Mapper (senza voce si applica 512). |
| `/t REG_DWORD` | Tipo di dati: numero a 32 bit. |
| `/d 65536` | Nuovo limite, come consigliato da Microsoft. |
| `/f` | Sovrascrive un valore esistente senza chiedere conferma. |

</details>

Endpoint Mapper legge il valore solo all'avvio; senza riavviare il server, la modifica non ha effetto. Se il problema si ripresenta, Microsoft consiglia di aumentare ulteriormente il valore. Secondo Microsoft, la misura ha risolto il problema nei casi dei clienti gestiti finora. Secondo Born, Nino Bilic (Microsoft) definisce la voce una soluzione temporanea che non rappresenta un rischio per la sicurezza, ma non è la soluzione definitiva. Un lettore di Born segnala un server in funzione senza interruzioni da oltre 26 ore con 65536. Alcuni amministratori impostano valori più elevati, come 16777215; finora non è documentato alcun caso in cui 65536 non sia stato sufficiente dopo un riavvio.

Questa query in PowerShell mostra se il valore è impostato:

```powershell
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Rpc" -Name EpmapSavedContextCap
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-Path "HKLM:\SOFTWARE\Microsoft\Rpc"` | Chiave del Registro in cui si trova il valore. |
| `-Name EpmapSavedContextCap` | Legge solo questo valore; se manca, PowerShell segnala un errore e si applica il limite predefinito di 512. |

</details>

Impostate il valore su tutti i server Exchange, compresi quelli che finora non hanno subito interruzioni. Sui server che non dispongono ancora degli aggiornamenti di ottobre, potete impostare il valore prima dell'installazione; il riavvio successivo all'installazione lo attiverà. In un DAG, riavviate i membri uno dopo l'altro come descritto più avanti. Frankys Web fornisce inoltre uno script PowerShell che imposta il valore, salva lo stato precedente e può essere annullato con `-Remove`.

Secondo Microsoft, gli aggiornamenti Windows non devono essere disinstallati esplicitamente. Un lettore di Frankys Web segnala inoltre problemi di accesso RDP (CredSSP) dopo la disinstallazione di KB5126868. Se la misura nel Registro non è sufficiente, Microsoft offre un'ulteriore contromisura tramite Support for business. Non appena sarà disponibile una correzione, dovreste rimuovere nuovamente il valore del Registro, poiché il limite più elevato indebolisce la protezione contro CVE-2026-96300.

### Questioni aperte

- **Correzione**: Microsoft non indica una data. Un lettore di Born ipotizza che Microsoft possa annullare il limite tramite un Known Issue Rollback (Feature-ID 63096240); non vi è conferma.
- **Valore**: Non è chiaro se 65536 sia sufficiente anche in ambienti di grandi dimensioni fino al prossimo Patch Tuesday. Continuate quindi a monitorare l'Event ID 1006.
- **Altri ruoli server**: Un lettore di Born rileva che sono interessati anche server senza Exchange. Microsoft si esprime solo su Exchange.
- **Backup**: Non è chiarito se l'errore VSS Writer segnalato sia una conseguenza dell'interruzione.

## Sintomi

I sintomi riportati finora:

- OWA non funziona con `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)`; nel registro Application compare l'Event ID 1006.
- I client Outlook perdono la connessione alla cassetta postale.
- Le e-mail non vengono recapitate né internamente né esternamente e le code di trasporto crescono.
- Nei DAG, le copie dei database non replicano più; secondo le segnalazioni dei lettori di Born, gli switchover durante il passaggio alla modalità di manutenzione non riescono.
- Exchange Admin Center mostra comunque i database come montati (Frankys Web).

L'errore non si verifica direttamente dopo l'installazione ma, secondo le segnalazioni, dopo circa 22-30 ore di attività. Il riavvio del server ripristina il funzionamento, ma l'interruzione si ripete dopo circa lo stesso tempo. Nei commenti su Born si osserva che il servizio di replica delle cassette postali termina durante o dopo le attività di manutenzione del database e che in seguito iniziano gli errori MdbOffline; alla luce della causa ormai nota, si tratta di un effetto collaterale, non dell'innesco.

## Aggiornamenti interessati

Microsoft cita nel supplemento all'annuncio di Exchange questi aggiornamenti cumulativi di Windows:

| Windows Server | Aggiornamento |
| --- | --- |
| Windows Server 2025 | KB5126646 |
| Windows Server 2022 | KB5126655 |
| Windows Server 2019 | KB5126651 |
| Windows Server 2016 | KB5126868 |

Nello stesso Patch Tuesday sono state pubblicate anche le SU di Exchange (Exchange SE: KB5128895, «SU11») e gli aggiornamenti cumulativi per .NET Framework. In un Exchange SE interessato su Windows Server 2019 erano installati questi tre pacchetti:

- Security Update For Exchange Server Subscription Edition (KB5128895)
- 2026-10 Cumulative Update for .NET Framework 3.5, 4.7.2 and 4.8 for Windows Server 2019 for x64 (KB5128923)
- 2026-10 Cumulative Update for Windows Server 2019 for x64-based Systems (KB5126651)

Microsoft non menziona l'aggiornamento .NET nel supplemento. I lettori di Born riferiscono tuttavia di interruzioni con la combinazione di aggiornamento Windows e .NET (in quel caso KB5128928 su Windows Server 2022) e di un server con KB5126651 e SU di Exchange, ma senza aggiornamento .NET, che funzionava ancora dopo oltre 24 ore. Non è quindi chiaro se l'aggiornamento .NET sia coinvolto.

Riguardo alla SU di Exchange: nei forum e nei commenti si afferma che la situazione migliora se è installata Exchange SU11. Ciò non può essere confermato; le interruzioni si sono verificate anche su un server con SU11. Viceversa, diversi amministratori segnalano interruzioni senza una SU di Exchange installata, in linea con quanto affermato da Microsoft. La SU di Exchange chiude sei vulnerabilità, tra cui un'esecuzione di codice remoto critica tramite un'e-mail appositamente predisposta; dettagli nell'[articolo sulla SU di ottobre](/blog/exchange-security-updates-oktober-2026). Ritardarne l'installazione non risolve il problema.

## Verificare lo stato

Per prima cosa, individuate su ogni server Exchange quali aggiornamenti di ottobre sono installati:

```powershell
$kbs = "KB5126646", "KB5126655", "KB5126651", "KB5126868", "KB5128923"
Get-HotFix -Id $kbs -ErrorAction SilentlyContinue |
  Select-Object PSComputerName, HotFixID, InstalledOn
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `$kbs`, `-Id` | Elenco dei numeri KB cercati: le quattro CU di Windows dal supplemento Microsoft e l'aggiornamento .NET per Windows Server 2019. |
| `-ErrorAction SilentlyContinue` | Elimina il messaggio di errore per i numeri KB non installati su questo server. |
| `Select-Object PSComputerName, HotFixID, InstalledOn` | Mostra server, numero KB e data di installazione; dalla data è possibile stimare quando attendersi la successiva interruzione. |

</details>

`Get-HotFix` non elenca gli aggiornamenti .NET in modo affidabile su tutti i sistemi; in caso di dubbio, la cronologia degli aggiornamenti nelle Impostazioni mostra lo stato completo. Verificate la build di Exchange in Exchange Management Shell con `Get-ExchangeServer | Format-Table Name, AdminDisplayVersion` e confrontatela con i [numeri di build di Exchange](/tools/exchange-builds).

Quindi verificate lo stato di database, copie e code in Exchange Management Shell:

```powershell
Get-MailboxDatabaseCopyStatus -Server $env:COMPUTERNAME |
  Format-Table Name, Status, CopyQueueLength, ReplayQueueLength, ContentIndexState
Get-Queue | Where-Object MessageCount -gt 0 |
  Format-Table Identity, DeliveryType, Status, MessageCount, LastError
Get-WinEvent -FilterHashtable @{LogName='Application'; Id=1006; StartTime=(Get-Date).AddDays(-2)} |
  Select-Object TimeCreated, ProviderName, Message
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `Get-MailboxDatabaseCopyStatus -Server $env:COMPUTERNAME` | Stato di tutte le copie di database sul server locale; senza DAG compare solo la copia attiva. |
| `CopyQueueLength`, `ReplayQueueLength` | Numero di file di log non ancora copiati o non ancora riprodotti; valori in aumento indicano che la replica è ferma. |
| `ContentIndexState` | Stato dell'indice di ricerca della copia. |
| `Get-Queue \| Where-Object MessageCount -gt 0` | Mostra solo le code che contengono e-mail. |
| `LastError` | Ultimo errore della coda; in questo problema, tipicamente errori di connessione al database delle cassette postali. |
| `-FilterHashtable @{LogName='Application'; Id=1006; …}` | Cerca nel registro Application l'Event ID 1006, come indicato da Microsoft. |
| `StartTime=(Get-Date).AddDays(-2)` | Limita la ricerca agli ultimi due giorni, ossia al periodo dal Patch Tuesday. |

</details>

L'Event ID 1006 viene registrato da più componenti di Exchange; è determinante la voce con il messaggio MdbOffline nel testo.

## Procedura fino alla correzione

I punti seguenti risalgono al 16 ottobre 2026, quando non esisteva ancora una contromisura ufficiale. Dal 17 ottobre 2026, la misura nel Registro descritta nell'aggiornamento sopra riportato è al primo posto; i punti da 3 a 6 restano validi come soluzione di riserva.

1. **Server Exchange non ancora aggiornati**: Inizialmente la raccomandazione era di rimandare gli aggiornamenti Windows di ottobre. Con la misura nel Registro non è più necessario: impostate il valore e installate gli aggiornamenti che chiudono anch'essi vulnerabilità di sicurezza.

2. **Installare comunque la SU di Exchange**: Secondo Microsoft, la SU di Exchange KB5128895 (o i pacchetti ESU per 2016/2019) non è l'innesco e chiude una vulnerabilità critica sfruttabile tramite e-mail senza account.

3. **Server già interessati**: Un riavvio ripristina il funzionamento per circa un giorno. Pianificate riavvii controllati a intervalli nettamente inferiori a 22 ore, ad esempio ogni notte, anziché attendere la successiva interruzione. Secondo i lettori di Born, anche il riavvio dei soli servizi Exchange aiuta temporaneamente; per quanto tempo non è documentato.

4. **DAG**: Riavviate i membri uno dopo l'altro e spostate prima i database attivi su un server appena riavviato. Dopo ogni riavvio, verificate con `Get-MailboxDatabaseCopyStatus` che le copie siano nuovamente `Healthy` e che le code vengano smaltite prima di riavviare il server successivo.

5. **Monitorare le code**: Tenete sotto controllo `Get-Queue` e l'Event ID 1006, idealmente con un avviso, affinché un'interruzione non venga rilevata solo dalle segnalazioni degli utenti.

6. **Verificare il backup**: Un lettore di Born segnala un VSS Writer non riuscito nel backup dopo l'interruzione. Controllate i backup delle ultime notti.

Per la disinstallazione degli aggiornamenti Windows: dal 2021 Microsoft distribuisce l'aggiornamento cumulativo insieme al Servicing Stack Update come pacchetto combinato. `wusa /uninstall` non rimuove un pacchetto di questo tipo; a tale scopo è necessario `DISM /Online /Remove-Package` con il nome del pacchetto ottenuto da `DISM /Online /Get-Packages`. Dal 17 ottobre 2026 Microsoft sconsiglia esplicitamente la disinstallazione; con la misura nel Registro non vi è più alcun motivo per farlo.

Microsoft ora elenca il problema come Known Issue nelle note di rilascio degli aggiornamenti Windows (stato 19 ottobre 2026, non risolto). Questo articolo verrà aggiornato non appena sarà disponibile una correzione.

## Fonti

1.  [Released: October 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-october-2026-exchange-server-security-updates/4563014): Supplemento del 15 ottobre 2026 con gli aggiornamenti Windows interessati, messaggio di errore, Event ID 1006 e l'indicazione che la SU di Exchange non è necessaria per il problema.

2.  [Exchange Server SE: Problems with the October updates for Windows Server – Frankys Web](https://www.frankysweb.de/en/exchange-server-se-problems-with-the-october-update-kb5128895/): Interruzione dopo circa 24 ore, database ancora visualizzati come montati nell'EAC, riavvio come soluzione temporanea, commenti sui riavvii DAG.

3.  [Achtung: Exchange Server SE Okt. 2026-SU führen zu Ausfällen – Borns IT- und Windows-Blog](https://borncity.com/blog/2026/10/15/achtung-exchange-server-okt-2026-su-fuehren-zu-ausfaellen/): Segnalazioni dei lettori su tempistica, KB coinvolti (anche .NET), servizio di replica, modalità di manutenzione ed errori VSS.

4.  [Oktober-Updates setzen Exchange-Server ausser Betrieb – IT Magazine](https://www.itmagazine.ch/artikel/88263/Oktober-Updates_setzen_Exchange-Server_ausser_Betrieb.html): Riepilogo svizzero della conferma di Microsoft.

5.  [Exchange servers may become unstable after installation of October 2026 Windows updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/exchange-servers-may-become-unstable-after-installation-of-october-2026-windows-/4565382): Articolo del team Exchange del 17 ottobre 2026 con il valore del Registro EpmapSavedContextCap, la raccomandazione di non disinstallare gli aggiornamenti Windows e l'avviso sugli effetti dipendenti dal carico.

6.  [October 13, 2026: KB5126646 (OS Build 26100.33650) – Microsoft Support](https://support.microsoft.com/en-us/servicing/os/windows-server/2026/10/kb5126646-windows-server-2025-security-update): Known Issue con il limite di 512 chiamate di ricerca degli endpoint, stato non risolto, ulteriore contromisura tramite Microsoft Support for business.

7.  [Microsoft analysiert Exchange-Server-Probleme durch Windows-Oktober-2026-Updates – Borns IT- und Windows-Blog](https://borncity.com/blog/2026/10/17/microsoft-analysiert-exchange-server-probleme-durch-windows-oktober-2026-updates/): Analisi del lettore di RpcEpMap.dll (EpmapSavedContextCap, Feature-ID), dichiarazione di Nino Bilic, esperienze con 65536 e valori più elevati.

8.  [CVE-2026-96300 – OpenCVE](https://app.opencve.io/cve/CVE-2026-96300): Vulnerabilità Denial of Service del servizio RPC Endpoint Mapper, CWE-770, pubblicata il 13 ottobre 2026.

9.  [Aggiornamenti di sicurezza Exchange di ottobre 2026 – rafaelpfister.ch](/blog/exchange-security-updates-oktober-2026): Contenuto della SU di Exchange KB5128895 e dei pacchetti ESU, vulnerabilità e indicazioni per l'installazione.
