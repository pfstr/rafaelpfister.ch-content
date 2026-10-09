---
title: "E-postautentisering: SPF, DKIM och DMARC"
blatt: "mail-auth"
description: "SPF, DKIM och DMARC i teknisk kontext: identiteter, DNS-utvärdering, signaturer, alignment, policyer, rapportering, vidarebefordran, förtroendegränser och drift för meddelandeadministratörer."
fakten:
  - label: SPF-syfte
    wert: IP auktoriserar RFC5321.MailFrom eller HELO
    href: https://datatracker.ietf.org/doc/html/rfc7208#section-2
  - label: SPF-publicering
    wert: en TXT-post med v=spf1
    href: https://datatracker.ietf.org/doc/html/rfc7208#section-3
  - label: SPF-DNS-budget
    wert: högst 10 termer som utlöser uppslag
    href: https://datatracker.ietf.org/doc/html/rfc7208#section-4.6.4
  - label: DKIM-syfte
    wert: domänsignatur för utvalda rubriker och brödtext
    href: https://datatracker.ietf.org/doc/html/rfc6376#section-3
  - label: DKIM-nyckel
    wert: selector._domainkey.signing-domain
    href: https://datatracker.ietf.org/doc/html/rfc6376#section-3.6.2.1
  - label: DKIM-kryptografi
    wert: RSA-SHA256 eller Ed25519-SHA256
    href: https://datatracker.ietf.org/doc/html/rfc8463#section-3
  - label: DMARC-standard
    wert: RFC 9989; rapportering i RFC 9990 och 9991
    href: https://datatracker.ietf.org/doc/html/rfc9989
  - label: DMARC-identitet
    wert: en Author Domain från exakt ett From-fält
    href: https://datatracker.ietf.org/doc/html/rfc9989#section-4.2
  - label: DMARC-pass
    wert: SPF eller DKIM består och är aligned
    href: https://datatracker.ietf.org/doc/html/rfc9989#section-4.4
  - label: Alignment
    wert: relaxed som standard; strict valfritt
    href: https://datatracker.ietf.org/doc/html/rfc9989#section-4.4
  - label: Policyer
    wert: none · quarantine · reject
    href: https://datatracker.ietf.org/doc/html/rfc9989#section-4.7
  - label: Driftbevis
    wert: Authentication-Results plus Aggregate Reports
    href: https://datatracker.ietf.org/doc/html/rfc8601
werbung:
  - tools
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: 1247c1771c81b476bf23da2eeee6feb35a3d16c7146e2c61d6c0fa5625c55bbd
translationModel: gpt-5.6-terra
translatedAt: 2026-10-09T11:32:16.382Z
translationReview: automatic
---

# E-postautentisering: SPF, DKIM och DMARC

SPF, DKIM och DMARC besvarar tre olika frågor om den använda avsändardomänen. SPF kontrollerar den sändande IP-adressen, DKIM en kryptografisk signatur och DMARC kopplingen mellan båda resultaten och den synliga From-domänen. Ingen av dessa mekanismer autentiserar en person eller bevisar att ett meddelande är ofarligt. Ett DMARC-pass säger endast att användningen av Author Domain har auktoriserats enligt de publicerade reglerna ([RFC 7208, avsnitt 1](https://datatracker.ietf.org/doc/html/rfc7208#section-1), [RFC 6376, avsnitt 1](https://datatracker.ietf.org/doc/html/rfc6376#section-1), [RFC 9989, avsnitt 1](https://datatracker.ietf.org/doc/html/rfc9989#section-1)).

För meddelandeadministratörer är systemet framför allt en kedja av ansvarsområden. Den utgående tjänsten måste skapa lämpliga envelope-domäner och DKIM-signaturer. Den auktoritativa [DNS](/kb/dns) måste leverera SPF-policy, DKIM-nycklar och DMARC-policy korrekt och i rätt tid. Den mottagande [SMTP](/kb/smtp)-gränsservern behöver den ursprungliga klient-IP-adressen, en resolver, kryptografisk verifiering och en definierad förtroendegräns för resultaten. En rapportinsamlare måste behandla opålitlig XML säkert, identifiera dubbletter och göra data analyserbara över tid. Ett `dmarc=pass` är bara slutresultatet av denna distribuerade pipeline.

Förklaringen följer identiteterna i ett e-postmeddelande: envelope-avsändare, synlig From-domän och DKIM-signatur. SPF och DKIM förklaras först var för sig, därefter kopplar DMARC samman deras resultat via alignment, policy och rapportering.

## Identiteter och förtroendegränser

Ett meddelande innehåller flera avsändarbegrepp som inte kan bytas ut mot varandra. **RFC5321.MailFrom** är SMTP-envelopeets Reverse Path och det primära SPF-objektet. Vid tom Reverse Path, som används för leveransrapporter, härleder SPF identiteten från HELO/EHLO. **RFC5322.From** står i meddelandehuvudet, visas av e-postklienten och ger DMARC dess Author Domain. En DKIM-signatur anger med `d=` sin Signing Domain och med `s=` väljaren för den publika nyckeln. SMTP AUTH autentiserar i sin tur en klient mot en submission-tjänst, men är varken SPF, DKIM eller DMARC ([RFC 5321, avsnitt 3.3 och 4.5.5](https://datatracker.ietf.org/doc/html/rfc5321#section-3.3), [RFC 5322, avsnitt 3.6.2](https://datatracker.ietf.org/doc/html/rfc5322#section-3.6.2), [RFC 7208, avsnitt 2.3 och 2.4](https://datatracker.ietf.org/doc/html/rfc7208#section-2.3), [RFC 6376, avsnitt 3.5](https://datatracker.ietf.org/doc/html/rfc6376#section-3.5)).

| Identitet | Källa | Verifierare | Primärt påstående |
|---|---|---|---|
| anslutande IP-adress | TCP-anslutning vid mottagande MTA | SPF | denna värd är eller är inte auktoriserad för den kontrollerade SMTP-domänen |
| HELO/EHLO-domän | SMTP-dialog | SPF | SMTP-klientens domänidentitet |
| RFC5321.MailFrom-domän | SMTP-envelope | SPF och DMARC | Bounce- respektive Return-Path-domän |
| DKIM `d=` och `s=` | `DKIM-Signature` | DKIM och DMARC | Signing Domain och nyckelväljare |
| RFC5322.From-domän | synligt meddelandehuvud | DMARC | Author Domain som alignment måste upprättas mot |
| `authserv-id` | `Authentication-Results` | interna konsumenter | vilken betrodd verifieringstjänst som skapade resultatet |

Denna separation är en säkerhetsgräns. En angripare kan helt klara SPF och DKIM med en egen domän och ändå ange ett främmande varumärke i visningsnamnet. DMARC begränsar obehörig användning av domänen i det synliga `From:`, inte look-alike-domäner, bedrägeri med visningsnamn, komprometterade legitima avsändare eller skadligt innehåll ([RFC 7208, avsnitt 11.2](https://datatracker.ietf.org/doc/html/rfc7208#section-11.2), [RFC 9989, avsnitt 2.2 och 11.4](https://datatracker.ietf.org/doc/html/rfc9989#section-11.4)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-mail-auth.svg?v=20260813" title="Interaktive Infografik: Identitäten, Alignment, DMARC-Entscheidung, Reporting und indirekte Nachrichtenflüsse" loading="lazy">
  <a href="/images/kb-interaktiv-mail-auth.svg?v=20260813">Öppna infografik om SPF, DKIM och DMARC</a>
</iframe>

## SPF: Auktorisering av den anslutande IP-adressen

Sender Policy Framework är en DNS-baserad auktorisering för domänen i `MAIL FROM` eller HELO. Mottagaren utvärderar klient-IP, den kontrollerade domänen, avsändaridentiteten och det lokala värdnamnet med funktionen `check_host()` som definieras i RFC 7208. Posten finns som en TXT-resurs direkt på den berörda domänen och börjar med `v=spf1`; den avvecklade DNS-RR-typen SPF används inte ([RFC 7208, avsnitt 3.1 och 4.1](https://datatracker.ietf.org/doc/html/rfc7208#section-3.1)).

En post som `v=spf1 ip4:192.0.2.0/24 include:_spf.sender.example -all` utvärderas från vänster till höger. En mekanism som matchar avslutar behandlingen med sin qualifier. `+` betyder `pass` och är standardvärdet, `-` betyder `fail`, `~` betyder `softfail`, `?` betyder `neutral`. `all`, `ip4` och `ip6` kräver inga ytterligare DNS-uppslag under normal utvärdering; `include`, `a`, `mx`, `ptr`, `exists` och `redirect` gör det ([RFC 7208, avsnitt 4.6.1 till 4.6.4](https://datatracker.ietf.org/doc/html/rfc7208#section-4.6)).

| Resultat | Protokollbetydelse | Administratörsfråga |
|---|---|---|
| `pass` | IP-adressen är auktoriserad för denna SPF-identitet | Är exakt denna domän också aligned med RFC5322.From? |
| `fail` | den publicerade policyn auktoriserar inte IP-adressen | fel källa, stulen domänanvändning eller föråldrad policy? |
| `softfail` | svagt negativt påstående från domänen | används den fortfarande i en kontrollerad övergångsfas eller döljer den drift? |
| `neutral` | inget auktoriseringspåstående | saknas en avslutande mekanism eller är neutralitet avsiktlig? |
| `none` | ingen tillämplig SPF-policy | kontrollerades rätt MailFrom-/HELO-domän? |
| `temperror` | tillfälligt fel, vanligtvis DNS | kontrollera resolver, timeout och auktoritativ tillgänglighet |
| `permerror` | posten eller utvärderingen är permanent ogiltig | kontrollera syntax, flera SPF-poster, rekursion och uppslagsbudget |

### Uppslagsbudget och beroenden

SPF begränsar summan av `include`, `a`, `mx`, `ptr`, `exists` och `redirect` över hela den rekursiva utvärderingen till tio termer. Ett överskridande måste ge `permerror`. För `mx` och `ptr` gäller ytterligare adressgränser; fler än två tomma svar eller NXDOMAIN-resultat, så kallade Void Lookups, bör också leda till `permerror` ([RFC 7208, avsnitt 4.6.4](https://datatracker.ietf.org/doc/html/rfc7208#section-4.6.4)).

Budgeten är en körtidsgräns och inte bara en teckenkontroll. Ett enda `include` kan införa ytterligare includes, MX-upplösningar och felområden. `include` delegerar endast frågan om den aktuella värden där uppnår ett `pass`; `redirect` övertar hela policyn för en annan domän efter misslyckad mekanismkontroll. RFC 7208 rekommenderar `include` för att överskrida administrativa gränser och `redirect` snarare för att centralisera domäner som administreras tillsammans ([RFC 7208, avsnitt 5.2 och 6.1](https://datatracker.ietf.org/doc/html/rfc7208#section-5.2)). För drift ska därför ägare, syfte, ändringsväg och uppmätt worst-case-budget för varje externt include finnas i inventariet.

### Vidarebefordran och SPF-domän

En klassisk vidarebefordrare ansluter med sin egen IP-adress till nästa mottagare men behåller det ursprungliga `MAIL FROM`. Därmed kontrolleras en IP-adress mot SPF-policyn för en främmande domän, och SPF kan misslyckas trots att den ursprungliga inlämningen var legitim. RFC 7208 beskriver omskrivning av Reverse Path till en domän som tillhör förmedlaren som motåtgärd; e-postlistor gör ofta detta ändå ([RFC 7208, bilaga D.2](https://datatracker.ietf.org/doc/html/rfc7208#appendix-D.2)). Det åtgärdar SPF-pass för den nya envelope-domänen, men skapar bara DMARC-pass om den nya domänen är aligned med det synliga `From:`. För indirekta vägar är därför en bevarad, aligned DKIM-signatur särskilt viktig.

## DKIM: Signatur för en domän

DomainKeys Identified Mail lägger till ett `DKIM-Signature`-huvudfält. Signeraren kanoniserar utvalda rubriker och brödtexten, bildar body-hashen `bh=`, signerar de fastställda uppgifterna och publicerar nyckeln under `s=._domainkey.d=`. Verifieraren återskapar samma data, hämtar DNS-TXT-nyckeln och kontrollerar signatur och body-hash. Ett pass visar att de signerade komponenterna inte har ändrats på ett identifierbart sätt sedan signeringen och att signeraren kontrollerade den privata nyckeln för Signing Domain. DKIM styrker varken en fysisk person eller innehållets sanningshalt ([RFC 6376, avsnitt 3.5 till 3.8](https://datatracker.ietf.org/doc/html/rfc6376#section-3.5)).

Viktiga taggar är `a=` för algoritm, `c=` för header-/body-kanonisering, `d=` för Signing Domain, `s=` för väljare, `h=` för namnen på signerade rubriker, `bh=` för body-hash och `b=` för signaturen. `t=` och `x=` kan bära signeringstidpunkt och utgångstidpunkt, men är inget tillförlitligt replay-skydd. Det valfria `l=` begränsar det signerade brödtextområdet och kan därmed möjliggöra att osignerat innehåll läggs till; RFC 6376 beskriver uttryckligen denna missbruksrisk ([RFC 6376, avsnitt 3.5 och 8.2](https://datatracker.ietf.org/doc/html/rfc6376#section-8.2)).

Kanoniseringen `simple` tolererar nästan inga ändringar. `relaxed` normaliserar bland annat vissa skrivsätt och whitespace så att vanliga transportändringar inte i onödan bryter en signatur. Rubriker och brödtext kan använda olika metoder; utan `c=` gäller `simple/simple`. Kanonisering ändrar inte det överförda meddelandet, utan endast dess indataform för signering respektive verifiering ([RFC 6376, avsnitt 3.4](https://datatracker.ietf.org/doc/html/rfc6376#section-3.4)).

### Nycklar, algoritmer och rotation

RFC 8301 kräver minst 1024 bitar för RSA, rekommenderar minst 2048 bitar för signerare och förklarar RSA-SHA1 som historisk. RFC 8463 lägger till Ed25519-SHA256 och tillåter parallella signaturer med olika väljare för övergångskompatibilitet ([RFC 8301, avsnitt 3.1 och 3.2](https://datatracker.ietf.org/doc/html/rfc8301#section-3), [RFC 8463, avsnitt 5 och 6](https://datatracker.ietf.org/doc/html/rfc8463#section-5)). Vilken algoritm som används är fortfarande ett interoperabilitetsbeslut: Ett standardiserat krav på verifierarsidan innebär inte automatiskt att varje verklig mottagarplattform implementerar det felfritt.

Väljare skiljer nyckelbyten från domänen. Vid rotation publiceras först den nya offentliga nyckeln, därefter signeras med den nya privata nyckeln och den gamla DNS-nyckeln tas bort först när gamla meddelanden inte längre behöver kunna verifieras regelbundet. RFC 6376 avråder från att återanvända en väljare med en ny nyckel, eftersom gamla signaturfel då inte kan skiljas från förfalskningar. Ett tomt `p=` i nyckelposten återkallar nyckeln ([RFC 6376, avsnitt 3.1 och 6.1.2](https://datatracker.ietf.org/doc/html/rfc6376#section-3.1)).

Den privata nyckeln hör inte hemma i DNS eller i allmänna konfigurationsrepositories. Driftutformningen måste fastställa nyckelägare, generering, skyddad lagring, signerarens åtkomst, rotation, nödåterkallelse, beslut om säkerhetskopiering och revisionsspår. RFC 6376 kräver omsorg vid skydd av privata nycklar och nämner krypterad lagring samt kryptografisk hårdvara som möjliga skyddsåtgärder. Flera sändningsplattformar bör ha separata väljare så att en komprometterad plattform kan isoleras utan globalt nyckelbyte. Denna arkitektur följer den administrativa uppdelning av väljarens namnområde som DKIM avser ([RFC 6376, avsnitt 3.1 och 8.3 samt bilaga C](https://datatracker.ietf.org/doc/html/rfc6376#section-8.3)).

SPF kan misslyckas efter vidarebefordran trots att meddelandet förblev oförändrat. DKIM kan däremot överleva en vidarebefordran men brytas av en ändrad footer. DMARC kopplar därför samman båda metoderna genom så kallad alignment.

## DMARC: Alignment, policy och utvärdering

DMARC kräver ett enda korrekt formaterat RFC5322.From-fält och extraherar exakt en **Author Domain** från det. Det beaktar den SPF-autentiserade domänen och alla framgångsrikt verifierade DKIM-Signing Domains. Ett DMARC-pass föreligger när minst ett SPF- eller DKIM-resultat är `pass` och dess domän är aligned med Author Domain. Båda mekanismerna behöver alltså inte lyckas samtidigt; i drift är båda önskvärda eftersom de kan misslyckas vid olika indirekta flöden ([RFC 9989, avsnitt 4.2 till 4.4 och 5.1](https://datatracker.ietf.org/doc/html/rfc9989#section-4.2)).

Vid **strict alignment** måste domänerna vara identiska. Vid **relaxed alignment** måste de ha samma Organizational Domain. `adkim=s` respektive `aspf=s` kräver strict; utan dessa taggar gäller relaxed. Fastställandet av Organizational Domain sker enligt RFC 9989 genom en begränsad DNS Tree Walk och inte längre enbart genom en Public Suffix List. Walken frågar högst åtta namnivåer och tar hänsyn till `psd=y` respektive `psd=n` ([RFC 9989, avsnitt 4.4 och 4.10](https://datatracker.ietf.org/doc/html/rfc9989#section-4.10)). Denna ändring är relevant med blandade gamla och nya verifierare; RFC 9989 pekar själv på möjliga olika alignment-resultat.

### Policy-post och taggar

DMARC-posten finns under `_dmarc.<domain>` och använder tagg/värde-syntax. `v=DMARC1` identifierar formatet. `p=` beskriver önskad behandling av misslyckade meddelanden från policy-domänen: `none`, `quarantine` eller `reject`. `sp=` kan behandla befintliga underdomäner annorlunda, `np=` icke-existerande underdomäner. `rua=` anger mål för Aggregate Reports, `ruf=` valfria mål för Failure Reports. `t=y` markerar testläget som definieras i RFC 9989. Okända taggar ignoreras ([RFC 9989, avsnitt 4.5 till 4.8](https://datatracker.ietf.org/doc/html/rfc9989#section-4.5)).

Den tidigare procentuella utrullningen med `pct=` ingår inte längre i protokollet enligt RFC 9989. Driftserfarenhet visade inkonsekvent tillämpning; `t=` ersätter endast den tidigare specialeffekten av `pct=0`, inte godtyckliga procentsteg. Stegvis införande måste därför planeras genom separata Author Domains, underdomänpolicyer, riktade e-postflöden och robust rapportutvärdering, inte via en påstått standardiserad procentregulator ([RFC 9989, bilaga A.6](https://datatracker.ietf.org/doc/html/rfc9989#appendix-A.6)).

En policy är en publicerad preferens från Domain Owner. Mottagaren får avvika på grund av lokal policy, rykte, indirekta flöden eller andra insikter och kan dokumentera overrides i rapporter. `p=reject` gör inte automatiskt ett meddelande olevererbart i SMTP-bemärkelse hos varje mottagare; det skapar ett tydligt, automatiserbart påstående för hantering av DMARC-fail ([RFC 9989, avsnitt 5.3 och 5.4](https://datatracker.ietf.org/doc/html/rfc9989#section-5.3)).

En DMARC-policy bör skärpas först när alla legitima sändningsvägar är kända. Aggregate Reports ger data för denna inventering och visar vilka system som faktiskt sänder under en domän.

## Rapportering som operativ datapipeline

RFC 9990 skiljer Aggregate Reporting från DMARC-kärnspecifikationen. En rapport sammanfattar meddelanden efter käll-IP, utvärderad policy, disposition, alignment och autentiseringsresultat. Dataformatet är XML; filen bör komprimeras med GZIP och får då filändelsen `.xml.gz`. Enskilda poster innehåller bland annat `source_ip`, `count`, `header_from`, SPF- och DKIM-resultat samt möjliga orsaker till policy-override ([RFC 9990, avsnitt 3.1 och 3.4](https://datatracker.ietf.org/doc/html/rfc9990#section-3.1)).

En insamlare är därför en säkerhetsrelevant ingest-tjänst. Den tar emot oombedd e-post och bilagor, dekomprimerar främmande data, parsar XML, deduplicerar rapporter och aggregerar volym. Storleksgränser, dekomprimeringsbudget, XML-parser utan externa entiteter, skydd mot skadlig kod, karantän för felaktiga filer, idempotens och lagringstid är delar av arkitekturen. RFC 9989 varnar för avsiktligt felaktiga rapporter och DoS mot rapportmål; RFC 9990 reglerar dubbletter och externa mål ([RFC 9989, avsnitt 11.2](https://datatracker.ietf.org/doc/html/rfc9989#section-11.2), [RFC 9990, avsnitt 3.5.4 och 4](https://datatracker.ietf.org/doc/html/rfc9990#section-3.5.4)).

Om `rua=` ligger utanför Organizational Domain måste rapportmottagaren auktorisera relationen genom en ytterligare DNS-TXT-post. Därmed kan en domän inte översvämma godtyckliga tredje parter med rapporter ([RFC 9990, avsnitt 4](https://datatracker.ietf.org/doc/html/rfc9990#section-4)). Failure Reports enligt RFC 9991 kan avslöja information om enskilda meddelanden. På grund av integritetsriskerna begränsar många operatörer dem; Aggregate Reports är den rekommenderade insynskanalen utan slutanvändarinnehåll ([RFC 9991, avsnitt 7](https://datatracker.ietf.org/doc/html/rfc9991#section-7)).

För administratörsutvärdering är tidsserier viktigare än ett enskilt dagsvärde: volym per käll-IP och Author Domain, aligned SPF, aligned DKIM, DMARC-fail, `temperror`, `permerror`, okända väljare, policy-overrides och rapportörstäckning. Rapporter är observationer från enskilda mottagare och ingen fullständig sändningsredovisning; mottagare är inte skyldiga att leverera varje önskad utvärdering eller disposition ([RFC 9989, avsnitt 1 och 6](https://datatracker.ietf.org/doc/html/rfc9989#section-6)).

## Lita rätt på Authentication-Results

Den mottagande verifieringstjänsten kan lägga SPF-, DKIM- och DMARC-resultat i huvudfältet `Authentication-Results`. `authserv-id` identifierar tjänsten eller den administrativa hanteringsdomän som har verifierat. Detta fält är endast bevisande inom en definierad förtroendegräns. En extern avsändare kan själv infoga ett övertygande `Authentication-Results: ... dmarc=pass` ([RFC 8601, avsnitt 1.5.4 till 1.6](https://datatracker.ietf.org/doc/html/rfc8601#section-1.5.4)).

Border-MTA:n måste därför ta bort främmande instanser eller endast tillåta uttryckligen betrodda producenter. Interna konsumenter behöver en lista över tillåtna `authserv-id`-värden och måste ta hänsyn till positionen i den betrodda `Received`-kedjan respektive intern modell för metadata-proveniens. RFC 8601 varnar uttryckligen för att använda huvudfältet för filterbeslut utan en verifierad, kompatibel Border-MTA ([RFC 8601, avsnitt 5 och 7.1](https://datatracker.ietf.org/doc/html/rfc8601#section-5)).

ARC enligt RFC 8617 kan transportera autentiseringsresultat och efterföljande ändringar över förmedlare i en signerad kedja. ARC publiceras som ett experimentellt protokoll och ger ingen global förtroenderot: Slutmottagaren avgör fortfarande vilka ARC-sealrar den litar på. En giltig ARC-kedjestatus är därför kontext för lokal policy, ingen ersättning för DMARC och inget automatiskt beslut om godkännande ([RFC 8617, avsnitt 1 och 5](https://datatracker.ietf.org/doc/html/rfc8617#section-1)).

Hittills har det handlat om direkt leverans. Vidarebefordringar, listor och gateways ändrar dock IP-adress, envelope eller innehåll – och därmed just indata till de tre kontrollerna.

## Indirekta flöden och typiska brytpunkter

Vidarebefordringar, e-postlistor, säkerhetsgateways och ärendehanteringssystem ändrar olika delar av meddelandet. En vidarebefordran ändrar den anslutande IP-adressen och kan bryta SPF. En e-postlista kan ändra Subject, List-rubriker, brödtext-footer eller MIME-struktur och därmed bryta DKIM. Den kan dessutom skriva om envelope-avsändaren och det synliga `From:`. RFC 7960 beskriver dessa interoperabilitetsproblem och respektive bieffekter; det finns ingen universell korrigering som samtidigt lämnar identitet, listfunktion och befintlig mottagarpolicy oförändrade ([RFC 7960, avsnitt 3 och 4](https://datatracker.ietf.org/doc/html/rfc7960#section-3)).

För felsökning måste administratörer därför jämföra tillståndet **före och efter varje förmedlare**: klient-IP, HELO, MailFrom, RFC5322.From, befintliga DKIM-signaturer, `Authentication-Results`, nya `Received`-fält och innehållsmutationer. Ett slutresultat `dmarc=fail` visar inte ensamt vilket hopp som förlorade den aligned identiteten.

## Teknisk uppbyggnad av en produktionsplattform

E-postautentisering är ingen enskild appliance utan ett distribuerat kontroll- och dataplan:

| Komponent | Teknikstack | Beständigt tillstånd | Centralt felområde |
|---|---|---|---|
| auktoritativ DNS | TXT-RRset, DNSSEC valfritt, zon- eller API-baserad ändring | SPF, DKIM Public Keys, DMARC Policy | föråldrade poster, split-horizon, TTL, felaktig delegering |
| Outbound-Signer | MTA-filter, bibliotek eller gateway; RSA/Ed25519 och SHA-256 | Private Keys, väljarkonfiguration, signeringspolicy | nyckelåtkomst, felaktigt `d=`, saknad signatur på delflöden |
| Inbound-Verifier | Border-MTA/filter, rekursiv resolver, kryptobibliotek, policy-motor | resolvercache, förtroendegräns, lokala overrides | fel klient-IP efter proxy, DNS-timeout, manipulerade Auth-Results |
| DMARC-policy-modul | alignment, DNS Tree Walk, domän-/policylogik | cache och policyversion | gammal RFC-7489-modell, felaktig Organizational Domain |
| Report-Generator | MTA-telemetri, aggregator, XML/GZIP, SMTP-sändning | tidsfönster, poster, rapport-ID | dataförlust, dubbletter, ofullständig rapportörstäckning |
| Report-Collector | brevlåda, dekompressor, XML-parser, databas, instrumentpanel | rårapporter, normalisering, tidsserier | parserangrepp, felaktig deduplicering, okontrollerad lagring |

Programmeringsspråk och produkt kan bytas ut, men inte protokollobjekt och förtroendegränser. En MTA kan implementera signering och kontroll i C, Rust, Java, Go eller genom en separat filterprocess. Avgörande för driften är samma frågor: Varifrån kommer klient-IP efter lastbalanserare? Vilken resolver och cache används? Var finns den privata nyckeln? Vilken process får signera? Vilka rubriker tas bort före förtroendegränsen? Hur korreleras policyversion, DNS-svar, Message-ID, Queue-ID och resultat? RFC:erna definierar wire- och utvärderingssemantik, inte den konkreta processmodellen.

### Kontrollerad införande- och ändringsprocess

RFC 9989 beskriver en robust ordning för Domain Owners: publicera aligned SPF, konfigurera aligned DKIM, skapa en brevlåda för Aggregate Reports, publicera DMARC först i Monitoring Mode med `p=none`, utvärdera rapporter, åtgärda brister och först därefter besluta om enforcement ([RFC 9989, avsnitt 5.1](https://datatracker.ietf.org/doc/html/rfc9989#section-5.1)). Detta leder till ett verifierbart flöde för change management:

1. Inventera alla Author Domains, envelope-domäner, HELO-namn, sändningsprodukter, klientorganisationer, reläer, vidarebefordrare och nyckelägare.
2. Styrk minst en aligned identitet för varje legitim e-postström; SPF och DKIM tillsammans minskar beroendet av en enskild indirekt väg.
3. Gör rapportmottagning och säker utvärdering produktionsklara före `rua=`.
4. Driv `p=none` som en mätfas, men förväxla den inte med skyddseffekt.
5. Klassificera okända källor: legitima och felkonfigurerade, avvecklade, missbrukande eller förvanskade av indirekt flöde.
6. Inför enforcement per kontrollerbar domän, dokumentera undantag och övervaka med rapporter samt leveranstelemetri.
7. Öva regelbundet nyckelrotation, leverantörsbyte, DNS-rollback, rapportbortfall och komprometterad signerare.

Ett godkännande bör inte bara acceptera en syntaktiskt giltig TXT-post. Det kräver testmeddelanden via varje sändningsväg, headerbevis hos mottagaren, rapportdata från flera mottagardomäner, mätning av SPF-budgeten, kontroll av selector-TTL:er och en rollback som inte lämnar Author Domain oskyddad eller olevererbar.

Dessa beroenden leder till en fast diagnosordning: identifiera först använda domäner, kontrollera sedan DNS och signatur och bedöm slutligen alignment och DMARC-policy.

## Diagnosverktyg

Följande frågor använder `example.ch` och exempelväljaren `s2026a`. Produktiva namn och lokalt lagrade meddelanden får endast undersökas i auktoriserade miljöer.

### Läs SPF, DMARC och DKIM i DNS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Mail-Authentifizierungs-DNS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName example.ch -Type TXT -DnsOnly
Resolve-DnsName _dmarc.example.ch -Type TXT -DnsOnly
Resolve-DnsName s2026a._domainkey.example.ch -Type TXT -DnsOnly</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig +noall +answer example.ch TXT
dig +noall +answer _dmarc.example.ch TXT
dig +noall +answer s2026a._domainkey.example.ch TXT</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) och [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) visar TXT-RRseten, inte deras fullständiga protokollutvärdering. Flera Character Strings i en enskild TXT-post måste fogas samman utan ytterligare tecken; flera SPF- eller DMARC-poster under samma namn är däremot ett fel ([RFC 7208, avsnitt 3.2 och 4.5](https://datatracker.ietf.org/doc/html/rfc7208#section-3.2), [RFC 9989, avsnitt 4.5](https://datatracker.ietf.org/doc/html/rfc9989#section-4.5)). För frågor om split-horizon eller DNSSEC måste auktoritativ och rekursiv vy kontrolleras separat.

### Hitta betrodda resultat i meddelandehuvudet

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Header-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Content .\message.eml |
  Select-String -Pattern '^(Authentication-Results|DKIM-Signature|Received):' -Context 0,8</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">grep -E -A8 '^(Authentication-Results|DKIM-Signature|Received):' message.eml</code></pre>
  </div>
</div>

[`Get-Content`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-content) och [`Select-String`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/select-string) respektive [`grep`](https://www.gnu.org/software/grep/manual/grep.html) hjälper vid en första genomgång. På grund av header folding och flera fält ersätter textsökning inte en RFC-kompatibel parser. Avgörande är den betrodda `authserv-id`, dess position i förhållande till den egna `Received`-gränsen, de faktiskt kontrollerade domänerna och att skilja råresultat från alignment.

### Kontrollera Aggregate Report strukturellt

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für DMARC-XML-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">[xml]$report = Get-Content .\report.xml -Raw
$report.SelectNodes("//*[local-name()='record']").Count
$report.SelectSingleNode("//*[local-name()='org_name']").InnerText</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">xmllint --noout report.xml
xmllint --xpath 'count(//*[local-name()="record"])' report.xml
xmllint --xpath 'string(//*[local-name()="org_name"])' report.xml</code></pre>
  </div>
</div>

[`Get-Content`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-content) läser här en redan uppackad lokal XML-fil; [`xmllint`](https://gnome.pages.gitlab.gnome.org/libxml2/xmllint.html) kontrollerar struktur och XPath-frågor. Okända rapporter bör inte öppnas interaktivt med privilegierade skrivbordsverktyg. Denna enskilda kontroll styrker varken schemakonformitet eller säker massbearbetning, dubblettidentifiering eller korrekt aggregering.

### Koppla fel systematiskt till en gräns

| Observation | Trolig orsak | Nästa robusta bevis |
|---|---|---|
| `spf=none` | fel identitet eller ingen SPF-post | MailFrom/HELO från SMTP- respektive Auth-resultatet och TXT på det exakta namnet |
| `spf=permerror` | syntax, flera poster, rekursion eller DNS-budget | utvärdera hela det rekursiva SPF-trädet och räknare för uppslag/Void |
| SPF består, DMARC misslyckas | SPF-domänen är inte aligned | jämför RFC5321.MailFrom med RFC5322.From i konfigurerat alignment-läge |
| `dkim=fail (body hash did not verify)` | brödtext ändrad efter signering | jämför signeringshopp, MIME-normalisering, footer/disclaimer och kanonisering |
| `dkim=temperror` | nyckeluppslag misslyckades tillfälligt | väljarnamn, resolversvar, timeout och auktoritativ DNS-tillgänglighet |
| DKIM består, DMARC misslyckas | endast icke-aligned signatur består | kontrollera alla `d=`-domäner individuellt mot Author Domain |
| Mottagare rapporterar olika DMARC-resultat | olika vägar, DNS-cacher, verifierarmodeller eller mutationer | korrelera samma Message-ID/signatur med ett fullständigt header och DNS-tidpunkt för varje fall |
| okänd käll-IP i Aggregate Reports | ny legitim avsändare, vidarebefordrare eller missbruk | fastställ volym, headerexempel, reverse-/leverantörstilldelning och intern tjänsteägare |
| `Authentication-Results` motsäger varandra | flera verifieringshopp eller förfalskat externt fält | använd endast resultat inom den definierade förtroendegränsen |
| `p=reject`, men meddelandet levereras | lokal override hos mottagaren | kontrollera `disposition`, override-orsak och ytterligare filterresultat |

## Teknisk historik

SPF uppstod ur flera förslag för SMTP-avsändarauktorisering och publicerades 2006 som det experimentella RFC 4408. RFC 7208 flyttade SPF till Standards Track 2014, tog bort den separata DNS-RR-typen SPF och förtydligade bland annat DNS- och Void-gränser ([RFC 4408](https://datatracker.ietf.org/doc/html/rfc4408), [RFC 7208, bilaga B](https://datatracker.ietf.org/doc/html/rfc7208#appendix-B)). Dess arkitektur förblev medvetet orienterad mot SMTP-anslutningen och Reverse Path.

DKIM förenade erfarenheter från DomainKeys och Identified Internet Mail. RFC 4871 standardiserade DKIM 2007; RFC 6376 ersatte det 2011 och skärpte modellen för signatur, nyckel och verifiering. RFC 8301 uppdaterade algoritmer och RSA-nyckellängder 2018, RFC 8463 lade till Ed25519-SHA256 ([RFC 4871](https://datatracker.ietf.org/doc/html/rfc4871), [RFC 6376](https://datatracker.ietf.org/doc/html/rfc6376), [RFC 8301](https://datatracker.ietf.org/doc/html/rfc8301), [RFC 8463](https://datatracker.ietf.org/doc/html/rfc8463)).

DMARC publicerades först 2015 med RFC 7489 som ett informativt dokument. RFC 9989 ersatte 2026 RFC 7489 och PSD-utvidgningen RFC 9091 som Standards Track-specifikation; Aggregate Reporting och Failure Reporting flyttades samtidigt ut till RFC 9990 och RFC 9991. Övergången införde bland annat DNS Tree Walk, `np`, `psd` och `t` samt tog bort `pct` ([RFC 9989, bilaga C](https://datatracker.ietf.org/doc/html/rfc9989#appendix-C), [RFC 9990](https://datatracker.ietf.org/doc/html/rfc9990), [RFC 9991](https://datatracker.ietf.org/doc/html/rfc9991)).

Den maskinläsbara vidarebefordran av verifieringsresultat utvecklades från RFC 5451 via RFC 7001 och RFC 7601 till RFC 8601. ARC publicerades 2019 genom RFC 8617 som ett experimentellt försök att vidarebefordra autentiseringsresultat från indirekta flöden signerat ([RFC 8601, avsnitt 6](https://datatracker.ietf.org/doc/html/rfc8601#section-6), [RFC 8617](https://datatracker.ietf.org/doc/html/rfc8617)). Denna historik förklarar varför verkliga plattformar parallellt kan visa gamla DMARC-taggar, PSL-baserade Organizational Domains, olika DKIM-algoritmer och olika förtroendemodeller för Auth-Results.

## Källor

- [RFC 7208 – Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208) – SPF-identiteter, postutvärdering, resultat och DNS-gränser.
- [RFC 6376 – DomainKeys Identified Mail Signatures](https://datatracker.ietf.org/doc/html/rfc6376) – modell för DKIM-signatur, nyckel och verifiering.
- [RFC 9989 – Domain-Based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc9989) – DMARC-kärnprotokoll, alignment, policy, DNS Tree Walk och drift.
- [RFC 5321, avsnitt 3.3 och 4.5.5](https://datatracker.ietf.org/doc/html/rfc5321)
- [RFC 5322, avsnitt 3.6.2](https://datatracker.ietf.org/doc/html/rfc5322)
- [RFC 8301 – DKIM Cryptographic Algorithm and Key Usage Update](https://datatracker.ietf.org/doc/html/rfc8301) – SHA-256 och RSA-nyckellängder.
- [RFC 8463 – Ed25519-SHA256 for DKIM](https://datatracker.ietf.org/doc/html/rfc8463) – ytterligare signatur- och nyckelalgoritm.
- [RFC 9990 – DMARC Aggregate Reporting](https://datatracker.ietf.org/doc/html/rfc9990) – XML-datamodell, transport, dubbletter och externa rapportmål.
- [RFC 9991 – DMARC Failure Reporting](https://datatracker.ietf.org/doc/html/rfc9991) – Failure Reports per meddelande och integritetsskydd.
- [RFC 8601 – Authentication-Results](https://datatracker.ietf.org/doc/html/rfc8601) – headerformat, `authserv-id` och förtroendegräns.
- [RFC 8617 – Authenticated Received Chain](https://datatracker.ietf.org/doc/html/rfc8617) – experimentell ARC-kedja för indirekta flöden.
- [RFC 7960, avsnitt 3 och 4](https://datatracker.ietf.org/doc/html/rfc7960)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – DNS-frågor i Windows.
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html) – DNS-frågor i Unix-system.
- [Microsoft Learn – Get-Content](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-content) – analys av lokala filer och headers i Windows.
- [Microsoft Learn – Select-String](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/select-string) – mönsteranalys i PowerShell.
- [GNU grep manual](https://www.gnu.org/software/grep/manual/grep.html) – textsökning i Linux och Unix.
- [libxml2 – xmllint](https://gnome.pages.gitlab.gnome.org/libxml2/xmllint.html) – XML- och XPath-kontroll.
- [RFC 4408 – Sender Policy Framework, Experimental](https://datatracker.ietf.org/doc/html/rfc4408) – föregångare till RFC 7208.
- [RFC 4871 – DomainKeys Identified Mail Signatures](https://datatracker.ietf.org/doc/html/rfc4871) – föregångare till RFC 6376.
