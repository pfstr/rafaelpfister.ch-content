---
title: "KB5131085 on Windows Server 2019: Send-MailMessage fails after NTLM signing"
navTitle: "KB5131085 mail sending"
description: "After the November update KB5131085, sending mail with Send-MailMessage to Postfix servers fails on Windows Server 2019. The cause is believed to be the new NTLM signing from KB5135557. This article explains why an SMTP client uses NTLM in the first place, how to investigate the case, and which workarounds are available: force AUTH LOGIN, disable NTLM on the mail server, or temporarily revert it through the registry."
date: "2026-11-17"
kategorie: "Windows and SMTP"
timeToRead: "7 min read"
themen:
  - windows-client
  - smtp-mailflow
produkte:
  - "windows-client"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
  - "releases"
slug: "kb5131085-on-windows-server-2019-send-mailmessage-fails-after-ntlm-signing"
translationId: "article-ca3e77cefa049fca"
translationOf: kb5131085-send-mailmessage-ntlm
url: https://rafaelpfister.ch/en/blog/kb5131085-on-windows-server-2019-send-mailmessage-fails-after-ntlm-signing
translationSourceHash: 6416155fbbec934196848b06f3468ff46dbca040310b92cf8fd3171db1858227
translationModel: gpt-5.6-terra
translatedAt: 2026-11-17T09:05:28.438Z
translationReview: automatic
---

# KB5131085 on Windows Server 2019: Send-MailMessage fails after NTLM signing

Since the November 10, 2026 Patch Tuesday, administrators have reported that scripts on Windows Server 2019 can no longer send mail. Backup reports, monitoring alerts, and similar tasks that have been sending to a Postfix server for years using the PowerShell cmdlet `Send-MailMessage` are affected. Sending works again after uninstalling cumulative update KB5131085 (OS Build 17763.9487). Günter Born described the case on November 17, 2026, on Born's IT and Windows Blog, prompted by a reader report.

Microsoft has not yet listed the issue as a Known Issue. The only Known Issue entry for KB5131085 concerns activation commands with error `0x80041008`. The cause is therefore not officially confirmed, but it can be inferred from the changes in the update and the behavior of the .NET class `SmtpClient`.

## What KB5131085 changes in NTLM

The update contains protections against NTLM relay attacks, which Microsoft describes in KB5135557 ("NTLM signing security improvements in Windows"). They apply to Windows 11, Windows 10 Enterprise LTSC 2021, and Windows Server 2016 through 2025. The KB5131085 changelog states that clients now offer NTLM Message Signing by default for all non-anonymous connections.

Technically, this means that a Windows client now sets the flag `NTLMSSP_NEGOTIATE_SIGN` (`0x00000010`) and the required `NTLMSSP_NEGOTIATE_KEY_EXCH` (`0x40000000`) in NTLM negotiation, even if the application did not request integrity. Previously, such applications received only `NTLMSSP_NEGOTIATE_ALWAYS_SIGN`, a placeholder without cryptographic protection. Microsoft is rolling out the change in two phases:

| Phase | Date | Behavior |
|---|---|---|
| 1 | November 10, 2026 | Clients offer signing on all connections. Servers accept signed and unsigned clients and log unsigned ones. |
| 2 | May 2027 | Windows servers require signing by default and reject clients without signing with `SEC_E_UNSUPPORTED_FUNCTION` (`0x80090302`). |

According to Microsoft, a Windows client falls back to the old behavior without error if the server does not confirm signing. All Windows servers do so. This statement does not account for peers with their own NTLM implementation, such as a Linux mail server.

## Why an SMTP client uses NTLM

`Send-MailMessage` in Windows PowerShell 5.1 internally uses the .NET class `System.Net.Mail.SmtpClient`. It checks the `AUTH` line in the server's EHLO response and tries mechanisms in a fixed order: first `GSSAPI` (Negotiate), then `NTLM`, then `WDigest`, and finally `LOGIN`. If the server offers NTLM, `SmtpClient` uses NTLM, even when the username and password are supplied through `-Credential`.

Postfix itself does not implement SMTP authentication; it delegates it to Cyrus SASL or Dovecot. Both can offer NTLM: Cyrus SASL through its own plugin, and Dovecot up to version 2.3 through `ntlm` in `auth_mechanisms` (Dovecot 2.4 no longer includes NTLM). If NTLM is enabled, Postfix announces something like `AUTH PLAIN LOGIN NTLM`, and every Windows client authenticates via NTLM without anyone noticing. With KB5131085, the client additionally offers signing and key exchange in this negotiation. If the mail server's NTLM implementation cannot handle this, authentication fails and `Send-MailMessage` aborts. A commenter on the Born article identifies NTLM signing as the cause, and the finding fits: the reader's workaround works because it excludes NTLM.

## Investigating the case

First, determine whether the mail server offers NTLM. From a Linux machine, the EHLO response after STARTTLS shows the offered mechanisms:

```bash
openssl s_client -starttls smtp -crlf -quiet \
  -connect mail.example.com:587
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-starttls smtp` | Establishes the connection in plaintext and switches to TLS via STARTTLS |
| `-crlf` | Sends line endings as CRLF, as required by SMTP |
| `-quiet` | Suppresses certificate and session details |
| `-connect mail.example.com:587` | Destination host and submission port |

</details>

After establishing the connection, enter `EHLO test`. If the line `250-AUTH` contains `NTLM`, the server is affected. On the Postfix server itself, two commands show which SASL backend is active and which mechanisms Dovecot offers:

```bash
postconf smtpd_sasl_type
doveconf auth_mechanisms
```

On the Windows server, the NTLM log records connections with the negotiated flags starting with Windows Server 2019. Event 4022 is the client entry:

```powershell
Get-WinEvent -LogName 'Microsoft-Windows-NTLM/Operational' -MaxEvents 50 |
  Where-Object { $_.Id -in 4022, 4023 } |
  Format-List TimeCreated, Id, Message
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-LogName 'Microsoft-Windows-NTLM/Operational'` | Reads the operational NTLM log |
| `-MaxEvents 50` | Limits output to the 50 most recent entries |
| `Where-Object { $_.Id -in 4022, 4023 }` | Filters for audit events for client (4022) and server (4023) |
| `Format-List TimeCreated, Id, Message` | Displays the timestamp, event ID, and text with the Negotiated Flags |

</details>

An entry at the time of the sending attempt with the mail server as its destination confirms that `SmtpClient` used NTLM.

## Workaround 1: Force AUTH LOGIN in the script

`Send-MailMessage` does not provide a parameter to specify the authentication mechanism. Calling the `SmtpClient` class directly can do so: a `CredentialCache` provides credentials only for a specific mechanism. If only `Login` is configured, `SmtpClient` skips NTLM and uses `AUTH LOGIN`. The following script is based on the reader's example from the Born article:

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
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Import-Clixml -Path …` | Loads a PSCredential previously saved with `Get-Credential \| Export-Clixml`; bound to the user and computer through DPAPI |
| `$cache.Add($server, $port, 'Login', …)` | Stores the credentials only for the host, port, and mechanism `Login` |
| `SmtpClient($server, $port)` | Creates the SMTP client for the server and submission port |
| `EnableSsl = $true` | Switches to TLS via STARTTLS before transmitting credentials |
| `Credentials = $cache` | Passes the CredentialCache instead of a simple NetworkCredential |
| `BodyEncoding = [System.Text.Encoding]::UTF8` | Encodes the text as UTF-8 so umlauts are delivered correctly |

</details>

`EnableSsl` is mandatory here: `AUTH LOGIN` transmits the username and password only Base64-encoded, meaning in plaintext without TLS. The workaround requires a change to each script, but does not change the system configuration and leaves the update's NTLM protection intact.

## Workaround 2: Disable NTLM on the mail server

If you operate the Postfix server yourself, you can fix the problem centrally: remove `ntlm` from `auth_mechanisms` in the Dovecot configuration (usually `/etc/dovecot/conf.d/10-auth.conf`), leaving only `plain login`, then reload Dovecot:

```bash
sudo systemctl reload dovecot
```

For Cyrus SASL, `mech_list: plain login` in the file `smtpd.conf` in the SASL configuration directory restricts the mechanisms accordingly. Postfix then no longer announces NTLM, and `Send-MailMessage` falls back to `LOGIN` without script changes. First check whether other clients depend on NTLM, such as older Outlook profiles with the "Secure Password Authentication (SPA)" option. For TLS submission, NTLM provides no security benefit over `PLAIN` and `LOGIN`.

## Workaround 3: Registry, temporary only

KB5135557 documents a registry value that prevents the client from offering signing:

```powershell
reg add 'HKLM\SYSTEM\CurrentControlSet\Control\Lsa\MSV1_0' /v ClientAllowUnsignedNtlm /t REG_DWORD /d 1 /f
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `HKLM\…\Lsa\MSV1_0` | Key for the NTLM authentication package |
| `/v ClientAllowUnsignedNtlm` | Value name; `0` (default) offers signing, while `1` restores the old behavior |
| `/t REG_DWORD` | Value data type |
| `/d 1` | Sets the value to 1 |
| `/f` | Overwrites without prompting |

</details>

According to Microsoft, no restart is required. However, the value applies to all NTLM connections on the server, not just mail sending, and thus removes the update's relay protection for the entire system. Microsoft describes it as a compatibility measure only and reserves the right to remove the registry values later. It can at most bridge the time until workaround 1 or 2 is implemented. Uninstalling KB5131085 is not an alternative because it removes all security fixes from the November Patch Tuesday.

## In the medium term: Replace Send-MailMessage

Microsoft has listed `Send-MailMessage` as obsolete for years and advises against its use because the cmdlet does not guarantee secure connections. The .NET documentation also recommends other libraries such as MailKit for new development instead of `SmtpClient`. For scripts that send mail through Exchange Online, the supported approach is the Microsoft Graph API with `Send-MgUserMail` and app registration; setup is described in the article [Accessing Exchange Online with PowerShell: Replacing EWS with Microsoft Graph](/blog/microsoft-graph-powershell-postfach-anbindung). At the latest with phase 2 in May 2027, it is worthwhile to inventory which scripts and services still use NTLM: audit events 4022 and 4023 provide the list.

## Sources

1.  [Windows Server 2019: KB5131085 disrupts mail sending with PowerShell (Born's IT and Windows Blog)](https://borncity.com/blog/2026/11/17/windows-server-2019-kb5131085-stoert-mail-versand-per-powershell/): Reader report from November 17, 2026, including the symptom, uninstallation test, CredentialCache script, and the note about NTLM signing in the comments.

2.  [KB5131085 (OS Build 17763.9487)](https://support.microsoft.com/help/5131085): Changelog for the November update for Windows Server 2019 and Windows 10 Enterprise LTSC 2019, including the NTLM entry and the Known Issue for error 0x80041008.

3.  [KB5135557: NTLM signing security improvements in Windows](https://support.microsoft.com/en-us/help/5135557): Flags, timeline for the two phases, audit events 4022/4023, and the ClientAllowUnsignedNtlm and ServerRequireNtlmSign registry values.

4.  [SmtpAuthenticationManager.cs (.NET Framework Reference Source)](https://github.com/microsoft/referencesource/blob/master/System/net/System/Net/mail/SmtpAuthenticationManager.cs): Registration order of the SMTP authentication modules Negotiate, NTLM, Digest, and Login in SmtpClient.

5.  [Send-MailMessage (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/send-mailmessage): Note that the cmdlet is obsolete and does not guarantee secure connections.

6.  [SmtpClient Class (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/api/system.net.mail.smtpclient): Recommendation to use MailKit or other libraries for new development.

7.  [Postfix SASL Howto](https://www.postfix.org/SASL_README.html): SMTP authentication in Postfix through Cyrus SASL or Dovecot, including restrictions on the offered mechanisms.

8.  [Dovecot: Authentication Mechanisms Overview](https://doc.dovecot.org/main/core/config/auth/mechanisms/overview.html): Current list of SASL mechanisms in Dovecot, which no longer includes NTLM.
