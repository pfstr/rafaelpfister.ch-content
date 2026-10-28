---
title: "Hva er egentlig Tailscale, og hvilke fordeler tilbyr det sammenlignet med tradisjonelle VPN-forbindelser?"
navTitle: "Tailscale vs. VPN"
description: "Tailscale bygger et mesh-VPN på WireGuard, der enheter kobler seg direkte til hverandre i stedet for via en sentral VPN-konsentrator. Hvordan Coordination Server, NAT-traversering og DERP-reléer samvirker, hvilke fordeler det har sammenlignet med IPsec- og SSL-VPN, og hvilke avhengigheter og begrensninger du bør kjenne til før bruk."
date: "2026-10-01"
kategorie: "VPN og fjerntilgang"
timeToRead: "11 min lesetid"
themen:
  - vpn-fernzugriff
produkte:
  - "tailscale"
protokolle:
  - "tcp"
  - "haertung"
slug: "hva-er-egentlig-tailscale-og-hvilke-fordeler-tilbyr-det-sammenlignet-med-tradisjonelle-vpn"
translationId: "article-91fddf1e0239f5c4"
translationOf: tailscale-vorteile-vpn
translationSourceHash: f7642d64432f28cd0e53fd833c1accbd7a2a206550be04e07ace9cea8d23a04f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-27T09:23:52.661Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/hva-er-egentlig-tailscale-og-hvilke-fordeler-tilbyr-det-sammenlignet-med-tradisjonelle-vpn
---

# Hva er egentlig Tailscale, og hvilke fordeler tilbyr det sammenlignet med tradisjonelle VPN-forbindelser?

Tailscale er en VPN-tjeneste som kobler enheter til et privat nettverk, et såkalt tailnet. Teknisk er det basert på WireGuard. Forskjellen fra et klassisk bedrifts-VPN ligger i topologien: Enhetene oppretter sine krypterte tunneler direkte med hverandre (mesh). En sentral VPN-konsentrator som all trafikk går gjennom, faller bort. Det sentrale er kun administrasjonen: En Coordination Server distribuerer offentlige nøkler, adresser og tilgangsregler, men ser ikke selve nyttedataene.

For å komme i gang trenger du bare en konto hos en Identity Provider (Microsoft, Google, GitHub, Apple eller en OIDC-leverandør) og klienten på hver enhet. Portvideresendinger i brannmuren er som regel ikke nødvendig. Dermed er Tailscale utbredt både for hjemmenettverk og for fjerntilgang til servere og kundesystemer.

## Slik fungerer et tradisjonelt VPN

Et klassisk VPN for fjerntilgang fungerer etter hub-and-spoke-prinsippet. I kanten av bedriftsnettverket står en VPN-gateway (brannmur eller appliance) som er tilgjengelig fra internett. Klienten på den bærbare PC-en oppretter en tunnel til denne gatewayen; vanlige alternativer er IPsec/IKEv2 (UDP 500 og 4500), SSL-VPN-varianter over TCP 443 eller OpenVPN. Etter innlogging får enheten en adresse fra en pool og ruter til de interne nettverkene.

Denne modellen har fungert godt i flere tiår, men har strukturelle egenskaper som skaper arbeid i dagens drift:

- **Offentlig tilgjengelig gateway:** VPN-konsentratoren må være tilgjengelig fra internett og er dermed et foretrukket angrepsmål. Sårbarheter i VPN-appliances har blitt aktivt utnyttet gjentatte ganger de siste årene; den amerikanske myndigheten CISA beordret til og med frakobling av berørte Ivanti-gatewayer fra nettet i januar 2024 gjennom Emergency Directive 24-01.
- **Single point of failure og flaskehals:** All trafikk går via gatewayen. Dersom den svikter eller båndbredden er oppbrukt, rammer det alle brukere samtidig.
- **Omveier (hairpinning):** Når to ansatte på hjemmekontor arbeider mot den samme serveren i skyen, går trafikken først til datasenteret og deretter ut igjen.
- **Grove tilgangsrettigheter:** Etter at forbindelsen er opprettet, får enheten ofte tilgang til et helt nettverkssegment. Finkornede regler per bruker og tjeneste er mulig, men vedlikeholdes sjelden konsekvent.
- **Site-to-site-arbeid:** Hvert ekstra sted trenger sin egen tunnel med avstemte parametere (Phase-1/Phase-2-proposals, pre-shared keys eller sertifikater, faste offentlige IP-adresser).

## Slik er Tailscale bygget opp

Tailscale skiller Control Plane fra Data Plane. Control Plane er Coordination Server, som Tailscale drifter som en skytjeneste. Data Plane er WireGuard-tunnelene mellom enhetene.

| Komponent | Oppgave |
|---|---|
| Klient (`tailscaled`) | Oppretter nøkkelparet lokalt, bygger WireGuard-tunneler til de andre nodene, håndhever tilgangsreglene lokalt |
| Coordination Server | Autentiserer enheter via Identity Provider, distribuerer offentlige nøkler, adresser, DNS-innstillinger og policyen til alle noder |
| DERP-reléer | Videresender krypterte pakker når en direkte forbindelse ikke kan opprettes |
| Peer Relays | Egne enheter i tailnettet som fungerer som relé med høyere gjennomstrømming; de foretrekkes fremfor DERP |

Den private nøkkelen til en enhet forlater aldri enheten. Coordination Server kjenner bare de offentlige nøklene og kan derfor ikke dekryptere trafikken. Hver enhet får en fast adresse fra området `100.64.0.0/10` (adresserommet for Carrier-Grade NAT) samt en IPv6-adresse fra `fd7a:115c:a1e0::/48`. Via MagicDNS er enhetene også tilgjengelige under navnet sitt, for eksempel `nas` eller `nas.tailnet-name.ts.net`.

### NAT-traversering: hvorfor portvideresending ikke er nødvendig

De fleste enheter befinner seg bak en NAT-ruter eller brannmur og er ikke direkte tilgjengelige utenfra. Tailscale løser dette med NAT-traversering: Begge sider finner sin offentlige adresse og den tildelte porten via STUN, utveksler denne informasjonen via Coordination Server og sender samtidig UDP-pakker til hverandre. De utgående pakkene oppretter en tilstandsoppføring i begge brannmurene, slik at pakker fra motparten deretter kan komme inn (UDP Hole Punching).

Hvis dette ikke lykkes, for eksempel ved restriktive brannmurer som blokkerer utgående UDP, eller ved bestemte former for Carrier-Grade NAT, går trafikken via et DERP-relé over HTTPS. Også der forblir den ende-til-ende-kryptert med WireGuard; reléet ser bare krypterte pakker. Prisen er høyere latenstid og lavere gjennomstrømming. Peer Relays reduserer denne ulempen ved at en egen enhet med god tilkobling tar seg av videresendingen.

## Fordelene sammenlignet med et tradisjonelt VPN

| Kriterium | Tradisjonelt VPN | Tailscale |
|---|---|---|
| Topologi | Hub-and-spoke via en sentral gateway | Mesh, direkte forbindelser mellom enhetene |
| Inngående porter | Gatewayen må være tilgjengelig fra internett | Ingen inngående portvideresending nødvendig |
| Autentisering | Lokale kontoer, RADIUS, sertifikater, ofte separat MFA | Innlogging via eksisterende Identity Provider, inkludert MFA-en der |
| Tilgangsrettigheter | Ofte per nettverkssegment | Per bruker, gruppe, enhet og port i én sentral policy |
| Nytt sted | Site-to-site-tunnel med avstemte parametere | Installer klient eller Subnet Router |
| Utfall av sentralen | Ingen tilgang for noen lenger | Eksisterende forbindelser fortsetter; nye enheter og policyendringer venter |
| Protokoll | IPsec, SSL-VPN, OpenVPN | WireGuard |

### Mindre angrepsflate

Siden klientene oppretter forbindelsene sine utgående, trenger ingen enhet en åpen port på internett. En server som bare skal være tilgjengelig via tailnettet, kan binde tjenestene sine utelukkende til Tailscale-grensesnittet. Den blir da ikke synlig for portskannere på internett. WireGuard i seg selv er med rundt 4000 linjer med kjerne-kode betydelig mindre enn typiske IPsec- eller SSL-VPN-implementeringer og bruker et fast sett med moderne metoder (Curve25519, ChaCha20-Poly1305, BLAKE2s). Det finnes ingen forhandling av cipher suites, slik det er i IPsec og som regelmessig fører til feilkonfigurasjoner.

### Identitet i stedet for nettverksadresse

Hver enhet i tailnettet er knyttet til en bruker eller en tagg. Innloggingen skjer via Identity Provider-en du allerede bruker; MFA og Conditional Access fra Microsoft Entra ID gjelder dermed også for nettverkstilgangen. Nye enheter kan bare kobles til tailnettet etter vellykket innlogging hos Identity Provider-en, og hver enhetsnøkkel utløper som standard etter 180 dager. Når en ansatt forlater virksomheten, sperrer du kontoen hos Identity Provider-en; med SCIM-provisionering (fra Standard-abonnementet) deaktiveres brukeren automatisk i tailnettet, og enhetene mister tilgangen.

### Tilgangsregler etter prinsippet om minste privilegium

Som standard kan enhver enhet i et nytt tailnett nå alle andre enheter. For produksjonsbruk definerer du tilgangsrettighetene i en sentral policyfil (HuJSON). Følgende eksempel tillater administratorgruppen SSH og HTTPS på alle servere med taggen `tag:server`, mens alle øvrige brukere bare får HTTPS til intranettserveren:

```json
{
  "groups": {
    "group:admins": ["admin@example.com"]
  },
  "tagOwners": {
    "tag:server": ["group:admins"]
  },
  "grants": [
    {
      "src": ["group:admins"],
      "dst": ["tag:server"],
      "ip":  ["tcp:22", "tcp:443"]
    },
    {
      "src": ["autogroup:member"],
      "dst": ["intranet"],
      "ip":  ["tcp:443"]
    }
  ],
  "hosts": {
    "intranet": "100.101.102.103"
  }
}
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `groups` | Definerer brukergrupper; medlemmer angis via innloggingsadressen sin hos Identity Provider-en |
| `tagOwners` | Angir hvem som kan tildele en tagg til enheter; taggede enheter tilhører ingen bruker, men taggen |
| `grants` | Liste over tillatte forbindelser; det som ikke er uttrykkelig tillatt, blokkeres |
| `src` | Kilde for forbindelsen: bruker, gruppe, tagg eller `autogroup:member` (alle brukere i tailnettet) |
| `dst` | Mål for forbindelsen: tagg, vertsalias, enhet eller subnett |
| `ip` | Tillatte protokoller og porter, for eksempel `tcp:22` eller `*` for alt |
| `hosts` | Aliasnavn for tailnett-adresser eller subnett som brukes i reglene |

</details>

Reglene håndheves lokalt av hver klient. En pakke som policyen ikke tillater, forkastes allerede av målenheten. Coordination Server distribuerer bare policyen.

### Direkte forbindelser i stedet for omveier

Fordi tunnelene går direkte mellom enhetene, tar trafikken den korteste veien. To enheter på samme kontor kommuniserer lokalt, og en bærbar PC på hjemmekontor når en skyserver direkte. Det reduserer latenstiden og avlaster internettforbindelsen ved hovedlokasjonen.

### Koble til eksisterende nettverk

Ikke alle enheter kan kjøre en Tailscale-klient, for eksempel skrivere, eldre NAS-systemer eller industrielle styresystemer. I slike tilfeller overtar en Subnet Router rollen som gateway: En Linux-server i målnettverket annonserer det lokale subnettet i tailnettet (advertise), og autoriserte enheter når det via denne.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
echo 'net.ipv4.ip_forward = 1' | \
  sudo tee /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
sudo tailscale up \
  --advertise-routes=192.168.10.0/24 \
  --advertise-tags=tag:server
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `curl -fsSL …/install.sh \| sh` | Laster ned det offisielle installasjonsskriptet og setter opp distribusjonens pakkelager |
| `net.ipv4.ip_forward = 1` | Tillater Linux-kjernen å videresende pakker mellom grensesnitt; uten denne innstillingen fungerer ingen Subnet Router |
| `sysctl -p <datei>` | Laster innstillingen umiddelbart, uten omstart |
| `tailscale up` | Logger enheten inn i tailnettet; ved første kjøring vises en innloggingslenke |
| `--advertise-routes=<subnetz>` | Annonserer det angitte subnettet i tailnettet; flere subnett skilles med komma |
| `--advertise-tags=<tag>` | Tildeler en tagg til enheten som tilgangsreglene viser til |

</details>

Den annonserte ruten må deretter godkjennes i administrasjonskonsollen, såfremt ingen automatisk godkjenning (`autoApprovers`) er konfigurert i policyen. Tilsvarende kan en enhet brukes som Exit Node med `--advertise-exit-node`. Klienter som velger denne Exit Node, videresender da all internettrafikken sin gjennom den, noe som tilsvarer full tunnel-modus i et klassisk VPN.

### Mindre driftsarbeid

Coordination Server administrerer nøkkelrotasjon, adressetildeling, DNS og ruting. En ny lokasjon trenger en Subnet Router med internettforbindelse, men ingen fast offentlig IP-adresse og ingen koordinering av IPsec-parametere med motparten. For fjerntilgang til servere finnes også Tailscale SSH (innlogging via tailnett-identitet uten distribuerte SSH-nøkler) og `tailscale serve` (deling av en lokal nettjeneste i tailnettet).

## Begrensninger og avhengigheter

Tailscale erstatter den klassiske VPN-gatewayen, men flytter en del av ansvaret til en ekstern tjeneste. Du bør kontrollere disse punktene før innføring.

| Tema | Hva du må være oppmerksom på |
|---|---|
| Avhengighet av leverandøren | Control Plane er en skytjeneste fra Tailscale Inc. Ved driftsstans fortsetter eksisterende forbindelser, men nye enheter, innlogginger og policyendringer er ikke mulig før tjenesten er gjenopprettet |
| Metadata | Enhetsnavn, tailnett-adresser, offentlige IP-adresser, brukerkontoer og tilkoblingstidspunkter behandles hos leverandøren; ikke nyttedataene |
| Tillit til nøkkeldistribusjon | Coordination Server bestemmer hvilke offentlige nøkler en enhet godtar. Tailnet Lock krever i tillegg en signatur fra betrodde egne enheter |
| Kildekode | Klientkjernen er åpen kildekode (BSD-3-Clause), men Coordination Server er det ikke. Headscale er et åpen kildekode, selvhostet alternativ med redusert funksjonsomfang |
| Adressekonflikter | `100.64.0.0/10` brukes også av enkelte leverandører for Carrier-Grade NAT og av andre VPN-produkter; overlapp fører til rutingsproblemer |
| Sameksistens med andre VPN-klienter | En annen VPN-klient med full tunnel kan omdirigere Tailscale-trafikken. Området `100.64.0.0/10` og Tailscale-tjenesten må da unntas fra tunnelen |
| Ytelse via reléer | Dersom ingen direkte forbindelse kan opprettes, reduseres gjennomstrømmingen merkbart via DERP; `tailscale netcheck` viser om UDP utgående fungerer |
| Ingen erstatning for webfiltrering | Tailscale regulerer tilgangen til interne ressurser. Det håndterer ikke innholdsfiltrering og inspeksjon av internettrafikk |

For selskaper i Sveits gjelder den reviderte personvernloven (revDSG). Siden leverandøren behandler personopplysninger om brukerne (kontoer, enheter, tilkoblingsmetadata), skal Tailscale føres opp i fortegnelsen over behandlingsaktiviteter dersom virksomheten din må føre en slik (fra 250 ansatte eller ved behandlinger med høy risiko). Kontroller leverandørens databehandleravtale (Data Processing Addendum) og det rettslige grunnlaget for utlevering til utlandet i henhold til art. 16 revDSG, for eksempel sertifisering under Swiss-U.S. Data Privacy Framework eller standard kontraktsklausuler. Dersom databehandling hos leverandøren er utelukket, gjenstår Headscale som selvdriftet Control Plane.

> **EU-merknad:** For filialer i EU gjelder GDPR og reglene for overføring til tredjeland (art. 44 ff. GDPR). Vurderingen er innholdsmessig tilsvarende; der er EU-U.S. Data Privacy Framework avgjørende.

## Kostnader

Personal-abonnementet er gratis og omfatter opptil seks brukere, et ubegrenset antall brukerenheter og 50 taggede enheter (per oktober 2026). For forretningsbruk koster Standard-abonnementet 8 USD og Premium 18 USD per bruker per måned. Premium inkluderer blant annet Network Flow Logs, loggstrømming og utvidede alternativer for Tailscale SSH. Enterprise tilbys individuelt.

## Hvem Tailscale passer for

Tailscale passer særlig når brukere og ressurser er distribuert: hjemmekontor, skyservere hos flere leverandører, små avdelingskontorer uten fast IP-adresse eller fjernvedlikehold hos kunder. For små og mellomstore bedrifter kan det erstatte en VPN-gateway fullstendig. I større miljøer kjører det ofte parallelt med det eksisterende VPN-et, for eksempel for administrativ tilgang til servere, der de finkornede tilgangsreglene gir størst nytte.

Det er mindre egnet når kravene forutsetter en fullstendig selvdrevet infrastruktur og Headscale ikke dekker funksjonsomfanget, eller når all internettrafikk må filtreres sentralt. I så fall trengs fortsatt en Secure Web Gateway som supplerer Tailscale.

For testing er det nok å installere klienten på to enheter og logge inn med samme konto. Med `tailscale status` ser du deretter alle enhetene i tailnettet, og med `tailscale ping <gerät>` kan du kontrollere om en direkte forbindelse eller et relé brukes.

## Endringslogg

- **25. oktober 2026:** Kildelenker oppdatert til Tailscales nye adresser.

## Kilder

1.  [Tailscale: How Tailscale works](https://tailscale.com/blog/how-tailscale-works): Arkitektur med Coordination Server, WireGuard-mesh og lokal håndheving av reglene.

2.  [Tailscale: How NAT traversal works](https://tailscale.com/blog/how-nat-traversal-works): detaljert forklaring av STUN, UDP Hole Punching og tilfellene der et relé blir nødvendig.

3.  [Tailscale Docs: DERP servers](https://tailscale.com/docs/reference/derp-servers): Funksjon og kryptering for reléserverne.

4.  [Tailscale Docs: Tailscale Peer Relays](https://tailscale.com/docs/features/peer-relay): egne enheter som relé med prioritet foran DERP.

5.  [Tailscale Docs: Grants](https://tailscale.com/docs/features/access-control/grants): syntaks for tilgangsreglene i policyfilen.

6.  [Tailscale Docs: Subnet routers](https://tailscale.com/docs/features/subnet-routers): oppsett av Subnet Routers, inkludert IP-videresending og godkjenning av rutene.

7.  [Tailscale Docs: Exit nodes](https://tailscale.com/docs/features/exit-nodes): videresende all internettrafikk via en enhet i tailnettet.

8.  [Tailscale Docs: Tailnet Lock](https://tailscale.com/docs/features/tailnet-lock): signering av nye enheter av egne, betrodde noder.

9.  [Tailscale: Pricing](https://tailscale.com/pricing): abonnementer og grenser, hentet 1. oktober 2026.

10.  [WireGuard: Next Generation Kernel Network Tunnel (Whitepaper)](https://www.wireguard.com/papers/wireguard.pdf): protokolldesign og de kryptografiske metodene som brukes.

11.  [GitHub: tailscale/tailscale](https://github.com/tailscale/tailscale): kildekoden til klienten under BSD-3-Clause.

12.  [GitHub: juanfont/headscale](https://github.com/juanfont/headscale): åpen kildekode-implementering av Coordination Server for selvdrift.

13.  [CISA: Emergency Directive 24-01](https://www.cisa.gov/news-events/directives/ed-24-01-mitigate-ivanti-connect-secure-and-ivanti-policy-secure-vulnerabilities): pålegg om å koble sårbare Ivanti-VPN-gatewayer fra nettet i januar 2024.

14.  [Fedlex: Bundesgesetz über den Datenschutz (DSG)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 16 om utlevering av personopplysninger til utlandet.
