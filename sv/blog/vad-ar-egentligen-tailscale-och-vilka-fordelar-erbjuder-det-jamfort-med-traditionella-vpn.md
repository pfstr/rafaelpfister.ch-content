---
title: "Vad är egentligen Tailscale och vilka fördelar erbjuder det jämfört med traditionella VPN-anslutningar?"
navTitle: "Tailscale vs. VPN"
description: "Tailscale bygger ett mesh-VPN ovanpå WireGuard, där enheter ansluter direkt till varandra i stället för via en central VPN-koncentrator. Hur Coordination Server, NAT-traversering och DERP-reläer samverkar, vilka fördelar det har jämfört med IPsec- och SSL-VPN samt vilka beroenden och begränsningar du bör känna till före användning."
date: "2026-10-01"
kategorie: "VPN och fjärråtkomst"
timeToRead: "11 min. läsning"
themen:
  - vpn-fernzugriff
produkte:
  - "tailscale"
protokolle:
  - "tcp"
  - "haertung"
slug: "vad-ar-egentligen-tailscale-och-vilka-fordelar-erbjuder-det-jamfort-med-traditionella-vpn"
translationId: "article-91fddf1e0239f5c4"
translationOf: tailscale-vorteile-vpn
translationSourceHash: f7642d64432f28cd0e53fd833c1accbd7a2a206550be04e07ace9cea8d23a04f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-27T09:23:07.437Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/vad-ar-egentligen-tailscale-och-vilka-fordelar-erbjuder-det-jamfort-med-traditionella-vpn
---

# Vad är egentligen Tailscale och vilka fördelar erbjuder det jämfört med traditionella VPN-anslutningar?

Tailscale är en VPN-tjänst som ansluter enheter till ett privat nätverk, ett så kallat Tailnet. Tekniskt bygger den på WireGuard. Skillnaden jämfört med ett klassiskt företags-VPN ligger i topologin: Enheterna upprättar sina krypterade tunnlar direkt sinsemellan (mesh). En central VPN-koncentrator, genom vilken all trafik passerar, behövs inte. Endast administrationen är centraliserad: En Coordination Server distribuerar publika nycklar, adresser och åtkomstregler, men ser själv inga nyttodata.

För att komma i gång räcker det med ett konto hos en Identity Provider (Microsoft, Google, GitHub, Apple eller en OIDC-leverantör) och klienten på varje enhet. Portvidarebefordringar i brandväggen behövs normalt inte. Därmed är Tailscale vanligt både för hemnätverk och för fjärråtkomst till servrar och kundsystem.

## Så fungerar ett traditionellt VPN

Ett klassiskt VPN för fjärråtkomst fungerar enligt hub-and-spoke-principen. Vid kanten av företagsnätverket finns en VPN-gateway (brandvägg eller appliance) som är nåbar från internet. Klienten på den bärbara datorn upprättar en tunnel till denna gateway; vanliga alternativ är IPsec/IKEv2 (UDP 500 och 4500), SSL-VPN-varianter via TCP 443 eller OpenVPN. Efter inloggning får enheten en adress från en pool och rutter till de interna nätverken.

Denna modell har bevisat sitt värde under årtionden, men har strukturella egenskaper som medför arbete i dagens drift:

- **Publikt nåbar gateway:** VPN-koncentratorn måste vara nåbar från internet och är därmed ett attraktivt angreppsmål. Sårbarheter i VPN-appliances har under de senaste åren upprepade gånger utnyttjats aktivt; den amerikanska myndigheten CISA beordrade i januari 2024 till och med att berörda Ivanti-gatewayer skulle kopplas bort från nätet genom Emergency Directive 24-01.
- **Single Point of Failure och flaskhals:** All trafik passerar genom gatewayen. Om den fallerar eller bandbredden är uttömd påverkas alla användare samtidigt.
- **Omvägar (hairpinning):** Om två medarbetare som arbetar hemifrån använder samma server i molnet passerar trafiken först datacentret och går sedan ut igen.
- **Grova åtkomsträttigheter:** Efter att anslutningen har upprättats får enheten ofta åtkomst till ett helt nätverkssegment. Finmaskiga regler per användare och tjänst är möjliga, men underhålls sällan konsekvent.
- **Arbete med site-to-site:** Varje ytterligare plats behöver en egen tunnel med samordnade parametrar (Phase-1/Phase-2-proposals, Pre-Shared Keys eller certifikat, fasta publika IP-adresser).

## Så är Tailscale uppbyggt

Tailscale separerar Control Plane från Data Plane. Control Plane är Coordination Server, som Tailscale driver som molntjänst. Data Plane är WireGuard-tunnlarna mellan enheterna.

| Komponent | Uppgift |
|---|---|
| Klient (`tailscaled`) | Skapar nyckelparet lokalt, upprättar WireGuard-tunnlar till andra noder, tillämpar åtkomstreglerna lokalt |
| Coordination Server | Autentiserar enheter via Identity Provider, distribuerar publika nycklar, adresser, DNS-inställningar och policyn till alla noder |
| DERP-reläer | Vidarebefordrar krypterade paket när en direkt anslutning inte kan upprättas |
| Peer Relays | Egna enheter i Tailnet som fungerar som reläer med högre genomströmning; de prioriteras före DERP |

En enhets privata nyckel lämnar aldrig enheten. Coordination Server känner endast till de publika nycklarna och kan därför inte dekryptera trafiken. Varje enhet får en fast adress från området `100.64.0.0/10` (adressutrymmet för Carrier-Grade NAT) samt en IPv6-adress från `fd7a:115c:a1e0::/48`. Via MagicDNS kan enheterna dessutom nås under sina namn, till exempel `nas` eller `nas.tailnet-name.ts.net`.

### NAT-traversering: varför inga portvidarebefordringar behövs

De flesta enheter står bakom en NAT-router eller brandvägg och är inte direkt nåbara utifrån. Tailscale löser detta med NAT-traversering: Båda sidor fastställer sin publika adress och tilldelade port via STUN, utbyter denna information via Coordination Server och skickar samtidigt UDP-paket till varandra. De utgående paketen öppnar en tillståndspost i båda brandväggarna, genom vilken motpartens paket sedan kan komma in (UDP Hole Punching).

Om detta inte lyckas, exempelvis vid restriktiva brandväggar som blockerar utgående UDP eller vid vissa former av Carrier-Grade NAT, går trafiken via ett DERP-relä över HTTPS. Även då förblir den krypterad ända till ända med WireGuard; reläet ser endast krypterade paket. Priset är högre latens och lägre genomströmning. Peer Relays minskar denna nackdel genom att en egen enhet med bra anslutning tar över vidarebefordringen.

## Fördelarna jämfört med ett traditionellt VPN

| Kriterium | Traditionellt VPN | Tailscale |
|---|---|---|
| Topologi | Hub-and-spoke via en central gateway | Mesh, direkta anslutningar mellan enheterna |
| Inkommande portar | Gatewayen måste vara nåbar från internet | Inga inkommande portvidarebefordringar behövs |
| Autentisering | Lokala konton, RADIUS, certifikat, ofta separat MFA | Inloggning via befintlig Identity Provider inklusive dess MFA |
| Åtkomsträttigheter | Ofta per nätverkssegment | Per användare, grupp, enhet och port i en central policy |
| Ny plats | Site-to-site-tunnel med samordnade parametrar | Installera klient eller Subnet Router |
| Centralt fel | Ingen åtkomst för någon | Befintliga anslutningar fortsätter; nya enheter och policyändringar väntar |
| Protokoll | IPsec, SSL-VPN, OpenVPN | WireGuard |

### Mindre attackyta

Eftersom klienterna upprättar sina anslutningar utgående behöver ingen enhet ha en öppen port på internet. En server som endast ska vara nåbar via Tailnet kan binda sina tjänster uteslutande till Tailscale-gränssnittet. Då är den inte synlig för portskannrar på internet. WireGuard självt är med cirka 4000 rader kärnkod betydligt mindre än typiska IPsec- eller SSL-VPN-implementeringar och använder en fast uppsättning moderna metoder (Curve25519, ChaCha20-Poly1305, BLAKE2s). Det finns ingen förhandling av cipher suites, som vid IPsec regelbundet leder till felkonfigurationer.

### Identitet i stället för nätverksadress

Varje enhet i Tailnet är knuten till en användare eller en tagg. Inloggningen sker via den Identity Provider som du redan använder; MFA och Conditional Access från Microsoft Entra ID gäller därmed även för nätverksåtkomst. Nya enheter kan endast anslutas till Tailnet efter lyckad inloggning hos Identity Provider, och varje enhetsnyckel löper som standard ut efter 180 dagar. Om en medarbetare lämnar företaget spärrar du kontot hos Identity Provider; med SCIM-provisionering (från Standard-planen) avaktiveras användaren automatiskt i Tailnet och dess enheter förlorar åtkomsten.

### Åtkomstregler enligt principen om minsta behörighet

Som standard kan varje enhet i ett nytt Tailnet nå alla andra enheter. För produktionsanvändning definierar du åtkomsträttigheterna i en central policyfil (HuJSON). Följande exempel tillåter administratörsgruppen SSH och HTTPS till alla servrar med taggen `tag:server`, medan övriga användare endast får HTTPS till intranätservern:

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
<summary>Förklaring av alternativen</summary>

| Alternativ | Effekt |
|---|---|
| `groups` | Definierar användargrupper; medlemmar anges via sin inloggningsadress hos Identity Provider |
| `tagOwners` | Anger vem som får tilldela en tagg till enheter; taggade enheter tillhör ingen användare utan taggen |
| `grants` | Lista över tillåtna anslutningar; allt som inte uttryckligen är tillåtet blockeras |
| `src` | Anslutningens källa: användare, grupp, tagg eller `autogroup:member` (alla användare i Tailnet) |
| `dst` | Anslutningens mål: tagg, host-alias, enhet eller subnät |
| `ip` | Tillåtna protokoll och portar, till exempel `tcp:22` eller `*` för allt |
| `hosts` | Aliasnamn för Tailnet-adresser eller subnät som används i reglerna |

</details>

Reglerna tillämpas lokalt av varje klient. Ett paket som inte tillåts av policyn kasseras redan av målenheten. Coordination Server distribuerar endast policyn.

### Direkta anslutningar i stället för omvägar

Eftersom tunnlarna finns direkt mellan enheterna tar trafiken den kortaste vägen. Två enheter på samma kontor kommunicerar lokalt och en bärbar dator på hemmakontoret når en molnserver direkt. Detta minskar latensen och avlastar internetanslutningen vid huvudkontoret.

### Ansluta befintliga nätverk

Alla enheter kan inte köra en Tailscale-klient, exempelvis skrivare, äldre NAS-system eller industriella styrsystem. I sådana fall tar en Subnet Router rollen som gateway: En Linux-server i målnätet annonserar det lokala subnätet i Tailnet (advertise), och behöriga enheter når det via denna.

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
<summary>Förklaring av alternativen</summary>

| Alternativ | Effekt |
|---|---|
| `curl -fsSL …/install.sh \| sh` | Hämtar det officiella installationsskriptet och konfigurerar distributionens paketförråd |
| `net.ipv4.ip_forward = 1` | Tillåter Linux-kärnan att vidarebefordra paket mellan gränssnitt; utan denna inställning fungerar ingen Subnet Router |
| `sysctl -p <datei>` | Läser in inställningen direkt, utan omstart |
| `tailscale up` | Anmäler enheten till Tailnet; vid första körningen visas en inloggningslänk |
| `--advertise-routes=<subnetz>` | Annonserar det angivna subnätet i Tailnet; flera subnät separeras med kommatecken |
| `--advertise-tags=<tag>` | Tilldelar enheten en tagg som åtkomstreglerna hänvisar till |

</details>

Den annonserade rutten måste därefter godkännas i administratörskonsolen, om ingen automatisk godkänning (`autoApprovers`) är konfigurerad i policyn. På samma sätt kan en enhet köras som Exit Node med `--advertise-exit-node`. Klienter som väljer denna Exit Node vidarebefordrar då hela sin internettrafik via den, vilket motsvarar full-tunnel-läget i ett klassiskt VPN.

### Mindre driftarbete

Nyckelrotation, adressfördelning, DNS och routing hanteras av Coordination Server. En ny plats behöver en Subnet Router med internetåtkomst, men ingen fast publik IP-adress och ingen samordning av IPsec-parametrar med motparten. För fjärråtkomst till servrar finns dessutom Tailscale SSH (inloggning via Tailnet-identitet utan distribuerade SSH-nycklar) och `tailscale serve` (delning av en lokal webbtjänst i Tailnet).

## Begränsningar och beroenden

Tailscale ersätter den klassiska VPN-gatewayen, men flyttar en del av ansvaret till en extern tjänst. Du bör kontrollera dessa punkter före införandet.

| Ämne | Att tänka på |
|---|---|
| Beroende av leverantören | Control Plane är en molntjänst från Tailscale Inc. Vid avbrott fortsätter befintliga anslutningar, men nya enheter, inloggningar och policyändringar är inte möjliga förrän tjänsten är återställd |
| Metadata | Enhetsnamn, Tailnet-adresser, publika IP-adresser, användarkonton och anslutningstidpunkter behandlas av leverantören; inte nyttodata |
| Förtroende för nyckeldistributionen | Coordination Server bestämmer vilka publika nycklar en enhet accepterar. Tailnet Lock kräver dessutom en signatur från betrodda egna enheter |
| Källkod | Klientkärnan har öppen källkod (BSD-3-Clause), men Coordination Server har det inte. Headscale är ett alternativ med öppen källkod för egen drift med reducerad funktionalitet |
| Adresskonflikter | `100.64.0.0/10` används även av vissa leverantörer för Carrier-Grade NAT och av andra VPN-produkter; överlappningar leder till routingproblem |
| Samexistens med andra VPN-klienter | En andra VPN-klient med full tunnel kan omdirigera Tailscale-trafiken. Området `100.64.0.0/10` och Tailscale-tjänsten måste då undantas från tunneln |
| Prestanda via reläer | Om ingen direkt anslutning kan upprättas minskar genomströmningen märkbart via DERP; `tailscale netcheck` visar om UDP fungerar utgående |
| Ingen ersättning för webbfiltrering | Tailscale reglerar åtkomst till interna resurser. Det hanterar inte innehållsfiltrering och inspektion av internettrafik |

För företag i Schweiz gäller den reviderade dataskyddslagen (revDSG). Eftersom leverantören behandlar användarnas personuppgifter (konton, enheter, anslutningsmetadata) ska du ta upp Tailscale i registret över behandlingsaktiviteter, om ditt företag måste föra ett sådant (från 250 anställda eller vid behandlingar med hög risk). Kontrollera leverantörens avtal om personuppgiftsbehandling (Data Processing Addendum) och den rättsliga grunden för utlämnande till utlandet enligt art. 16 revDSG, exempelvis en certifiering enligt Swiss-U.S. Data Privacy Framework eller standardavtalsklausuler. Om behandling av data hos leverantören är utesluten återstår Headscale som egenägd Control Plane.

> **EU-anmärkning:** För verksamheter i EU gäller GDPR och dess regler för överföring till tredjeland (art. 44 ff. GDPR). Bedömningen är innehållsmässigt liknande; där är EU-U.S. Data Privacy Framework avgörande.

## Kostnader

Planen Personal är kostnadsfri och omfattar upp till sex användare, valfritt antal användarenheter och 50 taggade enheter (status oktober 2026). För företagsanvändning kostar Standard-planen 8 USD och Premium 18 USD per användare och månad. Premium lägger bland annat till Network Flow Logs, Log Streaming och utökade alternativ för Tailscale SSH. Enterprise erbjuds individuellt.

## Vem Tailscale passar för

Tailscale passar särskilt väl när användare och resurser är distribuerade: hemmakontor, molnservrar hos flera leverantörer, små externa platser utan fast IP-adress eller fjärrunderhåll hos kunder. För små och medelstora företag kan det helt ersätta en VPN-gateway. I större miljöer körs det ofta parallellt med det befintliga VPN:et, exempelvis för administrativ åtkomst till servrar, där de finmaskiga åtkomstreglerna ger störst nytta.

Det är mindre lämpligt när krav föreskriver en helt egenägd infrastruktur och Headscale inte täcker funktionsomfånget, eller när all internettrafik måste filtreras centralt. I så fall behövs fortfarande en Secure Web Gateway som kompletterar Tailscale.

För att testa räcker det att installera klienten på två enheter och logga in med samma konto. Med `tailscale status` ser du sedan alla enheter i Tailnet, och med `tailscale ping <gerät>` kontrollerar du om en direkt anslutning eller ett relä används.

## Ändringshistorik

- **25 oktober 2026:** Källänkar uppdaterade till Tailscales nya adresser.

## Källor

1.  [Tailscale: How Tailscale works](https://tailscale.com/blog/how-tailscale-works): Arkitektur med Coordination Server, WireGuard-mesh och lokal tillämpning av reglerna.

2.  [Tailscale: How NAT traversal works](https://tailscale.com/blog/how-nat-traversal-works): detaljerad förklaring av STUN, UDP Hole Punching och de fall där ett relä behövs.

3.  [Tailscale Docs: DERP servers](https://tailscale.com/docs/reference/derp-servers): Reläservrarnas funktion och kryptering.

4.  [Tailscale Docs: Tailscale Peer Relays](https://tailscale.com/docs/features/peer-relay): egna enheter som relä med prioritet före DERP.

5.  [Tailscale Docs: Grants](https://tailscale.com/docs/features/access-control/grants): syntax för åtkomstreglerna i policyfilen.

6.  [Tailscale Docs: Subnet routers](https://tailscale.com/docs/features/subnet-routers): konfigurering av Subnet Routers inklusive IP-forwarding och godkännande av rutterna.

7.  [Tailscale Docs: Exit nodes](https://tailscale.com/docs/features/exit-nodes): dirigera all internettrafik via en enhet i Tailnet.

8.  [Tailscale Docs: Tailnet Lock](https://tailscale.com/docs/features/tailnet-lock): signering av nya enheter av egna, betrodda noder.

9.  [Tailscale: Pricing](https://tailscale.com/pricing): planer och gränser, hämtat den 1 oktober 2026.

10.  [WireGuard: Next Generation Kernel Network Tunnel (Whitepaper)](https://www.wireguard.com/papers/wireguard.pdf): protokolldesign och använda kryptografiska metoder.

11.  [GitHub: tailscale/tailscale](https://github.com/tailscale/tailscale): klientens källkod under BSD-3-Clause.

12.  [GitHub: juanfont/headscale](https://github.com/juanfont/headscale): implementering med öppen källkod av Coordination Server för egen drift.

13.  [CISA: Emergency Directive 24-01](https://www.cisa.gov/news-events/directives/ed-24-01-mitigate-ivanti-connect-secure-and-ivanti-policy-secure-vulnerabilities): beslut om att koppla bort sårbara Ivanti VPN-gatewayer i januari 2024.

14.  [Fedlex: Bundesgesetz über den Datenschutz (DSG)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 16 om utlämnande av personuppgifter till utlandet.
