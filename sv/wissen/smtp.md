---
title: "SMTP: transport, kö och leveransansvar"
blatt: "smtp"
description: "Simple Mail Transfer Protocol ur administratörens perspektiv: rollerna för MUA, MSA, MTA och MDA, envelope och meddelandeinnehåll, SMTP-tillståndsmaskin, acceptansgräns, MX-routning, kö och återförsök, svarskoder, ESMTP-utökningar, submission, TLS-policy och skiktad diagnostik."
fakten:
  - label: Fullständigt namn
    wert: Simple Mail Transfer Protocol
    href: https://datatracker.ietf.org/doc/html/rfc5321
  - label: Grundmodell
    wert: Command/Reply och Store-and-Forward
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-2.3.1
  - label: Transportobjekt
    wert: Envelope plus meddelandeinnehåll
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-2.3.1
  - label: Transport
    wert: tillförlitlig, ordnad dataström; på internet TCP
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-1.1
  - label: Server-till-server
    wert: SMTP över TCP 25
    href: https://datatracker.ietf.org/doc/html/rfc6409#section-1
  - label: Submission
    wert: 587 med STARTTLS · 465 med implicit TLS
    href: https://datatracker.ietf.org/doc/html/rfc8314#section-3
  - label: Kärnroller
    wert: MUA · MSA · MTA · MDA respektive LMTP
    href: https://datatracker.ietf.org/doc/html/rfc5598#section-4
  - label: Funktioner
    wert: EHLO förhandlar ESMTP-utökningar per session
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-2.2
  - label: Acceptansgräns
    wert: 250 efter End-of-Data överför ansvaret
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4
  - label: Felklasser
    wert: 4xx tillfälligt · 5xx permanent · x.y.z som detalj
    href: https://datatracker.ietf.org/doc/html/rfc3463#section-2
  - label: Beroenden
    wert: DNS · TCP · TLS · SASL · MIME · beständig kö
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-2.3
  - label: Normhistorik
    wert: RFC 821 → RFC 2821 → RFC 5321
    href: https://datatracker.ietf.org/doc/html/rfc5321#appendix-C
werbung:
  - tools
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: bd51241025647a47c1c52f82747a0f7eee7d7b8e42d57f52057e7123d6bc3825
translationModel: gpt-5.6-terra
translatedAt: 2026-10-13T11:17:42.283Z
translationReview: required
---

# SMTP: transport, kö och leveransansvar

**Simple Mail Transfer Protocol** (SMTP) för ett meddelande från en e-postprocess till nästa. Klienten ansluter till en server, frågar efter dess funktioner, anger envelope-avsändare och mottagare och överför först därefter meddelandeinnehållet. Servern besvarar varje steg med en numerisk kod. Ur denna till synes enkla följd uppstår routning, köhantering, återförsök och den avgörande frågan om vilken server som just nu bär ansvaret för meddelandet ([RFC 5321, avsnitt 1 och 2](https://datatracker.ietf.org/doc/html/rfc5321#section-1)).

För meddelandeadministratörer ligger den avgörande egenskapen i **ansvarsövergången**. En lyckad TCP-uppkoppling innebär bara att en lyssnare går att nå. Ett accepterat `RCPT TO` reserverar en mottagare inom den pågående transaktionen. Först ett positivt avslut efter det fullständiga meddelandeinnehållet överför ansvaret till det mottagande SMTP-systemet att leverera meddelandet eller rapportera ett senare fel till envelope-avsändaren ([RFC 5321, avsnitt 4.1.1.4 och 6.1](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4)). Kö-ID, svarskod, fjärr-MTA, mottagarstatus och `Received`-spår är därför inte ovidkommande loggdetaljer, utan beviskedjan för en leverans.

Förklaringen följer ett meddelande från den inlämnande klienten via SMTP-session, envelope och kö till nästa e-postserver eller postlåda. DNS, återförsök, statuskoder, säkerhet och diagnostik fördjupas där de påverkar vägen.

## Roller och ansvarsområden

SMTP känner först endast till **klient** och **server**. I en verklig leveranskedja växlar dessa roller vid varje hopp: En Mail Transfer Agent tar som server emot ett meddelande, lagrar det i sin kö och blir själv klient vid nästa leveransförsök. Endast genom detta rollbyte kan loggar och ansvar läsas korrekt ([RFC 5321, avsnitt 2.3.3](https://datatracker.ietf.org/doc/html/rfc5321#section-2.3.3), [RFC 5598, avsnitt 4](https://datatracker.ietf.org/doc/html/rfc5598#section-4), [RFC 6409, avsnitt 2.1](https://datatracker.ietf.org/doc/html/rfc6409#section-2.1)).

| Roll | Uppgift | Typisk förtroende- och tillståndsgräns |
|---|---|---|
| **MUA** – Message User Agent | skapar ett meddelande och visar mottagna meddelanden | användaridentitet, lokala utkast, adressbok och presentation; ingen offentlig relaytjänst |
| **MSA** – Message Submission Agent | tar emot nya meddelanden från behöriga användare eller program | autentiserad identitet, avsändarbehörighet, submission-policy och omedelbar felåterkoppling |
| **MTA** – Message Transfer Agent | routar och överför accepterade meddelanden till ytterligare MTA:er | beständig kö, retry-schemaläggare, DNS-målval, TLS-policy och leveransansvar |
| **Gateway eller relay** | omvandlar mellan säkerhetszoner, organisationer eller meddelandesystem | policy, omskrivning, innehållskontroll, protokollkonvertering och egen kö |
| **MDA** – Message Delivery Agent | överlämnar till postlåda, fil, pipe eller proprietär lagring | slutlig mottagarupplösning, kvot och postlådekonsistens |
| **LMTP-server** | tar emot ett meddelande från MTA:n för lokal leverans | rapporterar en separat status för varje mottagare efter innehållet och behöver själv ingen långvarig relaykö |

MSA:n är inte bara en MTA på en annan port. Message Submission skiljer den kontrollerade inlämningen av nya meddelanden från den öppna överföringen mellan e-postdomäner. En MSA får bland annat kontrollera avsändarrättigheter, lägga till saknade obligatoriska fält och anpassa meddelanden till lokala regler; en efterföljande MTA bör i princip inte godtyckligt ändra färdigt meddelandeinnehåll ([RFC 6409, avsnitt 1, 6 och 8](https://datatracker.ietf.org/doc/html/rfc6409#section-1)). Denna gräns är särskilt relevant för DKIM-signaturer: Varje efterföljande ändring av signerade headerfält eller body kan göra signaturen ogiltig.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-smtp-zustellpfad.svg?v=20260813" title="Interaktive Infografik: SMTP-Rollen, Sitzungsablauf, Envelope, Queue und Zustellverantwortung" loading="lazy">
  <a href="/images/kb-interaktiv-smtp-zustellpfad.svg?v=20260813">Öppna infografik över SMTP-leveransväg och acceptansgräns</a>
</iframe>

Grafiken visar logiska gränser. MSA, utgående MTA, gateway och inkommande MTA kan köras som separata tjänster, finnas på samma system eller döljas av en molnleverantör. Standarderna föreskriver ingen processarkitektur. Administrativt är uppdelningen ändå nödvändig, eftersom varje accepterad överlämning kan skapa en ny kö, ett nytt retry-tillstånd och en ny ansvarig.

## Envelope och meddelandeinnehåll

Innan dialogen granskas i detalj måste två dataområden hållas åtskilda: **envelope** för transporten och **meddelandeinnehåll** för mottagare och e-postprogram. Envelope består av en Reverse Path från `MAIL FROM` och en eller flera Forward Paths från `RCPT TO`. Innehållet följer efter `DATA` och innehåller ett headerområde enligt Internet Message Format samt body, ofta med MIME-struktur ([RFC 5321, avsnitt 2.3.1](https://datatracker.ietf.org/doc/html/rfc5321#section-2.3.1), [RFC 5322, avsnitt 2.1](https://datatracker.ietf.org/doc/html/rfc5322#section-2.1)).

| Element | Protokollobjekt | Vad det faktiskt används till |
|---|---|---|
| `MAIL FROM:<bounce@example.net>` | Envelope Reverse Path | mål för icke-leveransrapporter och indata för SPF; får vara tomt för DSN:er |
| `RCPT TO:<user@example.ch>` | Envelope Forward Path | faktisk routning och mottagarbeslut; kan upprepas per mottagare |
| `From: Author <author@example.net>` | Meddelandeheader | synlig författaridentitet och DMARC-referensdomän; inget SMTP-routningskommando |
| `To:` och `Cc:` | Meddelandeheader | synliga målangivelser för människor och MUA:er; behöver inte motsvara alla envelope-mottagare |
| `Bcc:` | Skapelseinformation | Bcc-mottagare levereras via envelope-adresser utan att deras adress avslöjas för de andra kopiorna |
| `Message-ID:` | Meddelandeheader | innehållslig meddelandeidentifierare; inte samma sak som kö-ID eller DSN-envelope-ID |
| `Return-Path:` | Trace-header som skapas vid slutlig leverans | bevarar den senast använda Reverse Path; bör inte övertas som en fritt satt avsändarrad från ett inkommande meddelande |

SMTP-servrar får inte härleda den slutliga rutten från `To:`, `Cc:` eller `From:`. Omvänt behöver envelope-adresser inte förekomma i meddelandeheadern. Därför fungerar Bcc, e-postlistor, vidarebefordringar, SRS och separata bounce-adresser. Separationen förklarar också varför ett meddelande i klienten visuellt kan komma från `ceo@example.ch`, medan dess envelope-avsändare använder en annan domän. SPF granskar SMTP-sökvägen; DMARC skapar dessutom en koppling till det synliga `From:`. Detaljerna för dessa kontroller hör hemma i [E-postautentisering](/kb/mail-auth) ([RFC 5321, bilaga C](https://datatracker.ietf.org/doc/html/rfc5321#appendix-C), [RFC 5322, avsnitt 3.6](https://datatracker.ietf.org/doc/html/rfc5322#section-3.6)).

En tom Reverse Path överförs som `MAIL FROM:<>`. Den förhindrar felrapportslingor och krävs för Delivery Status Notifications. En server som kategoriskt avvisar tomma avsändare skadar därför e-postsystemets återkopplingsväg ([RFC 5321, avsnitt 3.6.3](https://datatracker.ietf.org/doc/html/rfc5321#section-3.6.3)).

## SMTP-session och transaktion

På en [TCP-anslutning](/kb/tcp) körs en **SMTP-session**; i den kan flera meddelanden följa som separata **e-posttransaktioner**. Servern börjar med en `220`-hälsning; klienten identifierar sig med `EHLO` och får en sessionsspecifik lista över ESMTP-funktioner. Därefter består en transaktion av `MAIL`, minst ett accepterat `RCPT` samt `DATA` eller en förhandlad ersättning såsom `BDAT` ([RFC 5321, avsnitt 3.1 och 4.1](https://datatracker.ietf.org/doc/html/rfc5321#section-3.1)).

1. **Anslutning och hälsning:** Klienten ansluter; servern svarar med `220`. Om den redan här fördröjer eller vägrar finns ännu ingen e-posttransaktion.
2. **Funktionsförhandling:** `EHLO client.example` försätter båda sidor i initialt tillstånd och levererar utökningar som `SIZE`, `STARTTLS`, `PIPELINING` eller `SMTPUTF8`. En klient får inte förutsätta att en annan klusternod eller en senare session erbjuder samma funktioner.
3. **Valfri TLS-uppgradering:** Efter lyckad `STARTTLS` förkastas SMTP-tillståndet och tidigare inlärda funktioner. Klienten skickar `EHLO` på nytt inom den skyddade kanalen ([RFC 3207, avsnitt 4](https://datatracker.ietf.org/doc/html/rfc3207#section-4)).
4. **Envelope-avsändare:** `MAIL FROM` öppnar en transaktion och tömmer tidigare envelope- och databuffertar. ESMTP-parametrar som `SIZE`, `BODY` eller `SMTPUTF8` hör till detta kommando.
5. **Mottagare:** Varje `RCPT TO` besvaras individuellt. En server kan acceptera två mottagare och permanent avvisa en tredje; klienten skickar då innehållet för den accepterade delmängden.
6. **Innehåll:** Efter `DATA` följer vid beredskap `354`. Klienten överför header och body som CRLF-avslutade rader, maskerar punkter i början av rader och avslutar med en rad som bara består av `.`.
7. **Avslut:** Svaret efter denna End-of-Data-markör avgör om servern övertar meddelandet som helhet för de tidigare accepterade mottagarna. `RSET` förkastar den pågående transaktionen utan att avsluta sessionen; `QUIT` avslutar den ordnat.

SMTP är lockstep-orienterat, men ESMTP-`PIPELINING` får skicka tillåtna kommando-grupper utan att invänta varje enskilt svar. Svaren förblir i ankomstordning och måste fortfarande korrekt kunna kopplas till varje kommando ([RFC 2920](https://datatracker.ietf.org/doc/html/rfc2920)). En paketfångst med flera omedelbart efterföljande `MAIL`-/`RCPT`-rader är därför inte automatiskt ett protokollbrott.

Dialogen är därmed fullständig, men leveransansvaret har ännu inte klargjorts. Det växlar först med ett mycket specifikt svar från den mottagande servern.

## Acceptansgräns: 250 efter End-of-Data

Den viktigaste SMTP-gränsen ligger efter meddelandeinnehållet. Ett `250` på `MAIL FROM` accepterar endast Reverse Path. Ett `250` på `RCPT TO` accepterar en mottagare för den pågående transaktionen. Först ett `250` efter End-of-Data-markören betyder att servern har övertagit ansvaret för meddelandet. Ett korrekt system måste därefter antingen leverera det eller generera en icke-leveransrapport vid ett senare upptäckt fel; den sändande klienten får ta bort sin lokala kopia från det aktiva leveransansvaret ([RFC 5321, avsnitt 4.1.1.4 och 6.1](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4)).

Denna semantik gör dialogens sista del kritisk. Om anslutningen bryts efter punkten men före ett fullständigt mottaget svar vet klienten inte säkert om servern har lagrat meddelandet. Den måste kunna försöka leverera igen; därmed är dubbletter möjliga. SMTP erbjuder ingen global exakt-en-gång-garanti. System behöver därför idempotent spårning via kö-ID, `Message-ID`, mottagare, tid, storlek och vid behov en proprietär transaktionsreferens, utan att likställa dessa identifierare.

Innehållsfilter, DLP, skadlig programvarukontroll eller DKIM-signering bör integreras i acceptanspipelinen så att servern före det slutliga positiva svaret vet om den kan överta beständigt ansvar. En framförliggande proxy som för tidigt rapporterar `250` och därefter förlorar meddelandet internt kan inte längre returnera felet till den ursprungliga SMTP-klienten. Just här måste en arkitektur fastställa om acceptans bekräftas först efter durable write, efter replikering, efter skanning eller redan vid överlämning till en intern broker. Detta är en produktegenskap, inte ett krav i SMTP-standarden, och ska styrkas genom tillverkardokumentation samt fel- och återställningstester.

Efter acceptans måste MTA:n avgöra vart den ska skicka meddelandet härnäst. För internetmail börjar detta beslut i DNS.

## Routning via DNS och MX

För direkt internetleverans bestämmer den sändande MTA:n nästa mål utifrån domänen för envelope-mottagaren. Den frågar [DNS](/kb/dns) efter MX-poster, sorterar kandidater efter stigande Preference-värde och löser de valda Exchange-namnen till A- eller AAAA-adresser. Flera mål med samma Preference är likvärdiga kandidater; en backup-MX med högre värde är inte en replik av primärkön, utan en självständig SMTP-mottagare med egen policy och leveransansvar ([RFC 5321, avsnitt 5](https://datatracker.ietf.org/doc/html/rfc5321#section-5)).

Om en MX-RRset saknas behandlar SMTP historiskt domänen som en implicit MX på själva domännamnet och försöker dess adresser. En explicit **Null MX** med Preference `0` och Exchange `.` förklarar däremot att en domän inte tar emot e-post ([RFC 7505](https://datatracker.ietf.org/doc/html/rfc7505)). `NXDOMAIN`, `NODATA`, `SERVFAIL`, DNSSEC-valideringsfel och en nåbar MX utan fungerande SMTP-tjänst är olika tillstånd och måste förbli synliga separat i köorsaken.

Interna e-postvägar använder ofta statiska next hops, smarthosts, anslutningar eller routningstabeller i stället för offentligt MX-val. Envelope-mottagaren förblir ändå leveransadressen; den lokala routningspolicyn avgör bara vilket nästa system den anförtros åt. Vid varje diagnostik ska därför tre namn noteras: mottagardomän, valt next-hop-namn och faktiskt ansluten IP-adress.

Även när MTA:n känner till målet är leverans inte alltid möjlig omedelbart. Tillfälliga fel leder därför till kön i stället för att meddelandet förloras.

## Kö, återförsök och DSN

En fullfjädrad MTA är ett Store-and-Forward-system. Om den inte kan nå nästa hopp omedelbart eller om detta svarar tillfälligt, behåller MTA:n meddelande och envelope i en beständig kö, planerar ett nytt försök och hanterar status per mottagare. RFC 5321 kräver en retry-strategi, men föreskriver inte en enskild algoritm för alla produkter. Den rekommenderar fördröjda återförsök, en avisering vid längre fördröjning och flerdagars lagring före slutligt uppgivande ([RFC 5321, avsnitt 4.5.4](https://datatracker.ietf.org/doc/html/rfc5321#section-4.5.4)). Produktvärden som backoff, maximal kölivslängd, parallellitet och destination concurrency måste därför läsas från den konkreta MTA-konfigurationen.

En meningsfull köpost innehåller minst:

- oföränderligt meddelandeinnehåll eller en refererad blob;
- envelope-avsändare och status för varje envelope-mottagare;
- tidpunkt för acceptans, nästa försök och utgångsgräns;
- vald transport, next hop, senaste adress och TLS-/policybeslut;
- senaste SMTP- och Enhanced-statuskod samt fjärrtext;
- lokalt kö-ID samt korrelerbara ID:n från tidigare eller efterföljande hopp;
- tillstånd för aktiv, deferred, hold, delivered, bounced och quarantined;
- replikerings-, filsystems- eller databastillstånd om acceptansen ska vara hög tillgänglig.

Tillfälliga fel försöks normalt igen. Permanenta fel före den egna acceptansen returneras synkront till den sändande SMTP-klienten. Permanenta fel efter den egna acceptansen leder till en **Delivery Status Notification** (DSN) till Reverse Path. RFC 3464 definierar för detta ett maskinläsbart `multipart/report`-meddelande med per-message- och per-recipient-fält som `Reporting-MTA`, `Final-Recipient`, `Action`, `Status` och valfritt `Remote-MTA` eller `Diagnostic-Code` ([RFC 3464](https://datatracker.ietf.org/doc/html/rfc3464)).

En bounce är inte automatiskt tillförlitlig. DSN:er kan förfalskas, avslöja det ursprungliga innehållet eller skapa backscatter vid felaktigt satt envelope-avsändare. En gateway bör om möjligt avvisa ogiltiga meddelanden under den ursprungliga SMTP-sessionen i stället för att acceptera dem och senare skicka tillbaka dem till en möjligen förfalskad adress.

## Svarskoder och diagnostisk betydelse

SMTP-svar har en tresiffrig baskod. Det första sifferområdet beskriver det omedelbara tillståndet; texten är avsedd för människor och kan vara produktspecifik ([RFC 5321, avsnitt 4.2](https://datatracker.ietf.org/doc/html/rfc5321#section-4.2)).

| Klass | Betydelse i dialogen | Konsekvens för den sändande processen |
|---|---|---|
| `2xx` | positivt slutförande | utför nästa protokollsteg; bokför ansvaret som överfört efter End-of-Data |
| `3xx` | positiv mellanstatus | skicka ytterligare data, exempelvis meddelandeinnehållet efter `354` |
| `4xx` | tillfälligt negativt slutförande | behåll meddelandet respektive berörd mottagare i kön och försök igen enligt retry-policy |
| `5xx` | permanent negativt slutförande | upprepa inte oförändrat; rapportera synkront till föregående klient eller generera DSN efter tidigare acceptans |

**Enhanced Status Codes** kompletterar formatet `x.y.z`. `x` är framgång, tillfällig eller permanent; `y` betecknar det grova felområdet och `z` detaljen. Därmed tillhör `5.1.1` adressklassen och betecknar en obefintlig målpostlåda, medan `4.4.x` klassificerar nätverks- och routningsproblem. De registrerade betydelserna finns i [IANA-registret för Enhanced Status Codes](https://www.iana.org/assignments/smtp-enhanced-status-codes/smtp-enhanced-status-codes.xhtml); en tillverkartext får inte utvärderas i strid med den numeriska semantiken ([RFC 3463](https://datatracker.ietf.org/doc/html/rfc3463), [RFC 5248](https://datatracker.ietf.org/doc/html/rfc5248)).

Samma kod kan avse något olika beroende på protokollfas. `550` på `MAIL FROM` avvisar avsändaren eller dess behörighet, på `RCPT TO` en specifik mottagare och efter innehållet hela transaktionen. Ett ärende utan fjärrvärd, tid, fas, baskoder, Enhanced Code och fullständig svarstext är därför ofullständigt.

## ESMTP och utökningar

Extended SMTP använder `EHLO` för att förhandla funktioner per anslutning. IANA för det bindande [registret över SMTP Service Extensions](https://www.iana.org/assignments/smtp/smtp.xhtml). En klient får bara använda en utökning om den erbjöds i den aktuella sessionen; efter STARTTLS måste den hämta listan på nytt.

| EHLO-nyckelord | Teknisk effekt | Administrativ relevans |
|---|---|---|
| `SIZE` | Servern kan ange en fast storleksgräns; klienten kan deklarera den förväntade storleken vid `MAIL FROM` | undviker onödig dataöverföring men garanterar ännu ingen acceptans ([RFC 1870](https://datatracker.ietf.org/doc/html/rfc1870)) |
| `PIPELINING` | tillåter definierade kommando-grupper utan Round Trip per kommando | ökar genomströmningen över latenta länkar; svaren måste korreleras exakt i ordning ([RFC 2920](https://datatracker.ietf.org/doc/html/rfc2920)) |
| `8BITMIME` | tillåter 8-bitars MIME-bodytransport med `BODY=8BITMIME` | är ingen tillåtelse för godtyckliga binära data och ändrar inte headerreglerna ([RFC 6152](https://datatracker.ietf.org/doc/html/rfc6152)) |
| `CHUNKING` | introducerar `BDAT` med explicita byteantal som alternativ till `DATA` | ingen punkttransparens; förutsättning för `BINARYMIME` ([RFC 3030](https://datatracker.ietf.org/doc/html/rfc3030)) |
| `SMTPUTF8` | tillåter internationaliserade postlådor och UTF-8 i därför definierade headerfält | varje nödvändigt efterföljande hopp måste stödja funktionen, annars uppstår ett interoperabilitetsfel ([RFC 6531](https://datatracker.ietf.org/doc/html/rfc6531)) |
| `DSN` | överför parametrar för leveransaviseringar till `MAIL` och `RCPT` | skiljer mellan önskade rapporter om framgång, fördröjning och fel ([RFC 3461](https://datatracker.ietf.org/doc/html/rfc3461)) |
| `STARTTLS` | växlar från klartextdialogen till en TLS-kanal | återställer SMTP-tillståndet; säkerheten beror vid MTA-överföring även på policy ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207)) |
| `AUTH` | binder SASL-autentisering till SMTP och listar mekanismer | avsedd för submission; erbjudna mekanismer kan ändras efter STARTTLS ([RFC 4954](https://datatracker.ietf.org/doc/html/rfc4954)) |

Capability Drift mellan klusternoder är ett vanligt driftproblem. Om endast enskilda lyssnare erbjuder `SMTPUTF8`, samma `SIZE`-värde eller identiska AUTH-mekanismer uppstår till synes slumpmässiga leveransfel. Övervakning bör därför inte bara kontrollera en VIP, utan varje relevant backend-sökväg med samma SNI, EHLO-namn och källnät.

Hittills handlade det om transport mellan servrar. Inlämning av användare och program är en egen tjänst med andra portar, identiteter och risker för missbruk.

## Submission, relay och autentisering

Port **25** används för MTA-till-MTA-överföring. Port **587** betecknar Message Submission; port **465** är registrerad för submission med TLS aktivt från början som tjänsten `submissions`. Port 465 är inte en alternativ MX-port: Offentlig MX-leverans har ingen portangivelse och använder port 25 ([RFC 6409, avsnitt 1 och 3](https://datatracker.ietf.org/doc/html/rfc6409#section-3.1), [RFC 8314, avsnitt 7.3](https://datatracker.ietf.org/doc/html/rfc8314#section-7.3), [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)).

En submission-tjänst kombinerar autentisering med auktorisering. `AUTH` använder SASL och kan bära mekanismer som PLAIN över TLS eller OAuth-baserade Bearer-token. Den autentiserade identiteten, den begärda auktoriseringsidentiteten och den tillåtna envelope-avsändaren är olika storheter. En lyckad inloggning ger inte automatiskt rätt att använda varje avsändardomän eller varje envelope-avsändare ([RFC 4422, avsnitt 3.4](https://datatracker.ietf.org/doc/html/rfc4422#section-3.4), [RFC 4954](https://datatracker.ietf.org/doc/html/rfc4954), [RFC 7628](https://datatracker.ietf.org/doc/html/rfc7628)).

Open relay uppstår när en MTA vidarebefordrar meddelanden från obehöriga källor till icke-lokala mål. En säker konfiguration avgör därför separat om lokala mottagardomäner, betrodda källnät, autentiserad submission och explicita anslutningar. IP-allowlisting är nätverksauktorisering och ingen ersättning för en användaridentitet; vid NAT, moln-egress eller delade relayer kan den frisläppa en betydligt större avsändarkrets än avsett.

Först när roll och port är fastställda kan lämplig TLS-policy bestämmas: Submission ska vara skyddad och autentiserad, medan öppen MTA-transport behöver ytterligare domänpolicyer.

## TLS-policy för SMTP

STARTTLS skyddar den enskilda anslutningen när båda sidor genomför uppgraderingen. Vid öppen MTA-överföring förblir rent STARTTLS dock opportunistiskt: En aktiv angripare kan ta bort erbjudandet eller manipulera DNS-mål, och en avsändare utan ytterligare policy kan falla tillbaka till klartext. [DANE för SMTP](https://datatracker.ietf.org/doc/html/rfc7672) binder TLSA-data via DNSSEC till MX-målvalet. [MTA-STS](https://datatracker.ietf.org/doc/html/rfc8461) publicerar en PKIX-baserad MX- och TLS-policy via DNS och HTTPS. [TLS-RPT](https://datatracker.ietf.org/doc/html/rfc8460) rapporterar aggregerade fel för sådana policyer. Kryptografiska detaljer, certifikatkedjor och testmetoder behandlas i artikeln [TLS](/kb/tls).

Submission skiljer sig: Här är klient och tjänst administrativt kopplade, och användaren får inte tyst falla tillbaka till klartext. RFC 8314 rekommenderar implicit TLS för Mail Submission och Access och kräver att klartext behandlas som föråldrad. SMTP AUTH med lösenords- eller Bearer-mekanismer kräver en tillräckligt skyddad kanal; en server bör inte ens erbjuda svaga mekanismer före STARTTLS ([RFC 8314, avsnitt 3](https://datatracker.ietf.org/doc/html/rfc8314#section-3), [RFC 4954, avsnitt 14](https://datatracker.ietf.org/doc/html/rfc4954#section-14)).

## Lokal leverans med LMTP

**Local Mail Transfer Protocol** (LMTP) använder en SMTP-liknande dialog för överlämning till ett system utan egen långvarig e-postkö, exempelvis en postlådetjänst. Den väsentliga skillnaden visar sig efter meddelandeinnehållet: LMTP levererar ett eget slutsvar för varje tidigare accepterad mottagare. Därmed kan e-postlagret leverera framgångsrikt till en mottagare och avvisa en annan på grund av kvot eller ett lokalt fel, utan att den framförliggande MTA:n måste behandla hela transaktionen som en enhet ([RFC 2033, avsnitt 4.2](https://datatracker.ietf.org/doc/html/rfc2033#section-4.2)).

LMTP flyttar inte ut ansvaret ur systemet. Om e-postlagret tillfälligt avvisar en mottagare måste den framförliggande MTA:n fortsätta behålla denna mottagare i sin kö. Med Unix-sockets försvinner visserligen IP-nätvägen, men inte behovet av att övervaka socketbehörigheter, backpressure, timeouts, per-recipient-svar och beständighetsgränsen.

## Implementerings- och teknikstack

SMTP definierar trådprotokollet, inte den interna programvaruarkitekturen. I praktiken förenar en MTA flera tekniska skikt:

| Skikt | Typiska komponenter | Kritiskt drifttillstånd |
|---|---|---|
| Tjänsteval | DNS-resolver, MX/A/AAAA, statiska routningstabeller | cache, DNSSEC-status, next-hop-val och ansluten IP |
| Nätverkstransport | TCP-stack, lyssnare, lastbalanserare, proxy | anslutningsgränser, timeouts, bevarande av käll-IP och backpressure |
| Kanalskydd | TLS-bibliotek, certifikat- och policyengine | SNI, truststore, DANE/MTA-STS-beslut och fallback |
| Protokollfrontend | SMTP-parser, tillståndsmaskin, AUTH/SASL, rate limits | session state, erbjudna funktioner och missbruksgränser |
| Policy och innehåll | adressupplösning, anti-spam, skadlig programvara, DLP, signering | deterministisk ordning, timeouts och beteende vid skannerfel |
| Persistens | kö-/spoolfiler, databas eller blob store | durable write före `250`, ledigt utrymme, replikering och återställning |
| Schemaläggare och routning | retry-timers, destination concurrency, transporttabeller | rättvisa, backoff, köålder och blockerade domäner |
| Leveransadapter | SMTP, LMTP, lokal postlåda, fil, pipe eller API | per-recipient-status, idempotens och överlämningsgräns |
| Observerbarhet | strukturerade loggar, kö-ID, mätvärden, spår och DSN | korrelation över hopp utan att känsligt innehåll går förlorat |

Implementeringarna omsätter dessa skikt på olika sätt. [Postfix](https://www.postfix.org/OVERVIEW.html) delar upp mottagning, cleanup, Queue Manager och leveransagenter i samverkande processer och skiljer `incoming`, `active` och `deferred` som kötillstånd. [Exim](https://www.exim.org/exim-html-current/doc/html/spec_html/ch-how_exim_receives_and_delivers_mail.html) lagrar envelope/headers och body i spoolen och låter queue-runners behandla förfallna leveransförsök. [Apache James](/kb/apache-james) körs på JVM och kopplar ihop protokollservrar, kö och en Mailet-pipeline med utbytbara lagringsbackends. Alla tre talar SMTP, men deras process-, kö-, utöknings- och återställningsmodeller är inte utbytbara.

Programspråk och runtime påverkar paketering, patchprocess, minnesprofil och utbyggbarhet, inte semantiken i SMTP-dialogen. Postfix byggs med en inbyggd C-toolchain för Unix-liknande system ([Postfix INSTALL](https://www.postfix.org/INSTALL.html)); [Exim-dokumentationen](https://www.exim.org/exim-html-2.00/doc/html/oview.html) anger ANSI C som implementeringsspråk. Apache James beskriver sig däremot som en Java-/Scala-plattform på JVM ([Apache James](https://james.apache.org/)). En molntjänst kan avbilda samma RFC-roller genom proprietära mikrotjänster, distribuerade köer och API:er. För driftgodkännande måste därför inte bara EHLO-funktioner utan även persistenslöfte, retry-modell, köexport, backup, nyckelhantering, skalningsgränser och felområden dokumenteras.

Av leveransförloppet följer en fast felsökning: Bestäm målet, kontrollera anslutning och SMTP-dialog, utvärdera svaret och följ sedan kön samt nästa hopp.

## Diagnostik från DNS till kön

En tillförlitlig SMTP-diagnostik följer leveransvägen och håller observationsnivåerna åtskilda. Tester bör genomföras med testdomäner och auktoriserade mottagare; främmande servrar får inte missbrukas för relay-, adressvaliderings- eller lasttester.

### Bestäm MX-mål och adresser

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-DNS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
    </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
    </code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) och [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) visar publicerade DNS-data. Utdata måste granskas med avseende på Preference, Null MX, CNAME-kedjor, IPv4/IPv6 och den resolver som faktiskt frågats. Ett DNS-svar bevisar varken lyssnar- eller TLS-funktion.

### Kontrollera TCP-nåbarhet för valt hopp

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-TCP-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
    </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
    </code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) och [`nc`](https://man.openbsd.org/nc) kontrollerar här endast DNS-upplösning och TCP-uppkoppling. En framgång på 25 säger inte om SMTP-hälsningen kommer i tid; en framgång på 587 eller 465 säger inget om TLS- och AUTH-policy.

### Kontrollera STARTTLS, EHLO och certifikat

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-STARTTLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
    </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
    </code></pre>
  </div>
</div>

[`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) utför med `-starttls smtp` den nödvändiga klartextdialogen, sätter SMTP-EHLO-namnet med `-name` och SNI med `-servername`. `-verify_hostname` och `-verify_return_error` gör den rena handskakningen till ett explicit PKIX-namntest. Resultatet beskriver OpenSSL med sin truststore, inte automatiskt den produktiva MTA:ns policy.

### Testa envelope och svarfas kontrollerat

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für kontrollierten SMTP-Dialog">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
    </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
    </code></pre>
  </div>
</div>

[`swaks`](https://jetmore.org/john/code/swaks/) loggar SMTP-kommandon och svar och kan medvetet avsluta testet efter `RCPT` utan att skicka meddelandedata. Verktyget är inte en systemkomponent i Windows; `swaks.exe` respektive en Perl-installation måste komma från en kontrollerad källa. Ett RCPT-test får endast utföras mot egna eller uttryckligen godkända system och adresser, eftersom det utlöser policyer, kataloguppslag och rate limits.

### Undersök lyssnare och paket på servern

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-Listener- und Paketprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
    </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
    </code></pre>
  </div>
</div>

[`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) och [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) kopplar lyssnare och TCP-tillstånd till lokala processer. [`pktmon`](https://learn.microsoft.com/windows-server/networking/technologies/pktmon/pktmon) och [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) fångar paket; produktiva fångster kan innehålla envelope-adresser, okrypterat innehåll och autentiseringsdata och måste skyddas, begränsas och raderas i enlighet med detta. [`Wireshark`](https://www.wireshark.org/docs/wsug_html/index.html) lämpar sig för analys av TCP, SMTP och TLS-handskakningar, men ersätter varken kö- eller policyloggar.

### Tilldela fel till en gräns

| Observation | Trolig gräns | Nästa tillförlitliga kontroll |
|---|---|---|
| ingen MX eller Null MX | DNS-publicering för mottagardomänen | auktoritativt svar, DNSSEC-status, negativ cache-TTL |
| TCP-timeout till en av flera MX | nätväg eller enskild MX | varje MX-adress och IP-familj separat, SYN/SYN-ACK och returväg |
| anslutning upprättad, ingen `220` | lyssnare, proxy, tarpit eller överbelastning | hälsningslatens, connection limits, frontend- och backendloggar |
| `STARTTLS` saknas bara på enskilda anslutningar | Capability Drift, proxy eller policy efter källnät | varje klusternod, EHLO före/efter TLS, lastbalanserarsökväg |
| `4xx` vid `MAIL FROM` | avsändar-, rate- eller resurspolicy | fullständig kod, käll-IP, envelope-avsändare, kö-/policylogg |
| `5xx` vid ett `RCPT TO` | mottagar- eller relaybeslut | berörd envelope-mottagare, Accepted Domains, Directory Edge |
| `4xx` efter End-of-Data | skanner, lagring, kö eller innehållspolicy | lokal acceptanspipeline, spoolutrymme, skannerlatens, ingen köövertagning |
| `250` efter End-of-Data, senare DSN | efterföljande leverans misslyckades efter egen acceptans | rapporterande MTA, fjärr-MTA, kö-ID, DSN-status och ursprunglig mottagare |
| kön växer för många domäner | lokal schemaläggare, DNS, nät eller resurser | åldersfördelning, leveranstakter, aktiva/deferred-tillstånd, globala beroenden |
| kön växer endast för en domän | Remote-MX, TLS-policy eller destination throttling | senaste fel per MX/IP, DANE/MTA-STS, concurrency och retry-tid |
| postlåda saknas trots lyckat SMTP-hopp | MDA/LMTP eller internt routningshopp | senaste betrodda `Received`-rad, LMTP per-recipient-svar, storeloggar |
| sporadiska dubbletter efter timeout | oklart svar vid acceptansgränsen | avbrottstidpunkt relativt slutligt `250`, kö- och Message-ID-korrelation |

## Observerbarhet och driftmodell

En SMTP-monitor bör inte bara skicka ett testmeddelande. Den måste göra systemets enskilda tillstånd synliga:

- Greeting-, EHLO-, STARTTLS-, AUTH- och End-of-Data-latens var för sig;
- anslutningar, sessioner, aktiva transaktioner och avvisningar per fas;
- erbjudna ESMTP-funktioner per lyssnare och backend;
- acceptansfrekvens, antal mottagare, meddelandestorlek och durable-write-latens;
- kövolym efter ålder, mål, transport, status och senaste fel;
- retry-frekvens, äldsta meddelande, utgående leveranser och DSN-generering;
- DNS-mål, ansluten IP, IPv4/IPv6, TLS-läge, cipher och policyresultat;
- svarskoder som strukturerade fält i stället för enbart fritext;
- korrelation av edge-ID, internt kö-ID, fjärrkö-ID, `Message-ID` och mottagare;
- LMTP-/postlåde-status per mottagare samt karantän- och releasevägar;
- spoolkapacitet, I/O-latens, databas- eller brokertillstånd och replikeringsfördröjning;
- skyddsgränser för loggar och fångster, eftersom envelope, headers och innehåll innehåller personuppgifter eller konfidentiella data.

Enbart köbackuper räcker inte. Återställning måste ta hänsyn till konfiguration, routningstabeller, certifikat och nycklar, kömetadata, meddelandeblobbar, karantän, signaturtillstånd och idempotens. Efter en återställning får en redan levererad mottagarkopia inte okontrollerat skickas på nytt; samtidigt får ännu öppna mottagare av ett delvis levererat meddelande inte gå förlorade. RPO och RTO måste därför definieras per tillståndskomponent och testas med en verklig kö med flera mottagare.

## Teknisk historia

SMTP uppstod ur tidigare ARPANET-postförfaranden. Jon Postels [RFC 821](https://datatracker.ietf.org/doc/html/rfc821) standardiserade 1982 den modell som fortfarande är igenkännbar med Sender-SMTP, Receiver-SMTP, `MAIL`, `RCPT`, `DATA` och tresiffriga svar. Meddelandeformatet beskrevs samma år separat i RFC 822. Denna separation mellan överföringsprotokoll och meddelandeformat bestod.

Med MIME växte SMTP bortom ren 7-bitars text. ESMTP-ramverket införde `EHLO`, registrerade funktioner och parametrar vid `MAIL` respektive `RCPT`; dess version från 1995 fanns i [RFC 1869](https://datatracker.ietf.org/doc/html/rfc1869). Utökningar som byggde på detta, såsom `SIZE`, `8BITMIME`, `PIPELINING`, `AUTH`, `STARTTLS`, `CHUNKING`, DSN och senare `SMTPUTF8`, utvecklade protokollet stegvis utan att skapa en parallell «SMTP 2»-stack.

[RFC 2821](https://datatracker.ietf.org/doc/html/rfc2821) konsoliderade SMTP 2001 och ersatte RFC 821 samt flera kompletterande dokument. [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321) följde 2008 som ytterligare en konsolidering; motsvarande Internet Message Format finns i [RFC 5322](https://datatracker.ietf.org/doc/html/rfc5322). Message Submission specificerades först separat och konsoliderades i RFC 6409. Den tekniska utvecklingen skedde därmed genom tydligare roller, utökningsregister, internationaliserade adresser, leveransrapporter och transportpolicy, medan den grundläggande tillståndsfulla dialogen förblev kompatibel.

Även programvaruarkitekturerna utvecklades åt olika håll. Historisk Sendmail-kompatibilitet präglade lokala kommandoradsgränssnitt; Postfix satsade på samverkande processer med separerade privilegier och explicita köer, Exim på en starkt konfigurerbar router-/transport- och spoolstrategi, och JVM-system som Apache James gjorde bearbetningen utbyggbar genom komponenter och Mailets. Molnplattformar fördelar samma logiska roller på frontends, brokers, policytjänster och regionala köer. För administratörer förblir därför RFC-semantiken den gemensamma nämnaren, medan persistens, skalning och återställning alltid måste styrkas produktspecifikt.

## Källor

- [RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321) – SMTP-modell, routning, kommandon, svar, kö, återförsök, spår och leveransansvar.
- [RFC 5598, avsnitt 4](https://datatracker.ietf.org/doc/html/rfc5598)
- [RFC 6409 – Message Submission for Mail](https://datatracker.ietf.org/doc/html/rfc6409) – separation mellan submission och relay, roller, port 587 och MSA-policy.
- [RFC 5322 – Internet Message Format](https://datatracker.ietf.org/doc/html/rfc5322) – headers, body, adressfält, Message-ID och tracefält.
- [RFC 3207 – SMTP Service Extension for Secure SMTP over TLS](https://datatracker.ietf.org/doc/html/rfc3207) – STARTTLS och återställning av SMTP-tillståndet.
- [RFC 2920 – SMTP Command Pipelining](https://datatracker.ietf.org/doc/html/rfc2920) – ordnade kommando-grupper utan Round Trip per kommando.
- [RFC 7505 – A Null MX No Service Resource Record](https://datatracker.ietf.org/doc/html/rfc7505) – uttrycklig markering av domäner utan e-postfunktion.
- [RFC 3464 – Delivery Status Notification Format](https://datatracker.ietf.org/doc/html/rfc3464) – maskinläsbar DSN-struktur och mottagarstatus.
- [IANA – SMTP Enhanced Status Codes](https://www.iana.org/assignments/smtp-enhanced-status-codes/smtp-enhanced-status-codes.xhtml) – registrerade betydelser för detaljkoderna.
- [RFC 3463 – Enhanced Mail System Status Codes](https://datatracker.ietf.org/doc/html/rfc3463) – struktur och klasser för utökade statuskoder.
- [RFC 5248 – SMTP Enhanced Status Code Registry](https://datatracker.ietf.org/doc/html/rfc5248) – registreringsmodell för Enhanced Codes.
- [IANA – Simple Mail Transfer Protocol Parameters](https://www.iana.org/assignments/smtp/smtp.xhtml) – SMTP-kommandon, utökningar och EHLO-nyckelord.
- [RFC 1870 – SMTP SIZE Extension](https://datatracker.ietf.org/doc/html/rfc1870) – storleksförhandling och avvisning före innehållet.
- [RFC 6152 – SMTP Service Extension for 8-bit MIME Transport](https://datatracker.ietf.org/doc/html/rfc6152) – 8BITMIME och BODY-parametrar.
- [RFC 3030 – CHUNKING and BINARYMIME](https://datatracker.ietf.org/doc/html/rfc3030) – BDAT-överföring och binära MIME-meddelanden.
- [RFC 6531 – SMTP Extension for Internationalized Email](https://datatracker.ietf.org/doc/html/rfc6531) – SMTPUTF8 och internationella postlådor.
- [RFC 3461 – SMTP Service Extension for Delivery Status Notifications](https://datatracker.ietf.org/doc/html/rfc3461) – envelope-parametrar för DSN-begäranden.
- [RFC 4954 – SMTP Service Extension for Authentication](https://datatracker.ietf.org/doc/html/rfc4954) – AUTH-kommando och SASL-profil för SMTP.
- [RFC 8314, avsnitt 7.3](https://datatracker.ietf.org/doc/html/rfc8314)
- [IANA – Service Name and Transport Protocol Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) – koppling av SMTP, Submission och Submissions till portar.
- [RFC 4422, avsnitt 3.4](https://datatracker.ietf.org/doc/html/rfc4422)
- [RFC 7628 – SASL Mechanisms for OAuth](https://datatracker.ietf.org/doc/html/rfc7628) – OAuth-baserade SASL-mekanismer för SMTP och andra protokoll.
- [RFC 7672 – SMTP Security via DANE TLS](https://datatracker.ietf.org/doc/html/rfc7672) – DNSSEC- och TLSA-bunden SMTP-transportpolicy.
- [RFC 8461 – SMTP MTA Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc8461) – PKIX-baserad MX- och TLS-policy.
- [RFC 8460 – SMTP TLS Reporting](https://datatracker.ietf.org/doc/html/rfc8460) – aggregerade rapporter om transport-TLS-fel.
- [RFC 2033, avsnitt 4.2](https://datatracker.ietf.org/doc/html/rfc2033)
- [Postfix – Architecture Overview](https://www.postfix.org/OVERVIEW.html) – processroller, Queue Manager och leveransagenter.
- [Exim – How Exim receives and delivers mail](https://www.exim.org/exim-html-current/doc/html/spec_html/ch-how_exim_receives_and_delivers_mail.html) – spool-, routnings-, transport- och queue-runner-modell.
- [Postfix – Installation From Source Code](https://www.postfix.org/INSTALL.html) – krav på inbyggd build, plattform och toolchain.
- [Exim – teknisk översikt](https://www.exim.org/exim-html-2.00/doc/html/oview.html) – ANSI C som implementeringsspråk och tekniska produktegenskaper.
- [Apache James Project](https://james.apache.org/) – JVM-, Java-/Scala- och komponentarkitektur.
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – DNS-frågor i Windows.
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html) – DNS-frågor i Unix-system.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – TCP-nåbarhet i Windows.
- [OpenBSD – nc(1)](https://man.openbsd.org/nc) – TCP-anslutningar med Netcat.
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/) – STARTTLS-, SNI- och certifikatdiagnostik.
- [Swaks – Swiss Army Knife for SMTP](https://jetmore.org/john/code/swaks/) – SMTP-transaktions- och utökningstest.
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) – lokala TCP-lyssnare och anslutningar.
- [Linux man-pages – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html) – diagnostik av socket- och TCP-tillstånd.
- [Microsoft Learn – Packet Monitor](https://learn.microsoft.com/windows-server/networking/technologies/pktmon/pktmon) – paketfångst i Windows och PCAPNG-konvertering.
- [tcpdump(1) manual](https://www.tcpdump.org/manpages/tcpdump.1.html) – paketbaserad diagnostik i Unix-system.
- [Wireshark User’s Guide](https://www.wireshark.org/docs/wsug_html/index.html) – interaktiv analys av nätverks- och protokollspår.
- [RFC 821 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc821) – historisk SMTP-specifikation från 1982.
- [RFC 1869 – SMTP Service Extensions](https://datatracker.ietf.org/doc/html/rfc1869) – historiskt ESMTP-ramverk och EHLO-modell.
- [RFC 2821 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc2821) – SMTP-konsolidering från 2001.
- [RFC 3848 – ESMTP and LMTP Transmission Types](https://datatracker.ietf.org/doc/html/rfc3848) – `with`-värden som ESMTPSA i Received-headers.
