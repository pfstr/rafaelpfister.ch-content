---
title: "Microsoft-Graph-Verbindung in PowerShell bauen: Postfach auslesen und Mails senden"
navTitle: "Graph in PowerShell"
description: "EWS wird für Exchange Online am 1. Oktober 2026 abgeschaltet. Diese Schritt-für-Schritt-Anleitung baut die Graph-Anbindung in PowerShell komplett auf: App-Registrierung, zertifikatsbasierte App-Only-Anmeldung, Postfach auslesen, ZIP-Anhänge herunterladen und Mails versenden – mit beispielhaften Code-Snippets."
date: "2026-07-11"
kategorie: "Totemomail"
timeToRead: "5 min to read"
themen:
  - "microsoft-365-exchange"
slug: "microsoft-graph-powershell-postfach-anbindung"
url: "https://rafaelpfister.ch/blog/microsoft-graph-powershell-postfach-anbindung"
aiPrompt: |
  Du bist mein PowerShell- und Microsoft-365-Assistent. Hilf mir, eine App-Only-Anbindung an Microsoft Graph aufzubauen, um ein Postfach auszulesen und Mails zu versenden – als Ersatz für das abgekündigte Exchange Web Services (EWS, Ende: 1. Oktober 2026).

  Gehe mit mir Schritt für Schritt vor und frage nach den Werten, die nur ich kenne (Tenant-ID, App-/Client-ID, Zertifikats-Thumbprint, Ziel-Postfach):
  1. Benötigte Graph-PowerShell-Module installieren (nur Authentication, Mail, Users.Actions).
  2. App-Registrierung in Entra ID mit Application-Permissions Mail.ReadWrite und Mail.Send, danach Admin Consent.
  3. Zertifikatsbasierte App-Only-Anmeldung einrichten (kein Client Secret).
  4. Verbindung testen und ein Postfach auslesen.
  5. Optional: Anhänge (z. B. ZIP) herunterladen und eine Report-Mail versenden.

  Gib mir für jeden Schritt den konkreten PowerShell-Code, erkläre Sicherheitsaspekte (Least Privilege, Application Access Policy zur Einschränkung auf einzelne Postfächer) und weise auf typische Fehlerquellen hin.
---

# Microsoft-Graph-Verbindung in PowerShell bauen: Postfach auslesen und Mails senden

Exchange Web Services (EWS) wird für Exchange Online am **1\. Oktober 2026** abgeschaltet. Wer per Skript auf Postfächer zugreift – etwa um automatisiert zugestellte Dateien abzuholen –, muss auf die Microsoft Graph API wechseln. Diese Anleitung baut die komplette Graph-Anbindung in PowerShell auf: App-Registrierung, zertifikatsbasierte Anmeldung, Lesen und Herunterladen von Anhängen sowie Mailversand – jeweils mit dem konkreten Code.

Als durchgehendes Beispiel dient ein unbeaufsichtigtes Skript, das ZIP-Anhänge aus einem Postfach herunterlädt, entpackt und anschliessend einen Report verschickt. Platzhalter wie `example.com` und die Tenant-/App-IDs durch eigene Werte ersetzen.

## 1\. Voraussetzungen

Es genügen drei Module des Microsoft-Graph-SDK – nicht das gesamte Meta-Modul `Microsoft.Graph`:

```powershell
Install-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions -Scope AllUsers
```

## 2\. App-Registrierung in Entra ID

Unbeaufsichtigte Skripte melden nicht einen Benutzer an, sondern eine App mit eigenen Rechten (App-Only). Im [Entra Admin Center](https://entra.microsoft.com) unter App registrations eine neue Registrierung anlegen und ihr unter API permissions → Microsoft Graph → Application permissions zwei Rechte geben:

-   `Mail.ReadWrite` – Mails lesen und nach der Verarbeitung verschieben
    
-   `Mail.Send` – die Report-Mail versenden
    

Danach **Grant admin consent** klicken und Tenant-ID sowie Application (client) ID notieren.

## 3\. Zertifikat statt Client Secret

App-Only authentifiziert per Client Secret oder Zertifikat. Für Scheduled Tasks ist das Zertifikat die sauberere Wahl: Der private Schlüssel bleibt im Zertifikatspeicher, es liegt kein Passwort im Skript. Zertifikat auf dem ausführenden Server erzeugen und den öffentlichen Teil exportieren:

```powershell
$cert = New-SelfSignedCertificate -Subject "CN=eCall-Graph" `
    -CertStoreLocation "Cert:\LocalMachine\My" `
    -KeyExportPolicy NonExportable -KeySpec Signature `
    -KeyLength 2048 -NotAfter (Get-Date).AddYears(2)

Export-Certificate -Cert $cert -FilePath .\eCall-Graph.cer
$cert.Thumbprint   # -> im Skript als Thumbprint verwenden
```

Die exportierte `.cer`\-Datei in der App-Registrierung unter Certificates & secrets hochladen. Das Konto des Scheduled Tasks braucht Leserecht auf den privaten Schlüssel (`certlm.msc` → Zertifikat → All Tasks → Manage Private Keys).

## 4\. Zugriff auf einzelne Postfächer einschränken

Application Permissions gelten sonst **tenant-weit** – die App dürfte jedes Postfach im Tenant lesen. Eine Application Access Policy begrenzt sie auf eine Mail-aktivierte Sicherheitsgruppe mit den erlaubten Postfächern (Exchange Online PowerShell, einmalig):

```powershell
New-ApplicationAccessPolicy -AppId "<App-ID>" `
    -PolicyScopeGroupId "graph-mailboxes@example.com" `
    -AccessRight RestrictAccess `
    -Description "eCall Graph: nur Log-Postfach"

# Wirksamkeit pruefen
Test-ApplicationAccessPolicy -AppId "<App-ID>" -Identity "ecall-logs@example.com"
```

## 5\. Verbindung aufbauen

Die Anmeldung nutzt Tenant-ID, App-ID und den Zertifikat-Thumbprint – ganz ohne Benutzerinteraktion:

```powershell
$TenantId   = "00000000-0000-0000-0000-000000000000"
$ClientId   = "00000000-0000-0000-0000-000000000000"
$Thumbprint = "0000000000000000000000000000000000000000"
$Mailbox    = "ecall-logs@example.com"

Import-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions
Connect-MgGraph -TenantId $TenantId -ClientId $ClientId `
    -CertificateThumbprint $Thumbprint -NoWelcome
```

## 6\. Mails lesen und ZIP-Anhänge herunterladen

Der Kern: Posteingang durchgehen, ZIP-Anhänge speichern, entpacken und die verarbeitete Mail nach „Gelöschte Elemente“ verschieben. Entscheidend ist der Download über den `/$value`\-Endpunkt mit `Invoke-MgGraphRequest -OutputFilePath` – das streamt den Rohinhalt direkt in eine Datei und funktioniert auch bei grossen Anhängen zuverlässig:

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

    # verarbeitete Mail in "Geloeschte Elemente" verschieben
    Move-MgUserMessage -UserId $Mailbox -MessageId $msg.Id -DestinationId "deleteditems" | Out-Null
}
```

Bei mehr als 100 Mails mit `Get-MgUserMessage -All` oder Paging arbeiten; für einen Monatslauf reicht meist ein Batch.

## 7\. Report-Mail über Graph senden

Auch `Send-MailMessage` ist veraltet. Über dieselbe App-Registrierung (Recht `Mail.Send`) geht die Mail direkt via Graph hinaus – hier mit einer Datei als base64-kodiertem Anhang:

```powershell
$pfad = "D:\Reports
eport.csv"
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

## 8\. Unbeaufsichtigt ausführen

Als geplante Aufgabe läuft das Skript ohne Anmeldung, weil das Zertifikat im Store des Kontos liegt:

```powershell
$action  = New-ScheduledTaskAction -Execute "powershell.exe" `
    -Argument '-NoProfile -ExecutionPolicy Bypass -File "D:\Scripts\graph-import.ps1"'
$trigger = New-ScheduledTaskTrigger -Daily -At 06:00
Register-ScheduledTask -TaskName "eCall-Graph-Import" -Action $action -Trigger $trigger `
    -User "DOMAIN\svc-ecall" -Password (Read-Host "Passwort")
```

Das vollständige Skript – inklusive Logging, Fehlerbehandlung und der eigentlichen Verrechnungslogik – liegt als lauffähiges Beispiel auf GitHub: [github.com/pfstr/eCall-Log-Analyzer](https://github.com/pfstr/eCall-Log-Analyzer)

## Quellen

1.  [Microsoft – «Retirement of Exchange Web Services in Exchange Online»](https://techcommunity.microsoft.com/blog/exchange/retirement-of-exchange-web-services-in-exchange-online/3924440) — Ankündigung und Stichtag (1. Oktober 2026) für das Ende von EWS in Exchange Online.
    
2.  [Microsoft Learn – «Get access without a user (App-only)»](https://learn.microsoft.com/en-us/graph/auth-v2-service) — App-Only-Authentifizierung gegen Microsoft Graph mit Zertifikat.
    
3.  [Microsoft Learn – «Limiting application permissions to specific mailboxes»](https://learn.microsoft.com/en-us/graph/auth-limit-mailbox-access) — Application Access Policy zur Einschränkung der App auf einzelne Postfächer.
