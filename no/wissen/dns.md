---
title: "DNS: oppløsning, delegering og drift"
blatt: "dns"
description: "Domain Name System sett fra en administrators perspektiv: navnerom, soner og delegering, rekursiv oppløsning, RRset, cache og TTL, negative svar, UDP og TCP, EDNS, DNSSEC, sonereplikering samt diagnostikk for meldingsinfrastrukturer."
fakten:
  - label: Fullt navn
    wert: Domain Name System
    href: https://datatracker.ietf.org/doc/html/rfc1034
  - label: Grunnmodell
    wert: distribuert hierarkisk navnerom og datagrunnlag
    href: https://datatracker.ietf.org/doc/html/rfc9499#section-2
  - label: Kjernestandarder
    wert: RFC 1034 · RFC 1035 · STD 13
    href: https://www.rfc-editor.org/info/std13
  - label: Spørringsnøkkel
    wert: QNAME · QTYPE · QCLASS
    href: https://datatracker.ietf.org/doc/html/rfc1035#section-4.1.2
  - label: Dataenhet
    wert: Resource Record Set (RRset)
    href: https://datatracker.ietf.org/doc/html/rfc9499#section-5
  - label: Serverroller
    wert: autoritativ · rekursiv · videresender
    href: https://datatracker.ietf.org/doc/html/rfc9499#section-6
  - label: Transport
    wert: UDP og TCP · Port 53
    href: https://datatracker.ietf.org/doc/html/rfc7766
  - label: Utvidelser
    wert: EDNS(0) via OPT
    href: https://datatracker.ietf.org/doc/html/rfc6891
  - label: Konsistens
    wert: TTL-styrte positive og negative cacher
    href: https://datatracker.ietf.org/doc/html/rfc2308
  - label: Integritet
    wert: "DNSSEC: DNSKEY · DS · RRSIG · NSEC"
    href: https://datatracker.ietf.org/doc/html/rfc4034
  - label: Sonesynkronisering
    wert: NOTIFY · AXFR · IXFR
    href: https://datatracker.ietf.org/doc/html/rfc1996
  - label: E-posttilknytning
    wert: MX · PTR · TXT og avledede målnavn
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-5
werbung:
  - tools
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: f68d5b35d0c022538eb216baafcdf1c277fffbe2c2db0ed4a3b519c01ba63062
translationModel: gpt-5.6-terra
translatedAt: 2026-10-04T10:39:49.656Z
translationReview: required
---

# DNS: oppløsning, delegering og drift

DNS svarer på spørsmålet om hvilken informasjon som er publisert for et bestemt navn, og hvem som er ansvarlig for det. Systemet er samtidig et hierarkisk navnerom, en distribuert database og en binær spørreprotokoll. Det leverer ikke bare IP-adresser, men også navneservere, e-postmål, tjenesteendepunkter, nøkler og retningslinjer. De som bare betrakter DNS som «navneoppløsning», overser derfor nettopp postene som meldingsutveksling og identitet er avhengige av ([RFC 9499](https://datatracker.ietf.org/doc/html/rfc9499)).

Den følgende veien starter ved en applikasjons stub-resolver, følger cache og delegeringer frem til den autoritative serveren og fører deretter svaret tilbake. Denne prosessen gjør det mulig å plassere TTL, Glue, TCP-fallback, DNSSEC, soneadministrasjon og typiske feilbilder i sammenheng.

For meldingsadministratorer er DNS et overordnet styringssystem. En MTA finner neste e-postmål via DNS, avsenderautentisering leser retningslinjer og nøkler, sertifikatprosesser kan bruke DNSSEC-sikrede data, og katalog- eller Kerberos-klienter finner tjenester via SRV-poster. DNS kontrollerer imidlertid ikke om den funne tjenesten er frisk. Et syntaktisk korrekt MX-svar kan peke på en utilgjengelig SMTP-lytter; et vellykket A-oppslag sier ingenting om TLS, autentisering eller applikasjonstilstand.

## Arkitektur og roller

Ansvaret for navn er organisert som et tre. Ved den navnløse roten `.` begynner toppnivådomener som `ch.`, og under dem følger delegerte domener og flere etiketter. Et avsluttende punktum gjør et navn fullt kvalifisert og hindrer lokale søkesuffikser. Denne lille skrivemåten er relevant i drift: `mail.example.ch` kan på en klient suppleres med et søkedomen, mens `mail.example.ch.` ikke kan det.

Et **domene** er en del av navnerommet. En **sone** er derimot den administrativt sammenhengende datamengden som en autoritativ server har lokalt ansvar for. En delegering skiller en underordnet sone fra foreldresonen. Forelderen publiserer da et NS-RRset og, når et navneservernavn ligger innenfor den delegerte underordnede sonen, nødvendige A- eller AAAA-data som **Glue** for å nå den. Det grunnleggende skillet mellom navnerom, soner og delegering stammer fra [RFC 1034](https://datatracker.ietf.org/doc/html/rfc1034#section-4.2); aktuelle begreper oppsummeres i [RFC 9499, seksjon 7](https://datatracker.ietf.org/doc/html/rfc9499#section-7).

Fire logiske roller forklarer oppløsningsveien:

| Rolle | Kunnskap og oppgave | Viktig driftsgrense |
|---|---|---|
| Stub-resolver | mottar applikasjonens forespørsel og videresender den til en konfigurert resolver | søkesuffikser, lokal hosts-fil og klientcache kan påvirke resultatet før selve DNS |
| Rekursiv resolver | leverer et endelig svar fra cache eller ved iterative forespørsler | er tillitsgrense for cache, filtrering, logging og DNSSEC-validering |
| Videresender | overtar rekursive forespørsler fra en annen resolver | flytter oppløsning og observerbarhet til en ytterligere operatør |
| Autoritativ server | svarer fra lokalt lastede soner og setter AA-biten i autoritative svar | kjenner ikke tjenestehelse og skal ikke tilby åpen rekursjon for fremmede navn |

Et produkt kan implementere flere roller, men de bør likevel betraktes separat i drift. En feil i den autoritative tjenesten påvirker publiseringen av egne soner; en feil i den rekursive tjenesten påvirker navneoppløsningen for egne klienter. Felles prosesser, adresser eller feilområder gjør dette skillet vanskeligere.

## Navneoppløsning trinn for trinn

En applikasjon spør normalt ikke selv rot- og autoritative servere. Stub-resolveren overleverer en rekursiv forespørsel til en resolver. Hvis denne mangler en brukbar cacheoppføring, følger den delegeringene fra roten via toppnivådomenet til ansvarlig sone. Hver henvisning angir neste NS-RRset og eventuelt Glue-adresser. Resolveren setter sammen det endelige svaret, validerer eventuelt DNSSEC og cacher resultatet ([RFC 1034, seksjon 4.3](https://datatracker.ietf.org/doc/html/rfc1034#section-4.3)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1016" src="/images/kb-interaktiv-dns-aufloesung.svg?v=20260813" title="Interaktive Infografik: DNS-Auflösung von Stub Resolver über Cache, Root und Delegationen bis zur autoritativen Antwort" loading="lazy">
  <a href="/images/kb-interaktiv-dns-aufloesung.svg?v=20260813">Åpne infografikk om DNS-oppløsning</a>
</iframe>

Den tegnede lineære veien er en kaldstartmodell. I en varm resolvercache finnes vanligvis allerede rot- og TLD-delegeringer, slik at bare en del av trinnene er nødvendige. Også QNAME-minimering, videresending, lokale soner eller aggressive DNSSEC-cacher kan endre den synlige pakkestien. Det avgjørende er fortsatt å tilordne hver observasjon til en rolle: Et ikke-autoritativt cachesvar er ikke bevis på hva den ansvarlige autoritative serveren leverer akkurat nå.

## Teknisk oppbygning av en DNS-melding

På wire-nivå består en DNS-melding av Header, Question, Answer, Authority og Additional Section. Spørsmålet angir QNAME, QTYPE og QCLASS. Svar og henvisninger vises som Resource Records i de øvrige seksjonene. Flagg viser blant annet autoritet, ønske om rekursjon og trunkering; Response Code beskriver resultatet. For analysen teller derfor ikke bare Answer-teksten, men også hvilken server den kom fra og med hvilke flagg ([RFC 1035, seksjon 4.1](https://datatracker.ietf.org/doc/html/rfc1035#section-4.1)).

For diagnostikk er disse spesielt relevante:

| Signal | Betydning | Typisk administratorspørsmål |
|---|---|---|
| `AA` | svaret er autoritativt for navnet som besvares | Ble ansvarlig sone spurt direkte, eller bare en cache? |
| `TC` | svaret ble avkortet for brukt transport | Fungerer gjentakelse over TCP, og tillater brannmuren TCP/53? |
| `RD` / `RA` | rekursjon forespurt / tilbudt av serveren | Ble en autoritativ server ved et uhell brukt som resolver? |
| `AD` | den svarende validatoren anser dataene som autentiserte | Er resolveren pålitelig, og er transporten til den beskyttet? |
| `CD` | klienten krever at valideringsfeil ikke forkastes hos resolveren | Valideres DNSSEC nå, eller undersøkes bare rådata? |
| `RCODE` | resultatstatus som NOERROR, NXDOMAIN, SERVFAIL eller REFUSED | Er navnet feil, typen ikke tilgjengelig, oppløsningen forstyrret eller forespørselen avvist av policy? |

EDNS(0) utvider med en pseudoaktig **OPT-post** ekstra flagg, alternativer og en større annonsert UDP-nyttelast uten å erstatte grunnformatet ([RFC 6891](https://datatracker.ietf.org/doc/html/rfc6891)). DNSSEC-DO-biten befinner seg i dette utvidede flaggfeltet, ikke i den opprinnelige DNS-headeren.

## Resource Records og RRset

Det faglige innholdet ligger i Resource Records. Eiernavn, type og klasse bestemmer hva dataene hører til; TTL og RDATA gir cachetid og typespesifikk verdi. Alle poster med samme eier, type og klasse utgjør et RRset og deler én TTL. Flere MX-, A- eller AAAA-verdier er derfor en samlet cachet mengde, ikke enkeltobjekter som kan styres uavhengig ([RFC 2181, seksjon 5](https://datatracker.ietf.org/doc/html/rfc2181#section-5)).

| Type | Funksjon | Viktig grense |
|---|---|---|
| `SOA` | sonemetadata, Serial, Refresh/Retry/Expire og negativ cacheparameter | én SOA-RR per sone ved apex; endring av Serial styrer synkronisering med sekundærservere |
| `NS` | autoritative servere for en sone eller delegering | målet for en NS kan ikke være et alias |
| `A` / [`AAAA`](https://datatracker.ietf.org/doc/html/rfc3596#section-2.1) | IPv4- eller IPv6-adresse for et navn | sier ingenting om tjenesteport eller tilgjengelighet |
| `CNAME` | alias fra et navn til et kanonisk navn | kan som hovedregel ikke eksistere sammen med andre data hos samme eier |
| `MX` | Mail Exchanger med Preference | målet må kunne løses til A/AAAA og kan ikke være en CNAME |
| `PTR` | omvendt oppslag, vanligvis under `in-addr.arpa.` eller `ip6.arpa.` | omvendt sone tilhører vanligvis adresseinnehaveren, ikke operatøren av den fremoverrettede sonen |
| `TXT` | én eller flere tegnstrenger uten protokollovergripende semantikk | tolkningen oppstår først gjennom SPF, DKIM, DMARC eller en annen mekanisme |
| [`SRV`](https://datatracker.ietf.org/doc/html/rfc2782) | tjeneste, transport, prioritet, vekt, port og mål | klienten må implementere SRV-semantikken for den aktuelle tjenesten |
| [`CAA`](https://datatracker.ietf.org/doc/html/rfc8659) | policy for sertifikatmyndigheter for domenenavn | er verken transportkryptering eller serversertifikat |
| `DS`, `DNSKEY`, `RRSIG`, `NSEC` | DNSSEC-tillitskjede, nøkler, signaturer og autentisert ikke-eksistens | beskytter DNS-dataintegritet, ikke konfidensialiteten |

Teksten i en sonefil er bare **Presentation Form**. På nettverket overføres navn etikett for etikett, tall binært og enkelte navn eventuelt komprimert. Den som kopierer en tegnstreng fra et brukergrensesnitt, bør derfor skille mellom om brukergrensesnittet allerede har satt sammen anførselstegn, escape-sekvenser eller flere TXT-strenger til én logisk nyttelast.

## Delegering, Glue og autoritet

Med en delegering overfører en sone ansvaret for et undertre. Forelderen publiserer NS-RRset for den underordnede sonen; den underordnede sonen oppgir selv sine autoritative servere på nytt. Dersom sidene avviker fra hverandre, kan resolverne velge ulike veier. Glue-adresser i forelderen løser bare høna-og-egget-problemet med tilgjengelighet. De autoritative A-/AAAA-verdiene ved navneservernavnet forblir selvstendige data med egen TTL og vedlikehold.

Særlig kritisk er **in-bailiwick Glue**: Hvis `example.ch.` delegeres til `ns1.example.ch.`, trenger resolveren adressen til `ns1.example.ch.` før den kan spørre den underordnede sonen. Uten Glue oppstår en oppløsningssløyfe. Hvis delegeringen derimot peker på `ns1.provider.net.`, kan adressen finnes via en annen delegeringskjede.

Ved bytte av navneserver må derfor minst fire tilstander kontrolleres: ny sone lastet på alle servere, Child-NS-RRset tilpasset, Parent-delegering tilpasset og nødvendig Glue oppdatert. Først deretter bør gamle servere tas ut av drift eller tilgjengelighet.

## Cache, TTL og negative svar

Etter oppløsning lever et svar videre i cacher. TTL-en er maksimal brukstid fra tidspunktet den aktuelle resolveren lærte det. Det finnes derfor ingen global felles nedtelling. Stub-resolvere, videresendere, rekursive resolvere og applikasjoner kan forkaste samme gamle RRset på ulike tidspunkter. En TTL som senkes før en migrering, hjelper bare for svar som lastes på nytt etterpå; allerede cachede data kan ikke tilbakekalles.

Også ikke-eksistens caches. **NXDOMAIN** betyr at det forespurte navnet ikke finnes; **NODATA** er et NOERROR-svar der navnet finnes, men ikke noe RRset av den forespurte typen. Den autoritative serveren legger ved sin SOA-RR i Authority Section i begge svarene. Den negative cachetiden er minimum av SOA-TTL og SOA.MINIMUM ([RFC 2308, seksjon 3 til 5](https://datatracker.ietf.org/doc/html/rfc2308#section-3)). Dette forklarer hvorfor en nyopprettet DKIM-selektor eller et vertsnavn etter et tidligere mislykket forsøk fortsatt først kan fremstå som ikke-eksisterende.

`SERVFAIL` må skilles fra dette: Resolveren klarte ikke å opprette et brukbart svar. Årsaker er blant annet tidsavbrudd mot autoritative servere, en ødelagt delegering, en DNSSEC-valideringsfeil eller en intern ressursbegrensning. `REFUSED` betyr derimot at den forespurte serveren ikke utfører operasjonen på grunn av sin policy. Et diagnoseverktøy må derfor vise RCODE, AA-bit, svarende server og seksjoner; en utdata som bare melder «ingen adresse», visker ut avgjørende forskjeller.

## Transport: UDP, TCP og krypterte resolverbaner

For denne utvekslingen må nettverksbanen tillate UDP og TCP på port 53. TCP er ikke begrenset til soneoverføringer: En resolver kan bruke det direkte og må kunne falle tilbake til det etter et avkortet UDP-svar. Blokkert TCP blir derfor ofte først synlig ved store, DNSSEC-rike eller svarrike RRset. Små A-forespørsler forblir grønne og gir et feilaktig bilde ([RFC 7766](https://datatracker.ietf.org/doc/html/rfc7766)).

Uten EDNS er DNS-nyttelasten over UDP begrenset til 512 byte. EDNS lar den spørrende annonsere en større mottakbar nyttelast. En for stor verdi kan imidlertid tvinge frem IP-fragmentering; dersom en bane mister eller blokkerer fragmenter, oppstår det typiske bildet der små svar fungerer og store svar får tidsavbrudd. EDNS-spesifikasjonen anbefaler å ta hensyn til faktisk mottakskapasitet og banen, og ved problemer falle tilbake til mindre verdier eller TCP ([RFC 6891, seksjon 6.2](https://datatracker.ietf.org/doc/html/rfc6891#section-6.2)).

[DNS over TLS](https://datatracker.ietf.org/doc/html/rfc7858), [DNS over HTTPS](https://datatracker.ietf.org/doc/html/rfc8484) eller [DNS over QUIC](https://datatracker.ietf.org/doc/html/rfc9250) krypterer en DNS-transportbane. Disse prosedyrene endrer verken soneinnhold eller delegering og erstatter ikke DNSSEC: Transportkryptering beskytter forbindelsen til en resolver, mens DNSSEC autentiserer data langs delegeringskjeden. Klassisk DNS på port 53 er ukryptert; den felles terminologien for transporttypene er definert i [RFC 9499, seksjon 6](https://datatracker.ietf.org/doc/html/rfc9499#section-6).

## DNSSEC og tillitskjeden

DNSSEC utvider navneoppløsningen med verifiserbar opprinnelse og integritet. En sone signerer RRset med RRSIG og publiserer de offentlige nøklene som DNSKEY. Forelderen kobler den underordnede sonen via DS til den overliggende tillitskjeden; NSEC eller NSEC3 kan også bevise ikke-eksistens. En validerende resolver starter ved sitt Trust Anchor og kontrollerer denne kjeden frem til svaret. Innholdet forblir offentlig: DNSSEC krypterer ingen forespørsel ([RFC 4033](https://datatracker.ietf.org/doc/html/rfc4033), [RFC 4034](https://datatracker.ietf.org/doc/html/rfc4034)).

For drift er ikke bare nøkkelfiler relevante, men flere tidsmessig koblede tilstander:

- RRSIG-er har start og utløp; feil systemtid eller manglende re-signering kan gjøre en hel sone **bogus**.
- En DS i forelderen må samsvare med en brukbar DNSKEY i den underordnede sonen. En foreldreløs DS er verre for validerende resolvere enn en bevisst usignert delegering.
- Ved rollover må publisering, signering, endring i forelderen, TTL-er og cachetider planlegges som en tilstandsmaskin.
- En validerende resolver leverer ofte SERVFAIL for bogus data. En test uten validering kan samtidig vise et tilsynelatende normalt svar.

AD-flagget alene er bare så pålitelig som resolveren og veien til den. For en uavhengig kontroll må en administrator undersøke kjeden med et validerende verktøy og lokalisere den feilende overgangen mellom DS, DNSKEY og RRSIG.

## Autoritativ drift og dataflyt

Bak det autoritative svaret ligger en egen distribusjonsvei. I en klassisk modell leder en primærkilde sonen, DNS NOTIFY informerer sekundærservere om en ny SOA-Serial, og AXFR eller IXFR overfører henholdsvis komplette eller inkrementelle data. En frisk lytter beviser derfor ikke at serveren har lastet den nye sonen. Serial, overføringsstatus og svar fra hver autoritative node må sees samlet ([RFC 1996](https://datatracker.ietf.org/doc/html/rfc1996), [RFC 5936](https://datatracker.ietf.org/doc/html/rfc5936), [RFC 1995](https://datatracker.ietf.org/doc/html/rfc1995)).

Sonen kan genereres fra tekstfiler, en database, et API, Active Directory eller en Git-/CI-pipeline. Dette implementeringsvalget endrer ikke DNS-wire-protokollen, men bestemmer transaksjonsgrenser, revisjonsmulighet og gjenoppretting. RFC 2136 definerer atomiske dynamiske oppdateringer med Prerequisites; et leverandør-API-kall er derimot en separat styringsprotokoll og må dokumentere sine egne konsistens- og feilregler ([RFC 2136](https://datatracker.ietf.org/doc/html/rfc2136)).

En DNS-sikkerhetskopi er først brukbar når den kan gjenopprette sonen som faktisk leveres. Avhengig av plattform inkluderer dette:

- sone eller kildedatabase, inkludert SOA-Serial og dynamisk journal;
- serverkonfigurasjon, Views, ACL-er, videresendere og katalogtilordning;
- TSIG-secrets, DNSSEC-private nøkler og tilstander for automatiske rollover;
- foreldedata utenfor egen sone, særlig delegering, Glue og DS;
- en testet vei for å forsyne sekundærservere på nytt og kontrollere sonedata semantisk.

Sekundærservere er tilgjengelighetskopier, men ikke automatisk historiske sikkerhetskopier. En feilaktig eller ondsinnet endring kan raskt replikeres til alle autoritative servere via NOTIFY og soneoverføring.

## Implementerings- og teknologistack

DNS betegner ikke én enkelt daemon. Den felles teknologistacken består av navn og RRset, binært Query-/Response-format, UDP og TCP, cachelogikk og eventuelt DNSSEC. Autoritative servere, rekursive resolvere, videresendere og administrert DNS implementerer disse byggesteinene forskjellig. For drift teller derfor først et produkts rolle, deretter språk eller pakking:

| Implementering | Primær rolle | Teknisk fokus |
|---|---|---|
| [BIND 9](https://bind9.readthedocs.io/en/latest/) | autoritativ og/eller rekursiv | universell navneserver, sonefiler, dynamiske oppdateringer, DNSSEC og diagnoseverktøy |
| [Unbound](https://unbound.docs.nlnetlabs.nl/en/latest/manpages/unbound.html) | rekursiv cache-resolver | modulær resolver-, validator- og cache-pipeline; ingen autoritativ hovedrolle |
| [Knot DNS](https://www.knot-dns.cz/) | autoritativ | kun autoritativ, parallell behandling, soneoverføring, DDNS og DNSSEC |
| [Windows Server DNS](https://learn.microsoft.com/windows-server/networking/dns/dns-overview) | autoritativ og rekursiv | valgfrie AD-integrerte soner, sikre dynamiske oppdateringer, policyer, cache og videresending |

Skillet mellom rollene er viktigere enn produktnavnet. For en offentlig autoritativ plattform teller sonedistribusjon, sekundærservere, DNSSEC-signering og DDoS-resiliens; for en bedriftsresolver teller cache, videresending, interne navnerom, policy, personvern og validering.

## DNS i e-post- og identitetsdrift

I meldingsutveksling blir et DNS-svar umiddelbart en leveringsvei. En sendende MTA spør MX-RRset for mottakerdomenet, foretrekker den laveste verdien og behandler like preferanser som likeverdige. Bare hvis ingen MX finnes, gjelder domenet selv som implisitt mål. Når MX-poster finnes, er en A-/AAAA-verdi ved domene-apex ikke en erstatning. Hvert MX-mål trenger egne adresser og kan ikke være et alias; et domene uten e-postmottak publiserer Null MX `0 .` ([RFC 5321, seksjon 5](https://datatracker.ietf.org/doc/html/rfc5321#section-5), [RFC 2181, seksjon 10.3](https://datatracker.ietf.org/doc/html/rfc2181#section-10.3), [RFC 7505](https://datatracker.ietf.org/doc/html/rfc7505)).

PTR-poster løses via det omvendte adressetreet. Fremover- og reverssone har ofte ulike eiere; endringer må derfor koordineres mellom domene- og IP-adresseoperatør. En PTR er et navn, ikke et kryptografisk identitetsbevis. Mottakende e-postplattformer kan bruke konsistent fremover-/reversoppløsning som et signal, men deres konkrete omdømme- eller akseptpolicy er ingen DNS-egenskap.

SPF, DKIM og DMARC bruker DNS som publiseringskanal, men definerer egne evalueringsregler. SPF leser én logisk TXT-nyttelast og begrenser DNS-utløsende termer; DKIM adresserer nøkler via selektorer; DMARC ligger under `_dmarc`. Disse mekanismene hører faglig hjemme i [SPF, DKIM og DMARC](/kb/mail-auth). DNS-administratoren må for dem først og fremst beherske eiernavn, TXT-strengoppdeling, svarstørrelse, TTL, delegering og negative cachetider korrekt.

Også [LDAP](/kb/ldap) og [Kerberos](/kb/kerberos) bruker ofte SRV-poster for tjenesteoppdagelse. En SRV-post inneholder i tillegg til mål og port en prioritet og en vekt. Disse verdiene er ikke en universell konfigurasjon for lastbalansering; bare klienter som implementerer den aktuelle SRV-prosedyren, tolker dem.

## Diagnostikk

Diagnostikken begynner derfor aldri med «DNS fungerer ikke», men med navn, type, klasse, forespurt server, transport og tidspunkt. Bedriftsresolovere, offentlige resolvere og autoritative servere kan midlertidig eller gjennom Split DNS og policy levere ulike svar. Dette avviket er ikke målingsstøy, men det viktigste tegnet på hvor oppløsningsveien divergerer.

### Spør RRset målrettet

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für gezielte DNS-Abfragen">
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

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) kan tvinge frem en bestemt server, posttype, kun DNS og TCP. [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) viser i tillegg flagg, seksjoner, RCODE, svarende server og spørringstid. Uten eksplisitt `-Server` eller `@server` testes den konfigurerte resolveren, ikke nødvendigvis den autoritative kilden.

### Test TCP og DNSSEC separat

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für DNS-Transport- und DNSSEC-Prüfung">
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

[`delv`](https://bind9.readthedocs.io/en/latest/manpages.html#delv-dns-lookup-and-validation-utility) bruker BIND-resolver- og validatorlogikken for å kontrollere en DNSSEC-kjede. En vellykket forespørsel med `+dnssec` beviser derimot bare at DNSSEC-data ble forespurt og levert; den validerer ikke automatisk kjeden. Den separate TCP-testen avdekker brannmurer som tillater UDP/53, men blokkerer TCP/53.

### Tøm lokale cacher målrettet

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem zum Leeren des lokalen DNS-Caches">
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

[`Clear-DnsClientCache`](https://learn.microsoft.com/powershell/module/dnsclient/clear-dnsclientcache) tømmer cachen til Windows DNS-klienten. [`resolvectl`](https://man7.org/linux/man-pages/man1/resolvectl.1.html) styrer cachen til `systemd-resolved`; på systemer med `nscd`, `dnsmasq`, en lokal Unbound eller en applikasjonscache er en annen cache ansvarlig. En tømming av klienten endrer aldri cachen til en oppstrømsresolver.

### Tilordne feilbildet til overleveringspunktet

| Observasjon | Sannsynlig nivå | Neste kontroll |
|---|---|---|
| en resolver leverer gammel verdi, autoritative servere den nye | positiv cache | gjenstående TTL, videresenderkjede, applikasjonscache |
| NXDOMAIN vedvarer etter at posten er opprettet | negativ cache eller feil sone | SOA i negativt svar, foreldredelegering, eiernavn |
| NOERROR uten Answer | navnet finnes, forespurt type mangler | CNAME-kjede, nøyaktig QTYPE, NODATA-SOA |
| bare store eller signerte svar får tidsavbrudd | EDNS, fragmentering eller TCP-fallback | `TC`, mindre EDNS-størrelse, eksplisitt TCP-test, brannmur |
| validerende resolver leverer SERVFAIL, ikke-validerende et svar | DNSSEC bogus | DS/DNSKEY-samsvar, RRSIG-tider, algoritme, systemtid |
| autoritative servere leverer ulike serialer | replikering | NOTIFY, IXFR/AXFR, overførings-ACL, journal, primærkilde |
| offentlige og interne klienter ser ulike mål | Split DNS eller resolverpolicy | forespurt resolver, View-tilordning, klientundernett, videresender |
| MX finnes, levering feiler før SMTP | avledet målnavn eller transport | MX-Preference, A/AAAA for MX-målet, TCP/25, CNAME-forbud |

## Overvåking og driftskriterier

Overvåking må gjenspeile samme vei. Ett enkelt A-oppslag mot lokal standardresolver oppdager verken en ødelagt delegering, blokkert TCP, utløpte signaturer eller en sekundærserver med gammel Serial. For meldingsutveksling og identitet bør derfor minst følgende signaler samles inn separat:

- autoritative svar fra hver publiserte NS over UDP og TCP, inkludert AA-bit og SOA-Serial;
- Parent-delegering, Child-NS-RRset, Glue og ved signerte soner DS/DNSKEY-overgangen;
- rekursiv latenstid, cache-treffforhold, frekvens for Timeout, SERVFAIL, REFUSED og NXDOMAIN;
- svarstørrelse, trunkering, EDNS-feil og TCP-fallback;
- utløpstidspunkter for RRSIG-er, status for key rollover og signeringskøer;
- NOTIFY-, AXFR- og IXFR-suksess samt sonealder på hver sekundærserver;
- faglige RRset som MX, tilhørende A/AAAA-adresser, PTR og TXT-navnene som kreves for [e-postautentisering](/kb/mail-auth).

En syntetisk test bør kontrollere både den vanlige klientveien og den autoritative kilden. Bare slik kan man skille mellom om en feil ligger i det publiserte datasettet, delegeringen, en resolvercache eller applikasjonen.

## Sikkerhet og feilområder

De to serverrollene krever ulike beskyttelsestiltak. Rekursjon skal bare være tilgjengelig for pålitelige klienter; en åpen resolver kan misbrukes til refleksjons- og forsterkningsangrep. Autoritative servere må derimot forbli tilgjengelige globalt, men må ikke løse vilkårlige fremmede navn rekursivt. Å blande roller øker angrepsflaten og gjør lastårsaker vanskeligere å oppdage ([RFC 5358](https://datatracker.ietf.org/doc/html/rfc5358)).

DNSSEC beskytter publiserte RRset mot uoppdaget endring, men ikke serverprosessen eller tilgjengeligheten. **TSIG** autentiserer enkeltstående DNS-meldinger med en delt nøkkel og brukes blant annet for oppdateringer og soneoverføringer; det er ingen offentlig signatur av sonedataene ([RFC 8945](https://datatracker.ietf.org/doc/html/rfc8945)). Overførings-ACL-er, TSIG-secrets og DNSSEC-private nøkler er separate beskyttelsesobjekter.

Split DNS og policy-svar kan være nødvendige, men skaper flere sannheter for samme QNAME/QTYPE. Minst tilordningskriterium, kildesone, videresendingsvei, DNSSEC-atferd og overvåking per View må dokumenteres. Ellers blir et ønsket avvik tolket som cachefeil eller replikeringsproblem ved neste feil.

## Teknisk historie

Før DNS distribuerte Internett sentralt vedlikeholdte vertstabeller. Med flere nettverk, verter og uavhengige operatører ble denne metoden en flaskehals. Paul Mockapetris beskrev i 1983 en hierarkisk delegerbar navnetjeneste i RFC 882 og RFC 883. RFC 1034 og RFC 1035 erstattet denne versjonen i 1987 og utgjør fortsatt kjernen i systemet som STD 13.

Den første fungerende serverimplementeringen **Jeeves** kjørte i 1983/84 på DEC-TOPS-20-systemer. Kort tid etter oppsto Berkeley Internet Name Domain Package **BIND** for Unix ved University of California, Berkeley, med DARPA-finansiering. BIND 8 kom i 1997, og BIND 9 i september 2000 som en omfattende nyutvikling. Denne utviklingshistorien dokumenteres av ISC i [A Brief History of the DNS and BIND](https://bind9.readthedocs.io/en/latest/history.html).

Protokollen vokste gradvis uten å erstatte den hierarkiske kjernen: NOTIFY og IXFR fremskyndet soneavstemming på 1990-tallet, EDNS utvidet meldingsmodellen i 1999 og ble senere konsolidert i RFC 6891, DNSSEC fikk de grunnleggende DNSKEY-/DS-/RRSIG-mekanismene som gjelder i dag i 2005, og krypterte resolvertransporter kom til med DoT, DoH og DoQ. DNS er derfor ikke en frosset protokoll fra 1987, men et utvidbart system der bakoverkompatibilitet og lange cachetilstander preger enhver endring i drift.

## Kilder

- [RFC Editor – STD 13: Domain Name System](https://www.rfc-editor.org/info/std13)
- [RFC 9499 – DNS Terminology](https://datatracker.ietf.org/doc/html/rfc9499) – aktuelle begreper for roller, soner, cache, DNSSEC og transport.
- [RFC 1034 – Domain Names: Concepts and Facilities](https://datatracker.ietf.org/doc/html/rfc1034) – navnerom, soner, delegering, resolvere og serverroller.
- [RFC 1035 – Domain Names: Implementation and Specification](https://datatracker.ietf.org/doc/html/rfc1035) – wire-format, Resource Records, meldingsseksjoner og masterfiler.
- [RFC 6891 – Extension Mechanisms for DNS (EDNS(0))](https://datatracker.ietf.org/doc/html/rfc6891) – OPT, utvidede flagg og UDP-nyttelast.
- [RFC 2181, seksjon 5](https://datatracker.ietf.org/doc/html/rfc2181)
- [`AAAA`](https://datatracker.ietf.org/doc/html/rfc3596)
- [RFC 2782 – A DNS RR for specifying the location of services](https://datatracker.ietf.org/doc/html/rfc2782) – oppbygning og valg av SRV-poster.
- [RFC 8659 – DNS Certification Authority Authorization](https://datatracker.ietf.org/doc/html/rfc8659) – CAA-poster og evaluering av sertifikatmyndigheter.
- [RFC 2308 – Negative Caching of DNS Queries](https://datatracker.ietf.org/doc/html/rfc2308) – NXDOMAIN, NODATA, SOA og negativ cachetid.
- [RFC 7766 – DNS Transport over TCP](https://datatracker.ietf.org/doc/html/rfc7766) – obligatorisk TCP-støtte og forbindelsesatferd.
- [RFC 7858 – DNS over TLS](https://datatracker.ietf.org/doc/html/rfc7858) – DNS over TLS.
- [RFC 8484 – DNS Queries over HTTPS](https://datatracker.ietf.org/doc/html/rfc8484) – DNS over HTTPS.
- [RFC 9250 – DNS over Dedicated QUIC Connections](https://datatracker.ietf.org/doc/html/rfc9250) – DNS over QUIC.
- [RFC 4033 – DNS Security Introduction and Requirements](https://datatracker.ietf.org/doc/html/rfc4033) – DNSSEC-beskyttelsesmål, validering og grenser.
- [RFC 4034 – Resource Records for DNSSEC](https://datatracker.ietf.org/doc/html/rfc4034) – DNSKEY, DS, RRSIG og NSEC.
- [RFC 1996 – DNS NOTIFY](https://datatracker.ietf.org/doc/html/rfc1996) – varsling av sekundærservere.
- [RFC 5936 – DNS Zone Transfer Protocol (AXFR)](https://datatracker.ietf.org/doc/html/rfc5936) – komplette soneoverføringer over TCP.
- [RFC 1995 – Incremental Zone Transfer (IXFR)](https://datatracker.ietf.org/doc/html/rfc1995) – inkrementell soneavstemming.
- [RFC 2136 – Dynamic Updates in DNS](https://datatracker.ietf.org/doc/html/rfc2136) – atomiske endringer med Prerequisites.
- [BIND 9](https://bind9.readthedocs.io/en/latest/)
- [Unbound Documentation – unbound(8)](https://unbound.docs.nlnetlabs.nl/en/latest/manpages/unbound.html) – rekursiv cache og DNSSEC-validering.
- [Knot DNS](https://www.knot-dns.cz/) – kun autoritativ implementering og driftsfunksjoner.
- [Microsoft Learn – DNS in Windows Server](https://learn.microsoft.com/windows-server/networking/dns/dns-overview) – Windows-DNS-roller, AD-integrasjon, cache og videresending.
- [RFC 5321, seksjon 5](https://datatracker.ietf.org/doc/html/rfc5321)
- [RFC 7505 – Null MX](https://datatracker.ietf.org/doc/html/rfc7505) – eksplisitt merking av domener uten e-postmottak.
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – DNS-forespørsler i Windows.
- [ISC BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html) – forespørselsalternativer, servervalg og tolkning av utdata.
- [`Clear-DnsClientCache`](https://learn.microsoft.com/powershell/module/dnsclient/clear-dnsclientcache)
- [`resolvectl`](https://man7.org/linux/man-pages/man1/resolvectl.1.html)
- [RFC 5358 – Preventing Use of Recursive Nameservers in Reflector Attacks](https://datatracker.ietf.org/doc/html/rfc5358) – rekursjonsbegrensning og misbruksbeskyttelse.
- [RFC 8945 – Secret Key Transaction Authentication for DNS (TSIG)](https://datatracker.ietf.org/doc/html/rfc8945) – meldingsautentisering for oppdateringer og overføringer.
- [ISC – A Brief History of the DNS and BIND](https://bind9.readthedocs.io/en/latest/history.html) – Jeeves, Berkeley, BIND 8 og BIND 9.
