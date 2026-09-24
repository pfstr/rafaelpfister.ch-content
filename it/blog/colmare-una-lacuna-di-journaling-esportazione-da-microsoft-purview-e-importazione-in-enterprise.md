---
title: "Colmare una lacuna di journaling: esportazione da Microsoft Purview e importazione in Enterprise Vault"
navTitle: "Lacuna di journaling"
description: "Se i rapporti di journaling di Exchange Online non vengono generati per alcuni giorni, non è possibile crearli retroattivamente. Tuttavia, il contenuto si trova ancora nelle cassette postali. Come esportarlo in formato PST tramite la nuova eDiscovery di Purview, perché il PST Migrator di Enterprise Vault nella maggior parte dei casi non è adatto e come eseguire la reimportazione tramite la cassetta postale di journaling senza spostare il flusso di journaling in corso, con due script generici per l’importazione, lo spostamento e la registrazione."
date: "2026-09-22"
kategorie: "Archiviazione e journaling"
timeToRead: "18 min di lettura"
themen:
  - archivierung-journaling
  - microsoft-365-exchange
  - exchange-onprem-hybrid
produkte:
  - "exchange-online"
  - "exchange-on-premises"
protokolle:
  - "backup-dr"
  - "powershell"
  - "troubleshooting"
slug: "colmare-una-lacuna-di-journaling-esportazione-da-microsoft-purview-e-importazione-in-enterprise"
translationId: "article-5b24118ac7567d96"
aiPrompt: |
  Du bist mein Exchange- und Archivierungsassistent. Für einen Zeitraum sind keine Journalberichte im Enterprise-Vault-Journalarchiv angekommen. Hilf mir Schritt für Schritt: Zeitraum der Lücke aus den Postfachstatistiken bestimmen, Aufbewahrungsrichtlinien prüfen, die Suche in der neuen Purview-eDiscovery mit korrekt geklammerter KQL-Abfrage anlegen, den Export als PST je Postfach ohne Ordnerstruktur konfigurieren, die PST-Dateien per New-MailboxImportRequest in das Journalpostfach importieren, die importierten Unterordner per EWS in den Posteingang leeren und den Nachimport so drosseln, dass der laufende Journalstrom Vorrang behält. Weise mich auf die Grenzen hin: fehlende Umschlagempfänger und Mehrfachkopien ohne Entdopplung.
translationOf: journaling-luecke-purview-export-enterprise-vault-import
url: https://rafaelpfister.ch/it/blog/colmare-una-lacuna-di-journaling-esportazione-da-microsoft-purview-e-importazione-in-enterprise
translationSourceHash: 532fe3454abb44c445145c6fd3e7424685e8bc5a28dfa18d25cff8a7f5516094
translationModel: gpt-5.6-terra
translatedAt: 2026-09-24T08:39:50.612Z
translationReview: automatic
---

# Colmare una lacuna di journaling: esportazione da Microsoft Purview e importazione in Enterprise Vault

Il journaling è un processo di trasporto. Exchange genera il rapporto di journaling nel momento in cui il messaggio transita nel trasporto e lo recapita alla cassetta postale di journaling. Se questa consegna fallisce, ad esempio perché un connettore o un oggetto destinatario è configurato in modo errato, non esiste alcuna funzione che recuperi il rapporto in seguito. Ciò che è transitato nel trasporto in quel periodo manca dall’archivio.

Il contenuto dei messaggi resta però nelle cassette postali dei soggetti coinvolti e, con un criterio di conservazione senza scadenza, anche quando gli utenti hanno eliminato i messaggi. Ne deriva l’unica via praticabile per il recupero: esportare il periodo da tutte le cassette postali e importare le copie nell’archivio di journaling. La procedura seguente usa la nuova eDiscovery in Microsoft Purview per l’esportazione e Veritas Enterprise Vault 15 per l’importazione; i dati misurati provengono da una reimportazione di diverse centinaia di migliaia di messaggi, durante la quale sono nati i due script.

## Cosa può sostituire un’esportazione e cosa no

Un rapporto di journaling è composto dalla busta e dal contenuto. La busta contiene i destinatari effettivamente raggiunti dal trasporto: incluse le copie nascoste e i membri risolti delle liste di distribuzione. Una copia nella cassetta postale non contiene queste informazioni. Non è più possibile stabilire chi abbia ricevuto un messaggio in copia nascosta dopo la reimportazione. Il contenuto stesso, invece, è ricostruibile integralmente se sono soddisfatte due condizioni.

In primo luogo, deve essere applicata una conservazione alle cassette postali, affinché anche i messaggi eliminati si trovino ancora negli elementi recuperabili. È possibile verificarlo nella Security & Compliance PowerShell:

```powershell
Connect-IPPSSession
Get-RetentionCompliancePolicy -DistributionDetail |
  Format-List Name, Enabled, Mode, ExchangeLocation, ExchangeLocationException
Get-RetentionComplianceRule |
  Format-List Name, Policy, RetentionComplianceAction, RetentionDuration
```

Un criterio con `RetentionComplianceAction: Keep` e `RetentionDuration: Unlimited` tramite `ExchangeLocation: All` protegge completamente il contenuto. L’elenco delle eccezioni in `ExchangeLocationException` indica le cassette postali alle quali ciò non si applica. Per queste è ricostruibile solo ciò che è ancora presente.

In secondo luogo, un’esportazione conta copie di cassette postali, non messaggi. Un messaggio inviato a quindici destinatari interni compare quindici volte nel risultato. Nel caso descritto, per un singolo giorno 98'131 copie corrispondevano a circa 6'800 messaggi univoci, determinati dal tracciamento dei messaggi. La decisione su come gestire queste copie spetta alla compliance, non all’IT (vedere la sezione sulla deduplicazione).

## Determinare il periodo

L’inizio della lacuna è indicato nella stessa cassetta postale di journaling, sul server Exchange che la ospita. L’ora dell’ultimo rapporto consegnato ne rappresenta l’inizio:

```powershell
Get-MailboxFolderStatistics "journal@example.com" -IncludeOldestAndNewestItems |
  Where-Object { $_.ItemsInFolder -gt 0 } |
  Format-Table Name, ItemsInFolder, OldestItemReceivedDate, NewestItemReceivedDate -AutoSize
```

Lo stesso vale per la cassetta postale indicata in `JournalingReportNdrTo` della configurazione di trasporto. Raccoglie i rapporti di journaling non recapitabili e la sua voce più recente conferma il momento. La fine della lacuna è il momento in cui la regola di journaling è stata modificata verso la destinazione riparata. Tra i due estremi si trova il periodo di esportazione, in UTC.

Il tracciamento dei messaggi fornisce il numero atteso per il confronto: una riga per destinatario e messaggio, con `MessageId`. Un rapporto storico copre l’intero periodo:

```powershell
Connect-ExchangeOnline
$bericht = @{
  ReportTitle   = "Journal-Luecke"
  ReportType    = "MessageTrace"
  StartDate     = "2026-09-10"
  EndDate       = "2026-09-12"
  NotifyAddress = "admin@example.com"
}
Start-HistoricalSearch @bericht
```

I valori `MessageId` univoci di questo rapporto sono il numero rispetto al quale viene misurata la reimportazione finale.

## Esportazione dalla nuova eDiscovery

Microsoft ha disattivato le interfacce classiche per Content Search ed eDiscovery (Standard) il 31 agosto 2025. Ciò che compare ancora in molte guide, in particolare l’opzione di deduplicazione durante l’esportazione, non esiste più nella nuova eDiscovery. La procedura nel portale Purview:

1. In *eDiscovery*, creare o aprire un caso. Per l’esportazione, l’account esecutore necessita del ruolo *eDiscovery Manager* e di una licenza Microsoft 365 E3 o E5.
2. Creare una ricerca. Origine dati: tutte le cassette postali oppure, per un test, una sola cassetta postale.
3. Usare KQL come query. Le parentesi sono decisive, perché `AND` ha una precedenza maggiore rispetto a `OR`:

```text
kind:email AND ((received>=2026-09-10 AND received<2026-09-12)
             OR (sent>=2026-09-10 AND sent<2026-09-12))
```

Senza `kind:email` la ricerca include anche voci di calendario, contatti e attività. Nel caso descritto, ciò ha fatto la differenza tra 473'725 e 98'131 risultati per un giorno. Senza le parentesi esterne, la seconda condizione di data si applica soltanto a `sent`, e il risultato contiene messaggi dell’intero patrimonio della cassetta postale.

4. Eseguire *Generate statistics*. Nelle ricerche a livello di tenant, questo accelera notevolmente l’esportazione perché vengono elaborate solo le cassette postali con risultati. Prima dell’esportazione, recuperare le posizioni che terminano con errore mediante *Retry failed locations*; altrimenti mancheranno dal risultato e ciò emergerà solo dopo il download.
5. Selezionare *Export* con le impostazioni seguenti.

| Opzione | Effetto |
|---|---|
| *Export type: Export items with items report* | Messaggi più `items.csv`; il solo rapporto non contiene contenuti |
| *Export format: Create PSTs for messages* | Un PST per cassetta postale anziché singoli file `.msg` |
| *Maximum PST package size* | A partire da questa dimensione, un PST viene suddiviso in parti (`.001.pst`, `.002.pst`) |
| *Maximum .zip package size* | Dimensione dei pacchetti di download; deve essere almeno pari alla dimensione del PST |
| *Organize data from different locations into separate folders or PSTs* | **Attivare**: un file per cassetta postale, denominato `<smtp-adresse>.001.pst` |
| *Include folder and path of the source* | **Disattivare**: tutti i messaggi finiscono nella cartella `Items` del PST anziché nella struttura di cartelle della cassetta postale |
| *Give each item a friendly name* | Nessun effetto nell’esportazione PST |

L’impostazione *Include folder and path of the source* determina l’importazione successiva. Con la struttura delle cartelle, l’importazione crea nella cassetta postale di journaling le cartelle di ogni cassetta postale, nella lingua del rispettivo utente: `Posteingang`, `Posta in arrivo`, `Boîte de réception`. Senza struttura di cartelle, tutto si trova in una cartella `Items`, e la reimportazione dispone di un solo punto dal quale proseguire.

Il download fornisce pacchetti Zip. Microsoft raccomanda 7-Zip o uno strumento equivalente anziché Esplora file di Windows. I pacchetti scadono 14 giorni dopo la creazione. Il file `items.csv` dal rapporto di processo dell’esportazione (in *Process manager*, Export, Reports) dimostra cosa contiene l’esportazione e cosa è stato ignorato; fa parte della documentazione.

### Deduplicazione

La classica Content Search disponeva della casella *Enable de-duplication*: i messaggi con `InternetMessageId`, `ConversationTopic` e `BodyTagInfo` identici venivano esportati una sola volta, mentre le altre posizioni di ritrovamento venivano elencate in `Results.csv`. La nuova eDiscovery non offre questa possibilità nell’esportazione diretta da una ricerca. L’unica strada documentata è un *Review Set*: caricare i risultati della ricerca, eseguire Analytics, usare il filtro generato automaticamente *For Review*, che esclude i duplicati, ed esportare dal Review Set. Ciò richiede eDiscovery Premium e quindi una licenza E5 per l’utente esecutore. In Microsoft Q&A, vari utenti segnalano duplicati nonostante questa procedura; prima dell’esportazione completa è consigliabile un test di un giorno.

Senza deduplicazione vale quanto segue: Enterprise Vault esegue la deduplicazione a livello di archiviazione (allegati e corpi dei messaggi identici vengono archiviati una sola volta), ma non a livello di elemento. Ogni copia della cassetta postale diventa una voce di archivio distinta, un risultato di ricerca distinto e un contatore distinto. In questo caso, il conteggio per la prova non può essere desunto dall’archivio, ma solo da `items.csv` rispetto al tracciamento dei messaggi. La decisione di accettare le copie o riesportare tramite un Review Set viene presa prima dell’importazione, non dopo.

## Le modalità in Enterprise Vault

Enterprise Vault include PST Migrator per i file PST, come procedura guidata nella console di amministrazione (*Archives*, clic destro, *Import PST*) e come variante scriptabile tramite Policy Manager EVPM. Per la reimportazione in un archivio di journaling, questa modalità presenta tre limitazioni.

La prima è la licenza. Il Migrator richiede la funzionalità `EVPSTM` (*Exchange PST Migrator*). In un sito che gestisce solo il journaling, spesso non è concessa in licenza e la procedura guidata si interrompe alla prima pagina con *Required license not installed*. Policy Manager usa lo stesso Migrator e restituisce lo stesso messaggio.

La seconda è la destinazione. La procedura guidata non offre gli archivi di journaling come destinazione, ma solo archivi di cassette postali e archivi Internet Mail. Un’importazione diretta nell’archivio di journaling è possibile solo tramite EVPM con `ArchiveName`, aggirando così le regole di elaborazione del Journaling Task. Come destinazione resta un proprio Shared Archive nel Vault Store del journaling, che la ricerca copre insieme all’archivio di journaling.

La terza è la tracciabilità: il Migrator scrive nel file PST, contrassegnando gli elementi migrati, quindi il patrimonio viene modificato e i rapporti di migrazione per file devono essere confrontati con i numeri dell’esportazione.

La seconda modalità non richiede alcuna licenza aggiuntiva: l’Exchange Journaling Task di Enterprise Vault non archivia solo rapporti di journaling. Se riconosce un elemento come rapporto di journaling, lo estrae e acquisisce i dati della busta. Archivia invece direttamente un elemento ordinario, proprio come ha sempre fatto ai tempi del semplice journaling Exchange senza busta. Importando i file PST nella cassetta postale di journaling, i messaggi vengono scritti dal task regolare nel normale archivio di journaling, con la stessa categoria di conservazione, la stessa indicizzazione e senza modificare l’architettura di archiviazione.

Questo può essere dimostrato con un singolo messaggio di test nella cassetta postale di journaling: se poco dopo non si trova più in alcuna cartella, neppure in *Invalid Journal Report*, e la ricerca lo trova nell’archivio di journaling, il task archivia messaggi ordinari.

Questa modalità presenta due caratteristiche che determinano la procedura. Il task elabora esclusivamente il livello superiore della posta in arrivo, non le sottocartelle. Lo si può vedere dalla cassetta postale di journaling: la cartella di ricerca *Initial Trawl With Pending* sotto *Enterprise Vault Search Folders* conta esattamente gli elementi della posta in arrivo, senza includere le sottocartelle. Inoltre, l’importazione tramite Exchange crea sempre delle cartelle; il modo è descritto nella sezione successiva.

## Importazione nella cassetta postale di journaling

`New-MailboxImportRequest` importa un file PST sul lato server tramite Mailbox Replication Service. Il file deve trovarsi in una condivisione sulla quale *Exchange Trusted Subsystem* dispone del controllo completo, e l’account esecutore richiede il ruolo *Mailbox Import Export*, che per impostazione predefinita non è assegnato a nessuno:

```powershell
Get-ManagementRoleAssignment -Role "Mailbox Import Export" |
  Format-Table RoleAssigneeName, RoleAssigneeType -AutoSize
New-ManagementRoleAssignment -Role "Mailbox Import Export" -User "admin@example.com"
```

L’importazione stessa:

```powershell
$import = @{
  Mailbox          = "journal@example.com"
  FilePath         = "\\EVSERVER01\pstimport$\unzipped\user@example.com.001.pst"
  TargetRootFolder = "Inbox"
  Name             = "NI-user_example.com.001"
  BatchName        = "Nachimport"
}
New-MailboxImportRequest @import
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-Mailbox` | Cassetta postale di destinazione, qui la cassetta postale di journaling |
| `-FilePath` | Percorso UNC del file PST; i percorsi locali vengono rifiutati |
| `-TargetRootFolder` | Cartella sotto la quale viene collocato il contenuto PST; senza indicazione, le cartelle PST vengono mappate alle cartelle della cassetta postale con lo stesso nome |
| `-SourceRootFolder` | Cartella del PST dalla quale importare; in pratica la cartella stessa viene comunque creata (vedere il testo) |
| `-Name` | Nome univoco della richiesta; con migliaia di file viene derivato dal nome file |
| `-BatchName` | Raggruppamento per query e statistiche |

</details>

Tre osservazioni emerse dal test determinano la procedura successiva. Senza `TargetRootFolder` e con un PST strutturato (struttura di cartelle esportata), MRS crea la radice del PST come cartella separata accanto alla posta in arrivo, nel caso di una cassetta postale tedesca `Oberste-Ebene-des-Informationsspeichers` con le sottocartelle `Posteingang`, `Gesendete-Elemente` e così via, oltre a `Recoverable-Items` con `Deletions`. Con `TargetRootFolder "Inbox"` e un PST piatto, tutti i messaggi finiscono in `/Inbox/Items`. E `SourceRootFolder "Items"` in combinazione con `TargetRootFolder "Inbox"` ha prodotto nel test anch’esso `/Inbox/Items`; il livello non è stato rimosso.

In tutti e tre i casi, i messaggi si trovano al di fuori della visuale del Journaling Task. L’ultimo passaggio, spostarli in modo piatto nella posta in arrivo, non può essere eseguito con gli strumenti nativi di Exchange; avviene tramite EWS. L’EWS Managed API è presente come `Microsoft.Exchange.WebServices.dll` nella directory di installazione di Enterprise Vault e il Vault Service Account dispone del controllo completo sulla cassetta postale di journaling, quindi non è necessaria alcuna autorizzazione aggiuntiva. Lo script viene eseguito sul server EV in Windows PowerShell 5.1, poiché la libreria si basa su .NET Framework.

Questo passaggio di spostamento è al contempo il regolatore dell’intera procedura: decide quanti messaggi il task vede contemporaneamente.

## Il test con una cassetta postale

Prima di elaborare migliaia di file, è possibile verificare l’intera procedura con una sola cassetta postale. È opportuno usare la cassetta postale della persona che ha eseguito l’esportazione: ne ha l’autorizzazione, contiene i suoi dati e il file è piccolo. La procedura:

1. Copia del file PST in una cartella dedicata, hash SHA-256 dell’originale in un file CSV.
2. Importazione con `TargetRootFolder "Inbox"`, statistiche con `Get-MailboxImportRequestStatistics`: `ItemsTransferred` è il numero atteso.
3. Statistiche delle cartelle della cassetta postale di journaling: dove si trovano gli elementi?
4. Spostamento da `/Inbox/Items` alla posta in arrivo tramite EWS.
5. Conteggio degli elementi vecchi nella posta in arrivo ogni pochi minuti: gli elementi con `DateTimeReceived` precedente all’inizio del flusso corrente sono quelli importati. Se il numero scende, il task sta archiviando.
6. Confronto: ricerca nell’archivio di journaling (in Discovery Accelerator o nella ricerca EV) con periodo e proprietario della cassetta postale, numero di risultati rispetto a `ItemsTransferred`.

Nel caso descritto sono stati importati 128 elementi e archiviati 126. I due rimanenti erano una bozza e un elemento senza mittente; il task lascia gli elementi senza mittente. Le bozze non sono mai transitate nel trasporto e non appartengono a un journal, perciò lo script di spostamento le esclude tramite il flag `IsDraft`, indipendentemente dalla lingua. Anche gli elementi delle cartelle *Versions* e *Konflikte* (versioni precedenti soggette a conservazione, residui di sincronizzazione) non vengono spostati.

Dopo il test, il registro eventi `Veritas Enterprise Vault` mostra ciò che il task ha rifiutato. L’evento 3071 (*could not be archived as it may be corrupt*) e l’evento 3288 (*no longer archive pending*) si ripetono per gli stessi elementi a ogni passaggio; tali elementi vanno spostati in una cartella esterna alla posta in arrivo, affinché il task non tenti nuovamente di elaborarli a ogni passaggio.

## Script 1: importazione in batch

Il primo script viene eseguito nell’Exchange Management Shell su un server Exchange. Crea richieste di importazione in batch, attende, registra per ogni file lo stato e `ItemsTransferred` in un file CSV e rimuove le richieste completate. Le esecuzioni ripetute saltano i file già presenti nel registro. Il prefisso `NI` indica la reimportazione.

```powershell
$NIPostfach  = "journal@example.com"
$NIQuellen   = @("\\EVSERVER01\pstimport$\unzipped", "\\EVSERVER01\pstimport$\unzipped2")
$NIProtokoll = "\\EVSERVER01\pstimport$\protokoll\import-protokoll.csv"

function Schreibe-NIImport($Ordner, $Datei, $Request, $Status, $Items, $Meldung) {
  [pscustomobject]@{
    Zeit = Get-Date -Format "yyyy-MM-dd HH:mm:ss"; Ordner = $Ordner; Datei = $Datei
    Request = $Request; Status = $Status; Items = $Items; Meldung = $Meldung
  } | Export-Csv -Path $NIProtokoll -Append -NoTypeInformation -Encoding UTF8
}

function Get-NIDateien {
  foreach ($q in $NIQuellen) {
    $o = Split-Path $q -Leaf
    Get-ChildItem $q -Filter *.pst | Sort-Object Name | ForEach-Object {
      [pscustomobject]@{
        Ordner = $o; Datei = $_.Name; Pfad = $_.FullName
        MB = [math]::Round($_.Length / 1MB, 1)
        Request = "NI-" + $o + "-" + ($_.Name -replace "@", "_" -replace "\.pst$", "")
      }
    }
  }
}

function Get-NIErledigt {
  if (Test-Path $NIProtokoll) {
    Import-Csv $NIProtokoll |
      Where-Object { $_.Status -in @("angelegt", "Completed", "CompletedWithWarning") } |
      Select-Object -ExpandProperty Request -Unique
  }
}

function Start-NICharge([int]$Anzahl = 100) {
  $erledigt = @(Get-NIErledigt)
  $alle  = @(Get-NIDateien)
  $offen = @($alle | Where-Object { $erledigt -notcontains $_.Request } | Select-Object -First $Anzahl)
  foreach ($d in $offen) {
    $auftrag = @{
      Mailbox          = $NIPostfach
      FilePath         = $d.Pfad
      TargetRootFolder = "Inbox"
      Name             = $d.Request
      BatchName        = "Nachimport-" + $d.Ordner
      ErrorAction      = "Stop"
    }
    try {
      $null = New-MailboxImportRequest @auftrag
      Schreibe-NIImport $d.Ordner $d.Datei $d.Request "angelegt" "" ""
    } catch {
      Schreibe-NIImport $d.Ordner $d.Datei $d.Request "Fehler-Anlage" "" $_.Exception.Message
    }
  }
  "angelegt: $($offen.Count)   verbleibend: $($alle.Count - $erledigt.Count - $offen.Count)"
}

function Wait-NICharge {
  do {
    $alle = @(Get-MailboxImportRequest -Mailbox $NIPostfach | Where-Object { $_.Name -like "NI-*" })
    $laufend = @($alle | Where-Object { $_.Status -notin @("Completed", "Failed", "CompletedWithWarning") })
    "$(Get-Date -Format HH:mm:ss)  laufend: $($laufend.Count)   fertig: $($alle.Count - $laufend.Count)"
    if ($laufend.Count -gt 0) { Start-Sleep -Seconds 60 }
  } while ($laufend.Count -gt 0)
  foreach ($r in $alle) {
    $s = $r | Get-MailboxImportRequestStatistics
    if (-not $s) { Schreibe-NIImport "" "" $r.Name "Statistik-Fehler" "" ""; continue }
    $ordner = $r.BatchName -replace "^Nachimport-", ""
    $datei  = if ($s.FilePath) { Split-Path $s.FilePath -Leaf } else { "" }
    Schreibe-NIImport $ordner $datei $r.Name $s.Status $s.ItemsTransferred $s.Message
    if ($s.Status -in @("Completed", "CompletedWithWarning")) {
      $r | Remove-MailboxImportRequest -Confirm:$false
    }
  }
  Get-MailboxStatistics $NIPostfach | Format-List ItemCount, TotalItemSize
}

function Resume-NICharge([int]$Anzahl = 200, [int]$MaxLager = 20000) {
  $stat  = Get-MailboxFolderStatistics $NIPostfach
  $lager = ($stat | Where-Object { $_.FolderPath -like "/Inbox/*" } | Measure-Object ItemsInFolder -Sum).Sum
  "in Unterordnern wartend: $lager"
  if ($lager -gt $MaxLager) { "Lager voll, nichts freigegeben"; return }
  Get-MailboxImportRequest -Mailbox $NIPostfach -Status Suspended |
    Select-Object -First $Anzahl |
    Resume-MailboxImportRequest -Confirm:$false
  "freigegeben: $Anzahl"
}

function Get-NIImportStand {
  $p = Import-Csv $NIProtokoll
  $p | Group-Object Status | Select-Object Name, Count | Format-Table -AutoSize
  "Elemente importiert: " + (($p | Where-Object { $_.Status -like "Completed*" } | Measure-Object Items -Sum).Sum)
  "Dateien gesamt: " + @(Get-NIDateien).Count
}
```

<details class="options-details">
<summary>Spiegazione delle funzioni</summary>

| Funzione | Effetto |
|---|---|
| `Get-NIDateien` | Elenco di tutti i file PST dalle cartelle di origine con nome della richiesta derivato; file con lo stesso nome in cartelle diverse, ad esempio un’esportazione per ogni giorno, restano distinguibili |
| `Start-NICharge -Anzahl` | Crea fino a N richieste di importazione per file non ancora registrati |
| `Wait-NICharge` | Attende finché non è più in esecuzione alcuna richiesta, scrive stato e `ItemsTransferred` nel registro, rimuove le richieste completate; quelle non riuscite restano disponibili per l’analisi. Le statistiche e la rimozione avvengono tramite pipeline perché `-Identity` in una shell remota non accetta l’oggetto Identity deserializzato |
| `Resume-NICharge -Anzahl -MaxLager` | Rilascia le richieste sospese solo se nelle sottocartelle della posta in arrivo attendono meno di `MaxLager` messaggi |
| `Get-NIImportStand` | Riepilogo per stato, somma degli elementi importati, numero totale di file |

</details>

È utile conoscere due comportamenti di MRS. Per impostazione predefinita, Exchange consente dieci richieste simultanee verso la stessa cassetta postale di destinazione; tutte le successive restano nella coda con un motivo quale `StalledDueToTarget_MailboxCapacityExceeded` o `StalledDueToTarget_MdbReplication` e avanzano in seguito. Si tratta della gestione del carico di lavoro, non di un problema di capacità, anche se il nome può suggerirlo. Inoltre, `Get-MailboxImportRequest` mostra lo stato con ritardo; le statistiche sono più aggiornate.

La velocità effettiva era di circa sei file PST al minuto, con una dimensione media di pochi megabyte. MRS è quindi nettamente più veloce di Enterprise Vault. Tutto ciò che è stato importato rimane nella cassetta postale di journaling finché il task non lo archivia e lo elimina. `Resume-NICharge` collega quindi il rilascio di ulteriori richieste alla quantità non ancora spostata, affinché la cassetta postale non cresca fino alla dimensione dell’intera esportazione.

## Script 2: spostamento e cadenza

Il secondo script viene eseguito come Vault Service Account sul server EV in Windows PowerShell 5.1. Sposta in modo piatto i messaggi da tutte le sottocartelle della posta in arrivo nella posta in arrivo, esclude le bozze e le cartelle *Versions* e *Konflikte*, e aggiunge nuovi elementi solo quando la posta in arrivo è sotto una soglia.

```powershell
Add-Type -Path "C:\Program Files (x86)\Enterprise Vault\Microsoft.Exchange.WebServices.dll"
$NIProtokoll = "E:\pstimport\protokoll\verschieben-protokoll.csv"
$ews = New-Object Microsoft.Exchange.WebServices.Data.ExchangeService(
  [Microsoft.Exchange.WebServices.Data.ExchangeVersion]::Exchange2013_SP1)
$ews.UseDefaultCredentials = $true
$ews.Url = New-Object Uri("https://mailserver01.example.com/EWS/Exchange.asmx")
$mb = New-Object Microsoft.Exchange.WebServices.Data.Mailbox("journal@example.com")

function Schreibe-NIMove($Schritt, $Wert, $Meldung) {
  [pscustomobject]@{
    Zeit = Get-Date -Format "yyyy-MM-dd HH:mm:ss"; Schritt = $Schritt; Wert = $Wert; Meldung = $Meldung
  } | Export-Csv -Path $NIProtokoll -Append -NoTypeInformation -Encoding UTF8
}

function Get-NIInbox {
  [Microsoft.Exchange.WebServices.Data.Folder]::Bind($ews,
    (New-Object Microsoft.Exchange.WebServices.Data.FolderId(
      [Microsoft.Exchange.WebServices.Data.WellKnownFolderName]::Inbox, $mb)))
}

function Get-NIRueckstand {
  $inbox = Get-NIInbox
  $f = New-Object Microsoft.Exchange.WebServices.Data.SearchFilter+IsLessThan(
    [Microsoft.Exchange.WebServices.Data.ItemSchema]::DateTimeReceived, (Get-Date).AddHours(-2))
  $r = $inbox.FindItems($f, (New-Object Microsoft.Exchange.WebServices.Data.ItemView(1)))
  [pscustomobject]@{ Zeit = Get-Date -Format "HH:mm:ss"; Posteingang = $inbox.TotalCount; Alt = $r.TotalCount }
}

function Get-NIUnterordner {
  $inbox = Get-NIInbox
  $liste = New-Object System.Collections.ArrayList
  $offset = 0
  do {
    $fv = New-Object Microsoft.Exchange.WebServices.Data.FolderView(500, $offset)
    $fv.Traversal = [Microsoft.Exchange.WebServices.Data.FolderTraversal]::Deep
    $res = $inbox.FindFolders($fv)
    foreach ($f in $res.Folders) { $null = $liste.Add($f) }
    $offset += 500
  } while ($res.MoreAvailable)
  $ausgeschlossen = @("Versions", "Konflikte", "Conflicts", "Conflitti", "Conflits")
  $liste | Where-Object { $_.TotalCount -gt 0 -and $_.DisplayName -notin $ausgeschlossen }
}

function Move-NIPortion([int]$Max = 2000) {
  $inbox = Get-NIInbox
  $keinEntwurf = New-Object Microsoft.Exchange.WebServices.Data.SearchFilter+IsEqualTo(
    [Microsoft.Exchange.WebServices.Data.ItemSchema]::IsDraft, $false)
  $n = 0
  foreach ($ordner in Get-NIUnterordner) {
    $runden = 0
    do {
      $seite = $ordner.FindItems($keinEntwurf, (New-Object Microsoft.Exchange.WebServices.Data.ItemView(100)))
      if ($seite.Items.Count -gt 0) {
        $ids = New-Object 'System.Collections.Generic.List[Microsoft.Exchange.WebServices.Data.ItemId]'
        foreach ($i in $seite.Items) { $ids.Add($i.Id) }
        $antwort = $ews.MoveItems($ids, $inbox.Id)
        $fehler = @($antwort | Where-Object { $_.Result -ne "Success" }).Count
        if ($fehler -gt 0) { Schreibe-NIMove "Fehler" $fehler $ordner.DisplayName; break }
        $n += $seite.Items.Count
      }
      $runden++
    } while ($seite.Items.Count -gt 0 -and $n -lt $Max -and $runden -lt 200)
    if ($n -ge $Max) { break }
  }
  Schreibe-NIMove "Verschoben" $n ""
  return $n
}

function Start-NIDauerlauf([int]$Portion = 2000, [int]$MaxPosteingang = 3000) {
  $stop = "E:\pstimport\protokoll\STOP.txt"
  while (-not (Test-Path $stop)) {
    $s = Get-NIRueckstand
    if ($s.Posteingang -gt $MaxPosteingang) {
      "$($s.Zeit)  Posteingang $($s.Posteingang)  alt $($s.Alt)  warte auf EV"
      Schreibe-NIMove "Rueckstand" $s.Alt ("Posteingang " + $s.Posteingang)
      Start-Sleep -Seconds 300
      continue
    }
    $n = Move-NIPortion -Max $Portion
    "$(Get-Date -Format HH:mm:ss)  verschoben $n  (Posteingang vorher $($s.Posteingang))"
    if ($n -eq 0) { Start-Sleep -Seconds 600 }
  }
  "STOP-Datei gefunden, Dauerlauf beendet"
}

function Get-NIMoveStand {
  $p = Import-Csv $NIProtokoll
  "verschoben gesamt: " + (($p | Where-Object { $_.Schritt -eq "Verschoben" } | Measure-Object Wert -Sum).Sum)
  "Fehler: " + @($p | Where-Object { $_.Schritt -eq "Fehler" }).Count
  Get-NIRueckstand
}
```

<details class="options-details">
<summary>Spiegazione delle funzioni</summary>

| Funzione | Effetto |
|---|---|
| `Get-NIRueckstand` | Numero di elementi nella posta in arrivo e numero di elementi più vecchi di due ore; questi ultimi sono quelli reimportati, perché il flusso di journaling in corso contiene solo orari di ricezione recenti |
| `Get-NIUnterordner` | Tutte le sottocartelle della posta in arrivo con contenuto, escluse *Versions* e *Konflikte* |
| `Move-NIPortion -Max` | Sposta fino a N messaggi senza flag di bozza nella posta in arrivo, in pagine da 100 tramite `MoveItems` con un elenco tipizzato `ItemId`; un array PowerShell non corrisponde alla firma |
| `Start-NIDauerlauf -Portion -MaxPosteingang` | Ciclo infinito: misura la posta in arrivo ogni cinque minuti e aggiunge elementi solo quando è sotto la soglia; termina non appena esiste il file `STOP.txt` |
| `Get-NIMoveStand` | Somma degli elementi spostati, numero di errori, arretrato attuale |

</details>

La soglia `MaxPosteingang` è il numero più importante della procedura. Deve essere superiore alla normale riserva di lavoro del task, nel caso descritto circa 1'500 rapporti a 100 messaggi al minuto e circa 20 minuti di ritardo, e sufficientemente bassa affinché il task non lasci mai indietro il flusso corrente. Il motivo è illustrato nella sezione successiva.

## Dati misurati e regolazione

Con le impostazioni standard del Journaling Task, 5 connessioni simultanee al server Exchange e 1'000 elementi per passaggio, Enterprise Vault ha archiviato i messaggi reimportati durante la prima notte a circa 5'600 all’ora. Al mattino, le statistiche delle cartelle hanno mostrato il costo: la posta in arrivo era a 16'500 anziché 1'500. Il flusso di journaling corrente, circa 6'000 rapporti all’ora, era in ritardo di oltre due ore. Il task aveva elaborato anche i messaggi vecchi, anteponendoli ai rapporti correnti.

La capacità del task è la somma di entrambi i flussi. La reimportazione può ricevere solo la parte che resta dopo l’attività quotidiana. Perciò l’esecuzione continua misura la posta in arrivo stessa, non il numero di elementi vecchi, e aggiunge elementi solo quando il flusso è aggiornato.

Tre misurazioni mostrano dove si trova il limite del task. Le code tra il task e Storage Service sono code MSMQ; se quelle di Storage Service sono vuote mentre quelle del task sono piene, il task è rallentato nella lettura da Exchange, non dall’archiviazione:

```powershell
Get-Counter -Counter '\MSMQ Queue(*)\Messages in Queue' -SampleInterval 5 -MaxSamples 6 |
  ForEach-Object {
    $t = $_.Timestamp.ToString("HH:mm:ss")
    $_.CounterSamples |
      Where-Object { $_.InstanceName -like "*enterprise vault*" -and $_.CookedValue -gt 0 } |
      ForEach-Object { "$t  $($_.InstanceName)  $($_.CookedValue)" }
  }
```

Sul lato Exchange, la latenza RPC viene contata per tipo di client, non il numero di richieste; valori inferiori a 10 ms indicano che il server dispone di riserve:

```powershell
$zaehler = @('\MSExchangeIS Client Type(*)\RPC Average Latency', '\Processor(_Total)\% Processor Time')
Get-Counter -ComputerName MAILSERVER01 -Counter $zaehler -SampleInterval 5 -MaxSamples 6 |
  ForEach-Object {
    $t = $_.Timestamp.ToString("HH:mm:ss")
    $_.CounterSamples | Where-Object { $_.CookedValue -gt 0 } |
      ForEach-Object { "$t  $($_.InstanceName)  $($_.Path.Split('\')[-1])  $([math]::Round($_.CookedValue,1))" }
  }
```

E il criterio di limitazione del Vault Service Account: Veritas richiede un criterio con `RcaMaxConcurrency: Unlimited`; con il criterio standard, Exchange limita le connessioni MAPI simultanee per account e ulteriori connessioni del task non vengono stabilite:

```powershell
Get-ThrottlingPolicyAssociation -Identity "svc-ev" | Format-List ThrottlingPolicyId
Get-ThrottlingPolicy | Format-Table Name, IsServiceAccount, RcaMaxConcurrency, EwsMaxConcurrency -AutoSize
```

Se il task rallenta, l’impostazione appropriata si trova nelle proprietà del task (*Enterprise Vault Servers*, server, *Tasks*, Journaling Task, scheda *Settings*):

| Impostazione | Valore predefinito | Effetto |
|---|---|---|
| *Number of concurrent connections to Exchange Server* | 5 | Thread che leggono in parallelo dalla cassetta postale; l’effetto è lineare finché la latenza Exchange e le code Storage restano normali |
| *Maximum number of items per target per pass* | 1000 | Elementi per passaggio; in caso di grande arretrato, riavvia meno spesso |

Le modifiche si applicano dopo il riavvio del task, clic destro, *Stop*, quindi *Start*. È opportuno raddoppiare e poi misurare usando i timestamp nel registro di spostamento, quindi procedere allo stadio successivo. Non è possibile avere un secondo Journaling Task sulla stessa cassetta postale; una cassetta postale di journaling è assegnata a un solo task. La vera parallelizzazione si ottiene con una seconda cassetta postale di journaling e un task dedicato su un secondo server EV, nel cui archivio viene importata la seconda metà dei file. Si tratta di una modifica dell’architettura di archiviazione e deve essere concordata con le operations.

## Cosa emerge ai margini

Una reimportazione di queste dimensioni mette in luce aspetti che nessuno aveva osservato prima. Due di questi provengono dal caso descritto, perché è probabile che si ripetano.

Il monitoraggio segnalava sul server Exchange *RPC Requests/sec* oltre la soglia, con valori predefiniti di 60 e 70 richieste al secondo dal modello di controllo. Il carico di base del server, anche senza importazione, era già oltre la soglia critica. Le richieste al secondo sono un dato di carico; il dato di integrità è la latenza, che era inferiore a un millisecondo. La soglia va ridefinita in base al carico di base ricavato dalla cronologia del monitoraggio, ad esempio al doppio e al triplo del tipico picco giornaliero, mantenendo la soglia di latenza.


Nella posta in arrivo della cassetta postale di journaling c’erano da anni quattro elementi che il task ritentava a ogni passaggio, rifiutandoli con l’evento 3071. Non erano emersi in alcuna analisi perché nessuno leggeva regolarmente il registro eventi `Veritas Enterprise Vault`. Lo stesso valeva per la cassetta postale di raccolta da `JournalingReportNdrTo`: 117'000 rapporti di journaling non recapitabili dal 2020, un’indicazione che l’archiviazione aveva già perso ripetutamente rapporti prima della lacuna attuale.

## Documentazione e conclusione

Al termine della reimportazione vi sono tre file e una ricerca. `items.csv` dall’esportazione Purview documenta ciò che è stato esportato. `import-protokoll.csv` contiene, per file, ciò che Exchange ha acquisito. `verschieben-protokoll.csv` contiene ciò che è stato consegnato al task e quando. La ricerca nell’archivio di journaling nel periodo della lacuna, suddivisa per giorni, restituisce il numero nell’archivio. Le differenze fra questi numeri sono spiegabili: bozze, elementi senza mittente, cartelle *Versions* e *Konflikte*, cassette postali escluse dalla conservazione. Proprio questa spiegazione è la dichiarazione per la compliance, insieme ai due limiti che nessuna reimportazione elimina: destinatari mancanti nella busta e, senza deduplicazione, copie anziché messaggi.

Segue poi la pulizia. Le sottocartelle vuote e le bozze rimaste vengono rimosse dalla cassetta postale di journaling, la condivisione con i file PST viene rimossa e i file PST vengono eliminati non appena concluso il confronto. L’assegnazione del ruolo *Mailbox Import Export* viene revocata; le impostazioni del task restano se le operations conoscono i dati misurati. Anche l’incidente stesso riceve un monitoraggio della consegna di journaling, poiché la lacuna è emersa da una segnalazione casuale, non dal monitoraggio.

## Fonti

1.  [Microsoft Learn: Export search results in eDiscovery](https://learn.microsoft.com/en-us/purview/edisc-search-export): elenco completo delle opzioni di esportazione della nuova eDiscovery, dimensioni dei pacchetti, comportamento di *Organize data* e *Include folder and path*, scadenza dei pacchetti dopo 14 giorni.

2.  [Microsoft Learn: Deduplication in eDiscovery search results](https://learn.microsoft.com/en-us/purview/ediscovery-de-duplication-in-search-results): proprietà di confronto della deduplicazione classica e indicazione della disattivazione delle interfacce classiche il 31 agosto 2025.

3.  [Microsoft Learn: Export items from a review set in eDiscovery](https://learn.microsoft.com/en-us/purview/edisc-review-set-export): modalità di esportazione tramite un Review Set, che nella nuova eDiscovery è l’unica a escludere i duplicati.

4.  [Microsoft Q&A: eDiscovery cases, deduplication on export](https://learn.microsoft.com/en-us/answers/questions/2201291/ediscovery-cases-deduplication-on-export): conferma che l’esportazione diretta non offre più la deduplicazione, con testimonianze sul percorso tramite Review Set.

5.  [Microsoft Learn: New-MailboxImportRequest](https://learn.microsoft.com/en-us/powershell/module/exchange/new-mailboximportrequest): parametri della richiesta di importazione, prerequisiti per condivisione e ruolo *Mailbox Import Export*.

6.  [Microsoft Learn: Mailboxes are stalled during a migration](https://learn.microsoft.com/en-us/troubleshoot/exchange/migration/mailboxes-stalled-during-migration): gestione del carico di lavoro con dieci richieste simultanee per destinazione e stati `StalledDueToTarget` come comportamento previsto.

7.  [Veritas: Enterprise Vault PST Migration, wizard-assisted migration](https://www.veritas.com/support/en_US/doc/95955885-161896939-0/v11744603-161896939): procedura del PST Migrator, requisiti di accesso dello Storage Service e tipi di archivio di destinazione consentiti.

8.  [Veritas VOX: Import PST to a Journal Archive](https://vox.veritas.com/t5/Enterprise-Vault/Import-PST-to-a-Journal-Archive/td-p/272928): esperienze della community sul motivo per cui la procedura guidata non offre archivi di journaling e su come EVPM lavora con `ArchiveName`.
