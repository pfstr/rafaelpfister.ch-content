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
url: https://rafaelpfister.ch/it/blog/migrazione-exo-senza-remote-move
translationSourceHash: 861f11b6e2f1e316ca773f049637fa2ac6ed5efdab5ec74d8c28178f3ea7e98c
translationModel: gpt-5.6-terra
translatedAt: 2026-08-08T13:48:07.104Z
translationReview: automatic
---

# Migrazione EXO senza Remote Move

Un Hybrid Remote Move è il modo normale per spostare una mailbox Exchange locale, inclusi i contenuti, in Exchange Online. Non tutte le organizzazioni consentono questo percorso di migrazione. Se una policy di sicurezza esclude i Remote Move, può essere giustificato un approccio deliberatamente diverso: la mailbox locale viene salvata come PST, separata dall'utente AD sincronizzato e viene effettuato il provisioning di una nuova mailbox vuota in Exchange Online per lo stesso utente.

Questo approccio **non è una migrazione di mailbox**. Non trasferisce messaggi, calendari, regole né autorizzazioni nel cloud. Il PST serve esclusivamente come backup e, in questo scenario, non viene importato. Pertanto, il processo è adatto solo se una mailbox di destinazione vuota è accettabile dal punto di vista funzionale e la perdita della configurazione attiva della mailbox è stata espressamente approvata.

## Stato di destinazione e prerequisiti inderogabili

Dopo il cutover, lo stesso utente AD continua a esistere. Tuttavia, localmente non viene più gestito come `UserMailbox`, `SharedMailbox`, `RoomMailbox` o `EquipmentMailbox`, bensì come `RemoteMailbox`. Dopo la sincronizzazione, questo oggetto rappresenta la nuova mailbox in Exchange Online.

Lo stato desiderato è il seguente:

1. La mailbox locale è completamente salvata come PST.
2. La mailbox locale è disconnessa, ma non ancora eliminata definitivamente entro la retention configurata.
3. L'utente AD esistente è abilitato come RemoteMailbox.
4. L'indirizzo principale, gli alias e il vecchio `LegacyExchangeDN` vengono mantenuti.
5. Entra Connect ha sincronizzato le modifiche.
6. Alle mailbox utente è assegnato un piano di servizio Exchange Online.
7. Exchange Online mostra una vera mailbox cloud e il flusso di posta termina lì.

Prima dell'avvio, occorre inoltre chiarire quanto segue:

- La condivisione PST è raggiungibile tramite UNC. Il gruppo `Exchange Trusted Subsystem` dispone di diritti di lettura e scrittura.
- L'account esecutore dispone del ruolo di gestione `Mailbox Import Export`.
- Il PST è soltanto il backup concordato; non è previsto alcun import successivo.
- Conservazione, Litigation Hold, eDiscovery e requisiti normativi sono verificati separatamente.
- Deleghe, Send As, Send on Behalf, inoltri, regole della posta in arrivo, dispositivi mobili e accessi applicativi sono inventariati.
- Durante l'esportazione e la commutazione, i messaggi in arrivo vengono trattenuti in modo controllato presso il gateway a monte. Utenti e applicazioni non devono più scrivere nella mailbox di origine.
- La retention del database delle mailbox locali copre la finestra di rollback.

## Perché un elenco di approvazione CSV è indispensabile

Una pipeline diretta come `Get-Mailbox | Disable-Mailbox` è troppo rischiosa per questa procedura. Potrebbe includere anche mailbox di sistema, di Discovery o non approvate per altri motivi. Il processo seguente lavora quindi con due approvazioni esplicite:

- `Action=CUTOVER` determina quale riga può essere effettivamente commutata.
- `PstVerified=YES` conferma che il file di esportazione è stato verificato dal punto di vista tecnico e organizzativo.

Per prima cosa viene creato soltanto l'inventario:

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

Il file viene quindi corretto dal punto di vista funzionale. Solo le mailbox effettivamente approvate ricevono `Action=CUTOVER`. Le mailbox di sistema e gli oggetti speciali non devono essere inclusi in questo elenco.

## Fase 1: salvare la mailbox primaria e l'archivio come PST

`New-MailboxExportRequest` scrive soltanto in un percorso UNC. Per ogni mailbox viene generato un nome file univoco. Un archivio online attivo dell'Exchange locale viene esportato separatamente:

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

L'esportazione è approvata solo quando **ogni** richiesta ha lo stato `Completed`:

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Get-MailboxExportRequestStatistics -IncludeReport |
    Format-Table DisplayName,Status,PercentComplete,FailureCode,Message -AutoSize

Get-MailboxExportRequest -BatchName $BatchName |
    Where-Object Status -ne "Completed"
```

Inoltre, devono essere verificati esistenza, dimensione, leggibilità, acquisizione nel backup e protezione dell'accesso ai file. Solo dopo viene impostato `PstVerified=YES` per la riga CSV corrispondente.

## Fase 2: salvare dati della mailbox e attributi Exchange

Prima della prima modifica viene creato uno snapshot leggibile da macchina per ogni mailbox. È più importante di uno screenshot, perché alias, GUID e il `LegacyExchangeDN` possono essere ricostruiti esattamente in seguito:

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

Deleghe e inoltri richiedono esportazioni separate. Almeno queste informazioni devono essere salvate separatamente:

```powershell
$mailbox = Get-Mailbox -Identity user01@contoso.com

$mailbox | Format-List ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward
Get-MailboxPermission -Identity $mailbox.Identity
Get-ADPermission -Identity $mailbox.DistinguishedName
Get-InboxRule -Mailbox $mailbox.Identity
Get-CalendarProcessing -Identity $mailbox.Identity -ErrorAction SilentlyContinue
```

Queste configurazioni non vengono trasferite automaticamente alla nuova mailbox cloud.

## Fase 3: disconnettere la mailbox locale e abilitare RemoteMailbox

Il cutover effettivo è breve, ma ha conseguenze rilevanti. `Disable-Mailbox` rimuove gli attributi Exchange dall'utente AD e disconnette la mailbox locale. I dati della mailbox restano disponibili come mailbox disconnessa fino alla scadenza della retention del database. Subito dopo, `Enable-RemoteMailbox` abilita lo stesso utente AD per Exchange Online.

Lo script seguente elabora esclusivamente righe approvate due volte. Conserva l'indirizzo SMTP principale, tutti gli indirizzi proxy esistenti e il vecchio `LegacyExchangeDN` come indirizzo X500. La voce X500 evita gli NDR quando si risponde a messaggi più vecchi o si utilizzano vecchie voci di completamento automatico di Outlook.

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

Lo script non è intenzionalmente un motore di migrazione completamente autonomo. Interrompe l'esecuzione alla prima incongruenza, affinché un amministratore possa valutare la causa e lo stato. Prima di un batch in produzione, il codice dovrebbe essere convalidato con poche mailbox di test e con le versioni di Exchange utilizzate.

## Fase 4: sincronizzare, assegnare licenze e verificare

Dopo la modifica locale viene avviato un ciclo delta sul server Entra Connect:

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

Per le mailbox utente deve quindi essere assegnato un piano di servizio Exchange Online valido, ad esempio tramite l'assegnazione di licenze basata su gruppi. Le mailbox condivise, di sala e di apparecchiature devono essere valutate in base alle attuali condizioni di licenza Microsoft e alle funzionalità richieste.

Il provisioning è asincrono. Microsoft indica per le modifiche normali generalmente meno di 30 minuti, ma in singoli casi fino a 24 ore. Durante questo periodo, il flusso di posta a monte dovrebbe trattenere i messaggi in modo controllato, anziché consegnarli a una destinazione non ancora pronta.

Il controllo locale deve ora mostrare una RemoteMailbox:

```powershell
Get-RemoteMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,RemoteRoutingAddress,EmailAddresses

Get-Mailbox -Identity user01@contoso.com -ErrorAction SilentlyContinue
```

In Exchange Online viene verificato se il precedente MailUser è diventato una vera mailbox:

```powershell
Get-EXORecipient -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,EmailAddresses

Get-EXOMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,ExchangeGuid
```

Il batch è considerato completato solo se anche i seguenti test hanno esito positivo:

- Consegna dall'esterno e dall'interno
- Invio verso l'esterno e l'interno
- Risposta a un vecchio messaggio, per verificare l'indirizzo X500
- Accesso con Outlook e Outlook sul web
- Deleghe e Send As
- Inoltri e regole di trasporto
- Prenotazioni di sale e apparecchiature
- Applicazioni, scanner e relay SMTP
- Message Trace con consegna alla nuova mailbox cloud

## Rollback e pulizia

La mailbox locale di origine non deve essere eliminata con `Remove-StoreMailbox` durante la fase di convalida. Finché è presente come mailbox disconnessa entro la retention della mailbox, esiste ancora una possibilità tecnica di ripristino. Tuttavia, un rollback richiede un'inversione controllata degli attributi RemoteMailbox e la riconnessione della mailbox locale; allo stesso tempo, occorre impedire la presenza di due destinazioni di consegna attive.

Prima di un rollback devono quindi essere salvati il flusso di posta, lo stato di sincronizzazione e i messaggi già ricevuti nel cloud. Il ritorno non è un semplice one-liner e deve far parte del change come runbook testato.

Dopo l'accettazione positiva, le richieste di esportazione vengono pulite, i file PST vengono archiviati secondo il piano di protezione e conservazione e le autorizzazioni temporanee sulla condivisione di esportazione vengono rimosse:

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Remove-MailboxExportRequest -Confirm:$false
```

Le mailbox disconnesse dovrebbero essere eliminate definitivamente solo dopo la scadenza della finestra di rollback concordata e in conformità con il piano di retention.

## Conclusione

Se gli Hybrid Remote Move non sono consentiti e non è necessario trasferire dati della mailbox in Exchange Online, un utente AD sincronizzato esistente può essere commutato in modo controllato da una mailbox locale a una nuova mailbox cloud. La parte critica non è `Enable-RemoteMailbox`, bensì il controllo del processo attorno a essa: inventario completo, backup PST verificato, approvazioni esplicite, mantenimento degli indirizzi proxy e X500, flusso di posta controllato e una vera finestra di rollback.

## Fonti

1. [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchange/enable-remotemailbox?view=exchange-ps): abilita un utente AD locale esistente per una mailbox nel servizio basato sul cloud e documenta gli switch per mailbox utente, condivise, di sala e di apparecchiature.

2. [Microsoft Learn – New-MailboxExportRequest](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-mailboxexportrequest?view=exchange-ps): riferimento per l'esportazione di mailbox locali primarie e archiviate in file PST.

3. [Microsoft Learn – Mailbox import and export requests](https://learn.microsoft.com/en-us/exchange/mailbox-import-and-export-requests-exchange-2013-help): prerequisiti per la condivisione di esportazione, le autorizzazioni e Exchange Trusted Subsystem.

4. [Microsoft Learn – Disable or delete a mailbox in Exchange Server](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disable-or-delete-mailboxes): comportamento di `Disable-Mailbox`, rimozione degli attributi Exchange e conservazione della mailbox disconnessa.

5. [Microsoft Learn – Disconnected mailboxes](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disconnected-mailboxes): riconnessione, ripristino ed eliminazione definitiva delle mailbox disconnesse.

6. [Microsoft Learn – Delays in provisioning of a user or mailbox](https://learn.microsoft.com/en-us/troubleshoot/exchange/user-and-shared-mailboxes/delays-provision-mailbox-sync-changes): durata tipica e risoluzione dei problemi relativi al provisioning in Exchange Online.

7. [Microsoft Learn – Move mailboxes between on-premises and Exchange Online organizations](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes): il normale Hybrid Remote Move come riferimento e distinzione rispetto alla nuova configurazione descritta qui.
