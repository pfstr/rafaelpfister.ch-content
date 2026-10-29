---
title: "NatJack: NAT-tabellattacker förklarade, vem som påverkas och vilka åtgärder som behövs"
navTitle: "NatJack förklarat"
description: "NatJack är en ny attackklass mot NAT-enheter: En angripare bakom samma router, brandvägg eller Docker-brygga kan ta över andras TCP-anslutningar, fånga upp DNS-svar eller slå ut NAT. Så fungerar attacken, vilka produkter som påverkas, när åtgärder krävs och vilka som fungerar."
date: "2026-10-28"
kategorie: "Säkerhetsbrister"
timeToRead: "10 min lästid"
themen:
  - sicherheitsluecken
produkte:
  - "sicherheitsluecken"
protokolle:
  - "tcp"
  - "dns"
  - "container"
  - "haertung"
hauptthema: "sicherheitsluecken"
slug: "natjack-nat-tabellattacker-forklarade-vem-som-paverkas-och-vilka-atgarder-som-behovs"
translationId: "article-223ba44d60e8cbc8"
translationOf: natjack-nat-tabellen-angriff
url: https://rafaelpfister.ch/sv/blog/natjack-nat-tabellattacker-forklarade-vem-som-paverkas-och-vilka-atgarder-som-behovs
translationSourceHash: c29a779430bbe591ef42a7db22926ad80fea50613dbc60458b9e88e40edadc02
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T21:32:31.487Z
translationReview: automatic
---

# NatJack: NAT-tabellattacker förklarade, vem som påverkas och vilka åtgärder som behövs

NatJack är en attackklass mot enheter som använder Network Address Translation (NAT): routrar, brandväggar, hypervisornätverk, containerbryggor och NAT-gateways i molnet. En angripare som befinner sig bakom samma NAT-enhet som sitt offer manipulerar dess NAT-tabell så att svar från internet inte längre når offret utan angriparen. Därmed kan befintliga TCP-anslutningar tas över, DNS-svar fångas upp och förfalskas eller NAT-enheten blockeras för alla andra användare. Forskningen publicerades av Malcolm Stagg (SODIUM-24, LLC) den 6 augusti 2026 på Black Hat USA, samordnad via CERT/CC (VU#268870) med 13 tillverkare. I 23 testade produkter rapporterades 95 enskilda fynd; enligt vitboken var varje testad enhet med NAT sårbar för minst en del av attackerna.

Kort slutsats: För ett hemnät med egna enheter behövs inga åtgärder. Åtgärder krävs där externa eller opålitliga användare och arbetslaster använder samma NAT-enhet som era produktiva system: gäst-wifi vid företagsbrandväggen, Docker- och Kubernetes-värdar med externa images, test-VM:ar i NAT-nätet på en Hyper-V-värd samt hosting- och coworkingmiljöer. Den mest effektiva åtgärden är kryptering (TLS även internt, säkrat DNS), den näst mest effektiva är att separera förtroendezoner på egna NAT-enheter.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Stöd vid bedömningen</p>
<p>Om ni vill kontrollera om er brandväggs-, container- eller hypervisormiljö påverkas, använd gärna <a href="https://adeptio.ch/en">kontaktformuläret på adeptio.ch</a>.</p>
</div>

## Vad en NAT-tabell gör

Bakom en NAT-enhet har enheterna privata adresser enligt RFC 1918, till exempel `10.0.10.21`. När en enhet upprättar en anslutning utåt ersätter NAT-enheten avsändaradress och port med sin externa adress och en extern port och sparar kopplingen i NAT-tabellen. När serverns svar kommer tillbaka slår enheten upp i tabellen vilken intern enhet det tillhör. Under Linux hanterar Netfilter detta med sin anslutningsspårning `conntrack`; de flesta vanliga routrar, OpenWRT, Docker och Kubernetes använder denna kod.

Ursprungligen var NAT en övergångslösning för bristen på IPv4-adresser och ingen säkerhetsgräns. I praktiken har det dock blivit en sådan: På lager 2 separerar VLAN, portisolering, Dynamic ARP Inspection och vSwitch-isolering enheterna ordentligt från varandra, och ARP-spoofing är knappast längre möjligt i välskötta nät. På lager 3/4 sammanförs de separerade näten däremot ofta i samma NAT-enhet igen. Det är just här NatJack angriper.

![Delad NAT-enhet med två separata VLAN: offrets NAT-post ersätts av en post från angriparen, varefter målsserverns svar går till angriparen](../images/natjack-nat-tabellen-angriff/geteiltes-nat.svg)

## Så fungerar attacken

Alla NatJack-varianter följer samma mönster: Angriparen får NAT-enheten att ta bort offrets tabellpost och skapar omedelbart en egen post med samma externa parametrar. NAT-enheten vidarebefordrar sedan serverns nästa svar till angriparen. Varken servern eller offret får något felmeddelande.

Det möjliggörs av beteenden som utformades för ett samarbetsvilligt nät och som är meningsfulla där:

- **Loose Mode:** Netfilter lägger som standard även till TCP-anslutningar i tabellen vars handskakning den inte har sett. Detta ska bevara anslutningar efter en omstart av routern eller ett vägbyte, men tillåter också att poster skapas utan en giltig anslutningsuppbyggnad.
- **Generös kontroll av reset-paket:** NAT-enheten kontrollerar sekvensnummer mindre strikt än ändpunkterna själva, så att den inte kastar bort giltiga paket. Paket som en ändpunkt skulle ignorera kan NAT-enheten ändå bedöma som ett anslutningsslut.
- **Port Preservation och Endpoint-Independent Mapping:** Netfilter använder om möjligt den interna porten som extern port och fortsätter att använda en redan tilldelad koppling för alla mål. Beteendet vid portkonflikter gör det möjligt att härleda vilka portar andra enheter använder just nu.
- **Begränsad tabell:** Om tabellen är full kastar Netfilter slumpmässigt bort ännu ej bekräftade poster, till exempel öppna DNS-förfrågningar. Bekräftade anslutningar ligger kvar i upp till fem dagar.

För de flesta varianter måste angriparen kunna skicka paket med förfalskad avsändaradress. Om angripare och offer finns i samma subnät räcker det med en enhet eller container med råsocket-rättigheter (`NET_RAW`). Upstream-varianten fungerar även över subnäts- och VLAN-gränser; då behöver angriparen dessutom en egen server på internet som kan skicka paket med förfalskad avsändaradress. Vitboken beskriver paketsekvenserna och proof-of-concept-koden; den aviserade tekniska rapporten med testframework har ännu inte publicerats.

## De fem sårbarheterna i korthet

Vitboken beskriver fem sårbarheter. Bedömningarna är typiska värden och kan avvika beroende på produkt.

| Sårbarhet | Effekt | CVSS 3.1 (typiskt) |
|---|---|---|
| Denial of Service | Angriparen fyller NAT-tabellen med bekräftade poster. Ingen annan enhet kan upprätta nya anslutningar, i Netfilter i upp till fem dagar eller tills omstart. | 7.4 (High) |
| Avslöjande av port och IP | Angriparen upptäcker att en annan enhet är ansluten till en viss server och fastställer dess port och interna adress. Detta fungerar främst som ett förstadium till övriga attacker. | 4.7 (Medium) |
| DNS-kapning via UDP | Angriparen fångar med viss sannolikhet upp DNS-svaret för en annan enhet och skickar ett förfalskat svar, till exempel med adressen till en egen server. | 10.0 (Critical) |
| TCP-kapning (downstream) | Angriparen tar över en befintlig TCP-anslutning från en enhet i samma subnät, läser serverns svar och skickar i offrets namn. | 9.6 (Critical) |
| TCP-kapning (upstream) | Som ovan, men över subnäts- och VLAN-gränser med hjälp av angriparens externa server. | 10.0 (Critical) |

Alla protokoll utan stark kryptering påverkas: HTTP, DNS via UDP, Telnet-baserade administrationsprotokoll samt e-postprotokoll som SMTP och IMAP när de körs utan TLS. Vid en TLS-anslutning kan angriparen visserligen avbryta anslutningen, men får inga läsbara data och kan inte injicera något. Vitboken påpekar dessutom att attackpaket kan skickas med reducerad TTL så att de når NAT-enheten men inte kommer vidare in i nätet. Det försvårar upptäckt.

Jämfört med ARP-spoofing har NatJack en liknande effekt, men kringgår etablerade skyddsmekanismer på lager 2. Tidigare arbete av Yang et al. (CVE-2023-30305 till CVE-2023-30314) visade TCP-kapning via NAT-tabeller i wifi-routrar med osäkra icke-standardalternativ; NatJack fungerar i standardkonfigurationen.

## Vem som påverkas

Enligt vitboken är praktiskt taget alla Linux-baserade routrar sårbara, liksom system med helt annan kodbas. Följande översikt sammanfattar läget i patchmatrisen på natjack.io (senast uppdaterad den 19 augusti 2026).

| Område | Produkter | Status |
|---|---|---|
| Linux / Netfilter | Linux-kärnan, OpenWRT, DD-WRT, nästan alla Linux-baserade routrar | Delfix med CVE-2026-63913 från Linux 7.1; ökar svårighetsgraden, men ger inget fullständigt skydd. OpenWRT och DD-WRT behöver Netfilter-ändringarna. |
| BSD | FreeBSD pf, OpenBSD pf | FreeBSD från 15.0.0 kraftigt begränsat genom en ändring från OpenBSD, äldre versioner påverkas. OpenBSD har varit betydligt bättre skyddat sedan 2016. |
| Windows | Windows NAT (Hyper-V vSwitch med NAT) | CVE-2026-56181 (juli 2026) och CVE-2026-56179 (augusti 2026), båda klassade som måttliga. Åtgärden för CVE-2026-56179 är inaktiverad som standard och aktiveras via registret. Windows Internet Connection Sharing granskas. |
| Nätverksenheter | Cisco Firepower och Meraki, NETGEAR Nighthawk, Google Nest WiFi Pro, SpaceX Starlink, Amazon eero | Beroende på tillverkare delskydd, konfigurationsrekommendationer eller ingen uppdatering. eero har publicerat ett eget meddelande. |
| Hotspots | iOS Personlig hotspot, macOS internetdelning, Android-hotspot | påverkas |
| Virtualisering och containrar | Hyper-V, KVM/QEMU, Apple Virtualization Framework, Docker (Bridge), FreeBSD Jails, GKE, MicroK8s med Calico eller Flannel, Azure AKS | påverkas; skydd främst genom konfiguration (se nedan) |
| Moln | AWS NAT Gateway och NLB, Google Cloud NAT, Cloud Run, Azure Firewall, Azure NAT Gateway, Azure Functions | AWS har härdat på serversidan, kunder behöver inte göra något. AWS Lambda skyddas av avsaknaden av `NET_RAW`. För övriga tjänster, kontrollera status på natjack.io. |

## Behövs åtgärder?

Avgörande är vem som finns bakom samma NAT-enhet; huruvida er routermodell finns med i listan är sekundärt. Varje variant förutsätter att angriparen redan kontrollerar en enhet, VM eller container bakom NAT-enheten. Där det endast rör egna, betrodda enheter skulle en angripare med den åtkomsten ändå ha enklare vägar. Vitboken konstaterar därför att de flesta hemanvändare inte påverkas allvarligt, medan konsekvenserna är betydligt svårare för företag och tjänsteleverantörer.

![Tre nivåer: inget omedelbart åtgärdsbehov för enbart egna enheter, kontrollera och härda vid separata gästnät på samma brandvägg, åtgärder krävs vid externa användare eller arbetslaster bakom samma NAT](../images/natjack-nat-tabellen-angriff/massnahmen-noetig.svg)

I schweiziska små och medelstora företag är mellannivån vanligast: Gäst-wifi är separerat med VLAN men går ut på internet via samma brandvägg som företagsnätet. VLAN-separeringen förhindrar ARP-spoofing och direkt åtkomst, men båda näten delar fortfarande NAT-tabellen. För downstream-varianten skulle angriparen behöva befinna sig i samma subnät som sitt offer; upstream-varianten fungerar över VLAN-gränsen, men är mer komplicerad och kräver en extern server. Åtgärder är tydligt nödvändiga där externa personer finns i samma subnät (hotell-, vårdmottagnings- eller skol-wifi utan klientisolering) och på container- och Kubernetes-värdar där arbetslaster med olika förtroendenivåer körs. Ett komprometterat npm- eller PyPI-paket i en container räcker där som utgångspunkt.

För e-posttjänster är det värt att särskilt granska interna anslutningar utan TLS: skannrar, verksamhetsapplikationer och övervakningssystem som levererar okrypterat till ett internt relä via port 25, samt IMAP utan TLS. Artikeln [E-postleverans via ett relä: kontrollera TLS och autentisering](/blog/mail-relay-tls-authentisierung-pruefen) beskriver hur du kontrollerar TLS och autentisering vid reläet.

## Åtgärder

Vitbokens rekommendationer kan ordnas i fem lager. De skiljer sig åt i vilka sårbarheter de hjälper mot.

![Fem skyddslager: kryptering, separation av förtroendezoner, anti-spoofing, NAT-härdning och övervakning, vardera med sin effekt på kapning, portavslöjande och Denial of Service](../images/natjack-nat-tabellen-angriff/schutzschichten.svg)

### Kryptering

TLS på alla anslutningar, även mellan interna system, gör kapade TCP-anslutningar värdelösa: Angriparen kan avbryta dem men varken läsa dem eller injicera data. För DNS rekommenderar vitboken DNSSEC mot förfalskade svar samt DNS over HTTPS eller DNS over TLS mot avlyssning. Kryptering hjälper inte mot portavslöjande och Denial of Service.

### Separation av förtroendezoner

Driftssätt opålitliga enheter och arbetslaster i egna subnät och VLAN, med L3/L4-ACL:er som förhindrar all åtkomst till betrodda värdar. Mer effektivt är en egen NAT-enhet eller en egen extern adress per förtroendezon, eftersom grunden för alla varianter saknas utan en gemensam NAT-tabell. I molnet innebär det att betrodda och externa arbetslaster inte ska drivas bakom samma NAT Gateway och att dedikerade i stället för delade IP-adresser ska användas för serverless-tjänster. I Kubernetes körs opålitliga poddar på egna noder.

### Anti-spoofing

Aktivera IP Source Guard eller en jämförbar funktion på switchen respektive åtkomstpunkten och uRPF (Reverse Path Filtering) på routern, så att klienter endast kan skicka med sin egen adress. I containrar förhindrar borttagning av capabilityn `NET_RAW` att en komprometterad process kan skapa förfalskade paket. Docker tilldelar denna capability som standard.

```bash
docker run --cap-drop=NET_RAW --user 1000:1000 example/app:1.4
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `run` | Startar en ny container från en image |
| `--cap-drop=NET_RAW` | Tar bort containerns rätt till råsockets; förfalskade IP-paket kan då inte längre skapas |
| `--user 1000:1000` | Kör processen med användar- och grupp-ID 1000 i stället för som root |
| `example/app:1.4` | Containerns image och tagg (exempel) |

</details>

`ping` behöver i vissa images `NET_RAW`; på aktuella kärnor fungerar det vanligen även utan via oprivilegierade ICMP-sockets. I Kubernetes hanteras detta av containerns `securityContext`:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - NET_RAW
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `runAsNonRoot: true` | Kubernetes startar inte containern om den skulle köras som root |
| `allowPrivilegeEscalation: false` | Hindrar processen från att få ytterligare rättigheter, till exempel via setuid-program |
| `capabilities.drop: [NET_RAW]` | Tar bort råsocket-capabilityn, på samma sätt som `--cap-drop=NET_RAW` i Docker |

</details>

Pod Security Standard `restricted` kräver att containrar avger alla capabilities (`drop: [ALL]`), och omfattar därmed `NET_RAW`.

### NAT-härdning

Installera tillgängliga uppdateringar: kärna respektive firmware med Netfilter-fixen, Windows-uppdateringarna från juli och augusti 2026 på Hyper-V-värdar med NAT-nät och FreeBSD från 15.0.0. Om kärnan i er distribution innehåller fixen för CVE-2026-63913 framgår av kärnpaketets ändringslogg.

På Linux-routrar och containervärdar kan Loose Mode dessutom stängas av. Därefter tar Netfilter endast upp TCP-anslutningar med fullständig handskakning i tabellen.

```bash
sysctl net.netfilter.nf_conntrack_tcp_loose
sudo sysctl -w net.netfilter.nf_conntrack_tcp_loose=0
echo "net.netfilter.nf_conntrack_tcp_loose = 0" | sudo tee /etc/sysctl.d/90-natjack.conf
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `sysctl <schlüssel>` | Visar aktuellt värde; `1` innebär att Loose Mode är aktivt (standard) |
| `-w` | Skriver värdet direkt till den körande kärnan, utan omstart |
| `nf_conntrack_tcp_loose=0` | Stänger av övertagandet av anslutningar utan observerad handskakning |
| `tee /etc/sysctl.d/90-natjack.conf` | Sparar inställningen permanent så att den överlever en omstart |

</details>

Loose Mode finns av en anledning: Efter en omstart av routern eller en failover i ett kluster med `conntrackd` avbryts befintliga anslutningar utan det. Kontrollera detta före ändringen, särskilt på högtilgängliga brandväggar. Vitboken påpekar uttryckligen att flera åtgärder måste väga säkerhet, tillgänglighet och protokollkompatibilitet mot varandra.

Ytterligare inställningar, om produkten erbjuder dem:

- **Anslutningsgräns per klient:** Vitboken rekommenderar en gräns på cirka 10 000 samtidiga anslutningar per klient. I pf-baserade brandväggar som OPNsense och pfSense finns inställningen i de avancerade alternativen för brandväggsregeln (pf: `max-src-states`).
- **Stäng av Port Preservation och Endpoint-Independent Mapping:** Försvårar portidentifiering. I pf ersätter en NAT-regel utan `static-port` källporten slumpmässigt. Observera att vissa VoIP- och peer-to-peer-applikationer bygger på fasta mappningar.
- **ISN-randomisering:** I pf via `modulate state` i stället för `keep state`. Regler med `flags any` bör undvikas eftersom de tillåter anslutningar utan giltig handskakning.

### Övervakning

Vitboken nämner flera tecken på attackförsök: en full eller nästan full NAT-tabell, paketflöden från en klient över ett stort intervall av källportar, samma IP-adress vid två MAC-adresser eller en MAC-adress med flera IP-adresser, flöden av RST-paket med varierande sekvensnummer, sekvensnummer långt utanför TCP-fönstret och paket med påfallande låg TTL. Under Linux visas tabellens fyllnadsgrad genom en jämförelse av två värden:

```bash
cat /proc/sys/net/netfilter/nf_conntrack_count /proc/sys/net/netfilter/nf_conntrack_max
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `nf_conntrack_count` | Antal poster som för närvarande är upptagna i tabellen |
| `nf_conntrack_max` | Tabellens maximala storlek; beror på arbetsminnet (vid 4 GB eller mer 262 144 poster) |

</details>

Om det första värdet varaktigt ligger nära det andra är det anledning att utreda, oavsett NatJack. Många brandväggar visar samma nyckeltal som "State Table" eller "Sessions" i dashboarden och kan larma vid ett tröskelvärde.

## Bedömning

NatJack utnyttjar antaganden som har funnits i NAT-implementationer i årtionden, inte ett enskilt programmeringsfel. Därför finns det heller ingen patch som helt eliminerar attackklassen: Linux och Windows har ökat svårighetsgraden, AWS har förbättrat på serversidan, och många enheter förblir utan uppdatering. De långsiktiga åtgärderna ligger i den egna arkitekturen: krypterade protokoll även i det interna nätet och ingen gemensam NAT-enhet för system som ni litar på i olika hög grad. Den som har genomfört båda kan hantera NatJack enligt normal patchrytm.

## Källor

1.  [NatJack (natjack.io)](https://natjack.io/): Forskarens projektsida med patchmatris per produkt, CVE-lista och rekommendationer; matrisens status den 19 augusti 2026.

2.  [NatJack White Paper 1.0 (PDF)](https://natjack.io/natjack_white_paper_1.0.pdf): Malcolm Stagg, SODIUM-24, LLC, augusti 2026; teknisk beskrivning av de fem sårbarheterna, CVSS-bedömningar, åtgärder och indikatorer.

3.  [Black Hat USA 2026: Breaking Trust Boundaries](https://blackhat.com/us-26/briefings/schedule/#breaking-trust-boundaries-exploiting-design-assumptions-in-network-infrastructure-53311): Föredrag från den 6 augusti 2026, där attackklassen offentliggjordes.

4.  [Microsoft: CVE-2026-56181](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-56181): Windows NAT, downstream-variant i Hyper-V; uppdatering juli 2026.

5.  [Microsoft: CVE-2026-56179](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-56179): Windows NAT, upstream-variant i Hyper-V; uppdatering augusti 2026, åtgärd aktiveras via registret.

6.  [Linux-kärnan: Netfilter-commit för CVE-2026-63913](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/net/netfilter/nf_conntrack_proto_tcp.c?id=2bb6d82b586ea5a4cb73bbdd6b7432e96096bc77): Delåtgärd i TCP-spårningen i Netfilter.

7.  [FreeBSD: pf-commit](https://github.com/freebsd/freebsd-src/commit/9485d22286603ef2ac2e2009dd7dfcc6a59e2283): Ändring övertagen från OpenBSD som i FreeBSD 15.0.0 kraftigt försvårar de flesta varianterna.

8.  [eero: Protecting eero customers from NatJack](https://eero.com/blog/articles/protecting-eero-customers-from-natjack): Tillverkarens meddelande om de egna enheterna.

9.  [AWS: Troubleshoot NAT gateways](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-troubleshooting.html): Information om portutnyttjande efter den serversidiga härdningen som AWS hänvisar till i sitt uttalande.

10.  [Linux-kärnans dokumentation: Netfilter Conntrack Sysctl](https://docs.kernel.org/networking/nf_conntrack-sysctl.html): Beskrivning av `nf_conntrack_tcp_loose`, `nf_conntrack_max` och övriga Conntrack-parametrar.
