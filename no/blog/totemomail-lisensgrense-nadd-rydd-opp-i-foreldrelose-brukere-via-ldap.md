---
title: "Totemomail-lisensgrense nådd: rydd opp i foreldreløse brukere via LDAP"
navTitle: "Lisensgrense nådd"
description: "Deaktiverte AD-kontoer blir værende i totemomail og opptar fortsatt lisenser. Med verifisert LDAPS-tilgang og Cleanup Agent blir Active Directory den autoritative kilden."
date: "2026-06-26"
kategorie: "Totemomail"
timeToRead: "9 min lesetid"
themen:
  - totemomail
slug: "totemomail-lisensgrense-nadd-rydd-opp-i-foreldrelose-brukere-via-ldap"
translationOf: "totemomail-licensed-user-limit-ldap-cleanup"
url: "https://rafaelpfister.ch/no/blog/totemomail-lisensgrense-nadd-rydd-opp-i-foreldrelose-brukere-via-ldap"
translationId: article-cdc60310665049b8
translationReview: automatic
translationSourceHash: 273d9af1e81522e2b2a99614880ebfac17f5c4ab3bb3a1fbdbc940554a5931da
translatedAt: 2026-07-29T12:29:38.972Z
---

# Totemomail-lisensgrense nådd: rydd opp i foreldreløse brukere via LDAP

Meldingen *«The licensed user limit has been reached»* betyr ikke at e-postflyten stopper umiddelbart. Den indikerer underlisensiering. I miljøer som har vært i drift lenge, skyldes dette som regel ikke plutselig vekst, men tidligere ansatte: AD-kontoen ble deaktivert, den interne brukeren i totemomail ble stående og opptar fortsatt en lisens.

Den varige løsningen er regelmessig LDAP-synkronisering med Active Directory. Følgende trinn setter opp forbindelsen og Cleanup Agent og kontrollerer hele kjeden før første produksjonskjøring. Vertnavn, DN-er og tjenestekontoer med `example.com` er plassholdere og må tilpasses eget miljø.

## Hvilke brukere opptar en lisens

Totemomail skiller mellom to brukerklasser. Bare interne brukere teller mot lisensgrensen.

| Brukertype | Beskrivelse | Lisensrelevant |
| --- | --- | --- |
| Internal Users | Brukere i egen organisasjon som sender og mottar kryptert | Ja |
| External Users | Eksterne kommunikasjonspartnere (WebMail, PDF, S/MIME, PGP) | Nei |


En intern bruker opprettes så snart vedkommende kommuniserer via gatewayen for første gang. Dette skjer automatisk. Fjerning skjer derimot ikke automatisk: Når en medarbeider forlater organisasjonen, deaktiverer du vanligvis AD-kontoen. Totemomail-oppføringen blir imidlertid stående. Over tid samler det seg dermed foreldreløse kontoer som fortsatt opptar lisenser.

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

Viktig: Underlisensiering blokkerer ikke e-postflyten. Dette er en lisensmessig, ikke teknisk, tilstand. Du har dermed tid til en ryddig løsning, men bør ikke ignorere tilstanden permanent.

## Fra umiddelbart tiltak til varig løsning

### Manuell sletting

Du kan søke etter og slette interne brukere enkeltvis under **Internal Users**. Det løser den akutte situasjonen, men problemet kommer tilbake etter noen måneder. Med flere tusen kontoer blir dette ikke en god løsning.

### LDAP-integrasjon med Cleanup Agent

Den robuste veien er integrasjon med Active Directory via LDAP. En agent sammenligner de interne brukerne regelmessig med katalogen og fjerner eller deaktiverer kontoer som ikke lenger finnes i AD. Dermed blir AD den autoritative kilden, og offboarding-prosessen din i AD sørger samtidig for lisenshygiene.

## LDAP-grunnleggende

| Begrep | Betydning |
| --- | --- |
| DN (Distinguished Name) | Entydig sti til et objekt, f.eks. `CN=John Doe,OU=Users,DC=corp,DC=example,DC=com` |
| Base DN / Search Base | Søkets rot, f.eks. `DC=corp,DC=example,DC=com` |
| Bind DN | Kontoen som totemomail autentiserer seg mot AD med |
| Filter | LDAP-søkeuttrykk, f.eks. `(&(objectClass=user)(sAMAccountName=jdoe))` |


### Porter

| Port | Protokoll | Bruk |
| --- | --- | --- |
| 389 | LDAP | ukryptert / STARTTLS |
| 636 | LDAPS | LDAP over TLS |
| 3268 | Global Catalog | skogomfattende søk, ukryptert |
| 3269 | Global Catalog SSL | skogomfattende søk over TLS |


I et miljø med ett domene holder det med port 636 mot en Domain Controller. Hvis du drifter en forest med flere domener, er det bare Global Catalog (port 3269) som gir skogomfattende resultater. En DC på port 636 kjenner bare objektene i sitt eget domene og besvarer søk utenfor sin partisjon med en referral (en detalj som ofte overses i miljøer med flere domener).

### userAccountControl

Om en AD-konto er deaktivert, står i bitfeltet `userAccountControl`. Flagget `ACCOUNTDISABLE` har verdien `2`. Med LDAP-matchingregelen `1.2.840.113556.1.4.803` (`LDAP_MATCHING_RULE_BIT_AND`) evaluerer du enkeltbiter:

```text
# Aktive Benutzer
(&(objectClass=user)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))

# Deaktivierte Benutzer
(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))
```

## Trinn 1: Tjenestekonto i AD

For integrasjonen oppretter du en dedikert konto med rene leserettigheter. Ikke bruk en administratorkonto til dette. Bind-brukeren trenger bare å kunne lese AD.

```powershell
New-ADUser -Name "svc-totemomail-ldap" `
  -SamAccountName "svc-totemomail-ldap" `
  -UserPrincipalName "svc-totemomail-ldap@corp.example.com" `
  -Path "OU=Service Accounts,DC=corp,DC=example,DC=com" `
  -AccountPassword (Read-Host -AsSecureString "Passord") `
  -PasswordNeverExpires $true `
  -Enabled $true
```

En vanlig domenebruker kan allerede lese AD, så kontoen trenger ingen ekstra rettigheter. For passordet anbefales en lang, tilfeldig verdi som du lagrer i passordhvelvet ditt.

Hvis sikkerhetspolicyen din krever det, kan du også bruke en gMSA (Group Managed Service Account). Totemomail forventer imidlertid Bind DN og passord, og i praksis brukes derfor som regel en klassisk tjenestekonto med `PasswordNeverExpires`.

## Trinn 2: Kontroller LDAP-forbindelsen på kommandolinjen

Før du konfigurerer noe i totemomail, bør du verifisere LDAP-forbindelsen på kommandolinjen. Dette er trinnet de fleste hopper over. Hvis `ldapsearch` fungerer, fungerer også integrasjonen i totemomail. Hvis testen feiler, vet du i det minste hvor problemet ligger, i stedet for å gjette i totemomail-GUI-et.

### 2.1 Portkontroll

På Linux, for eksempel fra totemomail-appliancen:

```bash
nc -vz dc01.corp.example.com 636
nmap -p 389,636,3268,3269 dc01.corp.example.com
```

På Windows med PowerShell:

```powershell
Test-NetConnection -ComputerName dc01.corp.example.com -Port 636
```

Hvis det ikke opprettes noen forbindelse her, har du et firewall- eller rutingproblem, ikke et LDAP-problem.

### 2.2 Kontroller TLS-sertifikatet

I praksis feiler LDAPS oftest på grunn av sertifikatet. Se derfor hva DC-en leverer:

```bash
openssl s_client -connect dc01.corp.example.com:636 -showcerts </dev/null
```

Vær oppmerksom på to ting:

-   `**subject=**` **/** `**issuer=**`: Vertnavnet i sertifikatet (CN eller SAN) må samsvare med vertnavnet du kobler til via. Kobler du til via IP-adressen, feiler kontrollen hvis sertifikatet bare inneholder FQDN-en.
    
-   `**Verify return code: 0 (ok)**`: Den utstedende CA-en må være kjent for totemomail. Med en intern Enterprise-CA må du importere root- eller issuing-sertifikatet i truststore for totemomail.
    

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

| Flagg | Betydning |
| --- | --- |
| `-x` | Enkel autentisering (Bind DN og passord) |
| `-H` | LDAP-URI inkludert skjema (`ldaps://`) og port |
| `-D` | Bind DN |
| `-W` | Be om passord interaktivt |
| `-b` | Search Base |
| deretter | Filter, etterfulgt av attributtene som skal returneres |


Hvis spørringen returnerer objektet med attributtene, er forbindelsen etablert. Du kan finne ut hvor mange kontoer som er deaktivert i AD med bitfilteret:

```bash
ldapsearch -x -H ldaps://dc01.corp.example.com:636 \
  -D "CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com" -W \
  -b "DC=corp,DC=example,DC=com" \
  "(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))" \
  sAMAccountName | grep -c sAMAccountName
```

### 2.4 Verktøy på Windows

`**ldp.exe**` er Microsofts grafiske LDAP-verktøy, som finnes på hver DC og er en del av RSAT. Du kobler til via `Connection → Connect` (vert, port 636, aktiver SSL), autentiserer deg med `Connection → Bind` og navigerer gjennom katalogtreet via `View → Tree` med Base DN.

Uten RSAT kan du komme i mål i PowerShell med ADSI-søkeren:

```powershell
$searcher = [adsisearcher]"(&(objectClass=user)(sAMAccountName=jdoe))"
$searcher.SearchRoot = [adsi]"LDAP://dc01.corp.example.com/DC=corp,DC=example,DC=com"
$searcher.FindOne().Properties
```

Med RSAT og AD-modulen går det kortere:

```powershell
Get-ADUser -Server dc01.corp.example.com `
  -SearchBase "DC=corp,DC=example,DC=com" `
  -Filter "Enabled -eq '$true'" |
  Measure-Object
```

På klassisk vis via `dsquery`, tilgjengelig på hver DC:

```bash
dsquery user -disabled -limit 0
```

Gå først videre i totemomail når en av disse testene kjører uten feil.

## Trinn 3: Konfigurer LDAP-forbindelsen i totemomail

Opprett LDAP-katalogen i Admin-GUI-et under **Directories / LDAP**. Bruk nøyaktig verdiene du testet tidligere:

| Felt | Eksempelverdi |
| --- | --- |
| Host / URL | `ldaps://dc01.corp.example.com:636` |
| Bind DN | `CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com` |
| Bind Password | Passord for tjenestekontoen |
| Base DN | `DC=corp,DC=example,DC=com` |
| User Filter | `(&(objectClass=user)(objectCategory=person))` |
| Login Attribute | `sAMAccountName` (alternativt `mail` eller `userPrincipalName`) |


Hvis du bruker LDAPS mot en intern CA, må du importere root- eller issuing-sertifikatet i truststore for totemomail. Ellers feiler TLS-handshaken med «certificate verify failed», selv om `ldapsearch` med `-x` fungerte på forhånd: `ldapsearch` kontrollerer nemlig ikke sertifikatet strengt i denne formen.

Etter lagring kjører du den innebygde testforbindelsen. Den bekrefter bindingen.

## Trinn 4: Opprett Cleanup Agent

Under **Maintenance → Agents → Add** oppretter du en agent av typen **«Check presence of internal users in directories»**.

### 4.1 Fanen «Schedule»

![](../images/oSiutQSlKTW0tMY5HUtWCMGuXQ.png)

*Her kjører agenten månedlig den 1. klokken 00:30. Med «Agent runs on server» angir du hvilken node i klyngen som skal kjøre den.*

| Felt | Anbefaling | Begrunnelse |
| --- | --- | --- |
| The agent should run | `monthly`, dag `1`, `00:30` | utenfor arbeidstid; månedlig er tilstrekkelig for lisenshygiene |
| Agent enabled | aktiver først etter testkjøringen | se trinn 5 |
| Produced emails are not sent but cached in a queue | aktiver for første kjøring | testkjøring uten e-postutsendelse |
| Agent runs on server | én node i klyngen | jobben skal bare kjøre på én node |


### 4.2 Fanen «Parameters»

![](../images/Y6XzxZWGYIcZoJnZkFL0vUHXxQ.png)

*Parameterne styrer hvilke interne brukere som slettes, deaktiveres eller opprettes på nytt.*

| Parameter | Anbefaling | Virkning |
| --- | --- | --- |
| Delete inactive users that are not found in a directory? | aktiver | Inaktive interne brukere uten AD-oppføring slettes. Dette er kjernen i lisensoppryddingen. |
| Delete blocked users that are not found in a directory? | aktiver | Blokkerte interne brukere uten AD-oppføring slettes også |
| Delete administrators? | la stå tomt | Administratorkontoer skal ikke slettes automatisk |
| Only set users found in the defined groups to inactive | valgfritt | Brukere settes til inaktive i stedet for å slettes. Et innledende `!` unntar medlemmene av den angitte gruppen. Skill DN-er med `;`. |
| Additional filter attribute | valgfritt | Ekstra attributt for søk i katalogen, f.eks. `proxyAddresses` |
| Delete inactive/blocked users that are found in the defined groups | la stå tomt | gjelder bare når gruppeparameteren er angitt |
| Create users based on group membership | valgfritt | oppretter nye interne brukere basert på AD-gruppemedlemskap. Skill flere grupper med `;`. |


Negasjonen i feltet *«Only set users found in the defined groups to inactive»* fungerer med et `!` foran en gruppe-DN. Medlemmene i denne gruppen unntas fra handlingen:

```text
CN=Mitarbeiter,OU=Groups,DC=corp,DC=example,DC=com;!CN=Dienstkonten,OU=Groups,DC=corp,DC=example,DC=com
```

I dette eksempelet setter du brukere i gruppen *Medarbeidere* til inaktive ved fravær i AD, mens medlemmer i gruppen *Tjenestekontoer* forblir urørt.

## Trinn 5: Testkjøring og validering

Ikke la agenten kjøre mot produksjonsdata uten en testkjøring. Gå i stedet frem i denne rekkefølgen:

1.  **Aktiver kømodus**: via alternativet *«Produced emails are not sent but cached in a queue»*. Agenten finner de planlagte handlingene uten å sende e-post.
    
2.  **Kjør manuelt** og evaluer agentloggen: Hvor mange brukere ville bli berørt, og finnes uventede kontoer, som funksjonspostbokser, i listen?
    
3.  **Sannsynlighetssjekk mot** `**ldapsearch**`: Antallet brukere som ikke ble funnet i AD, bør samsvare med den manuelle LDAP-spørringen.
    
4.  Hvis resultatet stemmer, deaktiverer du kømodus, setter *Agent enabled* og aktiverer planen.
    
5.  Etter første produksjonskjøring kontrollerer du **Settings → Overview → User Information** på nytt. *Available Users* bør da igjen ligge i positivt område.
    

## Feilsøking

| Symptom | Årsak | Tiltak |
| --- | --- | --- |
| `Can't contact LDAP server` | Port 636 er ikke tilgjengelig / feil vert | kontroller med `Test-NetConnection` eller `nc -vz`, sjekk firewall |
| `Invalid credentials (49)` | Bind DN eller passord er feil | oppgi Bind DN som fullstendig DN, ikke som `user@domain` |
| `certificate verify failed` | CA-en er ukjent i truststore | importer root- eller issuing-CA |
| Vertnavnmismatch i TLS | Tilkobling via IP i stedet for FQDN | bruk sertifikatets CN/SAN som vert |
| `Referral (10)` | Søket krysser domenegrensen | bruk Global Catalog på port 3269 i stedet for DC på 636 |
| Deaktiverte brukere oppdages ikke | manglende `userAccountControl`\-filter | bruk bit-matchingregelen `:1.2.840.113556.1.4.803:=2` |
| Agenten sletter for mange kontoer | Filteret er for bredt / feil Base DN | test i kømodus, avgrens Base DN |


Med flagget `-d 1` gir `ldapsearch` debug-utdata for tilkoblingsopprettelsen:

```bash
ldapsearch -d 1 -x -H ldaps://dc01.corp.example.com:636 ...
```

Slik ser du om TLS-handshaken eller først bindingen feiler. Totemomail-GUI-et viser ikke dette skillet i sin generiske feilmelding.

## Sikkerhet

-   **Skrivebeskyttet tjenestekonto.** Bind-brukeren trenger utelukkende leserettigheter.
    
-   **LDAPS i stedet for LDAP.** Bruk port 636 eller 3269. LDAP på port 389 overfører bind-passordet i klartekst. Active Directory krever dessuten i økende grad sikrede forbindelser med LDAP Channel Binding og Signing.
    
-   **Passordrotasjon.** `PasswordNeverExpires` er driftsmessig praktisk. Dokumenter kontoen og roter passordet etter plan.
    
-   **Overvåking.** Overvåk *Available Users* (ideelt sett med varsling), i stedet for å vente på bjelleadvarselen.
    
-   **Første kjøring i kømodus.** Et feil filter kan ramme et stort antall kontoer.
    

## Den sikre prosessen i fire trinn

Å nå lisensgrensen er ikke en teknisk feil, men en følge av en manglende offboarding-prosess. Den varige løsningen er regelmessig synkronisering mot Active Directory som autoritativ kilde. Rekkefølgen er avgjørende:

1.  Verifiser LDAP-forbindelsen på kommandolinjen (`ldapsearch`, `openssl s_client`, `Test-NetConnection`)
    
2.  Konfigurer forbindelsen i totemomail
    
3.  Valider agenten i kømodus
    
4.  Sett agenten i produksjon
    

De som følger denne rekkefølgen, løser det akutte lisensproblemet og forhindrer at det kommer tilbake.

## Kilder

1.  [totemo / Kiteworks – totemomail (Email Protection Gateway)](https://totemo.com/en/resources/downloads): Produktdokumentasjon for totemomail (lisensmodell, LDAP-integrasjon, Cleanup Agent); teknologien videreføres hos Kiteworks som Email Protection Gateway.
    
2.  [Microsoft Learn – «UserAccountControl property flags»](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties): Betydningen av flaggene, blant annet `ACCOUNTDISABLE` (0x0002) og `NORMAL_ACCOUNT`.
    
3.  [Microsoft Learn – «Search Filter Syntax»](https://learn.microsoft.com/en-us/windows/win32/adsi/search-filter-syntax): Bitvis LDAP-filter via matchingregel-OID-en `1.2.840.113556.1.4.803` (LDAP\_MATCHING\_RULE\_BIT\_AND).
    
4.  [OpenLDAP – «ldapsearch» (manpage)](https://www.openldap.org/software/man.cgi?query=ldapsearch): Kallalternativer (`-x`, `-H ldaps://`, `-D`, `-W`, `-b`) for bind og søk.
    
5.  [Microsoft Learn – «Service overview and network port requirements»](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/service-overview-and-network-port-requirements): LDAP-portene 389/636 samt Global Catalog-portene 3268/3269.
