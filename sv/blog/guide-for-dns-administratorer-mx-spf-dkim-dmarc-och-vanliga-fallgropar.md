---
title: "Guide för DNS-administratörer: MX, SPF, DKIM, DMARC och vanliga felkällor"
navTitle: "E-post-DNS-poster"
description: "Den som ansvarar för en zon får oftast färdiga e-postposter och ska bara publicera dem. Det som regelbundet går fel: 255-byte-gränsen för DKIM, dubbla SPF-poster, uppslagsgränsen, MX på ett CNAME, automatiskt tillagt zonsuffix och policyer som ingen längre upprätthåller."
date: "2026-08-04"
kategorie: "SMTP och e-postflöde"
timeToRead: "15 min lästid"
themen:
  - smtp-mailflow
  - e-mail-verschluesselung
protokolle:
  - "dns"
  - "smtp"
  - "tls"
  - "verschluesselung"
  - "mail-auth"
hauptthema: "smtp-mailflow"
related:
  - smtp-verbindung-testen-linux
  - ghost-sender-exchange-online-nebeneingang
slug: "guide-for-dns-administratorer-mx-spf-dkim-dmarc-och-vanliga-fallgropar"
translationId: "article-e4699ad7fcea2e20"
translationOf: dns-records-e-mail-stolpersteine
translationSourceHash: e78b63f0179944cf4c36caf7aabfb9ceb0c5dd40d2d3cf1a25e92fee8cbacaca
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:44:25.286Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/guide-for-dns-administratorer-mx-spf-dkim-dmarc-och-vanliga-fallgropar
---

# Guide för DNS-administratörer: MX, SPF, DKIM, DMARC och vanliga felkällor

Den som ansvarar för en DNS-zon får sällan e-postposter skrivna från grunden. E-postteamet, en leverantör eller en marknadsföringstjänst skickar en rad med uppmaningen att den "bara behöver publiceras". Just där uppstår de flesta felen, eftersom e-postposter är den posttyp där ett skrivfel kan få två helt olika följder. Antingen upphör leveransen omedelbart och någon hör av sig inom några minuter, eller så fortsätter den oförändrat och endast avsändarkontrollen faller tyst bort. Det andra fallet förblir regelbundet oupptäckt i månader, tills en stor mottagare sätter domänen i karantän.

Sedan Google och Yahoo skärpte sina krav för massavsändare i februari 2024 och Microsoft följde efter i maj 2025 har toleransen för halvkonfigurerade domäner blivit liten. SPF, DKIM och en DMARC-post är inte längre en bonus för avsändare från en viss volym, utan en förutsättning för leverans.

Alla exempel i denna artikel använder `example.com` och generiska selektorer. De visade värdena är förkortade för att förbli läsbara.

## Regler som gäller för varje e-postpost

### 255-byte-gränsen för TXT-poster

En TXT-post består enligt RFC 1035 av en eller flera `character-strings`, och en enskild sådan teckensträng rymmer högst 255 byte. Posten som helhet får vara längre, men måste då delas upp i flera teckensträngar. System som utvärderar den fogar ihop dessa delar igen utan avgränsare.

I praktiken blir detta relevant på exakt ett ställe: vid DKIM-nycklar med 2048 bitar. Deras Base64-värde är omkring 400 tecken långt och ryms inte i en teckensträng.

```text
selector1._domainkey.example.com. 3600 IN TXT (
    "v=DKIM1; k=rsa; "
    "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA0ZenWBnGUqydzg5w"
    "yWxRRNBZjbagzDh1BlW3b145Wer/GWfbz6XkCyqsN918N+/Va6mVe37rXNaZAS"
    "/js/L3m7d2OlUp5I3jHC5EsU6XwU5trKFxPWErLxtanYbXLXabyVIRGkop+1s3"
    "SXNg2Oy5eNyZf5MGlEAo+JM6oXtgkABQQn5kE1ShzalXJUVL/wIDAQAB" )
```

De flesta DNS-hanteringssystem gör denna uppdelning själva när värdet anges via det vanliga inmatningsfältet. Den som i stället sätter citattecken manuellt måste hålla gränsen exakt. Ett radbrutet värde med ett mellanslag i skarven ger en nyckel som finns syntaktiskt men inte längre passar kryptografiskt.

Kontrollen efteråt är viktig, eftersom en felaktigt sammansatt nyckel ser helt oansenlig ut i GUI:t:

```bash
dig +short TXT selector1._domainkey.example.com | tr -d '" ' | wc -c
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `+short` | Visar endast postvärdena, utan rubriker och metadata |
| `TXT selector1._domainkey.example.com` | Posttyp och namn på DKIM-nyckelposten |
| `tr -d '" '` | Tar bort citattecken och mellanslag, och sätter därmed ihop delteckensträngarna så som en verifierare läser dem |
| `wc -c` | Räknar tecknen i det sammansatta värdet; längden måste stämma med mallen |

</details>

### En post per syfte

SPF och DMARC är definierade så att exakt en passande post får finnas under ett namn. Vid SPF leder två `v=spf1`-poster till en `permerror`, och kontrollen betraktas då som misslyckad, inte som godkänd. Vid DMARC ignorerar mottagare domänen helt om flera poster börjar med `v=DMARC1`: i stället för en strikt policy gäller då ingen alls.

Detta är överlägset det vanligaste felet i uppvuxna zoner. En ny tjänsteleverantör ansluts, någon lägger till "sin" SPF-post i stället för att utöka den befintliga, och från den stunden misslyckas kontrollen för samtliga avsändare. Kontrollera därför alltid vad som redan finns före varje ny post:

```bash
dig +short TXT example.com | grep -i spf1
dig +short TXT _dmarc.example.com
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `+short` | Visar endast postvärdena, utan rubriker och metadata |
| `TXT` | Efterfrågad posttyp |
| `example.com`, `_dmarc.example.com` | Efterfrågade namn: själva domänen för SPF, `_dmarc`-namnet för DMARC |
| `grep -i spf1` | Filtrerar fram SPF-raden; `-i` ignorerar stora och små bokstäver |

</details>

För DKIM gäller motsatsen: där är en post per selektor avsedd, och flera selektorer sida vid sida är normalt eftersom varje sändningstjänst har sin egen nyckel.

### Zonsuffixet i webbgränssnitt

I Infoblox, Windows DNS och praktiskt taget alla hostinggränssnitt läggs zonnamnet automatiskt till det angivna namnet. Den som anger det fullständigt kvalificerade namnet i fältet "Namn" får en post som är dubbelt så lång som avsett:

```text
Eingabe:   selector1._domainkey.example.com
Ergebnis:  selector1._domainkey.example.com.example.com
```

I zonfilen är motsvarigheten den saknade avslutande punkten. `mail.example.com` utan punkt i slutet är ett relativt namn och kompletteras med zonnamnet, medan `mail.example.com.` med punkt är absolut. För MX- och CNAME-mål avgör denna enda punkt om domänen är nåbar.

### Copy-paste är den vanligaste felkällan

Värden för e-postposter skrivs nästan aldrig in, utan kopieras från en PDF, ett ärende, en Excel-cell eller en chatt. Då uppstår skador som förblir osynliga i inmatningsfältet:

- Ett dubbelt `p=` i början av DKIM-nyckeln, eftersom prefixet sattes två gånger vid sammansättningen. Värdet `v=DKIM1;k=rsa;p=p=MIIBIjAN...` förekommer regelbundet i praktiken och ger en oanvändbar nyckel.
- Typografiska citattecken från Word i stället för raka.
- Hårda mellanslag från PDF-layouter som ser ut som vanliga.
- Radbrytningar mitt i Base64-blocket när värdet sträckte sig över flera rader i PDF:en.

Base64 känner endast tecknen A till Z, a till z, 0 till 9, `+`, `/` och `=` som utfyllnadstecken. Allt annat i `p=`-delen är ett fel. Ett kort filter före inmatning sparar senare felsökning:

```bash
printf '%s' "$KEY" | tr -d 'A-Za-z0-9+/=' | wc -c
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `'%s' "$KEY"` | Skriver ut nyckelvärdet oförändrat och utan tillagd radbrytning |
| `tr -d 'A-Za-z0-9+/='` | Tar bort alla tecken som är giltiga för Base64; endast främmande tecken återstår |
| `wc -c` | Räknar de återstående tecknen |

</details>

Om något annat än `0` visas här innehåller nyckeln främmande tecken.

### Sänk TTL före ändringar

Före varje planerat byte av en MX-, SPF- eller DKIM-post bör TTL sänkas till ett lågt värde under några timmar, vanligtvis 300 sekunder. Annars ligger det gamla värdet kvar i externa resolvers i upp till en dag eller längre beroende på zon, och en återställning tar lika lång tid. Efter ändringen och en observationsperiod återställs TTL till det normala värdet.

## MX

MX-posten fastställer vilken värd som tar emot e-post för domänen. Det finns två regler som regelbundet överträds.

**Målet måste vara ett värdnamn med A- eller AAAA-post.** Varken en IP-adress eller ett CNAME är tillåtet. RFC 2181 fastslår uttryckligen att målet för en MX-post inte får vara ett alias. I praktiken fungerar det ändå hos många mottagare, men inte hos andra, vilket leder till felbilder som till synes bara berör enskilda avsändare.

```text
example.com.        IN  MX  10 mail1.example.com.
example.com.        IN  MX  20 mail2.example.com.
mail1.example.com.  IN  A   192.0.2.10
mail2.example.com.  IN  A   192.0.2.11
```

**Talet är en preferens, inte en viktning.** Det lägre värdet försöks först. En andra MX med högt tal är bara meningsfull om detta system känner till samma mottagarfilter. Backup-MX-poster till system utan mottagarkontroll är ett populärt mål för spam, eftersom angripare riktar in sig på den svagaste posten.

Domäner som endast skickar eller inte alls har med e-post att göra får en Null MX enligt RFC 7505. Den signalerar att domänen inte tar emot e-post och säkerställer ett omedelbart och entydigt avslag i stället för timeouter:

```text
example.com.  IN  MX  0 .
```

Null MX ersätter dock inte en SPF- och DMARC-post. Att inte ta emot betyder inte att ingen skickar i ert namn. Särskilt parkerade underdomäner används för spoofing eftersom få tittar där.

## A, AAAA, PTR och HELO-namnet

PTR-posten för den utgående IP-adressen finns inte i er zon, utan i `in-addr.arpa`-zonen hos leverantören som äger adressblocket. Den beställs därför från leverantören och sätts inte själv. Många stora mottagare kräver att PTR och den tillhörande framåtriktade posten stämmer överens, alltså att namnet i PTR åter upplöses till samma IP-adress.

```bash
dig +short -x 192.0.2.10
dig +short A mail1.example.com
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `+short` | Visar endast postvärdena, utan rubriker och metadata |
| `-x 192.0.2.10` | Omvänd uppslagning: dig skapar själv PTR-namnet i `in-addr.arpa`-zonen |
| `A mail1.example.com` | Framåtriktad uppslagning av namnet från PTR, för att kontrollera att rundgången leder till samma IP-adress |

</details>

Namnet som er e-postserver anger i HELO eller EHLO bör vara detsamma och också kunna upplösas. En gateway som presenterar sig som `localhost.localdomain` eller med ett internt namn bedöms sämre av större mottagare.

Var försiktig när du lägger till en AAAA-post. Så snart e-postservern blir nåbar och skickande via IPv6 gäller samma krav som för IPv4, i vissa delar ännu strängare. Google kräver en giltig PTR för sändande IPv6-adresser. Saknas den avvisas sändningen, medan den fungerade felfritt via IPv4. En AAAA-post på e-postservern är därför aldrig enbart en DNS-ändring.

## SPF

SPF fastställer vilka system som får skicka i domänens namn. Posten ligger som TXT på själva domänen.

```text
example.com.  IN  TXT  "v=spf1 mx include:spf.provider.example -all"
```

### Uppslagsgränsen

Utvärderingen av en SPF-post får utlösa högst tio DNS-frågande mekanismer. `include`, `a`, `mx`, `ptr`, `exists` och `redirect` räknas, och det rekursivt: varje `include` tar med sig uppslagningarna för den inkluderade posten. `ip4`, `ip6` och `all` räknas inte.

Om gränsen överskrids blir resultatet en `permerror`. För DMARC innebär detta en misslyckad SPF, oavsett om den sändande servern egentligen skulle vara auktoriserad. Det besvärliga är att felet ofta uppstår utan egen medverkan, eftersom en inkluderad leverantör utökar sin post. Den egna posten har inte förändrats, men leveransen försämras ändå.

Dessutom är högst två "void lookups" tillåtna, alltså frågor utan resultat. En `include` till en domän som inte längre finns räknas in här. Hänvisningar till avvecklade tjänsteleverantörer bör därför tas bort, inte lämnas kvar för säkerhets skull.

### Vad som inte hör hemma i en SPF-post

- **`ptr`** är visserligen specificerat, men betraktas sedan RFC 7208 som föråldrat och ska inte användas. Utvärderande system får ignorera det.
- **`+all`** auktoriserar varje godtycklig avsändare och är därmed skadligare än ingen SPF-post alls.
- **`?all`** är neutralt och därmed praktiskt taget värdelöst för DMARC.
- **En separat post av typen SPF (typ 99)** behövs inte längre. Den avskaffades i RFC 7208; SPF finns enbart i TXT.

Mellan `~all` (softfail) och `-all` (hardfail) avgör hur fullständigt sändningsvägarna är registrerade. Så länge det råder tvekan om detta är `~all` rätt val. Den som redan upprätthåller DMARC och utvärderar rapporterna kan gå över till `-all`.

### Underdomäner ärver inget

En SPF-post på `example.com` gäller inte för `newsletter.example.com`. Varje sändande underdomän behöver en egen post. För alla övriga rekommenderas en wildcardpost som klargör att inget kommer därifrån:

```text
*.example.com.  IN  TXT  "v=spf1 -all"
```

Observera: ett TXT-wildcard besvarar även frågor för namn som `_dmarc.sub.example.com`, om det inte finns någon uttrycklig post där. Det är oftast oproblematiskt, men kan förvirra felsökningen eftersom varje TXT-fråga får ett svar.

### SPF-flattening

Verktyg som löser upp alla `include`-hänvisningar och ersätter dem med de bakomliggande IP-adresserna löser uppslagsgränsen på bekostnad av underhållbarheten. Om leverantören ändrar sina adresser avbryts sändningen och ingen märker det, eftersom allt till synes stämmer i den egna posten. Den som väljer denna väg behöver därför en automatiserad jämförelse som regelbundet kontrollerar listan mot källan. Som ett engångsarbete misslyckas förfarandet förr eller senare.

## DKIM

DKIM signerar utgående meddelanden. Den offentliga nyckeln ligger under `<selector>._domainkey.<domain>`.

```text
selector1._domainkey.example.com.  IN  TXT  "v=DKIM1; k=rsa; p=MIIBIjANBg..."
```

Selektorn kan väljas fritt och anges av det sändande systemet. Ett beskrivande namn med datum underlättar senare rotation betydligt mer än `s1` och `s2`.

### Delegering via CNAME

Där sändningstjänsten erbjuder det är CNAME-varianten att föredra framför direkt inmatning:

```text
selector1._domainkey.example.com.  IN  CNAME  selector1.dkim.provider.example.
```

Leverantören kan då rotera sin nyckel självständigt utan att någon behöver arbeta i er zon. Annars blir just denna rotation regelbundet liggande, eftersom den kräver samordning mellan två team. Ett CNAME utesluter dock varje ytterligare post under samma namn; detta är en grundregel i DNS och ingen egenhet hos DKIM.

### Rotation utan avbrott

Vid nyckelbyte publiceras den nya selektorn först, sedan ställs den sändande servern om till den, och först därefter tas den gamla posten bort. Den som raderar den gamla nyckeln omedelbart gör signaturerna för alla meddelanden som fortfarande är på väg eller ligger i köer värdelösa och omöjliggör efterföljande kontroller. Några dagars fördröjning mellan omställning och radering är lämpligt.

En post med tom `p=` är för övrigt inte en defekt post, utan det specificerade sättet att markera en nyckel som återkallad.

### Nyckellängd

1024 bitar betraktas som föråldrat, 2048 bitar är standard. Större RSA-nycklar ger i praktiken ingen ytterligare nytta och ökar bara sannolikheten för att ett mellanliggande system inte behandlar posten korrekt.

## DMARC

DMARC kopplar SPF och DKIM till en instruktion om vad som ska hända vid misslyckad kontroll och skickar tillbaka rapporter. Posten ligger under `_dmarc.<domain>`.

```text
_dmarc.example.com.  IN  TXT  "v=DMARC1; p=none; rua=mailto:dmarc@example.com; sp=none; np=reject; adkim=r; aspf=r"
```

Sedan maj 2026 gäller den reviderade versionen med RFC 9989 samt rapportspecifikationerna RFC 9990 och RFC 9991, som ersätter RFC 7489. Tre ändringar är viktiga i praktiken:

- **`pct` har tagits bort.** Den stegvisa införandet via en procentsats finns inte längre. I stället finns `t=y`, som markerar domänen som testdrift: rapporter fortsätter att skickas, men policyn ska inte upprätthållas.
- **`np` är nytt.** Det anger policyn för icke-existerande underdomäner och stänger därmed en lucka som angripare gärna utnyttjar, eftersom påhittade underdomäner tidigare bara omfattades av `sp`. Om inget eget värde anges följer `np` värdet för `sp`.
- **Public Suffix List har ersatts av en `Tree Walk`.** Organisationsdomänen bestäms inte längre från en externt underhållen lista, utan genom stegvisa DNS-frågor längs namnträdet. För stora namnrymder med många nivåer förändrar detta utvärderingen märkbart.

### Alignment är själva kärnan

DMARC godkänns inte för att SPF eller DKIM tekniskt har godkänts, utan endast när minst en av dem dessutom stämmer överens med den synliga avsändardomänen i `From`-huvudet. SPF kontrolleras mot kuvertavsändardomänen, och den avviker regelbundet vid vidarebefordringar, nyhetsbrevstjänster och ärendesystem. Just därför klarar meddelanden med giltig SPF ibland inte DMARC-kontrollen.

Med `adkim=r` och `aspf=r` (relaxed, standard) räcker överensstämmelse på organisationsdomännivå. `s` kräver exakt överensstämmelse inklusive underdomän och misslyckas i praktiken nästan alltid på någon av sändningsvägarna.

### Externa rapportadresser behöver godkännande

Om rapporter ska skickas till en adress utanför den egna domänen, exempelvis till en DMARC-analystjänst, måste den mottagande domänen auktorisera detta. Utan denna post skickar många mottagare helt enkelt ingenting, och utvärderingen förblir tom trots att allt ser korrekt ut i den egna posten:

```text
example.com._report._dmarc.reports.provider.example.  IN  TXT  "v=DMARC1"
```

Denna post skapas av operatören för målszonen, inte av er. Hos kommersiella tjänster sker detta automatiskt, men inte för en egeninsatt insamlingsbrevlåda i en annan egen domän.

### Typiska syntaxfel

Taggnamn och policyvärden ska skrivas med små bokstäver, `p=Reject` är ogiltigt. Mellan taggarna står ett semikolon; en saknad avgränsare gör resten av raden ogiltig. Och `p` måste stå som första tagg efter `v`. En post som bara består av `v=DMARC1; rua=...` innehåller ingen policy och är ofullständig.

### Utrullningen

`p=none` är ett mättillstånd, inte ett mål. Det förändrar inte hur mottagare hanterar er e-post och tjänar endast till att hitta alla legitima sändningsvägar via rapporterna. Den som efter införandet inte inom några månader går via `quarantine` till `reject` har lagt ner arbetet utan att få skyddet. Den organisatoriska sidan av denna väg, inklusive beslutsunderlag, är ett eget ämne och beskrivs i DMARC Blueprint.

## MTA-STS och TLS-RPT

SMTP krypterar opportunistiskt: om motparten erbjuder STARTTLS krypteras kommunikationen, annars inte. En angripare som kan manipulera trafiken kan ta bort STARTTLS-annonseringen och därmed hålla anslutningen i klartext. MTA-STS stänger denna lucka för mottagande domäner.

MTA-STS består av två delar, och endast en av dem finns i DNS:

```text
_mta-sts.example.com.  IN  TXT    "v=STSv1; id=20260804120000"
mta-sts.example.com.   IN  CNAME  policyhost.example.net.
```

Själva policyn ligger som en fil under `https://mta-sts.example.com/.well-known/mta-sts.txt` och måste levereras via ett giltigt certifikat:

```text
version: STSv1
mode: enforce
mx: mail1.example.com
mx: mail2.example.com
max_age: 604800
```

Felkällorna ligger nästan alla utanför zonen:

- **`id` måste ändras vid varje policyändring.** Den är den enda indikationen för sändande system att en ny policy måste hämtas. Den som anpassar filen och lämnar `id` oförändrad arbetar mot cachelagrade kopior tills `max_age` löper ut.
- **MX-listan i policyn och MX-posterna måste överensstämma.** En ny MX som saknas i policyn avvisas av avsändare med `mode: enforce`. Vid migreringar måste policyn därför anpassas före MX-bytet.
- **`mode: testing` först.** I detta läge rapporteras överträdelser endast, de tvingas inte igenom. Bytet till `enforce` sker när rapporterna är rena.
- **En CAA-post kan blockera certifikatutgivning för policyvärden**, om en annan certifikatutfärdare anges där än den som används.

TLS-RPT levererar tillhörande rapporter och är en enda post:

```text
_smtp._tls.example.com.  IN  TXT  "v=TLSRPTv1; rua=mailto:tlsrpt@example.com"
```

TLS-RPT är meningsfullt även utan MTA-STS eftersom det över huvud taget synliggör misslyckad transportkryptering.

## DANE

DANE uppnår samma mål som MTA-STS, men förankrar förtroendet i DNS i stället för i webb-PKI:n. Det kräver en zon som är DNSSEC-signerad hela vägen, och utan DNSSEC är en TLSA-post verkningslös.

```text
_25._tcp.mail1.example.com.  IN  TLSA  3 1 1 <hash>
```

Avgörande i drift: vid varje certifikatbyte måste TLSA-posten stämma i förväg. Det vanliga förfarandet publicerar den nya hashen parallellt med den gamla, byter sedan certifikat och tar därefter bort den gamla posten. Den som vänder på denna ordning gör e-postservern onåbar för alla DANE-kontrollerande avsändare, däribland de stora tyskspråkiga leverantörerna. I Schweiz förekommer DANE betydligt mer sällan än MTA-STS, vilket oftast beror på att zonen saknar DNSSEC-signering.

## BIMI

BIMI visar varumärkeslogotypen i inkorgen och är den enda mekanism som behandlas här som ännu inte är en RFC, utan fortfarande hanteras som Internet-Draft.

```text
default._bimi.example.com.  IN  TXT  "v=BIMI1; l=https://example.com/logo.svg; a=https://example.com/vmc.pem"
```

Kraven är höga: en upprätthållen DMARC-policy med `quarantine` eller `reject`, en logotyp i formatet SVG Tiny Portable/Secure och för de flesta leverantörer ett avgiftsbelagt Verified Mark Certificate. BIMI är därmed inte en säkerhetsmekanism utan en fråga om synlighet, och den hör sist i ordningen, inte först.

## Andra relevanta poster

**Autodiscover och SRV:** Exchange-miljöer använder `autodiscover.example.com` som CNAME eller en SRV-post `_autodiscover._tcp.example.com`. Båda gäller klientkonfigurationen och inte e-postflödet, men förbises ofta vid migreringar och leder då till profiler som inte längre kan konfigureras.

**CAA:** Har inget direkt med e-post att göra, men avgör vilken certifikatutfärdare som får utfärda ett certifikat för `mta-sts.example.com` eller e-postserverns namn.

**Split-horizon-zoner:** Där en intern DNS-zon har samma namn som den publika saknas e-postposterna ofta internt. Interna system som utför en SPF- eller DKIM-kontroll får då andra resultat än omvärlden. Vid varje ändring av e-postposter bör därför frågan ställas om den interna zonen måste uppdateras.

## Några korta tester

Ställ medvetet alla frågor mot en publik resolver så att inte den interna cachen eller en split-horizon-zon svarar:

```bash
dig @1.1.1.1 +short MX example.com
dig @1.1.1.1 +short TXT example.com
dig @1.1.1.1 +short TXT _dmarc.example.com
dig @1.1.1.1 +short TXT selector1._domainkey.example.com
dig @1.1.1.1 +short TXT _mta-sts.example.com
dig @1.1.1.1 +short TXT _smtp._tls.example.com
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `@1.1.1.1` | Ställer frågan till denna resolver i stället för den som är konfigurerad i `/etc/resolv.conf` |
| `+short` | Visar endast postvärdena, utan rubriker och metadata |
| `MX`, `TXT` | Efterfrågade posttyper |
| `_dmarc.…`, `selector1._domainkey.…`, `_mta-sts.…`, `_smtp._tls.…` | Namnen under domänen som definierats för DMARC, DKIM, MTA-STS och TLS-RPT |

</details>

Mot den auktoritativa servern för att helt kringgå cachelagring:

```bash
dig +short NS example.com
dig @ns1.example.com +norecurse TXT _dmarc.example.com
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `NS example.com` | Hämtar zonens auktoritativa namnservrar |
| `@ns1.example.com` | Ställer följdfrågan direkt till en av dessa auktoritativa servrar |
| `+norecurse` | Sätter inte biten Recursion Desired; servern svarar endast från egna zondata, inte från en cache |

</details>

I Windows utan `dig`:

```text
nslookup -type=TXT _dmarc.example.com 1.1.1.1
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-type=TXT` | Posttyp att fråga efter |
| `_dmarc.example.com` | Efterfrågat namn |
| `1.1.1.1` | Resolver att använda i stället för den systemomfattande konfigurerade |

</details>

För den fullständiga utvärderingen inklusive SPF-uppslagsräkning, DKIM-selektorsökning och alignment-kontroll finns [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check), som kontrollerar en domän i ett svep mot alla poster som beskrivs här.

Det mest betydelsefulla testet är dock ett riktigt meddelande. Skicka ett e-postmeddelande till en brevlåda hos en stor leverantör och titta på raden `Authentication-Results` i huvudet. Den visar på en rad vad SPF, DKIM och DMARC faktiskt gav, och ersätter varje teori om zonfilen.

## Ordning vid en migrering

Vid byte av e-postleverantör har följande ordning visat sig fungera väl:

1. Sänk TTL för alla berörda poster till 300 sekunder, minst en dag i förväg.
2. Publicera den nya leverantörens DKIM-selektorer medan de gamla fortfarande finns kvar.
3. Utöka SPF med den nya leverantören utan att ta bort den gamla, och räkna samtidigt om uppslagsgränsen.
4. Anpassa policyn för MTA-STS till de nya MX-namnen och höj `id` innan MX-posterna byts.
5. Ställ om MX och övervaka leveransen.
6. Ta först efter några dagar utan anmärkningar bort gamla SPF-includes och DKIM-selektorer.
7. Återställ TTL.

Det vanligaste problemet i detta flöde är steg 6 för tidigt: gamla poster raderas tillsammans med omställningen, och allt som fortfarande går via den tidigare vägen misslyckas med avsändarkontrollen.

## Slutsats

E-postposter skiljer sig från alla andra DNS-poster genom att ett fel inte nödvändigtvis märks. En felaktig A-post leder till ett ärende inom några minuter, medan en dubbel SPF-post eller en DKIM-nyckel med ett tecken för mycket i stället leder till en leveransgrad som sakta sjunker under veckor.

Tre regler förhindrar de flesta av dessa fall. För det första: kontrollera före varje ny post vad som redan finns, i stället för att lägga en andra bredvid. För det andra: kontrollera efter varje ändring mot en publik resolver och jämför värdet tecken för tecken med mallen, inte bara visuellt. För det tredje: publicera alltid det nya först vid ändringar, växla sedan och ta därefter bort det gamla. Den som följer denna ordning har alltid en återväg för e-postposter.

## Ändringshistorik

- **25 oktober 2026:** Källänken uppdaterad till Microsofts nya adress.

## Källor

1.  [RFC 1035: Domain Names, Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035): Definierar bland annat 255-byte-gränsen för en enskild `character-string` i TXT-poster.

2.  [RFC 2181: Clarifications to the DNS Specification](https://www.rfc-editor.org/rfc/rfc2181): Anger i avsnitt 10.3 att målet för en MX-post inte får vara ett alias.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Uppslagsgräns på tio mekanismer, gräns för void lookups, avskaffande av RR-typen SPF och avrådan från `ptr`-mekanismen.

4.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): Uppbyggnad av nyckelposten under `_domainkey`, selektorns betydelse och det tomma `p=`.

5.  [RFC 9989: Domain-Based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc9989): Aktuell DMARC-specifikation från maj 2026, ersätter RFC 7489; bortfall av `pct`, ny tagg `np`, Tree Walk i stället för Public Suffix List.

6.  [RFC 9990: DMARC Aggregate Reporting](https://www.rfc-editor.org/rfc/rfc9990): Format och leverans av aggregerade rapporter, inklusive auktorisering av externa mottagardomäner.

7.  [RFC 7505: A "Null MX" No Service Resource Record for Domains That Accept No Mail](https://www.rfc-editor.org/rfc/rfc7505): Märkning av domäner som inte tar emot e-post.

8.  [RFC 8461: SMTP MTA Strict Transport Security (MTA-STS)](https://www.rfc-editor.org/rfc/rfc8461): DNS-post, policyfil, betydelsen av `id` och lägena `testing` och `enforce`.

9.  [RFC 8460: SMTP TLS Reporting](https://www.rfc-editor.org/rfc/rfc8460): Uppbyggnad av `_smtp._tls`-posten och rapporterna.

10.  [RFC 7672: SMTP Security via Opportunistic DNS-Based Authentication of Named Entities (DANE)](https://www.rfc-editor.org/rfc/rfc7672): TLSA-poster för SMTP och förutsättningen om en DNSSEC-signerad zon.

11.  [Brand Indicators for Message Identification (BIMI), Internet-Draft](https://datatracker.ietf.org/doc/draft-brand-indicators-for-message-identification/): Aktuellt läge för BIMI-specifikationen, fortfarande ingen RFC.

12.  [Google: Riktlinjer för e-postavsändare](https://support.google.com/a/answer/81126): Krav på avsändare, bland annat PTR-krav för sändande IPv6-adresser och kraven för massavsändare som gäller sedan februari 2024.

13.  [Microsoft: Strengthening Email Ecosystem, Outlook's New Requirements for High-Volume Senders](https://techcommunity.microsoft.com/blog/microsoftdefenderforoffice365blog/strengthening-email-ecosystem-outlook%E2%80%99s-new-requirements-for-high%E2%80%90volume-senders/4399730): Krav för avsändare från 5000 meddelanden per dag, giltiga sedan maj 2025.
