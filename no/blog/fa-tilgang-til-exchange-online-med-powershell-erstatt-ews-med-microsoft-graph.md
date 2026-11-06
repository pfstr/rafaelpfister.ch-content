---
title: "Få tilgang til Exchange Online med PowerShell: Erstatt EWS med Microsoft Graph"
navTitle: "Graph i PowerShell"
description: "Microsoft avvikler EWS i Exchange Online trinnvis fra oktober 2026, og fullstendig i april 2027. Slik registrerer du en app, autentiserer et PowerShell-skript med et sertifikat, begrenser tilgangen til bestemte postbokser og behandler meldinger og vedlegg via Microsoft Graph."
date: "2026-07-11"
kategorie: "Totemomail"
timeToRead: "5 min lesetid"
themen:
  - microsoft-365-exchange
slug: "fa-tilgang-til-exchange-online-med-powershell-erstatt-ews-med-microsoft-graph"
translationOf: "microsoft-graph-powershell-postfach-anbindung"
translationId: article-4c6a02c79b7bf0fe
translationReview: automatic
translationSourceHash: 38a149cf8f2933d92be1702594f7e41492b0a1924a56a20808172cc32be4a498
translatedAt: 2026-11-06T03:21:40.887Z
translationModel: gpt-5.6-terra
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/fa-tilgang-til-exchange-online-med-powershell-erstatt-ews-med-microsoft-graph
---

# Få tilgang til Exchange Online med PowerShell: Erstatt EWS med Microsoft Graph

Microsoft avvikler Exchange Web Services (EWS) i Exchange Online trinnvis fra **oktober 2026**; i **april 2027** er EWS fullstendig deaktivert. Skript som henter meldinger eller vedlegg fra en postboks, må derfor gå over til Microsoft Graph.

Eksemplet i denne artikkelen kjører uten brukerinnlogging: Det laster ned ZIP-vedlegg fra en postboks, pakker dem ut, flytter behandlede meldinger og sender deretter en rapport. Dette krever en appregistrering, et sertifikat og to Graph-tillatelser. Verdier som `example.com`, tenant-ID og app-ID er plassholdere.

## 1. Nødvendige PowerShell-moduler

Tre moduler fra Microsoft Graph SDK er tilstrekkelig, ikke hele metamodulen `Microsoft.Graph`:

```powershell
Install-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions -Scope AllUsers
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions` | De tre nødvendige delmodulene som posisjonsargument for `-Name`: innlogging, e-post-cmdleter og handlinger som `Send-MgUserMail` |
| `-Scope AllUsers` | Installerer modulene for hele maskinen under `Program Files`; nødvendig for at de også skal være tilgjengelige for tjenestekontoen til Scheduled Task som konfigureres senere (krever administratorrettigheter) |

</details>

## 2\. Appregistrering i Entra ID

Et uovervåket skript logger inn som en applikasjon med egne rettigheter. Opprett en ny registrering i [Entra Admin Center](https://entra.microsoft.com) under **App registrations**, og tildel disse to rettighetene under **API permissions → Microsoft Graph → Application permissions**:

-   `Mail.ReadWrite`: lese e-post og flytte den etter behandling
    
-   `Mail.Send`: sende rapport-e-posten
    
Gi deretter administratorsamtykke, og noter tenant-ID samt Application (Client) ID.

## 3\. Sertifikat i stedet for Client Secret

En app-only-innlogging fungerer med Client Secret eller sertifikat. For planlagte oppgaver er et sertifikat det beste valget: Den private nøkkelen forblir i sertifikatlageret, og skriptet inneholder ikke noe passord. Opprett sertifikatet på serveren som skal kjøre skriptet, og eksporter bare den offentlige delen:

```powershell
$cert = New-SelfSignedCertificate -Subject "CN=eCall-Graph" `
    -CertStoreLocation "Cert:\LocalMachine\My" `
    -KeyExportPolicy NonExportable -KeySpec Signature `
    -KeyLength 2048 -NotAfter (Get-Date).AddYears(2)

Export-Certificate -Cert $cert -FilePath .\eCall-Graph.cer
$cert.Thumbprint   # -> im Skript als Thumbprint verwenden
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `New-SelfSignedCertificate -Subject` | Sertifikatets emnenavn; brukes bare for gjenkjenning i sertifikatlageret |
| `-CertStoreLocation "Cert:\LocalMachine\My"` | Lagrer sertifikatet i maskinlageret, ikke i brukerlageret; dermed er det tilgjengelig uavhengig av innlogget bruker |
| `-KeyExportPolicy NonExportable` | Hindrer eksport av den private nøkkelen; den forlater ikke serveren |
| `-KeySpec Signature` | Oppretter en signeringsnøkkel; appinnloggingen bruker den til å signere token-request-assertion |
| `-KeyLength 2048` | RSA-nøkkellengde på 2048 bit |
| `-NotAfter (Get-Date).AddYears(2)` | Utløper om to år; deretter må sertifikatet fornyes og lastes opp på nytt |
| `Export-Certificate -Cert` | Sertifikatobjektet som skal eksporteres |
| `-FilePath` | Målfil; inneholder kun den offentlige delen som `.cer` |

</details>

Last opp den eksporterte `.cer`\-filen i appregistreringen under Certificates & secrets. Kontoen for Scheduled Task trenger lesetilgang til den private nøkkelen (`certlm.msc` → sertifikat → All Tasks → Manage Private Keys).

## 4\. Begrens tilgang til bestemte postbokser

Application Permissions gjelder i utgangspunktet alle postbokser i tenant-en. Begrens derfor appen med en Application Access Policy til en e-postaktivert sikkerhetsgruppe. Dette trinnet utføres én gang i Exchange Online PowerShell. Microsoft omtaler nå RBAC for Applications som erstatning for Application Access Policies; der fastsettes virkeområdet gjennom en Management Scope eller en Administrative Unit, og tillatelsen gis i Exchange Online i stedet for Entra ID.

```powershell
New-ApplicationAccessPolicy -AppId "<App-ID>" `
    -PolicyScopeGroupId "graph-mailboxes@example.com" `
    -AccessRight RestrictAccess `
    -Description "eCall Graph: nur Log-Postfach"

# Kontroller virkning
Test-ApplicationAccessPolicy -AppId "<App-ID>" -Identity "ecall-logs@example.com"
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `New-ApplicationAccessPolicy -AppId` | Application (Client) ID for appregistreringen som policyen gjelder for |
| `-PolicyScopeGroupId` | E-postaktivert sikkerhetsgruppe der medlemmene definerer virkeområdet |
| `-AccessRight RestrictAccess` | Begrenses appen til gruppens postbokser; alternativet `DenyAccess` ville sperret nettopp disse postboksene |
| `-Description` | Fritekst for dokumentasjon av policyen |
| `Test-ApplicationAccessPolicy -Identity` | Kontrollerer for en bestemt postboks om appen har tilgang til den (`AccessCheckResult: Granted` eller `Denied`) |

</details>

## 5\. Opprett forbindelse

Innloggingen bruker tenant-ID, app-ID og sertifikatets thumbprint, helt uten brukerinteraksjon:

```powershell
$TenantId   = "00000000-0000-0000-0000-000000000000"
$ClientId   = "00000000-0000-0000-0000-000000000000"
$Thumbprint = "0000000000000000000000000000000000000000"
$Mailbox    = "ecall-logs@example.com"

Import-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions
Connect-MgGraph -TenantId $TenantId -ClientId $ClientId `
    -CertificateThumbprint $Thumbprint -NoWelcome
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Connect-MgGraph -TenantId` | Tenant-en appen logger inn mot |
| `-ClientId` | Application (Client) ID for appregistreringen |
| `-CertificateThumbprint` | Velger innloggingssertifikatet fra det lokale sertifikatlageret via thumbprint; kombinasjonen av `-ClientId` og sertifikat gir en app-only-innlogging uten bruker |
| `-NoWelcome` | Undertrykker velkomstteksten etter innlogging; nyttig for skriptutdata og logger |

</details>

## 6. Les meldinger og last ned ZIP-vedlegg

Skriptet kan nå gå gjennom innboksen, lagre og pakke ut ZIP-vedlegg samt flytte behandlede meldinger til «Slettede elementer». Nedlastingen skjer via endepunktet `/$value` og `Invoke-MgGraphRequest -OutputFilePath`. Dermed havner råinnholdet direkte i en fil, uten at et stort vedlegg må holdes helt i arbeidsminnet:

```powershell
Add-Type -AssemblyName System.IO.Compression.FileSystem
$Zielordner = "D:\Import\{0:yyyyMMdd_HHmmss}" -f (Get-Date)

$messages = Get-MgUserMessage -UserId $Mailbox -Top 100 `
    -Property id, subject, hasAttachments

foreach ($msg in $messages) {
    $ordner = Join-Path $Zielordner $msg.Id
    New-Item -Path $ordner -ItemType Directory -Force | Out-Null

    $anhaenge = Get-MgUserMessageAttachment -UserId $Mailbox -MessageId $msg.Id |
        Where-Object {
            $_.AdditionalProperties['@odata.type'] -eq '#microsoft.graph.fileAttachment' -and
            $_.Name -like '*.zip'
        }

    foreach ($att in $anhaenge) {
        $zip = Join-Path $ordner $att.Name
        $uri = "https://graph.microsoft.com/v1.0/users/$Mailbox/messages/$($msg.Id)/attachments/$($att.Id)/`$value"
        Invoke-MgGraphRequest -Method GET -Uri $uri -OutputFilePath $zip
        [System.IO.Compression.ZipFile]::ExtractToDirectory($zip, $ordner)
    }

    # flytt behandlet e-post til «Slettede elementer»
    Move-MgUserMessage -UserId $Mailbox -MessageId $msg.Id -DestinationId "deleteditems" | Out-Null
}
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Add-Type -AssemblyName System.IO.Compression.FileSystem` | Laster .NET-assemblyen med klassen `ZipFile` for utpakking |
| `Get-MgUserMessage -UserId` | Postboksen som meldingene leses fra; ved app-only-innlogging er denne angivelsen obligatorisk |
| `-Top 100` | Begrenser forespørselen til maksimalt 100 meldinger per kall |
| `-Property id, subject, hasAttachments` | Ber bare om de nødvendige feltene; dette reduserer svaret og gjør kallet raskere |
| `Get-MgUserMessageAttachment -MessageId` | Melding der vedleggene listes opp |
| `Invoke-MgGraphRequest -Method GET` | HTTP-metode for direktekallet mot Graph-API-et |
| `-Uri` | Endepunktet som kalles; den vedlagte `/$value` leverer det rå filinnholdet i vedlegget i stedet for et JSON-objekt |
| `-OutputFilePath` | Skriver svaret direkte til målfilen uten å holde hele vedlegget i arbeidsminnet |
| `Move-MgUserMessage -DestinationId "deleteditems"` | Flytter den behandlede meldingen til målmappen; `deleteditems` er det velkjente mappenavnet for «Slettede elementer» |

</details>

Ved mer enn 100 e-poster: arbeid med `Get-MgUserMessage -All` eller paging; for en månedlig kjøring er én batch som regel nok.

## 7\. Send rapport-e-post via Graph

Også `Send-MailMessage` er utdatert. Med samme appregistrering (rettighet `Mail.Send`) sendes e-posten direkte via Graph, her med en fil som base64-kodet vedlegg:

```powershell
$pfad = "D:\Reports\report.csv"
$body = @{
    message = @{
        subject = "eCall Report"
        body    = @{ contentType = "HTML"; content = "<b>Lauf erfolgreich</b>" }
        toRecipients = @(@{ emailAddress = @{ address = "empfaenger@example.com" } })
        attachments  = @(@{
            "@odata.type" = "#microsoft.graph.fileAttachment"
            name          = Split-Path $pfad -Leaf
            contentBytes  = [Convert]::ToBase64String([IO.File]::ReadAllBytes($pfad))
        })
    }
    saveToSentItems = $true
}
Send-MgUserMail -UserId "reporting@example.com" -BodyParameter $body
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-UserId` | Postboksen e-posten sendes på vegne av; den må dekkes av virkeområdet til Application Access Policy |
| `-BodyParameter` | Hele meldingen som Hashtable i Graph-skjemaet: `message` med emne, brødtekst, mottakere og vedlegg samt `saveToSentItems` for lagring i «Sendte elementer» |

</details>

## 8\. Kjør uovervåket

Som planlagt oppgave kjører skriptet uten innlogging, fordi sertifikatet ligger i kontoens lager:

```powershell
$action  = New-ScheduledTaskAction -Execute "powershell.exe" `
    -Argument '-NoProfile -ExecutionPolicy Bypass -File "D:\Scripts\graph-import.ps1"'
$trigger = New-ScheduledTaskTrigger -Daily -At 06:00
Register-ScheduledTask -TaskName "eCall-Graph-Import" -Action $action -Trigger $trigger `
    -User "DOMAIN\svc-ecall" -Password (Read-Host "Passwort")
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `New-ScheduledTaskAction -Execute` | Programmet som skal kjøres, her `powershell.exe` |
| `-Argument` | Kommandolinje for programmet: `-NoProfile` hopper over profilskript, `-ExecutionPolicy Bypass` omgår skriptpolicyen for dette kallet, `-File` angir skriptet |
| `New-ScheduledTaskTrigger -Daily -At 06:00` | Daglig utløser kl. 06:00 |
| `Register-ScheduledTask -TaskName` | Navn på oppgaven i Oppgaveplanlegging |
| `-Action` / `-Trigger` | Knytter den tidligere opprettede handlingen og utløseren til oppgaven |
| `-User` | Kontoen oppgaven kjører under; den private nøkkelen må være lesbar i sertifikatlageret |
| `-Password (Read-Host "Passwort")` | Ber interaktivt om kontopassordet slik at oppgaven også kan starte uten innlogget bruker; det havner dermed ikke i skriptet eller historikkfilen |

</details>

Det fullstendige eksemplet med logging og feilhåndtering finnes på GitHub: [pfstr/ecall-log-analyzer](https://github.com/pfstr/ecall-log-analyzer).

## Endringslogg

- **25. oktober 2026:** Korrigert: EWS i Exchange Online avvikles trinnvis fra oktober 2026 og deaktiveres først fullstendig i april 2027. Lagt til: RBAC for Applications som Microsofts erstatning for Application Access Policies, med ny kilde og oppdatert tidsplan.

## Kilder

1.  [Microsoft – «Retirement of Exchange Web Services in Exchange Online»](https://techcommunity.microsoft.com/blog/exchange/retirement-of-exchange-web-services-in-exchange-online/3924440): Kunngjøring fra 2023 med den opprinnelige fristen oktober 2026 for slutten på EWS i Exchange Online.
    
2.  [Microsoft Learn – «Get access without a user (App-only)»](https://learn.microsoft.com/en-us/graph/auth-v2-service): App-only-autentisering mot Microsoft Graph med sertifikat.
    
3.  [Microsoft Learn – «Role Based Access Control for Applications in Exchange Online»](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac): Begrensning av en app til bestemte postbokser; RBAC for Applications erstatter Application Access Policies, inkludert migrasjonstrinn.
    
4.  [Microsoft Learn – «Deprecation of Exchange Web Services in Exchange Online»](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/deprecation-of-ews-exchange-online): Oppdatert tidsplan, trinnvis avvikling fra oktober 2026 og fullstendig deaktivering i april 2027.
