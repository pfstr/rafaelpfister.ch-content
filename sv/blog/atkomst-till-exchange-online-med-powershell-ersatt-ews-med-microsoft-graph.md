---
title: "Åtkomst till Exchange Online med PowerShell: ersätt EWS med Microsoft Graph"
navTitle: "Graph i PowerShell"
description: "EWS upphör i Exchange Online den 1 oktober 2026. Så registrerar du en app, autentiserar ett PowerShell-skript med ett certifikat, begränsar åtkomsten till enskilda postlådor och hanterar meddelanden och bilagor via Microsoft Graph."
date: "2026-07-11"
kategorie: "Totemomail"
timeToRead: "5 min läsning"
themen:
  - microsoft-365-exchange
slug: "atkomst-till-exchange-online-med-powershell-ersatt-ews-med-microsoft-graph"
translationOf: "microsoft-graph-powershell-postfach-anbindung"
translationId: article-4c6a02c79b7bf0fe
translationReview: automatic
translationSourceHash: 66e214f25e8088562270157199f9ff55fcd9828362abf145170a12f287cd0f6c
translatedAt: 2026-09-05T07:56:32.501Z
url: https://rafaelpfister.ch/sv/blog/atkomst-till-exchange-online-med-powershell-ersatt-ews-med-microsoft-graph
translationModel: gpt-5.6-terra
---

# Åtkomst till Exchange Online med PowerShell: ersätt EWS med Microsoft Graph

Microsoft avvecklar Exchange Web Services (EWS) i Exchange Online den **1 oktober 2026**. Skript som hämtar meddelanden eller bilagor från en postlåda måste därför övergå till Microsoft Graph.

Exemplet i den här artikeln körs utan användarinloggning: det laddar ned ZIP-bilagor från en postlåda, packar upp dem, flyttar bearbetade meddelanden och skickar sedan en rapport. För detta behövs en appregistrering, ett certifikat och två Graph-behörigheter. Värden som `example.com`, klient-ID och app-ID är platshållare.

## 1. Nödvändiga PowerShell-moduler

Det räcker med tre moduler från Microsoft Graph SDK, inte hela metamodulen `Microsoft.Graph`:

```powershell
Install-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions -Scope AllUsers
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions` | De tre nödvändiga delmodulerna som positionsargument `-Name`: inloggning, e-post-cmdlets och åtgärder som `Send-MgUserMail` |
| `-Scope AllUsers` | Installerar modulerna systemomfattande under `Program Files`; krävs för att de även ska vara tillgängliga för det tjänstkonto som senare konfigureras för Scheduled Task (kräver administratörsbehörighet) |

</details>

## 2\. Appregistrering i Entra ID

Ett obevakat skript loggar in som en applikation med egna behörigheter. Skapa en ny registrering i [Entra Admin Center](https://entra.microsoft.com) under **App registrations** och tilldela dessa två behörigheter under **API permissions → Microsoft Graph → Application permissions**:

-   `Mail.ReadWrite`: läsa e-post och flytta den efter bearbetning
    
-   `Mail.Send`: skicka rapportmeddelandet
    
Bevilja sedan administratörsgodkännande och notera klient-ID samt Application (Client) ID.

## 3\. Certifikat i stället för Client Secret

En app-only-inloggning fungerar med Client Secret eller certifikat. För schemalagda uppgifter är ett certifikat det bättre valet: den privata nyckeln stannar i certifikatarkivet och skriptet innehåller inget lösenord. Skapa certifikatet på servern som kör skriptet och exportera endast den offentliga delen:

```powershell
$cert = New-SelfSignedCertificate -Subject "CN=eCall-Graph" `
    -CertStoreLocation "Cert:\LocalMachine\My" `
    -KeyExportPolicy NonExportable -KeySpec Signature `
    -KeyLength 2048 -NotAfter (Get-Date).AddYears(2)

Export-Certificate -Cert $cert -FilePath .\eCall-Graph.cer
$cert.Thumbprint   # -> im Skript als Thumbprint verwenden
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `New-SelfSignedCertificate -Subject` | Certifikatets ämnesnamn; används endast för igenkänning i certifikatarkivet |
| `-CertStoreLocation "Cert:\LocalMachine\My"` | Lagrar certifikatet i datorarkivet, inte i användararkivet; det är därmed tillgängligt oberoende av den inloggade användaren |
| `-KeyExportPolicy NonExportable` | Förhindrar export av den privata nyckeln; den lämnar inte servern |
| `-KeySpec Signature` | Skapar en signeringsnyckel; appinloggningen använder den för att signera token-request-assertion |
| `-KeyLength 2048` | RSA-nyckellängd på 2048 bitar |
| `-NotAfter (Get-Date).AddYears(2)` | Giltighetstidens slut om två år; därefter måste certifikatet förnyas och laddas upp på nytt |
| `Export-Certificate -Cert` | Certifikatobjekt som ska exporteras |
| `-FilePath` | Målfil; innehåller som `.cer` endast den offentliga delen |

</details>

Ladda upp den exporterade `.cer`\-filen i appregistreringen under Certificates & secrets. Kontot för Scheduled Task behöver läsbehörighet till den privata nyckeln (`certlm.msc` → certifikat → All Tasks → Manage Private Keys).

## 4\. Begränsa åtkomsten till enskilda postlådor

Application Permissions gäller inledningsvis för alla postlådor i klientorganisationen. Begränsa därför appen med en Application Access Policy till en e-postaktiverad säkerhetsgrupp. Detta steg körs en gång i Exchange Online PowerShell:

```powershell
New-ApplicationAccessPolicy -AppId "<App-ID>" `
    -PolicyScopeGroupId "graph-mailboxes@example.com" `
    -AccessRight RestrictAccess `
    -Description "eCall Graph: nur Log-Postfach"

# Kontrollera att den gäller
Test-ApplicationAccessPolicy -AppId "<App-ID>" -Identity "ecall-logs@example.com"
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `New-ApplicationAccessPolicy -AppId` | Application (Client) ID för den appregistrering som principen gäller |
| `-PolicyScopeGroupId` | E-postaktiverad säkerhetsgrupp vars medlemmar definierar omfattningen |
| `-AccessRight RestrictAccess` | Begränsar appen till gruppens postlådor; alternativet `DenyAccess` skulle blockera just dessa postlådor |
| `-Description` | Fritext för dokumentation av principen |
| `Test-ApplicationAccessPolicy -Identity` | Kontrollerar för en specifik postlåda om appen får åtkomst till den (`AccessCheckResult: Granted` eller `Denied`) |

</details>

## 5\. Upprätta anslutningen

Inloggningen använder klient-ID, app-ID och certifikatets tumavtryck, helt utan användarinteraktion:

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
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `Connect-MgGraph -TenantId` | Klientorganisation som appen loggar in mot |
| `-ClientId` | Application (Client) ID för appregistreringen |
| `-CertificateThumbprint` | Väljer inloggningscertifikatet via dess tumavtryck från det lokala certifikatarkivet; kombinationen av `-ClientId` och certifikat ger en app-only-inloggning utan användare |
| `-NoWelcome` | Undertrycker välkomsttexten efter inloggning; lämpligt för skriptutdata och loggar |

</details>

## 6. Läs meddelanden och ladda ned ZIP-bilagor

Nu kan skriptet gå igenom inkorgen, spara och packa upp ZIP-bilagor samt flytta bearbetade meddelanden till «Borttagna objekt». Nedladdningen sker via slutpunkten `/$value` och `Invoke-MgGraphRequest -OutputFilePath`. Därmed hamnar råinnehållet direkt i en fil utan att en stor bilaga behöver hållas helt i arbetsminnet:

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

    # flytta bearbetat e-postmeddelande till "Borttagna objekt"
    Move-MgUserMessage -UserId $Mailbox -MessageId $msg.Id -DestinationId "deleteditems" | Out-Null
}
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `Add-Type -AssemblyName System.IO.Compression.FileSystem` | Läser in .NET-assemblyn med klassen `ZipFile` för uppackning |
| `Get-MgUserMessage -UserId` | Postlådan vars meddelanden läses; vid app-only-inloggning är angivelsen obligatorisk |
| `-Top 100` | Begränsar frågan till högst 100 meddelanden per anrop |
| `-Property id, subject, hasAttachments` | Begär endast de fält som behövs; det minskar svaret och snabbar upp anropet |
| `Get-MgUserMessageAttachment -MessageId` | Meddelande vars bilagor listas |
| `Invoke-MgGraphRequest -Method GET` | HTTP-metod för direktanropet mot Graph API |
| `-Uri` | Anropad slutpunkt; det bifogade `/$value` levererar bilagans råa filinnehåll i stället för ett JSON-objekt |
| `-OutputFilePath` | Skriver svaret direkt till målfilen utan att hålla hela bilagan i arbetsminnet |
| `Move-MgUserMessage -DestinationId "deleteditems"` | Flyttar det bearbetade meddelandet till målmappen; `deleteditems` är det välkända mappnamnet för «Borttagna objekt» |

</details>

Vid fler än 100 e-postmeddelanden: arbeta med `Get-MgUserMessage -All` eller sidindelning; för en månadskörning räcker vanligtvis en batch.

## 7\. Skicka rapportmeddelande via Graph

Även `Send-MailMessage` är föråldrat. Via samma appregistrering (behörigheten `Mail.Send`) skickas e-postmeddelandet direkt via Graph, här med en fil som base64-kodad bilaga:

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
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `-UserId` | Postlåda i vars namn e-postmeddelandet skickas; måste omfattas av Application Access Policy |
| `-BodyParameter` | Hela meddelandet som Hashtable i Graph-schemat: `message` med ämne, brödtext, mottagare och bilagor samt `saveToSentItems` för lagring i «Skickade objekt» |

</details>

## 8\. Kör obevakat

Som en schemalagd uppgift körs skriptet utan inloggning eftersom certifikatet finns i kontots arkiv:

```powershell
$action  = New-ScheduledTaskAction -Execute "powershell.exe" `
    -Argument '-NoProfile -ExecutionPolicy Bypass -File "D:\Scripts\graph-import.ps1"'
$trigger = New-ScheduledTaskTrigger -Daily -At 06:00
Register-ScheduledTask -TaskName "eCall-Graph-Import" -Action $action -Trigger $trigger `
    -User "DOMAIN\svc-ecall" -Password (Read-Host "Passwort")
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `New-ScheduledTaskAction -Execute` | Program som ska köras, här `powershell.exe` |
| `-Argument` | Kommandorad för programmet: `-NoProfile` hoppar över პროფილskript, `-ExecutionPolicy Bypass` kringgår skriptprincipen för detta anrop och `-File` anger skriptet |
| `New-ScheduledTaskTrigger -Daily -At 06:00` | Daglig utlösare klockan 06:00 |
| `Register-ScheduledTask -TaskName` | Uppgiftens namn i Aktivitetsschemaläggaren |
| `-Action` / `-Trigger` | Kopplar den tidigare skapade åtgärden och utlösaren till uppgiften |
| `-User` | Konto som uppgiften körs under; den privata nyckeln måste vara läsbar i dess certifikatarkiv |
| `-Password (Read-Host "Passwort")` | Frågar interaktivt efter kontots lösenord så att uppgiften kan starta även utan en inloggad användare; på så sätt hamnar det inte i skriptet eller historikfilen |

</details>

Det fullständiga exemplet med loggning och felhantering finns på GitHub: [pfstr/eCall-Log-Analyzer](https://github.com/pfstr/eCall-Log-Analyzer).

## Källor

1.  [Microsoft – «Retirement of Exchange Web Services in Exchange Online»](https://techcommunity.microsoft.com/blog/exchange/retirement-of-exchange-web-services-in-exchange-online/3924440): Meddelande och datum (1 oktober 2026) för slutet på EWS i Exchange Online.
    
2.  [Microsoft Learn – «Get access without a user (App-only)»](https://learn.microsoft.com/en-us/graph/auth-v2-service): App-only-autentisering mot Microsoft Graph med certifikat.
    
3.  [Microsoft Learn – «Limiting application permissions to specific mailboxes»](https://learn.microsoft.com/en-us/graph/auth-limit-mailbox-access): Application Access Policy för att begränsa appen till enskilda postlådor.
