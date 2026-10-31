---
title: "EXO-migrering uten Remote Move"
navTitle: "EXO-migrering uten Remote Move"
description: "Slik klargjøres lokale Exchange-postbokser kontrollert som nye, tomme postbokser i Exchange Online: PST-sikkerhetskopi, CSV-godkjenning, RemoteMailbox, synkronisering, validering og rollback."
date: "2026-08-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "12 min lesetid"
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
translationSourceHash: 8ef7108bef25cbfdabbdd58d6ed17a1cc34ba2da65eb7e685180b0976f99d62c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:35:13.068Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/exo-migrering-uten-remote-move
---

# EXO-migrering uten Remote Move

En Hybrid Remote Move er den vanlige måten å flytte en lokal Exchange-postboks med innhold til Exchange Online på. Ikke alle organisasjoner tillater denne migreringsveien. Hvis en sikkerhetspolicy utelukker Remote Moves, kan en bevisst annerledes tilnærming være forsvarlig: Den lokale postboksen sikkerhetskopieres som PST, kobles fra den synkroniserte AD-brukeren, og en ny, tom postboks i Exchange Online klargjøres for samme bruker.

Denne veien er **ikke en postboksmigrering**. Den overfører verken meldinger, kalendere, regler eller tillatelser til skyen. PST-en brukes utelukkende som sikkerhetskopi og importeres ikke i dette scenariet. Fremgangsmåten egner seg derfor bare når en tom målpostboks er faglig akseptabel, og tap av den aktive postboks-konfigurasjonen uttrykkelig er godkjent.

## Måltilstand og ufravikelige forutsetninger

Etter cutover beholdes samme AD-bruker. Lokalt administreres den imidlertid ikke lenger som `UserMailbox`, `SharedMailbox`, `RoomMailbox` eller `EquipmentMailbox`, men som `RemoteMailbox`. Etter synkroniseringen representerer dette objektet den nye postboksen i Exchange Online.

Ønsket tilstand er som følger:

1. Den lokale postboksen er fullstendig sikkerhetskopiert som PST.
2. Den lokale postboksen er frakoblet, men ennå ikke endelig slettet innenfor konfigurert oppbevaringstid.
3. Den eksisterende AD-brukeren er aktivert som RemoteMailbox.
4. Primæradresse, aliaser og den gamle `LegacyExchangeDN` beholdes.
5. Entra Connect har synkronisert endringene.
6. For brukerpostbokser er en Exchange Online-tjenesteplan tilordnet.
7. Exchange Online viser en ekte skypostboks, og e-postflyten ender der.

Før oppstart må dessuten følgende avklares:

- PST-delingsområdet er tilgjengelig via UNC. Gruppen `Exchange Trusted Subsystem` har lese- og skrivetillatelser der.
- Kontoen som utfører arbeidet, har administrasjonsrollen `Mailbox Import Export`.
- PST-en er kun den avtalte sikkerhetskopien; ingen senere import er planlagt.
- Oppbevaring, Litigation Hold, eDiscovery og regulatoriske krav er kontrollert separat.
- Delegeringer, Send-As, Send-on-Behalf, videresendinger, innboksregler, mobile enheter og applikasjonstilganger er kartlagt.
- Under eksport og omkobling holdes innkommende meldinger kontrollert tilbake i den foranliggende gatewayen. Brukere og applikasjoner må ikke lenger skrive til kildepostboksen.
- Oppbevaringstiden for den lokale postboksdatabasen dekker rollback-vinduet.

## Hvorfor en CSV-godkjenningsliste er uunnværlig

En direkte pipeline som `Get-Mailbox | Disable-Mailbox` er for risikabel for denne prosessen. Den kan også omfatte system-, discovery- eller andre ikke-godkjente postbokser. Den følgende prosessen arbeider derfor med to eksplisitte godkjenninger:

- `Action=CUTOVER` angir hvilken rad som faktisk kan omkobles.
- `PstVerified=YES` bekrefter at eksportfilen er teknisk og organisatorisk kontrollert.

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

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Get-Mailbox -ResultSize Unlimited` | Opphever standardgrensen på 1000 resultater; uten denne parameteren mangler postbokser i store miljøer i inventaret |
| `-RecipientTypeDetails UserMailbox,SharedMailbox,RoomMailbox,EquipmentMailbox` | Begrenser spørringen til de fire postbokstypene som skal omkobles; system- og discovery-postbokser utelates |
| `Sort-Object PrimarySmtpAddress` | Sorterer utdataene etter primær SMTP-adresse, slik at CSV-filen er stabilt sortert ved faglig gjennomgang |
| `Export-Csv -Path` | Målbane for CSV-filen |
| `-NoTypeInformation` | Undertrykker typeoverskriften `#TYPE ...`, som eldre PowerShell-versjoner ellers skriver som første linje |
| `-Encoding UTF8` | Skriver filen med UTF-8-koding, slik at spesialtegn i visningsnavn bevares korrekt |

</details>

Filen ryddes deretter faglig. Bare faktisk godkjente postbokser får `Action=CUTOVER`. Systempostbokser og spesialobjekter hører ikke hjemme i denne listen.

## Fase 1: Sikkerhetskopier primærpostboks og arkiv som PST

`New-MailboxExportRequest` skriver bare til en UNC-bane. Det opprettes et entydig filnavn for hver postboks. Et aktivt onlinearkiv i lokal Exchange eksporteres separat:

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
| `Where-Object Action -eq "CUTOVER"` | Behandler kun eksplisitt godkjente rader |
| `New-MailboxExportRequest -Mailbox` | Kildepostboks for eksporten (her identiteten fra CSV-raden) |
| `-FilePath` | Målbane for PST-filen; må være en UNC-bane, siden cmdleten avviser lokale baner |
| `-Name` | Entydig navn på forespørselen; muliggjør senere målrettet tilordning av primær- og arkiveksport |
| `-BatchName` | Samler alle forespørsler i en kjøring under et batchnavn; grunnlag for statuskontroll og opprydding |
| `-IsArchive` | Eksporterer onlinearkivet i stedet for primærpostboksen; derfor den andre forespørselen per postboks med aktivt arkiv |

</details>

Eksporten er først godkjent når **alle** forespørsler har statusen `Completed`:

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
| `Format-Table ... -AutoSize` | Tabellvisning av de angitte egenskapene; `-AutoSize` tilpasser kolonnebreddene til innholdet |
| `Where-Object Status -ne "Completed"` | Filtrerer på alle forespørsler som ennå ikke er fullført eller har feilet; utdataene må være tomme før man fortsetter |

</details>

I tillegg må filenes eksistens, størrelse, lesbarhet, overføring til sikkerhetskopi og tilgangsbeskyttelse kontrolleres. Først deretter settes `PstVerified=YES` for den aktuelle CSV-raden.

## Fase 2: Sikkerhetskopier postboksdata og Exchange-attributter

Før den første endringen opprettes et maskinlesbart øyeblikksbilde for hver postboks. Det er viktigere enn et skjermbilde, fordi aliaser, GUID-er og `LegacyExchangeDN` senere kan rekonstrueres nøyaktig:

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
| `New-Item -ItemType Directory -Path ... -Force` | Oppretter katalogen for øyeblikksbilder; `-Force` undertrykker feilen dersom den allerede finnes |
| `Get-Mailbox -Identity` | Henter det aktuelle postboksobjektet for den respektive CSV-raden |
| `Select-Object Identity,...,ServerName` | Reduserer objektet til attributtene som kreves for senere rekonstruksjon (GUID-er, adresser, `LegacyExchangeDN`, database) |
| `Export-Clixml` | Serialiserer objektet som typesbevarende CLIXML; i motsetning til CSV bevares flerverdier som `EmailAddresses` fullstendig og kan leses inn igjen med `Import-Clixml` |

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
| `Format-List ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward` | Viser postboksens tre videresendingsattributter i listeformat |
| `Get-MailboxPermission -Identity` | Viser postbokstillatelser som Full Access |
| `Get-ADPermission -Identity` | Viser AD-tillatelser på brukerobjektet, herunder Send-As; forventer her Distinguished Name |
| `Get-InboxRule -Mailbox` | Viser postboksens serverbaserte innboksregler |
| `Get-CalendarProcessing -Identity` | Viser bestillingskonfigurasjonen; relevant for rom- og utstyrspostbokser |
| `-ErrorAction SilentlyContinue` | Undertrykker feilen for postbokstyper uten bestillingskonfigurasjon, slik at sikkerhetskopieringen ikke avbrytes |

</details>

Disse konfigurasjonene overføres ikke automatisk til den nye skypostboksen.

## Fase 3: Koble fra lokal postboks og aktiver RemoteMailbox

Selve cutoveren er kort, men får vidtrekkende konsekvenser. `Disable-Mailbox` fjerner Exchange-attributtene fra AD-brukeren og kobler fra den lokale postboksen. Postboksdataene beholdes som disconnected mailbox til databaseretensjonen utløper. Umiddelbart etterpå aktiverer `Enable-RemoteMailbox` samme AD-bruker for Exchange Online.

Det følgende skriptet behandler kun dobbelt godkjente rader. Det bevarer den primære SMTP-adressen, alle eksisterende proxy-adresser og den gamle `LegacyExchangeDN` som X500-adresse. X500-oppføringen forhindrer NDR-er ved svar på eldre meldinger eller ved gamle Outlook-autofullføringsoppføringer.

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
| `Get-Mailbox -Identity ... -ErrorAction Stop` | Henter kildepostboksen; `-ErrorAction Stop` gjør en oppslagsfeil til en terminerende feil, i stedet for å fortsette stille |
| `Disable-Mailbox -Identity` | Fjerner Exchange-attributtene fra AD-brukeren og kobler fra den lokale postboksen; dataene blir værende som disconnected mailbox i databasen |
| `-Confirm:$false` | Undertrykker den interaktive forespørselen; godkjenningen skjer her via CSV-listen, ikke i ledeteksten |
| `Enable-RemoteMailbox -Identity` | Aktiverer samme AD-bruker som RemoteMailbox for Exchange Online |
| `-Alias` | Setter Exchange-aliaset tilbake til verdien fra godkjenningslisten |
| `-PrimarySmtpAddress` | Beholder den tidligere primære SMTP-adressen |
| `-RemoteRoutingAddress` | Måladresse i `mail.onmicrosoft.com`-rutingsdomenet, som lokal Exchange bruker for å nå skypostboksen |
| `-ACLableSyncedObjectEnabled` | Merker objektet som ACL-kompatibelt, slik at tillatelser som Full Access fortsatt kan evalueres i Exchange Online etter synkronisering |
| `-Shared` / `-Room` / `-Equipment` | Oppretter den aktuelle spesialtypen i stedet for en brukerpostboks; skriptet setter nøyaktig én bryter som passer kildetypen |
| `Set-RemoteMailbox -EmailAddressPolicyEnabled $false` | Unntar objektet fra e-postadressepolicyen, slik at den ikke overskriver de manuelt angitte adressene |
| `-EmailAddresses` | Setter den komplette, dedupliserte adresselisten, inkludert gamle proxy-adresser, rutingsadresse og X500-oppføring |
| `Get-RemoteMailbox -Identity` | Kontrollspørring av resultatet direkte etter omkoblingen |

</details>

Skriptet er bevisst ikke et helautomatisk migreringsverktøy. Det avslutter kjøringen ved første avvik, slik at en administrator kan vurdere årsak og tilstand. Før en produksjonsbatch bør koden valideres med noen få testpostbokser og Exchange-versjonene som er i bruk.

## Fase 4: Synkroniser, lisensier og verifiser

Etter den lokale endringen startes en delta-syklus på Entra Connect-serveren:

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-PolicyType Delta` | Synkroniserer kun objekter som er endret siden forrige syklus; alternativet `Initial` ville være en fullstendig og betydelig lengre kjøring |

</details>

For brukerpostbokser må det deretter være tilordnet en gyldig Exchange Online-tjenesteplan, for eksempel via gruppebasert lisensiering. Delte postbokser, rompostbokser og utstyrspostbokser må vurderes i henhold til gjeldende Microsoft-lisensvilkår og nødvendige funksjoner.

Klargjøringen er asynkron. Microsoft angir vanligvis under 30 minutter for normale endringer, men i enkelte tilfeller opptil 24 timer. I denne perioden bør den foranliggende e-postflyten holde meldinger kontrollert tilbake, i stedet for å levere dem til et mål som ennå ikke er klart.

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
| `Get-RemoteMailbox -Identity` | Må returnere det omkoblede objektet som RemoteMailbox |
| `Format-List RecipientTypeDetails,...` | Viser type, adresser og rutingsadresse for kontroll i listeformat |
| `Get-Mailbox -Identity ... -ErrorAction SilentlyContinue` | Motkontroll: Kallet må ikke returnere noe, ettersom det ikke lenger finnes en tilkoblet lokal postboks; `-ErrorAction SilentlyContinue` undertrykker den forventede feilmeldingen |

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
| `Get-EXOMailbox -Identity` | Returnerer kun ekte skypostbokser; et treff beviser fullført klargjøring |
| `Format-List ...,ExchangeGuid` | Viser kontrollattributtene; `ExchangeGuid` identifiserer den nye skypostboksen entydig |

</details>

Batchen anses først som fullført når også følgende tester er vellykkede:

- Levering fra eksternt og internt
- Sending til eksternt og internt
- Svar på en gammel melding for å kontrollere X500-adressen
- Pålogging med Outlook og Outlook på nettet
- Delegeringer og Send-As
- Videresendinger og transportregler
- Rom- og utstyrsbestillinger
- Applikasjoner, skannere og SMTP-reléer
- Meldingssporing med levering til den nye skypostboksen

## Rollback og opprydding

Den lokale kildepostboksen må ikke slettes med `Remove-StoreMailbox` i valideringsfasen. Så lenge den finnes som disconnected mailbox innenfor postboksoppbevaringstiden, finnes det fortsatt en teknisk tilbakefallsmulighet. En rollback krever imidlertid en kontrollert reversering av RemoteMailbox-attributtene og ny tilkobling av den lokale postboksen; samtidig må det forhindres at to aktive leveringsmål oppstår.

Før rollback må derfor e-postflyt, synkroniseringsstatus og meldinger som allerede er mottatt i skyen, sikkerhetskopieres. En tilbakevending er ikke en enkel énlinskommando og må inngå som et testet runbook i endringen.

Etter vellykket godkjenning ryddes eksportforespørslene, PST-filer arkiveres i henhold til beskyttelses- og oppbevaringskonseptet, og midlertidige tillatelser på eksportdelingen fjernes:

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Remove-MailboxExportRequest -Confirm:$false
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Get-MailboxExportRequest -BatchName` | Velger nøyaktig eksportforespørslene for den fullførte batchen |
| `Remove-MailboxExportRequest -Confirm:$false` | Fjerner forespørslene uten spørsmål; selve PST-filene berøres ikke |

</details>

De frakoblede postboksene bør først ryddes endelig etter utløpet av det avtalte rollback-vinduet og i samsvar med oppbevaringskonseptet.

## Konklusjon

Når Hybrid Remote Moves ikke er tillatt og ingen postboksdata må overføres til Exchange Online, kan en eksisterende synkronisert AD-bruker kontrollert omkobles fra en lokal postboks til en ny skypostboks. Den kritiske delen er ikke `Enable-RemoteMailbox`, men prosesskontrollen rundt den: fullstendig inventarisering, verifisert PST-sikkerhetskopi, eksplisitte godkjenninger, bevaring av proxy- og X500-adresser, kontrollert e-postflyt og et reelt rollback-vindu.

## Endringslogg

- **25. oktober 2026:** Kildelenke oppdatert til den nye adressen på Microsoft Learn.

## Kilder

1. [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox?view=exchange-ps): Aktiverer en eksisterende lokal AD-bruker for en postboks i den skybaserte tjenesten og dokumenterer bryterne for bruker-, delte-, rom- og utstyrspostbokser.

2. [Microsoft Learn – New-MailboxExportRequest](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-mailboxexportrequest?view=exchange-ps): Referanse for eksport av primære og arkiverte lokale postbokser til PST-filer.

3. [Microsoft Learn – Mailbox import and export requests](https://learn.microsoft.com/en-us/exchange/mailbox-import-and-export-requests-exchange-2013-help): Forutsetninger for eksportdeling, tillatelser og Exchange Trusted Subsystem.

4. [Microsoft Learn – Disable or delete a mailbox in Exchange Server](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disable-or-delete-mailboxes): Virkemåte for `Disable-Mailbox`, fjerning av Exchange-attributtene og oppbevaring av den frakoblede postboksen.

5. [Microsoft Learn – Disconnected mailboxes](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disconnected-mailboxes): Koble til på nytt, gjenopprette og endelig slette frakoblede postbokser.

6. [Microsoft Learn – Delays in provisioning of a user or mailbox](https://learn.microsoft.com/en-us/troubleshoot/exchange/user-and-shared-mailboxes/delays-provision-mailbox-sync-changes): Typisk varighet og feilsøking ved klargjøring i Exchange Online.

7. [Microsoft Learn – Move mailboxes between on-premises and Exchange Online organizations](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes): Den ordinære Hybrid Remote Move som referanse og avgrensning mot den nye oppbyggingen som beskrives her.
