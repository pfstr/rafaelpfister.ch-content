---
title: "Prerequisites for Remote PowerShell to Work"
navTitle: "Remote PowerShell"
description: "PowerShell remoting rarely fails because of the command itself, but rather because of prerequisites: the WinRM service, listener, firewall, authentication, and the specifics of local accounts. What needs to be configured on the target and client sides, how to check it with Test-WSMan, and why Access denied usually has nothing to do with the password."
date: "2026-09-01"
kategorie: "Windows and PowerShell"
timeToRead: "10 min read"
themen:
  - windows-client
produkte:
  - "windows-client"
protokolle:
  - "powershell"
  - "haertung"
slug: "prerequisites-for-remote-powershell-to-work"
translationId: "article-7315c1ae9e67a24d"
aiPrompt: |
  Du bist mein Windows-Administrationsassistent. Hilf mir, PowerShell-Remoting (WinRM) zwischen zwei Rechnern einzurichten und Fehler einzugrenzen: Dienst und Listener auf der Zielseite, Firewall, TrustedHosts auf der Clientseite, Authentisierung bei Domänen- und lokalen Konten, und die Prüfung mit Test-WSMan.
translationOf: remote-powershell-voraussetzungen
url: https://rafaelpfister.ch/en/blog/prerequisites-for-remote-powershell-to-work
translationSourceHash: 2969f02b5e677daaea867ea7c19fe929dc58f628cc4e47f3b165e85329836464
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T08:45:26.532Z
translationReview: automatic
---

# Prerequisites for Remote PowerShell to Work

`Invoke-Command` and `Enter-PSSession` are quick to type, but the connection is only established once the prerequisites are met on both sides. PowerShell remoting is based on WS-Management (WinRM), a SOAP-based management service over HTTP. When a session fails, it is almost never due to the cmdlet itself, but rather to a missing service, a closed port, a firewall rule, or authentication. This article walks through the prerequisites in order and shows how to check each one individually.

First, the terminology: the target computer is the computer on which the commands are to run; the client is the computer from which you connect. By default, WinRM listens on port 5985 (HTTP) and, if configured, port 5986 (HTTPS). HTTP traffic on 5985 is encrypted at the message level once authentication uses Kerberos or NTLM.

## Cmdlets at a glance

For reference, here are the cmdlets covered in this article:

<details class="options-details">
<summary>Options at a glance</summary>

| Cmdlet | Purpose |
|---|---|
| `Enable-PSRemoting` | Configures WinRM on the target side: service, listener, firewall rule |
| `Test-WSMan` | Checks whether the remote computer's WinRM service responds |
| `Enter-PSSession` | Opens an interactive remote session to a computer |
| `Invoke-Command` | Runs a command block on one or more computers |
| `Set-Item WSMan:\localhost\Client\TrustedHosts` | Adds trusted remote computers for authentication outside a domain |
| `Get-Service WinRM` | Shows the status and startup type of the WinRM service |

</details>

## Target side: Configure WinRM

On the target computer, a single command configures the service, listener, and firewall rule. Run it in PowerShell with administrator privileges:

```powershell
Enable-PSRemoting -Force
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Force` | Runs without prompting |
| `-SkipNetworkProfileCheck` | Configures remoting even when a network connection is classified as public |

</details>

`Enable-PSRemoting` starts the WinRM service, sets its startup type to automatic, creates an HTTP listener, and adds the appropriate firewall rule. One caveat concerns the network profile: if a network adapter is classified as public, the command refuses to configure remoting by default. On servers or in controlled networks, `-SkipNetworkProfileCheck` allows setup to proceed anyway.

The scope of the firewall rule is important. For public network profiles, the default rule restricts access to the local subnet. If you connect over a different network, such as a VPN, this restriction applies and the connection fails even though the service is running. In that case, open the rule specifically for the required address range, rather than broadly for all addresses:

```powershell
Set-NetFirewallRule -Name 'WINRM-HTTP-In-TCP*' -RemoteAddress 100.64.0.0/10
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Name 'WINRM-HTTP-In-TCP*'` | Selects the WinRM HTTP rules created by Enable-PSRemoting using the name pattern |
| `-RemoteAddress <Bereich>` | Restricts permitted source addresses to the specified range (here, a CIDR block); `Any` permits any address |

</details>

## Client side: TrustedHosts and service

The WinRM service must be running on the client; otherwise, even setting configuration values will fail. Check that first:

```powershell
Get-Service WinRM
```

If the service is Stopped, start it with `Start-Service WinRM` (administrator privileges required). On clients, the startup type is often manual, so the service stops again after a restart. If you access systems regularly from this computer, set the startup type to automatic.

The second point concerns authentication outside a domain. If you connect by IP address or in a workgroup, the client cannot verify the remote computer through Kerberos and falls back to NTLM. For security reasons, WinRM refuses this unless the remote computer has been added as trusted. Add the target address to TrustedHosts (administrator privileges required):

```powershell
Set-Item WSMan:\localhost\Client\TrustedHosts -Value '100.105.207.14' -Force
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Value <Liste>` | Trusted remote computers (IP address or name), multiple entries separated by commas, with `*` as a wildcard |
| `-Force` | Sets the value without prompting |
| `-Concatenate` | Appends to the existing list instead of replacing it |

</details>

TrustedHosts is a setting on the client, not the target computer, and affects client security: the listed remote computers are considered trusted without their identities being cryptographically verified. Therefore, add specific addresses rather than the `*` wildcard. In a domain with Kerberos, the entry is not necessary; the proper approach outside a domain without TrustedHosts is an HTTPS listener with a certificate trusted by the client.

## Authentication: why Access denied is rarely about the password

A common issue with local accounts is an Access denied message even though the password is correct. The reason is remote UAC filtering: for local accounts (other than the built-in Administrator), Windows removes administrative privileges by default when access is made over the network. The sign-in succeeds, but every action requiring elevated privileges is denied. If the remote computer reports Access denied rather than invalid credentials, this is the likely reason.

You can fix this on the target computer with a registry value that gives local administrators full privileges over the network:

```powershell
Set-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System' -Name LocalAccountTokenFilterPolicy -Value 1 -Type DWord
```

This is an intentional relaxation: local administrator accounts receive full privileges over the network. Set this value only in controlled networks and with strong passwords. In a domain, use a domain account instead, and this issue does not arise.

When connecting, specify the user name for local accounts with the computer name in front of it so that the target system resolves the account locally:

```powershell
$cred = Get-Credential
Enter-PSSession -ComputerName 100.105.207.14 -Credential $cred
```

In the sign-in dialog, enter the user as `RECHNERNAME\Benutzer`; for domain accounts, use `DOMAENE\Benutzer`. A PIN used for Windows sign-in does not work over the network; you need the account password. For a Microsoft account, this is its password, and the account name may differ from the display name.

## Check in the right order

Narrow down errors from the bottom up to quickly see which prerequisite is missing.

First, check whether the port is reachable:

```powershell
Test-NetConnection -ComputerName 100.105.207.14 -Port 5985
```

If the port does not respond, the listener is missing or the firewall is blocking it. If it responds, check the WinRM service on the remote computer:

```powershell
Test-WSMan -ComputerName 100.105.207.14
```

A response with a protocol version and manufacturer means that the service and listener are in place. Only then test with credentials:

```powershell
Invoke-Command -ComputerName 100.105.207.14 -Credential $cred -ScriptBlock { $env:COMPUTERNAME }
```

If this call returns the name of the remote computer, all prerequisites have been met.

## Common errors and their causes

| Message or symptom | Likely cause | Approach |
|---|---|---|
| Port 5985 unreachable | No listener or firewall is blocking it | Run `Enable-PSRemoting`; check the firewall rule and scope |
| WinRM cannot complete the operation | Service on the target side is stopped, or access is allowed only from the local subnet | Start the service; open the firewall rule for the required address range |
| The WinRM client cannot process the request … TrustedHosts | Non-domain connection without a TrustedHosts entry | Add the target address to TrustedHosts on the client, or use HTTPS |
| Access is denied (despite the correct password) | Remote UAC filtering for a local account | Set `LocalAccountTokenFilterPolicy` to 1, or use a domain account |
| Access to a second resource fails in the session | Double hop: credentials are not forwarded | Run the task directly on the target, or use CredSSP or delegated credentials |

## Limitations: the double-hop problem

One limitation remains even with a complete configuration and can only be worked around: by default, a remote session cannot forward your credentials to a third system. If you access a network share or another server in a session on the target computer, it fails because credentials are unavailable. This double-hop problem is a security feature, not a misconfiguration. For most support tasks, it is sufficient to run the command directly on the target computer. Where forwarding is truly necessary, CredSSP or constrained delegation come into play, both with their own security trade-offs.

## Sources

1.  [about_Remote_Requirements (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements): Prerequisites for PowerShell remoting, permissions, and network profiles.

2.  [Enable-PSRemoting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/enable-psremoting): What the command configures, including the network profile caveat and firewall rule.

3.  [about_Remote_Troubleshooting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_troubleshooting): TrustedHosts, authentication outside the domain, and common error messages.

4.  [Making the second hop in PowerShell Remoting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/scripting/learn/remoting/ps-remoting-second-hop): Cause of the double-hop problem and solution approaches with their trade-offs.
