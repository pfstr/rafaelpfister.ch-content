---
title: "Migrazione EXO senza Remote Move"
navTitle: "Migrazione EXO senza Remote Move"
description: "Come eseguire il provisioning controllato di mailbox Exchange locali come nuove mailbox Exchange Online vuote: backup PST, approvazione CSV, RemoteMailbox, sincronizzazione, convalida e rollback."
date: "2026-08-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "12 min di lettura"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
  - active-directory-entra
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
protokolle:
  - "powershell"
  - "migration"
related:
  - typische-ursachen-fuer-mail-loops-und-deren-behebung
  - totemomail-m365
  - ghost-sender-exchange-online-nebeneingang
slug: "migrazione-exo-senza-remote-move"
translationId: "article-8f3c1b7a62d94e50"
draft: false
translationOf: exchange-mailboxen-ohne-remote-move-cloud-neu-erstellen
translationSourceHash: dc64d2c419e3ac0f4dd730785b3cd7f37c3f23effd2317feb4d61a46fa33401a
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T10:20:30.944Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/migrazione-exo-senza-remote-move
---

# Migrazione EXO senza Remote Move

Un Hybrid Remote Move è il metodo normale per spostare una mailbox Exchange locale, inclusi i contenuti, in Exchange Online. Non tutte le organizzazioni consentono questo percorso di migrazione. Se una policy di sicurezza esclude i Remote Move, può essere giustificato un approccio deliberatamente diverso: la mailbox locale viene sottoposta a backup come PST, separata dall'utente AD sincronizzato e viene eseguito il provisioning di una nuova mailbox vuota in Exchange Online per lo stesso utente.

Questa procedura **non è una migrazione di mailbox**. Non trasferisce messaggi, calendari, regole o autorizzazioni nel cloud. Il PST funge esclusivamente da backup e in questo scenario non viene importato. Il processo è pertanto adatto solo se una mailbox di destinazione vuota è accettabile dal punto di vista aziendale e la perdita della configurazione attiva della mailbox è esplicitamente approvata.

## Stato di destinazione e requisiti imprescindibili

Dopo il cutover, lo stesso utente AD continua a esistere. Tuttavia, localmente non viene più gestito come `UserMailbox`, `SharedMailbox`, `RoomMailbox` o `EquipmentMailbox`, bensì come `RemoteMailbox`. Dopo la sincronizzazione, questo oggetto rappresenta la nuova mailbox in Exchange Online.

Lo stato desiderato è il seguente:

1. È stato eseguito un backup completo della mailbox locale in formato PST.
2. La mailbox locale è disconnessa, ma non ancora eliminata definitivamente entro il periodo di conservazione configurato.
3. L'utente AD esistente è abilitato come RemoteMailbox.
4. L'indirizzo primario, gli alias e il vecchio `LegacyExchangeDN` vengono mantenuti.
5. Entra Connect ha sincronizzato le modifiche.
6. Per le mailbox utente è assegnato un piano di servizio Exchange Online.
7. Exchange Online mostra una vera mailbox cloud e il flusso di posta termina lì.

Prima dell'avvio devono inoltre essere chiariti i seguenti aspetti:

- La condivisione PST è raggiungibile tramite UNC. Il gruppo `Exchange Trusted Subsystem` dispone di autorizzazioni di lettura e scrittura.
- L'account esecutore dispone del ruolo di gestione `Mailbox Import Export`.
- Il PST è esclusivamente il backup concordato; non è previsto alcun import successivo.
- Conservazione, Litigation Hold, eDiscovery e requisiti normativi sono verificati separatamente.
- Deleghe, Send-As, Send-on-Behalf, inoltri, regole della posta in arrivo, dispositivi mobili e accessi applicativi sono inventariati.
- Durante l'esportazione e la commutazione, i messaggi in entrata vengono trattenuti in modo controllato al gateway a monte. Utenti e applicazioni non devono più scrivere nella mailbox di origine.
- La conservazione del database delle mailbox locali copre la finestra di rollback.

## Perché un elenco di approvazione CSV è indispensabile

Una pipeline diretta come `Get-Mailbox | Disable-Mailbox` è troppo rischiosa per questa procedura. Potrebbe includere anche mailbox di sistema, Discovery o mailbox non altrimenti approvate. Il processo seguente utilizza pertanto due approvazioni esplicite:

- `Action=CUTOVER` determina quale riga può effettivamente essere convertita.
- `PstVerified=YES` conferma che il file di esportazione è stato verificato dal punto di vista tecnico e organizzativo.

Inizialmente viene creato solo l'inventario:

```powershell
$CsvPath = "C:\Migration\mailboxes.csv"
$RemoteRoutingDomain = "contoso.mail.onmicrosoft.com"

Get-Mailbox -ResultSize Unlimited -RecipientTypeDetails `
    UserMailbox,SharedMailbox,RoomMailbox,EquipmentMailbox |
    Sort-Object PrimarySmtpAddress |
    ForEach-Object {
        [pscustomobject]@{
            Identity             = $_.Identity
            DisplayName          = $_.DisplayName
            PrimarySmtpAddress   = $_.PrimarySmtpAddress.ToString()
            Alias                = $_.Alias
            SourceType           = $_.RecipientTypeDetails.ToString()
            ArchiveStatus        = $_.ArchiveStatus.ToString()
            ServerName           = $_.ServerName
            Database             = $_.Database.ToString()
            RemoteRoutingAddress = "$($_.Alias)@$RemoteRoutingDomain"
            Action               = "REVIEW"
            PstVerified          = "NO"
        }
    } |
    Export-Csv -Path $CsvPath -NoTypeInformation -Encoding UTF8
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `Get-Mailbox -ResultSize Unlimited` | Rimuove il limite predefinito di 1000 risultati; senza questo parametro, negli ambienti di grandi dimensioni mancano mailbox nell'inventario |
| `-RecipientTypeDetails UserMailbox,SharedMailbox,RoomMailbox,EquipmentMailbox` | Limita la query ai quattro tipi di mailbox da convertire; le mailbox di sistema e Discovery restano escluse |
| `Sort-Object PrimarySmtpAddress` | Ordina l'output in base all'indirizzo SMTP primario, in modo che il file CSV rimanga ordinato in modo stabile durante la revisione aziendale |
| `Export-Csv -Path` | Percorso di destinazione del file CSV |
| `-NoTypeInformation` | Sopprime la riga di intestazione del tipo `#TYPE ...`, che le versioni PowerShell meno recenti altrimenti scrivono come prima riga |
| `-Encoding UTF8` | Scrive il file con codifica UTF-8, affinché gli umlaut nei nomi visualizzati vengano mantenuti correttamente |

</details>

Il file viene quindi ripulito dal punto di vista aziendale. Solo alle mailbox effettivamente approvate viene assegnato `Action=CUTOVER`. Le mailbox di sistema e gli oggetti speciali non devono comparire in questo elenco.

## Fase 1: eseguire il backup della mailbox primaria e dell'archivio come PST

`New-MailboxExportRequest` scrive solo su un percorso UNC. Per ogni mailbox viene generato un nome file univoco. Un archivio online attivo dell'Exchange locale viene esportato separatamente:

```powershell
$CsvPath = "C:\Migration\mailboxes.csv"
$PstShare = "\\fileserver\exchange-pst$"
$BatchName = "EXO-NewMailbox-20260807"

$targets = Import-Csv $CsvPath | Where-Object Action -eq "CUTOVER"

foreach ($row in $targets) {
    $safeName = ($row.PrimarySmtpAddress -replace '[^a-zA-Z0-9@._-]', '_')
    $primaryPath = "$PstShare\$safeName.pst"

    New-MailboxExportRequest `
        -Mailbox $row.Identity `
        -FilePath $primaryPath `
        -Name "Primary-$safeName" `
        -BatchName $BatchName

    if ($row.ArchiveStatus -eq "Active") {
        New-MailboxExportRequest `
            -Mailbox $row.Identity `
            -IsArchive `
            -FilePath "$PstShare\$safeName-archive.pst" `
            -Name "Archive-$safeName" `
            -BatchName $BatchName
    }
}
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `Import-Csv $CsvPath` | Legge l'elenco di approvazione; ogni riga diventa un oggetto con le colonne CSV come proprietà |
| `Where-Object Action -eq "CUTOVER"` | Elabora solo le righe esplicitamente approvate |
| `New-MailboxExportRequest -Mailbox` | Mailbox di origine dell'esportazione, qui l'identità della riga CSV |
| `-FilePath` | Percorso di destinazione del file PST; deve essere un percorso UNC, poiché il cmdlet rifiuta i percorsi locali |
| `-Name` | Nome univoco della richiesta; consente in seguito l'associazione mirata dell'esportazione primaria e di quella dell'archivio |
| `-BatchName` | Raggruppa tutte le richieste di un'esecuzione sotto un nome batch; base per le richieste di stato e la pulizia |
| `-IsArchive` | Esporta l'archivio online anziché la mailbox primaria; per questo viene creata una seconda richiesta per ogni mailbox con archivio attivo |

</details>

L'esportazione viene approvata solo quando **ogni** richiesta ha lo stato `Completed`:

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Get-MailboxExportRequestStatistics -IncludeReport |
    Format-Table DisplayName,Status,PercentComplete,FailureCode,Message -AutoSize

Get-MailboxExportRequest -BatchName $BatchName |
    Where-Object Status -ne "Completed"
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `Get-MailboxExportRequest -BatchName` | Elenca tutte le richieste di esportazione del batch specificato |
| `Get-MailboxExportRequestStatistics -IncludeReport` | Integra le statistiche con il rapporto dettagliato della cronologia, che contiene le cause degli errori delle singole richieste |
| `Format-Table ... -AutoSize` | Visualizzazione tabellare delle proprietà indicate; `-AutoSize` adatta la larghezza delle colonne al contenuto |
| `Where-Object Status -ne "Completed"` | Filtra tutte le richieste non ancora completate o non riuscite; l'output deve essere vuoto prima di procedere |

</details>

Devono inoltre essere verificati esistenza, dimensione, leggibilità, presa in carico dal backup e protezione degli accessi ai file. Solo dopo viene impostato `PstVerified=YES` per la relativa riga CSV.

## Fase 2: salvare i dati della mailbox e gli attributi Exchange

Prima della prima modifica viene creato uno snapshot leggibile da una macchina per ogni mailbox. È più importante di uno screenshot, poiché alias, GUID e il `LegacyExchangeDN` possono essere ricostruiti esattamente in seguito:

```powershell
$SnapshotPath = "C:\Migration\Snapshots"
New-Item -ItemType Directory -Path $SnapshotPath -Force | Out-Null

Import-Csv "C:\Migration\mailboxes.csv" |
    Where-Object { $_.Action -eq "CUTOVER" -and $_.PstVerified -eq "YES" } |
    ForEach-Object {
        $mailbox = Get-Mailbox -Identity $_.Identity
        $safeName = ($_.PrimarySmtpAddress -replace '[^a-zA-Z0-9@._-]', '_')

        $mailbox |
            Select-Object Identity,DistinguishedName,ExchangeGuid,ArchiveGuid,
                RecipientTypeDetails,PrimarySmtpAddress,EmailAddresses,
                LegacyExchangeDN,Alias,Database,ServerName |
            Export-Clixml "$SnapshotPath\$safeName.xml"
    }
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `New-Item -ItemType Directory -Path ... -Force` | Crea la directory degli snapshot; `-Force` sopprime l'errore se esiste già |
| `Get-Mailbox -Identity` | Recupera l'oggetto mailbox corrente per la rispettiva riga CSV |
| `Select-Object Identity,...,ServerName` | Riduce l'oggetto agli attributi necessari per una ricostruzione successiva (GUID, indirizzi, `LegacyExchangeDN`, database) |
| `Export-Clixml` | Serializza l'oggetto come CLIXML con conservazione dei tipi; a differenza del CSV, i valori multipli quali `EmailAddresses` vengono mantenuti integralmente e sono nuovamente leggibili tramite `Import-Clixml` |

</details>

Deleghe e inoltri richiedono esportazioni separate. Almeno le seguenti informazioni dovrebbero essere salvate separatamente:

```powershell
$mailbox = Get-Mailbox -Identity user01@contoso.com

$mailbox | Format-List ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward
Get-MailboxPermission -Identity $mailbox.Identity
Get-ADPermission -Identity $mailbox.DistinguishedName
Get-InboxRule -Mailbox $mailbox.Identity
Get-CalendarProcessing -Identity $mailbox.Identity -ErrorAction SilentlyContinue
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `Format-List ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward` | Mostra i tre attributi di inoltro della mailbox in formato elenco |
| `Get-MailboxPermission -Identity` | Elenca le autorizzazioni della mailbox, come Full Access |
| `Get-ADPermission -Identity` | Elenca le autorizzazioni AD sull'oggetto utente, tra cui Send-As; richiede qui il Distinguished Name |
| `Get-InboxRule -Mailbox` | Elenca le regole lato server della posta in arrivo della mailbox |
| `Get-CalendarProcessing -Identity` | Mostra la configurazione delle prenotazioni; rilevante per le mailbox di sala e dispositivo |
| `-ErrorAction SilentlyContinue` | Sopprime l'errore per i tipi di mailbox privi di configurazione delle prenotazioni, affinché il backup non si interrompa |

</details>

Queste configurazioni non vengono trasferite automaticamente alla nuova mailbox cloud.

## Fase 3: disconnettere la mailbox locale e abilitare RemoteMailbox

Il cutover effettivo è breve, ma ha conseguenze rilevanti. `Disable-Mailbox` rimuove gli attributi Exchange dall'utente AD e disconnette la mailbox locale. I dati della mailbox restano disponibili come disconnected mailbox fino alla scadenza della conservazione del database. Subito dopo, `Enable-RemoteMailbox` abilita lo stesso utente AD per Exchange Online.

Lo script seguente elabora esclusivamente righe approvate due volte. Mantiene l'indirizzo SMTP primario, tutti gli indirizzi proxy esistenti e il vecchio `LegacyExchangeDN` come indirizzo X500. La voce X500 evita NDR nelle risposte a messaggi meno recenti o con voci di completamento automatico di Outlook obsolete.

```powershell
$CsvPath = "C:\Migration\mailboxes.csv"
$PstShare = "\\fileserver\exchange-pst$"

$targets = Import-Csv $CsvPath |
    Where-Object { $_.Action -eq "CUTOVER" -and $_.PstVerified -eq "YES" }

foreach ($row in $targets) {
    $safeName = ($row.PrimarySmtpAddress -replace '[^a-zA-Z0-9@._-]', '_')
    $pstPath = "$PstShare\$safeName.pst"

    if (-not (Test-Path $pstPath)) {
        throw "PST fehlt: $pstPath"
    }

    if ((Get-Item $pstPath).Length -eq 0) {
        throw "PST ist leer: $pstPath"
    }

    $mailbox = Get-Mailbox -Identity $row.Identity -ErrorAction Stop
    $primary = $mailbox.PrimarySmtpAddress.ToString()
    $legacyDn = $mailbox.LegacyExchangeDN

    if ($primary -ine $row.PrimarySmtpAddress) {
        throw "Primaere SMTP-Adresse stimmt nicht mit der Freigabeliste ueberein: $($row.Identity)"
    }

    $preservedAddresses = @($mailbox.EmailAddresses | ForEach-Object ToString)

    Disable-Mailbox -Identity $row.Identity -Confirm:$false

    $enableParams = @{
        Identity             = $row.Identity
        Alias                = $row.Alias
        PrimarySmtpAddress   = $primary
        RemoteRoutingAddress = $row.RemoteRoutingAddress
        ACLableSyncedObjectEnabled = $true
    }

    switch ($row.SourceType) {
        "SharedMailbox"    { $enableParams.Shared = $true }
        "RoomMailbox"      { $enableParams.Room = $true }
        "EquipmentMailbox" { $enableParams.Equipment = $true }
        "UserMailbox"      { }
        default             { throw "Nicht unterstuetzter Mailbox-Typ: $($row.SourceType)" }
    }

    Enable-RemoteMailbox @enableParams

    $orderedAddresses = @(
        "SMTP:$primary"
        $preservedAddresses |
            Where-Object { $_ -notmatch '^SMTP:' -or $_.Substring(5) -ine $primary }
        "smtp:$($row.RemoteRoutingAddress)"
        "X500:$legacyDn"
    )

    $seen = [System.Collections.Generic.HashSet[string]]::new(
        [System.StringComparer]::OrdinalIgnoreCase
    )
    $finalAddresses = foreach ($address in $orderedAddresses) {
        if ($seen.Add($address)) { $address }
    }

    Set-RemoteMailbox -Identity $row.Identity -EmailAddressPolicyEnabled $false
    Set-RemoteMailbox -Identity $row.Identity -EmailAddresses $finalAddresses
    Set-RemoteMailbox -Identity $row.Identity -PrimarySmtpAddress $primary

    Get-RemoteMailbox -Identity $row.Identity |
        Format-List DisplayName,RecipientTypeDetails,PrimarySmtpAddress,
            RemoteRoutingAddress,EmailAddresses
}
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `Get-Mailbox -Identity ... -ErrorAction Stop` | Recupera la mailbox di origine; `-ErrorAction Stop` trasforma un errore di ricerca in un errore che interrompe l'esecuzione, anziché proseguire silenziosamente |
| `Disable-Mailbox -Identity` | Rimuove gli attributi Exchange dall'utente AD e disconnette la mailbox locale; i dati restano nel database come disconnected mailbox |
| `-Confirm:$false` | Sopprime la richiesta di conferma interattiva; l'approvazione avviene qui tramite l'elenco CSV, non al prompt |
| `Enable-RemoteMailbox -Identity` | Abilita lo stesso utente AD come RemoteMailbox per Exchange Online |
| `-Alias` | Reimposta l'alias Exchange al valore dell'elenco di approvazione |
| `-PrimarySmtpAddress` | Mantiene l'indirizzo SMTP primario precedente |
| `-RemoteRoutingAddress` | Indirizzo di destinazione nel dominio di routing `mail.onmicrosoft.com`, attraverso il quale Exchange locale raggiunge la mailbox cloud |
| `-ACLableSyncedObjectEnabled` | Contrassegna l'oggetto come compatibile con ACL, affinché autorizzazioni quali Full Access rimangano valutabili in Exchange Online dopo la sincronizzazione |
| `-Shared` / `-Room` / `-Equipment` | Crea il rispettivo tipo speciale anziché una mailbox utente; lo script imposta esattamente un'opzione corrispondente al tipo di origine |
| `Set-RemoteMailbox -EmailAddressPolicyEnabled $false` | Esclude l'oggetto dalla policy degli indirizzi e-mail, affinché questa non sovrascriva gli indirizzi impostati manualmente |
| `-EmailAddresses` | Imposta l'elenco completo di indirizzi deduplicato, inclusi i vecchi indirizzi proxy, l'indirizzo di routing e la voce X500 |
| `Get-RemoteMailbox -Identity` | Query di controllo del risultato subito dopo la conversione |

</details>

Lo script non è intenzionalmente uno strumento di migrazione completamente automatizzato. Interrompe l'esecuzione alla prima incongruenza, affinché un amministratore possa valutare causa e stato. Prima di un batch di produzione, il codice dovrebbe essere convalidato con poche mailbox di test e con le versioni Exchange in uso.

## Fase 4: sincronizzare, assegnare licenze e verificare

Dopo la modifica locale viene avviato un ciclo delta sul server Entra Connect:

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-PolicyType Delta` | Sincronizza solo gli oggetti modificati dall'ultimo ciclo; l'alternativa `Initial` eseguirebbe un ciclo completo, notevolmente più lungo |

</details>

Per le mailbox utente deve quindi essere assegnato un piano di servizio Exchange Online valido, ad esempio tramite l'assegnazione di licenze basata su gruppi. Le mailbox condivise, di sala e di dispositivo devono essere valutate secondo le attuali condizioni di licenza Microsoft e le funzionalità necessarie.

Il provisioning è asincrono. Microsoft indica per le modifiche normali di solito meno di 30 minuti, ma in singoli casi fino a 24 ore. Durante questo periodo, il flusso di posta a monte dovrebbe trattenere i messaggi in modo controllato invece di recapitarli a una destinazione non ancora pronta.

Il controllo locale deve ora mostrare una RemoteMailbox:

```powershell
Get-RemoteMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,RemoteRoutingAddress,EmailAddresses

Get-Mailbox -Identity user01@contoso.com -ErrorAction SilentlyContinue
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `Get-RemoteMailbox -Identity` | Deve restituire l'oggetto convertito come RemoteMailbox |
| `Format-List RecipientTypeDetails,...` | Mostra tipo, indirizzi e indirizzo di routing in formato elenco per il controllo |
| `Get-Mailbox -Identity ... -ErrorAction SilentlyContinue` | Controverifica: il comando non deve più restituire nulla, poiché localmente non esiste più una mailbox connessa; `-ErrorAction SilentlyContinue` sopprime il messaggio di errore previsto |

</details>

In Exchange Online viene verificato se il precedente MailUser è diventato una vera mailbox:

```powershell
Get-EXORecipient -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,EmailAddresses

Get-EXOMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,ExchangeGuid
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `Get-EXORecipient -Identity` | Mostra il tipo di destinatario in Exchange Online; è previsto `UserMailbox` o il tipo speciale, non più `MailUser` |
| `Get-EXOMailbox -Identity` | Restituisce solo vere mailbox cloud; un risultato dimostra il completamento del provisioning |
| `Format-List ...,ExchangeGuid` | Elenca gli attributi di controllo; il `ExchangeGuid` identifica univocamente la nuova mailbox cloud |

</details>

Il batch è considerato completato solo se hanno esito positivo anche i seguenti test:

- Recapito dall'esterno e dall'interno
- Invio verso l'esterno e l'interno
- Risposta a un vecchio messaggio, per verificare l'indirizzo X500
- Accesso con Outlook e Outlook sul Web
- Deleghe e Send-As
- Inoltri e regole di trasporto
- Prenotazioni di sale e dispositivi
- Applicazioni, scanner e relay SMTP
- Message Trace con recapito alla nuova mailbox cloud

## Rollback e pulizia

La mailbox di origine locale non deve essere eliminata con `Remove-StoreMailbox` durante la fase di convalida. Finché è presente entro il periodo di conservazione della mailbox come disconnected mailbox, esiste ancora una possibilità tecnica di ripristino. Tuttavia, un rollback richiede l'inversione controllata degli attributi RemoteMailbox e la riconnessione della mailbox locale; allo stesso tempo occorre impedire la presenza di due destinazioni di recapito attive.

Prima di un rollback è pertanto necessario salvaguardare il flusso di posta, lo stato della sincronizzazione e i messaggi già ricevuti nel cloud. Il ritorno non è un semplice comando su una riga e deve far parte del change come runbook testato.

Dopo l'accettazione positiva, le richieste di esportazione vengono rimosse, i file PST archiviati secondo il concetto di protezione e conservazione e le autorizzazioni temporanee sulla condivisione di esportazione eliminate:

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Remove-MailboxExportRequest -Confirm:$false
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `Get-MailboxExportRequest -BatchName` | Seleziona esattamente le richieste di esportazione del batch completato |
| `Remove-MailboxExportRequest -Confirm:$false` | Rimuove le richieste senza conferma; i file PST stessi non ne vengono influenzati |

</details>

Le disconnected mailboxes dovrebbero essere eliminate definitivamente solo dopo la scadenza della finestra di rollback concordata e secondo il concetto di conservazione.

## Conclusione

Se gli Hybrid Remote Move non sono consentiti e non è necessario trasferire dati delle mailbox in Exchange Online, un utente AD sincronizzato esistente può essere convertito in modo controllato da una mailbox locale a una nuova mailbox cloud. La parte critica non è `Enable-RemoteMailbox`, ma il controllo del processo che lo circonda: inventario completo, backup PST verificato, approvazioni esplicite, mantenimento degli indirizzi proxy e X500, flusso di posta controllato e una reale finestra di rollback.

## Fonti

1. [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchange/enable-remotemailbox?view=exchange-ps): abilita un utente AD locale esistente per una mailbox nel servizio basato sul cloud e documenta le opzioni per mailbox utente, condivise, di sala e di dispositivo.

2. [Microsoft Learn – New-MailboxExportRequest](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-mailboxexportrequest?view=exchange-ps): riferimento per l'esportazione di mailbox locali primarie e archiviate in file PST.

3. [Microsoft Learn – Mailbox import and export requests](https://learn.microsoft.com/en-us/exchange/mailbox-import-and-export-requests-exchange-2013-help): prerequisiti per la condivisione di esportazione, le autorizzazioni e Exchange Trusted Subsystem.

4. [Microsoft Learn – Disable or delete a mailbox in Exchange Server](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disable-or-delete-mailboxes): comportamento di `Disable-Mailbox`, rimozione degli attributi Exchange e conservazione della mailbox disconnessa.

5. [Microsoft Learn – Disconnected mailboxes](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disconnected-mailboxes): riconnessione, ripristino ed eliminazione definitiva delle mailbox disconnesse.

6. [Microsoft Learn – Delays in provisioning of a user or mailbox](https://learn.microsoft.com/en-us/troubleshoot/exchange/user-and-shared-mailboxes/delays-provision-mailbox-sync-changes): durata tipica e risoluzione dei problemi nel provisioning in Exchange Online.

7. [Microsoft Learn – Move mailboxes between on-premises and Exchange Online organizations](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes): il normale Hybrid Remote Move come riferimento e delimitazione rispetto alla nuova configurazione descritta qui.
