---
title: "Totemomail-lisensgrense nådd: rydd opp i foreldreløse brukere via LDAP"
navTitle: "Lisensgrense nådd"
description: "Deaktiverte AD-kontoer blir værende i totemomail og opptar fortsatt lisenser. Med verifisert LDAPS-tilgang og Cleanup Agent blir Active Directory den ledende kilden."
date: "2026-06-26"
kategorie: "Totemomail"
timeToRead: "9 min lesetid"
themen:
  - totemomail
slug: "totemomail-lisensgrense-nadd-rydd-opp-i-foreldrelose-brukere-via-ldap"
translationOf: "totemomail-licensed-user-limit-ldap-cleanup"
translationId: article-cdc60310665049b8
translationReview: automatic
translationSourceHash: 3d7892da089472f95cc44e06e8ecb8654f53c40099d6ff970060102346823159
translatedAt: 2026-11-06T03:28:31.088Z
translationModel: gpt-5.6-terra
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/totemomail-lisensgrense-nadd-rydd-opp-i-foreldrelose-brukere-via-ldap
---

# Totemomail-lisensgrense nådd: rydd opp i foreldreløse brukere via LDAP

Meldingen *«The licensed user limit has been reached»* betyr ikke at e-postflyten stopper umiddelbart. Den indikerer underlisensiering. I miljøer som har vært i drift lenge, skyldes dette vanligvis ikke plutselig vekst, men tidligere ansatte: AD-kontoen ble deaktivert, den interne brukeren i totemomail ble værende og opptar fortsatt en lisens.

Den varige løsningen er regelmessig LDAP-synkronisering med Active Directory. Følgende trinn konfigurerer forbindelsen og Cleanup Agent og kontrollerer hele kjeden før første produksjonskjøring. Vertnavn, DN-er og tjenestekontoer med `example.com` er plassholdere og må tilpasses eget miljø.

## Hvilke brukere opptar en lisens

Totemomail skiller mellom to brukerklasser. Bare interne brukere teller mot lisensgrensen.

| Brukertype | Beskrivelse | Lisensrelevant |
| --- | --- | --- |
| Internal Users | Brukere i egen organisasjon som sender og mottar kryptert | Ja |
| External Users | Eksterne kommunikasjonspartnere (WebMail, PDF, S/MIME, PGP) | Nei |


En intern bruker opprettes så snart vedkommende kommuniserer via gatewayen for første gang. Dette skjer automatisk. Fjerning skjer derimot ikke automatisk: Når en medarbeider forlater organisasjonen, deaktiverer du vanligvis AD-kontoen. Totemomail-oppføringen blir imidlertid stående. Over tid samler det seg dermed opp foreldreløse kontoer som fortsatt opptar lisenser.

### Statusvisning

Du finner gjeldende status under **Settings → Overview → User Information**.

![](../images/953te2zhdJ61lxda1mj04QrlQA.png)

*Available Users står på* `*-17*`*. De 4017 interne brukerne har færre lisensierte plasser tilgjengelig.*

De viktige linjene:

-   **Internal users** (`4017`): opprettede interne brukere
    
-   **Internal blocked users** (`14`): blokkert, men fortsatt lisensrelevant
    
-   **Available Users** (`-17`): tilgjengelige lisenser; en negativ verdi betyr underlisensiering
    

Så snart *Available Users* faller under null, ser du advarselen ved bjellen:

![](../images/lcL4owxA3iEdg3L9ZFd2bIioE.png)

*«The licensed user limit has been reached.» E-postflyten fortsetter, men meldingen forblir synlig permanent.*

Viktig: Underlisensiering blokkerer ikke e-postflyten. Det er en lisensmessig, ikke en teknisk tilstand. Du har altså tid til en ryddig løsning, men bør ikke ignorere tilstanden permanent.

## Fra umiddelbar tiltak til varig løsning

### Manuell sletting

Du kan søke etter og slette interne brukere enkeltvis under **Internal Users**. Det løser den akutte situasjonen, men problemet kommer tilbake etter noen måneder. Ved flere tusen kontoer er dette ikke praktisk.

### LDAP-tilkobling med Cleanup Agent

Den bærekraftige løsningen er å koble til Active Directory via LDAP. En agent sammenligner jevnlig interne brukere med katalogen og fjerner eller deaktiverer kontoer som ikke lenger finnes i AD. Dermed blir AD den ledende kilden, og offboarding-prosessen i AD tar samtidig hånd om lisenshygienen.

## LDAP-grunnleggende

| Begrep | Betydning |
| --- | --- |
| DN (Distinguished Name) | Entydig sti til et objekt, f.eks. `CN=John Doe,OU=Users,DC=corp,DC=example,DC=com` |
| Base DN / Search Base | Søkerot, f.eks. `DC=corp,DC=example,DC=com` |
| Bind DN | Kontoen som totemomail bruker til å autentisere seg mot AD |
| Filter | LDAP-søkeuttrykk, f.eks. `(&(objectClass=user)(sAMAccountName=jdoe))` |


### Porter

| Port | Protokoll | Bruk |
| --- | --- | --- |
| 389 | LDAP | ukryptert / STARTTLS |
| 636 | LDAPS | LDAP over TLS |
| 3268 | Global Catalog | søk på tvers av skogen, ukryptert |
| 3269 | Global Catalog SSL | søk på tvers av skogen over TLS |


I et enkeltdomenemiljø holder det med port 636 mot en domenekontroller. Driver du en skog med flere domener, er det bare Global Catalog (port 3269) som gir resultater på tvers av skogen. En DC på port 636 kjenner kun objektene i sitt eget domene og svarer på søk utenfor sin partisjon med en referral (en detalj som ofte overses i miljøer med flere domener).

### userAccountControl

Om en AD-konto er deaktivert, angis i bitfeltet `userAccountControl`. Flagget `ACCOUNTDISABLE` har verdien `2`. Med LDAP-matching-regelen `1.2.840.113556.1.4.803` (`LDAP_MATCHING_RULE_BIT_AND`) evaluerer du individuelle biter:

```text
# Aktive Benutzer
(&(objectClass=user)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))

# Deaktivierte Benutzer
(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))
```

## Trinn 1: Tjenestekonto i AD

For tilkoblingen oppretter du en dedikert konto med bare lesetilgang. Ikke bruk en administratorkonto til dette. Bind-brukeren trenger kun å kunne lese AD.

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
| `-Enabled $true` | Oppretter kontoen aktivert direkte (standard er deaktivert) |

</details>

En vanlig domenebruker kan allerede lese AD, så kontoen trenger ingen ekstra rettigheter. For passordet anbefales en lang, tilfeldig verdi som lagres i passordhvelvet ditt.

Hvis sikkerhetspolicyen krever det, kan du også bruke en gMSA (Group Managed Service Account). Totemomail forventer imidlertid Bind DN og passord, og i praksis brukes derfor som regel en klassisk tjenestekonto med `PasswordNeverExpires`.

## Trinn 2: Kontroller LDAP-forbindelsen på kommandolinjen

Før du konfigurerer noe i totemomail, bør du verifisere LDAP-forbindelsen på kommandolinjen. Dette er trinnet de fleste hopper over. Fungerer `ldapsearch`, fungerer også tilkoblingen i totemomail. Mislykkes testen, vet du i det minste hvor den feiler, i stedet for å gjette i totemomail-GUI-et.

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
| `nc -v` | Detaljert utdata: melder om vellykket eller mislykket opprettelse av forbindelse |
| `nc -z` | Kontrollerer kun forbindelsen, sender ingen data |
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
| `-ComputerName` | Målvert for forbindelseskontrollen |
| `-Port` | TCP-porten som skal kontrolleres, her 636 for LDAPS |

</details>

Hvis forbindelsen ikke opprettes her, har du et firewall- eller routingproblem, ikke et LDAP-problem.

### 2.2 Kontroller TLS-sertifikatet

I praksis feiler LDAPS oftest på grunn av sertifikatet. Se derfor hva DC-en leverer:

```bash
openssl s_client -connect dc01.corp.example.com:636 -showcerts </dev/null
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `s_client` | TLS-testklient fra OpenSSL: oppretter forbindelsen og viser handshake-detaljer |
| `-connect dc01.corp.example.com:636` | Målvert og port for TLS-handshaken |
| `-showcerts` | Viser hele sertifikatkjeden levert av serveren, ikke bare serversertifikatet |
| `</dev/null` | Lukker standardinndata slik at `s_client` avsluttes etter handshaken, i stedet for å vente på inndata |

</details>

Vær oppmerksom på to ting:

-   `**subject=**` **/** `**issuer=**`: Vertsnavnet i sertifikatet (CN eller SAN) må samsvare med vertsnavnet du kobler til via. Kobler du til via IP-adressen, feiler kontrollen hvis sertifikatet bare inneholder FQDN-en.
    
-   `**Verify return code: 0 (ok)**`: Den utstedende CA-en må være kjent for totemomail. Ved en intern Enterprise CA må du importere root- eller issuing-sertifikatet i totemomails truststore.
    

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
| `-x` | Simple Authentication (Bind DN og passord) |
| `-H` | LDAP-URI inkludert skjema (`ldaps://`) og port |
| `-D` | Bind DN |
| `-W` | Spør om passord interaktivt |
| `-b` | Search Base |
| deretter | Filter, etterfulgt av attributtene som skal returneres |

</details>

Hvis spørringen returnerer objektet med attributtene, er forbindelsen etablert. Du kan finne ut hvor mange kontoer som er deaktivert i AD ved hjelp av bitfilteret:

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
| `-x`, `-H`, `-D`, `-W`, `-b` | som i spørringen over: Simple Bind over LDAPS med Search Base |
| `sAMAccountName` | eneste forespurte attributt; begrenser utdata til én attributtlinje per treff |
| `grep -c sAMAccountName` | teller linjene med denne attributten og dermed de funne kontoene |

</details>

### 2.4 Verktøy under Windows

`**ldp.exe**` er Microsofts grafiske LDAP-verktøy, som finnes på hver DC og er en del av RSAT. Du kobler til via `Connection → Connect` (vert, port 636, aktiver SSL), autentiserer deg med `Connection → Bind` og navigerer i katalogtreet via `View → Tree` med Base DN.

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
| `SearchRoot` | startpunkt for søket som ADSI-sti: server pluss Base DN |
| `FindOne()` | returnerer første treff; `.Properties` viser attributtene |

</details>

Med RSAT og AD-modulen er det kortere:

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
| `-Server` | Domenekontrolleren som spørringen kjøres mot |
| `-SearchBase` | Søkeroten som Distinguished Name |
| `-Filter` | Filter i PowerShell-syntaks; her kun aktiverte kontoer |
| `Measure-Object` | teller de returnerte objektene i stedet for å liste dem opp |

</details>

Klassisk via `dsquery`, tilgjengelig på hver DC:

```bash
dsquery user -disabled -limit 0
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `user` | Objekttype for søket: brukerkontoer |
| `-disabled` | kun deaktiverte kontoer |
| `-limit 0` | ingen begrensning av antall treff (standard: 100) |

</details>

Gå først videre i totemomail når én av disse testene er gjennomført uten feil.

## Trinn 3: Konfigurer LDAP-forbindelsen i totemomail

Opprett LDAP-katalogen i Admin-GUI-et under **Directories / LDAP**. Bruk nøyaktig de verdiene du testet tidligere:

| Felt | Eksempelverdi |
| --- | --- |
| Host / URL | `ldaps://dc01.corp.example.com:636` |
| Bind DN | `CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com` |
| Bind Password | Passord for tjenestekontoen |
| Base DN | `DC=corp,DC=example,DC=com` |
| User Filter | `(&(objectClass=user)(objectCategory=person))` |
| Login Attribute | `sAMAccountName` (alternativt `mail` eller `userPrincipalName`) |


Hvis du bruker LDAPS mot en intern CA, må du importere root- eller issuing-sertifikatet i totemomails truststore. Ellers feiler TLS-handshaken med «certificate verify failed», selv om `ldapsearch` med `-x` fungerte tidligere: `ldapsearch` kontrollerer nemlig ikke sertifikatet strengt på denne måten.

Etter lagring utløser du den innebygde testforbindelsen. Den bekrefter bind.

## Trinn 4: Opprett Cleanup Agent

Under **Maintenance → Agents → Add** oppretter du en agent av typen **«Check presence of internal users in directories»**.

### 4.1 Fanen «Schedule»

![](../images/oSiutQSlKTW0tMY5HUtWCMGuXQ.png)

*Agenten kjører her månedlig den 1. kl. 00:30. Med «Agent runs on server» angir du den utførende noden i klyngen.*

| Felt | Anbefaling | Begrunnelse |
| --- | --- | --- |
| The agent should run | `monthly`, dag `1`, `00:30` | utenfor arbeidstid; månedlig er nok for lisenshygiene |
| Agent enabled | aktiver først etter testkjøringen | se trinn 5 |
| Produced emails are not sent but cached in a queue | aktiver for første kjøring | testkjøring uten utsending av e-post |
| Agent runs on server | én node i klyngen | jobben skal bare kjøre på én node |


### 4.2 Fanen «Parameters»

![](../images/Y6XzxZWGYIcZoJnZkFL0vUHXxQ.png)

*Parameterne styrer hvilke interne brukere som slettes, deaktiveres eller opprettes på nytt.*

| Parameter | Anbefaling | Virkning |
| --- | --- | --- |
| Delete inactive users that are not found in a directory? | aktiver | Inaktive interne brukere uten AD-oppføring slettes. Dette er kjernen i lisensoppryddingen. |
| Delete blocked users that are not found in a directory? | aktiver | Blokkerte interne brukere uten AD-oppføring slettes også |
| Delete administrators? | la stå tomt | Administratorkontoer skal ikke slettes automatisk |
| Only set users found in the defined groups to inactive | valgfritt | Brukere settes til inaktive i stedet for å slettes. En innledende `!` unntar medlemmene av den angitte gruppen. Skill DN-er med `;`. |
| Additional filter attribute | valgfritt | Ekstra attributt for søk i katalogen, f.eks. `proxyAddresses` |
| Delete inactive/blocked users that are found in the defined groups | la stå tomt | gjelder bare når gruppeparameteren er angitt |
| Create users based on group membership | valgfritt | oppretter nye interne brukere basert på AD-gruppemedlemskap. Skill flere grupper med `;`. |


Negasjonen i feltet *«Only set users found in the defined groups to inactive»* fungerer med en `!` foran en gruppe-DN. Medlemmene i denne gruppen unntas fra handlingen:

```text
CN=Mitarbeiter,OU=Groups,DC=corp,DC=example,DC=com;!CN=Dienstkonten,OU=Groups,DC=corp,DC=example,DC=com
```

I dette eksemplet setter du brukere i gruppen *Mitarbeiter* til inaktive dersom de mangler i AD, mens medlemmer i gruppen *Dienstkonten* forblir urørt.

## Trinn 5: Testkjøring og validering

Ikke la agenten kjøre mot produksjonsmiljøet uten en testkjøring. Gå i stedet frem i denne rekkefølgen:

1.  **Aktiver kømodus**: via alternativet *«Produced emails are not sent but cached in a queue»*. Agenten fastslår de planlagte handlingene uten å sende e-post.
    
2.  **Kjør manuelt** og evaluer agentloggen: Hvor mange brukere ville blitt berørt, og finnes uventede kontoer, som funksjonspostbokser, i listen?
    
3.  **Plausibilitetskontroll mot** `**ldapsearch**`: Antallet brukere som ikke ble funnet i AD, bør samsvare med den manuelle LDAP-spørringen.
    
4.  Hvis resultatet stemmer, deaktiverer du kømodus, setter *Agent enabled* og aktiverer planen.
    
5.  Etter første produksjonskjøring kontrollerer du **Settings → Overview → User Information** på nytt. *Available Users* bør da igjen være positiv.
    

## Feilsøking

| Symptom | Årsak | Tiltak |
| --- | --- | --- |
| `Can't contact LDAP server` | Port 636 er ikke tilgjengelig / feil vert | kontroller med `Test-NetConnection` eller `nc -vz`, kontroller firewall |
| `Invalid credentials (49)` | Bind DN eller passord er feil | angi Bind DN som fullstendig DN, ikke som `user@domain` |
| `certificate verify failed` | CA-en er ukjent for truststore | importer root- eller issuing-CA |
| Vertsnavn-mismatch i TLS | Tilkobling via IP i stedet for FQDN | bruk sertifikatets CN/SAN som vert |
| `Referral (10)` | Søket krysser domenegrensen | bruk Global Catalog på port 3269 i stedet for DC på 636 |
| Deaktiverte brukere oppdages ikke | manglende `userAccountControl`\-filter | bruk bit-matching-regelen `:1.2.840.113556.1.4.803:=2` |
| Agenten sletter for mange kontoer | Filteret er for bredt / feil Base DN | test i kømodus, begrens Base DN |


Med flagget `-d 1` gir `ldapsearch` debug-utdata for forbindelsesopprettelsen:

```bash
ldapsearch -d 1 -x -H ldaps://dc01.corp.example.com:636 ...
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-d 1` | Debug-nivå 1: logger prosessen for forbindelsesopprettelsen inkludert TLS-handshake til stderr |
| `-x`, `-H` | Simple Bind og LDAP-URI som i spørringene over |

</details>

Slik ser du om TLS-handshaken eller først bind feiler. Denne forskjellen viser ikke totemomail-GUI-et bak sin generiske feilmelding.

## sikkerhet

-   **Skrivebeskyttet tjenestekonto.** Bind-brukeren trenger kun lesetilgang.
    
-   **LDAPS i stedet for LDAP.** Bruk port 636 eller 3269. LDAP på port 389 overfører bind-passordet i klartekst. Active Directory håndhever dessuten i økende grad sikrede forbindelser med LDAP Channel Binding og Signing.
    
-   **Passordrotasjon.** `PasswordNeverExpires` er driftsmessig praktisk. Dokumenter kontoen og roter passordet etter plan.
    
-   **Overvåking.** Overvåk *Available Users* (helst med varsling), i stedet for å vente på bjelleadvarselen.
    
-   **Første kjøring i kømodus.** Et feil filter kan ramme et stort antall kontoer.
    

## Den sikre prosessen i fire trinn

Å nå lisensgrensen er ikke en teknisk feil, men en konsekvens av en manglende offboarding-prosess. Den varige løsningen er regelmessig sammenligning med Active Directory som ledende kilde. Rekkefølgen er avgjørende:

1.  Verifiser LDAP-forbindelsen på kommandolinjen (`ldapsearch`, `openssl s_client`, `Test-NetConnection`)
    
2.  Konfigurer forbindelsen i totemomail
    
3.  Valider agenten i kømodus
    
4.  Sett agenten i produksjon
    

De som følger denne rekkefølgen, løser det akutte lisensproblemet og hindrer at det oppstår igjen.

## Endringslogg

- **25. oktober 2026:** Kilde erstattet: Den tidligere dokumentasjonssiden på totemo.com videresender til kiteworks.com; det henvises nå til Kiteworks-siden om videreføringen av totemo som Email Protection Gateway.

## Kilder

1.  [Kiteworks – «Unlocking the Power of Kiteworks Email Protection Gateway (EPG) with totemo Integration»](https://www.kiteworks.com/video-why-kiteworks-and-totemo-are-better-together/): Kiteworks' overtakelse av totemo og videreføring av teknologien som Email Protection Gateway (EPG); den tidligere nedlastings- og dokumentasjonssiden på totemo.com videresender nå til startsiden på kiteworks.com.
    
2.  [Microsoft Learn – «UserAccountControl property flags»](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties): Betydningen av flaggene, blant annet `ACCOUNTDISABLE` (0x0002) og `NORMAL_ACCOUNT`.
    
3.  [Microsoft Learn – «Search Filter Syntax»](https://learn.microsoft.com/en-us/windows/win32/adsi/search-filter-syntax): Bitvis LDAP-filter via matching-regel-OID-en `1.2.840.113556.1.4.803` (LDAP\_MATCHING\_RULE\_BIT\_AND).
    
4.  [OpenLDAP – «ldapsearch» (man-side)](https://www.openldap.org/software/man.cgi?query=ldapsearch): Kallalternativer (`-x`, `-H ldaps://`, `-D`, `-W`, `-b`) for bind og søk.
    
5.  [Microsoft Learn – «Service overview and network port requirements»](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/service-overview-and-network-port-requirements): LDAP-portene 389/636 samt Global Catalog-portene 3268/3269.
