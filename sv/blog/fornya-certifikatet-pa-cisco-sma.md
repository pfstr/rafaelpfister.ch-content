---
title: "Förnya certifikatet på Cisco SMA"
navTitle: "SMA-certifikat"
description: "Certifikat kan endast installeras på Cisco SMA via CLI, och aktuella AsyncOS-versioner validerar hela kedjan vid import: utan en lagrad rot-CA misslyckas den. Artikeln visar vägarna till ett nytt nyckelpar, OpenSSL-metoden i detalj, hur du hanterar RC2-40-CBC-felet i OpenSSL 3 och importerar den interna rot-CA:n till enhetens truststore."
date: "2026-08-04"
kategorie: "Cisco ESA / SMA"
timeToRead: "11 min lästid"
themen:
  - cisco-esa-sma
  - smtp-mailflow
hauptthema: "cisco-esa-sma"
slug: "fornya-certifikatet-pa-cisco-sma"
translationId: "article-69d93a1e5e081848"
aiPrompt: |
  Du bist mein Assistent für die Zertifikatserneuerung auf einer Cisco SMA (Secure Email and Web Manager). Führe mich Schritt für Schritt durch den Ablauf aus diesem Artikel: 1. Wahl des Wegs zum Schlüsselpaar (OpenSSL-CSR in der eigenen Umgebung, PFX von der CA oder Umweg über eine ESA), 2. CN- und SAN-Liste für meine Hostnamen, 3. je nach Weg CSR-Erzeugung mit OpenSSL oder Konvertierung der PFX-Datei nach PEM inklusive Umgang mit dem Fehler RC2-40-CBC, 4. bei interner CA Import der Root-CA in die Custom-Liste der Appliance, 5. Installation über certconfig in der CLI, 6. Kontrolle. Frage mich zuerst nach den Hostnamen meiner Appliances und der Quarantäneseite, ob die ausstellende CA intern oder öffentlich ist und welche OpenSSL-Version ich installiert habe. Passe alle Befehle an meine Dateinamen an und erinnere mich vor dem Abschluss daran, die certconfig-Session nicht mit Ctrl+C zu beenden und die Änderung mit commit zu aktivieren.
translationOf: cisco-sma-zertifikat-erneuern
url: https://rafaelpfister.ch/sv/blog/fornya-certifikatet-pa-cisco-sma
translationSourceHash: 6dc8240e5839f04d73103bb79e45ad14bdc9a7a16e02e2c57f9a4f33be24b53c
translationModel: gpt-5.6-terra
translatedAt: 2026-08-06T06:11:16.314Z
translationReview: automatic
---

# Förnya certifikatet på Cisco SMA

Cisco SMA (Security Management Appliance, numera under namnet Cisco Secure Email and Web Manager) hanterar i många e-postmiljöer central skräppostkarantän och rapportering för Secure Email-gatewayerna. Dess HTTPS-certifikat täcker admin-GUI:t och karantänsidan där slutanvändare kan granska och frisläppa sina kvarhållna e-postmeddelanden. När det löper ut bryts inget e-postflöde. Utgången märks ändå omedelbart: varje öppning av karantänsidan avslutas med en certifikatvarning i webbläsaren, och just de användare som medvetenhetsutbildningar lär att inte klicka vidare vid sådana varningar förväntas då ignorera dem.

Vid en förnyelse i ett kundprojekt fanns det två fallgropar: först svarade OpenSSL 3 på PFX-filen från den interna CA:n med ett kryptiskt fel om `RC2-40-CBC`, därefter vägrade enheten att importera det färdiga certifikatet eftersom den inte kände till den utfärdande rot-CA:n. Båda hindren och deras lösningar beskrivs längre ned.

## Vad SMA gör annorlunda än ESA

På ESA kan hela certifikatlivscykeln hanteras via GUI:t (`Network > Certificates`). SMA kan inte det: servercertifikatet installeras uteslutande via CLI, med kommandot `certconfig` i en SSH-session. SMA:s GUI visar endast certifikat; enbart listorna över betrodda certifikatutfärdare kan hanteras där, mer om det senare.

Därtill finns två ytterligare särdrag:

- Klistra in-dialogen accepterar endast PEM-formatet. En PFX-fil (PKCS#12) måste konverteras före installationen; aktuella AsyncOS-versioner erbjuder dessutom direktimport av PKCS#12, men filen måste först överföras till enheten.
- Äldre AsyncOS-versioner (enligt Cisco-technoten) skapar varken nycklar eller CSR:er själva, så nyckelparet måste skapas utanför enheten; de tre möjliga sätten beskrivs längre ned. Aktuella versioner kan med `certconfig > CERTIFICATE > NEW` skapa ett självsignerat certifikat med CSR direkt på enheten. Det hjälper dock inte för ett gemensamt certifikat över flera enheter, eftersom den privata nyckeln då aldrig lämnar enheten.

Ett enskilt certifikat kan antingen användas för alla tjänster (inkommande och utgående TLS, HTTPS-administrationsåtkomst, LDAPS) eller lagras separat för varje tjänst. Detta styrs i dialogen `certconfig`; kommandots rubrik visar alltid den aktiva tilldelningen (`Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.`). Det finns ingen separat tilldelningsdialog som på ESA, och inget kan ändras via GUI:t. I de flesta miljöer är ett certifikat för allt det pragmatiska valet: namnlistan omfattar ändå enheternas FQDN:er, och separata nyckelpar mångdubblar arbetet vid varje förnyelse.

Att dialogen på en karantänenhet frågar efter inkommande och utgående TLS är vid första anblicken förvirrande, eftersom SMA inte ligger i någon MX-sökväg. Den använder ändå SMTP i båda riktningarna. Inkommande (Receiving) är mottagningssidan: ESA:erna levererar karantänsatta meddelanden via SMTP till SMA, till den centrala skräppostkarantänen på port 6025 och till de centrala policy-, virus- och utbrottskarantänerna på port 7025; de senare anslutningarna är TLS-krypterade som standard, och SMA presenterar då just detta certifikat. Utgående (Delivery) är sändningssidan: om en användare frisläpper ett meddelande från karantänen levererar SMA själv det via sina SMTP-rutter tillbaka till e-postflödet, och enheten skickar även karantänmeddelanden, schemalagda rapporter och varningar som egna e-postmeddelanden. För förnyelsen innebär detta: HTTPS är i praktiken kritiskt, och de båda SMTP-tjänsterna följer helt enkelt med när certifikatet används för alla tjänster.

## Fastställ namn: CN och SAN

Oavsett vägen till nyckelparet fastställs namnlistan först. Common Name ska vara värdnamnet som användarna använder för att öppna karantänsidan. SAN-listan ska dessutom innehålla enheternas FQDN:er, så att direktåtkomst till admin-GUI:t också fungerar utan varning. För en miljö med två enheter ser namnlistan ut så här:

| Fält | Värde |
|---|---|
| CN | `spam-quarantine.example.ch` |
| SAN | `spam-quarantine.example.ch` |
| SAN | `sma01.example.ch` |
| SAN | `sma02.example.ch` |

Två kommentarer: webbläsare utvärderar sedan länge endast SAN-poster, CN ensamt räcker inte. Karantänens värdnamn måste därför också finnas som SAN. Och korta värdnamn utan domändel (exempelvis `SMA01`) utfärdas endast av en intern CA; offentliga CA:er signerar inte interna namn.

## Tre vägar till det nya nyckelparet

För ett certifikat som täcker flera enheter och karantänens värdnamn måste nyckelparet skapas utanför enheten. Tre etablerade sätt finns:

1. Skapa nyckel och CSR med OpenSSL inom den egna miljön. Den privata nyckeln skapas där den behövs och lämnar aldrig miljön. Den rekommenderade metoden, med detaljer i nästa avsnitt.
2. CA:n skapar nyckelparet och levererar en PFX-fil. Det fungerar, men har två nackdelar: nyckeln passerar genom andra händer (lösenordet bör därför skickas via en separat kanal och inte i samma e-postmeddelande som filen), och beroende på CA-verktyg kan en RC2-krypterad PFX returneras, som OpenSSL 3 endast öppnar med extra arbete; mer om detta nedan.
3. Omvägen via en ESA, dokumenterad i Cisco-technoten: skapa där under `Network > Certificates` ett certifikat med SMA:ns CN, hämta CSR:en och låt CA:n signera den, ladda upp det signerade certifikatet till ESA:n igen och exportera allt som PFX. Även här krävs till slut konvertering till PEM.

## Starta OpenSSL under Windows

Alla följande steg körs med OpenSSL på ett system inom miljön, exempelvis en administrationsserver. Light Edition av Windows-byggena från Shining Light Productions räcker, installationsprogrammet är cirka 6 MB stort och kan verifieras mot checksummalistan som slproweb publicerar.

Installationsprogrammet lägger allt under `C:\Program Files\OpenSSL-Win64` och den körbara filen finns i `bin\openssl.exe`. Den läggs inte till i sökvägen: den som skriver `openssl` i en ny kommandotolk får ett felmeddelande. Tre sätt leder till målet:

- Öppna posten `Win64 OpenSSL Command Prompt` i Start-menyn. Den startar `start.bat` från installationskatalogen, ställer in miljön och hälsar med utdata från `openssl version -a`. I detta fönster fungerar `openssl` direkt.
- Ange den fullständiga sökvägen: `"C:\Program Files\OpenSSL-Win64\bin\openssl.exe" version`.
- Lägg permanent till `C:\Program Files\OpenSSL-Win64\bin` i miljövariabeln `Path`; därefter är `openssl` tillgängligt i varje skal.

Den som redan använder Git för Windows klarar sig utan ytterligare installation: det inkluderar eget OpenSSL (`C:\Program Files\Git\mingw64\bin\openssl.exe`), och i Git Bash finns det direkt i sökvägen. Aktuella Git-versioner levererar OpenSSL 3.5 med aktiv Legacy-provider, så `-legacy` från avsnittet om PFX-konvertering fungerar även där. Det kan kontrolleras så här:

```bash
openssl list -providers -provider legacy
```

Git Bash har dock en fallgrop: den tolkar argument som börjar med `/` som sökvägar och skriver om dem. Av `-subj "/C=CH/O=Example AG/CN=..."` blir `C:/Program Files/Git/C=CH/O=Example AG/CN=...`, och OpenSSL avbryts:

```text
req: subject name is expected to be in the format /type0=value0/type1=value1/type2=...
where characters may be escaped by \. This name is not in that format:
'C:/Program Files/Git/C=CH/O=Example AG/CN=spam-quarantine.example.ch'
```

Ett inledande `MSYS_NO_PATHCONV=1` stänger av omskrivningen för det enskilda anropet. Problemet uppstår inte i Kommandotolken, PowerShell eller OpenSSL Command Prompt.

## Skapa nyckel och CSR med OpenSSL

Ett enda anrop skapar nyckel och CSR med hela SAN-listan:

```bash
openssl req -new -newkey rsa:2048 -noenc -keyout spam-quarantine.example.ch.key -out spam-quarantine.example.ch.csr -subj "/C=CH/O=Example AG/CN=spam-quarantine.example.ch" -addext "subjectAltName=DNS:spam-quarantine.example.ch,DNS:sma01.example.ch,DNS:sma02.example.ch"
```

CSR-filen skickas till CA:n och nyckeln stannar på servern. Det signerade certifikatet återkommer med Intermediate-certifikatet, vanligen direkt som PEM. Därmed finns allt redo för installationen och PFX-konvertering bortfaller helt med denna metod.

Nyckelfilen är okrypterad (`-noenc`), eftersom `certconfig` förväntar sig den i just detta skick. Fram till installationen ska den behållas på servern med restriktiva behörigheter och därefter raderas eller flyttas till lösenordshanteraren.

## Konvertera PFX till PEM

Detta och nästa avsnitt gäller väg 2 och 3, som båda avslutas med en PFX-fil. `certconfig` förväntar sig certifikat och privat nyckel som PEM, med okrypterad nyckel. Båda uppgifterna utförs med ett enda OpenSSL-anrop:

```bash
openssl pkcs12 -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

`-noenc` (fram till OpenSSL 3.0 hette alternativet `-nodes`) skriver den privata nyckeln utan lösenfras i utdatafilen. Frågan efter importlösenordet sker utan eko, och inga asterisker visas heller. Den skapade PEM-filen innehåller certifikat, nyckel och medföljande kedjecertifikat i en fil och måste därför skyddas: radera den efter installationen eller flytta den till lösenordshanteraren.

## När OpenSSL 3 vägrar PFX-filen

För äldre PFX-filer avbryts konverteringen i OpenSSL 3.x med detta meddelande:

```text
Error outputting keys and certificates
error:0308010C:digital envelope routines:inner_evp_generic_fetch:unsupported:
crypto/evp/evp_fetch.c: Global default library context, Algorithm (RC2-40-CBC : 0)
```

Orsaken är inte en defekt fil utan ett designbeslut: OpenSSL 3 har flyttat äldre algoritmer som RC2, RC4 och DES till en separat Legacy-provider, som inte laddas som standard. Många PFX-exporter från äldre Windows-system och CA-verktyg krypterar dock certifikatdelen av containern just med RC2-40-CBC. OpenSSL 1.1 öppnade sådana filer utan problem, medan OpenSSL 3 avvisar dem.

Lösningen är ett enda ytterligare alternativ:

```bash
openssl pkcs12 -legacy -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

`-legacy` laddar Legacy-providern för detta anrop, och därefter kör konverteringen igenom. Förutsättningen är en OpenSSL-installation som innehåller Legacy-providern; så är fallet med vanliga Windows-byggen.

Den som vill bli av med felet permanent bör åtgärda källan och låta PFX-filen exporteras med modern kryptering: aktuella exportdialoger och CA-verktyg erbjuder AES-256, vilket helt eliminerar Legacy-omvägen.

Som grafiskt alternativ fungerar XCA (X Certificate and Key Management): läs in PFX-filen via `Importieren > PKCS#12`, exportera sedan certifikatet som PEM på fliken `Zertifikate` och nyckeln separat som okrypterad PEM på fliken `Private Schlüssel`. Båda exporterna behövs, eftersom `certconfig` frågar efter certifikat och nyckel separat. XCA innehåller sitt eget kryptobibliotek och öppnar även containrar med Legacy-algoritmer.

Ett ord om nedladdningskällan: OpenSSL-projektet publicerar inga Windows-binärer själva utan hänvisar till tredjepartsbyggen som Win64 OpenSSL från Shining Light Productions. Nedladdningsportaler med egna installationsprogram är fel adress för ett kryptoverktyg.

## Importera först intern rot-CA till enhetens truststore

Aktuella AsyncOS-versioner validerar hela kedjan när en certifikatprofil skapas. Om certifikatet kommer från en intern CA vars rot enheten inte känner till avbryts importen med detta meddelande:

```text
Cannot import certificate: The certificate authority validation could not be
trusted for certificate because unable to get local issuer certificate
```

Enheten har två listor över betrodda certifikatutfärdare: den medföljande systemlistan och en Custom-lista för egna CA:er. Den interna rot-CA:n hör hemma i Custom-listan, och den måste läggas in innan servercertifikatet installeras. Endast det offentliga CA-certifikatet som PEM-fil behövs (`-----BEGIN CERTIFICATE-----` till `-----END CERTIFICATE-----`), ingen privat nyckel.

Så här överförs rot-CA:n till enheten via webbgränssnittet:

1. Öppna `Network > Certificates`.
2. Klicka på `Edit Settings` i avsnittet `Certificate Authorities`.
3. Välj alternativet `Enable` vid `Custom List`.
4. Ladda upp PEM-filen via `Choose File`.
5. Kör `Submit` och därefter `Commit Changes`.
6. Kontrollera under `Network > Certificates > Manage Trusted Root Certificates` att CA:n visas i listan över anpassade certifikat.

Om det redan finns en Custom-lista ska den först exporteras och den nya CA:n läggas till i det befintliga PEM-paketet: importen ersätter listan, annars försvinner tidigare lagrade CA:er. I en kedja med mellanliggande steg importeras först rot-CA:n och sedan Intermediate-CA:n. AsyncOS kontrollerar vid import bland annat utgångsdatum, dubbletter och den satta flaggan `CA:TRUE` och avvisar en Intermediate så länge den tillhörande roten saknas. Samma import kan även göras via CLI: `certconfig > CERTAUTHORITY > CUSTOM > IMPORT`, därefter `commit`.

Två avgränsningar: för uppdateringar via en TLS-inspekterande proxy har SMA en separat truststore (`updateconfig > TRUSTED_CERTIFICATES > ADD`), och Custom-CA-listan används inte där. Och rot-CA:n på SMA eliminerar inte webbläsarvarningar: klienterna behöver fortfarande roten via sin egen certifikatdistribution, vanligtvis via GPO, och enheten måste leverera servercertifikatet tillsammans med Intermediate-certifikatet.

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

Bakom `CERTIFICATE` finns åtgärderna `IMPORT` (PKCS#12-fil som tidigare laddats upp till enheten), `PASTE` (klistra in certifikat i CLI), `NEW` (skapa självsignerat certifikat med CSR), `EDIT`, `EXPORT`, `DELETE` och `PRINT` (visar tilldelningen till tjänsterna). Den vanliga vägen via SSH är `PASTE`: dialogen frågar efter ett namn för profilen, därefter certifikatet, den privata nyckeln och valfritt Intermediate-certifikat från CA:n, var och en som PEM-block och avslutad med en enskild `.` på en egen rad. En avslutande fråga om FQDN-kontrollen av Common Name kan besvaras med standardvärdet. Intermediate-certifikatet måste ingå i profilen, annars saknar klienterna kedjan och beroende på webbläsare kan varningen kvarstå trots giltigt certifikat.

Äldre AsyncOS-versioner (enligt Cisco-technoten) visar i stället en dialog `SETUP`. Den börjar med frågan `Do you want to use one certificate/key for receiving, delivery, HTTPS management access, and LDAPS?`: ett `y` tilldelar samma par till alla fyra tjänster, medan ett `n` går igenom frågan om certifikat, nyckel och Intermediate en gång per tjänst. Principen för inklistring är identisk.

Två punkter avgör om det lyckas: avsluta inte sessionen med Ctrl+C, eftersom det omedelbart förkastar alla ändringar. Kör också `commit` i slutet; först då blir certifikatet aktivt. Vid två enheter upprepas proceduren på båda, eftersom certifikatkonfigurationen inte synkroniseras mellan SMA:er.

## Kontroll

Det snabbaste testet körs utifrån mot karantänsidan. Slutanvändaråtkomsten till skräppostkarantänen ligger som standard på HTTPS-port 83, om inget annat konfigurerades vid aktiveringen:

```bash
openssl s_client -connect spam-quarantine.example.ch:83 -servername spam-quarantine.example.ch </dev/null 2>/dev/null | openssl x509 -noout -subject -enddate
```

Utdata måste visa det nya Subject och det nya utgångsdatumet. På enheten listar `certconfig` med åtgärden `PRINT` de aktiva certifikaten, och webbläsarkontrollen mot admin-GUI:t och karantänsidan bekräftar att kedjan är korrekt uppbyggd.

## Källor

1.  [How to Generate and Install a Certificate on an SMA](https://www.cisco.com/c/en/us/support/docs/security/content-security-management-appliance/118460-technote-sma-00.html): Cisco-technote med certconfig-processen för äldre AsyncOS-versioner, PEM-kravet och sätten att skapa certifikat via ESA eller OpenSSL.

2.  [Common Administrative Tasks, AsyncOS 16.0 för Cisco Secure Email and Web Manager](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_01011.html): Kapitel i administrationsguiden om hantering av Certificate Authority-listor (system- och Custom-lista), inklusive kontroller vid CA-import.

3.  [Comprehensive Spam Quarantine Setup Guide on ESA and SMA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118692-configure-esa-00.html): Cisco-guide för skräppostkarantän, inklusive slutanvändaråtkomst via HTTPS på port 83.

4.  [openssl-req](https://docs.openssl.org/master/man1/openssl-req/): Referens för att skapa nyckel och CSR, inklusive `-addext` för SAN-listan.

5.  [openssl-pkcs12](https://docs.openssl.org/master/man1/openssl-pkcs12/): Referens för konverteringsalternativen, bland annat `-noenc` (tidigare `-nodes`) och `-legacy`.

6.  [OpenSSL 3.0 Migration Guide](https://docs.openssl.org/3.0/man7/migration_guide/): Bakgrund om flytten av äldre algoritmer till Legacy-providern.

7.  [XCA: X Certificate and Key Management](https://hohnstaedt.de/xca/): Verktyg med öppen källkod för import och export av PKCS#12- och PEM-strukturer.

8.  [Win32/Win64 OpenSSL](https://slproweb.com/products/Win32OpenSSL.html): Windows-byggen från Shining Light Productions, som OpenSSL-projektet hänvisar till, inklusive publicerad checksummalista.

9.  [MSYS2: Filesystem Paths](https://www.msys2.org/docs/filesystem-paths/): Beskrivning av den automatiska sökvägsomskrivningen som i Git Bash ändrar argumentet `-subj`, inklusive `MSYS_NO_PATHCONV`.
