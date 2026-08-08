---
title: "EXO-migrering utan Remote Move"
navTitle: "EXO-migrering utan Remote Move"
description: "Så etableras lokala Exchange-postlådor kontrollerat som nya, tomma Exchange Online-postlådor: PST-säkerhetskopiering, CSV-godkännande, RemoteMailbox, synkronisering, validering och rollback."
date: "2026-08-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "12 min. lästid"
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
slug: "exo-migrering-utan-remote-move"
translationId: "article-8f3c1b7a62d94e50"
draft: false
translationOf: exchange-mailboxen-ohne-remote-move-cloud-neu-erstellen
url: https://rafaelpfister.ch/sv/blog/exo-migrering-utan-remote-move
translationSourceHash: 861f11b6e2f1e316ca773f049637fa2ac6ed5efdab5ec74d8c28178f3ea7e98c
translationModel: gpt-5.6-terra
translatedAt: 2026-08-08T13:48:49.070Z
translationReview: automatic
---

# EXO-migrering utan Remote Move

En Hybrid Remote Move är det normala sättet att flytta en lokal Exchange-postlåda med dess innehåll till Exchange Online. Alla organisationer tillåter inte denna migreringsväg. Om en säkerhetspolicy utesluter Remote Moves kan en medvetet annorlunda metod vara försvarbar: Den lokala postlådan säkerhetskopieras som PST, separeras från den synkroniserade AD-användaren och en ny, tom postlåda etableras i Exchange Online för samma användare.

Denna metod är **inte en postlådemigrering**. Den överför varken meddelanden, kalendrar, regler eller behörigheter till molnet. PST-filen fungerar enbart som säkerhetskopia och importeras inte i detta scenario. Processen är därför endast lämplig när en tom målpostlåda är verksamhetsmässigt acceptabel och förlusten av den aktiva postlådekonfigurationen uttryckligen har godkänts.

## Målläge och hårda förutsättningar

Efter cutover finns samma AD-användare kvar. Lokalt hanteras den dock inte längre som `UserMailbox`, `SharedMailbox`, `RoomMailbox` eller `EquipmentMailbox`, utan som `RemoteMailbox`. Efter synkroniseringen representerar detta objekt den nya postlådan i Exchange Online.

Det önskade läget ser ut så här:

1. Den lokala postlådan är fullständigt säkerhetskopierad som PST.
2. Den lokala postlådan är frånkopplad, men ännu inte slutgiltigt raderad inom den konfigurerade kvarhållningstiden.
3. Den befintliga AD-användaren är aktiverad som RemoteMailbox.
4. Primär adress, alias och den gamla `LegacyExchangeDN` behålls.
5. Entra Connect har synkroniserat ändringarna.
6. För användarpostlådor har en Exchange Online-tjänstplan tilldelats.
7. Exchange Online visar en riktig molnpostlåda och e-postflödet avslutas där.

Följande måste dessutom vara klarlagt före start:

- PST-resursen är tillgänglig via UNC. Gruppen `Exchange Trusted Subsystem` har läs- och skrivbehörighet där.
- Kontot som kör processen har hanteringsrollen `Mailbox Import Export`.
- PST-filen är endast den överenskomna säkerhetskopian; ingen senare import planeras.
- Kvarhållning, Litigation Hold, eDiscovery och regulatoriska krav har granskats separat.
- Delegeringar, Send-As, Send-on-Behalf, vidarebefordringar, Inbox-regler, mobila enheter och programåtkomst har inventerats.
- Under export och omkoppling hålls inkommande meddelanden kontrollerat kvar vid den föregående gatewayen. Användare och program får inte längre skriva till källpostlådan.
- Kvarhållningen för den lokala postlådedatabasen täcker rollback-fönstret.

## Varför en CSV-godkännandelista är oumbärlig

En direkt pipeline som `Get-Mailbox | Disable-Mailbox` är för riskabel för denna process. Den skulle även kunna inkludera system-, Discovery- eller andra ej godkända postlådor. Följande process arbetar därför med två uttryckliga godkännanden:

- `Action=CUTOVER` avgör vilken rad som faktiskt får ställas om.
- `PstVerified=YES` bekräftar att exportfilen har kontrollerats tekniskt och organisatoriskt.

Först skapas endast inventeringen:

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

Filen rensas sedan ur verksamhetsperspektiv. Endast postlådor som faktiskt har godkänts får `Action=CUTOVER`. Systempostlådor och specialobjekt hör inte hemma i denna lista.

## Fas 1: Säkerhetskopiera primär postlåda och arkiv som PST

`New-MailboxExportRequest` skriver endast till en UNC-sökväg. Ett unikt filnamn skapas för varje postlåda. Ett aktivt onlinearkiv i lokal Exchange exporteras separat:

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

Exporten godkänns först när **varje** begäran har statusen `Completed`:

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Get-MailboxExportRequestStatistics -IncludeReport |
    Format-Table DisplayName,Status,PercentComplete,FailureCode,Message -AutoSize

Get-MailboxExportRequest -BatchName $BatchName |
    Where-Object Status -ne "Completed"
```

Dessutom måste filernas existens, storlek, läsbarhet, övertagande i säkerhetskopiering och åtkomstskydd kontrolleras. Först därefter sätts `PstVerified=YES` för motsvarande CSV-rad.

## Fas 2: Säkerhetskopiera postlådedata och Exchange-attribut

Före den första ändringen skapas en maskinläsbar ögonblicksbild för varje postlåda. Den är viktigare än en skärmbild, eftersom alias, GUID:er och `LegacyExchangeDN` senare kan återskapas exakt:

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

Delegeringar och vidarebefordringar kräver egna exporter. Minst följande information bör säkerhetskopieras separat:

```powershell
$mailbox = Get-Mailbox -Identity user01@contoso.com

$mailbox | Format-List ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward
Get-MailboxPermission -Identity $mailbox.Identity
Get-ADPermission -Identity $mailbox.DistinguishedName
Get-InboxRule -Mailbox $mailbox.Identity
Get-CalendarProcessing -Identity $mailbox.Identity -ErrorAction SilentlyContinue
```

Dessa konfigurationer flyttas inte automatiskt till den nya molnpostlådan.

## Fas 3: Koppla från lokal postlåda och aktivera RemoteMailbox

Den faktiska cutovern är kort men får stora konsekvenser. `Disable-Mailbox` tar bort Exchange-attributen från AD-användaren och kopplar från den lokala postlådan. Postlådedatan finns kvar som en disconnected mailbox tills databasens kvarhållningstid löper ut. Omedelbart därefter aktiverar `Enable-RemoteMailbox` samma AD-användare för Exchange Online.

Följande skript behandlar endast dubbelt godkända rader. Det bevarar den primära SMTP-adressen, alla befintliga proxyadresser och den gamla `LegacyExchangeDN` som X500-adress. X500-posten förhindrar NDR:er vid svar på äldre meddelanden eller vid gamla Outlook-autocomplete-poster.

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

Skriptet är avsiktligt inte en helt autonom migreringsmotor. Det avbryter körningen vid den första motsägelsen så att en administratör kan bedöma orsak och tillstånd. Före en produktionsbatch bör koden valideras med några få testpostlådor och de Exchange-versioner som används.

## Fas 4: Synkronisera, licensiera och verifiera

Efter den lokala ändringen startas en delta-cykel på Entra Connect-servern:

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

För användarpostlådor måste därefter en giltig Exchange Online-tjänstplan vara tilldelad, exempelvis via gruppbaserad licensiering. Delade postlådor, rums- och enhetspostlådor ska bedömas enligt aktuella Microsoft-licensvillkor och de funktioner som behövs.

Etableringen är asynkron. Microsoft anger vanligen mindre än 30 minuter för normala ändringar, men i enskilda fall upp till 24 timmar. Under denna tid bör det föregående e-postflödet hålla kvar meddelanden kontrollerat i stället för att leverera dem till ett mål som ännu inte är klart.

Den lokala kontrollen måste nu visa en RemoteMailbox:

```powershell
Get-RemoteMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,RemoteRoutingAddress,EmailAddresses

Get-Mailbox -Identity user01@contoso.com -ErrorAction SilentlyContinue
```

I Exchange Online kontrolleras om den tidigare MailUser har blivit en riktig postlåda:

```powershell
Get-EXORecipient -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,EmailAddresses

Get-EXOMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,ExchangeGuid
```

Batchen betraktas som avslutad först när även följande tester har lyckats:

- Leverans externt och internt
- Skickande externt och internt
- Svar på ett gammalt meddelande för att kontrollera X500-adressen
- Inloggning med Outlook och Outlook på webben
- Delegeringar och Send-As
- Vidarebefordringar och transportregler
- Rums- och enhetsbokningar
- Program, skannrar och SMTP-reläer
- Message Trace med leverans till den nya molnpostlådan

## Rollback och städning

Den lokala källpostlådan får inte raderas med `Remove-StoreMailbox` under valideringsfasen. Så länge den finns kvar som disconnected mailbox inom postlådekvarhållningen finns fortfarande en teknisk möjlighet att återgå. En rollback kräver dock en kontrollerad återställning av RemoteMailbox-attributen och att den lokala postlådan återansluts; samtidigt måste det förhindras att två aktiva leveransmål uppstår.

Före en rollback måste därför e-postflöde, synkroniseringstillstånd och meddelanden som redan har kommit in i molnet säkerhetskopieras. En återgång är inte en enkel enradsåtgärd och ska ingå som ett testat runbook i ändringen.

Efter lyckad acceptans rensas exportbegäranden, PST-filer arkiveras enligt skydds- och kvarhållningskonceptet och tillfälliga behörigheter på exportresursen tas bort:

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Remove-MailboxExportRequest -Confirm:$false
```

De disconnected mailboxes bör slutligt rensas först efter att det överenskomna rollback-fönstret har löpt ut och i enlighet med kvarhållningskonceptet.

## Slutsats

Om Hybrid Remote Moves inte är tillåtna och inga postlådedata behöver flyttas till Exchange Online kan en befintlig synkroniserad AD-användare kontrollerat ställas om från en lokal postlåda till en ny molnpostlåda. Den kritiska delen är inte `Enable-RemoteMailbox`, utan processkontrollen runt omkring: fullständig inventering, verifierad PST-säkerhetskopiering, uttryckliga godkännanden, bevarande av proxy- och X500-adresser, ett kontrollerat e-postflöde samt ett verkligt rollback-fönster.

## Källor

1. [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchange/enable-remotemailbox?view=exchange-ps): Aktiverar en befintlig lokal AD-användare för en postlåda i den molnbaserade tjänsten och dokumenterar växlarna för användar-, delade, rums- och enhetspostlådor.

2. [Microsoft Learn – New-MailboxExportRequest](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-mailboxexportrequest?view=exchange-ps): Referens för export av primära och arkiverade lokala postlådor till PST-filer.

3. [Microsoft Learn – Mailbox import and export requests](https://learn.microsoft.com/en-us/exchange/mailbox-import-and-export-requests-exchange-2013-help): Förutsättningar för exportresurs, behörigheter och Exchange Trusted Subsystem.

4. [Microsoft Learn – Disable or delete a mailbox in Exchange Server](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disable-or-delete-mailboxes): Beteendet för `Disable-Mailbox`, borttagning av Exchange-attributen och kvarhållning av den frånkopplade postlådan.

5. [Microsoft Learn – Disconnected mailboxes](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disconnected-mailboxes): Återanslutning, återställning och slutgiltig radering av frånkopplade postlådor.

6. [Microsoft Learn – Delays in provisioning of a user or mailbox](https://learn.microsoft.com/en-us/troubleshoot/exchange/user-and-shared-mailboxes/delays-provision-mailbox-sync-changes): Typisk varaktighet och felsökning vid etablering i Exchange Online.

7. [Microsoft Learn – Move mailboxes between on-premises and Exchange Online organizations](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes): Den reguljära Hybrid Remote Move som referens och avgränsning mot den nyetablering som beskrivs här.
