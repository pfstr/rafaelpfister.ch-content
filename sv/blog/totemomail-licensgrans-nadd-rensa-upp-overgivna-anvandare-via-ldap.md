---
title: "Totemomail-licensgräns nådd: rensa upp övergivna användare via LDAP"
navTitle: "Licensgräns nådd"
description: "Inaktiverade AD-konton finns kvar i totemomail och upptar fortsatt licenser. Med en kontrollerad LDAPS-anslutning och Cleanup Agent blir Active Directory den ledande källan."
date: "2026-06-26"
kategorie: "Totemomail"
timeToRead: "9 min läsning"
themen:
  - totemomail
slug: "totemomail-licensgrans-nadd-rensa-upp-overgivna-anvandare-via-ldap"
translationOf: "totemomail-licensed-user-limit-ldap-cleanup"
url: "https://rafaelpfister.ch/sv/blog/totemomail-licensgrans-nadd-rensa-upp-overgivna-anvandare-via-ldap"
translationId: article-cdc60310665049b8
translationReview: automatic
translationSourceHash: 273d9af1e81522e2b2a99614880ebfac17f5c4ab3bb3a1fbdbc940554a5931da
translatedAt: 2026-07-29T12:29:38.964Z
---

# Totemomail-licensgräns nådd: rensa upp övergivna användare via LDAP

Meddelandet *«The licensed user limit has been reached»* betyder inte att e-postflödet stoppas omedelbart. Det visar att ni har för få licenser. I miljöer som har varit i drift länge beror orsaken oftast inte på plötslig tillväxt, utan på tidigare medarbetare: AD-kontot har inaktiverats, den interna användaren i totemomail finns kvar och upptar fortfarande en licens.

Den hållbara lösningen är en regelbunden LDAP-synkronisering med Active Directory. Följande steg konfigurerar anslutningen och Cleanup Agent samt kontrollerar hela kedjan före den första produktionskörningen. Värdnamn, DN:er och tjänstkonton med `example.com` är platshållare och måste anpassas till den egna miljön.

## Vilka användare som upptar en licens

Totemomail skiljer mellan två användarklasser. Endast interna användare räknas mot licensgränsen.

| Användartyp | Beskrivning | Licensrelevant |
| --- | --- | --- |
| Internal Users | Användare i den egna organisationen som skickar och tar emot krypterat | Ja |
| External Users | Externa kommunikationspartner (WebMail, PDF, S/MIME, PGP) | Nej |


En intern användare skapas så snart användaren kommunicerar via gatewayen för första gången. Det sker automatiskt. Borttagning sker däremot inte automatiskt: När en medarbetare lämnar organisationen inaktiverar ni vanligtvis AD-kontot. Totemomail-posten finns dock kvar. Med åren samlas därmed övergivna konton som fortsatt upptar licenser.

### Statusvisning

Aktuell status hittar ni under **Settings → Overview → User Information**.

![](../images/953te2zhdJ61lxda1mj04QrlQA.png)

*Available Users är* `*-17*`*. För de 4017 interna användarna finns ett mindre antal licensierade platser.*

De viktiga raderna:

-   **Internal users** (`4017`): skapade interna användare
    
-   **Internal blocked users** (`14`): blockerade, men fortsatt licensrelevanta
    
-   **Available Users** (`-17`): tillgängliga licenser; ett negativt värde innebär för få licenser
    

Så snart *Available Users* hamnar under noll visas varningen vid klockan:

![](../images/lcL4owxA3iEdg3L9ZFd2bIioE.png)

*„The licensed user limit has been reached." E-postflödet fortsätter, men meddelandet förblir permanent synligt.*

Viktigt: För få licenser blockerar inte e-postflödet. Det är ett licensrättsligt, inte ett tekniskt tillstånd. Ni har alltså tid för en korrekt lösning, men bör inte ignorera tillståndet permanent.

## Från omedelbar åtgärd till hållbar lösning

### Manuell radering

Ni kan söka efter och radera interna användare en och en under **Internal Users**. Det löser den akuta situationen, men problemet återkommer efter några månader. Med flera tusen konton blir detta ohållbart.

### LDAP-anslutning med Cleanup Agent

Den hållbara vägen är att ansluta till Active Directory via LDAP. En agent jämför regelbundet de interna användarna med katalogen och tar bort eller inaktiverar konton som inte längre finns i AD. Därmed blir AD den ledande källan, och er offboardingprocess i AD sköter samtidigt licenshygienen.

## LDAP-grunder

| Term | Betydelse |
| --- | --- |
| DN (Distinguished Name) | Unik sökväg till ett objekt, t.ex. `CN=John Doe,OU=Users,DC=corp,DC=example,DC=com` |
| Base DN / Search Base | Sökningens rot, t.ex. `DC=corp,DC=example,DC=com` |
| Bind DN | Konto som totemomail använder för att autentisera sig mot AD |
| Filter | LDAP-sökuttryck, t.ex. `(&(objectClass=user)(sAMAccountName=jdoe))` |


### Portar

| Port | Protokoll | Användning |
| --- | --- | --- |
| 389 | LDAP | okrypterat / STARTTLS |
| 636 | LDAPS | LDAP över TLS |
| 3268 | Global Catalog | skogsomfattande sökning, okrypterat |
| 3269 | Global Catalog SSL | skogsomfattande sökning över TLS |


I en miljö med en enda domän räcker port 636 mot en domänkontrollant. Om ni driver en skog med flera domäner ger endast Global Catalog (port 3269) skogsomfattande resultat. En DC på port 636 känner enbart till objekten i sin egen domän och besvarar sökningar utanför sin partition med en referral (en detalj som ofta förbises i miljöer med flera domäner).

### userAccountControl

Om ett AD-konto är inaktiverat anges i bitfältet `userAccountControl`. Flaggan `ACCOUNTDISABLE` har värdet `2`. Med LDAP-matchingregeln `1.2.840.113556.1.4.803` (`LDAP_MATCHING_RULE_BIT_AND`) kan ni utvärdera enskilda bitar:

```text
# Aktive Benutzer
(&(objectClass=user)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))

# Deaktivierte Benutzer
(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))
```

## Steg 1: Tjänstkonto i AD

För anslutningen skapar ni ett dedikerat konto med endast läsrättigheter. Använd inte ett administratörskonto för detta. Bind-användaren behöver bara kunna läsa AD.

```powershell
New-ADUser -Name "svc-totemomail-ldap" `
  -SamAccountName "svc-totemomail-ldap" `
  -UserPrincipalName "svc-totemomail-ldap@corp.example.com" `
  -Path "OU=Service Accounts,DC=corp,DC=example,DC=com" `
  -AccountPassword (Read-Host -AsSecureString "Lösenord") `
  -PasswordNeverExpires $true `
  -Enabled $true
```

En vanlig domänanvändare kan redan läsa AD, så kontot behöver inga ytterligare rättigheter. För lösenordet rekommenderas ett långt, slumpmässigt värde som lagras i ert lösenordsvalv.

Om er säkerhetspolicy föreskriver det kan ni även använda ett gMSA (Group Managed Service Account). Totemomail förväntar sig dock Bind-DN och lösenord, varför ett klassiskt tjänstkonto med `PasswordNeverExpires` vanligen används i praktiken.

## Steg 2: Kontrollera LDAP-anslutningen på kommandoraden

Innan ni konfigurerar något i totemomail bör ni verifiera LDAP-anslutningen på kommandoraden. Detta är steget som de flesta hoppar över. Om `ldapsearch` fungerar, fungerar även anslutningen i totemomail. Om testet misslyckas vet ni åtminstone var det fastnar, i stället för att gissa i totemomail-GUI:t.

### 2.1 Portkontroll

Under Linux, exempelvis från totemomail-appliancen:

```bash
nc -vz dc01.corp.example.com 636
nmap -p 389,636,3268,3269 dc01.corp.example.com
```

Under Windows med PowerShell:

```powershell
Test-NetConnection -ComputerName dc01.corp.example.com -Port 636
```

Om ingen anslutning kan upprättas här har ni ett brandväggs- eller routningsproblem, inte ett LDAP-problem.

### 2.2 Kontrollera TLS-certifikatet

I praktiken misslyckas LDAPS oftast på grund av certifikatet. Kontrollera därför vad DC:n levererar:

```bash
openssl s_client -connect dc01.corp.example.com:636 -showcerts </dev/null
```

Var uppmärksam på två saker:

-   `**subject=**` **/** `**issuer=**`: Värdnamnet i certifikatet (CN respektive SAN) måste stämma överens med värdnamnet som ni ansluter till. Om ni ansluter via IP-adressen misslyckas kontrollen om certifikatet endast innehåller FQDN.
    
-   `**Verify return code: 0 (ok)**`: Utfärdande CA måste vara känd för totemomail. Vid en intern Enterprise-CA måste ni importera dess rot- eller utfärdarcertifikat till totemomails truststore.
    

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

| Flagga | Betydelse |
| --- | --- |
| `-x` | Enkel autentisering (Bind-DN och lösenord) |
| `-H` | LDAP-URI inklusive schema (`ldaps://`) och port |
| `-D` | Bind-DN |
| `-W` | Fråga efter lösenord interaktivt |
| `-b` | Sökbas |
| därefter | Filter, följt av attributen som ska returneras |


Om frågan returnerar objektet med dess attribut fungerar anslutningen. Hur många konton som är inaktiverade i AD fastställer ni med bitfiltret:

```bash
ldapsearch -x -H ldaps://dc01.corp.example.com:636 \
  -D "CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com" -W \
  -b "DC=corp,DC=example,DC=com" \
  "(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))" \
  sAMAccountName | grep -c sAMAccountName
```

### 2.4 Verktyg under Windows

`**ldp.exe**` är Microsofts grafiska LDAP-verktyg, som finns på varje DC och ingår i RSAT. Ni ansluter via `Connection → Connect` (värd, port 636, aktivera SSL), autentiserar er med `Connection → Bind` och navigerar genom katalogträdet via `View → Tree` med Base DN.

Utan RSAT kan ni använda ADSI-sökaren i PowerShell:

```powershell
$searcher = [adsisearcher]"(&(objectClass=user)(sAMAccountName=jdoe))"
$searcher.SearchRoot = [adsi]"LDAP://dc01.corp.example.com/DC=corp,DC=example,DC=com"
$searcher.FindOne().Properties
```

Med RSAT och AD-modulen går det kortare:

```powershell
Get-ADUser -Server dc01.corp.example.com `
  -SearchBase "DC=corp,DC=example,DC=com" `
  -Filter "Enabled -eq '$true'" |
  Measure-Object
```

Klassiskt via `dsquery`, tillgängligt på varje DC:

```bash
dsquery user -disabled -limit 0
```

Först när ett av dessa test körs utan problem går ni vidare i totemomail.

## Steg 3: Konfigurera LDAP-anslutningen i totemomail

LDAP-katalogen skapar ni i Admin-GUI:t under **Directories / LDAP**. Ange exakt de värden som ni tidigare har testat:

| Fält | Exempelvärde |
| --- | --- |
| Host / URL | `ldaps://dc01.corp.example.com:636` |
| Bind DN | `CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com` |
| Bind Password | Tjänstkontots lösenord |
| Base DN | `DC=corp,DC=example,DC=com` |
| User Filter | `(&(objectClass=user)(objectCategory=person))` |
| Login Attribute | `sAMAccountName` (alternativt `mail` eller `userPrincipalName`) |


Om ni använder LDAPS mot en intern CA måste ni importera dess rot- eller utfärdarcertifikat till totemomails truststore. Annars misslyckas TLS-handshaken med ”certificate verify failed”, även om `ldapsearch` tidigare fungerade med `-x`: `ldapsearch` kontrollerar nämligen inte certifikatet strikt på detta sätt.

Efter att ha sparat utlöser ni den inbyggda testanslutningen. Den bekräftar bindningen.

## Steg 4: Skapa Cleanup Agent

Under **Maintenance → Agents → Add** skapar ni en agent av typen **„Check presence of internal users in directories"**.

### 4.1 Fliken „Schedule"

![](../images/oSiutQSlKTW0tMY5HUtWCMGuXQ.png)

*Agenten körs här varje månad den 1:a klockan 00:30. Med „Agent runs on server" anger ni den körande noden i klustret.*

| Fält | Rekommendation | Motivering |
| --- | --- | --- |
| The agent should run | `monthly`, dag `1`, `00:30` | utanför arbetstid; månadsvis räcker för licenshygienen |
| Agent enabled | aktivera först efter testkörningen | se steg 5 |
| Produced emails are not sent but cached in a queue | aktivera för första körningen | testkörning utan e-postutskick |
| Agent runs on server | en nod i klustret | jobbet ska bara köras på en nod |


### 4.2 Fliken „Parameters"

![](../images/Y6XzxZWGYIcZoJnZkFL0vUHXxQ.png)

*Parametrarna styr vilka interna användare som raderas, inaktiveras eller skapas.*

| Parameter | Rekommendation | Effekt |
| --- | --- | --- |
| Delete inactive users that are not found in a directory? | aktivera | Inaktiva interna användare utan AD-post raderas. Detta är kärnan i licensrensningen. |
| Delete blocked users that are not found in a directory? | aktivera | Blockerade interna användare utan AD-post raderas också |
| Delete administrators? | lämna tomt | Administratörskonton ska inte raderas automatiskt |
| Only set users found in the defined groups to inactive | valfritt | Användare sätts till inaktiva i stället för att raderas. Ett inledande `!` undantar medlemmarna i den angivna gruppen. Separera DN:er med `;`. |
| Additional filter attribute | valfritt | Ytterligare attribut för sökning i katalogen, t.ex. `proxyAddresses` |
| Delete inactive/blocked users that are found in the defined groups | lämna tomt | gäller endast om gruppparametern är angiven |
| Create users based on group membership | valfritt | skapar nya interna användare baserat på AD-gruppmedlemskap. Separera flera grupper med `;`. |


Negationen i fältet *„Only set users found in the defined groups to inactive"* fungerar med ett `!` före en grupp-DN. Medlemmarna i denna grupp undantas från åtgärden:

```text
CN=Mitarbeiter,OU=Groups,DC=corp,DC=example,DC=com;!CN=Dienstkonten,OU=Groups,DC=corp,DC=example,DC=com
```

I detta exempel sätter ni användare i gruppen *Medarbetare* till inaktiva när de saknas i AD, medan medlemmar i gruppen *Tjänstkonton* lämnas orörda.

## Steg 5: Testkörning och validering

Låt inte agenten köras mot produktionsbeståndet utan en testkörning. Gör i stället så här:

1.  **Aktivera köläge**: via alternativet *„Produced emails are not sent but cached in a queue"*. Agenten fastställer de planerade åtgärderna utan att skicka e-post.
    
2.  **Kör manuellt** och utvärdera agentloggen: Hur många användare skulle påverkas, och finns oväntade konton som funktionsbrevlådor i listan?
    
3.  **Rimlighetskontroll mot** `**ldapsearch**`: Antalet användare som inte hittas i AD bör stämma överens med den manuella LDAP-frågan.
    
4.  Om resultatet stämmer inaktiverar ni köläget, sätter *Agent enabled* och aktiverar schemat.
    
5.  Efter den första produktionskörningen kontrollerar ni **Settings → Overview → User Information** igen. *Available Users* bör då åter ligga på ett positivt värde.
    

## Felsökning

| Symptom | Orsak | Åtgärd |
| --- | --- | --- |
| `Can't contact LDAP server` | Port 636 är inte nåbar / fel värd | kontrollera med `Test-NetConnection` respektive `nc -vz`, kontrollera brandväggen |
| `Invalid credentials (49)` | Bind-DN eller lösenord är fel | Ange Bind-DN som fullständigt DN, inte som `user@domain` |
| `certificate verify failed` | CA är okänd för truststore | Importera rot- eller utfärdar-CA |
| Värdnamnsmismatch i TLS | Anslutning via IP i stället för FQDN | Använd certifikatets CN/SAN som värd |
| `Referral (10)` | Sökningen passerar domängränsen | Använd Global Catalog på port 3269 i stället för DC på 636 |
| Inaktiverade användare identifieras inte | saknat `userAccountControl`\-filter | Använd bit-matchingregeln `:1.2.840.113556.1.4.803:=2` |
| Agenten raderar för många konton | Filtret är för brett / Base DN fel | Testa i köläge, begränsa Base DN |


Med flaggan `-d 1` ger `ldapsearch` debug-utdata för anslutningsuppbyggnaden:

```bash
ldapsearch -d 1 -x -H ldaps://dc01.corp.example.com:636 ...
```

Då ser ni om TLS-handshaken eller först bindningen misslyckas. Denna skillnad visar totemomail-GUI:t inte i sitt generiska felmeddelande.

## Säkerhet

-   **Skrivskyddat tjänstkonto.** Bind-användaren behöver endast läsrättigheter.
    
-   **LDAPS i stället för LDAP.** Använd port 636 respektive 3269. LDAP på port 389 överför bind-lösenordet i klartext. Active Directory kräver dessutom i allt högre grad säkrade anslutningar med LDAP Channel Binding och Signing.
    
-   **Lösenordsrotation.** `PasswordNeverExpires` är praktiskt genomförbart i drift. Dokumentera kontot och rotera lösenordet enligt plan.
    
-   **Övervakning.** Övervaka *Available Users* (helst med larm), i stället för att vänta på klockvarningen.
    
-   **Första körningen i köläge.** Ett felaktigt filter kan träffa ett stort antal konton.
    

## Det säkra förfarandet i fyra steg

Att nå licensgränsen är inte ett tekniskt fel, utan följden av en saknad offboardingprocess. Den hållbara lösningen är regelbunden synkronisering mot Active Directory som ledande källa. Avgörande är ordningen:

1.  Verifiera LDAP-anslutningen på kommandoraden (`ldapsearch`, `openssl s_client`, `Test-NetConnection`)
    
2.  Konfigurera anslutningen i totemomail
    
3.  Validera agenten i köläge
    
4.  Sätt agenten i produktion
    

Den som följer denna ordning löser det akuta licensproblemet och förhindrar att det återkommer.

## Källor

1.  [totemo / Kiteworks – totemomail (Email Protection Gateway)](https://totemo.com/en/resources/downloads): Produktdokumentation för totemomail (licensmodell, LDAP-anslutning, Cleanup Agent); tekniken drivs vidare hos Kiteworks som Email Protection Gateway.
    
2.  [Microsoft Learn – «UserAccountControl property flags»](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties): Flaggornas betydelse, bl.a. `ACCOUNTDISABLE` (0x0002) och `NORMAL_ACCOUNT`.
    
3.  [Microsoft Learn – «Search Filter Syntax»](https://learn.microsoft.com/en-us/windows/win32/adsi/search-filter-syntax): Bitvis LDAP-filter via matchingregelns OID `1.2.840.113556.1.4.803` (LDAP\_MATCHING\_RULE\_BIT\_AND).
    
4.  [OpenLDAP – «ldapsearch» (manpage)](https://www.openldap.org/software/man.cgi?query=ldapsearch): Anropsalternativ (`-x`, `-H ldaps://`, `-D`, `-W`, `-b`) för bindning och sökning.
    
5.  [Microsoft Learn – «Service overview and network port requirements»](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/service-overview-and-network-port-requirements): LDAP-portarna 389/636 samt Global Catalog-portarna 3268/3269.
