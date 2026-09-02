---
title: "Förnya certifikatet på Cisco SMA"
navTitle: "SMA-certifikat"
description: "Certifikat kan bara installeras på Cisco SMA via CLI, och aktuella AsyncOS-versioner validerar hela kedjan vid import: utan en lagrad rot-CA misslyckas den. Artikeln visar vägarna till det nya nyckelparet, OpenSSL-metoden i detalj, hur du hanterar RC2-40-CBC-felet i OpenSSL 3 och importerar den interna rot-CA:n till apparatens truststore."
date: "2026-08-04"
kategorie: "Cisco ESA / SMA"
timeToRead: "11 min läsning"
themen:
  - cisco-esa-sma
  - smtp-mailflow
hauptthema: "cisco-esa-sma"
slug: "fornya-certifikatet-pa-cisco-sma"
translationId: "article-69d93a1e5e081848"
aiPrompt: |
  Du bist mein Assistent für die Zertifikatserneuerung auf einer Cisco SMA (Secure Email and Web Manager). Führe mich Schritt für Schritt durch den Ablauf aus diesem Artikel: 1. Wahl des Wegs zum Schlüsselpaar (OpenSSL-CSR in der eigenen Umgebung, PFX von der CA oder Umweg über eine ESA), 2. CN- und SAN-Liste für meine Hostnamen, 3. je nach Weg CSR-Erzeugung mit OpenSSL oder Konvertierung der PFX-Datei nach PEM inklusive Umgang mit dem Fehler RC2-40-CBC, 4. bei interner CA Import der Root-CA in die Custom-Liste der Appliance, 5. Installation über certconfig in der CLI, 6. Kontrolle. Frage mich zuerst nach den Hostnamen meiner Appliances und der Quarantäneseite, ob die ausstellende CA intern oder öffentlich ist und welche OpenSSL-Version ich installiert habe. Passe alle Befehle an meine Dateinamen an und erinnere mich vor dem Abschluss daran, die certconfig-Session nicht mit Ctrl+C zu beenden und die Änderung mit commit zu aktivieren.
translationOf: cisco-sma-zertifikat-erneuern
translationSourceHash: c99ce64a5e63875b84c7b6f14a7f2fb7e51290fedbdc93d99201cdc97a743508
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T09:14:07.689Z
translationReview: automatic
url: https://rafaelpfister.ch/sv/blog/fornya-certifikatet-pa-cisco-sma
---

# Förnya certifikatet på Cisco SMA

Cisco SMA (Security Management Appliance, numera under namnet Cisco Secure Email and Web Manager) hanterar i många e-postmiljöer den centrala spamkarantänen och rapporteringen för Secure Email-gatewayerna. Dess HTTPS-certifikat omfattar administratörsgränssnittet och karantänsidan, där slutanvändare kan granska och frisläppa sina kvarhållna e-postmeddelanden. När det löper ut bryts inget e-postflöde. Utgången blir ändå omedelbart synlig: varje besök på karantänsidan avslutas med en certifikatvarning i webbläsaren, och just de användare som utbildas i säkerhetsmedvetenhet att inte klicka vidare vid sådana varningar förväntas då ignorera dem.

Vid en förnyelse i ett kundprojekt uppstod två problem: först svarade OpenSSL 3 på den interna CA:ns PFX-fil med ett kryptiskt fel om `RC2-40-CBC`, sedan vägrade apparaten att importera det färdiga certifikatet eftersom den inte kände till den utfärdande rot-CA:n. Båda hindren och deras lösningar beskrivs nedan.

## Vad SMA gör annorlunda än ESA

På ESA kan hela certifikatets livscykel hanteras via GUI:t (`Network > Certificates`). SMA kan inte det: servercertifikatet installeras uteslutande via CLI, med kommandot `certconfig` i en SSH-session. SMA:s GUI visar bara certifikat; endast listorna över betrodda certifikatutfärdare kan underhållas där, mer om det senare.

Därtill finns två ytterligare särdrag:

- Klistra in-dialogen accepterar bara PEM-formatet. En PFX-fil (PKCS#12) måste konverteras före installationen; aktuella AsyncOS-versioner erbjuder dessutom direktimport av PKCS#12, men filen måste först överföras till apparaten.
- Äldre AsyncOS-versioner (enligt Cisco-technoten) skapar själva varken nycklar eller CSR:er, så nyckelparet måste skapas utanför; de tre möjliga vägarna beskrivs längre ned. Aktuella versioner kan med `certconfig > CERTIFICATE > NEW` skapa ett självsignerat certifikat med CSR direkt på apparaten. För ett gemensamt certifikat över flera apparater hjälper det dock inte, eftersom den privata nyckeln då aldrig lämnar apparaten.

Ett enskilt certifikat kan antingen användas för alla tjänster (inkommande och utgående TLS, HTTPS-administrationsåtkomst, LDAPS) eller lagras separat för varje tjänst. Detta styrs i dialogen `certconfig`; kommandots rubrik visar alltid den aktiva tilldelningen (`Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.`). Det finns ingen separat tilldelningsdialog som på ESA, och inget kan ändras via GUI:t. I de flesta miljöer är ett certifikat för allt det pragmatiska valet: namnlistan omfattar ändå apparaternas FQDN:er, och separata nyckelpar mångdubblar arbetet vid varje förnyelse.

Att dialogen på en karantänapparat frågar efter inkommande och utgående TLS kan först verka förvirrande, eftersom SMA inte står i någon MX-sökväg. Den kommunicerar ändå SMTP i båda riktningarna. Inkommande (Receiving) är mottagningssidan: ESA:erna levererar med SMTP karantänsatta meddelanden till SMA, till den centrala spamkarantänen på port 6025 och till de centrala policy-, virus- och outbreak-karantänerna på port 7025; de senare anslutningarna är TLS-krypterade som standard, och där presenterar SMA just detta certifikat. Utgående (Delivery) är sändsidan: när en användare frisläpper ett meddelande från karantänen levererar SMA själv det tillbaka till e-postflödet via sina SMTP-rutter, och apparaten skickar även karantännotifieringar, schemalagda rapporter och varningar som egna e-postmeddelanden. För förnyelsen innebär det: HTTPS är i praktiken kritiskt, medan de båda SMTP-tjänsterna helt enkelt följer med i certifikatet för alla tjänster.

## Fastställ namn: CN och SAN

Oavsett vägen till nyckelparet fastställs först namnlistan. Common Name ska vara värdnamnet som användarna använder för att öppna karantänsidan. SAN-listan ska dessutom innehålla apparaternas FQDN:er så att även direkt åtkomst till administratörsgränssnittet fungerar utan varning. För en miljö med två apparater ser namnlistan ut så här:

| Fält | Värde |
|---|---|
| CN | `spam-quarantine.example.ch` |
| SAN | `spam-quarantine.example.ch` |
| SAN | `sma01.example.ch` |
| SAN | `sma02.example.ch` |

Två anmärkningar: webbläsare utvärderar sedan länge endast SAN-poster, så CN räcker inte ensam. Karantänens värdnamn måste därför också finnas som SAN. Korta värdnamn utan domändel (exempelvis `SMA01`) utfärdas dessutom endast av en intern CA; offentliga CA:er signerar inga interna namn.

## Tre vägar till det nya nyckelparet

För ett certifikat som omfattar flera apparater och karantänens värdnamn måste nyckelparet skapas utanför apparaten. Tre vägar har etablerats:

1. Skapa nyckel och CSR med OpenSSL i den egna miljön. Den privata nyckeln skapas där den behövs och lämnar aldrig miljön. Detta är den rekommenderade vägen, med detaljer i nästa avsnitt.
2. CA:n skapar nyckelparet och levererar en PFX-fil. Det fungerar, men har två nackdelar: nyckeln passerar genom andra händer (lösenordet bör därför skickas via en separat kanal och inte i samma e-postmeddelande som filen), och beroende på CA-verktyget kan en RC2-krypterad PFX returneras som OpenSSL 3 bara öppnar med extra arbete; mer om det nedan.
3. Omvägen via en ESA, dokumenterad i Cisco-technoten: skapa där under `Network > Certificates` ett certifikat med SMA:ns CN, hämta CSR:en och låt CA:n signera den, ladda upp det signerade certifikatet till ESA:n igen och exportera allt som PFX. Även här krävs i slutändan konvertering till PEM.

## De viktigaste alternativen i openssl

För orientering följer här underkommandona och alternativen i `openssl` som förekommer i denna artikel, fritt översatta från OpenSSL-dokumentationen:

<details class="options-details">
<summary>Översikt över alternativ</summary>

| Alternativ | Betydelse |
|---|---|
| `req` | Underkommando för certifikatbegäranden (CSR): skapa, visa, kontrollera |
| `-new` | Skapar en ny begäran |
| `-newkey rsa:2048` | Skapar samtidigt ett nytt RSA-nyckelpar med 2048 bitar |
| `-noenc` | Skriver den privata nyckeln okrypterad (fram till OpenSSL 3.0: `-nodes`) |
| `-keyout datei` | Målfil för den privata nyckeln |
| `-out datei` | Målfil för utdata, här CSR respektive PEM |
| `-subj text` | Begärans subject i formatet `/C=…/O=…/CN=…` |
| `-addext text` | Lägger till en utökning till begäran, här SAN-listan |
| `pkcs12` | Underkommando för PKCS#12-behållare (PFX): skapa och packa upp |
| `-in datei` | Indatafil |
| `-legacy` | Läser även in Legacy-providern för äldre algoritmer som RC2 |
| `list` | Underkommando för att visa installationens funktioner |
| `-providers` | Listar de inlästa providerna |
| `-provider name` | Läser dessutom in angiven provider för detta anrop |
| `s_client` | Underkommando: TLS-testklient för anslutningar till en server |
| `-connect host:port` | Målserver och port för TLS-anslutningen |
| `-servername name` | Anger Server Name Indication (SNI) i TLS-handskakningen |
| `x509` | Underkommando för att visa och bearbeta certifikat |
| `-noout` | Undertrycker utmatningen av det kodade certifikatet |
| `-subject` | Skriver ut certifikatets subject |
| `-enddate` | Skriver ut utgångsdatumet (notAfter) |

</details>

De fullständiga referenserna finns i OpenSSL-dokumentationen som en egen manpage per underkommando: `openssl-req(1)`, `openssl-pkcs12(1)`, `openssl-s_client(1)` och `openssl-x509(1)`.

## Starta OpenSSL på Windows

Alla följande steg körs med OpenSSL på ett system inom miljön, till exempel en administratörsserver. Light Edition av Windows-byggnaderna från Shining Light Productions räcker; installationsprogrammet är cirka 6 MB stort och kan verifieras mot kontrollsummelistan som slproweb publicerar.

Installationsprogrammet lägger allt under `C:\Program Files\OpenSSL-Win64` och den körbara filen finns i `bin\openssl.exe`. Den läggs inte till i sökvägen: den som skriver `openssl` i en ny kommandotolk får ett felmeddelande. Det finns tre möjligheter:

- Öppna posten `Win64 OpenSSL Command Prompt` i Start-menyn. Den startar `start.bat` från installationskatalogen, ställer in miljön och hälsar med utdata från `openssl version -a`. I detta fönster fungerar `openssl` direkt.
- Ange hela sökvägen: `"C:\Program Files\OpenSSL-Win64\bin\openssl.exe" version`.
- Lägg permanent till `C:\Program Files\OpenSSL-Win64\bin` i miljövariabeln `Path`; därefter är `openssl` tillgängligt i varje skal.

Den som redan använder Git för Windows klarar sig utan ytterligare installation: det innehåller sitt eget OpenSSL (`C:\Program Files\Git\mingw64\bin\openssl.exe`), och i Git Bash finns det omedelbart i sökvägen. Aktuella Git-versioner levererar OpenSSL 3.5 med aktiv Legacy-provider, så `-legacy` från avsnittet om PFX-konvertering fungerar där också. Kontrollera det så här:

```bash
openssl list -providers -provider legacy
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `list` | Visar funktionerna i OpenSSL-installationen |
| `-providers` | Listar de inlästa providerna med namn, version och status |
| `-provider legacy` | Läser dessutom in providern `legacy` för detta anrop; om den visas i listan är den tillgänglig |

</details>

Git Bash har dock en egenhet: den tolkar argument som börjar med `/` som sökvägar och skriver om dem. Av `-subj "/C=CH/O=Example AG/CN=..."` blir `C:/Program Files/Git/C=CH/O=Example AG/CN=...`, och OpenSSL avbryts:

```text
req: subject name is expected to be in the format /type0=value0/type1=value1/type2=...
where characters may be escaped by \. This name is not in that format:
'C:/Program Files/Git/C=CH/O=Example AG/CN=spam-quarantine.example.ch'
```

Ett inledande `MSYS_NO_PATHCONV=1` stänger av omskrivningen för det enskilda anropet. Problemet uppstår inte i kommandotolken, PowerShell eller OpenSSL Command Prompt.

## Skapa nyckel och CSR med OpenSSL

Ett enda anrop skapar nyckel och CSR med den fullständiga SAN-listan:

```bash
openssl req -new -newkey rsa:2048 -noenc \
  -keyout spam-quarantine.example.ch.key \
  -out spam-quarantine.example.ch.csr \
  -subj "/C=CH/O=Example AG/CN=spam-quarantine.example.ch" \
  -addext "subjectAltName=DNS:spam-quarantine.example.ch,DNS:sma01.example.ch,DNS:sma02.example.ch"
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-new` | Skapar en ny certifikatbegäran (CSR) |
| `-newkey rsa:2048` | Skapar samtidigt ett nytt RSA-nyckelpar med 2048 bitar |
| `-noenc` | Skriver den privata nyckeln okrypterad till filen |
| `-keyout …` | Målfil för den privata nyckeln |
| `-out …` | Målfil för CSR:en |
| `-subj …` | Subject med land, organisation och Common Name |
| `-addext …` | Lägger till SAN-utökningen med alla DNS-namn i begäran |

</details>

CSR-filen skickas till CA:n, medan nyckeln stannar på servern. Tillbaka kommer det signerade certifikatet med intermediate-certifikat, vanligen direkt som PEM. Därmed finns allt klart för installationen, och PFX-konvertering faller helt bort med denna metod.

Nyckelfilen är okrypterad (`-noenc`), eftersom `certconfig` förväntar sig den i just detta format. Fram till installationen ska den förvaras på servern med restriktiva behörigheter, och därefter raderas eller flyttas till lösenordshanteraren.

## Konvertera PFX till PEM

Detta och nästa avsnitt gäller vägarna 2 och 3, som avslutas med en PFX-fil. `certconfig` förväntar sig certifikat och privat nyckel som PEM och nyckeln okrypterad. Ett enda OpenSSL-anrop utför båda momenten:

```bash
openssl pkcs12 -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `pkcs12` | Underkommando för att skapa och packa upp PKCS#12-behållare |
| `-in …` | PFX-indatafilen |
| `-out …` | PEM-utdatafilen med certifikat, nyckel och kedjecertifikat |
| `-noenc` | Skriver den privata nyckeln utan lösenfras (fram till OpenSSL 3.0 hette alternativet `-nodes`) |

</details>

Importlösenordet efterfrågas utan eko, och inga asterisker visas. Den skapade PEM-filen innehåller certifikat, nyckel och medföljande kedjecertifikat i en fil och är därför skyddsvärd: radera den efter installationen eller flytta den till lösenordshanteraren.

## När OpenSSL 3 vägrar PFX-filen

Med äldre PFX-filer avbryts konverteringen i OpenSSL 3.x med följande meddelande:

```text
Error outputting keys and certificates
error:0308010C:digital envelope routines:inner_evp_generic_fetch:unsupported:
crypto/evp/evp_fetch.c: Global default library context, Algorithm (RC2-40-CBC : 0)
```

Orsaken är inte en skadad fil, utan ett designbeslut: OpenSSL 3 har flyttat äldre algoritmer som RC2, RC4 och DES till en separat Legacy-provider som inte laddas som standard. Många PFX-exporter från äldre Windows-system och CA-verktyg krypterar dock certifikatdelen av behållaren just med RC2-40-CBC. OpenSSL 1.1 öppnade sådana filer utan problem, medan OpenSSL 3 avvisar dem.

Lösningen är ett enda extra alternativ:

```bash
openssl pkcs12 -legacy -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-legacy` | Läser in Legacy-providern för detta anrop; därmed blir äldre algoritmer som RC2-40-CBC tillgängliga igen och konverteringen slutförs |

</details>

Förutsättningen är en OpenSSL-installation som innehåller Legacy-providern; så är fallet för vanliga Windows-byggnader.

Den som vill bli av med felet permanent kan åtgärda källan och exportera PFX-filen med modern kryptering: aktuella exportdialoger och CA-verktyg erbjuder AES-256, vilket gör Legacy-omvägen helt överflödig.

Som grafiskt alternativ fungerar XCA (X Certificate and Key Management): importera PFX-filen via `Importieren > PKCS#12`, exportera sedan certifikatet som PEM på fliken `Zertifikate` och nyckeln separat som okrypterad PEM på fliken `Private Schlüssel`. Båda exporterna behövs, eftersom `certconfig` frågar efter certifikat och nyckel var för sig. XCA innehåller sitt eget kryptobibliotek och öppnar även behållare med äldre algoritmer.

Ett ord också om nedladdningskällan: OpenSSL-projektet publicerar självt inga Windows-binärfiler utan hänvisar till tredjepartsbyggen som Win64 OpenSSL från Shining Light Productions. Nedladdningsportaler med egna installationsprogram är fel ställe för ett kryptografiskt verktyg.

## Importera intern rot-CA till apparatens truststore först

Aktuella AsyncOS-versioner validerar hela kedjan när en certifikatprofil skapas. Om certifikatet kommer från en intern CA vars rot apparaten inte känner till, avbryts importen med följande meddelande:

```text
Cannot import certificate: The certificate authority validation could not be
trusted for certificate because unable to get local issuer certificate
```

Apparaten har två listor över betrodda certifikatutfärdare: den medföljande systemlistan och en Custom-lista för egna CA:er. Den interna rot-CA:n hör hemma i Custom-listan, innan servercertifikatet installeras. Endast det offentliga CA-certifikatet som PEM-fil behövs (`-----BEGIN CERTIFICATE-----` till `-----END CERTIFICATE-----`), ingen privat nyckel.

Så här laddas rot-CA:n upp till apparaten via webbgränssnittet:

1. Öppna `Network > Certificates`.
2. Klicka på `Edit Settings` i avsnittet `Certificate Authorities`.
3. Välj alternativet `Enable` vid `Custom List`.
4. Ladda upp PEM-filen via `Choose File`.
5. Kör `Submit` och sedan `Commit Changes`.
6. Kontrollera under `Network > Certificates > Manage Trusted Root Certificates` att CA:n visas i listan över anpassade certifikat.

Om det redan finns en Custom-lista ska den exporteras först och den nya CA:n läggas till i det befintliga PEM-paketet: importen ersätter listan, annars försvinner tidigare sparade CA:er. I en kedja med mellansteg ska rot-CA:n importeras först och därefter intermediate-CA:n. AsyncOS kontrollerar vid import bland annat utgångsdatum, dubbletter och satt `CA:TRUE`-flagga och avvisar en intermediate så länge motsvarande rot saknas. Samma import kan även göras via CLI: `certconfig > CERTAUTHORITY > CUSTOM > IMPORT`, följt av `commit`.

Två avgränsningar: för uppdateringar via en TLS-inspekterande proxy använder SMA en separat truststore (`updateconfig > TRUSTED_CERTIFICATES > ADD`), Custom-CA-listan används inte där. Och en rot-CA på SMA eliminerar inte webbläsarvarningar: klienterna behöver fortfarande roten via sin egen certifikatdistribution, vanligtvis via GPO, och apparaten måste leverera servercertifikatet tillsammans med intermediate-certifikatet.

## Installation med certconfig

Logga in på SMA via SSH och starta `certconfig`. I aktuella AsyncOS-versioner arbetar dialogen med certifikatprofiler:

```text
sma01.example.ch> certconfig

Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.

Choose the operation you want to perform:
- CERTIFICATE - Import, Create a request, Edit or Remove Certificate Profiles
- CERTAUTHORITY - Manage System and Customized Authorities
- CRL - Manage Certificate Revocation Lists
[]> certificate
```

Bakom `CERTIFICATE` finns operationerna `IMPORT` (PKCS#12-fil som tidigare laddats upp till apparaten), `PASTE` (klistra in certifikat i CLI), `NEW` (skapa självsignerat certifikat med CSR), `EDIT`, `EXPORT`, `DELETE` och `PRINT` (visar tilldelningen till tjänsterna). Den vanliga vägen via SSH är `PASTE`: dialogen frågar efter ett namn för profilen, sedan certifikatet, den privata nyckeln och valfritt CA:ns intermediate-certifikat, alltid som PEM-block som avslutas med ett enda `.` på en egen rad. Den avslutande frågan om FQDN-kontroll av Common Name kan besvaras med standardvärdet. Intermediate-certifikatet ska ingå i profilen, annars saknar klienterna kedjan och beroende på webbläsare kan varningen kvarstå trots ett giltigt certifikat.

Äldre AsyncOS-versioner (enligt Cisco-technoten) visar i stället en dialog `SETUP`. Den börjar med frågan `Do you want to use one certificate/key for receiving, delivery, HTTPS management access, and LDAPS?`: ett `y` tilldelar samma par till alla fyra tjänster, medan ett `n` går igenom frågorna om certifikat, nyckel och intermediate en gång per tjänst. Principen för inklistring är densamma.

Två punkter avgör om det lyckas eller misslyckas: avsluta inte sessionen med Ctrl+C, eftersom det omedelbart förkastar alla ändringar. Och kör `commit` i slutet; först då aktiveras certifikatet. Med två apparater upprepas processen på båda, eftersom certifikatkonfigurationen inte synkroniseras mellan SMA:er.

## Kontroll

Det snabbaste testet körs utifrån mot karantänsidan. Slutanvändaråtkomsten till spamkarantänen ligger som standard på HTTPS-port 83, om inget annat konfigurerades vid aktiveringen:

```bash
openssl s_client -connect spam-quarantine.example.ch:83 \
  -servername spam-quarantine.example.ch </dev/null 2>/dev/null |
  openssl x509 -noout -subject -enddate
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `s_client` | TLS-testklient: upprättar anslutningen och skickar vidare det presenterade certifikatet |
| `-connect …:83` | Målserver och port, här HTTPS-porten för spamkarantänen |
| `-servername …` | Anger Server Name Indication (SNI) så att servern levererar rätt certifikat |
| `x509` | Bearbetar det vidarebefordrade certifikatet |
| `-noout` | Undertrycker utmatningen av det kodade certifikatet |
| `-subject` | Skriver ut certifikatets subject |
| `-enddate` | Skriver ut utgångsdatumet (notAfter) |

</details>

Utdata måste visa det nya subjectet och det nya utgångsdatumet. På apparaten listar `certconfig` med operationen `PRINT` de aktiva certifikaten, och en webbläsarkontroll mot administratörsgränssnittet och karantänsidan bekräftar att kedjan är korrekt uppbyggd.

## Källor

1.  [How to Generate and Install a Certificate on an SMA](https://www.cisco.com/c/en/us/support/docs/security/content-security-management-appliance/118460-technote-sma-00.html): Cisco-technote med certconfig-processen för äldre AsyncOS-versioner, PEM-kravet och metoderna för att skapa certifikat via ESA eller OpenSSL.

2.  [Common Administrative Tasks, AsyncOS 16.0 för Cisco Secure Email and Web Manager](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_01011.html): Kapitel i administratörsguiden om hantering av Certificate Authority-listorna (system- och Custom-lista), inklusive kontrollerna vid CA-import.

3.  [Comprehensive Spam Quarantine Setup Guide on ESA and SMA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118692-configure-esa-00.html): Cisco-guide till spamkarantänen, inklusive slutanvändaråtkomst via HTTPS på port 83.

4.  [openssl-req](https://docs.openssl.org/master/man1/openssl-req/): Referens för att skapa nyckel och CSR, inklusive `-addext` för SAN-listan.

5.  [openssl-pkcs12](https://docs.openssl.org/master/man1/openssl-pkcs12/): Referens för konverteringsalternativen, bland annat `-noenc` (tidigare `-nodes`) och `-legacy`.

6.  [OpenSSL 3.0 Migration Guide](https://docs.openssl.org/3.0/man7/migration_guide/): Bakgrund om flytten av äldre algoritmer till Legacy-providern.

7.  [XCA: X Certificate and Key Management](https://hohnstaedt.de/xca/): Verktyg med öppen källkod för import och export av PKCS#12- och PEM-strukturer.

8.  [Win32/Win64 OpenSSL](https://slproweb.com/products/Win32OpenSSL.html): Windows-byggnader från Shining Light Productions som OpenSSL-projektet hänvisar till, inklusive publicerad kontrollsummelista.

9.  [MSYS2: Filesystem Paths](https://www.msys2.org/docs/filesystem-paths/): Beskrivning av den automatiska sökvägsomskrivningen som ändrar argumentet `-subj` i Git Bash, inklusive `MSYS_NO_PATHCONV`.

10.  [openssl-s_client](https://docs.openssl.org/master/man1/openssl-s_client/): Referens för TLS-testklienten, bland annat `-connect` och `-servername`.

11.  [openssl-x509](https://docs.openssl.org/master/man1/openssl-x509/): Referens för visningsalternativen, bland annat `-noout`, `-subject` och `-enddate`.
