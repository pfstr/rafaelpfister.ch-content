---
title: "Claude Desktop si arresta continuamente: “GPU process gone” con codice di uscita 101457950, causa e soluzione"
navTitle: "Arresto anomalo di Claude Desktop"
description: "L'app Claude Desktop su Windows si chiude completamente con “GPU process gone: exitCode 101457950” (0x060C201E), spesso seguita dalla finestra di riparazione dell'app Store. La catena completa delle cause: Code Integrity blocca vk_swiftshader.dll, la catena di fallback di Chromium si esaurisce, l'auto-arresto integrato chiude l'app. Con una soluzione permanente (passaggio all'installazione classica senza MSIX), autodiagnosi tramite registro eventi e analisi fino al minidump."
date: "2026-08-25"
kategorie: "Claude"
timeToRead: "10 min di lettura"
themen:
  - claude
slug: "il-crash-gpu-0x060c201e-nell-app-desktop-claude-un-indagine-fino-al-minidump"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
translationSourceHash: 769984b49b04b65b0b8f8a91ce3b6dd65e2eef1a4212bed32b83422f431a8559
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:24:34.663Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/il-crash-gpu-0x060c201e-nell-app-desktop-claude-un-indagine-fino-al-minidump
---

L'app Claude Desktop su Windows si chiude senza messaggi di errore, tutte le sessioni Claude Code in corso vanno perse e, talvolta, l'app riparte solo dopo una «Riparazione» dalle impostazioni di Windows. Nel log dell'app compare quindi questa riga:

```text
GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
```

101457950 in esadecimale è `0x060C201E`. Se trovate questa firma nel vostro log, siete nel posto giusto: questo articolo documenta l'intera catena causale di questo arresto anomalo, le misure immediate che rendono nuovamente stabile l'app e l'autodiagnosi con cui confermare il riscontro sul vostro sistema in due minuti. Sono interessate le installazioni MSIX (dal Microsoft Store o tramite setup MSIX) su GPU di tutti i produttori, dalle iGPU Intel a NVIDIA e AMD; l'hardware, anticipiamolo, non è la causa. L'installazione classica senza MSIX non è interessata, ed è precisamente la soluzione.

## La soluzione in breve: passare all'installazione classica

L'errore effettivo risiede nel pacchetto di installazione MSIX e può essere corretto solo da Anthropic (ancora aperto al 27.08.2026, issue [#81341](https://github.com/anthropics/claude-code/issues/81341); è interessata anche l'attuale versione 1.37937.3). La stessa app è però disponibile anche come installazione classica senza MSIX e non è soggetta alla verifica della firma AppX che termina il processo GPU. Il passaggio è quindi l'unica misura che elimina completamente l'arresto anomalo; è confermato sia nell'issue [#81341](https://github.com/anthropics/claude-code/issues/81341) sia sul sistema qui analizzato. Le funzionalità sono identiche e il feed di aggiornamento fornisce le stesse versioni per entrambe le varianti.

**Passaggio 1: scaricare ed eseguire l'installer classico.** Il download da [claude.com/download](https://claude.com/download) fornisce un installer Squirrel che installa l'app in `%LOCALAPPDATA%\AnthropicClaude` (non sono necessari diritti di amministratore). Da riga di comando:

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
| `<url>` | fonte ufficiale dell'installer; identica alla destinazione del reindirizzamento di download da claude.ai |

</details>

Dopo il download, verificate la firma (`Get-AuthenticodeSignature`, previsto: `Valid`, emittente «Anthropic, PBC») e avviate il file. L'installer deposita inizialmente una versione di base meno recente; il meccanismo di aggiornamento la porta alla versione attuale, automaticamente al primo avvio oppure immediatamente tramite:

```powershell
& "$env:LOCALAPPDATA\AnthropicClaude\Update.exe" `
  --update https://downloads.claude.ai/releases/win32/x64
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `--update <url>` | scarica la versione più recente dal feed di release indicato e la installa come nuova directory `app-<version>` |

</details>

**Passaggio 2: trasferire la configurazione.** La versione MSIX conserva accesso, configurazione del server MCP e impostazioni nel proprio contenitore virtualizzato; l'app classica legge `%APPDATA%\Claude`. Copiate una volta sola (prima chiudete l'app MSIX; le due varianti non possono comunque essere eseguite contemporaneamente a causa di un lock di istanza singola condiviso):

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
| `/E` | copia tutte le sottocartelle, comprese quelle vuote |
| `/XD <namen>` | esclude le directory indicate; qui cache e dati di runtime, che la nuova app ricrea autonomamente |

</details>

Le cronologie delle chat non vanno perse: si trovano nell'account claude.ai oppure, per le sessioni Claude Code, in `%USERPROFILE%\.claude` e non dipendono dall'installazione dell'app.

**Passaggio 3: rimuovere il pacchetto MSIX.** Altrimenti le vecchie scorciatoie continueranno ad avviare la variante che va in arresto anomalo:

```powershell
Get-AppxPackage Claude | Remove-AppxPackage
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Claude` | argomento posizionale del nome di `Get-AppxPackage`: filtra i pacchetti AppX/MSIX installati in base al nome del pacchetto (sono consentiti caratteri jolly) |
| `Remove-AppxPackage` | rimuove per l'account utente corrente il pacchetto passato tramite pipeline |

</details>

La voce del menu Start «Anthropic → Claude» appartiene quindi all'installazione classica; un eventuale collegamento fissato alla barra delle applicazioni deve essere creato di nuovo.

## Se dovete rimanere sul pacchetto MSIX

Senza passare all'altra installazione restano solo misure che riducono la frequenza degli arresti anomali senza eliminarne la causa:

**Usare con parsimonia il browser incorporato.** Le pagine nell'area browser/anteprima dell'app innescano l'arresto anomalo. Chi chiude l'area dopo l'uso, anziché lasciare le schede aperte, riduce sensibilmente la frequenza degli arresti; questa correlazione è documentata più volte con dati numerici nel thread della community.

**Disattivare WebGPU.** L'avvio con `--disable-features=WebGPU` impedisce il fattore scatenante più frequente. Con un pacchetto MSIX, il percorso di installazione cambia a ogni aggiornamento; serve quindi un launcher che lo risolva di nuovo a ogni avvio:

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
| `for /f "delims="` | elabora l'output del comando riga per riga; `delims=` vuoto assegna l'intera riga, inclusi gli spazi nel percorso, a `%%i` |
| `-NoProfile` | avvia PowerShell senza script di profilo, per un avvio rapido e riproducibile |
| `-Command` | esegue l'espressione indicata; `(Get-AppxPackage Claude).InstallLocation` restituisce il percorso di installazione corrente del pacchetto |
| `start ""` | avvia il programma separatamente dalla finestra batch; le virgolette vuote sono il titolo della finestra, qui vuoto |
| `--disable-features=WebGPU` | opzione Chromium: disattiva la funzionalità indicata, qui l'API WebGPU |

</details>

Funziona solo se l'app viene avviata anche tramite questo launcher.

Nella prima versione di questo articolo, la raccomandazione principale era attivare l'accelerazione hardware tramite `isHardwareAccelerationDisabled: false` in `config.json`. Questa raccomandazione è superata: nelle versioni attuali (1.37937.x) il flag non esiste più, l'app si avvia per impostazione predefinita con l'accelerazione hardware attiva e ciononostante va in arresto anomalo (dettagli nell'appendice qui sotto).

Peraltro, una «Riparazione» o la reinstallazione del pacchetto MSIX non risolve il problema: cura soltanto il sintomo conseguente (maggiori dettagli sotto). Anche gli aggiornamenti dei driver grafici sono tempo sprecato.

## Autodiagnosi: confermare il riscontro sul proprio sistema

Sono sufficienti due verifiche. La prima è la firma dell'arresto nel log dell'app:

```powershell
Select-String -Path "$env:LOCALAPPDATA\Claude\Logs\main.log" `
  -Pattern 'GPU process gone'
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Path` | file in cui cercare, qui il log principale dell'app |
| `-Pattern` | criterio di ricerca, espressione regolare; restituisce tutte le righe con la firma dell'arresto |

</details>

La seconda, e questa è la prova effettiva, è il log CodeIntegrity di Windows:

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
| `-FilterHashtable` | filtra già al recupero: `LogName` indica il registro eventi, `Id` l'ID evento 3033, blocco di Code Integrity |
| `-MaxEvents 30` | limita l'interrogazione ai 30 risultati più recenti |
| `Where-Object { … -match 'claude' }` | conserva solo gli eventi il cui testo del messaggio contiene il percorso dell'app |
| `Select-Object TimeCreated, Message` | riduce l'output a timestamp e messaggio per il confronto con gli orari degli arresti |

</details>

Sui sistemi interessati troverete voci Event 3033 con timestamp che coincidono al secondo con gli orari degli arresti, con questo messaggio:

```text
Code Integrity determined that a process
(...\WindowsApps\Claude_..._x64__pzs8sxrjxfjjc\app\claude.exe)
attempted to load ...\app\vk_swiftshader.dll that did not meet
the Microsoft signing level requirements.
```

Sul sistema qui analizzato, sette arresti su sette nell'arco di tre settimane coincidevano al secondo con tale evento, incluso un arresto di controllo provocato intenzionalmente.

## La catena causale completa

L'arresto anomalo è l'ultimo anello di una catena di quattro elementi emersa da due analisi: la traccia Code Integrity dal problema della community [#81698](https://github.com/anthropics/claude-code/issues/81698) e una nostra analisi di minidump ([#89250](https://github.com/anthropics/claude-code/issues/89250)).

**Elemento 1: una pagina nel browser incorporato richiede il rendering software.** Un fattore scatenante tipico è una chiamata WebGPU (`navigator.gpu.requestAdapter()`), riconoscibile nel log della finestra da questo avviso immediatamente precedente all'arresto:

```text
[warn] The powerPreference option is currently ignored
       when calling requestAdapter() on Windows.
```

Se l'app viene eseguita senza accelerazione hardware, il percorso passa necessariamente dall'implementazione software Vulkan SwiftShader: il processo GPU tenta di caricare la `vk_swiftshader.dll` inclusa.

**Elemento 2: Windows Code Integrity blocca la DLL dell'app stessa.** Il processo GPU viene eseguito con il criterio di hardening «MicrosoftSignedOnly» (verificabile tramite `Get-ProcessMitigation`). Affinché un'app Store possa caricare le proprie DLL firmate dal produttore, il pacchetto MSIX deve includere un catalogo di firme `AppxMetadata\CodeIntegrity.cat`. Proprio questo file manca nel pacchetto distribuito, come dimostrato da membri della community ispezionando il file MSIX. La conseguenza: la verifica della firma non riesce, Windows registra l'evento 3033 e termina forzatamente il processo GPU. Il codice di uscita `0x060C201E` è un errore di integrità AppX del loader Windows, non un codice Chromium; per questo non compare in alcun sorgente Chromium e per questo il processo GPU non lascia alcun crash dump: non c'è alcuna eccezione di cui eseguire il dump.

**Elemento 3: la catena di fallback di Chromium si esaurisce.** Quando il processo GPU va in arresto, Chromium arretra di un livello di rendering: GL hardware, poi GL software, quindi compositore di visualizzazione puro. Solo quando non resta più alcun livello interviene l'auto-arresto integrato. Nel codice sorgente della versione inclusa (Chromium 148.0.7778.280 in Electron 42.9.2) è riportato letteralmente così:

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

**Elemento 4: il processo principale si chiude intenzionalmente.** Questo `LOG(FATAL)` è il momento in cui «l'app va in arresto anomalo». Lo dimostra un minidump del processo principale: `EXCEPTION_BREAKPOINT` (un `int3` intenzionale, non un errore del driver), nessuna singola DLL del driver grafico nel processo e, in memoria in chiaro:

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

Il fatto che questo dump esista è stata la parte più difficile dell'analisi: l'integrazione Sentry dell'app consuma i dump Crashpad al successivo avvio dell'app, li invia alla telemetria del produttore e li cancella localmente. Per questo la cartella Crashpad è sempre vuota per l'utente. Il rimedio è un osservatore indipendente dall'albero dei processi dell'app, avviato tramite WMI affinché l'arresto dell'app non lo termini anch'esso, che scandaglia ogni 200 millisecondi il database Crashpad alla ricerca di `*.dmp` e copia immediatamente altrove i risultati prima che vengano eliminati. L'analisi è gestita dal pacchetto Python `minidump`, senza alcun WinDbg.

## Perché «disattivare l'accelerazione hardware» peggiora tutto

La catena spiega anche il riscontro più controintuitivo. Qui l'accelerazione hardware disattivata ha contemporaneamente due effetti fatali. Innanzitutto forza il percorso SwiftShader, quindi proprio il tentativo di caricare la DLL che Code Integrity blocca; con l'accelerazione hardware attiva, invece, `vk_swiftshader.dll` non è quasi mai necessaria. In secondo luogo, il processo GPU si avvia già al limite inferiore della catena di fallback: basta un singolo arresto e interviene l'elemento 4. Questo spiega anche l'osservazione nel thread della community secondo cui un blocco Code Integrity talvolta non ha conseguenze e talvolta chiude l'app: dipende da quanti livelli di fallback restano al processo del browser.

Particolarmente sfortunato: l'app prevedeva una disattivazione automatica dell'accelerazione hardware dopo problemi (`isHardwareAccelerationAutoDisabled`). Pensata come misura di stabilità, portava i sistemi interessati proprio nella configurazione in cui il successivo arresto costa l'intera app.

## Appendice 27.08.2026: la sola accelerazione hardware non basta

La prima versione di questo articolo raccomandava l'accelerazione hardware attiva come misura immediata più efficace e, per due giorni, l'app è effettivamente rimasta senza arresti. Poi è arrivato l'aggiornamento automatico alla 1.37937.3 e con esso tre arresti in un pomeriggio, ciascuno con il noto evento 3033 relativo a `vk_swiftshader.dll`. Due riscontri ne derivano:

Primo, il catalogo di firme mancante manca anche nel pacchetto MSIX attuale; il problema di fondo persiste invariato nella 1.37937.3.

Secondo, l'accelerazione hardware attiva protegge solo statisticamente: allunga la catena di fallback, ma non impedisce a Chromium di percorrerla comunque fino al livello SwiftShader sotto carico o dopo un errore del processo GPU hardware. Non appena ciò accade, Code Integrity blocca la DLL e la catena può comunque esaurirsi. Inoltre, i flag di configurazione `isHardwareAccelerationDisabled`/`isHardwareAccelerationAutoDisabled` sono scomparsi da `config.json` nella 1.37937.x; non è più possibile fissare lì l'impostazione.

Di conseguenza, l'unica soluzione affidabile è rimasta il passaggio all'installazione classica descritto sopra. Dal passaggio sul sistema qui analizzato: stessa versione dell'app, utilizzo identico compresa l'area browser, nessun singolo evento 3033 e nessun arresto anomalo.

## Il sintomo conseguente: il ciclo di riparazione

Il fallimento di Code Integrity ha un effetto collaterale che molte persone interessate considerano un problema a sé: dopo l'incidente, Windows talvolta classifica il pacchetto dell'app come «Modified, NeedsRemediation». L'app non si avvia più affatto finché non viene reimpostata tramite Impostazioni → App → Claude → Opzioni avanzate → «Ripara». Chi quindi «deve riparare continuamente l'app» vede lo stesso problema di fondo, solo un anello più avanti: la riparazione corregge lo stato del pacchetto, non la causa; il successivo arresto segue al prossimo tentativo di caricamento DLL bloccato.

## Stato delle segnalazioni

La causa di pacchettizzazione è segnalata come [#81341](https://github.com/anthropics/claude-code/issues/81341), il thread di raccolta con le prove della community è [#81698](https://github.com/anthropics/claude-code/issues/81698), l'analisi del minidump con la spiegazione della catena di fallback è [#89250](https://github.com/anthropics/claude-code/issues/89250), un ulteriore report dettagliato incluso il ciclo di riparazione è [#80444](https://github.com/anthropics/claude-code/issues/80444). La correzione effettiva, un catalogo di firme completo nel pacchetto MSIX, dipende da Anthropic e manca ancora anche nella 1.37937.3. Fino ad allora vale quanto segue: passare all'installazione classica; chi deve rimanere sul pacchetto MSIX chiuda con disciplina l'area browser e, se necessario, disattivi WebGPU tramite flag. Sul sistema qui analizzato, l'app non ha più avuto arresti anomali dal passaggio all'installazione classica, senza un solo ulteriore evento 3033.

## Fonti

1.  [GitHub issue #81698: GPU process crash kills entire app](https://github.com/anthropics/claude-code/issues/81698): il thread di raccolta con le prove della community sulla catena Code Integrity, i dati relativi a produttori diversi e la correlazione con il pannello browser.

2.  [GitHub issue #81341: CIG + vk_swiftshader.dll kills GPU process](https://github.com/anthropics/claude-code/issues/81341): la causa di pacchettizzazione; catalogo CodeIntegrity mancante nel MSIX.

3.  [GitHub issue #89250: analisi del minidump dell'arresto dell'app](https://github.com/anthropics/claude-code/issues/89250): la seconda metà della catena qui descritta, con il metodo di acquisizione del dump e proposte di correzione.

4.  [GitHub issue #80444: crash GPU con analisi forense e ciclo di riparazione](https://github.com/anthropics/claude-code/issues/80444): report individuale dettagliato con cronologie, valutazione del registro eventi e il riscontro che ogni arresto imposta il pacchetto sullo stato «Modified».

5.  [Claude Desktop: pagina ufficiale di download](https://claude.com/download): fonte dell'installer Windows classico, x64 e ARM64.

6.  [Codice sorgente Chromium: gpu_data_manager_impl_private.cc (tag 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): la funzione IntentionallyCrashBrowserForUnusableGpuProcess e la logica di fallback.

7.  [Documentazione Electron: child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): l'evento con cui un'app Electron può osservare gli arresti del processo GPU e adottare contromisure proprie.

8.  [Pacchetto Python minidump](https://pypi.org/project/minidump/): strumento per l'analisi dei dump, record delle eccezioni, elenco moduli e stringhe in memoria.

9.  [webgpureport.org](https://webgpureport.org/): pagina diagnostica WebGPU; utilizzata come trigger minimo per l'arresto di controllo e per il test comparativo nell'attuale Chromium.
