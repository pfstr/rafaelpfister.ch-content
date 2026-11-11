---
title: "RDP sign-in fails: Password must be changed before first sign-in"
navTitle: "RDP: Change password"
description: "A new account with an initial password, and the Remote Desktop connection terminates with “You must change your password before logging on the first time.” The cause is Network Level Authentication, which does not support password changes. The password can still be changed: through another Windows session, from your own PC, using PowerShell, or through RD Web Access."
date: "2026-11-11"
kategorie: "Windows client"
timeToRead: "6 min read"
themen:
  - windows-client
  - active-directory-entra
produkte:
  - "windows-client"
  - "entra"
protokolle:
  - "troubleshooting"
  - "powershell"
slug: "rdp-sign-in-fails-password-must-be-changed-before-first-sign-in"
translationId: "article-96f19bf85c2f7ccd"
draft: false
translationOf: rdp-kennwort-aendern-nla
url: https://rafaelpfister.ch/en/blog/rdp-sign-in-fails-password-must-be-changed-before-first-sign-in
translationSourceHash: 016256f24e5c0c258d7cbf225d3f62ac577cf37c42844ad4a5f5084f85e7e8bf
translationModel: gpt-5.6-terra
translatedAt: 2026-11-11T13:48:59.383Z
translationReview: automatic
---

# RDP sign-in fails: Password must be changed before first sign-in

I recently encountered the situation again with a newly configured account in a customer environment: The initial password is correct, but the Remote Desktop connection is still terminated immediately. The Windows client `mstsc` displays the following message:

```text
You must change your password before logging on the first time.
Please update your password or contact your system administrator
or technical support.
```

Other RDP clients and connection managers indicate the same condition in their own wording, for example, “The user password must be changed before logging on for the first time.” For an expired password, the message is “This user account's password has expired. The password must change in order to logon.” The cause and solution are the same in both cases.

In short: The password cannot be changed through the RDP connection in this situation. It must be changed beforehand by another means, and in most cases, a network connection to a Domain Controller is sufficient.

## Cause: NLA authenticates before the session is established

The account has the **User must change password at next logon** option set in Active Directory (technically: `pwdLastSet` = 0). This is not a problem at a workstation in the domain: The sign-in screen recognizes the condition and immediately asks for the new password.

With RDP, this step is missing as soon as Network Level Authentication (NLA) is enabled. NLA authenticates the user through CredSSP before a session with a sign-in screen is created on the target system. Authentication with a password that must be changed fails, and no password-change dialog is provided at this point. The client terminates the connection and displays only the message. NLA is the default in current Windows versions and is enforced by Group Policy in hardened environments.

## Options for users

There are four ways to change the password without using the affected RDP connection. Which one is suitable depends on what can be reached from the user's own workstation.

| Option | Requirement |
|---|---|
| 1. Password portal or service desk | provided by the operator |
| 2. Existing Windows session in the domain | access with another account or to another system |
| 3. Password dialog on your own PC | Domain Controller reachable, for example through VPN |
| 4. PowerShell with `NetUserChangePassword` | TCP 445 to the Domain Controller reachable |

### Option 1: Password portal or service desk

In larger environments, there is usually a designated route for external staff: a self-service portal, a password feature in the VPN or Citrix portal, or a PAM system. If none is available, the service desk can remove the flag or set a password without requiring it to be changed (see the section for administrators).

### Option 2: Existing Windows session in the domain

If a session already exists on a system in the domain, such as on a jump host or with a second account, the password can be changed there. In an RDP session, `Ctrl+Alt+End` opens the security screen of the remote system (`Ctrl+Alt+Del` would be processed on the local PC). Under **Change a password**, the username field can be edited: Enter the affected account there in the form `DOMAIN\benutzer`, followed by the initial password and the new password twice.

### Option 3: Password dialog on your own PC

The same dialog also works on a PC that is not a member of the domain, provided it can reach a Domain Controller, for example through a VPN connection. Press `Ctrl+Alt+Del`, choose **Change a password**, and replace the prefilled local username with `DOMAIN\benutzer`. The PC must be able to resolve the domain name through DNS; without the DNS servers of the target environment, Windows cannot find a Domain Controller.

### Option 4: PowerShell with NetUserChangePassword

The Windows API `NetUserChangePassword` changes a password directly on a Domain Controller. It verifies the previous password during the change itself and does not require prior sign-in. It therefore also works when the password has expired or must be changed, including from a PC outside the domain. The script imports the function through P/Invoke and prompts for both passwords without displaying them:

```powershell
$api = Add-Type -Namespace Win32 -Name NetApi32 -PassThru -MemberDefinition @'
[DllImport("netapi32.dll", CharSet = CharSet.Unicode)]
public static extern uint NetUserChangePassword(
    string domainname, string username,
    string oldpassword, string newpassword);
'@

$dc   = 'dc01.example.com'
$user = 'm.muster'
$alt  = Read-Host -AsSecureString 'Bisheriges Kennwort'
$neu  = Read-Host -AsSecureString 'Neues Kennwort'

$klartext = { param($s) [System.Net.NetworkCredential]::new('', $s).Password }
$rc = $api::NetUserChangePassword(
    $dc, $user, (& $klartext $alt), (& $klartext $neu))

if ($rc -eq 0) { 'Kennwort geändert' } else { "Fehler $rc" }
```

<details class="options-details">
<summary>Options explained</summary>

| Argument | Effect |
|---|---|
| `domainname` | DNS or NetBIOS name of a Domain Controller or the domain. From a PC outside the domain, specify the Domain Controller directly (name or IP address). |
| `username` | Sign-in name of the account (`sAMAccountName`), without the domain portion. |
| `oldpassword` | Previous password, in this case the initial password. It must match exactly. |
| `newpassword` | New password. It must meet the domain password policy. |

</details>

A return value of 0 indicates success. The most common error codes are:

| Code | Meaning |
|---|---|
| 5 | Access denied, for example because the Domain Controller does not allow the call without authentication |
| 86 | The previous password is incorrect |
| 2221 | The username was not found |
| 2245 | The new password does not meet the password policy (length, complexity, history, minimum age) |
| 2351 | The specified server name is invalid |

The call uses SMB and requires TCP port 445 to the Domain Controller. In heavily segmented networks, this port is often blocked from client networks or VPNs; in that case, option 1 remains.

`Set-ADAccountPassword` from the Active Directory module does not help the affected user in this situation: The cmdlet requires an authenticated connection to Active Directory Web Services, and that very sign-in is not possible with the affected account. It is the tool for the administrator.

## Disable NLA in the client: only without enforced NLA

It is often recommended to disable CredSSP in the .rdp file:

```text
enablecredsspsupport:i:0
```

The client then connects without NLA and receives the classic sign-in screen of the target system, where the password can be changed. This works only if the target system accepts connections without NLA. If **Allow connections only from computers running Remote Desktop with Network Level Authentication** is enabled there, it rejects the connection. This is the default in current Windows Server versions.

## Options for administrators

On the administration side, the problem can be prevented or resolved:

- **Remove the flag:** `Set-ADUser -Identity m.muster -ChangePasswordAtLogon $false` removes the requirement to change the password. The user can sign in with the initial password and change it during the session using `Ctrl+Alt+End`. The initial password remains valid until then and must therefore be transmitted through a secure channel.
- **Reset the password:** `Set-ADAccountPassword -Identity m.muster -Reset` sets a new password. Then check whether the flag is still set.
- **RD Web Access:** If the role is available, it can offer password changes for expired passwords. In IIS Manager, under **Sites → Default Web Site → RDWeb → Pages**, set the value `PasswordChangeEnabled` to `true` in **Application Settings**. The password-change page is located at `/RDWeb/Pages/en-US/password.aspx`.
- **Accounts for RDP-only access:** For users who work exclusively through RDP (external service providers, administrators on jump hosts), the requirement to change the password at first sign-in cannot be fulfilled without one of the methods above. The process for the first password change should therefore be part of onboarding.

Disabling NLA on the target system also resolves the symptom, but removes a layer of protection: Without NLA, every client on the network can reach the sign-in screen and thus the RDP service before authenticating. Microsoft advises against this.

## Sources

1.  [Remote Desktop Client Troubleshooting: Disconnect Codes and Reasons](https://learn.microsoft.com/en-us/archive/technet-wiki/37870.remote-desktop-client-troubleshooting-disconnect-codes-and-reasons): List of RDP client disconnect codes, including 4615 (password must be changed before first sign-in) and 3847 (password expired), with the message text.

2.  [NetUserChangePassword function (lmaccess.h)](https://learn.microsoft.com/en-us/windows/win32/api/lmaccess/nf-lmaccess-netuserchangepassword): API reference with parameters, permissions, and return values.

3.  [How to change your own expired password when you can't login to RDP](https://evotec.xyz/how-to-change-your-own-expired-password-when-you-cant-login-to-rdp/): Practical report by Evotec on changing a password through NetUserChangePassword from a PC outside the domain, including a note about TCP 445 and the ineffective enablecredsspsupport entry when NLA is enforced.

4.  [Supported RDP properties](https://learn.microsoft.com/en-us/azure/virtual-desktop/rdp-properties): Reference for .rdp properties, including enablecredsspsupport with values and default.

5.  [Remote Desktop Web Access troubleshooting](https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/remote-desktop-web-access-troubleshooting): Section on changing expired passwords through RD Web Access and the PasswordChangeEnabled setting.

6.  [Set-ADAccountPassword](https://learn.microsoft.com/en-us/powershell/module/activedirectory/set-adaccountpassword): Cmdlet reference with the Reset, OldPassword, NewPassword, Credential, and Server parameters.

7.  [User can't authenticate or must authenticate twice](https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/cannot-authenticate-must-authenticate-twice): Microsoft guidance on sign-in failures with NLA; disabling NLA is marked as not recommended.

8.  [Change password in another domain](https://bartwullems.blogspot.com/2019/12/change-password-in-another-domain.html): Blog post on the editable username field in the “Change a password” dialog, which can be used to change the password of an account in another domain.

9.  [Allow users to reset expired password via RD Web Access](https://woshub.com/allow-users-to-reset-expired-password-via-rd-webaccess-windows-server-2012/): Windows OS Hub guide to the PasswordChangeEnabled setting and the password.aspx page path.
