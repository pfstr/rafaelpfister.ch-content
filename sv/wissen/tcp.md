---
title: "TCP: tillstånd, dataflöde och diagnostik"
blatt: "tcp"
description: "TCP som transportgrund för meddelande- och katalogtjänster: socketmodell, anslutningstillstånd, sekvensnummer, bekräftelser, flödes- och överbelastningskontroll, timers, typiska felbilder och diagnostikverktyg för administratörer."
fakten:
  - label: Fullständigt namn
    wert: Transmission Control Protocol
    href: https://datatracker.ietf.org/doc/html/rfc9293
  - label: Standard
    wert: RFC 9293 · STD 7
    href: https://www.rfc-editor.org/info/rfc9293
  - label: Tjänstemodell
    wert: tillförlitlig, ordnad, dubbelriktad byteström
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-2
  - label: Slutpunkt
    wert: IP-adress och 16-bitars portnummer
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.1
  - label: Anslutning
    wert: unikt par av lokal och fjärrsocket
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.1
  - label: Header
    wert: minst 20 byte plus alternativ
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.1
  - label: Tillförlitlighet
    wert: sekvensnummer, kumulativa ACK:er och retransmission
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.3
  - label: Flödeskontroll
    wert: mottagningsfönster rwnd
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.8.4
  - label: Överbelastningskontroll
    wert: cwnd · Slow Start · Congestion Avoidance
    href: https://datatracker.ietf.org/doc/html/rfc5681
  - label: Portintervall
    wert: 0–1023 · 1024–49151 · 49152–65535
    href: https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml
  - label: Kryptering
    wert: ingen; skydd genom TLS, IPsec eller TCP-AO
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-7
  - label: Diagnostik
    wert: porttest · sockettillstånd · paketinsamling
    href: https://www.tcpdump.org/manpages/tcpdump.1.html
werbung:
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: 6c749be6d80debccf17ceb4386ba0e298c11deedf855db326e7d20da9593fc05
translationModel: gpt-5.6-terra
translatedAt: 2026-10-14T09:38:08.930Z
translationReview: automatic
---

# TCP: tillstånd, dataflöde och diagnostik

TCP omvandlar IP:s opålitliga pakettransport till en ordnad, dubbelriktad dataström för applikationer. Applikationen skriver byte till en socket; TCP segmenterar dem, numrerar deras position, bekräftar mottagna områden och överför luckor på nytt. SMTP, IMAP, POP3, LDAP och HTTPS bygger på detta löfte. När en sådan applikation «hänger» måste administratören därför först skilja mellan om TCP fortfarande väntar på en bekräftelse eller om applikationsprotokollet redan blockerar.

Artikeln följer en anslutnings livscykel: socket och listener, trevägshandskakning, dataöverföring, flödes- och överbelastningskontroll, nedkoppling och TIME-WAIT. Från detta normala förlopp härleds sedan diagnostik, kapacitetsplanering, övervakning och härdning.

TCP känner dock varken meddelanden eller användare, certifikat eller e-postadresser. Det bevarar inte heller applikationsgränser: två efterföljande skrivoperationer kan sammanföras i ett segment eller fördelas över flera segment. En lyckad TCP-handskakning bevisar därför bara att två TCP-slutpunkter kunde etablera en anslutning. Om en SMTP-banner visas därefter, TLS förhandlas eller en LDAP-bindning lyckas avgörs av den ovanliggande applikationen.

För administratörer är TCP mindre relevant som skiktmodell än som ett **tillstånds- och signalsystem**. SYN, ACK, FIN och RST visar vid vilken överlämningspunkt en anslutning misslyckas. Sockettillstånd, retransmissioner, mottagningsfönster och timers förklarar varför en tjänst kan vara nåbar men långsam eller till synes frusen.

Förklaringen följer en TCP-anslutning från det första SYN via den ordnade byteströmmen till avslut eller reset. Fönster, retransmissioner, överbelastning och diagnostik förklaras i respektive fas.

## Socket- och anslutningsmodell

Innan paket flödar binder en applikation en socket till adress och port. En server lyssnar exempelvis på TCP 25; varje accepterad anslutning identifieras därefter entydigt av källadress, källport, destinationsadress och destinationsport. Därför kan tusentals klienter använda samma listener, så länge minst deras källadress eller efemära port skiljer sig åt. För diagnostik räcker alltså inte «port 25»: det fullständiga anslutningsparet behövs alltid ([RFC 9293, avsnitt 3.1](https://datatracker.ietf.org/doc/html/rfc9293#section-3.1)).

På en server binder en process en socket till en lokal adress och försätter den i tillståndet **LISTEN**. Valet av bindningsadress är driftsmässigt viktigt:

| Bindning | Synlighet | Typisk följd |
|---|---|---|
| `127.0.0.1:25` eller `::1:25` | endast lokal värd | Övervakning på servern är oanmärklig, fjärrsystem får ingen anslutning |
| `0.0.0.0:25` | alla lokala IPv4-adresser | Listenern följer ytterligare IPv4-adresser, om brandvägg och routning tillåter det |
| `[::]:25` | IPv6-wildcard | Dual-stack-beteendet beror på operativsystemet och alternativet `IPV6_V6ONLY` |
| konkret tjänste-IP | endast denna adress | kontrollerad separation, men känslig för IP- eller klusterändringar |

Serverporten anges vanligen genom en [IANA-registrering](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml); klienten väljer normalt en efemär port från intervallet som hanteras av operativsystemet. Brandväggar, NAT-gatewayer och lastbalanserare måste därför inte bara tillåta den välkända destinationsporten, utan hela returvägen till den dynamiska källporten.

### TCP Control Block

För varje anslutning håller TCP-stacken ett **Transmission Control Block** (TCB). Där finns bland annat tillstånd, lokala och fjärranslutna sockets, sänd- och mottagningssekvens, fönster, timers och säkerhetsparametrar. TCB är skälet till att en anslutning är tillståndsfull och till att lastbalanserare eller brandväggar måste arbeta med symmetrisk returväg: om ett svarssegment når en annan nod utan passande tillstånd kan den förkasta det eller svara med RST.

## Anslutningsetablering, dataöverföring och nedkoppling

Med aktiv öppning börjar det synliga anslutningstillståndet. Klienten skickar SYN med initialt sekvensnummer, listenern svarar med SYN+ACK och eget sekvensnummer, därefter bekräftar klienten med ACK. Redan dessa tre segment förhandlar alternativ som Maximum Segment Size, Window Scale, Selective Acknowledgment och Timestamps. Om ett paket saknas, svaret kommer via en annan väg eller en mellanprodukt ändrar alternativen, visar handskakningen redan var nåbarhet och senare prestanda skiljs åt.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 976" src="/images/kb-interaktiv-tcp-verbindung.svg?v=20260813" title="Interaktive Infografik: TCP-Verbindungslebenszyklus und Diagnosepunkte" loading="lazy">
  <a href="/images/kb-interaktiv-tcp-verbindung.svg?v=20260813">Öppna infografik om TCP-anslutningens livscykel</a>
</iframe>

Efter handskakningen befinner sig anslutningen i **ESTABLISHED**. Båda sidor kan skicka data oberoende av varandra och stänga sin sändriktning med FIN. En ordnad nedkoppling kräver därför vanligtvis ett FIN och ACK per riktning. Sidan som skickar det sista ACK:t förblir i **TIME-WAIT**, så att försenade segment från en gammal anslutning inte hamnar i en ny anslutning med samma socketpar ([RFC 9293, avsnitt 3.3.2 och 3.6](https://datatracker.ietf.org/doc/html/rfc9293#section-3.3.2)).

Ett **RST** är ingen ordnad avslutning. Det avbryter anslutningen och förkastar utestående tillstånd. Vid anslutningsetablering betyder ett giltigt RST vanligen att målvärden är nåbar, men att socketen inte accepteras. Efter redan överförda data kan ett RST komma från applikationen, operativsystemet eller en tillståndsfull mellannod. RST:ets position i paketinsamlingen är därför viktigare än det allmänna felmeddelandet «connection reset».

### Tillstånden som driftindikatorer

| Tillstånd | Teknisk betydelse | Vad en ansamling antyder |
|---|---|---|
| `LISTEN` | passiv socket väntar på SYN | förväntat servertillstånd; saknad listener tyder på tjänste- eller bindningsproblem |
| `SYN-SENT` | lokalt SYN skickat, matchande svar saknas | filter, rutt, fel adress eller överbelastad motpart |
| `SYN-RECEIVED` | SYN accepterat, sista ACK saknas | SYN-flood, asymmetrisk returväg, klient- eller brandväggsproblem |
| `ESTABLISHED` | sekvensutrymmen synkroniserade | normalt; många långlivade anslutningar kan vara avsiktliga, exempelvis med IMAP eller HTTP/2 |
| `FIN-WAIT-1/2` | lokal sida har stängt | motpart eller applikation avslutar sin riktning fördröjt |
| `CLOSE-WAIT` | främmande FIN mottaget, lokal applikation har ännu inte stängt | ofta en resursläcka eller blockerad applikationstråd |
| `LAST-ACK` | lokalt FIN skickat efter främmande avslut | motpartens ACK saknas eller returvägen är störd |
| `TIME-WAIT` | gammal fyrtuple skyddas före återanvändning | normalt för aktivt stängande sida; problematiskt först tillsammans med port- eller TCB-tryck |

Tillståndsnamnen definieras i [RFC 9293](https://datatracker.ietf.org/doc/html/rfc9293#section-3.3.2). Ett högt antal är inte ett fel i sig. Avgörande är ålder, tillväxthastighet, berörda motparter och relationen till anslutningsfrekvens, filbeskrivare, efemärt portintervall och applikationsgränser.

## Sekvensnummer, ACK:er och retransmission

Efter handskakningen räknar TCP dataströmmen byte för byte. Sekvensnumret anger den första byten i ett segment; ACK anger nästa förväntade byte och bekräftar därmed allt före den. Paket får försvinna, komma dubbelt eller byta ordning. Mottagaren kan buffra dem, men lämnar endast vidare en sammanhängande ström till applikationen. Just därför kan ett enda saknat område blockera en till synes lugn applikation, trots att senare paket redan har anlänt till värden.

Om ett segment eller dess ACK går förlorat kan avsändaren upptäcka det saknade området på två sätt:

1. **Retransmission Timeout (RTO):** En timer härleds från utjämnad Round-Trip Time och dess variation. När den löper ut skickas det äldsta obekräftade segmentet igen. [RFC 6298](https://datatracker.ietf.org/doc/html/rfc6298) definierar beräkning, minimivärden och exponentiell backoff.
2. **Fast Retransmit:** Flera dubbla ACK:er visar en lucka i dataströmmen. Avsändaren kan skicka det förmodat förlorade området igen utan att vänta på RTO ([RFC 5681](https://datatracker.ietf.org/doc/html/rfc5681#section-3.2)).

En retransmission bevisar inte automatiskt paketförlust i nätverket. En insamling kan ha gjorts på endast ena sidan, paket kan saknas på grund av capture-drops eller visa en spurious retransmission. Wireshark markerar uttryckligen sin TCP-analys som en tolkning som härleds från den observerade paketströmmen ([Wireshark User’s Guide – TCP Analysis](https://www.wireshark.org/docs/wsug_html_chunked/ChAdvTCPAnalysis.html)). För ett tillförlitligt uttalande bör därför insamlingar vid klient, server och en relevant mellanpunkt korreleras om möjligt.

## Flödeskontroll och överbelastningskontroll

Enbart tillförlitlighet förhindrar ännu inte att en snabb avsändare överbelastar mottagare eller nät. TCP begränsar därför antalet samtidigt obekräftade byte med två fönster, där det mindre i slutänden gäller:

- **Mottagningsfönstret** `rwnd` skyddar mottagaren. Det beskriver den för närvarande tillgängliga bufferten. Om mottagaren rapporterar Zero Window måste avsändaren, förutom periodiska Window Probes, vänta. Ett varaktigt Zero Window är ofta inte ett nätverksproblem, utan betyder att den mottagande applikationen inte läser sina socketdata tillräckligt snabbt.
- **Congestion Window** `cwnd` skyddar vägen. Det hanteras lokalt av avsändaren och anpassas utifrån bekräftelser och förlustsignaler. [RFC 5681](https://datatracker.ietf.org/doc/html/rfc5681) beskriver Slow Start, Congestion Avoidance, Fast Retransmit och Fast Recovery.

Ungefär det mindre av `rwnd` och `cwnd` är verksamt. Hög bandbredd ensam garanterar därför inte hög genomströmning. På en väg med stor bandbredds-latensprodukt måste tillräcklig mängd obekräftade data tillåtas. Alternativet Window Scale utökar 16-bitarsfönstret; Timestamps stöder bland annat RTT-mätning och skydd mot gamla sekvensnummer ([RFC 7323](https://datatracker.ietf.org/doc/html/rfc7323)). Båda alternativen förhandlas under handskakningen och kan inte aktiveras senare för samma anslutning.

För korta SMTP-transaktioner dominerar ofta antalet Round Trips: TCP-handskakning, valfri TLS-handskakning och flera protokollsvar. För stora bilagor och replikeringsströmmar blir dessutom RTT, förlust, fönster, offloading och algoritm för överbelastningskontroll relevanta. Ett genomströmningsproblem är därför först «bandbredd» när handskakningstid, applikationspauser, Zero Windows och retransmissioner har uteslutits.

## Timers, keepalives och idle timeouts

När inget omedelbart svar kommer avgör timers nästa steg. Retransmission Timeout utlöser dataåteröverföringar, Persist Timer håller en anslutning vid liv trots Zero Window, och TIME-WAIT förhindrar att gamla segment hamnar i en ny anslutning. Operativsystem och applikationer kompletterar med keepalive och User Timeout. Den som bara loggar «timeout» förlorar dessa skillnader och därmed den egentliga felorsaken.

TCP-keepalives är som standard ofta inaktiverade eller börjar först efter långa inaktivitetsperioder. De ersätter ingen hälsokontroll på applikationsnivå. En lyckad keepalive säger endast att TCP-peeren svarar; den bevisar inte att en SMTP-worker bearbetar meddelanden eller att en LDAP-fråga kan hanteras. Omvänt kopplar brandväggar och lastbalanserare ofta från inaktiva anslutningar tidigare än ändsystemen. Då finns på ena sidan en till synes giltig socket kvar tills nästa dataförsök utlöser en timeout eller RST.

För långlivade IMAP-, LDAP-pool- eller API-anslutningar måste därför fyra värden passa ihop: applikationens idle timeout, TCP-keepalive-intervall, varje mellannods idle timeout och klientens återanslutningslogik. Den minsta oanmälda timeouten bestämmer det observerade beteendet.

## Diagnostik från socket till paket

Felsökningen blir tillförlitlig så snart den beskriver samma anslutning som applikationen: källvärd, målnamn, upplöst adress, destinationsport, tidpunkt och förväntat protokoll. En ping ersätter inte dessa uppgifter. ICMP kan vara blockerat trots att TCP fungerar; omvänt bevisar ett ekosvar varken en listener eller tillståndet hos en Stateful Firewall.

### Nåbarhet och listener

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für den TCP-Verbindungstest">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection -ComputerName mail.example.ch -Port 25 -InformationLevel Detailed

Get-NetTCPConnection -State Listen -LocalPort 25 |
  Select-Object LocalAddress, LocalPort, OwningProcess</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz -w 5 mail.example.ch 25

sudo ss -ltnp 'sport = :25'</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) kombinerar under Windows namnupplösning, destinationsadress, porttest och valfri routediagnostik. Under Unix öppnar [`nc`](https://man.openbsd.org/nc.1) riktade TCP- eller UDP-anslutningar; [`telnet`](https://man.openbsd.org/telnet.1) kan också öppna en TCP-port, men hanterar då Telnet-protokollet och är ingen helt transparent ersättning. [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) läser socketinformation från Linux-kärnan och kan filtrera tillstånd, timers, köer och processer. Ett lyckat anslutningstest öppnar och stänger en verklig TCP-anslutning; på produktionstjänster bör det därför användas riktat och inte som högfrekvent portskanning.

### Paketinsamling och tillståndsfilter

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für den TCP-Paketmitschnitt">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">pktmon filter remove
pktmon filter add MailTcp -t TCP -p 25
pktmon start --capture --pkt-size 0 --file-name C:\Temp\mail-tcp.etl
# Problem reproduzieren, danach:
pktmon stop
pktmon etl2pcap C:\Temp\mail-tcp.etl --out C:\Temp\mail-tcp.pcapng</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sudo tcpdump -i any -nn -s 0 -w /tmp/mail-tcp.pcap \
  'host 192.0.2.25 and tcp port 25'

ss -tin state established '( dport = :25 or sport = :25 )'</code></pre>
  </div>
</div>

[`Pktmon`](https://learn.microsoft.com/windows-server/networking/technologies/pktmon/pktmon) samlar in paket i Windows-nätverksstacken och kan konvertera sitt ETL-format till PCAPNG. [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) använder under Unix ett capture-uttryck och skriver här en PCAP-fil för senare analys. En fullständig insamling behåller payload och kan innehålla åtkomstuppgifter eller meddelandeinnehåll. Den ska behandlas som en e-postexport: filtrera snävt, samla in kortvarigt, lagra skyddat och radera efter analysen. Med TLS förblir adresser, portar, paketstorlekar, timing, handskakningsmetadata och TCP-signaler synliga, men inte det krypterade applikationsinnehållet.

### Signalbild och sannolikaste överlämningspunkt

| Observation | TCP-perspektiv | Sannolikaste nästa kontroll |
|---|---|---|
| SYN upprepas, inget svar | anslutningsetableringen når ingen svarande peer | DNS-mål, rutt, Security Group, brandväggsdrop, asymmetrisk returväg |
| SYN får omedelbart RST | målsökvägen fungerar, socketen avvisas | listener, bindningsadress, NAT-mål, portkonfiguration |
| handskakning lyckas, ingen banner | TCP är etablerat, applikationen levererar inga data | worker, trådpool, proxyprotokoll, applikationslogg, backendberoende |
| RST direkt efter klientdata | peer eller middlebox förkastar sessionen | fel protokoll, ACL, TLS på klartextport, inspektionsenhet |
| ökande retransmissioner och Dup ACKs | förlust, omordning eller saknade capture-paket | gränssnittsfel, överbelastning, MTU/PMTUD, vägbyte, motinsamling |
| Zero Window | mottagaren rapporterar full buffert | CPU, applikationstråd, lagring, Garbage Collection, socketbuffert |
| många gamla `CLOSE-WAIT` | peeren har avslutat, lokal app stänger inte | filbeskrivarläcka, blockerad kodväg, connection pool |
| anslutningar misslyckas endast under last | tillstånds- eller kapacitetsgräns | listen-backlog, SYN-cookies, NAT/Conntrack, efemära portar, FD-gräns |
| små data fungerar, stora hänger | storleksberoende vägfel | MTU, PMTUD/ICMP-filter, MSS-clamping, TLS-record-/proxygränser |

## Kapacitet och felområden

Efter den enskilda anslutningen följer kapacitetsfrågan. Varje socket tar upp tillstånd hos klient och server, ofta dessutom i brandvägg, NAT, lastbalanserare eller proxy. Det hållbara antalet anslutningar bestäms därför inte enbart av CPU eller bandbredd, utan av den minsta tabellen, kön eller portmängden längs den verkliga vägen.

Administratörer bör åtminstone skilja mellan följande resurser:

- **Listen-backlog och SYN-backlog:** begränsar fullt respektive halvt öppna anslutningar;
- **Filbeskrivare och processgränser:** en ledig port räcker inte om applikationen inte längre kan acceptera en socket;
- **Efemära portar:** utgående reläer och proxyservrar kan parallellt nå samma målsocket endast med olika lokala sockets;
- **NAT- och Conntrack-tabeller:** håller egna timers och kan uttömmas trots lediga resurser i ändsystemen;
- **Arbetsminne per socket:** sänd-/mottagningsbuffertar, TLS-tillstånd och applikationskontext multipliceras med antalet anslutningar;
- **TIME-WAIT-fördelning:** den aktivt stängande sidan bär detta tillstånd; arkitekturbeslut flyttar därför lasten;
- **Lastbalanserarens affinitet och returväg:** tillståndsfulla enheter måste se båda riktningarna för samma anslutning.

Att blint förkorta TIME-WAIT-, keepalive- eller retransmissionstimers löser sällan orsaken. Det förändrar protokollskyddet och felsemantiken. Först måste anslutningsfrekvens, hålltid, fyrtuplefördelning och den faktiskt uttömda tabellen mätas.

## Övervakning för meddelandemiljöer

Av dessa tillstånd följer övervakningen. Enbart gränssnittsutnyttjande missar långsamma handskakningar, retransmissioner, reset-stormar och en mottagare med stängt fönster. En användbar dashboard kombinerar därför värd-, nät- och applikationsperspektiv och visar minst följande tidsserier sida vid sida:

| Metrik | Utsaga | Larm endast i samband med |
|---|---|---|
| Anslutningsframgångsgrad och p95/p99 | nåbarhet och etableringstid | mål, IP-familj, port och plats |
| nya anslutningar per sekund | last och churn | etablerade anslutningar och genomsnittlig hålltid |
| `SYN-SENT` / `SYN-RECEIVED` | halvöppna anslutningar | ålder, backlog-drops, SYN-cookies |
| RST efter riktning och livsfas | aktiva avbrott | före/efter handskakning, sändande nod, applikationskod |
| retransmissionsgrad | förlust- eller omordningssignal | RTT, gränssnittsdrops, captureposition |
| RTT och RTO | vägfördröjning och timerbeteende | region, motpart, tid på dagen |
| Zero-Window-varaktighet | mottagaren läser inte | CPU, lagring, JVM/GC, trådpooler |
| `CLOSE-WAIT`-ålder | lokal applikation stänger inte | process, release och connection pool |
| NAT-/Conntrack-utnyttjande | tillstånd i mellannoden | timeouts, portanvändning, returväg |

Syntetiska porttester bör kopplas till ett protokolltest. För SMTP innebär det: TCP-connect, banner, EHLO, valfritt StartTLS och en kontrollerad transaktion är separata mätpunkter. Endast då förblir det tydligt om nätväg, TLS eller e-postapplikationen orsakar latensen.

## Säkerhet och härdning

TCP tillhandahåller ordning och felkorrigering, men varken konfidentialitet eller kryptografisk identitet. Applikationer kompletterar därför oftast med [TLS](/kb/tls), nät vid behov med IPsec och särskilda TCP-anslutningar med TCP-AO. Sekvensnummer försvårar visserligen blind injicering, men ersätter ingen autentisering. Härdning måste dessutom beakta SYN-floods, förfalskade resettar, för stora köer och listenerar som kan missbrukas ([RFC 9293, avsnitt 7](https://datatracker.ietf.org/doc/html/rfc9293#section-7)).

För exponerade meddelandetjänster är följande kontroller relevanta:

- publicera endast faktiskt nödvändiga listeners och adressfamiljer;
- modellera server- och klientroller separat i brandväggar i stället för att generellt definiera «TCP tillåtet»;
- övervaka SYN-backlog, SYN-cookies och anslutningsfrekvenser innan hårda rate limits träffar legitima e-postservrar;
- acceptera Proxy Protocol endast mellan uttryckligen betrodda peers;
- dokumentera TCP- och TLS-terminering, så att käll-IP, timeout och RST-ansvar förblir spårbara;
- skydda paketinsamlingar och socketdiagnostik som potentiellt känsliga driftsdata;
- undvik onödig middlebox-manipulation av MSS, alternativ och idle timers, och testa varje nödvändig avvikelse.

## Teknisk historia

Den idag självklara separationen mellan IP och TCP uppstod först under internets tidiga utveckling. [RFC 675](https://datatracker.ietf.org/doc/html/rfc675) beskrev 1974 ännu ett gemensamt Internet Transmission Control Program. Den senare uppdelningen gav IP den paketförmedlade transporten mellan nät och TCP den tillförlitliga end-to-end-sessionen. [RFC 793](https://datatracker.ietf.org/doc/html/rfc793) standardiserade detta TCP 1981.

| År | Utveckling |
|---:|---|
| 1974 | RFC 675 beskriver Internet Transmission Control Program som föregångare |
| 1981 | RFC 793 definierar TCP; RFC 791 beskriver det separata Internet Protocol |
| 1989 | RFC 1122 konsoliderar värdkrav och preciserar många TCP-regler |
| 1992 | RFC 1323 inför stora fönster och Timestamps för högprestandalänkar |
| 2009 | RFC 5681 konsoliderar Slow Start, Congestion Avoidance, Fast Retransmit och Fast Recovery |
| 2011 | RFC 6298 standardiserar beräkningen av Retransmission Timeout |
| 2014 | RFC 7323 ersätter RFC 1323 och uppdaterar Window Scale samt Timestamps |
| 2022 | RFC 9293 ersätter RFC 793 och sammanför åter kärnstandarden som varit spridd över årtionden |

Implementeringarna utvecklades parallellt: Selective Acknowledgment, Explicit Congestion Notification, moderna algoritmer för överbelastningskontroll, TCP Fast Open och offloading i operativsystem utökar kärnan. För felsökning förblir dock den ursprungliga modellen med socketpar, sekvensutrymme, fönster, tillstånd och timers hållbar.

## Källor

- [RFC Editor – RFC 9293: Transmission Control Protocol](https://www.rfc-editor.org/info/rfc9293)
- [RFC 9293 – Transmission Control Protocol](https://datatracker.ietf.org/doc/html/rfc9293) – konsoliderad TCP-specifikation, header, tillstånd, sekvensmodell, gränssnitt och säkerhet.
- [IANA – Service Name and Transport Protocol Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) – registrerade tjänster och portintervall.
- [RFC 6298 – Computing TCP's Retransmission Timer](https://datatracker.ietf.org/doc/html/rfc6298) – RTT-, RTO- och backoff-algoritm.
- [RFC 5681 – TCP Congestion Control](https://datatracker.ietf.org/doc/html/rfc5681) – Slow Start, Congestion Avoidance, Fast Retransmit och Fast Recovery.
- [Wireshark User’s Guide – TCP Analysis](https://www.wireshark.org/docs/wsug_html_chunked/ChAdvTCPAnalysis.html) – härledning av Retransmission, Zero Window, Out-of-Order och ytterligare analyssignaler.
- [RFC 7323 – TCP Extensions for High Performance](https://datatracker.ietf.org/doc/html/rfc7323) – Window Scale, Timestamps, RTTM och PAWS.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – TCP-port-, namnupplösnings- och routediagnostik under Windows.
- [OpenBSD Manual – nc(1)](https://man.openbsd.org/nc.1) – TCP-/UDP-anslutningar, listeners och porttester med Netcat.
- [OpenBSD Manual – telnet(1)](https://man.openbsd.org/telnet.1) – Telnet-klient, anslutningsetablering och protokollbeteende.
- [iproute2 – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html) – sockettillstånd, timers, köer och filter.
- [Microsoft Learn – Pktmon](https://learn.microsoft.com/windows-server/networking/technologies/pktmon/pktmon) – Windows-paketmonitor och PCAPNG-konvertering.
- [tcpdump(1) Manual](https://www.tcpdump.org/manpages/tcpdump.1.html) – capture-alternativ, filter och utdatafiler.
- [RFC 675 – Specification of Internet Transmission Control Program](https://datatracker.ietf.org/doc/html/rfc675) – teknisk föregångare till TCP och IP.
- [RFC 793 – Transmission Control Protocol](https://datatracker.ietf.org/doc/html/rfc793) – ursprunglig TCP-specifikation från 1981.
