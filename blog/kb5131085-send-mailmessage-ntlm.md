---
title: "KB5131085 auf Windows Server 2019: Send-MailMessage scheitert nach der NTLM-Signierung"
navTitle: "KB5131085 Mailversand"
description: "Nach dem November-Update KB5131085 schlägt auf Windows Server 2019 der Mailversand per Send-MailMessage gegen Postfix-Server fehl. Als Ursache gilt die neue NTLM-Signierung aus KB5135557. Der Artikel erklärt, warum ein SMTP-Client überhaupt NTLM verwendet, wie Sie den Fall prüfen und welche Workarounds es gibt: AUTH LOGIN erzwingen, NTLM auf dem Mailserver abschalten oder vorübergehend per Registry zurückstellen."
date: "2026-11-17"
kategorie: "Windows und SMTP"
timeToRead: "7 Min. Lesezeit"
themen:
  - "windows-client"
  - "smtp-mailflow"
produkte:
  - "windows-client"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
  - "releases"
slug: "kb5131085-send-mailmessage-ntlm"
translationId: "article-ca3e77cefa049fca"
url: "https://rafaelpfister.ch/blog/kb5131085-send-mailmessage-ntlm"
---

# KB5131085 auf Windows Server 2019: Send-MailMessage scheitert nach der NTLM-Signierung

Seit dem Patchday vom 10. November 2026 melden Administratoren, dass Skripte auf Windows Server 2019 keine Mails mehr versenden. Betroffen sind Backup-Reports, Monitoring-Meldungen und ähnliche Aufgaben, die seit Jahren mit dem PowerShell-Cmdlet `Send-MailMessage` an einen Postfix-Server senden. Nach der Deinstallation des kumulativen Updates KB5131085 (OS Build 17763.9487) funktioniert der Versand wieder. Günter Born hat den Fall am 17. November 2026 auf Borns IT- und Windows-Blog beschrieben, ausgelöst durch die Meldung eines Lesers.

Microsoft führt das Problem bisher nicht als Known Issue. Der einzige Known-Issue-Eintrag zu KB5131085 betrifft Aktivierungsbefehle mit Fehler `0x80041008`. Die Ursache ist deshalb nicht offiziell bestätigt, sie lässt sich aber aus den Änderungen des Updates und aus dem Verhalten der .NET-Klasse `SmtpClient` gut herleiten.

## Was KB5131085 an NTLM ändert

Das Update enthält Schutzmassnahmen gegen NTLM-Relay-Angriffe, die Microsoft in KB5135557 („NTLM signing security improvements in Windows") beschreibt. Sie gelten für Windows 11, Windows 10 Enterprise LTSC 2021 und Windows Server 2016 bis 2025. Im Changelog von KB5131085 steht dazu, dass Clients ab sofort NTLM Message Signing standardmässig für alle nicht-anonymen Verbindungen anbieten.

Technisch heisst das: Ein Windows-Client setzt in der NTLM-Aushandlung jetzt das Flag `NTLMSSP_NEGOTIATE_SIGN` (`0x00000010`) und das dafür nötige `NTLMSSP_NEGOTIATE_KEY_EXCH` (`0x40000000`), auch wenn die Anwendung keine Integrität angefordert hat. Bisher bekamen solche Anwendungen nur `NTLMSSP_NEGOTIATE_ALWAYS_SIGN`, einen Platzhalter ohne kryptografischen Schutz. Microsoft verteilt die Änderung in zwei Phasen:

| Phase | Datum | Verhalten |
|---|---|---|
| 1 | 10. November 2026 | Clients bieten Signing auf allen Verbindungen an. Server akzeptieren signierte und unsignierte Clients und protokollieren unsignierte. |
| 2 | Mai 2027 | Windows-Server verlangen Signing standardmässig und lehnen Clients ohne Signing mit `SEC_E_UNSUPPORTED_FUNCTION` (`0x80090302`) ab. |

Laut Microsoft fällt ein Windows-Client ohne Fehler auf das alte Verhalten zurück, wenn der Server Signing nicht bestätigt. Alle Windows-Server tun das. Gegenstellen mit einer eigenen NTLM-Implementierung, etwa ein Linux-Mailserver, sind in dieser Aussage nicht berücksichtigt.

## Warum ein SMTP-Client NTLM verwendet

`Send-MailMessage` in Windows PowerShell 5.1 verwendet intern die .NET-Klasse `System.Net.Mail.SmtpClient`. Diese prüft die `AUTH`-Zeile in der EHLO-Antwort des Servers und probiert die Verfahren in einer festen Reihenfolge: zuerst `GSSAPI` (Negotiate), dann `NTLM`, dann `WDigest`, zuletzt `LOGIN`. Bietet der Server NTLM an, nimmt `SmtpClient` NTLM, auch wenn Benutzername und Passwort per `-Credential` übergeben werden.

Postfix selbst implementiert keine SMTP-Authentifizierung, es delegiert sie an Cyrus SASL oder Dovecot. Beide können NTLM anbieten: Cyrus SASL über ein eigenes Plugin, Dovecot bis Version 2.3 über `ntlm` in `auth_mechanisms` (Dovecot 2.4 führt NTLM nicht mehr). Ist NTLM aktiv, kündigt Postfix etwa `AUTH PLAIN LOGIN NTLM` an, und jeder Windows-Client authentifiziert sich per NTLM, ohne dass es jemand bemerkt. Mit KB5131085 bietet der Client in dieser Aushandlung zusätzlich Signing und Schlüsselaustausch an. Wenn die NTLM-Implementierung des Mailservers damit nicht umgehen kann, scheitert die Anmeldung, und `Send-MailMessage` bricht ab. Ein Kommentator im Born-Artikel nennt die NTLM-Signierung als Ursache, und der Befund passt dazu: Der Workaround des Lesers funktioniert, weil er NTLM ausschliesst.

## Den Fall prüfen

Klären Sie zuerst, ob der Mailserver NTLM anbietet. Von einem Linux-Rechner aus zeigt die EHLO-Antwort nach STARTTLS die angebotenen Verfahren:

```bash
openssl s_client -starttls smtp -crlf -quiet \
  -connect mail.example.com:587
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-starttls smtp` | Baut die Verbindung im Klartext auf und wechselt per STARTTLS auf TLS |
| `-crlf` | Sendet Zeilenenden als CRLF, wie SMTP es verlangt |
| `-quiet` | Unterdrückt Zertifikats- und Sitzungsdetails |
| `-connect mail.example.com:587` | Ziel-Host und Submission-Port |

</details>

Geben Sie nach dem Verbindungsaufbau `EHLO test` ein. Enthält die Zeile `250-AUTH` den Eintrag `NTLM`, ist der Server betroffen. Auf dem Postfix-Server selbst zeigen zwei Befehle, welches SASL-Backend aktiv ist und welche Verfahren Dovecot anbietet:

```bash
postconf smtpd_sasl_type
doveconf auth_mechanisms
```

Auf dem Windows-Server protokolliert das NTLM-Log ab Windows Server 2019 Verbindungen mit den ausgehandelten Flags. Event 4022 ist der Client-Eintrag:

```powershell
Get-WinEvent -LogName 'Microsoft-Windows-NTLM/Operational' -MaxEvents 50 |
  Where-Object { $_.Id -in 4022, 4023 } |
  Format-List TimeCreated, Id, Message
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-LogName 'Microsoft-Windows-NTLM/Operational'` | Liest das operative NTLM-Protokoll |
| `-MaxEvents 50` | Begrenzt die Ausgabe auf die 50 neuesten Einträge |
| `Where-Object { $_.Id -in 4022, 4023 }` | Filtert auf die Audit-Events für Client (4022) und Server (4023) |
| `Format-List TimeCreated, Id, Message` | Zeigt Zeitpunkt, Event-ID und den Text mit den Negotiated Flags |

</details>

Ein Eintrag zum Zeitpunkt des Versandversuchs mit dem Mailserver als Ziel bestätigt, dass `SmtpClient` NTLM verwendet hat.

## Workaround 1: AUTH LOGIN im Skript erzwingen

`Send-MailMessage` bietet keinen Parameter, um das Anmeldeverfahren festzulegen. Die Klasse `SmtpClient` direkt aufgerufen kann das: Ein `CredentialCache` liefert die Anmeldedaten nur für ein bestimmtes Verfahren aus. Ist nur `Login` hinterlegt, überspringt `SmtpClient` NTLM und nimmt `AUTH LOGIN`. Das folgende Skript basiert auf dem Beispiel des Lesers aus dem Born-Artikel:

```powershell
$server = 'mail.example.com'
$port   = 587
$cred   = Import-Clixml -Path 'C:\Skripte\smtp.cred'

$cache = New-Object System.Net.CredentialCache
$cache.Add($server, $port, 'Login', $cred.GetNetworkCredential())

$smtp = New-Object System.Net.Mail.SmtpClient($server, $port)
$smtp.EnableSsl   = $true
$smtp.Credentials = $cache

$msg = New-Object System.Net.Mail.MailMessage
$msg.From = 'backup@example.com'
$msg.To.Add('admin@example.com')
$msg.Subject      = 'Backup-Report'
$msg.Body         = 'Sicherung abgeschlossen.'
$msg.BodyEncoding = [System.Text.Encoding]::UTF8

try {
    $smtp.Send($msg)
}
finally {
    $msg.Dispose()
    $smtp.Dispose()
}
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `Import-Clixml -Path …` | Lädt ein zuvor mit `Get-Credential \| Export-Clixml` gespeichertes PSCredential; per DPAPI an Benutzer und Rechner gebunden |
| `$cache.Add($server, $port, 'Login', …)` | Hinterlegt die Anmeldedaten nur für Host, Port und das Verfahren `Login` |
| `SmtpClient($server, $port)` | Erstellt den SMTP-Client für Server und Submission-Port |
| `EnableSsl = $true` | Wechselt per STARTTLS auf TLS, bevor die Anmeldedaten übertragen werden |
| `Credentials = $cache` | Übergibt den CredentialCache statt einer einfachen NetworkCredential |
| `BodyEncoding = [System.Text.Encoding]::UTF8` | Kodiert den Text als UTF-8, damit Umlaute korrekt ankommen |

</details>

`EnableSsl` ist hier zwingend: `AUTH LOGIN` überträgt Benutzername und Passwort nur Base64-kodiert, ohne TLS also im Klartext. Der Workaround ist ein Eingriff pro Skript, ändert aber nichts an der Systemkonfiguration und lässt den NTLM-Schutz des Updates intakt.

## Workaround 2: NTLM auf dem Mailserver abschalten

Wenn Sie den Postfix-Server selbst betreiben, können Sie das Problem zentral beheben: Entfernen Sie `ntlm` aus `auth_mechanisms` in der Dovecot-Konfiguration (meist `/etc/dovecot/conf.d/10-auth.conf`), sodass nur noch `plain login` übrig bleibt, und laden Sie Dovecot neu:

```bash
sudo systemctl reload dovecot
```

Bei Cyrus SASL beschränkt `mech_list: plain login` in der Datei `smtpd.conf` des SASL-Konfigurationsverzeichnisses die Verfahren entsprechend. Danach kündigt Postfix kein NTLM mehr an, und `Send-MailMessage` fällt ohne Skriptänderung auf `LOGIN` zurück. Prüfen Sie vorher, ob andere Clients auf NTLM angewiesen sind, etwa ältere Outlook-Profile mit der Option „Gesicherte Kennwortauthentifizierung (SPA)". Für Submission über TLS bringt NTLM gegenüber `PLAIN` und `LOGIN` keinen Sicherheitsgewinn.

## Workaround 3: Registry, nur vorübergehend

KB5135557 dokumentiert einen Registry-Wert, mit dem der Client Signing nicht mehr anbietet:

```powershell
reg add 'HKLM\SYSTEM\CurrentControlSet\Control\Lsa\MSV1_0' /v ClientAllowUnsignedNtlm /t REG_DWORD /d 1 /f
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `HKLM\…\Lsa\MSV1_0` | Schlüssel des NTLM-Authentifizierungspakets |
| `/v ClientAllowUnsignedNtlm` | Name des Werts; `0` (Standard) bietet Signing an, `1` stellt das alte Verhalten wieder her |
| `/t REG_DWORD` | Datentyp des Werts |
| `/d 1` | Setzt den Wert auf 1 |
| `/f` | Überschreibt ohne Rückfrage |

</details>

Laut Microsoft ist kein Neustart nötig. Der Wert gilt allerdings für alle NTLM-Verbindungen des Servers, nicht nur für den Mailversand, und hebt damit den Relay-Schutz des Updates für das ganze System auf. Microsoft beschreibt ihn als reine Kompatibilitätsmassnahme und behält sich vor, die Registry-Werte später zu entfernen. Er überbrückt höchstens die Zeit, bis Workaround 1 oder 2 umgesetzt ist. Das Deinstallieren von KB5131085 ist keine Alternative, weil damit alle Sicherheitskorrekturen des November-Patchdays fehlen.

## Mittelfristig: Send-MailMessage ersetzen

Microsoft führt `Send-MailMessage` seit Jahren als veraltet und rät von der Verwendung ab, weil das Cmdlet keine sicheren Verbindungen garantiert. Auch für `SmtpClient` empfiehlt die .NET-Dokumentation für neue Entwicklungen andere Bibliotheken wie MailKit. Für Skripte, die Mails über Exchange Online versenden, ist die Microsoft Graph API mit `Send-MgUserMail` und App-Registrierung der unterstützte Weg; die Einrichtung ist im Artikel [Mit PowerShell auf Exchange Online zugreifen: EWS durch Microsoft Graph ersetzen](/blog/microsoft-graph-powershell-postfach-anbindung) beschrieben. Spätestens mit Phase 2 im Mai 2027 lohnt sich eine Bestandsaufnahme, welche Skripte und Dienste noch NTLM verwenden: Die Audit-Events 4022 und 4023 liefern dafür die Liste.

## Quellen

1.  [Windows Server 2019: KB5131085 stört Mail-Versand per PowerShell (Borns IT- und Windows-Blog)](https://borncity.com/blog/2026/11/17/windows-server-2019-kb5131085-stoert-mail-versand-per-powershell/): Leserbericht vom 17. November 2026 mit Symptom, Deinstallations-Test, CredentialCache-Skript und dem Hinweis auf die NTLM-Signierung in den Kommentaren.

2.  [KB5131085 (OS Build 17763.9487)](https://support.microsoft.com/help/5131085): Changelog des November-Updates für Windows Server 2019 und Windows 10 Enterprise LTSC 2019 mit dem NTLM-Eintrag und dem Known Issue zu Fehler 0x80041008.

3.  [KB5135557: NTLM signing security improvements in Windows](https://support.microsoft.com/en-us/help/5135557): Flags, Zeitplan der zwei Phasen, Audit-Events 4022/4023 und die Registry-Werte ClientAllowUnsignedNtlm und ServerRequireNtlmSign.

4.  [SmtpAuthenticationManager.cs (.NET Framework Reference Source)](https://github.com/microsoft/referencesource/blob/master/System/net/System/Net/mail/SmtpAuthenticationManager.cs): Registrierungsreihenfolge der SMTP-Anmeldemodule Negotiate, NTLM, Digest und Login in SmtpClient.

5.  [Send-MailMessage (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/send-mailmessage): Hinweis, dass das Cmdlet veraltet ist und keine sicheren Verbindungen garantiert.

6.  [SmtpClient Class (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/api/system.net.mail.smtpclient): Empfehlung, für neue Entwicklungen MailKit oder andere Bibliotheken zu verwenden.

7.  [Postfix SASL Howto](https://www.postfix.org/SASL_README.html): SMTP-Authentifizierung in Postfix über Cyrus SASL oder Dovecot, inklusive Einschränkung der angebotenen Verfahren.

8.  [Dovecot: Authentication Mechanisms Overview](https://doc.dovecot.org/main/core/config/auth/mechanisms/overview.html): Aktuelle Liste der SASL-Verfahren in Dovecot, in der NTLM nicht mehr enthalten ist.
