---
title: "EXO-migrering utan Remote Move"
navTitle: "EXO-migrering utan Remote Move"
description: "Så här etableras lokala Exchange-postlådor kontrollerat som nya, tomma postlådor i Exchange Online: PST-säkerhetskopiering, CSV-godkännande, RemoteMailbox, synkronisering, validering och återställning."
date: "2026-08-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "12 min. läsning"
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
translationSourceHash: 8ef7108bef25cbfdabbdd58d6ed17a1cc34ba2da65eb7e685180b0976f99d62c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:34:31.348Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/exo-migrering-utan-remote-move
---

# EXO-migrering utan Remote Move

En Hybrid Remote Move är det normala sättet att flytta en lokal Exchange-postlåda inklusive innehåll till Exchange Online. Alla organisationer tillåter inte denna migreringsväg. Om en säkerhetspolicy utesluter Remote Moves kan ett medvetet annorlunda tillvägagångssätt vara försvarbart: Den lokala postlådan säkerhetskopieras som PST, kopplas från den synkroniserade AD-användaren och en ny, tom postlåda etableras i Exchange Online för samma användare.

Denna väg är **inte en postlådemigrering**. Den överför varken meddelanden, kalendrar, regler eller behörigheter till molnet. PST-filen används uteslutande som säkerhetskopia och importeras inte i detta scenario. Processen är därför endast lämplig om en tom målpostlåda är verksamhetsmässigt acceptabel och förlusten av den aktiva postlådans konfiguration uttryckligen har godkänts.

## Måltillstånd och hårda förutsättningar

Efter cutover finns samma AD-användare kvar. Lokalt hanteras den dock inte längre som `UserMailbox`, `SharedMailbox`, `RoomMailbox` eller `EquipmentMailbox`, utan som `RemoteMailbox`. Efter synkroniseringen representerar detta objekt den nya postlådan i Exchange Online.

Det önskade tillståndet ser ut så här:

1. Den lokala postlådan är fullständigt säkerhetskopierad som PST.
2. Den lokala postlådan är frånkopplad men ännu inte slutgiltigt borttagen inom den konfigurerade retentionstiden.
3. Den befintliga AD-användaren är aktiverad som RemoteMailbox.
4. Primär adress, alias och den gamla `LegacyExchangeDN` behålls.
5. Entra Connect har synkroniserat ändringarna.
6. För användarpostlådor har en Exchange Online-tjänstplan tilldelats.
7. Exchange Online visar en riktig molnpostlåda och e-postflödet avslutas där.

Följande måste också vara klarlagt innan start:

- PST-resursen är tillgänglig via UNC. Gruppen `Exchange Trusted Subsystem` har läs- och skrivbehörighet där.
- Det körande kontot har hanteringsrollen `Mailbox Import Export`.
- PST-filen är endast den överenskomna säkerhetskopian; ingen senare import är planerad.
- Bevarande, Litigation Hold, eDiscovery och regulatoriska krav har granskats separat.
- Ställföreträdare, Send-As, Send-on-Behalf, vidarebefordringar, Inkorgsregler, mobila enheter och programåtkomster är inventerade.
- Under export och omkoppling hålls inkommande meddelanden kontrollerat kvar i den främre gatewayen. Användare och program får inte längre skriva till källpostlådan.
- Retentionen för den lokala postlådedatabasen täcker återställningsfönstret.

## Varför en CSV-godkännandelista är oumbärlig

En direkt pipeline som `Get-Mailbox | Disable-Mailbox` är för riskabel för detta förfarande. Den skulle även kunna omfatta system-, Discovery- eller på annat sätt ej godkända postlådor. Följande process arbetar därför med två uttryckliga godkännanden:

- `Action=CUTOVER` avgör vilken rad som faktiskt får ställas om.
- `PstVerified=YES` bekräftar att exportfilen har granskats tekniskt och organisatoriskt.

Först skapas endast inventariet:

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
<summary>Alternativen förklaras</summary>

| Alternativ | Effekt |
|---|---|
| `Get-Mailbox -ResultSize Unlimited` | Tar bort standardgränsen på 1 000 resultat; utan denna parameter saknas postlådor i inventariet i stora miljöer |
| `-RecipientTypeDetails UserMailbox,SharedMailbox,RoomMailbox,EquipmentMailbox` | Begränsar frågan till de fyra postlådetyper som ska ställas om; system- och Discovery-postlådor lämnas utanför |
| `Sort-Object PrimarySmtpAddress` | Sorterar utdata efter den primära SMTP-adressen så att CSV-filen är stabilt ordnad vid verksamhetsgranskningen |
| `Export-Csv -Path` | Sökväg till CSV-filen |
| `-NoTypeInformation` | Undertrycker typ-rubriken `#TYPE ...`, som äldre PowerShell-versioner annars skriver som första rad |
| `-Encoding UTF8` | Skriver filen UTF-8-kodad så att diakritiska tecken i visningsnamn bevaras korrekt |

</details>

Filen rensas sedan verksamhetsmässigt. Endast faktiskt godkända postlådor får `Action=CUTOVER`. Systempostlådor och specialobjekt hör inte hemma i denna lista.

## Fas 1: Säkerhetskopiera primärpostlåda och arkiv som PST

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

<details class="options-details">
<summary>Alternativen förklaras</summary>

| Alternativ | Effekt |
|---|---|
| `Import-Csv $CsvPath` | Läser in godkännandelistan; varje rad blir ett objekt med CSV-kolumnerna som egenskaper |
| `Where-Object Action -eq "CUTOVER"` | Bearbetar endast uttryckligen godkända rader |
| `New-MailboxExportRequest -Mailbox` | Källpostlåda för exporten, här Identity från CSV-raden |
| `-FilePath` | Målsökväg för PST-filen; måste vara en UNC-sökväg eftersom cmdleten avvisar lokala sökvägar |
| `-Name` | Unikt namn på begäran; möjliggör senare specifik koppling mellan primär- och arkivexport |
| `-BatchName` | Samlar alla begäranden i en körning under ett batchnamn; grund för statusfrågor och rensning |
| `-IsArchive` | Exporterar onlinearkivet i stället för primärpostlådan; därför den andra begäran per postlåda med aktivt arkiv |

</details>

Exporten är godkänd först när **varje** begäran har statusen `Completed`:

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Get-MailboxExportRequestStatistics -IncludeReport |
    Format-Table DisplayName,Status,PercentComplete,FailureCode,Message -AutoSize

Get-MailboxExportRequest -BatchName $BatchName |
    Where-Object Status -ne "Completed"
```

<details class="options-details">
<summary>Alternativen förklaras</summary>

| Alternativ | Effekt |
|---|---|
| `Get-MailboxExportRequest -BatchName` | Listar alla exportbegäranden för den angivna batchen |
| `Get-MailboxExportRequestStatistics -IncludeReport` | Kompletterar statistiken med den detaljerade historikrapporten där felorsaker för enskilda begäranden finns |
| `Format-Table ... -AutoSize` | Tabellvisning av de angivna egenskaperna; `-AutoSize` anpassar kolumnbredderna till innehållet |
| `Where-Object Status -ne "Completed"` | Filtrerar alla ännu inte slutförda eller misslyckade begäranden; utdata måste vara tom innan processen fortsätter |

</details>

Dessutom måste filernas existens, storlek, läsbarhet, övertagande i säkerhetskopiering och åtkomstskydd kontrolleras. Först därefter anges `PstVerified=YES` för motsvarande CSV-rad.

## Fas 2: Säkerhetskopiera postlådedata och Exchange-attribut

Före den första ändringen skapas en maskinläsbar ögonblicksbild per postlåda. Den är viktigare än en skärmbild eftersom alias, GUID:er och `LegacyExchangeDN` senare kan rekonstrueras exakt:

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
<summary>Alternativen förklaras</summary>

| Alternativ | Effekt |
|---|---|
| `New-Item -ItemType Directory -Path ... -Force` | Skapar katalogen för ögonblicksbilden; `-Force` undertrycker felet om den redan finns |
| `Get-Mailbox -Identity` | Hämtar det aktuella postlådeobjektet för respektive CSV-rad |
| `Select-Object Identity,...,ServerName` | Reducerar objektet till de attribut som krävs för senare rekonstruktion (GUID:er, adresser, `LegacyExchangeDN`, databas) |
| `Export-Clixml` | Serialiserar objektet som typbevarande CLIXML; till skillnad från CSV bevaras flervärda egenskaper som `EmailAddresses` fullständigt och kan läsas in igen med `Import-Clixml` |

</details>

Delegeringar och vidarebefordringar kräver egna exporter. Åtminstone följande information bör säkerhetskopieras separat:

```powershell
$mailbox = Get-Mailbox -Identity user01@contoso.com

$mailbox | Format-List ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward
Get-MailboxPermission -Identity $mailbox.Identity
Get-ADPermission -Identity $mailbox.DistinguishedName
Get-InboxRule -Mailbox $mailbox.Identity
Get-CalendarProcessing -Identity $mailbox.Identity -ErrorAction SilentlyContinue
```

<details class="options-details">
<summary>Alternativen förklaras</summary>

| Alternativ | Effekt |
|---|---|
| `Format-List ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward` | Visar postlådans tre vidarebefordringsattribut i listvy |
| `Get-MailboxPermission -Identity` | Listar postlådebehörigheter som Full Access |
| `Get-ADPermission -Identity` | Listar AD-behörigheter på användarobjektet, bland annat Send-As; förväntar här Distinguished Name |
| `Get-InboxRule -Mailbox` | Listar postlådans serversidesregler för inkorgen |
| `Get-CalendarProcessing -Identity` | Visar bokningskonfigurationen; relevant för rums- och utrustningspostlådor |
| `-ErrorAction SilentlyContinue` | Undertrycker felet för postlådetyper utan bokningskonfiguration så att säkerhetskopieringen inte avbryts |

</details>

Dessa konfigurationer flyttas inte automatiskt till den nya molnpostlådan.

## Fas 3: Koppla från lokal postlåda och aktivera RemoteMailbox

Själva cutover-processen är kort men får stora följder. `Disable-Mailbox` tar bort Exchange-attributen från AD-användaren och kopplar från den lokala postlådan. Postlådedatan behålls som en disconnected mailbox tills databasretentionen löper ut. Omedelbart därefter aktiverar `Enable-RemoteMailbox` samma AD-användare för Exchange Online.

Följande skript bearbetar endast dubbelt godkända rader. Det bevarar den primära SMTP-adressen, alla befintliga proxyadresser och den gamla `LegacyExchangeDN` som X500-adress. X500-posten förhindrar NDR:er vid svar på äldre meddelanden eller gamla Outlook-autoslutförandeposter.

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
<summary>Alternativen förklaras</summary>

| Alternativ | Effekt |
|---|---|
| `Get-Mailbox -Identity ... -ErrorAction Stop` | Hämtar källpostlådan; `-ErrorAction Stop` gör ett uppslagsfel till ett avbrytande fel i stället för att fortsätta tyst |
| `Disable-Mailbox -Identity` | Tar bort Exchange-attributen från AD-användaren och kopplar från den lokala postlådan; data finns kvar som disconnected mailbox i databasen |
| `-Confirm:$false` | Undertrycker den interaktiva frågan; godkännande sker här via CSV-listan, inte vid prompten |
| `Enable-RemoteMailbox -Identity` | Aktiverar samma AD-användare som RemoteMailbox för Exchange Online |
| `-Alias` | Återställer Exchange-aliaset till värdet från godkännandelistan |
| `-PrimarySmtpAddress` | Behåller den tidigare primära SMTP-adressen |
| `-RemoteRoutingAddress` | Måladress i routningsdomänen `mail.onmicrosoft.com`, via vilken den lokala Exchange når molnpostlådan |
| `-ACLableSyncedObjectEnabled` | Märker objektet som ACL-kompatibelt så att behörigheter som Full Access kan utvärderas i Exchange Online efter synkroniseringen |
| `-Shared` / `-Room` / `-Equipment` | Skapar respektive specialtyp i stället för en användarpostlåda; skriptet anger exakt en växel som passar källtypen |
| `Set-RemoteMailbox -EmailAddressPolicyEnabled $false` | Undantar objektet från e-postadresspolicyn så att den inte skriver över manuellt angivna adresser |
| `-EmailAddresses` | Anger den fullständiga, dubblettfria adresslistan inklusive gamla proxyadresser, routningsadress och X500-post |
| `Get-RemoteMailbox -Identity` | Kontrollfråga av resultatet direkt efter omställningen |

</details>

Skriptet är medvetet inte ett helautomatiskt migreringsverktyg. Det avbryter körningen vid första motsägelse så att en administratör kan bedöma orsak och tillstånd. Före en produktionsbatch bör koden valideras med ett fåtal testpostlådor och de Exchange-versioner som används.

## Fas 4: Synkronisera, licensiera och verifiera

Efter den lokala ändringen startas en deltacykel på Entra Connect-servern:

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

<details class="options-details">
<summary>Alternativen förklaras</summary>

| Alternativ | Effekt |
|---|---|
| `-PolicyType Delta` | Synkroniserar endast objekt som har ändrats sedan den senaste cykeln; alternativet `Initial` skulle vara en fullständig och betydligt längre körning |

</details>

För användarpostlådor måste därefter en giltig Exchange Online-tjänstplan vara tilldelad, exempelvis via gruppbaserad licensiering. Delade postlådor samt rums- och utrustningspostlådor ska bedömas enligt aktuella Microsoft-licensvillkor och de funktioner som behövs.

Etableringen är asynkron. Microsoft anger vanligtvis mindre än 30 minuter för normala ändringar, men i enskilda fall upp till 24 timmar. Under denna tid bör det främre e-postflödet hålla meddelanden kontrollerat kvar i stället för att leverera dem till ett mål som ännu inte är redo.

Den lokala kontrollen måste nu visa en RemoteMailbox:

```powershell
Get-RemoteMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,RemoteRoutingAddress,EmailAddresses

Get-Mailbox -Identity user01@contoso.com -ErrorAction SilentlyContinue
```

<details class="options-details">
<summary>Alternativen förklaras</summary>

| Alternativ | Effekt |
|---|---|
| `Get-RemoteMailbox -Identity` | Måste returnera det omställda objektet som RemoteMailbox |
| `Format-List RecipientTypeDetails,...` | Visar typ, adresser och routningsadress för kontroll i listvy |
| `Get-Mailbox -Identity ... -ErrorAction SilentlyContinue` | Motkontroll: anropet får inte längre returnera något eftersom ingen ansluten postlåda längre finns lokalt; `-ErrorAction SilentlyContinue` undertrycker det då förväntade felmeddelandet |

</details>

I Exchange Online kontrolleras om den tidigare MailUser har blivit en riktig postlåda:

```powershell
Get-EXORecipient -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,EmailAddresses

Get-EXOMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,ExchangeGuid
```

<details class="options-details">
<summary>Alternativen förklaras</summary>

| Alternativ | Effekt |
|---|---|
| `Get-EXORecipient -Identity` | Visar mottagartypen i Exchange Online; `UserMailbox` respektive specialtypen förväntas, inte längre `MailUser` |
| `Get-EXOMailbox -Identity` | Returnerar endast riktiga molnpostlådor; en träff bevisar att etableringen är klar |
| `Format-List ...,ExchangeGuid` | Listar kontrollattributen; `ExchangeGuid` identifierar den nya molnpostlådan entydigt |

</details>

Batchen anses avslutad först när även följande tester har lyckats:

- Leverans externt och internt
- Skickande externt och internt
- Svar på ett gammalt meddelande för att kontrollera X500-adressen
- Inloggning med Outlook och Outlook på webben
- Ställföreträdare och Send-As
- Vidarebefordringar och transportregler
- Rums- och utrustningsbokningar
- Program, skannrar och SMTP-reläer
- Message Trace med leverans till den nya molnpostlådan

## Återställning och rensning

Den lokala källpostlådan får inte raderas med `Remove-StoreMailbox` under valideringsfasen. Så länge den finns som disconnected mailbox inom postlåderetentionen finns fortfarande en teknisk möjlighet att återgå. En återställning kräver dock en kontrollerad återställning av RemoteMailbox-attributen och återanslutning av den lokala postlådan; samtidigt måste det förhindras att två aktiva leveransmål uppstår.

Före en återställning måste därför e-postflöde, synkroniseringsstatus och meddelanden som redan har kommit in i molnet säkras. Att byta tillbaka är ingen enkel enradare och ska ingå som ett testat runbook i ändringen.

Efter godkänd acceptans rensas exportbegäranden, PST-filer arkiveras enligt skydds- och bevarandekonceptet och temporära behörigheter på exportresursen tas bort:

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Remove-MailboxExportRequest -Confirm:$false
```

<details class="options-details">
<summary>Alternativen förklaras</summary>

| Alternativ | Effekt |
|---|---|
| `Get-MailboxExportRequest -BatchName` | Väljer exakt exportbegärandena i den avslutade batchen |
| `Remove-MailboxExportRequest -Confirm:$false` | Tar bort begärandena utan fråga; själva PST-filerna påverkas inte |

</details>

De disconnected mailboxes bör slutgiltigt rensas först efter det överenskomna återställningsfönstret och enligt retention-konceptet.

## Slutsats

Om Hybrid Remote Moves inte är tillåtna och inga postlådedata behöver tas över till Exchange Online kan en befintlig synkroniserad AD-användare kontrollerat ställas om från en lokal postlåda till en ny molnpostlåda. Den kritiska delen är inte `Enable-RemoteMailbox`, utan processkontrollen runt omkring: fullständig inventering, verifierad PST-säkerhetskopiering, uttryckliga godkännanden, bevarande av proxy- och X500-adresser, ett kontrollerat e-postflöde samt ett verkligt återställningsfönster.

## Ändringshistorik

- **25 oktober 2026:** Källänken uppdaterades till den nya adressen på Microsoft Learn.

## Källor

1. [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox?view=exchange-ps): Aktiverar en befintlig lokal AD-användare för en postlåda i den molnbaserade tjänsten och dokumenterar växlarna för användar-, delade, rums- och utrustningspostlådor.

2. [Microsoft Learn – New-MailboxExportRequest](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-mailboxexportrequest?view=exchange-ps): Referens för export av primära och arkiverade lokala postlådor till PST-filer.

3. [Microsoft Learn – Mailbox import and export requests](https://learn.microsoft.com/en-us/exchange/mailbox-import-and-export-requests-exchange-2013-help): Förutsättningar för exportresurs, behörigheter och Exchange Trusted Subsystem.

4. [Microsoft Learn – Disable or delete a mailbox in Exchange Server](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disable-or-delete-mailboxes): Beteende för `Disable-Mailbox`, borttagning av Exchange-attributen och bevarande av den frånkopplade postlådan.

5. [Microsoft Learn – Disconnected mailboxes](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disconnected-mailboxes): Återanslutning, återställning och slutgiltig radering av frånkopplade postlådor.

6. [Microsoft Learn – Delays in provisioning of a user or mailbox](https://learn.microsoft.com/en-us/troubleshoot/exchange/user-and-shared-mailboxes/delays-provision-mailbox-sync-changes): Typisk tidsåtgång och felsökning vid etablering i Exchange Online.

7. [Microsoft Learn – Move mailboxes between on-premises and Exchange Online organizations](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes): Den vanliga Hybrid Remote Move som referens och avgränsning mot den nyetablering som beskrivs här.
