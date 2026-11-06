---
title: "Totemomail license limit reached: clean up orphaned users via LDAP"
navTitle: "License limit reached"
description: "Disabled AD accounts remain in totemomail and continue to consume licenses. With a tested LDAPS connection and the cleanup agent, Active Directory becomes the authoritative source."
date: "2026-06-26"
kategorie: "Totemomail"
timeToRead: "9 min read"
themen:
  - totemomail
slug: "totemomail-licensed-user-limit-reached-ldap-cleanup"
translationOf: "totemomail-licensed-user-limit-ldap-cleanup"
translationId: article-cdc60310665049b8
translatedAt: 2026-11-06T03:24:56.659Z
translationReview: required
translationSourceHash: 3d7892da089472f95cc44e06e8ecb8654f53c40099d6ff970060102346823159
translationModel: gpt-5.6-terra
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/totemomail-licensed-user-limit-reached-ldap-cleanup
---

# Totemomail license limit reached: clean up orphaned users via LDAP

The *“The licensed user limit has been reached”* message does not mean that mail flow stops immediately. It indicates underlicensing. In long-running environments, the cause is usually not sudden growth, but former employees: the AD account was disabled, while the internal user in totemomail remained and continues to consume a license.

The sustainable solution is regular LDAP synchronization with Active Directory. The following steps set up the connection and cleanup agent and verify the entire path before the first production run. Host names, DNs, and service accounts containing `example.com` are placeholders and must be adapted to your own environment.

## Which users consume a license

Totemomail distinguishes between two user classes. Only internal users count toward the license limit.

| User type | Description | License-relevant |
| --- | --- | --- |
| Internal Users | Users in your own organization who send and receive encrypted messages | Yes |
| External Users | External communication partners (WebMail, PDF, S/MIME, PGP) | No |


An internal user is created as soon as they communicate through the gateway for the first time. This happens automatically. Removal, however, does not: when an employee leaves the organization, you typically disable the AD account. The totemomail entry remains. Over the years, this accumulates orphaned accounts that continue to consume licenses.

### Status display

You can find the current status under **Settings → Overview → User Information**.

![](../images/953te2zhdJ61lxda1mj04QrlQA.png)

*Available Users is set to* `*-17*`*. The 4017 internal users have fewer licensed seats available.*

The important rows:

-   **Internal users** (`4017`): created internal users
    
-   **Internal blocked users** (`14`): blocked but still license-relevant
    
-   **Available Users** (`-17`): available licenses; a negative value means underlicensing
    

As soon as *Available Users* falls below zero, you will see the warning at the bell:

![](../images/lcL4owxA3iEdg3L9ZFd2bIioE.png)

*“The licensed user limit has been reached.” Mail flow continues, but the message remains permanently visible.*

Important: underlicensing does not block mail flow. It is a licensing issue, not a technical condition. You therefore have time for a clean solution, but should not ignore the condition permanently.

## From immediate action to a long-term solution

### Manual deletion

You can search for and delete internal users individually under **Internal Users**. This resolves the immediate condition, but the problem returns after a few months. With several thousand accounts, this is not practical.

### LDAP integration with cleanup agent

The viable approach is integration with Active Directory via LDAP. An agent regularly compares internal users with the directory and removes or disables accounts that no longer exist in AD. This makes AD the authoritative source, and your AD offboarding process takes care of license hygiene at the same time.

## LDAP basics

| Term | Meaning |
| --- | --- |
| DN (Distinguished Name) | Unique path to an object, e.g., `CN=John Doe,OU=Users,DC=corp,DC=example,DC=com` |
| Base DN / Search Base | Root of the search, e.g., `DC=corp,DC=example,DC=com` |
| Bind DN | Account used by totemomail to authenticate to AD |
| Filter | LDAP search expression, e.g., `(&(objectClass=user)(sAMAccountName=jdoe))` |


### Ports

| Port | Protocol | Use |
| --- | --- | --- |
| 389 | LDAP | unencrypted / STARTTLS |
| 636 | LDAPS | LDAP over TLS |
| 3268 | Global Catalog | forest-wide search, unencrypted |
| 3269 | Global Catalog SSL | forest-wide search over TLS |


In a single-domain environment, port 636 to a domain controller is sufficient. If you operate a forest with multiple domains, only the Global Catalog (port 3269) provides forest-wide results. A DC on port 636 knows only the objects in its own domain and responds to searches outside its partition with a referral (a detail that is often overlooked in multi-domain environments).

### userAccountControl

Whether an AD account is disabled is stored in the `userAccountControl` bit field. The `ACCOUNTDISABLE` flag has the value `2`. Use the LDAP matching rule `1.2.840.113556.1.4.803` (`LDAP_MATCHING_RULE_BIT_AND`) to evaluate individual bits:

```text
# Aktive Benutzer
(&(objectClass=user)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))

# Deaktivierte Benutzer
(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))
```

## Step 1: Service account in AD

For the integration, create a dedicated account with read-only permissions. Do not use an administrator account for this. The bind user only needs to be able to read AD.

```powershell
New-ADUser -Name "svc-totemomail-ldap" `
  -SamAccountName "svc-totemomail-ldap" `
  -UserPrincipalName "svc-totemomail-ldap@corp.example.com" `
  -Path "OU=Service Accounts,DC=corp,DC=example,DC=com" `
  -AccountPassword (Read-Host -AsSecureString "Passwort") `
  -PasswordNeverExpires $true `
  -Enabled $true
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Name` | Display name and CN of the new account |
| `-SamAccountName` | Login name (pre-Windows 2000 logon name) |
| `-UserPrincipalName` | UPN in the format `benutzer@domäne` |
| `-Path` | Target OU as a Distinguished Name |
| `-AccountPassword` | Password as a SecureString; `Read-Host -AsSecureString` prompts for it securely at the console |
| `-PasswordNeverExpires $true` | Password does not expire |
| `-Enabled $true` | Create the account enabled immediately (the default would be disabled) |

</details>

A regular domain user can already read AD, so the account requires no additional permissions. For the password, use a long, random value and store it in your password vault.

If your security policy requires it, you can also use a gMSA (Group Managed Service Account). However, totemomail expects a bind DN and password, which is why in practice a traditional service account with `PasswordNeverExpires` is usually used.

## Step 2: Verify the LDAP connection on the command line

Before configuring anything in totemomail, verify the LDAP connection on the command line. This is the step most people skip. If `ldapsearch` works, the integration in totemomail will work too. If the test fails, you at least know where it fails instead of guessing in the totemomail GUI.

### 2.1 Port check

On Linux, for example from the totemomail appliance:

```bash
nc -vz dc01.corp.example.com 636
nmap -p 389,636,3268,3269 dc01.corp.example.com
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `nc -v` | Verbose output: reports whether connection establishment succeeds or fails |
| `nc -z` | Check connection establishment only; do not send data |
| `dc01.corp.example.com 636` | Target host and port for the check |
| `nmap -p 389,636,3268,3269` | List of TCP ports to check |
| `dc01.corp.example.com` | Target host for the port scan |

</details>

On Windows with PowerShell:

```powershell
Test-NetConnection -ComputerName dc01.corp.example.com -Port 636
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-ComputerName` | Target host for the connectivity test |
| `-Port` | TCP port to test, 636 here for LDAPS |

</details>

If no connection can be established here, you have a firewall or routing issue, not an LDAP issue.

### 2.2 Check the TLS certificate

In practice, LDAPS most often fails because of the certificate. Therefore, inspect what the DC delivers:

```bash
openssl s_client -connect dc01.corp.example.com:636 -showcerts </dev/null
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `s_client` | OpenSSL TLS test client: establishes the connection and displays handshake details |
| `-connect dc01.corp.example.com:636` | Target host and port for the TLS handshake |
| `-showcerts` | Displays the complete certificate chain provided by the server, not just the server certificate |
| `</dev/null` | Closes standard input so that `s_client` exits after the handshake instead of waiting for input |

</details>

Watch for two things:

-   `**subject=**` **/** `**issuer=**`: The host name in the certificate (CN or SAN) must match the host name used to connect. If you connect via IP address, verification fails when the certificate contains only the FQDN.
    
-   `**Verify return code: 0 (ok)**`: The issuing CA must be known to totemomail. With an internal Enterprise CA, you must import its root or issuing certificate into the totemomail trust store.
    

### 2.3 Bind and search with ldapsearch

`ldapsearch` belongs to `ldap-utils` (Debian/Ubuntu) or `openldap-clients` (RHEL):

```bash
ldapsearch -x \
  -H ldaps://dc01.corp.example.com:636 \
  -D "CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com" \
  -W \
  -b "DC=corp,DC=example,DC=com" \
  "(&(objectClass=user)(sAMAccountName=jdoe))" \
  dn sAMAccountName mail userAccountControl
```

<details class="options-details">
<summary>Options explained</summary>

| Flag | Meaning |
| --- | --- |
| `-x` | Simple authentication (bind DN and password) |
| `-H` | LDAP URI including scheme (`ldaps://`) and port |
| `-D` | Bind DN |
| `-W` | Prompt for password interactively |
| `-b` | Search base |
| afterward | Filter, followed by the attributes to return |

</details>

If the query returns the object with its attributes, the connection is working. Use the bit filter to determine how many accounts are disabled in AD:

```bash
ldapsearch -x -H ldaps://dc01.corp.example.com:636 \
  -D "CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com" -W \
  -b "DC=corp,DC=example,DC=com" \
  "(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))" \
  sAMAccountName | grep -c sAMAccountName
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-x`, `-H`, `-D`, `-W`, `-b` | as in the query above: simple bind over LDAPS with search base |
| `sAMAccountName` | only requested attribute; keeps output to one attribute line per match |
| `grep -c sAMAccountName` | counts lines containing this attribute, and thus the accounts found |

</details>

### 2.4 Tools on Windows

`**ldp.exe**` is Microsoft's graphical LDAP tool, available on every DC and included with RSAT. Connect using `Connection → Connect` (host, port 636, enable SSL), authenticate with `Connection → Bind`, and navigate the directory tree using the base DN via `View → Tree`.

Without RSAT, you can use the ADSI searcher in PowerShell:

```powershell
$searcher = [adsisearcher]"(&(objectClass=user)(sAMAccountName=jdoe))"
$searcher.SearchRoot = [adsi]"LDAP://dc01.corp.example.com/DC=corp,DC=example,DC=com"
$searcher.FindOne().Properties
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `[adsisearcher]"(…)"` | creates a `DirectorySearcher` with the specified LDAP filter |
| `SearchRoot` | Search starting point as an ADSI path: server plus base DN |
| `FindOne()` | returns the first match; `.Properties` displays its attributes |

</details>

With RSAT and the AD module, it is shorter:

```powershell
Get-ADUser -Server dc01.corp.example.com `
  -SearchBase "DC=corp,DC=example,DC=com" `
  -Filter "Enabled -eq '$true'" |
  Measure-Object
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Server` | Domain controller against which the query runs |
| `-SearchBase` | Root of the search as a Distinguished Name |
| `-Filter` | Filter in PowerShell syntax; enabled accounts only here |
| `Measure-Object` | Counts returned objects instead of listing them |

</details>

Classically via `dsquery`, available on every DC:

```bash
dsquery user -disabled -limit 0
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `user` | Search object type: user accounts |
| `-disabled` | Disabled accounts only |
| `-limit 0` | No limit on the number of results (default: 100) |

</details>

Only once one of these tests completes successfully should you proceed in totemomail.

## Step 3: Configure the LDAP connection in totemomail

Create the LDAP directory in the admin GUI under **Directories / LDAP**. Use exactly the values you tested beforehand:

| Field | Example value |
| --- | --- |
| Host / URL | `ldaps://dc01.corp.example.com:636` |
| Bind DN | `CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com` |
| Bind Password | Service account password |
| Base DN | `DC=corp,DC=example,DC=com` |
| User Filter | `(&(objectClass=user)(objectCategory=person))` |
| Login Attribute | `sAMAccountName` (alternatively `mail` or `userPrincipalName`) |


If you use LDAPS with an internal CA, you must import its root or issuing certificate into the totemomail trust store. Otherwise, the TLS handshake fails with “certificate verify failed,” even if `ldapsearch` previously worked with `-x`: in this form, `ldapsearch` does not strictly verify the certificate.

After saving, trigger the built-in test connection. It confirms the bind.

## Step 4: Create the cleanup agent

Under **Maintenance → Agents → Add**, create an agent of type **“Check presence of internal users in directories.”**

### 4.1 “Schedule” tab

![](../images/oSiutQSlKTW0tMY5HUtWCMGuXQ.png)

*Here, the agent runs monthly on the 1st at 12:30 a.m. Use “Agent runs on server” to specify the executing node in the cluster.*

| Field | Recommendation | Reason |
| --- | --- | --- |
| The agent should run | `monthly`, day `1`, `00:30` | outside business hours; monthly is sufficient for license hygiene |
| Agent enabled | enable only after the test run | see step 5 |
| Produced emails are not sent but cached in a queue | enable for the first run | test run without sending email |
| Agent runs on server | one cluster node | the job should run on one node only |


### 4.2 “Parameters” tab

![](../images/Y6XzxZWGYIcZoJnZkFL0vUHXxQ.png)

*The parameters control which internal users are deleted, disabled, or newly created.*

| Parameter | Recommendation | Effect |
| --- | --- | --- |
| Delete inactive users that are not found in a directory? | enable | Inactive internal users without an AD entry are deleted. This is the core of license cleanup. |
| Delete blocked users that are not found in a directory? | enable | Blocked internal users without an AD entry are also deleted |
| Delete administrators? | leave blank | Administrator accounts should not be deleted automatically |
| Only set users found in the defined groups to inactive | optional | Users are set to inactive instead of being deleted. A leading `!` excludes members of the specified group. Separate DNs with `;`. |
| Additional filter attribute | optional | Additional attribute for the directory search, e.g., `proxyAddresses` |
| Delete inactive/blocked users that are found in the defined groups | leave blank | applies only if the group parameter is set |
| Create users based on group membership | optional | Creates new internal users based on AD group membership. Separate multiple groups with `;`. |


Negation in the *“Only set users found in the defined groups to inactive”* field works through a `!` preceding a group DN. Members of this group are excluded from the action:

```text
CN=Mitarbeiter,OU=Groups,DC=corp,DC=example,DC=com;!CN=Dienstkonten,OU=Groups,DC=corp,DC=example,DC=com
```

In this example, users in the *Employees* group are set to inactive when absent from AD, while members of the *Service Accounts* group remain untouched.

## Step 5: Test run and validation

Do not let the agent run against production data without a test run. Instead, proceed in this order:

1.  **Enable queue mode**: using the *“Produced emails are not sent but cached in a queue”* option. The agent determines the planned actions without sending email.
    
2.  **Run manually** and evaluate the agent log: how many users would be affected, and are unexpected accounts such as functional mailboxes on the list?
    
3.  **Plausibility check against** `**ldapsearch**`: The number of users not found in AD should match your manual LDAP query.
    
4.  If the result is correct, disable queue mode, enable *Agent enabled*, and activate the schedule.
    
5.  After the first production run, check **Settings → Overview → User Information** again. *Available Users* should then be back in the positive range.
    

## Troubleshooting

| Symptom | Cause | Action |
| --- | --- | --- |
| `Can't contact LDAP server` | Port 636 unreachable / incorrect host | check with `Test-NetConnection` or `nc -vz`, and check the firewall |
| `Invalid credentials (49)` | Incorrect bind DN or password | Specify the bind DN as a complete DN, not as `user@domain` |
| `certificate verify failed` | CA unknown to the trust store | Import the root or issuing CA |
| Host name mismatch in TLS | Connecting via IP instead of FQDN | Use the certificate CN/SAN as the host |
| `Referral (10)` | Search crosses the domain boundary | Use the Global Catalog on port 3269 instead of a DC on 636 |
| Disabled users are not detected | Missing `userAccountControl`\-filter | Use bit matching rule `:1.2.840.113556.1.4.803:=2` |
| Agent deletes too many accounts | Filter too broad / incorrect base DN | Test in queue mode and restrict the base DN |


With the `-d 1` flag, `ldapsearch` provides debug output for connection establishment:

```bash
ldapsearch -d 1 -x -H ldaps://dc01.corp.example.com:636 ...
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-d 1` | Debug level 1: logs the connection establishment process, including the TLS handshake, to stderr |
| `-x`, `-H` | Simple bind and LDAP URI as in the queries above |

</details>

This lets you see whether the TLS handshake or only the bind fails. The totemomail GUI does not show this distinction behind its generic error message.

## Security

-   **Read-only service account.** The bind user requires read permissions only.
    
-   **LDAPS instead of LDAP.** Use port 636 or 3269. LDAP on port 389 transmits the bind password in clear text. Active Directory is increasingly enforcing secured connections through LDAP Channel Binding and Signing anyway.
    
-   **Password rotation.** `PasswordNeverExpires` is operationally practical. Document the account and rotate the password according to plan.
    
-   **Monitoring.** Monitor *Available Users* (ideally through alerting) instead of waiting for the bell warning.
    
-   **First run in queue mode.** An incorrect filter can affect a large number of accounts.
    

## The safe process in four steps

Reaching the license limit is not a technical defect, but the consequence of a missing offboarding process. The sustainable solution is regular synchronization with Active Directory as the authoritative source. The order is crucial:

1.  Verify the LDAP connection on the command line (`ldapsearch`, `openssl s_client`, `Test-NetConnection`)
    
2.  Configure the connection in totemomail
    
3.  Validate the agent in queue mode
    
4.  Put the agent into production
    

Anyone who follows this order resolves the immediate licensing issue and prevents it from recurring.

## Change log

- **October 25, 2026:** Source replaced: the former documentation page on totemo.com redirects to kiteworks.com; the Kiteworks page about continuing totemo as an Email Protection Gateway is now referenced.

## Sources

1.  [Kiteworks – “Unlocking the Power of Kiteworks Email Protection Gateway (EPG) with totemo Integration”](https://www.kiteworks.com/video-why-kiteworks-and-totemo-are-better-together/): Kiteworks' acquisition of totemo and continuation of the technology as Email Protection Gateway (EPG); the former download and documentation page on totemo.com now redirects to the kiteworks.com homepage.
    
2.  [Microsoft Learn – “UserAccountControl property flags”](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties): Meaning of the flags, including `ACCOUNTDISABLE` (0x0002) and `NORMAL_ACCOUNT`.
    
3.  [Microsoft Learn – “Search Filter Syntax”](https://learn.microsoft.com/en-us/windows/win32/adsi/search-filter-syntax): Bitwise LDAP filter using matching rule OID `1.2.840.113556.1.4.803` (LDAP\_MATCHING\_RULE\_BIT\_AND).
    
4.  [OpenLDAP – “ldapsearch” (man page)](https://www.openldap.org/software/man.cgi?query=ldapsearch): Call options (`-x`, `-H ldaps://`, `-D`, `-W`, `-b`) for binding and searching.
    
5.  [Microsoft Learn – “Service overview and network port requirements”](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/service-overview-and-network-port-requirements): LDAP ports 389/636 and Global Catalog ports 3268/3269.
