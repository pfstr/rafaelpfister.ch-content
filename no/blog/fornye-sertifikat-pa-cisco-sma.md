---
title: "Fornye sertifikat på Cisco SMA"
navTitle: "SMA-sertifikat"
description: "Sertifikater kan bare installeres på Cisco SMA via CLI, og aktuelle AsyncOS-versjoner validerer hele kjeden ved import: Uten en lagret rot-CA mislykkes den. Artikkelen viser veiene til et nytt nøkkelpar, OpenSSL-metoden i detalj, håndtering av RC2-40-CBC-feilen i OpenSSL 3 og import av den interne rot-CA-en til apparatets klareringslager."
date: "2026-08-04"
kategorie: "Cisco ESA / SMA"
timeToRead: "11 min lesetid"
themen:
  - cisco-esa-sma
  - smtp-mailflow
hauptthema: "cisco-esa-sma"
slug: "fornye-sertifikat-pa-cisco-sma"
translationId: "article-69d93a1e5e081848"
aiPrompt: |
  Du bist mein Assistent für die Zertifikatserneuerung auf einer Cisco SMA (Secure Email and Web Manager). Führe mich Schritt für Schritt durch den Ablauf aus diesem Artikel: 1. Wahl des Wegs zum Schlüsselpaar (OpenSSL-CSR in der eigenen Umgebung, PFX von der CA oder Umweg über eine ESA), 2. CN- und SAN-Liste für meine Hostnamen, 3. je nach Weg CSR-Erzeugung mit OpenSSL oder Konvertierung der PFX-Datei nach PEM inklusive Umgang mit dem Fehler RC2-40-CBC, 4. bei interner CA Import der Root-CA in die Custom-Liste der Appliance, 5. Installation über certconfig in der CLI, 6. Kontrolle. Frage mich zuerst nach den Hostnamen meiner Appliances und der Quarantäneseite, ob die ausstellende CA intern oder öffentlich ist und welche OpenSSL-Version ich installiert habe. Passe alle Befehle an meine Dateinamen an und erinnere mich vor dem Abschluss daran, die certconfig-Session nicht mit Ctrl+C zu beenden und die Änderung mit commit zu aktivieren.
translationOf: cisco-sma-zertifikat-erneuern
translationSourceHash: c99ce64a5e63875b84c7b6f14a7f2fb7e51290fedbdc93d99201cdc97a743508
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T09:16:39.896Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/fornye-sertifikat-pa-cisco-sma
---

# Fornye sertifikat på Cisco SMA

Cisco SMA (Security Management Appliance, nå markedsført under navnet Cisco Secure Email and Web Manager) håndterer i mange e-postmiljøer sentral spam-karantene og rapportering for Secure Email-gatewayene. HTTPS-sertifikatet dekker administratorgrensesnittet og karantenesiden, der sluttbrukere kan se gjennom og frigi tilbakeholdte e-poster. Når det utløper, stopper ingen e-postflyt. Utløpet blir likevel umiddelbart synlig: Hvert besøk på karantenesiden ender med en sertifikatadvarsel i nettleseren, og nettopp brukerne som bevisstgjøringsopplæringen lærer å ikke klikke videre ved slike advarsler, skal da ignorere dem.

Ved en fornyelse i et kundeprosjekt oppstod det to problemer: Først svarte OpenSSL 3 med en kryptisk feil på PFX-filen fra den interne CA-en om `RC2-40-CBC`, deretter nektet apparatet å importere det ferdige sertifikatet fordi utstedende rot-CA ikke var kjent. Begge hindringene og løsningene følger nedenfor.

## Hva SMA gjør annerledes enn ESA

På ESA kan hele sertifikatets livssyklus håndteres via GUI-et (`Network > Certificates`). SMA kan ikke dette: Serversertifikatet installeres utelukkende via CLI, med kommandoen `certconfig` i en SSH-økt. SMA-GUI-et viser bare sertifikater; bare listene over klarerte sertifiseringsinstanser kan vedlikeholdes der, mer om det senere.

I tillegg er det to andre særtrekk:

- Innlimingsdialogen godtar bare PEM-formatet. En PFX-fil (PKCS#12) må konverteres før installasjon; aktuelle AsyncOS-versjoner tilbyr også direkte PKCS#12-import, men filen må først overføres til apparatet.
- Eldre AsyncOS-versjoner (statusen i Cisco-technoten) oppretter verken nøkler eller CSR-er selv, så nøkkelparet må opprettes utenfor; de tre aktuelle metodene følger nedenfor. Aktuelle versjoner kan med `certconfig > CERTIFICATE > NEW` opprette et selvsignert sertifikat med CSR direkte på apparatet. Det hjelper imidlertid ikke for et felles sertifikat på tvers av flere apparater, siden den private nøkkelen aldri forlater apparatet.

Ett enkelt sertifikat kan enten betjene alle tjenester (inngående og utgående TLS, HTTPS-administrasjonstilgang, LDAPS) eller lagres separat per tjeneste. Dette styres i dialogen `certconfig`; kommandotittelen viser til enhver tid den aktive tilordningen (`Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.`). Det finnes ingen separat tilordningsdialog som på ESA, og dette kan ikke endres via GUI-et. I de fleste miljøer er ett sertifikat for alt det pragmatiske valget: Navnelisten dekker uansett FQDN-ene til apparatene, og separate nøkkelpar mangedobler arbeidet ved hver fornyelse.

At dialogen på et karanteneapparat spør om innkommende og utgående TLS, virker først forvirrende, siden SMA ikke står i noen MX-bane. Den kommuniserer likevel SMTP i begge retninger. Inbound (Receiving) er mottakssiden: ESA-ene leverer karantenebelagte meldinger via SMTP til SMA-en, til sentral spam-karantene på port 6025 og til sentrale policy-, virus- og outbreak-karantener på port 7025; sistnevnte forbindelser er TLS-kryptert som standard, og SMA-en presenterer nettopp dette sertifikatet. Outbound (Delivery) er sendesiden: Når en bruker frigir en melding fra karantenen, leverer SMA-en den selv tilbake til e-postflyten via sine SMTP-ruter, og apparatet sender også karantenevarsler, planlagte rapporter og varsler som egne e-poster. For fornyelsen betyr dette: HTTPS er kritisk i praksis, mens de to SMTP-tjenestene bare følger med når sertifikatet brukes for alle tjenester.

## Fastsette navn: CN og SAN

Uavhengig av hvordan nøkkelparet opprettes, må navnelisten bestemmes først. Common Name skal være vertsnavnet brukerne benytter for å åpne karantenesiden. SAN-listen skal i tillegg inneholde FQDN-ene til apparatene, slik at direkte tilgang til administratorgrensesnittet også fungerer uten advarsel. For et miljø med to apparater ser navnelisten slik ut:

| Felt | Verdi |
|---|---|
| CN | `spam-quarantine.example.ch` |
| SAN | `spam-quarantine.example.ch` |
| SAN | `sma01.example.ch` |
| SAN | `sma02.example.ch` |

To merknader: Nettlesere har lenge kun evaluert SAN-oppføringene; CN alene er ikke tilstrekkelig. Karantenevertsnavnet må derfor også stå som SAN. Korte vertsnavn uten domeneandel (for eksempel `SMA01`) utstedes dessuten bare av en intern CA; offentlige CA-er signerer ikke interne navn.

## Tre måter å få et nytt nøkkelpar på

For et sertifikat som dekker flere apparater og karantenevertsnavnet, må nøkkelparet opprettes utenfor apparatet. Tre metoder har etablert seg:

1. Opprett nøkkel og CSR med OpenSSL i eget miljø. Den private nøkkelen oppstår der den trengs og forlater aldri miljøet. Dette er den anbefalte metoden; detaljer i neste avsnitt.
2. CA-en oppretter nøkkelparet og leverer en PFX-fil. Det fungerer, men har to ulemper: Nøkkelen går gjennom andres hender (passordet bør derfor sendes i en separat kanal og ikke i samme e-post som filen), og avhengig av CA-verktøyet kan man få tilbake en RC2-kryptert PFX som OpenSSL 3 bare åpner med ekstra innsats; mer om dette nedenfor.
3. Omveien via en ESA, dokumentert i Cisco-technoten: Opprett et sertifikat der under `Network > Certificates` med SMA-ens CN, last ned CSR-en og få den signert av CA-en, last det signerte sertifikatet opp til ESA-en igjen og eksporter det hele som PFX. Også her må det til slutt konverteres til PEM.

## De viktigste alternativene i openssl

For orientering følger underkommandoene og alternativene i `openssl` som brukes i denne artikkelen, oversatt fritt fra OpenSSL-dokumentasjonen:

<details class="options-details">
<summary>Oversikt over alternativer</summary>

| Alternativ | Betydning |
|---|---|
| `req` | Underkommando for sertifikatforespørsler (CSR): opprette, vise, kontrollere |
| `-new` | Oppretter en ny forespørsel |
| `-newkey rsa:2048` | Oppretter et nytt RSA-nøkkelpar på 2048 bit |
| `-noenc` | Skriver den private nøkkelen ukryptert (frem til OpenSSL 3.0: `-nodes`) |
| `-keyout datei` | Målfil for den private nøkkelen |
| `-out datei` | Målfil for utdata, her CSR eller PEM |
| `-subj text` | Subject for forespørselen i formatet `/C=…/O=…/CN=…` |
| `-addext text` | Legger til en utvidelse i forespørselen, her SAN-listen |
| `pkcs12` | Underkommando for PKCS#12-containere (PFX): opprette og pakke ut |
| `-in datei` | Inndatafil |
| `-legacy` | Laster også Legacy-provider for eldre algoritmer som RC2 |
| `list` | Underkommando for å vise installasjonens funksjoner |
| `-providers` | Viser de innlastede providerne |
| `-provider name` | Laster den angitte provideren i tillegg for dette kallet |
| `s_client` | Underkommando: TLS-testklient for forbindelser til en server |
| `-connect host:port` | Målvert og port for TLS-forbindelsen |
| `-servername name` | Setter Server Name Indication (SNI) i TLS-håndtrykket |
| `x509` | Underkommando for visning og behandling av sertifikater |
| `-noout` | Undertrykker utdata av det kodede sertifikatet |
| `-subject` | Viser sertifikatets Subject |
| `-enddate` | Viser utløpsdatoen (notAfter) |

</details>

Den fullstendige referansen finnes i OpenSSL-dokumentasjonen som en egen manpage for hver underkommando: `openssl-req(1)`, `openssl-pkcs12(1)`, `openssl-s_client(1)` og `openssl-x509(1)`.

## Starte OpenSSL på Windows

Alle følgende trinn kjøres med OpenSSL, på et system i miljøet, for eksempel en administrasjonsserver. Light Edition av Windows-byggene fra Shining Light Productions er tilstrekkelig; installasjonsprogrammet er rundt 6 MB stort og kan verifiseres mot kontrollsummelisten publisert av slproweb.

Installasjonsprogrammet legger alt under `C:\Program Files\OpenSSL-Win64` og den kjørbare filen ligger i `bin\openssl.exe`. Den legges ikke til i søkebanen: Den som skriver `openssl` i en ny ledetekst, får en feilmelding. Det finnes tre muligheter:

- Åpne oppføringen `Win64 OpenSSL Command Prompt` fra Start-menyen. Den starter `start.bat` fra installasjonskatalogen, setter miljøet og viser utdata fra `openssl version -a`. I dette vinduet fungerer `openssl` direkte.
- Oppgi hele banen: `"C:\Program Files\OpenSSL-Win64\bin\openssl.exe" version`.
- Legg `C:\Program Files\OpenSSL-Win64\bin` permanent til miljøvariabelen `Path`; deretter er `openssl` tilgjengelig i alle skall.

Den som allerede bruker Git for Windows, trenger ingen ekstra installasjon: Det leveres med sin egen OpenSSL (`C:\Program Files\Git\mingw64\bin\openssl.exe`), og i Git Bash ligger den umiddelbart i søkebanen. Aktuelle Git-versjoner leverer OpenSSL 3.5 med aktiv Legacy-provider, slik at `-legacy` fra avsnittet om PFX-konvertering også fungerer der. Dette kan kontrolleres slik:

```bash
openssl list -providers -provider legacy
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `list` | Viser funksjonene til OpenSSL-installasjonen |
| `-providers` | Viser innlastede providere med navn, versjon og status |
| `-provider legacy` | Laster i tillegg provideren `legacy` for dette kallet; vises den i listen, er den tilgjengelig |

</details>

Git Bash har imidlertid en særegenhet: Den tolker argumenter som begynner med `/` som stier og skriver dem om. `-subj "/C=CH/O=Example AG/CN=..."` blir til `C:/Program Files/Git/C=CH/O=Example AG/CN=...`, og OpenSSL avbryter:

```text
req: subject name is expected to be in the format /type0=value0/type1=value1/type2=...
where characters may be escaped by \. This name is not in that format:
'C:/Program Files/Git/C=CH/O=Example AG/CN=spam-quarantine.example.ch'
```

Et innledende `MSYS_NO_PATHCONV=1` deaktiverer omskrivingen for det enkelte kallet. Problemet oppstår ikke i Ledetekst, PowerShell og OpenSSL Command Prompt.

## Opprette nøkkel og CSR med OpenSSL

Ett enkelt kall oppretter nøkkel og CSR med hele SAN-listen:

```bash
openssl req -new -newkey rsa:2048 -noenc \
  -keyout spam-quarantine.example.ch.key \
  -out spam-quarantine.example.ch.csr \
  -subj "/C=CH/O=Example AG/CN=spam-quarantine.example.ch" \
  -addext "subjectAltName=DNS:spam-quarantine.example.ch,DNS:sma01.example.ch,DNS:sma02.example.ch"
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-new` | Oppretter en ny sertifikatforespørsel (CSR) |
| `-newkey rsa:2048` | Oppretter et nytt RSA-nøkkelpar på 2048 bit |
| `-noenc` | Skriver den private nøkkelen ukryptert til filen |
| `-keyout …` | Målfil for den private nøkkelen |
| `-out …` | Målfil for CSR-en |
| `-subj …` | Subject med land, organisasjon og Common Name |
| `-addext …` | Legger SAN-utvidelsen med alle DNS-navnene til forespørselen |

</details>

CSR-filen sendes til CA-en, mens nøkkelen blir værende på serveren. Tilbake kommer det signerte sertifikatet med intermediate, vanligvis direkte som PEM. Da er alt klart for installasjon, og PFX-konvertering faller helt bort med denne metoden.

Nøkkelfilen er ukryptert (`-noenc`), fordi `certconfig` forventer den nettopp slik. Frem til installasjonen oppbevares den med restriktive tillatelser på serveren, og deretter slettes den eller flyttes til passordbehandleren.

## Konvertere PFX til PEM

Dette og neste avsnitt gjelder metodene 2 og 3, som ender med en PFX-fil. `certconfig` forventer sertifikat og privat nøkkel som PEM, med nøkkelen ukryptert. Begge deler utføres med ett enkelt OpenSSL-kall:

```bash
openssl pkcs12 -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `pkcs12` | Underkommando for å opprette og pakke ut PKCS#12-containere |
| `-in …` | PFX-inndatafilen |
| `-out …` | PEM-utdatafilen med sertifikat, nøkkel og kjedesertifikater |
| `-noenc` | Skriver den private nøkkelen uten passfrase (frem til OpenSSL 3.0 het alternativet `-nodes`) |

</details>

Importpassordet etterspørres uten ekko, og det vises heller ingen stjerner. Den opprettede PEM-filen inneholder sertifikat, nøkkel og medfølgende kjedesertifikater i én fil, og må derfor beskyttes: Slett den etter installasjonen eller flytt den til passordbehandleren.

## Når OpenSSL 3 avviser PFX-filen

For eldre PFX-filer avbrytes konverteringen under OpenSSL 3.x med denne meldingen:

```text
Error outputting keys and certificates
error:0308010C:digital envelope routines:inner_evp_generic_fetch:unsupported:
crypto/evp/evp_fetch.c: Global default library context, Algorithm (RC2-40-CBC : 0)
```

Årsaken er ikke en ødelagt fil, men en designbeslutning: OpenSSL 3 har flyttet eldre algoritmer som RC2, RC4 og DES til en separat Legacy-provider som ikke lastes som standard. Mange PFX-eksporter fra eldre Windows-systemer og CA-verktøy krypterer imidlertid sertifikatdelen av containeren nettopp med RC2-40-CBC. OpenSSL 1.1 åpnet slike filer uten problemer, mens OpenSSL 3 avviser dem.

Løsningen er ett ekstra alternativ:

```bash
openssl pkcs12 -legacy -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-legacy` | Laster Legacy-provideren for dette kallet; da blir eldre algoritmer som RC2-40-CBC tilgjengelige igjen, og konverteringen fullføres |

</details>

Forutsetningen er en OpenSSL-installasjon som inkluderer Legacy-provideren; dette er tilfellet for de vanlige Windows-byggene.

Den som vil bli kvitt feilen permanent, kan gjøre noe ved kilden og få PFX-filen eksportert med moderne kryptering: Aktuelle eksportdialoger og CA-verktøy tilbyr AES-256, og da faller Legacy-omveien helt bort.

Som grafisk alternativ fungerer XCA (X Certificate and Key Management): Importer PFX-filen via `Importieren > PKCS#12`, eksporter deretter sertifikatet som PEM fra fanen `Zertifikate`, og nøkkelen separat som ukryptert PEM fra fanen `Private Schlüssel`. Begge eksportene trengs; `certconfig` spør etter sertifikat og nøkkel hver for seg. XCA leveres med sitt eget kryptobibliotek og åpner også containere med eldre algoritmer.

Et ord om nedlastingskilden: OpenSSL-prosjektet publiserer ikke selv Windows-binærfiler, men viser til tredjepartsbygg som Win64 OpenSSL fra Shining Light Productions. Nedlastingsportaler med egne installasjonsprogrammer er feil sted for et kryptoverktøy.

## Importer intern rot-CA i apparatets klareringslager først

Aktuelle AsyncOS-versjoner validerer hele kjeden når en sertifikatprofil opprettes. Hvis sertifikatet kommer fra en intern CA hvis rot apparatet ikke kjenner, avbrytes importen med denne meldingen:

```text
Cannot import certificate: The certificate authority validation could not be
trusted for certificate because unable to get local issuer certificate
```

Apparatet har to lister over klarerte sertifiseringsinstanser: den medfølgende systemlisten og en Custom-liste for egne CA-er. Den interne rot-CA-en hører hjemme i Custom-listen, og den må legges inn før serversertifikatet installeres. Bare det offentlige CA-sertifikatet som PEM-fil trengs (`-----BEGIN CERTIFICATE-----` til `-----END CERTIFICATE-----`), ingen privat nøkkel.

Slik overføres rot-CA-en til apparatet via webgrensesnittet:

1. Åpne `Network > Certificates`.
2. Klikk `Edit Settings` i seksjonen `Certificate Authorities`.
3. Velg alternativet `Enable` ved `Custom List`.
4. Last opp PEM-filen via `Choose File`.
5. Utfør `Submit` og deretter `Commit Changes`.
6. Kontroller under `Network > Certificates > Manage Trusted Root Certificates` at CA-en vises i listen over egendefinerte sertifikater.

Hvis det allerede finnes en Custom-liste, eksporter den først og legg den nye CA-en til det eksisterende PEM-bundlet: Importen erstatter listen, ellers forsvinner tidligere lagrede CA-er. I en kjede med mellomledd må rot-CA-en importeres først og deretter intermediate-CA-en. AsyncOS kontrollerer blant annet utløpsdato, duplikater og satt `CA:TRUE`-flagg under importen, og avviser en intermediate så lenge den tilhørende roten mangler. Den samme importen kan også utføres via CLI: `certconfig > CERTAUTHORITY > CUSTOM > IMPORT`, deretter `commit`.

To avgrensninger: For oppdateringer via en TLS-inspeksjonsproxy har SMA-en et separat klareringslager (`updateconfig > TRUSTED_CERTIFICATES > ADD`), og Custom-CA-listen gjelder ikke der. Og rot-CA-en på SMA-en fjerner ikke nettleseradvarsler: Klientene trenger fortsatt roten via sin egen sertifikatdistribusjon, vanligvis gjennom GPO, og apparatet må levere serversertifikatet med intermediate.

## Installasjon med certconfig

Logg på SMA-en via SSH og start `certconfig`. I aktuelle AsyncOS-versjoner arbeider dialogen med sertifikatprofiler:

```text
sma01.example.ch> certconfig

Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.

Choose the operation you want to perform:
- CERTIFICATE - Import, Create a request, Edit or Remove Certificate Profiles
- CERTAUTHORITY - Manage System and Customized Authorities
- CRL - Manage Certificate Revocation Lists
[]> certificate
```

Bak `CERTIFICATE` ligger operasjonene `IMPORT` (PKCS#12-fil som på forhånd er lastet opp til apparatet), `PASTE` (lim inn sertifikatet i CLI-et), `NEW` (opprett selvsignert sertifikat med CSR), `EDIT`, `EXPORT`, `DELETE` og `PRINT` (viser tilordningen til tjenestene). Den vanlige metoden via SSH er `PASTE`: Dialogen spør etter et navn for profilen, deretter sertifikatet, den private nøkkelen og eventuelt CA-ens intermediate-sertifikat, hver som PEM-blokk, avsluttet med en enkelt `.` på egen linje. Et avsluttende spørsmål om FQDN-kontroll av Common Name kan besvares med standardverdien. Intermediate hører hjemme i profilen; ellers mangler klientene kjeden, og avhengig av nettleser kan advarselen bli værende til tross for et gyldig sertifikat.

Eldre AsyncOS-versjoner (statusen i Cisco-technoten) viser i stedet en dialog `SETUP`. Den begynner med spørsmålet `Do you want to use one certificate/key for receiving, delivery, HTTPS management access, and LDAPS?`: Et `y` tilordner samme par til alle fire tjenestene, mens et `n` går gjennom spørsmålene om sertifikat, nøkkel og intermediate én gang per tjeneste. Prinsippet for innliming er identisk.

To punkter avgjør om det lykkes: Ikke avslutt økten med Ctrl+C, for det forkaster straks alle endringer. Kjør også `commit` til slutt; først da blir sertifikatet aktivt. Med to apparater gjentas prosessen på begge, siden sertifikatkonfigurasjonen ikke synkroniseres mellom SMA-er.

## Kontroll

Den raskeste testen kjøres utenfra mot karantenesiden. Sluttbrukertilgangen til spam-karantenen ligger som standard på HTTPS-port 83, dersom ikke noe annet ble konfigurert ved aktivering:

```bash
openssl s_client -connect spam-quarantine.example.ch:83 \
  -servername spam-quarantine.example.ch </dev/null 2>/dev/null |
  openssl x509 -noout -subject -enddate
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `s_client` | TLS-testklient: oppretter forbindelsen og videresender det presenterte sertifikatet |
| `-connect …:83` | Målvert og port, her HTTPS-porten til spam-karantenen |
| `-servername …` | Setter Server Name Indication (SNI), slik at serveren leverer riktig sertifikat |
| `x509` | Behandler det videresendte sertifikatet |
| `-noout` | Undertrykker utdata av det kodede sertifikatet |
| `-subject` | Viser sertifikatets Subject |
| `-enddate` | Viser utløpsdatoen (notAfter) |

</details>

Utdataene må vise det nye Subject og den nye utløpsdatoen. På apparatet viser `certconfig` med operasjonen `PRINT` de aktive sertifikatene, og kontroll i nettleseren mot administratorgrensesnittet og karantenesiden bekrefter at kjeden er riktig oppbygd.

## Kilder

1.  [How to Generate and Install a Certificate on an SMA](https://www.cisco.com/c/en/us/support/docs/security/content-security-management-appliance/118460-technote-sma-00.html): Cisco-technote med certconfig-prosessen i eldre AsyncOS-versjoner, PEM-kravet og metodene for sertifikatopprettelse via ESA eller OpenSSL.

2.  [Common Administrative Tasks, AsyncOS 16.0 for Cisco Secure Email and Web Manager](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_01011.html): Kapittel i administratorveiledningen om administrasjon av Certificate Authority-listene (system- og Custom-liste), inkludert kontrollene ved CA-import.

3.  [Comprehensive Spam Quarantine Setup Guide on ESA and SMA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118692-configure-esa-00.html): Cisco-veiledning for spam-karantene, inkludert sluttbrukertilgang via HTTPS på port 83.

4.  [openssl-req](https://docs.openssl.org/master/man1/openssl-req/): Referanse for opprettelse av nøkkel og CSR, inkludert `-addext` for SAN-listen.

5.  [openssl-pkcs12](https://docs.openssl.org/master/man1/openssl-pkcs12/): Referanse for konverteringsalternativene, blant annet `-noenc` (tidligere `-nodes`) og `-legacy`.

6.  [OpenSSL 3.0 Migration Guide](https://docs.openssl.org/3.0/man7/migration_guide/): Bakgrunn om flyttingen av eldre algoritmer til Legacy-provideren.

7.  [XCA: X Certificate and Key Management](https://hohnstaedt.de/xca/): Åpen kildekode-verktøy for import og eksport av PKCS#12- og PEM-strukturer.

8.  [Win32/Win64 OpenSSL](https://slproweb.com/products/Win32OpenSSL.html): Windows-bygg fra Shining Light Productions, som OpenSSL-prosjektet viser til, inkludert publisert kontrollsummeliste.

9.  [MSYS2: Filesystem Paths](https://www.msys2.org/docs/filesystem-paths/): Beskrivelse av den automatiske stiomskrivingen som endrer argumentet `-subj` i Git Bash, inkludert `MSYS_NO_PATHCONV`.

10.  [openssl-s_client](https://docs.openssl.org/master/man1/openssl-s_client/): Referanse for TLS-testklienten, blant annet `-connect` og `-servername`.

11.  [openssl-x509](https://docs.openssl.org/master/man1/openssl-x509/): Referanse for visningsalternativene, blant annet `-noout`, `-subject` og `-enddate`.
