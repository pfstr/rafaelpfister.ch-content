---
title: "Claude Desktop si arresta continuamente: \"GPU process gone\" con codice di uscita 101457950, causa e soluzione"
navTitle: "Arresto anomalo di Claude Desktop"
description: "L'app Claude Desktop su Windows si chiude completamente con \"GPU process gone: exitCode 101457950\" (0x060C201E), spesso seguita dalla finestra di riparazione dell'app Store. La catena completa delle cause: Code Integrity blocca vk_swiftshader.dll, la catena di fallback di Chromium si esaurisce, l'interruzione automatica integrata chiude l'app. Con una soluzione permanente (passaggio all'installazione classica senza MSIX), autodiagnosi tramite registro eventi e analisi fino al minidump."
date: "2026-08-25"
kategorie: "Claude"
timeToRead: "10 min di lettura"
themen:
  - claude
slug: "il-crash-gpu-0x060c201e-nell-app-desktop-claude-un-indagine-fino-al-minidump"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
translationSourceHash: 9bd177698f756b588c97f11ea4c73ea48efcfb96c8dbe1435f5ff9fc11dffec0
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:29:58.042Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/il-crash-gpu-0x060c201e-nell-app-desktop-claude-un-indagine-fino-al-minidump
---

L'app Claude Desktop su Windows si chiude senza messaggi di errore, tutte le sessioni Claude Code in corso vanno perse e talvolta l'app riparte solo dopo una "Riparazione" dalle impostazioni di Windows. Nel log dell'app compare quindi questa riga:

```text
GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
```

101457950 in esadecimale è `0x060C201E`. Se trovate questa firma nel vostro log, siete nel posto giusto: questo articolo documenta l'intera catena causale di questo arresto anomalo, le misure immediate che rendono di nuovo stabile l'app e l'autodiagnosi con cui potete confermare il riscontro sul vostro sistema in due minuti. Sono interessate le installazioni MSIX (dal Microsoft Store o tramite setup MSIX) su tutti i produttori di GPU, dalle iGPU Intel a NVIDIA e AMD; l'hardware, per anticiparlo, non è la causa. L'installazione classica senza MSIX non è interessata, ed è proprio questa la soluzione.

## La soluzione in breve: passare all'installazione classica

L'errore effettivo risiede nel pacchetto di installazione MSIX e può essere corretto solo da Anthropic (l'issue [#81341](https://github.com/anthropics/claude-code/issues/81341) risulta ancora aperta al 25.10.2026; all'inizio di settembre 2026, le segnalazioni nell'issue hanno confermato l'errore anche nella versione 1.40609.1). Tuttavia, la stessa app è disponibile anche come installazione classica senza MSIX e non è soggetta alla verifica della firma AppX che termina il processo GPU. Il passaggio è quindi l'unica misura che elimina completamente l'arresto anomalo; è confermato sia nell'issue [#81341](https://github.com/anthropics/claude-code/issues/81341) sia sul sistema qui analizzato. Le funzionalità sono identiche e il feed di aggiornamento fornisce le stesse versioni per entrambe le varianti.

**Passo 1: scaricare ed eseguire l'installer classico.** Il download da [claude.com/download](https://claude.com/download) fornisce un installer Squirrel che installa l'app in `%LOCALAPPDATA%\AnthropicClaude` (non sono richiesti diritti di amministratore). Da riga di comando:

```powershell
curl.exe -L -o "$env:USERPROFILE\Downloads\Claude-Setup-x64.exe" `
  "https://storage.googleapis.com/osprey-downloads-c02f6a0d-347c-492b-a752-3e0651722e97/nest-win-x64/Claude-Setup-x64.exe"
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-L` | segue i reindirizzamenti HTTP fino al file effettivo |
| `-o <pfad>` | file di destinazione; qui la cartella Download |
| `<url>` | fonte ufficiale dell'installer; identica alla destinazione del reindirizzamento del download da claude.ai |

</details>

Dopo il download, verificate la firma (`Get-AuthenticodeSignature`, atteso: `Valid`, emittente "Anthropic, PBC") e avviate il file. L'installer deposita inizialmente una versione base più vecchia; il meccanismo di aggiornamento la porta alla versione corrente, automaticamente al primo avvio oppure immediatamente con:

```powershell
& "$env:LOCALAPPDATA\AnthropicClaude\Update.exe" `
  --update https://downloads.claude.ai/releases/win32/x64
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `--update <url>` | scarica l'ultima versione dal feed di release indicato e la installa come nuova directory `app-<version>` |

</details>

**Passo 2: trasferire la configurazione.** La versione MSIX conserva login, configurazione dei server MCP e impostazioni nel proprio contenitore virtualizzato; l'app classica legge `%APPDATA%\Claude`. Copiate una sola volta (prima chiudete l'app MSIX; le due varianti non possono comunque essere eseguite contemporaneamente a causa di un blocco condiviso per singola istanza):

```powershell
robocopy "$env:LOCALAPPDATA\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude" `
  "$env:APPDATA\Claude" /E /XD Cache "Code Cache" GPUCache claude-code Crashpad logs sentry
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `<quelle>` | cartella di configurazione nell'AppData virtualizzato del pacchetto MSIX |
| `<ziel>` | cartella di configurazione dell'installazione classica |
| `/E` | copia tutte le sottodirectory, comprese quelle vuote |
| `/XD <namen>` | salta le directory indicate; qui cache e dati di runtime che la nuova app crea autonomamente |

</details>

Le cronologie delle chat non vengono perse: risiedono nell'account claude.ai oppure (per le sessioni Claude Code) in `%USERPROFILE%\.claude` e non dipendono dall'installazione dell'app.

**Passo 3: rimuovere il pacchetto MSIX.** Altrimenti, i vecchi collegamenti continueranno ad avviare la variante che si arresta:

```powershell
Get-AppxPackage Claude | Remove-AppxPackage
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Claude` | argomento posizionale Name di `Get-AppxPackage`: filtra i pacchetti AppX/MSIX installati in base al nome del pacchetto (sono ammessi caratteri jolly) |
| `Remove-AppxPackage` | rimuove il pacchetto passato tramite pipeline per l'account utente corrente |

</details>

La voce del menu Start "Anthropic → Claude" appartiene quindi all'installazione classica; un eventuale elemento fissato sulla barra delle applicazioni deve essere creato di nuovo.

## Se dovete rimanere con il pacchetto MSIX

Senza il passaggio, restano solo misure che riducono la frequenza degli arresti anomali senza eliminare la causa:

**Usare con parsimonia il browser incorporato.** Le pagine nell'area browser/anteprima dell'app innescano l'arresto. Chi chiude l'area dopo l'uso, invece di lasciare le schede aperte, riduce sensibilmente la frequenza degli arresti; questa correlazione è documentata più volte con dati numerici nel thread della community.

**Disattivare WebGPU.** Un avvio con `--disable-features=WebGPU` impedisce il trigger più frequente. Con un pacchetto MSIX, il percorso di installazione cambia a ogni aggiornamento; serve quindi un launcher che lo risolva di nuovo a ogni avvio:

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `for /f "delims="` | elabora l'output del comando riga per riga; `delims=` vuoto trasferisce l'intera riga (compresi gli spazi nel percorso) in `%%i` |
| `-NoProfile` | avvia PowerShell senza script di profilo, per un avvio rapido e riproducibile |
| `-Command` | esegue l'espressione indicata; `(Get-AppxPackage Claude).InstallLocation` restituisce il percorso di installazione corrente del pacchetto |
| `start ""` | avvia il programma separatamente dalla finestra batch; le virgolette vuote sono il titolo della finestra (qui vuoto) |
| `--disable-features=WebGPU` | opzione Chromium: disattiva la funzionalità indicata, qui l'API WebGPU |

</details>

Funziona solo se l'app viene avviata anche tramite questo launcher.

Nella prima versione di questo articolo, la raccomandazione principale era attivare l'accelerazione hardware tramite `isHardwareAccelerationDisabled: false` in `config.json`. Questa raccomandazione è superata: nelle versioni attuali (1.37937.x) il flag non esiste più, l'app si avvia per impostazione predefinita con l'accelerazione hardware attiva e si arresta comunque (dettagli nell'appendice sotto).

Una "Riparazione" o la reinstallazione del pacchetto MSIX non risolve il problema: cura soltanto il sintomo conseguente (maggiori dettagli sotto). Anche gli aggiornamenti dei driver grafici sono tempo perso.

## Autodiagnosi: confermare il riscontro sul proprio sistema

Sono sufficienti due controlli. Primo, la firma dell'arresto nel log dell'app:

```powershell
Select-String -Path "$env:LOCALAPPDATA\Claude\Logs\main.log" `
  -Pattern 'GPU process gone'
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Path` | file da cercare, qui il log principale dell'app |
| `-Pattern` | schema di ricerca (espressione regolare); restituisce tutte le righe con la firma dell'arresto |

</details>

Secondo, e questa è la prova effettiva, il log CodeIntegrity di Windows:

```powershell
Get-WinEvent -FilterHashtable @{
  LogName='Microsoft-Windows-CodeIntegrity/Operational'; Id=3033
} -MaxEvents 30 | Where-Object { $_.Message -match 'claude' } |
  Select-Object TimeCreated, Message
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-FilterHashtable` | filtra già al recupero: `LogName` indica il registro eventi, `Id` l'ID evento 3033 (blocco Code Integrity) |
| `-MaxEvents 30` | limita la query ai 30 risultati più recenti |
| `Where-Object { … -match 'claude' }` | mantiene solo gli eventi il cui testo del messaggio contiene il percorso dell'app |
| `Select-Object TimeCreated, Message` | riduce l'output a timestamp e messaggio per il confronto con gli orari degli arresti |

</details>

Sui sistemi interessati troverete voci Event 3033 con timestamp che coincidono al secondo con gli orari degli arresti, con questo messaggio:

```text
Code Integrity determined that a process
(...\WindowsApps\Claude_..._x64__pzs8sxrjxfjjc\app\claude.exe)
attempted to load ...\app\vk_swiftshader.dll that did not meet
the Microsoft signing level requirements.
```

Sul sistema qui analizzato, sette arresti su sette nell'arco di tre settimane coincidevano al secondo con un evento di questo tipo, incluso un arresto di controllo provocato intenzionalmente.

## La catena causale completa

L'arresto è l'ultimo anello di una catena di quattro elementi risultata da due analisi: la traccia Code Integrity dall'issue della community [#81698](https://github.com/anthropics/claude-code/issues/81698) e un'analisi autonoma del minidump ([#89250](https://github.com/anthropics/claude-code/issues/89250)).

**Elemento 1: una pagina nel browser incorporato richiede il rendering software.** Un trigger tipico è una chiamata WebGPU (`navigator.gpu.requestAdapter()`), riconoscibile nel log della finestra da questo avviso immediatamente prima dell'arresto:

```text
[warn] The powerPreference option is currently ignored
       when calling requestAdapter() on Windows.
```

Se l'app viene eseguita senza accelerazione hardware, il percorso passa obbligatoriamente dall'implementazione software Vulkan SwiftShader: il processo GPU tenta di caricare la `vk_swiftshader.dll` inclusa.

**Elemento 2: Windows Code Integrity blocca la DLL dell'app stessa.** Il processo GPU viene eseguito con il criterio di protezione "MicrosoftSignedOnly" (verificabile tramite `Get-ProcessMitigation`). Affinché un'app Store possa caricare le proprie DLL firmate dal produttore, il pacchetto MSIX deve includere un catalogo di firme `AppxMetadata\CodeIntegrity.cat`. Proprio questo file manca nel pacchetto distribuito, come hanno dimostrato i membri della community ispezionando il file MSIX. La conseguenza: la verifica della firma fallisce, Windows registra Event 3033 e termina forzatamente il processo GPU. Il codice di uscita `0x060C201E` è un errore di integrità AppX del loader Windows, non un codice Chromium; per questo non compare in alcuna fonte Chromium e per questo il processo GPU non lascia nemmeno un crash dump: non esiste alcuna eccezione di cui produrre un dump.

**Elemento 3: la catena di fallback di Chromium si esaurisce.** Quando il processo GPU si arresta, Chromium retrocede di un livello di rendering: GL hardware, poi GL software, poi compositore di visualizzazione puro. Solo quando non resta più alcun livello interviene l'interruzione automatica integrata. Nel codice sorgente della versione inclusa (Chromium 148.0.7778.280 in Electron 42.9.2) è letteralmente così:

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

**Elemento 4: il processo principale si termina deliberatamente.** Questa `LOG(FATAL)` è il momento in cui "l'app si arresta". È documentato da un minidump del processo principale: `EXCEPTION_BREAKPOINT` (un `int3` intenzionale, non un errore del driver), nessuna singola DLL del driver grafico nel processo e, in chiaro nella memoria:

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

Il fatto che questo dump esista è stata la parte più difficile dell'analisi: l'integrazione Sentry dell'app consuma i dump Crashpad al successivo avvio dell'app, li invia alla telemetria del produttore e li cancella localmente. Per questo la cartella Crashpad è sempre vuota per l'utente. Il rimedio è un osservatore indipendente dall'albero dei processi dell'app (avviato tramite WMI affinché l'arresto dell'app non lo termini anch'esso), che scandaglia il database Crashpad ogni 200 millisecondi alla ricerca di `*.dmp` e copia immediatamente altrove i risultati prima che vengano eliminati. L'analisi è eseguita dal pacchetto Python `minidump`, senza WinDbg.

## Perché "disattivare l'accelerazione hardware" peggiora tutto

La catena spiega anche il riscontro più controintuitivo. L'accelerazione hardware disattivata ha qui contemporaneamente due effetti fatali. Primo, forza il percorso SwiftShader, ovvero proprio il tentativo di caricamento della DLL che Code Integrity blocca; con l'accelerazione hardware attiva, invece, `vk_swiftshader.dll` non è quasi mai necessaria. Secondo, il processo GPU parte già dal fondo della catena di fallback: basta un solo arresto e interviene l'elemento 4. Questo spiega anche l'osservazione nel thread della community secondo cui un blocco Code Integrity talvolta non produce conseguenze e talvolta chiude l'app: dipende da quanti livelli di fallback restano al processo browser.

Particolarmente sfortunato: l'app disponeva di una disattivazione automatica dell'accelerazione hardware dopo problemi (`isHardwareAccelerationAutoDisabled`). Pensata come misura di stabilità, portava i sistemi interessati proprio nella configurazione in cui il successivo arresto costa l'intera app.

## Appendice 27.08.2026: la sola accelerazione hardware non basta

La prima versione di questo articolo raccomandava l'accelerazione hardware attiva come misura immediata più efficace, e per due giorni l'app è rimasta effettivamente senza arresti. Poi è arrivato l'aggiornamento automatico alla 1.37937.3 e, con esso, tre arresti in un pomeriggio, ciascuno con il noto Event 3033 relativo a `vk_swiftshader.dll`. Da ciò emergono due riscontri:

Primo, il catalogo di firme mancante è assente anche nel pacchetto MSIX attuale; il problema di fondo rimane invariato nella 1.37937.3.

Secondo, l'accelerazione hardware attiva protegge solo statisticamente: allunga la catena di fallback, ma non impedisce che Chromium la percorra comunque fino al livello SwiftShader sotto carico o dopo un errore del processo GPU hardware. Non appena questo accade, Code Integrity blocca la DLL e la catena può comunque esaurirsi. Inoltre, i flag di configurazione `isHardwareAccelerationDisabled`/`isHardwareAccelerationAutoDisabled` sono scomparsi da `config.json` nella 1.37937.x; non è più possibile fissare l'impostazione lì.

Rimane quindi come unica soluzione affidabile il passaggio all'installazione classica descritto sopra. Dal passaggio sul sistema qui analizzato: stessa versione dell'app, utilizzo identico inclusa l'area browser, nessun singolo Event 3033 e nessun arresto.

## Il sintomo conseguente: il ciclo di riparazione

Il fallimento di Code Integrity ha un effetto secondario che molti interessati ritengono un problema separato: dopo l'evento, Windows talvolta classifica il pacchetto dell'app come "Modified, NeedsRemediation". L'app quindi non si avvia più affatto finché non la si reimposta tramite Impostazioni → App → Claude → Opzioni avanzate → "Ripara". Chi quindi "deve riparare continuamente l'app" vede lo stesso problema di fondo, solo un anello più avanti: la riparazione corregge lo stato del pacchetto, non la causa; il prossimo arresto segue al successivo tentativo di caricamento della DLL bloccato.

## Stato delle segnalazioni

La causa di packaging è stata segnalata come [#81341](https://github.com/anthropics/claude-code/issues/81341), il thread collettivo con le prove della community è [#81698](https://github.com/anthropics/claude-code/issues/81698), l'analisi del minidump con la spiegazione della catena di fallback è [#89250](https://github.com/anthropics/claude-code/issues/89250), un ulteriore report dettagliato incluso il ciclo di riparazione è [#80444](https://github.com/anthropics/claude-code/issues/80444). La correzione effettiva, un catalogo di firme completo nel pacchetto MSIX, spetta ad Anthropic. Al 25.10.2026, la #81341 è ancora aperta e le segnalazioni di inizio settembre 2026 confermano l'errore anche nella versione 1.40609.1. La #80444 è stata chiusa il 15 settembre 2026 senza indicare una versione corretta, la #89250 il 20 ottobre 2026 automaticamente per inattività; nessuna delle due issue menziona una correzione. Fino ad allora vale quanto segue: passare all'installazione classica; chi deve rimanere con il pacchetto MSIX chiuda disciplinatamente l'area browser e, se necessario, disattivi WebGPU tramite flag. Sul sistema qui analizzato, l'app non ha più avuto arresti dal passaggio all'installazione classica, senza un solo ulteriore Event 3033.

## Cronologia delle modifiche

- **25 ottobre 2026:** Aggiunto: l'issue #81341 risulta ancora aperta al 25.10.2026, l'errore è confermato anche nella versione 1.40609.1. Le issue #80444 e #89250 sono state chiuse senza indicare una correzione.

## Fonti

1.  [Issue GitHub #81698: GPU process crash kills entire app](https://github.com/anthropics/claude-code/issues/81698): Il thread collettivo con le prove della community sulla catena Code Integrity, i dati trasversali ai produttori e la correlazione con il pannello browser.

2.  [Issue GitHub #81341: CIG + vk_swiftshader.dll kills GPU process](https://github.com/anthropics/claude-code/issues/81341): La causa di packaging; catalogo CodeIntegrity mancante nell'MSIX.

3.  [Issue GitHub #89250: analisi del minidump dell'interruzione dell'app](https://github.com/anthropics/claude-code/issues/89250): La seconda metà della catena qui descritta con metodo di cattura dei dump e proposte di correzione.

4.  [Issue GitHub #80444: arresto GPU con analisi forense e ciclo di riparazione](https://github.com/anthropics/claude-code/issues/80444): Report individuale dettagliato con cronologie, analisi del registro eventi e il riscontro che ogni arresto porta il pacchetto nello stato "Modified".

5.  [Claude Desktop: pagina di download ufficiale](https://claude.com/download): Fonte dell'installer Windows classico (x64 e ARM64).

6.  [Codice sorgente Chromium: gpu_data_manager_impl_private.cc (tag 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): La funzione IntentionallyCrashBrowserForUnusableGpuProcess e la logica di fallback.

7.  [Documentazione Electron: child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): L'evento con cui un'app Electron può osservare gli arresti del processo GPU e adottare proprie contromisure.

8.  [Pacchetto Python minidump](https://pypi.org/project/minidump/): Strumento per l'analisi dei dump (record delle eccezioni, elenco dei moduli, stringhe in memoria).

9.  [webgpureport.org](https://webgpureport.org/): Pagina diagnostica WebGPU; è servita come trigger minimo per l'arresto di controllo e per il test comparativo nel Chromium attuale.
