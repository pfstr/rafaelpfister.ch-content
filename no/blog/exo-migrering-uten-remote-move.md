---
title: "EXO-migrering uten Remote Move"
navTitle: "EXO-migrering uten Remote Move"
description: "Slik klargjøres lokale Exchange-postbokser kontrollert som nye, tomme postbokser i Exchange Online: PST-sikkerhetskopi, CSV-godkjenning, RemoteMailbox, synkronisering, validering og tilbakeføring."
date: "2026-08-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "12 min. lesetid"
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
slug: "exo-migrering-uten-remote-move"
translationId: "article-8f3c1b7a62d94e50"
draft: false
translationOf: exchange-mailboxen-ohne-remote-move-cloud-neu-erstellen
url: https://rafaelpfister.ch/no/blog/exo-migrering-uten-remote-move
translationSourceHash: 861f11b6e2f1e316ca773f049637fa2ac6ed5efdab5ec74d8c28178f3ea7e98c
translationModel: gpt-5.6-terra
translatedAt: 2026-08-08T13:49:11.266Z
translationReview: automatic
---

# EXO-migrering uten Remote Move

En Hybrid Remote Move er den vanlige metoden for å flytte en lokal Exchange-postboks med innhold til Exchange Online. Ikke alle organisasjoner tillater denne migreringsveien. Hvis en sikkerhetspolicy utelukker Remote Moves, kan en bevisst annerledes tilnærming være forsvarlig: Den lokale postboksen sikkerhetskopieres som PST, kobles fra den synkroniserte AD-brukeren, og det klargjøres en ny, tom postboks i Exchange Online for den samme brukeren.

Denne metoden er **ikke en postboksmigrering**. Den overfører verken meldinger, kalender, regler eller tillatelser til skyen. PST-filen brukes utelukkende som sikkerhetskopi og importeres ikke i dette scenariet. Fremgangsmåten egner seg derfor bare når en tom målpostboks er faglig akseptabel, og tapet av den aktive postboks-konfigurasjonen er uttrykkelig godkjent.

## Måltilstand og strenge forutsetninger

Etter cutover beholdes den samme AD-brukeren. Lokalt administreres den imidlertid ikke lenger som `UserMailbox`, `SharedMailbox`, `RoomMailbox` eller `EquipmentMailbox`, men som `RemoteMailbox`. Etter synkroniseringen representerer dette objektet den nye postboksen i Exchange Online.

Ønsket tilstand ser slik ut:

1. Den lokale postboksen er fullstendig sikkerhetskopiert som PST.
2. Den lokale postboksen er frakoblet, men ennå ikke endelig slettet innenfor den konfigurerte oppbevaringsperioden.
3. Den eksisterende AD-brukeren er aktivert som RemoteMailbox.
4. Primæradresse, aliaser og den gamle `LegacyExchangeDN` beholdes.
5. Entra Connect har synkronisert endringene.
6. For brukerpostbokser er en Exchange Online-tjenesteplan tildelt.
7. Exchange Online viser en reell skypostboks, og e-postflyten ender der.

Før oppstart må følgende også være avklart:

- PST-delingsområdet er tilgjengelig via UNC. Gruppen `Exchange Trusted Subsystem` har lese- og skrivetilgang der.
- Kontoen som utfører oppgaven, har administrasjonsrollen `Mailbox Import Export`.
- PST-filen er bare den avtalte sikkerhetskopien; ingen senere import er planlagt.
- Oppbevaring, Litigation Hold, eDiscovery og regulatoriske krav er vurdert separat.
- Stedfortredere, Send-As, Send-on-Behalf, videresendinger, innboksregler, mobile enheter og applikasjonstilganger er kartlagt.
- Under eksport og omkobling holdes innkommende meldinger kontrollert tilbake i den foranstilte gatewayen. Brukere og applikasjoner må ikke lenger skrive til kildepostboksen.
- Retensjonen for den lokale postboksdatabasen dekker tilbakeføringsvinduet.

## Hvorfor en CSV-godkjenningsliste er uunnværlig

En direkte pipeline som `Get-Mailbox | Disable-Mailbox` er for risikabel for denne prosessen. Den kan også inkludere system-, Discovery- eller andre ikke-godkjente postbokser. Den følgende prosessen arbeider derfor med to eksplisitte godkjenninger:

- `Action=CUTOVER` avgjør hvilken rad som faktisk kan omkobles.
- `PstVerified=YES` bekrefter at eksportfilen er kontrollert teknisk og organisatorisk.

Først opprettes kun inventaret:

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

Deretter kvalitetssikres filen faglig. Bare postbokser som faktisk er godkjent, får `Action=CUTOVER`. Systempostbokser og spesialobjekter hører ikke hjemme i denne listen.

## Fase 1: Sikre primærpostboks og arkiv som PST

`New-MailboxExportRequest` skriver kun til en UNC-bane. Det opprettes et unikt filnavn for hver postboks. Et aktivt nettarkiv i lokal Exchange eksporteres separat:

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

Eksporten er først godkjent når **hver** forespørsel har statusen `Completed`:

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Get-MailboxExportRequestStatistics -IncludeReport |
    Format-Table DisplayName,Status,PercentComplete,FailureCode,Message -AutoSize

Get-MailboxExportRequest -BatchName $BatchName |
    Where-Object Status -ne "Completed"
```

I tillegg må filenes eksistens, størrelse, lesbarhet, overføring til sikkerhetskopi og tilgangsbeskyttelse kontrolleres. Først deretter settes `PstVerified=YES` for den aktuelle CSV-raden.

## Fase 2: Sikre postboksdata og Exchange-attributter

Før den første endringen opprettes et maskinlesbart øyeblikksbilde per postboks. Det er viktigere enn et skjermbilde, fordi aliaser, GUID-er og `LegacyExchangeDN` senere kan rekonstrueres nøyaktig:

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

Delegasjoner og videresendinger krever egne eksporter. Minst denne informasjonen bør sikres separat:

```powershell
$mailbox = Get-Mailbox -Identity user01@contoso.com

$mailbox | Format-List ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward
Get-MailboxPermission -Identity $mailbox.Identity
Get-ADPermission -Identity $mailbox.DistinguishedName
Get-InboxRule -Mailbox $mailbox.Identity
Get-CalendarProcessing -Identity $mailbox.Identity -ErrorAction SilentlyContinue
```

Disse konfigurasjonene overføres ikke automatisk til den nye skypostboksen.

## Fase 3: Koble fra lokal postboks og aktivere RemoteMailbox

Selve cutoveren er kort, men får store konsekvenser. `Disable-Mailbox` fjerner Exchange-attributtene fra AD-brukeren og kobler fra den lokale postboksen. Postboksdataene beholdes som en frakoblet postboks frem til databasens retensjon utløper. Rett etterpå aktiverer `Enable-RemoteMailbox` den samme AD-brukeren for Exchange Online.

Følgende skript behandler utelukkende dobbelt godkjente rader. Det bevarer den primære SMTP-adressen, alle eksisterende proxy-adresser og den gamle `LegacyExchangeDN` som en X500-adresse. X500-oppføringen forhindrer NDR-er ved svar på eldre meldinger eller ved gamle Outlook-autofullføringsoppføringer.

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

Skriptet er med hensikt ikke en fullt autonom migreringsmotor. Det avslutter kjøringen ved første avvik, slik at en administrator kan vurdere årsak og tilstand. Før en produksjonsbatch bør koden valideres med noen få testpostbokser og Exchange-versjonene som brukes.

## Fase 4: Synkronisere, lisensiere og verifisere

Etter den lokale endringen startes en delta-syklus på Entra Connect-serveren:

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

For brukerpostbokser må det deretter være tildelt en gyldig Exchange Online-tjenesteplan, for eksempel gjennom gruppebasert lisensiering. Delte postbokser, rompostbokser og utstyrspostbokser må vurderes ut fra gjeldende lisensvilkår fra Microsoft og nødvendige funksjoner.

Klargjøringen er asynkron. Microsoft oppgir vanligvis under 30 minutter for normale endringer, men i enkelte tilfeller opptil 24 timer. I denne perioden bør den foranstilte e-postflyten holde meldinger kontrollert tilbake i stedet for å levere dem til et mål som ennå ikke er klart.

Den lokale kontrollen må nå vise en RemoteMailbox:

```powershell
Get-RemoteMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,RemoteRoutingAddress,EmailAddresses

Get-Mailbox -Identity user01@contoso.com -ErrorAction SilentlyContinue
```

I Exchange Online kontrolleres det om den tidligere MailUser-en har blitt en reell postboks:

```powershell
Get-EXORecipient -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,EmailAddresses

Get-EXOMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,ExchangeGuid
```

Batchen anses først som fullført når følgende tester i tillegg er vellykkede:

- Levering eksternt og internt
- Sending eksternt og internt
- Svar på en gammel melding for å kontrollere X500-adressen
- Pålogging med Outlook og Outlook på nettet
- Stedfortredere og Send-As
- Videresendinger og transportregler
- Rom- og utstyrsbestillinger
- Applikasjoner, skannere og SMTP-reléer
- Message Trace med levering til den nye skypostboksen

## Tilbakeføring og opprydding

Den lokale kildepostboksen må ikke slettes med `Remove-StoreMailbox` i valideringsfasen. Så lenge den finnes som en frakoblet postboks innenfor postboksretensjonen, finnes det fortsatt en teknisk tilbakeføringsmulighet. En tilbakeføring krever imidlertid en kontrollert reversering av RemoteMailbox-attributtene og ny tilkobling av den lokale postboksen; samtidig må det forhindres at to aktive leveringsmål oppstår.

Før en tilbakeføring må derfor e-postflyt, synkroniseringsstatus og meldinger som allerede er mottatt i skyen, sikres. Å bytte tilbake er ikke en enkel énlinjekommando og skal inngå som en testet runbook i endringen.

Etter vellykket godkjenning ryddes eksportforespørslene opp, PST-filene arkiveres i henhold til beskyttelses- og oppbevaringskonseptet, og midlertidige tillatelser på eksportdelingen fjernes:

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Remove-MailboxExportRequest -Confirm:$false
```

De frakoblede postboksene bør først slettes endelig etter at det avtalte tilbakeføringsvinduet er utløpt, og i henhold til retensjonskonseptet.

## Konklusjon

Hvis Hybrid Remote Moves ikke er tillatt og ingen postboksdata må overføres til Exchange Online, kan en eksisterende synkronisert AD-bruker kontrolleres fra en lokal postboks til en ny skypostboks. Den kritiske delen er ikke `Enable-RemoteMailbox`, men prosesskontrollen rundt den: fullstendig kartlegging, verifisert PST-sikkerhetskopi, eksplisitte godkjenninger, bevaring av proxy- og X500-adresser, kontrollert e-postflyt og et reelt tilbakeføringsvindu.

## Kilder

1. [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchange/enable-remotemailbox?view=exchange-ps): Aktiverer en eksisterende lokal AD-bruker for en postboks i den skybaserte tjenesten og dokumenterer parameterne for bruker-, delte, rom- og utstyrspostbokser.

2. [Microsoft Learn – New-MailboxExportRequest](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-mailboxexportrequest?view=exchange-ps): Referanse for eksport av primære og arkiverte lokale postbokser til PST-filer.

3. [Microsoft Learn – Mailbox import and export requests](https://learn.microsoft.com/en-us/exchange/mailbox-import-and-export-requests-exchange-2013-help): Forutsetninger for eksportdeling, tillatelser og Exchange Trusted Subsystem.

4. [Microsoft Learn – Disable or delete a mailbox in Exchange Server](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disable-or-delete-mailboxes): Virkemåten til `Disable-Mailbox`, fjerning av Exchange-attributter og oppbevaring av den frakoblede postboksen.

5. [Microsoft Learn – Disconnected mailboxes](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disconnected-mailboxes): Koble til på nytt, gjenopprette og slette frakoblede postbokser permanent.

6. [Microsoft Learn – Delays in provisioning of a user or mailbox](https://learn.microsoft.com/en-us/troubleshoot/exchange/user-and-shared-mailboxes/delays-provision-mailbox-sync-changes): Typisk varighet og feilsøking ved klargjøring i Exchange Online.

7. [Microsoft Learn – Move mailboxes between on-premises and Exchange Online organizations](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes): Den vanlige Hybrid Remote Move som referanse og avgrensning mot nyoppsettet som beskrives her.
