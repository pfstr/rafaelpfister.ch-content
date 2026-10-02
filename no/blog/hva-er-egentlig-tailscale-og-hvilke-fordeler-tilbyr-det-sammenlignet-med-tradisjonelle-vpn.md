---
title: "Hva er egentlig Tailscale, og hvilke fordeler tilbyr det sammenlignet med tradisjonelle VPN-forbindelser?"
navTitle: "Tailscale vs. VPN"
description: "Tailscale bygger et mesh-VPN på WireGuard, der enheter kobler seg direkte sammen i stedet for via en sentral VPN-konsentrator. Hvordan Coordination Server, NAT-traversering og DERP-reléer samspiller, hvilke fordeler det har sammenlignet med IPsec- og SSL-VPN, og hvilke avhengigheter og begrensninger du bør kjenne til før bruk."
date: "2026-10-01"
kategorie: "VPN og fjernaksess"
timeToRead: "11 min. lesetid"
themen:
  - vpn-fernzugriff
produkte:
  - "tailscale"
protokolle:
  - "tcp"
  - "haertung"
slug: "hva-er-egentlig-tailscale-og-hvilke-fordeler-tilbyr-det-sammenlignet-med-tradisjonelle-vpn"
translationId: "article-91fddf1e0239f5c4"
aiPrompt: |
  Du bist mein Netzwerk-Assistent. Hilf mir einzuschätzen, ob Tailscale unser bestehendes VPN ganz oder teilweise ersetzen kann: Ist-Zustand aufnehmen (VPN-Gateway, Benutzer, Standorte, erreichbare Netze), Zugriffsregeln nach dem Prinzip der minimalen Rechte als Tailscale-Policy entwerfen, Subnet Router und Exit Nodes planen und Abhängigkeiten wie Identity Provider, Datenschutz nach revDSG und Koexistenz mit anderen VPN-Clients prüfen.
translationOf: tailscale-vorteile-vpn
url: https://rafaelpfister.ch/no/blog/hva-er-egentlig-tailscale-og-hvilke-fordeler-tilbyr-det-sammenlignet-med-tradisjonelle-vpn
translationSourceHash: 965ff1000bee1a9e9899d6ffd750d28abbea09dd50896551eae369b7bf20f162
translationModel: gpt-5.6-terra
translatedAt: 2026-10-02T10:07:33.149Z
translationReview: automatic
---

# Hva er egentlig Tailscale, og hvilke fordeler tilbyr det sammenlignet med tradisjonelle VPN-forbindelser?

Tailscale er en VPN-tjeneste som kobler enheter til et privat nettverk, det såkalte Tailnet. Teknisk sett er den basert på WireGuard. Forskjellen fra et klassisk bedrifts-VPN ligger i topologien: Enhetene oppretter sine krypterte tunneler direkte seg imellom (mesh). En sentral VPN-konsentrator som all trafikk går gjennom, faller bort. Kun administrasjonen forblir sentral: En Coordination Server distribuerer offentlige nøkler, adresser og tilgangsregler, men ser ikke selve dataene.

For å komme i gang trenger du bare en konto hos en Identity Provider (Microsoft, Google, GitHub, Apple eller en OIDC-leverandør) og klienten på hver enhet. Portvideresendinger i brannmuren er som regel ikke nødvendig. Derfor er Tailscale utbredt både for hjemmenettverk og fjernaksess til servere og kundesystemer.

## Slik fungerer et tradisjonelt VPN

Et klassisk VPN for fjernaksess fungerer etter hub-and-spoke-prinsippet. I utkanten av bedriftsnettverket står en VPN-gateway (brannmur eller appliance) som er tilgjengelig fra internett. Klienten på den bærbare PC-en oppretter en tunnel til denne gatewayen; vanlige løsninger er IPsec/IKEv2 (UDP 500 og 4500), SSL-VPN-varianter over TCP 443 eller OpenVPN. Etter innlogging får enheten en adresse fra en pool og ruter til de interne nettverkene.

Denne modellen har vist seg å fungere over flere tiår, men har strukturelle egenskaper som skaper arbeid i dagens drift:

- **Offentlig tilgjengelig gateway:** VPN-konsentratoren må være tilgjengelig fra internett og er dermed et foretrukket angrepsmål. Sårbarheter i VPN-appliances har gjentatte ganger blitt aktivt utnyttet de siste årene; den amerikanske myndigheten CISA beordret i januar 2024, gjennom Emergency Directive 24-01, til og med at berørte Ivanti-gatewayer skulle kobles fra nettet.
- **Single point of failure og flaskehals:** All trafikk går via gatewayen. Hvis den svikter eller båndbredden er brukt opp, rammer det alle brukere samtidig.
- **Omveier (hairpinning):** Hvis to ansatte på hjemmekontor arbeider mot samme server i skyen, går trafikken først til datasenteret og derfra ut igjen.
- **Grove tilgangsrettigheter:** Etter at forbindelsen er opprettet, får enheten ofte tilgang til et helt nettverkssegment. Finmaskede regler per bruker og tjeneste er mulig, men vedlikeholdes sjelden konsekvent.
- **Site-to-site-arbeid:** Hvert ekstra sted trenger sin egen tunnel med koordinerte parametere (fase 1-/fase 2-forslag, pre-shared keys eller sertifikater, faste offentlige IP-adresser).

## Slik er Tailscale bygget opp

Tailscale skiller Control Plane fra Data Plane. Control Plane er Coordination Server, som Tailscale driver som skytjeneste. Data Plane består av WireGuard-tunnelene mellom enhetene.

| Komponent | Oppgave |
|---|---|
| Klient (`tailscaled`) | Oppretter nøkkelparet lokalt, bygger WireGuard-tunneler til de andre nodene, håndhever tilgangsreglene lokalt |
| Coordination Server | Autentiserer enheter via Identity Provider, distribuerer offentlige nøkler, adresser, DNS-innstillinger og policyen til alle noder |
| DERP-reléer | Videresender krypterte pakker når en direkte forbindelse ikke kan opprettes |
| Peer Relays | Egne enheter i Tailnet som fungerer som relé med høyere gjennomstrømming; de prioriteres foran DERP |

Den private nøkkelen til en enhet forlater aldri enheten. Coordination Server kjenner bare de offentlige nøklene og kan derfor ikke dekryptere trafikken. Hver enhet får en fast adresse fra området `100.64.0.0/10` (adresseområdet for Carrier-Grade NAT) samt en IPv6-adresse fra `fd7a:115c:a1e0::/48`. Via MagicDNS er enhetene i tillegg tilgjengelige under navnet sitt, for eksempel `nas` eller `nas.tailnet-name.ts.net`.

### NAT-traversering: hvorfor portvideresendinger ikke er nødvendig

De fleste enheter står bak en NAT-ruter eller en brannmur og er ikke direkte tilgjengelige utenfra. Tailscale løser dette med NAT-traversering: Begge sider finner sin offentlige adresse og tildelte port via STUN, utveksler denne informasjonen via Coordination Server og sender samtidig UDP-pakker til hverandre. De utgående pakkene oppretter en tilstandsoppføring i begge brannmurene, slik at pakker fra motparten deretter kan komme inn (UDP hole punching).

Hvis dette ikke lykkes, for eksempel ved restriktive brannmurer som blokkerer utgående UDP, eller ved visse former for Carrier-Grade NAT, går trafikken via et DERP-relé over HTTPS. Også da forblir den ende-til-ende-kryptert med WireGuard; reléet ser bare krypterte pakker. Prisen er høyere forsinkelse og lavere gjennomstrømming. Peer Relays reduserer denne ulempen ved at en egen enhet med god tilkobling overtar videresendingen.

## Fordelene sammenlignet med et tradisjonelt VPN

| Kriterium | Tradisjonelt VPN | Tailscale |
|---|---|---|
| Topologi | Hub-and-spoke via en sentral gateway | Mesh, direkte forbindelser mellom enhetene |
| Innkommende porter | Gatewayen må være tilgjengelig fra internett | Ingen innkommende portvideresendinger nødvendig |
| Autentisering | Lokale kontoer, RADIUS, sertifikater, ofte separat MFA | Innlogging via eksisterende Identity Provider, inkludert MFA derfra |
| Tilgangsrettigheter | Ofte per nettverkssegment | Per bruker, gruppe, enhet og port i én sentral policy |
| Nytt sted | Site-to-site-tunnel med koordinerte parametere | Installer klient eller Subnet Router |
| Utfall av sentralen | Ingen tilgang for noen | Eksisterende forbindelser fortsetter; nye enheter og policyendringer venter |
| Protokoll | IPsec, SSL-VPN, OpenVPN | WireGuard |

### Mindre angrepsflate

Ettersom klientene oppretter forbindelsene utgående, trenger ingen enhet en åpen port på internett. En server som bare skal være tilgjengelig via Tailnet, kan binde tjenestene sine utelukkende til Tailscale-grensesnittet. Da er den ikke synlig for portskannere på internett. WireGuard selv er med rundt 4000 linjer kjernekode betydelig mindre enn typiske IPsec- eller SSL-VPN-implementasjoner og bruker et fast sett med moderne metoder (Curve25519, ChaCha20-Poly1305, BLAKE2s). Det finnes ingen forhandling av cipher suites, slik som ved IPsec regelmessig fører til feilkonfigurasjoner.

### Identitet i stedet for nettverksadresse

Hver enhet i Tailnet er knyttet til en bruker eller en tagg. Innloggingen skjer via Identity Provider som du allerede bruker; MFA og Conditional Access fra Microsoft Entra ID gjelder dermed også for nettverksaksess. Nye enheter kan bare knyttes til Tailnet etter vellykket innlogging hos Identity Provider, og hver enhetsnøkkel utløper som standard etter 180 dager. Hvis en medarbeider slutter i selskapet, sperrer du kontoen hos Identity Provider; med SCIM-provisioning (fra Standard-abonnementet) deaktiveres brukeren automatisk i Tailnet, og enhetene mister tilgangen.

### Tilgangsregler etter minste privilegiums prinsipp

Som standard kan alle enheter i et nytt Tailnet nå alle andre enheter. For produksjonsbruk definerer du tilgangsrettighetene i en sentral policyfil (HuJSON). Eksempelet nedenfor tillater administratorgruppen SSH og HTTPS til alle servere med taggen `tag:server`, mens alle andre brukere bare får HTTPS til intranettserveren:

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
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `groups` | Definerer brukergrupper; medlemmer angis via innloggingsadressen deres hos Identity Provider |
| `tagOwners` | Angir hvem som kan tildele en tagg til enheter; taggede enheter tilhører ingen bruker, men taggen |
| `grants` | Liste over tillatte forbindelser; det som ikke uttrykkelig er tillatt, blokkeres |
| `src` | Kilde for forbindelsen: bruker, gruppe, tagg eller `autogroup:member` (alle brukere i Tailnet) |
| `dst` | Mål for forbindelsen: tagg, vertsalias, enhet eller subnett |
| `ip` | Tillatte protokoller og porter, for eksempel `tcp:22` eller `*` for alt |
| `hosts` | Aliasnavn for Tailnet-adresser eller subnett som brukes i reglene |

</details>

Reglene håndheves lokalt av hver klient. En pakke som ikke er tillatt av policyen, forkastes allerede av målenheten. Coordination Server distribuerer bare policyen.

### Direkte forbindelser i stedet for omveier

Fordi tunnelene går direkte mellom enhetene, tar trafikken den korteste veien. To enheter på samme kontor kommuniserer lokalt, og en bærbar PC på hjemmekontor når en skyserver direkte. Dette reduserer forsinkelsen og avlaster internettforbindelsen på hovedstedet.

### Koble til eksisterende nettverk

Ikke alle enheter kan kjøre en Tailscale-klient, for eksempel skrivere, eldre NAS-systemer eller industrielle styringer. I slike tilfeller overtar en Subnet Router rollen som gateway: En Linux-server i mål-nettverket kunngjør det lokale subnettet i Tailnet (advertise), og autoriserte enheter når det via denne.

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
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `curl -fsSL …/install.sh \| sh` | Laster ned det offisielle installasjonsskriptet og setter opp distribusjonens pakkelager |
| `net.ipv4.ip_forward = 1` | Tillater Linux-kjernen å videresende pakker mellom grensesnitt; uten denne innstillingen fungerer ingen Subnet Router |
| `sysctl -p <datei>` | Laster inn innstillingen umiddelbart, uten omstart |
| `tailscale up` | Logger enheten inn i Tailnet; første gang vises en innloggingslenke |
| `--advertise-routes=<subnetz>` | Kunngjør det angitte subnettet i Tailnet; flere subnett skilles med komma |
| `--advertise-tags=<tag>` | Tildeler enheten en tagg som tilgangsreglene refererer til |

</details>

Den kunngjorte ruten må deretter godkjennes i administrasjonskonsollen, med mindre automatisk godkjenning (`autoApprovers`) er konfigurert i policyen. På tilsvarende måte kan en enhet drives som Exit Node med `--advertise-exit-node`. Klienter som velger denne Exit Node, videresender da all internett-trafikk via den, noe som tilsvarer full-tunnel-modus i et klassisk VPN.

### Mindre driftsarbeid

Coordination Server administrerer nøkkelrotasjon, adressetildeling, DNS og ruting. Et nytt sted trenger en Subnet Router med internettilgang, men ingen fast offentlig IP-adresse og ingen koordinering av IPsec-parametere med motparten. For fjernaksess til servere finnes dessuten Tailscale SSH (innlogging via Tailnet-identitet uten distribuerte SSH-nøkler) og `tailscale serve` (deling av en lokal webtjeneste i Tailnet).

## Begrensninger og avhengigheter

Tailscale erstatter den klassiske VPN-gatewayen, men flytter en del av ansvaret til en ekstern tjeneste. Du bør kontrollere disse punktene før innføring.

| Tema | Hva du må være oppmerksom på |
|---|---|
| Avhengighet av leverandøren | Control Plane er en skytjeneste fra Tailscale Inc. Ved et utfall fortsetter eksisterende forbindelser, men nye enheter, innlogginger og policyendringer er ikke mulig før tjenesten er gjenopprettet |
| Metadata | Enhetsnavn, Tailnet-adresser, offentlige IP-adresser, brukerkontoer og tilkoblingstidspunkter behandles hos leverandøren; ikke selve dataene |
| Tillit til nøkkeldistribusjon | Coordination Server bestemmer hvilke offentlige nøkler en enhet godtar. Tailnet Lock krever i tillegg en signatur fra pålitelige egne enheter |
| Kildekode | Klientkjernen er åpen kildekode (BSD-3-Clause), men Coordination Server er det ikke. Headscale er et alternativ med åpen kildekode som kan driftes selv, med redusert funksjonsomfang |
| Adressekonflikter | `100.64.0.0/10` brukes også av enkelte leverandører til Carrier-Grade NAT og av andre VPN-produkter; overlapp fører til rutingproblemer |
| Samspill med andre VPN-klienter | En annen VPN-klient med full tunnel kan omdirigere Tailscale-trafikken. Området `100.64.0.0/10` og Tailscale-tjenesten må der unntas fra tunnelen |
| Ytelse via reléer | Hvis det ikke opprettes en direkte forbindelse, reduseres gjennomstrømmingen merkbart via DERP; `tailscale netcheck` viser om UDP utgående fungerer |
| Ikke en erstatning for webfiltrering | Tailscale regulerer tilgang til interne ressurser. Det utfører ikke innholdsfiltrering eller inspeksjon av internett-trafikk |

For selskaper i Sveits gjelder den reviderte personvernloven (revDSG). Siden det behandles personopplysninger om brukerne hos leverandøren (kontoer, enheter, tilkoblingsmetadata), må du føre opp Tailscale i oversikten over behandlingsaktiviteter dersom selskapet ditt er pålagt å føre en slik oversikt (fra 250 ansatte eller ved behandlinger med høy risiko). Kontroller leverandørens databehandleravtale (Data Processing Addendum) og det rettslige grunnlaget for utlevering til utlandet i henhold til art. 16 revDSG, for eksempel sertifisering under Swiss-U.S. Data Privacy Framework eller standard kontraktsklausuler. Hvis databehandling hos leverandøren er utelukket, gjenstår Headscale som selvdriftet Control Plane.

> **EU-merknad:** For avdelinger i EU gjelder GDPR og reglene for overføring til tredjeland (art. 44 flg. GDPR). Vurderingen er innholdsmessig tilsvarende; der er EU-U.S. Data Privacy Framework avgjørende.

## Kostnader

Personal-abonnementet er gratis og omfatter opptil seks brukere, et ubegrenset antall brukerenheter og 50 taggede enheter (per oktober 2026). For forretningsbruk koster Standard-abonnementet 8 USD og Premium 18 USD per bruker per måned. Premium inkluderer blant annet Network Flow Logs, loggstrømming og utvidede alternativer for Tailscale SSH. Enterprise tilbys individuelt.

## Hvem Tailscale passer for

Tailscale passer særlig når brukere og ressurser er distribuert: hjemmekontor, skyservere hos flere leverandører, små avdelingskontorer uten fast IP-adresse eller fjernvedlikehold hos kunder. For små og mellomstore bedrifter kan det erstatte en VPN-gateway fullstendig. I større miljøer brukes det ofte parallelt med eksisterende VPN, for eksempel for administrasjonsaksess til servere, der de finmaskede tilgangsreglene gir størst nytte.

Det er mindre egnet hvis kravene forutsetter en fullt selvdriftet infrastruktur og Headscale ikke dekker funksjonsomfanget, eller hvis all internett-trafikk må filtreres sentralt. I så fall er det fortsatt nødvendig med en Secure Web Gateway som supplerer Tailscale.

For å teste holder det å installere klienten på to enheter og logge inn med samme konto. Med `tailscale status` ser du deretter alle enhetene i Tailnet, og med `tailscale ping <gerät>` kontrollerer du om en direkte forbindelse eller et relé brukes.

## Kilder

1.  [Tailscale: How Tailscale works](https://tailscale.com/blog/how-tailscale-works): Arkitektur med Coordination Server, WireGuard-mesh og lokal håndheving av reglene.

2.  [Tailscale: How NAT traversal works](https://tailscale.com/blog/how-nat-traversal-works): detaljert forklaring av STUN, UDP hole punching og tilfellene der et relé er nødvendig.

3.  [Tailscale Docs: DERP servers](https://tailscale.com/kb/1232/derp-servers): funksjon og kryptering for reléserverne.

4.  [Tailscale Docs: Tailscale Peer Relays](https://tailscale.com/kb/1591/peer-relays): egne enheter som relé, prioritert foran DERP.

5.  [Tailscale Docs: Grants](https://tailscale.com/kb/1324/grants): syntaks for tilgangsreglene i policyfilen.

6.  [Tailscale Docs: Subnet routers](https://tailscale.com/kb/1019/subnets): oppsett av Subnet Routers, inkludert IP-videresending og godkjenning av rutene.

7.  [Tailscale Docs: Exit nodes](https://tailscale.com/kb/1103/exit-nodes): videresend all internett-trafikk via en enhet i Tailnet.

8.  [Tailscale Docs: Tailnet Lock](https://tailscale.com/kb/1226/tailnet-lock): signering av nye enheter av egne, pålitelige noder.

9.  [Tailscale: Pricing](https://tailscale.com/pricing): abonnementer og grenser, hentet 1. oktober 2026.

10.  [WireGuard: Next Generation Kernel Network Tunnel (Whitepaper)](https://www.wireguard.com/papers/wireguard.pdf): protokolldesign og kryptografiske metoder som brukes.

11.  [GitHub: tailscale/tailscale](https://github.com/tailscale/tailscale): kildekode for klienten under BSD-3-Clause.

12.  [GitHub: juanfont/headscale](https://github.com/juanfont/headscale): implementering med åpen kildekode av Coordination Server for selvdrift.

13.  [CISA: Emergency Directive 24-01](https://www.cisa.gov/news-events/directives/ed-24-01-mitigate-ivanti-connect-secure-and-ivanti-policy-secure-vulnerabilities): pålegg om å koble sårbare Ivanti-VPN-gatewayer fra nettet i januar 2024.

14.  [Fedlex: Bundesgesetz über den Datenschutz (DSG)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 16 om utlevering av personopplysninger til utlandet.
