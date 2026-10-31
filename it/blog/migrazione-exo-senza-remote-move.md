---
title: "Migrazione EXO senza Remote Move"
navTitle: "Migrazione EXO senza Remote Move"
description: "Come effettuare in modo controllato il provisioning di mailbox Exchange locali come nuove mailbox vuote in Exchange Online: backup PST, approvazione CSV, RemoteMailbox, sincronizzazione, convalida e rollback."
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
translationSourceHash: 8ef7108bef25cbfdabbdd58d6ed17a1cc34ba2da65eb7e685180b0976f99d62c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:33:12.994Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/migrazione-exo-senza-remote-move
---

# Migrazione EXO senza Remote Move

Un Hybrid Remote Move è il metodo normale per spostare una mailbox Exchange locale, inclusi i contenuti, in Exchange Online. Non tutte le organizzazioni consentono questo percorso di migrazione. Se una policy di sicurezza esclude i Remote Move, può essere giustificato un approccio deliberatamente diverso: la mailbox locale viene salvata come PST, separata dall'utente AD sincronizzato e per lo stesso utente viene effettuato il provisioning di una nuova mailbox vuota in Exchange Online.

Questo approccio **non è una migrazione di mailbox**. Non trasferisce messaggi, calendari, regole né autorizzazioni nel cloud. Il PST serve esclusivamente come backup e, in questo scenario, non viene importato. Questa procedura è quindi adatta solo se una mailbox di destinazione vuota è accettabile dal punto di vista operativo e la perdita della configurazione attiva della mailbox è espressamente approvata.

## Stato di destinazione e prerequisiti imprescindibili

Dopo il cutover, lo stesso utente AD continua a esistere. Tuttavia, localmente non viene più gestito come `UserMailbox`, `SharedMailbox`, `RoomMailbox` o `EquipmentMailbox`, bensì come `RemoteMailbox`. Dopo la sincronizzazione, questo oggetto rappresenta la nuova mailbox in Exchange Online.

Lo stato desiderato è il seguente:

1. La mailbox locale è completamente salvata come PST.
2. La mailbox locale è disconnessa, ma non ancora eliminata definitivamente entro il periodo di retention configurato.
3. L'utente AD esistente è abilitato come RemoteMailbox.
4. Indirizzo primario, alias e il precedente `LegacyExchangeDN` vengono mantenuti.
5. Entra Connect ha sincronizzato le modifiche.
6. Per le mailbox utente è assegnato un piano di servizio Exchange Online.
7. Exchange Online mostra una vera mailbox cloud e il flusso di posta termina lì.

Prima di iniziare, devono inoltre essere chiariti i seguenti aspetti:

- La condivisione PST è raggiungibile tramite UNC. Il gruppo `Exchange Trusted Subsystem` dispone di autorizzazioni di lettura e scrittura.
- L'account esecutore dispone del ruolo di gestione `Mailbox Import Export`.
- Il PST è esclusivamente il backup concordato; non è previsto alcun import successivo.
- Retention, Litigation Hold, eDiscovery e requisiti normativi sono verificati separatamente.
- Deleghe, Send-As, Send-on-Behalf, inoltri, regole Posta in arrivo, dispositivi mobili e accessi applicativi sono inventariati.
- Durante l'esportazione e la commutazione, i messaggi in arrivo vengono trattenuti in modo controllato presso il gateway a monte. Utenti e applicazioni non devono più scrivere nella mailbox di origine.
- La retention del database mailbox locale copre la finestra di rollback.

## Perché un elenco di approvazione CSV è indispensabile

Una pipeline diretta come `Get-Mailbox | Disable-Mailbox` è troppo rischiosa per questa operazione. Potrebbe includere anche mailbox di sistema, di Discovery o altre mailbox non approvate. La procedura seguente lavora quindi con due approvazioni esplicite:

- `Action=CUTOVER` stabilisce quale riga può essere effettivamente convertita.
- `PstVerified=YES` conferma che il file di esportazione è stato verificato dal punto di vista tecnico e organizzativo.

Per prima cosa viene creato solo l'inventario:

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
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Get-Mailbox -ResultSize Unlimited` | Rimuove il limite predefinito di 1000 risultati; senza questo parametro, negli ambienti grandi alcune mailbox non compaiono nell'inventario |
| `-RecipientTypeDetails UserMailbox,SharedMailbox,RoomMailbox,EquipmentMailbox` | Limita la query ai quattro tipi di mailbox da convertire; le mailbox di sistema e Discovery restano escluse |
| `Sort-Object PrimarySmtpAddress` | Ordina l'output in base all'indirizzo SMTP primario, affinché il file CSV sia ordinato in modo stabile durante la revisione operativa |
| `Export-Csv -Path` | Percorso di destinazione del file CSV |
| `-NoTypeInformation` | Sopprime l'intestazione del tipo `#TYPE ...`, che le versioni meno recenti di PowerShell altrimenti scrivono come prima riga |
| `-Encoding UTF8` | Scrive il file con codifica UTF-8, affinché gli umlaut nei nomi visualizzati siano mantenuti correttamente |

</details>

Il file viene quindi ripulito dal punto di vista operativo. Solo le mailbox effettivamente approvate ricevono `Action=CUTOVER`. Le mailbox di sistema e gli oggetti speciali non devono comparire in questo elenco.

## Fase 1: salvare la mailbox primaria e l'archivio come PST

`New-MailboxExportRequest` scrive solo in un percorso UNC. Per ogni mailbox viene generato un nome file univoco. Un archivio online attivo dell'Exchange locale viene esportato separatamente:

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
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Import-Csv $CsvPath` | Legge l'elenco di approvazione; ogni riga diventa un oggetto con le colonne CSV come proprietà |
| `Where-Object Action -eq "CUTOVER"` | Elabora solo le righe esplicitamente approvate |
| `New-MailboxExportRequest -Mailbox` | Mailbox di origine dell'esportazione, qui l'identità dalla riga CSV |
| `-FilePath` | Percorso di destinazione del file PST; deve essere un percorso UNC, il cmdlet rifiuta i percorsi locali |
| `-Name` | Nome univoco della richiesta; consente in seguito l'associazione mirata dell'esportazione primaria e dell'archivio |
| `-BatchName` | Raggruppa tutte le richieste di un'esecuzione sotto un nome batch; base per la query dello stato e la pulizia |
| `-IsArchive` | Esporta l'archivio online anziché la mailbox primaria; per questo è necessaria una seconda richiesta per ogni mailbox con archivio attivo |

</details>

L'esportazione è approvata solo quando **ogni** richiesta ha lo stato `Completed`:

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Get-MailboxExportRequestStatistics -IncludeReport |
    Format-Table DisplayName,Status,PercentComplete,FailureCode,Message -AutoSize

Get-MailboxExportRequest -BatchName $BatchName |
    Where-Object Status -ne "Completed"
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Get-MailboxExportRequest -BatchName` | Elenca tutte le richieste di esportazione del batch specificato |
| `Get-MailboxExportRequestStatistics -IncludeReport` | Integra le statistiche con il report cronologico dettagliato, che contiene le cause degli errori delle singole richieste |
| `Format-Table ... -AutoSize` | Visualizzazione tabellare delle proprietà indicate; `-AutoSize` adatta la larghezza delle colonne al contenuto |
| `Where-Object Status -ne "Completed"` | Filtra tutte le richieste non ancora completate o non riuscite; l'output deve essere vuoto prima di proseguire |

</details>

Vanno inoltre verificati esistenza, dimensione, leggibilità, acquisizione nel backup e protezione degli accessi ai file. Solo dopo viene impostato `PstVerified=YES` per la corrispondente riga CSV.

## Fase 2: salvare i dati della mailbox e gli attributi Exchange

Prima della prima modifica viene creato per ogni mailbox uno snapshot leggibile automaticamente. È più importante di uno screenshot, poiché alias, GUID e il `LegacyExchangeDN` possono essere successivamente ricostruiti con precisione:

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
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `New-Item -ItemType Directory -Path ... -Force` | Crea la directory dello snapshot; `-Force` sopprime l'errore qualora esista già |
| `Get-Mailbox -Identity` | Recupera l'oggetto mailbox corrente per la rispettiva riga CSV |
| `Select-Object Identity,...,ServerName` | Riduce l'oggetto agli attributi necessari per una successiva ricostruzione: GUID, indirizzi, `LegacyExchangeDN`, database |
| `Export-Clixml` | Serializza l'oggetto come CLIXML con mantenimento dei tipi; a differenza del CSV, i valori multipli quali `EmailAddresses` vengono mantenuti integralmente e possono essere riletti tramite `Import-Clixml` |

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
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Format-List ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward` | Mostra i tre attributi di inoltro della mailbox in visualizzazione elenco |
| `Get-MailboxPermission -Identity` | Elenca le autorizzazioni della mailbox, come Full Access |
| `Get-ADPermission -Identity` | Elenca le autorizzazioni AD sull'oggetto utente, incluso Send-As; qui richiede il Distinguished Name |
| `Get-InboxRule -Mailbox` | Elenca le regole Posta in arrivo lato server della mailbox |
| `Get-CalendarProcessing -Identity` | Mostra la configurazione delle prenotazioni; rilevante per mailbox di sala e apparecchiature |
| `-ErrorAction SilentlyContinue` | Sopprime l'errore per i tipi di mailbox privi di configurazione delle prenotazioni, affinché il backup non si interrompa |

</details>

Queste configurazioni non vengono trasferite automaticamente alla nuova mailbox cloud.

## Fase 3: disconnettere la mailbox locale e abilitare RemoteMailbox

Il cutover effettivo è breve, ma ha conseguenze importanti. `Disable-Mailbox` rimuove gli attributi Exchange dall'utente AD e disconnette la mailbox locale. I dati della mailbox rimangono disponibili come mailbox disconnessa fino alla scadenza della retention del database. Subito dopo, `Enable-RemoteMailbox` abilita lo stesso utente AD per Exchange Online.

Lo script seguente elabora esclusivamente le righe approvate due volte. Mantiene l'indirizzo SMTP primario, tutti gli indirizzi proxy esistenti e il precedente `LegacyExchangeDN` come indirizzo X500. La voce X500 evita gli NDR quando si risponde a messaggi precedenti o si utilizzano vecchie voci di completamento automatico di Outlook.

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
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Get-Mailbox -Identity ... -ErrorAction Stop` | Recupera la mailbox di origine; `-ErrorAction Stop` rende un errore di lookup un errore bloccante, anziché proseguire silenziosamente |
| `Disable-Mailbox -Identity` | Rimuove gli attributi Exchange dall'utente AD e disconnette la mailbox locale; i dati restano come mailbox disconnessa nel database |
| `-Confirm:$false` | Sopprime la richiesta interattiva; l'approvazione avviene qui tramite l'elenco CSV, non al prompt |
| `Enable-RemoteMailbox -Identity` | Abilita lo stesso utente AD come RemoteMailbox per Exchange Online |
| `-Alias` | Reimposta l'alias Exchange al valore dell'elenco di approvazione |
| `-PrimarySmtpAddress` | Mantiene l'indirizzo SMTP primario precedente |
| `-RemoteRoutingAddress` | Indirizzo di destinazione nel dominio di routing `mail.onmicrosoft.com`, tramite il quale l'Exchange locale raggiunge la mailbox cloud |
| `-ACLableSyncedObjectEnabled` | Contrassegna l'oggetto come compatibile con ACL, affinché autorizzazioni come Full Access possano essere valutate in Exchange Online dopo la sincronizzazione |
| `-Shared` / `-Room` / `-Equipment` | Crea il rispettivo tipo speciale anziché una mailbox utente; lo script imposta esattamente un parametro corrispondente al tipo di origine |
| `Set-RemoteMailbox -EmailAddressPolicyEnabled $false` | Esclude l'oggetto dalla policy degli indirizzi e-mail, affinché questa non sovrascriva gli indirizzi impostati manualmente |
| `-EmailAddresses` | Imposta l'elenco completo e deduplicato degli indirizzi, inclusi vecchi indirizzi proxy, indirizzo di routing e voce X500 |
| `Get-RemoteMailbox -Identity` | Query di controllo del risultato subito dopo la conversione |

</details>

Lo script non è intenzionalmente uno strumento di migrazione completamente automatizzato. Interrompe l'esecuzione alla prima incongruenza, affinché un amministratore possa valutare causa e stato. Prima di un batch in produzione, il codice dovrebbe essere convalidato con poche mailbox di test e con le versioni Exchange utilizzate.

## Fase 4: sincronizzare, assegnare licenze e verificare

Dopo la modifica locale, sul server Entra Connect viene avviato un ciclo delta:

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-PolicyType Delta` | Sincronizza solo gli oggetti modificati dall'ultimo ciclo; l'alternativa `Initial` sarebbe un'esecuzione completa e notevolmente più lunga |

</details>

Per le mailbox utente deve essere successivamente assegnato un piano di servizio Exchange Online valido, ad esempio tramite licenze basate su gruppi. Le mailbox condivise, di sala e delle apparecchiature devono essere valutate in base alle attuali condizioni di licenza Microsoft e alle funzionalità richieste.

Il provisioning è asincrono. Microsoft indica per le modifiche normali generalmente meno di 30 minuti, ma in singoli casi fino a 24 ore. Durante questo periodo, il flusso di posta a monte dovrebbe trattenere i messaggi in modo controllato, anziché recapitarli a una destinazione non ancora pronta.

Il controllo locale deve ora mostrare una RemoteMailbox:

```powershell
Get-RemoteMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,RemoteRoutingAddress,EmailAddresses

Get-Mailbox -Identity user01@contoso.com -ErrorAction SilentlyContinue
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Get-RemoteMailbox -Identity` | Deve restituire l'oggetto convertito come RemoteMailbox |
| `Format-List RecipientTypeDetails,...` | Mostra tipo, indirizzi e indirizzo di routing per il controllo in visualizzazione elenco |
| `Get-Mailbox -Identity ... -ErrorAction SilentlyContinue` | Controprova: la chiamata non deve più restituire nulla, poiché localmente non esiste più una mailbox connessa; `-ErrorAction SilentlyContinue` sopprime il messaggio di errore atteso |

</details>

In Exchange Online si verifica se il precedente MailUser è diventato una vera mailbox:

```powershell
Get-EXORecipient -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,EmailAddresses

Get-EXOMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,ExchangeGuid
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Get-EXORecipient -Identity` | Mostra il tipo di destinatario in Exchange Online; è previsto `UserMailbox` oppure il tipo speciale, non più `MailUser` |
| `Get-EXOMailbox -Identity` | Restituisce solo vere mailbox cloud; un risultato attesta il provisioning completato |
| `Format-List ...,ExchangeGuid` | Elenca gli attributi di controllo; `ExchangeGuid` identifica in modo univoco la nuova mailbox cloud |

</details>

Il batch è considerato completato solo quando hanno avuto successo anche i seguenti test:

- Recapito dall'esterno e dall'interno
- Invio verso l'esterno e l'interno
- Risposta a un vecchio messaggio per verificare l'indirizzo X500
- Accesso con Outlook e Outlook sul web
- Deleghe e Send-As
- Inoltri e regole di trasporto
- Prenotazioni di sale e apparecchiature
- Applicazioni, scanner e relay SMTP
- Message Trace con recapito alla nuova mailbox cloud

## Rollback e pulizia

Durante la fase di convalida, la mailbox di origine locale non deve essere eliminata con `Remove-StoreMailbox`. Finché è presente come mailbox disconnessa entro la retention della mailbox, esiste ancora una possibilità tecnica di ripristino. Un rollback richiede tuttavia un'inversione controllata degli attributi RemoteMailbox e la riconnessione della mailbox locale; allo stesso tempo bisogna impedire che esistano due destinazioni di recapito attive.

Prima di un rollback, vanno quindi messi in sicurezza il flusso di posta, lo stato della sincronizzazione e i messaggi già ricevuti nel cloud. Il ripristino non è un semplice comando di una riga e deve far parte del change come runbook testato.

Dopo l'accettazione positiva, le richieste di esportazione vengono pulite, i file PST archiviati secondo il piano di protezione e retention e le autorizzazioni temporanee sulla condivisione di esportazione rimosse:

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Remove-MailboxExportRequest -Confirm:$false
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Get-MailboxExportRequest -BatchName` | Seleziona esattamente le richieste di esportazione del batch completato |
| `Remove-MailboxExportRequest -Confirm:$false` | Rimuove le richieste senza conferma; i file PST stessi restano inalterati |

</details>

Le mailbox disconnesse dovrebbero essere eliminate definitivamente solo dopo la scadenza della finestra di rollback concordata e in conformità al piano di retention.

## Conclusione

Se gli Hybrid Remote Move non sono consentiti e non è necessario trasferire dati delle mailbox in Exchange Online, un utente AD sincronizzato esistente può essere convertito in modo controllato da una mailbox locale a una nuova mailbox cloud. La parte critica non è `Enable-RemoteMailbox`, ma il controllo del processo circostante: inventario completo, backup PST verificato, approvazioni esplicite, mantenimento degli indirizzi proxy e X500, flusso di posta controllato e una reale finestra di rollback.

## Cronologia delle modifiche

- **25 ottobre 2026:** collegamento alle Fonti aggiornato al nuovo indirizzo di Microsoft Learn.

## Fonti

1. [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox?view=exchange-ps): abilita un utente AD locale esistente per una mailbox nel servizio basato sul cloud e documenta i parametri per mailbox utente, condivise, di sala e delle apparecchiature.

2. [Microsoft Learn – New-MailboxExportRequest](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-mailboxexportrequest?view=exchange-ps): riferimento per l'esportazione di mailbox locali primarie e archiviate in file PST.

3. [Microsoft Learn – Mailbox import and export requests](https://learn.microsoft.com/en-us/exchange/mailbox-import-and-export-requests-exchange-2013-help): prerequisiti per la condivisione di esportazione, le autorizzazioni e Exchange Trusted Subsystem.

4. [Microsoft Learn – Disable or delete a mailbox in Exchange Server](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disable-or-delete-mailboxes): comportamento di `Disable-Mailbox`, rimozione degli attributi Exchange e conservazione della mailbox disconnessa.

5. [Microsoft Learn – Disconnected mailboxes](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disconnected-mailboxes): riconnessione, ripristino ed eliminazione definitiva delle mailbox disconnesse.

6. [Microsoft Learn – Delays in provisioning of a user or mailbox](https://learn.microsoft.com/en-us/troubleshoot/exchange/user-and-shared-mailboxes/delays-provision-mailbox-sync-changes): durata tipica e risoluzione dei problemi durante il provisioning in Exchange Online.

7. [Microsoft Learn – Move mailboxes between on-premises and Exchange Online organizations](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes): il normale Hybrid Remote Move come riferimento e distinzione rispetto alla nuova configurazione descritta qui.
