---
title: "Aggiornamenti di sicurezza di Exchange di settembre 2026: nove vulnerabilità, risolto il problema dei wrapper, rilasciata la v2"
navTitle: "Exchange SU 09/2026"
description: "L’aggiornamento SU di settembre risolve nove vulnerabilità in Exchange SE e 2019 (otto in Exchange 2016), tra cui una falla di spoofing con CVSS 9.3, e corregge il problema dei wrapper negli ambienti ibridi. Il 2 ottobre è seguita una v2 con una CVE aggiuntiva; si aggiungono tre problemi noti con workaround e un SettingOverride che ora deve essere rimosso."
date: "2026-10-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "11 min di lettura"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
  - "powershell"
slug: "aggiornamenti-di-sicurezza-di-exchange-di-settembre-2026-nove-vulnerabilita-risolto-il-problema"
translationId: article-53db0c02bc33f9bd
translationOf: exchange-security-updates-september-2026
translationSourceHash: 4b2c59794f87e999c5b89865e0005faa7d5cbb36478f6b8475d5e3f121186a7d
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T08:30:40.269Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/aggiornamenti-di-sicurezza-di-exchange-di-settembre-2026-nove-vulnerabilita-risolto-il-problema
---

# Aggiornamenti di sicurezza di Exchange di settembre 2026: nove vulnerabilità, risolto il problema dei wrapper, rilasciata la v2

Microsoft ha pubblicato l’8 settembre 2026 aggiornamenti di sicurezza (SU) per Exchange Server. Correggono nove vulnerabilità in Exchange SE e Exchange 2019, otto in Exchange 2016. Nessuna era nota pubblicamente in precedenza, nessuna risulta sfruttata attivamente secondo il Security Update Guide e Microsoft le classifica tutte come *Important* con «Exploitation Less Likely». Tuttavia, il valore CVSS massimo di 9.3 è nettamente superiore a quello del mese precedente. Questo mese è particolare per tre motivi: l’SU risolve il problema dei *messaggi wrapper* nelle cassette postali condivise, aperto da giugno; introduce tre nuovi problemi noti o ancora persistenti; e il 2 ottobre Microsoft ha rilasciato una **Versione 2** che corregge un’ulteriore vulnerabilità.

## Per quali versioni di Exchange è disponibile l’aggiornamento

Gli SU dell’8 settembre 2026 sono disponibili per le seguenti versioni:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121608, build 15.2.2562.49; disponibile pubblicamente.
- **Exchange Server 2019 CU15**: KB5121609, build 15.2.1748.51; solo tramite il **programma ESU Period 2**.
- **Exchange Server 2019 CU14**: KB5121610, build 15.2.1544.46; solo tramite ESU Period 2.
- **Exchange Server 2016 CU23**: KB5121611, build 15.1.2507.73; solo tramite ESU Period 2.

Exchange 2016 e 2019 sono fuori supporto. Secondo Microsoft, gli SU da maggio a ottobre 2026 sono disponibili solo alle organizzazioni iscritte al programma ESU Period 2. Secondo gli articoli KB, questa idoneità si estende fino a ottobre 2026. Si aggiunge la pressione da Exchange Online: dalla seconda settimana di settembre, Exchange Online limita e blocca il flusso di posta ibrido dai server con una versione precedente a ottobre 2025; i dettagli sono nell’[articolo sul Transport Enforcement](/blog/exchange-online-transport-enforcement-hybrid-server). Exchange Online stesso è già protetto secondo l’annuncio; negli ambienti ibridi, tuttavia, ogni server Exchange necessita comunque dell’SU, così come le macchine con gli Exchange Management Tools.

È possibile confrontare la propria versione con la panoramica dei [numeri di build di Exchange](/tools/exchange-builds).

## Panoramica delle vulnerabilità

| CVE | Tipo | CVSS |
| --- | --- | --- |
| CVE-2026-69356 | Spoofing (Cross-Site Scripting) | 9.3 |
| CVE-2026-69641 | Elevation of Privilege | 9.1 |
| CVE-2026-69355 | Remote Code Execution | 8.8 |
| CVE-2026-55007 | Remote Code Execution | 8.1 |
| CVE-2026-69380 | Elevation of Privilege | 8.1 |
| CVE-2026-69378 | Denial of Service | 7.5 |
| CVE-2026-69361 | Spoofing (Server-Side Request Forgery) | 6.5 |
| CVE-2026-69375 | Tampering | 6.5 |
| CVE-2026-69382 | Information Disclosure | 5.9 |

CVE-2026-55007 non riguarda Exchange 2016; il Security Update Guide indica per questa CVE solo Exchange SE e 2019 CU14/CU15. Un dettaglio della documentazione: negli articoli KB dei quattro SU di settembre manca CVE-2026-69380 nell’elenco delle CVE. Il Security Update Guide indica però esattamente le quattro build di settembre come correzione per questa CVE (stato al 7 ottobre 2026).

**[CVE-2026-69356](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69356)** ha il valore più alto, CVSS 9.3. Secondo Microsoft, un aggressore non autenticato può inviare un invito al calendario predisposto con un link dannoso a una riunione; se il destinatario apre la riunione e seleziona il link per partecipare, viene attivato il Cross-Site Scripting. È quindi necessaria un’interazione dell’utente, ma non un account nell’organizzazione.

**[CVE-2026-69380](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69380)** (Elevation of Privilege, CVSS 8.1) richiede soltanto un account con pochi privilegi e una cassetta postale assegnata. Secondo le FAQ del Security Update Guide, un aggressore può impersonare un altro utente sfruttando debolezze nella verifica delle richieste e dei token di identità e assumere il controllo delle cassette postali di tutti gli utenti Exchange: leggere e inviare e-mail, nonché scaricare allegati. Come punto di partenza è sufficiente un singolo account utente compromesso.

**[CVE-2026-69641](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69641)** (Elevation of Privilege, CVSS 9.1) porta allo stesso risultato, il controllo di tutte le cassette postali, ma richiede l’appartenenza a un gruppo di ruoli con privilegi elevati.

Le due vulnerabilità di Remote Code Execution hanno presupposti diversi: [CVE-2026-69355](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69355) (CVSS 8.8) richiede un account autenticato con pochi privilegi; [CVE-2026-55007](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-55007) (CVSS 8.1) può essere attivata senza autenticazione tramite un allegato Visio predisposto, ma secondo Microsoft richiede memoria di lavoro persistentemente insufficiente nel sistema di destinazione. Le altre quattro falle: CVE-2026-69378 (DoS tramite ricorsione non controllata, senza autenticazione), CVE-2026-69361 (SSRF, il server invia richieste HTTP a sistemi interni o di loopback), CVE-2026-69375 (un aggressore autenticato può sostituire contenuti di file) e CVE-2026-69382 (divulgazione di credenziali tramite un algoritmo crittografico debole, richiede un cookie di autenticazione già sottratto).

## Versione 2 del 2 ottobre: aggiunta CVE-2026-96940

Il 2 ottobre 2026 Microsoft ha pubblicato la «Versione 2» degli SU di settembre. Secondo l’annuncio, l’unica differenza rispetto alla prima versione è la correzione aggiuntiva per **[CVE-2026-96940](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940). Questa vulnerabilità di Elevation of Privilege ha CVSS 8.8, non è nota pubblicamente né sfruttata, ma Microsoft è l’unica tra le dieci CVE a valutarla come **«Exploitation More Likely»**. Un aggressore autenticato può così accedere alle cassette postali di altri utenti della stessa organizzazione e leggere e-mail e allegati. Exchange Online è già stato corretto lato server.

| Versione | KB | Build v2 |
| --- | --- | --- |
| Exchange SE RTM | KB5129955 | 15.2.2562.53 |
| Exchange 2019 CU15 | KB5129956 | 15.2.1748.53 |
| Exchange 2019 CU14 | KB5129957 | 15.2.1544.48 |
| Exchange 2016 CU23 | KB5129958 | 15.1.2507.75 |

In pratica, la correzione per CVE-2026-96940 è contenuta solo nelle build v2. I server sui quali è già in esecuzione l’SU dell’8 settembre necessitano inoltre della v2. Chi applica ora la patch può installare direttamente la v2, poiché gli SU sono cumulativi. Gli articoli KB non dichiarano espressamente se i server con il primo SU di settembre debbano installare obbligatoriamente la v2; poiché la nuova CVE viene corretta solo con la v2, questa è l’interpretazione più ovvia.

## Problema dei wrapper risolto: rimuovere ora il SettingOverride

Il problema noto dall’SU di giugno, per cui negli ambienti ibridi compaiono *messaggi wrapper* nella posta in arrivo delle cassette postali condivise, è risolto con l’SU di settembre in tutte e quattro le versioni. Nell’[articolo di agosto](/blog/exchange-security-updates-august-2026) era ancora indicato che il SettingOverride documentato come workaround poteva rimanere. Dopo l’installazione dell’SU di settembre vale il contrario: Microsoft raccomanda, nell’articolo di supporto associato, di verificare e rimuovere l’override.

```powershell
Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Comando | Effetto |
|---|---|
| `Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Verifica se l’override del workaround è impostato nell’organizzazione. |
| `Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Rimuove l’override non appena è installato l’SU di settembre. |

</details>

Se il primo comando segnala che l’oggetto `DisableBlockSharedAndUserMailboxHeaders` non è stato trovato, secondo Microsoft non occorre fare altro.

Per Exchange SE, l’SU corregge inoltre un errore nelle query di disponibilità ibrida tramite Microsoft Graph: gli utenti on-premises vedevano gli orari occupati delle cassette postali di Exchange Online spostati del proprio offset UTC, senza messaggi di errore.

## Problemi noti

**I calendari pubblicati (.ics) restituiscono HTTP 500 alle app calendario.** Il problema persiste dall’SU di agosto (Exchange SE dalla build 15.2.2562.46, nonché Exchange 2019 e 2016) e non è risolto né nell’SU di settembre né nella v2. Gli abbonamenti a calendari pubblicati anonimamente non si aggiornano più; nel browser lo stesso URL funziona. Secondo Microsoft, la causa è che Exchange riconosce i client tramite lo User-Agent: le app calendario senza identificazione del browser finiscono in un percorso di codice disattivato dall’SU di agosto. Come workaround, Microsoft descrive una regola di riscrittura URL nel sito «Exchange Back End» di IIS, che per richieste .ics sotto `/owa/calendar/` aggiunge il parametro `layout=premium`; per questo deve essere installato il modulo IIS URL Rewrite. I passaggi esatti, tramite IIS Manager o direttamente in `applicationHost.config`, sono riportati nell’[articolo di supporto KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672). Microsoft non indica una data per la correzione (stato al 7 ottobre 2026).

**Disponibilità per cassette postali delegate in ambienti ibridi (solo Exchange SE).** Se la query di disponibilità è configurata esclusivamente tramite l’API Graph, le query per cassette postali di Exchange Online tramite accesso on-premises delegato falliscono. Outlook segnala «Your server location could not be determined», OWA visualizza «No information» e nei log EWS compare `(403) Forbidden`. Il workaround documentato inoltra nuovamente le query tramite EWS anziché tramite Graph:

```powershell
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=False"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-Identity EnableRouteThroughMSGraphFeature` | L’override che controlla l’inoltro delle query di disponibilità tramite Microsoft Graph. |
| `-Parameters "Enabled=False"` | Disattiva il percorso Graph; le query tornano a passare tramite EWS. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Invia la chiamata diagnostica al servizio di topologia. |
| `-Component VariantConfiguration -Argument Refresh` | Ricarica la Variant Configuration affinché l’override abbia effetto senza attese. |

</details>

Nell’annuncio della v2 Microsoft indica questo problema tra quelli risolti. Chi ha impostato il workaround dovrebbe verificare dopo l’installazione della v2 nell’[articolo di supporto KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) se debba essere annullato; al 7 ottobre 2026 non vi erano ancora istruzioni in merito.

**Deadlock di ContentEngine dovuto alla mancanza dei file di regole coreani di WordBreaker (solo Exchange SE).** Con l’SU di settembre (build 15.2.2562.49 e build v2 15.2.2562.53), i file di regole per il WordBreaker coreano aggiornato non vengono installati. Le conseguenze sono risultati di ricerca mancanti, recapito della posta ritardato, client Outlook o MAPI che si bloccano o perdono la connessione. [KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098) indica come interessati gli ambienti che elaborano e-mail con contenuti in coreano. Questo non dipende dalla lingua dei propri utenti: messaggi con testo coreano raggiungono anche organizzazioni senza corrispondenza in coreano, ad esempio come spam o newsletter. Gli ambienti svizzeri non sono quindi esclusi a priori. Microsoft sta ancora indagando sul problema; verifica e workaround sono descritti nella sezione successiva.

## Workaround per il deadlock di WordBreaker

Il workaround di KB5130098 aggiunge due file di regole dal pacchetto SQL Server 2025 Express RTM. Microsoft lo vincola a precise condizioni: non deve essere applicato solo a causa dei sintomi, ma esclusivamente sui server che soddisfano tutti i criteri di verifica e nei quali mancano **entrambi** i file. L’[Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) non verifica questi file; la versione v26.10.02 del 2 ottobre 2026 non contiene un controllo corrispondente (stato al 9 ottobre 2026). Il controllo va quindi effettuato manualmente.

### 1. Verificare il server

Lo script seguente non modifica nulla nel sistema. Legge il percorso di installazione dalla variabile d’ambiente `ExchangeInstallPath` e restituisce i valori indicati da KB5130098 come prerequisiti.

```powershell
$native  = Join-Path $env:ExchangeInstallPath 'Bin\Search\Ceres\Native'
$exsetup = Get-Item (Join-Path $env:ExchangeInstallPath 'Bin\ExSetup.exe')
$dll     = Get-Item (Join-Path $native 'korwbrkr.dll')
[pscustomobject]@{
    ExSetup         = $exsetup.VersionInfo.FileVersion
    KorwbrkrVersion = $dll.VersionInfo.FileVersion
    KorwbrkrBytes   = $dll.Length
    KorwbrkrSha256  = (Get-FileHash -LiteralPath $dll.FullName -Algorithm SHA256).Hash
    KoTokenRule     = Test-Path (Join-Path $native 'ko.token.rule.bin')
    KoComplexRule   = Test-Path (Join-Path $native 'ko.complex.rule.bin')
}
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Riga | Effetto |
|---|---|
| `Join-Path $env:ExchangeInstallPath 'Bin\Search\Ceres\Native'` | Costruisce il percorso della directory in cui devono trovarsi i file di regole. |
| `Get-Item ...\ExSetup.exe` | Legge il file la cui versione indica la build Exchange installata. |
| `VersionInfo.FileVersion` | Versione del file dalle risorse dell’EXE o della DLL. |
| `Get-FileHash -LiteralPath ... -Algorithm SHA256` | Calcola l’hash SHA256 di `korwbrkr.dll` installato. |
| `Test-Path ...` | Restituisce `True` se il rispettivo file di regole è già presente. |

</details>

Il workaround è consentito solo se tutti i valori corrispondono:

| Valore | Previsto |
| --- | --- |
| `ExSetup` | 15.02.2562.049 o 15.02.2562.053 |
| `KorwbrkrVersion` | 16.0.5194.1000 |
| `KorwbrkrBytes` | 326544 |
| `KorwbrkrSha256` | 1C6BD8E144BA677EBCC83323AE59DB3881918170F9B3A5189B44611558B92C61 |
| `KoTokenRule`, `KoComplexRule` | entrambi `False` |

Se la build, la versione o l’hash della DLL differiscono, Microsoft rimanda al supporto. Se uno dei due file di regole è già presente, interrompere e non sovrascrivere nulla.

### 2. Procurarsi i file di regole

I file provengono da `SQLEXPR_x64_ENU.exe`, il pacchetto inglese x64 di SQL Server 2025 Express RTM (versione 17.0.1000.7, 748772024 byte). Microsoft consiglia di eseguire questo passaggio su una workstation di gestione, non sul server Exchange. SQL Server non viene installato: il pacchetto viene solo estratto e l’MSI Full-Text viene estratto solo amministrativamente. Verificare prima la firma e l’hash del download:

```powershell
Get-AuthenticodeSignature .\SQLEXPR_x64_ENU.exe |
    Select-Object Status, SignerCertificate
Get-FileHash .\SQLEXPR_x64_ENU.exe -Algorithm SHA256
```

Sono previsti lo stato `Valid` con un certificato Microsoft e il valore SHA256 `74AA90C11202A5524E769B9BC22531BAEF22D91E9B2D2E8C3CB99E89A65C5297`. Quindi decomprimere ed estrarre:

```powershell
$work = 'C:\Temp\KoreanRules'
$p = Start-Process .\SQLEXPR_x64_ENU.exe -ArgumentList '/q', "/x:$work\Media" -Wait -PassThru
$p.ExitCode
$msi = "$work\Media\x64\Setup\SQL_FULLTEXT.MSI"
$msiArgs = '/a', $msi, "TARGETDIR=$work\Files", '/qn', '/L*V', "$work\extract.log"
$p = Start-Process msiexec.exe -ArgumentList $msiArgs -Wait -PassThru
$p.ExitCode
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `/q` | Decomprime senza finestre di dialogo. |
| `/x:<Pfad>` | Modalità di estrazione del pacchetto SQL: decomprime i supporti di installazione nella cartella senza avviare SQL Setup. |
| `-Wait -PassThru` | Attende il termine del processo e restituisce l’oggetto processo, affinché sia leggibile il codice di uscita. |
| `msiexec /a <MSI>` | Installazione amministrativa: deposita il contenuto dell’MSI come file, senza installare né registrare nulla. |
| `TARGETDIR=<Pfad>` | Cartella di destinazione per i file estratti. |
| `/qn` | Senza interfaccia utente. |
| `/L*V <Datei>` | Registro dettagliato che indica la causa in caso di errore. |

</details>

Entrambi i codici di uscita devono essere `0`. Se l’estrazione fallisce, salvare `extract.log` e interrompere. Verificare poi dimensione e hash dei due file:

```powershell
Get-ChildItem "$work\Files" -Recurse -Include 'ko.token.rule.bin', 'ko.complex.rule.bin' |
    Select-Object FullName, Length,
        @{ n = 'SHA256'; e = { (Get-FileHash -LiteralPath $_.FullName -Algorithm SHA256).Hash } }
```

| File | Byte | SHA256 |
| --- | --- | --- |
| `ko.token.rule.bin` | 56132 | 8F2BD853593913EB8F73DCD4FCAC4216F216A0FF76A4569DF071BE3C36773010 |
| `ko.complex.rule.bin` | 717792 | 0390D1E9A76EF33283025CF8F164430E311584B9535949C4EA1A74B6BB107B87 |

I file si trovano nell’albero estratto sotto `...\MSSQL\Binn\ftcomponents\wordbreakers`. Trasferire solo questi due file in una cartella di staging sul server Exchange, non i supporti SQL e in particolare non `korwbrkr.dll` dal pacchetto SQL.

### 3. Copiare i file

```powershell
$native  = Join-Path $env:ExchangeInstallPath 'Bin\Search\Ceres\Native'
$staging = 'C:\Temp\KoStaging'
foreach ($f in 'ko.token.rule.bin', 'ko.complex.rule.bin') {
    Copy-Item -LiteralPath (Join-Path $staging $f) -Destination $native
    Get-FileHash -LiteralPath (Join-Path $native $f) -Algorithm SHA256
}
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `Copy-Item -LiteralPath <Quelle>` | Copia il file dalla cartella di staging; il percorso viene valutato senza caratteri jolly. |
| `-Destination $native` | Directory di destinazione accanto a `korwbrkr.dll` installato. |
| `Get-FileHash ... -Algorithm SHA256` | Verifica di nuovo l’hash nel percorso di destinazione rispetto alla tabella sopra. |

</details>

Utilizzare `Copy-Item`, non `Move-Item`. Un file copiato riceve le autorizzazioni ereditate della directory di destinazione, mentre un file spostato conserva le autorizzazioni della cartella di origine. I file necessitano dei normali diritti di lettura di `Native`. Non sostituire altri file in questa directory.

### 4. Riavviare Search Host Controller

```powershell
Restart-Service -Name HostControllerService
```

Il riavvio interrompe temporaneamente la ricerca e l’elaborazione dei contenuti sul server; Microsoft prevede a tal fine una finestra di manutenzione. Non è necessario riavviare il server stesso. Non terminare forzatamente processi Exchange: se il servizio non si avvia correttamente, secondo Microsoft il passo successivo è contattare il supporto.

### 5. Verifica

Host Controller avvia autonomamente il processo `NodeRunner.exe` di ContentEngine; non avviarlo manualmente. Questa query mostra se è in esecuzione:

```powershell
Get-CimInstance Win32_Process -Filter "Name = 'NodeRunner.exe'" |
    Where-Object CommandLine -like '*ContentEngineNode1*' |
    Select-Object ProcessId, CreationDate
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `Get-CimInstance Win32_Process -Filter "Name = 'NodeRunner.exe'"` | Legge tutti i processi NodeRunner con la relativa riga di comando. |
| `Where-Object CommandLine -like '*ContentEngineNode1*'` | Filtra il processo di ContentEngine. |
| `Select-Object ProcessId, CreationDate` | Mostra ID processo e ora di avvio; l’ora di avvio deve essere successiva al riavvio del servizio. |

</details>

Se non compare alcun processo o se termina ripetutamente, non intervenire su altri server. Altrimenti, verificare in Outlook sul Web con una cassetta postale la cui copia attiva del database si trova su questo server se le nuove e-mail vengono recapitate e trovate, incluse le e-mail con testo coreano. Controllare inoltre il servizio originariamente interessato dal problema (recapito, connessione Outlook). Monitorare separatamente l’arretrato nell’indice di ricerca: il fatto che le nuove e-mail vengano trovate non prova che gli elementi meno recenti siano già stati elaborati. Intervenire sugli altri server singolarmente e solo dopo una verifica riuscita del precedente.

## Installazione e attività successive

Microsoft raccomanda la procedura nota: inventariare con l’[Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), determinare il percorso con l’[Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) se la versione è obsoleta, installare l’SU, riavviare il server e verificare che tutti i servizi Exchange siano in esecuzione. L’Health Checker mostra inoltre se l’SU è installato correttamente. Il Security Update Guide indica un riavvio obbligatorio per gli aggiornamenti.

Dopo l’installazione vi sono tre attività da eseguire:

1. Rimuovere il SettingOverride del wrapper `DisableBlockSharedAndUserMailboxHeaders`, se impostato (vedere sopra).

2. In Exchange SE, controllare con lo script di verifica della [sezione WordBreaker](#workaround-für-den-wordbreaker-deadlock) se mancano i due file di regole e verificare se si presenta il problema di disponibilità. Applicare i workaround solo se sono soddisfatte le rispettive condizioni.

3. Per i calendari pubblicati con abbonati esterni, configurare la regola di riscrittura URL di KB5126672, se non è già stato fatto dall’SU di agosto.

Da luglio resta inoltre da controllare se la mitigazione CVE-2026-42897 (M2.1.0) è ancora attiva; come rimuoverla è indicato nell’[articolo sull’SU di luglio](/blog/exchange-security-updates-juli-2026).

## Procedura consigliata

Installare direttamente le build v2 del 2 ottobre su tutti i server Exchange e sulle macchine con Management Tools; i server con l’SU dell’8 settembre necessitano inoltre della v2 per CVE-2026-96940. La vulnerabilità di spoofing con CVSS 9.3 e l’acquisizione delle cassette postali tramite un account utente con pochi privilegi (CVE-2026-69380) sono ragioni sufficienti per non attendere il prossimo Patch Tuesday. Rimuovere poi l’override del wrapper, verificare i tre problemi noti ed eseguire l’Health Checker. L’Health Checker non segnala i file WordBreaker mancanti; a questo scopo è necessario lo script di verifica della sezione WordBreaker. Per Exchange 2016 e 2019 il programma ESU termina nell’ottobre 2026; la migrazione a Exchange SE non può più essere rinviata.

## Fonti

1.  [Released: September 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-september-2026-exchange-server-security-updates/4554411): Annuncio ufficiale del rilascio con versioni supportate, indicazione ESU, problemi noti, problemi risolti e procedura di installazione (consultato tramite il feed RSS dell’Exchange Team Blog).

2.  [Released: September 2026 V2 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-september-2026-v2-exchange-server-security-updates/ba-p/4561718): Annuncio della v2 del 2 ottobre 2026; l’unica differenza è CVE-2026-96940.

3.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM: September 8, 2026 (KB5121608) – Microsoft Support](https://support.microsoft.com/help/5121608): Elenco CVE, problemi risolti e i tre problemi noti per Exchange SE.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU15: September 8, 2026 (KB5121609) – Microsoft Support](https://support.microsoft.com/help/5121609): Articolo KB per Exchange 2019 CU15.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU14: September 8, 2026 (KB5121610) – Microsoft Support](https://support.microsoft.com/help/5121610): Articolo KB per Exchange 2019 CU14.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: September 8, 2026 (KB5121611) – Microsoft Support](https://support.microsoft.com/help/5121611): Articolo KB per Exchange 2016 CU23, senza CVE-2026-55007.

7.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Tipo, CVSS, gravità, valutazione dello sfruttamento e FAQ su tutte e nove le CVE di settembre e su CVE-2026-96940, incluse le build interessate per ciascuna CVE.

8.  [Description of version 2 of the security update for Microsoft Exchange Server Subscription Edition RTM October 2, 2026 (KB5129955) – Microsoft Support](https://support.microsoft.com/help/5129955): Articolo KB sulla v2 per Exchange SE; gli articoli v2 per 2019 e 2016 sono KB5129956 fino a KB5129958.

9.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Numeri di build degli SU di settembre e della v2 del 2 ottobre 2026.

10. [Wrapper messages appear in shared mailbox in hybrid environments after installing the June 2026 Security Update – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Correzione nell’SU di settembre e istruzioni per rimuovere il SettingOverride.

11. [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Causa e workaround di riscrittura URL.

12. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Sintomi e workaround SettingOverride per Exchange SE.

13. [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Build SE interessate, criteri di verifica, dimensioni e hash dei file previsti, reperimento da SQL Server 2025 Express, riavvio del servizio e controllo.

14. [Hybrid free/busy through Microsoft Graph incorrectly shifts busy times by requester timezone – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5125804): Errore di fuso orario in Exchange SE corretto con l’SU di settembre.

15. [Neue Sicherheitsupdates für Exchange Server (September 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-september-2026/): Analisi in lingua tedesca delle nove CVE con valori CVSS e build.

16. [Exchange Server: Sicherheitsupdates 8. September 2026 – Borns Tech and Windows World](https://borncity.com/blog/?p=329371): Riepilogo in lingua tedesca con riferimenti al successivo aggiornamento sostitutivo.

17. [microsoft/CSS-Exchange – GitHub](https://github.com/microsoft/CSS-Exchange): Codice sorgente dell’Exchange Health Checker; release v26.10.02 del 2 ottobre 2026 senza verifica dei file WordBreaker (stato al 9 ottobre 2026).
