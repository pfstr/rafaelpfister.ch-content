---
title: "Totemomail-licensgränsen nådd: rensa upp övergivna användare via LDAP"
navTitle: "Licensgränsen nådd"
description: "Inaktiverade AD-konton finns kvar i totemomail och fortsätter att uppta licenser. Med en verifierad LDAPS-anslutning och Cleanup-Agent blir Active Directory den styrande källan."
date: "2026-06-26"
kategorie: "Totemomail"
timeToRead: "9 min lästid"
themen:
  - totemomail
slug: "totemomail-licensgrans-nadd-rensa-upp-overgivna-anvandare-via-ldap"
translationOf: "totemomail-licensed-user-limit-ldap-cleanup"
translationId: article-cdc60310665049b8
translationReview: automatic
translationSourceHash: e3ae37a51d159128640441aba3bc6993b47b8c5d10968228c05281452940756d
translatedAt: 2026-09-05T08:01:44.593Z
url: https://rafaelpfister.ch/sv/blog/totemomail-licensgrans-nadd-rensa-upp-overgivna-anvandare-via-ldap
translationModel: gpt-5.6-terra
---

# Totemomail-licensgränsen nådd: rensa upp övergivna användare via LDAP

Meddelandet *«The licensed user limit has been reached»* innebär inte att e-postflödet omedelbart stoppas. Det visar på underlicensiering. I miljöer som har varit i drift länge beror orsaken oftast inte på en plötslig tillväxt, utan på tidigare medarbetare: AD-kontot har inaktiverats, den interna användaren i totemomail finns kvar och upptar fortfarande en licens.

Den långsiktiga lösningen är en regelbunden LDAP-synkronisering med Active Directory. Följande steg konfigurerar anslutningen och Cleanup-Agent samt kontrollerar hela kedjan före den första produktiva körningen. Värdnamn, DN och tjänstkonton med `example.com` är platshållare och måste anpassas till den egna miljön.

## Vilka användare som upptar en licens

Totemomail skiljer mellan två användarklasser. Endast interna användare räknas mot licensgränsen.

| Användartyp | Beskrivning | Licensrelevant |
| --- | --- | --- |
| Internal Users | Användare i den egna organisationen som skickar och tar emot krypterat | Ja |
| External Users | Externa kommunikationspartner (WebMail, PDF, S/MIME, PGP) | Nej |


En intern användare skapas så snart personen kommunicerar via gatewayen för första gången. Det sker automatiskt. Borttagning sker däremot inte automatiskt: När en medarbetare lämnar organisationen inaktiverar du normalt AD-kontot. Posten i totemomail finns dock kvar. Med åren samlas därmed övergivna konton som fortsätter att uppta licenser.

### Statusvisning

Du hittar aktuell status under **Settings → Overview → User Information**.

![](../images/953te2zhdJ61lxda1mj04QrlQA.png)

*Available Users står på* `*-17*`*. De 4017 interna användarna har färre licensierade platser till sitt förfogande.*

De viktiga raderna:

-   **Internal users** (`4017`): skapade interna användare
    
-   **Internal blocked users** (`14`): spärrade, men fortfarande licensrelevanta
    
-   **Available Users** (`-17`): tillgängliga licenser; ett negativt värde innebär underlicensiering
    

Så snart *Available Users* sjunker under noll visas varningen vid klockan:

![](../images/lcL4owxA3iEdg3L9ZFd2bIioE.png)

*”The licensed user limit has been reached.” E-postflödet fortsätter, men meddelandet förblir synligt permanent.*

Viktigt: Underlicensieringen blockerar inte e-postflödet. Det är ett licensmässigt, inte tekniskt tillstånd. Du har alltså tid att hitta en bra lösning, men bör inte ignorera tillståndet permanent.

## Från omedelbar åtgärd till långsiktig lösning

### Manuell radering

Du kan söka efter och radera interna användare en och en under **Internal Users**. Det löser den akuta situationen, men problemet återkommer efter några månader. Med flera tusen konton är det inte praktiskt genomförbart.

### LDAP-anslutning med Cleanup-Agent

Den hållbara vägen är att ansluta till Active Directory via LDAP. En agent jämför regelbundet interna användare med katalogen och tar bort eller inaktiverar konton som inte längre finns i AD. Därmed blir AD den styrande källan, och offboarding-processen i AD tar samtidigt hand om licenshygienen.

## LDAP-grunder

| Begrepp | Betydelse |
| --- | --- |
| DN (Distinguished Name) | Unik sökväg till ett objekt, t.ex. `CN=John Doe,OU=Users,DC=corp,DC=example,DC=com` |
| Base DN / Search Base | Sökningens rot, t.ex. `DC=corp,DC=example,DC=com` |
| Bind DN | Kontot som totemomail använder för att autentisera sig mot AD |
| Filter | LDAP-sökuttryck, t.ex. `(&(objectClass=user)(sAMAccountName=jdoe))` |


### Portar

| Port | Protokoll | Användning |
| --- | --- | --- |
| 389 | LDAP | okrypterat / STARTTLS |
| 636 | LDAPS | LDAP över TLS |
| 3268 | Global Catalog | sökning i hela skogen, okrypterat |
| 3269 | Global Catalog SSL | sökning i hela skogen över TLS |


I en Single-Domain-miljö räcker port 636 mot en Domain Controller. Om du driver en skog med flera domäner är det endast Global Catalog (port 3269) som ger resultat för hela skogen. En DC på port 636 känner endast till objekten i sin egen domän och svarar på sökningar utanför sin partition med en referral (en detalj som ofta förbises i miljöer med flera domäner).

### userAccountControl

Om ett AD-konto är inaktiverat anges i bitfältet `userAccountControl`. Flaggan `ACCOUNTDISABLE` har värdet `2`. Med LDAP-matchingregeln `1.2.840.113556.1.4.803` (`LDAP_MATCHING_RULE_BIT_AND`) kan du utvärdera enskilda bitar:

```text
# Aktive Benutzer
(&(objectClass=user)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))

# Deaktivierte Benutzer
(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))
```

## Steg 1: Servicekonto i AD

För anslutningen skapar du ett dedikerat konto med endast läsbehörighet. Använd inte ett administratörskonto för detta. Bind-användaren behöver endast kunna läsa AD.

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
<summary>Förklarade alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `-Name` | Visningsnamn och CN för det nya kontot |
| `-SamAccountName` | Inloggningsnamn (pre-Windows 2000-logonnamn) |
| `-UserPrincipalName` | UPN i formatet `benutzer@domäne` |
| `-Path` | Mål-OU som Distinguished Name |
| `-AccountPassword` | Lösenord som SecureString; `Read-Host -AsSecureString` frågar efter det dolt i konsolen |
| `-PasswordNeverExpires $true` | Lösenordet upphör inte att gälla |
| `-Enabled $true` | Skapa kontot direkt som aktiverat (standard är inaktiverat) |

</details>

En vanlig domänanvändare kan redan läsa AD, så kontot behöver inga ytterligare behörigheter. För lösenordet rekommenderas ett långt, slumpmässigt värde som lagras i lösenordsvalvet.

Om säkerhetspolicyn föreskriver det kan du även använda ett gMSA (Group Managed Service Account). Totemomail förväntar sig dock Bind DN och lösenord, vilket gör att ett klassiskt servicekonto med `PasswordNeverExpires` oftast används i praktiken.

## Steg 2: kontrollera LDAP-anslutningen på kommandoraden

Innan du konfigurerar något i totemomail bör du verifiera LDAP-anslutningen på kommandoraden. Det är steget som de flesta hoppar över. Fungerar `ldapsearch`, fungerar även anslutningen i totemomail. Om testet misslyckas vet du åtminstone var det brister, i stället för att gissa i totemomail-GUI:t.

### 2.1 Portkontroll

I Linux, exempelvis från totemomail-appliance:

```bash
nc -vz dc01.corp.example.com 636
nmap -p 389,636,3268,3269 dc01.corp.example.com
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `nc -v` | Utförlig utdata: rapporterar om anslutningen lyckas eller misslyckas |
| `nc -z` | Kontrollera endast anslutningen, skicka inga data |
| `dc01.corp.example.com 636` | Målvärd och målport för kontrollen |
| `nmap -p 389,636,3268,3269` | Lista över TCP-portar som ska kontrolleras |
| `dc01.corp.example.com` | Målvärd för portskanningen |

</details>

I Windows med PowerShell:

```powershell
Test-NetConnection -ComputerName dc01.corp.example.com -Port 636
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `-ComputerName` | Målvärd för anslutningskontrollen |
| `-Port` | TCP-port som ska kontrolleras, här 636 för LDAPS |

</details>

Om ingen anslutning kan upprättas här har du ett brandväggs- eller routningsproblem, inte ett LDAP-problem.

### 2.2 Kontrollera TLS-certifikatet

I praktiken misslyckas LDAPS oftast på grund av certifikatet. Titta därför på vad DC:n levererar:

```bash
openssl s_client -connect dc01.corp.example.com:636 -showcerts </dev/null
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `s_client` | TLS-testklient från OpenSSL: upprättar anslutningen och visar handskakningsdetaljer |
| `-connect dc01.corp.example.com:636` | Målvärd och port för TLS-handskakningen |
| `-showcerts` | Visar hela certifikatkedjan som servern levererar, inte bara servercertifikatet |
| `</dev/null` | Stänger standardindata så att `s_client` avslutas efter handskakningen i stället för att vänta på indata |

</details>

Var uppmärksam på två saker:

-   `**subject=**` **/** `**issuer=**`: Värdnamnet i certifikatet (CN respektive SAN) måste matcha det värdnamn som du ansluter till. Om du ansluter via IP-adressen misslyckas kontrollen om certifikatet endast innehåller FQDN.
    
-   `**Verify return code: 0 (ok)**`: Den utfärdande CA:n måste vara känd för totemomail. Vid en intern Enterprise-CA måste du importera dess root- eller issuing-certifikat till totemomails truststore.
    

### 2.3 Bind och sökning med ldapsearch

`ldapsearch` ingår i `ldap-utils` (Debian/Ubuntu) respektive `openldap-clients` (RHEL):

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
<summary>Förklarade alternativ</summary>

| Flagga | Betydelse |
| --- | --- |
| `-x` | Enkel autentisering (Bind DN och lösenord) |
| `-H` | LDAP-URI inklusive schema (`ldaps://`) och port |
| `-D` | Bind DN |
| `-W` | Fråga efter lösenordet interaktivt |
| `-b` | Search Base |
| därefter | Filter, följt av de attribut som ska returneras |

</details>

Om frågan returnerar objektet med dess attribut fungerar anslutningen. Du kan fastställa hur många konton i AD som är inaktiverade med bitfiltret:

```bash
ldapsearch -x -H ldaps://dc01.corp.example.com:636 \
  -D "CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com" -W \
  -b "DC=corp,DC=example,DC=com" \
  "(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))" \
  sAMAccountName | grep -c sAMAccountName
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `-x`, `-H`, `-D`, `-W`, `-b` | som i frågan ovan: enkel bindning via LDAPS med Search Base |
| `sAMAccountName` | enda begärda attributet; begränsar utdata till en attributrad per träff |
| `grep -c sAMAccountName` | räknar raderna med detta attribut och därmed de hittade kontona |

</details>

### 2.4 Verktyg i Windows

`**ldp.exe**` är Microsofts grafiska LDAP-verktyg, som finns på varje DC och ingår i RSAT. Du ansluter via `Connection → Connect` (värd, port 636, aktivera SSL), autentiserar dig med `Connection → Bind` och navigerar genom katalogträdet med Base DN via `View → Tree`.

Utan RSAT kan du använda ADSI-Searcher i PowerShell:

```powershell
$searcher = [adsisearcher]"(&(objectClass=user)(sAMAccountName=jdoe))"
$searcher.SearchRoot = [adsi]"LDAP://dc01.corp.example.com/DC=corp,DC=example,DC=com"
$searcher.FindOne().Properties
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `[adsisearcher]"(…)"` | skapar en `DirectorySearcher` med angivet LDAP-filter |
| `SearchRoot` | Sökningens startpunkt som ADSI-sökväg: server plus Base DN |
| `FindOne()` | returnerar första träffen; `.Properties` visar dess attribut |

</details>

Med RSAT och AD-modulen går det snabbare:

```powershell
Get-ADUser -Server dc01.corp.example.com `
  -SearchBase "DC=corp,DC=example,DC=com" `
  -Filter "Enabled -eq '$true'" |
  Measure-Object
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `-Server` | Domain Controller som frågan körs mot |
| `-SearchBase` | Sökningens rot som Distinguished Name |
| `-Filter` | Filter i PowerShell-syntax; här endast aktiverade konton |
| `Measure-Object` | räknar returnerade objekt i stället för att lista dem |

</details>

Klassiskt via `dsquery`, tillgängligt på varje DC:

```bash
dsquery user -disabled -limit 0
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `user` | Objekttyp för sökningen: användarkonton |
| `-disabled` | endast inaktiverade konton |
| `-limit 0` | ingen begränsning av antalet träffar (standard: 100) |

</details>

Fortsätt i totemomail först när ett av dessa test har genomförts utan problem.

## Steg 3: konfigurera LDAP-anslutningen i totemomail

Skapa LDAP-katalogen i administrations-GUI:t under **Directories / LDAP**. Använd exakt de värden som du tidigare testade:

| Fält | Exempelvärde |
| --- | --- |
| Host / URL | `ldaps://dc01.corp.example.com:636` |
| Bind DN | `CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com` |
| Bind Password | Lösenord för servicekontot |
| Base DN | `DC=corp,DC=example,DC=com` |
| User Filter | `(&(objectClass=user)(objectCategory=person))` |
| Login Attribute | `sAMAccountName` (alternativt `mail` eller `userPrincipalName`) |


Om du använder LDAPS mot en intern CA måste du importera dess root- eller issuing-certifikat till totemomails truststore. Annars misslyckas TLS-handskakningen med ”certificate verify failed”, även om `ldapsearch` med `-x` fungerade tidigare: `ldapsearch` validerar nämligen inte certifikatet strikt i denna form.

Efter att du har sparat startar du den inbyggda testanslutningen. Den bekräftar bindningen.

## Steg 4: skapa Cleanup-Agent

Under **Maintenance → Agents → Add** skapar du en agent av typen **„Check presence of internal users in directories"**.

### 4.1 Fliken „Schedule"

![](../images/oSiutQSlKTW0tMY5HUtWCMGuXQ.png)

*Agenten körs här månadsvis den 1:a klockan 00:30. Med „Agent runs on server” anger du den körande noden i klustret.*

| Fält | Rekommendation | Motivering |
| --- | --- | --- |
| The agent should run | `monthly`, dag `1`, `00:30` | utanför arbetstid; månadsvis räcker för licenshygienen |
| Agent enabled | aktivera först efter testkörningen | se steg 5 |
| Produced emails are not sent but cached in a queue | aktivera för första körningen | testkörning utan e-postutskick |
| Agent runs on server | en nod i klustret | jobbet ska endast köras på en nod |


### 4.2 Fliken „Parameters"

![](../images/Y6XzxZWGYIcZoJnZkFL0vUHXxQ.png)

*Parametrarna styr vilka interna användare som raderas, inaktiveras eller skapas.*

| Parameter | Rekommendation | Funktion |
| --- | --- | --- |
| Delete inactive users that are not found in a directory? | aktivera | Inaktiva interna användare utan AD-post raderas. Detta är kärnan i licensrensningen. |
| Delete blocked users that are not found in a directory? | aktivera | Spärrade interna användare utan AD-post raderas också |
| Delete administrators? | lämna tomt | Administratörskonton ska inte raderas automatiskt |
| Only set users found in the defined groups to inactive | valfritt | Användare sätts till inaktiva i stället för att raderas. Ett inledande `!` undantar medlemmarna i den angivna gruppen. Separera DN med `;`. |
| Additional filter attribute | valfritt | ytterligare attribut för sökningen i katalogen, t.ex. `proxyAddresses` |
| Delete inactive/blocked users that are found in the defined groups | lämna tomt | används endast om gruppparametern är angiven |
| Create users based on group membership | valfritt | skapar nya interna användare baserat på AD-gruppmedlemskap. Separera flera grupper med `;`. |


Negeringen i fältet *„Only set users found in the defined groups to inactive”* fungerar med ett `!` före ett grupp-DN. Medlemmarna i denna grupp undantas från åtgärden:

```text
CN=Mitarbeiter,OU=Groups,DC=corp,DC=example,DC=com;!CN=Dienstkonten,OU=Groups,DC=corp,DC=example,DC=com
```

I detta exempel sätts användare i gruppen *Mitarbeiter* till inaktiva när de saknas i AD, medan medlemmar i gruppen *Dienstkonten* lämnas orörda.

## Steg 5: testkörning och validering

Låt inte agenten köras mot produktionsbeståndet utan en testkörning. Gör i stället följande i denna ordning:

1.  **Aktivera köläge**: med alternativet *„Produced emails are not sent but cached in a queue”*. Agenten identifierar de planerade åtgärderna utan att skicka e-post.
    
2.  **Kör manuellt** och analysera agentloggen: Hur många användare skulle påverkas, och finns oväntade konton som funktionsbrevlådor i listan?
    
3.  **Kontrollera rimligheten mot** `**ldapsearch**`: Antalet användare som inte hittas i AD bör stämma överens med den manuella LDAP-frågan.
    
4.  Om resultatet stämmer, inaktivera köläget, aktivera *Agent enabled* och slå på schemat.
    
5.  Efter den första produktiva körningen kontrollerar du **Settings → Overview → User Information** igen. *Available Users* bör då vara positivt igen.
    

## Felsökning

| Symptom | Orsak | Åtgärd |
| --- | --- | --- |
| `Can't contact LDAP server` | Port 636 inte tillgänglig / fel värd | kontrollera med `Test-NetConnection` respektive `nc -vz`, kontrollera brandväggen |
| `Invalid credentials (49)` | Bind DN eller lösenord felaktigt | Ange Bind DN som fullständigt DN, inte som `user@domain` |
| `certificate verify failed` | CA okänd i truststore | importera root- eller issuing-CA |
| Värdnamnsmatchning misslyckas i TLS | anslutning via IP i stället för FQDN | använd certifikatets CN/SAN som värd |
| `Referral (10)` | Sökningen överskrider domängränsen | använd Global Catalog på port 3269 i stället för DC på 636 |
| Inaktiverade användare identifieras inte | `userAccountControl`\-filter saknas | använd bit-matchingregeln `:1.2.840.113556.1.4.803:=2` |
| Agenten raderar för många konton | filtret för brett / fel Base DN | testa i köläge, begränsa Base DN |


Med flaggan `-d 1` ger `ldapsearch` debug-utdata för anslutningsupprättandet:

```bash
ldapsearch -d 1 -x -H ldaps://dc01.corp.example.com:636 ...
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `-d 1` | Debugnivå 1: loggar anslutningsupprättandet inklusive TLS-handskakning på stderr |
| `-x`, `-H` | Enkel bindning och LDAP-URI som i frågorna ovan |

</details>

Därmed ser du om TLS-handskakningen eller först bindningen misslyckas. Denna skillnad visar totemomail-GUI:t inte bakom sitt generiska felmeddelande.

## Säkerhet

-   **Read-only servicekonto.** Bind-användaren behöver uteslutande läsbehörighet.
    
-   **LDAPS i stället för LDAP.** Använd port 636 respektive 3269. LDAP på port 389 överför Bind-lösenordet i klartext. Active Directory kräver dessutom i allt högre grad säkrade anslutningar med LDAP Channel Binding och Signing.
    
-   **Lösenordsrotation.** `PasswordNeverExpires` är praktiskt genomförbart i drift. Dokumentera kontot och rotera lösenordet enligt plan.
    
-   **Övervakning.** Övervaka *Available Users* (helst med aviseringar) i stället för att vänta på varningen vid klockan.
    
-   **Första körningen i köläge.** Ett felaktigt filter kan träffa ett stort antal konton.
    

## Det säkra förfarandet i fyra steg

Att licensgränsen nås är inte ett tekniskt fel, utan följden av en saknad offboarding-process. Den långsiktiga lösningen är regelbunden jämförelse med Active Directory som styrande källa. Ordningen är avgörande:

1.  Verifiera LDAP-anslutningen på kommandoraden (`ldapsearch`, `openssl s_client`, `Test-NetConnection`)
    
2.  Konfigurera anslutningen i totemomail
    
3.  Validera agenten i köläge
    
4.  Sätt agenten i produktion
    

Den som följer denna ordning löser det akuta licensproblemet och förhindrar att det återkommer.

## Källor

1.  [totemo / Kiteworks – totemomail (Email Protection Gateway)](https://totemo.com/en/resources/downloads): Produktdokumentation om totemomail (licensmodell, LDAP-anslutning, Cleanup-Agent); tekniken vidareutvecklas hos Kiteworks som Email Protection Gateway.
    
2.  [Microsoft Learn – «UserAccountControl property flags»](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties): Betydelsen av flaggorna, bland annat `ACCOUNTDISABLE` (0x0002) och `NORMAL_ACCOUNT`.
    
3.  [Microsoft Learn – «Search Filter Syntax»](https://learn.microsoft.com/en-us/windows/win32/adsi/search-filter-syntax): Bitvis LDAP-filter via matchingregelns OID `1.2.840.113556.1.4.803` (LDAP\_MATCHING\_RULE\_BIT\_AND).
    
4.  [OpenLDAP – «ldapsearch» (manpage)](https://www.openldap.org/software/man.cgi?query=ldapsearch): Anropsalternativ (`-x`, `-H ldaps://`, `-D`, `-W`, `-b`) för bindning och sökning.
    
5.  [Microsoft Learn – «Service overview and network port requirements»](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/service-overview-and-network-port-requirements): LDAP-portarna 389/636 samt Global Catalog-portarna 3268/3269.
