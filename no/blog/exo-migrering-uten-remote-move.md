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
translationSourceHash: dc64d2c419e3ac0f4dd730785b3cd7f37c3f23effd2317feb4d61a46fa33401a
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T08:59:05.431Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/exo-migrering-uten-remote-move
---

# EXO-migrering uten Remote Move

En Hybrid Remote Move er den vanlige måten å flytte en lokal Exchange-postboks med innhold til Exchange Online på. Ikke alle organisasjoner tillater denne migreringsveien. Dersom en sikkerhetspolicy utelukker Remote Moves, kan en bevisst annerledes tilnærming være forsvarlig: Den lokale postboksen sikkerhetskopieres som PST, kobles fra den synkroniserte AD-brukeren, og det klargjøres en ny, tom postboks i Exchange Online for samme bruker.

Denne veien er **ikke en postboksmigrering**. Den overfører verken meldinger, kalendere, regler eller tillatelser til skyen. PST-filen brukes kun som sikkerhetskopi og importeres ikke i dette scenarioet. Fremgangsmåten egner seg derfor bare dersom en tom målpostboks er faglig akseptabel, og tapet av den aktive postboks-konfigurasjonen er uttrykkelig godkjent.

## Måltilstand og ufravikelige forutsetninger

Etter cutover beholdes samme AD-bruker. Lokalt administreres den imidlertid ikke lenger som `UserMailbox`, `SharedMailbox`, `RoomMailbox` eller `EquipmentMailbox`, men som `RemoteMailbox`. Etter synkroniseringen representerer dette objektet den nye postboksen i Exchange Online.

Ønsket tilstand ser slik ut:

1. Den lokale postboksen er fullstendig sikkerhetskopiert som PST.
2. Den lokale postboksen er frakoblet, men ikke endelig slettet innenfor den konfigurerte retention-perioden.
3. Den eksisterende AD-brukeren er aktivert som RemoteMailbox.
4. Primæradresse, aliaser og den gamle `LegacyExchangeDN` beholdes.
5. Entra Connect har synkronisert endringene.
6. En Exchange Online-tjenesteplan er tilordnet brukerpostbokser.
7. Exchange Online viser en ekte skypostboks, og e-postflyten ender der.

Følgende må også være avklart før oppstart:

- PST-delingen er tilgjengelig via UNC. Gruppen `Exchange Trusted Subsystem` har lese- og skrivetilgang der.
- Kontoen som utfører arbeidet, har administrasjonsrollen `Mailbox Import Export`.
- PST-filen er kun den avtalte sikkerhetskopien; ingen senere import er planlagt.
- Oppbevaring, Litigation Hold, eDiscovery og regulatoriske krav er kontrollert separat.
- Delegeringer, Send-As, Send-on-Behalf, videresendinger, innboksregler, mobile enheter og applikasjonstilganger er kartlagt.
- Under eksport og omkobling holdes innkommende meldinger kontrollert tilbake i den foranliggende gatewayen. Brukere og applikasjoner må ikke lenger skrive til kildepostboksen.
- Retention for den lokale postboksdatabasen dekker tilbakeføringsvinduet.

## Hvorfor en CSV-godkjenningsliste er uunnværlig

En direkte pipeline som `Get-Mailbox | Disable-Mailbox` er for risikabel for denne operasjonen. Den kan også inkludere system-, Discovery- eller andre ikke-godkjente postbokser. Den følgende prosessen arbeider derfor med to eksplisitte godkjenninger:

- `Action=CUTOVER` angir hvilken rad som faktisk kan omstilles.
- `PstVerified=YES` bekrefter at eksportfilen er kontrollert teknisk og organisatorisk.

Først opprettes bare inventaret:

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
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Get-Mailbox -ResultSize Unlimited` | Opphever standardgrensen på 1000 resultater; uten denne parameteren mangler postbokser i inventaret i store miljøer |
| `-RecipientTypeDetails UserMailbox,SharedMailbox,RoomMailbox,EquipmentMailbox` | Begrenser spørringen til de fire postbokstypene som skal omstilles; system- og Discovery-postbokser holdes utenfor |
| `Sort-Object PrimarySmtpAddress` | Sorterer utdata etter den primære SMTP-adressen, slik at CSV-filen er stabilt sortert ved faglig gjennomgang |
| `Export-Csv -Path` | Målbane for CSV-filen |
| `-NoTypeInformation` | Undertrykker typeoverskriften `#TYPE ...`, som eldre PowerShell-versjoner ellers skriver som første linje |
| `-Encoding UTF8` | Skriver filen UTF-8-kodet, slik at spesialtegn i visningsnavn bevares korrekt |

</details>

Filen renses deretter faglig. Bare postbokser som faktisk er godkjent, får `Action=CUTOVER`. Systempostbokser og spesialobjekter hører ikke hjemme i denne listen.

## Fase 1: Sikkerhetskopier primærpostboks og arkiv som PST

`New-MailboxExportRequest` skriver bare til en UNC-bane. Det opprettes et entydig filnavn for hver postboks. Et aktivt nettarkiv i lokal Exchange eksporteres separat:

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
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Import-Csv $CsvPath` | Leser inn godkjenningslisten; hver rad blir et objekt med CSV-kolonnene som egenskaper |
| `Where-Object Action -eq "CUTOVER"` | Behandler bare eksplisitt godkjente rader |
| `New-MailboxExportRequest -Mailbox` | Kildepostboks for eksporten (her identiteten fra CSV-raden) |
| `-FilePath` | Målbane for PST-filen; må være en UNC-bane, lokale baner avvises av cmdleten |
| `-Name` | Entydig navn på forespørselen; muliggjør senere målrettet tilordning av primær- og arkiveksport |
| `-BatchName` | Samler alle forespørsler fra én kjøring under et batchnavn; grunnlag for statusspørring og opprydding |
| `-IsArchive` | Eksporterer nettarkivet i stedet for primærpostboksen; derfor den andre forespørselen per postboks med aktivt arkiv |

</details>

Eksporten er først godkjent når **alle** forespørsler har status `Completed`:

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Get-MailboxExportRequestStatistics -IncludeReport |
    Format-Table DisplayName,Status,PercentComplete,FailureCode,Message -AutoSize

Get-MailboxExportRequest -BatchName $BatchName |
    Where-Object Status -ne "Completed"
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Get-MailboxExportRequest -BatchName` | Viser alle eksportforespørsler for den angitte batchen |
| `Get-MailboxExportRequestStatistics -IncludeReport` | Supplerer statistikken med den detaljerte historikkrapporten, som inneholder feilårsaker for enkeltforespørsler |
| `Format-Table ... -AutoSize` | Tabellarisk visning av de angitte egenskapene; `-AutoSize` tilpasser kolonnebreddene til innholdet |
| `Where-Object Status -ne "Completed"` | Filtrerer på alle forespørsler som ennå ikke er fullført eller som har feilet; utdata må være tomme før du fortsetter |

</details>

I tillegg må filenes eksistens, størrelse, lesbarhet, overføring til sikkerhetskopi og tilgangsbeskyttelse kontrolleres. Først deretter settes `PstVerified=YES` for den aktuelle CSV-raden.

## Fase 2: Sikkerhetskopier postboksdata og Exchange-attributter

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

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `New-Item -ItemType Directory -Path ... -Force` | Oppretter snapshot-katalogen; `-Force` undertrykker feilen dersom den allerede finnes |
| `Get-Mailbox -Identity` | Henter gjeldende postboksobjekt for den aktuelle CSV-raden |
| `Select-Object Identity,...,ServerName` | Reduserer objektet til attributtene som kreves for senere rekonstruksjon (GUID-er, adresser, `LegacyExchangeDN`, database) |
| `Export-Clixml` | Serialiserer objektet som typebevarende CLIXML; i motsetning til CSV beholdes flerverdier som `EmailAddresses` fullstendig og kan leses inn igjen med `Import-Clixml` |

</details>

Delegeringer og videresendinger krever egne eksporter. Minst denne informasjonen bør sikkerhetskopieres separat:

```powershell
$mailbox = Get-Mailbox -Identity user01@contoso.com

$mailbox | Format-List ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward
Get-MailboxPermission -Identity $mailbox.Identity
Get-ADPermission -Identity $mailbox.DistinguishedName
Get-InboxRule -Mailbox $mailbox.Identity
Get-CalendarProcessing -Identity $mailbox.Identity -ErrorAction SilentlyContinue
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Format-List ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward` | Viser postboksens tre videresendingsattributter i listevisning |
| `Get-MailboxPermission -Identity` | Viser postbokstillatelser som Full Access |
| `Get-ADPermission -Identity` | Viser AD-tillatelser på brukerobjektet, inkludert Send-As; forventer her Distinguished Name |
| `Get-InboxRule -Mailbox` | Viser postboksens serverbaserte innboksregler |
| `Get-CalendarProcessing -Identity` | Viser bestillingskonfigurasjonen; relevant for rom- og utstyrspostbokser |
| `-ErrorAction SilentlyContinue` | Undertrykker feilen for postbokstyper uten bestillingskonfigurasjon, slik at sikkerhetskopieringen ikke avbrytes |

</details>

Disse konfigurasjonene flyttes ikke automatisk til den nye skypostboksen.

## Fase 3: Koble fra lokal postboks og aktiver RemoteMailbox

Selve cutoveren er kort, men konsekvensrik. `Disable-Mailbox` fjerner Exchange-attributtene fra AD-brukeren og kobler fra den lokale postboksen. Postboksdataene beholdes som en disconnected mailbox frem til databaseretentionen utløper. Rett etterpå aktiverer `Enable-RemoteMailbox` samme AD-bruker for Exchange Online.

Det følgende skriptet behandler utelukkende dobbelt godkjente rader. Det bevarer den primære SMTP-adressen, alle eksisterende proxy-adresser og den gamle `LegacyExchangeDN` som X500-adresse. X500-oppføringen forhindrer NDR-er ved svar på eldre meldinger eller gamle Outlook-autofullføringsoppføringer.

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
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Get-Mailbox -Identity ... -ErrorAction Stop` | Henter kildepostboksen; `-ErrorAction Stop` gjør en oppslagsfeil til en avbrytende feil i stedet for å fortsette stille |
| `Disable-Mailbox -Identity` | Fjerner Exchange-attributtene fra AD-brukeren og kobler fra den lokale postboksen; dataene beholdes som en disconnected mailbox i databasen |
| `-Confirm:$false` | Undertrykker det interaktive spørsmålet; godkjenningen skjer her via CSV-listen, ikke i ledeteksten |
| `Enable-RemoteMailbox -Identity` | Aktiverer samme AD-bruker som RemoteMailbox for Exchange Online |
| `-Alias` | Setter Exchange-aliaset tilbake til verdien fra godkjenningslisten |
| `-PrimarySmtpAddress` | Bevarer den tidligere primære SMTP-adressen |
| `-RemoteRoutingAddress` | Måladresse i `mail.onmicrosoft.com`-rutingsdomenet, som den lokale Exchange bruker for å nå skypostboksen |
| `-ACLableSyncedObjectEnabled` | Merker objektet som ACL-kompatibelt, slik at tillatelser som Full Access kan evalueres i Exchange Online etter synkronisering |
| `-Shared` / `-Room` / `-Equipment` | Oppretter den aktuelle spesialtypen i stedet for en brukerpostboks; skriptet setter nøyaktig én bryter som passer kildetypen |
| `Set-RemoteMailbox -EmailAddressPolicyEnabled $false` | Unntar objektet fra e-postadressepolicyen, slik at den ikke overskriver manuelt angitte adresser |
| `-EmailAddresses` | Setter den komplette, dedupliserte adresselisten, inkludert gamle proxy-adresser, rutingsadresse og X500-oppføring |
| `Get-RemoteMailbox -Identity` | Kontrollspørring av resultatet rett etter omstillingen |

</details>

Skriptet er bevisst ikke et helautomatisk migreringsverktøy. Det avslutter kjøringen ved den første uoverensstemmelsen, slik at en administrator kan vurdere årsak og tilstand. Før en produksjonsbatch bør koden valideres med noen få testpostbokser og Exchange-versjonene som brukes.

## Fase 4: Synkroniser, lisensier og verifiser

Etter den lokale endringen startes en delta-syklus på Entra Connect-serveren:

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-PolicyType Delta` | Synkroniserer bare objekter som er endret siden forrige syklus; alternativet `Initial` ville vært en fullstendig og betydelig lengre kjøring |

</details>

For brukerpostbokser må det deretter være tilordnet en gyldig Exchange Online-tjenesteplan, for eksempel via gruppebasert lisensiering. Delte postbokser, rom- og utstyrspostbokser må vurderes i henhold til gjeldende Microsoft-lisensvilkår og nødvendige funksjoner.

Klargjøringen er asynkron. Microsoft oppgir vanligvis under 30 minutter for normale endringer, men i enkelttilfeller opptil 24 timer. I løpet av denne tiden bør den foranliggende e-postflyten holde meldinger kontrollert tilbake i stedet for å levere dem til et mål som ennå ikke er klart.

Den lokale kontrollen må nå vise en RemoteMailbox:

```powershell
Get-RemoteMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,RemoteRoutingAddress,EmailAddresses

Get-Mailbox -Identity user01@contoso.com -ErrorAction SilentlyContinue
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Get-RemoteMailbox -Identity` | Må returnere det omstilte objektet som RemoteMailbox |
| `Format-List RecipientTypeDetails,...` | Viser type, adresser og rutingsadresse for kontroll i listevisning |
| `Get-Mailbox -Identity ... -ErrorAction SilentlyContinue` | Motkontroll: Kallet må ikke returnere noe, fordi det ikke lenger finnes en tilkoblet postboks lokalt; `-ErrorAction SilentlyContinue` undertrykker den forventede feilmeldingen |

</details>

I Exchange Online kontrolleres det om den tidligere MailUser har blitt en ekte postboks:

```powershell
Get-EXORecipient -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,EmailAddresses

Get-EXOMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,ExchangeGuid
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Get-EXORecipient -Identity` | Viser mottakertypen i Exchange Online; forventet er `UserMailbox` eller spesialtypen, ikke lenger `MailUser` |
| `Get-EXOMailbox -Identity` | Returnerer bare ekte skypostbokser; et treff beviser at klargjøringen er fullført |
| `Format-List ...,ExchangeGuid` | Viser kontrollattributtene; `ExchangeGuid` identifiserer den nye skypostboksen entydig |

</details>

Batchen anses først som fullført når følgende tester i tillegg er vellykkede:

- Levering eksternt og internt
- Sending eksternt og internt
- Svar på en gammel melding for å kontrollere X500-adressen
- Innlogging med Outlook og Outlook på nettet
- Delegeringer og Send-As
- Videresendinger og transportregler
- Rom- og utstyrsbestillinger
- Applikasjoner, skannere og SMTP-reléer
- Message Trace med levering til den nye skypostboksen

## Tilbakeføring og opprydding

Den lokale kildepostboksen må ikke slettes med `Remove-StoreMailbox` i valideringsfasen. Så lenge den finnes som disconnected mailbox innenfor postboksretentionen, finnes det fortsatt en teknisk mulighet for tilbakeføring. En tilbakeføring krever imidlertid en kontrollert reversering av RemoteMailbox-attributtene og ny tilkobling av den lokale postboksen; samtidig må det forhindres at to aktive leveringsmål oppstår.

Før en tilbakeføring må derfor e-postflyt, synkroniseringsstatus og meldinger som allerede er mottatt i skyen, sikres. Et tilbakebytte er ikke en enkel énløser og må inngå som et testet runbook i endringen.

Etter vellykket godkjenning ryddes eksportforespørsler opp, PST-filer arkiveres i henhold til beskyttelses- og oppbevaringskonseptet, og midlertidige tillatelser på eksportdelingen fjernes:

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Remove-MailboxExportRequest -Confirm:$false
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Get-MailboxExportRequest -BatchName` | Velger nøyaktig eksportforespørslene for den fullførte batchen |
| `Remove-MailboxExportRequest -Confirm:$false` | Fjerner forespørslene uten spørsmål; selve PST-filene påvirkes ikke av dette |

</details>

De frakoblede postboksene bør først ryddes endelig etter at det avtalte tilbakeføringsvinduet er utløpt og i henhold til retention-konseptet.

## Konklusjon

Når Hybrid Remote Moves ikke er tillatt og ingen postboksdata må overføres til Exchange Online, kan en eksisterende synkronisert AD-bruker kontrollerte omstilles fra en lokal postboks til en ny skypostboks. Den kritiske delen er ikke `Enable-RemoteMailbox`, men prosesskontrollen rundt det: fullstendig inventarisering, verifisert PST-sikkerhetskopi, eksplisitte godkjenninger, bevaring av proxy- og X500-adresser, kontrollert e-postflyt og et reelt tilbakeføringsvindu.

## Kilder

1. [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchange/enable-remotemailbox?view=exchange-ps): Aktiverer en eksisterende lokal AD-bruker for en postboks i den skybaserte tjenesten og dokumenterer bryterne for bruker-, delte, rom- og utstyrspostbokser.

2. [Microsoft Learn – New-MailboxExportRequest](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-mailboxexportrequest?view=exchange-ps): Referanse for eksport av primære og arkiverte lokale postbokser til PST-filer.

3. [Microsoft Learn – Mailbox import and export requests](https://learn.microsoft.com/en-us/exchange/mailbox-import-and-export-requests-exchange-2013-help): Forutsetninger for eksportdeling, tillatelser og Exchange Trusted Subsystem.

4. [Microsoft Learn – Disable or delete a mailbox in Exchange Server](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disable-or-delete-mailboxes): Virkemåte for `Disable-Mailbox`, fjerning av Exchange-attributter og oppbevaring av den frakoblede postboksen.

5. [Microsoft Learn – Disconnected mailboxes](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disconnected-mailboxes): Ny tilkobling, gjenoppretting og endelig sletting av frakoblede postbokser.

6. [Microsoft Learn – Delays in provisioning of a user or mailbox](https://learn.microsoft.com/en-us/troubleshoot/exchange/user-and-shared-mailboxes/delays-provision-mailbox-sync-changes): Typisk varighet og feilsøking ved klargjøring i Exchange Online.

7. [Microsoft Learn – Move mailboxes between on-premises and Exchange Online organizations](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes): Den ordinære Hybrid Remote Move som referanse og avgrensning mot nyopprettingen beskrevet her.
