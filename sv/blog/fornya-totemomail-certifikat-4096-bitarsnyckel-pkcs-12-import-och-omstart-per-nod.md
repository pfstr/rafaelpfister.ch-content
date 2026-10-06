---
title: "Förnya TotemoMail-certifikat: 4096-bitarsnyckel, PKCS#12-import och omstart per nod"
navTitle: "Förnya certifikat"
description: "TotemoMails ansökningsdialog skapar endast 2048-bitarsnycklar utan alternativa namn. Många interna CA:er signerar dock numera endast 4096 bitar. Nyckel och ansökan skapas därför med openssl, följt av beställning hos PKI-instansen, PKCS#12-import, bindning till anslutningar och omstart per nod."
date: "2026-10-06"
kategorie: "TotemoMail"
timeToRead: "12 min lästid"
themen:
  - totemomail
  - e-mail-verschluesselung
produkte:
  - "totemomail"
protokolle:
  - "tls"
  - "smtp"
slug: "fornya-totemomail-certifikat-4096-bitarsnyckel-pkcs-12-import-och-omstart-per-nod"
translationId: "article-1e59c4ee01e408a3"
translationOf: totemomail-zertifikat-erneuern
url: https://rafaelpfister.ch/sv/blog/fornya-totemomail-certifikat-4096-bitarsnyckel-pkcs-12-import-och-omstart-per-nod
translationSourceHash: 3ba1992d93abcd33fda47f86cb3b1ea4c8884c36fcfa41fa5c098f4aff9dff32
translationModel: gpt-5.6-terra
translatedAt: 2026-10-06T10:34:26.965Z
translationReview: automatic
---

# Förnya TotemoMail-certifikat: 4096-bitarsnyckel, PKCS#12-import och omstart per nod

Att förnya servercertifikatet för ett TotemoMail-kluster (numera Kiteworks Email Protection Gateway) ser ut som en rutinuppgift: skapa en ansökan i administrationsgränssnittet, få den signerad och importera svaret. I praktiken misslyckas den vägen ofta redan i första steget. Dialogrutan ”New PKCS#10” skapar alltid en 2048-bitarsnyckel och erbjuder inget fält för alternativa namn (Subject Alternative Names). Många interna certifikatutfärdare signerar numera endast 4096 bitar, och ett certifikat utan alternativa namn betraktas inte ens som giltigt för ett namn av aktuella TLS-motparter.

Följande förfarande har visat sig fungera vid ett byte hösten 2026 i förproduktion och produktion: nyckel och ansökan med openssl på gatewayen, beställning hos PKI-instansen, import som PKCS#12, bindning till anslutningarna och omstart nod för nod. Vid bytet upptäcktes flera egenheter, bland annat ett fel i gränssnittet som uppstår när det gamla certifikatet kopplas loss.

## Två certifikat med olika syften

Ett TotemoMail-kluster bakom Exchange Online behöver normalt två typer av certifikat.

| Typ | Syfte | Utfärdare | Giltighetstid |
|---|---|---|---|
| Internt | Webbgränssnitt, administration, interna SMTP-anslutningar. Innehåller nodernas interna namn. | intern PKI, till exempel Active Directory Certificate Services | valfri, vanligtvis 12 till 13 månader |
| Offentligt | Sträckan mellan Exchange Online och gatewayen, om Exchange Online ska kontrollera certifikatet | offentlig certifikatutfärdare | högst 200 dagar sedan 15.03.2026, högst 100 dagar från 15.03.2027 |

Ett enda certifikat för båda syftena fungerar inte. Offentliga certifikatutfärdare utfärdar inga interna namn och inga kortnamn utan domän, och varje offentligt certifikat syns i Certificate Transparency-loggarna. Omvänt litar Exchange Online inte på någon intern kedja. Hur Exchange Online hanterar sträckan till en krypteringsgateway beskrivs i artikeln [Mail-slinga med krypteringsgateway bakom EXO](https://rafaelpfister.ch/blog/verschluesselungsgateway-hinter-exchange-online).

Följande steg gäller det interna certifikatet. För det offentliga är förfarandet identiskt fram till importen; skillnaderna beskrivs i sista avsnittet.

## Översikt över förfarandet

1. Skapa nyckel och ansökan med openssl på en nod.
2. Beställ ansökan hos PKI-instansen.
3. Kontrollera det levererade certifikatet.
4. Sammanfoga certifikat, nyckel och mellanliggande certifikat till en PKCS#12-fil och hämta den till datorn med webbläsaren.
5. Importera i TotemoMail-gränssnittet och bind till anslutningarna.
6. Starta om varje nod separat och kontrollera.
7. Testa e-postflödet och städa sedan upp.

Planera minst tre veckor före utgångsdatumet. Så länge det gamla certifikatet är giltigt finns en återväg. Om det finns en förproduktion i er miljö, byt där först och använd genomgången som referens.

## Steg 1: Nyckel och ansökan med openssl

Arbeta på en nod i klustret med din personliga användare, inte med tjänstkontot `totemo`. Då kan du senare hämta PKCS#12-filen direkt via `scp`. Root-behörighet behövs inte för något av stegen.

```bash
umask 077
mkdir -m 700 ~/csr-2026
cd ~/csr-2026
```

Konfigurationen innehåller innehavare, användningsändamål och alla alternativa namn. Namnen i exemplet är platshållare: tre noder och tjänstenamnen som klustret nås under internt.

```bash
cat > intern.cnf <<'EOF'
[ req ]
default_md         = sha256
prompt             = no
distinguished_name = dn
req_extensions     = ext

[ dn ]
C  = CH
O  = Beispiel AG
CN = SecureMail

[ ext ]
basicConstraints = critical, CA:FALSE
keyUsage         = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName   = @alt

[ alt ]
DNS.1 = gw01.intern.example.ch
DNS.2 = gw02.intern.example.ch
DNS.3 = gw03.intern.example.ch
DNS.4 = securemail.intern.example.ch
EOF
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Post | Effekt |
|---|---|
| `default_md = sha256` | Hashmetod för signering av ansökan |
| `prompt = no` | Använd värden från filen i stället för att fråga interaktivt |
| `req_extensions = ext` | Skriv tillägg från avsnittet `[ ext ]` i ansökan |
| `basicConstraints = critical, CA:FALSE` | Slutcertifikat, ingen certifikatutfärdare |
| `keyUsage` | Nyckel för signering och nyckelutbyte, som är vanligt för TLS-servrar |
| `extendedKeyUsage = serverAuth, clientAuth` | Server- och klientautentisering. Gatewayen är server på vissa sträckor och klient på andra. |
| `subjectAltName = @alt` | alternativa namn från avsnittet `[ alt ]` |

</details>

Ta endast med fullständiga namn. Kortnamn utan domän avvisas av många registreringsinstanser, och du bör inte förlita dig på att de hamnar i certifikatet.

Skapa nyckeln utan lösenfras och skydda den med filbehörigheter. Den finns bara på noden fram till importen och raderas sedan. En lösenfras på en nyckel som existerar i några dagar ger litet skydd och skapar en ny risk: om den går förlorad blir nyckeln oanvändbar och certifikatet måste utfärdas på nytt.

```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -out intern.key
openssl req -new -key intern.key -config intern.cnf -out intern.csr
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `genpkey -algorithm RSA` | skapa en ny privat nyckel av typen RSA |
| `-pkeyopt rsa_keygen_bits:4096` | nyckellängd 4096 bitar |
| `-out intern.key` | fil för nyckeln, utan lösenfras |
| `req -new` | skapa en ny certifikatansökan (CSR) |
| `-key intern.key` | använd befintlig nyckel |
| `-config intern.cnf` | innehavare, tillägg och namn från konfigurationen |
| `-out intern.csr` | fil för ansökan |

</details>

Kontrollera ansökan innan den lämnar maskinen:

```bash
openssl req -in intern.csr -noout -verify -subject
openssl req -in intern.csr -noout -text | grep -E "Public-Key|Signature Algorithm" | head -2
openssl req -in intern.csr -noout -text | grep -o "DNS:[^,]*" | wc -l
```

Det förväntade är `verify OK`, rätt innehavare, `4096 bit` och antalet namn. Anteckna dessutom fingeravtrycket för den offentliga nyckeln. Med det kan du senare entydigt koppla det levererade certifikatet till denna nyckel, även om två ansökningar har samma innehavare:

```bash
openssl req -in intern.csr -noout -pubkey |
  openssl pkey -pubin -outform DER |
  openssl dgst -sha256
```

## Steg 2: Beställning hos PKI-instansen

Instansen som utfärdar certifikatet behöver följande uppgifter per ansökan:

- CSR:en som text
- SHA-256-fingeravtrycket för den offentliga nyckeln
- typen: intern eller offentlig, och vid flera miljöer vilken
- namnlistan för kopiering, ett namn per rad
- nyckellängd 4096, utökad användning Server Authentication och Client Authentication
- datumet då certifikatet måste finnas tillgängligt, minst två veckor före utgångsdatumet

Vid ett byte med flera miljöer och certifikattyper är det värt att lägga en översikt i början av mejlet: vilka certifikat som finns, vilka som utfärdas internt och vilka offentligt samt i vilken ordning de behövs. Ingen ändring i systemet krävs för utfärdandet. Certifikaten för alla miljöer kan därför utfärdas samtidigt, även om de installeras efter varandra.

Vid bytet hösten 2026 framkom flera punkter hos registreringsinstansen (RA) som sannolikt förekommer på liknande sätt i många miljöer:

- **RA:n anger innehavaren själv.** I certifikatet stod endast land, organisation och Common Name, även om ansökan innehöll organisationsenhet, ort och kanton.
- **Inga kortnamn.** Namn utan domän saknades i det utfärdade certifikatet.
- **Namn överfördes manuellt.** RA:n tog inte över de alternativa namnen från CSR:en, utan de skrevs in i formuläret. Ett namn kom fram avklippt. Den levererade namnlistan måste därför alltid kontrolleras.
- **Fel profil.** En RA som hanterar både den interna CA:n och en offentlig CA utfärdade först ansökningar om offentliga certifikat med den interna profilen. Den interna CA:n stod som utfärdare. Sådana certifikat är värdelösa för sträckan till Exchange Online.
- **Endast ett namn för offentliga enkelcertifikat.** En Single-Domain-produkt avbryter med en ansökan med två namn, till exempel med ”Only one Subject Alternative Name is allowed”. Ett namn räcker dock, se avsnittet om det offentliga certifikatet.

Certifikat som uppstår genom felutfärdanden bör PKI-instansen sedan spärra. De har en giltig nyckel och är annars giltiga i ett år.

## Steg 3: Kontrollera leveransen

Lägg det levererade certifikatet i samma katalog som nyckeln, exempelvis med `cat > intern.crt`, klistra in innehållet, `Strg+D`. Kontrollera sedan:

```bash
openssl x509 -in intern.crt -noout -subject -issuer -serial -dates
openssl x509 -in intern.crt -noout -ext subjectAltName,extendedKeyUsage,keyUsage
```

Kontrollera utfärdaren, den fullständiga namnlistan utan skrivfel och dubbletter samt de båda användningsändamålen. Jämförelsen av fingeravtrycken visar om certifikatet och nyckeln hör ihop. Båda värdena måste vara lika:

```bash
openssl x509 -in intern.crt -noout -pubkey |
  openssl pkey -pubin -outform DER |
  openssl dgst -sha256
openssl pkey -in intern.key -pubout -outform DER |
  openssl dgst -sha256
```

För ett offentligt certifikat tillkommer två kontroller. Policy-OID:n `2.23.140.1.2.2` står för ett organisationsvaliderat certifikat enligt CA/Browser Forums regler, och certifikatet måste innehålla inbäddade Certificate Transparency-bevis. Några minuter efter utfärdandet visas det under sitt namn på crt.sh. Om båda saknas är det inget offentligt certifikat, oberoende av hur leveransen är märkt.

## Steg 4: Skapa PKCS#12-filen

TotemoMail importerar certifikat och nyckel tillsammans som PKCS#12. Hämta därför certifikatet för den utfärdande mellaninstansen. Adressen finns i certifikatet under `Authority Information Access`:

```bash
openssl x509 -in intern.crt -noout -ext authorityInfoAccess
curl -sS -o issuing.crt http://pki.example.ch/crt/Issuing-CA.crt
file issuing.crt
```

Om `file` inte rapporterar `PEM certificate`, är filen DER-kodad och måste konverteras:

```bash
openssl x509 -inform DER -in issuing.crt -out issuing.pem
mv issuing.pem issuing.crt
```

Mellaninstansens Subject Key Identifier måste stämma överens med certifikatets Authority Key Identifier. Skapa sedan filen. Kommandot frågar efter ett exportlösenord som du behöver vid importen:

```bash
openssl pkcs12 -export \
  -inkey intern.key \
  -in intern.crt \
  -certfile issuing.crt \
  -name "SecureMail" \
  -out intern.p12
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-export` | skapa PKCS#12-fil |
| `-inkey intern.key` | privat nyckel |
| `-in intern.crt` | utfärdat certifikat |
| `-certfile issuing.crt` | ytterligare certifikat i kedjan, här mellaninstansen |
| `-name "SecureMail"` | visningsnamn för posten i filen |
| `-out intern.p12` | målfil, skyddad med exportlösenordet |

</details>

Kontroll: två poster måste visas, certifikatet och mellaninstansen:

```bash
openssl pkcs12 -in intern.p12 -nokeys 2>/dev/null | grep -E "subject=|issuer="
```

TotemoMail-gränssnittet körs i webbläsaren, vanligtvis på en jumphost. Filen måste dit. I Windows finns OpenSSH-klienten med `scp`:

```powershell
New-Item -ItemType Directory -Path "$env:USERPROFILE\Downloads\zert" -Force | Out-Null
scp benutzer@gw01.intern.example.ch:csr-2026/intern.p12 "$env:USERPROFILE\Downloads\zert\"
scp benutzer@gw01.intern.example.ch:csr-2026/issuing.crt "$env:USERPROFILE\Downloads\zert\"
```

Om du trots allt skapade nyckeln som `totemo`, ligger den under `/opt/totemomail`, och din användare kan inte läsa den. Kopiera då PKCS#12-filen kort till `/tmp`, hämta den därifrån och radera den genast igen. Omvägen via urklipp med Base64 fungerar visserligen, men är felbenägen med en rad på omkring 10 000 tecken.

## Steg 5: Importera och binda i TotemoMail

Under `Key Management`:

1. **`Issuer Certificates`**: importera mellaninstansen `issuing.crt`.
2. **`Own Server Certificates`**, knappen **`Import`**: Dialogrutan erbjuder två vägar. Till vänster är ”Import certificate” för certifikat med nyckel, alltså PKCS#12-filen. Till höger är ”Import a PKCS#10 certificate reply” endast för svar på ansökningar som TotemoMail har skapat självt. Välj till vänster och ange exportlösenordet i andra steget.
3. Öppna det nya certifikatet (pennsymbol). Under `Connector` ska alla anslutningar vara ikryssade, i den beskrivna installationen `8443=Admin`, `443=SecMail`, `7444=MailAPI`, `10443=SENDIT` och `8444=AdminAPI`. `Host` är satt till `*`.
4. Öppna det gamla certifikatet och avmarkera alla anslutningar utom en.

Punkt 4 omfattar felet som uppstod vid bytet: om du avmarkerar **alla** anslutningar på det gamla certifikatet rapporterar gränssnittet ”Could not edit selected server certificate”. Loggen innehåller:

```
ERROR [EditServerCertBean] Could not edit certificate
ch.totemo.core.actions.ActionException: Failed to edit key.
Caused by: java.lang.NullPointerException
```

TotemoMail tolererar inget servercertifikat utan anslutning. Låt därför en anslutning vara kvar, till exempel `8444=AdminAPI`, och radera det gamla certifikatet efter observationsperioden. Att radera i stället för att avmarkera fungerar; certifikatet hamnar under `Deleted Certificates`.

Tre observationer underlättar förståelsen av bindningen:

- **Anslutningslistan gäller bara webbtjänsterna.** Port 25 visas inte där. Om inget certifikat av typen SMTPS finns använder SMTP HTTPS-certifikatet. Därför visar port 25 det nya certifikatet efter bytet, trots att ingen markering finns i listan för SMTPS.
- **En uttrycklig tilldelning har företräde framför asterisken.** Anslutningen som lämnas kvar på det gamla certifikatet fortsätter att visa det gamla, även om det nya med `*` är angivet för alla.
- **Listan är gemensam för klustret.** Den ser likadan ut på varje nod, oavsett om noden redan har tagit över ändringen. Vad en nod faktiskt visar framgår endast av en kontroll på portarna.

## Steg 6: Omstart nod för nod

TotemoMail läser in certifikaten vid start. Efter importen visar alla noder fortfarande det gamla certifikatet tills de har startats om. Starta om varje nod separat, aldrig alla samtidigt, så att noder alltid är aktiva bakom lastbalanseraren. Börja med de noder som du inte är inloggad via, och ta noden med det öppna gränssnittet sist.

På noden som `totemo`:

```bash
totemomail stop
totemomail start
```

Ytterligare anvisningar om kontrollerad avstängning finns i artikeln [Viktigaste kontrollerna för TotemoMail-administratörer](https://rafaelpfister.ch/blog/totemomail-server-stoppen-queues-bereinigen).

Webbtjänsterna är åter tillgängliga efter ungefär en minut, SMTP-tjänsten på port 25 först efter några minuter. Ett tomt svar på port 25 kort efter starten tyder alltså ännu inte på ett fel. Först när en nod har växlat om på alla portar är det dags för nästa.

Kontrollen över alla noder och portar:

```bash
for h in gw01 gw02 gw03; do
  for p in 25 443 8443; do
    if [ "$p" = 25 ]; then s="-starttls smtp"; else s=""; fi
    c=$(echo | openssl s_client -connect "$h:$p" $s 2>/dev/null |
        openssl x509 -noout -serial -enddate 2>/dev/null | tr '\n' ' ')
    printf "%-6s %-5s %s\n" "$h" "$p" "$c"
  done
done
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `s_client -connect host:port` | upprätta TLS-anslutning till tjänsten |
| `-starttls smtp` | för port 25, genomför först SMTP-dialogen och växla sedan till TLS med STARTTLS |
| `x509 -noout` | läs certifikatet utan att skriva ut det |
| `-serial -enddate` | visa serienummer och utgångsdatum |

</details>

Om port 25 fortfarande är tom efter fem minuter visar `ss -lnt | grep ':25 '` om tjänsten lyssnar, och loggen under `/opt/totemomail` orsaken.

Alternativet `-showcerts` visar om TotemoMail skickar med mellaninstansen:

```bash
echo | openssl s_client -connect gw01:25 -starttls smtp -showcerts 2>/dev/null | grep -E " s:| i:"
```

Vid det beskrivna bytet visades endast slutcertifikatet, trots att mellaninstansen fanns i PKCS#12-filen och under `Issuer Certificates`. För interna sträckor där ingen kontrollerar certifikatet får detta inga följder. För kontroll av Exchange Online måste kedjan däremot vara komplett.

## Steg 7: Tester, återväg, städning

Testa e-postflödet i båda riktningarna efter sista omstarten: ett meddelande utifrån genom slingan och ett utåt via gatewayen. I meddelandespårningen i Exchange Online måste båda sträckorna visas som levererade, och i riktning mot gatewayen får inget fastna i kön.

**Återvägen** består av att markera anslutningarna igen på det gamla certifikatet, avmarkera alla utom en på det nya och starta om noderna separat på nytt. Det fungerar bara så länge det gamla certifikatet är giltigt.

Efter lyckade tester:

- radera PKCS#12-filen på jumphosten
- radera arbetskatalogen på noden; nyckeln finns nu i TotemoMails nyckellager
- radera det gamla certifikatet efter några dagar och starta om noderna separat en gång till så att även den sista anslutningen växlar om
- låt PKI-instansen spärra felutfärdanden
- lägg in det nya utgångsdatumet i bevakningen

Meddela administratörerna om bytet när det nya certifikatet inte längre innehåller kortnamn. Den som tidigare öppnat gränssnittet med `https://gw01:8443` får därefter en certifikatvarning. Det gäller även övervakningar och skript som använder kortnamn.

## Egenheter i korthet

| Observation | Följd | Hantering |
|---|---|---|
| ”New PKCS#10” skapar 2048 bitar utan alternativa namn | Ansökan oanvändbar för 4096-bitars-CA:er | skapa nyckel och ansökan med openssl |
| Ändringar får effekt först efter omstart | Noder fortsätter att visa det gamla certifikatet | starta om varje nod separat |
| Port 25 kommer några minuter efter webbtjänsterna | tomt svar kort efter start | vänta, ta sedan nästa nod |
| Certifikat utan anslutning utlöser en NullPointerException | gammalt certifikat kan inte kopplas loss helt | lämna en anslutning kvar, radera senare |
| Uttrycklig tilldelning har företräde framför `*` | en anslutning fortsätter att visa det gamla certifikatet | radera gammalt certifikat efter observationen |
| Endast slutcertifikatet skickas | motparter som kontrollerar kan inte bygga kedjan | klargör före kontroll genom Exchange Online |
| RA:n överför namn manuellt | skrivfel och saknade namn möjliga | kontrollera namnlistan i leveransen |

## Det offentliga certifikatet för sträckan till Exchange Online

För att Exchange Online ska kontrollera sträckan till gatewayen utöver krypteringen behöver gatewayen ett offentligt certifikat på port 25. Först då kan den utgående anslutningen på `TlsSettings DomainValidation` ställas om med en `TlsDomain`, och den inkommande anslutningen bindas till certifikatet via `TlsSenderCertificateName`.

Ett enda namn räcker för detta. Exchange Online jämför i båda riktningarna endast namnet i certifikatet med det angivna värdet; adressen bakom kontrolleras inte. Ett certifikat för namnet som gatewayen är känd under utåt täcker både ut- och återvägen. Vid beställningen bör du uttryckligen fråga efter Client Authentication: flera offentliga certifikatutfärdare tog bort denna användning från TLS-certifikat 2026. Den behövs för återvägen, där gatewayen identifierar sig som klient.

För användning i TotemoMail framträder följande väg utifrån observationerna ovan: importera det offentliga certifikatet som typen SMTPS, så att port 25 visar det medan webbtjänsterna behåller det interna. Två punkter måste klarläggas först. Kedjan måste skickas komplett, och port 25 visar därefter det offentliga certifikatet för alla avsändare, även interna såsom en föregående gateway. Om en avsändare förväntar sig ett visst certifikat bryts den sträckan.

## Källor

1.  [CA/Browser Forum: Ballot SC081v3](https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/): Tidsplan för giltighetstiden för offentliga TLS-certifikat, 200 dagar från mars 2026, 100 dagar från mars 2027, 47 dagar från mars 2029.

2.  [Microsoft Learn: Set-OutboundConnector](https://learn.microsoft.com/powershell/module/exchange/set-outboundconnector): Parametrarna `TlsSettings` och `TlsDomain` för kontroll av certifikatet hos motparten.

3.  [Microsoft Learn: Set-InboundConnector](https://learn.microsoft.com/powershell/module/exchange/set-inboundconnector): Parametern `TlsSenderCertificateName` för tilldelning via avsändarens certifikat.

4.  [OpenSSL-dokumentation: openssl-req](https://docs.openssl.org/master/man1/openssl-req/): Struktur för konfigurationsfilen och alternativ för certifikatansökningar.

5.  [OpenSSL-dokumentation: openssl-pkcs12](https://docs.openssl.org/master/man1/openssl-pkcs12/): Skapa och kontrollera PKCS#12-filer.

6.  [crt.sh](https://crt.sh): Sökning i Certificate Transparency-loggarna för att kontrollera utfärdandet av ett offentligt certifikat.
