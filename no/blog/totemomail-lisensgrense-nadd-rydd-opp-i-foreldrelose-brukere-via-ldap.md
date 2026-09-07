---
title: "Totemomail-lisensgrense nådd: Rydd opp i foreldreløse brukere via LDAP"
navTitle: "Lisensgrense nådd"
description: "Deaktiverte AD-kontoer blir værende i totemomail og opptar fortsatt lisenser. Med verifisert LDAPS-tilgang og Cleanup Agent blir Active Directory den autoritative kilden."
date: "2026-06-26"
kategorie: "Totemomail"
timeToRead: "9 min lesetid"
themen:
  - totemomail
slug: "totemomail-lisensgrense-nadd-rydd-opp-i-foreldrelose-brukere-via-ldap"
translationOf: "totemomail-licensed-user-limit-ldap-cleanup"
translationId: article-cdc60310665049b8
translationReview: automatic
translationSourceHash: e3ae37a51d159128640441aba3bc6993b47b8c5d10968228c05281452940756d
translatedAt: 2026-09-05T08:02:33.709Z
url: https://rafaelpfister.ch/no/blog/totemomail-lisensgrense-nadd-rydd-opp-i-foreldrelose-brukere-via-ldap
translationModel: gpt-5.6-terra
---

# Totemomail-lisensgrense nådd: Rydd opp i foreldreløse brukere via LDAP

Meldingen *«The licensed user limit has been reached»* betyr ikke at e-postflyten stopper umiddelbart. Den indikerer underlisensiering. I miljøer som har vært i drift lenge, skyldes dette som regel ikke plutselig vekst, men tidligere ansatte: AD-kontoen ble deaktivert, den interne brukeren i totemomail ble værende og opptar fortsatt en lisens.

Den varige løsningen er regelmessig LDAP-synkronisering med Active Directory. De følgende trinnene setter opp tilkoblingen og Cleanup Agent og kontrollerer hele kjeden før første kjøring i produksjon. Vertsnavn, DN-er og tjenestekontoer med `example.com` er plassholdere og må tilpasses eget miljø.

## Hvilke brukere opptar en lisens

Totemomail skiller mellom to brukerklasser. Bare interne brukere teller mot lisensgrensen.

| Brukertype | Beskrivelse | Lisensrelevant |
| --- | --- | --- |
| Internal Users | Brukere i egen organisasjon som sender og mottar kryptert | Ja |
| External Users | Eksterne kommunikasjonspartnere (WebMail, PDF, S/MIME, PGP) | Nei |


En intern bruker opprettes så snart vedkommende kommuniserer via gatewayen for første gang. Dette skjer automatisk. Fjerning skjer derimot ikke automatisk: Når en medarbeider forlater organisasjonen, deaktiverer dere vanligvis AD-kontoen. Totemomail-oppføringen blir imidlertid værende. Over tid samler det seg dermed opp foreldreløse kontoer som fortsatt opptar lisenser.

### Statusvisning

Du finner gjeldende status under **Settings → Overview → User Information**.

![](../images/953te2zhdJ61lxda1mj04QrlQA.png)

*Available Users er* `*-17*`*. De 4017 interne brukerne har færre lisensierte plasser tilgjengelig enn dette.*

De viktige linjene:

-   **Internal users** (`4017`): opprettede interne brukere
    
-   **Internal blocked users** (`14`): blokkerte, men fortsatt lisensrelevante
    
-   **Available Users** (`-17`): tilgjengelige lisenser; en negativ verdi betyr underlisensiering
    

Så snart *Available Users* faller under null, vises advarselen ved bjellen:

![](../images/lcL4owxA3iEdg3L9ZFd2bIioE.png)

*«The licensed user limit has been reached.» E-postflyten fortsetter, men meldingen forblir synlig permanent.*

Viktig: Underlisensieringen blokkerer ikke e-postflyten. Dette er en lisensmessig tilstand, ikke en teknisk. Du har derfor tid til å finne en ryddig løsning, men bør ikke ignorere tilstanden permanent.

## Fra umiddelbart tiltak til varig løsning

### Manuell sletting

Du kan søke etter og slette interne brukere enkeltvis under **Internal Users**. Dette løser den akutte situasjonen, men problemet kommer tilbake etter noen måneder. Med flere tusen kontoer er dette ikke praktisk.

### LDAP-tilkobling med Cleanup Agent

Den bærekraftige løsningen er å koble til Active Directory via LDAP. En agent sammenligner interne brukere regelmessig med katalogen og fjerner eller deaktiverer kontoer som ikke lenger finnes i AD. Dermed blir AD den autoritative kilden, og offboarding-prosessen i AD tar samtidig hånd om lisenshygienen.

## Grunnleggende om LDAP

| Begrep | Betydning |
| --- | --- |
| DN (Distinguished Name) | Entydig sti til et objekt, f.eks. `CN=John Doe,OU=Users,DC=corp,DC=example,DC=com` |
| Base DN / Search Base | Søkerot, f.eks. `DC=corp,DC=example,DC=com` |
| Bind DN | Kontoen totemomail bruker for å autentisere seg mot AD |
| Filter | LDAP-søkeuttrykk, f.eks. `(&(objectClass=user)(sAMAccountName=jdoe))` |


### Porter

| Port | Protokoll | Bruk |
| --- | --- | --- |
| 389 | LDAP | ukryptert / STARTTLS |
| 636 | LDAPS | LDAP over TLS |
| 3268 | Global Catalog | søk i hele forestet, ukryptert |
| 3269 | Global Catalog SSL | søk i hele forestet over TLS |


I et miljø med ett domene holder det med port 636 mot en Domain Controller. Kjører du et forest med flere domener, er det bare Global Catalog (port 3269) som gir resultater for hele forestet. En DC på port 636 kjenner bare objektene i sitt eget domene og svarer på søk utenfor sin partisjon med en referral (en detalj som ofte overses i miljøer med flere domener).

### userAccountControl

Om en AD-konto er deaktivert, står i bitfeltet `userAccountControl`. Flagget `ACCOUNTDISABLE` har verdien `2`. Med LDAP-matching-regelen `1.2.840.113556.1.4.803` (`LDAP_MATCHING_RULE_BIT_AND`) evaluerer du enkeltbiter:

```text
# Aktive Benutzer
(&(objectClass=user)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))

# Deaktivierte Benutzer
(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))
```

## Trinn 1: Tjenestekonto i AD

Opprett en dedikert konto med kun lesetilgang for tilkoblingen. Ikke bruk en administratorkonto til dette. Bind-brukeren trenger bare å kunne lese AD.

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
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-Name` | Visningsnavn og CN for den nye kontoen |
| `-SamAccountName` | Påloggingsnavn (påloggingsnavn før Windows 2000) |
| `-UserPrincipalName` | UPN i formatet `benutzer@domäne` |
| `-Path` | Mål-OU som Distinguished Name |
| `-AccountPassword` | Passord som SecureString; `Read-Host -AsSecureString` ber om det skjult i konsollen |
| `-PasswordNeverExpires $true` | Passordet utløper ikke |
| `-Enabled $true` | Oppretter kontoen direkte aktivert (standard er deaktivert) |

</details>

En vanlig domenebruker kan allerede lese AD, så kontoen trenger ingen ekstra rettigheter. For passordet anbefales en lang, tilfeldig verdi som lagres i passordhvelvet ditt.

Hvis sikkerhetspolicyen din legger opp til det, kan du også bruke en gMSA (Group Managed Service Account). Totemomail forventer imidlertid Bind DN og passord, så i praksis brukes oftest en klassisk tjenestekonto med `PasswordNeverExpires`.

## Trinn 2: Kontroller LDAP-tilkoblingen på kommandolinjen

Før du konfigurerer noe i totemomail, bør du verifisere LDAP-tilkoblingen på kommandolinjen. Dette er trinnet de fleste hopper over. Fungerer `ldapsearch`, fungerer også tilkoblingen i totemomail. Mislykkes testen, vet du i det minste hvor det feiler, i stedet for å gjette i totemomail-GUI-et.

### 2.1 Portkontroll

Under Linux, for eksempel fra totemomail-appliancen:

```bash
nc -vz dc01.corp.example.com 636
nmap -p 389,636,3268,3269 dc01.corp.example.com
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `nc -v` | Detaljert utdata: melder om tilkoblingsforsøket lykkes eller mislykkes |
| `nc -z` | Kontroller bare tilkoblingsopprettelsen, uten å sende data |
| `dc01.corp.example.com 636` | Målvert og målport for kontrollen |
| `nmap -p 389,636,3268,3269` | Liste over TCP-porter som skal kontrolleres |
| `dc01.corp.example.com` | Målvert for portskanningen |

</details>

Under Windows med PowerShell:

```powershell
Test-NetConnection -ComputerName dc01.corp.example.com -Port 636
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-ComputerName` | Målvert for tilkoblingskontrollen |
| `-Port` | TCP-porten som skal kontrolleres, her 636 for LDAPS |

</details>

Hvis det ikke opprettes noen forbindelse her, har du et brannmur- eller rutingproblem, ikke et LDAP-problem.

### 2.2 Kontroller TLS-sertifikatet

I praksis feiler LDAPS oftest på sertifikatet. Se derfor på hva DC-en leverer:

```bash
openssl s_client -connect dc01.corp.example.com:636 -showcerts </dev/null
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `s_client` | TLS-testklient fra OpenSSL: oppretter tilkoblingen og viser handshake-detaljer |
| `-connect dc01.corp.example.com:636` | Målvert og port for TLS-handshake |
| `-showcerts` | Viser hele sertifikatkjeden levert av serveren, ikke bare serversertifikatet |
| `</dev/null` | Lukker standard inndata slik at `s_client` avsluttes etter handshake i stedet for å vente på inndata |

</details>

Vær oppmerksom på to ting:

-   `**subject=**` **/** `**issuer=**`: Vertsnavnet i sertifikatet (CN eller SAN) må samsvare med vertsnavnet du kobler deg til via. Kobler du til via IP-adressen, vil kontrollen feile dersom sertifikatet bare inneholder FQDN-en.
    
-   `**Verify return code: 0 (ok)**`: Den utstedende CA-en må være kjent for totemomail. Ved en intern Enterprise CA må du importere dens rot- eller utstedersertifikat i totemomail-truststore.
    

### 2.3 Bind og søk med ldapsearch

`ldapsearch` hører til `ldap-utils` (Debian/Ubuntu) eller `openldap-clients` (RHEL):

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
<summary>Forklaring av alternativer</summary>

| Flagg | Betydning |
| --- | --- |
| `-x` | Enkel autentisering (Bind DN og passord) |
| `-H` | LDAP-URI inkludert skjema (`ldaps://`) og port |
| `-D` | Bind DN |
| `-W` | Be om passord interaktivt |
| `-b` | Search Base |
| deretter | Filter, etterfulgt av attributtene som skal returneres |

</details>

Hvis spørringen returnerer objektet med attributtene, er tilkoblingen på plass. Du kan finne ut hvor mange kontoer i AD som er deaktivert, med bitfilteret:

```bash
ldapsearch -x -H ldaps://dc01.corp.example.com:636 \
  -D "CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com" -W \
  -b "DC=corp,DC=example,DC=com" \
  "(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))" \
  sAMAccountName | grep -c sAMAccountName
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-x`, `-H`, `-D`, `-W`, `-b` | som i spørringen ovenfor: enkel bind via LDAPS med Search Base |
| `sAMAccountName` | eneste etterspurte attributt; begrenser utdata til én attributtlinje per treff |
| `grep -c sAMAccountName` | teller linjene med dette attributtet og dermed de funne kontoene |

</details>

### 2.4 Verktøy under Windows

`**ldp.exe**` er Microsofts grafiske LDAP-verktøy, tilgjengelig på alle DC-er og som del av RSAT. Du kobler til via `Connection → Connect` (vert, port 636, aktiver SSL), autentiserer deg med `Connection → Bind` og navigerer gjennom katalogtreet via `View → Tree` med Base DN.

Uten RSAT kan du bruke ADSI-Searcher i PowerShell:

```powershell
$searcher = [adsisearcher]"(&(objectClass=user)(sAMAccountName=jdoe))"
$searcher.SearchRoot = [adsi]"LDAP://dc01.corp.example.com/DC=corp,DC=example,DC=com"
$searcher.FindOne().Properties
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `[adsisearcher]"(…)"` | oppretter en `DirectorySearcher` med det angitte LDAP-filteret |
| `SearchRoot` | Startpunkt for søket som ADSI-sti: server pluss Base DN |
| `FindOne()` | returnerer første treff; `.Properties` viser attributtene |

</details>

Med RSAT og AD-modulen går det kortere:

```powershell
Get-ADUser -Server dc01.corp.example.com `
  -SearchBase "DC=corp,DC=example,DC=com" `
  -Filter "Enabled -eq '$true'" |
  Measure-Object
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-Server` | Domain Controller som spørringen kjøres mot |
| `-SearchBase` | Søkerot som Distinguished Name |
| `-Filter` | Filter i PowerShell-syntaks; her bare aktiverte kontoer |
| `Measure-Object` | teller de returnerte objektene i stedet for å liste dem opp |

</details>

Klassisk via `dsquery`, tilgjengelig på alle DC-er:

```bash
dsquery user -disabled -limit 0
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `user` | Objekttype for søket: brukerkontoer |
| `-disabled` | bare deaktiverte kontoer |
| `-limit 0` | ingen begrensning av antall treff (standard: 100) |

</details>

Gå først videre i totemomail når en av disse testene kjører uten feil.

## Trinn 3: Konfigurer LDAP-tilkoblingen i totemomail

Opprett LDAP-katalogen i admin-GUI-et under **Directories / LDAP**. Bruk nøyaktig de verdiene du testet tidligere:

| Felt | Eksempelverdi |
| --- | --- |
| Host / URL | `ldaps://dc01.corp.example.com:636` |
| Bind DN | `CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com` |
| Bind Password | Passord for tjenestekontoen |
| Base DN | `DC=corp,DC=example,DC=com` |
| User Filter | `(&(objectClass=user)(objectCategory=person))` |
| Login Attribute | `sAMAccountName` (alternativt `mail` eller `userPrincipalName`) |


Hvis du bruker LDAPS med en intern CA, må du importere dens rot- eller utstedersertifikat i totemomail-truststore. Ellers feiler TLS-handshake med «certificate verify failed», selv om `ldapsearch` med `-x` fungerte tidligere: `ldapsearch` kontrollerer nemlig ikke sertifikatet strengt i denne formen.

Etter lagring kjører du den innebygde testtilkoblingen. Den bekrefter bind.

## Trinn 4: Opprett Cleanup Agent

Under **Maintenance → Agents → Add** oppretter du en agent av typen **«Check presence of internal users in directories»**.

### 4.1 Fanen «Schedule»

![](../images/oSiutQSlKTW0tMY5HUtWCMGuXQ.png)

*Agenten kjører her månedlig den 1. kl. 00:30. Via «Agent runs on server» angir du utførende node i klyngen.*

| Felt | Anbefaling | Begrunnelse |
| --- | --- | --- |
| The agent should run | `monthly`, dag `1`, `00:30` | utenfor arbeidstiden; månedlig er tilstrekkelig for lisenshygiene |
| Agent enabled | aktiver først etter testkjøringen | se trinn 5 |
| Produced emails are not sent but cached in a queue | aktiver for første kjøring | testkjøring uten e-postsending |
| Agent runs on server | én node i klyngen | jobben skal bare kjøre på én node |


### 4.2 Fanen «Parameters»

![](../images/Y6XzxZWGYIcZoJnZkFL0vUHXxQ.png)

*Parametrene styrer hvilke interne brukere som slettes, deaktiveres eller opprettes.*

| Parameter | Anbefaling | Virkning |
| --- | --- | --- |
| Delete inactive users that are not found in a directory? | aktiver | Inaktive interne brukere uten AD-oppføring slettes. Dette er kjernen i lisensoppryddingen. |
| Delete blocked users that are not found in a directory? | aktiver | Blokkerte interne brukere uten AD-oppføring slettes også |
| Delete administrators? | la stå tomt | Administratorkontoer skal ikke slettes automatisk |
| Only set users found in the defined groups to inactive | valgfritt | Brukere settes til inaktive i stedet for å slettes. Et innledende `!` unntar medlemmene av den angitte gruppen. Skill DN-er med `;`. |
| Additional filter attribute | valgfritt | Ekstra attributt for søket i katalogen, f.eks. `proxyAddresses` |
| Delete inactive/blocked users that are found in the defined groups | la stå tomt | gjelder bare når gruppeparameteren er satt |
| Create users based on group membership | valgfritt | oppretter nye interne brukere basert på AD-gruppemedlemskap. Skill flere grupper med `;`. |


Negasjonen i feltet *«Only set users found in the defined groups to inactive»* fungerer med et `!` foran en gruppe-DN. Medlemmene av denne gruppen unntas fra handlingen:

```text
CN=Mitarbeiter,OU=Groups,DC=corp,DC=example,DC=com;!CN=Dienstkonten,OU=Groups,DC=corp,DC=example,DC=com
```

I dette eksemplet settes brukere i gruppen *Mitarbeiter* til inaktive når de ikke finnes i AD, mens medlemmer av gruppen *Dienstkonten* ikke berøres.

## Trinn 5: Testkjøring og validering

Ikke la agenten kjøre mot produksjonsbestanden uten en testkjøring. Gå i stedet frem i denne rekkefølgen:

1.  **Aktiver kømodus**: via alternativet *«Produced emails are not sent but cached in a queue»*. Agenten identifiserer planlagte handlinger uten å sende e-post.
    
2.  **Kjør manuelt** og evaluer agentloggen: Hvor mange brukere ville blitt berørt, og finnes det uventede kontoer som funksjonspostkasser i listen?
    
3.  **Sannsynlighetskontroll mot** `**ldapsearch**`: Antallet brukere som ikke ble funnet i AD, bør samsvare med den manuelle LDAP-spørringen.
    
4.  Hvis resultatet stemmer, deaktiverer du kømodus, setter *Agent enabled* og aktiverer planen.
    
5.  Etter første kjøring i produksjon kontrollerer du **Settings → Overview → User Information** igjen. *Available Users* bør da være positivt igjen.
    

## Feilsøking

| Symptom | Årsak | Tiltak |
| --- | --- | --- |
| `Can't contact LDAP server` | Port 636 ikke tilgjengelig / feil vert | kontroller med `Test-NetConnection` eller `nc -vz`, kontroller brannmuren |
| `Invalid credentials (49)` | Bind DN eller passord er feil | Angi Bind DN som fullstendig DN, ikke som `user@domain` |
| `certificate verify failed` | CA-en er ukjent for truststore | importer rot- eller utstedende CA |
| Vertsnavn-mismatch i TLS | Tilkobling via IP i stedet for FQDN | bruk sertifikatets CN/SAN som vert |
| `Referral (10)` | Søket krysser domenegrensen | bruk Global Catalog på port 3269 i stedet for DC på 636 |
| Deaktiverte brukere gjenkjennes ikke | manglende `userAccountControl`\-filter | bruk bit-matching-regelen `:1.2.840.113556.1.4.803:=2` |
| Agenten sletter for mange kontoer | Filteret er for bredt / feil Base DN | test i kømodus, begrens Base DN |


Med flagget `-d 1` gir `ldapsearch` debug-utdata for tilkoblingsopprettelsen:

```bash
ldapsearch -d 1 -x -H ldaps://dc01.corp.example.com:636 ...
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-d 1` | Debug-nivå 1: logger prosessen for tilkoblingsopprettelsen, inkludert TLS-handshake, til stderr |
| `-x`, `-H` | Enkel bind og LDAP-URI som i spørringene ovenfor |

</details>

Slik ser du om TLS-handshake eller først bind feiler. Totemomail-GUI-et viser ikke dette skillet bak sin generiske feilmelding.

## sikkerhet

-   **Skrivebeskyttet tjenestekonto.** Bind-brukeren trenger utelukkende lesetilgang.
    
-   **LDAPS i stedet for LDAP.** Bruk port 636 eller 3269. LDAP på port 389 overfører bind-passordet i klartekst. Active Directory krever dessuten i økende grad sikrede tilkoblinger med LDAP Channel Binding og Signing.
    
-   **Passordrotasjon.** `PasswordNeverExpires` er praktisk i drift. Dokumenter kontoen og roter passordet etter en plan.
    
-   **Overvåking.** Overvåk *Available Users* (helst med varsling) i stedet for å vente på bjelleadvarselen.
    
-   **Første kjøring i kømodus.** Et feilaktig filter kan ramme et stort antall kontoer.
    

## Den sikre prosessen i fire trinn

Å nå lisensgrensen er ikke en teknisk feil, men resultatet av en manglende offboarding-prosess. Den varige løsningen er regelmessig synkronisering med Active Directory som autoritativ kilde. Rekkefølgen er avgjørende:

1.  Verifiser LDAP-tilkoblingen på kommandolinjen (`ldapsearch`, `openssl s_client`, `Test-NetConnection`)
    
2.  Konfigurer tilkoblingen i totemomail
    
3.  Valider agenten i kømodus
    
4.  Sett agenten i produksjon
    

De som følger denne rekkefølgen, løser det akutte lisensproblemet og hindrer at det kommer tilbake.

## Kilder

1.  [totemo / Kiteworks – totemomail (Email Protection Gateway)](https://totemo.com/en/resources/downloads): Produktdokumentasjon for totemomail (lisensmodell, LDAP-tilkobling, Cleanup Agent); teknologien videreføres hos Kiteworks som Email Protection Gateway.
    
2.  [Microsoft Learn – «UserAccountControl property flags»](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties): Betydningen av flaggene, blant annet `ACCOUNTDISABLE` (0x0002) og `NORMAL_ACCOUNT`.
    
3.  [Microsoft Learn – «Search Filter Syntax»](https://learn.microsoft.com/en-us/windows/win32/adsi/search-filter-syntax): Bitvis LDAP-filter via matching-regel-OID-en `1.2.840.113556.1.4.803` (LDAP\_MATCHING\_RULE\_BIT\_AND).
    
4.  [OpenLDAP – «ldapsearch» (Manpage)](https://www.openldap.org/software/man.cgi?query=ldapsearch): Kallalternativer (`-x`, `-H ldaps://`, `-D`, `-W`, `-b`) for bind og søk.
    
5.  [Microsoft Learn – «Service overview and network port requirements»](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/service-overview-and-network-port-requirements): LDAP-portene 389/636 samt Global Catalog-portene 3268/3269.
