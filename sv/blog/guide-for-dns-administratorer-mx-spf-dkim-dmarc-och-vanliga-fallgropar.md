---
title: "Guide för DNS-administratörer: MX, SPF, DKIM, DMARC och vanliga felkällor"
navTitle: "DNS-poster för e-post"
description: "Den som ansvarar för en zon får oftast färdiga e-postposter och ska bara publicera dem. Det som regelbundet går fel: 255-bytegränsen för DKIM, dubbla SPF-poster, uppslagsgränsen, MX på en CNAME, det automatiskt tillagda zon-suffixet och policyer som ingen längre upprätthåller."
date: "2026-08-04"
kategorie: "SMTP och e-postflöde"
timeToRead: "15 min lästid"
themen:
  - smtp-mailflow
  - e-mail-verschluesselung
produkte:
  - "uebergreifend"
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
aiPrompt: |
  Du bist mein Assistent für DNS-Records rund um E-Mail. Ich gebe dir einen Record-Wert oder eine Zonendatei, du prüfst sie gegen die Regeln aus diesem Artikel: Syntax, doppelte Records, SPF-Lookup-Limit und Void-Lookups, DKIM-Base64 auf Copy-Paste-Schäden, DMARC-Tags nach RFC 9989 inklusive sp und np, externe Report-Adressen mit Autorisierungsrecord, MX ohne CNAME-Ziel, MTA-STS-ID. Frage mich zuerst: 1. um welche Domain und welchen Record es geht, 2. ob die Domain sendet, empfängt oder beides, 3. welche Versanddienste beteiligt sind (Marketing, ERP, Ticketsystem, Scan-to-Mail), 4. welches DNS-System die Zone hält. Gib mir am Ende den korrigierten Record als kopierfertige Zeile plus die dig-Befehle zur Kontrolle.
translationOf: dns-records-e-mail-stolpersteine
translationSourceHash: 63c8a888f2ebd4548bd4222c4273896228649bf02f0406082ec337194af65280
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:09:21.458Z
translationReview: required
url: https://rafaelpfister.ch/sv/blog/guide-for-dns-administratorer-mx-spf-dkim-dmarc-och-vanliga-fallgropar
---

# Guide för DNS-administratörer: MX, SPF, DKIM, DMARC och vanliga felkällor

Den som ansvarar för en DNS-zon får sällan e-postposter levererade som hen själv har skrivit. E-postteamet, en leverantör eller en marknadsföringstjänst skickar en rad med upplysningen att den ”bara behöver publiceras”. Just detta orsakar de flesta felen, eftersom e-postposter är den typ av post där ett skrivfel kan få två helt olika konsekvenser. Antingen bryts leveransen omedelbart och någon hör av sig inom några minuter, eller så fortsätter den oförändrat och endast avsändarkontrollen faller tyst bort. Det andra fallet förblir regelbundet oupptäckt i månader, tills en stor mottagare sätter domänen i karantän.

Sedan Google och Yahoo skärpte sina krav för massavsändare i februari 2024 och Microsoft följde efter i maj 2025 har toleransen för halvt konfigurerade domäner blivit liten. SPF, DKIM och en DMARC-post är för avsändare över en viss volym inte längre en bonus utan en förutsättning för leverans.

Alla exempel i denna artikel använder `example.com` och generiska selektorer. De visade värdena är förkortade för att förbli läsbara.

## Regler som gäller för varje e-postpost

### 255-bytegränsen för TXT-poster

En TXT-post består enligt RFC 1035 av en eller flera `character-strings`, och varje sådan teckensträng rymmer högst 255 byte. Posten som helhet får vara längre, men måste då delas upp i flera teckensträngar. System som utvärderar den fogar sedan ihop dessa delar utan skiljetecken.

I praktiken blir detta relevant på exakt ett ställe: DKIM-nycklar med 2048 bitar. Deras Base64-värde är omkring 400 tecken långt och ryms inte i en teckensträng.

```text
selector1._domainkey.example.com. 3600 IN TXT (
    "v=DKIM1; k=rsa; "
    "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA0ZenWBnGUqydzg5w"
    "yWxRRNBZjbagzDh1BlW3b145Wer/GWfbz6XkCyqsN918N+/Va6mVe37rXNaZAS"
    "/js/L3m7d2OlUp5I3jHC5EsU6XwU5trKFxPWErLxtanYbXLXabyVIRGkop+1s3"
    "SXNg2Oy5eNyZf5MGlEAo+JM6oXtgkABQQn5kE1ShzalXJUVL/wIDAQAB" )
```

De flesta DNS-hanteringssystem utför denna uppdelning själva när värdet anges i det vanliga inmatningsfältet. Den som i stället sätter citationstecken manuellt måste hålla gränsen exakt. Ett radbrutet värde med ett mellanslag i skarven ger en nyckel som syntaktiskt finns men kryptografiskt inte längre stämmer.

Kontrollen efteråt är viktig, eftersom en felaktigt sammansatt nyckel ser helt oansenlig ut i GUI:t:

```bash
dig +short TXT selector1._domainkey.example.com | tr -d '" ' | wc -c
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `+short` | Visar endast postvärdena, utan rubriker och metadata |
| `TXT selector1._domainkey.example.com` | Posttyp och namn för DKIM-nyckelposten |
| `tr -d '" '` | Tar bort citationstecken och mellanslag, och sätter därmed ihop delsträngarna så som en verifierare läser dem |
| `wc -c` | Räknar tecknen i det sammansatta värdet; längden måste stämma överens med mallen |

</details>

### En post per syfte

SPF och DMARC är definierade så att exakt en passande post får finnas för ett namn. För SPF leder två `v=spf1`-poster till ett `permerror`, och kontrollen anses därmed misslyckad, inte godkänd. För DMARC ignorerar mottagare domänen helt om flera poster börjar med `v=DMARC1`: i stället för en strikt policy tillämpas då ingen alls.

Detta är det överlägset vanligaste felet i zoner som vuxit över tid. En ny tjänsteleverantör ansluts, någon lägger till ”sin” SPF-post i stället för att utöka den befintliga, och från den stunden misslyckas kontrollen för samtliga avsändare. Kontrollera därför alltid vad som redan finns innan du lägger till en ny post:

```bash
dig +short TXT example.com | grep -i spf1
dig +short TXT _dmarc.example.com
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `+short` | Visar endast postvärdena, utan rubriker och metadata |
| `TXT` | Efterfrågad posttyp |
| `example.com`, `_dmarc.example.com` | Efterfrågade namn: själva domänen för SPF, namnet `_dmarc` för DMARC |
| `grep -i spf1` | Filtrerar fram SPF-raden; `-i` ignorerar stora och små bokstäver |

</details>

För DKIM gäller motsatsen: där är en post per selektor avsedd, och flera selektorer sida vid sida är det normala eftersom varje sändningstjänst har sin egen nyckel.

### Zon-suffixet i webbgränssnitt

I Infoblox, Windows DNS och i praktiskt taget alla webbgränssnitt för hosting läggs zonnamnet automatiskt till det angivna namnet. Den som anger det fullständigt kvalificerade namnet i fältet ”Namn” får en post som är dubbelt så lång som avsett:

```text
Eingabe:   selector1._domainkey.example.com
Ergebnis:  selector1._domainkey.example.com.example.com
```

I zonfilen är motsvarigheten den saknade avslutande punkten. `mail.example.com` utan punkt i slutet är ett relativt namn och kompletteras med zonnamnet, medan `mail.example.com.` med punkt är absolut. För MX- och CNAME-mål avgör denna enda punkt om domänen går att nå.

### Copy-paste är den vanligaste felkällan

Värden för e-postposter skrivs nästan aldrig in, utan kopieras från en PDF, ett ärende, en Excel-cell eller en chatt. Då uppstår skador som förblir osynliga i inmatningsfältet:

- Ett dubbelt `p=` i början av DKIM-nyckeln, eftersom prefixet sattes två gånger vid sammanfogningen. Värdet `v=DKIM1;k=rsa;p=p=MIIBIjAN...` förekommer regelbundet i praktiken och ger en oanvändbar nyckel.
- Typografiska citationstecken från Word i stället för raka.
- Hårda mellanslag från PDF-layouter som ser ut som vanliga.
- Radbrytningar mitt i Base64-blocket när värdet i PDF:en löpte över flera rader.

Base64 känner endast tecknen A till Z, a till z, 0 till 9, `+`, `/` och `=` som utfyllnadstecken. Allt annat i delen `p=` är ett fel. Ett kort filter före inmatning sparar senare felsökning:

```bash
printf '%s' "$KEY" | tr -d 'A-Za-z0-9+/=' | wc -c
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `'%s' "$KEY"` | Skriver ut nyckelvärdet oförändrat och utan tillagd radbrytning |
| `tr -d 'A-Za-z0-9+/='` | Tar bort alla Base64-giltiga tecken; endast främmande tecken återstår |
| `wc -c` | Räknar de återstående tecknen |

</details>

Om resultatet här är något annat än `0` innehåller nyckeln främmande tecken.

### Sänk TTL före ändringar

Före varje planerat byte av en MX-, SPF- eller DKIM-post bör TTL sänkas till ett lågt värde under några timmar, vanligtvis 300 sekunder. Annars kan det gamla värdet ligga kvar i externa resolvrar i en dag eller längre, beroende på zon, och en återställning tar lika lång tid. Efter ändringen och en observationsperiod sätts TTL tillbaka till det ordinarie värdet.

## MX

MX-posten fastställer vilken värd som tar emot e-post för domänen. Det finns två regler som regelbundet överträds.

**Målet måste vara ett värdnamn med A- eller AAAA-post.** Varken en IP-adress eller en CNAME är tillåten. RFC 2181 fastslår uttryckligen att målet för en MX-post inte får vara ett alias. I praktiken fungerar det ändå hos många mottagare, men inte hos andra, vilket leder till felbilder som till synes bara berör enskilda avsändare.

```text
example.com.        IN  MX  10 mail1.example.com.
example.com.        IN  MX  20 mail2.example.com.
mail1.example.com.  IN  A   192.0.2.10
mail2.example.com.  IN  A   192.0.2.11
```

**Talet är en preferens, inte en viktning.** Det lägre värdet provas först. En andra MX med ett högt tal är bara meningsfull om systemet känner till samma mottagarfilter. Backup-MX-poster på system utan mottagarkontroll är ett populärt mål för skräppost, eftersom angripare medvetet styr mot den svagaste posten.

Domäner som endast skickar eller inte har något med e-post att göra får en Null MX enligt RFC 7505. Den signalerar att domänen inte tar emot e-post och ger ett omedelbart och tydligt avslag i stället för timeout:

```text
example.com.  IN  MX  0 .
```

Null MX ersätter dock inte en SPF- eller DMARC-post. Att inte ta emot betyder inte att ingen skickar i ditt namn. Särskilt parkerade underdomäner används för spoofing eftersom få personer tittar till dem.

## A, AAAA, PTR och HELO-namnet

PTR-posten för den utgående IP-adressen ligger inte i din zon, utan i leverantörens `in-addr.arpa`-zon, som äger adressblocket. Den beställs därför hos leverantören och sätts inte av dig själv. Många stora mottagare kräver att PTR och den tillhörande framåtriktade posten stämmer överens, alltså att namnet i PTR åter löses upp till samma IP-adress.

```bash
dig +short -x 192.0.2.10
dig +short A mail1.example.com
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `+short` | Visar endast postvärdena, utan rubriker och metadata |
| `-x 192.0.2.10` | Omvänd uppslagning: dig skapar själv PTR-namnet i zonen `in-addr.arpa` |
| `A mail1.example.com` | Framåtriktad uppslagning av namnet från PTR, för att kontrollera att rundgången leder till samma IP-adress |

</details>

Namnet som din e-postserver anger i HELO eller EHLO bör vara detsamma och också kunna lösas upp. En gateway som identifierar sig som `localhost.localdomain` eller med ett internt namn bedöms sämre av större mottagare.

Var försiktig när du lägger till en AAAA-post. Så snart e-postservern blir tillgänglig och sändande över IPv6 gäller samma krav som för IPv4, delvis ännu strängare. Google kräver en giltig PTR för sändande IPv6-adresser. Saknas den avvisas sändningen, medan den fungerade felfritt över IPv4. En AAAA-post för e-postservern är därför aldrig enbart en DNS-ändring.

## SPF

SPF fastställer vilka system som får skicka i domänens namn. Posten ligger som TXT på själva domänen.

```text
example.com.  IN  TXT  "v=spf1 mx include:spf.provider.example -all"
```

### Uppslagsgränsen

Utvärderingen av en SPF-post får högst utlösa tio DNS-frågande mekanismer. `include`, `a`, `mx`, `ptr`, `exists` och `redirect` räknas, och det rekursivt: varje `include` tar med sig uppslagningarna från den inkluderade posten. `ip4`, `ip6` och `all` räknas inte.

Om gränsen överskrids blir resultatet ett `permerror`. För DMARC innebär det att SPF inte godkänns, oavsett om den sändande servern egentligen skulle vara auktoriserad. Det besvärliga är att felet ofta uppstår utan egen medverkan, eftersom en inkluderad leverantör utökar sin post. Den egna posten har inte ändrats, men leveransen försämras ändå.

Dessutom är högst två ”void lookups” tillåtna, alltså frågor utan resultat. Ett `include` till en domän som inte längre finns räknas in här. Hänvisningar till avvecklade tjänsteleverantörer ska därför tas bort och inte lämnas kvar för säkerhets skull.

### Vad som inte hör hemma i en SPF-post

- **`ptr`** är visserligen specificerat men betraktas som föråldrat sedan RFC 7208 och ska inte användas. Utvärderande system får ignorera det.
- **`+all`** auktoriserar valfri avsändare och är därmed skadligare än att inte ha någon SPF-post alls.
- **`?all`** är neutralt och därmed praktiskt taget värdelöst för DMARC.
- **En separat post av typen SPF (typ 99)** behövs inte längre. Den är avskaffad sedan RFC 7208; SPF finns endast i TXT.

Mellan `~all` (softfail) och `-all` (hardfail) avgör hur fullständigt sändningsvägarna är kartlagda. Så länge det råder tvivel om detta är `~all` rätt val. Den som redan upprätthåller DMARC och utvärderar rapporterna kan gå över till `-all`.

### Underdomäner ärver inget

En SPF-post på `example.com` gäller inte för `newsletter.example.com`. Varje sändande underdomän behöver en egen post. För alla övriga rekommenderas en wildcard-post som klargör att inget skickas därifrån:

```text
*.example.com.  IN  TXT  "v=spf1 -all"
```

Observera: ett TXT-wildcard besvarar även frågor för namn som `_dmarc.sub.example.com`, såvida ingen explicit post finns där. Det är oftast oproblematiskt, men kan förvirra felsökningen eftersom varje TXT-fråga får ett svar.

### SPF-flattening

Verktyg som löser upp alla `include`-referenser och ersätter dem med bakomliggande IP-adresser löser uppslagsgränsen på bekostnad av underhållbarheten. Om leverantören ändrar sina adresser bryts sändningen, och ingen märker det eftersom allt till synes stämmer i den egna posten. Den som väljer denna väg behöver därför en automatiserad avstämning som regelbundet kontrollerar listan mot källan. Som engångsarbete misslyckas metoden förr eller senare.

## DKIM

DKIM signerar utgående meddelanden. Den offentliga nyckeln finns under `<selector>._domainkey.<domain>`.

```text
selector1._domainkey.example.com.  IN  TXT  "v=DKIM1; k=rsa; p=MIIBIjANBg..."
```

Selektorn kan väljas fritt och anges av det sändande systemet. Ett beskrivande namn med datum underlättar senare rotation betydligt mer än `s1` och `s2`.

### Delegering med CNAME

Där sändningstjänsten erbjuder det bör CNAME-varianten föredras framför direkt inmatning:

```text
selector1._domainkey.example.com.  IN  CNAME  selector1.dkim.provider.example.
```

Leverantören kan då rotera sin nyckel självständigt utan att någon behöver arbeta i din zon. Annars blir just denna rotation regelbundet liggande, eftersom den kräver samordning mellan två team. En CNAME utesluter dock varje ytterligare post med samma namn; det är en grundregel i DNS och inte något särskilt för DKIM.

### Rotation utan avbrott

Vid nyckelbyte publiceras den nya selektorn först, sedan ställs den sändande servern om till den och först därefter tas den gamla posten bort. Den som tar bort den gamla nyckeln omedelbart ogiltigförklarar signaturerna på alla meddelanden som fortfarande är på väg eller ligger i köer och omöjliggör efterföljande kontroller. Några dagars framförhållning mellan omställning och borttagning är lämpligt.

En post med tom `p=` är för övrigt inte en defekt post, utan det specificerade sättet att markera en nyckel som återkallad.

### Nyckellängd

1024 bitar anses föråldrat, 2048 bitar är standard. Större RSA-nycklar ger i praktiken ingen ytterligare nytta och ökar endast sannolikheten för att ett mellanliggande system inte behandlar posten korrekt.

## DMARC

DMARC kopplar samman SPF och DKIM med en instruktion om vad som ska ske vid en misslyckad kontroll och levererar rapporter tillbaka. Posten finns under `_dmarc.<domain>`.

```text
_dmarc.example.com.  IN  TXT  "v=DMARC1; p=none; rua=mailto:dmarc@example.com; sp=none; np=reject; adkim=r; aspf=r"
```

Sedan maj 2026 gäller den reviderade versionen med RFC 9989 samt rapportspecifikationerna RFC 9990 och RFC 9991, som ersätter RFC 7489. Tre ändringar är viktiga i praktiken:

- **`pct` har tagits bort.** Den stegvisa införandet via en procentsats finns inte längre. I stället finns `t=y`, som markerar domänen som i testdrift: rapporterna fortsätter, men policyn ska inte upprätthållas.
- **`np` är nytt.** Det anger policyn för icke-existerande underdomäner och täpper därmed till en lucka som angripare gärna utnyttjar, eftersom påhittade underdomäner tidigare endast omfattades av `sp`. Utan egen uppgift följer `np` värdet för `sp`.
- **Public Suffix List har ersatts av en `Tree Walk`.** Organisationsdomänen bestäms inte längre från en externt underhållen lista, utan genom stegvisa DNS-frågor längs namnträdet. För stora namnrymder med många nivåer förändrar detta utvärderingen märkbart.

### Alignment är själva kärnan

DMARC godkänns inte för att SPF eller DKIM tekniskt har godkänts, utan endast om minst ett av dem dessutom stämmer överens med den synliga avsändardomänen i rubriken `From`. SPF kontrolleras mot domänen för kuvertavsändaren, och den avviker regelbundet vid vidarebefordran, nyhetsbrevstjänster och ärendehanteringssystem. Därför klarar meddelanden med giltig SPF ibland inte DMARC-kontrollen.

Med `adkim=r` och `aspf=r` (relaxed, standardinställningen) räcker överensstämmelse på organisationsdomännivå. `s` kräver exakt överensstämmelse inklusive underdomän och misslyckas i praktiken nästan alltid för någon av sändningsvägarna.

### Externa rapportadresser behöver godkännande

Om rapporter ska skickas till en adress utanför den egna domänen, exempelvis till en DMARC-analystjänst, måste den mottagande domänen auktorisera detta. Utan denna post skickar många mottagare helt enkelt ingenting, och utvärderingen förblir tom medan allt ser korrekt ut i den egna posten:

```text
example.com._report._dmarc.reports.provider.example.  IN  TXT  "v=DMARC1"
```

Denna post skapas av operatören för målszonen, inte av dig. Hos kommersiella tjänster sker det automatiskt, men inte för en egeninsatt insamlingsbrevlåda i en annan egen domän.

### Typiska syntaxfel

Taggnamn och policyvärden ska skrivas med små bokstäver, `p=Reject` är ogiltigt. Mellan taggarna står ett semikolon; ett saknat avgränsningstecken gör resten av raden ogiltig. Och `p` måste stå som första tagg efter `v`. En post som endast består av `v=DMARC1; rua=...` innehåller ingen policy och är ofullständig.

### Utrullningen

`p=none` är ett mättillstånd, inte ett mål. Det ändrar ingenting i mottagarnas hantering av dina e-postmeddelanden och tjänar endast till att via rapporterna hitta alla legitima sändningsvägar. Den som efter införandet inte inom några månader går från `quarantine` till `reject` har lagt ner arbetet utan att få skyddet. Den organisatoriska sidan av denna väg, inklusive beslutsunderlag, är ett eget ämne och beskrivs i DMARC-blueprinten.

## MTA-STS och TLS-RPT

SMTP krypterar opportunistiskt: om motparten erbjuder STARTTLS krypteras anslutningen, annars inte. En angripare som kan manipulera trafiken kan ta bort STARTTLS-annonseringen och därmed hålla anslutningen okrypterad. MTA-STS täpper till denna lucka för mottagande domäner.

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

- **`id` måste ändras vid varje policyändring.** Det är den enda indikationen för sändande system på att en ny policy ska hämtas. Den som ändrar filen och låter `id` stå kvar arbetar mot cachade kopior tills `max_age` löper ut.
- **MX-listan i policyn och MX-posterna måste stämma överens.** En ny MX som saknas i policyn avvisas av avsändare med `mode: enforce`. Vid migreringar ska policyn därför anpassas före MX-bytet.
- **`mode: testing` först.** I detta läge rapporteras överträdelser endast, de upprätthålls inte. Bytet till `enforce` sker när rapporterna är korrekta.
- **En CAA-post kan blockera certifikatutfärdandet för policyvärden**, om en annan certifikatutfärdare är angiven där än den som används.

TLS-RPT levererar de tillhörande rapporterna och består av en enda post:

```text
_smtp._tls.example.com.  IN  TXT  "v=TLSRPTv1; rua=mailto:tlsrpt@example.com"
```

TLS-RPT är meningsfullt även utan MTA-STS, eftersom det över huvud taget synliggör misslyckad transportkryptering.

## DANE

DANE uppnår samma mål som MTA-STS, men förankrar förtroendet i DNS i stället för i webb-PKI:n. Det förutsätter en genomgående DNSSEC-signerad zon, och utan DNSSEC är en TLSA-post verkningslös.

```text
_25._tcp.mail1.example.com.  IN  TLSA  3 1 1 <hash>
```

Avgörande i drift: vid varje certifikatbyte måste TLSA-posten stämma i förväg. Den vanliga metoden publicerar den nya hashen parallellt med den gamla, byter sedan certifikatet och tar därefter bort den gamla posten. Den som vänder på denna ordning gör e-postservern otillgänglig för alla DANE-kontrollerande avsändare, däribland de stora tyskspråkiga leverantörerna. I Schweiz är DANE betydligt ovanligare än MTA-STS, vilket oftast beror på att zonen saknar DNSSEC-signering.

## BIMI

BIMI visar varumärkeslogotypen i inkorgen och är den enda mekanism som behandlas här och som ännu inte är en RFC, utan fortfarande hanteras som ett Internet-Draft.

```text
default._bimi.example.com.  IN  TXT  "v=BIMI1; l=https://example.com/logo.svg; a=https://example.com/vmc.pem"
```

Kraven är höga: en upprätthållen DMARC-policy med `quarantine` eller `reject`, en logotyp i formatet SVG Tiny Portable/Secure och för de flesta leverantörer ett avgiftsbelagt Verified Mark Certificate. BIMI är därmed inte en säkerhetsmekanism utan en fråga om synlighet, och hör hemma sist i ordningen, inte först.

## Ytterligare poster i sammanhanget

**Autodiscover och SRV:** Exchange-miljöer använder `autodiscover.example.com` som CNAME eller en SRV-post `_autodiscover._tcp.example.com`. Båda avser klientkonfiguration och inte e-postflödet, men förbises gärna vid migreringar och leder då till profiler som inte längre kan konfigureras.

**CAA:** Har inget direkt med e-post att göra, men avgör vilken certifikatutfärdare som får utfärda ett certifikat för `mta-sts.example.com` eller e-postserverns namn.

**Split-horizon-zoner:** Där en intern DNS-zon har samma namn som den offentliga saknas e-postposterna ofta internt. Interna system som utför en SPF- eller DKIM-kontroll kommer då fram till andra resultat än omvärlden. Vid varje ändring av e-postposter bör man därför fråga sig om den interna zonen behöver uppdateras.

## Några korta tester

Utför medvetet alla frågor mot en offentlig resolver, så att inte den interna cachen eller en split-horizon-zon svarar:

```bash
dig @1.1.1.1 +short MX example.com
dig @1.1.1.1 +short TXT example.com
dig @1.1.1.1 +short TXT _dmarc.example.com
dig @1.1.1.1 +short TXT selector1._domainkey.example.com
dig @1.1.1.1 +short TXT _mta-sts.example.com
dig @1.1.1.1 +short TXT _smtp._tls.example.com
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `@1.1.1.1` | Skickar frågan till denna resolver i stället för den som konfigurerats i `/etc/resolv.conf` |
| `+short` | Visar endast postvärdena, utan rubriker och metadata |
| `MX`, `TXT` | Efterfrågade posttyper |
| `_dmarc.…`, `selector1._domainkey.…`, `_mta-sts.…`, `_smtp._tls.…` | Namnen under domänen som definierats för DMARC, DKIM, MTA-STS och TLS-RPT |

</details>

Mot den auktoritativa servern, för att helt kringgå caching:

```bash
dig +short NS example.com
dig @ns1.example.com +norecurse TXT _dmarc.example.com
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `NS example.com` | Tar fram zonens auktoritativa namnservrar |
| `@ns1.example.com` | Skickar följdfrågan direkt till en av dessa auktoritativa servrar |
| `+norecurse` | Sätter inte biten Recursion Desired; servern svarar endast från sina egna zondata, inte från en cache |

</details>

I Windows utan `dig`:

```text
nslookup -type=TXT _dmarc.example.com 1.1.1.1
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-type=TXT` | Posttyp som ska efterfrågas |
| `_dmarc.example.com` | Efterfrågat namn |
| `1.1.1.1` | Resolver som ska användas i stället för den systemomfattande konfigurerade |

</details>

För en fullständig utvärdering, inklusive SPF-räkning av uppslagningar, sökning efter DKIM-selektorer och alignment-kontroll, finns [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check), som kontrollerar en domän i en och samma körning mot alla poster som beskrivs här.

Det mest träffsäkra testet är dock ett verkligt meddelande. Skicka ett e-postmeddelande till en brevlåda hos en stor leverantör och titta på raden `Authentication-Results` i huvudet. Den visar på en rad vad SPF, DKIM och DMARC faktiskt har gett, och ersätter all teori om zonfilen.

## Ordning vid en migrering

Vid byte av e-postleverantör har följande ordning visat sig fungera väl:

1. Sänk TTL för alla berörda poster till 300 sekunder, minst en dag i förväg.
2. Publicera den nya leverantörens DKIM-selektorer medan de gamla fortfarande finns kvar.
3. Utöka SPF med den nya leverantören utan att ta bort den gamla, och räkna samtidigt om uppslagsgränsen.
4. Anpassa policyn till de nya MX-namnen för MTA-STS och höj `id` innan MX-posterna byts.
5. Byt MX och övervaka leveransen.
6. Ta först efter några dagar utan anmärkningar bort gamla SPF-includes och DKIM-selektorer.
7. Återställ TTL.

Det vanligaste problemet i detta förlopp är att steg 6 sker för tidigt: gamla poster tas bort tillsammans med omställningen, och allt som fortfarande går via den tidigare vägen misslyckas med avsändarkontrollen.

## Slutsats

E-postposter skiljer sig från alla andra DNS-poster genom att ett fel inte nödvändigtvis märks. En felaktig A-post leder till ett ärende inom några minuter, medan en dubbel SPF-post eller en DKIM-nyckel med ett tecken för mycket i stället leder till en leveransgrad som långsamt sjunker under veckor.

Tre regler förhindrar de flesta sådana fall. För det första: kontrollera före varje ny post vad som redan finns, i stället för att lägga en andra bredvid. För det andra: kontrollera efter varje ändring mot en offentlig resolver och jämför värdet tecken för tecken med mallen, inte bara visuellt. För det tredje: publicera alltid det nya först vid omställningar, växla sedan över och ta därefter bort det gamla. Den som följer denna ordning har alltid en väg tillbaka för e-postposter.

## Källor

1.  [RFC 1035: Domain Names, Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035): Definierar bland annat 255-bytegränsen för en enskild `character-string` i TXT-poster.

2.  [RFC 2181: Clarifications to the DNS Specification](https://www.rfc-editor.org/rfc/rfc2181): Fastslår i avsnitt 10.3 att målet för en MX-post inte får vara ett alias.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Uppslagsgräns på tio mekanismer, gräns för void lookups, avskaffande av RR-typen SPF och avrådan från mekanismen `ptr`.

4.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): Uppbyggnad av nyckelposten under `_domainkey`, selektorns betydelse och det tomma `p=`.

5.  [RFC 9989: Domain-Based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc9989): Aktuell DMARC-specifikation från maj 2026, ersätter RFC 7489; borttagning av `pct`, ny tagg `np`, Tree Walk i stället för Public Suffix List.

6.  [RFC 9990: DMARC Aggregate Reporting](https://www.rfc-editor.org/rfc/rfc9990): Format och leverans av aggregerade rapporter, inklusive auktorisering av externa mottagardomäner.

7.  [RFC 7505: A "Null MX" No Service Resource Record for Domains That Accept No Mail](https://www.rfc-editor.org/rfc/rfc7505): Märkning av domäner som inte tar emot e-post.

8.  [RFC 8461: SMTP MTA Strict Transport Security (MTA-STS)](https://www.rfc-editor.org/rfc/rfc8461): DNS-post, policyfil, betydelsen av `id` och lägena `testing` och `enforce`.

9.  [RFC 8460: SMTP TLS Reporting](https://www.rfc-editor.org/rfc/rfc8460): Uppbyggnad av posten `_smtp._tls` och rapporterna.

10.  [RFC 7672: SMTP Security via Opportunistic DNS-Based Authentication of Named Entities (DANE)](https://www.rfc-editor.org/rfc/rfc7672): TLSA-poster för SMTP och kravet på en DNSSEC-signerad zon.

11.  [Brand Indicators for Message Identification (BIMI), Internet-Draft](https://datatracker.ietf.org/doc/draft-brand-indicators-for-message-identification/): Aktuell status för BIMI-specifikationen, fortfarande ingen RFC.

12.  [Google: Riktlinjer för e-postavsändare](https://support.google.com/a/answer/81126): Krav på avsändare, bland annat krav på PTR för sändande IPv6-adresser och reglerna för massavsändare som gäller sedan februari 2024.

13.  [Microsoft: Strengthening Email Ecosystem, Outlook's New Requirements for High-Volume Senders](https://techcommunity.microsoft.com/blog/microsoft-defender-for-office365-blog/strengthening-email-ecosystem-outlook%E2%80%99s-new-requirements-for-high%E2%80%90volume-senders/4399008): Krav för avsändare från 5 000 meddelanden per dag, giltiga sedan maj 2025.
