---
title: "TCP: tilstander, dataflyt og diagnose"
blatt: "tcp"
description: "TCP som transportgrunnlag for meldings- og katalogtjenester: socketmodell, forbindelsestilstander, sekvensnumre, bekreftelser, flyt- og køkontroll, tidtakere, typiske feilbilder og diagnoseverktøy for administratorer."
fakten:
  - label: Fullt navn
    wert: Transmission Control Protocol
    href: https://datatracker.ietf.org/doc/html/rfc9293
  - label: Standard
    wert: RFC 9293 · STD 7
    href: https://www.rfc-editor.org/info/rfc9293
  - label: Tjenestemodell
    wert: pålitelig, ordnet, toveis bytestrøm
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-2
  - label: Endepunkt
    wert: IP-adresse og 16-bits portnummer
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.1
  - label: Forbindelse
    wert: entydig par av lokal og ekstern socket
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.1
  - label: Header
    wert: minst 20 byte pluss alternativer
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.1
  - label: Pålitelighet
    wert: sekvensnumre, kumulative ACK-er og retransmisjon
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.3
  - label: Flytkontroll
    wert: mottaksvindu rwnd
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.8.4
  - label: Køkontroll
    wert: cwnd · Slow Start · Congestion Avoidance
    href: https://datatracker.ietf.org/doc/html/rfc5681
  - label: Portområder
    wert: 0–1023 · 1024–49151 · 49152–65535
    href: https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml
  - label: Kryptering
    wert: ingen; beskyttelse med TLS, IPsec eller TCP-AO
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-7
  - label: Diagnose
    wert: porttest · sockettilstand · pakkefangst
    href: https://www.tcpdump.org/manpages/tcpdump.1.html
werbung:
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: 6c749be6d80debccf17ceb4386ba0e298c11deedf855db326e7d20da9593fc05
translationModel: gpt-5.6-terra
translatedAt: 2026-10-14T09:39:11.618Z
translationReview: automatic
---

# TCP: tilstander, dataflyt og diagnose

TCP gjør den upålitelige pakketransporten i IP om til en ordnet, toveis datastrøm for applikasjoner. Applikasjonen skriver byte til en socket; TCP segmenterer dem, nummererer posisjonen deres, bekrefter mottatte områder og overfører hull på nytt. SMTP, IMAP, POP3, LDAP og HTTPS bygger på dette løftet. Når en slik applikasjon «henger», må administratoren derfor først skille mellom om TCP fortsatt venter på en bekreftelse, eller om applikasjonsprotokollen allerede blokkerer.

Artikkelen følger livsløpet til en forbindelse: socket og listener, treveis håndtrykk, dataoverføring, flyt- og køkontroll, avslutning og TIME-WAIT. Fra dette normale forløpet utledes deretter diagnose, kapasitetsplanlegging, overvåking og herding.

TCP kjenner imidlertid verken meldinger eller brukere, sertifikater eller e-postadresser. Den bevarer heller ikke applikasjonsgrenser: To påfølgende skriveoperasjoner kan slås sammen i ett segment eller fordeles på flere segmenter. Et vellykket TCP-håndtrykk beviser derfor bare at to TCP-endepunkter kunne opprette en forbindelse. Om det deretter vises et SMTP-banner, TLS forhandles eller en LDAP-binding lykkes, avgjøres av applikasjonen over.

For administratorer er TCP mindre relevant som lagmodell enn som et **tilstands- og signalsystem**. SYN, ACK, FIN og RST viser ved hvilket overleveringspunkt en forbindelse feiler. Sockettilstander, retransmisjoner, mottaksvinduer og tidtakere forklarer hvorfor en tjeneste kan være tilgjengelig, men langsom eller tilsynelatende frosset.

Forklaringen følger en TCP-forbindelse fra første SYN via den ordnede bytestrømmen til avslutning eller reset. Vinduer, retransmisjoner, overbelastning og diagnose forklares i de respektive fasene.

## Socket- og forbindelsesmodell

Før pakker flyter, binder en applikasjon en socket til adresse og port. En server lytter for eksempel på TCP 25; hver aksepterte forbindelse identifiseres deretter entydig av kildeadresse, kildeport, destinasjonsadresse og destinasjonsport. Derfor kan tusenvis av klienter bruke samme listener, så lenge minst kildeadressen eller den ephemeral porten deres er ulik. For diagnose er «port 25» derfor ikke tilstrekkelig: Det kreves alltid hele forbindelsesparet ([RFC 9293, avsnitt 3.1](https://datatracker.ietf.org/doc/html/rfc9293#section-3.1)).

På en server binder en prosess en socket til en lokal adresse og setter den i tilstanden **LISTEN**. Valget av bindingsadresse er driftsmessig viktig:

| Binding | Synlighet | Typisk konsekvens |
|---|---|---|
| `127.0.0.1:25` eller `::1:25` | kun lokal vert | Overvåking på serveren er uproblematisk, eksterne systemer får ingen forbindelse |
| `0.0.0.0:25` | alle lokale IPv4-adresser | Listeneren følger ekstra IPv4-adresser, dersom brannmur og ruting tillater det |
| `[::]:25` | IPv6-wildcard | Dual-stack-atferd avhenger av operativsystemet og alternativet `IPV6_V6ONLY` |
| konkret tjeneste-IP | kun denne adressen | kontrollert separasjon, men følsom for IP- eller klyngeendringer |

Serverporten er vanligvis angitt gjennom en [IANA-registrering](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml); klienten velger normalt en ephemeral port fra området som administreres av operativsystemet. Brannmurer, NAT-gatewayer og lastbalanserere må derfor ikke bare tillate den kjente destinasjonsporten, men også hele returveien til den dynamiske kildeporten.

### TCP Control Block

For hver forbindelse opprettholder TCP-stakken en **Transmission Control Block** (TCB). Den inneholder blant annet tilstand, lokale og eksterne sockets, sende- og mottakssekvens, vinduer, tidtakere og sikkerhetsparametere. TCB-en er grunnen til at en forbindelse er tilstandsbevart, og til at lastbalanserere eller brannmurer må arbeide med symmetrisk returvei: Når et svarsegment når en annen node uten passende tilstand, kan den forkaste det eller svare med RST.

## Forbindelsesopprettelse, dataoverføring og avslutning

Med aktiv åpning begynner den synlige forbindelsestilstanden. Klienten sender SYN med initialt sekvensnummer, listeneren svarer med SYN+ACK og eget sekvensnummer, og deretter bekrefter klienten med ACK. Allerede disse tre segmentene forhandler alternativer som Maximum Segment Size, Window Scale, Selective Acknowledgment og Timestamps. Mangler en pakke, kommer svaret via en annen vei eller endrer et mellomledd alternativene, viser håndtrykket allerede hvor tilgjengelighet og senere ytelse skiller lag.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 976" src="/images/kb-interaktiv-tcp-verbindung.svg?v=20260813" title="Interaktive Infografik: TCP-Verbindungslebenszyklus und Diagnosepunkte" loading="lazy">
  <a href="/images/kb-interaktiv-tcp-verbindung.svg?v=20260813">Åpne infografikk om TCP-forbindelsens livssyklus</a>
</iframe>

Etter håndtrykket befinner forbindelsen seg i **ESTABLISHED**. Begge sider kan sende data uavhengig av hverandre og lukke senderetningen med FIN. En ordnet avslutning krever derfor vanligvis én FIN og ACK per retning. Siden som sender den siste ACK-en, forblir i **TIME-WAIT**, slik at forsinkede segmenter fra en gammel forbindelse ikke havner i en ny forbindelse med samme socketpar ([RFC 9293, avsnitt 3.3.2 og 3.6](https://datatracker.ietf.org/doc/html/rfc9293#section-3.3.2)).

En **RST** er ikke en ordnet avslutning. Den avbryter forbindelsen og forkaster utestående tilstand. Ved forbindelsesopprettelse betyr en gyldig RST vanligvis at måleverten er tilgjengelig, men at socketen ikke aksepteres. Etter allerede overførte data kan en RST komme fra applikasjonen, operativsystemet eller en tilstandsbevart mellomnode. Plasseringen av RST-en i pakkefangsten er derfor viktigere enn den generelle feilmeldingen «connection reset».

### Tilstandene som driftsindikatorer

| Tilstand | Teknisk betydning | Hva en opphopning antyder |
|---|---|---|
| `LISTEN` | passiv socket venter på SYN | forventet servertilstand; manglende listener peker på tjeneste- eller bindingsproblem |
| `SYN-SENT` | lokal SYN sendt, passende svar mangler | filter, rute, feil adresse eller overbelastet motpart |
| `SYN-RECEIVED` | SYN akseptert, siste ACK mangler | SYN-flood, asymmetrisk returvei, klient- eller brannmurproblem |
| `ESTABLISHED` | sekvensrom synkronisert | normalt; mange langvarige forbindelser kan være tilsiktet, for eksempel med IMAP eller HTTP/2 |
| `FIN-WAIT-1/2` | lokal side har lukket | motpart eller applikasjon avslutter retningen forsinket |
| `CLOSE-WAIT` | ekstern FIN mottatt, lokal applikasjon har ennå ikke lukket | ofte en ressurslekkasje eller blokkert applikasjonstråd |
| `LAST-ACK` | lokal FIN sendt etter ekstern avslutning | motpartens ACK mangler eller returveien er forstyrret |
| `TIME-WAIT` | gammelt firetuppel beskyttes før gjenbruk | normalt på aktivt lukkende side; problematisk først sammen med port- eller TCB-press |

Tilstandsnavnene er definert i [RFC 9293](https://datatracker.ietf.org/doc/html/rfc9293#section-3.3.2). Et høyt antall er ikke en feil i seg selv. Avgjørende er alder, vekstrate, berørte motparter og forholdet til forbindelsesrate, filbeskrivere, ephemeral portområde og applikasjonsgrenser.

## Sekvensnumre, ACK-er og retransmisjon

Etter håndtrykket teller TCP datastrømmen byte for byte. Sekvensnummeret angir den første byten i et segment; ACK-en angir den neste forventede byten og bekrefter dermed alt før den. Pakker kan gå tapt, ankomme dobbelt eller bytte rekkefølge. Mottakeren kan bufre dem, men gir applikasjonen først en sammenhengende strøm videre. Nettopp derfor kan ett enkelt manglende område blokkere en tilsynelatende rolig applikasjon, selv om senere pakker allerede har ankommet verten.

Går et segment eller ACK-en tapt, kan avsenderen oppdage det manglende området på to måter:

1. **Retransmission Timeout (RTO):** En tidtaker avledes fra glattet Round-Trip Time og variasjonen i den. Når den utløper, sendes det eldste ubekreftede segmentet på nytt. [RFC 6298](https://datatracker.ietf.org/doc/html/rfc6298) definerer beregning, minimumsverdier og eksponentiell backoff.
2. **Fast Retransmit:** Flere dupliserte ACK-er viser et hull i datastrømmen. Avsenderen kan sende det antatt tapte området på nytt uten å vente på RTO ([RFC 5681](https://datatracker.ietf.org/doc/html/rfc5681#section-3.2)).

En retransmisjon beviser ikke automatisk pakketap i nettverket. En fangst kan ha blitt gjort på bare én side, pakker kan mangle på grunn av capture-drops, eller den kan vise en spurious retransmisjon. Wireshark merker uttrykkelig sin TCP-analyse som en tolkning avledet fra den observerte pakkestrømmen ([Wireshark User’s Guide – TCP Analysis](https://www.wireshark.org/docs/wsug_html_chunked/ChAdvTCPAnalysis.html)). For en robust konklusjon bør fangster på klient, server og et relevant mellompunkt derfor korreleres så langt som mulig.

## Flytkontroll og køkontroll

Pålitelighet alene hindrer ikke at en rask avsender overbelaster mottakeren eller nettet. TCP begrenser derfor antallet ubekreftede byte samtidig med to vinduer, der det minste til slutt virker:

- **Mottaksvinduet** `rwnd` beskytter mottakeren. Det beskriver den tilgjengelige bufferen akkurat nå. Melder mottakeren Zero Window, må avsenderen vente, bortsett fra periodiske Window Probes. Et vedvarende Zero Window er ofte ikke et nettverksproblem, men betyr at den mottakende applikasjonen ikke leser socketdataene raskt nok.
- **Congestion Window** `cwnd` beskytter banen. Det føres lokalt av avsenderen og tilpasses basert på bekreftelser og tapsignaler. [RFC 5681](https://datatracker.ietf.org/doc/html/rfc5681) beskriver Slow Start, Congestion Avoidance, Fast Retransmit og Fast Recovery.

I grove trekk er det minste av `rwnd` og `cwnd` som gjelder. Høy båndbredde alene garanterer derfor ikke høy gjennomstrømning. På en bane med stort båndbredde-latensprodukt må en tilstrekkelig mengde ubekreftede data tillates. Window-Scale-alternativet utvider 16-bits-vinduet; Timestamps støtter blant annet RTT-måling og beskyttelse mot gamle sekvensnumre ([RFC 7323](https://datatracker.ietf.org/doc/html/rfc7323)). Begge alternativene forhandles i håndtrykket og kan ikke aktiveres senere for samme forbindelse.

For korte SMTP-transaksjoner dominerer ofte antallet round trips: TCP-håndtrykk, eventuelt TLS-håndtrykk og flere protokollsvar. For store vedlegg og replikasjonsstrømmer blir i tillegg RTT, tap, vinduer, offloading og congestion-control-algoritme relevante. Et gjennomstrømningsproblem er derfor først «båndbredde» når håndtrykkstid, applikasjonspauser, Zero Windows og retransmisjoner er utelukket.

## Tidtakere, keepalives og idle timeouts

Der det ikke kommer et umiddelbart svar, avgjør tidtakere neste trinn. Retransmission Timeout utløser dataretransmisjoner, Persist Timer holder en forbindelse oppe til tross for Zero Window, og TIME-WAIT hindrer at gamle segmenter havner i en ny forbindelse. Operativsystemer og applikasjoner supplerer med keepalive samt User Timeout. Den som bare logger «timeout», mister disse forskjellene og dermed den faktiske feilårsaken.

TCP-keepalives er som standard ofte deaktivert eller starter først etter lange perioder uten aktivitet. De erstatter ikke en applikasjonssidig helsesjekk. En vellykket keepalive sier bare at TCP-peeren svarer; den beviser ikke at en SMTP-worker behandler meldinger eller at en LDAP-spørring kan betjenes. Omvendt kobler brannmurer og lastbalanserere ofte fra inaktive forbindelser tidligere enn endesystemene. Da blir det på én side igjen en tilsynelatende gyldig socket, inntil neste dataforsøk utløser timeout eller RST.

For langvarige IMAP-, LDAP-pool- eller API-forbindelser må derfor fire verdier passe sammen: Applikasjonens idle timeout, TCP-keepalive-intervall, idle timeout for hver mellomnode og klientens gjenoppkoblingslogikk. Den minste uannonserte timeouten bestemmer den observerte atferden.

## Diagnose fra socket til pakke

Feilsøkingen blir pålitelig så snart den beskriver samme forbindelse som applikasjonen: kildevert, mål-navn, oppløst adresse, målport, tidspunkt og forventet protokoll. En ping erstatter ikke disse opplysningene. ICMP kan være blokkert selv om TCP fungerer; omvendt beviser et echo-svar verken en listener eller tilstanden til en stateful brannmur.

### Tilgjengelighet og listener

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

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) kombinerer navneoppløsning, måladresse, porttest og valgfri rutediagnose i Windows. I Unix åpner [`nc`](https://man.openbsd.org/nc.1) målrettet TCP- eller UDP-forbindelser; [`telnet`](https://man.openbsd.org/telnet.1) kan også åpne en TCP-port, men behandler da Telnet-protokollen og er ikke en fullstendig transparent erstatning. [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) leser socketinformasjon fra Linux-kjernen og kan filtrere tilstander, tidtakere, køer og prosesser. En vellykket connect-test åpner og lukker en reell TCP-forbindelse; på produksjonstjenester bør den derfor brukes målrettet og ikke som en høyfrekvent portskanning.

### Pakkefangst og tilstandsfiltre

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

[`Pktmon`](https://learn.microsoft.com/windows-server/networking/technologies/pktmon/pktmon) fanger pakker i Windows-nettverksstakken og kan konvertere ETL-formatet sitt til PCAPNG. [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) bruker i Unix et capture-uttrykk og skriver her en PCAP-fil for senere analyse. En fullstendig pakkefangst beholder payload og kan inneholde tilgangsdata eller meldingsinnhold. Den skal behandles som en e-posteksport: filtrer strengt, fang kortvarig, lagre beskyttet og slett etter analysen. Med TLS forblir adresser, porter, pakkestørrelser, timing, metadata fra håndtrykket og TCP-signaler synlige, men ikke det krypterte applikasjonsinnholdet.

### Signalbilde og mest sannsynlige overleveringspunkt

| Observasjon | TCP-syn | Mest sannsynlige neste kontroll |
|---|---|---|
| SYN gjentas, ingen respons | forbindelsesopprettelsen når ingen svarende peer | DNS-mål, rute, Security Group, brannmur-drop, asymmetrisk returvei |
| SYN får umiddelbart RST | målbanen fungerer, socketen avvises | listener, bindingsadresse, NAT-mål, portkonfigurasjon |
| håndtrykk lykkes, intet banner | TCP står, applikasjonen leverer ingen data | worker, threadpool, proxy-protokoll, applikasjonslogg, backendavhengighet |
| RST umiddelbart etter klientdata | peer eller middlebox forkaster økten | feil protokoll, ACL, TLS på klartekstport, inspeksjonsenhet |
| økende retransmisjoner og Dup ACK-er | tap, reordering eller manglende capture-pakker | grensesnittfeil, overbelastning, MTU/PMTUD, baneendring, motfangst |
| Zero Window | mottaker rapporterer full buffer | CPU, applikasjonstråd, lagring, garbage collection, socketbuffer |
| mange gamle `CLOSE-WAIT` | peer har avsluttet, lokal app lukker ikke | filbeskriverlekkasje, blokkert kodebane, connection pool |
| connect-feil kun under belastning | tilstands- eller kapasitetsgrense | listen-backlog, SYN-cookies, NAT/conntrack, ephemeral porter, FD-grense |
| små data fungerer, store henger | størrelsesavhengig banefeil | MTU, PMTUD/ICMP-filter, MSS-clamping, TLS-record-/proxygrenser |

## Kapasitet og feilområder

Etter enkeltforbindelsen følger kapasitetsproblemet. Hver socket opptar tilstand hos klient og server, ofte i tillegg i brannmur, NAT, lastbalanserer eller proxy. Det bærekraftige antallet forbindelser bestemmes derfor ikke bare av CPU eller båndbredde, men av den minste tabellen, køen eller portmengden langs den faktiske banen.

Administratorer bør minst skille mellom følgende ressurser:

- **Listen-backlog og SYN-backlog:** begrenser henholdsvis fullstendig og halvåpne forbindelser;
- **Filbeskrivere og prosessgrenser:** en ledig port er ikke nok når applikasjonen ikke lenger kan akseptere en socket;
- **ephemeral porter:** utgående reléer og proxyer kan bare nå samme målsocket parallelt med ulike lokale sockets;
- **NAT- og conntrack-tabeller:** opprettholder egne tidtakere og kan tømmes selv med ledige ressurser i endesystemene;
- **Arbeidsminne per socket:** sende-/mottaksbuffere, TLS-tilstand og applikasjonskontekst mangedobles med antallet forbindelser;
- **TIME-WAIT-fordeling:** den aktivt lukkende siden bærer denne tilstanden; arkitekturvalg flytter derfor belastningen;
- **Lastbalanserer-affinitet og returvei:** tilstandsbevarte enheter må se begge retninger av samme forbindelse.

Å forkorte TIME-WAIT-, keepalive- eller retransmisjonstidtakere blindt løser sjelden årsaken. Det endrer protokollbeskyttelse og feilsemantikk. Først må forbindelsesrate, holdetid, firetuppelfordeling og tabellen som faktisk er uttømt, måles.

## Overvåking for meldingsmiljøer

Overvåkingen følger av disse tilstandene. Grensesnittutnyttelse alene overser langsomme håndtrykk, retransmisjoner, reset-stormer og en mottaker med lukket vindu. Et nyttig dashboard kobler derfor vert-, nett- og applikasjonssyn og viser minst følgende tidsserier ved siden av hverandre:

| Metrikk | Utsagn | Alarm kun i sammenheng med |
|---|---|---|
| Connect-suksessrate og p95/p99 | tilgjengelighet og opprettelsestid | mål, IP-familie, port og lokasjon |
| nye forbindelser per sekund | belastning og churn | etablerte forbindelser og gjennomsnittlig holdetid |
| `SYN-SENT` / `SYN-RECEIVED` | halvåpne forbindelser | alder, backlog-drops, SYN-cookies |
| RST etter retning og livsfase | aktive avbrudd | før/etter håndtrykk, sendende node, applikasjonskode |
| retransmisjonsrate | tap- eller reorderingsignal | RTT, grensesnitt-drops, capture-posisjon |
| RTT og RTO | baneforsinkelse og tidtakeratferd | region, motpart, tid på dagen |
| Zero-Window-varighet | mottaker leser ikke | CPU, lagring, JVM/GC, threadpools |
| `CLOSE-WAIT`-alder | lokal applikasjon lukker ikke | prosess, release og connection pool |
| NAT-/conntrack-utnyttelse | tilstand i mellomnoden | timeouter, portbruk, returvei |

Syntetiske porttester bør kobles med en protokolltest. For SMTP betyr det: TCP-connect, banner, EHLO, eventuelt StartTLS og en kontrollert transaksjon er separate målepunkter. Bare slik forblir det tydelig om nettverksbane, TLS eller e-postapplikasjon forårsaker latensen.

## Sikkerhet og herding

TCP gir rekkefølge og feilkorreksjon, men verken konfidensialitet eller kryptografisk identitet. Applikasjoner supplerer dette vanligvis med [TLS](/kb/tls), nett eventuelt med IPsec og spesielle TCP-forbindelser med TCP-AO. Sekvensnumre gjør riktignok blind injisering vanskeligere, men erstatter ikke autentisering. Herding må dessuten ta hensyn til SYN-floods, forfalskede resets, for store køer og listenere som kan misbrukes ([RFC 9293, avsnitt 7](https://datatracker.ietf.org/doc/html/rfc9293#section-7)).

For eksponerte meldingstjenester er følgende kontroller relevante:

- publiser kun faktisk nødvendige listenere og adressefamilier;
- modeller server- og klientroller separat i brannmurer, i stedet for å definere generelt «TCP tillatt»;
- overvåk SYN-backlog, SYN-cookies og forbindelsesrater før harde rate limits rammer legitime e-postservere;
- aksepter Proxy Protocol kun mellom eksplisitt pålitelige peers;
- dokumenter TCP- og TLS-terminering, slik at kilde-IP, timeout og RST-ansvar forblir sporbare;
- beskytt pakkefangster og socketdiagnoser som potensielt sensitive driftsdata;
- unngå unødvendig middlebox-manipulasjon av MSS, alternativer og idle-timere, og test ethvert nødvendig avvik.

## Teknisk historie

Det som i dag er det selvfølgelige skillet mellom IP og TCP, oppstod først under den tidlige internettutviklingen. [RFC 675](https://datatracker.ietf.org/doc/html/rfc675) beskrev i 1974 fortsatt et felles Internet Transmission Control Program. Den senere oppdelingen ga IP den pakkesvitsjede transporten mellom nett og TCP den pålitelige ende-til-ende-økten. [RFC 793](https://datatracker.ietf.org/doc/html/rfc793) standardiserte dette TCP-et i 1981.

| År | Utvikling |
|---:|---|
| 1974 | RFC 675 beskriver Internet Transmission Control Program som forgjenger |
| 1981 | RFC 793 definerer TCP; RFC 791 beskriver det separate Internet Protocol |
| 1989 | RFC 1122 konsoliderer vertskrav og presiserer en rekke TCP-regler |
| 1992 | RFC 1323 innfører store vinduer og Timestamps for høyytelseslinjer |
| 2009 | RFC 5681 konsoliderer Slow Start, Congestion Avoidance, Fast Retransmit og Fast Recovery |
| 2011 | RFC 6298 standardiserer beregningen av Retransmission Timeout |
| 2014 | RFC 7323 erstatter RFC 1323 og oppdaterer Window Scale og Timestamps |
| 2022 | RFC 9293 erstatter RFC 793 og samler igjen kjernestandarden som var spredt over tiår |

Implementasjonene utviklet seg videre parallelt: Selective Acknowledgment, Explicit Congestion Notification, moderne congestion-control-algoritmer, TCP Fast Open og operativsystem-offloading utvider kjernen. For feilsøking forblir likevel den opprinnelige modellen med socketpar, sekvensrom, vinduer, tilstander og tidtakere levedyktig.

## Kilder

- [RFC Editor – RFC 9293: Transmission Control Protocol](https://www.rfc-editor.org/info/rfc9293)
- [RFC 9293 – Transmission Control Protocol](https://datatracker.ietf.org/doc/html/rfc9293) – konsolidert TCP-spesifikasjon, header, tilstander, sekvensmodell, grensesnitt og sikkerhet.
- [IANA – Service Name and Transport Protocol Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) – registrerte tjenester og portområder.
- [RFC 6298 – Computing TCP's Retransmission Timer](https://datatracker.ietf.org/doc/html/rfc6298) – RTT-, RTO- og backoff-algoritme.
- [RFC 5681 – TCP Congestion Control](https://datatracker.ietf.org/doc/html/rfc5681) – Slow Start, Congestion Avoidance, Fast Retransmit og Fast Recovery.
- [Wireshark User’s Guide – TCP Analysis](https://www.wireshark.org/docs/wsug_html_chunked/ChAdvTCPAnalysis.html) – utledning av retransmisjon, Zero Window, Out-of-Order og andre analysesignaler.
- [RFC 7323 – TCP Extensions for High Performance](https://datatracker.ietf.org/doc/html/rfc7323) – Window Scale, Timestamps, RTTM og PAWS.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – TCP-port-, navneoppløsnings- og rutediagnose i Windows.
- [OpenBSD Manual – nc(1)](https://man.openbsd.org/nc.1) – TCP-/UDP-forbindelser, listenere og porttester med Netcat.
- [OpenBSD Manual – telnet(1)](https://man.openbsd.org/telnet.1) – Telnet-klient, forbindelsesopprettelse og protokollatferd.
- [iproute2 – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html) – sockettilstander, tidtakere, køer og filtre.
- [Microsoft Learn – Pktmon](https://learn.microsoft.com/windows-server/networking/technologies/pktmon/pktmon) – Windows-pakketilstandsovervåking og PCAPNG-konvertering.
- [tcpdump(1) Manual](https://www.tcpdump.org/manpages/tcpdump.1.html) – capture-alternativer, filtre og utdatafiler.
- [RFC 675 – Specification of Internet Transmission Control Program](https://datatracker.ietf.org/doc/html/rfc675) – teknisk forgjenger til TCP og IP.
- [RFC 793 – Transmission Control Protocol](https://datatracker.ietf.org/doc/html/rfc793) – opprinnelig TCP-spesifikasjon fra 1981.
