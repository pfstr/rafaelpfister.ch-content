---
title: "Totemomail licence limit reached: clean up orphaned users via LDAP"
navTitle: "Licence limit reached"
description: "Disabled AD accounts remain in totemomail and continue to consume licences. With a tested LDAPS connection and the cleanup agent, Active Directory becomes the authoritative source."
date: "2026-06-26"
kategorie: "Totemomail"
timeToRead: "9 min read"
themen:
  - totemomail
slug: "totemomail-licensed-user-limit-reached-ldap-cleanup"
translationOf: "totemomail-licensed-user-limit-ldap-cleanup"
url: "https://rafaelpfister.ch/en/blog/totemomail-licensed-user-limit-reached-ldap-cleanup"
translationId: article-cdc60310665049b8
translatedAt: 2026-07-28T11:10:30.447Z
translationReview: automatic
translationSourceHash: 273d9af1e81522e2b2a99614880ebfac17f5c4ab3bb3a1fbdbc940554a5931da
---

# Totemomail licence limit reached: clean up orphaned users via LDAP

The message *«The licensed user limit has been reached»* does not mean that mail flow stops immediately. It indicates under-licensing. In long-running environments, the cause is usually not sudden growth but former employees: the AD account was disabled, while the internal user in totemomail remained and continues to consume a licence.

The sustainable solution is regular LDAP synchronisation with Active Directory. The following steps configure the connection and cleanup agent, and check the entire path before the first production run. Host names, DNs and service accounts with `example.com` are placeholders and must match your own environment.

## Which users consume a licence

Totemomail distinguishes between two user classes. Only internal users count towards the licence limit.

| User type | Description | Relevant to licensing |
| --- | --- | --- |
| Internal Users | Users in your own organisation who send and receive encrypted messages | Yes |
| External Users | External communication partners (WebMail, PDF, S/MIME, PGP) | No |


An internal user is created as soon as they communicate through the gateway for the first time. This happens automatically. Removal does not: when an employee leaves the organisation, you would usually disable their AD account. However, the totemomail entry remains. Over the years, orphaned accounts accumulate and continue to consume licences.

### Status display

You can find the current status under **Settings → Overview → User Information**.

![](../images/953te2zhdJ61lxda1mj04QrlQA.png)

*Available Users is set to* `*-17*`*. The 4017 internal users exceed the number of licensed seats available.*

The important lines are:

-   **Internal users** (`4017`): created internal users
    
-   **Internal blocked users** (`14`): blocked but still relevant to licensing
    
-   **Available Users** (`-17`): available licences; a negative value indicates under-licensing
    

As soon as *Available Users* falls below zero, you will see the warning on the bell:

![](../images/lcL4owxA3iEdg3L9ZFd2bIioE.png)

*“The licensed user limit has been reached.” Mail flow continues, but the message remains permanently visible.*

Important: under-licensing does not block mail flow. It is a licensing condition, not a technical one. You therefore have time to implement a proper solution, but should not ignore the condition permanently.

## From immediate action to a lasting solution

### Manual deletion

You can search for and delete internal users individually under **Internal Users**. This resolves the immediate issue, but the problem will return after a few months. With several thousand accounts, this is not a practical approach.

### LDAP integration with cleanup agent

The robust approach is integration with Active Directory via LDAP. An agent regularly compares internal users with the directory and removes or disables accounts that no longer exist in AD. This makes AD the authoritative source, and your AD offboarding process takes care of licence hygiene at the same time.

## LDAP basics

| Term | Meaning |
| --- | --- |
| DN (Distinguished Name) | Unique path to an object, e.g. `CN=John Doe,OU=Users,DC=corp,DC=example,DC=com` |
| Base DN / Search Base | Root of the search, e.g. `DC=corp,DC=example,DC=com` |
| Bind DN | Account used by totemomail to authenticate to AD |
| Filter | LDAP search expression, e.g. `(&(objectClass=user)(sAMAccountName=jdoe))` |


### Ports

| Port | Protocol | Use |
| --- | --- | --- |
| 389 | LDAP | unencrypted / STARTTLS |
| 636 | LDAPS | LDAP over TLS |
| 3268 | Global Catalog | forest-wide search, unencrypted |
| 3269 | Global Catalog SSL | forest-wide search over TLS |


In a single-domain environment, port 636 to a Domain Controller is sufficient. If you operate a forest with multiple domains, only the Global Catalog (port 3269) provides forest-wide results. A DC on port 636 knows only the objects in its own domain and answers searches outside its partition with a referral (a detail often overlooked in multi-domain environments).

### userAccountControl

Whether an AD account is disabled is stored in the `userAccountControl` bit field. The `ACCOUNTDISABLE` flag has the value `2`. Use the LDAP matching rule `1.2.840.113556.1.4.803` (`LDAP_MATCHING_RULE_BIT_AND`) to evaluate individual bits:

```text
# Aktive Benutzer
(&(objectClass=user)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))

# Deaktivierte Benutzer
(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))
```

## Step 1: Service account in AD

For the integration, create a dedicated account with read-only permissions. Do not use an administrator account. The bind user only needs to be able to read AD.

```powershell
New-ADUser -Name "svc-totemomail-ldap" `
  -SamAccountName "svc-totemomail-ldap" `
  -UserPrincipalName "svc-totemomail-ldap@corp.example.com" `
  -Path "OU=Service Accounts,DC=corp,DC=example,DC=com" `
  -AccountPassword (Read-Host -AsSecureString "Password") `
  -PasswordNeverExpires $true `
  -Enabled $true
```

An ordinary domain user can already read AD, so the account requires no additional permissions. Use a long, random password and store it in your password vault.

If your security policy requires it, you can also use a gMSA (Group Managed Service Account). However, totemomail expects a bind DN and password, so in practice a conventional service account with `PasswordNeverExpires` is usually used.

## Step 2: Check the LDAP connection on the command line

Before configuring anything in totemomail, verify the LDAP connection on the command line. This is the step most people skip. If `ldapsearch` works, the integration in totemomail will work too. If the test fails, you at least know where it is failing instead of guessing in the totemomail GUI.

### 2.1 Port check

On Linux, for example from the totemomail appliance:

```bash
nc -vz dc01.corp.example.com 636
nmap -p 389,636,3268,3269 dc01.corp.example.com
```

On Windows with PowerShell:

```powershell
Test-NetConnection -ComputerName dc01.corp.example.com -Port 636
```

If no connection can be established here, you have a firewall or routing issue, not an LDAP issue.

### 2.2 Check the TLS certificate

In practice, LDAPS most often fails because of the certificate. Therefore, inspect what the DC provides:

```bash
openssl s_client -connect dc01.corp.example.com:636 -showcerts </dev/null
```

Pay attention to two things:

-   `**subject=**` **/** `**issuer=**`: The host name in the certificate (CN or SAN) must match the host name used to connect. If you connect via the IP address, validation fails when the certificate contains only the FQDN.
    
-   `**Verify return code: 0 (ok)**`: The issuing CA must be known to totemomail. If you use an internal enterprise CA, import its root or issuing certificate into the totemomail trust store.
    

### 2.3 Bind and search with ldapsearch

`ldapsearch` is included with `ldap-utils` (Debian/Ubuntu) or `openldap-clients` (RHEL):

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
| `-x` | Simple Authentication (bind DN and password) |
| `-H` | LDAP URI including scheme (`ldaps://`) and port |
| `-D` | Bind DN |
| `-W` | Prompt for password interactively |
| `-b` | Search Base |
| afterwards | Filter, followed by the attributes to return |


If the query returns the object with its attributes, the connection is established. To determine how many accounts are disabled in AD, use the bit filter:

```bash
ldapsearch -x -H ldaps://dc01.corp.example.com:636 \
  -D "CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com" -W \
  -b "DC=corp,DC=example,DC=com" \
  "(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))" \
  sAMAccountName | grep -c sAMAccountName
```

### 2.4 Tools on Windows

`**ldp.exe**` is Microsoft's graphical LDAP tool, available on every DC and included with RSAT. Connect via `Connection → Connect` (host, port 636, enable SSL), authenticate using `Connection → Bind`, and navigate the directory tree via `View → Tree` using the Base DN.

Without RSAT, you can use the ADSI searcher in PowerShell:

```powershell
$searcher = [adsisearcher]"(&(objectClass=user)(sAMAccountName=jdoe))"
$searcher.SearchRoot = [adsi]"LDAP://dc01.corp.example.com/DC=corp,DC=example,DC=com"
$searcher.FindOne().Properties
```

With RSAT and the AD module, it is shorter:

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

Only proceed in totemomail once one of these tests completes successfully.

## Step 3: Configure the LDAP connection in totemomail

Create the LDAP directory in the Admin GUI under **Directories / LDAP**. Use exactly the values you tested beforehand:

| Field | Example value |
| --- | --- |
| Host / URL | `ldaps://dc01.corp.example.com:636` |
| Bind DN | `CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com` |
| Bind Password | Password of the service account |
| Base DN | `DC=corp,DC=example,DC=com` |
| User Filter | `(&(objectClass=user)(objectCategory=person))` |
| Login Attribute | `sAMAccountName` (alternatively `mail` or `userPrincipalName`) |


If you use LDAPS with an internal CA, import its root or issuing certificate into the totemomail trust store. Otherwise, the TLS handshake will fail with “certificate verify failed”, even if `ldapsearch` previously worked with `-x`: in this form, `ldapsearch` does not validate the certificate strictly.

After saving, trigger the built-in test connection. It confirms the bind.

## Step 4: Create the cleanup agent

Under **Maintenance → Agents → Add**, create an agent of type **“Check presence of internal users in directories”**.

### 4.1 “Schedule” tab

![](../images/oSiutQSlKTW0tMY5HUtWCMGuXQ.png)

*Here, the agent runs monthly on the 1st at 00:30. Use “Agent runs on server” to define the executing node in the cluster.*

| Field | Recommendation | Reason |
| --- | --- | --- |
| The agent should run | `monthly`, day `1`, `00:30` | outside business hours; monthly is sufficient for licence hygiene |
| Agent enabled | enable only after the test run | see Step 5 |
| Produced emails are not sent but cached in a queue | enable for the first run | test run without sending emails |
| Agent runs on server | one node in the cluster | the job should run on only one node |


### 4.2 “Parameters” tab

![](../images/Y6XzxZWGYIcZoJnZkFL0vUHXxQ.png)

*The parameters control which internal users are deleted, disabled or newly created.*

| Parameter | Recommendation | Effect |
| --- | --- | --- |
| Delete inactive users that are not found in a directory? | enable | Inactive internal users without an AD entry are deleted. This is the core of licence cleanup. |
| Delete blocked users that are not found in a directory? | enable | Blocked internal users without an AD entry are also deleted |
| Delete administrators? | leave blank | Administrator accounts should not be deleted automatically |
| Only set users found in the defined groups to inactive | optional | Users are set to inactive rather than deleted. A preceding `!` excludes members of the specified group. Separate DNs with `;`. |
| Additional filter attribute | optional | Additional attribute for searching the directory, e.g. `proxyAddresses` |
| Delete inactive/blocked users that are found in the defined groups | leave blank | applies only when the group parameter is set |
| Create users based on group membership | optional | Creates new internal users based on AD group membership. Separate multiple groups with `;`. |


Negation in the *“Only set users found in the defined groups to inactive”* field works by placing `!` before a group DN. Members of this group are excluded from the action:

```text
CN=Mitarbeiter,OU=Groups,DC=corp,DC=example,DC=com;!CN=Dienstkonten,OU=Groups,DC=corp,DC=example,DC=com
```

In this example, users in the *Employees* group are set to inactive when absent from AD, while members of the *Service Accounts* group remain untouched.

## Step 5: Test run and validation

Do not run the agent against production data without a test run. Instead, proceed in this order:

1.  **Enable queue mode**: using the *“Produced emails are not sent but cached in a queue”* option. The agent identifies the planned actions without sending emails.
    
2.  **Run manually** and evaluate the agent log: how many users would be affected, and are unexpected accounts such as functional mailboxes in the list?
    
3.  **Check plausibility against** `**ldapsearch**`: the number of users not found in AD should match your manual LDAP query.
    
4.  If the result is correct, disable queue mode, enable *Agent enabled*, and activate the schedule.
    
5.  After the first production run, check **Settings → Overview → User Information** again. *Available Users* should then be positive again.
    

## Troubleshooting

| Symptom | Cause | Action |
| --- | --- | --- |
| `Can't contact LDAP server` | Port 636 unreachable / incorrect host | check with `Test-NetConnection` or `nc -vz`, check the firewall |
| `Invalid credentials (49)` | Bind DN or password incorrect | specify the bind DN as a full DN, not as `user@domain` |
| `certificate verify failed` | CA unknown to the trust store | import the root or issuing CA |
| Hostname mismatch in TLS | connection via IP instead of FQDN | use the certificate CN/SAN as the host |
| `Referral (10)` | search crosses the domain boundary | use the Global Catalog on port 3269 instead of a DC on 636 |
| Disabled users are not recognised | missing `userAccountControl`\-filter | use the bit matching rule `:1.2.840.113556.1.4.803:=2` |
| Agent deletes too many accounts | filter too broad / Base DN incorrect | test in queue mode, restrict the Base DN |


With the `-d 1` flag, `ldapsearch` outputs connection setup debugging information:

```bash
ldapsearch -d 1 -x -H ldaps://dc01.corp.example.com:636 ...
```

This lets you see whether the TLS handshake or only the bind fails. The generic error message in the totemomail GUI does not show this distinction.

## Security

-   **Read-only service account.** The bind user requires read-only permissions exclusively.
    
-   **LDAPS instead of LDAP.** Use port 636 or 3269. LDAP on port 389 transmits the bind password in plain text. Active Directory is increasingly enforcing secured connections through LDAP Channel Binding and Signing anyway.
    
-   **Password rotation.** `PasswordNeverExpires` is operationally practical. Document the account and rotate the password according to a schedule.
    
-   **Monitoring.** Monitor *Available Users* (ideally through alerting) rather than waiting for the bell warning.
    
-   **First run in queue mode.** An incorrect filter can affect a large number of accounts.
    

## The safe process in four steps

Reaching the licence limit is not a technical defect, but the consequence of a missing offboarding process. The sustainable solution is regular synchronisation with Active Directory as the authoritative source. The sequence is crucial:

1.  Verify the LDAP connection on the command line (`ldapsearch`, `openssl s_client`, `Test-NetConnection`)
    
2.  Configure the connection in totemomail
    
3.  Validate the agent in queue mode
    
4.  Put the agent into production
    

Those who follow this sequence resolve the immediate licensing issue and prevent it from returning.

## Sources

1.  [totemo / Kiteworks – totemomail (Email Protection Gateway)](https://totemo.com/en/resources/downloads): Product documentation for totemomail (licensing model, LDAP integration, cleanup agent); the technology is now continued by Kiteworks as Email Protection Gateway.
    
2.  [Microsoft Learn – «UserAccountControl property flags»](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties): Meaning of the flags, including `ACCOUNTDISABLE` (0x0002) and `NORMAL_ACCOUNT`.
    
3.  [Microsoft Learn – «Search Filter Syntax»](https://learn.microsoft.com/en-us/windows/win32/adsi/search-filter-syntax): Bitwise LDAP filter using the matching-rule OID `1.2.840.113556.1.4.803` (LDAP\_MATCHING\_RULE\_BIT\_AND).
    
4.  [OpenLDAP – «ldapsearch» (man page)](https://www.openldap.org/software/man.cgi?query=ldapsearch): Command options (`-x`, `-H ldaps://`, `-D`, `-W`, `-b`) for binding and searching.
    
5.  [Microsoft Learn – «Service overview and network port requirements»](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/service-overview-and-network-port-requirements): LDAP ports 389/636 and Global Catalog ports 3268/3269.
