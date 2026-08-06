---
title: "Fornye sertifikat på Cisco SMA"
navTitle: "SMA-sertifikat"
description: "Sertifikater kan bare installeres på Cisco SMA via CLI, og aktuelle AsyncOS-versjoner validerer hele kjeden ved import: Uten en lagret rot-CA mislykkes importen. Artikkelen viser veiene til et nytt nøkkelpar, OpenSSL-metoden i detalj, hvordan du håndterer RC2-40-CBC-feilen i OpenSSL 3 og import av den interne rot-CA-en i appliance-ens tillitslager."
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
url: https://rafaelpfister.ch/no/blog/fornye-sertifikat-pa-cisco-sma
translationSourceHash: 6dc8240e5839f04d73103bb79e45ad14bdc9a7a16e02e2c57f9a4f33be24b53c
translationModel: gpt-5.6-terra
translatedAt: 2026-08-06T06:12:03.861Z
translationReview: automatic
---

# Fornye sertifikat på Cisco SMA

Cisco SMA (Security Management Appliance, nå markedsført under navnet Cisco Secure Email and Web Manager) håndterer i mange e-postmiljøer den sentrale spam-karantenen og rapporteringen for Secure Email-gatewayene. HTTPS-sertifikatet dekker admin-GUI-en og karantenesiden, der sluttbrukere kan se gjennom og frigi tilbakeholdte e-poster. Når det utløper, stopper ikke e-postflyten. Likevel blir utløpet straks synlig: Hvert besøk på karantenesiden ender med en sertifikatadvarsel i nettleseren, og nettopp brukerne som opplæring i sikkerhetsbevissthet lærer å ikke klikke videre ved slike advarsler, skal da ignorere dem.

Ved en fornyelse i et kundeprosjekt oppstod det to hindringer: Først avviste OpenSSL 3 PFX-filen fra den interne CA-en med en kryptisk feil om `RC2-40-CBC`, deretter nektet appliance-en å importere det ferdige sertifikatet fordi den ikke kjente den utstedende rot-CA-en. Begge hindringene og løsningene følger lenger ned.

## Hva SMA gjør annerledes enn ESA

På ESA kan hele sertifikatlivssyklusen håndteres via GUI-en (`Network > Certificates`). SMA kan ikke det: Serversertifikatet installeres utelukkende via CLI, med kommandoen `certconfig` i en SSH-økt. SMA-GUI-en viser bare sertifikater; bare listene over klarerte sertifiseringsinstanser kan vedlikeholdes der, mer om det senere.

I tillegg kommer to andre særtrekk:

- Innlimingsdialogen godtar bare PEM-formatet. En PFX-fil (PKCS#12) må konverteres før installasjon; aktuelle AsyncOS-versjoner tilbyr også direkte PKCS#12-import, men filen må først overføres til appliance-en.
- Eldre AsyncOS-versjoner (slik Cisco-teknote-en beskriver dem) oppretter verken nøkler eller CSR-er selv, så nøkkelparet må opprettes utenfor; de tre aktuelle metodene følger lenger ned. Aktuelle versjoner kan med `certconfig > CERTIFICATE > NEW` opprette et selvsignert sertifikat med CSR direkte på appliance-en. Dette hjelper imidlertid ikke for et felles sertifikat på tvers av flere applikasjoner, fordi den private nøkkelen da aldri forlater appliance-en.

Ett sertifikat kan enten brukes for alle tjenester (inngående og utgående TLS, HTTPS-administrasjonstilgang, LDAPS) eller lagres separat for hver tjeneste. Dette styres i dialogen `certconfig`; kommandotittelen viser alltid den aktive tildelingen (`Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.`). Det finnes ingen separat tildelingsdialog som på ESA, og dette kan ikke endres via GUI-en. I de fleste miljøer er ett sertifikat for alt det pragmatiske valget: Navnelisten dekker uansett FQDN-ene til appliance-ene, og separate nøkkelpar mangedobler arbeidet ved hver fornyelse.

At dialogen på en karantene-appliance spør om innkommende og utgående TLS, virker ved første øyekast forvirrende, siden SMA ikke står i noen MX-bane. Den kommuniserer likevel SMTP i begge retninger. Inngående (Receiving) er mottakssiden: ESA-ene leverer karantenelagte meldinger via SMTP til SMA, til den sentrale spam-karantenen på port 6025 og til de sentrale policy-, virus- og outbreak-karantenene på port 7025; de sistnevnte forbindelsene er TLS-kryptert som standard, og SMA presenterer nettopp dette sertifikatet der. Utgående (Delivery) er sendesiden: Når en bruker frigir en melding fra karantenen, leverer SMA den selv tilbake til e-postflyten via SMTP-rutene sine, og appliance-en sender også karantenevarsler, planlagte rapporter og varsler som egne e-poster. For fornyelsen betyr dette: HTTPS er i praksis kritisk, mens de to SMTP-tjenestene bare følger med når sertifikatet brukes for alle tjenester.

## Fastsett navn: CN og SAN

Uavhengig av veien til nøkkelparet må navnelisten først fastsettes. Common Name skal være vertsnavnet brukerne bruker for å åpne karantenesiden. SAN-listen skal i tillegg inneholde FQDN-ene til appliance-ene, slik at også direkte tilgang til admin-GUI-en fungerer uten advarsel. For et miljø med to appliance-er ser navnelisten slik ut:

| Felt | Verdi |
|---|---|
| CN | `spam-quarantine.example.ch` |
| SAN | `spam-quarantine.example.ch` |
| SAN | `sma01.example.ch` |
| SAN | `sma02.example.ch` |

To merknader: Nettlesere har lenge kun vurdert SAN-oppføringene, CN alene er ikke nok. Karantenevertsnavnet må derfor også stå som SAN. Og korte vertsnavn uten domeneandel (for eksempel `SMA01`) utstedes bare av en intern CA; offentlige CA-er signerer ikke interne navn.

## Tre veier til det nye nøkkelparet

For et sertifikat som dekker flere appliance-er og karantenevertsnavnet, må nøkkelparet opprettes utenfor appliance-en. Tre metoder har etablert seg:

1. Opprett nøkkel og CSR med OpenSSL i eget miljø. Den private nøkkelen opprettes der den skal brukes og forlater aldri miljøet. Den anbefalte metoden, med detaljer i neste avsnitt.
2. CA-en oppretter nøkkelparet og leverer en PFX-fil. Dette fungerer, men har to ulemper: Nøkkelen går gjennom fremmede hender (passordet bør derfor sendes i en separat kanal og ikke i samme e-post som filen), og avhengig av CA-verktøyet kan du få tilbake en RC2-kryptert PFX som OpenSSL 3 bare åpner med ekstra innsats; mer om dette nedenfor.
3. Omveien via en ESA, dokumentert i Cisco-teknote-en: Opprett et sertifikat med SMA-ens CN under `Network > Certificates`, last ned CSR-en og få den signert av CA-en, last det signerte sertifikatet opp igjen til ESA-en og eksporter alt som PFX. Også her må det til slutt konverteres til PEM.

## Starte OpenSSL på Windows

Alle følgende trinn utføres med OpenSSL på et system i miljøet, for eksempel en administrasjonsserver. Light Edition av Windows-byggene fra Shining Light Productions er tilstrekkelig; installasjonsfilen er på omtrent 6 MB og kan verifiseres mot sjekksumlisten publisert av slproweb.

Installereren legger alt under `C:\Program Files\OpenSSL-Win64` , og den kjørbare filen ligger i `bin\openssl.exe`. Den legger seg ikke til i søkestien: Den som skriver `openssl` i en ny ledetekst, får en feilmelding. Tre veier fører til målet:

- Åpne oppføringen `Win64 OpenSSL Command Prompt` fra Start-menyen. Den starter `start.bat` fra installasjonskatalogen, setter opp miljøet og viser utdata fra `openssl version -a`. I dette vinduet fungerer `openssl` direkte.
- Oppgi hele banen: `"C:\Program Files\OpenSSL-Win64\bin\openssl.exe" version`.
- Legg `C:\Program Files\OpenSSL-Win64\bin` permanent til miljøvariabelen `Path`; deretter er `openssl` tilgjengelig i alle skall.

De som allerede bruker Git for Windows, klarer seg uten ekstra installasjon: Det inkluderer sin egen OpenSSL (`C:\Program Files\Git\mingw64\bin\openssl.exe`), og i Git Bash ligger den umiddelbart i søkestien. Aktuelle Git-versjoner leverer OpenSSL 3.5 med aktiv Legacy Provider, så `-legacy` fra avsnittet om PFX-konvertering fungerer også der. Dette kan kontrolleres slik:

```bash
openssl list -providers -provider legacy
```

Git Bash har imidlertid én fallgruve: Den tolker argumenter som begynner med `/` som stier og skriver dem om. Av `-subj "/C=CH/O=Example AG/CN=..."` blir `C:/Program Files/Git/C=CH/O=Example AG/CN=...`, og OpenSSL avbryter:

```text
req: subject name is expected to be in the format /type0=value0/type1=value1/type2=...
where characters may be escaped by \. This name is not in that format:
'C:/Program Files/Git/C=CH/O=Example AG/CN=spam-quarantine.example.ch'
```

En innledende `MSYS_NO_PATHCONV=1` deaktiverer omskrivingen for det enkelte kallet. Problemet oppstår ikke i ledeteksten, PowerShell eller OpenSSL Command Prompt.

## Opprette nøkkel og CSR med OpenSSL

Ett enkelt kall oppretter nøkkel og CSR med hele SAN-listen:

```bash
openssl req -new -newkey rsa:2048 -noenc -keyout spam-quarantine.example.ch.key -out spam-quarantine.example.ch.csr -subj "/C=CH/O=Example AG/CN=spam-quarantine.example.ch" -addext "subjectAltName=DNS:spam-quarantine.example.ch,DNS:sma01.example.ch,DNS:sma02.example.ch"
```

CSR-filen sendes til CA-en, mens nøkkelen blir liggende på serveren. Du får tilbake det signerte sertifikatet med intermediate-sertifikatet, vanligvis direkte som PEM. Da er alt klart for installasjonen, og PFX-konvertering faller helt bort med denne metoden.

Nøkkelfilen er ukryptert (`-noenc`), fordi `certconfig` forventer den slik. Frem til installasjonen oppbevares den med restriktive tillatelser på serveren, og deretter slettes den eller flyttes til passordbehandleren.

## Konvertere PFX til PEM

Dette og neste avsnitt gjelder metode 2 og 3, som ender med en PFX-fil. `certconfig` forventer sertifikat og privat nøkkel som PEM, og nøkkelen må være ukryptert. Begge deler håndteres av ett enkelt OpenSSL-kall:

```bash
openssl pkcs12 -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

`-noenc` (frem til OpenSSL 3.0 het alternativet `-nodes`) skriver den private nøkkelen uten passfrase til utdatafilen. Importpassordet etterspørres uten ekko, og det vises heller ingen stjerner. Den opprettede PEM-filen inneholder sertifikat, nøkkel og medfølgende kjedesertifikater i én fil og må beskyttes tilsvarende: Slett den etter installasjonen eller flytt den til passordbehandleren.

## Når OpenSSL 3 avviser PFX-filen

Med eldre PFX-filer avbrytes konverteringen i OpenSSL 3.x med denne meldingen:

```text
Error outputting keys and certificates
error:0308010C:digital envelope routines:inner_evp_generic_fetch:unsupported:
crypto/evp/evp_fetch.c: Global default library context, Algorithm (RC2-40-CBC : 0)
```

Årsaken er ikke en ødelagt fil, men en designbeslutning: OpenSSL 3 har flyttet gamle algoritmer som RC2, RC4 og DES til en separat Legacy Provider, som ikke lastes som standard. Mange PFX-eksporter fra eldre Windows-systemer og CA-verktøy krypterer imidlertid sertifikatdelen av containeren nettopp med RC2-40-CBC. OpenSSL 1.1 åpnet slike filer uten problemer, mens OpenSSL 3 avviser dem.

Løsningen er ett ekstra alternativ:

```bash
openssl pkcs12 -legacy -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

`-legacy` laster også Legacy Provider for dette kallet, og deretter fullføres konverteringen. Forutsetningen er en OpenSSL-installasjon som inkluderer Legacy Provider; dette er tilfellet for de vanlige Windows-byggene.

Den som vil bli kvitt feilen permanent, kan løse den ved kilden og eksportere PFX-filen med moderne kryptering: Aktuelle eksportdialoger og CA-verktøy tilbyr AES-256, slik at Legacy-omveien faller helt bort.

Som grafisk alternativ fungerer XCA (X Certificate and Key Management): Importer PFX-filen via `Importieren > PKCS#12`, eksporter deretter sertifikatet som PEM fra fanen `Zertifikate` og nøkkelen separat som ukryptert PEM fra fanen `Private Schlüssel`. Begge eksportene er nødvendige, `certconfig` ber om sertifikat og nøkkel hver for seg. XCA har sitt eget kryptobibliotek og åpner også containere med gamle algoritmer.

Et ord om nedlastingskilden: OpenSSL-prosjektet publiserer ikke selv Windows-binærfiler, men viser til tredjepartsbygg som Win64 OpenSSL fra Shining Light Productions. Nedlastingsportaler med egne installasjonsprogrammer er feil sted for et kryptoverktøy.

## Importer intern rot-CA i appliance-ens tillitslager først

Aktuelle AsyncOS-versjoner validerer hele kjeden når en sertifikatprofil opprettes. Hvis sertifikatet kommer fra en intern CA som appliance-en ikke kjenner roten til, avbrytes importen med denne meldingen:

```text
Cannot import certificate: The certificate authority validation could not be
trusted for certificate because unable to get local issuer certificate
```

Appliance-en har to lister over klarerte sertifiseringsinstanser: den medfølgende systemlisten og en egendefinert liste for egne CA-er. Den interne rot-CA-en hører hjemme i den egendefinerte listen, og den må importeres før serversertifikatet installeres. Du trenger bare det offentlige CA-sertifikatet som PEM-fil (`-----BEGIN CERTIFICATE-----` til `-----END CERTIFICATE-----`), ikke noen privat nøkkel.

Slik får du rot-CA-en til appliance-en via webgrensesnittet:

1. Åpne `Network > Certificates`.
2. Klikk på `Edit Settings` i avsnittet `Certificate Authorities`.
3. Velg alternativet `Enable` ved `Custom List`.
4. Last opp PEM-filen med `Choose File`.
5. Utfør `Submit` og deretter `Commit Changes`.
6. Kontroller under `Network > Certificates > Manage Trusted Root Certificates` at CA-en vises i listen over egendefinerte sertifikater.

Hvis det allerede finnes en egendefinert liste, eksporter den først og legg den nye CA-en til i det eksisterende PEM-bundlet: Importen erstatter listen, ellers forsvinner tidligere lagrede CA-er. Ved en kjede med mellomledd må du først importere rot-CA-en, deretter intermediate-CA-en. AsyncOS kontrollerer ved import blant annet utløpsdato, duplikater og det angitte `CA:TRUE`-flagget, og avviser en intermediate så lenge den tilhørende roten mangler. Den samme importen kan også gjøres via CLI: `certconfig > CERTAUTHORITY > CUSTOM > IMPORT`, deretter `commit`.

To avgrensninger: For oppdateringer via en TLS-inspeksjonsproxy bruker SMA et separat tillitslager (`updateconfig > TRUSTED_CERTIFICATES > ADD`), og den egendefinerte CA-listen gjelder ikke der. Og rot-CA-en på SMA fjerner ikke nettleseradvarsler: Klientene trenger fortsatt rot-CA-en gjennom sin egen sertifikatdistribusjon, typisk via GPO, og appliance-en må levere serversertifikatet med intermediate-sertifikatet.

## Installasjon med certconfig

Logg på SMA via SSH og start `certconfig`. På aktuelle AsyncOS-versjoner arbeider dialogen med sertifikatprofiler:

```text
sma01.example.ch> certconfig

Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.

Choose the operation you want to perform:
- CERTIFICATE - Import, Create a request, Edit or Remove Certificate Profiles
- CERTAUTHORITY - Manage System and Customized Authorities
- CRL - Manage Certificate Revocation Lists
[]> certificate
```

Bak `CERTIFICATE` ligger operasjonene `IMPORT` (PKCS#12-fil som tidligere er lastet opp til appliance-en), `PASTE` (lim inn sertifikat i CLI-en), `NEW` (opprett selvsignert sertifikat med CSR), `EDIT`, `EXPORT`, `DELETE` og `PRINT` (viser tildelingen til tjenestene). Den vanlige metoden via SSH er `PASTE`: Dialogen ber om et navn på profilen, deretter sertifikatet, den private nøkkelen og eventuelt CA-ens intermediate-sertifikat, hver som en PEM-blokk avsluttet med én enkelt `.` på egen linje. Et avsluttende spørsmål om FQDN-kontroll av Common Name kan besvares med standardverdien. Intermediate-sertifikatet må inkluderes i profilen, ellers mangler klientene kjeden, og avhengig av nettleseren kan advarselen vedvare til tross for gyldig sertifikat.

Eldre AsyncOS-versjoner (slik Cisco-teknote-en beskriver dem) viser i stedet en `SETUP`-dialog. Den starter med spørsmålet `Do you want to use one certificate/key for receiving, delivery, HTTPS management access, and LDAPS?`: Et `y` tildeler samme par til alle fire tjenester, mens et `n` går gjennom spørsmålene om sertifikat, nøkkel og intermediate én gang per tjeneste. Prinsippet for innliming er identisk.

To punkter avgjør om det lykkes eller ikke: Ikke avslutt økten med Ctrl+C, da forkastes alle endringer umiddelbart. Kjør til slutt `commit`; først da er sertifikatet aktivt. Med to appliance-er gjentas prosessen på begge, sertifikatkonfigurasjonen synkroniseres ikke mellom SMA-er.

## Kontroll

Den raskeste testen utføres utenfra mot karantenesiden. Sluttbrukertilgangen til spam-karantenen ligger som standard på HTTPS-port 83, dersom ikke noe annet ble konfigurert ved aktivering:

```bash
openssl s_client -connect spam-quarantine.example.ch:83 -servername spam-quarantine.example.ch </dev/null 2>/dev/null | openssl x509 -noout -subject -enddate
```

Utdataene må vise det nye Subject og den nye utløpsdatoen. På appliance-en viser `certconfig` med operasjonen `PRINT` de aktive sertifikatene, og nettleserkontrollen mot admin-GUI-en og karantenesiden bekrefter at kjeden er korrekt bygget opp.

## Kilder

1.  [How to Generate and Install a Certificate on an SMA](https://www.cisco.com/c/en/us/support/docs/security/content-security-management-appliance/118460-technote-sma-00.html): Cisco-teknote med certconfig-prosessen for eldre AsyncOS-versjoner, PEM-kravet og metodene for sertifikatopprettelse via ESA eller OpenSSL.

2.  [Common Administrative Tasks, AsyncOS 16.0 for Cisco Secure Email and Web Manager](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_01011.html): Kapittel i administrasjonsveiledningen om håndtering av Certificate Authority-listene (system- og egendefinert liste), inkludert kontrollene ved CA-import.

3.  [Comprehensive Spam Quarantine Setup Guide on ESA and SMA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118692-configure-esa-00.html): Cisco-veiledning om spam-karantene, inkludert sluttbrukertilgang via HTTPS på port 83.

4.  [openssl-req](https://docs.openssl.org/master/man1/openssl-req/): Referanse for opprettelse av nøkkel og CSR, inkludert `-addext` for SAN-listen.

5.  [openssl-pkcs12](https://docs.openssl.org/master/man1/openssl-pkcs12/): Referanse for konverteringsalternativene, blant annet `-noenc` (tidligere `-nodes`) og `-legacy`.

6.  [OpenSSL 3.0 Migration Guide](https://docs.openssl.org/3.0/man7/migration_guide/): Bakgrunn om utflyttingen av gamle algoritmer til Legacy Provider.

7.  [XCA: X Certificate and Key Management](https://hohnstaedt.de/xca/): Åpen kildekode-verktøy for import og eksport av PKCS#12- og PEM-strukturer.

8.  [Win32/Win64 OpenSSL](https://slproweb.com/products/Win32OpenSSL.html): Windows-bygg fra Shining Light Productions, som OpenSSL-prosjektet viser til, inkludert publisert sjekksumliste.

9.  [MSYS2: Filesystem Paths](https://www.msys2.org/docs/filesystem-paths/): Beskrivelse av den automatiske stiomskrivingen som endrer argumentet `-subj` i Git Bash, inkludert `MSYS_NO_PATHCONV`.
