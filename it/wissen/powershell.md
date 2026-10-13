---
title: "PowerShell: pipeline di oggetti, runtime e amministrazione sicura"
blatt: "powershell"
description: "PowerShell per amministratori di infrastrutture e messaggistica: edizioni e runtime .NET, host, motore, runspace e pipeline di oggetti, moduli e provider, remoting tramite WSMan e SSH, autenticazione e JEA, semantica degli errori e delle modifiche, parallelizzazione, logging, storia tecnica e automazione pronta per la produzione."
fakten:
  - label: Categoria
    wert: Shell · linguaggio di scripting · framework di automazione e configurazione
    href: https://learn.microsoft.com/en-us/powershell/scripting/overview
  - label: Stack tecnologico
    wert: C# · .NET · System.Management.Automation · programmi nativi e API del sistema operativo
    href: https://github.com/PowerShell/PowerShell
  - label: Edizioni
    wert: Desktop su .NET Framework · Core su .NET moderno
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_powershell_editions
  - label: Processo
    wert: powershell.exe per Windows PowerShell · pwsh per PowerShell 6 e versioni successive
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pwsh
  - label: Modello dei dati
    wert: oggetti .NET con tipo, proprietà e metodi anziché esclusivamente testo
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_objects
  - label: Esecuzione
    wert: Host → Engine → Runspace → Pipeline → Command Processor
    href: https://learn.microsoft.com/en-us/powershell/scripting/developer/hosting/runspace-samples
  - label: Estensione
    wert: moduli di script, binari e manifest tramite PSModulePath
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_modules
  - label: Remoting
    wert: WSMan/WinRM o SSH; il codice viene eseguito in un runspace remoto
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements
  - label: Oggetti remoti
    wert: perlopiù snapshot serializzati; i metodi vengono persi durante la deserializzazione
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_output
  - label: Execution Policy
    wert: funzione di sicurezza di Windows contro l'esecuzione involontaria, non un confine di sicurezza
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies
  - label: Delega
    wert: JEA limita identità, endpoint e comandi consentiti
    href: https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/jea/overview
  - label: Evidenza operativa
    wert: versione del codice · risoluzione dei moduli · identità · input · modifica · risultato · log
    href: https://learn.microsoft.com/en-us/powershell/scripting/security/security-features
werbung:
  - newsletter
ctaThemen:
  - powershell
  - microsoft-365-exchange
  - automatisierung
translationSourceHash: a8375bc58323f55f2e87dfe08969d32b9ae12af81cbc67673940011c3245a20a
translationModel: gpt-5.6-terra
translatedAt: 2026-10-12T11:27:50.469Z
translationReview: automatic
---

# PowerShell: pipeline di oggetti, runtime e amministrazione sicura

PowerShell è al tempo stesso shell interattiva, linguaggio di scripting e framework di incorporamento. Questo la distingue sia da una semplice riga di comando sia da un linguaggio di programmazione ordinario: i comandi restituiscono oggetti tipizzati in una pipeline, le applicazioni possono incorporare il motore nei propri processi e i prodotti di amministrazione pubblicano la propria interfaccia gestionale come cmdlet, provider o API HTTP. Microsoft descrive quindi PowerShell come framework multipiattaforma di automazione e configurazione per dati strutturati, interfacce REST e modelli di oggetti ([Microsoft – What is PowerShell?](https://learn.microsoft.com/en-us/powershell/scripting/overview)).

Per l'esercizio non è quindi rilevante soltanto la sintassi. Sono determinanti l'edizione concreta e il runtime .NET, il contesto di processo e runspace, i moduli caricati, l'identità sul sistema di destinazione, il trasporto e l'autorizzazione, la semantica degli errori e la prova di ciò che uno script ha effettivamente modificato. Un comando che funziona in una sessione interattiva non costituisce ancora un'automazione affidabile.

La spiegazione segue un comando attraverso parser, motore, runspace e pipeline di oggetti fino al sistema coinvolto. Moduli, provider, remoting, parallelismo, sicurezza ed esercizio vengono quindi messi in relazione con questo modello di runtime.

PowerShell è contemporaneamente shell, linguaggio di automazione e runtime di amministrazione. È decisivo che i cmdlet trasmettano oggetti .NET tramite una pipeline e che i moduli di prodotto generino da essi chiamate API locali o remote. L'articolo segue questo percorso dal comando fino all'effetto nel sistema di destinazione.

## Architettura del runtime

Un'esecuzione di PowerShell è composta da più livelli:

| Livello | Compito | Stato rilevante per l'amministrazione |
|---|---|---|
| Host | fornisce console, editor, servizio o applicazione incorporata | nome host, interattivo/non presidiato, input/output, profilo |
| Motore | analizza il linguaggio, associa i parametri e coordina i comandi | edizione, versione, .NET, Language Mode, impostazioni |
| Runspace | contesto di esecuzione isolato con Session State | identità, variabili, funzioni, moduli, drive, endpoint |
| Pipeline | collega il Command Processor e trasporta oggetti/stream | ordine, binding, enumerazione, stream di errore e informazione |
| Comando | cmdlet, funzione, script, alias o programma nativo | origine, versione del modulo, parametri, effetti collaterali |
| Adapter/Provider/API | traduce verso .NET, sistema operativo, store o servizio remoto | endpoint, autenticazione, autorizzazioni, rate limit, ambito di guasto |

L'**host** non è il motore. Windows Terminal, la console classica, Visual Studio Code, un Scheduled Task o un'applicazione incorporata in C# possono eseguire lo stesso motore PowerShell in modo diverso. Un **runspace** è l'ambiente operativo nel quale vengono eseguiti i comandi di una pipeline; più runspace all'interno di un processo possono fornire Session State e parallelismo differenti ([PowerShell Glossary](https://learn.microsoft.com/en-us/powershell/scripting/learn/glossary), [Microsoft – Runspace samples](https://learn.microsoft.com/en-us/powershell/scripting/developer/hosting/runspace-samples)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-powershell.svg?v=20260813" title="Interaktive Infografik: PowerShell-Laufzeit von Host, Engine und Runspace über Objektpipeline, Module, Provider und APIs bis zu Remoting, Vertrauensgrenzen und Betriebsnachweis" loading="lazy">
  <a href="/images/kb-interaktiv-powershell.svg?v=20260813">Apri direttamente la grafica interattiva</a>.
</iframe>

Il nome del processo è un importante elemento diagnostico: `powershell.exe` avvia Windows PowerShell, `pwsh` la linea PowerShell basata su .NET moderno. Entrambi possono essere installati affiancati in Windows e vedere quindi moduli, profili, impostazioni predefinite TLS e assembly diversi. L'interfaccia a riga di comando di `pwsh` documenta inoltre come vengono associati file, comandi, codifiche e chiamate non interattive ([Microsoft – about_Pwsh](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pwsh)).

### Inventariare runtime e contesto

[$PSVersionTable](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_automatic_variables) mostra edizione, motore e piattaforma; [`Get-Host`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-host) descrive l'host, non automaticamente la versione sottostante del sistema operativo. Per una segnalazione di errore riproducibile sono necessari almeno percorso del processo, edizione, versione, descrizione .NET, architettura, Language Mode e percorso dei moduli.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für PowerShell-Laufzeitinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Process -Id $PID | Select-Object Path,Id,StartTime
$PSVersionTable
Get-Host
[System.Runtime.InteropServices.RuntimeInformation]::FrameworkDescription
$ExecutionContext.SessionState.LanguageMode
$env:PSModulePath -split [IO.Path]::PathSeparator</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">pwsh -NoProfile -Command '
  Get-Process -Id $PID | Select-Object Path,Id,StartTime
  $PSVersionTable
  Get-Host
  [System.Runtime.InteropServices.RuntimeInformation]::FrameworkDescription
  $ExecutionContext.SessionState.LanguageMode
  $env:PSModulePath -split [IO.Path]::PathSeparator
'</code></pre>
  </div>
</div>

## Edizioni, .NET e compatibilità

Dalla versione PowerShell 5.1 Microsoft distingue le edizioni **Desktop** e **Core**. Desktop viene eseguita sul .NET Framework specifico di Windows; Core viene eseguita su .NET moderno ed è disponibile su Windows, Linux e macOS. Il campo `CompatiblePSEditions` di un manifest di modulo di terze parti è tuttavia solo un metadato: PowerShell non applica automaticamente la compatibilità dichiarata ([Microsoft – about_PowerShell_Editions](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_powershell_editions)).

Windows PowerShell 5.1 e PowerShell 7+ sono quindi due prodotti separati, non due opzioni di avvio della stessa installazione. PowerShell 7 non può sostituire Windows PowerShell in-place; profili, percorsi dei moduli e parti della superficie API differiscono. La guida alla migrazione di Microsoft raccomanda di testare esplicitamente dipendenze e compatibilità dei moduli ([Microsoft – Migrating from Windows PowerShell 5.1 to PowerShell 7](https://learn.microsoft.com/en-us/powershell/scripting/whats-new/migrating-from-windows-powershell-51-to-7)).

La decisione di compatibilità viene presa per ogni workload:

- **Motore:** lo script richiede Windows PowerShell/Desktop oppure PowerShell/Core?
- **Sistema operativo:** richiama cmdlet esclusivi di Windows, COM, Registry, WMI o programmi nativi?
- **Modulo:** è portabile come modulo di script, compilato come modulo binario per il runtime .NET di destinazione corretto oppure utilizzabile solo tramite un processo di compatibilità?
- **Autenticazione:** il runtime supporta il flusso richiesto, l'archivio certificati e la cache dei token?
- **Serializzazione:** l'oggetto necessario resta vivo localmente oppure proviene da un altro processo come snapshot deserializzato?
- **Supporto:** il produttore indica esattamente questa combinazione di motore, modulo e prodotto?

PowerShell 7 può caricare moduli Windows PowerShell selezionati tramite una sessione implicita di remoting/compatibilità. Ciò produce tuttavia oggetti deserializzati e un ulteriore confine di processo; non sostituisce una prova di compatibilità nativa ([Microsoft – Using Windows PowerShell compatibility](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_windows_powershell_compatibility)).

Una volta definite edizione e runtime, la pipeline di oggetti spiega la differenza più importante rispetto alle shell orientate al testo: filtri, ordinamento ed esportazione lavorano sulle proprietà anziché sulle posizioni delle colonne.

## La pipeline di oggetti

Una pipeline Unix classica collega flussi di byte o testo. All'interno di un processo, PowerShell trasmette fondamentalmente oggetti .NET. Un oggetto possiede tipo, proprietà e metodi; il componente successivo della pipeline associa gli input secondo regole di parametro e può accedere alle proprietà senza analizzare nuovamente l'output tabellare ([Microsoft – about_Objects](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_objects), [Microsoft – about_Pipelines](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pipelines)).

La tabella visibile non è l'oggetto. Il sistema di formattazione seleziona viste e genera istruzioni di formattazione solo alla fine della pipeline. `Format-Table` e `Format-List` appartengono quindi al margine di output; se vengono usati nel mezzo di una pipeline di dati, i comandi successivi non ricevono più gli oggetti di amministrazione originali ([Microsoft – about_Format.ps1xml](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_format.ps1xml)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für PowerShell-Objektpipeline">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$service = Get-Service -Name WinRM
$service.GetType().FullName
$service | Get-Member
$service | Select-Object Name,Status,StartType
$service | ConvertTo-Json -Depth 3</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">pwsh -NoProfile -Command '
  $process = Get-Process -Id $PID
  $process.GetType().FullName
  $process | Get-Member
  $process | Select-Object Id,ProcessName,Path
  $process | ConvertTo-Json -Depth 3
'</code></pre>
  </div>
</div>

### Binding, enumerazione e stream

Il binding della pipeline avviene per ogni parametro in base al valore o al nome della proprietà. Le raccolte vengono in genere enumerate elemento per elemento; ciò modifica comportamento e fabbisogno di memoria rispetto a un singolo elenco. Per funzioni affidabili si dichiarano esplicitamente i parametri di pipeline, si elabora l'input nel blocco `process` e non si inquina accidentalmente l'output con valori ausiliari.

PowerShell gestisce diversi stream logici: Success, Error, Warning, Verbose, Debug, Information e Progress. Solo lo stream Success costituisce il normale output di oggetti; gli errori possono essere visibili senza interrompere la pipeline. Reindirizzamento e unione degli stream fanno parte dell'interfaccia di chiamata e devono essere definiti consapevolmente per Scheduled Tasks, CI e monitoraggio ([Microsoft – about_Output_Streams](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_output_streams)).

I programmi nativi restano un punto di transizione. La loro quotatura degli argomenti, codifica dei caratteri, output standard e codici di uscita non seguono completamente il modello dei cmdlet. Un wrapper dovrebbe gestire `$LASTEXITCODE`, i codici di uscita previsti, la codifica e stderr, nonché normalizzare il risultato in un proprio oggetto. La mera presenza di testo su stderr non equivale per ogni programma a un errore.

## Risoluzione dei comandi, moduli e provider

[`Get-Command`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/get-command) mostra quale alias, funzione, cmdlet, script o programma nativo viene effettivamente chiamato per un nome. Questo è più importante della sintassi visibile: un alias o una funzione con lo stesso nome nel profilo può mascherare un cmdlet. La chiamata completamente qualificata `Modulname\\Verb-Noun` rende esplicita l'origine.

I moduli pacchettizzano comandi, dati di tipo e formattazione nonché dipendenze. Un modulo di script usa generalmente `.psm1`, un modulo binario un assembly .NET; un manifest `.psd1` descrive tra l'altro il modulo radice, la versione, i comandi esportati, i requisiti e le edizioni compatibili. Un manifest non è obbligatorio per il caricamento, ma è necessario per la pubblicazione nella PowerShell Gallery ([Microsoft – about_Modules](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_modules), [Microsoft – about_Module_Manifests](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_module_manifests)).

Il motore cerca i moduli tramite `PSModulePath`. Sono possibili più versioni installate; il caricamento automatico e chiamate `Import-Module` imprecise possono quindi selezionare su un server un'implementazione diversa da quella nell'ambiente di test. Uno script di produzione inventaria quindi nome, versione, percorso e repository del modulo e fissa le dipendenze nel deployment anziché reinstallarle senza controllo durante ogni esecuzione ([Microsoft – about_PSModulePath](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_psmodulepath)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für PowerShell-Modulinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Command Get-Mailbox -All |
  Select-Object Name,CommandType,Source,Version,Definition
Get-Module -ListAvailable |
  Sort-Object Name,Version |
  Select-Object Name,Version,Path,CompatiblePSEditions
Get-InstalledModule -ErrorAction SilentlyContinue |
  Select-Object Name,Version,Repository</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">pwsh -NoProfile -Command '
  Get-Command Invoke-RestMethod -All |
    Select-Object Name,CommandType,Source,Version,Definition
  Get-Module -ListAvailable |
    Sort-Object Name,Version |
    Select-Object Name,Version,Path,CompatiblePSEditions
  Get-InstalledModule -ErrorAction SilentlyContinue |
    Select-Object Name,Version,Repository
'</code></pre>
  </div>
</div>

I **provider** forniscono archivi dati gerarchici con un'interfaccia simile al file system. `FileSystem:`, `Registry:`, `Certificate:`, `Variable:`, `Function:` o `WSMan:` appaiono come drive, ma dispongono di capacità ed effetti collaterali diversi. Prima di usare codice generico, `Get-PSProvider` mostra quali operazioni supporta un provider. La specifica storica del linguaggio cita provider e drive come modello fondamentale per dati altrimenti difficilmente accessibili ([Microsoft – PowerShell language specification, basic concepts](https://learn.microsoft.com/en-us/powershell/scripting/lang-spec/chapter-03)).

Un comando locale viene eseguito nel processo corrente; il remoting sposta l'esecuzione in un runspace sul sistema di destinazione. Cambiano così identità, percorso di rete, moduli e accesso alle risorse locali.

## Remoting: client locale, trasporto, runspace remoto

PowerShell Remoting esegue il codice fondamentalmente sul lato di destinazione. Localmente vengono preparati comando e argomenti; trasporto, listener e Session Configuration autenticano la chiamata e creano un runspace remoto. Il relativo sistema operativo, motore, moduli, policy, identità e accesso alla rete determinano il risultato.

In Windows, il remoting classico usa WS-Management tramite WinRM. PowerShell 7 supporta inoltre il remoting SSH in Windows, Linux e macOS. Entrambi i trasporti veicolano dati di PowerShell Remoting, ma non offrono le stesse funzioni endpoint: Microsoft rileva ad esempio che il remoting SSH non supporta la configurazione degli endpoint PowerShell e quindi non supporta JEA ([Microsoft – about_Remote_Requirements](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements), [Microsoft – PowerShell remoting over SSH](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/ssh-remoting-in-powershell)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für PowerShell-Remoting">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-WSMan -ComputerName mail01.example.ch
$session = New-PSSession -ComputerName mail01.example.ch -ConfigurationName Microsoft.PowerShell
Invoke-Command -Session $session -ScriptBlock {
  [pscustomobject]@{
    Host = $env:COMPUTERNAME
    Edition = $PSVersionTable.PSEdition
    Version = $PSVersionTable.PSVersion.ToString()
  }
}
Remove-PSSession $session</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ssh -G admin@mail01.example.ch | grep -E '^(hostname|user|port|identityfile) '
pwsh -NoProfile -Command '
  $s = New-PSSession -HostName mail01.example.ch -UserName admin
  Invoke-Command -Session $s -ScriptBlock {
    [pscustomobject]@{
      Host = [Environment]::MachineName
      Edition = $PSVersionTable.PSEdition
      Version = $PSVersionTable.PSVersion.ToString()
    }
  }
  Remove-PSSession $s
'</code></pre>
  </div>
</div>

Gli strumenti utilizzati sono documentati in [`Test-WSMan`](https://learn.microsoft.com/en-us/powershell/module/microsoft.wsman.management/test-wsman), [`New-PSSession`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/new-pssession), [`Invoke-Command`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/invoke-command) e nella pagina man OpenSSH relativa a [`ssh`](https://man.openbsd.org/ssh).

### Serializzazione e perdita di oggetti

La maggior parte degli oggetti remoti viene serializzata sul sistema di destinazione e ricostruita localmente. Il risultato è uno snapshot con proprietà, ma di norma senza i metodi dell'oggetto live originario; il nome del tipo contiene spesso il prefisso `Deserialized.`. I dati di formattazione e Type-Depth determinano quali proprietà vengono trasmesse. Le modifiche allo snapshot locale non modificano l'oggetto di destinazione ([Microsoft – about_Remote_Output](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_output)).

Filtri, aggregazione e dati sensibili vengono quindi elaborati per quanto possibile **sul sistema di destinazione**, trasferendo solo il risultato necessario. Ciò riduce latenza, volume di serializzazione e fuoriuscita di dati:

```text
schlecht:  200'000 Remoteobjekte → serialisieren → übertragen → lokal filtern
gut:       remote filtern/projizieren/aggregieren → 200 Ergebnisobjekte übertragen
```

Il noto «second hop» non è una particolarità di PowerShell, ma un confine di delega: l'identità che si autentica dal client amministrativo al server A non dispone automaticamente di credenziali riutilizzabili per il server B. Soluzioni come Kerberos Constrained Delegation, CredSSP o un account di servizio modificano il modello di fiducia e vengono impiegate solo con un'analisi esplicita delle minacce ([Microsoft – Making the second hop in PowerShell Remoting](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/ps-remoting-second-hop)).

## Autenticazione, autorizzazione e minimo privilegio

Il remoting separa quattro domande:

1. **Fiducia nel trasporto:** con quale host comunica il client e come vengono verificati la relativa chiave o il certificato?
2. **Autenticazione:** quale persona, Managed Identity, app, certificato o chiave SSH esegue l'accesso?
3. **Autorizzazione:** quale Session Configuration, ruolo e autorizzazione di prodotto consentono quali operazioni?
4. **Identità del processo:** con quale account locale o virtuale viene effettivamente eseguito il comando remoto?

**Just Enough Administration (JEA)** crea endpoint PowerShell limitati. Le Role Capabilities definiscono cmdlet, funzioni, provider e programmi esterni visibili; le Session Configurations associano utenti o gruppi ai ruoli. Account virtuali o gMSA possono eseguire azioni privilegiate senza assegnare all'utente chiamante un account amministrativo permanente. Transcripts e log supportano la tracciabilità ([Microsoft – JEA overview](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/jea/overview)).

JEA non limita automaticamente ogni via di elusione. Vengono verificati congiuntamente comandi visibili, parametri, provider, accesso ai file, tipi .NET richiamabili e programmi esterni. Un editor, compilatore o scriptblock generico apparentemente innocuo può ampliare l'insieme di comandi previsto.

## Execution Policy, firme e controllo delle applicazioni

La PowerShell Execution Policy è in Windows una funzione di sicurezza contro l'esecuzione involontaria di script, **non un confine di sicurezza**. Un utente autorizzato a eseguire comandi può richiamare in altro modo il contenuto di uno script bloccato. Nei sistemi non Windows la policy non viene applicata come in Windows ([Microsoft – about_Execution_Policies](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies)).

`AllSigned` e `RemoteSigned` usano Authenticode per verificare origine e integrità di determinati tipi di file PowerShell. Una firma valida indica tuttavia solo che il file è rimasto invariato dalla firma e corrisponde a un percorso di certificazione attendibile; non prova che il codice sia sicuro o tecnicamente corretto ([Microsoft – about_Signing](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_signing)).

Una catena di controllo affidabile combina:

- Source Control, review e artefatti di build immutabili;
- verifica di firma e hash;
- repository di moduli controllati e versioni delle dipendenze;
- App Control/System Lockdown o controllo applicativo equivalente;
- identità con minimo privilegio ed endpoint di remoting limitati;
- segreti esterni al codice sorgente;
- Script Block/Module Logging, Transcripts e invio centralizzato dei log.

Microsoft classifica Execution Policy come Defense-in-Depth, mentre App Control nel System Lockdown può determinare il Language Mode per runspace. Script Block Logging scrive comandi, funzioni e script elaborati nel PowerShell Operational Log; un logging più intenso può a sua volta acquisire segreti ([Microsoft – PowerShell security features](https://learn.microsoft.com/en-us/powershell/scripting/security/security-features)).

### Verificare firma e artefatto

[`Get-AuthenticodeSignature`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/get-authenticodesignature) verifica in Windows lo stato Authenticode; [`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash) e [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) confrontano un artefatto con un valore previsto pubblicato indipendentemente.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für PowerShell-Artefaktprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$script = 'C:\\Ops\\Deploy-MailPolicy.ps1'
Get-AuthenticodeSignature $script |
  Select-Object Status,StatusMessage,SignerCertificate,TimeStamperCertificate
Get-FileHash $script -Algorithm SHA256
Get-ExecutionPolicy -List</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sha256sum /opt/ops/Deploy-MailPolicy.ps1
pwsh -NoProfile -Command '
  Get-FileHash /opt/ops/Deploy-MailPolicy.ps1 -Algorithm SHA256
  Get-ExecutionPolicy -List
'</code></pre>
  </div>
</div>

## Segreti e identità non interattive

Un `SecureString` non è un vault. Microsoft raccomanda, per i nuovi sviluppi, di sostituire ove possibile le password con certificati, autenticazione Windows o altri metodi senza password. Il modulo SecretManagement fornisce un'interfaccia cmdlet uniforme per Extension Vaults; il vault registrato implementa archiviazione e autenticazione. La registrazione è legata all'utente e ciò deve essere considerato per gli account di servizio e l'automazione headless ([Microsoft – PowerShell security features](https://learn.microsoft.com/en-us/powershell/scripting/security/security-features), [Microsoft – Understanding SecretManagement](https://learn.microsoft.com/en-us/powershell/utility-modules/secretmanagement/get-started/understanding-secretmanagement)).

Per ogni automazione vengono documentati separatamente:

- **Workload Identity:** account di servizio, gMSA, Managed Identity, App Registration o certificato;
- **Origine delle credenziali:** vault, archivio certificati, workload federation o broker interattivo;
- **Autorizzazione:** ruolo locale, ruolo RBAC, API scope o Application Permission;
- **Ciclo di vita:** emissione, rotazione, scadenza, revoca e Break Glass;
- **Luogo di esecuzione:** agent, Scheduled Task, runner, funzione o client amministrativo;
- **Percorso di rete:** DNS, proxy, fiducia TLS, endpoint e rate limit;
- **Ancoraggio di audit:** Run-ID, ID identità, commit del codice e ID di modifica/ticket.

Le variabili d'ambiente e di pipeline sono meccanismi di trasporto, non archivi sicuri. Possono finire in processi figli, dump, output diagnostici o log CI. Anche l'accesso a un vault è limitato al minimo scope dei segreti e a una breve durata.

L'automazione sicura deve riconoscere se un'operazione è davvero fallita. Un messaggio rosso, un'eccezione, un errore non terminante e un codice di uscita nativo sono segnali diversi.

## Semantica degli errori: una riga rossa visibile non è un modello di uscita

PowerShell distingue errori non terminanti, terminanti a livello di istruzione e terminanti a livello di script. Molti cmdlet scrivono per un singolo oggetto un `ErrorRecord` non terminante e proseguono con il resto della pipeline. Un `try/catch` intercetta solo gli errori che escono dal blocco controllato; `-ErrorAction Stop` può elevare gli errori non terminanti appropriati per questa chiamata ([Microsoft – about_Error_Handling](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_error_handling), [Microsoft – about_Try_Catch_Finally](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_try_catch_finally)).

Il codice di produzione definisce quindi:

- quali errori per oggetto sono tollerati e registrati;
- quali errori interrompono l'intera esecuzione;
- quale codice di uscita il processo restituisce a scheduler o CI;
- se un retry è sicuro e idempotente;
- quale ID di correlazione collega errore, oggetto di destinazione e richiesta API;
- quali risorse vengono chiuse nel blocco `finally`;
- come vengono rilevate e compensate le modifiche parziali.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für PowerShell-Fehlerbehandlung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$ErrorActionPreference = 'Stop'
try {
  $result = Invoke-RestMethod -Uri $endpoint -Method Get -TimeoutSec 30
  $result | ConvertTo-Json -Depth 10
}
catch {
  Write-Error -ErrorRecord $_
  exit 1
}
finally {
  Remove-Variable result -ErrorAction SilentlyContinue
}</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">pwsh -NoProfile -NonInteractive -Command '
  $ErrorActionPreference = "Stop"
  try {
    $result = Invoke-RestMethod -Uri $env:OPS_ENDPOINT -Method Get -TimeoutSec 30
    $result | ConvertTo-Json -Depth 10
  } catch {
    Write-Error -ErrorRecord $_
    exit 1
  } finally {
    Remove-Variable result -ErrorAction SilentlyContinue
  }
'</code></pre>
  </div>
</div>

L'esempio usa [`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod), il cui comportamento HTTP e in caso di errore deve essere incluso consapevolmente nella gestione degli errori. Per i programmi nativi viene verificato inoltre `$LASTEXITCODE`. La variabile di processo `$?` e gli stream di errore PowerShell non sostituiscono un contratto documentato sui codici di uscita del programma richiamato.

## Semantica sicura delle modifiche e idempotenza

Un'automazione amministrativa separa **rilevamento, pianificazione, modifica e verifica**. Confronta stato effettivo e stato desiderato mediante ID stabili, produce un insieme esplicito di modifiche ed esegue solo la differenza necessaria. In seguito rilegge lo stato tramite l'interfaccia determinante. «Il cmdlet non ha restituito alcun errore» non è una prova dello stato.

Le Advanced Functions possono offrire, con `SupportsShouldProcess`, i Common Parameters `-WhatIf` e `-Confirm`. La funzione richiama `$PSCmdlet.ShouldProcess()` immediatamente prima dell'effetto collaterale. Microsoft documenta tuttavia limiti nella propagazione di `WhatIfPreference` tra moduli di script; un `-WhatIf` riuscito è quindi un'anteprima utile, non una garanzia transazionale ([Microsoft – Everything about ShouldProcess](https://learn.microsoft.com/en-us/powershell/scripting/learn/deep-dives/everything-about-shouldprocess), [Microsoft – about_CommonParameters](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_commonparameters)).

Un processo di modifica robusto conserva:

1. ID di destinazione immutabile e stato precedente;
2. differenza attesa e approvazione tecnica;
3. identità utilizzata, versione dell'endpoint e del modulo;
4. ID di risposta/richiesta del sistema di destinazione;
5. stato riletto e test funzionale dipendente;
6. regola di rollback o roll-forward a partire dal Point of No Return.

L'idempotenza è una proprietà dell'operazione e del suo modello di stato, non del linguaggio. `Set-*` può essere idempotente, `Add-*` tipicamente no; un'operazione HTTP PUT può essere ripetibile, mentre un POST crea un nuovo oggetto a ogni retry. Prima di un retry automatico vengono chiariti semantica API, ID richiesta e stato lato server.

## Parallelismo, job e runspace

PowerShell può parallelizzare il lavoro tramite Background Jobs, Thread Jobs, Remoting Jobs, `ForEach-Object -Parallel` o propri Runspace Pools. Questi modelli differiscono per isolamento del processo, serializzazione, costi di avvio e stato condiviso. I job remoti e di processo serializzano gli oggetti; Thread Jobs e runspace condividono un processo e possono quindi essere più rapidi, ma anche meno isolati ([Microsoft – about_Jobs](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_jobs), [Microsoft – ForEach-Object -Parallel](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/foreach-object)).

Il parallelismo ottimale è limitato dal sistema di destinazione:

- rate limit API e Retry-After;
- massimo di shell e operazioni di remoting per utente;
- pool di connessioni LDAP/database;
- throttling e costo dei cmdlet Exchange;
- memoria per runspace e dimensione degli oggetti serializzati;
- requisiti di ordine e blocco sul sistema di destinazione;
- budget di errore e capacità di ripresa mirata.

Più thread non aumentano automaticamente il throughput. Uno script amministrativo misura lunghezza della coda, latenza, tasso di successo e throttling, limita la concorrenza e usa jitter/backoff. Memorizza un checkpoint per ogni oggetto di destinazione, anziché ripetere ciecamente l'intero batch dopo un errore parziale.

## Moduli di prodotto e API HTTP

PowerShell è spesso solo il lato client di una Control Plane remota. Il cmdlet può validare localmente i parametri e poi richiamare REST, WSMan, RPC, CIM o un'API proprietaria. Lo spazio degli errori comprende allora motore locale, modulo, acquisizione del token, DNS, proxy, [TLS](/kb/tls), versione API, autorizzazione, throttling e servizio di prodotto.

Il **Microsoft Graph PowerShell SDK** genera cmdlet dai metadati dell'API Graph. Le autorizzazioni restano autorizzazioni Graph; il nome del cmdlet non elimina la semantica Delegated rispetto ad Application, il consenso e le limitazioni delle risorse ([Microsoft – Microsoft Graph PowerShell overview](https://learn.microsoft.com/en-us/powershell/microsoftgraph/overview)). Il modulo **ExchangeOnlineManagement** fornisce cmdlet specifici per Exchange e usa connessioni basate su REST per numerose operazioni; le superfici Exchange e Graph non sono API intercambiabili nonostante le sovrapposizioni ([Microsoft – About the Exchange Online PowerShell module](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell-v2)).

Prima di un'esecuzione produttiva vengono memorizzati:

| Livello | Da registrare |
|---|---|
| Client | motore/edizione, processo, sistema operativo, fuso orario |
| Codice | repository, commit, artefatto firmato, configurazione |
| Dipendenza | nome del modulo, versione, percorso, repository, hash |
| Identità | Object-/App-ID, tipo di autenticazione, Tenant/Realm |
| Autorizzazione | ruoli, scope, consenso, Endpoint/Session Configuration |
| Richiesta | ID destinazione, operazione, Correlation-/Request-ID, momento |
| Risultato | stato, classe di errore, retry, stato modificato |
| Verifica | readback indipendente o verifica del protocollo |

La ricerca degli errori inizia quindi dal contesto di esecuzione: edizione, host, utente, lingua, versione del modulo e percorso di remoting. Solo dopo viene esaminato il prodotto di destinazione.

## Diagnosi: prima il contesto di esecuzione, poi il prodotto di destinazione

Un problema PowerShell viene esaminato lungo le dipendenze effettive:

1. **Chiamata:** quale processo, host, utente, Working Directory, modalità profilo e NonInteractive?
2. **Risoluzione:** quale comando e quale versione del modulo vengono realmente usati?
3. **Policy locale:** Language Mode, App Control, Execution Policy, Trust Store e proxy?
4. **Trasporto:** DNS, TCP, TLS, WSMan/SSH o HTTPS sono raggiungibili?
5. **Autenticazione:** quale flow, token/chiave/certificato, orario e tenant?
6. **Autorizzazione:** quale ruolo locale, JEA Capability, API scope e ruoli di prodotto?
7. **Stato di destinazione:** l'oggetto esiste con l'ID stabile utilizzato e nello scope corretto?
8. **Risposta del server:** codice di errore, Request-ID, throttling, log di audit e di prodotto?

[`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) verifica in Windows la raggiungibilità TCP, [`nc`](https://man.openbsd.org/nc) nei sistemi Unix. Ciò non prova né l'autenticazione TLS riuscita né quella dell'applicazione, ma separa un errore di connessione dai livelli superiori. [`Get-WinEvent`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent) legge gli Event Logs in Windows; [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) legge il journal systemd.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für PowerShell-Transport- und Logdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection mail01.example.ch -Port 5986 -InformationLevel Detailed
Get-WinEvent -LogName 'Microsoft-Windows-PowerShell/Operational' -MaxEvents 100 |
  Select-Object TimeCreated,Id,LevelDisplayName,Message
Get-Command Invoke-Command -All |
  Select-Object CommandType,Source,Version,Definition</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz mail01.example.ch 22
journalctl --since '-30 min' --no-pager -u sshd
pwsh -NoProfile -Command '
  Get-Command Invoke-Command -All |
    Select-Object CommandType,Source,Version,Definition
'</code></pre>
  </div>
</div>

## Logging, transcript e protezione dei dati

PowerShell può usare Script Block Logging, Module Logging, Protected Event Logging e Transcription. In Windows, gli eventi Script Block vengono visualizzati nel canale `Microsoft-Windows-PowerShell/Operational`. [`Start-Transcript`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.host/start-transcript) registra la rappresentazione testuale di una sessione; non è un audit log strutturato completo e non rileva ogni attività esterna all'host ([Microsoft – about_Logging](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_logging)).

Il logging aumenta la visibilità e contemporaneamente il rischio di archiviare token, password, contenuti di e-mail o dati personali. Microsoft raccomanda Protected Event Logging per i log Script Block sensibili. Vengono inoltre definiti conservazione, accesso, inoltro centralizzato, sincronizzazione temporale e archiviazione protetta da manomissioni ([Microsoft – PowerShell security features](https://learn.microsoft.com/en-us/powershell/scripting/security/security-features)).

Per le proprie automazioni, gli eventi strutturati sono più adatti dei testi della console formattati liberamente. Un evento contiene almeno ora in UTC, Run-ID, versione del codice, ID di destinazione, azione, risultato, durata, Request-ID e una causa di errore classificata. Segreti e payload non necessari vengono rimossi prima della registrazione.

## Controllo qualità e deployment

[`Invoke-ScriptAnalyzer`](https://learn.microsoft.com/en-us/powershell/utility-modules/psscriptanalyzer/overview) esamina il codice PowerShell con regole statiche. Rileva molti problemi di stile, compatibilità e sicurezza, ma non sostituisce né i test né un'integrazione di prodotto. Una pipeline affidabile comprende:

- controllo del parser e dell'analizzatore;
- Unit Tests per funzioni pure e mappatura dati;
- Contract Tests rispetto a schemi API e risposte mock;
- test di integrazione con autorizzazioni minime;
- output `-WhatIf`/piano, ove tecnicamente possibile;
- test di insiemi vuoti, duplicati, paginazione, throttling ed errori parziali;
- firma, hash e artefatto immutabile;
- rollout graduale con metriche e criteri di interruzione;
- verifica readback indipendente sul sistema di destinazione.

I profili vengono in genere esclusi dall'automazione produttiva con `-NoProfile`. I processi non interattivi usano `-NonInteractive`, codifiche definite, percorsi espliciti e un fuso orario fisso. Il deployment distribuisce i moduli oppure usa un repository interno controllato; non installa indiscriminatamente la dipendenza più recente a ogni esecuzione.

## Storia tecnica

Jeffrey Snover formulò nel **Monad Manifesto** un modello amministrativo uniforme e orientato agli oggetti: piccoli strumenti, denominazione coerente, funzioni comuni di parser e pipeline nonché interfacce amministrative tramite .NET. Microsoft descrive il documento come l'origine dello sviluppo che in seguito divenne PowerShell ([PowerShell Team – Monad Manifesto](https://devblogs.microsoft.com/powershell/monad-manifesto-the-origin-of-windows-powershell/)).

Nell'aprile 2006 Microsoft rinominò in **Windows PowerShell** il progetto sviluppato con il nome in codice Monad; la prima linea di prodotto era strettamente legata a Windows e .NET Framework ([Microsoft Windows Server Blog – Monad’s new name](https://www.microsoft.com/en-us/windows-server/blog/2006/04/25/monads-new-name-windows-powershell/)). In questa linea nacquero le convenzioni verbo-sostantivo dei cmdlet, la pipeline di oggetti, provider, remoting, moduli, workflow e Desired State Configuration.

Nel 2016 Microsoft pubblicò PowerShell con licenza MIT e sviluppò una linea multipiattaforma basata su .NET Core. PowerShell Core 6 introdusse il supporto per Linux/macOS e il remoting SSH; PowerShell 7 riprese il nome del prodotto senza «Core». La base di codice e lo sviluppo per PowerShell 7+ si trovano nel repository pubblico; le modifiche non vengono riportate in Windows PowerShell 5.1 ([PowerShell GitHub repository](https://github.com/PowerShell/PowerShell), [PowerShell Team – Announcing PowerShell 7](https://devblogs.microsoft.com/powershell/announcing-PowerShell-7-0/)).

Questa storia spiega molti limiti operativi attuali: due edizioni in Windows, moduli con dipendenze da .NET Framework, diversi trasporti di remoting e una combinazione di interfacce di amministrazione Windows locali e API HTTP multipiattaforma. Per gli amministratori, PowerShell è quindi meno «un numero di versione» che una combinazione concreta di motore, runtime, modulo, trasporto e prodotto di destinazione.

## Ciò che un amministratore dovrebbe annotare a colpo d'occhio

Uno script è gestibile solo quando, oltre al codice, sono noti anche runtime, moduli, identità, sistema di destinazione e gestione degli errori. La tabella riassume l'evidenza minima necessaria.

| Domanda | Evidenza minima |
|---|---|
| Quale PowerShell è in esecuzione? | percorso del processo, PSEdition, PSVersion, .NET, OS/architettura |
| Cosa viene eseguito? | CommandType, Source, Version, Definition risolti |
| In quale contesto? | host, runspace, modalità profilo, Working Directory, Language Mode |
| Con quali dipendenze? | percorso del modulo, manifest, versione, repository, hash |
| Contro quale destinazione? | endpoint, Tenant/Realm, ID oggetto stabile, versione API/sessione |
| Con quale identità? | utente/app/Managed Identity, auth flow, ID certificato/chiave |
| Con quali diritti? | ruolo locale, endpoint JEA, scope, RBAC del prodotto |
| Cosa deve cambiare? | differenza effettivo/desiderato, piano, scope, approvazione |
| Cosa è realmente cambiato? | risposta del server, Request-ID, readback e test funzionale |
| Come viene gestito un errore parziale? | ErrorAction, codice di uscita, retry, checkpoint, compensazione |
| Come viene eseguito l'audit? | Run-ID, log strutturati, Transcript/Operational Log, conservazione |
| Come viene ripetuto o annullato? | chiave di idempotenza, Point of No Return, roll-forward/-back |

## Fonti

- [Microsoft Learn – Invoke-RestMethod](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod)
- [Microsoft – What is PowerShell?](https://learn.microsoft.com/en-us/powershell/scripting/overview)
- [Microsoft – PowerShell Glossary](https://learn.microsoft.com/en-us/powershell/scripting/learn/glossary)
- [Microsoft – Runspace samples](https://learn.microsoft.com/en-us/powershell/scripting/developer/hosting/runspace-samples)
- [Microsoft – about_Pwsh](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pwsh)
- [Microsoft – about_Automatic_Variables](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_automatic_variables)
- [Microsoft – Get-Host](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-host)
- [Microsoft – about_PowerShell_Editions](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_powershell_editions)
- [Microsoft – Migrating from Windows PowerShell 5.1 to PowerShell 7](https://learn.microsoft.com/en-us/powershell/scripting/whats-new/migrating-from-windows-powershell-51-to-7)
- [Microsoft – Using Windows PowerShell compatibility](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_windows_powershell_compatibility)
- [Microsoft – about_Objects](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_objects)
- [Microsoft – about_Pipelines](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pipelines)
- [Microsoft – about_Format.ps1xml](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_format.ps1xml)
- [Microsoft – about_Output_Streams](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_output_streams)
- [Microsoft – Get-Command](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/get-command)
- [Microsoft – about_Modules](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_modules)
- [Microsoft – about_Module_Manifests](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_module_manifests)
- [Microsoft – about_PSModulePath](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_psmodulepath)
- [Microsoft – PowerShell language specification, basic concepts](https://learn.microsoft.com/en-us/powershell/scripting/lang-spec/chapter-03)
- [Microsoft – about_Remote_Requirements](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements)
- [Microsoft – PowerShell remoting over SSH](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/ssh-remoting-in-powershell)
- [Microsoft – Test-WSMan](https://learn.microsoft.com/en-us/powershell/module/microsoft.wsman.management/test-wsman)
- [Microsoft – New-PSSession](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/new-pssession)
- [Microsoft – Invoke-Command](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/invoke-command)
- [OpenBSD – ssh](https://man.openbsd.org/ssh)
- [Microsoft – about_Remote_Output](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_output)
- [Microsoft – Making the second hop in PowerShell Remoting](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/ps-remoting-second-hop)
- [Microsoft – JEA overview](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/jea/overview)
- [Microsoft – about_Execution_Policies](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies)
- [Microsoft – about_Signing](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_signing)
- [Microsoft – PowerShell security features](https://learn.microsoft.com/en-us/powershell/scripting/security/security-features)
- [Microsoft – Get-AuthenticodeSignature](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/get-authenticodesignature)
- [Microsoft – Get-FileHash](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – SHA-2 utilities](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Microsoft – Understanding SecretManagement](https://learn.microsoft.com/en-us/powershell/utility-modules/secretmanagement/get-started/understanding-secretmanagement)
- [Microsoft – about_Error_Handling](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_error_handling)
- [Microsoft – about_Try_Catch_Finally](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_try_catch_finally)
- [Microsoft – Everything about ShouldProcess](https://learn.microsoft.com/en-us/powershell/scripting/learn/deep-dives/everything-about-shouldprocess)
- [Microsoft – about_CommonParameters](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_commonparameters)
- [Microsoft – about_Jobs](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_jobs)
- [Microsoft – ForEach-Object -Parallel](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/foreach-object)
- [Microsoft – Microsoft Graph PowerShell overview](https://learn.microsoft.com/en-us/powershell/microsoftgraph/overview)
- [Microsoft – About the Exchange Online PowerShell module](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell-v2)
- [Microsoft – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [Microsoft – Get-WinEvent](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [Microsoft – Start-Transcript](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.host/start-transcript)
- [Microsoft – about_Logging](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_logging)
- [Microsoft – PSScriptAnalyzer overview](https://learn.microsoft.com/en-us/powershell/utility-modules/psscriptanalyzer/overview)
- [PowerShell Team – Monad Manifesto](https://devblogs.microsoft.com/powershell/monad-manifesto-the-origin-of-windows-powershell/)
- [Microsoft Windows Server Blog – Monad’s new name](https://www.microsoft.com/en-us/windows-server/blog/2006/04/25/monads-new-name-windows-powershell/)
- [PowerShell – GitHub repository](https://github.com/PowerShell/PowerShell)
- [PowerShell Team – Announcing PowerShell 7](https://devblogs.microsoft.com/powershell/announcing-PowerShell-7-0/)
