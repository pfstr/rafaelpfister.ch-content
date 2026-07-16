---
title: "Totemomail: \"The licensed user limit has been reached\" (Automatically clean up internal users via LDAP)"
navTitle: "License limit reached"
description: "Anyone who has been running a totemomail environment for years is familiar with this phenomenon: At some point, a red warning appears in the upper-right corner next to the bell icon—*“The licensed user limit has been reached.”* The system continues to run, but you’re suddenly operating in a state of **under-licensing**. In this article, we’ll break down totemomail’s licensing model, explain why the number of internal users grows unnoticed, and set up an LDAP connection step by step—including an automatic cleanup agent—along with the CLI tools you’ll need to thoroughly test the LDAP connection beforehand."
date: "2026-06-26"
kategorie: "Totemomail"
timeToRead: "9 min to read"
themen:
  - "totemomail"
slug: "totemomail-licensed-user-limit-reached-ldap-cleanup"
translationOf: "totemomail-the-licensed-user-limit-has-been-reached-–-interne-user-per-ldap-automatisch-aufräumen"
url: "https://rafaelpfister.ch/en/blog/totemomail-licensed-user-limit-reached-ldap-cleanup"
---

# Totemomail: "The licensed user limit has been reached" (Automatically clean up internal users via LDAP)

If you run a totemomail environment for several years, you will eventually see the message *“The licensed user limit has been reached."* occur. The system continues to run, but is in a state of under-licensing. The cause is almost always the failure to offboard internal users. This article explains the licensing model, shows how to set up an LDAP connection, and describes the Cleanup Agent, including the tests you should run beforehand at the command line.

The host names, DNs, and service accounts in this article are generic examples (`example.com`). Adjust them to suit your environment.

## Licensing Model

totemomail distinguishes between two classes of users, only one of which is relevant for licensing purposes.

| User type | Description | License-relevant |
| --- | --- | --- |
| Internal Users | Users of your own organization who send and receive encrypted mail | Yes |
| External Users | External communication partners (WebMail, PDF, S/MIME, PGP) | No |

An internal user is created the first time they communicate through the gateway. This happens automatically. Removal, however, is not automatic: When an employee leaves the organization, you typically deactivate their AD account—but the totemomail entry remains. Over the years, orphaned accounts accumulate in this way, continuing to occupy licenses.

### Status Indicator

You can find the latest information at **Settings → Overview → User Information**.

![](../images/953te2zhdJ61lxda1mj04QrlQA.png)

*"Available Users" says* `*-17*`*. The 4,017 internal users are matched by a smaller number of licensed seats.*

The key lines:

-   **Internal users** (`4017`) – created internal users
    
-   **Internal blocked users** (`14`) – blocked, but still subject to licensing
    
-   **Available Users** (`-17`) – available licenses; a negative value indicates under-licensing
    

As soon as *Available Users* drops below zero, you'll see the warning on the bell:

![](../images/lcL4owxA3iEdg3L9ZFd2bIioE.png)

*"The licensed user limit has been reached." Email traffic continues, but the message remains visible at all times.*

Important: Sublicensing does not block the flow of email. It is a licensing issue, not a technical one. This means you have time to find a proper solution, but you should not ignore this situation indefinitely.

## Possible Solutions

### Manual Deletion

You can search for internal users under **Internal Users** and delete them one by one. This resolves the immediate issue, but the problem will return after a few months. With several thousand accounts, you won't be happy with this approach.

### LDAP Connection with Cleanup Agent

The most reliable approach is to connect to Active Directory via LDAP. An agent regularly synchronizes internal users with the directory and removes or deactivates accounts that no longer exist in AD. This makes AD the primary source, and your offboarding process in AD takes care of license management at the same time. The rest of this article is dedicated to this approach.

## LDAP Basics

| Term | Meaning |
| --- | --- |
| DN (Distinguished Name) | Unique path to an object, e.g. `CN=John Doe,OU=Users,DC=corp,DC=example,DC=com` |
| Base DN / Search Base | Root of the search, e.g. `DC=corp,DC=example,DC=com` |
| Bind DN | Account totemomail uses to authenticate against AD |
| Filter | LDAP search expression, e.g. `(&(objectClass=user)(sAMAccountName=jdoe))` |

### Ports

| Port | Protocol | Usage |
| --- | --- | --- |
| 389 | LDAP | unencrypted / STARTTLS |
| 636 | LDAPS | LDAP over TLS |
| 3268 | Global Catalog | forest-wide search, unencrypted |
| 3269 | Global Catalog SSL | forest-wide search over TLS |

In a single-domain environment, port 636 is sufficient for communicating with a domain controller. If you are running a forest with multiple domains, only the Global Catalog (port 3269) will provide forest-wide results. A DC on port 636 is aware only of the objects in its own domain and responds to searches outside its partition with a referral—a detail that is often overlooked in multi-domain environments.

### userAccountControl

Whether an AD account is deactivated is indicated in the bit field `userAccountControl`. The flag `ACCOUNTDISABLE` has the value `2`. The LDAP matching rule `1.2.840.113556.1.4.803` (`LDAP_MATCHING_RULE_BIT_AND`) evaluates the individual bits:

```text
# Aktive Benutzer
(&(objectClass=user)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))

# Deaktivierte Benutzer
(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))
```

## Step 1: Service Account in AD

To set up the connection, create a dedicated account with read-only permissions. Do not use an administrator account for this—the Bind user must only be able to read the AD.

```powershell
New-ADUser -Name "svc-totemomail-ldap" `
  -SamAccountName "svc-totemomail-ldap" `
  -UserPrincipalName "svc-totemomail-ldap@corp.example.com" `
  -Path "OU=Service Accounts,DC=corp,DC=example,DC=com" `
  -AccountPassword (Read-Host -AsSecureString "Passwort") `
  -PasswordNeverExpires $true `
  -Enabled $true
```

A standard domain user can already read the AD, so the account does not need any additional permissions. For the password, we recommend using a long, random string that you store in your password manager.

If your security policy allows it, you can also use a gMSA (Group Managed Service Account). However, totemomail expects a Bind DN and password, which is why in practice a standard service account with `PasswordNeverExpires` is usually used.

## Step 2: Check the LDAP connection from the command line

Before you configure anything in totemomail, you should verify the LDAP connection from the command line. This is the step that most people skip. If `ldapsearch` works, the connection in totemomail will also work. If the test fails, at least you'll know where the problem lies, instead of having to guess in the totemomail GUI.

### 2.1 Port Check

On Linux, for example, from the totemomail appliance:

```bash
nc -vz dc01.corp.example.com 636
nmap -p 389,636,3268,3269 dc01.corp.example.com
```

On Windows using PowerShell:

```powershell
Test-NetConnection -ComputerName dc01.corp.example.com -Port 636
```

If a connection cannot be established here, you have a firewall or routing issue, not an LDAP issue.

### 2.2 Check the TLS Certificate

In practice, LDAPS most often fails because of the certificate. Therefore, check what the DC returns:

```bash
openssl s_client -connect dc01.corp.example.com:636 -showcerts </dev/null
```

Keep two things in mind:

-   `**subject=**` **/** `**issuer=**` – The hostname in the certificate (CN or SAN) must match the hostname you are connecting to. If you connect using the IP address, the validation will fail if the certificate contains only the FQDN.
    
-   `**Verify return code: 0 (ok)**` – The issuing CA must be recognized by totemomail. If you are using an internal enterprise CA, you must import its root or issuing certificate into the totemomail trust store.
    

### 2.3 Bind and Search with ldapsearch

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

| Flag | Meaning |
| --- | --- |
| `-x` | Simple authentication (bind DN and password) |
| `-H` | LDAP URI including scheme (`ldaps://`) and port |
| `-D` | Bind DN |
| `-W` | Prompt for the password interactively |
| `-b` | Search base |
| after that | the filter, followed by the attributes to return |

If the query returns the object along with its attributes, the connection is established. You can use the bit filter to determine how many accounts are disabled in AD:

```bash
ldapsearch -x -H ldaps://dc01.corp.example.com:636 \
  -D "CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com" -W \
  -b "DC=corp,DC=example,DC=com" \
  "(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))" \
  sAMAccountName | grep -c sAMAccountName
```

### 2.4 Tools in Windows

`**ldp.exe**` is Microsoft's graphical LDAP tool, which is available on every domain controller and is part of RSAT. You connect via `Connection → Connect` (Host, Port 636, enable SSL), authenticate using `Connection → Bind` and navigate via `View → Tree` through the directory tree using the base DN.

Without RSAT, you can achieve the same result in PowerShell using the ADSI Searcher:

```powershell
$searcher = [adsisearcher]"(&(objectClass=user)(sAMAccountName=jdoe))"
$searcher.SearchRoot = [adsi]"LDAP://dc01.corp.example.com/DC=corp,DC=example,DC=com"
$searcher.FindOne().Properties
```

It's quicker with RSAT and the AD module:

```powershell
Get-ADUser -Server dc01.corp.example.com `
  -SearchBase "DC=corp,DC=example,DC=com" `
  -Filter "Enabled -eq '$true'" |
  Measure-Object
```

Traditionally via `dsquery`, available on every DC:

```bash
dsquery user -disabled -limit 0
```

Only after one of these tests runs successfully should you proceed to totemomail.

## Step 3: Configure the LDAP connection in totemomail

You can configure the LDAP directory in the Admin GUI under **Directories / LDAP**. Be sure to use exactly the same values you tested earlier:

| Field | Example value |
| --- | --- |
| Host / URL | `ldaps://dc01.corp.example.com:636` |
| Bind DN | `CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com` |
| Bind Password | password of the service account |
| Base DN | `DC=corp,DC=example,DC=com` |
| User Filter | `(&(objectClass=user)(objectCategory=person))` |
| Login Attribute | `sAMAccountName` (alternatively `mail` or `userPrincipalName`) |

If you use LDAPS with an internal CA, you must import its root or issuing certificate into the totemomail trust store. Otherwise, the TLS handshake will fail with the error "certificate verify failed," even if `ldapsearch` with `-x` used to work – `ldapsearch` It does not strictly verify the certificate in this form.

After saving, trigger the built-in test connection. It confirms the binding.

## Step 4: Create a cleanup agent

Under **Maintenance → Agents → Add** Create an agent of type **“Check presence of internal users in directories"**.

### 4.1 The “Schedule" Tab

![](../images/oSiutQSlKTW0tMY5HUtWCMGuXQ.png)

*The agent runs here on the 1st of every month at 12:30 a.m. Use "Agent runs on server" to specify the node in the cluster that will execute the agent.*

| Field | Recommendation | Reason |
| --- | --- | --- |
| The agent should run | `monthly`, day `1`, `00:30` | outside business hours; monthly is sufficient for license hygiene |
| Agent enabled | enable only after the test run | see step 5 |
| Produced emails are not sent but cached in a queue | enable for the first run | test run without sending mail |
| Agent runs on server | one node of the cluster | the job should only run on a single node |

### 4.2 The “Parameters" Tab

![](../images/Y6XzxZWGYIcZoJnZkFL0vUHXxQ.png)

*These parameters control which internal users are deleted, deactivated, or created.*

| Parameter | Recommendation | Effect |
| --- | --- | --- |
| Delete inactive users that are not found in a directory? | enable | Inactive internal users without an AD entry are deleted. This is the core of the license cleanup. |
| Delete blocked users that are not found in a directory? | enable | Blocked internal users without an AD entry are deleted as well |
| Delete administrators? | leave empty | administrator accounts should not be deleted automatically |
| Only set users found in the defined groups to inactive | optional | Users are set to inactive instead of deleted. A leading `!` excludes the members of the given group. Separate DNs with `;`. |
| Additional filter attribute | optional | additional attribute for the directory search, e.g. `proxyAddresses` |
| Delete inactive/blocked users that are found in the defined groups | leave empty | only applies when the group parameter is set |
| Create users based on group membership | optional | creates new internal users based on AD group membership. Separate multiple groups with `;`. |

Negation in the Field *“Only set users found in the defined groups to inactive"* works via a `!` before a group DN. The members of this group are excluded from the action:

```text
CN=Mitarbeiter,OU=Groups,DC=corp,DC=example,DC=com;!CN=Dienstkonten,OU=Groups,DC=corp,DC=example,DC=com
```

In this example, users in the group *Employees* are set to "inactive" when they are absent from AD, while members of the group *Service Accounts* remain untouched.

## Step 5: Test Run and Validation

Do not run the agent on the production database without first performing a test run. Instead, proceed in the following order:

1.  **Enable Queue Mode** – via the option *“Produced emails are not sent but cached in a queue"*. The agent determines the scheduled actions without sending any emails.
    
2.  **Run Manually** and analyze the agent log: How many users would be affected, and are there any unexpected accounts, such as functional mailboxes, in the list?
    
3.  **Plausibility vs.** `**ldapsearch**` – The number of users not found in the AD should match the results of your manual LDAP query.
    
4.  If the result is correct, disable queue mode, set *Agent enabled* and activate the schedule.
    
5.  After the first successful run, check **Settings → Overview → User Information** again. *Available Users* should then be back in positive territory.
    

## Troubleshooting

| Symptom | Cause | Action |
| --- | --- | --- |
| `Can't contact LDAP server` | port 636 not reachable / wrong host | check with `Test-NetConnection` or `nc -vz`, verify the firewall |
| `Invalid credentials (49)` | wrong bind DN or password | specify the bind DN as a full DN, not as `user@domain` |
| `certificate verify failed` | CA unknown to the trust store | import the root or issuing CA |
| Hostname mismatch in TLS | connecting via IP instead of FQDN | use the certificate's CN/SAN as the host |
| `Referral (10)` | search crosses the domain boundary | use the Global Catalog on port 3269 instead of a DC on 636 |
| Disabled users are not detected | missing `userAccountControl` filter | use the bit matching rule `:1.2.840.113556.1.4.803:=2` |
| Agent deletes too many accounts | filter too broad / wrong base DN | test in queue mode, narrow the base DN |

With the flag `-d 1`, `ldapsearch` provides debug output for the connection setup:

```bash
ldapsearch -d 1 -x -H ldaps://dc01.corp.example.com:636 ...
```

This way, you can see whether the TLS handshake or the bind fails first. The totemomail GUI does not show this distinction in its generic error message.

## Security

-   **Read-only Service-Account.** The Bind user needs read-only access.
    
-   **LDAPS instead of LDAP.** Use port 636 or 3269. LDAP on port 389 transmits the bind password in plain text. With LDAP channel binding and signing, Active Directory is increasingly enforcing secure connections anyway.
    
-   **Password rotation.** `PasswordNeverExpires` is operationally feasible. Document the account and rotate the password according to schedule.
    
-   **Monitoring.** Monitor *Available Users* – ideally via alerts – rather than waiting for the bell to ring.
    
-   **First run in queue mode.** A faulty filter can affect a large number of accounts.
    

## Conclusion

Reaching the license limit is not a technical issue, but rather the result of a lack of an offboarding process. The long-term solution is to regularly synchronize with Active Directory as the primary source. The order is crucial:

1.  Verify the LDAP connection from the command line (`ldapsearch`, `openssl s_client`, `Test-NetConnection`)
    
2.  Configure the connection in Totemomail
    
3.  Validate an agent in queue mode
    
4.  Activate the agent
    

If you follow this procedure, you will not only resolve the immediate licensing issue, but also ensure that it does not happen again.

## Sources

1.  [totemo / Kiteworks – totemomail (Email Protection Gateway)](https://totemo.com/en/resources/downloads) — Product documentation for totemomail (licensing model, LDAP integration, Cleanup Agent); the technology is being continued at Kiteworks as the Email Protection Gateway.
    
2.  [Microsoft Learn – «UserAccountControl property flags»](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties) — Meaning of the flags, among other things `ACCOUNTDISABLE` (0x0002) and `NORMAL_ACCOUNT`.
    
3.  [Microsoft Learn – «Search Filter Syntax»](https://learn.microsoft.com/en-us/windows/win32/adsi/search-filter-syntax) — Bitwise LDAP filter using the matching rule OID `1.2.840.113556.1.4.803` (LDAP\_MATCHING\_RULE\_BIT\_AND).
    
4.  [OpenLDAP – "ldapsearch" (Man Page)](https://www.openldap.org/software/man.cgi?query=ldapsearch) — Call options (`-x`, `-H ldaps://`, `-D`, `-W`, `-b`) for Bind and Search.
    
5.  [Microsoft Learn – «Service overview and network port requirements»](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/service-overview-and-network-port-requirements) — LDAP ports 389/636 and Global Catalog ports 3268/3269.
