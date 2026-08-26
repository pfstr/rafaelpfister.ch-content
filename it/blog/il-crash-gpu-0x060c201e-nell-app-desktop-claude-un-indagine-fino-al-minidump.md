---
title: "Claude Desktop si blocca continuamente: \"GPU process gone\" con codice di uscita 101457950, causa e soluzione"
navTitle: "Arresto anomalo di Claude Desktop"
description: "L'app Claude Desktop su Windows si chiude completamente con \"GPU process gone: exitCode 101457950\" (0x060C201E), spesso seguita dalla finestra di riparazione dell'app Store. La catena causale completa: Code Integrity blocca vk_swiftshader.dll, la catena di fallback di Chromium si esaurisce, l'interruzione automatica integrata chiude l'app. Con soluzione immediata, autodiagnosi tramite registro eventi e analisi fino al minidump."
date: "2026-08-25"
kategorie: "Claude"
timeToRead: "9 min di lettura"
themen:
  - claude
slug: "il-crash-gpu-0x060c201e-nell-app-desktop-claude-un-indagine-fino-al-minidump"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
translationSourceHash: 2cf7e9455d4d9b5c148e7b55fd0433206810dc26e53bacb85e1d2dc82a0444c6
translationModel: gpt-5.6-terra
translatedAt: 2026-08-26T04:07:41.462Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/il-crash-gpu-0x060c201e-nell-app-desktop-claude-un-indagine-fino-al-minidump
---

L'app Claude Desktop su Windows si chiude senza messaggi di errore, tutte le sessioni Claude Code in corso vengono perse e talvolta l'app si avvia di nuovo solo dopo una "Riparazione" dalle impostazioni di Windows. Nel registro dell'app compare quindi questa riga:

```text
GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
```

101457950 in esadecimale è `0x060C201E`. Se trovate questa firma nel vostro registro, siete nel posto giusto: questo articolo documenta la catena causale completa di questo arresto anomalo, le misure immediate che rendono nuovamente stabile l'app e l'autodiagnosi con cui potete confermare il riscontro sul vostro sistema in due minuti. Sono interessate le installazioni dal Microsoft Store (MSIX) su tutti i produttori di GPU, dalle iGPU Intel a NVIDIA e AMD; l'hardware, anticipiamolo, non è la causa.

## La soluzione in breve

L'errore vero e proprio risiede nel pacchetto di installazione dell'app e può essere corretto solo da Anthropic (ancora aperto al 25.08.2026, issue [#81341](https://github.com/anthropics/claude-code/issues/81341)). Fino ad allora, tre misure rendono stabile l'app, in ordine di efficacia:

**1. Attivare l'accelerazione hardware.** Controllate questi due valori nel file `%LOCALAPPDATA%\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude\config.json` e, se necessario, impostateli su `false` (chiudete prima l'app, quindi riavviatela):

```json
"isHardwareAccelerationDisabled": false,
"isHardwareAccelerationAutoDisabled": false
```

Sembra paradossale, poiché disattivare l'accelerazione hardware è solitamente la scelta più stabile. Con questo bug è il contrario, e il perché è spiegato dalla catena causale più sotto: l'impostazione determina se un arresto del processo GPU costa solo un livello di fallback oppure l'intera app.

**2. Usare con parsimonia il browser incorporato.** Le pagine nell'area browser/anteprima dell'app innescano l'arresto anomalo. Chi chiude l'area dopo l'uso invece di lasciare le schede aperte riduce drasticamente la frequenza degli arresti; questa correlazione è documentata più volte con dati numerici nel thread della community.

**3. Facoltativo: disattivare WebGPU.** Un avvio con `--disable-features=WebGPU` impedisce completamente l'innesco più frequente. Con un'app Store il percorso di installazione cambia a ogni aggiornamento, quindi serve un launcher che lo risolva nuovamente a ogni avvio:

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

Il limite: funziona solo se l'app viene avviata anche tramite questo launcher. La misura 1 funziona a ogni avvio.

Tra l'altro, "Riparare" o reinstallare l'app non risolve il problema, ma cura solo il sintomo conseguente (maggiori dettagli sotto). Anche gli aggiornamenti dei driver grafici sono uno sforzo inutile.

## Autodiagnosi: confermare il riscontro sul proprio sistema

Bastano due verifiche. Primo, la firma dell'arresto nel registro dell'app:

```powershell
Select-String -Path "$env:LOCALAPPDATA\Claude\Logs\main.log" `
  -Pattern 'GPU process gone'
```

Secondo, e questa è la prova vera e propria, il registro Code Integrity di Windows:

```powershell
Get-WinEvent -FilterHashtable @{
  LogName='Microsoft-Windows-CodeIntegrity/Operational'; Id=3033
} -MaxEvents 30 | Where-Object { $_.Message -match 'claude' } |
  Select-Object TimeCreated, Message
```

Sui sistemi interessati troverete voci Event-3033 i cui timestamp coincidono al secondo con gli orari degli arresti, con questo messaggio:

```text
Code Integrity determined that a process
(...\WindowsApps\Claude_..._x64__pzs8sxrjxfjjc\app\claude.exe)
attempted to load ...\app\vk_swiftshader.dll that did not meet
the Microsoft signing level requirements.
```

Sul sistema qui esaminato, sette arresti su sette nell'arco di tre settimane coincidevano al secondo con uno di questi eventi, incluso un arresto di controllo provocato intenzionalmente.

## La catena causale completa

L'arresto anomalo è l'ultimo anello di una catena di quattro elementi emersa da due analisi: la traccia Code Integrity nell'issue della community [#81698](https://github.com/anthropics/claude-code/issues/81698) e un'analisi autonoma del minidump ([#89250](https://github.com/anthropics/claude-code/issues/89250)).

**Elemento 1: una pagina nel browser incorporato necessita del rendering software.** L'innesco tipico è una chiamata WebGPU (`navigator.gpu.requestAdapter()`), riconoscibile nel registro della finestra da questo avviso immediatamente precedente all'arresto:

```text
[warn] The powerPreference option is currently ignored
       when calling requestAdapter() on Windows.
```

Se l'app viene eseguita senza accelerazione hardware, il percorso passa necessariamente attraverso l'implementazione Vulkan software SwiftShader: il processo GPU tenta di caricare la `vk_swiftshader.dll` fornita con l'app.

**Elemento 2: Windows Code Integrity blocca la DLL dell'app stessa.** Il processo GPU viene eseguito con il criterio di hardening "MicrosoftSignedOnly" (verificabile tramite `Get-ProcessMitigation`). Affinché un'app Store possa caricare le proprie DLL firmate dal produttore, il pacchetto MSIX deve includere un catalogo di firme `AppxMetadata\CodeIntegrity.cat`. Proprio questo file manca nel pacchetto distribuito, come hanno dimostrato membri della community ispezionando il file MSIX. Conseguenza: la verifica della firma fallisce, Windows registra l'evento 3033 e termina forzatamente il processo GPU. Il codice di uscita `0x060C201E` è un errore di integrità AppX del loader Windows, non un codice Chromium; per questo non si trova in nessun sorgente Chromium e per questo il processo GPU non lascia nemmeno un crash dump: non esiste un'eccezione di cui produrre un dump.

**Elemento 3: la catena di fallback di Chromium si esaurisce.** Quando il processo GPU si arresta, Chromium retrocede di un livello di rendering: GL hardware, poi GL software, quindi il puro compositore display. Solo quando non rimane più alcun livello interviene l'interruzione automatica integrata. Nel codice sorgente della versione inclusa (Chromium 148.0.7778.280 in Electron 42.9.2) è riportata letteralmente così:

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

**Elemento 4: il processo principale si termina intenzionalmente.** Questo `LOG(FATAL)` è il momento in cui "l'app si blocca". Lo dimostra un minidump del processo principale: `EXCEPTION_BREAKPOINT` (un `int3` intenzionale, non un errore del driver), nessuna DLL del driver grafico nel processo e, in memoria in chiaro:

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

Il fatto che questo dump esista è stata la parte più difficile dell'analisi: l'integrazione Sentry dell'app consuma i dump Crashpad al successivo avvio dell'app, li invia alla telemetria del produttore e li elimina localmente. Per questo la cartella Crashpad è sempre vuota per l'utente. Un osservatore indipendente dall'albero dei processi dell'app offre una soluzione (avviato tramite WMI, in modo che l'arresto dell'app non lo termini a sua volta), che scandaglia il database Crashpad ogni 200 millisecondi alla ricerca di `*.dmp` e copia immediatamente altrove i risultati prima che vengano eliminati. L'analisi viene eseguita dal pacchetto Python `minidump`, senza alcun bisogno di WinDbg.

## Perché "disattivare l'accelerazione hardware" peggiora tutto

La catena spiega anche il riscontro più controintuitivo. L'accelerazione hardware disattivata ha qui contemporaneamente due effetti fatali. Primo, forza il percorso SwiftShader, cioè proprio il tentativo di caricamento della DLL che Code Integrity blocca; con l'accelerazione hardware attiva, invece, `vk_swiftshader.dll` non serve quasi mai. Secondo, il processo GPU parte già all'estremità inferiore della catena di fallback: basta un solo arresto e interviene l'elemento 4. Ciò spiega anche l'osservazione dal thread della community secondo cui un blocco Code Integrity talvolta non ha conseguenze e talvolta chiude l'app: dipende da quanti livelli di fallback restano al processo browser.

Particolarmente sfortunato: l'app dispone di una disattivazione automatica dell'accelerazione hardware dopo problemi (`isHardwareAccelerationAutoDisabled`). Pensata come misura di stabilità, porta i sistemi interessati proprio nella configurazione in cui il successivo arresto costa l'intera app.

## Il sintomo conseguente: il ciclo di riparazione

Il fallimento di Code Integrity ha un effetto collaterale che molti interessati considerano un problema a sé stante: Windows talvolta classifica il pacchetto dell'app dopo l'evento come "Modified, NeedsRemediation". L'app non si avvia più affatto finché non viene ripristinata tramite Impostazioni → App → Claude → Opzioni avanzate → "Ripara". Chi quindi "deve riparare continuamente" l'app vede lo stesso problema di base, soltanto un anello più avanti: la riparazione corregge lo stato del pacchetto, non la causa; il successivo arresto seguirà al prossimo tentativo di caricamento della DLL bloccato.

## Stato delle segnalazioni

La causa di packaging è segnalata come [#81341](https://github.com/anthropics/claude-code/issues/81341), il thread di raccolta con le prove della community è [#81698](https://github.com/anthropics/claude-code/issues/81698), l'analisi del minidump con la spiegazione della catena di fallback è [#89250](https://github.com/anthropics/claude-code/issues/89250). La correzione effettiva, un catalogo di firme completo nel pacchetto MSIX, spetta ad Anthropic. Fino ad allora: accelerazione hardware attiva, chiudere con disciplina l'area browser e, se necessario, disattivare WebGPU tramite flag. Sul sistema qui esaminato, l'app non ha più avuto arresti anomali dall'applicazione della misura 1.

## Fonti

1.  [Issue GitHub #81698: GPU process crash kills entire app](https://github.com/anthropics/claude-code/issues/81698): Il thread di raccolta con le prove della community sulla catena Code Integrity, i dati relativi ai vari produttori e la correlazione con il pannello browser.

2.  [Issue GitHub #81341: CIG + vk_swiftshader.dll kills GPU process](https://github.com/anthropics/claude-code/issues/81341): La causa di packaging; catalogo CodeIntegrity mancante nel MSIX.

3.  [Issue GitHub #89250: Analisi del minidump dell'interruzione dell'app](https://github.com/anthropics/claude-code/issues/89250): La seconda metà della catena qui descritta, con metodo di acquisizione del dump e proposte di correzione.

4.  [Codice sorgente Chromium: gpu_data_manager_impl_private.cc (tag 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): La funzione IntentionallyCrashBrowserForUnusableGpuProcess e la logica di fallback.

5.  [Documentazione Electron: child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): L'evento con cui un'app Electron può rilevare gli arresti del processo GPU e adottare contromisure proprie.

6.  [Pacchetto Python minidump](https://pypi.org/project/minidump/): Strumento per l'analisi dei dump (record di eccezione, elenco moduli, stringhe in memoria).

7.  [webgpureport.org](https://webgpureport.org/): Pagina diagnostica WebGPU; è servita come innesco minimo per l'arresto di controllo e per il test di confronto nel Chromium attuale.
