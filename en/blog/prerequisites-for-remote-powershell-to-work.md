---
title: "Requirements for Remote PowerShell to Work"
navTitle: "Remote PowerShell"
description: "PowerShell remoting rarely fails because of the command itself, but rather because of the prerequisites: the WinRM service, listener, firewall, authentication, and the specifics of local accounts. What must be configured on the target and client sides, how to check it with Test-WSMan, and why Access denied usually has nothing to do with the password."
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
translationOf: remote-powershell-voraussetzungen
translationSourceHash: edebb6e6be909c38c0d70f1bf09aaac37328f86bf2def4e23a26a34456b3492f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T09:56:33.204Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/prerequisites-for-remote-powershell-to-work
---

# Requirements for Remote PowerShell to Work

`Invoke-Command` and `Enter-PSSession` are quick to type, but the connection is established only when the prerequisites are met on both sides. PowerShell remoting is based on WS-Management (WinRM), a SOAP-based management service over HTTP. When a session fails, it is almost never due to the cmdlet itself, but rather a missing service, a closed port, a firewall rule, or authentication. This article walks through the prerequisites one by one and shows you how to check each of them.

First, the terminology: The target computer is the computer on which the commands are to run; the client is the computer from which you connect. By default, WinRM listens on port 5985 (HTTP) and, when configured, on port 5986 (HTTPS). HTTP traffic on 5985 is encrypted at the message level as soon as authentication uses Kerberos or NTLM.

## The cmdlets at a glance

For reference, here are the cmdlets used in this article:

<details class="options-details">
<summary>Options at a glance</summary>

| Cmdlet | Purpose |
|---|---|
| `Enable-PSRemoting` | Configures WinRM on the target side: service, listener, firewall rule |
| `Test-WSMan` | Checks whether the remote computer's WinRM service responds |
| `Enter-PSSession` | Opens an interactive remote session to a computer |
| `Invoke-Command` | Runs a command block on one or more computers |
| `Set-Item WSMan:\localhost\Client\TrustedHosts` | Adds trusted remote computers for authentication outside a domain |
| `Get-Service WinRM` | Displays the WinRM service status and startup type |

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

`Enable-PSRemoting` starts the WinRM service, sets its startup type to automatic, creates an HTTP listener, and adds the appropriate firewall rule. One caveat concerns the network profile: If a network adapter is classified as public, the command refuses to configure it by default. On servers or in controlled networks, `-SkipNetworkProfileCheck` allows the configuration to proceed anyway.

The scope of the firewall rule is important. For public network profiles, the default rule restricts access to the local subnet. If you connect over another network, such as a VPN, this restriction applies and the connection fails despite the service running. Open the rule specifically for the required address range, not broadly for all addresses:

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

The WinRM service must be running on the client, otherwise even setting configuration values fails. Check this first:

```powershell
Get-Service WinRM
```

If the service is Stopped, start it with `Start-Service WinRM` (administrator privileges required). The startup type is often manual on clients, so the service stops again after a restart. If you access remote computers regularly from this machine, set the startup type to automatic.

The second point concerns authentication outside a domain. If you connect by IP address or in a workgroup, the client cannot verify the remote computer through Kerberos and falls back to NTLM. For security reasons, WinRM refuses this until the remote computer has been added as trusted. Add the target address to TrustedHosts (administrator privileges required):

```powershell
Set-Item WSMan:\localhost\Client\TrustedHosts -Value '100.105.207.14' -Force
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Value <Liste>` | Trusted remote computers (IP address or name), multiple entries separated by commas, `*` as a wildcard |
| `-Force` | Sets the value without prompting |
| `-Concatenate` | Appends to the existing list instead of replacing it |

</details>

TrustedHosts is a client setting, not a target computer setting, and affects client security: The listed remote computers are considered trusted without their identity being cryptographically verified. Therefore, add specific addresses rather than the wildcard `*`. In a domain with Kerberos, the entry is not required; the proper approach outside a domain without TrustedHosts is an HTTPS listener with a certificate trusted by the client.

## Authentication: Why Access denied is rarely caused by the password

A common issue with local accounts is the Access denied message even though the password is correct. The reason is remote UAC filtering: With local accounts (other than the built-in Administrator), Windows removes administrative privileges by default for access over the network. Sign-in succeeds, but every action requiring elevated privileges is denied. If the remote computer reports Access denied rather than invalid credentials, this is the likely cause.

You can fix this on the target computer with a registry value that grants local administrators full rights over the network:

```powershell
Set-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System' -Name LocalAccountTokenFilterPolicy -Value 1 -Type DWord
```

This is a deliberate relaxation: Local administrator accounts are thereby granted full rights over the network. Set the value only in controlled networks and with strong passwords. In a domain, it is better to use a domain account, in which case this issue does not arise.

When connecting, enter the username for local accounts preceded by the computer name so that the target system resolves the account locally:

```powershell
$cred = Get-Credential
Enter-PSSession -ComputerName 100.105.207.14 -Credential $cred
```

In the sign-in dialog, enter the user as `RECHNERNAME\Benutzer` and domain accounts as `DOMAENE\Benutzer`. A PIN used for Windows sign-in does not work over the network; the account password is required. For a Microsoft account, this is its password, and the account name may differ from the display name.

## Check in the right order

Narrow down issues from the bottom up, and you will quickly see which prerequisite is missing.

First, check whether the port is reachable:

```powershell
Test-NetConnection -ComputerName 100.105.207.14 -Port 5985
```

If the port does not respond, the listener is missing or the firewall is blocking it. If it responds, check the remote computer's WinRM service:

```powershell
Test-WSMan -ComputerName 100.105.207.14
```

A response containing the protocol version and vendor means that the service and listener are in place. Only then test with credentials:

```powershell
Invoke-Command -ComputerName 100.105.207.14 -Credential $cred -ScriptBlock { $env:COMPUTERNAME }
```

If this call returns the remote computer's name, all prerequisites are met.

## Common errors and their causes

| Message or symptom | Likely cause | Approach |
|---|---|---|
| Port 5985 unreachable | No listener or firewall is blocking it | Run `Enable-PSRemoting`, check the firewall rule and its scope |
| WinRM cannot complete the operation | Service on the target side is stopped, or access is allowed only from the local subnet | Start the service, open the firewall rule for the required address range |
| The WinRM client cannot process the request … TrustedHosts | Non-domain connection without a TrustedHosts entry | Add the target address to TrustedHosts on the client, or use HTTPS |
| Access is denied (despite the correct password) | Remote UAC filtering with a local account | Set `LocalAccountTokenFilterPolicy` to 1, or use a domain account |
| Access to a second resource fails in the session | Double hop: credentials are not forwarded | Perform the task directly on the target, or use CredSSP or delegated credentials |

## Limitations: The double-hop problem

One limitation remains even with a complete configuration and can only be worked around: By default, a remote session cannot forward your credentials to a third system. If you access a network share or another server from a session on the target computer, it fails because credentials are missing. This double-hop problem is a security feature, not a misconfiguration. For most support tasks, it is sufficient to run the command directly on the target computer. Where forwarding is truly required, CredSSP or constrained delegation come into play, both with their own security trade-offs.

## Change log

- **October 25, 2026:** Updated the source link to the new Microsoft Learn address.

## Sources

1.  [about_Remote_Requirements (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements): Requirements for PowerShell remoting, permissions, and network profiles.

2.  [Enable-PSRemoting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/enable-psremoting): What the command configures, including the network profile caveat and firewall rule.

3.  [about_Remote_Troubleshooting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_troubleshooting): TrustedHosts, authentication outside the domain, and common error messages.

4.  [Making the second hop in PowerShell Remoting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/ps-remoting-second-hop): Cause of the double-hop problem and the solutions with their trade-offs.
