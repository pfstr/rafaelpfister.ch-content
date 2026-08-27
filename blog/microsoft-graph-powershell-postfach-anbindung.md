---
title: "Mit PowerShell auf Exchange Online zugreifen: EWS durch Microsoft Graph ersetzen"
navTitle: "Graph in PowerShell"
description: "EWS endet in Exchange Online am 1. Oktober 2026. So registrieren Sie eine App, melden ein PowerShell-Skript per Zertifikat an, beschränken den Zugriff auf einzelne Postfächer und verarbeiten Nachrichten und Anhänge über Microsoft Graph."
date: "2026-07-11"
kategorie: "Totemomail"
timeToRead: "5 Min. Lesezeit"
themen:
  - "microsoft-365-exchange"
produkte:
  - "totemomail"
  - "exchange"
protokolle:
  - "powershell"
  - "tls"
slug: "microsoft-graph-powershell-postfach-anbindung"
translationId: "article-4c6a02c79b7bf0fe"
url: "https://rafaelpfister.ch/blog/microsoft-graph-powershell-postfach-anbindung"
---

# Mit PowerShell auf Exchange Online zugreifen: EWS durch Microsoft Graph ersetzen

Microsoft stellt Exchange Web Services (EWS) in Exchange Online am **1. Oktober 2026** ein. Skripte, die Nachrichten oder Anhänge aus einem Postfach abholen, müssen deshalb auf Microsoft Graph wechseln.

Das Beispiel in diesem Beitrag läuft ohne Benutzeranmeldung: Es lädt ZIP-Anhänge aus einem Postfach, entpackt sie, verschiebt verarbeitete Nachrichten und versendet anschliessend einen Bericht. Dafür braucht es eine App-Registrierung, ein Zertifikat und zwei Graph-Berechtigungen. Werte wie `example.com`, Tenant-ID und App-ID sind Platzhalter.

## 1. Benötigte PowerShell-Module

Es genügen drei Module des Microsoft-Graph-SDK, nicht das gesamte Meta-Modul `Microsoft.Graph`:

```powershell
Install-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions -Scope AllUsers
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions` | Die drei benötigten Teilmodule als Positionsargument `-Name`: Anmeldung, Mail-Cmdlets und Aktionen wie `Send-MgUserMail` |
| `-Scope AllUsers` | Installiert die Module maschinenweit unter `Program Files`; nötig, damit sie auch dem später konfigurierten Dienstkonto des Scheduled Tasks zur Verfügung stehen (erfordert Administratorrechte) |

</details>

## 2\. App-Registrierung in Entra ID

Ein unbeaufsichtigtes Skript meldet sich als Anwendung mit eigenen Rechten an. Legen Sie im [Entra Admin Center](https://entra.microsoft.com) unter **App registrations** eine neue Registrierung an und vergeben Sie unter **API permissions → Microsoft Graph → Application permissions** diese beiden Rechte:

-   `Mail.ReadWrite`: Mails lesen und nach der Verarbeitung verschieben
    
-   `Mail.Send`: die Report-Mail versenden
    

Erteilen Sie danach die Administratorzustimmung und notieren Sie Tenant-ID sowie Application (Client) ID.

## 3\. Zertifikat statt Client Secret

Eine App-Only-Anmeldung funktioniert mit Client Secret oder Zertifikat. Für geplante Aufgaben ist ein Zertifikat die bessere Wahl: Der private Schlüssel bleibt im Zertifikatspeicher, im Skript steht kein Passwort. Erzeugen Sie das Zertifikat auf dem ausführenden Server und exportieren Sie nur den öffentlichen Teil:

```powershell
$cert = New-SelfSignedCertificate -Subject "CN=eCall-Graph" `
    -CertStoreLocation "Cert:\LocalMachine\My" `
    -KeyExportPolicy NonExportable -KeySpec Signature `
    -KeyLength 2048 -NotAfter (Get-Date).AddYears(2)

Export-Certificate -Cert $cert -FilePath .\eCall-Graph.cer
$cert.Thumbprint   # -> im Skript als Thumbprint verwenden
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `New-SelfSignedCertificate -Subject` | Antragstellername des Zertifikats; dient nur der Wiedererkennung im Zertifikatspeicher |
| `-CertStoreLocation "Cert:\LocalMachine\My"` | Legt das Zertifikat im Computerspeicher ab, nicht im Benutzerspeicher; damit ist es unabhängig vom angemeldeten Benutzer verfügbar |
| `-KeyExportPolicy NonExportable` | Verhindert den Export des privaten Schlüssels; er verlässt den Server nicht |
| `-KeySpec Signature` | Erzeugt einen Signaturschlüssel; die App-Anmeldung signiert damit das Token-Request-Assertion |
| `-KeyLength 2048` | RSA-Schlüssellänge 2048 Bit |
| `-NotAfter (Get-Date).AddYears(2)` | Gültigkeitsende in zwei Jahren; danach muss das Zertifikat erneuert und neu hochgeladen werden |
| `Export-Certificate -Cert` | Zu exportierendes Zertifikatsobjekt |
| `-FilePath` | Zieldatei; enthält als `.cer` nur den öffentlichen Teil |

</details>

Die exportierte `.cer`\-Datei in der App-Registrierung unter Certificates & secrets hochladen. Das Konto des Scheduled Tasks braucht Leserecht auf den privaten Schlüssel (`certlm.msc` → Zertifikat → All Tasks → Manage Private Keys).

## 4\. Zugriff auf einzelne Postfächer einschränken

Application Permissions gelten zunächst für alle Postfächer im Tenant. Begrenzen Sie die App deshalb mit einer Application Access Policy auf eine mailaktivierte Sicherheitsgruppe. Dieser Schritt wird einmalig in Exchange Online PowerShell ausgeführt:

```powershell
New-ApplicationAccessPolicy -AppId "<App-ID>" `
    -PolicyScopeGroupId "graph-mailboxes@example.com" `
    -AccessRight RestrictAccess `
    -Description "eCall Graph: nur Log-Postfach"

# Wirksamkeit pruefen
Test-ApplicationAccessPolicy -AppId "<App-ID>" -Identity "ecall-logs@example.com"
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `New-ApplicationAccessPolicy -AppId` | Application (Client) ID der App-Registrierung, für die die Richtlinie gilt |
| `-PolicyScopeGroupId` | Mailaktivierte Sicherheitsgruppe, deren Mitglieder den Geltungsbereich definieren |
| `-AccessRight RestrictAccess` | Beschränkt die App auf die Postfächer der Gruppe; die Alternative `DenyAccess` würde genau diese Postfächer sperren |
| `-Description` | Freitext zur Dokumentation der Richtlinie |
| `Test-ApplicationAccessPolicy -Identity` | Prüft für ein konkretes Postfach, ob die App darauf zugreifen darf (`AccessCheckResult: Granted` oder `Denied`) |

</details>

## 5\. Verbindung aufbauen

Die Anmeldung nutzt Tenant-ID, App-ID und den Zertifikat-Thumbprint, ganz ohne Benutzerinteraktion:

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
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `Connect-MgGraph -TenantId` | Tenant, gegen den sich die App anmeldet |
| `-ClientId` | Application (Client) ID der App-Registrierung |
| `-CertificateThumbprint` | Wählt das Anmeldezertifikat über seinen Thumbprint aus dem lokalen Zertifikatspeicher; die Kombination aus `-ClientId` und Zertifikat ergibt eine App-Only-Anmeldung ohne Benutzer |
| `-NoWelcome` | Unterdrückt den Begrüssungstext nach der Anmeldung; sinnvoll für Skriptausgaben und Logs |

</details>

## 6. Nachrichten lesen und ZIP-Anhänge herunterladen

Nun kann das Skript den Posteingang durchlaufen, ZIP-Anhänge speichern und entpacken sowie verarbeitete Nachrichten nach «Gelöschte Elemente» verschieben. Der Download erfolgt über den Endpunkt `/$value` und `Invoke-MgGraphRequest -OutputFilePath`. Dadurch landet der Rohinhalt direkt in einer Datei, ohne einen grossen Anhang vollständig im Arbeitsspeicher zu halten:

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

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `Add-Type -AssemblyName System.IO.Compression.FileSystem` | Lädt die .NET-Assembly mit der Klasse `ZipFile` zum Entpacken |
| `Get-MgUserMessage -UserId` | Postfach, dessen Nachrichten gelesen werden; bei App-Only-Anmeldung ist die Angabe zwingend |
| `-Top 100` | Begrenzt die Abfrage auf maximal 100 Nachrichten pro Aufruf |
| `-Property id, subject, hasAttachments` | Fordert nur die benötigten Felder an; das verkleinert die Antwort und beschleunigt den Aufruf |
| `Get-MgUserMessageAttachment -MessageId` | Nachricht, deren Anhänge aufgelistet werden |
| `Invoke-MgGraphRequest -Method GET` | HTTP-Methode des Direktaufrufs gegen die Graph-API |
| `-Uri` | Aufgerufener Endpunkt; das angehängte `/$value` liefert den rohen Dateiinhalt des Anhangs statt eines JSON-Objekts |
| `-OutputFilePath` | Schreibt die Antwort direkt in die Zieldatei, ohne den Anhang komplett im Arbeitsspeicher zu halten |
| `Move-MgUserMessage -DestinationId "deleteditems"` | Verschiebt die verarbeitete Nachricht in den Zielordner; `deleteditems` ist der wohlbekannte Ordnername für «Gelöschte Elemente» |

</details>

Bei mehr als 100 Mails mit `Get-MgUserMessage -All` oder Paging arbeiten; für einen Monatslauf reicht meist ein Batch.

## 7\. Report-Mail über Graph senden

Auch `Send-MailMessage` ist veraltet. Über dieselbe App-Registrierung (Recht `Mail.Send`) geht die Mail direkt via Graph hinaus, hier mit einer Datei als base64-kodiertem Anhang:

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
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-UserId` | Postfach, in dessen Namen die Mail versendet wird; muss vom Geltungsbereich der Application Access Policy abgedeckt sein |
| `-BodyParameter` | Die komplette Nachricht als Hashtable im Graph-Schema: `message` mit Betreff, Body, Empfängern und Anhängen sowie `saveToSentItems` für die Ablage in «Gesendete Elemente» |

</details>

## 8\. Unbeaufsichtigt ausführen

Als geplante Aufgabe läuft das Skript ohne Anmeldung, weil das Zertifikat im Store des Kontos liegt:

```powershell
$action  = New-ScheduledTaskAction -Execute "powershell.exe" `
    -Argument '-NoProfile -ExecutionPolicy Bypass -File "D:\Scripts\graph-import.ps1"'
$trigger = New-ScheduledTaskTrigger -Daily -At 06:00
Register-ScheduledTask -TaskName "eCall-Graph-Import" -Action $action -Trigger $trigger `
    -User "DOMAIN\svc-ecall" -Password (Read-Host "Passwort")
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `New-ScheduledTaskAction -Execute` | Auszuführendes Programm, hier `powershell.exe` |
| `-Argument` | Kommandozeile für das Programm: `-NoProfile` überspringt Profilskripte, `-ExecutionPolicy Bypass` umgeht die Skriptrichtlinie für diesen Aufruf, `-File` benennt das Skript |
| `New-ScheduledTaskTrigger -Daily -At 06:00` | Täglicher Auslöser um 06:00 Uhr |
| `Register-ScheduledTask -TaskName` | Name der Aufgabe in der Aufgabenplanung |
| `-Action` / `-Trigger` | Verknüpft die zuvor erstellte Aktion und den Auslöser mit der Aufgabe |
| `-User` | Konto, unter dem die Aufgabe läuft; in dessen Zertifikatspeicher muss der private Schlüssel lesbar sein |
| `-Password (Read-Host "Passwort")` | Fragt das Kontopasswort interaktiv ab, damit die Aufgabe auch ohne angemeldeten Benutzer starten kann; so landet es nicht im Skript oder in der Verlaufsdatei |

</details>

Das vollständige Beispiel mit Protokollierung und Fehlerbehandlung liegt auf GitHub: [pfstr/eCall-Log-Analyzer](https://github.com/pfstr/eCall-Log-Analyzer).

## Quellen

1.  [Microsoft – «Retirement of Exchange Web Services in Exchange Online»](https://techcommunity.microsoft.com/blog/exchange/retirement-of-exchange-web-services-in-exchange-online/3924440): Ankündigung und Stichtag (1. Oktober 2026) für das Ende von EWS in Exchange Online.
    
2.  [Microsoft Learn – «Get access without a user (App-only)»](https://learn.microsoft.com/en-us/graph/auth-v2-service): App-Only-Authentifizierung gegen Microsoft Graph mit Zertifikat.
    
3.  [Microsoft Learn – «Limiting application permissions to specific mailboxes»](https://learn.microsoft.com/en-us/graph/auth-limit-mailbox-access): Application Access Policy zur Einschränkung der App auf einzelne Postfächer.
