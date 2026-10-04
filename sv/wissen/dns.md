---
title: "DNS: upplösning, delegering och drift"
blatt: "dns"
description: "Domain Name System ur ett administratörsperspektiv: namnrymd, zoner och delegering, rekursiv upplösning, RRset, cache och TTL, negativa svar, UDP och TCP, EDNS, DNSSEC, zonreplikering samt diagnostik för meddelandeinfrastrukturer."
fakten:
  - label: Fullständigt namn
    wert: Domain Name System
    href: https://datatracker.ietf.org/doc/html/rfc1034
  - label: Grundmodell
    wert: distribuerad hierarkisk namnrymd och datamängd
    href: https://datatracker.ietf.org/doc/html/rfc9499#section-2
  - label: Kärnstandarder
    wert: RFC 1034 · RFC 1035 · STD 13
    href: https://www.rfc-editor.org/info/std13
  - label: Frågenyckel
    wert: QNAME · QTYPE · QCLASS
    href: https://datatracker.ietf.org/doc/html/rfc1035#section-4.1.2
  - label: Dataenhet
    wert: Resource Record Set (RRset)
    href: https://datatracker.ietf.org/doc/html/rfc9499#section-5
  - label: Serverroller
    wert: auktoritativ · rekursiv · vidarebefordrare
    href: https://datatracker.ietf.org/doc/html/rfc9499#section-6
  - label: Transport
    wert: UDP och TCP · Port 53
    href: https://datatracker.ietf.org/doc/html/rfc7766
  - label: Utökningar
    wert: EDNS(0) via OPT
    href: https://datatracker.ietf.org/doc/html/rfc6891
  - label: Konsistens
    wert: TTL-styrda positiva och negativa cachar
    href: https://datatracker.ietf.org/doc/html/rfc2308
  - label: Integritet
    wert: "DNSSEC: DNSKEY · DS · RRSIG · NSEC"
    href: https://datatracker.ietf.org/doc/html/rfc4034
  - label: Zonsynkronisering
    wert: NOTIFY · AXFR · IXFR
    href: https://datatracker.ietf.org/doc/html/rfc1996
  - label: E-postanknytning
    wert: MX · PTR · TXT och härledda målnamn
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-5
werbung:
  - tools
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: f68d5b35d0c022538eb216baafcdf1c277fffbe2c2db0ed4a3b519c01ba63062
translationModel: gpt-5.6-terra
translatedAt: 2026-10-04T10:38:40.359Z
translationReview: required
---

# DNS: upplösning, delegering och drift

DNS besvarar frågan om vilken information som publiceras för ett visst namn och vem som ansvarar för den. Systemet är samtidigt en hierarkisk namnrymd, en distribuerad databas och ett binärt frågeprotokoll. Det tillhandahåller inte bara IP-adresser, utan även namnservrar, e-postmål, tjänsteslutpunkter, nycklar och policyer. Den som endast ser DNS som «namnupplösning» missar därför just de poster som meddelandehantering och identitet är beroende av ([RFC 9499](https://datatracker.ietf.org/doc/html/rfc9499)).

Följande väg börjar vid en applikations stub resolver, följer cache och delegeringar till den auktoritativa servern och leder därefter svaret tillbaka. Detta flöde gör det möjligt att sätta TTL, Glue, TCP-fallback, DNSSEC, zondrift och typiska felbilder i sitt sammanhang.

För meddelandeadministratörer är DNS ett överordnat styrsystem. En MTA fastställer därigenom nästa e-postmål, avsändarautentisering läser policyer och nycklar, certifikatförfaranden kan inkludera DNSSEC-skyddade data, och katalog- eller Kerberos-klienter söker tjänster via SRV-poster. DNS kontrollerar dock inte om den hittade tjänsten är frisk. Ett syntaktiskt korrekt MX-svar kan peka på en ouppnåelig SMTP-lyssnare; en lyckad A-uppslagning säger inget om TLS, autentisering eller applikationstillstånd.

## Arkitektur och roller

Ansvarsområdet för namn är organiserat som ett träd. Vid den namnlösa roten `.` börjar toppdomäner som `ch.`, följda av delegerade domäner och ytterligare etiketter. En avslutande punkt gör ett namn fullständigt och förhindrar lokala söksuffix. Denna lilla skrivskillnad är driftmässigt relevant: `mail.example.ch` kan kompletteras med en Search Domain på en klient, `mail.example.ch.` kan det inte.

En **domän** är en del av namnrymden. En **zon** är däremot den administrativt sammanhängande datamängd för vilken en auktoritativ server har lokalt ansvar. En delegering bryter ut en child-zon ur parent-zonen. Parent-zonen publicerar då ett NS-RRset och, om ett namnservernamn ligger inom den delegerade child-zonen, de A- eller AAAA-data som behövs för att nå den som **Glue**. Den grundläggande uppdelningen mellan namnrymd, zoner och delegering kommer från [RFC 1034](https://datatracker.ietf.org/doc/html/rfc1034#section-4.2); aktuella begrepp sammanfattas i [RFC 9499, avsnitt 7](https://datatracker.ietf.org/doc/html/rfc9499#section-7).

Fyra logiska roller förklarar upplösningsvägen:

| Roll | Kunskap och uppgift | Viktig driftsgräns |
|---|---|---|
| Stub resolver | tar emot applikationens fråga och skickar den vidare till en konfigurerad resolver | Söksuffix, lokal hosts-fil och klientcache kan påverka resultatet före själva DNS |
| Rekursiv resolver | levererar ett slutsvar från cachen eller genom iterativa frågor | är en förtroendegräns för cache, filtrering, loggning och DNSSEC-validering |
| Vidarebefordrare | hanterar rekursiva frågor från en annan resolver | flyttar upplösning och observerbarhet till en ytterligare operatör |
| Auktoritativ server | svarar från lokalt laddade zoner och sätter AA-biten i auktoritativa svar | känner inte till tjänstens hälsa och ska inte erbjuda öppen rekursion för främmande namn |

En produkt kan implementera flera roller, men de bör ändå betraktas separat i drift. Ett fel i den auktoritativa tjänsten påverkar publiceringen av egna zoner; ett fel i den rekursiva tjänsten påverkar namnupplösningen för egna klienter. Gemensamma processer, adresser eller felområden försvårar denna åtskillnad.

## Namnupplösning steg för steg

En applikation frågar normalt inte själv root- och auktoritativa servrar. Dess stub resolver överlämnar en rekursiv fråga till en resolver. Saknas en användbar cachepost följer resolvern delegeringarna från root via toppdomänen till den ansvariga zonen. Varje referral anger nästa NS-RRset och vid behov Glue-adresser. Resolvern sammanställer slutsvaret, validerar vid behov DNSSEC och cachar resultatet ([RFC 1034, avsnitt 4.3](https://datatracker.ietf.org/doc/html/rfc1034#section-4.3)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1016" src="/images/kb-interaktiv-dns-aufloesung.svg?v=20260813" title="Interaktive Infografik: DNS-Auflösung von Stub Resolver über Cache, Root und Delegationen bis zur autoritativen Antwort" loading="lazy">
  <a href="/images/kb-interaktiv-dns-aufloesung.svg?v=20260813">Öppna infografik om DNS-upplösning</a>
</iframe>

Den ritade linjära vägen är en kallstartsmodell. I en varm resolvercache finns root- och TLD-delegeringar oftast redan, så att endast en del av stegen behövs. Även QNAME-minimering, vidarebefordran, lokala zoner eller aggressiva DNSSEC-cachar kan ändra den synliga paketvägen. Avgörande är fortfarande att tilldela varje observation en roll: Ett icke-auktoritativt cachesvar är inget bevis på vad den ansvariga auktoritativa servern levererar just nu.

## Teknisk uppbyggnad av ett DNS-meddelande

På nätet består ett DNS-meddelande av Header, Question, Answer, Authority och Additional Section. Frågan anger QNAME, QTYPE och QCLASS. Svar och hänvisningar visas som Resource Records i de återstående avsnitten. Flaggor visar bland annat auktoritet, rekursionsönskemål och trunkering; Response Code beskriver resultatet. Vid analys är därför inte bara Answer-texten viktig, utan även från vilken server och med vilka flaggor den kom ([RFC 1035, avsnitt 4.1](https://datatracker.ietf.org/doc/html/rfc1035#section-4.1)).

Särskilt relevanta för diagnostik är:

| Signal | Betydelse | Typisk administratörsfråga |
|---|---|---|
| `AA` | Svaret är auktoritativt för det besvarade namnet | Frågades den ansvariga zonen direkt eller bara en cache? |
| `TC` | Svaret trunkerades för den använda transporten | Fungerar återförsök över TCP och tillåter brandväggen TCP/53? |
| `RD` / `RA` | Rekursion begärd / erbjuds av servern | Användes en auktoritativ server av misstag som resolver? |
| `AD` | Den svarande validatorn anser att data är autentiserade | Är resolvern pålitlig och är transporten till den skyddad? |
| `CD` | Klienten kräver att valideringsfel inte förkastas av resolvern | Valideras DNSSEC just nu eller undersöks bara rådata? |
| `RCODE` | Resultatstatus såsom NOERROR, NXDOMAIN, SERVFAIL eller REFUSED | Är namnet fel, typen obefintlig, upplösningen störd eller frågan nekad av policy? |

EDNS(0) lägger till ytterligare flaggor, alternativ och en större annonserad UDP-nyttolast genom en pseudoartad **OPT-post**, utan att ersätta grundformatet ([RFC 6891](https://datatracker.ietf.org/doc/html/rfc6891)). DNSSEC:s DO-bit finns i detta utökade flaggfält, inte i det ursprungliga DNS-huvudet.

## Resource Records och RRset

Det fackliga innehållet finns i Resource Records. Owner Name, typ och klass avgör vad data tillhör; TTL och RDATA tillhandahåller cachetid och typspecifikt värde. Alla poster med samma Owner, typ och klass bildar ett RRset och delar en TTL. Flera MX-, A- eller AAAA-värden är därför en gemensamt cachad mängd, inte enskilda objekt som kan styras oberoende av varandra ([RFC 2181, avsnitt 5](https://datatracker.ietf.org/doc/html/rfc2181#section-5)).

| Typ | Funktion | Viktig gräns |
|---|---|---|
| `SOA` | Zonmetadata, Serial, Refresh/Retry/Expire och negativ cacheparameter | ett SOA-RR per zon vid apex; ändring av Serial styr synkroniseringen med secondaries |
| `NS` | auktoritativa servrar för en zon eller delegering | målet för ett NS får inte vara ett alias |
| `A` / [`AAAA`](https://datatracker.ietf.org/doc/html/rfc3596#section-2.1) | IPv4- eller IPv6-adress för ett namn | säger inget om tjänsteport eller nåbarhet |
| `CNAME` | Alias från ett namn till ett kanoniskt namn | får i princip inte samexistera med annan data vid samma Owner |
| `MX` | Mail Exchanger med Preference | målet måste kunna lösas till A/AAAA och får inte vara en CNAME |
| `PTR` | omvänd uppslagning, oftast under `in-addr.arpa.` eller `ip6.arpa.` | reverse-zonen tillhör vanligen adressinnehavaren, inte operatören av forward-zonen |
| `TXT` | en eller flera teckensträngar utan protokollöverskridande semantik | tolkningen uppstår först genom SPF, DKIM, DMARC eller ett annat förfarande |
| [`SRV`](https://datatracker.ietf.org/doc/html/rfc2782) | tjänst, transport, prioritet, vikt, port och mål | klienten måste implementera SRV-semantiken för respektive tjänst |
| [`CAA`](https://datatracker.ietf.org/doc/html/rfc8659) | policy för certifikatutfärdare för domännamn | är varken transportkryptering eller ett servercertifikat |
| `DS`, `DNSKEY`, `RRSIG`, `NSEC` | DNSSEC-förtroendekedja, nycklar, signaturer och autentiserad icke-existens | skyddar DNS-datans integritet, inte dess konfidentialitet |

Texten i en zonfil är bara **Presentation Form**. På nätet överförs namn etikett för etikett, tal binärt och vissa namn valfritt komprimerade. Den som kopierar en teckensträng från ett användargränssnitt bör därför skilja mellan om gränssnittet redan har satt samman citationstecken, escape-sekvenser eller flera TXT-strängar till en logisk nyttolast.

## Delegering, Glue och auktoritet

Med en delegering överlämnar en zon ansvaret för ett underträd. Parent-zonen publicerar child-zonens NS-RRset; child-zonen anger själv sina auktoritativa servrar igen. Om båda sidor avviker från varandra kan resolvrar ta olika vägar. Glue-adresser i parent-zonen löser endast hönan-och-ägget-problemet med nåbarhet. De auktoritativa A-/AAAA-värdena vid namnservernamnet förblir självständiga data med egen TTL och förvaltning.

Särskilt kritisk är **in-bailiwick Glue**: Om `example.ch.` delegeras till `ns1.example.ch.`, behöver resolvern adressen till `ns1.example.ch.` innan den kan fråga child-zonen. Utan Glue skulle en upplösningscirkel uppstå. Om delegeringen däremot pekar på `ns1.provider.net.`, kan dess adress fastställas via en annan delegeringskedja.

Vid byte av namnserver måste därför minst fyra tillstånd kontrolleras: ny zon laddad på alla servrar, child-NS-RRset anpassat, parent-delegering anpassad och nödvändig Glue uppdaterad. Först därefter bör gamla servrar tas ur drift eller göras onåbara.

## Cache, TTL och negativa svar

Efter upplösningen lever ett svar vidare i cachar. Dess TTL är den maximala användningstiden från den tidpunkt då respektive resolver lärde sig det. Det finns därför ingen global gemensam nedräkning. Stub resolvers, vidarebefordrare, rekursiva resolvrar och applikationer kan kasta samma gamla RRset vid olika tidpunkter. En TTL som sänks före en migrering hjälper endast för svar som laddas på nytt därefter; redan cachade data kan inte återkallas.

Även icke-existens cachas. **NXDOMAIN** betyder att det efterfrågade namnet inte finns; **NODATA** är ett NOERROR-svar där namnet finns men inget RRset av den efterfrågade typen. Den auktoritativa servern lägger vid båda svaren sitt SOA-RR i Authority Section. Den negativa cachetiden är det lägsta värdet av SOA-TTL och SOA.MINIMUM ([RFC 2308, avsnitt 3 till 5](https://datatracker.ietf.org/doc/html/rfc2308#section-3)). Det förklarar varför en ny DKIM-selektor eller ett nytt värdnamn efter ett tidigare misslyckat försök först kan fortsätta att visas som obefintligt.

`SERVFAIL` ska skiljas från detta: Resolvern kunde inte skapa ett användbart svar. Orsaker omfattar bland annat timeout till auktoritativa servrar, en trasig delegering, DNSSEC-valideringsfel eller en intern resursbegränsning. `REFUSED` betyder däremot att den efterfrågade servern inte utför åtgärden på grund av sin policy. Ett diagnostikverktyg måste därför visa RCODE, AA-bit, svarande server och Sections; en utdata som endast rapporterar «ingen adress» suddar ut avgörande skillnader.

## Transport: UDP, TCP och krypterade resolvervägar

För detta utbyte måste nätvägen tillåta UDP och TCP på port 53. TCP är inte begränsat till zonöverföringar: En resolver får använda det direkt och måste kunna växla över till det efter ett trunkerat UDP-svar. Blockerad TCP märks därför ofta först vid stora, DNSSEC-rika eller svarsrika RRset. Små A-frågor förblir gröna och ger en felaktig bild ([RFC 7766](https://datatracker.ietf.org/doc/html/rfc7766)).

Utan EDNS är DNS-nyttolasten över UDP begränsad till 512 byte. EDNS gör det möjligt för frågaren att annonsera en större mottagbar nyttolast. Ett för stort värde kan dock tvinga fram IP-fragmentering; om en väg tappar eller blockerar fragment uppstår den typiska bilden att små svar fungerar medan stora svar timeoutar. EDNS-specifikationen rekommenderar att beakta den faktiska mottagningsförmågan och vägen och att vid problem växla till mindre värden eller TCP ([RFC 6891, avsnitt 6.2](https://datatracker.ietf.org/doc/html/rfc6891#section-6.2)).

[DNS over TLS](https://datatracker.ietf.org/doc/html/rfc7858), [DNS over HTTPS](https://datatracker.ietf.org/doc/html/rfc8484) eller [DNS over QUIC](https://datatracker.ietf.org/doc/html/rfc9250) krypterar en DNS-transportväg. Dessa förfaranden ändrar varken zoninnehåll eller delegering och ersätter inte DNSSEC: transportkryptering skyddar anslutningen till en resolver, DNSSEC autentiserar data längs delegeringskedjan. Klassisk DNS på port 53 är okrypterad; den gemensamma terminologin för transporttyper definieras i [RFC 9499, avsnitt 6](https://datatracker.ietf.org/doc/html/rfc9499#section-6).

## DNSSEC och förtroendekedjan

DNSSEC kompletterar namnupplösningen med verifierbart ursprung och integritet. En zon signerar RRset med RRSIG och publicerar de offentliga nycklarna som DNSKEY. Parent-zonen kopplar child-zonen till den överordnade förtroendekedjan via DS; NSEC eller NSEC3 kan även bevisa icke-existens. En validerande resolver börjar vid sin Trust Anchor och kontrollerar denna kedja fram till svaret. Innehållet förblir offentligt: DNSSEC krypterar ingen fråga ([RFC 4033](https://datatracker.ietf.org/doc/html/rfc4033), [RFC 4034](https://datatracker.ietf.org/doc/html/rfc4034)).

För drift är inte bara nyckelfiler relevanta, utan flera tidsmässigt kopplade tillstånd:

- RRSIG har start och utgång; felaktig systemtid eller utebliven omsignering kan göra en hel zon **bogus**.
- En DS i parent-zonen måste matcha en användbar DNSKEY i child-zonen. En föräldralös DS är värre för validerande resolvrar än en medvetet osignerad delegering.
- Vid rollovers måste publicering, signering, parent-ändring, TTL och cachetider planeras som en tillståndsmaskin.
- En validerande resolver levererar ofta SERVFAIL för bogus-data. Ett test utan validering kan samtidigt visa ett till synes normalt svar.

AD-flaggan ensam är bara lika pålitlig som resolvern och vägen till den. För en oberoende kontroll måste en administratör undersöka kedjan med ett validerande verktyg och lokalisera den felaktiga övergången mellan DS, DNSKEY och RRSIG.

## Auktoritativ drift och dataflöde

Bakom det auktoritativa svaret finns en egen distributionsväg. I en klassisk modell hanterar en primär källa zonen, DNS NOTIFY informerar secondaries om en ny SOA-Serial och AXFR eller IXFR överför fullständiga respektive inkrementella data. En frisk lyssnare bevisar därför ännu inte att servern har laddat den nya zonen. Serial, överföringsstatus och svar från varje auktoritativ nod måste beaktas tillsammans ([RFC 1996](https://datatracker.ietf.org/doc/html/rfc1996), [RFC 5936](https://datatracker.ietf.org/doc/html/rfc5936), [RFC 1995](https://datatracker.ietf.org/doc/html/rfc1995)).

Zonen kan skapas från textfiler, en databas, ett API, Active Directory eller en Git-/CI-pipeline. Detta implementeringsval ändrar inte DNS-wire-protokollet, men avgör transaktionsgränser, granskningsbarhet och återställning. RFC 2136 definierar atomiska dynamiska uppdateringar med Prerequisites; ett leverantörs-API-anrop är däremot ett separat styrprotokoll och måste dokumentera sina egna konsekvens- och felregler ([RFC 2136](https://datatracker.ietf.org/doc/html/rfc2136)).

En DNS-säkerhetskopia är användbar först när den faktiskt levererade zonen kan återställas från den. Beroende på plattform omfattar detta:

- Zon eller källdatabas inklusive SOA-Serial och dynamisk journal;
- serverkonfiguration, Views, ACL:er, Forwarder och katalogtilldelning;
- TSIG-Secrets, DNSSEC-privatnycklar och tillstånd för automatiska rollovers;
- parent-data utanför den egna zonen, i synnerhet delegering, Glue och DS;
- en testad väg för att återförsörja secondaries och semantiskt kontrollera zondata.

Secondaries är tillgänglighetskopior, men inte automatiskt historiska säkerhetskopior. En felaktig eller skadlig ändring kan snabbt replikeras till alla auktoritativa servrar via NOTIFY och zonöverföring.

## Implementerings- och teknikstack

DNS betecknar ingen enskild daemon. Den gemensamma teknikstacken består av namn och RRset, binärt Query-/Response-format, UDP och TCP, cachelogik samt valfritt DNSSEC. Auktoritativa servrar, rekursiva resolvrar, vidarebefordrare och Managed DNS implementerar dessa byggstenar på olika sätt. För drift räknas därför först en produkts roll, därefter dess språk eller paketering:

| Implementering | Primär roll | Tekniskt fokus |
|---|---|---|
| [BIND 9](https://bind9.readthedocs.io/en/latest/) | auktoritativ och/eller rekursiv | universell namnserver, zonfiler, dynamiska uppdateringar, DNSSEC och diagnostikverktyg |
| [Unbound](https://unbound.docs.nlnetlabs.nl/en/latest/manpages/unbound.html) | rekursiv caching-resolver | modulär resolver-, validator- och cache-pipeline; ingen primär auktoritativ roll |
| [Knot DNS](https://www.knot-dns.cz/) | auktoritativ | endast auktoritativ, parallell bearbetning, zonöverföring, DDNS och DNSSEC |
| [Windows Server DNS](https://learn.microsoft.com/windows-server/networking/dns/dns-overview) | auktoritativ och rekursiv | valfria AD-integrerade zoner, säkra dynamiska uppdateringar, policyer, cache och vidarebefordran |

Uppdelningen av roller är viktigare än produktnamnet. För en offentlig auktoritativ plattform är zondistribution, secondaries, DNSSEC-signering och DDoS-resiliens avgörande; för en företagsresolver är cache, vidarebefordran, interna namnrymder, policy, dataskydd och validering avgörande.

## DNS i e-post- och identitetsdrift

Vid meddelandehantering blir ett DNS-svar omedelbart en leveransväg. En sändande MTA frågar efter MX-RRset för mottagardomänen, föredrar det minsta värdet och behandlar samma preferenser som likvärdiga. Endast om ingen MX finns gäller själva domänen som implicit mål. När MX-poster finns är ett A-/AAAA-värde vid domänens apex ingen ersättning. Varje MX-mål behöver egna adresser och får inte vara ett alias; en domän utan e-postmottagning publicerar Null MX `0 .` ([RFC 5321, avsnitt 5](https://datatracker.ietf.org/doc/html/rfc5321#section-5), [RFC 2181, avsnitt 10.3](https://datatracker.ietf.org/doc/html/rfc2181#section-10.3), [RFC 7505](https://datatracker.ietf.org/doc/html/rfc7505)).

PTR-poster löses via det omvända adressträdet. Forward- och reverse-zon har ofta olika ägare; ändringar måste därför samordnas mellan domän- och IP-adressoperatören. En PTR är ett namn, inte ett kryptografiskt identitetsbevis. Mottagande e-postplattformar kan använda konsekvent forward-/reverse-upplösning som signal, men deras specifika reputations- eller acceptanspolicy är ingen DNS-egenskap.

SPF, DKIM och DMARC använder DNS som publiceringskanal, men definierar egna utvärderingsregler. SPF läser en enda logisk TXT-nyttolast och begränsar DNS-orsakande termer; DKIM adresserar nycklar via selektorer; DMARC ligger under `_dmarc`. Dessa förfaranden behandlas fackligt i [SPF, DKIM och DMARC](/kb/mail-auth). DNS-administratören måste för dem framför allt behärska Owner Name, uppdelning av TXT-strängar, svarsstorlek, TTL, delegering och negativa cachetider korrekt.

Även [LDAP](/kb/ldap) och [Kerberos](/kb/kerberos) använder ofta SRV-poster för tjänsteupptäckt. En SRV-post innehåller, utöver mål och port, prioritet och vikt. Dessa värden är ingen universell lastbalanserarkonfiguration; endast klienter som implementerar respektive SRV-förfarande tolkar dem.

## Diagnostik

Diagnostik börjar därför aldrig med «DNS fungerar inte», utan med namn, typ, klass, frågad server, transport och tidpunkt. Företagsresolvrar, offentliga resolvrar och auktoritativa servrar kan tillfälligt eller genom Split DNS och policy leverera olika svar. Denna avvikelse är inte mätbrus, utan den viktigaste indikationen på var upplösningsvägen delar sig.

### Fråga RRset målmedvetet

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

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) kan tvinga fram en viss server, posttyp, enbart DNS och TCP. [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) visar dessutom flaggor, Sections, RCODE, svarande server och frågetid. Utan explicit `-Server` respektive `@server` testas den konfigurerade resolvern, inte nödvändigtvis den auktoritativa källan.

### Kontrollera TCP och DNSSEC separat

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

[`delv`](https://bind9.readthedocs.io/en/latest/manpages.html#delv-dns-lookup-and-validation-utility) använder BIND:s resolver- och validatorlogik för att kontrollera en DNSSEC-kedja. En lyckad fråga med `+dnssec` bevisar däremot endast att DNSSEC-data begärdes och levererades; den validerar inte kedjan automatiskt. Det separata TCP-testet avslöjar brandväggar som tillåter UDP/53 men blockerar TCP/53.

### Töm lokala cachar målmedvetet

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

[`Clear-DnsClientCache`](https://learn.microsoft.com/powershell/module/dnsclient/clear-dnsclientcache) tömmer Windows DNS-klientens cache. [`resolvectl`](https://man7.org/linux/man-pages/man1/resolvectl.1.html) styr cachen för `systemd-resolved`; på system med `nscd`, `dnsmasq`, en lokal Unbound eller en applikationscache ansvarar en annan cache. Att tömma klientens cache förändrar aldrig cachen hos en upstream-resolver.

### Tilldela felbilden till överlämningspunkten

| Observation | Trolig nivå | Nästa kontroll |
|---|---|---|
| en resolver levererar gammalt värde, auktoritativa servrar det nya | positiv cache | återstående TTL, vidarebefordrarkedja, applikationscache |
| NXDOMAIN kvarstår efter att posten skapats | negativ cache eller fel zon | SOA i negativt svar, parent-delegering, Owner Name |
| NOERROR utan Answer | namnet finns, efterfrågad typ saknas | CNAME-kedja, exakt QTYPE, NODATA-SOA |
| endast stora eller signerade svar timeoutar | EDNS, fragmentering eller TCP-fallback | `TC`, mindre EDNS-storlek, explicit TCP-test, brandvägg |
| validerande resolver levererar SERVFAIL, icke-validerande ett svar | DNSSEC bogus | DS/DNSKEY-matchning, RRSIG-tider, algoritm, systemtid |
| auktoritativa servrar levererar olika Serial | replikering | NOTIFY, IXFR/AXFR, transfer-ACL, journal, primärkälla |
| offentliga och interna klienter ser olika mål | Split DNS eller resolverpolicy | frågad resolver, View-tilldelning, klientsubnät, vidarebefordrare |
| MX finns, leverans misslyckas före SMTP | härlett målnamn eller transport | MX-Preference, A/AAAA för MX-målet, TCP/25, CNAME-förbud |

## Övervakning och driftkriterier

Övervakningen måste avbilda samma väg. En enda A-uppslagning mot den lokala standardresolvern upptäcker varken en trasig delegering, blockerad TCP, utgångna signaturer eller en secondary med gammal Serial. För meddelandehantering och identitet bör därför minst följande signaler samlas in separat:

- auktoritativa svar från varje publicerad NS över UDP och TCP, inklusive AA-bit och SOA-Serial;
- parent-delegering, child-NS-RRset, Glue och för signerade zoner DS/DNSKEY-övergången;
- rekursiv latens, cache-hit-kvot, timeout-, SERVFAIL-, REFUSED- och NXDOMAIN-frekvenser;
- svarsstorlek, trunkering, EDNS-fel och TCP-fallback;
- utgångstidpunkter för RRSIG, Key-Rollover-tillstånd och signeringsköer;
- NOTIFY-, AXFR- och IXFR-framgång samt zonens ålder på varje secondary;
- fackliga RRset som MX, tillhörande A/AAAA-adresser, PTR och de TXT-namn som krävs för [e-postautentisering](/kb/mail-auth).

Ett syntetiskt test bör kontrollera både den normala klientvägen och den auktoritativa källan. Endast så går det att skilja på om en störning finns i det publicerade datainnehållet, i delegeringen, i en resolvercache eller i applikationen.

## Säkerhet och felområden

De båda serverrollerna kräver olika skyddsåtgärder. Rekursion ska endast vara tillgänglig för betrodda klienter; en öppen resolver kan missbrukas för reflektions- och förstärkningsattacker. Auktoritativa servrar måste däremot förbli globalt nåbara, men får inte rekursivt lösa godtyckliga främmande namn. Att blanda roller utökar attackytan och gör belastningsorsaker svårare att identifiera ([RFC 5358](https://datatracker.ietf.org/doc/html/rfc5358)).

DNSSEC skyddar publicerade RRset mot omärkt förändring, men inte serverprocessen och inte tillgängligheten. **TSIG** autentiserar enskilda DNS-meddelanden med en delad nyckel och används bland annat för uppdateringar och zonöverföringar; det är ingen offentlig signatur av zondata ([RFC 8945](https://datatracker.ietf.org/doc/html/rfc8945)). Transfer-ACL:er, TSIG-Secrets och DNSSEC-privatnycklar är separata skyddsobjekt.

Split DNS och policysvar kan vara nödvändiga, men skapar flera sanningar för samma QNAME/QTYPE. Minst tilldelningskriterium, källzon, vidarebefordringsväg, DNSSEC-beteende och övervakning per View måste dokumenteras. Annars tolkas en avsiktlig avvikelse vid nästa störning som cachefel eller replikeringsproblem.

## Teknisk historia

Före DNS distribuerade Internet centralt underhållna värdtabeller. Med fler nät, värdar och oberoende operatörer blev detta förfarande en flaskhals. Paul Mockapetris beskrev 1983 en hierarkiskt delegerbar namntjänst i RFC 882 och RFC 883. RFC 1034 och RFC 1035 ersatte denna version 1987 och utgör fortfarande kärnan i systemet som STD 13.

Den första fungerande serverimplementeringen **Jeeves** kördes 1983/84 på DEC-TOPS-20-system. Kort därefter skapades Berkeley Internet Name Domain Package **BIND** för Unix vid University of California, Berkeley, med DARPA-finansiering. BIND 8 kom 1997, BIND 9 i september 2000 som en omfattande omarbetning. ISC dokumenterar denna utvecklingshistoria i [A Brief History of the DNS and BIND](https://bind9.readthedocs.io/en/latest/history.html).

Protokollet växte stegvis utan att ersätta den hierarkiska kärnan: NOTIFY och IXFR påskyndade zonsynkroniseringen under 1990-talet, EDNS utökade meddelandemodellen 1999 och konsoliderades senare i RFC 6891, DNSSEC fick 2005 de idag grundläggande DNSKEY-/DS-/RRSIG-mekanismerna, och krypterade resolvertransporter tillkom med DoT, DoH och DoQ. DNS är därför inget fryst protokoll från 1987, utan ett utbyggbart system vars bakåtkompatibilitet och långa cachetillstånd präglar varje driftändring.

## Källor

- [RFC Editor – STD 13: Domain Name System](https://www.rfc-editor.org/info/std13)
- [RFC 9499 – DNS Terminology](https://datatracker.ietf.org/doc/html/rfc9499) – aktuella begrepp för roller, zoner, cache, DNSSEC och transport.
- [RFC 1034 – Domain Names: Concepts and Facilities](https://datatracker.ietf.org/doc/html/rfc1034) – namnrymd, zoner, delegering, resolvrar och serverroller.
- [RFC 1035 – Domain Names: Implementation and Specification](https://datatracker.ietf.org/doc/html/rfc1035) – wire-format, Resource Records, meddelandeavsnitt och Master Files.
- [RFC 6891 – Extension Mechanisms for DNS (EDNS(0))](https://datatracker.ietf.org/doc/html/rfc6891) – OPT, utökade flaggor och UDP-nyttolast.
- [RFC 2181, avsnitt 5](https://datatracker.ietf.org/doc/html/rfc2181)
- [`AAAA`](https://datatracker.ietf.org/doc/html/rfc3596)
- [RFC 2782 – A DNS RR for specifying the location of services](https://datatracker.ietf.org/doc/html/rfc2782) – uppbyggnad och val av SRV-poster.
- [RFC 8659 – DNS Certification Authority Authorization](https://datatracker.ietf.org/doc/html/rfc8659) – CAA-poster och utvärdering av certifikatutfärdare.
- [RFC 2308 – Negative Caching of DNS Queries](https://datatracker.ietf.org/doc/html/rfc2308) – NXDOMAIN, NODATA, SOA och negativ cachetid.
- [RFC 7766 – DNS Transport over TCP](https://datatracker.ietf.org/doc/html/rfc7766) – obligatoriskt TCP-stöd och anslutningsbeteende.
- [RFC 7858 – DNS over TLS](https://datatracker.ietf.org/doc/html/rfc7858) – DNS över TLS.
- [RFC 8484 – DNS Queries over HTTPS](https://datatracker.ietf.org/doc/html/rfc8484) – DNS över HTTPS.
- [RFC 9250 – DNS over Dedicated QUIC Connections](https://datatracker.ietf.org/doc/html/rfc9250) – DNS över QUIC.
- [RFC 4033 – DNS Security Introduction and Requirements](https://datatracker.ietf.org/doc/html/rfc4033) – DNSSEC-skyddsmål, validering och begränsningar.
- [RFC 4034 – Resource Records for DNSSEC](https://datatracker.ietf.org/doc/html/rfc4034) – DNSKEY, DS, RRSIG och NSEC.
- [RFC 1996 – DNS NOTIFY](https://datatracker.ietf.org/doc/html/rfc1996) – avisering av secondary-servrar.
- [RFC 5936 – DNS Zone Transfer Protocol (AXFR)](https://datatracker.ietf.org/doc/html/rfc5936) – fullständiga zonöverföringar över TCP.
- [RFC 1995 – Incremental Zone Transfer (IXFR)](https://datatracker.ietf.org/doc/html/rfc1995) – inkrementell zonsynkronisering.
- [RFC 2136 – Dynamic Updates in DNS](https://datatracker.ietf.org/doc/html/rfc2136) – atomiska ändringar med Prerequisites.
- [BIND 9](https://bind9.readthedocs.io/en/latest/)
- [Unbound Documentation – unbound(8)](https://unbound.docs.nlnetlabs.nl/en/latest/manpages/unbound.html) – rekursiv cache och DNSSEC-validering.
- [Knot DNS](https://www.knot-dns.cz/) – endast auktoritativ implementering och driftsfunktioner.
- [Microsoft Learn – DNS in Windows Server](https://learn.microsoft.com/windows-server/networking/dns/dns-overview) – Windows-DNS-roller, AD-integration, cache och vidarebefordran.
- [RFC 5321, avsnitt 5](https://datatracker.ietf.org/doc/html/rfc5321)
- [RFC 7505 – Null MX](https://datatracker.ietf.org/doc/html/rfc7505) – uttrycklig markering av domäner utan e-postmottagning.
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – DNS-frågor i Windows.
- [ISC BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html) – frågealternativ, serverval och tolkning av utdata.
- [`Clear-DnsClientCache`](https://learn.microsoft.com/powershell/module/dnsclient/clear-dnsclientcache)
- [`resolvectl`](https://man7.org/linux/man-pages/man1/resolvectl.1.html)
- [RFC 5358 – Preventing Use of Recursive Nameservers in Reflector Attacks](https://datatracker.ietf.org/doc/html/rfc5358) – begränsning av rekursion och skydd mot missbruk.
- [RFC 8945 – Secret Key Transaction Authentication for DNS (TSIG)](https://datatracker.ietf.org/doc/html/rfc8945) – meddelandeautentisering för uppdateringar och överföringar.
- [ISC – A Brief History of the DNS and BIND](https://bind9.readthedocs.io/en/latest/history.html) – Jeeves, Berkeley, BIND 8 och BIND 9.
