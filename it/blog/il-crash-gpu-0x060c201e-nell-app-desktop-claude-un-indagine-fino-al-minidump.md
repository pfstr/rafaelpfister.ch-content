---
title: "Il crash GPU 0x060C201E nell'app desktop Claude: un'indagine fino al minidump"
navTitle: "Crash GPU 0x060C201E"
description: "L'app desktop Claude si chiude ripetutamente con 'GPU process gone'. All'inizio tutto fa pensare a un bug del driver AMD, poi gli esperimenti smentiscono l'ipotesi e infine un minidump intercettato rivela la causa reale: l'interruzione automatica integrata in Chromium, 'GPU process isn't usable. Goodbye.'"
date: "2026-08-25"
kategorie: "Claude"
timeToRead: "12 min di lettura"
themen:
  - claude
slug: "il-crash-gpu-0x060c201e-nell-app-desktop-claude-un-indagine-fino-al-minidump"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
url: https://rafaelpfister.ch/it/blog/il-crash-gpu-0x060c201e-nell-app-desktop-claude-un-indagine-fino-al-minidump
translationSourceHash: 6bd2b58fe661a5639010e16b417412ca9e85f687bae94531890c8fefaef4050d
translationModel: gpt-5.6-terra
translatedAt: 2026-08-25T04:05:29.845Z
translationReview: automatic
---

Dalla fine di luglio, la mia app desktop Claude si chiude più volte al giorno su Windows. Nessuna finestra di dialogo, nessun messaggio di errore: l'app semplicemente sparisce, insieme a tutte le sessioni Claude Code in esecuzione. Più di 25 volte finora. È ora di smettere di riavviarla e verificare dove si verifica effettivamente l'errore. Anticipo già questo: il principale sospettato iniziale si rivela estraneo ai fatti, e la causa reale è infine scritta nero su bianco in un minidump che l'app non voleva affatto rendere disponibile.

## La traccia nel log

L'app memorizza i suoi log in `%LOCALAPPDATA%\Claude\Logs`, mentre le generazioni precedenti e la configurazione si trovano nel percorso Store virtualizzato `%LOCALAPPDATA%\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude`. In `main.log` compare esattamente la stessa cosa prima di ogni crash:

```text
16:01:38 [info] GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
16:03:34 [info] Starting app { appVersion: '1.34493.1', ... }
```

101457950 in esadecimale è `0x060C201E`. Tenete a mente questo numero: è la firma del bug. Il log della finestra fornisce anche il fattore scatenante: immediatamente prima di ogni crash, una pagina nel browser incorporato dell'app richiede un adattatore WebGPU.

```text
16:01:38 [warn] The powerPreference option is currently ignored
                when calling requestAdapter() on Windows.
16:01:38 [warn] A valid external Instance reference no longer exists.
14:59:15 [warn] WebGL: CONTEXT_LOST_WEBGL: loseContext: context lost
```

Quindi: `navigator.gpu.requestAdapter()` entra nell'enumerazione degli adattatori di Dawn nel processo GPU di Chromium, il processo GPU va in crash e, anziché riavviarlo, l'app chiude l'intera applicazione.

## Sospettato n. 1: il driver grafico

La macchina dispone di una Radeon RX 7900 XT con Adrenalin 32.0.31035.1003; l'app include Electron 42.9.2 con Chromium 148. La spiegazione comoda è sul tavolo: vecchio codice Dawn incontra un driver RDNA3, il driver va in crash, caso chiuso. Comoda, plausibile e, come si vedrà: sbagliata. Ma procediamo con ordine, perché si può confutare solo con degli esperimenti.

Due elementi si sono rivelati in anticipo false piste. La iGPU disabilitata in Gestione dispositivi (stato "Error") è semplicemente il codice 22, disabilitata intenzionalmente. E l'app aveva già disattivato l'accelerazione hardware (`isHardwareAccelerationDisabled: true` in config.json), cosa che non ha minimamente impressionato i crash. Perché questa impostazione aggravhi addirittura il problema si vedrà solo alla fine.

## Esperimento 1: controprova in Chromium attuale

Stesso carico, stessa macchina, browser attuale: webgpureport.org in Chromium 151 inizializza completamente WebGPU, adattatore `amd / rdna-3`, inclusa la creazione del device, senza alcuna anomalia. Il driver attuale con Dawn attuale è quindi pulito.

## Esperimento 2: Electron 42.9.2 stock, percorso hardware

Se Electron 42 non va d'accordo con questo driver, deve essere possibile riprodurlo in 20 righe. Quindi: esattamente la stessa versione di Electron dell'app come puro progetto di test, una finestra, una pagina, una chiamata `requestAdapter()`:

```js
const { app, BrowserWindow, crashReporter } = require('electron');
crashReporter.start({ submitURL: '', uploadToServer: false });
app.on('child-process-gone', (e, d) =>
  console.log('GONE: ' + JSON.stringify(d)));
app.whenReady().then(() => {
  const win = new BrowserWindow({ show: false });
  win.loadFile('index.html'); // ruft requestAdapter() auf
});
```

Risultato con accelerazione hardware: `adapter ok (amd/rdna-3), device ok`. Nessun crash. Il percorso D3D12 di Electron 42 con questo driver funziona perfettamente. L'ipotesi secondo cui "il vecchio codice Dawn non tollera il driver RDNA3" è dunque confutata.

## Esperimento 3: Electron 42.9.2 stock, percorso software come nell'app

L'app però viene eseguita senza accelerazione hardware. Quindi lo stesso esperimento con `app.disableHardwareAcceleration()`, più un contesto WebGL attivo, che in modalità software usa SwiftShader, e `powerPreference: 'high-performance'` nella richiesta dell'adattatore, per riprodurre esattamente la sequenza dei log dell'app:

```text
[renderer] webgl context: WebKit WebGL
[renderer] The powerPreference option is currently ignored
           when calling requestAdapter() on Windows.
[renderer] No available adapters.
[renderer] RESULT: adapter=null
TIMEOUT: no crash after 25s
```

Lo stesso avviso powerPreference del log dell'app, lo stesso percorso del codice fino all'enumerazione dell'adattatore, e poi la risposta corretta: nessun adattatore disponibile, rifiuto pulito, processo vivo. Electron 42.9.2 stock semplicemente non va in crash su questa macchina, indipendentemente dal percorso.

## Esperimento 4: altro hardware, stessa firma

Prima di continuare a indovinare, vale la pena guardare l'issue tracker, e lì diventa chiaro: il crash identico con lo stesso codice di uscita 0x060C201E è stato segnalato più volte, tra l'altro su una GPU per portatile NVIDIA RTX 5080. Nel relativo registro eventi di sistema: nessun evento TDR, nessun reset del driver. Il driver, indipendentemente dal produttore, non è la causa. La causa del crash risiede nel processo GPU dell'app stessa o, come si vedrà subito, nella reazione dell'app al suo crash.

## Esperimento 5: recuperare il minidump che l'app elimina

Fino a qui mancava la prova decisiva: un crash dump. La cartella Crashpad dell'app era vuota dopo ogni crash, il che inizialmente sembrava indicare il crash reporting disattivato. L'elenco dei processi dice altro: è in esecuzione un processo `crashpad-handler`, la cui riga di comando punta al database nel profilo Roaming e a un URL di caricamento segnaposto. È il tipico schema dell'integrazione Sentry nelle app Electron: Crashpad scrive il dump localmente, la libreria Sentry lo consuma al successivo avvio dell'app, lo invia alla telemetria del produttore e lo elimina localmente. I dump dunque esistono, solo non per l'utente.

La soluzione è poco spettacolare: un osservatore indipendente dall'albero dei processi dell'app, avviato tramite WMI affinché il crash dell'app non lo trascini con sé, che scandaglia ogni 200 millisecondi il database Crashpad alla ricerca di `*.dmp` e copia immediatamente altrove i risultati. Poi si provoca il crash in modo mirato: aprire webgpureport.org nel browser incorporato dell'app. Pochi secondi dopo, nella cartella di backup compare un minidump da 35 MB, che Sentry tenta invano di eliminare al successivo avvio dell'app.

## Il minidump: nessun driver in vista

L'analisi con il pacchetto Python `minidump` fornisce tre risultati che ribaltano completamente il quadro:

```text
Exception: EXCEPTION_BREAKPOINT (0x80000003)
Adresse:   Claude.exe+0x5e8a6c9
Prozess:   PID 27660
```

Primo: il processo scaricato non è il processo GPU, bensì il processo **main** dell'app, il cui PID compare nei log dell'app come `electron_main`. Secondo: l'eccezione è un breakpoint, ovvero un `int3` eseguito intenzionalmente. È così che Chromium si chiude quando interviene un `CHECK()` o `LOG(FATAL)`; un errore del driver apparirebbe come Access Violation. Terzo: nell'elenco dei moduli del processo non è caricata neppure una DLL di driver grafico.

E nella memoria del dump compare in chiaro il fatale messaggio di log:

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

## La soluzione: l'interruzione automatica integrata di Chromium

Quella riga non è un malfunzionamento, è intenzionale. Nel codice sorgente di Chromium della versione esattamente inclusa, 148.0.7778.280, in `gpu_data_manager_impl_private.cc` si trova:

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

Viene richiamata da `FallBackToNextGpuMode()`: se il processo GPU va in crash, Chromium retrocede di un livello, da Hardware-GL a Software-GL fino al compositore di visualizzazione puro. Se l'elenco delle modalità di fallback è vuoto, Chromium chiude intenzionalmente il processo del browser, perché senza un processo GPU funzionante non può più nemmeno coordinare il rendering software.

Questo spiega anche perché l'app venga colpita molto più duramente di un browser normale: si avvia con l'accelerazione hardware disabilitata, quindi già all'estremità inferiore della catena di fallback. Se poi una pagina nel browser incorporato richiede WebGPU e il processo GPU software va in crash, non resta più alcun livello su cui Chromium possa ripiegare. La fermata successiva è "Goodbye". In un Chrome normale con accelerazione hardware attiva, lo stesso crash costa un livello di fallback e il browser continua a funzionare.

Particolarmente sfortunato: la configurazione dell'app conosce un campo `isHardwareAccelerationAutoDisabled`, quindi l'app apparentemente disattiva da sola l'accelerazione hardware dopo dei problemi. Pensata come misura contro i crash, questa impostazione accorcia proprio la catena di fallback e rende l'interruzione automatica fatale più probabile anziché meno. Un meccanismo di protezione e un interruttore d'emergenza che si innescano a vicenda.

## Cosa resta del codice di uscita

Resta il processo figlio GPU stesso, che ogni volta innesca la sequenza. Non lascia un proprio crash report, sebbene il gestore Crashpad funzioni dimostrabilmente, dato che pochi secondi dopo ha creato il dump del processo main. Questo suggerisce che il processo GPU non generi un'eccezione normale, ma venga terminato bruscamente, nello stile di `TerminateProcess`, e che proprio da qui provenga il codice di uscita non documentato 0x060C201E. L'ultimo miglio è quindi presso Anthropic: la sua telemetria Sentry riceve i dump che vengono eliminati localmente, compresa la questione se il crash reporting copra affatto il processo GPU.

## Workaround e stato delle segnalazioni

Poiché il fattore scatenante sono le richieste WebGPU delle pagine nel browser incorporato, fino alla correzione aiuta disabilitare WebGPU tramite un flag di Chromium. In un'installazione Store il percorso di installazione cambia a ogni aggiornamento, perciò un piccolo launcher lo risolve nuovamente a ogni avvio:

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

Dopo la modifica: non un solo crash. L'analisi completa è stata segnalata: gli esperimenti di laboratorio e i riferimenti ai duplicati nel primo issue, la valutazione del minidump con la catena causale nel secondo. Le tre correzioni sensate derivano direttamente dai risultati: chiarire la causa del crash nel processo GPU software, i cui dump sono nella telemetria del produttore; disabilitare WebGPU in modo mirato in caso di crash GPU ripetuti invece di lasciare esaurire la catena di fallback; e riconsiderare la disattivazione automatica dell'accelerazione hardware, poiché accorcia la catena.

## Postilla: il workaround non basta, la soluzione è più profonda

Quella stessa sera, un altro crash con firma identica. Il motivo è semplice: il launcher con `--disable-features=WebGPU` ha effetto solo quando l'app viene avviata tramite esso. Con il consueto avvio dal menu Start, l'app viene eseguita senza il flag e, per un'app Store, non esiste un modo pulito per fornire permanentemente flag da riga di comando a un avvio normale.

Ma la soluzione permanente è già nella catena causale di questo articolo: l'interruzione automatica fatale presuppone che la catena di fallback sia vuota, ed è immediatamente vuota solo perché l'app si avvia con l'accelerazione hardware disabilitata. Occorre quindi riattivare l'accelerazione hardware nella `config.json` dell'app:

```json
"isHardwareAccelerationDisabled": false
```

Questo ha effetto dal successivo avvio dell'app e risolve entrambi gli aspetti del problema in una volta sola. Primo, `requestAdapter()` viene allora eseguito tramite il percorso hardware, che su questa macchina si è dimostrato stabile, come mostra l'esperimento 2: adattatore e device senza errori. Secondo, Chromium torna ad avere livelli di fallback in riserva: qualora il processo GPU dovesse andare nuovamente in crash, il browser passa al rendering software e continua a funzionare invece di chiudersi. La disattivazione originaria dell'accelerazione hardware, probabilmente impostata tempo fa come misura di stabilità, era in realtà il presupposto del crash.

La conclusione della ricerca dell'errore: la spiegazione più ovvia, "era il driver", avrebbe portato a un'odissea infruttuosa tra i driver. A confutarla sono bastate due ore di laboratorio con la versione reale del motore, e la causa è emersa solo nel minidump che l'app elimina di routine. Quando un processo GPU va in crash, prima di attribuire la colpa a un produttore occorre quindi iniziare con quattro verifiche: la controprova nel browser attuale, la controprova nella versione pura del motore, verificare se altro hardware mostra la stessa firma e il dump del processo che decide effettivamente l'interruzione.

## Fonti

1.  [Causa principale: 'GPU process isn't usable. Goodbye.' di Chromium (GitHub-Issue #89250)](https://github.com/anthropics/claude-code/issues/89250): L'analisi del minidump di questo articolo come bug report, inclusi metodo di acquisizione e proposte di correzione.

2.  [Primo report personale con risultati di laboratorio (GitHub-Issue #89226)](https://github.com/anthropics/claude-code/issues/89226): Gli esperimenti da 1 a 3 e la correzione dell'ipotesi AMD, con riferimenti ai duplicati.

3.  [Il crash del processo GPU chiude l'intera app (GitHub-Issue #81698)](https://github.com/anthropics/claude-code/issues/81698): Lo stesso crash con identico codice di uscita su NVIDIA RTX 5080, senza eventi TDR; scagiona i driver grafici.

4.  [Codice sorgente Chromium: gpu_data_manager_impl_private.cc (tag 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): La funzione IntentionallyCrashBrowserForUnusableGpuProcess e la logica di fallback.

5.  [Documentazione Electron: child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): L'evento con cui un'app Electron può osservare i crash del processo GPU e adottare proprie contromisure.

6.  [Pacchetto Python minidump](https://pypi.org/project/minidump/): Strumento per l'analisi del dump, con record dell'eccezione, elenco dei moduli e stringhe in memoria, senza WinDbg.

7.  [webgpureport.org](https://webgpureport.org/): Pagina diagnostica WebGPU; usata come fattore scatenante minimo e come controprova in Chromium attuale.
