---
title: "NatJack: NAT-tabellangrep forklart, hvem som er berørt og hvilke tiltak som er nødvendige"
navTitle: "NatJack forklart"
description: "NatJack er en ny angrepsklasse mot NAT-enheter: En angriper bak samme ruter, brannmur eller Docker-bro kan overta andres TCP-forbindelser, fange opp DNS-svar eller lamme NAT. Slik fungerer angrepet, hvilke produkter som er berørt, når tiltak er nødvendig og hvilke som virker."
date: "2026-10-28"
kategorie: "Sikkerhetssårbarheter"
timeToRead: "10 min lesetid"
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
slug: "natjack-nat-tabellangrep-forklart-hvem-som-er-berort-og-hvilke-tiltak-som-er-nodvendige"
translationId: "article-223ba44d60e8cbc8"
translationOf: natjack-nat-tabellen-angriff
url: https://rafaelpfister.ch/no/blog/natjack-nat-tabellangrep-forklart-hvem-som-er-berort-og-hvilke-tiltak-som-er-nodvendige
translationSourceHash: c29a779430bbe591ef42a7db22926ad80fea50613dbc60458b9e88e40edadc02
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T21:33:24.633Z
translationReview: automatic
---

# NatJack: NAT-tabellangrep forklart, hvem som er berørt og hvilke tiltak som er nødvendige

NatJack er en angrepsklasse mot enheter som utfører Network Address Translation (NAT): rutere, brannmurer, hypervisornettverk, containerbroer og skybaserte NAT-gatewayer. En angriper som befinner seg bak samme NAT-enhet som offeret, manipulerer NAT-tabellen slik at svar fra internett ikke lenger kommer frem til offeret, men til angriperen. Dette kan brukes til å overta eksisterende TCP-forbindelser, fange opp og forfalske DNS-svar eller blokkere NAT-enheten for alle andre brukere. Forskningen ble publisert av Malcolm Stagg (SODIUM-24, LLC) 6. august 2026 på Black Hat USA, koordinert via CERT/CC (VU#268870) med 13 produsenter. I 23 testede produkter ble 95 enkeltfunn rapportert; ifølge whitepaperet var alle testede enheter med NAT sårbare for minst én del av angrepene.

Kort oppsummert: For et hjemmenettverk med egne enheter er det ikke nødvendig å gjøre noe. Tiltak er nødvendige der eksterne eller ikke-klarerte brukere og arbeidslaster bruker samme NAT-enhet som produksjonssystemene dine: gjeste-Wi-Fi på bedriftsbrannmuren, Docker- og Kubernetes-verter med eksterne images, test-VM-er i NAT-nettet til en Hyper-V-vert, hosting- og coworking-miljøer. Det mest effektive tiltaket er kryptering (TLS også internt, sikret DNS), mens det nest mest effektive er å skille tillitssonene på egne NAT-enheter.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Hjelp med vurderingen</p>
<p>Hvis du ønsker å undersøke om brannmur-, container- eller hypervisormiljøet ditt er berørt, kan du bruke <a href="https://adeptio.ch/en">kontaktskjemaet på adeptio.ch</a>.</p>
</div>

## Hva en NAT-tabell gjør

Bak en NAT-enhet har enhetene private adresser fra RFC 1918, for eksempel `10.0.10.21`. Når en enhet oppretter en utgående forbindelse, erstatter NAT-enheten avsenderadressen og porten med sin eksterne adresse og en ekstern port, og lagrer tilordningen i NAT-tabellen. Når svaret fra serveren kommer tilbake, slår enheten opp i tabellen for å finne hvilken intern enhet det tilhører. På Linux håndterer Netfilter dette med sin forbindelsessporing `conntrack`; de fleste vanlige rutere, OpenWRT, Docker og Kubernetes bruker denne koden.

Opprinnelig var NAT en overgangsløsning for mangelen på IPv4-adresser, ikke en sikkerhetsgrense. I praksis har det imidlertid blitt en slik grense: På lag 2 skiller VLAN-er, portisolasjon, Dynamic ARP Inspection og vSwitch-isolasjon enhetene tydelig fra hverandre, og ARP-spoofing er knapt mulig i godt vedlikeholdte nettverk. På lag 3/4 samles de adskilte nettverkene imidlertid ofte igjen i samme NAT-enhet. Det er nettopp her NatJack angriper.

![Delt NAT-enhet med to adskilte VLAN-er: Offerets NAT-oppføring erstattes av en oppføring fra angriperen, og svar fra målserveren går deretter til angriperen](../images/natjack-nat-tabellen-angriff/geteiltes-nat.svg)

## Slik fungerer angrepet

Alle NatJack-varianter følger samme mønster: Angriperen får NAT-enheten til å fjerne offerets tabelloppføring og oppretter umiddelbart en egen oppføring med de samme eksterne parameterne. NAT-enheten videresender deretter serverens neste svar til angriperen. Verken serveren eller offeret mottar en feilmelding.

Dette er mulig på grunn av atferd som er utviklet for et samarbeidsvillig nettverk, der den også er fornuftig:

- **Loose Mode:** Netfilter tar som standard også TCP-forbindelser inn i tabellen der den ikke har sett håndtrykket. Dette skal bevare forbindelser etter omstart av ruter eller endret rute, men gjør det også mulig å opprette oppføringer uten gyldig forbindelsesopprettelse.
- **Romslig kontroll av reset-pakker:** NAT-enheten kontrollerer sekvensnumre mindre strengt enn endepunktene selv, slik at den ikke forkaster gyldige pakker. Pakker som et endepunkt ville ignorert, kan NAT-enheten likevel tolke som avslutning av forbindelsen.
- **Port Preservation og Endpoint-Independent Mapping:** Netfilter overtar så langt det er mulig den interne porten som ekstern port og bruker en tidligere tildelt tilordning videre for alle mål. Atferden ved portkonflikter kan brukes til å utlede hvilke porter andre enheter bruker akkurat nå.
- **Begrenset tabell:** Når tabellen er full, forkaster Netfilter tilfeldige oppføringer som ennå ikke er bekreftet, for eksempel åpne DNS-forespørsler. Bekreftede forbindelser blir stående i opptil fem dager.

For de fleste variantene må angriperen kunne sende pakker med forfalsket avsenderadresse. Hvis angriper og offer er i samme subnett, holder det med en enhet eller container med råsocket-rettigheter (`NET_RAW`). Oppstrømsvarianten fungerer også på tvers av subnett- og VLAN-grenser; da trenger angriperen i tillegg sin egen server på internett som kan sende pakker med forfalsket avsenderadresse. Whitepaperet beskriver pakkesekvensene og proof-of-concept-koden; den annonserte tekniske rapporten med testrammeverket er ennå ikke publisert.

## De fem sårbarhetene i oversikt

Whitepaperet beskriver fem sårbarheter. Vurderingene er typiske verdier; de varierer etter produkt.

| Sårbarhet | Konsekvens | CVSS 3.1 (typisk) |
|---|---|---|
| Denial of Service | Angriperen fyller NAT-tabellen med bekreftede oppføringer. Ingen andre enheter kan opprette nye forbindelser, i Netfilter i opptil fem dager eller frem til omstart. | 7.4 (High) |
| Avsløring av port og IP | Angriperen oppdager at en annen enhet er koblet til en bestemt server, og finner porten og den interne adressen. Dette brukes først og fremst som et fortrinn for de øvrige angrepene. | 4.7 (Medium) |
| DNS-kapring over UDP | Angriperen fanger med en viss sannsynlighet opp DNS-svaret til en annen enhet og sender et forfalsket svar, for eksempel med adressen til en egen server. | 10.0 (Critical) |
| TCP-kapring (nedstrøms) | Angriperen overtar en eksisterende TCP-forbindelse fra en enhet i samme subnett, leser serverens svar og sender på vegne av offeret. | 9.6 (Critical) |
| TCP-kapring (oppstrøms) | Som over, men på tvers av subnett- og VLAN-grenser ved hjelp av en ekstern server kontrollert av angriperen. | 10.0 (Critical) |

Alle protokoller uten sterk kryptering er berørt: HTTP, DNS over UDP, Telnet-baserte administrasjonsprotokoller og e-postprotokoller som SMTP og IMAP når de kjører uten TLS. Ved en TLS-forbindelse kan angriperen riktignok avbryte forbindelsen, men får ikke lesbare data og kan ikke injisere noe. Whitepaperet påpeker dessuten at angrepspakker kan sendes med redusert TTL slik at de når NAT-enheten, men ikke kommer videre inn i nettverket. Dette vanskeliggjør oppdagelse.

Sammenlignet med ARP-spoofing har NatJack en tilsvarende effekt, men omgår de etablerte sikkerhetsmekanismene på lag 2. Tidligere arbeid av Yang et al. (CVE-2023-30305 til CVE-2023-30314) viste TCP-kapring via NAT-tabeller på Wi-Fi-rutere med usikre ikke-standardalternativer; NatJack fungerer i standardkonfigurasjonen.

## Hvem er berørt

Ifølge whitepaperet er praktisk talt alle Linux-baserte rutere sårbare, i tillegg til systemer med helt andre kodebaser. Oversikten nedenfor oppsummerer statusen i patchmatrisen på natjack.io (sist oppdatert 19. august 2026).

| Område | Produkter | Status |
|---|---|---|
| Linux / Netfilter | Linux-kjerne, OpenWRT, DD-WRT, nesten alle Linux-baserte rutere | Delfiks med CVE-2026-63913 fra Linux 7.1; øker innsatsen, men gir ikke fullstendig beskyttelse. OpenWRT og DD-WRT trenger Netfilter-endringene. |
| BSD | FreeBSD pf, OpenBSD pf | FreeBSD fra 15.0.0 er sterkt forbedret gjennom en endring fra OpenBSD, eldre versjoner er berørt. OpenBSD har vært betydelig bedre beskyttet siden 2016. |
| Windows | Windows NAT (Hyper-V vSwitch med NAT) | CVE-2026-56181 (juli 2026) og CVE-2026-56179 (august 2026), begge vurdert som moderate. Tiltaket for CVE-2026-56179 er deaktivert som standard og aktiveres via registeret. Windows Internet Connection Sharing er under vurdering. |
| Nettverksenheter | Cisco Firepower og Meraki, NETGEAR Nighthawk, Google Nest WiFi Pro, SpaceX Starlink, Amazon eero | Avhengig av produsent: delvis beskyttelse, konfigurasjonsanbefalinger eller ingen oppdatering. eero har publisert en egen melding. |
| Hotspots | iOS Personlig hotspot, macOS Internett-deling, Android-hotspot | berørt |
| Virtualisering og containere | Hyper-V, KVM/QEMU, Apple Virtualization Framework, Docker (Bridge), FreeBSD Jails, GKE, MicroK8s med Calico eller Flannel, Azure AKS | berørt; beskyttelse hovedsakelig gjennom konfigurasjon (se nedenfor) |
| Sky | AWS NAT Gateway og NLB, Google Cloud NAT, Cloud Run, Azure Firewall, Azure NAT Gateway, Azure Functions | AWS har herdet på serversiden, og kunder trenger ikke gjøre noe. AWS Lambda er beskyttet fordi `NET_RAW` mangler. For de øvrige tjenestene må statusen på natjack.io kontrolleres. |

## Er tiltak nødvendige?

Det avgjørende er hvem som befinner seg bak samme NAT-enhet; om rutermodellen din står på listen, er av underordnet betydning. Alle varianter forutsetter at angriperen allerede kontrollerer en enhet, en VM eller en container bak NAT-enheten. Der dette kun er egne, klarerte enheter, ville en angriper med denne tilgangen uansett hatt enklere veier. Whitepaperet fastslår derfor at de fleste hjemmebrukere ikke er alvorlig berørt, mens konsekvensene er betydelig alvorligere for virksomheter og tjenesteleverandører.

![Tre nivåer: ingen umiddelbar handlingsplikt ved kun egne enheter, vurder og herd ved adskilte gjestenett på samme brannmur, tiltak nødvendig ved eksterne brukere eller arbeidslaster på samme NAT](../images/natjack-nat-tabellen-angriff/massnahmen-noetig.svg)

I sveitsiske SMB-er er mellomnivået det vanligste tilfellet: Gjeste-Wi-Fi er adskilt med VLAN, men går via samme brannmur til internett som bedriftsnettverket. VLAN-separasjonen hindrer ARP-spoofing og direkte tilgang, men begge nettverkene deler fortsatt NAT-tabellen. For nedstrømsvarianten måtte angriperen være i samme subnett som offeret; oppstrømsvarianten fungerer på tvers av VLAN-grensen, men er mer krevende og trenger en ekstern server. Tiltak er klart nødvendige der fremmede sitter i samme subnett (Wi-Fi på hotell, legekontor eller skole uten klientisolasjon) og på container- og Kubernetes-verter der arbeidslaster med ulike tillitsnivåer kjører. En kompromittert npm- eller PyPI-pakke i en container er nok som utgangspunkt der.

For e-posttjenester lønner det seg å se spesifikt på interne forbindelser uten TLS: skannere, fagapplikasjoner og overvåkingssystemer som leverer ukryptert via port 25 til et internt relé, samt IMAP uten TLS. Artikkelen [E-postsending via et relé: Kontroller TLS og autentisering](/blog/mail-relay-tls-authentisierung-pruefen) beskriver hvordan du kontrollerer TLS og autentisering på reléet.

## Tiltak

Anbefalingene i whitepaperet kan ordnes i fem lag. De skiller seg i hvilke sårbarheter de bidrar mot.

![Fem beskyttelseslag: kryptering, separasjon av tillitssoner, anti-spoofing, NAT-herding og overvåking, med henholdsvis effekt på kapring, portavsløring og Denial of Service](../images/natjack-nat-tabellen-angriff/schutzschichten.svg)

### Kryptering

TLS på alle forbindelser, også mellom interne systemer, gjør overtagne TCP-forbindelser verdiløse: Angriperen kan avbryte dem, men verken avlytte eller injisere data. For DNS anbefaler whitepaperet DNSSEC mot forfalskede svar samt DNS over HTTPS eller DNS over TLS mot avlytting. Kryptering hjelper ikke mot portavsløring og Denial of Service.

### Separasjon av tillitssoner

Drift ikke-klarerte enheter og arbeidslaster i egne subnett og VLAN-er, med L3/L4-ACL-er som blokkerer all tilgang til klarerte verter. Enda mer effektivt er en egen NAT-enhet eller egen ekstern adresse per tillitssone, for uten en delt NAT-tabell mangler grunnlaget for alle variantene. I skyen betyr dette: Ikke kjør klarerte og eksterne arbeidslaster bak samme NAT Gateway, og bruk dedikerte i stedet for delte IP-adresser for serverless-tjenester. I Kubernetes kjøres ikke-klarerte pods på egne noder.

### Anti-spoofing

Aktiver IP Source Guard eller en tilsvarende funksjon på switchen eller tilgangspunktet, samt uRPF (Reverse Path Filtering) på ruteren, slik at klienter kun kan sende med sin egen adresse. I containere forhindrer fjerning av capability-en `NET_RAW` at en kompromittert prosess kan generere forfalskede pakker. Docker tildeler denne capability-en som standard.

```bash
docker run --cap-drop=NET_RAW --user 1000:1000 example/app:1.4
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Effekt |
|---|---|
| `run` | Starter en ny container fra et image |
| `--cap-drop=NET_RAW` | Fjerner containerens rett til råsockets; dermed kan den ikke lenger generere forfalskede IP-pakker |
| `--user 1000:1000` | Kjører prosessen med bruker- og gruppe-ID 1000 i stedet for som root |
| `example/app:1.4` | Image og tagg for containeren (eksempel) |

</details>

`ping` krever `NET_RAW` i enkelte images; på nyere kjerner fungerer det vanligvis også uten, via uprivilegerte ICMP-sockets. I Kubernetes håndteres dette av containerens `securityContext`:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - NET_RAW
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Effekt |
|---|---|
| `runAsNonRoot: true` | Kubernetes starter ikke containeren hvis den ville kjørt som root |
| `allowPrivilegeEscalation: false` | Hindrer at prosessen får ytterligere rettigheter, for eksempel via setuid-programmer |
| `capabilities.drop: [NET_RAW]` | Fjerner råsocket-capability-en, på samme måte som `--cap-drop=NET_RAW` i Docker |

</details>

Pod Security Standard `restricted` krever at containere gir fra seg alle capabilities (`drop: [ALL]`), og dekker dermed `NET_RAW`.

### NAT-herding

Installer tilgjengelige oppdateringer: kjerne eller fastvare med Netfilter-fiksen, Windows-oppdateringene fra juli og august 2026 på Hyper-V-verter med NAT-nettverk, og FreeBSD fra 15.0.0. Om distribusjonens kjerne inneholder fiksen for CVE-2026-63913, fremgår av endringsloggen for kjernepakken.

På Linux-rutere og containerverter kan Loose Mode i tillegg slås av. Deretter tar Netfilter kun TCP-forbindelser med komplett håndtrykk inn i tabellen.

```bash
sysctl net.netfilter.nf_conntrack_tcp_loose
sudo sysctl -w net.netfilter.nf_conntrack_tcp_loose=0
echo "net.netfilter.nf_conntrack_tcp_loose = 0" | sudo tee /etc/sysctl.d/90-natjack.conf
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Effekt |
|---|---|
| `sysctl <schlüssel>` | Viser gjeldende verdi; `1` betyr at Loose Mode er aktiv (standard) |
| `-w` | Skriver verdien umiddelbart til den kjørende kjernen, uten omstart |
| `nf_conntrack_tcp_loose=0` | Slår av overtakelse av forbindelser der håndtrykket ikke er observert |
| `tee /etc/sysctl.d/90-natjack.conf` | Lagrer innstillingen permanent, slik at den overlever en omstart |

</details>

Loose Mode finnes av en grunn: Etter omstart av ruteren eller failover i en klynge med `conntrackd` brytes eksisterende forbindelser uten den. Kontroller dette før endringen, særlig for høytilgjengelige brannmurer. Whitepaperet påpeker uttrykkelig at flere tiltak krever avveiinger mellom sikkerhet, tilgjengelighet og protokollkompatibilitet.

Andre innstillinger, dersom produktet tilbyr dem:

- **Forbindelsesgrense per klient:** Whitepaperet anbefaler en grense på om lag 10 000 samtidige forbindelser per klient. I pf-baserte brannmurer som OPNsense og pfSense finnes innstillingen i de avanserte alternativene for brannmurregelen (pf: `max-src-states`).
- **Slå av Port Preservation og Endpoint-Independent Mapping:** Gjør portoppdagelse vanskeligere. I pf erstatter en NAT-regel uten `static-port` kildeporten tilfeldig. Vær oppmerksom på at enkelte VoIP- og peer-to-peer-applikasjoner er avhengige av faste tilordninger.
- **ISN-randomisering:** I pf via `modulate state` i stedet for `keep state`. Regler med `flags any` bør unngås fordi de tillater forbindelser uten gyldig håndtrykk.

### Overvåking

Whitepaperet nevner flere tegn på angrepsforsøk: en full eller nesten full NAT-tabell, pakkeflommer fra én klient over et stort område av kildeporter, samme IP-adresse ved to MAC-adresser eller én MAC-adresse med flere IP-adresser, flommer av RST-pakker med varierende sekvensnumre, sekvensnumre langt utenfor TCP-vinduet og pakker med påfallende lav TTL. Under Linux viser en sammenligning av to verdier hvor full tabellen er:

```bash
cat /proc/sys/net/netfilter/nf_conntrack_count /proc/sys/net/netfilter/nf_conntrack_max
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Effekt |
|---|---|
| `nf_conntrack_count` | Antall oppføringer som for tiden er brukt i tabellen |
| `nf_conntrack_max` | Maksimal størrelse på tabellen; avhenger av arbeidsminnet (med 4 GB eller mer 262 144 oppføringer) |

</details>

Hvis den første verdien vedvarende ligger nær den andre, er det grunn til å undersøke nærmere, uavhengig av NatJack. Mange brannmurer viser den samme nøkkeltallsverdien som "State Table" eller "Sessions" i kontrollpanelet og kan varsle ved en terskelverdi.

## Vurdering

NatJack utnytter antakelser som har vært innebygget i NAT-implementeringer i flere tiår, og ikke en enkelt programmeringsfeil. Derfor finnes det heller ingen patch som fjerner angrepsklassen fullstendig: Linux og Windows har økt innsatsen som kreves, AWS har forbedret på serversiden, og mange enheter forblir uten oppdatering. De varige tiltakene ligger i egen arkitektur: krypterte protokoller også i det interne nettverket og ingen delt NAT-enhet for systemer du har ulik grad av tillit til. De som har implementert begge deler, kan møte NatJack med normal patchrytme.

## Kilder

1.  [NatJack (natjack.io)](https://natjack.io/): Forskerens prosjektside med patchmatrise per produkt, CVE-liste og anbefalinger; status for matrisen 19. august 2026.

2.  [NatJack White Paper 1.0 (PDF)](https://natjack.io/natjack_white_paper_1.0.pdf): Malcolm Stagg, SODIUM-24, LLC, august 2026; teknisk beskrivelse av de fem sårbarhetene, CVSS-vurderinger, avbøtende tiltak og indikatorer.

3.  [Black Hat USA 2026: Breaking Trust Boundaries](https://blackhat.com/us-26/briefings/schedule/#breaking-trust-boundaries-exploiting-design-assumptions-in-network-infrastructure-53311): Foredrag fra 6. august 2026, da angrepsklassen ble offentlig kjent.

4.  [Microsoft: CVE-2026-56181](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-56181): Windows NAT, nedstrømsvariant ved Hyper-V; oppdatering juli 2026.

5.  [Microsoft: CVE-2026-56179](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-56179): Windows NAT, oppstrømsvariant ved Hyper-V; oppdatering august 2026, tiltaket må aktiveres via registeret.

6.  [Linux-kjerne: Netfilter-commit for CVE-2026-63913](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/net/netfilter/nf_conntrack_proto_tcp.c?id=2bb6d82b586ea5a4cb73bbdd6b7432e96096bc77): Delvis avbøting i TCP-sporingen i Netfilter.

7.  [FreeBSD: pf-commit](https://github.com/freebsd/freebsd-src/commit/9485d22286603ef2ac2e2009dd7dfcc6a59e2283): Endring overtatt fra OpenBSD, som i FreeBSD 15.0.0 gjør de fleste variantene betydelig vanskeligere.

8.  [eero: Protecting eero customers from NatJack](https://eero.com/blog/articles/protecting-eero-customers-from-natjack): Produsentens melding om egne enheter.

9.  [AWS: Troubleshoot NAT gateways](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-troubleshooting.html): Informasjon om portbelastning etter herdingen på serversiden, som AWS viser til i sin uttalelse.

10.  [Linux-kjernedokumentasjon: Netfilter Conntrack Sysctl](https://docs.kernel.org/networking/nf_conntrack-sysctl.html): Beskrivelse av `nf_conntrack_tcp_loose`, `nf_conntrack_max` og de øvrige Conntrack-parameterne.
