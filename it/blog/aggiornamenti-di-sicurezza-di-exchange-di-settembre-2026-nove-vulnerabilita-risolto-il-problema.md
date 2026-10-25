---
title: "Aggiornamenti di sicurezza Exchange di settembre 2026: nove vulnerabilità, problema dei wrapper risolto, rilasciata la v2"
navTitle: "Exchange SU 09/2026"
description: "L'SU di settembre chiude nove vulnerabilità in Exchange SE e 2019 (otto in Exchange 2016), inclusa una falla di spoofing con CVSS 9.3, e risolve il problema dei wrapper negli ambienti ibridi. Il 2 ottobre è seguita una v2 con una CVE aggiuntiva; inoltre vi sono tre problemi noti con workaround e un SettingOverride che ora dovrebbe essere rimosso."
date: "2026-10-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "11 min. di lettura"
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
translationSourceHash: fb8f756123ccec8c1983887594f47b521485f9fcf463783b480bd5eaaac8a427
translationModel: gpt-5.6-terra
translatedAt: 2026-10-25T19:39:36.051Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/aggiornamenti-di-sicurezza-di-exchange-di-settembre-2026-nove-vulnerabilita-risolto-il-problema
---

# Aggiornamenti di sicurezza Exchange di settembre 2026: nove vulnerabilità, problema dei wrapper risolto, rilasciata la v2

Microsoft ha pubblicato l'8 settembre 2026 aggiornamenti di sicurezza (SU) per Exchange Server. Chiudono nove vulnerabilità in Exchange SE e Exchange 2019 e otto in Exchange 2016. Nessuna era nota pubblicamente in anticipo, nessuna risulta sfruttata attivamente secondo il Security Update Guide e Microsoft le classifica tutte come *Important* con «Exploitation Less Likely». Il valore CVSS più alto, pari a 9.3, supera tuttavia nettamente quello del mese precedente. Questo mese è particolare per tre motivi: l'SU risolve il problema, aperto da giugno, dei *messaggi wrapper* nelle cassette postali condivise; introduce tre problemi noti nuovi o persistenti; e il 2 ottobre Microsoft ha rilasciato una **versione 2** che risolve un'ulteriore vulnerabilità.

## Per quali versioni di Exchange è disponibile l'aggiornamento

Gli SU dell'8 settembre 2026 sono disponibili per le seguenti versioni:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121608, build 15.2.2562.49; disponibile pubblicamente.
- **Exchange Server 2019 CU15**: KB5121609, build 15.2.1748.51; solo tramite il **programma ESU Period 2**.
- **Exchange Server 2019 CU14**: KB5121610, build 15.2.1544.46; solo tramite ESU Period 2.
- **Exchange Server 2016 CU23**: KB5121611, build 15.1.2507.73; solo tramite ESU Period 2.

Exchange 2016 e 2019 sono fuori supporto. Secondo Microsoft, gli SU da maggio a ottobre 2026 sono disponibili solo per le organizzazioni iscritte al programma ESU Period 2. Secondo gli articoli KB, questa idoneità dura fino a ottobre 2026. Si aggiunge la pressione di Exchange Online: dalla seconda settimana di settembre, Exchange Online limita e blocca il flusso di posta ibrido proveniente da server con un livello inferiore a ottobre 2025; dettagli nell'[articolo sul Transport Enforcement](/blog/exchange-online-transport-enforcement-hybrid-server). Exchange Online stesso è già protetto secondo l'annuncio; negli ambienti ibridi, tuttavia, ogni server Exchange necessita dell'SU, così come le macchine con gli Exchange Management Tools.

Potete confrontare il vostro livello con la panoramica dei [numeri di build di Exchange](/tools/exchange-builds).

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

CVE-2026-55007 non riguarda Exchange 2016; il Security Update Guide indica per essa solo Exchange SE e 2019 CU14/CU15. Un dettaglio relativo alla documentazione: negli articoli KB dei quattro SU di settembre manca CVE-2026-69380 nell'elenco delle CVE. Il Security Update Guide indica tuttavia esattamente le quattro build di settembre come correzione per questa CVE (al 7 ottobre 2026).

**[CVE-2026-69356](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69356)** ha il valore più alto, CVSS 9.3. Secondo Microsoft, un aggressore non autenticato può inviare un invito di calendario predisposto con un collegamento dannoso a una riunione; se la destinataria o il destinatario apre la riunione e seleziona il link per partecipare, viene attivato il Cross-Site Scripting. È quindi necessaria un'interazione dell'utente, ma non un account nell'organizzazione.

**[CVE-2026-69380](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69380)** (Elevation of Privilege, CVSS 8.1) richiede soltanto un account con pochi privilegi e una cassetta postale assegnata. Secondo le FAQ nel Security Update Guide, un aggressore può, sfruttando debolezze nella verifica delle richieste e dei token di identità, impersonare un altro utente e compromettere le cassette postali di tutti gli utenti Exchange: leggere e inviare email e scaricare allegati. Come punto di partenza è sufficiente un singolo account utente compromesso.

**[CVE-2026-69641](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69641)** (Elevation of Privilege, CVSS 9.1) porta allo stesso risultato, la compromissione di tutte le cassette postali, ma richiede l'appartenenza a un gruppo di ruoli con privilegi elevati.

Per le due vulnerabilità di Remote Code Execution, il punto di partenza è diverso: [CVE-2026-69355](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69355) (CVSS 8.8) richiede un account autenticato con pochi privilegi, mentre [CVE-2026-55007](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-55007) (CVSS 8.1) può essere attivata senza autenticazione tramite un allegato Visio predisposto, ma secondo Microsoft richiede una persistente carenza di memoria sul sistema di destinazione. Le altre quattro falle: CVE-2026-69378 (DoS dovuto a ricorsione non controllata, senza autenticazione), CVE-2026-69361 (SSRF, il server invia richieste HTTP a sistemi interni o di loopback), CVE-2026-69375 (un aggressore autenticato può sostituire contenuti di file) e CVE-2026-69382 (divulgazione di credenziali tramite un algoritmo crittografico debole, richiede un cookie di autenticazione già sottratto).

## Versione 2 del 2 ottobre: aggiunta CVE-2026-96940

Il 2 ottobre 2026 Microsoft ha pubblicato la «versione 2» degli SU di settembre. Secondo l'annuncio, l'unica differenza rispetto alla prima versione è la correzione aggiuntiva per **[CVE-2026-96940](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940). Questa vulnerabilità di Elevation of Privilege ha CVSS 8.8, non è né nota pubblicamente né sfruttata, ma Microsoft la valuta come l'unica delle dieci CVE con **«Exploitation More Likely»**. Un aggressore autenticato può così accedere alle cassette postali di altri utenti della stessa organizzazione e leggere email con relativi allegati. Exchange Online è già stato corretto lato server.

| Versione | KB | Build v2 |
| --- | --- | --- |
| Exchange SE RTM | KB5129955 | 15.2.2562.53 |
| Exchange 2019 CU15 | KB5129956 | 15.2.1748.53 |
| Exchange 2019 CU14 | KB5129957 | 15.2.1544.48 |
| Exchange 2016 CU23 | KB5129958 | 15.1.2507.75 |

In pratica, la correzione per CVE-2026-96940 è presente solo nelle build v2. I server sui quali è già in esecuzione l'SU dell'8 settembre necessitano inoltre della v2. Chi applica ora le patch può installare direttamente la v2, poiché gli SU sono cumulativi. Gli articoli KB non dicono espressamente se i server con il primo SU di settembre debbano obbligatoriamente installare la v2; poiché la nuova CVE viene chiusa solo con la v2, questa è l'interpretazione più ovvia.

## Problema dei wrapper risolto: rimuovere ora il SettingOverride

Il problema noto dall'SU di giugno, per cui negli ambienti ibridi compaiono *messaggi wrapper* nella posta in arrivo delle cassette postali condivise, è risolto con l'SU di settembre in tutte e quattro le versioni. Nell'[articolo di agosto](/blog/exchange-security-updates-august-2026) si indicava ancora che il SettingOverride documentato come workaround poteva rimanere in vigore. Dopo l'installazione dell'SU di settembre vale il contrario: Microsoft raccomanda nell'articolo di supporto associato di controllare e rimuovere l'override.

```powershell
Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Comando | Effetto |
|---|---|
| `Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Verifica se l'override del workaround è impostato nell'organizzazione. |
| `Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Rimuove l'override non appena è installato l'SU di settembre. |

</details>

Se il primo comando segnala che l'oggetto `DisableBlockSharedAndUserMailboxHeaders` non è stato trovato, secondo Microsoft non occorre fare altro.

Per Exchange SE, l'SU risolve inoltre un errore nelle richieste di disponibilità ibrida tramite Microsoft Graph: gli utenti on-premises vedevano gli orari occupati delle cassette postali di Exchange Online spostati del proprio scostamento UTC, senza messaggi di errore.

## Problemi noti

**I calendari pubblicati (.ics) restituiscono HTTP 500 alle app di calendario.** Il problema esiste dall'SU di agosto (Exchange SE dalla build 15.2.2562.46, nonché Exchange 2019 e 2016) e non è risolto né nell'SU di settembre né nella v2. Gli abbonamenti a calendari pubblicati anonimamente non si aggiornano più; la stessa URL funziona nel browser. Secondo Microsoft, la causa è che Exchange riconosce i client dal relativo User-Agent: le app calendario prive dell'identificativo del browser finiscono in un percorso di codice che l'SU di agosto ha disabilitato. Come workaround, Microsoft descrive una regola di riscrittura URL nel sito «Exchange Back End» in IIS, che per richieste .ics sotto `/owa/calendar/` aggiunge il parametro `layout=premium`; a questo scopo deve essere installato il modulo IIS URL Rewrite. I passaggi esatti, tramite IIS Manager o direttamente nel file `applicationHost.config`, sono descritti nell'[articolo di supporto KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672). Dal 13 ottobre 2026, secondo KB5126672, il problema è risolto con gli SU di ottobre (KB5128895 per Exchange SE, KB5128896 fino a KB5128898 per 2019 e 2016); dopo la loro installazione la regola di riscrittura va rimossa.

**Disponibilità per cassette postali delegate in ambienti ibridi (solo Exchange SE).** Se la query di disponibilità è configurata esclusivamente tramite l'API Graph, le query per cassette postali di Exchange Online tramite accesso on-premises delegato falliscono. Outlook segnala «Your server location could not be determined», OWA mostra «No information» e nei log EWS appare `(403) Forbidden`. Il workaround documentato instrada nuovamente le query tramite EWS anziché Graph:

```powershell
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=False"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Identity EnableRouteThroughMSGraphFeature` | L'override che controlla l'inoltro delle query di disponibilità tramite Microsoft Graph. |
| `-Parameters "Enabled=False"` | Disattiva il percorso Graph; le query passano nuovamente tramite EWS. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Indirizza la chiamata diagnostica al servizio di topologia. |
| `-Component VariantConfiguration -Argument Refresh` | Ricarica la Variant Configuration affinché l'override abbia effetto senza attesa. |

</details>

Nell'annuncio della v2, Microsoft elenca questo problema tra quelli risolti. Nel frattempo, l'[articolo di supporto KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) segnala che il problema è stato risolto nel servizio. Se avete impostato il workaround, KB5127092 indica di annullarlo:

```powershell
Get-SettingOverride -Identity EnableRouteThroughMSGraphFeature
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=True"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Get-SettingOverride -Identity EnableRouteThroughMSGraphFeature` | Mostra se l'override è impostato e quale valore ha. |
| `-Parameters "Enabled=True"` | Riattiva il percorso Graph per le query di disponibilità. |
| `-Component VariantConfiguration -Argument Refresh` | Ricarica la Variant Configuration affinché la modifica abbia effetto senza attesa. |

</details>

**Deadlock di ContentEngine dovuto alla mancanza di file regola Korean WordBreaker (solo Exchange SE).** Con l'SU di settembre (build 15.2.2562.49 e build v2 15.2.2562.53), i file regola per il Korean WordBreaker aggiornato non vengono installati. Le conseguenze sono risultati di ricerca mancanti, recapito di posta ritardato, client Outlook o MAPI che si bloccano o perdono la connessione. [KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098) indica come interessati gli ambienti che elaborano email con contenuti in lingua coreana. Ciò non dipende dalla lingua dei propri utenti: messaggi con testo coreano raggiungono anche organizzazioni senza corrispondenza coreana, ad esempio come spam o newsletter. Gli ambienti svizzeri non sono quindi esclusi a priori. Dal 13 ottobre 2026, secondo KB5130098, il problema è risolto con KB5128895 (build 15.2.2562.56) o un aggiornamento più recente; in seguito, secondo Microsoft, non sono necessari ulteriori passaggi, neppure sui server sui quali è stato applicato il workaround manuale. La verifica e il workaround nella sezione successiva riguardano solo i server con l'SU di settembre o la v2.

## Workaround per il deadlock di WordBreaker

Il workaround di KB5130098 aggiunge due file regola dal pacchetto SQL Server 2025 Express RTM. Microsoft lo vincola a condizioni precise: non deve essere applicato soltanto a causa dei sintomi, ma solo sui server che soddisfano tutti i criteri di verifica e ai quali mancano **entrambi** i file. L'[Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) non verifica questi file; la versione v26.10.02 del 2 ottobre 2026 non contiene una verifica corrispondente (al 9 ottobre 2026). Il controllo va quindi effettuato manualmente.

### 1. Verificare il server

Lo script seguente non modifica nulla nel sistema. Legge il percorso di installazione dalla variabile d'ambiente `ExchangeInstallPath` e restituisce i valori indicati da KB5130098 come prerequisiti.

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
<summary>Opzioni spiegate</summary>

| Riga | Effetto |
|---|---|
| `Join-Path $env:ExchangeInstallPath 'Bin\Search\Ceres\Native'` | Crea il percorso della directory in cui devono trovarsi i file regola. |
| `Get-Item ...\ExSetup.exe` | Legge il file la cui versione indica la build Exchange installata. |
| `VersionInfo.FileVersion` | Versione del file dalle risorse dell'EXE o DLL. |
| `Get-FileHash -LiteralPath ... -Algorithm SHA256` | Calcola l'hash SHA256 del file `korwbrkr.dll` installato. |
| `Test-Path ...` | Restituisce `True` se il rispettivo file regola è già presente. |

</details>

Il workaround è consentito solo se tutti i valori corrispondono:

| Valore | Atteso |
| --- | --- |
| `ExSetup` | 15.02.2562.049 o 15.02.2562.053 |
| `KorwbrkrVersion` | 16.0.5194.1000 |
| `KorwbrkrBytes` | 326544 |
| `KorwbrkrSha256` | 1C6BD8E144BA677EBCC83323AE59DB3881918170F9B3A5189B44611558B92C61 |
| `KoTokenRule`, `KoComplexRule` | entrambi `False` |

Se build, versione o hash della DLL differiscono, Microsoft rimanda al supporto. Se uno dei due file regola è già presente, interrompete l'operazione e non sovrascrivete nulla.

### 2. Procurarsi i file regola

I file provengono da `SQLEXPR_x64_ENU.exe`, il pacchetto x64 inglese di SQL Server 2025 Express RTM (versione 17.0.1000.7, 748772024 byte). Microsoft raccomanda di eseguire questo passaggio su una workstation di gestione, non sul server Exchange. SQL Server non viene installato: il pacchetto viene solo estratto e l'MSI Full-Text viene estratto solo a livello amministrativo. Verificate prima firma e hash del download:

```powershell
Get-AuthenticodeSignature .\SQLEXPR_x64_ENU.exe |
    Select-Object Status, SignerCertificate
Get-FileHash .\SQLEXPR_x64_ENU.exe -Algorithm SHA256
```

Sono previsti lo stato `Valid` con un certificato Microsoft e il valore SHA256 `74AA90C11202A5524E769B9BC22531BAEF22D91E9B2D2E8C3CB99E89A65C5297`. Quindi estraete e decomprimete:

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
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `/q` | Estrae senza finestre di dialogo. |
| `/x:<Pfad>` | Modalità Extract del pacchetto SQL: estrae i supporti di installazione nella cartella senza avviare SQL Setup. |
| `-Wait -PassThru` | Attende il termine del processo e restituisce l'oggetto processo, affinché sia leggibile il codice di uscita. |
| `msiexec /a <MSI>` | Installazione amministrativa: deposita il contenuto dell'MSI come file senza installare né registrare alcunché. |
| `TARGETDIR=<Pfad>` | Cartella di destinazione per i file estratti. |
| `/qn` | Senza interfaccia utente. |
| `/L*V <Datei>` | Registro dettagliato che mostra la causa in caso di errore. |

</details>

Entrambi i codici di uscita devono essere `0`. Se l'estrazione fallisce, conservate `extract.log` e interrompete l'operazione. Quindi verificate dimensione e hash dei due file:

```powershell
Get-ChildItem "$work\Files" -Recurse -Include 'ko.token.rule.bin', 'ko.complex.rule.bin' |
    Select-Object FullName, Length,
        @{ n = 'SHA256'; e = { (Get-FileHash -LiteralPath $_.FullName -Algorithm SHA256).Hash } }
```

| File | Byte | SHA256 |
| --- | --- | --- |
| `ko.token.rule.bin` | 56132 | 8F2BD853593913EB8F73DCD4FCAC4216F216A0FF76A4569DF071BE3C36773010 |
| `ko.complex.rule.bin` | 717792 | 0390D1E9A76EF33283025CF8F164430E311584B9535949C4EA1A74B6BB107B87 |

I file si trovano nell'albero estratto sotto `...\MSSQL\Binn\ftcomponents\wordbreakers`. Trasferite solo questi due file in una cartella di staging sul server Exchange, nessun supporto SQL e in particolare non il file `korwbrkr.dll` dal pacchetto SQL.

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
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Copy-Item -LiteralPath <Quelle>` | Copia il file dalla cartella di staging; il percorso viene valutato senza caratteri jolly. |
| `-Destination $native` | Directory di destinazione accanto al file `korwbrkr.dll` installato. |
| `Get-FileHash ... -Algorithm SHA256` | Verifica nuovamente l'hash nella destinazione rispetto alla tabella sopra. |

</details>

Utilizzate `Copy-Item`, non `Move-Item`. Un file copiato riceve le autorizzazioni ereditate dalla directory di destinazione, mentre un file spostato conserva le autorizzazioni della cartella di origine. I file necessitano delle normali autorizzazioni di lettura di `Native`. Non sostituite altri file in questa directory.

### 4. Riavviare Search Host Controller

```powershell
Restart-Service -Name HostControllerService
```

Il riavvio interrompe temporaneamente la ricerca e l'elaborazione dei contenuti sul server; Microsoft prevede a questo scopo una finestra di manutenzione. Non è necessario riavviare il server stesso. Non terminate forzatamente i processi Exchange: se il servizio non si avvia correttamente, secondo Microsoft il passo successivo è il supporto.

### 5. Verifica

Host Controller avvia autonomamente il processo `NodeRunner.exe` di ContentEngine; non avviatelo manualmente. Questa query mostra se è in esecuzione:

```powershell
Get-CimInstance Win32_Process -Filter "Name = 'NodeRunner.exe'" |
    Where-Object CommandLine -like '*ContentEngineNode1*' |
    Select-Object ProcessId, CreationDate
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Get-CimInstance Win32_Process -Filter "Name = 'NodeRunner.exe'"` | Legge tutti i processi NodeRunner insieme alla riga di comando. |
| `Where-Object CommandLine -like '*ContentEngineNode1*'` | Filtra il processo di ContentEngine. |
| `Select-Object ProcessId, CreationDate` | Mostra ID processo e orario di avvio; l'orario di avvio deve essere successivo al riavvio del servizio. |

</details>

Se non compare alcun processo o se termina ripetutamente, non intervenite su altri server. Altrimenti, verificate in Outlook sul Web con una cassetta postale la cui copia attiva del database si trova su questo server se le nuove email vengono recapitate e trovate, incluse quelle con testo coreano. Controllate inoltre il servizio originariamente interessato dal problema (recapito, connessione Outlook). Monitorate separatamente l'arretrato nell'indice di ricerca: il fatto che le nuove email vengano trovate non prova che gli elementi più vecchi siano già stati elaborati. Intervenite sugli altri server singolarmente e solo dopo aver verificato con successo quello precedente.

## Installazione e attività successive

Microsoft raccomanda la procedura nota: inventariare con l'[Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), determinare il percorso con l'[Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) se il livello è obsoleto, installare l'SU, riavviare il server e verificare che tutti i servizi Exchange siano in esecuzione. Successivamente, Health Checker mostra anche se l'SU è stato installato correttamente. Il Security Update Guide indica un riavvio obbligatorio per gli aggiornamenti.

Dopo l'installazione sono necessarie tre attività:

1. Rimuovere il Wrapper-SettingOverride `DisableBlockSharedAndUserMailboxHeaders` se impostato (vedere sopra).

2. In Exchange SE, controllare con lo script di verifica della [sezione WordBreaker](#workaround-für-den-wordbreaker-deadlock) se mancano i due file regola e verificare se si presenta il problema di disponibilità. Applicate i workaround solo se sono soddisfatte le rispettive condizioni.

3. Per calendari pubblicati con abbonati esterni, configurare la regola di riscrittura URL di KB5126672, se ciò non è ancora stato fatto dall'SU di agosto e gli SU di ottobre non sono ancora installati.

Da luglio rimane inoltre il controllo che la mitigazione CVE-2026-42897 (M2.1.0) sia ancora attiva; come rimuoverla è descritto nell'[articolo sull'SU di luglio](/blog/exchange-security-updates-juli-2026).

## Procedura consigliata

Dal 13 ottobre 2026, secondo gli articoli KB, gli SU di ottobre sostituiscono quelli di settembre (Exchange SE: KB5128895, build 15.2.2562.56). Poiché gli SU sono cumulativi, installate direttamente l'SU di ottobre su tutti i server Exchange e sulle macchine con Management Tools; dettagli nell'[articolo sull'SU di ottobre](/blog/exchange-security-updates-oktober-2026). Dove ciò non è ancora possibile, un server con l'SU dell'8 settembre necessita almeno della v2 per CVE-2026-96940. La vulnerabilità di spoofing con CVSS 9.3 e la compromissione della cassetta postale tramite un account utente con pochi privilegi (CVE-2026-69380) sono motivi sufficienti per non attendere il prossimo Patch Tuesday. Rimuovete quindi l'override dei wrapper, verificate i tre problemi noti ed eseguite Health Checker. Health Checker non rileva i file WordBreaker mancanti; a tale scopo è necessario lo script di verifica nella sezione WordBreaker. Per Exchange 2016 e 2019 il programma ESU termina nell'ottobre 2026; la migrazione a Exchange SE non può più essere rimandata.

## Fonti

1.  [Released: September 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-september-2026-exchange-server-security-updates/4554411): annuncio ufficiale del rilascio con versioni supportate, indicazione ESU, problemi noti, problemi risolti e procedura di installazione (consultato tramite il feed RSS dell'Exchange Team Blog).

2.  [Released: September 2026 V2 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-september-2026-v2-exchange-server-security-updates/ba-p/4561718): annuncio della v2 del 2 ottobre 2026; l'unica differenza è CVE-2026-96940.

3.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM: September 8, 2026 (KB5121608) – Microsoft Support](https://support.microsoft.com/help/5121608): elenco CVE, problemi risolti e i tre problemi noti per Exchange SE.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU15: September 8, 2026 (KB5121609) – Microsoft Support](https://support.microsoft.com/help/5121609): articolo KB per Exchange 2019 CU15.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU14: September 8, 2026 (KB5121610) – Microsoft Support](https://support.microsoft.com/help/5121610): articolo KB per Exchange 2019 CU14.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: September 8, 2026 (KB5121611) – Microsoft Support](https://support.microsoft.com/help/5121611): articolo KB per Exchange 2016 CU23, senza CVE-2026-55007.

7.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): tipo, CVSS, gravità, valutazione dello sfruttamento e FAQ su tutte le nove CVE di settembre e su CVE-2026-96940, incluse le build interessate per ciascuna CVE.

8.  [Description of version 2 of the security update for Microsoft Exchange Server Subscription Edition RTM October 2, 2026 (KB5129955) – Microsoft Support](https://support.microsoft.com/help/5129955): articolo KB sulla v2 per Exchange SE; gli articoli v2 per 2019 e 2016 sono KB5129956 fino a KB5129958.

9.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): numeri di build degli SU di settembre e della v2 del 2 ottobre 2026.

10. [Wrapper messages appear in shared mailbox in hybrid environments after installing the June 2026 Security Update – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): correzione nell'SU di settembre e istruzioni per rimuovere il SettingOverride.

11. [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): causa, workaround di riscrittura URL e correzione con gli SU di ottobre del 13 ottobre 2026.

12. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): sintomi, workaround SettingOverride per Exchange SE, correzione nel servizio e comandi per il ripristino.

13. [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): build SE interessate, criteri di verifica, dimensioni e hash previsti dei file, ottenimento da SQL Server 2025 Express, riavvio del servizio, verifica e correzione con KB5128895.

14. [Hybrid free/busy through Microsoft Graph incorrectly shifts busy times by requester timezone – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5125804): errore di fuso orario in Exchange SE risolto con l'SU di settembre.

15. [Neue Sicherheitsupdates für Exchange Server (September 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-september-2026/): analisi in lingua tedesca delle nove CVE con valori CVSS e build.

16. [Exchange Server: Sicherheitsupdates 8. September 2026 – Borns Tech and Windows World](https://borncity.com/blog/?p=329371): riepilogo in lingua tedesca con indicazioni sul successivo aggiornamento sostitutivo.

17. [microsoft/CSS-Exchange – GitHub](https://github.com/microsoft/CSS-Exchange): codice sorgente di Exchange Health Checker; release v26.10.02 del 2 ottobre 2026 senza verifica dei file WordBreaker (al 9 ottobre 2026).

18. [Description of the security update for Microsoft Exchange Server Subscription Edition RTM October 13, 2026 (KB5128895) – Microsoft Support](https://support.microsoft.com/help/5128895): SU di ottobre che sostituisce l'SU di settembre e risolve l'errore .ics nonché il deadlock WordBreaker.
